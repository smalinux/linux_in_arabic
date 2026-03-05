## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي له الملف

الملف `gpio-exar.txt` جزء من **GPIO subsystem** في Linux kernel، وبالتحديد هو **Device Tree binding document** للـ GPIO controller الموجود في شرائح Exar UART.

---

### الصورة الكبيرة — تخيل الموضوع كده

تخيل إنك اشتريت **كارت PCI** بيضيف للكمبيوتر بتاعك عدة **منافذ Serial (UART)**، الكارت ده اسمه **Exar XR17V35X** — وده بيُستخدم كتير في الأنظمة الصناعية والـ embedded systems اللي محتاجة ports RS232/RS485 كتير.

دلوقتي، المفاجأة: الكارت ده مش بس بيوفر منافذ Serial، لكن جواه كمان **pins إضافية** اسمها **MPIO — Multi-Purpose I/O**، يعني pins ممكن تستخدمها كـ GPIO عادية — تقدر تجيب منها input أو تحط عليها output.

المشكلة إن الـ MPIO pins دي مش مستقلة، هي **مشتركة** مع الـ UART نفسه جوا نفس الـ chip ونفس الـ memory space، فكيف يعرف الـ Linux kernel:
- إمتى يبدأ الـ GPIO من؟ (أنهو pin الأول)
- كام GPIO متاح؟

**الجواب هو الملف ده** — `gpio-exar.txt`.

---

### ما هو الملف ده بالظبط؟

ده **Device Tree binding document**، مش كود C. هو توثيق رسمي بيشرح إزاي تكتب الـ **Device Tree node** الخاص بـ GPIO controller اللي جوا شريحة Exar — علشان الـ kernel driver يعرف يشتغل صح.

الـ **Device Tree** هو طريقة في Linux (بتتستخدم مع كتير من الأنظمة) لوصف الـ hardware للـ kernel من غير ما الكود يكون hard-coded. بدل ما الـ driver يخمّن، الـ bootloader أو الـ firmware بيقوله: "الـ hardware عندك فيه ده وده".

الملف بيقول: علشان تفعّل الـ GPIO في شريحة Exar، لازم تحدد في الـ Device Tree:
- **`exar,first-pin`**: رقم أول pin قابل للتصدير كـ GPIO (من 0 لـ 15)
- **`ngpios`**: عدد الـ GPIO pins المطلوبة (من 1 لـ 16)

---

### قصة المشكلة اللي الكود بيحلها

تخيل شركة صناعية عندها ماكينة محتاجة:
- 4 منافذ Serial للتواصل مع أجهزة قياس
- 3 pins للتحكم في مصابيح indicator
- 2 pins لقراءة sensors رقمية

بدل ما يشتروا كارتين — واحد للـ UART وواحد للـ GPIO — الـ Exar XR17V35X بيعمل الاتنين. لكن لما الـ Linux kernel بيبوت، الـ UART driver هو اللي بيفتح الكارت الأول. إزاي الـ GPIO driver يعرف أي جزء من الـ memory space بتاعت الكارت هو بتاعه؟

الجواب: الـ **8250_exar.c** (UART driver) بينشئ **platform device** للـ GPIO، وبيديه الـ properties اللي اتحددت في الـ Device Tree (أو firmware properties)، ومن بينها `exar,first-pin` و `ngpios`. الـ GPIO driver بعد كده بيعمل **regmap** على نفس الـ I/O memory اللي عمله الـ UART driver، وبيتحكم في الـ MPIO registers.

---

### ليه الـ `first-pin` مهم؟

الـ chip عندها 16 MPIO pin (0 لـ 15)، لكن ممكن بعضها اتستخدم جوا الـ board لأغراض تانية. الـ **`exar,first-pin`** بيقول للـ driver: "ابدأ من pin رقم X"، علشان الـ kernel ميحاولش يتحكم في pins اتحجزت للـ board نفسه.

---

### العلاقة بين الملفات

```
Device Tree / ACPI properties
          ↓
  gpio-exar.txt  ← التوثيق (الملف ده)
          ↓
  drivers/gpio/gpio-exar.c  ← الـ GPIO driver نفسه
          ↑
  drivers/tty/serial/8250/8250_exar.c  ← الـ UART driver
  (بينشئ platform_device للـ GPIO ويديله properties)
```

---

### الملفات المكوّنة للـ Subsystem ده

| الملف | الدور |
|---|---|
| `Documentation/devicetree/bindings/gpio/gpio-exar.txt` | التوثيق الرسمي لـ DT binding (الملف ده) |
| `drivers/gpio/gpio-exar.c` | الـ GPIO driver نفسه للـ Exar MPIO |
| `drivers/tty/serial/8250/8250_exar.c` | الـ UART/PCI driver الأب، بينشئ الـ GPIO platform device |
| `drivers/tty/serial/8250/8250_exar_st16c554.c` | دعم شرائح Exar قديمة |
| `include/linux/gpio/driver.h` | الـ GPIO chip API اللي بيستخدمها الـ driver |
| `Documentation/devicetree/bindings/gpio/gpio.txt` | التوثيق العام لـ GPIO bindings في DT |

---

### ملخص سريع

- **الـ Exar XR17V35X** شريحة UART + GPIO في نفس الوقت
- **الـ MPIO pins** هي الـ GPIO جوا الشريحة دي
- **الملف ده** بيوثق إزاي تعرّف الـ GPIO controller ده في الـ Device Tree
- المطلوب بس خاصيتين: `exar,first-pin` و `ngpios`
- الـ GPIO driver بيكون **child** للـ UART driver على مستوى الـ platform bus
## Phase 2: شرح الـ GPIO Framework

### المشكلة — ليه الـ GPIO Subsystem موجود أصلاً؟

أي SoC فيه عشرات الـ GPIO pins — بعضها على الـ SoC نفسه، وبعضها على chips خارجية (GPIO expanders) متصلة على I2C أو SPI، وبعضها جزء من chips متعددة الوظائف زي الـ Exar XR17V35X اللي هو UART chip بس فيه MPIO pins جنب الـ UART channels.

لو مفيش framework موحد، كل driver هيعمل:
- وصول مباشر لـ registers خاصة بيه
- API مختلف عن كل chip تاني
- consumer code (اللي بيستخدم الـ GPIO) محتاج يعرف تفاصيل كل chip بالاسم

النتيجة: chaos كامل — driver لـ LED بيختلف عن driver لـ button بيختلف عن driver لـ regulator كل واحد فيهم بيكلم hardware مختلف بطريقة مختلفة.

### الحل — الـ GPIOLIB Framework

الـ kernel حل المشكلة بـ **abstraction layer** اسمه **gpiolib**. الفكرة الجوهرية:

- كل GPIO controller (سواء كان على الـ SoC أو chip خارجية) بيسجّل نفسه كـ **`gpio_chip`**
- الـ framework بيديه رقم فريد (أو range من الأرقام) يمثل الـ pins بتاعته
- أي **consumer** (LED driver, button driver, regulator driver...) بيتكلم مع الـ framework بـ API موحد
- الـ framework بيـ**delegate** العمليات الفعلية للـ driver المسجّل

---

### الـ Big Picture Architecture

```
  ┌──────────────────────────────────────────────────────────────┐
  │                     User Space / sysfs                       │
  │           /sys/class/gpio/*   ,   /dev/gpiochipN             │
  └─────────────────────────┬────────────────────────────────────┘
                            │  ioctl / sysfs writes
  ┌─────────────────────────▼────────────────────────────────────┐
  │                   GPIO Character Device                       │
  │              (gpiolib-cdev.c — modern API)                   │
  └─────────────────────────┬────────────────────────────────────┘
                            │
  ┌─────────────────────────▼────────────────────────────────────┐
  │                    GPIOLIB Core                               │
  │   (gpiolib.c / gpiolib-of.c / gpiolib-acpi.c)               │
  │                                                               │
  │  ┌──────────────┐   ┌──────────────┐   ┌──────────────────┐ │
  │  │  gpio_device │   │  gpio_desc   │   │   gpio_chip      │ │
  │  │  (per chip)  │──▶│  (per pin)   │──▶│  (driver ops)    │ │
  │  └──────────────┘   └──────────────┘   └──────────────────┘ │
  │                                                               │
  │  Consumer API:  gpiod_get(), gpiod_set_value(), gpiod_to_irq()│
  └─────────────────────────┬────────────────────────────────────┘
                            │  calls chip->get/set/direction_*
         ┌──────────────────┼──────────────────┐
         │                  │                  │
  ┌──────▼──────┐   ┌───────▼──────┐   ┌──────▼───────────────┐
  │  SoC GPIO   │   │  I2C/SPI     │   │  Exar XR17V35X       │
  │  (e.g.      │   │  Expander    │   │  MPIO pins            │
  │  pl061.c)   │   │  (pcf857x.c) │   │  (gpio-exar.c)        │
  └─────────────┘   └──────────────┘   └──────────────────────┘
  Hardware: SoC     Hardware: I2C bus   Hardware: PCI MMIO regs
  registers         registers           (MPIOLVL / MPIOSEL)
```

---

### التشبيه الواقعي — موظف استقبال في شركة كبيرة

تخيّل شركة فيها أقسام كتير (Sales, Engineering, Finance). كل قسم فيه تليفونات داخلية — بعضها Cisco، بعضها Avaya، بعضها أجهزة قديمة.

| العنصر في التشبيه | المقابل في الـ Kernel |
|---|---|
| الشركة كلها | الـ kernel |
| موظف الاستقبال (Receptionist) | الـ GPIOLIB Core |
| دليل التليفونات الداخلي | الـ global GPIO number space / gpio_desc array |
| كل قسم (Sales, Eng...) | كل `gpio_chip` مسجّل |
| الـ extension number | الـ GPIO offset داخل الـ chip |
| نوع الجهاز (Cisco/Avaya) | نوع الـ hardware (SoC / I2C expander / PCI chip) |
| الشخص اللي عايز يتصل | الـ consumer driver (LED / button / regulator) |

الـ consumer بيقول للـ receptionist "عايز أتكلم مع extension 42"، الـ receptionist بيترجم ده لـ "ده في قسم Engineering على جهاز Avaya" وبيعمل الاتصال. الـ consumer مش محتاج يعرف حاجة عن نوع الجهاز.

**لكن التشبيه أعمق من كده:**
- الـ `gpio_desc` هو الـ "entry في دليل التليفونات" — بيربط بين الرقم العالمي وبين الـ chip + offset
- الـ `gpio_chip.get()` هو "السنترال الداخلي لكل قسم" — كل قسم عنده طريقته الخاصة
- الـ `devm_gpiochip_add_data()` هو "تسجيل القسم الجديد في الدليل"
- الـ `first_pin` في الـ Exar هو "أول extension متاح في الفرع الجديد"

---

### الـ Core Abstraction — `struct gpio_chip`

الـ **`struct gpio_chip`** هو **قلب الـ GPIO framework**. ده الـ interface اللي كل GPIO driver لازم يـfill in عشان يتكلم مع الـ gpiolib core.

```c
struct gpio_chip {
    const char    *label;       /* اسم الـ controller للـ debug */
    struct device *parent;      /* الـ parent device */

    /* --- الـ ops اللي الـ driver بيimplementها --- */
    int  (*get_direction)(struct gpio_chip *gc, unsigned int offset);
    int  (*direction_input)(struct gpio_chip *gc, unsigned int offset);
    int  (*direction_output)(struct gpio_chip *gc, unsigned int offset, int value);
    int  (*get)(struct gpio_chip *gc, unsigned int offset);
    int  (*set)(struct gpio_chip *gc, unsigned int offset, int value);

    /* --- metadata --- */
    int  base;    /* أول رقم GPIO عالمي (-1 = dynamic allocation) */
    u16  ngpio;   /* عدد الـ pins اللي الـ chip بتتحكم فيها */
    bool can_sleep; /* true لو الـ get/set ممكن تنام (I2C/SPI expanders) */

    /* --- optional IRQ support --- */
    struct gpio_irq_chip irq;
};
```

**الفكرة الجوهرية:** الـ driver بيملأ الـ function pointers دي، والـ framework بيـ**own** الـ routing.

---

### علاقة الـ structs ببعض

```
struct exar_gpio_chip               struct gpio_chip
┌─────────────────────┐            ┌──────────────────────────────┐
│ gpio_chip    ───────┼──────────▶ │ label = "exar_gpio0"         │
│ regmap       ───┐   │            │ parent = &pdev->dev          │
│ index            │   │            │ ngpio = 16 (or 32 cascaded)  │
│ name[20]         │   │            │ base = -1 (dynamic)          │
│ first_pin = N    │   │            │ get_direction = exar_get_dir │
│ cascaded_offset  │   │            │ direction_input = exar_dir_in│
└─────────────────────┘            │ direction_output= exar_dir_out│
          │                        │ get = exar_get_value         │
          │                        │ set = exar_set_value         │
          │                        └──────────────────────────────┘
          │
          ▼
    struct regmap
    ┌────────────────────────┐
    │ reg_bits = 16          │
    │ val_bits = 8           │
    │ io_port = true         │   ──▶  PCI MMIO region 0
    └────────────────────────┘
              │
              ▼
    Hardware Registers:
    ┌─────────────────────────────────────────┐
    │ 0x90: MPIOLVL_LO  (pins 0..7  level)    │
    │ 0x93: MPIOSEL_LO  (pins 0..7  dir)      │
    │ 0x96: MPIOLVL_HI  (pins 8..15 level)    │
    │ 0x99: MPIOSEL_HI  (pins 8..15 dir)      │
    └─────────────────────────────────────────┘
```

**الـ `gpiochip_get_data()`** — دالة مهمة جداً في الـ framework: بتـretrieve الـ driver-private data (اللي في حالتنا `exar_gpio_chip`) من خلال الـ `gpio_chip` pointer. ده بيخلي الـ ops functions توصل لبيانات الـ driver من غير global variables.

---

### الـ regmap Subsystem — مفهوم مهم هنا

> **الـ regmap** هو abstraction layer للـ register access — بيوحّد الوصول لـ hardware registers سواء كانت MMIO أو I2C أو SPI. مش جزء من الـ GPIO framework، لكن الـ gpio-exar.c بيستخدمه عشان يقرأ ويكتب الـ MPIO registers.

الـ Exar driver بيستخدم `devm_regmap_init_mmio()` مع الـ PCI BAR0 الـ MMIO region. بعدين بدل ما يعمل `ioread8(p + offset)` مباشرة، بيستخدم:
- `regmap_test_bits()` — لقراءة bit معين
- `regmap_set_bits()` / `regmap_clear_bits()` — لضبط direction
- `regmap_write_bits()` — لكتابة value بـ mask

---

### كيف الـ Exar chip بتبني على الـ GPIO Framework

الـ Exar XR17V35X هو **Multi-Port UART chip** متصل على PCI. جنب كل الـ UART channels، الـ chip فيها **MPIO pins** (Multi-Purpose I/O) — دي basically GPIO pins لكن جوّا chip UART.

**الـ device tree binding** بيقول:
```
exar,first-pin = <0>;   /* ابدأ من Pin 0 من الـ 16 */
ngpios = <8>;           /* export 8 pins بس */
```

الـ `first_pin` بيسمح لـ consumer يقول "أنا مش عايز كل الـ MPIO pins، عايز فقط من pin 4 للـ 11 مثلاً".

**الـ offset translation:**
```c
/* كل call بييجي بـ offset (0-based من ناحية الـ framework) */
/* الـ driver بيترجمه لـ register address + bit position */
static unsigned int
exar_offset_to_sel_addr(struct exar_gpio_chip *exar_gpio, unsigned int offset)
{
    unsigned int pin = exar_gpio->first_pin + (offset % 16);
    unsigned int cascaded = offset / 16;  /* هل ده على الـ secondary chip؟ */
    unsigned int addr = pin / 8 ? EXAR_OFFSET_MPIOSEL_HI : EXAR_OFFSET_MPIOSEL_LO;
    return addr + (cascaded ? exar_gpio->cascaded_offset : 0);
}
```

الـ `cascaded_offset` بيتحسب من الـ PCI Device ID — الـ 4 bits الأخيرة بتقول عدد الـ UART channels في الـ primary، وده بيحدد مكان الـ secondary chip's registers.

---

### ما الـ Framework بيملكه مقابل ما بيـdelegate للـ Driver

| المسؤولية | مين بيعملها |
|---|---|
| Global GPIO number space management | **gpiolib core** |
| mapping بين GPIO number و chip+offset | **gpiolib core** |
| sysfs + chardev interface | **gpiolib core** |
| OF/ACPI device tree parsing | **gpiolib core** |
| IRQ domain setup (لو الـ chip بتدعم interrupts) | **gpiolib core** (بمساعدة الـ driver) |
| قراءة/كتابة الـ hardware registers | **الـ driver** (gpio-exar.c) |
| تحديد direction (input/output) | **الـ driver** |
| حساب الـ register address من الـ offset | **الـ driver** |
| PCI resource mapping | **الـ parent UART driver** (مش حتى gpio-exar!) |
| regmap initialization | **الـ driver** |

**ملاحظة مهمة على الـ Exar:** الـ `gpio_exar_probe()` مش بتـmap الـ PCI BAR نفسه — بتـreuse الـ map اللي الـ parent UART driver عمله بالفعل:
```c
p = pcim_iomap_table(pcidev)[0];  /* BAR0 mapped by UART driver */
```
ده نموذج لـ **platform_device** اللي بيتخلق من parent driver — الـ UART driver بيعمل `platform_device_register()` للـ GPIO جزء كـ child device.

---

### الـ IDA — إدارة الـ Indexes

الـ **`IDA` (ID Allocator)** هو subsystem بسيط في الـ kernel بيخليك تأخذ integer IDs فريدة. الـ gpio-exar بيستخدمه عشان يديّ كل instance اسم فريد (`exar_gpio0`, `exar_gpio1`, ...) في حالة وجود أكتر من card في النظام.

```c
static DEFINE_IDA(ida_index);   /* global IDA للـ driver كله */
// في الـ probe:
index = ida_alloc(&ida_index, GFP_KERNEL);
sprintf(exar_gpio->name, "exar_gpio%d", index);
// في الـ remove (عن طريق devm):
ida_free(&ida_index, exar_gpio->index);
```

الـ `devm_add_action_or_reset()` بيضمن إن الـ IDA entry هتتـfree automatically لما الـ device يتـremove — ده جزء من نموذج **devm (device-managed resources)**.

---

### تسجيل الـ chip في الـ Framework

```c
/* الخطوة الأخيرة في الـ probe — بتسجّل الـ chip في الـ gpiolib */
ret = devm_gpiochip_add_data(dev, &exar_gpio->gpio_chip, exar_gpio);
```

بعد الـ call ده:
1. الـ framework بيـassign رقم base (لأن `base = -1`)
2. بيخلق `gpio_device` و `gpio_desc` array (واحد لكل pin)
3. بيظهر في `/sys/class/gpio/gpiochipN`
4. بيظهر في `/dev/gpiochipN` للـ character device API

الـ `exar_gpio` pointer (الـ driver-private data) بيـ**pass كـ data** — الـ framework بيخزنه ويـreturn له عن طريق `gpiochip_get_data()` في كل callback.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Register Map — Cheatsheet

| الـ Macro | القيمة | الوظيفة |
|---|---|---|
| `EXAR_OFFSET_MPIOLVL_LO` | `0x90` | قراءة/كتابة قيم الـ GPIO pins 0–7 |
| `EXAR_OFFSET_MPIOSEL_LO` | `0x93` | تحديد اتجاه الـ GPIO pins 0–7 (input/output) |
| `EXAR_OFFSET_MPIOLVL_HI` | `0x96` | قراءة/كتابة قيم الـ GPIO pins 8–15 |
| `EXAR_OFFSET_MPIOSEL_HI` | `0x99` | تحديد اتجاه الـ GPIO pins 8–15 |
| `EXAR_UART_CHANNEL_SIZE` | `0x400` | حجم الـ memory region لكل UART channel (1KB) |

---

### الـ Regmap Config — Cheatsheet

| الحقل | القيمة | المعنى |
|---|---|---|
| `reg_bits` | 16 | عنوان الـ register بـ 16 bit |
| `val_bits` | 8 | قيمة الـ register بـ 8 bit |
| `io_port` | `true` | الـ access بيتم عبر MMIO |
| `name` | `"exar-gpio"` | اسم الـ regmap للـ debugging |

---

### الـ Device Tree Properties — Cheatsheet

| الـ Property | النوع | القيم المسموحة | الوظيفة |
|---|---|---|---|
| `exar,first-pin` | `u32` | 0–15 | أول pin قابل للتصدير من الـ hardware |
| `ngpios` | `u32` | 1–16 | عدد الـ GPIO pins المطلوبة |

---

### الـ Cascade Detection — Cheatsheet

| الشرط | المعنى |
|---|---|
| `pcidev->device & GENMASK(15, 12)` | الـ chip مكاسكيدة مع secondary chip |
| `ngpios += ngpios` | بيتضاعف عدد الـ pins عشان يشمل الـ secondary |
| `pcidev->device & GENMASK(3, 0)` | عدد UART channels في الـ primary chip |
| `cascaded_offset = channels * 0x400` | offset الـ secondary chip's registers |

---

### الـ Structs المهمة

#### `struct exar_gpio_chip`

ده الـ struct الأساسي في الـ driver — بيجمع كل المعلومات اللي تخص الـ GPIO controller الخاص بـ Exar.

```c
struct exar_gpio_chip {
    struct gpio_chip  gpio_chip;       /* embedded gpio_chip — التسجيل في kernel */
    struct regmap    *regmap;          /* abstraction layer للـ MMIO access */
    int               index;           /* رقم فريد من الـ IDA لتسمية الـ chip */
    char              name[20];        /* "exar_gpioN" — label في sysfs */
    unsigned int      first_pin;       /* أول pin فعلي في الـ hardware (0..15) */
    unsigned int      cascaded_offset; /* offset الـ secondary chip (0 لو مفيش cascade) */
};
```

**الـ first_pin** بيسمح للـ platform device إنه يشير لجزء معين من الـ MPIO register بدل ما يبدأ من صفر دايمًا. مثلًا لو `first_pin=8` يبقى الـ driver بيتعامل مع الـ HI registers مباشرة.

**الـ cascaded_offset** بيكون قيمته `N * 0x400` حيث N هو عدد الـ UART channels في الـ primary chip. الـ secondary chip registers موجودة بعد كل channels الـ primary في الـ memory map.

---

#### `struct gpio_chip` (kernel struct — مستخدمة embedded)

الـ `gpio_chip` هو الـ interface الرسمي اللي الـ kernel GPIO subsystem بيشوفه.

| الحقل | القيمة المضبوطة | المعنى |
|---|---|---|
| `label` | `exar_gpio->name` | الاسم في sysfs |
| `parent` | `dev` | الـ platform_device |
| `direction_output` | `exar_direction_output` | callback لتحويل pin لـ output |
| `direction_input` | `exar_direction_input` | callback لتحويل pin لـ input |
| `get_direction` | `exar_get_direction` | callback لقراءة اتجاه الـ pin |
| `get` | `exar_get_value` | callback لقراءة قيمة الـ pin |
| `set` | `exar_set_value` | callback لكتابة قيمة الـ pin |
| `base` | `-1` | الـ kernel يختار الـ base number تلقائيًا |
| `ngpio` | من الـ DT | عدد الـ GPIO lines |

---

#### `struct regmap_config exar_regmap_config` (static instance)

```c
static const struct regmap_config exar_regmap_config = {
    .name     = "exar-gpio",
    .reg_bits = 16,   /* 16-bit address space */
    .val_bits = 8,    /* 8-bit registers */
    .io_port  = true, /* MMIO access */
};
```

الـ `regmap` بيعمل abstraction بين الـ driver والـ hardware — بدله كان لازم نستخدم `ioread8`/`iowrite8` مباشرة. ميزته إنه بيدي locking تلقائي وبيسهّل الـ debugging.

---

### Struct Relationship Diagram

```
platform_device (pdev)
│
├── dev.parent ──────────────────► pci_dev (pcidev)
│                                   └── device: PCI Device ID
│                                        ├── GENMASK(15,12): cascade?
│                                        └── GENMASK(3,0): num UART channels
│
└── dev (struct device)
     │
     └── devm_kzalloc()
          │
          ▼
     exar_gpio_chip
     ├── gpio_chip ◄──── embedded
     │    ├── label ──────► name[20]  ("exar_gpio0")
     │    ├── parent ─────► dev
     │    ├── ngpio
     │    └── ops ─────────► exar_get_direction()
     │                        exar_get_value()
     │                        exar_set_value()
     │                        exar_direction_input()
     │                        exar_direction_output()
     │
     ├── regmap ◄──── devm_regmap_init_mmio()
     │    └── base addr ──► pcim_iomap_table(pcidev)[0]
     │                       (MMIO region 0 of PCI device)
     │
     ├── index ◄──── ida_alloc(&ida_index)
     │
     ├── first_pin ◄── device_property_read_u32("exar,first-pin")
     │
     └── cascaded_offset ◄── calculated from pcidev->device
```

---

### Register Address Calculation Diagram

```
GPIO offset (logical, 0-based from driver's view)
│
├── pin = first_pin + (offset % 16)
│         └── Physical pin number in hardware
│
├── cascaded = offset / 16
│              └── 0 = primary chip, 1 = secondary chip
│
├── SEL register address:
│    ├── pin / 8 == 0  →  MPIOSEL_LO (0x93)
│    └── pin / 8 == 1  →  MPIOSEL_HI (0x99)
│    └── + cascaded_offset  (if secondary chip)
│
├── LVL register address:
│    ├── pin / 8 == 0  →  MPIOLVL_LO (0x90)
│    └── pin / 8 == 1  →  MPIOLVL_HI (0x96)
│    └── + cascaded_offset  (if secondary chip)
│
└── bit position in register:
     bit = pin % 8
     └── 0..7 within the 8-bit register
```

**مثال عملي:** لو `first_pin=4` والـ offset=3:
- `pin = 4 + 3 = 7` → pin 7
- `cascaded = 0` → primary chip
- `7 / 8 = 0` → MPIOLVL_LO = `0x90`
- `bit = 7 % 8 = 7` → bit 7 في الـ register

---

### Lifecycle Diagram

```
MODULE LOAD
    │
    ▼
module_platform_driver(gpio_exar_driver)
    │
    └── platform_driver_register()
         └── binds to "gpio_exar" platform devices

DEVICE PROBE: gpio_exar_probe()
    │
    ├── 1. get pci_dev parent
    │        pcidev = to_pci_dev(dev->parent)
    │
    ├── 2. get MMIO base
    │        p = pcim_iomap_table(pcidev)[0]
    │
    ├── 3. read DT properties
    │        first_pin ← "exar,first-pin"
    │        ngpios    ← "ngpios"
    │
    ├── 4. detect cascade
    │        if (pcidev->device & GENMASK(15,12))
    │            ngpios *= 2
    │            cascaded_offset = channels * 0x400
    │
    ├── 5. allocate exar_gpio_chip (devm)
    │
    ├── 6. init regmap
    │        devm_regmap_init_mmio(dev, p, &exar_regmap_config)
    │
    ├── 7. alloc IDA index
    │        ida_alloc(&ida_index)
    │        devm_add_action_or_reset(exar_devm_ida_free)
    │
    ├── 8. fill gpio_chip fields
    │        .label, .parent, .ops, .base=-1, .ngpio
    │
    └── 9. register GPIO chip
             devm_gpiochip_add_data()
                  └── GPIO subsystem assigns base number
                  └── chip visible in /sys/class/gpio/

DEVICE REMOVE (automatic via devm)
    │
    ├── devm_gpiochip_add_data → gpiochip_remove()
    ├── devm_regmap_init_mmio → regmap freed
    ├── exar_devm_ida_free → ida_free()
    └── devm_kzalloc → kfree()
```

---

### Call Flow Diagrams

#### قراءة قيمة pin (get)

```
userspace: cat /sys/class/gpio/gpioN/value
    │
    ▼
gpio_chip.get(chip, offset)
    = exar_get_value(chip, offset)
        │
        ├── exar_offset_to_lvl_addr(exar_gpio, offset)
        │    ├── pin = first_pin + (offset % 16)
        │    └── returns MPIOLVL_LO or MPIOLVL_HI + cascaded_offset
        │
        ├── exar_offset_to_bit(exar_gpio, offset)
        │    └── returns pin % 8
        │
        └── regmap_test_bits(regmap, addr, BIT(bit))
                 │
                 └── MMIO read at (base + addr)
                      └── test bit N in the byte
                           └── return 0 or 1
```

#### تحويل pin لـ output وكتابة قيمة

```
userspace: echo "out" > direction; echo "1" > value
    │
    ▼
gpio_chip.direction_output(chip, offset, value)
    = exar_direction_output(chip, offset, value)
        │
        ├── exar_set_value(chip, offset, value)
        │    ├── addr = exar_offset_to_lvl_addr()
        │    ├── bit  = exar_offset_to_bit()
        │    └── regmap_write_bits(regmap, addr, BIT(bit), bit_value)
        │             └── MMIO write: set/clear bit in MPIOLVL register
        │
        └── regmap_clear_bits(regmap, sel_addr, BIT(bit))
                 └── MMIO write: clear bit in MPIOSEL register
                      └── bit=0 → pin is OUTPUT
```

#### تحويل pin لـ input

```
gpio_chip.direction_input(chip, offset)
    = exar_direction_input(chip, offset)
        │
        ├── addr = exar_offset_to_sel_addr()
        ├── bit  = exar_offset_to_bit()
        └── regmap_set_bits(regmap, addr, BIT(bit))
                 └── MMIO write: set bit in MPIOSEL register
                      └── bit=1 → pin is INPUT
```

#### قراءة الاتجاه (get_direction)

```
gpio_chip.get_direction(chip, offset)
    = exar_get_direction(chip, offset)
        │
        ├── addr = exar_offset_to_sel_addr()
        ├── bit  = exar_offset_to_bit()
        └── regmap_test_bits(regmap, addr, BIT(bit))
                 ├── bit=1 → GPIO_LINE_DIRECTION_IN
                 └── bit=0 → GPIO_LINE_DIRECTION_OUT
```

---

### Register Bit Semantics

```
MPIOSEL register (direction):
  bit = 1  →  INPUT  (pin configured as input)
  bit = 0  →  OUTPUT (pin configured as output)

MPIOLVL register (level/value):
  bit = 1  →  HIGH (logic 1)
  bit = 0  →  LOW  (logic 0)

Each 8-bit register controls 8 consecutive pins:
  _LO registers → pins 0..7
  _HI registers → pins 8..15
```

---

### Locking Strategy

الـ driver مفيهوش **أي locking صريح** في الكود. ده مقصود عشان:

1. **الـ regmap بيعمل internal locking** تلقائيًا — كل عملية `regmap_write_bits`, `regmap_set_bits`, `regmap_clear_bits`, `regmap_test_bits` محمية بـ mutex داخلي في الـ regmap framework.

2. **الـ GPIO subsystem** نفسه بيدير الـ locking على مستوى الـ `gpio_chip` operations من خلال `spinlock` داخلي في `gpio_desc`.

3. **الـ IDA** (`ida_index`) يستخدم `ida_alloc`/`ida_free` اللي thread-safe بطبيعتها.

```
Lock Hierarchy:
─────────────────────────────────────────
GPIO subsystem lock  (gpio_desc spinlock)
    │
    └── regmap internal mutex
              │
              └── MMIO hardware access
                   (no separate chip lock needed)
─────────────────────────────────────────
IDA lock (separate, no ordering concern)
─────────────────────────────────────────
```

**ملاحظة مهمة:** الـ comment في `exar_set_value` بيوضح إن `regmap_write_bits()` بيكتب القيمة **دايمًا** حتى لو الـ register فعلًا عنده نفس القيمة — ده ضروري عشان الـ external pull-up/pull-down مش بيتعكس في الـ register value دايمًا.
## Phase 4: شرح الـ Functions

---

### ملخص كل الـ Functions والـ APIs

#### جدول الـ Helper Functions (Address/Bit Calculation)

| Function | Input | Output | الغرض |
|---|---|---|---|
| `exar_offset_to_sel_addr` | `exar_gpio`, `offset` | `unsigned int` addr | عنوان رجستر MPIOSEL |
| `exar_offset_to_lvl_addr` | `exar_gpio`, `offset` | `unsigned int` addr | عنوان رجستر MPIOLVL |
| `exar_offset_to_bit` | `exar_gpio`, `offset` | `unsigned int` bit | رقم الـ bit داخل الـ register |

#### جدول الـ GPIO Chip Operations

| Function | Callback في `gpio_chip` | الغرض |
|---|---|---|
| `exar_get_direction` | `.get_direction` | قراءة اتجاه الـ pin (input/output) |
| `exar_get_value` | `.get` | قراءة قيمة الـ pin |
| `exar_set_value` | `.set` | كتابة قيمة على الـ pin |
| `exar_direction_output` | `.direction_output` | تحويل الـ pin لـ output مع set القيمة |
| `exar_direction_input` | `.direction_input` | تحويل الـ pin لـ input |

#### جدول الـ Lifecycle Functions

| Function | النوع | الغرض |
|---|---|---|
| `gpio_exar_probe` | `platform_driver.probe` | تهيئة وتسجيل الـ driver |
| `exar_devm_ida_free` | devm cleanup action | تحرير الـ IDA index |
| `module_platform_driver` | macro | تسجيل وإلغاء تسجيل الـ platform driver |

---

### أولاً: Helper Functions — حساب العناوين والـ Bits

الـ Exar XR17V35X بيعمل map للـ MPIO registers على offset ثابت من base الـ UART channel. الـ pins مقسمة لـ groups من 8 (LO: pins 0–7، HI: pins 8–15). الـ 3 functions دول هم core الـ address translation اللي كل الـ GPIO ops بتعتمد عليه.

---

#### `exar_offset_to_sel_addr`

```c
static unsigned int
exar_offset_to_sel_addr(struct exar_gpio_chip *exar_gpio, unsigned int offset)
{
    unsigned int pin = exar_gpio->first_pin + (offset % 16);
    unsigned int cascaded = offset / 16;
    unsigned int addr = pin / 8 ? EXAR_OFFSET_MPIOSEL_HI : EXAR_OFFSET_MPIOSEL_LO;

    return addr + (cascaded ? exar_gpio->cascaded_offset : 0);
}
```

**الـ function دي بتحسب عنوان رجستر MPIOSEL** (المسؤول عن direction: input/output) للـ GPIO offset المطلوب. بتحسب أول الـ absolute pin number بإضافة `first_pin`، وبعدين بتقرر هل الـ pin في الـ LO register (pins 0–7) أو HI register (pins 8–15). لو الـ offset بيشير لـ cascaded device بتضيف `cascaded_offset`.

**Parameters:**
- `exar_gpio`: pointer للـ chip struct اللي فيه `first_pin` و`cascaded_offset`
- `offset`: رقم الـ GPIO من وجهة نظر الـ `gpio_chip` (0-based، relative)

**Return:** عنوان الـ MPIOSEL register المناسب (16-bit، يُستخدم مع `regmap`)

**Key details:**
- `offset % 16`: بيعمل wrap كل 16 pin لأن كل device عنده 16 MPIO كحد أقصى
- `offset / 16`: بيحدد هل الـ pin في الـ primary device أو الـ cascaded device
- الـ `cascaded_offset` بيُحسب في `probe` بناءً على الـ PCI Device ID

---

#### `exar_offset_to_lvl_addr`

```c
static unsigned int
exar_offset_to_lvl_addr(struct exar_gpio_chip *exar_gpio, unsigned int offset)
{
    unsigned int pin = exar_gpio->first_pin + (offset % 16);
    unsigned int cascaded = offset / 16;
    unsigned int addr = pin / 8 ? EXAR_OFFSET_MPIOLVL_HI : EXAR_OFFSET_MPIOLVL_LO;

    return addr + (cascaded ? exar_gpio->cascaded_offset : 0);
}
```

**نفس منطق `exar_offset_to_sel_addr` بالظبط** لكن بترجع عنوان رجستر MPIOLVL (المسؤول عن الـ level/value للـ pin). الـ `MPIOLVL` registers هي اللي بتتقرى وبتتكتب فيها قيم الـ GPIO.

**Parameters:** نفس الـ parameters السابقة.

**Return:** عنوان الـ MPIOLVL register (LO أو HI) للـ primary أو cascaded device.

**الفرق عن السابقة:** بتستخدم `EXAR_OFFSET_MPIOLVL_LO` (0x90) و`EXAR_OFFSET_MPIOLVL_HI` (0x96) بدل الـ SEL registers (0x93 و 0x99).

---

#### `exar_offset_to_bit`

```c
static unsigned int
exar_offset_to_bit(struct exar_gpio_chip *exar_gpio, unsigned int offset)
{
    unsigned int pin = exar_gpio->first_pin + (offset % 16);

    return pin % 8;
}
```

**بتحسب رقم الـ bit داخل الـ 8-bit register** الخاص بالـ pin المطلوب. كل register عرضه 8 bits، فالـ function بترجع الـ bit position (0–7) داخل الـ byte المقابل.

**Parameters:**
- `exar_gpio`: للوصول لـ `first_pin`
- `offset`: الـ GPIO offset

**Return:** bit position (0–7) داخل الـ register

**Key detail:** مش بتهتم بالـ cascaded offset لأن الـ bit position بيتحدد فقط بالـ pin number داخل الـ group بتاعه (LO أو HI).

---

### ثانياً: GPIO Chip Operations — الـ Runtime Callbacks

الـ functions دول بيتم تسجيلهم كـ callbacks في `struct gpio_chip` ويتاح استدعاؤهم من الـ GPIO subsystem أو من الـ user-space عبر `gpiolib`.

---

#### `exar_get_direction`

```c
static int exar_get_direction(struct gpio_chip *chip, unsigned int offset)
{
    struct exar_gpio_chip *exar_gpio = gpiochip_get_data(chip);
    unsigned int addr = exar_offset_to_sel_addr(exar_gpio, offset);
    unsigned int bit = exar_offset_to_bit(exar_gpio, offset);

    if (regmap_test_bits(exar_gpio->regmap, addr, BIT(bit)))
        return GPIO_LINE_DIRECTION_IN;

    return GPIO_LINE_DIRECTION_OUT;
}
```

**بتقرأ رجستر MPIOSEL وبتشوف هل الـ bit المقابل للـ pin مضروب أو لأ.** في الـ Exar hardware، الـ bit = 1 معناه input والـ bit = 0 معناه output.

**Parameters:**
- `chip`: pointer لـ `gpio_chip`، بيُستخدم للوصول للـ `exar_gpio` عبر `gpiochip_get_data`
- `offset`: رقم الـ GPIO line

**Return:**
- `GPIO_LINE_DIRECTION_IN` (1) لو الـ pin configured كـ input
- `GPIO_LINE_DIRECTION_OUT` (0) لو configured كـ output

**Caller context:** بيتستدعى من الـ gpiolib core، ممكن من أي context مش interrupt.

**Key detail:** `regmap_test_bits` بيعمل read-modify بدون write، فـ atomic read فقط.

---

#### `exar_get_value`

```c
static int exar_get_value(struct gpio_chip *chip, unsigned int offset)
{
    struct exar_gpio_chip *exar_gpio = gpiochip_get_data(chip);
    unsigned int addr = exar_offset_to_lvl_addr(exar_gpio, offset);
    unsigned int bit = exar_offset_to_bit(exar_gpio, offset);

    return !!(regmap_test_bits(exar_gpio->regmap, addr, BIT(bit)));
}
```

**بتقرأ القيمة الحالية للـ GPIO pin من رجستر MPIOLVL.** بتستخدم `!!` لتحويل النتيجة لـ 0 أو 1 بشكل صريح.

**Parameters:**
- `chip`: الـ gpio_chip
- `offset`: رقم الـ GPIO

**Return:** 0 أو 1 — القيمة الحالية للـ pin

**Key detail:** شغالة على output و input pins على حد سواء؛ لـ output pins القيمة المقروءة هي اللي اتكتبت آخر مرة (لا يوجد readback من الـ hardware state إلا عبر نفس الـ register).

---

#### `exar_set_value`

```c
static int exar_set_value(struct gpio_chip *chip, unsigned int offset, int value)
{
    struct exar_gpio_chip *exar_gpio = gpiochip_get_data(chip);
    unsigned int addr = exar_offset_to_lvl_addr(exar_gpio, offset);
    unsigned int bit = exar_offset_to_bit(exar_gpio, offset);
    unsigned int bit_value = value ? BIT(bit) : 0;

    return regmap_write_bits(exar_gpio->regmap, addr, BIT(bit), bit_value);
}
```

**بتكتب قيمة (0 أو 1) على الـ GPIO pin عبر رجستر MPIOLVL.** الـ `regmap_write_bits` بيضمن إن الكتابة بتحصل دايمًا حتى لو القيمة الحالية نفسها — مهم لما يكون في external pull-up/down.

**Parameters:**
- `chip`: الـ gpio_chip
- `offset`: رقم الـ GPIO
- `value`: القيمة المطلوبة (0 أو أي قيمة non-zero)

**Return:** 0 لو نجح، أو error code من `regmap_write_bits`

**Key detail:** الـ comment في الكود بيوضح سبب استخدام `regmap_write_bits` بدل `regmap_update_bits` — لضمان الكتابة الفعلية بغض النظر عن الـ cached value في الـ regmap.

**Side effect:** بيحدث الـ regmap cache بالقيمة الجديدة.

---

#### `exar_direction_output`

```c
static int exar_direction_output(struct gpio_chip *chip, unsigned int offset, int value)
{
    struct exar_gpio_chip *exar_gpio = gpiochip_get_data(chip);
    unsigned int addr = exar_offset_to_sel_addr(exar_gpio, offset);
    unsigned int bit = exar_offset_to_bit(exar_gpio, offset);
    int ret;

    ret = exar_set_value(chip, offset, value);
    if (ret)
        return ret;

    return regmap_clear_bits(exar_gpio->regmap, addr, BIT(bit));
}
```

**بتحول الـ GPIO pin لـ output** بعد ما بتحدد الـ initial value الأول. الترتيب مهم: بتكتب القيمة الأول قبل ما تحول الـ direction — ده بيمنع glitch على الـ line.

**Parameters:**
- `chip`: الـ gpio_chip
- `offset`: رقم الـ GPIO
- `value`: القيمة الابتدائية للـ output

**Return:** 0 لو نجح، أو error code

**Pseudocode flow:**
```
1. exar_set_value(chip, offset, value)  ← اكتب القيمة أولاً
   if error → return error
2. regmap_clear_bits(MPIOSEL, BIT(bit)) ← حول لـ output (bit=0)
   return result
```

**Key detail:** `regmap_clear_bits` بيعمل read-modify-write على الـ MPIOSEL register — بيعمل clear للـ bit المقابل (0 = output في الـ Exar hardware).

---

#### `exar_direction_input`

```c
static int exar_direction_input(struct gpio_chip *chip, unsigned int offset)
{
    struct exar_gpio_chip *exar_gpio = gpiochip_get_data(chip);
    unsigned int addr = exar_offset_to_sel_addr(exar_gpio, offset);
    unsigned int bit = exar_offset_to_bit(exar_gpio, offset);

    regmap_set_bits(exar_gpio->regmap, addr, BIT(bit));

    return 0;
}
```

**بتحول الـ GPIO pin لـ input** بضرب الـ bit المقابل في رجستر MPIOSEL (1 = input).

**Parameters:**
- `chip`: الـ gpio_chip
- `offset`: رقم الـ GPIO

**Return:** دايمًا 0 (regmap MMIO operations مش بترجع errors في الـ setup ده)

**Key detail:** الكود بيعمل ignore للـ return value من `regmap_set_bits` — الـ comment في `probe` بيوضح إن الـ mmio regmap operations من غير clock مش بترجع errors حقيقية.

---

### ثالثاً: Lifecycle Functions — الـ Probe والـ Cleanup

---

#### `exar_devm_ida_free`

```c
static void exar_devm_ida_free(void *data)
{
    struct exar_gpio_chip *exar_gpio = data;

    ida_free(&ida_index, exar_gpio->index);
}
```

**Cleanup callback بيتسجل مع `devm_add_action_or_reset`** لتحرير الـ IDA index لما الـ device بيتفصل أو في حالة error في الـ probe. ده جزء من الـ devm cleanup chain.

**Parameters:**
- `data`: void pointer للـ `exar_gpio_chip`، بيتعمله cast داخليًا

**Return:** void

**Caller context:** بيتستدعى من `devres_release_all` في سياق الـ device teardown أو من `devm_add_action_or_reset` نفسه لو حصل error بعد التسجيل.

---

#### `gpio_exar_probe`

```c
static int gpio_exar_probe(struct platform_device *pdev)
```

**أهم function في الـ driver** — بتتولى تهيئة كل حاجة من قراءة الـ properties لـ تسجيل الـ gpio_chip.

**Parameters:**
- `pdev`: الـ platform_device اللي الـ UART driver سجله كـ child device

**Return:** 0 لو نجح، أو negative error code

**Pseudocode flow مفصل:**

```
1. احصل على parent PCI device
   pcidev = to_pci_dev(dev->parent)

2. احصل على MMIO base من الـ PCI BAR[0]
   p = pcim_iomap_table(pcidev)[0]
   if !p → return -ENOMEM
   /* الـ UART driver عمل map للـ region ده قبل كده */

3. اقرأ device properties من DT/ACPI
   device_property_read_u32(dev, "exar,first-pin", &first_pin)
   device_property_read_u32(dev, "ngpios", &ngpios)

4. allocate exar_gpio_chip
   exar_gpio = devm_kzalloc(...)

5. حدد لو في cascaded device
   if (pcidev->device & GENMASK(15, 12)):
       ngpios *= 2
       cascaded_offset = (device_id & 0xF) * EXAR_UART_CHANNEL_SIZE

6. أنشئ regmap على الـ MMIO base
   exar_gpio->regmap = devm_regmap_init_mmio(dev, p, &exar_regmap_config)

7. احجز IDA index فريد
   index = ida_alloc(&ida_index, GFP_KERNEL)

8. سجل cleanup action لتحرير الـ IDA
   devm_add_action_or_reset(dev, exar_devm_ida_free, exar_gpio)

9. اضبط gpio_chip fields
   - label, parent, callbacks, base=-1, ngpio

10. سجل الـ gpio_chip
    devm_gpiochip_add_data(dev, &exar_gpio->gpio_chip, exar_gpio)
```

**Key details:**

- **الـ MMIO mapping:** الـ driver مش بيعمل ioremap بنفسه — بيعتمد على الـ parent UART driver اللي بيكون عمل `pcim_iomap` على BAR[0] قبل ما يسجل الـ child platform device.

- **Cascaded device detection:** الـ Exar XR17V358 (مثلاً) بيكون cascaded مع XR17V354. الـ PCI Device ID bits [15:12] بتشير لوجود cascaded device، وبيتحسب `cascaded_offset` من عدد channels (bits [3:0]) مضروب في حجم الـ channel (0x400 = 1KB).

- **`base = -1`:** بيقول للـ gpiolib يختار الـ base number دينامياً — أفضل ممارسة في الـ modern kernel.

- **`devm_gpiochip_add_data`:** بيتولى تسجيل الـ gpio_chip وبيضمن إلغاء التسجيل تلقائيًا لما الـ device بيتفصل، من غير الحاجة لـ `.remove` callback.

- **Error handling:** كل resource بيتحجز بـ `devm_*` — لو أي step فشل، الـ devres framework بيتولى cleanup كل اللي اتحجز قبله.

---

### رابعاً: الـ Regmap Configuration

```c
static const struct regmap_config exar_regmap_config = {
    .name     = "exar-gpio",
    .reg_bits = 16,   /* عنوان الـ register: 16-bit */
    .val_bits = 8,    /* عرض الـ register: 8-bit */
    .io_port  = true, /* MMIO access */
};
```

الـ Exar MPIO registers هي 8-bit registers موجودة على offsets ثابتة من الـ MMIO base. الـ `io_port = true` بيقول للـ regmap إنه MMIO مش I/O port، وبالتالي بيستخدم `readb`/`writeb` مباشرة بدون caching مشكلة.

---

### خامساً: ASCII Diagram — تدفق الـ Address Calculation

```
GPIO offset (0-based)
        │
        ├─── offset % 16 ──► absolute pin = first_pin + (offset % 16)
        │                              │
        │                    pin / 8 ──┤
        │                              ├── 0 → MPIOSEL_LO (0x93) أو MPIOLVL_LO (0x90)
        │                              └── 1 → MPIOSEL_HI (0x99) أو MPIOLVL_HI (0x96)
        │                              │
        │                    pin % 8 ──► bit position داخل الـ register
        │
        └─── offset / 16 ──► cascaded?
                                  │
                          0 → +0 (primary device)
                          1 → +cascaded_offset (secondary device)
```

---

### سادساً: جدول الـ Register Map

| Register | Offset | الغرض |
|---|---|---|
| `EXAR_OFFSET_MPIOLVL_LO` | 0x90 | Level (value) لـ pins 0–7 |
| `EXAR_OFFSET_MPIOSEL_LO` | 0x93 | Direction لـ pins 0–7 (1=input, 0=output) |
| `EXAR_OFFSET_MPIOLVL_HI` | 0x96 | Level (value) لـ pins 8–15 |
| `EXAR_OFFSET_MPIOSEL_HI` | 0x99 | Direction لـ pins 8–15 |

الـ cascaded device registers على نفس الـ offsets لكن بيتضاف `cascaded_offset` (= عدد channels × 0x400).
## Phase 5: دليل الـ Debugging الشامل

### Software Level

---

#### 1. مدخلات الـ debugfs

الـ GPIO subsystem بيكتب معلوماته في `/sys/kernel/debug/gpio` مش في debugfs منفصل، لكن ممكن تشوف الـ regmap cache كمان:

```bash
# اقرا حالة كل الـ GPIO chips المسجلة
cat /sys/kernel/debug/gpio

# مثال output للـ exar_gpio0
# gpiochip0: GPIOs 496-511, parent: platform/exar_gpio, exar_gpio0:
#  gpio-496 (                    ) in  hi
#  gpio-497 (                    ) out lo
```

```bash
# شوف الـ regmap cache للـ exar-gpio
ls /sys/kernel/debug/regmap/
# هتلاقي مجلد اسمه exar-gpio أو رقم الـ PCI device

cat /sys/kernel/debug/regmap/exar-gpio*/registers
# Output بيكون:
# 0090: 00    <- MPIOLVL_LO (قيم الـ pins 0..7)
# 0093: ff    <- MPIOSEL_LO (اتجاه الـ pins 0..7, 1=input)
# 0096: 00    <- MPIOLVL_HI (قيم الـ pins 8..15)
# 0099: ff    <- MPIOSEL_HI (اتجاه الـ pins 8..15)
```

```bash
# شوف الـ regmap cache config
cat /sys/kernel/debug/regmap/exar-gpio*/config
```

---

#### 2. مدخلات الـ sysfs

```bash
# اتعرف على الـ chip
ls /sys/class/gpio/
# gpiochip496  gpiochip0  ...

# اقرا معلومات الـ chip
cat /sys/class/gpio/gpiochip496/base      # رقم أول GPIO
cat /sys/class/gpio/gpiochip496/ngpio     # عدد الـ GPIOs
cat /sys/class/gpio/gpiochip496/label     # exar_gpio0

# export pin معين للـ userspace
echo 496 > /sys/class/gpio/export

# اقرا الاتجاه والقيمة
cat /sys/class/gpio/gpio496/direction     # in أو out
cat /sys/class/gpio/gpio496/value        # 0 أو 1

# غير الاتجاه
echo out > /sys/class/gpio/gpio496/direction
echo 1   > /sys/class/gpio/gpio496/value

# شيل الـ export
echo 496 > /sys/class/gpio/unexport

# شوف الـ PCI device اللي الـ GPIO chip تابع له
ls -la /sys/class/gpio/gpiochip496/device/
```

---

#### 3. الـ ftrace — Tracepoints والـ Events

```bash
# شوف الـ events المتاحة للـ GPIO
grep -r gpio /sys/kernel/tracing/available_events

# enable أهم GPIO events
echo 1 > /sys/kernel/tracing/events/gpio/enable

# أو events محددة
echo 1 > /sys/kernel/tracing/events/regmap/regmap_reg_read/enable
echo 1 > /sys/kernel/tracing/events/regmap/regmap_reg_write/enable

# ابدأ الـ trace
echo 1 > /sys/kernel/tracing/tracing_on

# اعمل العملية اللي عايز تتبعها
echo 1 > /sys/class/gpio/gpio496/value

# وقف الـ trace واقرا النتيجة
echo 0 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace

# مثال output:
# <...>-1234 [000] .... 123.456: regmap_reg_write: exar-gpio reg=0090 val=01
# <...>-1234 [000] .... 123.457: regmap_reg_read:  exar-gpio reg=0090 val=01

# filter على الـ exar regmap بس
echo 'name == "exar-gpio"' > /sys/kernel/tracing/events/regmap/regmap_reg_write/filter
echo 'name == "exar-gpio"' > /sys/kernel/tracing/events/regmap/regmap_reg_read/filter
```

---

#### 4. الـ printk والـ Dynamic Debug

```bash
# فعّل الـ dynamic debug لكل الـ gpio_exar module
echo "module gpio_exar +p" > /sys/kernel/debug/dynamic_debug/control

# أو فعّل file معين
echo "file drivers/gpio/gpio-exar.c +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل مع file + line number
echo "file drivers/gpio/gpio-exar.c line 81 +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل كل الـ gpio subsystem
echo "module gpio* +pflmt" > /sys/kernel/debug/dynamic_debug/control
# p=print, f=function name, l=line, m=module, t=thread

# شيك على اللي اتفعّل
cat /sys/kernel/debug/dynamic_debug/control | grep gpio_exar

# شوف الـ kernel messages
dmesg -w | grep -i exar
dmesg -w | grep -i gpio

# لو محتاج تضبط الـ log level
echo 8 > /proc/sys/kernel/printk

# boot parameter للـ debugging من الأول
# في الـ kernel cmdline:
# dyndbg="module gpio_exar +p"
```

---

#### 5. الـ Kernel Config Options للـ Debugging

| CONFIG Option | الوصف |
|---|---|
| `CONFIG_GPIO_EXAR` | الـ driver نفسه — لازم يكون `=m` أو `=y` |
| `CONFIG_DEBUG_GPIO` | يفعّل checks إضافية في الـ GPIO core |
| `CONFIG_GPIOLIB` | الـ GPIO library الأساسية |
| `CONFIG_REGMAP` | الـ regmap framework |
| `CONFIG_REGMAP_MMIO` | دعم الـ MMIO في regmap |
| `CONFIG_REGMAP_DEBUGFS` | يظهر الـ regmap في debugfs |
| `CONFIG_DYNAMIC_DEBUG` | يسمح بالـ dynamic debug |
| `CONFIG_KALLSYMS` | يظهر اسماء الـ functions في stack traces |
| `CONFIG_DEBUG_KERNEL` | يفعّل كل الـ kernel debugging |
| `CONFIG_PROVE_LOCKING` | يكتشف مشاكل الـ locking |
| `CONFIG_DEBUG_DEVRES` | يتتبع الـ devm allocations |
| `CONFIG_PCI_DEBUG` | debugging للـ PCI layer |

```bash
# شيك على الـ config الحالي
zcat /proc/config.gz | grep -E "GPIO|REGMAP|DEBUG_GPIO"
# أو
grep -E "CONFIG_GPIO|CONFIG_REGMAP|CONFIG_DEBUG_GPIO" /boot/config-$(uname -r)
```

---

#### 6. أدوات خاصة بالـ Subsystem

##### الـ gpiotools (libgpiod)

```bash
# install
apt-get install gpiod   # Debian/Ubuntu
dnf install libgpiod-utils  # Fedora

# اعرض كل الـ chips
gpiodetect
# مثال:
# gpiochip0 [exar_gpio0] (16 lines)

# اعرض كل الـ lines في chip معين
gpioinfo gpiochip0
# مثال:
# gpiochip0 - 16 lines:
#     line   0:      unnamed       unused   input  active-high
#     line   1:      unnamed       unused  output  active-high

# اقرا قيمة pin
gpioget gpiochip0 0

# اكتب قيمة على pin
gpioset gpiochip0 0=1

# monitor تغييرات
gpiomon gpiochip0 0
```

##### الـ PCI tools

```bash
# شوف الـ Exar PCI device
lspci -v | grep -A 10 -i exar
lspci -n | grep "13a8"   # Exar vendor ID

# اقرا الـ PCI config space
lspci -xxx -s <bus:dev.fn>

# شيك على الـ resource المتاح
cat /proc/iomem | grep -i exar
cat /sys/bus/pci/devices/0000:*/resource
```

---

#### 7. جدول رسائل الـ Error الشائعة

| رسالة الـ Kernel | المعنى | الحل |
|---|---|---|
| `gpio_exar: failed to read exar,first-pin` | الـ property `exar,first-pin` مش موجودة في الـ device properties | تأكد إن الـ UART driver بيمرر الـ properties صح للـ platform device |
| `gpio_exar: failed to read ngpios` | الـ property `ngpios` مش موجودة | تأكد من الـ platform_device properties في الـ UART driver |
| `gpio_exar: probe failed: -ENOMEM` | الـ PCI BAR 0 مش mapped — الـ UART driver ما عملش `pcim_iomap` | تأكد إن الـ Exar UART driver اتحمّل قبل الـ GPIO driver |
| `regmap_init_mmio failed` | فشل في إنشاء الـ regmap — عادةً مشكلة في الـ MMIO pointer | تحقق من الـ PCI BAR 0 address في `/proc/iomem` |
| `ida_alloc failed` | نفد الـ IDA indices — نادر جداً | اعمل reboot أو شيك على الـ ida_index leak |
| `devm_gpiochip_add_data failed` | تعارض في الـ GPIO base numbers أو الـ chip مش اتسجلت | شيك `dmesg` لرسالة تفاصيل أكتر من الـ GPIO core |
| `gpiochip_add: base ... invalid` | الـ GPIO base number اتاخد قبل كده | الـ driver بيستخدم `base = -1` فده نادر؛ شيك الـ drivers التانية |
| `regmap_read failed: -EIO` | خطأ في القراءة من الـ MMIO — ممكن hardware fault | اتحقق من الـ PCI link state وإن الـ device مش في power-down |

---

#### 8. أماكن استراتيجية لـ dump_stack() و WARN_ON()

```c
static int gpio_exar_probe(struct platform_device *pdev)
{
    /* نقطة 1: بعد ما بنجيب الـ PCI iomap table */
    p = pcim_iomap_table(pcidev)[0];
    if (!p) {
        dev_err(dev, "BAR0 not mapped — UART driver not loaded?\n");
        dump_stack();   /* يظهر call stack من ستاك الـ probe */
        return -ENOMEM;
    }

    /* نقطة 2: تحقق من قيمة first_pin */
    WARN_ON(first_pin > 15);   /* first_pin لازم يكون 0..15 */

    /* نقطة 3: تحقق من ngpios */
    WARN_ON(ngpios == 0 || ngpios > 16);
}

static int exar_get_direction(struct gpio_chip *chip, unsigned int offset)
{
    /* نقطة 4: تحقق من الـ offset */
    WARN_ON(offset >= chip->ngpio);
    ...
}

static int exar_set_value(struct gpio_chip *chip, unsigned int offset, int value)
{
    /* نقطة 5: تحقق من قيمة الـ regmap write */
    int ret = regmap_write_bits(...);
    WARN_ON(ret < 0);   /* ده نظرياً ما المفروضش يحصل مع MMIO */
    return ret;
}
```

---

### Hardware Level

---

#### 1. التحقق إن حالة الـ Hardware تطابق حالة الـ Kernel

```bash
# اقرا قيمة الـ pin من الـ kernel
gpioget gpiochip0 0

# قارن بقيمة الـ MPIOLVL register مباشرة
# الـ base address من /proc/iomem
cat /proc/iomem | grep -i exar
# مثال: f0000000-f0003fff : 0000:01:00.0

# اقرا الـ MPIOLVL_LO (offset 0x90)
# استخدم devmem2 لو متاح
devmem2 0xf0000090 b    # b = byte
# المفروض يطابق قيمة الـ gpioget

# الـ MPIOSEL_LO (offset 0x93) — 1=input, 0=output
devmem2 0xf0000093 b
```

**جدول الـ Registers:**

| Register | Offset | الوصف |
|---|---|---|
| `MPIOLVL_LO` | `0x90` | قيمة الـ pins 0..7 (read/write) |
| `MPIOSEL_LO` | `0x93` | اتجاه الـ pins 0..7 (1=input, 0=output) |
| `MPIOLVL_HI` | `0x96` | قيمة الـ pins 8..15 (read/write) |
| `MPIOSEL_HI` | `0x99` | اتجاه الـ pins 8..15 (1=input, 0=output) |

---

#### 2. تقنيات الـ Register Dump

```bash
# طريقة 1: devmem2
# install: apt-get install devmem2
BASE=0xf0000000   # غير ده حسب /proc/iomem

devmem2 $((BASE + 0x90)) b    # MPIOLVL_LO
devmem2 $((BASE + 0x93)) b    # MPIOSEL_LO
devmem2 $((BASE + 0x96)) b    # MPIOLVL_HI
devmem2 $((BASE + 0x99)) b    # MPIOSEL_HI

# طريقة 2: /dev/mem مع dd
dd if=/dev/mem bs=1 skip=$((BASE + 0x90)) count=1 2>/dev/null | xxd

# طريقة 3: io utility (للـ IO ports — مش مناسب هنا لأن الـ exar بيستخدم MMIO)
# الـ exar_regmap_config.io_port = true بس ده flag في الـ regmap مش I/O port فعلي

# طريقة 4: من الـ regmap debugfs (الأفضل)
cat /sys/kernel/debug/regmap/*/registers | grep -E "0090|0093|0096|0099"

# script لـ dump كامل للـ MPIO registers
#!/bin/bash
BASE=$(cat /proc/iomem | grep "exar\|13a8" | head -1 | awk -F- '{print "0x"$1}')
echo "Base: $BASE"
for offset in 0x90 0x93 0x96 0x99; do
    val=$(devmem2 $((BASE + offset)) b 2>/dev/null | grep "Read" | awk '{print $NF}')
    echo "Offset $offset: $val"
done
```

---

#### 3. Logic Analyzer / Oscilloscope

**نقاط القياس:**

```
XR17V35X Chip
+------------------+
|  MPIO[0..7]  ----+---> قيس هنا الـ signal level
|  MPIO[8..15] ----+---> قيس هنا
|  VCC         ----+---> 3.3V أو 1.8V حسب الـ board
|  GND         ----+---> reference
+------------------+
```

**إعدادات الـ Logic Analyzer:**

- **Sample rate**: 1 MHz كافي لـ GPIO عادي، 10 MHz لو الـ toggling سريع
- **Voltage threshold**: 1.65V لـ 3.3V logic، 0.9V لـ 1.8V logic
- **Trigger**: Rising/Falling edge على الـ pin اللي بتتبعه

**الـ Oscilloscope:**

```bash
# اعمل toggle سريع للـ pin عشان تشوفه على الـ scope
while true; do
    gpioset gpiochip0 0=1
    gpioset gpiochip0 0=0
done
# المفروض تشوف square wave بـ frequency ثابتة
# لو الـ wave مش نظيفة → مشكلة في الـ pull resistors أو الـ load
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في الـ Kernel Log

| المشكلة | Pattern في الـ dmesg | السبب المرجح |
|---|---|---|
| الـ pin مش بيتغير | قراءة regmap صح بس الـ hardware مش بيستجاوب | شيك الـ external pull-up/down resistors |
| الـ pin بيقرا قيمة غلط دايماً | `gpio-496 (unnamed) in hi` وهو المفروض `lo` | floating input أو pull-up مش متوصل |
| الـ driver مش بيتحمّل | `gpio_exar: probe failed: -ENOMEM` | الـ UART driver مش حمّل BAR0 |
| الـ PCI device مش ظاهر | مفيش output من `lspci \| grep exar` | مشكلة hardware/BIOS في الـ PCI enumeration |
| الـ cascaded device مش شغال | نص عدد الـ GPIOs المتوقع | تحقق من الـ PCI Device ID والـ `GENMASK(15,12)` |
| الـ regmap write بيرجع `-EIO` | `regmap: io error` | مشكلة في الـ PCIe link أو الـ power management |

---

#### 5. الـ Device Tree Debugging

> **ملاحظة مهمة:** الـ gpio-exar driver ده بيشتغل كـ **platform device** مش كـ DT device مباشرة — الـ UART driver (xr17v35x) هو اللي بيخلق الـ platform device وبيمرّر الـ properties. لكن لو استخدمت DT:

```bash
# شيك على الـ DT properties اللي اتعملت
cat /proc/device-tree/soc/gpio@*/exar,first-pin | xxd
cat /proc/device-tree/soc/gpio@*/ngpios | xxd

# أو باستخدام dtc
dtc -I fs /proc/device-tree 2>/dev/null | grep -A 5 "gpio-exar"

# تحقق إن الـ compatible string صح
cat /proc/device-tree/soc/gpio@*/compatible
# المفروض: "exar,xr17v35x-gpio" أو ما شابه

# شيك على الـ platform device properties في sysfs
cat /sys/bus/platform/devices/exar_gpio.*/properties/exar,first-pin 2>/dev/null
```

```bash
# لو بتستخدم ACPI بدل DT
cat /sys/bus/platform/devices/exar_gpio.0/firmware_node/path 2>/dev/null

# اقرا الـ properties اللي الـ UART driver مرّرها
ls /sys/bus/platform/devices/exar_gpio.0/
```

---

### Practical Commands

---

#### سيناريو 1: التحقق السريع إن الـ Driver اتحمّل

```bash
# 1. شيك إن الـ module موجود
lsmod | grep gpio_exar

# 2. شيك الـ probe message
dmesg | grep -i exar

# 3. شيك الـ GPIO chip
gpiodetect | grep exar

# مثال output ناجح:
# [    5.123] gpio_exar exar_gpio.0: GPIO chip exar_gpio0 registered
# gpiochip0 [exar_gpio0] (16 lines)
```

#### سيناريو 2: تتبع كل عمليات الـ Register Read/Write

```bash
# فعّل الـ tracing
echo 0 > /sys/kernel/tracing/tracing_on
echo "" > /sys/kernel/tracing/trace
echo 1 > /sys/kernel/tracing/events/regmap/regmap_reg_read/enable
echo 1 > /sys/kernel/tracing/events/regmap/regmap_reg_write/enable
echo 'name == "exar-gpio"' > /sys/kernel/tracing/events/regmap/regmap_reg_read/filter
echo 'name == "exar-gpio"' > /sys/kernel/tracing/events/regmap/regmap_reg_write/filter
echo 1 > /sys/kernel/tracing/tracing_on

# اعمل عملية
gpioset gpiochip0 0=1

# اقرا النتيجة
echo 0 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace

# مثال output:
# gpioset-4567 [001] .... 45.678: regmap_reg_write: exar-gpio reg=0090 val=01
# gpioset-4567 [001] .... 45.679: regmap_reg_read:  exar-gpio reg=0090 val=01
```

#### سيناريو 3: dump كامل لحالة الـ GPIO Pins

```bash
#!/bin/bash
# gpio-exar-dump.sh

CHIP=$(gpiodetect | grep exar | awk '{print $1}')
if [ -z "$CHIP" ]; then
    echo "No Exar GPIO chip found!"
    exit 1
fi

echo "=== Exar GPIO Dump ==="
echo "Chip: $CHIP"
echo ""
gpioinfo $CHIP

echo ""
echo "=== Register Values ==="
BASE=$(cat /proc/iomem | awk '/exar/{split($1,a,"-"); print "0x"a[1]; exit}')
echo "MMIO Base: $BASE"

for name_offset in "MPIOLVL_LO:0x90" "MPIOSEL_LO:0x93" "MPIOLVL_HI:0x96" "MPIOSEL_HI:0x99"; do
    name=${name_offset%%:*}
    offset=${name_offset##*:}
    val=$(devmem2 $((BASE + offset)) b 2>/dev/null | grep "Read at" | awk '{print $NF}')
    printf "%-15s (offset %s): %s\n" "$name" "$offset" "$val"
done
```

```bash
chmod +x gpio-exar-dump.sh
./gpio-exar-dump.sh

# مثال output:
# === Exar GPIO Dump ===
# Chip: gpiochip0
#
# gpiochip0 - 16 lines:
#     line   0:      unnamed       unused   input  active-high
#     line   1:      unnamed       unused  output  active-high
# ...
#
# === Register Values ===
# MMIO Base: 0xf0000000
# MPIOLVL_LO  (offset 0x90): 0x00000001
# MPIOSEL_LO  (offset 0x93): 0x000000FE
# MPIOLVL_HI  (offset 0x96): 0x00000000
# MPIOSEL_HI  (offset 0x99): 0x000000FF
```

#### سيناريو 4: اكتشاف مشكلة الـ Cascaded Device

```bash
# تحقق من الـ PCI Device ID
lspci -n | grep "13a8"
# مثال: 01:00.0 0200: 13a8:0358
# الـ 0358 → الـ nibble الأعلى 0x0 يعني مفيش cascade
# لو كان مثلاً 0x1358 → يعني cascaded مع 8 channels (0x8 = الـ nibble الأخير)

# شيك على ngpios بعد الـ probe
cat /sys/class/gpio/gpiochip*/ngpio
# لو cascade اتكشف → هيكون ضعف الـ ngpios الأصلي

# تحقق من الـ cascaded_offset في الـ dmesg
dmesg | grep -i "cascad"
```

#### سيناريو 5: اكتشاف الـ first_pin الغلط

```bash
# لو pin 0 مش بيشتغل صح، جرب تحسب العنوان يدوياً
# first_pin=0, offset=0:
# pin = 0 + (0 % 16) = 0
# cascaded = 0 / 16 = 0
# addr = (0/8) ? MPIOLVL_HI : MPIOLVL_LO = 0x90
# bit = 0 % 8 = 0

# قارن بالـ register dump
devmem2 $((BASE + 0x90)) b
# الـ bit 0 المفروض يتغير لما تعمل gpioset gpiochip0 0=1
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway — الـ GPIO مش بيتعرفش بعد boot

#### العنوان
الـ MPIO pins على XR17V358 مش بتظهرش في `/sys/class/gpio` بعد تشغيل gateway صناعي

#### السياق
شركة بتعمل industrial gateway بيستخدم PCI Express card فيها Exar XR17V358 (8-channel UART) عشان تتحكم في relay outputs و sense digital inputs من PLCs. الـ board بتشتغل على x86 industrial PC بـ Linux kernel مخصوص. المطلوب إن أول 4 pins يكونوا outputs للـ relays والـ 4 التانية inputs.

#### المشكلة
بعد load الـ driver، الأمر `ls /sys/class/gpio/` بيطلع `gpiochipX` لكن لما الـ application بتحاول تتحكم في الـ pins، مفيش حاجة بتتغير فعلاً. الـ relay مش بيشتغل.

#### التحليل
المشكلة في قراءة الـ Device Tree binding الغلط. الـ binding بيقول:

```
Required properties:
 - exar,first-pin: first exportable pins (0..15)
 - ngpios: number of exportable pins (1..16)
```

الفريق كتب في الـ platform data:
```c
/* خطأ: first-pin = 8 مع ngpios = 8 */
.first_pin = 8,
.ngpios   = 8,
```

في `gpio_exar_probe()`:
```c
ret = device_property_read_u32(dev, "exar,first-pin", &first_pin);
/* first_pin = 8 */
```

وفي `exar_offset_to_sel_addr()`:
```c
unsigned int pin = exar_gpio->first_pin + (offset % 16);
/* pin = 8 + 0 = 8  -->  pin/8 = 1  -->  MPIOSEL_HI */
```

الـ first_pin = 8 معناه إن حتى offset=0 بيروح على `MPIOSEL_HI` (register 0x99)، لكن الـ hardware مبرمج إن الـ relays متوصلين بـ pins 0–3 اللي في `MPIOSEL_LO` (0x93). الـ application بتكتب على registers فاضية.

#### الحل
صحح الـ platform data:
```c
/* صح: first-pin = 0 و ngpios = 8 */
static struct gpiod_lookup_table relay_gpio_table = {
    .dev_id = "gpio_exar",
    /* pins 0-3 output للـ relays، pins 4-7 input */
};
```

وفي الـ platform device properties:
```c
static const struct property_entry exar_props[] = {
    PROPERTY_ENTRY_U32("exar,first-pin", 0),  /* ابدأ من pin 0 */
    PROPERTY_ENTRY_U32("ngpios", 8),
    {}
};
```

تأكيد بعد التصحيح:
```bash
# شوف الـ chip
gpioinfo gpiochip0

# اختبر relay output (offset 0)
gpioset gpiochip0 0=1

# قرا input (offset 4)
gpioget gpiochip0 4
```

#### الدرس المستفاد
الـ `exar,first-pin` بيحدد الـ physical MPIO pin على الـ chip نفسه (0–15)، مش offset في الـ gpiochip. لو غلطت فيه، الـ address calculation في `exar_offset_to_sel_addr()` و`exar_offset_to_lvl_addr()` بتروح على registers غلط بالكامل.

---

### السيناريو 2: Android TV Box — الـ cascaded device مش بيشتغل صح

#### العنوان
الـ XR17V354 المتكاسكيد بيدي قيم غلط على GPIO بعد تفعيل cascading

#### السياق
منتج Android TV box بيستخدم XR17V354 (4-channel UART) مع secondary XR17V354 متكاسكيد عليه عشان يوصل لـ 8 UARTs. الـ secondary بيتحكم في LED status ووضع antenna switch. الـ PCI Device ID للـ primary هو `0x4354` (آخر 4 bits = `0x4` = 4 channels).

#### المشكلة
الـ LEDs على الـ secondary device مش بترد على أوامر الـ GPIO. الـ primary GPIOs شغالين كويس.

#### التحليل
في `gpio_exar_probe()`:
```c
if (pcidev->device & GENMASK(15, 12)) {
    ngpios += ngpios;  /* double the count for cascaded */
    exar_gpio->cascaded_offset = (pcidev->device & GENMASK(3, 0)) *
            EXAR_UART_CHANNEL_SIZE;
    /* cascaded_offset = 4 * 0x400 = 0x1000 */
}
```

الـ condition `pcidev->device & GENMASK(15, 12)` بتشيك على bits 15–12. الـ device ID `0x4354`:
- بالـ binary: `0100 0011 0101 0100`
- bits 15–12 = `0100` = 4 ≠ 0 → **condition صح، cascading يتفعل**

لكن الفريق بيحاول يتحكم في LEDs على الـ secondary عن طريق offsets 0–7 بدل 16–23:

```c
/* خطأ: بيكتب على primary بدل secondary */
gpiod_set_value(led_gpio, 1);  /* offset 2 بدل 18 */
```

في `exar_offset_to_lvl_addr()`:
```c
unsigned int cascaded = offset / 16;
/* offset=2  --> cascaded=0  --> بيروح على primary */
/* offset=18 --> cascaded=1  --> بيروح على secondary بـ cascaded_offset */
```

الـ secondary GPIOs بتبدأ من offset 16 مش من 0.

#### الحل
في الـ userspace أو kernel driver code:
```c
/* offsets للـ secondary (cascaded) device تبدأ من 16 */
#define SECONDARY_GPIO_BASE_OFFSET  16
#define LED_STATUS_GPIO             (SECONDARY_GPIO_BASE_OFFSET + 2)
#define ANTENNA_SWITCH_GPIO         (SECONDARY_GPIO_BASE_OFFSET + 0)

/* التحكم في LED على الـ secondary */
ret = gpiod_direction_output(led_gpiod, 0);
```

تحقق من الـ address calculation:
```bash
# شوف ngpios المتوقعة (لازم تكون 16 في حالة cascading)
cat /sys/class/gpio/gpiochip0/ngpio

# اختبر LED على secondary
gpioset gpiochip0 18=1
```

#### الدرس المستفاد
لما بيتفعل الـ cascading، الـ `ngpios` بتتضاعف والـ offsets للـ secondary بتبدأ من 16. دي مش convention خارجية — دي logic مبنية جوا `exar_offset_to_lvl_addr()` و`exar_offset_to_sel_addr()` اللي بيستخدموا `offset / 16` عشان يحددوا هل الـ pin على الـ primary ولا الـ secondary.

---

### السيناريو 3: IoT Sensor Hub — الـ direction مش بيتسيت صح

#### العنوان
الـ MPIO pins على XR17V352 بتقرا دايماً كـ input حتى بعد `direction_output`

#### السياق
IoT sensor hub بيستخدم XR17V352 (2-channel UART) على PCIe mini card. الـ MPIO pins بتتحكم في power enable لـ sensor modules تانية. الـ engineer بيعمل `direction_output` على pin معين لكن `get_direction` بترجع input.

#### المشكلة
بعد:
```c
gpiod_direction_output(power_enable_gpiod, 1);
```

الـ pin مش بيدي الـ power. وعند check:
```bash
gpioinfo gpiochip0
# بيطلع: "input" مش "output"
```

#### التحليل
في `exar_direction_output()`:
```c
static int exar_direction_output(struct gpio_chip *chip, unsigned int offset,
                                 int value)
{
    struct exar_gpio_chip *exar_gpio = gpiochip_get_data(chip);
    unsigned int addr = exar_offset_to_sel_addr(exar_gpio, offset);
    unsigned int bit = exar_offset_to_bit(exar_gpio, offset);
    int ret;

    ret = exar_set_value(chip, offset, value);  /* بيكتب LVL أولاً */
    if (ret)
        return ret;

    return regmap_clear_bits(exar_gpio->regmap, addr, BIT(bit));  /* بيعمل SEL=0 = output */
}
```

الـ XR17V35X datasheet بيقول إن الـ MPIOSEL bit = 1 يعني input، و= 0 يعني output. الـ `regmap_clear_bits` صح.

لكن المشكلة كانت في الـ regmap config:
```c
static const struct regmap_config exar_regmap_config = {
    .name      = "exar-gpio",
    .reg_bits  = 16,
    .val_bits  = 8,
    .io_port   = true,  /* <-- مهم جداً */
};
```

الـ `.io_port = true` بيخلي الـ regmap يستخدم I/O port access (inb/outb) بدل MMIO. الـ engineer غير الـ config في build محلي وشال الـ `io_port = true` ظناً إنه MMIO، فالـ `regmap_clear_bits` مش بيوصل للـ hardware registers صح.

تأكيد: عمل `ioremap` يدوياً وقرا الـ register:
```bash
# باستخدام devmem أو custom tool لقرا register 0x93
./read_pci_mmio 0x93
# لو بيرجع 0xFF = كل الـ bits = input = الكتابة مش اشتغلت
```

#### الحل
إرجاع الـ `io_port = true` في الـ regmap config:
```c
static const struct regmap_config exar_regmap_config = {
    .name     = "exar-gpio",
    .reg_bits = 16,
    .val_bits = 8,
    .io_port  = true,  /* لازم يكون true عشان PCI I/O space */
};
```

والتأكد إن الـ PCI BAR0 هو I/O space مش memory space:
```bash
lspci -v -s <bus:dev.fn> | grep "I/O ports"
```

#### الدرس المستفاد
الـ Exar XR17V35X بيستخدم PCI I/O space مش MMIO. الـ `.io_port = true` في `exar_regmap_config` مش اختياري — من غيره الـ regmap بيعمل `readb/writeb` على MMIO address بدل `inb/outb` على I/O port، والنتيجة كل الـ register access بيتعطل صامت.

---

### السيناريو 4: Automotive ECU — الـ first-pin خاطئ بيعطل UART

#### العنوان
تغيير `exar,first-pin` من 0 إلى 4 بيخرب الـ UART communication على XR17V358

#### السياق
automotive ECU بيستخدم XR17V358 في نظام diagnostics. الـ engineer المسؤول عن bring-up حاول يحرر الـ pins 0–3 للـ UART hardware flow control ويبدأ الـ GPIO من pin 4. غير الـ `exar,first-pin` إلى 4.

#### المشكلة
بعد التغيير:
- الـ GPIO بدأ يشتغل ظاهرياً
- لكن الـ UART channels فجأة بدأت تبعت data خاطئة وبعض الـ frames بتتفقد

#### التحليل
الـ XR17V35X MPIO pins مشتركة مع الـ UART hardware flow control signals. الـ pins 0–3 ممكن تكون CTS/RTS لأول channel أو channels. لما الـ engineer غير `first_pin = 4`:

في `exar_offset_to_sel_addr()`:
```c
unsigned int pin = exar_gpio->first_pin + (offset % 16);
/* first_pin=4, offset=0 --> pin=4 */
/* pin/8 = 0 --> MPIOSEL_LO (0x93) */
/* bit = pin%8 = 4 */
```

الـ `exar_direction_output` بيعمل:
```c
regmap_clear_bits(exar_gpio->regmap, 0x93, BIT(4));
/* بيعمل MPIOSEL bit 4 = 0 = output */
```

المشكلة إن pin 4 و5 هم RTS0 و CTS0 في الـ UART. تحويلهم لـ GPIO output بيكسر الـ hardware flow control للـ UART channel الأول، وده بيسبب data loss في الـ serial communication مع الـ ECU network.

التحقق:
```bash
# شوف حالة UART
stty -F /dev/ttyXR0 -a | grep "crtscts"
# لو crtscts مفعل والـ pin اتاخد كـ GPIO = مشكلة
```

#### الحل
راجع الـ XR17V358 datasheet عشان تحدد الـ MPIO pins الآمنة للاستخدام كـ GPIO:

```
MPIO 0-3:  قد تكون مخصصة لـ flow control (UART-dependent)
MPIO 8-15: عادةً free-use GPIOs
```

صح الـ platform properties:
```c
/* استخدم الـ pins 8-11 بدل 4-7 */
PROPERTY_ENTRY_U32("exar,first-pin", 8),
PROPERTY_ENTRY_U32("ngpios", 4),
```

وأوقف hardware flow control لو مش محتاجه:
```bash
stty -F /dev/ttyXR0 -crtscts
```

#### الدرس المستفاد
الـ `exar,first-pin` مش بس offset — ده بيحدد أي physical MPIO pins هتتحكم فيها الـ driver. بعض الـ MPIO pins مرتبطة بـ UART signals (CTS/RTS/DTR/DSR). لازم تراجع الـ datasheet قبل ما تختار `first-pin` وتتأكد إن الـ pins اللي بتستخدمها مش بتتعارض مع الـ UART hardware flow control.

---

### السيناريو 5: Custom Board Bring-up — الـ probe بيفشل بـ -ENOMEM

#### العنوان
`gpio_exar_probe` بيرجع `-ENOMEM` على custom PCIe board بيستخدم XR17V352

#### السياق
فريق bring-up لـ custom industrial board بيستخدم Exar XR17V352 كـ PCIe card. الـ UART driver شغال (الـ serial ports ظاهرين)، لكن الـ GPIO driver مش بيشتغل. الـ dmesg بيطلع:

```
gpio_exar: probe failed with error -12
```

الـ error -12 هو `-ENOMEM`.

#### التحليل
في `gpio_exar_probe()`:
```c
p = pcim_iomap_table(pcidev)[0];
if (!p)
    return -ENOMEM;  /* <-- هنا بيحصل الفشل */
```

الكود بيتوقع إن الـ UART driver (parent) عمل `pcim_iomap` على BAR0 قبل ما يسجل الـ platform device. لو الـ UART driver مش عامل ده، الـ `pcim_iomap_table(pcidev)[0]` بيرجع NULL.

التحقق:
```bash
# شوف لو BAR0 متmap
lspci -v -s <bus:dev.fn>
# لازم يكون:
# Memory at ... (32-bit, non-prefetchable) [size=...]
# أو
# I/O ports at ... [size=...]

# شوف kernel modules
lsmod | grep "xr17v35x\|exar"

# شوف dmesg لـ parent device
dmesg | grep -i "exar\|xr17"
```

الاحتمالات:
1. الـ UART driver للـ XR17V352 مش محمل أصلاً
2. الـ UART driver محمل لكن الـ pcim_iomap ماشتغلش
3. الـ platform device اتسجل قبل الـ UART driver يخلص الـ probe

#### الحل
تأكد إن الـ UART driver للـ Exar محمل قبل الـ GPIO driver:

```bash
# تحقق من الـ UART driver
modinfo xr17v35x 2>/dev/null || modinfo 8250_exar
lsmod | grep 8250_exar

# حمل الـ UART driver الصح أولاً
modprobe 8250_exar
modprobe gpio_exar

# تحقق من dmesg
dmesg | tail -20
```

لو المشكلة في الترتيب، أضف الـ dependency في `/etc/modules-load.d/`:
```
# /etc/modules-load.d/exar.conf
8250_exar
gpio_exar
```

أو في الـ Kconfig:
```kconfig
config GPIO_EXAR
    tristate "Exar XR17V35X GPIO support"
    depends on SERIAL_8250_EXAR  /* تأكيد إن الـ UART driver موجود */
```

تحقق نهائي:
```bash
# بعد التصحيح
ls /sys/class/gpio/ | grep gpio
gpiodetect
gpioinfo
```

#### الدرس المستفاد
الـ `gpio-exar` driver معتمد بشكل كامل على الـ UART parent driver عشان يعمل `pcim_iomap` على BAR0. ده مش بس dependency — الـ GPIO driver مفيهوش أي iomap خاص بيه، بيستخدم الـ memory اللي map-ها الـ parent. لو الـ parent مش شغال أو لسه محملش، الـ probe بيفشل بـ `-ENOMEM` من أول سطر فعلي في الكود. دايماً اتأكد إن `8250_exar` أو equivalent موجود وشغال قبل ما تـ debug الـ GPIO driver.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

| المقال | الوصف |
|--------|-------|
| [GPIO in the kernel: an introduction](https://lwn.net/Articles/532714/) | مقدمة شاملة لـ GPIO subsystem في الـ kernel — الـ legacy integer API والـ gpiolib |
| [GPIO in the kernel: future directions](https://lwn.net/Articles/533632/) | مناقشة مسار تطوير GPIO ومشاكل الـ integer-based API |
| [gpiolib: introduce descriptor-based GPIO interface](https://lwn.net/Articles/528226/) | أول patch series تقدّم الـ `gpio_desc`-based API |
| [New descriptor-based GPIO interface](https://lwn.net/Articles/565662/) | المراجعة الكاملة للـ descriptor API بعد الـ merge |
| [add gpio support to exar](https://lwn.net/Articles/710740/) | **الـ patch series الأصلية** اللي أضافت `gpio-exar.c` للـ kernel — تغطي الـ XR17V352/354/358 |
| [Driver for MaxLinear/Exar USB Serial adapters](https://lwn.net/Articles/750909/) | دعم الـ GPIO في الـ `xr_serial` driver للـ USB-to-serial |
| [Add support for MaxLinear/Exar USB to serial converters](https://lwn.net/Articles/838153/) | توسعة الدعم لأجهزة Exar الـ USB بعد الاستحواذ من MaxLinear |
| [gpio: Support for shared GPIO lines on boards](https://lwn.net/Articles/803629/) | الـ GPIO aggregator وإمكانية مشاركة الـ GPIO lines |

---

### توثيق الـ kernel الرسمي

**الـ`Documentation/` paths داخل الـ kernel source:**

```
Documentation/devicetree/bindings/gpio/gpio-exar.txt   ← ملف الـ binding الخاص بالـ Exar
Documentation/driver-api/gpio/intro.rst                ← مقدمة الـ GPIO subsystem
Documentation/driver-api/gpio/driver.rst               ← كتابة GPIO drivers
Documentation/driver-api/gpio/consumer.rst             ← استخدام GPIO من drivers أخرى
Documentation/driver-api/gpio/board.rst                ← GPIO mappings والـ device tree
Documentation/driver-api/gpio/legacy.rst               ← الـ legacy integer-based API
```

**روابط الـ kernel.org:**

- [GPIO Introduction](https://docs.kernel.org/driver-api/gpio/intro.html)
- [GPIO Descriptor Driver Interface](https://docs.kernel.org/driver-api/gpio/driver.html)
- [GPIO Descriptor Consumer Interface](https://docs.kernel.org/driver-api/gpio/consumer.html)
- [GPIO Mappings](https://docs.kernel.org/driver-api/gpio/board.html)

---

### الـ kernel commits المهمة

| الـ commit / الـ patch | الوصف |
|------------------------|-------|
| [gpio: exar: add gpio for exar cards — v7](https://www.uwsg.indiana.edu/hypermail/linux/kernel/1701.0/05107.html) | الـ patch الأخير قبل الـ merge الرسمي للـ `gpio-exar.c` |
| [serial: 8250: add gpio support to exar — patchwork](https://patchwork.kernel.org/patch/7889491/) | الـ patch الأولي على patchwork قبل الـ refactoring |
| [gpio-exar.c على torvalds/linux](https://github.com/torvalds/linux/blob/master/drivers/gpio/gpio-exar.c) | الملف الحالي مع كامل تاريخ الـ commits |
| [CONFIG_GPIO_EXAR — kernel config DB](https://cateee.net/lkddb/web-lkddb/GPIO_EXAR.html) | تفاصيل الـ Kconfig وتاريخ ظهور الـ driver في إصدارات الـ kernel |

---

### نقاشات الـ mailing list

| الرابط | الموضوع |
|--------|---------|
| [PATCH v10 review — Andy Shevchenko](https://lore.kernel.org/lkml/CAHp75VcSmfBrD+242ZZvm6v3D4D8cOOTHDZdmjLq-F_xQNciRw@mail.gmail.com/) | مراجعة Andy Shevchenko للـ patch النهائي للـ gpio-exar |
| [linux.kernel group — add gpio support to exar](https://groups.google.com/g/linux.kernel/c/ga8Z-7HWKQk) | النقاش الأصلي على linux-kernel list |
| [PATCH v3 — IOT2040 support with Exar XR17V352](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg1406515.html) | استخدام الـ MPIO pins مع الـ IOT2040 device |

---

### كتب مرجعية

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الفصل 1**: مقدمة الـ kernel modules وبنية الـ drivers
- **الفصل 14**: الـ Linux Device Model — الـ `platform_driver`، الـ `probe()`، الـ `remove()`
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 17**: Device Drivers — الـ character devices والـ misc devices
- **الفصل 14**: الـ Block Layer — مقارنة لفهم الـ character driver model المستخدم في GPIO

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 9**: الـ device drivers للأنظمة المدمجة
- **الفصل 15**: الـ device tree وكيف يُعرّف الـ hardware للـ kernel

---

### kernelnewbies.org

| الرابط | الفائدة |
|--------|---------|
| [LinuxChanges](https://kernelnewbies.org/LinuxChanges) | متابعة التغييرات في الـ GPIO subsystem عبر إصدارات الـ kernel |
| [Linux_6.2](https://kernelnewbies.org/Linux_6.2) | تغييرات الـ GPIO في الـ 6.2 |
| [Using GPIO from device tree — mailing list](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2015-August/014911.html) | سؤال عملي عن استخدام GPIO مع الـ device tree |
| [devm_gpiod_get usage](http://lists.kernelnewbies.org/pipermail/kernelnewbies/2016-August/016707.html) | شرح `devm_gpiod_get()` وكيف يختلف عن `gpio_request()` |

---

### elinux.org

| الرابط | الفائدة |
|--------|---------|
| [New GPIO Interface for User Space (PDF)](https://elinux.org/images/7/74/Elce2017_new_GPIO_interface.pdf) | شرح الـ character device API (`/dev/gpiochipN`) — ELCE 2017 |
| [Introduction to pin muxing and GPIO control (PDF)](https://elinux.org/images/a/a7/ELC-2021_Introduction_to_pin_muxing_and_GPIO_control_under_Linux.pdf) | الـ pin muxing وعلاقته بالـ GPIO — ELC 2021 |
| [EBC GPIO via mmap](https://elinux.org/EBC_GPIO_via_mmap) | الوصول للـ GPIO registers مباشرة عبر الـ memory mapping |
| [EBC GPIO Polling and Interrupts](https://elinux.org/EBC_gpio_Polling_and_Interrupts) | الـ GPIO interrupts والـ polling في الـ C code |

---

### مصطلحات للبحث

للعثور على معلومات إضافية، ابحث بالمصطلحات دي:

```
gpio-exar XR17V352 linux driver
MPIO Exar UART kernel gpio
CONFIG_GPIO_EXAR SERIAL_8250_EXAR
gpiolib descriptor interface kernel
devm_gpiochip_add_data linux kernel
platform_driver gpio pci auxiliary
linux gpio character device /dev/gpiochipN
gpio-aggregator linux kernel shared lines
```
## Phase 8: Writing simple module

### الفكرة

الـ `gpio_exar_probe` هي الدالة الأهم في الـ driver — بتتنفذ لما الـ platform device بتتسجل، وبتعمل setup للـ `gpio_chip` كامل. هنستخدم **kprobe** عشان نعترض دي ونطبع بيانات زي اسم الـ device واللي بيعمله.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * kprobe module: hook gpio_exar_probe to log Exar GPIO device registration
 */

#include <linux/kernel.h>      /* pr_info, printk helpers */
#include <linux/module.h>      /* MODULE_* macros, module_init/exit */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe */
#include <linux/platform_device.h> /* struct platform_device */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe hook on gpio_exar_probe to trace Exar GPIO registration");

/*
 * pre_handler: called just before gpio_exar_probe executes.
 * p   -> pointer to the kprobe struct itself (contains symbol name, addr, etc.)
 * regs-> CPU registers snapshot at probe point (pt_regs)
 *
 * On x86_64, first argument (pdev) lives in %rdi.
 * regs_get_kernel_argument() abstracts arch-specific register access.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Retrieve first argument: struct platform_device *pdev */
    struct platform_device *pdev =
        (struct platform_device *)regs_get_kernel_argument(regs, 0);

    if (!pdev)
        return 0;

    pr_info("[exar_kprobe] gpio_exar_probe called\n");
    pr_info("[exar_kprobe]   platform device name : %s\n",
            pdev->name ? pdev->name : "<null>");
    pr_info("[exar_kprobe]   platform device id   : %d\n", pdev->id);

    /* Log parent device name if available (the PCI device) */
    if (pdev->dev.parent && pdev->dev.parent->driver)
        pr_info("[exar_kprobe]   parent driver        : %s\n",
                pdev->dev.parent->driver->name);

    return 0; /* 0 = let the real function continue */
}

/*
 * post_handler: called right after gpio_exar_probe returns (before ret).
 * flags -> arch-specific flags, usually 0.
 *
 * نستخدمه نطبع إن الـ probe خلص — بيفيد في تتبع المدة والنتيجة.
 */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * On x86_64, return value is in %rax.
     * regs_return_value() reads it in a portable way.
     */
    long retval = regs_return_value(regs);

    pr_info("[exar_kprobe] gpio_exar_probe returned: %ld (%s)\n",
            retval, retval == 0 ? "success" : "error");
}

/* kprobe struct: target the symbol by name */
static struct kprobe kp = {
    .symbol_name    = "gpio_exar_probe",
    .pre_handler    = handler_pre,
    .post_handler   = handler_post,
};

static int __init exar_kprobe_init(void)
{
    int ret;

    /*
     * register_kprobe() بتحط breakpoint على عنوان الـ symbol.
     * لو الـ symbol مش موجود أو مش قابل للـ probe بترجع error.
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[exar_kprobe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[exar_kprobe] planted on gpio_exar_probe @ %px\n", kp.addr);
    return 0;
}

static void __exit exar_kprobe_exit(void)
{
    /*
     * unregister_kprobe() ضروري في الـ exit عشان يشيل الـ breakpoint
     * ويمنع الـ handler من الاتنفيذ بعد ما الـ module اتفكك من الـ kernel.
     * لو مش عملناه هيحصل kernel panic أول ما الـ probe بتتضرب.
     */
    unregister_kprobe(&kp);
    pr_info("[exar_kprobe] removed from gpio_exar_probe\n");
}

module_init(exar_kprobe_init);
module_exit(exar_kprobe_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `<linux/kprobes.h>` | بيجيب `struct kprobe`، `register_kprobe`، `regs_get_kernel_argument` |
| `<linux/platform_device.h>` | عشان نعمل cast للـ argument لـ `struct platform_device *` |
| `<linux/module.h>` | الـ macros الأساسية للـ module |

#### الـ `handler_pre`

**الـ `regs_get_kernel_argument(regs, 0)`** بتجيب الـ argument الأول للدالة (الـ `pdev`) من الـ registers بطريقة portable بدل ما ناخد `regs->di` مباشرة. الكود بعدين بيطبع اسم الـ platform device وعلاقته بالـ parent PCI device — معلومات مفيدة لتأكيد إن الـ Exar chip اتتعرف عليها صح.

#### الـ `handler_post`

**الـ `regs_return_value(regs)`** بتقرأ الـ return value بعد ما الدالة خلصت. بنستخدمها نعرف هل الـ `gpio_chip` اتسجل بنجاح ولا فيه error (مثلاً لو الـ `devm_gpiochip_add_data` فشلت).

#### الـ `struct kprobe kp`

الـ `.symbol_name` بيخلي الـ kernel يحل العنوان بنفسه وقت الـ registration — مش محتاجين نحدد عنوان ثابت. الـ `.pre_handler` و`.post_handler` بيتنفذوا قبل وبعد الدالة على التوالي.

#### الـ `module_init` / `module_exit`

- الـ `init`: بيسجل الـ kprobe ويطبع العنوان اللي اتحط عليه الـ breakpoint — مفيد للتأكد.
- الـ `exit`: لازم يشيل الـ probe قبل ما الـ module يتفكك، غير كده الـ kernel هيحاول يكمل executing عند عنوان مش موجود وهيعمل crash.

---

### كيفية البناء والتجربة

```bash
# Makefile بسيط
obj-m += exar_kprobe.o

# build
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# تحميل
sudo insmod exar_kprobe.ko

# مشاهدة الـ log
sudo dmesg | grep exar_kprobe

# فك التحميل
sudo rmmod exar_kprobe
```

> **ملاحظة**: لو الـ `gpio-exar` driver مش محمل، مش هتشوف الـ probe بتتضرب. جرب `modprobe gpio_exar` أو وصّل Exar device يدوياً عشان الـ platform device تتسجل وتشغّل الـ probe.
