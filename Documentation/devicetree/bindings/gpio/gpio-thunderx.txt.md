## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي له الملف

الملف ده جزء من **GPIO subsystem** في Linux kernel، وبالتحديد هو **Device Tree binding document** للـ GPIO controller الخاص بـ **Cavium ThunderX** — معالج ARM64 server-grade بيستخدم في Data Centers.

---

### إيه هو الـ GPIO أصلاً؟

تخيل إن جهاز الكمبيوتر بتاعك فيه أرجل كهربائية صغيرة ممكن تتحكم فيها بالبرمجة — تقرا منها أو تكتب عليها. ده هو الـ **GPIO (General Purpose Input/Output)**. كل رجل (pin) ممكن:
- تكون **input**: تقرا حالتها (0 أو 1) — زي زرار ضغطت عليه ولا لأ
- تكون **output**: تتحكم فيها (شغّل LED، حرّك موتور، إلخ)

في الـ servers والـ embedded systems، الـ GPIO بيُستخدم عشان:
- يتحكم في reset lines للـ hardware
- يقرا حالة power buttons أو fault indicators
- يوصل الـ CPU بمكونات خارجية بسيطة

---

### الـ ThunderX — مين ده؟

**Cavium ThunderX** هو معالج ARM64 مصمم للـ data centers (servers). بيتميز بإنه بيشيل كتير من الـ hardware functions جوّاه، من بينها الـ GPIO controller. الـ GPIO controller ده مش متوصل على الـ bus التقليدي (زي I2C أو SPI) — هو بيستخدم **PCI interface**، وده مهم جداً نفهمه.

---

### الـ Device Tree Binding — إيه دوره؟

خليني أحكيلك قصة:

لما Linux kernel بيشتغل على جهاز جديد، هو محتاج يعرف: "إيه الـ hardware الموجود؟ وفين؟ وبيشتغل إزاي؟" في الـ x86 عندنا BIOS/ACPI بيعمل ده. لكن في الـ ARM servers والـ embedded systems، بنستخدم **Device Tree** — ملف نصي بيوصف الـ hardware زي خريطة.

الـ **binding document** ده (الملف اللي بندرسه) هو "العقد" أو "القاموس" اللي بيقول:
> "لو لقيت node في Device Tree بيوصف ThunderX GPIO، لازم تبقى على الشكل ده... وده اللي كل property بتاعتها معناها."

يعني ده مش كود C — ده **توثيق للـ protocol** اللي الـ firmware بيتفاهم بيه مع الـ kernel.

---

### ليه الـ compatible property اختيارية؟

ده السؤال الذكي. عادةً في Device Tree، الـ `compatible` string هي أهم property — الـ kernel بيبحث عنها عشان يعرف يستدعي أنهي driver. لكن هنا المثير للاهتمام:

الـ ThunderX GPIO controller هو **PCI device**. يعني الـ kernel بيلاقيه تلقائياً عن طريق **PCI bus enumeration** (زي ما الـ Windows بيلاقي device جديد لما تحشره في الـ USB). الـ PCI vendor ID وdevice ID بيعملوا نفس الدور — فالـ `compatible` بتبقى optional وغير مستخدمة فعلاً في الـ binding.

---

### الـ Interrupt Controller — التعقيد الحقيقي

الـ ThunderX GPIO مش بس بيقرأ ويكتب pins. كل pin ممكن يولّد **interrupt** — يعني لو الـ pin اتغيرت قيمتها، الـ CPU يتنبّه فوراً من غير ما يظل يسأل "اتغيرت؟ اتغيرت؟" (ده اسمه polling وهو مكلف).

عشان يعمل ده، الـ controller بيستخدم **MSI-X (Message Signaled Interrupts Extended)** — نظام interrupts متقدم متاح على الـ PCI bus. كل GPIO line عندها MSI-X entry خاص بيها.

---

### الصورة الكاملة بشكل مبسط

```
[ Server Board ]
      |
[ PCI Bus ]
      |
[ ThunderX GPIO Controller ]   <-- الملف ده بيوصفه للـ kernel
      |
  _____|_____
 |    |    |
Pin0 Pin1 Pin2 ...   <-- كل pin ممكن input أو output أو interrupt source
```

الـ Device Tree node بيقول للـ kernel:
1. "فيه GPIO controller هنا على الـ PCI bus عند العنوان ده"
2. "هو بيتحكم في عدد معين من الـ pins"
3. "وهو كمان interrupt controller — كل pin ممكن يبعت interrupt"

---

### الملفات المرتبطة المهمة

| الملف | الدور |
|-------|-------|
| `drivers/gpio/gpio-thunderx.c` | الـ driver نفسه — الكود اللي بينفّذ الـ binding |
| `Documentation/devicetree/bindings/gpio/gpio.txt` | القواعد العامة لكل GPIO bindings |
| `Documentation/devicetree/bindings/interrupt-controller/interrupts.txt` | توصيف الـ interrupt flags المستخدمة |
| `include/linux/gpio/driver.h` | الـ `gpio_chip` struct اللي كل GPIO driver بيبني عليه |
| `include/dt-bindings/gpio/gpio.h` | الـ macros زي `GPIO_ACTIVE_HIGH` المستخدمة في الـ DTS files |

---

### ملفات الـ Subsystem الأساسية

**Core GPIO framework:**
- `drivers/gpio/gpiolib.c` — قلب الـ GPIO subsystem
- `drivers/gpio/gpiolib-of.c` — ربط الـ GPIO بالـ Device Tree
- `include/linux/gpio/driver.h` — واجهة الـ driver

**ThunderX-specific:**
- `drivers/gpio/gpio-thunderx.c` — الـ hardware driver
- `Documentation/devicetree/bindings/gpio/gpio-thunderx.txt` — الملف اللي بندرسه (الـ binding)

**Interrupt integration:**
- `include/linux/irq.h` — الـ IRQ framework
- `include/linux/pci.h` — الـ PCI bus interface (لأن الـ controller PCI-based)
## Phase 2: شرح الـ GPIO Framework

### المشكلة اللي الـ GPIO Subsystem بيحلها

في أي SoC أو embedded board، عندك مئات الـ pins. بعضها GPIO بسيط، بعضها multiplexed مع UART أو SPI. الـ hardware مختلف جداً من chip للتاني — STM32 بيعمل الحاجة بطريقة، Cavium ThunderX بيعملها بطريقة تانية خالص (PCI device + MSI-X interrupts). من غير framework موحد، كل driver consumer (LED, button, regulator) لازم يعرف تفاصيل كل chip — ده كارثة.

**الـ GPIOlib** هو الإجابة. هو الطبقة اللي بتعزل الـ consumers عن الـ hardware. بيقدم API موحد، وبيخلي الـ driver يشتغل على أي GPIO controller من غير ما يعرف أي حاجة عن الـ hardware تحته.

---

### الـ Solution — نهج الـ kernel

الـ kernel بيعمل abstraction بثلاث طبقات:

1. **GPIO Provider (Driver):** بيسجّل `struct gpio_chip` فيها function pointers زي `get`, `set`, `direction_input`. ده بيعمله الـ chip driver (زي `gpio-thunderx.c`).
2. **GPIOlib Core:** بتستقبل الـ `gpio_chip`، بتديها رقم، بتعمل `gpio_desc` لكل line، وبتوفر API موحد للـ consumers.
3. **GPIO Consumer:** بيطلب GPIO بالاسم من الـ device tree أو ACPI، من غير ما يعرف أي controller بيتكلم معاه.

---

### الـ Real-World Analogy — مقلب عميق

تخيل **مبنى بنك فيه كاشيرين** (GPIO lines). كل كاشير عنده درج (input/output register). أنت كـ customer (consumer driver) مش محتاج تعرف اسم الكاشير أو في أنهي فرع — بس بتقول للـ receptionist (GPIOlib): "عايز الكاشير اللي اسمه `reset-gpio`"، وهي بتوديك.

| الـ Analogy | الـ Kernel Concept |
|---|---|
| Customer | Consumer driver (LED, button, etc.) |
| Receptionist | GPIOlib core (`gpiod_get()`) |
| اسم الكاشير | GPIO name في الـ device tree |
| الكاشير نفسه | `struct gpio_desc` (single line) |
| الفرع كله | `struct gpio_chip` (controller) |
| إجراءات كل فرع | Function pointers في `gpio_chip` |
| البنك المركزي | `gpio_device` (internal state) |
| رقم الفرع | `chip->base` (GPIO number space) |
| درج الكاشير | Hardware register (`GPIO_BIT_CFG`, `GPIO_RX_DAT`) |

---

### الـ Big Picture Architecture

```
                    ┌─────────────────────────────────────────┐
                    │           Consumer Drivers               │
                    │  (leds-gpio, gpio-keys, regulator-gpio)  │
                    └──────────────────┬──────────────────────┘
                                       │ gpiod_get() / gpiod_set_value()
                                       │ <linux/gpio/consumer.h>
                    ┌──────────────────▼──────────────────────┐
                    │              GPIOlib Core                │
                    │   gpio_desc[]  gpio_device  sysfs/debugfs│
                    │   OF/ACPI lookup, request/free, locking  │
                    └──────┬──────────────────────────┬────────┘
                           │ gpio_chip ops            │ irq_domain ops
             ┌─────────────▼──────────┐   ┌───────────▼───────────┐
             │  gpio-thunderx driver  │   │  gpio-pl061 / gpio-   │
             │  (PCI, MSI-X, MMIO)   │   │  mxc / gpio-omap etc. │
             └─────────────┬──────────┘   └───────────────────────┘
                           │ PCI BAR0 MMIO
             ┌─────────────▼──────────────────────────────────────┐
             │         Cavium ThunderX ASIC                        │
             │  GPIO_RX_DAT | GPIO_TX_SET/CLR | GPIO_BIT_CFG[n]   │
             │  GPIO_INTR[n] — per-pin interrupt + MSI-X vector   │
             └─────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ core abstraction هي `struct gpio_chip`. ده هو الـ "interface contract" بين الـ hardware driver والـ GPIOlib.

```c
struct gpio_chip {
    const char  *label;         /* اسم الـ controller */
    int          base;          /* أول رقم GPIO (-1 = dynamic) */
    u16          ngpio;         /* عدد الـ lines */
    bool         can_sleep;     /* هل الـ access بيستلزم sleep? */

    /* --- الـ function pointers اللي الـ driver بيملأها --- */
    int  (*request)(struct gpio_chip *gc, unsigned int offset);
    int  (*direction_input)(struct gpio_chip *gc, unsigned int offset);
    int  (*direction_output)(struct gpio_chip *gc, unsigned int offset, int value);
    int  (*get)(struct gpio_chip *gc, unsigned int offset);
    int  (*set)(struct gpio_chip *gc, unsigned int offset, int value);
    int  (*set_multiple)(struct gpio_chip *gc, unsigned long *mask, unsigned long *bits);
    int  (*set_config)(struct gpio_chip *gc, unsigned int offset, unsigned long config);
    int  (*get_direction)(struct gpio_chip *gc, unsigned int offset);
    int  (*to_irq)(struct gpio_chip *gc, unsigned int offset);

    /* --- الـ IRQ integration --- */
    struct gpio_irq_chip irq;   /* embedded irqchip support */
};
```

كل الـ GPIOs على نفس الـ controller عندها **offset** بداية من 0. الـ GPIOlib هي اللي بتحول الـ global GPIO number → offset عشان تعدي للـ driver.

---

### الـ gpio_desc — الـ Handle الفعلي

الـ `struct gpio_desc` هو الـ handle اللي الـ consumer بيشتغل بيه. مش بيشوف الـ `gpio_chip` خالص.

```
gpio_desc
  ├── gdev  ──→  gpio_device
  │                 └── chip  ──→  gpio_chip  (driver ops)
  └── flags      (requested, active_low, open_drain, ...)
```

---

### الـ GPIO IRQ Chip — `struct gpio_irq_chip`

ده embeds إمكانية الـ interrupt controller جوه الـ `gpio_chip` نفسه. في حالة ThunderX، كل GPIO line عندها **MSI-X vector** منفصل — مش interrupt مشترك على كل الـ lines.

```c
struct gpio_irq_chip {
    struct irq_chip      *chip;           /* irq_chip ops (mask/unmask/ack) */
    struct irq_domain    *domain;         /* child domain لـ GPIO lines */
    struct irq_domain    *parent_domain;  /* parent domain (MSI-X هنا) */

    int (*child_to_parent_hwirq)(...);    /* translate GPIO offset → parent hwirq */
    int (*populate_parent_alloc_arg)(...);/* fill MSI alloc info */

    irq_flow_handler_t   handler;
    unsigned int         default_type;
};
```

ده بيخلي كل GPIO line تعمل `gpiod_to_irq()` وتشتغل كـ interrupt source للـ consumer — مثلاً `gpio-keys` بيعمل `request_irq()` على الـ GPIO line مباشرة.

---

### كيف ThunderX بتعمل implement الـ GPIO Framework

#### الـ Private State

```c
struct thunderx_gpio {
    struct gpio_chip    chip;           /* embedded — اللي بتتسجل في GPIOlib */
    u8 __iomem         *register_base; /* PCI BAR0 mapped */
    struct msix_entry  *msix_entries;  /* per-line MSI-X — مش shared! */
    struct thunderx_line *line_entries; /* per-line filter bits + back-pointer */
    raw_spinlock_t      lock;
    unsigned long       invert_mask[2]; /* بitmask للـ XOR/open-drain compensation */
    unsigned long       od_mask[2];
    int                 base_msi;
};
```

`struct thunderx_gpio` بتحتوي `struct gpio_chip` كـ embedded member. الـ driver بيستخدم `gpiochip_get_data()` عشان يرجع من الـ `gpio_chip *` للـ `thunderx_gpio *`:

```c
static int thunderx_gpio_dir_in(struct gpio_chip *chip, unsigned int line)
{
    /* get private data embedded around gpio_chip */
    struct thunderx_gpio *txgpio = gpiochip_get_data(chip);
    ...
    writeq(txgpio->line_entries[line].fil_bits,
           txgpio->register_base + bit_cfg_reg(line));
    ...
}
```

#### تسجيل الـ Chip في الـ probe

```c
static int thunderx_gpio_probe(struct pci_dev *pdev, ...)
{
    /* 1. Enable PCI device & map BAR0 */
    pcim_enable_device(pdev);
    pcim_iomap_regions(pdev, 1 << 0, KBUILD_MODNAME);

    /* 2. Read number of GPIOs from GPIO_CONST register */
    ngpio = readq(base + GPIO_CONST) & GPIO_CONST_GPIOS_MASK;

    /* 3. Enable MSI-X — one vector per GPIO line */
    pci_enable_msix_range(pdev, txgpio->msix_entries, ngpio, ngpio);

    /* 4. Fill gpio_chip ops */
    chip->request          = thunderx_gpio_request;
    chip->direction_input  = thunderx_gpio_dir_in;
    chip->direction_output = thunderx_gpio_dir_out;
    chip->get              = thunderx_gpio_get;
    chip->set              = thunderx_gpio_set;
    chip->set_multiple     = thunderx_gpio_set_multiple;
    chip->set_config       = thunderx_gpio_set_config;
    chip->get_direction    = thunderx_gpio_get_direction;

    /* 5. Fill gpio_irq_chip for hierarchical IRQ domain */
    girq->parent_domain        = irq_get_irq_data(msix_entries[0].vector)->domain;
    girq->child_to_parent_hwirq = thunderx_gpio_child_to_parent_hwirq;
    girq->populate_parent_alloc_arg = thunderx_gpio_populate_parent_alloc_info;

    /* 6. Register with GPIOlib */
    devm_gpiochip_add_data(dev, chip, txgpio);

    /* 7. Push IRQ domain entries — link each GPIO line to its MSI-X */
    for (i = 0; i < ngpio; i++)
        irq_domain_push_irq(girq->domain, msix_entries[i].vector, &fwspec);
}
```

---

### الـ Hardware Register Map في ThunderX

```
BAR0 + 0x000  → GPIO_RX_DAT      : قراءة حالة الـ 64 pin الأولى (bank 0)
BAR0 + 0x008  → GPIO_TX_SET      : set bits بالـ atomically
BAR0 + 0x010  → GPIO_TX_CLR      : clear bits atomically
BAR0 + 0x090  → GPIO_CONST       : [7:0] = ngpio, [15:8] = base_msi
BAR0 + 0x400 + 8*n → GPIO_BIT_CFG[n] : per-pin config
BAR0 + 0x800 + 8*n → GPIO_INTR[n]   : per-pin interrupt control
BAR0 + 0x1400 → GPIO_2ND_BANK    : بداية الـ bank الثاني (pins 64-127)
```

الـ `GPIO_BIT_CFG[n]` هو قلب الـ per-pin config:

```
bit  0      → TX_OE      : output enable
bit  1      → PIN_XOR    : invert polarity (مهم لـ open-drain و falling-edge IRQ)
bit  2      → INT_EN     : enable interrupt
bit  3      → INT_TYPE   : 0=level, 1=edge
bits[11:4]  → FIL_MASK   : glitch filter (sel + count)
bit 12      → TX_OD      : open-drain mode
bits[25:16] → PIN_SEL    : 0=GPIO mode, غيره = alternate function
```

---

### الـ IRQ Flow في ThunderX — Hierarchical Domain

ده مش الـ simple cascaded interrupt model. ThunderX بيستخدم **hierarchical IRQ domain** — اللي بتحتاج فهم الـ IRQ subsystem:

> **الـ IRQ Domain** هو الـ mapping بين hardware interrupt numbers (hwirq) والـ Linux virtual IRQ numbers (virq). الـ hierarchical domains بتخلي domain يكون "child" من domain تاني، وكل level بيعمل translate للـ hwirq للـ level اللي فوقه.

```
GPIO Consumer
    │ request_irq(virq_for_gpio_5)
    ▼
GPIO IRQ Domain (child)
    │ GPIO hwirq=5  →  parent hwirq = MSI-X vector for GPIO 5
    ▼
MSI-X IRQ Domain (parent)
    │ MSI-X hwirq  →  APIC/GIC interrupt
    ▼
Hardware: ThunderX MSI-X → MSIX table → CPU interrupt
```

الـ `child_to_parent_hwirq` callback هو اللي بيعمل الـ translation:

```c
static int thunderx_gpio_child_to_parent_hwirq(struct gpio_chip *gc,
                                               unsigned int child,  /* GPIO offset */
                                               unsigned int child_type,
                                               unsigned int *parent, /* MSI-X hwirq */
                                               unsigned int *parent_type)
{
    struct thunderx_gpio *txgpio = gpiochip_get_data(gc);
    struct irq_data *irqd;
    unsigned int irq;

    /* get the Linux IRQ number for this GPIO's MSI-X vector */
    irq = txgpio->msix_entries[child].vector;
    irqd = irq_domain_get_irq_data(gc->irq.parent_domain, irq);
    *parent = irqd_to_hwirq(irqd);         /* parent hwirq */
    *parent_type = IRQ_TYPE_LEVEL_HIGH;
    return 0;
}
```

---

### ما الـ GPIOlib بتملكه vs. ما بتفوّضه للـ Driver

| المسؤولية | الـ GPIOlib تملكه | الـ Driver بيعمله |
|---|---|---|
| Global numbering | نعم — `chip->base` وبتخصص dynamic | لأ |
| الـ `gpio_desc` management | نعم — array of descs | لأ |
| Request/free tracking | نعم — flags على الـ desc | لأ |
| sysfs / debugfs export | نعم — `/sys/class/gpio/` | لأ |
| device tree lookup | نعم — `gpiod_get()` + OF parser | لأ |
| قراءة/كتابة الـ hardware | لأ | نعم — `get()`, `set()` |
| ضبط direction | لأ | نعم — `direction_input/output()` |
| Glitch filter / open-drain | لأ | نعم — `set_config()` |
| IRQ masking/acking | لأ | نعم — `irq_chip` ops |
| MSI-X allocation | لأ | نعم — `pci_enable_msix_range()` |
| IRQ domain hierarchy | توفير الـ infrastructure | نعم — يعمل `child_to_parent_hwirq` |
| Lockdep annotations | نعم — `lock_key` في `gpio_irq_chip` | لأ |

---

### الـ Device Tree Binding — ربط الـ Hardware بالـ Framework

الملف `gpio-thunderx.txt` بيوصف الـ DT node للـ controller:

```dts
gpio_6_0: gpio@6,0 {
    compatible = "cavium,thunder-8890-gpio";
    reg = <0x3000 0 0 0 0>;   /* PCI DEVFN = 0x30 (bus 6, func 0) */
    gpio-controller;           /* marks this node as GPIO provider */
    #gpio-cells = <2>;         /* <pin_number, flags> */
    interrupt-controller;      /* also an IRQ provider */
    #interrupt-cells = <2>;    /* <pin_number, trigger_flags> */
};
```

لكن في الواقع، ThunderX **مش بتستخدم الـ compatible string للـ probe** — لأن الـ probe بيحصل عن طريق الـ PCI ID `(0x177D, 0xA00A)`. الـ DT node بيتستخدم بس عشان الـ GPIOlib تعرف إزاي تعمل lookup للـ GPIO lines من الـ consumers اللي في الـ device tree.

الـ `#gpio-cells = <2>` معناه إن كل GPIO reference في الـ DT بتيجي بـ cell اتنين:
- **Cell 0:** رقم الـ pin (offset داخل الـ controller)
- **Cell 1:** flags (`GPIO_ACTIVE_HIGH`, `GPIO_ACTIVE_LOW`, `GPIO_OPEN_DRAIN` — من `<dt-bindings/gpio/gpio.h>`)

مثال consumer:

```dts
some-device {
    reset-gpios = <&gpio_6_0 5 GPIO_ACTIVE_LOW>;
    /* GPIO pin 5 على الـ controller gpio_6_0، active low */
};
```

الـ consumer driver بيعمل:

```c
struct gpio_desc *rst = devm_gpiod_get(dev, "reset", GPIOD_OUT_HIGH);
/* الـ GPIOlib بتعمل lookup في الـ DT، بتلاقي gpio_6_0، بتستدعي thunderx_gpio_request() */
gpiod_set_value(rst, 1); /* → thunderx_gpio_set() → writeq() على الـ BAR0 */
```
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Register Map — Overview سريع

| Register | Offset | الوظيفة |
|---|---|---|
| `GPIO_RX_DAT` | `0x000` | قراءة القيمة الحالية للـ pins (input data) |
| `GPIO_TX_SET` | `0x008` | كتابة `1` يـ set الـ bit (output high) |
| `GPIO_TX_CLR` | `0x010` | كتابة `1` يـ clear الـ bit (output low) |
| `GPIO_CONST` | `0x090` | يحتوي عدد الـ GPIOs وبداية الـ MSI index |
| `GPIO_BIT_CFG` | `0x400 + 8*n` | إعدادات الـ pin رقم n (direction, interrupt, filter...) |
| `GPIO_INTR` | `0x800 + 8*n` | إدارة الـ interrupt للـ pin رقم n |
| `GPIO_2ND_BANK` | `0x1400` | offset للـ bank الثاني (pins 64-127) |

---

### الـ GPIO_BIT_CFG Flags — Cheatsheet

| Bit/Field | الماكرو | المعنى |
|---|---|---|
| Bit 0 | `GPIO_BIT_CFG_TX_OE` | Output Enable — الـ pin يبقط output |
| Bit 1 | `GPIO_BIT_CFG_PIN_XOR` | قلب قيمة الـ pin (invert polarity) |
| Bit 2 | `GPIO_BIT_CFG_INT_EN` | تفعيل الـ interrupt لهذا الـ pin |
| Bit 3 | `GPIO_BIT_CFG_INT_TYPE` | `0` = level, `1` = edge triggered |
| Bits 11:4 | `GPIO_BIT_CFG_FIL_MASK` | Glitch filter (SEL + CNT) |
| Bits 4:7 | `GPIO_BIT_CFG_FIL_CNT_SHIFT=4` | عداد الـ filter |
| Bits 8:11 | `GPIO_BIT_CFG_FIL_SEL_SHIFT=8` | اختيار الـ clock divisor للـ filter |
| Bit 12 | `GPIO_BIT_CFG_TX_OD` | Open-Drain mode |
| Bits 25:16 | `GPIO_BIT_CFG_PIN_SEL_MASK` | اختيار signal مش GPIO — لو != 0 الـ pin مش GPIO |

---

### الـ GPIO_INTR Flags — Cheatsheet

| Bit | الماكرو | المعنى |
|---|---|---|
| Bit 0 | `GPIO_INTR_INTR` | الـ interrupt status — W1C (write 1 to clear = ACK) |
| Bit 1 | `GPIO_INTR_INTR_W1S` | set الـ interrupt يدوياً (test) |
| Bit 2 | `GPIO_INTR_ENA_W1C` | disable الـ interrupt (write 1 to clear enable) |
| Bit 3 | `GPIO_INTR_ENA_W1S` | enable الـ interrupt (write 1 to set enable) |

---

### الـ Pin Config Options (pinconf)

| الـ Config | الدعم | ملاحظة |
|---|---|---|
| `PIN_CONFIG_DRIVE_OPEN_DRAIN` | مدعوم | بيعمل hardware inversion — بيتسجل في `invert_mask` تلقائياً |
| `PIN_CONFIG_DRIVE_PUSH_PULL` | مدعوم | يلغي الـ open-drain والـ inversion |
| `PIN_CONFIG_INPUT_DEBOUNCE` | مدعوم | max ~1228µs — يحسب SEL/CNT للـ filter |

---

### الـ Structs الأساسية

#### `struct thunderx_line`

الـ struct ده بيمثل معلومات **كل pin على حدة**، بيُستخدم كـ per-line metadata.

```c
struct thunderx_line {
    struct thunderx_gpio *txgpio;  /* pointer للـ parent controller */
    unsigned int          line;    /* رقم الـ pin */
    unsigned int          fil_bits; /* قيمة الـ glitch filter المحفوظة */
};
```

- **`txgpio`**: back-pointer للـ parent بيُستخدم في الـ IRQ handlers اللي بتاخد `irq_data` بس مش `gpio_chip`.
- **`fil_bits`**: بيُحفظ هنا عشان لو الـ driver غيّر الـ direction أو الـ config، يرجع يحط نفس الـ filter من غير ما يضيعه.

---

#### `struct thunderx_gpio`

الـ struct الرئيسي — بيمثل الـ **GPIO controller كله**.

```c
struct thunderx_gpio {
    struct gpio_chip    chip;           /* embedded gpio_chip — واجهة الـ kernel */
    u8 __iomem         *register_base; /* base address بعد الـ iomap */
    struct msix_entry  *msix_entries;  /* array — واحد لكل pin */
    struct thunderx_line *line_entries; /* array — metadata لكل pin */
    raw_spinlock_t      lock;          /* يحمي الـ register writes + البت masks */
    unsigned long       invert_mask[2]; /* بتماسك الـ pins اللي مقلوبة (2 banks × 64 bit) */
    unsigned long       od_mask[2];    /* بتماسك الـ pins اللي في open-drain mode */
    int                 base_msi;      /* أول MSI-X entry index في الـ PCI config */
};
```

**أهم الـ fields:**

| Field | الحجم | الوظيفة |
|---|---|---|
| `chip` | كبير (embedded) | واجهة الـ GPIO subsystem — لازم يكون أول field |
| `register_base` | pointer | MMIO base — كل الـ register access بيمر عليه |
| `msix_entries` | `ngpio × msix_entry` | كل pin عنده MSI-X vector خاص بيه |
| `line_entries` | `ngpio × thunderx_line` | metadata + filter لكل pin |
| `lock` | `raw_spinlock_t` | يحمي الـ atomic register operations |
| `invert_mask[2]` | 128 bit | bitmap — يتعلم الـ pins اللي polarity بتاعتها مقلوبة |
| `od_mask[2]` | 128 bit | bitmap — يتعلم الـ pins اللي في open-drain |
| `base_msi` | int | بيُضاف على `2 * line` عشان يحسب الـ MSI entry index |

---

### مخطط علاقات الـ Structs (ASCII)

```
 ┌──────────────────────────────────────────────────────────────┐
 │                    struct thunderx_gpio                       │
 │                                                              │
 │  ┌──────────────────┐   ┌────────────────────────────────┐  │
 │  │  struct gpio_chip │   │  struct thunderx_line[]        │  │
 │  │  (embedded)       │   │  [0] { txgpio*, line=0, fil }  │  │
 │  │                   │   │  [1] { txgpio*, line=1, fil }  │  │
 │  │  .irq ──────────┐ │   │  ...                           │  │
 │  │  .get/set/dir.. │ │   │  [n] { txgpio*, line=n, fil }  │  │
 │  └──────────────────┘ │   └────────────────────────────────┘  │
 │                       │                                      │
 │  ┌────────────────────┤   ┌────────────────────────────────┐  │
 │  │ struct gpio_irq_chip│  │  struct msix_entry[]           │  │
 │  │ (via chip.irq)     │   │  [0] { entry=base+0, vector }  │  │
 │  │ .parent_domain ────┼──▶│  [1] { entry=base+2, vector }  │  │
 │  │ .child_to_parent.. │   │  ...                           │  │
 │  └────────────────────┘   └────────────────────────────────┘  │
 │                                                              │
 │  register_base ──▶ [ MMIO region: GPIO_RX_DAT, BIT_CFG... ]  │
 │  lock (raw_spinlock)                                         │
 │  invert_mask[2]  (128-bit bitmap)                            │
 │  od_mask[2]      (128-bit bitmap)                            │
 └──────────────────────────────────────────────────────────────┘

 ┌─────────────────────────────────────────────────┐
 │           struct thunderx_line[i]               │
 │  txgpio ──────────────────────────▶ thunderx_gpio│
 │  line   = i                                      │
 │  fil_bits = glitch filter config                 │
 └─────────────────────────────────────────────────┘

 ┌────────────────────────────────────────────────────────┐
 │                   struct pci_dev                       │
 │  drvdata ──────────────────────────▶ thunderx_gpio     │
 └────────────────────────────────────────────────────────┘
```

---

### الـ Lifecycle — من الـ probe للـ remove

```
pci_driver.probe(pdev)
  │
  ├─▶ devm_kzalloc(thunderx_gpio)
  │     └─▶ raw_spin_lock_init(&txgpio->lock)
  │
  ├─▶ pcim_enable_device(pdev)
  ├─▶ pcim_iomap_regions(pdev, BAR0)
  │     └─▶ txgpio->register_base = tbl[0]
  │
  ├─▶ readq(GPIO_CONST)
  │     └─▶ ngpio = c & 0xFF
  │     └─▶ base_msi = (c >> 8) & 0xFF
  │
  ├─▶ devm_kcalloc(msix_entries[ngpio])
  ├─▶ devm_kcalloc(line_entries[ngpio])
  │
  ├─▶ for i in 0..ngpio:
  │     ├─▶ read bit_cfg register
  │     ├─▶ msix_entries[i].entry = base_msi + 2*i
  │     ├─▶ line_entries[i] = { .line=i, .txgpio=txgpio, .fil_bits=... }
  │     └─▶ sync od_mask / invert_mask from hardware
  │
  ├─▶ pci_enable_msix_range(ngpio vectors)
  │
  ├─▶ gpio_chip setup (ops, ngpio, label...)
  ├─▶ gpio_irq_chip setup (parent_domain, handlers...)
  │
  ├─▶ devm_gpiochip_add_data(chip, txgpio)
  │     └─▶ الـ GPIO subsystem يسجل الـ chip
  │
  └─▶ for i in 0..ngpio:
        └─▶ irq_domain_push_irq(domain, msix_entries[i].vector, fwspec)
              └─▶ يربط كل MSI-X vector بـ GPIO IRQ domain

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

pci_driver.remove(pdev)
  │
  ├─▶ for i in 0..ngpio:
  │     └─▶ irq_domain_pop_irq(domain, msix_entries[i].vector)
  │
  └─▶ irq_domain_remove(chip.irq.domain)
        └─▶ devm يتكفل بتحرير باقي الموارد تلقائياً
```

---

### Call Flow — GPIO Direction & Value

#### `direction_input(chip, line)`

```
user / kernel subsystem calls: chip->direction_input(chip, line)
  │
  ├─▶ thunderx_gpio_is_gpio(txgpio, line)
  │     └─▶ readq(GPIO_BIT_CFG[line])
  │           └─▶ GPIO_BIT_CFG_PIN_SEL_MASK == 0 ? يكمل : -EIO
  │
  ├─▶ raw_spin_lock(&txgpio->lock)
  │
  ├─▶ clear_bit(line, invert_mask)
  ├─▶ clear_bit(line, od_mask)
  ├─▶ writeq(fil_bits only, GPIO_BIT_CFG[line])
  │     └─▶ TX_OE = 0 → الـ pin يبقى input
  │
  └─▶ raw_spin_unlock
```

#### `direction_output(chip, line, value)`

```
chip->direction_output(chip, line, value)
  │
  ├─▶ thunderx_gpio_is_gpio(...)
  ├─▶ raw_spin_lock
  │
  ├─▶ thunderx_gpio_set(chip, line, value)
  │     └─▶ writeq(BIT(bank_bit), GPIO_TX_SET or GPIO_TX_CLR)
  │
  ├─▶ bit_cfg = fil_bits | TX_OE
  │     ├─▶ invert_mask[line]? → bit_cfg |= PIN_XOR
  │     └─▶ od_mask[line]?    → bit_cfg |= TX_OD
  │
  ├─▶ writeq(bit_cfg, GPIO_BIT_CFG[line])
  └─▶ raw_spin_unlock
```

#### `set_multiple(chip, mask, bits)`

```
chip->set_multiple(chip, mask, bits)
  │
  └─▶ for bank in 0..ngpio/64:
        ├─▶ set_bits   = bits[bank] & mask[bank]
        ├─▶ clear_bits = ~bits[bank] & mask[bank]
        ├─▶ writeq(set_bits,   GPIO_TX_SET[bank])
        └─▶ writeq(clear_bits, GPIO_TX_CLR[bank])
            (بدون lock — كل عملية atomic في الـ hardware)
```

---

### Call Flow — الـ Interrupts

#### تفعيل interrupt على pin

```
irq subsystem calls: irq_chip->irq_set_type(d, flow_type)
  │
  ├─▶ irqd_set_trigger_type(d, flow_type)
  │
  ├─▶ bit_cfg = fil_bits | GPIO_BIT_CFG_INT_EN
  │
  ├─▶ flow_type & IRQ_TYPE_EDGE_BOTH ?
  │     ├─▶ yes: irq_set_handler_locked(handle_fasteoi_ack_irq)
  │     │         bit_cfg |= GPIO_BIT_CFG_INT_TYPE  (edge mode)
  │     └─▶ no:  irq_set_handler_locked(handle_fasteoi_mask_irq)
  │
  ├─▶ raw_spin_lock
  ├─▶ FALLING or LEVEL_LOW? → bit_cfg |= PIN_XOR, set_bit(invert_mask)
  │                  else   → clear_bit(invert_mask)
  ├─▶ clear_bit(od_mask)
  ├─▶ writeq(bit_cfg, GPIO_BIT_CFG[line])
  └─▶ raw_spin_unlock
```

#### سلسلة الـ IRQ عند حدوث interrupt

```
Hardware GPIO pin تغيّرت
  │
  └─▶ MSI-X vector يُطلق interrupt (PCI)
        │
        └─▶ parent irq domain handler
              │
              └─▶ thunderx_gpio irq domain (chained)
                    │
                    ├─▶ handle_fasteoi_ack_irq   (edge)
                    │     └─▶ thunderx_gpio_irq_ack()
                    │           └─▶ writeq(GPIO_INTR_INTR, INTR[line])  ← W1C
                    │
                    └─▶ handle_fasteoi_mask_irq  (level)
                          └─▶ thunderx_gpio_irq_mask_ack()
                                └─▶ writeq(ENA_W1C | INTR_INTR, INTR[line])
```

#### enable / disable الـ interrupt

```
irq_enable(d):
  gpiochip_enable_irq(gc, hwirq)   ← يُبلّغ الـ GPIO subsystem
  irq_chip_enable_parent(d)         ← يُبلّغ الـ parent (MSI-X)
  thunderx_gpio_irq_unmask(d)
    └─▶ writeq(GPIO_INTR_ENA_W1S, INTR[line])  ← يفتح الـ interrupt

irq_disable(d):
  thunderx_gpio_irq_mask(d)
    └─▶ writeq(GPIO_INTR_ENA_W1C, INTR[line])  ← يقفل الـ interrupt
  irq_chip_disable_parent(d)
  gpiochip_disable_irq(gc, hwirq)
```

---

### Call Flow — ربط الـ MSI-X بالـ GPIO IRQ Domain

```
thunderx_gpio_probe()
  │
  └─▶ girq->child_to_parent_hwirq = thunderx_gpio_child_to_parent_hwirq
        │
        └─▶ عند طلب irq mapping:
              child (GPIO line number)
                └─▶ txgpio->msix_entries[child].vector
                      └─▶ irq_domain_get_irq_data(parent_domain, vector)
                            └─▶ *parent = irqd_to_hwirq(irqd)
                                *parent_type = IRQ_TYPE_LEVEL_HIGH
```

---

### استراتيجية الـ Locking

#### `raw_spinlock_t txgpio->lock`

الـ `raw_spinlock` (مش `spinlock` العادي) بيُستخدم عشان الكود لازم يشتغل حتى في الـ **PREEMPT_RT** kernels من غير ما يُعطل الـ real-time guarantees.

| العملية | هل تاخد الـ lock؟ | السبب |
|---|---|---|
| `dir_in` | نعم | بيكتب `GPIO_BIT_CFG` + بيعدل `invert_mask` / `od_mask` |
| `dir_out` | نعم | نفس السبب |
| `set_config` | نعم | بيقرأ ثم يعدل الـ mask bits وبيكتب الـ register |
| `irq_set_type` | نعم | بيعدل `invert_mask` وبيكتب `GPIO_BIT_CFG` |
| `get` | لا | read-only operation — الـ hardware atomic |
| `set` | لا | كل write لـ `TX_SET`/`TX_CLR` atomic في الـ hardware |
| `set_multiple` | لا | نفس السبب — write per bank |
| `irq_ack/mask/unmask` | لا | write واحدة لـ `GPIO_INTR` — atomic |

#### ترتيب الـ Locking (lock ordering)

```
raw_spinlock (txgpio->lock)
    │
    └─▶ لا يوجد lock آخر بيُأخذ جوّه ← آمن من deadlock

irq_set_handler_locked(d, ...)
    └─▶ بيُستدعى قبل raw_spin_lock في irq_set_type
        عشان يحافظ على الـ IRQ core lock ordering
```

#### ملاحظة على الـ `set_multiple`

الـ `TX_SET` و `TX_CLR` registers مصممة hardware-atomic — كتابة `1` في أي bit = set أو clear الـ bit ده بس من غير ما يأثر على باقي الـ bits. عشان كده الـ `set_multiple` مش محتاج lock على الإطلاق.

---

### مخطط شامل — الـ Architecture كاملة

```
 ┌──────────────┐     PCI Bus      ┌──────────────────────────────────┐
 │  pci_driver  │ ───────────────▶ │       ThunderX SoC               │
 │  .probe()    │                  │  ┌────────────────────────────┐   │
 └──────────────┘                  │  │  GPIO MMIO Region (BAR0)   │   │
        │                          │  │  GPIO_RX_DAT   @0x000      │   │
        ▼                          │  │  GPIO_TX_SET   @0x008      │   │
 ┌──────────────────────────────┐  │  │  GPIO_TX_CLR   @0x010      │   │
 │     thunderx_gpio            │  │  │  GPIO_CONST    @0x090      │   │
 │  ┌────────────────────────┐  │  │  │  GPIO_BIT_CFG  @0x400+8n  │   │
 │  │     gpio_chip           │  │◀─▶│  │  GPIO_INTR     @0x800+8n  │   │
 │  │  (GPIO subsystem API)   │  │  │  └────────────────────────────┘   │
 │  │  .get/.set/.dir_in/out  │  │  │                                   │
 │  │  .set_config            │  │  │  ┌──────────────────────────┐    │
 │  └────────────────────────┘  │  │  │  MSI-X Vectors           │    │
 │                               │  │  │  [0] → IRQ for pin 0     │    │
 │  ┌────────────────────────┐  │  │  │  [1] → IRQ for pin 1     │    │
 │  │   gpio_irq_chip         │  │◀─▶│  │  ...                     │    │
 │  │   (IRQ subsystem API)   │  │  │  └──────────────────────────┘    │
 │  │  .enable/.mask/.ack...  │  │  └──────────────────────────────────┘
 │  └────────────────────────┘  │
 │                               │
 │  msix_entries[]  (per pin)    │
 │  line_entries[]  (per pin)    │
 │  invert_mask[2]  (128 bits)   │
 │  od_mask[2]      (128 bits)   │
 │  raw_spinlock    lock         │
 └──────────────────────────────┘
        │
        ▼
 ┌──────────────────────────────┐
 │   Linux GPIO Subsystem        │
 │   /sys/class/gpio/...         │
 │   gpiod_get / gpiod_set_value │
 └──────────────────────────────┘
        │
        ▼
 ┌──────────────────────────────┐
 │   Linux IRQ Subsystem         │
 │   request_irq / free_irq      │
 │   IRQ domain (hierarchical)   │
 └──────────────────────────────┘
```
## Phase 4: شرح الـ Functions

---

### ملخص عام: الـ Binding Properties والـ Driver API

الـ `gpio-thunderx` binding بيوصف الـ GPIO controller الخاص بـ Cavium ThunderX/OCTEON-TX. اللي بيميزه إنه بيتعرف عن طريق الـ **PCI subsystem** مش عن طريق الـ compatible string التقليدي — يعني الـ DT binding جزئي، والـ driver الحقيقي هو PCI driver.

---

### الـ Binding Properties — Cheatsheet

| Property | Required? | Value | الوصف |
|---|---|---|---|
| `reg` | نعم | `<devfn 0 0 0 0>` | عنوان الـ PCI device في الـ bus |
| `gpio-controller` | نعم | (flag) | يعرّف الـ node كـ GPIO controller |
| `#gpio-cells` | نعم | `<2>` | عدد cells لوصف GPIO specifier |
| `compatible` | اختياري | `"cavium,thunder-8890-gpio"` | unused فعلياً — الـ PCI ID هو اللي بيعمل match |
| `interrupt-controller` | اختياري | (flag) | يعرّف الـ node كـ interrupt controller |
| `#interrupt-cells` | اختياري (لازم مع interrupt-controller) | `<2>` | عدد cells لوصف interrupt specifier |

---

### الـ Driver Functions — Cheatsheet

| Function | Category | الدور |
|---|---|---|
| `thunderx_gpio_probe` | Registration | entry point لتسجيل الـ GPIO chip عبر PCI |
| `thunderx_gpio_remove` | Cleanup | إزالة الـ irq domain وتنظيف الـ MSI-X |
| `thunderx_gpio_request` | Runtime/GPIO | التحقق إن الـ pin متاح كـ GPIO |
| `thunderx_gpio_dir_in` | Runtime/GPIO | ضبط الـ pin كـ input |
| `thunderx_gpio_dir_out` | Runtime/GPIO | ضبط الـ pin كـ output |
| `thunderx_gpio_get` | Runtime/GPIO | قراءة قيمة الـ pin |
| `thunderx_gpio_set` | Runtime/GPIO | كتابة قيمة الـ pin |
| `thunderx_gpio_set_multiple` | Runtime/GPIO | كتابة أكتر من pin في نفس الوقت |
| `thunderx_gpio_get_direction` | Runtime/GPIO | معرفة direction الـ pin |
| `thunderx_gpio_set_config` | Runtime/GPIO | ضبط open-drain / push-pull / debounce |
| `thunderx_gpio_irq_ack` | IRQ | إزالة الـ interrupt flag |
| `thunderx_gpio_irq_mask` | IRQ | تعطيل الـ interrupt |
| `thunderx_gpio_irq_mask_ack` | IRQ | mask + ack في operation واحدة |
| `thunderx_gpio_irq_unmask` | IRQ | تفعيل الـ interrupt |
| `thunderx_gpio_irq_set_type` | IRQ | ضبط نوع الـ trigger (edge/level) |
| `thunderx_gpio_irq_enable` | IRQ | full enable مع parent chaining |
| `thunderx_gpio_irq_disable` | IRQ | full disable مع parent chaining |
| `thunderx_gpio_child_to_parent_hwirq` | IRQ/Hierarchy | mapping من GPIO line لـ MSI-X vector |
| `thunderx_gpio_populate_parent_alloc_info` | IRQ/Hierarchy | ملء الـ msi_alloc_info للـ parent domain |
| `thunderx_gpio_is_gpio` | Helpers | التحقق إن الـ pin مش مستخدم لـ alternate function |
| `bit_cfg_reg` / `intr_reg` | Helpers | حساب عنوان الـ register |

---

### فئة 1: الـ Device Tree Binding Properties

#### ###  `reg`

```
reg = <0x3000 0 0 0 0>;  /* DEVFN = 0x30 → Bus 0, Device 6, Function 0 */
```

الـ `reg` بتعرّف مكان الـ PCI device في الـ bus. الـ first cell هي الـ **BDF (Bus:Device:Function)** المشفرة. الـ ThunderX GPIO device بيتعرف بيها أولاً قبل أي DT compatible string.

- **الخلية الأولى**: `(bus << 16) | (device << 11) | (function << 8)` — هنا `0x3000` = device 6، function 0.
- **باقي الـ 4 cells**: أصفار — لأن الـ PCI binding بيستخدم 5-cell format لكن العنوان الفعلي في الـ cell الأولى.

لو الـ `reg` غلط، الـ `thunderx_gpio_probe` مش هيُستدعى أصلاً لأن الـ PCI bus scan هيفشل في match الـ device.

---

#### ### `gpio-controller`

```
gpio-controller;
```

الـ flag ده **مطلوب إجبارياً** — بيخلي الـ kernel يعامل الـ node ده كـ provider للـ GPIO lines. بدونه، أي consumer بيعمل `gpiod_get()` مش هيقدر يعمل reference للـ pins دي.

---

#### ### `#gpio-cells = <2>`

```
#gpio-cells = <2>;
```

بيحدد إن كل GPIO specifier بيتكون من **خليتين**:

| Cell | المعنى |
|---|---|
| الأولى | رقم الـ GPIO pin بالنسبة للـ controller |
| الثانية | الـ flags (active high/low) — مُعرّفة في `gpio.txt` |

مثال من consumer:
```dts
reset-gpios = <&gpio_6_0 15 GPIO_ACTIVE_LOW>;
/*                         ^pin  ^flags */
```

---

#### ### `interrupt-controller` و `#interrupt-cells = <2>`

```
interrupt-controller;
#interrupt-cells = <2>;
```

هذين معاً بيحولوا الـ GPIO node لـ **interrupt provider**. كل GPIO line تقدر تولد interrupt مستقل عبر **MSI-X vector** خاص بيها.

| Cell | المعنى |
|---|---|
| الأولى | رقم الـ GPIO pin |
| الثانية | نوع الـ trigger — `IRQ_TYPE_EDGE_RISING`, `IRQ_TYPE_LEVEL_HIGH` إلخ |

مثال:
```dts
interrupts-extended = <&gpio_6_0 10 IRQ_TYPE_EDGE_RISING>;
```

الـ driver بيدعم: `IRQ_TYPE_EDGE_RISING`, `IRQ_TYPE_EDGE_FALLING`, `IRQ_TYPE_EDGE_BOTH`, `IRQ_TYPE_LEVEL_HIGH`, `IRQ_TYPE_LEVEL_LOW`.

---

#### ### `compatible = "cavium,thunder-8890-gpio"`

```
compatible = "cavium,thunder-8890-gpio";  /* optional, unused */
```

الـ `compatible` string موجود في الـ binding بس **مش بيُستخدم** فعلياً للـ matching. الـ driver هو `pci_driver` بيعمل match عبر:

```c
static const struct pci_device_id thunderx_gpio_id_table[] = {
    { PCI_DEVICE(PCI_VENDOR_ID_CAVIUM, 0xA00A) },
    { 0, }
};
```

يعني الـ Vendor ID `0x177D` + Device ID `0xA00A` هم اللي بيحددوا الـ device، مش الـ DT compatible.

---

### فئة 2: Registration — `thunderx_gpio_probe`

```c
static int thunderx_gpio_probe(struct pci_dev *pdev,
                               const struct pci_device_id *id)
```

الـ probe هي الـ entry point الوحيدة للـ driver. بتنفذ كل steps التهيئة من allocate الـ structs لحد register الـ gpio_chip والـ irq domain.

**Parameters:**
- `pdev`: الـ PCI device struct اللي match الـ `thunderx_gpio_id_table`.
- `id`: الـ entry في الـ ID table اللي عملت الـ match — مش بتُستخدم في الـ function.

**Return:** `0` نجاح، أو error code سلبي.

**Pseudocode Flow:**

```
probe():
  1. devm_kzalloc → txgpio struct
  2. raw_spin_lock_init(&txgpio->lock)
  3. pcim_enable_device → enable PCI
  4. pcim_iomap_regions → BAR0 mapping
  5. read GPIO_CONST register:
       if CN88XX (subsystem 0xa10a): ngpio=50, base_msi=48
       else: ngpio = GPIO_CONST[7:0], base_msi = GPIO_CONST[15:8]
  6. devm_kcalloc → msix_entries[ngpio]
  7. devm_kcalloc → line_entries[ngpio]
  8. for each line:
       read GPIO_BIT_CFG
       set msix_entries[i].entry = base_msi + 2*i
       set fil_bits (existing or GLITCH_FILTER_400NS)
       restore od_mask, invert_mask from hardware
  9. pci_enable_msix_range(ngpio, ngpio) → exact ngpio vectors
  10. fill gpio_chip callbacks
  11. fill gpio_irq_chip:
        parent_domain = domain of msix_entries[0].vector
        child_to_parent_hwirq = thunderx_gpio_child_to_parent_hwirq
  12. devm_gpiochip_add_data → register chip
  13. for each line:
        irq_domain_push_irq(domain, msix_vector, fwspec)
```

**Key details:**
- الـ `raw_spinlock_t` (مش `spinlock_t`) لأن بعض الـ paths بتُستدعى من interrupt context حتى في realtime kernels.
- الـ `devm_*` allocations بتضمن cleanup تلقائي لو الـ probe فشل في أي مرحلة.
- الـ MSI-X entries بتُحجز بـ `base_msi + 2*i` — يعني كل line عندها **vector زوجي** (even offset) في الـ MSI-X table.

---

### فئة 3: Cleanup — `thunderx_gpio_remove`

```c
static void thunderx_gpio_remove(struct pci_dev *pdev)
```

بتعكس الـ probe بالترتيب. أهم خطوة هي `irq_domain_pop_irq` لكل line قبل `irq_domain_remove`.

**Key details:**
- الـ GPIO chip نفسه بيتزيل تلقائياً عبر `devm_gpiochip_add_data` (devres).
- لازم pop الـ irq domains **قبل** remove الـ domain، وإلا kernel panic.

---

### فئة 4: GPIO Runtime Operations

#### ### `thunderx_gpio_request`

```c
static int thunderx_gpio_request(struct gpio_chip *chip, unsigned int line)
```

بتتحقق إن الـ `GPIO_BIT_CFG_PIN_SEL_MASK` == 0 قبل ما أي consumer يستخدم الـ pin. لو الـ pin مستخدم لـ alternate function (مثلاً UART أو I2C)، الـ field ده بيكون non-zero.

- **Return:** `0` لو متاح، `-EIO` لو مستخدم لـ alternate function.
- **Caller:** الـ GPIO core عند أول `gpiod_get()` أو `gpio_request()`.

---

#### ### `thunderx_gpio_dir_in`

```c
static int thunderx_gpio_dir_in(struct gpio_chip *chip, unsigned int line)
```

بتضبط الـ pin كـ input بكتابة `fil_bits` فقط في `GPIO_BIT_CFG` — بدون `TX_OE` bit يعني input mode. كمان بتعمل `clear_bit` لكل من `invert_mask` و `od_mask`.

- **Locking:** `raw_spin_lock` لحماية الـ register write.
- **Side effects:** بتعمل reset لأي invert أو open-drain configuration سابقة.

---

#### ### `thunderx_gpio_dir_out`

```c
static int thunderx_gpio_dir_out(struct gpio_chip *chip, unsigned int line,
                                 int value)
```

بتضبط الـ pin كـ output مع قيمة ابتدائية. بتبني الـ `bit_cfg` من `fil_bits | TX_OE` وبتضيف `PIN_XOR` لو invert_mask مضبوط، و `TX_OD` لو open-drain مفعّل.

- **Parameters:** `value` هي القيمة الابتدائية (0 أو 1).
- **Key detail:** `thunderx_gpio_set()` بتُستدعى **داخل** الـ spinlock قبل كتابة الـ bit_cfg.

---

#### ### `thunderx_gpio_get` / `thunderx_gpio_set`

```c
static int thunderx_gpio_get(struct gpio_chip *chip, unsigned int line)
static int thunderx_gpio_set(struct gpio_chip *chip, unsigned int line, int value)
```

**الـ `get`:** بتقرأ `GPIO_RX_DAT` — الـ receive data register. لو الـ `invert_mask` مضبوط، بتعكس القيمة. بتدعم الـ second bank (pins 64-127) عبر `bank * GPIO_2ND_BANK` offset.

**الـ `set`:** بتكتب في إما `GPIO_TX_SET` أو `GPIO_TX_CLR` — مش `GPIO_TX_DAT`. ده نهج **read-free atomic** لتجنب الـ read-modify-write race. كل register بيستقبل bitmask:
- كتابة `BIT(n)` في `GPIO_TX_SET` → يضبط pin n على 1
- كتابة `BIT(n)` في `GPIO_TX_CLR` → يضبط pin n على 0

---

#### ### `thunderx_gpio_set_multiple`

```c
static int thunderx_gpio_set_multiple(struct gpio_chip *chip,
                                      unsigned long *mask,
                                      unsigned long *bits)
```

بتكتب أكتر من pin في نفس الوقت بكفاءة. لكل bank (64 pin):
- `set_bits = bits[bank] & mask[bank]` → الـ pins اللي هتتضبط على 1
- `clear_bits = ~bits[bank] & mask[bank]` → الـ pins اللي هتتضبط على 0

بعدين writeq لـ `TX_SET` و `TX_CLR` في نفس الـ bank. مفيش locking هنا لأن الـ set/clear registers أصلاً atomic من الـ hardware.

---

#### ### `thunderx_gpio_set_config`

```c
static int thunderx_gpio_set_config(struct gpio_chip *chip,
                                    unsigned int line,
                                    unsigned long cfg)
```

بتدعم 3 configs من الـ `pinconf` API:

| Config | الدور | ملاحظة |
|---|---|---|
| `PIN_CONFIG_DRIVE_OPEN_DRAIN` | ضبط open-drain | بيفعّل `TX_OD` و `PIN_XOR` (hardware quirk: open-drain بيعكس الـ signal!) |
| `PIN_CONFIG_DRIVE_PUSH_PULL` | ضبط push-pull | بيمسح `TX_OD` و `PIN_XOR` |
| `PIN_CONFIG_INPUT_DEBOUNCE` | ضبط glitch filter | الـ argument بالـ microseconds، max 1228µs |

**الـ Debounce calculation:**
```c
arg *= 400;  /* تحويل لـ 2.5ns clocks */
sel = 0;
while (arg > 15) {
    sel++;
    arg++;      /* round up دايماً */
    arg >>= 1;  /* تقليص بعامل 2 */
}
/* النتيجة: filter_time = arg * 2^sel * 2.5ns */
```

**Key detail:** لو الـ pin كان output وتغيرت الـ invert/od settings، بتُعيد استدعاء `thunderx_gpio_dir_out` لتطبيق الإعدادات الجديدة فوراً.

---

### فئة 5: IRQ Operations

#### ### `thunderx_gpio_irq_ack`

```c
static void thunderx_gpio_irq_ack(struct irq_data *d)
```

بتكتب `GPIO_INTR_INTR` (bit 0) في الـ `GPIO_INTR` register. هذا الـ bit هو **W1C (Write-1-to-Clear)** — يعني الكتابة بـ 1 بتمسح الـ pending interrupt flag. بتُستدعى من الـ irq handler بعد معالجة الـ interrupt.

---

#### ### `thunderx_gpio_irq_mask` / `thunderx_gpio_irq_unmask`

```c
static void thunderx_gpio_irq_mask(struct irq_data *d)
static void thunderx_gpio_irq_unmask(struct irq_data *d)
```

**الـ mask:** بتكتب `GPIO_INTR_ENA_W1C` (bit 2) — **W1C** على الـ enable bit → disable.
**الـ unmask:** بتكتب `GPIO_INTR_ENA_W1S` (bit 3) — **W1S (Write-1-to-Set)** على الـ enable bit → enable.

الـ hardware بيدعم W1C وW1S للـ atomic enable/disable بدون read-modify-write.

---

#### ### `thunderx_gpio_irq_mask_ack`

```c
static void thunderx_gpio_irq_mask_ack(struct irq_data *d)
```

بتعمل mask وack في كتابة واحدة: `GPIO_INTR_ENA_W1C | GPIO_INTR_INTR`. ده optimization للـ level-triggered interrupts اللي محتاجة mask قبل ما تتمسح.

---

#### ### `thunderx_gpio_irq_set_type`

```c
static int thunderx_gpio_irq_set_type(struct irq_data *d,
                                      unsigned int flow_type)
```

الـ function الأهم في الـ IRQ subsection. بتضبط سلوك الـ interrupt بالكامل:

| flow_type | الـ INT_TYPE bit | الـ handler | الـ PIN_XOR |
|---|---|---|---|
| `EDGE_RISING` | 1 (edge) | `handle_fasteoi_ack_irq` | 0 |
| `EDGE_FALLING` | 1 (edge) | `handle_fasteoi_ack_irq` | 1 (invert) |
| `EDGE_BOTH` | 1 (edge) | `handle_fasteoi_ack_irq` | 0 |
| `LEVEL_HIGH` | 0 (level) | `handle_fasteoi_mask_irq` | 0 |
| `LEVEL_LOW` | 0 (level) | `handle_fasteoi_mask_irq` | 1 (invert) |

**Key details:**
- لـ edge: `handle_fasteoi_ack_irq` — بتعمل ack بعد الـ handler.
- لـ level: `handle_fasteoi_mask_irq` — بتعمل mask أثناء الـ handling وunmask بعده.
- الـ `PIN_XOR` بيُستخدم لـ emulate الـ falling/low detection لأن الـ hardware أصلاً بيكتشف rising/high.
- الـ `IRQCHIP_SET_TYPE_MASKED` flag بيضمن إن الـ interrupt يكون masked أثناء تغيير الـ type.

---

#### ### `thunderx_gpio_irq_enable` / `thunderx_gpio_irq_disable`

```c
static void thunderx_gpio_irq_enable(struct irq_data *d)
static void thunderx_gpio_irq_disable(struct irq_data *d)
```

هذين بيعملوا **full enable/disable** مع الـ parent MSI-X:

**Enable flow:**
```
gpiochip_enable_irq(gc, line)   → activate irq resources
irq_chip_enable_parent(d)       → enable MSI-X vector في الـ parent domain
thunderx_gpio_irq_unmask(d)     → W1S على GPIO_INTR_ENA
```

**Disable flow:**
```
thunderx_gpio_irq_mask(d)       → W1C على GPIO_INTR_ENA
irq_chip_disable_parent(d)      → disable MSI-X في الـ parent domain
gpiochip_disable_irq(gc, line)  → release irq resources
```

الترتيب مهم: في الـ enable نفعّل من الـ outside-in، وفي الـ disable من الـ inside-out.

---

### فئة 6: IRQ Hierarchy Helpers

#### ### `thunderx_gpio_child_to_parent_hwirq`

```c
static int thunderx_gpio_child_to_parent_hwirq(struct gpio_chip *gc,
                                               unsigned int child,
                                               unsigned int child_type,
                                               unsigned int *parent,
                                               unsigned int *parent_type)
```

بتعمل translation من GPIO line number للـ MSI-X hardware IRQ number في الـ parent domain. بتجيب الـ `irq_data` للـ MSI-X vector المقابل وبتكشف الـ hwirq بتاعه.

- الـ `parent_type` دايماً `IRQ_TYPE_LEVEL_HIGH` لأن الـ MSI-X interrupts level-triggered من وجهة نظر الـ parent.
- بتُستدعى أثناء الـ `irq_domain_push_irq` في الـ probe.

---

#### ### `thunderx_gpio_populate_parent_alloc_info`

```c
static int thunderx_gpio_populate_parent_alloc_info(struct gpio_chip *chip,
                                                    union gpio_irq_fwspec *gfwspec,
                                                    unsigned int parent_hwirq,
                                                    unsigned int parent_type)
```

بتملي الـ `msi_alloc_info_t` بالـ `hwirq` للـ parent domain. ده مطلوب لأن الـ parent domain هنا هو **MSI domain** مش standard IRQ domain، وبيحتاج `msi_alloc_info_t` مش `irq_fwspec`.

---

### فئة 7: Helper Functions

#### ### `bit_cfg_reg` / `intr_reg`

```c
static unsigned int bit_cfg_reg(unsigned int line)
{
    return 8 * line + GPIO_BIT_CFG;  /* 0x400 + line*8 */
}

static unsigned int intr_reg(unsigned int line)
{
    return 8 * line + GPIO_INTR;     /* 0x800 + line*8 */
}
```

كل GPIO line عندها register خاص بيها بـ 8-byte stride. مع أكتر من 64 line، الـ line 64+ بتروح للـ `GPIO_2ND_BANK` (offset `0x1400`).

---

#### ### `thunderx_gpio_is_gpio` / `thunderx_gpio_is_gpio_nowarn`

```c
static bool thunderx_gpio_is_gpio(struct thunderx_gpio *txgpio, unsigned int line)
static bool thunderx_gpio_is_gpio_nowarn(struct thunderx_gpio *txgpio, unsigned int line)
```

بيتحققوا إن `GPIO_BIT_CFG_PIN_SEL_MASK` (bits 25:16) == 0. لو non-zero، يبقى الـ pin مضبوط لـ alternate function (مثلاً TWSI أو UART) ومش مستخدم كـ GPIO.

- `_nowarn` نسخة صامتة — بتُستدعى من `get_direction` عشان تتجنب الـ WARN أثناء الـ probe scan.
- `is_gpio` (مع warn) بتُستدعى من كل request/direction function.

---

### الـ Register Map الخاص بالـ Driver

```
Offset      Register          الوظيفة
─────────────────────────────────────────────────
0x000       GPIO_RX_DAT       قراءة حالة الـ pins (input)
0x008       GPIO_TX_SET       W1S: تضبط pins على 1
0x010       GPIO_TX_CLR       W1C: تضبط pins على 0
0x090       GPIO_CONST        عدد الـ GPIO lines والـ MSI base
0x400+n*8   GPIO_BIT_CFG[n]   config لكل pin (TX_OE, XOR, filter, etc)
0x800+n*8   GPIO_INTR[n]      interrupt control لكل pin
0x1400      GPIO_2ND_BANK     بداية الـ bank الثاني (pins 64-127)
```

---

### العلاقة بين الـ DT Binding والـ Driver

```
DT Node (gpio_6_0)
    │
    ├── reg = <0x3000 0 0 0 0>
    │         └── PCI BDF → kernel يبعت PCI scan
    │                           └── match PCI_DEVICE(0x177D, 0xA00A)
    │                                   └── thunderx_gpio_probe()
    │
    ├── gpio-controller + #gpio-cells = <2>
    │         └── consumers يقدروا يعملوا: <&gpio_6_0 PIN FLAGS>
    │
    └── interrupt-controller + #interrupt-cells = <2>
              └── consumers يقدروا يعملوا: <&gpio_6_0 PIN TRIGGER_TYPE>
                  كل pin → MSI-X vector مستقل → chained إلى الـ parent domain
```
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. مداخل الـ debugfs المتعلقة بـ GPIO

الـ GPIO subsystem بيعرض معلومات مفيدة جداً عبر الـ debugfs. بالنسبة لـ ThunderX/OCTEON-TX اللي بيتعرف كـ PCI device، في مداخل إضافية على مستوى الـ PCI.

```bash
# Mount debugfs لو مش موجود
mount -t debugfs none /sys/kernel/debug

# إظهار كل الـ GPIO controllers المسجلة في الـ kernel
cat /sys/kernel/debug/gpio

# الـ output المتوقع — كل GPIO chip بيظهر مع pins
# gpiochip0: GPIOs 0-47, parent: pci 0000:06:00.0, thunder-gpio, can sleep:
#  gpio-0  (                    ) out lo
#  gpio-5  (reset               ) out hi ACTIVE LOW
#  gpio-12 (                    ) in  lo IRQ
```

الـ bus address في الـ DT مثال `<0x3000 0 0 0 0>` بتوافق DEVFN=0x30 أي Bus=0, Device=6, Function=0، يعني الـ PCI address هيكون `0000:06:00.0`.

```bash
# تفاصيل الـ GPIO chip المرتبطة بـ PCI device معين
ls /sys/kernel/debug/pinctrl/
cat /sys/kernel/debug/pinctrl/pinctrl-maps

# إظهار الـ gpio_chip المرتبطة بـ PCI device
cat /sys/kernel/debug/gpio | grep -A 60 "06:00.0"
```

#### 2. مداخل الـ sysfs المهمة

```bash
# المسار الأساسي للـ GPIO exported pins
ls /sys/class/gpio/

# export pin للاختبار اليدوي (مثلاً pin 5 من chip offset 0)
echo <absolute_gpio_number> > /sys/class/gpio/export

# قراءة/كتابة القيمة
cat /sys/class/gpio/gpio<N>/value
echo 1 > /sys/class/gpio/gpio<N>/value

# اتجاه الـ pin
cat /sys/class/gpio/gpio<N>/direction
echo "out" > /sys/class/gpio/gpio<N>/direction

# معلومات الـ PCI device المرتبطة بالـ controller
ls /sys/bus/pci/devices/0000:06:00.0/
cat /sys/bus/pci/devices/0000:06:00.0/vendor   # يجب أن يكون 0x177d (Cavium)
cat /sys/bus/pci/devices/0000:06:00.0/device   # Device ID
cat /sys/bus/pci/devices/0000:06:00.0/class    # يجب أن يكون 0x0c8000

# الـ resource الخاص بالـ BAR (Base Address Register)
cat /sys/bus/pci/devices/0000:06:00.0/resource

# الـ IRQ المعين لكل GPIO
cat /sys/kernel/irq/*/actions | grep gpio
ls /sys/kernel/irq/
```

#### 3. استخدام الـ ftrace

```bash
# تفعيل الـ tracing لأحداث الـ GPIO
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable

# أو تفعيل events محددة
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_direction/enable
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_value/enable

# مشاهدة الـ trace في real-time
cat /sys/kernel/debug/tracing/trace_pipe

# تفعيل tracing للـ IRQ اللي بتيجي من GPIO
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_entry/enable
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_exit/enable

# تصفية الـ events لـ GPIO IRQ handler بس
echo 'name == "thunder_gpio_irq_handler"' > \
    /sys/kernel/debug/tracing/events/irq/irq_handler_entry/filter

# تفعيل function tracer على driver معين
echo thunder_gpio_* > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace

# إيقاف الـ tracing
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo nop > /sys/kernel/debug/tracing/current_tracer
```

#### 4. تفعيل الـ printk والـ Dynamic Debug

```bash
# تفعيل dynamic debug لكل الـ GPIO subsystem
echo "module gpio_thunderx +p" > /sys/kernel/debug/dynamic_debug/control

# أو تفعيل ملف بعينه
echo "file drivers/gpio/gpio-thunderx.c +p" > \
    /sys/kernel/debug/dynamic_debug/control

# تفعيل مع الـ line numbers والـ function names
echo "file drivers/gpio/gpio-thunderx.c +pflmt" > \
    /sys/kernel/debug/dynamic_debug/control

# تفعيل debug للـ PCI probe
echo "file drivers/pci/probe.c +p" > /sys/kernel/debug/dynamic_debug/control

# رفع مستوى الـ kernel log لرؤية كل الـ messages
dmesg -n 8
# أو
echo 8 > /proc/sys/kernel/printk

# مشاهدة logs مباشرة
dmesg -w | grep -i "thunder\|gpio\|cavium"
```

#### 5. خيارات الـ Kernel Config للـ Debugging

| CONFIG Option | الوصف | متى تستخدمها |
|---|---|---|
| `CONFIG_DEBUG_GPIO` | يفعّل الـ extra validation وlogs في GPIO core | دايماً في بيئة الـ dev |
| `CONFIG_GPIO_CDEV_V1` | يفعّل الـ character device interface v1 | اختبار الـ userspace tools |
| `CONFIG_GPIOLIB_IRQCHIP` | دعم الـ IRQ من خلال GPIO | لازم مفعّل لـ ThunderX |
| `CONFIG_GPIO_SYSFS` | يعرض الـ GPIOs عبر sysfs | اختبار يدوي |
| `CONFIG_PROVE_LOCKING` | كشف deadlocks في GPIO locks | debugging الـ IRQ handlers |
| `CONFIG_DEBUG_SHIRQ` | اختبار الـ shared IRQs | لو GPIO IRQ بيتشارك |
| `CONFIG_LOCK_STAT` | إحصائيات الـ spinlocks | كشف الـ contention |
| `CONFIG_PCI_DEBUG` | تفاصيل PCI probe وconfig space | مشاكل الـ PCI discovery |
| `CONFIG_DYNAMIC_DEBUG` | تفعيل pr_debug ديناميكياً | لازم لـ dynamic debug |
| `CONFIG_KASAN` | كشف memory errors | crash غريب في الـ driver |
| `CONFIG_KCSAN` | كشف data races | مشاكل الـ concurrency |

```bash
# التحقق من الـ config الحالية
zcat /proc/config.gz | grep -E "CONFIG_DEBUG_GPIO|CONFIG_GPIO|CONFIG_PCI_DEBUG"
# أو
cat /boot/config-$(uname -r) | grep -E "CONFIG_DEBUG_GPIO|CONFIG_GPIO"
```

#### 6. أدوات الـ devlink والـ subsystem-specific Tools

```bash
# فحص الـ PCI device الخاص بـ ThunderX GPIO
lspci -vvv -s 0000:06:00.0

# الـ output المتوقع يحتوي:
# Vendor: Cavium (177d)
# Class: System peripheral
# BAR0: Memory register space
# Capabilities: MSI-X — مهم للـ GPIO interrupts

# عدد الـ MSI-X vectors المتاحة (كل GPIO pin ممكن يكون له vector مستقل)
cat /sys/bus/pci/devices/0000:06:00.0/msix_cap

# الـ interrupts المعينة للـ device
cat /proc/interrupts | grep "0000:06:00.0\|thunder"

# أدوات userspace للـ GPIO (من حزمة libgpiod)
gpiodetect                          # رصد كل الـ GPIO chips
gpioinfo gpiochip0                  # معلومات تفصيلية عن chip معينة
gpioget gpiochip0 5                 # قراءة pin 5
gpioset gpiochip0 3=1               # كتابة pin 3
gpiomon gpiochip0 5                 # مراقبة تغيرات pin 5

# فحص الـ IRQ affinity للـ GPIO interrupts
for irq in $(cat /proc/interrupts | grep thunder | awk '{print $1}' | tr -d ':'); do
    echo "IRQ $irq affinity: $(cat /proc/irq/$irq/smp_affinity_list)"
done
```

#### 7. جدول رسائل الـ Error الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `thunder_gpio: probe of 0000:06:00.0 failed with error -ENOMEM` | فشل تخصيص الذاكرة أثناء الـ probe | تحقق من الذاكرة المتاحة، راجع `dmesg | grep -i "oom\|memory"` |
| `thunder_gpio: Unable to request region` | الـ BAR region مش متاحة أو محجوزة | تحقق من `cat /proc/iomem | grep "06:00"` وتأكد مافيش تعارض |
| `genirq: Flags mismatch irq <N>` | تعارض في flags الـ IRQ بين الـ GPIO pin والـ consumer | راجع `IRQF_SHARED` flags في الـ driver |
| `gpio-thunderx: failed to set irq type` | محاولة ضبط trigger type غير مدعوم | تحقق من الـ trigger type المطلوب، ThunderX يدعم edge وlevel |
| `Unable to handle kernel paging request` في `thunder_gpio` | الـ MMIO base address غلط أو unmapped | تحقق من الـ `ioremap` وصحة الـ BAR address |
| `gpio gpiochip0: (thunder-gpio): gpiochip_add_data() failed` | فشل تسجيل الـ GPIO chip | عادةً بسبب تعارض في GPIO range |
| `PCI: Cannot allocate resource region` | مشكلة في الـ PCI resource allocation | تحقق من BIOS/ACPI settings والـ resource limits |
| `thunder_gpio irq_set_affinity failed` | فشل ضبط CPU affinity للـ IRQ | تحقق من الـ NUMA topology وتوافق الـ CPU mask |
| `devm_ioremap_resource failed` | فشل mapping الـ register space | تحقق من حجم الـ BAR وصحة الـ resource في DT |
| `No GPIO chip found for this PCI device` | الـ PCI ID مش متطابق مع الـ driver | تحقق من `PCI_DEVICE` table في الـ driver |

#### 8. نقاط استراتيجية لـ dump_stack() وWARN_ON()

```c
/* في دالة الـ probe — التحقق من الـ BAR mapping */
static int thunder_gpio_probe(struct pci_dev *pdev, ...)
{
    void __iomem *base = pcim_iomap(pdev, 0, 0);
    WARN_ON(!base);  /* لو base == NULL، في مشكلة في الـ PCI BAR */

    /* التحقق من صحة عدد الـ GPIOs */
    WARN_ON(ngpios == 0 || ngpios > 64);
}

/* في الـ IRQ handler — التحقق من الـ context */
static irqreturn_t thunder_gpio_irq_handler(int irq, void *dev_id)
{
    /* يجب أن نكون في interrupt context */
    WARN_ON(!in_interrupt());

    /* التحقق من صحة الـ pending register */
    u64 pending = readq(base + GPIO_INTR_OFFSET);
    if (unlikely(!pending)) {
        /* spurious interrupt */
        WARN_ONCE(1, "thunder_gpio: spurious IRQ %d\n", irq);
        return IRQ_NONE;
    }
}

/* في دالة الـ direction_output — التحقق من صحة الـ GPIO number */
static int thunder_gpio_direction_output(struct gpio_chip *chip,
                                          unsigned int offset, int value)
{
    WARN_ON(offset >= chip->ngpio);
    /* dump_stack في حالة critical error */
    if (offset >= chip->ngpio) {
        dump_stack();
        return -EINVAL;
    }
}

/* في دالة الـ irq_set_type — التحقق من الـ type المدعوم */
static int thunder_gpio_irq_set_type(struct irq_data *d, unsigned int type)
{
    WARN_ON(type == IRQ_TYPE_NONE);
}
```

---

### Hardware Level

#### 1. التحقق من توافق حالة الـ Hardware مع الـ Kernel

```bash
# قراءة حالة الـ GPIO registers مباشرة عبر الـ PCI BAR
# أولاً: نحدد الـ physical address للـ BAR0
cat /sys/bus/pci/devices/0000:06:00.0/resource
# الـ output: start_addr  end_addr  flags
# مثال: 0x0000801000000000  0x00008010000fffff  0x0000000000040200

# مقارنة القيمة المقروءة من الـ register مع ما يعرضه الـ kernel
cat /sys/kernel/debug/gpio  # الـ kernel view
# ثم قراءة الـ register مباشرة (الخطوات في قسم الـ register dump)
```

#### 2. تقنيات الـ Register Dump

```bash
# استخدام devmem2 لقراءة الـ GPIO registers مباشرة
# (يتطلب تثبيت devmem2 وصلاحيات root)

# تحديد الـ BAR0 address
BAR0=$(cat /sys/bus/pci/devices/0000:06:00.0/resource | \
       awk 'NR==1{printf "0x%s", $1}')

echo "ThunderX GPIO BAR0 base: $BAR0"

# قراءة GPIO_BIT_CFG register (offset 0x400 + pin*8 لكل pin)
# Pin 0 config register
devmem2 $((BAR0 + 0x400)) q

# GPIO_RX_DAT — حالة الـ input pins
devmem2 $((BAR0 + 0x0)) q

# GPIO_TX_SET — set output pins
devmem2 $((BAR0 + 0x8)) q

# GPIO_TX_CLR — clear output pins
devmem2 $((BAR0 + 0x10)) q

# GPIO_INTR — interrupt pending register
devmem2 $((BAR0 + 0x200)) q

# بديل باستخدام /dev/mem مباشرة (أقل أمان)
dd if=/dev/mem bs=8 count=1 skip=$((BAR0 / 8)) 2>/dev/null | xxd

# استخدام io utility لقراءة MMIO
io -4 $((BAR0 + 0x400))
```

**تفسير قيم الـ GPIO_BIT_CFG Register:**

```
Bit [0]    : TX_OE — 1 = output enable
Bit [1]    : RX_DAT — قيمة الـ input
Bit [2]    : TX_DAT — قيمة الـ output
Bit [11:8] : INT_TYPE — 0=level_low, 1=level_high, 2=edge_fall, 3=edge_rise, 4=edge_both
Bit [12]   : INT_EN — تفعيل الـ interrupt
Bit [13]   : FILTR_EN — تفعيل فلترة الـ noise
```

#### 3. نصائح الـ Logic Analyzer والـ Oscilloscope

**نقاط القياس:**
- اختبار continuity بين الـ GPIO pin على الـ SoC وأقرب test point على الـ board
- قياس الـ voltage levels: ThunderX GPIO تعمل عادةً على 1.8V أو 3.3V حسب الـ bank

```
ThunderX GPIO Timing Specs:
┌─────────────────────────────────────┐
│  Output transition time: < 5ns      │
│  Input setup time: > 2ns            │
│  Input hold time: > 1ns             │
│  Max toggle frequency: ~100MHz      │
└─────────────────────────────────────┘
```

**إعداد الـ Logic Analyzer:**
```
- Sample rate: على الأقل 10x أعلى تردد متوقع
- Trigger: على الـ edge الأولى من الـ GPIO signal
- Capture length: كافية لتغطي عدة interrupt cycles
- Protocol decoder: لو GPIO متصل بـ I2C/SPI فعّل الـ decoder المناسب
```

**استخدام الـ kernel كـ logic analyzer بسيط:**
```bash
# تسجيل توقيت تغيرات الـ GPIO باستخدام gpiomon
gpiomon --num-events=100 --format="%e %o %s %n" gpiochip0 5 > /tmp/gpio_log.txt

# تحليل الـ timing
awk '{if(NR>1) print $3-prev, $0; prev=$3}' /tmp/gpio_log.txt
```

#### 4. مشاكل الـ Hardware الشائعة وأنماط الـ Kernel Log

| المشكلة | نمط الـ Kernel Log | طريقة التشخيص |
|---|---|---|
| الـ GPIO pin مش بيستجيب | لا توجد interrupts في `cat /proc/interrupts` لهذا الـ pin | قس الـ voltage على الـ pin بالـ oscilloscope |
| noise على الـ GPIO input | `thunder_gpio: spurious IRQ` متكرر | فعّل الـ FILTR_EN في GPIO_BIT_CFG، أضف capacitor للـ hardware |
| الـ interrupt مش بيصحى | IRQ counter في `/proc/interrupts` ثابت | تحقق من MSI-X enable والـ INT_EN bit في GPIO_BIT_CFG |
| voltage mismatch | الـ kernel يرى "high" والـ hardware "low" | تحقق من الـ ACTIVE_LOW flag وقيمة VDDIO |
| تعارض PCI resources | `pci 0000:06:00.0: BAR 0: no space for` | راجع الـ BIOS memory map وحجم الـ BAR المطلوب |
| الـ GPIO يتقلب بدون سبب | قيمة متغيرة في `gpioinfo` | floating input — أضف pull-up/pull-down resistor |

#### 5. Debugging الـ Device Tree

```bash
# التحقق من الـ DT المُحمَّل فعلياً في الـ kernel
# لكن ThunderX بيستخدم PCI binding مش DT مباشرة
# الـ DT node الاختياري كما في المثال:
# gpio@6,0 { compatible = "cavium,thunder-8890-gpio"; reg = <0x3000 0 0 0 0>; }

# فحص الـ DT المُحمَّل
ls /sys/firmware/devicetree/base/
# البحث عن نود الـ GPIO
find /sys/firmware/devicetree/base/ -name "compatible" \
    -exec sh -c 'val=$(cat "{}"); echo "$val" | grep -q "cavium" && echo "Found: {}"' \;

# تحويل الـ DT المُحمَّل لـ dts قابل للقراءة
dtc -I fs /sys/firmware/devicetree/base/ 2>/dev/null | \
    grep -A 20 "thunder-8890-gpio"

# التحقق من صحة الـ reg property
# reg = <0x3000 0 0 0 0> يعني DEVFN=0x30
python3 -c "devfn=0x30; print(f'Device: {devfn>>3}, Function: {devfn&7}')"
# Output: Device: 6, Function: 0  →  يوافق 0000:06:00.0

# التحقق من الـ PCI ID في الـ driver table
grep -r "177d" /sys/bus/pci/devices/*/vendor
# يجب أن يعطي 0x177d لـ Cavium devices

# فحص الـ interrupt-map في DT لو موجود
find /sys/firmware/devicetree/base/ -name "interrupt-map" | \
    xargs -I{} sh -c 'echo "=== {} ==="; hexdump -C "{}"'

# مقارنة الـ #gpio-cells في DT مع ما يتوقعه الـ consumer
# يجب أن يكون 2: [pin_number, flags]
find /sys/firmware/devicetree/base/ -name "#gpio-cells" | \
    xargs -I{} sh -c 'val=$(hexdump -e "1/4 \"%d\"" "{}"); echo "{}: $val"'
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للنسخ

```bash
#!/bin/bash
# thunder_gpio_debug.sh — سكريبت شامل لـ debugging الـ ThunderX GPIO

PCI_ADDR="0000:06:00.0"
GPIOCHIP="gpiochip0"

echo "=== 1. PCI Device Info ==="
lspci -vvv -s $PCI_ADDR 2>/dev/null | grep -E "Vendor|Device|Class|BAR|IRQ|MSI"

echo "=== 2. GPIO Chip Info ==="
cat /sys/kernel/debug/gpio | grep -A 60 "$PCI_ADDR"

echo "=== 3. GPIO Lines State ==="
gpioinfo $GPIOCHIP 2>/dev/null

echo "=== 4. Interrupts ==="
cat /proc/interrupts | grep -E "thunder|gpio|$PCI_ADDR"

echo "=== 5. PCI Resources ==="
cat /sys/bus/pci/devices/$PCI_ADDR/resource

echo "=== 6. Driver Binding ==="
ls -la /sys/bus/pci/devices/$PCI_ADDR/driver

echo "=== 7. Recent dmesg ==="
dmesg | grep -i -E "thunder|gpio|cavium" | tail -30

echo "=== 8. IRQ Affinity ==="
for irq in $(cat /proc/interrupts | grep -i thunder | awk '{print $1}' | tr -d ':'); do
    echo "  IRQ $irq: $(cat /proc/irq/$irq/smp_affinity_list 2>/dev/null)"
done

echo "=== 9. MSI-X Vectors ==="
cat /sys/bus/pci/devices/$PCI_ADDR/msi_irqs/* 2>/dev/null

echo "=== 10. Power State ==="
cat /sys/bus/pci/devices/$PCI_ADDR/power_state
```

#### مثال على الـ Output وتفسيره

```bash
# أمر:
cat /sys/kernel/debug/gpio

# الـ output:
# gpiochip0: GPIOs 440-487, parent: pci 0000:06:00.0, thunder-gpio, can sleep:
#  gpio-440 (                    ) in  lo
#  gpio-445 (reset-button        ) in  hi ACTIVE LOW
#  gpio-450 (led-status          ) out lo
#  gpio-455 (                    ) in  lo IRQ

# التفسير:
# - GPIOs 440-487: النطاق المطلق في الـ kernel GPIO namespace
# - can sleep: الـ driver يمكن أن يستخدم sleeping functions (PCI MMIO)
# - "in lo": input، القيمة الحالية low
# - "ACTIVE LOW": القيمة المنطقية معكوسة
# - IRQ: الـ pin مسجل كـ interrupt source

# أمر:
cat /proc/interrupts | grep thunder

# الـ output:
# 45:     0     0   1024      0   PCI-MSI 524288-edge   thunder-gpio-0
# 46:     0     0      0      2   PCI-MSI 524289-edge   thunder-gpio-1
# 55:     0     0      0      0   PCI-MSI 524298-edge   thunder-gpio-10

# التفسير:
# - العمود الأول: رقم الـ IRQ
# - الأعمدة التالية: عدد مرات الـ interrupt لكل CPU
# - PCI-MSI: يؤكد استخدام MSI-X (كل GPIO له vector مستقل)
# - thunder-gpio-N: اسم الـ handler مع رقم الـ pin

# أمر:
lspci -s 0000:06:00.0 -v

# الـ output:
# 06:00.0 System peripheral: Cavium, Inc. CN88XX GPIO (rev 01)
#     Subsystem: Cavium, Inc. CN88XX GPIO
#     Flags: bus master, fast devsel, latency 0, IRQ 255, NUMA node 0
#     Memory at 801000000000 (64-bit, prefetchable) [size=1M]
#     Capabilities: [70] MSI-X: Enable+ Count=64 Masked-
#     Kernel driver in use: thunder_gpio

# التفسير:
# - Count=64: 64 MSI-X vector = 64 GPIO pin كل منه interrupt مستقل
# - Memory at 801000000000: الـ BAR0 physical address للـ register access
# - Kernel driver in use: thunder_gpio — الـ driver محمّل بنجاح
```

```bash
# تفعيل ftrace وتشغيل اختبار GPIO
echo function > /sys/kernel/debug/tracing/current_tracer
echo "thunder_gpio_*" > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on

# تشغيل اختبار
gpioset gpiochip0 5=1
gpioset gpiochip0 5=0

echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace

# الـ output المتوقع:
# gpioset-1234  [002] ....  123.456789: thunder_gpio_direction_output <-gpiod_direction_output
# gpioset-1234  [002] ....  123.456791: thunder_gpio_set <-gpiod_set_value
# gpioset-1234  [002] ....  123.456820: thunder_gpio_set <-gpiod_set_value

# التفسير: يُظهر سلسلة استدعاء الـ functions مع الـ timestamp
# الفرق بين الـ timestamps يكشف أي function تأخذ وقت أطول
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: SmartNIC بيـ reset مش شغال صح

#### العنوان
**الـ GPIO reset line على Cavium ThunderX CN88XX في SmartNIC مش بتشتغل**

#### السياق
شركة بتبني SmartNIC على أساس **Cavium ThunderX CN88XX** — الكارت بيتركب في سيرفر data center عشان يعمل offload للـ networking. الـ SmartNIC فيها QSFP+ module محتاج GPIO line عشان يعمل reset لما يحصل link failure أو firmware update.

الـ Device Tree بتاع الـ SmartNIC موجود في الـ PCI subsystem، والـ GPIO controller بتاعه بيتعرف عن طريق PCI enumeration مش عن طريق DT node تقليدي.

#### المشكلة
الـ engineer بيعمل `gpiod_set_value(reset_gpio, 1)` في الـ driver بتاع الـ QSFP module — بس الـ QSFP مش بيـ reset. الـ dmesg ملوش errors، بس الـ hardware مش بيستجاوبش.

#### التحليل
الـ binding بيوضح إن الـ `compatible` string هي **optional** لأن الـ PCI driver binding هو اللي بيتحكم:

```
- compatible: "cavium,thunder-8890-gpio", unused as PCI driver binding is used.
```

يعني الـ GPIO controller بيتعرف من خلال الـ PCI Device/Function number — مش من الـ `compatible`. لو الـ engineer كتب الـ DT node بشكل غلط في الـ `reg` property، الـ GPIO controller مش هيتـ map صح.

الـ `reg` في المثال:
```c
reg = <0x3000 0 0 0 0>; /* DEVFN = 0x30 (6:0) */
```

الـ `0x3000` ده بيمثل الـ PCI DEVFN — الـ device number 6، function 0. لو المهندس كتب DEVFN غلط، الـ kernel هيـ bind الـ GPIO controller على controller تاني تماماً.

خطوات التحليل:
```bash
# اتأكد إن الـ GPIO controller اتـ enumerate صح
ls /sys/bus/pci/devices/ | grep -i cavium

# شوف الـ GPIO chip اتسجل
cat /sys/kernel/debug/gpio

# اتأكد من DEVFN في DT
grep -r "gpio@" /proc/device-tree/
```

لو `gpioinfo` بيظهر الـ chip بس الـ line direction مش بتتغير، الـ مشكلة في الـ `reg` value.

#### الحل
```dts
/* غلط — DEVFN مش صح */
gpio_bad: gpio@5,0 {
    reg = <0x2800 0 0 0 0>; /* DEVFN = 0x28 (5:0) — wrong device */
    gpio-controller;
    #gpio-cells = <2>;
};

/* صح — DEVFN = 0x30 = device 6, function 0 */
gpio_6_0: gpio@6,0 {
    reg = <0x3000 0 0 0 0>;
    gpio-controller;
    #gpio-cells = <2>;
};
```

بعد تصحيح الـ DT:
```bash
# verify الـ PCI address صح
lspci -nn | grep -i gpio
# أو
cat /proc/device-tree/pci*/gpio*/reg | xxd
```

#### الدرس المستفاد
على ThunderX، الـ `compatible` string مش هي اللي بتحدد الـ driver binding — الـ **PCI DEVFN** في الـ `reg` property هو الأساس. أي خطأ في الـ DEVFN بيخلي الـ GPIO controller يتـ map على device غلط بدون أي error واضح.

---

### السيناريو 2: Interrupt من GPIO مش بيوصل للـ kernel

#### العنوان
**الـ GPIO interrupt controller على OCTEON TX CN83XX في networking appliance مش بيـ trigger**

#### السياق
شركة بتبني **networking appliance** باستخدام **OCTEON TX CN83XX** — الجهاز بيشتغل كـ hardware firewall مع line rate packet processing. فيه push button على الـ front panel بيـ trigger GPIO interrupt عشان يعمل factory reset.

#### المشكلة
الـ factory reset button مش بيشتغل. الـ driver بتاعه بيعمل `request_irq()` بدون error، بس لما المستخدم يضغط الـ button، مفيش interrupt بيوصل.

#### التحليل
الـ binding بيوضح إن `interrupt-controller` و `#interrupt-cells` properties هما **optional** — بس لو عايز تستخدم الـ GPIO كـ interrupt source، لازم يكونوا موجودين:

```
- interrupt-controller: Marks the device node as an interrupt controller.
- #interrupt-cells: Must be present and have value of 2 if
                    "interrupt-controller" is present.
  - First cell is the GPIO pin number relative to the controller.
  - Second cell is triggering flags as defined in interrupts.txt.
```

الـ engineer نسي يضيف `interrupt-controller` في الـ DT node:

```dts
/* ناقص interrupt-controller */
gpio_6_0: gpio@6,0 {
    reg = <0x3000 0 0 0 0>;
    gpio-controller;
    #gpio-cells = <2>;
    /* interrupt-controller; — متعملتش */
    /* #interrupt-cells = <2>; — متعملتش */
};
```

لما الـ button driver بيعمل:
```c
irq = gpiod_to_irq(button_gpio);
request_irq(irq, button_handler, IRQF_TRIGGER_FALLING, "factory_reset", dev);
```

الـ `gpiod_to_irq()` بيرجع `-ENXIO` أو `-ENOTSUPP` لأن الـ GPIO controller مش معلن كـ interrupt controller في الـ DT.

```bash
# تأكد من الـ error
dmesg | grep -i "gpio\|irq\|interrupt"
# هيظهر حاجة زي:
# gpio-thunderx: GPIO 15 cannot be used as interrupt
```

#### الحل
```dts
gpio_6_0: gpio@6,0 {
    compatible = "cavium,thunder-8890-gpio";
    reg = <0x3000 0 0 0 0>;
    gpio-controller;
    #gpio-cells = <2>;
    /* إضافة الـ interrupt controller properties */
    interrupt-controller;
    #interrupt-cells = <2>;
};

/* في الـ button node */
factory_reset_btn: button {
    gpios = <&gpio_6_0 15 GPIO_ACTIVE_LOW>;
    interrupts = <15 IRQ_TYPE_EDGE_FALLING>;
    interrupt-parent = <&gpio_6_0>;
};
```

#### الدرس المستفاد
على ThunderX/OCTEON TX، الـ GPIO controller قادر يشتغل كـ interrupt controller — بس ده مش بيحصل تلقائياً. لازم تصرح بـ `interrupt-controller` و `#interrupt-cells = <2>` في الـ DT node صراحةً، وإلا الـ `gpiod_to_irq()` هيفشل بصمت أو برجوع error.

---

### السيناريو 3: تعارض بين GPIO controllers في multi-socket server

#### العنوان
**تعارض GPIO numbering على CN88XX dual-socket server في data center**

#### السياق
**Data center server** بيستخدم **Cavium ThunderX CN88XX** في configuration dual-socket. كل socket عندها PCI hierarchy منفصل وبالتالي عندها GPIO controller منفصل. الـ BMC driver محتاج يتحكم في power LED على GPIO pin 5 — بس مش عارف هو على controller أنهي.

#### المشكلة
الـ BMC driver بيعمل write على GPIO pin 5 — الـ LED مش بتشتغل، بل أحياناً حاجة تانية بتتأثر (fan control pin على socket تانية).

#### التحليل
الـ binding بيحدد إن:
```
- First cell is the GPIO pin number relative to the controller.
```

الكلمة المهمة هي **relative to the controller** — يعني GPIO pin 5 على `gpio_6_0` مختلف عن GPIO pin 5 على `gpio_7_0`. المشكلة إن الـ engineer استخدم absolute GPIO number بدل ما يحدد الـ controller بشكل صريح.

في الـ DT للـ dual-socket:
```dts
/* Socket 0 — PCI domain 0 */
gpio_s0: gpio@6,0 {
    reg = <0x3000 0 0 0 0>;
    gpio-controller;
    #gpio-cells = <2>;
    interrupt-controller;
    #interrupt-cells = <2>;
};

/* Socket 1 — PCI domain 1 */
gpio_s1: gpio@6,0 {
    reg = <0x3000 0 0 0 0>;
    gpio-controller;
    #gpio-cells = <2>;
    interrupt-controller;
    #interrupt-cells = <2>;
};
```

لو الـ BMC driver بيستخدم legacy GPIO number بدل `gpiod_get()` مع phandle:
```c
/* غلط — absolute number مش محدد */
gpio_request(5, "power_led");      /* أنهي controller؟ */

/* صح — محدد بـ DT phandle */
led_gpio = gpiod_get(&pdev->dev, "power-led", GPIOD_OUT_LOW);
```

والـ DT بتاع الـ BMC device:
```dts
bmc {
    /* غلط */
    power-led-gpio = <5>;

    /* صح */
    power-led-gpios = <&gpio_s0 5 GPIO_ACTIVE_HIGH>;
};
```

#### الحل
```bash
# شوف كل الـ GPIO controllers
gpiodetect
# هيظهر:
# gpiochip0 [cavium,thunder-8890-gpio] (64 lines)  -- socket 0
# gpiochip1 [cavium,thunder-8890-gpio] (64 lines)  -- socket 1

# تأكد من اللي بيتتحكم فيه
gpioinfo gpiochip0 | grep -i led
gpioinfo gpiochip1 | grep -i led
```

تصحيح الـ DT consumer node:
```dts
bmc_controller {
    /* صريح — socket 0, pin 5, active high */
    power-led-gpios = <&gpio_s0 5 GPIO_ACTIVE_HIGH>;
    /* fan-ctrl على socket 1 */
    fan-ctrl-gpios = <&gpio_s1 12 GPIO_ACTIVE_HIGH>;
};
```

#### الدرس المستفاد
في multi-socket ThunderX servers، كل socket عندها GPIO controller مستقل. لازم دايماً تحدد الـ controller بشكل صريح عن طريق الـ phandle في الـ DT consumer node — استخدام absolute GPIO numbers بيأدي لتعارضات صعبة الاكتشاف.

---

### السيناريو 4: GPIO في storage controller بيتـ export غلط لـ userspace

#### العنوان
**الـ NVMe drive fault LED مش بيشتغل على OCTEON TX CN81XX storage controller**

#### السياق
شركة بتبني **storage controller** على **OCTEON TX CN81XX** — الجهاز بيتحكم في 24 NVMe drive في chassis. كل drive عندها fault LED مربوطة بـ GPIO pin. الـ userspace daemon (`ledctrl`) بيكتب على `/sys/class/gpio/` عشان يشغل الـ LEDs لما يلاقي drive fault.

#### المشكلة
الـ daemon بيكتب على `/sys/class/gpio/gpio42/value` — مفيش error، بس الـ LED مش بتضوي. الـ engineer يتأكد إن الـ GPIO 42 هو الصح.

#### التحليل
الـ `#gpio-cells = <2>` يعني إن كل GPIO specifier في الـ DT بيتكون من **خليتين**:
```
- First cell is the GPIO pin number relative to the controller.
- Second cell is a standard generic flag bitfield as described in gpio.txt.
```

الـ second cell (flags) بيتضمن active high/low polarity. لو الـ LED مربوطة active-low (بتضوي لما الـ pin = 0) بس الـ DT بيحددها كـ active-high، الـ kernel هيعمل inversion تلقائي — يعني لما الـ daemon يكتب 1 (تشغيل)، الـ kernel يكتب 0 فعلياً على الـ hardware.

بس لو الـ daemon بيستخدم legacy sysfs interface بدل `libgpiod`، ما بيعرفش عن الـ polarity inversion:

```bash
# legacy sysfs — ما بيعرفش الـ polarity
echo 1 > /sys/class/gpio/gpio42/value

# صح — استخدام gpioset مع libgpiod
gpioset gpiochip0 42=1
```

بس المشكلة الأعمق — الـ `gpio-cells = <2>` بيعني إن الـ DT consumer لازم يحدد الـ flags:

```dts
/* في DT الـ LED consumer */
fault-leds {
    /* غلط — ما حددش active polarity */
    led0-gpios = <&gpio_6_0 42>;     /* خلية واحدة بس — error! */

    /* صح */
    led0-gpios = <&gpio_6_0 42 GPIO_ACTIVE_LOW>;
};
```

لو الـ LED consumer حدد `GPIO_ACTIVE_LOW` بشكل غلط كـ `GPIO_ACTIVE_HIGH`، كل write هيكون معكوس.

#### الحل
```bash
# تأكد من الـ direction والـ value الفعلي
cat /sys/kernel/debug/gpio | grep gpio42
# هيظهر: gpio-42 (fault_led0|led0-gpios) out lo

# اتأكد من الـ hardware باستخدام libgpiod
gpioget gpiochip0 42

# شوف الـ DT node
dtc -I fs /sys/firmware/devicetree/base | grep -A5 fault-led
```

تصحيح الـ DT:
```dts
fault_leds: fault-leds {
    compatible = "gpio-leds";
    led@0 {
        label = "nvme0:fault";
        /* NVMe fault LED active-low — LED on when GPIO = 0 */
        gpios = <&gpio_6_0 42 GPIO_ACTIVE_LOW>;
        default-state = "off";
    };
};
```

#### الدرس المستفاد
الـ `#gpio-cells = <2>` على ThunderX بيلزم كل consumer يحدد الـ polarity flags كـ second cell. نسيان الـ flags أو تحديدهم بشكل غلط بيأدي لـ polarity inversion صامت — خصوصاً مهم في storage controllers اللي فيها fault LEDs بتشتغل active-low.

---

### السيناريو 5: GPIO interrupt latency عالي في real-time networking على CN83XX

#### العنوان
**الـ GPIO interrupt latency على OCTEON TX CN83XX بيأثر على packet timestamping في تطبيق PTP**

#### السياق
شركة بتبني **PTP (Precision Time Protocol) grandmaster clock** على **OCTEON TX CN83XX**. الجهاز بيستقبل 1PPS signal من GPS receiver على GPIO pin — الـ 1PPS interrupt بيتستخدم لمزامنة الـ system clock. أي latency في الـ GPIO interrupt بيأثر مباشرة على دقة الـ timestamps.

#### المشكلة
الـ 1PPS jitter بيوصل لـ 50 microseconds بدل الـ target 1 microsecond. الـ engineer بيشك في الـ GPIO interrupt path.

#### التحليل
الـ binding بيحدد الـ interrupt cell format:
```
- First cell is the GPIO pin number relative to the controller.
- Second cell is triggering flags as defined in interrupts.txt.
```

الـ triggering flags الممكنة (من `interrupts.txt`):
- `IRQ_TYPE_EDGE_RISING` = 1
- `IRQ_TYPE_EDGE_FALLING` = 2
- `IRQ_TYPE_EDGE_BOTH` = 3
- `IRQ_TYPE_LEVEL_HIGH` = 4
- `IRQ_TYPE_LEVEL_LOW` = 8

لو الـ 1PPS signal بيتـ trigger على `IRQ_TYPE_EDGE_BOTH` بدل `IRQ_TYPE_EDGE_RISING`، الـ interrupt handler بيتشغل مرتين في الثانية — مرة على rising edge (الصح) ومرة على falling edge (غلط). ده بيضيف overhead وبيزيد الـ jitter.

كمان، لو الـ DT node ما حددش الـ GPIO controller كـ `interrupt-controller` بشكل صريح، الـ interrupt قد يتـ route عبر intermediate interrupt controller بيضيف latency إضافي.

```dts
/* DT بتاع الـ 1PPS input */
pps_input: pps@0 {
    compatible = "pps-gpio";
    /* غلط — EDGE_BOTH بيتسبب في double trigger */
    gpios = <&gpio_6_0 10 GPIO_ACTIVE_HIGH>;
    interrupts = <10 IRQ_TYPE_EDGE_BOTH>;
    interrupt-parent = <&gpio_6_0>;
};
```

```bash
# قياس الـ interrupt latency
cyclictest -p 99 -t 1 -n -i 1000000 &

# شوف الـ interrupt count
watch -n1 cat /proc/interrupts | grep pps

# لو بيزيد بمعدل 2/second بدل 1/second — EDGE_BOTH هي المشكلة
```

#### الحل
```dts
/* صح — GPIO controller معلن كـ interrupt controller */
gpio_6_0: gpio@6,0 {
    compatible = "cavium,thunder-8890-gpio";
    reg = <0x3000 0 0 0 0>;
    gpio-controller;
    #gpio-cells = <2>;
    interrupt-controller;      /* مهم للـ latency المنخفض */
    #interrupt-cells = <2>;
};

/* صح — EDGE_RISING فقط لـ 1PPS */
pps_input: pps@0 {
    compatible = "pps-gpio";
    gpios = <&gpio_6_0 10 GPIO_ACTIVE_HIGH>;
    interrupts = <10 IRQ_TYPE_EDGE_RISING>;   /* rising edge فقط */
    interrupt-parent = <&gpio_6_0>;
    assert-falling-edge;     /* optional: للـ accuracy */
};
```

```bash
# التحقق من الـ interrupt trigger type
cat /proc/interrupts | grep pps
# يجب يكون 1 interrupt/second بالظبط

# قياس الـ jitter بعد الإصلاح
phc2sys -s CLOCK_REALTIME -c /dev/ptp0 -m -q 2>&1 | grep rms
# target: rms < 100ns
```

#### الدرس المستفاد
في تطبيقات real-time على OCTEON TX CN83XX، اختيار الـ interrupt trigger type في الـ `#interrupt-cells` الـ second cell بيأثر بشكل مباشر على الـ timing accuracy. `IRQ_TYPE_EDGE_RISING` بدل `IRQ_TYPE_EDGE_BOTH` بيقلل الـ interrupt overhead بالنص، وإعلان الـ GPIO node كـ `interrupt-controller` بيضمن إن الـ interrupt path مباشر بدون routing إضافي.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

| المقال | الوصف |
|--------|-------|
| [GPIO in the kernel: an introduction](https://lwn.net/Articles/532909/) | مقدمة شاملة لـ GPIO subsystem في الـ kernel — نقطة البداية الأساسية |
| [GPIO in the kernel: future directions](https://lwn.net/Articles/533632/) | اتجاهات تطوير الـ GPIO API — يشرح لماذا تم استبدال الـ integer-based API |
| [New descriptor-based GPIO interface](https://lwn.net/Articles/565662/) | الـ `gpiod_*` API الحديث اللي بيستخدمه الـ thunderx driver |
| [gpiolib: introduce descriptor-based GPIO interface](https://lwn.net/Articles/528226/) | الـ patch الأصلي اللي أدخل `struct gpio_desc` |
| [gpio: introduce descriptor-based interface](https://lwn.net/Articles/531848/) | تفاصيل implementation الـ descriptor interface |
| [Documentation: gpiolib: document new interface](https://lwn.net/Articles/574055/) | توثيق الـ gpiod API الجديد |
| [genirq/gpio: Add driver for ThunderX and OCTEON-TX SoCs](https://lwn.net/Articles/719714/) | **الأهم** — الـ patch series الأصلي لـ `gpio-thunderx` driver |
| [The pin control subsystem](https://lwn.net/Articles/468759/) | شرح الـ pinctrl subsystem اللي بيتقاطع مع GPIO |

---

### توثيق الـ Kernel الرسمي

الـ `Documentation/` في الـ kernel source:

```
Documentation/devicetree/bindings/gpio/gpio-thunderx.txt   ← ملف الـ binding ده نفسه
Documentation/devicetree/bindings/gpio/gpio.txt            ← الـ generic GPIO DT flags
Documentation/devicetree/bindings/interrupt-controller/interrupts.txt
Documentation/driver-api/gpio/index.rst                    ← دليل كتابة GPIO drivers
Documentation/driver-api/gpio/driver.rst                   ← واجهة gpio_chip
Documentation/driver-api/gpio/consumer.rst                 ← واجهة gpiod_get/set
Documentation/driver-api/gpio/board.rst                    ← GPIO mappings
Documentation/driver-api/gpio/legacy.rst                   ← الـ API القديم (مش موصى بيه)
Documentation/admin-guide/gpio/                            ← استخدام GPIO من user space
```

الـ online kernel docs (من lwn static mirror):

- [General Purpose I/O — kernel docs](https://static.lwn.net/kerneldoc/driver-api/gpio/index.html)
- [GPIO Driver Interface](https://static.lwn.net/kerneldoc/driver-api/gpio/driver.html)
- [GPIO Mappings](https://static.lwn.net/kerneldoc/driver-api/gpio/board.html)
- [GPIO Descriptor Consumer Interface](https://docs.kernel.org/driver-api/gpio/consumer.html)
- [Legacy GPIO Interfaces](https://static.lwn.net/kerneldoc/driver-api/gpio/legacy.html)

---

### الـ Kernel Source Files المهمة

الملفات المباشرة المرتبطة بالـ driver:

```
drivers/gpio/gpio-thunderx.c          ← الـ driver نفسه
include/linux/gpio/driver.h           ← تعريف struct gpio_chip
include/linux/irq.h                   ← irq_domain APIs
drivers/gpio/gpiolib.c                ← core gpiolib implementation
```

---

### Commits المهمة

- **الـ commit الأصلي** — إضافة `gpio-thunderx.c` للـ kernel:
  الـ patch موجود في: [linux.kernel.narkive — PATCH v2 2/3: gpio: Add gpio driver support for ThunderX and OCTEON-TX](https://linux.kernel.narkive.com/XYD3RSdh/patch-v2-2-3-gpio-add-gpio-driver-support-for-thunderx-and-octeon-tx)

- **MAINTAINERS update** — تحديث maintainer الـ driver:
  [lore.kernel.org — PATCH: MAINTAINERS: update Cavium ThunderX drivers](https://lore.kernel.org/linux-arm-kernel/20191119165549.14570-2-rrichter@marvell.com/)

- **Linux Kernel Driver Database** — معلومات الـ `CONFIG_GPIO_THUNDERX`:
  [cateee.net — CONFIG_GPIO_THUNDERX](https://cateee.net/lkddb/web-lkddb/GPIO_THUNDERX.html)

---

### نقاشات الـ Mailing List

- [PATCH v2 1/3: net: mdio-octeon: Modify driver to work on both ThunderX and Octeon](https://www.mail-archive.com/netdev@vger.kernel.org/msg72072.html) — بيوضح pattern تطوير drivers الـ Cavium
- [lore.kernel.org — MAINTAINERS v2 patch](https://lore.kernel.org/all/20191119190436.17875-2-rrichter@marvell.com/) — تحديثات الـ maintainership بعد مغادرة Cavium
- [Patchwork — v2 MAINTAINERS update](https://patchwork.kernel.org/project/linux-soc/patch/20191119190436.17875-2-rrichter@marvell.com/)

---

### Kernelnewbies.org

- [LinuxChanges — تتبع تغييرات الـ GPIO عبر إصدارات الـ kernel](https://kernelnewbies.org/LinuxChanges)
- [Linux 6.2 — GPIO changes](https://kernelnewbies.org/Linux_6.2)
- [Linux 6.6 — GPIO changes](https://kernelnewbies.org/Linux_6.6)
- [Linux 6.13 — GPIO changes](https://kernelnewbies.org/Linux_6.13)
- [mailing list: Using GPIO from device tree on platform devices](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2015-August/014911.html)

---

### eLinux.org

- [New GPIO Interface for User Space — ELCE 2017 slides (PDF)](https://elinux.org/images/7/74/Elce2017_new_GPIO_interface.pdf) — شرح الـ character device interface الحديث
- [Introduction to pin muxing and GPIO control under Linux — ELC 2021 (PDF)](https://elinux.org/images/a/a7/ELC-2021_Introduction_to_pin_muxing_and_GPIO_control_under_Linux.pdf)
- [EBC GPIO via mmap](https://elinux.org/EBC_GPIO_via_mmap) — مثال عملي على memory-mapped GPIO
- [EBC GPIO Polling and Interrupts](https://elinux.org/EBC_gpio_Polling_and_Interrupts) — تفاعل مع GPIO interrupts من user space

---

### كتب موصى بيها

| الكتاب | الفصل ذو الصلة |
|--------|----------------|
| **Linux Device Drivers (LDD3)** — Rubini, Corbet, Kroah-Hartman | Chapter 10: Interrupt Handling |
| **Linux Kernel Development** — Robert Love | Chapter 7: Interrupts and Interrupt Handlers |
| **Embedded Linux Primer** — Christopher Hallinan | Chapter 15: Devices, Drivers, and the Kernel |
| **Professional Linux Kernel Architecture** — Wolfgang Mauerer | Chapter 14: Device Drivers |

> الـ LDD3 متاحة مجاناً: [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

---

### مصطلحات البحث للمزيد

```
gpio-thunderx linux kernel
cavium thunderx gpio driver
CONFIG_GPIO_THUNDERX
gpio_chip irq_domain msi-x
GPIOLIB_IRQCHIP kernel
gpiod descriptor interface kernel
PCI GPIO driver linux
gpio device tree bindings linux
cavium octeon gpio
irq_domain hierarchy linux kernel
```
## Phase 8: Writing simple module

### الفكرة

الـ ThunderX GPIO driver بيسجّل `gpio_chip` وبيشتغل فوق PCI. أهم operation ممكن نعمل لها kprobe هي **`thunderx_gpio_dir_out`** — الـ function اللي بتتنادى كل ما kernel يطلب يغيّر direction بتاع pin لـ output. ده بيحصل كتير (boot, power management, peripheral init) وبيكشف معلومات مهمة: رقم الـ chip، رقم الـ pin، والـ value المطلوبة.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * thunderx_gpio_kprobe.c
 *
 * Kprobe hook on thunderx_gpio_dir_out() — logs every attempt to set
 * a ThunderX GPIO pin as output, including chip base, pin offset, and
 * the requested initial value.
 *
 * Tested against drivers/gpio/gpio-thunderx.c (upstream 5.x / 6.x).
 */

#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit        */
#include <linux/kernel.h>      /* pr_info, pr_fmt                          */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe, …        */
#include <linux/gpio/driver.h> /* struct gpio_chip — to read .base / .label */
#include <linux/ptrace.h>      /* struct pt_regs — gives us the arguments  */

#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt

/* ------------------------------------------------------------------ *
 * Pre-handler — runs BEFORE the probed instruction executes.
 *
 * Prototype of the hooked function (from gpio-thunderx.c):
 *   static int thunderx_gpio_dir_out(struct gpio_chip *chip,
 *                                    unsigned int      offset,
 *                                    int               value);
 *
 * On x86-64 (System V ABI):
 *   rdi = chip   (arg0)
 *   rsi = offset (arg1)
 *   rdx = value  (arg2)
 *
 * On arm64 (AAPCS64):
 *   x0  = chip
 *   x1  = offset
 *   x2  = value
 * ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    struct gpio_chip *chip;
    unsigned int      offset;
    int               value;

#if defined(CONFIG_X86_64)
    chip   = (struct gpio_chip *)regs->di;
    offset = (unsigned int)regs->si;
    value  = (int)regs->dx;
#elif defined(CONFIG_ARM64)
    chip   = (struct gpio_chip *)regs->regs[0];
    offset = (unsigned int)regs->regs[1];
    value  = (int)(long)regs->regs[2];
#else
#  error "Unsupported architecture — add register mapping here"
#endif

    /*
     * chip->base  : absolute GPIO number of pin 0 in this chip
     * chip->label : human-readable name ("thunderx-gpio" or similar)
     * offset      : pin index within this chip (0 … ngpio-1)
     * value       : initial output level (0 = low, 1 = high)
     */
    pr_info("dir_out called | chip='%s' base=%d pin=%u abs_gpio=%d val=%d\n",
            chip->label  ? chip->label : "?",
            chip->base,
            offset,
            chip->base + (int)offset,
            value);

    return 0; /* 0 = let the original function run normally */
}

/* ------------------------------------------------------------------ *
 * Post-handler — runs AFTER the probed instruction (optional).
 * We just confirm the call returned without crashing — good for debug.
 * ------------------------------------------------------------------ */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /* regs->ax (x86) / regs->regs[0] (arm64) holds return value here */
#if defined(CONFIG_X86_64)
    pr_info("dir_out returned %ld\n", (long)regs->ax);
#elif defined(CONFIG_ARM64)
    pr_info("dir_out returned %ld\n", (long)regs->regs[0]);
#endif
}

/* ------------------------------------------------------------------ *
 * kprobe descriptor
 *
 * symbol_name  : الاسم بالظبط زي ما هو في /proc/kallsyms
 *                لو الـ symbol مش exported يشتغل برضو لأن kprobes
 *                بتحل العنوان من kallsyms مش من EXPORT_SYMBOL.
 * ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "thunderx_gpio_dir_out",
    .pre_handler = handler_pre,
    .post_handler = handler_post,
};

/* ------------------------------------------------------------------ */
static int __init thunderx_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("register_kprobe failed: %d\n", ret);
        pr_err("Is thunderx-gpio loaded? Check /proc/kallsyms\n");
        return ret;
    }

    pr_info("kprobe planted at %pS (addr=%px)\n",
            kp.addr, kp.addr);
    return 0;
}

static void __exit thunderx_probe_exit(void)
{
    /*
     * لازم نعمل unregister قبل ما الـ module يتشال من memory —
     * لو سبنا الـ hook وشلنا الـ module هيحصل kernel panic لأن
     * الـ handler pointer هيبقى dangling.
     */
    unregister_kprobe(&kp);
    pr_info("kprobe removed — nmissed=%lu\n", kp.nmissed);
}

module_init(thunderx_probe_init);
module_exit(thunderx_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Learner <example@example.com>");
MODULE_DESCRIPTION("Kprobe demo: trace thunderx_gpio_dir_out calls");
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/module.h` | `module_init`, `module_exit`, `MODULE_LICENSE` |
| `linux/kprobes.h` | `struct kprobe`, `register_kprobe`, `unregister_kprobe` |
| `linux/gpio/driver.h` | `struct gpio_chip` عشان نقرأ منها `base` و `label` |
| `linux/ptrace.h` | `struct pt_regs` اللي بتخلينا نقرأ الـ CPU registers |

الـ includes دي بس اللي محتاجينها — مفيش حاجة زيادة عشان نفضل الـ module خفيف.

---

#### الـ `handler_pre`

ده الـ callback الأهم. بيتشغّل مباشرةً **قبل** أول instruction من `thunderx_gpio_dir_out`. بنقرأ الـ arguments من الـ registers بدل ما نعدّل signature الـ function الأصلية — لأن kprobes مش بتديك pointer للـ args مباشرة، انت بتجيبهم من الـ ABI.

الـ `#if CONFIG_X86_64 / CONFIG_ARM64` موجودة عشان الـ ThunderX بيشتغل على arm64 في الواقع، لكن ممكن تشتغل على x86 في بيئة تطوير — فاحتطنا للحالتين.

---

#### الـ `handler_post`

اختياري بس مفيد في debugging — بيأكدلك إن الـ function رجعت بدون panic وبيطبع الـ return value. تقدر تحذفه في production.

---

#### الـ `struct kprobe`

- **`symbol_name`**: الـ kprobe subsystem بيدور على الاسم ده في `kallsyms` وقت الـ `register_kprobe` — مش محتاج الـ function تكون `EXPORT_SYMBOL`.
- **`nmissed`**: بعد الـ unregister ممكن تطبعها تعرف كام مرة الـ probe اتفوّتت (مثلاً لو كان preemption مغلول).

---

#### الـ `module_init` / `module_exit`

الـ `init` بيحط الـ probe ويطبع العنوان الفعلي (مفيد للتأكد إنه وقع على الـ function الصح). الـ `exit` لازم يعمل `unregister` — لو اتغاضينا عنه والـ module اتشال، أي call لـ `thunderx_gpio_dir_out` بعد كده هيـjump لـ handler بعنوان invalid وده kernel panic فوري.

---

### طريقة التجربة

```bash
# بناء الـ module (Makefile بسيط)
cat > Makefile <<'EOF'
obj-m += thunderx_gpio_kprobe.o
KDIR  ?= /lib/modules/$(shell uname -r)/build
all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules
clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
EOF

make

# تحميل
sudo insmod thunderx_gpio_kprobe.ko

# تأكيد إن الـ probe اتحط
sudo dmesg | tail -5

# شوف الـ output لما يحصل GPIO direction change
# (ممكن تحفّزه عن طريق أي script بيتعامل مع /sys/class/gpio)
sudo dmesg -w &
echo out | sudo tee /sys/class/gpio/gpio<N>/direction

# إزالة
sudo rmmod thunderx_gpio_kprobe
sudo dmesg | tail -3   # هتشوف nmissed
```

---

### ملاحظة على الـ Symbol

لو `thunderx_gpio_dir_out` مش ظاهرة في `/proc/kallsyms` (بسبب inlining أو CONFIG مختلف) استخدم بدلها **`thunderx_gpio_set`** أو **`thunderx_gpio_get`** — نفس الأسلوب بالظبط، بس بتغيّر `symbol_name` وتعدّل قراءة الـ args بناءً على prototype كل function.
