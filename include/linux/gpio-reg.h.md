## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي ليه الملف

الـ `gpio/gpio-reg.h` جزء من **GPIO Subsystem** في Linux kernel. المشرفون عليه هم Linus Walleij وBartosz Golaszewski، والـ mailing list هو `linux-gpio@vger.kernel.org`.

---

### الصورة الكبيرة — قبل أي كود

#### الـ GPIO إيه أصلاً؟

تخيل إن عندك مكيف قديم، وعايز تتحكم فيه بـ microcontroller. المكيف عنده سلك واحد: لو بعتله 1 اتشغّل، لو بعتله 0 اتقفل. ده بالظبط الـ **GPIO — General Purpose Input/Output**.

الـ GPIO هو pin (طرف) في الـ chip أو الـ SoC بتاعك، ممكن:
- يبعت إشارة **output** — زي إنك تشغّل LED أو relay.
- يستقبل إشارة **input** — زي إنك تقرأ حالة زرار أو sensor.

#### المشكلة اللي `gpio-reg` بيحلها

في بعض الـ hardware القديم أو البسيط جداً، بيكون عندك **register واحد فقط** (زي عنوان في الـ memory) بيتحكم في مجموعة من الـ GPIO lines مع بعض. ده مش chip GPIO متطور بيدعم software direction-switching — لأ، كل pin اتجاهه **ثابت من تصميم الـ hardware**:

- الـ bits دي للـ input وبس.
- الـ bits دي للـ output وبس.
- مش ممكن تغير ده في الـ runtime.

مثال حقيقي: في الـ **SA-1100 Assabet** board (ARM قديم من تسعينيات الكيبل):

```
Register @ 0x...
  Bit  0-23: Output GPIOs (LEDs, control lines)
  Bit 24-31: Input GPIOs  (buttons, status lines)
  — لو حاولت تحول output لـ input: مستحيل، الـ hardware مش بيسمح
```

بدل ما كل driver يعيد اختراع العجلة ويكتب نفس الكود من أوله، Russell King في 2016 كتب `gpio-reg`: **helper بسيط جداً يسجّل register واحد كـ gpio_chip كامل**.

---

### الهدف من الملف

**الـ `include/linux/gpio/gpio-reg.h`** هو مجرد **header صغير جداً** بيعرّف الـ API العام للـ `gpio-reg`:

```c
/* سجّل register واحد كـ gpio chip */
struct gpio_chip *gpio_reg_init(struct device *dev, void __iomem *reg,
    int base, int num, const char *label, u32 direction, u32 def_out,
    const char *const *names, struct irq_domain *irqdom, const int *irqs);

/* استرجع الـ output values بعد suspend/resume */
int gpio_reg_resume(struct gpio_chip *gc);
```

ده الـ contract بين الـ drivers اللي بتستخدم الـ feature دي وبين الـ implementation.

---

### القصة الكاملة بأسلوب بسيط

تخيل إنك بتبني لعبة إلكترونية قديمة. عندك chip صغير فيه register بحجم 32-bit. كل bit بتحكم في حاجة:

```
Bit 0 → يولّع الـ LED الأحمر
Bit 1 → يولّع الـ LED الأزرق
Bit 8 → يقرأ حالة الـ زرار "Start"
Bit 9 → يقرأ حالة الـ زرار "Select"
```

لما Linux بيشتغل على الـ hardware ده، محتاج يفهم إن:
- Bits 0-7 = output فقط.
- Bits 8-15 = input فقط.
- الباقي مش موجود.

الـ `gpio_reg_init` بتاخد كل المعلومات دي في call واحدة وبتسجّل الـ register ده كـ **gpio_chip** قانوني في Linux، فكل الـ kernel subsystems التانية (زي الـ pinctrl، والـ interrupt controller، والـ userspace عبر `/dev/gpiochipX`) تقدر تتكلم معاه بشكل طبيعي.

---

### المعاملات المهمة في `gpio_reg_init`

| Parameter | الدور |
|-----------|-------|
| `reg` | عنوان الـ memory-mapped register |
| `num` | عدد الـ GPIO pins (max 32) |
| `direction` | bitmask — 1 = input، 0 = output (ثابت) |
| `def_out` | قيمة الـ output الابتدائية عند الـ boot |
| `names` | أسماء الـ pins (للـ debugging) |
| `irqdom` + `irqs` | لو الـ pins مربوطة بـ interrupts |

---

### سلوك خاص مهم

**الـ double-read للـ input:**
بعض الـ registers القديمة بتعمل "latch" — يعني لو قرأت القيمة مرة واحدة ممكن تاخد قيمة قديمة. الحل إنك تقرأ مرتين:

```c
/* input pins: اقرأ مرتين عشان تتأكد من القيمة الصح */
readl_relaxed(r->reg);          /* flush any latch */
val = readl_relaxed(r->reg);    /* القيمة الحقيقية */
```

**الـ spinlock:**
عشان output bits متعددة ممكن تتعدل من threads مختلفة في نفس الوقت، بيستخدم `spin_lock_irqsave` لحماية الـ read-modify-write:

```c
spin_lock_irqsave(&r->lock, flags);
val = r->out;
val |= mask;   /* أو val &= ~mask */
r->out = val;
writel_relaxed(val, r->reg);
spin_unlock_irqrestore(&r->lock, flags);
```

---

### الملفات المرتبطة اللي المفروض تعرفها

#### الملفات الأساسية

| الملف | الدور |
|-------|-------|
| `include/linux/gpio/gpio-reg.h` | **الملف ده** — الـ public API |
| `drivers/gpio/gpio-reg.c` | الـ implementation الكامل |
| `include/linux/gpio/driver.h` | تعريف `struct gpio_chip` الأساسي |
| `include/linux/gpio/consumer.h` | الـ API للـ drivers اللي بتستهلك الـ GPIO |

#### الـ Users الحقيقيين

| الملف | الاستخدام |
|-------|-----------|
| `arch/arm/mach-sa1100/assabet.c` | Board قديم — register واحد لـ 32 GPIO |
| `arch/arm/mach-sa1100/neponset.c` | Board تاني من نفس العيلة |

#### ملفات الـ Subsystem الأشمل

| الملف | الدور |
|-------|-------|
| `drivers/gpio/gpiolib.c` | قلب الـ GPIO subsystem |
| `drivers/gpio/gpiolib-of.c` | دعم الـ Device Tree |
| `include/linux/gpio.h` | الـ legacy API |
| `include/linux/of_gpio.h` | الـ Device Tree GPIO API |
| `include/linux/gpio/regmap.h` | نظير أحدث يستخدم regmap بدل raw MMIO |

---

### خلاصة سريعة

```
Hardware Register (32-bit)
        │
        ▼
  gpio_reg_init()
        │
        ▼
  struct gpio_chip  ←── Linux GPIO Framework
        │
        ├── /sys/class/gpio/
        ├── /dev/gpiochipX
        └── kernel drivers (IRQ, pinctrl, etc.)
```

الـ `gpio-reg.h` هو **بوابة صغيرة جداً** لفكرة كبيرة: إنك تأخذ أي register عادي في أي hardware قديم أو بسيط، وتحوّله لـ GPIO chip كامل يتكلم مع باقي الـ Linux kernel بلغة واحدة.
## Phase 2: شرح الـ GPIO Subsystem Framework (مع تركيز على gpio-reg)

### المشكلة اللي الـ GPIO Subsystem بيحلها

في أي SoC أو embedded board، عندك مئات الـ GPIO pins — بعضها على الـ SoC نفسه، بعضها على expander chip متصل بـ I2C أو SPI، وبعضها مجرد bit في register ثابت في memory-mapped peripheral. كل واحد فيهم بيتكلم بطريقة مختلفة تماماً على مستوى الـ hardware.

**المشكلة الحقيقية**: لو مفيش abstraction موحد، كل driver محتاج يعرف هو بالظبط فين الـ GPIO اللي بيستخدمه، وهيكلم أنهي hardware بالظبط. ده بيعمل coupling شديد بين الـ consumer (اللي محتاج GPIO) والـ provider (اللي بيوفر الـ GPIO).

مثال عملي: driver الـ SD card محتاج يتحكم في pin الـ card-detect. مش المفروض يعرف إن الـ pin ده جزء من SoC GPIO block أو من expander على I2C. محتاج بس يقول "اقرأ الـ GPIO رقم كذا".

---

### الحل اللي الـ Kernel اتبعه

**الـ GPIO Subsystem** (المعروف بـ `gpiolib`) بيعمل طبقة abstraction كاملة:

- بيعرف **واجهة موحدة** للـ consumers (أي كود بيحتاج GPIO).
- بيطلب من كل GPIO controller إنه يسجل نفسه عن طريق `struct gpio_chip`.
- بيعمل **namespace** موحد: كل GPIO ليه رقم Linux فريد (أو في الحديث: descriptor `gpiod_*` API).

**الـ gpio-reg** تحديداً هو أبسط implementation ممكن: controller واحد، register واحد في memory، كل bit فيه = GPIO واحد، والاتجاه (input/output) ثابت من وقت الـ init ومش بيتغير.

---

### التشبيه الواقعي (مع Mapping كامل)

تخيل **لوحة مفاتيح في مبنى حكومي**:

| التشبيه | المفهوم الحقيقي |
|---------|----------------|
| المبنى الحكومي كله | الـ SoC أو الـ board |
| قسم الأمن (بوابة رئيسية) | الـ `gpiolib` core |
| الغرف المختلفة في المبنى | controllers مختلفة (SoC GPIO block، I2C expander، gpio-reg) |
| الباب الثابت اللي بيفتح برة بس (one-way) | GPIO ذو اتجاه ثابت كـ input |
| الباب الثابت اللي بيفتح جوه بس (one-way) | GPIO ذو اتجاه ثابت كـ output |
| بطاقة الدخول الموحدة | الـ `gpiod_*` API (consumer API) |
| سجل الأمن (registry) | الـ global GPIO namespace في gpiolib |
| موظف مبنى بيستخدم بطاقته | أي driver بيطلب `gpiod_get()` |
| كل غرفة بتحدد قواعدها الداخلية | كل `gpio_chip` بيوفر callbacks (`get`, `set`, `direction_*`) |
| اللوحة التحكم البسيطة لغرفة واحدة | `gpio-reg`: single register، كل switch = GPIO واحد |

الـ **gpio-reg** تحديداً زي غرفة فيها لوحة مفاتيح بسيطة: كل مفتاح إما locked كـ input (read-only) أو locked كـ output (write-only)، ومفيش حاجة بتغير اتجاهه بعد ما الغرفة اتبنت.

---

### Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Consumer Layer                              │
│   (device drivers, board code, user-space via /dev/gpiochipN)  │
│                                                                 │
│   gpiod_get() / gpiod_set_value() / gpiod_direction_input()    │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                   gpiolib Core (drivers/gpio/gpiolib.c)         │
│                                                                 │
│  - Global GPIO number space                                     │
│  - GPIO descriptor (gpiod) management                          │
│  - Sysfs / chardev interface (/dev/gpiochipN)                  │
│  - IRQ translation (gpio_to_irq)                               │
│  - DT / ACPI / board-file lookup                               │
│                                                                 │
│  Calls into registered gpio_chip callbacks                      │
└──────┬────────────┬──────────────┬──────────────────────────────┘
       │            │              │
       ▼            ▼              ▼
┌────────────┐ ┌──────────┐ ┌─────────────────────────────────┐
│ SoC GPIO   │ │ I2C/SPI  │ │  gpio-reg (gpio-reg.c)          │
│ Block      │ │ Expander │ │                                 │
│ (e.g.      │ │ (e.g.    │ │  struct gpio_reg {              │
│ pl061.c)   │ │ pcf857x) │ │    struct gpio_chip gc;         │
│            │ │          │ │    spinlock_t lock;             │
│ gpio_chip  │ │ gpio_chip│ │    u32 direction; /* fixed */   │
│ registered │ │registered│ │    u32 out;       /* shadow */  │
│ via        │ │ via      │ │    void __iomem *reg;           │
│ gpiochip_  │ │ gpiochip_│ │    struct irq_domain *irqdomain;│
│ add_data() │ │ add_data │ │    const int *irqs;             │
└────────────┘ └──────────┘ │  }                              │
                             └─────────────────────────────────┘
                                          │
                                          ▼
                             ┌─────────────────────────┐
                             │  Memory-Mapped Register  │
                             │  (single 32-bit reg)     │
                             │                          │
                             │  bit 0 = GPIO 0          │
                             │  bit 1 = GPIO 1          │
                             │  ...                     │
                             │  bit N = GPIO N          │
                             └─────────────────────────┘
```

---

### الـ Core Abstraction: `struct gpio_chip`

ده قلب الـ GPIO subsystem. أي GPIO controller — مهما كان hardware complexity — لازم يملأ `struct gpio_chip` ويسجله.

```c
struct gpio_chip {
    const char      *label;          /* اسم الـ controller */
    struct device   *parent;         /* الـ device اللي بيملكه */

    /* ---- Direction Control ---- */
    int (*get_direction)(struct gpio_chip *gc, unsigned int offset);
    int (*direction_input)(struct gpio_chip *gc, unsigned int offset);
    int (*direction_output)(struct gpio_chip *gc, unsigned int offset, int value);

    /* ---- Value Read/Write ---- */
    int (*get)(struct gpio_chip *gc, unsigned int offset);
    int (*set)(struct gpio_chip *gc, unsigned int offset, int value);

    /* ---- Bulk Operations (optimization) ---- */
    int (*get_multiple)(struct gpio_chip *gc, unsigned long *mask, unsigned long *bits);
    int (*set_multiple)(struct gpio_chip *gc, unsigned long *mask, unsigned long *bits);

    /* ---- IRQ Support ---- */
    int (*to_irq)(struct gpio_chip *gc, unsigned int offset);

    /* ---- Identity ---- */
    int   base;   /* أول GPIO number في الـ global namespace (-1 = dynamic) */
    u16   ngpio;  /* عدد الـ GPIOs في الـ chip */
    const char *const *names; /* أسماء اختيارية لكل line */

    /* ---- IRQ chip integration (عند تفعيل CONFIG_GPIOLIB_IRQCHIP) ---- */
    struct gpio_irq_chip irq;
};
```

**الـ `offset`** هو رقم داخل الـ chip بيتراوح من `0` لـ `ngpio-1`. الـ gpiolib هو اللي بيترجم بين الـ global GPIO number والـ offset الداخلي.

---

### تشريح `struct gpio_reg` — الـ Internal State

الـ `gpio-reg` بيعمل `struct gpio_reg` الخاصة بيه وبيضمن فيها `struct gpio_chip`:

```c
struct gpio_reg {
    struct gpio_chip gc;       /* لازم يكون أول field — عشان container_of */
    spinlock_t lock;           /* يحمي الـ out register من race conditions */
    u32 direction;             /* bitmask: 1 = input، 0 = output (fixed at init) */
    u32 out;                   /* shadow copy للـ output value في الـ RAM */
    void __iomem *reg;         /* pointer للـ memory-mapped register */
    struct irq_domain *irqdomain; /* لو الـ GPIOs ليها interrupts */
    const int *irqs;           /* mapping: GPIO offset → hwirq number */
};

/* بيجيب pointer على gpio_reg من pointer على gpio_chip المضمن فيه */
#define to_gpio_reg(x) container_of(x, struct gpio_reg, gc)
```

```
Memory Layout of struct gpio_reg:
┌──────────────────────────────────────────┐
│  struct gpio_chip gc          ← offset 0 │
│  ┌────────────────────────────────────┐   │
│  │ label, parent, callbacks, base ... │   │
│  └────────────────────────────────────┘   │
├──────────────────────────────────────────┤
│  spinlock_t lock                          │
├──────────────────────────────────────────┤
│  u32 direction   (e.g. 0b00001111 = 4 in)│
├──────────────────────────────────────────┤
│  u32 out         (shadow of HW reg)       │
├──────────────────────────────────────────┤
│  void __iomem *reg  → MMIO address        │
├──────────────────────────────────────────┤
│  struct irq_domain *irqdomain             │
├──────────────────────────────────────────┤
│  const int *irqs                          │
└──────────────────────────────────────────┘

container_of(gc_ptr, struct gpio_reg, gc)
        │
        ▼
بيطرح offset الـ gc من الـ pointer يرجع لأول الـ struct
```

---

### ليه الـ Shadow Register (`r->out`)؟

ده نقطة دقيقة جداً في الـ hardware design. الـ gpio-reg بيفترض إن الـ output GPIOs **مش قابلة للقراءة من الـ hardware register** — وده شائع جداً في الـ SoC peripherals. بعض الـ registers write-only من ناحية الـ hardware.

الحل: بيحتفظ بـ copy في الـ RAM (`r->out`) ويعمل عليها الـ read-modify-write، وبعدين يكتب النتيجة على الـ hardware.

```c
static int gpio_reg_set(struct gpio_chip *gc, unsigned int offset, int value)
{
    struct gpio_reg *r = to_gpio_reg(gc);
    unsigned long flags;
    u32 val, mask = BIT(offset);

    spin_lock_irqsave(&r->lock, flags);  /* حماية من interrupt context */
    val = r->out;                         /* اقرأ من الـ shadow (مش من HW) */
    if (value)
        val |= mask;
    else
        val &= ~mask;
    r->out = val;                         /* حدّث الـ shadow */
    writel_relaxed(val, r->reg);         /* اكتب على الـ hardware */
    spin_unlock_irqrestore(&r->lock, flags);

    return 0;
}
```

**ليه `spin_lock_irqsave` وليس `mutex`؟** لأن الـ GPIO set ممكن يتعمل من interrupt context (زي داخل IRQ handler). الـ mutex بينام والـ spinlock لأ.

---

### الـ Input Double-Read Pattern

```c
static int gpio_reg_get(struct gpio_chip *gc, unsigned offset)
{
    struct gpio_reg *r = to_gpio_reg(gc);
    u32 val, mask = BIT(offset);

    if (r->direction & mask) {
        /* Input GPIO: double-read عشان بعض الـ registers بتعمل latch */
        readl_relaxed(r->reg);      /* أول read: unlock/latch */
        val = readl_relaxed(r->reg); /* تاني read: القيمة الحقيقية */
    } else {
        val = r->out;               /* Output GPIO: مش ممكن قراءة من HW */
    }
    return !!(val & mask);
}
```

الـ **double-read** ده مش bug، ده intentional. بعض الـ hardware registers (خصوصاً في الـ FPGAs و legacy peripherals) بيعمل latch للقيمة بعد أول read فقط، والقيمة الفعلية بتيجي في التاني.

---

### الـ Fixed Direction — المفهوم الأساسي لـ gpio-reg

الـ `direction` field بيتحدد **مرة واحدة فقط** في الـ `gpio_reg_init` ومش بيتغير أبداً. ده يعني:

- لو `direction & BIT(n)` = 1 → GPIO n هو **input only**، ومحدش يقدر يحوله output.
- لو `direction & BIT(n)` = 0 → GPIO n هو **output only**، ومحدش يقدر يحوله input.

```c
static int gpio_reg_direction_output(struct gpio_chip *gc, unsigned offset, int value)
{
    struct gpio_reg *r = to_gpio_reg(gc);
    if (r->direction & BIT(offset))  /* لو الـ GPIO ده input في الـ HW */
        return -ENOTSUPP;            /* ارفض التحويل */
    gc->set(gc, offset, value);
    return 0;
}
```

ده بيعكس hardware واقعي: بعض الـ CPU boards عندها signals معينة hardwired للـ input (زي status bits من FPGA) أو output (زي control lines لـ reset pins).

---

### الـ IRQ Integration في gpio-reg

الـ gpio-reg بيدعم optional interrupt mapping. لو الـ caller بعت `irqs` array في الـ `gpio_reg_init`، الـ gpiolib هيعرف إنه يترجم من GPIO offset لـ IRQ number:

```c
static int gpio_reg_to_irq(struct gpio_chip *gc, unsigned offset)
{
    struct gpio_reg *r = to_gpio_reg(gc);
    int irq = r->irqs[offset];  /* IRQ number (hw أو linux حسب وجود irqdomain) */

    if (irq >= 0 && r->irqdomain)
        irq = irq_find_mapping(r->irqdomain, irq);  /* hw → linux IRQ translation */

    return irq;
}
```

**الـ `irq_domain`** هنا: هو subsystem منفصل بيعمل mapping بين hardware interrupt numbers وLinux virtual IRQ numbers. لو محتاج تفهمه أعمق، هو موضوع مستقل (IRQ Domain subsystem في `kernel/irq/`).

```
GPIO offset 3
     │
     ▼
r->irqs[3] = 47  (hwirq رقم 47)
     │
     ▼ (لو في irqdomain)
irq_find_mapping(irqdomain, 47) = 234  (Linux virq)
     │
     ▼
Consumer يستخدم virq 234 مع request_irq()
```

---

### الـ Resume Support — Power Management

```c
int gpio_reg_resume(struct gpio_chip *gc)
{
    struct gpio_reg *r = to_gpio_reg(gc);
    unsigned long flags;

    spin_lock_irqsave(&r->lock, flags);
    writel_relaxed(r->out, r->reg);  /* أعد كتابة الـ shadow على الـ HW */
    spin_unlock_irqrestore(&r->lock, flags);

    return 0;
}
```

بعد الـ suspend، الـ hardware register بيرجع لـ reset value. الـ `r->out` (الـ shadow) لسه بيحتفظ بالقيمة الصح في الـ RAM. الـ `gpio_reg_resume` بيعيد كتابة الـ shadow على الـ hardware.

ده pattern شائع جداً في embedded: **"RAM is your ground truth, HW is your mirror"**.

---

### الـ `gpio_reg_init` — نقطة الدخول الوحيدة

```c
struct gpio_chip *gpio_reg_init(
    struct device *dev,      /* لـ devm_ memory management */
    void __iomem *reg,       /* عنوان الـ MMIO register */
    int base,                /* أول GPIO number (-1 = dynamic) */
    int num,                 /* عدد الـ GPIOs (max 32) */
    const char *label,       /* اسم الـ chip */
    u32 direction,           /* bitmask: 1=in، 0=out */
    u32 def_out,             /* قيمة أولية للـ outputs */
    const char *const *names,/* أسماء اختيارية للـ lines */
    struct irq_domain *irqdom,/* irq domain أو NULL */
    const int *irqs          /* irq mapping array أو NULL */
);
```

الدالة دي بتعمل كل حاجة:
1. تخصص `struct gpio_reg` (بـ `devm_kzalloc` لو في device).
2. تملأ الـ `gpio_chip` callbacks.
3. تسجل الـ chip مع الـ gpiolib بـ `gpiochip_add_data`.
4. ترجع pointer على الـ `gpio_chip` جوه الـ struct.

---

### ما الـ gpio-reg بيملكه مقابل ما يفوّضه

```
┌─────────────────────────────────────────────────────────┐
│             ما gpio-reg بيملكه ويتحكم فيه              │
├─────────────────────────────────────────────────────────┤
│ • Shadow register management (r->out)                   │
│ • Spinlock protection                                   │
│ • Fixed direction enforcement                           │
│ • Double-read logic for inputs                          │
│ • IRQ number translation (to_irq callback)              │
│ • Resume logic (re-write shadow to HW)                  │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│          ما gpio-reg بيفوّضه للـ gpiolib core           │
├─────────────────────────────────────────────────────────┤
│ • Global GPIO namespace management                      │
│ • Consumer API (gpiod_get, gpiod_set_value, ...)        │
│ • Sysfs / chardev interface                             │
│ • Device Tree / ACPI lookup                             │
│ • IRQ chip initialization (GPIOLIB_IRQCHIP)             │
│ • Locking at the descriptor level (gpio_desc)           │
│ • Debug / monitoring (debugfs)                          │
└─────────────────────────────────────────────────────────┘
```

---

### مثال عملي: Freescale/NXP iMX Board

في بعض الـ iMX boards، الـ FPGA المتصل بيها بيعرض status signals كـ bits في register واحد. بدل ما تكتب driver كامل، بتستخدم `gpio-reg`:

```c
/* في board setup code */
static const char *fpga_gpio_names[] = {
    "fpga-ready", "ddr-init-done", "pcie-link-up", "usb-oc-n", NULL
};

static const int fpga_irqs[] = { 45, 46, -1, 47 }; /* -1 = no irq */

static int __init board_init_fpga_gpios(void)
{
    void __iomem *reg = ioremap(FPGA_STATUS_REG_ADDR, 4);

    /* كل الـ 4 GPIOs هي inputs (direction = 0xF = 0b1111) */
    gpio_reg_init(NULL, reg, -1, 4,
                  "fpga-status",
                  0xF,          /* all inputs */
                  0x0,          /* def_out (don't care for inputs) */
                  fpga_gpio_names,
                  NULL,         /* no irqdomain, use Linux IRQ numbers directly */
                  fpga_irqs);
    return 0;
}
```

بعد كده، أي driver تاني ممكن يعمل:
```c
struct gpio_desc *ready = gpiod_get(dev, "fpga-ready", GPIOD_IN);
int val = gpiod_get_value(ready);
```

من غير ما يعرف أي حاجة عن الـ FPGA register أو الـ board wiring.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

### 0. الـ Flags والـ Constants — Cheatsheet

| الاسم | القيمة | المعنى |
|---|---|---|
| `GPIO_LINE_DIRECTION_IN` | `1` | الـ GPIO اتجاهه input ثابت |
| `GPIO_LINE_DIRECTION_OUT` | `0` | الـ GPIO اتجاهه output ثابت |
| `BIT(offset)` | `1 << offset` | bitmask للـ GPIO رقم offset |
| `GFP_KERNEL` | kernel flag | تخصيص ذاكرة normal sleeping context |
| `ERR_PTR(-ENOMEM)` | error pointer | فشل التخصيص من الذاكرة |
| `ERR_PTR(-ENOTSUPP)` | error pointer | محاولة تغيير direction ثابت |

**الـ `direction` bitmask** — كل bit يمثل GPIO واحد:

| Bit | القيمة | المعنى |
|---|---|---|
| `direction & BIT(n) == 1` | input | الـ GPIO رقم n اتجاهه input |
| `direction & BIT(n) == 0` | output | الـ GPIO رقم n اتجاهه output |

**الـ `def_out` bitmask** — نفس المبدأ للـ output الافتراضي عند الـ init.

---

### 1. الـ Structs الأساسية

#### `struct gpio_reg` — الـ Internal State (في gpio-reg.c)

ده الـ struct الداخلي اللي مش مُعلَن في الـ header، بيحتضن كل حالة الـ driver:

```c
struct gpio_reg {
    struct gpio_chip gc;        /* embedded gpio_chip — must be first */
    spinlock_t lock;            /* protects r->out and register writes */
    u32 direction;              /* bitmask: 1=IN, 0=OUT per GPIO line */
    u32 out;                    /* shadow of current output register value */
    void __iomem *reg;          /* mapped MMIO address of the single HW reg */
    struct irq_domain *irqdomain; /* optional IRQ domain for gpio->irq mapping */
    const int *irqs;            /* array[ngpio] of irq numbers or hw-irq offsets */
};
```

| الحقل | النوع | الدور |
|---|---|---|
| `gc` | `struct gpio_chip` | الـ embedded chip — الـ core يتعامل معاه مباشرة |
| `lock` | `spinlock_t` | يحمي الكتابة على `r->out` والـ register |
| `direction` | `u32` | bitmask ثابت للاتجاه، بيُحدَّد مرة واحدة وقت الـ init |
| `out` | `u32` | shadow register — بيخزن آخر قيمة كتبناها، لأن HW مش دايما readable |
| `reg` | `void __iomem *` | عنوان الـ MMIO register الوحيد |
| `irqdomain` | `struct irq_domain *` | لو موجود، بيُستخدم لتحويل hwirq → linux irq |
| `irqs` | `const int *` | مصفوفة mapping: GPIO offset → irq number |

---

#### `struct gpio_chip` — الـ Public GPIO Abstraction (في gpio/driver.h)

ده الـ struct اللي الـ kernel يشوفه ويتعامل معاه:

```c
struct gpio_chip {
    const char          *label;           /* human-readable chip name */
    struct gpio_device  *gpiodev;         /* opaque internal gpiolib state */
    struct device       *parent;          /* optional parent device */
    struct fwnode_handle *fwnode;         /* firmware node (DT/ACPI) */
    struct module       *owner;

    /* ops — filled by driver */
    int  (*request)(struct gpio_chip *gc, unsigned int offset);
    void (*free)(struct gpio_chip *gc, unsigned int offset);
    int  (*get_direction)(struct gpio_chip *gc, unsigned int offset);
    int  (*direction_input)(struct gpio_chip *gc, unsigned int offset);
    int  (*direction_output)(struct gpio_chip *gc, unsigned int offset, int value);
    int  (*get)(struct gpio_chip *gc, unsigned int offset);
    int  (*get_multiple)(struct gpio_chip *gc, unsigned long *mask, unsigned long *bits);
    int  (*set)(struct gpio_chip *gc, unsigned int offset, int value);
    int  (*set_multiple)(struct gpio_chip *gc, unsigned long *mask, unsigned long *bits);
    int  (*set_config)(struct gpio_chip *gc, unsigned int offset, unsigned long config);
    int  (*to_irq)(struct gpio_chip *gc, unsigned int offset);
    void (*dbg_show)(struct seq_file *s, struct gpio_chip *gc);
    int  (*init_valid_mask)(struct gpio_chip *gc, unsigned long *valid_mask, unsigned int ngpios);
    int  (*add_pin_ranges)(struct gpio_chip *gc);
    int  (*en_hw_timestamp)(struct gpio_chip *gc, u32 offset, unsigned long flags);
    int  (*dis_hw_timestamp)(struct gpio_chip *gc, u32 offset, unsigned long flags);

    int            base;       /* first GPIO number (or -1 for dynamic) */
    u16            ngpio;      /* total number of lines this chip controls */
    u16            offset;     /* offset within multi-chip device */
    const char *const *names;  /* optional per-line name strings */
    bool           can_sleep;  /* true for I2C/SPI expanders */

#ifdef CONFIG_GPIOLIB_IRQCHIP
    struct gpio_irq_chip irq;  /* embedded irqchip support */
#endif
};
```

| الحقل المهم | المعنى |
|---|---|
| `base` | أول رقم GPIO في الـ global namespace (deprecated للقيم الثابتة) |
| `ngpio` | عدد الـ lines، الأقصى في gpio-reg هو 32 (بسبب `u32`) |
| `names` | أسماء بشرية لكل line، اختياري |
| `irq` | الـ embedded irqchip (يتفعل مع `CONFIG_GPIOLIB_IRQCHIP`) |

---

#### `struct gpio_irq_chip` — الـ IRQ Integration (في gpio/driver.h)

بيُضمَّن جوا `gpio_chip.irq`، بيربط الـ GPIO controller بالـ IRQ subsystem:

```c
struct gpio_irq_chip {
    struct irq_chip        *chip;          /* irq_chip ops pointer */
    struct irq_domain      *domain;        /* hwirq → linux irq mapping */
    irq_flow_handler_t      handler;       /* e.g. handle_level_irq */
    unsigned int            default_type;  /* IRQ_TYPE_* default trigger */
    irq_flow_handler_t      parent_handler;/* cascaded parent ISR */
    unsigned int            num_parents;
    unsigned int           *parents;       /* parent IRQ numbers */
    bool                    threaded;      /* nested threading? */
    bool                    initialized;
    /* ... */
};
```

**ملاحظة:** `gpio-reg` مش بيستخدم `gpio_irq_chip` مباشرة — بدل كده بيعمل mapping يدوي عبر `to_irq` callback + `irq_find_mapping()`.

---

### 2. رسم علاقات الـ Structs

```
Memory Layout of gpio_reg:
┌─────────────────────────────────────────────┐
│              struct gpio_reg                │
│ ┌─────────────────────────────────────────┐ │
│ │          struct gpio_chip gc            │ │  ← embedded at offset 0
│ │  label, base, ngpio, names              │ │
│ │  ops: get, set, get_direction, ...      │ │
│ │  *gpiodev ──────────────────────────────┼─┼──→ struct gpio_device (opaque, gpiolib)
│ │  *parent  ──────────────────────────────┼─┼──→ struct device
│ └─────────────────────────────────────────┘ │
│  spinlock_t lock                            │
│  u32 direction   (bitmask, fixed at init)   │
│  u32 out         (shadow register)          │
│  void __iomem *reg ─────────────────────────┼──→ [MMIO Hardware Register]
│  *irqdomain ────────────────────────────────┼──→ struct irq_domain
│  *irqs[]    ────────────────────────────────┼──→ int[ngpio] (hw or linux irq nums)
└─────────────────────────────────────────────┘

Macro: to_gpio_reg(gc) = container_of(gc, struct gpio_reg, gc)
       يرجع لـ gpio_reg كامل من pointer على الـ gc المُضمَّن
```

```
Pointer Graph:

  [Driver / Platform Code]
         │
         │ gpio_reg_init(dev, reg, ...)
         ▼
  ┌─────────────┐        embedded        ┌──────────────────┐
  │  gpio_reg   │◄──────────────────────►│   gpio_chip (gc) │
  │  (private)  │  container_of          │  (public API)    │
  └──────┬──────┘                        └────────┬─────────┘
         │                                        │
         │ r->reg                                 │ gc.gpiodev
         ▼                                        ▼
  [MMIO Register]                       ┌──────────────────┐
  (single u32 HW reg)                   │  gpio_device     │
                                        │  (gpiolib core)  │
         │ r->irqdomain                 └──────────────────┘
         ▼
  ┌──────────────┐
  │  irq_domain  │──→ irq_find_mapping(domain, hwirq) → linux irq#
  └──────────────┘

         │ r->irqs[]
         ▼
  [ int array: GPIO 0..N → irq/hwirq numbers ]
```

---

### 3. دورة حياة الـ gpio_reg

```
LIFECYCLE DIAGRAM
═════════════════

  [Platform/Driver Init]
         │
         ▼
  gpio_reg_init(dev, reg, base, num, label,
                direction, def_out, names, irqdom, irqs)
         │
         ├─ kzalloc / devm_kzalloc → struct gpio_reg *r
         ├─ spin_lock_init(&r->lock)
         ├─ r->direction = direction   (fixed, never changes)
         ├─ r->out       = def_out     (initial output state)
         ├─ r->reg       = reg         (MMIO pointer)
         ├─ r->irqs      = irqs
         ├─ fill r->gc.ops (get, set, direction_*, to_irq, ...)
         │
         ▼
  gpiochip_add_data(&r->gc, r)   OR   devm_gpiochip_add_data(dev, &r->gc, r)
         │
         │   ← gpiolib registers the chip
         │   ← assigns GPIO numbers (base..base+ngpio-1)
         │   ← creates sysfs entries
         │
         ▼
  [CHIP ACTIVE — normal operation]
         │
         ├── gpio_reg_get()        ← reads HW reg (double-read for inputs)
         ├── gpio_reg_set()        ← spin_lock → update r->out → writel → unlock
         ├── gpio_reg_set_multiple() ← same, but bitwise mask operation
         ├── gpio_reg_get_direction() ← returns from r->direction bitmask (no HW access)
         ├── gpio_reg_direction_input/output() ← validates against fixed direction
         ├── gpio_reg_to_irq()     ← looks up r->irqs[offset], optionally maps via domain
         │
         ▼
  [SUSPEND / RESUME]
         │
         ├── suspend: HW register may lose state (power-gated)
         │
         └── gpio_reg_resume(gc)
                 │
                 ├─ spin_lock_irqsave
                 ├─ writel_relaxed(r->out, r->reg)  ← restore shadow → HW
                 └─ spin_unlock_irqrestore

         ▼
  [REMOVAL]
         │
         ├── If devm: automatic cleanup on device removal
         ├── gpiochip_remove(&r->gc) — unregisters from gpiolib
         └── kfree(r) (if non-devm)
```

---

### 4. Call Flow Diagrams

#### gpio_reg_set() — كتابة قيمة على output GPIO

```
consumer calls: gpiod_set_value(desc, value)
    │
    ▼
gpiolib core: gpio_chip->set(gc, offset, value)
    │
    ▼
gpio_reg_set(gc, offset, value)
    │
    ├─ r = to_gpio_reg(gc)          // recover full gpio_reg from embedded gc
    ├─ mask = BIT(offset)
    │
    ├─ spin_lock_irqsave(&r->lock)  // disable IRQs, take lock
    │
    ├─ val = r->out                 // read shadow (NOT hardware)
    ├─ if (value) val |= mask       // set bit
    │  else       val &= ~mask      // clear bit
    ├─ r->out = val                 // update shadow
    ├─ writel_relaxed(val, r->reg)  // commit to hardware MMIO register
    │
    └─ spin_unlock_irqrestore()     // release lock, restore IRQs
```

#### gpio_reg_get() — قراءة قيمة GPIO

```
consumer calls: gpiod_get_value(desc)
    │
    ▼
gpiolib core: gpio_chip->get(gc, offset)
    │
    ▼
gpio_reg_get(gc, offset)
    │
    ├─ r = to_gpio_reg(gc)
    ├─ mask = BIT(offset)
    │
    ├─ if (r->direction & mask)     // INPUT line?
    │       readl_relaxed(r->reg)   // dummy first read (latch workaround)
    │       val = readl_relaxed(r->reg)  // actual read
    │  else
    │       val = r->out            // OUTPUT: read from shadow, not HW
    │
    └─ return !!(val & mask)        // extract single bit
```

#### gpio_reg_to_irq() — تحويل GPIO إلى IRQ number

```
consumer calls: gpiod_to_irq(desc)
    │
    ▼
gpiolib core: gpio_chip->to_irq(gc, offset)
    │
    ▼
gpio_reg_to_irq(gc, offset)
    │
    ├─ r = to_gpio_reg(gc)
    ├─ irq = r->irqs[offset]        // get raw irq/hwirq number
    │
    ├─ if (irq >= 0 && r->irqdomain)
    │       irq = irq_find_mapping(r->irqdomain, irq)  // hwirq → linux virq
    │
    └─ return irq
```

#### gpio_reg_init() — التهيئة الكاملة

```
platform driver probe()
    │
    ▼
gpio_reg_init(dev, reg, base, num, label,
              direction, def_out, names, irqdom, irqs)
    │
    ├─ allocate struct gpio_reg (devm or kzalloc)
    ├─ spin_lock_init
    ├─ fill r->gc fields:
    │       .label, .base, .ngpio, .names
    │       .get_direction  = gpio_reg_get_direction
    │       .direction_input  = gpio_reg_direction_input
    │       .direction_output = gpio_reg_direction_output
    │       .get            = gpio_reg_get
    │       .set            = gpio_reg_set
    │       .set_multiple   = gpio_reg_set_multiple
    │       .to_irq         = gpio_reg_to_irq  (only if irqs != NULL)
    ├─ r->direction = direction
    ├─ r->out       = def_out
    ├─ r->reg       = reg (MMIO)
    ├─ r->irqs      = irqs
    │
    ├─ gpiochip_add_data / devm_gpiochip_add_data
    │       │
    │       ▼
    │   gpiolib core:
    │       ← validate gc fields
    │       ← assign GPIO numbers
    │       ← create gpio_device
    │       ← register with sysfs / debugfs
    │       ← setup irqchip if gc->irq initialized
    │
    └─ return &r->gc  (or ERR_PTR on failure)
```

---

### 5. استراتيجية الـ Locking

#### الـ Lock الوحيد: `r->lock` (spinlock)

```
┌─────────────────────────────────────────────────────────┐
│  spinlock_t r->lock                                     │
│                                                         │
│  يحمي:                                                  │
│    • r->out  (shadow register value)                    │
│    • writel() إلى r->reg (MMIO write)                   │
│                                                         │
│  يُستخدم في:                                            │
│    • gpio_reg_set()          spin_lock_irqsave          │
│    • gpio_reg_set_multiple() spin_lock_irqsave          │
│    • gpio_reg_resume()       spin_lock_irqsave          │
└─────────────────────────────────────────────────────────┘
```

**ليه `spin_lock_irqsave` وليس `spin_lock`؟**
لأن الـ `to_irq` callback ممكن يُستدعى من interrupt context، وبالتالي لو الـ set() اتعملت من process context وفي نفس الوقت جه interrupt يحاول يعمل write، يحصل deadlock. الـ `irqsave` بيعطل الـ IRQs على الـ CPU الحالية طول فترة الـ critical section.

#### اللي مش محتاج lock:

| العملية | السبب |
|---|---|
| `gpio_reg_get_direction()` | بيقرأ `r->direction` اللي read-only بعد الـ init |
| `gpio_reg_direction_input/output()` | نفس السبب، `r->direction` ثابت |
| `gpio_reg_get()` — input path | بيقرأ من HW register فقط، مفيش كتابة على shared state |
| `gpio_reg_get()` — output path | بيقرأ `r->out` بس، والقراءة atomic على 32-bit aligned |
| `gpio_reg_to_irq()` | بيقرأ فقط من `r->irqs[]` و `r->irqdomain` اللي ما بتتغيرش |

#### ترتيب الـ Locks (Lock Ordering):

مفيش nested locking في الـ driver ده. الـ `r->lock` هو الـ lock الوحيد، وده بيبسط الموضوع ويمنع الـ deadlock من الأساس.

الـ gpiolib core نفسه عنده `gpio_device` locks خاصة بيه، بس الـ gpio-reg مش بيمسها مباشرة — الـ core بيستدعي الـ ops والـ ops بتأخد `r->lock` بس.

```
Lock hierarchy (single level):
  gpiolib internal lock (gpio_device)
      └─ r->lock (gpio_reg spinlock)   ← taken inside ops callbacks
         └─ (nothing deeper)
```

**الـ double-read workaround:**
في `gpio_reg_get()` للـ input GPIOs، الـ code بيعمل `readl_relaxed()` مرتين:

```c
readl_relaxed(r->reg);          /* dummy read — clears latch */
val = readl_relaxed(r->reg);    /* actual value */
```

ده مش محتاج lock لأن الـ HW register قراءته idempotent، ومفيش shared mutable state بيتغير هنا.
## Phase 4: شرح الـ Functions

### جدول الـ API — Cheatsheet

| Function | Signature | الدور |
|---|---|---|
| `gpio_reg_init` | `struct gpio_chip *gpio_reg_init(dev, reg, base, num, label, direction, def_out, names, irqdom, irqs)` | تسجيل register جديد كـ gpio_chip في النظام |
| `gpio_reg_resume` | `int gpio_reg_resume(struct gpio_chip *gc)` | إعادة كتابة output register بعد suspend |
| `gpio_reg_get_direction` | `static int gpio_reg_get_direction(gc, offset)` | إرجاع اتجاه line معينة (input/output) |
| `gpio_reg_direction_input` | `static int gpio_reg_direction_input(gc, offset)` | محاولة تعيين line كـ input |
| `gpio_reg_direction_output` | `static int gpio_reg_direction_output(gc, offset, value)` | محاولة تعيين line كـ output مع set value |
| `gpio_reg_get` | `static int gpio_reg_get(gc, offset)` | قراءة قيمة line واحدة من الـ register |
| `gpio_reg_set` | `static int gpio_reg_set(gc, offset, value)` | كتابة قيمة line واحدة في الـ register |
| `gpio_reg_set_multiple` | `static int gpio_reg_set_multiple(gc, mask, bits)` | كتابة عدة lines دفعة واحدة |
| `gpio_reg_to_irq` | `static int gpio_reg_to_irq(gc, offset)` | تحويل GPIO offset لـ Linux IRQ number |

---

### الـ Internal Struct الأساسية

```c
struct gpio_reg {
    struct gpio_chip gc;       /* embedded gpio_chip — يُعرَّف عبره في gpiolib */
    spinlock_t lock;           /* يحمي r->out والـ MMIO write */
    u32 direction;             /* bitmask: 1 = input, 0 = output — fixed at init */
    u32 out;                   /* shadow register للـ output — لا نقرأ من HW */
    void __iomem *reg;         /* عنوان الـ MMIO register */
    struct irq_domain *irqdomain; /* optional — لترجمة HW IRQ لـ Linux IRQ */
    const int *irqs;           /* array: irq number لكل GPIO line */
};

#define to_gpio_reg(x) container_of(x, struct gpio_reg, gc)
```

**الـ `gpio_reg`** هو الـ private context الخاص بالـ driver. الـ `gpio_chip` مضمّن في أوله، فبمجرد ما gpiolib بتنادي أي callback بيعدي `gc*`، بنعمل `to_gpio_reg` نرجع الـ `gpio_reg` كامل.

---

### Category 1: Registration & Initialization

#### `gpio_reg_init`

```c
struct gpio_chip *gpio_reg_init(
    struct device     *dev,       /* الـ parent device، أو NULL */
    void __iomem      *reg,       /* عنوان الـ MMIO register */
    int                base,      /* أول GPIO number، أو -1 للـ dynamic allocation */
    int                num,       /* عدد الـ GPIO lines، max 32 */
    const char        *label,     /* اسم الـ chip في sysfs/debugfs */
    u32                direction, /* bitmask: 1 = input، 0 = output */
    u32                def_out,   /* القيمة الابتدائية للـ output lines */
    const char *const *names,     /* أسماء الـ lines، أو NULL */
    struct irq_domain *irqdom,    /* IRQ domain، أو NULL */
    const int         *irqs       /* array of IRQ numbers، أو NULL */
);
```

**الدور:**
بتعمل `alloc` للـ `gpio_reg`، بتملا الـ `gpio_chip` callbacks، وبتسجّل الـ chip في gpiolib. لو `dev != NULL` بتستخدم `devm_kzalloc` و `devm_gpiochip_add_data` عشان كل الـ cleanup يتعمل automatically لما الـ device بتتفصل. لو `dev == NULL` بتستخدم `kzalloc` و `gpiochip_add_data` وإدارة الـ lifetime بتبقى مسؤولية الـ caller.

**Parameters:**

| Parameter | الوصف |
|---|---|
| `dev` | الـ parent device للـ devm management. لو `NULL` فالـ caller مسؤول عن الـ lifetime |
| `reg` | الـ MMIO address للـ single register اللي بيتحكم في كل الـ GPIO lines |
| `base` | أول GPIO number في النظام. `-1` يعني gpiolib بيختار dynamic |
| `num` | عدد الـ lines، maximum 32 لأن الـ register 32-bit |
| `label` | string بيظهر في `/sys/kernel/debug/gpio` وفي الـ logs |
| `direction` | bitmask ثابت — كل bit بيحدد اتجاه الـ line المقابلة إلى الأبد |
| `def_out` | القيمة اللي بتتكتب في الـ register من أول لما الـ chip بيتسجّل |
| `names` | array of strings طول `num`، بيظهر في debugfs للـ human-readable identification |
| `irqdom` | لو مش NULL، الـ `irqs` بيكون HW IRQ numbers وبنترجمها عبر هذا الـ domain |
| `irqs` | array of IRQ mappings لكل line. لو `NULL` الـ `to_irq` callback مش بيتسجّل |

**Return value:**
بترجع `struct gpio_chip *` على الـ embedded `gc` جوا الـ `gpio_reg`. لو في error بترجع `ERR_PTR(-ENOMEM)` لو فشل الـ alloc، أو `ERR_PTR(ret)` لو فشل `gpiochip_add_data`. الـ caller يتحقق بـ `IS_ERR()`.

**Key details:**
- الـ `direction` bitmask **ثابت** — مفيش function بتغيره بعد الـ init. الـ driver صُمِّم للـ hardware اللي فيها كل GPIO line اتجاهها مفصول physically.
- الـ `def_out` بيتحفظ في `r->out` (الـ shadow) لكنه **ما بيتكتبش في الـ register** هنا — الكتابة الفعلية بتحصل أول ما gpiolib بيطلب `set` أو في `gpio_reg_resume`.
- الـ `spin_lock_init` بيحصل هنا قبل أي callback يتسجّل.
- لو `irqs != NULL` بس `irqdom == NULL`، الـ `irqs` بيُعامَل كـ Linux IRQ numbers مباشرة.

**Pseudocode flow:**
```
gpio_reg_init():
    if dev:
        r = devm_kzalloc(dev, sizeof(*r))
    else:
        r = kzalloc(sizeof(*r))

    if !r: return ERR_PTR(-ENOMEM)

    spin_lock_init(&r->lock)

    /* fill gpio_chip */
    r->gc.label            = label
    r->gc.get_direction    = gpio_reg_get_direction
    r->gc.direction_input  = gpio_reg_direction_input
    r->gc.direction_output = gpio_reg_direction_output
    r->gc.set              = gpio_reg_set
    r->gc.get              = gpio_reg_get
    r->gc.set_multiple     = gpio_reg_set_multiple
    if irqs:
        r->gc.to_irq       = gpio_reg_to_irq
    r->gc.base  = base
    r->gc.ngpio = num
    r->gc.names = names

    /* fill private fields */
    r->direction  = direction
    r->out        = def_out
    r->reg        = reg
    r->irqs       = irqs
    r->irqdomain  = irqdom

    if dev:
        ret = devm_gpiochip_add_data(dev, &r->gc, r)
    else:
        ret = gpiochip_add_data(&r->gc, r)

    return ret ? ERR_PTR(ret) : &r->gc
```

**Who calls it:**
الـ platform driver أو الـ SoC init code في `probe()`. عادةً بنلاقيها في كود الـ board support اللي بيعمل map لـ FPGA registers أو legacy system controllers كـ single GPIO register.

---

### Category 2: Power Management

#### `gpio_reg_resume`

```c
int gpio_reg_resume(struct gpio_chip *gc);
```

**الدور:**
بعد الـ system resume من sleep، الـ MMIO registers بترجع لقيمها الابتدائية (أو undefined). الـ function دي بتكتب الـ shadow value `r->out` في الـ register تاني عشان تعيد حالة الـ output lines زي ما كانت قبل الـ suspend.

**Parameters:**

| Parameter | الوصف |
|---|---|
| `gc` | الـ `gpio_chip` pointer اللي gpiolib بتعدّيه — بنستخدم `to_gpio_reg` نرجع الـ `gpio_reg` |

**Return value:**
دايمًا بترجع `0`. مفيش error paths.

**Key details:**
- بتأخذ الـ `spinlock` بـ `spin_lock_irqsave` عشان تحمي الـ `r->out` من أي تعديل concurrent.
- الـ `writel_relaxed` بيكتب 32-bit في الـ register — `relaxed` يعني مفيش memory barrier إضافي، وده كافي هنا لأن الـ locking بيوفر الـ ordering اللازم.
- الـ function دي **بتكتب الـ output lines بس** — الـ input lines مش بتتأثر لأنها بتتقرأ من الـ HW مباشرة.
- الـ caller (الـ platform driver) المفروض يناديها من `dev_pm_ops.resume` أو `dev_pm_ops.restore`.

```c
/* example usage in driver */
static int mydrv_resume(struct device *dev)
{
    struct mydrv *priv = dev_get_drvdata(dev);
    return gpio_reg_resume(priv->gc);
}
```

**Who calls it:**
الـ platform driver اللي استخدم `gpio_reg_init`. gpiolib نفسها **ما بتنادي عليها** — الـ driver مسؤول عن ندائها في الـ resume path.

---

### Category 3: Direction Control (Static Enforcement)

#### `gpio_reg_get_direction`

```c
static int gpio_reg_get_direction(struct gpio_chip *gc, unsigned offset);
```

**الدور:**
بترجع اتجاه الـ GPIO line المحددة بناءً على الـ `direction` bitmask اللي اتحدد في وقت الـ init. مفيش قراءة من الـ hardware — الاتجاه ثابت وبيتحفظ في الـ software.

**Parameters:**

| Parameter | الوصف |
|---|---|
| `gc` | الـ gpio_chip pointer |
| `offset` | رقم الـ GPIO line (0-based) داخل الـ chip |

**Return value:**
- `GPIO_LINE_DIRECTION_IN` (= 1) لو الـ bit هو 1
- `GPIO_LINE_DIRECTION_OUT` (= 0) لو الـ bit هو 0

**Key details:**
بتعمل `r->direction & BIT(offset)` — عملية بسيطة بدون locking لأن `r->direction` read-only بعد الـ init.

---

#### `gpio_reg_direction_input`

```c
static int gpio_reg_direction_input(struct gpio_chip *gc, unsigned offset);
```

**الدور:**
الـ gpiolib بتنادي عليها لما consumer بيطلب يحوّل line لـ input. لأن الاتجاه ثابت، بترجع نجاح بس لو الـ line أصلًا input، وترجع `-ENOTSUPP` لو هي output.

**Return value:**
- `0` لو الـ line فعلًا input
- `-ENOTSUPP` لو الـ line output — مش ممكن تتغير

---

#### `gpio_reg_direction_output`

```c
static int gpio_reg_direction_output(struct gpio_chip *gc, unsigned offset, int value);
```

**الدور:**
بتحاول تعيين line كـ output. لو الـ line أصلًا input بترجع `-ENOTSUPP`. لو هي output بتنادي `gc->set` عشان تكتب الـ value الابتدائية وبترجع `0`.

**Parameters:**

| Parameter | الوصف |
|---|---|
| `gc` | الـ gpio_chip pointer |
| `offset` | رقم الـ line |
| `value` | القيمة الابتدائية للـ output (0 أو 1) |

**Key details:**
الـ locking بيحصل داخل `gpio_reg_set` اللي بتنادي عليها، مش هنا.

---

### Category 4: Read / Write Operations

#### `gpio_reg_get`

```c
static int gpio_reg_get(struct gpio_chip *gc, unsigned offset);
```

**الدور:**
بتقرأ قيمة line واحدة. للـ input lines بتعمل double-read من الـ hardware لأن بعض الـ registers بتعمل latch بعد أول قراءة. للـ output lines بترجع القيمة من الـ shadow `r->out` مباشرة لأن الـ hardware مش بيعكس الـ output value قابل للقراءة.

**Parameters:**

| Parameter | الوصف |
|---|---|
| `gc` | الـ gpio_chip pointer |
| `offset` | رقم الـ GPIO line |

**Return value:**
`0` أو `1` — قيمة الـ bit المقابلة.

**Key details:**
```c
if (r->direction & mask) {
    readl_relaxed(r->reg);      /* dummy read — flush latch */
    val = readl_relaxed(r->reg); /* actual value */
} else {
    val = r->out;               /* output: من الـ shadow */
}
return !!(val & mask);
```
- الـ double-read ضروري لبعض الـ FPGA/CPLD registers اللي بتعمل sample للـ input بعد أول read access.
- مفيش locking هنا — الـ 32-bit read atomic على كل الـ architectures المعتمدة، والـ `r->out` write-protected بالـ spinlock في `gpio_reg_set`.

---

#### `gpio_reg_set`

```c
static int gpio_reg_set(struct gpio_chip *gc, unsigned int offset, int value);
```

**الدور:**
بتكتب قيمة line واحدة. بتعمل read-modify-write على الـ shadow `r->out` تحت الـ spinlock، وبعدين بتكتب الـ register كامل.

**Parameters:**

| Parameter | الوصف |
|---|---|
| `gc` | الـ gpio_chip pointer |
| `offset` | رقم الـ line |
| `value` | القيمة الجديدة (0 أو 1) |

**Return value:**
دايمًا `0`.

**Key details:**
- `spin_lock_irqsave` لأن الـ set ممكن يتنادى من IRQ context.
- الـ `writel_relaxed` بيكتب الـ 32-bit register كامل — مفيش partial write — ده سبب وجود الـ shadow `r->out`.
- الـ pattern:
```c
spin_lock_irqsave(&r->lock, flags);
val = r->out;
if (value)
    val |= BIT(offset);
else
    val &= ~BIT(offset);
r->out = val;
writel_relaxed(val, r->reg);
spin_unlock_irqrestore(&r->lock, flags);
```

---

#### `gpio_reg_set_multiple`

```c
static int gpio_reg_set_multiple(
    struct gpio_chip *gc,
    unsigned long    *mask,  /* بitmask للـ lines المراد تعديلها */
    unsigned long    *bits   /* القيم الجديدة للـ lines */
);
```

**الدور:**
بتعدّل عدة GPIO lines في operation واحدة. أكفأ من تكرار `gpio_reg_set` لكل line على حدة لأن الـ MMIO write بتحصل مرة واحدة.

**Parameters:**

| Parameter | الوصف |
|---|---|
| `gc` | الـ gpio_chip pointer |
| `mask` | الـ bits اللي عايز تعدّلها |
| `bits` | القيم الجديدة — بتُطبَّق بس على الـ bits الموجودة في `mask` |

**Return value:**
دايمًا `0`.

**Key details:**
```c
spin_lock_irqsave(&r->lock, flags);
r->out = (r->out & ~*mask) | (*bits & *mask);
writel_relaxed(r->out, r->reg);
spin_unlock_irqrestore(&r->lock, flags);
```
المعادلة الكلاسيكية: `(old & ~mask) | (new & mask)` — بتحفظ الـ bits اللي مش داخلة في الـ mask وبتحطّ الجديدة.

---

### Category 5: IRQ Mapping

#### `gpio_reg_to_irq`

```c
static int gpio_reg_to_irq(struct gpio_chip *gc, unsigned offset);
```

**الدور:**
بتحوّل GPIO line offset لـ Linux IRQ number. لو في `irqdomain` بتستخدم `irq_find_mapping` ترجع الـ Linux IRQ المقابل للـ HW IRQ number. لو مفيش `irqdomain`، الـ `irqs[offset]` نفسه هو الـ Linux IRQ number.

**Parameters:**

| Parameter | الوصف |
|---|---|
| `gc` | الـ gpio_chip pointer |
| `offset` | رقم الـ GPIO line |

**Return value:**
الـ Linux IRQ number، أو قيمة سالبة لو مش موجود mapping.

**Key details:**
```c
int irq = r->irqs[offset];
if (irq >= 0 && r->irqdomain)
    irq = irq_find_mapping(r->irqdomain, irq);
return irq;
```
- الـ `irq_find_mapping` بتبحث في الـ IRQ domain's radix tree عن الـ mapping. لو مش موجود بترجع `0` (اللي هو `IRQ_NOTCONNECTED` effectively).
- الـ callback دي بتتسجّل في `r->gc.to_irq` بس لو `irqs != NULL` — لو الـ chip مش ليها interrupt support، الـ pointer بيفضل NULL وgpiolib ما بتستدعيهاش.

---

### ملخص تصميمي: ليه الـ Shadow Register؟

**الـ `r->out`** موجود لسبب جوهري: الـ hardware register ده write-only في أغلب الـ use cases. الـ GPIO controller ده مصمّم للـ legacy systems اللي فيها الـ output lines ما بتنعكسش في الـ read path.

```
Write path:
  gpio_reg_set(gc, 5, 1)
      → r->out |= BIT(5)    ← shadow update
      → writel(r->out, reg) ← write كامل الـ 32-bit register

Read path (output):
  gpio_reg_get(gc, 5)
      → return !!(r->out & BIT(5))  ← من الـ shadow — ما بنقرأش HW

Read path (input):
  gpio_reg_get(gc, 3)     [r->direction & BIT(3) = 1]
      → readl(reg)         ← dummy — flush latch
      → val = readl(reg)   ← real read من HW
      → return !!(val & BIT(3))
```

الـ `spinlock` بيحمي consistency الـ shadow مع الـ hardware، خصوصًا في الـ SMP systems أو لما الـ interrupt handler بيعمل `set` في نفس وقت الـ process context.
## Phase 5: دليل الـ Debugging الشامل

الـ `gpio-reg` driver بسيط جداً — register واحدة بس، لكن لما حاجة بتغلط فيه بيكون صعب تعرف مين المشكلة: الـ hardware، الـ Device Tree، ولا الـ driver نفسه. الدليل ده هيخليك تعرف تفرق.

---

### Software Level

#### 1. Debugfs

الـ GPIO subsystem بيعمل entries تحت `/sys/kernel/debug/gpio` تلقائياً لما يكون `CONFIG_DEBUG_FS=y`.

```bash
# اقرأ حالة كل GPIO chips المسجلة في النظام
cat /sys/kernel/debug/gpio
```

**مثال output:**

```
gpiochip0: GPIOs 480-511, parent: platform/soc:gpio-leds, soc-leds:
 gpio-480 (                    |led-red         ) out lo
 gpio-481 (                    |led-green       ) out hi
 gpio-482 (                    |reset           ) in  lo IRQ
```

**تفسير الـ fields:**

| Field | المعنى |
|-------|--------|
| `gpiochip0` | اسم الـ chip، غالباً الـ `label` اللي اتبعت لـ `gpio_reg_init` |
| `GPIOs 480-511` | الـ base + num اللي اتبعتوا |
| `out lo / out hi` | الـ output GPIOs — قيمتها من `r->out` المحفوظة في الـ struct |
| `in  lo` | الـ input GPIOs — بتتقرأ من الـ register فعلاً |
| `IRQ` | الـ GPIO ده ليه irq mapping |

```bash
# لو عايز تعرف GPIO chip معين بالاسم
grep -A 20 "your-label" /sys/kernel/debug/gpio
```

الـ gpio-reg بيحفظ قيم الـ output في `r->out` — مش بيقرأها من الـ register. يعني لو قرأت `out lo` ومش متأكد، الرقم ده من الـ shadow register الـ software مش من الـ hardware فعلاً.

---

#### 2. Sysfs

```bash
# اعرف رقم الـ GPIO chip (base number)
ls /sys/class/gpio/

# مثال: lو الـ base = 480
echo 480 > /sys/class/gpio/export
cat /sys/class/gpio/gpio480/direction    # in أو out
cat /sys/class/gpio/gpio480/value        # 0 أو 1

# اعرف اسم الـ chip
cat /sys/class/gpio/gpiochip480/label
cat /sys/class/gpio/gpiochip480/ngpio
cat /sys/class/gpio/gpiochip480/base
```

**مهم:** الـ gpio-reg بيستخدم `direction` bitmask ثابتة — محدش يقدر يغير direction من الـ sysfs. لو حاولت:

```bash
echo out > /sys/class/gpio/gpio482/direction
# النتيجة: bash: echo: write error: Operation not supported
```

ده لأن `gpio_reg_direction_output` بيرجع `-ENOTSUPP` لو الـ bit في `r->direction` = 1 (input).

---

#### 3. Ftrace

```bash
# فعّل GPIO tracepoints
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable

# أو tracepoints محددة
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_value/enable
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_direction/enable

# ابدأ الـ trace
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... شغّل الـ use case
echo 0 > /sys/kernel/debug/tracing/tracing_on

# اقرأ النتيجة
cat /sys/kernel/debug/tracing/trace
```

**مثال output:**

```
          <idle>-0     [001] d... 12345.678901: gpio_value: 480 set 1
     systemd-1    [000] d... 12345.679001: gpio_value: 481 get 0
```

لو مشوفتش أي events رغم إنك بتكتب على GPIO — يبقى على الأغلب الـ `set` function مش بتتكال أصلاً، ابحث في الـ consumer code.

---

#### 4. Dynamic Debug / Printk

الـ gpio-reg driver نفسه ما فيهوش `dev_dbg` calls — بسيط جداً. لكن ممكن تشغّل الـ dynamic debug على مستوى الـ GPIO subsystem:

```bash
# شغّل debug messages للـ gpiochip core
echo 'file drivers/gpio/gpiolib.c +p' > /sys/kernel/debug/dynamic_debug/control

# شغّل debug لكل الـ gpio drivers
echo 'file drivers/gpio/* +p' > /sys/kernel/debug/dynamic_debug/control

# شغّل debug لـ gpio-reg تحديداً
echo 'file drivers/gpio/gpio-reg.c +p' > /sys/kernel/debug/dynamic_debug/control

# تحقق إيه اللي اتفعّل
cat /sys/kernel/debug/dynamic_debug/control | grep gpio
```

لو عايز تضيف `dev_dbg` في الـ driver وقت الـ development:

```c
/* في gpio_reg_set — بعد الـ writel */
dev_dbg(gc->parent, "GPIO%u set to %d, reg now 0x%08x\n",
        offset, value, r->out);
```

---

#### 5. Kernel Config Options للـ Debugging

| Config | الغرض |
|--------|--------|
| `CONFIG_DEBUG_GPIO` | يفعّل extra validation في gpiolib — الأهم |
| `CONFIG_GPIO_SYSFS` | يظهر الـ GPIOs في `/sys/class/gpio` |
| `CONFIG_DEBUG_FS` | يفعّل `/sys/kernel/debug/gpio` |
| `CONFIG_GPIOLIB_IRQCHIP` | يفعّل IRQ integration debugging |
| `CONFIG_DEBUG_SPINLOCK` | يكتشف الـ spinlock misuse في `gpio_reg_set` |
| `CONFIG_LOCKDEP` | يكتشف الـ lock ordering violations |
| `CONFIG_PROVE_LOCKING` | مع LOCKDEP — أقوى |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل `dev_dbg` dynamic activation |
| `CONFIG_IRQ_DOMAIN_DEBUG` | يساعد لو فيه مشكلة في الـ irq mapping |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "GPIO|GPIOLIB" | grep -v "^#"
```

---

#### 6. أدوات محددة للـ GPIO Subsystem

```bash
# gpiodetect: اعرف كل gpio chips
gpiodetect
# مثال output:
# gpiochip0 [soc-leds] (32 lines)

# gpioinfo: تفاصيل كل line
gpioinfo gpiochip0

# gpioget: اقرأ قيمة
gpioget gpiochip0 2

# gpioset: اكتب قيمة
gpioset gpiochip0 2=1

# gpiomon: راقب التغييرات على GPIO input
gpiomon --num-events=5 gpiochip0 2
```

هذه الأدوات من package `gpiod` وبتتكلم مع الـ kernel عبر character device `/dev/gpiochip0`.

---

#### 7. جدول الـ Error Messages الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---------------------|--------|-------|
| `gpio-reg: Unable to register gpiochip` | فشل `gpiochip_add_data` | تحقق من تعارض الـ base number، أو مشكلة في الـ devm allocation |
| `Operation not supported` (ENOTSUPP) | محاولة تغيير direction لـ GPIO ثابت | الـ direction في `gpio-reg` ثابت — راجع الـ `direction` bitmask اللي بعتها |
| `No such device` (ENODEV) | محاولة export GPIO مش موجود | تأكد إن الـ base + offset ضمن الـ range المسجل |
| `gpio_chip->get returned error -EINVAL` | الـ get function فشلت | نادر في gpio-reg، لكن ممكن يحصل لو الـ IO mapping غلط |
| `irq: no irq domain found` | الـ irqdomain pointer غلط أو NULL | تحقق من الـ `irqdom` parameter اللي بعته لـ `gpio_reg_init` |
| `kernel BUG at lib/iomap.c` | الـ IO address غلط أو unmapped | تأكد إن `ioremap` اتعمل قبل ما تبعت الـ `reg` pointer |
| `spinlock bad magic` | corruption في الـ spinlock | memory corruption — فعّل `CONFIG_DEBUG_SPINLOCK` |

---

#### 8. Strategic Points لـ `dump_stack()` و `WARN_ON()`

```c
/* في gpio_reg_init — تحقق من الـ parameters قبل الاستخدام */
if (WARN_ON(!reg))
    return ERR_PTR(-EINVAL);

if (WARN_ON(num > 32 || num <= 0))
    return ERR_PTR(-EINVAL);

/* في gpio_reg_set — تحقق إن الـ offset صح */
if (WARN_ON(offset >= gc->ngpio))
    return -EINVAL;

/* في gpio_reg_get — إضافة تحقق على الـ direction */
if (WARN_ON(!(r->direction & mask) && /* trying to read output */
            (r->direction & mask) == 0)) {
    /* output GPIO — بنرجع الـ shadow value */
}
```

لو الـ driver بيتصرف غريب وقت الـ resume:

```c
int gpio_reg_resume(struct gpio_chip *gc)
{
    struct gpio_reg *r = to_gpio_reg(gc);
    /* أضف هنا لو عايز تتأكد إن الـ write وصل */
    u32 verify;
    unsigned long flags;

    spin_lock_irqsave(&r->lock, flags);
    writel_relaxed(r->out, r->reg);
    /* للـ debugging فقط */
    verify = readl_relaxed(r->reg);
    if (WARN_ON((verify & ~r->direction) != (r->out & ~r->direction)))
        dump_stack();
    spin_unlock_irqrestore(&r->lock, flags);

    return 0;
}
```

---

### Hardware Level

#### 1. التحقق إن حالة الـ Hardware بتوافق الـ Kernel

الـ gpio-reg بيكتب على register واحدة بس. الـ shadow value المحفوظة في `r->out` المفروض تساوي اللي في الـ hardware.

```bash
# الخطوة الأولى: اعرف الـ physical address من الـ DT أو platform data
# مثال: register عند 0x10020000

# قارن الـ shadow value (من debugfs) مع الـ hardware (من devmem2)
cat /sys/kernel/debug/gpio
# gpio-480 (led-red) out hi  → shadow = 1

# اقرأ الـ register من الـ hardware مباشرةً
devmem2 0x10020000 w
# Value at address 0x10020000: 0x00000003
# bit 0 = 1 (gpio-480 = hi), bit 1 = 1 (gpio-481 = hi)
```

لو الـ shadow value مختلفة عن الـ hardware — يبقى فيه مشكلة في الـ write path أو الـ hardware بيتجاهل الـ write.

---

#### 2. Register Dump Techniques

```bash
# devmem2 — الأسهل، يحتاج package منفصل
devmem2 <physical_addr> w        # read 32-bit word
devmem2 <physical_addr> w <val>  # write 32-bit word

# /dev/mem — يحتاج CONFIG_DEVMEM=y وصلاحيات root
dd if=/dev/mem bs=4 count=1 skip=$((0x10020000 / 4)) 2>/dev/null | xxd

# io utility (من package hwtools)
io -4 0x10020000                 # read
io -4 0x10020000 0x00000003      # write

# من داخل kernel module للـ debugging
void __iomem *base = ioremap(0x10020000, 4);
pr_info("GPIO reg = 0x%08x\n", readl(base));
iounmap(base);
```

**مهم للـ double-read:** الـ gpio-reg بيعمل double-read للـ input GPIOs عشان بعض الـ hardware بيعمل latch بعد أول read. لو بتعمل `devmem2` مرة واحدة ممكن تاخد قيمة قديمة — اقرأ مرتين:

```bash
devmem2 0x10020000 w  # dummy read
devmem2 0x10020000 w  # القيمة الحقيقية
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

لما تستخدم logic analyzer على GPIO output من gpio-reg:

- **Signal to probe:** الـ GPIO pin المقابل للـ bit اللي بتكتب عليه
- **Trigger:** rising/falling edge على الـ GPIO
- **توقعات timing:** الـ `writel_relaxed` مش بيعمل memory barrier — يعني ممكن يكون فيه تأخير صغير بين الـ kernel call والـ pin change
- **لو الـ signal مش بيتغير:** تحقق من إن الـ bit direction = 0 (output) وإن الـ GPIO مش masked

**للـ input GPIOs:**
- استخدم oscilloscope تشوف الـ glitches — خصوصاً لو بتشتكي من قراءات خاطئة
- الـ double-read اللي بتعمله الـ driver مش مع debouncing — الـ glitches الأقل من cycle time ممكن تتقرأ بشكل خاطئ

---

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة الـ Hardware | الـ Pattern في الـ Log | التشخيص |
|--------------------|----------------------|----------|
| الـ register مش mapped | `kernel panic - not syncing: Unhandled fault` أو `Unable to handle kernel paging request` | الـ `ioremap` أو الـ physical address غلط |
| الـ clock للـ GPIO block مش مشغّل | GPIO بيكتب بدون error لكن الـ pin مش بيتغير | تحقق من الـ clock enable registers قبل استخدام الـ GPIO |
| الـ power domain مش enabled | device مش بيتجاوب | ابحث عن `pm_runtime` warnings في الـ log |
| تعارض في الـ pin mux | GPIO بيكتب لكن function تانية شايلة الـ pin | ابحث عن `pinctrl: pin X already requested` |
| الـ voltage level مش صح | الـ signal low عمياً أو floating | scope على الـ pin — مش مشكلة kernel |
| الـ output لا أثر ليه | bit في direction = 1 (input) عن طريق الخطأ | `cat /sys/kernel/debug/gpio` وتحقق direction |

---

#### 5. Device Tree Debugging

الـ gpio-reg driver نفسه مش بيقرأ الـ DT مباشرةً — الـ platform driver اللي بيستخدمه هو اللي بيعمل ده. لكن لازم تتحقق من:

```bash
# اعرف الـ DT node المرتبط بالـ GPIO chip
cat /sys/class/gpio/gpiochip480/of_node/name 2>/dev/null
# أو
ls -la /sys/class/gpio/gpiochip480/of_node

# اعرف الـ DT properties
cat /proc/device-tree/soc/gpio@10020000/compatible
cat /proc/device-tree/soc/gpio@10020000/reg  | xxd   # الـ physical address

# قارن الـ reg property مع اللي في الـ driver
# الـ driver بيستقبل void __iomem *reg — المفروض هو ioremap للعنوان ده
```

**تحقق من الـ GPIO consumer في الـ DT:**

```bash
# لو GPIO consumer (LED، reset، إلخ) مش شغال
cat /proc/device-tree/leds/led-red/gpios | xxd
# output: phandle (4 bytes) + gpio_number (4 bytes) + flags (4 bytes)
```

**مثال DT صح:**

```dts
/* الـ GPIO chip */
gpio_reg: gpio-controller@10020000 {
    compatible = "vendor,my-gpio";
    reg = <0x10020000 0x4>;      /* register واحدة — 4 bytes */
    gpio-controller;
    #gpio-cells = <2>;
    gpio-line-names = "led-red", "led-green", "reset", ...;
};

/* الـ consumer */
leds {
    led-red {
        gpios = <&gpio_reg 0 GPIO_ACTIVE_HIGH>;
    };
};
```

**لو الـ DT compile فيه errors:**

```bash
dtc -I dtb -O dts /path/to/dtb | grep -A 10 "gpio-controller"
# أو
fdtdump /path/to/dtb | grep -A 10 "10020000"
```

---

### Practical Commands

#### شيت الـ Commands الجاهزة

```bash
# ===== تشخيص أولي =====

# 1. اعرف كل GPIO chips في النظام
cat /sys/kernel/debug/gpio

# 2. اعرف chip معين بالـ label
grep -A 40 "my-gpio-label" /sys/kernel/debug/gpio

# 3. اعرف الـ base number للـ chip
cat /sys/class/gpio/gpiochip480/base
cat /sys/class/gpio/gpiochip480/ngpio
cat /sys/class/gpio/gpiochip480/label

# ===== قراءة وكتابة من userspace =====

# 4. Export GPIO
echo 480 > /sys/class/gpio/export

# 5. قرأ قيمة
cat /sys/class/gpio/gpio480/value

# 6. اكتب قيمة (لو output)
echo 1 > /sys/class/gpio/gpio480/value

# 7. باستخدام libgpiod (أحدث وأفضل)
gpiodetect
gpioinfo gpiochip0
gpioget --active-low gpiochip0 2
gpioset gpiochip0 2=1 3=0

# ===== تحقق من الـ Hardware Register =====

# 8. اقرأ الـ register مباشرةً (غيّر العنوان)
devmem2 0x10020000 w

# 9. double-read للـ input latching
devmem2 0x10020000 w; devmem2 0x10020000 w

# ===== Ftrace =====

# 10. فعّل GPIO tracing
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
sleep 2
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace | grep gpio

# ===== Dynamic Debug =====

# 11. فعّل debug لكل gpio subsystem
echo 'file drivers/gpio/* +p' > /sys/kernel/debug/dynamic_debug/control
# راقب النتيجة
dmesg -w | grep gpio

# ===== Resume/Suspend debugging =====

# 12. اختبر الـ resume يدوياً
echo mem > /sys/power/state   # suspend
# بعد الـ wakeup
cat /sys/kernel/debug/gpio    # تحقق إن القيم اتستعادت

# ===== IRQ debugging =====

# 13. تحقق من الـ IRQ mapping
cat /proc/interrupts | grep gpio

# 14. تحقق من irq domain
ls /sys/kernel/debug/irq/domains/
cat /sys/kernel/debug/irq/domains/GPIO/

# ===== الـ Kernel Log =====

# 15. راقب الـ GPIO messages
dmesg | grep -i gpio
dmesg | grep -i "gpio-reg"
journalctl -k | grep -i gpio
```

#### تفسير مثال output كامل

```bash
$ cat /sys/kernel/debug/gpio
gpiochip0: GPIOs 480-511, parent: platform/10020000.gpio, soc-control-gpios:
 gpio-480 (                    |led-power       ) out hi
 gpio-481 (                    |wifi-reset      ) out lo
 gpio-482 (                    |button          ) in  lo IRQ
 gpio-483 (                    |card-detect     ) in  hi IRQ
```

**التفسير:**
- **`out hi` على gpio-480:** الـ shadow register `r->out` عنده bit 0 = 1 — الـ LED شغال
- **`out lo` على gpio-481:** الـ wifi-reset في الـ low state — الـ wifi in reset
- **`in  lo` على gpio-482:** الـ button مش مضغوط (active low → قيمة 0 من الـ hardware register)
- **`IRQ`:** gpio-482 وgpio-483 عندهم irq mapping — الـ `irqs` array موجود وصح

لو شفت GPIO مكتوب `out` بس مش مكتوب اسم (اللي بين `|...|`) — يبقى الـ `names` array اتبعتله `NULL` أو الـ index ده `NULL`.

لو الـ GPIO مش ظاهر خالص — يبقى الـ `gpiochip_add_data` فشل، دور في `dmesg` على error.
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Industrial Gateway على AM62x — الـ GPIO مش بيتحدث بعد الـ suspend

#### العنوان
**GPIO output عايد غلط بعد الـ resume على industrial gateway**

#### السياق
شركة بتبني industrial gateway على SoC **TI AM62x**. الـ gateway بيتحكم في relay outputs عن طريق register واحد memory-mapped بيتحكم في 8 GPIO lines — كل line بتفتح أو تقفل valve في خط إنتاج. الـ driver المستخدم هو `gpio-reg` لأن الـ GPIOs اتجاهها ثابت (كلها output) ومربوطة بـ register واحد.

#### المشكلة
بعد ما الـ system يدخل في `suspend-to-RAM` ويرجع، الـ relay outputs بترجع كلها `0` (مغلقة) حتى لو كانت بعضها مفتوحة قبل الـ suspend. ده بيسبب إيقاف مفاجئ في العمليات الصناعية.

#### التحليل
**الـ `gpio_reg_resume`** هو اللي المفروض يحل المشكلة دي:

```c
int gpio_reg_resume(struct gpio_chip *gc)
{
    struct gpio_reg *r = to_gpio_reg(gc);
    unsigned long flags;

    spin_lock_irqsave(&r->lock, flags);
    writel_relaxed(r->out, r->reg);  /* يكتب القيمة المحفوظة في r->out */
    spin_unlock_irqrestore(&r->lock, flags);

    return 0;
}
```

الـ `r->out` محفوظ في الـ RAM — لو الـ RAM اتحفظ صح في الـ suspend، المفروض الـ resume يكتبه تاني. المشكلة إن الـ driver code مش بيستدعي `gpio_reg_resume` في الـ `.resume` callback تبعه. الـ engineer نسي يربط الـ function بالـ power management hooks في الـ platform driver اللي بيستخدم `gpio_reg_init`.

**التتبع:**
```
suspend → hardware register اتصفر
resume  → gpio_reg_resume() مش اتستدعى
        → r->out = 0x5A (القيمة الصح في الـ RAM)
        → لكن الـ register لسه بيقرأ 0x00
```

#### الحل
في الـ platform driver اللي بيستخدم `gpio_reg_init`، لازم يضيف الـ resume callback:

```c
static int my_gpio_reg_resume(struct device *dev)
{
    struct my_priv *priv = dev_get_drvdata(dev);

    /* استدعاء gpio_reg_resume عشان يكتب r->out تاني في الـ hardware */
    return gpio_reg_resume(priv->gc);
}

static const struct dev_pm_ops my_gpio_pm_ops = {
    .resume = my_gpio_reg_resume,
    .restore = my_gpio_reg_resume,
};

static struct platform_driver my_driver = {
    .driver = {
        .name = "my-gpio-reg",
        .pm   = &my_gpio_pm_ops,
    },
    ...
};
```

#### الدرس المستفاد
**الـ `gpio_reg_resume` مش بيتستدعى أوتوماتيك** — المكتبة بتوفره بس لازم الـ platform driver يربطه يدويًا بالـ PM framework. أي driver بيستخدم `gpio-reg` في system بيدعم الـ suspend لازم يعمل ده صراحةً.

---

### السيناريو الثاني: Android TV Box على Allwinner H616 — الـ HDMI لا بيشتغل

#### العنوان
**HDMI enable GPIO مش اشتغل بسبب `direction` bitmask غلط**

#### السياق
فريق بيطور Android TV box على **Allwinner H616**. الـ HDMI signal path بيحتاج GPIO لتشغيل الـ HDMI power switch (output) وGPIO تاني لقراءة الـ HPD — Hot Plug Detect (input). الاتنين موجودين في register واحد في custom PMIC متصل بالـ SoC. الـ driver بيستخدم `gpio_reg_init`.

#### المشكلة
الـ HDMI مش بيشتغل خالص. الـ `gpio_set_value` للـ enable pin مش بيعمل حاجة.

#### التحليل
الـ `gpio_reg_direction_output` بيرفض الـ request بسبب الـ bitmask:

```c
static int gpio_reg_direction_output(struct gpio_chip *gc, unsigned offset,
    int value)
{
    struct gpio_reg *r = to_gpio_reg(gc);

    if (r->direction & BIT(offset))
        return -ENOTSUPP;   /* الـ bit ده إنت قلت إنه input! */

    gc->set(gc, offset, value);
    return 0;
}
```

الـ engineer كتب الـ `direction` bitmask معكوس. في الـ API، **`1` = input، `0` = output**:

```c
/* الغلط: bit 0 = HPD (input), bit 1 = HDMI_EN (output) */
/* المفروض: direction = 0b01 = 0x01  (bit 0 هو الـ input) */
/* اللي اتكتب:  direction = 0b10 = 0x02  (bit 1 اتعمل input بالغلط) */

gc = gpio_reg_init(dev, reg, base, 2, "hdmi-gpio",
    0x02,   /* غلط: بيقول إن bit 1 = HDMI_EN هو input */
    0x00, names, NULL, NULL);
```

لما الـ consumer بيطلب `direction_output` على offset 1 (HDMI_EN):
```c
r->direction & BIT(1)  →  0x02 & 0x02 = 0x02 (non-zero) → return -ENOTSUPP
```

#### الحل
تصحيح الـ bitmask:

```c
/*
 * bit 0 = HDMI_EN  → output (0)
 * bit 1 = HPD      → input  (1)
 * direction = 0b10 = 0x02
 */
gc = gpio_reg_init(dev, reg, base, 2, "hdmi-gpio",
    0x02,   /* صح: bit 1 هو الـ HPD وهو input */
    0x00, names, NULL, NULL);
```

للتأكد بعد التصحيح:
```bash
# تحقق من الـ direction
cat /sys/kernel/debug/gpio | grep hdmi-gpio
```

#### الدرس المستفاد
**الـ `direction` bitmask في `gpio_reg_init` — `1` يعني input مش output** — اللي هو عكس التفكير الطبيعي. اتصدق بالـ `gpio_reg_get_direction` في الكود:

```c
return r->direction & BIT(offset) ? GPIO_LINE_DIRECTION_IN :
                                    GPIO_LINE_DIRECTION_OUT;
```

لازم دايمًا تكتب comment واضح جنب الـ bitmask في الكود.

---

### السيناريو الثالث: IoT Sensor Node على STM32MP1 — Race Condition في الـ SPI CS

#### العنوان
**تلف بيانات SPI بسبب GPIO set بدون lock في context خاطئ**

#### السياق
نظام IoT على **STM32MP1** بيقرأ بيانات من sensor عبر SPI. الـ chip select الخاص بالـ sensor مش مدعوم في الـ SPI controller، فالـ engineer استخدم GPIO من `gpio-reg` كـ manual CS. الـ sensor driver بيتشغل من interrupt context أحيانًا.

#### المشكلة
بيانات SPI بتتفسد بشكل عشوائي، وفي kernel traces بتظهر:

```
BUG: spinlock recursion on CPU#0
```

#### التحليل
الـ `gpio_reg_set` بيستخدم `spin_lock_irqsave`:

```c
static int gpio_reg_set(struct gpio_chip *gc, unsigned int offset, int value)
{
    struct gpio_reg *r = to_gpio_reg(gc);
    unsigned long flags;
    u32 val, mask = BIT(offset);

    spin_lock_irqsave(&r->lock, flags);   /* يمسك الـ lock ويقفل الـ IRQs */
    val = r->out;
    if (value)
        val |= mask;
    else
        val &= ~mask;
    r->out = val;
    writel_relaxed(val, r->reg);
    spin_unlock_irqrestore(&r->lock, flags);

    return 0;
}
```

الـ flow اللي بيحصل:

```
Thread A (process context):
  gpio_set_value(cs_gpio, 0)
    → spin_lock_irqsave(&r->lock)  ← مسك الـ lock
    → interrupt يحصل هنا!

IRQ Handler:
  sensor_irq_handler()
    → gpio_set_value(cs_gpio, 1)   ← بيحاول يمسك نفس الـ lock
    → DEADLOCK / recursion warning
```

الـ `spin_lock_irqsave` المفروض يمنع ده عن طريق disable الـ IRQs، لكن المشكلة إن الـ `spin_lock_irqsave` بيقفل الـ local CPU IRQs فقط. لو الـ IRQ على CPU تاني في SMP system، الـ deadlock بيحصل.

#### الحل
الـ SPI CS GPIO المفروض مش يتستخدم من interrupt context. لازم الـ sensor driver يعمل الـ CS toggle في process context أو يستخدم workqueue:

```c
/* بدل الاستدعاء المباشر من الـ IRQ handler */
static irqreturn_t sensor_irq_handler(int irq, void *data)
{
    struct sensor_priv *priv = data;
    /* schedule work بدل الـ GPIO toggle المباشر */
    schedule_work(&priv->read_work);
    return IRQ_HANDLED;
}

static void sensor_read_work(struct work_struct *work)
{
    /* هنا آمن نعمل GPIO toggle */
    gpiod_set_value(priv->cs_gpio, 0);
    spi_sync(priv->spi, &priv->msg);
    gpiod_set_value(priv->cs_gpio, 1);
}
```

#### الدرس المستفاد
الـ `gpio-reg` بيستخدم `spinlock` مع `irqsave` — ده كافي في single-core، لكن في **SMP** لو نفس الـ GPIO اتستخدم من process context وinterrupt context على cores مختلفة، لازم الـ design يتغير وينقل التعامل مع الـ GPIO لـ process context.

---

### السيناريو الرابع: Custom Board Bring-up على RK3562 — الـ GPIO names مش بتظهر في sysfs

#### العنوان
**`gpio_reg_init` بيتعدى ناجح لكن الـ GPIO names مش بتظهر في debugfs**

#### السياق
فريق bring-up board جديد على **RK3562** بيستخدم CPLD صغير متوصل بالـ SoC عن طريق memory-mapped interface. الـ CPLD بيتحكم في 6 إشارات: reset lines لـ peripherals مختلفة. الـ engineer بيستخدم `gpio_reg_init` عشان يعرض الـ CPLD outputs كـ GPIO chip.

#### المشكلة
الـ `gpio_reg_init` بترجع pointer ناجح، لكن لما بيعمل:

```bash
cat /sys/kernel/debug/gpio
```

بيشوف الـ chip بس من غير أسماء للـ lines:

```
gpiochip5: GPIOs 496-501, cpld-gpio:
 gpio-496 (                    )
 gpio-497 (                    )
 ...
```

المفروض يشوف:
```
 gpio-496 (rst-eth            )
 gpio-497 (rst-usb            )
```

#### التحليل
الـ `gpio_reg_init` بيمرر الـ `names` array للـ `gc.names`:

```c
r->gc.names = names;   /* pointer assignment فقط */
```

الـ `struct gpio_chip` بيتوقع الـ `names` تكون array of `const char *` — **مش بتتعمل لها copy**. الـ engineer مرر pointer لـ array على الـ stack:

```c
static int my_probe(struct platform_device *pdev)
{
    /* array على الـ stack — هتتمسح بعد ما الـ probe يخلص! */
    const char *names[] = {
        "rst-eth", "rst-usb", "rst-hdmi",
        "rst-wifi", "rst-bt", "pwr-led"
    };

    gc = gpio_reg_init(&pdev->dev, reg, -1, 6, "cpld-gpio",
        0x00, 0x00, names, NULL, NULL);
    /* هنا names اتبقى dangling pointer */
}
```

بعد ما الـ probe بينتهي، الـ stack frame بيتمسح والـ `names` pointers بتبقى dangling — الـ `gpio_chip` بيحتفظ بـ pointer على memory اتحررت.

#### الحل
لازم الـ `names` array تكون `static` أو allocated على الـ heap:

```c
/* الحل 1: static */
static const char * const cpld_gpio_names[] = {
    "rst-eth", "rst-usb", "rst-hdmi",
    "rst-wifi", "rst-bt",  "pwr-led"
};

/* الحل 2: devm_kasprintf */
static int my_probe(struct platform_device *pdev)
{
    const char **names;
    int i;

    names = devm_kcalloc(&pdev->dev, 6, sizeof(*names), GFP_KERNEL);
    names[0] = devm_kstrdup(&pdev->dev, "rst-eth", GFP_KERNEL);
    /* ... */

    gc = gpio_reg_init(&pdev->dev, reg, -1, 6, "cpld-gpio",
        0x00, 0x00, names, NULL, NULL);
}
```

التحقق:
```bash
cat /sys/kernel/debug/gpio | grep cpld
# المفروض يطلع الأسماء صح
```

#### الدرس المستفاد
الـ `gpio_reg_init` بيعمل **shallow copy** للـ `names` pointer — مش بتعمل copy للـ strings نفسها. الـ lifetime بتاع الـ `names` array والـ strings اللي بتشاور عليها لازم يكون على الأقل زي الـ `gpio_chip` نفسه. استخدم `devm_` allocations أو `static` arrays دايمًا.

---

### السيناريو الخامس: Automotive ECU على i.MX8 — IRQ mapping غلط بيسبب kernel panic

#### العنوان
**Kernel panic عند أول GPIO interrupt على automotive ECU**

#### السياق
ECU على **i.MX8** بيستخدم custom safety ASIC متوصل عبر memory-mapped register. الـ ASIC بيوفر 4 GPIO lines — 3 منها output (relay control) وواحدة input (fault detection). الـ fault input لازم يطلع interrupt لما يتغير حالته. الـ engineer بيستخدم `gpio_reg_init` مع `irq_domain` وarray من الـ IRQ numbers.

#### المشكلة
الـ system بيشتغل كويس، لكن لما الـ fault signal يجي (الـ GPIO input يتغير)، الـ kernel يعمل panic:

```
Unable to handle kernel NULL pointer dereference at virtual address 0000000000000000
Call trace:
 irq_find_mapping+0x...
 gpio_reg_to_irq+0x...
```

#### التحليل
الـ `gpio_reg_to_irq` بيعمل mapping من GPIO offset لـ Linux IRQ number:

```c
static int gpio_reg_to_irq(struct gpio_chip *gc, unsigned offset)
{
    struct gpio_reg *r = to_gpio_reg(gc);
    int irq = r->irqs[offset];

    if (irq >= 0 && r->irqdomain)
        irq = irq_find_mapping(r->irqdomain, irq);  /* هنا الـ crash */

    return irq;
}
```

الـ `irq_find_mapping` بترجع `0` لو ما لقتش الـ mapping — وده valid value يعتبر "no IRQ" في بعض الـ contexts. المشكلة إن الـ engineer مرر `irqdom` لكن الـ hardware IRQ number في الـ `irqs` array غلط:

```c
/* الغلط: irqs[3] بيشاور على hardware IRQ رقم 0 */
/* في الـ irqdomain ده، 0 مش valid mapping */
static const int asic_irqs[] = { -1, -1, -1, 0 };
/*                                            ^ غلط — المفروض يكون الـ HW IRQ الفعلي */
```

الـ `irq_find_mapping` مع HW IRQ = 0 على domain مش فيه mapping بترجع 0، وبعدين الـ GPIO subsystem بيحاول يعمل `request_irq(0, ...)` اللي بيعمل crash.

**التتبع الصح:**
```c
if (irq >= 0 && r->irqdomain)
    /* irq = 0 (HW number) → irq_find_mapping → return 0 (no mapping found) */
    irq = irq_find_mapping(r->irqdomain, irq);
/* irq = 0 → الـ caller بيفكر ده valid IRQ number */
```

الـ fix الصح إن الـ irqs array لازم يحتوي على الـ hardware IRQ number الصح من الـ ASIC datasheet، مثلًا `32` أو `64`:

```c
/* قرر من الـ ASIC datasheet إن الـ fault signal على HW IRQ 45 */
static const int asic_irqs[] = { -1, -1, -1, 45 };
```

#### الحل
1. راجع الـ ASIC datasheet للـ interrupt number الصح.
2. تحقق إن الـ irqdomain اتعمل correctly وفيه mapping للـ HW IRQ ده:

```bash
# شوف الـ irq domains
cat /proc/interrupts
cat /sys/kernel/debug/irq/domains/*/name

# تحقق من الـ mapping
cat /sys/kernel/debug/irq/domains/*/mappings
```

3. لو الـ GPIO ده input ومحتاج interrupt، تأكد إن الـ `-1` في الـ `irqs` array محجوز للـ GPIOs اللي مش ليها interrupts فقط:

```c
/*
 * -1  = no IRQ for this GPIO
 * 45  = HW IRQ number 45 in the given irqdomain
 */
static const int asic_irqs[] = { -1, -1, -1, 45 };

gc = gpio_reg_init(dev, reg, -1, 4, "asic-gpio",
    BIT(3),          /* bit 3 = input (fault) */
    0x00,
    asic_gpio_names,
    asic_irqdomain,  /* irqdomain للـ ASIC */
    asic_irqs);
```

#### الدرس المستفاد
**الـ `irqs` array في `gpio_reg_init` بيحتوي على hardware IRQ numbers مش Linux IRQ numbers** — لما `irqdomain` يكون non-NULL. الـ `gpio_reg_to_irq` بيعمل translation باستخدام `irq_find_mapping`. لازم تتأكد إن الـ HW IRQ number موجود فعلًا كـ mapping في الـ domain قبل ما تمرره، وإلا الـ `irq_find_mapping` هترجع `0` وده ممكن يسبب panic.
## Phase 7: مصادر ومراجع

### مقدمة

**الـ** `gpio-reg` driver هو driver بسيط جداً — بيوفر interface موحدة لأي hardware بيستخدم **memory-mapped register واحد** عشان يتحكم في مجموعة GPIO pins. الـ header الخاص بيه (`include/linux/gpio/gpio-reg.h`) بيعرّف function اتنين بس: `gpio_reg_init()` و `gpio_reg_resume()`. المصادر اللي جاية هتساعدك تفهم الـ GPIO subsystem اللي بيعمل عليه الـ driver ده بشكل أعمق.

---

### مقالات LWN.net المهمة

| المقال | الأهمية |
|--------|---------|
| [GPIO in the kernel: an introduction](https://lwn.net/Articles/532714/) | أهم مقال — بيشرح الـ `gpio_chip` structure والـ API الأساسي |
| [gpio: gpio-regmap: Support few custom operations](https://lwn.net/Articles/857361/) | بيشرح الفرق بين الـ `gpio-reg` البسيط والـ `gpio-regmap` الأحدث |
| [gpio: Document GPIO hogging mechanism](https://lwn.net/Articles/641117/) | مهم لفهم إزاي الـ GPIO lines بتتحجز في الـ boot |
| [gpio: Support for shared GPIO lines on boards](https://lwn.net/Articles/803629/) | بيناقش الـ shared GPIO lines — مرتبط بالـ `def_out` parameter |
| [Documentation: gpiolib: document new interface](https://lwn.net/Articles/574055/) | بيوثق الـ descriptor-based API اللي بيتبنيه الـ `gpio_chip` |

---

### التوثيق الرسمي للـ Kernel

الـ GPIO subsystem عنده documentation ممتاز جوه الـ kernel نفسه:

```
Documentation/driver-api/gpio/
├── index.rst          ← نقطة البداية
├── intro.rst          ← مقدمة عن الـ GPIO framework
├── driver.rst         ← دليل كتابة GPIO chip drivers (الأهم)
├── board.rst          ← إزاي بتربط GPIO بالـ device tree
├── consumer.rst       ← إزاي الـ drivers التانية بتستخدم GPIO
└── drivers-on-gpio.rst← subsystem drivers بتبني فوق GPIO
```

الروابط المباشرة لتوثيق الـ kernel الرسمي:

- [General Purpose Input/Output (GPIO)](https://docs.kernel.org/driver-api/gpio/index.html) — الفهرس الرئيسي
- [GPIO Driver Interface](https://docs.kernel.org/driver-api/gpio/driver.html) — **الأهم** لكاتب driver زي `gpio-reg`
- [Introduction to GPIO](https://docs.kernel.org/driver-api/gpio/intro.html) — نظرة عامة على الـ subsystem
- [GPIO Mappings](https://docs.kernel.org/driver-api/gpio/board.html) — ربط الـ GPIO بالـ device tree و platform data

---

### ملفات الـ Kernel المرتبطة مباشرة

الملفات دي في الـ kernel source هي المرجع الأساسي لفهم `gpio-reg`:

```
include/linux/gpio/gpio-reg.h   ← الـ header — اتنين functions بس
drivers/gpio/gpio-reg.c         ← الـ implementation الكاملة
drivers/gpio/gpiolib.c          ← قلب الـ GPIO subsystem
include/linux/gpio/driver.h     ← تعريف struct gpio_chip
```

**الـ `gpio-reg.c`** نفسه بيوضح إزاي بيستخدم الـ `bgpio_init()` من الـ `gpiolib` وبيضيف فوقيه الـ IRQ support وإدارة الـ register.

---

### الـ Commits المهمة في تاريخ الـ Driver

ابحث عن الـ commits دي في [GitHub - torvalds/linux](https://github.com/torvalds/linux/blob/master/include/linux/gpio/gpio-reg.h):

```bash
# شوف تاريخ الملف من البداية
git log --follow --oneline -- drivers/gpio/gpio-reg.c
git log --follow --oneline -- include/linux/gpio/gpio-reg.h

# شوف أول commit أدخل الـ driver
git log --diff-filter=A -- drivers/gpio/gpio-reg.c
```

- **الـ driver اتكتب في الأساس عشان الـ ARM platforms** اللي بتستخدم register واحد 32-bit للـ GPIO
- الـ implementation الكاملة موجودة في: [drivers/gpio/gpio-reg.c على GitHub](https://github.com/torvalds/linux/blob/master/drivers/gpio/gpio-reg.c) (via Xilinx fork للمقارنة: [linux-xlnx gpio-reg.c](https://github.com/Xilinx/linux-xlnx/blob/master/drivers/gpio/gpio-reg.c))
- الـ header الرسمي: [include/linux/gpio/gpio-reg.h على GitHub](https://github.com/torvalds/linux/blob/master/include/linux/gpio/gpio-reg.h)

---

### نقاشات الـ Mailing List

- [linux-gpio mailing list archive على lore.kernel.org](https://lore.kernel.org/linux-gpio/) — ابحث فيه عن `gpio-reg` أو `gpio_reg_init`
- [GPIO Driver Interface Discussion — Patchwork](https://patchwork.kernel.org/project/linux-arm-kernel/list/?q=gpio-reg) — متابعة الـ patches المرتبطة بالـ ARM GPIO

بالنسبة للنقاشات التاريخية، الـ `gpio-reg` driver اتناقش في سياق الـ ARM platform GPIO needs، خصوصاً للـ boards اللي عندها GPIO controller بسيط في 32-bit register.

---

### الكتب المرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 1**: Overview of the Linux Kernel — سياق عام
- **الفصل 9**: Communicating with Hardware — بيشرح الـ `ioremap()` و `ioread32()` اللي `gpio-reg` بيعتمد عليهم
- **الفصل 14**: The Linux Device Model — بيشرح الـ `struct device` اللي بتتمرر لـ `gpio_reg_init()`

متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 7**: Interrupts and Interrupt Handlers — مهم لفهم الـ `irq_domain` parameter
- **الفصل 17**: Devices and Modules — الـ device model اللي بيشتغل فيه الـ `gpio_chip`
- **الفصل 19**: Portability — بيشرح فلسفة الـ platform-specific drivers زي `gpio-reg`

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 15**: Debugging Embedded Linux — بيتكلم عن debugging الـ GPIO drivers
- **الفصل 16**: Kernel Debugging Techniques — ممكن تطبقه على `gpio-reg`

#### Mastering Embedded Linux Programming — Frank Vasquez & Chris Simmonds
- بيتكلم عن الـ GPIO character device interface والـ `gpio_chip` بشكل تطبيقي

---

### مصادر kernelnewbies.org

- [Linux 6.2 Changes — kernelnewbies.org](https://kernelnewbies.org/Linux_6.2) — بيذكر تحديثات الـ GPIO subsystem
- [Linux 6.13 Changes — kernelnewbies.org](https://kernelnewbies.org/Linux_6.13) — آخر تحديثات GPIO
- [Linux 6.11 Changes — kernelnewbies.org](https://kernelnewbies.org/Linux_6.11) — GPIO PWM driver وتحسينات
- [LinuxChanges — kernelnewbies.org](https://kernelnewbies.org/LinuxChanges) — timeline كامل للتغييرات

---

### مصادر elinux.org

- [EBC GPIO via mmap](https://elinux.org/EBC_GPIO_via_mmap) — مثال عملي لـ memory-mapped GPIO زي ما بيعمل `gpio-reg`
- [Rpi GPIO Registers — elinux.org](https://elinux.org/Rpi_Datasheet_751_GPIO_Registers) — مثال حقيقي لـ register layout
- [EBC GPIO Polling and Interrupts](https://elinux.org/EBC_gpio_Polling_and_Interrupts) — بيشرح الـ IRQ integration اللي بيعمله `gpio_reg_init()`
- [New GPIO Interface for User Space (PDF)](https://elinux.org/images/7/74/Elce2017_new_GPIO_interface.pdf) — شرح الـ character device API الحديث
- [Introduction to pin muxing and GPIO control under Linux (PDF)](https://elinux.org/images/a/a7/ELC-2021_Introduction_to_pin_muxing_and_GPIO_control_under_Linux.pdf) — مقدمة ممتازة للـ GPIO ecosystem

---

### مصادر إضافية مفيدة

- [GPIO Driver documentation — docs.kernel.org](https://docs.kernel.org/driver-api/gpio/driver.html) — دليل كتابة الـ `gpio_chip` drivers
- [Linux kernel GPIO user space interface — sergioprado.blog](https://sergioprado.blog/new-linux-kernel-gpio-user-space-interface/) — بيشرح الـ character device API
- [GPIO Descriptor Driver Interface (v4.20)](https://docs.kernel.org/4.20/driver-api/gpio/driver.html) — نسخة أقدم مفيدة للمقارنة
- [Old GPIO documentation (gpio.txt)](https://android.googlesource.com/kernel/common/+/bcmdhd-3.10/Documentation/gpio.txt) — التوثيق القديم قبل الـ descriptor API

---

### كلمات البحث المفيدة

استخدم الكلمات دي للبحث عن معلومات أكتر:

```
# بحث في Google / DuckDuckGo
linux kernel "gpio-reg" driver site:lore.kernel.org
linux "gpio_reg_init" "gpio_chip" implementation
linux kernel "bgpio_init" "gpio-reg" relationship
linux GPIO "memory mapped" "single register" driver

# بحث في Git
git log --all --grep="gpio-reg" -- drivers/gpio/
git log --all --grep="gpio_reg_init"

# بحث في Kernel Mailing List
lore.kernel.org/linux-gpio/?q=gpio-reg
```

---

### خلاصة أولويات القراءة

| الأولوية | المصدر | السبب |
|---------|--------|-------|
| 1 | [GPIO Driver Interface — kernel docs](https://docs.kernel.org/driver-api/gpio/driver.html) | فهم `gpio_chip` structure ضروري |
| 2 | [GPIO in the kernel: an introduction — LWN](https://lwn.net/Articles/532714/) | سياق تاريخي وتقني مهم |
| 3 | [gpio-reg.c source — GitHub](https://github.com/torvalds/linux/blob/master/drivers/gpio/gpio-reg.c) | الـ implementation الفعلية |
| 4 | [gpio-regmap custom ops — LWN](https://lwn.net/Articles/857361/) | فهم البديل الأحدث |
| 5 | [EBC GPIO via mmap — elinux.org](https://elinux.org/EBC_GPIO_via_mmap) | مثال تطبيقي على نفس الفكرة |
## Phase 8: Writing simple module

### الهدف

**`gpio_reg_resume`** هي الدالة الأنسب للـ kprobe هنا — بيتم استدعاؤها لما يرجع الـ system من الـ suspend وبتعيد تهيئة الـ memory-mapped GPIO chip. ده بيخلينا نشوف أي GPIO chip بترجع من الـ suspend ونطبع اسمها.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on gpio_reg_resume — prints the label of any memory-mapped
 * GPIO chip that resumes from system suspend.
 */
#include <linux/kernel.h>      /* pr_info, pr_err */
#include <linux/module.h>      /* MODULE_* macros, module_init/exit */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe */
#include <linux/gpio/driver.h> /* struct gpio_chip — to read .label */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Doc Project");
MODULE_DESCRIPTION("kprobe on gpio_reg_resume to trace GPIO chip resume events");

/*
 * pre_handler — called just BEFORE gpio_reg_resume executes.
 *
 * Signature of the real function:
 *   int gpio_reg_resume(struct gpio_chip *gc);
 *
 * On x86-64 the first argument lives in %rdi.
 * On ARM64 it lives in x0.
 * regs_get_kernel_argument(regs, 0) is portable across both.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Retrieve the first argument: pointer to gpio_chip */
    struct gpio_chip *gc =
        (struct gpio_chip *)regs_get_kernel_argument(regs, 0);

    if (!gc) {
        pr_info("gpio_reg_resume: called with NULL gpio_chip!\n");
        return 0;
    }

    /*
     * gc->label holds the human-readable name passed to gpio_reg_init(),
     * e.g. "gpio-reg" or whatever the board driver set.
     * gc->ngpio is the number of lines this chip exposes.
     * gc->base is the first GPIO number in the global numberspace.
     */
    pr_info("gpio_reg_resume: chip='%s' base=%d ngpio=%u resuming\n",
            gc->label  ? gc->label  : "(no label)",
            gc->base,
            gc->ngpio);

    return 0; /* 0 = let the real function run normally */
}

/* The kprobe descriptor — we only need pre_handler for read-only tracing */
static struct kprobe kp = {
    .symbol_name = "gpio_reg_resume", /* function to probe by name */
    .pre_handler = handler_pre,
};

static int __init gpio_reg_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("gpio_reg_resume probe: register_kprobe failed (%d)\n", ret);
        return ret;
    }

    pr_info("gpio_reg_resume probe: planted at %p\n", kp.addr);
    return 0;
}

static void __exit gpio_reg_probe_exit(void)
{
    /*
     * unregister_kprobe removes the breakpoint and waits for any
     * in-flight handler to finish before returning — safe to unload.
     */
    unregister_kprobe(&kp);
    pr_info("gpio_reg_resume probe: removed\n");
}

module_init(gpio_reg_probe_init);
module_exit(gpio_reg_probe_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `<linux/kernel.h>` | عشان نستخدم `pr_info` و `pr_err` |
| `<linux/module.h>` | الـ macros الأساسية لأي kernel module |
| `<linux/kprobes.h>` | تعريف `struct kprobe` ودوال التسجيل |
| `<linux/gpio/driver.h>` | عشان نـ cast الـ argument لـ `struct gpio_chip` ونقرأ فيلد `label` |

---

#### الـ `handler_pre` — الـ Callback

**الـ `regs_get_kernel_argument(regs, 0)`** بتجيب أول argument بطريقة portable بدون ما نفكر في اسم الـ register (rdi على x86-64 أو x0 على ARM64). بعدين بنـ cast القيمة لـ `struct gpio_chip *` عشان نقدر نقرأ الـ metadata بتاعة الـ chip.

الـ `pr_info` بتطبع ثلاث معلومات مهمة: اسم الـ chip، ورقم الـ base في الـ global GPIO numberspace، وعدد الـ GPIO lines — ده بيخلينا نعرف بالظبط أي chip عملت resume.

---

#### الـ `struct kprobe`

بنحدد `symbol_name` بالاسم النصي عشان الـ kernel يحسب العنوان وقت الـ runtime من الـ kallsyms، ومحتاجين بس `pre_handler` لأننا بس عايزين نقرأ الـ arguments قبل ما الدالة تشتغل.

---

#### الـ `module_init` / `module_exit`

**`register_kprobe`** بتحط breakpoint على الدالة المطلوبة وتربط الـ handler بيها. لو فشلت (مثلاً الدالة مش موجودة أو الـ CONFIG_KPROBES مش enabled) بترجع error code سالب. **`unregister_kprobe`** في الـ exit ضرورية عشان تشيل الـ breakpoint وتضمن إن مفيش handler شغال وقت الـ unload — من غير ده ممكن يحصل kernel panic.

---

### Makefile للبناء

```makefile
obj-m += gpio_reg_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

---

### تشغيل واختبار

```bash
# بناء الـ module
make

# تحميله
sudo insmod gpio_reg_probe.ko

# trigger الـ resume (على جهاز حقيقي)
sudo rtcwake -m mem -s 5

# مشاهدة الـ output
dmesg | grep gpio_reg_resume

# إزالة الـ module
sudo rmmod gpio_reg_probe
```

مثال على الـ output المتوقع في `dmesg`:

```
[  123.456789] gpio_reg_resume probe: planted at ffffffffc0123456
[  128.001234] gpio_reg_resume: chip='gpio-reg' base=448 ngpio=8 resuming
[  130.000001] gpio_reg_resume probe: removed
```
