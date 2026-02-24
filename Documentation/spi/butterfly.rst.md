## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ Subsystem اللي ينتمي له الملف ده؟

الملف `Documentation/spi/butterfly.rst` جزء من **SPI Subsystem** في Linux kernel، والـ maintainer بتاعه هو Mark Brown. الـ subsystem ده بيغطي كل حاجة من `drivers/spi/` و`include/linux/spi/` و`Documentation/spi/`.

---

### الفكرة الكبيرة — بالمثال الحقيقي

تخيل إنك عندك كمبيوتر قديم فيه **parallel port** (المنفذ اللي كان بيتوصل بيه الطابعة زمان — الـ DB-25 اللي فيه 25 سلك). وعندك كمان كارت صغير اسمه **AVR Butterfly** — كارت تجريبي بـ 20 دولار من Atmel، عليه microcontroller وشاشة LCD وسنسورات وفلاشة وزرار toggle.

**المشكلة:** إزاي أخلي Linux يتكلم مع الـ Butterfly ده؟ ما فيش USB، ما فيش Ethernet. الوسيلة الوحيدة المتاحة هي الـ parallel port القديم.

**الحل اللي بيشرحه الملف ده:** تعمل **كابل محلي الصنع** بتوصّل أسلاك الـ parallel port بـ pins الـ Butterfly، وبكده تحول الـ parallel port إلى **SPI bus** بالكامل عن طريق **bit-banging** — يعني Linux بيتحكم يدوياً في كل سلك SCK و MOSI و MISO عن طريق كتابة bits في registers الـ parport، فبيحاكي بروتوكول SPI من غير أي hardware SPI حقيقي.

---

### القصة الكاملة — خطوة بخطوة

#### المشهد الأول: ليه أصلاً؟

**الـ SPI** (Serial Peripheral Interface) بروتوكول تسلسلي بسيط جداً: 4 أسلاك فقط — SCK (clock)، MOSI (بيانات من master للـ slave)، MISO (بيانات من slave للـ master)، و CS/SS (chip select). بيُستخدم مع Flash memory وسنسورات وشاشات وكتير.

الـ **parallel port** عنده pins رقمية ممكن تكتب وتقرأ فيها بسهولة من Linux. فكرة **bit-banging** هي إنك "تلعب" بالـ pins دي يدوياً بالسوفتوير لمحاكاة أي بروتوكول — في الحالة دي SPI.

#### المشهد التاني: الكابل

الملف بيشرح إزاي توصّل الكابل بالضبط. الوصلة الأولى بتربط Linux بـ SPI bus فيه الـ AVR نفسه وشريحة **DataFlash** (AT45DB041B — فلاشة 512 كيلوبايت موجودة على الـ Butterfly):

```
Signal    Butterfly Pin    Parport (DB-25)
SCK       J403.PB1/SCK     pin 2 / D0
RESET     J403.nRST        pin 3 / D1
VCC       J403.VCC_EXT     pin 8 / D6
MOSI      J403.PB2/MOSI    pin 9 / D7
MISO      J403.PB3/MISO    pin 11 / S7,nBUSY
GND       J403.GND         pin 23 / GND
```

ملحوظة: الـ parallel port **مش بيوفر power حقيقية**، فبيتم سرقة الـ VCC من أكتر من pin لتأمين تيار كافي للدارة.

#### المشهد التالت: إزاي يتكلم Linux مع الـ DataFlash؟

عشان Linux يـ master الـ bus ويكلم شريحة الـ DataFlash مباشرة، لازم:
1. **تفلاش firmware جديد** على الـ AVR يعطّل الـ SPI controller الخاص بيه (مش عايزه يتنافس مع Linux).
2. **توصّل chip select** للـ DataFlash عبر pin إضافي من الـ parport.
3. **تفعّل driver** الـ `mtd_dataflash` في Linux عشان يشوف الفلاشة كـ MTD device.

#### البديل الإبداعي: AVR كـ SPI Slave

ممكن كمان تفلاش firmware يخلي الـ AVR نفسه يشتغل كـ **SPI slave** — يعني Linux يتكلم معاه مباشرة عبر الـ SPI protocol ده، وتكتب driver خاص بيك للـ protocol اللي أنت صممته.

#### المشهد الرابع: الـ USI Bus الثاني

الـ Butterfly عنده controller تاني اسمه **USI** (Universal Serial Interface) على J405. ده بيسمح بـ SPI bus تاني — فتقدر تبقى عندك في نفس الوقت:
- Bus 1: Linux يتكلم مع الـ DataFlash
- Bus 2: Linux يتكلم مع الـ AVR بـ firmware خاص

---

### ليه الموضوع ده مهم؟

**الـ spi_butterfly** هو مثال حي على:
- **SPI bitbang framework** في Linux — يعني أي hardware عنده GPIO pins يقدر يعمل SPI من غير hardware SPI controller.
- **Prototyping tool** — تقدر تطور وتختبر SPI drivers على Linux حقيقي من غير ما تحتاج hardware متخصص.
- **Bridge** بين Linux kernel SPI stack وأي microcontroller — لما تخلصي من الـ prototype، نفس الـ driver بيشتغل مع أي SPI controller حقيقي بدون تعديل.

---

### الـ Files المهمة اللي المفروض تعرفها

| الملف | الدور |
|---|---|
| `drivers/spi/spi-butterfly.c` | الـ driver الفعلي — بيحول الـ parport لـ SPI bus |
| `drivers/spi/spi-bitbang.c` | الـ framework اللي بيوفر الـ bit-bang engine |
| `include/linux/spi/spi_bitbang.h` | الـ struct وAPI الخاصة بالـ bitbang layer |
| `include/linux/spi/spi.h` | الـ SPI core API كلها |
| `drivers/spi/spi-bitbang-txrx.h` | الـ inline functions لإرسال/استقبال bits يدوياً |
| `drivers/parport/` | الـ parport subsystem اللي بيعرض abstraction للـ parallel port |
| `include/linux/parport.h` | API الـ parport — الـ `parport_write_data()` وغيرها |
| `drivers/mtd/devices/mtd_dataflash.c` | الـ driver الخاص بشريحة الـ AT45 DataFlash |
| `Documentation/spi/spi-summary.rst` | شرح SPI framework كامل في Linux |
| `Documentation/spi/spi-lm70llp.rst` | مثال مشابه — parport-to-LM70 temperature sensor |

---

### الـ Subsystem Files بالتفصيل

#### Core:
- `drivers/spi/spi.c` — الـ SPI core
- `drivers/spi/spi-bitbang.c` — الـ bit-bang engine

#### Headers:
- `include/linux/spi/spi.h`
- `include/linux/spi/spi_bitbang.h`
- `include/linux/spi/flash.h`

#### Hardware Drivers (أمثلة):
- `drivers/spi/spi-butterfly.c` — الـ parport-to-Butterfly adapter (موضوعنا)
- `drivers/spi/spi-gpio.c` — SPI عبر GPIO عام
- `drivers/spi/spi-pl022.c` — ARM PrimeCell SPI controller

#### Related Subsystems:
- `drivers/parport/` — الـ parallel port layer
- `drivers/mtd/devices/` — الـ MTD (Memory Technology Devices) للـ DataFlash
## Phase 2: شرح الـ SPI Framework — من الـ Parport للـ Kernel Bus

---

### المشكلة اللي الـ SPI Subsystem بيحلها

تخيّل عندك embedded board فيها:
- Flash chip بتشتغل بـ SPI.
- Temperature sensor بيشتغل بـ SPI.
- Touchscreen controller بيشتغل بـ SPI.
- وكل دول متوصلين بـ SPI controller واحد جوه الـ SoC.

بدون abstraction layer، كل driver هيحتاج يعرف:
1. إزاي يتحكم في الـ clock.
2. إزاي يعمل chip select.
3. إزاي ينتظر التحويل ينتهي.
4. إزاي يتعامل مع الـ DMA.

ده معناه **code duplication** هائل وعدم portability تام. لو غيّرت الـ SoC من i.MX6 لـ BCM2835، كل الـ protocol drivers هتبقى محتاج تعيد كتابتها.

**الـ SPI subsystem** بيحل المشكلة دي بفصل:
- **من يعرف الهاردوير** (Controller Driver) — واحد لكل SoC.
- **من يعرف البروتوكول** (Protocol Driver) — واحد لكل chip نوع.
- **ومن يربط الاتنين** (SPI Core) — shared infrastructure للكل.

---

### الفكرة الجوهرية: Layered Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                     Userspace / Kernel Consumers               │
│         MTD Layer, hwmon, input, net, custom drivers           │
└──────────────────────────┬─────────────────────────────────────┘
                           │ spi_sync() / spi_async()
┌──────────────────────────▼─────────────────────────────────────┐
│                     Protocol Drivers                           │
│   spi-nor (flash), lm70 (temp), ads7846 (touch), mmc_spi ...  │
│   يتعاملوا مع struct spi_device فقط، مش مع الهاردوير          │
└──────────────────────────┬─────────────────────────────────────┘
                           │ struct spi_message / spi_transfer
┌──────────────────────────▼─────────────────────────────────────┐
│                       SPI Core                                 │
│              drivers/spi/spi.c + include/linux/spi/spi.h       │
│    - Queue management                                          │
│    - Message scheduling                                        │
│    - sysfs integration                                         │
│    - Device/driver binding                                     │
└──────┬───────────────────────────────────────┬─────────────────┘
       │ transfer_one() / set_cs()             │
┌──────▼────────────┐               ┌──────────▼─────────────────┐
│  HW Controllers   │               │   Bitbang Controllers      │
│ spi-imx, spi-dw   │               │  spi-gpio, spi_butterfly,  │
│ spi-bcm2835, ...  │               │  spi_lm70llp, ...          │
└──────┬────────────┘               └──────────┬─────────────────┘
       │                                       │
┌──────▼────────────┐               ┌──────────▼─────────────────┐
│  SoC SPI Hardware │               │  Parallel Port / GPIO      │
│  (Registers, DMA) │               │  (Software bit-banging)    │
└───────────────────┘               └────────────────────────────┘
```

**الـ spi_butterfly** بيقع في الجانب الأيمن تماماً: هو **bitbang controller driver** بيحوّل الـ parallel port (parport) إلى SPI bus.

---

### التشبيه الحقيقي: شركة شحن دولية

تخيّل إنك بتبعت طرود (data) لعملاء (SPI devices):

| المفهوم الفعلي | التشبيه |
|---|---|
| **SPI Protocol Driver** | العميل اللي كتب الطرد وحدد محتواه |
| **struct spi_message** | الطرد كامل مع غلافه والعنوان |
| **struct spi_transfer** | كل قطعة جوه الطرد (بعضها fragile) |
| **SPI Core** | شركة DHL — تستقبل الطرد وتراتب التسليم |
| **Controller Driver** | السائق اللي يعرف الطريق (الهاردوير الفعلي) |
| **struct spi_device** | عنوان التسليم + معلومات المستلم |
| **Chip Select** | جرس باب المستلم تحديداً |
| **SCK Clock** | سرعة السيارة — مش المستلم يتحكم فيها، السائق |
| **Bitbanger (spi_butterfly)** | سائق بيجيب عربية قديمة (parport) بدل الـ truck الحديث |

**الفارق المهم في التشبيه:** شركة DHL (SPI Core) ما بتعرفش ولا مهتمة هل السائق بيستخدم Mercedes أو تروسيكل. اللي بيهمها إن الطرد وصل. ده بالظبط اللي بيحصل: **protocol driver ما بيعرفش حاجة عن الهاردوير**.

---

### الـ spi_butterfly: ليه موجود؟

**المشكلة العملية:** في بداية تطوير firmware لـ AVR Butterfly board، محتاج:
1. تعمل flash للـ firmware الجديد على الـ AVR.
2. تتواصل مع DataFlash chip متصلة بنفس الـ SPI bus.
3. وأنت محتاج تعمل ده من Linux PC عادي **بدون** SPI controller hardware.

**الحل:** كابل بسيط من الـ parallel port إلى الـ AVR Butterfly، والـ spi_butterfly driver بيحوّل الـ parport إلى SPI controller كامل تقدر تستخدمه مع أي SPI protocol driver عادي.

```
Linux PC                              AVR Butterfly Board
─────────                             ─────────────────────
DB-25 Parport                         J403 Connector
─────────────                         ─────────────
Pin 2  (D0)  ──────── SCK ─────────► PB1 (SCK)
Pin 9  (D7)  ──────── MOSI ────────► PB2 (MOSI)
Pin 11 (S7)  ◄─────── MISO ──────── PB3 (MISO)
Pin 3  (D1)  ──────── RESET ───────► nRST
Pin 8  (D6)  ──────── VCC  ─────────► VCC_EXT
Pin 23 (GND) ──────── GND  ──────────► GND

                                      J400 Connector
Pin 17 (C3)  ──────── CS ──────────► PB0/nSS  (DataFlash)
```

الـ driver بيعمل **software bitbanging**: بدل ما الهاردوير يولّد الـ clock والـ data automatically، الـ CPU نفسه بيكتب bits في registers الـ parport واحدة واحدة. بطيء؟ أيوه. بس شغّال.

---

### الـ Core Abstractions: الأربع Structs الأساسية

#### 1. `struct spi_controller` — تمثيل الـ Bus Master

```c
struct spi_controller {
    struct device   dev;
    u16             bus_num;        /* رقم الـ SPI bus (0, 1, 2, ...) */
    u16             num_chipselect; /* عدد الـ CS lines */

    /* ─── Hooks لازم controller driver ينفّذها ─── */
    int  (*setup)(struct spi_device *spi);
    int  (*transfer_one)(struct spi_controller *ctlr,
                         struct spi_device *spi,
                         struct spi_transfer *transfer);
    void (*set_cs)(struct spi_device *spi, bool enable);

    /* ─── Queue management (SPI Core بيملّيها) ─── */
    struct list_head    queue;
    struct kthread_worker *kworker;
};
```

الـ `spi_butterfly` driver بيخلق instance من ده ويملّي الـ hooks بـ parport bitbang functions.

#### 2. `struct spi_device` — تمثيل الجهاز على الـ Bus

```c
struct spi_device {
    struct device       dev;
    struct spi_controller *controller;
    u32             max_speed_hz;   /* أقصى سرعة clock للجهاز ده */
    u8              chip_select;    /* CS line رقم كام؟ */
    u8              bits_per_word;  /* 8, 16, 32 ... */
    u16             mode;           /* SPI_MODE_0..3 + flags */
    char            modalias[SPI_NAME_SIZE]; /* اسم الـ driver */
};
```

الـ DataFlash chip على الـ butterfly board بيُمثَّل بـ `spi_device` واحد متصل بالـ `spi_controller` الخاص بالـ parport.

#### 3. `struct spi_transfer` — وحدة النقل الواحدة

```c
struct spi_transfer {
    const void  *tx_buf;    /* بيانات هترسل (أو NULL لو read فقط) */
    void        *rx_buf;    /* buffer استقبال  (أو NULL لو write فقط) */
    unsigned    len;        /* عدد البايتات */

    u32         speed_hz;   /* override السرعة لهذا الـ transfer تحديداً */
    u8          bits_per_word;
    u16         delay;      /* microseconds delay بعد الـ transfer */
    bool        cs_change;  /* هل تغيّر CS بعد الـ transfer ده؟ */

    struct list_head transfer_list; /* ربط في الـ spi_message */
};
```

#### 4. `struct spi_message` — الـ Atomic Transaction

```c
struct spi_message {
    struct list_head    transfers;  /* linked list من spi_transfer */
    struct spi_device   *spi;
    bool                is_dma_mapped;

    /* completion callback */
    void (*complete)(void *context);
    void *context;
    int  status;    /* 0 = success */

    struct list_head    queue;      /* في queue الـ controller */
};
```

**المهم:** طول ما الـ `spi_message` شغّالة، الـ CS line بتفضل active ومفيش أي message تانية تقدر تأخذ الـ bus. ده الـ **atomicity** guarantee.

---

### العلاقة بين الـ Structs

```
spi_controller
│
├── bus_num = 0
├── transfer_one() ──────────────► parport bitbang function
├── set_cs()       ──────────────► toggle parport pin
│
└── [registered devices]
    │
    ├── spi_device (DataFlash, cs=0)
    │      max_speed_hz = 15000000
    │      mode = SPI_MODE_3
    │
    └── spi_device (AVR as SPI slave, cs=1)  [optional]

spi_message
│
├── spi ──────────────────────────► spi_device (DataFlash)
│
└── transfers (linked list)
    │
    ├── spi_transfer #1: tx_buf = [opcode READ], len=4
    │                    cs_change = false
    │
    └── spi_transfer #2: rx_buf = [data buffer], len=512
                         cs_change = false
```

الـ SPI Core بياخد الـ `spi_message`، بيعدّي على الـ transfers الواحدة واحدة، ويستدعي `transfer_one()` الخاصة بالـ controller (الـ parport bitbanger في حالة butterfly).

---

### الـ Bitbang Pattern: إزاي يشتغل

الـ SPI Core فيه helper layer اسمها **`spi-bitbang`** في `drivers/spi/spi-bitbang.c`. الـ `spi_butterfly` بيعتمد عليها.

```
Protocol Driver
    │
    │  spi_sync(spi, msg)
    ▼
SPI Core
    │
    │  ctlr->transfer_one_message()
    ▼
spi-bitbang layer (generic)
    │
    │  for each transfer in message:
    │      bitbang->set_mosi(val)
    │      bitbang->txrx_word[bits_per_word](spi, speed, word)
    ▼
spi_butterfly specifics
    │
    │  toggle parport data pins
    │  read parport status pins
    ▼
Linux parport subsystem
    │
    │  outb() / inb() to parport I/O ports
    ▼
Physical Parallel Port Hardware
    │
    ▼
AVR Butterfly / DataFlash
```

**الـ parport subsystem** ده subsystem تاني في الكيرنل مسؤول عن abstraction الـ parallel ports. الـ `spi_butterfly` بيكون "client" بتاعه.

---

### SPI Clock Modes: تفصيل مهم

الـ SPI مفيش standard protocol واحد — في 4 modes حسب **CPOL** و**CPHA**:

| Mode | CPOL | CPHA | Clock Idle | Sample Edge | الاستخدام الشائع |
|------|------|------|------------|-------------|-----------------|
| 0    | 0    | 0    | Low        | Rising      | ADCs, sensors كتير |
| 1    | 0    | 1    | Low        | Falling     | أقل شيوعاً |
| 2    | 1    | 0    | High       | Falling     | بعض displays |
| 3    | 1    | 1    | High       | Rising      | Atmel DataFlash |

الـ DataFlash الموجود على الـ AVR Butterfly بيشتغل بـ **Mode 3**. الـ `spi_butterfly` driver لازم يضبط الـ `mode` صح في الـ `spi_device` قبل ما يبدأ يتكلم مع الـ chip.

```
Mode 0 (CPOL=0, CPHA=0):          Mode 3 (CPOL=1, CPHA=1):
─────────────────────────          ─────────────────────────
SCK:  ‾\_/‾\_/‾\_/‾\_             SCK:  _/‾\_/‾\_/‾\_/‾
MOSI: ═══D7══D6══D5══             MOSI: ════D7══D6══D5══
           ↑                                    ↑
       sample here                          sample here
```

---

### ما يملكه الـ SPI Core مقابل ما يفوّضه للـ Drivers

| المسؤولية | SPI Core يملكه؟ | يفوّضه للـ Driver؟ |
|---|---|---|
| Queue management | نعم | لا |
| Message atomicity | نعم | لا |
| sysfs nodes (/sys/bus/spi/) | نعم | لا |
| Device/driver binding | نعم | لا |
| Clock generation | لا | Controller Driver |
| Chip Select toggling | لا | Controller Driver |
| DMA setup/teardown | لا (يساعد فقط) | Controller Driver |
| Bit ordering (MSB/LSB) | لا | Controller Driver |
| Protocol interpretation | لا | Protocol Driver |
| Register map of device | لا | Protocol Driver |
| Error recovery logic | لا | Protocol Driver |

---

### الـ spi_butterfly في السياق الأكبر: Development Tool

الـ `spi_butterfly` مش production driver — هو **development و prototyping tool**. الفكرة الذكية فيه إنه:

1. يخليك تكتب protocol driver عادي مئة بالمئة (`mtd_dataflash` driver).
2. تشغّله مع parport bitbanger على PC عادي.
3. لما تنتقل لـ real hardware بـ SPI controller فعلي (مثلاً `spi-imx`)، **نفس protocol driver يشتغل بدون تغيير سطر واحد**.

ده بالظبط المعنى من الـ SPI abstraction: الـ `mtd_dataflash` driver ما بيعرفش ولا مهتم هل الـ SPI controller ده parport bitbanger أو SoC hardware IP.

```
نفس Protocol Driver (mtd_dataflash):
                │
      ┌─────────┴──────────┐
      ▼                    ▼
spi_butterfly          spi-imx (i.MX6)
(Parport bitbang)      (Hardware DMA)
      │                    │
      ▼                    ▼
AVR Butterfly         Production Board
(Development)         (Production)
```

هو ده سر قوة الـ Linux kernel driver model: **write once, run on any hardware**.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Macros والـ Config Options

#### Pin Mapping — DATA Register (Write-Only, Parport pins 2–9)

| Macro | Value | Parport Pin | الوظيفة |
|---|---|---|---|
| `spi_sck_bit` | `1 << 0` | pin 2 / D0 | **SCK** — clock line للـ SPI |
| `butterfly_nreset` | `1 << 1` | pin 3 / D1 | **nRESET** للـ AVR Butterfly |
| `vcc_bits` | `(1<<6)\|(1<<5)` | pins 7,8 / D5,D6 | تغذية الـ VCC للـ Butterfly وللـ DataFlash |
| `spi_mosi_bit` | `1 << 7` | pin 9 / D7 | **MOSI** — data out |

#### Pin Mapping — STATUS Register (Read-Only)

| Macro | Value | Parport Pin | الوظيفة |
|---|---|---|---|
| `spi_miso_bit` | `PARPORT_STATUS_BUSY` | pin 11 / S7 | **MISO** — data in (NOT negated) |

#### Pin Mapping — CONTROL Register (Write-Only)

| Macro | Value | Parport Pin | الوظيفة |
|---|---|---|---|
| `spi_cs_bit` | `PARPORT_CONTROL_SELECT` | pin 17 / C3 | **nCS** للـ DataFlash (inverted on output) |

#### Bitbang CS States

| Constant | Value | المعنى |
|---|---|---|
| `BITBANG_CS_ACTIVE` | `1` | تفعيل الـ chip select (normally active-low nCS) |
| `BITBANG_CS_INACTIVE` | `0` | تعطيل الـ chip select |

#### SPI Mode المدعوم

| Mode | CPOL | CPHA | المستخدم في الدرايفر |
|---|---|---|---|
| `SPI_MODE_0` | 0 | 0 | نعم — الوحيد المُنفَّذ |
| `SPI_MODE_1/2/3` | — | — | لا |

#### MTD Partitions للـ DataFlash (AT45DB041B)

| Partition | Offset | Size | الوظيفة |
|---|---|---|---|
| `"bookkeeping"` | 0 | (8+248)*264 = ~66 KB | sector 0+1 — bookkeeping |
| `"filesystem"` | `MTDPART_OFS_APPEND` | `MTDPART_SIZ_FULL` | ~462 KB — filesystem |

---

### أهم الـ Structs

#### 1. `struct butterfly`

**الغرض:** الـ struct الرئيسي للدرايفر — يجمع كل حالة الـ adapter في مكان واحد.

```c
struct butterfly {
    /* يجب أن يكون أول عنصر دائماً — bitbang يفترض ده */
    struct spi_bitbang   bitbang;    /* bitbang SPI controller state */

    struct parport      *port;       /* pointer للـ parport الفيزيائي */
    struct pardevice    *pd;         /* تسجيل الدرايفر مع subsystem الـ parport */

    u8                   lastbyte;   /* آخر byte كُتب على DATA register (cached) */

    struct spi_device   *dataflash;  /* الـ AT45DB041B DataFlash device */
    struct spi_device   *butterfly;  /* الـ AVR Butterfly نفسه كـ SPI device */
    struct spi_board_info info[2];   /* static board info للـ device تسجيل */
};
```

**العلاقات:**
- **`bitbang`** → يحمل pointer لـ `struct spi_controller` (الـ `ctlr`) — هو الجسر لـ SPI core.
- **`port`** → `struct parport` — الـ parport الفيزيائي اللي بنكتب عليه bits.
- **`pd`** → `struct pardevice` — تسجيلنا مع parport subsystem عشان نعمل `claim`.
- **`dataflash`** و **`butterfly`** → `struct spi_device` — الأجهزة على الـ SPI bus.
- **`info[2]`** → static array فيه الـ board info لكل device قبل ما يتسجل.

---

#### 2. `struct spi_bitbang`

**الغرض:** wrapper يحوّل أي GPIO/parport لـ SPI controller عن طريق bit-banging.

```c
struct spi_bitbang {
    struct mutex              lock;       /* يحمي حالة الـ busy */
    u8                        busy;       /* هل في transfer جارية؟ */
    u8                        use_dma;    /* DMA support (مش مستخدم هنا) */
    u16                       flags;      /* extra SPI mode flags */

    struct spi_controller    *ctlr;       /* pointer للـ SPI controller */

    int  (*setup_transfer)(struct spi_device *, struct spi_transfer *);
    void (*chipselect)(struct spi_device *, int is_on);  /* butterfly_chipselect */
    void (*set_mosi_idle)(struct spi_device *);
    int  (*txrx_bufs)(struct spi_device *, struct spi_transfer *);

    /* array of txrx functions — واحدة لكل SPI mode */
    spi_bb_txrx_word_fn txrx_word[SPI_MODE_X_MASK + 1];

    int  (*set_line_direction)(struct spi_device *, bool output);
};
```

**العلاقات:**
- **`ctlr`** → `struct spi_controller` — الـ SPI core يشتغل مع ده مش مع `spi_bitbang` مباشرة.
- **`chipselect`** → callback لـ `butterfly_chipselect()`.
- **`txrx_word[SPI_MODE_0]`** → callback لـ `butterfly_txrx_word_mode0()`.

---

#### 3. `struct spi_controller` (host)

**الغرض:** يمثّل الـ SPI bus controller في الـ kernel — الـ SPI core يتعامل معاه.

الـ butterfly بيـset فيه:
```c
host->bus_num        = 42;   /* رقم الـ bus في النظام */
host->num_chipselect = 2;    /* عدد الأجهزة اللي ممكن نتكلم معاها */
```

---

#### 4. `struct parport_driver`

**الغرض:** تسجيل الدرايفر مع الـ parport subsystem.

```c
static struct parport_driver butterfly_driver = {
    .name       = "spi_butterfly",
    .match_port = butterfly_attach,   /* يتكلم لما يلاقي parport */
    .detach     = butterfly_detach,   /* يتكلم لما الـ parport يتشال */
};
```

---

#### 5. `struct spi_board_info` (info[2])

**الغرض:** static description للـ SPI device قبل تسجيله.

```c
/* DataFlash */
pp->info[0].max_speed_hz   = 15 * 1000 * 1000;  /* 15 MHz */
pp->info[0].modalias       = "mtd_dataflash";
pp->info[0].platform_data  = &flash;             /* MTD partitions */
pp->info[0].chip_select    = 1;
pp->info[0].controller_data = pp;               /* يرجع لـ struct butterfly */
```

---

### Struct Relationship Diagram

```
  module_parport_driver(butterfly_driver)
           │
           ▼
  struct parport_driver
  ┌──────────────────────┐
  │ .name = "spi_butterfly"│
  │ .match_port ──────────┼──► butterfly_attach()
  │ .detach ──────────────┼──► butterfly_detach()
  └──────────────────────┘
           │
           │ يتكلم عند detect
           ▼
  struct butterfly  ◄──────────────────────────────────────────┐
  ┌─────────────────────────────────────────────────────────┐   │
  │  [0] struct spi_bitbang  bitbang                        │   │
  │       ┌────────────────────────────────────┐            │   │
  │       │ .ctlr ──────────────────────────────┼───►  struct spi_controller (host)
  │       │ .chipselect → butterfly_chipselect  │            │
  │       │ .txrx_word[0] → butterfly_txrx_word │            │
  │       │ .lock (mutex)                       │            │
  │       └────────────────────────────────────┘            │
  │                                                         │
  │  struct parport      *port ──────────────────────────────┼──► struct parport
  │       ┌─────────────────────────┐                       │         (parport subsystem)
  │       │ parport_write_data()    │                       │
  │       │ parport_read_status()   │                       │
  │       │ parport_frob_control()  │                       │
  │       └─────────────────────────┘                       │
  │                                                         │
  │  struct pardevice    *pd ────────────────────────────────┼──► struct pardevice
  │       └── parport_claim() / parport_release()           │
  │                                                         │
  │  u8 lastbyte   (shadow register للـ DATA pins)          │
  │                                                         │
  │  struct spi_device   *dataflash ◄── spi_new_device()    │
  │       └── modalias = "mtd_dataflash"                    │
  │           platform_data → struct flash_platform_data    │
  │               └── parts[] → struct mtd_partition[2]     │
  │                                                         │
  │  struct spi_device   *butterfly  (unused currently)     │
  │                                                         │
  │  struct spi_board_info info[2]                          │
  │       └── controller_data = pp ────────────────────────┘
  └─────────────────────────────────────────────────────────┘
```

---

### Lifecycle Diagram

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                    MODULE INIT                                   │
  │  module_parport_driver(butterfly_driver)                        │
  │    → parport_register_driver()                                  │
  │    → kernel scans existing parports → calls butterfly_attach()  │
  └────────────────────┬────────────────────────────────────────────┘
                       │
                       ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │                  butterfly_attach(parport *p)                    │
  │                                                                  │
  │  1. spi_alloc_host(dev, sizeof(*pp))                            │
  │       → allocates struct spi_controller + struct butterfly       │
  │                                                                  │
  │  2. host->bus_num = 42, host->num_chipselect = 2                │
  │     pp->bitbang.ctlr = host                                     │
  │     pp->bitbang.chipselect = butterfly_chipselect               │
  │     pp->bitbang.txrx_word[SPI_MODE_0] = butterfly_txrx_word_mode0│
  │                                                                  │
  │  3. parport_register_dev_model() → struct pardevice *pd         │
  │  4. parport_claim(pd) → نحجز الـ parport لنفسنا               │
  │                                                                  │
  │  5. Power-up sequence:                                           │
  │       parport_frob_control(nCS=0)   ← DataFlash deselected      │
  │       lastbyte |= vcc_bits → parport_write_data()  ← VCC ON     │
  │       msleep(5)                                                  │
  │       lastbyte |= butterfly_nreset → write()  ← take out reset  │
  │       msleep(100)                                                │
  │                                                                  │
  │  6. spi_bitbang_start(&pp->bitbang)                             │
  │       → registers spi_controller with SPI core                  │
  │                                                                  │
  │  7. spi_new_device(host, &info[0])                              │
  │       → creates spi_device for DataFlash                        │
  │       → binds "mtd_dataflash" driver                            │
  │                                                                  │
  │  8. butterfly = pp  (global pointer)                             │
  └────────────────────┬────────────────────────────────────────────┘
                       │
                       ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │                    RUNTIME — SPI Transfers                       │
  │                                                                  │
  │  mtd_dataflash driver calls spi_sync() / spi_write_then_read()  │
  │    → SPI core → spi_bitbang → butterfly_chipselect(ACTIVE)      │
  │    → butterfly_txrx_word_mode0()                                │
  │         → bitbang_txrx_be_cpha0()                               │
  │              → setsck() / setmosi() / getmiso()                 │
  │                   → parport_write_data() / parport_read_status() │
  │    → butterfly_chipselect(INACTIVE)                             │
  └────────────────────┬────────────────────────────────────────────┘
                       │
                       ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │                  butterfly_detach(parport *p)                    │
  │                                                                  │
  │  1. spi_bitbang_stop(&pp->bitbang)                              │
  │       → unregisters spi_controller + child spi_devices          │
  │  2. parport_write_data(port, 0) ← VCC OFF                       │
  │  3. msleep(10)                                                   │
  │  4. parport_release(pd)                                         │
  │  5. parport_unregister_device(pd)                               │
  │  6. spi_controller_put(host) ← free memory                      │
  └─────────────────────────────────────────────────────────────────┘
```

---

### Call Flow Diagrams

#### SPI Transfer Flow

```
  mtd_dataflash driver
    └─► spi_sync(spi_device, spi_message)
          └─► SPI core queue
                └─► spi_bitbang (txrx_bufs default)
                      ├─► butterfly_chipselect(spi, BITBANG_CS_ACTIVE)
                      │     ├─► setsck(spi, SPI_CPOL=0)       ← SCK low
                      │     │     └─► parport_write_data(port, byte)
                      │     └─► parport_frob_control(port, spi_cs_bit, 0)
                      │                                         ← nCS asserted (inverted)
                      │
                      ├─► butterfly_txrx_word_mode0(spi, nsecs, word, bits, flags)
                      │     └─► bitbang_txrx_be_cpha0(spi, nsecs, 0, flags, word, bits)
                      │           ├─► setmosi(spi, bit)
                      │           │     └─► parport_write_data(port, byte | mosi_bit)
                      │           ├─► setsck(spi, 1)  ← rising edge → latch
                      │           │     └─► parport_write_data(port, byte | sck_bit)
                      │           ├─► getmiso(spi)
                      │           │     └─► parport_read_status(port) & BUSY
                      │           └─► setsck(spi, 0)  ← falling edge
                      │
                      └─► butterfly_chipselect(spi, BITBANG_CS_INACTIVE)
                            └─► parport_frob_control(port, spi_cs_bit, spi_cs_bit)
                                                         ← nCS deasserted
```

#### Attach Flow (Power-up Sequence)

```
  parport subsystem detects port
    └─► butterfly_attach(parport *p)
          ├─► spi_alloc_host()          ← allocate controller + private data
          ├─► setup bitbang ops
          ├─► parport_register_dev_model()  ← register with parport
          ├─► parport_claim()           ← exclusive access to port
          │
          ├─► [Power Sequence]
          │     ├─► frob_control(nCS=0) ← DataFlash off
          │     ├─► write_data(vcc_bits) ← VCC on (2 signal pins as power!)
          │     ├─► msleep(5)
          │     ├─► write_data(|= nreset) ← AVR out of reset
          │     └─► msleep(100)
          │
          ├─► spi_bitbang_start()       ← register SPI controller with kernel
          └─► spi_new_device()          ← create & bind DataFlash spi_device
```

#### Chipselect Logic Detail

```
  butterfly_chipselect(spi, value)
    │
    ├─ value != BITBANG_CS_INACTIVE ?
    │    └─► setsck(spi, spi->mode & SPI_CPOL)   ← set clock idle state first
    │
    ├─ spi_cs_bit == PARPORT_CONTROL_INIT ?
    │    └─► value = !value                        ← invert if needed (not our case)
    │
    └─► parport_frob_control(port, spi_cs_bit, value ? spi_cs_bit : 0)
         ← PARPORT_CONTROL_SELECT is already inverted in hardware,
           so writing bit=1 → pin LOW → nCS asserted ✓
```

---

### Locking Strategy

#### Overview

الدرايفر ده **بسيط جداً** من ناحية الـ locking لأنه:
1. بيشتغل على **parport واحد** فقط (global `butterfly` pointer).
2. الـ SPI transfers تحت سيطرة **`spi_bitbang.lock`** اللي بيديره الـ bitbang framework.
3. مفيش interrupt handlers — كل حاجة polling.

#### Locking Table

| Lock | نوعه | موجود في | بيحمي إيه |
|---|---|---|---|
| `spi_bitbang.lock` | `struct mutex` | `struct spi_bitbang` | تسلسل الـ SPI transfers، حالة `busy` |
| implicit parport lock | داخلي في `parport_claim()` | parport subsystem | exclusive access للـ physical port |
| global `butterfly` ptr | implicit (single-threaded attach) | module level | منع تسجيل أكتر من adapter واحد |

#### Lock Ordering

```
  parport_claim()  [يحصل أولاً في attach — يحجز الـ port]
       │
       ▼
  spi_bitbang.lock  [يحصل لكل transfer — يحمي التسلسل]
       │
       ▼
  parport DATA/STATUS/CONTROL registers  [hardware access — no sleep allowed هنا]
```

**ملاحظة مهمة:** الـ `spidelay(X)` معمول `do {} while(0)` — يعني الدرايفر مش بيعمل أي delay بين الـ clock edges، وده بيخليه أسرع بس ممكن يعمل مشاكل مع أجهزة بطيئة.

#### Global State Problem

الكود نفسه بيعلّق على المشكلة:
```c
/* REVISIT remove this ugly global and its "only one" limitation */
static struct butterfly *butterfly;
```

الـ global pointer مش محمي بـ lock رسمي — بيعتمد على إن `butterfly_attach` و`butterfly_detach` بيتنادوا من parport subsystem بشكل serialized، لكن ده assumption مش مضمون بالكامل في multi-CPU systems.
## Phase 4: شرح الـ Functions

---

### ملخص عام — Cheatsheet

#### GPIO / Bitbang Primitives

| Function | Signature (simplified) | Purpose |
|---|---|---|
| `setsck` | `(spi_device*, int is_on)` | يضبط bit الـ SCK في الـ parport data register |
| `setmosi` | `(spi_device*, int is_on)` | يضبط bit الـ MOSI في الـ parport data register |
| `getmiso` | `(spi_device*) → int` | يقرأ حالة الـ MISO من الـ parport status register |
| `butterfly_chipselect` | `(spi_device*, int value)` | يتحكم في الـ chip-select عبر parport control register |
| `butterfly_txrx_word_mode0` | `(spi_device*, nsecs, word, bits, flags) → u32` | ينفذ TX/RX كلمة واحدة على SPI_MODE_0 |

#### Bitbang TX/RX Helpers (من spi-bitbang-txrx.h)

| Function | Purpose |
|---|---|
| `bitbang_txrx_be_cpha0` | Big-endian, CPHA=0 — SPI MODE 0 أو 2 |
| `bitbang_txrx_be_cpha1` | Big-endian, CPHA=1 — SPI MODE 1 أو 3 |
| `bitbang_txrx_le_cpha0` | Little-endian, CPHA=0 |
| `bitbang_txrx_le_cpha1` | Little-endian, CPHA=1 |

#### Registration / Lifecycle (SPI Bitbang Framework)

| Function | Purpose |
|---|---|
| `spi_alloc_host` | يخصص `spi_controller` + private data |
| `spi_controller_get_devdata` | يجيب pointer للـ private data من الـ controller |
| `spi_bitbang_start` | يسجل الـ controller ويبدأ queue |
| `spi_bitbang_stop` | يوقف الـ queue ويلغي تسجيل child devices |
| `spi_new_device` | ينشئ `spi_device` جديد على الـ bus |
| `spi_controller_put` | يحرر الـ reference على الـ controller |

#### Parport Registration

| Function | Purpose |
|---|---|
| `parport_register_dev_model` | يسجل driver على الـ parport |
| `parport_claim` | يطالب بالحصرية على الـ port |
| `parport_release` | يحرر الحصرية |
| `parport_unregister_device` | يلغي تسجيل device من الـ parport |
| `parport_write_data` | يكتب byte على data pins (D0-D7) |
| `parport_read_status` | يقرأ الـ status pins (S5-S7) |
| `parport_frob_control` | يعدّل بعض bits في الـ control register |

#### Module Entry / Notification

| Macro/Symbol | Purpose |
|---|---|
| `module_parport_driver` | يسجل `parport_driver` تلقائياً عند load/unload الـ module |

---

### المجموعة 1: GPIO Bit Manipulation Primitives

هذه الـ functions هي أدنى طبقة في الـ driver — كل واحدة منها مسؤولة عن تغيير bit واحد على الـ parallel port لمحاكاة إشارة SPI معينة. الفكرة الأساسية هي أن الـ driver يحتفظ بـ `lastbyte` داخل `struct butterfly` عشان يتجنب الـ read-modify-write على الـ hardware في كل مرة — هو يعدّل الـ byte في الـ RAM ثم يكتب الـ byte كله دفعة واحدة.

---

#### `spidev_to_pp`

```c
static inline struct butterfly *spidev_to_pp(struct spi_device *spi)
{
    return spi->controller_data;
}
```

**ما تعمله:** تعمل cast من `spi_device` للـ `struct butterfly` الخاصة بيه. الـ `controller_data` field اتضبطت يدوياً في `butterfly_attach` لما اتعمل `spi_board_info`.

**Parameters:**
- `spi` — الـ SPI device المطلوب إيجاد الـ parent adapter بتاعه

**Return:** pointer لـ `struct butterfly` — مضمون مش NULL لأنه اتضبط وقت التسجيل

**Key details:** مفيش locking هنا — الـ caller هو اللي مسؤول عن الـ context. دي helper خالصة.

---

#### `setsck`

```c
static inline void setsck(struct spi_device *spi, int is_on)
{
    struct butterfly *pp = spidev_to_pp(spi);
    u8 bit, byte = pp->lastbyte;

    bit = spi_sck_bit;  /* bit 0 = parport D0 = pin 2 */

    if (is_on)
        byte |= bit;
    else
        byte &= ~bit;
    parport_write_data(pp->port, byte);
    pp->lastbyte = byte;
}
```

**ما تعمله:** ترفع أو تخفض خط الـ SCK (Serial Clock) على الـ parallel port. بتعدّل الـ D0 pin (pin 2 في الـ DB-25) اللي متوصل بـ SCK في الـ Butterfly connector J403.

**Parameters:**
- `spi` — الـ SPI device (من الـ bitbang framework)
- `is_on` — 1 يرفع الـ clock، 0 يخفضه

**Return:** void

**Key details:** الـ `spi_sck_bit = (1 << 0)` يعني أدنى bit في data byte. الكتابة على الـ parport بيتم عبر `parport_write_data` اللي بتكتب الـ 8 bits دفعة واحدة — لازم `lastbyte` يكون محدّث عشان مينكسرش أي signal تاني.

**Caller:** `bitbang_txrx_be_cpha0` في الـ TX/RX loop

---

#### `setmosi`

```c
static inline void setmosi(struct spi_device *spi, int is_on)
{
    struct butterfly *pp = spidev_to_pp(spi);
    u8 bit, byte = pp->lastbyte;

    bit = spi_mosi_bit;  /* bit 7 = parport D7 = pin 9 */

    if (is_on)
        byte |= bit;
    else
        byte &= ~bit;
    parport_write_data(pp->port, byte);
    pp->lastbyte = byte;
}
```

**ما تعمله:** تضبط بيت الـ MOSI (Master Out Slave In) — وهو الـ D7 pin على الـ parport، pin 9 على الـ DB-25، متوصل بـ PB2/MOSI في الـ Butterfly.

**Parameters:**
- `spi` — الـ SPI device
- `is_on` — قيمة البيت اللي هيتبعته للـ slave

**Return:** void

**Key details:** `spi_mosi_bit = (1 << 7)` — أعلى bit في data register. نفس pattern الـ `setsck` بالظبط من حيث read-cache-modify-write.

**Caller:** `bitbang_txrx_be_cpha0` — بيتبعت كل bit في الـ word MSB-first

---

#### `getmiso`

```c
static inline int getmiso(struct spi_device *spi)
{
    struct butterfly *pp = spidev_to_pp(spi);
    int value;
    u8 bit;

    bit = spi_miso_bit;  /* = PARPORT_STATUS_BUSY = pin 11 */

    /* only STATUS_BUSY is NOT negated */
    value = !(parport_read_status(pp->port) & bit);
    return (bit == PARPORT_STATUS_BUSY) ? value : !value;
}
```

**ما تعمله:** تقرأ حالة الـ MISO من الـ slave. بتستخدم الـ nBUSY/S7 pin (pin 11) في الـ parport status register.

**Parameters:**
- `spi` — الـ SPI device

**Return:** 0 أو 1 فقط — مهم جداً لأن الـ `bitbang_txrx_be_cpha0` بيعمل `word |= getmiso()` مباشرة

**Key details:** الـ BUSY pin هو الوحيد في الـ status register اللي **مش** معكوس (NOT negated) على hardware level. الـ logic هنا:
- `value = !(status & BUSY_bit)` — عكس أول لأن active = low
- لو الـ bit هو BUSY (وهو الحالة دي): return `value` (نتيجة التقلب الأول بس)
- لو bit تاني (معكوس hardware): return `!value` (تقلبين = بيرجع للأصل)

النتيجة: دايماً 0 أو 1 تعبر عن القيمة الحقيقية على الـ wire.

**Caller:** `bitbang_txrx_be_cpha0` في نهاية كل clock cycle لقراءة البيت المستلم

---

### المجموعة 2: Chip Select Control

#### `butterfly_chipselect`

```c
static void butterfly_chipselect(struct spi_device *spi, int value)
{
    struct butterfly *pp = spidev_to_pp(spi);

    /* set default clock polarity */
    if (value != BITBANG_CS_INACTIVE)
        setsck(spi, spi->mode & SPI_CPOL);

    if (spi_cs_bit == PARPORT_CONTROL_INIT)
        value = !value;

    parport_frob_control(pp->port, spi_cs_bit, value ? spi_cs_bit : 0);
}
```

**ما تعمله:** تتحكم في إشارة الـ Chip Select للـ DataFlash. الـ CS بيتبعت على الـ PARPORT_CONTROL_SELECT (pin 17، C3). لما بيتفعّل الـ CS بيضبط كمان الـ clock polarity الصح بناءً على `SPI_CPOL`.

**Parameters:**
- `spi` — الـ SPI device المراد activate/deactivate الـ CS بتاعه
- `value` — `BITBANG_CS_ACTIVE` (1) أو `BITBANG_CS_INACTIVE` (0)

**Return:** void

**Key details:** معظم bits في الـ PARPORT_CONTROL register بتكون **معكوسة** hardware. الـ code بيتعامل مع الحالة الخاصة بـ `PARPORT_CONTROL_INIT` (اللي معكوس)، لكن `PARPORT_CONTROL_SELECT` **مش معكوس**، فمفيش inversion. الـ `parport_frob_control` بيضبط bit معين من غير ما يأثر على الـ bits التانية.

**Caller:** الـ `spi_bitbang` framework بيستدعيها قبل وبعد كل SPI transfer

---

### المجموعة 3: TX/RX Data Transfer

#### `butterfly_txrx_word_mode0`

```c
static u32 butterfly_txrx_word_mode0(struct spi_device *spi,
                                      unsigned nsecs, u32 word,
                                      u8 bits, unsigned flags)
{
    return bitbang_txrx_be_cpha0(spi, nsecs, 0, flags, word, bits);
}
```

**ما تعمله:** الـ wrapper الخاص بالـ driver لتنفيذ نقل كلمة واحدة بـ SPI_MODE_0 (CPOL=0, CPHA=0). بيفوّض الشغل كله لـ `bitbang_txrx_be_cpha0` من الـ shared header.

**Parameters:**
- `spi` — الـ device المتكلم معاه
- `nsecs` — التأخير بين transitions للـ clock (0 في الـ driver ده لأن `spidelay` = NOP)
- `word` — الكلمة المطلوب إرسالها (MSB-first)
- `bits` — عدد البيتات في الكلمة (عادةً 8)
- `flags` — `SPI_CONTROLLER_NO_TX` أو `SPI_CONTROLLER_NO_RX` لو half-duplex

**Return:** الـ word المستلم من الـ slave (RX)

**Key details:** الـ `#define spidelay(X) do { } while (0)` يعني إن الـ driver بيشتغل بأقصى سرعة ممكنة للـ parport من غير delays — مناسب لأن الـ parport نفسه بطيء كفاية. الـ `cpol=0` يعني الـ clock يبدأ low.

**Caller:** `spi_bitbang` framework بيستدعي `txrx_word[SPI_MODE_0]` لكل كلمة في الـ transfer

---

#### `bitbang_txrx_be_cpha0` (من spi-bitbang-txrx.h)

```c
static inline u32 bitbang_txrx_be_cpha0(struct spi_device *spi,
        unsigned nsecs, unsigned cpol, unsigned flags,
        u32 word, u8 bits)
```

**ما تعمله:** ينفذ الـ SPI bit-banging loop الفعلي بـ big-endian bit order وـ CPHA=0. يبعت ويستقبل في نفس الوقت (full-duplex) bit by bit، MSB first.

**Parameters:**
- `spi` — الـ device
- `nsecs` — setup time بالنانوثانية
- `cpol` — قطبية الـ clock الساكنة (0 للـ MODE 0 و 2)
- `flags` — `SPI_CONTROLLER_NO_TX` / `SPI_CONTROLLER_NO_RX`
- `word` — الكلمة المطلوب إرسالها
- `bits` — عدد البيتات

**Return:** الـ word المستلم

**Pseudocode Flow:**

```
word <<= (32 - bits)          // align MSB to bit31

for each bit (MSB first):
    if TX enabled and MSB changed:
        setmosi(spi, word & bit31)  // set MOSI
    spidelay(nsecs)                 // T(setup)

    setsck(spi, !cpol)             // rising edge (CPOL=0 → go HIGH)
    spidelay(nsecs)

    word <<= 1                      // shift out sent bit
    if RX enabled:
        word |= getmiso(spi)        // sample on leading edge

    setsck(spi, cpol)              // return clock to idle (LOW)

return word                         // contains received data in low bits
```

**Key details:** الـ optimization الذكي هو `oldbit` — بيتجنب كتابة `setmosi` لو البيت الجديد نفس السابق، ده بيقلل الـ parport writes ويزيد السرعة. CPHA=0 يعني الـ data يتثبت قبل الـ rising edge ويُقرأ عليها.

---

### المجموعة 4: Driver Registration & Lifecycle

#### `butterfly_attach`

```c
static void butterfly_attach(struct parport *p)
```

**ما تعمله:** الـ probe function الخاصة بالـ parport driver. بيُستدعى أوتوماتيك لما يُكتشف parallel port. بيعمل كل حاجة من تخصيص الـ controller لحد تسجيل الـ SPI devices.

**Parameters:**
- `p` — الـ `struct parport` اللي تم اكتشافه بواسطة الـ parport subsystem

**Return:** void (الأخطاء بتتعامل معها داخلياً بـ goto)

**Key details:**
- مفيش real probing — بيفترض دايماً إن Butterfly متوصل
- الـ `bus_num = 42` hardcoded — مش مشكلة لأنه للـ testing
- بيستخدم global `butterfly` pointer (قيد "one adapter only")
- الـ VCC بيتوفر من data pins (D6+D5) — مش ideal لكنه يشتغل
- الـ reset sequence: assert reset → wait 5ms → deassert → wait 100ms

**Pseudocode Flow:**

```
if (butterfly already exists || no device) → return

host = spi_alloc_host(dev, sizeof(*pp))
pp = spi_controller_get_devdata(host)

// setup SPI bitbang
host->bus_num = 42
host->num_chipselect = 2
pp->bitbang.chipselect = butterfly_chipselect
pp->bitbang.txrx_word[SPI_MODE_0] = butterfly_txrx_word_mode0

// register with parport
pd = parport_register_dev_model(p, "spi_butterfly", ...)
parport_claim(pd)

// power up sequence
parport_frob_control(...)      // CS inactive
pp->lastbyte |= vcc_bits       // enable VCC
parport_write_data(...)
msleep(5)
pp->lastbyte |= butterfly_nreset  // release RESET
parport_write_data(...)
msleep(100)

// start SPI subsystem
spi_bitbang_start(&pp->bitbang)

// register DataFlash SPI device (mtd_dataflash driver)
pp->dataflash = spi_new_device(pp->bitbang.ctlr, &pp->info[0])

butterfly = pp  // save global reference
return

// error paths:
clean2: parport_write_data(0)  // cut VCC
        parport_release(pp->pd)
clean1: parport_unregister_device(pd)
clean0: spi_controller_put(host)
```

**Caller:** الـ parport subsystem بيستدعيها كـ `match_port` callback عند كل parport discovery

---

#### `butterfly_detach`

```c
static void butterfly_detach(struct parport *p)
```

**ما تعمله:** cleanup function — بيُستدعى لما الـ parport يتشال أو الـ module يتـ unload. بيوقف كل الـ SPI activity، يطفي الـ VCC، ويحرر كل الـ resources.

**Parameters:**
- `p` — الـ parport المراد فصله

**Return:** void

**Key details:**
- `spi_bitbang_stop` بيـ unregister الـ child SPI devices (dataflash) تلقائياً
- ترتيب الـ cleanup مهم: أوقف SPI أولاً، اقطع الكهرباء، حرر الـ parport
- الـ `msleep(10)` بعد قطع الـ VCC بيدي وقت للـ decoupling capacitors تتفرغ
- الـ global `butterfly` بيتصفر قبل أي cleanup

**Cleanup sequence:**

```
spi_bitbang_stop(&pp->bitbang)   // unregisters dataflash device too
parport_write_data(pp->port, 0)  // VCC off (D5+D6 = 0)
msleep(10)
parport_release(pp->pd)
parport_unregister_device(pp->pd)
spi_controller_put(pp->bitbang.ctlr)
```

**Caller:** الـ parport subsystem كـ `detach` callback

---

### المجموعة 5: SPI Framework APIs المستخدمة

#### `spi_alloc_host`

```c
struct spi_controller *spi_alloc_host(struct device *dev, unsigned int size)
```

**ما تعمله:** يخصص `struct spi_controller` مع private data بحجم `size`. الـ private data بتتضمن هنا `struct butterfly` كاملة.

**Parameters:**
- `dev` — الـ parent device (physport->dev)
- `size` — حجم الـ private data المطلوب

**Return:** pointer لـ `spi_controller` أو NULL عند فشل الـ allocation

**Key details:** بيستخدم `kzalloc` داخلياً — الـ private data بتكون zeroed. الـ controller مش مسجّل بعد (ده شغل `spi_bitbang_start`).

---

#### `spi_bitbang_start`

```c
int spi_bitbang_start(struct spi_bitbang *spi)
```

**ما تعمله:** يسجل الـ SPI controller في الـ kernel SPI subsystem ويبدأ الـ transfer queue. بعد استدعائه، ممكن تضيف SPI devices وتعمل transfers.

**Parameters:**
- `spi` — الـ `struct spi_bitbang` المضبوطة مسبقاً (chipselect + txrx_word مضبوطين)

**Return:** 0 عند النجاح، error code سالب عند الفشل

**Key details:** داخلياً بيستدعي `spi_register_controller` — بعدها الـ driver بيظهر في `/sys/bus/spi/`.

---

#### `spi_bitbang_stop`

```c
void spi_bitbang_stop(struct spi_bitbang *spi)
```

**ما تعمله:** عكس `spi_bitbang_start` — يلغي تسجيل الـ controller ويـ unregister كل الـ child SPI devices المربوطة بيه.

**Parameters:**
- `spi` — نفس الـ bitbang struct

**Return:** void

**Key details:** بيستدعي `spi_unregister_controller` اللي بيـ trigger `spi_unregister_device` على كل device على الـ bus. يعني `dataflash->remove()` بيتستدعي أوتوماتيك.

---

#### `spi_new_device`

```c
struct spi_device *spi_new_device(struct spi_controller *ctlr,
                                   struct spi_board_info *chip)
```

**ما تعمله:** ينشئ ويسجل `spi_device` جديد على bus محدد بناءً على الـ `spi_board_info`. في الـ driver ده بيُستخدم لتسجيل الـ at45db041b DataFlash chip.

**Parameters:**
- `ctlr` — الـ SPI controller المراد إضافة الـ device عليه
- `chip` — معلومات الـ device (modalias, max_speed_hz, chip_select, platform_data)

**Return:** pointer لـ `spi_device` أو NULL عند الفشل — الـ driver (mtd_dataflash) بيتربط بيه أوتوماتيك عبر modalias matching

---

### المجموعة 6: Module Registration

#### `module_parport_driver`

```c
module_parport_driver(butterfly_driver);
```

**ما تعمله:** macro بيوسّع لـ `module_init` + `module_exit` تسجل وتلغي تسجيل `struct parport_driver` تلقائياً.

**الـ struct:**

```c
static struct parport_driver butterfly_driver = {
    .name       = "spi_butterfly",
    .match_port = butterfly_attach,   /* called on port discovery */
    .detach     = butterfly_detach,   /* called on port removal */
};
```

**Key details:** الـ `.match_port` بيتستدعى لكل parport موجود في الـ system وقت الـ registration، وكمان لأي parport جديد يظهر لاحقاً. مفيش `.probe` بالمعنى الحديث — الـ matching بيحصل على أي port.

---

### تدفق الـ SPI Transfer الكامل

```
Protocol Driver (mtd_dataflash)
        |
        | spi_sync() / spi_message
        ▼
spi_bitbang framework
        |
        | for each spi_transfer in message:
        |   butterfly_chipselect(spi, BITBANG_CS_ACTIVE)
        |   for each word:
        |       butterfly_txrx_word_mode0(spi, 0, word, 8, flags)
        |           └─► bitbang_txrx_be_cpha0(...)
        |                   ├─► setmosi(spi, bit)  ──► parport D7
        |                   ├─► setsck(spi, 1)     ──► parport D0
        |                   ├─► getmiso(spi)        ◄── parport S7
        |                   └─► setsck(spi, 0)     ──► parport D0
        |   butterfly_chipselect(spi, BITBANG_CS_INACTIVE)
        ▼
spi_message complete callback
```

---

### تخطيط الـ struct butterfly

```
struct butterfly {
    struct spi_bitbang  bitbang;      ← MUST be first (pointer cast trick)
    │   ├── struct spi_controller *ctlr
    │   ├── chipselect fn ptr
    │   └── txrx_word[SPI_MODE_0] fn ptr
    │
    struct parport      *port;        ← parport hardware handle
    struct pardevice    *pd;          ← registered pardevice
    u8                   lastbyte;    ← shadow of parport data register
    │
    struct spi_device   *dataflash;   ← at45db041b SPI device
    struct spi_device   *butterfly;   ← AVR butterfly SPI device (optional)
    struct spi_board_info info[2];    ← board info for both devices
}
```

**ملاحظة مهمة:** التعليق `/* REVISIT ... for now, this must be first */` مهم — الـ `spi_bitbang` لازم يكون أول حاجة في الـ struct عشان الـ `spi_controller_get_devdata` بيرجع pointer للـ struct كلها، والـ bitbang framework بيتعامل معها كـ `struct spi_bitbang *` مباشرة في بعض المسارات.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. مدخلات الـ debugfs

الـ `spi_butterfly` بيشتغل كـ SPI bitbang controller فوق الـ parport، مفيش لها `debugfs` entries خاصة بيها مباشرة، بس الـ SPI subsystem العام بيوفر:

```bash
# mount debugfs لو مش mounted
mount -t debugfs none /sys/kernel/debug

# شوف SPI controller entries
ls /sys/kernel/debug/spi*
ls /sys/kernel/debug/mtd/

# DataFlash partition info
cat /sys/kernel/debug/mtd/mtd0
```

الـ bus number للـ butterfly هو **42** (hardcoded في `butterfly_attach`):
```bash
# تحقق إن الـ controller اتسجل صح
ls /sys/kernel/debug/ | grep spi
```

---

#### 2. مدخلات الـ sysfs

الـ SPI subsystem بيظهر في sysfs على الشكل ده:

```bash
# الـ SPI host controller (bus 42)
ls /sys/class/spi_master/spi42/

# الـ SPI device للـ DataFlash (bus 42, chipselect 1)
ls /sys/bus/spi/devices/spi42.1/

# modalias: هيقولك الـ driver المربوط بالـ device
cat /sys/bus/spi/devices/spi42.1/modalias
# expected: mtd_dataflash

# الـ MTD device اللي اتعمل من الـ DataFlash
ls /sys/class/mtd/
cat /sys/class/mtd/mtd0/name
# expected: butterflash

# parport device
ls /sys/bus/parport/devices/
cat /sys/bus/parport/devices/parport0/drvctl
```

| الـ sysfs Path | الـ Value المتوقعة | المعنى |
|---|---|---|
| `/sys/class/spi_master/spi42/` | directory | الـ butterfly controller اتسجل |
| `/sys/bus/spi/devices/spi42.1/modalias` | `mtd_dataflash` | الـ DataFlash device موجود |
| `/sys/class/mtd/mtd0/name` | `butterflash` | الـ MTD layer شاف الـ flash |
| `/sys/class/mtd/mtd0/size` | `528384` (≈528KB) | الـ AT45DB041B هو 528KB |

---

#### 3. الـ ftrace — Tracepoints والـ Events

```bash
# enable tracing للـ SPI subsystem
echo 1 > /sys/kernel/debug/tracing/events/spi/enable

# أو تفعيل events بشكل selective
echo 1 > /sys/kernel/debug/tracing/events/spi/spi_transfer_start/enable
echo 1 > /sys/kernel/debug/tracing/events/spi/spi_transfer_stop/enable
echo 1 > /sys/kernel/debug/tracing/events/spi/spi_controller_busy/enable
echo 1 > /sys/kernel/debug/tracing/events/spi/spi_controller_idle/enable
echo 1 > /sys/kernel/debug/tracing/events/spi/spi_message_start/enable
echo 1 > /sys/kernel/debug/tracing/events/spi/spi_message_done/enable

# اقرأ الـ trace
cat /sys/kernel/debug/tracing/trace

# مثال على function tracing للـ bitbang
echo butterfly_txrx_word_mode0 > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... شغّل العملية اللي عايز تتتبعها ...
cat /sys/kernel/debug/tracing/trace
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

**مثال على output متوقع:**

```
     kworker/0:0-123   [000] ....  123.456789: spi_transfer_start: spi42.1 len=4
     kworker/0:0-123   [000] ....  123.456799: spi_transfer_stop: spi42.1 len=4
```

---

#### 4. الـ printk والـ Dynamic Debug

الـ driver بيستخدم `pr_debug()` في أماكن استراتيجية — بالـ dynamic debug تقدر تفعّلها بدون recompile:

```bash
# تفعيل كل الـ pr_debug في الـ spi-butterfly module
echo "module spi_butterfly +p" > /sys/kernel/debug/dynamic_debug/control

# أو تفعيل بـ file name
echo "file drivers/spi/spi-butterfly.c +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل للـ SPI bitbang core كمان
echo "file drivers/spi/spi-bitbang.c +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل للـ mtd_dataflash driver
echo "module mtd_dataflash +p" > /sys/kernel/debug/dynamic_debug/control

# تحقق من الـ dynamic debug المفعّلة دلوقتي
cat /sys/kernel/debug/dynamic_debug/control | grep butterfly
```

**الـ pr_debug messages المهمة في الكود:**

```c
pr_debug("%s: powerup/reset Butterfly\n", p->name);
// ظهورها = الـ parport اتعرف والـ driver بدأ يشغّل الـ AVR

pr_debug("%s: dataflash at %s\n", p->name, dev_name(&pp->dataflash->dev));
// ظهورها = الـ DataFlash chip اتعرف وعنده spi device

pr_debug("%s: butterfly probe, fail %d\n", p->name, status);
// ظهورها = الـ probe فشل، والـ status هو الـ error code
```

**kernel boot params لو عايز verbosity من الأول:**

```bash
# في /etc/default/grub
GRUB_CMDLINE_LINUX="... dyndbg='module spi_butterfly +p' loglevel=8"
```

---

#### 5. الـ Kernel Config Options للـ Debugging

```bash
# تحقق من الـ config الحالية
zcat /proc/config.gz | grep -E "CONFIG_(SPI|MTD|PARPORT|DEBUG)" | sort
```

| الـ Config Option | الغرض |
|---|---|
| `CONFIG_SPI_DEBUG` | تفعيل verbose logging في الـ SPI core |
| `CONFIG_SPI_BUTTERFLY` | الـ driver نفسه (يجب يكون `=m` أو `=y`) |
| `CONFIG_SPI_BITBANG` | الـ bitbang core (dependency) |
| `CONFIG_MTD_DATAFLASH` | الـ DataFlash protocol driver |
| `CONFIG_MTD_DATAFLASH_WRITE_VERIFY` | verify بعد كل write للـ DataFlash |
| `CONFIG_MTD_PARTITIONS` | دعم الـ partitioning للـ flash |
| `CONFIG_PARPORT_PC` | الـ PC parallel port driver |
| `CONFIG_PARPORT_PC_FIFO` | الـ FIFO mode للـ parport |
| `CONFIG_DEBUG_SPI` | debug checks في الـ SPI framework |
| `CONFIG_DYNAMIC_DEBUG` | لازم يكون مفعّل عشان تشتغل `pr_debug` |
| `CONFIG_KALLSYMS` | مهم لقراءة stack traces |
| `CONFIG_FRAME_POINTER` | stack traces أوضح |
| `CONFIG_KPROBES` | لو عايز تضع probes ديناميكية |

```bash
# تحقق إن الـ modules اتحمّلوا صح
lsmod | grep -E "spi_butterfly|spi_bitbang|mtd_dataflash|parport"
```

---

#### 6. أدوات الـ Subsystem

```bash
# مشاهدة الـ MTD devices
cat /proc/mtd
# expected output:
# dev:    size   erasesize  name
# mtd0: 00010590 00000104  "bookkeeping"
# mtd1: 00072390 00000104  "filesystem"

# قراءة من الـ DataFlash مباشرة (اختبار hardware)
dd if=/dev/mtd0 bs=264 count=8 | hexdump -C

# أو باستخدام mtd-utils
mtdinfo /dev/mtd0
mtdinfo /dev/mtd1

# flash_eraseall للـ filesystem partition
flash_eraseall /dev/mtd1

# نانوسيكل بسيط: اكتب بيانات test واقرأها تاني
echo "test data" | dd of=/dev/mtd0 bs=1 seek=0
dd if=/dev/mtd0 bs=9 count=1 | cat

# parport status
cat /proc/sys/dev/parport/parport0/modes
cat /proc/sys/dev/parport/parport0/base-addr
```

---

#### 7. جدول رسائل الـ Errors الشائعة

| الـ Error Message في الـ dmesg | المعنى | الحل |
|---|---|---|
| `butterfly probe, fail -ENOMEM` | فشل في allocate الـ SPI host structure | تحقق من available memory، جرب `echo 3 > /proc/sys/vm/drop_caches` |
| `butterfly probe, fail -EBUSY` | الـ parport مستخدم من driver تاني | `rmmod lp` أو `rmmod ppdev` أو `fuser /dev/parport0` |
| `spi_bitbang_start failed` | فشل تسجيل الـ SPI controller | تحقق من `CONFIG_SPI_BITBANG=y/m` و `dmesg` لمزيد من التفاصيل |
| `mtd_dataflash` لا يظهر | الـ AVR لسه مش disabled لـ SPI | flash الـ AVR firmware اللي بيعمل `PRR.2` set وبيعطل الـ PORTB pullups |
| `parport0: PC-style at 0x378` (no IRQ) | الـ parport شغّال بدون interrupt | normal للـ butterfly — بيشتغل بـ polling |
| `spi42.1: timeout` | الـ DataFlash مش بيرد | تحقق من الـ wiring، الـ VCC (pins 7,8)، و الـ chip select (pin 17) |
| `mtd_dataflash: unrecognized JEDEC id` | الـ chip مش AT45 series أو wiring غلط | تحقق من MISO/MOSI معكوسين؟ تحقق من الـ CPOL/CPHA (لازم mode 0) |
| `butterfly: SPI transfer timeout` | مشكلة في الـ bitbang timing | الـ parport بطيء أو مش بيشتغل بـ SPP mode |

---

#### 8. أماكن استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في butterfly_attach() — بعد parport_claim فشل */
status = parport_claim(pd);
if (status < 0) {
    WARN_ON(status != -EBUSY); /* لو مش busy يبقى في حاجة غريبة */
    goto clean1;
}

/* في butterfly_chipselect() — تحقق إن الـ pp مش NULL */
WARN_ON(!pp);
WARN_ON(!pp->port);

/* في getmiso() — تحقق من bit polarity logic */
/* لو القيم المقروءة دايماً 0 أو دايماً 1 */

/* في butterfly_txrx_word_mode0() — تحقق من الـ bits */
WARN_ON(bits > 32);
WARN_ON(bits == 0);
```

**نمط عملي — إضافة dump_stack مؤقت للـ debugging:**

```c
/* في butterfly_attach() قبل spi_bitbang_start() */
pr_info("butterfly: about to start bitbang on port %s\n", p->name);
/* dump_stack(); */ /* uncomment مؤقتاً عشان تعرف من أين اتطلبت */
status = spi_bitbang_start(&pp->bitbang);
```

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State يطابق الـ Kernel State

الـ `spi_butterfly` بيعمل bitbang على الـ parallel port — مفيش registers حقيقية للـ SPI controller، الكل بيحصل بـ software على الـ parport pins.

```
Kernel State             Hardware State
─────────────────        ──────────────────────────────────
pp->lastbyte bit 0  ←→  parport pin 2 (SCK)
pp->lastbyte bit 7  ←→  parport pin 9 (MOSI)
PARPORT_STATUS_BUSY ←→  parport pin 11 (MISO) — inverted
PARPORT_CONTROL_SELECT←→parport pin 17 (CS/nSELECT) — inverted
pp->lastbyte bit 5,6←→  parport pins 7,8 (VCC للـ DataFlash)
butterfly_nreset    ←→  parport pin 3 (AVR nRESET)
```

**للتحقق من الـ state:**

```bash
# اقرأ الـ parport data register الحالي
cat /sys/kernel/debug/dynamic_debug/control | grep butterfly

# أو load الـ ppdev module وتحكم يدوياً
modprobe ppdev
python3 -c "
import struct, fcntl, os
PPRSTATUS = 0x40017004
PPRDATA   = 0x40017001
fd = os.open('/dev/parport0', os.O_RDWR)
# اقرأ data lines
buf = bytearray(1)
fcntl.ioctl(fd, PPRDATA, buf)
print(f'Data lines: {bin(buf[0])}')
# اقرأ status lines
fcntl.ioctl(fd, PPRSTATUS, buf)
print(f'Status: {bin(buf[0])}')
os.close(fd)
"
```

**توقعات حالة الـ hardware بعد الـ attach:**

| الـ Parport Pin | الـ Signal | الـ Expected Level | ملاحظة |
|---|---|---|---|
| 2 (D0) | SCK | Low (CPOL=0 idle) | الـ clock ساكن |
| 3 (D1) | nRESET | High | الـ AVR خارج reset |
| 7,8 (D5,D6) | VCC | High | الـ power للـ DataFlash |
| 9 (D7) | MOSI | Don't care (idle) | |
| 11 (S7) | MISO | Floating/High | input من الـ DataFlash |
| 17 (C3) | nCS | High (inactive) | chip select inactive |

---

#### 2. Register Dump Techniques

الـ parallel port registers في الـ PC architecture:

```bash
# الـ base address للـ parport0 الكلاسيكي
cat /proc/sys/dev/parport/parport0/base-addr
# عادةً: 0x378

# قراءة الـ registers يدوياً باستخدام ioperm (لو عندك access)
# Data register:   base + 0 (R/W)
# Status register: base + 1 (R only)
# Control register:base + 2 (R/W)

# باستخدام inb/outb tool أو python:
python3 -c "
import mmap, os, struct

# /dev/port بيدي access للـ I/O ports
with open('/dev/port', 'r+b') as f:
    f.seek(0x378)  # Data register
    data = struct.unpack('B', f.read(1))[0]
    print(f'Parport Data (0x378): 0x{data:02X} = {bin(data)}')

    f.seek(0x379)  # Status register
    status = struct.unpack('B', f.read(1))[0]
    print(f'Parport Status (0x379): 0x{status:02X} = {bin(status)}')

    f.seek(0x37A)  # Control register
    ctrl = struct.unpack('B', f.read(1))[0]
    print(f'Parport Control (0x37A): 0x{ctrl:02X} = {bin(ctrl)}')
"
```

**تفسير الـ Status Register (0x379):**

```
Bit 7 (nBUSY/MISO): لو 0 = MISO high (بسبب inversion في الكود)
Bit 6 (nACK):       إشارة ACK
Bit 5 (PAPEROUT):   لو بتستخدم USI bus
Bit 4 (SELECT):
Bit 3 (nERROR):
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

**نقاط القياس على كابل الـ DB-25:**

```
                    ┌─────────────────────────────────────────┐
                    │  DB-25 Female (Parport Side)             │
                    │                                          │
   SCK  ──── Pin 2  │ ▣  ○  ○  ○  ○  ○  ○  ○  ○  ○  ○  ○  ○ │
   RST  ──── Pin 3  │  ○  ○  ○  ○  ○  ○  ○  ○  ○  ○  ○  ○   │
   VCC  ──── Pin 7  │                                          │
   VCC  ──── Pin 8  │         (VCC للـ DataFlash)              │
   MOSI ──── Pin 9  │                                          │
   MISO ──── Pin 11 │      (STATUS_BUSY — inverted)            │
   CS   ──── Pin 17 │      (CONTROL_SELECT — inverted)         │
   GND  ──── Pin 23 │                                          │
   GND  ──── Pin 24 │                                          │
                    └─────────────────────────────────────────┘
```

**إعدادات الـ Logic Analyzer:**

```
Sample Rate:  >= 2 MHz (الـ bitbang بطيء — عادةً أقل من 100 KHz)
Voltage:      3.3V threshold (الـ AVR Butterfly بيشتغل على 3V)
Channels:     SCK, MOSI, MISO, CS
Protocol:     SPI Mode 0 (CPOL=0, CPHA=0)
Bit Order:    MSB first

Triggers:
  - Trigger on CS going LOW (active) → يدل على بداية transaction
  - بعدها capture SCK و MOSI و MISO
```

**الـ DataFlash Commands اللي المفروض تشوفها:**

```
0x9F → Read Manufacturer ID  (JEDEC)
       Response: 0x1F, 0x24, 0x00 (Atmel AT45DB041B)
0xD7 → Read Status Register
       Bit 7: 1 = ready, 0 = busy
0x52 → Continuous Array Read (low frequency)
```

**Oscilloscope checks:**

```
- VCC على pins 7,8: لازم تقيس 3.3V ± 10% بعد الـ attach
- SCK idle: لازم يكون Low (CPOL=0)
- CS idle: لازم يكون High (inactive, لأن الـ control bit inverted)
- Rise/Fall time للـ SCK: < 100ns عشان مفيش signal integrity issues
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماط الـ Kernel Log

| مشكلة الـ Hardware | ما اللي بيظهر في الـ dmesg | الحل |
|---|---|---|
| الـ MOSI و MISO معكوسين في الكابل | الـ device بيتعرف بس كل read يرجع garbage | اعكس الـ wires بين pin 9 و pin 11 |
| الـ VCC مش كفاية (pins 7,8 load عالي) | الـ DataFlash بييجي ready ثم يـ timeout | أضف external 3.3V supply لـ J400 |
| الـ parport بيشتغل في ECP mode مش SPP | بعض الـ bits مش بتتكتب صح | ادخل BIOS وغيّر الـ parport mode لـ SPP أو Normal |
| الـ AVR فيه firmware قديم (SPI enabled) | الـ DataFlash مش بيشتغل مع الـ Linux | flash the AVR with firmware that sets `PRR.2` |
| كابل parport طويل جداً (> 2m) | intermittent read errors، MISO unstable | استخدم كابل أقصر، أو أضف 100Ω series resistors على الـ SCK و MOSI |
| الـ CS (pin 17) مش محتاج external pull | الـ DataFlash بيتأخر في response | الـ PARPORT_CONTROL_SELECT inverted — اتحقق إن الـ parport control register يكتب صح |

---

#### 5. Device Tree Debugging

الـ `spi_butterfly` **مش بيستخدم Device Tree** — هو بيعتمد على الـ parport detection التلقائي عبر `parport_register_driver`. مفيش DT node مطلوب.

لو بتحاول تشغّله على embedded platform بتستخدم DT:

```bash
# تحقق إن الـ parport موجود في الـ DT أو detected
dmesg | grep -i parport
# لازم تلاقي: "parport0: PC-style at 0x378"

# لو بتستخدم GPIO bitbang SPI بدل parport على embedded board
# الـ DT node المقابل بيكون:
# spi-gpio {
#     compatible = "spi-gpio";
#     sck-gpios = <&gpio0 2 GPIO_ACTIVE_HIGH>;
#     mosi-gpios = <&gpio0 9 GPIO_ACTIVE_HIGH>;
#     miso-gpios = <&gpio0 11 GPIO_ACTIVE_HIGH>;
#     cs-gpios = <&gpio0 17 GPIO_ACTIVE_LOW>;
#     num-chipselects = <2>;
#     ...
# };

# تحقق من DT compiled و loaded
dtc -I fs /proc/device-tree 2>/dev/null | grep -A5 "spi-gpio"

# أو باستخدام fdtdump
dtc -I dtb -O dts /boot/dtb/your-board.dtb | grep -A10 spi
```

---

### Practical Commands

#### اختبار شامل خطوة بخطوة

```bash
#!/bin/bash
# butterfly_debug.sh — سكريبت debugging شامل

echo "=== Step 1: Check modules ==="
lsmod | grep -E "spi_butterfly|spi_bitbang|mtd_dataflash|parport_pc|parport"

echo ""
echo "=== Step 2: Load modules if needed ==="
modprobe parport_pc io=0x378 irq=7    # adjust to your system
modprobe spi_butterfly
modprobe mtd_dataflash                 # لو مش module dependency

echo ""
echo "=== Step 3: Check parport detection ==="
dmesg | tail -30 | grep -E "parport|butterfly|spi|mtd"

echo ""
echo "=== Step 4: Check SPI bus ==="
ls /sys/class/spi_master/
# لازم تلاقي spi42

echo ""
echo "=== Step 5: Check SPI devices ==="
ls /sys/bus/spi/devices/
cat /sys/bus/spi/devices/spi42.1/modalias 2>/dev/null || echo "No spi42.1 device!"

echo ""
echo "=== Step 6: Check MTD ==="
cat /proc/mtd
mtdinfo -a 2>/dev/null || cat /sys/class/mtd/mtd0/name 2>/dev/null || echo "No MTD device!"

echo ""
echo "=== Step 7: Read DataFlash status ==="
dd if=/dev/mtd0 bs=264 count=1 2>&1 | head -5
```

#### تفعيل الـ dynamic debug والمراقبة

```bash
# تفعيل كل الـ debug messages
echo "module spi_butterfly +pflmt" > /sys/kernel/debug/dynamic_debug/control
echo "module spi_bitbang +pflmt" > /sys/kernel/debug/dynamic_debug/control

# متابعة الـ dmesg في real time
dmesg -w &

# إعادة تحميل الـ driver
rmmod spi_butterfly && modprobe spi_butterfly

# إيقاف المتابعة
kill %1
```

#### فحص الـ SPI transfer باستخدام spidev (للاختبار)

```bash
# لو عندك spidev device مضاف
# أضف spidev device على bus 42:
# echo spidev 42.0 > /sys/bus/spi/devices/spi42.0/../new_device

# اختبار transfer بسيط
python3 -c "
import spidev
spi = spidev.SpiDev()
spi.open(42, 1)  # bus 42, cs 1 (dataflash)
spi.max_speed_hz = 1000000  # 1 MHz — آمن للـ bitbang
spi.mode = 0  # SPI_MODE_0

# Read DataFlash JEDEC ID
response = spi.xfer2([0x9F, 0x00, 0x00, 0x00])
print(f'JEDEC ID: {[hex(b) for b in response]}')
# Expected: [0x00, 0x1F, 0x24, 0x00] → Atmel AT45DB041B
spi.close()
"
```

#### تفسير نتائج الـ AT45DB041B JEDEC ID

```
0x1F → Atmel manufacturer ID
0x24 → AT45DB041B device ID (density code)
0x00 → extended device info

لو الـ response كله 0xFF: MISO مش متوصل أو الـ chip مش بياخد power
لو الـ response كله 0x00: MOSI/MISO معكوسين، أو الـ chip في reset
لو response غير متوقع: تحقق من الـ SPI mode (لازم mode 0)
```

#### اختبار Read/Write للـ DataFlash

```bash
# قراءة أول 264 bytes (page 0)
dd if=/dev/mtd0 of=/tmp/butterfly_page0.bin bs=264 count=1
hexdump -C /tmp/butterfly_page0.bin

# كتابة pattern بسيط للاختبار
python3 -c "import sys; sys.stdout.buffer.write(b'\xAA\x55' * 132)" > /tmp/test_pattern.bin
dd if=/tmp/test_pattern.bin of=/dev/mtd0 bs=264 count=1

# قراءة تاني والمقارنة
dd if=/dev/mtd0 of=/tmp/butterfly_verify.bin bs=264 count=1
diff /tmp/test_pattern.bin /tmp/butterfly_verify.bin && echo "PASS: Read/Write OK" || echo "FAIL: Mismatch!"

# erase الـ partition الأول قبل الـ test
flash_eraseall -j /dev/mtd0  # -j للـ JFFS2 format
```

#### مراقبة الـ SPI events بالـ ftrace

```bash
# إعداد ftrace كامل
cd /sys/kernel/debug/tracing

# صفّر أي trace قديم
echo "" > trace
echo nop > current_tracer

# فعّل SPI events
echo 1 > events/spi/enable

# ابدأ الـ capture
echo 1 > tracing_on

# نفّذ عملية على الـ DataFlash
dd if=/dev/mtd0 bs=264 count=4 of=/dev/null 2>&1

# وقّف الـ capture
echo 0 > tracing_on

# اقرأ النتيجة
cat trace | grep spi | head -40

# مثال على output متوقع:
# dd-1234 [000] d... 456.789: spi_message_start: spi42.1
# dd-1234 [000] d... 456.790: spi_transfer_start: spi42.1 len=4 tx=[9f 00 00 00] rx=[]
# dd-1234 [000] d... 456.791: spi_transfer_stop: spi42.1 len=4 tx=[] rx=[00 1f 24 00]
# dd-1234 [000] d... 456.791: spi_message_done: spi42.1 status=0
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: bring-up لـ DataFlash على industrial gateway بـ parallel port

#### العنوان
**فشل تهيئة `mtd_dataflash` على gateway صناعي قديم بـ parport**

#### السياق
شركة تعمل على نظام SCADA قديم على لوحة x86 صناعية تشغّل Linux مدمج. اللوحة فيها parallel port حقيقي (DB-25) ومتصل بها AVR Butterfly عبر كابل ISP محلي الصنع بالمواصفات الموجودة في الـ doc. الـ DataFlash chip هي `at45db041b` ومطلوب منها تخزين بيانات sensor بشكل دوري. الـ kernel مبني من source مع تفعيل `CONFIG_SPI_BUTTERFLY=m` و`CONFIG_MTD_DATAFLASH=m`.

#### المشكلة
بعد `modprobe spi_butterfly`، الـ kernel log يظهر:

```
parport0: AVR Butterfly
```

لكن ما في أي device تحت `/dev/mtd*` وما ظهر `butterflash` في `cat /proc/mtd`.

#### التحليل
المشكلة في الـ firmware على الـ AVR. الـ doc تقول بوضوح:

```
Then to let Linux master that bus to talk to the DataFlash chip, you must
(a) flash new firmware that disables SPI (set PRR.2, and disable pullups
    by clearing PORTB.[0-3])
```

الـ `butterfly_attach()` في `spi-butterfly.c` بتحاول تسجّل `mtd_dataflash` على bus 42:

```c
pp->info[0].max_speed_hz = 15 * 1000 * 1000;
strcpy(pp->info[0].modalias, "mtd_dataflash");
pp->info[0].platform_data = &flash;
pp->info[0].chip_select = 1;
pp->info[0].controller_data = pp;
pp->dataflash = spi_new_device(pp->bitbang.ctlr, &pp->info[0]);
```

الـ `spi_new_device()` بتنجح وبترجع device، لكن لما الـ `mtd_dataflash` driver يحاول يتكلم مع الـ chip، الـ AVR لسه شايل SCK و MOSI pins عبر الـ SPI controller الداخلي فيه ومش بيخلّيها تشتغل صح. الـ MISO بيرجع garbage لأن الـ AVR pullups على `PORTB.[0-3]` لسه مفعّلة.

ثانياً، الـ `getmiso()` في الـ driver:

```c
static inline int getmiso(struct spi_device *spi)
{
    struct butterfly *pp = spidev_to_pp(spi);
    int value;
    u8 bit;

    bit = spi_miso_bit;  /* PARPORT_STATUS_BUSY, pin 11 */

    /* only STATUS_BUSY is NOT negated */
    value = !(parport_read_status(pp->port) & bit);
    return (bit == PARPORT_STATUS_BUSY) ? value : !value;
}
```

الـ `PARPORT_STATUS_BUSY` على pin 11 بيرجع inverted value. لو الـ AVR لسه مسيطر على الخط، الـ MISO data هتبقى undefined.

#### الحل

**الخطوة الأولى**: فلاش الـ AVR بـ firmware جديد يعمل:
```c
/* في AVR firmware */
PRR |= (1 << PRSPI);           /* تعطيل SPI hardware */
PORTB &= ~((1<<0)|(1<<1)|(1<<2)|(1<<3)); /* تعطيل pullups */
DDRB  &= ~((1<<1)|(1<<2)|(1<<3));        /* pins as input */
```

**الخطوة الثانية**: التأكد من توصيل chipselect لـ DataFlash:
```
pin 17/C3,nSELECT  -->  J400.PB0/nSS
pin 7/D5           -->  J400.VCC_EXT
```

**الخطوة الثالثة**: بعد `modprobe`:
```bash
dmesg | grep butterfl
cat /proc/mtd
hexdump -C /dev/mtd0 | head -4
```

#### الدرس المستفاد
الـ `spi_butterfly` driver ما بيعمل hardware probe حقيقي — كما في comment الكود:
```c
/* REVISIT:  this just _assumes_ a butterfly is there ... no probe */
```
لازم الـ firmware على الـ AVR يكون جاهز قبل ما تحمّل الـ module، وإلا الـ SPI transactions هتفشل صامتة.

---

### السيناريو الثاني: تعارض الـ parport مع driver تاني

#### العنوان
**`parport_claim()` بيرجع `-EBUSY` ومش قادر أشغّل `spi_butterfly`**

#### السياق
مطوّر على جهاز x86 desktop عنده parallel port بيستخدمه أحياناً مع printer قديم عبر `lp` module، وأحياناً مع AVR Butterfly للتطوير. حاول يحمّل الـ `spi_butterfly` module لكن فشل.

#### المشكلة
```
dmesg:
parport0: spi_butterfly: cannot claim parport
```

الـ `butterfly_attach()` بتفشل عند:

```c
status = parport_claim(pd);
if (status < 0)
    goto clean1;
```

#### التحليل
الـ `parport_claim()` بترجع `-EBUSY` لأن device تاني حاجز الـ parport. لما تشوف:

```bash
cat /proc/sys/dev/parport/parport0/devices
```

بتلاقي `lp0` أو `ppdev` شايل الـ port. الـ `butterfly_attach()` بتتسمّى تلقائي من الـ parport subsystem:

```c
static struct parport_driver butterfly_driver = {
    .name =      "spi_butterfly",
    .match_port = butterfly_attach,   /* called on parport registration */
    .detach =     butterfly_detach,
};
module_parport_driver(butterfly_driver);
```

يعني بمجرد `insmod spi_butterfly.ko`، الـ `butterfly_attach()` بتتنادى فوراً على كل parport موجود. لو الـ `lp` module محمّل، الـ port مش free.

الـ cleanup path صح في الـ driver لأنه بيعمل `goto clean1` اللي بيعمل:

```c
clean1:
    parport_unregister_device(pd);
clean0:
    spi_controller_put(host);
done:
    pr_debug("%s: butterfly probe, fail %d\n", p->name, status);
```

فما في memory leak، بس الـ module بيلوّد من غير ما يشتغل.

#### الحل

```bash
# تحرير الـ parport أولاً
rmmod lp
rmmod ppdev

# أو لو CUPS شغّال
sudo systemctl stop cups

# بعدين
modprobe spi_butterfly

# للتحقق
dmesg | tail -5
ls /dev/mtd*
```

لو عايز تشغّلهم مع بعض، استخدم `parport_yield()` لكن الـ driver الحالي ما بيدعم sharing.

#### الدرس المستفاد
الـ `parport_claim()` exclusive — ما ينفع تشارك الـ parport بين `spi_butterfly` وأي device تاني. خطط لإيقاف كل parport consumers قبل تحميل الـ module.

---

### السيناريو الثالث: استخدام USI bus للتواصل مع AVR كـ SPI slave

#### العنوان
**تطوير protocol driver مخصص للتحكم في IoT sensor node عبر AVR كـ SPI slave**

#### السياق
شركة IoT عندها sensor node مبني على AVR Butterfly. الـ node بيقيس temperature وhumidity وبيخزّنهم محلياً. المطلوب ربط الـ host Linux (x86 embedded) بالـ AVR عبر الـ USI interface (J405) ليقرأ البيانات بشكل دوري، مع إبقاء الـ DataFlash تحت سيطرة الـ AVR.

#### المشكلة
المهندس قرأ الـ doc وحاول يستخدم نفس الـ `spi_butterfly` driver ليتكلم مع الـ AVR كـ slave، لكن الـ `spi_new_device()` للـ AVR device ما نجح، والـ driver ما عنده modalias مناسب.

#### التحليل
الـ doc بتشرح:

```
Or you could flash firmware making the AVR into an SPI slave (keeping the
DataFlash in reset) and tweak the spi_butterfly driver to make it bind to
the driver for your custom SPI-based protocol.
```

وللـ USI bus:
```
SCK   J403.PE4/USCK  pin 5/D3
MOSI  J403.PE5/DI    pin 6/D4
MISO  J403.PE6/DO    pin 12/S5,nPAPEROUT
```

الـ `butterfly_attach()` الحالي بيسجّل device واحد فقط بـ modalias `"mtd_dataflash"`:

```c
pp->info[0].max_speed_hz = 15 * 1000 * 1000;
strcpy(pp->info[0].modalias, "mtd_dataflash");
```

الـ `pp->butterfly` field موجود في الـ struct:
```c
struct butterfly {
    struct spi_bitbang  bitbang;
    struct parport      *port;
    struct pardevice    *pd;
    u8                  lastbyte;
    struct spi_device   *dataflash;
    struct spi_device   *butterfly;  /* AVR device - unused currently! */
    struct spi_board_info info[2];
};
```

لكن `pp->butterfly` ما بيتملى أبداً في الكود الحالي.

#### الحل

الـ firmware على AVR يشتغل كـ SPI slave ويخزّن readings. على جانب الـ kernel، نعدّل `butterfly_attach()`:

```c
/* بعد تسجيل dataflash */

/* تسجيل AVR كـ SPI slave device */
pp->info[1].max_speed_hz = 1 * 1000 * 1000; /* 1MHz آمن للـ USI */
strcpy(pp->info[1].modalias, "avr-iot-sensor"); /* اسم driver المخصص */
pp->info[1].chip_select = 0;
pp->info[1].controller_data = pp;
pp->butterfly = spi_new_device(pp->bitbang.ctlr, &pp->info[1]);
if (pp->butterfly)
    pr_info("%s: AVR IoT sensor at %s\n", p->name,
            dev_name(&pp->butterfly->dev));
```

ثم نكتب `avr-iot-sensor` protocol driver يستخدم `spi_read()` لقراءة البيانات.

```bash
# التحقق من تسجيل الـ devices
ls /sys/bus/spi/devices/
# spi42.0  -> dataflash
# spi42.1  -> avr-iot-sensor
```

#### الدرس المستفاد
الـ `struct butterfly` عندها placeholder لـ `pp->butterfly` لكنه ungated في الكود الأصلي. الـ driver مصمّم للتوسّع — تقدر تضيف AVR slave device بتعديل بسيط في `butterfly_attach()` مع إضافة protocol driver خاص.

---

### السيناريو الرابع: debugging غلطة MISO inversion تخلّي البيانات corrupt

#### العنوان
**قراءات DataFlash غلط بسبب سوء فهم MISO signal inversion على parport**

#### السياق
مطوّر junior يعمل على custom AVR board (مش Butterfly الأصلي) خلال bring-up. استخدم كابل parport محلي وربط MISO على pin 10 (ACK) بدل pin 11 (BUSY).

#### المشكلة
الـ `hexdump /dev/mtd0` بيطلع بيانات مقلوبة — كل bit معكوس. بعض الـ reads بينجحوا بشكل عشوائي وبعضهم بيفشلوا.

#### التحليل
الـ `getmiso()` في الـ driver مبني خصيصاً لـ `PARPORT_STATUS_BUSY` على pin 11:

```c
#define spi_miso_bit  PARPORT_STATUS_BUSY  /* pin 11 */

static inline int getmiso(struct spi_device *spi)
{
    struct butterfly *pp = spidev_to_pp(spi);
    int value;
    u8 bit;

    bit = spi_miso_bit;

    /* only STATUS_BUSY is NOT negated */
    value = !(parport_read_status(pp->port) & bit);
    return (bit == PARPORT_STATUS_BUSY) ? value : !value;
}
```

الجزء المهم هو comment `"only STATUS_BUSY is NOT negated"` — يعني بقية status bits على الـ parport بتكون inverted في hardware، لكن `STATUS_BUSY` (pin 11) هو الوحيد اللي **مش** معكوس في hardware. لذلك الكود بيعكسه يدوياً بـ `value = !()`.

لو المهندس وصّل MISO على pin 10 (ACK = `PARPORT_STATUS_ACK`)، الـ `spi_miso_bit` لازم يتغيّر، وآليه الـ inversion هتكون عكس المطلوب لأن الـ condition `(bit == PARPORT_STATUS_BUSY)` هتبقى false.

الـ `bitbang_txrx_be_cpha0()` اللي بتستخدم `getmiso()`:

```c
static u32
butterfly_txrx_word_mode0(struct spi_device *spi, unsigned nsecs, u32 word,
                          u8 bits, unsigned flags)
{
    return bitbang_txrx_be_cpha0(spi, nsecs, 0, flags, word, bits);
}
```

كل bit مقروء بيتبنى عليه الـ word كاملاً — أي inversion هتخلّي الـ data كله غلط.

#### الحل

**الخطوة الأولى**: التحقق من التوصيل بـ oscilloscope أو logic analyzer:
```bash
# قبل أي شيء، تأكد من pin mapping
# pin 11 = nBUSY = MISO (المطلوب في الـ doc)
# pin 10 = nACK  = IRQ  (للـ USI bus فقط)
```

**الخطوة الثانية**: لو الكابل صح، تحقق من الـ parport mode:
```bash
# تأكد إن الـ parport في SPP mode
cat /proc/sys/dev/parport/parport0/modes
# المطلوب: SPP أو ECP في compatibility mode
```

**الخطوة الثالثة**: إضافة debug بسيط:
```c
/* في getmiso() مؤقتاً */
int raw = parport_read_status(pp->port);
pr_debug("status raw=0x%02x, bit=0x%02x, result=%d\n",
         raw, bit, value);
```

```bash
echo 8 > /proc/sys/kernel/printk
modprobe spi_butterfly
dmesg | grep "status raw"
```

#### الدرس المستفاد
الـ parport status pins لها inversion quirks مختلفة. الـ doc تحدد pin 11 (nBUSY) تحديداً كـ MISO لأنه الوحيد في status register اللي **مش** inverted في hardware. تغيير الـ pin يستلزم تعديل منطق الـ inversion في `getmiso()`.

---

### السيناريو الخامس: إعادة فلاش firmware على AVR Butterfly لتطوير custom bootloader

#### العنوان
**استخدام `spi_butterfly` + `mtd_dataflash` لتحديث firmware على AVR عبر Linux host**

#### السياق
فريق R&D يعمل على custom bootloader لـ AVR Butterfly في منتج industrial وحتاجوا آلية لتحديث الـ firmware remotely عبر Linux host بدون AVR programmer مخصص. الفكرة: Linux يكتب الـ firmware الجديد على DataFlash، ثم AVR bootloader يقرأه من DataFlash ويفلاش نفسه.

#### المشكلة
بعد كتابة الـ firmware على DataFlash عبر `/dev/mtd1` (partition "filesystem")، الـ AVR ما بيعرف يبوت من DataFlash. الـ partition layout في الـ driver ما بيتوافق مع توقعات الـ bootloader.

#### التحليل
الـ driver بيعرّف partitions ثابتة:

```c
static struct mtd_partition partitions[] = { {
    /* sector 0 = 8 pages * 264 bytes/page (1 block)
     * sector 1 = 248 pages * 264 bytes/page
     */
    .name   = "bookkeeping",    /* 66 KB */
    .offset = 0,
    .size   = (8 + 248) * 264,
}, {
    /* sector 2 = 256 pages * 264 bytes/page
     * sectors 3-5 = 512 pages * 264 bytes/page
     */
    .name   = "filesystem",     /* 462 KB */
    .offset = MTDPART_OFS_APPEND,
    .size   = MTDPART_SIZ_FULL,
} };
```

الـ `at45db041b` عنده 4 Mbit (512 KB) مقسّمة على pages بحجم 264 bytes. الـ bootloader على AVR متوقع يلاقي الـ firmware image من بداية الـ DataFlash (offset 0)، لكن الـ driver حاطط "bookkeeping" في الـ 66 KB الأولى.

علاوة على ذلك، الـ flash platform data:
```c
static struct flash_platform_data flash = {
    .name   = "butterflash",
    .parts  = partitions,
    .nr_parts = ARRAY_SIZE(partitions),
};
```

الـ `mtd_dataflash` driver بيستخدم هذا لتهيئة الـ partitions. لتغيير الـ layout، لازم تعدّل الـ static partitions أو تستخدم kernel cmdline partitions.

#### الحل

**الخطوة الأولى**: استخدام `cmdlinepart` لـ override الـ partitions عبر kernel cmdline:

```bash
# في /boot/grub/grub.cfg أو kernel cmdline
mtdparts=butterflash:128k(bootloader),-(firmware)
```

```bash
# التحقق
cat /proc/mtd
# mtd0: 00020000 00001000 "bootloader"
# mtd1: 00060000 00001000 "firmware"
```

**الخطوة الثانية**: كتابة الـ firmware image:

```bash
# تأكد إن الـ AVR في reset أثناء الكتابة
# pin 3/D1 = nRESET

flash_erase /dev/mtd1 0 0
nandwrite -p /dev/mtd1 firmware.bin
```

**الخطوة الثالثة**: بعد الكتابة، حرّر الـ reset:

```bash
# الـ butterfly_attach() بيعمل ده تلقائياً:
# pp->lastbyte |= butterfly_nreset;
# parport_write_data(pp->port, pp->lastbyte);
# msleep(100);

# لو محتاج manual reset:
# rmmod spi_butterfly && modprobe spi_butterfly
```

**الخطوة الرابعة**: التحقق من الكتابة:

```bash
md5sum firmware.bin
dd if=/dev/mtd1 bs=1 count=$(stat -c%s firmware.bin) | md5sum
```

#### الدرس المستفاد
الـ partition layout في `spi-butterfly.c` مصمم لـ JFFS2 filesystem وليس لـ firmware flashing مباشر. استخدم `cmdlinepart` لـ override الـ layout حسب احتياجات bootloader الخاص بك، مع الأخذ بعين الاعتبار إن الـ page size على `at45db041b` هو 264 bytes (وليس 256) وده بيأثر على حسابات الـ offset.
## Phase 7: مصادر ومراجع

### مصادر رسمية — kernel.org

| المصدر | الرابط |
|--------|--------|
| Official kernel doc — spi_butterfly | [kernel.org/doc/html/v5.18/spi/butterfly.html](https://www.kernel.org/doc/html/v5.18/spi/butterfly.html) |
| Official kernel doc — SPI summary | [docs.kernel.org/driver-api/spi.html](https://docs.kernel.org/driver-api/spi.html) |
| Official kernel doc — SPI overview (6.3) | [docs.kernel.org/6.3/spi/spi-summary.html](https://docs.kernel.org/6.3/spi/spi-summary.html) |
| Raw documentation file | [kernel.org/doc/Documentation/spi/butterfly](https://www.kernel.org/doc/Documentation/spi/butterfly) |
| SPI userspace API (spidev) | [static.lwn.net/kerneldoc/spi/spidev.html](https://static.lwn.net/kerneldoc/spi/spidev.html) |

الـ `Documentation/spi/butterfly.rst` في kernel source هو نقطة البداية لفهم الـ hardware wiring بين الـ parallel port والـ AVR Butterfly.

---

### مقالات LWN.net

الـ [LWN.net](https://lwn.net) هو المرجع الأهم لتتبع تطور الـ SPI subsystem من الأول:

| المقال | الرابط | الأهمية |
|--------|--------|---------|
| **Simple SPI framework** — أول تقديم للـ SPI core من David Brownell | [lwn.net/Articles/154425](https://lwn.net/Articles/154425/) | مهم جداً — الأصل |
| **SPI core** — patch announcement | [lwn.net/Articles/138037](https://lwn.net/Articles/138037/) | تاريخي — أول commit |
| **SPI core refresh** — تحديثات على الـ framework | [lwn.net/Articles/162268](https://lwn.net/Articles/162268/) | تطور الـ API |
| **Add SPI over GPIO driver** — يذكر SPI_BUTTERFLY في Kconfig | [lwn.net/Articles/290068](https://lwn.net/Articles/290068/) | سياق bitbang drivers |
| **Introduction to SPI NAND framework** | [lwn.net/Articles/715120](https://lwn.net/Articles/715120/) | توسع الـ SPI ecosystem |
| **SPI slave mode support** | [lwn.net/Articles/723440](https://lwn.net/Articles/723440/) | slave mode — مفيد للفهم |
| **SPI kernel documentation** على LWN mirror | [static.lwn.net/kerneldoc/driver-api/spi.html](https://static.lwn.net/kerneldoc/driver-api/spi.html) | مرجع سريع |

> **ملاحظة**: David Brownell هو كاتب الـ SPI framework الأصلي وصاحب `spi_butterfly` driver. الـ LWN نشر obituary له في [lwn.net/Articles/437156](https://lwn.net/Articles/437156/) بعد وفاته — وهو مرجع لفهم مساهماته الضخمة في kernel.

---

### Linux Kernel Driver Database

**الـ `CONFIG_SPI_BUTTERFLY`** موثق في LKDDb مع كل الـ kernel versions اللي ظهر فيها:

- [cateee.net/lkddb/web-lkddb/SPI_BUTTERFLY.html](https://cateee.net/lkddb/web-lkddb/SPI_BUTTERFLY.html) — يغطي kernels 2.6.16 → 6.6-rc
- [cateee.net/lkddb/web-lkddb/SPI_BITBANG.html](https://cateee.net/lkddb/web-lkddb/SPI_BITBANG.html) — الـ `CONFIG_SPI_BITBANG` اللي يعتمد عليه الـ butterfly driver

---

### ملفات kernel source ذات الصلة

```
Documentation/spi/butterfly.rst          ← الـ doc الرئيسي
Documentation/spi/spi-summary.rst        ← overview للـ SPI subsystem
Documentation/spi/spidev.rst             ← userspace API
drivers/spi/spi-butterfly.c              ← الـ driver نفسه
drivers/spi/spi-bitbang.c               ← الـ bitbang framework
include/linux/spi/spi.h                 ← الـ SPI API headers
include/linux/spi/spi_bitbang.h         ← bitbang helpers
```

الـ `drivers/spi/spi-butterfly.c` هو الملف الرئيسي — يستخدم `spi_bitbang` framework ويعرّف `parport_driver` لاستقبال الـ parallel port events.

---

### Kernel Newbies

صفحات kernelnewbies.org مفيدة لتتبع تغييرات الـ SPI عبر الإصدارات:

| الصفحة | الرابط |
|--------|--------|
| Linux_2_6_24 — أول دعم رسمي لـ SPI مستقر | [kernelnewbies.org/Linux_2_6_24](https://kernelnewbies.org/Linux_2_6_24) |
| Linux_3.15-DriversArch — SPI driver changes | [kernelnewbies.org/Linux_3.15-DriversArch](https://kernelnewbies.org/Linux_3.15-DriversArch) |
| Linux_4.18 — SPI updates | [kernelnewbies.org/Linux_4.18](https://kernelnewbies.org/Linux_4.18) |
| Linux_6.5 — أحدث SPI controller enhancements | [kernelnewbies.org/Linux_6.5](https://kernelnewbies.org/Linux_6.5) |
| FirstKernelPatch — guide للمبتدئين | [kernelnewbies.org/FirstKernelPatch](https://kernelnewbies.org/FirstKernelPatch) |

---

### eLinux.org

الـ elinux.org ما فيهوش صفحة مخصصة للـ butterfly، بس فيه مراجع مفيدة للـ SPI على embedded boards:

| الصفحة | الرابط |
|--------|--------|
| RPi SPI — SPI على Raspberry Pi | [elinux.org/RPi_SPI](https://elinux.org/RPi_SPI) |
| BeagleBoard Zippy — SPI/I2C expansion | [elinux.org/BeagleBoard_Zippy](https://elinux.org/BeagleBoard_Zippy) |

---

### مراجع خارجية إضافية

| المصدر | الرابط |
|--------|--------|
| Microchip DevHelp — Linux SPI Applications | [developerhelp.microchip.com/…/apps-spi](https://developerhelp.microchip.com/xwiki/bin/view/software-tools/linux/apps-spi/) |
| mjmwired kernel doc mirror | [mjmwired.net/kernel/Documentation/spi/butterfly](https://mjmwired.net/kernel/Documentation/spi/butterfly) |
| GitHub mirror — spotify/linux | [github.com/spotify/linux/…/butterfly](https://github.com/spotify/linux/blob/master/Documentation/spi/butterfly) |

---

### كتب مرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل الأهم**: Chapter 6 — "Advanced Char Driver Operations" و Chapter 18 — مشابه للـ bus drivers
- الكتاب مجاني على: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)
- **ملاحظة**: LDD3 صدر قبل الـ SPI framework — لذا مفيش فصل مخصص للـ SPI، بس الـ bus/driver model المشروح فيه هو الأساس

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل المفيد**: Chapter 17 — "Devices and Modules"
- يشرح الـ `platform_driver`, `bus_type`, `device_driver` اللي الـ SPI يبني عليها
- الـ bitbang concept مرتبط بـ GPIO handling المشروح في نفس الفصل

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل المفيد**: Chapter 8 — "Device Driver Basics" وChapter 14 — "Kernel Initialization"
- يشرح إزاي الـ SPI و I2C بيتعاملوا في بيئة embedded
- فيه أمثلة على AVR و Atmel processors اللي الـ Butterfly منها

#### Linux System Programming — Robert Love
- مفيد لفهم الـ `parport` interface من ناحية userspace

---

### مصطلحات بحث مفيدة

للباحث عن معلومات أكتر عن الـ `spi_butterfly` وسياقه:

```
# للبحث في kernel mailing list (lore.kernel.org)
spi_butterfly parport bitbang
CONFIG_SPI_BUTTERFLY David Brownell
"AVR Butterfly" SPI linux driver

# للبحث في kernel source
git log --oneline -- drivers/spi/spi-butterfly.c
git log --oneline -- drivers/spi/spi_butterfly.c

# للبحث في LWN
site:lwn.net "SPI butterfly"
site:lwn.net "David Brownell" SPI

# للبحث العام
linux kernel parport SPI bitbang driver
AVR Butterfly linux programming ISP SPI
spi_bitbang framework linux kernel example
```

---

### روابط سريعة — ملخص

```
┌─────────────────────────────────────────────────────────────────┐
│  المرجع                         │  النوع                        │
├─────────────────────────────────────────────────────────────────┤
│  kernel.org/doc/…/butterfly     │  Official doc                 │
│  lwn.net/Articles/154425        │  SPI framework introduction    │
│  lwn.net/Articles/138037        │  SPI core first patch          │
│  lwn.net/Articles/162268        │  SPI core refresh              │
│  cateee.net/…/SPI_BUTTERFLY     │  Kconfig history               │
│  kernelnewbies.org/Linux_2_6_24 │  First stable SPI support      │
│  elinux.org/RPi_SPI             │  Practical SPI on embedded     │
│  docs.kernel.org/driver-api/spi │  Current SPI API reference     │
└─────────────────────────────────────────────────────────────────┘
```
## Phase 8: Writing simple module

### الفكرة

**الـ** `spi_butterfly` driver بيشتغل فوق SPI bus عادي — وأحسن حاجة نعمل module يراقب أي SPI message بيتبعت على الـ bus باستخدام **tracepoint** جاهز في الـ kernel اسمه `spi_message_done`. ده tracepoint بيتفعّل كل ما transfer خلص، وبيكشف معلومات زي رقم الـ bus، الـ chip select، وعدد البايتات اللي اتبعتت فعلاً.

الـ tracepoint ده معرّف في `include/trace/events/spi.h` ومتاح لأي module بشرط `MODULE_LICENSE("GPL")`.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * spi_spy.c — trace every completed SPI message on any bus.
 * Hooks the spi_message_done tracepoint, which fires after each
 * spi_message finishes — exactly what the butterfly driver triggers
 * when it does a bitbang transfer over the parallel port.
 */

/* Core kernel headers always needed in any module */
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>

/*
 * We need the SPI types (spi_message, spi_device, spi_controller)
 * before pulling in the tracepoint header, so the compiler knows
 * what fields exist inside those structs.
 */
#include <linux/spi/spi.h>

/*
 * CREATE_TRACE_POINTS must be defined in exactly ONE .c file in the
 * module; it tells the preprocessor to emit the actual tracepoint
 * function bodies (not just declarations).
 */
#define CREATE_TRACE_POINTS
#include <trace/events/spi.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("SPI Spy <spy@example.com>");
MODULE_DESCRIPTION("Tracepoint hook on spi_message_done — monitors every completed SPI transfer");

/* ------------------------------------------------------------------ */
/*  Callback — runs every time a spi_message finishes                  */
/* ------------------------------------------------------------------ */

/*
 * The prototype must match exactly what TRACE_EVENT(spi_message_done)
 * declares: TP_PROTO(struct spi_message *msg).
 * We receive a pointer to the finished message object.
 */
static void spi_spy_message_done(void *ignore, struct spi_message *msg)
{
    struct spi_device     *spi  = msg->spi;          /* device that owns the message   */
    struct spi_controller *ctrl = spi->controller;   /* the host controller behind it  */

    /*
     * spi_get_chipselect() is the official accessor for the chip-select
     * index (index 0 = primary CS).  Avoids direct struct access which
     * can break across kernel versions.
     */
    int cs = spi_get_chipselect(spi, 0);

    /* Print bus number, chip-select, and how many bytes actually moved */
    pr_info("spi_spy: spi%d.%d — done: actual=%u bytes / frame=%u bytes\n",
            ctrl->bus_num,
            cs,
            msg->actual_length,
            msg->frame_length);
}

/* ------------------------------------------------------------------ */
/*  module_init — register the tracepoint probe                        */
/* ------------------------------------------------------------------ */

static int __init spi_spy_init(void)
{
    int ret;

    /*
     * register_trace_spi_message_done() is auto-generated from the
     * DEFINE_EVENT macro in trace/events/spi.h.
     * First arg  = our callback function.
     * Second arg = opaque data passed as "ignore" in the callback
     *              (NULL because we don't need it).
     */
    ret = register_trace_spi_message_done(spi_spy_message_done, NULL);
    if (ret) {
        pr_err("spi_spy: failed to register tracepoint: %d\n", ret);
        return ret;
    }

    pr_info("spi_spy: loaded — watching spi_message_done\n");
    return 0;
}

/* ------------------------------------------------------------------ */
/*  module_exit — MUST unregister or kernel will call freed code       */
/* ------------------------------------------------------------------ */

static void __exit spi_spy_exit(void)
{
    /*
     * unregister_trace_spi_message_done() removes our callback from
     * the tracepoint's probe list.  Without this, if the module is
     * unloaded while the tracepoint is still live, the kernel will
     * call a function pointer that no longer exists → kernel panic.
     *
     * tracepoint_synchronize_unregister() waits until all CPUs that
     * were already inside the tracepoint finish — prevents use-after-free.
     */
    unregister_trace_spi_message_done(spi_spy_message_done, NULL);
    tracepoint_synchronize_unregister();

    pr_info("spi_spy: unloaded\n");
}

module_init(spi_spy_init);
module_exit(spi_spy_exit);
```

---

### شرح كل جزء

#### الـ Includes

- **الـ** `linux/spi/spi.h` لازم يجي **قبل** `trace/events/spi.h` عشان الـ tracepoint macros بتشتغل على `struct spi_message` و`struct spi_controller` — لو ما عرفهمش الـ compiler هيشتكي.
- **الـ** `CREATE_TRACE_POINTS` بيخلي الـ preprocessor يولّد الـ function bodies الفعلية للـ tracepoints بدل ما يبقوا just declarations — لازم يتعرف في ملف `.c` واحد بس في الـ module.

#### الـ Callback `spi_spy_message_done`

- بياخد `struct spi_message *msg` — ده الـ message اللي خلّص.
- منه بنعرف الـ `spi_device` (مين الـ chip) والـ `spi_controller` (مين الـ bus master زي الـ parport bitbanger في butterfly).
- **الـ** `actual_length` بيكشف كام byte اتنقل فعلاً، مقابل `frame_length` اللي هو الـ المطلوب — الفرق بيدل على error.

#### الـ `module_init`

- بيسجّل الـ callback على الـ tracepoint باستخدام `register_trace_spi_message_done()` — الاسم ده auto-generated من اسم الـ tracepoint.
- لو الـ registration فشل (مثلاً الـ tracepoint مش compiled في الـ kernel) بيرجع الـ error بدل ما يفضل صامت.

#### الـ `module_exit`

- **الـ** `unregister_trace_spi_message_done()` بيشيل الـ callback من الـ probe list — ده إجباري عشان لو الـ module اتـ unload والـ tracepoint لسه فاعل هيحصل kernel panic.
- **الـ** `tracepoint_synchronize_unregister()` بيضمن إن كل الـ CPUs اللي كانت جوا الـ tracepoint خلّصت قبل ما نكمل — ده RCU-based synchronization يمنع الـ use-after-free.

---

### الـ Makefile وتشغيل الـ Module

```makefile
# Makefile
obj-m += spi_spy.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

```bash
# بناء وتحميل الـ module
make
sudo insmod spi_spy.ko

# شوف الـ output لما أي SPI transfer يحصل
sudo dmesg -w | grep spi_spy

# فك التحميل
sudo rmmod spi_spy
```

---

### مثال على الـ Output

لما الـ `spi_butterfly` driver يبعت SPI message على الـ parport bus:

```
[  142.331201] spi_spy: loaded — watching spi_message_done
[  142.889045] spi_spy: spi0.0 — done: actual=4 bytes / frame=4 bytes
[  142.889312] spi_spy: spi0.0 — done: actual=256 bytes / frame=256 bytes
[  145.001100] spi_spy: unloaded
```

**الـ** `spi0.0` يعني الـ bus الأول، الـ chip select الأول — وده بالظبط إيه اللي بيعمله الـ butterfly عبر الـ J403 connector.
