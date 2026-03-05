## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ Subsystem اللي بيتبعه الملف ده؟

الملف ده جزء من **GPIO subsystem** في Linux kernel، وبالتحديد من قسم **Device Tree bindings** — اللي بيوصف لـ kernel إزاي يتعرف على hardware معين من خلال الـ device tree.

---

### القصة من الأول

تخيل إنك بتبني نظام فضائي أو نظام تحكم صناعي دقيق جداً. مش بتستخدم x86 أو ARM عادي — بتستخدم معالج **SPARC** اسمه **Leon** من شركة **Aeroflex Gaisler** السويدية المتخصصة في الفضاء والأنظمة الحساسة.

شركة Gaisler بتبيع مكتبة اسمها **GRLIB** — مكتبة IP cores مكتوبة بـ **VHDL** (لغة وصف الـ hardware). جوا الـ GRLIB دي في core اسمه **GRGPIO** — ده chip افتراضي (synthesizable) بيتحط جوا الـ FPGA أو الـ ASIC وبيديك pins عامة تقدر تتحكم فيها بـ software.

---

### الـ GPIO ده إيه أصلاً؟

**GPIO** = General Purpose Input/Output.

تخيلها زي مقابس كهربا صغيرة جوا الـ chip — كل pin ممكن تعمله:
- **Input**: تقرأ منه (زي sensor أو زرار).
- **Output**: تكتب عليه (زي LED أو relay).

الـ **GRGPIO core** بيديك لحد **32 pin** ممكن تتحكم فيهم individually، وكل pin ممكن يطلع **interrupt** لو حصل تغيير فيه.

---

### المشكلة اللي الـ GRGPIO بيحلها

في بيئة Leon/SPARC، الـ hardware بيتعرف على نفسه تلقائياً عبر **AMBA Plug & Play** — زي USB plug and play بس للـ on-chip buses. بس لما بتشغّل Linux على النظام ده، الـ kernel محتاج يعرف:
- فين registers الـ GRGPIO في الـ memory؟
- أد إيه عنده GPIO lines؟
- أي line بيطلع interrupt؟
- الـ interrupt ده بييجي على أي IRQ number في الـ CPU؟

الإجابة على الأسئلة دي كلها بتيجي من **Device Tree** — ملف بيوصف الـ hardware بشكل static. والملف اللي إحنا بندرسه (`gpio-grgpio.txt`) هو **العقد** اللي بيقول: "لو لقيت node في الـ device tree بالخصائص دي، يبقى GRGPIO core."

---

### الملف ده بالظبط بيعمل إيه؟

**`gpio-grgpio.txt`** هو **Device Tree binding documentation** — يعني مش كود، ده وثيقة بتشرح الـ schema (القالب) اللي لازم يتبعه أي device tree node عشان يمثّل GRGPIO core.

لما الـ kernel يبوت، الـ device tree parser بيقرأ الـ `.dtb` file، بيلاقي node بـ `name = "GAISLER_GPIO"` أو `"01_01a"`، فبيعرف إنه GRGPIO وبيلود الـ driver المناسب.

---

### الخصائص المذكورة في الملف

| الخاصية | إجبارية؟ | المعنى |
|---|---|---|
| `name` | نعم | لازم تكون `"GAISLER_GPIO"` أو `"01_01a"` — الـ match ID |
| `reg` | نعم | عنوان وحجم registers الـ GRGPIO في الـ memory map |
| `interrupts` | نعم | قائمة الـ IRQ numbers اللي الـ core ممكن يرفعها للـ CPU |
| `nbits` | لا | عدد الـ GPIO lines (default = 32) |
| `irqmap` | لا | array بيقول: GPIO line رقم X متوصل بـ IRQ رقم Y |

---

### سيناريو عملي

```
FPGA board (Leon SPARC system)
├── AMBA Bus
│   ├── GRGPIO Core (32 GPIO pins)
│   │   ├── Pin 0  ──► LED
│   │   ├── Pin 1  ◄── Button (interrupt-capable)
│   │   ├── Pin 2  ◄── Sensor (interrupt-capable)
│   │   └── ...
│   └── Other IP cores
└── CPU (Leon3/Leon4)
```

الـ device tree node بيوصف:
- فين الـ GRGPIO registers؟ → `reg`
- أد إيه GPIO lines؟ → `nbits = <32>`
- أي lines ليها IRQ؟ → `irqmap = <0 0 0xffffffff ...>` (يعني pin 0 و 1 ليهم irq، pin 2 مفيش)

الـ driver (`gpio-grgpio.c`) بيقرأ المعلومات دي ويسجّل الـ GPIO chip في Linux GPIO subsystem.

---

### ليه الـ irqmap مهمة؟

الـ GRGPIO core ممكن يكون عنده أكتر من IRQ line واحدة للـ CPU — يعني مثلاً عنده 3 interrupt outputs. الـ `irqmap` بيقول: "GPIO pin 5 متوصل بـ interrupt output رقم 0 من الـ core، وpin 7 متوصل بـ interrupt output رقم 2."

لو pin مش عايزه يطلع interrupt، بتحط `0xffffffff` في الـ irqmap عنده.

---

### الملفات المهمة في الـ Subsystem ده

| الملف | الدور |
|---|---|
| `Documentation/devicetree/bindings/gpio/gpio-grgpio.txt` | **الملف الحالي** — binding spec لـ GRGPIO |
| `drivers/gpio/gpio-grgpio.c` | الـ driver نفسه — بيقرأ الـ DT ويشغّل الـ hardware |
| `include/linux/gpio/driver.h` | الـ API الأساسية لكتابة GPIO drivers في Linux |
| `drivers/gpio/gpio-generic.c` | generic helper بيساعد drivers تبسط الـ register access |
| `include/linux/irqdomain.h` | الـ IRQ domain API اللي بيستخدمها الـ driver لإدارة interrupts |
| `Documentation/devicetree/bindings/gpio/` | مجلد فيه binding docs لكل GPIO controllers |

---

### ملخص سريع

الملف ده هو "جواز سفر" الـ GRGPIO core في عالم Linux Device Tree — بيحدد الشكل الصح اللي لازم يكون عليه الـ DT node عشان الـ kernel يعرف إنه بيتعامل مع GPIO controller من نوع GRGPIO الخاص بـ Aeroflex Gaisler، المستخدم في أنظمة الفضاء والـ real-time المبنية على معالجات Leon SPARC.
## Phase 2: شرح الـ GPIO Framework

### المشكلة — ليه الـ GPIO Subsystem موجود أصلاً؟

الـ **GPIO (General Purpose Input/Output)** هو أبسط شكل من أشكال التواصل بين الـ processor والـ hardware — pin واحدة بتقرا أو بتكتب عليها 0 أو 1.

المشكلة مش في الفكرة، المشكلة في **التنوع الرهيب** في الـ hardware:

| المصدر | مثال |
|---|---|
| SoC GPIO controller | Raspberry Pi GPIO banks |
| FPGA IP core | GRLIB GRGPIO على Leon SPARC |
| I2C expander | MCP23017 عبر I2C bus |
| SPI expander | 74HC595 عبر SPI |
| PMIC | GPIOs داخل voltage regulator |

كل واحد فيهم ليه registers مختلفة، addresses مختلفة، وطريقة ضبط مختلفة. لو كل driver بيتعامل مع الـ GPIO بطريقته الخاصة، الكود هيبقى **فوضى كاملة** — كل consumer (LED driver، button driver، regulator) محتاج يعرف تفاصيل كل controller.

---

### الحل — الـ Kernel بيعمل إيه؟

الـ kernel بيحط **طبقة abstraction** اسمها **gpiolib**. الفكرة:

1. كل GPIO controller بيسجل نفسه عبر `struct gpio_chip`.
2. الـ gpiolib بيخزن كل الـ chips ويديها **namespace موحد**.
3. أي consumer بيتكلم مع الـ gpiolib API فقط — مش مع الـ hardware مباشرة.

الـ consumer مش محتاج يعرف إن الـ GPIO ده جاي من FPGA IP core أو من I2C expander — الـ API واحد.

---

### الـ Real-World Analogy — مكتب البريد

تخيل **مكتب بريد مركزي** في مدينة كبيرة.

- **المدينة** = الـ Linux kernel.
- **مكتب البريد المركزي** = الـ **gpiolib**.
- **فروع البريد المختلفة** (فرع الجيزة، فرع المعادي، فرع مدينة نصر) = كل **GPIO controller** (GRGPIO، BCM2835، MCP23017).
- **موظف كل فرع** = الـ `gpio_chip` operations (get, set, direction).
- **العميل** اللي بيبعت خطاب = الـ consumer driver (LED، button، sensor).
- **رقم الخطاب/الشحنة** = الـ GPIO number (global أو descriptor).
- **الـ irqmap** في GRGPIO = **دليل التوجيه الداخلي** — يحدد أي صندوق بريد (GPIO line) متصل بأي جرس إنذار (underlying IRQ).

العميل مش محتاج يعرف الفرع اللي هيوصله الخطاب — بيقول للمكتب المركزي "أنا عايز أبعت لـ GPIO رقم 42" والمكتب المركزي بيعرف هو مين المسؤول.

**الربط الدقيق:**

| عنصر في الـ analogy | المقابل الحقيقي في الـ kernel |
|---|---|
| مكتب البريد المركزي | `gpiolib` (drivers/gpio/gpiolib.c) |
| فرع البريد | `struct gpio_chip` — يمثل controller واحد |
| رقم الشحنة العالمي | GPIO number أو `struct gpio_desc` |
| موظف الفرع | callback functions: `get`, `set`, `direction_input` |
| جرس الإنذار في الفرع | underlying IRQ من الـ controller |
| دليل التوجيه الداخلي | `irqmap` property في devicetree + `grgpio_lirq` |
| طلب العميل | `gpiod_get()` أو `gpio_request()` |

---

### الـ Big Picture Architecture

```
  ┌─────────────────────────────────────────────────────────────┐
  │                      Consumer Drivers                        │
  │  LED driver  │  Button driver  │  Regulator  │  Net driver  │
  └──────────────────────┬──────────────────────────────────────┘
                         │  gpiod_get() / gpiod_set_value()
                         ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                  gpiolib (Core Subsystem)                    │
  │  - GPIO descriptor management  (gpio_desc)                   │
  │  - Namespace & numbering        (gpio_device)                │
  │  - sysfs / chardev interface    (/dev/gpiochipN)             │
  │  - OF/ACPI/swnode lookup        (gpiolib-of.c)               │
  │  - IRQ integration              (GPIOLIB_IRQCHIP)            │
  └──────┬────────────────┬───────────────────┬─────────────────┘
         │                │                   │
         ▼                ▼                   ▼
  ┌────────────┐  ┌──────────────┐  ┌──────────────────────┐
  │  GRGPIO    │  │  BCM2835     │  │  MCP23017            │
  │  (FPGA IP) │  │  (RPi SoC)   │  │  (I2C expander)      │
  │            │  │              │  │                      │
  │ gpio_chip  │  │  gpio_chip   │  │  gpio_chip           │
  │ .get       │  │  .get        │  │  .get (sleeps!)      │
  │ .set       │  │  .set        │  │  .set (sleeps!)      │
  │ .direction │  │  .direction  │  │  .direction          │
  │ .to_irq    │  │  irq_chip    │  │  (threaded IRQ)      │
  └────────────┘  └──────────────┘  └──────────────────────┘
         │
  ┌──────────────────────────────┐
  │         GRGPIO Registers     │
  │  0x00 DATA   (read inputs)   │
  │  0x04 OUTPUT (write outputs) │
  │  0x08 DIR    (direction)     │
  │  0x0c IMASK  (IRQ mask)      │
  │  0x10 IPOL   (IRQ polarity)  │
  │  0x14 IEDGE  (edge/level)    │
  │  0x20 IMAP   (IRQ mapping)   │
  └──────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ **`struct gpio_chip`** هو الـ abstraction الأساسية. ده العقد بين الـ controller driver والـ gpiolib.

```c
struct gpio_chip {
    const char  *label;      /* اسم الـ controller */
    u16          ngpio;      /* عدد الـ GPIO lines */
    int          base;       /* رقم الـ GPIO الأول (-1 = dynamic) */

    /* Operations — بيملأها الـ driver */
    int  (*get_direction)(struct gpio_chip *gc, unsigned int offset);
    int  (*direction_input)(struct gpio_chip *gc, unsigned int offset);
    int  (*direction_output)(struct gpio_chip *gc, unsigned int offset, int value);
    int  (*get)(struct gpio_chip *gc, unsigned int offset);
    int  (*set)(struct gpio_chip *gc, unsigned int offset, int value);
    int  (*to_irq)(struct gpio_chip *gc, unsigned int offset);

    /* IRQ integration (optional) */
    struct gpio_irq_chip irq;
};
```

الـ driver بيملأ الـ function pointers دي، وبيسجل الـ chip بـ `devm_gpiochip_add_data()`. من اللحظة دي، الـ gpiolib بيعرف يتعامل مع الـ controller.

---

### الـ Struct Relationships في GRGPIO

```
  struct grgpio_priv
  ┌─────────────────────────────────────────┐
  │  chip: struct gpio_generic_chip         │
  │  ┌──────────────────────────────────┐   │
  │  │  gc: struct gpio_chip            │   │
  │  │   .ngpio = 32                    │   │
  │  │   .to_irq = grgpio_to_irq        │   │
  │  │   .get / .set / .direction       │   │──► gpiolib يستخدمها
  │  └──────────────────────────────────┘   │
  │                                         │
  │  regs: void __iomem *  ──────────────────────► MMIO registers
  │  imask: u32  (shadow of GRGPIO_IMASK)   │
  │                                         │
  │  domain: struct irq_domain *  ───────────────► IRQ Domain
  │                                         │
  │  uirqs[32]: struct grgpio_uirq          │
  │  ┌─────────────────────────────┐        │
  │  │  uirq: u8  (Linux IRQ num)  │        │   الـ IRQs الفعلية اللي
  │  │  refcnt: atomic_t           │        │   الـ controller بيستخدمها
  │  └─────────────────────────────┘        │
  │                                         │
  │  lirqs[32]: struct grgpio_lirq          │
  │  ┌─────────────────────────────┐        │
  │  │  index: s8  (-1 = no irq)   │        │   كل GPIO line → uirq index
  │  │  irq: u8   (virtual IRQ)    │        │
  │  └─────────────────────────────┘        │
  └─────────────────────────────────────────┘
```

---

### الـ IRQ Subsystem داخل GRGPIO — تعمق أكتر

ده الجزء الأكثر تعقيداً في الـ driver. الـ GRGPIO عنده **معمارية IRQ طبقتين**:

**الطبقة الأولى — Underlying IRQs:**
الـ GRGPIO controller نفسه مربوط بعدد محدود من الـ IRQs على الـ interrupt controller الرئيسي (LEON interrupt controller). هذه هي الـ `uirqs`.

**الطبقة الثانية — Per-GPIO Virtual IRQs:**
كل GPIO line بيحتاج يقدر يطلع IRQ مستقل للـ consumer. الـ kernel بيخلق **virtual IRQs** عبر الـ `irq_domain`.

**الـ irqmap في Device Tree:**
```
irqmap = <0 0 1 0xffffffff 1 ...>;
```
الرقم ده هو الـ **index في مصفوفة الـ interrupts** في الـ DT node. بمعنى: GPIO[0] → interrupt[0]، GPIO[2] → interrupt[1]، GPIO[3] → لا يوجد IRQ.

**تسلسل الأحداث لما GPIO line تطلع interrupt:**

```
  GPIO Pin يتغير
        │
        ▼
  GRGPIO Hardware يرفع underlying IRQ (uirq)
        │
        ▼
  grgpio_irq_handler() يتشغل
        │
        ▼ يلف على كل lirqs[i]
  بيشوف أي GPIO lines مفعل ومربوط بالـ uirq ده
        │
        ▼
  generic_handle_irq(lirq->irq)  ← يعمل trigger للـ virtual IRQ
        │
        ▼
  Consumer driver handler يتشغل
```

**إدارة الـ refcount:**
لو GPIO[0] وGPIO[1] الاتنين مربوطين بنفس الـ underlying IRQ، الـ `request_irq()` بيتعمل مرة واحدة بس. الـ `atomic_t refcnt` في `grgpio_uirq` بيضمن ده:

```c
/* في grgpio_irq_map: */
if (atomic_fetch_add(1, &uirq->refcnt) == 0) {
    /* أول مرة نطلب الـ underlying IRQ */
    request_irq(uirq->uirq, grgpio_irq_handler, ...);
}

/* في grgpio_irq_unmap: */
if (atomic_dec_and_test(&uirq->refcnt)) {
    /* آخر مستخدم — حرر الـ IRQ */
    free_irq(uirq->uirq, priv);
}
```

**ملاحظة على الـ IRQ Domain:** الـ `irq_domain` هو subsystem منفصل مسؤول عن الـ mapping بين **hardware IRQ numbers** و **Linux virtual IRQ numbers**. الـ GRGPIO بيستخدم `irq_domain_create_linear()` — بيعمل linear mapping حيث hwirq = GPIO offset.

---

### الـ gpio_generic_chip — طبقة إضافية

الـ GRGPIO مش بيستخدم `struct gpio_chip` مباشرة، بيستخدم `struct gpio_generic_chip` اللي بيوفر **register-based get/set/direction** جاهزة:

```c
config = (struct gpio_generic_chip_config) {
    .dev    = dev,
    .sz     = 4,                      /* register size = 32-bit */
    .dat    = regs + GRGPIO_DATA,     /* input data register */
    .set    = regs + GRGPIO_OUTPUT,   /* output register */
    .dirout = regs + GRGPIO_DIR,      /* direction register */
    .flags  = GPIO_GENERIC_BIG_ENDIAN_BYTE_ORDER,
};
gpio_generic_chip_init(&priv->chip, &config);
```

الـ `gpio_generic_chip` بيملأ `gc->get` و `gc->set` و `gc->direction_input` وغيرها تلقائياً بناءً على الـ register addresses دي. الـ driver GRGPIO مش محتاج يكتب الـ functions دي من الأول.

---

### الـ Probe Flow — خطوة بخطوة

```
grgpio_probe()
    │
    ├── devm_kzalloc()           ← allocate private state
    ├── devm_platform_ioremap_resource()  ← map MMIO registers
    ├── gpio_generic_chip_init()  ← setup get/set/direction via registers
    │
    ├── gc->to_irq = grgpio_to_irq  ← hook لتحويل GPIO → virtual IRQ
    ├── gc->ngpio  = from DT "nbits" property (default 32)
    ├── gc->base   = -1  (dynamic numbering)
    │
    ├── [if irqmap present in DT]:
    │       ├── irq_domain_create_linear()  ← create IRQ domain
    │       └── for each GPIO line:
    │               └── لو عنده IRQ: platform_get_irq() → save uirq
    │
    └── devm_gpiochip_add_data()  ← register with gpiolib
```

---

### الـ gpiolib بيملك إيه — والـ Driver بيملك إيه؟

| المسؤولية | المالك |
|---|---|
| Namespace وترقيم الـ GPIO lines | **gpiolib** |
| sysfs / chardev interface | **gpiolib** |
| Device Tree lookup وتحويل phandle → gpio_desc | **gpiolib** (gpiolib-of.c) |
| Consumer API: `gpiod_get()`, `gpiod_set_value()` | **gpiolib** |
| كتابة/قراءة الـ hardware registers | **Driver (GRGPIO)** |
| ضبط اتجاه الـ pin (input/output) على الـ hardware | **Driver (GRGPIO)** |
| إدارة الـ underlying IRQs وربطها بالـ GPIO lines | **Driver (GRGPIO)** |
| IRQ polarity/edge configuration على الـ hardware | **Driver (GRGPIO)** |
| تحديد عدد الـ GPIO lines المتاحة | **Driver (GRGPIO)** — من DT |
| تحويل GPIO offset → virtual IRQ | **Driver** (to_irq) + **gpiolib** (irq_domain) |
| Thread safety للـ IRQ operations | **مشترك** — driver يستخدم locks من gpio_generic |

---

### الخلاصة الهندسية

الـ GPIO subsystem في الـ Linux kernel هو مثال كلاسيكي على نمط **provider/consumer** مع **abstraction layer** في المنتصف:

- الـ **GRGPIO driver** هو الـ provider — يعرف كيف يكلم الـ FPGA IP core.
- الـ **gpiolib** هو الـ broker — يدير الـ namespace ويوفر API موحد.
- الـ **consumer drivers** (LED, button, etc.) هم الـ consumers — يستخدمون الـ API من غير ما يعرفوا أي hardware تحتهم.

الـ GRGPIO بالذات بيضيف تعقيداً إضافياً في الـ IRQ layer: لأن الـ FPGA controller عنده **many-to-one** mapping بين الـ GPIO lines والـ underlying IRQs، الـ driver لازم يدير الـ multiplexing ده يدوياً باستخدام الـ `irqmap` والـ `refcnt` mechanism.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Register Map — Cheatsheet

| Offset | Macro | الوظيفة |
|--------|-------|---------|
| `0x00` | `GRGPIO_DATA` | قراءة حالة الـ GPIO pins (input data) |
| `0x04` | `GRGPIO_OUTPUT` | كتابة قيمة الـ output على الـ pins |
| `0x08` | `GRGPIO_DIR` | تحديد الاتجاه: `1` = output، `0` = input |
| `0x0C` | `GRGPIO_IMASK` | interrupt mask — `1` يعني enable الـ interrupt للـ pin ده |
| `0x10` | `GRGPIO_IPOL` | interrupt polarity — `0` = low/falling، `1` = high/rising |
| `0x14` | `GRGPIO_IEDGE` | interrupt edge/level — `0` = level، `1` = edge |
| `0x18` | `GRGPIO_BYPASS` | bypass register (hardware-specific) |
| `0x20+` | `GRGPIO_IMAP_BASE` | interrupt map — بيربط كل line بـ underlying irq |

---

### الـ IRQ Type Flags — Cheatsheet

| IRQ Type | `IPOL` bit | `IEDGE` bit | الوصف |
|----------|-----------|------------|-------|
| `IRQ_TYPE_LEVEL_LOW` | `0` | `0` | trigger لما الـ pin يبقى low |
| `IRQ_TYPE_LEVEL_HIGH` | `1` | `0` | trigger لما الـ pin يبقى high |
| `IRQ_TYPE_EDGE_FALLING` | `0` | `1` | trigger على الـ falling edge |
| `IRQ_TYPE_EDGE_RISING` | `1` | `1` | trigger على الـ rising edge |

---

### الـ Config Flags — Cheatsheet

| Flag / Constant | القيمة | الاستخدام |
|-----------------|--------|-----------|
| `GRGPIO_MAX_NGPIO` | `32` | أقصى عدد GPIO lines |
| `GPIO_GENERIC_BIG_ENDIAN_BYTE_ORDER` | - | الـ GRLIB/SPARC بيشتغل big-endian |
| `IRQCHIP_IMMUTABLE` | - | الـ irq_chip مش هيتغير بعد التسجيل |
| `irqmap[i] = 0xffffffff` (i.e., `-1` كـ s32) | - | الـ GPIO line ده مفيهوش interrupt |

---

### Device Tree Properties — Cheatsheet

| Property | Required? | الوصف |
|----------|-----------|-------|
| `name` | نعم | لازم يكون `"GAISLER_GPIO"` أو `"01_01a"` |
| `reg` | نعم | عنوان وحجم الـ register set |
| `interrupts` | نعم | الـ underlying irqs بتاعت الـ core |
| `nbits` | لا | عدد الـ GPIO lines، الـ default هو 32 |
| `irqmap` | لا | array بيربط كل GPIO line بـ interrupt index |

---

### الـ Structs المهمة

---

#### `struct grgpio_uirq`

**الغرض:** بتمثل الـ **underlying IRQ** — يعني الـ interrupt الفعلي اللي بيجي من الـ GRGPIO core نفسه للـ CPU. الـ core ممكن يكون عنده أكتر من interrupt واحد، وكل GPIO line بيتربط بواحد منهم.

```c
struct grgpio_uirq {
    atomic_t refcnt; /* عداد المستخدمين — لما يوصل 0 يتعمله free_irq */
    u8 uirq;         /* رقم الـ IRQ الفعلي من kernel's irq space */
};
```

| Field | النوع | الشرح |
|-------|-------|-------|
| `refcnt` | `atomic_t` | بيعد كام GPIO line شغالة على نفس الـ underlying irq — لما يوصل 0 يتعمله `free_irq` |
| `uirq` | `u8` | رقم الـ IRQ الفعلي اللي بيتعمله `request_irq` |

---

#### `struct grgpio_lirq`

**الغرض:** بتمثل الـ **logical IRQ** — يعني الـ interrupt اللي بيشوفه الـ consumer (مثلاً device driver تاني) لما يطلب interrupt من GPIO line معين.

```c
struct grgpio_lirq {
    s8 index; /* index في uirqs[] — أو -1 لو مفيش irq */
    u8 irq;   /* رقم الـ IRQ اللي اتوزع للـ GPIO line ده */
};
```

| Field | النوع | الشرح |
|-------|-------|-------|
| `index` | `s8` | بيوصّل الـ logical irq بالـ underlying irq عن طريق `uirqs[index]` — القيمة `-1` تعني مفيش interrupt لـ line ده |
| `irq` | `u8` | رقم الـ virtual IRQ اللي بيتوزع من الـ irq domain |

---

#### `struct grgpio_priv`

**الغرض:** الـ **private data** الرئيسية للـ driver — بتجمع كل حاجة خاصة بالـ GRGPIO instance، من registers لحد الـ IRQ management.

```c
struct grgpio_priv {
    struct gpio_generic_chip chip;          /* الـ generic GPIO chip (wrapper) */
    void __iomem *regs;                     /* pointer للـ MMIO registers */
    struct device *dev;                     /* الـ device للـ logging وغيره */

    u32 imask;                              /* shadow copy للـ IMASK register */

    struct irq_domain *domain;              /* الـ irq domain بتاع الـ GPIO lines */

    struct grgpio_uirq uirqs[GRGPIO_MAX_NGPIO]; /* الـ underlying irqs */
    struct grgpio_lirq lirqs[GRGPIO_MAX_NGPIO]; /* الـ logical irqs per GPIO line */
};
```

| Field | الشرح |
|-------|-------|
| `chip` | الـ `gpio_generic_chip` اللي بيحتوي على `gpio_chip` وعمليات القراءة والكتابة |
| `regs` | بيتعمله `ioremap` في الـ probe — كل الـ register access بيمشي منه |
| `dev` | بيستخدم في `dev_err/dev_info` وفي `devm_*` allocations |
| `imask` | shadow copy للـ IMASK register — بيتكتب في كل `grgpio_set_imask` call |
| `domain` | الـ irq_domain اللي بيترجم من GPIO line number لـ virtual IRQ number |
| `uirqs[]` | array بحجم 32 — كل عنصر بيمثل underlying irq ممكن يشترك فيه أكتر من GPIO line |
| `lirqs[]` | array بحجم 32 — كل عنصر بيمثل الـ logical irq للـ GPIO line رقم i |

---

#### `struct irq_chip` — instance: `grgpio_irq_chip`

**الغرض:** بيعرف العمليات اللي الـ kernel بيستدعيها لإدارة الـ IRQs على مستوى الـ GPIO.

```c
static const struct irq_chip grgpio_irq_chip = {
    .name        = "grgpio",
    .irq_mask    = grgpio_irq_mask,    /* disable الـ interrupt للـ line ده */
    .irq_unmask  = grgpio_irq_unmask,  /* enable الـ interrupt للـ line ده */
    .irq_set_type = grgpio_irq_set_type, /* تحديد edge/level والـ polarity */
    .flags = IRQCHIP_IMMUTABLE,
    GPIOCHIP_IRQ_RESOURCE_HELPERS,
};
```

---

#### `struct irq_domain_ops` — instance: `grgpio_irq_domain_ops`

```c
static const struct irq_domain_ops grgpio_irq_domain_ops = {
    .map   = grgpio_irq_map,    /* ربط virtual IRQ بـ GPIO line */
    .unmap = grgpio_irq_unmap,  /* تحرير الـ mapping وإلغاء الـ underlying irq */
};
```

---

### Struct Relationship Diagram

```
platform_device
      │
      ▼
 grgpio_probe()
      │
      ├──allocates──► grgpio_priv
      │                    │
      │                    ├──contains──► gpio_generic_chip (chip)
      │                    │                    │
      │                    │                    └──contains──► gpio_chip (gc)
      │                    │                                        │
      │                    │                                        ├── ngpio
      │                    │                                        ├── base
      │                    │                                        ├── label
      │                    │                                        └── to_irq ──► grgpio_to_irq()
      │                    │
      │                    ├──regs──► MMIO registers (0x00..0x20+)
      │                    │
      │                    ├──domain──► irq_domain
      │                    │               │
      │                    │               └── ops ──► grgpio_irq_domain_ops
      │                    │                               ├── .map   = grgpio_irq_map
      │                    │                               └── .unmap = grgpio_irq_unmap
      │                    │
      │                    ├──uirqs[32]──► grgpio_uirq[]
      │                    │                   ├── [0]: { uirq=X, refcnt=N }
      │                    │                   ├── [1]: { uirq=Y, refcnt=M }
      │                    │                   └── ...
      │                    │
      │                    └──lirqs[32]──► grgpio_lirq[]
      │                                        ├── [0]: { index=0, irq=V0 } ──► uirqs[0]
      │                                        ├── [1]: { index=0, irq=V1 } ──► uirqs[0]
      │                                        ├── [2]: { index=1, irq=V2 } ──► uirqs[1]
      │                                        ├── [3]: { index=-1 }  (no irq)
      │                                        └── ...
      │
      └──registers──► gpiochip (kernel GPIO subsystem)
                            │
                            └──irq_chip──► grgpio_irq_chip
```

**ملاحظة مهمة على العلاقة بين `lirqs` و `uirqs`:** أكتر من GPIO line ممكن يشتركوا في نفس الـ underlying irq. مثلاً `lirqs[0].index = 0` و `lirqs[1].index = 0` يعني الاتنين بيشتركوا في `uirqs[0]`. الـ `refcnt` هو اللي بيحدد إمتى يتعمل `request_irq` / `free_irq`.

---

### Lifecycle Diagram

```
═══════════════════════════════════════════════════════════
                    DRIVER LIFECYCLE
═══════════════════════════════════════════════════════════

[1] PROBE — grgpio_probe()
    │
    ├─► devm_kzalloc → allocate grgpio_priv
    │
    ├─► devm_platform_ioremap_resource → map MMIO registers
    │
    ├─► gpio_generic_chip_init → initialize gpio_generic_chip
    │       └── setup: dat=DATA_REG, set=OUTPUT_REG, dirout=DIR_REG
    │
    ├─► read "nbits" from DT → set gc->ngpio (default: 32)
    │
    ├─► [if irqmap in DT]:
    │       ├─► irq_domain_create_linear → create irq_domain (size=ngpio)
    │       ├─► devm_add_action_or_reset → register cleanup (irq_domain_remove)
    │       └─► for each gpio line:
    │               ├─► read irqmap[i] → lirqs[i].index
    │               └─► platform_get_irq(index) → uirqs[index].uirq
    │
    └─► devm_gpiochip_add_data → register with kernel GPIO subsystem

[2] IRQ MAPPING — on first use of a GPIO line's IRQ
    │
    ├─► consumer calls gpiod_to_irq() or similar
    │       └─► gc->to_irq = grgpio_to_irq()
    │               └─► irq_create_mapping(domain, gpio_offset)
    │                       └─► grgpio_irq_map()
    │                               ├─► store lirq->irq
    │                               ├─► if atomic_fetch_add(refcnt)==0:
    │                               │       └─► request_irq(uirq, grgpio_irq_handler)
    │                               └─► irq_set_chip_and_handler → handle_simple_irq

[3] RUNTIME — interrupt firing
    │
    ├─► hardware GPIO pin toggles → underlying irq fires
    │       └─► grgpio_irq_handler(uirq, priv)
    │               └─► for each gpio line:
    │                       if imask & BIT(i) && lirq->index matches uirq:
    │                           └─► generic_handle_irq(lirq->irq)

[4] IRQ UNMAP — when consumer frees the IRQ
    │
    └─► grgpio_irq_unmap()
            ├─► clear chip/handler from irq
            ├─► grgpio_set_imask(priv, i, 0) → disable in IMASK
            ├─► lirq->irq = 0
            └─► if atomic_dec_and_test(refcnt):
                    └─► free_irq(uirq, priv)

[5] REMOVE — driver unbind (devm handles it all)
    │
    ├─► devm cleanup: irq_domain_remove(domain)
    ├─► devm cleanup: gpiochip removed
    └─► devm cleanup: iounmap, kfree
```

---

### Call Flow Diagrams

#### تغيير direction أو قراءة/كتابة GPIO

```
user/driver
    │
    ▼
gpiod_set_value(desc, val)
    │
    ▼
gpio_chip->set() [من gpio_generic — يُحسب تلقائياً من config]
    │
    ▼
gpio_generic_write_reg(&priv->chip, regs + GRGPIO_OUTPUT, val)
    │
    ▼
writel(val, GRGPIO_OUTPUT)  ← MMIO write للهارد وير
```

#### تفعيل الـ IRQ على GPIO line

```
consumer driver
    │
    ▼
devm_request_irq(dev, gpio_to_irq(gpio), handler, ...)
    │
    ▼
gc->to_irq(gc, offset)  [= grgpio_to_irq]
    │
    ├── check: offset < ngpio && lirqs[offset].index >= 0
    │
    ▼
irq_create_mapping(priv->domain, offset)
    │
    ▼
grgpio_irq_map(domain, virq, hwirq=offset)
    │
    ├── lirq->irq = virq
    ├── uirq = &priv->uirqs[lirq->index]
    │
    ├── [if refcnt was 0]:
    │       request_irq(uirq->uirq, grgpio_irq_handler, ...)
    │
    └── irq_set_chip_and_handler(virq, &grgpio_irq_chip, handle_simple_irq)
```

#### تشغيل الـ IRQ (interrupt fires)

```
hardware pin edge/level detected
    │
    ▼
CPU receives underlying irq (uirq->uirq)
    │
    ▼
grgpio_irq_handler(irq=uirq, dev=priv)
    │
    ├── lock (gpio_generic_lock_irqsave)
    │
    ├── for i in 0..ngpio:
    │       if (imask & BIT(i))
    │       && (lirqs[i].index >= 0)
    │       && (uirqs[lirqs[i].index].uirq == irq):
    │           generic_handle_irq(lirqs[i].irq)
    │               │
    │               └──► consumer's interrupt handler runs
    │
    └── unlock
```

#### تعطيل/تفعيل الـ IRQ mask

```
kernel IRQ core
    │
    ├── mask:   grgpio_irq_mask(irq_data)
    │               ├── grgpio_set_imask(priv, offset, 0)
    │               │       └── writel(imask & ~BIT(offset), GRGPIO_IMASK)
    │               └── gpiochip_disable_irq(gc, hwirq)
    │
    └── unmask: grgpio_irq_unmask(irq_data)
                    ├── gpiochip_enable_irq(gc, hwirq)
                    └── grgpio_set_imask(priv, offset, 1)
                            └── writel(imask | BIT(offset), GRGPIO_IMASK)
```

---

### Locking Strategy

#### الـ Lock المستخدم

الـ driver مش بيعرّف lock خاص بيه — بيعتمد على **`gpio_generic_chip`'s built-in spinlock** اللي بيتعمله access عن طريق:

| Macro / Function | الاستخدام |
|------------------|-----------|
| `guard(gpio_generic_lock_irqsave)(&priv->chip)` | lock + disable IRQs (scoped — يتعمله unlock تلقائي عند خروج الـ scope) |
| `scoped_guard(gpio_generic_lock_irqsave, &priv->chip)` | نفس الفكرة لكن في بلوك |
| `gpio_generic_chip_lock_irqsave(&priv->chip, flags)` | manual lock مع حفظ الـ flags |
| `gpio_generic_chip_unlock_irqrestore(&priv->chip, flags)` | manual unlock مع restore الـ flags |

#### إيه اللي بيحميه الـ Lock؟

| المورد | الحماية |
|--------|---------|
| `priv->imask` (shadow register) | spinlock — أي تعديل عليه داخل `grgpio_set_imask` |
| `GRGPIO_IMASK` register | spinlock — كل read/write للـ MMIO |
| `GRGPIO_IPOL` و `GRGPIO_IEDGE` | spinlock — في `grgpio_irq_set_type` |
| `lirq->irq` | spinlock — في `grgpio_irq_map` و `grgpio_irq_unmap` |

#### الـ `atomic_t refcnt` في `grgpio_uirq`

الـ `refcnt` بيتعمله access بـ atomic operations (`atomic_fetch_add`, `atomic_dec_and_test`) — **بدون** الحاجة للـ spinlock — لأن الـ check-and-act على الـ `request_irq`/`free_irq` لازم يحصل **خارج** الـ lock (لأن `request_irq` ممكن تنام).

#### Lock Ordering — مفيش nested locks

الـ driver بيستخدم lock واحد بس (الـ gpio_generic spinlock). ومفيش أي nested locking scenario — بالتالي مفيش خطر deadlock من اتجاه الـ locking.

#### تحذير مهم في `grgpio_irq_unmap`

```c
if (atomic_dec_and_test(&uirq->refcnt)) {
    gpio_generic_chip_unlock_irqrestore(&priv->chip, flags); /* ← unlock FIRST */
    free_irq(uirq->uirq, priv);                              /* ← then free */
    return;
}
```

الـ `free_irq` بيحتاج ينتظر لحد ما الـ IRQ handler ينهي — لو الـ lock لسه ممسوك والـ IRQ handler بيحاول يمسك نفس الـ lock → **deadlock**. عشان كده بيتعمله `unlock` الأول.
## Phase 4: شرح الـ Functions

---

### ملخص — Cheatsheet للـ DT Properties والـ Driver Functions

#### الـ Device Tree Binding Properties

| Property | نوعها | مطلوبة؟ | القيمة الافتراضية |
|---|---|---|---|
| `name` | string | Required | — |
| `reg` | `<addr len>` | Required | — |
| `interrupts` | `<irq...>` | Required | — |
| `nbits` | u32 | Optional | 32 |
| `irqmap` | `<u32 array>` | Optional | لا يوجد IRQ support |

#### الـ Driver Functions الرئيسية

| Function | الفئة | الغرض |
|---|---|---|
| `grgpio_probe` | Registration | entry point، بيعمل init للـ chip |
| `grgpio_to_irq` | GPIO API | بيحوّل GPIO offset لـ Linux IRQ number |
| `grgpio_set_imask` | Helper | بيعدّل interrupt mask register |
| `grgpio_irq_map` | IRQ Domain | بيربط hwirq بـ Linux IRQ |
| `grgpio_irq_unmap` | IRQ Domain | بيعكس الـ mapping ويحرر الـ underlying IRQ |
| `grgpio_irq_handler` | IRQ Runtime | الـ ISR الرئيسي للـ core |
| `grgpio_irq_mask` | IRQ Chip | بيعمل mask لـ GPIO IRQ محدد |
| `grgpio_irq_unmask` | IRQ Chip | بيعمل unmask لـ GPIO IRQ محدد |
| `grgpio_irq_set_type` | IRQ Chip | بيضبط polarity وedge/level |
| `grgpio_irq_domain_remove` | Cleanup | devm action لإزالة الـ irq domain |

---

### فئة 1: الـ Device Tree Binding Properties

الـ binding بتوصف الـ GRGPIO IP core من Aeroflex Gaisler — ده GPIO controller بيظهر في الـ GRLIB VHDL IP library على أنظمة Leon/SPARC. في بيئته الطبيعية الـ properties دي بتتبنى تلقائياً من الـ AMBA plug&play، لكن لما بنستخدمه في DT بنحددها يدوياً.

---

#### Property: `name`

```
name = "GAISLER_GPIO";
/* أو */
name = "01_01a";
```

**بتعمل إيه:**
الـ `name` property هي اللي بيستخدمها الـ driver عشان يتطابق مع الـ device من خلال `of_match_table`. الـ driver بيحدد اتنين قيم مقبولة: الاسم التجاري `"GAISLER_GPIO"` أو الـ vendor/device ID المختصر `"01_01a"` اللي بييجي من الـ AMBA plug&play identification.

**القيم المقبولة:**

| القيمة | السياق |
|---|---|
| `"GAISLER_GPIO"` | الاسم الكامل في GRLIB |
| `"01_01a"` | Vendor=0x01, Device=0x01a (AMBA P&P ID) |

**الربط بالكود:**

```c
static const struct of_device_id grgpio_match[] = {
    {.name = "GAISLER_GPIO"},
    {.name = "01_01a"},
    {},
};
MODULE_DEVICE_TABLE(of, grgpio_match);
```

الـ OF subsystem بيقارن الـ `name` property في الـ DT node بالـ `grgpio_match` table. لو ماتطابقتش، الـ `grgpio_probe` مش هيتشال خالص.

---

#### Property: `reg`

```
reg = <0x80000b00 0x100>;
```

**بتعمل إيه:**
بتحدد الـ base address والـ size للـ register set بتاع الـ GRGPIO core في الـ address space. الـ driver بيستخدمها عشان يعمل `ioremap` للـ registers.

**البنية:**
- `<address length>` — كلاهما 32-bit cells في SPARC/Leon systems.

**الربط بالكود:**

```c
regs = devm_platform_ioremap_resource(ofdev, 0);
if (IS_ERR(regs))
    return PTR_ERR(regs);
```

الـ `devm_platform_ioremap_resource` بيجيب الـ `reg[0]` من الـ DT ويعمله `ioremap` تلقائياً. الـ base address ده بيتقسم على registers بالـ offsets دي:

| Offset | Register | الوظيفة |
|---|---|---|
| `0x00` | `GRGPIO_DATA` | قراءة حالة الـ GPIO pins |
| `0x04` | `GRGPIO_OUTPUT` | كتابة output values |
| `0x08` | `GRGPIO_DIR` | direction control (1=output) |
| `0x0c` | `GRGPIO_IMASK` | interrupt enable mask |
| `0x10` | `GRGPIO_IPOL` | interrupt polarity |
| `0x14` | `GRGPIO_IEDGE` | edge vs level select |
| `0x18` | `GRGPIO_BYPASS` | bypass register |
| `0x20+` | `GRGPIO_IMAP_BASE` | interrupt mapping registers |

---

#### Property: `interrupts`

```
interrupts = <6 2>, <7 2>, <8 2>;
```

**بتعمل إيه:**
بتحدد الـ "underlying" interrupts بتاعة الـ GRGPIO core — يعني الـ IRQ lines اللي بتوصل من الـ core للـ interrupt controller. الـ GRGPIO core ممكن يشارك أكتر من IRQ line واحدة، وكل GPIO line بتتمابق لـ underlying IRQ معين (أو مفيش).

**ملاحظة مهمة:**
الـ `interrupts` property وحدها مش كفاية عشان الـ driver يشتغل بـ IRQ support — لازم يكون فيه `irqmap` property كمان، غير كده الـ driver بيشتغل بدون interrupt support خالص.

**الربط بالكود:**

```c
ret = platform_get_irq(ofdev, lirq->index);
if (ret <= 0) {
    continue; /* skip this gpio line silently */
}
priv->uirqs[lirq->index].uirq = ret;
```

الـ `lirq->index` ده index في الـ `interrupts` array — يعني `index=0` يجيب أول IRQ، `index=1` يجيب التاني، إلخ.

---

#### Property: `nbits`

```
nbits = <8>;
```

**بتعمل إيه:**
بتحدد عدد الـ GPIO lines الفعلية اللي الـ core بيدعمها. لو مش موجودة أو قيمتها غلط، الـ driver بيفترض 32 (الحد الأقصى `GRGPIO_MAX_NGPIO`).

**القيود:**
- لازم تكون > 0
- لازم تكون ≤ 32 (`GRGPIO_MAX_NGPIO`)
- لو خرجت عن النطاق ده، الـ driver بيتجاهلها ويحط 32

**الربط بالكود:**

```c
err = of_property_read_u32(np, "nbits", &prop);
if (err || prop <= 0 || prop > GRGPIO_MAX_NGPIO) {
    gc->ngpio = GRGPIO_MAX_NGPIO; /* default: 32 */
    dev_dbg(dev, "No or invalid nbits property: assume %d\n", gc->ngpio);
} else {
    gc->ngpio = prop;
}
```

الـ `gc->ngpio` ده اللي بيحدد كام GPIO line الـ `gpio_chip` بيعرضها لباقي الـ kernel.

---

#### Property: `irqmap`

```
irqmap = <0 0 0xffffffff 1 0xffffffff ...>;
```

**بتعمل إيه:**
ده array بيحدد لكل GPIO line انها متربطة بأنهي underlying interrupt (index في الـ `interrupts` array)، أو `0xffffffff` يعني مفيش IRQ لهذه الـ line. لو الـ property دي مش موجودة، الـ driver مش بيوفر أي interrupt support خالص.

**البنية:**
- طول الـ array لازم يكون على الأقل = `ngpio`
- كل entry هي:
  - index في الـ `interrupts` property (0-based) → GPIO line دي متربطة بـ underlying IRQ ده
  - `0xffffffff` → الـ line دي مفيش ليها IRQ

**مثال عملي:**

```
interrupts = <6 2>, <7 2>;   /* underlying irq index 0 و 1 */
nbits = <4>;
irqmap = <0 0 0xffffffff 1>;
/*
 * GPIO[0] → underlying irq[0] (irq رقم 6)
 * GPIO[1] → underlying irq[0] (irq رقم 6)
 * GPIO[2] → لا يوجد IRQ
 * GPIO[3] → underlying irq[1] (irq رقم 7)
 */
```

**الربط بالكود:**

```c
irqmap = (s32 *)of_get_property(np, "irqmap", &size);
if (irqmap) {
    if (size < gc->ngpio) {
        dev_err(...); return -EINVAL;
    }

    /* إنشاء irq domain */
    priv->domain = irq_domain_create_linear(...);

    for (i = 0; i < gc->ngpio; i++) {
        lirq = &priv->lirqs[i];
        lirq->index = irqmap[i]; /* -1 (0xffffffff as s32) = no irq */

        if (lirq->index < 0)
            continue;

        ret = platform_get_irq(ofdev, lirq->index);
        priv->uirqs[lirq->index].uirq = ret;
    }
}
```

لاحظ إن `0xffffffff` لما بيتقرأ كـ `s32` بيبقى `-1`، وده اللي الـ driver بيستخدمه كـ sentinel value.

---

### فئة 2: Registration Functions

#### `grgpio_probe`

```c
static int grgpio_probe(struct platform_device *ofdev)
```

**بتعمل إيه:**
ده الـ entry point الرئيسي للـ driver. بيعمل:
1. allocate وينشئ الـ `grgpio_priv` structure
2. يعمل `ioremap` للـ register space من `reg` property
3. ينشئ الـ `gpio_generic_chip` بـ DATA/OUTPUT/DIR registers
4. يقرأ `nbits` ويضبط `gc->ngpio`
5. لو فيه `irqmap`، ينشئ الـ `irq_domain` ويربط كل GPIO line بـ underlying IRQ

**Parameters:**
- `ofdev` — الـ `platform_device` اللي جاي من OF/DT subsystem

**Return:**
- `0` — نجح
- `-ENOMEM` — فشل الـ allocation
- `-EINVAL` — `irqmap` أقصر من `ngpio` أو مقدرش ينشئ الـ `irq_domain`
- `PTR_ERR(regs)` — فشل الـ `ioremap`

**Pseudocode Flow:**

```
grgpio_probe():
    priv = devm_kzalloc()
    regs = devm_platform_ioremap_resource()

    config = {
        .dat = regs + GRGPIO_DATA,
        .set = regs + GRGPIO_OUTPUT,
        .dirout = regs + GRGPIO_DIR,
        .flags = GPIO_GENERIC_BIG_ENDIAN_BYTE_ORDER,
    }
    gpio_generic_chip_init(&priv->chip, &config)

    priv->imask = read(GRGPIO_IMASK)    /* snapshot الـ current mask */

    gc->to_irq = grgpio_to_irq
    gc->ngpio = read "nbits" or default 32

    if "irqmap" exists in DT:
        irq_domain_create_linear(gc->ngpio, &grgpio_irq_domain_ops)
        devm_add_action_or_reset(grgpio_irq_domain_remove)
        for each gpio line:
            lirq->index = irqmap[i]
            if index >= 0:
                uirq->uirq = platform_get_irq(index)

    devm_gpiochip_add_data(gc, priv)
    return 0
```

**Key Details:**
- الـ `GPIO_GENERIC_BIG_ENDIAN_BYTE_ORDER` flag مهم لأن SPARC big-endian
- الـ `gc->base = -1` معناه الـ kernel يختار الـ base number تلقائياً (dynamic allocation)
- كل الـ allocations بـ `devm_*` عشان الـ cleanup تبقى تلقائية

---

### فئة 3: GPIO API Functions

#### `grgpio_to_irq`

```c
static int grgpio_to_irq(struct gpio_chip *gc, unsigned offset)
```

**بتعمل إيه:**
بيحوّل GPIO line offset لـ Linux virtual IRQ number. ده الـ callback اللي الـ GPIO framework بيستخدمه لما حاجة تطلب IRQ لـ GPIO معينة.

**Parameters:**
- `gc` — الـ `gpio_chip` pointer
- `offset` — رقم الـ GPIO line (0 إلى ngpio-1)

**Return:**
- Linux virtual IRQ number (양수) — نجح
- `-ENXIO` — لو الـ offset خارج النطاق أو الـ line مفيش ليها IRQ mapping

**Key Details:**
- بيستخدم `irq_create_mapping` اللي بتشغّل `grgpio_irq_map` كـ side effect لو الـ mapping مش موجودة
- لو `lirq->index < 0` (يعني `irqmap[offset] == 0xffffffff`) بيرجع `-ENXIO` فوراً

```c
static int grgpio_to_irq(struct gpio_chip *gc, unsigned offset)
{
    struct grgpio_priv *priv = gpiochip_get_data(gc);

    if (offset >= gc->ngpio)
        return -ENXIO;

    if (priv->lirqs[offset].index < 0)
        return -ENXIO;

    return irq_create_mapping(priv->domain, offset);
}
```

---

### فئة 4: IRQ Domain Operations

#### `grgpio_irq_map`

```c
static int grgpio_irq_map(struct irq_domain *d, unsigned int irq,
                          irq_hw_number_t hwirq)
```

**بتعمل إيه:**
ده الـ `.map` callback للـ `irq_domain`. بيتشال لما `irq_create_mapping` تتشغّل من `grgpio_to_irq`. مهمته إنه يربط الـ Linux virtual IRQ بالـ GPIO hardware line، ويطلب الـ underlying interrupt من الـ parent interrupt controller لو محدش طلبه قبل كده.

**Parameters:**
- `d` — الـ `irq_domain` بتاع الـ driver
- `irq` — الـ Linux virtual IRQ number المخصص
- `hwirq` — الـ hardware IRQ number = GPIO offset

**Return:**
- `0` — نجح
- `-EINVAL` — الـ `priv` null أو الـ GPIO line مفيش ليها IRQ mapping
- return value من `request_irq` — لو الـ underlying IRQ request فشل

**Key Details:**
- بيستخدم `atomic_fetch_add` على `uirq->refcnt` — الـ underlying IRQ بيتطلب بس لو `refcnt` كان صفر (أول GPIO line تطلبه)
- بيضبط `handle_simple_irq` كـ flow handler
- بيشغّل `irq_set_noprobe` عشان يمنع الـ auto-probing
- **Locking:** بيعمل lock على الـ GPIO generic chip spinlock لما يعدّل `lirq->irq`

**Pseudocode Flow:**

```
grgpio_irq_map(d, irq, hwirq):
    lirq = &priv->lirqs[hwirq]
    if lirq->index < 0: return -EINVAL

    lock(chip_spinlock)
    lirq->irq = irq
    uirq = &priv->uirqs[lirq->index]
    unlock(chip_spinlock)

    if atomic_fetch_add(1, &uirq->refcnt) == 0:
        /* أول GPIO line تستخدم هذا الـ underlying IRQ */
        request_irq(uirq->uirq, grgpio_irq_handler, ...)

    irq_set_chip_data(irq, priv)
    irq_set_chip_and_handler(irq, &grgpio_irq_chip, handle_simple_irq)
    irq_set_noprobe(irq)
```

---

#### `grgpio_irq_unmap`

```c
static void grgpio_irq_unmap(struct irq_domain *d, unsigned int irq)
```

**بتعمل إيه:**
عكس الـ `grgpio_irq_map`. بيبحث عن الـ GPIO line المرتبطة بالـ IRQ ده، بيعمل mask لها، ولو ده آخر مستخدم للـ underlying IRQ (refcnt وصل صفر) بيعمل `free_irq`.

**Parameters:**
- `d` — الـ `irq_domain`
- `irq` — الـ Linux virtual IRQ اللي هيتشال mapping منه

**Key Details:**
- بيعمل linear scan على كل الـ GPIO lines عشان يلاقي الـ `lirq->irq == irq`
- `WARN_ON(index < 0)` — لو معدتش تلاقي GPIO line بيبقى ده bug
- **Locking:** معظم العمل بيحصل تحت الـ chip spinlock، بس `free_irq` بيتشال برا الـ lock عشان ممكن يبقى blocking

```c
/* Reference counting pattern */
if (atomic_dec_and_test(&uirq->refcnt)) {
    /* آخر GPIO line تستخدم هذا underlying IRQ */
    gpio_generic_chip_unlock_irqrestore(&priv->chip, flags);
    free_irq(uirq->uirq, priv);
    return;
}
```

---

### فئة 5: IRQ Chip Operations

#### `grgpio_irq_set_type`

```c
static int grgpio_irq_set_type(struct irq_data *d, unsigned int type)
```

**بتعمل إيه:**
بيضبط الـ interrupt trigger type لـ GPIO line معينة عن طريق تعديل الـ `GRGPIO_IPOL` (polarity) و`GRGPIO_IEDGE` (edge/level) registers.

**Parameters:**
- `d` — الـ `irq_data` للـ GPIO IRQ
- `type` — الـ trigger type المطلوب

**الأنواع المدعومة:**

| Type | IPOL bit | IEDGE bit | المعنى |
|---|---|---|---|
| `IRQ_TYPE_LEVEL_LOW` | 0 | 0 | active low level |
| `IRQ_TYPE_LEVEL_HIGH` | 1 | 0 | active high level |
| `IRQ_TYPE_EDGE_FALLING` | 0 | 1 | falling edge |
| `IRQ_TYPE_EDGE_RISING` | 1 | 1 | rising edge |

**Return:**
- `0` — نجح
- `-EINVAL` — type مش مدعوم (مثلاً `IRQ_TYPE_EDGE_BOTH` مش supported)

**Key Details:**
- بيستخدم `guard(gpio_generic_lock_irqsave)` — الـ C scoped locking من kernel 6.x
- بيعمل read-modify-write على الـ registers عشان يحافظ على باقي الـ bits

---

#### `grgpio_irq_mask`

```c
static void grgpio_irq_mask(struct irq_data *d)
```

**بتعمل إيه:**
بيعمل mask للـ interrupt الخاصة بـ GPIO line معينة عن طريق clear الـ bit المقابل في `GRGPIO_IMASK` register. بعدين بيستدعي `gpiochip_disable_irq` للإشعار بالـ irqchip framework.

**Key Details:**
- الترتيب مهم: أولاً mask الـ hardware، بعدين `gpiochip_disable_irq`
- بيستخدم `scoped_guard` عشان الـ lock يتفك تلقائياً
- بيعمل shadow register update في `priv->imask`

---

#### `grgpio_irq_unmask`

```c
static void grgpio_irq_unmask(struct irq_data *d)
```

**بتعمل إيه:**
عكس الـ mask — بيشغّل الـ interrupt للـ GPIO line. بيستدعي `gpiochip_enable_irq` الأول، بعدين بيعمل set للـ bit في `GRGPIO_IMASK`.

**Key Details:**
- الترتيب عكس الـ mask: `gpiochip_enable_irq` الأول، بعدين hardware unmask
- ده عشان نضمن إن الـ IRQ framework جاهز قبل ما الـ hardware يبدأ يطلع interrupts

---

### فئة 6: IRQ Handler

#### `grgpio_irq_handler`

```c
static irqreturn_t grgpio_irq_handler(int irq, void *dev)
```

**بتعمل إيه:**
ده الـ ISR الرئيسي اللي بيتسجل على الـ underlying IRQs. لما underlying IRQ يحصل، الـ handler ده بيعمل loop على كل الـ GPIO lines اللي:
1. ليها interrupt مفعّلة (`imask` bit set)
2. ليها valid mapping (`lirq->index >= 0`)
3. مربوطة بالـ underlying IRQ ده

وبيستدعي `generic_handle_irq` لكل واحدة منهم.

**Parameters:**
- `irq` — الـ underlying Linux IRQ number اللي اتشغّل
- `dev` — الـ `grgpio_priv` pointer (registered كـ `dev_id`)

**Return:**
- `IRQ_HANDLED` — دايماً (حتى لو مفيش GPIO line matched، بس بيطبع warning)

**Key Details:**
- بيشتغل تحت الـ chip spinlock طول الوقت
- لو مفيش أي GPIO line matched → `dev_warn` — ده يعني bug في الـ irqmap أو unexpected IRQ
- الـ `generic_handle_irq` بيتشال جوه الـ lock — ده okay لأن `handle_simple_irq` مش بياخد locks تانية

**Pseudocode Flow:**

```
grgpio_irq_handler(irq, dev):
    lock(chip_spinlock)
    match = 0

    for i in range(ngpio):
        lirq = &priv->lirqs[i]
        if imask[i] AND lirq->index >= 0 AND uirqs[lirq->index].uirq == irq:
            generic_handle_irq(lirq->irq)
            match = 1

    if not match:
        dev_warn("No gpio line matched irq %d")

    return IRQ_HANDLED
```

---

### فئة 7: Helper Functions

#### `grgpio_set_imask`

```c
static void grgpio_set_imask(struct grgpio_priv *priv, unsigned int offset,
                             int val)
```

**بتعمل إيه:**
helper function بتعدّل الـ interrupt mask shadow register (`priv->imask`) وبتكتب القيمة الجديدة على الـ `GRGPIO_IMASK` hardware register.

**Parameters:**
- `priv` — الـ private driver data
- `offset` — GPIO line number (0-based)
- `val` — 1 لتفعيل الـ interrupt، 0 لإلغائه

**Key Details:**
- لازم تتشال جوه الـ chip spinlock دايماً
- بتستخدم shadow register (`priv->imask`) عشان تعمل read-modify-write بدون قراءة من الـ hardware
- الـ shadow register مهم عشان الـ hardware بعض الأوقات مش بيرجع نفس القيمة المكتوبة

---

#### `grgpio_irq_domain_remove`

```c
static void grgpio_irq_domain_remove(void *data)
```

**بتعمل إيه:**
ده `devm` action callback بيتسجّل عن طريق `devm_add_action_or_reset`. بيتشال تلقائياً لما الـ device يتعمله `unbind` أو `remove`، وبيزيل الـ irq domain.

**Parameters:**
- `data` — الـ `irq_domain` pointer

**Key Details:**
- مش بيتشال يدوياً — الـ `devm` framework بيديره
- بيستخدم `irq_domain_remove` اللي بيشيل الـ domain من الـ global list

---

### الـ irq_chip و irq_domain_ops Structures

```c
static const struct irq_chip grgpio_irq_chip = {
    .name       = "grgpio",
    .irq_mask   = grgpio_irq_mask,
    .irq_unmask = grgpio_irq_unmask,
    .irq_set_type = grgpio_irq_set_type,
    .flags = IRQCHIP_IMMUTABLE,
    GPIOCHIP_IRQ_RESOURCE_HELPERS, /* يضيف irq_request_resources/irq_release_resources */
};
```

الـ `IRQCHIP_IMMUTABLE` flag معناها إن الـ `irq_chip` structure نفسها مش هتتعدّل بعد التسجيل — ده protection ضد بعض الـ kernel mechanisms اللي كانت بتعدّل الـ irq_chip in-place.

```c
static const struct irq_domain_ops grgpio_irq_domain_ops = {
    .map   = grgpio_irq_map,
    .unmap = grgpio_irq_unmap,
};
```

الـ driver بيستخدم **linear irq domain** — مناسب لعدد GPIO lines محدود (≤32) والـ hwirq numbers متتالية من 0.

---

### الـ Data Structures — تعمق

```c
struct grgpio_priv {
    struct gpio_generic_chip chip; /* يلف الـ gpio_chip + lock + big-endian ops */
    void __iomem *regs;            /* base address بعد الـ ioremap */
    struct device *dev;

    u32 imask;                     /* shadow لـ GRGPIO_IMASK register */
    struct irq_domain *domain;     /* linear irq domain للـ GPIO IRQs */

    struct grgpio_uirq uirqs[32]; /* الـ underlying IRQs من parent controller */
    struct grgpio_lirq lirqs[32]; /* الـ per-GPIO-line IRQ info */
};

struct grgpio_uirq {
    atomic_t refcnt; /* كام GPIO line بتستخدم هذا underlying IRQ */
    u8 uirq;         /* Linux IRQ number للـ underlying interrupt */
};

struct grgpio_lirq {
    s8 index; /* index في uirqs[] أو -1 لو مفيش IRQ */
    u8 irq;   /* Linux virtual IRQ المخصص لهذه الـ GPIO line */
};
```

**العلاقة بين الـ structures:**

```
GPIO Line [i]
    └─► lirqs[i].index ──────────────► uirqs[index].uirq ──► parent IRQ controller
    └─► lirqs[i].irq   ──► Linux virtual IRQ (exposed to consumers)
```

المشترك في نفس الـ underlying IRQ:
```
GPIO[0] ──► lirqs[0].index = 0 ─┐
GPIO[1] ──► lirqs[1].index = 0 ─┴──► uirqs[0].uirq = irq#6, refcnt=2
GPIO[3] ──► lirqs[3].index = 1 ──────► uirqs[1].uirq = irq#7, refcnt=1
GPIO[2] ──► lirqs[2].index = -1  (مفيش IRQ)
```
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs — المسارات والقراءة

**الـ GPIO subsystem** بيوفر entries في debugfs تخص كل `gpio_chip` مسجّل.

```bash
# اتأكد إن debugfs متماونت
mount | grep debugfs
# لو مش موجود:
mount -t debugfs none /sys/kernel/debug

# اعرض كل الـ GPIO chips المسجّلة
cat /sys/kernel/debug/gpio

# مثال على الـ output:
# gpiochip0: GPIOs 0-31, parent: platform/80000b00.gpio, /proc/device-tree/...:
#  gpio-0  (                    ) in  lo IRQ
#  gpio-4  (                    ) out hi
```

الـ entry ده بيعرض:
- رقم الـ gpio ونطاق الـ chip
- اسم الـ parent device
- اتجاه كل line (in/out)
- القيمة الحالية (hi/lo)
- لو فيه IRQ مربوط

```bash
# مشاهدة تفاصيل irq domain الخاص بـ grgpio
cat /sys/kernel/debug/irq/domains/grgpio*
```

---

#### 2. sysfs — المسارات والقيم

```bash
# اعرض كل الـ GPIO chips
ls /sys/class/gpio/

# Export gpio line معينة (مثلاً gpio رقم 5)
echo 5 > /sys/class/gpio/export

# اقرأ الاتجاه
cat /sys/class/gpio/gpio5/direction   # in أو out

# اقرأ القيمة
cat /sys/class/gpio/gpio5/value       # 0 أو 1

# اكتب قيمة (لازم تبقى output أول)
echo out > /sys/class/gpio/gpio5/direction
echo 1   > /sys/class/gpio/gpio5/value

# اعرض الـ edge interrupt setting
cat /sys/class/gpio/gpio5/edge        # none / rising / falling / both

# بعد الانتهاء
echo 5 > /sys/class/gpio/unexport
```

**الـ sysfs entries الأساسية لأي gpiochip:**

| Entry | المحتوى |
|---|---|
| `/sys/class/gpio/gpiochipN/base` | أول رقم GPIO في الـ chip |
| `/sys/class/gpio/gpiochipN/ngpio` | عدد الـ lines |
| `/sys/class/gpio/gpiochipN/label` | اسم الـ chip (مثلاً اسم الـ DT node) |

---

#### 3. ftrace — الـ Tracepoints والـ Events

```bash
# شوف الـ gpio events المتاحة
ls /sys/kernel/debug/tracing/events/gpio/

# الـ events المهمة:
#   gpio_direction    — لما بيتغير اتجاه line
#   gpio_value        — لما بيتقرأ/يتكتب قيمة

# فعّل الـ gpio events كلها
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable

# فعّل tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل الكود اللي بتحقق منه، بعدين اقرأ الـ trace
cat /sys/kernel/debug/tracing/trace

# مثال على output:
# gpio_direction: gpio=5 in=0   <- set to output
# gpio_value:     gpio=5 get=0 value=1

# وقفّل الـ tracing
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo 0 > /sys/kernel/debug/tracing/events/gpio/enable

# لو حابب تتبع function calls جوه الـ driver نفسه
echo grgpio_* > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

---

#### 4. printk و dynamic debug

**الـ grgpio driver** بيستخدم `dev_dbg` و `dev_err` و `dev_warn` و `dev_info`.

```bash
# فعّل dynamic debug لـ driver الـ grgpio بالكامل
echo 'module grgpio +p' > /sys/kernel/debug/dynamic_debug/control

# أو بالـ source file
echo 'file drivers/gpio/gpio-grgpio.c +p' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ messages الطالعة
dmesg -w | grep -i grgpio

# لو حابب تفعّل كل gpio debug messages
echo 'module gpio-grgpio +pflmt' > /sys/kernel/debug/dynamic_debug/control
# p = print, f = function name, l = line, m = module, t = thread
```

الـ messages الـ `dev_dbg` في الكود:

```c
/* في grgpio_probe — بتطلع لو nbits مش موجودة في DT */
dev_dbg(dev, "No or invalid nbits property: assume %d\n", gc->ngpio);

/* في grgpio_irq_map — بتطلع لما بيتعمل irq mapping */
dev_dbg(priv->dev, "Mapping irq %d for gpio line %d\n", irq, offset);
```

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الغرض |
|---|---|
| `CONFIG_GPIO_GRGPIO` | الـ driver نفسه |
| `CONFIG_DEBUG_GPIO` | تفعيل GPIO debugging |
| `CONFIG_GPIOLIB_IRQCHIP` | دعم IRQ chip في gpiolib |
| `CONFIG_GENERIC_IRQ_DEBUGFS` | irq domain info في debugfs |
| `CONFIG_IRQ_DOMAIN_DEBUG` | debug الـ irq domain mappings |
| `CONFIG_PROVE_LOCKING` | اكتشاف deadlocks في spinlocks |
| `CONFIG_DEBUG_SPINLOCK` | debug الـ spinlock operations |
| `CONFIG_DYNAMIC_DEBUG` | تفعيل dev_dbg dynamically |
| `CONFIG_KALLSYMS` | أسماء رمزية في stack traces |
| `CONFIG_STACKTRACE` | dump_stack support |

```bash
# تحقق من الـ config الحالية
zcat /proc/config.gz | grep -E 'GPIO|IRQ_DOMAIN'
```

---

#### 6. أدوات خاصة بالـ Subsystem

```bash
# gpiodetect — بيعرض كل الـ GPIO chips
gpiodetect
# مثال output:
# gpiochip0 [/proc/device-tree/gpio@80000b00] (32 lines)

# gpioinfo — تفاصيل كل line
gpioinfo gpiochip0

# gpioget — يقرأ قيمة line
gpioget gpiochip0 5

# gpioset — يكتب قيمة
gpioset gpiochip0 5=1

# gpiomon — يراقب تغيرات line
gpiomon gpiochip0 5

# فحص الـ IRQ stats
cat /proc/interrupts | grep grgpio

# فحص الـ IRQ affinity
cat /proc/irq/<irq_number>/affinity_hint
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ Kernel | المعنى | الحل |
|---|---|---|
| `failed to initialize the generic GPIO chip` | فشل `gpio_generic_chip_init` | تحقق من الـ register mapping وصحة `config.sz` |
| `irqmap shorter than ngpio (%d < %d)` | الـ `irqmap` array في DT أقصر من عدد الـ GPIO lines | صحّح الـ DT: اعمل `irqmap` بنفس عدد الـ `ngpio` |
| `Could not add irq domain` | فشل `irq_domain_create_linear` | تحقق من الـ fwnode وتوفر الـ IRQ resources |
| `Could not request underlying irq %d` | فشل `request_irq` على الـ parent IRQ | تحقق إن الـ IRQ مش محجوز من driver تاني |
| `No gpio line matched irq %d` | الـ handler اتنفّذ بس مفيش line مرتبطة بالـ IRQ ده | مشكلة في الـ `irqmap` أو الـ `GRGPIO_IMASK` |
| `Could not add gpiochip` | فشل `devm_gpiochip_add_data` | تحقق من تعارض الـ GPIO base numbers |
| `No or invalid nbits property: assume 32` | مش error بالأساس — warning بس | أضف `nbits` في DT لو الـ hardware أقل من 32 line |
| `Could not add irq domain` | الـ fwnode مش صالح | تحقق من `of_node` في الـ DT |

---

#### 8. مواضع dump_stack() و WARN_ON()

الـ `WARN_ON` الموجودة في الكود:

```c
/* في grgpio_irq_unmap — لو مفيش lirq مرتبطة بالـ irq اللي بيتفك */
WARN_ON(index < 0);
```

نقاط استراتيجية تفيدك تحطّ فيها debug assertions:

```c
/* في grgpio_probe — بعد قراءة IMASK تحقق إن القيمة منطقية */
WARN_ON(priv->imask & ~((1U << gc->ngpio) - 1));

/* في grgpio_irq_handler — لو الـ handler اتنادى وmask = 0 */
WARN_ON(!priv->imask);

/* في grgpio_set_imask — تحقق إن offset في النطاق */
WARN_ON(offset >= GRGPIO_MAX_NGPIO);
```

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State مطابق لحالة الـ Kernel

```bash
# اقرأ رقم الـ GPIO chip base
cat /sys/class/gpio/gpiochip0/base   # مثلاً 0

# قارن عدد الـ lines بالـ hardware spec
cat /sys/class/gpio/gpiochip0/ngpio  # لازم يطابق nbits في DT أو 32

# تحقق من الـ IRQ mapping
cat /proc/interrupts | grep grgpio
# لازم يظهر كل الـ underlying IRQs اللي في DT

# شوف الـ irq domain
cat /sys/kernel/debug/irq/domains/
ls /sys/kernel/debug/irq/domains/ | grep grgpio
```

---

#### 2. Register Dump — قراءة الـ Registers مباشرة

الـ GRGPIO registers على أساس الـ base address من DT (مثلاً `0x80000b00`):

| Offset | اسم الـ Register | الوصف |
|--------|----------------|--------|
| `0x00` | `GRGPIO_DATA` | قيمة الـ GPIO pins الحالية (read) |
| `0x04` | `GRGPIO_OUTPUT` | output register (write) |
| `0x08` | `GRGPIO_DIR` | اتجاه كل pin (1=output) |
| `0x0C` | `GRGPIO_IMASK` | interrupt mask |
| `0x10` | `GRGPIO_IPOL` | interrupt polarity |
| `0x14` | `GRGPIO_IEDGE` | edge/level select |
| `0x18` | `GRGPIO_BYPASS` | bypass register |
| `0x20+` | `GRGPIO_IMAP` | interrupt map entries |

```bash
# استخدم devmem2 لقراءة الـ registers (x = hex, w = 32-bit word)
BASE=0x80000b00

devmem2 $((BASE + 0x00)) w   # DATA register
devmem2 $((BASE + 0x04)) w   # OUTPUT register
devmem2 $((BASE + 0x08)) w   # DIR register
devmem2 $((BASE + 0x0C)) w   # IMASK register
devmem2 $((BASE + 0x10)) w   # IPOL register
devmem2 $((BASE + 0x14)) w   # IEDGE register

# مثال على output:
# /dev/mem opened.
# Memory mapped at address 0xb6f23000.
# Read at address 0x80000b0C (0xb6f23b0c): 0x000000FF
# يعني أول 8 lines interrupt mask مفعّل
```

```bash
# لو devmem2 مش موجود، استخدم /dev/mem مع dd
dd if=/dev/mem bs=4 count=1 skip=$((BASE/4 + 3)) 2>/dev/null | xxd
```

**تفسير قيم الـ DIR register:**
```
DIR = 0x000000FF  →  pins 0-7 outputs, pins 8-31 inputs
DIR = 0x00000000  →  كل الـ pins inputs
DIR = 0xFFFFFFFF  →  كل الـ pins outputs
```

---

#### 3. Logic Analyzer / Oscilloscope

لتشخيص مشاكل الـ GRGPIO hardware:

- **تحقق من الـ Clock:** الـ GRGPIO جزء من GRLIB AMBA bus — تأكد إن الـ AMBA clock شغّال بالتردد الصح
- **اتبع الـ IRQ lines:** اربط الـ oscilloscope على pin الـ interrupt الخارجي وقارن بالـ `IMASK` و `IPOL`
- **راقب الـ GPIO toggles:** لو بتكتب 1 على pin، لازم تشوفه على الـ scope في نفس اللحظة
- **فحص الـ edge detection:** اختبر rising/falling edges وتأكد إن الـ `IEDGE` register مضبوط صح

**نصائح عملية:**
- الـ GRGPIO big-endian بالـ byte order — لو الـ values غريبة، اتحقق من الـ endianness في الـ driver flag `GPIO_GENERIC_BIG_ENDIAN_BYTE_ORDER`
- لو الـ GPIO مش بيستجاوش، تحقق من الـ AMBA PnP config إن الـ core registered صح

---

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في الـ Kernel Log

| المشكلة | النمط في dmesg | التشخيص |
|---|---|---|
| الـ base address غلط في DT | `failed to initialize the generic GPIO chip` + kernel oops | صحّح `reg` في DT |
| الـ IRQ line مش واصلة | `Could not request underlying irq` | تحقق من الـ AMBA PnP interrupt assignment |
| `nbits` أكبر من الـ hardware الفعلي | قراءة pins زيادة تعطي values عشوائية | صحّح `nbits` في DT |
| الـ IMASK بيتكتب بس IRQ مش بيطلع | `No gpio line matched irq` | تحقق من `irqmap` في DT |
| big-endian mismatch | الـ DIR/OUTPUT registers بيتكتبوا عكس المتوقع | تأكد من `GPIO_GENERIC_BIG_ENDIAN_BYTE_ORDER` flag |
| GPIO output مش بيتغيّر | قراءة DATA مختلفة عن OUTPUT | احتمال الـ pin مش configured كـ output — فحص DIR |

---

#### 5. Device Tree Debugging

```bash
# اعرض الـ DT node الخاص بـ grgpio
find /proc/device-tree -name "GAISLER_GPIO" -o -name "01_01a" 2>/dev/null

# أو ابحث بالـ compatible / name
for node in /proc/device-tree/**; do
  [ -f "$node/name" ] && grep -q "GAISLER_GPIO\|01_01a" "$node/name" 2>/dev/null && echo "$node"
done

# اقرأ الـ properties
NODE=/proc/device-tree/gpio@80000b00

hexdump -C $NODE/reg           # base address وsize
hexdump -C $NODE/interrupts    # interrupt numbers
hexdump -C $NODE/nbits         # عدد الـ lines
hexdump -C $NODE/irqmap        # irq mapping array

# تحقق إن الـ nbits منطقية (لازم <= 32)
od -t u4 $NODE/nbits

# تحقق من الـ irqmap — كل entry أما index صالح أو 0xFFFFFFFF
od -t x4 $NODE/irqmap
```

**مثال على DT صحيح:**
```dts
gpio0: gpio@80000b00 {
    compatible = "GAISLER_GPIO";  /* أو "01_01a" */
    reg = <0x80000b00 0x100>;
    interrupts = <4 0>, <5 0>;
    nbits = <8>;
    irqmap = <0 0 1 1 0xffffffff 0xffffffff 0xffffffff 0xffffffff>;
};
```

```bash
# تحقق إن الـ driver التقى بالـ node
dmesg | grep grgpio
# المتوقع:
# grgpio 80000b00.gpio: regs=0x..., base=-1, ngpio=8, irqs=on
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للنسخ والتشغيل

```bash
#!/bin/bash
# === GRGPIO Full Debug Script ===

BASE_ADDR=0x80000b00   # غيّر ده حسب الـ DT الخاص بيك

echo "=== 1. Kernel dmesg for grgpio ==="
dmesg | grep -i grgpio

echo ""
echo "=== 2. GPIO chips registered ==="
gpiodetect 2>/dev/null || cat /sys/kernel/debug/gpio

echo ""
echo "=== 3. GPIO chip info ==="
for chip in /sys/class/gpio/gpiochip*; do
    echo "Chip: $chip"
    echo "  base:  $(cat $chip/base)"
    echo "  ngpio: $(cat $chip/ngpio)"
    echo "  label: $(cat $chip/label)"
done

echo ""
echo "=== 4. IRQ stats for grgpio ==="
cat /proc/interrupts | grep -E "grgpio|GPIO"

echo ""
echo "=== 5. Register dump ==="
for offset in 0x00 0x04 0x08 0x0C 0x10 0x14 0x18; do
    name=""
    case $offset in
        0x00) name="DATA  " ;;
        0x04) name="OUTPUT" ;;
        0x08) name="DIR   " ;;
        0x0C) name="IMASK " ;;
        0x10) name="IPOL  " ;;
        0x14) name="IEDGE " ;;
        0x18) name="BYPASS" ;;
    esac
    addr=$(printf "0x%08x" $((BASE_ADDR + offset)))
    val=$(devmem2 $addr w 2>/dev/null | grep "Read at" | awk '{print $NF}')
    echo "  $name @ $addr = $val"
done

echo ""
echo "=== 6. Dynamic debug activation ==="
echo 'module grgpio +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo "Dynamic debug enabled — watch dmesg"

echo ""
echo "=== 7. ftrace gpio events ==="
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
echo "ftrace enabled — run your test, then:"
echo "  cat /sys/kernel/debug/tracing/trace"
```

---

#### كيفية تفسير الـ Output

**مثال: قراءة IMASK = 0x0000000F**
```
Bits 0-3 مفعّلين → أول 4 GPIO lines ليها interrupts مفعّلة
Bits 4-31 = 0   → باقي الـ lines مش ليها interrupt
```

**مثال: قراءة IPOL = 0x00000005، IEDGE = 0x00000000**
```
IPOL bit 0 = 1, IEDGE bit 0 = 0  → GPIO 0: IRQ_TYPE_LEVEL_HIGH
IPOL bit 2 = 1, IEDGE bit 2 = 0  → GPIO 2: IRQ_TYPE_LEVEL_HIGH
باقي الـ bits = 0                 → IRQ_TYPE_LEVEL_LOW
```

**مثال: dmesg ناجح بعد load الـ driver**
```
[    2.345678] grgpio 80000b00.gpio: regs=0xc08f2000, base=-1, ngpio=32, irqs=on
```
- `base=-1` يعني الـ kernel اختار الـ base تلقائياً (GPIO_DYNAMIC_BASE)
- `irqs=on` يعني الـ irq domain اتعمل بنجاح
- لو `irqs=off` — يعني الـ `irqmap` مش موجودة في DT
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: LEON3 Industrial Gateway — GPIO Interrupt لا بيشتغل

#### العنوان
**GPIO IRQ Mapping خاطئ على LEON3-based Industrial Gateway**

#### السياق
شركة بتصنع industrial gateway بيشتغل على LEON3 SPARC processor (Cobham Gaisler UT699) — مستخدم في أنظمة التحكم الصناعي. الـ gateway بيستقبل إشارات من sensors خارجية عن طريق GPIO lines، وكل line محتاجة تعمل interrupt عشان الـ response يكون fast. الـ FPGA bitstream فيه GRGPIO core معمول بـ 16 GPIO line، كل line ليها IRQ منفصل.

#### المشكلة
بعد boot، الـ GPIO interrupts مش بتتفعّل خالص. الـ driver بيـ register بس الـ ISR مش بيتنادى حتى لما الـ sensor يعمل trigger. الـ `/proc/interrupts` مش بيبيّن أي count على الـ GPIO IRQ lines.

#### التحليل
الـ binding بيقول:

```
- irqmap : An array with an index for each gpio line.
           An index is either a valid index into the interrupts property array,
           or 0xffffffff that indicates no irq for that line.
           Driver provides no interrupt support if not present.
```

الـ DT node كان كده:

```dts
/* خاطئ — irqmap غايبة خالص */
gpio0: gpio@80000900 {
    compatible = "GAISLER_GPIO";
    reg = <0x80000900 0x100>;
    interrupts = <4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19>;
    nbits = <16>;
    /* irqmap مش موجودة!! */
};
```

الـ driver بيشوف إن `irqmap` property غايبة فبيـ disable interrupt support كلها — حتى لو `interrupts` property موجودة وصح. الـ GPIO lines بتشتغل كـ polled فقط.

#### الحل

```dts
/* صح — irqmap محددة لكل line */
gpio0: gpio@80000900 {
    compatible = "GAISLER_GPIO";
    reg = <0x80000900 0x100>;
    interrupts = <4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19>;
    nbits = <16>;
    /*
     * كل index بيشاور على position في interrupts array
     * line 0 → interrupts[0]=4, line 1 → interrupts[1]=5, ...
     */
    irqmap = <0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15>;
};
```

التحقق بعد الحل:

```bash
# تأكد إن الـ interrupts اتـ register صح
cat /proc/interrupts | grep gpio

# اختبار GPIO interrupt يدوياً
gpio_value_toggle /dev/gpio0 line5
```

#### الدرس المستفاد
**الـ `irqmap` property مش optional من ناحية وظيفية** — لو غايبة، الـ driver بيـ silently بيعطّل كل interrupt support. لازم دايماً تتأكد منها حتى لو `interrupts` property موجودة وصح.

---

### السيناريو 2: Space System RTEMS — عدد GPIO Lines خاطئ

#### العنوان
**`nbits` Mismatch يسبب Memory Corruption في نظام فضائي**

#### السياق
نظام تحكم في satellite subsystem بيشتغل على LEON4FT (fault-tolerant SPARC) مع RTEMS RTOS. الـ FPGA design فيه GRGPIO core معمول بـ 24 GPIO line بس الـ DT node مش بيحدد `nbits` خالص. النظام بيتحكم في deployment mechanisms وهو تطبيق safety-critical.

#### المشكلة
الـ driver بيـ allocate arrays بناءً على عدد الـ GPIO lines. بدون `nbits`، الـ driver بيـ assume 32 lines (القيمة الـ default). الـ 8 lines الزيادة دول بيـ read من hardware registers بـ undefined state، وبيـ cause issues في الـ IRQ mapping arrays لما بيكون فيه `irqmap` بتحدد 24 entries بس الـ driver متوقع 32.

```dts
/* خاطئ — nbits غايب، hardware فعلاً 24 lines */
gpio0: gpio@fff00100 {
    compatible = "GAISLER_GPIO";
    reg = <0xfff00100 0x100>;
    interrupts = <6 7 8>;
    /* nbits غايبة → driver هيـ assume 32 */
    irqmap = <0 1 2 0xffffffff 0xffffffff 0xffffffff
              0xffffffff 0xffffffff 0xffffffff 0xffffffff
              0xffffffff 0xffffffff 0xffffffff 0xffffffff
              0xffffffff 0xffffffff 0xffffffff 0xffffffff
              0xffffffff 0xffffffff 0xffffffff 0xffffffff
              0xffffffff 0xffffffff>;
    /* irqmap فيها 24 entries بس driver بيتوقع 32 ← mismatch! */
};
```

الـ binding بيقول صراحة:

```
- nbits : The number of gpio lines.
          If not present driver assumes 32 lines.
```

#### التحليل
الـ driver بيـ iterate على 32 GPIO lines. لما بيوصل لـ `irqmap[24]` إلى `irqmap[31]`، الـ array بتنتهي وبيـ read ما بعدها في memory — undefined behavior. في RTEMS على LEON4FT، ده ممكن يـ trigger memory protection fault أو يـ corrupt adjacent data structures.

#### الحل

```dts
/* صح — nbits محددة بوضوح */
gpio0: gpio@fff00100 {
    compatible = "GAISLER_GPIO";
    reg = <0xfff00100 0x100>;
    interrupts = <6 7 8>;
    nbits = <24>; /* تطابق الـ FPGA hardware design */
    irqmap = <0 1 2
              0xffffffff 0xffffffff 0xffffffff
              0xffffffff 0xffffffff 0xffffffff
              0xffffffff 0xffffffff 0xffffffff
              0xffffffff 0xffffffff 0xffffffff
              0xffffffff 0xffffffff 0xffffffff
              0xffffffff 0xffffffff 0xffffffff
              0xffffffff 0xffffffff 0xffffffff>;
    /* 24 entries بالظبط تطابق nbits */
};
```

التحقق من الـ FPGA hardware:

```bash
# اقرأ GRGPIO status register عشان تتأكد من عدد الـ lines الفعلي
# GRGPIO Capability Register offset 0x18 بيبيّن NLINES field
devmem 0xfff00118 32
# Bits [4:0] = number of GPIO lines - 1
```

#### الدرس المستفاد
في safety-critical systems، **دايماً حدد `nbits` صراحة** حتى لو الـ default هو 32. الـ mismatch بين `nbits` وعدد entries في `irqmap` ممكن يـ cause subtle memory bugs صعبة تتـ debug في real hardware.

---

### السيناريو 3: GRLIB SoC Custom Board Bring-up — اسم الـ Compatible خاطئ

#### العنوان
**Driver لا يـ Probe بسبب `compatible` String غلط**

#### السياق
فريق bring-up بيشغّل custom board مبنية على GRLIB FPGA platform. الـ board مستخدمة في industrial control system. المهندس بيكتب الـ DT من scratch بالاعتماد على GRLIB GRIP manual.

#### المشكلة
بعد boot، مفيش `/dev/gpiochip*` بيظهر خالص. الـ kernel log بيبيّن:

```
[    1.234567] OF: no driver found for compatible: "gaisler,grgpio"
```

الـ driver مش بيـ probe خالص.

#### التحليل
الـ binding بيقول صراحة:

```
- name : Should be "GAISLER_GPIO" or "01_01a"
```

دول هم الـ compatible strings اللي الـ driver بيـ match عليها. الـ convention في GRLIB OpenFirmware DT هو إن الـ compatible بييجي من الـ AMBA plug&play vendor/device ID:

- `GAISLER_GPIO` → الـ human-readable form
- `01_01a` → الـ hex vendor/device ID form (vendor `01` = Gaisler, device `01a` = GRGPIO)

المهندس كتب:

```dts
/* خاطئ */
gpio0: gpio@80000b00 {
    compatible = "gaisler,grgpio"; /* Linux-style compatible — مش صح هنا */
    reg = <0x80000b00 0x100>;
    interrupts = <3>;
    nbits = <8>;
};
```

#### الحل

```dts
/* صح — استخدم الـ compatible strings المحددة في الـ binding */
gpio0: gpio@80000b00 {
    compatible = "GAISLER_GPIO", "01_01a"; /* الاتنين عشان تضمن الـ match */
    reg = <0x80000b00 0x100>;
    interrupts = <3>;
    nbits = <8>;
    irqmap = <0 0xffffffff 0xffffffff 0xffffffff
              0xffffffff 0xffffffff 0xffffffff 0xffffffff>;
};
```

التحقق:

```bash
# تأكد إن الـ driver اتـ probe
dmesg | grep -i gpio
ls /sys/bus/platform/drivers/grgpio/
ls /dev/gpiochip*

# تأكد من الـ compatible matching
cat /sys/bus/platform/devices/80000b00.gpio/of_node/compatible
```

#### الدرس المستفاد
الـ GRLIB ecosystem بيستخدم **non-standard compatible strings** مأخوذة من الـ AMBA plug&play database. مش هينفع تستخدم الـ Linux-standard `vendor,device` format. دايماً ارجع للـ binding documentation الأصلية.

---

### السيناريو 4: LEON3 IoT Sensor Node — إدارة IRQ Sharing غلط

#### العنوان
**`irqmap` بـ Shared Interrupt Index تسبب spurious interrupts**

#### السياق
IoT sensor node مبني على LEON3 SPARC soft-core داخل Xilinx Spartan FPGA. الـ node بيقرأ بيانات من multiple sensors عن طريق GPIO. الـ FPGA design فيه GRGPIO core بـ 8 lines، بس الـ interrupt controller عنده resources محدودة — بس 2 interrupt lines متاحين للـ GPIO.

#### المشكلة
المهندس حاول يوفر interrupt lines بإنه يـ map كل 8 GPIO lines على نفس 2 interrupts:

```dts
/* محاولة خاطئة لـ interrupt sharing */
gpio0: gpio@80000700 {
    compatible = "GAISLER_GPIO";
    reg = <0x80000700 0x100>;
    interrupts = <5 6>;
    nbits = <8>;
    irqmap = <0 0 0 0 1 1 1 1>;
    /* lines 0-3 → interrupt 5, lines 4-7 → interrupt 6 */
};
```

الـ system بيـ show spurious interrupts وبعض الـ GPIO events بيتـ miss.

#### التحليل
الـ binding بيوضح إن كل index في `irqmap` هو **index into the interrupts property array**:

```
irqmap : An array with an index for each gpio line.
         An index is either a valid index into the interrupts property array,
         or 0xffffffff that indicates no irq for that line.
```

يعني:
- `irqmap = <0 0 0 0 1 1 1 1>` → lines 0-3 كلهم بيـ share `interrupts[0]` (IRQ 5)، وlines 4-7 بيـ share `interrupts[1]` (IRQ 6)

المشكلة مش في الـ DT نفسه — ده valid mapping. المشكلة إن الـ GRGPIO hardware مش بيـ support shared interrupts بشكل طبيعي على نفس line. لما GPIO line 0 تـ trigger، الـ ISR بيتفعّل بس مش بيعرف يفرق بينها وبين line 1 أو 2 أو 3 بدون قراءة interrupt status register.

```c
/*
 * ISR لازم يقرأ GRGPIO Interrupt Flag Register
 * عشان يعرف أي line بالظبط عمل trigger
 * GRGPIO register map:
 * offset 0x00: I/O port value
 * offset 0x04: I/O port direction
 * offset 0x08: interrupt mask
 * offset 0x0C: interrupt polarity
 * offset 0x10: interrupt edge
 * offset 0x14: bypass
 * offset 0x18: capability
 * offset 0x20: interrupt map register (IRQMAP)
 * offset 0x24: interrupt status/flag register ← المهم هنا
 */
```

#### الحل
إذا كان الـ FPGA design يسمح، الأفضل تـ assign interrupt منفصل لكل line. لو مش ممكن، تأكد إن الـ ISR بيعمل proper demuxing:

```dts
/* Optimal: interrupt منفصل لكل line لو متاح */
gpio0: gpio@80000700 {
    compatible = "GAISLER_GPIO";
    reg = <0x80000700 0x100>;
    interrupts = <5 6 7 8 9 10 11 12>; /* interrupt منفصل لكل line */
    nbits = <8>;
    irqmap = <0 1 2 3 4 5 6 7>;
};
```

```bash
# مراقبة الـ spurious interrupts
watch -n 1 'cat /proc/interrupts | grep -E "gpio|5:|6:"'
```

#### الدرس المستفاد
الـ `irqmap` property بتـ allow sharing interrupts بين multiple GPIO lines، بس ده بيضع مسؤولية الـ demuxing على الـ driver/ISR. الأفضل دايماً **interrupt منفصل لكل GPIO line** في applications حساسة للـ timing.

---

### السيناريو 5: GRLIB FPGA Automotive ECU — AMBA Plug&Play مقابل Manual DT

#### العنوان
**تعارض بين AMBA Auto-discovery وManual DT في Automotive ECU**

#### السياق
شركة automotive بتطوّر ECU (Engine Control Unit) على LEON3FT SPARC processor مع GRLIB. النظام بيشتغل على Linux with real-time patches. الـ FPGA بيتضمن GRGPIO core للتحكم في actuators وقراءة sensors.

الـ binding بيذكر:

```
Note: In the ordinary environment for the GRGPIO core, a Leon SPARC system,
these properties are built from information in the AMBA plug&play.
```

#### المشكلة
الـ system كان شغّال تمام مع AMBA auto-discovery. بعد ما فريق الـ BSP قرر يـ migrate لـ manual DT لتحسين control وdocumentation، الـ GPIO stop working. الـ `reg` address في الـ manual DT كانت خاطئة.

```dts
/* Manual DT — reg address خاطئ */
gpio0: gpio@80000600 {
    compatible = "GAISLER_GPIO";
    reg = <0x80000600 0x100>; /* address غلط! */
    interrupts = <4>;
    nbits = <16>;
    irqmap = <0 0xffffffff 0xffffffff 0xffffffff
              0xffffffff 0xffffffff 0xffffffff 0xffffffff
              0xffffffff 0xffffffff 0xffffffff 0xffffffff
              0xffffffff 0xffffffff 0xffffffff 0xffffffff>;
};
```

#### التحليل
في بيئة LEON/GRLIB، الـ AMBA plug&play بيـ auto-detect الـ base address من الـ AMBA configuration area. لما بتكتب manual DT، لازم تـ verify الـ address يدوياً من الـ AMBA PnP scan.

طريقة معرفة الـ address الصح:

```bash
# GRLIB AMBA PnP scan — اقرأ الـ configuration area
# الـ AMBA PnP area عادةً على 0xFFFFF000 في LEON systems
devmem 0xFFFFF000 32   # AHB master 0
devmem 0xFFFFF800 32   # APB slaves start

# أو استخدم GRMON debugger
grmon -uart /dev/ttyUSB0 -baud 115200
> apbmst
# هيبيّن كل APB slaves وعناوينهم
```

لما اتأكدوا من الـ AMBA PnP، لاقوا إن الـ GRGPIO فعلاً على `0x80000900` مش `0x80000600`.

#### الحل

```dts
/* صح — reg address متحقق من AMBA PnP */
gpio0: gpio@80000900 {
    compatible = "GAISLER_GPIO";
    reg = <0x80000900 0x100>; /* address متحقق من AMBA PnP scan */
    interrupts = <4>;
    nbits = <16>;
    irqmap = <0 0xffffffff 0xffffffff 0xffffffff
              0xffffffff 0xffffffff 0xffffffff 0xffffffff
              0xffffffff 0xffffffff 0xffffffff 0xffffffff
              0xffffffff 0xffffffff 0xffffffff 0xffffffff>;
};

/* اختبار functional */
/* actuator control */
gpio-leds {
    compatible = "gpio-leds";
    actuator0 {
        gpios = <&gpio0 0 GPIO_ACTIVE_HIGH>;
        label = "fuel-injector-1";
    };
};
```

التحقق النهائي:

```bash
# تأكد إن الـ GPIO chip اتـ register بشكل صح
gpiodetect
# المفروض يبيّن: gpiochip0 [80000900.gpio] (16 lines)

# اختبار GPIO output
gpioset gpiochip0 0=1
gpioget gpiochip0 0

# مراقبة GPIO interrupt
gpiomon --num-events=5 gpiochip0 0
```

#### الدرس المستفاد
لما بتـ migrate من AMBA auto-discovery لـ manual DT في GRLIB systems، **دايماً اعمل AMBA PnP scan أولاً** باستخدام GRMON أو devmem عشان تتحقق من الـ base addresses. الـ addresses في manual DT لازم تطابق بالظبط ما بيـ report عنه الـ AMBA hardware.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي بتغطي الـ GPIO subsystem في الـ Linux kernel، من المقدمة لحد الـ descriptor-based API:

| المقال | الوصف |
|--------|--------|
| [GPIO in the kernel: an introduction](https://lwn.net/Articles/532714/) | مقدمة شاملة للـ GPIO في الـ kernel — بتشرح الـ gpiolib وازاي الـ drivers بتتكلم معاه |
| [GPIO in the kernel: future directions](https://lwn.net/Articles/533632/) | بيتكلم عن الـ descriptor-based interface وعن مشاكل الـ legacy integer API |
| [gpiolib: introduce descriptor-based GPIO interface](https://lwn.net/Articles/528226/) | أول patch بيعرّف الـ `gpio_desc *` بدل الـ integer |
| [gpio: introduce descriptor-based interface](https://lwn.net/Articles/531848/) | الـ patch الرسمي اللي اتقبل في الـ mainline لتعريف الـ `gpiod_*` API |
| [New descriptor-based GPIO interface](https://lwn.net/Articles/565662/) | ملخص للـ API الجديد بعد ما اتقبل |
| [Documentation: gpiolib: document new interface](https://lwn.net/Articles/574055/) | توثيق الـ API الجديد في الـ kernel docs |
| [The pin control subsystem](https://lwn.net/Articles/468759/) | بيشرح العلاقة بين الـ pinctrl subsystem والـ GPIO — مهم لفهم الـ mux |
| [gpio: sysfs interface](https://lwn.net/Articles/286435/) | الـ sysfs interface القديمة للـ GPIO من userspace |
| [gpio: Document GPIO hogging mechanism](https://lwn.net/Articles/641117/) | الـ GPIO hogging — طريقة الـ auto-request عند الـ probe |
| [[PATCH] gpio: Add driver for basic memory-mapped GPIO controllers](https://lwn.net/Articles/402868/) | مثال على كتابة driver لـ memory-mapped GPIO — قريب من الـ GRGPIO |

---

### التوثيق الرسمي في الـ Kernel

الملفات دي موجودة في `Documentation/` داخل الـ kernel source:

```
Documentation/devicetree/bindings/gpio/gpio-grgpio.txt   ← الـ DT binding للـ GRGPIO
Documentation/driver-api/gpio/index.rst                  ← الدخول الرئيسي للـ GPIO API
Documentation/driver-api/gpio/driver.rst                 ← كتابة GPIO chip driver
Documentation/driver-api/gpio/consumer.rst               ← استخدام GPIO من driver تاني
Documentation/driver-api/gpio/board.rst                  ← GPIO mappings (DT / ACPI / platform)
Documentation/driver-api/gpio/legacy.rst                 ← الـ integer-based legacy API
Documentation/admin-guide/gpio/                          ← الـ userspace interface (sysfs / chardev)
```

**الـ online links:**

- [General Purpose Input/Output (GPIO) — kernel docs](https://docs.kernel.org/driver-api/gpio/index.html)
- [GPIO Driver Interface](https://static.lwn.net/kerneldoc/driver-api/gpio/driver.html)
- [GPIO Descriptor Consumer Interface](https://docs.kernel.org/driver-api/gpio/consumer.html)
- [GPIO Mappings](https://static.lwn.net/kerneldoc/driver-api/gpio/board.html)
- [Legacy GPIO Interfaces](https://static.lwn.net/kerneldoc/driver-api/gpio/legacy.html)
- [Subsystem drivers using GPIO](https://static.lwn.net/kerneldoc/driver-api/gpio/drivers-on-gpio.html)
- [gpio-grgpio.txt على kernel.org](https://www.kernel.org/doc/Documentation/devicetree/bindings/gpio/gpio-grgpio.txt)

---

### الـ GRLIB وGaisler Documentation

الـ GRGPIO جزء من مكتبة الـ GRLIB VHDL IP، والـ documentation الرسمي بتاع Aeroflex Gaisler هو المرجع الأساسي لفهم الـ hardware registers:

- [GRLIB IP Core User's Manual (grip.pdf)](http://www.gaisler.com/products/grlib/grip.pdf) — المرجع الرسمي للـ GRGPIO core، بيشرح الـ registers والـ interrupt mapping
- [Linux for LEON — Gaisler](https://www.gaisler.com/products/linux-for-leon) — الـ page الرسمية لدعم Linux على معالجات LEON SPARC
- [LEON Linux User's Manual (PDF)](https://download.gaisler.com/products/leon-linux/doc/leon-linux.pdf) — دليل شامل لبناء وتشغيل Linux على الـ Leon3/Leon4

---

### Commits وتاريخ الكود

الـ driver `drivers/gpio/gpio-grgpio.c` اتضاف في الـ kernel v3.10 — للبحث في تاريخه:

```bash
# مشاهدة تاريخ الـ driver
git log --oneline -- drivers/gpio/gpio-grgpio.c

# مشاهدة الـ commit اللي أضاف الـ DT binding
git log --oneline -- Documentation/devicetree/bindings/gpio/gpio-grgpio.txt

# البحث عن أي تغيير في الـ GRGPIO
git log --all --grep="grgpio" --oneline
```

الـ driver على GitHub mirror:
- [gpio-grgpio.c على linux4sam/linux-at91](https://github.com/linux4sam/linux-at91/blob/master/drivers/gpio/gpio-grgpio.c)

---

### نقاشات Mailing List

- [LKML — linux-kernel mailing list archive](https://lkml.org/) — ابحث عن `grgpio` أو `GAISLER_GPIO`
- [[PATCH v3] spi: Add support for Aeroflex Gaisler GRLIB cores](https://sourceforge.net/p/spi-devel/mailman/message/30490217/) — مثال على patch تاني من Gaisler للـ GRLIB يوضح نمط الكود المتبع

للبحث في الـ LKML مباشرة:
```
https://lkml.org/lkml/?q=grgpio
https://lore.kernel.org/linux-gpio/?q=grgpio
```

---

### Kernelnewbies.org

- [Linux Kernel Newbies — LinuxChanges](https://kernelnewbies.org/LinuxChanges) — بيوضح الـ GPIO changes في كل إصدار kernel
- [Linux 6.2 Release Notes](https://kernelnewbies.org/Linux_6.2) — بيذكر GPIO latch driver وsoftware nodes في gpiolib
- [Using GPIO from device tree on platform devices](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2015-August/014911.html) — نقاش عملي عن الـ DT GPIO bindings
- [devm_gpiod_get usage](http://lists.kernelnewbies.org/pipermail/kernelnewbies/2016-August/016707.html) — مثال على استخدام الـ descriptor API في driver حقيقي

---

### eLinux.org

- [New GPIO Interface for User Space (ELCE 2017 PDF)](https://elinux.org/images/7/74/Elce2017_new_GPIO_interface.pdf) — عرض كامل عن الـ character device GPIO interface الجديد
- [Introduction to pin muxing and GPIO control under Linux (ELC 2021 PDF)](https://elinux.org/images/a/a7/ELC-2021_Introduction_to_pin_muxing_and_GPIO_control_under_Linux.pdf) — شرح للعلاقة بين pinmux والـ GPIO
- [EBC GPIO via mmap](https://elinux.org/EBC_GPIO_via_mmap) — مثال عملي على الـ memory-mapped GPIO access من userspace
- [EBC GPIO Polling and Interrupts](https://elinux.org/EBC_gpio_Polling_and_Interrupts) — التعامل مع الـ GPIO interrupts من sysfs

---

### كتب مُوصى بيها

| الكتاب | الفصل ذو الصلة |
|--------|----------------|
| **Linux Device Drivers, 3rd Ed. (LDD3)** — Corbet, Rubini, Kroah-Hartman | Chapter 1 (Driver model), متاح مجاناً على [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/) |
| **Linux Kernel Development, 3rd Ed.** — Robert Love | Chapter 17 (Devices and Modules) — بيشرح الـ device model اللي بيبني عليه الـ GPIO subsystem |
| **Embedded Linux Primer, 2nd Ed.** — Christopher Hallinan | Chapter 15 (Devices and Drivers) — بيغطي الـ GPIO من منظور الـ embedded systems |
| **Mastering Embedded Linux Programming** — Frank Vasquez, Chris Simmonds | Chapter 11 (Interfacing with Device Drivers) — مثال عملي على الـ GPIO character device API |

---

### Search Terms للبحث عن معلومات أكتر

لو عايز تعمق أكتر، استخدم الـ search terms دي:

```
# على Google / DuckDuckGo
linux kernel grgpio driver
GAISLER_GPIO device tree binding
GRLIB AMBA plug and play linux
Leon SPARC linux gpio
gpiolib chip driver implementation
gpio_chip irq domain linux kernel
OF_PLATFORM gpio driver example

# على lore.kernel.org
https://lore.kernel.org/linux-gpio/

# على elixir.bootlin.com (kernel cross-reference)
https://elixir.bootlin.com/linux/latest/source/drivers/gpio/gpio-grgpio.c
```
## Phase 8: Writing simple module

### الفكرة

**الـ** `grgpio_probe` هي نقطة الدخول الرئيسية للـ driver — بتتشغل لما الـ kernel يلاقي device node في الـ device tree يطابق `GAISLER_GPIO` أو `01_01a`. هنستخدم **kprobe** نـ hook عليها عشان نطبع معلومات عن كل GRGPIO device بيتعمله probe، زي عدد الـ GPIO lines وحالة الـ IRQ support.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe module: hook grgpio_probe to log GRGPIO device info at probe time
 */

#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/platform_device.h>
#include <linux/of.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on grgpio_probe to trace GRGPIO device initialization");

/*
 * pre_handler runs BEFORE grgpio_probe executes.
 * pt_regs gives us access to CPU registers at the probe point.
 * On x86_64: rdi = first arg (struct platform_device *ofdev)
 * On SPARC/ARM: use regs->regs[0] or regs->ARM_r0 accordingly
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    struct platform_device *ofdev;
    struct device_node *np;
    u32 nbits = 32; /* default if property absent */

#ifdef CONFIG_X86_64
    /* First argument is in rdi on x86_64 System V ABI */
    ofdev = (struct platform_device *)regs->di;
#elif defined(CONFIG_ARM64)
    /* First argument is in x0 on AArch64 */
    ofdev = (struct platform_device *)regs->regs[0];
#elif defined(CONFIG_SPARC)
    /* First argument is in o0 on SPARC */
    ofdev = (struct platform_device *)regs->u_regs[UREG_I0];
#else
    /* Fallback: skip logging on unknown arch */
    pr_info("grgpio_kprobe: probe fired (arch unknown, no arg decode)\n");
    return 0;
#endif

    if (!ofdev)
        return 0;

    np = ofdev->dev.of_node;

    /* Read optional nbits property from DT node */
    of_property_read_u32(np, "nbits", &nbits);

    pr_info("grgpio_kprobe: probe called for device '%s' | "
            "DT node: '%s' | nbits=%u | irqmap present: %s\n",
            dev_name(&ofdev->dev),
            np ? np->full_name : "(no node)",
            nbits,
            of_get_property(np, "irqmap", NULL) ? "yes" : "no");

    return 0; /* 0 = continue normal execution */
}

/* post_handler runs AFTER grgpio_probe returns (before caller resumes) */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /* ax/x0 holds the return value on x86_64/arm64 */
#ifdef CONFIG_X86_64
    long ret = (long)regs->ax;
#elif defined(CONFIG_ARM64)
    long ret = (long)regs->regs[0];
#else
    long ret = 0;
#endif
    pr_info("grgpio_kprobe: grgpio_probe returned %ld (%s)\n",
            ret, ret == 0 ? "success" : "error");
}

/* Define the kprobe: target symbol is the function name as a string */
static struct kprobe kp = {
    .symbol_name    = "grgpio_probe",
    .pre_handler    = handler_pre,
    .post_handler   = handler_post,
};

static int __init grgpio_kprobe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("grgpio_kprobe: register_kprobe failed, ret=%d\n", ret);
        /*
         * Common causes:
         *  -EINVAL  -> symbol not found or CONFIG_KPROBES disabled
         *  -ENOENT  -> function is inline or __kprobes annotated
         */
        return ret;
    }

    pr_info("grgpio_kprobe: planted kprobe at %pS\n", kp.addr);
    return 0;
}

static void __exit grgpio_kprobe_exit(void)
{
    /*
     * Must unregister before the module is unloaded.
     * If we don't, the kernel will call a handler that no longer exists
     * → instant panic.
     */
    unregister_kprobe(&kp);
    pr_info("grgpio_kprobe: kprobe removed from grgpio_probe\n");
}

module_init(grgpio_kprobe_init);
module_exit(grgpio_kprobe_exit);
```

---

### Makefile

```makefile
obj-m += grgpio_kprobe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

```bash
# بناء وتحميل الـ module
make
sudo insmod grgpio_kprobe.ko
dmesg | grep grgpio_kprobe
sudo rmmod grgpio_kprobe
```

---

### شرح كل جزء

#### الـ Includes

```c
#include <linux/kprobes.h>
#include <linux/platform_device.h>
#include <linux/of.h>
```

**الـ** `kprobes.h` بيجيب `struct kprobe` وكل الـ API الخاص بالـ hooking. **الـ** `platform_device.h` و `of.h` محتاجينهم عشان نـ cast الـ argument وناخد properties من الـ device tree node.

#### الـ `handler_pre`

بيتشغل قبل ما `grgpio_probe` تبدأ تنفذ. بناخد الـ pointer للـ `platform_device` من الـ registers (ABI-specific)، وبعدين نقرأ `nbits` من الـ DT ونطبع اسم الـ device وعدد الـ GPIO lines وهل فيه `irqmap` ولا لأ — ده بيعطينا snapshot كامل عن أي GRGPIO device بتيجي للـ probe.

#### الـ `handler_post`

بيتشغل بعد ما `grgpio_probe` ترجع. بيطبع الـ return value — لو `0` يبقى الـ device اتسجل صح، لو error يبقى فيه مشكلة في الـ initialization.

#### الـ `struct kprobe`

**الـ** `symbol_name` بيقول للـ kernel فين يحط الـ breakpoint بالاسم — الـ kernel بيحوله لـ address تلقائيًا عن طريق kallsyms. مش محتاج نحدد address يدويًا.

#### الـ `module_init` / `module_exit`

`register_kprobe` بتزرع الـ hook في الـ kernel memory. لو فشلت (مثلًا الـ symbol مش موجود أو الـ function كانت inlined وراحت) بنرجع الـ error ومش بنكمل. في الـ exit، `unregister_kprobe` إجباري — لو الـ module اتشال من الـ memory من غير ما يتعمل unregister، أي trigger للـ probe هيـ crash الـ kernel لأنه هيحاول يجري code مش موجود.

---

### ملاحظة مهمة

**الـ** `grgpio_probe` ممكن تتعمل `inline` من compiler أو تتعمل `__kprobes` لو الـ kernel version اتغير — في الحالة دي `register_kprobe` هترجع `-EINVAL`. تقدر تتحقق بـ:

```bash
grep grgpio_probe /proc/kallsyms
```

لو مش موجودة، الـ function اتدمجت في كود تاني وmust hook a caller instead.
