## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الملف `include/linux/gpio/forwarder.h` جزء من **GPIO Aggregator** subsystem، اللي بيتبع الـ GPIO framework في Linux kernel. المسؤول عنه في MAINTAINERS هو **Geert Uytterhoeven**، والـ mailing list هو `linux-gpio@vger.kernel.org`.

---

### المشكلة اللي بيحلها — القصة الكاملة

تخيل إنك عندك جهاز embedded زي Raspberry Pi أو board صناعية، وعندك GPIO controller واحد فيه 32 pin. كل pin ممكن تقدر تقرأه أو تكتب عليه — تشغل LED، تحس بزرار، تتحكم في relay.

دلوقتي فيه مشكلتين حقيقيتين:

**المشكلة الأولى — الأمان:**
لينكس بيعرض الـ GPIO controller كـ `/dev/gpiochip0`، والوصول ليه all-or-nothing — يعني إما المستخدم يقدر يوصل لكل الـ 32 pin، أو محدش يقدر يوصل. مفيش طريقة تقول "المستخدم A يوصل بس للـ pin 5 و6" من غير ما تكتب كود جديد.

**المشكلة التانية — الـ Virtual Machines:**
لو عندك VM وعايز تديله تحكم في مجموعة pins معينة، محتاج تديله الـ controller كله — ده خطر أمني وهجوم surface أكبر.

**الحل — GPIO Aggregator:**
الـ Aggregator بيعمل إيه؟ بياخد مجموعة pins من controllers مختلفة أو من نفس الـ controller، ويلمّهم في **virtual gpio_chip جديدة**. الـ VM أو المستخدم بيشوف cchip جديدة فيها بس الـ pins اللي محتاجها، مش الكل.

مثال واقعي:
```bash
# خد pin 19 من e6052000.gpio وpins 20-21 من e6050000.gpio
# واعملهم virtual chip جديدة
$ echo 'e6052000.gpio 19 e6050000.gpio 20-21' > /sys/bus/platform/drivers/gpio-aggregator/new_device
```

النتيجة: `/dev/gpiochip5` جديدة بـ 3 pins بس.

---

### دور الـ `forwarder.h` تحديداً

هنا بييجي دور **GPIO Forwarder** — ده الـ layer الجوهرية اللي بتربط الـ virtual chip الجديدة بالـ pins الأصلية.

تخيل الموضوع كـ **وكيل أو proxy**:

```
Userspace / VM
     |
     | يكلم gpio_chip الوهمية
     v
[ gpiochip_fwd — الـ Forwarder ]
     |
     | يحوّل كل عملية للـ pin الأصلي
     v
[ gpio_chip الحقيقية في الـ hardware ]
```

الـ `forwarder.h` بيعرّف الـ API اللي بيخلّي الـ gpio-aggregator (وأي driver تاني) يعمل الآتي:

| الدالة | الوظيفة |
|--------|---------|
| `devm_gpiochip_fwd_alloc()` | ينشئ virtual forwarder chip بعدد معين من الـ GPIOs |
| `gpiochip_fwd_desc_add()` | يربط pin معين في الـ virtual chip بـ `gpio_desc` أصلي |
| `gpiochip_fwd_desc_free()` | يفك الربط لـ pin معين |
| `gpiochip_fwd_register()` | يسجّل الـ virtual chip في الـ kernel |
| `gpiochip_fwd_get_gpiochip()` | يجيب الـ `gpio_chip` الداخلية |
| `gpiochip_fwd_get_data()` | يجيب الـ private data المرتبطة |
| `gpiochip_fwd_gpio_*()` | مجموعة دوال لتنفيذ العمليات: get, set, direction, irq, config |

---

### مين بيستخدم الـ Forwarder؟

**1. gpio-aggregator** — الاستخدام الرئيسي:
```c
// drivers/gpio/gpio-aggregator.c
#include <linux/gpio/forwarder.h>

// ينشئ virtual chip
fwd = devm_gpiochip_fwd_alloc(dev, ngpios);

// يربط كل pin بمصدره الحقيقي
gpiochip_fwd_desc_add(fwd, real_gpio_desc, virtual_offset);

// يسجّل الـ chip
gpiochip_fwd_register(fwd, aggregator_data);
```

**2. pinctrl-upboard** — لـ UP Board (Intel-based SBCs):
```c
// drivers/pinctrl/pinctrl-upboard.c
// بيستخدم الـ forwarder لعمل GPIO chip مخصصة
// فوق الـ Intel SoC GPIOs مع منطق إضافي
fwd = devm_gpiochip_fwd_alloc(dev, pctrl->pctrl_data->ngpio);
chip = gpiochip_fwd_get_gpiochip(fwd);
gpiochip_fwd_register(fwd, pctrl);
```

---

### الفرق بين الـ Forwarder والـ Aggregator

| | GPIO Aggregator | GPIO Forwarder |
|--|--|--|
| **الدور** | logic عالي المستوى — يجمع pins | layer تقنية — proxy API |
| **الملف** | `drivers/gpio/gpio-aggregator.c` | `include/linux/gpio/forwarder.h` + implementation داخل aggregator.c |
| **من يستخدمه** | المستخدم عبر sysfs/configfs | الـ drivers اللي تحتاج virtual chip |

---

### ملفات مهمة يجب معرفتها

```
include/linux/gpio/
├── forwarder.h          ← الملف الحالي — API للـ virtual forwarding chip
├── driver.h             ← تعريف struct gpio_chip الأساسية
├── consumer.h           ← API لاستهلاك GPIOs (gpiod_get/set/...)
├── machine.h            ← gpiod_lookup_table لربط GPIOs بـ devices

drivers/gpio/
├── gpio-aggregator.c    ← المستخدم الرئيسي — ينفّذ الـ forwarder + aggregation logic
├── gpio-line-mux.c      ← GPIO line multiplexer (مشابه لكن لـ mux)
├── gpiolib.c            ← قلب الـ GPIO subsystem

drivers/pinctrl/
├── pinctrl-upboard.c    ← مستخدم ثاني للـ forwarder

Documentation/admin-guide/gpio/
├── gpio-aggregator.rst  ← شرح كامل للـ aggregator من ناحية المستخدم
```

---

### خلاصة

**الـ `forwarder.h`** هو عقد بين أي driver عايز يعمل **virtual gpio_chip** وبين الـ GPIO subsystem. بيوفّر API نظيفة تخلّيك تعمل chip وهمية، تربطها بـ pins حقيقية من أي مكان، وتسجّلها في الـ kernel — من غير ما تعيد اختراع العجلة في كل driver. الاستخدام الأساسي هو الـ **GPIO Aggregator** اللي بيحل مشكلة access control وعزل الـ GPIOs للـ VMs والمستخدمين.
## Phase 2: شرح الـ GPIO Forwarder Framework

### المشكلة اللي بيحلها الـ GPIO Forwarder

في الـ embedded Linux، في سيناريو شائع جداً: عندك GPIO lines موزعة على أكتر من GPIO controller (أو chip)، وعايز تجمعهم في virtual chip واحد يظهر للـ userspace أو لأي driver تاني كأنه device واحد متماسك.

المشكلة الأصلية بتيجي من فكرة الـ **GPIO Aggregator**:

- الـ userspace (أو driver تاني) محتاج يتحكم في مجموعة GPIO lines كـ logical unit
- الـ lines دي فعلياً موزعة على controllers مختلفة في الـ hardware
- كل controller ليه `gpio_chip` خاص بيه وـ API خاص
- محتاج حل kernel-level يعمل **indirection layer** يـ abstract التنوع ده

بدون الـ forwarder، كل driver محتاج يعمل manually الـ mapping بين الـ virtual offset والـ `gpio_desc` الحقيقي، وده يؤدي لكود مكرر وـ error-prone في كل driver.

---

### الحل اللي الـ Kernel اتخده

الـ kernel بيحل المشكلة بالـ **GPIO Forwarder**: طبقة وساطة بتسجل نفسها كـ `gpio_chip` كامل في الـ gpiolib، لكن كل عملية جوفاها بتـ delegate لـ `gpio_desc` array — كل `desc` منهم بيشاور على الـ GPIO الحقيقي في الـ hardware chip الأصلي.

الفكرة الجوهرية: الـ forwarder مش controller حقيقي، هو **proxy** بيتكلم باللغة العامة للـ gpiolib من ناحية، ومن ناحية تانية بيستخدم الـ consumer API (`gpiod_*`) عشان يتحدث مع الـ real GPIO lines.

---

### تشبيه من الواقع

تخيل شركة اتصالات بتشغّل call center. الـ clients بيتصلوا بـ رقم موحد (الـ virtual `gpio_chip`)، بس الـ agents الفعليين شغالين في فروع مختلفة في مدن مختلفة (الـ real GPIO controllers). الـ call center manager (الـ `gpiochip_fwd`) عنده قائمة (الـ `descs[]` array) فيها: "السؤال رقم 0 يروح لـ Cairo office، رقم 1 يروح لـ Alex office"، إلخ.

الـ mapping:

| عنصر التشبيه | الكونسبت الحقيقي في الـ kernel |
|---|---|
| رقم الموحد | الـ virtual `gpio_chip` المسجل في الـ gpiolib |
| قائمة التوجيه | `fwd->descs[]` array |
| كل فرع | `gpio_desc` يشاور على real GPIO في real controller |
| الـ call center manager | `struct gpiochip_fwd` نفسها |
| طلب العميل (get/set/direction) | الـ `gpio_chip` ops callbacks |
| تنفيذ الطلب في الفرع | `gpiod_get_value()` / `gpiod_set_value()` على الـ desc |
| حالة "الفرع مقفول للنوم" | `can_sleep` flag + mutex/spinlock decision |

---

### البيكتشر العام — أين يقع الـ Forwarder في الـ Kernel؟

```
╔══════════════════════════════════════════════════════════════════╗
║                        USERSPACE                                ║
║   /sys/class/gpio/   ←→  libgpiod  ←→  application             ║
╚══════════════════════════════════════════════════════════════════╝
                              │
                              ▼
╔══════════════════════════════════════════════════════════════════╗
║                    KERNEL: gpiolib core                         ║
║   gpiod_get / gpiod_set / gpiod_direction_*                     ║
╚══════════════════════════════════════════════════════════════════╝
                              │
              ┌───────────────┴────────────────┐
              ▼                                ▼
╔═════════════════════════╗      ╔══════════════════════════════╗
║   Real gpio_chip        ║      ║   Virtual gpio_chip          ║
║  (e.g. pl061, gpio-mmio)║      ║   (gpiochip_fwd)             ║
║                         ║      ║                              ║
║  .get  → hw register    ║      ║  .get  → gpiod_get_value()   ║
║  .set  → hw register    ║      ║  .set  → gpiod_set_value()   ║
╚═════════════════════════╝      ║  .to_irq → gpiod_to_irq()   ║
                                 ╚══════════════════════════════╝
                                              │
                          ┌───────────────────┼───────────────────┐
                          ▼                   ▼                   ▼
                   gpio_desc[0]        gpio_desc[1]        gpio_desc[N]
                   (chip A, pin 5)    (chip B, pin 12)   (chip A, pin 9)
                          │                   │                   │
                    ╔═══════════╗       ╔═══════════╗      ╔═══════════╗
                    ║  GPIO     ║       ║  GPIO     ║      ║  GPIO     ║
                    ║  Chip A   ║       ║  Chip B   ║      ║  Chip A   ║
                    ╚═══════════╝       ╚═══════════╝      ╚═══════════╝
```

**الـ consumers** (اللي بيستخدم الـ forwarder):
- الـ `gpio-aggregator` driver (الأساسي)
- أي driver بيبني virtual GPIO chip من lines متفرقة

**الـ providers** (اللي الـ forwarder بيـ delegate ليهم):
- أي `gpio_chip` real مسجل في الـ gpiolib (pl061، mmio، i2c expanders، إلخ)

---

### الـ Core Abstraction: `struct gpiochip_fwd`

ده القلب النابض للـ framework. كل الـ state المحتاجه لتشغيل الـ virtual chip موجودة هنا:

```c
struct gpiochip_fwd {
    struct gpio_chip chip;          /* الـ virtual chip نفسه — بيتسجل في gpiolib */
    struct gpio_desc **descs;       /* مصفوفة: offset → real gpio_desc */
    union {
        struct mutex mlock;         /* لما can_sleep = true (i2c expanders مثلاً) */
        spinlock_t slock;           /* لما can_sleep = false (MMIO GPIOs) */
    };
    struct gpiochip_fwd_timing *delay_timings; /* اختياري: ramp up/down delays */
    void *data;                     /* driver-private data */
    unsigned long *valid_mask;      /* bitmap: أي offsets فيهم desc حقيقي */
    unsigned long tmp[];            /* flexible array للـ bulk operations */
};
```

**ملاحظة مهمة على الـ locking**: الـ forwarder بيختار نوع الـ lock وقت الـ register مش وقت الـ alloc، لأن بس بعد ما تضاف كل الـ descs ممكن تعرف هل أي منهم يحتاج sleep أو لا:

```c
/* من gpiochip_fwd_register() */
if (!bitmap_full(fwd->valid_mask, chip->ngpio))
    chip->can_sleep = true;  /* فيه descs ناقصة → assume sleep */

if (chip->can_sleep)
    mutex_init(&fwd->mlock);
else
    spin_lock_init(&fwd->slock);
```

---

### علاقة الـ structs ببعض — رسم توضيحي

```
struct gpiochip_fwd
┌──────────────────────────────────────────────────────┐
│  chip (struct gpio_chip)                              │
│  ┌────────────────────────────────────────────────┐  │
│  │  .label = dev_name(dev)                        │  │
│  │  .ngpio = N                                    │  │
│  │  .request      = gpio_fwd_request              │  │
│  │  .get_direction = gpio_fwd_get_direction        │  │
│  │  .direction_input = gpio_fwd_direction_input   │  │
│  │  .direction_output = gpio_fwd_direction_output │  │
│  │  .get          = gpio_fwd_get                  │  │
│  │  .get_multiple = gpio_fwd_get_multiple_locked  │  │
│  │  .set          = gpio_fwd_set                  │  │
│  │  .set_multiple = gpio_fwd_set_multiple_locked  │  │
│  │  .set_config   = gpio_fwd_set_config           │  │
│  │  .to_irq       = gpio_fwd_to_irq               │  │
│  └────────────────────────────────────────────────┘  │
│                                                       │
│  descs (struct gpio_desc **)                          │
│  ┌──────┬──────┬──────┬──────┐                       │
│  │[0]   │[1]   │[2]   │[N-1] │  ← كل واحد → real pin│
│  └──┬───┴──┬───┴──┬───┴──────┘                       │
│     │      │      │                                   │
│  valid_mask (bitmap)                                  │
│  ┌──────────────────────────────┐                     │
│  │  bit 0=1, bit 1=1, bit 2=0  │  ← 2 تم إضافتهم   │
│  └──────────────────────────────┘                     │
│                                                       │
│  union { mlock | slock }  ← حسب can_sleep            │
│  data (void *)             ← driver-private           │
│  tmp[] (flexible array)    ← bulk ops buffer          │
└──────────────────────────────────────────────────────┘
         │           │           │
         ▼           ▼           ▼
    gpio_desc    gpio_desc    NULL (offset 2)
    chip_A/5     chip_B/12
```

---

### الـ Lifecycle — خطوات إنشاء الـ Forwarder

الـ API بيتبع نمط واضح وله ترتيب محدد:

```
1. devm_gpiochip_fwd_alloc(dev, ngpios)
       ↓
   allocate struct gpiochip_fwd
   allocate descs[] array
   allocate valid_mask bitmap
   wire up all gpio_chip ops callbacks
   chip->base = -1  (dynamic GPIO number assignment)

2. gpiochip_fwd_desc_add(fwd, desc, offset)  [× N]
       ↓
   test_and_set_bit(offset, valid_mask)
   if gpiod_cansleep(desc) → chip->can_sleep = true
   fwd->descs[offset] = desc

3. gpiochip_fwd_register(fwd, data)
       ↓
   check if bitmap_full → set can_sleep if not
   init mutex or spinlock based on can_sleep
   fwd->data = data
   devm_gpiochip_add_data(parent, chip, fwd)
       ↓
   الـ gpiolib بيسجل الـ chip وتبدأ الـ GPIO ops تشتغل
```

---

### تدفق العملية — مثال: `gpio_fwd_get()`

لما أي caller بيعمل `gpiod_get_value()` على GPIO من الـ virtual chip:

```
caller: gpiod_get_value(virtual_desc)
    │
    ▼
gpiolib core
    │  يشوف إن الـ desc ينتمي لـ fwd->chip
    ▼
gpio_fwd_get(chip, offset)
    │
    ├─ chip->can_sleep == false?
    │      ↓
    │   gpiod_get_value(fwd->descs[offset])
    │      ↓
    │   gpiolib core → real chip's .get() → HW register read
    │
    └─ chip->can_sleep == true?
           ↓
        gpiod_get_value_cansleep(fwd->descs[offset])
           ↓
        same path but allows sleeping (e.g., I2C read)
```

---

### الـ Bulk Operations والـ `tmp[]` Buffer

الـ `tmp[]` flexible array في نهاية الـ struct هو optimization ذكي. بدل ما تعمل `kmalloc` في كل bulk operation، الـ buffer موجود جاهز embedded في الـ struct نفسه:

```c
#define fwd_tmp_values(fwd)  (&(fwd)->tmp[0])
#define fwd_tmp_descs(fwd)   ((void *)&(fwd)->tmp[BITS_TO_LONGS((fwd)->chip.ngpio)])
#define fwd_tmp_size(ngpios) (BITS_TO_LONGS((ngpios)) + (ngpios))
```

الـ layout جوه `tmp[]`:

```
tmp[]
┌──────────────────────────────┬────────────────────────────────┐
│  values bitmap               │  descs[] pointers              │
│  (BITS_TO_LONGS(ngpio) longs)│  (ngpio pointers)              │
│  ← fwd_tmp_values()          │  ← fwd_tmp_descs()             │
└──────────────────────────────┴────────────────────────────────┘
```

لازم تـ lock قبل تستخدمه لأن الـ buffer shared بين كل الـ callers:

```c
static int gpio_fwd_get_multiple_locked(struct gpio_chip *chip, ...)
{
    struct gpiochip_fwd *fwd = gpiochip_get_data(chip);
    unsigned long flags;

    if (chip->can_sleep) {
        mutex_lock(&fwd->mlock);          /* sleeping lock for slow buses */
        error = gpio_fwd_get_multiple(fwd, mask, bits);
        mutex_unlock(&fwd->mlock);
    } else {
        spin_lock_irqsave(&fwd->slock, flags);   /* fast lock for MMIO */
        error = gpio_fwd_get_multiple(fwd, mask, bits);
        spin_unlock_irqrestore(&fwd->slock, flags);
    }
    ...
}
```

---

### الـ Delay Feature — Open-Drain RC Filter Support

الـ forwarder بيدعم اختيارياً الـ **GPIO delay**: وقت انتظار بعد تغيير state الـ line. ده مفيد جداً لـ open-drain outputs اللي بيستخدموا RC filter:

```
GPIO line (open-drain)
     │
     ├── R ──── VCC
     │
     └── C ──── GND

عند "ramp up" (الـ FET يقفل) → الـ C بيتشحن عبر R → وقت صعود بطيء
عند "ramp down" (الـ FET يفتح) → الـ C يتفرغ سريع → وقت نزول سريع
```

الـ `gpiochip_fwd_timing`:

```c
struct gpiochip_fwd_timing {
    u32 ramp_up_us;    /* microseconds to wait after driving high */
    u32 ramp_down_us;  /* microseconds to wait after driving low */
};
```

يتحدد من الـ device tree عبر `of_xlate` callback مخصص بيقرأ 3 args بدل 2:
```
/* device tree: <&forwarder_chip  line_num  ramp_up_us  ramp_down_us> */
gpios = <&aggregator 0 1000 500>;  /* line 0: 1ms up, 500µs down */
```

---

### ما الـ Forwarder يمتلكه vs ما يفوّضه للـ Drivers

| المسؤولية | الـ Forwarder يملكه | يفوّضه لـ |
|---|---|---|
| تسجيل الـ virtual gpio_chip | نعم (عبر `devm_gpiochip_add_data`) | — |
| الـ offset → desc mapping | نعم (`fwd->descs[]`) | — |
| الـ locking strategy | نعم (mutex vs spinlock) | — |
| الـ bulk ops buffering | نعم (`tmp[]`) | — |
| تنفيذ get/set فعلياً | لا | `gpiod_*` consumer API |
| إدارة الـ IRQ domain | لا | الـ real gpio_chip |
| الـ pinctrl integration | لا | الـ real gpio_chip |
| hardware initialization | لا | الـ real gpio_chip driver |
| الـ DT binding | جزئي (delay xlate فقط) | الـ aggregator driver |

---

### ربط بالـ Subsystems التانية

- **gpiolib** (الأساسي): هو الـ framework الأم — الـ forwarder بيسجل فيه كـ `gpio_chip` عادي، ولازم تفهم الـ `gpio_chip` / `gpio_desc` model قبل تفهم الـ forwarder
- **device model / devres**: الـ `devm_*` functions بتربط lifetime الـ forwarder بـ device، لما الـ device يتسحب كل الـ allocated resources بتتحرر تلقائياً
- **pinctrl subsystem**: الـ real GPIO chips غالباً بتتكلم مع pinctrl للـ muxing — الـ forwarder مش بيتدخل في ده خالص
- **IRQ domain**: الـ `gpio_fwd_to_irq()` بيـ delegate مباشرة لـ `gpiod_to_irq()` على الـ real desc — يعني الـ IRQ domain الحقيقي في الـ real controller مش في الـ forwarder
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### مقدمة سريعة عن الـ File

**الـ `include/linux/gpio/forwarder.h`** هو الـ public API الخاص بـ **GPIO Forwarder** subsystem — طبقة وسيطة بتسمح لـ driver يعمل كـ "وكيل" (proxy) بين الـ GPIO consumers والـ GPIO controllers الحقيقية. الفكرة الأساسية: عوضاً ما كل driver يتكلم مع `gpio_chip` مباشرة، بيخلق `gpiochip_fwd` يلم مجموعة `gpio_desc` من أماكن مختلفة ويعرضها كـ chip واحدة موحدة.

---

### 0. جدول الـ Flags والـ Constants المهمة

الـ header نفسه ما بيعرّف flags مباشرة — لكنه بيعتمد على constants من `gpio/driver.h` و`gpio/consumer.h`:

#### **اتجاهات الـ GPIO Line**

| Constant | قيمة | المعنى |
|---|---|---|
| `GPIO_LINE_DIRECTION_IN` | `1` | الـ pin مضبوط كـ input |
| `GPIO_LINE_DIRECTION_OUT` | `0` | الـ pin مضبوط كـ output |

#### **الـ `gpiod_flags` — enum لضبط الاتجاه عند الاستحواذ**

| القيمة | Bits الفعّالة | المعنى |
|---|---|---|
| `GPIOD_ASIS` | `0x00` | ماتغيرش أي حاجة |
| `GPIOD_IN` | `BIT(0)` | input mode |
| `GPIOD_OUT_LOW` | `BIT(0)\|BIT(1)` | output، قيمة ابتدائية = 0 |
| `GPIOD_OUT_HIGH` | `BIT(0)\|BIT(1)\|BIT(2)` | output، قيمة ابتدائية = 1 |
| `GPIOD_OUT_LOW_OPEN_DRAIN` | `OUT_LOW\|BIT(3)` | open-drain، low |
| `GPIOD_OUT_HIGH_OPEN_DRAIN` | `OUT_HIGH\|BIT(3)` | open-drain، high |

#### **الـ `GPIOD_FLAGS_BIT_*` — Bits جوا gpiod_flags**

| Bit Macro | Bit رقم | المعنى |
|---|---|---|
| `GPIOD_FLAGS_BIT_DIR_SET` | 0 | تم تحديد الاتجاه |
| `GPIOD_FLAGS_BIT_DIR_OUT` | 1 | الاتجاه = output |
| `GPIOD_FLAGS_BIT_DIR_VAL` | 2 | القيمة الابتدائية = high |
| `GPIOD_FLAGS_BIT_OPEN_DRAIN` | 3 | open-drain mode |

---

### 1. الـ Structs المهمة

#### **`struct gpiochip_fwd` — الـ Forwarder Object الأساسي**

الـ struct ده opaque في الـ header (مش معرّف فيه، بس declared). تعريفه الحقيقي في الـ implementation file (`drivers/gpio/gpio-forwarder.c`). هو الكائن اللي:

- بيحتضن مجموعة من `gpio_desc` pointers (كل واحد بيمثل GPIO line حقيقية من أي مصدر)
- بيعمل wrap ليهم تحت `gpio_chip` واحدة موحدة
- بيسمح بالتوجيه (forwarding) لكل العمليات من consumer للـ descriptor الحقيقي

**الحقول المتوقعة داخلياً (من تحليل الـ API):**

| حقل (مُستنتج) | النوع | الدور |
|---|---|---|
| `chip` | `struct gpio_chip` | الـ chip المسجّلة في gpiolib |
| `descs[]` | `struct gpio_desc *[]` | مصفوفة بـ descriptors الـ GPIOs الفعلية |
| `ngpios` | `unsigned int` | عدد الـ GPIO lines |
| `data` | `void *` | بيانات خاصة بالـ driver |
| lock | `spinlock_t` أو `mutex` | حماية الوصول المتزامن |

---

#### **`struct gpio_desc` — الـ GPIO Line Descriptor**

Opaque struct في الـ header. كل `gpio_desc` يمثل خط GPIO واحد في النظام. يتضمن داخلياً:

| حقل (داخلي) | النوع | الدور |
|---|---|---|
| `gdev` | `struct gpio_device *` | الـ device الأم |
| `flags` | `unsigned long` | حالة الـ line (direction، active_low، إلخ) |
| `label` | `const char *` | اسم المستخدم اللي طلبه |
| `name` | `const char *` | الاسم الثابت |

---

#### **`struct gpio_chip` — الـ GPIO Controller الافتراضي**

ده الـ struct المركزي في GPIO subsystem. الـ forwarder بيبني `gpio_chip` كاملة ويملأ callback pointers بتاعتها بـ forwarding functions:

| حقل | النوع | الدور |
|---|---|---|
| `label` | `const char *` | اسم الـ controller |
| `gpiodev` | `struct gpio_device *` | الـ internal state holder |
| `parent` | `struct device *` | الـ device الأم |
| `ngpio` | `u16` | عدد الـ lines |
| `base` | `int` | أول رقم GPIO (أو -1 لـ dynamic) |
| `can_sleep` | `bool` | هل العمليات ممكن تنام؟ |
| `request` | `fn ptr` | callback عند طلب line |
| `get_direction` | `fn ptr` | اتجاه الـ line |
| `direction_input` | `fn ptr` | ضبط كـ input |
| `direction_output` | `fn ptr` | ضبط كـ output |
| `get` | `fn ptr` | قراءة قيمة line |
| `get_multiple` | `fn ptr` | قراءة متعددة بـ bitmask |
| `set` | `fn ptr` | كتابة قيمة line |
| `set_multiple` | `fn ptr` | كتابة متعددة بـ bitmask |
| `set_config` | `fn ptr` | ضبط إعدادات (pull، drive، إلخ) |
| `to_irq` | `fn ptr` | تحويل GPIO لـ IRQ number |
| `irq` | `struct gpio_irq_chip` | إدارة الـ IRQs (لو CONFIG_GPIOLIB_IRQCHIP) |

---

#### **`struct gpio_irq_chip` — الـ IRQ Controller جوا الـ GPIO**

بيتم تضمينه داخل `gpio_chip` وبيدير الـ interrupts:

| حقل | النوع | الدور |
|---|---|---|
| `chip` | `struct irq_chip *` | الـ IRQ chip implementation |
| `domain` | `struct irq_domain *` | ترجمة hwirq ↔ Linux IRQ |
| `handler` | `irq_flow_handler_t` | الـ IRQ handler |
| `default_type` | `unsigned int` | نوع الـ trigger الافتراضي |
| `threaded` | `bool` | هل الـ handling threaded؟ |
| `valid_mask` | `unsigned long *` | bitmask للـ GPIOs الصالحة كـ IRQ |
| `num_parents` | `unsigned int` | عدد الـ parent interrupts |
| `parents` | `unsigned int *` | مصفوفة الـ parent IRQ numbers |
| `initialized` | `bool` | علامة التهيئة |

---

#### **`struct gpio_descs` — مجموعة Descriptors**

بتُرجع من `gpiod_get_array()` وبتلم أكتر من descriptor في object واحد:

| حقل | النوع | الدور |
|---|---|---|
| `info` | `struct gpio_array *` | بيانات داخلية للـ bulk ops |
| `ndescs` | `unsigned int` | عدد الـ descriptors |
| `desc[]` | `struct gpio_desc *[]` | flexible array من الـ descriptors |

---

### 2. مخطط علاقات الـ Structs

```
Consumer Driver
     │
     │  devm_gpiochip_fwd_alloc()
     ▼
┌─────────────────────────────────────────┐
│           struct gpiochip_fwd           │
│  ┌──────────────────────────────────┐   │
│  │       struct gpio_chip           │   │
│  │  (forwarding callbacks inside)   │◄──┼── gpiochip_fwd_get_gpiochip()
│  └──────────────────────────────────┘   │
│                                         │
│  void *data ◄────────────────────────── ┼── gpiochip_fwd_get_data()
│                                         │
│  descs[0] ──► struct gpio_desc ──► real gpio_chip A
│  descs[1] ──► struct gpio_desc ──► real gpio_chip A
│  descs[2] ──► struct gpio_desc ──► real gpio_chip B
│  descs[N] ──► struct gpio_desc ──► real gpio_chip C
└─────────────────────────────────────────┘
         │
         │ gpiochip_fwd_register()
         ▼
    gpiolib core
    (gpiochip_add_data)
         │
         ▼
  struct gpio_device
  (kernel-internal state)
```

```
struct gpio_chip
     │
     ├── struct gpio_device *gpiodev  ──► kernel internal (opaque)
     ├── struct device *parent        ──► platform device / MFD child
     ├── struct gpio_irq_chip irq
     │        │
     │        ├── struct irq_chip *chip    ──► IRQ subsystem
     │        └── struct irq_domain *domain ──► hwirq↔virq mapping
     └── callbacks (request/get/set/...)
```

---

### 3. دورة حياة الـ Forwarder (Lifecycle)

```
┌─────────────────────────────────────────────────────────┐
│                    CREATION PHASE                       │
│                                                         │
│  devm_gpiochip_fwd_alloc(dev, ngpios)                   │
│    ├── kzalloc(sizeof(gpiochip_fwd) + ngpios * ptr)     │
│    ├── init internal gpio_chip struct                   │
│    ├── set ngpio = ngpios                               │
│    └── register devm cleanup callback                   │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                  POPULATION PHASE                       │
│                                                         │
│  gpiochip_fwd_desc_add(fwd, desc, offset)  [× ngpios]  │
│    ├── validate offset < ngpios                         │
│    ├── fwd->descs[offset] = desc                        │
│    └── mark slot as occupied                            │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│               REGISTRATION PHASE                        │
│                                                         │
│  gpiochip_fwd_register(fwd, data)                       │
│    ├── store data pointer                               │
│    ├── wire up forwarding callbacks to gpio_chip        │
│    └── gpiochip_add_data(&fwd->chip, fwd)               │
│          └── gpiolib registers chip in global list      │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                   USAGE PHASE                           │
│                                                         │
│  Consumer calls gpiochip_fwd_gpio_get(fwd, offset)      │
│    └── internally: gpiod_get_raw_value(descs[offset])   │
│                                                         │
│  Consumer calls gpiochip_fwd_gpio_set(fwd, offset, val) │
│    └── internally: gpiod_set_raw_value(descs[offset])   │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                  TEARDOWN PHASE                         │
│                                                         │
│  gpiochip_fwd_desc_free(fwd, offset)   [optional]       │
│    └── release specific slot descriptor                 │
│                                                         │
│  devm cleanup (automatic on device removal)             │
│    ├── gpiochip_remove(&fwd->chip)                      │
│    └── kfree(fwd)                                       │
└─────────────────────────────────────────────────────────┘
```

---

### 4. Call Flow Diagrams

#### **مسار قراءة قيمة GPIO**

```
consumer calls gpiochip_fwd_gpio_get(fwd, offset)
  │
  ├── validate: offset < fwd->chip.ngpio
  ├── desc = fwd->descs[offset]
  │
  └── gpiod_get_raw_value(desc)
        │
        └── gpio_chip->get(gc, hwoffset)
              │
              └── real hardware read (I2C/SPI/MMIO register)
```

#### **مسار ضبط اتجاه GPIO**

```
consumer calls gpiochip_fwd_gpio_direction_output(fwd, offset, value)
  │
  ├── desc = fwd->descs[offset]
  │
  └── gpiod_direction_output_raw(desc, value)
        │
        └── gpio_chip->direction_output(gc, hwoffset, value)
              │
              └── real hardware configure direction + initial value
```

#### **مسار bulk set_multiple**

```
consumer calls gpiochip_fwd_gpio_set_multiple(fwd, mask, bits)
  │
  ├── iterate bits in mask
  ├── for each set bit i:
  │     desc = fwd->descs[i]
  │     gpiod_set_raw_value(desc, bit_value(bits, i))
  │
  └── [each call may hit different real gpio_chip backends]
```

#### **مسار تحويل GPIO → IRQ**

```
consumer calls gpiochip_fwd_gpio_to_irq(fwd, offset)
  │
  ├── desc = fwd->descs[offset]
  │
  └── gpiod_to_irq(desc)
        │
        └── gpio_chip->to_irq(gc, hwoffset)
              │
              └── irq_find_mapping(domain, hwoffset)
                    │
                    └── returns Linux virq number
```

#### **مسار تسجيل الـ Forwarder كـ gpio_chip**

```
driver calls gpiochip_fwd_register(fwd, priv_data)
  │
  ├── fwd->data = priv_data
  ├── wire fwd->chip.get        = gpiochip_fwd_get_cb
  ├── wire fwd->chip.set        = gpiochip_fwd_set_cb
  ├── wire fwd->chip.direction_input  = ...
  ├── wire fwd->chip.direction_output = ...
  ├── wire fwd->chip.to_irq     = ...
  │
  └── gpiochip_add_data(&fwd->chip, fwd)
        │
        ├── gpiolib validates chip fields
        ├── assigns base GPIO number (dynamic if base < 0)
        ├── creates sysfs entries
        └── makes chip available to consumers
```

---

### 5. استراتيجية الـ Locking

#### **طبقات الـ Locking في GPIO Forwarder**

```
┌──────────────────────────────────────────────────────────────┐
│  Level 1 — gpiolib core lock                                 │
│  يحميها: gpio_device->lock (spinlock)                        │
│  بيحمي: الوصول لـ gpio_device internal state                 │
│  owned by: gpiolib، مش الـ forwarder مباشرة                 │
└──────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  Level 2 — gpio_chip backend lock                            │
│  يحميها: real gpio_chip driver (I2C mutex / MMIO spinlock)   │
│  بيحمي: الوصول للهاردوير الفعلي                             │
│  owned by: underlying GPIO driver                            │
└──────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  Level 3 — IRQ lock (لو GPIO مربوط بـ interrupt)             │
│  يحميها: irq_chip lock_key (lockdep class)                   │
│  بيحمي: الـ IRQ domain و enable/disable state                │
│  owned by: gpio_irq_chip                                     │
└──────────────────────────────────────────────────────────────┘
```

#### **قاعدة الـ `can_sleep`**

**الـ `can_sleep` flag** في `gpio_chip` هو مفتاح حاسم في موضوع الـ locking:

| الحالة | `can_sleep` | Lock Context المسموح |
|---|---|---|
| GPIO على MMIO مباشرة | `false` | atomic context مسموح (spinlock) |
| GPIO على I2C/SPI expander | `true` | process context فقط (mutex) |
| GPIO forwarder | يرث من الـ backing descs | لو أي desc بـ `can_sleep` فالكل يتعامل كـ sleepable |

#### **ترتيب الـ Locking (Lock Ordering)**

لازم دايماً تمشي في الاتجاه ده عشان تتجنب deadlock:

```
1. gpiolib core lock (gpio_device->lock)
   └──► 2. real gpio_chip driver lock (I2C adapter lock أو MMIO spinlock)
              └──► 3. IRQ chip lock (irq_desc->lock)
```

**ملاحظة مهمة:** الـ `gpiochip_fwd_gpio_set_multiple` و`gpiochip_fwd_gpio_get_multiple` بيعملوا loop على descriptors بتاعة chips مختلفة. ده ممكن يسبب مشكلة لو كل chip بتاخد lock — لكن gpiolib بيتجنب ده بالترتيب الداخلي وعدم الـ nesting بين chips.

#### **الـ IRQ Locking بالتفصيل**

```c
/* لما consumer يعمل request_irq على GPIO */
gpiochip_fwd_gpio_to_irq(fwd, offset)
  └── gpiod_to_irq(desc)
        └── gc->to_irq(gc, hwoffset)
              └── irq_find_mapping(gc->irq.domain, hwoffset)
                    /* lock: irq_domain_mutex (read) */
```

الـ `lock_key` و`request_key` في `gpio_irq_chip` بيوفروا **lockdep annotations** للـ kernel lock validator — بيخلي kernel يتحقق إن lock ordering صح وقت runtime.
## Phase 4: شرح الـ Functions

### ملخص سريع — Cheatsheet

#### Lifecycle & Registration

| Function | الغرض | Export Namespace |
|---|---|---|
| `devm_gpiochip_fwd_alloc()` | تخصيص وتهيئة الـ forwarder | `GPIO_FORWARDER` |
| `gpiochip_fwd_desc_add()` | إضافة `gpio_desc` لـ slot محدد | `GPIO_FORWARDER` |
| `gpiochip_fwd_desc_free()` | تحرير `gpio_desc` من الـ forwarder | `GPIO_FORWARDER` |
| `gpiochip_fwd_register()` | تسجيل الـ `gpio_chip` مع gpiolib | `GPIO_FORWARDER` |

#### Accessors

| Function | الغرض |
|---|---|
| `gpiochip_fwd_get_gpiochip()` | استرجاع الـ `gpio_chip` المدمج |
| `gpiochip_fwd_get_data()` | استرجاع الـ driver-private data |

#### GPIO Operations (Runtime)

| Function | الغرض |
|---|---|
| `gpiochip_fwd_gpio_request()` | التحقق من صلاحية الـ line |
| `gpiochip_fwd_gpio_get_direction()` | قراءة اتجاه الـ line |
| `gpiochip_fwd_gpio_direction_input()` | تعيين الـ line كـ input |
| `gpiochip_fwd_gpio_direction_output()` | تعيين الـ line كـ output مع قيمة |
| `gpiochip_fwd_gpio_get()` | قراءة قيمة line واحد |
| `gpiochip_fwd_gpio_get_multiple()` | قراءة قيم عدة lines في آن |
| `gpiochip_fwd_gpio_set()` | كتابة قيمة line واحد |
| `gpiochip_fwd_gpio_set_multiple()` | كتابة قيم عدة lines في آن |
| `gpiochip_fwd_gpio_set_config()` | ضبط pinconf لـ line محدد |
| `gpiochip_fwd_gpio_to_irq()` | تحويل GPIO line إلى Linux IRQ number |

---

### التصميم العام — الـ `gpiochip_fwd` كـ Virtual GPIO Chip

**الـ GPIO Forwarder** هو طبقة تجريد بتخلّي driver يعمل فوقها يتكلم مع `gpio_desc` descriptors جاية من chips تانية، من غير ما يحتاج يعرف الـ hardware details. الـ `struct gpiochip_fwd` بتحتوي على:

```c
struct gpiochip_fwd {
    struct gpio_chip chip;           /* the virtual gpio_chip exposed to gpiolib */
    struct gpio_desc **descs;        /* array: virtual offset -> real gpio_desc */
    union {
        struct mutex mlock;          /* used when chip->can_sleep == true */
        spinlock_t slock;            /* used when chip->can_sleep == false */
    };
    struct gpiochip_fwd_timing *delay_timings; /* optional ramp delays per line */
    void *data;                      /* driver-private data */
    unsigned long *valid_mask;       /* bitmap: which offsets have valid descs */
    unsigned long tmp[];             /* flex array: tmp values + tmp descs for multi-ops */
};
```

**الـ `valid_mask`** هي bitmap بتتحكم في الـ line validity — أي line مش موجود فيها بيرجع `-ENODEV`. الـ `tmp[]` flex array بتاخد حيّز = `BITS_TO_LONGS(ngpios) + ngpios` words — النص الأول لـ values bitmap، النص التاني لـ `gpio_desc*` array — وده بيخلي الـ `get_multiple`/`set_multiple` thread-safe بدون heap allocation.

```
#define fwd_tmp_values(fwd)  (&(fwd)->tmp[0])
#define fwd_tmp_descs(fwd)   ((void *)&(fwd)->tmp[BITS_TO_LONGS(fwd->chip.ngpio)])
#define fwd_tmp_size(ngpios) (BITS_TO_LONGS(ngpios) + (ngpios))
```

---

### Group 1: Allocation & Registration

هذه المجموعة بتبني الـ `gpiochip_fwd` object من الصفر. الترتيب الطبيعي:
`devm_gpiochip_fwd_alloc` → `gpiochip_fwd_desc_add` (لكل line) → `gpiochip_fwd_register`.

---

#### `devm_gpiochip_fwd_alloc()`

```c
struct gpiochip_fwd *devm_gpiochip_fwd_alloc(struct device *dev,
                                              unsigned int ngpios);
```

بتخصص الـ `gpiochip_fwd` struct كاملة بالـ devm allocator، يعني الـ cleanup هيتحصل أوتوماتيك عند `device_unregister`. بتبني جوّاها `fwd->descs[]` array و`fwd->valid_mask` bitmap وبتحدد جميع callbacks الـ `gpio_chip` الداخلي.

**Parameters:**
- `dev` — الـ parent device، بيُستخدم في `devm_kzalloc` وكـ `chip->parent`
- `ngpios` — عدد الـ GPIO lines في الـ forwarder

**Return Value:**
pointer لـ `gpiochip_fwd` أو `ERR_PTR(-ENOMEM)` لو فشل الـ allocation.

**Key Details:**
- بيخصص الـ struct بـ `struct_size(fwd, tmp, fwd_tmp_size(ngpios))` لحجز مكان للـ flexible array `tmp[]` المستخدمة في عمليات `set/get_multiple`.
- بيضبط `chip->base = -1` يعني gpiolib هيختار الـ base number تلقائياً (dynamic numbering).
- الـ `chip->can_sleep` متضبطش هنا — بيتضبط في `gpiochip_fwd_desc_add` أو `gpiochip_fwd_register`.
- الـ `valid_mask` بيبدأ كلو أصفار (كل الـ lines invalid) وبيتعبي تدريجياً عبر `gpiochip_fwd_desc_add`.
- الـ chip label بيتضبط على `dev_name(dev)` تلقائياً.

**Pseudocode:**
```
fwd = devm_kzalloc(dev, struct_size(fwd, tmp, fwd_tmp_size(ngpios)))
fwd->descs        = devm_kcalloc(dev, ngpios, sizeof(*descs))
fwd->valid_mask   = devm_bitmap_zalloc(dev, ngpios)

chip->label       = dev_name(dev)
chip->parent      = dev
chip->owner       = THIS_MODULE
chip->request     = gpio_fwd_request
chip->get_direction  = gpio_fwd_get_direction
chip->direction_input  = gpio_fwd_direction_input
chip->direction_output = gpio_fwd_direction_output
chip->get         = gpio_fwd_get
chip->get_multiple = gpio_fwd_get_multiple_locked
chip->set         = gpio_fwd_set
chip->set_multiple = gpio_fwd_set_multiple_locked
chip->set_config  = gpio_fwd_set_config
chip->to_irq      = gpio_fwd_to_irq
chip->base        = -1
chip->ngpio       = ngpios
return fwd
```

**من بيناديها:** `gpiochip_fwd_create()` داخل `gpio_aggregator_probe`، وأي driver خارجي بيستخدم الـ GPIO Forwarder API.

---

#### `gpiochip_fwd_desc_add()`

```c
int gpiochip_fwd_desc_add(struct gpiochip_fwd *fwd,
                          struct gpio_desc *desc,
                          unsigned int offset);
```

بتربط `gpio_desc` حقيقية من chip تانية بـ slot معيّنة في الـ forwarder. بتستخدم `test_and_set_bit()` على الـ `valid_mask` لضمان إن الـ slot مش محجوزة قبل كده. لو الـ `desc` يحتاج sleep (مثلاً I²C GPIO expander)، بيضبط `chip->can_sleep = true` للـ chip كلها.

**Parameters:**
- `fwd` — الـ forwarder المخصص مسبقاً
- `desc` — الـ `gpio_desc` المستهدفة من أي chip تانية (مش owned بالـ forwarder)
- `offset` — رقم الـ slot في الـ forwarder (0 حتى ngpios-1)

**Return Value:**
- `0` — نجح
- `-EINVAL` — لو الـ offset خارج النطاق
- `-EEXIST` — لو الـ slot ده مضاف قبل كده

**Key Details:**
- بيستخدم `test_and_set_bit` عشان يتفادى race condition لو اتنادت من contexts متعددة.
- مبيعملش locking داخلي — الـ caller مسؤول عن التزامن قبل `gpiochip_fwd_register`.
- لو أي line واحدة `can_sleep`، الـ chip كلها بتبقى `can_sleep = true` ولا بيرجع لـ false — قرار conservative لضمان صحة الـ locking في `set_multiple`/`get_multiple`.
- بيطبع `dev_dbg` بـ offset ورقم الـ GPIO وIRQ بعد الـ add.

**من بيناديها:** `gpiochip_fwd_create()` في loop على كل descs، وأي driver بيبني الـ forwarder يدوياً.

---

#### `gpiochip_fwd_desc_free()`

```c
void gpiochip_fwd_desc_free(struct gpiochip_fwd *fwd, unsigned int offset);
```

بتشيل الـ `gpio_desc` من الـ slot وبتحرر الـ reference عليها بـ `gpiod_put()` لو الـ slot كانت valid. بتستخدم `test_and_clear_bit()` atomic لتجنب الـ double-free.

**Parameters:**
- `fwd` — الـ forwarder
- `offset` — رقم الـ slot المراد تحريرها

**Return Value:** void

**Key Details:**
- `test_and_clear_bit` بيضمن إن `gpiod_put` بتتنادى بس لو الـ bit كان set — آمن للـ double-call.
- مبتضبطش `fwd->descs[offset] = NULL` — الـ `valid_mask` هو المرجع الوحيد للصلاحية.
- الـ cleanup الـ devm ماشي على مستوى الـ device كله، لكن الـ function دي للتحرير الـ selective أثناء الـ runtime.

---

#### `gpiochip_fwd_register()`

```c
int gpiochip_fwd_register(struct gpiochip_fwd *fwd, void *data);
```

بتختم عملية بناء الـ forwarder وبتسجّله في gpiolib بـ `devm_gpiochip_add_data`. قبل التسجيل بتقرر نوع الـ lock المناسب: `mutex` لو الـ chip `can_sleep`، وإلا `spinlock`.

**Parameters:**
- `fwd` — الـ forwarder الجاهز (بعد `alloc` + `desc_add`)
- `data` — driver-private data بيتخزن في `fwd->data` وقابل للاسترداد بـ `gpiochip_fwd_get_data()`

**Return Value:**
- `0` — نجح
- errno سالب من `devm_gpiochip_add_data` لو فشل التسجيل

**Key Details:**
- لو `valid_mask` مش ممتلئ (مش كل الـ lines اتضافت)، بيضبط `can_sleep = true` قسراً — عشان الـ lines اللي هتتضاف بعد التسجيل ممكن تكون على slow bus.
- بعد `devm_gpiochip_add_data`، الـ chip بتبقى visible لـ userspace ولكل subsystems تانية — لازم يبقى الـ setup كامل قبل النداء ده.
- الـ `chip->get_direction` بتتنادى أثناء التسجيل نفسه من gpiolib وبترجع `-ENODEV` للـ slots الفاضية.

**Pseudocode:**
```
if not bitmap_full(valid_mask, ngpios):
    chip->can_sleep = true

if chip->can_sleep:
    mutex_init(&fwd->mlock)
else:
    spin_lock_init(&fwd->slock)

fwd->data = data
return devm_gpiochip_add_data(chip->parent, chip, fwd)
```

---

### Group 2: Accessors

---

#### `gpiochip_fwd_get_gpiochip()`

```c
struct gpio_chip *gpiochip_fwd_get_gpiochip(struct gpiochip_fwd *fwd);
```

بترجع pointer للـ `gpio_chip` المدمج جوّا الـ `gpiochip_fwd`. مفيدة للـ drivers اللي محتاجة تمرر `gpio_chip*` لـ gpiolib APIs تانية.

**Parameters:** `fwd` — الـ GPIO forwarder

**Return Value:** `&fwd->chip` — pointer مباشر، مش copy، مينفعش يكون NULL.

**Key Details:**
- الـ `gpio_chip` embedded مش مخصص بشكل منفصل — جزء من الـ `gpiochip_fwd` struct.
- الـ function نفسها بتستخدم داخلياً في كل الـ `gpiochip_fwd_gpio_*` wrappers.
- الـ chip بيبقى valid طول عمر الـ devm device.

---

#### `gpiochip_fwd_get_data()`

```c
void *gpiochip_fwd_get_data(struct gpiochip_fwd *fwd);
```

بترجع الـ driver-private data اللي اتحطت في `gpiochip_fwd_register()`. مكافئة لـ `gpiochip_get_data()` لكن بتتعامل مع الـ `gpiochip_fwd` مباشرةً.

**Parameters:** `fwd` — الـ GPIO forwarder

**Return Value:** الـ `data` pointer اللي اتمرر في `gpiochip_fwd_register()`، أو NULL لو اتمرر NULL.

---

### Group 3: GPIO Runtime Operations

كل الـ functions دي هي **public wrappers** فوق static callbacks داخلية. الـ wrapper بياخد `gpiochip_fwd *` عوضاً عن `gpio_chip *`، وبيعمل delegate للـ static function اللي بتشتغل على الـ `gpio_chip` المدمج.

```
gpiochip_fwd_gpio_XXX(fwd, ...)
    → gpiochip_fwd_get_gpiochip(fwd)      → &fwd->chip
    → gpio_fwd_XXX(chip, ...)             ← static, not exported
        → fwd = gpiochip_get_data(chip)
        → gpiod_XXX(fwd->descs[offset], ...)
            → Real GPIO chip driver
```

---

#### `gpiochip_fwd_gpio_request()`

```c
int gpiochip_fwd_gpio_request(struct gpiochip_fwd *fwd, unsigned int offset);
```

بتتحقق إن الـ slot المطلوبة موجودة فعلاً (`valid_mask` bit set). لو الـ line مش valid بترجع `-ENODEV` عشان gpiolib يعرف إن الـ line دي غير قابلة للاستخدام.

**Parameters:**
- `fwd` — الـ forwarder
- `offset` — رقم الـ virtual line

**Return Value:**
- `0` — الـ line موجودة وجاهزة
- `-ENODEV` — الـ slot فاضية

**Key Details:**
- مبتعملش أي GPIO request على الـ underlying descriptor — الـ `gpio_desc` بيكون مطلوب من الـ consumer الأصلي بالفعل.
- بيتنادى من gpiolib عند أي `gpiod_get()` أو `gpio_request()` على الـ forwarder chip.
- لا locking — الـ `test_bit()` على الـ bitmap آمنة للقراءة.

---

#### `gpiochip_fwd_gpio_get_direction()`

```c
int gpiochip_fwd_gpio_get_direction(struct gpiochip_fwd *fwd, unsigned int offset);
```

بتقرأ الاتجاه الحالي للـ GPIO line من خلال `gpiod_get_direction()` على الـ underlying descriptor.

**Parameters:**
- `fwd` — الـ forwarder
- `offset` — رقم الـ virtual line

**Return Value:**
- `GPIO_LINE_DIRECTION_IN` (1) — input
- `GPIO_LINE_DIRECTION_OUT` (0) — output
- `-ENODEV` — لو الـ slot فاضية (valid_mask bit = 0)

**Key Details:**
- الـ check على `valid_mask` هنا ضروري لأن gpiolib بتنادي الـ callback ده على كل الـ lines أثناء `devm_gpiochip_add_data` حتى اللي لسه ماتسجلتش.

---

#### `gpiochip_fwd_gpio_direction_input()`

```c
int gpiochip_fwd_gpio_direction_input(struct gpiochip_fwd *fwd,
                                      unsigned int offset);
```

بتحوّل الـ GPIO line للـ input mode بالـ `gpiod_direction_input()` على الـ underlying descriptor. بتفوّض للـ chip driver الحقيقي بشكل مباشر.

**Parameters:**
- `fwd` — الـ forwarder
- `offset` — رقم الـ virtual line

**Return Value:** `0` أو errno سالب من الـ underlying chip driver.

**Key Details:** المسؤولية الكاملة على الـ chip driver الحقيقي — الـ forwarder بس بيمرر الطلب بدون locking إضافي.

---

#### `gpiochip_fwd_gpio_direction_output()`

```c
int gpiochip_fwd_gpio_direction_output(struct gpiochip_fwd *fwd,
                                       unsigned int offset,
                                       int value);
```

بتحوّل الـ line للـ output وبتضبط قيمتها الابتدائية في نفس الوقت عبر `gpiod_direction_output()`.

**Parameters:**
- `fwd` — الـ forwarder
- `offset` — رقم الـ virtual line
- `value` — القيمة الابتدائية (logical — ACTIVE_LOW بيتأخذ في الحسبان)

**Return Value:** `0` أو errno سالب من الـ underlying driver.

---

#### `gpiochip_fwd_gpio_get()`

```c
int gpiochip_fwd_gpio_get(struct gpiochip_fwd *fwd, unsigned int offset);
```

بتقرأ القيمة الـ logical لـ line واحدة. لو الـ chip `can_sleep` بتستخدم `gpiod_get_value_cansleep()`، وإلا `gpiod_get_value()`.

**Parameters:**
- `fwd` — الـ forwarder
- `offset` — رقم الـ virtual line

**Return Value:**
- `0` أو `1` — القيمة الـ logical للـ line (مع مراعاة ACTIVE_LOW)
- errno سالب لو في error

**Key Details:**
- الاختيار بين `cansleep` وغيرها بيحصل runtime بناءً على `chip->can_sleep`.
- القيمة الـ logical بتأخذ في الحسبان الـ ACTIVE_LOW polarity تلقائياً من gpiolib.

---

#### `gpiochip_fwd_gpio_get_multiple()`

```c
int gpiochip_fwd_gpio_get_multiple(struct gpiochip_fwd *fwd,
                                   unsigned long *mask,
                                   unsigned long *bits);
```

بتقرأ عدة lines في operation واحدة بكفاءة. داخلياً بتجمع الـ `gpio_desc` pointers المقابلة للـ bits المحددة في `mask`، بتحطهم في `fwd->tmp` (الـ flex array)، وبتنادي `gpiod_get_array_value[_cansleep]()`.

**Parameters:**
- `mask` — bitmap بيحدد الـ lines المطلوب قراءتها (bit N = line N)
- `bits` — bitmap بيتملّى بالقيم المقروءة للـ lines المحددة في mask

**Return Value:**
- `0` — نجح، `bits` مملوء
- errno سالب من `gpiod_get_array_value[_cansleep]()`

**Key Details:**
- الـ internal function `gpio_fwd_get_multiple()` بتبني array مضغوطة من descs للـ lines الـ masked فقط، ثم بتوزع النتايج.
- محمية بـ `fwd->mlock` (mutex) لو `can_sleep`، وبـ `fwd->slock` (spinlock + irqsave) لو لأ.
- الـ locking ضروري لأن الـ `tmp[]` scratch buffer مشترك وممكن تتضارب لو اتنادى من contexts متعددة.

**Pseudocode للـ internal `gpio_fwd_get_multiple()`:**
```c
bitmap_clear(values, 0, fwd->chip.ngpio)
j = 0
for each set bit i in mask:
    descs[j++] = fwd->descs[i]

gpiod_get_array_value[_cansleep](j, descs, NULL, values)

j = 0
for each set bit i in mask:
    bits[i] = values[j++]
```

---

#### `gpiochip_fwd_gpio_set()`

```c
int gpiochip_fwd_gpio_set(struct gpiochip_fwd *fwd, unsigned int offset,
                          int value);
```

بتكتب قيمة logical على line واحدة. لو في `delay_timings` مضبوطة، بتضيف ramp delay بعد الكتابة بناءً على اتجاه التحويل والـ polarity.

**Parameters:**
- `fwd` — الـ forwarder
- `offset` — رقم الـ virtual line
- `value` — القيمة الـ logical المطلوب كتابتها

**Return Value:**
- `0` — نجح
- errno سالب من `gpiod_set_value[_cansleep]()`

**Key Details:**
- الـ delay feature مصممة للـ open-drain مع RC filter. الـ `gpio_fwd_delay()` بيحسب الـ delay المناسب بناءً على `ramp_up_us`/`ramp_down_us` والـ polarity (ACTIVE_LOW بتعكس المنطق).
- الـ delay بيستخدم `fsleep()` في `can_sleep` context و`udelay()` في atomic context.

---

#### `gpiochip_fwd_gpio_set_multiple()`

```c
int gpiochip_fwd_gpio_set_multiple(struct gpiochip_fwd *fwd,
                                   unsigned long *mask,
                                   unsigned long *bits);
```

بتكتب قيم على عدة lines في operation واحدة. بتستخدم `fwd->tmp[]` كـ scratch buffer وبتحميها بالـ lock المناسب — نفس pattern الـ `get_multiple` بس عكسياً.

**Parameters:**
- `mask` — bitmap الـ lines المطلوب تعديلها
- `bits` — bitmap القيم الجديدة للـ lines الـ masked

**Return Value:**
- `0` — نجح
- errno سالب من `gpiod_set_array_value[_cansleep]()`

**Key Details:**
- محمية بنفس الـ lock (mutex أو spinlock) حسب `can_sleep`.
- الـ bit j في `tmp_values` بيقابل الـ desc j في `tmp_descs` — mapping مضغوط.

---

#### `gpiochip_fwd_gpio_set_config()`

```c
int gpiochip_fwd_gpio_set_config(struct gpiochip_fwd *fwd, unsigned int offset,
                                 unsigned long config);
```

بتمرر packed pinconf config (بالـ format الـ standard `PIN_CONF_PACKED()`) للـ underlying descriptor عبر `gpiod_set_config()`.

**Parameters:**
- `fwd` — الـ forwarder
- `offset` — رقم الـ virtual line
- `config` — packed config value (pull-up/down، drive strength، debounce، إلخ)

**Return Value:**
- `0` — نجح
- `-ENOTSUPP` — لو الـ controller مش داعم الـ config
- errno سالب تاني من الـ underlying driver

**Key Details:** بتفوّض مباشرة لـ `gpiod_set_config()` — الـ forwarder مش بيفلتر أي config.

---

#### `gpiochip_fwd_gpio_to_irq()`

```c
int gpiochip_fwd_gpio_to_irq(struct gpiochip_fwd *fwd, unsigned int offset);
```

بترجع الـ Linux virtual IRQ number الخاص بالـ GPIO line عبر `gpiod_to_irq()` على الـ underlying descriptor.

**Parameters:**
- `fwd` — الـ forwarder
- `offset` — رقم الـ virtual line

**Return Value:**
- IRQ number موجب (>= 0) — نجح
- errno سالب — الـ line مش interrupt-capable

**Key Details:**
- الـ IRQ الراجع هو الـ IRQ الخاص بالـ underlying chip الأصلية، مش IRQ جديد — الـ forwarder مش بيعمل IRQ domain خاص بيه.
- مفيد للـ drivers اللي محتاجة تربط interrupt handler على GPIO line موجودة في الـ forwarder.

---

### الـ Locking Model

```
                ┌───────────────────────────────────────────┐
                │              gpiochip_fwd                 │
                │                                           │
  can_sleep=true│  mlock (mutex)    ──► get/set_multiple   │
  can_sleep=false│  slock (spinlock) ──► get/set_multiple   │
                │                                           │
  Single ops:   │  No forwarder lock — gpiolib handles it  │
                └───────────────────────────────────────────┘
```

الـ lock في الـ forwarder بيحمي فقط الـ `tmp[]` flex array اللي بتستخدمها الـ multi-ops. الـ single-line operations (get, set, direction) مش بتحتاج lock على مستوى الـ forwarder لأن كل descriptor بيتحمى بـ lock خاص بيه على مستوى gpiolib.

---

### الـ Export Namespace

كل الـ functions المصدّرة تستخدم:

```c
EXPORT_SYMBOL_NS_GPL(func_name, "GPIO_FORWARDER");
```

**الـ** module اللي بيستخدم الـ API ده لازم يعمل:

```c
MODULE_IMPORT_NS("GPIO_FORWARDER");
```

ده عزل متعمد من الـ kernel — بيضمن إن الـ drivers اللي بتستخدم الـ forwarder API بتعمل ده explicitly وبشكل documented.
## Phase 5: دليل الـ Debugging الشامل

الـ `gpiochip_fwd` (GPIO Forwarder) هو subsystem بيعمل virtual `gpio_chip` بيـforward العمليات لـ `gpio_desc` descriptors حقيقية موجودة في chips تانية. الـ debugging بتاعه بيتقسم لـ software level وـ hardware level.

---

### Software Level

#### 1. debugfs Entries

الـ GPIO subsystem بيعمل entries تحت `/sys/kernel/debug/gpio`:

```bash
# شوف كل الـ gpiochips المسجلة — هتلاقي الـ forwarded chip هنا
cat /sys/kernel/debug/gpio

# مثال على الـ output:
# gpiochip0: GPIOs 0-31, parent: platform/gpio-controller, my-gpio:
#  gpio-5  (                    ) in  hi
#  gpio-6  (                    ) out lo
#
# gpiochip1: GPIOs 32-35, parent: i2c/1-0074, gpio-forwarder:
#  gpio-32 (fwd[0]              ) in  hi
#  gpio-33 (fwd[1]              ) out lo
```

**تفسير الـ output:**
- الـ label `gpio-forwarder` في اسم الـ parent → ده الـ forwarded chip
- الـ offsets في الـ forwarded chip بتتطابق مع اللي اتضافت بـ `gpiochip_fwd_desc_add()`
- لو line تظهر كـ "not requested" رغم إنها اتضافت بـ `gpiochip_fwd_desc_add` → الـ consumer لسه ما استدعاش `gpiochip_fwd_gpio_request`

```bash
# شوف الـ IRQ domains المسجلة
ls /sys/kernel/debug/irq/domains/

# تفاصيل domain معين
cat /sys/kernel/debug/irq/domains/GPIO/name
cat /sys/kernel/debug/irq/domains/GPIO/allocated_irqs
```

| الـ Entry | المحتوى | أمر القراءة |
|-----------|---------|-------------|
| `/sys/kernel/debug/gpio` | كل الـ chips والـ lines وحالتها | `cat /sys/kernel/debug/gpio` |
| `/sys/kernel/debug/irq/domains/` | الـ IRQ domains + الـ `gpio_to_irq` mappings | `ls /sys/kernel/debug/irq/domains/` |
| `/sys/kernel/debug/pinctrl/*/pins` | الـ pinmux state للـ pins المرتبطة | `cat /sys/kernel/debug/pinctrl/*/pins` |

---

#### 2. sysfs Entries

```bash
# اعرف كل gpio chips الموجودة
ls /sys/class/gpio/

# الـ forwarded chip هتظهر كـ gpiochipN
cat /sys/class/gpio/gpiochip32/label     # اسم الـ chip
cat /sys/class/gpio/gpiochip32/ngpio     # عدد الـ GPIOs
cat /sys/class/gpio/gpiochip32/base      # أول رقم GPIO

# export GPIO للـ userspace واتحقق من الـ direction والـ value
echo 32 > /sys/class/gpio/export
cat /sys/class/gpio/gpio32/direction
cat /sys/class/gpio/gpio32/value
echo 32 > /sys/class/gpio/unexport

# شوف الـ device الـ forwarder متعلق بيه
ls -la /sys/class/gpio/gpiochip32/device/
```

```bash
# اطبع label + base + ngpio لكل chip — سهّل تحديد الـ forwarder
for chip in /sys/class/gpio/gpiochip*; do
    label=$(cat "$chip/label" 2>/dev/null)
    base=$(cat "$chip/base" 2>/dev/null)
    ngpio=$(cat "$chip/ngpio" 2>/dev/null)
    echo "$chip: label=$label  base=$base  ngpio=$ngpio"
done
```

---

#### 3. ftrace — Tracepoints والـ Events

```bash
# فعّل tracing للـ GPIO subsystem
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on

# أو events محددة
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_direction/enable
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_value/enable

# اقرأ الـ trace
cat /sys/kernel/debug/tracing/trace

# مثال على الـ output:
# <...>-1234  [001] ....  123.456789: gpio_direction: gpio=32 is output
# <...>-1234  [001] ....  123.456800: gpio_value: gpio=32 is 1
```

```bash
# trace بالـ function_graph للـ forwarder functions
TRACE=/sys/kernel/debug/tracing
echo 0 > $TRACE/tracing_on
echo function > $TRACE/current_tracer
echo 'gpiochip_fwd*' > $TRACE/set_ftrace_filter
echo 1 > $TRACE/tracing_on
# ... شغّل الكود المطلوب اختباره ...
echo 0 > $TRACE/tracing_on
cat $TRACE/trace
# cleanup
echo nop > $TRACE/current_tracer
echo > $TRACE/set_ftrace_filter
```

**تفسير ftrace output:**
```
# gpiochip_fwd_gpio_get <-- fwd=ffff888001234567 offset=0
# gpiochip_fwd_gpio_get --> ret=1           → القراءة ناجحة، القيمة HIGH
#
# gpiochip_fwd_gpio_request <-- offset=2
# gpiochip_fwd_gpio_request --> ret=-16     → -EBUSY: الـ line محجوزة من consumer تاني
```

---

#### 4. printk / dynamic debug

```bash
# فعّل dynamic debug لـ GPIO forwarder implementation
echo "file drivers/gpio/gpio-forwarder.c +pflm" > /sys/kernel/debug/dynamic_debug/control

# فعّل لكل الـ GPIO subsystem
echo "file drivers/gpio/* +p" > /sys/kernel/debug/dynamic_debug/control

# تحقق من الإعدادات الفعّالة
grep "gpio-forwarder\|gpiolib" /sys/kernel/debug/dynamic_debug/control

# رفع loglevel عشان KERN_DEBUG تظهر
echo 8 > /proc/sys/kernel/printk

# تابع الـ output
dmesg -w | grep -i "gpio\|forwarder"
```

**في kernel cmdline للـ early boot debug:**
```
dyndbg="file drivers/gpio/gpio-forwarder.c +p" loglevel=8
```

**مثال على الـ output بعد التفعيل:**
```
[  5.123456] gpio-forwarder: gpiochip_fwd_register: registering 4 GPIOs
[  5.123460] gpio-forwarder: gpiochip_fwd_desc_add: offset 0 linked to gpio32
[  5.123462] gpio-forwarder: gpiochip_fwd_gpio_request: offset 0 requested ok
```

---

#### 5. Kernel Config Options للـ Debugging

```bash
# تحقق من الـ configs اللي عندك
zcat /proc/config.gz | grep -E "CONFIG_(GPIO|DEBUG_GPIO|GPIOLIB)"
```

| الـ Config | الوظيفة | ملاحظة |
|------------|---------|--------|
| `CONFIG_GPIOLIB` | الـ GPIO library الأساسية | **لازم enabled** |
| `CONFIG_GPIO_FORWARDER` | يبني الـ forwarder driver | بدونه `gpiochip_fwd_alloc` مش موجودة |
| `CONFIG_DEBUG_GPIO` | extra validation + debug output لكل GPIO operations | **أهم خيار** |
| `CONFIG_GPIOLIB_IRQCHIP` | debugging الـ IRQ forwarding في الـ `gpio_to_irq` path | لازم لـ `gpiochip_fwd_gpio_to_irq` |
| `CONFIG_GPIO_SYSFS` | يكشف الـ GPIOs عن طريق sysfs | للـ manual debugging |
| `CONFIG_GPIO_CDEV` | character device interface للـ `gpioget`/`gpioset` | للـ userspace tools |
| `CONFIG_DEBUG_FS` | لازم enabled عشان debugfs يشتغل | بدونه مفيش `/sys/kernel/debug/gpio` |
| `CONFIG_DYNAMIC_DEBUG` | بيتيح `+p` flags للـ GPIO modules | مهم جداً |
| `CONFIG_PROVE_LOCKING` | يكشف deadlocks في الـ GPIO spinlocks | مفيد جداً |
| `CONFIG_LOCKDEP` | dependency tracking للـ GPIO locks | يكشف issues في `gpiochip_fwd_*` callbacks |
| `CONFIG_KASAN` | heap corruption detection | يكشف bugs في `devm_gpiochip_fwd_alloc` |

```
# .config المثالية للـ debugging session:
CONFIG_DEBUG_GPIO=y
CONFIG_GPIOLIB_IRQCHIP=y
CONFIG_GPIO_SYSFS=y
CONFIG_GPIO_CDEV=y
CONFIG_DEBUG_FS=y
CONFIG_DYNAMIC_DEBUG=y
CONFIG_PROVE_LOCKING=y
CONFIG_LOCKDEP=y
CONFIG_KASAN=y
CONFIG_DEBUG_KERNEL=y
```

---

#### 6. gpio-tools (libgpiod)

```bash
# تثبيت الأدوات
apt install gpiod   # أو: dnf install libgpiod-utils

# اعرض كل الـ chips — هتشوف الـ forwarder chip لو اتسجلت
gpiodetect
# مثال:
# gpiochip0 [pinctrl-bcm2835] (54 lines)
# gpiochip1 [pca9539] (16 lines)        ← backing chip
# gpiochip2 [gpio-forwarder] (4 lines)  ← الـ forwarder

# اعرض تفاصيل الـ forwarded chip
gpioinfo gpiochip2

# اقرأ قيمة GPIO معين عبر الـ forwarder
gpioget gpiochip2 0

# اكتب قيمة output
gpioset gpiochip2 0=1

# راقب events (لو الـ line بتدعم IRQ عبر gpiochip_fwd_gpio_to_irq)
gpiomon gpiochip2 0
```

---

#### 7. جدول رسائل الـ Error الشائعة

| رسالة الـ kernel log | المعنى | الحل |
|----------------------|--------|-------|
| `gpiochip_add_data: GPIOs N..M failed to register` | فشل تسجيل الـ virtual chip — ممكن overlap في الـ GPIO range | تحقق إن الـ base range مش محجوز من chip تانية |
| `gpio-N: enforced single user` | الـ line بتتطلب مرتين | استدعي `gpiochip_fwd_desc_free` قبل `gpiochip_fwd_desc_add` تاني |
| `-EBUSY` من `gpiochip_fwd_gpio_request` | الـ backing GPIO line محجوزة من consumer تاني | `cat /sys/kernel/debug/gpio` تحديد المالك |
| `-EINVAL` من `gpiochip_fwd_gpio_to_irq` | الـ backing chip مش بتدعم IRQ على الـ offset ده | تحقق من `gpio_irq_chip` في الـ backing chip |
| `-ENOMEM` من `devm_gpiochip_fwd_alloc` | memory allocation فشلت | تحقق من `/proc/meminfo` |
| `GPIO chip: Failed to request GPIO N` | `gpiochip_fwd_gpio_request` فشلت على الـ backing line | تأكد الـ backing GPIO chip شغّالة وـ probed |
| `gpiochip: tried to insert a chip with zero lines` | `ngpios == 0` في `devm_gpiochip_fwd_alloc` | راجع الـ `ngpios` parameter |
| `WARNING: at drivers/gpio/gpiolib.c:N` | `WARN_ON` داخل gpiolib تفعّل | اقرأ stack trace — غالباً invalid offset أو null desc |
| `irq: no parent irq` | الـ IRQ hierarchy مش مكتملة في `gpiochip_fwd_gpio_to_irq` | تحقق من `parent_domain` في `gpio_irq_chip` للـ backing chip |
| `gpio gpiochip0: chip was not properly removed` | race condition في الـ lifecycle | تأكد من ترتيب `gpiochip_fwd_register` وـ `devm_*` cleanup |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في gpiochip_fwd_gpio_request — تأكد الـ offset valid وـ desc موجود */
int gpiochip_fwd_gpio_request(struct gpiochip_fwd *fwd, unsigned int offset)
{
    if (WARN_ON(!fwd))                    /* null pointer guard */
        return -EINVAL;
    if (WARN_ON(offset >= fwd->ngpios))   /* bounds check */
        return -EINVAL;
    if (WARN_ON(!fwd->descs[offset])) {   /* desc not added yet */
        dump_stack();                      /* من فين اتطلب؟ */
        return -EINVAL;
    }
    return gpiod_request(fwd->descs[offset], "fwd");
}

/* في gpiochip_fwd_desc_add — تحقق من عدم التكرار */
int gpiochip_fwd_desc_add(struct gpiochip_fwd *fwd,
                          struct gpio_desc *desc, unsigned int offset)
{
    if (WARN_ON(!fwd || !desc))
        return -EINVAL;
    if (WARN_ON(offset >= fwd->ngpios))
        return -EINVAL;
    if (WARN_ON(fwd->descs[offset])) {    /* slot already occupied */
        dump_stack();                      /* اطبع call trace كامل */
        return -EBUSY;
    }
    fwd->descs[offset] = desc;
    return 0;
}

/* في gpiochip_fwd_register — منع الـ double-register */
int gpiochip_fwd_register(struct gpiochip_fwd *fwd, void *data)
{
    if (WARN_ON(!fwd))
        return -EINVAL;
    /* gpiochip_add_data يفشل لو الـ chip مسجلة بالفعل */
    return gpiochip_add_data(&fwd->chip, data);
}

/* في gpiochip_fwd_gpio_get — تأكد الـ backing desc لسه valid */
int gpiochip_fwd_gpio_get(struct gpiochip_fwd *fwd, unsigned int offset)
{
    struct gpio_desc *desc;

    if (WARN_ON(!fwd || offset >= fwd->ngpios))
        return -EINVAL;
    desc = fwd->descs[offset];
    if (WARN_ON(!desc)) {                 /* desc اتحذف بعد التسجيل؟ */
        dump_stack();
        return -ENODEV;
    }
    return gpiod_get_value(desc);
}
```

---

### Hardware Level

#### 1. التحقق من تطابق Hardware State مع Kernel State

الـ forwarder طبقة virtual — المشكلة ممكن تكون في:
- الـ parent GPIO chip (الـ hardware الحقيقية)
- الـ forwarded virtual interface
- الـ mapping بين الاتنين جوه `gpiochip_fwd_desc_add`

```bash
# 1. اقرأ القيمة من الـ parent chip مباشرةً
gpioget gpiochip0 5           # الـ backing GPIO pin

# 2. اقرأ نفس القيمة من الـ forwarder
gpioget gpiochip2 0           # offset 0 في الـ forwarder = GPIO 5 في الـ parent

# لو مختلفين → مشكلة في الـ forwarding logic أو الـ desc mapping

# 3. قارن الـ direction
gpioinfo gpiochip0 | grep "line 5"
gpioinfo gpiochip2 | grep "line 0"
```

```bash
# مقارنة منهجية بالـ sysfs
BACKING_GPIO=5
FWD_GPIO=32        # base الـ forwarder + offset

echo 5 > /sys/class/gpio/export
echo 32 > /sys/class/gpio/export

KERNEL_BACKING=$(cat /sys/class/gpio/gpio5/value)
KERNEL_FWD=$(cat /sys/class/gpio/gpio32/value)

echo "Backing GPIO-5: $KERNEL_BACKING  |  Forwarded GPIO-32: $KERNEL_FWD"
# لو مختلفين = bug في الـ gpiochip_fwd_gpio_get implementation

echo 5 > /sys/class/gpio/unexport
echo 32 > /sys/class/gpio/unexport
```

---

#### 2. Register Dump Techniques

الـ forwarder نفسه virtual — الـ register dump بيكون على الـ backing hardware chip:

```bash
# مثال: BCM2835/BCM2711 (Raspberry Pi)
# GPIO Base: 0xFE200000 (Pi4) أو 0x3F200000 (Pi3)

# GPLEV0 — current level لـ GPIO 0-31
devmem2 0xFE200034 w

# GPFSEL0 — function select GPIO 0-9 (3 bits per GPIO)
devmem2 0xFE200000 w

# GPSET0 — set output high GPIO 0-31
devmem2 0xFE20001C w

# GPCLR0 — set output low GPIO 0-31
devmem2 0xFE200028 w

# مثال على تفسير GPLEV0:
# Value = 0x00020000
# Bit 17 = 1 → GPIO-17 is HIGH
```

```bash
# لو الـ backing chip هو I2C GPIO expander (مثل PCA9539)
i2cdetect -y 1                  # تأكد الـ device موجود على bus 1

# قراءة input ports
i2cget -y 1 0x74 0x00           # Input Port 0
i2cget -y 1 0x74 0x01           # Input Port 1

# قراءة configuration (direction: 1=input, 0=output)
i2cget -y 1 0x74 0x06           # Config Port 0
i2cget -y 1 0x74 0x07           # Config Port 1

# مقارنة direction register مع ما يقوله الـ kernel:
# cat /sys/kernel/debug/gpio | grep gpiochip2
```

```bash
# بديل لو devmem2 مش متاح
python3 - <<'EOF'
import mmap, struct
GPIO_BASE = 0xFE200000
PAGE_SIZE = 4096
with open('/dev/mem', 'r+b') as f:
    m = mmap.mmap(f.fileno(), PAGE_SIZE, offset=GPIO_BASE & ~(PAGE_SIZE-1))
    off = GPIO_BASE & (PAGE_SIZE-1)
    lev0 = struct.unpack_from('<I', m, off + 0x34)[0]
    print(f'GPLEV0 = 0x{lev0:08X}')
    for i in range(32):
        print(f'  GPIO-{i:2d}: {"HIGH" if (lev0>>i)&1 else "LOW"}')
    m.close()
EOF
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

```
Signal               | ما تبحث عنه
---------------------|-------------------------------------------
GPIO output line     | هل الـ level بيتغير لما kernel تكتب؟
                     | لو لأ → مشكلة في الـ backing chip أو الـ pinmux
GPIO input line      | هل الـ kernel بتقرأ الـ level الصح؟
                     | قارن القراءة بـ multimeter
IRQ line             | هل الـ edge/level detection صح؟
                     | لو الـ IRQ مش بيـfire → مشكلة في gpiochip_fwd_gpio_to_irq
I2C bus (SDA/SCL)    | لو الـ backing chip I2C expander
                     | هل الـ read/write transactions بتحصل صح؟
                     | تحقق من الـ ACK bits
```

**نقاط عملية:**
- قس على الـ GPIO pin مباشرةً على الـ PCB — مش على الـ connector
- قس على الـ GPIO expander IC نفسه لو موجود
- Rise time > 1µs → pull-up ضعيف أو capacitance عالية
- HIGH يجب > 0.7 × VCC، وـ LOW يجب < 0.3 × VCC
- لو signal مش واصل لـ VCC/GND → driver strength ناقص

---

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة | Pattern في الـ Kernel Log | التشخيص |
|---------|--------------------------|---------|
| الـ I2C expander مش متصل | `i2c i2c-1: nack from addr 0x74` | `i2cdetect -y 1` — تأكد وجود الـ device |
| الـ GPIO مش configured direction | `gpio-N: direction output not supported` | راجع backing chip capabilities |
| الـ pull-up/pull-down مش configured | القراءة unstable تتغير عشوائياً | أضف `PIN_CONFIG_BIAS_PULL_UP` في `set_config` |
| voltage level mismatch (3.3V vs 1.8V) | دايماً HIGH حتى لو LOW | level shifter مطلوب في الـ hardware |
| الـ IRQ مش بييجي | `IRQ N: nobody cared` في dmesg | تحقق من `gpiochip_fwd_gpio_to_irq` ومن الـ interrupt routing |
| الـ backing chip ما اتسجلتش | `probe failed: -ENODEV` | تحقق من الـ DT compatible string |
| Power supply unstable | GPIO values تتغير عشوائياً | فحص VCC بالـ oscilloscope |
| race condition في الـ lifecycle | `gpio chip: chip was not properly removed` | راجع ترتيب `devm_*` cleanup وـ `gpiochip_fwd_register` |

---

#### 5. Device Tree Debugging

```bash
# اقرأ الـ compiled DT اللي اشتغل فعلاً
dtc -I fs /proc/device-tree 2>/dev/null | grep -A20 "gpio"

# تحقق من الـ backing GPIO chip node
cat /proc/device-tree/soc/i2c@7e804000/gpio@74/compatible
# Expected: nxp,pca9539

# تأكد إن الـ gpio-controller property موجودة
ls /proc/device-tree/soc/i2c@7e804000/gpio@74/gpio-controller 2>/dev/null \
  && echo "gpio-controller present" || echo "MISSING gpio-controller!"

# تحقق من #gpio-cells (لازم يكون 2)
od -t d4 /proc/device-tree/soc/i2c@7e804000/gpio@74/\#gpio-cells
```

```bash
# احفظ الـ DT الحالي وقارنه مع الـ source
dtc -I fs -O dts /proc/device-tree > /tmp/current.dts 2>/dev/null
diff /tmp/current.dts your_board.dts | head -50

# الـ overlays المطبقة
dtoverlay -l
```

**مثال DT لـ backing chip صح:**
```dts
&i2c1 {
    gpio_expander: gpio@74 {
        compatible = "nxp,pca9539";
        reg = <0x74>;
        gpio-controller;
        #gpio-cells = <2>;           /* لازم 2 عشان gpiod_get يشتغل */
        interrupt-parent = <&gpio>;
        interrupts = <4 IRQ_TYPE_LEVEL_LOW>;
        interrupt-controller;
        #interrupt-cells = <2>;
    };
};
```

**نقاط التحقق في الـ DT:**
- `gpio-controller` موجودة في node الـ backing chip
- `#gpio-cells = <2>` مش `<1>`
- `reg` بيطابق الـ I2C address الفعلي على الـ bus
- `interrupts` بيطابق الـ INT pin الموصل في الـ schematic

---

### Practical Commands

#### Script شامل للـ debugging من الصفر

```bash
#!/bin/bash
# gpio-forwarder-debug.sh

echo "=== [1] GPIO Chips ==="
gpiodetect 2>/dev/null || for chip in /sys/class/gpio/gpiochip*; do
    echo "  $(basename $chip): label=$(cat $chip/label) base=$(cat $chip/base) ngpio=$(cat $chip/ngpio)"
done

echo ""
echo "=== [2] debugfs GPIO State ==="
cat /sys/kernel/debug/gpio 2>/dev/null || echo "debugfs not mounted - run: mount -t debugfs none /sys/kernel/debug"

echo ""
echo "=== [3] Enable Dynamic Debug ==="
echo "file drivers/gpio/gpio-forwarder.c +pflm" \
  > /sys/kernel/debug/dynamic_debug/control 2>/dev/null && echo "OK" || echo "FAILED"

echo ""
echo "=== [4] Enable GPIO ftrace events ==="
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable 2>/dev/null
echo 1 > /sys/kernel/debug/tracing/tracing_on 2>/dev/null

echo ""
echo "=== [5] Recent GPIO Kernel Messages ==="
dmesg | grep -i "gpio\|forwarder\|gpiochip" | tail -30

echo ""
echo "=== [6] IRQ Domains ==="
ls /sys/kernel/debug/irq/domains/ 2>/dev/null

echo ""
echo "=== [7] Pinctrl Handles ==="
cat /sys/kernel/debug/pinctrl/pinctrl-handles 2>/dev/null | grep -i gpio | head -20
```

```bash
# فعّل GPIO debug logging وراقب real-time
echo 'file drivers/gpio/gpio-forwarder.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/gpio/gpiolib.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 8 > /proc/sys/kernel/printk
dmesg -w &
DMESG_PID=$!

# شغّل الـ test هنا ...
gpioget gpiochip2 0
gpioset gpiochip2 1=1

kill $DMESG_PID

# أوقف الـ debug logging
echo 'file drivers/gpio/gpio-forwarder.c -p' > /sys/kernel/debug/dynamic_debug/control
```

```bash
# اختبار كامل لكل lines في الـ forwarder chip
GPIO_CHIP="gpiochip2"
NGPIO=$(cat /sys/class/gpio/$GPIO_CHIP/ngpio)
BASE=$(cat /sys/class/gpio/$GPIO_CHIP/base)

echo "Testing $GPIO_CHIP ($NGPIO lines, base=$BASE)"
for i in $(seq 0 $((NGPIO - 1))); do
    GPIO=$((BASE + i))
    echo $GPIO > /sys/class/gpio/export 2>/dev/null
    DIR=$(cat /sys/class/gpio/gpio${GPIO}/direction 2>/dev/null || echo "N/A")
    VAL=$(cat /sys/class/gpio/gpio${GPIO}/value 2>/dev/null || echo "N/A")
    echo "  fwd-offset=$i  gpio=$GPIO  direction=$DIR  value=$VAL"
    echo $GPIO > /sys/class/gpio/unexport 2>/dev/null
done
```

```bash
# مقارنة backing chip مع forwarder — تحقق من صحة الـ forwarding
echo "=== Backing chip (gpiochip0) GPIO-5 ==="
gpioget gpiochip0 5

echo "=== Forwarded chip (gpiochip2) offset-0 ==="
gpioget gpiochip2 0

# لو مختلفين اطبع stack trace من الـ kernel
echo "file drivers/gpio/gpio-forwarder.c +pflm" > /sys/kernel/debug/dynamic_debug/control
gpioget gpiochip2 0
dmesg | tail -20
```

```bash
# تحقق من الـ IRQ mapping عبر gpiochip_fwd_gpio_to_irq
BASE=$(cat /sys/class/gpio/gpiochip2/base)
for i in 0 1 2 3; do
    GPIO=$((BASE + i))
    echo $GPIO > /sys/class/gpio/export 2>/dev/null
    EDGE=$(cat /sys/class/gpio/gpio${GPIO}/edge 2>/dev/null || echo "no-irq")
    echo "  fwd-offset=$i  gpio=$GPIO  edge/irq=$EDGE"
    echo $GPIO > /sys/class/gpio/unexport 2>/dev/null
done
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — GPIO Forwarder بيفشل بعد Soft Reset

#### العنوان
`gpiochip_fwd_register` بيرجع `-EBUSY` بعد soft reset في industrial gateway

#### السياق
Industrial IoT gateway مبني على **RK3562** بيشغّل Linux 6.6. الـ board فيها I2C-attached GPIO expander (PCA9555) بيوفّر 16 GPIO لـ relay outputs وdigital inputs. الـ gateway driver بيستخدم `gpiochip_fwd` عشان يجمّع الـ GPIOs دي في virtual chip واحدة. بعد soft reset (مش power cycle)، الـ relay driver بيوقف تماماً.

#### المشكلة

```
gpiochip_fwd_register: -EBUSY
relay-ctrl: failed to initialize GPIO forwarder
```

الـ relays بتقف، والـ gateway بيبقى isolated من الشبكة الصناعية.

#### التحليل
الـ `devm_gpiochip_fwd_alloc` بيربط الـ cleanup بعمر الـ `struct device`:

```c
/* forwarder.h */
struct gpiochip_fwd *devm_gpiochip_fwd_alloc(struct device *dev,
                                             unsigned int ngpios);
int gpiochip_fwd_register(struct gpiochip_fwd *fwd, void *data);
```

لما الـ driver بيعمل re-probe بعد reset، الـ device object لسه شايل الـ old forwarder اللي لسه registered في gpiolib. الـ `gpiochip_fwd_register` بتحاول تسجّل chip جديدة بنفس الاسم — فبترجع `-EBUSY`.

الكود الغلط:

```c
/* BUG: re-probe بدون cleanup للـ chip القديمة */
fwd = devm_gpiochip_fwd_alloc(&pdev->dev, 16);
if (IS_ERR(fwd))
    return PTR_ERR(fwd);

for (i = 0; i < 16; i++)
    gpiochip_fwd_desc_add(fwd, descs[i], i);

ret = gpiochip_fwd_register(fwd, drv_data);
/* هيرجع -EBUSY لأن chip الـ probe الأول لسه registered */
```

#### الحل

**تأكّد إن الـ device بيتحرر فعلاً قبل re-probe:**

```c
static int relay_driver_remove(struct platform_device *pdev)
{
    struct relay_drv *drv = platform_get_drvdata(pdev);
    int i;

    /* حرّر كل الـ offsets قبل ما الـ devm يحرر الـ fwd */
    for (i = 0; i < drv->ngpios; i++)
        gpiochip_fwd_desc_free(drv->fwd, i);

    /* devm هيحرر fwd نفسه automatically بعد return */
    return 0;
}
```

**للتحقق:**

```bash
# شوف الـ gpiochips المسجّلة قبل وبعد reset
cat /sys/kernel/debug/gpio

# unbind/bind يدوي للاختبار
echo 1-0027 > /sys/bus/i2c/drivers/pca9555/unbind
echo 1-0027 > /sys/bus/i2c/drivers/pca9555/bind
dmesg | tail -20
```

#### الدرس المستفاد
الـ `devm_gpiochip_fwd_alloc` بيربط الـ cleanup بعمر الـ device object، مش بعمر الـ driver binding. لازم تتأكد إن الـ device بيتحرر فعلاً قبل re-probe، أو تعمل `gpiochip_fwd_desc_free` يدوياً في الـ `remove` callback قبل الـ devm cleanup.

---

### السيناريو 2: Android TV Box على Allwinner H616 — HDMI HPD مش بيشتغل بعد Kernel Upgrade

#### العنوان
الـ HDMI Hot-Plug Detection بطّل يشتغل بعد تحديث الـ kernel من 6.1 لـ 6.6

#### السياق
Android TV box على **Allwinner H616**. الـ HDMI HPD (Hot-Plug Detection) pin متوصّل عن طريق intermediate driver بيعمل GPIO forwarding بين main SoC GPIO bank والـ DRM/HDMI driver. بعد upgrade، التلفزيون بطّل يتعرّف على الـ box لما تشيله وتحطّه تاني.

#### المشكلة
الـ DRM driver بيطلب IRQ:

```c
irq = gpiochip_fwd_gpio_to_irq(fwd, HPD_OFFSET);
```

بترجع قيمة سالبة أو IRQ للـ pin الغلط كلياً.

#### التحليل
الـ header:

```c
/* forwarder.h */
int gpiochip_fwd_desc_add(struct gpiochip_fwd *fwd,
                          struct gpio_desc *desc,
                          unsigned int offset);

int gpiochip_fwd_gpio_to_irq(struct gpiochip_fwd *fwd,
                              unsigned int offset);
```

الـ `gpiochip_fwd_desc_add` بيعمل binding ثابت بين `gpio_desc` وـ offset عددي. في الـ kernel الجديد، اتضاف EDID GPIO pin في نفس البنية، فـ offset الـ HPD اتغيّر من 3 لـ 4، بس الكود ما اتحدثش:

```
/* قبل الـ upgrade */
gpiochip_fwd_desc_add(fwd, hpd_desc, 3)  /* HPD على offset 3 */
gpiochip_fwd_gpio_to_irq(fwd, 3)         /* IRQ صح */

/* بعد الـ upgrade: offset 3 بقى لـ pin تاني */
gpiochip_fwd_desc_add(fwd, hpd_desc, 3)  /* غلط! */
gpiochip_fwd_gpio_to_irq(fwd, 3)         /* IRQ للـ pin الغلط */
```

#### الحل

**استخدم DT property للـ offset بدل hardcode:**

```c
/* قرّ الـ offset من DT عشان تتحمى من kernel layout changes */
ret = of_property_read_u32(dev->of_node, "hdmi-hpd-offset", &hpd_offset);
if (ret) {
    dev_err(dev, "missing hdmi-hpd-offset in DT\n");
    return ret;
}

hpd_desc = devm_gpiod_get(&dev, "hdmi-hpd", GPIOD_IN);
if (IS_ERR(hpd_desc))
    return PTR_ERR(hpd_desc);

ret = gpiochip_fwd_desc_add(fwd, hpd_desc, hpd_offset);
if (ret)
    return ret;

irq = gpiochip_fwd_gpio_to_irq(fwd, hpd_offset);
```

```dts
&hdmi_fwd_node {
    hdmi-hpd-offset = <4>; /* محدّث بعد إضافة EDID pin */
    hdmi-hpd-gpios = <&pio 7 3 GPIO_ACTIVE_HIGH>;
};
```

```bash
# تحقق من الـ IRQ mapping
cat /proc/interrupts | grep hpd

# شوف الـ GPIO descriptor المربوط بالـ offset الجديد
cat /sys/kernel/debug/gpio | grep fwd
```

#### الدرس المستفاد
الـ `gpiochip_fwd_desc_add` بيعمل binding ثابت على offset عددي — أي تغيير في layout الـ GPIOs يكسر الـ IRQ mapping. استخدام DT property للـ offset بيحمي من هذا الكسر عند kernel upgrades.

---

### السيناريو 3: IoT Sensor Node على STM32MP1 — SPI CS Lines بتتعارض بعد Re-probe

#### العنوان
SPI chip-select lines بتتخلط بسبب double `desc_add` بعد deferred probe

#### السياق
IoT sensor node على **STM32MP1** بيشغّل 3 SPI sensors (BME680 لـ humidity، ICM-42688 لـ IMU، LIS3DH لـ accelerometer). كل sensor فيه CS على GPIO pin منفصل. الـ system driver بيستخدم `gpiochip_fwd` لعزل الـ SPI GPIO bank وتوفيره لـ SPI controller. بعد deferred probe sequence، الـ SPI transactions بتتخلط والـ sensors بترجع بيانات غلط.

#### المشكلة
الـ driver بيعمل `gpiochip_fwd_desc_add` مرتين على نفس الـ `gpio_desc` بعد deferred probe re-trigger — مرة على offset 0 ومرة على offset 1:

```c
/* المرة الأولى في probe — صح */
gpiochip_fwd_desc_add(fwd, cs0_desc, 0);

/* deferred probe بيعيد trigger الـ probe */
/* BUG: نفس الـ cs0_desc اتحط على offset 1 كمان */
gpiochip_fwd_desc_add(fwd, cs0_desc, 1);
```

#### التحليل

```c
/* forwarder.h */
int gpiochip_fwd_desc_add(struct gpiochip_fwd *fwd,
                          struct gpio_desc *desc,
                          unsigned int offset);
void gpiochip_fwd_desc_free(struct gpiochip_fwd *fwd,
                            unsigned int offset);
```

الـ `gpiochip_fwd_desc_add` ما بتعمل check إن الـ `gpio_desc` اتضاف قبل كده على offset تاني. ده بيخلي فيه dangling mapping: offset 0 وـ offset 1 بيوصّلوا لنفس الـ physical CS pin.

التسلسل الصح:

```
probe 1: desc_add(fwd, cs0, 0)    ← تمام
deferred re-probe:
  desc_free(fwd, 0)               ← لازم تحرر أولاً
  desc_add(fwd, cs0, 0)           ← add تاني صح
```

#### الحل

```c
static int spi_fwd_probe(struct platform_device *pdev)
{
    struct spi_fwd_priv *priv = dev_get_drvdata(&pdev->dev);
    int i, ret;

    /* لو سبق وـ add → حرّر الكل أولاً */
    if (priv->fwd_initialized) {
        for (i = 0; i < NUM_CS; i++)
            gpiochip_fwd_desc_free(priv->fwd, i);
        priv->fwd_initialized = false;
    }

    /* add بترتيب صح */
    for (i = 0; i < NUM_CS; i++) {
        ret = gpiochip_fwd_desc_add(priv->fwd, priv->cs_descs[i], i);
        if (ret) {
            dev_err(&pdev->dev, "desc_add failed at offset %d\n", i);
            return ret;
        }
    }

    priv->fwd_initialized = true;
    return 0;
}
```

```bash
# كشف التعارض
dmesg | grep "gpio.*already requested"

# شوف مين بيستخدم الـ GPIO
cat /sys/kernel/debug/gpio | grep -E "cs[0-2]"
```

#### الدرس المستفاد
الـ `gpiochip_fwd_desc_add` ما بتتحقق من duplicate descriptors. في deferred probe scenarios، لازم تتحقق من حالة الـ fwd وتعمل `gpiochip_fwd_desc_free` على كل الـ offsets القديمة قبل re-add.

---

### السيناريو 4: Automotive ECU على i.MX8MP — GPIO Direction معكوس يسبب Fuel Injector Fault

#### العنوان
Crash في interrupt handler بسبب فهم غلط لقيمة `gpiochip_fwd_gpio_get_direction`

#### السياق
Automotive ECU على **i.MX8MP** بيشغّل Linux 6.1 مع PREEMPT_RT. الـ ECU بيتحكم في fuel injector control lines عن طريق GPIO forwarder بيعزل SoC GPIOs عن safety isolation layer. أثناء تشغيل المحرك، الـ engine management unit بيسجّل fault وبيوقف الحقن.

#### المشكلة
الـ safety ISR بيفحص direction الـ GPIO ويحاول يصحّحه — بس بيعمله بشكل معكوس تماماً:

```c
/* BUG: direction_input بدل ما يثبّت الـ output */
int dir = gpiochip_fwd_gpio_get_direction(fwd, INJECTOR_OFFSET);
if (dir == 0)  /* البرمجر فهم 0=input */
    gpiochip_fwd_gpio_direction_input(fwd, INJECTOR_OFFSET); /* خطأ! */
```

#### التحليل

```c
/* forwarder.h */
int gpiochip_fwd_gpio_get_direction(struct gpiochip_fwd *fwd,
                                    unsigned int offset);
int gpiochip_fwd_gpio_direction_input(struct gpiochip_fwd *fwd,
                                      unsigned int offset);
int gpiochip_fwd_gpio_direction_output(struct gpiochip_fwd *fwd,
                                       unsigned int offset,
                                       int value);
```

قيم الـ `get_direction` في الـ kernel:

```c
/* من <linux/gpio/driver.h> */
#define GPIO_LINE_DIRECTION_IN  1   /* ← input */
#define GPIO_LINE_DIRECTION_OUT 0   /* ← output */
```

`0 = output`، مش input — معكوس من اللي ممكن تتوقعه. الـ programmer فهم `0=input` فكان كل ما الـ pin في output mode (القيمة 0) بيحوّله لـ input — يعني بيكسر الـ injector control في كل مرة.

#### الحل

```c
#include <linux/gpio/driver.h>

static irqreturn_t injector_safety_isr(int irq, void *data)
{
    struct ecu_priv *priv = data;
    int dir;

    dir = gpiochip_fwd_gpio_get_direction(priv->fwd, INJECTOR_OFFSET);

    /* استخدم الـ constants مش hardcoded values */
    if (dir == GPIO_LINE_DIRECTION_IN) {
        /* الـ pin اتحوّل لـ input بشكل غير متوقع — استعادة فورية */
        dev_crit(priv->dev, "injector GPIO became input! restoring output\n");
        /* drive low للـ safety قبل إعادة enable الحقن */
        gpiochip_fwd_gpio_direction_output(priv->fwd,
                                           INJECTOR_OFFSET, 0);
    }

    return IRQ_HANDLED;
}
```

```bash
# مراقبة direction الـ GPIO في runtime
watch -n 0.5 'cat /sys/class/gpio/gpio<N>/direction'

# تتبّع direction changes
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_direction/enable
cat /sys/kernel/debug/tracing/trace | grep injector
```

#### الدرس المستفاد
**`GPIO_LINE_DIRECTION_OUT = 0`** و **`GPIO_LINE_DIRECTION_IN = 1`** — القيم معكوسة من الـ intuition الطبيعي. دايماً استخدم الـ constants من `<linux/gpio/driver.h>` مش الأرقام الحرفية. في Automotive context ده ممكن يؤدي لـ safety-critical failure مباشرة.

---

### السيناريو 5: Custom Board Bring-up على AM62x — `gpiochip_fwd_gpio_set_multiple` ما بتشتغلش Atomically

#### العنوان
LED matrix بتعمل flickering بسبب إن `set_multiple` مش بتعمل bulk write حقيقي

#### السياق
Custom industrial HMI board على **TI AM62x** (Sitara). الـ board فيها 8-LED status matrix متوصّلة على PCA9555 I2C GPIO expander. الـ UI driver بيستخدم `gpiochip_fwd_gpio_set_multiple` عشان يعمل update لكل الـ LEDs في وقت واحد ويتجنّب flickering. بعد الـ bring-up، الـ LEDs بتبان بتـ update واحدة واحدة.

#### المشكلة
الـ logic analyzer على الـ I2C bus بيكشف 8 transactions منفصلة لكل LED update بدل transaction واحدة للـ output register كله — ده بيسبب flickering واضح.

#### التحليل

```c
/* forwarder.h */
int gpiochip_fwd_gpio_set_multiple(struct gpiochip_fwd *fwd,
                                   unsigned long *mask,
                                   unsigned long *bits);
```

الـ `gpiochip_fwd_gpio_set_multiple` بتعمل loop على كل bit في الـ mask وبتكلّم `gpiochip_fwd_gpio_set` لكل واحدة بشكل منفصل. الـ PCA9555 driver عنده `set_multiple` حقيقي بيعمل I2C write واحدة للـ output register — بس الـ forwarder ما بيستخدمش ده، بيـbpass ده ويعمل individual sets.

الـ developer كان بيكتب:

```c
unsigned long mask = 0xFF;   /* كل الـ 8 LEDs */
unsigned long bits = led_state;
/* يتوقع I2C transaction واحدة — مش بيحصل */
gpiochip_fwd_gpio_set_multiple(fwd, &mask, &bits);
```

#### الحل

**الحل الأول: استخدم `gpiochip_fwd_get_gpiochip` للوصول لـ bulk operation:**

```c
/* forwarder.h */
struct gpio_chip *gpiochip_fwd_get_gpiochip(struct gpiochip_fwd *fwd);
```

```c
static void led_matrix_update_atomic(struct led_drv *drv, u8 state)
{
    struct gpio_chip *chip = gpiochip_fwd_get_gpiochip(drv->fwd);
    unsigned long mask = 0xFF;
    unsigned long bits = state;

    /* لو الـ chip بتدعم bulk set → استخدمه مباشرة */
    if (chip && chip->set_multiple) {
        chip->set_multiple(chip, &mask, &bits);
    } else {
        /* fallback للـ forwarder */
        gpiochip_fwd_gpio_set_multiple(drv->fwd, &mask, &bits);
    }
}
```

**الحل الثاني: اكتب للـ hardware register مباشرة لـ guaranteed atomicity:**

```c
static void led_matrix_update_direct(struct led_drv *drv, u8 state)
{
    /* PCA9555 output port 0 register */
    i2c_smbus_write_byte_data(drv->i2c_client,
                              PCA9555_OUTPUT_0, state);
}
```

```bash
# تحقق من الـ I2C transactions بـ logic analyzer أو i2c-tools
i2cdump -y 1 0x20  /* شوف output register قبل وبعد */

# trace الـ GPIO set calls
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_value/enable
cat /sys/kernel/debug/tracing/trace | grep -A1 "set"

# شوف الـ gpio_chip اللي behind الـ forwarder
cat /sys/kernel/debug/gpio | grep -B2 "fwd"
```

#### الدرس المستفاد
الـ `gpiochip_fwd_gpio_set_multiple` مش بتضمن atomicity على الـ underlying hardware — هي loop على individual sets. لو محتاج bulk write حقيقي (I2C transaction واحدة مثلاً)، استخدم `gpiochip_fwd_get_gpiochip` للوصول لـ `set_multiple` بتاع الـ parent chip، أو اكتب للـ hardware register مباشرة عن طريق الـ parent bus driver.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي المقالات الأهم على LWN.net اللي بتغطي الـ GPIO subsystem والـ GPIO Forwarder تحديداً:

| المقال | الأهمية |
|--------|---------|
| [gpio: Add GPIO Aggregator Driver](https://lwn.net/Articles/798971/) | الـ patch series الأصلية اللي أضافت الـ `gpiolib-fwd.c` والـ `gpiochip_fwd` |
| [gpio: Add GPIO Aggregator](https://lwn.net/Articles/809647/) | نسخة محدّثة من الـ aggregator مع تحسينات على الـ API |
| [GPIO in the kernel: an introduction](https://lwn.net/Articles/532714/) | مقدمة شاملة للـ GPIO subsystem والـ `gpio_chip` structure |
| [GPIO implementation framework](https://lwn.net/Articles/256461/) | أول تفصيل للـ framework من البداية، مهم لفهم الأساس |
| [gpio: consumer: new virtual driver](https://lwn.net/Articles/942479/) | virtual GPIO consumer driver — مرتبط مباشرة بمفهوم الـ forwarding |
| [gpio: aggregator: Introduce delay support](https://lwn.net/Articles/934274/) | تطوير لاحق على الـ aggregator بدعم delay للـ output pins |
| [gpio: sysfs interface (updated)](https://lwn.net/Articles/286435/) | الـ sysfs interface للـ GPIO — مهم لفهم كيف يتعامل userspace مع الـ gpiochip |

---

### التوثيق الرسمي في kernel.org

الـ `forwarder.h` جزء من الـ **GPIO Aggregator** subsystem. التوثيق الرسمي:

- **GPIO Aggregator (Admin Guide):**
  `Documentation/admin-guide/gpio/gpio-aggregator.rst`
  → [docs.kernel.org/admin-guide/gpio/gpio-aggregator.html](https://docs.kernel.org/admin-guide/gpio/gpio-aggregator.html)

- **GPIO Driver Interface:**
  `Documentation/driver-api/gpio/driver.rst`
  → [docs.kernel.org/driver-api/gpio/driver.html](https://docs.kernel.org/driver-api/gpio/driver.html)

- **GPIO General Index:**
  `Documentation/driver-api/gpio/index.rst`
  → [docs.kernel.org/driver-api/gpio/index.html](https://docs.kernel.org/driver-api/gpio/index.html)

- **GPIO Consumer Interface:**
  `Documentation/driver-api/gpio/consumer.rst`
  → [docs.kernel.org/driver-api/gpio/consumer.html](https://docs.kernel.org/driver-api/gpio/consumer.html)

- **GPIO Board/Mappings:**
  `Documentation/driver-api/gpio/board.rst`
  → [docs.kernel.org/driver-api/gpio/board.html](https://docs.kernel.org/driver-api/gpio/board.html)

- **Legacy GPIO Interfaces:**
  `Documentation/driver-api/gpio/legacy.rst`
  → [static.lwn.net/kerneldoc/driver-api/gpio/legacy.html](https://static.lwn.net/kerneldoc/driver-api/gpio/legacy.html)

---

### ملفات الـ Source Code المرتبطة

الـ `forwarder.h` معرَّف كـ public API لملف implementation التالي:

```
drivers/gpio/gpiolib-fwd.c      ← الـ implementation الكامل لـ gpiochip_fwd
drivers/gpio/gpio-aggregator.c  ← الـ driver اللي بيستخدم الـ forwarder
include/linux/gpio/forwarder.h  ← الـ header موضوع البحث
```

---

### Kernel Commits المهمة

الـ commits اللي أدخلت الـ GPIO Forwarder إلى الـ kernel:

- **الـ commit الأصلي للـ GPIO Aggregator (Linux 5.9):**
  الـ patch series من Geert Uytterhoeven أضاف:
  - الـ `gpiolib-fwd.c` كـ forwarder helper
  - الـ `gpio-aggregator.c` كـ aggregator driver
  - export لـ `gpiod_request`, `gpiod_free`, `gpiochip_get_desc`

- **مناقشات الـ mailing list عن الـ forwarder refactoring:**
  - [PATCH v9: gpio: aggregator: refactor to add GPIO desc in forwarder](https://www.mail-archive.com/linux-hardening@vger.kernel.org/msg10383.html)
  - [PATCH v9: handle runtime registration of gpio_desc in gpiochip_fwd](https://www.mail-archive.com/linux-hardening@vger.kernel.org/msg10386.html)
  - [PATCH v9: add possibility to attach data to the forwarder](https://www.mail-archive.com/linux-hardening@vger.kernel.org/msg10388.html)
  - [PATCH v5: export symbols of the GPIO forwarder library](https://www.mail-archive.com/linux-hardening@vger.kernel.org/msg09393.html)

- **الـ diff بخصوص الـ gpiochip_fwd على kernel.googlesource:**
  [0486beaf88d2](https://kernel.googlesource.com/pub/scm/linux/kernel/git/brauner/linux/+/0486beaf88d2460e9dbcbba281dab683a838f0c6%5E1..0486beaf88d2460e9dbcbba281dab683a838f0c6/)

---

### مصادر eLinux.org

- [Pin Control and GPIO Update (PDF)](https://elinux.org/images/c/cd/Pincontrol-gpio-update.pdf) — overview شامل لـ pinctrl وGPIO في Linux
- [Leapster Explorer: GPIO subsystem](https://elinux.org/Leapster_Explorer:_GPIO_subsystem) — مثال عملي على subsystem كامل

---

### مصادر Bootlin وغيرها

- [GPIO Aggregator, a virtual GPIO chip — Bootlin Blog](https://bootlin.com/blog/gpio-aggregator-a-virtual-gpio-chip/) — شرح عملي ممتاز للـ aggregator وعلاقته بالـ forwarder
- [Stop using /sys/class/gpio — The Good Penguin](https://www.thegoodpenguin.co.uk/blog/stop-using-sys-class-gpio-its-deprecated/) — ليه الـ new descriptor-based API أهم من القديم

---

### كتب مُوصى بيها

#### Linux Device Drivers 3rd Edition (LDD3)
- **Chapter 6: Advanced Char Driver Operations** — بيغطي interrupts وI/O
- **Chapter 14: The Linux Device Model** — فهم الـ `device`, `devm_*` lifecycle
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **Chapter 17: Devices and Modules** — شرح الـ device model والـ bus/driver registration
- بيساعد على فهم الـ `devm_gpiochip_fwd_alloc` وإزاي الـ devm lifecycle بيشتغل

#### Embedded Linux Primer — Christopher Hallinan
- **Chapter 15: Embedded Linux GPIO** — GPIO من منظور embedded systems
- بيغطي الـ device tree GPIO bindings والـ consumer API

---

### Kernel Newbies — مناقشات مفيدة

- [gpiod_get() - How to get GPIO from chip & offset?](http://lists.kernelnewbies.org/pipermail/kernelnewbies/2022-August/022663.html)
- [Using GPIO from device tree on platform devices](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2015-August/014911.html)
- [Linux_6.9 — GPIO changes](https://kernelnewbies.org/Linux_6.9) — changelog بيشمل GPIO updates

---

### Search Terms للبحث عن معلومات أكتر

لو حابب تعمق أكتر، استخدم الـ search terms دي:

```
gpiochip_fwd linux kernel
gpio aggregator forwarder linux
gpiolib-fwd.c implementation
devm_gpiochip_fwd_alloc
gpio forwarder library kernel
linux GPIO virtualization gpiochip
gpio-aggregator.c driver analysis
linux kernel gpio descriptor API
gpiod_get gpiod_put consumer kernel
```

---

### ملخص المصادر حسب الأولوية

| الأولوية | المصدر | السبب |
|---------|--------|-------|
| 1 | [LWN: GPIO Aggregator Driver](https://lwn.net/Articles/798971/) | الـ patch series الأصلية للـ forwarder |
| 2 | [docs.kernel.org GPIO Aggregator](https://docs.kernel.org/admin-guide/gpio/gpio-aggregator.html) | التوثيق الرسمي |
| 3 | [Bootlin: GPIO Aggregator blog](https://bootlin.com/blog/gpio-aggregator-a-virtual-gpio-chip/) | شرح عملي بأمثلة |
| 4 | [LWN: GPIO in the kernel intro](https://lwn.net/Articles/532714/) | أساس الـ GPIO subsystem |
| 5 | [mail-archive: PATCH v9 series](https://www.mail-archive.com/linux-hardening@vger.kernel.org/msg10383.html) | التطور الأخير للـ forwarder API |
## Phase 8: Writing simple module

### الفكرة

هنعمل module بيستخدم **kprobe** عشان يعمل hook على `gpiochip_fwd_register` — الدالة اللي بتسجّل الـ forwarded GPIO chip في الـ kernel. كل ما حد حاول يسجّل chip، المودل يطبع اسم الـ device جوّا kernel log.

الدالة دي تمّ اختيارها لأنها:
- **exported** وبتتكلم من context آمن (process context)
- بتحصل نادراً فمش هتغرق الـ log
- بتحمل pointer على `gpiochip_fwd` و`data` اللي ممكن نطبع منها معلومات مفيدة

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_gpiofwd.c
 * Hooks gpiochip_fwd_register() to log every GPIO forwarder registration.
 */
#include <linux/module.h>      /* module_init, module_exit, MODULE_* macros */
#include <linux/kernel.h>      /* pr_info */
#include <linux/kprobes.h>     /* kprobe struct and API */
#include <linux/gpio/forwarder.h> /* gpiochip_fwd — forward declaration only */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Doc Project");
MODULE_DESCRIPTION("kprobe on gpiochip_fwd_register to log GPIO forwarder registrations");

/*
 * pre_handler is called right before gpiochip_fwd_register() executes.
 *
 * Signature of the hooked function:
 *   int gpiochip_fwd_register(struct gpiochip_fwd *fwd, void *data);
 *
 * On x86-64:
 *   regs->di = first arg  (fwd)
 *   regs->si = second arg (data)
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * نجيب الـ arguments من الـ registers مباشرةً.
     * ما بنـdereference الـ fwd pointer هنا عشان الـ struct داخلي
     * وmight not be exported fully — بس طباعة العناوين كافية للـ demo.
     */
    void *fwd  = (void *)regs->di;  /* first arg: gpiochip_fwd* */
    void *data = (void *)regs->si;  /* second arg: driver private data */

    pr_info("kprobe_gpiofwd: gpiochip_fwd_register called | fwd=%px data=%px\n",
            fwd, data);

    return 0; /* 0 = let the original function continue normally */
}

/* Called after gpiochip_fwd_register() returns — optional, we use it to log ret value */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * regs->ax على x86-64 بيحمل return value بعد ما الدالة خلصت.
     * لو الـ register نجح ax = 0، غير كده بتكون error code.
     */
    pr_info("kprobe_gpiofwd: gpiochip_fwd_register returned %ld\n",
            (long)regs->ax);
}

/* kprobe descriptor — بنحدد اسم الدالة اللي عايزين نعمل عليها hook */
static struct kprobe kp = {
    .symbol_name    = "gpiochip_fwd_register",
    .pre_handler    = handler_pre,
    .post_handler   = handler_post,
};

static int __init kprobe_gpiofwd_init(void)
{
    int ret;

    /*
     * register_kprobe بتحط breakpoint على أول instruction
     * في gpiochip_fwd_register. لو الدالة مش موجودة في kernel
     * (مثلاً CONFIG_GPIO_FORWARDER=n) هترجع -ENOENT.
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("kprobe_gpiofwd: register_kprobe failed, ret=%d\n", ret);
        return ret;
    }

    pr_info("kprobe_gpiofwd: planted at %px\n", kp.addr);
    return 0;
}

static void __exit kprobe_gpiofwd_exit(void)
{
    /*
     * لازم نعمل unregister قبل ما الـ module يتشال من الذاكرة،
     * عشان لو جه interrupt أثناء الـ unload والـ handler لسّه
     * registered هيـcrash الـ kernel (use-after-free).
     */
    unregister_kprobe(&kp);
    pr_info("kprobe_gpiofwd: unregistered, kprobe removed\n");
}

module_init(kprobe_gpiofwd_init);
module_exit(kprobe_gpiofwd_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | الـ macros الأساسية لأي kernel module |
| `linux/kernel.h` | `pr_info` / `pr_err` |
| `linux/kprobes.h` | `struct kprobe`, `register_kprobe`, `unregister_kprobe` |
| `linux/gpio/forwarder.h` | forward declarations للـ types — مش محتاجينه فعلاً بس بيوضّح الـ dependency |

**الـ** `linux/kprobes.h` هو القلب — بيوفّر الـ API كلّه اللي محتاجينه نعمل hook على أي kernel symbol.

---

#### الـ `handler_pre`

```c
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
```

بيتنفّذ قبل أول instruction في `gpiochip_fwd_register`. **الـ** `pt_regs` بيحمل state الـ CPU لحظة الـ hook، فنقدر نجيب الـ arguments من `regs->di` و`regs->si` على x86-64 بدون ما نلمس الـ stack.

**الـ** return value لازم يكون `0` — أي قيمة تانية بتخلي الـ kprobe يعيد تشغيل الـ instruction بشكل مختلف وده advanced use case مش محتاجه هنا.

---

#### الـ `handler_post`

```c
static void handler_post(struct kprobe *p, struct pt_regs *regs, unsigned long flags)
```

بيتنفّذ بعد ما الدالة ترجع. **الـ** `regs->ax` على x86-64 بيحمل الـ return value، فنقدر نعرف على طول لو الـ registration نجح ولا لأ.

---

#### الـ `struct kprobe kp`

```c
static struct kprobe kp = {
    .symbol_name = "gpiochip_fwd_register",
    ...
};
```

**الـ** `.symbol_name` بيخلي الـ kernel يحل الـ symbol تلقائياً من `kallsyms` — مش محتاجين نحدد address يدوياً. لو الـ symbol مش موجود أو غير exported سيرجع error في `register_kprobe`.

---

#### الـ `module_init` / `module_exit`

- **`register_kprobe`** بتحط software breakpoint (على x86 بتستبدل أول byte بـ `int3`). لو فشلت يجب نرجع الـ error فوراً.
- **`unregister_kprobe`** في الـ exit ضرورية جداً — لو تركنا الـ kprobe مسجّل وشلنا الـ module، الـ kernel هيحاول يستدعي `handler_pre` اللي اتشال من الذاكرة وده kernel panic فوري.

---

### Makefile للبناء

```makefile
obj-m += kprobe_gpiofwd.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### تجربة المودل

```bash
# بناء
make

# تحميل
sudo insmod kprobe_gpiofwd.ko

# تحقق إن الـ kprobe اتسجّل
sudo dmesg | grep kprobe_gpiofwd

# لو عندك GPIO forwarder device، جرّب bind/unbind عشان تطلق الـ trigger
# مثلاً:
echo "some-device" | sudo tee /sys/bus/platform/drivers/gpio-forwarder/bind

# تشيل المودل
sudo rmmod kprobe_gpiofwd
sudo dmesg | tail -5
```

**مثال output متوقع:**

```
[  42.123456] kprobe_gpiofwd: planted at ffffffffc0ab1234
[  45.789012] kprobe_gpiofwd: gpiochip_fwd_register called | fwd=ffff888003a10000 data=ffff888003b20000
[  45.789015] kprobe_gpiofwd: gpiochip_fwd_register returned 0
[  50.001234] kprobe_gpiofwd: unregistered, kprobe removed
```

---

### ليه `gpiochip_fwd_register` تحديداً؟

| معيار | التقييم |
|-------|---------|
| **آمنة للـ kprobe** | process context، مش في interrupt handler |
| **نادرة الاستدعاء** | بتتكلم بس وقت تسجيل الـ chip مش في كل GPIO operation |
| **معلومات مفيدة** | بتاخد `fwd` و`data` اللي بيمثلوا core objects |
| **exported** | موجودة في `kallsyms` وممكن نحل اسمها |
