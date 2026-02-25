## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الـ file ده جزء من **GPIO Regmap** subsystem — وده subsystem صغير جوه الـ GPIO framework الأكبر في Linux kernel. الـ maintainer بتاعه هو Michael Walle وحالته "Maintained".

---

### المشكلة اللي بيحلها

تخيل إن عندك شريحة PMIC أو MFD (Multi-Function Device) زي شريحة تحكم في الطاقة على board صغير. الشريحة دي فيها GPIO lines — يعني pins ممكن تتحكم فيها software — بس مش متوصلة مباشرة بالـ CPU. الـ CPU بيكلم الشريحة دي عن طريق I2C أو SPI، والكلام بيتم عن طريق registers جوه الشريحة.

كل driver كان بيعيد اختراع العجلة: يقرأ register، يعمل AND/OR على bit معين، يكتب تاني. كل ده كود متكرر في عشرات الـ drivers.

**الـ regmap** اخترعها الـ kernel عشان توحد طريقة الوصول للـ registers (سواء I2C أو SPI أو MMIO) خلف API موحد مع caching وغيره.

**الـ gpio-regmap.c** جاء بعد كده وقال: "ما دام الـ GPIO chip بتاعك مجرد registers في regmap — خليني أكتب الـ gpio_chip callbacks مرة واحدة للكل، وانت بس قولي عناوين الـ registers."

---

### القصة بالتفصيل البسيط

تخيل عندك PMIC فيها 8 GPIO pins. كل pin محكوم بـ bit في registers:

```
Register 0x10 → DATA register  (اقرأ قيمة الـ pin)
Register 0x11 → SET register   (اكتب 1 تشغل الـ pin)
Register 0x12 → CLEAR register (اكتب 1 تطفي الـ pin)
Register 0x13 → DIR register   (حدد input أو output)
```

قبل gpio-regmap، كل driver PMIC كان بيكتب نفس الكود: `regmap_read`, `regmap_update_bits`, إلخ — بس بعناوين مختلفة.

**بعد gpio-regmap**، صاحب الـ driver بيعمل كده بس:

```c
static const struct gpio_regmap_config my_pmic_gpio_config = {
    .parent        = dev,
    .regmap        = regmap,
    .ngpio         = 8,
    .reg_dat_base  = GPIO_REGMAP_ADDR(0x10),
    .reg_set_base  = GPIO_REGMAP_ADDR(0x11),
    .reg_clr_base  = GPIO_REGMAP_ADDR(0x12),
    .reg_dir_out_base = GPIO_REGMAP_ADDR(0x13),
};

devm_gpio_regmap_register(dev, &my_pmic_gpio_config);
```

خلاص. الـ gpio-regmap بيعمل الباقي كله.

---

### إيه اللي بيعمله الـ file ده تحديداً

الـ `gpio-regmap.c` هو الـ **generic driver core** اللي بيوفر:

| الوظيفة | الـ Function |
|---|---|
| قراءة قيمة pin | `gpio_regmap_get()` |
| كتابة قيمة pin | `gpio_regmap_set()` أو `gpio_regmap_set_with_clear()` |
| معرفة اتجاه pin | `gpio_regmap_get_direction()` |
| تغيير اتجاه pin | `gpio_regmap_set_direction()` |
| تحويل offset → register + mask | `gpio_regmap_simple_xlate()` |
| تسجيل الـ controller | `gpio_regmap_register()` |
| إلغاء التسجيل | `gpio_regmap_unregister()` |
| نسخة مُدارة للـ lifecycle | `devm_gpio_regmap_register()` |

---

### الـ reg_mask_xlate: الفكرة الذكية

لما عندك أكتر من 1 register للـ GPIO (مثلاً 32 GPIO pin موزعين على 4 registers × 8 bits)، محتاج تحسب: "الـ GPIO رقم 17 ده في أنهي register وأنهي bit؟"

الـ `gpio_regmap_simple_xlate()` بتعمل الحسبة دي:

```c
unsigned int line   = offset % gpio->ngpio_per_reg;  // bit number
unsigned int stride = offset / gpio->ngpio_per_reg;  // register index
*reg  = base + stride * gpio->reg_stride;            // register address
*mask = BIT(line);                                   // bit mask
```

لو الـ hardware بتاعك عنده layout غريب، تقدر تدي callback خاص `reg_mask_xlate` في الـ config.

---

### تفصيلة مهمة: reg_dat vs reg_set

بعض الـ hardware عنده register واحد بيستخدمه للقراءة والكتابة (read-modify-write). غيره عنده register منفصل للـ SET وتاني للـ CLEAR (زي GPIO hardware سريع).

الـ gpio-regmap بيتعامل مع الحالتين:

- لو `reg_dat_base == reg_set_base`: يعني نفس الـ register، فبيستخدم `regmap_read_bypassed()` عشان يتجنب قراءة الـ cache المتغير من output writes.
- لو `reg_clr_base` موجود: بيستخدم write للـ set register أو الـ clear register بدل read-modify-write.

---

### الـ IRQ Support

الـ file بيدعم optional interrupt handling بطريقتين:
1. تدي `irq_domain` جاهز.
2. تدي `regmap_irq_chip` وهو يعمل الـ domain لوحده (لو `CONFIG_REGMAP_IRQ` مفعل).

---

### ASCII Diagram: مكانه في النظام

```
  Driver بتاع PMIC/MFD
  (e.g. gpio-sl28cpld.c)
          │
          │  يعمل config struct ويستدعي
          ▼
  devm_gpio_regmap_register()
  ┌─────────────────────────────┐
  │       gpio-regmap.c         │
  │  ┌─────────────────────┐    │
  │  │   struct gpio_chip  │    │
  │  │   .get  = ...       │    │
  │  │   .set  = ...       │    │
  │  │   .get_direction    │    │
  │  └─────────────────────┘    │
  │           │                 │
  │    regmap_read/write        │
  └───────────┼─────────────────┘
              │
  ┌───────────▼─────────────────┐
  │        regmap core          │
  │  (I2C / SPI / MMIO backend) │
  └─────────────────────────────┘
              │
  ┌───────────▼─────────────────┐
  │    Physical Hardware        │
  │  (GPIO registers in chip)   │
  └─────────────────────────────┘
```

---

### الـ Files المكوّنة للـ Subsystem

| الملف | الدور |
|---|---|
| `drivers/gpio/gpio-regmap.c` | الـ core implementation — الملف ده |
| `include/linux/gpio/regmap.h` | الـ public API وتعريف `gpio_regmap_config` |
| `include/linux/gpio/driver.h` | تعريف `gpio_chip` وكل الـ GPIO driver framework |
| `drivers/base/regmap/regmap.c` | الـ regmap core اللي بيتكلم مع الـ hardware |
| `include/linux/regmap.h` | الـ regmap API headers |

### أمثلة Drivers بتستخدم gpio-regmap

| الـ Driver | الشريحة |
|---|---|
| `drivers/gpio/gpio-sl28cpld.c` | SL28CPLD CPLD chip |
| `drivers/gpio/gpio-fxl6408.c` | FXL6408 I2C GPIO expander |
| `drivers/gpio/gpio-ds4520.c` | DS4520 I2C GPIO |
| `drivers/gpio/gpio-tn48m.c` | TN48M switch GPIO |
| `drivers/iio/adc/ad7173.c` | AD7173 ADC مع GPIO |

### ملفات مهمة المقرئ لازم يعرفها

- **`include/linux/gpio/regmap.h`** — أهم ملف: فيه `gpio_regmap_config` اللي بيتملى قبل الـ register.
- **`include/linux/gpio/driver.h`** — فيه `gpio_chip` struct الأساسي اللي كل driver GPIO بيبنيه.
- **`include/linux/regmap.h`** — فيه `regmap_read`, `regmap_write`, `regmap_update_bits` وغيرها.
- **`drivers/gpio/gpiolib.c`** — الـ GPIO core library اللي `gpiochip_add_data` و`gpiochip_remove` معرفين فيها.
## Phase 2: شرح الـ GPIO Regmap Framework

### المشكلة — ليه الـ subsystem ده موجود؟

في أي SoC أو PMIC بتتعامل معاه عبر I2C أو SPI، بتلاقي GPIO expanders زي PCA9538، MAX7310، أو BD9571MWV. كل chip من دول بيبقى فيها registers للـ input (DAT)، output (SET)، direction (DIR)، وأحياناً clear (CLR). المشكلة إن كل driver كان بيعيد كتابة نفس الـ boilerplate بالظبط:

- اقرأ الـ register
- حوّل الـ GPIO offset لـ bit mask
- اعمل regmap_read/write

ده بيحصل في عشرات الـ drivers — **gpio-max77620.c**، **gpio-bd9571mwv.c**، **gpio-da9052.c**. كل واحد بيحتوي على كود متشابه جداً بس مكتوب من أول. أي bug في الـ logic بيتكرر في كل مكان، وأي تحسين بيحتاج تعديل عشرات الملفات.

---

### الحل — الـ gpio-regmap Framework

الـ kernel قدّم **`gpio-regmap`** كـ generic GPIO driver جاهز يشتغل مع أي hardware بيوفر registers standard للـ GPIO control عن طريق regmap. الفكرة:

1. الـ driver بيوصف الـ hardware بـ `struct gpio_regmap_config` — base addresses للـ registers بس
2. الـ `gpio-regmap` يتولى كل الـ callbacks للـ `gpio_chip` (get، set، direction)
3. أي تعقيد في الـ address translation؟ الـ driver يوفر custom `reg_mask_xlate` callback

---

### الـ Subsystems التانية اللي لازم تعرفها الأول

- **regmap subsystem**: abstraction layer فوق الـ I2C/SPI/MMIO يخليك تتعامل مع registers الـ device بـ unified API مع caching ولocking. كل قراءة/كتابة في gpio-regmap بتعدي عليه.
- **gpiolib**: الـ GPIO core في الـ kernel. بيمسك قائمة كل الـ GPIO controllers المسجلين، وبيعرض الـ API للـ consumers. الـ `gpio_chip` هو الـ interface ما بين الـ driver والـ gpiolib.
- **IRQ domain subsystem**: نظام الـ kernel لترجمة hardware IRQ numbers لـ Linux IRQ numbers — الـ gpio-regmap بيدعم ربط الـ GPIO lines بـ IRQ domain.

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                  User Space / Kernel Consumers                   │
│           gpiod_get_value() / gpiod_set_value() / etc.          │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                        gpiolib core                              │
│                   (drivers/gpio/gpiolib.c)                       │
│   owns: gpio_device, gpio_desc, namespacing, sysfs, chardev     │
└────────────────────────────┬────────────────────────────────────┘
                             │ calls gpio_chip callbacks
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                   gpio-regmap framework                          │
│              (drivers/gpio/gpio-regmap.c)                        │
│                                                                  │
│  struct gpio_regmap {                                            │
│      struct gpio_chip  ◄── registered with gpiolib              │
│      struct regmap *   ◄── points to regmap instance            │
│      reg_dat_base      ◄── input register(s) base               │
│      reg_set_base      ◄── output set register(s) base          │
│      reg_clr_base      ◄── output clear register(s) base        │
│      reg_dir_in/out    ◄── direction register(s) base           │
│      reg_mask_xlate()  ◄── callback: offset → (reg, mask)       │
│  }                                                               │
└────────────────────────────┬────────────────────────────────────┘
                             │ regmap_read() / regmap_write()
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      regmap subsystem                            │
│              (drivers/base/regmap/regmap.c)                      │
│   owns: caching, locking, format conversion, bus abstraction    │
└────────────────────────────┬────────────────────────────────────┘
                             │ actual bus transfer
                ┌────────────┴──────────┐
                ▼                       ▼
        ┌──────────────┐       ┌──────────────────┐
        │  I2C driver  │       │   SPI driver     │
        └──────┬───────┘       └────────┬─────────┘
               │                        │
        ┌──────▼───────────────────────▼─────────┐
        │       Hardware: PMIC / GPIO Expander    │
        │   e.g. MAX77620, BD9571MWV, DA9052      │
        └────────────────────────────────────────┘
```

---

### الـ Analogy — مبنى الشركة والسنترال

تخيّل شركة كبيرة فيها مبنى واحد فيه 100 موظف — ده الـ SoC GPIO مباشر، تتكلم مع أي واحد فيه مباشرةً. دلوقتي الشركة افتتحت فروع في مدن تانية، وكل فرع فيه موظفين كمان. لو عايز تكلم موظف في فرع:

1. بتتصل بـ **سنترال الفرع** (الـ I2C/SPI bus)
2. السنترال بيوصلك بـ **مكتب الفرع** (الـ regmap instance)
3. مكتب الفرع بيحوّل طلبك لـ **رقم داخلي** (register address + bit mask)
4. الموظف (الـ GPIO line) بيستجاوب

الـ **gpio-regmap framework** هو زي **نظام التحويل المركزي** — كل من عايز يكلم موظف في أي فرع بيتبع نفس الإجراء.

| جزء الـ analogy | المقابل في الـ kernel |
|---|---|
| مبنى الشركة المركزي | SoC GPIO controller مع MMIO مباشر |
| الفرع في مدينة تانية | PMIC أو GPIO expander على I2C/SPI |
| السنترال | I2C/SPI bus driver |
| مكتب الفرع | `struct regmap` instance |
| دليل الأرقام الداخلية للفرع | `reg_dat_base`, `reg_set_base`, إلخ |
| نظام التحويل المركزي | gpio-regmap framework |
| رقم الموظف (بالترتيب في المبنى) | GPIO offset |
| الرقم الداخلي الفعلي بالفرع | register address + bit mask |
| نظام التوصيل للرقم الداخلي | `reg_mask_xlate()` callback |
| طلب "فين الموظف ده؟" | `gpio_regmap_get_direction()` |
| طلب "وصّله الرسالة دي" | `gpio_regmap_set()` |
| مكتب الفرع بيختار هيكلّم مين من التليفونين | `reg_dat_base == reg_set_base` logic |

---

### الـ Core Abstraction — الفكرة المحورية

الـ core abstraction في gpio-regmap هي فكرة الـ **Register-Offset Mapping**:

كل GPIO line في الـ hardware عبارة عن **bit واحد في register معين**. الـ framework بيحتاج يعرف بس:
1. إيه **base address** لكل نوع من الـ registers (data, set, clear, direction)
2. إزاي يـ**translate** رقم الـ GPIO (offset) لـ (register address, bit mask)

ده الـ `reg_mask_xlate` callback — وله default implementation اسمها `gpio_regmap_simple_xlate`:

```c
static int gpio_regmap_simple_xlate(struct gpio_regmap *gpio,
                                    unsigned int base, unsigned int offset,
                                    unsigned int *reg, unsigned int *mask)
{
    /* which bit inside the register */
    unsigned int line   = offset % gpio->ngpio_per_reg;
    /* which register in the bank (0, 1, 2, ...) */
    unsigned int stride = offset / gpio->ngpio_per_reg;

    *reg  = base + stride * gpio->reg_stride; /* absolute register address */
    *mask = BIT(line);                        /* single-bit mask */

    return 0;
}
```

يعني لو عندك 16 GPIO على 2 registers كل واحد 8 bits (`ngpio_per_reg=8`, `reg_stride=1`):

```
GPIO 0  → reg = base+0, mask = 0x01  (BIT 0)
GPIO 7  → reg = base+0, mask = 0x80  (BIT 7)
GPIO 8  → reg = base+1, mask = 0x01  (BIT 0 of next register)
GPIO 15 → reg = base+1, mask = 0x80  (BIT 7 of next register)
```

---

### الـ Structs وعلاقتها ببعض

```
┌──────────────────────────────────────────────────────────────┐
│          gpio_regmap_config  (input from parent driver)      │
│  parent      *device                                         │
│  regmap      *regmap  ──────────────────────────────────┐   │
│  reg_dat_base  (e.g. 0x10)                               │   │
│  reg_set_base  (e.g. 0x10 or 0x11)                       │   │
│  reg_clr_base  (e.g. 0x12, optional)                     │   │
│  reg_dir_in_base / reg_dir_out_base                      │   │
│  ngpio         (e.g. 16)                                 │   │
│  ngpio_per_reg (e.g. 8)                                  │   │
│  reg_stride    (e.g. 1)                                  │   │
│  reg_mask_xlate()  ─────────────────────────┐            │   │
│  fixed_direction_output (bitmap, optional)  │            │   │
│  drvdata (opaque)                           │            │   │
└─────────────────────────────────────────────┼────────────┼───┘
                    gpio_regmap_register() copies           │
                                             │              │
                                             ▼              ▼
┌──────────────────────────────────────────────────────────────┐
│                 gpio_regmap  (runtime state, heap allocated) │
│                                                              │
│  parent      *device                                         │
│  regmap      *regmap  ◄──────────────────────── regmap layer │
│  reg_dat_base                                                │
│  reg_set_base                                                │
│  reg_clr_base                                                │
│  reg_dir_in_base / reg_dir_out_base                          │
│  ngpio_per_reg                                               │
│  reg_stride                                                  │
│  reg_mask_xlate()  ◄── custom callback OR simple_xlate       │
│  fixed_direction_output  (bitmap copy, optional)             │
│  driver_data  (returned via gpio_regmap_get_drvdata())       │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  gpio_chip  (embedded, NOT a pointer)               │    │
│  │  .get             = gpio_regmap_get                 │    │
│  │  .set             = gpio_regmap_set                 │    │─── registered
│  │               OR    gpio_regmap_set_with_clear      │    │    with gpiolib
│  │  .get_direction   = gpio_regmap_get_direction       │    │
│  │  .direction_input = gpio_regmap_direction_input     │    │
│  │  .direction_output= gpio_regmap_direction_output    │    │
│  │  .request         = gpiochip_generic_request        │    │
│  │  .free            = gpiochip_generic_free           │    │
│  │  .can_sleep       = regmap_might_sleep(regmap)      │    │
│  └─────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
                             │ gpiochip_add_data(chip, gpio)
                             ▼
┌──────────────────────────────────────────────────────────────┐
│                     gpiolib core                             │
│   gpio_device (wraps gpio_chip, adds namespace, chardev)    │
└──────────────────────────────────────────────────────────────┘
```

**ملاحظة**: الـ `gpio_chip` بيتضمَّن **مباشرة** (embedded) جوه الـ `gpio_regmap`. وعشان نوصل من الـ `gpio_chip` pointer للـ `gpio_regmap` الأب، بنستخدم `gpiochip_get_data(chip)` اللي بترجع الـ `gpio` pointer اللي اتحط في `gpiochip_add_data(chip, gpio)`.

---

### مخطط قرار الـ set callback

```
gpio_regmap_register()
        │
        ▼
reg_set_base && reg_clr_base ─── YES ──▶ chip->set = gpio_regmap_set_with_clear
        │                                  (كتابة mask في SET أو CLR register)
        NO
        │
        ▼
reg_set_base فقط ─────────── YES ──▶ chip->set = gpio_regmap_set
        │                              (regmap_write_bits أو regmap_update_bits)
        NO
        │
        ▼
    chip->set = NULL  (input-only chip, reg_dat_base فقط)
```

---

### أنواع الـ Register Configurations المدعومة

#### 1. Input-Only (reg_dat_base فقط)
```
GPIO[n] ── DAT Register (read only via regmap_read)
```

#### 2. Output-Only (reg_set_base فقط)
```
GPIO[n] ── SET Register (write only, no DAT available)
```

#### 3. Bidirectional بـ register واحد للـ SET/DAT
```
reg_dat_base == reg_set_base (نفس العنوان)
GPIO[n] ── DATA/SET Register
           ├── read  → regmap_read_bypassed()  [skip cache — see below]
           └── write → regmap_write_bits()     [not regmap_update_bits]
```

#### 4. Set/Clear منفصلين (reg_set_base + reg_clr_base)
```
Write 1 ── SET Register  (bit = 1 sets the output HIGH)
Write 0 ── CLR Register  (bit = 1 clears the output LOW)
```

```c
static int gpio_regmap_set_with_clear(struct gpio_chip *chip,
                                      unsigned int offset, int val)
{
    /* route to SET register for val=1, CLR register for val=0 */
    if (val)
        base = gpio_regmap_addr(gpio->reg_set_base);
    else
        base = gpio_regmap_addr(gpio->reg_clr_base);

    ret = gpio->reg_mask_xlate(gpio, base, offset, &reg, &mask);
    return regmap_write(gpio->regmap, reg, mask);
}
```

---

### التفصيلة المهمة: Cache Problem وـ regmap_read_bypassed

لما `reg_dat_base == reg_set_base` (نفس الـ register بيُستخدم للقراءة والكتابة — شائع في بعض الـ MMIO controllers)، بيحصل مشكلة:

الـ regmap cache بيحتفظ بآخر قيمة **كتبتها** في الـ register. لما بتقرأ GPIO input من نفس الـ register، القيمة الفيزيائية على الـ pin ممكن تكون مختلفة عن الـ cached value.

```c
/* ensure we don't spoil any register cache with pin input values */
if (gpio->reg_dat_base == gpio->reg_set_base)
    ret = regmap_read_bypassed(gpio->regmap, reg, &val);  /* hardware, not cache */
else
    ret = regmap_read(gpio->regmap, reg, &val);            /* normal cached read */
```

الـ `regmap_read_bypassed` بيقرأ مباشرة من الـ hardware من غير ما يلمس الـ cache — بالتالي القيم المحفوظة في الـ cache للـ output bits تفضل سليمة ومش بتتخبط بقراءة الـ inputs.

نفس المنطق في الـ set:
```c
if (gpio->reg_dat_base == gpio->reg_set_base)
    ret = regmap_write_bits(gpio->regmap, reg, mask, mask_val);
    /* regmap_write_bits: does read-from-cache + merge + write to hw */
else
    ret = regmap_update_bits(gpio->regmap, reg, mask, mask_val);
    /* regmap_update_bits: reads hw, merges, writes — standard path */
```

---

### Direction Handling — كيف تتعامل مع الاتجاه

الـ framework بيدعم 3 سيناريوهات للـ direction بالترتيب التالي في `gpio_regmap_get_direction`:

**1. Fixed Direction من Bitmap:**
```c
if (gpio->fixed_direction_output) {
    if (test_bit(offset, gpio->fixed_direction_output))
        return GPIO_LINE_DIRECTION_OUT;
    else
        return GPIO_LINE_DIRECTION_IN;
}
```
لما فيه GPIO lines مختلطة في نفس الـ register — بعضها output ثابت وبعضها input ثابت.

**2. Implicit Direction من وجود الـ registers:**
```c
if (gpio->reg_dat_base && !gpio->reg_set_base)
    return GPIO_LINE_DIRECTION_IN;   /* input-only chip */
if (gpio->reg_set_base && !gpio->reg_dat_base)
    return GPIO_LINE_DIRECTION_OUT;  /* output-only chip */
```

**3. Direction Register:**
```c
/* reg_dir_out_base: bit=1 means OUTPUT → invert=0 */
/* reg_dir_in_base:  bit=1 means INPUT  → invert=1 */
if (!!(val & mask) ^ invert)
    return GPIO_LINE_DIRECTION_OUT;
else
    return GPIO_LINE_DIRECTION_IN;
```

الـ `invert` flag بيعالج الفرق بين الـ hardware اللي بيقول "1=output" والـ hardware اللي بيقول "1=input" — نفس الـ logic بس direction register polarity مختلف.

---

### التفصيلة الغريبة: GPIO_REGMAP_ADDR_ZERO

```c
#define GPIO_REGMAP_ADDR_ZERO ((unsigned int)(-1))   /* = 0xFFFFFFFF */
#define GPIO_REGMAP_ADDR(addr) ((addr) ? : GPIO_REGMAP_ADDR_ZERO)
```

المشكلة: الـ framework بيستخدم `0` كـ sentinel value يعني "الـ register ده مش محدد". بس لو الـ hardware register الفعلي عنده address = 0؟

الحل: الـ driver يكتب `GPIO_REGMAP_ADDR(0)` بدل `0` مباشرة — ده بيحوّله لـ `0xFFFFFFFF`. والـ `gpio_regmap_addr()` بتحوّله للـ 0 الحقيقي وقت الاستخدام:

```c
static unsigned int gpio_regmap_addr(unsigned int addr)
{
    if (addr == GPIO_REGMAP_ADDR_ZERO)
        return 0;    /* the real hardware address 0x00 */
    return addr;
}
```

---

### الـ devm Pattern

```c
struct gpio_regmap *devm_gpio_regmap_register(struct device *dev,
                                              const struct gpio_regmap_config *config)
{
    gpio = gpio_regmap_register(config);
    /* register cleanup callback with devres */
    devm_add_action_or_reset(dev, devm_gpio_regmap_unregister, gpio);
    return gpio;
}
```

الـ **devres** (device resource management) هو نظام الـ kernel لإدارة موارد الـ device تلقائياً. لما الـ device بتتـdetach أو فشل الـ probe، الـ kernel بيستدعي `devm_gpio_regmap_unregister` تلقائياً — ده بيقلل من bugs الـ resource leaks بشكل كبير.

---

### الـ IRQ Integration

لما بيكون عندك GPIO controller بيعمل interrupts، فيه طريقتين:

**1. External IRQ Domain:**
```c
/* driver creates irq_domain himself */
config->irq_domain = my_irq_domain;
/* gpio-regmap just calls: */
gpiochip_irqchip_add_domain(chip, irq_domain);
```

**2. regmap_irq (الأسهل):**
```c
/* driver provides a regmap_irq_chip descriptor */
config->regmap_irq_chip = &my_irq_chip;
config->regmap_irq_line = client->irq;

/* gpio-regmap does internally: */
regmap_add_irq_chip_fwnode(..., config->regmap_irq_chip, &gpio->irq_chip_data);
irq_domain = regmap_irq_get_domain(gpio->irq_chip_data);
gpiochip_irqchip_add_domain(chip, irq_domain);
```

الـ **regmap_irq subsystem** هو layer جاهز لـ interrupt controllers اللي بتشتغل بالـ regmap — بيتعامل مع الـ IRQ status/mask registers تلقائياً.

---

### الـ Framework بيملك إيه vs. بيفوّضه للـ driver

| المسؤولية | مين بيتولاها؟ |
|---|---|
| register read/write الفعلي | **regmap subsystem** |
| caching الـ registers وlocking | **regmap subsystem** |
| bus transport (I2C/SPI) | **bus driver** |
| تسجيل الـ GPIO chip مع الـ kernel | **gpio-regmap** |
| translate GPIO offset → (reg, mask) — default | **gpio-regmap** |
| translate GPIO offset → (reg, mask) — custom | **parent driver** (reg_mask_xlate) |
| get / set / direction logic | **gpio-regmap** |
| اختيار base registers | **parent driver** |
| عدد الـ GPIOs | **parent driver** (config.ngpio) |
| custom xlate function | **parent driver** (اختياري) |
| IRQ chip/domain setup | **parent driver** (بيجيب irq_domain أو regmap_irq_chip) |
| lifetime management | **devres** (لو استخدم devm_*) |

---

### Flow كامل — قراءة قيمة GPIO

```
consumer: gpiod_get_value(desc)
    │
    ▼ gpiolib core
    calls: chip->get(chip, offset)
    │
    ▼ gpio_regmap_get(chip, offset)
    │
    ├── 1. اختار الـ base address:
    │       reg_dat_base موجود؟ → استخدمه
    │       لأ → استخدم reg_set_base (output-only chip)
    │
    ├── 2. reg_mask_xlate(gpio, base, offset, &reg, &mask)
    │       line   = offset % ngpio_per_reg  → رقم الـ bit
    │       stride = offset / ngpio_per_reg  → رقم الـ register
    │       *reg   = base + stride * reg_stride
    │       *mask  = BIT(line)
    │
    ├── 3. هل reg_dat_base == reg_set_base؟
    │       نعم → regmap_read_bypassed(regmap, reg, &val)  [skip cache]
    │       لأ  → regmap_read(regmap, reg, &val)
    │
    └── 4. return !!(val & mask)  → 0 أو 1
```

---

### مثال حقيقي — GPIO Expander على I2C

تخيّل الـ **BD9571MWV** (PMIC من Rohm). فيه 8 GPIO lines على registers:
- `0x40`: GPIO data (input read)
- `0x41`: GPIO direction (1=output)

```c
static const struct gpio_regmap_config bd9571mwv_gpio_config = {
    .parent           = dev,
    .regmap           = regmap,
    .ngpio            = 8,
    .ngpio_per_reg    = 8,
    .reg_dat_base     = GPIO_REGMAP_ADDR(0x40),  /* input register at addr 0x40 */
    .reg_set_base     = GPIO_REGMAP_ADDR(0x40),  /* same register used for output */
    .reg_dir_out_base = GPIO_REGMAP_ADDR(0x41),  /* direction register */
};

devm_gpio_regmap_register(dev, &bd9571mwv_gpio_config);
```

الـ framework بيعمل:
1. يخصص `gpio_regmap` struct على الـ heap
2. يشوف `reg_dat_base == reg_set_base` → يختار `regmap_write_bits` للـ set و`regmap_read_bypassed` للـ get
3. يشوف `reg_dir_out_base` موجود → يفعّل `direction_input` و`direction_output` callbacks
4. يسجّل الـ chip مع gpiolib بـ `gpiochip_add_data()`
5. لما الـ device بتتـdetach، الـ devres بيستدعي `gpio_regmap_unregister` تلقائياً

من غير الـ gpio-regmap، الـ driver كان هيحتاج يكتب كل الـ callbacks يدوياً — ويتعامل بنفسه مع كل edge cases الـ cache والـ direction logic.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Macros والـ Config Options المهمة

#### الـ Macros الأساسية

| Macro | القيمة | الغرض |
|-------|--------|--------|
| `GPIO_REGMAP_ADDR_ZERO` | `(unsigned int)(-1)` | sentinel value بتقول إن عنوان الـ register هو `0` فعلاً (مش إنه مش موجود) |
| `GPIO_REGMAP_ADDR(addr)` | `(addr) ?: GPIO_REGMAP_ADDR_ZERO` | helper بيحوّل `0` لـ sentinel عشان تفرق بين "مش معيّن" و"عنوانه صفر" |
| `GPIO_LINE_DIRECTION_IN` | `1` | الـ GPIO في وضع input |
| `GPIO_LINE_DIRECTION_OUT` | `0` | الـ GPIO في وضع output |

#### الـ Config Options المؤثرة في الكود

| Config | أثره على الكود |
|--------|---------------|
| `CONFIG_REGMAP_IRQ` | بيفعّل دعم الـ IRQ عن طريق `regmap_irq_chip`، وبيضيف `regmap_irq_line` و`irq_chip_data` للـ `gpio_regmap` |
| `CONFIG_GPIOLIB_IRQCHIP` | بيفعّل `gpiochip_irqchip_add_domain()` الحقيقية — بدونه بترجع error |
| `CONFIG_LOCKDEP` | بيضيف lockdep keys تلقائياً في `gpiochip_add_data()` |

#### قواعد الـ Register Bases (cheatsheet)

| الحالة | `reg_dat_base` | `reg_set_base` | `reg_clr_base` | `reg_dir_*` |
|--------|:-:|:-:|:-:|:-:|
| Input-only | ✔ | ✗ | ✗ | ✗ |
| Output-only | ✗ | ✔ | ✗ | ✗ |
| Output مع clear منفصل | ✗ | ✔ | ✔ | ✗ |
| Bidirectional | ✔ | ✔ | اختياري | واحد منهم |
| `reg_dir_in_base` و`reg_dir_out_base` معاً | — | — | — | ❌ ممنوع |

---

### الـ Structs المهمة

#### 1. `struct gpio_regmap`

**الغرض:** الـ struct الداخلي الخاص بالدرايفر — بيمثّل instance واحد من GPIO controller مبني على regmap. مش مكشوف للـ users مباشرةً، بس بيتم تمريره كـ driver data في الـ gpio_chip.

| الحقل | النوع | الشرح |
|-------|------|--------|
| `parent` | `struct device *` | الـ device الأب (مثلاً i2c_client) |
| `regmap` | `struct regmap *` | الـ regmap handle اللي بيتتم من خلاله كل قراءة/كتابة |
| `gpio_chip` | `struct gpio_chip` | **مضمّن** (embedded)، هو اللي بيتسجل في gpiolib |
| `reg_stride` | `int` | المسافة بين كل register وإللي بعده من نفس النوع |
| `ngpio_per_reg` | `int` | عدد الـ GPIO lines في كل register واحد |
| `reg_dat_base` | `unsigned int` | عنوان base لـ input (data) registers |
| `reg_set_base` | `unsigned int` | عنوان base لـ set/output registers |
| `reg_clr_base` | `unsigned int` | عنوان base لـ clear registers (لو موجود) |
| `reg_dir_in_base` | `unsigned int` | عنوان base لـ direction registers بصيغة "1=input" |
| `reg_dir_out_base` | `unsigned int` | عنوان base لـ direction registers بصيغة "1=output" |
| `fixed_direction_output` | `unsigned long *` | bitmap بيحدد الـ lines اللي اتجاهها ثابت كـ output |
| `regmap_irq_line` | `int` | رقم الـ IRQ (بس لو `CONFIG_REGMAP_IRQ`) |
| `irq_chip_data` | `struct regmap_irq_chip_data *` | state الـ IRQ chip (بس لو `CONFIG_REGMAP_IRQ`) |
| `reg_mask_xlate` | function pointer | بيحوّل (base, offset) → (reg address, bitmask) |
| `driver_data` | `void *` | بيانات خاصة بالدرايفر، الـ framework مش بيلمسها |

---

#### 2. `struct gpio_regmap_config`

**الغرض:** الـ struct اللي بيملاه الدرايفر الخارجي قبل ما يستدعي `gpio_regmap_register()`. هو الـ "وصفة" اللي بتقول للـ framework كيف يبني الـ controller.

| الحقل | إجباري؟ | الشرح |
|-------|:-------:|--------|
| `parent` | ✔ | الـ device الأب — لازم يكون موجود |
| `regmap` | ✔ | الـ regmap handle |
| `fwnode` | ✗ | firmware node، لو فاضي بياخد fwnode الـ parent |
| `label` | ✗ | اسم وصفي، لو فاضي بياخد `dev_name(parent)` |
| `ngpio` | ✗ | عدد الـ GPIOs، لو صفر بيستدعي `gpiochip_get_ngpios()` |
| `names` | ✗ | array من الأسماء البديلة للـ lines |
| `reg_dat_base` | شرطي | base لـ input registers |
| `reg_set_base` | شرطي | base لـ set registers |
| `reg_clr_base` | ✗ | base لـ clear registers |
| `reg_dir_in_base` | ✗ | base لـ direction-in registers |
| `reg_dir_out_base` | ✗ | base لـ direction-out registers |
| `reg_stride` | ✗ | default=1، المسافة بين الـ registers |
| `ngpio_per_reg` | ✗ | default=ngpio، عدد الـ GPIOs في كل register |
| `irq_domain` | ✗ | لو الـ controller بيدعم IRQs |
| `fixed_direction_output` | ✗ | bitmap للـ lines ذات الاتجاه الثابت |
| `reg_mask_xlate` | ✗ | custom xlate، لو فاضي بيستخدم `gpio_regmap_simple_xlate` |
| `init_valid_mask` | ✗ | callback لتهيئة الـ valid mask |
| `drvdata` | ✗ | private data بيتخزن في `gpio_regmap.driver_data` |
| `regmap_irq_chip` | ✗ | (REGMAP_IRQ) وصف الـ IRQ chip |
| `regmap_irq_line` | ✗ | (REGMAP_IRQ) رقم الـ IRQ |
| `regmap_irq_flags` | ✗ | (REGMAP_IRQ) الـ `IRQF_*` flags |

---

#### 3. `struct gpio_chip`

**الغرض:** الـ struct القياسي اللي بيعرّفه gpiolib لأي GPIO controller. الـ `gpio_regmap` بيملاه ويسجله. بيحتوي على الـ ops callbacks اللي بيستدعيها الـ kernel عند أي عملية على GPIO.

| الحقل المهم | ما بيتضبطه gpio-regmap |
|------------|------------------------|
| `parent` | من `config->parent` |
| `fwnode` | من `config->fwnode` |
| `base` | `-1` (dynamic allocation) |
| `ngpio` | من `config->ngpio` أو `gpiochip_get_ngpios()` |
| `label` | من `config->label` أو `dev_name()` |
| `can_sleep` | من `regmap_might_sleep()` |
| `init_valid_mask` | من `config->init_valid_mask` |
| `request` | `gpiochip_generic_request` |
| `free` | `gpiochip_generic_free` |
| `get` | `gpio_regmap_get` |
| `set` | `gpio_regmap_set` أو `gpio_regmap_set_with_clear` |
| `get_direction` | `gpio_regmap_get_direction` |
| `direction_input` | `gpio_regmap_direction_input` (لو في dir registers) |
| `direction_output` | `gpio_regmap_direction_output` (لو في dir registers) |

---

### مخطط علاقات الـ Structs (ASCII)

```
Driver Code (e.g. mfd_cell driver)
          │
          │ fills
          ▼
┌─────────────────────────────────┐
│     gpio_regmap_config          │
│  ┌──────────────────────────┐   │
│  │ parent: *device          │   │
│  │ regmap: *regmap          │   │
│  │ reg_dat_base             │   │
│  │ reg_set_base             │   │
│  │ reg_clr_base             │   │
│  │ reg_dir_in_base          │   │
│  │ reg_dir_out_base         │   │
│  │ reg_mask_xlate()         │   │
│  │ init_valid_mask()        │   │
│  └──────────────────────────┘   │
└────────────┬────────────────────┘
             │ gpio_regmap_register()
             ▼
┌────────────────────────────────────────┐
│           gpio_regmap                  │
│  ┌──────────────────────────────────┐  │
│  │ parent ──────────────────────► device │
│  │ regmap ──────────────────────► regmap │
│  │ reg_dat/set/clr/dir_* bases     │  │
│  │ reg_stride, ngpio_per_reg       │  │
│  │ fixed_direction_output (bitmap) │  │
│  │ reg_mask_xlate()                │  │
│  │ driver_data ──────────────────► void* │
│  │                                 │  │
│  │ ┌───────────────────────────┐   │  │
│  │ │     gpio_chip  (embedded) │   │  │
│  │ │  parent, fwnode           │   │  │
│  │ │  ngpio, base=-1           │   │  │
│  │ │  can_sleep                │   │  │
│  │ │  get/set/direction ops    │   │  │
│  │ │  gpiodev ───────────────► gpio_device (gpiolib internal) │
│  │ └───────────────────────────┘   │  │
│  │                                 │  │
│  │ [CONFIG_REGMAP_IRQ]             │  │
│  │  regmap_irq_line                │  │
│  │  irq_chip_data ───────────────► regmap_irq_chip_data │
│  └──────────────────────────────────┘  │
└────────────────────────────────────────┘

gpio_chip.gpiodev ──► gpio_device (managed by gpiolib core)
gpio_chip.irq.domain ──► irq_domain (IRQ translation table)
```

---

### مخطط دورة الحياة (Lifecycle)

```
╔══════════════════════════════════════════════════════════╗
║                    CREATION PHASE                        ║
╠══════════════════════════════════════════════════════════╣
║                                                          ║
║  Driver fills gpio_regmap_config{}                       ║
║      │                                                   ║
║      ▼                                                   ║
║  gpio_regmap_register(config)  ─── OR ───                ║
║  devm_gpio_regmap_register(dev, config)                  ║
║      │                                                   ║
║      ├─ validate config (parent? reg_dat or reg_set?)    ║
║      ├─ kzalloc(gpio_regmap)                             ║
║      ├─ copy all bases/strides from config               ║
║      ├─ fill gpio_chip fields                            ║
║      ├─ wire up ops (get/set/direction callbacks)         ║
║      ├─ gpiochip_get_ngpios() if ngpio==0                ║
║      ├─ bitmap_alloc() for fixed_direction_output        ║
║      ├─ default reg_mask_xlate if not provided           ║
║      │                                                   ║
║      ▼                                                   ║
║  gpiochip_add_data(chip, gpio)  ◄── registers with gpiolib ║
║      │                                                   ║
║      ├─[CONFIG_REGMAP_IRQ && irq_chip set]               ║
║      │   regmap_add_irq_chip_fwnode()                    ║
║      │   regmap_irq_get_domain()  → irq_domain           ║
║      │                                                   ║
║      ├─[irq_domain present]                              ║
║      │   gpiochip_irqchip_add_domain(chip, irq_domain)  ║
║      │                                                   ║
║      └──► return gpio_regmap*  (success)                 ║
║                                                          ║
╠══════════════════════════════════════════════════════════╣
║                    USAGE PHASE                           ║
╠══════════════════════════════════════════════════════════╣
║                                                          ║
║  Consumer calls gpiod_get() / gpio_request()             ║
║      │                                                   ║
║      ▼                                                   ║
║  gpiolib core ──► chip->get() / chip->set()              ║
║               ──► chip->get_direction()                  ║
║               ──► chip->direction_input/output()         ║
║      │                                                   ║
║      ▼                                                   ║
║  gpio_regmap_* functions translate offset → reg+mask     ║
║      │                                                   ║
║      ▼                                                   ║
║  regmap_read() / regmap_write() / regmap_update_bits()   ║
║      │                                                   ║
║      ▼                                                   ║
║  Hardware register (I2C/SPI/MMIO via regmap bus)         ║
║                                                          ║
╠══════════════════════════════════════════════════════════╣
║                    TEARDOWN PHASE                        ║
╠══════════════════════════════════════════════════════════╣
║                                                          ║
║  gpio_regmap_unregister(gpio)                            ║
║   ─OR─ (devm: automatic on device unbind)                ║
║      │                                                   ║
║      ├─[CONFIG_REGMAP_IRQ && irq_chip_data]              ║
║      │   regmap_del_irq_chip(line, irq_chip_data)        ║
║      │                                                   ║
║      ├─ gpiochip_remove(&gpio->gpio_chip)                ║
║      ├─ bitmap_free(gpio->fixed_direction_output)        ║
║      └─ kfree(gpio)                                      ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

---

### مخطط Call Flow — قراءة GPIO (gpio_regmap_get)

```
Consumer: gpiod_get_value(desc)
    │
    ▼
gpiolib core: gpiochip_get_data(chip) → gpio_regmap*
    │
    ▼
gpio_regmap_get(chip, offset)
    │
    ├─ reg_dat_base set?
    │     YES → base = gpio_regmap_addr(reg_dat_base)
    │     NO  → base = gpio_regmap_addr(reg_set_base)
    │
    ▼
reg_mask_xlate(gpio, base, offset, &reg, &mask)
    │
    ├─ default: gpio_regmap_simple_xlate()
    │     line   = offset % ngpio_per_reg
    │     stride = offset / ngpio_per_reg
    │     *reg   = base + stride * reg_stride
    │     *mask  = BIT(line)
    │
    ▼
reg_dat_base == reg_set_base?
    │
    ├─ YES → regmap_read_bypassed(regmap, reg, &val)
    │         (تتجاوز الـ cache عشان القيم الحالية فعلاً)
    │
    └─ NO  → regmap_read(regmap, reg, &val)
                │
                ▼
           regmap bus (I2C/SPI/MMIO)
                │
                ▼
           hardware register read
    │
    ▼
return !!(val & mask)   ← 0 أو 1
```

---

### مخطط Call Flow — كتابة GPIO (gpio_regmap_set)

```
Consumer: gpiod_set_value(desc, val)
    │
    ▼
gpio_regmap_set(chip, offset, val)
    │
    ├─ base = gpio_regmap_addr(reg_set_base)
    │
    ▼
reg_mask_xlate(gpio, base, offset, &reg, &mask)
    │
    ▼
val != 0 ?
    ├─ YES → mask_val = mask
    └─ NO  → mask_val = 0
    │
    ▼
reg_dat_base == reg_set_base?
    ├─ YES → regmap_write_bits(regmap, reg, mask, mask_val)
    │         (بيكتب كل الـ bits دفعة واحدة، مش read-modify-write)
    └─ NO  → regmap_update_bits(regmap, reg, mask, mask_val)
                (read-modify-write)
```

---

### مخطط Call Flow — كتابة مع Set/Clear منفصلين

```
Consumer: gpiod_set_value(desc, val)
    │
    ▼
gpio_regmap_set_with_clear(chip, offset, val)
    │
    ├─ val != 0 → base = reg_set_base → regmap_write(reg, mask)
    │               ← يكتب الـ mask على reg_set → يرفع الـ pin
    │
    └─ val == 0 → base = reg_clr_base → regmap_write(reg, mask)
                    ← يكتب الـ mask على reg_clr → يخفض الـ pin
```

---

### مخطط Call Flow — قراءة/ضبط الاتجاه

```
gpio_regmap_get_direction(chip, offset)
    │
    ├─[fixed_direction_output bitmap set]
    │   test_bit(offset, bitmap) ?
    │     YES → GPIO_LINE_DIRECTION_OUT
    │     NO  → GPIO_LINE_DIRECTION_IN
    │
    ├─[reg_dat_base only, no reg_set_base]
    │   → GPIO_LINE_DIRECTION_IN  (input-only controller)
    │
    ├─[reg_set_base only, no reg_dat_base]
    │   → GPIO_LINE_DIRECTION_OUT (output-only controller)
    │
    ├─[reg_dir_out_base set]
    │   base = reg_dir_out_base, invert = 0
    │
    └─[reg_dir_in_base set]
        base = reg_dir_in_base, invert = 1
    │
    ▼
reg_mask_xlate(gpio, base, offset, &reg, &mask)
    │
    ▼
regmap_read(regmap, reg, &val)
    │
    ▼
!!(val & mask) XOR invert
    ├─ true  → GPIO_LINE_DIRECTION_OUT
    └─ false → GPIO_LINE_DIRECTION_IN

─────────────────────────────────────────────

gpio_regmap_set_direction(chip, offset, output)
    │
    ├─ اختيار base و invert (نفس المنطق فوق)
    │
    ▼
reg_mask_xlate(gpio, base, offset, &reg, &mask)
    │
    ▼
invert ?
    ├─ YES: val = output ? 0 : mask
    │        (reg_dir_in_base: 1=input, 0=output)
    └─ NO:  val = output ? mask : 0
             (reg_dir_out_base: 1=output, 0=input)
    │
    ▼
regmap_update_bits(regmap, reg, mask, val)

─────────────────────────────────────────────

gpio_regmap_direction_output(chip, offset, value)
    │
    ├─ gpio_regmap_set(chip, offset, value)   ← اضبط القيمة الأول
    └─ gpio_regmap_set_direction(chip, offset, true)  ← اتجاه output
```

---

### مخطط Call Flow — التسجيل الكامل

```
devm_gpio_regmap_register(dev, config)
    │
    ▼
gpio_regmap_register(config)
    │
    ├─ validate config
    ├─ kzalloc(gpio_regmap)
    ├─ fill gpio_regmap fields from config
    ├─ fill gpio_chip fields
    ├─ wire ops callbacks
    ├─ gpiochip_get_ngpios() if needed
    ├─ bitmap_alloc + bitmap_copy for fixed_direction_output
    │
    ▼
gpiochip_add_data(chip, gpio)
    │  ← يسجل الـ chip مع gpiolib
    │  ← يحجز GPIO number range
    │
    ▼ [CONFIG_REGMAP_IRQ]
regmap_add_irq_chip_fwnode()
    │  ← ينشئ regmap_irq_chip_data
    │
    ▼
regmap_irq_get_domain(irq_chip_data) → irq_domain
    │
    ▼ [irq_domain present]
gpiochip_irqchip_add_domain(chip, irq_domain)
    │  ← يربط IRQ domain بالـ gpio_chip
    │
    ▼
return gpio_regmap*
    │
    ▼ [devm path only]
devm_add_action_or_reset(dev, devm_gpio_regmap_unregister, gpio)
    ← يسجل cleanup callback تلقائي
```

---

### استراتيجية الـ Locking

الـ `gpio-regmap.c` نفسه **مش بيدير أي lock صراحةً** — ده تصميم مقصود. الـ locking بيتم على مستويين:

#### المستوى الأول: gpiolib core

الـ gpiolib هو اللي بيمسك الـ lock على مستوى الـ `gpio_chip` قبل ما يستدعي أي callback (`get`, `set`, `direction_*`). بيستخدم `srcu` و spinlocks داخلياً في `gpio_device`.

#### المستوى الثاني: regmap

الـ `struct regmap` بيحتوي على الـ lock الخاص بيه (`regmap->lock`، عادةً mutex أو spinlock حسب نوع الـ bus). كل `regmap_read` / `regmap_write` / `regmap_update_bits` بتاخد الـ lock ده داخلياً.

```
┌─────────────────────────────────────────────────────┐
│                  Locking Layers                     │
│                                                     │
│  gpiolib srcu/internal lock                         │
│    └── protects gpio_chip registration/removal      │
│                                                     │
│  regmap->lock (mutex or spinlock)                   │
│    └── protects register read/write/update ops      │
│    └── held during: regmap_read, regmap_write,      │
│                     regmap_update_bits,             │
│                     regmap_write_bits               │
│                                                     │
│  regmap cache                                       │
│    └── regmap_read_bypassed() يتجاوز الـ cache      │
│        لما reg_dat_base == reg_set_base             │
│        (عشان القيم الحالية من الـ HW مش الـ cache) │
└─────────────────────────────────────────────────────┘
```

#### نقطة مهمة: `can_sleep`

```c
chip->can_sleep = regmap_might_sleep(config->regmap);
```

لو الـ regmap محتاج sleep (I2C/SPI)، بتتعلّم gpiolib إن الـ ops دي ممكن تنام — وبالتالي **لازم تتستدعي من context تنام فيه** (مش في interrupt handler مثلاً). ده بيأثر على الـ IRQ handling اللي لازم يبقى threaded.

#### ترتيب الـ Locking (Lock Ordering)

```
gpiolib_lock
    └── regmap_lock
            └── bus_lock (i2c_adapter_lock / spi_controller_lock)
```

**مفيش lock عكسي** في الكود ده — الكود بسيط وخطي بدون deadlock risks.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### الـ Public API (exported)

| Function | Signature (مختصرة) | الغرض |
|---|---|---|
| `gpio_regmap_register` | `struct gpio_regmap *f(config)` | تسجيل GPIO controller من regmap config |
| `gpio_regmap_unregister` | `void f(gpio)` | إلغاء تسجيل الـ controller وتحرير الموارد |
| `devm_gpio_regmap_register` | `struct gpio_regmap *f(dev, config)` | نسخة devm تتنظف أوتوماتيك عند driver detach |
| `gpio_regmap_get_drvdata` | `void *f(gpio)` | استرجاع الـ driver-private data |

#### الـ Internal (static) Functions

| Function | الغرض |
|---|---|
| `gpio_regmap_addr` | تحويل الـ `GPIO_REGMAP_ADDR_ZERO` sentinel لعنوان 0 حقيقي |
| `gpio_regmap_simple_xlate` | الـ default translation من offset لـ reg + mask |
| `gpio_regmap_get` | قراءة قيمة GPIO line |
| `gpio_regmap_set` | كتابة قيمة GPIO بـ read-modify-write |
| `gpio_regmap_set_with_clear` | كتابة قيمة GPIO بـ dedicated set/clear registers |
| `gpio_regmap_get_direction` | قراءة اتجاه الـ pin (input/output) |
| `gpio_regmap_set_direction` | ضبط اتجاه الـ pin |
| `gpio_regmap_direction_input` | wrapper: اتجاه input |
| `gpio_regmap_direction_output` | wrapper: اتجاه output مع set قيمة أولية |
| `devm_gpio_regmap_unregister` | devm action callback للـ cleanup |

---

### المجموعة الأولى: الـ Address & Translation Helpers

دي الـ functions المسؤولة عن تحويل الـ GPIO logical offset لـ register address و bitmask فعلي. الـ regmap GPIO driver بيتعامل مع hardware فيه registers متعددة، كل register ممكن يحتوي على أكتر من GPIO line، فالـ translation هي القلب اللي يربط الـ abstract offset بالـ physical register.

---

#### `gpio_regmap_addr`

```c
static unsigned int gpio_regmap_addr(unsigned int addr)
```

**ما يعمله:** بتحول الـ sentinel value الخاصة بالـ address zero. الـ kernel بيستخدم `0` كـ "not set"، فـ hardware اللي عنوانه فعلاً `0` بيستخدم `GPIO_REGMAP_ADDR_ZERO` (== `UINT_MAX`) كـ workaround، والـ function دي بتحوله للـ `0` الحقيقي.

**Parameters:**
- `addr`: العنوان اللي ممكن يكون `GPIO_REGMAP_ADDR_ZERO` أو عنوان عادي.

**Return value:** `0` لو `addr == GPIO_REGMAP_ADDR_ZERO`، وإلا يرجع `addr` كما هو.

**Key details:** بيتكلم عن مشكلة تصميمية — إزاي تميز بين "العنوان مش محدد" و"العنوان = 0 فعلاً". الـ macro `GPIO_REGMAP_ADDR(x)` في الـ header بيستخدم نفس الفكرة.

---

#### `gpio_regmap_simple_xlate`

```c
static int gpio_regmap_simple_xlate(struct gpio_regmap *gpio,
                                    unsigned int base, unsigned int offset,
                                    unsigned int *reg, unsigned int *mask)
```

**ما يعمله:** الـ default implementation للـ `reg_mask_xlate` callback. بتافترض إن الـ GPIOs موزعة بالترتيب: أول `ngpio_per_reg` GPIO في الـ register الأول، التالية في الـ register التاني بـ stride ثابتة.

```
offset=5, ngpio_per_reg=8, base=0x10, reg_stride=4:
  line   = 5 % 8 = 5
  stride = 5 / 8 = 0
  reg    = 0x10 + 0 * 4 = 0x10
  mask   = BIT(5) = 0x20
```

**Parameters:**
- `gpio`: الـ `gpio_regmap` instance للوصول لـ `ngpio_per_reg` و `reg_stride`.
- `base`: الـ base address للـ register group (dat, set, clr, dir).
- `offset`: الـ GPIO logical offset (0..ngpio-1).
- `reg`: [out] عنوان الـ register المستهدف.
- `mask`: [out] الـ bitmask للـ GPIO داخل الـ register.

**Return value:** دايماً `0` (success). الـ custom xlate ممكن ترجع error.

**Key details:** الـ drivers اللي عندها non-linear mapping (زي GPIOs موزعة على banks غير منتظمة) بتحط `reg_mask_xlate` custom في الـ config بدلاً من الاعتماد على الـ default.

---

### المجموعة التانية: الـ GPIO Value Operations (Runtime I/O)

الـ functions دي بتنفذ الـ `gpio_chip` operations الخاصة بقراءة وكتابة قيم الـ GPIO lines.

---

#### `gpio_regmap_get`

```c
static int gpio_regmap_get(struct gpio_chip *chip, unsigned int offset)
```

**ما يعمله:** بتقرا القيمة الحالية للـ GPIO line. لو عندنا `reg_dat_base` (input register)، بتقرا منه. لو مفيش ومتاح بس `reg_set_base`، بتقرا منه (output-only hardware). في الحالة الخاصة اللي فيها `reg_dat_base == reg_set_base` (نفس الـ register للـ input والـ output)، بتستخدم `regmap_read_bypassed` عشان ما تلوطش الـ register cache بقيم الـ input.

**Parameters:**
- `chip`: الـ `gpio_chip` المضمّن في `gpio_regmap`.
- `offset`: الـ GPIO line number (0..ngpio-1).

**Return value:** `0` أو `1` (قيمة الـ line)، أو قيمة سالبة عند error من الـ xlate أو الـ regmap read.

**Key details:**
- الـ `regmap_read_bypassed` بيتجاوز الـ cache وبيقرا من الـ hardware مباشرة، ده مهم جداً عشان الـ input value ما تاثرش على الـ cached output value.
- لو `reg_dat_base != reg_set_base`، بيستخدم `regmap_read` العادي (بيستفيد من الـ cache).

**Pseudocode:**
```
gpio_regmap_get:
  if reg_dat_base:
      base = gpio_regmap_addr(reg_dat_base)
  else:
      base = gpio_regmap_addr(reg_set_base)

  xlate(base, offset) -> reg, mask

  if reg_dat_base == reg_set_base:
      regmap_read_bypassed(reg) -> val   // bypass cache
  else:
      regmap_read(reg) -> val

  return !!(val & mask)
```

---

#### `gpio_regmap_set`

```c
static int gpio_regmap_set(struct gpio_chip *chip, unsigned int offset, int val)
```

**ما يعمله:** بتكتب قيمة على GPIO line باستخدام الـ `reg_set_base`. لو `reg_dat_base == reg_set_base`، بتستخدم `regmap_write_bits` عشان تكتب فقط الـ bit المطلوب بدون read-modify-write كامل (الـ regmap cache بيوفر الـ shadow). لو مختلفين، بتستخدم `regmap_update_bits` اللي بتعمل read-modify-write.

**Parameters:**
- `chip`: الـ gpio_chip.
- `offset`: الـ GPIO line.
- `val`: القيمة (`0` أو `!=0`).

**Return value:** `0` عند نجاح، قيمة سالبة عند خطأ.

**Key details:**
- `regmap_write_bits(reg, mask, val)`: بيكتب بـ mask مباشرة من الـ cache بدون hardware read، مناسب لما الـ register نفسه بيمثل قيم الـ output فقط.
- `regmap_update_bits(reg, mask, val)`: بيعمل read (من cache أو hardware) ثم modify ثم write.
- الـ `mask_val` بيتحسب: لو `val != 0` → `mask_val = mask`، لو `val == 0` → `mask_val = 0`.

---

#### `gpio_regmap_set_with_clear`

```c
static int gpio_regmap_set_with_clear(struct gpio_chip *chip,
                                      unsigned int offset, int val)
```

**ما يعمله:** بتكتب قيمة على GPIO لما الـ hardware عنده registers منفصلة للـ set والـ clear (زي ARM PL061 أو GPIOs على بعض MFD chips). لو `val != 0`: بتكتب الـ mask على `reg_set_base`. لو `val == 0`: بتكتب الـ mask على `reg_clr_base`.

**Parameters:**
- `chip`, `offset`, `val`: نفس الشرح فوق.

**Return value:** نتيجة `regmap_write`.

**Key details:**
- بتستخدم `regmap_write` مش `regmap_update_bits` لأن الكتابة لـ set/clear registers بـ "1" في الـ bit المطلوب كافية — الـ hardware نفسه بيفهم الـ semantics.
- ده أكفأ من الـ read-modify-write لأنه atomic على مستوى الـ register.
- الـ `chip->set` بيتضبط على الـ function دي في `gpio_regmap_register` فقط لو `reg_clr_base` محدد.

---

### المجموعة التالتة: الـ Direction Operations

بتتحكم في اتجاه الـ GPIO lines (input/output). الـ hardware ممكن يبقى input-only، output-only، أو bidirectional. الـ driver بيدعم الثلاث حالات.

---

#### `gpio_regmap_get_direction`

```c
static int gpio_regmap_get_direction(struct gpio_chip *chip, unsigned int offset)
```

**ما يعمله:** بتحدد اتجاه الـ GPIO line بالطريقة الأنسب حسب الـ config:

1. لو `fixed_direction_output` bitmap موجود: بتتحقق من الـ bit المقابل.
2. لو بس `reg_dat_base` (بدون `reg_set_base`): input-only → ترجع `GPIO_LINE_DIRECTION_IN`.
3. لو بس `reg_set_base` (بدون `reg_dat_base`): output-only → ترجع `GPIO_LINE_DIRECTION_OUT`.
4. لو `reg_dir_out_base` أو `reg_dir_in_base` موجود: بتقرا من الـ register وبتعمل XOR مع الـ `invert` flag.

**Parameters:**
- `chip`, `offset`.

**Return value:** `GPIO_LINE_DIRECTION_IN` (1)، `GPIO_LINE_DIRECTION_OUT` (0)، أو قيمة سالبة (error أو `-ENOTSUPP`).

**Key details:**
- الـ `invert` flag بيتعامل مع فرق semantics الـ hardware: بعض الـ chips رجستر الـ direction بيحط `1` = output (`reg_dir_out_base`)، وبعضها `1` = input (`reg_dir_in_base`).
- `XOR` الـ `!!(val & mask) ^ invert` بيوحّد الـ two conventions.

**Pseudocode:**
```
gpio_regmap_get_direction:
  if fixed_direction_output:
      return test_bit(offset) ? OUT : IN

  if only reg_dat_base:  return IN
  if only reg_set_base:  return OUT

  if reg_dir_out_base:
      base = reg_dir_out_base, invert = 0
  elif reg_dir_in_base:
      base = reg_dir_in_base,  invert = 1
  else:
      return -ENOTSUPP

  xlate(base, offset) -> reg, mask
  regmap_read(reg) -> val

  return (!!(val & mask) ^ invert) ? OUT : IN
```

---

#### `gpio_regmap_set_direction`

```c
static int gpio_regmap_set_direction(struct gpio_chip *chip,
                                     unsigned int offset, bool output)
```

**ما يعمله:** بتكتب في الـ direction register عشان تغير اتجاه الـ GPIO. بتتعامل مع الـ `invert` logic بنفس أسلوب `gpio_regmap_get_direction`.

**Parameters:**
- `chip`, `offset`.
- `output`: `true` = output، `false` = input.

**Return value:** نتيجة `regmap_update_bits`.

**Key details:**
- لو `invert == 1` (reg_dir_in_base): `output=true` → كتابة `0` في الـ bit (لأن `1` = input في الـ hardware ده).
- لو `invert == 0` (reg_dir_out_base): `output=true` → كتابة `mask` في الـ bit.
- بتستخدم `regmap_update_bits` عشان ما تأثرش على الـ GPIO lines التانية في نفس الـ register.
- لو مفيش direction registers، ترجع `-ENOTSUPP`.

---

#### `gpio_regmap_direction_input`

```c
static int gpio_regmap_direction_input(struct gpio_chip *chip,
                                       unsigned int offset)
```

**ما يعمله:** wrapper بسيط بيستدعي `gpio_regmap_set_direction(chip, offset, false)`.

**Who calls it:** الـ `gpio_chip->direction_input` callback، بيتستدعى من gpiolib عند `gpiod_direction_input()`.

---

#### `gpio_regmap_direction_output`

```c
static int gpio_regmap_direction_output(struct gpio_chip *chip,
                                        unsigned int offset, int value)
```

**ما يعمله:** بتضبط الـ GPIO كـ output وبتحط القيمة الأولية قبل التحويل. ترتيب الخطوات مهم: أول set القيمة (وهو لسه input) عشان نتجنب glitch على الـ line، بعدين change direction.

**Parameters:**
- `chip`, `offset`.
- `value`: القيمة الأولية للـ output.

**Return value:** نتيجة `gpio_regmap_set_direction`.

**Key details:** الـ set ثم direction pattern ده standard في GPIO drivers لتجنب الـ glitch — لو غيّرت الـ direction الأول وبعدين set القيمة، الـ line هتبقى undefined لحظة.

**Who calls it:** `gpio_chip->direction_output` callback من gpiolib.

---

### المجموعة الرابعة: الـ Registration & Lifecycle

الـ functions دي هي الـ public API الرئيسي للـ driver ومسؤولة عن إنشاء وتدمير الـ GPIO controller.

---

#### `gpio_regmap_register`

```c
struct gpio_regmap *gpio_regmap_register(const struct gpio_regmap_config *config)
```

**ما يعمله:** بتسجل GPIO controller كامل من config struct. بتعمل validation، allocation، ضبط الـ gpio_chip callbacks، وتسجيل الـ gpiochip مع الـ gpiolib. لو محدد، بتضيف الـ IRQ domain.

**Parameters:**
- `config`: الـ `gpio_regmap_config` struct اللي فيه كل الـ hardware description.

**Return value:** pointer لـ `gpio_regmap` عند نجاح، أو `ERR_PTR` عند فشل (`-EINVAL`, `-ENOMEM`, وغيرهم).

**Key details — Validation:**
- لازم `config->parent` موجود.
- لازم واحد على الأقل من `reg_dat_base` أو `reg_set_base`.
- لو محدد direction register، لازم الاتنين `reg_dat_base` و `reg_set_base` موجودين.
- `reg_dir_out_base` و `reg_dir_in_base` exclusive — مش ينفع الاتنين مع بعض.

**Key details — gpio_chip setup:**
- `chip->base = -1`: dynamic GPIO number allocation (مش static).
- `chip->can_sleep = regmap_might_sleep(regmap)`: ده مهم جداً للـ I2C/SPI expanders اللي بتحتاج sleep في الـ access.
- الـ `chip->set` callback بيتختار بناءً على الـ config: لو في `reg_clr_base` → `gpio_regmap_set_with_clear`، وإلا → `gpio_regmap_set`.
- `direction_input` و `direction_output` بيتضافوا بس لو في direction registers.

**Key details — ngpio handling:**
- لو `config->ngpio == 0`، بتقرا العدد من الـ firmware (DT/ACPI) عبر `gpiochip_get_ngpios`.

**Key details — IRQ:**
- لو `CONFIG_REGMAP_IRQ` وفيه `regmap_irq_chip`: بتسجل `regmap_irq` chip وتجيب الـ domain منه.
- لو مفيش `regmap_irq_chip` بس فيه `irq_domain`: بتستخدمه مباشرة.
- في الحالتين، بتسجل الـ domain على الـ gpiochip عبر `gpiochip_irqchip_add_domain`.

**Error path:**
```
err_remove_gpiochip  → gpiochip_remove
err_free_bitmap      → bitmap_free(fixed_direction_output)
err_free_gpio        → kfree(gpio)
```

**Pseudocode:**
```
gpio_regmap_register(config):
  validate config -> ERR_PTR(-EINVAL) if invalid

  gpio = kzalloc(sizeof(*gpio))

  // copy config fields to gpio struct
  gpio->parent, regmap, reg_*_base, ...

  // setup gpio_chip
  chip->base = -1  // dynamic
  chip->can_sleep = regmap_might_sleep()
  chip->get  = gpio_regmap_get
  chip->set  = (reg_clr_base) ? set_with_clear : set
  chip->get_direction = gpio_regmap_get_direction
  if dir registers:
      chip->direction_input/output = ...

  // resolve ngpio
  if !config->ngpio:
      gpiochip_get_ngpios(chip)

  // copy fixed direction bitmap if provided
  if config->fixed_direction_output:
      bitmap_alloc + bitmap_copy

  // defaults
  ngpio_per_reg = config->ngpio_per_reg ?: config->ngpio
  reg_stride    = config->reg_stride    ?: 1
  reg_mask_xlate = config->reg_mask_xlate ?: gpio_regmap_simple_xlate

  gpiochip_add_data(chip, gpio) -> err_free_bitmap on fail

  // IRQ setup
  if regmap_irq_chip:
      regmap_add_irq_chip_fwnode(...) -> err_remove_gpiochip on fail
      irq_domain = regmap_irq_get_domain(...)
  else:
      irq_domain = config->irq_domain

  if irq_domain:
      gpiochip_irqchip_add_domain(chip, irq_domain)

  return gpio
```

---

#### `gpio_regmap_unregister`

```c
void gpio_regmap_unregister(struct gpio_regmap *gpio)
```

**ما يعمله:** بتعمل teardown كامل للـ GPIO controller بالترتيب الصح: أول بتشيل الـ IRQ chip (لو موجود)، بعدين بتشيل الـ gpiochip من الـ gpiolib، بعدين بتحرر الـ bitmap، وأخيراً بتحرر الـ gpio struct نفسه.

**Parameters:**
- `gpio`: الـ `gpio_regmap` instance اللي تم إنشاؤه بـ `gpio_regmap_register`.

**Return value:** لا يرجع قيمة (`void`).

**Key details:**
- الترتيب مهم جداً: لازم `regmap_del_irq_chip` قبل `gpiochip_remove` عشان ما يبقاش في IRQ handlers شاغلين الـ GPIO.
- `bitmap_free` على `NULL` آمن (no-op)، فمش محتاج null check.
- بعد `kfree(gpio)` مباشرة، الـ pointer invalid.

---

#### `devm_gpio_regmap_unregister`

```c
static void devm_gpio_regmap_unregister(void *res)
```

**ما يعمله:** الـ devm action callback. بتستدعي `gpio_regmap_unregister` على الـ resource pointer.

**Parameters:**
- `res`: void pointer للـ `gpio_regmap` instance.

**Key details:** بيتسجل عبر `devm_add_action_or_reset` ومش بيتستدعى مباشرة. الـ devres infrastructure بتستدعيه عند `device_unregister` أو driver detach.

---

#### `devm_gpio_regmap_register`

```c
struct gpio_regmap *devm_gpio_regmap_register(struct device *dev,
                                              const struct gpio_regmap_config *config)
```

**ما يعمله:** نسخة resource-managed من `gpio_regmap_register`. بتسجل الـ GPIO controller وبتضيف cleanup action على الـ device lifecycle. لما الـ device بتتنظف، الـ `gpio_regmap_unregister` بيتستدعى أوتوماتيك.

**Parameters:**
- `dev`: الـ device اللي عليه بيتسجل الـ cleanup action (ممكن يختلف عن `config->parent` في نظرية، بس عادةً نفسه).
- `config`: نفس الـ config struct.

**Return value:** نفس `gpio_regmap_register` — pointer أو `ERR_PTR`.

**Key details:**
- `devm_add_action_or_reset`: لو فشل الـ add action نفسه، بيستدعي الـ cleanup فوراً ويرجع error. ده بيضمن مفيش resource leak.
- الـ pattern ده standard في kernel drivers الحديثة عشان تبسيط الـ error paths في الـ probe function.

**Pseudocode:**
```
devm_gpio_regmap_register(dev, config):
  gpio = gpio_regmap_register(config)
  if IS_ERR(gpio):
      return gpio  // propagate error

  ret = devm_add_action_or_reset(dev, devm_gpio_regmap_unregister, gpio)
  if ret:
      // gpio already unregistered by devm_add_action_or_reset
      return ERR_PTR(ret)

  return gpio
```

---

#### `gpio_regmap_get_drvdata`

```c
void *gpio_regmap_get_drvdata(struct gpio_regmap *gpio)
```

**ما يعمله:** بترجع الـ `driver_data` pointer اللي الـ driver حطه في `config->drvdata` عند التسجيل. بتستخدم في الـ `reg_mask_xlate` callback عشان الـ driver يوصل لـ private state خاصة بيه.

**Parameters:**
- `gpio`: الـ `gpio_regmap` instance.

**Return value:** الـ `void *` driver-private pointer (ممكن يكون `NULL`).

**Key details:**
- الـ `gpio_regmap` struct هو opaque للـ caller، الـ field الوحيد اللي متاح عبر API هو الـ drvdata.
- الـ function دي `EXPORT_SYMBOL_GPL` — يعني متاحة للـ kernel modules.

---

### ملخص الـ Callback Assignment في `gpio_regmap_register`

```
┌─────────────────────────┬──────────────────────────────────────────────┐
│ gpio_chip callback      │ يتضبط على                                    │
├─────────────────────────┼──────────────────────────────────────────────┤
│ request                 │ gpiochip_generic_request (دايماً)             │
│ free                    │ gpiochip_generic_free    (دايماً)             │
│ get                     │ gpio_regmap_get          (دايماً)             │
│ set                     │ gpio_regmap_set_with_clear  (لو reg_clr_base) │
│                         │ gpio_regmap_set             (لو بس reg_set)   │
│                         │ NULL                        (لو input-only)   │
│ get_direction           │ gpio_regmap_get_direction   (دايماً)          │
│ direction_input         │ gpio_regmap_direction_input (لو dir reg)      │
│ direction_output        │ gpio_regmap_direction_output (لو dir reg)     │
└─────────────────────────┴──────────────────────────────────────────────┘
```

### الـ `fixed_direction_output` Bitmap

الـ bitmap ده بيستخدمه الـ drivers اللي عندها GPIO lines اتجاهها fixed (مش قابل للتغيير) لكنها موجودة في نفس الـ register مع lines قابلة للتغيير. بدل ما الـ driver يحط direction register، بيحط bitmap كل bit فيه يمثل GPIO line واحدة:
- `1` = output فيكسد
- `0` = input فيكسد

الـ `gpio_regmap_get_direction` بتتحقق منه أول حاجة قبل ما تتكلم مع الـ hardware.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs — المسارات والقراءة

**الـ GPIO subsystem** بيعرض معلومات مفيدة جداً في الـ debugfs:

```bash
# mount debugfs لو مش موجود
mount -t debugfs none /sys/kernel/debug

# اعرض كل الـ GPIO chips المسجّلة مع state كل line
cat /sys/kernel/debug/gpio
```

**مثال على الـ output:**

```
gpiochip5: GPIOs 496-503, parent: i2c/1-0020, pca9557, can sleep:
 gpio-496 (                    ) in  lo IRQ
 gpio-497 (reset               ) out hi
 gpio-498 (                    ) in  hi IRQ
```

**التفسير:**
- `in`/`out` → الـ direction الحالية كما تُحدَّد من `gpio_regmap_get_direction()`
- `hi`/`lo` → القيمة المقروءة من `gpio_regmap_get()`
- `IRQ` → الـ line مربوطة بـ interrupt domain

**الـ regmap** كمان عنده debugfs خاص بيه:

```bash
# اعرض كل الـ regmap instances
ls /sys/kernel/debug/regmap/

# اعرض registers الـ device (مثلاً I2C device على address 0x20)
cat /sys/kernel/debug/regmap/1-0020/registers

# اعرض حالة الـ cache
cat /sys/kernel/debug/regmap/1-0020/cache_bypass
cat /sys/kernel/debug/regmap/1-0020/cache_dirty
cat /sys/kernel/debug/regmap/1-0020/cache_only
```

**مثال output لـ `registers`:**

```
00: ff
01: 00
02: ff
03: f0
```

ده الـ `reg_dat_base` و`reg_set_base` و`reg_dir_out_base` كما خُزّنت في الـ regmap cache.

---

#### 2. sysfs — المسارات المهمة

```bash
# اعرض كل الـ gpiochips
ls /sys/class/gpio/

# export GPIO معين (رقم 497) عشان تتحكم فيه من user space
echo 497 > /sys/class/gpio/export

# اقرأ الـ direction
cat /sys/class/gpio/gpio497/direction    # "in" أو "out"

# اقرأ الـ value
cat /sys/class/gpio/gpio497/value        # "0" أو "1"

# اكتب output value
echo 1 > /sys/class/gpio/gpio497/value

# اعرض معلومات الـ chip
cat /sys/class/gpio/gpiochip496/label    # chip->label من gpio_regmap_register()
cat /sys/class/gpio/gpiochip496/ngpio    # عدد الـ GPIOs
cat /sys/class/gpio/gpiochip496/base     # أول GPIO رقم

# اعرض الـ IRQ المرتبطة بالـ chip
cat /proc/interrupts | grep gpio
```

---

#### 3. ftrace — الـ Tracepoints والـ Events

**تفعيل الـ GPIO tracepoints:**

```bash
# اعرض الـ events المتاحة
ls /sys/kernel/debug/tracing/events/gpio/

# فعّل كل GPIO events
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable

# أو فعّل events محددة
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_value/enable
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_direction/enable

# فعّل الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# نفّذ العملية
gpioget gpiochip5 3

# اقرأ النتائج
cat /sys/kernel/debug/tracing/trace
```

**مثال output:**

```
gpioget-1234  [001] ....  123.456789: gpio_direction: gpio=499 in
gpioget-1234  [001] ....  123.456790: gpio_value: gpio=499 get=1
```

**تتبع دوال الـ gpio-regmap مع function_graph:**

```bash
# تتبع الدوال الداخلية للـ driver
cat > /sys/kernel/debug/tracing/set_ftrace_filter << 'EOF'
gpio_regmap_get
gpio_regmap_set
gpio_regmap_set_with_clear
gpio_regmap_get_direction
gpio_regmap_set_direction
regmap_read
regmap_write
regmap_update_bits
regmap_read_bypassed
EOF

echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# نفّذ الاختبار
gpioget gpiochip5 0

echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

**مثال output يوضح الـ cache bypass:**

```
 1)               |  gpio_regmap_get() {
 1)               |    gpio_regmap_simple_xlate() {
 1)   0.150 us    |    }
 1)               |    regmap_read_bypassed() {   /* reg_dat_base == reg_set_base */
 1)               |      i2c_smbus_read_byte_data() {
 1)   1.230 us    |      }
 1)   1.890 us    |    }
 1)   2.350 us    |  }
```

لما تشوف `regmap_read_bypassed` مش `regmap_read` — ده يأكد إن `reg_dat_base == reg_set_base` والـ driver بيتجنب تلويث الـ cache بقيم الـ input.

---

#### 4. printk / Dynamic Debug

```bash
# فعّل dynamic debug لملف gpio-regmap.c تحديداً
echo "file gpio-regmap.c +pflmt" > /sys/kernel/debug/dynamic_debug/control

# فعّل للـ module كله
echo "module gpio_regmap +pflm" > /sys/kernel/debug/dynamic_debug/control

# فعّل للـ regmap subsystem
echo "module regmap +p" > /sys/kernel/debug/dynamic_debug/control

# اتحقق من الـ control entries الحالية
cat /sys/kernel/debug/dynamic_debug/control | grep gpio-regmap

# اتابع الـ log
dmesg -w | grep -E "gpio|regmap"
```

**الـ flags المتاحة في `+pflmt`:**
- `p` → تفعيل الـ printk
- `f` → اضافة اسم الـ function
- `l` → اضافة رقم الـ line
- `m` → اضافة اسم الـ module
- `t` → اضافة الـ thread id

**تفعيل من الـ kernel cmdline:**

```
dyndbg="module gpio_regmap +p; module regmap +p"
```

**نقاط مناسبة لإضافة `dev_dbg` مؤقتاً في الـ driver:**

```c
static int gpio_regmap_get(struct gpio_chip *chip, unsigned int offset)
{
    struct gpio_regmap *gpio = gpiochip_get_data(chip);
    unsigned int base, val, reg, mask;
    int ret;

    /* ... */
    ret = regmap_read(gpio->regmap, reg, &val);

    /* debug point: اطبع القيمة المقروءة من الـ register */
    dev_dbg(gpio->parent,
            "%s: offset=%u reg=0x%x mask=0x%x val=0x%x ret=%d\n",
            __func__, offset, reg, mask, val, ret);

    return !!(val & mask);
}
```

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الوصف | التأثير |
|---|---|---|
| `CONFIG_GPIOLIB` | الـ GPIO core | dependency أساسية |
| `CONFIG_GPIO_REGMAP` | الـ driver نفسه | لازم enabled |
| `CONFIG_REGMAP` | الـ regmap core | dependency أساسية |
| `CONFIG_REGMAP_IRQ` | دعم الـ IRQ في regmap | للـ `regmap_irq_chip` support |
| `CONFIG_REGMAP_DEBUGFS` | يفعّل `/sys/kernel/debug/regmap/` | مهم جداً للـ debugging |
| `CONFIG_DEBUG_GPIO` | extra validation وlogs في gpiolib | يضيف checks في `gpiochip_add_data` |
| `CONFIG_GPIO_SYSFS` | الـ sysfs interface | للـ export/unexport من user space |
| `CONFIG_DYNAMIC_DEBUG` | الـ dynamic debug framework | يفعّل `dyndbg=` |
| `CONFIG_GPIOLIB_IRQCHIP` | الـ IRQ chip integration | للـ IRQ debugging |
| `CONFIG_LOCKDEP` | كشف deadlocks | مهم في الـ IRQ context |
| `CONFIG_PROVE_LOCKING` | تحقق صارم من الـ locking | يكشف race conditions |
| `CONFIG_KASAN` | كشف memory bugs | يكشف buffer overflows في `bitmap_alloc` |
| `CONFIG_TRACING` | يفعّل ftrace | للـ function_graph tracing |

```bash
# تأكد من الـ configs المفعّلة
zcat /proc/config.gz | grep -E "CONFIG_(GPIO|REGMAP|DEBUG_GPIO|LOCKDEP)"
```

---

#### 6. أدوات خاصة بالـ Subsystem

**الـ libgpiod tools:**

```bash
# اعرض كل الـ chips والـ lines
gpiodetect

# اعرض chip معين بالتفصيل
gpioinfo gpiochip5

# اقرأ قيمة GPIO
gpioget gpiochip5 0

# اكتب قيمة
gpioset gpiochip5 1=1

# راقب تغيرات الـ GPIO
gpiomon --num-events=10 --rising-edge --falling-edge gpiochip5 2
```

**مثال output لـ `gpioinfo`:**

```
gpiochip5 - 8 lines:
        line  0:      unnamed       unused   input  active-high
        line  1:      "reset"       "reset"  output active-high [used]
        line  2:      "INT"         "irq"    input  active-low  [used]
```

**الـ i2ctools للـ I2C-based regmap:**

```bash
# اعرض devices على الـ I2C bus
i2cdetect -y 1

# اقرأ register مباشرة من الـ hardware
i2cget -y 1 0x20 0x00    # reg_dat_base

# dump كل الـ registers
i2cdump -y 1 0x20

# اكتب register
i2cset -y 1 0x20 0x03 0xFF   # reg_dir_out_base
```

**الـ spidev_test للـ SPI-based regmap:**

```bash
# اختبر الـ SPI communication
spidev_test -D /dev/spidev0.0 -v -p "\x03\x00"
```

---

#### 7. جدول رسائل الـ Errors الشائعة

| رسالة الـ Error | المعنى | الحل |
|---|---|---|
| `gpio_regmap_register: -EINVAL` (no parent) | `config->parent` = NULL | تأكد إن `config.parent = dev` متضبطة |
| `gpio_regmap_register: -EINVAL` (no data/set) | مفيش `reg_dat_base` ولا `reg_set_base` | لازم تحدد واحدة على الأقل |
| `gpio_regmap_register: -EINVAL` (dir without dat+set) | ضبطت `reg_dir_*` بدون الاتنين `dat` و`set` | لازم تحدد `reg_dat_base` و`reg_set_base` مع direction registers |
| `gpio_regmap_register: -EINVAL` (both dir bases) | ضبطت `reg_dir_out_base` و`reg_dir_in_base` مع بعض | استخدم واحدة بس — هما exclusive |
| `gpiochip_add_data: -EBUSY` | الـ GPIO range متضارب مع chip تاني | تأكد من عدم تكرار الـ base address |
| `get_direction: -ENOTSUPP` | مفيش direction registers | استخدم `fixed_direction_output` bitmap |
| `regmap_read: -EIO` | فشل قراءة الـ hardware | افحص الـ I2C/SPI bus والـ power |
| `regmap_update_bits: -ETIMEDOUT` | timeout في الـ bus | افحص الـ clock speed والـ pull-up resistors |
| `gpiochip_irqchip_add_domain: -EINVAL` | الـ IRQ domain مش متوافق | تأكد من `CONFIG_GPIOLIB_IRQCHIP` |
| `regmap_add_irq_chip_fwnode: -ENOMEM` | مفيش ذاكرة | قلّل عدد الـ IRQs أو افحص الـ GFP flags |
| `bitmap_alloc failed` | مفيش ذاكرة لـ `fixed_direction_output` | `ngpio` كبير جداً أو system memory منخفض |
| `gpiochip_get_ngpios failed` | مش قادر يحدد عدد الـ GPIOs من الـ fwnode | حدد `ngpio` في الـ config صراحةً |
| `BUG: scheduling while atomic` | الـ chip على I2C/SPI لكن استُدعي من atomic context | `can_sleep` = true → الـ regmap عارف إن الـ bus بيحتاج sleep |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في gpio_regmap_get() — تحقق من الـ offset */
static int gpio_regmap_get(struct gpio_chip *chip, unsigned int offset)
{
    struct gpio_regmap *gpio = gpiochip_get_data(chip);

    /* offset لازم يكون في الحدود */
    if (WARN_ON(offset >= chip->ngpio))
        return -EINVAL;

    /* الـ regmap لازم يكون موجود */
    if (WARN_ON(!gpio->regmap)) {
        dump_stack();
        return -EINVAL;
    }
    ...
}

/* في gpio_regmap_set_direction() — تحقق من وجود direction registers */
static int gpio_regmap_set_direction(struct gpio_chip *chip,
                                     unsigned int offset, bool output)
{
    struct gpio_regmap *gpio = gpiochip_get_data(chip);

    /*
     * ده مش المفروض يحصل لأن gpio_regmap_register()
     * مش بتسجل direction_input لو مفيش direction registers
     */
    if (WARN_ONCE(!gpio->reg_dir_in_base && !gpio->reg_dir_out_base,
                  "direction_input called but no direction register configured\n")) {
        dump_stack();
        return -ENOTSUPP;
    }
    ...
}

/* في gpio_regmap_register() — تحقق من الـ xlate function */
struct gpio_regmap *gpio_regmap_register(const struct gpio_regmap_config *config)
{
    /* WARN لو ngpio_per_reg أكبر من BITS_PER_LONG */
    WARN_ON(config->ngpio_per_reg > BITS_PER_LONG);
    ...
}
```

**نقاط مناسبة لـ `dump_stack()`:**
- بعد `regmap_read` / `regmap_write` اللي بترجع error غير متوقع في production
- في `devm_gpio_regmap_unregister` لو الـ pointer = NULL
- في `gpio_regmap_simple_xlate` لو الـ `reg` حيجي out of range

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State بيطابق الـ Kernel State

```bash
# 1. اقرأ قيمة GPIO من الـ kernel
gpioget gpiochip5 0

# 2. اقرأ نفس الـ register مباشرة من الـ hardware عبر I2C
# مثلاً: reg_dat_base = 0x00 في PCA9557
i2cget -y 1 0x20 0x00

# لو القيمتين مختلفتين — فيه مشكلة في الـ cache
# اتحقق من حالة الـ cache
cat /sys/kernel/debug/regmap/1-0020/cache_bypass
cat /sys/kernel/debug/regmap/1-0020/cache_dirty
```

**فهم الـ cache bypass في الـ driver:**

```c
/* من gpio_regmap_get() في gpio-regmap.c */
if (gpio->reg_dat_base == gpio->reg_set_base)
    /* bypass الـ cache لأن نفس الـ register بيُستخدم للـ input والـ output */
    ret = regmap_read_bypassed(gpio->regmap, reg, &val);
else
    /* استخدم الـ cache بشكل طبيعي */
    ret = regmap_read(gpio->regmap, reg, &val);
```

لو `reg_dat_base == reg_set_base`، الـ cache ممكن يرجع القيمة المكتوبة (output) مش القيمة الحقيقية (input). الـ `regmap_read_bypassed` بيحل المشكلة دي.

```bash
# اختبر الـ cache bypass manually
# فعّل cache bypass كله
echo 1 > /sys/kernel/debug/regmap/1-0020/cache_bypass
gpioget gpiochip5 0   # هيقرأ مباشرة من الـ hardware
echo 0 > /sys/kernel/debug/regmap/1-0020/cache_bypass
```

---

#### 2. Register Dump — قراءة الـ Registers مباشرة

**عبر الـ regmap debugfs (الأفضل):**

```bash
# اعرض كل الـ registers وقيمها الحالية في الـ cache
cat /sys/kernel/debug/regmap/1-0020/registers

# مثال output لـ PCA9557 (8-bit I2C GPIO expander):
# 00: ff   <- Input Port (reg_dat_base)
# 01: 00   <- Output Port (reg_set_base)
# 02: ff   <- Polarity Inversion
# 03: f0   <- Configuration (reg_dir_out_base) — lower 4 bits = output
```

**عبر i2cget للـ I2C devices:**

```bash
# قراءة كل الـ registers من 0x00 لـ 0x07
for reg in $(seq 0 7); do
    printf "reg 0x%02x: 0x%02x\n" $reg \
        $(i2cget -y 1 0x20 $(printf "0x%02x" $reg) 2>/dev/null || echo "err")
done
```

**عبر /dev/mem للـ MMIO registers:**

```bash
# قرأ 32-bit register على physical address 0x40020000
devmem2 0x40020000 w    # IDR — input data
devmem2 0x40020014 w    # ODR — output data
devmem2 0x40020018 w    # BSRR — bit set/reset
devmem2 0x40020004 w    # MODER — mode (direction)
```

**باستخدام io utility:**

```bash
# قرأ 4 bytes من address
io -4 0x40020000
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

**للـ I2C-based GPIO expanders:**

```
Setup:
  - Channel 1: SDA
  - Channel 2: SCL
  - Sample rate: ≥ 10× the I2C clock (≥ 4 MHz للـ 400kHz)
  - Trigger: falling edge على SDA مع SCL=HIGH (START condition)

Transaction قراءة input (PCA9557, addr=0x20, reg_dat=0x00):
  START → [0x41 = 0x20<<1 | R] → ACK → [DATA] → ACK → STOP

Transaction كتابة output (reg_set=0x01):
  START → [0x40 = 0x20<<1 | W] → ACK → [0x01] → ACK → [0xFF] → ACK → STOP

نقاط مهمة:
  - لو مفيش ACK بعد الـ address → عنوان غلط أو device مش powered
  - Clock Stretching → device بطيء، افحص الـ timing requirements
  - Repeated START → regmap بيقرأ بعد كتابة الـ register address
```

**للـ SPI-based GPIO expanders:**

```
Setup:
  - Channel 1: SCLK
  - Channel 2: MOSI
  - Channel 3: MISO
  - Channel 4: CS (active low)
  - Trigger: falling edge على CS

نقاط المراقبة:
  - تأكد إن CS بيبقى low طول الـ transaction بالكامل
  - الـ command byte الأول: MSB = 0 (write) أو 1 (read)
  - تأكد من CPOL/CPHA المناسبين للـ device من الـ regmap config
  - MISO لازم يكون valid في الـ half cycle الصح
```

**للـ GPIO lines نفسها:**

```bash
# استخدم logic analyzer على الـ GPIO pins وقارن مع الـ kernel timestamps
# فعّل gpio tracing
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on

# راقب الـ pin فيزيائياً بالـ logic analyzer
# الـ propagation delay بين كتابة الـ kernel والتغيير الفعلي لازم يكون < 1ms للـ I2C عادةً
```

---

#### 4. مشاكل Hardware الشائعة وأنماطها في الـ Kernel Log

| المشكلة | النمط في الـ Kernel Log | السبب | الحل |
|---|---|---|---|
| I2C device مش بيرد | `i2c i2c-1: ... timeout` / `-ETIMEDOUT` | pull-up resistors غلط أو مفيش power | افحص الـ pull-ups (عادةً 4.7kΩ) والـ VCC |
| قيم GPIO دايماً 0 | `gpio_regmap_get` بترجع 0 | `reg_dat_base` غلط أو pin مش connected | تحقق من الـ register addresses في الـ datasheet |
| الـ IRQ مش بيشتغل | `no irq for bank N` | `irq_domain` مش متضبط | تأكد من `CONFIG_GPIOLIB_IRQCHIP` وصحة الـ DT |
| الـ direction مش بيتغير | الـ `regmap_update_bits` ناجح لكن الـ pin ثابت | الـ pin fixed في الـ hardware أو `reg_dir_out_base` غلط | افحص الـ hardware ومقارن بالـ datasheet |
| SPI read دايماً 0xFF | `regmap_read` بترجع 0xFF | MISO مش connected أو CS polarity معكوسة | افحص التوصيلات والـ regmap_config SPI flags |
| قيم قديمة في الـ cache | GPIO kernel value مختلف عن الـ hardware | الـ cache مش بيعمل bypass للـ input registers | تأكد إن `reg_dat_base == reg_set_base` → يستخدم `regmap_read_bypassed` |
| `BUG: scheduling while atomic` | crash في الـ interrupt context | chip على I2C لكن استُدعي من atomic context | الـ `chip->can_sleep = true` لازم يكون set |
| power-on reset عشوائي | GPIO flip غير متوقع عند boot | direction registers مش initialized | ابدأ بـ `fixed_direction_output` أو init registers صراحةً |

---

#### 5. Device Tree Debugging

**تحقق من الـ DT node:**

```bash
# اعرض الـ compiled DT
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 30 "gpio-expander"

# اقرأ properties من /proc/device-tree
ls /proc/device-tree/soc/i2c@40005400/
cat /proc/device-tree/soc/i2c@40005400/gpio@20/compatible
cat /proc/device-tree/soc/i2c@40005400/gpio@20/reg | od -An -tu4

# تأكد من gpio-controller property
ls /proc/device-tree/soc/i2c@40005400/gpio@20/gpio-controller && \
    echo "gpio-controller: OK" || echo "gpio-controller: MISSING!"

# اقرأ ngpios
cat /proc/device-tree/soc/i2c@40005400/gpio@20/ngpios | od -An -tu4
```

**مثال DT node صحيح:**

```dts
&i2c1 {
    gpio_expander: gpio@20 {
        compatible = "nxp,pca9557";
        reg = <0x20>;

        gpio-controller;
        #gpio-cells = <2>;

        /* لازم يطابق ngpio في الـ driver config */
        ngpios = <8>;

        /* الـ IRQ line من الـ GPIO expander */
        interrupt-parent = <&gpio1>;
        interrupts = <5 IRQ_TYPE_LEVEL_LOW>;
        interrupt-controller;
        #interrupt-cells = <2>;
    };
};
```

**أخطاء DT شائعة:**

```bash
# تأكد إن الـ device اتعرف صح من الـ DT
ls /sys/bus/i2c/devices/1-0020/
cat /sys/bus/i2c/devices/1-0020/name    # لازم يطابق compatible string

# تأكد من الـ fwnode
# الـ gpio_regmap_register() بتستخدم config->fwnode لو set
# وإلا بتاخد dev_fwnode(config->parent) تلقائياً

# لو الـ chip مش بيتسجل كـ gpio-controller
dmesg | grep -i "gpio-controller\|gpio_chip\|gpiochip"
```

---

### Practical Commands

#### 1. سكريبت تشخيص شامل

```bash
#!/bin/bash
# gpio-regmap-debug.sh — تشخيص شامل لـ gpio-regmap device

CHIP_LABEL="${1:-}"          # مثال: pca9557
I2C_BUS="${2:-1}"
I2C_ADDR="${3:-0x20}"        # device address

echo "=== [1] GPIO Chips ==="
gpiodetect

echo ""
echo "=== [2] GPIO State (debugfs) ==="
cat /sys/kernel/debug/gpio

echo ""
echo "=== [3] Regmap Registers ==="
REGMAP_DEV="${I2C_BUS}-${I2C_ADDR#0x}"
if [ -d "/sys/kernel/debug/regmap/$REGMAP_DEV" ]; then
    cat /sys/kernel/debug/regmap/$REGMAP_DEV/registers
else
    echo "regmap debugfs not found for $REGMAP_DEV"
    echo "Available regmap instances:"
    ls /sys/kernel/debug/regmap/ 2>/dev/null
fi

echo ""
echo "=== [4] Direct I2C Hardware Read ==="
for reg in 0 1 2 3 4 5 6 7; do
    val=$(i2cget -y $I2C_BUS $I2C_ADDR $reg 2>/dev/null)
    printf "  reg[0x%02x] = %s\n" $reg "${val:-N/A}"
done

echo ""
echo "=== [5] GPIO sysfs Values ==="
for CHIP in $(ls /sys/class/gpio/ | grep gpiochip); do
    LABEL=$(cat /sys/class/gpio/$CHIP/label 2>/dev/null)
    BASE=$(cat /sys/class/gpio/$CHIP/base 2>/dev/null)
    NGPIO=$(cat /sys/class/gpio/$CHIP/ngpio 2>/dev/null)
    [ -n "$CHIP_LABEL" ] && [[ "$LABEL" != *"$CHIP_LABEL"* ]] && continue
    echo "  $CHIP ($LABEL): base=$BASE, ngpio=$NGPIO"
    for i in $(seq 0 $((NGPIO-1))); do
        GPIO=$((BASE+i))
        echo $GPIO > /sys/class/gpio/export 2>/dev/null
        DIR=$(cat /sys/class/gpio/gpio$GPIO/direction 2>/dev/null || echo "?")
        VAL=$(cat /sys/class/gpio/gpio$GPIO/value 2>/dev/null || echo "?")
        printf "    GPIO%-4d (line %2d): dir=%-4s val=%s\n" $GPIO $i "$DIR" "$VAL"
        echo $GPIO > /sys/class/gpio/unexport 2>/dev/null
    done
done

echo ""
echo "=== [6] IRQ Status ==="
cat /proc/interrupts | grep -i gpio

echo ""
echo "=== [7] Kernel Errors ==="
dmesg | grep -iE "gpio_regmap|regmap.*err|gpio.*err" | tail -20
```

---

#### 2. تفعيل ftrace شامل

```bash
#!/bin/bash
# gpio-ftrace.sh — تفعيل function_graph tracing لـ gpio-regmap

cd /sys/kernel/debug/tracing

# صفّر
echo "" > trace
echo nop > current_tracer

# ضبط الـ filter
cat > set_ftrace_filter << 'EOF'
gpio_regmap_get
gpio_regmap_set
gpio_regmap_set_with_clear
gpio_regmap_get_direction
gpio_regmap_set_direction
gpio_regmap_direction_input
gpio_regmap_direction_output
gpio_regmap_simple_xlate
regmap_read
regmap_write
regmap_update_bits
regmap_write_bits
regmap_read_bypassed
EOF

echo function_graph > current_tracer
echo 1 > tracing_on

echo "Tracing active. Press Enter after your test..."
read

echo 0 > tracing_on
cat trace
```

---

#### 3. مقارنة Hardware State مع Kernel State

```bash
#!/bin/bash
# compare-hw-kernel.sh — قارن قيم GPIO بين الـ kernel والـ hardware

I2C_BUS=1
I2C_ADDR=0x20
REG_DAT=0x00   # reg_dat_base

# قرأ من الـ hardware
HW_RAW=$(i2cget -y $I2C_BUS $I2C_ADDR $REG_DAT 2>/dev/null)
HW_VAL=$((${HW_RAW:-0}))
echo "Hardware register 0x$(printf '%02x' $REG_DAT) = $HW_RAW (decimal: $HW_VAL)"

# قارن بـ kernel sysfs
CHIP=$(ls /sys/class/gpio/ | grep gpiochip | head -1)
BASE=$(cat /sys/class/gpio/$CHIP/base 2>/dev/null || echo 0)
NGPIO=$(cat /sys/class/gpio/$CHIP/ngpio 2>/dev/null || echo 8)

echo ""
echo "Line | Kernel | Hardware | Match?"
echo "-----|--------|----------|-------"

for i in $(seq 0 $((NGPIO-1))); do
    GPIO=$((BASE+i))
    echo $GPIO > /sys/class/gpio/export 2>/dev/null
    K_VAL=$(cat /sys/class/gpio/gpio$GPIO/value 2>/dev/null || echo "?")
    HW_BIT=$(( (HW_VAL >> i) & 1 ))
    MATCH=$([ "$K_VAL" = "$HW_BIT" ] && echo "OK" || echo "*** MISMATCH ***")
    printf "  %2d | %6s | %8d | %s\n" $i "$K_VAL" $HW_BIT "$MATCH"
    echo $GPIO > /sys/class/gpio/unexport 2>/dev/null
done
```

---

#### 4. فحص سريع للـ IRQ

```bash
# اعرض interrupt counters
watch -n 1 'cat /proc/interrupts | grep -iE "gpio|pca"'

# اعرض IRQ domain mapping
cat /sys/kernel/debug/irq/domains/*/imap 2>/dev/null | head -30

# اختبر interrupt triggering
echo "falling" > /sys/class/gpio/gpio502/edge
cat /sys/class/gpio/gpio502/value &
# غيّر الـ pin فيزيائياً وشوف لو الـ IRQ counter زاد في /proc/interrupts
```

---

#### 5. اختبار Set/Clear Register Mechanism

```bash
# للـ chips عندها reg_set_base و reg_clr_base منفصلين
# الـ gpio_regmap_set_with_clear بيكتب mask في reg_set أو reg_clr

# اكتب 1 → يكتب في reg_set_base
echo 496 > /sys/class/gpio/export
echo out > /sys/class/gpio/gpio496/direction
echo 1 > /sys/class/gpio/gpio496/value

# اقرأ الـ hardware
i2cget -y 1 0x20 0x01   # SET register — المفروض يكون فيه bit set

# اكتب 0 → يكتب في reg_clr_base
echo 0 > /sys/class/gpio/gpio496/value

# اقرأ الـ hardware
i2cget -y 1 0x20 0x02   # CLR register — المفروض يكون فيه bit set

echo 496 > /sys/class/gpio/unexport
```

## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — GPIO Expander عبر I2C بيكتب قيمة غلط

#### العنوان
**GPIO expander** فوق I2C على gateway صناعي بيعمل output غلط بسبب خلط `reg_dat_base` بـ `reg_set_base`

#### السياق
شركة بتبني industrial gateway بـ RK3562. الـ gateway عنده PMIC خارجي (TCA6416A) متوصل بـ I2C، بيتحكم في relay outputs وـ digital inputs في نفس الـ bank. الـ driver بيستخدم `gpio-regmap` كـ abstraction layer.

#### المشكلة
الـ relay بيتقفل (HIGH) صح، لكن لما البرنامج بيقرأ حالة الـ GPIO بعد ما يكتب عليه، بيرجع قيمة قديمة (stale) من الـ cache بدل القيمة الفعلية على الـ line. الـ engineer بيشوف في dmesg:

```
gpio-regmap 1-0020: unexpected state on relay GPIO 3
```

#### التحليل
في `gpio_regmap_get()`:

```c
/* ensure we don't spoil any register cache with pin input values */
if (gpio->reg_dat_base == gpio->reg_set_base)
    ret = regmap_read_bypassed(gpio->regmap, reg, &val);
else
    ret = regmap_read(gpio->regmap, reg, &val);
```

الـ TCA6416A عنده register واحد بيقرأ منه ويكتب فيه (read-back = actual pin state). لما الـ driver بيتعمل configure بـ `reg_dat_base != reg_set_base` (يعني أديهم نفس القيمة بس في fieldين منفصلين بدل ما يعملوا `reg_dat_base == reg_set_base`)، الكود بيعمل `regmap_read()` عادي من الـ cache بدل `regmap_read_bypassed()`، فبيرجع آخر قيمة اتكتبت بدل الـ pin level الفعلي.

الـ config كانت كده:

```c
/* WRONG config */
.reg_dat_base = GPIO_REGMAP_ADDR(0x00),
.reg_set_base = GPIO_REGMAP_ADDR(0x02),  /* same physical reg but different address! */
```

بينما الصح إن الـ TCA6416A بيستخدم نفس الـ register للقراءة والكتابة:

```c
/* CORRECT config */
.reg_dat_base = GPIO_REGMAP_ADDR(0x00),
.reg_set_base = GPIO_REGMAP_ADDR(0x00),  /* same address => triggers bypassed read */
```

#### الحل

```c
static const struct gpio_regmap_config tca6416_gpio_config = {
    .parent        = &client->dev,
    .regmap        = regmap,
    /* both point to same register => gpio_regmap_get() will use regmap_read_bypassed */
    .reg_dat_base  = GPIO_REGMAP_ADDR(TCA6416_INPUT_P0),
    .reg_set_base  = GPIO_REGMAP_ADDR(TCA6416_INPUT_P0),
    .reg_dir_out_base = GPIO_REGMAP_ADDR(TCA6416_CONFIG_P0),
    .ngpio         = 16,
    .ngpio_per_reg = 8,
    .reg_stride    = 1,
};
```

التحقق:

```bash
# قرا الـ register مباشرة من الـ I2C bus
i2cget -y 1 0x20 0x00
# قارن مع gpioget
gpioget gpiochip2 3
```

#### الدرس المستفاد
لما الـ hardware بيستخدم نفس الـ register للـ input readback والـ output write، لازم `reg_dat_base == reg_set_base` في الـ config عشان `gpio_regmap_get()` يستخدم `regmap_read_bypassed()` ويتجاوز الـ cache ويقرأ القيمة الحقيقية من الـ hardware.

---

### السيناريو 2: Android TV Box على Allwinner H616 — HDMI HPD GPIO بيرفض يتحول لـ input

#### العنوان
**الـ HDMI Hot-Plug Detection GPIO** بيرفض يتحول لـ input بسبب `reg_dir_out_base` و`reg_dir_in_base` متضبطين مع بعض

#### السياق
المنتج: Android TV box بـ Allwinner H616. الـ HDMI HPD signal متوصل بـ GPIO عبر PMIC خارجي (AXP20x series) متصل بـ SPI. الـ driver بيستخدم `gpio-regmap` لتسجيل الـ GPIO bank.

#### المشكلة
لما الـ system بيبدأ، الـ HDMI مش بيتعرف وإن كانت الشاشة متوصلة. الـ dmesg بيظهر:

```
gpio spi0.0: direction_input failed for GPIO 5: -EINVAL
```

#### التحليل
في `gpio_regmap_register()`:

```c
/* we don't support having both registers simultaneously for now */
if (config->reg_dir_out_base && config->reg_dir_in_base)
    return ERR_PTR(-EINVAL);
```

الـ driver أصلي كان مكتوب لـ PMIC تاني بيعمل `reg_dir_out_base` و`reg_dir_in_base` معاً ظناً إن الـ hardware بيحتاجهم مع بعض، لكن `gpio-regmap` بيرفض الـ combination دي صراحةً لأن مفيش hardware بيحتاج الاتنين في نفس الوقت.

الـ config الغلط:

```c
/* WRONG: both set simultaneously */
.reg_dir_out_base = GPIO_REGMAP_ADDR(AXP_GPIO_CTRL),
.reg_dir_in_base  = GPIO_REGMAP_ADDR(AXP_GPIO_CTRL),  /* same reg, rejected! */
```

#### الحل
الـ AXP GPIO control register بيستخدم `reg_dir_out_base` فقط، الـ bit = 1 يعني output، bit = 0 يعني input:

```c
static const struct gpio_regmap_config axp_gpio_config = {
    .parent           = &spi->dev,
    .regmap           = axp_regmap,
    .reg_dat_base     = GPIO_REGMAP_ADDR(AXP_GPIO_INPUT),
    .reg_set_base     = GPIO_REGMAP_ADDR(AXP_GPIO_OUTPUT),
    /* Use ONLY reg_dir_out_base, never both */
    .reg_dir_out_base = GPIO_REGMAP_ADDR(AXP_GPIO_DIRECTION),
    /* reg_dir_in_base = 0 => not set */
    .ngpio            = 4,
    .ngpio_per_reg    = 4,
};
```

التحقق:

```bash
# تأكد إن الـ gpio chip اتسجل
cat /sys/kernel/debug/gpio | grep axp
# تحقق من الـ direction
gpioinfo gpiochip3
```

#### الدرس المستفاد
`gpio-regmap` بيرفض صراحةً إن `reg_dir_out_base` و`reg_dir_in_base` يتضبطوا مع بعض. الـ hardware عنده register واحد بيتحكم في الاتجاهين، استخدم واحد بس: إما `reg_dir_out_base` (1=output) أو `reg_dir_in_base` (1=input). الكود بيعكس النتيجة تلقائياً عبر `invert` flag في `gpio_regmap_get_direction()` و`gpio_regmap_set_direction()`.

---

### السيناريو 3: IoT Sensor Node على STM32MP1 — SET/CLR Registers بيعمل glitch على SPI CS

#### العنوان
**SPI chip-select GPIO** بيعمل glitch لأن `gpio_regmap_set_with_clear()` بيكتب `mask` كامل في الـ CLR register

#### السياق
المنتج: IoT sensor node بـ STM32MP157. الـ SPI flash متوصل بـ GPIO expander خارجي (PCAL6408A) عبر I2C. الـ expander عنده output latch register واحد بس، ومفيش SET/CLR registers منفصلة. الـ engineer حاول يعمل optimization وضبط `reg_clr_base` على نفس عنوان الـ output register.

#### المشكلة
الـ SPI transactions بتتعطل بشكل عشوائي، والـ oscilloscope بيظهر إن الـ CS line بتعمل pulse غريبة في منتصف الـ transaction.

#### التحليل
لما `reg_set_base` و`reg_clr_base` متضبطين مع بعض في `gpio_regmap_register()`:

```c
if (gpio->reg_set_base && gpio->reg_clr_base)
    chip->set = gpio_regmap_set_with_clear;
```

وفي `gpio_regmap_set_with_clear()`:

```c
static int gpio_regmap_set_with_clear(struct gpio_chip *chip,
                                      unsigned int offset, int val)
{
    /* ... */
    if (val)
        base = gpio_regmap_addr(gpio->reg_set_base);
    else
        base = gpio_regmap_addr(gpio->reg_clr_base);

    ret = gpio->reg_mask_xlate(gpio, base, offset, &reg, &mask);
    if (ret)
        return ret;

    return regmap_write(gpio->regmap, reg, mask);  /* writes mask as value! */
}
```

لاحظ: `regmap_write(gpio->regmap, reg, mask)` بيكتب الـ `mask` كـ value كاملة في الـ register — مش read-modify-write. لو الـ hardware مش فعلاً dedicated SET/CLR registers، ده بيعمل overwrite لكل الـ outputs في الـ bank.

الـ config الغلط:

```c
/* WRONG: PCAL6408A has no separate CLR register */
.reg_set_base = GPIO_REGMAP_ADDR(PCAL6408A_OUTPUT_REG),
.reg_clr_base = GPIO_REGMAP_ADDR(PCAL6408A_OUTPUT_REG),  /* same address! */
```

فلما الـ CS بيتعمل deassert، الكود بيكتب `BIT(cs_pin)` فقط في الـ register — وده بيعمل clear لكل الـ CS lines التانية في نفس الوقت.

#### الحل
الـ PCAL6408A مفيش له dedicated CLR register، الصح هو استخدام `reg_set_base` فقط:

```c
static const struct gpio_regmap_config pcal6408a_gpio_config = {
    .parent           = &client->dev,
    .regmap           = pcal_regmap,
    .reg_dat_base     = GPIO_REGMAP_ADDR(PCAL6408A_INPUT_REG),
    .reg_set_base     = GPIO_REGMAP_ADDR(PCAL6408A_OUTPUT_REG),
    /* NO reg_clr_base => uses gpio_regmap_set() with regmap_write_bits() */
    .reg_dir_out_base = GPIO_REGMAP_ADDR(PCAL6408A_CONFIG_REG),
    .ngpio            = 8,
    .ngpio_per_reg    = 8,
};
```

بكده `gpio_regmap_register()` بيختار:

```c
else if (gpio->reg_set_base)
    chip->set = gpio_regmap_set;  /* uses regmap_write_bits => safe */
```

والـ `gpio_regmap_set()` بيستخدم `regmap_write_bits` اللي بتعمل read-modify-write وبتعدل الـ bit المطلوب بس.

```bash
# تحقق من الـ glitch باستخدام ftrace
echo gpio > /sys/kernel/debug/tracing/set_event
echo 1 > /sys/kernel/debug/tracing/tracing_on
# شغل الـ SPI transaction
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace | grep "gpiochip"
```

#### الدرس المستفاد
`gpio_regmap_set_with_clear()` بيكتب الـ `mask` مباشرة في الـ register بـ `regmap_write()` — مش read-modify-write. ده صحيح بس لو الـ hardware فعلاً عنده atomic SET/CLR registers (زي STM32 BSRR). لو مش موجودين، استخدم `reg_set_base` بدون `reg_clr_base`.

---

### السيناريو 4: Automotive ECU على i.MX8MP — `fixed_direction_output` bitmap بيحمي safety GPIO من تغيير الاتجاه

#### العنوان
**watchdog trigger GPIO** على ECU اتقفل كـ output عن طريق `fixed_direction_output` bitmap وده منع الـ safety logic من تغيير الاتجاه عن طريق الخطأ

#### السياق
المنتج: automotive ECU بـ i.MX8MP. الـ ECU عنده GPIO expander خارجي (MAX7319) متوصل بـ I2C. الـ GPIO bank مخلوط: بعض الـ lines output-only (watchdog, relay) وبعضها input-only (door sensor, ignition). الـ driver بيستخدم `fixed_direction_output` bitmap في الـ config.

#### المشكلة
بعد software update، الـ watchdog GPIO أخد command من user-space يحاول يحوله لـ input، والنتيجة كانت إن الـ watchdog بطل يشتغل وبعدين الـ system reset نفسه.

#### التحليل
في `gpio_regmap_get_direction()`:

```c
if (gpio->fixed_direction_output) {
    if (test_bit(offset, gpio->fixed_direction_output))
        return GPIO_LINE_DIRECTION_OUT;
    else
        return GPIO_LINE_DIRECTION_IN;
}
```

لما `fixed_direction_output` متضبط، الكود بيرجع الـ direction الثابتة بدون ما يقرأ من الـ hardware — ده سلوك `get_direction` فقط. لكن `gpio_regmap_set_direction()` مش بيعمل check على الـ `fixed_direction_output` bitmap:

```c
static int gpio_regmap_set_direction(struct gpio_chip *chip,
                                     unsigned int offset, bool output)
{
    if (gpio->reg_dir_out_base) {
        base = gpio_regmap_addr(gpio->reg_dir_out_base);
        invert = 0;
    } else if (gpio->reg_dir_in_base) {
        base = gpio_regmap_addr(gpio->reg_dir_in_base);
        invert = 1;
    } else {
        return -ENOTSUPP;
    }
    /* ... continues to write hardware */
}
```

الـ driver الأصلي للـ MAX7319 ما ضبطش `reg_dir_in_base` ولا `reg_dir_out_base` (لأنه output-only expander) وضبط `fixed_direction_output` بـ all-ones bitmap. فـ `direction_input` callback مش موجودة في الـ `gpio_chip`:

```c
/* in gpio_regmap_register() */
if (gpio->reg_dir_in_base || gpio->reg_dir_out_base) {
    chip->direction_input  = gpio_regmap_direction_input;
    chip->direction_output = gpio_regmap_direction_output;
}
/* if neither set => direction callbacks are NULL => gpiolib blocks direction change */
```

المشكلة الحقيقية كانت في الـ DT: الـ watchdog GPIO تعريفه ما كانش بيمنع user-space من تغييره لأن `gpio-reserved-ranges` مش متضبط.

#### الحل

في الـ Device Tree:

```dts
&i2c3 {
    max7319: gpio@38 {
        compatible = "maxim,max7319";
        reg = <0x38>;
        gpio-controller;
        #gpio-cells = <2>;
        ngpios = <8>;
        /* lock watchdog GPIO from userspace */
        gpio-reserved-ranges = <0 1>;  /* GPIO 0 = watchdog, reserved */
    };
};
```

وفي الـ driver config:

```c
/* bitmap: GPIO 0,1,2 are fixed outputs (watchdog, relay1, relay2) */
static const unsigned long max7319_fixed_out = BIT(0) | BIT(1) | BIT(2);

static const struct gpio_regmap_config max7319_config = {
    .parent                 = &client->dev,
    .regmap                 = regmap,
    .reg_set_base           = GPIO_REGMAP_ADDR(MAX7319_OUT_REG),
    /* output-only: no reg_dat_base, no direction registers */
    .ngpio                  = 8,
    .fixed_direction_output = &max7319_fixed_out,
};
```

#### الدرس المستفاد
`fixed_direction_output` بيحدد الـ direction اللي `get_direction()` بيرجعها، لكنه مش بيمنع الـ kernel أو user-space من استدعاء `direction_input()`. الحماية الحقيقية من التغيير تيجي من: (1) عدم تسجيل `direction_input` callback بعدم ضبط `reg_dir_in_base`/`reg_dir_out_base`، و(2) استخدام `gpio-reserved-ranges` في الـ DT.

---

### السيناريو 5: Custom Board Bring-up على AM62x — `ngpio_per_reg` غلط بيعمل register address خطأ

#### العنوان
**GPIO offset 16** فأكتر بيكتب في register عنوانه غلط بسبب `ngpio_per_reg` و`reg_stride` غير متضبطين صح

#### السياق
المنتج: custom evaluation board بـ TI AM62x. الـ board عنده GPIO expander (PCAL6524) متوصل بـ I2C، بيوفر 24 GPIO موزعين على 3 registers (8 GPIOs per register). الـ engineer الجديد بيعمل bring-up للـ driver.

#### المشكلة
الـ GPIOs من 0 لـ 15 بتشتغل صح. لكن GPIOs من 16 لـ 23 مش بتستجيب — لا read ولا write صح. الـ i2c-tools بيأكد إن الـ hardware سليم.

#### التحليل
الـ `gpio_regmap_simple_xlate()` هي المسؤولة عن حساب الـ register address:

```c
static int gpio_regmap_simple_xlate(struct gpio_regmap *gpio,
                                    unsigned int base, unsigned int offset,
                                    unsigned int *reg, unsigned int *mask)
{
    unsigned int line   = offset % gpio->ngpio_per_reg;
    unsigned int stride = offset / gpio->ngpio_per_reg;

    *reg  = base + stride * gpio->reg_stride;
    *mask = BIT(line);

    return 0;
}
```

لو الـ engineer ضبط `ngpio_per_reg = 16` بدل `8` (ظن إن كل register بيحتوي 16 GPIO):

```c
/* WRONG config */
.ngpio_per_reg = 16,   /* should be 8 */
.reg_stride    = 1,
.reg_dat_base  = GPIO_REGMAP_ADDR(PCAL6524_IN_P0),  /* 0x00 */
```

لـ offset = 16:
```
line   = 16 % 16 = 0
stride = 16 / 16 = 1
reg    = 0x00 + 1 * 1 = 0x01   ← WRONG, should be 0x02 (port 2)
```

الـ PCAL6524 registers:
- Port 0 Input: `0x00`
- Port 1 Input: `0x01`
- Port 2 Input: `0x02`

بالـ config الغلط، GPIO 16 بيتكتب في reg `0x01` (port 1) بدل `0x02` (port 2).

الـ config الصح:

```c
/* CORRECT: PCAL6524 = 3 ports x 8 GPIOs each */
.ngpio_per_reg = 8,    /* 8 GPIOs per 8-bit register */
.reg_stride    = 1,    /* registers are consecutive: 0x00, 0x01, 0x02 */
```

الحساب الصح لـ offset = 16:
```
line   = 16 % 8 = 0
stride = 16 / 8 = 2
reg    = 0x00 + 2 * 1 = 0x02   ← CORRECT (port 2)
```

#### الحل

```c
static const struct gpio_regmap_config pcal6524_config = {
    .parent           = &client->dev,
    .regmap           = pcal6524_regmap,
    .reg_dat_base     = GPIO_REGMAP_ADDR(PCAL6524_IN_P0),     /* 0x00 */
    .reg_set_base     = GPIO_REGMAP_ADDR(PCAL6524_OUT_P0),    /* 0x04 */
    .reg_dir_out_base = GPIO_REGMAP_ADDR(PCAL6524_CONFIG_P0), /* 0x0C */
    .ngpio            = 24,
    .ngpio_per_reg    = 8,   /* 8-bit registers */
    .reg_stride       = 1,   /* consecutive addresses */
};
```

التحقق السريع:

```bash
# اكتب على GPIO 16 (port 2, bit 0)
gpioset gpiochip4 16=1
# تحقق من الـ register مباشرة عبر i2c
i2cget -y 2 0x22 0x04  # OUT_P0
i2cget -y 2 0x22 0x05  # OUT_P1
i2cget -y 2 0x22 0x06  # OUT_P2 — should show 0x01
```

#### الدرس المستفاد
`ngpio_per_reg` هو عدد الـ GPIO bits في كل register (مش عدد الـ registers). لازم يساوي عرض الـ register فعلياً (8 لـ 8-bit registers). الـ `reg_stride` هو الفرق بالـ bytes بين registers متتالية. غلطة في أي منهم بتعمل silent mismatch — الكود مش بيطلع error، بس بيكتب في register غلط تماماً.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي تغطي الـ gpio-regmap وما يتعلق بيه:

| المقال | الرابط | الأهمية |
|--------|--------|----------|
| gpio-regmap: Support few custom operations | [lwn.net/Articles/857361](https://lwn.net/Articles/857361/) | بيتكلم عن إضافة custom callbacks لـ gpio_regmap عند التسجيل |
| RTL8231 GPIO expander — uses gpio-regmap | [lwn.net/Articles/859520](https://lwn.net/Articles/859520/) | مثال real-world على استخدام gpio-regmap في GPIO expander |
| drivers: gpio: QIXIS FPGA GPIO — fixed_direction_output | [lwn.net/Articles/1038833](https://lwn.net/Articles/1038833/) | إضافة bitmap الـ `fixed_direction_output` لـ gpio-regmap |
| regmap: Generic I2C and SPI register map library | [lwn.net/Articles/451789](https://lwn.net/Articles/451789/) | الأساس — بيشرح إزاي اتصنعت الـ regmap framework |
| regmap: Add asynchronous I/O support | [lwn.net/Articles/534893](https://lwn.net/Articles/534893/) | تطور الـ regmap بعد التأسيس |
| GPIO Driver Interface (kernel docs mirror) | [static.lwn.net/kerneldoc/driver-api/gpio/driver.html](https://static.lwn.net/kerneldoc/driver-api/gpio/driver.html) | الـ official GPIO driver API |

---

### توثيق الـ kernel الرسمي

**الـ `Documentation/` paths الأهم:**

```
Documentation/driver-api/gpio/driver.rst      # GPIO driver interface — إزاي تكتب GPIO driver
Documentation/driver-api/gpio/board.rst       # GPIO board mappings
Documentation/driver-api/gpio/consumer.rst    # إزاي تستخدم GPIO من driver تاني
Documentation/core-api/regmap.rst             # الـ regmap API كاملة
```

**الـ online links:**
- [GPIO Driver Interface](https://docs.kernel.org/driver-api/gpio/driver.html)
- [GPIO Mappings](https://docs.kernel.org/driver-api/gpio/board.html)
- [GPIO — General Index](https://docs.kernel.org/driver-api/gpio/index.html)
- [GPIO Descriptor Driver Interface](https://docs.kernel.org/driver-api/gpio/driver.html)

**الـ header files المهمة في الكود:**

```
include/linux/gpio/regmap.h    # struct gpio_regmap_config — الـ public API
include/linux/gpio/driver.h    # struct gpio_chip — القلب
include/linux/regmap.h         # regmap_read/write/update_bits وغيرها
drivers/gpio/gpiolib.h         # internal gpiolib helpers
```

---

### الـ Commits المهمة في الـ Kernel

**الـ patch series الأصلية — إدخال gpio-regmap (أبريل/مايو 2020):**
- Michael Walle أضاف الـ driver عشان يوحّد كود الـ GPIO controllers البسيطة اللي بتستخدم regmap
- الـ patchwork entry: [patchwork.ozlabs.org — v2,10/16](https://patchwork.ozlabs.org/project/linux-gpio/patch/20200402203656.27047-11-michael@walle.cc/)
- الـ lore link للـ final version: [`lore.kernel.org/r/20200528145845.31436-3-michael@walle.cc`](https://lore.kernel.org/r/20200528145845.31436-3-michael@walle.cc)

**الـ commit message الأصلي قال:**

> "there are quite a lot simple GPIO controller which are using regmap to access the hardware. This driver tries to be a base to unify existing code into one place."

**تطورات لاحقة مهمة:**

| الموضوع | الرابط |
|---------|--------|
| دعم registers بأكتر من bit لكل GPIO (`ngpio_per_reg`) | [lore.kernel.org](https://lore.kernel.org/all/4ca4580a3f5157e3ac7a5c8943ef607b@walle.cc/) |
| expose الـ `struct gpio_regmap` في الـ header — William Gray | [lore.kernel.org](https://lore.kernel.org/lkml/5c0354c87d4d2a082cf0c331076d5aad18a93169.1677547393.git.william.gray@linaro.org/) |

**لمشاهدة كل تغييرات الملف:**
```bash
git log --follow drivers/gpio/gpio-regmap.c
```

---

### نقاشات الـ Mailing List

| الموضوع | الرابط |
|---------|--------|
| PATCH 0/3 — gpio-regmap support for register fields and other hooks | [lore.kernel.org](https://lore.kernel.org/linux-gpio/4ca4580a3f5157e3ac7a5c8943ef607b@walle.cc/t/) |
| PATCH v6 0/3 — gpio: generic regmap implementation | [lkml.kernel.org](https://lkml.kernel.org/lkml/6cec4ed5-3a4a-62d8-5209-da3b0863dcd1@linux.intel.com/T/) |
| PATCH 2/3 — Expose struct gpio_regmap | [lore.kernel.org](https://lore.kernel.org/lkml/5c0354c87d4d2a082cf0c331076d5aad18a93169.1677547393.git.william.gray@linaro.org/) |
| Re: Support registers with more than one bit per GPIO | [lkml.org](https://lkml.org/lkml/2022/7/4/1051) |
| gpio: i8255 — migrate to gpio-regmap API (review by Andy Shevchenko) | [lore.kernel.org](https://lore.kernel.org/lkml/Y35bgFmiMW3uTm/O@smile.fi.intel.com/) |
| PATCH 1/3 — Support registers with more than one bit per GPIO | [lists.archive.carbon60.com](https://lists.archive.carbon60.com/linux/kernel/4618445) |

---

### الكتب الموصى بها

#### Linux Device Drivers 3rd Edition (LDD3)
- **Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman**
- الفصول المهمة:
  - **Chapter 9** — Communicating with Hardware (memory-mapped I/O)
  - **Chapter 10** — Interrupt Handling
  - **Chapter 14** — The Linux Device Model — `struct device`, `devm_*` APIs
- متاحة مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **Chapter 17** — Devices and Modules
- **Chapter 7** — Interrupts and Interrupt Handlers
- بيغطي الـ device model اللي بتبني عليه الـ `gpio_chip`

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **Chapter 8** — Device Driver Fundamentals
- **Chapter 15** — Debugging Embedded Linux Applications
- مناسب لفهم الـ GPIO في سياق embedded systems وإزاي الـ regmap بيسهّل كتابة الـ drivers

#### Mastering Linux Device Driver Development — John Madieu (Packt)
- **Chapter 2** — Leveraging the Regmap API
- ده أكتر كتاب بيتكلم عن الـ regmap بالتفصيل في سياق driver development
- متوفر online: [subscription.packtpub.com](https://subscription.packtpub.com/book/mobile/9781789342048/3/ch03lvl1sec10/introduction-to-regmap-and-its-data-structures-i2c-spi-and-mmio)

---

### kernelnewbies.org

صفحات بتتبع تغييرات الـ GPIO عبر kernel releases:

| الصفحة | ما يخص الـ GPIO |
|--------|----------------|
| [Linux_6.13](https://kernelnewbies.org/Linux_6.13) | دعم Aspeed G7 GPIO |
| [Linux_6.11](https://kernelnewbies.org/Linux_6.11) | إضافة GPIO PWM driver |
| [Linux_6.6](https://kernelnewbies.org/Linux_6.6) | تغييرات GPIO متعددة |
| [Linux_6.2](https://kernelnewbies.org/Linux_6.2) | إضافة GPIO latch driver ودعم software nodes في gpiolib |
| [LinuxChanges](https://kernelnewbies.org/LinuxChanges) | الفهرس الرئيسي لكل التغييرات |

---

### elinux.org

| الصفحة | الموضوع |
|--------|---------|
| [Pin Control and GPIO Update (PDF)](https://elinux.org/images/c/cd/Pincontrol-gpio-update.pdf) | شرح العلاقة بين pinctrl وGPIO subsystems — مهم لفهم الصورة الكبيرة |
| [Leapster Explorer: GPIO subsystem](https://elinux.org/Leapster_Explorer:_GPIO_subsystem) | مثال embedded GPIO subsystem حي |
| [Hammer LED Driver Module](https://elinux.org/Hammer_LED_Driver_Module) | مثال بسيط على كتابة GPIO driver من الصفر |

---

### مقالات تقنية خارجية

- **Using regmaps to make Linux drivers more generic** — Collabora Blog (2020):
  [collabora.com — regmaps to make drivers more generic](https://www.collabora.com/news-and-blog/blog/2020/05/27/using-regmaps-to-make-linux-drivers-more-generic/)
  — مقال ممتاز بيشرح exactly الـ use case اللي اتصمم عشانه الـ gpio-regmap

- **Linux Drivers 3 — Regmap in Detail**:
  [hackerbikepacker.com/linux-drivers-regmap](https://hackerbikepacker.com/linux-drivers-regmap)
  — شرح تفصيلي للـ regmap API بأمثلة

- **Linux Kernel Driver DataBase — CONFIG_GPIO_REGMAP**:
  [cateee.net/lkddb — GPIO_REGMAP](https://cateee.net/lkddb/web-lkddb/GPIO_REGMAP.html)
  — dependencies والـ Kconfig للـ driver وكل الـ kernel versions اللي فيها

---

### الكود على الإنترنت

```
# Elixir cross-reference — أفضل طريقة لقراءة الكود مع links
https://elixir.bootlin.com/linux/latest/source/drivers/gpio/gpio-regmap.c
https://elixir.bootlin.com/linux/latest/source/include/linux/gpio/regmap.h
https://elixir.bootlin.com/linux/latest/source/include/linux/gpio/driver.h

# الكود على Android kernel (Google)
https://android.googlesource.com/kernel/common.git/+/refs/heads/android-mainline/drivers/gpio/gpio-regmap.c
```

---

### Search Terms للبحث عن مزيد من المعلومات

```
# في الـ mailing list
site:lore.kernel.org gpio-regmap
site:lore.kernel.org "gpio_regmap_register"
site:lore.kernel.org "devm_gpio_regmap_register"

# في الـ LWN
site:lwn.net "gpio_chip" regmap
site:lwn.net gpio-regmap

# بحث عام
linux kernel gpio regmap driver tutorial
linux gpio_chip regmap implementation
linux devm_gpio_regmap_register example
linux kernel regmap_read_bypassed gpio
linux gpio expander i2c regmap driver example
linux gpio-regmap fixed_direction_output bitmap
linux kernel gpiochip_irqchip_add_domain regmap
linux reg_mask_xlate callback gpio

# في الـ kernel source نفسه
grep -rl "devm_gpio_regmap_register\|gpio_regmap_register" drivers/
git log --all --oneline -- drivers/gpio/ | grep -i regmap
```
## Phase 8: Writing simple module

### الفكرة

الـ `gpio_regmap_register()` هي أهم function exported في الملف — هي اللي بتسجل أي GPIO controller معتمد على regmap. هنعمل kprobe عليها عشان نشوف إمتى بيتسجل controller جديد ونطبع معلومات عنه زي عدد الـ GPIOs واسم الـ parent device.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * kprobe module: trace gpio_regmap_register() calls
 *
 * Every time a regmap-based GPIO controller is registered,
 * we print its label, parent device name, and ngpio count.
 */

#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/gpio/regmap.h>   /* gpio_regmap_config */
#include <linux/device.h>        /* dev_name() */

/* ------------------------------------------------------------------ */
/* kprobe pre-handler: fires just before gpio_regmap_register() runs   */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64, the first argument is in RDI.
     * On ARM64, it is in x0.
     * kprobes gives us regs; we cast the first argument register
     * to the pointer type we expect.
     */
#ifdef CONFIG_X86_64
    const struct gpio_regmap_config *cfg =
        (const struct gpio_regmap_config *)regs->di;
#elif defined(CONFIG_ARM64)
    const struct gpio_regmap_config *cfg =
        (const struct gpio_regmap_config *)regs->regs[0];
#else
    /* Fallback: unsupported arch — just note the call */
    pr_info("gpio_regmap_register: called (arch not decoded)\n");
    return 0;
#endif

    /* Basic sanity: config and parent must not be NULL */
    if (!cfg || !cfg->parent) {
        pr_info("gpio_regmap_register: called with NULL config or parent\n");
        return 0;
    }

    pr_info("gpio_regmap_register: parent=%s label=%s ngpio=%d "
            "reg_dat_base=0x%x reg_set_base=0x%x\n",
            dev_name(cfg->parent),               /* e.g. "1-0041"        */
            cfg->label ? cfg->label : "(null)",  /* optional label string */
            cfg->ngpio,                          /* number of GPIO lines  */
            cfg->reg_dat_base,                   /* input register base   */
            cfg->reg_set_base);                  /* output/set reg base   */

    return 0; /* 0 = let the real function continue normally */
}

/* ------------------------------------------------------------------ */
/* kprobe descriptor                                                   */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "gpio_regmap_register",  /* function to probe        */
    .pre_handler = handler_pre,             /* our callback             */
};

/* ------------------------------------------------------------------ */
/* module init / exit                                                  */
/* ------------------------------------------------------------------ */
static int __init gpio_regmap_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("gpio_regmap_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("gpio_regmap_probe: planted kprobe on %s @ %p\n",
            kp.symbol_name, kp.addr);
    return 0;
}

static void __exit gpio_regmap_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("gpio_regmap_probe: kprobe removed\n");
}

module_init(gpio_regmap_probe_init);
module_exit(gpio_regmap_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe tracer for gpio_regmap_register()");
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|--------|-------|
| `linux/kprobes.h` | بيعرّف `struct kprobe` و `register_kprobe` / `unregister_kprobe` — أساس الـ hooking |
| `linux/gpio/regmap.h` | بيجيب تعريف `struct gpio_regmap_config` اللي هنفسر منها الـ argument |
| `linux/device.h` | بيجيب `dev_name()` عشان نطبع اسم الـ parent device بشكل مقروء |

---

#### الـ `handler_pre`

الـ `pre_handler` بيتشغل مباشرةً **قبل** ما الـ CPU ينفذ أول instruction في `gpio_regmap_register`. بنجيب الـ argument الأول (الـ `config`) من الـ registers حسب الـ calling convention لكل architecture، لأن الـ kernel لسه ما دخلش جوف الـ function ولا عمل stack frame جديد.

بنطبع باستخدام `pr_info` اسم الـ parent device وعدد الـ GPIO lines وعناوين الـ registers الأساسية — ده بيساعد تتبع أي driver بيسجل controller جديد وبياناته التفصيلية في وقت التشغيل.

---

#### الـ `struct kprobe`

- **`symbol_name`**: الـ kernel بيحل الاسم ده لـ address وقت `register_kprobe`، مش محتاج تعرف العنوان يدويًا.
- **`pre_handler`**: الـ pointer على الـ callback اللي هيتشغل — لو كنا محتاجين نشوف القيمة اللي رجعتها الـ function كنا هنستخدم `post_handler` بدل كده.

---

#### الـ `module_init` و `module_exit`

- **init**: بيسجل الـ kprobe — لو فشل (مثلاً الـ function غير موجودة أو مجموعة على `__init`) بيرجع error فورًا.
- **exit**: الـ `unregister_kprobe` ضروري جدًا — لو ما اتعملش والـ module اتفك من الـ kernel، الـ handler pointer بيبقى dangling pointer وهيودي لـ kernel panic في أول استدعاء.

---

### Makefile

```makefile
obj-m += gpio_regmap_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### تجربة الـ module

```bash
# بناء الـ module
make

# تحميله
sudo insmod gpio_regmap_probe.ko

# شوف الـ log (لو عندك device بيستخدم gpio-regmap هيظهر تلقائي)
sudo dmesg | grep gpio_regmap

# تفريغه
sudo rmmod gpio_regmap_probe

# تأكيد إنه اتشال
sudo dmesg | tail -5
```

مثال لـ output متوقع على جهاز فيه I2C GPIO expander مثل PCA9557:

```
[  12.345678] gpio_regmap_probe: planted kprobe on gpio_regmap_register @ ffffffffc0ab1234
[  12.901234] gpio_regmap_register: parent=1-0041 label=pca9557 ngpio=8 reg_dat_base=0x0 reg_set_base=0x1
```
