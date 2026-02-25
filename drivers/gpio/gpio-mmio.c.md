## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الملف `gpio-mmio.c` جزء من **GPIO Subsystem** في Linux kernel، واللي بتحكمه المنتجين الرئيسيين:
- **Linus Walleij** و**Bartosz Golaszewski**
- الـ mailing list: `linux-gpio@vger.kernel.org`
- كل الكود موجود في `drivers/gpio/` و`include/linux/gpio/`

---

### ما هو الـ GPIO؟

**GPIO** (General Purpose Input/Output) هو ببساطة "سلك رقمي" بتقدر تتحكم فيه بالبرمجيانت — تقرأ منه (input) أو تكتب عليه (output). كل embedded system أو SoC فيها عشرات أو مئات من السلوك دي.

تخيل إنك بتتحكم في LED، زرار، أو relay — دي كلها GPIOs.

---

### القصة اللي بيحلها الملف ده

#### المشكلة

في عالم الـ embedded systems (FPGAs، ASICs، microcontrollers، SoCs)، كل شركة بتصمم GPIO controller بطريقتها:

- فيه controller عنده **register واحد** تقرأ وتكتب فيه مباشرةً.
- فيه controller عنده **register للـ set** وآخر للـ **clear** — بدل ما تكتب القيمة كاملة.
- فيه controller عنده **register للـ direction** (يحدد هل الـ pin input ولا output).
- فيه hardware بـ **big-endian** bit ordering — يعني الـ bit رقم 0 في الـ register بيمثل آخر GPIO مش أوله.

لو كل driver لكل chip بتكتبه من الأول، هيبقى فيه تكرار هائل وbugs كتير. لازم يكون فيه **generic driver** يشتغل مع أي controller بياخد فقط عناوين الـ registers.

#### الحل: `gpio-mmio.c`

الملف ده هو الـ **generic MMIO GPIO driver** — يعني driver عام يشتغل مع أي GPIO controller بيعتمد على **Memory-Mapped I/O** (MMIO). يعني بدل ما الـ CPU يبعت أوامر على bus خاص، الـ registers بتتعامل معاها كأنها عناوين في الـ RAM العادية.

```
CPU --[Memory Bus]--> GPIO Registers (MMIO)
         ↓
   readl(0xFEED0000)  → قراءة حالة الـ GPIO pins
   writel(0x01, 0xFEED0004) → كتابة قيمة على الـ pin
```

---

### تخيل المشكلة بمثال حقيقي

**مثال:** عندك FPGA بتصممها بنفسك (مثلاً بـ Verilog). فيها 8 LEDs متوصلة بـ GPIO register على عنوان `0x40000000`. لو اللاب ورقة كتبت عليها:
- `Bit 0 = 1` → LED 0 يولع
- `Bit 7 = 1` → LED 7 يولع

بدل ما تكتب driver جديد من الأول، بتقوله: "فيه register واحد على العنوان ده بيمثل 8 GPIOs" — والـ `gpio-mmio.c` يتولى الباقي.

هو بالظبط كده — الـ comment في الكود بيقول:

> *"Implementing such a GPIO controller in FPGA is trivial."*

---

### الأنماط الثلاثة للـ Output

| النمط | الـ Registers | الآلية |
|-------|--------------|--------|
| **Single DAT** | `dat` فقط | تكتب القيمة كاملة — set بـ 1، clear بـ 0 |
| **SET + CLR pair** | `set` + `clr` | تكتب 1 في `set` لتشغيل، تكتب 1 في `clr` لإطفاء |
| **SET only** | `set` فقط | `set` بيحفظ القيمة ويوصلها للـ pin |

الفرق بين النمط الأول والثاني مهم جداً لأن مع **SET+CLR** مش محتاج lock لأن كل عملية atomic على مستوى الـ hardware.

---

### الأنماط الثلاثة للـ Direction

| النمط | التفاصيل |
|-------|----------|
| **No direction register** | الـ GPIO إما input-only أو output-only |
| **`dirout` register** | `1` في الـ bit = output، `0` = input |
| **`dirin` register** | `1` في الـ bit = input، `0` = output |

---

### مشكلة الـ Shadow Register

لما بيكون عندك register واحد للـ input والـ output معاً، ولو عايز تعمل **read-modify-write** (اقرأ القيمة الحالية، غير bit معين، اكتب الجديدة)، بيحصل race condition:

```
Thread A: read → 0b00001111
Thread B: read → 0b00001111   ← نفس القيمة!
Thread A: set bit 4 → write 0b00011111
Thread B: set bit 5 → write 0b00101111  ← ضيّع تعديل Thread A!
```

الحل: **`sdata`** — الـ shadow register. الـ driver بيحتفظ بنسخة داخلية من القيمة الحالية ويعمل عليها الـ modify بعد lock، بدل ما يقرأ من الـ hardware.

```c
guard(raw_spinlock_irqsave)(&chip->lock);
chip->sdata |= mask;   // عدّل الـ shadow
chip->write_reg(chip->reg_dat, chip->sdata);  // اكتب للـ hardware
```

---

### مشكلة الـ Big-Endian Bit Order

بعض الـ hardware (مثلاً `bcm6345-gpio`) بتعتبر إن **bit 31 = GPIO 0** مش bit 0. ده بيخلي التحويل بين رقم الـ GPIO والـ bit mask محتاج تعكس الترتيب:

```c
static unsigned long gpio_mmio_line2mask(struct gpio_chip *gc, unsigned int line)
{
    struct gpio_generic_chip *chip = to_gpio_generic_chip(gc);
    if (chip->be_bits)
        return BIT(chip->bits - 1 - line);  // عكس الترتيب
    return BIT(line);                        // ترتيب طبيعي
}
```

---

### البنية الأساسية

```
struct gpio_generic_chip
├── struct gpio_chip gc          ← الـ base struct اللي gpiolib بتشوفها
├── read_reg / write_reg         ← function pointers للـ 8/16/32/64-bit access
├── reg_dat / reg_set / reg_clr  ← عناوين الـ MMIO registers
├── reg_dir_out / reg_dir_in     ← عناوين الـ direction registers
├── sdata                        ← shadow data register
├── sdir                         ← shadow direction register
├── lock                         ← raw spinlock للحماية
├── bits                         ← عدد الـ bits (8/16/32/64)
└── be_bits                      ← big-endian bit order flag
```

---

### الـ Supported Devices (من الـ Device Tree)

الـ driver بيدعم هذه الـ hardware controllers:

| الـ Compatible String | الجهاز |
|----------------------|--------|
| `brcm,bcm6345-gpio` | Broadcom BCM6345 (DSL routers) |
| `wd,mbl-gpio` | Western Digital MyBook Live |
| `ni,169445-nand-gpio` | National Instruments |
| `intel,ixp4xx-expansion-bus-mmio-gpio` | Intel IXP4xx |
| `opencores,gpio` | OpenCores generic GPIO |

---

### الـ Files المهمة في الـ Subsystem

#### Core Files

| الملف | الدور |
|-------|-------|
| `drivers/gpio/gpio-mmio.c` | **الملف ده** — Generic MMIO GPIO driver |
| `drivers/gpio/gpiolib.c` | قلب الـ GPIO subsystem — إدارة كل الـ chips |
| `drivers/gpio/gpiolib.h` | Internal header للـ gpiolib |
| `drivers/gpio/gpiolib-devres.c` | الـ devm_ helpers للـ resource management |
| `drivers/gpio/gpiolib-cdev.c` | الـ userspace character device interface |
| `drivers/gpio/gpiolib-of.c` | Device Tree integration |
| `drivers/gpio/gpiolib-acpi-core.c` | ACPI integration |

#### Headers

| الملف | الدور |
|-------|-------|
| `include/linux/gpio/generic.h` | تعريف `struct gpio_generic_chip` و`struct gpio_generic_chip_config` |
| `include/linux/gpio/driver.h` | تعريف `struct gpio_chip` — الـ base struct لكل GPIO controller |
| `include/linux/gpio.h` | الـ consumer API للـ kernel code |

#### Related Drivers (نفس الفكرة بـ hardware مختلف)

| الملف | الجهاز |
|-------|--------|
| `drivers/gpio/gpio-74xx-mmio.c` | 74xx logic chips via MMIO |
| `drivers/gpio/gpio-regmap.c` | MMIO controllers via regmap framework |
| `drivers/gpio/gpio-dwapb.c` | Synopsys DesignWare APB GPIO |

---

### الـ Architecture بالصورة الكبيرة

```
Userspace (gpioset / libgpiod)
         │
         ▼
  gpiolib-cdev.c   ← /dev/gpiochipN
         │
         ▼
    gpiolib.c      ← GPIO core (request, get, set, direction)
         │
         ▼
  gpio-mmio.c      ← Generic MMIO driver (gpio_chip callbacks)
         │
         ▼
  MMIO Registers   ← Physical hardware (FPGA / ASIC / SoC)
```

الـ `gpio-mmio.c` بيملأ الـ function pointers في `struct gpio_chip` (زي `get`, `set`, `direction_input`, `direction_output`) — وبكده الـ `gpiolib.c` بيتعامل معاه بشكل موحد بغض النظر عن الـ hardware.
## Phase 2: شرح الـ GPIO MMIO Framework

---

### المشكلة اللي بيحلها الـ subsystem ده

في أي SoC أو FPGA أو ASIC، في عشرات الـ GPIO controllers مختلفين — كل واحد ليه registers بعناوين مختلفة وبعضهم بيستخدم "set/clear pair" وبعضهم "single data register" وبعضهم big-endian وبعضهم little-endian. لو كتبت driver مستقل لكل controller ده، هتعيد نفس الكود مئات المرات مع فرق بسيط جداً في الـ register layout.

**المشكلة الحقيقية:**
- Hardware diversity: نفس الفكرة (GPIO عبارة عن bits في register) بس implementation مختلفة
- Code duplication: كل chip ممكن يعمل driver مستقل، لكن الـ logic نفسها
- Kernel API mismatch: الـ consumer (اللي بيستخدم GPIO) محتاج interface موحدة بغض النظر عن الـ hardware

---

### الحل اللي اتبعه الـ kernel

الـ kernel قدم **GPIO subsystem** كطبقة abstraction وجوا الـ subsystem ده في driver خاص اسمه `gpio-mmio` يخدم أي GPIO controller اللي:
1. بيتحكم فيه عن طريق **Memory-Mapped I/O registers**
2. كل bit في الـ register بيمثل GPIO line واحدة
3. عدد الـ GPIOs = عرض الـ register بالـ bits (8 أو 16 أو 32 أو 64)

الفكرة: بدل ما تكتب driver لكل chip، تدي الـ driver ده عناوين الـ registers وهو يتكفل بالباقي.

---

### Analogy عميقة — لوحة مفاتيح كهربائية في فندق

تخيل فندق كبير فيه لوحات تحكم في الكهرباء. كل غرفة عندها لوحة مختلفة:
- غرفة A: مفتاح واحد بيشغل أو يطفي (data register فقط)
- غرفة B: زرار "تشغيل" وزرار "إطفاء" منفصلين (set + clear registers)
- غرفة C: اللوحة مكتوب عليها من اليمين لليسار (big-endian bits)

**مدير الفندق (الـ gpio-mmio driver)** عنده كارنيه موحد (الـ `gpio_chip` interface) بيتعامل مع الكهرباء، بس بيعرف إزاي يتعامل مع كل غرفة لأن عنده تعليمات مخصوصة لكل نوع.

الـ mapping التفصيلي:

| المثال في الفندق | المقابل في الكود |
|---|---|
| غرفة بمفتاح واحد | `reg_dat` فقط |
| غرفة بزرار تشغيل + إطفاء | `reg_set` + `reg_clr` |
| لوحة مكتوبة من اليمين | `be_bits = true` → `BIT(bits-1-line)` |
| مدير يحفظ حالة الكهرباء | `sdata` shadow register |
| سجل من الغرفة input-only | `GPIO_GENERIC_NO_OUTPUT` flag |
| مدير يعرف إزاي يقرأ أي لوحة | `read_reg` / `write_reg` function pointers |
| قفل على لوحة التحكم | `raw_spinlock_t lock` |

---

### المعمارية الكاملة — Big Picture

```
  ┌─────────────────────────────────────────────────────────────────────┐
  │                      User Space / Kernel Drivers                    │
  │          (يطلبوا GPIO عبر gpiod_get / gpio_request)                 │
  └───────────────────────────┬─────────────────────────────────────────┘
                              │
  ┌───────────────────────────▼─────────────────────────────────────────┐
  │                        gpiolib core                                  │
  │          (drivers/gpio/gpiolib.c  +  gpiolib-of.c  ...)              │
  │   يحول طلبات الـ consumer لـ calls على gpio_chip callbacks           │
  └───────────────────────────┬─────────────────────────────────────────┘
                              │   calls: gc->get / gc->set /
                              │          gc->direction_input / etc.
  ┌───────────────────────────▼─────────────────────────────────────────┐
  │               GPIO MMIO Framework  (gpio-mmio.c)                     │
  │                                                                      │
  │  ┌────────────────────────────────────────────────────────────┐     │
  │  │            struct gpio_generic_chip                         │     │
  │  │  ┌──────────────────┐   function pointers:                 │     │
  │  │  │  struct gpio_chip│   read_reg  ──► gpio_mmio_read8/16/32│     │
  │  │  │  (embedded)      │   write_reg ──► gpio_mmio_write8/... │     │
  │  │  └──────────────────┘                                      │     │
  │  │  reg_dat  ──► pointer to DATA register in MMIO             │     │
  │  │  reg_set  ──► pointer to SET  register in MMIO (optional)  │     │
  │  │  reg_clr  ──► pointer to CLR  register in MMIO (optional)  │     │
  │  │  reg_dir_out ──► direction OUT register (optional)         │     │
  │  │  reg_dir_in  ──► direction IN  register (optional)         │     │
  │  │  sdata  ──► shadow copy of data register (in RAM)          │     │
  │  │  sdir   ──► shadow copy of direction register (in RAM)     │     │
  │  │  lock   ──► raw_spinlock to protect sdata/sdir             │     │
  │  └────────────────────────────────────────────────────────────┘     │
  └───────────────────────────┬─────────────────────────────────────────┘
                              │  readb/writeb, readl/writel, ioread32be...
  ┌───────────────────────────▼─────────────────────────────────────────┐
  │                      MMIO Hardware Registers                         │
  │                                                                      │
  │   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
  │   │ dat reg  │  │ set reg  │  │ clr reg  │  │dirout reg│           │
  │   │0xFE00000 │  │0xFE00004 │  │0xFE00008 │  │0xFE0000C │           │
  │   └──────────┘  └──────────┘  └──────────┘  └──────────┘           │
  │                                                                      │
  │   Hardware: BCM6345 / WD MBL / NI NAND / Intel IXP4xx / OpenCores   │
  └─────────────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ `struct gpio_generic_chip` هو قلب الـ framework. هو بيعمل شيئين:

1. **يحتوي** على `struct gpio_chip` كـ embedded field — ده اللي بيتكلم مع الـ gpiolib core
2. **يضيف** فوقيه طبقة تعامل مع registers: function pointers للقراءة والكتابة، وعناوين الـ registers، وـ shadow copies

```
struct gpio_generic_chip {
    struct gpio_chip gc;          // ◄── الـ interface للـ gpiolib core (embedded)

    // function pointers للـ I/O width
    unsigned long (*read_reg)(void __iomem *reg);
    void (*write_reg)(void __iomem *reg, unsigned long data);

    // MMIO register addresses
    void __iomem *reg_dat;        // ◄── data register (قراءة الحالة الحالية)
    void __iomem *reg_set;        // ◄── اختياري: set output high
    void __iomem *reg_clr;        // ◄── اختياري: set output low
    void __iomem *reg_dir_out;    // ◄── اختياري: direction = output
    void __iomem *reg_dir_in;     // ◄── اختياري: direction = input

    // State tracking in RAM
    unsigned long sdata;          // ◄── shadow of data register
    unsigned long sdir;           // ◄── shadow of direction (1=output)

    raw_spinlock_t lock;          // ◄── protects sdata + sdir
    int bits;                     // ◄── 8, 16, 32, or 64
    bool be_bits;                 // ◄── big-endian bit order?
    bool dir_unreadable;          // ◄── register can't be read back
    bool pinctrl;                 // ◄── delegate direction to pinctrl
};
```

الـ `container_of` macro بيخلي الـ framework يتنقل بين الـ `gpio_chip` و الـ `gpio_generic_chip` بسهولة:

```c
static inline struct gpio_generic_chip *
to_gpio_generic_chip(struct gpio_chip *gc)
{
    return container_of(gc, struct gpio_generic_chip, gc);
}
```

---

### الـ Structs وعلاقتها ببعض

```
  ┌─────────────────────────────────────────────────────┐
  │          struct gpio_generic_chip_config             │
  │  (config data يتحدد من device tree أو board file)   │
  │                                                      │
  │  dev → parent device                                 │
  │  sz  → register width in bytes (1/2/4/8)            │
  │  dat → MMIO addr of data register                   │
  │  set → MMIO addr of set register (optional)         │
  │  clr → MMIO addr of clear register (optional)       │
  │  dirout → MMIO addr of dir-out register (optional)  │
  │  dirin  → MMIO addr of dir-in  register (optional)  │
  │  flags → GPIO_GENERIC_BIG_ENDIAN | ...               │
  └─────────────────┬───────────────────────────────────┘
                    │  passed to
                    ▼
  ┌─────────────────────────────────────────────────────┐
  │         gpio_generic_chip_init()                     │
  │   ينشئ ويملأ struct gpio_generic_chip               │
  └─────────────────┬───────────────────────────────────┘
                    │  initializes
                    ▼
  ┌─────────────────────────────────────────────────────┐
  │           struct gpio_generic_chip                   │
  │                                                      │
  │  ┌───────────────────────────────────────┐          │
  │  │         struct gpio_chip (gc)          │◄─────────┼── gpiolib يتعامل معاه
  │  │                                        │          │
  │  │  gc.set ─────────────────────────────►│──────────┼──► gpio_mmio_set()
  │  │  gc.get ─────────────────────────────►│──────────┼──► gpio_mmio_get()
  │  │  gc.direction_input ────────────────►│──────────┼──► gpio_mmio_dir_in()
  │  │  gc.direction_output ───────────────►│──────────┼──► gpio_mmio_dir_out_*()
  │  │  gc.get_direction ──────────────────►│──────────┼──► gpio_mmio_get_dir()
  │  │  gc.get_multiple ───────────────────►│──────────┼──► gpio_mmio_get_multiple*()
  │  │  gc.set_multiple ───────────────────►│──────────┼──► gpio_mmio_set_multiple*()
  │  └───────────────────────────────────────┘          │
  │                                                      │
  │  read_reg  ──────────────────────────────────────────┼──► gpio_mmio_read8/16/32/64
  │  write_reg ──────────────────────────────────────────┼──► gpio_mmio_write8/16/32/64
  │                                                      │
  │  reg_dat / reg_set / reg_clr                         │
  │  reg_dir_out / reg_dir_in                            │
  │  sdata / sdir / lock / bits / be_bits                │
  └─────────────────────────────────────────────────────┘
```

---

### أنواع الـ Register Layouts المدعومة

الـ driver بيدعم ثلاث سيناريوهات لكتابة GPIO outputs:

#### السيناريو 1 — Single Data Register

```
 GPIO Controller Hardware
 ┌───────────────────────┐
 │  DAT register         │  ← قراءة وكتابة من/إلى نفس المكان
 │  bit[N] = 1 → HIGH    │
 │  bit[N] = 0 → LOW     │
 └───────────────────────┘

 Driver Operation (gpio_mmio_set):
   lock()
   if val: sdata |= mask
   else:   sdata &= ~mask
   write_reg(reg_dat, sdata)  ← لازم spinlock لأن read-modify-write
   unlock()
```

**المشكلة هنا:** قراءة ثم تعديل ثم كتابة (RMW) — لازم lock عشان thread-safe.

#### السيناريو 2 — Set/Clear Pair

```
 GPIO Controller Hardware
 ┌───────────────────────┐   ┌───────────────────────┐
 │  SET register         │   │  CLR register         │
 │  write bit[N]=1 → HIGH│   │  write bit[N]=1 → LOW │
 └───────────────────────┘   └───────────────────────┘

 Driver Operation (gpio_mmio_set_with_clear):
   if val: write_reg(reg_set, mask)   ← بدون lock! أتوميك على hardware
   else:   write_reg(reg_clr, mask)
```

**الميزة:** مفيش RMW — كل عملية كتابة أتوميك على hardware، مفيش حاجة للـ lock.

#### السيناريو 3 — Set Register Only (no clear)

```
 GPIO Controller Hardware
 ┌───────────────────────┐
 │  SET register         │  ← للـ output فقط
 │  DAT register         │  ← للـ input فقط
 └───────────────────────┘

 Driver Operation (gpio_mmio_set_set):
   lock()
   if val: sdata |= mask
   else:   sdata &= ~mask
   write_reg(reg_set, sdata)  ← يكتب الـ shadow على reg_set
   unlock()
```

---

### الـ Big-Endian Bit Order — مشكلة دقيقة

الهاردوير الـ big-endian بيعكس ترتيب الـ bits: GPIO line 0 يكون في bit 31 مش bit 0.

```
Little-endian bit order (normal):
  register bits:  [31][30]...[1][0]
  GPIO line:       31  30  ...  1  0
  BIT(line) ✓

Big-endian bit order:
  register bits:  [31][30]...[1][0]
  GPIO line:        0   1  ...  30  31
  BIT(bits-1-line) ✓

gpio_mmio_line2mask():
  if (chip->be_bits)
      return BIT(chip->bits - 1 - line);  // ← عكس
  return BIT(line);                        // ← عادي
```

---

### الـ Shadow Registers — ليه محتاجينهم؟

الهاردوير في بعض الأحيان **مش بيرجع نفس القيمة** اللي اتكتبت فيه (write-only registers أو output-only registers). الحل هو الـ **shadow register** — نسخة في RAM بتتزامن مع الهاردوير.

```
  RAM                          MMIO Hardware
  ┌──────────┐                ┌──────────────┐
  │  sdata   │◄── sync ──────►│  reg_dat     │
  │(32 bits) │                │(32 bits MMIO)│
  └──────────┘                └──────────────┘

  عند set bit 5:
    1. lock
    2. sdata |= BIT(5)        ← تعديل الـ shadow
    3. write_reg(reg_dat, sdata) ← كتابة الكل للهاردوير
    4. unlock

  عند get direction (dir_unreadable=true):
    return !!(chip->sdir & mask)  ← من RAM مش من hardware
```

---

### إزاي الـ setup بيحصل — flow كامل

```
gpio_mmio_pdev_probe()
    │
    ├── platform_get_resource_byname("dat") → get MMIO address
    ├── platform_get_resource_byname("set") → optional
    ├── platform_get_resource_byname("clr") → optional
    ├── platform_get_resource_byname("dirout") → optional
    ├── platform_get_resource_byname("dirin")  → optional
    │
    ├── devm_kzalloc(gpio_generic_chip)
    │
    └── gpio_generic_chip_init()
            │
            ├── validate: sz must be power of 2
            ├── chip->bits = sz * 8
            ├── raw_spin_lock_init(&chip->lock)
            ├── gpiochip_get_ngpios() → or default to bits
            │
            ├── gpio_mmio_setup_io()
            │       ├── assign reg_dat, reg_set, reg_clr
            │       └── assign gc->set / gc->get callbacks
            │           based on which registers exist
            │
            ├── gpio_mmio_setup_accessors()
            │       └── assign read_reg/write_reg based on
            │           chip->bits: 8→read8, 16→read16...
            │           + big-endian byte order variants
            │
            ├── gpio_mmio_setup_direction()
            │       └── assign gc->direction_input/output/get_direction
            │           based on dirout/dirin/flags
            │
            ├── chip->sdata = read initial value from hardware
            └── chip->sdir  = read initial direction from hardware
```

---

### الـ Direction Handling — التفاصيل

للـ direction في 3 حالات:

| Hardware Support | Callbacks المختارة |
|---|---|
| لا يوجد direction register (fixed direction) | `gpio_mmio_simple_dir_in/out` |
| `dirout` register موجود (1=output) | `gpio_mmio_dir_in/out/get_dir` |
| `dirin` register موجود (1=input) | نفس callbacks بس logic معكوسة |

الـ `gpio_mmio_get_dir()` بيتعامل مع الحالات الثلاث:

```c
static int gpio_mmio_get_dir(struct gpio_chip *gc, unsigned int gpio)
{
    struct gpio_generic_chip *chip = to_gpio_generic_chip(gc);

    /* إذا الـ register مش قابل للقراءة، رجع من الـ shadow */
    if (chip->dir_unreadable) {
        if (chip->sdir & gpio_mmio_line2mask(gc, gpio))
            return GPIO_LINE_DIRECTION_OUT;
        return GPIO_LINE_DIRECTION_IN;
    }

    /* إذا في dirout register، اقرأ منه مباشرة */
    if (chip->reg_dir_out) {
        if (chip->read_reg(chip->reg_dir_out) & gpio_mmio_line2mask(gc, gpio))
            return GPIO_LINE_DIRECTION_OUT;
        return GPIO_LINE_DIRECTION_IN;
    }

    /* إذا في dirin register، 1=input يعني output = NOT bit */
    if (chip->reg_dir_in)
        if (!(chip->read_reg(chip->reg_dir_in) & gpio_mmio_line2mask(gc, gpio)))
            return GPIO_LINE_DIRECTION_OUT;

    return GPIO_LINE_DIRECTION_IN;
}
```

---

### ما بيملكه الـ Framework vs ما بيفوضه للـ Driver

| المسؤولية | gpio-mmio | gpio_chip / gpiolib |
|---|---|---|
| تحديد register addresses | ملكه — من config struct | — |
| اختيار read/write width | ملكه — setup_accessors() | — |
| shadow registers (sdata/sdir) | ملكه | — |
| spinlock للـ RMW operations | ملكه | — |
| big-endian bit mapping | ملكه | — |
| تسجيل الـ chip في الـ kernel | — | `devm_gpiochip_add_data()` |
| IRQ support | — | gpiolib_irqchip |
| pinctrl integration | delegation via flags | pinctrl subsystem |
| device tree parsing (ngpio) | partial — `gpiochip_get_ngpios()` | gpiolib |
| user-space interface (sysfs/chardev) | — | gpiolib core |

**ملحوظة مهمة:** الـ pinctrl subsystem (مذكور في الكود) هو subsystem منفصل بيتعامل مع إعداد الـ pin multiplexing — إيه الوظيفة اللي الـ pin بيعملها (GPIO أم UART أم SPI). الـ gpio-mmio بيتكامل معاه اختياريا عبر `GPIO_GENERIC_PINCTRL_BACKEND` flag.

---

### Real-World Devices المدعومة

من الـ `of_device_id` table في الكود:

| Compatible String | Hardware |
|---|---|
| `brcm,bcm6345-gpio` | Broadcom BCM6345 (روتر home) |
| `wd,mbl-gpio` | Western Digital My Book Live (NAS) |
| `ni,169445-nand-gpio` | National Instruments NAND GPIO |
| `intel,ixp4xx-expansion-bus-mmio-gpio` | Intel IXP4xx (network processor) |
| `opencores,gpio` | OpenCores generic GPIO IP core (للـ FPGA) |

كلهم بيشتركوا في نفس المبدأ: GPIO = bits في MMIO register.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Config Options — Cheatsheet

#### flags بتاعة `gpio_generic_chip_config`

| Flag | Value | المعنى |
|------|-------|--------|
| `GPIO_GENERIC_BIG_ENDIAN` | `BIT(0)` | الـ bit order معكوس — bit 31 = line 0 |
| `GPIO_GENERIC_UNREADABLE_REG_SET` | `BIT(1)` | الـ `reg_set` مش ممكن تقراه (write-only) |
| `GPIO_GENERIC_UNREADABLE_REG_DIR` | `BIT(2)` | الـ direction register مش ممكن تقراه |
| `GPIO_GENERIC_BIG_ENDIAN_BYTE_ORDER` | `BIT(3)` | byte order big-endian (يستخدم `ioread/iowrite` be variants) |
| `GPIO_GENERIC_READ_OUTPUT_REG_SET` | `BIT(4)` | الـ `reg_set` بيعكس القيمة الفعلية للـ output |
| `GPIO_GENERIC_NO_OUTPUT` | `BIT(5)` | الـ chip input-only، مفيش write |
| `GPIO_GENERIC_NO_SET_ON_INPUT` | `BIT(6)` | لما بتحول line لـ input، متكتبش فيها قيمة |
| `GPIO_GENERIC_PINCTRL_BACKEND` | `BIT(7)` | استخدم pinctrl لتغيير الـ direction |
| `GPIO_GENERIC_NO_INPUT` | `BIT(8)` | الـ chip output-only، مفيش read |

#### الـ Register Modes المدعومة

| `reg_dat` | `reg_set` | `reg_clr` | الوضع |
|-----------|-----------|-----------|--------|
| ✓ فقط | — | — | single data register |
| ✓ | ✓ | — | set register منفصل، clear عن طريق كتابة 0 |
| ✓ | ✓ | ✓ | set/clear pair كاملة |

#### الـ Direction Register Modes

| `reg_dir_out` | `reg_dir_in` | الوضع |
|---------------|--------------|--------|
| — | — | fixed direction (bidirectional simple) |
| ✓ | — | 1 = output |
| — | ✓ | 1 = input |
| ✓ | ✓ | كلتيهما موجودتان، يتم synchronization |

#### Register Widths المدعومة

| `sz` بالبايت | `bits` | ملاحظة |
|-------------|--------|--------|
| 1 | 8 | `readb`/`writeb` |
| 2 | 16 | `readw`/`writew` أو `ioread16be`/`iowrite16be` |
| 4 | 32 | `readl`/`writel` أو `ioread32be`/`iowrite32be` |
| 8 | 64 | `readq`/`writeq` — little-endian فقط |

#### الـ Kconfig المرتبطة

| Option | الأثر |
|--------|-------|
| `CONFIG_GPIO_GENERIC_PLATFORM` | يضيف الـ platform driver (`gpio_mmio_pdev_probe`) |
| `CONFIG_GPIOLIB_IRQCHIP` | يفعّل الـ IRQ support داخل `gpio_chip` |
| `CONFIG_PINCTRL` | يفعّل الـ pinctrl integration |

---

### 1. الـ Structs المهمة

#### `struct gpio_generic_chip_config`
> **الهدف:** بيانات الـ configuration اللي بيبعتها الـ caller لـ `gpio_generic_chip_init()` — كأنها constructor arguments.

| Field | Type | المعنى |
|-------|------|--------|
| `dev` | `struct device *` | الـ parent device (إلزامي) |
| `sz` | `unsigned long` | حجم الـ register بالبايت (1/2/4/8) |
| `dat` | `void __iomem *` | عنوان register القراءة |
| `set` | `void __iomem *` | عنوان register الكتابة (set) |
| `clr` | `void __iomem *` | عنوان register المسح (clear) — ممكن NULL |
| `dirout` | `void __iomem *` | عنوان direction-out register — ممكن NULL |
| `dirin` | `void __iomem *` | عنوان direction-in register — ممكن NULL |
| `flags` | `unsigned long` | combination من الـ `GPIO_GENERIC_*` flags |

**الاتصالات:** يتم استهلاكه كاملاً داخل `gpio_generic_chip_init()` ومش محتاج يبقى موجود بعد الـ init.

---

#### `struct gpio_generic_chip`
> **الهدف:** الـ main object بتاع الـ driver — بيغلف `gpio_chip` ويضيف كل الـ MMIO state.

| Field | Type | المعنى |
|-------|------|--------|
| `gc` | `struct gpio_chip` | الـ base object (أول field — لازم يفضل أول) |
| `read_reg` | `fn ptr` | accessor للقراءة من الـ MMIO (8/16/32/64 bit) |
| `write_reg` | `fn ptr` | accessor للكتابة على الـ MMIO |
| `be_bits` | `bool` | هل الـ bit order معكوس (big-endian bit order)؟ |
| `reg_dat` | `void __iomem *` | عنوان الـ data register في الذاكرة |
| `reg_set` | `void __iomem *` | عنوان الـ set register |
| `reg_clr` | `void __iomem *` | عنوان الـ clear register |
| `reg_dir_out` | `void __iomem *` | عنوان direction-output register |
| `reg_dir_in` | `void __iomem *` | عنوان direction-input register |
| `dir_unreadable` | `bool` | الـ direction register مش قابل للقراءة |
| `pinctrl` | `bool` | هل بستخدم pinctrl backend؟ |
| `bits` | `int` | عدد الـ bits في الـ register (= sz × 8) |
| `lock` | `raw_spinlock_t` | يحمي `sdata` و`sdir` |
| `sdata` | `unsigned long` | shadow copy للـ data register |
| `sdir` | `unsigned long` | shadow copy للـ direction (1 = output) |

**الاتصالات:**
- يحتوي على `gpio_chip` كأول field — يسمح بـ `container_of` في `to_gpio_generic_chip()`
- الـ `read_reg`/`write_reg` بيتم assign قيمتهم في `gpio_mmio_setup_accessors()`
- الـ `lock` بيحمي `sdata` و`sdir` اللي بيتحدّثوا في الـ set/direction functions

---

#### `struct gpio_chip`
> **الهدف:** الـ abstract interface الـ gpiolib بيشوفه — callbacks فقط + metadata.

| Field | Type | المعنى |
|-------|------|--------|
| `label` | `const char *` | اسم الـ chip في sysfs/debugfs |
| `parent` | `struct device *` | الـ parent device |
| `base` | `int` | رقم أول GPIO (-1 = dynamic) |
| `ngpio` | `u16` | عدد الـ GPIO lines |
| `request` | `fn ptr` | hook عند طلب GPIO |
| `free` | `fn ptr` | hook عند تحرير GPIO |
| `get_direction` | `fn ptr` | يرجع الـ direction (0=out, 1=in) |
| `direction_input` | `fn ptr` | يحول line لـ input |
| `direction_output` | `fn ptr` | يحول line لـ output مع قيمة |
| `get` | `fn ptr` | يقرأ قيمة line واحدة |
| `get_multiple` | `fn ptr` | يقرأ قيم lines متعددة بـ bitmask |
| `set` | `fn ptr` | يكتب قيمة line واحدة |
| `set_multiple` | `fn ptr` | يكتب قيم lines متعددة بـ bitmask |

**الاتصالات:**
- الـ `gpio-mmio.c` بيفيل كل الـ callbacks دي في `gpio_mmio_setup_io()` و`gpio_mmio_setup_direction()`
- الـ gpiolib core بيستخدم `gpio_chip` مباشرة عبر `devm_gpiochip_add_data()`

---

### 2. مخطط علاقات الـ Structs

```
  ┌─────────────────────────────────────────────────────────────┐
  │             struct gpio_generic_chip                        │
  │                                                             │
  │  ┌──────────────────────────────────────────────────────┐  │
  │  │  struct gpio_chip gc   ◄── container_of() ──────────  │  │
  │  │  (أول field دايماً)                                   │  │
  │  └───────────┬──────────────────────────────────────────┘  │
  │              │ callbacks assigned by gpio_mmio_setup_*()   │
  │  ┌───────────▼──────────────────────────────┐              │
  │  │  fn: get / set / direction_input/output  │              │
  │  │  fn: get_multiple / set_multiple         │              │
  │  │  fn: get_direction / request / free      │              │
  │  └──────────────────────────────────────────┘              │
  │                                                             │
  │  void __iomem *reg_dat  ──────────► [MMIO hardware reg]    │
  │  void __iomem *reg_set  ──────────► [MMIO hardware reg]    │
  │  void __iomem *reg_clr  ──────────► [MMIO hardware reg]    │
  │  void __iomem *reg_dir_out ────────► [MMIO hardware reg]   │
  │  void __iomem *reg_dir_in ─────────► [MMIO hardware reg]   │
  │                                                             │
  │  unsigned long (*read_reg)(void __iomem *)                  │
  │      └──► gpio_mmio_read8 / read16 / read32 / read64       │
  │           (أو BE variants)                                  │
  │                                                             │
  │  void (*write_reg)(void __iomem *, unsigned long)           │
  │      └──► gpio_mmio_write8 / write16 / write32 / write64   │
  │                                                             │
  │  raw_spinlock_t lock  ─── يحمي ──► sdata, sdir            │
  │  unsigned long sdata  (shadow of reg_dat/reg_set)           │
  │  unsigned long sdir   (shadow direction: 1=output)          │
  └─────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────┐
  │         struct gpio_generic_chip_config (مؤقت)              │
  │   dev, sz, dat, set, clr, dirout, dirin, flags              │
  │                  │                                           │
  │                  ▼ تتهضم في gpio_generic_chip_init()        │
  └───────────────────────────────────────────────────────────── ┘

  ┌────────────────────────────────────┐
  │  struct platform_device            │
  │  (platform driver path only)       │
  │    │                               │
  │    ▼ gpio_mmio_pdev_probe()        │
  │  platform_get_resource_byname()    │
  │  devm_ioremap_resource()           │
  │    │                               │
  │    ▼ يملأ gpio_generic_chip_config │
  │    ▼ يستدعي gpio_generic_chip_init │
  │    ▼ devm_gpiochip_add_data()      │
  └────────────────────────────────────┘
```

---

### 3. دورة الحياة — من الإنشاء للتدمير

```
CREATION
────────
  platform_device OR
  custom driver code
       │
       ▼
  allocate struct gpio_generic_chip
  (devm_kzalloc أو static)
       │
       ▼
  fill gpio_generic_chip_config {
    .dev, .sz,
    .dat, .set, .clr,
    .dirout, .dirin,
    .flags
  }
       │
       ▼
  gpio_generic_chip_init(chip, cfg)
  ├── validate sz is power-of-2
  ├── chip->bits = sz * 8
  ├── raw_spin_lock_init(&chip->lock)
  ├── gc->base = -1 (dynamic)
  ├── chip->be_bits = flags & BIG_ENDIAN
  ├── gpiochip_get_ngpios() → gc->ngpio
  ├── gpio_mmio_setup_io()
  │     assign gc->get, gc->set,
  │            gc->get_multiple, gc->set_multiple
  ├── gpio_mmio_setup_accessors()
  │     assign chip->read_reg, chip->write_reg
  ├── gpio_mmio_setup_direction()
  │     assign gc->direction_input/output
  │            gc->get_direction
  ├── chip->sdata = read initial hw state
  └── chip->sdir  = read initial direction

REGISTRATION
────────────
       ▼
  devm_gpiochip_add_data(&dev, &gc, NULL)
  └── gpiolib core registers chip
  └── sysfs/debugfs entries created
  └── IRQ domain setup (if configured)

RUNTIME
───────
  consumer calls gpiod_get() / gpio_request()
       │
       ▼
  gc->request() → gpio_mmio_request()
       │
       ▼
  gc->direction_input() or gc->direction_output()
  gc->get() / gc->set() / gc->get_multiple() / gc->set_multiple()

TEARDOWN
────────
  device_unregister() or driver unbind
       │
       ▼
  devm cleanup runs automatically:
  ├── devm_gpiochip_add_data → gpiochip_remove()
  │     gpiolib unregisters chip
  │     IRQ domain torn down
  └── devm_ioremap_resource → iounmap()
        MMIO mappings released
```

---

### 4. مخططات الـ Call Flow

#### 4.1 gpio_set (single pin)

```
consumer: gpiod_set_value(desc, 1)
  │
  ▼ gpiolib core
  gc->set(gc, offset, 1)
  │
  ├─[if reg_set && reg_clr]──► gpio_mmio_set_with_clear()
  │                              chip->write_reg(chip->reg_set, mask)
  │                              (no lock needed — atomic hw operation)
  │
  ├─[if reg_set only]────────► gpio_mmio_set_set()
  │                              guard(raw_spinlock_irqsave)(&chip->lock)
  │                              chip->sdata |= mask
  │                              chip->write_reg(chip->reg_set, chip->sdata)
  │
  └─[if dat only]─────────────► gpio_mmio_set()
                                 guard(raw_spinlock_irqsave)(&chip->lock)
                                 chip->sdata |= / &= ~mask
                                 chip->write_reg(chip->reg_dat, chip->sdata)
```

#### 4.2 gpio_get (single pin)

```
consumer: gpiod_get_value(desc)
  │
  ▼ gpiolib core
  gc->get(gc, offset)
  │
  ├─[READ_OUTPUT_REG_SET flag]─► gpio_mmio_get_set()
  │   if sdir has bit set → read from reg_set (output)
  │   else                 → read from reg_dat (input)
  │   return !!(val & mask)
  │
  └─[default]──────────────────► gpio_mmio_get()
      chip->read_reg(chip->reg_dat) & mask
      (no lock — single read)
```

#### 4.3 direction_output

```
consumer: gpiod_direction_output(desc, val)
  │
  ▼ gpiolib core
  gc->direction_output(gc, offset, val)
  │
  ├─[NO_SET_ON_INPUT flag]────► gpio_mmio_dir_out_dir_first()
  │   gpio_mmio_dir_out()         ← set direction register first
  │   gc->set()                   ← then set value
  │
  └─[default]─────────────────► gpio_mmio_dir_out_val_first()
      gc->set()                   ← set value first
      gpio_mmio_dir_out()         ← then set direction register
      │
      ▼ gpio_mmio_dir_out()
        guard(raw_spinlock_irqsave)(&chip->lock)
        chip->sdir |= mask
        if reg_dir_in:  write_reg(reg_dir_in, ~sdir)
        if reg_dir_out: write_reg(reg_dir_out, sdir)
      │
      ▼ gpio_mmio_dir_return(gc, gpio, true)
        if chip->pinctrl:
          pinctrl_gpio_direction_output()
```

#### 4.4 get_multiple (native endian)

```
consumer: gpiod_get_array_value(...)
  │
  ▼ gpiolib core
  gc->get_multiple(gc, mask, bits)
  │
  ▼ gpio_mmio_get_multiple()
    *bits &= ~*mask              /* clear bits in requested range */
    *bits |= read_reg(reg_dat) & *mask
    return 0
```

#### 4.5 set_multiple (with set/clear registers)

```
consumer: gpiod_set_array_value(...)
  │
  ▼ gpiolib core
  gc->set_multiple(gc, mask, bits)
  │
  ▼ gpio_mmio_set_multiple_with_clear()
    gpio_mmio_multiple_get_masks()
      iterate set_bit positions in mask:
        if bit in bits → set_mask |= line2mask(i)
        else           → clear_mask |= line2mask(i)
    if set_mask:   write_reg(reg_set, set_mask)
    if clear_mask: write_reg(reg_clr, clear_mask)
    (no lock needed — separate set/clear regs are atomic)
```

#### 4.6 platform probe flow

```
kernel detects platform_device "basic-mmio-gpio"
  │
  ▼ gpio_mmio_pdev_probe(pdev)
    platform_get_resource_byname(pdev, "dat") → r
    sz = resource_size(r)

    gpio_mmio_map(pdev, "dat", sz)   → dat (devm_ioremap)
    gpio_mmio_map(pdev, "set", sz)   → set (NULL if not present)
    gpio_mmio_map(pdev, "clr", sz)   → clr (NULL if not present)
    gpio_mmio_map(pdev, "dirout", sz)→ dirout
    gpio_mmio_map(pdev, "dirin", sz) → dirin

    devm_kzalloc(gen_gc)

    if device_is_big_endian(dev):
      flags |= GPIO_GENERIC_BIG_ENDIAN_BYTE_ORDER

    gpio_generic_chip_init(gen_gc, &config)
      (see lifecycle diagram above)

    devm_gpiochip_add_data(&pdev->dev, &gen_gc->gc, NULL)
      → chip registered with gpiolib
```

---

### 5. استراتيجية الـ Locking

#### الـ Lock الموجود

```
raw_spinlock_t  chip->lock
```

الـ `raw_spinlock_t` يُستخدم بدلاً من `spinlock_t` العادي لأن الـ GPIO set ممكن يتنادى من **real-time contexts** — الـ `raw_spinlock` مش بيتأثر بالـ PREEMPT_RT patching.

#### ما اللي يحميه الـ lock؟

| Resource | محمي بالـ lock؟ | السبب |
|----------|----------------|-------|
| `chip->sdata` | نعم | read-modify-write atomic على shadow register |
| `chip->sdir` | نعم | read-modify-write atomic على shadow direction |
| كتابة `reg_dat` / `reg_set` + `sdata` معاً | نعم | لازم يبقوا consistent |
| كتابة `reg_dir_out` / `reg_dir_in` + `sdir` | نعم | نفس السبب |
| `read_reg(reg_dat)` بدون modify | لا | read-only لا يحتاج lock |
| `write_reg(reg_set, mask)` مع set/clear pair | لا | كل write مستقلة — no RMW |

#### ترتيب الـ Locking (Lock Ordering)

الـ `gpio-mmio.c` عنده lock واحد بس (`chip->lock`)، فمفيش خطر deadlock داخلي. لكن لازم تعرف:

```
المستوى الأعلى (callers):
  gpiod_set_value()         ← بياخد gpiod_lock()   (gpiolib mutex)
    │
    ▼ بينادي gc->set()
      gpio_mmio_set()
        guard(raw_spinlock_irqsave)(&chip->lock)  ← chip->lock
```

الترتيب دايماً: **gpiolib lock → chip->lock** — مش مسموح العكس أبداً.

#### متى بيتاخد الـ lock ومتى لا؟

```c
/* يحتاج lock — RMW على sdata */
static int gpio_mmio_set(struct gpio_chip *gc, unsigned int gpio, int val)
{
    guard(raw_spinlock_irqsave)(&chip->lock);
    chip->sdata |= mask;           /* modify shadow */
    chip->write_reg(chip->reg_dat, chip->sdata); /* write hw */
}

/* لا يحتاج lock — كل write مستقلة، لا يوجد shadow */
static int gpio_mmio_set_with_clear(struct gpio_chip *gc, unsigned int gpio, int val)
{
    if (val)
        chip->write_reg(chip->reg_set, mask);  /* atomic set */
    else
        chip->write_reg(chip->reg_clr, mask);  /* atomic clear */
}
```

#### الـ `guard()` و`scoped_guard()` — الـ Cleanup-based Locking

الـ driver بيستخدم الـ `guard()` macro الجديدة (من `<linux/cleanup.h>`) بدلاً من `spin_lock_irqsave` / `spin_unlock_irqrestore` اليدوية:

```c
/* lock يُفتح عند دخول الـ scope ويُقفل تلقائياً عند الخروج */
guard(raw_spinlock_irqsave)(&chip->lock);

/* نفس الفكرة لكن للـ block محدد */
scoped_guard(raw_spinlock_irqsave, &chip->lock) {
    chip->sdir &= ~mask;
    chip->write_reg(chip->reg_dir_in, ~chip->sdir);
}
```

الميزة: مستحيل تنسى الـ unlock حتى لو رجعت بـ `return` في النص.

---

### ملخص العلاقات

```
struct gpio_generic_chip_config
        │
        │ (consumed once at init time)
        ▼
struct gpio_generic_chip
  ├── struct gpio_chip gc          ◄── gpiolib يشوف cده فقط
  │       ├── gc->get              = gpio_mmio_get[_set][_multiple]
  │       ├── gc->set              = gpio_mmio_set[_with_clear][_set]
  │       ├── gc->direction_input  = gpio_mmio_dir_in[_simple][_err]
  │       ├── gc->direction_output = gpio_mmio_dir_out_[dir|val]_first
  │       └── gc->get_direction    = gpio_mmio_get_dir
  │
  ├── read_reg  → gpio_mmio_read{8,16,32,64}[be]
  ├── write_reg → gpio_mmio_write{8,16,32,64}[be]
  │
  ├── reg_dat/set/clr/dir_out/dir_in  →  MMIO hardware registers
  │
  ├── raw_spinlock_t lock
  ├── unsigned long sdata  (shadow of output register)
  └── unsigned long sdir   (shadow of direction: 1=output)
```
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Group 1: MMIO Read/Write Accessors

| Function | Width | Endian | I/O Primitive |
|---|---|---|---|
| `gpio_mmio_read8` | 8-bit | N/A | `readb()` |
| `gpio_mmio_write8` | 8-bit | N/A | `writeb()` |
| `gpio_mmio_read16` | 16-bit | LE | `readw()` |
| `gpio_mmio_write16` | 16-bit | LE | `writew()` |
| `gpio_mmio_read16be` | 16-bit | BE | `ioread16be()` |
| `gpio_mmio_write16be` | 16-bit | BE | `iowrite16be()` |
| `gpio_mmio_read32` | 32-bit | LE | `readl()` |
| `gpio_mmio_write32` | 32-bit | LE | `writel()` |
| `gpio_mmio_read32be` | 32-bit | BE | `ioread32be()` |
| `gpio_mmio_write32be` | 32-bit | BE | `iowrite32be()` |
| `gpio_mmio_read64` | 64-bit | LE only | `readq()` |
| `gpio_mmio_write64` | 64-bit | LE only | `writeq()` |

#### Group 2: Bit Mask Helper

| Function | الوظيفة |
|---|---|
| `gpio_mmio_line2mask` | تحويل رقم الـ GPIO line لـ bitmask (مع دعم BE bit order) |

#### Group 3: GPIO Get (Read Value)

| Function | الوظيفة |
|---|---|
| `gpio_mmio_get` | قراءة قيمة pin واحد من `reg_dat` |
| `gpio_mmio_get_set` | قراءة pin واحد — outputs من `reg_set`، inputs من `reg_dat` |
| `gpio_mmio_get_multiple` | قراءة عدة pins دفعة واحدة (native endian) |
| `gpio_mmio_get_multiple_be` | قراءة عدة pins بـ big endian bit order |
| `gpio_mmio_get_set_multiple` | قراءة عدة pins مع split بين outputs (reg_set) وينputs (reg_dat) |

#### Group 4: GPIO Set (Write Value)

| Function | الوظيفة |
|---|---|
| `gpio_mmio_set_none` | stub — لا يعمل حاجة (input-only GPIO) |
| `gpio_mmio_set` | كتابة قيمة pin في `reg_dat` مع shadow + lock |
| `gpio_mmio_set_with_clear` | كتابة باستخدام set/clear register pair |
| `gpio_mmio_set_set` | كتابة قيمة pin في `reg_set` مع shadow + lock |
| `gpio_mmio_set_multiple` | كتابة عدة pins في `reg_dat` |
| `gpio_mmio_set_multiple_set` | كتابة عدة pins في `reg_set` |
| `gpio_mmio_set_multiple_with_clear` | كتابة عدة pins باستخدام set/clear pair |
| `gpio_mmio_multiple_get_masks` | helper يحسب set_mask و clear_mask من mask+bits |
| `gpio_mmio_set_multiple_single_reg` | helper مشترك لـ set_multiple مع reg واحد |

#### Group 5: Direction Control

| Function | الوظيفة |
|---|---|
| `gpio_mmio_dir_in` | يضبط pin كـ input مع تحديث `reg_dir_in`/`reg_dir_out` |
| `gpio_mmio_dir_out` | helper داخلي يكتب direction registers |
| `gpio_mmio_dir_out_dir_first` | يضبط direction أولاً ثم يكتب القيمة |
| `gpio_mmio_dir_out_val_first` | يكتب القيمة أولاً ثم يضبط direction |
| `gpio_mmio_simple_dir_in` | input-only chip — pinctrl callback فقط |
| `gpio_mmio_simple_dir_out` | output-only chip — set + pinctrl callback |
| `gpio_mmio_dir_in_err` | stub يرجع `-EINVAL` (output-only GPIO) |
| `gpio_mmio_dir_out_err` | stub يرجع `-EINVAL` (input-only GPIO) |
| `gpio_mmio_get_dir` | قراءة اتجاه الـ pin من hardware أو shadow |
| `gpio_mmio_dir_return` | helper مشترك يستدعي pinctrl إن وُجد |

#### Group 6: Setup & Registration

| Function | الوظيفة |
|---|---|
| `gpio_mmio_setup_accessors` | يختار read/write function pointers بناءً على width + endian |
| `gpio_mmio_setup_io` | يضبط get/set callbacks بناءً على تكوين الـ registers |
| `gpio_mmio_setup_direction` | يضبط direction callbacks بناءً على تكوين الـ registers |
| `gpio_mmio_request` | callback عند طلب GPIO line من المستخدم |
| `gpio_generic_chip_init` | **Entry point** — يهيئ الـ chip كاملاً |

#### Group 7: Platform Driver (CONFIG_GPIO_GENERIC_PLATFORM)

| Function | الوظيفة |
|---|---|
| `gpio_mmio_map` | تعيين MMIO resource بالاسم مع size sanity check |
| `gpio_mmio_pdev_probe` | probe function للـ platform driver |

---

### Group 1: MMIO Read/Write Accessors

هذه الـ functions هي الطبقة الأدنى في الـ driver — function pointers تُخزَّن في `chip->read_reg` و `chip->write_reg` وتُستدعى في كل عملية GPIO. اختيارها يتم مرة واحدة في `gpio_mmio_setup_accessors()`.

التصميم يعزل باقي الكود عن حجم الـ register (8/16/32/64 bit) وعن الـ byte order، مما يجعل بقية الـ functions generic تماماً.

---

#### `gpio_mmio_read8` / `gpio_mmio_write8`

```c
static unsigned long gpio_mmio_read8(void __iomem *reg);
static void gpio_mmio_write8(void __iomem *reg, unsigned long data);
```

يقرأ/يكتب byte واحد من/إلى MMIO address باستخدام `readb()`/`writeb()`. الـ 8-bit registers لا تحتاج byte-swap لأنها single-byte.

- **Parameters:** `reg` — الـ MMIO address المُعيَّن بالفعل / `data` — القيمة المراد كتابتها
- **Return:** القيمة المقروءة كـ `unsigned long`
- **Key details:** `readb`/`writeb` تضمن الـ memory barrier المطلوب على كل architecture

---

#### `gpio_mmio_read16` / `gpio_mmio_write16`

```c
static unsigned long gpio_mmio_read16(void __iomem *reg);
static void gpio_mmio_write16(void __iomem *reg, unsigned long data);
```

يستخدم `readw()`/`writew()` — native 16-bit LE access. على ARM هذه تُترجَم لـ `ldrh`/`strh`.

---

#### `gpio_mmio_read16be` / `gpio_mmio_write16be`

```c
static unsigned long gpio_mmio_read16be(void __iomem *reg);
static void gpio_mmio_write16be(void __iomem *reg, unsigned long data);
```

يستخدم `ioread16be()`/`iowrite16be()` اللي تعمل byte-swap على LE architectures. تُستخدَم لـ controllers زي بعض FPGA designs المبنية على big endian bus.

---

#### `gpio_mmio_read32` / `gpio_mmio_write32` / `*32be`

نفس المنطق لـ 32-bit. الـ BE variant غير متاحة لـ 64-bit لأن `ioread64be` غير موجودة في الـ kernel.

---

#### `gpio_mmio_read64` / `gpio_mmio_write64`

```c
#if BITS_PER_LONG >= 64
static unsigned long gpio_mmio_read64(void __iomem *reg);
static void gpio_mmio_write64(void __iomem *reg, unsigned long data);
#endif
```

متاحة فقط على 64-bit architectures. تستخدم `readq()`/`writeq()`. الـ big endian byte order غير مدعومة لـ 64-bit (يرجع `-EINVAL` في `setup_accessors`).

---

### Group 2: Bit Mask Helper

#### `gpio_mmio_line2mask`

```c
static unsigned long gpio_mmio_line2mask(struct gpio_chip *gc,
                                          unsigned int line)
```

يحوّل رقم الـ GPIO line (0-based) إلى الـ bitmask المقابلة في الـ hardware register.

- **Parameters:**
  - `gc` — الـ `gpio_chip` embedded في `gpio_generic_chip`
  - `line` — رقم الـ GPIO line (0 .. ngpio-1)
- **Return:** `unsigned long` — الـ bitmask
- **Key details:**
  - لو `chip->be_bits == true`، line 0 هو MSB: `BIT(bits - 1 - line)`
  - عادي: `BIT(line)`
  - كل الـ get/set functions تعتمد عليها — أي bug هنا يأثر على كل حاجة

**مثال:** على 32-bit BE register، line 0 → bit 31، line 1 → bit 30.

---

### Group 3: GPIO Get (Read Value)

هذه المجموعة تُعيَّن لـ `gc->get` و `gc->get_multiple`. الاختيار بينها يتم في `gpio_mmio_setup_io()` بناءً على configuration flags.

---

#### `gpio_mmio_get`

```c
static int gpio_mmio_get(struct gpio_chip *gc, unsigned int gpio)
```

الأبسط — يقرأ من `reg_dat` فقط. يُستخدَم عندما لا توجد `reg_set` أو عندما `reg_set` لا يعكس قيمة الـ output.

- **Parameters:** `gc` / `gpio` — رقم الـ line
- **Return:** 0 أو 1 (قيمة الـ pin)
- **Who calls it:** gpiolib core عند استدعاء `gpiod_get_value()`

---

#### `gpio_mmio_get_set`

```c
static int gpio_mmio_get_set(struct gpio_chip *gc, unsigned int gpio)
```

أذكى من السابق — يفرق بين outputs وinputs. لو الـ pin مضبوط كـ output (bit مضبوط في `chip->sdir`)، يقرأ من `reg_set` لأنه يعكس القيمة الفعلية على الـ output pin. لو input، يقرأ من `reg_dat`.

- **Key details:** يستخدم `chip->sdir` (shadow direction) — لا يقرأ hardware direction register

**Pseudocode:**
```
pinmask = line2mask(gpio)
if sdir & pinmask:        // output pin
    return read(reg_set) & pinmask
else:                     // input pin
    return read(reg_dat) & pinmask
```

---

#### `gpio_mmio_get_multiple`

```c
static int gpio_mmio_get_multiple(struct gpio_chip *gc,
                                   unsigned long *mask,
                                   unsigned long *bits)
```

يقرأ عدة pins دفعة واحدة بـ single register read — أكفأ من قراءة كل pin لوحده.

- **Parameters:**
  - `mask` — bitmask للـ pins المطلوب قراءتها
  - `bits` — [output] تُحدَّث بقيم الـ pins
- **Key details:**
  - `*bits &= ~*mask` أولاً لتصفير الـ bits المعنية قبل الكتابة
  - يعمل فقط مع native endian registers — لا يُعيَّن لـ `gc->get_multiple` لو `be_bits == true`

---

#### `gpio_mmio_get_multiple_be`

```c
static int gpio_mmio_get_multiple_be(struct gpio_chip *gc,
                                      unsigned long *mask,
                                      unsigned long *bits)
```

نفس الهدف لكن مع big endian bit order — أكثر تعقيداً لأن bit mapping معكوس.

**Pseudocode:**
```
readmask = 0
for each bit in mask:
    readmask |= line2mask(bit)     // mirror mask to BE bit positions

val = read(reg_dat) & readmask

for each set bit in val:           // mirror result back to LE bit positions
    *bits |= line2mask(bit)
```

الـ double mirror هنا ضروري: مرة لبناء الـ readmask بـ BE positions، ومرة لتحويل النتيجة لـ line-number indexed bits.

---

#### `gpio_mmio_get_set_multiple`

```c
static int gpio_mmio_get_set_multiple(struct gpio_chip *gc,
                                       unsigned long *mask,
                                       unsigned long *bits)
```

نسخة multiple من `gpio_mmio_get_set` — يقسم الـ pins بين outputs (يقرأ من `reg_set`) وinputs (يقرأ من `reg_dat`).

- **Key details:** لا يُعيَّن لـ `gc->get_multiple` لو `be_bits == true` (complexity issue موثَّق في comment)

---

### Group 4: GPIO Set (Write Value)

#### `gpio_mmio_set_none`

```c
static int gpio_mmio_set_none(struct gpio_chip *gc, unsigned int gpio, int val)
```

stub يرجع 0 بدون أي عمل. يُستخدَم عندما يكون الـ GPIO input-only (`GPIO_GENERIC_NO_OUTPUT`).

---

#### `gpio_mmio_set`

```c
static int gpio_mmio_set(struct gpio_chip *gc, unsigned int gpio, int val)
```

الـ set function الأساسية لـ controllers ذات register واحد للـ data. تحتفظ بـ shadow register `chip->sdata` لأن read-modify-write على الـ hardware يحتاج lock.

- **Parameters:** `gpio` — رقم الـ line / `val` — 0 أو non-zero
- **Locking:** `guard(raw_spinlock_irqsave)` — يحمي الـ shadow data وعملية الكتابة معاً
- **Key details:**
  - لا يمكن استخدام `raw_spinlock` من context يستطيع النوم
  - الـ shadow `sdata` ضروري لأن الـ register قد لا يكون readable في بعض الـ configs

**Pseudocode:**
```
lock_irqsave
mask = line2mask(gpio)
if val: sdata |= mask
else:   sdata &= ~mask
write_reg(reg_dat, sdata)
unlock_irqrestore
```

---

#### `gpio_mmio_set_with_clear`

```c
static int gpio_mmio_set_with_clear(struct gpio_chip *gc, unsigned int gpio,
                                     int val)
```

يستخدم set/clear register pair — الطريقة الأنظف لـ hardware ذات dedicated set وclear registers. **لا يحتاج lock** لأن كل register يكتب فيه بـ bitmask مباشرة بدون read-modify-write.

- **Key details:** كتابة bit=1 في `reg_set` يرفع الـ pin، كتابة bit=1 في `reg_clr` يخفضه. atomic من ناحية الـ hardware على أغلب الـ designs.

```c
if (val)
    write_reg(reg_set, mask);
else
    write_reg(reg_clr, mask);
```

---

#### `gpio_mmio_set_set`

```c
static int gpio_mmio_set_set(struct gpio_chip *gc, unsigned int gpio, int val)
```

لـ controllers التي لديها `reg_set` فقط بدون `reg_clr` — الـ reg_set يكتب القيمة الكاملة (write 1 to set, write 0 to clear). محتاج shadow + lock مثل `gpio_mmio_set`.

---

#### `gpio_mmio_multiple_get_masks`

```c
static void gpio_mmio_multiple_get_masks(struct gpio_chip *gc,
                                          unsigned long *mask,
                                          unsigned long *bits,
                                          unsigned long *set_mask,
                                          unsigned long *clear_mask)
```

helper مشترك يحسب `set_mask` (الـ pins المراد رفعها) و`clear_mask` (الـ pins المراد خفضها) من input `mask` و`bits`.

- **Parameters:**
  - `mask` — أي pins تُعدَّل
  - `bits` — القيم المطلوبة
  - `set_mask` [output] — pins يجب رفعها
  - `clear_mask` [output] — pins يجب خفضها
- **Key details:** يمر على كل bit في mask ويقسمها بناءً على قيمتها في `bits`. يستخدم `gpio_mmio_line2mask` لدعم BE bit order.

---

#### `gpio_mmio_set_multiple_single_reg`

```c
static void gpio_mmio_set_multiple_single_reg(struct gpio_chip *gc,
                                               unsigned long *mask,
                                               unsigned long *bits,
                                               void __iomem *reg)
```

helper مشترك لـ `gpio_mmio_set_multiple` و`gpio_mmio_set_multiple_set` — يكتب عدة pins في register واحد مع shadow + lock.

- **Locking:** `guard(raw_spinlock_irqsave)` — يحمي `chip->sdata` وعملية الكتابة

---

#### `gpio_mmio_set_multiple` / `gpio_mmio_set_multiple_set`

```c
static int gpio_mmio_set_multiple(struct gpio_chip *gc,
                                   unsigned long *mask,
                                   unsigned long *bits)
static int gpio_mmio_set_multiple_set(struct gpio_chip *gc,
                                       unsigned long *mask,
                                       unsigned long *bits)
```

الأولى تكتب في `reg_dat`، الثانية في `reg_set`. كلاهما يستدعي `gpio_mmio_set_multiple_single_reg`.

---

#### `gpio_mmio_set_multiple_with_clear`

```c
static int gpio_mmio_set_multiple_with_clear(struct gpio_chip *gc,
                                              unsigned long *mask,
                                              unsigned long *bits)
```

يكتب عدة pins باستخدام set/clear pair. **لا يحتاج lock** لأنه يكتب reg_set وreg_clr منفصلَين بدون read-modify-write.

- **Key details:** يكتب `reg_set` و`reg_clr` بـ single write كل منهما — فعّال جداً

---

### Group 5: Direction Control

هذه المجموعة تُعيَّن لـ `gc->direction_input`، `gc->direction_output`، و`gc->get_direction`. الاختيار يتم في `gpio_mmio_setup_direction()`.

---

#### `gpio_mmio_dir_return`

```c
static int gpio_mmio_dir_return(struct gpio_chip *gc, unsigned int gpio,
                                 bool dir_out)
```

helper مشترك يستدعي `pinctrl_gpio_direction_output()` أو `pinctrl_gpio_direction_input()` لو `chip->pinctrl == true`. لو لا يوجد pinctrl backend، يرجع 0 مباشرة.

- **Who calls it:** جميع direction functions في نهايتها

---

#### `gpio_mmio_dir_in`

```c
static int gpio_mmio_dir_in(struct gpio_chip *gc, unsigned int gpio)
```

يضبط الـ pin كـ input — يحدّث shadow direction `sdir` ويكتب hardware registers.

- **Locking:** `scoped_guard(raw_spinlock_irqsave)` — يحمي `sdir` والكتابة للـ registers

**Pseudocode:**
```
lock_irqsave {
    sdir &= ~line2mask(gpio)           // clear output bit
    if reg_dir_in:
        write(reg_dir_in, ~sdir)       // 1=input in reg_dir_in
    if reg_dir_out:
        write(reg_dir_out, sdir)       // 1=output in reg_dir_out
}
return pinctrl direction_input callback
```

- **Key details:** `reg_dir_in` يتوقع `1=input` لذا يُكتب `~sdir`. `reg_dir_out` يتوقع `1=output` لذا يُكتب `sdir` مباشرة.

---

#### `gpio_mmio_dir_out`

```c
static void gpio_mmio_dir_out(struct gpio_chip *gc, unsigned int gpio, int val)
```

helper داخلي يضبط الـ pin كـ output في الـ registers فقط. لا يستدعي pinctrl. يُستدعى من `dir_out_dir_first` و`dir_out_val_first`.

---

#### `gpio_mmio_dir_out_dir_first`

```c
static int gpio_mmio_dir_out_dir_first(struct gpio_chip *gc,
                                        unsigned int gpio, int val)
```

يضبط الـ direction أولاً ثم يكتب القيمة. يُستخدَم مع `GPIO_GENERIC_NO_SET_ON_INPUT` حيث الـ hardware يرفض كتابة القيمة على pin مضبوط كـ input.

```
gpio_mmio_dir_out(gc, gpio, val)    // direction first
gc->set(gc, gpio, val)              // then value
return pinctrl callback
```

---

#### `gpio_mmio_dir_out_val_first`

```c
static int gpio_mmio_dir_out_val_first(struct gpio_chip *gc,
                                        unsigned int gpio, int val)
```

يكتب القيمة أولاً ثم يضبط الـ direction. الأفضل في أغلب الحالات لتجنب glitch على الـ output pin — القيمة جاهزة قبل تحويل الـ pin لـ output.

```
gc->set(gc, gpio, val)              // value first (no glitch)
gpio_mmio_dir_out(gc, gpio, val)    // then direction
return pinctrl callback
```

---

#### `gpio_mmio_simple_dir_in` / `gpio_mmio_simple_dir_out`

```c
static int gpio_mmio_simple_dir_in(struct gpio_chip *gc, unsigned int gpio)
static int gpio_mmio_simple_dir_out(struct gpio_chip *gc, unsigned int gpio,
                                     int val)
```

للـ chips التي ليس لها direction registers (bidirectional بدون config). `simple_dir_out` يكتب القيمة أولاً ثم يستدعي pinctrl. `simple_dir_in` يستدعي pinctrl فقط.

---

#### `gpio_mmio_dir_in_err` / `gpio_mmio_dir_out_err`

```c
static int gpio_mmio_dir_in_err(struct gpio_chip *gc, unsigned int gpio)
static int gpio_mmio_dir_out_err(struct gpio_chip *gc, unsigned int gpio,
                                  int val)
```

stubs يرجعان `-EINVAL`. يُعيَّنان عندما يكون الـ GPIO output-only أو input-only على التوالي.

---

#### `gpio_mmio_get_dir`

```c
static int gpio_mmio_get_dir(struct gpio_chip *gc, unsigned int gpio)
```

يقرأ اتجاه الـ pin الحالي. يدعم ثلاث حالات:

1. **`dir_unreadable == true`**: يرجع من shadow `chip->sdir`
2. **`reg_dir_out` موجود**: يقرأ من hardware مباشرة
3. **`reg_dir_in` موجود**: يقرأ من hardware (logic معكوس)

- **Return:** `GPIO_LINE_DIRECTION_OUT` (0) أو `GPIO_LINE_DIRECTION_IN` (1) — تعريفات من `<linux/gpio/driver.h>`
- **Key details:** لا يحتاج lock لأن القراءة atomic على عكس الكتابة

---

### Group 6: Setup & Registration

هذه أهم مجموعة — تُهيئ الـ chip وتختار الـ function pointers المناسبة.

---

#### `gpio_mmio_setup_accessors`

```c
static int gpio_mmio_setup_accessors(struct device *dev,
                                      struct gpio_generic_chip *chip,
                                      bool byte_be)
```

يختار `chip->read_reg` و`chip->write_reg` بناءً على `chip->bits` و`byte_be` flag.

- **Parameters:**
  - `dev` — لـ error logging فقط
  - `chip` — الـ struct المراد تهيئته
  - `byte_be` — `true` لو `GPIO_GENERIC_BIG_ENDIAN_BYTE_ORDER` مضبوط
- **Return:** 0 أو `-EINVAL` لو الـ width غير مدعوم أو 64-bit BE
- **Key details:** يُستدعى مرة واحدة في `gpio_generic_chip_init`. بعد ذلك كل الكود يستدعي `chip->read_reg(addr)` مباشرة.

```
switch chip->bits:
  case 8:  read=read8,    write=write8
  case 16: read=read16be, write=write16be  (if byte_be)
           read=read16,   write=write16    (else)
  case 32: read=read32be, write=write32be  (if byte_be)
           read=read32,   write=write32    (else)
  case 64: if byte_be → EINVAL
           read=read64, write=write64
  default: EINVAL
```

---

#### `gpio_mmio_setup_io`

```c
static int gpio_mmio_setup_io(struct gpio_generic_chip *chip,
                               const struct gpio_generic_chip_config *cfg)
```

يضبط `chip->reg_dat`, `chip->reg_set`, `chip->reg_clr` ويختار `gc->set`, `gc->set_multiple`, `gc->get`, `gc->get_multiple` بناءً على الـ configuration.

- **Parameters:** `chip` + `cfg` — الـ config المُمرَّر من المستخدم
- **Return:** `-EINVAL` لو `cfg->dat == NULL`
- **Key details:** `reg_dat` إلزامي دائماً. باقي الـ registers اختيارية وتؤثر على اختيار الـ function pointers.

**منطق اختيار set callback:**

| cfg->set | cfg->clr | النتيجة |
|---|---|---|
| ✓ | ✓ | `set_with_clear` — pair registers |
| ✓ | ✗ | `set_set` — reg_set يكتب الكاملة |
| NO_OUTPUT flag | — | `set_none` |
| ✗ | ✗ | `set` — reg_dat فقط |

**منطق اختيار get callback:**

```
if !UNREADABLE_REG_SET && READ_OUTPUT_REG_SET:
    get = gpio_mmio_get_set         (reads output from reg_set)
    if !be_bits: get_multiple = gpio_mmio_get_set_multiple
else:
    get = gpio_mmio_get             (reads always from reg_dat)
    if be_bits: get_multiple = gpio_mmio_get_multiple_be
    else:       get_multiple = gpio_mmio_get_multiple
```

---

#### `gpio_mmio_setup_direction`

```c
static int gpio_mmio_setup_direction(struct gpio_generic_chip *chip,
                                      const struct gpio_generic_chip_config *cfg)
```

يضبط `gc->direction_input`، `gc->direction_output`، `gc->get_direction` بناءً على وجود `cfg->dirout`/`cfg->dirin` و flags.

- **Return:** دائماً 0 (لا يوجد error path حالياً)
- **Key details:**
  - لو `dirout` أو `dirin` موجودَين: يستخدم full direction functions مع hardware register writes
  - لو لا شيء: يستخدم simple variants
  - `GPIO_GENERIC_NO_SET_ON_INPUT` يحدد الترتيب في `dir_out` — dir_first vs val_first

---

#### `gpio_mmio_request`

```c
static int gpio_mmio_request(struct gpio_chip *gc, unsigned int gpio_pin)
```

callback يُستدعى من gpiolib عند طلب GPIO line بـ `gpiod_get()`. يتحقق من حدود الـ gpio number ولو `chip->pinctrl == true` يستدعي `gpiochip_generic_request()` لـ pinctrl muxing.

- **Return:** 0 أو `-EINVAL` أو error من pinctrl subsystem
- **Who calls it:** gpiolib core في `gpiod_request()`

---

#### `gpio_generic_chip_init` ★ (Entry Point)

```c
int gpio_generic_chip_init(struct gpio_generic_chip *chip,
                            const struct gpio_generic_chip_config *cfg)
```

الـ function الوحيدة الـ exported (`EXPORT_SYMBOL_GPL`). تُهيئ الـ `gpio_generic_chip` struct كاملاً.

- **Parameters:**
  - `chip` — الـ struct المراد تهيئته (allocated بالفعل من المستخدم)
  - `cfg` — الـ configuration: registers, flags, device, size
- **Return:** 0 أو negative error

**Pseudocode flow:**

```
// 1. Validate register size
if !is_power_of_2(cfg->sz) → EINVAL

chip->bits = cfg->sz * 8
if bits > BITS_PER_LONG → EINVAL

// 2. Basic init
raw_spin_lock_init(&chip->lock)
gc->parent = dev
gc->label = dev_name(dev)
gc->base = -1          // dynamic GPIO number allocation
gc->request = gpio_mmio_request
chip->be_bits = !!(flags & GPIO_GENERIC_BIG_ENDIAN)

// 3. GPIO count (from device tree "ngpios" property or default to bits)
gpiochip_get_ngpios(gc, dev) or gc->ngpio = chip->bits

// 4. Setup function pointers
gpio_mmio_setup_io(chip, cfg)         // get/set callbacks
gpio_mmio_setup_accessors(dev, chip, BE_flag)  // read_reg/write_reg
gpio_mmio_setup_direction(chip, cfg)  // direction callbacks

// 5. Pinctrl backend
if GPIO_GENERIC_PINCTRL_BACKEND:
    chip->pinctrl = true
    gc->free = gpiochip_generic_free

// 6. Shadow data init
chip->sdata = read_reg(reg_dat)
if set_set && !UNREADABLE_REG_SET:
    chip->sdata = read_reg(reg_set)   // prefer reg_set as source of truth

// 7. Shadow direction init
if GPIO_GENERIC_UNREADABLE_REG_DIR:
    chip->dir_unreadable = true
else if reg_dir_out || reg_dir_in:
    if reg_dir_out: sdir = read(reg_dir_out)
    elif reg_dir_in: sdir = ~read(reg_dir_in)
    // sync: if both exist, write reg_dir_in from reg_dir_out state
    if reg_dir_out && reg_dir_in:
        write(reg_dir_in, ~sdir)
```

- **Locking:** `raw_spin_lock_init` — لا يحتاج lock هنا لأنه init context
- **Side effects:** يكتب `reg_dir_in` لو الـ chip لها كلا الـ direction registers (sync)
- **Who calls it:** platform probe function أو أي driver يريد استخدام generic GPIO

---

### Group 7: Platform Driver

#### `gpio_mmio_map`

```c
static void __iomem *gpio_mmio_map(struct platform_device *pdev,
                                    const char *name,
                                    resource_size_t sane_sz)
```

يبحث عن IORESOURCE_MEM بالاسم ويعمل `devm_ioremap_resource()`. يتحقق أن الـ resource size مطابقة لـ `sane_sz` (size الـ `dat` register).

- **Return:** `void __iomem *` أو `NULL` لو غير موجود أو `IOMEM_ERR_PTR(-EINVAL)` لو size مختلف أو error من `devm_ioremap_resource`
- **Key details:** كل الـ registers يجب أن تكون نفس الـ size — هذا الـ sanity check يضمن ذلك

---

#### `gpio_mmio_pdev_probe`

```c
static int gpio_mmio_pdev_probe(struct platform_device *pdev)
```

الـ probe function للـ platform driver. يُهيئ GPIO chip من device tree أو platform data.

- **Return:** 0 أو negative error
- **Key details:**
  - يقرأ resources بالأسماء: `"dat"` (إلزامي), `"set"`, `"clr"`, `"dirout"`, `"dirin"` (كلها اختيارية)
  - يدعم `"no-output"` device property
  - يدعم `"label"` device property لتسمية الـ chip
  - `"gpio-mmio,base"` property للـ legacy static GPIO numbering (deprecated، board files فقط)
  - يستخدم `devm_gpiochip_add_data()` — الـ chip تُحذَف تلقائياً عند الـ unbind

**Device tree example:**
```
gpio0: gpio@10000000 {
    compatible = "opencores,gpio";
    reg = <0x10000000 0x1>,    /* dat */
          <0x10000001 0x1>,    /* set */
          <0x10000002 0x1>;    /* clr */
    reg-names = "dat", "set", "clr";
    #gpio-cells = <2>;
    gpio-controller;
    ngpios = <8>;
};
```

**Supported compatible strings:**

| Compatible | Hardware |
|---|---|
| `brcm,bcm6345-gpio` | Broadcom BCM6345 |
| `wd,mbl-gpio` | Western Digital My Book Live |
| `ni,169445-nand-gpio` | National Instruments |
| `intel,ixp4xx-expansion-bus-mmio-gpio` | Intel IXP4xx |
| `opencores,gpio` | OpenCores generic GPIO |

---

### ملاحظات عامة على الـ Locking Strategy

الـ driver يستخدم `raw_spinlock_t` (وليس `spinlock_t`) لأن:

1. GPIO operations قد تحدث من **interrupt context** أو **atomic context**
2. الـ `raw_spinlock` لا يتأثر بـ `PREEMPT_RT` — يبقى spinning حتى في RT kernels
3. `guard(raw_spinlock_irqsave)` من `<linux/cleanup.h>` يضمن unlock تلقائي عند الخروج من الـ scope

الـ shadow registers (`sdata`, `sdir`) ضرورية لأن:
- بعض الـ hardware registers write-only (لا يمكن قراءتها)
- الـ read-modify-write على الـ hardware غير atomic — يجب تخزين الحالة محلياً والكتابة بالكامل
- الـ lock يحمي الـ shadow register وعملية الكتابة كوحدة واحدة
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs — الـ Entries المهمة

الـ GPIO subsystem بيكشف معلومات في `/sys/kernel/debug/gpio` و `/sys/kernel/debug/pinctrl`.

```bash
# عرض حالة كل GPIO lines في النظام
cat /sys/kernel/debug/gpio

# مثال على الـ output:
# gpiochip0: GPIOs 0-31, parent: platform/10010000.gpio, basic-mmio-gpio:
#  gpio-0  (                    ) out lo
#  gpio-1  (spi-cs              ) out hi  [kernel]
#  gpio-5  (button              ) in  hi  [kernel]
```

```bash
# عرض كل الـ pinctrl devices ومعلوماتها
ls /sys/kernel/debug/pinctrl/

# عرض pin states لـ controller معين
cat /sys/kernel/debug/pinctrl/<pinctrl-name>/pins
cat /sys/kernel/debug/pinctrl/<pinctrl-name>/pingroups
cat /sys/kernel/debug/pinctrl/<pinctrl-name>/pinmux-pins
```

الـ `gpio_chip.dbg_show` callback ممكن الـ driver يـ override عشان يضيف معلومات extra زي قيمة الـ shadow registers.

---

#### 2. sysfs — الـ Entries المهمة

```bash
# اكتشاف كل GPIO chips في النظام
ls /sys/bus/platform/devices/ | grep gpio

# معلومات الـ gpiochip
ls /sys/class/gpio/
# gpiochip0  gpiochip32  ...

cat /sys/class/gpio/gpiochip0/base      # أول GPIO number
cat /sys/class/gpio/gpiochip0/ngpio     # عدد الـ lines
cat /sys/class/gpio/gpiochip0/label     # اسم الـ chip (من dev_name)

# export GPIO للـ userspace (مثلاً GPIO 5)
echo 5 > /sys/class/gpio/export
cat /sys/class/gpio/gpio5/direction     # in أو out
cat /sys/class/gpio/gpio5/value         # 0 أو 1

# تغيير الـ direction والـ value
echo out > /sys/class/gpio/gpio5/direction
echo 1   > /sys/class/gpio/gpio5/value
```

---

#### 3. ftrace — الـ Tracepoints والـ Events

الـ GPIO subsystem عنده tracepoints في `drivers/gpio/gpiolib.c`:

```bash
# تفعيل كل GPIO events
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable

# أو events محددة
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_direction/enable
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_value/enable

# قراءة الـ trace buffer
cat /sys/kernel/debug/tracing/trace

# مثال على الـ output:
# kworker/0:1-42    [000] ....  123.456789: gpio_direction: gpio=5 direction=out
# kworker/0:1-42    [000] ....  123.456790: gpio_value: gpio=5 get=0 value=1
```

```bash
# Function tracer لتتبع دوال الـ gpio-mmio نفسها
echo function > /sys/kernel/debug/tracing/current_tracer
echo 'gpio_mmio_*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

```bash
# تتبع gpio_generic_chip_init بالكامل مع الـ callgraph
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo gpio_generic_chip_init > /sys/kernel/debug/tracing/set_graph_function
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

---

#### 4. printk / Dynamic Debug

```bash
# تفعيل dynamic debug لـ gpio-mmio module
echo 'module gpio_generic +p' > /sys/kernel/debug/dynamic_debug/control
echo 'module gpio_mmio +p'    > /sys/kernel/debug/dynamic_debug/control

# تفعيل لملف معين
echo 'file drivers/gpio/gpio-mmio.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
# p = print, f = function name, l = line number, m = module, t = timestamp

# عرض ما تم تفعيله
cat /sys/kernel/debug/dynamic_debug/control | grep gpio

# مراقبة الـ kernel log في real-time
dmesg -w
# أو
journalctl -k -f
```

```bash
# رفع مستوى الـ loglevel مؤقتاً (0=emerg .. 8=debug)
echo 8 > /proc/sys/kernel/printk

# أو عبر dmesg
dmesg -n 8
```

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الهدف |
|---|---|
| `CONFIG_GPIOLIB` | تفعيل الـ GPIO library الأساسية |
| `CONFIG_GPIO_GENERIC` | تفعيل الـ generic MMIO GPIO driver |
| `CONFIG_GPIO_GENERIC_PLATFORM` | دعم الـ platform device probe |
| `CONFIG_DEBUG_GPIO` | تفعيل فحوصات runtime وـ debugfs entries |
| `CONFIG_DEBUG_FS` | لازم لـ `/sys/kernel/debug/gpio` |
| `CONFIG_DYNAMIC_DEBUG` | تفعيل dynamic debug messages |
| `CONFIG_PROVE_LOCKING` | كشف مشاكل الـ spinlock (lockdep) |
| `CONFIG_LOCK_STAT` | إحصائيات الـ lock contention |
| `CONFIG_GPIOLIB_IRQCHIP` | دعم interrupt من GPIO |
| `CONFIG_GPIO_SYSFS` | كشف GPIO في sysfs |
| `CONFIG_PINCTRL` | لو الـ chip بيستخدم pinctrl backend |
| `CONFIG_FTRACE` | لـ function tracing |
| `CONFIG_KASAN` | كشف memory corruption |

```bash
# فحص الـ config الحالي
zcat /proc/config.gz | grep -E 'GPIO|PINCTRL|DEBUG_FS|DYNAMIC_DEBUG'
# أو
grep -E 'CONFIG_GPIO|CONFIG_DEBUG_GPIO' /boot/config-$(uname -r)
```

---

#### 6. أدوات خاصة بالـ Subsystem

**الـ libgpiod — أداة userspace للـ GPIO:**

```bash
# تثبيت الأداة
apt install gpiod        # Debian/Ubuntu
dnf install libgpiod-utils  # Fedora

# عرض كل GPIO chips
gpiodetect

# عرض lines لـ chip معين
gpioinfo gpiochip0

# قراءة قيمة line
gpioget gpiochip0 5

# كتابة قيمة
gpioset gpiochip0 5=1

# مراقبة events (edge detection)
gpiomon gpiochip0 5
```

**الـ devlink (لا ينطبق مباشرة على gpio-mmio) — لكن platform device:**

```bash
# عرض platform device info
ls /sys/bus/platform/devices/10010000.gpio/
cat /sys/bus/platform/devices/10010000.gpio/driver_override
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الخطأ | المعنى | الحل |
|---|---|---|
| `unsupported data width %u bits` | الـ `sz` مش 1/2/4/8 bytes | تأكد من `gpio-mmio` resource size في DT أو board file |
| `64 bit big endian byte order unsupported` | تركيبة `64bit + BE` مش مدعومة | استخدم LE أو حوّل لـ 32bit |
| `gpio_mmio_pdev_probe: -EINVAL` | resource "dat" مش موجود | فحص DT أو platform_data، لازم يكون في "dat" resource |
| `devm_ioremap_resource failed` | الـ memory region مش متاح أو متداخل | فحص `/proc/iomem` للـ conflicts |
| `gpiochip_get_ngpios failed` | مش لاقي "ngpios" property أو قيمة غلط | أضف `ngpios` في DT أو الـ chip بيستخدم `bits` default |
| `-ENOMEM` في probe | فشل `devm_kzalloc` | مشكلة في الـ memory، فحص kernel OOM logs |
| `pinctrl_gpio_direction_output: -EINVAL` | الـ pin مش متعرف في pinctrl | فحص pin mapping في DT بين gpio وpinctrl |
| `gpio_mmio_request: -EINVAL` | رقم الـ gpio أكبر من `ngpio` | بق في الـ consumer code أو `ngpio` غلط |
| `GPIO_GENERIC_NO_OUTPUT set, cannot drive output` | محاولة set لـ input-only chip | فحص الـ `no-output` DT property أو الـ flags |
| `resource size mismatch` in `gpio_mmio_map` | resources مش نفس size | كل resources (set/clr/dirout) لازم نفس size زي "dat" |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في gpio_generic_chip_init — بعد فشل is_power_of_2 */
if (!is_power_of_2(cfg->sz)) {
    dev_err(dev, "sz=%lu is not power of 2\n", cfg->sz);
    WARN_ON(1);   /* <-- هنا: لو caller بعت sz غريب */
    return -EINVAL;
}

/* في gpio_mmio_setup_io — لو reg_dat == NULL */
if (!chip->reg_dat) {
    dev_err(dev, "reg_dat is NULL, cannot operate\n");
    dump_stack();  /* <-- هنا: لتتبع من أين جاء الـ NULL */
    return -EINVAL;
}

/* في gpio_mmio_get_set — لو sdir inconsistent */
WARN_ON(chip->reg_set == NULL && (chip->sdir & pinmask));

/* في gpio_mmio_set — لو write_reg pointer مش initialized */
WARN_ON(!chip->write_reg);

/* في gpio_generic_chip_init — بعد قراءة sdata الأولية */
dev_dbg(dev, "initial sdata=0x%lx sdir=0x%lx bits=%d\n",
        chip->sdata, chip->sdir, chip->bits);
```

---

### Hardware Level

#### 1. التحقق من أن الـ Hardware State مطابق لـ Kernel State

الـ driver عنده **shadow registers**: `sdata` (بيظهر القيمة الكتابية المحفوظة) و`sdir` (الـ direction المحفوظ).

```bash
# قارن الـ kernel state مع الـ hardware:

# 1. اقرأ الـ shadow من kernel (عبر debugfs)
cat /sys/kernel/debug/gpio
# GPIO 5: out hi  ← kernel يقول output=1

# 2. اقرأ نفس الـ register مباشرة من hardware
devmem2 0x10010000 b   # اقرأ reg_dat (8-bit مثلاً)
# Value at address 0x10010000: 0x20  ← bit 5 = 1 ✓

# لو القيمتين مختلفتين = مشكلة في الـ shadow register أو race condition
```

---

#### 2. Register Dump Techniques

```bash
# تثبيت الأدوات
apt install devmem2

# قراءة register 8-bit عند عنوان 0x10010000
devmem2 0x10010000 b

# قراءة 16-bit
devmem2 0x10010000 h

# قراءة 32-bit
devmem2 0x10010000 w

# كتابة قيمة (خطر! تأكد من الـ address)
devmem2 0x10010000 b 0xFF

# لو devmem2 مش متاح، استخدم /dev/mem
python3 -c "
import mmap, os, struct
fd = os.open('/dev/mem', os.O_RDONLY)
m = mmap.mmap(fd, 4096, offset=0x10010000)
val = struct.unpack('<I', m[0:4])[0]
print(hex(val))
m.close()
os.close(fd)
"

# عرض كل memory regions المسجلة
cat /proc/iomem | grep -i gpio
```

```bash
# مثال عملي لـ BCM6345 GPIO controller (compatible في gpio-mmio):
# base address افتراضي: 0x10000034 (dat), 0x10000038 (set), 0x1000003C (clr)

devmem2 0x10000034 w   # reg_dat  → قراءة القيم الحالية
devmem2 0x10000038 w   # reg_set  → آخر set operation
devmem2 0x1000003C w   # reg_clr  → آخر clear operation
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

```
GPIO MMIO Debug — ما تتوقعه على الـ wire:

1. عند gpio_mmio_set(gc, pin=5, val=1):
   ┌─ CPU write bus ─────────────────────────────┐
   │  Address: 0x10010000 (reg_dat)               │
   │  Data:    0b00100000 (bit 5 set)             │
   │  Strobe:  ____╔════╗____                      │
   └─────────────────────────────────────────────┘
   GPIO Pin 5: ____╔═══════════════ (يرتفع)

2. عند gpio_mmio_set_with_clear(gc, pin=5, val=0):
   Write إلى reg_clr لا reg_dat
   GPIO Pin 5: ══════════╗_________ (ينزل)

3. للـ big-endian (be_bits=true):
   Pin 0 → bit (bits-1) في الـ register
   Pin 31 → bit 0
   يعني الـ bit pattern على الـ bus معكوس!
```

**نصائح للـ Logic Analyzer:**
- اربط probe على الـ address bus + data bus + write strobe
- ابحث عن write pattern متكرر لنفس الـ address = الـ driver شغال
- لو شايف reads بدون writes لـ output pin = مشكلة في shadow register
- لو الـ GPIO pin مش بيتحرك رغم الـ write = مشكلة في الـ clock/power للـ peripheral

---

#### 4. Common Hardware Issues وـ Kernel Log Patterns

| مشكلة HW | Pattern في الـ Kernel Log | السبب المحتمل |
|---|---|---|
| الـ GPIO مش بيتحرك رغم set | لا يوجد error — driver يقول ok | Power/clock للـ GPIO block معطل، فحص clock driver |
| قيمة مقروءة دائماً 0 أو FF | لا يوجد error | الـ MMIO address غلط أو الـ peripheral مش powered |
| كل الـ GPIOs output رغم direction=input | لا يوجد error | الـ reg_dir_in/out address غلط في DT |
| glitch على الـ GPIO عند direction change | لا يوجد error | `dir_first` vs `val_first` — استخدم `GPIO_GENERIC_NO_SET_ON_INPUT` flag |
| crash عند access | `BUG: unable to handle kernel paging request at 0xXXXX` | الـ ioremap address غلط، فحص DT memory region |
| IRQ storm بعد GPIO init | `irq X: nobody cared` | الـ GPIO edge trigger مش محدد صح |
| الـ direction register inconsistent | لا يوجد error | الـ hardware عنده `dir_in` و`dir_out` منفصلين ومش متزامنين — الـ driver بيعملهم sync في `gpio_generic_chip_init` |

---

#### 5. Device Tree Debugging — تحقق من الـ DT

```bash
# عرض الـ DT node للـ GPIO controller
cat /proc/device-tree/gpio@10010000/compatible
# brcm,bcm6345-gpio أو أي compatible مدعوم

# عرض كل properties
ls /proc/device-tree/gpio@10010000/
# reg  compatible  #gpio-cells  gpio-controller  ngpios  no-output  ...

# قراءة الـ reg property (base address + size)
hexdump -C /proc/device-tree/gpio@10010000/reg
# 00000000  00 00 00 00 10 01 00 00  00 00 00 00 00 00 00 04  ← base=0x10010000, size=4

# التحقق من ngpios
cat /proc/device-tree/gpio@10010000/ngpios
# أو بالـ hexdump لو مش printable
hexdump -C /proc/device-tree/gpio@10010000/ngpios

# التحقق من الـ no-output property
ls /proc/device-tree/gpio@10010000/ | grep no-output

# استخدام dtc لعرض الـ DT كاملاً
dtc -I fs /proc/device-tree 2>/dev/null | grep -A 20 "gpio@10010000"
```

**مثال DT صحيح لـ gpio-mmio:**

```dts
gpio0: gpio@10010000 {
    compatible = "brcm,bcm6345-gpio";
    reg-names = "dat", "set", "clr", "dirout";
    reg = <0x10010000 4>, <0x10010004 4>,
          <0x10010008 4>, <0x1001000C 4>;
    #gpio-cells = <2>;
    gpio-controller;
    ngpios = <32>;
    /* big-endian لو الـ hardware BE */
    /* no-output; */
};
```

**نقاط التحقق:**
- `reg` sizes لازم كلها متساوية (لازم `sz` واحد للكل في `gpio_mmio_map`)
- `#gpio-cells = <2>` إجباري
- `gpio-controller` إجباري
- لو عندك `dirout` و`dirin` معاً، الـ driver هيعملهم sync تلقائياً

---

### Practical Commands

#### جاهزة للـ Copy-Paste

```bash
# === فحص أولي شامل ===
echo "=== GPIO Chips ===" && gpiodetect 2>/dev/null || ls /sys/class/gpio/
echo "=== GPIO Debug ===" && cat /sys/kernel/debug/gpio 2>/dev/null
echo "=== iomem ===" && grep -i gpio /proc/iomem
echo "=== DT Compat ===" && grep -r "basic-mmio-gpio\|bcm6345-gpio\|wd,mbl-gpio" /proc/device-tree/ 2>/dev/null | cat
```

```bash
# === تفعيل كامل الـ debug messages لـ gpio-mmio ===
echo 8 > /proc/sys/kernel/printk
echo 'file drivers/gpio/gpio-mmio.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 'module gpio_generic +p' > /sys/kernel/debug/dynamic_debug/control
dmesg -w &
# الآن أعد تحميل الـ module أو اعمل GPIO operation
```

```bash
# === ftrace: تتبع كل دوال gpio_mmio ===
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo function > current_tracer
echo 'gpio_mmio_*' > set_ftrace_filter
echo '' > trace
echo 1 > tracing_on
# اعمل العملية اللي تريد تتتبعها
gpioset gpiochip0 5=1
echo 0 > tracing_on
cat trace | head -50
```

```bash
# === فحص shadow register عبر /sys/kernel/debug/gpio ===
watch -n 0.5 'cat /sys/kernel/debug/gpio | grep -A 35 "gpiochip0"'
```

```bash
# === قراءة MMIO registers مباشرة (استبدل BASE_ADDR) ===
BASE=0x10010000
SZ=4   # register width بالـ bytes

echo "reg_dat =" $(devmem2 $((BASE + 0x00)) w 2>/dev/null | grep "Value" | awk '{print $NF}')
echo "reg_set =" $(devmem2 $((BASE + 0x04)) w 2>/dev/null | grep "Value" | awk '{print $NF}')
echo "reg_clr =" $(devmem2 $((BASE + 0x08)) w 2>/dev/null | grep "Value" | awk '{print $NF}')
echo "reg_dir =" $(devmem2 $((BASE + 0x0C)) w 2>/dev/null | grep "Value" | awk '{print $NF}')
```

```bash
# === مقارنة kernel state مع hardware ===
# kernel يقول GPIO 5 = hi output
cat /sys/kernel/debug/gpio | grep "gpio-5"
# hardware يقول كام؟
devmem2 0x10010000 w   # اقرأ reg_dat وتحقق من bit 5
```

```bash
# === اختبار GPIO من userspace بدون driver (صرف للـ testing) ===
# export GPIO 5
echo 5 > /sys/class/gpio/export
echo out > /sys/class/gpio/gpio5/direction
echo 1 > /sys/class/gpio/gpio5/value
cat /sys/class/gpio/gpio5/value   # لازم يرجع 1
echo 0 > /sys/class/gpio/gpio5/value
echo 5 > /sys/class/gpio/unexport
```

```bash
# === lockdep: كشف spinlock issues ===
# في الـ kernel config: CONFIG_PROVE_LOCKING=y CONFIG_LOCK_STAT=y
# ثم شغّل العملية وابحث في dmesg
dmesg | grep -E "WARNING|BUG|lockdep|deadlock|GPIO"
```

```bash
# === فحص الـ DT بشكل كامل ===
dtc -I fs /proc/device-tree 2>/dev/null | grep -B2 -A30 "gpio-controller"
```

#### تفسير الـ Output

```
# مثال: cat /sys/kernel/debug/gpio
gpiochip0: GPIOs 0-31, parent: platform/10010000.gpio, basic-mmio-gpio:
 gpio-0  (                    ) out lo    ← output, قيمتها 0
 gpio-1  (spi0-cs0            ) out hi    ← output، مستخدم من SPI driver
 gpio-5  (user-button         ) in  hi    ← input، قيمتها 1 (مضغوط؟)
 gpio-10 (                    ) in  lo    ← input، حر وقيمتها 0

# لو gpio مش ظاهر في القائمة = لسه ما اتعملوش request
# لو الـ direction غلط = مشكلة في reg_dirout/reg_dirin أو الـ flags
# لو القيمة ثابتة دايماً = مشكلة في الـ hardware أو الـ shadow register
```

```
# مثال: ftrace output
gpio_mmio_set_with_clear() {      ← بيستخدم set/clr registers
  gpio_mmio_line2mask();          ← يحسب الـ bit mask
}
gpio_mmio_get_dir() {             ← بيقرأ direction
  gpio_mmio_line2mask();
}
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: FPGA GPIO على Industrial Gateway بـ AM62x — بيانات قراءة مقلوبة

#### العنوان
GPIO من FPGA بـ big-endian bit order على board صناعي — القراءة معكوسة تماماً

#### السياق
شركة بتصنع **industrial gateway** بـ **TI AM62x** SoC. الـ FPGA المتصل بيتحكم في 32 relay output. الـ driver بيستخدم `gpio-mmio` عبر device tree لأن الـ FPGA register بسيط: 32-bit MMIO register واحد بدون set/clear registers.

#### المشكلة
الـ engineer بيلاحظ إن relay رقم 0 بيتشغّل لما يكتب على GPIO line رقم 31، والعكس صحيح. كل الـ GPIO lines مقلوبة. الـ `gpioset` بتشتغل لكن على الـ line الغلط:

```bash
# المفروض يشغّل relay 0، بس بيشغّل relay 31
gpioset gpiochip0 0=1
```

#### التحليل
الـ FPGA وثّقه vendor بـ "bit 31 = channel 0"، يعني **big-endian bit order**. لكن الـ device tree مفيهوش `GPIO_GENERIC_BIG_ENDIAN` flag:

```c
/* gpio_mmio_line2mask() — السطر 127-134 في gpio-mmio.c */
static unsigned long gpio_mmio_line2mask(struct gpio_chip *gc, unsigned int line)
{
    struct gpio_generic_chip *chip = to_gpio_generic_chip(gc);

    if (chip->be_bits)
        return BIT(chip->bits - 1 - line); /* big-endian: line 0 → bit 31 */

    return BIT(line); /* little-endian: line 0 → bit 0 */
}
```

لأن `chip->be_bits` بـ `false`، كل استدعاء لـ `gpio_mmio_set()` بيكتب `BIT(line)` مباشرة. الـ FPGA بيفسّر bit 0 على إنه channel 31.

الـ flag بيتحدد في `gpio_generic_chip_init()` السطر 647:
```c
chip->be_bits = !!(flags & GPIO_GENERIC_BIG_ENDIAN);
```

والـ flag بيجي من `gpio_mmio_pdev_probe()` لكنه مش بيتقرأ من DT تلقائياً — لازم يتبعت كـ `GPIO_GENERIC_BIG_ENDIAN` في الـ `flags` field.

#### الحل
**1. تعديل الـ device tree:**

```dts
/* قبل التعديل */
fpga-gpio@40000000 {
    compatible = "opencores,gpio";
    reg = <0x40000000 0x4>,   /* dat */
          <0x40000004 0x4>;   /* dirout */
    reg-names = "dat", "dirout";
    gpio-controller;
    #gpio-cells = <2>;
    ngpios = <32>;
};

/* بعد التعديل — إضافة big-endian property */
fpga-gpio@40000000 {
    compatible = "opencores,gpio";
    reg = <0x40000000 0x4>,
          <0x40000004 0x4>;
    reg-names = "dat", "dirout";
    gpio-controller;
    #gpio-cells = <2>;
    ngpios = <32>;
    big-endian;   /* يخلّي device_is_big_endian() ترجع true → GPIO_GENERIC_BIG_ENDIAN_BYTE_ORDER */
};
```

لكن لاحظ: `GPIO_GENERIC_BIG_ENDIAN_BYTE_ORDER` ≠ `GPIO_GENERIC_BIG_ENDIAN`. الـ byte order بيغيّر الـ read/write functions فقط (السطر 657-658)، بينما `be_bits` بتعتمد على `GPIO_GENERIC_BIG_ENDIAN` (السطر 647). الحل الصح هو custom platform driver بيبعت الـ flag الصح:

```c
/* في custom board driver */
config.flags = GPIO_GENERIC_BIG_ENDIAN;
gpio_generic_chip_init(gen_gc, &config);
```

**2. التحقق بعد الإصلاح:**
```bash
# قراءة direction register لو readable
devmem2 0x40000004 w

# اختبار relay 0 بشكل مباشر
gpioset gpiochip0 0=1
# المفروض تشوف bit 31 بيتحوّل في register
devmem2 0x40000000 w
# النتيجة: 0x80000000
```

#### الدرس المستفاد
الفرق بين `GPIO_GENERIC_BIG_ENDIAN` (bit mirroring) و`GPIO_GENERIC_BIG_ENDIAN_BYTE_ORDER` (byte order of register access) دقيق جداً. لازم تفهم إن الـ FPGA vendor لما بيقول "big endian" ممكن يقصد bit numbering مش byte order. دايماً اقرأ الـ register value بـ `devmem2` وقارن يدوياً.

---

### السيناريو الثاني: STM32MP1 Custom Board — GPIO Direction Register يرجع -EINVAL وقت الـ Probe

#### العنوان
الـ probe بيفشل بـ `-EINVAL` على STM32MP1 بسبب register size mismatch في الـ device tree

#### السياق
**custom industrial sensor board** بـ **STM32MP157** — بيستخدم FPGA صغير عبر AHB bus كـ GPIO expander. الـ engineer بيكتب DT node بناءً على example من vendor وبيشغّل الـ kernel، بس الـ GPIO chip مش ظاهر في `gpioinfo`.

#### المشكلة
```bash
dmesg | grep gpio
# [    2.134] basic-mmio-gpio: probe failed: -22
```

الـ `-EINVAL` = `-22` بيجي بس مش واضح من فين.

#### التحليل
الـ probe function `gpio_mmio_pdev_probe()` السطر 732 بيعمل الآتي:

```c
/* السطر 749-753: الـ dat register بيحدد الـ sz */
r = platform_get_resource_byname(pdev, IORESOURCE_MEM, "dat");
if (!r)
    return -EINVAL;

sz = resource_size(r);  /* حجم الـ dat resource */
```

ثم بيمرر `sz` لكل `gpio_mmio_map()` استدعاء:

```c
/* السطر 705-720: gpio_mmio_map */
static void __iomem *gpio_mmio_map(struct platform_device *pdev,
                                   const char *name, resource_size_t sane_sz)
{
    r = platform_get_resource_byname(pdev, IORESOURCE_MEM, name);
    if (!r)
        return NULL;   /* مفيش resource = OK، بيرجع NULL */

    sz = resource_size(r);
    if (sz != sane_sz)
        return IOMEM_ERR_PTR(-EINVAL);  /* الحجم مختلف = ERROR */

    return devm_ioremap_resource(&pdev->dev, r);
}
```

الـ engineer عمل `dat` بحجم 4 bytes (32-bit) لكن `dirout` بحجم 2 bytes (16-bit) عن طريق الغلط في الـ DT:

```dts
/* الـ DT الغلط */
reg = <0x60000000 0x4>,   /* dat: 4 bytes */
      <0x60000010 0x2>;   /* dirout: 2 bytes — غلط! */
reg-names = "dat", "dirout";
```

#### الحل

```dts
/* الـ DT الصح */
fpga-expander@60000000 {
    compatible = "basic-mmio-gpio";
    reg = <0x60000000 0x4>,   /* dat: 4 bytes */
          <0x60000010 0x4>;   /* dirout: 4 bytes — نفس الحجم */
    reg-names = "dat", "dirout";
    gpio-controller;
    #gpio-cells = <2>;
    ngpios = <32>;
};
```

**أو لو الـ hardware فعلاً 16-bit:**

```dts
reg = <0x60000000 0x2>,   /* dat: 2 bytes */
      <0x60000010 0x2>;   /* dirout: 2 bytes */
```

**التحقق:**
```bash
# بعد إصلاح DT وreboot
gpioinfo
# المفروض تظهر gpiochip جديدة بـ 32 lines

# أو تتبع الـ probe
echo 'file gpio-mmio.c +p' > /sys/kernel/debug/dynamic_debug/control
dmesg | tail -20
```

#### الدرس المستفاد
الـ `gpio_mmio_map()` بتعمل size sanity check صارم — **كل** الـ MMIO resources لازم تكون بنفس حجم الـ `dat` resource. مش كفاية إن الـ hardware يشتغل بـ 16-bit access، لازم الـ DT يعكس الحجم الصح في كل الـ `reg` entries.

---

### السيناريو الثالث: Allwinner H616 Android TV Box — Race Condition في GPIO Set من SPI ISR

#### العنوان
بيانات SPI بتتلخبط بسبب GPIO set من داخل interrupt handler على Allwinner H616

#### السياق
**Android TV box** بـ **Allwinner H616** — الـ custom board عنده FPGA بيعمل عليه SPI communication، وبيستخدم GPIO من `gpio-mmio` driver (FPGA GPIO expander) كـ chip-select بدال الـ SPI native CS. الـ SPI interrupt handler بيعمل `gpio_set_value()` لرفع الـ CS بعد اكتمال الـ transfer.

#### المشكلة
بشكل عشوائي، بيانات SPI بتتلخبط في اللحظات الأولى من الـ system init. الـ `lockdep` بيطلع warning:

```
BUG: spinlock wrong CPU on CPU#2, swapper/0
  lock: chip->lock
  ...
  _raw_spin_lock called from gpio_mmio_set
```

#### التحليل
الـ `gpio_mmio_set()` السطر 229 بتستخدم `raw_spinlock_irqsave`:

```c
static int gpio_mmio_set(struct gpio_chip *gc, unsigned int gpio, int val)
{
    struct gpio_generic_chip *chip = to_gpio_generic_chip(gc);
    unsigned long mask = gpio_mmio_line2mask(gc, gpio);

    guard(raw_spinlock_irqsave)(&chip->lock);  /* السطر 234 */

    if (val)
        chip->sdata |= mask;
    else
        chip->sdata &= ~mask;

    chip->write_reg(chip->reg_dat, chip->sdata);  /* RMW على sdata */

    return 0;
}
```

الـ `raw_spinlock_irqsave` صح للـ ISR. لكن المشكلة مختلفة: الـ FPGA عنده `reg_set` و`reg_clr` لكن الـ DT مش عارّف غير `dat`. يعني بيستخدم `gpio_mmio_set()` بدال `gpio_mmio_set_with_clear()`.

في `gpio_mmio_set()` في الـ RMW pattern:
1. قراءة `chip->sdata`
2. تعديله
3. كتابته لـ `reg_dat`

لو انقطع interrupt بين خطوتين لـ CPU تانية تحاول تعمل نفس الشيء على نفس الـ chip، الـ lock بيحل المشكلة الواحدة. لكن المشكلة إن الـ FPGA نفسه عنده **set/clear registers** ومش بيُستخدموا — يعني تغييرات hardware غير متزامنة.

**الحل الحقيقي** هو إعلام الـ driver عن الـ set/clear registers:

```c
/* السطر 246-258: gpio_mmio_set_with_clear — no RMW needed */
static int gpio_mmio_set_with_clear(struct gpio_chip *gc, unsigned int gpio,
                                    int val)
{
    struct gpio_generic_chip *chip = to_gpio_generic_chip(gc);
    unsigned long mask = gpio_mmio_line2mask(gc, gpio);

    if (val)
        chip->write_reg(chip->reg_set, mask);   /* كتابة atomic */
    else
        chip->write_reg(chip->reg_clr, mask);   /* كتابة atomic */

    return 0;
}
```

لا lock، لا RMW — كل عملية كتابة atomic.

#### الحل

```dts
/* قبل — يستخدم dat فقط */
fpga-gpio@01c20800 {
    compatible = "basic-mmio-gpio";
    reg = <0x01c20800 0x4>;
    reg-names = "dat";
    ...
};

/* بعد — يستخدم set/clr للـ atomic operations */
fpga-gpio@01c20800 {
    compatible = "basic-mmio-gpio";
    reg = <0x01c20800 0x4>,    /* dat */
          <0x01c20804 0x4>,    /* set */
          <0x01c20808 0x4>;    /* clr */
    reg-names = "dat", "set", "clr";
    gpio-controller;
    #gpio-cells = <2>;
    ngpios = <32>;
};
```

بعد هذا `gpio_mmio_setup_io()` السطر 538 بيختار `gpio_mmio_set_with_clear` تلقائياً:

```c
if (cfg->set && cfg->clr) {
    chip->reg_set = cfg->set;
    chip->reg_clr = cfg->clr;
    gc->set = gpio_mmio_set_with_clear;  /* atomic، no lock needed */
    ...
}
```

#### الدرس المستفاد
لو الـ FPGA/hardware عنده set/clear registers، **لازم تستخدمها** حتى لو الـ `dat` register يكفي نظرياً. الـ atomic set/clear أسرع وأأمن في الـ ISR context ومفيش حاجة لـ spinlock.

---

### السيناريو الرابع: i.MX8 Automotive ECU — `get_direction` بيرجع نتيجة غلط بعد Resume من Suspend

#### العنوان
الـ GPIO direction بيتضيع بعد system suspend/resume على i.MX8 ECU

#### السياق
**automotive ECU** بـ **NXP i.MX8M Plus** للتحكم في actuators. الـ ECU بيدخل في **suspend** لما يكون السيارة واقفة للتوفير في الطاقة. الـ GPIO expander هو FPGA بسيط بـ direction register قابل للقراءة. بعد الـ resume، بعض الـ output GPIOs بتتعامل معاها الـ kernel كـ inputs.

#### المشكلة
```bash
# قبل suspend
gpioinfo gpiochip3
# line 5: output

# بعد resume
gpioinfo gpiochip3
# line 5: input  ← غلط!
```

الـ FPGA نفسه hardware reset بيحصله عند resume وبيرجع كل الـ direction registers لقيمة default (كلها inputs).

#### التحليل
`gpio_generic_chip_init()` السطر 683-697 بيقرأ الـ direction register مرة واحدة وقت الـ init:

```c
if ((chip->reg_dir_out || chip->reg_dir_in) &&
    !(flags & GPIO_GENERIC_UNREADABLE_REG_DIR)) {
    if (chip->reg_dir_out)
        chip->sdir = chip->read_reg(chip->reg_dir_out);  /* يُقرأ مرة */
    else if (chip->reg_dir_in)
        chip->sdir = ~chip->read_reg(chip->reg_dir_in);
    ...
}
```

الـ `chip->sdir` بيتحدّث صح في الـ `gpio_mmio_dir_out()` السطر 436:

```c
static void gpio_mmio_dir_out(struct gpio_chip *gc, unsigned int gpio, int val)
{
    struct gpio_generic_chip *chip = to_gpio_generic_chip(gc);

    guard(raw_spinlock_irqsave)(&chip->lock);

    chip->sdir |= gpio_mmio_line2mask(gc, gpio);  /* تحديث shadow */

    if (chip->reg_dir_in)
        chip->write_reg(chip->reg_dir_in, ~chip->sdir);
    if (chip->reg_dir_out)
        chip->write_reg(chip->reg_dir_out, chip->sdir);  /* كتابة للـ hardware */
}
```

بعد suspend/resume، الـ `chip->sdir` صح في الـ memory لكن الـ hardware register اتمسح. الـ `gpio_mmio_get_dir()` السطر 406 في الحالة الطبيعية بيقرأ من الـ hardware:

```c
if (chip->reg_dir_out) {
    if (chip->read_reg(chip->reg_dir_out) & gpio_mmio_line2mask(gc, gpio))
        return GPIO_LINE_DIRECTION_OUT;
    return GPIO_LINE_DIRECTION_IN;  /* الـ hardware اتمسح → بيرجع INPUT */
}
```

#### الحل
**1. إضافة suspend/resume callbacks في custom platform driver:**

```c
static int my_fpga_gpio_suspend(struct device *dev)
{
    /* لا شيء — الـ sdir محفوظ في memory */
    return 0;
}

static int my_fpga_gpio_resume(struct device *dev)
{
    struct gpio_generic_chip *chip = dev_get_drvdata(dev);

    /* إعادة كتابة direction register من الـ shadow */
    guard(raw_spinlock_irqsave)(&chip->lock);

    if (chip->reg_dir_out)
        chip->write_reg(chip->reg_dir_out, chip->sdir);
    if (chip->reg_dir_in)
        chip->write_reg(chip->reg_dir_in, ~chip->sdir);

    /* إعادة كتابة data register */
    chip->write_reg(chip->reg_dat, chip->sdata);

    return 0;
}

static DEFINE_SIMPLE_DEV_PM_OPS(my_fpga_gpio_pm_ops,
                                my_fpga_gpio_suspend,
                                my_fpga_gpio_resume);
```

**2. أو استخدام `GPIO_GENERIC_UNREADABLE_REG_DIR` flag** لو الـ hardware دايماً بيتعمله reset وانت مش قادر تعمل proper resume:

```c
/* هذا يخلّي gpio_mmio_get_dir() يعتمد على chip->sdir دايماً */
config.flags |= GPIO_GENERIC_UNREADABLE_REG_DIR;
```

```c
/* السطر 410-415 من gpio-mmio.c */
if (chip->dir_unreadable) {
    if (chip->sdir & gpio_mmio_line2mask(gc, gpio))
        return GPIO_LINE_DIRECTION_OUT;  /* يعتمد على shadow فقط */
    return GPIO_LINE_DIRECTION_IN;
}
```

#### الدرس المستفاد
الـ `gpio-mmio` driver مفيهوش suspend/resume built-in لأنه generic. في أي FPGA/hardware بيتعمله reset عند power cycle، لازم تكتب custom suspend/resume تعيد كتابة `chip->sdata` و`chip->sdir` للـ hardware من الـ shadow registers.

---

### السيناريو الخامس: RK3562 IoT Board — FPGA GPIO بـ 64-bit Register يفشل الـ Probe على 32-bit Kernel

#### العنوان
FPGA بـ 64-bit wide GPIO register مش شغّال على RK3562 kernel مبني لـ 32-bit

#### السياق
شركة بتصمم **IoT sensor hub** بـ **Rockchip RK3562**. الـ firmware team بنوا kernel بـ `CONFIG_ARM` (32-bit) مش `CONFIG_ARM64`. الـ FPGA المستخدم عنده GPIO data register بعرض 64-bit ليتحكم في 64 channel دفعة واحدة.

#### المشكلة
```bash
dmesg | grep gpio
# [    1.892] basic-mmio-gpio fpga-gpio: unsupported data width 64 bits
# [    1.893] basic-mmio-gpio: probe of fpga-gpio failed with error -22
```

#### التحليل
في `gpio_mmio_setup_accessors()` السطر 460:

```c
static int gpio_mmio_setup_accessors(struct device *dev,
                                     struct gpio_generic_chip *chip,
                                     bool byte_be)
{
    switch (chip->bits) {
    case 8:  ...
    case 16: ...
    case 32: ...
#if BITS_PER_LONG >= 64
    case 64:
        if (byte_be) {
            dev_err(dev, "64 bit big endian byte order unsupported\n");
            return -EINVAL;
        } else {
            chip->read_reg  = gpio_mmio_read64;
            chip->write_reg = gpio_mmio_write64;
        }
        break;
#endif /* BITS_PER_LONG >= 64 */
    default:
        dev_err(dev, "unsupported data width %u bits\n", chip->bits);
        return -EINVAL;  /* ← هنا بيوقع على 32-bit kernel */
    }
    ...
}
```

على الـ 32-bit kernel، `BITS_PER_LONG = 32`، فالـ `case 64` بيتحذف بالكامل من الـ compilation. الـ `chip->bits = 64` بيوقع في `default` case ويرجع `-EINVAL`.

ثم في `gpio_generic_chip_init()` السطر 638-640:

```c
chip->bits = cfg->sz * 8;   /* sz=8 bytes → bits=64 */
if (chip->bits > BITS_PER_LONG)  /* 64 > 32 → return -EINVAL */
    return -EINVAL;
```

في الواقع، الـ code بيرجع `-EINVAL` من السطر 639-640 **قبل** ما يوصل للـ `setup_accessors`. الأمان مضمون.

#### الحل
**الخيار 1: بناء kernel بـ 64-bit (الصح):**
```bash
# في Kconfig أو defconfig
CONFIG_ARM64=y
# مش
# CONFIG_ARM is not set
```

**الخيار 2: تقسيم الـ 64-bit FPGA register لـ register-pairs في 32-bit:**

لو مش ممكن تغيير الـ kernel architecture، تقسّم الـ FPGA logic لـ جزئين 32-bit:

```dts
/* بدال register واحد 64-bit */
fpga-gpio-lo@60000000 {
    compatible = "basic-mmio-gpio";
    reg = <0x60000000 0x4>;   /* 32 channels الأول */
    reg-names = "dat";
    gpio-controller;
    #gpio-cells = <2>;
    ngpios = <32>;
};

fpga-gpio-hi@60000004 {
    compatible = "basic-mmio-gpio";
    reg = <0x60000004 0x4>;   /* 32 channels التاني */
    reg-names = "dat";
    gpio-controller;
    #gpio-cells = <2>;
    ngpios = <32>;
};
```

**الخيار 3: استخدام `ngpios` لتقليل الـ lines المستخدمة:**
```dts
/* لو بس محتاج 32 channel من أصل 64 */
fpga-gpio@60000000 {
    compatible = "basic-mmio-gpio";
    reg = <0x60000000 0x4>;   /* 4 bytes = 32 bits فقط */
    reg-names = "dat";
    ngpios = <32>;
    ...
};
```

**التحقق على target:**
```bash
uname -m
# armv7l  → 32-bit kernel

# بعد التعديل
gpioinfo
# gpiochip4: "fpga-gpio-lo", 32 lines
# gpiochip5: "fpga-gpio-hi", 32 lines
```

#### الدرس المستفاد
الـ `gpio-mmio` driver واضح: **64-bit registers تتطلب 64-bit kernel**. الـ check في `gpio_generic_chip_init()` السطر 639 هو early guard. لو بتصمم FPGA GPIO جديد لـ embedded system، خلّي عرض الـ register يتناسب مع الـ target kernel architecture من البداية.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات المتعلقة بـ `gpio-mmio.c` وبيئتها:

| المقال | الوصف |
|--------|-------|
| [PATCH v4: gpio: Add driver for basic memory-mapped GPIO controllers](https://lwn.net/Articles/402868/) | الـ patch الأصلي اللي أضاف الـ driver لـ generic MMIO GPIO — نقطة البداية لفهم التصميم |
| [GPIO in the kernel: an introduction](https://lwn.net/Articles/532714/) | مقدمة شاملة لـ GPIO subsystem في الـ kernel — ضروري قبل ما تتعمق في الكود |
| [GPIO in the kernel: future directions](https://lwn.net/Articles/533632/) | الجزء التاني من السلسلة — بيتكلم عن الـ descriptor-based API اللي حلت محل الـ legacy integer API |
| [The platform device API](https://lwn.net/Articles/448499/) | بيشرح الـ `platform_device` و`platform_driver` اللي gpio-mmio بيبني عليهم |
| [Semantics of MMIO mapping attributes across architectures](https://lwn.net/Articles/698014/) | عمق في `readl`/`writel` وضمانات الـ memory ordering — مهم لفهم ليه الـ accessors بالشكل ده |

---

### توثيق الـ Kernel الرسمي

الـ `Documentation/` paths دي مباشرة متعلقة بالكود:

```
Documentation/driver-api/gpio/driver.rst
```
الـ canonical reference لكتابة GPIO chip driver — بيشرح `struct gpio_chip`، الـ callbacks، الـ locking، والـ irqchip integration.

```
Documentation/driver-api/gpio/index.rst
```
نقطة الدخول للـ GPIO subsystem documentation كله.

```
Documentation/driver-api/gpio/board.rst
```
بيشرح إزاي الـ board code بتربط الـ GPIO lines بالـ consumers — مهم لفهم `bgpio_init()` في سياقه.

```
Documentation/driver-api/gpio/consumer.rst
```
الـ consumer API — بيكمل الصورة من الطرف التاني.

**الروابط الـ online:**
- [GPIO Driver Interface](https://docs.kernel.org/driver-api/gpio/driver.html)
- [General Purpose Input/Output (GPIO)](https://docs.kernel.org/driver-api/gpio/index.html)
- [GPIO Mappings](https://www.kernel.org/doc/html/latest/driver-api/gpio/board.html)

---

### الـ Source Code على GitHub

```
drivers/gpio/gpio-mmio.c          — الـ driver نفسه
drivers/gpio/gpiolib.c            — الـ core library
include/linux/gpio/driver.h       — الـ struct gpio_chip definition
include/linux/gpio/generic.h      — bgpio_init() declaration
```

- [gpio-mmio.c على torvalds/linux](https://github.com/torvalds/linux/blob/master/drivers/gpio/gpio-mmio.c)

---

### Commits مهمة

| الـ commit / الرابط | الأهمية |
|---------------------|---------|
| [gpio: mmio: Fix up inverted direction registers](https://github.com/raspberrypi/linux/commit/d799a4de0a250f1bdd99765bb8e55a5e2f469a1f) | fix لـ `BGPIOF_NO_OUTPUT` والـ inverted direction logic |
| [Linux-Kernel Archive: GPIO changes for v3.20](http://lkml.iu.edu/hypermail/linux/kernel/1502.1/00930.html) | إضافة `bgpio_set_multiple` وتحويل drivers كتير للـ MMIO library |

---

### نقاشات الـ Mailing List

| الرابط | الموضوع |
|--------|---------|
| [PATCH v3: gpio: mmio: handle "ngpios" properly in bgpio_init()](https://lore.kernel.org/lkml/202303050354.HH9DhJsr-lkp@intel.com/t/) | كيفية التعامل مع `ngpios` property في الـ Device Tree |
| [gpio: mmio: Support two direction registers — Patchwork](https://patchwork.ozlabs.org/patch/1046720/) | إضافة دعم register تانية لـ direction — بيشرح `dirin`/`dirout` design |
| [gpio: mmio: Make pin2mask() a private business](https://patches.linaro.org/patch/116504/) | إزالة vtable overhead من كل gpiochip callback |
| [gpio: mmio: add DT support for memory-mapped GPIOs](https://patchwork.kernel.org/project/linux-arm-kernel/patch/3e75e8316ceac62f61f3ffc839faaac8789ca936.1463078228.git.chunkeey@googlemail.com/) | إضافة Device Tree support للـ driver |
| [LKML: Re: gpio: mmio: drop the "bgpio" prefix](https://lkml.org/lkml/2025/10/27/1931) | نقاش حول إعادة تسمية الـ API من `bgpio_` لـ اسم أوضح |

---

### Kernelnewbies.org

| الرابط | المحتوى |
|--------|---------|
| [Linux_6.13 — Kernel Newbies](https://kernelnewbies.org/Linux_6.13) | تغييرات الـ GPIO subsystem في آخر إصدارات الـ kernel |
| [Linux_6.6 — Kernel Newbies](https://kernelnewbies.org/Linux_6.6) | يتضمن تحديثات الـ GPIO driver infrastructure |
| [LinuxChanges — Kernel Newbies](https://kernelnewbies.org/LinuxChanges) | تاريخ التغييرات عبر الإصدارات |
| [Using gpio from device tree on platform devices](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2015-August/014911.html) | نقاش عملي عن الـ DT integration مع GPIO platform drivers |
| [devm_gpiod_get usage](http://lists.kernelnewbies.org/pipermail/kernelnewbies/2016-August/016707.html) | كيفية استخدام الـ managed GPIO descriptor API |

---

### eLinux.org

| الرابط | المحتوى |
|--------|---------|
| [EBC GPIO via mmap](https://elinux.org/EBC_GPIO_via_mmap) | مثال عملي على الوصول لـ GPIO registers عبر `/dev/mem` و`mmap` — بيوضح نفس الـ concept اللي gpio-mmio بيعمله في الـ kernel space |
| [New GPIO Interface for User Space (ELCE 2017)](https://elinux.org/images/7/74/Elce2017_new_GPIO_interface.pdf) | presentation بتشرح الـ character device API اللي حلت محل sysfs |
| [Introduction to pin muxing and GPIO control under Linux (ELC 2021)](https://elinux.org/images/a/a7/ELC-2021_Introduction_to_pin_muxing_and_GPIO_control_under_Linux.pdf) | شرح شامل لـ pinctrl وGPIO subsystems مع بعض |
| [EBC gpio Polling and Interrupts](https://elinux.org/EBC_gpio_Polling_and_Interrupts) | تفاصيل عملية عن GPIO interrupts من user space |

---

### كتب موصى بيها

#### Linux Device Drivers (LDD3)
- **الفصل 9**: Data Types in the Kernel — الـ `u8`/`u16`/`u32` types المستخدمة في الـ read/write accessors
- **الفصل 9**: I/O Memory — `ioremap`، `readl`/`writel`، الـ barriers
- متاح مجانًا: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd ed.)
- **الفصل 17**: Devices and Modules — فهم `platform_driver` و`module_platform_driver`
- **الفصل 20**: Patches, Hacking, and the Community — لفهم سياق الـ patches المذكورة فوق

#### Embedded Linux Primer — Christopher Hallinan (2nd ed.)
- **الفصل 15**: Kernel Initialization — بيشرح إزاي الـ platform devices بتتسجل أثناء الـ boot
- **الفصل 16**: Using the GPIO Subsystem — تطبيقات عملية على hardware حقيقي

#### Linux Driver Development for Embedded Processors — Alberto Liberal de los Ríos
- فصل كامل عن GPIO character drivers و MMIO mapping على ARM

---

### مصطلحات للبحث

لو عايز تكمل بحثك، استخدم الـ search terms دي:

```
bgpio_init linux kernel
gpio_chip set_direction_input
gpio mmio platform_device resource
BGPIOF_READ_OUTPUT_REG_SET
ioread32 iowrite32 linux gpio
gpiod_get_direction kernel
devm_bgpio_init
gpio-generic.c linux
struct gpio_chip linux kernel
gpio mmio big endian kernel
```

**للبحث في الـ kernel source مباشرة:**
```bash
# ابحث عن كل drivers بتستخدم bgpio_init
git log --all --oneline -- drivers/gpio/gpio-mmio.c

# شوف من بيستخدم bgpio_init
grep -r "bgpio_init" drivers/gpio/

# تتبع تغييرات struct gpio_chip
git log --follow -p include/linux/gpio/driver.h
```
## Phase 8: Writing simple module

### الـ Target: `gpio_generic_chip_init()`

الدالة `gpio_generic_chip_init()` هي الوحيدة المـ export بـ `EXPORT_SYMBOL_GPL` في الملف، وهي نقطة الدخول الأساسية لأي driver بيحتاج يسجّل GPIO controller مبني على MMIO. كل مرة بيتعمل probe لـ platform device عنده GPIO من النوع ده، هيتعمل call لها — يعني هي المكان المثالي للـ hook.

سنستخدم **kprobe** على return point الدالة (kretprobe) عشان نقدر نشوف:
- اسم الـ device اللي اتسجّل
- عدد الـ GPIO lines (ngpio)
- عرض الـ register بالـ bits
- إذا كانت big-endian ولا لأ
- قيمة الـ return (نجاح أو فشل)

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kretprobe on gpio_generic_chip_init()
 * Logs every MMIO GPIO chip registration attempt with chip details.
 */

/* Core kernel headers */
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/gpio/driver.h>  /* struct gpio_chip */
#include <linux/gpio/generic.h> /* struct gpio_generic_chip, to_gpio_generic_chip */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("GPIO Watcher");
MODULE_DESCRIPTION("kretprobe on gpio_generic_chip_init to log MMIO GPIO registrations");

/* ------------------------------------------------------------------ */
/* Entry handler — called just BEFORE gpio_generic_chip_init() runs    */
/* ------------------------------------------------------------------ */
static int entry_handler(struct kretprobe_instance *ri, struct pt_regs *regs)
{
    /*
     * We don't need to do anything on entry in this example;
     * all interesting data is visible after the function returns
     * (ngpio gets populated inside the function).
     * Returning 0 means "proceed normally, call ret_handler too".
     */
    return 0;
}

/* ------------------------------------------------------------------ */
/* Return handler — called right AFTER gpio_generic_chip_init() returns */
/* ------------------------------------------------------------------ */
static int ret_handler(struct kretprobe_instance *ri, struct pt_regs *regs)
{
    /*
     * regs_return_value() extracts the return value of the probed function
     * from the architecture's return register (e.g. rax on x86-64).
     */
    long retval = regs_return_value(regs);

    /*
     * ri->fp points to the first argument of the probed call:
     *   gpio_generic_chip_init(struct gpio_generic_chip *chip, ...)
     *
     * We cast it back so we can walk the struct and read chip details.
     * This is safe here because the function has already finished and
     * the struct is still alive (it lives in the caller's probe() function).
     */
    struct gpio_generic_chip *gen_chip =
            (struct gpio_generic_chip *)ri->fp;

    /* The embedded gpio_chip holds the label and ngpio count */
    struct gpio_chip *gc = &gen_chip->gc;

    if (retval == 0) {
        /* Successful registration — dump chip summary */
        pr_info("[gpio_mmio_watch] chip registered: label=\"%s\" "
                "ngpio=%u bits=%d be_bits=%d\n",
                gc->label  ? gc->label : "(null)",
                gc->ngpio,
                gen_chip->bits,
                (int)gen_chip->be_bits);

        /* Show the shadow data register value at init time */
        pr_info("[gpio_mmio_watch]   sdata=0x%08lx  sdir=0x%08lx\n",
                gen_chip->sdata, gen_chip->sdir);
    } else {
        /* Failed registration — still useful to know */
        pr_warn("[gpio_mmio_watch] chip init FAILED: label=\"%s\" ret=%ld\n",
                gc->label ? gc->label : "(null)", retval);
    }

    return 0; /* always return 0 from a kretprobe handler */
}

/* ------------------------------------------------------------------ */
/* kretprobe descriptor                                                 */
/* ------------------------------------------------------------------ */
static struct kretprobe gpio_init_probe = {
    .handler        = ret_handler,   /* called on function return  */
    .entry_handler  = entry_handler, /* called on function entry   */
    .maxactive      = 4,             /* max concurrent instances   */
    /* symbol_name tells the kprobe subsystem which function to hook */
    .kp.symbol_name = "gpio_generic_chip_init",
};

/* ------------------------------------------------------------------ */
/* module_init                                                          */
/* ------------------------------------------------------------------ */
static int __init gpio_mmio_watch_init(void)
{
    int ret;

    /*
     * register_kretprobe() patches the first bytes of the target function
     * with a breakpoint (int3 on x86) so the kernel diverts execution to
     * our handlers.  It returns 0 on success, negative on error (e.g. if
     * the symbol does not exist or CONFIG_KPROBES is not set).
     */
    ret = register_kretprobe(&gpio_init_probe);
    if (ret < 0) {
        pr_err("[gpio_mmio_watch] register_kretprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[gpio_mmio_watch] kretprobe registered on gpio_generic_chip_init\n");
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit                                                          */
/* ------------------------------------------------------------------ */
static void __exit gpio_mmio_watch_exit(void)
{
    /*
     * unregister_kretprobe() removes the breakpoint and waits for any
     * in-flight handler to finish before returning.  Forgetting this step
     * would leave a dangling breakpoint pointing to freed module memory,
     * causing a kernel crash on the next GPIO registration.
     */
    unregister_kretprobe(&gpio_init_probe);
    pr_info("[gpio_mmio_watch] kretprobe unregistered. "
            "missed probes: %d\n", gpio_init_probe.nmissed);
}

module_init(gpio_mmio_watch_init);
module_exit(gpio_mmio_watch_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/kprobes.h` | يعرّف `struct kretprobe`, `register_kretprobe`, `regs_return_value` |
| `linux/gpio/driver.h` | يعرّف `struct gpio_chip` و constants زي `GPIO_LINE_DIRECTION_*` |
| `linux/gpio/generic.h` | يعرّف `struct gpio_generic_chip` و الماكرو `to_gpio_generic_chip` |

الـ `linux/module.h` و `linux/kernel.h` ضروريين لأي kernel module يستخدم `pr_info` و `MODULE_*` macros.

---

#### الـ `entry_handler`

بنسجّله عشان الـ kretprobe framework يحتاج نقطة entry لو أردنا نحفظ arguments قبل ما الدالة تتنفذ، هنا خلّيناه فاضي وبيرجّع `0` يعني "اكمل طبيعي". لو أردنا نحفظ قيمة argument معينة في `ri->data` كنا نعمله هنا.

---

#### الـ `ret_handler`

**الـ `ri->fp`** بيحتفظ بقيمة الـ frame pointer اللي بيشاور على أول argument لما الدالة اتنادت — ده هو `struct gpio_generic_chip *chip`. بعد ما الدالة خلصت، نقدر نقراه بأمان عشان الـ struct لسه موجود في stack الـ caller.

**الـ `regs_return_value(regs)`** بيجيب قيمة الـ return register بطريقة portable عبر architectures مختلفة.

بنطبع `label` و `ngpio` و `bits` عشان دي المعلومات الأساسية اللي بتعرّف الـ GPIO chip المتسجّلة، وبنطبع `sdata` و `sdir` عشان يوضّحوا الحالة الأولية للـ shadow registers اللي بيحفظ فيها الـ driver آخر قيمة كتبها على الـ hardware.

---

#### الـ `struct kretprobe`

| الحقل | القيمة | السبب |
|---|---|---|
| `handler` | `ret_handler` | الـ callback اللي بيتشغّل عند الـ return |
| `entry_handler` | `entry_handler` | الـ callback عند الـ entry |
| `maxactive` | `4` | أقصى عدد instances متزامنين — بيحمي من ضياع probes لو أكتر من CPU نادت الدالة في نفس الوقت |
| `kp.symbol_name` | `"gpio_generic_chip_init"` | اسم الـ symbol المـ export اللي هنضع عليه الـ hook |

---

#### الـ `module_exit` وأهمية `unregister_kretprobe`

لو خرج الـ module من غير ما يعمل `unregister_kretprobe`، الـ breakpoint هيفضل موجود وبيشاور على كود اتمسح من الذاكرة — أول call بعد كده هتعمل kernel panic. كمان `gpio_init_probe.nmissed` بيخلّيك تعرف عدد المرات اللي اتكلت فيها الدالة وما اتنفّذش فيها الـ handler بسبب الـ maxactive.

---

### Makefile للتجميع

```makefile
obj-m += gpio_mmio_watch.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

### تجربة الـ Module

```bash
# تحميل الـ module
sudo insmod gpio_mmio_watch.ko

# شوف الـ log (لو فيه GPIO devices بتتسجّل دلوقتي)
sudo dmesg | grep gpio_mmio_watch

# simulate registration عن طريق re-bind لأي platform GPIO device
echo "some-gpio-device" | sudo tee /sys/bus/platform/drivers/basic-mmio-gpio/unbind
echo "some-gpio-device" | sudo tee /sys/bus/platform/drivers/basic-mmio-gpio/bind

# إزالة الـ module
sudo rmmod gpio_mmio_watch
sudo dmesg | tail -5
```

مثال للـ output المتوقع:

```
[gpio_mmio_watch] kretprobe registered on gpio_generic_chip_init
[gpio_mmio_watch] chip registered: label="fpga-gpio0" ngpio=32 bits=32 be_bits=0
[gpio_mmio_watch]   sdata=0x00000000  sdir=0x0000ffff
[gpio_mmio_watch] kretprobe unregistered. missed probes: 0
```
