## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

**الـ GPIO Aggregator** جزء من subsystem الـ **GPIO** في Linux kernel، وبالتحديد من drivers الـ GPIO الموجودة في `drivers/gpio/`. الـ maintainer هو Geert Uytterhoeven وبيتابع الملف عبر قائمة بريد `linux-gpio@vger.kernel.org`.

---

### الصورة الكبيرة — قبل أي كود

#### المشكلة اللي بتحلها

تخيل عندك لوحة تحكم صناعية (مصنع، أو نظام embedded)، وعندك controller GPIO واحد فيه 32 pin. بعض الـ pins دي بتتحكم في باب، وبعضها بيتحكم في مضخة مية، وبعضها بيتحكم في إنذار.

في Linux، الـ GPIO controller بيظهر كـ `/dev/gpiochip0` — جهاز واحد بكل الـ 32 pin. المشكلة إن لو فتحت الباب لأي user أو VM يوصل لـ `/dev/gpiochip0`، بتفتح له الباب على **كل** الـ 32 pin — مش بس اللي هو محتاجه. ده خطر أمني واضح.

المشكلة التانية: لو عندك **Virtual Machine** شغالة وعايز تديها تحكم في بعض الـ GPIO pins، محتاج تعمل لها passthrough. لو ديت لها الـ gpiochip كله، خطر. ولو حاولت تعمل pin-by-pin passthrough، ده معقد وسطح الهجوم (attack surface) بيكبر.

#### الحل — الـ GPIO Aggregator

الـ **GPIO Aggregator** هو driver بيعمل **gpio_chip وهمي جديد** من مجموعة مختارة من pins — يجوا من chip واحدة أو أكتر.

تخيله زي مفتاح توزيع كهربي (patch panel): بدل ما توصّل الكابل الأصلي بكل الأسلاك، بتعمل panel صغيرة فيها بس الأسلاك اللي محتاجها، وبتوصل الـ VM أو الـ user بالـ panel دي بس.

```
Physical GPIO Controllers
┌─────────────────┐    ┌─────────────────┐
│  gpiochip0      │    │  gpiochip1      │
│  pins: 0..31   │    │  pins: 0..31   │
└────┬───┬───┬───┘    └────┬───┬────────┘
     │19  │20  │21          │5   │6
     └────┴────┴────────────┴────┘
                    │
          GPIO Aggregator (kernel)
                    │
         ┌──────────▼──────────┐
         │  gpio-aggregator.0  │  ← virtual gpiochip جديد
         │  line0 ← chip0/pin19│
         │  line1 ← chip0/pin20│
         │  line2 ← chip0/pin21│
         │  line3 ← chip1/pin5 │
         │  line4 ← chip1/pin6 │
         └─────────────────────┘
                    │
              /dev/gpiochipX  ← User أو VM يوصل هنا بس
```

#### ليه ده مفيد؟

1. **Access Control دقيق**: تقدر تعمل chown/chmod على الـ gpiochip الوهمي فيوصله user معين بس — زي ما بتعمل على أي file في Linux.
2. **تبسيط الـ VM Passthrough**: الـ VM بتاخد gpiochip كامل ومش محتاجة تعرف التفاصيل، فده بيقلل الـ attack surface.
3. **Generic GPIO Driver**: بيخليك تشغّل أي device بسيط بيشتغل بـ GPIO (زي باب أو lock) من غير ما تكتب driver خاص ليه — كفاية تعرّفه في Device Tree وتربطه بالـ aggregator.

---

### طرق إنشاء الـ Aggregator

#### 1. عبر Sysfs (الطريقة القديمة والسريعة)

بتكتب string فيها أسماء الـ chips والـ offsets في ملف واحد:

```bash
# اجمع pin 19 من chip الأولى + pins 20-21 من chip التانية
$ echo 'e6052000.gpio 19 e6050000.gpio 20-21' > \
  /sys/bus/platform/drivers/gpio-aggregator/new_device

# لما تخلص، احذفه
$ echo gpio-aggregator.0 > \
  /sys/bus/platform/drivers/gpio-aggregator/delete_device
```

#### 2. عبر Configfs (الطريقة الحديثة والمرنة)

بتعمل directories وبتضبط كل line على حدة، وبعدين تفعّله:

```bash
# إنشاء aggregator اسمه agg0
$ mkdir /sys/kernel/config/gpio-aggregator/agg0

# تعريف line0
$ mkdir /sys/kernel/config/gpio-aggregator/agg0/line0
$ echo gpiochip0 > /sys/kernel/config/gpio-aggregator/agg0/line0/key
$ echo 6         > /sys/kernel/config/gpio-aggregator/agg0/line0/offset
$ echo my_pin    > /sys/kernel/config/gpio-aggregator/agg0/line0/name

# تفعيل الـ aggregator
$ echo 1 > /sys/kernel/config/gpio-aggregator/agg0/live
```

#### 3. كـ Generic GPIO Driver مع Device Tree

لو عندك device بسيط في DT بيستخدم GPIO (زي باب إلكتروني):

```c
// في Device Tree
door {
    compatible = "myvendor,mydoor";
    gpios = <&gpio2 19 GPIO_ACTIVE_HIGH>,
            <&gpio2 20 GPIO_ACTIVE_LOW>;
    gpio-line-names = "open", "lock";
};
```

```bash
# ربط الـ device بالـ aggregator driver
$ echo gpio-aggregator > /sys/bus/platform/devices/door/driver_override
$ echo door > /sys/bus/platform/drivers/gpio-aggregator/bind

# النتيجة: gpiochip جديد اسمه "door" فيه line "open" و"lock"
$ gpioinfo door
```

---

### قصة المشكلة كاملة

**السيناريو**: عندك نظام embedded بيشغّل VM لنظام تحكم في مصنع. الـ VM محتاجة تتحكم في 3 pins بس من أصل 32 pin في الـ GPIO controller.

**قبل الـ Aggregator**: لازم إما تدي الـ VM الـ gpiochip كله (خطر)، أو تعمل نظام معقد يوزع الـ pins يدوياً (صعب ومش آمن).

**بعد الـ Aggregator**:
1. Admin بيعمل virtual gpiochip فيه الـ 3 pins بس.
2. بيعمل `chown vm_user /dev/gpiochipX` على الـ virtual chip.
3. الـ VM بتشوف gpiochip كامل (بـ 3 pins) وتتعامل معاه عادي.
4. مفيش طريقة للـ VM توصل للـ pins التانية.

---

### الملفات اللي بتكوّن الـ Subsystem ده

| الملف | الدور |
|-------|-------|
| `drivers/gpio/gpio-aggregator.c` | **Core implementation** — الـ driver الرئيسي، منطق الـ sysfs والـ configfs والـ aggregation |
| `include/linux/gpio/forwarder.h` | **GPIO Forwarder API** — الـ interface اللي بيستخدمه الـ aggregator لإنشاء الـ virtual chip |
| `include/linux/gpio/driver.h` | **GPIO Driver API** — تعريفات `gpio_chip` والـ operations الأساسية |
| `include/linux/gpio/consumer.h` | **GPIO Consumer API** — `gpiod_*` functions للاستخدام من الـ driver |
| `include/linux/gpio/machine.h` | **GPIO Machine API** — `gpiod_lookup_table` لربط الـ GPIOs بالـ device |
| `drivers/gpio/dev-sync-probe.c` | **Synchronous Probe Helper** — helper بيتأكد إن الـ device اتـ probe بنجاح قبل ما يرجع |
| `Documentation/admin-guide/gpio/gpio-aggregator.rst` | **هذا الملف** — توثيق الـ userspace interface |
| `Documentation/admin-guide/gpio/gpio-sim.rst` | GPIO simulator — ذات الـ domain، لأغراض الـ testing |
| `Documentation/admin-guide/gpio/gpio-virtuser.rst` | GPIO virtual user — tool لاختبار الـ GPIO من userspace |

### ملفات مرتبطة مهم تعرفها

- `drivers/gpio/gpio-mockup.c` — GPIO mockup driver للـ testing
- `drivers/gpio/gpio-line-mux.c` — GPIO line multiplexer (فكرة مشابهة للـ aggregation)
- `Documentation/driver-api/gpio/` — الـ API documentation للمطورين
## Phase 2: شرح الـ GPIO Aggregator Framework

### المشكلة اللي بيحلها الـ GPIO Aggregator

#### مشكلة الـ Access Control

الـ Linux kernel بيعرض كل GPIO controller كـ `/dev/gpiochipN` character device. الـ access control على المستوى ده بيكون all-or-nothing — إما المستخدم عنده access على الـ chip كلها، أو ملوش access خالص. ده بيخلق مشكلتين:

1. **مشكلة الأمان**: لو عايز تدي process معينة أو VM تتحكم في GPIO رقم 5 فقط من gpiochip0 اللي فيها 32 GPIO، مفيش طريقة مباشرة — هتديهم access على الـ chip كلها أو تحرمهم منها كلها.

2. **مشكلة الـ Virtual Machines**: الـ VM محتاجة تعرف بالظبط أنهيها pins تاخد وأنهيها تسيب — ده يوسّع الـ attack surface وبيزوّد التعقيد.

#### مشكلة الـ Simple GPIO Devices

أجهزة بسيطة زي LED controller أو relay driver أو door lock بتتوصف في الـ Device Tree بـ GPIOs لكن من غير driver مخصص في الـ kernel. الـ engineer مجبور إما:
- يكتب driver كامل في الـ kernel لجهاز بسيط جداً.
- يكلم الـ GPIO مباشرة من userspace بطريقة غير منظمة.

---

### الحل اللي بيقدمه الـ GPIO Aggregator

الـ **GPIO Aggregator** بيعمل الآتي:
- بياخد مجموعة GPIOs من chip واحدة أو أكتر (بـ offset مختلفة).
- بيلملمهم في **gpio_chip افتراضية جديدة** — وكأنها controller مستقل.
- الـ chip الجديدة دي بتظهر كـ `/dev/gpiochipX` عادية، وبتتتحكم فيها بـ UNIX permissions زي أي file.

الـ framework بيوفر طريقتين للإنشاء:
- **Sysfs**: كتابة string لـ `/sys/bus/platform/drivers/gpio-aggregator/new_device`
- **Configfs**: إنشاء directory tree تحت `/config/gpio-aggregator/`

---

### التشبيه الواقعي — لوحة الباتش (Patch Panel)

تخيل مبنى فيه **central telecom room** فيها patch panel ضخم بـ 96 port. ده هو الـ GPIO controller الأصلي (`gpiochip0` مثلاً بـ 96 pin).

كل قسم في المبنى (HR، IT، المحاسبة) محتاج access على ports محددة بس — مش على اللوحة كلها.

الـ technician بيعمل **sub-patch panel صغيرة** لكل قسم — بياخد منها الـ ports المطلوبة بس ويوصّلها في اللوحة الصغيرة دي.

| مكوّن الـ analogy | المقابل في الـ kernel |
|---|---|
| الـ central patch panel (96 port) | الـ `gpio_chip` الأصلية (e.g. gpiochip0) |
| الـ ports الفردية | الـ GPIO lines بـ offsets محددة |
| الـ sub-panel بتاعة قسم IT | الـ aggregated `gpio_chip` الجديدة |
| الـ cable من central لـ sub-panel | الـ `gpiochip_fwd` forwarding mechanism |
| الـ UNIX file ownership على `/dev/gpiochipX` | الـ UNIX permissions على اللوحة الصغيرة |
| القسم اللي بيستخدم اللوحة الصغيرة | الـ VM أو الـ user process |
| الـ technician | الـ sysadmin اللي بيكتب في `new_device` |

الـ original chip ما اتغيرتش — الـ aggregator بس بيعمل "view" منتقاة فوقيها.

---

### البنية العامة — Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Userspace                                 │
│                                                                  │
│  echo 'e6052000.gpio 19 e6050000.gpio 20-21' > new_device       │
│  gpioget /dev/gpiochip5 0 1 2                                   │
└────────────────────────┬────────────────────────────────────────┘
                         │ sysfs / configfs / ioctl
┌────────────────────────▼────────────────────────────────────────┐
│                      gpiolib (kernel core)                       │
│                                                                  │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │              GPIO Aggregator Driver                       │  │
│   │                                                           │  │
│   │  ┌──────────────┐    ┌───────────────────────────────┐  │  │
│   │  │  new_device   │    │      gpiochip_fwd (forwarder) │  │  │
│   │  │  sysfs attr  │───▶│                               │  │  │
│   │  └──────────────┘    │  gpio_desc* descs[]           │  │  │
│   │  ┌──────────────┐    │  gpio_chip  fwd_chip          │  │  │
│   │  │  configfs    │───▶│                               │  │  │
│   │  │  interface   │    │  .get()  ──────────────────┐  │  │  │
│   │  └──────────────┘    │  .set()  ──────────────────┤  │  │  │
│   │                      │  .direction_input() ───────┤  │  │  │
│   │                      └───────────────────────────┬┘  │  │  │
│   └──────────────────────────────────────────────────┼───┘  │  │
│                                                       │       │  │
│        ┌──────────────────────────────────────────────▼──┐   │  │
│        │            Real GPIO chips (providers)           │   │  │
│        │                                                  │   │  │
│        │  ┌───────────────┐    ┌───────────────────────┐ │   │  │
│        │  │ e6052000.gpio │    │    e6050000.gpio       │ │   │  │
│        │  │ gpiochip0     │    │    gpiochip1           │ │   │  │
│        │  │ (96 lines)    │    │    (32 lines)          │ │   │  │
│        │  │               │    │                        │ │   │  │
│        │  │ line 19 ──────│────│──── line 20, 21 ───── │ │   │  │
│        │  └───────────────┘    └───────────────────────┘ │   │  │
│        └──────────────────────────────────────────────────┘   │  │
│                                                                  │  │
│        ┌──────────────────────────────────────────────────┐   │  │
│        │         Aggregated virtual chip (consumer view)   │   │  │
│        │  /dev/gpiochip5   "gpio-aggregator.0"             │   │  │
│        │  line 0 → e6052000.gpio[19]                       │   │  │
│        │  line 1 → e6050000.gpio[20]                       │   │  │
│        │  line 2 → e6050000.gpio[21]                       │   │  │
│        └──────────────────────────────────────────────────┘   │  │
└─────────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ abstraction المحوري هنا هو الـ **`gpiochip_fwd` (GPIO Chip Forwarder)**.

الفكرة: بدل ما تكلم الـ hardware مباشرة، الـ aggregator بيسجل `gpio_chip` جديدة callbacks بتاعتها كلها بتعمل **dispatch** للـ descriptor الأصلي بتاع الـ real chip.

```c
/*
 * gpiochip_fwd هو الـ object الوسيط اللي بيربط
 * الـ virtual chip بالـ real descriptors
 */
struct gpiochip_fwd {
    struct gpio_chip chip;      /* الـ virtual gpio_chip اللي بتتسجل في gpiolib */
    struct gpio_desc **descs;   /* array of pointers للـ real gpio_desc objects */
    /* ... mutex, can_sleep flag, إلخ */
};
```

لما حد بيعمل `gpiod_get_value()` على الـ virtual chip:

```
userspace ioctl
      │
      ▼
gpiolib core يكلم fwd_chip->get(offset)
      │
      ▼
gpiochip_fwd_gpio_get(fwd, offset)
      │
      ▼  /* ببساطة: */
gpiod_get_value(fwd->descs[offset])  /* real descriptor */
      │
      ▼
real gpio_chip->get(real_offset)
      │
      ▼
hardware register
```

---

### طرق الإنشاء بالتفصيل

#### 1. Sysfs Interface

```bash
# aggregator بياخد line 19 من chip أولى + lines 20,21 من chip تانية
$ echo 'e6052000.gpio 19 e6050000.gpio 20-21' > \
    /sys/bus/platform/drivers/gpio-aggregator/new_device

# بتظهر virtual chip جديدة
$ gpioinfo gpio-aggregator.0
gpiochip5 - 3 lines:
    line 0: unnamed  unused  input  active-high
    line 1: unnamed  unused  input  active-high
    line 2: unnamed  unused  input  active-high

# تحذف الـ aggregator
$ echo gpio-aggregator.0 > \
    /sys/bus/platform/drivers/gpio-aggregator/delete_device
```

الـ format بتاع الـ string:
```
[<gpio-line-name>] [<gpiochip-label> <offsets>] ...
```
- اسم line مباشرة: بيتعمل lookup بالاسم
- اسم chip + offsets: بيتعمل lookup بالـ label والـ offset range

#### 2. Configfs Interface

الـ **Configfs** subsystem (لازم تفهمه الأول): filesystem خاص في الـ kernel بيخلي userspace ينشئ kernel objects من خلال إنشاء directories وكتابة attributes — زي DT لكن dynamic وقت الـ runtime.

```bash
# إنشاء aggregator
$ mkdir /config/gpio-aggregator/agg0

# إضافة line 0
$ mkdir /config/gpio-aggregator/agg0/line0
$ echo gpiochip0 > /config/gpio-aggregator/agg0/line0/key
$ echo 6         > /config/gpio-aggregator/agg0/line0/offset
$ echo my-led    > /config/gpio-aggregator/agg0/line0/name

# إضافة line 1
$ mkdir /config/gpio-aggregator/agg0/line1
$ echo gpiochip0 > /config/gpio-aggregator/agg0/line1/key
$ echo 7         > /config/gpio-aggregator/agg0/line1/offset

# تفعيل الـ aggregator
$ echo 1 > /config/gpio-aggregator/agg0/live
```

الـ attributes بتاعة كل line:

| Attribute | القيمة الافتراضية | الوظيفة |
|---|---|---|
| `key` | فاضي | اسم الـ chip أو اسم الـ line |
| `offset` | -1 | رقم الـ offset لو الـ key هو chip name |
| `name` | فاضي | اسم اختياري للـ line في الـ aggregator |

الـ `live` attribute هو الزناد — بيبدأ الـ probe synchronously في Configfs (بيرجع error لو فشل) على عكس Sysfs اللي بيكون async.

```
Configfs directory structure:
/config/gpio-aggregator/
├── agg0/
│   ├── live          (rw: 0/1)
│   ├── dev_name      (ro: "gpio-aggregator.0")
│   ├── line0/
│   │   ├── key       ("gpiochip0")
│   │   ├── offset    (6)
│   │   └── name      ("my-led")
│   └── line1/
│       ├── key       ("gpiochip0")
│       ├── offset    (7)
│       └── name      ("")
└── _sysfs.0/         (auto-generated لو اتنشأ من sysfs)
    ├── live
    └── line0/ ...
```

---

### الاستخدام كـ Generic GPIO Driver

الـ GPIO Aggregator بيقدر يشتغل كـ platform driver عادي ويـ bind لأجهزة موصوفة في الـ Device Tree من غير driver مخصص:

```dts
/* Device Tree node */
door {
    compatible = "myvendor,mydoor";

    gpios = <&gpio2 19 GPIO_ACTIVE_HIGH>,
            <&gpio2 20 GPIO_ACTIVE_LOW>;
    gpio-line-names = "open", "lock";
};
```

```bash
# ربط الجهاز يدوياً بالـ aggregator driver
$ echo gpio-aggregator > /sys/bus/platform/devices/door/driver_override
$ echo door > /sys/bus/platform/drivers/gpio-aggregator/bind

# النتيجة: virtual chip باسم الجهاز
$ gpioinfo door
gpiochip12 - 2 lines:
    line 0: "open"  unused  input  active-high
    line 1: "lock"  unused  input  active-high
```

ده مفيد جداً في الـ industrial control زي الـ `spidev` بالظبط بس للـ GPIO.

---

### العلاقة بين الـ Structs

```
┌─────────────────────────────────────────────────────┐
│                  gpiochip_fwd                        │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │              gpio_chip (fwd_chip)             │   │
│  │                                               │   │
│  │  label:    "gpio-aggregator.0"               │   │
│  │  ngpio:    3                                  │   │
│  │  base:     -1 (dynamic)                       │   │
│  │                                               │   │
│  │  .get()           ─────────────────────────┐ │   │
│  │  .set()           ─────────────────────────┤ │   │
│  │  .direction_input()  ──────────────────────┤ │   │
│  │  .direction_output() ──────────────────────┤ │   │
│  │  .get_direction()    ──────────────────────┤ │   │
│  │  .set_config()       ──────────────────────┘ │   │
│  └──────────────────────────────────────────────┘   │
│                         │                            │
│                         ▼                            │
│  descs[]:                                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │ gpio_desc│  │ gpio_desc│  │ gpio_desc│          │
│  │ [0]      │  │ [1]      │  │ [2]      │          │
│  │ →chip0   │  │ →chip1   │  │ →chip1   │          │
│  │ →offset19│  │ →offset20│  │ →offset21│          │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘          │
└───────┼─────────────┼─────────────┼──────────────────┘
        │             │             │
        ▼             ▼             ▼
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │gpio_chip │  │gpio_chip │  │gpio_chip │
  │e6052000  │  │e6050000  │  │e6050000  │
  │(real HW) │  │(real HW) │  │(real HW) │
  └──────────┘  └──────────┘  └──────────┘
```

---

### ماذا يمتلك الـ Aggregator مقابل ما يفوّضه

#### ما يمتلكه الـ GPIO Aggregator:

| المسؤولية | التفاصيل |
|---|---|
| **إنشاء وتسجيل** الـ virtual `gpio_chip` | بيسجّلها في gpiolib كـ chip كاملة |
| **Namespace isolation** | كل line في الـ aggregator عندها offset جديد من 0 |
| **Ownership و permissions** | الـ `/dev/gpiochipX` الجديدة بتتتحكم فيها بـ UNIX perms |
| **Lifecycle management** | إنشاء وتدمير الـ virtual chip عند الطلب |
| **Sysfs و Configfs** interface | الـ API بتاع الـ userspace لإنشاء الـ aggregators |

#### ما يُفوِّضه لـ الـ real drivers:

| المسؤولية | مين بيعملها |
|---|---|
| **قراءة وكتابة** الـ GPIO فعلياً | الـ real `gpio_chip` driver (e.g. Renesas GPIO driver) |
| **Direction control** | الـ real driver |
| **IRQ handling** | الـ real driver (الـ aggregator مش بيدعم IRQs للـ aggregated lines) |
| **Hardware initialization** | الـ real driver |
| **Pinctrl integration** | الـ real driver |
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Enums والـ Config Options

#### الـ `gpio_lookup_flags` enum

| Flag | Value | المعنى |
|------|-------|---------|
| `GPIO_ACTIVE_HIGH` | `0 << 0` | المنطق عادي — high = 1 |
| `GPIO_ACTIVE_LOW` | `1 << 0` | المنطق معكوس — low = 1 |
| `GPIO_OPEN_DRAIN` | `1 << 1` | open-drain output |
| `GPIO_OPEN_SOURCE` | `1 << 2` | open-source output |
| `GPIO_PERSISTENT` | `0 << 3` | الـ GPIO يحتفظ بقيمته |
| `GPIO_TRANSITORY` | `1 << 3` | الـ GPIO قد يتغير وحده |
| `GPIO_PULL_UP` | `1 << 4` | pull-up مفعّل |
| `GPIO_PULL_DOWN` | `1 << 5` | pull-down مفعّل |
| `GPIO_PULL_DISABLE` | `1 << 6` | pull مُعطَّل |
| `GPIO_LOOKUP_FLAGS_DEFAULT` | `ACTIVE_HIGH \| PERSISTENT` | القيمة الافتراضية |

#### الـ `FWD_FEATURE_*` Flags (بيتفلاگز الـ Forwarder)

| Flag | Value | المعنى |
|------|-------|---------|
| `FWD_FEATURE_DELAY` | `BIT(0)` | يفعّل رفع/خفض الـ ramp delay لكل line عبر DT |

#### الـ Defines الثابتة

| Define | Value | المعنى |
|--------|-------|---------|
| `AGGREGATOR_MAX_GPIOS` | `512` | أقصى عدد GPIOs في aggregator واحد |
| `AGGREGATOR_LEGACY_PREFIX` | `"_sysfs"` | بريفكس اسم config group اللي أنشأه sysfs |
| `U16_MAX` (sentinel) | `65535` | يدل على lookup بالاسم مش بالـ offset |

---

### 1. الـ Structs المهمة

#### 1.1 `struct gpio_aggregator`

**الغرض:** يمثّل **controller افتراضي** كامل — بيجمع مجموعة GPIO lines من chips حقيقية في وحدة واحدة. ده الـ "مدير" الأساسي للـ aggregator.

```c
struct gpio_aggregator {
    struct dev_sync_probe_data probe_data; // بيانات المزامنة مع probe
    struct config_group group;             // configfs group node
    struct gpiod_lookup_table *lookups;   // جدول بحث GPIO → chip الحقيقي
    struct mutex lock;                     // يحمي الحالة الداخلية
    int id;                                // رقم تعريف فريد في IDR

    struct list_head list_head;            // قائمة gpio_aggregator_line مرتبة

    bool init_via_sysfs;                   // true لو الإنشاء كان من sysfs
    char args[];                           // flexible array — args الـ sysfs النصية
};
```

| Field | الدور |
|-------|-------|
| `probe_data` | يحتوي `platform_device *` وـ`completion` للمزامنة مع الـ probe |
| `group` | تمثيل الـ aggregator في شجرة configfs |
| `lookups` | جدول الـ GPIO lookup اللي بيُسجَّل في gpiolib |
| `lock` | mutex يحمي `list_head`، `lookups`، وحالة active/activating |
| `id` | مفتاح البحث في `gpio_aggregator_idr` |
| `list_head` | رأس قائمة مرتبة من `gpio_aggregator_line` |
| `init_via_sysfs` | يفرّق السلوك بين sysfs وconfigfs |
| `args[]` | flexible array — النص المكتوب في `new_device` sysfs attribute |

---

#### 1.2 `struct gpio_aggregator_line`

**الغرض:** يمثّل **سطر GPIO واحد** داخل الـ aggregator — بيعرّف من فين تجيب الـ GPIO الحقيقية.

```c
struct gpio_aggregator_line {
    struct config_group group;   // configfs group node لـ lineN/
    struct gpio_aggregator *parent; // مؤشر للـ aggregator الأب
    struct list_head entry;      // حلقة في قائمة الأب

    unsigned int idx;            // رقم الـ line في الـ aggregator (0, 1, 2...)
    const char *name;            // اسم مخصص للـ line (اختياري)
    const char *key;             // اسم chip أو اسم line
    int offset;                  // -1 = lookup بالاسم، >= 0 = رقم offset في الـ chip
    enum gpio_lookup_flags flags; // ACTIVE_HIGH/LOW, pull, إلخ
};
```

| Field | الدور |
|-------|-------|
| `key` | لو offset == -1 → اسم الـ GPIO line؛ لو offset >= 0 → label الـ chip |
| `offset` | رقم الـ line في الـ chip؛ U16_MAX يُستخدم داخليًا للـ line-name lookup |
| `flags` | بيتفلاگز من `gpio_lookup_flags` |

---

#### 1.3 `struct gpio_aggregator_pdev_meta`

**الغرض:** بيانات platform صغيرة تُمرَّر للـ `platform_device` عند الإنشاء.

```c
struct gpio_aggregator_pdev_meta {
    bool init_via_sysfs; // يفرّق الـ probe behavior
};
```

---

#### 1.4 `struct gpiochip_fwd`

**الغرض:** الـ **GPIO Forwarder** — الـ `gpio_chip` الوهمي اللي بيحوّل كل عمليات GPIO لـ descriptors حقيقية.

```c
struct gpiochip_fwd {
    struct gpio_chip chip;              // الـ gpio_chip المسجّل في gpiolib
    struct gpio_desc **descs;           // مصفوفة مؤشرات للـ GPIO descriptors الحقيقية
    union {
        struct mutex mlock;             // يحمي tmp[] لو can_sleep
        spinlock_t slock;               // يحمي tmp[] لو !can_sleep
    };
    struct gpiochip_fwd_timing *delay_timings; // توقيتات ramp up/down لكل line
    void *data;                         // driver-private data
    unsigned long *valid_mask;          // بتماب — أي slots فيها descs صالحة
    unsigned long tmp[];                // scratch buffer لعمليات get/set_multiple
};
```

| Field | الدور |
|-------|-------|
| `chip` | الـ gpio_chip الفعلي المرئي لـ gpiolib |
| `descs` | كل `descs[i]` = gpio_desc للـ GPIO الحقيقية |
| `mlock/slock` | اختيار union بناءً على `chip.can_sleep` |
| `valid_mask` | بتماب يُمكّن الـ `gpio_fwd_request` من رفض slots غير مُهيّأة |
| `tmp[]` | flexible scratch array — أول `BITS_TO_LONGS(ngpio)` longs = values، الباقي = descs لعمليات array |

---

#### 1.5 `struct gpiochip_fwd_timing`

**الغرض:** يخزّن توقيتات ramp up/down لكل GPIO line لدعم open-drain بـ RC filter.

```c
struct gpiochip_fwd_timing {
    u32 ramp_up_us;    // تأخير ما بعد set HIGH (microseconds)
    u32 ramp_down_us;  // تأخير ما بعد set LOW (microseconds)
};
```

---

#### 1.6 `struct gpiod_lookup_table` و `struct gpiod_lookup`

**الغرض:** جدول بحث المعياري في gpiolib — بيربط platform device بـ GPIO chips.

```c
struct gpiod_lookup {
    const char *key;       // label الـ chip أو اسم الـ line
    u16 chip_hwnum;        // رقم الـ GPIO في الـ chip (U16_MAX = lookup بالاسم)
    const char *con_id;    // اسم الوظيفة (NULL في الـ aggregator)
    unsigned int idx;      // فهرس ترتيبي
    unsigned long flags;   // gpio_lookup_flags
};

struct gpiod_lookup_table {
    struct list_head list;        // حلقة في قائمة gpiolib العالمية
    const char *dev_id;           // اسم الـ platform device (مثلاً "gpio-aggregator.0")
    struct gpiod_lookup table[];  // flexible array من gpiod_lookup مُنتهي بـ {}
};
```

---

#### 1.7 `struct dev_sync_probe_data`

**الغرض:** يضمن المزامنة مع الـ probe عند كتابة `live=1` في configfs.

```c
struct dev_sync_probe_data {
    struct platform_device *pdev;       // مؤشر للـ pdev بعد التسجيل
    const char *name;
    struct notifier_block bus_notifier; // يستمع لـ bus events
    struct completion probe_completion; // ينتظر اكتمال probe
    bool driver_bound;                  // صح لو الـ probe نجح
};
```

---

### 2. مخططات العلاقات بين الـ Structs

```
                        gpio_aggregator_idr (IDR)
                               │
                    ┌──────────▼──────────┐
                    │  gpio_aggregator    │
                    │  ─────────────────  │
                    │  probe_data         │◄─── dev_sync_probe_data
                    │  group ─────────────│───► configfs group
                    │  lookups ───────────│───► gpiod_lookup_table
                    │  lock (mutex)       │          │
                    │  id                 │          │ table[]
                    │  list_head ─────────│──┐       ▼
                    │  init_via_sysfs     │  │   gpiod_lookup[]
                    │  args[]             │  │   (key, hwnum, idx, flags)
                    └─────────────────────┘  │
                                             │ list_for_each
                    ┌────────────────────────▼──┐
                    │  gpio_aggregator_line      │
                    │  ─────────────────────     │
                    │  group ────────────────────│───► configfs group
                    │  parent ───────────────────│───► gpio_aggregator (أب)
                    │  entry (list_head)          │
                    │  idx                        │
                    │  key / offset / name        │
                    │  flags                      │
                    └────────────────────────────┘

                    platform_device (gpio-aggregator.N)
                               │
                               │ probe()
                               ▼
                    ┌─────────────────────┐
                    │   gpiochip_fwd      │
                    │   ───────────────   │
                    │   chip ─────────────│───► gpio_chip (مسجّل في gpiolib)
                    │   descs[] ──────────│───► gpio_desc* (من chips حقيقية)
                    │   valid_mask        │
                    │   mlock / slock     │
                    │   delay_timings ────│───► gpiochip_fwd_timing[]
                    │   data              │
                    │   tmp[]             │
                    └─────────────────────┘
```

---

### 3. دورة الحياة (Lifecycle)

#### 3.1 المسار عبر Configfs

```
mkdir /config/gpio-aggregator/agg0
       │
       ▼
gpio_aggregator_make_group()
  → gpio_aggregator_alloc()          [kzalloc + idr_alloc]
  → config_group_init_type_name()
  → dev_sync_probe_init()
  → يُرجع &aggr->group

mkdir agg0/line0
       │
       ▼
gpio_aggregator_device_make_group()
  → gpio_aggregator_line_alloc()     [kzalloc + kstrdup]
  → config_group_init_type_name()
  → gpio_aggregator_line_add()       [يُدرج في القائمة مرتّبًا]

echo gpiochip0 > agg0/line0/key
echo 6         > agg0/line0/offset
echo test0     > agg0/line0/name

echo 1 > agg0/live
       │
       ▼
gpio_aggregator_device_live_store()
  → try_module_get()
  → gpio_aggregator_lockup_configfs() [configfs_depend_item]
  → [lock aggr->lock]
  → gpio_aggregator_activate()
       │
       ├─► gpio_aggregator_make_device_sw_node()   [software fwnode]
       ├─► gpio_aggregator_add_gpio() × N          [يبني lookup table]
       ├─► gpiod_add_lookup_table()
       └─► dev_sync_probe_register()
              │
              ▼
         platform_device_register_full()
              │
              ▼
         gpio_aggregator_probe()                   [في الـ platform driver]
              │
              ├─► gpiod_count()
              ├─► devm_gpiod_get_index() × N
              └─► gpiochip_fwd_create()
                     │
                     ├─► devm_gpiochip_fwd_alloc()
                     ├─► gpiochip_fwd_desc_add() × N
                     └─► gpiochip_fwd_register()
                            │
                            ▼
                       devm_gpiochip_add_data()
                            │
                            ▼
                      /dev/gpiochipX جاهز ✓

echo 0 > agg0/live
       │
       ▼
gpio_aggregator_deactivate()
  → dev_sync_probe_unregister()      [platform_device_unregister]
  → gpiod_remove_lookup_table()
  → kfree(lookups->dev_id)
  → kfree(lookups)

rmdir agg0/line0
       │
       ▼
gpio_aggregator_line_release()
  → [lock aggr->lock]
  → gpio_aggregator_line_del()
  → kfree(key) + kfree(name) + kfree(line)

rmdir agg0
       │
       ▼
gpio_aggregator_device_release()
  → gpio_aggregator_free()
       → idr_remove()
       → mutex_destroy()
       → kfree(aggr)
```

#### 3.2 المسار عبر Sysfs (Legacy)

```
echo 'e6052000.gpio 19 e6050000.gpio 20-21' > new_device
       │
       ▼
gpio_aggregator_new_device_store()
  → try_module_get()
  → gpio_aggregator_alloc()          [size = count+1 للـ args]
  → memcpy(aggr->args, buf)
  → aggr->init_via_sysfs = true
  → كزalloc lookups table
  → config_group_init_type_name()    [اسم = "_sysfs.N"]
  → dev_sync_probe_init()
  → configfs_register_group()        [يظهر في /config/gpio-aggregator/_sysfs.N]
  → gpio_aggregator_parse()          [يفرّق lines ويبني lookup table]
  → gpiod_add_lookup_table()
  → platform_device_register_data()  [مع pdev_meta]
       │
       ▼
  gpio_aggregator_probe()            [نفس المسار أعلاه]

echo gpio-aggregator.0 > delete_device
       │
       ▼
gpio_aggregator_delete_device_store()
  → idr_find() + idr_remove()
  → gpio_aggregator_destroy()
       → [lock] gpio_aggregator_deactivate()
       → gpio_aggregator_free_lines()
       → configfs_unregister_group()
       → kfree(aggr)
```

---

### 4. تدفق الاستدعاء (Call Flow)

#### 4.1 تدفق القراءة (gpio_get)

```
userspace: gpioget gpiochipX 0
    │
    ▼
gpio_chip.get(chip, offset=0)         ← gpio_fwd_get()
    │
    ▼
gpiod_get_value(fwd->descs[0])        ← لو !can_sleep
  أو
gpiod_get_value_cansleep(fwd->descs[0]) ← لو can_sleep
    │
    ▼
gpio_desc الحقيقية في chip الأصلية
    │
    ▼
قراءة register الـ hardware الفعلي
```

#### 4.2 تدفق قراءة متعددة (get_multiple)

```
gpio_chip.get_multiple(chip, mask, bits)    ← gpio_fwd_get_multiple_locked()
    │
    ├─ can_sleep → mutex_lock(&fwd->mlock)
    │               gpio_fwd_get_multiple()
    │               mutex_unlock()
    │
    └─ !can_sleep → spin_lock_irqsave(&fwd->slock, flags)
                    gpio_fwd_get_multiple()
                    spin_unlock_irqrestore()
                         │
                         ▼
              يبني مصفوفة descs[] مؤقتة في fwd->tmp
                         │
                         ▼
              gpiod_get_array_value[_cansleep](j, descs, NULL, values)
                         │
                         ▼
              ينسخ النتائج من values المؤقتة لـ bits المخرجات
```

#### 4.3 تدفق الكتابة مع delay (gpio_fwd_set)

```
gpio_chip.set(chip, offset, value)    ← gpio_fwd_set()
    │
    ├─ gpiod_set_value[_cansleep](fwd->descs[offset], value)
    │
    └─ لو fwd->delay_timings:
           gpio_fwd_delay(chip, offset, value)
               │
               ├─ يقرأ is_active_low
               ├─ يختار ramp_up_us أو ramp_down_us
               └─ fsleep() أو udelay()
```

#### 4.4 تدفق التفعيل (live=1) الكامل

```
write(live_attr, "1")
    │
    ▼
gpio_aggregator_device_live_store()
    │
    ├─ try_module_get(THIS_MODULE)           ← يمنع unload أثناء العمل
    │
    ├─ gpio_aggregator_lockup_configfs()     ← يقفل configfs entries
    │      └─ configfs_depend_item_unlocked() per line
    │
    ├─ [guard mutex aggr->lock]
    │       └─ gpio_aggregator_activate()
    │               │
    │               ├─ gpio_aggregator_make_device_sw_node()
    │               │       └─ fwnode_create_software_node()
    │               │
    │               ├─ gpio_aggregator_add_gpio() × N
    │               │       └─ krealloc(lookups, ...)
    │               │
    │               ├─ gpiod_add_lookup_table()
    │               │
    │               └─ dev_sync_probe_register()
    │                       ├─ bus_notifier registration
    │                       ├─ platform_device_register_full()
    │                       └─ wait_for_completion()    ← ينتظر probe
    │
    └─ module_put(THIS_MODULE)
```

---

### 5. استراتيجية القفل (Locking Strategy)

#### الـ Locks المستخدمة

| Lock | النوع | ما يحميه |
|------|-------|----------|
| `gpio_aggregator_lock` | `DEFINE_MUTEX` (عالمي) | الـ `gpio_aggregator_idr` |
| `aggr->lock` | `struct mutex` (per-aggregator) | `list_head`، `lookups`، `probe_data.pdev`، حالة active/activating |
| `fwd->mlock` | `struct mutex` | `fwd->tmp[]` لو `chip.can_sleep` |
| `fwd->slock` | `spinlock_t` | `fwd->tmp[]` لو `!chip.can_sleep` |
| `configfs_subsys->su_mutex` | داخلي في configfs | شجرة الـ configfs |

#### ترتيب القفل (Lock Ordering)

```
gpio_aggregator_lock  (الأعلى — يؤخذ أولًا)
        │
        ▼
    aggr->lock        (per-aggregator — يؤخذ تانيًا)
        │
        ▼
    fwd->mlock        (per-forwarder — يؤخذ أخيرًا)
   أو fwd->slock
```

#### قواعد مهمة

**الـ `lockdep_assert_held(&aggr->lock)`** موجودة في:
- `gpio_aggregator_is_active()`
- `gpio_aggregator_is_activating()`
- `gpio_aggregator_count_lines()`
- `gpio_aggregator_line_add()`
- `gpio_aggregator_line_del()`

هذا يضمن إن المطوّر ما ينسى يأخذ القفل قبل استدعاء الدوال الداخلية.

**تفصيل حرج:** في `gpio_aggregator_free_lines()` (مسار sysfs)، لا يُمسك `aggr->lock` أثناء `configfs_unregister_group()` لتجنب deadlock مع configfs المتداخل. بدلًا من ذلك، الحماية جاية من `try_module_get()` اللي يضمن أن `new_device` و`delete_device` ما يتشغلوا في نفس الوقت مع الـ module unload.

**الـ `fwd->mlock` vs `fwd->slock`:**
- لو أي GPIO line تتطلب النوم (`gpiod_cansleep()`)، يُفعَّل `chip.can_sleep` ويُستخدم `mutex`
- لو كل الـ lines لا تنام، يُستخدم `spinlock` للأداء في سياق interrupt-safe
- القرار يحدث في `gpiochip_fwd_register()` أو عند `gpiochip_fwd_desc_add()`

#### جدول: ما يتطلب قفل قبل الاستخدام

| Resource | القفل المطلوب |
|----------|---------------|
| `gpio_aggregator_idr` | `gpio_aggregator_lock` |
| `aggr->list_head` | `aggr->lock` |
| `aggr->lookups` | `aggr->lock` |
| `aggr->probe_data.pdev` | `aggr->lock` |
| `line->key` / `line->name` / `line->offset` | `aggr->lock` |
| `fwd->tmp[]` لعمليات multiple | `fwd->mlock` أو `fwd->slock` |
| `fwd->descs[]` (قراءة فقط بعد register) | لا قفل مطلوب |
## Phase 4: شرح الـ Functions

---

### ملخص كل الـ Functions والـ APIs — Cheatsheet

#### Group 1: Aggregator Lifecycle (Internal)

| Function | Signature موجزة | الغرض |
|---|---|---|
| `gpio_aggregator_alloc` | `(aggr**, size_t) → int` | تخصيص struct + IDR slot |
| `gpio_aggregator_free` | `(aggr) → void` | تحرير IDR + kfree |
| `gpio_aggregator_add_gpio` | `(aggr, key, hwnum, n*) → int` | إضافة GPIO entry للـ lookup table |
| `gpio_aggregator_is_active` | `(aggr) → bool` | هل الـ pdev probed فعلاً؟ |
| `gpio_aggregator_is_activating` | `(aggr) → bool` | هل الـ pdev موجود بس لسه ما اتـ probe؟ |
| `gpio_aggregator_count_lines` | `(aggr) → size_t` | عدد الـ lines في الـ list |
| `gpio_aggregator_activate` | `(aggr) → int` | بناء lookup table + تسجيل pdev |
| `gpio_aggregator_deactivate` | `(aggr) → void` | إلغاء pdev + lookup table |
| `gpio_aggregator_destroy` | `(aggr) → void` | deactivate + free lines + configfs unregister |
| `gpio_aggregator_parse` | `(aggr) → int` | تحليل args string (sysfs legacy path) |
| `gpio_aggregator_remove_all` | `(void) → void` | cleanup كل الـ aggregators عند module exit |

#### Group 2: Line Management (Internal)

| Function | Signature موجزة | الغرض |
|---|---|---|
| `gpio_aggregator_line_alloc` | `(parent, idx, key, offset) → line*` | تخصيص gpio_aggregator_line |
| `gpio_aggregator_line_add` | `(aggr, line) → void` | إضافة line للـ list بالترتيب |
| `gpio_aggregator_line_del` | `(aggr, line) → void` | حذف line من الـ list |
| `gpio_aggregator_free_lines` | `(aggr) → void` | حذف كل الـ lines وتحرير ذاكرتها |

#### Group 3: GPIO Forwarder — Public API (Exported)

| Function | Export Symbol | الغرض |
|---|---|---|
| `devm_gpiochip_fwd_alloc` | `GPIO_FORWARDER` | تخصيص gpiochip_fwd + chip setup |
| `gpiochip_fwd_desc_add` | `GPIO_FORWARDER` | ربط gpio_desc بـ offset في الـ forwarder |
| `gpiochip_fwd_desc_free` | `GPIO_FORWARDER` | تحرير gpio_desc من offset معين |
| `gpiochip_fwd_register` | `GPIO_FORWARDER` | تسجيل الـ gpiochip في kernel |
| `gpiochip_fwd_get_gpiochip` | `GPIO_FORWARDER` | جلب الـ gpio_chip pointer |
| `gpiochip_fwd_get_data` | `GPIO_FORWARDER` | جلب driver-private data |
| `gpiochip_fwd_gpio_request` | `GPIO_FORWARDER` | validate إن الـ line موجودة |
| `gpiochip_fwd_gpio_get_direction` | `GPIO_FORWARDER` | قراءة اتجاه الـ line |
| `gpiochip_fwd_gpio_direction_input` | `GPIO_FORWARDER` | تعيين input |
| `gpiochip_fwd_gpio_direction_output` | `GPIO_FORWARDER` | تعيين output |
| `gpiochip_fwd_gpio_get` | `GPIO_FORWARDER` | قراءة قيمة line واحدة |
| `gpiochip_fwd_gpio_get_multiple` | `GPIO_FORWARDER` | قراءة قيم multiple lines |
| `gpiochip_fwd_gpio_set` | `GPIO_FORWARDER` | كتابة قيمة line واحدة |
| `gpiochip_fwd_gpio_set_multiple` | `GPIO_FORWARDER` | كتابة قيم multiple lines |
| `gpiochip_fwd_gpio_set_config` | `GPIO_FORWARDER` | تطبيق pinconf على line |
| `gpiochip_fwd_gpio_to_irq` | `GPIO_FORWARDER` | جلب IRQ number للـ line |

#### Group 4: Forwarder — Internal gpio_chip Callbacks

| Function | الغرض |
|---|---|
| `gpio_fwd_request` | `.request` callback — validate valid_mask |
| `gpio_fwd_get_direction` | `.get_direction` callback |
| `gpio_fwd_direction_input` | `.direction_input` callback |
| `gpio_fwd_direction_output` | `.direction_output` callback |
| `gpio_fwd_get` | `.get` callback |
| `gpio_fwd_get_multiple` | core logic للـ bulk read |
| `gpio_fwd_get_multiple_locked` | `.get_multiple` callback مع lock |
| `gpio_fwd_set` | `.set` callback |
| `gpio_fwd_set_multiple` | core logic للـ bulk write |
| `gpio_fwd_set_multiple_locked` | `.set_multiple` callback مع lock |
| `gpio_fwd_set_config` | `.set_config` callback |
| `gpio_fwd_to_irq` | `.to_irq` callback |
| `gpio_fwd_delay` | تطبيق ramp-up/down delay بعد set |

#### Group 5: Configfs Callbacks

| Function | الغرض |
|---|---|
| `gpio_aggregator_make_group` | إنشاء aggregator جديد عند mkdir في `/config/gpio-aggregator/` |
| `gpio_aggregator_device_make_group` | إنشاء line جديدة عند mkdir في `/config/gpio-aggregator/<name>/` |
| `gpio_aggregator_device_release` | تحرير aggregator عند rmdir |
| `gpio_aggregator_line_release` | تحرير line عند rmdir |
| `gpio_aggregator_device_live_show/store` | قراءة/كتابة الـ `live` attribute |
| `gpio_aggregator_device_dev_name_show` | قراءة الـ `dev_name` attribute |
| `gpio_aggregator_line_key_show/store` | قراءة/كتابة الـ `key` attribute |
| `gpio_aggregator_line_offset_show/store` | قراءة/كتابة الـ `offset` attribute |
| `gpio_aggregator_line_name_show/store` | قراءة/كتابة الـ `name` attribute |
| `gpio_aggregator_lockup_configfs` | lock/unlock configfs entries بـ configfs_depend |

#### Group 6: Sysfs Legacy Interface

| Function | الغرض |
|---|---|
| `gpio_aggregator_new_device_store` | معالجة الكتابة على `new_device` |
| `gpio_aggregator_delete_device_store` | معالجة الكتابة على `delete_device` |

#### Group 7: Platform Driver

| Function | الغرض |
|---|---|
| `gpio_aggregator_probe` | probe callback — يجلب GPIO descs وينشئ forwarder |
| `gpio_aggregator_init` | module init |
| `gpio_aggregator_exit` | module exit |
| `gpio_aggregator_idr_remove` | idr_for_each callback عند module exit |
| `gpio_aggregator_make_device_sw_node` | بناء software fwnode بـ gpio-line-names property |
| `gpiochip_fwd_create` | static helper — alloc + add descs + setup delay + register |
| `gpiochip_fwd_setup_delay_line` | إعداد delay timing للـ OF case |
| `gpiochip_fwd_delay_of_xlate` | custom of_xlate لدعم 3-cell GPIO specifier (line, ramp_up_us, ramp_down_us) |

---

### Group 1: Aggregator Lifecycle (Internal)

هذه الـ functions مسؤولة عن إنشاء وتدمير الـ `gpio_aggregator` struct اللي هو الـ core object في الـ driver. كل aggregator ليه IDR slot عشان يتعرف عليه بالـ id عند الـ delete_device. الـ lifecycle ده مقسوم على ثلاث مراحل: alloc، activate، deactivate/destroy.

---

#### `gpio_aggregator_alloc`

```c
static int gpio_aggregator_alloc(struct gpio_aggregator **aggr, size_t arg_size)
```

بتخصص `struct gpio_aggregator` من الـ heap بحجم `sizeof(*aggr) + arg_size` (الـ flexible array member `args[]` بتخزن فيه الـ sysfs args string). بعدين بتحجز IDR slot بـ `idr_alloc` تحت `gpio_aggregator_lock`. لو النجاح، بترجع الـ id في `aggr->id` وبتعمل initialize للـ list و mutex.

**Parameters:**
- `aggr`: double pointer، الـ function بتكتب فيه الـ allocated pointer
- `arg_size`: حجم الـ flexible array للـ args string (0 للـ configfs path)

**Return:** 0 نجاح، negative errno فشل (ENOMEM أو idr_alloc error)

**Key details:** بتستخدم `scoped_guard(mutex, &gpio_aggregator_lock)` للـ IDR access. الـ `no_free_ptr` idiom بيضمن إن الـ `__free(kfree)` cleanup attribute ما يتشغلش لو نجحت. الـ IDR protected بـ global `gpio_aggregator_lock`.

**Caller:** `gpio_aggregator_new_device_store` و `gpio_aggregator_make_group`

---

#### `gpio_aggregator_free`

```c
static void gpio_aggregator_free(struct gpio_aggregator *aggr)
```

بتحذف الـ aggregator من الـ IDR وبتعمل `mutex_destroy` + `kfree`. دي الـ counterpart لـ `gpio_aggregator_alloc`.

**Parameters:**
- `aggr`: الـ aggregator المراد تحريره

**Key details:** بتحتاج `gpio_aggregator_lock` للـ IDR remove. لازم تتأكد إن الـ lines اتحذفت الأول قبل ما تطلع `aggr` نفسه.

**Caller:** `gpio_aggregator_device_release` (configfs release callback) و `gpio_aggregator_destroy`

---

#### `gpio_aggregator_add_gpio`

```c
static int gpio_aggregator_add_gpio(struct gpio_aggregator *aggr,
                                    const char *key, int hwnum, unsigned int *n)
```

بتضيف entry واحد في الـ `gpiod_lookup_table` الخاصة بالـ aggregator. بتعمل `krealloc` للـ table عشان تستوعب entry إضافي، وبتستخدم الـ macro `GPIO_LOOKUP_IDX(key, hwnum, NULL, *n, 0)` لملء الـ entry. الـ `*n` بيتزاد كل call وبيستخدمه الـ index للـ consumer.

**Parameters:**
- `aggr`: الـ aggregator اللي هيتضاف ليه الـ GPIO
- `key`: إما chip label أو line name (حسب الـ hwnum)
- `hwnum`: offset داخل الـ chip، أو `U16_MAX` للـ lookup بالاسم
- `n`: pointer للـ counter (بيتزاد in-place)

**Return:** 0 نجاح، `-ENOMEM` لو `krealloc` فشل

**Key details:** الـ entry الأخير في الـ table دايماً zeroed (sentinel). الـ U16_MAX هو convention الـ gpiolib للـ line-name lookup.

**Caller:** `gpio_aggregator_parse` و `gpio_aggregator_activate`

---

#### `gpio_aggregator_is_active`

```c
static bool gpio_aggregator_is_active(struct gpio_aggregator *aggr)
```

بترجع true لو الـ platform device موجود **وعمل probe بنجاح** (يعني `platform_get_drvdata` بيرجع non-NULL). ده بيميز بين حالة "الـ pdev اتسجل بس لسه ما اتـ probe" وحالة "اتـ probe فعلاً".

**Key details:** `lockdep_assert_held(&aggr->lock)` — لازم تـ call دي وإنت شايل `aggr->lock`.

---

#### `gpio_aggregator_is_activating`

```c
static bool gpio_aggregator_is_activating(struct gpio_aggregator *aggr)
```

بترجع true لو الـ pdev موجود بس الـ `platform_get_drvdata` بيرجع NULL — يعني الـ probe لسه ما اكتملش (أو الـ deferred probe). دي حالة transient خاصة بالـ legacy sysfs path.

**Key details:** `lockdep_assert_held(&aggr->lock)` مطلوب. الـ configfs path بيستخدم `dev_sync_probe_register` اللي بيـ wait synchronously لحد ما الـ probe يكمل.

---

#### `gpio_aggregator_activate`

```c
static int gpio_aggregator_activate(struct gpio_aggregator *aggr)
```

ده الـ core function اللي بيحول الـ aggregator من "configured" لـ "live". بيمر على الخطوات دي:

```
1. تحقق من وجود lines (error لو 0 lines)
2. kzalloc lookup table (entry واحد للـ sentinel أول ما)
3. إنشاء software fwnode بـ gpio-line-names property
4. بناء platform_device_info (اسم DRV_NAME + aggr->id + fwnode)
5. loop على list_head: gpio_aggregator_add_gpio لكل line
6. kasprintf لـ lookups->dev_id = "gpio-aggregator.N"
7. gpiod_add_lookup_table → تسجيل الـ table في الـ kernel
8. dev_sync_probe_register → تسجيل pdev وانتظار probe
```

**Parameters:**
- `aggr`: الـ aggregator المراد تفعيله

**Return:** 0 نجاح، negative errno لأي خطوة فشلت (مع cleanup كامل عبر goto chain)

**Key details:** الـ cleanup متسلسل بشكل دقيق عبر labels: `err_remove_lookup_table` → `err_remove_swnode` → `err_remove_lookups`. الـ `dev_sync_probe_register` بيـ block لحد ما الـ probe يكمل (للـ configfs path). الـ list دايماً sorted فالـ line indices بتكون sequential.

**Caller:** `gpio_aggregator_device_live_store` عند كتابة `1`

---

#### `gpio_aggregator_deactivate`

```c
static void gpio_aggregator_deactivate(struct gpio_aggregator *aggr)
```

عكس الـ activate: بتـ call `dev_sync_probe_unregister` (بتحذف الـ pdev)، بعدين `gpiod_remove_lookup_table`، وبعدين بتحرر الـ dev_id string والـ lookup table struct.

**Key details:** الترتيب مهم جداً — لازم تحذف الـ pdev الأول قبل ما تحذف الـ lookup table، عشان لو في driver شايل reference على الـ table ما يـ crash.

**Caller:** `gpio_aggregator_destroy` و `gpio_aggregator_device_live_store` عند كتابة `0`

---

#### `gpio_aggregator_parse`

```c
static int gpio_aggregator_parse(struct gpio_aggregator *aggr)
```

بتحلل الـ `aggr->args` string اللي جت من `new_device` sysfs write. البروتوكول:

```
"[<line_name> | <chip_label> <offset_list>] ..."
```

**Pseudocode flow:**
```
args = skip_spaces(aggr->args)
key = next token
while tokens remain:
    offsets = next token
    if offsets is NOT a number list:
        // named GPIO line
        line_alloc(key, offset=-1)
        add_gpio(key, U16_MAX)
        key = offsets
    else:
        // chip + offset list
        bitmap_parselist(offsets)
        for each set bit i:
            line_alloc(key, i)
            add_gpio(key, i)
    key = next token
```

الـ function بتستخدم `get_options` لتحديد إذا كان الـ token ده أرقام أم اسم line. لكل line بتعمل `configfs_register_group` عشان يظهر في الـ configfs تلقائياً (legacy sysfs entries).

**Key details:** تحت `_sysfs.N` configfs subtree. لو حصل error في أي نقطة، `gpio_aggregator_free_lines` بتنظف كل اللي اتعمل.

---

### Group 2: Line Management (Internal)

الـ `gpio_aggregator_line` هو الـ object اللي بيمثل GPIO line واحدة داخل الـ aggregator. الـ list دايماً sorted بـ `idx` تصاعدياً.

---

#### `gpio_aggregator_line_alloc`

```c
static struct gpio_aggregator_line *
gpio_aggregator_line_alloc(struct gpio_aggregator *parent, unsigned int idx,
                           char *key, int offset)
```

بتخصص `gpio_aggregator_line` وبتعمل `kstrdup` للـ key لو موجود. الـ flags بتتأخذ الـ default value `GPIO_LOOKUP_FLAGS_DEFAULT`.

**Parameters:**
- `parent`: الـ aggregator الأب
- `idx`: رقم الـ line داخل الـ aggregator (الـ virtual offset)
- `key`: chip label أو line name (nullable)
- `offset`: hardware offset داخل الـ chip، أو -1 للـ name lookup

**Return:** pointer أو ERR_PTR(-ENOMEM)

---

#### `gpio_aggregator_line_add`

```c
static void gpio_aggregator_line_add(struct gpio_aggregator *aggr,
                                     struct gpio_aggregator_line *line)
```

بتضيف الـ line في المكان الصح في الـ sorted list حسب `idx`. بتمشي على الـ list وبتعمل `list_add_tail` قبل أول element أكبر من `line->idx`. لو ما فيش، بتضيفها في الآخر.

**Key details:** `lockdep_assert_held(&aggr->lock)` مطلوب.

---

#### `gpio_aggregator_free_lines`

```c
static void gpio_aggregator_free_lines(struct gpio_aggregator *aggr)
```

بتمشي على الـ list بـ `list_for_each_entry_safe` وبتعمل:
1. `configfs_unregister_group` لكل line
2. `gpio_aggregator_line_del` تحت `aggr->lock` (scoped_guard صغير)
3. `kfree` للـ key والـ name والـ line نفسه

**Key details:** في الـ legacy sysfs path، لازم ما تشيلش `aggr->lock` وإنت بتـ call `configfs_unregister_group` عشان ممكن deadlock. الـ comment في الكود بيوضح إن الـ `new_device/delete_device` path والـ module unload path متحصرش مع بعض بسبب `try_module_get`.

---

### Group 3 & 4: GPIO Forwarder

الـ **GPIO Forwarder** هو الـ virtual `gpio_chip` اللي بيـ forward كل operations للـ descriptors الحقيقية. ده الجزء اللي بيظهر لـ userspace كـ `/dev/gpiochipN` جديد.

#### الـ `gpiochip_fwd` struct

```c
struct gpiochip_fwd {
    struct gpio_chip chip;        // embedded chip
    struct gpio_desc **descs;     // array of real descriptors
    union {
        struct mutex mlock;       // lock for sleeping chips
        spinlock_t slock;         // lock for non-sleeping chips
    };
    struct gpiochip_fwd_timing *delay_timings;  // per-line delays (optional)
    void *data;                   // driver-private data
    unsigned long *valid_mask;    // bitmap of valid (registered) lines
    unsigned long tmp[];          // scratch space for bulk ops
};
```

الـ `tmp[]` array ده flexible array بيخدم غرضين: أول `BITS_TO_LONGS(ngpios)` longs هي الـ values bitmap، والباقي هو array من `gpio_desc*` pointers للـ bulk ops. الـ macros `fwd_tmp_values` و `fwd_tmp_descs` بيحسبوا offsets الاتنين.

---

#### `devm_gpiochip_fwd_alloc`

```c
struct gpiochip_fwd *devm_gpiochip_fwd_alloc(struct device *dev,
                                              unsigned int ngpios)
```

بتخصص `gpiochip_fwd` بحجم يشمل الـ tmp[] scratch space. بتعمل `devm_kcalloc` للـ descs array و `devm_bitmap_zalloc` للـ valid_mask. بعدين بتملي كل الـ `gpio_chip` callbacks. الـ `chip.base = -1` يعني الـ kernel هيختار الـ base تلقائياً.

**Parameters:**
- `dev`: الـ parent device (الـ platform device)
- `ngpios`: عدد الـ lines في الـ forwarder

**Return:** `gpiochip_fwd*` أو ERR_PTR(-ENOMEM)

**Key details:** كل allocations بـ `devm_*` فهي بتتحرر تلقائياً عند device removal. الـ lock type (mutex vs spinlock) بيتحدد بعدين في `gpiochip_fwd_register` حسب `can_sleep`.

**Caller:** `gpiochip_fwd_create` (static) أو أي external driver عبر الـ header

---

#### `gpiochip_fwd_desc_add`

```c
int gpiochip_fwd_desc_add(struct gpiochip_fwd *fwd, struct gpio_desc *desc,
                           unsigned int offset)
```

بتربط `gpio_desc` حقيقي بـ virtual offset في الـ forwarder. بتعمل `test_and_set_bit` في `valid_mask` — لو الـ bit كان set بالفعل بترجع `-EEXIST`. لو الـ desc بيحتاج sleep (slow GPIO controller) بتعمل `chip->can_sleep = true` عشان يتعكس على الـ locking strategy.

**Parameters:**
- `fwd`: الـ forwarder
- `desc`: الـ real GPIO descriptor
- `offset`: الـ virtual offset داخل الـ forwarder

**Return:** 0 نجاح، `-EINVAL` لو offset خارج الـ range، `-EEXIST` لو مسجل قبل كده

**Key details:** بعد ما تخلص من `gpiochip_fwd_desc_add` لكل الـ lines، لازم تـ call `gpiochip_fwd_register`. ممكن تضيف lines بعد التسجيل للـ runtime hot-add (valid_mask بيحدد availability).

---

#### `gpiochip_fwd_desc_free`

```c
void gpiochip_fwd_desc_free(struct gpiochip_fwd *fwd, unsigned int offset)
```

بتحذف descriptor من offset معين: بتعمل `test_and_clear_bit` في `valid_mask`، وبتـ call `gpiod_put` على الـ desc لو كان valid.

---

#### `gpiochip_fwd_register`

```c
int gpiochip_fwd_register(struct gpiochip_fwd *fwd, void *data)
```

بتكمل setup الـ forwarder وبتسجله في الـ GPIO subsystem. لو مش كل الـ lines مسجلة (valid_mask مش full)، بتـ force `can_sleep = true` عشان الـ runtime additions ممكن تكون sleeping. بعدين بتعمل `mutex_init` أو `spin_lock_init` حسب `can_sleep`. الـ `data` بتتخزن في `fwd->data`. وأخيراً `devm_gpiochip_add_data`.

**Parameters:**
- `fwd`: الـ forwarder اللي اتجهز
- `data`: driver-private data (nullable)

**Return:** 0 أو error من `devm_gpiochip_add_data`

**Caller:** `gpiochip_fwd_create` و أي external driver

---

#### `gpiochip_fwd_create` (static)

```c
static struct gpiochip_fwd *gpiochip_fwd_create(struct device *dev,
                                                 unsigned int ngpios,
                                                 struct gpio_desc *descs[],
                                                 unsigned long features)
```

Helper داخلي بيجمع الثلاث خطوات: alloc + add all descs + setup delay (لو `FWD_FEATURE_DELAY` set) + register. ده اللي بيـ call `gpio_aggregator_probe` مباشرة.

**Parameters:**
- `features`: bitfield — حالياً `FWD_FEATURE_DELAY` (BIT(0)) هو الوحيد المعرف، للـ `gpio-delay` compatible

---

#### `gpio_fwd_get` و `gpio_fwd_set`

```c
static int gpio_fwd_get(struct gpio_chip *chip, unsigned int offset)
static int gpio_fwd_set(struct gpio_chip *chip, unsigned int offset, int value)
```

**الـ `gpio_fwd_get`:** بتختار `gpiod_get_value_cansleep` أو `gpiod_get_value` حسب `chip->can_sleep`. بترجع الـ logical value (بتأخذ ACTIVE_LOW في الحسبان).

**الـ `gpio_fwd_set`:** نفس المنطق للكتابة. بعد الكتابة الناجحة، لو في `fwd->delay_timings` بتـ call `gpio_fwd_delay` لتطبيق الـ ramp timing.

---

#### `gpio_fwd_get_multiple_locked` و `gpio_fwd_set_multiple_locked`

```c
static int gpio_fwd_get_multiple_locked(struct gpio_chip *chip,
                                         unsigned long *mask, unsigned long *bits)
static int gpio_fwd_set_multiple_locked(struct gpio_chip *chip,
                                         unsigned long *mask, unsigned long *bits)
```

دول الـ wrappers اللي بتـ acquire الـ lock المناسب (mutex لو can_sleep، spinlock + irqsave لو !can_sleep) وبعدين بيـ delegate للـ unlocked versions.

**الـ `gpio_fwd_get_multiple` core logic:**
```
1. bitmap_clear(values)
2. for each set bit i in mask:
       descs[j++] = fwd->descs[i]   // collect real descs
3. gpiod_get_array_value[_cansleep](j, descs, NULL, values)
4. for each set bit i in mask:
       assign bit i in bits = bit j++ from values
```

بيستخدم الـ `fwd->tmp[]` scratch space عشان ما يعملش allocation داخل الـ lock.

**الـ `gpio_fwd_set_multiple` core logic:**
```
1. for each set bit i in mask:
       values[j] = bit i from bits
       descs[j++] = fwd->descs[i]
2. gpiod_set_array_value[_cansleep](j, descs, NULL, values)
```

---

#### `gpio_fwd_delay`

```c
static void gpio_fwd_delay(struct gpio_chip *chip, unsigned int offset, int value)
```

بتحسب إذا كانت العملية ده ramp-up أو ramp-down (بتأخذ ACTIVE_LOW في الحسبان) وبتـ sleep المناسب (`fsleep` لو can_sleep، `udelay` لو !can_sleep). مفيدة لـ RC filter open-drain outputs.

---

#### `gpiochip_fwd_delay_of_xlate` (CONFIG_OF_GPIO only)

```c
static int gpiochip_fwd_delay_of_xlate(struct gpio_chip *chip,
                                        const struct of_phandle_args *gpiospec,
                                        u32 *flags)
```

بتـ override الـ `of_xlate` callback لدعم 3-cell GPIO specifier: `<&chip line ramp_up_us ramp_down_us>`. بتخزن الـ timings في `fwd->delay_timings[line]` وبترجع رقم الـ line.

**Key details:** الـ `chip->of_gpio_n_cells = 3` بيتعمل set في `gpiochip_fwd_setup_delay_line`. ده ضروري عشان الـ OF layer يعرف عدد cells في الـ specifier.

---

### Group 5: Configfs Interface

الـ configfs interface هي الـ primary interface الحديثة للـ gpio-aggregator. الـ hierarchy هي:

```
/config/gpio-aggregator/          ← gpio_aggregator_type
    agg0/                         ← gpio_aggregator_device_type
        live                      ← CONFIGFS_ATTR (rw)
        dev_name                  ← CONFIGFS_ATTR_RO
        line0/                    ← gpio_aggregator_line_type
            key                   ← CONFIGFS_ATTR (rw)
            offset                ← CONFIGFS_ATTR (rw)
            name                  ← CONFIGFS_ATTR (rw)
        line1/
            ...
```

---

#### `gpio_aggregator_make_group`

```c
static struct config_group *
gpio_aggregator_make_group(struct config_group *group, const char *name)
```

**الـ `.make_group` callback لجذر الـ subsystem.** بيتـ call عند `mkdir /config/gpio-aggregator/<name>`. برفض الأسماء اللي بتبدأ بـ `_sysfs` (محجوزة). بيعمل `gpio_aggregator_alloc` + `config_group_init_type_name` + `dev_sync_probe_init`.

**Return:** `&aggr->group` أو ERR_PTR

---

#### `gpio_aggregator_device_make_group`

```c
static struct config_group *
gpio_aggregator_device_make_group(struct config_group *group, const char *name)
```

**الـ `.make_group` callback لكل aggregator device.** بيتـ call عند `mkdir /config/gpio-aggregator/<name>/lineN`. بيـ validate إن الاسم بالضبط `line<N>` وإن الـ index مش متكرر. الـ aggregators المنشأة بـ sysfs بتـ return `-EPERM`. لو الـ aggregator active بترجع `-EBUSY`.

**Key details:** بيـ acquire `aggr->lock` عشان يتحقق من الـ state وأثناء إضافة الـ line.

---

#### `gpio_aggregator_device_live_store`

```c
static ssize_t
gpio_aggregator_device_live_store(struct config_item *item, const char *page,
                                   size_t count)
```

ده الـ entry point الرئيسي لتفعيل/تعطيل الـ aggregator. بيـ parse الـ boolean، وبـ `try_module_get` يمنع module unload أثناء العملية.

**Flow:**
```
1. kstrtobool(page, &live)
2. try_module_get(THIS_MODULE)
3. if live && !init_via_sysfs:
       gpio_aggregator_lockup_configfs(aggr, true)   // lock configfs entries
4. acquire aggr->lock:
       if activating or (live == current_state): ret = -EPERM
       elif live: ret = gpio_aggregator_activate(aggr)
       else: gpio_aggregator_deactivate(aggr)
5. if failed: gpio_aggregator_lockup_configfs(aggr, false)  // unlock
6. module_put(THIS_MODULE)
```

**Key details:** الـ `gpio_aggregator_lockup_configfs` بيستخدم `configfs_depend_item_unlocked` عشان يمنع rmdir على الـ line directories وإنت عايز الـ device يكون live. ده بيحمي من race condition بين الـ device teardown و directory removal.

---

#### `gpio_aggregator_lockup_configfs`

```c
static void gpio_aggregator_lockup_configfs(struct gpio_aggregator *aggr, bool lock)
```

بتـ iterate على كل الـ lines وبتـ call `configfs_depend_item_unlocked` (lock) أو `configfs_undepend_item_unlocked` (unlock) لكل واحدة. الـ "depend" mechanism في configfs بيمنع rmdir على الـ items المعتمد عليها.

**Key details:** بتـ call "unlocked" variant لأنها بتتـ call من outside الـ configfs mutex context.

---

#### Line Attribute Callbacks (key, offset, name)

الثلاثة callbacks دول بيشتركوا في نفس الـ pattern:

```c
// show: return current value
guard(mutex)(&aggr->lock);
return sysfs_emit(page, ...);

// store: parse + validate + set
guard(mutex)(&aggr->lock);
if (is_activating || is_active) return -EBUSY;
// set the field
return count;
```

**الـ `offset_store` validation:**
- `offset == -1`: lookup by line name
- `0 <= offset <= 65534`: lookup by chip + offset (U16_MAX محجوز للـ name lookup في gpiolib)
- أي قيمة تانية: `-EINVAL`

---

#### `gpio_aggregator_make_device_sw_node`

```c
static struct fwnode_handle *
gpio_aggregator_make_device_sw_node(struct gpio_aggregator *aggr)
```

بتنشئ software node (بديل عن DT node) بـ property واحدة: `gpio-line-names`. بتبني array من strings من الـ `line->name` لكل line (empty string لو name == NULL). بتستخدم `fwnode_create_software_node`.

**Return:** `fwnode_handle*` أو NULL لو 0 lines أو ERR_PTR(-ENOMEM)

**Key details:** الـ fwnode ده بيتـ attach للـ platform_device_info وبيتـ release بعدين في error path بـ `fwnode_remove_software_node`. الـ kernel GPIO subsystem بيقرأ الـ `gpio-line-names` property لتسمية الـ virtual lines في الـ gpiochip.

---

### Group 6: Sysfs Legacy Interface

---

#### `gpio_aggregator_new_device_store`

```c
static ssize_t gpio_aggregator_new_device_store(struct device_driver *driver,
                                                 const char *buf, size_t count)
```

**الـ write handler لـ `/sys/bus/platform/drivers/gpio-aggregator/new_device`.** ده الـ legacy interface الأبسط. بيـ parse الـ format `"chip_label offsets ..."` مباشرة.

**Flow:**
```
1. try_module_get
2. gpio_aggregator_alloc(aggr, count+1)    // extra space for args
3. memcpy(aggr->args, buf, count+1)
4. aggr->init_via_sysfs = true
5. alloc lookup table + dev_id
6. config_group_init → configfs_register_group (under _sysfs.N/)
7. gpio_aggregator_parse(aggr)            // parse args → lines + lookup entries
8. gpiod_add_lookup_table
9. platform_device_register_data          // fire probe immediately (async)
10. aggr->probe_data.pdev = pdev
```

**الفرق عن الـ configfs path:** هنا الـ probe بيكون async (لو فيه deferred probe ممكن يتأجل). الـ configfs path بيستخدم `dev_sync_probe_register` اللي بيـ wait synchronously.

**Error path:** cleanup كامل بترتيب عكسي عبر goto labels.

---

#### `gpio_aggregator_delete_device_store`

```c
static ssize_t gpio_aggregator_delete_device_store(struct device_driver *driver,
                                                    const char *buf, size_t count)
```

بتحذف aggregator موجود. بتـ parse الاسم `gpio-aggregator.N`، بتـ extract الـ id، وبتبحث عنه في الـ IDR. بتعمل `idr_remove` أول ما بيأكد إن الـ aggregator موجود وإنه `init_via_sysfs` (ما تنحذفش aggregators created via configfs من هنا).

**Key details:** الـ `idr_remove` بيتعمل تحت `gpio_aggregator_lock` بشكل explicit (مش `scoped_guard`) عشان يـ release الـ lock قبل ما يـ call `gpio_aggregator_destroy` اللي بتـ call configfs operations.

---

#### `gpio_aggregator_destroy`

```c
static void gpio_aggregator_destroy(struct gpio_aggregator *aggr)
```

بتنفذ الـ full teardown لـ sysfs-created aggregators:
1. لو active أو activating: `gpio_aggregator_deactivate`
2. `gpio_aggregator_free_lines`
3. `configfs_unregister_group`
4. `kfree(aggr)`

**Key details:** مختلفة عن `gpio_aggregator_free` — دي بتعمل كل حاجة، أما `gpio_aggregator_free` بتعمل بس IDR remove + kfree. الـ `gpio_aggregator_device_release` configfs callback بتـ call `gpio_aggregator_free` مباشرة عشان الـ configfs path بيكون شيل الـ lines والـ deactivation بالفعل.

---

### Group 7: Platform Driver

---

#### `gpio_aggregator_probe`

```c
static int gpio_aggregator_probe(struct platform_device *pdev)
```

ده الـ probe callback اللي بيتـ call بعد ما الـ platform device يتسجل. ده المكان اللي بيتحول فيه الـ lookup table لـ GPIO descriptors حقيقية وبيتنشأ الـ virtual gpiochip.

**Flow:**
```
1. n = gpiod_count(dev, NULL)        // كم GPIO في الـ lookup table؟
2. alloc descs[] array
3. meta = dev_get_platdata → check init_via_sysfs flag
4. for i in 0..n:
       descs[i] = devm_gpiod_get_index(dev, NULL, i, GPIOD_ASIS)
       // لو configfs-created + deferred probe: convert EPROBE_DEFER → ENODEV
5. features = device_get_match_data(dev)  // FWD_FEATURE_DELAY من DT
6. fwd = gpiochip_fwd_create(dev, n, descs, features)
7. platform_set_drvdata(pdev, fwd)        // يخلي is_active يرجع true
8. devm_kfree(dev, descs)                 // الـ fwd عنده copy خاصته
```

**Key details:** الـ `GPIOD_ASIS` بيعني ما يغيرش الـ direction. الـ deferred probe handling مختلف حسب الـ creation path:
- **sysfs path:** بيـ propagate EPROBE_DEFER عادي (backward compatibility)
- **configfs path:** بيـ convert EPROBE_DEFER لـ ENODEV عشان `dev_sync_probe_register` بيعمل treat خاص للـ ENODEV
- **DT path (device_of_node):** بيـ propagate EPROBE_DEFER عادي

**Caller:** kernel platform bus عند تسجيل الـ device

---

#### `gpio_aggregator_init`

```c
static int __init gpio_aggregator_init(void)
```

**Module init.** الترتيب مهم:
1. `config_group_init` + `mutex_init` للـ configfs subsystem
2. `configfs_register_subsystem` — لازم يكون أول
3. `platform_driver_register` — لازم يكون **بعد** الـ configfs عشان لو user كتب على `new_device` قبل ما configfs يجهز هيـ crash

**Caller:** `module_init`

---

#### `gpio_aggregator_exit`

```c
static void __exit gpio_aggregator_exit(void)
```

**Module exit.** الترتيب:
1. `gpio_aggregator_remove_all` — بيدمر كل الـ sysfs-created aggregators
2. `platform_driver_unregister`
3. `configfs_unregister_subsystem`

---

#### `gpio_aggregator_remove_all`

```c
static void __exit gpio_aggregator_remove_all(void)
```

بتـ call `idr_for_each` مع `gpio_aggregator_idr_remove` callback على كل entry. **لا تستخدم `gpio_aggregator_lock`** هنا عمداً — الـ comment في الكود بيشرح إن الـ `new_device/delete_device` path والـ module unload path متحصرش مع بعض بسبب `try_module_get`، فمفيش race. استخدام الـ lock هنا كان هيعمل deadlock لأن `gpio_aggregator_idr_remove` بتـ call configfs ops اللي ممكن تـ acquire نفس الـ lock.

**Key details:** الـ configfs-created aggregators مش موجودين في الـ IDR لما يصل للـ module exit عشان وجودهم كان هيمنع unloading (configfs dependency).

---

### ملاحظات عامة على الـ Locking

```
gpio_aggregator_lock (global mutex)
    ↳ يحمي gpio_aggregator_idr
    ↳ لا يتداخل مع aggr->lock

aggr->lock (per-aggregator mutex)
    ↳ يحمي aggr->list_head, probe_data state
    ↳ لازم تشيله قبل أي configfs operation لتجنب deadlock

fwd->mlock (mutex) OR fwd->slock (spinlock)
    ↳ يحمي fwd->tmp[] scratch area أثناء bulk GPIO ops
    ↳ يتحدد النوع في gpiochip_fwd_register حسب can_sleep
```

**قاعدة الـ lock ordering:** لا تـ acquire `gpio_aggregator_lock` وإنت شايل `aggr->lock`، ولا تـ call configfs functions وإنت شايل أي منهم.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. مدخلات الـ debugfs

الـ GPIO subsystem مش بيعمل expose مباشر لـ debugfs entries خاصة بالـ aggregator، بس الـ GPIO core بيوفر:

```bash
# Mount debugfs لو مش موجود
mount -t debugfs debugfs /sys/kernel/debug

# اقرأ حالة كل GPIO lines في النظام
cat /sys/kernel/debug/gpio
```

**تفسير الـ output:**

```
gpiochip12: GPIOs 496-497, parent: platform/gpio-aggregator.0, gpio-aggregator.0:
 gpio-496 (open             ): input  hi IRQ
 gpio-497 (lock             ): output lo
```

- السطر الأول: اسم الـ chip، نطاق الـ GPIO numbers، والـ parent device
- كل سطر تاني: رقم الـ GPIO + اسم الـ line + الاتجاه (input/output) + القيمة (hi/lo) + وجود IRQ

```bash
# شوف الـ gpiochip_fwd logging (dev_dbg messages)
cat /sys/kernel/debug/gpio | grep aggregator
```

---

#### 2. مدخلات الـ sysfs

| المسار | الوصف | كيف تقراه |
|---|---|---|
| `/sys/bus/platform/drivers/gpio-aggregator/` | الـ driver directory بتاع الـ aggregator | `ls` لتشوف الـ attributes |
| `/sys/bus/platform/drivers/gpio-aggregator/new_device` | write-only لإنشاء aggregator جديد | اكتب فيه بس |
| `/sys/bus/platform/drivers/gpio-aggregator/delete_device` | write-only لحذف aggregator | اكتب فيه بس |
| `/sys/bus/platform/devices/gpio-aggregator.N/` | الـ platform device directory | `ls` لشوف الـ attributes |
| `/sys/bus/platform/devices/gpio-aggregator.N/gpiochipX/` | الـ gpiochip المنشأ | `ls` للتفاصيل |
| `/sys/bus/platform/devices/gpio-aggregator.N/gpiochipX/ngpio` | عدد الـ lines | `cat` |
| `/sys/bus/platform/devices/gpio-aggregator.N/gpiochipX/label` | label الـ chip | `cat` |
| `/sys/kernel/config/gpio-aggregator/` | الـ configfs root | `ls` لشوف الـ aggregators |
| `/sys/kernel/config/gpio-aggregator/<name>/live` | حالة الـ aggregator | `cat` (0 أو 1) |
| `/sys/kernel/config/gpio-aggregator/<name>/dev_name` | اسم الـ device على الـ platform bus | `cat` |
| `/sys/kernel/config/gpio-aggregator/<name>/<lineY>/key` | chip label أو line name | `cat` |
| `/sys/kernel/config/gpio-aggregator/<name>/<lineY>/offset` | offset داخل الـ chip | `cat` |

```bash
# تحقق من كل الـ aggregators الموجودة
ls /sys/kernel/config/gpio-aggregator/

# اقرأ حالة aggregator معين
cat /sys/kernel/config/gpio-aggregator/agg0/live
cat /sys/kernel/config/gpio-aggregator/agg0/dev_name

# اقرأ تفاصيل line معين
cat /sys/kernel/config/gpio-aggregator/agg0/line0/key
cat /sys/kernel/config/gpio-aggregator/agg0/line0/offset

# تحقق من الـ gpiochip المنشأ
ls /sys/devices/platform/gpio-aggregator.0/
cat /sys/devices/platform/gpio-aggregator.0/gpiochip*/label
cat /sys/devices/platform/gpio-aggregator.0/gpiochip*/ngpio
```

---

#### 3. الـ ftrace — Tracepoints والـ Events

الـ GPIO subsystem عنده tracepoints جاهزة:

```bash
# شوف الـ GPIO events المتاحة
ls /sys/kernel/debug/tracing/events/gpio/

# فعّل كل GPIO events
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable

# فعّل events محددة
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_direction/enable
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_value/enable

# ابدأ الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# اعمل الـ operation اللي عايز تتبعها
gpioset gpio-aggregator.0 0=1

# اقرأ الـ trace
cat /sys/kernel/debug/tracing/trace

# وقف الـ tracing
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo > /sys/kernel/debug/tracing/trace  # امسح الـ buffer
```

**تتبع الـ probe function بالـ function tracer:**

```bash
# تتبع الـ gpio_aggregator_probe
echo gpio_aggregator_probe > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# أعد تحميل الـ module أو أنشئ aggregator جديد
echo 'gpiochip0 0-1' > /sys/bus/platform/drivers/gpio-aggregator/new_device

cat /sys/kernel/debug/tracing/trace
```

**تتبع الـ forwarder calls:**

```bash
# تتبع كل functions في الـ gpio_fwd_* namespace
echo 'gpio_fwd_*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

---

#### 4. الـ printk والـ dynamic debug

الـ driver بيستخدم `pr_fmt` مع prefix `"gpio-aggregator: "` وبيستخدم `dev_dbg()` في `gpiochip_fwd_desc_add()`.

```bash
# فعّل الـ dynamic debug لملف الـ driver
echo 'file gpio-aggregator.c +p' > /sys/kernel/debug/dynamic_debug/control

# أو فعّل لكل الـ gpio subsystem
echo 'module gpio_aggregator +p' > /sys/kernel/debug/dynamic_debug/control

# تحقق من الـ dynamic debug المفعّلة
cat /sys/kernel/debug/dynamic_debug/control | grep gpio-aggregator

# رفع مستوى الـ printk مؤقتاً
echo 8 > /proc/sys/kernel/printk

# شوف الـ kernel messages في real-time
dmesg -w | grep gpio-aggregator

# أو ابحث في الـ dmesg العادي
dmesg | grep -i gpio-aggregator
dmesg | grep -i 'gpio-aggregator\|gpiochip_fwd\|gpio_fwd'
```

**مثال على الـ dev_dbg output عند إضافة GPIO descriptor:**

```
[  123.456789] gpio-aggregator gpio-aggregator.0: 0 => gpio 496 irq -1
[  123.456800] gpio-aggregator gpio-aggregator.0: 1 => gpio 497 irq 42
```

---

#### 5. الـ Kernel Config Options للـ Debugging

| الـ Config Option | الغرض |
|---|---|
| `CONFIG_GPIO_AGGREGATOR` | تفعيل الـ driver نفسه (مطلوب) |
| `CONFIG_DEBUG_GPIO` | تفعيل debug messages في الـ GPIO core |
| `CONFIG_GPIO_SYSFS` | الـ sysfs interface للـ GPIO |
| `CONFIG_GPIOLIB_IRQCHIP` | دعم الـ IRQ للـ GPIO |
| `CONFIG_OF_GPIO` | دعم الـ Device Tree للـ GPIO |
| `CONFIG_DEBUG_FS` | لازم لـ debugfs |
| `CONFIG_TRACING` | لازم لـ ftrace |
| `CONFIG_DYNAMIC_DEBUG` | الـ dynamic debug |
| `CONFIG_LOCKDEP` | كشف مشاكل الـ locking (الـ driver بيستخدم `lockdep_assert_held`) |
| `CONFIG_DEBUG_MUTEXES` | debug الـ mutex |
| `CONFIG_DEBUG_SPINLOCK` | debug الـ spinlock |
| `CONFIG_PROVE_LOCKING` | تحقق من صحة الـ locking order |
| `CONFIG_CONFIGFS_FS` | لازم للـ configfs interface |

```bash
# تحقق من الـ configs المفعلة في الـ kernel الحالي
zcat /proc/config.gz | grep -E 'GPIO_AGGREGATOR|DEBUG_GPIO|CONFIGFS'
# أو
cat /boot/config-$(uname -r) | grep -E 'GPIO_AGGREGATOR|DEBUG_GPIO'
```

---

#### 6. أدوات الـ gpiod userspace

```bash
# تثبيت gpio tools
apt install gpiod   # Debian/Ubuntu
dnf install libgpiod-utils  # Fedora

# قائمة كل الـ gpiochips بما فيها الـ aggregated ones
gpiodetect

# معلومات تفصيلية عن aggregated chip
gpioinfo gpio-aggregator.0

# قراءة قيمة line من الـ aggregator
gpioget gpio-aggregator.0 0

# كتابة قيمة
gpioset gpio-aggregator.0 0=1

# مراقبة الـ GPIO events
gpiomon gpio-aggregator.0 0

# تحقق من الـ driver binding
cat /sys/bus/platform/devices/gpio-aggregator.0/driver
ls -la /sys/bus/platform/devices/gpio-aggregator.0/driver
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الخطأ | المعنى | الحل |
|---|---|---|
| `gpio-aggregator: No GPIOs specified` | كتبت string فاضية في `new_device` | تحقق من format الـ string: `<chip> <offsets>` |
| `gpio-aggregator: Cannot parse <offsets>: -22` | الـ offset string غير صحيح | استخدم format صحيح: أرقام أو ranges (`0-3`) |
| `gpio-aggregator: Failed to register the 'gpio-aggregator' configfs subsystem: <N>` | فشل تسجيل الـ configfs | تحقق من `CONFIG_CONFIGFS_FS` وأن الـ configfs مـ mount |
| `gpio-aggregator: Failed to register the platform driver: <N>` | فشل تسجيل الـ driver | غالباً conflict في اسم الـ driver |
| `gpio-aggregator: Deferred probe canceled for creation via configfs.` | GPIO غير متاح بعد وتم إلغاء الـ deferred probe | أعد `echo 1 > live` لاحقاً بعد ما الـ GPIO يبقى متاح |
| `write error: Device or resource busy` عند الكتابة في `key/offset/name` | الـ aggregator شغّال (live=1) | اعمل `echo 0 > live` الأول |
| `write error: Operation not permitted` عند `echo 1 > live` | الـ aggregator already active أو في حالة activating | تحقق من `cat live` |
| `write error: No such file or directory` عند `delete_device` | اسم الـ device غلط أو أُنشئ عبر configfs | استخدم الاسم الصحيح وتأكد إنه أُنشئ عبر sysfs |
| `write error: Invalid argument` عند `mkdir line<N>` | اسم الـ directory مش بالصيغة `lineN` | استخدم `line0`, `line1`, ... بالترتيب |
| `-EPROBE_DEFER` في الـ probe | الـ GPIO المطلوب مش متاح لسه | طبيعي، الـ kernel هيعيد المحاولة تلقائياً |
| `-ENODEV` في `gpio_fwd_request` | الـ line offset خارج الـ `valid_mask` | تحقق من الـ offset الصحيح في الـ lookup table |

---

#### 8. نقاط استراتيجية للـ dump_stack() والـ WARN_ON()

نقاط مناسبة للـ instrumentation في الـ code:

```c
/* في gpio_aggregator_activate() - بعد فشل gpiod_add_lookup_table */
static int gpio_aggregator_activate(struct gpio_aggregator *aggr)
{
    /* ... */
    gpiod_add_lookup_table(aggr->lookups);

    ret = dev_sync_probe_register(&aggr->probe_data, &pdevinfo);
    if (ret) {
        /* هنا ممكن تضيف WARN_ON لتتبع فشل الـ probe */
        WARN_ON(ret == -ENODEV);
        dump_stack();  /* لو عايز stack trace كامل */
        goto err_remove_lookup_table;
    }
    /* ... */
}

/* في gpio_aggregator_probe() - عند فشل devm_gpiod_get_index */
for (i = 0; i < n; i++) {
    descs[i] = devm_gpiod_get_index(dev, NULL, i, GPIOD_ASIS);
    if (IS_ERR(descs[i])) {
        /* WARN_ON لو مش EPROBE_DEFER - خطأ غير متوقع */
        WARN_ON(descs[i] != ERR_PTR(-EPROBE_DEFER) &&
                descs[i] != ERR_PTR(-ENODEV));
        return PTR_ERR(descs[i]);
    }
}

/* في gpiochip_fwd_desc_add() - تحقق من valid_mask */
int gpiochip_fwd_desc_add(struct gpiochip_fwd *fwd, struct gpio_desc *desc,
                          unsigned int offset)
{
    /* WARN_ON لو offset خارج النطاق - indicates bug في الـ caller */
    if (WARN_ON(offset >= chip->ngpio))
        return -EINVAL;

    if (test_and_set_bit(offset, fwd->valid_mask)) {
        /* WARN_ON لو نفس الـ offset اتضاف مرتين */
        WARN_ON(1);
        return -EEXIST;
    }
    /* ... */
}
```

---

### Hardware Level

#### 1. التحقق من أن حالة الـ Hardware تطابق حالة الـ Kernel

```bash
# الخطوة 1: اعرف الـ GPIO numbers الفيزيائية
cat /sys/kernel/debug/gpio | grep -A5 'gpio-aggregator'

# مثال output:
# gpiochip12: GPIOs 496-497, parent: platform/gpio-aggregator.0
#  gpio-496 (open): input  hi
#  gpio-497 (lock): output lo

# الخطوة 2: اعرف الـ chip الأصلية اللي بتعيد التوجيه ليها
# لو aggregator.0 بيضم gpiochip0 offset 19 و gpiochip1 offset 20-21
gpioinfo gpiochip0   # شوف الـ line 19
gpioinfo gpiochip1   # شوف الـ lines 20-21

# الخطوة 3: قارن القيم
gpioget gpio-aggregator.0 0   # القيمة عبر الـ aggregator
gpioget gpiochip0 19           # القيمة المباشرة من الـ chip الأصلية

# لازم يطلعوا نفس القيمة - لو مختلفين فيه مشكلة في الـ polarity أو الـ flags
```

---

#### 2. تقنيات الـ Register Dump

```bash
# استخدام devmem2 لقراءة registers الـ GPIO controller الأصلي
# أولاً: اعرف base address الـ GPIO controller من الـ DT أو /proc/iomem
cat /proc/iomem | grep -i gpio

# مثال: Renesas R-Car GPIO
# e6052000-e60520ff : e6052000.gpio

# اقرأ registers (مثال: data register على offset 0x04)
devmem2 0xe6052004 w

# باستخدام /dev/mem مباشرة (يحتاج CONFIG_DEVMEM=y)
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    m = mmap.mmap(f.fileno(), 0x100, offset=0xe6052000)
    # قرأ INDT register (input data) على offset 0x04
    val = struct.unpack('<I', m[0x04:0x08])[0]
    print(f'GPIO Data Register: 0x{val:08x}')
    m.close()
"

# استخدام io utility (من package iotools)
io -4 -r 0xe6052004   # قراءة 32-bit register
```

---

#### 3. نصائح الـ Logic Analyzer والـ Oscilloscope

**إعداد الـ Logic Analyzer:**

```
Channel 0 → الـ GPIO line المطلوب (مثلاً: open)
Channel 1 → الـ GPIO line التاني (مثلاً: lock)
Channel 2 → IRQ line لو موجود
Trigger: Rising edge على Channel 0

Sample rate: على الأقل 10x أعلى تردد متوقع
Buffer: كبير بما يكفي لتتقاط الـ event
```

**ربط الـ software events بالـ hardware:**

```bash
# استخدم gpio_set ثم راقب على الـ oscilloscope
# قبل ما تضغط enter شغّل الـ capture على الـ scope

# أضف gpio toggle لتحديد نقطة الـ trigger في الـ time
gpioset gpio-aggregator.0 0=1  # ارفع الـ line
sleep 0.001
gpioset gpio-aggregator.0 0=0  # انزّل الـ line
```

**فحص الـ timing delays في حالة `gpio-delay` compatible:**

```
الـ ramp_up_us و ramp_down_us بيتحكموا في الـ delay
افحص على الـ scope إن الـ delay بيطابق القيمة في الـ DT
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماط الـ Kernel Log

| المشكلة الفيزيائية | نمط الـ Kernel Log | التشخيص |
|---|---|---|
| الـ GPIO chip مش موجود | `gpio-aggregator: Cannot parse <chip_label>` أو `-ENODEV` | تحقق من `gpiodetect` إن الـ chip موجودة |
| الـ GPIO line مشغول (requested بواسطة driver تاني) | `gpio-aggregator gpio-aggregator.0: ... -EBUSY` | شوف `cat /sys/kernel/debug/gpio` مين شاغل الـ line |
| خطأ في الـ polarity | الـ GPIO بيقرأ عكس المتوقع | تحقق من `GPIO_ACTIVE_LOW` في الـ DT أو الـ lookup flags |
| الـ GPIO line مش بتتعرف بالاسم | `-ENOENT` عند الـ probe | تحقق من الـ line name الصحيح في `gpioinfo` |
| مشكلة في الـ timing (open-drain مع RC filter) | الـ output مش بيصل للمستوى المطلوب في الوقت المحدد | زوّد `ramp_up_us` في الـ DT |
| الـ GPIO controller مش initialized بعد | `-EPROBE_DEFER` | طبيعي، لكن لو استمر: تحقق من الـ initcall order |

---

#### 5. تحقق من الـ Device Tree

```bash
# اقرأ الـ compiled DT من الـ kernel
cat /proc/device-tree/door/compatible 2>/dev/null
# أو
ls /proc/device-tree/ | grep -i gpio

# استخدام dtc لتفكيك الـ DTB الحالي
dtc -I dtb -O dts /sys/firmware/fdt 2>/dev/null | grep -A20 'gpio-aggregator\|gpio-delay'

# تحقق من الـ DT للـ GPIO controllers الأصلية
dtc -I dtb -O dts /sys/firmware/fdt 2>/dev/null | grep -B2 -A15 'gpio-controller'

# مثال DT لـ generic GPIO device مع الـ aggregator
# door {
#     compatible = "myvendor,mydoor";
#     gpios = <&gpio2 19 GPIO_ACTIVE_HIGH>,
#             <&gpio2 20 GPIO_ACTIVE_LOW>;
#     gpio-line-names = "open", "lock";
# };

# تحقق من إن الـ gpios property متطابقة مع الـ hardware schematic
# GPIO_ACTIVE_HIGH = 0, GPIO_ACTIVE_LOW = 1 في الـ DT

# تحقق من الـ software node المنشأ ديناميكياً
ls /sys/kernel/debug/supplemental_firmware_data/ 2>/dev/null
# أو عبر الـ device properties
cat /sys/devices/platform/gpio-aggregator.0/of_node/gpio-line-names 2>/dev/null
```

---

### Practical Commands

#### مجموعة Commands جاهزة للنسخ

**-- فحص الحالة الأساسية --**

```bash
#!/bin/bash
# gpio-aggregator-debug.sh - Script شامل للـ debugging

echo "=== Loaded GPIO Aggregator Modules ==="
lsmod | grep gpio

echo -e "\n=== GPIO Aggregator Driver Status ==="
ls /sys/bus/platform/drivers/gpio-aggregator/ 2>/dev/null || echo "Driver not loaded"

echo -e "\n=== All Aggregated Chips ==="
gpiodetect 2>/dev/null | grep aggregator

echo -e "\n=== Configfs State ==="
if mountpoint -q /sys/kernel/config; then
    find /sys/kernel/config/gpio-aggregator/ -maxdepth 3 \
        -name 'live' -o -name 'key' -o -name 'offset' -o -name 'dev_name' 2>/dev/null \
        | while read f; do
            echo "$f = $(cat $f 2>/dev/null)"
        done
else
    echo "configfs not mounted"
fi

echo -e "\n=== GPIO Debug Info ==="
cat /sys/kernel/debug/gpio 2>/dev/null | grep -A10 aggregator

echo -e "\n=== Recent Kernel Messages ==="
dmesg | grep -i gpio-aggregator | tail -20
```

**-- إنشاء aggregator عبر sysfs والتحقق منه --**

```bash
# إنشاء aggregator يضم line 19 من gpiochip0 و lines 20-21 من gpiochip1
echo 'gpiochip0 19 gpiochip1 20-21' > \
    /sys/bus/platform/drivers/gpio-aggregator/new_device

# تحقق من الـ output
dmesg | tail -5
# Expected:
# gpio-aggregator gpio-aggregator.0: 0 => gpio 496 irq -1
# gpio-aggregator gpio-aggregator.0: 1 => gpio 497 irq -1
# gpio-aggregator gpio-aggregator.0: 2 => gpio 498 irq 42

# اعرف اسم الـ gpiochip المنشأ
ls /sys/devices/platform/gpio-aggregator.0/gpiochip*

# تفاصيل الـ lines
gpioinfo gpio-aggregator.0
# Expected:
# gpiochip12 - 3 lines:
#   line   0:      unnamed   unused  input  active-high
#   line   1:      unnamed   unused  input  active-high
#   line   2:      unnamed   unused  input  active-high
```

**-- إنشاء aggregator عبر configfs --**

```bash
# mount configfs لو مش mounted
mount -t configfs configfs /sys/kernel/config

# إنشاء aggregator
mkdir /sys/kernel/config/gpio-aggregator/myagg

# إضافة line0
mkdir /sys/kernel/config/gpio-aggregator/myagg/line0
echo "gpiochip0" > /sys/kernel/config/gpio-aggregator/myagg/line0/key
echo "19"        > /sys/kernel/config/gpio-aggregator/myagg/line0/offset
echo "my-signal" > /sys/kernel/config/gpio-aggregator/myagg/line0/name

# تحقق من الـ config
echo "key:    $(cat /sys/kernel/config/gpio-aggregator/myagg/line0/key)"
echo "offset: $(cat /sys/kernel/config/gpio-aggregator/myagg/line0/offset)"
echo "name:   $(cat /sys/kernel/config/gpio-aggregator/myagg/line0/name)"

# تفعيل الـ aggregator
echo 1 > /sys/kernel/config/gpio-aggregator/myagg/live

# تحقق من النجاح
cat /sys/kernel/config/gpio-aggregator/myagg/live      # يطلع 1
cat /sys/kernel/config/gpio-aggregator/myagg/dev_name  # يطلع gpio-aggregator.N

# تعطيل وحذف
echo 0 > /sys/kernel/config/gpio-aggregator/myagg/live
rmdir /sys/kernel/config/gpio-aggregator/myagg/line0
rmdir /sys/kernel/config/gpio-aggregator/myagg
```

**-- تفعيل الـ Dynamic Debug وقراءة النتائج --**

```bash
# فعّل كل debug messages للـ module
echo 'module gpio_aggregator +pmfl' > /sys/kernel/debug/dynamic_debug/control

# +p = print message
# +m = include module name
# +f = include function name
# +l = include line number

# تحقق من التفعيل
grep gpio_aggregator /sys/kernel/debug/dynamic_debug/control

# أنشئ aggregator وشوف الـ messages
dmesg -C  # امسح الـ buffer
echo 'gpiochip0 0' > /sys/bus/platform/drivers/gpio-aggregator/new_device
dmesg
# Expected output:
# [  456.123] gpio_aggregator gpio-aggregator.0: gpio-aggregator.c:763 [gpio_aggregator] gpiochip_fwd_desc_add: 0 => gpio 0 irq -1

# عطّل الـ debug بعد الانتهاء
echo 'module gpio_aggregator -p' > /sys/kernel/debug/dynamic_debug/control
```

**-- تتبع الـ GPIO operations بالـ ftrace --**

```bash
# إعداد شامل للـ ftrace
cd /sys/kernel/debug/tracing

# صفّي الـ filter
echo > set_ftrace_filter

# اختار الـ functions المطلوبة
echo 'gpio_aggregator_probe
gpio_aggregator_activate
gpio_aggregator_deactivate
gpiochip_fwd_create
gpio_fwd_get
gpio_fwd_set' > set_ftrace_filter

# استخدم function_graph لترى الـ call hierarchy
echo function_graph > current_tracer
echo 1 > tracing_on

# اعمل الـ operation
gpioget gpio-aggregator.0 0

# اقرأ الـ trace
cat trace

# مثال الـ output:
# # tracer: function_graph
# #
#  0)               |  gpio_fwd_get() {
#  0)   1.234 us    |    gpiod_get_value();
#  0)   2.567 us    |  }

# نظّف
echo nop > current_tracer
echo > set_ftrace_filter
echo 0 > tracing_on
```

**-- التحقق من الـ driver binding لـ DT device --**

```bash
# Bind device يدوياً للـ gpio-aggregator driver
DEVICE="door"  # اسم الـ device في الـ DT

echo gpio-aggregator > /sys/bus/platform/devices/${DEVICE}/driver_override
echo ${DEVICE} > /sys/bus/platform/drivers/gpio-aggregator/bind

# تحقق من الـ binding
ls -la /sys/bus/platform/devices/${DEVICE}/driver
# Expected: .../driver -> ../../../../bus/platform/drivers/gpio-aggregator

# شوف الـ gpiochip المنشأ
gpioinfo ${DEVICE}

# فك الـ binding
echo ${DEVICE} > /sys/bus/platform/drivers/gpio-aggregator/unbind
echo > /sys/bus/platform/devices/${DEVICE}/driver_override
```

**-- فحص الـ lockdep للـ mutex issues --**

```bash
# فعّل الـ lockdep reporting
echo 1 > /proc/sys/kernel/lock_stat 2>/dev/null || true

# اعمل operations على الـ aggregator
echo 'gpiochip0 0-3' > /sys/bus/platform/drivers/gpio-aggregator/new_device
cat /sys/kernel/config/gpio-aggregator/_sysfs.0/live
gpioget gpio-aggregator.0 0

# شوف لو فيه lock warnings
dmesg | grep -E 'possible circular|lock order|deadlock'

# شوف إحصائيات الـ locks
cat /proc/lock_stat | grep gpio
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Industrial Gateway على AM62x — GPIO لـ relay control بدون driver مخصص

#### العنوان
تشغيل relay module على AM62x gateway صناعي باستخدام gpio-aggregator كـ generic driver

#### السياق
مهندس embedded بيشتغل على industrial IoT gateway مبني على **Texas Instruments AM62x**. الـ gateway بيتحكم في 4 relays متوصلين على `gpiochip2` (GPIO bank 2). الـ product team طلبوا إن الـ relay control يكون accessible من userspace process بـ limited permissions، من غير ما أي process تاني يقدر يوصله.

#### المشكلة
الـ `/dev/gpiochip2` بيكشف كل الـ GPIOs في bank 2 (48 line)، ومش ممكن تحدد access لـ subset بس. الـ process اللي بتتحكم في الـ relays بتشتغل under user `relay-ctrl`، لكن لو أديتها access لـ `/dev/gpiochip2` هتقدر تلمس كل الـ GPIOs في الـ bank.

#### التحليل

**الـ gpio-aggregator** بيحل المشكلة دي بالظبط. بدل ما تدي access لـ gpiochip2 كله، بتعمل virtual gpiochip جديد بيضم بس الـ 4 lines المطلوبة.

الـ flow داخل الكود:

```c
// في gpio_aggregator_new_device_store():
// لما تكتب في new_device، بيتعمل lookup table
lookups->table[*n] = GPIO_LOOKUP_IDX(key, hwnum, NULL, *n, 0);
// key = "gpiochip2", hwnum = offset الـ GPIO الحقيقي
```

بعدين في `gpio_aggregator_probe()`:
```c
n = gpiod_count(dev, NULL);  // بيعد الـ 4 GPIOs المسجلين في lookup table
for (i = 0; i < n; i++) {
    descs[i] = devm_gpiod_get_index(dev, NULL, i, GPIOD_ASIS);
    // بيجيب descriptor لكل GPIO من gpiochip2
}
fwd = gpiochip_fwd_create(dev, n, descs, features);
// بيعمل virtual chip بـ 4 lines بس
```

#### الحل

```bash
# الـ relays موجودين على gpiochip2، offsets 10, 11, 12, 13
$ echo 'gpiochip2 10-13' > \
    /sys/bus/platform/drivers/gpio-aggregator/new_device

# الـ output هيبين إن aggregator اتعمل
# مثلاً: gpio-aggregator.0 → /dev/gpiochip5

# تغيير ownership للـ virtual chip بس
$ chown root:relay-ctrl /dev/gpiochip5
$ chmod 660 /dev/gpiochip5

# verify
$ gpioinfo gpio-aggregator.0
gpiochip5 - 4 lines:
    line   0:      unnamed       unused   output  active-high
    line   1:      unnamed       unused   output  active-high
    line   2:      unnamed       unused   output  active-high
    line   3:      unnamed       unused   output  active-high
```

لو عايز persistent configuration، استخدم configfs:

```bash
$ mkdir /sys/kernel/config/gpio-aggregator/relay-ctrl
$ for i in 0 1 2 3; do
    mkdir /sys/kernel/config/gpio-aggregator/relay-ctrl/line${i}
    echo gpiochip2 > /sys/kernel/config/gpio-aggregator/relay-ctrl/line${i}/key
    echo $((10 + i)) > /sys/kernel/config/gpio-aggregator/relay-ctrl/line${i}/offset
    echo "relay${i}" > /sys/kernel/config/gpio-aggregator/relay-ctrl/line${i}/name
  done
$ echo 1 > /sys/kernel/config/gpio-aggregator/relay-ctrl/live
```

#### الدرس المستفاد
**الـ gpio-aggregator بيوفر access control دقيق على مستوى الـ GPIO line** بدون ما تحتاج تكتب driver. الـ `gpiochip_fwd_create()` بيعمل virtual chip بـ subset بس من الـ lines، وبكده تقدر تطبق Unix permissions عليه زي أي device file.

---

### السيناريو الثاني: Android TV Box على Allwinner H616 — HDMI CEC GPIO بـ DT binding

#### العنوان
ربط GPIO-operated HDMI CEC circuit على Allwinner H616 بالـ gpio-aggregator من الـ Device Tree بدون driver مخصص

#### السياق
مهندس kernel بيعمل bring-up لـ Android TV box مبني على **Allwinner H616**. الـ board فيها external CEC transceiver chip بيتحكم فيه عن طريق GPIO واحد للـ TX وتاني للـ RX. مفيش in-kernel driver للـ chip ده، والـ team قررت تتحكم فيه من userspace.

#### المشكلة
لما بيحاول يعمل bind للـ device من الـ DT للـ gpio-aggregator باستخدام `driver_override`، الـ device مش بيتعرف عليه لأن الـ compatible value مش موجود في `gpio_aggregator_dt_ids[]`. وحتى لو استخدم `driver_override`، الـ gpiochip بيتعمل بس بيظهر بـ offset عشوائي وبدون أسماء للـ lines.

#### التحليل

في الكود:
```c
// gpio_aggregator_dt_ids[] موجود في آخر الـ driver
static const struct of_device_id gpio_aggregator_dt_ids[] = {
    {
        .compatible = "gpio-delay",
        .data = (void *)FWD_FEATURE_DELAY,
    },
    /*
     * Add GPIO-operated devices controlled from userspace below,
     * or use "driver_override" in sysfs.
     */
    {}
};
```

**الـ compatible** للـ CEC chip مش موجود. الـ `gpio_aggregator_probe()` بيقرأ الـ GPIO descriptors من الـ DT:

```c
n = gpiod_count(dev, NULL);  // عدد الـ GPIOs في الـ DT node
for (i = 0; i < n; i++) {
    descs[i] = devm_gpiod_get_index(dev, NULL, i, GPIOD_ASIS);
    // بيجيب كل GPIO من الـ gpios property في DT
}
```

الـ `gpio-line-names` property في الـ DT بتتحول لـ software node بواسطة `gpio_aggregator_make_device_sw_node()` اللي بيبني `fwnode` بـ `gpio-line-names` property.

#### الحل

**الـ DT node:**
```dts
/* في board DTS */
cec_transceiver: cec-gpio {
    compatible = "myvendor,h616-cec-gpio";

    gpios = <&pio 2 5 GPIO_ACTIVE_HIGH>,   /* PC5: CEC TX */
            <&pio 2 6 GPIO_ACTIVE_HIGH>;    /* PC6: CEC RX */
    gpio-line-names = "cec-tx", "cec-rx";
};
```

**الـ bind من userspace (بدون تعديل الـ kernel):**
```bash
$ echo gpio-aggregator > \
    /sys/bus/platform/devices/cec-gpio/driver_override
$ echo cec-gpio > \
    /sys/bus/platform/drivers/gpio-aggregator/bind

# verify
$ gpioinfo cec-gpio
gpiochip8 - 2 lines:
    line   0:    "cec-tx"       unused   input  active-high
    line   1:    "cec-rx"       unused   input  active-high
```

**لو عايز permanent binding، تضيف الـ compatible في الـ driver:**
```c
// في gpio_aggregator_dt_ids[]
{ .compatible = "myvendor,h616-cec-gpio", },
```

#### الدرس المستفاد
**الـ gpio-aggregator بيشتغل كـ generic driver لأي GPIO-operated device في الـ DT** باستخدام `driver_override`. الـ `gpio-line-names` في الـ DT بيتعكس على الـ virtual gpiochip مباشرة. الـ compatible value اللازم يتضاف لـ `gpio_aggregator_dt_ids[]` بس لو عايز automatic binding.

---

### السيناريو الثالث: RK3562 IoT Sensor Board — Probe Defer مع GPIO Expander

#### العنوان
حل مشكلة `-EPROBE_DEFER` في gpio-aggregator لما بيحاول يوصل لـ GPIO على I2C expander على RK3562

#### السياق
مهندس بيشتغل على IoT sensor board مبني على **Rockchip RK3562**. الـ board فيها I2C GPIO expander (PCA9555) بيوفر 16 GPIO إضافي. المهندس عامل gpio-aggregator عن طريق sysfs يضم GPIO من الـ expander وGPIO من الـ SoC في virtual chip واحد.

#### المشكلة
لما بيعمل:
```bash
$ echo 'gpiochip0 5 pca9555 3' > \
    /sys/bus/platform/drivers/gpio-aggregator/new_device
```
بيرجعله error وفي `dmesg` بيلاقي:
```
gpio-aggregator gpio-aggregator.0: error -517 getting GPIO at index 1
```
الـ `-517` هو `-EPROBE_DEFER`، يعني الـ PCA9555 driver لسه مش loaded أو الـ I2C bus لسه مش جاهز.

#### التحليل

في `gpio_aggregator_probe()`:
```c
for (i = 0; i < n; i++) {
    descs[i] = devm_gpiod_get_index(dev, NULL, i, GPIOD_ASIS);
    if (IS_ERR(descs[i])) {
        /*
         * Deferred probing is not suitable when the aggregator
         * is created via configfs. They should just retry later
         * whenever they like. For device creation via sysfs,
         * error is propagated without overriding for backward
         * compatibility.
         */
        if (!init_via_sysfs && !dev_of_node(dev) &&
            descs[i] == ERR_PTR(-EPROBE_DEFER)) {
            pr_warn("Deferred probe canceled for creation via configfs.\n");
            return -ENODEV;
        }
        return PTR_ERR(descs[i]);  // بيرجع -EPROBE_DEFER للـ sysfs
    }
}
```

**الفرق المهم:** لما بيتعمل عن طريق **sysfs** (`init_via_sysfs = true`)، الـ `-EPROBE_DEFER` بيتبعت زي ما هو، يعني الـ kernel ممكن يعيد probe تلقائياً. لكن لما بيتعمل عن طريق **configfs**، الـ kernel بيحول الـ `-EPROBE_DEFER` لـ `-ENODEV` لأن الـ configfs مش بيدعم deferred probing.

#### الحل

**الحل الصح: انتظر الـ I2C expander يتعرف أول.**

```bash
# اتحقق إن PCA9555 موجود
$ ls /sys/bus/i2c/devices/
1-0020/  # PCA9555 على I2C-1 address 0x20

# اتحقق إن driver loaded
$ cat /sys/bus/i2c/devices/1-0020/driver
../../../../../../bus/i2c/drivers/pca953x

# اتحقق إن الـ gpiochip موجود
$ gpiodetect | grep pca
gpiochip3 [pca9555] (16 lines)

# دلوقتي عمل الـ aggregator
$ echo 'gpiochip0 5 gpiochip3 3' > \
    /sys/bus/platform/drivers/gpio-aggregator/new_device
```

**أو استخدم configfs مع retry:**
```bash
$ mkdir /sys/kernel/config/gpio-aggregator/sensor-agg
$ mkdir /sys/kernel/config/gpio-aggregator/sensor-agg/line0
$ echo gpiochip0 > .../line0/key
$ echo 5 > .../line0/offset
$ mkdir /sys/kernel/config/gpio-aggregator/sensor-agg/line1
$ echo gpiochip3 > .../line1/key
$ echo 3 > .../line1/offset

# لو PCA9555 مش ready هيرجع error وتقدر تعيد المحاولة
$ echo 1 > /sys/kernel/config/gpio-aggregator/sensor-agg/live
# لو فشل، انتظر وعيد:
$ echo 1 > /sys/kernel/config/gpio-aggregator/sensor-agg/live
```

#### الدرس المستفاد
**الـ gpio-aggregator بيعالج الـ probe deferral بشكل مختلف حسب طريقة الإنشاء.** عبر sysfs، الـ `-EPROBE_DEFER` بيتمرر والـ kernel ممكن يعيد المحاولة. عبر configfs، بيترفض deferred probe ويرجع `-ENODEV` لأن الـ user المفروض يتحكم في timing. دايماً تأكد إن كل الـ GPIO controllers المطلوبة ready قبل ما تعمل الـ aggregator.

---

### السيناريو الرابع: STM32MP1 Automotive ECU — RC Filter Open-Drain GPIO بـ gpio-delay

#### العنوان
تحكم في open-drain output بـ RC filter على STM32MP1 ECU باستخدام gpio-delay DT compatible

#### السياق
مهندس بيعمل bring-up لـ automotive ECU مبني على **STM32MP157**. الـ ECU فيها output لـ LIN bus transceiver enable line موصل عن طريق open-drain GPIO مع RC filter خارجي (10kΩ pull-up + 100nF cap). الـ RC filter بيعمل ramp-up time حوالي 500µs لما الـ line ترتفع، ولازم الـ software ينتظر قبل ما يبدأ يبعت على الـ LIN bus.

#### المشكلة
الـ GPIO driver العادي بيضرب الـ value فوراً من غير ما ينتظر الـ RC charging. الـ LIN transceiver بيفشل في بداية الـ transmission لأن الـ enable line لسه مش وصلت الـ threshold.

#### التحليل

الـ gpio-aggregator بيدعم `gpio-delay` compatible اللي بيضيف `FWD_FEATURE_DELAY`:

```c
static const struct of_device_id gpio_aggregator_dt_ids[] = {
    {
        .compatible = "gpio-delay",
        .data = (void *)FWD_FEATURE_DELAY,  // feature flag
    },
    {}
};
```

في `gpiochip_fwd_setup_delay_line()`:
```c
static int gpiochip_fwd_setup_delay_line(struct gpiochip_fwd *fwd)
{
    struct gpio_chip *chip = &fwd->chip;

    fwd->delay_timings = devm_kcalloc(chip->parent, chip->ngpio,
                      sizeof(*fwd->delay_timings),
                      GFP_KERNEL);

    chip->of_xlate = gpiochip_fwd_delay_of_xlate;
    chip->of_gpio_n_cells = 3;  // 3 cells: line, ramp_up_us, ramp_down_us
    return 0;
}
```

الـ `gpio_fwd_set()` بعد ما يضرب الـ value بيشوف لو في delay:
```c
static int gpio_fwd_set(struct gpio_chip *chip, unsigned int offset, int value)
{
    struct gpiochip_fwd *fwd = gpiochip_get_data(chip);
    int ret;

    if (chip->can_sleep)
        ret = gpiod_set_value_cansleep(fwd->descs[offset], value);
    else
        ret = gpiod_set_value(fwd->descs[offset], value);
    if (ret)
        return ret;

    if (fwd->delay_timings)
        gpio_fwd_delay(chip, offset, value);  // الانتظار هنا

    return ret;
}
```

```c
static void gpio_fwd_delay(struct gpio_chip *chip, unsigned int offset, int value)
{
    // بيحسب هل ramp_up_us ولا ramp_down_us حسب الـ active_low flag
    if ((!is_active_low && value) || (is_active_low && !value))
        delay_us = delay_timings->ramp_up_us;
    else
        delay_us = delay_timings->ramp_down_us;

    if (chip->can_sleep)
        fsleep(delay_us);  // accurate sleep
    else
        udelay(delay_us);  // busy-wait في interrupt context
}
```

#### الحل

**الـ DT node للـ gpio-delay:**
```dts
lin_enable: lin-enable-gpio {
    compatible = "gpio-delay";

    /*
     * args: <gpio_offset ramp_up_us ramp_down_us>
     * 500µs ramp-up (RC charging), 100µs ramp-down
     */
    gpios = <&gpioa 8 GPIO_ACTIVE_HIGH 500 100>;
    gpio-line-names = "lin-enable";
};
```

```bash
# بعد boot، verify إن الـ device اتعمل
$ gpioinfo lin-enable
gpiochip6 - 1 lines:
    line   0: "lin-enable"  unused  output  active-high

# لو عايز تتحكم فيه من userspace
$ gpioset lin-enable lin-enable=1
# الـ driver هيضرب الـ GPIO وينتظر 500µs تلقائياً قبل ما يرجع
```

**debugging الـ timing:**
```bash
# اتحقق من الـ timings المحددة
$ cat /sys/kernel/debug/gpio | grep lin-enable
# استخدم oscilloscope على الـ LIN enable pin لتأكيد الـ ramp
```

#### الدرس المستفاد
**الـ `gpio-delay` compatible بيوفر hardware timing compensation مدمج في الـ gpio-aggregator** بدون ما تعدل أي driver. الـ 3-cell GPIO specifier `<gpio ramp_up_us ramp_down_us>` بيتحدد per-line، وده مفيد جداً في الـ RC filter circuits والـ open-drain outputs اللي محتاجة charge time.

---

### السيناريو الخامس: i.MX8M — تأمين GPIO لـ Virtual Machine باستخدام gpio-aggregator

#### العنوان
تصدير GPIO subset آمن لـ VM على i.MX8M بدون ما تكشف كامل الـ gpiochip

#### السياق
مهندس virtualization بيشتغل على **NXP i.MX8M Plus** platform بتشغل KVM hypervisor. الـ guest VM محتاجة تتحكم في 3 status LEDs موصولين على `gpiochip1` offsets 20, 21, 22. لو أديت الـ VM الـ `/dev/gpiochip1` كامل، هتقدر تلمس كل الـ GPIOs في الـ bank، وفيهم GPIOs حساسة زي reset lines.

#### المشكلة
مفيش طريقة native في Linux تحدد access لـ subset من GPIO chip لـ VM. الـ VFIO GPIO support غير متاح على الـ platform ده. المطلوب إن الـ VM تشوف gpiochip بـ 3 lines بس.

#### التحليل

الـ gpio-aggregator بيحل المشكلة بالـ workflow ده:

```
Host kernel
    ↓
gpiochip1 (imx-gpio): 32 lines (offsets 0-31)
    ↓
gpio-aggregator: يضم offsets 20, 21, 22 في virtual chip
    ↓
gpiochip7 (gpio-aggregator.0): 3 lines فقط
    ↓
VFIO passthrough أو virtio-gpio → Guest VM
    ↓
Guest يشوف gpiochip0: 3 lines (led0, led1, led2)
```

الـ aggregator بيستخدم `gpiochip_fwd_create()` اللي بيعمل `gpio_chip` جديد بـ `ngpio = 3` بس، وكل الـ operations بتتوجه لـ `fwd->descs[]` اللي بيشير للـ descriptors الحقيقية في `gpiochip1`.

```c
// في gpiochip_fwd_desc_add():
int gpiochip_fwd_desc_add(struct gpiochip_fwd *fwd, struct gpio_desc *desc,
              unsigned int offset)
{
    // ...
    if (gpiod_cansleep(desc))
        chip->can_sleep = true;  // i.MX GPIO: false عادةً

    fwd->descs[offset] = desc;  // mapping من virtual offset → real desc
    return 0;
}
```

#### الحل

**على الـ Host:**
```bash
# عمل الـ aggregator بـ configfs (أكتر تحكم)
$ mkdir /sys/kernel/config/gpio-aggregator/vm-leds

$ mkdir /sys/kernel/config/gpio-aggregator/vm-leds/line0
$ echo gpiochip1 > .../vm-leds/line0/key
$ echo 20 > .../vm-leds/line0/offset
$ echo led0 > .../vm-leds/line0/name

$ mkdir /sys/kernel/config/gpio-aggregator/vm-leds/line1
$ echo gpiochip1 > .../vm-leds/line1/key
$ echo 21 > .../vm-leds/line1/offset
$ echo led1 > .../vm-leds/line1/name

$ mkdir /sys/kernel/config/gpio-aggregator/vm-leds/line2
$ echo gpiochip1 > .../vm-leds/line2/key
$ echo 22 > .../vm-leds/line2/offset
$ echo led2 > .../vm-leds/line2/name

$ echo 1 > /sys/kernel/config/gpio-aggregator/vm-leds/live

# اعرف رقم الـ gpiochip الجديد
$ cat /sys/kernel/config/gpio-aggregator/vm-leds/dev_name
gpio-aggregator.0

$ ls /sys/devices/platform/gpio-aggregator.0/
gpiochip7/  # هو gpiochip7

# للـ virtio-gpio passthrough للـ VM
$ qemu-system-aarch64 \
    -object memory-backend-memfd,... \
    -device virtio-gpio-pci,ngpio=3,gpio-backend=gpio-aggregator.0 \
    ...
```

**على الـ Guest VM:**
```bash
$ gpioinfo
gpiochip0 - 3 lines:
    line   0:       "led0"  unused  output  active-high
    line   1:       "led1"  unused  output  active-high
    line   2:       "led2"  unused  output  active-high

# VM تتحكم في الـ LEDs
$ gpioset gpiochip0 0=1 1=0 2=1
```

**لما VM تخلص، إيقاف الـ aggregator:**
```bash
$ echo 0 > /sys/kernel/config/gpio-aggregator/vm-leds/live
$ rmdir /sys/kernel/config/gpio-aggregator/vm-leds/line{0,1,2}
$ rmdir /sys/kernel/config/gpio-aggregator/vm-leds
```

#### الدرس المستفاد
**الـ gpio-aggregator هو الحل الأمثل لـ GPIO isolation في بيئات الـ virtualization.** بدل ما تعطي الـ VM access لـ gpiochip كامل، بتعمل virtual chip بـ subset بس من الـ lines المطلوبة. الـ configfs interface بيوفر lifecycle management دقيق: تقدر تعمل الـ aggregator قبل الـ VM وتمسحه بعدها. الـ `dev_name` attribute بيسهل تعرف الـ device path للـ VFIO/virtio passthrough.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي اتكتبت على LWN.net عن الـ GPIO Aggregator:

| المقالة | الوصف |
|---------|-------|
| [gpio: Add GPIO Aggregator Driver](https://lwn.net/Articles/798971/) | أول patch قدّم الـ driver، مناقشة مجتمع الـ kernel لأول تصميم |
| [gpio: Add GPIO Aggregator](https://lwn.net/Articles/809647/) | النسخة المحدّثة من الـ patch بعد review cycles متعددة |
| [gpio: aggregator: Introduce delay support for individual output pins](https://lwn.net/Articles/934274/) | إضافة دعم الـ delay لـ output pins فردية |
| [gpio: aggregator: Incorporate gpio-delay functionality](https://lwn.net/Articles/934739/) | دمج وظائف الـ gpio-delay جوّا الـ aggregator مباشرةً |
| [gpio: consumer: new virtual driver](https://lwn.net/Articles/942479/) | driver افتراضي جديد متعلق بنظام الـ GPIO |

---

### الوثائق الرسمية في الـ Kernel

```
Documentation/admin-guide/gpio/gpio-aggregator.rst   ← الملف الأساسي
Documentation/admin-guide/gpio/index.rst             ← فهرس GPIO
Documentation/driver-api/gpio/driver.rst             ← واجهة كتابة GPIO drivers
Documentation/driver-api/gpio/board.rst              ← GPIO Mappings في الـ DT
Documentation/driver-api/gpio/drivers-on-gpio.rst    ← subsystem drivers فوق GPIO
```

الـ Kconfig option المقابلة:

```
CONFIG_GPIO_AGGREGATOR=m
```

موجودة في `drivers/gpio/Kconfig` — ممكن تشوف كل الـ options المتعلقة بيها من هناك.

الـ source file الرئيسي:

```
drivers/gpio/gpio-aggregator.c
```

---

### Kernel Commits المهمة

| الـ commit | الوصف |
|-----------|-------|
| `828546e2` | `gpio: Add GPIO Aggregator` — أول commit أضاف الـ driver في kernel 5.7 |
| patch series v3 | [v3,0/7] gpio: Add GPIO Aggregator/Repeater — على [patchwork.kernel.org](https://patchwork.kernel.org/cover/11263641/) |

لاستعراض تاريخ الـ commits على الملف مباشرةً:

```bash
git log --oneline drivers/gpio/gpio-aggregator.c
```

---

### نقاشات Mailing List

الـ patches اتناقشت بشكل أساسي على قائمتين:

- **linux-gpio** — قائمة الـ GPIO subsystem الرسمية
- **linux-hardening** — نقاشات الأمان والـ forwarder API

أهم الـ threads على [mail-archive.com](https://www.mail-archive.com/linux-hardening@vger.kernel.org/msg09329.html):

- `[PATCH v5 08/12] gpio: aggregator: export symbols of the GPIO forwarder library`
- `[PATCH v5 10/12] gpio: aggregator: add possibility to attach data to the forwarder`
- `[PATCH v9 08/10] gpio: aggregator: add possibility to attach data to the forwarder`

المشرفون الرئيسيون في النقاشات:
- **Geert Uytterhoeven** — المؤلف الأصلي
- **Andy Shevchenko** — reviewer أساسي
- **Linus Walleij** — مشرف subsystem الـ GPIO

---

### مقالات خارجية مهمة

#### Bootlin Blog

[GPIO Aggregator, a virtual gpio chip – Bootlin](https://bootlin.com/blog/gpio-aggregator-a-virtual-gpio-chip/)

مقالة عملية بتشرح إزاي تستخدم الـ GPIO Aggregator في embedded systems من خلال الـ Device Tree، مع أمثلة `gpioinfo` واضحة.

#### توثيق Kernel الرسمي أونلاين

- [docs.kernel.org — GPIO Aggregator](https://docs.kernel.org/admin-guide/gpio/gpio-aggregator.html)
- [kernel.org v5.8 — GPIO Aggregator](https://www.kernel.org/doc/html/v5.8/admin-guide/gpio/gpio-aggregator.html)

#### Kernel Newbies

صفحات kernel newbies بتوثّق الـ features اللي دخلت في كل version:

- [Linux_5.7 — kernelnewbies.org](https://kernelnewbies.org/Linux_5.7) — الـ version اللي دخل فيها الـ GPIO Aggregator لأول مرة
- [Linux_6.11 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.11) — تحديثات GPIO subsystem
- [Linux_6.13 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.13) — أحدث تغييرات

#### Kernel Driver Database

[CONFIG_GPIO_AGGREGATOR — cateee.net](https://cateee.net/lkddb/web-lkddb/GPIO_AGGREGATOR.html) — بيوضح في أنهي kernel versions الـ config option ظهرت.

---

### كتب مقترحة

#### Linux Device Drivers 3rd Edition (LDD3)

- **الفصل 1**: مقدمة الـ kernel وكيفية بناء الـ modules
- **الفصل 14**: The Linux Device Model — لفهم الـ `platform_driver` و `bus_type`
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Ed.)

- **الفصل 17**: Devices and Modules — يشرح `module_init` وتسجيل الـ drivers
- **الفصل 18**: Debugging — أدوات مفيدة لاختبار الـ GPIO drivers
- ده المرجع اللي بيشرح الـ kernel internals بأسلوب عملي واضح

#### Embedded Linux Primer — Christopher Hallinan (2nd Ed.)

- **الفصل 15**: Kernel Initialization — تفهم ازاي الـ platform devices بتتسجل
- بيركز على ARM وـ embedded systems اللي بتستخدم GPIO كتير

#### The Linux Programming Interface — Michael Kerrisk

- **الفصل 15 و62**: File permissions و character devices — مهم لفهم جزء الـ access control في الـ aggregator

---

### Search Terms للبحث عن معلومات أكثر

```
gpio-aggregator linux kernel
gpio aggregator sysfs new_device
gpio aggregator configfs live attribute
gpio forwarder library kernel
gpio_chip aggregation virtual
CONFIG_GPIO_AGGREGATOR
gpio aggregator DT binding generic driver
driver_override gpio aggregator
gpio aggregator VM security isolation
gpiochip userspace access control linux
```

---

### مصادر الـ Source Code مباشرةً

```bash
# قراءة الـ driver
vim drivers/gpio/gpio-aggregator.c

# الـ Kconfig
grep -n GPIO_AGGREGATOR drivers/gpio/Kconfig

# الـ documentation
vim Documentation/admin-guide/gpio/gpio-aggregator.rst

# history كامل للملف
git log --follow -p drivers/gpio/gpio-aggregator.c
```
## Phase 8: Writing simple module

### الـ Hook المختار — Tracepoint: `gpio_value`

الـ tracepoint ده موجود في `include/trace/events/gpio.h` وبيتفعّل في كل مرة kernel بيعمل get أو set لقيمة أي GPIO line — سواء كانت عبر controller عادي أو عبر gpio-aggregator. ده بيخلّيه أداة مثالية لمراقبة نشاط الـ GPIO في النظام كله بدون ما نعدّل في الـ driver الأصلي.

---

### الـ Module كاملاً

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * gpio_value_watcher.c
 *
 * Hooks the gpio_value tracepoint to log every GPIO read/write.
 * Useful for observing gpio-aggregator activity in real-time.
 */

/* --- Includes --- */
#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>       /* pr_info */
#include <linux/tracepoint.h>   /* for_each_kernel_tracepoint, tracepoint_probe_register */

/*
 * We need to define TRACE_SYSTEM before pulling in the header,
 * and create the tracepoint stubs so we can register a probe.
 */
#define CREATE_TRACE_POINTS
#include <trace/events/gpio.h>  /* gpio_value tracepoint definition */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("GPIO Watcher <example@kernel.org>");
MODULE_DESCRIPTION("Tracepoint probe on gpio_value — monitors every GPIO get/set");

/* ------------------------------------------------------------------ */
/*  Tracepoint probe callback                                           */
/* ------------------------------------------------------------------ */

/**
 * probe_gpio_value - called by the kernel on every gpio_value tracepoint hit.
 * @ignore:  unused void* data pointer passed at registration time
 * @gpio:    the global GPIO number (matches /sys/class/gpio/gpioN)
 * @get:     1 if this is a read (get), 0 if this is a write (set)
 * @value:   the logical level read or written (0 or 1)
 */
static void probe_gpio_value(void *ignore,
                             unsigned int gpio,
                             int get,
                             int value)
{
    /*
     * pr_info prints to the kernel log (dmesg).
     * %s ternary tells us the direction: read or write.
     * This fires for EVERY GPIO in the system, including
     * all lines managed by a gpio-aggregator virtual chip.
     */
    pr_info("gpio_value: gpio=%u  op=%-3s  level=%d\n",
            gpio,
            get ? "GET" : "SET",
            value);
}

/* ------------------------------------------------------------------ */
/*  Module init / exit                                                  */
/* ------------------------------------------------------------------ */

static int __init gpio_watcher_init(void)
{
    int ret;

    /*
     * Register our probe function on the gpio_value tracepoint.
     * The kernel will call probe_gpio_value() every time any driver
     * (including gpio-aggregator's forwarding layer) fires this tracepoint.
     *
     * NULL as third arg means "no private data" passed to the probe.
     */
    ret = register_trace_gpio_value(probe_gpio_value, NULL);
    if (ret) {
        pr_err("gpio_watcher: failed to register gpio_value probe: %d\n", ret);
        return ret;
    }

    pr_info("gpio_watcher: loaded — watching all GPIO get/set via gpio_value tracepoint\n");
    return 0;
}

static void __exit gpio_watcher_exit(void)
{
    /*
     * Unregister the probe BEFORE the module is unloaded from memory.
     * If we skip this, the kernel will call a function pointer that no
     * longer exists → guaranteed NULL-deref / kernel panic.
     */
    unregister_trace_gpio_value(probe_gpio_value, NULL);

    /*
     * tracepoint_synchronize_unregister() waits until any CPU that is
     * currently inside the tracepoint's RCU read-side critical section
     * has finished. Without this, our module memory could be freed while
     * a tracepoint callback is still running on another CPU.
     */
    tracepoint_synchronize_unregister();

    pr_info("gpio_watcher: unloaded\n");
}

module_init(gpio_watcher_init);
module_exit(gpio_watcher_exit);
```

---

### Makefile

```makefile
obj-m += gpio_value_watcher.o

# Set KDIR to your running kernel's build tree
KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ Includes

```c
#include <linux/tracepoint.h>
#define CREATE_TRACE_POINTS
#include <trace/events/gpio.h>
```

الـ `CREATE_TRACE_POINTS` بيقول للـ preprocessor يولّد الـ struct الداخلي للـ tracepoint من ملف الـ events. من غير السطر ده، الـ `register_trace_gpio_value` مش هتتعرف.

---

#### الـ Callback — `probe_gpio_value`

```c
static void probe_gpio_value(void *ignore,
                             unsigned int gpio,
                             int get,
                             int value)
```

البروتوتايب لازم يتطابق تماماً مع `TP_PROTO` اللي معرّفة في `gpio.h`:
```c
TP_PROTO(unsigned gpio, int get, int value)
```
الـ `void *ignore` هو الـ data pointer اللي بنحطه في `register_trace_*` — في حالتنا `NULL` لأننا مش محتاجين state خاص.

الـ `gpio` هو رقم GPIO العالمي في النظام. لو عندك aggregator بـ 3 lines مثلاً، كل line ليها رقم منفصل هنشوفه هنا مع كل عملية get/set.

---

#### الـ Registration في `module_init`

```c
ret = register_trace_gpio_value(probe_gpio_value, NULL);
```

الدالة دي بتُولَّد أوتوماتيك من الـ `TRACE_EVENT` macro في وقت الـ build. بترجع 0 لو نجحت، أو error code لو الـ tracepoint مش enabled في الـ kernel config.

---

#### الـ Unregistration في `module_exit`

```c
unregister_trace_gpio_value(probe_gpio_value, NULL);
tracepoint_synchronize_unregister();
```

السطر الأول بيشيل الـ probe من قائمة الـ callbacks. السطر التاني ضروري جداً — بيستنّى كل الـ CPUs تخلّص أي استدعاء جاري للـ callback قبل ما module memory تتحرّر، ده بيمنع race condition كلاسيكي.

---

### كيفية الاختبار

```bash
# بناء الـ module
make

# تحميله
sudo insmod gpio_value_watcher.ko

# محاكاة نشاط GPIO عبر gpio-aggregator
echo "gpiochip0 6-7" | sudo tee /sys/bus/platform/drivers/gpio-aggregator/new_device

# قراءة القيمة من userspace (يستخدم gpioget من libgpiod)
gpioget gpiochip0 6

# مشاهدة اللوج
dmesg | grep gpio_watcher

# تفريغ الـ module
sudo rmmod gpio_value_watcher
```

**مثال على الـ output في dmesg:**

```
gpio_watcher: loaded — watching all GPIO get/set via gpio_value tracepoint
gpio_watcher: gpio_value: gpio=496  op=GET  level=1
gpio_watcher: gpio_value: gpio=497  op=SET  level=0
gpio_watcher: unloaded
```

---

### جدول الـ Tracepoint Arguments

| الـ Field | النوع | المعنى |
|-----------|-------|---------|
| `gpio` | `unsigned int` | رقم GPIO العالمي في النظام |
| `get` | `int` | `1` = قراءة، `0` = كتابة |
| `value` | `int` | القيمة المنطقية `0` أو `1` |

---

### ليه `gpio_value` تحديداً؟

الـ gpio-aggregator بيشتغل عبر `gpiochip_fwd_gpio_get` و `gpiochip_fwd_gpio_set` اللي موجودين في نفس الـ driver. كلاهما في النهاية بيمرّوا بالـ GPIO core اللي بيطلق الـ `gpio_value` tracepoint. بالتالي الـ hook ده بيشوف كل عمليات الـ aggregator من غير ما نحتاج نعدّل في الـ driver نفسه أو نستخدم kprobe على دالة داخلية.
