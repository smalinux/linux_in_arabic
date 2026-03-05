## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الـ `gpio-aggregator.c` جزء من **GPIO Subsystem** في Linux kernel، تحت مسار `drivers/gpio/`.
المشرف عليه: Geert Uytterhoeven — مدرج في `MAINTAINERS` تحت قسم **GPIO AGGREGATOR**.

---

### المشكلة اللي بيحلها بالتفصيل

#### تخيل السيناريو ده

عندك جهاز embedded — مثلاً Raspberry Pi أو أي board صناعي — فيه شريحة GPIO واحدة كبيرة زي `gpiochip0` فيها 40 pin. النظام فيه:
- **Application A** محتاجة pins 0, 3, 7
- **Virtual Machine** محتاجة pins 10, 11, 12
- **User عادي** محتاج pins 20, 21

المشكلة: الـ `/dev/gpiochip0` بيُعطي access للشريحة **كلها** أو **ولا حاجة** — مفيش access control بالـ pin. لو أعطيت الـ VM حق الوصول لـ `gpiochip0`، هتقدر تتحكم في الـ 40 pin كلهم!

**الـ GPIO Aggregator بيحل المشكلة دي بالضبط.**

---

### الفكرة الأساسية (ELI5)

تخيل عندك **لوحة كهربائية كبيرة** فيها 40 مفتاح. أنت عايز تدي كل شخص access لمجموعة مفاتيح معينة بس.

الـ GPIO Aggregator بيعمل "**لوحة صغيرة افتراضية**" جديدة — بيجمع فيها بس المفاتيح اللي انت محتاجها — وبيعرضها كـ **`gpiochip` جديدة مستقلة**. كأنك بتعمل copy من المفاتيح دي في لوحة منفصلة وبتديها للشخص بالكامل.

```
gpiochip0 (40 pins)
  pin 0  ─────────┐
  pin 3  ──────────┤  gpio-aggregator.0  →  /dev/gpiochip5  (للـ VM)
  pin 7  ─────────┘

  pin 10 ─────────┐
  pin 11 ──────────┤  gpio-aggregator.1  →  /dev/gpiochip6  (للـ User)
  pin 12 ─────────┘
```

---

### مكونات الملف الثلاثة الرئيسية

#### 1. الـ GPIO Forwarder (`gpiochip_fwd`)

ده الـ **engine** الداخلي. بيعمل `gpio_chip` جديدة وبيربط كل عملية فيها (get, set, direction...) بالـ descriptors الأصلية.

```c
struct gpiochip_fwd {
    struct gpio_chip chip;       // الـ virtual gpiochip الجديدة
    struct gpio_desc **descs;    // مؤشرات للـ GPIOs الأصلية
    struct gpiochip_fwd_timing *delay_timings; // delays اختيارية
    unsigned long *valid_mask;   // أي pins متاحة فعلاً
    unsigned long tmp[];         // buffer مؤقت لعمليات متعددة
};
```

لما حد يعمل `gpio_set(virtual_pin_0)` على الـ chip الجديدة، الـ forwarder بيروح يعمل `gpiod_set_value` على الـ descriptor الأصلي المقابل.

**ميزة إضافية:** بيدعم **RC filter delays** عبر `gpio-delay` compatible — مفيد لـ open-drain outputs اللي محتاجة وقت ramp-up/ramp-down.

#### 2. الـ Sysfs Interface (legacy)

الطريقة القديمة لإنشاء aggregator — بتكتب في:
```
/sys/bus/platform/drivers/gpio-aggregator/new_device
```

مثال:
```bash
# اجمع pin 19 من gpiochip e6052000 و pins 20-21 من e6050000
echo 'e6052000.gpio 19 e6050000.gpio 20-21' > new_device

# الحذف
echo gpio-aggregator.0 > delete_device
```

#### 3. الـ Configfs Interface (modern)

الطريقة الحديثة — بتنشئ directory structure في `/config/gpio-aggregator/`:

```
/config/gpio-aggregator/
└── my-device/          ← mkdir لإنشاء aggregator جديد
    ├── line0/          ← mkdir لكل line
    │   ├── key         ← اسم الـ gpiochip أو line name
    │   ├── offset      ← رقم الـ pin (-1 = lookup by name)
    │   └── name        ← اسم مخصص للـ virtual line
    ├── line1/
    ├── dev_name        ← اسم الـ device المُنشأ (read-only)
    └── live            ← اكتب '1' لتفعيل، '0' لإيقاف
```

---

### القصة الكاملة: إزاي الكود بيشتغل

```
User كتب في configfs:
  mkdir /config/gpio-aggregator/my-vm-gpio
  mkdir /config/gpio-aggregator/my-vm-gpio/line0
  echo "e6052000.gpio" > .../line0/key
  echo 5 > .../line0/offset
  echo 1 > .../my-vm-gpio/live
         │
         ▼
  gpio_aggregator_activate()
         │
         ├─► بيبني gpiod_lookup_table ← ربط الـ virtual pins بالأصلية
         │
         ├─► بيعمل software node (fwnode) بأسماء الـ lines
         │
         ├─► dev_sync_probe_register() ← بيسجل platform_device
         │            └─► ينتظر لحد ما probe ينتهي
         │
         ▼
  gpio_aggregator_probe()  [platform driver]
         │
         ├─► gpiod_count() ← عد الـ GPIOs المطلوبة
         ├─► devm_gpiod_get_index() ← جيب الـ descriptors الأصلية
         └─► gpiochip_fwd_create() ← انشئ الـ virtual chip
                    │
                    └─► devm_gpiochip_add_data() ← سجلها في kernel
                                │
                                ▼
                    /dev/gpiochipN  ← متاح للـ user/VM!
```

---

### الـ `dev_sync_probe` — ليه موجود؟

لما بتسجل `platform_device`، الـ probe بيحصل **asynchronously**. الـ `dev_sync_probe` بيستخدم `completion` و `bus_notifier` عشان **ينتظر** لحد ما الـ probe ينتهي فعلاً — ده مهم لـ configfs لأن الـ user بيتوقع إن الـ device موجود فعلاً لما يكتب `1` في `live`.

---

### الـ `gpio-delay` Feature

لو الـ Device Tree فيه `compatible = "gpio-delay"`، الـ forwarder بيدعم delays:
```
gpio-delay {
    compatible = "gpio-delay";
    gpios = <&gpiochip 5 3 1000 500>;  /* pin, flags, ramp_up_us, ramp_down_us */
};
```
ده بيسمح بمحاكاة open-drain circuits اللي بتحتاج RC charging time.

---

### الـ Locking Model

| الـ Lock | يحمي إيه |
|---|---|
| `gpio_aggregator_lock` (mutex) | الـ `gpio_aggregator_idr` — الـ IDR الـ global |
| `aggr->lock` (mutex) | state الـ aggregator الواحد (active/inactive/list) |
| `fwd->mlock` (mutex) | عمليات `get/set_multiple` لو الـ chip تحتاج sleep |
| `fwd->slock` (spinlock) | نفس الغرض لو مش محتاج sleep |

---

### الفرق بين الـ Interfaces

| الخاصية | Sysfs (legacy) | Configfs (modern) |
|---|---|---|
| المسار | `/sys/bus/platform/drivers/gpio-aggregator/` | `/config/gpio-aggregator/` |
| الإنشاء | كتابة string في `new_device` | `mkdir` |
| الحذف | كتابة اسم في `delete_device` | `rmdir` |
| المرونة | محدودة | كاملة |
| الـ live toggle | لا | نعم |
| البادئة | — | لا يجوز تبدأ بـ `_sysfs` |

---

### الملفات ذات الصلة

#### الـ Core (GPIO Subsystem)

| الملف | الدور |
|---|---|
| `drivers/gpio/gpiolib.c` | قلب الـ GPIO subsystem — إدارة الـ chips والـ descriptors |
| `drivers/gpio/gpiolib-cdev.c` | الـ `/dev/gpiochipN` character device interface |
| `drivers/gpio/gpiolib-sysfs.c` | الـ sysfs القديم للـ GPIO |
| `drivers/gpio/dev-sync-probe.c/.h` | مزامنة الـ probe مع الـ configfs |

#### الـ Headers الأساسية

| الملف | الدور |
|---|---|
| `include/linux/gpio/driver.h` | تعريف `struct gpio_chip` والـ API للـ drivers |
| `include/linux/gpio/consumer.h` | الـ `gpiod_*` consumer API |
| `include/linux/gpio/machine.h` | الـ `gpiod_lookup_table` وماكروهات الـ GPIO_LOOKUP |
| `include/linux/gpio/forwarder.h` | الـ API الـ exported للـ `gpiochip_fwd_*` functions |

#### الـ Documentation

| الملف | الدور |
|---|---|
| `Documentation/admin-guide/gpio/gpio-aggregator.rst` | دليل الاستخدام الكامل |
| `Documentation/admin-guide/gpio/gpio-mockup.rst` | أداة testing مشابهة |
| `Documentation/admin-guide/gpio/gpio-sim.rst` | GPIO simulator |

#### ملفات Hardware Drivers للمقارنة

| الملف | الدور |
|---|---|
| `drivers/gpio/gpio-74x164.c` | مثال real driver لـ shift register |
| `drivers/gpio/gpio-line-mux.c` | مشابه — multiplexing للـ GPIO lines |
## Phase 2: شرح الـ GPIO Aggregator Framework

---

### المشكلة — ليه الـ subsystem ده موجود أصلاً؟

في الأنظمة المدمجة، الـ GPIO lines موزعة على أكتر من GPIO controller. مثلاً، في بورد مدمجة عندك:

- `gpiochip0` (SoC GPIO) → يتحكم في pin 5 و pin 12
- `gpiochip1` (GPIO expander على I2C) → يتحكم في pin 3

المشكلة: أي driver أو userspace application يحتاج يتعامل مع الاتنين دول كـ وحدة واحدة متجانسة — مثلاً، relay board بـ 8 أطراف، كل طرف على chip مختلفة.

**الـ GPIOLIB** الأساسي مش بيقدم abstraction تجمع lines من controllers مختلفين في device واحد. كل chip موجودة بشكل مستقل. ده بيخلي الكود المستهلك معقد — لازم يـ track كل chip بشكل منفصل، ويـ manage الـ ownership، ويعمل error handling متعدد.

**المشكلة التانية**: في production systems، الـ userspace tool (زي `libgpiod`) مش المفروض يعرف التوزيع الداخلي الـ hardware. المفروض يشوف "relay_bank" device بـ 8 pins، ويتعامل معاه بشكل مجرد.

---

### الحل — الـ kernel بيتعامل إزاي مع الموضوع ده؟

الـ `gpio-aggregator` driver بيحل المشكلة دي بـ:

1. **تجميع** GPIO lines من controllers مختلفة في **virtual GPIO chip** واحدة.
2. تعريض الـ virtual chip دي كـ platform device عادي على الـ kernel.
3. توفير واجهتين للتحكم:
   - **Legacy sysfs** (قديم): كتابة string مباشرة لـ `/sys/bus/platform/drivers/gpio-aggregator/new_device`
   - **ConfigFS** (جديد): إنشاء directories وكتابة attributes بشكل structured.

الـ driver بيعمل ده عن طريق:
- إنشاء **`gpiochip_fwd`** (GPIO Forwarder) — يمثل الـ virtual chip.
- كل operation على الـ virtual chip (get/set/direction) يتم **forwarding** تلقائي للـ descriptor الأصلي في الـ hardware chip.

---

### التشبيه العملي — The Patch Panel Analogy

تخيل **patch panel** في data center:

```
Physical Servers          Patch Panel          Switch Ports
(gpiochip0, gpiochip1) → (gpio-aggregator) → (virtual gpiochip)
     port A1    ──────────── slot 0  ────────── virtual line 0
     port B7    ──────────── slot 1  ────────── virtual line 1
     port C3    ──────────── slot 2  ────────── virtual line 2
```

| Patch Panel Concept | Kernel Concept |
|---|---|
| Physical server port | GPIO descriptor (`gpio_desc`) على hardware chip |
| Patch panel slot | `gpio_aggregator_line` — وحدة التعريف والربط |
| Patch panel as a whole | `gpio_aggregator` — الـ aggregator object |
| The switch port (consumer side) | `gpiochip_fwd` — الـ virtual chip المسجلة في GPIOLIB |
| Connecting cables | `gpiod_lookup_table` — الجدول الذي يخبر GPIOLIB بمكان الـ GPIO الحقيقي |
| Patch panel admin interface | ConfigFS / sysfs interface |
| Activating a panel configuration | `gpio_aggregator_activate()` — يسجل الـ platform device |

لما consumer يطلب `gpio_get(virtual_chip, 2)`:
1. الـ `gpio_fwd_get()` بتشتغل.
2. بتجيب الـ `gpio_desc` المخزون في `fwd->descs[2]`.
3. بتنادي `gpiod_get_value()` على الـ descriptor الأصلي.
4. الـ value بترجع كأنها من chip واحدة.

---

### الـ Big Picture Architecture

```
User / Driver
    │
    │  gpiod_get_index(dev, NULL, i, GPIOD_ASIS)
    ▼
┌─────────────────────────────────────────────┐
│            GPIOLIB Consumer API             │
│  (include/linux/gpio/consumer.h)            │
└────────────────────┬────────────────────────┘
                     │  lookup via gpiod_lookup_table
                     ▼
┌─────────────────────────────────────────────┐
│         gpio-aggregator platform device     │
│         (gpio-aggregator.0, .1, ...)        │
│                                             │
│  gpio_aggregator_probe()                    │
│     ↓                                       │
│  gpiochip_fwd_create()                      │
│     ↓                                       │
│  ┌──────────────────────────────────────┐   │
│  │         gpiochip_fwd                 │   │
│  │  ┌──────────────┐  ┌─────────────┐  │   │
│  │  │  gpio_chip   │  │   descs[]   │  │   │
│  │  │  (virtual)   │  │ desc* desc* │  │   │
│  │  │  ngpio = N   │  │ desc* ...   │  │   │
│  │  └──────┬───────┘  └──────┬──────┘  │   │
│  │         │  forward ops    │         │   │
│  └─────────┼─────────────────┼─────────┘   │
└────────────┼─────────────────┼─────────────┘
             │                 │
     ┌───────▼───────┐ ┌───────▼───────┐
     │  gpiochip0    │ │  gpiochip1    │
     │  (SoC GPIO)   │ │  (I2C expan.) │
     │  line 5,12    │ │  line 3       │
     └───────────────┘ └───────────────┘

ConfigFS / sysfs (control plane)
┌────────────────────────────────────────────┐
│  /sys/kernel/config/gpio-aggregator/       │
│    mydevice/                               │
│      line0/  key="gpiochip0" offset=5      │
│      line1/  key="gpiochip0" offset=12     │
│      line2/  key="gpiochip1" offset=3      │
│      live → "1"  (activate!)               │
└────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ core abstraction هنا هي **"Virtual GPIO Chip عن طريق Forwarding"**.

الكيرنل بدل ما يحاول يعمل merge حقيقي للـ GPIO controllers، بيعمل حاجة أبسط: يسجل **chip جديدة** بـ vtable كاملة، وكل function في الـ vtable دي بتعمل **delegation** للـ descriptor الحقيقي.

```
Virtual chip operation request
         │
         ▼
   gpio_fwd_get(chip, offset)
         │
         ├──► fwd->descs[offset]  →  gpio_desc* للـ hardware pin الأصلي
         │
         └──► gpiod_get_value(fwd->descs[offset])
                    │
                    ▼
             Hardware GPIO Controller
```

ده يعني الـ **GPIOLIB** مش محتاج يعرف حاجة عن الـ aggregation — هو بيشوف chip عادية.

---

### هياكل البيانات الأساسية وعلاقتها ببعض

#### `struct gpio_aggregator` — الـ aggregator نفسه

```c
struct gpio_aggregator {
    struct dev_sync_probe_data probe_data; // يربطه بالـ platform_device
    struct config_group group;             // ConfigFS node
    struct gpiod_lookup_table *lookups;    // جدول البحث للـ GPIOLIB
    struct mutex lock;                     // يحمي الـ state
    int id;                                // IDR-assigned unique ID

    struct list_head list_head;            // قائمة gpio_aggregator_line
    bool init_via_sysfs;                   // هل أُنشئ عبر sysfs legacy؟
    char args[];                           // raw sysfs args (legacy only)
};
```

#### `struct gpio_aggregator_line` — تعريف كل خط

```c
struct gpio_aggregator_line {
    struct config_group group;      // ConfigFS sub-node (line0, line1, ...)
    struct gpio_aggregator *parent; // pointer للـ parent aggregator
    struct list_head entry;         // entry في قائمة الـ parent

    unsigned int idx;               // ترتيبه في الـ virtual chip
    const char *name;               // اسم مخصص للـ virtual line
    const char *key;                // label للـ chip الأصلية أو اسم الـ line
    int offset;                     // الـ offset داخل الـ hardware chip (-1 = by name)
    enum gpio_lookup_flags flags;   // ACTIVE_LOW, OPEN_DRAIN, إلخ
};
```

#### `struct gpiochip_fwd` — الـ Virtual Chip

```c
struct gpiochip_fwd {
    struct gpio_chip chip;          // الـ virtual chip المسجلة في GPIOLIB
    struct gpio_desc **descs;       // array of pointers للـ hardware descriptors
    union {
        struct mutex mlock;         // لو الـ chip can_sleep (I2C expander)
        spinlock_t slock;           // لو الـ chip لا تنام (SoC GPIO)
    };
    struct gpiochip_fwd_timing *delay_timings; // اختياري: ramp-up/down delays
    void *data;                     // driver-private data
    unsigned long *valid_mask;      // bitmap للـ lines الصالحة
    unsigned long tmp[];            // scratch buffer للـ multi-line ops
};
```

#### العلاقة بين الـ structs

```
gpio_aggregator
      │
      ├── config_group (ConfigFS node)
      │         └── gpio_aggregator_line[] (sub-groups: line0, line1...)
      │                   │
      │                   └── {key, offset} → gpiod_lookup entry
      │
      ├── gpiod_lookup_table → registered with GPIOLIB
      │         └── table[] → gpiod_lookup entries
      │
      └── probe_data.pdev → platform_device
                │
                └── platform_set_drvdata → gpiochip_fwd
                          │
                          ├── gpio_chip (registered as virtual gpiochip)
                          └── descs[] → gpio_desc* (hardware lines)
```

---

### دورة الحياة الكاملة — من ConfigFS لـ Hardware

#### المرحلة 1: التعريف (ConfigFS)

```bash
# إنشاء aggregator جديد
mkdir /sys/kernel/config/gpio-aggregator/relay_board

# تعريف الخط الأول
mkdir /sys/kernel/config/gpio-aggregator/relay_board/line0
echo "gpiochip0" > relay_board/line0/key
echo "5"         > relay_board/line0/offset

# تعريف الخط الثاني (من I2C expander)
mkdir /sys/kernel/config/gpio-aggregator/relay_board/line1
echo "i2c-expander" > relay_board/line1/key
echo "3"            > relay_board/line1/offset
```

كل `mkdir line0` بيستدعي `gpio_aggregator_device_make_group()` اللي بينشئ `gpio_aggregator_line`.

#### المرحلة 2: الـ Activation

```bash
echo 1 > /sys/kernel/config/gpio-aggregator/relay_board/live
```

ده بيستدعي `gpio_aggregator_activate()`:

```
gpio_aggregator_activate()
    │
    ├── يبني gpiod_lookup_table من الـ lines
    │     └── GPIO_LOOKUP_IDX("gpiochip0", 5, NULL, 0, 0)
    │         GPIO_LOOKUP_IDX("i2c-expander", 3, NULL, 1, 0)
    │
    ├── gpiod_add_lookup_table(aggr->lookups)
    │     → يسجل الجدول مع GPIOLIB
    │
    └── dev_sync_probe_register(...)
          → ينشئ platform_device "gpio-aggregator.0"
          → يستدعي gpio_aggregator_probe()
```

#### المرحلة 3: الـ Probe

```c
gpio_aggregator_probe(pdev):
    n = gpiod_count(dev, NULL)          // = 2
    for i in 0..n:
        descs[i] = devm_gpiod_get_index(dev, NULL, i, GPIOD_ASIS)
        // GPIOLIB يبحث في lookup_table → يجد الـ hardware descriptor

    fwd = gpiochip_fwd_create(dev, n, descs, features)
    // ينشئ virtual chip ويسجلها في GPIOLIB

    platform_set_drvdata(pdev, fwd)
```

#### المرحلة 4: الاستخدام (Forwarding)

```
consumer calls: gpiod_set_value(virtual_desc_0, 1)
      │
      ▼
gpio_fwd_set(chip, offset=0, value=1)
      │
      ├── fwd->descs[0] → gpio_desc* for gpiochip0 line 5
      └── gpiod_set_value(hardware_desc, 1)
              │
              ▼
        SoC GPIO register write
```

---

### الـ Locking Strategy

الـ driver بيستخدم locking strategy ذكية بناءً على نوع الـ GPIO:

| النوع | Lock | السبب |
|---|---|---|
| SoC GPIO (لا ينام) | `spinlock_t slock` | آمن في interrupt context |
| I2C/SPI expander (ينام) | `struct mutex mlock` | لازم sleeping context |

الـ `can_sleep` flag في `gpio_chip` بيتحدد تلقائياً:
- لو أي line في الـ `descs[]` بتنام → كل الـ chip `can_sleep = true`
- ده مهم لأن `get_multiple` و `set_multiple` محتاجين يعرفوا أي lock يستخدموا

---

### الـ Multi-line Operations — التفصيل

الـ `gpio_fwd_get_multiple()` / `gpio_fwd_set_multiple()` بيتعاملوا مع **bitmap operations** بكفاءة:

```c
// tmp[] في نهاية gpiochip_fwd مقسوم لقسمين:
#define fwd_tmp_values(fwd)  (&(fwd)->tmp[0])
#define fwd_tmp_descs(fwd)   ((void *)&(fwd)->tmp[BITS_TO_LONGS((fwd)->chip.ngpio)])
```

```
fwd->tmp[] layout in memory:
┌──────────────────────────┬──────────────────────────────────┐
│  values bitmap           │  descs[] pointer array           │
│  BITS_TO_LONGS(ngpio)    │  ngpio * sizeof(gpio_desc*)      │
│  words                   │                                  │
└──────────────────────────┴──────────────────────────────────┘
 ↑ fwd_tmp_values(fwd)       ↑ fwd_tmp_descs(fwd)
```

الخوارزمية:
1. loop على الـ set bits في `mask` → يجمع الـ descriptors المقابلة
2. ينادي `gpiod_get_array_value()` مرة واحدة بدل N مرة
3. يعيد توزيع النتائج على الـ bits array

ده optimization مهم لـ performance — بدل N system calls، call واحدة.

---

### الـ Delay Feature — GPIO كـ RC Filter

في بعض الدوائر، الـ GPIO output بيحتاج وقت للوصول للـ level المطلوب (مثلاً open-drain مع RC filter):

```c
struct gpiochip_fwd_timing {
    u32 ramp_up_us;    // delay بعد set HIGH
    u32 ramp_down_us;  // delay بعد set LOW
};
```

ده بيتفعّل عبر Device Tree:

```dts
relay_gpio: relay-gpio {
    compatible = "gpio-delay";
    gpios = <&gpio0 5 GPIO_ACTIVE_HIGH 1000 500>;
    //                                 ^    ^
    //                              ramp_up ramp_down (us)
};
```

الـ `gpiochip_fwd_delay_of_xlate()` بتـ parse هذه الأرقام وتخزنها في `fwd->delay_timings[offset]`.

---

### الواجهتان — ConfigFS vs Legacy Sysfs

#### الـ Legacy Sysfs Interface

```bash
# إنشاء: string format: "chip_label offset1,offset2 ..."
echo "gpiochip0 5,12 gpiochip1 3" > \
    /sys/bus/platform/drivers/gpio-aggregator/new_device

# حذف
echo "gpio-aggregator.0" > \
    /sys/bus/platform/drivers/gpio-aggregator/delete_device
```

الـ `gpio_aggregator_parse()` بتـ parse الـ string وتبني الـ `gpio_aggregator_line` objects.

#### الـ ConfigFS Interface (Modern)

```
/sys/kernel/config/gpio-aggregator/
    my_aggregator/
        line0/
            key      ← chip label أو line name
            offset   ← hardware offset (-1 = lookup by name)
            name     ← اسم مخصص للـ virtual line
        line1/
            ...
        dev_name     ← (read-only) اسم الـ platform device
        live         ← 0/1 لتفعيل/إيقاف الـ aggregator
```

الـ configfs بيعمل **locking على الـ entries** لما الـ device يكون active عبر `configfs_depend_item_unlocked()` — ده بيمنع حذف line directory وهو الـ device شغال.

---

### ما الـ subsystem يمتلكه vs ما يفوضه للـ drivers

| **يمتلكه الـ gpio-aggregator** | **يفوضه للـ آخرين** |
|---|---|
| إنشاء الـ virtual `gpio_chip` وتسجيلها | الـ GPIOLIB (`gpiochip_add_data`) يملك registration details |
| بناء `gpiod_lookup_table` | الـ GPIOLIB يملك الـ lookup mechanism |
| إدارة ConfigFS / sysfs hierarchy | الـ configfs framework يملك الـ FS operations |
| الـ forwarding logic (vtable functions) | الـ hardware driver الأصلي يملك الـ actual GPIO control |
| إدارة الـ IDR للـ IDs | لا يفوض — يملكه كاملاً |
| الـ `valid_mask` bitmap | الـ hardware driver يحدد أي pins صالحة |
| الـ `can_sleep` detection | الـ GPIOLIB descriptor يحدد ذلك عبر `gpiod_cansleep()` |
| الـ delay timings | الـ Device Tree / DT parser يوفر الأرقام |

---

### مفاهيم مسبقة مطلوبة

- **GPIOLIB**: الـ subsystem الأساسي للـ GPIO في الكيرنل — يوفر `gpio_chip`, `gpio_desc`, وكل الـ consumer/driver APIs.
- **ConfigFS**: filesystem في `/sys/kernel/config/` بيسمح لـ userspace بإنشاء kernel objects عن طريق `mkdir`/`rmdir`/كتابة files — بديل أنظف من sysfs للـ dynamic object creation.
- **Platform Device**: device لا يكتشفه الكيرنل تلقائياً (لا PCI، لا USB) — لازم يتسجل يدوياً أو عبر Device Tree.
- **IDR**: Integer ID allocator — data structure يخصص unique integer IDs ويربطها بـ pointers، آمن مع الـ concurrency.
- **`devm_*` APIs**: resource-managed allocations تتحرر تلقائياً لما الـ device يتحرر.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Defines والـ Constants

| الـ Macro / Constant | القيمة | الغرض |
|---|---|---|
| `AGGREGATOR_MAX_GPIOS` | 512 | الحد الأقصى لعدد الـ GPIO lines في aggregator واحد |
| `AGGREGATOR_LEGACY_PREFIX` | `"_sysfs"` | البريفيكس المحجوز للـ aggregators المُنشأة via legacy sysfs |
| `FWD_FEATURE_DELAY` | `BIT(0)` | فيتشر لتفعيل الـ ramp-up/down delay على GPIO lines |
| `fwd_tmp_values(fwd)` | `&(fwd)->tmp[0]` | Pointer للـ values bitmap في الـ tmp flex array |
| `fwd_tmp_descs(fwd)` | `(void*)&(fwd)->tmp[BITS_TO_LONGS(ngpio)]` | Pointer لمصفوفة الـ gpio_desc pointers في الـ tmp array |
| `fwd_tmp_size(ngpios)` | `BITS_TO_LONGS(n) + n` | الحجم الكلي للـ tmp flex array (bitmask + desc pointers) |

---

### الـ Enum: `gpio_lookup_flags`

> معرّفة في `include/linux/gpio/machine.h`

| الـ Flag | القيمة | المعنى |
|---|---|---|
| `GPIO_ACTIVE_HIGH` | `0 << 0` | الـ line active عند high (default) |
| `GPIO_ACTIVE_LOW` | `1 << 0` | الـ line active عند low (مقلوب logic) |
| `GPIO_OPEN_DRAIN` | `1 << 1` | Open-drain output |
| `GPIO_OPEN_SOURCE` | `1 << 2` | Open-source output |
| `GPIO_PERSISTENT` | `0 << 3` | الـ state يتحافظ عليه (default) |
| `GPIO_TRANSITORY` | `1 << 3` | الـ state مؤقت، ممكن يتغير |
| `GPIO_PULL_UP` | `1 << 4` | تفعيل pull-up resistor |
| `GPIO_PULL_DOWN` | `1 << 5` | تفعيل pull-down resistor |
| `GPIO_PULL_DISABLE` | `1 << 6` | تعطيل الـ pull resistors |
| `GPIO_LOOKUP_FLAGS_DEFAULT` | `ACTIVE_HIGH \| PERSISTENT` | القيمة الافتراضية عند إنشاء line جديدة |

---

### الـ Config Options المهمة

| الـ Kconfig | التأثير |
|---|---|
| `CONFIG_OF_GPIO` | لو مفعّل، يُضيف دعم `gpiochip_fwd_delay_of_xlate` لقراءة الـ ramp timings من device tree |
| `CONFIG_GPIOLIB` | لازم مفعّل عشان الكود كله يشتغل |

---

### الـ Structs التفصيلية

#### 1. `struct gpio_aggregator`

**الغرض:** يمثّل virtual GPIO device كامل — من التكوين حتى الـ platform device. ده الـ top-level object.

```c
struct gpio_aggregator {
    struct dev_sync_probe_data probe_data; /* sync probe state + pdev pointer */
    struct config_group group;             /* configfs group node              */
    struct gpiod_lookup_table *lookups;    /* جدول lookup للـ GPIO lines       */
    struct mutex lock;                     /* يحمي الـ state وقت التعديل      */
    int id;                                /* IDR ID فريد                      */
    struct list_head list_head;            /* قائمة gpio_aggregator_line مرتبة */
    bool init_via_sysfs;                   /* صح = أُنشئ via legacy sysfs      */
    char args[];                           /* flex array: raw sysfs args string */
};
```

**العلاقات:**
- يحمل `list_head` → قائمة مرتبة من `gpio_aggregator_line`
- يحمل `probe_data.pdev` → الـ `platform_device` المُستنشأ
- يحمل `lookups` → `gpiod_lookup_table` تُسجَّل في gpiolib
- مُسجَّل في `gpio_aggregator_idr` (IDR global) عبر `id`

---

#### 2. `struct gpio_aggregator_line`

**الغرض:** يمثّل GPIO line واحدة داخل الـ aggregator — بمعلوماتها (chip, offset, name, flags).

```c
struct gpio_aggregator_line {
    struct config_group group;     /* configfs group: "line0", "line1", ...    */
    struct gpio_aggregator *parent;/* pointer للـ aggregator الأب             */
    struct list_head entry;        /* link في قائمة الـ aggregator             */
    unsigned int idx;              /* index الـ line (0-based, sequential)     */
    const char *name;              /* اسم مخصص للـ virtual line (optional)    */
    const char *key;               /* chip label أو line name للبحث           */
    int offset;                    /* offset داخل الـ chip، أو -1 = lookup by name */
    enum gpio_lookup_flags flags;  /* ACTIVE_HIGH/LOW, OPEN_DRAIN, إلخ        */
};
```

**ملاحظات:**
- لو `offset == -1` → الـ `key` اسم line يُبحث عنه بالاسم (gpiolib يستخدم `U16_MAX` داخلياً)
- لو `0 <= offset <= 65534` → الـ `key` اسم chip والـ `offset` هو الـ hwnum
- القائمة دايماً مرتبة تصاعدياً بالـ `idx`

---

#### 3. `struct gpio_aggregator_pdev_meta`

**الغرض:** بيانات metadata تُمرَّر كـ platform data للـ `platform_device` عند الإنشاء via legacy sysfs.

```c
struct gpio_aggregator_pdev_meta {
    bool init_via_sysfs; /* صح = الـ device اتعمل via sysfs */
};
```

**الاستخدام:** الـ `gpio_aggregator_probe` يقرأها عشان يعرف يمنع الـ deferred probe أو لا.

---

#### 4. `struct gpiochip_fwd`

**الغرض:** الـ GPIO forwarder الفعلي — يعمل virtual `gpio_chip` يمرّر كل operations للـ real GPIO descriptors.

```c
struct gpiochip_fwd {
    struct gpio_chip chip;              /* الـ virtual gpio_chip المُسجَّل في gpiolib */
    struct gpio_desc **descs;           /* مصفوفة pointers للـ real gpio_desc      */
    union {
        struct mutex mlock;             /* يحمي tmp[] لو chip.can_sleep == true    */
        spinlock_t slock;               /* يحمي tmp[] لو chip.can_sleep == false   */
    };
    struct gpiochip_fwd_timing *delay_timings; /* مصفوفة timing per-line (optional) */
    void *data;                         /* driver-private data                      */
    unsigned long *valid_mask;          /* bitmap: أي offsets فيها descs صالحة     */
    unsigned long tmp[];                /* flex: bitmask values + desc pointers     */
};
```

**ملاحظات مهمة:**
- الـ `tmp[]` flex array فيه منطقتين: bitmask للـ values، ثم array من `gpio_desc*` — كلهم allocated دفعة واحدة
- الـ `valid_mask` بيتشيك في كل operation عشان يرفض offsets مش مكوّنة
- لو أي line واحدة `can_sleep` → كل الـ chip بيبقى `can_sleep = true`

---

#### 5. `struct gpiochip_fwd_timing`

**الغرض:** تخزين delay timings لكل GPIO line (رفع/خفض).

```c
struct gpiochip_fwd_timing {
    u32 ramp_up_us;   /* تأخير بالـ microseconds عند set إلى active */
    u32 ramp_down_us; /* تأخير بالـ microseconds عند set إلى inactive */
};
```

**الاستخدام:** موجود في `fwd->delay_timings[offset]`، يُستخدم في `gpio_fwd_delay()` بعد كل set operation.

---

#### 6. `struct dev_sync_probe_data`

**الغرض:** ضمان إن الـ platform device اتعمل probe بنجاح قبل ما الـ aggregator يكمل activation.

```c
struct dev_sync_probe_data {
    struct platform_device *pdev;       /* الـ pdev المُنشأ                        */
    const char *name;                   /* اسم الـ device                          */
    struct notifier_block bus_notifier; /* notifier يراقب أحداث الـ bus           */
    struct completion probe_completion; /* completion object للـ sync              */
    bool driver_bound;                  /* صح = الـ driver اتعمل bind بنجاح       */
};
```

---

#### 7. `struct gpiod_lookup_table`

**الغرض:** جدول يربط GPIO lines بـ device معين، يُسجَّل في gpiolib.

```c
struct gpiod_lookup_table {
    struct list_head list;    /* link في القائمة الـ global في gpiolib */
    const char *dev_id;       /* اسم الـ device: "gpio-aggregator.N"  */
    struct gpiod_lookup table[]; /* flex array من gpiod_lookup entries */
};
```

**كل `gpiod_lookup` entry بيحتوي على:**

| الـ Field | المعنى |
|---|---|
| `key` | اسم الـ chip أو اسم الـ line |
| `chip_hwnum` | رقم الـ line داخل الـ chip، أو `U16_MAX` لـ lookup by name |
| `con_id` | NULL في الـ aggregator (الـ lines تُجلب by index) |
| `idx` | الـ index داخل الـ aggregator (0, 1, 2, ...) |
| `flags` | gpio_lookup_flags |

---

### رسم العلاقات بين الـ Structs

```
                    [gpio_aggregator_idr]  (IDR global)
                           │
                           │ id → *aggr
                           ▼
              ┌─────────────────────────────────────┐
              │       gpio_aggregator               │
              │  ┌─────────────────────────────┐    │
              │  │  dev_sync_probe_data         │    │
              │  │    pdev ──────────────────────────┼──→ platform_device
              │  │    bus_notifier               │    │
              │  │    probe_completion           │    │
              │  └─────────────────────────────┘    │
              │  config_group (configfs node)        │
              │  mutex lock                          │
              │  id (IDR key)                        │
              │  lookups ─────────────────────────────┼──→ gpiod_lookup_table
              │  list_head ◄──────────────────────────┼──┐
              └─────────────────────────────────────┘  │
                           │                           │
              (list_head → entry links)                │
                           │                           │
              ┌────────────┴──────────────┐            │
              │  gpio_aggregator_line     │            │
              │  config_group             │            │
              │  *parent ─────────────────┼────────────┘
              │  list_head entry (linked) │
              │  idx, name, key, offset   │
              │  flags (lookup_flags)     │
              └────────────┬──────────────┘
                           │ (كل line تُضاف كـ entry في gpiod_lookup_table)
                           ▼
              ┌──────────────────────────────────────┐
              │       gpiod_lookup_table             │
              │  dev_id = "gpio-aggregator.N"        │
              │  table[0] = {key, hwnum, NULL, 0, 0} │
              │  table[1] = {key, hwnum, NULL, 1, 0} │
              │  table[N] = {0}  (sentinel)          │
              └──────────────────────────────────────┘
                           │
                (gpiod_add_lookup_table → gpiolib global list)
                           │
                           ▼
              ┌──────────────────────────────────────┐
              │         platform_device              │
              │   name = "gpio-aggregator"           │
              │   id = aggr->id                      │
              │   drvdata → gpiochip_fwd (after probe)│
              └──────────────────────────────────────┘
                           │ probe
                           ▼
              ┌──────────────────────────────────────────────┐
              │           gpiochip_fwd                       │
              │  chip (gpio_chip) ─── registered in gpiolib  │
              │  descs[0] ──────────────→ gpio_desc (real)   │
              │  descs[1] ──────────────→ gpio_desc (real)   │
              │  descs[N] ──────────────→ gpio_desc (real)   │
              │  valid_mask (bitmap)                         │
              │  mlock or slock (حسب can_sleep)              │
              │  delay_timings[] (optional)                  │
              │  tmp[] flex: [bitmask values | desc ptrs]    │
              └──────────────────────────────────────────────┘
```

---

### Lifecycle Diagrams

#### أولاً: الـ Configfs Interface (المسار الجديد)

```
module_init()
    │
    ├─► config_group_init(&subsys.su_group)
    ├─► mutex_init(&subsys.su_mutex)
    ├─► configfs_register_subsystem()   → /sys/kernel/config/gpio-aggregator/
    └─► platform_driver_register()

mkdir /sys/kernel/config/gpio-aggregator/mychip
    │
    └─► gpio_aggregator_make_group()
            │
            ├─► gpio_aggregator_alloc()        → kzalloc + idr_alloc + mutex_init
            ├─► config_group_init_type_name()  → configfs node
            └─► dev_sync_probe_init()

mkdir /sys/kernel/config/gpio-aggregator/mychip/line0
    │
    └─► gpio_aggregator_device_make_group()
            │
            ├─► gpio_aggregator_line_alloc()   → kzalloc + kstrdup(key)
            ├─► config_group_init_type_name()
            └─► gpio_aggregator_line_add()     → list_add_tail (sorted by idx)

echo "gpiochip0" > .../line0/key
echo "5"         > .../line0/offset
echo "my-signal" > .../line0/name

echo 1 > .../mychip/live
    │
    └─► gpio_aggregator_device_live_store()
            │
            ├─► try_module_get()
            ├─► gpio_aggregator_lockup_configfs(lock=true)
            │       └─► configfs_depend_item_unlocked() per line
            └─► gpio_aggregator_activate()
                    │
                    ├─► gpio_aggregator_count_lines()    → must > 0
                    ├─► kzalloc(lookups table)
                    ├─► gpio_aggregator_make_device_sw_node()
                    │       └─► fwnode_create_software_node() [gpio-line-names]
                    ├─► loop lines → gpio_aggregator_add_gpio()
                    │       └─► krealloc(lookups) + GPIO_LOOKUP_IDX()
                    ├─► kasprintf(dev_id = "gpio-aggregator.N")
                    ├─► gpiod_add_lookup_table()
                    └─► dev_sync_probe_register()
                            └─► platform_device_register_full()
                                    │
                                    ▼
                            gpio_aggregator_probe()
                                    │
                                    ├─► gpiod_count()
                                    ├─► devm_gpiod_get_index() × N
                                    ├─► gpiochip_fwd_create()
                                    │       ├─► devm_gpiochip_fwd_alloc()
                                    │       ├─► gpiochip_fwd_desc_add() × N
                                    │       ├─► gpiochip_fwd_setup_delay_line() [optional]
                                    │       └─► gpiochip_fwd_register()
                                    │               └─► devm_gpiochip_add_data()
                                    └─► platform_set_drvdata(pdev, fwd)

echo 0 > .../mychip/live
    │
    └─► gpio_aggregator_deactivate()
            ├─► dev_sync_probe_unregister() → platform_device_unregister()
            │       └─► devm cleanup: gpiochip removed, descs released
            ├─► gpiod_remove_lookup_table()
            ├─► kfree(lookups->dev_id)
            └─► kfree(lookups)

rmdir /sys/kernel/config/gpio-aggregator/mychip/line0
    │
    └─► gpio_aggregator_line_release()
            ├─► gpio_aggregator_line_del()  → list_del
            ├─► kfree(key), kfree(name)
            └─► kfree(line)

rmdir /sys/kernel/config/gpio-aggregator/mychip
    │
    └─► gpio_aggregator_device_release()
            └─► gpio_aggregator_free()
                    ├─► idr_remove(&gpio_aggregator_idr, id)
                    ├─► mutex_destroy()
                    └─► kfree(aggr)
```

---

#### ثانياً: الـ Legacy Sysfs Interface

```
echo "gpiochip0 0-3" > /sys/bus/platform/drivers/gpio-aggregator/new_device
    │
    └─► gpio_aggregator_new_device_store()
            │
            ├─► try_module_get()
            ├─► gpio_aggregator_alloc(&aggr, count+1)
            │       └─► aggr->args = copy of buf
            ├─► aggr->init_via_sysfs = true
            ├─► kzalloc(lookups) + kasprintf(dev_id)
            ├─► config_group_init_type_name(&aggr->group, "_sysfs.N", ...)
            ├─► dev_sync_probe_init()
            ├─► configfs_register_group() → /sys/kernel/config/gpio-aggregator/_sysfs.N/
            ├─► gpio_aggregator_parse(aggr)
            │       ├─► next_arg() loop parsing "chip offsets" pairs
            │       ├─► gpio_aggregator_line_alloc() per line
            │       ├─► configfs_register_group() per line
            │       └─► gpio_aggregator_add_gpio() per line
            ├─► gpiod_add_lookup_table()
            └─► platform_device_register_data()
                    └─► (same probe path as above)

echo "gpio-aggregator.0" > /sys/bus/platform/drivers/gpio-aggregator/delete_device
    │
    └─► gpio_aggregator_delete_device_store()
            ├─► kstrtouint() → id
            ├─► idr_find() → aggr (must have init_via_sysfs == true)
            ├─► idr_remove()
            └─► gpio_aggregator_destroy()
                    ├─► gpio_aggregator_deactivate() [if active]
                    ├─► gpio_aggregator_free_lines()
                    │       └─► configfs_unregister_group() per line
                    ├─► configfs_unregister_group(&aggr->group)
                    └─► kfree(aggr)
```

---

### Call Flow Diagrams

#### قراءة قيمة GPIO واحدة

```
user calls gpiod_get_value(desc)
  │
  └─► [gpiolib] gpio_chip->get(chip, offset)
              = gpio_fwd_get(chip, offset)
                  │
                  ├─► gpiochip_get_data(chip)  → fwd
                  └─► chip->can_sleep ?
                          ├─ YES → gpiod_get_value_cansleep(fwd->descs[offset])
                          └─ NO  → gpiod_get_value(fwd->descs[offset])
                                          └─► real hardware register read
```

#### قراءة متعددة (get_multiple)

```
gpiolib calls chip->get_multiple(chip, mask, bits)
  = gpio_fwd_get_multiple_locked(chip, mask, bits)
      │
      ├─► gpiochip_get_data(chip) → fwd
      ├─► can_sleep?
      │     ├─ YES → mutex_lock(&fwd->mlock)
      │     └─ NO  → spin_lock_irqsave(&fwd->slock, flags)
      │
      └─► gpio_fwd_get_multiple(fwd, mask, bits)
              │
              ├─► descs = fwd_tmp_descs(fwd)        → temp array في tmp[]
              ├─► values = fwd_tmp_values(fwd)       → temp bitmask في tmp[]
              ├─► bitmap_clear(values, 0, ngpio)
              ├─► for_each_set_bit(i, mask):
              │       descs[j++] = fwd->descs[i]    → pack descs
              ├─► can_sleep?
              │     ├─ YES → gpiod_get_array_value_cansleep(j, descs, NULL, values)
              │     └─ NO  → gpiod_get_array_value(j, descs, NULL, values)
              └─► for_each_set_bit(i, mask):
                      __assign_bit(i, bits, test_bit(j++, values))
              │
      └─► unlock (mutex or spinlock)
```

#### كتابة قيمة مع delay

```
gpiolib calls chip->set(chip, offset, value)
  = gpio_fwd_set(chip, offset, value)
      │
      ├─► gpiochip_get_data(chip) → fwd
      ├─► can_sleep?
      │     ├─ YES → gpiod_set_value_cansleep(fwd->descs[offset], value)
      │     └─ NO  → gpiod_set_value(fwd->descs[offset], value)
      │                   └─► real hardware register write
      │
      └─► fwd->delay_timings != NULL ?
              └─► gpio_fwd_delay(chip, offset, value)
                      │
                      ├─► is_active_low = gpiod_is_active_low(fwd->descs[offset])
                      ├─► value XOR is_active_low → ramp_up أو ramp_down
                      └─► can_sleep?
                            ├─ YES → fsleep(delay_us)
                            └─ NO  → udelay(delay_us)
```

#### activation flow عند كتابة `1` في `live`

```
write "1" → configfs attr "live"
  └─► gpio_aggregator_device_live_store()
          │
          ├─► kstrtobool() → live=true
          ├─► try_module_get(THIS_MODULE)   [prevent unload race]
          ├─► !init_via_sysfs ?
          │       └─► gpio_aggregator_lockup_configfs(true)
          │               └─► configfs_depend_item_unlocked() per line
          │                       [يمنع rmdir أثناء تشغيل الـ device]
          │
          ├─► scoped_guard(mutex, &aggr->lock)
          │       ├─► is_activating? → return -EPERM
          │       ├─► already active? → return -EPERM
          │       └─► gpio_aggregator_activate(aggr)
          │               ├─► count_lines() > 0
          │               ├─► alloc lookups table
          │               ├─► make_device_sw_node() [gpio-line-names fwnode]
          │               ├─► fill gpiod_lookup entries
          │               ├─► kasprintf(dev_id)
          │               ├─► gpiod_add_lookup_table()
          │               └─► dev_sync_probe_register()
          │                       ├─► register bus notifier
          │                       ├─► platform_device_register_full()
          │                       └─► wait_for_completion() [sync probe]
          │
          └─► error? → gpio_aggregator_lockup_configfs(false)
              success? → configfs stays locked until live=0
```

---

### Locking Strategy

#### الـ Locks الموجودة

| الـ Lock | النوع | ما يحميه |
|---|---|---|
| `gpio_aggregator_lock` | `DEFINE_MUTEX` (global) | الـ `gpio_aggregator_idr` (IDR) |
| `aggr->lock` | `struct mutex` per-aggregator | state الـ aggregator: `probe_data.pdev`، الـ list، الـ `lookups` |
| `fwd->mlock` | `struct mutex` per-forwarder | الـ `tmp[]` flex array لما `can_sleep == true` |
| `fwd->slock` | `spinlock_t` per-forwarder | الـ `tmp[]` flex array لما `can_sleep == false` |
| `subsys->su_mutex` | جزء من configfs subsystem | configfs subsystem operations |

---

#### قواعد الـ Lock Ordering

**الترتيب الصح** لتجنب deadlock:

```
1. gpio_aggregator_lock    (global IDR lock)
      ↓
2. aggr->lock              (per-aggregator state lock)
      ↓
3. fwd->mlock / fwd->slock (per-forwarder tmp buffer lock)
```

**ملاحظات مهمة على الـ Locking:**

1. **`gpio_aggregator_free_lines()`** — لا تُمسك `aggr->lock` خارج الـ `scoped_guard` الداخلية لأن `configfs_unregister_group()` ممكن يطلب الـ lock نفسه، فلو حاملينه → **deadlock**. الكود يعلّق على ده صراحةً.

2. **`gpio_aggregator_remove_all()`** عند الـ module unload — **لا** تُمسك `gpio_aggregator_lock` حول `idr_for_each` لأن `gpio_aggregator_idr_remove` يوصل لـ configfs، وده يسبب **lock inversion**. الـ safety هنا عن طريق الـ `try_module_get` الـ mutual exclusion.

3. **Configfs depend/undepend** — يُنفَّذ **خارج** `aggr->lock` لأن `configfs_depend_item_unlocked` ممكن تأخد locks داخلية في configfs، وده يتعارض مع `aggr->lock`.

4. **`lockdep_assert_held(&aggr->lock)`** — موجودة في:
   - `gpio_aggregator_is_active()`
   - `gpio_aggregator_is_activating()`
   - `gpio_aggregator_count_lines()`
   - `gpio_aggregator_line_add()`
   - `gpio_aggregator_line_del()`

   ده يضمن إن المستخدمين دول دايماً بيشتغلوا تحت الـ lock.

5. **`fwd->slock` هو `spinlock`** → يُمسك مع `irqsave` لأن الـ GPIO operations ممكن تُستدعى من interrupt context لو `can_sleep == false`.

---

### ملخص العلاقات الكاملة (ASCII Overview)

```
/sys/kernel/config/gpio-aggregator/
└── <name>/                     ← gpio_aggregator.group
    ├── live                    ← يتحكم في activate/deactivate
    ├── dev_name                ← read-only: اسم الـ platform device
    └── line0/                  ← gpio_aggregator_line.group
        ├── key                 ← chip label أو line name
        ├── offset              ← hwnum أو -1
        └── name                ← virtual line name

gpio_aggregator_idr
    └── [id=N] → gpio_aggregator
                     ├── probe_data.pdev → platform_device "gpio-aggregator.N"
                     │                           └── drvdata → gpiochip_fwd
                     │                                              └── chip → /dev/gpiochipM
                     ├── lookups → gpiod_lookup_table (registered in gpiolib)
                     └── list_head → [line0] → [line1] → [line2] → ...

/sys/bus/platform/drivers/gpio-aggregator/
    ├── new_device    ← legacy: "chipLabel offset1,offset2"
    └── delete_device ← legacy: "gpio-aggregator.N"
```
## Phase 4: شرح الـ Functions

---

### ملخص الـ Functions — Cheatsheet

#### Category 1: Aggregator Lifecycle

| Function | Signature Summary | Purpose |
|---|---|---|
| `gpio_aggregator_alloc` | `(aggr**, size_t) → int` | تخصيص `gpio_aggregator` وتسجيله في الـ IDR |
| `gpio_aggregator_free` | `(aggr) → void` | حذف الـ aggregator من الـ IDR وتحرير الذاكرة |
| `gpio_aggregator_destroy` | `(aggr) → void` | deactivate + free lines + unregister configfs |
| `gpio_aggregator_activate` | `(aggr) → int` | إنشاء platform device وإضافة lookup table |
| `gpio_aggregator_deactivate` | `(aggr) → void` | حذف platform device وإزالة lookup table |

#### Category 2: Line Management

| Function | Signature Summary | Purpose |
|---|---|---|
| `gpio_aggregator_line_alloc` | `(parent, idx, key, offset) → line*` | تخصيص `gpio_aggregator_line` |
| `gpio_aggregator_line_add` | `(aggr, line) → void` | إدراج line في الـ list مرتبة حسب `idx` |
| `gpio_aggregator_line_del` | `(aggr, line) → void` | حذف line من الـ list |
| `gpio_aggregator_free_lines` | `(aggr) → void` | تحرير كل الـ lines في aggregator |
| `gpio_aggregator_add_gpio` | `(aggr, key, hwnum, n*) → int` | إضافة entry في الـ lookup table |

#### Category 3: State Queries

| Function | Signature Summary | Purpose |
|---|---|---|
| `gpio_aggregator_is_active` | `(aggr) → bool` | هل الـ platform device موجود ومتصل بـ driver؟ |
| `gpio_aggregator_is_activating` | `(aggr) → bool` | هل الـ pdev موجود لكن لسه ما اتـprobed؟ |
| `gpio_aggregator_count_lines` | `(aggr) → size_t` | عدد الـ lines المسجلة |

#### Category 4: GPIO Forwarder — Core

| Function | Signature Summary | Purpose |
|---|---|---|
| `devm_gpiochip_fwd_alloc` | `(dev, ngpios) → fwd*` | تخصيص forwarder بدون تسجيل |
| `gpiochip_fwd_desc_add` | `(fwd, desc, offset) → int` | إضافة `gpio_desc` لـ slot معين في الـ forwarder |
| `gpiochip_fwd_desc_free` | `(fwd, offset) → void` | إزالة وتحرير `gpio_desc` من الـ forwarder |
| `gpiochip_fwd_register` | `(fwd, data) → int` | تسجيل الـ gpiochip في الـ kernel |
| `gpiochip_fwd_create` | `(dev, ngpios, descs[], features) → fwd*` | alloc + add descs + register في خطوة واحدة |

#### Category 5: GPIO Forwarder — Operations (Public API)

| Function | Exported Symbol | Purpose |
|---|---|---|
| `gpiochip_fwd_get_gpiochip` | `GPIO_FORWARDER` | إرجاع الـ `gpio_chip` الداخلي |
| `gpiochip_fwd_get_data` | `GPIO_FORWARDER` | إرجاع الـ driver-private data |
| `gpiochip_fwd_gpio_request` | `GPIO_FORWARDER` | التحقق من صلاحية الـ offset |
| `gpiochip_fwd_gpio_get_direction` | `GPIO_FORWARDER` | قراءة اتجاه الـ line |
| `gpiochip_fwd_gpio_direction_input` | `GPIO_FORWARDER` | ضبط الـ line كـ input |
| `gpiochip_fwd_gpio_direction_output` | `GPIO_FORWARDER` | ضبط الـ line كـ output مع value |
| `gpiochip_fwd_gpio_get` | `GPIO_FORWARDER` | قراءة قيمة line واحدة |
| `gpiochip_fwd_gpio_get_multiple` | `GPIO_FORWARDER` | قراءة قيم عدة lines |
| `gpiochip_fwd_gpio_set` | `GPIO_FORWARDER` | كتابة قيمة line واحدة |
| `gpiochip_fwd_gpio_set_multiple` | `GPIO_FORWARDER` | كتابة قيم عدة lines |
| `gpiochip_fwd_gpio_set_config` | `GPIO_FORWARDER` | ضبط pinconf لـ line |
| `gpiochip_fwd_gpio_to_irq` | `GPIO_FORWARDER` | تحويل GPIO offset لـ IRQ number |

#### Category 6: Sysfs / Legacy Interface

| Function | Purpose |
|---|---|
| `gpio_aggregator_new_device_store` | معالج `new_device` sysfs — إنشاء aggregator من string |
| `gpio_aggregator_delete_device_store` | معالج `delete_device` sysfs — حذف aggregator بالاسم |
| `gpio_aggregator_parse` | تحليل الـ args string لـ legacy sysfs |

#### Category 7: Configfs Callbacks

| Function | Purpose |
|---|---|
| `gpio_aggregator_make_group` | إنشاء aggregator group جديد في configfs |
| `gpio_aggregator_device_make_group` | إنشاء line group جديد داخل aggregator |
| `gpio_aggregator_device_release` | تحرير aggregator عند حذف configfs item |
| `gpio_aggregator_line_release` | تحرير line عند حذف configfs item |
| `gpio_aggregator_lockup_configfs` | lock/unlock الـ configfs entries اللي الـ device يعتمد عليها |
| `gpio_aggregator_make_device_sw_node` | إنشاء software node يحمل `gpio-line-names` |

#### Category 8: Module Init/Exit

| Function | Purpose |
|---|---|
| `gpio_aggregator_init` | تسجيل configfs subsystem + platform driver |
| `gpio_aggregator_exit` | تنظيف كل الـ aggregators + إلغاء التسجيل |
| `gpio_aggregator_remove_all` | iterate على الـ IDR وحذف كل aggregator |
| `gpio_aggregator_idr_remove` | callback لـ `idr_for_each` — يحذف aggregator واحد |

---

### Group 1: Aggregator Lifecycle

**الـ group ده مسؤول عن إنشاء وتدمير الـ `gpio_aggregator` objects.** كل aggregator بيتعرف بـ unique ID جواه الـ IDR، وبيتكلم مع الـ platform device subsystem عشان ينشئ virtual GPIO chip.

---

#### `gpio_aggregator_alloc`

```c
static int gpio_aggregator_alloc(struct gpio_aggregator **aggr, size_t arg_size)
```

بتخصص `gpio_aggregator` struct مع flexible array للـ args (للـ legacy sysfs). بتسجله في الـ `gpio_aggregator_idr` وبتديه ID فريد. بتعمل initialize للـ mutex والـ list.

- **`aggr`**: مؤشر مؤشر — الـ out parameter، الـ struct الجديد بيتحط فيه.
- **`arg_size`**: حجم الـ flexible `args[]` array — صفر للـ configfs، `count + 1` للـ sysfs.
- **Returns**: `0` للنجاح، أو negative errno لو الـ allocation أو الـ IDR فشلوا.
- **Key Details**: بيستخدم `scoped_guard(mutex, &gpio_aggregator_lock)` للـ idr_alloc، وبيستخدم `no_free_ptr` عشان ينقل الـ ownership من الـ cleanup pointer للـ out param بدون double-free.

```
gpio_aggregator_alloc():
  kzalloc(sizeof(aggr) + arg_size)
  scoped_guard(mutex):
    idr_alloc() → id
  aggr->id = id
  INIT_LIST_HEAD(&aggr->list_head)
  mutex_init(&aggr->lock)
  *aggr = no_free_ptr(new)
```

---

#### `gpio_aggregator_free`

```c
static void gpio_aggregator_free(struct gpio_aggregator *aggr)
```

بتشيل الـ aggregator من الـ IDR وبتحرر الـ mutex والـ struct نفسه. مش بتعمل cleanup للـ lines أو الـ configfs — ده مسؤولية الـ caller.

- **`aggr`**: الـ aggregator المراد تحريره.
- **Returns**: void.
- **Key Details**: الـ mutex acquisition للـ IDR بالـ `scoped_guard`. الـ caller لازم يكون شال الـ lines قبل ما يكال الـ function دي.

---

#### `gpio_aggregator_activate`

```c
static int gpio_aggregator_activate(struct gpio_aggregator *aggr)
```

دي الـ function الأساسية اللي بتحول aggregator من configured لـ live device. بتنشئ `gpiod_lookup_table` وبتملأها من الـ lines list، بتنشئ software node يحمل line names، وبتسجل platform device. بتستخدم `dev_sync_probe_register` عشان تنتظر الـ probe يخلص synchronously.

- **`aggr`**: الـ aggregator المراد تفعيله، لازم يكون مقفول بـ `aggr->lock`.
- **Returns**: `0` للنجاح، أو negative errno.
- **Key Details**:
  - بتتحقق إن الـ lines count > 0 وإن كل line عندها `key` وإن الـ indices متسلسلة من 0.
  - الـ `line->offset == -1` يعني lookup by line name (بيبعت `U16_MAX` للـ gpiolib).
  - لو حصل error، بتعمل cleanup كامل لكل اللي اتعمل (lookup table, swnode, lookups).
  - الـ caller (configfs `live` attribute أو `new_device_store`) لازم يكون شايل الـ `aggr->lock`.

```
gpio_aggregator_activate():
  count lines → error if 0
  kzalloc lookup table
  gpio_aggregator_make_device_sw_node() → swnode
  for each line (in order):
    validate line->key != NULL && line->idx == n
    gpio_aggregator_add_gpio(aggr, key, offset, &n)
  kasprintf dev_id = "gpio-aggregator.N"
  gpiod_add_lookup_table()
  dev_sync_probe_register() → creates pdev, waits for probe
  return 0
  --- error paths ---
  err_remove_lookup_table: remove table
  err_remove_swnode: fwnode_remove_software_node
  err_remove_lookups: kfree lookups
```

---

#### `gpio_aggregator_deactivate`

```c
static void gpio_aggregator_deactivate(struct gpio_aggregator *aggr)
```

عكس الـ activate — بتشيل الـ platform device وبتزيل الـ lookup table. `dev_sync_probe_unregister` بتنتظر الـ probe ينتهي لو كان شغال. الـ caller لازم يكون شايل `aggr->lock`.

- **`aggr`**: الـ aggregator المراد إيقافه.
- **Key Details**: ترتيب الـ cleanup مهم — أول حاجة الـ pdev بعدين الـ lookup table بعدين التحرير.

---

#### `gpio_aggregator_destroy`

```c
static void gpio_aggregator_destroy(struct gpio_aggregator *aggr)
```

الـ full teardown للـ sysfs-created aggregators. بتعمل deactivate لو active، بتحرر الـ lines، بتشيل الـ configfs group، وبتحرر الـ struct. مش بتُستخدم للـ configfs-created aggregators اللي الـ `device_release` callback بيتكفل بيها.

---

### Group 2: Line Management

**الـ lines هي اللبنة الأساسية للـ aggregator. كل line بتمثل GPIO واحد حقيقي.**

---

#### `gpio_aggregator_line_alloc`

```c
static struct gpio_aggregator_line *
gpio_aggregator_line_alloc(struct gpio_aggregator *parent, unsigned int idx,
                           char *key, int offset)
```

بتخصص `gpio_aggregator_line` وبتحدد الـ parent والـ index والـ key والـ offset. بتعمل `kstrdup` للـ key لو موجود.

- **`parent`**: الـ aggregator المالك.
- **`idx`**: الترتيب الرقمي للـ line جوه الـ aggregator (0, 1, 2, ...).
- **`key`**: اسم الـ GPIO chip أو الـ line name — ممكن يكون NULL (للـ configfs path اللي الـ key بييجي لاحقاً).
- **`offset`**: الـ hardware offset جوه الـ chip، أو `-1` للـ lookup by name.
- **Returns**: pointer للـ line الجديدة، أو `ERR_PTR(-ENOMEM)`.

---

#### `gpio_aggregator_line_add`

```c
static void gpio_aggregator_line_add(struct gpio_aggregator *aggr,
                                     struct gpio_aggregator_line *line)
```

بتدرج line في الـ `aggr->list_head` بترتيب تصاعدي حسب `line->idx`. ده ضروري عشان الـ activate يقدر يتحقق من التسلسل ببساطة.

- **Key Details**: `lockdep_assert_held(&aggr->lock)` — لازم يكال تحت `aggr->lock`.

---

#### `gpio_aggregator_add_gpio`

```c
static int gpio_aggregator_add_gpio(struct gpio_aggregator *aggr,
                                    const char *key, int hwnum, unsigned int *n)
```

بتوسع الـ `gpiod_lookup_table` بـ entry جديد باستخدام `krealloc`. بتستخدم `GPIO_LOOKUP_IDX` macro لملء الـ entry. دايماً بتحافظ على zero terminator بعد آخر entry.

- **`key`**: اسم الـ GPIO chip أو line name.
- **`hwnum`**: الـ hardware number، أو `U16_MAX` للـ lookup by name.
- **`n`**: الـ counter الحالي، بيتزود بعد الإضافة.
- **Returns**: `0` للنجاح، `ENOMEM` لو الـ krealloc فشل.

---

#### `gpio_aggregator_free_lines`

```c
static void gpio_aggregator_free_lines(struct gpio_aggregator *aggr)
```

بتمشي على كل الـ lines وبتعمل `configfs_unregister_group` لكل واحدة، بعدين بتشيلها من الـ list وبتحرر الـ key والـ name والـ struct. بتستخدم `list_for_each_entry_safe` عشان تتعامل مع الـ deletion during iteration.

- **Key Details**: بتمسك `aggr->lock` بشكل minimal داخل الـ `scoped_guard` لتجنب deadlock مع الـ configfs internal locks، لأن `configfs_unregister_group` ممكن يطلب locks تانية.

---

### Group 3: GPIO Forwarder — Core

**الـ forwarder هو الـ virtual `gpio_chip` اللي بيعمل delegate لكل عمليات الـ GPIO للـ real descriptors.**

---

#### `devm_gpiochip_fwd_alloc`

```c
struct gpiochip_fwd *devm_gpiochip_fwd_alloc(struct device *dev,
                                              unsigned int ngpios)
```

بتخصص `gpiochip_fwd` struct مع الـ flexible `tmp[]` array (بيحتوي على bitmap values + descs pointers للـ multi-GPIO ops)، وبتخصص الـ `descs` array والـ `valid_mask` bitmap. بتملأ كل الـ `gpio_chip` callbacks بالـ forwarder functions، وبتضبط `chip->base = -1` عشان الـ kernel يختار base تلقائياً.

- **`dev`**: الـ parent device — الـ devm allocations بتتربط بيه.
- **`ngpios`**: عدد الـ GPIO lines في الـ forwarder.
- **Returns**: pointer للـ `gpiochip_fwd` الجديد، أو `ERR_PTR(-ENOMEM)`.
- **Key Details**:
  - الـ `tmp[]` size = `BITS_TO_LONGS(ngpios)` + `ngpios` longs.
  - الـ chip callbacks بتتضبط هنا: `request`, `get_direction`, `direction_input/output`, `get`, `get_multiple`, `set`, `set_multiple`, `set_config`, `to_irq`.
  - الـ locking (mutex vs spinlock) بيتحدد بعدين في `gpiochip_fwd_register`.

---

#### `gpiochip_fwd_desc_add`

```c
int gpiochip_fwd_desc_add(struct gpiochip_fwd *fwd, struct gpio_desc *desc,
                           unsigned int offset)
```

بتضيف `gpio_desc` لـ slot معين في الـ forwarder. بتتحقق إن الـ offset في الحدود وإنه مش occupied قبل كده. لو الـ desc محتاج sleep، بتضبط `chip->can_sleep = true` عشان الـ whole forwarder يبقى sleeping.

- **`fwd`**: الـ forwarder.
- **`desc`**: الـ GPIO descriptor الحقيقي.
- **`offset`**: الـ virtual offset في الـ forwarder.
- **Returns**: `0` للنجاح، `-EINVAL` لو offset خارج الحدود، `-EEXIST` لو slot مشغول.
- **Key Details**: الـ `chip->can_sleep` بيبقى one-way flag — لو أي line محتاجة sleep، الكل بيبقى can_sleep.

---

#### `gpiochip_fwd_desc_free`

```c
void gpiochip_fwd_desc_free(struct gpiochip_fwd *fwd, unsigned int offset)
```

بتمسح bit من الـ `valid_mask` وبتعمل `gpiod_put` للـ descriptor لو كان موجود. ده important للـ dynamic line registration scenarios.

---

#### `gpiochip_fwd_register`

```c
int gpiochip_fwd_register(struct gpiochip_fwd *fwd, void *data)
```

بتكمل إعداد الـ forwarder وبتسجله كـ gpiochip فعلي في الـ kernel. لو مش كل الـ descs متسجلة (يعني الـ valid_mask مش full)، بتضبط `can_sleep = true` عشان الـ runtime additions ممكن تيجي من sleeping context. بعدين بتعمل initialize للـ مناسب من mutex أو spinlock، وأخيراً بتكال `devm_gpiochip_add_data`.

- **`data`**: driver-private data محفوظة في `fwd->data`.
- **Returns**: `0` للنجاح، أو negative errno من `devm_gpiochip_add_data`.
- **Key Details**: الـ locking strategy (mutex لو `can_sleep`، spinlock لو !`can_sleep`) بتتحدد هنا نهائياً. لو `can_sleep` بيتغير بعد kده (مش expected)، الـ spinlock يبقى غلط — ده design constraint.

---

#### `gpiochip_fwd_create` (static)

```c
static struct gpiochip_fwd *gpiochip_fwd_create(struct device *dev,
                                                 unsigned int ngpios,
                                                 struct gpio_desc *descs[],
                                                 unsigned long features)
```

الـ convenience wrapper اللي بيعمل alloc + add + optional delay setup + register في sequence واحدة. بتستخدمها الـ `gpio_aggregator_probe`.

- **`features`**: bitmask من `FWD_FEATURE_*` — حالياً بس `FWD_FEATURE_DELAY`.
- **Returns**: `fwd*` للنجاح أو `ERR_PTR`.

```
gpiochip_fwd_create():
  devm_gpiochip_fwd_alloc(dev, ngpios)
  for i in 0..ngpios:
    gpiochip_fwd_desc_add(fwd, descs[i], i)
  if FWD_FEATURE_DELAY:
    gpiochip_fwd_setup_delay_line(fwd)  /* CONFIG_OF_GPIO only */
  gpiochip_fwd_register(fwd, NULL)
  return fwd
```

---

### Group 4: GPIO Forwarder — Internal Chip Callbacks

**دي الـ static functions اللي بتتسجل كـ `gpio_chip` callbacks. ما بيتكلموش مباشرة من users — بيتكلموا من الـ gpiolib core.**

---

#### `gpio_fwd_request`

```c
static int gpio_fwd_request(struct gpio_chip *chip, unsigned int offset)
```

بتتحقق إن الـ bit موجود في الـ `valid_mask` — لو مش موجود معناه إن الـ desc مش متسجل وبترجع `-ENODEV`. ده بيسمح للـ forwarder إنه يكون sparse (مش كل الـ slots مملوءة).

---

#### `gpio_fwd_get_direction`

```c
static int gpio_fwd_get_direction(struct gpio_chip *chip, unsigned int offset)
```

بتيجب direction الـ underlying desc. بتتحقق من الـ `valid_mask` أول لأنها بتتكال أثناء الـ gpiochip registration نفسها (قبل ما كل الـ descs يتضافوا). بترجع `0` للـ output، `1` للـ input.

---

#### `gpio_fwd_get` / `gpio_fwd_set`

```c
static int gpio_fwd_get(struct gpio_chip *chip, unsigned int offset)
static int gpio_fwd_set(struct gpio_chip *chip, unsigned int offset, int value)
```

الـ `get` بتختار بين `gpiod_get_value_cansleep` و `gpiod_get_value` حسب `chip->can_sleep`. الـ `set` كمان، وبعد الكتابة لو `fwd->delay_timings` موجود بتكال `gpio_fwd_delay` للـ ramp timing.

---

#### `gpio_fwd_get_multiple` (helper)

```c
static int gpio_fwd_get_multiple(struct gpiochip_fwd *fwd, unsigned long *mask,
                                 unsigned long *bits)
```

بتستخدم الـ `tmp[]` array الـ preallocated كـ temporary buffer للـ descriptors والـ values. بتعمل compact للـ masked descs في array متراصة، بتكال `gpiod_get_array_value[_cansleep]`، وبعدين بتوزع النتائج تاني على الـ bits bitmap.

- **Key Details**: الـ `fwd_tmp_descs` و `fwd_tmp_values` macros بتحسب الـ offsets داخل الـ `tmp[]` flexible array تلقائياً حسب `ngpios`.

---

#### `gpio_fwd_get_multiple_locked` / `gpio_fwd_set_multiple_locked`

```c
static int gpio_fwd_get_multiple_locked(struct gpio_chip *chip,
                                        unsigned long *mask, unsigned long *bits)
static int gpio_fwd_set_multiple_locked(struct gpio_chip *chip,
                                        unsigned long *mask, unsigned long *bits)
```

الـ wrappers اللي بتمسك الـ lock المناسب (mutex لو `can_sleep`، spinlock مع irqsave لو !`can_sleep`) قبل ما تكال الـ `gpio_fwd_get_multiple` أو `gpio_fwd_set_multiple`. الـ lock لازم عشان الـ `tmp[]` buffer shared بين كل الـ callers.

---

#### `gpio_fwd_delay`

```c
static void gpio_fwd_delay(struct gpio_chip *chip, unsigned int offset, int value)
```

بتحسب هل المفروض ramp-up أو ramp-down delay حسب الـ value والـ `ACTIVE_LOW` flag. بتستخدم `fsleep` (للـ sleeping context) أو `udelay` (للـ atomic context). بتُستخدم مع الـ `gpio-delay` DT binding.

---

#### `gpio_fwd_set_config` / `gpio_fwd_to_irq`

```c
static int gpio_fwd_set_config(struct gpio_chip *chip, unsigned int offset,
                                unsigned long config)
static int gpio_fwd_to_irq(struct gpio_chip *chip, unsigned int offset)
```

تفويض مباشر للـ `gpiod_set_config` و `gpiod_to_irq` على الـ underlying descriptor. لا locking لأنهم thread-safe من جهة gpiolib.

---

#### `gpiochip_fwd_delay_of_xlate` (CONFIG_OF_GPIO)

```c
static int gpiochip_fwd_delay_of_xlate(struct gpio_chip *chip,
                                        const struct of_phandle_args *gpiospec,
                                        u32 *flags)
```

الـ custom `of_xlate` callback للـ `gpio-delay` DT node. بتستقبل 3 cells: `<line ramp_up_us ramp_down_us>`. بتملأ `fwd->delay_timings[line]` وبترجع الـ line number.

---

#### `gpiochip_fwd_setup_delay_line` (CONFIG_OF_GPIO)

```c
static int gpiochip_fwd_setup_delay_line(struct gpiochip_fwd *fwd)
```

بتخصص الـ `delay_timings` array وبتضبط `chip->of_xlate` و `chip->of_gpio_n_cells = 3`. النسخة اللي مفيهاش `CONFIG_OF_GPIO` stub بترجع `0` فوراً.

---

### Group 5: Configfs Interface

**الـ configfs بيسمح لـ userspace ينشئ aggregators بشكل declarative عن طريق mkdir وكتابة attributes.**

---

#### `gpio_aggregator_make_group`

```c
static struct config_group *
gpio_aggregator_make_group(struct config_group *group, const char *name)
```

بتتكال لما يعمل user `mkdir /sys/kernel/config/gpio-aggregator/<name>`. بتتحقق إن الاسم ما بيبدأش بـ `_sysfs` (محجوز للـ legacy path). بتعمل alloc للـ aggregator، بتعمل initialize الـ `probe_data`، وبترجع الـ config_group.

- **Returns**: `config_group*` للنجاح، `ERR_PTR(-EINVAL)` لو الاسم محجوز.
- **Caller Context**: يتكال من configfs subsystem تحت الـ configfs subsystem mutex.

---

#### `gpio_aggregator_device_make_group`

```c
static struct config_group *
gpio_aggregator_device_make_group(struct config_group *group, const char *name)
```

بتتكال لما يعمل user `mkdir /sys/kernel/config/gpio-aggregator/<name>/line<N>`. بـ parse الاسم للـ format `line<uint>`. بتتحقق إن الـ aggregator مش active، وإن الـ idx مش مكرر. بتخصص الـ line وبتضيفها للـ list.

- **Key Details**: الـ aggregator اللي اتعمل عن طريق sysfs بيرفض الـ mkdir (`-EPERM`) — الـ lines بتاعته اتعملت في الـ `gpio_aggregator_parse`.

---

#### `gpio_aggregator_line_release` / `gpio_aggregator_device_release`

```c
static void gpio_aggregator_line_release(struct config_item *item)
static void gpio_aggregator_device_release(struct config_item *item)
```

الـ configfs callbacks اللي بتتكال عند `rmdir`. الـ `line_release` بتشيل الـ line من الـ list وبتحرر الـ key والـ name والـ struct. الـ `device_release` بتكال `gpio_aggregator_free` مباشرة لأن الـ deactivation حصل قبل كده.

---

#### `gpio_aggregator_lockup_configfs`

```c
static void gpio_aggregator_lockup_configfs(struct gpio_aggregator *aggr,
                                             bool lock)
```

لما الـ device بيتفعل (`live = 1`)، لازم نمنع حذف الـ configfs entries اللي الـ device بيعتمد عليها. بتستخدم `configfs_depend_item_unlocked` على كل line لتأمينها، و `configfs_undepend_item_unlocked` عند الإيقاف. بتشتغل على الـ leaf lines فقط لأن ده كافي لتأمين الـ hierarchy بالكامل.

---

#### `gpio_aggregator_make_device_sw_node`

```c
static struct fwnode_handle *
gpio_aggregator_make_device_sw_node(struct gpio_aggregator *aggr)
```

بتبني `property_entry` يحمل `gpio-line-names` string array من الـ names المضبوطة للـ lines. الـ software node ده بيتحط في الـ `pdevinfo.fwnode` عشان يظهر في الـ device properties. ده بيسمح للـ probe يلاقي الـ line names من الـ firmware node بدون DT.

- **Returns**: `fwnode_handle*` للنجاح، `NULL` لو ما فيش lines، `ERR_PTR(-ENOMEM)` لو الـ allocation فشل.

---

#### Configfs Attributes — Line

| Attribute | show | store | Constraints |
|---|---|---|---|
| `key` | يرجع `line->key` | يضبط الـ chip label أو line name | مرفوض لو active/activating |
| `name` | يرجع `line->name` | يضبط الـ virtual line name | مرفوض لو active/activating |
| `offset` | يرجع `line->offset` | يضبط الـ hardware offset (-1 لـ name lookup) | مرفوض لو active/activating, max U16_MAX-1 |

#### `gpio_aggregator_line_key_store` / `gpio_aggregator_line_name_store`

```c
static ssize_t gpio_aggregator_line_key_store(struct config_item *item,
                                               const char *page, size_t count)
static ssize_t gpio_aggregator_line_name_store(struct config_item *item,
                                                const char *page, size_t count)
```

بتعمل `kstrndup` + `strim` للـ input، بتمسك `aggr->lock`، بتتحقق إن الـ device مش active، وبتحدث الـ field. بتستخدم `no_free_ptr` لنقل الـ ownership.

---

#### Configfs Attributes — Device

| Attribute | Type | Purpose |
|---|---|---|
| `dev_name` | RO | اسم الـ platform device (`gpio-aggregator.N`) |
| `live` | RW | `1` لتفعيل الـ device، `0` لإيقافه |

#### `gpio_aggregator_device_live_store`

```c
static ssize_t gpio_aggregator_device_live_store(struct config_item *item,
                                                   const char *page, size_t count)
```

أهم configfs attribute في الـ driver. بتعمل `kstrtobool` للـ input، بتستخدم `try_module_get` لمنع الـ module unload أثناء التفعيل، وبتكال `gpio_aggregator_activate` أو `gpio_aggregator_deactivate`. الـ `gpio_aggregator_lockup_configfs` بتتكال قبل/بعد التفعيل لتأمين الـ dependencies.

```
live_store(live=1):
  try_module_get()
  gpio_aggregator_lockup_configfs(lock=true)  /* if not sysfs */
  scoped_guard(mutex, &aggr->lock):
    if activating or already active → -EPERM
    gpio_aggregator_activate()
  if activate failed:
    gpio_aggregator_lockup_configfs(lock=false)
  module_put()

live_store(live=0):
  try_module_get()
  scoped_guard(mutex, &aggr->lock):
    if activating or not active → -EPERM
    gpio_aggregator_deactivate()
  if deactivate ok:
    gpio_aggregator_lockup_configfs(lock=false)
  module_put()
```

---

### Group 6: Legacy Sysfs Interface

**الواجهة القديمة قبل configfs — بتسمح إنشاء aggregator بـ string واحدة.**

---

#### `gpio_aggregator_parse`

```c
static int gpio_aggregator_parse(struct gpio_aggregator *aggr)
```

بتحلل الـ `aggr->args` string اللي جاية من `new_device_store`. الـ format: `<chip_label> <offset_list> [<chip_label> <offset_list>]...` أو `<line_name> <line_name>...`. بتستخدم `next_arg` لتقطيع الـ tokens وبتفرق بين GPIO chip + offsets وـ named GPIO lines.

- **Algorithm**:
  1. جيب أول token كـ key.
  2. لكل token جاي، لو `get_options` بيرجع error أو بيسيب remainder → ده line name مش offset list.
  3. لو offset list → `bitmap_parselist` وبعدين loop على كل set bit.
  4. لكل GPIO: `gpio_aggregator_line_alloc` + `configfs_register_group` + `gpio_aggregator_add_gpio`.
- **Returns**: `0` للنجاح، أو negative errno مع cleanup.

---

#### `gpio_aggregator_new_device_store`

```c
static ssize_t gpio_aggregator_new_device_store(struct device_driver *driver,
                                                 const char *buf, size_t count)
```

الـ handler للـ `/sys/bus/platform/drivers/gpio-aggregator/new_device`. بتعمل:
1. `try_module_get` لمنع unload.
2. تخصيص aggregator مع `count+1` bytes للـ args.
3. Copy الـ buf للـ args.
4. إنشاء lookup table مع dev_id.
5. تسجيل configfs group (باسم `_sysfs.N`).
6. `gpio_aggregator_parse` لتحليل الـ args وإنشاء الـ lines.
7. `gpiod_add_lookup_table`.
8. `platform_device_register_data` لإنشاء الـ pdev.

- **Key Details**: الـ `_sysfs` prefix يمنع أي تلاعب من configfs. الـ `init_via_sysfs = true` flag بتغير behavior الـ probe وتمنع deferred probing.

---

#### `gpio_aggregator_delete_device_store`

```c
static ssize_t gpio_aggregator_delete_device_store(struct device_driver *driver,
                                                    const char *buf, size_t count)
```

الـ handler لـ `/sys/bus/platform/drivers/gpio-aggregator/delete_device`. بتستقبل `gpio-aggregator.N` string، بتـparse الـ ID، وبتبحث في الـ IDR. بتتحقق إن الـ aggregator اتعمل عن طريق sysfs (لا configfs). بتشيله من الـ IDR وبتكال `gpio_aggregator_destroy`.

- **Key Details**: الـ IDR removal بيحصل قبل الـ `gpio_aggregator_destroy` عشان يمنع أي وصول تاني للـ object ده من IDR lookups.

---

### Group 7: Platform Driver — Probe

---

#### `gpio_aggregator_probe`

```c
static int gpio_aggregator_probe(struct platform_device *pdev)
```

الـ probe function للـ `gpio-aggregator` platform driver. بتتكال لما يتسجل platform device (سواء من الـ new_device_store أو من الـ configfs live=1 أو من الـ DT).

```
gpio_aggregator_probe():
  n = gpiod_count(dev, NULL)          /* عدد الـ GPIOs المحددة في lookup table */
  descs = devm_kmalloc_array(n)
  meta = dev_get_platdata()
  for i in 0..n:
    descs[i] = devm_gpiod_get_index(dev, NULL, i, GPIOD_ASIS)
    /* لو configfs + EPROBE_DEFER → -ENODEV (no deferral) */
  features = device_get_match_data()  /* من DT match data */
  fwd = gpiochip_fwd_create(dev, n, descs, features)
  platform_set_drvdata(pdev, fwd)     /* marks device as "active" */
  devm_kfree(descs)                   /* فاضي دلوقتي — fwd عنده كوبيته */
  return 0
```

- **`meta->init_via_sysfs`**: لو `true`، الـ EPROBE_DEFER بيتمرر للـ caller (backward compat).
- **Key Details**: الـ `platform_set_drvdata(pdev, fwd)` ده اللي بيخلي `gpio_aggregator_is_active()` يرجع `true`. لو الـ probe فشل، الـ `drvdata` بيبقى NULL وهو معناه "activating" لو الـ pdev موجود.

---

### Group 8: Module Init/Exit

---

#### `gpio_aggregator_init`

```c
static int __init gpio_aggregator_init(void)
```

بتعمل:
1. `config_group_init` + `mutex_init` للـ configfs subsystem.
2. `configfs_register_subsystem` لإنشاء `/sys/kernel/config/gpio-aggregator/`.
3. `platform_driver_register` لتسجيل الـ driver (وبالتالي expose الـ sysfs attributes).

- **Key Details**: الترتيب مهم جداً — configfs لازم يتسجل أول قبل الـ driver attributes تظهر، عشان لو user وصل `new_device_store` قبل ما configfs يكون جاهز، هيحصل race condition.

---

#### `gpio_aggregator_exit`

```c
static void __exit gpio_aggregator_exit(void)
```

عكس الـ init بالترتيب الصح:
1. `gpio_aggregator_remove_all` — تحذف كل الـ sysfs-created aggregators.
2. `platform_driver_unregister`.
3. `configfs_unregister_subsystem`.

---

#### `gpio_aggregator_remove_all`

```c
static void __exit gpio_aggregator_remove_all(void)
```

بتكال `idr_for_each` مع `gpio_aggregator_idr_remove` callback على كل entry. مش بتمسك `gpio_aggregator_lock` أثناء الـ iteration عمداً — لأن `gpio_aggregator_idr_remove` بتوصل للـ configfs groups (اللي محتاجة locks تانية)، فلو مسكنا الـ lock هيحصل lock inversion. الـ safety بتيجي من إن الـ module unload path والـ new_device/delete_device path ما بيجروش مع بعض (بسبب `try_module_get`).

---

### ملاحظات على الـ Locking Strategy

```
Locks في الـ Driver:
┌─────────────────────────────────────────────────────────┐
│ gpio_aggregator_lock (global mutex)                     │
│   └── يحمي: gpio_aggregator_idr                        │
│                                                         │
│ aggr->lock (per-aggregator mutex)                       │
│   └── يحمي: aggr->list_head, aggr->probe_data.pdev     │
│   └── لازم يتمسك أثناء: activate, deactivate, is_*    │
│   └── ما يتمسكش أثناء: configfs_register/unregister   │
│                                                         │
│ fwd->mlock (mutex, لو can_sleep)                        │
│ fwd->slock (spinlock + irqsave, لو !can_sleep)         │
│   └── يحمي: fwd->tmp[] (shared buffer للـ multi-ops)  │
└─────────────────────────────────────────────────────────┘

Lock Ordering:
  gpio_aggregator_lock → aggr->lock (لو محتاجين الاتنين)
  aggr->lock → fwd->mlock/slock (ما بيحصلش في نفس path)
```

---

### Public API Summary (EXPORT_SYMBOL_NS_GPL)

كل الـ functions دي exported في namespace `GPIO_FORWARDER` وبتُستخدم من drivers تانية عايزة تبني forwarder مخصص:

| Function | استخدامها |
|---|---|
| `devm_gpiochip_fwd_alloc` | اول خطوة: تخصيص |
| `gpiochip_fwd_desc_add` | إضافة descs dynamically |
| `gpiochip_fwd_desc_free` | إزالة desc وقت runtime |
| `gpiochip_fwd_register` | تسجيل نهائي كـ gpiochip |
| `gpiochip_fwd_get_gpiochip` | الوصول للـ chip struct |
| `gpiochip_fwd_get_data` | الوصول للـ private data |
| `gpiochip_fwd_gpio_*` | الـ operations API الكامل |
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. مدخلات الـ debugfs

الـ GPIO subsystem بيكشف معلومات تفصيلية عبر `debugfs`. الـ gpio-aggregator نفسه مش بيضيف entries خاصة بيه، لكن بيعتمد على entries الـ GPIO العامة:

```bash
# mount debugfs لو مش mounted
mount -t debugfs none /sys/kernel/debug

# شوف كل الـ GPIO chips المسجلة (بما فيها الـ aggregator الـ virtual chip)
cat /sys/kernel/debug/gpio

# مثال على الـ output
# gpiochip5: GPIOs 496-499, parent: platform/gpio-aggregator.0, gpio-aggregator.0:
#  gpio-496 (                    |?) out hi ACTIVE HIGH
#  gpio-497 (                    |?) in  lo ACTIVE HIGH
```

```bash
# شوف الـ pinctrl state لو الـ parent chip مرتبط بـ pinctrl
ls /sys/kernel/debug/pinctrl/
cat /sys/kernel/debug/pinctrl/<chip>/pinconf-pins
```

**الـ configfs entries** — الجزء الأهم في الـ aggregator:

```bash
# شوف كل الـ aggregators المعرفة عبر configfs
ls /sys/kernel/config/gpio-aggregator/

# شوف lines داخل aggregator معين
ls /sys/kernel/config/gpio-aggregator/<aggr-name>/

# اقرأ خصائص line معينة
cat /sys/kernel/config/gpio-aggregator/<aggr-name>/<line-name>/key
cat /sys/kernel/config/gpio-aggregator/<aggr-name>/<line-name>/offset
cat /sys/kernel/config/gpio-aggregator/<aggr-name>/<line-name>/name
cat /sys/kernel/config/gpio-aggregator/<aggr-name>/<line-name>/active_low
```

---

#### 2. مدخلات الـ sysfs

```bash
# الـ aggregator بيظهر كـ platform device
ls /sys/bus/platform/devices/ | grep gpio-aggregator

# شوف attributes الـ driver
ls /sys/bus/platform/drivers/gpio-aggregator/

# الـ legacy interface: إنشاء وحذف devices عبر sysfs
# (الـ new_device و delete_device attributes)
cat /sys/bus/platform/drivers/gpio-aggregator/new_device 2>/dev/null || echo "write-only"

# شوف الـ gpiochip الـ virtual المرتبط بالـ aggregator device
ls /sys/bus/platform/devices/gpio-aggregator.0/gpio/

# شوف direction وvalue لكل line في الـ virtual chip
cat /sys/class/gpio/gpiochip496/label
cat /sys/class/gpio/gpiochip496/ngpio
cat /sys/class/gpio/gpiochip496/base

# export line وقرأ قيمتها
echo 496 > /sys/class/gpio/export
cat /sys/class/gpio/gpio496/value
cat /sys/class/gpio/gpio496/direction
echo 496 > /sys/class/gpio/unexport
```

---

#### 3. الـ ftrace — Tracepoints والـ Events

```bash
# شوف الـ GPIO tracepoints المتاحة
ls /sys/kernel/debug/tracing/events/gpio/

# فعّل كل GPIO events
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable

# أو فعّل events محددة
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_direction/enable
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_value/enable

# ابدأ الـ trace
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe &

# شغّل العملية اللي عايز تتبعها
gpioget gpiochip5 0

# وقف الـ trace
echo 0 > /sys/kernel/debug/tracing/tracing_on

# اقرأ النتيجة
cat /sys/kernel/debug/tracing/trace
```

**استخدام function_graph tracer لتتبع دوال الـ forwarder:**

```bash
echo function_graph > /sys/kernel/debug/tracing/current_tracer

# فلتر على دوال الـ gpio_fwd فقط
echo 'gpio_fwd*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo 'gpiochip_fwd*' >> /sys/kernel/debug/tracing/set_ftrace_filter

echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... شغّل العملية ...
cat /sys/kernel/debug/tracing/trace
echo nop > /sys/kernel/debug/tracing/current_tracer
```

---

#### 4. الـ printk والـ Dynamic Debug

الـ driver بيستخدم `pr_fmt` مع prefix `"gpio-aggregator: "` وبيستخدم `dev_dbg()` في `gpiochip_fwd_desc_add()`.

```bash
# فعّل dynamic debug لكل الـ gpio-aggregator messages
echo 'module gpio_aggregator +p' > /sys/kernel/debug/dynamic_debug/control

# أو فعّل بناءً على الملف
echo 'file gpio-aggregator.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل مع line numbers وfunction names
echo 'file gpio-aggregator.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ rules المفعلة الحالية
cat /sys/kernel/debug/dynamic_debug/control | grep gpio-aggregator

# الـ dev_dbg في gpiochip_fwd_desc_add بيطبع:
# "gpio-aggregator.0: 0 => gpio 496 irq -1"
# لكل line مضافة للـ forwarder
```

**عبر kernel cmdline (boot time):**

```
dyndbg="file gpio-aggregator.c +p"
```

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الغرض |
|---|---|
| `CONFIG_GPIO_AGGREGATOR` | تفعيل الـ driver أصلاً |
| `CONFIG_DEBUG_GPIO` | debug checks + verbose GPIO errors |
| `CONFIG_GPIO_CDEV_V1` | legacy chardev support للـ testing |
| `CONFIG_GPIOLIB_FASTPATH_LIMIT` | يتحكم في الـ fast path threshold |
| `CONFIG_DEBUG_FS` | لازم يكون مفعل عشان debugfs يشتغل |
| `CONFIG_DYNAMIC_DEBUG` | لتفعيل `dev_dbg()` messages |
| `CONFIG_LOCKDEP` | يكشف lock ordering violations (الـ code فيه `lockdep_assert_held()` صريح) |
| `CONFIG_PROVE_LOCKING` | يفعّل full lock dependency checking |
| `CONFIG_DEBUG_MUTEXES` | debug الـ mutex في `gpio_aggregator.lock` و `gpiochip_fwd.mlock` |
| `CONFIG_DEBUG_SPINLOCK` | debug الـ spinlock في `gpiochip_fwd.slock` |
| `CONFIG_KASAN` | كشف memory corruption في الـ `kzalloc/krealloc` calls |
| `CONFIG_CONFIGFS_FS` | لازم يكون مفعل للـ configfs interface |
| `CONFIG_OF_GPIO` | لـ Device Tree support و `gpio-delay` compatible |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'GPIO_AGGREGATOR|DEBUG_GPIO|CONFIGFS|LOCKDEP'
```

---

#### 6. أدوات الـ Subsystem

**الـ gpiotools (libgpiod):**

```bash
# شوف كل الـ GPIO chips بما فيها الـ virtual aggregator
gpiodetect

# اعرض كل lines في الـ aggregator chip
gpioinfo gpiochip5

# اقرأ قيمة line معينة
gpioget --chip gpiochip5 0 1 2

# اكتب قيمة
gpioset --chip gpiochip5 0=1 1=0

# راقب edge events
gpiomon --chip gpiochip5 0
```

**التحقق من platform device:**

```bash
# شوف الـ platform data والـ resources
ls -la /sys/bus/platform/devices/gpio-aggregator.0/
cat /sys/bus/platform/devices/gpio-aggregator.0/driver_override

# شوف الـ uevent
cat /sys/bus/platform/devices/gpio-aggregator.0/uevent
```

**التحقق من الـ IDR (Aggregator IDs):**

```bash
# كل aggregator بياخد ID من IDR، شوفهم عبر sysfs
ls /sys/bus/platform/devices/ | grep -E 'gpio-aggregator\.[0-9]+'
```

---

#### 7. جدول رسائل الـ Errors الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `gpio-aggregator: Cannot get GPIO count of 'gpiochipX'` | الـ chip مش موجود أو مش loaded | تأكد إن الـ GPIO chip driver محمّل قبل الـ aggregator |
| `gpio-aggregator: failed to find gpio chip` | الـ key في الـ lookup table مش بيطابق أي chip label | تحقق من `gpiodetect` وتأكد من صحة الـ chip label |
| `gpio-aggregator: Failed to register the platform driver: -N` | خطأ في تسجيل الـ platform driver | تحقق من `dmesg` للسبب التفصيلي، عادة resource conflict |
| `gpio-aggregator: Failed to register the 'gpio-aggregator' configfs subsystem: -N` | configfs مش مفعل أو mount issue | تأكد من `CONFIG_CONFIGFS_FS=y` وإن `/sys/kernel/config` مـ mounted |
| `gpio-aggregator: Deferred probe canceled for creation via configfs.` | GPIO dependency مش موجودة وقت الـ probe | أعد إنشاء الـ aggregator بعد ما تتأكد إن الـ GPIO chip موجود |
| `gpio-aggregator: gpiochipN: registered N GPIOs` (debug) | تسجيل ناجح للـ virtual chip | طبيعي — بيظهر مع `debug=1` |
| `-ENODEV` من `gpio_fwd_request` | محاولة request لـ offset خارج الـ `valid_mask` | تحقق إن كل الـ lines أُضيفت بـ `gpiochip_fwd_desc_add` قبل الـ register |
| `-EPROBE_DEFER` | GPIO dependency لسه مش جاهزة | طبيعي — الـ kernel هيعيد الـ probe تلقائياً |
| `-EINVAL` من `gpiochip_fwd_desc_add` | offset >= ngpio | تأكد من عدد الـ GPIOs في `devm_gpiochip_fwd_alloc` |
| `-EEXIST` من `gpiochip_fwd_desc_add` | نفس الـ offset أُضيف مرتين | الـ valid_mask bit مضبوط مسبقاً — مش مفروض يحصل في normal flow |

---

#### 8. أماكن استراتيجية لـ dump_stack() و WARN_ON()

الأماكن الأكتر أهمية في الـ code:

```c
/* 1. في gpio_aggregator_probe() بعد gpiod_count() */
n = gpiod_count(dev, NULL);
WARN_ON(n > AGGREGATOR_MAX_GPIOS); /* الـ 512 limit check */

/* 2. في gpiochip_fwd_desc_add() عند إضافة desc */
int gpiochip_fwd_desc_add(struct gpiochip_fwd *fwd, struct gpio_desc *desc,
                          unsigned int offset)
{
    WARN_ON(!fwd->descs);       /* null descs array */
    WARN_ON(IS_ERR(desc));      /* invalid descriptor */
    /* ... */
}

/* 3. في gpio_fwd_get_multiple() لو values corrupt */
if (WARN_ON(j > fwd->chip.ngpio))
    return -EINVAL;

/* 4. عند الـ lock violations — موجودة فعلاً في الكود */
lockdep_assert_held(&aggr->lock); /* في gpio_aggregator_is_active() وغيرها */

/* 5. في gpio_aggregator_alloc() لو IDR overflow */
WARN_ON(ret >= INT_MAX);
```

**تفعيل الـ KASAN للكشف عن الـ krealloc issues في `gpio_aggregator_add_gpio()`:**

```bash
# في الـ kernel config
CONFIG_KASAN=y
CONFIG_KASAN_GENERIC=y
# ثم شوف dmesg بعد إنشاء aggregator
dmesg | grep -E 'BUG|KASAN|gpio-aggregator'
```

---

### Hardware Level

#### 1. التحقق من توافق الـ Hardware State مع الـ Kernel State

الـ gpio-aggregator بيعمل **virtual forwarding** — كل write/read بيمر للـ physical GPIO driver الأصلي. التحقق بيتم على مستويين:

```bash
# المستوى الأول: تحقق من الـ virtual chip (aggregator)
gpioinfo gpiochip5   # الـ virtual forwarder chip

# المستوى الثاني: تحقق من الـ physical chips المصدر
gpiodetect           # شوف كل الـ chips
gpioinfo gpiochip0   # الـ physical chip اللي بيوفر الـ lines

# قارن القيم: لازم تكون متطابقة (مع مراعاة ACTIVE_LOW)
gpioget gpiochip5 0   # قيمة الـ virtual line
gpioget gpiochip0 5   # قيمة الـ physical line المقابلة

# لو الـ aggregator معرّف في configfs، تحقق من الـ offset mapping
cat /sys/kernel/config/gpio-aggregator/my-aggr/line0/offset  # الـ hwnum
cat /sys/kernel/config/gpio-aggregator/my-aggr/line0/key     # chip label
```

---

#### 2. تقنيات الـ Register Dump

الـ GPIO aggregator نفسه مش بيتعامل مباشرة مع hardware registers — بيمرر للـ physical driver. لكن لو عايز تتحقق من الـ physical registers:

```bash
# باستخدام devmem2 (لازم تعرف العنوان من datasheet)
# مثال: RPI GPIO base address
devmem2 0xFE200000 w    # قرأ GPIO Function Select Register 0

# باستخدام /dev/mem مباشرة (خطر - استخدم بحذر)
dd if=/dev/mem bs=4 count=1 skip=$((0xFE200000/4)) 2>/dev/null | xxd

# لو الـ driver بيكشف registers عبر debugfs
ls /sys/kernel/debug/gpio/
```

**تحقق من state الـ GPIO via sysfs الـ physical chip:**

```bash
# اعرف الـ base number للـ physical chip
cat /sys/class/gpio/gpiochipX/base
cat /sys/class/gpio/gpiochipX/ngpio
cat /sys/class/gpio/gpiochipX/label

# export الـ physical GPIO مباشرة وقارن
echo $((BASE + HWNUM)) > /sys/class/gpio/export
cat /sys/class/gpio/gpio$((BASE + HWNUM))/value
cat /sys/class/gpio/gpio$((BASE + HWNUM))/direction
```

---

#### 3. نصائح الـ Logic Analyzer والـ Oscilloscope

الـ gpio-aggregator مثير للاهتمام بسبب feature الـ **GPIO delay** (`FWD_FEATURE_DELAY` / `gpio-delay` compatible):

```
Logic Analyzer Setup:
┌─────────────────────────────────────────┐
│  Channel 0 → Physical GPIO output       │
│  Channel 1 → Trigger signal (SW write)  │
│                                         │
│  Measurement: ramp_up_us delay          │
│  Expected: matches DT args[1] value     │
└─────────────────────────────────────────┘
```

```bash
# الـ DT للـ gpio-delay يحدد timing:
# gpio-delay node:
#   compatible = "gpio-delay";
#   gpios = <&gpiochip0 5 0   500 200>,   # line 0: ramp_up=500us ramp_down=200us
#            <&gpiochip0 6 0  1000 500>;   # line 1: ramp_up=1000us ramp_down=500us

# قس الـ delay على الـ oscilloscope:
# - ابعت set command لـ gpioset
# - قس الوقت من trigger حتى يتغير الـ signal فعلاً
# - قارن بـ ramp_up_us في الـ DT
```

**نقاط الـ probe على الـ oscilloscope:**

| النقطة | ما تقيسه |
|---|---|
| Vcc → GPIO pin | voltage level (3.3V أو 5V حسب الـ chip) |
| GPIO output → GND | rise/fall time، مقارنة بالـ ramp timing |
| SCL/SDA (لو I2C GPIO expander) | protocol correctness |
| IRQ line | edge polarity مقارنة بـ `GPIO_ACTIVE_LOW` flag |

---

#### 4. مشاكل الـ Hardware الشائعة وأنماط الـ Kernel Log

| المشكلة الـ Hardware | نمط الـ Kernel Log | التشخيص |
|---|---|---|
| الـ GPIO chip مش موجود وقت الـ probe | `deferred probe canceled for creation via configfs` | شوف `gpiodetect` — الـ chip label مش موجود |
| خطأ في الـ ACTIVE_LOW config | القيمة المقروءة عكسية من المتوقع | تحقق من `flags` في configfs line أو DT `GPIO_ACTIVE_LOW` |
| الـ GPIO line مش configured كـ output | `gpiod_set_value: gpio write failed -22` | الـ direction لسه input — استخدم `direction_output` أولاً |
| تضارب ownership (line مطلوبة من driver تاني) | `gpiod_request: gpio-496 is already requested` | `gpioinfo` يظهر الـ consumer name |
| Open-drain مش configured صح | voltage stuck عند الـ pull-up level | تحقق من `GPIO_OPEN_DRAIN` في الـ lookup flags |
| Race في multi-GPIO set | data corruption في الـ `fwd_tmp_descs` buffer | تفعيل `CONFIG_DEBUG_SPINLOCK` — الـ slock/mlock المسؤول عن الحماية |
| الـ ramp delay طويل جداً في atomic context | kernel warning لـ sleeping in atomic | الـ aggregator يضبط `can_sleep=true` تلقائياً لو أي line تحتاج sleep |

---

#### 5. تشخيص الـ Device Tree

الـ gpio-aggregator بيدعم الـ DT عبر `compatible = "gpio-delay"` لـ `gpiochip_fwd_delay_of_xlate`:

```bash
# تحقق من الـ DT الفعلي على الـ device
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "gpio-delay"

# أو باستخدام fdtdump
cat /sys/firmware/fdt | fdtdump - 2>/dev/null | grep -A 10 gpio-delay

# تحقق من عدد args الـ OF (لازم يكون 3 لـ gpio-delay: line, ramp_up, ramp_down)
cat /sys/firmware/devicetree/base/gpio-delay@0/\#gpio-cells

# لو الـ DT مش محمّل صح، هتلاقي:
dmesg | grep "OF: /gpio-delay"
dmesg | grep "gpio.*invalid"
```

**مثال DT صحيح لـ gpio-delay:**

```dts
/* في الـ DTS */
gpio_delay_0: gpio-delay {
    compatible = "gpio-delay";
    #gpio-cells = <3>;   /* line, ramp_up_us, ramp_down_us */
    gpios = <&gpio0 10 GPIO_ACTIVE_HIGH 1000 500>,
            <&gpio0 11 GPIO_ACTIVE_HIGH  500 100>;
};

consumer_node {
    gpios = <&gpio_delay_0 0 0>;  /* استخدام line 0 من الـ aggregator */
};
```

```bash
# تحقق إن of_gpio_n_cells = 3 مضبوطة صح
dmesg | grep -i "gpio_n_cells\|of_xlate\|gpio-delay"
```

---

### Practical Commands

#### جاهز للـ Copy-Paste

**سيناريو 1: إنشاء aggregator وفحصه (configfs)**

```bash
#!/bin/bash
# Mount configfs لو مش موجود
mountpoint -q /sys/kernel/config || mount -t configfs none /sys/kernel/config

AGGR="my-test-aggr"

# إنشاء aggregator group
mkdir /sys/kernel/config/gpio-aggregator/${AGGR}

# إضافة line (افرض إن gpiochip0 line 5 متاحة)
mkdir /sys/kernel/config/gpio-aggregator/${AGGR}/line0
echo "gpiochip0"  > /sys/kernel/config/gpio-aggregator/${AGGR}/line0/key
echo "5"          > /sys/kernel/config/gpio-aggregator/${AGGR}/line0/offset

mkdir /sys/kernel/config/gpio-aggregator/${AGGR}/line1
echo "gpiochip0"  > /sys/kernel/config/gpio-aggregator/${AGGR}/line1/key
echo "6"          > /sys/kernel/config/gpio-aggregator/${AGGR}/line1/offset

# فعّل (probe) الـ aggregator
echo 1 > /sys/kernel/config/gpio-aggregator/${AGGR}/activate

# تحقق من النجاح
sleep 0.2
gpiodetect | grep aggregator
gpioinfo $(gpiodetect | grep aggregator | awk '{print $1}')
```

**سيناريو 2: Legacy sysfs interface**

```bash
# إنشاء aggregator عبر sysfs القديم
# الصيغة: "<chip_label> <offset>[,<offset>...] [name=<name>]"
echo "gpiochip0 5,6,7" > /sys/bus/platform/drivers/gpio-aggregator/new_device

# شوف الـ device المنشأ
dmesg | tail -5
ls /sys/bus/platform/devices/ | grep gpio-aggregator

# حذف الـ aggregator
# أولاً اعرف الـ ID من dmesg أو ls
echo "gpio-aggregator.0" > /sys/bus/platform/drivers/gpio-aggregator/delete_device
```

**سيناريو 3: تشخيص كامل**

```bash
#!/bin/bash
echo "=== GPIO Aggregator Debug Report ==="
echo ""

echo "--- Kernel Module Status ---"
lsmod | grep gpio_aggregator || echo "Module not loaded"
modinfo gpio_aggregator 2>/dev/null | grep -E "version|filename"

echo ""
echo "--- Platform Devices ---"
ls /sys/bus/platform/devices/ | grep gpio-aggregator

echo ""
echo "--- All GPIO Chips ---"
gpiodetect 2>/dev/null || cat /sys/kernel/debug/gpio 2>/dev/null | head -30

echo ""
echo "--- ConfigFS State ---"
if mountpoint -q /sys/kernel/config 2>/dev/null; then
    find /sys/kernel/config/gpio-aggregator/ -type f -exec sh -c \
        'echo "$1: $(cat $1 2>/dev/null)"' _ {} \; 2>/dev/null
else
    echo "configfs not mounted"
fi

echo ""
echo "--- Recent dmesg ---"
dmesg | grep -iE 'gpio-aggregator|gpio_aggregator|gpiochip_fwd' | tail -20

echo ""
echo "--- Dynamic Debug Rules ---"
grep gpio.aggregator /sys/kernel/debug/dynamic_debug/control 2>/dev/null || echo "dynamic_debug not available"
```

**سيناريو 4: تفعيل الـ tracing وتسجيل النتيجة**

```bash
#!/bin/bash
TRACE_DIR=/sys/kernel/debug/tracing

# صفّي الـ trace buffer
echo > ${TRACE_DIR}/trace

# فعّل GPIO events
echo 1 > ${TRACE_DIR}/events/gpio/enable

# فعّل function tracing للـ aggregator functions
echo 'gpio_fwd_*' > ${TRACE_DIR}/set_ftrace_filter
echo 'gpiochip_fwd_*' >> ${TRACE_DIR}/set_ftrace_filter
echo function > ${TRACE_DIR}/current_tracer

echo 1 > ${TRACE_DIR}/tracing_on

echo "--- Performing GPIO operations ---"
CHIP=$(gpiodetect | grep aggregator | head -1 | awk '{print $1}')
if [ -n "$CHIP" ]; then
    gpioget --chip ${CHIP} 0
    gpioset --chip ${CHIP} 0=1
    gpioset --chip ${CHIP} 0=0
fi

echo 0 > ${TRACE_DIR}/tracing_on

echo "--- Trace Output ---"
cat ${TRACE_DIR}/trace

# تنظيف
echo nop > ${TRACE_DIR}/current_tracer
echo > ${TRACE_DIR}/set_ftrace_filter
echo 0 > ${TRACE_DIR}/events/gpio/enable
```

**سيناريو 5: تشخيص مشكلة الـ delay (gpio-delay compatible)**

```bash
#!/bin/bash
# تحقق من إن الـ gpio-delay device موجود
ls /sys/bus/platform/devices/ | grep gpio-delay || echo "No gpio-delay device found"

# شوف الـ DT node
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -B2 -A15 "gpio-delay"

# تحقق من الـ gpio-cells (لازم 3 لـ delay support)
find /sys/firmware/devicetree/base -name "#gpio-cells" | while read f; do
    val=$(cat "$f" | xxd -p)
    if [ "$val" = "00000003" ]; then
        echo "3-cell gpio node found at: $(dirname $f)"
        cat $(dirname $f)/compatible 2>/dev/null
    fi
done

# قس الـ delay فعلياً (تحتاج logic analyzer أو oscilloscope)
# لكن ممكن تتحقق من timing في الـ kernel log
echo 'module gpio_aggregator +p' > /sys/kernel/debug/dynamic_debug/control
dmesg -c > /dev/null  # صفّي الـ buffer

# شغّل عملية set وراقب الـ timing
time gpioset --chip gpiochip5 0=1
time gpioset --chip gpiochip5 0=0

# المخرجات المتوقعة:
# real    0m0.001500s   ← إذا ramp_up_us=1000
# real    0m0.000700s   ← إذا ramp_down_us=500
```

**مثال على تفسير الـ output:**

```
# مخرج gpioinfo لـ aggregator chip:
gpiochip5 - 2 lines:
        line   0:      unnamed       unused   input  active-high
        line   1:      unnamed       unused   output active-high [used]

# التفسير:
# - "line 0": virtual GPIO index 0 في الـ forwarder
# - "unnamed": مفيش اسم مخصص للـ line (يمكن تحديده عبر configfs name attribute)
# - "unused": مفيش consumer حالياً
# - "input/output": الـ direction الحالي
# - "active-high": مفيش ACTIVE_LOW flag
# - "[used]": في consumer بيستخدمها حالياً

# مخرج cat /sys/kernel/debug/gpio:
# gpiochip5: GPIOs 496-497, parent: platform/gpio-aggregator.0, gpio-aggregator.0:
#  gpio-496 (                    |sysfs              ) in  lo ACTIVE HIGH
#  gpio-497 (my-signal           |my-driver          ) out hi ACTIVE LOW
#                                  ↑consumer name       ↑direction  ↑polarity
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Industrial Gateway على RK3562 — GPIO من chipات مختلفة في driver واحد

#### العنوان
دمج GPIOs من GPIO controllers مختلفة في industrial gateway لتشغيل relay board

#### السياق
بورد industrial gateway مبنية على **RK3562** بتستخدم relay board خارجية بتتحكم في 8 relays. المشكلة إن الـ 8 GPIOs مش كلها في نفس الـ GPIO controller — 4 منهم في `gpio1` و4 تانيين في `gpio3`. الـ userspace application محتاج يشوفهم كـ device واحد متسلسل من 0 لـ 7.

#### المشكلة
الـ application بيطلب `/dev/gpiochip` وبيلاقي الـ GPIOs موزعة على chipات مختلفة. كل `gpiochip` ليه numbering منفصل. محاولة التحكم في الـ relay بـ offset 0-7 على chip واحد بتفشل لأن بعض الـ lines مش موجودة فيه.

#### التحليل

في `gpio_aggregator_probe()`:
```c
n = gpiod_count(dev, NULL);   /* بيعد كل الـ GPIOs في lookup table */
for (i = 0; i < n; i++) {
    descs[i] = devm_gpiod_get_index(dev, NULL, i, GPIOD_ASIS);
    /* كل desc بياخد من الـ chip المناسب حسب الـ lookup table */
}
fwd = gpiochip_fwd_create(dev, n, descs, features);
```

الـ `gpiochip_fwd_create()` بيعمل `gpiochip_fwd` واحد بـ `ngpio = 8`، وبيخزن الـ descs في `fwd->descs[]`. الـ `gpiochip_fwd_desc_add()` بيشوف إذا أي desc من chip يحتاج sleep:

```c
if (gpiod_cansleep(desc))
    chip->can_sleep = true;
/* لو أي GPIO من I2C-based expander → كل الـ chip هيبقى can_sleep */
```

الـ forwarding في `gpio_fwd_set()`:
```c
static int gpio_fwd_set(struct gpio_chip *chip, unsigned int offset, int value)
{
    struct gpiochip_fwd *fwd = gpiochip_get_data(chip);
    /* offset 0-3 → descs[0..3] → gpio1 */
    /* offset 4-7 → descs[4..7] → gpio3 */
    ret = gpiod_set_value(fwd->descs[offset], value);
}
```

#### الحل

**خطوة 1 — إنشاء الـ aggregator عبر configfs:**
```bash
# mount configfs لو مش متعمل
mount -t configfs none /sys/kernel/config

cd /sys/kernel/config/gpio-aggregator
mkdir relay-board

# إضافة الـ lines
mkdir relay-board/line0
echo "gpio1" > relay-board/line0/key
echo "4"     > relay-board/line0/offset

mkdir relay-board/line1
echo "gpio1" > relay-board/line1/key
echo "5"     > relay-board/line1/offset

# ... line2, line3 من gpio1

mkdir relay-board/line4
echo "gpio3" > relay-board/line4/key
echo "2"     > relay-board/line4/offset

# ... line5, line6, line7 من gpio3

# تفعيل الـ aggregator
echo 1 > relay-board/live
```

**خطوة 2 — التحقق:**
```bash
cat relay-board/dev_name
# gpio-aggregator.0
gpioinfo gpio-aggregator.0
# gpiochip بـ 8 lines متسلسلة
```

#### الدرس المستفاد
الـ `gpiochip_fwd` بيعمل abstraction layer كاملة. الـ `fwd->descs[]` array بيخزن pointers للـ `gpio_desc` الحقيقية بغض النظر عن مصدرها. الـ `gpio_aggregator_add_gpio()` بيبني الـ `gpiod_lookup_table` اللي بتربط كل index بـ key (chip label) و offset.

---

### السيناريو الثاني: Android TV Box على Allwinner H616 — GPIO delay لـ HDMI power sequencing

#### العنوان
استخدام `gpio-delay` compatible لإدارة تسلسل power-up لـ HDMI على Allwinner H616

#### السياق
Android TV box مبنية على **Allwinner H616**. الـ HDMI panel محتاج power sequencing دقيق: الـ VCC بيجي الأول، وبعد 5ms بيجي الـ enable، وعند الإطفاء بيتأخر الـ enable لـ 3ms قبل ما يقطع VCC. لو الـ timing غلط الشاشة بتعمل flicker أو مش بتشتغل خالص.

#### المشكلة
الـ HDMI driver بيطلب GPIO واحد للـ enable، لكن في الواقع في 2 GPIOs بتتحكم في 2 power rails، وكل واحد محتاج delay مختلف. مفيش driver جاهز بيعمل ده.

#### التحليل

الـ driver بيدعم `compatible = "gpio-delay"` بـ `FWD_FEATURE_DELAY`:

```c
static const struct of_device_id gpio_aggregator_dt_ids[] = {
    {
        .compatible = "gpio-delay",
        .data = (void *)FWD_FEATURE_DELAY,  /* BIT(0) */
    },
    {}
};
```

في `gpio_aggregator_probe()`:
```c
features = (uintptr_t)device_get_match_data(dev);
/* features = FWD_FEATURE_DELAY */
fwd = gpiochip_fwd_create(dev, n, descs, features);
```

في `gpiochip_fwd_create()`:
```c
if (features & FWD_FEATURE_DELAY) {
    error = gpiochip_fwd_setup_delay_line(fwd);
    /* بيعمل fwd->delay_timings array */
    /* بيضبط chip->of_xlate = gpiochip_fwd_delay_of_xlate */
    /* chip->of_gpio_n_cells = 3  ← بدل 2 العادي */
}
```

الـ `of_xlate` الجديد بياخد 3 args: `<line ramp_up_us ramp_down_us>`:
```c
static int gpiochip_fwd_delay_of_xlate(..., const struct of_phandle_args *gpiospec, ...)
{
    line = gpiospec->args[0];
    timings->ramp_up_us   = gpiospec->args[1];  /* delay عند set=1 */
    timings->ramp_down_us = gpiospec->args[2];  /* delay عند set=0 */
    return line;
}
```

وعند الـ set:
```c
static void gpio_fwd_delay(struct gpio_chip *chip, unsigned int offset, int value)
{
    bool is_active_low = gpiod_is_active_low(fwd->descs[offset]);
    if ((!is_active_low && value) || (is_active_low && !value))
        delay_us = delay_timings->ramp_up_us;
    else
        delay_us = delay_timings->ramp_down_us;
    if (chip->can_sleep)
        fsleep(delay_us);
    else
        udelay(delay_us);
}
```

#### الحل

**Device Tree node:**
```dts
hdmi_pwr_seq: gpio-delay {
    compatible = "gpio-delay";
    /* <&chip line ramp_up_us ramp_down_us> */
    gpios = <&pio 2 5 GPIO_ACTIVE_HIGH 5000 0>,   /* VCC: 5ms up */
            <&pio 2 6 GPIO_ACTIVE_HIGH 0    3000>; /* EN:  3ms down */
    gpio-line-names = "hdmi-vcc", "hdmi-en";
};

&hdmi {
    hdmi-supply = <&hdmi_pwr_seq 0>;  /* يستخدم virtual line 0 */
    enable-gpios = <&hdmi_pwr_seq 1>; /* يستخدم virtual line 1 */
};
```

**التحقق:**
```bash
# بعد boot
gpioinfo | grep hdmi
# يظهر gpiochip خاص بالـ aggregator بـ 2 lines
dmesg | grep gpio-aggregator
# gpio-aggregator gpio-aggregator.0: 0 => gpio 69 irq -1
# gpio-aggregator gpio-aggregator.0: 1 => gpio 70 irq -1
```

#### الدرس المستفاد
الـ `FWD_FEATURE_DELAY` بيغير الـ `of_gpio_n_cells` لـ 3، يعني الـ DT لازم يضيف قيمتين للـ timing في كل GPIO reference. الـ `gpio_fwd_delay()` بيحترم الـ `active_low` flag عند تحديد اتجاه الـ delay، ده مهم جداً لـ open-drain outputs.

---

### السيناريو الثالث: IoT Sensor Board على STM32MP1 — تعارض في probe بسبب GPIO غير موجود وقت التسجيل

#### العنوان
`-EPROBE_DEFER` مع configfs aggregator على STM32MP1 بيسبب device مش بيشتغل أبداً

#### السياق
Custom IoT board على **STM32MP1** فيها GPIO expander خارجي على I2C (مثلاً PCA9555). الـ aggregator بيجمع GPIOs من الـ internal GPIO controller ومن الـ PCA9555. المشكلة إن الـ PCA9555 driver بياخد وقت في الـ probe بسبب ترتيب الـ init.

#### المشكلة
عند activation الـ aggregator عبر configfs بـ `echo 1 > live`، الـ `gpio_aggregator_probe()` بيرجع `-EPROBE_DEFER` لأن الـ PCA9555 لسه ما probe-ش. المتوقع إن الـ device هيتعمل probe تاني أوتوماتيك، لكن ده مش بيحصل.

#### التحليل

في `gpio_aggregator_probe()` فيه logic خاص بـ configfs:

```c
for (i = 0; i < n; i++) {
    descs[i] = devm_gpiod_get_index(dev, NULL, i, GPIOD_ASIS);
    if (IS_ERR(descs[i])) {
        /*
         * لو الـ aggregator اتعمل via configfs (مش sysfs ومش DT)
         * → بيلغي الـ deferred probe ويرجع -ENODEV
         */
        if (!init_via_sysfs && !dev_of_node(dev) &&
            descs[i] == ERR_PTR(-EPROBE_DEFER)) {
            pr_warn("Deferred probe canceled for creation via configfs.\n");
            return -ENODEV;
        }
        return PTR_ERR(descs[i]);
    }
}
```

الـ device اتعمل عبر configfs (مش DT ومش sysfs legacy)، فالـ `init_via_sysfs = false` و `dev_of_node(dev) = NULL`. لما `devm_gpiod_get_index()` يرجع `-EPROBE_DEFER`، الـ driver يحوله لـ `-ENODEV` ويطبع warning. الـ device مش هيعمل retry أوتوماتيك.

#### الحل

**الخيار 1 — انتظار PCA9555 يجهز قبل activation:**
```bash
# تحقق إن الـ I2C expander probe-ش
ls /sys/bus/i2c/devices/0-0020/gpio/
# لو الـ directory موجود → جاهز

# بعد ما الـ expander يجهز، فعّل الـ aggregator
echo 1 > /sys/kernel/config/gpio-aggregator/my-aggr/live
```

**الخيار 2 — اكتب udev rule أو init script:**
```bash
# /etc/udev/rules.d/99-gpio-aggr.rules
ACTION=="add", SUBSYSTEM=="gpio", KERNEL=="gpiochip*", \
    ATTRS{label}=="pca9555", \
    RUN+="/usr/local/bin/activate-gpio-aggr.sh"
```

```bash
#!/bin/bash
# activate-gpio-aggr.sh
sleep 0.1  # تأكد من إكمال الـ probe
echo 1 > /sys/kernel/config/gpio-aggregator/my-aggr/live
```

**الخيار 3 — إنشاء الـ aggregator عبر DT بدل configfs:**
```dts
/* لو استخدمنا DT node → dev_of_node(dev) != NULL */
/* وبالتالي -EPROBE_DEFER هيتعامل معاه عادي */
my_aggregator: gpio-aggregator {
    compatible = "gpio-aggregator";
    gpios = <&gpio0 5 GPIO_ACTIVE_HIGH>,
            <&pca9555 3 GPIO_ACTIVE_HIGH>;
    gpio-line-names = "sensor-power", "sensor-reset";
};
```

**التحقق:**
```bash
dmesg | grep "Deferred probe"
# gpio-aggregator: Deferred probe canceled for creation via configfs.
# ده معناه إن configfs-created device اتأثر بالمشكلة
```

#### الدرس المستفاد
الـ comment في الكود واضح: configfs-created devices مش بتدعم deferred probe أوتوماتيك — المهندس المسؤول عن activate الـ device هو المسؤول عن الـ timing. لو محتاج deferred probe، استخدم DT node أو legacy sysfs اللي بيتعامل مع `-EPROBE_DEFER` عادي.

---

### السيناريو الرابع: Automotive ECU على i.MX8 — حذف line أثناء الـ device نشط

#### العنوان
محاولة `rmdir` لـ configfs line أثناء الـ aggregator نشط على i.MX8 في بيئة automotive

#### السياق
ECU (Electronic Control Unit) على **i.MX8** في سيارة. الـ aggregator بيجمع GPIOs للتحكم في actuators. في بيئة production، في script بيحاول تعديل الـ GPIO mapping أثناء التشغيل عن طريق حذف line من configfs وإعادة إنشائها بـ offset مختلف.

#### المشكلة
الـ script بيعمل `rmdir line2` وهو الـ device لسه active. المتوقع خطأ واضح، لكن النتيجة غير متوقعة: بعض الأحيان بيرجع `-EBUSY`، وأحياناً بيحصل deadlock في الـ kernel.

#### التحليل

لما الـ device يتفعّل بـ `echo 1 > live`، الدالة `gpio_aggregator_lockup_configfs()` بتشتغل:

```c
static void gpio_aggregator_lockup_configfs(struct gpio_aggregator *aggr, bool lock)
{
    struct configfs_subsystem *subsys = aggr->group.cg_subsys;
    struct gpio_aggregator_line *line;

    list_for_each_entry(line, &aggr->list_head, entry) {
        if (lock)
            configfs_depend_item_unlocked(subsys, &line->group.cg_item);
        else
            configfs_undepend_item_unlocked(&line->group.cg_item);
    }
}
```

الـ `configfs_depend_item_unlocked()` بيمنع الـ `rmdir` على الـ line items طالما الـ device نشط. أي محاولة `rmdir` ترجع `-EBUSY`.

لكن المشكلة الحقيقية هي لو الـ script بيحاول:
1. `rmdir line2` — بيفشل بـ `-EBUSY` ✓
2. بيحاول يكتب في `line2/key` وهو الـ device active

في `gpio_aggregator_line_key_store()`:
```c
guard(mutex)(&aggr->lock);

if (gpio_aggregator_is_activating(aggr) ||
    gpio_aggregator_is_active(aggr))
    return -EBUSY;  /* محمي صح */
```

الـ deadlock محتمل لو حصل استخدام خاطئ للـ `aggr->lock` مع الـ configfs subsystem mutex في ترتيب عكسي. الكود الحالي يتجنب ده بالـ `_unlocked` variants.

#### الحل

**الممارسة الصحيحة:**
```bash
# دايماً deactivate الأول
echo 0 > /sys/kernel/config/gpio-aggregator/ecu-gpios/live

# انتظر تأكيد
while [ "$(cat /sys/kernel/config/gpio-aggregator/ecu-gpios/live)" = "1" ]; do
    sleep 0.05
done

# دلوقتي ممكن تعدل
rmdir /sys/kernel/config/gpio-aggregator/ecu-gpios/line2
mkdir /sys/kernel/config/gpio-aggregator/ecu-gpios/line2
echo "gpio2" > /sys/kernel/config/gpio-aggregator/ecu-gpios/line2/key
echo "8"     > /sys/kernel/config/gpio-aggregator/ecu-gpios/line2/offset

# أعد التفعيل
echo 1 > /sys/kernel/config/gpio-aggregator/ecu-gpios/live
```

**لو عايز تتحقق من الـ state:**
```bash
cat /sys/kernel/config/gpio-aggregator/ecu-gpios/live
# 1 = active, 0 = inactive
```

#### الدرس المستفاد
الـ `configfs_depend_item_unlocked()` هو الآلية اللي بتحمي الـ configfs entries من التعديل أثناء الـ device نشط. ده مش bug — ده design. في بيئة automotive، الـ scripts اللازم تتحقق من الـ `live` attribute قبل أي تعديل على الـ lines.

---

### السيناريو الخامس: Custom Board Bring-up على AM62x — legacy sysfs interface بيتعارض مع configfs

#### العنوان
مهندس bring-up يستخدم legacy sysfs interface على AM62x ويحاول بعدين يعدّل via configfs فتحصل مشكلة

#### السياق
مهندس bringup عامل custom board على **AM62x** (Texas Instruments). في الأول بيستخدم الـ legacy sysfs interface السريع للـ prototyping، وبعد ما الـ board بتستقر بيحاول يعدّل الـ configuration عبر configfs لإنه أكثر flexibility.

#### المشكلة
المهندس عمل aggregator عبر sysfs:
```bash
echo "gpio-0-15 1,3,5 gpio-1-24 2,4" > \
    /sys/bus/platform/drivers/gpio-aggregator/new_device
```
طلع `gpio-aggregator.0`. دلوقتي بيحاول يعمل `mkdir` في `/sys/kernel/config/gpio-aggregator/`:
```bash
mkdir /sys/kernel/config/gpio-aggregator/_sysfs.0
# -EINVAL
```

وبيحاول يعمل `mkdir line5` تحت الـ `_sysfs.0` group اللي اتعمل أوتوماتيك:
```bash
mkdir /sys/kernel/config/gpio-aggregator/_sysfs.0/line5
# -EPERM
```

#### التحليل

**المشكلة الأولى — اسم `_sysfs` محجوز:**

في `gpio_aggregator_make_group()`:
```c
if (strncmp(name, AGGREGATOR_LEGACY_PREFIX,
            sizeof(AGGREGATOR_LEGACY_PREFIX) - 1) == 0)
    return ERR_PTR(-EINVAL);
/* AGGREGATOR_LEGACY_PREFIX = "_sysfs" */
```

أي اسم يبدأ بـ `_sysfs` ممنوع من اليوزر، لأنه reserved للـ auto-generated groups.

**المشكلة التانية — legacy groups ممنوع تعديلها:**

في `gpio_aggregator_device_make_group()`:
```c
if (aggr->init_via_sysfs)
    /*
     * Aggregators created via legacy sysfs interface are exposed as
     * default groups, which means rmdir(2) is prohibited for them.
     * For simplicity, and to avoid confusion, we also prohibit mkdir(2).
     */
    return ERR_PTR(-EPERM);
```

لما الـ aggregator اتعمل عبر sysfs، الـ `aggr->init_via_sysfs = true`، وده بيمنع أي `mkdir` لـ sub-groups جديدة.

**سبب التصميم:** الـ legacy sysfs interface بيبني الـ lines كلها في وقت واحد من الـ string المُمرَّرة، بينما الـ configfs interface بيبني كل line على حدة بـ `mkdir`. خلط الـ approaches بيكسر الـ consistency.

**Flow الـ sysfs creation:**
```c
static ssize_t gpio_aggregator_new_device_store(..., const char *buf, ...)
{
    /* 1. يعمل gpio_aggregator مع init_via_sysfs = true */
    aggr->init_via_sysfs = true;

    /* 2. يسجل group في configfs (للـ visibility فقط) */
    configfs_register_group(&gpio_aggregator_subsys.su_group, &aggr->group);

    /* 3. يعمل parse للـ args string */
    gpio_aggregator_parse(aggr);  /* بيبني الـ lines */

    /* 4. يسجل الـ platform device مباشرة */
    pdev = platform_device_register_data(NULL, DRV_NAME, aggr->id, &meta, sizeof(meta));
}
```

الـ device بياخد `pdev` مباشرة بدون `live` toggle، عكس الـ configfs workflow.

#### الحل

**الخيار الصح — احذف الـ legacy device وابدأ من configfs:**
```bash
# احذف الـ legacy device
echo "gpio-aggregator.0" > \
    /sys/bus/platform/drivers/gpio-aggregator/delete_device

# تأكد من الحذف
ls /sys/bus/platform/devices/ | grep gpio-aggregator
# ما يظهرش gpio-aggregator.0

# ابدأ من أول بـ configfs
cd /sys/kernel/config/gpio-aggregator
mkdir my-board-gpios

mkdir my-board-gpios/line0
echo "gpio-0-15" > my-board-gpios/line0/key
echo "1"         > my-board-gpios/line0/offset

mkdir my-board-gpios/line1
echo "gpio-0-15" > my-board-gpios/line1/key
echo "3"         > my-board-gpios/line1/offset

mkdir my-board-gpios/line2
echo "gpio-0-15" > my-board-gpios/line2/key
echo "5"         > my-board-gpios/line2/offset

mkdir my-board-gpios/line3
echo "gpio-1-24" > my-board-gpios/line3/key
echo "2"         > my-board-gpios/line3/offset

mkdir my-board-gpios/line4
echo "gpio-1-24" > my-board-gpios/line4/key
echo "4"         > my-board-gpios/line4/offset

# فعّل
echo 1 > my-board-gpios/live

# تحقق
cat my-board-gpios/dev_name
gpioinfo $(cat my-board-gpios/dev_name)
```

**لو محتاج تضيف line5 لاحقاً:**
```bash
# deactivate أولاً
echo 0 > my-board-gpios/live

mkdir my-board-gpios/line5
echo "gpio-1-24" > my-board-gpios/line5/key
echo "6"         > my-board-gpios/line5/offset

# reactivate
echo 1 > my-board-gpios/live
```

**ملاحظة على naming convention — الـ lines لازم sequential من 0:**

في `gpio_aggregator_activate()`:
```c
list_for_each_entry(line, &aggr->list_head, entry) {
    if (!line->key || line->idx != n) {
        ret = -EINVAL;  /* لو في gap في الترقيم */
        goto err_remove_swnode;
    }
    /* ... */
    n++;
}
```

يعني لو عندك `line0`, `line1`, `line3` بدون `line2` → الـ activation هيفشل بـ `-EINVAL`.

#### الدرس المستفاد
الـ legacy sysfs interface و configfs interface طريقتين **منفصلتين تماماً** ومش ممكن تتخلط. الـ `_sysfs` prefix محجوز، والـ `init_via_sysfs` flag بيمنع أي تعديل configfs على الـ legacy devices. لو بتعمل bring-up، الأفضل تبدأ بـ configfs مباشرة — أوضح وأكثر flexibility. الـ sysfs legacy موجود لـ backward compatibility بس.
## Phase 7: مصادر ومراجع

---

### مصادر رسمية — التوثيق الرسمي للـ Kernel

| المصدر | الرابط |
|--------|--------|
| **GPIO Aggregator** — الصفحة الرسمية في kernel docs | [docs.kernel.org/admin-guide/gpio/gpio-aggregator.html](https://docs.kernel.org/admin-guide/gpio/gpio-aggregator.html) |
| **GPIO Driver Interface** — واجهة كتابة الـ driver | [static.lwn.net/kerneldoc/driver-api/gpio/driver.html](https://static.lwn.net/kerneldoc/driver-api/gpio/driver.html) |
| **GPIO Mappings** — ربط الـ GPIOs بالـ platform devices | [static.lwn.net/kerneldoc/driver-api/gpio/board.html](https://static.lwn.net/kerneldoc/driver-api/gpio/board.html) |
| **Subsystem drivers using GPIO** | [static.lwn.net/kerneldoc/driver-api/gpio/drivers-on-gpio.html](https://static.lwn.net/kerneldoc/driver-api/gpio/drivers-on-gpio.html) |
| **General Purpose I/O (GPIO) index** | [kernel.org/doc/html/latest/driver-api/gpio/index.html](https://www.kernel.org/doc/html/latest/driver-api/gpio/index.html) |
| **Configfs GPIO Simulator** — مشابه للـ aggregator في الفكرة | [docs.kernel.org/admin-guide/gpio/gpio-sim.html](https://docs.kernel.org/admin-guide/gpio/gpio-sim.html) |
| **Virtual GPIO Consumer** | [kernel.org/doc/html/v6.11/admin-guide/gpio/gpio-virtuser.html](https://www.kernel.org/doc/html/v6.11/admin-guide/gpio/gpio-virtuser.html) |

الـ `Documentation/` paths الأساسية داخل الـ source tree:

```
Documentation/admin-guide/gpio/gpio-aggregator.rst
Documentation/driver-api/gpio/driver.rst
Documentation/driver-api/gpio/board.rst
Documentation/driver-api/gpio/consumer.rst
```

---

### مقالات LWN.net

الـ LWN.net نشر عدة مقالات تغطي إضافة الـ GPIO Aggregator للـ kernel:

| المقال | الرابط |
|--------|--------|
| **gpio: Add GPIO Aggregator** — أول patch series بيتناولها review | [lwn.net/Articles/820204/](https://lwn.net/Articles/820204/) |
| **gpio: Add GPIO Aggregator Driver** — النسخة الأولى من الـ driver | [lwn.net/Articles/798971/](https://lwn.net/Articles/798971/) |
| **gpio: aggregator: Introduce delay support for individual output pins** | [lwn.net/Articles/934274/](https://lwn.net/Articles/934274/) |
| **gpio: aggregator: Incorporate gpio-delay functionality** | [lwn.net/Articles/934739/](https://lwn.net/Articles/934739/) |

الـ GPIO aggregator اتضاف رسمياً في **Linux 5.8** (صيف 2020). الـ patch series مر بـ 7+ iterations من 2019 لحد ما اتقبل.

---

### Patchwork وـ Mailing List

| المصدر | الرابط |
|--------|--------|
| **[v5, 0/5] gpio: Add GPIO Aggregator** — patch على Patchwork | [patchwork.kernel.org](https://patchwork.kernel.org/project/qemu-devel/cover/20200218151812.7816-1-geert+renesas@glider.be/) |
| **[PATCH v7 0/6] gpio: Add GPIO Aggregator** — آخر نسخة قبل الـ merge | [mail-archive.com/linux-kernel](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg2166915.html) |
| **gpio: aggregator: refactor code for GPIO desc** | [mail-archive.com/linux-hardening](https://www.mail-archive.com/linux-hardening@vger.kernel.org/msg09362.html) |
| **gpio: aggregator: add possibility to attach data to forwarder** | [mail-archive.com/linux-hardening v8](https://www.mail-archive.com/linux-hardening@vger.kernel.org/msg09683.html) |
| **LKML: Geert Uytterhoeven: Re: dt-bindings gpio-aggregator** | [lkml.org/lkml/2026/2/20/605](https://lkml.org/lkml/2026/2/20/605) |
| **MAINTAINERS: add linux-gpio mailing list** — تاريخ إضافة الـ mailing list | [linux.kernel.narkive.com](https://linux.kernel.narkive.com/jlZHi26J/patch-maintainers-add-linux-gpio-mailing-list) |

الـ mailing list الرسمي للـ GPIO subsystem:

```
linux-gpio@vger.kernel.org
```

لأرشيف الـ patches والـ discussions:

```
https://lore.kernel.org/linux-gpio/
```

---

### Commits الأساسية في الـ Git History

الـ commits دي بتغطي إضافة وتطوير الـ driver:

```bash
# تاريخ الـ driver من الـ git log
git log --oneline drivers/gpio/gpio-aggregator.c

# الـ commit الأول اللي ضاف الـ driver (Linux 5.8)
# Author: Geert Uytterhoeven <geert+renesas@glider.be>
# Subject: gpio: Add GPIO Aggregator

# للبحث عن commits محددة:
git log --all --grep="gpio: aggregator" --oneline
git log --all --grep="gpio-aggregator" --oneline
```

الـ kernel.org Git browser:

```
https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/drivers/gpio/gpio-aggregator.c
```

---

### مقالات خارجية متخصصة

| المصدر | الرابط |
|--------|--------|
| **GPIO Aggregator, a virtual gpio chip** — Bootlin blog | [bootlin.com/blog/gpio-aggregator-a-virtual-gpio-chip/](https://bootlin.com/blog/gpio-aggregator-a-virtual-gpio-chip/) |
| **Introduction to pin muxing and GPIO control under Linux** — ELC 2021 slides | [elinux.org/images/a/a7/ELC-2021_Introduction_to_pin_muxing_and_GPIO_control_under_Linux.pdf](https://elinux.org/images/a/a7/ELC-2021_Introduction_to_pin_muxing_and_GPIO_control_under_Linux.pdf) |
| **New Linux kernel GPIO userspace interface** — sergioprado.blog | [sergioprado.blog/new-linux-kernel-gpio-user-space-interface/](https://sergioprado.blog/new-linux-kernel-gpio-user-space-interface/) |
| **GPIO Programming: Using the sysfs Interface** — ICS | [ics.com/blog/gpio-programming-using-sysfs-interface](https://www.ics.com/blog/gpio-programming-using-sysfs-interface) |
| **GPIO Aggregator (Linux 5.10 docs mirror)** | [infradead.org/~mchehab](https://www.infradead.org/~mchehab/kernel_docs/admin-guide/gpio/gpio-aggregator.html) |

---

### KernelNewbies — صفحات الـ Releases

الـ kernel releases اللي فيها تغييرات مهمة على الـ GPIO subsystem:

| Release | رابط |
|---------|------|
| **Linux 5.8** — اتضاف فيه gpio-aggregator | [kernelnewbies.org/Linux_5.8](https://kernelnewbies.org/Linux_5.8) |
| **Linux 6.2** — GPIO updates | [kernelnewbies.org/Linux_6.2](https://kernelnewbies.org/Linux_6.2) |
| **Linux 6.6** — GPIO enhancements | [kernelnewbies.org/Linux_6.6](https://kernelnewbies.org/Linux_6.6) |
| **Linux 6.8** — further GPIO work | [kernelnewbies.org/Linux_6.8](https://kernelnewbies.org/Linux_6.8) |
| **Linux 6.11** — GPIO updates | [kernelnewbies.org/Linux_6.11](https://kernelnewbies.org/Linux_6.11) |
| **Linux 6.13** — Aspeed G7 GPIO support | [kernelnewbies.org/Linux_6.13](https://kernelnewbies.org/Linux_6.13) |
| **LinuxChanges** — كل التغييرات التاريخية | [kernelnewbies.org/LinuxChanges](https://kernelnewbies.org/LinuxChanges) |

---

### كتب مرجعية

#### Linux Device Drivers, 3rd Edition (LDD3)

الكتاب المرجعي الأساسي لكتابة الـ drivers — متاح مجاناً:

```
https://lwn.net/Kernel/LDD3/
```

الفصول ذات الصلة:

| الفصل | الموضوع |
|-------|---------|
| Chapter 1 | An Introduction to Device Drivers |
| Chapter 14 | The Linux Device Model — الـ kobject, sysfs, platform_device |
| Chapter 15 | Memory Mapping and DMA |

الـ GPIO ما اتغطاش بشكل كبير في LDD3 لأنه قديم نسبياً، لكن الـ platform_device model والـ sysfs اللي بيستخدمهم الـ aggregator متشرحين تفصيلياً.

---

#### Linux Kernel Development, 3rd Edition — Robert Love

```
ISBN: 978-0-672-32946-3
```

| الفصل | الموضوع |
|-------|---------|
| Chapter 1 | Introduction to the Linux Kernel |
| Chapter 17 | Devices and Modules — device model overview |
| Chapter 13 | The Virtual Filesystem — relevant for sysfs internals |

---

#### Embedded Linux Primer, 2nd Edition — Christopher Hallinan

```
ISBN: 978-0-13-701670-4
```

| الفصل | الموضوع |
|-------|---------|
| Chapter 15 | Porting Linux to a Custom Board — GPIO setup |
| Chapter 16 | Kernel Debugging Techniques |

بيغطي استخدام الـ GPIO في بيئات الـ embedded بشكل عملي، وبيشرح device tree و platform devices اللي بيعتمد عليهم الـ aggregator.

---

#### كتب إضافية

| الكتاب | الموضوع |
|--------|---------|
| **The Linux Programming Interface** — Michael Kerrisk | sysfs, procfs, userspace interaction |
| **Professional Linux Kernel Architecture** — Wolfgang Mauerer | في-عمق لكل subsystems |

---

### Search Terms للبحث عن معلومات أكتر

للبحث في الـ mailing list archives على `lore.kernel.org`:

```
gpio aggregator site:lore.kernel.org
gpio-aggregator forwarder gpiochip_fwd
gpio aggregator configfs platform_device
gpio aggregator "Geert Uytterhoeven"
gpio aggregator DT binding compatible
```

للبحث في الـ kernel source:

```bash
# داخل الـ kernel tree
grep -r "gpio-aggregator" Documentation/
grep -r "gpiochip_fwd" drivers/gpio/
grep -r "gpio_aggregator" drivers/gpio/

# في الـ git history
git log --all --oneline --grep="aggregator" -- drivers/gpio/
```

للبحث العام:

```
linux kernel gpio aggregator virtual gpiochip
gpio aggregator access control FPGA
gpio aggregator sysfs configfs interface
gpio forwarder gpiochip_fwd kernel
gpio-aggregator device tree binding
```
## Phase 8: Writing simple module

### الفكرة

الـ `gpio-aggregator` بيـexport مجموعة functions جوه namespace اسمه `GPIO_FORWARDER`، أكثرهم إثارة للمراقبة هو `gpiochip_fwd_gpio_get` — اللي بيُرجع قيمة GPIO line معينة من forwarder. كل ما حد قرأ state الـ GPIO عن طريق الـ aggregator، الـ kprobe هيشتغل ويطبع رقم الـ offset والنتيجة.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * kprobe_fwd_get.c
 *
 * Hooks gpiochip_fwd_gpio_get() from gpio-aggregator
 * and logs every call: which forwarder, which offset, return value.
 */

/* ---- includes ---- */
#include <linux/kernel.h>   /* pr_info, pr_err              */
#include <linux/module.h>   /* MODULE_* macros              */
#include <linux/kprobes.h>  /* kprobe, register_kprobe ...  */
#include <linux/gpio/forwarder.h> /* struct gpiochip_fwd   */
#include <linux/gpio/driver.h>   /* struct gpio_chip, gpiochip_get_data */

/*
 * بنضم الـ headers دي عشان نعرف layout الـ struct gpiochip_fwd
 * وبالتالي نقدر نجيب chip->label من جوا الـ fwd pointer.
 */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Learner");
MODULE_DESCRIPTION("kprobe on gpiochip_fwd_gpio_get to trace GPIO forwarder reads");

/* ---- forward declaration of internal struct we need ---- */
/*
 * gpiochip_fwd مش exported كـ public header تماماً،
 * بس struct gpio_chip موجود في gpio/driver.h وهو
 * أول field في gpiochip_fwd — كفاية نعمل container_of.
 * بنعرف الـ struct هنا locally بالحد الأدنى اللي محتاجينه.
 */
struct gpiochip_fwd_minimal {
    struct gpio_chip chip;  /* first field — matches real layout */
    /* rest of fields don't matter for our probe */
};

/* ---- kprobe struct ---- */
static struct kprobe kp_fwd_get;

/*
 * الـ pre-handler بيتشغل قبل ما الـ function الأصلية تتنفذ.
 * pt_regs بيحمل قيم الـ registers لحظة الـ probe hit.
 *
 * calling convention على x86-64:
 *   arg0 (fwd)    → rdi
 *   arg1 (offset) → rsi
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Extract arguments from registers (x86-64 SysV ABI) */
    struct gpiochip_fwd_minimal *fwd =
            (struct gpiochip_fwd_minimal *)regs->di;
    unsigned int offset = (unsigned int)regs->si;

    /*
     * نطبع اسم الـ chip (=label) والـ offset المطلوب.
     * chip->label بيجي من dev_name() وقت الـ allocation
     * في devm_gpiochip_fwd_alloc().
     */
    pr_info("fwd_get called: chip=[%s] offset=%u\n",
            fwd->chip.label ? fwd->chip.label : "(null)",
            offset);

    return 0; /* 0 = let the original function run normally */
}

/*
 * الـ post-handler بيتشغل بعد ما الـ function تخلص.
 * ax بيحمل الـ return value (قيمة الـ GPIO: 0 أو 1 أو error).
 */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    long retval = regs_return_value(regs);

    /*
     * بنطبع النتيجة هنا عشان نشوف القيمة الفعلية اللي رجعت
     * (0=low, 1=high, أو negative errno لو في error).
     */
    pr_info("fwd_get returned: value=%ld (%s)\n",
            retval,
            retval == 0 ? "LOW" :
            retval == 1 ? "HIGH" : "ERROR");
}

/* ---- init ---- */
static int __init kprobe_fwd_get_init(void)
{
    int ret;

    /*
     * بنحدد اسم الـ function اللي هنعمل لها probe.
     * الـ kprobe subsystem هيحول الاسم لـ address
     * باستخدام kallsyms تلقائياً.
     */
    kp_fwd_get.symbol_name = "gpiochip_fwd_gpio_get";
    kp_fwd_get.pre_handler  = handler_pre;
    kp_fwd_get.post_handler = handler_post;

    ret = register_kprobe(&kp_fwd_get);
    if (ret < 0) {
        pr_err("register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("kprobe planted on gpiochip_fwd_gpio_get @ %p\n",
            kp_fwd_get.addr);
    return 0;
}

/* ---- exit ---- */
static void __exit kprobe_fwd_get_exit(void)
{
    /*
     * لازم نعمل unregister قبل ما الـ module يتـunload
     * عشان منخليش الـ kernel يحاول ينفذ handler code
     * بعد ما اتمسحت من الـ memory — ده race condition كارثي.
     */
    unregister_kprobe(&kp_fwd_get);
    pr_info("kprobe removed from gpiochip_fwd_gpio_get\n");
}

module_init(kprobe_fwd_get_init);
module_exit(kprobe_fwd_get_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/kprobes.h` | يعرّف `struct kprobe`، `register_kprobe`، `unregister_kprobe` |
| `linux/gpio/forwarder.h` | يعرّف `struct gpiochip_fwd` المستخدمة كـ arg أول |
| `linux/gpio/driver.h` | يعرّف `struct gpio_chip` اللي جوا الـ forwarder وفيه `label` |

**الـ** `linux/kernel.h` بيجيب `pr_info` و`pr_err` اللي بنستخدمهم للـ logging.

---

#### الـ Target Function

**الـ** `gpiochip_fwd_gpio_get` هي wrapper exported بـ `EXPORT_SYMBOL_NS_GPL` جوا namespace `GPIO_FORWARDER`. بتاخد `(struct gpiochip_fwd *fwd, unsigned int offset)` وبترجع قيمة الـ GPIO line. ده اختيار مثالي لأن:
- بتتنادى كل ما حد قرأ GPIO عبر الـ aggregator
- arguments بسيطة وسهل نجيبهم من الـ registers
- مش بتشتغل في interrupt context فـ `pr_info` آمنة

---

#### الـ pre_handler

**الـ** `pt_regs->di` على x86-64 بيحمل أول argument للـ function (الـ `fwd` pointer)، و`pt_regs->si` بيحمل التاني (الـ `offset`). بنقرأ `chip.label` اللي اتضبط في `devm_gpiochip_fwd_alloc()` من اسم الـ device الأب.

---

#### الـ post_handler

**الـ** `regs_return_value(regs)` portable macro بيجيب قيمة `rax` على x86-64 بعد رجوع الـ function — ده هو الـ return value الفعلي (0=low, 1=high). الـ `flags` parameter مش بنستخدمه هنا بس محتاج يكون موجود في الـ signature.

---

#### الـ module_init / module_exit

**الـ** `register_kprobe` بيزرع breakpoint في الـ kernel code بشكل atomic بدون ما يحتاج يوقف الـ system. **الـ** `unregister_kprobe` في الـ exit ضروري عشان يشيل الـ breakpoint قبل ما الـ module code يتـunmap من الـ memory — من غيره الـ kernel هيـexecute code معدمة وهيحصل kernel panic.

---

### الـ Makefile وطريقة التجميع

```makefile
obj-m += kprobe_fwd_get.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

```bash
# Build
make

# Load
sudo insmod kprobe_fwd_get.ko

# Watch output (trigger by reading any GPIO via the aggregator)
sudo dmesg -w | grep fwd_get

# Unload
sudo rmmod kprobe_fwd_get
```

---

### مثال على الـ Output المتوقع

```
[  142.301] gpio-aggregator: fwd_get called: chip=[gpio-aggregator.0] offset=2
[  142.301] gpio-aggregator: fwd_get returned: value=1 (HIGH)
[  142.850] gpio-aggregator: fwd_get called: chip=[gpio-aggregator.0] offset=0
[  142.850] gpio-aggregator: fwd_get returned: value=0 (LOW)
```
