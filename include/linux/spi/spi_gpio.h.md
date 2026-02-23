## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

**الـ** `spi_gpio.h` جزء من **SPI Subsystem** في Linux kernel، والـ maintainer هو Mark Brown على مجموعة `linux-spi@vger.kernel.org`. الكود الأساسي موجود في `drivers/spi/`.

---

### ما هو الـ SPI أصلاً؟

تخيل إنك عندك Arduino وعايز تكلمه مع شاشة صغيرة أو sensor. محتاج وصلات:
- **SCLK** (clock) — نبضة الساعة، بتنظم متى يتبعت كل بت
- **MOSI** (Master Out Slave In) — البيانات الخارجة من الـ master
- **MISO** (Master In Slave Out) — البيانات الجاية للـ master
- **CS/SS** (Chip Select) — بتختار أنهي جهاز تكلمه (زي ما بتضغط على زرار توصيل)

الـ **SPI (Serial Peripheral Interface)** بروتوكول تواصل متزامن سريع بين الـ CPU وأجهزة زي الـ flash chips, sensors, displays.

---

### المشكلة اللي بتحلها `spi_gpio.h`

#### القصة:

تخيل إنك بتبني جهاز embedded جديد. اشتريت chip فيها SPI hardware controller تمام. بس:

1. **الـ driver بتاعه مش جاهز لسه** أو فيه bug
2. أو الـ pins الخاصة بالـ SPI hardware اتحجزت لحاجة تانية
3. أو إنت بس عايز تعمل **prototype سريع** وتشوف إن الـ device بتشتغل أصلاً

الحل؟ **Bitbanging** — بدل ما تستخدم الـ hardware SPI controller، بتعمل بروتوكول SPI يدوياً عن طريق **GPIO pins عادية**.

يعني بدل ما الـ hardware يبعت bits بسرعة، إنت بتقول للـ CPU:
> "يلا ارفع الـ clock pin، حط بيت على MOSI، نزل الـ clock، واقرأ MISO" — وتكرر ده 8 مرات لكل byte.

الـ CPU بيعمل كل الشغل بنفسه بـ **software**، بطيء أكتر بكتير من الـ hardware، لكن بيشتغل على **أي GPIO pins** في الجهاز.

---

### دور `spi_gpio.h` تحديداً

الملف ده **header صغير جداً** — بيعرّف struct واحد بس:

```c
/**
 * struct spi_gpio_platform_data - parameter for bitbanged SPI host controller
 * @num_chipselect: how many target devices to allow
 */
struct spi_gpio_platform_data {
    u16 num_chipselect;  /* كام device ممكن يتوصل بالـ bus ده */
};
```

الـ **platform_data** ده بيتبعت مع الـ `platform_device` عشان يقول للـ driver:
- كام chip select (CS) pins موجودة؟
- يعني كام device ممكن يتكلم على الـ bus ده؟

الـ GPIO pins نفسها (SCK, MOSI, MISO, CS) بتتعرّف دلوقتي من **Device Tree** أو من الـ GPIO descriptor API — مش من الـ struct ده مباشرة.

---

### الصورة الكاملة: إزاي الأجزاء بتتشغل مع بعض

```
Device Tree / Platform Data
        |
        v
  spi_gpio_platform_data       <-- spi_gpio.h (الملف ده)
  (num_chipselect)
        |
        v
  drivers/spi/spi-gpio.c       <-- الـ driver الحقيقي
  (struct spi_gpio: sck, miso, mosi, cs_gpios)
        |
        v
  include/linux/spi/spi_bitbang.h  <-- framework لتنظيم الـ bit-by-bit sending
  (struct spi_bitbang)
        |
        v
  drivers/spi/spi-bitbang.c    <-- logic الـ bitbang نفسه
        |
        v
  include/linux/spi/spi.h      <-- SPI core API
        |
        v
  drivers/spi/spi.c            <-- SPI subsystem core
```

---

### ليه ده مفيد؟

| السيناريو | الفايدة |
|-----------|---------|
| بروتوتايبينج سريع | تشغّل device بدون hardware controller جاهز |
| debugging | تتأكد إن الـ device شغال، المشكلة في الـ HW driver |
| embedded boards قديمة | مفيش hardware SPI أصلاً |
| testing | اختبار الـ SPI devices بشكل software-only |

---

### الفرق بين الوضع الـ "native" والـ "bitbang"

```
Native SPI Hardware:
CPU --[SPI Controller HW]--> SCLK/MOSI/MISO/CS pins  (سريع، DMA ممكن)

Bitbang via GPIO:
CPU --[spi-gpio.c software loop]--> أي GPIO pins  (بطيء، لكن مرن)
```

---

### الملفات المهمة اللي المبرمج لازم يعرفها

#### الـ Header الأساسي (الملف ده):
- **`include/linux/spi/spi_gpio.h`** — `struct spi_gpio_platform_data` (الملف اللي بندرسه)

#### الـ Driver:
- **`drivers/spi/spi-gpio.c`** — التنفيذ الكامل للـ bitbang SPI عبر GPIO، بيستخدم `gpiod_set_value_cansleep()` و `gpiod_get_value_cansleep()`

#### الـ Bitbang Framework:
- **`include/linux/spi/spi_bitbang.h`** — `struct spi_bitbang` اللي بيربط الـ GPIO operations بالـ SPI core
- **`drivers/spi/spi-bitbang.c`** — logic الإرسال bit-by-bit (`bitbang_txrx_8`, `bitbang_txrx_16`, إلخ)
- **`drivers/spi/spi-bitbang-txrx.h`** — inline functions للـ clock modes (CPOL/CPHA)

#### الـ SPI Core:
- **`include/linux/spi/spi.h`** — الـ API الأساسية: `struct spi_device`, `struct spi_controller`, `spi_transfer()`
- **`drivers/spi/spi.c`** — الـ SPI subsystem core
- **`drivers/spi/internals.h`** — internal structs للـ SPI core

#### Device Tree Bindings:
- **`Documentation/devicetree/bindings/spi/spi-gpio.yaml`** — إزاي تعرّف الـ SPI GPIO bus في الـ Device Tree

---

### ملخص

**الـ** `spi_gpio.h` هو header بسيط لكنه نقطة انطلاق مهمة — بيعرّف الـ platform_data اللي بتقول للـ driver "انت عندك كام CS pin؟". الفكرة الكبيرة هي إن SPI hardware مش ضرورة عشان تكلم SPI device — ممكن تعمله بـ software على أي GPIO pins.
## Phase 2: شرح الـ SPI / SPI-GPIO (Bitbanging) Framework

---

### المشكلة اللي بيحلها الـ Subsystem ده

**SPI (Serial Peripheral Interface)** هو بروتوكول تواصل synchronous بيستخدمه الـ CPU عشان يتكلم مع chips زي ADCs وFlash وDisplay controllers وغيرها. الـ protocol محتاج 4 signals كحد أدنى:

| Signal | الوصف |
|--------|-------|
| **SCLK** | Clock — بيوفره الـ master |
| **MOSI** | Master Out Slave In — data من الـ CPU للجهاز |
| **MISO** | Master In Slave Out — data من الجهاز للـ CPU |
| **CS/SS** | Chip Select — بيختار الجهاز المعين |

**المشكلة الأولى:** مش كل SoC أو microcontroller بيجي بـ dedicated SPI hardware controller. في boards كتير، خصوصاً في early development أو في custom embedded designs، مفيش SPI hardware خالص.

**المشكلة التانية:** حتى لو فيه SPI hardware، ممكن pins الـ SPI controller الأصلي تكون مشغولة بحاجة تانية، أو الـ board design اضطر يوصل الـ SPI device على GPIO pins عادية.

**المشكلة التالتة:** الـ developer بيحتاج يكتب كود يشتغل صح بغض النظر هو شغّال على hardware SPI أم على GPIO-bitbanged SPI — يعني الـ device driver (اللي بيتكلم مع الـ Flash chip مثلاً) المفروض ميعرفش ولا يهتمش إيه اللي تحته.

---

### الحل: Bitbanging عبر الـ spi_gpio Driver

الـ kernel حل المشكلة دي بطريقتين متكاملتين:

1. **SPI Framework (العامود الرئيسي):** طبقة abstraction عامة في `drivers/spi/spi.c` بتوفر API موحد لكل حاجة فوقها (الـ consumer drivers). مش مهم الـ bus اتنفذ إزاي.

2. **spi_gpio Driver:** driver بيطبّق الـ SPI protocol يدوياً على GPIO pins عادية — ده اللي بيسموه **bitbanging**. بيسمع على `platform_device` باسم `"spi_gpio"` ويتسجل كـ SPI controller عادي.

**الـ `struct spi_gpio_platform_data`** (الموجودة في الملف اللي بندرسه) هي الـ glue اللي بتربط الـ board description بالـ driver ده.

---

### تشبيه من الواقع — المطبعة اليدوية والـ Printer Driver

تخيل معايا:

- **الـ printer driver في الـ OS** = الـ SPI consumer driver (مثلاً driver الـ W25Q Flash).
- **الـ printer API** = الـ SPI Framework API (`spi_sync()`, `spi_transfer`, إلخ).
- **الـ USB printer port** = SPI hardware controller عادي (سريع، بيستخدم hardware FIFO).
- **الطباعة يدوياً ورقة ورقة** = spi_gpio bitbanging (بطيء، بيعمل كل bit يدوياً عبر GPIO).
- **الـ platform_data** = الورقة اللي فيها "الطابعة دي بتدعم كام paper tray" — تعريف للـ hardware setup.

الـ printer driver (consumer) بيبعت print job لأي طابعة بنفس الـ API. الـ OS بيوصّله للطابعة الصح. مش مهم هي USB أم يدوية.

**الربط الدقيق بالـ kernel:**

| التشبيه | الـ Kernel المقابل |
|---|---|
| الـ OS print API | `spi_sync()` / `spi_message` API |
| الـ USB printer driver | SoC native SPI driver (e.g. `spi-pl022.c`) |
| الطباعة اليدوية | `spi_gpio` bitbanging driver |
| رقم الـ paper tray | `num_chipselect` في `spi_gpio_platform_data` |
| اسم الطابعة | `"spi_gpio"` + bus number كـ `platform_device.id` |
| تسجيل الطابعة في الـ OS | `platform_device_register()` |
| الـ printer job قايمة | `spi_controller.queue` |

---

### Big Picture Architecture

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                    SPI Consumer Drivers                         │
  │  (spidev, w25q-flash, mcp3201-adc, lcd-ili9341, tpm-tis, ...)  │
  │                                                                 │
  │         يستخدموا: spi_sync(), spi_async(), spi_message         │
  └─────────────────────────┬───────────────────────────────────────┘
                            │  SPI Framework API
                            ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │              SPI Core  (drivers/spi/spi.c)                      │
  │                                                                 │
  │  - spi_register_controller()  - spi_bus_type                   │
  │  - message queue management   - device matching                 │
  │  - DMA mapping helpers         - sysfs/debugfs                 │
  └────────┬──────────────────────────────────────────┬────────────┘
           │                                          │
           ▼                                          ▼
  ┌─────────────────┐                      ┌──────────────────────┐
  │  Native SPI     │                      │   spi_gpio Driver    │
  │  Controller     │                      │  (drivers/spi/       │
  │  Driver         │                      │   spi-gpio.c)        │
  │  e.g. PL022,    │                      │                      │
  │  BCM2835, IMX   │                      │  Bitbangs SPI over   │
  │                 │                      │  regular GPIO pins   │
  │  Hardware FIFO  │                      │                      │
  │  DMA support    │                      │  platform_device:    │
  │  HW clock gen   │                      │  name="spi_gpio"     │
  └────────┬────────┘                      │  id=bus_number       │
           │                               │  .platform_data =    │
           ▼                               │  spi_gpio_platform_  │
  ┌─────────────────┐                      │  data { .num_cs }    │
  │  SPI Hardware   │                      └──────────┬───────────┘
  │  Registers      │                                 │
  │  (MMIO)         │                                 ▼
  └─────────────────┘                      ┌──────────────────────┐
                                           │   GPIO Subsystem     │
                                           │ (gpiod_set_value,    │
                                           │  gpiod_get_value)    │
                                           └──────────┬───────────┘
                                                      │
                                                      ▼
                                           ┌──────────────────────┐
                                           │  Physical GPIO Pins  │
                                           │  SCLK / MOSI / MISO  │
                                           │  CS0 / CS1 / ...     │
                                           └──────────────────────┘
```

**ملاحظة مهمة:** الـ GPIO Subsystem ده subsystem مستقل. المهمة بتاعته هي abstracting GPIO pins بغض النظر عن الـ GPIO controller hardware. الـ `spi_gpio` بيعتمد عليه لأنه مش عايز يتكلم مع الـ GPIO hardware directly.

---

### الـ Core Abstraction: `spi_controller` كـ Software Controller

الفكرة المحورية في الـ SPI framework هي إن **كل اللي بيوفر SPI bus** — سواء كان hardware حقيقي أو GPIO bitbanging — يتمثل كـ `struct spi_controller`. الـ `spi_gpio` driver بيملأ الـ `spi_controller` ده بـ callbacks بتعمل الـ SPI بإيده.

```c
/*
 * spi_gpio driver بيعمل ده في probe() بتاعه تقريباً:
 * (simplified)
 */
ctlr = spi_alloc_host(&pdev->dev, sizeof(*spi_gpio));
ctlr->bus_num            = pdev->id;           /* رقم الـ bus */
ctlr->num_chipselect     = pdata->num_chipselect;
ctlr->set_cs             = spi_gpio_set_cs;    /* toggle CS via GPIO */
ctlr->transfer_one       = spi_gpio_transfer_one; /* bitbang one transfer */
spi_register_controller(ctlr);
```

الـ `transfer_one` callback هو القلب — فيه الـ bitbanging loop:

```c
/* pseudo-code for bitbang transfer */
for each bit in tx_buf:
    gpiod_set_value(mosi, bit);       /* put bit on MOSI */
    gpiod_set_value(sclk, 1);         /* rising edge */
    rx_bit = gpiod_get_value(miso);   /* sample MISO */
    gpiod_set_value(sclk, 0);         /* falling edge */
    append rx_bit to rx_buf;
```

---

### الـ struct spi_gpio_platform_data بالتفصيل

```c
/**
 * struct spi_gpio_platform_data - platform config for bitbanged SPI host
 * @num_chipselect: كام device ممكن يتوصل على الـ bus ده
 */
struct spi_gpio_platform_data {
    u16 num_chipselect;
};
```

الـ struct دي صغيرة جداً — ليه؟

لأن الـ GPIO pins نفسها (SCLK, MOSI, MISO, CS0, CS1…) بقى الـ kernel يجيبها من الـ Device Tree أو من GPIO lookup tables. زمان كانوا بيحطوا أرقام الـ GPIO مباشرةً في الـ platform_data، لكن ده deprecated. دلوقتي الـ driver بيطلب الـ GPIOs بالاسم:

```c
/* من spi-gpio.c */
spi_gpio->sck  = devm_gpiod_get(dev, "sck",  GPIOD_OUT_LOW);
spi_gpio->mosi = devm_gpiod_get(dev, "mosi", GPIOD_OUT_LOW);
spi_gpio->miso = devm_gpiod_get(dev, "miso", GPIOD_IN);
/* CS pins بتتجيب dynamically حسب num_chipselect */
```

يعني الـ `num_chipselect` هو الـ only piece of info مش ممكن الـ driver يجيبها من غير ما حد يقوله — هو اللي بيحدد كام CS GPIO لازم يطلب الـ driver.

---

### تسجيل الـ Device — ربط كل الحاجات ببعض

الـ board code (أو Device Tree) بيعمل `platform_device` باسم `"spi_gpio"` وبرقم = رقم الـ SPI bus. لما الـ `spi_gpio` driver يتحمّل، الـ platform bus بيشوف إن اسم الـ device بيطابق اسم الـ driver فيعمل `probe()`.

```c
/* board code example (legacy non-DT) */
static struct spi_gpio_platform_data my_spi_gpio_data = {
    .num_chipselect = 2,  /* عندنا chipين على الـ bus ده */
};

static struct platform_device my_spi_gpio_device = {
    .name = "spi_gpio",
    .id   = 1,            /* ده SPI bus رقم 1 */
    .dev  = {
        .platform_data = &my_spi_gpio_data,
    },
};
```

---

### علاقة الـ Structs ببعض — مخطط شامل

```
platform_device
┌──────────────────────────────┐
│ .name = "spi_gpio"           │
│ .id   = 1  (bus number)      │
│ .dev.platform_data ──────────┼──► spi_gpio_platform_data
└──────────────────────────────┘     ┌───────────────────┐
                                     │ .num_chipselect=2  │
                                     └───────────────────┘
         │
         │ probe() matches & calls
         ▼
spi_gpio driver (spi-gpio.c)
 - allocates spi_controller
 - requests GPIO descriptors (from DT/ACPI/lookup)
 - fills spi_controller callbacks

         │
         │ spi_register_controller()
         ▼
spi_controller
┌──────────────────────────────────┐
│ .bus_num        = 1              │
│ .num_chipselect = 2              │
│ .transfer_one   = spi_gpio_xfer  │◄── bitbang loop
│ .set_cs         = spi_gpio_cs    │◄── toggle GPIO
│ .dev  ──────────────────────────►│ parent = platform_device
└──────────────────────────────────┘
         │
         │ creates children on spi_bus_type
         ▼
spi_device (per child)
┌──────────────────────────────────┐
│ .controller ──────────────────►  │ spi_controller above
│ .chip_select[0] = 0 or 1         │
│ .max_speed_hz                    │
│ .mode (CPOL, CPHA, etc.)         │
└──────────────────────────────────┘
         │
         │ matched to protocol driver
         ▼
spi_driver (e.g. w25q-flash, mcp3201)
┌──────────────────────────────────┐
│ .probe()                         │
│ .remove()                        │
│ calls spi_sync() / spi_async()   │
└──────────────────────────────────┘
```

---

### ما بيملكه الـ SPI Framework مقابل ما بيفوّضه للـ Driver

#### الـ SPI Core Framework بيملك (owns):

- تعريف الـ `spi_bus_type` وعمليات الـ matching بين `spi_device` و `spi_driver`
- الـ message queue وترتيب التنفيذ
- الـ DMA mapping helpers (اختياري)
- الـ sysfs entries للـ devices والـ controllers
- الـ locking (bus_lock_mutex) لمنع race conditions
- الـ timeout handling والـ error propagation
- الـ CS GPIO management لو الـ controller طلب `use_gpio_descriptors`

#### الـ spi_gpio Driver بيملك (owns):

- تنفيذ الـ SPI protocol نفسه بيت بيت (الـ bitbanging loop)
- اختيار وتهيئة الـ GPIO pins
- الـ timing (الـ delays بين الـ clock edges)
- دعم الـ SPI modes الأربعة (CPOL/CPHA combinations)
- LSB-first أو MSB-first

#### الـ Consumer Driver (مثلاً Flash driver) بيملك:

- فهم بروتوكول الـ chip المعين (command bytes, address format, etc.)
- بناء `spi_message` و `spi_transfer` structures
- استدعاء `spi_sync()` أو `spi_async()`
- **مش عارف إيه اللي تحته** — hardware SPI أم GPIO bitbang

---

### متى تستخدم spi_gpio؟

| الحالة | التوصية |
|--------|---------|
| Hardware SPI متاح ومش مشغول | استخدم native SPI driver — أسرع بكتير، بيدعم DMA |
| Hardware SPI pins مشغولة أو مش موجودة | **spi_gpio** هو الحل |
| بتعمل prototype/bringup وعايز تختبر device قبل الـ hardware controller يشتغل | **spi_gpio** مثالي |
| Latency حرج جداً (> 10 MHz) | spi_gpio مش مناسب — bitbanging بطيء |
| مبادلة hardware SPI بـ GPIO SPI لاحقاً | الـ consumer driver هيفضل نفسه بدون تعديل |
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

### ملاحظة على الـ File

الـ `spi_gpio.h` هو header صغير جداً — يحتوي على struct واحد بس هو `spi_gpio_platform_data`. الـ complexity الحقيقية موجودة في الـ driver نفسه (`drivers/spi/spi-gpio.c`) والـ SPI core (`include/linux/spi/spi.h`). الـ header ده هو الـ interface بين الـ board setup code والـ bitbang GPIO SPI driver.

---

### 0. الـ Flags, Enums, Config Options

| Item | Value / Type | الوصف |
|------|-------------|--------|
| `CONFIG_SPI_GPIO` | kernel config | يفعّل الـ bitbang SPI driver باستخدام GPIO |
| `num_chipselect` | `u16` | عدد الـ chip select lines المتاحة (target devices) |
| platform device name | `"spi_gpio"` | اللي بتسجّل بيه الـ platform_device |
| platform device id | SPI bus number | رقم الـ bus اللي الـ driver هيمثّله |

> مفيش enums أو flags في الـ header ده — البساطة مقصودة لأن الـ GPIO numbers باتت تيجي من الـ device tree أو ACPI مش من الـ platform_data.

---

### 1. الـ Structs المهمة

#### `struct spi_gpio_platform_data`

**الغرض:** بتوفر الـ configuration اللي الـ `spi-gpio` driver محتاجه وقت الـ probe على الـ platforms اللي بتستخدم legacy board files (مش device tree).

```c
struct spi_gpio_platform_data {
    u16 num_chipselect;  /* number of CS lines, i.e. max target devices */
};
```

| Field | Type | الوصف |
|-------|------|--------|
| `num_chipselect` | `u16` | بيحدد كام device ممكن يتوصل على الـ bus ده — بيتحوّل لـ `master->num_chipselect` في الـ SPI core |

**الاتصال بـ Structs تانية:**

- **`platform_device.dev.platform_data`** → بيشاور على `spi_gpio_platform_data`
- الـ driver بياخده في `probe()` عن طريق `dev_get_platdata()`
- بيتحوّل لـ `spi_controller->num_chipselect` (الـ `struct spi_controller` في `spi.h`)

---

### 2. الـ Struct Relationship Diagram

```
Board Setup Code (arch/xxx/mach-xxx/board.c)
│
│  defines
▼
struct platform_device {
    .name = "spi_gpio",
    .id   = <bus_number>,
    .dev  = {
        .platform_data ──────────────────────────────────────────┐
    }                                                            │
}                                                               │
                                                                ▼
                                               struct spi_gpio_platform_data {
                                                   u16 num_chipselect;
                                               }
                                                                │
                                                                │ dev_get_platdata()
                                                                ▼
                                               spi-gpio driver (probe)
                                                                │
                                                                │ spi_alloc_host()
                                                                ▼
                                               struct spi_controller {
                                                   u16 num_chipselect;   ← copied from platform_data
                                                   int (*transfer_one)();
                                                   int (*setup)();
                                                   ...
                                               }
                                                                │
                                                                │ spi_register_controller()
                                                                ▼
                                               SPI Core (drivers/spi/spi.c)
                                                                │
                                                                ▼
                                               struct spi_device (per target)
                                                   .chip_select < num_chipselect
```

---

### 3. الـ Lifecycle Diagram

```
[Board Init / DT Probe]
        │
        │  platform_device_register() with spi_gpio_platform_data
        ▼
[spi-gpio driver: probe()]
        │
        ├─ dev_get_platdata()  →  reads num_chipselect
        │
        ├─ spi_alloc_host()   →  allocates struct spi_controller
        │
        ├─ sets controller->num_chipselect = pdata->num_chipselect
        │
        ├─ sets controller->transfer_one_message = spi_gpio_transfer
        │
        ├─ gpiod_get() for MOSI, MISO, SCK  (from board info or DT)
        │
        ├─ spi_register_controller()  →  SPI core يعرف الـ bus
        │
        └─ [READY] — SPI devices يقدروا يتسجّلوا على الـ bus
                │
                ▼
        [Runtime: spi_sync() / spi_async()]
                │
                ▼
        [spi-gpio bitbangs GPIO lines manually]
                │
                ├─ gpiod_set_value(SCK, 0/1)
                ├─ gpiod_set_value(MOSI, bit)
                └─ gpiod_get_value(MISO)

[Driver Remove / Shutdown]
        │
        ├─ spi_unregister_controller()
        ├─ gpiod_put() for all GPIO lines
        └─ spi_controller_put()
```

---

### 4. الـ Call Flow Diagrams

#### الـ Probe Flow (وقت التسجيل)

```
platform_driver_probe("spi_gpio")
  └─ spi_gpio_probe(pdev)
       ├─ pdata = dev_get_platdata(&pdev->dev)
       │      reads: pdata->num_chipselect
       │
       ├─ spi_alloc_host(&pdev->dev, sizeof(struct spi_gpio))
       │      allocates: spi_controller + private spi_gpio struct
       │
       ├─ spi_gpio->mosi = gpiod_get_optional(dev, "mosi", GPIOD_OUT_LOW)
       ├─ spi_gpio->miso = gpiod_get_optional(dev, "miso", GPIOD_IN)
       ├─ spi_gpio->sck  = gpiod_get(dev, "sck",  GPIOD_OUT_LOW)
       │
       ├─ master->num_chipselect = pdata ? pdata->num_chipselect : 1
       ├─ master->transfer_one   = spi_gpio_transfer_one
       ├─ master->set_cs         = spi_gpio_set_cs
       │
       └─ spi_register_controller(master)
              └─ SPI core scans for spi_board_info / DT nodes
                   └─ spi_new_device() per target → struct spi_device
```

#### الـ Transfer Flow (وقت الـ runtime)

```
spi_sync(spi_device, message)
  └─ SPI core queues message
       └─ spi_gpio_transfer_one(master, spi_device, transfer)
            ├─ bitbang_txrx_8()  or  bitbang_txrx_16()
            │    ├─ loop over bits:
            │    │    ├─ setmosi(sck=0, bit)   → gpiod_set_value(mosi_gpio, bit)
            │    │    ├─ setsck(1)             → gpiod_set_value(sck_gpio, 1)
            │    │    ├─ rxbit = getmiso()     → gpiod_get_value(miso_gpio)
            │    │    └─ setsck(0)             → gpiod_set_value(sck_gpio, 0)
            │    └─ returns received word
            └─ marks transfer complete → spi_finalize_current_transfer()
```

---

### 5. الـ Locking Strategy

الـ `spi_gpio.h` نفسه مفيهوش أي locks — هو بس data definition. الـ locking كله في طبقات تانية:

| الطبقة | الـ Lock | بيحمي إيه |
|--------|---------|-----------|
| SPI Core | `ctlr->bus_lock_mutex` | يمنع concurrent transfers على نفس الـ bus |
| SPI Core | `ctlr->io_mutex` | يحمي الـ transfer queue |
| GPIO subsystem | `gpio_lock` (spinlock داخلي) | يحمي الـ GPIO descriptor state |
| Bitbang layer | لا يوجد extra lock | الـ transfers بتتنفذ sequentially من الـ SPI workqueue |

**ترتيب الـ Locking (lock ordering):**
```
bus_lock_mutex
  └─ io_mutex
       └─ gpio subsystem internal lock
```

> الـ bitbang GPIO SPI driver مش thread-safe بطبيعته — بيعتمد على الـ SPI core إنه يضمن إن في transfer واحد بس بيشتغل في نفس الوقت على الـ bus عن طريق الـ `bus_lock_mutex`.
## Phase 4: شرح الـ Functions

> الـ header `spi_gpio.h` بيعرّف structure واحدة بس (`spi_gpio_platform_data`)، لكن الـ driver الحقيقي موجود في `drivers/spi/spi-gpio.c` — وده اللي فيه كل الـ functions. الشرح ده بيغطي الاتنين مع بعض.

---

### ملخص الـ API — Cheatsheet

#### الـ Data Structures

| الاسم | الموقع | الغرض |
|---|---|---|
| `struct spi_gpio_platform_data` | `include/linux/spi/spi_gpio.h` | Platform data بتحدد عدد الـ chip-selects |
| `struct spi_gpio` | `drivers/spi/spi-gpio.c` | الـ private state للـ driver — بيحتوي على `spi_bitbang` + الـ GPIO descriptors |
| `struct spi_bitbang` | `include/linux/spi/spi_bitbang.h` | الـ bitbang framework state — callbacks + lock + controller ref |

#### الـ Functions بالـ Category

| الـ Function | الـ Category | الوصف المختصر |
|---|---|---|
| `spi_gpio_probe()` | Registration | نقطة الدخول الرئيسية — بيسجّل الـ controller |
| `spi_gpio_probe_pdata()` | Registration | بيهيّأ الـ CS GPIOs من platform data |
| `spi_gpio_request()` | Registration | بيطلب الـ GPIO descriptors (SCK, MOSI, MISO) |
| `spi_gpio_setup()` | Runtime | بيهيّأ كل `spi_device` جديد — بيتصل من الـ SPI core |
| `spi_gpio_chipselect()` | Runtime | بيتحكم في خط الـ CS + يضبط CPOL عند التفعيل |
| `spi_gpio_set_direction()` | Runtime | بيحول MOSI بين output/input في وضع 3-wire |
| `spi_gpio_set_mosi_idle()` | Runtime | بيضبط مستوى MOSI وهو idle |
| `spi_gpio_cleanup()` | Cleanup | بيعمل cleanup للـ `spi_device` |
| `spi_to_spi_gpio()` | Helper | بيجيب pointer على `spi_gpio` من `spi_device` |
| `setsck()` | GPIO Helper | بيضبط خط الـ SCK |
| `setmosi()` | GPIO Helper | بيكتب بت على خط MOSI |
| `getmiso()` | GPIO Helper | بيقرأ بت من خط MISO (أو MOSI في 3-wire) |
| `spi_gpio_txrx_word_mode0/1/2/3()` | TX/RX | بيعمل TX/RX لكلمة واحدة — الـ 4 modes SPI |
| `spi_gpio_spec_txrx_word_mode0/1/2/3()` | TX/RX | نفسه لكن بيراعي NO_TX/NO_RX flags من الـ controller |

---

### Group 1: Registration & Probe

الغرض من الـ group ده إنه يعمل allocate للـ controller، يطلب الـ GPIO resources، يربط الـ callbacks بالـ bitbang framework، وفي الآخر يسجّل الـ SPI controller في الـ kernel.

---

#### `spi_gpio_probe()`

```c
static int spi_gpio_probe(struct platform_device *pdev)
```

ده الـ `.probe` callback للـ `platform_driver`. بيعمل allocate لـ `spi_controller` (مع embedding الـ `spi_gpio` struct جوّاه)، بيحدد الـ capabilities، وبيسجّل الـ controller.

**Parameters:**
- `pdev` — الـ `platform_device` اللي الـ driver اتربط بيه.

**Return value:**
- `0` عند النجاح، أو error code سالب (`-ENOMEM`, `-EPROBE_DEFER`, etc.).

**التدفق بالتفصيل:**

```
devm_spi_alloc_host()         // alloc controller + spi_gpio embedded
    |
    ├── fwnode موجود؟
    │   └── use_gpio_descriptors = true  // DT/ACPI يتكلم مع gpiolib مباشرة
    │
    └── لا → spi_gpio_probe_pdata()      // هيّئ CS من platform_data
            |
            └── devm_gpiod_get_index() لكل CS

spi_gpio_request()            // اطلب SCK, MOSI, MISO GPIOs
    |
    ├── ضبط host->bits_per_word_mask, mode_bits
    ├── لو مفيش MOSI → SPI_CONTROLLER_NO_TX
    ├── ربط callbacks: setup, cleanup, chipselect, set_line_direction
    │
    ├── اختيار txrx_word callbacks:
    │   ├── spec_* لو NO_TX/NO_RX محتمل
    │   └── normal لو كل الـ pins موجودة
    │
    └── spi_bitbang_init() → devm_spi_register_controller()
```

**Key details:**
- استخدام `devm_*` بيضمن إن كل الـ resources اتحررت تلقائيًا عند الـ remove.
- الـ flag `SPI_CONTROLLER_GPIO_SS` بيجبر الـ bitbang framework يستدعي `chipselect` callback دايمًا حتى لو مفيش hardware CS.

**Caller:** الـ kernel بيستدعيه عند match الـ platform device مع الـ driver.

---

#### `spi_gpio_probe_pdata()`

```c
static int spi_gpio_probe_pdata(struct platform_device *pdev,
                                struct spi_controller *host)
```

بيستخدم الـ `spi_gpio_platform_data` اللي جايا من الـ `platform_device` عشان يعمل alloc لـ array من الـ CS GPIO descriptors. ده مسار الـ legacy non-DT فقط.

**Parameters:**
- `pdev` — الـ platform device (بيجيب منه الـ platform data).
- `host` — الـ SPI controller اللي هيتم تعبئة `num_chipselect` فيه.

**Return value:**
- `0` عند النجاح أو لو `num_chipselect == 0` (يعني single always-selected device).
- `-ENODEV` لو مفيش platform data.
- `-ENOMEM` لو فشل الـ alloc.
- Error من `devm_gpiod_get_index()` لو GPIO descriptor مش موجود.

**Key details:**
- لو `pdata->num_chipselect == 0`، الدالة بترجع 0 من غير ما تعمل حاجة، وده معناه إن في device واحد دايمًا selected من غير خط CS.
- الـ CS GPIOs بيتعملوا init في `GPIOD_OUT_HIGH` (inactive, active-low default).

---

#### `spi_gpio_request()`

```c
static int spi_gpio_request(struct device *dev, struct spi_gpio *spi_gpio)
```

بيطلب الـ 3 GPIO lines الأساسية: SCK, MOSI, MISO — كلهم بـ `devm_gpiod_get_optional()` ما عدا SCK اللي mandatory.

**Parameters:**
- `dev` — الـ device اللي هيتعمل منسوب ليه الـ GPIO resources.
- `spi_gpio` — الـ private state اللي هتتخزن فيه الـ descriptors.

**Return value:**
- `0` عند النجاح.
- Error سالب لو SCK مش موجود أو لو MOSI/MISO رجعوا error (مش just absent).

**Key details:**
- `MOSI` بيتطلب `GPIOD_OUT_LOW` (output).
- `MISO` بيتطلب `GPIOD_IN` (input).
- `SCK` بيتطلب `GPIOD_OUT_LOW` ولازم يكون موجود.
- لو MOSI `NULL` (optional وغير موجود)، الـ probe بيضبط `SPI_CONTROLLER_NO_TX`.

---

### Group 2: Runtime — Device Setup & Teardown

الـ SPI core بيستدعي الـ functions دي عند add/remove كل `spi_device` على الـ bus.

---

#### `spi_gpio_setup()`

```c
static int spi_gpio_setup(struct spi_device *spi)
```

الـ `.setup` callback — بيهيّأ خط الـ CS الخاص بالـ device ده، بعدين بيستدعي `spi_bitbang_setup()` اللي بيعمل validate لـ bits-per-word وبيضبط الـ `txrx_word` function المناسبة للـ mode.

**Parameters:**
- `spi` — الـ `spi_device` اللي بيتسجّل دلوقتي.

**Return value:**
- `0` عند النجاح.
- Error من `gpiod_direction_output()` أو `spi_bitbang_setup()`.

**Key details:**
- الـ CS GPIO بيتحول لـ output inactive (يعني `!SPI_CS_HIGH`) فقط لو `!spi->controller_state` — يعني أول مرة يتعمل setup بس.
- الـ locking بيبقى مسؤولية الـ SPI core قبل ما يستدعي الـ callback ده.

---

#### `spi_gpio_cleanup()`

```c
static void spi_gpio_cleanup(struct spi_device *spi)
```

wrapper بسيط على `spi_bitbang_cleanup()` — بيحرر الـ `controller_state` المخصصة للـ device.

**Parameters:**
- `spi` — الـ device اللي بيتشال من الـ bus.

**Return value:** لا يوجد (void).

---

### Group 3: Runtime — Transfer Execution

---

#### `spi_gpio_chipselect()`

```c
static void spi_gpio_chipselect(struct spi_device *spi, int is_active)
```

بيتحكم في خط الـ CS قبل وبعد كل transfer. لو `is_active`، بيضبط SCK على القيمة الصح حسب CPOL قبل ما يشغّل الـ CS.

**Parameters:**
- `spi` — الـ target device.
- `is_active` — `1` عشان يفعّل (يختار الـ device)، `0` عشان يلغّي التفعيل.

**Return value:** لا يوجد (void).

**Key details:**
- الـ SCK بيتضبط على `SPI_CPOL` bit من `spi->mode` — ده ضروري عشان الـ clock يبدأ من الـ idle level الصح قبل أول edge.
- الـ CS polarity بتتعامل معاها صح: `SPI_CS_HIGH` → active high، غير كده active low.
- لو `cs_gpios` لا يوجد (single device)، بيتجاهل التحكم في الـ CS GPIO.

---

#### `spi_gpio_set_direction()`

```c
static int spi_gpio_set_direction(struct spi_device *spi, bool output)
```

الـ bitbang framework بيستدعيها في الـ 3-wire mode عشان يحول خط MOSI بين TX (output) وRX (input). بيضيف turnaround cycle في وضع `SPI_3WIRE_HIZ`.

**Parameters:**
- `spi` — الـ target device.
- `output` — `true` يحول MOSI لـ output، `false` يحوله لـ input.

**Return value:**
- `0` عند النجاح.
- Error من `gpiod_direction_input()` لو فشل.

**Key details:**
- الـ input direction بيتعمل **فقط** لو `SPI_3WIRE` mode — ده بيحمي الـ bus من float.
- الـ turnaround cycle (toggle SCK مرتين) بيتعمل لو `SPI_3WIRE_HIZ` — ده بيحط الـ line في high-impedance أثناء التحويل.
- `spidelay()` في الـ driver ده `do {} while(0)` — يعني مفيش timing delay فعلي.

---

#### `spi_gpio_set_mosi_idle()`

```c
static void spi_gpio_set_mosi_idle(struct spi_device *spi)
```

بيضبط مستوى MOSI لما الـ bus مش شاغل. بيراعي الـ `SPI_MOSI_IDLE_HIGH` flag من `spi->mode`.

**Parameters:**
- `spi` — الـ target device.

**Return value:** لا يوجد (void).

**Key details:**
- مهم لـ protocols اللي بتحدد مستوى MOSI بين الـ transfers عشان الـ target device ميختلطش.

---

### Group 4: GPIO Low-Level Helpers

الـ 3 functions دول هم الـ "hardware abstraction" الوحيدة في الـ driver — كل الـ bitbanging بيمشي عليهم.

---

#### `spi_to_spi_gpio()`

```c
static inline struct spi_gpio *__pure
spi_to_spi_gpio(const struct spi_device *spi)
```

بيعمل reverse lookup من الـ `spi_device` للـ `spi_gpio` private struct عن طريق `container_of` من خلال الـ `spi_bitbang`.

**Parameters:**
- `spi` — الـ SPI device.

**Return value:** Pointer على `struct spi_gpio`.

**Key details:**
- الـ attribute `__pure` بيقول للـ compiler إن الـ function دي بترجع نفس النتيجة لنفس الـ input — بيساعد الـ optimizer يشيل استدعاءات مكررة.
- مفيش locking هنا — الـ pointer نفسه ثابت بعد الـ probe.

---

#### `setsck()`

```c
static inline void setsck(const struct spi_device *spi, int is_on)
```

بيضبط خط SCK على القيمة المطلوبة. `gpiod_set_value_cansleep()` بتضمن إن حتى لو الـ GPIO controller بيحتاج sleep (مثلاً I2C GPIO expander) الـ call هتشتغل من process context.

**Parameters:**
- `spi` — الـ SPI device.
- `is_on` — `1` = high، `0` = low.

---

#### `setmosi()`

```c
static inline void setmosi(const struct spi_device *spi, int is_on)
```

بيكتب بت واحد على خط MOSI. نفس pattern الـ `setsck`.

---

#### `getmiso()`

```c
static inline int getmiso(const struct spi_device *spi)
```

بيقرأ مستوى خط الـ data الوارد. في وضع `SPI_3WIRE`، بيقرأ من MOSI (لأن الـ line المشتركة) — في الـ full-duplex بيقرأ من MISO.

**Return value:**
- `1` أو `0` (بعد double negation `!!` عشان يضمن تنظيف الـ value).

---

### Group 5: TX/RX Word Functions

الـ bitbang framework بيستدعي الـ callbacks دي لكل **كلمة** (word) على حدة. في 4 modes × 2 variants = 8 functions.

---

#### `spi_gpio_txrx_word_mode0/1/2/3()`

```c
static u32 spi_gpio_txrx_word_mode0(struct spi_device *spi,
        unsigned int nsecs, u32 word, u8 bits, unsigned int flags)
```

بيعمل TX/RX لكلمة واحدة في الـ SPI mode المحدد (0-3). الـ mode بيتحدد من خلال تركيب CPOL (حالة SCK idle) وCPHA (متى بيتعمل sample للبيانات).

| الـ Mode | CPOL | CPHA | الـ Function الداخلية |
|---|---|---|---|
| Mode 0 | 0 | 0 | `bitbang_txrx_be_cpha0(cpol=0)` |
| Mode 1 | 0 | 1 | `bitbang_txrx_be_cpha1(cpol=0)` |
| Mode 2 | 1 | 0 | `bitbang_txrx_be_cpha0(cpol=1)` |
| Mode 3 | 1 | 1 | `bitbang_txrx_be_cpha1(cpol=1)` |

**Parameters:**
- `spi` — الـ device.
- `nsecs` — الـ half-period delay (في الـ driver ده = 0 فعليًا بسبب `spidelay` macro).
- `word` — الكلمة اللي هتتبعتها.
- `bits` — عدد الـ bits في الكلمة (1-32).
- `flags` — بيتضمن `SPI_CONTROLLER_NO_RX`/`SPI_CONTROLLER_NO_TX` hints.

**Return value:** الكلمة اللي اتستقبلت (received word).

**Key details:**
- لو `SPI_LSB_FIRST` في `spi->mode`، بيستدعي `bitbang_txrx_le_*` عوض `_be_*` — ده بيقلب ترتيب البتات.
- الـ `flags` parameter في الـ `mode*` الأصلية بييجي من الـ caller (الـ bitbang framework)، مش من الـ controller.

---

#### `spi_gpio_spec_txrx_word_mode0/1/2/3()`

```c
static u32 spi_gpio_spec_txrx_word_mode0(struct spi_device *spi,
        unsigned int nsecs, u32 word, u8 bits, unsigned int flags)
```

نفس الـ `mode*` العادية لكن بيعمل **override** على الـ `flags` بـ `spi->controller->flags` — ده بيخلي الـ compiler يشوف الـ flags كـ runtime value وبكده يعمل تحقق صحيح من `SPI_CONTROLLER_NO_RX`/`SPI_CONTROLLER_NO_TX`.

**لماذا variant منفصلة؟** لأن لو الـ flags جاية كـ compile-time constant (من الـ framework)، الـ compiler ممكن يـ optimize ويشيل الـ checks للـ NO_RX/NO_TX. لكن لما الـ controller نفسه ممكن يبقى missing MOSI أو MISO في runtime، لازم الـ flags تيجي من الـ controller مباشرة في كل call.

---

### تدفق الـ Probe الكامل — ASCII Diagram

```
platform_device "spi_gpio" match
          |
          v
    spi_gpio_probe()
          |
          |── devm_spi_alloc_host()
          |       └── embedded spi_gpio struct
          |
          |── DT/ACPI path ──────────────── pdata path
          |   use_gpio_descriptors=true     spi_gpio_probe_pdata()
          |                                     └── alloc cs_gpios[]
          |                                         devm_gpiod_get_index() x N
          |
          |── spi_gpio_request()
          |       ├── mosi ← devm_gpiod_get_optional("mosi", OUT_LOW)
          |       ├── miso ← devm_gpiod_get_optional("miso", IN)
          |       └── sck  ← devm_gpiod_get("sck", OUT_LOW)  [mandatory]
          |
          |── ضبط host capabilities (bits_per_word, mode_bits, flags)
          |
          |── ربط callbacks في spi_bitbang:
          |       ├── chipselect    → spi_gpio_chipselect
          |       ├── set_mosi_idle → spi_gpio_set_mosi_idle
          |       ├── set_line_direction → spi_gpio_set_direction
          |       ├── setup_transfer → spi_bitbang_setup_transfer
          |       └── txrx_word[0..3] → mode0..3 (أو spec_ variant)
          |
          |── spi_bitbang_init()
          |
          └── devm_spi_register_controller()
                    └── SPI core يبدأ يقبل spi_device registrations
```

---

### تدفق الـ Transfer لكل Word

```
SPI core يستدعي spi_bitbang txrx_bufs()
          |
          └── لكل word في الـ transfer:
                    |
                    v
          spi_gpio_txrx_word_modeX()
                    |
                    v
          bitbang_txrx_be_cphaX()   (من spi-bitbang-txrx.h)
                    |
          لكل bit (MSB first):
          ├── CPHA=0: setmosi → setsck(1) → getmiso → setsck(0)
          └── CPHA=1: setsck(1) → setmosi → setsck(0) → getmiso
                    |
                    v
          return received_word
```

---

### ملاحظة على الـ `spidelay` Macro

```c
#define spidelay(nsecs)	do {} while (0)
```

الـ driver عمدًا بيلغّي الـ delay لأن الـ GPIO bitbanging بطيء كفاية إنه مش محتاج delay إضافي. السرعة القصوى عمليًا هي ~1 Mbit/s حتى من غير delays.
## Phase 5: دليل الـ Debugging الشامل

الـ `spi_gpio` driver بيشتغل كـ **bitbanged SPI host controller** فوق GPIO lines عادية — يعني مفيش hardware SPI peripheral، والـ kernel بيعمل كل الـ clock/data timing بالـ software. ده بيخلي الـ debugging له طابع خاص: المشاكل ممكن تكون في الـ GPIO state، في الـ timing، في الـ platform_data configuration، أو في الـ SPI protocol نفسه.

---

### Software Level

#### 1. debugfs Entries

الـ SPI subsystem بيعمل entries في `/sys/kernel/debug/spi/` — لازم تبقى `CONFIG_DEBUG_FS=y`.

```bash
# اعرض كل الـ SPI controllers الموجودة
ls /sys/kernel/debug/spi/

# الـ spi_gpio controller بيظهر باسم زي spi0 أو spiX حسب الـ bus id
ls /sys/kernel/debug/spi/spi0/

# إقرأ الـ statistics الخاصة بالـ controller
cat /sys/kernel/debug/spi/spi0/statistics
```

**تفسير الـ output:**
```
messages:            150     <- إجمالي الـ SPI messages المتنفذة
transfers:           300     <- إجمالي الـ transfers (كل message ممكن تحتوي على أكتر من transfer)
errors:                2     <- عدد الـ errors - لو > 0 ابحث في dmesg
timedout:              0     <- timeout = مشكلة في الـ hardware أو الـ GPIO
bytes_tx:           1200     <- bytes اتبعتت
bytes_rx:           1200     <- bytes اتستقبلت
```

لو `errors` أو `timedout` بيزيدوا، ابدأ بفحص الـ GPIO lines وتوقيت الـ clock.

```bash
# statistics خاصة بكل spi_device (target device) على الـ bus
cat /sys/kernel/debug/spi/spi0/spi0.0/statistics
```

#### 2. sysfs Entries

```bash
# اعرض كل الـ SPI devices المتسجلة
ls /sys/bus/spi/devices/

# معلومات الـ spi_gpio controller
cat /sys/bus/spi/devices/spi0/modalias
cat /sys/bus/spi/devices/spi0/driver_override

# الـ max_speed على الـ device
cat /sys/bus/spi/devices/spi0.0/driver       # الـ driver المربوط
ls  /sys/bus/spi/devices/spi0.0/

# الـ GPIO المستخدمة (لو الـ driver بيعرضها)
ls /sys/class/gpio/
# أو من خلال gpiod interface
ls /sys/kernel/debug/gpio
```

```bash
# تحقق من الـ platform device
ls /sys/bus/platform/devices/ | grep spi_gpio
cat /sys/bus/platform/devices/spi_gpio.0/driver

# عدد الـ chipselects المتاحة (من platform_data)
# مش بتظهر مباشرة لكن تقدر تعرفها من:
grep -r num_chipselect /sys/kernel/debug/ 2>/dev/null
```

#### 3. ftrace — Tracepoints والـ Events

```bash
# شوف الـ SPI tracepoints المتاحة
grep spi /sys/kernel/debug/tracing/available_events

# أهم الـ events للـ spi_gpio debugging:
# spi:spi_controller_idle
# spi:spi_controller_busy
# spi:spi_message_submit
# spi:spi_message_start
# spi:spi_message_done
# spi:spi_transfer_start
# spi:spi_transfer_stop

# فعّل الـ events وابدأ الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on
echo "spi:*" > /sys/kernel/debug/tracing/set_event

# شغّل الـ SPI communication اللي عاوز تتابعها
# مثلاً: اقرأ من device على الـ bus

# إقرأ الـ trace
cat /sys/kernel/debug/tracing/trace

# إيقاف الـ tracing
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo "" > /sys/kernel/debug/tracing/set_event
```

**مثال على الـ trace output وتفسيره:**
```
# TASK-PID     CPU#  TIMESTAMP  FUNCTION
  app-1234    [000]  123.456:  spi_message_submit: spi0.0
  spi0-1235   [000]  123.457:  spi_message_start:  spi0.0
  spi0-1235   [000]  123.458:  spi_transfer_start: spi0.0 len=4
  spi0-1235   [000]  123.510:  spi_transfer_stop:  spi0.0 len=4  <- 52µs لـ 4 bytes = ضعيفة لكن طبيعية للـ bitbang
  spi0-1235   [000]  123.511:  spi_message_done:   spi0.0 status=0
```

الوقت بين `transfer_start` و `transfer_stop` مهم — الـ bitbang بيكون أبطأ بكتير من الـ hardware SPI.

```bash
# تتبع GPIO operations مع SPI (لمشاكل الـ timing)
echo "gpio:*" >> /sys/kernel/debug/tracing/set_event
echo "spi:*"  >> /sys/kernel/debug/tracing/set_event
cat /sys/kernel/debug/tracing/trace | grep -A2 -B2 "gpio_set_value"
```

#### 4. printk و Dynamic Debug

```bash
# فعّل الـ dynamic debug للـ spi_gpio driver
echo "module spi_gpio +p" > /sys/kernel/debug/dynamic_debug/control

# أو فعّل كل الـ SPI subsystem
echo "module spi +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل الـ bitbang layer كمان (مهم جداً لأن spi_gpio بيبني فوقه)
echo "module spi_bitbang +p" > /sys/kernel/debug/dynamic_debug/control

# شوف الإخراج
dmesg -w | grep -E "(spi_gpio|spi_bitbang|SPI)"

# فعّل كل الـ levels (verbose جداً - استخدم بحذر)
echo "module spi_gpio +pflmt" > /sys/kernel/debug/dynamic_debug/control
# +p = printk, +f = function name, +l = line number, +m = module, +t = thread id
```

```bash
# لتفعيل dynamic debug من kernel cmdline (مفيد لمشاكل الـ boot-time)
# أضف في /etc/default/grub:
# GRUB_CMDLINE_LINUX="dyndbg=\"module spi_gpio +p\" dyndbg=\"module spi_bitbang +p\""
```

#### 5. Kernel Config Options للـ Debugging

| Config Option | الغرض | ملاحظة |
|---|---|---|
| `CONFIG_SPI_DEBUG` | يفعّل الـ debug messages في SPI core | أساسي |
| `CONFIG_SPI_GPIO` | يبني الـ driver كـ module (مش built-in) للتحميل/التفريغ السهل | مهم للـ dev |
| `CONFIG_DEBUG_GPIO` | يضيف checks على الـ GPIO operations | لمشاكل الـ GPIO |
| `CONFIG_GPIOLIB_IRQCHIP` | debugging لـ GPIO IRQ chains | لو بيستخدم interrupts |
| `CONFIG_DEBUG_SPINLOCK` | يكشف spinlock races في الـ SPI core | لمشاكل الـ concurrency |
| `CONFIG_LOCKDEP` | يكتشف deadlocks في الـ mutex/lock hierarchy | مفيد مع الـ spi_bitbang mutex |
| `CONFIG_PROVE_LOCKING` | strict lock ordering checks | لـ production debugging |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل `pr_debug` و `dev_dbg` في الـ runtime | ضروري |
| `CONFIG_TRACING` | يفعّل ftrace infrastructure | للـ tracepoints |
| `CONFIG_KASAN` | كشف memory corruption | لو في crashes |
| `CONFIG_DEBUG_FS` | يفعّل `/sys/kernel/debug/` | ضروري لكل ما فات |

```bash
# تحقق من الـ configs الحالية
zcat /proc/config.gz | grep -E "CONFIG_(SPI|GPIO|DEBUG)" | grep -v "^#"
# أو
grep -E "CONFIG_(SPI|GPIO|DEBUG)" /boot/config-$(uname -r)
```

#### 6. أدوات خاصة بالـ Subsystem

```bash
# spi-tools: أداة userspace لاختبار الـ SPI devices مباشرة
# تقدر تستخدم /dev/spidevX.Y لو SPI_GPIO مع spidev مفعّل

# اختبار read/write مباشر (يحتاج CONFIG_SPI_SPIDEV)
apt install spi-tools   # على Debian/Ubuntu
spi-config /dev/spidev0.0   # عرض الـ configuration

# أو استخدام spidev_test من kernel samples
ls /usr/share/doc/linux-doc/examples/spi/  # لو موجودة
# أو compile من kernel source:
# tools/spi/spidev_test.c

# اختبار مباشر بـ Python (سريع للتشخيص)
python3 -c "
import spidev
spi = spidev.SpiDev()
spi.open(0, 0)          # bus=0, device=0
spi.max_speed_hz = 1000  # 1KHz - بطيء للـ bitbang اختبار
resp = spi.xfer2([0xAA, 0x55])
print([hex(b) for b in resp])
spi.close()
"
```

```bash
# gpiodetect و gpioinfo لفحص الـ GPIO lines المستخدمة
apt install gpiod
gpiodetect          # اعرض كل الـ GPIO controllers
gpioinfo gpiochip0  # اعرض حالة كل الـ GPIO lines

# تحقق أن الـ lines المستخدمة للـ SPI_GPIO مش محجوزة من حاجة تانية
gpioinfo | grep -E "(MOSI|MISO|CLK|CS|spi)"
```

#### 7. جدول Common Error Messages

| رسالة الـ kernel | المعنى | الحل |
|---|---|---|
| `spi_gpio: probe of spi_gpio.N failed with error -ENODEV` | الـ platform_device مش موجود أو `num_chipselect` = 0 | تحقق من `struct spi_gpio_platform_data` في الـ board file أو DT |
| `spi_gpio: cannot find CLK gpio` | الـ GPIO المخصص لـ clock مش موجود أو مش متعرّف | تحقق من DT property `gpio-sck` أو الـ GPIO numbering |
| `spi_gpio: cannot find MOSI gpio` | نفس المشكلة للـ MOSI line | تحقق من `gpio-mosi` في الـ DT |
| `spi_bitbang: bogus setup` | `bits_per_word` خارج الـ range المدعوم | الـ spi_gpio بيدعم 1–32 bits، تحقق من `spi_device.bits_per_word` |
| `SPI setup() failed: -EINVAL` | إعداد غلط في الـ SPI mode أو الـ speed | تحقق من `spi_device.mode` و `max_speed_hz` |
| `spi_gpio: chipselect N: got -EBUSY` | الـ GPIO line للـ CS محجوز من driver تاني | افحص `/sys/kernel/debug/gpio` واعرف مين حاجز الـ GPIO |
| `spi transfer timed out` | الـ transfer مش اكتمل في الوقت | مشكلة في الـ GPIO operations أو الـ system load عالي جداً |
| `cannot transfer buffers` | الـ txrx_bufs callback فشل | الـ GPIO مش responsive، تحقق من الـ hardware |
| `spi_gpio probe failed -EPROBE_DEFER` | dependency مش جاهزة (مثلاً GPIO controller) | طبيعي في الـ boot — هيجرب تاني |

#### 8. Strategic Points لـ dump_stack() و WARN_ON()

```c
/* في spi_gpio_chipselect() — لو CS مش بيتفعّل صح */
static void spi_gpio_chipselect(struct spi_device *spi, int is_on)
{
    struct spi_gpio *spi_gpio = spi_to_spi_gpio(spi);

    /* WARN لو الـ gpio_desc مش موجود */
    WARN_ON(!spi->cs_gpiod[0]);

    /* dump_stack لفهم من بيستدعي الـ CS في وقت غلط */
    if (unlikely(is_on && spi_gpio->busy_flag))
        dump_stack();
}

/* في spi_bitbang_transfer_setup() — تحقق من الـ mode */
WARN_ON(spi->mode & ~(SPI_CPOL | SPI_CPHA | SPI_CS_HIGH | SPI_3WIRE));

/* في الـ probe() — تحقق من platform_data */
if (WARN_ON(!pdata || pdata->num_chipselect == 0))
    return -EINVAL;
```

نقاط استراتيجية للـ `WARN_ON()` في الـ spi_gpio flow:

1. **بعد `devm_gpiod_get()`** — تحقق إن الـ GPIO lines اتحجزت صح.
2. **في بداية كل `txrx_word()`** — تحقق إن الـ controller مش في حالة غريبة.
3. **في `chipselect()` callback** — تحقق إن الـ CS GPIO مش NULL.
4. **في `setup_transfer()`** — تحقق إن الـ speed مش أعلى من الـ maximum المسموح بيه للـ bitbang.

---

### Hardware Level

#### 1. التحقق إن حالة الـ Hardware مطابقة لحالة الـ Kernel

```bash
# الطريقة الأكثر مباشرة: قارن حالة الـ GPIO في الـ kernel مع الـ multimeter/scope

# اعرض الحالة الحالية لكل GPIO lines
cat /sys/kernel/debug/gpio
# الـ output هيبين:
#  gpio-XX (spi-gpio-MOSI        ) out lo    <- MOSI في حالة LOW
#  gpio-XX (spi-gpio-MISO        ) in  hi    <- MISO في HIGH
#  gpio-XX (spi-gpio-CLK         ) out lo    <- CLK idle
#  gpio-XX (spi-gpio-CS          ) out hi    <- CS inactive (active low)

# الـ sysfs interface للـ GPIO (legacy - لا تزال شغالة)
cat /sys/class/gpio/gpio<N>/value   # 0 أو 1
cat /sys/class/gpio/gpio<N>/direction  # in أو out
```

**مقارنة الـ kernel state بالـ hardware:**
- CLK في idle يفترض يكون LOW لـ `SPI_MODE_0` و `SPI_MODE_1`، وHIGH لـ `SPI_MODE_2` و `SPI_MODE_3` (الـ CPOL bit).
- CS يفترض يكون HIGH في idle لو `SPI_CS_HIGH` مش مضبوط (active low by default).
- بعد transfer، تحقق إن CLK رجع لـ idle state.

#### 2. Register Dump Techniques

الـ `spi_gpio` مبيستخدمش registers بشكل مباشر — بيعتمد على GPIO controller. للوصول لـ GPIO controller registers:

```bash
# معرفة الـ base address للـ GPIO controller من DT أو /proc/iomem
cat /proc/iomem | grep -i gpio
# مثال output:
# fe200000-fe2000b3 : /soc/gpio@fe200000

# استخدام devmem2 لقراءة الـ GPIO registers (مثال على Raspberry Pi BCM2835)
apt install devmem2

# قراءة GPIO Function Select register (بيحدد IN/OUT/ALT)
devmem2 0xfe200000 w    # GPFSEL0 - pins 0-9
devmem2 0xfe200004 w    # GPFSEL1 - pins 10-19
devmem2 0xfe200008 w    # GPFSEL2 - pins 20-29

# قراءة GPIO level register (الحالة الحالية)
devmem2 0xfe200034 w    # GPLEV0 - pins 0-31
devmem2 0xfe200038 w    # GPLEV1 - pins 32-53
```

```bash
# طريقة أمان أكثر: /dev/mem مع io utility
# التأكد إن CONFIG_STRICT_DEVMEM مش مفعّل أو تستخدم JTAG

# لو بتشتغل على embedded system بدون devmem2:
python3 -c "
import mmap, os, struct
fd = os.open('/dev/mem', os.O_RDONLY | os.O_SYNC)
BASE = 0xfe200000
SIZE = 0x100
mem = mmap.mmap(fd, SIZE, mmap.MAP_SHARED, mmap.PROT_READ, offset=BASE)
# GPLEV0
mem.seek(0x34)
val = struct.unpack('<I', mem.read(4))[0]
print(f'GPIO levels 0-31: 0x{val:08x}')
# فحص bit معين، مثلاً GPIO18 (SPI CLK على RPi)
print(f'GPIO18 (CLK): {(val >> 18) & 1}')
mem.close()
os.close(fd)
"
```

#### 3. Logic Analyzer / Oscilloscope Tips

**نقاط القياس:**
```
     MCU/SoC
      ────────────────────────────────────
      │  GPIO_CLK  ──────────────────────── CH1 (Clock)
      │  GPIO_MOSI ──────────────────────── CH2 (Data Out)
      │  GPIO_MISO ──────────────────────── CH3 (Data In)
      │  GPIO_CS   ──────────────────────── CH4 (Chip Select)
      ────────────────────────────────────
                          │
                     [Target Device]
```

**إعدادات الـ Logic Analyzer:**
- **Sample rate**: على الأقل 10x أسرع من الـ SPI clock. مثلاً لو bitbang بيعمل 100KHz، استخدم 1MHz sample rate.
- **Trigger**: على falling edge للـ CS line.
- **Protocol decoder**: فعّل SPI decoder في PulseView/Sigrok.

```bash
# استخدام sigrok-cli مع logic analyzer
sigrok-cli -d fx2lafw \
  --config samplerate=2m \
  --samples 1000000 \
  --channels D0=CLK,D1=MOSI,D2=MISO,D3=CS \
  --protocol-decoders spi:clk=CLK:mosi=MOSI:miso=MISO:cs=CS \
  --output-format ascii

# لو بتستخدم PulseView (GUI)
pulseview &
```

**أشياء تدور عليها:**
1. **الـ setup/hold time**: الـ bitbang بيكون بطيء بما يكفي عادة، لكن تحقق إن الـ data stable قبل الـ clock edge.
2. **الـ CS glitches**: لو CS بيعمل glitch في وسط الـ transfer.
3. **الـ MISO floating**: لو الـ target device مش connected، MISO ممكن يكون floating وتاخد garbage data.
4. **الـ clock idle level**: لازم يطابق الـ CPOL setting.

#### 4. Common Hardware Issues وكيف تظهر في الـ Kernel Log

| المشكلة الـ Hardware | الـ Pattern في dmesg | التشخيص |
|---|---|---|
| الـ MISO line مش متوصلة | `spi0.0: error reading data` أو garbage في الـ rx buffer | قيّس الـ MISO بالـ multimeter — لازم يكون driven من الـ slave |
| المقاومة Pull-up/Pull-down ناقصة | Transfer succeeds بس بيان غلط | فحص بالـ oscilloscope — هتلاقي slow rise/fall times |
| الـ VCC للـ SPI device مش شغال | `-ETIMEDOUT` أو device مش بيرد على CS | قيّس الـ VCC مباشرة على الـ device |
| Level mismatch (3.3V vs 5V) | Data corruption متكررة | استخدم level shifter، قيّس voltage على كل line |
| الـ CS مش بيوصل للـ device | Device مش بيبدأ الـ transfer أبداً | تحقق من الـ continuity بالـ multimeter |
| Capacitive loading عالي على الـ CLK | Timing errors بشكل متقطع | زوّد الـ drive strength للـ GPIO أو حط resistor مناسب |
| الـ SPI mode غلط (CPOL/CPHA) | أول byte صح والباقي غلط | غيّر الـ mode في الـ spi_board_info أو DT |

#### 5. Device Tree Debugging

```bash
# عرض الـ DT الفعلي اللي اتحمل
# الـ DT الكامل
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "spi_gpio\|spi-gpio"

# أو مباشرة
ls /sys/firmware/devicetree/base/
# ابحث عن node اسمه spi_gpio أو spi@X

# مثال DT صح للـ spi_gpio:
cat /proc/device-tree/spi_gpio/compatible 2>/dev/null
# يجب أن يطبع: spi-gpio

# اعرض كل properties للـ node
cat /sys/firmware/devicetree/base/spi_gpio/gpio-sck
cat /sys/firmware/devicetree/base/spi_gpio/gpio-mosi
cat /sys/firmware/devicetree/base/spi_gpio/gpio-miso

# تحقق من الـ num-chipselects
hexdump /sys/firmware/devicetree/base/spi_gpio/num-chipselects
# يجب أن يطبع رقم بيطابق الـ num_chipselect في platform_data
```

**DT مرجعي صحيح للـ spi_gpio:**
```dts
spi_gpio: spi {
    compatible = "spi-gpio";
    #address-cells = <1>;
    #size-cells = <0>;

    /* GPIO lines — رقم الـ GPIO ومن أي controller */
    gpio-sck  = <&gpio 11 GPIO_ACTIVE_HIGH>;  /* CLK  */
    gpio-mosi = <&gpio 10 GPIO_ACTIVE_HIGH>;  /* MOSI */
    gpio-miso = <&gpio  9 GPIO_ACTIVE_HIGH>;  /* MISO */
    num-chipselects = <1>;

    /* target device على الـ bus */
    flash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;  /* chipselect index */
        spi-max-frequency = <5000000>;  /* 5MHz max للـ bitbang */
        cs-gpios = <&gpio 8 GPIO_ACTIVE_LOW>;
    };
};
```

```bash
# تحقق إن الـ GPIO numbers في DT بتطابق الـ hardware
# قارن مع schematic أو pinout documentation

# لو في DT overlay، تحقق من التطبيق الصحيح
dtoverlay -l    # اعرض الـ overlays المطبقة
dtoverlay -a    # اعرض كل الـ overlays المتاحة

# فحص أخطاء الـ DT parsing في boot
dmesg | grep -E "(OF|DT|devicetree|spi_gpio)" | head -30
```

---

### Practical Commands

#### مجموعة أوامر شاملة لبدء التشخيص

```bash
#!/bin/bash
# spi_gpio_debug.sh — سكريبت تشخيص شامل

echo "=== [1] الـ SPI Controllers ==="
ls /sys/bus/spi/devices/ 2>/dev/null || echo "No SPI devices"
ls /sys/bus/platform/devices/ | grep spi_gpio

echo ""
echo "=== [2] الـ SPI Statistics ==="
for f in /sys/kernel/debug/spi/*/statistics; do
    echo "--- $f ---"
    cat "$f" 2>/dev/null
done

echo ""
echo "=== [3] الـ GPIO State ==="
cat /sys/kernel/debug/gpio | grep -E "(spi|SPI|CLK|MOSI|MISO)"

echo ""
echo "=== [4] الـ Device Tree ==="
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 15 "spi-gpio" | head -50

echo ""
echo "=== [5] آخر 50 رسالة SPI في dmesg ==="
dmesg | grep -i spi | tail -50

echo ""
echo "=== [6] الـ Driver Status ==="
cat /sys/bus/platform/devices/spi_gpio.*/driver 2>/dev/null || echo "Driver not loaded or no device"
lsmod | grep -E "(spi_gpio|spi_bitbang)"
```

```bash
# تشغيل الـ ftrace بطريقة سهلة
function spi_trace_start() {
    echo 0 > /sys/kernel/debug/tracing/tracing_on
    echo "" > /sys/kernel/debug/tracing/trace
    echo "spi:* gpio:*" > /sys/kernel/debug/tracing/set_event
    echo 1 > /sys/kernel/debug/tracing/tracing_on
    echo "Tracing started. Run your SPI operation now."
}

function spi_trace_stop() {
    echo 0 > /sys/kernel/debug/tracing/tracing_on
    echo "=== Trace Results ==="
    cat /sys/kernel/debug/tracing/trace | grep -v "^#" | head -100
    echo "" > /sys/kernel/debug/tracing/set_event
}

# استخدام:
spi_trace_start
# ... شغّل الـ SPI operation ...
spi_trace_stop
```

```bash
# تفعيل dynamic debug بشكل سريع
echo "module spi_gpio +pflmt" > /sys/kernel/debug/dynamic_debug/control
echo "module spi_bitbang +pflmt" > /sys/kernel/debug/dynamic_debug/control
# تابع الـ output
dmesg -w &
# ... شغّل الـ SPI operation ...
# لإيقاف الـ debug
echo "module spi_gpio -p" > /sys/kernel/debug/dynamic_debug/control
echo "module spi_bitbang -p" > /sys/kernel/debug/dynamic_debug/control
```

```bash
# فحص الـ GPIO assignment للـ spi_gpio بالتفصيل
cat /sys/kernel/debug/gpio | grep -B2 -A2 "spi"

# مثال على الـ output المتوقع:
# gpiochip0: GPIOs 0-53, parent: platform/fe200000.gpio, pinctrl-bcm2835:
#  gpio-8  (                    |spi-gpio-cs         ) out hi ACTIVE LOW
#  gpio-9  (                    |spi-gpio-miso       ) in  lo
#  gpio-10 (                    |spi-gpio-mosi       ) out lo
#  gpio-11 (                    |spi-gpio-clk        ) out lo
#
# لو الـ ACTIVE LOW مكتوبة على الـ CS = صح للـ SPI_CS_HIGH غير مفعّل
# لو out hi = CS inactive (لو active low)
```

```bash
# اختبار أداء الـ bitbang: كم transfer في الثانية ممكن؟
time dd if=/dev/urandom bs=1 count=1000 | \
  spidev_test -D /dev/spidev0.0 -s 100000 -p "test" 2>&1

# أو بـ Python للقياس الدقيق
python3 -c "
import spidev, time

spi = spidev.SpiDev()
spi.open(0, 0)
spi.max_speed_hz = 100000

N = 100
data = [0xAA] * 4

start = time.monotonic()
for _ in range(N):
    spi.xfer2(data)
elapsed = time.monotonic() - start

print(f'{N} transfers in {elapsed:.3f}s')
print(f'Rate: {N/elapsed:.1f} transfers/sec')
print(f'Effective speed: {N*4*8/elapsed/1000:.1f} Kbps')
spi.close()
"
# الـ bitbang عادة بيوصل 50-200 Kbps فقط مقارنة بـ hardware SPI
```

```bash
# فحص هل الـ spi_gpio driver اتحمل صح
# لازم تشوف الرسائل دي في dmesg بعد insmod/probe
dmesg | grep spi_gpio
# الـ output الصحيح:
# [   2.345678] spi_gpio spi_gpio.0: master is unqueued, this is deprecated
# [   2.345700] spi_gpio spi_gpio.0: registered controller spi0

# لو فيه مشاكل هتشوف:
# [   2.345678] spi_gpio spi_gpio.0: probe of spi_gpio.0 failed with error -22
# هنا -22 = -EINVAL = مشكلة في الـ platform_data أو الـ GPIO configuration
```

```bash
# إعادة تحميل الـ driver بدون reboot (للـ module-based build)
modprobe -r spi_gpio spi_bitbang
# تحقق من الـ logs
dmesg | tail -5
# أعد التحميل مع debug
modprobe spi_bitbang
modprobe spi_gpio
dmesg | tail -20

# بعد التحميل، تحقق من الـ state
cat /sys/kernel/debug/gpio | grep spi
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على AM62x — جهاز SPI لا يستجيب أثناء الـ bring-up

#### العنوان
**الـ bitbanged SPI لا يشتغل على AM62x بسبب غياب `num_chipselect` في platform data**

#### السياق
شركة تصنع **industrial gateway** بمعالج **TI AM62x**. المنتج بيحتاج يتواصل مع **ADC خارجي** (ADS8688) عبر SPI. المعالج عنده SPI hardware controllers، بس الـ PCB layout خلّى الـ CS line متوصلة على GPIO عادي بعيد عن الـ SPI controller الأصلي. المهندس قرر يستخدم `spi_gpio` (bitbanged SPI) كحل مؤقت في مرحلة الـ prototype.

#### المشكلة
بعد ما المهندس عمل `insmod spi-gpio.ko`، الـ driver بيتحمّل بدون error، بس أي محاولة للتواصل مع الـ ADC بترجع:
```
spi_gpio spi_gpio.1: no chipselect
spi1.0: Failed to enable chip select
```
الـ ADC مش بيرد خالص.

#### التحليل
الـ `spi_gpio` driver بيعتمد على `struct spi_gpio_platform_data` اللي معرّفة في:

```c
/* include/linux/spi/spi_gpio.h */
struct spi_gpio_platform_data {
    u16  num_chipselect;   /* عدد الـ CS lines المتاحة */
};
```

المهندس عمل الـ `platform_device` كده:

```c
static struct platform_device spi_gpio_device = {
    .name = "spi_gpio",
    .id   = 1,
    /* لم يضع .dev.platform_data */
};
```

الـ `spi_gpio` driver لما بيعمل `probe()`، بيقرأ `num_chipselect` من الـ `platform_data`. لو الـ pointer بـ `NULL` أو `num_chipselect == 0`، الـ controller بيُسجَّل بـ zero chipselects، وبالتالي ما فيش device يقدر يتعرف على نفسه على الـ bus.

#### الحل

```c
/* تعريف صح للـ platform data */
static struct spi_gpio_platform_data ads8688_spi_pdata = {
    .num_chipselect = 1,   /* عندنا device واحد */
};

static struct platform_device spi_gpio_device = {
    .name = "spi_gpio",
    .id   = 1,
    .dev  = {
        .platform_data = &ads8688_spi_pdata,
    },
};
```

لو الـ board بتستخدم Device Tree بدل board file:

```dts
/* am62x-gateway.dts */
spi1: spi-gpio {
    compatible = "spi-gpio";
    #address-cells = <1>;
    #size-cells = <0>;

    sck-gpios  = <&gpio0 10 GPIO_ACTIVE_HIGH>;
    mosi-gpios = <&gpio0 11 GPIO_ACTIVE_HIGH>;
    miso-gpios = <&gpio0 12 GPIO_ACTIVE_HIGH>;
    cs-gpios   = <&gpio0 13 GPIO_ACTIVE_LOW>;
    num-chipselects = <1>;   /* يقابل num_chipselect */

    adc@0 {
        compatible = "ti,ads8688";
        reg = <0>;
        spi-max-frequency = <1000000>;
    };
};
```

#### الدرس المستفاد
**الـ `num_chipselect`** في `spi_gpio_platform_data` مش اختياري — هو بيحدد عدد الـ CS lines اللي الـ controller هيديرها. نسيانه أو تصفيره بيخلي الـ SPI bus يتسجل بدون أي device slot، وبالتالي أي device حاول يتربط بيفشل صامت.

---

### السيناريو 2: Android TV Box على Allwinner H616 — تعدد الـ SPI devices وتعارض الـ CS

#### العنوان
**جهازين SPI على نفس الـ bitbanged bus بيتعارضوا بسبب `num_chipselect` ناقص**

#### السياق
منتج **Android TV Box** بمعالج **Allwinner H616**. الـ board عندها:
- **Flash NOR** (W25Q64) على CS0
- **Touchscreen controller** (XPT2046) على CS1

كلاهم متوصلين على GPIO bitbanged SPI لأن الـ hardware SPI controller محجوز للـ display.

#### المشكلة
الـ NOR flash بيشتغل تمام، بس لما بيحاولوا يقرأوا الـ touchscreen، بيلاقوا إن الـ sysfs بيُظهر device واحد بس:

```bash
$ ls /sys/bus/spi/devices/
spi1.0      # NOR flash فقط
# spi1.1 غايب!
```

والـ touchscreen مش بيظهر في الـ input system خالص.

#### التحليل
```c
/* spi_gpio_platform_data اللي المهندس كتبها */
static struct spi_gpio_platform_data h616_spi_pdata = {
    .num_chipselect = 1,   /* خطأ! عنده جهازين */
};
```

الـ `spi_gpio` driver بيُمرر `num_chipselect` مباشرةً لـ `spi_controller->num_chipselect`. الـ SPI core لما بيحاول يسجّل `spi1.1`، بيشوف إن الـ CS رقم 1 خارج الـ range المسموح (اللي هو 0 بس)، فبيرفض التسجيل.

الـ kernel log بيظهر:

```
spi_gpio spi_gpio.1: cs1 > max 0
spi-nor spi1.1: Failed to register SPI device
```

#### الحل

```c
/* التصحيح: num_chipselect = عدد الـ CS lines الفعلية */
static struct spi_gpio_platform_data h616_spi_pdata = {
    .num_chipselect = 2,   /* CS0 = NOR flash, CS1 = touchscreen */
};
```

في Device Tree:

```dts
spi_bitbang: spi-gpio {
    compatible = "spi-gpio";
    #address-cells = <1>;
    #size-cells = <0>;

    sck-gpios  = <&pio 2 5 GPIO_ACTIVE_HIGH>;
    mosi-gpios = <&pio 2 6 GPIO_ACTIVE_HIGH>;
    miso-gpios = <&pio 2 7 GPIO_ACTIVE_HIGH>;
    cs-gpios   = <&pio 2 8 GPIO_ACTIVE_LOW>,   /* CS0 */
                 <&pio 2 9 GPIO_ACTIVE_LOW>;    /* CS1 */
    num-chipselects = <2>;

    flash@0 {
        compatible = "winbond,w25q64";
        reg = <0>;
        spi-max-frequency = <25000000>;
    };

    touchscreen@1 {
        compatible = "ti,xpt2046";
        reg = <1>;
        spi-max-frequency = <2000000>;
        interrupt-parent = <&pio>;
        interrupts = <2 10 IRQ_TYPE_EDGE_FALLING>;
    };
};
```

#### الدرس المستفاد
**الـ `num_chipselect`** لازم يساوي عدد الـ CS lines الفعلية على الـ bus، مش عدد الـ devices المتوصلة دلوقتي. لو في المستقبل هيُضاف device تالت، لازم تزيد القيمة دي.

---

### السيناريو 3: IoT Sensor Board على STM32MP1 — الانتقال من Bitbanged لـ Native SPI Controller

#### العنوان
**migration من `spi_gpio` لـ native STM32 SPI controller بدون ما الـ kernel يـ crash**

#### السياق
فريق بيطور **IoT sensor node** بمعالج **STM32MP1**. في مرحلة الـ prototype استخدموا `spi_gpio` عشان الـ native SPI pins كانت محجوزة. في الـ production board، الـ routing اتغير والـ native SPI controller (SPI2) أصبح متاح.

#### المشكلة
المهندس حاول يشغّل الـ native SPI driver مع إبقاء الـ `spi_gpio` platform device موجود في الـ board file:

```
[    2.341] spi2: registered master spi2
[    2.342] spi_gpio spi_gpio.2: registered master spi2  ← تعارض رقم الـ bus!
[    2.343] spi2 spi2.0: ERROR: multiple masters on bus 2
```

الـ kernel دخل في undefined behavior وبدأ أي SPI transfer يرجع `-EINVAL`.

#### التحليل
التعليق في `spi_gpio.h` نفسه بيوضح الحل:

```c
/*
 * If the bitbanged bus is later switched to a "native" controller,
 * that platform_device and controller_data should be removed.
 */
```

الـ `.id` في `platform_device` بيحدد رقم الـ SPI bus. لو الـ native controller اتسجّل على `spi2`، والـ `spi_gpio` platform device كمان عنده `.id = 2`، السيستم بيحاول يسجّل bus رقم 2 مرتين.

#### الحل

**الخطوة 1:** إزالة الـ `spi_gpio` platform device من الـ board file نهائياً:

```c
/* board-stm32mp1-iot.c */

/* احذف أو comment out الكود ده */
#if 0
static struct spi_gpio_platform_data sensor_spi_pdata = {
    .num_chipselect = 2,
};
static struct platform_device sensor_spi_gpio = {
    .name = "spi_gpio",
    .id   = 2,
    .dev  = { .platform_data = &sensor_spi_pdata },
};
#endif
```

**الخطوة 2:** تفعيل الـ native SPI في Device Tree:

```dts
/* stm32mp151-iot-prod.dts */
&spi2 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&spi2_pins_a>;
    cs-gpios = <&gpiob 9 GPIO_ACTIVE_LOW>,
               <&gpiob 10 GPIO_ACTIVE_LOW>;

    sensor@0 {
        compatible = "bosch,bme280";
        reg = <0>;
        spi-max-frequency = <10000000>;
    };

    sensor@1 {
        compatible = "adi,adxl345";
        reg = <1>;
        spi-max-frequency = <5000000>;
    };
};
```

**الخطوة 3:** التحقق:

```bash
# تأكد إن bus واحد بس موجود
ls /sys/class/spi_master/
# المفروض يظهر spi2 بس

# قياس الـ clock speed الفعلي (native vs bitbanged)
cat /sys/kernel/debug/spi-2/statistics/transfers
```

#### الدرس المستفاد
الـ `spi_gpio` هو **حل مؤقت** للـ prototype. التعليق في الـ header صريح: لما تنتقل للـ native controller، **ازيل** الـ `platform_device` والـ `controller_data` تماماً. الـ `.id` المتعارض بيسبب مشاكل صعبة تـ debug.

---

### السيناريو 4: Automotive ECU على i.MX8 — تشخيص أداء بطيء في SPI bitbang

#### العنوان
**الـ CAN-FD transceiver بيتأخر في الاستجابة بسبب overhead الـ bitbanged SPI**

#### السياق
ECU في سيارة بمعالج **NXP i.MX8M Plus**. الـ ECU بيتواصل مع **MCP2518FD** (CAN-FD controller) عبر SPI. في الـ proto phase استخدموا `spi_gpio`، لكن في اختبارات الـ real-time، CAN frames بتتأخر بشكل غير مقبول (latency > 500µs).

#### المشكلة
الـ requirement الـ automotive بيشترط latency < 100µs للـ CAN interrupt handling. الـ oscilloscope بيُظهر إن الـ SPI transaction نفسه بياخد 400µs+ مع `spi_gpio`.

#### التحليل
الـ `spi_gpio` driver هو **software bitbanging** — كل bit بيتحكم فيه الـ CPU بشكل مباشر عبر GPIO register reads/writes. الـ overhead جاي من:

1. **GPIO function call overhead** لكل bit (toggle SCLK + sample MISO/set MOSI)
2. **Preemption** — الـ scheduler ممكن يقاطع الـ transfer في النص
3. **Cache misses** لو الـ GPIO registers مش في cache

الـ `spi_gpio_platform_data` نفسه ما بيوفرش أي mechanism للـ DMA أو الـ interrupt-driven operation — الـ design كله synchronous blocking.

للمقارنة، الـ native i.MX8 ECSPI controller بيعمل:
- DMA transfers في الـ background
- Hardware CS management
- الـ CPU مش محتاج يتدخل في كل bit

```bash
# قياس الفرق
time dd if=/dev/spidev0.0 of=/dev/null bs=4 count=1000 iflag=fullblock
# مع spi_gpio:   real 0m4.200s
# مع native SPI: real 0m0.041s
```

#### الحل

**الانتقال للـ native ECSPI3 على i.MX8:**

```dts
/* imx8mp-ecu.dts */
&ecspi3 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_ecspi3>;
    fsl,spi-num-chipselects = <1>;
    cs-gpios = <&gpio5 25 GPIO_ACTIVE_LOW>;

    can_fd: can@0 {
        compatible = "microchip,mcp2518fd";
        reg = <0>;
        clocks = <&clk_can 0>;
        spi-max-frequency = <20000000>;   /* 20 MHz مدعومة من native */
        interrupt-parent = <&gpio5>;
        interrupts = <26 IRQ_TYPE_EDGE_FALLING>;
    };
};
```

```c
/* تأكيد الـ pinmux */
MX8MP_PAD_ECSPI3_SCLK__ECSPI3_SCLK  0x82   /* drive strength high */
MX8MP_PAD_ECSPI3_MOSI__ECSPI3_MOSI  0x82
MX8MP_PAD_ECSPI3_MISO__ECSPI3_MISO  0x82
```

#### الدرس المستفاد
**الـ `spi_gpio` مش مناسب للـ real-time applications.** الـ struct `spi_gpio_platform_data` ما بتوفرش أي control على timing أو priority. في الـ automotive context، استخدم دايماً الـ hardware SPI controller مع DMA، واحتفظ بـ `spi_gpio` للـ debug والـ prototype بس.

---

### السيناريو 5: Custom Board Bring-up على RK3562 — تشخيص جهاز SPI جديد بـ bitbanging أولاً

#### العنوان
**استخدام `spi_gpio` كـ diagnostic tool لتأكيد wiring قبل تفعيل الـ native SPI controller**

#### السياق
مهندس بيعمل **bring-up لـ custom RK3562 board**. الـ board عندها **SPI NAND flash** (GD5F1GQ4U) متوصل على SPI1. الـ pinmux configuration للـ native SPI مش متأكد منها — احتمال في خطأ في الـ DTS أو في الـ PCB. بدل ما يضيع وقت في debug الـ native driver، قرر يـ verify الـ wiring أولاً بـ `spi_gpio`.

#### المشكلة
الـ native SPI driver (rockchip-spi) بيـ probe بنجاح، بس أي flash operation بيفشل:

```
[    3.102] rockchip-spi ff620000.spi: 0 bytes xfered
[    3.103] spi-nand: JEDEC id bytes: 00 00 00
```

كل الـ ID bytes صفر — يعني مش واصل signal صح.

#### التحليل والخطوات

**الخطوة 1: تفعيل `spi_gpio` كـ bypass للـ native controller:**

```dts
/* rk3562-custom-debug.dts */

/* تعطيل native SPI1 مؤقتاً */
&spi1 {
    status = "disabled";
};

/* تفعيل bitbanged SPI على نفس الـ pins */
spi_debug: spi-gpio {
    compatible = "spi-gpio";
    #address-cells = <1>;
    #size-cells = <0>;

    /* نفس الـ GPIO numbers اللي المفروض يستخدمها SPI1 */
    sck-gpios  = <&gpio1 RK_PB3 GPIO_ACTIVE_HIGH>;
    mosi-gpios = <&gpio1 RK_PB4 GPIO_ACTIVE_HIGH>;
    miso-gpios = <&gpio1 RK_PB5 GPIO_ACTIVE_HIGH>;
    cs-gpios   = <&gpio1 RK_PB6 GPIO_ACTIVE_LOW>;
    num-chipselects = <1>;

    flash@0 {
        compatible = "spi-nand";
        reg = <0>;
        spi-max-frequency = <1000000>;   /* سرعة منخفضة للـ debug */
    };
};
```

**الخطوة 2:** الـ `spi_gpio_platform_data` هنا بتيجي من الـ DT parsing عبر `num-chipselects`:

```c
/* في drivers/spi/spi-gpio.c */
pdata->num_chipselect = of_gpio_named_count(np, "cs-gpios");
/* بيملأ num_chipselect تلقائياً من الـ DT */
```

**الخطوة 3: تشغيل الـ test:**

```bash
# تأكيد إن الـ driver اتحمّل
dmesg | grep spi-gpio
# [    3.200] spi-gpio spi-gpio: registered master spi0

# قراءة JEDEC ID من الـ NAND flash
cat /sys/bus/spi/devices/spi0.0/jedec_id
# لو ظهر: c8 b1 48 → الـ wiring صح!
# لو ظهر: 00 00 00 → مشكلة hardware
```

**الخطوة 4:** لو `spi_gpio` شغّال، المشكلة في الـ pinmux للـ native controller:

```bash
# فحص الـ pinmux state
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip/pinmux-pins | grep gpio1
# لو الـ pins لسه على GPIO mode مش SPI mode → مشكلة DTS
```

**التصحيح في الـ DTS:**

```dts
/* الخطأ الشائع: missing pinctrl */
&spi1 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&spi1m0_pins>;   /* ← كان ناقص */

    flash@0 {
        compatible = "spi-nand";
        reg = <0>;
        spi-max-frequency = <80000000>;
    };
};
```

#### الدرس المستفاد
**الـ `spi_gpio` أداة تشخيص قوية** في الـ bring-up. لما الـ native SPI controller مش بيشتغل، استبدله مؤقتاً بـ `spi_gpio` على نفس الـ GPIO lines. لو الـ bitbanging شغّال والـ native مش شغّال، المشكلة بالتأكيد في الـ pinmux أو الـ clock configuration وليس في الـ hardware wiring. الـ `num_chipselect` في هذه الحالة بيتملى تلقائياً من عدد الـ `cs-gpios` في الـ DT.
## Phase 7: مصادر ومراجع

### مصادر رسمية من kernel.org

| المصدر | الرابط | الأهمية |
|--------|--------|---------|
| **SPI Summary** — نظرة عامة على subsystem كاملة | [docs.kernel.org/spi/spi-summary.html](https://docs.kernel.org/spi/spi-summary.html) | أساسي |
| **SPI Driver API** — توثيق الـ API الرسمي | [docs.kernel.org/driver-api/spi.html](https://docs.kernel.org/driver-api/spi.html) | أساسي |
| **SPI userspace API** — الـ spidev interface | [docs.kernel.org/spi/spidev.html](https://docs.kernel.org/spi/spidev.html) | مهم |
| **GPIO drivers-on-gpio** — subsystem drivers using GPIO | [static.lwn.net/kerneldoc/driver-api/gpio/drivers-on-gpio.html](https://static.lwn.net/kerneldoc/driver-api/gpio/drivers-on-gpio.html) | مهم |
| **Device Tree binding: spi-gpio.txt** | [kernel.org/doc/Documentation/devicetree/bindings/spi/spi-gpio.txt](https://www.kernel.org/doc/Documentation/devicetree/bindings/spi/spi-gpio.txt) | أساسي |
| **Device Tree binding: spi-gpio.yaml** (الإصدار الحديث) | [github.com/torvalds/linux — spi-gpio.yaml](https://github.com/torvalds/linux/blob/master/Documentation/devicetree/bindings/spi/spi-gpio.yaml) | أساسي |
| **GPIO Legacy Interface** | [static.lwn.net/kerneldoc/driver-api/gpio/legacy.html](https://static.lwn.net/kerneldoc/driver-api/gpio/legacy.html) | تاريخي |

---

### مقالات LWN.net

الـ **LWN.net** هو المرجع الأهم لمتابعة تطور kernel — دي أبرز المقالات المتعلقة بـ `spi_gpio`:

#### مقالات مباشرة عن SPI + GPIO

- **[Add SPI over GPIO driver](https://lwn.net/Articles/290068/)**
  المقال الأصلي اللي ناقش إضافة driver الـ SPI bitbang عبر GPIO للـ kernel. يشرح آلية الـ bitbanging وكيف اتعمل الـ `spi_gpio` كـ platform device.

- **[Add dynamic MMC-over-SPI-GPIO driver](https://lwn.net/Articles/290066/)**
  نقاش عن driver بيوفر واجهة sysfs لإنشاء وحذف GPIO-based MMC/SD interfaces — مثال عملي على استخدام `spi_gpio` كـ transport layer.

- **[SPI core](https://lwn.net/Articles/138037/)**
  مقال قديم بيوضح الأساس التاريخي لـ SPI subsystem في kernel وكيف اتبنى.

#### مقالات GPIO ذات صلة

- **[GPIO implementation framework](https://lwn.net/Articles/256461/)**
  شرح الـ framework اللي بيعتمد عليه `spi_gpio` في الوصول للـ pins.

- **[Bitbanging i2c bus driver using the GPIO API](https://lwn.net/Articles/230571/)**
  نفس المفهوم لكن لـ I2C — مفيد لفهم pattern الـ bitbang بشكل عام في kernel.

- **[Introduction to SPI NAND framework](https://lwn.net/Articles/719733/)**
  SPI NOR/NAND كـ use case متقدم فوق الـ SPI subsystem.

---

### كود المصدر في kernel

| الملف | الوصف |
|-------|-------|
| `include/linux/spi/spi_gpio.h` | الـ header الخاص بنا — يعرّف `spi_gpio_platform_data` |
| `drivers/spi/spi-gpio.c` | الـ driver الكامل — [github](https://github.com/torvalds/linux/blob/master/drivers/spi/spi-gpio.c) |
| `drivers/spi/spi-bitbang.c` | الـ bitbang core اللي بيعتمد عليه `spi-gpio.c` |
| `include/linux/spi/spi-bitbang.h` | الـ header للـ bitbang core |
| `Documentation/spi/` | مجلد التوثيق الكامل للـ SPI subsystem |

---

### Kernel Commits المهمة

- **David Brownell** هو كاتب الـ driver الأصلي (copyright 2006, 2008) — ابحث عن commits بـ:
  ```bash
  git log --all --author="David Brownell" -- drivers/spi/spi-gpio.c
  ```
- **Linus Walleij** عمل refactoring كبير سنة 2017 لاستخدام descriptor-based GPIO API بدل legacy API:
  ```bash
  git log --all --author="Linus Walleij" -- drivers/spi/spi-gpio.c
  ```
- إضافة YAML binding لـ Device Tree (إزالة `.txt` القديم):
  ```bash
  git log -- Documentation/devicetree/bindings/spi/spi-gpio.yaml
  ```

---

### Mailing List — نقاشات مهمة

- **[PATCH: spi: Driver for GPIO controlled SPI multiplexer](https://linux-spi.vger.kernel.narkive.com/cnNyJmfw/patch-spi-driver-for-gpio-controlled-spi-multiplexer)**
  نقاش عن driver بيستخدم GPIO للتحكم في SPI multiplexer — يبين كيف بتتداخل GPIO و SPI في الـ design.

- **[LKML Archive](https://www.spinics.net/lists/kernel/)**
  الأرشيف الكامل للـ Linux Kernel Mailing List — ابحث عن `spi_gpio` أو `spi-gpio` للاطلاع على نقاشات الـ patches التاريخية.

- للبحث المباشر في الـ mailing list:
  ```
  https://lore.kernel.org/linux-spi/?q=spi_gpio
  ```

---

### kernelnewbies.org

| الصفحة | الفائدة |
|--------|---------|
| **[Linux_2_6_24](https://kernelnewbies.org/Linux_2_6_24)** | أول إصدار دخلت فيه MMC-over-SPI تجريبياً — تاريخ إدخال SPI bitbang |
| **[Linux_3.15-DriversArch](https://kernelnewbies.org/Linux_3.15-DriversArch)** | تغييرات drivers/spi في 3.15 |
| **[Linux_4.18](https://kernelnewbies.org/Linux_4.18)** | تغييرات SPI في 4.18 |
| **[Linux_6.5](https://kernelnewbies.org/Linux_6.5)** | إضافات SPI controllers حديثة |

---

### elinux.org

| الصفحة | الفائدة |
|--------|---------|
| **[RPi SPI](https://elinux.org/RPi_SPI)** | شرح عملي لـ SPI على Raspberry Pi — مثال حقيقي للـ hardware SPI و GPIO bitbang |
| **[Tests:MSIOF-SPI-CS-GPIO](https://elinux.org/Tests:MSIOF-SPI-CS-GPIO)** | اختبارات عملية لـ SPI chip select عبر GPIO على Renesas board |
| **[SpiSlaveZero](https://elinux.org/SpiSlaveZero)** | مواصفة SPI peripheral قابلة للتنفيذ بـ bitbang |

---

### كتب مرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
الكتاب مجاني مفتوح المصدر — [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

- **Chapter 14: The Linux Device Model** — فهم الـ platform_device وكيف يرتبط بـ `spi_gpio_platform_data`
- **Chapter 6: Advanced Char Driver Operations** — الـ I/O model اللي بيبني عليه SPI

#### Linux Kernel Development — Robert Love (3rd Edition)
- **Chapter 17: Devices and Modules** — شرح driver model وكيف بتتسجل الـ drivers
- **Chapter 7: Interrupts and Interrupt Handlers** — مهم لفهم الـ async SPI transfers

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **Chapter 15: Devices and Drivers** — SPI و I2C في بيئة embedded
- **Chapter 16: Kernel Debugging Techniques** — debugging للـ SPI bitbang

#### Essential Linux Device Drivers — Sreekrishnan Venkateswaran
- **Chapter 8: I2C and SPI** — فصل كامل مخصص للـ SPI subsystem مع أمثلة كود

---

### مصادر إضافية

#### توثيق رسمي آخر

```
Documentation/spi/spi-summary.rst     -- نظرة عامة
Documentation/spi/spi-ldisc.rst       -- line discipline للـ SPI
Documentation/spi/pxa2xx.rst          -- مثال controller محدد
Documentation/devicetree/bindings/spi/ -- كل الـ DT bindings للـ SPI
```

#### أدوات Debugging

```bash
# فحص الـ SPI devices المسجلة في النظام
ls /sys/bus/spi/devices/

# معلومات عن SPI controllers
ls /sys/class/spi_master/

# kernel messages للـ SPI
dmesg | grep -i spi

# فحص GPIO المستخدمة
cat /sys/kernel/debug/gpio
```

---

### مصطلحات البحث

لإيجاد معلومات أكثر، استخدم الكلمات دي في البحث:

```
spi_gpio platform_data Linux kernel
spi-gpio device tree binding
Linux SPI bitbang GPIO driver
spi_bitbang_start kernel
spi_master_alloc GPIO Linux
linux/spi/spi_gpio.h num_chipselect
spi_gpio_platform_data embedded Linux
David Brownell SPI GPIO kernel patch
```

#### للبحث في lore.kernel.org (الأفضل للـ patches):
```
https://lore.kernel.org/linux-spi/
https://lore.kernel.org/linux-gpio/
```
## Phase 8: Writing simple module

### الفكرة

**الـ** `spi_gpio` driver بيستخدم الـ SPI subsystem العادي، وأي device بيتعمله setup بيمر على `spi_setup()` — دالة exported موجودة في `drivers/spi/spi.c`. هنعمل kprobe على `spi_setup` عشان نطبع معلومات الـ SPI device (اسمه، bus number، speed، mode، bits_per_word) في كل مرة بيتم فيها configure أي device على أي controller — بما فيه الـ bitbang controller اللي بيستخدم `spi_gpio`.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * spi_setup_probe.c
 *
 * kprobe on spi_setup() — prints SPI device info on every setup call.
 * Useful for observing spi_gpio (bitbang) and any other SPI controller.
 */

#include <linux/module.h>      /* MODULE_*, module_init/exit              */
#include <linux/kprobes.h>     /* kprobe API                              */
#include <linux/spi/spi.h>     /* struct spi_device, spi_setup prototype  */
#include <linux/kern_levels.h> /* KERN_INFO for pr_info                   */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Doc Project");
MODULE_DESCRIPTION("kprobe on spi_setup() to trace SPI device configuration");

/* ------------------------------------------------------------------ */
/* pre-handler: runs just before spi_setup() executes                  */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * spi_setup(struct spi_device *spi)
     * On x86-64 the first argument lives in RDI; on ARM64 in X0.
     * kprobes gives us pt_regs — we read the first argument with
     * the architecture-portable helper regs_get_kernel_argument().
     * Cast it back to the pointer type we know.
     */
    struct spi_device *spi = (struct spi_device *)
                             regs_get_kernel_argument(regs, 0);

    if (!spi)
        return 0;   /* نادر، بس نحمي نفسنا من NULL */

    pr_info("[spi_setup_probe] device=%-16s  bus=%d  "
            "max_speed=%u Hz  bits_per_word=%u  mode=0x%04x\n",
            dev_name(&spi->dev),          /* اسم الـ device في sysfs         */
            spi->controller->bus_num,     /* رقم الـ SPI bus                 */
            spi->max_speed_hz,            /* أقصى سرعة clock مسموح بيها     */
            spi->bits_per_word,           /* حجم الـ word (0 = default 8)    */
            spi->mode);                   /* SPI_MODE_0..3 + flags تانية     */

    return 0; /* صفر = الكنرنل يكمل تنفيذ spi_setup() بشكل طبيعي */
}

/* ------------------------------------------------------------------ */
/* kprobe struct                                                        */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "spi_setup",   /* اسم الدالة المستهدفة في kernel symbol table */
    .pre_handler = handler_pre,   /* callback يتشغل قبل الدالة                  */
};

/* ------------------------------------------------------------------ */
/* module_init                                                          */
/* ------------------------------------------------------------------ */
static int __init spi_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[spi_setup_probe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[spi_setup_probe] planted kprobe at %s (%px)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit                                                          */
/* ------------------------------------------------------------------ */
static void __exit spi_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("[spi_setup_probe] kprobe removed from %s\n", kp.symbol_name);
}

module_init(spi_probe_init);
module_exit(spi_probe_exit);
```

---

### شرح كل جزء

#### `#include <linux/kprobes.h>`
بتجيب الـ API الخاص بالـ kprobes: `struct kprobe`، `register_kprobe()`، `unregister_kprobe()`، و`regs_get_kernel_argument()`. من غيره ما تقدر تزرع probe في أي دالة.

#### `#include <linux/spi/spi.h>`
بيعرّف `struct spi_device` وكل الحقول اللي بنقرأها (`bus_num`، `max_speed_hz`، `mode`، `bits_per_word`). بدونه المؤلف مش هيعرف يعمل cast صح للـ pointer اللي جاي من `pt_regs`.

---

#### الـ `handler_pre` — الـ callback

```c
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
```

**الـ** `pt_regs` بيحتوي على حالة الـ registers لحظة الـ probe hit. بنستخدم `regs_get_kernel_argument(regs, 0)` عشان نجيب أول argument للدالة — هو `struct spi_device *spi` — بطريقة portable على x86-64 وARM64 من غير ما نحتاج نكتب assembly.

الـ `pr_info` بيطبع:
| الحقل | معناه |
|---|---|
| `dev_name(&spi->dev)` | اسم الـ device في `/sys/bus/spi/devices/` |
| `spi->controller->bus_num` | رقم الـ SPI controller (مثلاً `spi0`) |
| `spi->max_speed_hz` | أقصى clock speed مضبوط للـ device |
| `spi->bits_per_word` | حجم الـ word بالـ bits (0 = default 8-bit) |
| `spi->mode` | الـ SPI mode (CPOL/CPHA) والـ flags التانية |

الـ return value صفر عشان يخلي الكرنل يكمل تنفيذ `spi_setup()` بشكل طبيعي بدون أي تدخل.

---

#### `struct kprobe kp`

```c
static struct kprobe kp = {
    .symbol_name = "spi_setup",
    .pre_handler = handler_pre,
};
```

**الـ** `symbol_name` بيخلي الكرنل يحول الاسم لعنوان تلقائياً عن طريق `kallsyms`. لو حطّينا `.addr` مباشرة كنا محتاجين نعرف العنوان الحرفي اللي بيتغير مع كل build.

---

#### `module_init` — التسجيل

`register_kprobe(&kp)` بتزرع breakpoint في الذاكرة عند بداية `spi_setup()`. لو فشلت (مثلاً لو `CONFIG_KPROBES=n` أو الدالة `__kprobes` مش موجودة) بترجع قيمة سالبة ونطبع error ونوقف الـ init.

---

#### `module_exit` — إلغاء التسجيل

`unregister_kprobe(&kp)` ضروري جداً في الـ exit عشان يشيل الـ breakpoint من الذاكرة ويحرر الـ resources. لو ما عملناه ورميّنا الـ module، الكرنل ممكن يـcrash لما يوصل لعنوان الـ probe ويلاقيش handler.

---

### Makefile

```makefile
obj-m += spi_setup_probe.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

---

### تجربة على الجهاز

```bash
# بناء الـ module
make

# تحميله
sudo insmod spi_setup_probe.ko

# تفعيل أي SPI device أو تحميل spi_gpio driver
# ثم مشاهدة الـ log
sudo dmesg | grep spi_setup_probe

# مثال على output
# [spi_setup_probe] planted kprobe at spi_setup (ffffffffc0a12340)
# [spi_setup_probe] device=spi0.0            bus=0  max_speed=1000000 Hz  bits_per_word=8  mode=0x0000
# [spi_setup_probe] device=spi1.0            bus=1  max_speed=500000  Hz  bits_per_word=8  mode=0x0003

# إزالته
sudo rmmod spi_setup_probe
```

---

### ليه `spi_setup` تحديداً؟

**الـ** `spi_gpio` driver لما يـregister أي target device على الـ bitbang bus، الـ SPI core بيستدعي `spi_setup()` عشان يأكد إن الـ mode والـ clock مدعومين. ده بيخلي `spi_setup` نقطة مركزية تمر منها كل SPI devices بغض النظر عن نوع الـ controller — سواء hardware native أو bitbang via GPIO.
