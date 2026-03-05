## Phase 1: الصورة الكبيرة ببساطة

### ما هو هذا الملف؟

الملف `gpio-adnp.txt` هو **Device Tree binding** — يعني وثيقة رسمية بتحدد إزاي تكتب الـ device tree node عشان تعرّف الـ kernel بشريحة **ADNP** (Avionic Design N-bit GPIO Expander).

---

### الصورة الكبيرة — تخيل السيناريو ده

تخيل إنك بتصمم لوحة إلكترونية (embedded board) جوا طيارة أو معدة صناعية، والـ CPU اللي عندك بيطلع من اللوحة عنده مثلاً 8 أو 16 pin فقط زي GPIO. المشكلة إنك محتاج تتحكم في 64 pin في نفس الوقت — تفتح صمامات، تقرأ حساسات، تشغل ريليهات.

الحل؟ تشتري شريحة **GPIO Expander** — شريحة صغيرة بتتوصل بالـ CPU عن طريق **I2C** (سلكين بس!) وبتديك 8 أو 16 أو 32 أو 64 pin إضافية.

**الـ ADNP** هي شريحة كده بالظبط، من شركة **Avionic Design GmbH** الألمانية، متخصصة في معدات الطيران. الـ "N-bit" في الاسم يعني إن الشريحة ممكن تيجي بأعداد مختلفة من الـ pins (8، 16، 32، 64…).

---

### ليه محتاجين binding file؟

الـ Linux kernel مش بيعرف تلقائياً إن في شريحة ADNP متوصلة بالـ I2C على address معين. الـ **Device Tree** هو الملف اللي بيوصف الـ hardware للـ kernel — زي ورقة مواصفات بتقول "في شريحة GPIO على I2C address 0x41 عندها 64 pin وبتعمل interrupt".

الـ binding file ده (`gpio-adnp.txt`) هو **العقد الرسمي** اللي بيحدد القواعد: إيه الـ properties المطلوبة، وإيه الـ optional، وإيه مثال صح.

---

### إيه اللي بتعمله الشريحة؟

الشريحة بتوفر:

| الوظيفة | التفاصيل |
|---|---|
| **GPIO controller** | بيديك pins تقدر تقرأ منها أو تكتب فيها |
| **Interrupt controller** (اختياري) | لما أي pin بيتغير، بيبعت interrupt للـ CPU |
| **N-bit flexible** | تدعم 8 لـ 64 pin حسب الشريحة المستخدمة |
| **I2C interface** | التواصل مع الـ CPU بسلكين بس |

---

### الـ Registers جوا الشريحة

الـ driver (`gpio-adnp.c`) بيتعامل مع 5 registers:

| الـ Register | الاسم | الوظيفة |
|---|---|---|
| `GPIO_DDR` | Data Direction Register | بيحدد كل pin هو input ولا output |
| `GPIO_PLR` | Pin Level Register | بيقرأ أو بيكتب مستوى الـ pin |
| `GPIO_IER` | Interrupt Enable Register | بيفعّل أو بيعطل الـ interrupt لكل pin |
| `GPIO_ISR` | Interrupt Status Register | بيحدد انهي pins عندها interrupt pending |
| `GPIO_PTR` | (Pointer/Type) | بيحدد نوع الـ interrupt |

---

### إيه الـ Properties في الـ Binding؟

```
gpioext: gpio-controller@41 {
    compatible = "ad,gpio-adnp";    /* عشان الـ kernel يعرف الـ driver المناسب */
    reg = <0x41>;                   /* I2C address للشريحة على الـ bus */

    interrupt-parent = <&gpio>;    /* مين اللي بيستقبل الـ interrupt من الشريحة */
    interrupts = <160 1>;          /* رقم الـ interrupt وطريقة التشغيل */

    gpio-controller;               /* إعلان إن ده controller مش مجرد consumer */
    #gpio-cells = <2>;             /* كل GPIO بيحتاج 2 خلية: رقم + flags */

    interrupt-controller;          /* اختياري: ممكن يكون هو كمان interrupt controller */
    #interrupt-cells = <2>;        /* لو interrupt controller: 2 خلايا للوصف */

    nr-gpios = <64>;               /* عدد الـ pins في الشريحة */
};
```

**الـ `#gpio-cells = <2>`** معناها إن لما أي device تاني في الـ device tree يريد يستخدم pin من الشريحة دي، بيكتب رقمين:
- الأول: رقم الـ GPIO (0 لـ 63)
- التاني: الـ flags (0 = normal polarity, 1 = inverted)

---

### القصة الكاملة من البداية للنهاية

```
Board boots up
     │
     ▼
Device Tree loader يقرأ الـ .dts file
     │
     ▼
يلاقي node بـ compatible = "ad,gpio-adnp"
     │
     ▼
الـ kernel يبحث عن driver يعرف "ad,gpio-adnp"
     │
     ▼
يلاقي gpio-adnp.c في drivers/gpio/
     │
     ▼
adnp_i2c_probe() بتشتغل:
 - بتقرأ nr-gpios من الـ device tree
 - بتسجل gpio_chip مع الـ gpiolib
 - لو interrupt-controller موجودة: بتسجل irq_chip كمان
     │
     ▼
أي driver تاني يقدر يطلب GPIO من الشريحة دي
عن طريق الـ gpiolib interface
```

---

### الـ Subsystem اللي بينتمي له

الملف ده جزء من **GPIO Subsystem** في الـ Linux kernel. الـ subsystem ده مسؤول عن:
- توحيد interface التعامل مع كل أنواع GPIO controllers
- الـ **gpiolib** اللي بيربط الـ consumers بالـ controllers
- دعم **interrupt chaining** لما الـ GPIO يعمل كـ interrupt controller

---

### الملفات المهمة في الـ Subsystem

#### الـ Core والـ Framework

| الملف | الدور |
|---|---|
| `/drivers/gpio/gpiolib.c` | قلب الـ GPIO subsystem — بيدير كل الـ gpio chips |
| `/drivers/gpio/gpiolib-of.c` | الربط بين الـ device tree والـ gpiolib |
| `/include/linux/gpio/driver.h` | الـ API اللي الـ drivers بتستخدمه لتسجيل نفسها |
| `/include/linux/gpio/consumer.h` | الـ API اللي الـ consumers بتستخدمه لطلب GPIOs |
| `/include/linux/gpio.h` | الـ legacy GPIO API |

#### الـ Driver الخاص بالشريحة

| الملف | الدور |
|---|---|
| `/drivers/gpio/gpio-adnp.c` | الـ driver الكامل للشريحة ADNP |

#### الـ Binding Documentation

| الملف | الدور |
|---|---|
| `/Documentation/devicetree/bindings/gpio/gpio-adnp.txt` | **الملف ده** — binding spec |
| `/Documentation/devicetree/bindings/gpio/gpio.txt` | القواعد العامة لأي GPIO controller في DT |
| `/Documentation/devicetree/bindings/interrupt-controller/interrupts.txt` | مرجع للـ interrupt specifier المستخدم |

---

### ملاحظة مهمة للقارئ

الـ binding file ده بسيط جداً (34 سطر) لكنه كثيف المعنى. كل property فيه بتربط بين:
- **الـ hardware** (الشريحة الفعلية وسلوكها)
- **الـ device tree** (الوصف الـ declarative)
- **الـ driver** (`gpio-adnp.c`) اللي بيترجم كل ده لـ kernel objects

لو حابب تفهم أعمق، ابدأ بـ `gpio-adnp.c` وبعدين `gpio/driver.h`.
## Phase 2: شرح الـ GPIO Subsystem Framework

---

### المشكلة — ليه الـ GPIO Subsystem موجود أصلاً؟

في أي SoC زي i.MX8 أو STM32، عندك عشرات الـ GPIO pins مدمجة جوه الـ SoC نفسه.
بس في كتير من الـ embedded designs، الـ SoC pins بتبقى مش كافية، فبتضيف
**GPIO expander** خارجي بيتكلم مع الـ SoC عن طريق I2C أو SPI.

**المشكلة الحقيقية:** كل driver محتاج يتحكم في GPIO — سواء كان LED driver,
button driver, أو regulator — ما ينفعش يعرف هو شغال مع GPIO داخلي في الـ SoC
ولا مع expander خارجي. لو كل driver كتب كود خاص بيه للتعامل مع كل نوع hardware،
الكود هيبقى chaos خالص.

**التحديات:**
- GPIO داخلي في ARM SoC بيتعامل معاه عن طريق memory-mapped registers (fast, no sleep)
- GPIO expander زي الـ ADNP بيتعامل معاه عن طريق I2C (slow, **can sleep**)
- بعض الـ GPIO chips بتقدر تطلق interrupts، وبعضها مش بتقدر
- الـ Device Tree بيحدد الـ GPIOs بـ phandle + offset + flags

---

### الحل — الـ GPIO Subsystem

الـ kernel حل المشكلة بـ **abstraction layer** اسمه **gpiolib**.

الفكرة: أي hardware GPIO controller (سواء SoC أو expander) بيسجّل نفسه كـ
`gpio_chip` — وهو struct بيحتوي على function pointers. الـ consumers (باقي الـ drivers)
بيستخدموا API موحد مش فارق معاهم فين الـ GPIO فعلياً.

```
Consumer Driver          gpiolib Core           gpio_chip (Provider)
─────────────────        ────────────           ────────────────────
gpiod_get()        →     lookup + validate  →   chip->request()
gpiod_direction_input()  route to chip      →   chip->direction_input()
gpiod_get_value()        validate + call    →   chip->get()
gpiod_set_value()        validate + call    →   chip->set()
```

---

### التشبيه الحقيقي — مقبس الكهرباء العالمي

تخيّل إنك بتسافر وعندك شاحن laptop محتاج يشتغل في أي بلد.
بدل ما تغيّر الشاحن في كل بلد، بتستخدم **travel adapter**.

| التشبيه | المقابل في الـ kernel |
|---|---|
| الشاحن (consumer) | أي driver محتاج GPIO (LED, button, regulator) |
| Travel Adapter | `gpio_chip` — الـ abstraction layer |
| المقبس الأمريكي (110V) | GPIO داخلي في SoC (memory-mapped) |
| المقبس الأوروبي (220V) | GPIO expander زي ADNP عبر I2C |
| شركة التصنيع للـ adapter | Driver كاتب الـ `gpio_chip` callbacks |
| الـ voltage conversion | الـ register translation داخل الـ driver |
| ما تقدر توصل على 3 مقابس في نفس الوقت | الـ mutex على I2C bus (i2c_lock) |
| كود كهربائي عالمي يحمي المستخدم | الـ gpiolib core: validate, locking, sysfs |

الـ consumer (شاحن الـ laptop) ما بيعرفش ولا يهمه 110V ولا 220V.
وبالظبط كده، الـ LED driver ما بيعرفش ولا يهمه GPIO في SoC ولا في I2C expander.

---

### البنية الكاملة — Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      Consumer Layer                             │
│  LED Driver    Button Driver    Regulator Driver    SPI Driver  │
│      │               │                │                 │       │
│      └───────────────┴────────────────┴─────────────────┘       │
│                             │                                   │
│                    gpiod_get() / gpiod_set_value()               │
└─────────────────────────────┼───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│                     gpiolib Core                                │
│                                                                 │
│   ┌─────────────────┐    ┌──────────────────┐                  │
│   │  gpio_desc mgmt │    │  IRQ Domain mgmt │                  │
│   │  (per-line state│    │  (GPIO→IRQ map)  │                  │
│   │   flags, label) │    └──────────────────┘                  │
│   └─────────────────┘                                          │
│   ┌──────────────────────────────────────────┐                 │
│   │         struct gpio_chip (vtable)         │                 │
│   │  .get()  .set()  .direction_input()  ...  │                 │
│   └──────────────────────────────────────────┘                 │
│                                                                 │
│   ┌────────────┐  ┌────────────┐  ┌────────────┐               │
│   │ OF/DT xlate│  │ sysfs/char │  │  debugfs   │               │
│   │ (phandle→  │  │  device    │  │  dbg_show  │               │
│   │  gpio_desc)│  └────────────┘  └────────────┘               │
│   └────────────┘                                               │
└─────────────────────────────┬───────────────────────────────────┘
                              │  registers gpio_chip
         ┌────────────────────┼────────────────────┐
         │                    │                    │
┌────────▼──────┐   ┌─────────▼──────┐   ┌────────▼──────┐
│ SoC GPIO Ctrl │   │  ADNP Expander │   │  PCA953x etc. │
│ (MMIO-based)  │   │  (I2C-based)   │   │  (I2C-based)  │
│               │   │                │   │               │
│ gpio-mxc.c    │   │ gpio-adnp.c    │   │ gpio-pca953x.c│
└───────┬───────┘   └───────┬────────┘   └───────┬───────┘
        │                   │                    │
   ARM SoC Regs          I2C Bus             I2C/SMBus
   (direct MMIO)     (addr 0x41, etc.)
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ abstraction الأساسية في الـ GPIO subsystem هي الـ **`struct gpio_chip`**.

هو مش بس struct — هو **vtable** بالمعنى الحرفي: table of function pointers
بتمثل الـ capabilities بتاعة أي GPIO controller.

```c
struct gpio_chip {
    const char  *label;        /* اسم الـ chip */
    struct device *parent;     /* الـ device اللي بيملكه (I2C client مثلاً) */

    /* ── الـ vtable: الـ driver بيملّيها ── */
    int  (*direction_input) (struct gpio_chip *gc, unsigned int offset);
    int  (*direction_output)(struct gpio_chip *gc, unsigned int offset, int value);
    int  (*get)             (struct gpio_chip *gc, unsigned int offset);
    int  (*set)             (struct gpio_chip *gc, unsigned int offset, int value);
    int  (*get_direction)   (struct gpio_chip *gc, unsigned int offset);
    int  (*request)         (struct gpio_chip *gc, unsigned int offset); /* optional */
    void (*free)            (struct gpio_chip *gc, unsigned int offset); /* optional */

    /* ── metadata ── */
    int   base;      /* أول GPIO number في الـ global space, -1 = dynamic */
    u16   ngpio;     /* عدد الـ pins اللي الـ chip بيتحكم فيها */
    bool  can_sleep; /* لازم true لو access بيحتاج sleep (I2C/SPI) */

    /* ── IRQ integration ── */
    struct gpio_irq_chip irq; /* embedded irqchip لو الـ chip بتطلق interrupts */
};
```

الـ `offset` هو دايماً **chip-relative** (من 0 لـ ngpio-1).
الـ gpiolib core هو اللي بيعمل الـ translation من الـ global GPIO number
للـ chip + offset المناسب.

---

### الـ ADNP كـ gpio_chip — Deep Dive

الـ **ADNP** (Avionic Design N-bit GPIO expander) هو I2C chip
بيوفر حتى 64 GPIO pin مقسّمين على 8 registers (كل register 8 bits).

#### الـ Hardware Registers

```
Register Layout (per 8-pin bank, address = base << reg_shift):
┌──────────────┬───────────────────────────────────────────────────┐
│  Reg Offset  │  وظيفته                                          │
├──────────────┼───────────────────────────────────────────────────┤
│  DDR (0x00)  │  Data Direction Register: 1=output, 0=input       │
│  PLR (0x01)  │  Pin Level Register: اقرأ/اكتب قيمة الـ pin      │
│  IER (0x02)  │  Interrupt Enable Register: enable/disable IRQ    │
│  ISR (0x03)  │  Interrupt Status Register: pending IRQs          │
│  PTR (0x04)  │  (محجوز للـ hardware)                             │
└──────────────┴───────────────────────────────────────────────────┘
```

الـ `reg_shift` بيحدد كام register بنحتاجه:
- 8 GPIOs  → reg_shift = 0 → 1 register per type
- 16 GPIOs → reg_shift = 1 → 2 registers per type
- 64 GPIOs → reg_shift = 3 → 8 registers per type

```c
/* تحويل GPIO offset لـ register + bit position */
unsigned int reg = offset >> adnp->reg_shift; /* رقم الـ register */
unsigned int pos = offset & 7;                /* رقم الـ bit جوه الـ register */
```

#### الـ struct adnp — الـ Private State

```c
struct adnp {
    struct i2c_client *client;   /* الـ I2C device handle */
    struct gpio_chip   gpio;     /* الـ gpio_chip embedded (مش pointer!) */
    unsigned int       reg_shift;/* log2(num_gpios/8) */

    struct mutex i2c_lock;       /* يحمي الـ I2C bus access */
    struct mutex irq_lock;       /* يحمي تعديل الـ IRQ config */

    /* shadow registers للـ IRQ state (في RAM، مش في hardware) */
    u8 *irq_enable;  /* ايه الـ pins عندها IRQ enabled */
    u8 *irq_level;   /* آخر level لكل pin (لعمل edge detection) */
    u8 *irq_rise;    /* pins محتاجة RISING edge */
    u8 *irq_fall;    /* pins محتاجة FALLING edge */
    u8 *irq_high;    /* pins محتاجة LEVEL HIGH */
    u8 *irq_low;     /* pins محتاجة LEVEL LOW */
};
```

**ملاحظة مهمة:** الـ `gpio_chip` مش pointer — هو **embedded** جوه `struct adnp`.
الـ driver بيستخدم `gpiochip_get_data(chip)` للوصول لـ `adnp` من الـ `gpio_chip`.

#### العلاقة بين الـ Structs

```
struct adnp
┌──────────────────────────────────────────────────────────┐
│  client ──────────────────────► struct i2c_client        │
│                                 ┌────────────────────┐   │
│                                 │ addr = 0x41        │   │
│                                 │ irq = <parent IRQ> │   │
│                                 └────────────────────┘   │
│  reg_shift = 3  (for 64 GPIOs)                           │
│  i2c_lock  = mutex                                       │
│  irq_lock  = mutex                                       │
│                                                          │
│  gpio ◄── embedded struct gpio_chip                      │
│  ┌────────────────────────────────────────────────────┐  │
│  │ label    = "gpio-adnp"                             │  │
│  │ parent   = &client->dev                            │  │
│  │ ngpio    = 64                                      │  │
│  │ base     = -1  (dynamic allocation)                │  │
│  │ can_sleep= true  ← مهم جداً لأن I2C بيحتاج sleep  │  │
│  │                                                    │  │
│  │ .direction_input  = adnp_gpio_direction_input      │  │
│  │ .direction_output = adnp_gpio_direction_output     │  │
│  │ .get              = adnp_gpio_get                  │  │
│  │ .set              = adnp_gpio_set                  │  │
│  │ .dbg_show         = adnp_gpio_dbg_show             │  │
│  │                                                    │  │
│  │ irq ◄── embedded struct gpio_irq_chip              │  │
│  │ ┌──────────────────────────────────────────────┐  │  │
│  │ │ chip     → adnp_irq_chip (irq_chip vtable)   │  │  │
│  │ │ handler  = handle_simple_irq                 │  │  │
│  │ │ threaded = true                              │  │  │
│  │ │ domain   ← allocated by gpiolib core         │  │  │
│  │ └──────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  irq_enable[8]  ← shadow of IER registers               │
│  irq_level[8]   ← cached PLR for edge emulation         │
│  irq_rise[8]    ← software bitmask                      │
│  irq_fall[8]    ← software bitmask                      │
│  irq_high[8]    ← software bitmask                      │
│  irq_low[8]     ← software bitmask                      │
└──────────────────────────────────────────────────────────┘
```

---

### الـ GPIO Subsystem يملك إيه vs. بيفوّض إيه للـ Driver؟

#### ما بيملكه الـ gpiolib core:
| الوظيفة | التفاصيل |
|---|---|
| Global GPIO numbering | تخصيص base numbers للـ chips، وتحويل global → chip+offset |
| gpio_desc management | الـ struct اللي بتمثل كل line (requested، direction، label) |
| DT/OF translation | تحويل `<&gpioext 5 0>` لـ gpio_desc |
| sysfs / chardev interface | `/dev/gpiochipN` و `gpioget`/`gpioset` tools |
| IRQ domain creation | إنشاء الـ irq_domain للـ chip وربط GPIO numbers بـ Linux IRQ numbers |
| Locking policies | منع double-request لنفس الـ pin |
| debugfs integration | استدعاء `chip->dbg_show` لو موجودة |

#### ما بيفوّضه للـ Driver (gpio_chip callbacks):
| الوظيفة | في ADNP |
|---|---|
| قراءة قيمة pin | `adnp_gpio_get()` → I2C read من PLR register |
| كتابة قيمة pin | `adnp_gpio_set()` → read-modify-write على PLR |
| تحديد direction | `adnp_gpio_direction_input/output()` → كتابة DDR register |
| IRQ mask/unmask | تعديل shadow `irq_enable[]` |
| IRQ bus sync | فلاش الـ shadow registers للـ hardware في `bus_sync_unlock` |
| IRQ type | تسجيل edge/level type في الـ shadow bitmasks |
| Parent IRQ handling | `adnp_irq()` — threaded IRQ handler يقرأ ISR ويطلق child IRQs |

---

### الـ IRQ Subsystem — مفهوم تكميلي مهم

قبل ما تفهم الـ IRQ integration في الـ ADNP، لازم تعرف الـ **IRQ Domain** concept:

> **الـ irq_domain** هو translation table بيربط الـ hardware IRQ numbers (hwirq)
> بالـ Linux virtual IRQ numbers (virq). كل GPIO controller بيحتاج domain خاص بيه.

الـ ADNP بيستخدم **nested threaded IRQ** pattern:
- الـ parent IRQ (من الـ SoC GPIO) بيصحى thread واحد (الـ I2C access مش safe في hardirq context)
- الـ thread بيقرأ ISR registers عن طريق I2C
- بيحسب أي GPIOs عندها pending interrupt
- بيستدعي `handle_nested_irq()` على كل child IRQ

```
SoC GPIO Pin 160 fires IRQ
         │
         ▼
adnp_irq() [threaded handler, can sleep]
         │
         ├── I2C read PLR (current levels)
         ├── I2C read ISR (hardware pending)
         ├── I2C read IER (enabled mask)
         │
         ├── software edge detection:
         │   changed = current_level XOR adnp->irq_level[i]
         │   pending = changed & (rise_mask | fall_mask)
         │           | (high_mask & level) | (low_mask & ~level)
         │   pending &= isr & ier
         │
         └── for each set bit in pending:
             child_irq = irq_find_mapping(domain, gpio_offset)
             handle_nested_irq(child_irq)
                      │
                      ▼
             Consumer driver's IRQ handler
             (e.g., button driver)
```

#### ليه الـ bus_lock/bus_sync_unlock pattern؟

الـ I2C bus بطيء والـ kernel IRQ core بيستدعي `irq_mask/unmask` من contexts مختلفة.
الحل هو الـ **two-phase commit**:

1. `adnp_irq_bus_lock()` → يمسك الـ `irq_lock` mutex
2. تعديلات على الـ shadow registers (في RAM فقط، سريع)
3. `adnp_irq_bus_sync_unlock()` → يكتب التغييرات للـ hardware عبر I2C (ممكن يـ sleep)، ثم يفك الـ mutex

```c
/* في adnp_irq_bus_sync_unlock */
scoped_guard(mutex, &adnp->i2c_lock) {
    for (i = 0; i < num_regs; i++)
        adnp_write(adnp, GPIO_IER(adnp) + i,
                   adnp->irq_enable[i]); /* flush shadow → hardware */
}
mutex_unlock(&adnp->irq_lock);
```

هو اللي معرّف في `struct irq_chip` كـ:
- `.irq_bus_lock` = `adnp_irq_bus_lock`
- `.irq_bus_sync_unlock` = `adnp_irq_bus_unlock`

---

### Registration Flow — من probe لحد gpiochip ready

```
adnp_i2c_probe()
    │
    ├── device_property_read_u32("nr-gpios") → num_gpios = 64
    ├── devm_kzalloc → struct adnp
    ├── devm_mutex_init → i2c_lock
    │
    └── adnp_gpio_setup(adnp, 64, is_irq_controller=true)
            │
            ├── reg_shift = get_count_order(64) - 3 = 6 - 3 = 3
            │
            ├── chip->direction_input  = adnp_gpio_direction_input
            ├── chip->direction_output = adnp_gpio_direction_output
            ├── chip->get              = adnp_gpio_get
            ├── chip->set              = adnp_gpio_set
            ├── chip->can_sleep        = true
            ├── chip->ngpio            = 64
            ├── chip->base             = -1 (dynamic)
            │
            ├── adnp_irq_setup(adnp)
            │       ├── alloc shadow arrays (irq_enable, level, rise, fall, high, low)
            │       ├── read initial PLR → irq_level (لـ edge emulation)
            │       ├── write 0 → all IER (disable all IRQs)
            │       └── devm_request_threaded_irq(IRQF_TRIGGER_RISING | IRQF_ONESHOT)
            │
            ├── girq->chip    = &adnp_irq_chip
            ├── girq->handler = handle_simple_irq
            ├── girq->threaded= true
            │
            └── devm_gpiochip_add_data()
                    │
                    ▼
              gpiolib core:
              - assigns base GPIO numbers
              - creates irq_domain
              - registers sysfs/chardev
              - chip is now live
```

---

### Device Tree ربطها بالكود

```dts
gpioext: gpio-controller@41 {
    compatible = "ad,gpio-adnp";   /* يتطابق مع adnp_of_match */
    reg = <0x41>;                  /* I2C address → client->addr */

    interrupt-parent = <&gpio>;    /* الـ parent IRQ controller (SoC GPIO) */
    interrupts = <160 1>;          /* client->irq = virq(160), RISING */

    gpio-controller;               /* يعلن إن الـ node ده gpio_chip */
    #gpio-cells = <2>;             /* <pin_offset  flags> */

    interrupt-controller;          /* يعلن إن الـ chip ده بيطلق IRQs برضو */
    #interrupt-cells = <2>;

    nr-gpios = <64>;               /* device_property_read_u32("nr-gpios") */
};
```

Consumer بيستخدمه كده:
```dts
some-device {
    reset-gpios = <&gpioext 5 GPIO_ACTIVE_LOW>;
    /*               ^       ^  ^
                     |       |  flags (polarity)
                     |       chip-relative offset
                     phandle للـ gpio_chip */
};
```

الـ gpiolib core بيعمل الـ translation:
`<&gpioext 5 GPIO_ACTIVE_LOW>` → `gpio_chip` للـ ADNP + offset=5 + flag=inverted → `gpio_desc`
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Macros والـ Register Map

الـ driver بيتعامل مع الـ hardware عبر 5 registers لكل bank من 8 pins:

| Macro | Expression | Register | الوظيفة |
|-------|-----------|----------|---------|
| `GPIO_DDR(gpio)` | `0x00 << reg_shift` | Data Direction Register | بيتحكم في اتجاه الـ pin: input أو output |
| `GPIO_PLR(gpio)` | `0x01 << reg_shift` | Pin Level Register | بيقرأ أو بيكتب القيمة الفعلية للـ pin |
| `GPIO_IER(gpio)` | `0x02 << reg_shift` | Interrupt Enable Register | بيفعّل أو بيعطّل الـ interrupt لكل pin |
| `GPIO_ISR(gpio)` | `0x03 << reg_shift` | Interrupt Status Register | بيوضح الـ pins اللي عندها interrupt pending |
| `GPIO_PTR(gpio)` | `0x04 << reg_shift` | Pin Type Register | نوع الـ trigger (معرّف لكن مش مستخدم مباشرة في الكود) |

> **الـ `reg_shift`**: هو `log2(num_gpios) - 3`. مثلاً لو عندك 64 GPIO → `reg_shift = 3` → كل register يبدأ على offset مضروب في 8. ده معناه إن الـ DDR registers مثلاً هي: `0x00, 0x08, 0x10, ...` لكل bank.

---

### الـ IRQ Type Flags

| Flag | القيمة | المعنى |
|------|--------|--------|
| `IRQ_TYPE_EDGE_RISING` | `0x1` | trigger عند الصعود من low → high |
| `IRQ_TYPE_EDGE_FALLING` | `0x2` | trigger عند الهبوط من high → low |
| `IRQ_TYPE_LEVEL_HIGH` | `0x4` | trigger طالما الـ pin على high |
| `IRQ_TYPE_LEVEL_LOW` | `0x8` | trigger طالما الـ pin على low |

---

### الـ irq_chip Flags

| Flag | المعنى في السياق |
|------|-----------------|
| `IRQCHIP_IMMUTABLE` | الـ irq_chip object مش ممكن يتعدّل بعد التسجيل — حماية من الـ hierarchy IRQ |
| `GPIOCHIP_IRQ_RESOURCE_HELPERS` | macro بيضيف `.irq_print_chip` و `.irq_request_resources` و `.irq_release_resources` تلقائياً |

---

### الـ Probe Config Options (Device Tree)

| Property | نوعه | الوظيفة |
|----------|------|---------|
| `compatible` | string | لازم `"ad,gpio-adnp"` |
| `reg` | u32 | عنوان الـ I2C slave |
| `interrupts` | phandle + cells | الـ IRQ الآتي من الـ parent GPIO controller |
| `#gpio-cells` | u32 = 2 | cell أول = رقم الـ GPIO، تاني = polarity |
| `gpio-controller` | boolean | يعلن الـ device كـ GPIO controller |
| `nr-gpios` | u32 | عدد الـ pins (8، 16، 32، 64...) |
| `interrupt-controller` | boolean | اختياري — يفعّل الـ IRQ controller mode |
| `#interrupt-cells` | u32 = 2 | لو الـ device كـ interrupt controller |

---

### الـ Structs المهمة

#### 1. `struct adnp` — القلب الأساسي للـ driver

```c
struct adnp {
    struct i2c_client *client;   /* الـ I2C device الفعلي — بوابة التواصل مع الـ hardware */
    struct gpio_chip   gpio;     /* embedded struct — يمثّل الـ GPIO controller في kernel */
    unsigned int       reg_shift;/* كم bit نشيف عشان نوصل للـ register bank الصح */

    struct mutex i2c_lock;       /* يحمي كل عمليات القراءة/الكتابة على I2C bus */
    struct mutex irq_lock;       /* يحمي الـ irq_enable shadow register أثناء bus_lock/unlock */

    u8 *irq_enable;  /* shadow لـ IER register — أي pins enabled */
    u8 *irq_level;   /* آخر قيمة قرأناها للـ PLR — عشان نحسب edge */
    u8 *irq_rise;    /* pins مفروض تـ trigger عند rising edge */
    u8 *irq_fall;    /* pins مفروض تـ trigger عند falling edge */
    u8 *irq_high;    /* pins مفروض تـ trigger طول ما هي high */
    u8 *irq_low;     /* pins مفروض تـ trigger طول ما هي low */
};
```

**ملاحظة حيوية على الـ shadow buffers**: كل الـ 6 pointers (`irq_enable` → `irq_low`) بتشاور على contiguous block واحد بحجم `num_regs * 6` bytes. الـ layout:

```
[ irq_enable[0..N] | irq_level[0..N] | irq_rise[0..N] | irq_fall[0..N] | irq_high[0..N] | irq_low[0..N] ]
  offset=0           offset=N          offset=2N         offset=3N         offset=4N         offset=5N
```

---

#### 2. `struct gpio_chip` — الـ embedded GPIO abstraction

الـ `struct adnp` بيحتوي `gpio_chip` مش كـ pointer لكن كـ embedded value — ده معناه إن `container_of` / `gpiochip_get_data` بتشتغل بكفاءة.

| Field | القيمة في الـ driver | المعنى |
|-------|---------------------|--------|
| `direction_input` | `adnp_gpio_direction_input` | بيكتب الـ DDR register |
| `direction_output` | `adnp_gpio_direction_output` | بيكتب الـ DDR ثم الـ PLR |
| `get` | `adnp_gpio_get` | بيقرأ من الـ PLR |
| `set` | `adnp_gpio_set` | بيكتب على الـ PLR |
| `dbg_show` | `adnp_gpio_dbg_show` | لـ debugfs (لو `CONFIG_DEBUG_FS`) |
| `can_sleep` | `true` | I2C operations ممكن تـ sleep |
| `base` | `-1` | يخلي الـ kernel يختار الـ base تلقائياً |
| `ngpio` | `num_gpios` من DT | عدد الـ lines |
| `label` | `client->name` | اسم الـ device |
| `parent` | `&client->dev` | الـ device الأب |
| `irq` | `struct gpio_irq_chip` | الـ IRQ chip المدمج |

---

#### 3. `struct gpio_irq_chip` — الـ IRQ side بتاع gpio_chip

| Field | القيمة في الـ driver | المعنى |
|-------|---------------------|--------|
| `chip` | `&adnp_irq_chip` | الـ irq_chip operations |
| `parent_handler` | `NULL` | لأن الـ IRQ nested مش cascaded |
| `num_parents` | `0` | مفيش parent IRQ domain |
| `parents` | `NULL` | — |
| `default_type` | `IRQ_TYPE_NONE` | مفيش default trigger type |
| `handler` | `handle_simple_irq` | الـ flow handler لكل child IRQ |
| `threaded` | `true` | الـ IRQ handler بيشتغل في thread |
| `domain` | (auto by gpiolib) | الـ IRQ domain اللي بتتـ map فيه الـ child IRQs |

---

#### 4. `struct irq_chip adnp_irq_chip` — الـ static IRQ operations

```c
static const struct irq_chip adnp_irq_chip = {
    .name                  = "gpio-adnp",
    .irq_mask              = adnp_irq_mask,
    .irq_unmask            = adnp_irq_unmask,
    .irq_set_type          = adnp_irq_set_type,
    .irq_bus_lock          = adnp_irq_bus_lock,
    .irq_bus_sync_unlock   = adnp_irq_bus_unlock,
    .flags                 = IRQCHIP_IMMUTABLE,
    GPIOCHIP_IRQ_RESOURCE_HELPERS,
};
```

الـ `irq_bus_lock` / `irq_bus_sync_unlock` pattern موجود لأن الـ I2C بطيء — بيخليك تعمل batch لكل التغييرات وتكتبهم للـ hardware مرة واحدة لما تخلص.

---

### رسم العلاقات بين الـ Structs

```
┌─────────────────────────────────────────────────────────┐
│                     struct adnp                          │
│                                                         │
│  ┌──────────────────┐    ┌─────────────────────────┐   │
│  │ struct i2c_client│◄───│  *client                │   │
│  │  .irq            │    └─────────────────────────┘   │
│  │  .dev            │                                   │
│  │  .name           │    ┌──────────────────────────┐  │
│  └──────────────────┘    │ unsigned int reg_shift    │  │
│                          └──────────────────────────┘  │
│  ┌──────────────────────────────────────────────────┐  │
│  │           struct gpio_chip  (embedded)            │  │
│  │  .direction_input  → adnp_gpio_direction_input   │  │
│  │  .direction_output → adnp_gpio_direction_output  │  │
│  │  .get              → adnp_gpio_get               │  │
│  │  .set              → adnp_gpio_set               │  │
│  │  .parent  ─────────────► struct device           │  │
│  │  .ngpio, .base, .label, .can_sleep               │  │
│  │                                                   │  │
│  │  ┌────────────────────────────────────────────┐  │  │
│  │  │      struct gpio_irq_chip  (embedded)      │  │  │
│  │  │  .chip ──────► struct irq_chip (static)    │  │  │
│  │  │  .handler = handle_simple_irq              │  │  │
│  │  │  .threaded = true                          │  │  │
│  │  │  .domain ──────► struct irq_domain         │  │  │
│  │  │                   (auto by gpiolib)        │  │  │
│  │  └────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────┘  │
│                                                         │
│  struct mutex i2c_lock   ← يحمي كل I2C transactions   │
│  struct mutex irq_lock   ← يحمي shadow registers       │
│                                                         │
│  u8 *irq_enable ─┐                                     │
│  u8 *irq_level   │                                     │
│  u8 *irq_rise    ├──► contiguous buffer (num_regs×6)  │
│  u8 *irq_fall    │                                     │
│  u8 *irq_high    │                                     │
│  u8 *irq_low   ──┘                                     │
└─────────────────────────────────────────────────────────┘
```

---

### دورة حياة الـ Driver (Lifecycle)

```
[Device Tree / ACPI]
        │
        │  "ad,gpio-adnp" matched
        ▼
┌─────────────────────┐
│  adnp_i2c_probe()   │  ← kernel calls this when I2C device is found
│                     │
│  1. read nr-gpios   │  device_property_read_u32(dev, "nr-gpios")
│  2. alloc struct    │  devm_kzalloc → struct adnp on heap
│  3. init i2c_lock   │  devm_mutex_init
│  4. set client ptr  │  adnp->client = client
│  5. gpio_setup()    │
└────────┬────────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│  adnp_gpio_setup()                      │
│                                         │
│  1. calc reg_shift                      │
│     get_count_order(num_gpios) - 3      │
│                                         │
│  2. fill gpio_chip ops                  │
│     .direction_input/output, .get/.set  │
│     .can_sleep=true, .base=-1           │
│                                         │
│  3. if interrupt-controller:            │
│     └─► adnp_irq_setup()               │
│          ├─ init irq_lock               │
│          ├─ alloc 6×num_regs shadow buf │
│          ├─ read initial PLR levels     │
│          ├─ write 0 to all IER regs     │
│          └─ devm_request_threaded_irq   │
│              (adnp_irq, RISING|ONESHOT) │
│                                         │
│  4. fill gpio_irq_chip fields           │
│     gpio_irq_chip_set_chip(girq, chip)  │
│                                         │
│  5. devm_gpiochip_add_data()            │
│     ├─ assigns base number              │
│     ├─ creates sysfs entries            │
│     └─ creates IRQ domain (if irq chip) │
└─────────────────────────────────────────┘
         │
         ▼
   [Driver ACTIVE — GPIOs available to consumers]
         │
         │  (device removed / driver unload)
         ▼
┌──────────────────────────────┐
│  devm cleanup (auto)         │
│  ├─ gpiochip_remove()        │
│  ├─ free_irq()               │
│  ├─ kfree shadow buffers     │
│  └─ kfree struct adnp        │
└──────────────────────────────┘
```

---

### Call Flow: قراءة GPIO

```
consumer calls: gpiod_get_value(desc)
  │
  └─► gpio_chip->get(chip, offset)        [gpio_chip ops dispatch]
        = adnp_gpio_get(chip, offset)
            │
            ├─ gpiochip_get_data(chip)    → struct adnp *adnp
            ├─ reg  = offset >> reg_shift  (which bank)
            ├─ pos  = offset & 7           (which bit in bank)
            │
            └─► adnp_read(adnp, GPIO_PLR(adnp) + reg, &value)
                    │
                    └─► i2c_smbus_read_byte_data(client, offset)
                              │
                              ▼
                        [I2C hardware — reads PLR register]
                              │
                              ▼
                    return (value & BIT(pos)) ? 1 : 0
```

---

### Call Flow: كتابة GPIO (set direction output)

```
consumer calls: gpiod_direction_output(desc, value)
  │
  └─► gpio_chip->direction_output(chip, offset, value)
        = adnp_gpio_direction_output(chip, offset, value)
            │
            ├─ guard(mutex)(&adnp->i2c_lock)    ← lock I2C
            │
            ├─ adnp_read(GPIO_DDR + reg)         read current DDR
            ├─ val |= BIT(pos)                   set bit = output
            ├─ adnp_write(GPIO_DDR + reg, val)   write back
            ├─ adnp_read(GPIO_DDR + reg)         verify write
            ├─ if !(val & BIT(pos)) → -EPERM     hardware rejected it
            │
            └─► __adnp_gpio_set(adnp, offset, value)
                    │                            (still under i2c_lock)
                    ├─ adnp_read(GPIO_PLR + reg)
                    ├─ modify bit
                    └─ adnp_write(GPIO_PLR + reg, new_val)
                              │
                              ▼
                        [I2C hardware — writes DDR then PLR]
```

---

### Call Flow: معالجة الـ Interrupt

```
[hardware GPIO line changes level]
        │
        ▼
[parent interrupt controller fires]
        │
        └─► adnp_irq(irq, data)         [threaded IRQ handler]
                │
                ├─ for each bank i (0..num_regs-1):
                │   ├─ scoped_guard(mutex, &i2c_lock)
                │   │   ├─ adnp_read(GPIO_PLR + i) → level
                │   │   ├─ adnp_read(GPIO_ISR + i) → isr
                │   │   └─ adnp_read(GPIO_IER + i) → ier
                │   │
                │   ├─ changed = level ^ irq_level[i]   (edge detection SW)
                │   │
                │   ├─ pending  = changed & (irq_fall & ~level)  falling edge
                │   ├─ pending |= changed & (irq_rise &  level)  rising edge
                │   ├─ pending |= irq_high &  level               level high
                │   ├─ pending |= irq_low  & ~level               level low
                │   ├─ pending &= isr & ier                       mask disabled
                │   │
                │   └─ for_each_set_bit(bit, &pending, 8):
                │         child_irq = irq_find_mapping(domain, base+bit)
                │         handle_nested_irq(child_irq)
                │               │
                │               ▼
                │         [consumer's IRQ handler runs]
                │
                └─ return IRQ_HANDLED
```

---

### Call Flow: تغيير الـ IRQ Mask (irqchip bus lock protocol)

```
kernel IRQ core wants to mask/unmask a child IRQ:
        │
        ├─► adnp_irq_bus_lock(d)
        │       └─ mutex_lock(&adnp->irq_lock)    ← take irq_lock
        │
        ├─► adnp_irq_mask(d)   OR   adnp_irq_unmask(d)
        │       │
        │       ├─ modify irq_enable[reg] shadow  (in RAM only)
        │       └─ gpiochip_disable/enable_irq()  (accounting)
        │          [NO I2C write yet — lazy update]
        │
        └─► adnp_irq_bus_sync_unlock(d)
                │
                ├─ scoped_guard(mutex, &i2c_lock)     ← take i2c_lock
                │   └─ for each reg:
                │       adnp_write(GPIO_IER + i, irq_enable[i])
                │              │
                │              ▼
                │        [I2C hardware — writes IER]
                │
                └─ mutex_unlock(&adnp->irq_lock)      ← release irq_lock
```

---

### استراتيجية الـ Locking

| الـ Lock | نوعه | بيحمي إيه | من مين |
|----------|------|-----------|--------|
| `i2c_lock` | `struct mutex` | كل عمليات `adnp_read` / `adnp_write` على الـ I2C bus | كل الـ GPIO ops + IRQ handler + bus_sync_unlock |
| `irq_lock` | `struct mutex` | الـ shadow registers (`irq_enable`) أثناء تسلسل mask/unmask | `bus_lock` → `irq_set_type` / `irq_mask` / `irq_unmask` → `bus_sync_unlock` |

**ترتيب الـ locks (Lock Ordering)**:

```
irq_lock  يُؤخذ أولاً  (في adnp_irq_bus_lock)
    │
    └─► i2c_lock  يُؤخذ تانياً  (في adnp_irq_bus_sync_unlock)
```

> مهم جداً: الـ `irq_lock` مش بيُؤخذ أبداً من جوا `adnp_irq()` (الـ threaded handler) — الـ handler بيأخذ `i2c_lock` بس. ده بيمنع deadlock لأن `bus_sync_unlock` بيأخذ `irq_lock` ثم `i2c_lock` بالترتيب ده دايماً.

**ليه الـ IRQ handler مش محتاج `irq_lock`؟**

الـ shadow registers (`irq_rise`, `irq_fall`, `irq_high`, `irq_low`, `irq_level`) بيتقراهم فقط في `adnp_irq()` — والكتابة عليهم بتحصل في `adnp_irq_set_type()` اللي بتتنادى تحت `irq_lock`. الـ kernel بيضمن إن `irq_set_type` مش بتتنادى وقت ما الـ handler شغّال (عن طريق الـ `irq_bus_lock` protocol)، فمفيش race.

---

### ملخص ASCII: البنية الكاملة للـ Driver

```
Device Tree Node "ad,gpio-adnp"
        │
        ▼
  i2c_driver.probe()
        │
        ▼
  struct adnp  ◄──── devm_kzalloc (managed memory)
  ┌────────────────────────────────────────────────┐
  │                                                │
  │  i2c_client ─────────────────► I2C Hardware   │
  │  (reads/writes via SMBus byte data protocol)   │
  │                                                │
  │  gpio_chip (embedded)                          │
  │  ├── ops: get/set/direction_*                  │
  │  └── gpio_irq_chip (embedded)                  │
  │      ├── irq_chip ops ────► adnp_irq_chip      │
  │      └── irq_domain ──────► child IRQ space    │
  │                              (0..ngpio-1)      │
  │                                                │
  │  Mutex: i2c_lock ──── guards I2C bus access    │
  │  Mutex: irq_lock ──── guards IER shadow regs   │
  │                                                │
  │  Shadow Buffers (one contiguous alloc):        │
  │  [enable|level|rise|fall|high|low] × num_regs  │
  └────────────────────────────────────────────────┘
        │
        ▼
  devm_gpiochip_add_data()
  → GPIOs exported to kernel GPIO subsystem
  → IRQ domain created for child IRQs
  → sysfs/debugfs entries created
```
## Phase 4: شرح الـ Functions

---

### ملخص الـ Functions والـ APIs — Cheatsheet

#### مجموعة: I2C Low-Level Access

| Function | Signature (مختصرة) | الغرض |
|---|---|---|
| `adnp_read` | `(adnp, offset, *value) → int` | قراءة بايت واحد من register عبر I2C SMBus |
| `adnp_write` | `(adnp, offset, value) → int` | كتابة بايت واحد في register عبر I2C SMBus |

#### مجموعة: GPIO Core Operations

| Function | Signature (مختصرة) | الغرض |
|---|---|---|
| `adnp_gpio_get` | `(chip, offset) → int` | قراءة مستوى pin من PLR |
| `__adnp_gpio_set` | `(adnp, offset, value) → int` | كتابة مستوى pin بدون lock (internal) |
| `adnp_gpio_set` | `(chip, offset, value) → int` | كتابة مستوى pin مع mutex |
| `adnp_gpio_direction_input` | `(chip, offset) → int` | ضبط pin كـ input في DDR |
| `adnp_gpio_direction_output` | `(chip, offset, value) → int` | ضبط pin كـ output في DDR وكتابة initial value |
| `adnp_gpio_dbg_show` | `(s, chip)` | طباعة حالة كل pins في debugfs |

#### مجموعة: IRQ Chip Callbacks

| Function | Signature (مختصرة) | الغرض |
|---|---|---|
| `adnp_irq` | `(irq, data) → irqreturn_t` | threaded IRQ handler الرئيسي |
| `adnp_irq_mask` | `(d)` | تعطيل interrupt لـ GPIO معين في shadow register |
| `adnp_irq_unmask` | `(d)` | تفعيل interrupt لـ GPIO معين في shadow register |
| `adnp_irq_set_type` | `(d, type) → int` | ضبط trigger type (edge/level) في shadow registers |
| `adnp_irq_bus_lock` | `(d)` | أخذ `irq_lock` mutex قبل تعديل IRQ config |
| `adnp_irq_bus_unlock` | `(d)` | flush shadow registers لـ hardware وإطلاق `irq_lock` |

#### مجموعة: Setup & Probe

| Function | Signature (مختصرة) | الغرض |
|---|---|---|
| `adnp_irq_setup` | `(adnp) → int` | تهيئة IRQ subsystem وتسجيل threaded IRQ |
| `adnp_gpio_setup` | `(adnp, num_gpios, is_irq_controller) → int` | تهيئة وتسجيل gpio_chip |
| `adnp_i2c_probe` | `(client) → int` | نقطة دخول driver عند detection الـ I2C device |

---

### Register Map للـ ADNP

```
Offset base = register_bank * (1 << reg_shift)

GPIO_DDR(adnp)  = 0x00 << reg_shift  → Data Direction Register (1=output, 0=input)
GPIO_PLR(adnp)  = 0x01 << reg_shift  → Pin Level Register (current logic level)
GPIO_IER(adnp)  = 0x02 << reg_shift  → Interrupt Enable Register
GPIO_ISR(adnp)  = 0x03 << reg_shift  → Interrupt Status Register
GPIO_PTR(adnp)  = 0x04 << reg_shift  → (Pin Type Register — unused in driver)
```

الـ `reg_shift` = `get_count_order(num_gpios) - 3`، يعني لـ 64 GPIO → reg_shift=3، فكل bank عنده 8 pins وفي 8 banks.

---

### Group 1: I2C Low-Level Access

الغرض من المجموعة: دي الطبقة الأدنى في الـ driver — كل تعامل مع hardware بيمر من هنا. بتغلف `i2c_smbus_read_byte_data` و`i2c_smbus_write_byte_data` وبتعمل error logging موحد.

---

#### `adnp_read`

```c
static int adnp_read(struct adnp *adnp, unsigned offset, uint8_t *value)
```

بتقرأ بايت واحد من register محدد في الـ I2C expander عبر SMBus protocol. لو الـ I2C transaction فشلت، بتطبع error message وبترجع الـ error code. لو نجحت، بتحط القيمة في `*value`.

**Parameters:**
- `adnp` — pointer للـ device state struct
- `offset` — register address في الـ I2C device (مش GPIO offset)
- `value` — output pointer، هيتحط فيه البايت المقروء

**Return:** `0` عند النجاح، أو negative errno من الـ SMBus layer

**Key Details:**
- مش بتاخد أي lock — المتصل مسؤول عن الـ `i2c_lock` mutex
- بتستخدم `i2c_smbus_read_byte_data` اللي بترجع `int` (القيمة أو error)، والكود بيفرق بينهم بالـ `err < 0`

---

#### `adnp_write`

```c
static int adnp_write(struct adnp *adnp, unsigned offset, uint8_t value)
```

بتكتب بايت واحد في register محدد في الـ I2C expander. عند الفشل بتطبع error وبترجع الـ error code.

**Parameters:**
- `adnp` — device state
- `offset` — register address
- `value` — البايت المطلوب كتابته

**Return:** `0` عند النجاح، negative errno عند الفشل

**Key Details:**
- زي `adnp_read`، مش بتاخد lock — المتصل مسؤول
- SMBus write_byte_data بترسل command byte (الـ offset) ثم data byte (الـ value)

---

### Group 2: GPIO Core Operations

الغرض من المجموعة: بتنفذ الـ `gpio_chip` ops المطلوبة من الـ GPIO subsystem. كل function بتترجم GPIO offset لـ register index وbit position باستخدام `reg_shift`.

**المعادلة الأساسية لكل العمليات:**
```c
reg = offset >> reg_shift;   // رقم الـ register bank
pos = offset & 7;            // رقم البت داخل الـ bank (0-7)
```

---

#### `adnp_gpio_get`

```c
static int adnp_gpio_get(struct gpio_chip *chip, unsigned offset)
```

بتقرأ المستوى المنطقي الحالي لـ GPIO pin معين من الـ PLR (Pin Level Register). بترجع 1 لو الـ pin high، 0 لو low.

**Parameters:**
- `chip` — الـ `gpio_chip` embedded في `struct adnp`
- `offset` — رقم الـ GPIO (0 إلى ngpio-1)

**Return:** `1` أو `0` عند النجاح، negative errno عند فشل I2C

**Key Details:**
- مش بتاخد `i2c_lock` — دي ممكن تتسبب في race مع write operations لو متصل من context خارج الـ lock. الـ caller المعتاد هو GPIO framework في sleeping context
- بتقرأ PLR مش DDR — بتجيب الـ pin level الفعلي بغض النظر عن الاتجاه

---

#### `__adnp_gpio_set`

```c
static int __adnp_gpio_set(struct adnp *adnp, unsigned int offset, int value)
```

الـ internal implementation لكتابة قيمة GPIO. بتعمل read-modify-write على PLR: بتقرأ البايت، بتعدل البت المطلوب، وبتكتب البايت تاني. الـ double underscore بتوضح إن المتصل لازم يمسك الـ `i2c_lock`.

**Parameters:**
- `adnp` — device state
- `offset` — GPIO number
- `value` — القيمة المطلوبة (non-zero = high, 0 = low)

**Return:** `0` عند النجاح، negative errno عند فشل I2C

**Key Details:**
- الـ read-modify-write غير atomic على مستوى الـ hardware — لازم الـ caller يحمي بـ mutex
- لو الـ I2C read فشلت، الـ write مش بتحصل

**Pseudocode:**
```
read PLR[reg] → val
if value: val |= BIT(pos)
else:     val &= ~BIT(pos)
write PLR[reg] ← val
```

---

#### `adnp_gpio_set`

```c
static int adnp_gpio_set(struct gpio_chip *chip, unsigned int offset, int value)
```

الـ public wrapper لـ `__adnp_gpio_set`. بتاخد الـ `i2c_lock` بـ `guard(mutex)` (scoped lock من `linux/cleanup.h`) عشان تضمن atomicity للـ read-modify-write، ثم بتستدعي الـ internal function.

**Parameters:**
- `chip` — gpio_chip pointer
- `offset` — GPIO number
- `value` — 0 أو non-zero

**Return:** نفس return من `__adnp_gpio_set`

**Key Details:**
- الـ `guard(mutex)` بيعمل unlock تلقائي لما الـ scope ينتهي (C cleanup attribute)
- `chip->can_sleep = true` لأن الـ I2C transactions ممكن تنام

---

#### `adnp_gpio_direction_input`

```c
static int adnp_gpio_direction_input(struct gpio_chip *chip, unsigned offset)
```

بتضبط GPIO pin كـ input عن طريق clear البت المناسب في DDR (Data Direction Register). بعد الكتابة، بتعمل readback للتحقق إن الـ hardware قبل التغيير — لو البت لسه set، معناه إن الـ pin مش ممكن يبقى input وبترجع `-EPERM`.

**Parameters:**
- `chip` — gpio_chip
- `offset` — GPIO number

**Return:** `0` عند النجاح، `-EPERM` لو الـ hardware رفض، negative errno لفشل I2C

**Key Details:**
- بتاخد `i2c_lock` لأن العملية read-modify-write
- الـ readback verification خطوة مهمة — بعض الـ pins ممكن تكون hardware-locked كـ output

**Pseudocode:**
```
lock i2c_lock
read DDR[reg] → value
value &= ~BIT(pos)          // clear the direction bit → input
write DDR[reg] ← value
read DDR[reg] → value       // verify
if (value & BIT(pos)): return -EPERM   // still output → reject
return 0
```

---

#### `adnp_gpio_direction_output`

```c
static int adnp_gpio_direction_output(struct gpio_chip *chip, unsigned offset,
                                      int value)
```

بتضبط GPIO pin كـ output وبتكتب الـ initial value. بتعمل read-modify-write على DDR لتفعيل الـ output direction، وبعد readback verification بتستدعي `__adnp_gpio_set` لكتابة الـ initial level.

**Parameters:**
- `chip` — gpio_chip
- `offset` — GPIO number
- `value` — الـ initial output level

**Return:** `0` عند النجاح، `-EPERM` لو الـ hardware رفض الـ output direction، negative errno لفشل I2C

**Key Details:**
- الـ `i2c_lock` ممسوك طول العملية كلها — الـ set بالـ `__adnp_gpio_set` بيحصل داخل الـ lock
- لو الـ set فشلت بعد الـ direction change، الـ direction تغيرت فعلاً لكن الـ value مش متكتب — لكن الكود مش بيرجع الـ error من الـ set (بيعمل `__adnp_gpio_set` بدون فحص return)

---

#### `adnp_gpio_dbg_show`

```c
static void adnp_gpio_dbg_show(struct seq_file *s, struct gpio_chip *chip)
```

بتطبع حالة كل الـ GPIO pins في debugfs (عادةً عبر `/sys/kernel/debug/gpio`). بتعرض لكل pin: الـ direction، الـ level، وحالة الـ interrupt (enabled/pending).

**Parameters:**
- `s` — seq_file للكتابة فيه
- `chip` — gpio_chip

**Key Details:**
- بتستخدم `scoped_guard(mutex, &adnp->i2c_lock)` عشان تقرأ DDR, PLR, IER, ISR بشكل آمن
- مسجلة بشرط `IS_ENABLED(CONFIG_DEBUG_FS)` — مش موجودة في production builds
- بتلف على كل الـ register banks وعلى كل البتات داخل كل bank

---

### Group 3: IRQ Chip Callbacks

الغرض من المجموعة: الـ ADNP ممكن يشتغل كـ interrupt controller ثانوي (nested IRQ controller). الـ interrupt management بيمشي على نمط **slow bus locking**: التغييرات بتحصل في shadow registers أولاً، وبس لما الـ irq_bus_sync_unlock يُستدعى بتتكتب على الـ hardware فعلاً.

**نموذج الـ shadow registers:**

```
adnp->irq_enable[bank]  ← shadow لـ IER hardware register
adnp->irq_level[bank]   ← آخر مستوى مقروء لكشف edge changes
adnp->irq_rise[bank]    ← mask: إيه الـ pins عندها rising edge trigger
adnp->irq_fall[bank]    ← mask: إيه الـ pins عندها falling edge trigger
adnp->irq_high[bank]    ← mask: إيه الـ pins عندها level-high trigger
adnp->irq_low[bank]     ← mask: إيه الـ pins عندها level-low trigger
```

---

#### `adnp_irq`

```c
static irqreturn_t adnp_irq(int irq, void *data)
```

الـ threaded IRQ handler الرئيسي. بيتشغل في thread context (مش hardirq) عشان يقدر ينام أثناء I2C transactions. بيقرأ PLR وISR وIER لكل bank، وبيحسب الـ pending interrupts (edge + level)، وبيستدعي `handle_nested_irq` لكل child IRQ متعلق.

**Parameters:**
- `irq` — رقم الـ parent IRQ
- `data` — pointer لـ `struct adnp`

**Return:** `IRQ_HANDLED` دايماً

**Key Details:**
- Edge detection بيتم software — الـ hardware مش بيعمل edge detection مباشرة
- بيحتاج `i2c_lock` عشان يقرأ الـ registers
- الـ `irq_level` shadow register بيتحدث في `adnp_irq_setup` فقط (initial read)، لكن في الـ handler نفسه مش بيتحدث — ده معناه إن الـ edge detection بتقارن الـ current level بالـ initial level المقروء عند الـ setup

**Pseudocode:**
```
for each register bank i:
    read PLR[i] → level
    read ISR[i] → isr
    read IER[i] → ier

    changed = level XOR irq_level[i]     // بتات تغيرت من آخر مرة
    pending = changed & (
        (irq_fall[i] & ~level) |         // كانت high، دلوقتي low → falling edge
        (irq_rise[i] & level)            // كانت low، دلوقتي high → rising edge
    )
    pending |= (irq_high[i] & level)     // level high trigger
    pending |= (irq_low[i] & ~level)     // level low trigger
    pending &= isr & ier                 // فلتر: active فقط، enabled فقط

    for each set bit in pending:
        child_irq = irq_find_mapping(domain, base + bit)
        handle_nested_irq(child_irq)

return IRQ_HANDLED
```

---

#### `adnp_irq_mask`

```c
static void adnp_irq_mask(struct irq_data *d)
```

بتعطل IRQ لـ GPIO pin معين عن طريق clear البت في `irq_enable` shadow register. التغيير الفعلي في الـ hardware بيحصل في `adnp_irq_bus_unlock`. كمان بتستدعي `gpiochip_disable_irq` للامتثال للـ irqchip immutable pattern.

**Parameters:**
- `d` — irq_data للـ child interrupt، `d->hwirq` = GPIO number

**Key Details:**
- بتُستدعى من داخل `irq_lock` (محمية بـ `irq_bus_lock`)
- الـ `IRQCHIP_IMMUTABLE` flag يتطلب استخدام `gpiochip_disable_irq`/`gpiochip_enable_irq` بدل تعديل references مباشرة

---

#### `adnp_irq_unmask`

```c
static void adnp_irq_unmask(struct irq_data *d)
```

عكس `adnp_irq_mask` — بتفعل IRQ عن طريق set البت في `irq_enable` shadow register وبتستدعي `gpiochip_enable_irq`.

**Parameters:**
- `d` — irq_data للـ child interrupt

**Key Details:**
- نفس الـ locking model — بتحصل داخل `irq_lock`
- الفرق عن mask: بتستدعي `gpiochip_enable_irq` أولاً ثم بتعدل الـ shadow register

---

#### `adnp_irq_set_type`

```c
static int adnp_irq_set_type(struct irq_data *d, unsigned int type)
```

بتضبط نوع الـ trigger للـ interrupt — edge (rising/falling) أو level (high/low). بتعدل الـ shadow registers `irq_rise`, `irq_fall`, `irq_high`, `irq_low` بناءً على الـ type bitmask.

**Parameters:**
- `d` — irq_data
- `type` — bitmask من `IRQ_TYPE_*` (مثلاً `IRQ_TYPE_EDGE_RISING`)

**Return:** `0` دايماً (كل combinations مدعومة)

**Key Details:**
- ممكن تشغل combinations مع بعض (مثلاً EDGE_RISING | EDGE_FALLING)
- التغييرات في shadow registers بس — الـ hardware مش بيتعمل له flush هنا

**كيف بيشتغل:**
```c
// EDGE_RISING مثلاً:
if (type & IRQ_TYPE_EDGE_RISING)
    adnp->irq_rise[reg] |=  BIT(pos);
else
    adnp->irq_rise[reg] &= ~BIT(pos);
// نفس الـ pattern لـ FALLING, HIGH, LOW
```

---

#### `adnp_irq_bus_lock`

```c
static void adnp_irq_bus_lock(struct irq_data *d)
```

بتاخد `adnp->irq_lock` mutex — هي الخطوة الأولى في الـ slow bus locking sequence. الـ IRQ framework بيستدعيها قبل أي sequence من mask/unmask/set_type operations عشان يضمن atomic update للـ hardware.

**Parameters:**
- `d` — irq_data

**Key Details:**
- الـ mutex بيتُمسك حتى يُستدعى `irq_bus_sync_unlock`
- فصل الـ irq_lock عن الـ i2c_lock ضروري لتجنب deadlock

---

#### `adnp_irq_bus_unlock`

```c
static void adnp_irq_bus_unlock(struct irq_data *d)
```

الخطوة الأخيرة في الـ slow bus sequence — بتكتب كل shadow registers الخاصة بـ IER على الـ hardware، ثم بتطلق الـ `irq_lock`. ده بيضمن إن التغييرات في mask/unmask بتتكتب على الـ I2C device فعلاً.

**Parameters:**
- `d` — irq_data

**Key Details:**
- بتاخد `i2c_lock` داخلياً (بـ `scoped_guard`) أثناء كتابة الـ IER registers
- بتعمل flush لكل الـ banks في loop واحدة
- الـ `irq_lock` بيتُطلق في الآخر (مش `scoped_guard` — manual `mutex_unlock`)

**Pseudocode:**
```
lock i2c_lock (scoped)
for i in 0..num_regs:
    write IER[i] ← irq_enable[i]   // sync shadow → hardware
unlock i2c_lock (automatic at scope end)
unlock irq_lock
```

---

### Group 4: Setup & Probe

الغرض من المجموعة: بتعمل initialization كاملة للـ driver من الـ device tree data — تخصيص الذاكرة، تسجيل الـ gpio_chip، وإعداد الـ interrupt subsystem.

---

#### `adnp_irq_setup`

```c
static int adnp_irq_setup(struct adnp *adnp)
```

بتهيئ كل المتعلقات بالـ IRQ subsystem: تخصيص shadow registers، قراءة initial pin levels، تعطيل كل الـ hardware interrupts، وتسجيل الـ threaded IRQ handler.

**Parameters:**
- `adnp` — device state (يجب إن `adnp->client` و `adnp->reg_shift` متهيأين)

**Return:** `0` عند النجاح، negative errno عند الفشل

**Key Details:**
- الـ shadow buffers بتتخصص في block واحد كبير بحجم `num_regs * 6` bytes، وبعدين الـ pointers بتتوزع
  - هذا يجعل الـ `devm_kcalloc` يطلب 6 sections: enable, level, rise, fall, high, low
- بتعمل initial read للـ PLR عشان تعرف الـ starting levels للـ edge detection
- بتكتب صفر في كل IER registers → كل الـ hardware interrupts معطلة في البداية
- الـ IRQ handler مسجل كـ `IRQF_ONESHOT | IRQF_TRIGGER_RISING` على الـ parent IRQ

**تخصيص الـ shadow buffers:**
```c
// block واحد بحجم num_regs * 6
adnp->irq_enable = devm_kcalloc(..., num_regs, 6, GFP_KERNEL);
adnp->irq_level  = adnp->irq_enable + num_regs * 1;
adnp->irq_rise   = adnp->irq_enable + num_regs * 2;
adnp->irq_fall   = adnp->irq_enable + num_regs * 3;
adnp->irq_high   = adnp->irq_enable + num_regs * 4;
adnp->irq_low    = adnp->irq_enable + num_regs * 5;
```

---

#### `adnp_gpio_setup`

```c
static int adnp_gpio_setup(struct adnp *adnp, unsigned int num_gpios,
                           bool is_irq_controller)
```

بتهيئ الـ `gpio_chip` struct وبتسجله في الـ GPIO framework. لو الـ device مسجل كـ interrupt controller، بتستدعي `adnp_irq_setup` وبتملي الـ `gpio_irq_chip` struct بالمعلومات المطلوبة.

**Parameters:**
- `adnp` — device state
- `num_gpios` — عدد الـ GPIO pins (من device tree)
- `is_irq_controller` — `true` لو الـ property `interrupt-controller` موجودة في DT

**Return:** `0` عند النجاح، negative errno عند الفشل

**Key Details:**
- `reg_shift = get_count_order(num_gpios) - 3` — يحسب عدد registers المطلوبة (كل register 8 pins)
  - مثال: 64 GPIO → `get_count_order(64)=6`, `reg_shift=3`, `num_regs=8`
- `chip->base = -1` → dynamic GPIO base assignment من الـ framework
- `chip->can_sleep = true` → ضروري لأن I2C operations قد تنام
- الـ `gpio_irq_chip` بيُعبأ بـ `girq->threaded = true` ومش بيحدد `parent_handler` — ده معناه إن الـ parent IRQ معالَج في الـ driver مش في الـ framework
- `devm_gpiochip_add_data` بيضمن cleanup تلقائي عند الـ device removal

**Pseudocode:**
```
reg_shift = get_count_order(num_gpios) - 3
setup gpio_chip ops (get, set, direction_input, direction_output, dbg_show)
chip->base = -1, chip->ngpio = num_gpios

if is_irq_controller:
    adnp_irq_setup(adnp)          // register threaded IRQ
    setup gpio_irq_chip:
        chip = &adnp_irq_chip
        parent_handler = NULL     // driver handles parent IRQ
        threaded = true
        handler = handle_simple_irq

devm_gpiochip_add_data(dev, chip, adnp)
```

---

#### `adnp_i2c_probe`

```c
static int adnp_i2c_probe(struct i2c_client *client)
```

نقطة دخول الـ driver — بيتُستدعى من I2C core لما يُكشف device متوافق. بتقرأ `nr-gpios` من device tree، بتخصص `struct adnp`، بتهيئ الـ `i2c_lock`، وبتستدعي `adnp_gpio_setup`.

**Parameters:**
- `client` — الـ I2C client الممثل للـ device

**Return:** `0` عند النجاح، negative errno عند الفشل

**Key Details:**
- كل الـ allocations بـ `devm_*` → cleanup تلقائي عند `device_del`
- `device_property_read_bool(dev, "interrupt-controller")` بيتحقق من الـ DT property للـ IRQ controller mode
- `i2c_set_clientdata` بيخزن الـ `adnp` pointer في الـ client عشان يُستخدم لاحقاً (مثلاً في suspend/resume)

**Pseudocode:**
```
read "nr-gpios" from device tree → num_gpios
devm_kzalloc(struct adnp)
devm_mutex_init(i2c_lock)
adnp->client = client
is_irq = device_property_read_bool("interrupt-controller")
adnp_gpio_setup(adnp, num_gpios, is_irq)
i2c_set_clientdata(client, adnp)
return 0
```

---

### ملاحظات معمارية مهمة

#### Locking Strategy

```
┌─────────────────────────────────────────────────────┐
│                  Locking Hierarchy                  │
├─────────────────┬───────────────────────────────────┤
│   irq_lock      │ يُمسك طول irq_bus_lock →          │
│                 │ irq_bus_sync_unlock sequence       │
├─────────────────┼───────────────────────────────────┤
│   i2c_lock      │ يُمسك لكل I2C transaction         │
│                 │ ممكن يُمسك داخل irq_lock          │
│                 │ (لكن العكس ممنوع → deadlock)      │
└─────────────────┴───────────────────────────────────┘
```

#### Software Edge Detection

الـ hardware بيوفر فقط ISR (interrupt status) بدون edge detection حقيقي. الـ driver بيعوض ده بـ:
1. حفظ آخر مستوى في `irq_level`
2. مقارنته بالـ current level في `adnp_irq`
3. تطبيق masks الـ rise/fall للكشف عن الـ edges

#### الـ IRQCHIP_IMMUTABLE Pattern

```c
static const struct irq_chip adnp_irq_chip = {
    .flags = IRQCHIP_IMMUTABLE,
    GPIOCHIP_IRQ_RESOURCE_HELPERS,  // macro بيضيف get_irq_chip_data وغيرها
    ...
};
```

الـ `IRQCHIP_IMMUTABLE` يمنع الـ framework من تعديل الـ irq_chip struct بعد التسجيل — ده متطلب في الـ kernels الحديثة لـ GPIO irqchips.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. debugfs — القراءة والتفسير

الـ driver بيسجّل `dbg_show` callback لما `CONFIG_DEBUG_FS` يكون مفعّل، وده بيظهر حالة كل pin في الـ GPIO expander.

```bash
# اعرف اسم الـ gpiochip الخاص بالـ adnp
ls /sys/bus/i2c/devices/0-0041/gpio/

# اقرأ الـ debug output (بيستدعي adnp_gpio_dbg_show داخلياً)
cat /sys/kernel/debug/gpio
```

**مثال على الـ output:**

```
GPIOs 496-559, i2c/0-0041, gpio-adnp:
  0: output high IRQ disabled
  1: input  low  IRQ enabled  pending
  2: input  low  IRQ disabled
  ...
```

**تفسير الأعمدة:**

| العمود | المعنى |
|--------|---------|
| الرقم | رقم الـ GPIO النسبي داخل الـ chip (0 → nr-gpios-1) |
| `output` / `input` | اتجاه الـ pin (من GPIO_DDR) |
| `high` / `low` | المستوى الحالي (من GPIO_PLR) |
| `IRQ enabled/disabled` | هل الـ interrupt مفعّل (من GPIO_IER) |
| `pending` | في interrupt معلّق (من GPIO_ISR) |

```bash
# عرض كل الـ GPIO chips والـ pins في النظام
cat /sys/kernel/debug/gpio | grep -A 70 "gpio-adnp"
```

---

#### 2. sysfs — المسارات المهمة

```bash
# اعرف الـ I2C address للجهاز (مثلاً 0x41 على bus 0)
ls /sys/bus/i2c/devices/

# معلومات الـ GPIO chip
cat /sys/bus/i2c/devices/0-0041/gpio/gpiochip496/label
cat /sys/bus/i2c/devices/0-0041/gpio/gpiochip496/ngpio
cat /sys/bus/i2c/devices/0-0041/gpio/gpiochip496/base

# تحكم في pin معين (مثلاً GPIO 500 = base 496 + offset 4)
echo 500 > /sys/class/gpio/export
cat /sys/class/gpio/gpio500/direction
cat /sys/class/gpio/gpio500/value
echo "out" > /sys/class/gpio/gpio500/direction
echo 1 > /sys/class/gpio/gpio500/value
echo 500 > /sys/class/gpio/unexport

# فحص الـ IRQ المرتبط بالـ chip
cat /sys/bus/i2c/devices/0-0041/gpio/gpiochip496/device/irq
```

---

#### 3. ftrace — الـ Tracepoints والـ Events

```bash
# mount tracefs لو مش موجود
mount -t tracefs nodev /sys/kernel/tracing

# تفعيل الـ I2C events لمتابعة كل read/write للـ ADNP registers
echo 1 > /sys/kernel/tracing/events/i2c/enable

# تفعيل GPIO events
echo 1 > /sys/kernel/tracing/events/gpio/enable

# تفعيل IRQ events لمتابعة adnp_irq handler
echo 1 > /sys/kernel/tracing/events/irq/irq_handler_entry/enable
echo 1 > /sys/kernel/tracing/events/irq/irq_handler_exit/enable

# تصفية على الـ i2c adapter رقم 0 فقط
echo "adapter_nr==0" > /sys/kernel/tracing/events/i2c/filter

# ابدأ الـ tracing
echo 1 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace_pipe

# إيقاف
echo 0 > /sys/kernel/tracing/tracing_on
```

**مثال على output مفيد:**

```
i2c-0: i2c_write: adapter=0, addr=0x41, flags=0x0, len=2
  data: 02 ff   ← كتابة 0xff على GPIO_IER (enable all IRQs)
i2c-0: i2c_read: adapter=0, addr=0x41, flags=0x1, len=1
  data: 03      ← قراءة GPIO_ISR
```

---

#### 4. printk / dynamic debug

```bash
# تفعيل dynamic debug لكل الـ driver
echo "module gpio_adnp +p" > /sys/kernel/debug/dynamic_debug/control

# أو تفعيل لملف معين فقط
echo "file gpio-adnp.c +pflmt" > /sys/kernel/debug/dynamic_debug/control

# مشاهدة الـ messages
dmesg -w | grep -i "gpio-adnp\|adnp\|0-0041"

# رفع مستوى الـ loglevel مؤقتاً
echo 8 > /proc/sys/kernel/printk
```

الـ driver بيستخدم `dev_err()` في `adnp_read()` و`adnp_write()` عند الفشل — كل فشل I2C هيظهر تلقائياً.

---

#### 5. Kernel Config — خيارات الـ Debugging

| Config | الغرض |
|--------|--------|
| `CONFIG_DEBUG_FS` | يفعّل `adnp_gpio_dbg_show` — **ضروري** |
| `CONFIG_GPIO_ADNP` | الـ driver نفسه |
| `CONFIG_I2C_DEBUG_CORE` | يطبع كل I2C transaction على مستوى الـ core |
| `CONFIG_I2C_DEBUG_BUS` | debug على مستوى الـ bus adapter |
| `CONFIG_GPIOLIB_IRQCHIP` | يفعّل دعم الـ IRQ chip داخل gpiolib |
| `CONFIG_GPIO_SYSFS` | يفعّل الـ sysfs interface للـ GPIO |
| `CONFIG_DEBUG_GPIO` | يضيف extra validation في gpiolib |
| `CONFIG_PROVE_LOCKING` | يكتشف deadlock محتمل بين `i2c_lock` و`irq_lock` |
| `CONFIG_LOCKDEP` | تتبع الـ lock ordering |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل `pr_debug` / `dev_dbg` ديناميكياً |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "CONFIG_(DEBUG_FS|GPIO_ADNP|I2C_DEBUG|DEBUG_GPIO|GPIO_SYSFS)"
```

---

#### 6. i2c-tools — أدوات خاصة بالـ Subsystem

```bash
# اكتشاف الجهاز على الـ I2C bus
i2cdetect -y 0

# قراءة مباشرة لكل الـ registers (جهاز عنده 8 GPIOs = reg_shift=0)
# GPIO_DDR = reg 0x00
i2cget -y 0 0x41 0x00
# GPIO_PLR = reg 0x01
i2cget -y 0 0x41 0x01
# GPIO_IER = reg 0x02
i2cget -y 0 0x41 0x02
# GPIO_ISR = reg 0x03
i2cget -y 0 0x41 0x03
# GPIO_PTR = reg 0x04
i2cget -y 0 0x41 0x04

# لو عندك 64 GPIO (reg_shift=3): الـ registers بتبدأ من 0x00 << 3 = 0x00
# GPIO_DDR base = 0x00, GPIO_PLR base = 0x08, GPIO_IER = 0x10, GPIO_ISR = 0x18
# قراءة PLR register الأول (8 GPIOs الأولى)
i2cget -y 0 0x41 0x08

# dump كامل للـ registers لجهاز 64-GPIO
i2cdump -y 0 0x41
```

**فهم الـ reg_shift:**

```
reg_shift = get_count_order(num_gpios) - 3

num_gpios=8   → reg_shift=0 → GPIO_DDR=0x00, GPIO_PLR=0x01
num_gpios=16  → reg_shift=1 → GPIO_DDR=0x00, GPIO_PLR=0x02
num_gpios=32  → reg_shift=2 → GPIO_DDR=0x00, GPIO_PLR=0x04
num_gpios=64  → reg_shift=3 → GPIO_DDR=0x00, GPIO_PLR=0x08
```

---

#### 7. رسائل الـ Error الشائعة

| رسالة في الـ dmesg | المعنى | الحل |
|--------------------|--------|-------|
| `i2c_smbus_read_byte_data() failed: -5` | فشل I2C read، الجهاز مش راد (I/O error) | تحقق من التوصيل الفيزيائي والـ pull-up resistors |
| `i2c_smbus_write_byte_data() failed: -5` | فشل I2C write | نفس الأسباب، أو الجهاز في reset |
| `i2c_smbus_read_byte_data() failed: -6` | ENXIO — الجهاز مش موجود على العنوان ده | تحقق من `reg` في الـ DT وتوصيل ADDR pins |
| `can't request IRQ#X: -22` | فشل طلب الـ IRQ، مشكلة في الـ DT interrupt specifier | تحقق من `interrupts` property في الـ DT |
| `gpio-adnp: direction_input failed: -1` | الـ DDR register مش بيتغيّر (EPERM) | pin محمي بالـ hardware، أو مشكلة في I2C |
| `gpio-adnp: direction_output failed: -1` | نفس السبب للـ output | فحص hardware strapping للـ pin |
| `devm_kcalloc failed` | نفاد الـ memory | نادر جداً، فحص `/proc/meminfo` |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في adnp_read() — تحقق من فشل I2C متكرر */
static int adnp_read(struct adnp *adnp, unsigned offset, uint8_t *value)
{
    int err;
    err = i2c_smbus_read_byte_data(adnp->client, offset);
    if (err < 0) {
        dev_err(adnp->gpio.parent, "%s failed: %d\n",
            "i2c_smbus_read_byte_data()", err);
        /* نقطة مناسبة لـ WARN_ON في debug builds */
        WARN_ONCE(err == -ETIMEDOUT, "I2C timeout on adnp read");
        return err;
    }
    *value = err;
    return 0;
}

/* في adnp_irq() — تحقق من صحة الـ IRQ mapping */
static irqreturn_t adnp_irq(int irq, void *data)
{
    ...
    for_each_set_bit(bit, &pending, 8) {
        unsigned int child_irq;
        child_irq = irq_find_mapping(adnp->gpio.irq.domain, base + bit);
        /* لو child_irq == 0 يبقى في مشكلة في الـ IRQ domain */
        WARN_ON(child_irq == 0);
        handle_nested_irq(child_irq);
    }
    ...
}

/* في adnp_irq_bus_unlock() — تحقق إن الـ IER اتكتب صح */
static void adnp_irq_bus_unlock(struct irq_data *d)
{
    ...
    for (i = 0; i < num_regs; i++) {
        int ret = adnp_write(adnp, GPIO_IER(adnp) + i, adnp->irq_enable[i]);
        WARN_ON(ret < 0); /* فشل كتابة الـ IER خطر على سلامة الـ IRQs */
    }
    ...
}
```

---

### Hardware Level

---

#### 1. مطابقة حالة الـ Hardware مع الـ Kernel

```bash
# اقرأ GPIO_DDR (Data Direction Register) من الـ hardware
i2cget -y 0 0x41 0x00
# قارنه مع ما بيقوله الـ kernel:
cat /sys/kernel/debug/gpio | grep "gpio-adnp"
# كل bit=1 في DDR = output, bit=0 = input

# اقرأ GPIO_PLR (Pin Level Register) — الحالة الفعلية للـ pins
i2cget -y 0 0x41 0x01
# لو pin configured كـ output وقيمة PLR مختلفة عن اللي كتبته → مشكلة hardware

# اقرأ GPIO_IER (Interrupt Enable Register)
i2cget -y 0 0x41 0x02

# اقرأ GPIO_ISR (Interrupt Status Register) — هل في interrupt معلّق؟
i2cget -y 0 0x41 0x03
# أي bit=1 يعني في interrupt pending على الـ pin ده

# GPIO_PTR (Interrupt Type Register) — على حسب datasheet الجهاز
i2cget -y 0 0x41 0x04
```

**Script للمقارنة الكاملة:**

```bash
#!/bin/bash
BUS=0
ADDR=0x41

echo "=== ADNP Hardware Register Dump ==="
echo "DDR (Direction): $(i2cget -y $BUS $ADDR 0x00)"
echo "PLR (Pin Level): $(i2cget -y $BUS $ADDR 0x01)"
echo "IER (IRQ Enable): $(i2cget -y $BUS $ADDR 0x02)"
echo "ISR (IRQ Status): $(i2cget -y $BUS $ADDR 0x03)"
echo ""
echo "=== Kernel GPIO State ==="
cat /sys/kernel/debug/gpio | grep -A 20 "gpio-adnp"
```

---

#### 2. Register Dump باستخدام i2c-tools

الـ ADNP بيتواصل عبر I2C فقط — مفيش memory-mapped registers — فـ `devmem2` و`/dev/mem` مش مناسبين هنا. الأداة الصح هي `i2ctransfer`:

```bash
# قراءة 5 registers دفعة واحدة (DDR, PLR, IER, ISR, PTR)
i2ctransfer -y 0 w1@0x41 0x00 r5@0x41

# مثال على output:
# 0xff 0x3c 0x00 0x00 0x00
# DDR=0xff (كل الـ pins outputs)
# PLR=0x3c (pins 2-5 high)
# IER=0x00 (كل الـ IRQs disabled)

# لو الجهاز عنده 64 GPIO (8 banks)، اقرأ كل bank:
for reg in 0x00 0x08 0x10 0x18; do
    echo -n "Base reg 0x$(printf '%02x' $reg): "
    i2ctransfer -y 0 w1@0x41 $reg r8@0x41
done
```

---

#### 3. Logic Analyzer / Oscilloscope

**نقاط القياس:**

```
SDA ──── I2C Data Line   (قس على connector الـ I2C)
SCL ──── I2C Clock Line  (عادة 100kHz أو 400kHz)
INT ──── خط الـ Interrupt (يوصل لـ GPIO 160 في مثال الـ DT)
```

**ما تبحث عنه على الـ Logic Analyzer:**

```
طبيعي:
─────────────────────────────────────────────────
CLK  ┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐  (نبضات منتظمة أثناء transactions)
SDA  ╔══╗═╗═══╔═╗═══╗   (START, ADDRESS, DATA, STOP)
INT  ──────────────┐└── (pulse عند حدوث interrupt)

مشكلة: SCL stretch مفرط
CLK  ┌┐┌_____________┐┌┐  (slave بيمسك SCL — slow device)
```

**إعدادات الـ Logic Analyzer:**
- Sample rate: 4 MHz على الأقل لـ I2C 400kHz
- Protocol decoder: I2C → فلتر على address 0x41
- Trigger: على START condition أو على falling edge لـ INT

---

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في الـ Kernel Log

| المشكلة | ما يظهر في الـ dmesg | السبب المحتمل |
|---------|----------------------|---------------|
| Pull-up مفقود على SDA/SCL | `i2c_smbus_read_byte_data() failed: -5` متكرر | مقاومة pull-up ناقصة أو قيمتها كبيرة جداً |
| الجهاز مش موجود | `0-0041: probe failed` مع `-6` | عنوان I2C غلط، أو VCC مش موصّل |
| INT line مش موصّل | الـ driver يعمل لكن الـ IRQs مش بتشتغل، `irq_enable` بيتكتب لكن مفيش interrupts | الـ GPIO 160 مش موصّل فعلاً بـ INT pin الجهاز |
| Noise على INT line | interrupts وهمية متكررة، ISR=0x00 رغم وجود IRQ | ضيف capacitor 100nF على INT line |
| Power supply unstable | `i2c_smbus_*() failed: -5` عشوائي | مشكلة decoupling، ضيف 100nF قريب من VCC |
| Address conflict | `i2cdetect` بيظهر جهازين على نفس العنوان | فحص ADDR pins strapping |

---

#### 5. Device Tree Debugging

```bash
# تحقق إن الـ DT اتحمّل صح
ls /proc/device-tree/
# أو
find /proc/device-tree -name "compatible" -exec grep -l "gpio-adnp" {} \;

# اقرأ الـ properties المهمة
DTPATH=$(find /proc/device-tree -name "compatible" | xargs grep -l "ad,gpio-adnp" 2>/dev/null | head -1 | xargs dirname)
echo "DT node: $DTPATH"
cat $DTPATH/compatible
cat $DTPATH/reg | xxd          # I2C address
cat $DTPATH/nr-gpios | xxd     # عدد الـ GPIOs (big-endian u32)
ls $DTPATH/                    # كل الـ properties الموجودة

# تحقق من interrupt specifier
cat $DTPATH/interrupts | xxd
# مثال: 00 00 00 a0  00 00 00 01 → GPIO 160, type EDGE_RISING

# تحقق إن الـ device اتـ probe بنجاح
cat /sys/bus/i2c/devices/0-0041/driver
# المتوقع: لينك لـ /sys/bus/i2c/drivers/gpio-adnp

# لو الـ probe فشل
dmesg | grep "0-0041\|gpio-adnp"
```

**أخطاء DT شائعة:**

```dts
/* خطأ: nr-gpios مش موجود → probe يفشل بـ -EINVAL */
gpioext: gpio-controller@41 {
    compatible = "ad,gpio-adnp";
    reg = <0x41>;
    /* nr-gpios ناقصة! */
};

/* صح */
gpioext: gpio-controller@41 {
    compatible = "ad,gpio-adnp";
    reg = <0x41>;
    interrupt-parent = <&gpio>;
    interrupts = <160 1>;
    gpio-controller;
    #gpio-cells = <2>;
    interrupt-controller;
    #interrupt-cells = <2>;
    nr-gpios = <64>;
};
```

```bash
# تحقق من الـ nr-gpios بشكل صريح
python3 -c "
import struct
with open('$DTPATH/nr-gpios', 'rb') as f:
    val = struct.unpack('>I', f.read(4))[0]
    print(f'nr-gpios = {val}')
    print(f'reg_shift = {val.bit_length() - 4}')
"
```

---

### Practical Commands

---

#### مجموعة أوامر جاهزة للنسخ

```bash
#!/bin/bash
# ====================================================
# ADNP GPIO Expander Debug Script
# الـ I2C bus رقم 0، الجهاز على عنوان 0x41
# ====================================================
BUS=0
ADDR=0x41

# 1. تأكد إن الجهاز موجود
echo "[1] I2C Detect:"
i2cdetect -y $BUS | grep -E "^[0-9]|UU|41"

# 2. اقرأ كل الـ registers الأساسية
echo -e "\n[2] Register Dump (8-GPIO device):"
echo "DDR (0x00): $(i2cget -y $BUS $ADDR 0x00 2>&1)"
echo "PLR (0x01): $(i2cget -y $BUS $ADDR 0x01 2>&1)"
echo "IER (0x02): $(i2cget -y $BUS $ADDR 0x02 2>&1)"
echo "ISR (0x03): $(i2cget -y $BUS $ADDR 0x03 2>&1)"
echo "PTR (0x04): $(i2cget -y $BUS $ADDR 0x04 2>&1)"

# 3. الـ Kernel GPIO state
echo -e "\n[3] Kernel GPIO State:"
cat /sys/kernel/debug/gpio 2>/dev/null | grep -A 30 "gpio-adnp" || \
    echo "debugfs غير متاح — فعّل CONFIG_DEBUG_FS"

# 4. IRQ info
echo -e "\n[4] IRQ Info:"
CHIP_DIR=$(find /sys/bus/i2c/devices/${BUS}-00${ADDR#0x}/gpio/ -maxdepth 1 -type d 2>/dev/null | head -1)
if [ -n "$CHIP_DIR" ]; then
    BASE=$(cat $CHIP_DIR/base)
    NGPIO=$(cat $CHIP_DIR/ngpio)
    echo "GPIO base=$BASE, ngpio=$NGPIO"
    grep "gpio-adnp\|adnp" /proc/interrupts
fi

# 5. فحص الـ driver binding
echo -e "\n[5] Driver Binding:"
ls -la /sys/bus/i2c/devices/${BUS}-00${ADDR#0x}/driver 2>/dev/null || \
    echo "الـ driver مش مربوط — فحص dmesg"

# 6. آخر رسائل الـ dmesg
echo -e "\n[6] Recent dmesg:"
dmesg | tail -20 | grep -i "gpio\|adnp\|i2c\|irq" || dmesg | tail -5
```

---

#### تفعيل I2C + GPIO tracing كامل

```bash
# تفعيل شامل
echo 1 > /sys/kernel/tracing/events/i2c/enable
echo 1 > /sys/kernel/tracing/events/gpio/enable
echo 1 > /sys/kernel/tracing/events/irq/irq_handler_entry/enable
echo 1 > /sys/kernel/tracing/tracing_on

# شغّل العملية اللي بتسبب المشكلة هنا...

# اقرأ النتيجة
cat /sys/kernel/tracing/trace | grep -E "i2c|gpio|adnp" | head -50

# إيقاف وتنظيف
echo 0 > /sys/kernel/tracing/tracing_on
echo > /sys/kernel/tracing/trace
```

**مثال على output وتفسيره:**

```
# مثال: كتابة IER register عند enable interrupt
     kworker/0:1-45  [000] .....  123.456: i2c_write: i2c-0 #0 a=041 f=0000 l=2 [02][ff]
#                                                                 ↑addr  ↑flags      ↑reg ↑value
# reg=0x02 = IER، value=0xff = كل الـ 8 interrupts enabled

# مثال: قراءة ISR عند وقوع interrupt
     irq/5-gpio-adnp-78 [000] .....  123.789: i2c_read: i2c-0 #0 a=041 f=0001 l=1 [01]
# الـ thread name "irq/5-gpio-adnp" يؤكد إن adnp_irq() بيشتغل
```

---

#### فحص الـ IRQ domain

```bash
# اعرف الـ IRQ numbers المرتبطة بالـ ADNP
cat /proc/interrupts | grep -i "adnp\|gpio-adnp"

# مثال على output:
# 160:  0  0  GPIO  160 Edge  gpio-adnp   ← الـ parent IRQ
# 496:  0  0  gpio-adnp  0 Edge  some-consumer ← child IRQ

# اعرف كل الـ IRQ domains
cat /sys/kernel/debug/irq/domains/*/name 2>/dev/null | grep -i gpio
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على AM62x — GPIO Expander مش شايل الـ Interrupt

#### العنوان
الـ ADNP GPIO expander مش بيطلع interrupt رغم إن الـ hardware شغال تمام

#### السياق
بتعمل bring-up لـ industrial gateway مبني على **Texas Instruments AM62x**. الـ gateway عنده ADNP expander متصل على I2C bus، بيتحكم في 64 GPIO — منهم digital inputs من sensors صناعية. الـ interrupt line من الـ expander راحت على GPIO pin في الـ SoC.

#### المشكلة
الـ sensor بيعمل trigger على الـ GPIO expander، بس الـ Linux driver مش بيستقبل الـ interrupt. الـ /proc/interrupts مش بيعد أي شيء على الـ line دي.

#### التحليل
بتراجع الـ DT node:

```dts
gpioext: gpio-controller@41 {
    compatible = "ad,gpio-adnp";
    reg = <0x41>;

    interrupt-parent = <&main_gpio0>;
    interrupts = <42 IRQ_TYPE_LEVEL_HIGH>;  /* غلط! */

    gpio-controller;
    #gpio-cells = <2>;

    interrupt-controller;
    #interrupt-cells = <2>;

    nr-gpios = <64>;
};
```

الـ binding بيقول إن الـ `interrupts` property لازم تتبع الـ two-cell specifier حسب `interrupts.txt`، بس الـ interrupt type مش متحدد صح حسب الـ hardware — الـ ADNP بيطلع interrupt كـ active-low open-drain، والـ DT قايله `LEVEL_HIGH`.

بتراجع الـ driver في `drivers/gpio/gpio-adnp.c`:

```c
static int adnp_irq_setup(struct adnp *adnp)
{
    /* الـ driver بيستخدم الـ interrupt اللي جاي من DT كما هو */
    adnp->irq = client->irq;  /* client->irq اتبنى من الـ DT */
    ...
    err = request_threaded_irq(adnp->irq, NULL, adnp_irq,
                               IRQF_ONESHOT | IRQF_SHARED,
                               dev_name(&client->dev), adnp);
}
```

الـ `client->irq` اتحسب من الـ DT بالـ type الغلط، فالـ interrupt controller في الـ SoC مش بيتriggered.

#### الحل
تعديل الـ DT:

```dts
interrupt-parent = <&main_gpio0>;
interrupts = <42 IRQ_TYPE_LEVEL_LOW>;  /* active-low زي الـ hardware */
```

وبعدين verify بـ:

```bash
# تأكد إن الـ interrupt اتسجل صح
cat /proc/interrupts | grep adnp

# اختبر يدوي
gpioset gpiochip1 0=1
cat /proc/interrupts | grep adnp
```

#### الدرس المستفاد
الـ `interrupts` property في الـ ADNP binding مش بتحدد الـ polarity — ده مسؤولية الـ hardware team يوثق الـ schematic صح وتترجمه للـ `IRQ_TYPE_*` الصح في الـ DT.

---

### السيناريو 2: Android TV Box على Allwinner H616 — عدد الـ GPIOs غلط

#### العنوان
الـ `nr-gpios` property بقيمة غلط بتسبب kernel panic وقت الـ boot

#### السياق
شركة بتعمل Android TV box على **Allwinner H616**. الـ hardware designer حط ADNP expander بـ 32 GPIO بس، ووصّله على I2C-2. الـ DT اتكتب بسرعة ونُسخ من reference design تاني.

#### المشكلة
الـ board بتيجي تـ boot وبعدين kernel panic بـ:

```
gpio-adnp 1-0041: invalid register 4
Kernel panic - not syncing: gpiolib: don't claim nonexistent GPIO
```

#### التحليل
الـ DT المنسوخ فيه:

```dts
gpioext: gpio-controller@41 {
    compatible = "ad,gpio-adnp";
    reg = <0x41>;
    interrupt-parent = <&pio>;
    interrupts = <1 10 IRQ_TYPE_EDGE_FALLING>;
    gpio-controller;
    #gpio-cells = <2>;
    nr-gpios = <64>;   /* غلط! الـ hardware عنده 32 بس */
};
```

الـ driver بيحسب عدد الـ registers بناءً على `nr-gpios`:

```c
/* من gpio-adnp.c */
adnp->num_regs = DIV_ROUND_UP(nr_gpios, 8);
/* لو nr-gpios = 64  →  num_regs = 8 */
/* بس الـ chip الفعلي عنده 4 registers بس (32 GPIO) */
```

لما الـ driver يحاول يقرأ register رقم 4 أو أعلى، الـ I2C بيرجع error، وبعدها الـ gpio subsystem بيـ panic لأنه شايف GPIOs معرّفة بس مش accessible.

#### الحل

```dts
nr-gpios = <32>;   /* صح — يطابق الـ hardware الفعلي */
```

وللتحقق:

```bash
# بعد الـ boot
gpioinfo | grep -A 35 "gpio-adnp"
# لازم تشوف 32 line بس
```

#### الدرس المستفاد
الـ `nr-gpios` property في الـ ADNP binding **لازم يطابق الـ hardware chip بالظبط** — مش قيمة افتراضية ولا تخمين. دايمًا راجع الـ datasheet للـ chip المستخدم وحدد القيمة منه مش من الـ reference design.

---

### السيناريو 3: IoT Sensor Node على STM32MP1 — تعارض الـ I2C Address

#### العنوان
جهازين ADNP على نفس الـ I2C bus بنفس الـ address بيسبب corruption في الـ GPIO state

#### السياق
نظام IoT صناعي على **STM32MP1** بيستخدم **اتنين** ADNP expanders على نفس الـ I2C-1 bus — الأول بيتحكم في outputs، والتاني inputs من sensors. الـ hardware designer نسي يغير الـ address strapping على الـ PCB.

#### المشكلة
الـ GPIO values بتتغير عشوائيًا، والـ outputs بتتقلب من غير سبب. في أوقات الـ system بيـ lock up.

#### التحليل
الـ DT فيه:

```dts
/* الأول */
gpioext0: gpio-controller@41 {
    compatible = "ad,gpio-adnp";
    reg = <0x41>;
    nr-gpios = <64>;
    ...
};

/* التاني — نفس الـ address! */
gpioext1: gpio-controller@41 {
    compatible = "ad,gpio-adnp";
    reg = <0x41>;   /* collision! */
    nr-gpios = <64>;
    ...
};
```

الـ Linux I2C subsystem بيحاول يـ register اتنين devices على نفس الـ address، فبعض الـ transactions بتروح للغلط chip. الـ driver ما عندوش protection من الـ address collision على مستوى الـ DT.

بتتحقق:

```bash
i2cdetect -y 1
# هتشوف 41 بس مرة واحدة — الـ bus ما بيشوفش تعارض
dmesg | grep "i2c"
# هتلاقي warning أو error في التسجيل
```

#### الحل
لازم تغير الـ hardware — تعديل الـ address strapping resistors على الـ PCB للـ chip التاني على `0x42`، وبعدين تعدل الـ DT:

```dts
gpioext1: gpio-controller@42 {
    compatible = "ad,gpio-adnp";
    reg = <0x42>;   /* عنوان مختلف */
    nr-gpios = <64>;
    ...
};
```

وللتأكد:

```bash
i2cdetect -y 1
# هتشوف 41 و 42 كل واحد لوحده
i2cget -y 1 0x41 0x00
i2cget -y 1 0x42 0x00
```

#### الدرس المستفاد
الـ `reg` property في الـ ADNP binding هو الـ I2C slave address الفعلي المحدد بالـ hardware pins. لو محتاج أكتر من device على نفس الـ bus، لازم تضمن إن كل device عنده address مختلف من خلال الـ hardware strapping قبل ما تكتب الـ DT.

---

### السيناريو 4: Automotive ECU على i.MX8 — الـ Expander كـ Interrupt Controller

#### العنوان
الـ ADNP مستخدم كـ interrupt controller للـ CAN transceivers، بس الـ interrupts مش بتتوصل للـ drivers

#### السياق
ECU في سيارة مبني على **NXP i.MX8QM**. عنده ADNP expander بـ 64 GPIO، منهم 8 GPIOs متصلين بـ interrupt outputs من CAN transceivers مختلفة. محتاجين يستخدموا الـ expander كـ interrupt controller فوق I2C.

#### المشكلة
الـ CAN driver بيـ request interrupt بس ما بيستقبلش أي notification. الـ `request_irq` بيرجع 0 (success) بس الـ handler ما بيتcallش أبدًا.

#### التحليل
الـ DT للـ expander:

```dts
gpioext: gpio-controller@41 {
    compatible = "ad,gpio-adnp";
    reg = <0x41>;

    interrupt-parent = <&lsio_gpio1>;
    interrupts = <5 IRQ_TYPE_LEVEL_LOW>;

    gpio-controller;
    #gpio-cells = <2>;

    /* ده المهم — بيعلن إنه interrupt controller */
    interrupt-controller;
    #interrupt-cells = <2>;

    nr-gpios = <64>;
};
```

الـ CAN node:

```dts
can0: can@... {
    ...
    /* محاولة ربط الـ interrupt بالـ expander */
    interrupt-parent = <&gpioext>;
    interrupts = <10 IRQ_TYPE_EDGE_FALLING>;  /* GPIO 10 في الـ expander */
};
```

المشكلة: الـ ADNP driver بيـ implement الـ interrupt controller functionality، بس الـ GPIO pin المستخدم (10) لازم يتعمله configure كـ input في الـ expander أولًا. الـ DT مش بيحدد direction — ده بيتعمل من الـ driver اللي بيـ request الـ interrupt.

بتراجع الـ kernel log:

```bash
dmesg | grep -i "adnp\|gpio-adnp"
# هتشوف إيه بالظبط بيتسجل
cat /sys/kernel/debug/gpio
# تحقق من state الـ GPIO 10
```

اتضح إن الـ GPIO 10 اتعمله configure كـ output بغلطة من boot script قديم.

#### الحل

```bash
# تأكد إن الـ GPIO مش متاخد من حتة تانية
gpioinfo | grep "line 10"

# لو الـ boot script بيعمله output، احذف السطر ده منه
# أو أضف في الـ DT explicit direction hint لو الـ driver بيدعمه
```

وفي الـ CAN driver:

```c
/* الـ driver لازم يعمل gpio_direction_input قبل request_irq */
ret = gpio_direction_input(gpio_num);
if (ret)
    return ret;
ret = request_irq(irq, can_irq_handler, IRQF_TRIGGER_FALLING, ...);
```

#### الدرس المستفاد
لما بتستخدم الـ ADNP كـ interrupt controller (بالـ `interrupt-controller` و `#interrupt-cells = <2>` في الـ DT)، الـ GPIO pins اللي بتستخدمها كـ interrupt sources لازم تكون configured كـ inputs. الـ DT binding مش بيضمن الـ direction — ده مسؤولية الـ consumer driver أو الـ board initialization.

---

### السيناريو 5: Custom Board Bring-up على RK3562 — الـ Expander مش بيتـ Detect

#### العنوان
الـ `gpio-adnp` driver ما بيـ probe-ش رغم إن الـ DT صح ظاهريًا

#### السياق
فريق bring-up بيشتغل على custom board مبني على **Rockchip RK3562** لنظام POS (Point of Sale). الـ ADNP expander متصل على I2C-3 بالـ address `0x41`. الـ DT اتكتب بالظبط زي الـ example في الـ binding file.

#### المشكلة
الـ `dmesg` ما بيظهرش أي رسالة من الـ `gpio-adnp` driver، والـ `/sys/bus/i2c/devices/` ما بيظهرش الـ device خالص.

#### التحليل
أول خطوة — تتحقق إن الـ I2C device موجود على الـ bus:

```bash
i2cdetect -y 3
# لو الـ 41 ما ظهرش — مشكلة hardware أو power
# لو ظهر — مشكلة في الـ DT أو الـ driver binding
```

النتيجة: `41` مش ظاهر في الـ `i2cdetect`.

بتفحص الـ schematic — اتضح إن الـ ADNP chip محتاج 3.3V على الـ VCC pin، بس الـ power rail ده مش شغال قبل ما يعدي 500ms من الـ boot. الـ I2C controller بيحاول يـ scan قبل الـ chip يصحى.

بتراجع الـ DT:

```dts
gpioext: gpio-controller@41 {
    compatible = "ad,gpio-adnp";
    reg = <0x41>;
    interrupt-parent = <&gpio3>;
    interrupts = <15 IRQ_TYPE_LEVEL_LOW>;
    gpio-controller;
    #gpio-cells = <2>;
    interrupt-controller;
    #interrupt-cells = <2>;
    nr-gpios = <64>;
    /* مفيش أي power sequencing! */
};
```

#### الحل
إضافة regulator dependency في الـ DT لو في regulator بيتحكم في الـ VCC:

```dts
vcc_gpio_exp: regulator-gpio-exp {
    compatible = "regulator-fixed";
    regulator-name = "vcc-gpio-exp";
    regulator-min-microvolt = <3300000>;
    regulator-max-microvolt = <3300000>;
    enable-active-high;
    gpio = <&gpio0 5 GPIO_ACTIVE_HIGH>;
    startup-delay-us = <10000>;  /* 10ms delay بعد التشغيل */
};

gpioext: gpio-controller@41 {
    compatible = "ad,gpio-adnp";
    reg = <0x41>;
    interrupt-parent = <&gpio3>;
    interrupts = <15 IRQ_TYPE_LEVEL_LOW>;
    gpio-controller;
    #gpio-cells = <2>;
    interrupt-controller;
    #interrupt-cells = <2>;
    nr-gpios = <64>;
    vcc-supply = <&vcc_gpio_exp>;  /* ربط الـ power */
};
```

لو الـ power مش controllable، الحل يكون تأخير الـ probe:

```bash
# في الـ boot script، أضف delay قبل ما الـ driver يـ load
# أو استخدم deferred probe mechanism في الـ kernel
```

للتحقق بعد الحل:

```bash
dmesg | grep "gpio-adnp"
# المفروض تشوف:
# gpio-adnp 3-0041: probing N-bit GPIO expander
# gpio-adnp 3-0041: registered N GPIOs

ls /sys/bus/i2c/devices/3-0041/
gpioinfo | grep -A 65 "gpio-adnp"
```

#### الدرس المستفاد
الـ ADNP binding مش بيـ mention أي power supply properties، بس في الواقع الـ chip لازم يكون powered قبل الـ I2C probe. لو في power sequencing requirements على الـ board، لازم تتعامل معاها على مستوى الـ DT بالـ regulators أو على مستوى الـ boot sequence — وده بيتكشف بس في الـ bring-up الفعلي مش في الـ simulation.
## Phase 7: مصادر ومراجع

### توثيق Kernel الرسمي

| المصدر | الوصف |
|--------|-------|
| [`Documentation/devicetree/bindings/gpio/gpio-adnp.txt`](https://www.kernel.org/doc/Documentation/devicetree/bindings/gpio/gpio-adnp.txt) | الـ binding الرسمي لـ Avionic Design N-bit GPIO expander |
| [`Documentation/driver-api/gpio/index.rst`](https://docs.kernel.org/driver-api/gpio/index.html) | نقطة البداية لكل حاجة تخص الـ GPIO subsystem في الـ kernel |
| [`Documentation/driver-api/gpio/driver.rst`](https://docs.kernel.org/driver-api/gpio/driver.html) | واجهة كتابة الـ GPIO driver — الـ `gpio_chip` struct وكل الـ callbacks |
| [`Documentation/driver-api/gpio/consumer.rst`](https://docs.kernel.org/driver-api/gpio/consumer.html) | الـ descriptor-based API اللي بتستخدمه الـ drivers اللي بتستهلك GPIOs |
| [`Documentation/driver-api/gpio/board.rst`](https://docs.kernel.org/driver-api/gpio/board.html) | الـ GPIO mappings وازاي تعمل ربط بين الـ device tree والـ driver |
| [`Documentation/driver-api/gpio/legacy.rst`](https://static.lwn.net/kerneldoc/driver-api/gpio/legacy.html) | الـ legacy integer-based API — مهم للفهم التاريخي |
| [`Documentation/driver-api/gpio/intro.rst`](https://static.lwn.net/kerneldoc/driver-api/gpio/intro.html) | مقدمة رسمية للـ GPIO framework |

---

### مقالات LWN.net

دي أهم المقالات اللي بتشرح الـ GPIO framework وسياق الـ gpio-adnp driver:

- **[GPIO in the kernel: an introduction](https://lwn.net/Articles/532714/)** — شرح تفصيلي للـ gpiolib framework، الـ `gpio_chip` struct، والـ descriptor-based API. أهم مقال للبدء.

- **[Documentation: gpiolib: document new interface](https://lwn.net/Articles/574055/)** — patch بيوثق الـ new gpiod_ interface اللي استبدلت الـ legacy integer API.

- **[GPIO implementation framework](https://lwn.net/Articles/256461/)** — المقال الأصلي من 2008 اللي عرّف الـ gpiolib concept لأول مرة في الـ kernel.

- **[gpio: fxl6408: add I2C GPIO expander driver](https://lwn.net/Articles/926010/)** — مثال حديث لـ I2C GPIO expander driver مشابه للـ gpio-adnp، مفيد للمقارنة.

- **[Introduce STMFX I2C GPIO expander](https://lwn.net/Articles/751525/)** — driver تاني لـ I2C GPIO expander يوضح الـ pattern المتكرر.

- **[pinctrl: Add driver for Awinic AW9523/B I2C GPIO Expander](https://lwn.net/Articles/842681/)** — I2C GPIO expander مع pinctrl integration.

- **[GPIO — The Linux Kernel documentation (LWN mirror)](https://static.lwn.net/kerneldoc/admin-guide/gpio/index.html)** — mirror للتوثيق الرسمي على LWN.

---

### Kernel Commits المهمة

الـ commits دي في الـ repository هي التاريخ الكامل للـ `drivers/gpio/gpio-adnp.c`:

| الـ Commit Hash | الوصف |
|----------------|-------|
| `5e969a401a01` | **الـ commit الأصلي** — `gpio: Add Avionic Design N-bit GPIO expander support` |
| `565a0e9ab813` | `gpio: adnp: Use irqchip template` |
| `8773bacefcd7` | `gpio: adnp: Make use of device properties` |
| `d4c0cf340861` | `gpio: adnp: Convert to immutable irq_chip` |
| `0dfce460fe2e` | `gpio: adnp: use devm_mutex_init()` |
| `c7fe19ed3973` | `gpio: adnp: use lock guards for the I2C lock` |
| `21c853ad9309` | `gpio: adnp: use new line value setter callbacks` |
| `d9d87d90cc0b` | `treewide: rename GPIO set callbacks back to their original names` |

لتصفح الـ commits على GitHub:
```
https://github.com/torvalds/linux/commits/master/drivers/gpio/gpio-adnp.c
```

---

### Mailing List

الـ linux-gpio mailing list هي المكان الرسمي لنقاشات الـ GPIO drivers:

- **الأرشيف على GitHub**: [linux-mailinglist-archives/linux-gpio.vger.kernel.org.0](https://github.com/linux-mailinglist-archives/linux-gpio.vger.kernel.org.0)
- **الـ LKML**: [lkml.org](https://lkml.org/) — ابحث عن `gpio-adnp` أو `Avionic Design GPIO`
- **الـ linux-gpio list**: `linux-gpio@vger.kernel.org`
- **مثال على نقاش الـ MAINTAINERS**: [PATCH: add linux-gpio mailing list](https://linux.kernel.narkive.com/jlZHi26J/patch-maintainers-add-linux-gpio-mailing-list)

---

### kernelnewbies.org

- **[Using GPIO from device tree on platform devices](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2015-August/014911.html)** — نقاش عملي عن ربط الـ GPIO بالـ device tree
- **[devm_gpiod_get usage to get GPIO num](http://lists.kernelnewbies.org/pipermail/kernelnewbies/2016-August/016707.html)** — شرح لاستخدام الـ descriptor-based API
- **[LinuxChanges](https://kernelnewbies.org/LinuxChanges)** — متابعة التغييرات في الـ GPIO subsystem عبر الإصدارات

---

### elinux.org

- **[BoardBringUp-i2c](https://elinux.org/BoardBringUp-i2c)** — دليل عملي لـ bring-up الـ I2C GPIO expanders على embedded boards
- **[EBC Exercise 12 I2C](https://elinux.org/EBC_Exercise_12_I2C_-_xM)** — تمرين عملي على I2C devices مع Linux

---

### مصادر إضافية

- **[CONFIG_GPIO_ADNP — Linux Kernel Driver DataBase](https://cateee.net/lkddb/web-lkddb/GPIO_ADNP.html)** — معلومات الـ Kconfig الخاصة بالـ driver
- **[Avionic Design linux-l4t repository](https://github.com/avionic-design/linux-l4t)** — الـ repository الأصلي من الشركة اللي طورت الـ driver
- **[gpiolib.c على GitHub](https://github.com/torvalds/linux/blob/master/drivers/gpio/gpiolib.c)** — الـ implementation الأساسية للـ GPIO core

---

### كتب مرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 12**: Memory Technology Devices — يشمل مبادئ الـ I2C bus
- **الفصل 6**: Advanced Char Driver Operations — الـ interrupt handling المستخدم في الـ gpio-adnp
- متاح مجاناً: [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 7**: Interrupts and Interrupt Handlers — أساسي لفهم الـ `irq_chip` في الـ gpio-adnp
- **الفصل 5**: System Calls — فهم الـ kernel/userspace boundary للـ GPIO sysfs

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 15**: Debugging Embedded Linux Applications — debugging الـ I2C GPIO expanders
- **الفصل 8**: Device Drivers — مقدمة للـ driver model المستخدم في الـ gpio-adnp

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- **الفصل 7**: Modules — كيفية تحميل الـ gpio-adnp كـ kernel module

---

### Search Terms للبحث عن معلومات أكثر

```
linux kernel gpio expander i2c irqchip
gpio_chip struct implementation linux
gpiolib irq domain nested threaded
devm_gpiochip_add_data linux kernel
"gpio-adnp" OR "ad,gpio-adnp" linux
gpio irq_chip immutable linux kernel
i2c_smbus_read_byte_data gpio driver
linux device tree gpio-cells interrupt-controller
```
## Phase 8: Writing simple module

### الفكرة

الـ driver بتاع `gpio-adnp` بيسجّل I2C driver عن طريق `module_i2c_driver()`. أكتر function مناسبة للـ hook هي `adnp_i2c_probe` — دي بيتعملها call في كل مرة الـ kernel يشوف device متوافق مع `"ad,gpio-adnp"`. هنستخدم **kprobe** عشان نعترض الـ `adnp_i2c_probe` ونطبع معلومات الـ I2C client اللي اتـ probe عليه.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_adnp.c — hook adnp_i2c_probe via kprobe
 * Prints I2C address and IRQ info whenever an ADNP GPIO expander is probed.
 */

/* kprobe API */
#include <linux/kprobes.h>
/* pr_info / pr_err */
#include <linux/kernel.h>
/* module_init / module_exit / MODULE_* */
#include <linux/module.h>
/* struct i2c_client definition */
#include <linux/i2c.h>

/* ------------------------------------------------------------------ */
/* pre-handler: called BEFORE adnp_i2c_probe executes                  */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * adnp_i2c_probe signature:
     *   static int adnp_i2c_probe(struct i2c_client *client)
     *
     * On x86-64  → first arg in RDI
     * On ARM64   → first arg in X0
     * pt_regs_param() macro hides the arch difference.
     */
#ifdef CONFIG_X86_64
    struct i2c_client *client = (struct i2c_client *)regs->di;
#elif defined(CONFIG_ARM64)
    struct i2c_client *client = (struct i2c_client *)regs->regs[0];
#else
    /* fallback — will not compile cleanly on unsupported arches */
    struct i2c_client *client = NULL;
#endif

    if (!client)
        return 0;

    /* Print the I2C slave address (7-bit) and the IRQ line assigned */
    pr_info("[kprobe_adnp] adnp_i2c_probe called: "
            "I2C addr=0x%02x  irq=%d  adapter=\"%s\"\n",
            client->addr,
            client->irq,
            client->adapter ? client->adapter->name : "unknown");

    return 0; /* 0 → continue normal execution of the probed function */
}

/* ------------------------------------------------------------------ */
/* post-handler: called AFTER adnp_i2c_probe returns                   */
/* ------------------------------------------------------------------ */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /* regs->ax (x86-64) holds the return value after the call */
#ifdef CONFIG_X86_64
    long ret = (long)regs->ax;
#elif defined(CONFIG_ARM64)
    long ret = (long)regs->regs[0];
#else
    long ret = 0;
#endif
    pr_info("[kprobe_adnp] adnp_i2c_probe returned: %ld (%s)\n",
            ret, ret == 0 ? "OK" : "ERROR");
}

/* ------------------------------------------------------------------ */
/* kprobe struct — symbol_name points to the function we want to hook  */
/* ------------------------------------------------------------------ */
static struct kprobe adnp_probe = {
    .symbol_name = "adnp_i2c_probe", /* exact kernel symbol name */
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

/* ------------------------------------------------------------------ */
/* module_init                                                          */
/* ------------------------------------------------------------------ */
static int __init kprobe_adnp_init(void)
{
    int ret;

    ret = register_kprobe(&adnp_probe);
    if (ret < 0) {
        pr_err("[kprobe_adnp] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[kprobe_adnp] planted kprobe on \"%s\" at %px\n",
            adnp_probe.symbol_name, adnp_probe.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit                                                          */
/* ------------------------------------------------------------------ */
static void __exit kprobe_adnp_exit(void)
{
    unregister_kprobe(&adnp_probe);
    pr_info("[kprobe_adnp] kprobe removed from \"%s\"\n",
            adnp_probe.symbol_name);
}

module_init(kprobe_adnp_init);
module_exit(kprobe_adnp_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe hook on adnp_i2c_probe — GPIO ADNP expander tracer");
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `<linux/kprobes.h>` | فيه `struct kprobe` وكل الـ API الخاص بالـ kprobes |
| `<linux/kernel.h>` | فيه `pr_info` و `pr_err` اللي بنستخدمهم للطباعة |
| `<linux/module.h>` | لازم لأي kernel module — فيه `module_init`/`module_exit` والـ macros |
| `<linux/i2c.h>` | فيه تعريف `struct i2c_client` اللي بنقرأ منه الـ address والـ IRQ |

#### الـ `handler_pre`

ده الـ callback اللي بيتشغّل **قبل** ما `adnp_i2c_probe` تتنفّذ — بنجيب الـ `i2c_client` من الـ registers مباشرة (الـ ABI بيضمن إن الأرجومنت الأول يكون في RDI على x86-64) وبنطبع الـ I2C address والـ IRQ رقم عشان نعرف أي device اتـ probe.

#### الـ `handler_post`

بيتشغّل **بعد** رجوع الـ function عشان نشوف هل الـ probe نجح (return 0) أو فضل error — ده مفيد جداً في الـ debugging لأنه بيكشف لو الـ I2C communication أو الـ GPIO setup فشل.

#### الـ `adnp_probe` struct

الـ `symbol_name` بيحدد اسم الـ function في الـ kernel symbol table — الـ kprobe subsystem بيحوّله لـ address وقت الـ registration، ومحتاجين `pre_handler` و `post_handler` عشان نربط الـ callbacks.

#### `module_init` — `register_kprobe`

`register_kprobe` بتكتب **breakpoint instruction** في الذاكرة عند بداية `adnp_i2c_probe` — لو رجعت قيمة سالبة معناها إن الـ symbol مش موجود أو الـ kprobes مش مفعّل في الـ kernel config.

#### `module_exit` — `unregister_kprobe`

**لازم** نعمل `unregister_kprobe` في الـ exit عشان الـ kernel يرجّع الـ original instruction تاني ويحرر الـ resources — لو متعملتش الـ unregister هيحصل kernel panic في أول interrupt بعد ما الـ module يتـ unload.

---

### Makefile للـ Build

```makefile
obj-m += kprobe_adnp.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

### تشغيل وتجربة

```bash
# build
make

# load
sudo insmod kprobe_adnp.ko

# اشوف الـ log بعد ما تـ probe أي ADNP device
sudo dmesg | grep kprobe_adnp

# unload
sudo rmmod kprobe_adnp
```

**مثال على الـ output المتوقع:**

```
[kprobe_adnp] planted kprobe on "adnp_i2c_probe" at ffffffffc0a12340
[kprobe_adnp] adnp_i2c_probe called: I2C addr=0x41  irq=160  adapter="i2c-0"
[kprobe_adnp] adnp_i2c_probe returned: 0 (OK)
[kprobe_adnp] kprobe removed from "adnp_i2c_probe"
```

الـ address `0x41` مطابق للـ device tree example في `gpio-adnp.txt` — ده بيأكد إن الـ hook شغال صح وبيعترض الـ probe الحقيقي للـ ADNP expander.
