## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الملف `drivers/gpio/gpio-mmio.c` (المعروف تاريخياً بـ `gpio-generic.c`) جزء من **GPIO Subsystem** في Linux kernel. الـ maintainers هم Linus Walleij و Bartosz Golaszewski، والـ mailing list هو `linux-gpio@vger.kernel.org`.

---

### ما هو الـ GPIO أصلاً؟

**GPIO (General Purpose Input/Output)** هو طريقة إن الـ CPU أو الـ SoC يتكلم مع العالم الخارجي عن طريق pins رقمية — كل pin ممكن تبعتها صوت "1" أو "0"، أو تقرأ منها "1" أو "0". زي ما بتشغل كباسة كهربا ON/OFF.

في الأجهزة المدمجة (embedded systems) زي الراوتر، الموبايل، الـ PLC، عندك عادة **GPIO controller** — دايرة إلكترونية صغيرة جوه الـ chip بتتحكم في مجموعة pins. الـ software بيتكلم مع الـ controller ده عن طريق **registers في الذاكرة (MMIO = Memory-Mapped I/O)**.

---

### القصة: المشكلة اللي بيحلها الملف ده

تخيل إنك بتصنع 50 نوع راوتر مختلف، كل واحد فيهم فيه GPIO controller بسيط جوه الـ SoC. الـ controllers دول كلهم بيشتغلوا نفس الفكرة:

```
[ CPU ] ──write/read──► [ MMIO Register 32-bit ] ──► [ 32 GPIO Pins ]
```

بس كل واحد فيه فروق بسيطة:
- بعضهم عنده register واحد بس "dat" — بتكتب فيه 1 تشغل، 0 تطفي.
- بعضهم عنده register منفصل للـ "set" وتاني للـ "clear" — علشان تتجنب الـ read-modify-write.
- بعضهم عنده register اتجاه "dirout/dirin" — لو بتستخدم الـ pin كـ input أو output.
- بعضهم 8-bit، بعضهم 16-bit، 32-bit، أو 64-bit.
- بعضهم big-endian، بعضهم little-endian.

**لو ما فيش كود مشترك**، كل driver بيكتب نفس الـ logic من أول وجديد — ده يعني كود مكرر، bugs مكررة، وصيانة مستحيلة.

**الحل:** `gpio-mmio.c` هو **درايفر generic** بيقدم framework مشترك. أي GPIO controller بسيط من النوع ده — سواء في FPGA أو ASIC أو SoC — مش محتاج يكتب driver من الصفر. بس يقوله: "عندي register في العنوان ده، حجمه 32-bit، وعنده set/clr registers"، والـ framework يتكفل بكل حاجة.

---

### البنية من الناحية النظرية

```
┌─────────────────────────────────────────────────────┐
│              Linux GPIO Subsystem Core               │
│  (gpiolib.c, gpiolib-of.c, gpiolib-acpi-core.c)    │
└───────────────────┬─────────────────────────────────┘
                    │  struct gpio_chip callbacks
                    ▼
┌─────────────────────────────────────────────────────┐
│           gpio-mmio.c  (gpio-generic framework)      │
│                                                     │
│  gpio_generic_chip_init()  ◄── يستقبل config        │
│                                                     │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐            │
│  │  get/set │ │direction │ │ accessors│            │
│  │  logic   │ │ logic    │ │8/16/32bit│            │
│  └──────────┘ └──────────┘ └──────────┘            │
└───────────────────┬─────────────────────────────────┘
                    │  MMIO read/write
                    ▼
┌─────────────────────────────────────────────────────┐
│         Hardware GPIO Controller Registers           │
│   reg_dat │ reg_set │ reg_clr │ reg_dirout │ reg_dirin │
└─────────────────────────────────────────────────────┘
```

---

### الـ 3 أوضاع للكتابة (Output Modes)

| الوضع | الـ Registers | الآلية |
|-------|--------------|--------|
| Single dat | `dat` فقط | بتكتب الـ bit 1 أو 0 مباشرة |
| Set + Clear | `set` + `clr` | كتابة 1 في set يشغّل، كتابة 1 في clr يطفي |
| Set only | `set` فقط | تكتب الـ bit المطلوب، والـ 0 يطفي |

**ليه set/clr مفيدين؟** لأن في الـ SMP (متعدد الأنوية)، لو استخدمت read-modify-write على register واحد، ممكن يحصل race condition. لكن مع set/clr، كل عملية atomic.

### الـ 3 أوضاع للاتجاه (Direction Modes)

| الوضع | الـ Registers | المعنى |
|-------|--------------|--------|
| Bidirectional simple | لا يوجد | كل pins bidirectional افتراضياً |
| Output direction reg | `dirout` | كتابة 1 = output |
| Input direction reg | `dirin` | كتابة 1 = input |

---

### الـ Shadow Registers

الـ framework بيحتفظ بـ **shadow copies** من الـ registers في الذاكرة:

- **`sdata`** — نسخة من آخر قيمة كتبناها في `reg_dat` أو `reg_set`. بتساعد لما الـ register مش readable.
- **`sdir`** — نسخة من اتجاه كل pin (1 = output). بتساعد لما `reg_dir` مش readable.

لو حصل تغيير، بيعمل lock بـ `raw_spinlock_t` علشان يضمن إن الـ shadow والـ hardware يتغيروا مع بعض بدون race.

---

### Big Endian Support

بعض الـ hardware بيعكس ترتيب الـ bits — الـ bit الأعلى (bit 31) هو الـ GPIO line 0. الـ function `gpio_mmio_line2mask()` بتتعامل مع الحالتين:

```c
if (chip->be_bits)
    return BIT(chip->bits - 1 - line); /* big endian: اعكس */
return BIT(line);                       /* normal: عادي */
```

---

### Platform Driver المدمج

الملف كمان فيه platform driver كامل (`gpio_mmio_pdev_probe`) بيسمح للـ hardware اللي بيتعرّف عن طريق Device Tree أو ACPI إنه يشتغل تلقائياً بدون ما حد يكتب درايفر. الـ compatible strings اللي بيدعمها:

- `brcm,bcm6345-gpio` — Broadcom BCM6345
- `wd,mbl-gpio` — Western Digital
- `ni,169445-nand-gpio` — National Instruments
- `intel,ixp4xx-expansion-bus-mmio-gpio` — Intel IXP4xx
- `opencores,gpio` — OpenCores IP

---

### الملفات المهمة في الـ Subsystem

#### Core الـ GPIO Subsystem
| الملف | الدور |
|-------|-------|
| `drivers/gpio/gpiolib.c` | قلب الـ GPIO subsystem، يدير كل الـ chip registration والـ descriptors |
| `drivers/gpio/gpiolib.h` | internal header للـ gpiolib |
| `drivers/gpio/gpiolib-of.c` | ربط GPIO بالـ Device Tree |
| `drivers/gpio/gpiolib-acpi-core.c` | ربط GPIO بالـ ACPI |
| `drivers/gpio/gpiolib-cdev.c` | الـ character device interface للـ userspace (ABI الجديد) |
| `drivers/gpio/gpiolib-sysfs.c` | الـ sysfs interface القديم |
| `drivers/gpio/gpiolib-devres.c` | managed resources (devm_*) للـ GPIO |

#### الملف نفسه والـ Header
| الملف | الدور |
|-------|-------|
| `drivers/gpio/gpio-mmio.c` | **الملف الرئيسي** — generic MMIO GPIO framework |
| `include/linux/gpio/generic.h` | يعرّف `struct gpio_generic_chip`، `struct gpio_generic_chip_config`، والـ flags |
| `include/linux/gpio/driver.h` | يعرّف `struct gpio_chip` — الـ base struct لكل GPIO driver |

#### أمثلة على Drivers تستخدم الـ Framework
| الملف | الـ Hardware |
|-------|-------------|
| `drivers/gpio/gpio-ath79.c` | Qualcomm Atheros AR7xxx/9xxx |
| `drivers/gpio/gpio-mpc8xxx.c` | Freescale MPC8xxx |
| `drivers/gpio/gpio-vf610.c` | NXP Vybrid VF610 |
| `drivers/gpio/gpio-visconti.c` | Toshiba Visconti |
| `drivers/gpio/gpio-idt3243x.c` | IDT 79RC3243x |
| `drivers/gpio/gpio-loongson-64bit.c` | Loongson 64-bit SoCs |
| `drivers/gpio/gpio-brcmstb.c` | Broadcom STB |

---

### ملخص سريع

**الـ** `gpio-mmio.c` هو "مصنع جاهز" للـ GPIO controllers البسيطة. أي SoC أو FPGA فيه GPIO controller من النوع ده (MMIO registers) مش محتاج يكتب حاجة من الصفر — بس يعبي الـ `gpio_generic_chip_config` بعناوين الـ registers، ويستدعي `gpio_generic_chip_init()`، والـ framework يسجّله في الـ GPIO subsystem تلقائياً.
## Phase 2: شرح الـ GPIO MMIO (gpio-generic) Framework

---

### المشكلة اللي بيحلها الـ Subsystem

في أي SoC أو FPGA، عندك GPIO controller بيتكون في الجوهر من register واحد أو أكتر mapped في الـ physical address space. الـ CPU بيكتب/يقرأ من الـ register ده بـ MMIO (Memory-Mapped I/O) عشان يتحكم في الـ pins.

المشكلة إن كل hardware vendor بيعمل الـ registers بطريقة مختلفة شوية:
- في controller بيبعت value مباشرة في **data register** واحد (بتكتب 1 يعمل high، بتكتب 0 يعمل low).
- في تاني عنده **set register** و **clear register** منفصلين (بتكتب 1 في set يعمل high، بتكتب 1 في clear يعمل low — بدون read-modify-write).
- في ثالت عنده **direction register** يفصل input عن output.
- بعضهم **big-endian** في bit ordering.
- بعضهم 8-bit registers، بعضهم 16، 32، أو 64-bit.

لو مفيش abstraction، كل FPGA design أو SoC-variant لازم يكتب driver من الصفر مع نفس الـ boilerplate.

---

### الحل

**gpio-mmio** (اللي اتعرف تاريخياً بـ `gpio-generic`) هو driver جاهز وقابل للتهيئة بالكامل بيغطي كل الـ variations دي. بدل ما تكتب driver، بتديه الـ MMIO addresses بتاعت الـ registers وبعض flags، وهو بيختار الـ callbacks الصح ويسجّل الـ `gpio_chip` في الـ gpiolib.

الفكرة الجوهرية: **configuration-driven dispatch** — بدل inheritance أو إعادة كتابة، بتحدد الـ hardware topology بـ config struct، والـ driver يملأ الـ function pointers التلقائي.

---

### التشبيه الواقعي (Analogy)

تخيل **مفرمة لحمة صناعية** قابلة للتهيئة:

| جزء الـ Analogy | المقابل في الكود |
|---|---|
| الـ input chute | `reg_dat` — بتقرأ منه حالة الـ GPIO lines |
| زر الـ "forward" | `reg_set` — بتكتب 1 عشان تعمل pin = HIGH |
| زر الـ "reverse" | `reg_clr` — بتكتب 1 عشان تعمل pin = LOW |
| lever اختيار الاتجاه | `reg_dir_out` / `reg_dir_in` — بيحدد كل pin input أو output |
| حجم فتحة الـ input | `bits` (8/16/32/64) — عرض الـ register |
| اتجاه قراءة العداد | `be_bits` — هل bit 0 = line 0 أم bit 0 = line (N-1) |
| الـ safety lock | `raw_spinlock_t lock` + `sdata`/`sdir` — shadow registers لتفادي race conditions |
| التشغيل الآلي بمجرد التهيئة | `gpio_generic_chip_init()` بيملأ الـ function pointers تلقائياً |

بمجرد ما تحدد إزاي المفرمة متصلة (registers)، هي بتشتغل من غير ما تحتاج تفهم كل دواخلها.

---

### المكان في الـ Kernel Architecture

```
 ┌─────────────────────────────────────────────────────┐
 │           User Space / Kernel Consumers              │
 │   (Device Drivers, sysfs /sys/class/gpio, libgpiod)  │
 └────────────────────┬────────────────────────────────┘
                      │ gpiod_get() / gpiod_set_value()
 ┌────────────────────▼────────────────────────────────┐
 │                   gpiolib (gpiolib.c)                │
 │     - GPIO descriptor management (gpio_desc)        │
 │     - GPIO device registry (gpio_device)            │
 │     - sysfs / cdev interface                        │
 │     - IRQ integration (gpiolib-irqchip)             │
 └────────────────────┬────────────────────────────────┘
                      │ gpio_chip callbacks
                      │ (.get, .set, .direction_input, ...)
 ┌────────────────────▼────────────────────────────────┐
 │           gpio-mmio (gpio-generic) Driver            │
 │                                                      │
 │  gpio_generic_chip_init()                           │
 │  ┌──────────────────────────────────────────────┐   │
 │  │  gpio_mmio_setup_io()                        │   │
 │  │    → assigns: gc->get, gc->set,              │   │
 │  │               gc->get_multiple, gc->set_multiple│ │
 │  │                                              │   │
 │  │  gpio_mmio_setup_direction()                 │   │
 │  │    → assigns: gc->direction_input,           │   │
 │  │               gc->direction_output,          │   │
 │  │               gc->get_direction              │   │
 │  │                                              │   │
 │  │  gpio_mmio_setup_accessors()                 │   │
 │  │    → assigns: chip->read_reg, chip->write_reg│   │
 │  └──────────────────────────────────────────────┘   │
 └────────────────────┬────────────────────────────────┘
                      │ readb/readw/readl/readq
                      │ writeb/writew/writel/writeq
 ┌────────────────────▼────────────────────────────────┐
 │              MMIO Hardware Registers                 │
 │                                                      │
 │   reg_dat   reg_set   reg_clr   reg_dir_out/in      │
 │  [0x0000]  [0x0004]  [0x0008]   [0x000C]            │
 │                                                      │
 │   (SoC GPIO block / FPGA / BCM6345 / IXP4xx / ...)  │
 └─────────────────────────────────────────────────────┘
```

**Consumers** (اللي بتستخدم الـ GPIO):
- Kernel drivers: SPI bit-bang، LED، button، reset، power sequencing
- الـ `pinctrl` subsystem (لما `GPIO_GENERIC_PINCTRL_BACKEND` يكون set)

**Providers** (الـ hardware اللي بيشتغل عليه الـ driver):
- FPGA GPIO blocks
- `brcm,bcm6345-gpio` (Broadcom broadband SoCs)
- `opencores,gpio`
- `ni,169445-nand-gpio` (National Instruments embedded)
- `intel,ixp4xx-expansion-bus-mmio-gpio`
- Any custom ASIC/FPGA GPIO block

---

### الـ Core Abstraction

الـ abstraction المحورية هي **`struct gpio_generic_chip`** — وهي wrapping لـ `struct gpio_chip` بتضيف:

1. **Function pointers للـ MMIO access** (`read_reg` / `write_reg`) — بدل hard-coding الـ width.
2. **Shadow registers** (`sdata`, `sdir`) — لأن بعض hardware لا يمكن قراءة حالته الحالية (write-only registers)، فالـ driver بيحتفظ بـ software copy.
3. **Endianness handling** (`be_bits`) — بعض الـ hardware بيعكس ترتيب الـ bits.
4. **Register topology** — مجموعة `void __iomem *` pointers لكل register ممكن.

---

### تفصيل الـ Structs

#### `struct gpio_chip` (الـ base class)

```c
struct gpio_chip {
    const char     *label;        /* اسم الـ controller */
    struct gpio_device *gpiodev;  /* الـ internal state في gpiolib */
    struct device  *parent;       /* الـ parent device */

    /* Function pointers — gpiolib بتستدعيهم */
    int  (*request)(struct gpio_chip *gc, unsigned int offset);
    int  (*get_direction)(struct gpio_chip *gc, unsigned int offset);
    int  (*direction_input)(struct gpio_chip *gc, unsigned int offset);
    int  (*direction_output)(struct gpio_chip *gc, unsigned int offset, int value);
    int  (*get)(struct gpio_chip *gc, unsigned int offset);
    int  (*set)(struct gpio_chip *gc, unsigned int offset, int value);
    int  (*get_multiple)(struct gpio_chip *gc, unsigned long *mask, unsigned long *bits);
    int  (*set_multiple)(struct gpio_chip *gc, unsigned long *mask, unsigned long *bits);

    int   base;    /* أول GPIO number (أو -1 للـ dynamic allocation) */
    u16   ngpio;   /* عدد الـ GPIO lines */
    bool  can_sleep; /* هل الـ access ممكن يـ block؟ — لـ I2C/SPI expanders */
};
```

#### `struct gpio_generic_chip` (الـ MMIO extension)

```c
struct gpio_generic_chip {
    struct gpio_chip gc;        /* لازم يكون أول field — container_of يشتغل */

    /* MMIO accessors — بيتغير حسب register width والـ endianness */
    unsigned long (*read_reg)(void __iomem *reg);
    void (*write_reg)(void __iomem *reg, unsigned long data);

    /* Register addresses */
    void __iomem *reg_dat;      /* data / input register */
    void __iomem *reg_set;      /* set register (output high) */
    void __iomem *reg_clr;      /* clear register (output low) */
    void __iomem *reg_dir_out;  /* direction: 1=output */
    void __iomem *reg_dir_in;   /* direction: 1=input */

    /* Shadow state — software copy للـ hardware state */
    unsigned long sdata;        /* shadow of output data register */
    unsigned long sdir;         /* shadow of direction: 1=output */

    /* Configuration */
    bool be_bits;               /* big-endian bit ordering */
    bool dir_unreadable;        /* direction register is write-only */
    bool pinctrl;               /* has pinctrl backend */
    int  bits;                  /* register width in bits: 8/16/32/64 */

    raw_spinlock_t lock;        /* protects sdata/sdir + write sequences */
};
```

#### العلاقة البيانية بين الـ Structs

```
  struct gpio_generic_chip
  ┌──────────────────────────────────────────────┐
  │  struct gpio_chip gc  ◄─ gpiolib بتشوفها    │
  │  ┌────────────────────────────────────────┐  │
  │  │  .get         = gpio_mmio_get()        │  │
  │  │  .set         = gpio_mmio_set()        │  │
  │  │  .direction_input  = gpio_mmio_dir_in()│  │
  │  │  .direction_output = gpio_mmio_dir_out_│  │
  │  │                      val_first()       │  │
  │  │  .get_direction    = gpio_mmio_get_dir()│ │
  │  └────────────────────────────────────────┘  │
  │                                              │
  │  read_reg  = gpio_mmio_read32()  ◄── (مثلاً)│
  │  write_reg = gpio_mmio_write32()             │
  │                                              │
  │  reg_dat   = 0xFE010000  ◄── ioremap result  │
  │  reg_set   = 0xFE010004                      │
  │  reg_clr   = 0xFE010008                      │
  │  reg_dir_out = 0xFE01000C                    │
  │                                              │
  │  sdata = 0x0000_00A0  ◄── software shadow    │
  │  sdir  = 0x0000_00F0  ◄── output pins mask   │
  │  lock  (raw_spinlock)                        │
  └──────────────────────────────────────────────┘
           │
           │ container_of(gc, struct gpio_generic_chip, gc)
           ▼
  كل callback بيـ cast الـ gpio_chip pointer للـ gpio_generic_chip
```

---

### الـ Register Topologies المدعومة

#### Topology 1: Single Data Register

```
Write: read-modify-write على reg_dat
                   ┌──────────────────────────────┐
  GPIO lines  ──── │  DAT register  [0xAA]        │
                   │  bit[n] = 1 → line n = HIGH   │
                   │  bit[n] = 0 → line n = LOW    │
                   └──────────────────────────────┘
  يحتاج: shadow register (sdata) + spinlock عشان atomic read-modify-write
```

#### Topology 2: Set/Clear Pair

```
Write 1 to SET  → line goes HIGH (no read needed)
Write 1 to CLR  → line goes LOW  (no read needed)

  ┌──────────────────────┐   ┌──────────────────────┐
  │  SET register        │   │  CLR register        │
  │  Write 1 → HIGH      │   │  Write 1 → LOW       │
  └──────────────────────┘   └──────────────────────┘
  ميزته: atomic بطبيعته — مفيش race condition في الـ hardware
```

#### Topology 3: Set register + Data register للقراءة

```
  ┌──────────────────────┐
  │  DAT register        │ ← للقراءة فقط (actual pin state)
  └──────────────────────┘
  ┌──────────────────────┐
  │  SET register        │ ← للكتابة (active-low أو لو بتبعت shadow)
  └──────────────────────┘
```

---

### الـ Shadow Registers — ليه ضروريين؟

بعض الـ hardware registers **write-only** — يعني لما تكتب فيها تروح، مش ممكن تقرأ القيمة اللي كتبتها. المشكلة في الـ topology الأولى (single data register): لو عايز تعمل pin 3 = HIGH وأنت مش عارف القيمة الحالية للـ register، هتعمل read وتعدل، لكن لو process تانية أو interrupt عدّل الـ register في النص → **race condition**.

الحل:
```c
// في gpio_mmio_set():
guard(raw_spinlock_irqsave)(&chip->lock);  // disable IRQs + lock

if (val)
    chip->sdata |= mask;   // عدّل الـ shadow
else
    chip->sdata &= ~mask;

chip->write_reg(chip->reg_dat, chip->sdata);  // اكتب الـ shadow كله دفعة واحدة
// الـ lock بيتفك تلقائياً هنا (scoped guard)
```

- `sdata` = دايماً in-sync مع الـ hardware output value
- `sdir`  = دايماً in-sync مع الـ direction setting

---

### Endianness والـ Bit Ordering

الـ `be_bits` flag مش بيأثر على الـ byte order للقراءة، ده بيأثر على **ترتيب الـ bits جوّا الـ register**.

```
Normal (little-endian bits):
  bit[0] of register = GPIO line 0
  bit[1] of register = GPIO line 1
  ...
  bit[31] of register = GPIO line 31

Big-endian bits (be_bits = true):
  bit[31] of register = GPIO line 0
  bit[30] of register = GPIO line 1
  ...
  bit[0] of register = GPIO line 31
```

الـ `gpio_mmio_line2mask()` بيعالج ده:
```c
static unsigned long gpio_mmio_line2mask(struct gpio_chip *gc, unsigned int line)
{
    struct gpio_generic_chip *chip = to_gpio_generic_chip(gc);

    if (chip->be_bits)
        return BIT(chip->bits - 1 - line);  // عكس الـ bit
    return BIT(line);                        // مباشر
}
```

---

### الـ Direction Management

#### لو الـ Hardware عنده Direction Registers

```c
// direction_output — الترتيب مهم جداً
// خيار 1: dir_first (GPIO_GENERIC_NO_SET_ON_INPUT)
gpio_mmio_dir_out_dir_first():
    gpio_mmio_dir_out(gc, gpio, val);  // أول حوّل الاتجاه لـ output
    gc->set(gc, gpio, val);            // بعدين اكتب القيمة

// خيار 2: val_first (الأكثر شيوعاً)
gpio_mmio_dir_out_val_first():
    gc->set(gc, gpio, val);            // أول اكتب القيمة (pin لسه input)
    gpio_mmio_dir_out(gc, gpio, val);  // بعدين حوّل لـ output
```

الفرق بين الاتنين مهم عشان بعض الـ hardware لو حولت direction أول من غير ما تكتب value، الـ pin بيطلع بقيمة undefined للحظة (glitch) — ده ممكن يرحرح reset أو power line.

#### لو الـ Hardware مفيش عنده Direction Registers

```c
// Simple fixed direction
gc->direction_output = gpio_mmio_simple_dir_out;  // بيكتب القيمة بس
gc->direction_input  = gpio_mmio_simple_dir_in;   // بيطلب pinctrl لو موجود
```

---

### الـ Initialization Flow

```
gpio_mmio_pdev_probe()          ← Platform driver entry point
    │
    ├── platform_get_resource_byname(pdev, IORESOURCE_MEM, "dat")
    ├── devm_ioremap_resource()  ← map المناطق دي للـ virtual address space
    │
    ├── gpio_generic_chip_init(gen_gc, &config)
    │       │
    │       ├── chip->bits = cfg->sz * 8       (حساب عرض الـ register)
    │       ├── gpiochip_get_ngpios()          (من device tree أو = bits)
    │       │
    │       ├── gpio_mmio_setup_io()           (اختيار get/set callbacks)
    │       ├── gpio_mmio_setup_accessors()    (اختيار read_reg/write_reg)
    │       ├── gpio_mmio_setup_direction()    (اختيار direction callbacks)
    │       │
    │       ├── chip->sdata = chip->read_reg(chip->reg_dat)  ← sync shadow
    │       └── chip->sdir  = chip->read_reg(chip->reg_dir_out)
    │
    └── devm_gpiochip_add_data()   ← تسجيل في gpiolib
```

---

### ما يمتلكه الـ Subsystem مقابل ما يُفوّضه

| المهمة | المسؤول |
|--------|---------|
| اختيار الـ read/write function pointer حسب register width | **gpio-mmio** |
| تنفيذ الـ get/set callbacks بكل الـ variants | **gpio-mmio** |
| إدارة الـ shadow registers والـ locking | **gpio-mmio** |
| معالجة الـ endianness (byte order + bit order) | **gpio-mmio** |
| تهيئة وتسجيل الـ gpio_chip في gpiolib | **gpio-mmio** |
| تحديد أي registers موجودة وأي flags تنطبق | **Driver/Board code** |
| عمل ioremap للـ physical addresses | **Platform driver (gpio_mmio_pdev_probe)** |
| إدارة IRQs | **مش في gpio-mmio — يُفوَّض لـ gpiolib-irqchip** |
| إدارة الـ GPIO consumers (allocation, name lookup) | **gpiolib.c** |
| الـ pinmux (اختيار ما إذا الـ pin يكون GPIO أو function أخرى) | **pinctrl subsystem** |

---

### الـ Pinctrl Integration

الـ `GPIO_GENERIC_PINCTRL_BACKEND` flag بيوصّل gpio-mmio بالـ **pinctrl subsystem**.

**الـ pinctrl subsystem**: بيتحكم في الـ pin multiplexing — كل pin ممكن يكون GPIO أو UART TX أو SPI CLK أو غيره. لما driver يطلب GPIO line، pinctrl بيتأكد إن الـ pin مش بيستخدم في function تانية.

لما الـ flag ده يكون set:
```c
// في gpio_generic_chip_init():
chip->pinctrl = true;
gc->free = gpiochip_generic_free;   // لما الـ GPIO يتحرر → pinctrl يتبلّغ

// في gpio_mmio_request():
if (chip->pinctrl)
    return gpiochip_generic_request(gc, gpio_pin);  // ← pinctrl يحجز الـ pin

// في direction callbacks:
gpio_mmio_dir_return(gc, gpio, dir_out):
    if (chip->pinctrl)
        pinctrl_gpio_direction_output(gc, gpio);  // ← pinctrl يعدل الـ mux
```

---

### مثال عملي: BCM6345 GPIO

الـ `brcm,bcm6345-gpio` (موجود في الـ of_match table) ده GPIO controller شائع في Broadcom broadband SoCs (routers/DSL modems). بيكون عنده:

```
Device Tree:
gpio0: gpio@fffe0406 {
    compatible = "brcm,bcm6345-gpio";
    reg-names = "dirout", "dat";
    reg = <0xfffe0406 1>, <0xfffe0407 1>;
    #gpio-cells = <2>;
    gpio-controller;
};
```

الـ driver بيأخذ `dirout` و `dat` كـ platform resources، يعمل ioremap، ويمرّرهم لـ `gpio_generic_chip_init()`. من غير ما يكتب ولا سطر في الـ get/set/direction.

---

### نقطة دقيقة: الـ `get_set` vs `get` للـ Output Pins

لما pin يكون output، تقرأ قيمته منين؟ من `reg_dat` (actual pin state) أم من `reg_set` (اللي انت كتبته)?

الفرق مهم في حالة **open-drain** أو لما الـ hardware يعكس القيمة:

```c
// لو GPIO_GENERIC_READ_OUTPUT_REG_SET set:
gc->get = gpio_mmio_get_set;
// → لو output pin: اقرأ من reg_set
// → لو input pin: اقرأ من reg_dat

// لو مش set (الافتراضي):
gc->get = gpio_mmio_get;
// → دايماً اقرأ من reg_dat (actual pin state)
```

ده مهم جداً لـ debugging: تقدر تقرأ حالة الـ pin الفعلية (هل فيه load بيشدّه لـ low رغم إنك كتبت HIGH؟) من `reg_dat`.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

> **الملف الأصلي:** `drivers/gpio/gpio-mmio.c` (كان اسمه `gpio-generic.c` في كيرنلات أقدم)
> **الـ headers الأساسية:** `include/linux/gpio/generic.h` و `include/linux/gpio/driver.h`

---

### 0. الـ Flags والـ Config Options — Cheatsheet

#### الـ Flags المعرَّفة في `<linux/gpio/generic.h>`

| Flag | Bit | معناه |
|------|-----|-------|
| `GPIO_GENERIC_BIG_ENDIAN` | 0 | ترتيب البتات معكوس — البت 31 هو line 0 |
| `GPIO_GENERIC_UNREADABLE_REG_SET` | 1 | `reg_set` مش ممكن تقرأ منه (write-only) |
| `GPIO_GENERIC_UNREADABLE_REG_DIR` | 2 | `reg_dir` مش ممكن تقرأ منه — لازم تتبع الحالة بنفسك |
| `GPIO_GENERIC_BIG_ENDIAN_BYTE_ORDER` | 3 | ترتيب البايتات big-endian (يأثر على `ioread/iowrite`) |
| `GPIO_GENERIC_READ_OUTPUT_REG_SET` | 4 | `reg_set` بيعكس القيمة الحالية للـ output |
| `GPIO_GENERIC_NO_OUTPUT` | 5 | الـ chip input-only، مفيش output |
| `GPIO_GENERIC_NO_SET_ON_INPUT` | 6 | اتجه للـ direction أولاً قبل ما تكتب القيمة |
| `GPIO_GENERIC_PINCTRL_BACKEND` | 7 | استخدم pinctrl لضبط الاتجاه |
| `GPIO_GENERIC_NO_INPUT` | 8 | الـ chip output-only، مفيش input |

#### الـ Kconfig المتعلق بالملف

| Option | الوظيفة |
|--------|---------|
| `CONFIG_GPIO_GENERIC` | يبني `gpio-mmio.c` كـ module أو built-in |
| `CONFIG_GPIO_GENERIC_PLATFORM` | يفعّل `gpio_mmio_pdev_probe` وقت probe الـ platform device |

#### ثوابت اتجاه الـ GPIO

| Macro | قيمة | المعنى |
|-------|------|--------|
| `GPIO_LINE_DIRECTION_IN` | 1 | السطر input |
| `GPIO_LINE_DIRECTION_OUT` | 0 | السطر output |

---

### 1. الـ Structs المهمة

#### `struct gpio_generic_chip_config`
**الغرض:** بيانات الـ configuration اللي بتتمرر لـ `gpio_generic_chip_init()` — بتوصف كيف يتواصل الـ driver مع الـ hardware.

| Field | النوع | الشرح |
|-------|-------|-------|
| `dev` | `struct device *` | الـ parent device (إلزامي) |
| `sz` | `unsigned long` | عرض الـ register بالبايت (1، 2، 4، أو 8) |
| `dat` | `void __iomem *` | عنوان الـ data register (قراءة حالة الـ GPIO) |
| `set` | `void __iomem *` | عنوان الـ set register (رفع السطر HIGH) — ممكن يكون NULL |
| `clr` | `void __iomem *` | عنوان الـ clear register (خفض السطر LOW) — ممكن يكون NULL |
| `dirout` | `void __iomem *` | عنوان direction-out register (1 = output) |
| `dirin` | `void __iomem *` | عنوان direction-in register (1 = input) |
| `flags` | `unsigned long` | مجموعة من الـ `GPIO_GENERIC_*` flags |

**الارتباط:** بتتمرر كـ `const *` لـ `gpio_generic_chip_init()` وبعدها لا حاجة ليها — الـ driver ممكن يحطها على الـ stack.

---

#### `struct gpio_generic_chip`
**الغرض:** الـ object الرئيسي للـ driver — بيغلّف `struct gpio_chip` ويضيف تفاصيل الـ MMIO والـ shadow registers.

| Field | النوع | الشرح |
|-------|-------|-------|
| `gc` | `struct gpio_chip` | الـ core GPIO object — **أول field** عشان `container_of` يشتغل |
| `read_reg` | `fn ptr` | pointer لدالة القراءة المناسبة (8/16/32/64 bit) |
| `write_reg` | `fn ptr` | pointer لدالة الكتابة المناسبة |
| `be_bits` | `bool` | هل ترتيب البتات big-endian؟ |
| `reg_dat` | `void __iomem *` | عنوان data register في الـ hardware |
| `reg_set` | `void __iomem *` | عنوان set register |
| `reg_clr` | `void __iomem *` | عنوان clear register |
| `reg_dir_out` | `void __iomem *` | عنوان direction-out register |
| `reg_dir_in` | `void __iomem *` | عنوان direction-in register |
| `dir_unreadable` | `bool` | الـ direction registers مش ممكن تتقرأ |
| `pinctrl` | `bool` | هل يستخدم pinctrl backend؟ |
| `bits` | `int` | عدد البتات = `sz * 8` |
| `lock` | `raw_spinlock_t` | يحمي `sdata` و `sdir` |
| `sdata` | `unsigned long` | **shadow** للـ data register (القيمة المكتوبة آخر مرة) |
| `sdir` | `unsigned long` | **shadow** للـ direction (1 = output، 0 = input) |

**الارتباط بـ structs تانية:**
- يحتوي على `struct gpio_chip gc` — الـ GPIO core بيشتغل على الـ `gc` مباشرة
- `to_gpio_generic_chip(gc)` بيرجع لـ `gpio_generic_chip` باستخدام `container_of`

---

#### `struct gpio_chip` (من `<linux/gpio/driver.h>`)
**الغرض:** الـ abstraction الأساسية في GPIO subsystem — كل controller بيسجّل نفسه بـ `gpio_chip`.

| Field مهم | الشرح في سياق الـ driver |
|-----------|--------------------------|
| `parent` | الـ `struct device` الأب |
| `label` | اسم الـ chip (من `dev_name(dev)`) |
| `base` | أول GPIO number عالمي (-1 = kernel يختار) |
| `ngpio` | عدد الـ GPIO lines |
| `request` | callback لطلب سطر معين → `gpio_mmio_request` |
| `free` | callback لتحرير سطر → `gpiochip_generic_free` (لو pinctrl) |
| `get_direction` | → `gpio_mmio_get_dir` |
| `direction_input` | → `gpio_mmio_dir_in` أو `gpio_mmio_simple_dir_in` |
| `direction_output` | → `gpio_mmio_dir_out_dir_first` أو `gpio_mmio_dir_out_val_first` |
| `get` | → `gpio_mmio_get` أو `gpio_mmio_get_set` |
| `get_multiple` | → `gpio_mmio_get_multiple` أو `gpio_mmio_get_multiple_be` |
| `set` | → `gpio_mmio_set` أو `gpio_mmio_set_with_clear` أو غيره |
| `set_multiple` | → نظير لـ `set` لكن للـ batch operations |

---

### 2. مخططات العلاقات بين الـ Structs

```
┌─────────────────────────────────────────────────────────────────┐
│              struct gpio_generic_chip                           │
│                                                                 │
│  ┌─────────────────────────────┐                               │
│  │     struct gpio_chip gc     │ ◄── GPIO core يشتغل على ده   │
│  │  .parent  → struct device   │                               │
│  │  .get     → fn ptr          │                               │
│  │  .set     → fn ptr          │                               │
│  │  .direction_input → fn ptr  │                               │
│  │  .direction_output → fn ptr │                               │
│  └─────────────────────────────┘                               │
│                                                                 │
│  read_reg  ──► gpio_mmio_read8/16/32/64()                      │
│  write_reg ──► gpio_mmio_write8/16/32/64()                     │
│                                                                 │
│  reg_dat  ──► [ MMIO DATA   register ]                         │
│  reg_set  ──► [ MMIO SET    register ]                         │
│  reg_clr  ──► [ MMIO CLEAR  register ]                         │
│  reg_dir_out─► [ MMIO DIROUT register ]                        │
│  reg_dir_in──► [ MMIO DIRIN  register ]                        │
│                                                                 │
│  lock  ─── raw_spinlock_t (يحمي sdata, sdir)                  │
│  sdata ─── unsigned long  (shadow of reg_dat/reg_set)          │
│  sdir  ─── unsigned long  (shadow direction state)             │
└─────────────────────────────────────────────────────────────────┘
         │
         │ container_of(gc, gpio_generic_chip, gc)
         ▼
┌─────────────────────────┐
│   struct gpio_chip *gc  │ ──── يُسجَّل في GPIO core
│                         │      عبر devm_gpiochip_add_data()
└─────────────────────────┘
         │
         ▼
┌─────────────────────────┐
│  struct gpio_device     │  ◄── GPIO core ينشئه داخلياً
│  (internal to gpiolib)  │
└─────────────────────────┘
```

```
struct gpio_generic_chip_config
  .dev    ──► struct device (platform_device.dev)
  .dat    ──► void __iomem *  ─┐
  .set    ──► void __iomem *   ├── نتائج devm_ioremap_resource()
  .clr    ──► void __iomem *   │
  .dirout ──► void __iomem *   │
  .dirin  ──► void __iomem * ─┘
  .flags  ──► GPIO_GENERIC_* bitmask
         │
         │ تُمرَّر لـ gpio_generic_chip_init()
         ▼
struct gpio_generic_chip
  (ينسخ العناوين ويضبط الـ callbacks)
```

---

### 3. دورة حياة الـ Driver (Lifecycle Diagram)

```
                    ┌─────────────────────────────┐
                    │   platform_device registered │
                    │   (DT: "brcm,bcm6345-gpio"  │
                    │    أو platform_device_id)    │
                    └────────────┬────────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────────┐
                    │  gpio_mmio_pdev_probe()      │
                    │  ┌───────────────────────┐  │
                    │  │ platform_get_resource  │  │
                    │  │ _byname("dat")         │  │
                    │  └──────────┬────────────┘  │
                    │             │               │
                    │  ┌──────────▼────────────┐  │
                    │  │ gpio_mmio_map()        │  │
                    │  │ devm_ioremap_resource()│  │
                    │  │ لكل: dat,set,clr,      │  │
                    │  │      dirout,dirin      │  │
                    │  └──────────┬────────────┘  │
                    │             │               │
                    │  ┌──────────▼────────────┐  │
                    │  │ devm_kzalloc()         │  │
                    │  │ alloc gpio_generic_chip│  │
                    │  └──────────┬────────────┘  │
                    │             │               │
                    │  ┌──────────▼────────────┐  │
                    │  │ gpio_generic_chip_init │  │
                    │  │ ()                     │  │
                    │  └──────────┬────────────┘  │
                    │             │               │
                    │  ┌──────────▼────────────┐  │
                    │  │ devm_gpiochip_add_data │  │
                    │  │ ()  ← يسجّل في core    │  │
                    │  └──────────┬────────────┘  │
                    └────────────┼────────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────────┐
                    │   CHIP ACTIVE                │
                    │  GPIO lines متاحة للمستخدمين │
                    │  (kernel + userspace)        │
                    └────────────┬────────────────┘
                                 │
                    (device removed / driver unloaded)
                                 │
                                 ▼
                    ┌─────────────────────────────┐
                    │   devm cleanup يشتغل:        │
                    │   - gpiochip_remove()        │
                    │   - iounmap() للـ regs       │
                    │   - kfree() للـ chip         │
                    └─────────────────────────────┘
```

**داخل `gpio_generic_chip_init()` بالتفصيل:**

```
gpio_generic_chip_init(chip, cfg)
  │
  ├─ تحقق: is_power_of_2(cfg->sz)
  ├─ chip->bits = cfg->sz * 8
  ├─ تحقق: bits <= BITS_PER_LONG
  ├─ raw_spin_lock_init(&chip->lock)
  ├─ gc->parent, gc->label, gc->base = -1
  ├─ gc->request = gpio_mmio_request
  ├─ chip->be_bits ← GPIO_GENERIC_BIG_ENDIAN flag
  ├─ gpiochip_get_ngpios() → gc->ngpio (أو fallback لـ chip->bits)
  │
  ├─ gpio_mmio_setup_io(chip, cfg)
  │     ├─ chip->reg_dat = cfg->dat
  │     ├─ يختار gc->set callback بناءً على وجود set/clr
  │     └─ يختار gc->get callback بناءً على الـ flags
  │
  ├─ gpio_mmio_setup_accessors(dev, chip, byte_be)
  │     └─ يختار read_reg/write_reg بناءً على chip->bits
  │
  ├─ gpio_mmio_setup_direction(chip, cfg)
  │     ├─ لو dirout أو dirin موجودين:
  │     │   ├─ gc->direction_output ← dir_first أو val_first
  │     │   ├─ gc->direction_input  ← gpio_mmio_dir_in
  │     │   └─ gc->get_direction    ← gpio_mmio_get_dir
  │     └─ لو لا: simple_dir_in / simple_dir_out
  │
  ├─ chip->pinctrl ← GPIO_GENERIC_PINCTRL_BACKEND
  ├─ chip->sdata ← قراءة reg_dat (أو reg_set)
  ├─ chip->dir_unreadable ← UNREADABLE_REG_DIR
  └─ chip->sdir ← قراءة reg_dir_out (أو ~reg_dir_in)
```

---

### 4. مخططات تدفق العمليات (Call Flow Diagrams)

#### قراءة قيمة GPIO — `gpio_mmio_get()`

```
consumer calls: gpiod_get_value(desc)
  │
  ▼
gpiolib: gc->get(gc, gpio)
  │
  ▼
gpio_mmio_get(gc, gpio)
  ├─ to_gpio_generic_chip(gc)  → chip
  ├─ gpio_mmio_line2mask(gc, gpio)
  │     └─ be_bits ? BIT(bits-1-line) : BIT(line)
  └─ chip->read_reg(chip->reg_dat) & pinmask
        │
        └─► gpio_mmio_read32(reg)  → readl(reg)
                │
                └─► hardware MMIO register
```

#### كتابة قيمة GPIO — نمط set/clear

```
consumer calls: gpiod_set_value(desc, 1)
  │
  ▼
gpiolib: gc->set(gc, gpio, 1)
  │
  ▼
gpio_mmio_set_with_clear(gc, gpio, val=1)
  ├─ to_gpio_generic_chip(gc) → chip
  ├─ gpio_mmio_line2mask(gc, gpio) → mask
  └─ val==1: chip->write_reg(chip->reg_set, mask)
             │
             └─► gpio_mmio_write32(reg_set, mask)
                   └─► writel(mask, reg_set)  → MMIO SET register
```

#### كتابة — نمط single register (مع shadow)

```
gpio_mmio_set(gc, gpio, val)
  ├─ guard(raw_spinlock_irqsave)(&chip->lock)   ← يمسك اللوك
  ├─ val ? sdata |= mask : sdata &= ~mask        ← يعدّل الـ shadow
  └─ chip->write_reg(chip->reg_dat, chip->sdata) ← يكتب كامل الـ register
        └─► writel(sdata, reg_dat)
  └─ lock released automatically (guard)
```

#### ضبط الاتجاه output — نمط dir_first

```
consumer: gpiod_direction_output(desc, val)
  │
  ▼
gpio_mmio_dir_out_dir_first(gc, gpio, val)
  ├─ 1. gpio_mmio_dir_out(gc, gpio, val)
  │       ├─ guard(raw_spinlock_irqsave)(&chip->lock)
  │       ├─ chip->sdir |= mask
  │       ├─ reg_dir_in  ? write_reg(reg_dir_in,  ~sdir)
  │       └─ reg_dir_out ? write_reg(reg_dir_out,  sdir)
  │
  ├─ 2. gc->set(gc, gpio, val)   ← يكتب القيمة بعد الاتجاه
  │
  └─ 3. gpio_mmio_dir_return(gc, gpio, true)
         └─ pinctrl ? pinctrl_gpio_direction_output(gc, gpio) : 0
```

#### ضبط الاتجاه output — نمط val_first

```
gpio_mmio_dir_out_val_first(gc, gpio, val)
  ├─ 1. gc->set(gc, gpio, val)   ← القيمة أولاً (تجنب glitch)
  ├─ 2. gpio_mmio_dir_out(gc, gpio, val)
  └─ 3. gpio_mmio_dir_return(gc, gpio, true)
```

> **الفرق:** `GPIO_GENERIC_NO_SET_ON_INPUT` يحدد أيهما يُستخدَم — لو الـ hardware ممكن يحصل glitch لما تكتب على input pin، استخدم `dir_first`.

#### Batch write — `gpio_mmio_set_multiple_with_clear()`

```
gc->set_multiple(gc, mask, bits)
  │
  ▼
gpio_mmio_set_multiple_with_clear(gc, mask, bits)
  ├─ gpio_mmio_multiple_get_masks(gc, mask, bits, &set_mask, &clear_mask)
  │     └─ for_each_set_bit → يصنّف كل pin لـ set_mask أو clear_mask
  ├─ set_mask   ? write_reg(reg_set, set_mask)
  └─ clear_mask ? write_reg(reg_clr, clear_mask)
```

---

### 5. استراتيجية الـ Locking

#### الـ Lock المستخدَم

الـ driver بيستخدم `raw_spinlock_t lock` موجود في `struct gpio_generic_chip`.

**لماذا `raw_spinlock` وليس `spinlock` عادي؟**
- `raw_spinlock` لا يُعطَّل حتى في بيئة `PREEMPT_RT` — ضروري لأن العمليات على MMIO registers جداً سريعة.
- مناسب لـ interrupt context لأنه مع `irqsave`.

#### ما الذي يحميه اللوك

| المورد المحمي | السبب |
|--------------|-------|
| `chip->sdata` | read-modify-write على الـ shadow data register |
| `chip->sdir` | read-modify-write على الـ shadow direction register |
| MMIO write للـ `reg_dat`/`reg_set`/`reg_dir_*` | يضمن أن الـ shadow والـ hardware يُكتَبا معاً atomically |

#### متى يُمسَك اللوك

| الدالة | تمسك اللوك؟ | السبب |
|--------|------------|-------|
| `gpio_mmio_set()` | نعم (irqsave) | تعدّل sdata + تكتب reg_dat |
| `gpio_mmio_set_set()` | نعم (irqsave) | تعدّل sdata + تكتب reg_set |
| `gpio_mmio_set_multiple_single_reg()` | نعم (irqsave) | batch modify + write |
| `gpio_mmio_dir_out()` | نعم (irqsave) | تعدّل sdir + تكتب reg_dir |
| `gpio_mmio_dir_in()` | نعم (irqsave guard) | تعدّل sdir + تكتب reg_dir |
| `gpio_mmio_get()` | لا | قراءة فقط من hardware — atomic في x86/ARM |
| `gpio_mmio_set_with_clear()` | لا | كل write منفصل atomic — hardware يضمن |
| `gpio_mmio_set_multiple_with_clear()` | لا | نفس السبب |

#### ترتيب الـ Locking (Lock Ordering)

الـ driver بيمسك lock واحد فقط في المرة — مفيش nested locking:

```
chip->lock  ─────── أعلى مستوى، لا أحد يمسكه بعده
```

لو `pinctrl` مفعّل، `pinctrl_gpio_direction_*()` يُستدعى **بعد** تحرير اللوك — يمنع deadlock مع pinctrl subsystem.

#### الـ Guard API الحديث

الكيرنل بيستخدم `guard()` وسكتسكوب بدل `spin_lock_irqsave/unlock` اليدوية:

```c
/* طريقة قديمة */
unsigned long flags;
raw_spin_lock_irqsave(&chip->lock, flags);
/* ... */
raw_spin_unlock_irqrestore(&chip->lock, flags);

/* الطريقة الحديثة في الكود */
guard(raw_spinlock_irqsave)(&chip->lock);
/* اللوك يُحرَّر تلقائياً عند الخروج من الـ scope */
```

الـ header بيعرّف أيضاً `gpio_generic_lock_irqsave` guard لاستخدام خارجي من الـ drivers اللي تبني على `gpio_generic_chip`.

---

### ملخص Architecture

```
Hardware MMIO Registers
  │
  │  (readl/writel / ioread32be/iowrite32be)
  │
  ▼
gpio_mmio_read/write functions (8/16/32/64/be variants)
  │
  │  (function pointers: chip->read_reg, chip->write_reg)
  │
  ▼
struct gpio_generic_chip
  ├── sdata / sdir (shadow state, protected by raw_spinlock)
  ├── reg_* (MMIO addresses)
  └── struct gpio_chip gc
        │
        │  (gc->get, gc->set, gc->direction_*)
        │
        ▼
    GPIO Core (gpiolib)
        │
        ▼
    Consumer (kernel driver / userspace via /dev/gpiochipN)
```
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Category: Register I/O Accessors

| Function | Signature (simplified) | Purpose |
|---|---|---|
| `gpio_mmio_read8` | `(void __iomem *reg) → ulong` | Read 8-bit register |
| `gpio_mmio_write8` | `(void __iomem *reg, ulong data)` | Write 8-bit register |
| `gpio_mmio_read16` | `(void __iomem *reg) → ulong` | Read 16-bit LE register |
| `gpio_mmio_write16` | `(void __iomem *reg, ulong data)` | Write 16-bit LE register |
| `gpio_mmio_read32` | `(void __iomem *reg) → ulong` | Read 32-bit LE register |
| `gpio_mmio_write32` | `(void __iomem *reg, ulong data)` | Write 32-bit LE register |
| `gpio_mmio_read64` | `(void __iomem *reg) → ulong` | Read 64-bit register (64-bit only) |
| `gpio_mmio_write64` | `(void __iomem *reg, ulong data)` | Write 64-bit register (64-bit only) |
| `gpio_mmio_read16be` | `(void __iomem *reg) → ulong` | Read 16-bit big-endian register |
| `gpio_mmio_write16be` | `(void __iomem *reg, ulong data)` | Write 16-bit big-endian register |
| `gpio_mmio_read32be` | `(void __iomem *reg) → ulong` | Read 32-bit big-endian register |
| `gpio_mmio_write32be` | `(void __iomem *reg, ulong data)` | Write 32-bit big-endian register |

#### Category: GPIO Get (Input)

| Function | Purpose |
|---|---|
| `gpio_mmio_get` | قراءة قيمة pin واحد من reg_dat فقط |
| `gpio_mmio_get_set` | قراءة: outputs من reg_set، inputs من reg_dat |
| `gpio_mmio_get_multiple` | قراءة متعددة — native endian فقط |
| `gpio_mmio_get_multiple_be` | قراءة متعددة — big-endian mirrored bits |
| `gpio_mmio_get_set_multiple` | قراءة متعددة مع تفريق outputs/inputs |

#### Category: GPIO Set (Output)

| Function | Purpose |
|---|---|
| `gpio_mmio_set` | كتابة data register مع shadow + lock |
| `gpio_mmio_set_with_clear` | كتابة set/clr registers بشكل مستقل |
| `gpio_mmio_set_set` | كتابة set register فقط مع shadow + lock |
| `gpio_mmio_set_none` | stub — no output hardware |
| `gpio_mmio_set_multiple` | كتابة متعددة على data register |
| `gpio_mmio_set_multiple_set` | كتابة متعددة على set register |
| `gpio_mmio_set_multiple_with_clear` | كتابة متعددة على set/clr registers |

#### Category: Direction Control

| Function | Purpose |
|---|---|
| `gpio_mmio_dir_in` | تحويل line لـ input مع تحديث reg_dir |
| `gpio_mmio_dir_out` | (internal) تحديث shadow + reg_dir للـ output |
| `gpio_mmio_dir_out_dir_first` | تعيين direction أولاً ثم القيمة |
| `gpio_mmio_dir_out_val_first` | تعيين القيمة أولاً ثم direction |
| `gpio_mmio_simple_dir_in` | bidirectional GPIO بدون direction register |
| `gpio_mmio_simple_dir_out` | bidirectional GPIO بدون direction register |
| `gpio_mmio_dir_in_err` | stub — input ممنوع |
| `gpio_mmio_dir_out_err` | stub — output ممنوع |
| `gpio_mmio_get_dir` | قراءة الاتجاه الحالي لـ pin |

#### Category: Setup & Initialization

| Function | Purpose |
|---|---|
| `gpio_mmio_line2mask` | تحويل line number لـ bitmask (LE أو BE) |
| `gpio_mmio_multiple_get_masks` | حساب set_mask و clear_mask للكتابة المتعددة |
| `gpio_mmio_setup_accessors` | اختيار read/write function pointers حسب عرض السجل |
| `gpio_mmio_setup_io` | ربط callbacks الـ get/set حسب configuration |
| `gpio_mmio_setup_direction` | ربط callbacks الـ direction حسب configuration |
| `gpio_mmio_request` | تحقق من صحة pin وطلبه من pinctrl |
| `gpio_generic_chip_init` | الـ main initialization — الـ exported API |

#### Category: Platform Driver

| Function | Purpose |
|---|---|
| `gpio_mmio_map` | ioremap لـ named resource بتحقق من الحجم |
| `gpio_mmio_pdev_probe` | platform driver probe — يقرأ resources ويسجل chip |

#### Category: Inline API (في الـ header)

| Function | Purpose |
|---|---|
| `to_gpio_generic_chip` | `container_of` wrapper من `gpio_chip*` لـ `gpio_generic_chip*` |
| `gpio_generic_chip_set` | safe wrapper لاستدعاء `gc->set` |
| `gpio_generic_read_reg` | safe wrapper لاستدعاء `chip->read_reg` |
| `gpio_generic_write_reg` | safe wrapper لاستدعاء `chip->write_reg` |

---

### Group 1: Register I/O Accessors

**الغرض:** الـ driver يدعم registers بعرض 8/16/32/64 bit وبـ byte order LE أو BE. بدل ما يكون في كل مكان `if (bits == 32) readl(...)` — الـ initialization تختار function pointer واحد يُحفظ في `chip->read_reg` و `chip->write_reg`، وبعدين كل الكود يستخدمهم بشكل uniform.

---

#### `gpio_mmio_read8` / `gpio_mmio_write8`

```c
static unsigned long gpio_mmio_read8(void __iomem *reg);
static void gpio_mmio_write8(void __iomem *reg, unsigned long data);
```

**الـ read8** بتعمل `readb(reg)` وترجع القيمة كـ `unsigned long`. الـ write8 بتعمل `writeb(data, reg)`.

- **Parameters:** `reg` — الـ MMIO virtual address (من ioremap).
- **Return:** `read` ترجع قيمة الـ byte. `write` لا ترجع قيمة.
- **Key details:** الـ `readb`/`writeb` guaranteed memory barriers على كل architectures. مفيش locking هنا — الـ locking في المستوى الأعلى.

---

#### `gpio_mmio_read16be` / `gpio_mmio_write16be`

```c
static unsigned long gpio_mmio_read16be(void __iomem *reg);
static void gpio_mmio_write16be(void __iomem *reg, unsigned long data);
```

بتستخدم `ioread16be` / `iowrite16be` اللي بتعمل byte-swap على LE architectures. الفرق بين الـ `gpio_mmio_read16` الـ native و الـ BE: في BE، bit 15 في الـ hardware مربوط بـ GPIO line 0 في الـ software.

- **Key details:** الـ 64-bit BE غير مدعوم (`dev_err` + `-EINVAL` في setup_accessors).

---

#### `gpio_mmio_setup_accessors`

```c
static int gpio_mmio_setup_accessors(struct device *dev,
                                     struct gpio_generic_chip *chip,
                                     bool byte_be);
```

بتختار الـ `read_reg`/`write_reg` function pointers المناسبين حسب `chip->bits` و `byte_be`.

- **Parameters:**
  - `dev` — للـ `dev_err` لو الـ width غير مدعوم.
  - `chip` — الـ struct اللي هيتحدد فيه الـ pointers.
  - `byte_be` — true لو الـ flag `GPIO_GENERIC_BIG_ENDIAN_BYTE_ORDER` مضبوط.
- **Return:** `0` نجاح، `-EINVAL` لو الـ width غير مدعوم أو 64-bit BE.
- **Pseudo-flow:**
```
switch(chip->bits):
  8  → read8/write8
  16 → be? read16be/write16be : read16/write16
  32 → be? read32be/write32be : read32/write32
  64 → be? ERROR(-EINVAL)    : read64/write64
  *  → ERROR(-EINVAL)
```

---

### Group 2: Bit Mask Helpers

---

#### `gpio_mmio_line2mask`

```c
static unsigned long gpio_mmio_line2mask(struct gpio_chip *gc,
                                         unsigned int line);
```

بتحول رقم الـ GPIO line (0-based) لـ bitmask مناسب للـ register.

- **Parameters:** `gc` — الـ gpio_chip. `line` — رقم الـ GPIO (0..bits-1).
- **Return:** على LE: `BIT(line)` — bit 0 = line 0. على BE: `BIT(bits-1-line)` — bit 31 = line 0 في 32-bit register.
- **Why it matters:** كل عملية قراءة أو كتابة على bit بعينه تمر من هنا. الـ BE mirroring مش في الـ read/write accessor — ده هو اللي بيتعامل معاه.

---

#### `gpio_mmio_multiple_get_masks`

```c
static void gpio_mmio_multiple_get_masks(struct gpio_chip *gc,
                                         unsigned long *mask,
                                         unsigned long *bits,
                                         unsigned long *set_mask,
                                         unsigned long *clear_mask);
```

بتفصل الـ GPIO lines المطلوب تغييرها (في `mask`) لـ مجموعتين: اللي هتتعمل set (`*set_mask`) واللي هتتعمل clear (`*clear_mask`).

- **Parameters:**
  - `mask` — الـ bitmask للـ lines اللي هيتغيروا.
  - `bits` — القيم الجديدة المطلوبة.
  - `set_mask` / `clear_mask` — الناتج، hardware-translated عبر `gpio_mmio_line2mask`.
- **Key details:** بتلف على كل set bit في `mask` وبتستخدم `gpio_mmio_line2mask` فـ بتدعم BE أوتوماتيك.

---

### Group 3: GPIO Get (قراءة القيمة)

---

#### `gpio_mmio_get`

```c
static int gpio_mmio_get(struct gpio_chip *gc, unsigned int gpio);
```

بتقرأ قيمة GPIO pin واحد من الـ `reg_dat` (data/input register).

- **Parameters:** `gc` — الـ chip. `gpio` — رقم الـ line.
- **Return:** `0` أو `1`.
- **Key details:** مفيش locking — قراءة atomic على MMIO word. بتستخدم `gpio_mmio_line2mask` للـ BE compatibility. الـ `reg_dat` دايماً موجود (إجباري في `setup_io`).

---

#### `gpio_mmio_get_set`

```c
static int gpio_mmio_get_set(struct gpio_chip *gc, unsigned int gpio);
```

بتراعي إن بعض الـ hardware لما pin يكون output، قيمته الحقيقية بتتقرأ صح من `reg_set` مش من `reg_dat`. بتشوف الـ `sdir` shadow: لو الـ bit ده output (`sdir & pinmask`)، بتقرأ من `reg_set`؛ لو input، بتقرأ من `reg_dat`.

- **Return:** `0` أو `1`.
- **Assigned when:** `GPIO_GENERIC_READ_OUTPUT_REG_SET` flag مضبوط وملقيش `GPIO_GENERIC_UNREADABLE_REG_SET`.

---

#### `gpio_mmio_get_multiple`

```c
static int gpio_mmio_get_multiple(struct gpio_chip *gc,
                                  unsigned long *mask,
                                  unsigned long *bits);
```

بتقرأ قيم كذا line دفعة واحدة من register read واحد. بتعمل `*bits &= ~*mask` أولاً (تنظيف الـ bits اللي هتتغير) وبعدين `*bits |= read_reg(reg_dat) & *mask`.

- **Parameters:** `mask` — الـ lines المطلوبة. `bits` — الـ output اللي هيتحدث.
- **Return:** دايماً `0`.
- **Key details:** Native endian only — الـ `gpio_mmio_get_multiple_be` هي النسخة الـ BE.

---

#### `gpio_mmio_get_multiple_be`

```c
static int gpio_mmio_get_multiple_be(struct gpio_chip *gc,
                                     unsigned long *mask,
                                     unsigned long *bits);
```

نفس الـ `get_multiple` بس للـ big-endian mirrored bits. عشان الـ bit numbering معكوس، محتاج يبني `readmask` يدوياً بـ `gpio_mmio_line2mask` لكل bit، يقرأ الـ register، وبعدين يعكس النتيجة تاني في `*bits`.

- **Key details:** الـ complexity هنا بتبرر إن الـ `get_set_multiple` مش بتتعمل لـ BE — الـ comment في `setup_io` بيقول صراحة "too much complexity".

**Pseudo-flow:**
```
bits &= ~mask
for each bit in mask:
    readmask |= line2mask(bit)   // mirror to HW bit position
val = read_reg(reg_dat) & readmask
for each bit in val:
    bits |= line2mask(bit)       // mirror back to SW bit position
```

---

### Group 4: GPIO Set (كتابة القيمة)

---

#### `gpio_mmio_set`

```c
static int gpio_mmio_set(struct gpio_chip *gc, unsigned int gpio, int val);
```

بتكتب قيمة GPIO line واحد على الـ single data register (لما مفيش set/clr registers منفصلين).

- **Parameters:** `gc`, `gpio` — رقم الـ line, `val` — القيمة المطلوبة.
- **Return:** `0`.
- **Key details:**
  - بتاخد `raw_spinlock_irqsave` (`guard(raw_spinlock_irqsave)`) — ضروري لأن الـ read-modify-write على `chip->sdata` لازم يكون atomic.
  - بتعدل الـ shadow register `sdata` أولاً، وبعدين بتكتب الـ shadow كله على `reg_dat`. الـ shadow ضروري عشان الـ register ممكن يكون write-only أو بنكتب على bits متفرقة.
  - **Caller context:** GPIO core، يمكن من interrupt context، مكانه حليل مع `raw_spinlock_irqsave`.

---

#### `gpio_mmio_set_with_clear`

```c
static int gpio_mmio_set_with_clear(struct gpio_chip *gc,
                                    unsigned int gpio, int val);
```

للـ hardware اللي عنده `reg_set` و `reg_clr` منفصلين. بتكتب الـ bitmask في الـ register المناسب بس.

- **Key details:** **مفيش locking** — الـ hardware نفسه atomic. كل register بيتحكم في bit واحد فقط في الوقت ده، مفيش race على shadow register.
- **Side effect:** مفيش `sdata` shadow هنا — الـ hardware هو الـ source of truth.

---

#### `gpio_mmio_set_set`

```c
static int gpio_mmio_set_set(struct gpio_chip *gc, unsigned int gpio, int val);
```

للـ hardware اللي عنده `reg_set` فقط بدون `reg_clr` منفصل — الـ register بيعمل set للـ 1s وبيعمل clear للـ 0s.

- **Key details:** بتستخدم `raw_spinlock_irqsave` وبتعدل `sdata` shadow زي `gpio_mmio_set`، بس بتكتب على `reg_set` مش `reg_dat`.

---

#### `gpio_mmio_set_none`

```c
static int gpio_mmio_set_none(struct gpio_chip *gc, unsigned int gpio, int val);
```

Stub — بترجع `0` من غير ما تعمل حاجة. بتتعين لما `GPIO_GENERIC_NO_OUTPUT` مضبوط.

---

#### `gpio_mmio_set_multiple_single_reg`

```c
static void gpio_mmio_set_multiple_single_reg(struct gpio_chip *gc,
                                              unsigned long *mask,
                                              unsigned long *bits,
                                              void __iomem *reg);
```

Helper مشترك بين `gpio_mmio_set_multiple` و `gpio_mmio_set_multiple_set`. بتاخد الـ lock، بتحسب set/clear masks، بتعدل `sdata`، وبتكتب على الـ `reg` المحدد.

- **Parameters:** `reg` — الـ register اللي هيتكتب عليه (إما `reg_dat` أو `reg_set`).

---

#### `gpio_mmio_set_multiple_with_clear`

```c
static int gpio_mmio_set_multiple_with_clear(struct gpio_chip *gc,
                                             unsigned long *mask,
                                             unsigned long *bits);
```

للـ set/clr hardware: بتكتب `set_mask` على `reg_set` و `clear_mask` على `reg_clr` في calls منفصلين.

- **Key details:** مفيش locking — نفس منطق `gpio_mmio_set_with_clear`.

---

### Group 5: Direction Control

الـ driver بيدعم 3 سيناريوهات:
1. **Bidirectional بدون direction registers** — simple input/output فقط.
2. **Direction registers (dirout و/أو dirin)** — بتحدد الـ direction في الـ hardware.
3. **Input-only أو Output-only** — stubs بترجع `-EINVAL`.

---

#### `gpio_mmio_dir_return`

```c
static int gpio_mmio_dir_return(struct gpio_chip *gc, unsigned int gpio,
                                bool dir_out);
```

Helper بيعمل `pinctrl_gpio_direction_output` أو `_input` لو الـ pinctrl backend مفعّل. لو مفيش pinctrl، بترجع `0` مباشرة.

---

#### `gpio_mmio_dir_in` (الـ full version)

```c
static int gpio_mmio_dir_in(struct gpio_chip *gc, unsigned int gpio);
```

بتحوّل GPIO line لـ input لما في direction registers.

- **Key details:**
  - بتاخد `raw_spinlock_irqsave` لتعديل `sdir` shadow.
  - بتعمل clear للـ bit المناسب في `sdir`.
  - لو في `reg_dir_in`: بتكتب `~sdir` فيه (1 = input).
  - لو في `reg_dir_out`: بتكتب `sdir` فيه (0 = input).
  - بعد اللوك، بتستدعي `gpio_mmio_dir_return` للـ pinctrl.

---

#### `gpio_mmio_dir_out` (internal helper)

```c
static void gpio_mmio_dir_out(struct gpio_chip *gc, unsigned int gpio, int val);
```

Internal helper — بتعمل set للـ bit في `sdir` وبتكتب على direction registers. مش بتستدعي pinctrl ولا set — ده الجزء المشترك بين `dir_out_dir_first` و `dir_out_val_first`.

---

#### `gpio_mmio_dir_out_dir_first`

```c
static int gpio_mmio_dir_out_dir_first(struct gpio_chip *gc,
                                       unsigned int gpio, int val);
```

للـ hardware اللي لو عملت set للـ output value قبل ما تعمله output pin ممكن يسبب glitch. الترتيب: direction أولاً → قيمة تانياً → pinctrl.

---

#### `gpio_mmio_dir_out_val_first`

```c
static int gpio_mmio_dir_out_val_first(struct gpio_chip *gc,
                                       unsigned int gpio, int val);
```

للـ hardware اللي الأحسن تضبط القيمة قبل ما تعمل enable للـ output، عشان تتجنب glitch في الاتجاه التاني. الترتيب: قيمة أولاً → direction تانياً → pinctrl.

---

#### `gpio_mmio_get_dir`

```c
static int gpio_mmio_get_dir(struct gpio_chip *gc, unsigned int gpio);
```

بترجع `GPIO_LINE_DIRECTION_IN` (1) أو `GPIO_LINE_DIRECTION_OUT` (0) للـ pin المحدد.

- **Logic:**
  1. لو `dir_unreadable` — بترجع من `sdir` shadow.
  2. لو في `reg_dir_out` — بتقرأ منه مباشرة.
  3. لو في `reg_dir_in` — bit = 1 معناه input، فلو مش set يبقى output.
  4. Default: input.

- **Key details:** الـ `dir_unreadable` flag بيُضبط لما `GPIO_GENERIC_UNREADABLE_REG_DIR` موجود.

---

### Group 6: Setup & Configuration

---

#### `gpio_mmio_setup_io`

```c
static int gpio_mmio_setup_io(struct gpio_generic_chip *chip,
                              const struct gpio_generic_chip_config *cfg);
```

الـ function الأهم في الـ wiring — بتربط `gc->set`, `gc->set_multiple`, `gc->get`, `gc->get_multiple` بالـ implementations الصح حسب الـ hardware configuration.

- **Parameters:** `chip`, `cfg` — الـ config بتحدد الـ register addresses والـ flags.
- **Return:** `0` نجاح، `-EINVAL` لو `cfg->dat` is NULL (الـ dat register إجباري).

**Decision tree:**

```
/* Set path */
if cfg->set AND cfg->clr:
    → set_with_clear, set_multiple_with_clear
elif cfg->set (no clr):
    → set_set, set_multiple_set
elif NO_OUTPUT flag:
    → set_none, set_multiple = NULL
else (dat only):
    → set, set_multiple

/* Get path */
if READ_OUTPUT_REG_SET AND NOT UNREADABLE_REG_SET:
    → get_set
    if NOT be_bits → get_set_multiple
else:
    → get
    if be_bits → get_multiple_be
    else       → get_multiple
```

---

#### `gpio_mmio_setup_direction`

```c
static int gpio_mmio_setup_direction(struct gpio_generic_chip *chip,
                                     const struct gpio_generic_chip_config *cfg);
```

بتربط `gc->direction_input`, `gc->direction_output`, `gc->get_direction` حسب وجود direction registers والـ flags.

- **Return:** دايماً `0`.

**Decision tree:**

```
if cfg->dirout OR cfg->dirin:
    direction_input  → gpio_mmio_dir_in
    get_direction    → gpio_mmio_get_dir
    if NO_SET_ON_INPUT:
        direction_output → gpio_mmio_dir_out_dir_first
    else:
        direction_output → gpio_mmio_dir_out_val_first
else (no direction regs):
    if NO_OUTPUT: direction_output → err
    else:         direction_output → simple_dir_out
    if NO_INPUT:  direction_input  → err
    else:         direction_input  → simple_dir_in
```

---

#### `gpio_mmio_request`

```c
static int gpio_mmio_request(struct gpio_chip *gc, unsigned int gpio_pin);
```

Callback بيتستدعى لما GPIO consumer يطلب pin. بتتحقق إن `gpio_pin < ngpio`، وبعدين لو في pinctrl backend بتستدعي `gpiochip_generic_request` للـ muxing.

- **Return:** `0` نجاح، `-EINVAL` لو الـ pin out of range.

---

#### `gpio_generic_chip_init` ⭐

```c
int gpio_generic_chip_init(struct gpio_generic_chip *chip,
                           const struct gpio_generic_chip_config *cfg);
```

الـ **exported** API الوحيد في الملف — الـ entry point لأي driver خارجي يريد استخدام الـ generic MMIO GPIO infrastructure.

- **Parameters:**
  - `chip` — الـ `gpio_generic_chip` struct اللي المتصل allocate-اه.
  - `cfg` — الـ configuration: device، register addresses، sizes، flags.
- **Return:** `0` نجاح، negative error code عند الفشل.

**Key details:**
- بتتحقق إن `cfg->sz` power of 2 (8/16/32/64 bit register).
- بتحسب `chip->bits = cfg->sz * 8`.
- بتعمل `raw_spin_lock_init` على `chip->lock`.
- الـ `gc->base = -1` يخلي الـ GPIO core يعين الـ base number أوتوماتيك.
- `gc->request = gpio_mmio_request` — دايماً مضبوط.
- بتستدعي `gpiochip_get_ngpios` من الـ firmware (DT/ACPI)، وبترجع للـ `chip->bits` كـ fallback.
- لو `GPIO_GENERIC_PINCTRL_BACKEND`: `chip->pinctrl = true` و `gc->free = gpiochip_generic_free`.
- بتقرأ initial state من hardware لـ `sdata` و `sdir` — مهم لصحة الـ shadow registers من البداية.
- لو في `reg_dir_out` و `reg_dir_in` معاً، بتعمل sync: `write(reg_dir_in, ~sdir)` عشان hardware الاتنين يتطابقوا.

**Pseudo-flow:**
```c
validate sz (power_of_2 && <= BITS_PER_LONG)
chip->bits = sz * 8
raw_spin_lock_init(&chip->lock)
setup gc basics (parent, label, base=-1, request)
chip->be_bits = !!(flags & BIG_ENDIAN)
get_ngpios from firmware or fallback to bits

gpio_mmio_setup_io(chip, cfg)       // wires get/set
gpio_mmio_setup_accessors(...)      // wires read_reg/write_reg
gpio_mmio_setup_direction(chip, cfg) // wires direction

if PINCTRL_BACKEND: set pinctrl=true, gc->free

// Load initial shadow state
chip->sdata = read_reg(reg_dat)
if set_set && !UNREADABLE_REG_SET: chip->sdata = read_reg(reg_set)

if UNREADABLE_REG_DIR: dir_unreadable = true

// Load initial direction shadow
if dir regs && !UNREADABLE_REG_DIR:
    sdir = read(reg_dir_out) or ~read(reg_dir_in)
    if both regs: sync reg_dir_in = ~sdir
```

---

### Group 7: Platform Driver

---

#### `gpio_mmio_map`

```c
static void __iomem *gpio_mmio_map(struct platform_device *pdev,
                                   const char *name, resource_size_t sane_sz);
```

Helper بتعمل ioremap لـ named memory resource. لو الـ resource مش موجود بترجع `NULL` (مش error). لو موجود بس حجمه غلط بترجع `IOMEM_ERR_PTR(-EINVAL)`.

- **Parameters:** `name` — اسم الـ resource زي `"dat"`, `"set"`, `"clr"`. `sane_sz` — الحجم المتوقع.
- **Return:** virtual address، `NULL` لو مش موجود، أو `IOMEM_ERR_PTR` عند error.
- **Key details:** بتستخدم `devm_ioremap_resource` فـ الـ unmap بيحصل أوتوماتيك عند remove.

---

#### `gpio_mmio_pdev_probe`

```c
static int gpio_mmio_pdev_probe(struct platform_device *pdev);
```

الـ platform driver probe. بتقرأ الـ memory resources من الـ platform device، بتعمل ioremap لكل register، وبعدين بتسجل الـ GPIO chip.

- **Return:** `0` نجاح، negative error code.

**Supported platforms (of_match_table):**
- `brcm,bcm6345-gpio`
- `wd,mbl-gpio`
- `ni,169445-nand-gpio`
- `intel,ixp4xx-expansion-bus-mmio-gpio`
- `opencores,gpio`

**Device properties:**
- `no-output` → `GPIO_GENERIC_NO_OUTPUT`
- `label` → تجاوز الـ default label
- `gpio-mmio,base` → تحديد الـ base GPIO number يدوياً (للـ board files فقط، مش DT)

**Key details:**
- الـ `dat` resource إجباري — بترجع `-EINVAL` فوراً لو مش موجود.
- باقي الـ resources اختيارية — `NULL` معناها مش موجود، `IS_ERR` معناها error حقيقي.
- `devm_gpiochip_add_data` بيعمل register + auto-unregister عند remove.
- `platform_set_drvdata(pdev, &gen_gc->gc)` بيحفظ الـ pointer للـ suspend/resume لو احتاجه أي driver تاني.

**Probe flow:**
```
1. get "dat" resource → sz
2. map: dat, set, clr, dirout, dirin (all with sz)
3. alloc gen_gc (devm_kzalloc)
4. set BIG_ENDIAN_BYTE_ORDER if device_is_big_endian()
5. set NO_OUTPUT if "no-output" property
6. gpio_generic_chip_init(gen_gc, &config)
7. override label if "label" property
8. override base if "gpio-mmio,base" property
9. devm_gpiochip_add_data(...)
```

---

### الـ Locking Strategy — ملخص

| Scenario | Lock Used | Why |
|---|---|---|
| Single pin write (dat/set) | `raw_spinlock_irqsave` | RMW على shadow `sdata` |
| Multiple pins write (dat/set) | `raw_spinlock_irqsave` | RMW على shadow `sdata` |
| set/clr hardware | لا يوجد | كل register atomic بطبيعته |
| direction change | `raw_spinlock_irqsave` | RMW على shadow `sdir` |
| pin read | لا يوجد | MMIO read atomic على aligned address |

الـ `raw_spinlock_t` (مش `spinlock_t`) مستخدم عشان الـ GPIO set ممكن يتستدعى من realtime context أو من داخل IRQ handler.

---

### الـ Shadow Registers — ليه؟

كتير من الـ FPGA/ASIC GPIO implementations عندها registers write-only: لو قرأت منها الـ value مش محدد. الـ driver بيحتفظ بـ:

- **`sdata`** — نسخة من القيمة الحالية للـ output register. لما بنغير bit واحد، بنعدل `sdata` وبنكتب الـ `sdata` كله على الـ hardware.
- **`sdir`** — نسخة من direction register. نفس المنطق.

ده بيضمن إن الـ read-modify-write آمن حتى على write-only registers.
## Phase 5: دليل الـ Debugging الشامل

الـ driver اللي بنتكلم عنه هو `gpio-mmio.c` — الـ generic MMIO GPIO driver اللي بيدعم registers بعرض 8/16/32/64-bit وبيتعامل مع set/clear/direction registers بشكل مرن. الـ debugging بتاعه بيتوزع على مستوى الـ software (kernel state) والـ hardware (register state).

---

### Software Level

#### 1. مدخلات الـ debugfs

الـ GPIO subsystem بيعمل expose لحالة كل chip في `/sys/kernel/debug/gpio`:

```bash
# اقرأ حالة كل GPIO chips في النظام
cat /sys/kernel/debug/gpio

# مثال على الـ output:
# gpiochip0: GPIOs 496-511, parent: platform/10000000.gpio, 10000000.gpio:
#  gpio-496 (                    ) in  hi IRQ ACTIVE LOW
#  gpio-497 (                    ) out lo
#  gpio-498 (reset               ) out hi
```

**تفسير الـ output:**
- `in` / `out` → الاتجاه الحالي (مأخوذ من `chip->sdir` أو قراءة `reg_dir_out`)
- `hi` / `lo` → القيمة الحالية
- `IRQ` → الـ line مربوطة بـ interrupt
- `ACTIVE LOW` → الـ polarity معكوسة

```bash
# تتبع chip معينة
grep -A 20 "gpiochip0" /sys/kernel/debug/gpio

# شوف كل الـ consumers (من يستخدم الـ GPIO)
cat /sys/kernel/debug/gpio | grep -v "^\s*$"
```

#### 2. مدخلات الـ sysfs

```bash
# قائمة كل الـ gpiochips
ls /sys/class/gpio/

# مثال:
# gpiochip0  gpiochip32  export  unexport

# معلومات chip معينة
cat /sys/class/gpio/gpiochip0/label      # اسم الـ chip
cat /sys/class/gpio/gpiochip0/base       # أول رقم GPIO
cat /sys/class/gpio/gpiochip0/ngpio      # عدد الـ GPIOs

# export GPIO للتجربة اليدوية (legacy interface)
echo 496 > /sys/class/gpio/export
cat /sys/class/gpio/gpio496/direction    # in أو out
cat /sys/class/gpio/gpio496/value        # 0 أو 1

# غير الاتجاه والقيمة
echo out > /sys/class/gpio/gpio496/direction
echo 1   > /sys/class/gpio/gpio496/value

# cleanup
echo 496 > /sys/class/gpio/unexport
```

**ملاحظة مهمة:** الـ sysfs GPIO interface قديمة (deprecated منذ 4.8)، استخدمها بس للـ debugging السريع.

#### 3. الـ ftrace — tracepoints وأحداث الـ GPIO

```bash
# شوف كل الـ tracepoints المتعلقة بـ GPIO
grep -r "gpio" /sys/kernel/tracing/available_events

# الأحداث الأهم:
# gpio:gpio_direction
# gpio:gpio_value

# فعّل الـ tracing
cd /sys/kernel/tracing
echo 1 > tracing_on

# تتبع تغييرات الـ direction
echo "gpio:gpio_direction" > set_event
echo 1 > events/gpio/gpio_direction/enable

# تتبع تغييرات الـ value
echo "gpio:gpio_value" >> set_event
echo 1 > events/gpio/gpio_value/enable

# شغّل الاختبار ثم اقرأ النتيجة
cat trace

# مثال على الـ output:
# <...>-1234  [000] ....  123.456: gpio_direction: gpio=496 direction=out
# <...>-1234  [000] ....  123.457: gpio_value: gpio=496 is_irq=0 value=1
```

```bash
# تتبع الـ function calls داخل الـ driver
echo function > current_tracer
echo "gpio_mmio*" > set_ftrace_filter
echo 1 > tracing_on
cat trace
echo nop > current_tracer  # reset
```

#### 4. الـ printk والـ dynamic debug

الـ driver بيستخدم `dev_err()` للأخطاء الحرجة فقط. لتفعيل رسائل أكثر:

```bash
# فعّل dynamic debug لكل رسائل gpio-mmio
echo "file gpio-mmio.c +p" > /sys/kernel/debug/dynamic_debug/control

# أو لكل الـ GPIO subsystem
echo "module gpio_generic +p" > /sys/kernel/debug/dynamic_debug/control

# تحقق من الرسائل المفعّلة
cat /sys/kernel/debug/dynamic_debug/control | grep gpio-mmio

# تابع الـ kernel log
dmesg -w | grep -i gpio
journalctl -k -f | grep -i gpio
```

```bash
# رفع مستوى الـ loglevel مؤقتاً
echo 8 > /proc/sys/kernel/printk

# أو عبر kernel command line:
# gpio-mmio.dyndbg=+p loglevel=8
```

#### 5. خيارات الـ Kernel Config للـ Debugging

| Config Option | الوظيفة |
|---|---|
| `CONFIG_GPIO_GENERIC` | تفعيل الـ driver الأساسي |
| `CONFIG_GPIO_GENERIC_PLATFORM` | دعم platform device |
| `CONFIG_DEBUG_GPIO` | تفعيل فحوصات إضافية في GPIO core |
| `CONFIG_GPIO_SYSFS` | الـ sysfs interface للـ debugging |
| `CONFIG_GPIOLIB_FASTPATH_LIMIT` | ضبط عتبة الـ fast path |
| `CONFIG_PROVE_LOCKING` | كشف مشاكل الـ locking (lockdep) |
| `CONFIG_DEBUG_SPINLOCK` | تتبع مشاكل `raw_spinlock_t` |
| `CONFIG_LOCK_STAT` | إحصائيات الـ contention على `chip->lock` |
| `CONFIG_DYNAMIC_DEBUG` | تفعيل `dev_dbg()` / `pr_debug()` |
| `CONFIG_TRACING` | دعم ftrace tracepoints |
| `CONFIG_GPIO_CDEV` | الـ character device interface (حديث) |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "CONFIG_GPIO|CONFIG_DEBUG_GPIO"
# أو
grep -E "GPIO|DEBUG_GPIO" /boot/config-$(uname -r)
```

#### 6. أدوات خاصة بالـ subsystem — gpiotools

```bash
# تثبيت libgpiod tools
apt install gpiod   # Debian/Ubuntu
dnf install libgpiod-utils  # Fedora

# قائمة كل الـ chips
gpiodetect

# مثال:
# gpiochip0 [10000000.gpio] (16 lines)

# تفاصيل chip معينة
gpioinfo gpiochip0

# مثال:
# gpiochip0 - 16 lines:
#   line   0:      unnamed       unused   input  active-high
#   line   1:      "reset"         "kernel" output active-high [used]

# اقرأ قيمة line معينة
gpioget gpiochip0 0

# غير قيمة line
gpioset gpiochip0 0=1

# راقب تغييرات line (للـ interrupt testing)
gpiomon gpiochip0 0

# احصل على line لفترة محددة
gpioset --mode=time --sec=1 gpiochip0 0=1
```

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الخطأ | المعنى | الحل |
|---|---|---|
| `"64 bit big endian byte order unsupported"` | حاول تستخدم `sz=8` مع `GPIO_GENERIC_BIG_ENDIAN_BYTE_ORDER` | استخدم little-endian أو قلل الـ register width |
| `"unsupported data width %u bits"` | الـ `cfg->sz` مش power-of-2 أو قيمة غريبة | تأكد `sz` = 1, 2, 4, أو 8 |
| `"could not get dat resource"` | مفيش MMIO resource اسمها `"dat"` | افحص الـ Device Tree — لازم يكون `reg-names = "dat"` |
| `WARN_ON(!chip->gc.set)` | الـ chip اتعمل setup كـ no-output ثم حاول set | افحص الـ flag `GPIO_GENERIC_NO_OUTPUT` |
| `WARN_ON(!chip->read_reg)` | الـ read_reg لم يُعيَّن | الـ `gpio_generic_chip_init()` لم تكتمل بنجاح |
| `-EINVAL` من `gpio_generic_chip_init` | الـ `cfg->sz` مش power-of-2، أو bits > BITS_PER_LONG، أو `cfg->dat == NULL` | افحص الـ config المُمررة |
| `-EINVAL` من `gpio_mmio_request` | الـ gpio_pin >= ngpio | الـ DT أو الـ code بيطلب pin خارج النطاق |
| `gpiochip_add: gpios X..Y (name) failed to register` | تعارض في الـ GPIO number range | استخدم `gc->base = -1` للـ dynamic allocation |

#### 8. نقاط استراتيجية لـ dump_stack() وWARN_ON()

```c
/* في gpio_generic_chip_init() — تحقق من صحة الـ config */
WARN_ON(!cfg->dat);
WARN_ON(!is_power_of_2(cfg->sz));

/* في gpio_mmio_set() — تأكد الـ lock مأخوذ صح */
lockdep_assert_held(&chip->lock);

/* في gpio_mmio_get_dir() — تحقق من الاتساق */
WARN_ON(chip->dir_unreadable &&
        chip->reg_dir_out &&
        (chip->read_reg(chip->reg_dir_out) != chip->sdir));

/* في gpio_mmio_setup_io() — تأكد reg_dat موجود */
if (WARN_ON(!chip->reg_dat))
    return -EINVAL;

/* لتتبع من يغير الـ direction */
static int gpio_mmio_dir_in(struct gpio_chip *gc, unsigned int gpio)
{
    /* أضف هنا لو عندك مشكلة غير متوقعة */
    if (unlikely(gpio >= gc->ngpio)) {
        dump_stack();
        return -EINVAL;
    }
    /* ... */
}
```

---

### Hardware Level

#### 1. التحقق من أن حالة الـ Hardware تطابق حالة الـ Kernel

الـ driver بيحتفظ بـ **shadow registers** (`sdata` لـ data، `sdir` لـ direction). المشكلة تحدث لو تغير الـ hardware بشكل مستقل.

```bash
# الخطوة 1: اقرأ القيمة من الـ kernel
cat /sys/kernel/debug/gpio | grep "gpiochip0" -A 20

# الخطوة 2: اقرأ الـ register مباشرة من الـ hardware
# افترض الـ base address = 0x10000000
devmem2 0x10000000 b   # قرائة 8-bit
devmem2 0x10000000 w   # قرائة 16-bit
devmem2 0x10000000 l   # قرائة 32-bit

# الخطوة 3: قارن:
# kernel يقول line 0 = output high
# hardware register bit 0 = 1 ؟
# لو مختلف → مشكلة في shadow register أو race condition
```

**مقارنة `sdata` مع الـ hardware:**
```
kernel shadow:  sdata = 0b00000101  (line 0 و 2 = high)
hardware read:  reg_dat = 0b00000001  (line 0 فقط = high)
→ line 2 عندها مشكلة: shadow يقول high لكن الـ hardware ما اتغيرش
```

#### 2. تقنيات قراءة الـ Registers مباشرة

```bash
# تثبيت devmem2
apt install devmem2
# أو compile من source: https://github.com/hackndev/devmem2

# قراءة register واحد
devmem2 <physical_address> [b|h|w|l]
# b = byte (8-bit), h = half-word (16-bit), w = word (32-bit), l = long (64-bit)

# مثال: MMIO GPIO على Raspberry Pi (BCM2835)
# Base = 0x20200000
devmem2 0x20200000 l   # GPFSEL0 - function select
devmem2 0x20200034 l   # GPLEV0  - pin levels (dat register)
devmem2 0x2020001C l   # GPSET0  - set register
devmem2 0x20200028 l   # GPCLR0  - clear register

# كتابة قيمة
devmem2 0x20200028 l 0x4   # clear bit 2 (GPIO2)
```

```bash
# بديل باستخدام /dev/mem مباشرة (احتياج root)
python3 - << 'EOF'
import mmap, struct, os

GPIO_BASE = 0x10000000
PAGE_SIZE = 4096

with open('/dev/mem', 'r+b') as f:
    mem = mmap.mmap(f.fileno(), PAGE_SIZE,
                    mmap.MAP_SHARED,
                    mmap.PROT_READ | mmap.PROT_WRITE,
                    offset=GPIO_BASE)
    # قراءة dat register (offset 0)
    val = struct.unpack('<I', mem[0:4])[0]
    print(f"DAT register: 0x{val:08x} = {val:032b}")
    mem.close()
EOF
```

```bash
# استخدام io utility (من busybox أو standalone)
io -4 -r 0x10000000    # قراءة 32-bit
io -4 -w 0x10000000 0x00000001  # كتابة
```

#### 3. نصائح الـ Logic Analyzer والـ Oscilloscope

**Logic Analyzer (مثل Saleae Logic، Sigrok):**

```
الإعداد المثالي لـ MMIO GPIO debugging:
┌─────────────────────────────────────────────────┐
│  Channel 0 → GPIO line 0 (الـ pin المشبوه)       │
│  Channel 1 → GPIO line 1                          │
│  Channel 7 → CLK أو SYNC signal (للتوقيت)        │
│                                                   │
│  Sample Rate: 10x أسرع من أسرع تغيير متوقع       │
│  Trigger: Rising edge على الـ channel المشبوه     │
└─────────────────────────────────────────────────┘
```

```bash
# استخدام sigrok-cli للتقاط بياناتي
sigrok-cli --driver fx2lafw \
           --config samplerate=1MHz \
           --channels D0,D1,D2,D3 \
           --samples 100000 \
           --output-file gpio_capture.sr

# تحليل الـ capture
pulseview gpio_capture.sr &
```

**نصائح الـ Oscilloscope:**

- افحص الـ rise time والـ fall time — لو بطيئة جداً → مشكلة في الـ pull-up/down أو الـ drive strength.
- افحص الـ voltage levels: logic-high لازم > 0.7 × Vcc، logic-low < 0.3 × Vcc.
- ابحث عن glitches عند تغيير الـ direction (output → input أو العكس).
- افحص الـ shoot-through إذا عندك set/clear registers منفصلة.

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في الـ Kernel Log

| المشكلة | النمط في الـ dmesg / kernel log |
|---|---|
| عدم توافق الـ register width مع الـ hardware | `"unsupported data width 24 bits"` عند الـ probe |
| الـ dat register address غلط | الـ pin values دايماً `0` أو `0xFF` بغض النظر عن الـ hardware |
| الـ direction register معكوس (dirout vs dirin confusion) | الـ pin يعمل عكس المتوقع — `in` لما المفروض `out` |
| مشكلة الـ big-endian | الـ pins تعمل بترتيب معكوس (GPIO 0 يتصرف كـ GPIO 31) |
| race condition على `chip->lock` | `BUG: spinlock lockup` أو قيم عشوائية متقطعة |
| الـ hardware reset يمسح الـ direction register | بعد reset مفاجئ، كل الـ pins ترجع input فجأة |
| الـ set register unreadable (لكن مش مُعلَّم) | `chip->sdata` غير متزامن مع الـ hardware بعد reboot |

```bash
# راقب الـ kernel log مباشرة عند probe
dmesg -w &
modprobe gpio-generic  # أو أعد تشغيل الـ board
# ابحث عن:
# "gpio-generic: probing" — بدأ الـ probe
# أي "-EINVAL" أو "-ENOMEM" → فشل
```

#### 5. تفعيل وتشخيص الـ Device Tree

```bash
# تحقق من الـ DT الـ compiled المحمّل حالياً
dtc -I fs /proc/device-tree 2>/dev/null | grep -A 30 "gpio@"

# أو قرأ الـ raw DT
ls /proc/device-tree/
cat /proc/device-tree/soc/gpio@10000000/compatible | tr '\0' '\n'
cat /proc/device-tree/soc/gpio@10000000/reg | hexdump -C
cat /proc/device-tree/soc/gpio@10000000/reg-names | tr '\0' '\n'
```

**مثال DT صح لـ gpio-mmio:**
```dts
gpio0: gpio@10000000 {
    compatible = "opencores,gpio";  /* مطابق of_match_table */
    reg = <0x10000000 0x4>,         /* dat register */
          <0x10000004 0x4>,         /* set register */
          <0x10000008 0x4>,         /* clr register */
          <0x1000000C 0x4>,         /* dirout register */
          <0x10000010 0x4>;         /* dirin register */
    reg-names = "dat", "set", "clr", "dirout", "dirin";
    #gpio-cells = <2>;
    gpio-controller;
    ngpios = <16>;  /* عدد الـ GPIOs */
};
```

```bash
# تحقق من reg-names — ده الأهم
cat /proc/device-tree/soc/gpio@10000000/reg-names | tr '\0' '\n'
# لازم تظهر: dat
# اختيارية: set, clr, dirout, dirin

# تحقق من الـ compatible string
cat /proc/device-tree/soc/gpio@10000000/compatible | tr '\0' '\n'
# لازم تكون موجودة في gpio_mmio_of_match في gpio-mmio.c:
# "brcm,bcm6345-gpio" أو "wd,mbl-gpio" أو "opencores,gpio" إلخ

# افحص حجم الـ register (sz)
cat /proc/device-tree/soc/gpio@10000000/reg | hexdump -C
# الـ size field (bytes 4-7 في كل entry) لازم يكون 1، 2، 4، أو 8

# فحص خاصية no-output
cat /proc/device-tree/soc/gpio@10000000/no-output 2>/dev/null && echo "no-output set"
```

**تحقق من تطابق الـ DT مع الـ hardware:**

```
DT يقول: reg = <0x10000000 0x4>  →  sz = 4 bytes  →  bits = 32
Hardware: الـ GPIO controller عنده 16-bit register

→ المشكلة: الـ driver هيقرأ 32-bit بس الـ hardware 16-bit
→ الحل: غير reg إلى <0x10000000 0x2>
```

---

### Practical Commands

#### الـ commands الجاهزة للنسخ

```bash
#!/bin/bash
# ===== GPIO MMIO Quick Debug Script =====

GPIO_CHIP="gpiochip0"
GPIO_BASE_ADDR="0x10000000"  # غير حسب الـ hardware
REG_WIDTH="l"                # l=32bit, h=16bit, b=8bit

echo "=== 1. Kernel GPIO State ==="
cat /sys/kernel/debug/gpio

echo ""
echo "=== 2. Chip Info ==="
cat /sys/class/gpio/${GPIO_CHIP}/label
cat /sys/class/gpio/${GPIO_CHIP}/base
cat /sys/class/gpio/${GPIO_CHIP}/ngpio

echo ""
echo "=== 3. Hardware Register Read ==="
devmem2 ${GPIO_BASE_ADDR} ${REG_WIDTH}        # dat
devmem2 $((GPIO_BASE_ADDR + 4)) ${REG_WIDTH}  # set (لو موجود)
devmem2 $((GPIO_BASE_ADDR + 8)) ${REG_WIDTH}  # clr (لو موجود)

echo ""
echo "=== 4. Device Tree Check ==="
DT_PATH=$(find /proc/device-tree -name "reg-names" | xargs grep -l "dat" 2>/dev/null | head -1 | xargs dirname)
if [ -n "$DT_PATH" ]; then
    echo "Found GPIO DT node: $DT_PATH"
    cat "${DT_PATH}/compatible" | tr '\0' '\n'
    cat "${DT_PATH}/reg-names" | tr '\0' '\n'
fi

echo ""
echo "=== 5. Recent GPIO Errors ==="
dmesg | grep -iE "gpio|basic-mmio" | tail -20

echo ""
echo "=== 6. GPIO Lines via gpiodetect ==="
gpiodetect 2>/dev/null || echo "gpiodetect not available (install gpiod)"
gpioinfo ${GPIO_CHIP} 2>/dev/null
```

#### تفعيل الـ ftrace بشكل كامل

```bash
#!/bin/bash
# ===== Enable GPIO ftrace =====

TRACEFS="/sys/kernel/tracing"

# تنظيف
echo nop > ${TRACEFS}/current_tracer
echo > ${TRACEFS}/trace

# فعّل GPIO events
echo 1 > ${TRACEFS}/events/gpio/gpio_direction/enable
echo 1 > ${TRACEFS}/events/gpio/gpio_value/enable

# فعّل function tracing للـ driver
echo function > ${TRACEFS}/current_tracer
echo "gpio_mmio*:gpio_generic*" > ${TRACEFS}/set_ftrace_filter

echo 1 > ${TRACEFS}/tracing_on
echo "Tracing enabled. Run your test now..."
sleep 5
echo 0 > ${TRACEFS}/tracing_on

echo "=== Trace Results ==="
cat ${TRACEFS}/trace | grep -v "^#" | head -50

# cleanup
echo nop > ${TRACEFS}/current_tracer
echo > ${TRACEFS}/set_ftrace_filter
echo > ${TRACEFS}/events/gpio/gpio_direction/enable
echo > ${TRACEFS}/events/gpio/gpio_value/enable
```

#### قراءة وتفسير shadow registers

```bash
# استخدم crash tool أو gdb على vmcore لقراءة chip->sdata وchip->sdir
# أو أضف temporary printk في الـ driver:

# في gpio_mmio_set():
pr_debug("gpio%d: set val=%d, sdata before=0x%lx after=0x%lx\n",
         gpio, val,
         chip->sdata & ~mask,
         val ? chip->sdata | mask : chip->sdata & ~mask);

# فعّل dynamic debug لتشغيل هذه الرسائل:
echo "func gpio_mmio_set +p" > /sys/kernel/debug/dynamic_debug/control
dmesg -w | grep "gpio_mmio_set"
```

**مثال على تفسير الـ output:**

```
# gpio_direction event:
# task-pid [cpu] timestamp: gpio_direction: gpio=498 direction=out
#   → GPIO 498 تم تغييره لـ output

# gpio_value event:
# task-pid [cpu] timestamp: gpio_value: gpio=498 is_irq=0 value=1
#   → GPIO 498 قيمته صارت 1 (high)
#   → is_irq=0 يعني التغيير جاء من context عادي مش interrupt

# لو رأيت نفس الـ gpio يتغير مرات كثيرة بسرعة:
# → احتمال race condition أو consumer يعمل polling مكثف
# → افحص chip->lock contention بـ CONFIG_LOCK_STAT
```

#### فحص الـ lock contention

```bash
# لو عندك CONFIG_LOCK_STAT مفعّل
cat /proc/lock_stat | grep -A 5 "gpio"

# مثال output:
# class name          con-bounces  contentions  waittime-min  waittime-max  waittime-total  acq-bounces
# gpio_generic_lock:  0            0            0.00          0.00          0.00            1234
# → con-bounces > 0 يعني في threads متعددة بتتنافس على الـ lock
# → علامة على احتياج للـ optimization أو مشكلة في الـ design
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Industrial Gateway على Allwinner H616 — GPIO يكتب على register غلط

#### العنوان
**الـ FPGA GPIO expander على H616 بيكتب على `reg_dat` وفيه `set/clr` registers**

#### السياق
Industrial gateway بيشتغل على Allwinner H616 (ARM Cortex-A53). الـ FPGA expander على الـ board بيوفر 32 GPIO line عبر MMIO. الـ FPGA عنده ثلاث registers:
- `0x01C20800` → `dat` (read-only للقراءة الفعلية للـ pins)
- `0x01C20804` → `set` (write: بيطلع الـ pin high)
- `0x01C20808` → `clr` (write: بيطلع الـ pin low)

#### المشكلة
الـ DT node مكتوبه غلط — فيه resource واحد بس اسمه `dat`، مفيش `set` ولا `clr`. النتيجة إن الـ driver بيستخدم `gpio_mmio_set` بدل `gpio_mmio_set_with_clear`:

```c
/* gpio_mmio_setup_io() بيقع هنا */
} else {
    gc->set = gpio_mmio_set;          /* RMW على reg_dat */
    gc->set_multiple = gpio_mmio_set_multiple;
}
```

الـ `gpio_mmio_set` بيعمل read-modify-write على `reg_dat` وهو read-only في الـ FPGA، فالـ `readl` بيرجع garbage، والـ `writel` بيوصل للـ hardware بقيمة خاطئة. نص الـ GPIOs اللي المفروض تكون high بتبقى low.

#### التحليل
```c
static int gpio_mmio_set(struct gpio_chip *gc, unsigned int gpio, int val)
{
    struct gpio_generic_chip *chip = to_gpio_generic_chip(gc);
    unsigned long mask = gpio_mmio_line2mask(gc, gpio);

    guard(raw_spinlock_irqsave)(&chip->lock);

    if (val)
        chip->sdata |= mask;
    else
        chip->sdata &= ~mask;

    /* هنا chip->reg_dat هو نفس الـ read-only register في الـ FPGA */
    chip->write_reg(chip->reg_dat, chip->sdata);
    return 0;
}
```

`chip->sdata` اتحسب في `gpio_generic_chip_init()` بقراءة `reg_dat`:
```c
chip->sdata = chip->read_reg(chip->reg_dat);
```
لو `reg_dat` read-only وبيرجع `0xFFFFFFFF` كـ garbage، كل الـ `sdata` هتبقى `0xFFFFFFFF` من البداية.

#### الحل
تصحيح الـ DT node عشان يتعرف على الـ `set` و `clr` registers:

```dts
fpga_gpio: gpio@1c20800 {
    compatible = "opencores,gpio";
    reg = <0x01C20800 0x4>,  /* dat */
          <0x01C20804 0x4>,  /* set */
          <0x01C20808 0x4>;  /* clr */
    reg-names = "dat", "set", "clr";
    #gpio-cells = <2>;
    gpio-controller;
    ngpios = <32>;
};
```

بعد الـ fix، `gpio_mmio_setup_io()` هتشوف `cfg->set && cfg->clr` وتختار:
```c
if (cfg->set && cfg->clr) {
    chip->reg_set = cfg->set;
    chip->reg_clr = cfg->clr;
    gc->set = gpio_mmio_set_with_clear;   /* الصح */
    gc->set_multiple = gpio_mmio_set_multiple_with_clear;
}
```

Debug سريع قبل الـ fix:
```bash
# تحقق من الـ function pointers المختارة
cat /sys/kernel/debug/gpio
# شوف إيه الـ chip label والـ ngpio

# قارن القراءة من reg_dat بالـ expected
devmem2 0x01C20800 w
```

#### الدرس المستفاد
لما تكون عندك FPGA GPIO بـ `set/clr` registers، **لازم** تحدد الـ resources الثلاثة في الـ DT بالأسماء الصح (`dat`, `set`, `clr`). لو حذفتهم، الـ driver هيعمل RMW على الـ `dat` register وهو قد يكون read-only أو write-only، وده بيودي لـ data corruption صامت.

---

### السيناريو الثاني: Android TV Box على RK3562 — GPIO direction يُقرأ معكوس

#### العنوان
**الـ `reg_dir_in` بدل `reg_dir_out` على RK3562 بيخلي كل الـ output pins تُعامل كـ input**

#### السياق
Android TV box بيستخدم RK3562. الـ engineer بيضيف GPIO expander خارجي (custom ASIC) عبر MMIO. الـ ASIC عنده register واحد للـ direction، لكنه `DIR_IN` register (1 = input). الـ engineer بالغلط حطه كـ `dirout` في الـ DT.

#### المشكلة
HDMI hot-plug detect pin بيتقرأ دايماً كـ "output" رغم إنه configured كـ input. الـ `gpio_mmio_get_dir()` بيرجع نتيجة غلط:

```c
static int gpio_mmio_get_dir(struct gpio_chip *gc, unsigned int gpio)
{
    struct gpio_generic_chip *chip = to_gpio_generic_chip(gc);

    if (chip->reg_dir_out) {
        /* بيقرأ الـ DIR_IN register ويفسره كـ DIR_OUT */
        if (chip->read_reg(chip->reg_dir_out) & gpio_mmio_line2mask(gc, gpio))
            return GPIO_LINE_DIRECTION_OUT;  /* الـ bit=1 معناه input، مش output! */
        return GPIO_LINE_DIRECTION_IN;
    }
    ...
}
```

#### التحليل
لما `gpio_generic_chip_init()` بيحسب `chip->sdir`:

```c
if (chip->reg_dir_out)
    chip->sdir = chip->read_reg(chip->reg_dir_out);
```

لو الـ register اللي اتقرأ هو `DIR_IN` فعلياً، وكانت قيمته `0xFF` (كل الـ pins inputs)، بقى `chip->sdir = 0xFF` — يعني الـ kernel فاكر كل الـ pins outputs!

لما الـ `gpio_mmio_dir_in()` بتتنادى:
```c
static int gpio_mmio_dir_in(struct gpio_chip *gc, unsigned int gpio)
{
    struct gpio_generic_chip *chip = to_gpio_generic_chip(gc);

    scoped_guard(raw_spinlock_irqsave, &chip->lock) {
        chip->sdir &= ~gpio_mmio_line2mask(gc, gpio);

        if (chip->reg_dir_in)
            chip->write_reg(chip->reg_dir_in, ~chip->sdir);
        if (chip->reg_dir_out)
            chip->write_reg(chip->reg_dir_out, chip->sdir);  /* بيكتب ~DIR_IN في DIR_IN! */
    }
    ...
}
```

النتيجة: بيكتب `~chip->sdir` في الـ `DIR_IN` register بدل إنه يكتب `chip->sdir` مباشرة — عكس اللي المفروض.

#### الحل
تصحيح الـ DT ليستخدم `dirin` بدل `dirout`:

```dts
/* غلط */
custom_gpio: gpio@ff700000 {
    compatible = "opencores,gpio";
    reg = <0xff700000 0x4>,
          <0xff700004 0x4>;
    reg-names = "dat", "dirout";  /* غلط - ده register DIR_IN */
    ...
};

/* صح */
custom_gpio: gpio@ff700000 {
    compatible = "opencores,gpio";
    reg = <0xff700000 0x4>,
          <0xff700004 0x4>;
    reg-names = "dat", "dirin";   /* صح */
    ...
};
```

مع `dirin`، الـ `gpio_generic_chip_init()` هتحسب `sdir` صح:
```c
else if (chip->reg_dir_in)
    chip->sdir = ~chip->read_reg(chip->reg_dir_in);  /* عكس الـ DIR_IN = output mask */
```

Debug:
```bash
# قبل الـ fix: اتحقق من direction الفعلي
gpioinfo gpiochip2
# لو HDMI HPD pin بيظهر كـ "output" وهو configured input، فالمشكلة موجودة

# بعد الـ fix:
gpioget --chip gpiochip2 15  # HDMI HPD pin
```

#### الدرس المستفاد
الفرق بين `dirout` و `dirin` في الـ DT مش مجرد تسمية — الـ driver بيعامل كل واحد بمنطق معكوس عن الآخر. دايماً ارجع لـ datasheet الـ ASIC وتأكد: "هل الـ 1 في الـ register معناه input ولا output؟" قبل ما تختار الاسم.

---

### السيناريو الثالث: IoT Sensor Board على STM32MP1 — Race condition في Big-Endian GPIO

#### العنوان
**تشغيل `gpio_mmio_set_multiple` على STM32MP1 بـ `be_bits=true` بيعمل race condition صامت**

#### السياق
IoT sensor board بيستخدم STM32MP1. الـ board فيها custom FPGA بـ GPIO register بيستخدم big-endian bit order (bit 31 = line 0). الـ engineer حط `GPIO_GENERIC_BIG_ENDIAN` في الـ flags. المشكلة بتظهر لما thread اتنين بيحاولوا يكتبوا على GPIOs مختلفة في نفس الوقت.

#### المشكلة
الـ `gpio_mmio_set_multiple` بتتنادى من سياقين مختلفين. الأداة بتتوقف عن الاستجابة بشكل intermittent:

```c
static int gpio_mmio_set_multiple(struct gpio_chip *gc, unsigned long *mask,
                                  unsigned long *bits)
{
    struct gpio_generic_chip *chip = to_gpio_generic_chip(gc);
    /* بتنادي gpio_mmio_set_multiple_single_reg */
    gpio_mmio_set_multiple_single_reg(gc, mask, bits, chip->reg_dat);
    return 0;
}
```

الـ `gpio_mmio_set_multiple_single_reg` بتاخد الـ `raw_spinlock_irqsave`:
```c
static void gpio_mmio_set_multiple_single_reg(struct gpio_chip *gc,
                                              unsigned long *mask,
                                              unsigned long *bits,
                                              void __iomem *reg)
{
    struct gpio_generic_chip *chip = to_gpio_generic_chip(gc);
    unsigned long set_mask, clear_mask;

    guard(raw_spinlock_irqsave)(&chip->lock);  /* محمي صح */

    gpio_mmio_multiple_get_masks(gc, mask, bits, &set_mask, &clear_mask);
    chip->sdata |= set_mask;
    chip->sdata &= ~clear_mask;
    chip->write_reg(reg, chip->sdata);
}
```

المشكلة مش في الـ lock — المشكلة في الـ `gpio_mmio_multiple_get_masks()`:

```c
static void gpio_mmio_multiple_get_masks(struct gpio_chip *gc,
                                         unsigned long *mask,
                                         unsigned long *bits,
                                         unsigned long *set_mask,
                                         unsigned long *clear_mask)
{
    struct gpio_generic_chip *chip = to_gpio_generic_chip(gc);
    int i;

    *set_mask = 0;
    *clear_mask = 0;

    for_each_set_bit(i, mask, chip->bits) {
        if (test_bit(i, bits))
            *set_mask |= gpio_mmio_line2mask(gc, i);  /* BE mask */
        else
            *clear_mask |= gpio_mmio_line2mask(gc, i);
    }
}
```

الـ `gpio_mmio_line2mask` مع `be_bits=true`:
```c
static unsigned long gpio_mmio_line2mask(struct gpio_chip *gc, unsigned int line)
{
    struct gpio_generic_chip *chip = to_gpio_generic_chip(gc);

    if (chip->be_bits)
        return BIT(chip->bits - 1 - line);  /* line 0 → bit 31 */
    return BIT(line);
}
```

المشكلة الفعلية: الـ `mask` و `bits` اللي بتيجي من الـ GPIO framework بتستخدم الـ native bit order (line 0 = bit 0). لكن `gpio_mmio_multiple_get_masks` بتحوّل للـ BE bit order. الـ `for_each_set_bit(i, mask, ...)` بيمشي على الـ bits بالـ LE order صح. اللي بيحصل هو إن الـ `bits` array لازم تكون بنفس الـ convention — وده مضمون لأن الـ GPIO core بيبعت الـ mask والـ bits بـ line numbers، مش hardware bit positions. الـ bug الفعلي هو إن `gpio_mmio_get_set_multiple` مش بيتعين مع `be_bits=true`:

```c
/* في gpio_mmio_setup_io() */
if (!(cfg->flags & GPIO_GENERIC_UNREADABLE_REG_SET) &&
    (cfg->flags & GPIO_GENERIC_READ_OUTPUT_REG_SET)) {
    gc->get = gpio_mmio_get_set;
    if (!chip->be_bits)
        gc->get_multiple = gpio_mmio_get_set_multiple;  /* مش بيتعين مع be_bits! */
}
```

الـ comment في الكود بيوضح ده عمداً:
```c
/*
 * We deliberately avoid assigning the ->get_multiple() call
 * for big endian mirrored registers which are ALSO reflecting
 * their value in the set register when used as output.
 */
```

يعني ده `design decision` مقصود، مش bug. لكن الـ engineer مش عارف ليه الـ `get_multiple` مش شغال.

#### الحل
الـ engineer لازم يفهم إن الـ `get_multiple` fallback للـ `get` واحد واحد هو الـ expected behavior. لو محتاج performance، يضيف `GPIO_GENERIC_READ_OUTPUT_REG_SET` لكن مع `be_bits=false`:

```c
/* في الـ driver الخاص بيه */
static int myboard_gpio_probe(struct platform_device *pdev)
{
    struct gpio_generic_chip_config cfg = {
        .dev   = &pdev->dev,
        .sz    = 4,
        .dat   = base + 0x00,
        .set   = base + 0x04,
        .clr   = base + 0x08,
        /* إزالة GPIO_GENERIC_BIG_ENDIAN لو الـ FPGA يقدر يتعدل */
        .flags = GPIO_GENERIC_READ_OUTPUT_REG_SET,
    };
    ...
}
```

أو يستخدم `GPIO_GENERIC_BIG_ENDIAN` مع `GPIO_GENERIC_READ_OUTPUT_REG_SET` ويتقبل إن الـ `get_multiple` هيشتغل slow (per-line):

```bash
# تشخيص: شوف إيه الـ get_multiple pointer
cat /proc/kallsyms | grep gpio_mmio_get
```

#### الدرس المستفاد
الـ `gpio-mmio` فيها قرارات design مقصودة موثقة في الكود. لو شفت إن `get_multiple` مش بيتعين، ارجع للكود وشوف الـ comment. الـ big-endian + read-from-set-reg combination معقدة جداً لدرجة إن الـ maintainer قرر إنها مش تستاهل الـ complexity.

---

### السيناريو الرابع: Automotive ECU على i.MX8 — Glitch على SPI chip select أثناء Direction Change

#### العنوان
**تغيير direction الـ GPIO المتصل بـ SPI CS على i.MX8 بيعمل glitch يكسر الـ SPI transaction**

#### السياق
Automotive ECU بيستخدم i.MX8QM. الـ GPIO المتصل بـ SPI chip select بيتحكم فيه عبر GPIO expander مبني على `gpio-mmio`. الـ ASIC اللي عنده الـ GPIO controller بيحتاج تتحدد الـ direction قبل ما تكتب القيمة (`GPIO_GENERIC_NO_SET_ON_INPUT`). المشكلة ظهرت في safety validation.

#### المشكلة
لما الـ SPI driver بيطلب `gpio_direction_output(cs_gpio, 1)` عشان يرفع الـ CS، بيحصل glitch قصير جداً (< 10ns) على الـ CS pin بيسبب SPI communication error. الـ oscilloscope بيشوف الـ pin بيروح low للحظة قبل ما يروح high.

#### التحليل
الـ `gpio_mmio_dir_out_dir_first` هي المختارة لما `GPIO_GENERIC_NO_SET_ON_INPUT` مضبوطة:

```c
static int gpio_mmio_dir_out_dir_first(struct gpio_chip *gc, unsigned int gpio,
                                       int val)
{
    gpio_mmio_dir_out(gc, gpio, val);   /* خطوة 1: اغير الـ direction لـ output */
    gc->set(gc, gpio, val);             /* خطوة 2: اكتب القيمة */
    return gpio_mmio_dir_return(gc, gpio, true);
}
```

ما بين خطوة 1 وخطوة 2، الـ GPIO بقى output لكن قيمته `0` (الـ default عند الـ direction change في بعض الـ ASICs). ده بيسبب الـ glitch.

لو استخدمنا `gpio_mmio_dir_out_val_first` (السلوك الـ default):
```c
static int gpio_mmio_dir_out_val_first(struct gpio_chip *gc, unsigned int gpio,
                                       int val)
{
    gc->set(gc, gpio, val);             /* خطوة 1: اكتب القيمة الـ output register */
    gpio_mmio_dir_out(gc, gpio, val);   /* خطوة 2: اغير الـ direction */
    return gpio_mmio_dir_return(gc, gpio, true);
}
```

الـ order الـ صح للـ CS هو: اكتب `1` في output register الأول، وبعدين غير الـ direction. كده لما الـ pin يتحول لـ output، قيمته هتكون `1` فوراً من غير glitch.

الـ ASIC documentation بتقول إنه يقبل كتابة في output register وهو input mode — يعني `GPIO_GENERIC_NO_SET_ON_INPUT` غلط لهذا الـ ASIC.

#### الحل
إزالة `GPIO_GENERIC_NO_SET_ON_INPUT` من الـ flags:

```c
/* في الـ driver الخاص */
struct gpio_generic_chip_config cfg = {
    .dev   = &pdev->dev,
    .sz    = 4,
    .dat   = base + DAT_OFFSET,
    .dirout = base + DIROUT_OFFSET,
    /* إزالة GPIO_GENERIC_NO_SET_ON_INPUT */
    .flags = 0,
};
```

ده بيختار `gpio_mmio_dir_out_val_first` بدل `gpio_mmio_dir_out_dir_first`:
```c
/* في gpio_mmio_setup_direction() */
if (cfg->dirout || cfg->dirin) {
    ...
    if (cfg->flags & GPIO_GENERIC_NO_SET_ON_INPUT)
        gc->direction_output = gpio_mmio_dir_out_dir_first;
    else
        gc->direction_output = gpio_mmio_dir_out_val_first;  /* ده اللي هيتعين دلوقتي */
}
```

Debug:
```bash
# على i.MX8 مع ftrace
echo gpio_mmio_dir_out_dir_first > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
# شغل الـ SPI transaction
cat /sys/kernel/debug/tracing/trace
```

#### الدرس المستفاد
`GPIO_GENERIC_NO_SET_ON_INPUT` مش دايماً "أأمن" — هو بيسبب glitch لما تغير direction لـ active-low CS pins. دايماً اقرأ الـ ASIC datasheet وشوف: هل بيقبل كتابة في الـ output register وهو في input mode؟ لو آه، استخدم `val_first` (الـ default).

---

### السيناريو الخامس: Custom Board Bring-up على AM62x — `ngpios` مش بيتقرأ من DT

#### العنوان
**الـ `gpio-mmio` على AM62x بيرجع 32 GPIO بدل 16 والـ extra pins بتعمل kernel crash**

#### السياق
Custom board بتستخدم AM62x (TI Sitara) مع FPGA GPIO expander بـ 16-bit register، بس FPGA فيها 16 GPIO فعلياً بس (نص الـ register). الـ engineer حط `sz = 4` (32-bit) لأن الـ bus 32-bit، لكن فعلياً الـ GPIOs بس 16.

#### المشكلة
الـ kernel crash بيحصل لما حاجة بتحاول تستخدم GPIO 16-31:

```
kernel BUG at drivers/gpio/gpiolib.c:...
GPIO 23 out of range for chip gpiochip3 (ngpio=32)
```

في الوقت نفسه، الـ engineer شايف إن الـ `ngpios` property في الـ DT موجودة:
```dts
fpga_gpio: gpio@4a010000 {
    compatible = "opencores,gpio";
    reg = <0x4a010000 0x4>;
    reg-names = "dat";
    #gpio-cells = <2>;
    gpio-controller;
    ngpios = <16>;
};
```

#### التحليل
في `gpio_generic_chip_init()`:

```c
ret = gpiochip_get_ngpios(gc, dev);
if (ret)
    gc->ngpio = chip->bits;  /* fallback لـ bits لو gpiochip_get_ngpios فشل */
```

`gpiochip_get_ngpios()` بتقرأ الـ `ngpios` property من DT — لو نجحت، `gc->ngpio = 16` صح.

لكن المشكلة إن الـ engineer بيستخدم الـ driver عبر platform device من board file (مش DT)، والـ `dev` مش عنده `ngpios` property. فـ `gpiochip_get_ngpios()` بترجع error، والـ fallback بيعطي `chip->bits = 32`.

الـ `chip->bits` اتحسب من `cfg->sz`:
```c
chip->bits = cfg->sz * 8;  /* 4 * 8 = 32 */
```

الـ engineer حط `sz = 4` لأن الـ MMIO bus 32-bit، لكن FPGA فعلياً بس 16 GPIO.

#### الحل
**خيارات:**

**خيار 1**: حط `sz = 2` في الـ config عشان `chip->bits = 16`:
```c
struct gpio_generic_chip_config cfg = {
    .dev = &pdev->dev,
    .sz  = 2,    /* 16-bit = 16 GPIOs */
    .dat = ioremap(0x4a010000, 4),
    .flags = 0,
};
```

لكن ده هيستخدم `gpio_mmio_read16` / `gpio_mmio_write16` بدل الـ 32-bit accessors، وده قد يكون مشكلة على الـ 32-bit only bus.

**خيار 2**: اضيف `ngpios` property عبر platform data أو device properties:
```c
static const struct property_entry fpga_gpio_props[] = {
    PROPERTY_ENTRY_U32("ngpios", 16),
    {}
};

static const struct software_node fpga_gpio_swnode = {
    .properties = fpga_gpio_props,
};

/* في الـ probe أو board init */
device_add_software_node(&pdev->dev, &fpga_gpio_swnode);
```

بعد كده `gpiochip_get_ngpios()` هترجع `16` وكل حاجة تبقى تمام.

**خيار 3** (لو DT متاح): تأكد إن الـ DT node وصله للـ driver صح:
```bash
# تحقق إن الـ DT property بتتقرأ
cat /sys/firmware/devicetree/base/soc/gpio@4a010000/ngpios | xxd
# المفروض يظهر 00 00 00 10 (16 decimal في big-endian)
```

Debug إضافي:
```bash
# شوف ngpio الـ chip اللي اتسجل
gpioinfo 2>/dev/null | grep "gpiochip"

# تحقق من الـ chip bits عبر debugfs
cat /sys/kernel/debug/gpio
```

#### الدرس المستفاد
`sz` في `gpio_generic_chip_config` مش "حجم الـ MMIO bus" — هو "حجم الـ register الفعلي الممثل للـ GPIOs". لو الـ FPGA فيها 16 GPIO في register 32-bit، لازم تحدد `ngpios = <16>` في DT أو عبر software node. الـ fallback في `gpio_generic_chip_init()` هو `chip->bits` وده بيفترض إن كل bit في الـ register = GPIO واحد.

---
## Phase 7: مصادر ومراجع

### مقالات LWN.net

الـ LWN.net هو المصدر الأهم للمتابعة العميقة لتطور الـ Linux kernel، وفيما يلي أهم المقالات المرتبطة بالـ GPIO subsystem والـ gpio-generic driver:

| المقال | الوصف |
|--------|-------|
| [GPIO in the kernel: an introduction](https://lwn.net/Articles/532714/) | مقال Jonathan Corbet عام 2013 — مقدمة شاملة لـ GPIO API في الكيرنل، يشرح `gpio_chip`، الـ descriptor-based API، وكيف بدأت القصة مع David Brownell في 2.6.21 |
| [[PATCH v4] gpio: Add driver for basic memory-mapped GPIO controllers](https://lwn.net/Articles/402868/) | الـ patch الأصلي اللي أضاف الـ `bgpio` / `gpio-generic` driver — مهم جداً لفهم القرارات التصميمية الأولى للـ `bgpio_init()` وهياكل الـ `bgpio_chip` |
| [GPIO implementation framework](https://lwn.net/Articles/256461/) | المقال اللي وثّق أول framework للـ GPIO في الكيرنل، أساس كل اللي جاء بعده |
| [gpio: sysfs interface (updated)](https://lwn.net/Articles/286435/) | تحديث واجهة الـ sysfs للـ GPIO — مهم لفهم كيف يتعرض الـ GPIO لـ userspace |
| [Documentation/gpio.txt](https://lwn.net/Articles/532717/) | النسخة الأرشيفية من توثيق الـ GPIO الرسمي — مرجع تاريخي مفيد |

---

### التوثيق الرسمي للكيرنل (`Documentation/`)

الـ paths دي موجودة في أي kernel source tree:

```
Documentation/driver-api/gpio/index.rst       ← نقطة البداية
Documentation/driver-api/gpio/intro.rst       ← مقدمة تاريخية
Documentation/driver-api/gpio/driver.rst      ← واجهة كتابة الـ GPIO driver
Documentation/driver-api/gpio/consumer.rst    ← كيف تستخدم GPIO من driver تاني
Documentation/driver-api/gpio/board.rst       ← ربط GPIO بالـ platform/DT
Documentation/driver-api/gpio/legacy.rst      ← الـ integer-based API القديم
```

**الـ online links:**

- [GPIO Driver Interface — docs.kernel.org](https://docs.kernel.org/driver-api/gpio/driver.html) — يشرح `bgpio_init()` وعلاقتها بـ `gpio_chip`
- [GPIO General — docs.kernel.org](https://docs.kernel.org/driver-api/gpio/index.html) — الفهرس الرئيسي
- [GPIO Introduction](https://docs.kernel.org/driver-api/gpio/intro.html) — لماذا وُجد الـ GPIO subsystem
- [GPIO Consumer Interface](https://docs.kernel.org/driver-api/gpio/consumer.html) — الـ `gpiod_get()` وأخواتها
- [GPIO Mappings](https://docs.kernel.org/driver-api/gpio/board.html) — ربط GPIOs بالـ device tree

---

### الكود المصدري عبر Bootlin Elixir

الـ [Elixir Cross Reference](https://elixir.bootlin.com) أداة لا غنى عنها لتتبع تطور الكود عبر إصدارات الكيرنل:

| الملف / الرابط | الوصف |
|---------------|-------|
| [gpio-generic.c — v3.8](https://elixir.bootlin.com/linux/v3.8/source/drivers/gpio/gpio-generic.c) | نسخة مبكرة من الـ driver قبل ما يتحول لـ gpio-mmio |
| [gpio-generic.c — v4.4](https://elixir.bootlin.com/linux/v4.4.153/source/drivers/gpio/gpio-generic.c) | نسخة مستقرة مع الـ `bgpio_init()` الكاملة |
| [bgpio_init identifier — v4.6](https://elixir.bootlin.com/linux/v4.6/ident/bgpio_init) | تتبع كل استخدامات `bgpio_init` في الكيرنل |
| [gpio-mmio.c — latest](https://elixir.bootlin.com/linux/v6.11/source/drivers/gpio) | الـ driver الحالي بعد إعادة التسمية من gpio-generic |
| [include/linux/gpio/driver.h — v4.8](https://elixir.bootlin.com/linux/v4.8/source/include/linux/gpio/driver.h) | تعريف `struct gpio_chip` و`bgpio_init` الـ prototype |

> **ملاحظة:** الملف اتغير اسمه من `drivers/gpio/gpio-generic.c` لـ `drivers/gpio/gpio-mmio.c` اعتباراً من كيرنل 4.7 تقريباً.

---

### النقاشات على Mailing Lists

- [linux-gpio mailing list archive — lore.kernel.org](https://lore.kernel.org/linux-gpio/) — الأرشيف الرسمي لكل patches الـ GPIO
- [[PATCH v3] gpio: mmio: handle "ngpios" properly in bgpio_init()](https://lore.kernel.org/lkml/202303050354.HH9DhJsr-lkp@intel.com/t/) — نقاش مهم حول معالجة عدد الـ GPIOs في الـ init
- [[v2] gpio-generic: add bgpio_set_multiple functions](https://patchwork.ozlabs.org/project/linux-gpio/patch/3218650.3Kg56DKAcg@pcimr/) — إضافة دعم الـ `set_multiple` لتحسين الأداء
- [Re: [PATCH 0/5] pinctrl: replace legacy bgpio_init()](https://lkml.org/lkml/2025/8/19/691) — Linus Walleij يناقش استبدال `bgpio_init()` القديمة

---

### Commits مهمة في تاريخ الـ Driver

يمكن تتبع هذه الـ commits عبر `git log --follow -- drivers/gpio/gpio-generic.c` أو `drivers/gpio/gpio-mmio.c`:

```bash
# تتبع تاريخ الملف كاملاً
git log --follow --oneline -- drivers/gpio/gpio-mmio.c

# اللي أضاف bgpio_set_multiple
git log --oneline --all -S "bgpio_set_multiple" -- drivers/gpio/

# تاريخ إعادة التسمية
git log --oneline --diff-filter=R --summary -- drivers/gpio/gpio-generic.c
```

**Commits بارزة (للبحث عنها في `git.kernel.org`):**
- الـ commit اللي أضاف `gpio-generic.c` أول مرة (2010–2011، Anton Vorontsov)
- إعادة التسمية لـ `gpio-mmio.c` (kernel ~4.7)
- إضافة دعم الـ `set_multiple` لتحسين الأداء
- إضافة `BGPIOF_*` flags لدعم سيناريوهات مختلفة

---

### الكتب الموصى بها

#### Linux Device Drivers 3rd Edition (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصول ذات الصلة:**
  - Chapter 9: Communicating with Hardware — الـ I/O registers وـ memory-mapped I/O
  - Chapter 11: Data Types in the Kernel — `u8`/`u16`/`u32`/`u64` وأهميتها في الـ bit-width independence
- **متاح مجاناً:** [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصول ذات الصلة:**
  - Chapter 11: Timers and Time Management — spinlock و timing (قريب من مفاهيم الـ lock في `bgpio`)
  - Chapter 20: Patches, Hacking, and the Community — فهم مسار patches الـ GPIO
- **مفيد لـ:** فهم بنية الـ subsystems وكيف تُسجَّل الـ drivers

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصول ذات الصلة:**
  - Chapter 8: Device Driver Basics — ربط مباشر بالـ GPIO drivers
  - Chapter 14: Embedded Linux Applications — استخدام GPIO من userspace
- **مفيد لـ:** السياق العملي للـ FPGA و ASIC مع gpio-generic

#### The Linux Programming Interface — Michael Kerrisk
- مرجع ممتاز لواجهة الـ `/sys/class/gpio` من الـ userspace

---

### مصادر إضافية

#### KernelNewbies.org

- [Linux_3.6_DriverArch](https://kernelnewbies.org/Linux_3.6_DriverArch) — تغييرات بنية الـ driver في 3.6 تشمل الـ GPIO
- [Linux_5.10](https://kernelnewbies.org/Linux_5.10) — الـ GPIO CDEV uAPI الجديد (CHARACTER DEVICE)
- [Linux_6.8](https://kernelnewbies.org/Linux_6.8) — آخر تحديثات الـ GPIO subsystem
- [Finding GPIO names under Linux](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2016-May/016303.html) — نقاش عملي حول `gpiod_get()`

#### eLinux.org

- [Pin Control and GPIO Update (PDF)](https://elinux.org/images/c/cd/Pincontrol-gpio-update.pdf) — عرض تقديمي عن تطور الـ GPIO وـ pinctrl في الكيرنل
- [Leapster Explorer: GPIO subsystem](https://elinux.org/Leapster_Explorer:_GPIO_subsystem) — مثال embedded حقيقي لاستخدام الـ GPIO subsystem
- [DHT-Walnut GPIO](https://elinux.org/DHT-Walnut_GPIO) — PPC405 GPIO driver — مثال قديم يوضح register-level GPIO
- [RPi SPI](https://elinux.org/RPi_SPI) — GPIO chip select مع SPI — تطبيق عملي

#### مواقع أخرى مفيدة

- [Using gpio-generic and irq_chip_generic subsystems for gpio driver](http://maquefel.me/en/using-gpio-generic-and-irq_chip_generic-subsystems-for-gpio-driver/) — شرح تطبيقي ممتاز لكتابة driver حقيقي باستخدام `bgpio_init()`
- [Bootlin Training Materials](https://bootlin.com/training/linux-kernel/) — مواد تدريب مجانية تغطي GPIO drivers

---

### مصطلحات للبحث عنها

```
bgpio_init linux kernel
gpio_chip linux driver
memory-mapped GPIO controller linux
bgpio_set_multiple kernel
linux gpio descriptor API gpiod
gpio-mmio.c kernel driver
BGPIOF_READ_OUTPUT_REG_SET
linux gpio spinlock shadow register
gpio_chip set_multiple performance
Anton Vorontsov gpio-generic kernel
Linus Walleij gpio maintainer
linux-gpio mailing list lore.kernel.org
pinctrl gpio integration kernel
gpiochip_add_data bgpio
```

---

### ملخص المسار الموصى به للتعلم

```
1. اقرأ LDD3 Chapter 9 (memory-mapped I/O)
         ↓
2. اقرأ Documentation/driver-api/gpio/intro.rst
         ↓
3. اقرأ LWN Articles/532714 (GPIO introduction)
         ↓
4. اقرأ LWN Articles/402868 (الـ patch الأصلي لـ gpio-generic)
         ↓
5. استعرض الكود على Bootlin Elixir (v3.8 → latest)
         ↓
6. اتابع linux-gpio mailing list على lore.kernel.org
```
## Phase 8: Writing simple module

### الـ Function المختارة: `gpio_generic_chip_init`

**الـ function دي** هي الـ exported symbol الوحيدة في الملف (`EXPORT_SYMBOL_GPL`)، وبتعمل initialize لأي `gpio_generic_chip` مهما كان الـ platform. دي نقطة دخول مناسبة جداً للـ kprobe لأن كل driver بيستخدم الـ generic GPIO subsystem لازم يمر بيها، فبنقدر نشوف كل GPIO chip بيتسجل في الـ kernel مع تفاصيله.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_gpio_generic.c
 *
 * Hooks gpio_generic_chip_init() to log every generic GPIO chip
 * being registered: its label, number of lines, and register width.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit        */
#include <linux/kernel.h>       /* pr_info                                  */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe, ...      */
#include <linux/gpio/generic.h> /* struct gpio_generic_chip, _config        */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("GPIO Watcher");
MODULE_DESCRIPTION("kprobe on gpio_generic_chip_init to log every generic GPIO chip init");

/* -----------------------------------------------------------------------
 * الـ pre-handler: بيتنفذ قبل ما gpio_generic_chip_init تبدأ فعلاً.
 * بنقرأ الـ arguments من الـ registers قبل ما الـ function تعدّل أي حاجة.
 * ----------------------------------------------------------------------- */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * على x86-64: الـ argument الأول (chip) في rdi، الثاني (cfg) في rsi.
     * على ARM64:  الأول في x0، الثاني في x1.
     * بنستخدم الـ macro regs_get_kernel_argument() اللي portable جوه
     * الـ kprobes API بدل ما نـ hardcode الـ register names.
     */
#ifdef CONFIG_X86_64
    struct gpio_generic_chip        *chip = (struct gpio_generic_chip *)regs->di;
    struct gpio_generic_chip_config *cfg  = (struct gpio_generic_chip_config *)regs->si;
#elif defined(CONFIG_ARM64)
    struct gpio_generic_chip        *chip = (struct gpio_generic_chip *)regs->regs[0];
    struct gpio_generic_chip_config *cfg  = (struct gpio_generic_chip_config *)regs->regs[1];
#else
    /* fallback: cast from the first two saved regs — may need adjustment */
    struct gpio_generic_chip        *chip = (struct gpio_generic_chip *)regs->ARG1;
    struct gpio_generic_chip_config *cfg  = (struct gpio_generic_chip_config *)regs->ARG2;
#endif

    /*
     * بنـ validate الـ pointers عشان الـ kprobe ممكن يـ fire في أي وقت،
     * ولو فيه NULL access هيـ panic الـ kernel.
     */
    if (!chip || !cfg || !cfg->dev)
        return 0;

    /*
     * cfg->sz ده عرض الـ MMIO register بالـ bytes (1, 2, 4, أو 8).
     * gc->label لسه مش اتضبط هنا (الـ function لم تبدأ)، فبناخده من
     * cfg->dev اللي هو الـ parent device.
     *
     * بنطبع:
     *   - اسم الـ device (parent)
     *   - عرض الـ register بالـ bits (sz * 8)
     *   - هل فيه reg_set منفصل (set/clear model أم data model)
     *   - هل فيه direction registers
     */
    pr_info("gpio_generic_chip_init: device=%s reg_width=%lu bits, "
            "has_set=%s has_clr=%s has_dirout=%s has_dirin=%s\n",
            dev_name(cfg->dev),
            cfg->sz * 8,
            cfg->set    ? "yes" : "no",
            cfg->clr    ? "yes" : "no",
            cfg->dirout ? "yes" : "no",
            cfg->dirin  ? "yes" : "no");

    return 0; /* 0 = الـ kernel يكمل تنفيذ الـ function الأصلية */
}

/* -----------------------------------------------------------------------
 * الـ post-handler: بيتنفذ بعد ما gpio_generic_chip_init ترجع.
 * هنا بنعرف عدد الـ GPIO lines اللي اتضبطت (gc->ngpio).
 * ----------------------------------------------------------------------- */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
#ifdef CONFIG_X86_64
    struct gpio_generic_chip *chip = (struct gpio_generic_chip *)regs->di;
#elif defined(CONFIG_ARM64)
    struct gpio_generic_chip *chip = (struct gpio_generic_chip *)regs->regs[0];
#else
    struct gpio_generic_chip *chip = (struct gpio_generic_chip *)regs->ARG1;
#endif

    if (!chip)
        return;

    /*
     * بعد ما الـ function خلّصت، gc.ngpio و gc.label اتضبطوا صح.
     * بنطبع عدد الـ lines والـ label النهائي.
     */
    pr_info("gpio_generic_chip_init done: label=%s ngpio=%u\n",
            chip->gc.label ? chip->gc.label : "(null)",
            chip->gc.ngpio);
}

/* -----------------------------------------------------------------------
 * تعريف الـ kprobe: بنحدد اسم الـ function بالـ string،
 * الـ kernel بيحوّله لـ address وقت التسجيل.
 * ----------------------------------------------------------------------- */
static struct kprobe kp = {
    .symbol_name = "gpio_generic_chip_init",
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

/* -----------------------------------------------------------------------
 * module_init: بيسجّل الـ kprobe عند تحميل الـ module.
 * لو فشل التسجيل (مثلاً الـ symbol مش موجود أو CONFIG_KPROBES=n)
 * بنرجع error عشان الـ module ما يـ load خالص.
 * ----------------------------------------------------------------------- */
static int __init gpio_kprobe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("register_kprobe failed on gpio_generic_chip_init: %d\n", ret);
        return ret;
    }

    pr_info("kprobe planted on gpio_generic_chip_init at %p\n", kp.addr);
    return 0;
}

/* -----------------------------------------------------------------------
 * module_exit: بيشيل الـ kprobe عند إزالة الـ module.
 * الـ unregister ضروري عشان لو الـ module اتـ unload والـ kprobe لسه
 * مسجّل، أي استدعاء لـ gpio_generic_chip_init هيـ jump لـ handler code
 * اتشالت من الـ memory → kernel panic فوري.
 * ----------------------------------------------------------------------- */
static void __exit gpio_kprobe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("kprobe removed from gpio_generic_chip_init\n");
}

module_init(gpio_kprobe_init);
module_exit(gpio_kprobe_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | الـ macros الأساسية للـ module: `MODULE_LICENSE`، `module_init`، `module_exit` |
| `linux/kernel.h` | الـ `pr_info` / `pr_err` للطباعة في الـ kernel log |
| `linux/kprobes.h` | تعريف `struct kprobe` وكل الـ API الخاصة بالـ hooking |
| `linux/gpio/generic.h` | تعريف `struct gpio_generic_chip` و `struct gpio_generic_chip_config` اللي بنقرأ منهم البيانات |

**الـ** `linux/gpio/generic.h` مش فيها أي dependency ثقيلة، فالـ compile سريع ومباشر.

---

#### الـ `handler_pre`

بيتنفذ **قبل** دخول `gpio_generic_chip_init`، فالـ arguments لسه في الـ registers ولم يتغيروا. بنقرأ `cfg` عشان نعرف تفاصيل الـ chip قبل الـ init، زي عرض الـ register (بالـ bits) وهل فيه set/clear registers منفصلين أم لا. ده مفيد لفهم الـ GPIO model اللي بيستخدمه الـ driver.

---

#### الـ `handler_post`

بيتنفذ **بعد** ما الـ function ترجع بنجاح. في الوقت ده `gc.ngpio` و `gc.label` اتضبطوا جوه `gpio_generic_chip_init`، فبنطبع العدد الفعلي للـ GPIO lines والـ label النهائي. الـ pre وحده مش كفاية عشان بعض المعلومات مش متاحة غير بعد الـ init.

---

#### الـ `struct kprobe`

```
.symbol_name = "gpio_generic_chip_init"
```

بدل ما نحسب الـ address يدوياً، بنديه الاسم والـ kernel بيعمل `kallsyms_lookup_name` داخلياً وقت `register_kprobe`. لو الـ CONFIG_KPROBES أو CONFIG_KALLSYMS مش enabled هيرفض التسجيل بـ `-EINVAL`.

---

#### الـ `module_init` / `module_exit`

الـ `register_kprobe` بيحط **breakpoint** (على x86: `int3` / `brk` على ARM64) في أول byte من الـ function. الـ `unregister_kprobe` بيشيل الـ breakpoint ويرجع الـ original bytes، فلو الـ module اتـ unload من غير unregister الـ kernel هيـ execute garbage code وهيـ panic.

---

### طريقة التجربة

```bash
# بناء الـ module (Makefile بسيط في نفس الـ directory):
# obj-m += kprobe_gpio_generic.o
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# تحميل الـ module:
sudo insmod kprobe_gpio_generic.ko

# تحميل أي driver بيستخدم gpio-generic (مثلاً على QEMU virt):
sudo modprobe gpio-pl061

# مشاهدة الـ output:
sudo dmesg | grep gpio_generic_chip_init

# إزالة الـ module:
sudo rmmod kprobe_gpio_generic
```

**مثال على الـ output المتوقع:**
```
[  12.345678] kprobe planted on gpio_generic_chip_init at ffffffffc0123456
[  12.890123] gpio_generic_chip_init: device=pl061@9030000 reg_width=8 bits, has_set=no has_clr=no has_dirout=yes has_dirin=no
[  12.890456] gpio_generic_chip_init done: label=pl061@9030000 ngpio=8
```
