## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي له الملف

**الـ ARM/NOMADIK/Ux500 ARCHITECTURES** subsystem — مسؤول عنه Linus Walleij من Linaro، وده subsystem خاص بـ SoC عائلة Nomadik من STMicroelectronics، أشهرها DB8500 اللي اتستخدمت في موبايلات Nokia القديمة زي N9 وغيرها.

---

### القصة الكاملة — تخيل مبنى فيه لوحة توصيل ضخمة

تخيل إن عندك **مبنى حكومي** فيه 512 باب. كل باب (pin) ممكن يتفتح بـ 4 مفاتيح مختلفة:
- مفتاح GPIO (الباب بيشتغل كـ input أو output عادي)
- مفتاح Alt-A (الباب بيشتغل لجهاز معين زي UART)
- مفتاح Alt-B (الباب بيشتغل لجهاز تاني زي SPI)
- مفتاح Alt-C (الباب بيشتغل لجهاز تالت زي I2C)

المشكلة إن المبنى ده في كل مرة بياخد قيلولة (sleep)، الأبواب بتتصرف بشكل مختلف. وفي SoC زي DB8500 في وحدة تانية اسمها **PRCM (Power Reset Clock Manager)** بتتحكم في بعض الأبواب الخاصة جداً (Alt-C1 لـ Alt-C4) من خلال registers منفصلة خالص.

**الملف ده** `gpio-nomadik.h` هو **المخطط التفصيلي** للمبنى ده — بيحدد:
1. عنوان كل باب (register offsets)
2. شكل المفاتيح (enums وـ structs)
3. الـ contract بين الـ GPIO driver والـ pinctrl driver

---

### ليه الملف ده موجود أصلاً؟

الـ Nomadik GPIO IP block عبارة عن **AMBA peripheral** بيتكرر في الـ SoC كـ "bank" كل bank فيه 32 pin. الـ SoC ممكن يكون فيها لحد 512 pin = 16 bank.

**المشكلة الكبيرة**: في Linux، الـ GPIO driver والـ pinctrl driver مفصولين conceptually، بس في الـ Nomadik hardware هما شيء واحد فيزيائياً — نفس الـ registers! فلازم يشاركوا:
- نفس الـ `nmk_gpio_chip` struct
- نفس الـ spinlocks
- نفس الـ sleep-mode state

فالحل كان إن الـ shared definitions تتحط في header ملف مشترك بين الـ driver الاتنين.

---

### هدف الملف تفصيلياً

#### 1. تعريف عناوين الـ Hardware Registers

```c
#define NMK_GPIO_DAT    0x00  /* Data register — قراءة حالة الـ pins */
#define NMK_GPIO_DATS   0x04  /* Data Set — اكتب 1 عشان تعمل SET لـ pin */
#define NMK_GPIO_DATC   0x08  /* Data Clear — اكتب 1 عشان تعمل CLEAR لـ pin */
#define NMK_GPIO_DIR    0x10  /* Direction register */
#define NMK_GPIO_AFSLA  0x20  /* Alternate Function Select A */
#define NMK_GPIO_AFSLB  0x24  /* Alternate Function Select B */
#define NMK_GPIO_SLPC   0x1c  /* Sleep mode config */
#define NMK_GPIO_RIMSC  0x40  /* Rising-edge Interrupt Mask Set/Clear */
#define NMK_GPIO_FIMSC  0x44  /* Falling-edge Interrupt Mask Set/Clear */
```

**الـ DATS وـ DATC** بدل ما تعمل read-modify-write على DAT مباشرة، بتكتب في Set register أو Clear register مباشرة — ده بيحل مشكلة الـ race conditions بدون ما تحتاج lock في الـ simple cases.

#### 2. تعريف الـ Alternate Function Modes

```c
#define NMK_GPIO_ALT_GPIO  0        /* plain GPIO */
#define NMK_GPIO_ALT_A     1        /* func A: set bit in AFSLA */
#define NMK_GPIO_ALT_B     2        /* func B: set bit in AFSLB */
#define NMK_GPIO_ALT_C     (1 | 2)  /* func C: set BOTH bits */
```

وفي الـ DB8500 في alt-C variants إضافية (C1→C4) بتتحكم فيها الـ PRCM عبر registers منفصلة تماماً.

#### 3. تعريف الـ Sleep Mode

```c
enum nmk_gpio_slpm {
    NMK_GPIO_SLPM_INPUT,          /* pin behaves as input during sleep */
    NMK_GPIO_SLPM_NOCHANGE,       /* pin keeps its current function */
};
```

ده مهم جداً لأن الـ Nomadik SoCs بتدخل sleep كتير لتوفير البطارية، ولازم كل pin يتحدد يعمل إيه لما الـ SoC نايم.

#### 4. الـ `nmk_gpio_chip` struct — قلب النظام

```c
struct nmk_gpio_chip {
    struct gpio_chip chip;       /* Linux GPIO framework */
    void __iomem *addr;          /* base address of the 32-pin bank */
    struct clk *clk;             /* clock for the block */
    unsigned int bank;           /* which bank (0..15) */
    spinlock_t lock;             /* protects register access */
    bool sleepmode;              /* does this bank support sleep? */
    bool is_mobileye_soc;        /* limited STA2X11 variant */
    u32 edge_rising;             /* cached rising edge config */
    u32 edge_falling;            /* cached falling edge config */
    u32 rwimsc, fwimsc;          /* wake interrupt mask — rising/falling */
    u32 pull_up;                 /* pull-up state cache */
    u32 lowemi;                  /* low EMI drive strength */
};
```

#### 5. تعريف الـ PRCM GPIOCR — الـ Alt-C الخاصة

الـ `PRCM_GPIOCR_ALTCX` macro ده بيبني جدول ضخم لكل pin يدعم Alt-C1 لـ Alt-C4، بيقول:
- رقم الـ pin
- رقم الـ PRCM register
- رقم الـ bit في الـ register

```c
struct prcm_gpiocr_altcx_pin_desc {
    unsigned short pin;
    struct prcm_gpiocr_altcx altcx[PRCM_IDX_GPIOCR_ALTC_MAX]; /* C1..C4 */
};
```

#### 6. الـ SoC-specific data

```c
struct nmk_pinctrl_soc_data {
    const struct pinctrl_pin_desc *pins;   /* all pins description */
    const struct nmk_function *functions;  /* mux functions like "uart0" */
    const struct nmk_pingroup *groups;     /* pin groups like "uart0grp" */
    const struct prcm_gpiocr_altcx_pin_desc *altcx_pins; /* PRCM pins */
    const u16 *prcm_gpiocr_registers;     /* PRCM register addresses */
};
```

كل SoC (STN8815، DB8500، DB8540) بتسجل نفسها عبر:
```c
void nmk_pinctrl_db8500_init(const struct nmk_pinctrl_soc_data **soc);
```

---

### الـ Shared Symbols — الجسر بين الـ Drivers

الملف بيعلن symbols بيستخدمها كل driver من التاني:

**الـ `gpio-nomadik.c` بيوفرها للـ `pinctrl-nomadik.c`:**
```c
void __nmk_gpio_make_output(...);     /* اعمل pin output */
void __nmk_gpio_set_slpm(...);        /* غير sleep mode لـ pin */
struct nmk_gpio_chip *nmk_gpio_populate_chip(...); /* ابني chip من fwnode */
void nmk_gpio_dbg_show_one(...);      /* debugfs output */
```

**الـ `pinctrl-nomadik.c` بيوفرها للـ `gpio-nomadik.c`:**
```c
extern struct nmk_gpio_chip *nmk_gpio_chips[NMK_MAX_BANKS]; /* الـ banks array */
extern spinlock_t nmk_gpio_slpm_lock; /* global lock لعمليات sleep */
int nmk_prcm_gpiocr_get_mode(...);    /* اعرف mode الـ pin الحالي */
```

---

### الملفات المرتبطة اللي المفروض تعرفها

| الملف | الدور |
|-------|-------|
| `drivers/gpio/gpio-nomadik.c` | الـ GPIO driver الأساسي — بيتحكم في الـ 32-pin banks |
| `drivers/pinctrl/nomadik/pinctrl-nomadik.c` | الـ pinmux/pinconf driver — بيتحكم في الـ alt functions |
| `drivers/pinctrl/nomadik/pinctrl-nomadik-db8500.c` | بيانات الـ pins والـ functions للـ DB8500 SoC |
| `drivers/pinctrl/nomadik/pinctrl-nomadik-stn8815.c` | بيانات الـ pins والـ functions للـ STN8815 SoC |
| `include/dt-bindings/pinctrl/nomadik.h` | macros للـ Device Tree لتعريف pin configurations |
| `Documentation/devicetree/bindings/gpio/st,nomadik-gpio.yaml` | DT binding schema للـ GPIO controller |
| `arch/arm/mach-nomadik/` | الـ machine-specific code للـ Nomadik SoC |
| `arch/arm/mach-ux500/` | الـ machine-specific code للـ Ux500 (DB8500 family) |

---

### الملفات اللي بتكوّن الـ Subsystem

#### Core / Framework
- `include/linux/gpio/gpio-nomadik.h` — الـ shared header (الملف ده)
- `drivers/gpio/gpio-nomadik.c` — GPIO driver (32-pin bank management + IRQ)
- `drivers/pinctrl/nomadik/pinctrl-nomadik.c` — pinmux + pinconf driver

#### SoC-Specific Data
- `drivers/pinctrl/nomadik/pinctrl-nomadik-db8500.c` — DB8500 pin table
- `drivers/pinctrl/nomadik/pinctrl-nomadik-stn8815.c` — STN8815 pin table
- `include/dt-bindings/pinctrl/nomadik.h` — DT macros للـ pin config

#### AB-series (الـ PMIC GPIO)
- `drivers/pinctrl/nomadik/pinctrl-ab8500.c` — AB8500 PMIC GPIO
- `drivers/pinctrl/nomadik/pinctrl-ab8505.c` — AB8505 PMIC GPIO
- `drivers/pinctrl/nomadik/pinctrl-abx500.c` — ABx500 common code
- `drivers/pinctrl/nomadik/pinctrl-abx500.h` — ABx500 shared header
## Phase 2: شرح الـ GPIO / Pinctrl Nomadik Framework

### المشكلة — ليه المنظومة دي موجودة أصلاً؟

في أي SoC حديث زي ST-Ericsson DB8500 (اللي بيشتغل في موبايلات Nokia القديمة بنظام Symbian/MeeGo)، عندك مئات الـ pins الجسدية على الشريحة. كل pin ممكن يشتغل في أكتر من role واحد:

- **GPIO** عادي: input/output رقمي بسيط
- **Alternate Function A**: مثلاً يبقى TX لـ UART
- **Alternate Function B**: مثلاً يبقى SDA لـ I2C
- **Alternate Function C / C1..C4**: وظايف تانية أكثر تعقيداً، بعضها بيتحكم فيه عن طريق **PRCM** (Power Reset Clock Manager) مش عن طريق GPIO block نفسه

المشكلة إن ما فيش abstraction موحدة في الكيرنل زمان — كل driver كان بيعمل `ioremap` على رجسترات الـ GPIO بنفسه وبيكتب فيها directly. النتيجة: **تعارض** بين الـ drivers، **تكرار كود**، وصعوبة إدارة power modes في وقت النوم (sleep).

الـ Nomadik GPIO/Pinctrl framework جاء علشان يحل تحديداً:

1. **Multiplexing conflict**: مين مسموله يستخدم pin معين؟
2. **GPIO abstraction**: توفير API موحد فوق hardware-specific registers
3. **Sleep mode management**: الـ pins لازم تتصرف بشكل معين وقت suspend
4. **PRCM integration**: بعض الـ alternate-C functions محتاجة رجسترات PRCM خارج الـ GPIO block

---

### الحل — المنهج اللي اتبعه الكيرنل

الكيرنل قسم المشكلة لطبقتين متعاملتين مع بعض:

| الطبقة | المسؤولية |
|--------|-----------|
| **GPIO subsystem** (`gpio_chip`) | التحكم في direction, value, interrupt لكل pin |
| **Pinctrl subsystem** (`pinctrl_dev`) | إدارة muxing وتهيئة الـ pins قبل ما الـ drivers تستخدمها |

الـ Nomadik driver (`gpio-nomadik` + `pinctrl-nomadik`) بيعمل implement للاتنين في نفس الوقت لأن الـ hardware الـ Nomadik بيدمجهم في نفس الـ IP block.

---

### التشبيه الحقيقي — لوحة قواطع الكهرباء في عمارة

تخيل عمارة سكنية فيها لوحة قواطع كهربائية رئيسية:

| العنصر في التشبيه | المقابل في الكيرنل |
|-------------------|--------------------|
| كل **خط كهربائي** في العمارة | كل **GPIO pin** في الـ SoC |
| **القاطع** (circuit breaker) | الـ `nmk_gpio_chip` — بيتحكم في الـ pin state |
| **لوحة التوزيع الفرعية** لكل دور | الـ GPIO bank (32 pin لكل bank) |
| **مهندس الكهرباء** اللي بيقرر مين يستخدم أي خط | الـ `pinctrl` subsystem — بيقرر الـ mux setting |
| **نظام الطوارئ الليلي** (بيغير توصيلات معينة وقت الليل) | الـ sleep mode — `NMK_GPIO_SLPM_*` |
| **غرفة المولدات الخارجية** (PRCM) | الـ PRCM GPIOCR registers — بتتحكم في alternate-C خاص |
| **المخططات التنفيذية للعمارة** (blueprints) | الـ `nmk_pinctrl_soc_data` — بتوصف كل pins وfunctions |

لما الـ UART driver يطلب "أنا عايز أشتغل على pin 14 و 15"، الـ pinctrl subsystem بيراجع المخططات، يتأكد إن محدش تاني بيستخدمهم، ويوجّه المهندس (nmk_gpio_chip) يضبط القواطع الصح. أما لو الوظيفة المطلوبة تحتاج "غرفة المولدات" (PRCM)، بيبعت أوامر تانية لغرفة المولدات.

---

### الصورة الكبيرة — Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Consumer Drivers                             │
│   UART driver    I2C driver    SPI driver    Button driver      │
│       │               │            │               │           │
│       └───────────────┴────────────┴───────────────┘           │
│                            │                                    │
│              pinctrl_select_state() / gpiod_get()               │
└───────────────────────────┬─────────────────────────────────────┘
                            │
          ┌─────────────────┴──────────────────┐
          │                                    │
          ▼                                    ▼
┌──────────────────┐                ┌──────────────────────┐
│  Pinctrl Core    │                │   GPIO Core          │
│  (pinctrl.c)     │                │   (gpiolib.c)        │
│                  │                │                      │
│ - mux management │                │ - descriptor mgmt    │
│ - state machine  │                │ - sysfs/chardev      │
│ - pin grouping   │                │ - IRQ integration    │
└────────┬─────────┘                └──────────┬───────────┘
         │                                     │
         │    ┌────────────────────────────────┘
         │    │
         ▼    ▼
┌───────────────────────────────────────────────────────┐
│         pinctrl-nomadik + gpio-nomadik driver         │
│                                                       │
│  ┌──────────────────────────────────────────────┐     │
│  │  nmk_pinctrl_soc_data  (per-SoC data)        │     │
│  │  ├── pinctrl_pin_desc[]  (all pins)          │     │
│  │  ├── nmk_function[]      (UART, I2C, SPI...) │     │
│  │  └── nmk_pingroup[]      (pin groups + alt)  │     │
│  └──────────────────────────────────────────────┘     │
│                                                       │
│  ┌─────────────────┐    ┌────────────────────────┐    │
│  │ nmk_gpio_chip[] │    │ PRCM GPIOCR handling   │    │
│  │ (one per bank)  │    │ (for ALT_C1..C4)       │    │
│  │ ├── gpio_chip   │    │                        │    │
│  │ ├── void __iomem│    │ prcm_gpiocr_altcx[]    │    │
│  │ ├── clk         │    └────────────────────────┘    │
│  │ ├── edge_rising │                                  │
│  │ └── edge_falling│                                  │
│  └─────────────────┘                                  │
└───────────────────────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────┐
│              Nomadik GPIO Hardware                    │
│                                                       │
│   Bank 0 (GPIO 0-31)    Bank 1 (GPIO 32-63)   ...    │
│   ┌──────────────┐      ┌──────────────┐             │
│   │ DAT  DATS    │      │ DAT  DATS    │             │
│   │ DATC DIR     │      │ DATC DIR     │             │
│   │ AFSLA AFSLB  │      │ AFSLA AFSLB  │             │
│   │ RIMSC FIMSC  │      │ RIMSC FIMSC  │             │
│   └──────────────┘      └──────────────┘             │
│                                                       │
│   PRCM Block (external to GPIO block)                 │
│   ┌─────────────────────────────────────┐             │
│   │ GPIOCR1  GPIOCR2  GPIOCR3          │             │
│   └─────────────────────────────────────┘             │
└───────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الفكرة المحورية في المنظومة دي هي **فصل الـ "what" عن الـ "how"**:

- الـ **"what"** = الـ consumer driver بيطلب "أنا عايز UART0 يشتغل" — مش بيتكلم عن pins بالاسم
- الـ **"how"** = الـ pinctrl framework بيحدد إن UART0 يستخدم مثلاً pin 14 (TX) و pin 15 (RX)، وإن دول محتاجين ALT_A setting في رجسترات AFSLA

ده بيتحقق عبر **ثلاث مفاهيم أساسية**:

#### 1. الـ `pinctrl_pin_desc` — وصف كل pin

```c
struct pinctrl_pin_desc {
    unsigned int number;  /* رقم Pin عالمي في الـ SoC */
    const char *name;     /* "GPIO0", "GPIO1", ... */
    void *drv_data;       /* بيانات خاصة بالـ driver */
};
```

ده بيعمل **global namespace** لكل الـ pins — زي أرقام الشقق في العمارة.

#### 2. الـ `nmk_pingroup` — مجموعة pins بتؤدي وظيفة واحدة

```c
struct nmk_pingroup {
    struct pingroup grp;  /* اسم المجموعة + array of pin numbers */
    int altsetting;       /* ALT_A, ALT_B, or ALT_C */
};
```

مثلاً:
```c
// في SoC data:
// uart0_pins[] = { 14, 15 }  (TX, RX)
NMK_PIN_GROUP(uart0, NMK_GPIO_ALT_A)
// => grp.name = "uart0", grp.pins = uart0_pins, altsetting = ALT_A
```

#### 3. الـ `nmk_function` — الوظيفة اللي ممكن تتعمل assign للـ group

```c
struct nmk_function {
    const char *name;           /* "uart0", "i2c1", "spi2" */
    const char * const *groups; /* الـ pin groups اللي ممكن تعمل الـ function ده */
    unsigned int ngroups;
};
```

---

### الـ `nmk_gpio_chip` بالتفصيل

```c
struct nmk_gpio_chip {
    struct gpio_chip chip;    // الـ abstraction الأساسية — lazim موجودة
    void __iomem *addr;       // base address للـ GPIO bank registers
    struct clk *clk;          // الـ clock للـ IP block — لازم يتفعّل قبل أي access
    unsigned int bank;        // رقم الـ bank (0, 1, 2, ...)
    void (*set_ioforce)(bool enable); // callback لتفعيل ioforce (لازم للـ sleep mode)
    spinlock_t lock;          // حماية concurrent access على الـ registers
    bool sleepmode;           // هل الـ chip بتدعم sleep mode؟
    bool is_mobileye_soc;     // quirk خاص بـ Mobileye SoC

    /* حفظ الحالة الحالية — لأن الـ registers بتتغير */
    u32 edge_rising;   // bitmask للـ pins اللي عندها rising edge interrupt
    u32 edge_falling;  // bitmask للـ pins اللي عندها falling edge interrupt
    u32 real_wake;     // الـ pins الحقيقية اللي لازم توصحي السيستم من sleep
    u32 rwimsc;        // Rising Wake-up Interrupt Mask — shadow register
    u32 fwimsc;        // Falling Wake-up Interrupt Mask — shadow register
    u32 rimsc;         // Rising Interrupt Mask — shadow register
    u32 fimsc;         // Falling Interrupt Mask — shadow register
    u32 pull_up;       // bitmask للـ pins اللي عليها pull-up مفعّل
    u32 lowemi;        // Low-EMI mode — بيقلل التشويش الكهرومغناطيسي
};
```

**ليه الـ shadow registers؟**
الـ hardware مش بيسمح بـ read-modify-write على بعض الرجسترات (زي RIMSC) بأمان في سياق الـ interrupt. فالـ driver بيحتفظ بـ software copy ويعمل write-only للـ hardware عند الحاجة.

---

### رجسترات الـ Hardware — خريطة

```
Base + 0x00  NMK_GPIO_DAT     = القيم الحالية (read) أو set+clear معاً (write)
Base + 0x04  NMK_GPIO_DATS    = Data Set   — اكتب 1 علشان تعمل set للـ bit المقابل
Base + 0x08  NMK_GPIO_DATC    = Data Clear — اكتب 1 علشان تعمل clear للـ bit المقابل
Base + 0x0c  NMK_GPIO_PDIS    = Pull Disable — 1 = disable pull resistor
Base + 0x10  NMK_GPIO_DIR     = Direction — 1=output, 0=input
Base + 0x14  NMK_GPIO_DIRS    = Direction Set
Base + 0x18  NMK_GPIO_DIRC    = Direction Clear
Base + 0x1c  NMK_GPIO_SLPC    = Sleep Configuration
Base + 0x20  NMK_GPIO_AFSLA   = Alternate Function Select A (bit per pin)
Base + 0x24  NMK_GPIO_AFSLB   = Alternate Function Select B (bit per pin)
Base + 0x28  NMK_GPIO_LOWEMI  = Low EMI mode

Base + 0x40  NMK_GPIO_RIMSC   = Rising edge Interrupt Mask (wake from sleep)
Base + 0x44  NMK_GPIO_FIMSC   = Falling edge Interrupt Mask (wake from sleep)
Base + 0x48  NMK_GPIO_IS      = Interrupt Status
Base + 0x4c  NMK_GPIO_IC      = Interrupt Clear
Base + 0x50  NMK_GPIO_RWIMSC  = Rising Wake-up Interrupt Mask
Base + 0x54  NMK_GPIO_FWIMSC  = Falling Wake-up Interrupt Mask
Base + 0x58  NMK_GPIO_WKS     = Wake-up Status
Base + 0x5c  NMK_GPIO_EDGELEVEL = Edge/Level select (DB8540+)
Base + 0x60  NMK_GPIO_LEVEL   = Level configuration (DB8540+)
```

**كيف AFSLA + AFSLB بيحددوا الـ Alt Function:**

```
AFSLB[n] | AFSLA[n] | النتيجة
---------|----------|--------
    0    |    0     | GPIO (NMK_GPIO_ALT_GPIO)
    0    |    1     | Alternate A (NMK_GPIO_ALT_A)
    1    |    0     | Alternate B (NMK_GPIO_ALT_B)
    1    |    1     | Alternate C (NMK_GPIO_ALT_C) — يحتاج PRCM لـ C1..C4
```

---

### الـ ALT_C الخاص والـ PRCM

الـ Alternate-C مش بتتحكم فيه رجسترات الـ GPIO block بس. في pins معينة، لازم تكتب في **PRCM GPIOCR registers** علشان تحدد هل ده ALT_C1 ولا ALT_C2 ولا C3 ولا C4.

```c
struct prcm_gpiocr_altcx {
    bool used:1;      // هل الـ sub-variant ده متاح لهذا الـ pin؟
    u8 reg_index:2;   // GPIOCR1 ولا GPIOCR2 ولا GPIOCR3؟
    u8 control_bit:5; // رقم الـ bit الجوا الـ register
} __packed;           // packed لأن size حرج في الـ arrays الكبيرة

struct prcm_gpiocr_altcx_pin_desc {
    unsigned short pin;                          // رقم الـ pin
    struct prcm_gpiocr_altcx altcx[4];          // C1, C2, C3, C4
};
```

الـ macro `PRCM_GPIOCR_ALTCX` بيسهّل تعريف الـ table:

```c
PRCM_GPIOCR_ALTCX(14,          // pin number
    1, PRCM_IDX_GPIOCR1, 3,    // ALT_C1: reg=GPIOCR1, bit=3
    0, 0, 0,                   // ALT_C2: غير متاح
    0, 0, 0,                   // ALT_C3: غير متاح
    0, 0, 0)                   // ALT_C4: غير متاح
```

---

### الـ Sleep Mode

الـ `enum nmk_gpio_slpm` بيحدد سلوك الـ pin وقت الـ suspend:

```c
enum nmk_gpio_slpm {
    NMK_GPIO_SLPM_INPUT          = 0,  // الـ pin يتحول لـ input في النوم
    NMK_GPIO_SLPM_WAKEUP_ENABLE  = 0,  // نفس القيمة — يقدر يوصحي السيستم
    NMK_GPIO_SLPM_NOCHANGE       = 1,  // يفضل على حاله
    NMK_GPIO_SLPM_WAKEUP_DISABLE = 1,  // نفس القيمة — ما يوصحيش السيستم
};
```

الـ `nmk_gpio_chip` بيحتفظ بـ `real_wake` بيتفرق عن `rwimsc` / `fwimsc`:
- **`rwimsc` / `fwimsc`**: الـ interrupts اللي مفعّلة للـ wake-up
- **`real_wake`**: الـ pins اللي فعلاً المفروض توصحي السيستم (بعد تصفية الـ driver logic)

---

### الـ Struct Relationship Map

```
nmk_pinctrl_soc_data
├── pinctrl_pin_desc[]  ──────────────────────────────────────────┐
│   { number, name, drv_data }                                    │
│                                                                 │
├── nmk_function[]                                                │
│   { name, groups[], ngroups }                                   │
│           │                                                     │
│           ▼                                                     │
├── nmk_pingroup[]                                                │
│   { pingroup { name, pins[], npins }, altsetting }              │
│              pins[] references ──────────────────────────────►──┘
│
├── prcm_gpiocr_altcx_pin_desc[]
│   { pin, altcx[4] { used, reg_index, control_bit } }
│
└── prcm_gpiocr_registers[]  (u16[])
    GPIOCR1_addr, GPIOCR2_addr, GPIOCR3_addr


nmk_gpio_chips[NMK_MAX_BANKS]  (global array)
│
└─► nmk_gpio_chip
    ├── gpio_chip ◄──── registered with GPIO Core (gpiolib)
    │   ├── label
    │   ├── ngpio = 32 (NMK_GPIO_PER_CHIP)
    │   ├── .get_direction()
    │   ├── .direction_input()
    │   ├── .direction_output()
    │   ├── .get()
    │   ├── .set()
    │   └── .to_irq()
    │
    ├── addr ──────────► GPIO Bank Registers (MMIO)
    ├── clk ───────────► Clock for the IP block
    ├── bank ──────────► Bank index (0..NMK_MAX_BANKS-1)
    ├── lock ──────────► spinlock (protects register access)
    ├── sleepmode
    ├── edge_rising  (software shadow)
    ├── edge_falling (software shadow)
    ├── rwimsc/fwimsc/rimsc/fimsc (shadow registers)
    └── pull_up / lowemi
```

---

### إيه اللي بيملكه الـ Framework مقابل اللي بيفوضه للـ Drivers

#### بيملكه الـ Framework (ما بيلمسوش الـ drivers):

| الموضوع | التفاصيل |
|---------|----------|
| **Pin namespace** | الترقيم العالمي للـ pins، الأسماء، الـ lookup |
| **Mux conflict detection** | مين request pin معين، منع التعارض |
| **State machine** | الانتقال بين "default", "sleep", "idle" states |
| **GPIO-to-IRQ mapping** | ربط الـ GPIO numbers بأرقام الـ Linux IRQs |
| **sysfs / debugfs interface** | `/sys/class/gpio/`, `/sys/kernel/debug/pinctrl/` |

#### بيفوضه للـ Nomadik Driver:

| الموضوع | التفاصيل |
|---------|----------|
| **Register access** | كتابة على AFSLA/AFSLB/DAT/DIR/... |
| **PRCM interaction** | تفعيل ALT_C1..C4 عبر GPIOCR registers |
| **Sleep behavior** | تهيئة الـ SLPC register والـ wake-up masks |
| **Pull resistors** | PDIS register — enable/disable pull-up/pull-down |
| **Low EMI mode** | LOWEMI register |
| **Bank clock management** | enable/disable الـ clock للـ bank |
| **SoC-specific data** | الـ pin table وتعريفات الـ functions لكل SoC (STN8815, DB8500, DB8540) |

---

### تفاعل الـ gpio-nomadik مع الـ pinctrl-nomadik

الملف `gpio-nomadik.h` بيوضح إن في **اعتماد متبادل** بين الجزئين:

```c
/* gpio-nomadik exports (used by pinctrl-nomadik) */
void nmk_gpio_dbg_show_one(...);
void __nmk_gpio_make_output(struct nmk_gpio_chip *nmk_chip, ...);
void __nmk_gpio_set_slpm(struct nmk_gpio_chip *nmk_chip, ...);
struct nmk_gpio_chip *nmk_gpio_populate_chip(struct fwnode_handle *fwnode, ...);

/* pinctrl-nomadik exports (used by gpio-nomadik) */
#ifdef CONFIG_PINCTRL_NOMADIK
extern struct nmk_gpio_chip *nmk_gpio_chips[NMK_MAX_BANKS]; // الـ global array
extern spinlock_t nmk_gpio_slpm_lock;                        // lock مشترك للـ sleep
int nmk_prcm_gpiocr_get_mode(struct pinctrl_dev *pctldev, int gpio);
#endif
```

الـ `nmk_gpio_slpm_lock` مهم جداً: لما السيستم بيدخل sleep، لازم تغير الـ sleep mode لكل البنوك atomically — فالـ lock ده بيضمن consistency عبر كل الـ banks.

---

### الـ SoC Variants وإزاي بيتعاملوا مع بعض

```c
/* كل SoC بيعرّف function بتملا nmk_pinctrl_soc_data */
void nmk_pinctrl_stn8815_init(const struct nmk_pinctrl_soc_data **soc);
void nmk_pinctrl_db8500_init(const struct nmk_pinctrl_soc_data **soc);
void nmk_pinctrl_db8540_init(const struct nmk_pinctrl_soc_data **soc);
```

الـ pinctrl-nomadik driver بيقرر أي init function يشغّل حسب الـ Device Tree أو الـ `PINCTRL_NMK_*` constants:

```c
#define PINCTRL_NMK_STN8815  0   // ST Nomadik 8815 — أول chip
#define PINCTRL_NMK_DB8500   1   // ST-Ericsson DB8500 — في Nokia N9
```

الفايدة: نفس الـ pinctrl-nomadik core code بيشتغل مع كل الـ SoC variants، والاختلاف بس في الـ data (الـ pin table وعدد الـ banks وتعريفات الـ functions).
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Enums والـ Flags والـ Config Options — Cheatsheet

#### `enum nmk_gpio_pull` — وضع الـ Pull Resistor

| القيمة | المعنى |
|---|---|
| `NMK_GPIO_PULL_NONE` | مفيش pull resistor — الـ pin طايف |
| `NMK_GPIO_PULL_UP` | pull-up مفعّل — الـ pin بيشد لـ VCC لما مفيش إشارة |
| `NMK_GPIO_PULL_DOWN` | pull-down مفعّل — الـ pin بيشد للـ GND |

#### `enum nmk_gpio_slpm` — وضع الـ Sleep

| القيمة | المعنى |
|---|---|
| `NMK_GPIO_SLPM_INPUT` | الـ pin بيتحول لـ input وقت الـ sleep |
| `NMK_GPIO_SLPM_WAKEUP_ENABLE` | نفس `SLPM_INPUT` — بيعمل wake-up للـ chip |
| `NMK_GPIO_SLPM_NOCHANGE` | الـ pin مش بيتغيرش وقت الـ sleep |
| `NMK_GPIO_SLPM_WAKEUP_DISABLE` | نفس `SLPM_NOCHANGE` — بيمنع الـ wake-up |

#### `enum prcm_gpiocr_reg_index` — فهرس الـ PRCM Register

| القيمة | المعنى |
|---|---|
| `PRCM_IDX_GPIOCR1` | الـ register الأول `GPIOCR1` في الـ PRCM |
| `PRCM_IDX_GPIOCR2` | الـ register الثاني |
| `PRCM_IDX_GPIOCR3` | الـ register الثالث |

#### `enum prcm_gpiocr_altcx_index` — فهرس الـ Alt-C Function

| القيمة | المعنى |
|---|---|
| `PRCM_IDX_GPIOCR_ALTC1` | الـ alternate-C function الأولى |
| `PRCM_IDX_GPIOCR_ALTC2` | الثانية |
| `PRCM_IDX_GPIOCR_ALTC3` | الثالثة |
| `PRCM_IDX_GPIOCR_ALTC4` | الرابعة |
| `PRCM_IDX_GPIOCR_ALTC_MAX` | عدد الـ functions — بيُستخدم كـ array size |

#### الـ Alternate Functions — Cheatsheet

| الماكرو | القيمة | المعنى |
|---|---|---|
| `NMK_GPIO_ALT_GPIO` | `0` | الـ pin شغّال GPIO عادي |
| `NMK_GPIO_ALT_A` | `1` | الـ alternate function A — بيتحكم فيه `AFSLA` |
| `NMK_GPIO_ALT_B` | `2` | الـ alternate function B — بيتحكم فيه `AFSLB` |
| `NMK_GPIO_ALT_C` | `3` (A\|B) | الـ C الأساسي — الاتنين بيتحطوا بقيمة 1 |
| `NMK_GPIO_ALT_C1` | `0x07` | C الأولى — بيتحكم فيها الـ PRCM |
| `NMK_GPIO_ALT_C2` | `0x0B` | C الثانية |
| `NMK_GPIO_ALT_C3` | `0x0F` | C الثالثة |
| `NMK_GPIO_ALT_C4` | `0x13` | C الرابعة |

#### الـ Hardware Registers — Cheatsheet

| الماكرو | الأوفست | الوظيفة |
|---|---|---|
| `NMK_GPIO_DAT` | `0x00` | قراءة/كتابة قيم الـ pins |
| `NMK_GPIO_DATS` | `0x04` | Set — كتابة 1 بيرفع الـ pin |
| `NMK_GPIO_DATC` | `0x08` | Clear — كتابة 1 بيخفض الـ pin |
| `NMK_GPIO_PDIS` | `0x0C` | Pull disable — بيعطّل الـ pull resistors |
| `NMK_GPIO_DIR` | `0x10` | اتجاه الـ pin (0=input, 1=output) |
| `NMK_GPIO_DIRS` | `0x14` | Set direction لـ output |
| `NMK_GPIO_DIRC` | `0x18` | Clear direction لـ input |
| `NMK_GPIO_SLPC` | `0x1C` | Sleep configuration |
| `NMK_GPIO_AFSLA` | `0x20` | Alternate function select A |
| `NMK_GPIO_AFSLB` | `0x24` | Alternate function select B |
| `NMK_GPIO_LOWEMI` | `0x28` | Low EMI mode |
| `NMK_GPIO_RIMSC` | `0x40` | Rising edge interrupt mask (wake from sleep) |
| `NMK_GPIO_FIMSC` | `0x44` | Falling edge interrupt mask |
| `NMK_GPIO_IS` | `0x48` | Interrupt status |
| `NMK_GPIO_IC` | `0x4C` | Interrupt clear |
| `NMK_GPIO_RWIMSC` | `0x50` | Rising edge wake-up interrupt mask |
| `NMK_GPIO_FWIMSC` | `0x54` | Falling edge wake-up interrupt mask |
| `NMK_GPIO_WKS` | `0x58` | Wake-up status |
| `NMK_GPIO_EDGELEVEL` | `0x5C` | Edge/level select (DB8540+) |
| `NMK_GPIO_LEVEL` | `0x60` | Level mode register (DB8540+) |

#### الـ Config Options الأساسية

| الـ Kconfig | التأثير |
|---|---|
| `CONFIG_PINCTRL_STN8815` | بيفعّل `nmk_pinctrl_stn8815_init()` للـ SoC القديم |
| `CONFIG_PINCTRL_DB8500` | بيفعّل `nmk_pinctrl_db8500_init()` للـ DB8500 |
| `CONFIG_PINCTRL_DB8540` | بيفعّل `nmk_pinctrl_db8540_init()` للـ DB8540 |
| `CONFIG_PINCTRL_NOMADIK` | بيفعّل `nmk_gpio_chips[]` و`nmk_gpio_slpm_lock` كـ globals |
| `CONFIG_DEBUG_FS` | بيفعّل `nmk_gpio_dbg_show_one()` |

#### Layout Macros — تخطيط الـ Banks

| الماكرو | القيمة | المعنى |
|---|---|---|
| `GPIO_BLOCK_SHIFT` | `5` | كل bank = 2^5 = 32 pin |
| `NMK_GPIO_PER_CHIP` | `BIT(5)` = 32 | عدد الـ pins في كل chip |
| `NMK_MAX_BANKS` | `DIV_ROUND_UP(512, 32)` = 16 | أقصى عدد للـ banks في النظام |

---

### الـ Structs الأساسية

#### `struct nmk_gpio_chip`

**الغرض:** ده الـ main struct اللي بيمثّل bank واحدة من الـ GPIO في هاردوير Nomadik — كل bank فيها 32 pin.

```c
struct nmk_gpio_chip {
    struct gpio_chip chip;        /* Linux GPIO subsystem abstraction — must be first */
    void __iomem *addr;           /* base address of MMIO registers for this bank */
    struct clk *clk;              /* clock gate — must enable before register access */
    unsigned int bank;            /* bank index (0..NMK_MAX_BANKS-1) */
    void (*set_ioforce)(bool enable); /* platform callback to force I/O direction */
    spinlock_t lock;              /* protects all register reads/writes in this bank */
    bool sleepmode;               /* does this bank support sleep configuration? */
    bool is_mobileye_soc;         /* quirk flag for Mobileye SoC variant */
    /* shadow copies of interrupt/wakeup mask registers */
    u32 edge_rising;              /* bitmask: pins with rising-edge IRQ configured */
    u32 edge_falling;             /* bitmask: pins with falling-edge IRQ configured */
    u32 real_wake;                /* bitmask: pins that should actually wake the system */
    u32 rwimsc;                   /* shadow of NMK_GPIO_RWIMSC register */
    u32 fwimsc;                   /* shadow of NMK_GPIO_FWIMSC register */
    u32 rimsc;                    /* shadow of NMK_GPIO_RIMSC register */
    u32 fimsc;                    /* shadow of NMK_GPIO_FIMSC register */
    u32 pull_up;                  /* bitmask: which pins have pull-up enabled */
    u32 lowemi;                   /* bitmask: which pins are in low-EMI mode */
};
```

**الحقول المهمة بالتفصيل:**

| الحقل | النوع | الشرح |
|---|---|---|
| `chip` | `struct gpio_chip` | الـ Linux GPIO core struct — بيعمله embed مش pointer |
| `addr` | `void __iomem *` | عنوان الـ registers في الـ memory map |
| `clk` | `struct clk *` | الساعة — لازم تكون شغّالة عشان تكتب في الـ registers |
| `bank` | `unsigned int` | رقم الـ bank (0 = pins 0-31, 1 = pins 32-63, ...) |
| `set_ioforce` | function pointer | callback لـ PRCM لإجبار الـ I/O في sleep |
| `lock` | `spinlock_t` | spinlock بيحمي كل وصول للـ registers |
| `sleepmode` | `bool` | هل الـ bank دي بتدعم wakeup من sleep |
| `is_mobileye_soc` | `bool` | variant خاص — بيغيّر بعض التصرفات |
| `edge_rising/falling` | `u32` | bitmask — كل bit = pin واحد |
| `rwimsc / fwimsc` | `u32` | نسخة shadow من الـ wake-up interrupt mask registers |
| `rimsc / fimsc` | `u32` | نسخة shadow من interrupt mask registers العادية |
| `pull_up` | `u32` | tracking للـ pull-up — للمقارنة وقت sleep |
| `lowemi` | `u32` | shadow لـ `NMK_GPIO_LOWEMI` register |

**ملاحظة مهمة:** الـ shadow registers موجودة لأن بعض الـ MMIO registers write-only — مش ممكن تقراهم من الهاردوير، فالكود بيحتفظ بنسخة في RAM.

---

#### `struct prcm_gpiocr_altcx`

**الغرض:** بيصف entry واحدة لـ alternate-C function معينة في الـ PRCM GPIOCR register — حجمه 1 byte فقط بفضل `__packed`.

```c
struct prcm_gpiocr_altcx {
    bool used:1;       /* is this alt-C function actually used on this pin? */
    u8 reg_index:2;    /* which GPIOCR register (0=CR1, 1=CR2, 2=CR3) */
    u8 control_bit:5;  /* which bit in that register controls this function */
} __packed;
```

| الحقل | الـ Bits | الشرح |
|---|---|---|
| `used` | 1 bit | لو 0 يبقى الـ entry دي فاضية — بيتجاهلها الكود |
| `reg_index` | 2 bits | index في `prcm_gpiocr_registers[]` — أقصاه 3 |
| `control_bit` | 5 bits | رقم الـ bit داخل الـ register — أقصاه 31 |

الـ `__packed` ضروري لأن الـ struct بيُخزَّن في arrays كبيرة ثابتة في الـ SoC data — بيوفّر ذاكرة.

---

#### `struct prcm_gpiocr_altcx_pin_desc`

**الغرض:** بيصف pin معين ممكن يدعم واحدة أو أكتر من الـ alternate-C functions الإضافية عبر الـ PRCM.

```c
struct prcm_gpiocr_altcx_pin_desc {
    unsigned short pin;                                    /* global pin number */
    struct prcm_gpiocr_altcx altcx[PRCM_IDX_GPIOCR_ALTC_MAX]; /* 4 entries */
};
```

الـ `altcx` array فيها 4 slots — واحدة لكل C1/C2/C3/C4. لو `used == 0` في الـ slot يبقى الـ pin مش بيدعم الـ function دي. بيُستخدم مع الماكرو `PRCM_GPIOCR_ALTCX(...)` عشان تملي الـ array بشكل مقروء.

---

#### `struct nmk_function`

**الغرض:** بيصف function واحدة في نظام الـ pinmux — زي `uart0` أو `spi1`. بيُسجَّل في الـ pinctrl core.

```c
struct nmk_function {
    const char *name;               /* function name e.g. "uart0", "spi1" */
    const char * const *groups;     /* array of group names that can use this function */
    unsigned int ngroups;           /* count of groups */
};
```

---

#### `struct nmk_pingroup`

**الغرض:** بيصف group من الـ pins بتشارك نفس الـ altsetting — زي "uart0_pins". مبني فوق الـ generic `struct pingroup`.

```c
struct nmk_pingroup {
    struct pingroup grp;   /* generic: contains name + pins array + count */
    int altsetting;        /* NMK_GPIO_ALT_A/B/C to apply to all pins in this group */
};
```

الـ `altsetting` بيكون قيمة زي `NMK_GPIO_ALT_A` أو `NMK_GPIO_ALT_C` وبيتحط في الـ `AFSLA`/`AFSLB` registers.

---

#### `struct nmk_pinctrl_soc_data`

**الغرض:** ده الـ top-level descriptor للـ SoC كله من ناحية الـ pin controller — بيجمع كل الـ pins والـ functions والـ groups في struct واحد. ده الـ "database" اللي الـ driver بيشتغل منه.

```c
struct nmk_pinctrl_soc_data {
    const struct pinctrl_pin_desc *pins;          /* all pin descriptors for this SoC */
    unsigned int npins;
    const struct nmk_function *functions;          /* all mux functions */
    unsigned int nfunctions;
    const struct nmk_pingroup *groups;             /* all pin groups */
    unsigned int ngroups;
    const struct prcm_gpiocr_altcx_pin_desc *altcx_pins; /* pins with alt-C via PRCM */
    unsigned int npins_altcx;
    const u16 *prcm_gpiocr_registers;             /* PRCM GPIOCR register offsets */
};
```

---

#### `struct pingroup` (من `pinctrl/pinctrl.h`)

```c
struct pingroup {
    const char *name;              /* group name e.g. "uart0grp" */
    const unsigned int *pins;      /* array of pin numbers in this group */
    size_t npins;
};
```

---

#### `struct pinctrl_pin_desc` (من `pinctrl/pinctrl.h`)

```c
struct pinctrl_pin_desc {
    unsigned int number;   /* unique global pin number */
    const char *name;      /* human-readable name e.g. "GPIO0" */
    void *drv_data;        /* driver-specific data, pinctrl core won't touch it */
};
```

---

### علاقات الـ Structs — ASCII Diagram

```
nmk_pinctrl_soc_data
├─── pins[]  ──────────────► pinctrl_pin_desc[]
│                              ├── number
│                              ├── name
│                              └── drv_data
│
├─── functions[]  ─────────► nmk_function[]
│                              ├── name ("uart0", "spi1", ...)
│                              └── groups[] ──► (string array: group names)
│
├─── groups[]  ────────────► nmk_pingroup[]
│                              ├── grp (pingroup)
│                              │    ├── name
│                              │    ├── pins[] ──► (u32 array: pin numbers)
│                              │    └── npins
│                              └── altsetting (NMK_GPIO_ALT_*)
│
├─── altcx_pins[]  ────────► prcm_gpiocr_altcx_pin_desc[]
│                              ├── pin
│                              └── altcx[4]  ──► prcm_gpiocr_altcx
│                                                  ├── used (1 bit)
│                                                  ├── reg_index (2 bits)
│                                                  └── control_bit (5 bits)
│
└─── prcm_gpiocr_registers[]  ──► u16[] (PRCM register offsets)
                                    [0] → GPIOCR1 offset
                                    [1] → GPIOCR2 offset
                                    [2] → GPIOCR3 offset


nmk_gpio_chip
├─── chip (gpio_chip) ◄──── Linux GPIO core يتعامل مع الـ bank من هنا
│         └── .base = bank * 32
│
├─── addr  ──────────────────► MMIO base
│         + NMK_GPIO_DAT   (0x00)
│         + NMK_GPIO_DIRS  (0x14)
│         + NMK_GPIO_AFSLA (0x20)
│         + NMK_GPIO_RIMSC (0x40)
│         + ...
│
├─── clk   ──────────────────► struct clk (clock framework)
│
└─── lock  ──────────────────► spinlock (protects everything above)


nmk_gpio_chips[NMK_MAX_BANKS]   ← global array, protected by nmk_gpio_slpm_lock
  [0]  →  nmk_gpio_chip  (bank 0,  pins   0..31)
  [1]  →  nmk_gpio_chip  (bank 1,  pins  32..63)
  [2]  →  nmk_gpio_chip  (bank 2,  pins  64..95)
  ...
  [15] →  nmk_gpio_chip  (bank 15, pins 480..511)
```

---

### مخطط دورة الحياة (Lifecycle)

#### إنشاء وتسجيل الـ GPIO Bank

```
Platform/DT Boot
      │
      ▼
gpio-nomadik.c :: platform_driver.probe()
      │
      ├── nmk_gpio_populate_chip(fwnode, pdev)
      │     ├── kzalloc(nmk_gpio_chip)
      │     ├── ioremap(res)  ─────────────────► nmk_chip->addr
      │     ├── clk_get()     ─────────────────► nmk_chip->clk
      │     ├── clk_prepare_enable()
      │     ├── chip.base = bank * NMK_GPIO_PER_CHIP
      │     ├── chip.ngpio = NMK_GPIO_PER_CHIP (32)
      │     ├── spin_lock_init(&nmk_chip->lock)
      │     └── nmk_gpio_chips[bank] = nmk_chip   ← global registration
      │
      ├── gpiochip_add_data(&nmk_chip->chip, nmk_chip)
      │     └── يسجّل الـ 32 GPIO في Linux GPIO subsystem
      │
      └── devm_request_irq() ──► bank ISR registered

[System Running — normal operations]
      │
      ├── gpio_direction_input/output()
      │     └── writel(NMK_GPIO_DIRS/DIRC)
      │
      └── gpiod_set_value() / gpiod_get_value()
            └── writel(NMK_GPIO_DATS/DATC) / readl(NMK_GPIO_DAT)

[System Suspend]
      ├── save rimsc, fimsc → shadow
      ├── program NMK_GPIO_RWIMSC / FWIMSC for wakeup pins
      └── set NMK_GPIO_SLPC per pin (sleep mode config)

[System Resume]
      └── restore rimsc, fimsc from shadows

[Driver Remove]
      ├── gpiochip_remove()
      ├── clk_disable_unprepare()
      └── iounmap(addr)
```

#### دورة حياة الـ Pinctrl SoC Data

```
[Compile Time — static const data]
      │
      ├── SoC file defines static nmk_pinctrl_soc_data:
      │     ├── pins[]      (all pin descriptors)
      │     ├── functions[] (uart0, spi1, i2c0, ...)
      │     ├── groups[]    (uart0grp, spi1grp, ...)
      │     └── altcx_pins[] (PRCM-controlled pins)
      │
      └── init function just sets *soc = &soc_data

[pinctrl-nomadik driver probe()]
      │
      ├── nmk_pinctrl_db8500_init(&soc)
      │     └── fills soc pointer with static data
      │
      ├── pinctrl_register_and_init(&pctldesc, dev, drvdata, &pctldev)
      │     ├── pctldesc.pins  = soc->pins
      │     ├── pctldesc.npins = soc->npins
      │     └── registers pctlops / pmxops / confops vtables
      │
      └── pinctrl_enable(pctldev)
            └── pinctrl subsystem is now live
```

---

### مخططات تدفق الاستدعاء (Call Flow)

#### تغيير الـ Alternate Function لـ Pin

```
consumer driver calls pinctrl_select_state(pinctrl, state)
  → pinctrl core iterates map table entries
    → pinmux_ops->set_mux(pctldev, func_selector, group_selector)
      → nmk_pmx_set() in pinctrl-nomadik.c
        → lookup nmk_pingroup[group_selector].altsetting
          → nmk_gpio_populate_chip() → nmk_chip for that bank
            → clk_enable(nmk_chip->clk)
            → spin_lock_irqsave(&nmk_chip->lock)
              → readl(addr + NMK_GPIO_AFSLA)
              → readl(addr + NMK_GPIO_AFSLB)
              → modify bits per altsetting:
                  ALT_A → AFSLA[bit]=1, AFSLB[bit]=0
                  ALT_B → AFSLA[bit]=0, AFSLB[bit]=1
                  ALT_C → AFSLA[bit]=1, AFSLB[bit]=1
              → writel(new_afsla, addr + NMK_GPIO_AFSLA)
              → writel(new_afslb, addr + NMK_GPIO_AFSLB)
            → spin_unlock_irqrestore()
            → clk_disable()

        [if Alt-C sub-variant C1..C4 needed]
        → lookup pin in soc->altcx_pins[]
          → find prcm_gpiocr_altcx entry with used==1
            → reg_index → soc->prcm_gpiocr_registers[reg_index]
              → read PRCM base + register offset
              → set/clear control_bit
              → write back to PRCM register
```

#### قراءة/كتابة GPIO Value

```
user/driver calls gpiod_set_value(desc, 1)
  → gpiolib core resolves desc → nmk_gpio_chip
    → gpio_chip->set(chip, offset, 1)
      → nmk_gpio_set() in gpio-nomadik.c
        → spin_lock_irqsave(&nmk_chip->lock)
          → val=1: writel(BIT(offset), addr + NMK_GPIO_DATS)
          → val=0: writel(BIT(offset), addr + NMK_GPIO_DATC)
        → spin_unlock_irqrestore()

user/driver calls gpiod_get_value(desc)
  → gpio_chip->get(chip, offset)
    → nmk_gpio_get() in gpio-nomadik.c
      → readl(addr + NMK_GPIO_DAT)
        → return (val >> offset) & 1
```

#### معالجة الـ Interrupt

```
Hardware GPIO edge detected on pin N
      │
      ▼
GIC delivers IRQ to bank ISR
      │
      ▼
nmk_gpio_irq_handler()
  → spin_lock(&nmk_chip->lock)
    → status = readl(addr + NMK_GPIO_IS)
    → writel(status, addr + NMK_GPIO_IC)   ← clear all pending
  → spin_unlock(&nmk_chip->lock)
  → for each bit set in status:
      → generic_handle_irq(child_irq_for_pin)
        → consumer's irq handler runs
  → return IRQ_HANDLED
```

#### تفعيل الـ Sleep Mode

```
kernel PM calls nmk_gpio_suspend()
  → spin_lock(&nmk_gpio_slpm_lock)        ← global lock first
    → spin_lock(&nmk_chip->lock)           ← per-chip lock second
      → save rimsc, fimsc (for restore)
      → compute new wake masks from real_wake bitmask
      → writel(new_rwimsc, addr + NMK_GPIO_RWIMSC)
      → writel(new_fwimsc, addr + NMK_GPIO_FWIMSC)
      → if sleepmode:
          → for each pin:
              → __nmk_gpio_set_slpm(chip, pin, mode)
                → writel modified NMK_GPIO_SLPC
    → spin_unlock(&nmk_chip->lock)
  → spin_unlock(&nmk_gpio_slpm_lock)

kernel PM calls nmk_gpio_resume()
  → spin_lock(&nmk_gpio_slpm_lock)
    → spin_lock(&nmk_chip->lock)
      → writel(rimsc, addr + NMK_GPIO_RIMSC)   ← restore
      → writel(fimsc, addr + NMK_GPIO_FIMSC)
    → spin_unlock(&nmk_chip->lock)
  → spin_unlock(&nmk_gpio_slpm_lock)
```

#### Debug Show (debugfs)

```
user reads /sys/kernel/debug/gpio
  → gpiolib calls chip->dbg_show() per pin
    → nmk_gpio_dbg_show_one(s, pctldev, chip, offset)
      [only compiled if CONFIG_DEBUG_FS]
      → readl(addr + NMK_GPIO_DAT)    ← current value
      → readl(addr + NMK_GPIO_DIR)    ← direction
      → readl(addr + NMK_GPIO_PDIS)   ← pull state
      → readl(addr + NMK_GPIO_AFSLA)  ← alt function A bit
      → readl(addr + NMK_GPIO_AFSLB)  ← alt function B bit
        → if AFSLA==1 && AFSLB==1:
            → nmk_prcm_gpiocr_get_mode(pctldev, gpio)
              → lookup altcx_pins → determine C1/C2/C3/C4
        → seq_printf(s, formatted pin state)
```

---

### استراتيجية الـ Locking

#### الـ Locks الموجودة

| Lock | النوع | بيحمي إيه |
|---|---|---|
| `nmk_gpio_chip.lock` | `spinlock_t` (per-bank) | كل الـ MMIO register access + كل الـ shadow registers لهذا الـ bank |
| `nmk_gpio_slpm_lock` | `spinlock_t` (global) | تسلسل عمليات الـ sleep mode عبر كل الـ banks + الـ `nmk_gpio_chips[]` global array |

#### قواعد الـ Locking

**الـ `nmk_chip->lock`:**
- لازم يُمسك قبل أي `readl`/`writel` على الـ MMIO registers.
- بيُمسك في interrupt context كمان (spinlock مش mutex) — لأن الـ ISR نفسه بيحتاج يقرأ `NMK_GPIO_IS` ويكتب `NMK_GPIO_IC`.
- الـ shadow registers كلها (rimsc, fimsc, rwimsc, fwimsc, edge_rising, edge_falling, pull_up, lowemi) محمية بنفس الـ lock.

**الـ `nmk_gpio_slpm_lock`:**
- global lock يحمي **تسلسل** تغيير الـ sleep mode على أكتر من bank في نفس الوقت.
- السبب: عملية الـ suspend بتمشي على كل الـ banks بالترتيب، ولازم ما يحصلش race بين الـ pinctrl driver والـ gpio driver على نفس الـ `nmk_gpio_chips[]`.

#### ترتيب الـ Locks (Lock Ordering)

```
إذا احتجت الاتنين في نفس الوقت، لازم الترتيب ده دايماً:

    nmk_gpio_slpm_lock    (outer — global)
           │
           ▼
    nmk_chip->lock        (inner — per-bank)

عكس الترتيب ده = deadlock
```

#### دالتان بدون Lock داخلي

```c
/* caller MUST hold nmk_chip->lock before calling these */
void __nmk_gpio_make_output(struct nmk_gpio_chip *nmk_chip,
                            unsigned int offset, int val);

void __nmk_gpio_set_slpm(struct nmk_gpio_chip *nmk_chip,
                         unsigned int offset, enum nmk_gpio_slpm mode);
```

الـ prefix `__` في اسمهم اصطلاح kernel معناه **"caller holds the lock"** — مش safe تتنادي عليهم مباشرة من غير ما تمسك `nmk_chip->lock` الأول.

#### جدول ملخص الـ Locking

| العملية | `slpm_lock` | `chip->lock` | الـ IRQ Context |
|---|---|---|---|
| `direction_output/input` | لأ | نعم — `irqsave` | لأ |
| `get/set value` | لأ | نعم — `irqsave` | لأ |
| `__nmk_gpio_set_slpm` | بيتحكم فيه الـ caller | بيتحكم فيه الـ caller | — |
| `nmk_gpio_suspend` | نعم | نعم | لأ |
| `nmk_gpio_resume` | نعم | نعم | لأ |
| IRQ handler | لأ | نعم — `lock` فقط | نعم (already in IRQ) |
| `nmk_prcm_gpiocr_get_mode` | لأ | لأ | — |
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

| Function / Symbol | النوع | الغرض |
|---|---|---|
| `nmk_pinctrl_stn8815_init()` | Init API | تهيئة SoC data لـ STN8815 |
| `nmk_pinctrl_db8500_init()` | Init API | تهيئة SoC data لـ DB8500 |
| `nmk_pinctrl_db8540_init()` | Init API | تهيئة SoC data لـ DB8540 |
| `nmk_gpio_dbg_show_one()` | Debug API | طباعة حالة pin واحد في debugfs |
| `__nmk_gpio_make_output()` | Runtime Helper | تحويل GPIO لـ output وضبط قيمته |
| `__nmk_gpio_set_slpm()` | Runtime Helper | ضبط sleep mode لـ GPIO |
| `nmk_gpio_populate_chip()` | Init Helper | إنشاء وتسجيل `nmk_gpio_chip` من fwnode |
| `nmk_prcm_gpiocr_get_mode()` | Query Helper | قراءة alternate-C mode من PRCM GPIOCR |
| `NMK_PIN_GROUP(a, b)` | Macro | تعريف `nmk_pingroup` من اسم المصفوفة والـ altsetting |
| `PRCM_GPIOCR_ALTCX(...)` | Macro | تعريف `prcm_gpiocr_altcx_pin_desc` بشكل مضغوط |

#### Registers Map

| Register | Offset | الوظيفة |
|---|---|---|
| `NMK_GPIO_DAT` | 0x00 | قراءة قيمة الـ GPIO |
| `NMK_GPIO_DATS` | 0x04 | Set bits (write 1 to set) |
| `NMK_GPIO_DATC` | 0x08 | Clear bits (write 1 to clear) |
| `NMK_GPIO_PDIS` | 0x0C | Pull disable register |
| `NMK_GPIO_DIR` | 0x10 | Direction register |
| `NMK_GPIO_DIRS` | 0x14 | Direction Set |
| `NMK_GPIO_DIRC` | 0x18 | Direction Clear |
| `NMK_GPIO_SLPC` | 0x1C | Sleep mode configuration |
| `NMK_GPIO_AFSLA` | 0x20 | Alternate Function Select A |
| `NMK_GPIO_AFSLB` | 0x24 | Alternate Function Select B |
| `NMK_GPIO_LOWEMI` | 0x28 | Low EMI mode |
| `NMK_GPIO_RIMSC` | 0x40 | Rising edge interrupt mask |
| `NMK_GPIO_FIMSC` | 0x44 | Falling edge interrupt mask |
| `NMK_GPIO_IS` | 0x48 | Interrupt Status |
| `NMK_GPIO_IC` | 0x4C | Interrupt Clear |
| `NMK_GPIO_RWIMSC` | 0x50 | Rising Wake Interrupt Mask |
| `NMK_GPIO_FWIMSC` | 0x54 | Falling Wake Interrupt Mask |
| `NMK_GPIO_WKS` | 0x58 | Wake Status |
| `NMK_GPIO_EDGELEVEL` | 0x5C | Edge/Level select (DB8540+) |
| `NMK_GPIO_LEVEL` | 0x60 | Level register (DB8540+) |

---

### Group 1: SoC Initialization APIs

الهدف من المجموعة دي هو إنك تربط الـ pinctrl driver بالـ SoC-specific data (pins, groups, functions, PRCM registers). كل SoC عنده implementation مختلفة للـ `nmk_pinctrl_soc_data`، والـ functions دي بتملّي الـ pointer بيها. التصميم بيعتمد على الـ `CONFIG_PINCTRL_*` Kconfig guards عشان يمنع تضمين بيانات SoC مش موجود على الـ target.

---

#### `nmk_pinctrl_stn8815_init()`

```c
void nmk_pinctrl_stn8815_init(const struct nmk_pinctrl_soc_data **soc);
```

بتملّي الـ pointer اللي بيشاور على `nmk_pinctrl_soc_data` ببيانات الـ STN8815 SoC الخاصة بالـ Nomadik NHK-15 platform. بتعمل تهيئة static لـ table الـ pins والـ groups والـ functions الخاصة بالـ SoC ده. لو `CONFIG_PINCTRL_STN8815` مش enabled، بيُستبدل بـ inline stub فاضي.

**Parameters:**
- `soc`: pointer لـ pointer من النوع `const struct nmk_pinctrl_soc_data *`، الـ function بتكتب فيه الـ address بتاع الـ SoC data.

**Return value:** void.

**Key details:**
- بتُستدعى من الـ `probe` الخاص بالـ pinctrl driver قبل ما يبدأ يسجّل نفسه في الـ pinctrl core.
- الـ stub inline بيمنع linker errors لو الـ Kconfig option مش محددة.
- مفيش locking هنا لأنها بتحصل في init time فقط.

**Who calls it:** `pinctrl-nomadik.c` من جوه `probe()` قبل `pinctrl_register_and_init()`.

---

#### `nmk_pinctrl_db8500_init()`

```c
void nmk_pinctrl_db8500_init(const struct nmk_pinctrl_soc_data **soc);
```

نفس الفكرة بالظبط لكن للـ DB8500 (ST-Ericsson U8500 SoC) — أشهر SoC في العيلة دي واللي اتستخدم في موبايلات Samsung وSony قديمًا. بتملّي الـ `soc` pointer ببيانات الـ DB8500 الخاصة من الـ `pinctrl-db8500.c`.

**Parameters:**
- `soc`: نفس الـ double-pointer.

**Return value:** void.

**Key details:**
- DB8500 هو الـ SoC الأكثر شيوعًا في الـ Nomadik family وعنده أكبر عدد من الـ pins والـ groups.
- مشروطة بـ `CONFIG_PINCTRL_DB8500`.

**Who calls it:** `pinctrl-nomadik.c`.

---

#### `nmk_pinctrl_db8540_init()`

```c
void nmk_pinctrl_db8540_init(const struct nmk_pinctrl_soc_data **soc);
```

نفس النمط للـ DB8540، وده الـ SoC اللي أضاف الـ `NMK_GPIO_EDGELEVEL` و`NMK_GPIO_LEVEL` registers. بتفرق عن الـ DB8500 في إن بعض الـ pins عندها capabilities إضافية في الـ level-triggered interrupts.

**Parameters:**
- `soc`: نفس الـ double-pointer.

**Return value:** void.

**Key details:**
- DB8540 بيدعم الـ `NMK_GPIO_EDGELEVEL` (0x5C) و`NMK_GPIO_LEVEL` (0x60) اللي مش موجودين في الـ SoCs القديمة.
- مشروطة بـ `CONFIG_PINCTRL_DB8540`.

**Who calls it:** `pinctrl-nomadik.c`.

---

**Pseudocode flow للـ SoC init sequence:**

```
probe() in pinctrl-nomadik.c
    │
    ├── nmk_pinctrl_stn8815_init(&soc)  // أو db8500 أو db8540
    │       └── *soc = &nmk_stn8815_soc_data   // static table assignment
    │
    ├── pinctrl_register_and_init(&pctldesc, dev, nmkp, &nmkp->pctldev)
    │       └── pinctrl core يستخدم soc->pins, soc->groups, soc->functions
    │
    └── pinctrl_enable(nmkp->pctldev)
```

---

### Group 2: Runtime GPIO Helpers (Internal / Cross-Driver)

الـ functions دي مُعلنة بـ `__` prefix عشان توضح إنها **internal helpers** مش مفروض تُستدعى إلا من جوه الـ gpio-nomadik وpinctrl-nomadik نفسهم. بتشتغل مباشرة على الـ hardware registers بدون ما تمر بالـ GPIO framework العادي، عشان كده لازم المتصل يمسك الـ `nmk_chip->lock` قبل ما يكال الفانكشن دي.

---

#### `__nmk_gpio_make_output()`

```c
void __nmk_gpio_make_output(struct nmk_gpio_chip *nmk_chip,
                            unsigned int offset, int val);
```

بتحول الـ GPIO pin المحدد من input لـ output وفي نفس الوقت بتحدد قيمته (high أو low). بتكتب في الـ `NMK_GPIO_DIRS` (direction set) أو `NMK_GPIO_DIRC` (direction clear) وبعدين في `NMK_GPIO_DATS` أو `NMK_GPIO_DATC` لضبط القيمة. الكتابة بتتم بـ set/clear registers عشان تكون atomic ومتجنبة الـ read-modify-write.

**Parameters:**
- `nmk_chip`: الـ chip descriptor اللي فيه الـ base address وباقي info.
- `offset`: رقم الـ pin جوه الـ bank (0 → 31).
- `val`: القيمة المطلوبة — 0 لـ low، أي قيمة غير صفر لـ high.

**Return value:** void.

**Key details:**
- **Locking**: المتصل **لازم** يمسك `nmk_chip->lock` قبل الاستدعاء.
- بتستخدم الـ set/clear register pattern بدل الـ read-modify-write لتجنب الـ race conditions على مستوى الـ hardware.
- بتُستدعى من الـ pinctrl driver لما يحتاج يتحكم في GPIO pin كـ output مؤقتًا أثناء الـ mux operations.

**Who calls it:** `pinctrl-nomadik.c` — خصوصًا أثناء ضبط الـ alternate functions.

---

#### `__nmk_gpio_set_slpm()`

```c
void __nmk_gpio_set_slpm(struct nmk_gpio_chip *nmk_chip, unsigned int offset,
                         enum nmk_gpio_slpm mode);
```

بتضبط **sleep mode** للـ GPIO pin — يعني إيه اللي يحصل للـ pin لما الـ SoC يدخل sleep. القيمة `NMK_GPIO_SLPM_INPUT` بتخلي الـ pin input لما يدخل sleep، و`NMK_GPIO_SLPM_NOCHANGE` بيخليه زي ما هو. بتكتب في الـ `NMK_GPIO_SLPC` register.

**Parameters:**
- `nmk_chip`: الـ chip descriptor.
- `offset`: رقم الـ pin جوه الـ bank.
- `mode`: enum من `nmk_gpio_slpm`:
  - `NMK_GPIO_SLPM_INPUT` (= `NMK_GPIO_SLPM_WAKEUP_ENABLE`): الـ pin يبقى input في الـ sleep ويقدر يصحّي الـ SoC.
  - `NMK_GPIO_SLPM_NOCHANGE` (= `NMK_GPIO_SLPM_WAKEUP_DISABLE`): الـ pin ما بيتغيرش في الـ sleep.

**Return value:** void.

**Key details:**
- **Locking**: المتصل لازم يمسك `nmk_chip->lock` **و** الـ global `nmk_gpio_slpm_lock`.
- الـ `nmk_gpio_slpm_lock` موجود عشان الـ sleep configuration بتأثر على أكتر من bank في نفس الوقت.
- الـ enum aliases (`WAKEUP_ENABLE` = `INPUT`) بيوضح إن الـ sleep input mode هو اللي بيسمح بالـ wakeup.

**Who calls it:** `pinctrl-nomadik.c` أثناء ضبط الـ sleep states، وكمان من الـ suspend/resume path.

---

### Group 3: Chip Population / Registration

---

#### `nmk_gpio_populate_chip()`

```c
struct nmk_gpio_chip *nmk_gpio_populate_chip(struct fwnode_handle *fwnode,
                                             struct platform_device *pdev);
```

بتنشئ وتملّي `nmk_gpio_chip` من معلومات الـ firmware node (DT أو ACPI). بتعمل map للـ I/O registers، بتجيب الـ clock، وبتسجّل الـ `gpio_chip` في الـ kernel GPIO subsystem. هي نقطة الدخول الأساسية لتهيئة كل bank من الـ GPIO banks الخاصة بالـ Nomadik.

**Parameters:**
- `fwnode`: الـ firmware node descriptor (من DT عادةً) اللي بيحتوي على معلومات الـ GPIO controller.
- `pdev`: الـ platform device المرتبط بالـ bank ده.

**Return value:**
- pointer لـ `nmk_gpio_chip` جديد لو نجح.
- `ERR_PTR(-errno)` لو فشل (مثلاً: فشل الـ ioremap، أو فشل الـ clock، أو فشل `gpiochip_add_data()`).

**Key details:**
- بتخزن الـ chip في الـ global array `nmk_gpio_chips[bank]` اللي بيستخدمه الـ pinctrl driver للوصول للـ chip.
- بتعمل `clk_prepare_enable()` للـ GPIO clock.
- بتُهيّئ الـ `spinlock_t lock` الخاص بالـ chip.
- بتملّي الـ `gpio_chip` callbacks (direction_input, direction_output, get, set, إلخ).

**Pseudocode flow:**

```
nmk_gpio_populate_chip(fwnode, pdev)
    │
    ├── devm_kzalloc() → nmk_chip
    ├── of_property_read_u32("gpio-bank", &nmk_chip->bank)
    ├── devm_ioremap_resource() → nmk_chip->addr
    ├── devm_clk_get() → nmk_chip->clk
    ├── clk_prepare_enable(nmk_chip->clk)
    ├── spin_lock_init(&nmk_chip->lock)
    ├── nmk_chip->chip = { .label, .ngpio=32, .base, callbacks... }
    ├── gpiochip_add_data(&nmk_chip->chip, nmk_chip)
    ├── nmk_gpio_chips[nmk_chip->bank] = nmk_chip   // تسجيل global
    └── return nmk_chip
```

**Who calls it:** الـ `gpio-nomadik.c` probe function، وكمان الـ `pinctrl-nomadik.c` لما يحتاج يجيب chip لم يتسجل بعد.

---

### Group 4: Debug Interface

---

#### `nmk_gpio_dbg_show_one()`

```c
void nmk_gpio_dbg_show_one(struct seq_file *s, struct pinctrl_dev *pctldev,
                           struct gpio_chip *chip, unsigned int offset);
```

بتطبع معلومات تفصيلية عن حالة GPIO pin واحد في `debugfs`. بتقرأ الـ registers مباشرة من الـ hardware وبتعرض: الاتجاه (input/output)، القيمة الحالية، الـ alternate function المفعّل، حالة الـ pull، وحالة الـ sleep mode. بتُستدعى من الـ `gpiochip_dbg_show` callback.

**Parameters:**
- `s`: الـ `seq_file` اللي بيُكتب فيه الـ output.
- `pctldev`: الـ pinctrl device — بيُستخدم لعرض معلومات الـ pinmux.
- `chip`: الـ `gpio_chip` الخاص بالـ bank.
- `offset`: رقم الـ pin جوه الـ bank.

**Return value:** void.

**Key details:**
- مشروطة بـ `CONFIG_DEBUG_FS` — لو مش enabled، الـ stub الـ inline فارغ تمامًا.
- بتمسك الـ `nmk_chip->lock` أثناء قراءة الـ registers لضمان consistency.
- الـ symbol ده مُعلن في الـ header عشان `pinctrl-nomadik.c` يقدر يستخدمه في الـ `pin_dbg_show` callback الخاص به، مع إن التنفيذ موجود في `gpio-nomadik.c`.

**Who calls it:** `pinctrl-nomadik.c` من الـ `pinctrl_ops.pin_dbg_show` callback، وكمان الـ GPIO core مباشرة.

---

### Group 5: PRCM Query Helper

---

#### `nmk_prcm_gpiocr_get_mode()`

```c
int __maybe_unused nmk_prcm_gpiocr_get_mode(struct pinctrl_dev *pctldev,
                                            int gpio);
```

بتقرأ الـ alternate-C sub-mode (C1, C2, C3, C4) من الـ PRCM GPIOCR registers لـ pin معين. الـ Nomadik SoC بيستخدم الـ PRCM (Power Reset Clock Manager) عشان يتحكم في الـ extended alternate functions. الـ function دي بتسكان الـ `altcx_pins` table وبتقرأ الـ PRCM register المناسب.

**Parameters:**
- `pctldev`: الـ pinctrl device — بيُستخدم للوصول لـ driver data الخاص بالـ nmk pinctrl.
- `gpio`: رقم الـ GPIO pin في الـ global numbering space.

**Return value:**
- رقم الـ alternate-C sub-mode (1..4) لو الـ pin عنده ALTCX مفعّل.
- 0 لو الـ pin ده مش في وضع ALTCX.
- قيمة سالبة (error code) لو فشل.

**Key details:**
- مشروطة بـ `CONFIG_PINCTRL_NOMADIK`.
- الـ `__maybe_unused` attribute موجود عشان الـ compiler ما يشكيش لو الـ caller غير موجود في بعض الـ configs.
- بتفرّق بين 3 PRCM registers: `PRCM_IDX_GPIOCR1`, `PRCM_IDX_GPIOCR2`, `PRCM_IDX_GPIOCR3` عن طريق الـ `reg_index` المخزون في `prcm_gpiocr_altcx.reg_index`.
- الـ bit المستخدم في كل register محدد في `prcm_gpiocr_altcx.control_bit`.

**Who calls it:** `pinctrl-nomadik.c` من الـ debug show functions وأثناء الـ mux configuration للـ ALTCX pins.

---

### Group 6: Global Shared Symbols

الـ symbols دي مش functions، بس بتُعلَن في الـ header كـ `extern` عشان تُشارك بين الـ `gpio-nomadik.c` والـ `pinctrl-nomadik.c`.

---

#### `nmk_gpio_chips[]`

```c
extern struct nmk_gpio_chip *nmk_gpio_chips[NMK_MAX_BANKS];
```

مصفوفة global بتخزن pointers لكل `nmk_gpio_chip` bank تم تسجيله. الـ pinctrl driver بيستخدمها للوصول لأي bank بـ index مباشر بدل ما يبحث عنه. **الـ `NMK_MAX_BANKS`** = `DIV_ROUND_UP(512, 32)` = 16.

**Key details:**
- مشروطة بـ `CONFIG_PINCTRL_NOMADIK`.
- الكتابة فيها تحصل فقط من `nmk_gpio_populate_chip()`.
- القراءة منها من الـ pinctrl driver مشمولة بالـ `nmk_gpio_slpm_lock` أو الـ chip lock حسب السياق.

---

#### `nmk_gpio_slpm_lock`

```c
extern spinlock_t nmk_gpio_slpm_lock;
```

**الـ `nmk_gpio_slpm_lock`** هو Global spinlock بيحمي عمليات الـ sleep mode اللي بتأثر على أكتر من bank. لازم يتمسك مع الـ chip lock الخاص بكل bank لما بتعمل `__nmk_gpio_set_slpm()` على أكتر من pin في نفس الوقت.

**Key details:**
- مشروط بـ `CONFIG_PINCTRL_NOMADIK`.
- الـ locking order المحدد: **`nmk_gpio_slpm_lock` أولاً، ثم `nmk_chip->lock`** لتجنب الـ deadlock.
- بيُعرَّف في `gpio-nomadik.c` ومُعلن هنا عشان `pinctrl-nomadik.c` يستخدمه.

---

### Group 7: Data Structures & Macros

---

#### `struct nmk_gpio_chip`

```c
struct nmk_gpio_chip {
    struct gpio_chip chip;        // GPIO framework descriptor (يجب أن يكون أول عضو)
    void __iomem *addr;           // base address للـ GPIO bank registers
    struct clk *clk;              // clock reference
    unsigned int bank;            // رقم الـ bank (0..15)
    void (*set_ioforce)(bool);    // callback للتحكم في IOForce (DB8500)
    spinlock_t lock;              // يحمي الـ register access
    bool sleepmode;               // هل الـ chip يدعم sleep mode؟
    bool is_mobileye_soc;         // variant flag للـ Mobileye SoC
    /* shadow registers لتتبع الحالة */
    u32 edge_rising;              // الـ pins المضبوطة على rising edge
    u32 edge_falling;             // الـ pins المضبوطة على falling edge
    u32 real_wake;                // الـ pins اللي هي فعلاً wakeup sources
    u32 rwimsc;                   // Rising Wake Interrupt Mask shadow
    u32 fwimsc;                   // Falling Wake Interrupt Mask shadow
    u32 rimsc;                    // Rising Interrupt Mask shadow
    u32 fimsc;                    // Falling Interrupt Mask shadow
    u32 pull_up;                  // الـ pins اللي عليها pull-up
    u32 lowemi;                   // الـ pins في Low EMI mode
};
```

الـ shadow registers (rwimsc, fwimsc, rimsc, fimsc) ضرورية عشان الـ hardware registers دي write-only في بعض الأحيان، وكمان عشان نحتاج نرجعلها لما نطلع من الـ sleep.

---

#### `PRCM_GPIOCR_ALTCX()` Macro

```c
#define PRCM_GPIOCR_ALTCX(pin_num,          \
    altc1_used, altc1_ri, altc1_cb,          \
    altc2_used, altc2_ri, altc2_cb,          \
    altc3_used, altc3_ri, altc3_cb,          \
    altc4_used, altc4_ri, altc4_cb)
```

الـ macro ده بيختصر تعريف `prcm_gpiocr_altcx_pin_desc` اللي فيها 4 entries من `prcm_gpiocr_altcx`. كل alternate-C sub-function (C1..C4) بتحتاج:
- `used`: هل الـ function دي موجودة على الـ pin ده؟
- `reg_index`: أي PRCM GPIOCR register (1..3) بيتحكم فيها؟
- `control_bit`: أي bit في الـ register ده؟

مثال استخدام:

```c
/* pin 23: يدعم ALTC1 عن طريق GPIOCR1 bit 4 */
PRCM_GPIOCR_ALTCX(23,
    1, PRCM_IDX_GPIOCR1, 4,   /* ALTC1 */
    0, 0, 0,                   /* ALTC2 غير مستخدم */
    0, 0, 0,                   /* ALTC3 غير مستخدم */
    0, 0, 0),                  /* ALTC4 غير مستخدم */
```

---

#### `NMK_GPIO_ALT_*` — Alternate Function Encoding

```c
#define NMK_GPIO_ALT_GPIO   0          // GPIO mode
#define NMK_GPIO_ALT_A      1          // AFSLA=1, AFSLB=0
#define NMK_GPIO_ALT_B      2          // AFSLA=0, AFSLB=1
#define NMK_GPIO_ALT_C      (1|2)      // AFSLA=1, AFSLB=1 → hardware alt-C
#define NMK_GPIO_ALT_CX_SHIFT 2
#define NMK_GPIO_ALT_C1     ((1<<2)|NMK_GPIO_ALT_C)   // 0x07
#define NMK_GPIO_ALT_C2     ((2<<2)|NMK_GPIO_ALT_C)   // 0x0B
#define NMK_GPIO_ALT_C3     ((3<<2)|NMK_GPIO_ALT_C)   // 0x0F
#define NMK_GPIO_ALT_C4     ((4<<2)|NMK_GPIO_ALT_C)   // 0x13
```

**الـ encoding بيشتغل كده:**

```
AFSLA bit | AFSLB bit | الوضع
    0     |     0     | GPIO
    1     |     0     | Alt-A
    0     |     1     | Alt-B
    1     |     1     | Alt-C (base)
```

لما بيبقى الـ altsetting = `NMK_GPIO_ALT_C1..C4`، الـ lower 2 bits بتضبط الـ AFSLA/AFSLB على 1 (Alt-C في الـ GPIO block)، والـ upper bits بتوجّه الـ pinctrl driver يكتب في الـ PRCM GPIOCR عشان يختار الـ C sub-variant الصح.

---

#### `NMK_PIN_GROUP()` Macro

```c
#define NMK_PIN_GROUP(a, b) \
    { \
        .grp = PINCTRL_PINGROUP(#a, a##_pins, ARRAY_SIZE(a##_pins)), \
        .altsetting = b, \
    }
```

مثال:

```c
/* في pinctrl-db8500.c */
static const unsigned int i2c0_pins[] = { 147, 148 };
static const struct nmk_pingroup nmk_db8500_groups[] = {
    NMK_PIN_GROUP(i2c0, NMK_GPIO_ALT_A),
    /* ... */
};
```

الـ macro بيتوقع إن في مصفوفة اسمها `a##_pins` معرّفة قبله، وبيستخدم `PINCTRL_PINGROUP` من الـ pinctrl core لتعريف الـ `pingroup` المضمّن.

---

### Cross-Driver Architecture

```
gpio-nomadik.c                    pinctrl-nomadik.c
      │                                  │
      │  defines: nmk_gpio_chips[]       │
      │  defines: nmk_gpio_slpm_lock     │
      │  defines: nmk_gpio_dbg_show_one()│
      │  defines: __nmk_gpio_make_output()
      │  defines: __nmk_gpio_set_slpm()  │
      │  defines: nmk_gpio_populate_chip()
      │                                  │
      └────────── uses ──────────────────┘
                     │
              gpio-nomadik.h
         (الـ header ده هو الجسر بينهم)
```

الـ design pattern هنا بيفصل الـ GPIO controller (gpio-nomadik) عن الـ pinmux/pinconf controller (pinctrl-nomadik)، مع إنهم بيشتركوا في نفس الـ hardware. الـ header بيعلن الـ symbols المشتركة بدل ما يكرر الكود في الملفين.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. مدخلات الـ debugfs المتعلقة بـ Nomadik GPIO

الـ kernel لما بيكون `CONFIG_DEBUG_FS=y` وكمان `CONFIG_PINCTRL_NOMADIK` أو `CONFIG_GPIO_NOMADIK` متفعّلين، بيظهر في `/sys/kernel/debug/`:

| المسار | المحتوى |
|--------|---------|
| `/sys/kernel/debug/gpio` | حالة كل GPIO: direction, value, IRQ if any |
| `/sys/kernel/debug/pinctrl/<dev>/pins` | كل pin: رقمه، اسمه، function الحالية |
| `/sys/kernel/debug/pinctrl/<dev>/pingroups` | الـ groups المسجّلة مع pins كل group |
| `/sys/kernel/debug/pinctrl/<dev>/pinmux-functions` | الـ functions المتاحة + groups بتاعتها |
| `/sys/kernel/debug/pinctrl/<dev>/pinmux-pins` | مين hold كل pin (owner) |
| `/sys/kernel/debug/pinctrl/<dev>/gpio-ranges` | mapping بين GPIO numbers و pin numbers |

الدالة `nmk_gpio_dbg_show_one()` (معرّفة تحت `CONFIG_DEBUG_FS`) بتُظهر لكل offset في الـ chip:
- الـ alternate function الحالية (GPIO/Alt-A/Alt-B/Alt-C)
- اتجاه الـ pin (input/output)
- قيمة الـ DAT register
- حالة الـ pull (PDIS register)
- حالة الـ sleep mode (SLPM)
- الـ interrupt mask (RIMSC/FIMSC)

```bash
# قراءة حالة كل GPIO في النظام
cat /sys/kernel/debug/gpio

# قراءة pin states عبر pinctrl debugfs (استبدل اسم الـ controller الفعلي)
cat /sys/kernel/debug/pinctrl/pinctrl-db8500/pins

# مين يمسك كل pin حالياً
cat /sys/kernel/debug/pinctrl/pinctrl-db8500/pinmux-pins

# اعرف الـ groups كلها
cat /sys/kernel/debug/pinctrl/pinctrl-db8500/pingroups
```

مثال output لـ `pins`:

```
pin 0 (GPIO0_A): gpio-nomadik.0 (GPIO UNCLAIMED)
pin 1 (GPIO1_A): sdi0_dat0grp function sdi0 group sdi0_dat0grp
pin 4 (GPIO4_A): uart0grp function uart0 group uart0grp
```

---

#### 2. مدخلات الـ sysfs

| المسار | الاستخدام |
|--------|----------|
| `/sys/class/gpio/export` | تصدير رقم GPIO لـ userspace |
| `/sys/class/gpio/unexport` | إلغاء التصدير |
| `/sys/class/gpio/gpioN/direction` | قراءة/كتابة `in` أو `out` |
| `/sys/class/gpio/gpioN/value` | قراءة/كتابة القيمة |
| `/sys/class/gpio/gpioN/edge` | ضبط الـ edge trigger: `none`, `rising`, `falling`, `both` |
| `/sys/class/gpio/gpioN/active_low` | عكس المنطق |
| `/sys/bus/platform/drivers/gpio-nomadik` | الـ driver نفسه |
| `/sys/bus/platform/devices/<dev>/power/` | power management state |

```bash
# تصدير GPIO رقم 16
echo 16 > /sys/class/gpio/export

# اقرأ القيمة
cat /sys/class/gpio/gpio16/value

# اضبط كـ output وشغّله
echo out > /sys/class/gpio/gpio16/direction
echo 1   > /sys/class/gpio/gpio16/value

# ارجع للـ alternate function عبر pinctrl (لازم تعمل unexport الأول)
echo 16 > /sys/class/gpio/unexport
```

---

#### 3. الـ ftrace: tracepoints/events

الـ subsystem مالوش tracepoints مخصصة كتير، بس الـ GPIO core و IRQ core بيوفّروا أحداث مفيدة:

```bash
# فعّل GPIO tracepoints
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable

# فعّل IRQ tracepoints (مهم لو بتصحح interrupts على النوماديك)
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_entry/enable
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_exit/enable

# تتبع دوال nmk_gpio مباشرة
echo nmk_gpio_irq_handler > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل العملية اللي بتحاول تصحّحها، ثم:
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

تتبع stack كل مرة بيُستدعى فيها الـ handler:

```bash
echo 1 > /sys/kernel/debug/tracing/options/func_stack_trace
```

فلترة شاملة على كل دوال الـ driver:

```bash
echo 'nmk_gpio_* __nmk_gpio_*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
```

---

#### 4. الـ printk و dynamic debug

تفعيل الـ dynamic debug لـ driver gpio-nomadik وpinctrl-nomadik:

```bash
# لو الـ kernel مبني بـ CONFIG_DYNAMIC_DEBUG=y
echo "module gpio_nomadik +p"    > /sys/kernel/debug/dynamic_debug/control
echo "module pinctrl_nomadik +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل مع file + line number + function name في الرسالة
echo "module gpio_nomadik +pflmt" > /sys/kernel/debug/dynamic_debug/control
```

رفع مستوى الـ loglevel مؤقتاً:

```bash
# اظهر كل رسائل الـ debug على console
echo 8 > /proc/sys/kernel/printk

# أو عبر dmesg
dmesg -n 8
dmesg -w | grep -iE "nmk|nomadik|gpio"
```

لو حبيت تضيف `pr_debug()` يدوياً في الكود أثناء التطوير:

```c
/* في gpio-nomadik.c، داخل nmk_gpio_irq_handler مثلاً */
pr_debug("nmk bank%u: IS=0x%08x rimsc=0x%08x fimsc=0x%08x\n",
         nmk_chip->bank,
         readl(nmk_chip->addr + NMK_GPIO_IS),
         nmk_chip->rimsc,
         nmk_chip->fimsc);
```

---

#### 5. Kernel config options للـ debugging

| الـ Config | الوظيفة |
|-----------|---------|
| `CONFIG_DEBUG_FS` | يفعّل debugfs (شرط أساسي) |
| `CONFIG_GPIOLIB_IRQCHIP` | دعم IRQ داخل الـ GPIO chip |
| `CONFIG_GPIO_SYSFS` | يفعّل `/sys/class/gpio` |
| `CONFIG_DEBUG_GPIO` | يضيف extra checks وـ warnings في الـ gpiolib |
| `CONFIG_PINCTRL_NOMADIK` | الـ pinctrl driver للـ Nomadik |
| `CONFIG_GPIO_NOMADIK` | الـ GPIO driver نفسه |
| `CONFIG_DYNAMIC_DEBUG` | يتيح تفعيل/تعطيل الـ pr_debug بدون recompile |
| `CONFIG_LOCKDEP` | يكشف lock ordering bugs (الـ chip بيستخدم spinlock) |
| `CONFIG_PROVE_LOCKING` | مع LOCKDEP يكشف deadlocks بسبب spinlock في ISR |
| `CONFIG_IRQ_DOMAIN_DEBUG` | يُظهر IRQ domain mapping في debugfs |
| `CONFIG_OF` | دعم Device Tree (ضروري للـ fwnode_handle) |
| `CONFIG_PINCTRL_DB8500` | SoC-specific data للـ DB8500 |
| `CONFIG_PINCTRL_DB8540` | SoC-specific data للـ DB8540 |
| `CONFIG_PINCTRL_STN8815` | SoC-specific data للـ STN8815 |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "GPIO_NOMADIK|PINCTRL_NOMADIK|DEBUG_GPIO|DEBUG_FS"
```

---

#### 6. أدوات خاصة بالـ subsystem

الـ Nomadik مالوش `devlink` support مخصص (الـ devlink للـ network devices)، لكن الأدوات دي مفيدة:

```bash
# gpiodetect وgpioinfo من libgpiod
gpiodetect             # اعرف كل GPIO chips في النظام
gpioinfo gpiochip0     # تفاصيل chip معينة
gpiomon gpiochip0 4    # مراقبة GPIO رقم 4 live

# gpiofind للبحث باسم الـ GPIO
gpiofind "SOME-GPIO-NAME"
```

```bash
# طريقة سريعة لمعرفة كل GPIO chips
ls /sys/class/gpio/
# مثال output:
# export  gpiochip0  gpiochip32  gpiochip64  unexport
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ kernel log | المعنى | الحل |
|----------------------|--------|-------|
| `gpio-nomadik: unable to get clock` | فشل `clk_get()` | تحقق من clock tree في DT، اسم الـ clock صح؟ |
| `gpio-nomadik: failed to ioremap` | فشل mapping الـ registers | تحقق من `reg` في DT، المساحة متعارضة مع driver تاني؟ |
| `gpio-nomadik: failed to request irq` | الـ IRQ مش متاح | هل `interrupts` property صح في DT؟ هل interrupt controller موجود؟ |
| `nmk_gpio_populate_chip: can't find gpio chip` | `nmk_gpio_populate_chip()` مش لاقي الـ chip | الـ fwnode مش بيشاور على chip مسجّلة، راجع ترتيب الـ probe |
| `pinctrl: pin X already requested` | pin متحجوز من driver تاني | خطأ في DT، pin مشترك بين devices |
| `pinctrl: request already taken by X` | نفس المشكلة من اتجاه الـ mux | audit الـ pinctrl nodes في DT |
| `WARNING: at .../gpio-nomadik.c:XXX` | `WARN_ON()` اتنفّذ | شوف رقم السطر في الـ log، الأغلب state inconsistency |
| `BUG: spinlock lockup` | الـ spinlock في `nmk_gpio_chip.lock` تعمّله starvation | تحقق من ISR، هل في كود بياخد الـ lock لفترة طويلة؟ |
| `of_irq_get: no irq for node gpio` | الـ DT node مش محتوي على `interrupts` | أضف `interrupts` و`interrupt-parent` في DT |
| `genirq: Flags mismatch irq` | اتنين drivers طالبين نفس الـ IRQ بـ flags مختلفة | راجع `IRQF_*` flags، هل الـ IRQ shared? |
| `irq X: nobody cared` | interrupt بيتولّد ومفيش handler بيعالجه | الـ RIMSC/FIMSC مضبوطين لكن مفيش driver سجّل الـ IRQ |

---

#### 8. أماكن استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في nmk_gpio_populate_chip() لو الـ chip مش اتلقى */
if (!nmk_chip) {
    WARN(1, "nmk: no chip found for fwnode\n");
    dump_stack();
    return ERR_PTR(-ENODEV);
}

/* في الـ IRQ handler لو IS register بيرجع 0 بدون سبب (spurious IRQ) */
u32 status = readl(nmk_chip->addr + NMK_GPIO_IS);
if (!status) {
    WARN_ONCE(1, "nmk bank%u: spurious IRQ, IS=0\n", nmk_chip->bank);
    return IRQ_NONE;
}

/* لو edge_rising و edge_falling كلاهم 0 لكن interrupt جه */
WARN_ON_ONCE(!(nmk_chip->edge_rising | nmk_chip->edge_falling));

/* في __nmk_gpio_set_slpm() لو الـ sleepmode مش متاح على الـ chip */
if (!nmk_chip->sleepmode) {
    WARN_ON(mode == NMK_GPIO_SLPM_INPUT);
    return;
}

/* تحقق من صحة رقم الـ bank */
WARN_ON(nmk_chip->bank >= NMK_MAX_BANKS);
```

---

### Hardware Level

---

#### 1. التحقق أن حالة الـ Hardware مطابقة لحالة الـ kernel

الـ `nmk_gpio_chip` بيخزّن shadow copies للـ registers في:
- `nmk_chip->rimsc` — rising-edge interrupt mask (software shadow)
- `nmk_chip->fimsc` — falling-edge interrupt mask (software shadow)
- `nmk_chip->rwimsc` — rising-edge wakeup mask (software shadow)
- `nmk_chip->fwimsc` — falling-edge wakeup mask (software shadow)
- `nmk_chip->pull_up` — pull-up bitmap
- `nmk_chip->lowemi` — low-EMI mode bitmap

للتحقق من التطابق بين الـ shadow والـ hardware الفعلي:

```bash
# اقرأ القيمة من debugfs (عبر nmk_gpio_dbg_show_one)
cat /sys/kernel/debug/gpio

# أو قارن يدوياً عبر devmem2 (شوف قسم register dump)
# مثلاً: kernel يقول rimsc=0x00000020
# devmem2 0x8012E040 w  يجب أن يرجع 0x00000020
```

---

#### 2. Register Dump عبر devmem2 / io

الـ Nomadik GPIO banks مبنية على base addresses مختلفة حسب الـ SoC. مثال لـ DB8500:

| Bank | Base Address | الـ Pins |
|------|-------------|---------|
| Bank 0 | `0x8012E000` | GPIO 0–31 |
| Bank 1 | `0x8012E080` | GPIO 32–63 |
| Bank 2 | `0x8000E000` | GPIO 64–95 |
| Bank 3 | `0x8000E080` | GPIO 96–127 |

```bash
# ثبّت devmem2 أو استخدم busybox devmem
# (استبدل BASE بالـ base address الفعلي من DT)
BASE=0x8012E000

# DAT: القيمة الحالية لكل pin
devmem2 $((BASE + 0x00)) w    # NMK_GPIO_DAT

# DIR: input=0, output=1
devmem2 $((BASE + 0x10)) w    # NMK_GPIO_DIR

# PDIS: pull disable — بيت=1 يعني pull disabled
devmem2 $((BASE + 0x0c)) w    # NMK_GPIO_PDIS

# AFSLA + AFSLB: alternate function selection
devmem2 $((BASE + 0x20)) w    # NMK_GPIO_AFSLA
devmem2 $((BASE + 0x24)) w    # NMK_GPIO_AFSLB

# Interrupt masks
devmem2 $((BASE + 0x40)) w    # NMK_GPIO_RIMSC (rising)
devmem2 $((BASE + 0x44)) w    # NMK_GPIO_FIMSC (falling)
devmem2 $((BASE + 0x48)) w    # NMK_GPIO_IS    (interrupt status)
devmem2 $((BASE + 0x50)) w    # NMK_GPIO_RWIMSC (rising wakeup)
devmem2 $((BASE + 0x54)) w    # NMK_GPIO_FWIMSC (falling wakeup)
devmem2 $((BASE + 0x58)) w    # NMK_GPIO_WKS   (wakeup status)
```

**تفسير AFSLA/AFSLB:**

```
Bit N في AFSLA=A, Bit N في AFSLB=B
A=0, B=0  →  GPIO mode
A=1, B=0  →  NMK_GPIO_ALT_A
A=0, B=1  →  NMK_GPIO_ALT_B
A=1, B=1  →  NMK_GPIO_ALT_C (أو C1/C2/C3/C4 حسب PRCM_GPIOCR)
```

Script لعمل dump كامل لـ bank واحد:

```bash
#!/bin/sh
BASE=${1:-0x8012E000}
echo "=== NMK GPIO Bank dump @ $BASE ==="
for reg in 00 04 08 0c 10 14 18 1c 20 24 28 40 44 48 4c 50 54 58 5c 60; do
    name=$(case $reg in
      00) echo "DAT      ";; 04) echo "DATS     ";; 08) echo "DATC     ";;
      0c) echo "PDIS     ";; 10) echo "DIR      ";; 14) echo "DIRS     ";;
      18) echo "DIRC     ";; 1c) echo "SLPC     ";; 20) echo "AFSLA    ";;
      24) echo "AFSLB    ";; 28) echo "LOWEMI   ";; 40) echo "RIMSC    ";;
      44) echo "FIMSC    ";; 48) echo "IS       ";; 4c) echo "IC       ";;
      50) echo "RWIMSC   ";; 54) echo "FWIMSC   ";; 58) echo "WKS      ";;
      5c) echo "EDGELEVEL";; 60) echo "LEVEL    ";;
    esac)
    val=$(devmem2 $((BASE + 0x$reg)) w 2>/dev/null | grep "Read" | awk '{print $NF}')
    printf "0x%s  %-12s = %s\n" "$reg" "$name" "$val"
done
```

---

#### 3. Logic Analyzer / Oscilloscope

**نقاط القياس:**

| القياس | الإعداد |
|--------|---------|
| GPIO line نفسه | Trigger على الـ edge المتوقع، قارن مع kernel value |
| IRQ latency | Toggle GPIO في بداية ISR، قِس الفرق بالـ scope |
| IOFORCE line | مراقبة sleep mode transitions |
| Clock (PCLK) | تحقق أن الـ clock شغال أثناء register access |

**تلميحات:**

- الـ DB8500 بيشتغل على 1.8V غالباً، اضبط الـ threshold صح
- لو الـ kernel بيقول pin=high بس القياس بيظهر low: ابحث عن مشكلة في الـ pull أو ALT mode مفعّل
- لو interrupt مش بيتولّد: تحقق أن RIMSC/FIMSC فيهم الـ bit المناسب عبر `devmem2`
- لو فيه glitches: فعّل `NMK_GPIO_LOWEMI` register بالـ bit الخاص بالـ pin المشكوك فيه

```c
/* أضف toggle في الـ ISR لقياس latency بالـ scope */
static irqreturn_t nmk_gpio_irq_handler(int irq, void *dev_id)
{
    /* debug: toggle a known-free GPIO to measure entry latency */
    writel(BIT(DEBUG_PIN), nmk_chip->addr + NMK_GPIO_DATS); /* set */
    /* ... handler body ... */
    writel(BIT(DEBUG_PIN), nmk_chip->addr + NMK_GPIO_DATC); /* clear */
    return IRQ_HANDLED;
}
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في الـ kernel log

| المشكلة | النمط في الـ dmesg | السبب |
|--------|-------------------|-------|
| الـ GPIO مش بيتغيّر | لا يوجد لوج، القيمة ثابتة في debugfs | الـ pin في ALT mode مش GPIO، راجع AFSLA/AFSLB |
| Spurious interrupts | `irq X: nobody cared` أو handler بيُستدعى باستمرار | الـ edge غير صحيح أو الـ IC register مش بيتكلير |
| الـ wakeup مش شغال | النظام مش بيصحى من suspend | RWIMSC/FWIMSC مش مضبوطين، أو الـ PRCM GPIOCR الـ ALT_C wakeup path مش enabled |
| الـ pull مش شغال | الـ signal بيعوم (floating) | PDIS=1 يعني disabled، لازم يكون PDIS=0 عشان الـ pull يشتغل |
| الـ clock مش running | `clk_enable failed` في dmesg | الـ PRCM مش enabling الـ GPIO clock، راجع power domain في DT |
| Interference بين banks | bank A بيأثر على bank B | race condition في `nmk_gpio_slpm_lock`، تحقق من الـ spinlock |
| ALT_C function مش شغال | الـ peripheral مش بيتواصل رغم الـ pinmux صح | PRCM_GPIOCR register مش متضبط، تحقق بـ devmem2 |

---

#### 5. Device Tree Debugging

```bash
# اعرض الـ DT المُحمَّل (compiled) لـ GPIO node
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "gpio@8012e000"

# تحقق من الـ compatible string
cat /proc/device-tree/soc/gpio@8012e000/compatible 2>/dev/null | tr '\0' '\n'

# تحقق من الـ reg property (base address + size)
xxd /proc/device-tree/soc/gpio@8012e000/reg 2>/dev/null

# تحقق من الـ interrupts property
xxd /proc/device-tree/soc/gpio@8012e000/interrupts 2>/dev/null

# شوف كل GPIO و pinctrl nodes في الـ DT
find /proc/device-tree -name "*gpio*" -o -name "*pinctrl*" 2>/dev/null
```

**مثال صحيح لـ Nomadik GPIO bank في DT:**

```dts
gpio0: gpio@8012e000 {
    compatible = "st,nomadik-gpio";
    reg = <0x8012E000 0x80>;         /* base + size يطابق الـ datasheet */
    interrupts = <0 119 0x4>;        /* SPI IRQ number + edge */
    #gpio-cells = <2>;
    gpio-controller;
    interrupt-controller;
    #interrupt-cells = <2>;
    clocks = <&prcc_pclk 1 9>;      /* clock reference لازم يكون صح */
    st,supports-sleepmode;           /* لو الـ SoC بيدعمه */
    gpio-bank = <0>;                 /* رقم الـ bank */
};
```

**فحص PRCM GPIOCR لـ alternate-C:**

الـ `prcm_gpiocr_registers` array في `nmk_pinctrl_soc_data` بتحدد عناوين GPIOCR registers في الـ PRCM. لو pin محتاج ALT_C:

```bash
# تحقق أن PRCM GPIOCR registers مقروءة صح (عناوين من الـ DB8500 TRM)
devmem2 0x80157138 w   # PRCM_GPIOCR1
devmem2 0x8015713C w   # PRCM_GPIOCR2
devmem2 0x80157140 w   # PRCM_GPIOCR3
```

---

### Practical Commands

---

#### كل الأوامر جاهزة للـ copy-paste

```bash
## ===== 1. حالة عامة =====
cat /sys/kernel/debug/gpio
cat /sys/kernel/debug/pinctrl/pinctrl-db8500/pins
cat /sys/kernel/debug/pinctrl/pinctrl-db8500/pinmux-pins
cat /sys/kernel/debug/pinctrl/pinctrl-db8500/pingroups

## ===== 2. GPIO sysfs =====
echo 16 > /sys/class/gpio/export
cat /sys/class/gpio/gpio16/direction
cat /sys/class/gpio/gpio16/value
echo 16 > /sys/class/gpio/unexport

gpiodetect 2>/dev/null || ls /sys/class/gpio/

## ===== 3. Dynamic Debug =====
echo "module gpio_nomadik +pflmt"    > /sys/kernel/debug/dynamic_debug/control
echo "module pinctrl_nomadik +pflmt" > /sys/kernel/debug/dynamic_debug/control
dmesg -w | grep -iE "nmk|nomadik|gpio"

## ===== 4. ftrace على دوال الـ driver =====
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo > trace
echo 'nmk_gpio* __nmk_gpio*' > set_ftrace_filter
echo function > current_tracer
echo 1 > options/func_stack_trace
echo 1 > tracing_on
sleep 5
echo 0 > tracing_on
cat trace | head -100

## ===== 5. IRQ events =====
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_entry/enable
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_exit/enable
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable
cat /sys/kernel/debug/tracing/trace_pipe &
TRACE_PID=$!
# شغّل العملية اللي بتختبرها
kill $TRACE_PID

## ===== 6. Register dump (DB8500 bank0) =====
BASE=0x8012E000
echo "DAT  :" && devmem2 $((BASE+0x00)) w
echo "DIR  :" && devmem2 $((BASE+0x10)) w
echo "PDIS :" && devmem2 $((BASE+0x0c)) w
echo "AFSLA:" && devmem2 $((BASE+0x20)) w
echo "AFSLB:" && devmem2 $((BASE+0x24)) w
echo "RIMSC:" && devmem2 $((BASE+0x40)) w
echo "FIMSC:" && devmem2 $((BASE+0x44)) w
echo "IS   :" && devmem2 $((BASE+0x48)) w
echo "WKS  :" && devmem2 $((BASE+0x58)) w

## ===== 7. فحص DT =====
find /proc/device-tree -name "*gpio*" 2>/dev/null
find /proc/device-tree -name "*pinctrl*" 2>/dev/null

## ===== 8. فحص الـ IRQ mapping =====
cat /proc/interrupts | grep -i gpio
cat /sys/kernel/debug/irq_domain_mapping 2>/dev/null | head -40
```

---

#### تفسير output مهم

**مثال output لـ `/sys/kernel/debug/gpio`:**

```
GPIOs 0-31, platform/gpio-nomadik.0, gpio-nomadik:
 gpio-4  (                    ) in  hi
 gpio-16 (sdi0_dat0            ) out hi  [kernel]
 gpio-23 (uart0-tx             ) out hi  [kernel]
 gpio-24 (uart0-rx             ) in  hi
```

- **`[kernel]`**: الـ GPIO محجوز من kernel driver
- **`in hi` / `out lo`**: direction + value
- لو الـ GPIO مش ظاهر في اللوج: مش exported ولا requested، لكن ممكن يكون في alternate function

**مثال output لـ `pinmux-pins`:**

```
pin 16 (GPIO16_A): sdi0 (GPIO UNCLAIMED) function sdi0 group sdi0_dat0grp
pin 23 (GPIO23_A): uart0 (GPIO UNCLAIMED) function uart0 group uart0grp
```

- **`GPIO UNCLAIMED`** مع function: الـ pinctrl hold عليه لكن الـ GPIO subsystem مش requesting إياه (ده normal لـ alternate functions)
- لو الـ pin مش ظاهر خالص: مش registered في الـ pinctrl، راجع الـ DT

---

#### script تشخيص alternate function لـ pin محدد

```bash
#!/bin/bash
# check-nmk-altfunc.sh <PIN_NUMBER>
PIN=${1:-0}
BANK=$((PIN / 32))
BIT=$((PIN % 32))

# DB8500 bank base addresses
BASES=(0x8012E000 0x8012E080 0x8000E000 0x8000E080)
BASE=${BASES[$BANK]}

echo "Pin $PIN → Bank $BANK, Bit $BIT, Base=$(printf '0x%08X' $BASE)"

AFSLA_HEX=$(devmem2 $((BASE + 0x20)) w 2>/dev/null | awk '/Value/{print $NF}')
AFSLB_HEX=$(devmem2 $((BASE + 0x24)) w 2>/dev/null | awk '/Value/{print $NF}')

AFSLA=$(printf '%d' "$AFSLA_HEX" 2>/dev/null || echo 0)
AFSLB=$(printf '%d' "$AFSLB_HEX" 2>/dev/null || echo 0)

A=$(( (AFSLA >> BIT) & 1 ))
B=$(( (AFSLB >> BIT) & 1 ))
ALT=$((A | (B << 1)))

case $ALT in
    0) MODE="GPIO (NMK_GPIO_ALT_GPIO)" ;;
    1) MODE="NMK_GPIO_ALT_A" ;;
    2) MODE="NMK_GPIO_ALT_B" ;;
    3) MODE="NMK_GPIO_ALT_C — check PRCM_GPIOCR for C1/C2/C3/C4" ;;
esac

echo "AFSLA bit=$A  AFSLB bit=$B  → $MODE"

# قارن مع ما يعرفه الـ kernel
echo ""
echo "Kernel pinctrl view:"
cat /sys/kernel/debug/pinctrl/*/pins 2>/dev/null | grep "pin $PIN " | head -3
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول — UART مش شغال على DB8500 بعد suspend/resume

#### العنوان
**الـ UART بيصحى أبكم بعد كل resume على DB8500**

#### السياق
شغالين على industrial gateway بيستخدم ST-Ericsson DB8500 (نفس SoC اللي في Samsung Galaxy S II).
الـ gateway بيتكلم مع modem خارجي عن طريق UART4.
المنتج بيدخل suspend بعد فترة idle، وبعد wake-up الـ UART بيوقف استقبال بيانات تماماً.

#### المشكلة
بعد كل `echo mem > /sys/power/state` والـ wake-up اللي بعده:
```bash
# الـ UART شغال قبل suspend
cat /dev/ttyS3 &   # يستقبل بيانات عادي

# بعد resume — صمت تام، مفيش بيانات
```
الـ modem بيبعت، لكن الـ kernel مش بيستقبل interrupts من الـ GPIO اللي على الـ UART RX line.

#### التحليل
**الـ sleep mode** هو المشكلة. في `gpio-nomadik.h`:

```c
/* Sleep mode */
enum nmk_gpio_slpm {
    NMK_GPIO_SLPM_INPUT,
    NMK_GPIO_SLPM_WAKEUP_ENABLE = NMK_GPIO_SLPM_INPUT,   // نفس القيمة: 0
    NMK_GPIO_SLPM_NOCHANGE,
    NMK_GPIO_SLPM_WAKEUP_DISABLE = NMK_GPIO_SLPM_NOCHANGE, // نفس القيمة: 1
};
```

وفي `struct nmk_gpio_chip`:
```c
struct nmk_gpio_chip {
    ...
    bool sleepmode;    // هل الـ chip بيدعم sleep mode أصلاً؟
    u32 real_wake;     // الـ GPIOs اللي المفروض توصحي النظام
    u32 rwimsc;        // Rising  Wake-up Interrupt Mask — محفوظ قبل sleep
    u32 fwimsc;        // Falling Wake-up Interrupt Mask — محفوظ قبل sleep
    u32 rimsc;         // Rising  Interrupt Mask — بيتعمله restore بعد resume
    u32 fimsc;         // Falling Interrupt Mask — بيتعمله restore بعد resume
    ...
};
```

**الـ SLPC register** (offset `0x1C` = `NMK_GPIO_SLPC`) بيتحكم في سلوك كل pin وقت الـ sleep.
لما الـ pinctrl DT node مش بيحدد `st,slpm-input`، الـ pin بيتحول لـ `NMK_GPIO_SLPM_NOCHANGE` — يعني مش بيصحي النظام.

**المسار الكامل:**
1. قبل suspend: driver بيكتب `rwimsc`/`fwimsc` بناءً على `real_wake` mask.
2. لكن لو الـ GPIO مش في `real_wake`، الـ bit مش بيتعمله set في `rwimsc`.
3. بعد resume: `rimsc` بيتعمله restore بصحيح، لكن الـ interrupt اللي اتجنب وقت الـ sleep مش بيتعوض.

#### الحل
**خطوة 1 — تعديل الـ Device Tree:**
```dts
/* قبل التعديل — مفيش sleep config */
uart4_default: uart4_default {
    pins1 {
        pinmux = <PINMUX_GPIO68__FUNC_UART4_RX>;
    };
};

/* بعد التعديل — نفعّل wakeup على RX pin */
uart4_default: uart4_default {
    pins1 {
        pinmux = <PINMUX_GPIO68__FUNC_UART4_RX>;
        st,slpm-input;   /* NMK_GPIO_SLPM_INPUT = wakeup enable */
    };
};
```

**خطوة 2 — التحقق بعد التعديل:**
```bash
# تأكد إن الـ GPIO بيظهر في wake-up sources
cat /sys/kernel/debug/gpio | grep uart4_rx

# تفعيل wakeup من userspace
echo enabled > /sys/class/gpio/gpio68/power/wakeup

# اختبار suspend/resume
echo mem > /sys/power/state
# أرسل بيانات من الـ modem → المفروض النظام يصحى
```

**خطوة 3 — تأكيد الـ registers:**
```bash
# اقرأ RWIMSC register (offset 0x50) على bank GPIO2 (GPIO64-95)
devmem2 0x8011E050 w   # DB8500 GPIO bank 2 base + RWIMSC
# Bit 4 (GPIO68 - 64 = 4) المفروض = 1
```

#### الدرس المستفاد
**الـ `nmk_gpio_slpm` enum** فيه aliasing مقصود: `WAKEUP_ENABLE == SLPM_INPUT`. اسم الـ enum مضلل — مش معناه "input direction"، معناه "فعّل الـ pin كـ wakeup source وقت الـ sleep". لازم تفرق بين **direction** وبين **sleep behavior** في الـ NMK GPIO.

---

### السيناريو الثاني — الـ SPI بيعمل glitch غريب على STN8815

#### العنوان
**الـ SPI clock بيعمل extra pulse بعد كل transaction على STN8815**

#### السياق
شغالين على custom board بيستخدم ST Nomadik STN8815 (أقدم SoC في العيلة).
الـ board بيتحكم في ADC خارجي عن طريق SPI0.
الـ oscilloscope بيكشف إن بعد كل SPI transaction، في pulse زيادة على SCK line.

#### المشكلة
```
SCK: __|‾|_|‾|_|‾|_|‾|__|‾|__  ← الـ pulse الزيادة دي
                              ^
                    Extra glitch هنا!
```
الـ ADC بيفسر الـ pulse دي كـ bit إضافي، فالقراءات عشوائية.

#### التحليل
الـ `PINCTRL_NMK_STN8815 = 0` — ده package identifier موجود في الـ header:
```c
#define PINCTRL_NMK_STN8815  0
#define PINCTRL_NMK_DB8500   1
```

المشكلة في الـ **alternate function** configuration. في STN8815، الـ SPI pins بتشتغل على `NMK_GPIO_ALT_A`:
```c
#define NMK_GPIO_ALT_GPIO  0   /* pure GPIO */
#define NMK_GPIO_ALT_A     1   /* Alt A */
#define NMK_GPIO_ALT_B     2   /* Alt B */
#define NMK_GPIO_ALT_C     (NMK_GPIO_ALT_A | NMK_GPIO_ALT_B)  /* = 3 */
```

الـ **AFSLA** (offset `0x20`) وـ **AFSLB** (offset `0x24`) registers بيتحكموا في الـ alternate function:

| AFSLB bit | AFSLA bit | الـ Function |
|-----------|-----------|-------------|
| 0 | 0 | GPIO |
| 0 | 1 | Alt A |
| 1 | 0 | Alt B |
| 1 | 1 | Alt C |

المشكلة: لما الـ driver بيعمل SPI transaction وبعدين يرجع الـ pin لـ GPIO mode (بعض الـ drivers بيعمل كده للـ CS)، بيكون في لحظة transition فيها الـ AFSLA بتتمسح قبل ما AFSLB تتمسح — وده بيخلق state مؤقتة = `Alt B` بدل `GPIO` → pulse زيادة.

#### الحل
**إصلاح الـ pinctrl driver** علشان يعمل atomic write للـ registers:

```c
/* الطريقة الصح: امسح AFSLA وAFSLB مع بعض */
static void nmk_set_gpio_mode(struct nmk_gpio_chip *nmk_chip,
                               unsigned int offset)
{
    u32 afsla, afslb, bit = BIT(offset);

    /* اقرأ الحالة الحالية */
    afsla = readl(nmk_chip->addr + NMK_GPIO_AFSLA);  /* 0x20 */
    afslb = readl(nmk_chip->addr + NMK_GPIO_AFSLB);  /* 0x24 */

    /* امسح الاتنين مع بعض — لا تعمل write واحدة تاني */
    afsla &= ~bit;
    afslb &= ~bit;

    /* اكتب AFSLB أولاً — لأن 10b = Alt B (مؤذي أقل من 01b = Alt A مع SCK) */
    writel(afslb, nmk_chip->addr + NMK_GPIO_AFSLB);
    writel(afsla, nmk_chip->addr + NMK_GPIO_AFSLA);
}
```

**التأكد من الـ DT:**
```dts
spi0_default_state: spi0_default {
    spi0_dat_clk {
        pinmux = <PINMUX_GPIO4__FUNC_SPI0_CLK>,
                 <PINMUX_GPIO5__FUNC_SPI0_MOSI>,
                 <PINMUX_GPIO6__FUNC_SPI0_MISO>;
        /* altsetting = NMK_GPIO_ALT_A لـ STN8815 */
    };
};
```

#### الدرس المستفاد
الـ **AFSLA/AFSLB** register pair بيشتغلوا **معاً** لتحديد الـ alternate function. أي write غير atomic للاتنين بيخلق state وسطية مش مقصودة. على STN8815 المشكلة أوضح لأن الـ SoC أبطأ وفيه glitch filter أضعف مقارنة بـ DB8500.

---

### السيناريو الثالث — I2C بيتعلق على Mobileye SoC

#### العنوان
**الـ I2C bus بيتعلق بعد أول message على Mobileye EyeQ**

#### السياق
شغالين على automotive ECU بيستخدم Mobileye EyeQ SoC (اللي فيه `is_mobileye_soc` flag في الـ driver).
الـ ECU بيتكلم مع temperature sensor عن طريق I2C.
البورد بيشتغل تمام في الـ lab، لكن في السيارة الفعلية الـ I2C بيتعلق بعد أول قراءة.

#### المشكلة
```bash
i2cdetect -y 0
# يتعلق هنا — لا timeout، لا response
# dmesg يظهر:
# i2c i2c-0: timeout waiting for bus ready
```

#### التحليل
الـ `struct nmk_gpio_chip` فيها:
```c
struct nmk_gpio_chip {
    ...
    bool is_mobileye_soc;  /* flag خاص بـ Mobileye EyeQ */
    void (*set_ioforce)(bool enable);  /* callback لـ ioforce */
    ...
};
```

**الـ `set_ioforce` callback** هو المفتاح. في NMK GPIO، الـ **IOFORCE** mechanism بيخلي الـ GPIO controller يسيطر على الـ pad حتى لو الـ pin معمله alternate function. ده مهم لـ power management وللـ sleep.

على الـ Mobileye EyeQ، الـ SoC variant بيحتاج تسلسل مختلف لتفعيل/تعطيل IOFORCE عند تغيير الـ pin من I2C إلى GPIO والعكس. المشكلة:

1. الـ I2C driver بيطلب GPIO fallback للـ bus recovery.
2. الـ GPIO driver بيعمل `set_ioforce(true)` على الـ Mobileye path.
3. لكن `is_mobileye_soc` بيغير timing الـ IOFORCE — والـ I2C controller مش بيلاقي الـ SDA line حرة.
4. الـ I2C controller يدخل في wait loop لـ always → hang.

**الـ registers المتعلقة:**
- `NMK_GPIO_DAT` (`0x00`) — الـ data register، مش بيتأثر بـ IOFORCE بشكل صح على Mobileye
- `NMK_GPIO_DIR` (`0x10`) / `NMK_GPIO_DIRS` (`0x14`) / `NMK_GPIO_DIRC` (`0x18`) — Set/Clear للـ direction

#### الحل
**خطوة 1 — تشخيص من الـ debugfs:**
```bash
# شوف حالة الـ GPIO pins اللي على I2C
cat /sys/kernel/debug/gpio | grep i2c
# المفروض يظهر: in  lo  (لأن SDA/SCL مش مرفوعين؟)

# اقرأ DAT register مباشرة
devmem2 0xA03E0000 w   # I2C GPIO bank base + NMK_GPIO_DAT (0x00)
```

**خطوة 2 — إضافة workaround في الـ DT للـ Mobileye:**
```dts
i2c0_pins: i2c0-pins {
    pins {
        pinmux = <PINMUX_GPIO12__FUNC_I2C0_SDA>,
                 <PINMUX_GPIO13__FUNC_I2C0_SCL>;
        /* على Mobileye: لازم pull-up صريح */
        st,pull-up;       /* NMK_GPIO_PULL_UP */
        /* وتعطيل sleep mode على I2C pins */
        st,slpm-nochange; /* NMK_GPIO_SLPM_NOCHANGE */
    };
};
```

**خطوة 3 — تعديل الـ driver للـ Mobileye path:**
```c
/* في gpio-nomadik.c — مكان set_ioforce */
if (nmk_chip->is_mobileye_soc && nmk_chip->set_ioforce) {
    /* أضف delay صغير قبل IOFORCE على Mobileye */
    udelay(1);
    nmk_chip->set_ioforce(enable);
    udelay(1);  /* اديه وقت يستقر */
}
```

#### الدرس المستفاد
الـ `is_mobileye_soc` flag في الـ struct مش cosmetic — ده indicator إن الـ SoC variant بيحتاج **timing مختلف** في الـ IOFORCE sequence. أي path في الـ driver بيعمل `set_ioforce` لازم يراعي الـ flag ده. automotive SoCs بيكون فيها timing constraints أصعم من consumer electronics.

---

### السيناريو الرابع — GPIO interrupts مش شغالة على DB8540

#### العنوان
**الـ edge-triggered interrupts واقفة بعد الأول على DB8540**

#### السياق
شغالين على Android TV box بيستخدم DB8540 (اللي فيه `NMK_GPIO_EDGELEVEL` و`NMK_GPIO_LEVEL` registers — الجديدين في DB8540).
الـ box بيستخدم GPIO interrupt لاستقبال IR remote control signals.
الأول يشتغل، بعدين الـ interrupts تقف.

#### المشكلة
```bash
# الزر الأول بيشتغل
# بعد كده، مفيش interrupts
cat /proc/interrupts | grep gpio-ir
# العداد واقف
```

#### التحليل
الـ DB8540 أضاف registers جدد مش موجودين في STN8815/DB8500:

```c
/* These appear in DB8540 and later ASICs */
#define NMK_GPIO_EDGELEVEL 0x5C  /* NEW in DB8540 */
#define NMK_GPIO_LEVEL     0x60  /* NEW in DB8540 */
```

وفي `struct nmk_gpio_chip`:
```c
u32 edge_rising;   /* tracking الـ edges اللي configured */
u32 edge_falling;
```

الـ interrupt registers الكاملة:
```c
#define NMK_GPIO_RIMSC   0x40  /* Rising  Interrupt Mask Set/Clear */
#define NMK_GPIO_FIMSC   0x44  /* Falling Interrupt Mask Set/Clear */
#define NMK_GPIO_IS      0x48  /* Interrupt Status */
#define NMK_GPIO_IC      0x4c  /* Interrupt Clear */
#define NMK_GPIO_RWIMSC  0x50  /* Rising  Wake-up Interrupt Mask */
#define NMK_GPIO_FWIMSC  0x54  /* Falling Wake-up Interrupt Mask */
#define NMK_GPIO_WKS     0x58  /* Wake-up Status */
#define NMK_GPIO_EDGELEVEL 0x5C /* Edge/Level select — DB8540 */
#define NMK_GPIO_LEVEL   0x60  /* Level value — DB8540 */
```

**المشكلة الجوهرية:** على DB8540، لما الـ interrupt handler بيستخدم threaded IRQ، الـ `NMK_GPIO_IC` (Interrupt Clear) لازم يتعمل **بعد** ما الـ edge يتـ acknowledge. لكن لو الـ EDGELEVEL register مش معمول configure صح، الـ hardware ممكن يـ latch نفس الـ edge تاني وميعملش clear.

الـ `edge_rising` / `edge_falling` في الـ struct بيتعملوا track لكن لو الـ DB8540-specific EDGELEVEL register مش بيتكتب فيه الـ correct mode، الـ first edge بيخلص وبعدين الـ hardware بيدخل في level-sensitive mode بالـ wrong polarity.

#### الحل
**خطوة 1 — تشخيص:**
```bash
# اقرأ EDGELEVEL register على الـ GPIO bank
BANK_BASE=0x8011A000  # DB8540 GPIO bank base
devmem2 $((BANK_BASE + 0x5C)) w   # EDGELEVEL
devmem2 $((BANK_BASE + 0x48)) w   # IS — interrupt status
devmem2 $((BANK_BASE + 0x40)) w   # RIMSC — rising mask

# لو IS فيه الـ bit set ومش بيتـ clear → مشكلة clear sequence
```

**خطوة 2 — تعديل الـ DT للـ IR receiver:**
```dts
ir_receiver: ir-receiver {
    compatible = "gpio-ir-receiver";
    gpios = <&gpio2 15 GPIO_ACTIVE_LOW>;

    /* على DB8540: صريح edge-triggered */
    interrupt-parent = <&gpio2>;
    interrupts = <15 IRQ_TYPE_EDGE_BOTH>;
};
```

**خطوة 3 — تأكد إن الـ IC register بيتكتب صح:**
```c
/* في IRQ handler — لازم clear قبل process */
writel(BIT(offset), nmk_chip->addr + NMK_GPIO_IC);  /* 0x4C */
/* ثم اقرأ IS للتأكد */
val = readl(nmk_chip->addr + NMK_GPIO_IS);
if (val & BIT(offset))
    /* لسه pending — DB8540 edge latch issue */
    writel(BIT(offset), nmk_chip->addr + NMK_GPIO_IC);
```

#### الدرس المستفاد
الـ `NMK_GPIO_EDGELEVEL` و`NMK_GPIO_LEVEL` registers (الجديدين في DB8540) مش مجرد feature إضافية — دول بيغيروا الـ interrupt clear sequence بالكامل. أي code بيشتغل على DB8540 ولا بيستخدمهم صح هيلاقي spurious interrupt loss. لازم تعرف إن `PINCTRL_NMK_DB8500 = 1` مش بيغطي DB8540.

---

### السيناريو الخامس — الـ Low EMI mode بيخرب USB على STN8815

#### العنوان
**الـ USB بيعمل packet loss عشوائي على custom STN8815 board**

#### السياق
شغالين على IoT sensor hub بيستخدم STN8815.
الـ hub بيجمع بيانات من sensors وبيبعتها لـ host عن طريق USB Full Speed.
في بيئة noisy (مصنع بفيه motors)، الـ USB بيعمل packet loss وretransmissions كتير.
الـ EMC team طلبوا تفعيل الـ Low EMI mode على الـ GPIO pins.

#### المشكلة
بعد تفعيل Low EMI على كل الـ GPIO pins:
```bash
# قبل: USB يشتغل تمام
lsusb -v  # يظهر device عادي

# بعد تفعيل LOWEMI على كل البنوك
# USB بيظهر ويختفي
dmesg | grep usb
# usb 1-1: USB disconnect, device number 2
# usb 1-1: new full-speed USB device number 3 using nomadik-usb
```

#### التحليل
الـ `NMK_GPIO_LOWEMI` register موجود في الـ header:
```c
#define NMK_GPIO_LOWEMI  0x28  /* Low EMI (Electromagnetic Interference) control */
```

وفي الـ struct:
```c
struct nmk_gpio_chip {
    ...
    u32 lowemi;  /* حالة الـ LOWEMI register — محفوظة للـ suspend/resume */
    ...
};
```

**ما الـ LOWEMI بيعمله:** بيقلل الـ slew rate وبيخفض الـ drive strength للـ GPIO pins. ده بيقلل الـ EMI لكن بيأخر الـ rising/falling edges.

**المشكلة:** USB D+ وD- محتاجين slew rate معين ومحدد بـ USB spec. لما الـ LOWEMI اتفعّل على الـ USB GPIO pins:
- الـ eye diagram بتضيق
- الـ signal بيوصل delayed للـ USB PHY
- الـ USB host بيلاقي signal integrity violations → disconnect/reconnect

الـ `lowemi` field في الـ struct بيتعمله save/restore في الـ suspend — يعني لو الـ LOWEMI اتفعّل غلط، هيفضل كده حتى بعد resume.

#### الحل
**خطوة 1 — تشخيص:**
```bash
# اقرأ LOWEMI register على الـ bank اللي فيه USB pins
BANK_BASE=0x101E4000  # STN8815 GPIO bank base
devmem2 $((BANK_BASE + 0x28)) w   # LOWEMI register

# لو قيمته مش 0 على USB pins → هذه المشكلة
```

**خطوة 2 — تعديل الـ DT علشان تستثني USB pins من LOWEMI:**
```dts
/* ضبط LOWEMI بشكل انتقائي */
gpio_bank0: gpio@101E4000 {
    /* ... */

    usb_pins: usb-pins {
        pins = "GPIO28", "GPIO29";   /* D+ و D- */
        /* NO low-emi property هنا */
    };

    other_pins: other-pins {
        pins = "GPIO0", "GPIO1", "GPIO2";
        st,lowemi;   /* LOWEMI فقط على الـ pins اللي مش USB */
    };
};
```

**خطوة 3 — تأكيد الـ fix:**
```bash
# بعد التعديل
devmem2 $((BANK_BASE + 0x28)) w
# USB pin bits المفروض = 0 (LOWEMI مش فعّال)

# اختبار USB
while true; do
    dd if=/dev/urandom bs=1M count=10 | dd of=/dev/usb_device bs=1M 2>&1 | grep error
    sleep 1
done
# المفروض zero errors
```

**خطوة 4 — حفظ الـ lowemi state صح:**
```c
/* في suspend handler */
nmk_chip->lowemi = readl(nmk_chip->addr + NMK_GPIO_LOWEMI);

/* في resume handler */
writel(nmk_chip->lowemi, nmk_chip->addr + NMK_GPIO_LOWEMI);
/* الـ struct field بيحفظ الـ correct value بعد تعديل الـ DT */
```

#### الدرس المستفاد
الـ `NMK_GPIO_LOWEMI` (offset `0x28`) هو **per-pin control** — كل bit بيتحكم في pin واحد في البنك. تفعيله على كل البنك بدون تفكير **بيكسر** أي protocol بيحتاج precise timing زي USB، SPI high-speed، أو HDMI DDC. لازم تعرف handy rule: **low-speed digital signals (buttons, LEDs, slow I2C) → LOWEMI OK** — **high-speed signals → LOWEMI خطر**.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي بتغطي الـ subsystems المتعلقة بـ `gpio-nomadik` مباشرةً.

#### الـ GPIO Subsystem

| المقال | الوصف |
|--------|-------|
| [GPIO in the kernel: an introduction](https://lwn.net/Articles/532714/) | مقدمة شاملة للـ GPIO API — Jonathan Corbet، يناير 2013 |
| [GPIO in the kernel: future directions](https://lwn.net/Articles/533632/) | الاتجاهات المستقبلية للـ GPIO API، الانتقال لنموذج الـ descriptor |
| [gpiolib: introduce descriptor-based GPIO interface](https://lwn.net/Articles/528226/) | أول RFC لنظام الـ `gpiod_*` الـ descriptor-based |
| [gpio: rework locking and object life-time control](https://lwn.net/Articles/960024/) | إعادة هيكلة الـ locking في الـ gpiolib |
| [gpio: improve support for shared GPIOs](https://lwn.net/Articles/1039548/) | دعم الـ shared GPIOs في أحدث إصدارات الـ kernel |

#### الـ Pinctrl Subsystem

| المقال | الوصف |
|--------|-------|
| [The pin control subsystem](https://lwn.net/Articles/468759/) | أفضل overview للـ pinctrl — ازاي بيتكامل مع GPIO |
| [drivers: create a pin control subsystem](https://lwn.net/Articles/463335/) | الـ patch series الأصلي اللي أنشأ الـ pinctrl framework |
| [drivers: create a pin control subsystem v8](https://lwn.net/Articles/460768/) | النسخة النهائية قبل القبول في الـ mainline |
| [pin controller subsystem v7](https://lwn.net/Articles/459190/) | تطور التصميم في مرحلة الـ review |
| [pinctrl: add a generic pin config interface](https://lwn.net/Articles/467269/) | إضافة الـ `pinconf` — pull up/down، drive strength |
| [pinctrl: add a pin config interface](https://lwn.net/Articles/471826/) | التفاصيل الكاملة للـ pin configuration API |
| [Documentation/pinctrl.txt](https://lwn.net/Articles/465077/) | الـ documentation الرسمي للـ subsystem على LWN |
| [pinctrl: introduce the concept of a GPIO pin function category](https://lwn.net/Articles/1031226/) | أحدث تطوير في العلاقة بين pinctrl وـ GPIO |

---

### التوثيق الرسمي في الـ Kernel

#### GPIO

```
Documentation/driver-api/gpio/
├── index.rst          # نقطة البداية
├── intro.rst          # المقدمة المختصرة
├── driver.rst         # ازاي تكتب GPIO controller driver
├── consumer.rst       # ازاي تستخدم GPIO من driver تاني
├── board.rst          # GPIO mappings في DT
└── legacy.rst         # الـ API القديم (integer-based) — deprecated
```

الـ online links:
- [General Purpose Input/Output (GPIO)](https://static.lwn.net/kerneldoc/driver-api/gpio/index.html)
- [GPIO Driver Interface](https://static.lwn.net/kerneldoc/driver-api/gpio/driver.html)
- [Legacy GPIO Interfaces](https://static.lwn.net/kerneldoc/driver-api/gpio/legacy.html)

#### Pinctrl

```
Documentation/driver-api/pin-control.rst
Documentation/devicetree/bindings/pinctrl/ste,nomadik.txt
```

---

### الملفات المصدرية المرتبطة مباشرةً

```
drivers/gpio/gpio-nomadik.c                    # التنفيذ الكامل للـ driver
include/linux/gpio/gpio-nomadik.h              # الـ header الموثق هنا
drivers/pinctrl/nomadik/pinctrl-nomadik.c      # الـ pinctrl side
drivers/pinctrl/nomadik/pinctrl-db8500.c       # DB8500 SoC data
drivers/pinctrl/nomadik/pinctrl-db8540.c       # DB8540 SoC data
drivers/pinctrl/nomadik/pinctrl-stn8815.c      # STN8815 SoC data
```

الـ cross-reference على Bootlin Elixir:
- [gpio-nomadik.h](https://elixir.bootlin.com/linux/latest/source/include/linux/gpio/gpio-nomadik.h)
- [gpio-nomadik.c](https://elixir.bootlin.com/linux/latest/source/drivers/gpio/gpio-nomadik.c)

---

### Kernel Commits المهمة

#### نقطة البداية — 2009

```
commit 2ec1d3594563e0283873e24bb5d100dffee5d568
Date:   2009-07-02
[ARM] 5584/1: nomadik: add gpio driver and devices
```

أول commit أضاف الـ GPIO driver للـ Nomadik platform إلى الـ mainline kernel.

#### بداية الـ pinctrl integration — 2012

```
commit e98ea774c8d210364379329f042e7596f83ecc58
Date:   2012-04-26
pinctrl/nomadik: basic Nomadik pinctrl interface
```

أول خطوة في دمج الـ driver مع الـ pinctrl subsystem الجديد.

#### دعم الـ alternate-C functions — 2012

```
commit c22df08c7ffbfb281b0e5dff3fff4e192d1a7863
Date:   2012-09-27
pinctrl/nomadik: support other alternate-C functions
```

أضاف الـ `PRCM_GPIOCR_ALTCX` macro والـ structs المتعلقة بيها اللي موجودة في الـ header.

#### فصل الـ GPIO driver عن الـ pinctrl — 2024

```
commit 966942ae493650210b9514f3d4bfc95f78ef0129
Date:   2024-02-28
gpio: nomadik: extract GPIO platform driver from drivers/pinctrl/nomadik/
```

نقل الـ driver من `drivers/pinctrl/nomadik/` لـ `drivers/gpio/gpio-nomadik.c` — أهم restructure في تاريخ الـ driver.

#### دعم Mobileye EyeQ5 — 2024

```
commit 3c30cc26df0a3fc50b1f3fe4fd3a9b19a1704d95
Date:   2024-02-28
gpio: nomadik: support mobileye,eyeq5-gpio
```

أضاف الـ `is_mobileye_soc` flag في `struct nmk_gpio_chip` لدعم الـ Mobileye EyeQ5 SoC.

```bash
# لعرض الـ history الكامل
git log --oneline --follow -- drivers/gpio/gpio-nomadik.c
git log --oneline --follow -- include/linux/gpio/gpio-nomadik.h
```

---

### نقاشات الـ Mailing List

**الـ mailing list الرئيسي:** `linux-gpio@vger.kernel.org`
أرشيف كامل: [lore.kernel.org/linux-gpio](https://lore.kernel.org/linux-gpio/)

نقاشات موثقة:

- **[pinctrl: nomadik: Fix pull direction debug info](https://lore.kernel.org/all/20200806155322.GA25523@ola-jn9phv2.ad.garmin.com/)** — Andrew Halaney، أغسطس 2020
- **[PATCH 19/23 gpio: nomadik: grab optional reset control](https://lkml.org/lkml/2024/2/15/371)** — ضمن الـ patch series الكبير لدعم Mobileye EyeQ5
- **[gpio: nomadik: fix the debugfs helper stub](https://lkml.org/lkml/2025/9/15/519)** — Bartosz Golaszewski، 2025
- **[Add support for Mobileye EyeQ5 pin controller](https://lore.kernel.org/lkml/20231218-mbly-pinctrl-v1-0-2f7d366c2051@bootlin.com/T/)** — الـ pinctrl side من نفس الـ rework
- **[pinctrl/nomadik: implement pin configuration](https://linux-arm-kernel.infradead.narkive.com/3lM8gaRe/patch-06-12-pinctrl-nomadik-implement-pin-configuration)** — patch تاريخي مهم من 2012

---

### الكتب المُوصى بيها

#### Linux Device Drivers, 3rd Edition (LDD3)

| الفصل | الأهمية للـ Nomadik GPIO |
|-------|--------------------------|
| Chapter 9: Communicating with Hardware | الـ memory-mapped I/O وـ `ioread32`/`iowrite32` اللي بيستخدمها الـ driver |
| Chapter 10: Interrupt Handling | الـ `RIMSC`/`FIMSC` interrupt mask registers |
| Chapter 5: Concurrency and Race Conditions | الـ `spinlock_t lock` في `nmk_gpio_chip` |

متاح مجاناً: [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

> **ملاحظة:** LDD3 قديم (kernel 2.6). استخدمه للـ concepts فقط — الـ GPIO API اتغير بشكل كبير.

#### Linux Kernel Development, 3rd Edition — Robert Love

| الفصل | الأهمية |
|-------|---------|
| Chapter 7: Interrupts and Interrupt Handlers | فهم الـ `edge_rising`/`edge_falling` في `nmk_gpio_chip` |
| Chapter 10: Kernel Synchronization Methods | الـ `spinlock_t` patterns اللي بتشوفها في الـ driver |
| Chapter 14: The Block I/O Layer | الـ interrupt handling patterns العامة |

#### Embedded Linux Primer, 2nd Edition — Christopher Hallinan

| الفصل | الأهمية |
|-------|---------|
| Chapter 15: Debugging Embedded Linux Applications | `debugfs` وـ `seq_file` — `nmk_gpio_dbg_show_one()` بيستخدمهم |
| Chapter 16: Kernel Debugging Techniques | ازاي تـ debug GPIO driver على embedded hardware |

#### Professional Linux Kernel Architecture — Wolfgang Mauerer

الفصول الخاصة بالـ device drivers وإدارة الـ interrupts بتشرح الـ architecture اللي `nmk_gpio_chip` بني عليها.

---

### KernelNewbies.org

بيوثق التغييرات في كل إصدار kernel لكل subsystem:

| الصفحة | ما تجده فيها |
|--------|-------------|
| [Linux_6.9](https://kernelnewbies.org/Linux_6.9) | GPIO/pinctrl changes لـ 6.9 |
| [Linux_6.6](https://kernelnewbies.org/Linux_6.6) | GPIO/pinctrl changes لـ 6.6 |
| [Linux_6.5](https://kernelnewbies.org/Linux_6.5) | GPIO/pinctrl changes لـ 6.5 |
| [Linux_6.1](https://kernelnewbies.org/Linux_6.1) | GPIO/pinctrl changes لـ 6.1 |
| [Linux_5.16](https://kernelnewbies.org/Linux_5.16) | GPIO/pinctrl changes لـ 5.16 |
| [Linux_5.10](https://kernelnewbies.org/Linux_5.10) | GPIO/pinctrl changes لـ 5.10 |

كل صفحة فيها section مخصص لـ "Pin Controllers (pinctrl)" وـ "General Purpose I/O (gpio)" مع links للـ commits المتعلقة.

---

### eLinux.org

مش في صفحة مخصصة للـ Nomadik على eLinux، لكن في مصادر مفيدة:

- [Talk:GPIO](https://elinux.org/Talk:GPIO) — نقاشات حول GPIO API على embedded Linux
- [EBC Flashing an LED](https://elinux.org/EBC_Flashing_an_LED) — مثال عملي لاستخدام GPIO في embedded Linux

---

### Search Terms للبحث عن معلومات أكثر

للبحث في الـ mailing list archives على [lore.kernel.org](https://lore.kernel.org):

```
"nmk_gpio_chip"
"NMK_GPIO_RIMSC"
"pinctrl nomadik"
"gpio-nomadik"
"PINCTRL_NMK_DB8500"
"nmk_pinctrl_soc_data"
"Mobileye EyeQ5 gpio nomadik"
"ST-Ericsson nomadik gpio"
"Linus Walleij" nomadik
```

للبحث في الـ kernel source على [elixir.bootlin.com](https://elixir.bootlin.com):

```
nmk_gpio_chip
nmk_gpio_slpm
NMK_GPIO_ALT_C
prcm_gpiocr_altcx
nmk_pinctrl_db8500_init
nmk_gpio_populate_chip
```

للبحث العام:

```
site:lwn.net "pin control subsystem" OR "GPIO kernel"
site:kernelnewbies.org gpio pinctrl
"gpio-nomadik.c" linux kernel history
"PRCM GPIOCR" ST-Ericsson DB8500 alternate function
```
## Phase 8: Writing simple module

### الفكرة

**`__nmk_gpio_make_output`** هي دالة exported من `gpio-nomadik` بتغير اتجاه pin لـ output وبتكتب قيمة عليه مباشرةً في الـ MMIO registers. هنستخدم **kprobe** عشان نـ intercept أي استدعاء لها ونطبع اسم الـ chip والـ offset والقيمة المطلوبة — من غير ما نلمس الـ hardware خالص.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on __nmk_gpio_make_output
 * Traces every call that sets a Nomadik GPIO pin as output.
 */

#include <linux/module.h>       /* MODULE_* macros, module_init/exit        */
#include <linux/kernel.h>       /* pr_info                                  */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe, ...      */
#include <linux/gpio/gpio-nomadik.h> /* struct nmk_gpio_chip, enums        */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Nomadik GPIO Tracer");
MODULE_DESCRIPTION("kprobe on __nmk_gpio_make_output to trace GPIO direction changes");

/* -----------------------------------------------------------------
 * pre_handler — بيتشغل قبل ما الدالة الأصلية تنفذ ولو بمقدار instruction واحدة.
 * الـ regs بتعطينا الـ CPU registers اللي فيها الـ arguments (ABI-dependent).
 * ----------------------------------------------------------------- */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On ARM64 / x86-64 the first three arguments land in:
     *   ARM64 : x0=nmk_chip  x1=offset  x2=val
     *   x86-64: rdi=nmk_chip rsi=offset rdx=val
     * We cast regs->regs[0] (ARM64) or regs->di (x86-64) to the chip ptr.
     * Use a generic approach via the first arg register.
     */
#if defined(CONFIG_ARM64)
    struct nmk_gpio_chip *nmk_chip = (struct nmk_gpio_chip *)regs->regs[0];
    unsigned int offset             = (unsigned int)regs->regs[1];
    int          val                = (int)regs->regs[2];
#elif defined(CONFIG_X86_64)
    struct nmk_gpio_chip *nmk_chip = (struct nmk_gpio_chip *)regs->di;
    unsigned int offset             = (unsigned int)regs->si;
    int          val                = (int)regs->dx;
#else
    /* Fallback: just report that we hit the probe */
    struct nmk_gpio_chip *nmk_chip = NULL;
    unsigned int offset             = 0;
    int          val                = 0;
#endif

    if (nmk_chip) {
        pr_info("nmk_gpio_tracer: bank=%u offset=%u val=%d "
                "(gpio=%u) addr=%px\n",
                nmk_chip->bank,
                offset,
                val,
                (nmk_chip->bank * 32) + offset, /* absolute GPIO number */
                nmk_chip->addr);
    } else {
        pr_info("nmk_gpio_tracer: __nmk_gpio_make_output called "
                "(arch not decoded)\n");
    }

    return 0; /* 0 = continue execution normally */
}

/* -----------------------------------------------------------------
 * struct kprobe — بتربط الـ handler بالدالة المستهدفة عن طريق اسمها.
 * ----------------------------------------------------------------- */
static struct kprobe nmk_kp = {
    .symbol_name = "__nmk_gpio_make_output",
    .pre_handler = handler_pre,
};

/* -----------------------------------------------------------------
 * module_init — بيسجل الـ kprobe عند تحميل الـ module.
 * لو الدالة مش موجودة في الـ kernel (مش compiled)، register_kprobe بترجع -ENOENT.
 * ----------------------------------------------------------------- */
static int __init nmk_gpio_tracer_init(void)
{
    int ret;

    ret = register_kprobe(&nmk_kp);
    if (ret < 0) {
        pr_err("nmk_gpio_tracer: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("nmk_gpio_tracer: kprobe planted on %s (addr=%px)\n",
            nmk_kp.symbol_name, nmk_kp.addr);
    return 0;
}

/* -----------------------------------------------------------------
 * module_exit — بيشيل الـ kprobe عند إزالة الـ module.
 * لازم يتعمل دايمًا عشان منسيبش pointer يشاور على code اتشالت.
 * ----------------------------------------------------------------- */
static void __exit nmk_gpio_tracer_exit(void)
{
    unregister_kprobe(&nmk_kp);
    pr_info("nmk_gpio_tracer: kprobe removed\n");
}

module_init(nmk_gpio_tracer_init);
module_exit(nmk_gpio_tracer_exit);
```

---

### شرح كل جزء

#### الـ includes

**`linux/kprobes.h`** هو الـ header الأساسي اللي بيعرّف `struct kprobe` وكل الـ API بتاعه.
**`linux/gpio/gpio-nomadik.h`** محتاجينه عشان نعرف `struct nmk_gpio_chip` ونوصل لـ fields زي `bank` و`addr`.

#### الـ `handler_pre`

الـ callback ده بيتشغل قبل ما الـ CPU ينفذ الـ instruction الأولى في `__nmk_gpio_make_output`.
بنقرأ الـ arguments من الـ CPU registers مباشرةً حسب الـ calling convention، وبنطبع رقم الـ bank والـ offset والقيمة والـ GPIO number المطلق.

#### الـ `struct kprobe`

الـ `.symbol_name` بيخلي الـ kernel يدور على الدالة في الـ kallsyms ويحسب عنوانها أوتوماتيك.
الـ `.pre_handler` هو الـ pointer للـ callback اللي هيتشغل.

#### الـ `module_init`

`register_kprobe` بتكتب **breakpoint instruction** (INT3 على x86، BRK على ARM64) عند بداية الدالة في الـ memory.
بنتحقق من الـ return value عشان لو الدالة مش موجودة في الـ kernel نطلع بـ error واضح.

#### الـ `module_exit`

`unregister_kprobe` بتشيل الـ breakpoint وترجع الـ original bytes.
ده ضروري جدًا — لو سبنا الـ kprobe بعد ما الـ module اتشالت هيحصل kernel panic لأن الـ handler address بقى invalid.

---

### Makefile للبناء

```makefile
obj-m += nmk_gpio_tracer.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

### تشغيل وتجربة

```bash
# بناء الـ module
make

# تحميله
sudo insmod nmk_gpio_tracer.ko

# متابعة الـ output في الـ kernel log
sudo dmesg -w | grep nmk_gpio_tracer

# إزالته
sudo rmmod nmk_gpio_tracer
```

> **ملحوظة:** `__nmk_gpio_make_output` بيتشغل بس على hardware بيشغّل Nomadik/STN8815/DB8500 (مثلاً Samsung Galaxy S / ST-Ericsson platforms). على x86 VM هتلاقي `register_kprobe` بترجع `-ENOENT` لأن الدالة مش compiled في الـ kernel.
