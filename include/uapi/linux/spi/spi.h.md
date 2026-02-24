## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الـ file ده جزء من **SPI Subsystem** في الـ Linux kernel، والـ maintainer بتاعه هو Mark Brown. الـ subsystem ده بيغطي كل حاجة بتتعلق بـ **Serial Peripheral Interface (SPI)** — من الـ controller drivers، للـ device drivers، للـ userspace API.

---

### إيه هو الـ SPI؟

تخيل إنك عندك Arduino أو Raspberry Pi وعايز تكلم شريحة flash memory أو sensor أو DAC. الـ **SPI** هو بروتوكول تواصل serial سريع بيستخدم 4 أسلاك أساسية:

```
Master (CPU)          Slave (Device)
   ┌────────┐             ┌────────┐
   │  SCLK  │────────────►│  SCLK  │   clock
   │  MOSI  │────────────►│  MOSI  │   Master Out Slave In
   │  MISO  │◄────────────│  MISO  │   Master In Slave Out
   │  CS/SS │────────────►│  CS    │   Chip Select
   └────────┘             └────────┘
```

الـ Master هو اللي بيتحكم في الـ clock وبيختار الـ slave عن طريق **Chip Select (CS)** — لما CS بيبقى active (عادةً low)، الـ slave ده بيسمع ويرد. التواصل **full-duplex** يعني بتبعت وبتستقبل في نفس الوقت.

---

### الـ UAPI Layer — ليه موجود؟

الـ Linux kernel منقسم لجزأين:
- **Kernel space**: الكود اللي بيشتغل جوا الـ kernel
- **User space**: البرامج العادية اللي بتشغلها

الـ `uapi/` prefix معناها **User API** — يعني الـ definitions دي مشتركة بين الاتنين. البرنامج العادي والـ kernel driver بيتكلموا بنفس اللغة بالضبط.

---

### الـ File ده بالتحديد — `include/uapi/linux/spi/spi.h`

الـ file ده صغير جداً (44 سطر) لكنه **أساسي**. هو المكان الوحيد اللي بتُعرَّف فيه **SPI mode flags** اللي بيستخدمها كل من:
- الـ userspace programs (زي `spidev` tools)
- الـ kernel drivers نفسها (عن طريق `include/linux/spi/spi.h`)

#### الـ Problem اللي بيحلها

تخيل إنك بتكتب برنامج userspace بيتكلم مع flash memory عبر `/dev/spidev0.0`. لازم تقول للـ kernel:
- الـ clock بتشتغل إزاي؟ (phase و polarity)
- الـ chip select active high ولا low؟
- بتبعت البيانات MSB أول ولا LSB أول؟
- بتستخدم wire واحد ولا 4 wires (Quad SPI)؟

من غير تعريفات مشتركة، كل driver هيعمل flags بنفسه وهيبقى في فوضى. الـ file ده هو **العقد المشترك** بين الجميع.

---

### محتوى الـ File — تفصيلي

#### 1. الـ Clock Mode Flags

أهم حاجة في الـ SPI هي **كيفية قراءة الـ clock**. عندنا بيتين:

| Flag | Bit | المعنى |
|------|-----|--------|
| `SPI_CPHA` | 0 | **Clock Phase** — بتقرأ البيانات على الـ edge الأولى ولا التانية؟ |
| `SPI_CPOL` | 1 | **Clock Polarity** — الـ clock في حالة idle هو high ولا low؟ |

الـ combination بتاعتهم بيديك **4 modes**:

```
SPI_MODE_0 = CPOL=0, CPHA=0   ← الأشهر، أصله MicroWire
SPI_MODE_1 = CPOL=0, CPHA=1
SPI_MODE_2 = CPOL=1, CPHA=0
SPI_MODE_3 = CPOL=1, CPHA=1
```

كل device بياخد mode معين — لازم تقرأ الـ datasheet بتاعه.

#### 2. باقي الـ Flags

```c
#define SPI_CS_HIGH      _BITUL(2)  /* CS active HIGH بدل اللي عادةً LOW */
#define SPI_LSB_FIRST    _BITUL(3)  /* ابعت LSB قبل MSB */
#define SPI_3WIRE        _BITUL(4)  /* MOSI و MISO على نفس السلك */
#define SPI_LOOP         _BITUL(5)  /* loopback للـ testing */
#define SPI_NO_CS        _BITUL(6)  /* device وحيد على الـ bus، مفيش CS */
#define SPI_READY        _BITUL(7)  /* الـ slave يقدر يوقف الـ master */
```

#### 3. الـ Multi-Wire Flags — الحديثة

الـ SPI التقليدي بيستخدم سلك واحد للبيانات في كل اتجاه. لكن الـ flash memories الحديثة بتدعم **Dual SPI** و **Quad SPI** و **Octal SPI**:

```
Standard SPI:  1 wire  → 1 bit  per clock
Dual SPI:      2 wires → 2 bits per clock   (SPI_TX_DUAL / SPI_RX_DUAL)
Quad SPI:      4 wires → 4 bits per clock   (SPI_TX_QUAD / SPI_RX_QUAD)
Octal SPI:     8 wires → 8 bits per clock   (SPI_TX_OCTAL / SPI_RX_OCTAL)
```

ده مهم جداً للـ NOR flash chips زي W25Q128 اللي بتستخدمها في الـ embedded systems لأنه بيرفع الـ throughput بشكل ضخم.

#### 4. الـ `SPI_MODE_USER_MASK`

```c
#define SPI_MODE_USER_MASK  (_BITUL(19) - 1)
```

ده mask بيقول "كل الـ bits من 0 لـ 18 هي لـ userspace". الـ kernel بيحتفظ بالـ bits الأعلى لنفسه في `SPI_MODE_KERNEL_MASK`. ده ضمان إن الـ userspace والـ kernel مش هيتعارضوا.

---

### القصة الكاملة — كيف بيشتغل الكل مع بعض

```
Userspace App
    │
    │  open("/dev/spidev0.0")
    │  ioctl(fd, SPI_IOC_WR_MODE32, SPI_MODE_0 | SPI_TX_QUAD)
    │
    ▼
spidev.c  (drivers/spi/spidev.c)
    │  بيفسر الـ ioctl
    │  بيقارن الـ flags بـ SPI_MODE_USER_MASK
    │
    ▼
SPI Core  (drivers/spi/spi.c)
    │  بيحول الـ flags لـ spi_transfer struct
    │
    ▼
SPI Controller Driver  (مثلاً drivers/spi/spi-bcm2835.c)
    │  بيضبط الـ hardware registers
    │
    ▼
Hardware (SPI Controller على الـ SoC)
    │
    ▼
Physical Device (Flash, Sensor, DAC ...)
```

---

### الـ Files المهمة في الـ SPI Subsystem

#### Core

| File | الوظيفة |
|------|---------|
| `drivers/spi/spi.c` | قلب الـ subsystem — الـ bus logic والـ message queueing |
| `include/linux/spi/spi.h` | الـ kernel-internal API: `struct spi_device`, `struct spi_controller`, `struct spi_transfer` |
| `include/uapi/linux/spi/spi.h` | **الـ file ده** — الـ mode flags المشتركة بين kernel وuserspace |
| `include/uapi/linux/spi/spidev.h` | الـ ioctl API لـ `/dev/spidevX.Y` |

#### Userspace Interface

| File | الوظيفة |
|------|---------|
| `drivers/spi/spidev.c` | بيعمل `/dev/spidevX.Y` device لـ userspace access |
| `tools/spi/spidev_test.c` | test tool بيعمل SPI transfers من command line |
| `tools/spi/spidev_fdx.c` | tool لـ full-duplex testing |

#### Documentation

| File | الوظيفة |
|------|---------|
| `Documentation/spi/spi-summary.rst` | شرح كامل للـ subsystem |
| `Documentation/spi/spidev.rst` | شرح الـ userspace API |
| `Documentation/spi/multiple-data-lanes.rst` | شرح Dual/Quad/Octal SPI |

#### Hardware Drivers (أمثلة)

| File | الـ Hardware |
|------|-------------|
| `drivers/spi/spi-bcm2835.c` | Raspberry Pi (BCM2835 SoC) |
| `drivers/spi/spi-pl022.c` | ARM PrimeCell SSP |
| `drivers/spi/spi-imx.c` | NXP i.MX SoCs |
| `drivers/spi/spi-amd.c` | AMD platforms |
| `drivers/spi/spi-nxp-fspi.c` | NXP Flex SPI (Quad/Octal) |

#### SPI Memory Interface

| File | الوظيفة |
|------|---------|
| `include/linux/spi/spi-mem.h` | API خاص بـ SPI memory devices (NOR flash) |
| `drivers/mtd/spi-nor/` | SPI NOR flash subsystem بيبني عليه |

---

### ليه الـ File ده صغير ومهم في نفس الوقت؟

الـ UAPI headers لازم تبقى **stable** — يعني لو غيرت تعريف موجود، هتكسر كل الـ userspace programs القديمة. لذلك الـ kernel بيفصل الـ user-facing definitions في `uapi/` وبيتعامل معاها بحذر شديد.

الـ bits الجديدة (زي `SPI_MOSI_IDLE_LOW` و `SPI_MOSI_IDLE_HIGH`) اتضافت في الأخر من غير ما تأثر على القديم، وفي نفس الوقت الـ `SPI_MODE_USER_MASK` بيضمن إن الـ kernel عارف بالظبط إيه الـ bits اللي الـ user مسموله يعملها.
## Phase 2: شرح الـ SPI Framework

### المشكلة — ليه الـ SPI Subsystem موجود أصلاً؟

الـ **SPI (Serial Peripheral Interface)** بروتوكول hardware بسيط جداً: 4 أسلاك، full-duplex، synchronous. الـ hardware نفسه مش معقد. المشكلة مش في البروتوكول — المشكلة في الـ **software**.

من غير subsystem موحد، كل driver لكل chip (flash، sensor، display) هيكتب كوده الخاص عشان:
- يتعامل مع الـ SPI controller الخاص بالـ SoC
- يدير الـ chip select
- يتعامل مع الـ DMA أو PIO
- يقفل الـ bus لما بيبعت message
- يتعامل مع الـ clock polarity وال phase

النتيجة بدون framework: **كود متكرر، غير portable، ومربوط بالـ SoC**.

---

### الحل — الـ kernel بيعمل إيه؟

الـ SPI subsystem بيعمل **فصل واضح** بين طبقتين:

| الطبقة | المسؤولية |
|--------|-----------|
| **Protocol Driver** (`spi_driver`) | عارف إزاي يتكلم مع الـ chip (مثلاً W25Q flash) |
| **Controller Driver** (`spi_controller`) | عارف إزاي يشغّل الـ SPI hardware في الـ SoC |

الـ SPI core في المنتصف بيوفر:
- **Bus abstraction** موحدة (`spi_bus_type`)
- **Message queue** مُدارة
- **DMA mapping** تلقائي
- **Chip select** management
- **Clock/mode validation**

---

### المنظومة الكاملة — Big Picture Architecture

```
 ┌──────────────────────────────────────────────────────────────┐
 │                    Userspace / Kernel Consumers              │
 │   MTD subsystem    Networking    IIO    Input    Crypto ...  │
 └───────────────────────────┬──────────────────────────────────┘
                             │
 ┌───────────────────────────▼──────────────────────────────────┐
 │                   Protocol Drivers (spi_driver)              │
 │    spi-nor.c      mcp2515.c    ili9341.c    tpm_tis_spi.c    │
 │    (NOR Flash)   (CAN ctrl)   (LCD)        (TPM chip)        │
 │                                                              │
 │    يبنوا spi_message من spi_transfer segments               │
 │    يستدعوا spi_sync() / spi_async()                         │
 └───────────────────────────┬──────────────────────────────────┘
                             │  spi_message
 ┌───────────────────────────▼──────────────────────────────────┐
 │                    SPI Core (drivers/spi/spi.c)              │
 │                                                              │
 │  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐  │
 │  │  spi_bus_type│  │ Message Queue│  │  DMA Management   │  │
 │  │  (Device     │  │  (kthread    │  │  (can_dma hook +  │  │
 │  │   Model)     │  │   pump)      │  │   dmaengine)      │  │
 │  └──────────────┘  └──────────────┘  └───────────────────┘  │
 │                                                              │
 │  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐  │
 │  │  CS Timing   │  │  Mode/Clock  │  │  spi_res          │  │
 │  │  Management  │  │  Validation  │  │  (devres for msgs)│  │
 │  └──────────────┘  └──────────────┘  └───────────────────┘  │
 └───────────────────────────┬──────────────────────────────────┘
                             │  transfer_one() / transfer_one_message()
 ┌───────────────────────────▼──────────────────────────────────┐
 │               Controller Drivers (spi_controller)            │
 │    spi-bcm2835.c   spi-imx.c   spi-qcom-qspi.c   spi-pl022  │
 │    (RPi SoC)       (i.MX SoC)  (Qualcomm QSPI)  (ARM PL022) │
 │                                                              │
 │    بيتكلموا مباشرة مع الـ hardware registers               │
 │    بيملوا FIFO / يشغّلوا DMA / بيتعاملوا مع interrupts     │
 └───────────────────────────┬──────────────────────────────────┘
                             │
 ┌───────────────────────────▼──────────────────────────────────┐
 │                    Physical SPI Bus                          │
 │         MOSI ──── MISO ──── SCLK ──── CS0 CS1 CS2           │
 └──────────────────────────────────────────────────────────────┘
```

---

### التشبيه الواقعي — المطعم

تخيّل مطعم فيه:

| عنصر المطعم | المقابل في SPI |
|------------|----------------|
| **الزبون** | الـ consumer subsystem (MTD, IIO...) |
| **النادل** (Protocol Driver) | `spi_driver` — عارف الأكل بس مش عارف المطبخ |
| **أوردر الطلب** | `spi_message` |
| **كل صنف في الأوردر** | `spi_transfer` |
| **مدير المطعم** (SPI Core) | يتأكد إن الأوردرات بتتنفذ بالترتيب، يدير الطابور |
| **الطباخ** (Controller Driver) | `spi_controller` — عارف يشتغل على المعدة (الـ SoC hardware) |
| **المعدة/الموقد** | الـ SPI hardware registers |
| **CS (Chip Select)** | الـ "طاولة رقم كذا" — الطباخ يعرف لمين يبعت الأكل |

**التفصيل الدقيق:**

- الـ **نادل (protocol driver)** مش بيعرف المطبخ شغال بـ DMA ولا PIO — ده شغل المدير
- الـ **مدير (SPI core)** بيضمن إن طلب الزبون A يخلص قبل ما طلب الزبون B يبدأ (message atomicity)
- الـ **طباخ (controller driver)** لو الموقد (hardware) بيدعم DMA، بيستخدمه تلقائياً من غير ما النادل يعرف
- الـ **أوردر (spi_message)** ممكن يكون فيه أكتر من صنف (spi_transfer) وكلهم بيتنفذوا وهو chip select مضغوط
- لو حصل غلط في صنف واحد، المدير بيبلغ النادل عن طريق الـ `complete()` callback

---

### الـ Core Abstraction — الفكرة المحورية

**الـ spi_message هي الوحدة الأساسية للتعامل.** مش الـ byte ومش الـ transfer الواحدة.

```
spi_message = sequence of spi_transfers + atomic execution guarantee
```

يعني لما بتبعت `spi_message`، الـ bus بيتقفل للـ device ده من أول transfer لآخر transfer. مفيش device تاني يقدر يتكلم على الـ bus في النص.

ده مهم لأن كتير من الـ SPI chips بتحتاج **command + data** كـ transaction واحدة غير قابلة للتقطيع.

---

### الـ Core Structs وعلاقتها ببعض

```
spi_controller
│
├── struct device dev          ← registered in spi_bus_type
├── s16 bus_num                ← e.g., SPI0, SPI1, SPI2
├── u16 num_chipselect         ← كام chip ممكن يتوصل
│
├── [Capabilities]
│   ├── u32 mode_bits          ← الـ modes اللي الـ controller بيدعمها
│   ├── u32 bits_per_word_mask ← supported word sizes
│   ├── u32 min/max_speed_hz   ← clock limits
│   └── u16 flags              ← HALF_DUPLEX, NO_RX, NO_TX...
│
├── [Message Queue]
│   ├── struct kthread_worker *kworker   ← message pump thread
│   ├── struct list_head queue           ← pending messages
│   ├── struct spi_message *cur_msg      ← currently executing
│   └── spinlock_t queue_lock
│
├── [Callbacks — Controller fills these]
│   ├── int (*setup)(spi_device*)              ← configure for this device
│   ├── int (*transfer_one)(ctlr, spi, xfer)   ← send ONE transfer
│   ├── int (*transfer_one_message)(ctlr, msg) ← send whole message (optional)
│   ├── void (*set_cs)(spi_device*, bool)      ← assert/deassert CS
│   ├── int (*prepare_transfer_hardware)(ctlr) ← wake up hardware
│   └── int (*unprepare_transfer_hardware)(ctlr) ← sleep hardware
│
└── [DMA Support]
    ├── bool (*can_dma)(ctlr, spi, xfer)
    ├── struct dma_chan *dma_tx
    └── struct dma_chan *dma_rx

        ▼ يمتلك/يدير

spi_device (واحد لكل chip على الـ bus)
│
├── struct device dev
├── struct spi_controller *controller  ← pointer للـ controller اللي عليه
├── u32 mode                           ← SPI_MODE_0..3 + flags
├── u32 max_speed_hz
├── u8 bits_per_word
├── u8 chip_select[]                   ← physical CS numbers (up to 4)
├── struct gpio_desc *cs_gpiod[]       ← GPIO-based CS (optional)
├── struct spi_delay cs_setup/hold/inactive
└── struct spi_statistics __percpu *pcpu_statistics

        ▼ يُرسل إليه

spi_message
│
├── struct list_head transfers    ← linked list of spi_transfer
├── struct spi_device *spi        ← target device
├── int status                    ← result (0 = success)
├── void (*complete)(void*)       ← async completion callback
├── void *context                 ← passed to complete()
├── unsigned frame_length         ← total bytes expected
├── unsigned actual_length        ← bytes actually transferred
└── struct list_head resources    ← spi_res list (cleanup on complete)

        ▲ يتكوّن من

spi_transfer (الوحدة الفعلية للبيانات)
│
├── const void *tx_buf        ← data to send (NULL = send zeros)
├── void *rx_buf              ← buffer to receive (NULL = discard)
├── unsigned len              ← bytes in this transfer
├── u32 speed_hz              ← override device speed for this xfer
├── u8 bits_per_word          ← override device bpw for this xfer
├── unsigned cs_change:1      ← toggle CS after this transfer
├── unsigned cs_off:1         ← do this transfer with CS deasserted
├── unsigned tx_nbits:4       ← 1=single, 2=dual, 4=quad, 8=octal
├── unsigned rx_nbits:4
├── struct spi_delay delay    ← wait after this transfer
├── struct spi_delay word_delay ← wait between words
├── dma_addr_t tx_dma/rx_dma ← DMA addresses (filled by core)
├── struct sg_table tx_sg/rx_sg ← scatterlist for DMA
└── struct list_head transfer_list ← link in spi_message.transfers
```

---

### الـ SPI Modes — CPOL و CPHA بالتفصيل

الأربع modes المعرّفة في `uapi/linux/spi/spi.h` بتتحكم في شكل الـ clock:

**الـ CPOL (Clock Polarity):** إيه مستوى الـ clock وهو idle؟
- `CPOL=0`: clock = LOW وهو idle
- `CPOL=1`: clock = HIGH وهو idle

**الـ CPHA (Clock Phase):** البيانات بتتأخد على إيه edge؟
- `CPHA=0`: data sampled on **leading** edge (first edge)
- `CPHA=1`: data sampled on **trailing** edge (second edge)

```
MODE 0 (CPOL=0, CPHA=0):
CLK:  _┌─┐_┌─┐_┌─┐_
DATA: ─X───X───X──  ← sampled on rising edge

MODE 1 (CPOL=0, CPHA=1):
CLK:  _┌─┐_┌─┐_┌─┐_
DATA: ──X───X───X─  ← sampled on falling edge

MODE 2 (CPOL=1, CPHA=0):
CLK:  ─┘_┌─┘_┌─┘_
DATA: ─X───X───X──  ← sampled on falling edge

MODE 3 (CPOL=1, CPHA=1):
CLK:  ─┘_┌─┘_┌─┘_
DATA: ──X───X───X─  ← sampled on rising edge
```

الـ flags التانية في الـ `mode` field:

| Flag | المعنى |
|------|---------|
| `SPI_CS_HIGH` | الـ CS active HIGH بدل LOW |
| `SPI_LSB_FIRST` | بيبعت الـ LSB الأول (عكس الـ default) |
| `SPI_3WIRE` | MOSI وMISO على نفس السلك (half-duplex) |
| `SPI_LOOP` | loopback — TX متوصل بـ RX داخلياً |
| `SPI_NO_CS` | في device واحد على الـ bus، مفيش CS |
| `SPI_TX_DUAL/QUAD/OCTAL` | multi-lane TX (QSPI/OSPI) |
| `SPI_RX_DUAL/QUAD/OCTAL` | multi-lane RX |
| `SPI_CS_WORD` | toggle CS بعد كل word |
| `SPI_3WIRE_HIZ` | السلك بيعمل high-impedance في turnaround |
| `SPI_MOSI_IDLE_LOW/HIGH` | مستوى MOSI وهو idle |

**تقسيم الـ mode bits بين userspace وkernel:**

```c
/* uapi/linux/spi/spi.h — bits 0..18 (user-visible) */
#define SPI_MODE_USER_MASK   (_BITUL(19) - 1)

/* include/linux/spi/spi.h — bits 29..31 (kernel-only) */
#define SPI_NO_TX            BIT(31)  /* No transmit wire */
#define SPI_NO_RX            BIT(30)  /* No receive wire */
#define SPI_TPM_HW_FLOW      BIT(29)  /* TPM hardware flow control */
#define SPI_MODE_KERNEL_MASK (~(BIT(29) - 1))

/* compile-time check: the two masks must NOT overlap */
static_assert((SPI_MODE_KERNEL_MASK & SPI_MODE_USER_MASK) == 0, ...);
```

ده design مهم: الـ userspace (عبر `spidev`) ممكن يشوف ويغيّر الـ lower bits بس. الـ kernel-only bits محمية.

---

### Message Queue Flow — إزاي الـ message بتتنفذ

```
Protocol Driver
     │
     │  spi_async(spi, msg)  ← non-blocking, queues message
     │  or
     │  spi_sync(spi, msg)   ← blocking, waits for completion
     ▼
SPI Core: spi_async()
     │
     ├─ validates message
     ├─ calls ctlr->transfer(spi, msg)  ← adds to queue
     │
     ▼
kthread pump (spi_pump_messages)
     │
     ├─ calls ctlr->prepare_transfer_hardware()
     ├─ for each message:
     │   ├─ calls ctlr->prepare_message()    ← e.g., DMA mapping
     │   ├─ assert CS
     │   ├─ for each spi_transfer in message:
     │   │   ├─ (if can_dma) map buffers for DMA
     │   │   ├─ calls ctlr->transfer_one(ctlr, spi, xfer)
     │   │   ├─ waits for xfer_completion
     │   │   └─ handle cs_change
     │   ├─ deassert CS
     │   ├─ calls ctlr->unprepare_message()
     │   └─ calls msg->complete(msg->context)  ← notify protocol driver
     └─ calls ctlr->unprepare_transfer_hardware()
```

**fast path في `spi_sync`:** لو الـ queue فاضية والـ controller مش مشغول، الـ SPI core ممكن ينفذ الـ message **مباشرة في الـ calling context** من غير ما يحتاج الـ kthread. ده بيتتتبع في `spi_sync_immediate` stats.

---

### إيه اللي الـ SPI Core بيمتلكه مقابل اللي بيفوّض للـ Driver؟

#### الـ Core بيمتلك:
- بناء وإدارة الـ `spi_bus_type` (device model integration)
- تسجيل وإلغاء تسجيل الـ controllers والـ devices
- الـ message queue وكل الـ locking الخاص به
- الـ kthread pump ومنطق الـ scheduling
- الـ DMA mapping (عبر `can_dma` + dmaengine)
- الـ CS GPIO management (`use_gpio_descriptors`)
- الـ `spi_sync` fast path optimization
- الـ statistics collection (percpu)
- الـ `spi_res` lifecycle management
- الـ mode/speed validation قبل الـ transfer
- الـ message optimization framework

#### الـ Core بيفوّض للـ Controller Driver:
- تهيئة الـ hardware registers
- ضبط الـ clock divider عشان يطلع الـ speed المطلوبة
- تحكم في الـ CS (hardware CS أو بيتوكل على الـ core للـ GPIO CS)
- تنفيذ الـ actual transfer (DMA أو PIO أو interrupt-driven)
- الـ timing delays (cs_setup, cs_hold) على مستوى الـ hardware
- الـ word-level serialization

#### الـ Core بيفوّض للـ Protocol Driver:
- تحديد الـ SPI mode المطلوب للـ chip
- بناء الـ spi_transfer buffers (command + data layout)
- تفسير الـ received data
- إدارة الـ upper-layer protocol (MTD, IIO, etc.)

---

### مثال واقعي — قراءة من NOR Flash

```c
/* Protocol driver (spi-nor) يقرأ من W25Q128 */

/* Step 1: بناء الـ transfers */
struct spi_transfer xfers[2];
u8 cmd_buf[4] = { 0x03, addr_hi, addr_mid, addr_lo }; /* READ command */
u8 data_buf[256];

/* Transfer 1: إرسال الـ command */
memset(&xfers[0], 0, sizeof(xfers[0]));
xfers[0].tx_buf = cmd_buf;
xfers[0].len    = 4;

/* Transfer 2: استقبال البيانات */
memset(&xfers[1], 0, sizeof(xfers[1]));
xfers[1].rx_buf = data_buf;
xfers[1].len    = 256;

/* Step 2: بناء الـ message */
struct spi_message msg;
spi_message_init(&msg);
spi_message_add_tail(&xfers[0], &msg);
spi_message_add_tail(&xfers[1], &msg);

/* Step 3: إرسال (blocking) */
int ret = spi_sync(spi_device, &msg);

/*
 * اللي بيحصل تحت:
 * - CS بيتضغط
 * - xfers[0] بيتبعت (4 bytes command)
 * - xfers[1] بيتبعت (clock يستمر، MOSI = 0, MISO بترجع data)
 * - CS بيترفع
 * - msg.complete() بيتستدعى (أو spi_sync بيصحى)
 */
```

---

### الـ spi_driver — Protocol Driver Registration

```c
static const struct spi_device_id my_chip_ids[] = {
    { "my-sensor", 0 },
    { }
};

static struct spi_driver my_sensor_driver = {
    .driver = {
        .name  = "my-sensor",
        .owner = THIS_MODULE,
        .of_match_table = my_sensor_of_ids,
    },
    .id_table = my_chip_ids,
    .probe    = my_sensor_probe,
    .remove   = my_sensor_remove,
};

/* بدل module_init + module_exit */
module_spi_driver(my_sensor_driver);
```

**الـ probe()** بيستدعيه الـ SPI core لما بيلاقي device على الـ bus بيـ match اسمه. الـ `spi_device *` اللي بيجي بيكون جاهز بالـ mode وال speed من الـ Device Tree أو الـ ACPI.

---

### الـ spi_controller — Controller Driver Registration

```c
/* في الـ platform driver probe() */
struct spi_controller *ctlr;

/* تخصيص الـ controller + private data للـ driver */
ctlr = devm_spi_alloc_host(&pdev->dev, sizeof(struct my_spi_priv));

/* ملء الـ capabilities */
ctlr->bus_num         = pdev->id;
ctlr->num_chipselect  = 4;
ctlr->mode_bits       = SPI_CPOL | SPI_CPHA | SPI_CS_HIGH;
ctlr->bits_per_word_mask = SPI_BPW_RANGE_MASK(4, 16);
ctlr->min_speed_hz    = 400000;    /* 400 kHz */
ctlr->max_speed_hz    = 50000000;  /* 50 MHz */

/* ملء الـ callbacks */
ctlr->setup       = my_spi_setup;
ctlr->transfer_one = my_spi_transfer_one;  /* الـ core بيدير الباقي */
ctlr->set_cs      = my_spi_set_cs;

/* تسجيل في الـ SPI subsystem */
devm_spi_register_controller(&pdev->dev, ctlr);
```

لو الـ controller بيدعم DMA:
```c
ctlr->can_dma    = my_spi_can_dma;   /* الـ core يسأله */
ctlr->dma_tx     = dma_request_chan(...);
ctlr->dma_rx     = dma_request_chan(...);
/* الـ core هو اللي بيعمل الـ mapping تلقائياً */
```

---

### الـ Device Tree Integration

الـ SPI subsystem بيتكامل مع الـ **Device Tree** subsystem — اللي بيحتاج فهم مسبق: الـ OF (Open Firmware) layer بيحوّل الـ DT nodes لـ `struct device` objects.

```dts
&spi0 {
    status = "okay";
    cs-gpios = <&gpio 8 GPIO_ACTIVE_LOW>;

    flash@0 {
        compatible = "winbond,w25q128";
        reg = <0>;               /* CS0 */
        spi-max-frequency = <50000000>;
        spi-cpol;               /* CPOL=1 */
        spi-cpha;               /* CPHA=1 → MODE 3 */
    };

    sensor@1 {
        compatible = "ti,ads1015";
        reg = <1>;               /* CS1 */
        spi-max-frequency = <4000000>;
    };
};
```

الـ SPI core بيقرأ هذه الـ properties تلقائياً وبيملي `spi_device.mode` وال `max_speed_hz` قبل ما الـ `probe()` يتستدعى.

---

### الـ Multi-lane SPI — DSPI/QSPI/OSPI

الـ flags من `SPI_TX_DUAL` لـ `SPI_RX_OCTAL` بتتحكم في عدد الـ data lanes:

```
Standard SPI (1-bit):                QSPI (4-bit):

MOSI ──────────────────►             IO0 ──────────────────►
MISO ◄──────────────────             IO1 ◄──────────────────
SCLK ──────────────────►             IO2 ────────────────────
CS   ──────────────────►             IO3 ────────────────────
                                     SCLK ──────────────────►
                                     CS   ──────────────────►

Throughput: 1 bit/clock             Throughput: 4 bits/clock
```

في `spi_transfer`:
```c
xfer.tx_nbits = SPI_NBITS_QUAD;   /* TX على 4 lanes */
xfer.rx_nbits = SPI_NBITS_QUAD;   /* RX على 4 lanes */
```

الـ OSPI (Octal SPI) بيوصل لـ 8 lanes مع DDR (Double Data Rate) — بيبعت على الـ rising والـ falling edge:
```c
xfer.dtr_mode = true;  /* Double Transfer Rate */
```

---

### الـ spi_res — Resource Management

الـ `spi_res` نظام شبيه بالـ **devres** بس محدود بعمر الـ `spi_message`. مفيد لو الـ protocol driver محتاج يخصص موارد مؤقتة خلال الـ message processing:

```c
struct spi_res {
    struct list_head   entry;
    spi_res_release_t  release;  /* called when message completes */
    unsigned long long data[];   /* flexible array for driver use */
};
```

مثلاً، لو محتاج تخصص bounce buffer للـ DMA alignment، بتربطه بالـ message وبيتحرر تلقائياً لما الـ message تخلص.

---

### الـ Statistics — spi_statistics

كل `spi_device` وكل `spi_controller` عنده **per-CPU statistics** محمية بـ `u64_stats_sync`:

```c
struct spi_statistics {
    struct u64_stats_sync syncp;    /* protect 64-bit reads on 32-bit CPUs */

    u64_stats_t messages;           /* total messages processed */
    u64_stats_t transfers;          /* total transfers */
    u64_stats_t errors;
    u64_stats_t timedout;

    u64_stats_t spi_sync;
    u64_stats_t spi_sync_immediate; /* fast path executions */
    u64_stats_t spi_async;

    u64_stats_t bytes_tx;
    u64_stats_t bytes_rx;

    /* histogram of transfer sizes (helps tune DMA thresholds) */
    u64_stats_t transfer_bytes_histo[SPI_STATISTICS_HISTO_SIZE]; /* 17 buckets */
    u64_stats_t transfers_split_maxsize; /* times a xfer was split */
};
```

الـ per-CPU design ضروري لأن الـ SPI transfers ممكن تحصل من contexts مختلفة في نفس الوقت (مثلاً IRQ context للـ completion وthread context للـ submission).

الـ `u64_stats_sync` — بيحتاج فهم مسبق: هو mechanism بيستخدم `seqcount` على الـ 32-bit systems عشان يضمن consistent read لقيم الـ 64-bit من غير lock.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ File في جملة واحدة

الـ `include/uapi/linux/spi/spi.h` هو **الـ UAPI header** الوحيد للـ SPI subsystem — بيعرّف بس الـ flags والـ mode bits اللي بيشوفها الـ userspace. مفيش structs هنا خالص، كل الـ structs موجودة في الـ kernel-internal header.

---

### Step 0: Cheatsheet — الـ Flags والـ Mode Bits

#### جدول الـ Clock Mode Bits (الـ 2 bits الأساسيين)

| Macro | Bit | القيمة | المعنى |
|-------|-----|--------|--------|
| `SPI_CPHA` | 0 | `0x001` | Clock Phase — بيحدد امتى البيانات بتتسامبل (rising أو falling edge) |
| `SPI_CPOL` | 1 | `0x002` | Clock Polarity — بيحدد الـ idle state للـ clock (low أو high) |

#### جدول الـ SPI Modes الأربعة (CPOL × CPHA)

| Mode Macro | CPOL | CPHA | الـ Idle Clock | الـ Sample Edge | الاستخدام الشائع |
|------------|------|------|----------------|-----------------|-----------------|
| `SPI_MODE_0` | 0 | 0 | Low | Rising | الأكثر شيوعًا — EEPROM, ADC |
| `SPI_MODE_1` | 0 | 1 | Low | Falling | بعض الـ sensors |
| `SPI_MODE_2` | 1 | 0 | High | Falling | بعض الـ flash chips |
| `SPI_MODE_3` | 1 | 1 | High | Rising | SD cards (sometimes) |
| `SPI_MODE_X_MASK` | — | — | `0x003` | mask للـ bits 0-1 بس | للفلترة |

#### جدول الـ Extended Mode Flags (bits 2-18)

| Macro | Bit Index | القيمة Hex | المعنى |
|-------|-----------|------------|--------|
| `SPI_CS_HIGH` | 2 | `0x004` | الـ chip select active عند HIGH مش LOW |
| `SPI_LSB_FIRST` | 3 | `0x008` | إرسال LSB أولًا (العكس من الـ default MSB) |
| `SPI_3WIRE` | 4 | `0x010` | الـ MOSI والـ MISO على نفس الـ wire (half-duplex) |
| `SPI_LOOP` | 5 | `0x020` | Loopback mode — الـ TX متوصل بالـ RX داخليًا للتست |
| `SPI_NO_CS` | 6 | `0x040` | مفيش chip select — device واحد على الـ bus |
| `SPI_READY` | 7 | `0x080` | الـ slave بيشد MISO لـ low عشان يقول "استنى" |
| `SPI_TX_DUAL` | 8 | `0x100` | إرسال على 2 wires في نفس الوقت (MOSI + MISO) |
| `SPI_TX_QUAD` | 9 | `0x200` | إرسال على 4 wires (SPI Flash Quad mode) |
| `SPI_RX_DUAL` | 10 | `0x400` | استقبال على 2 wires |
| `SPI_RX_QUAD` | 11 | `0x800` | استقبال على 4 wires |
| `SPI_CS_WORD` | 12 | `0x1000` | toggle الـ CS بعد كل word مش بعد كل message |
| `SPI_TX_OCTAL` | 13 | `0x2000` | إرسال على 8 wires (Octal SPI / OPI) |
| `SPI_RX_OCTAL` | 14 | `0x4000` | استقبال على 8 wires |
| `SPI_3WIRE_HIZ` | 15 | `0x8000` | High-impedance turnaround في الـ 3-wire mode |
| `SPI_RX_CPHA_FLIP` | 16 | `0x10000` | عكس الـ CPHA في الـ RX-only transfers |
| `SPI_MOSI_IDLE_LOW` | 17 | `0x20000` | خلي الـ MOSI line لـ low لما الـ bus idle |
| `SPI_MOSI_IDLE_HIGH` | 18 | `0x40000` | خلي الـ MOSI line لـ high لما الـ bus idle |

#### الـ Mask المهم

| Macro | القيمة | المعنى |
|-------|--------|--------|
| `SPI_MODE_USER_MASK` | `_BITUL(19) - 1` = `0x7FFFF` | كل الـ bits من 0 لـ 18 — ده الـ boundary بين الـ userspace bits والـ kernel bits |

> **ملاحظة مهمة:** الـ `SPI_MODE_KERNEL_MASK` موجود في `include/linux/spi/spi.h` وبيغطي الـ bits العليا (من 31 نازلًا). الـ design المقصود: الـ userspace يشتغل من bit 0 لـ 18 صاعدًا، والـ kernel يشتغل من bit 31 نازلًا — ولازم ما يتداخلوش (في `static_assert` بيتحقق من ده).

---

### Step 1: الـ Structs في الـ File

الـ UAPI file نفسه (`uapi/linux/spi/spi.h`) **مفيش فيه structs** — بس macros. لكن الـ structs المرتبطة بالـ flags دي موجودة في `include/linux/spi/spi.h` اللي بيعمل `#include` للـ UAPI file.

الـ struct الرئيسي اللي بيستخدم الـ flags دي هو `spi_device`:

#### `struct spi_device`

**الغرض:** بيمثل device واحد متوصل على الـ SPI bus من ناحية الـ controller. كل SPI chip (flash, ADC, display) بيبقى له `spi_device` واحد.

| Field | النوع | الـ Flag المرتبط | الشرح |
|-------|-------|-----------------|-------|
| `mode` | `u32` | `SPI_MODE_*`, `SPI_CS_HIGH`, كل الـ flags | الـ field الرئيسي اللي بتتخزن فيه الـ flags |
| `max_speed_hz` | `u32` | — | أقصى سرعة clock بالـ Hz |
| `bits_per_word` | `u8` | — | عدد bits في كل word (عادةً 8) |
| `chip_select[]` | `u8[4]` | `SPI_NO_CS` | array من الـ physical CS lines |
| `cs_gpiod[]` | `struct gpio_desc*[4]` | `SPI_CS_HIGH` | GPIO lines للـ chip select |
| `num_tx_lanes` | `u8` | `SPI_TX_DUAL/QUAD/OCTAL` | عدد الـ TX data lanes |
| `num_rx_lanes` | `u8` | `SPI_RX_DUAL/QUAD/OCTAL` | عدد الـ RX data lanes |

#### الـ mode field تقسيم الـ bits

```
bit 31                    bit 19  bit 18          bit 0
|<-- SPI_MODE_KERNEL_MASK -->|    |<-- SPI_MODE_USER_MASK -->|
| SPI_NO_TX  SPI_NO_RX  ... |    | ... SPI_CPOL  SPI_CPHA   |
| (kernel-only, from high)   |    | (userspace-visible)      |
```

---

### Step 2: علاقات الـ Structs (ASCII Diagram)

```
uapi/linux/spi/spi.h
       │
       │ defines flags used in ↓
       ▼
┌─────────────────────────────────────────────┐
│              struct spi_device              │
│  .mode (u32) ←── SPI_CPHA, SPI_CPOL,       │
│                  SPI_CS_HIGH, SPI_TX_QUAD.. │
│  .controller ──────────────────────────────┼──► struct spi_controller
│  .dev (struct device) ─────────────────────┼──► Linux device model
│  .cs_gpiod[] ──────────────────────────────┼──► struct gpio_desc
│  .pcpu_statistics ─────────────────────────┼──► struct spi_statistics (per-cpu)
│  .word_delay / .cs_setup / .cs_hold ───────┼──► struct spi_delay
└─────────────────────────────────────────────┘
       │
       │ used in
       ▼
┌─────────────────────────────────────────────┐
│              struct spi_transfer            │
│  .speed_hz   (overrides spi_device value)   │
│  .bits_per_word (overrides spi_device value)│
│  .tx_buf / .rx_buf                          │
│  .cs_change  (relates to SPI_CS_WORD)       │
└─────────────────────────────────────────────┘
       │
       │ collected into
       ▼
┌─────────────────────────────────────────────┐
│              struct spi_message             │
│  list of spi_transfers                      │
│  .spi ──────────────────────────────────────┼──► struct spi_device (back ref)
│  .complete() callback                        │
└─────────────────────────────────────────────┘
```

---

### Step 3: Lifecycle Diagrams

#### دورة حياة الـ SPI Mode Flags

```
Board/DT config
      │
      │ spi_of_register_child() or spi_new_device()
      ▼
spi_device.mode ← parse "spi-cpol", "spi-cpha", "spi-cs-high" from DT
      │
      │ driver probe → spi_setup()
      ▼
spi_controller->setup(spi_device)
      │  ← validates mode flags against controller->mode_bits
      │  ← rejects unsupported flags (e.g. SPI_TX_QUAD if hw can't)
      ▼
flags now "committed" to hardware registers
      │
      │ spi_sync() / spi_async()
      ▼
each spi_transfer can override: speed_hz, bits_per_word
but NOT: CPOL/CPHA (those are per-device, set once)
      │
      │ device unbind / rmmod
      ▼
spi_dev_put() → put_device() → kfree
```

#### دورة حياة الـ Multi-lane Flags (DUAL/QUAD/OCTAL)

```
spi_device.mode |= SPI_TX_QUAD
      │
      │ spi_setup() validates:
      │   controller->mode_bits & SPI_TX_QUAD ?
      │        YES → OK
      │        NO  → return -EINVAL
      ▼
spi_transfer.tx_nbits = SPI_NBITS_QUAD (set by driver)
      │
      │ __spi_validate() checks:
      │   if tx_nbits == QUAD but !(mode & SPI_TX_QUAD) → error
      ▼
controller driver maps tx_nbits → hardware QUAD mode register
```

---

### Step 4: Call Flow Diagrams

#### تطبيق الـ mode على الـ hardware

```
userspace / driver sets spi->mode = SPI_MODE_3 | SPI_CS_HIGH
      │
      ▼
spi_setup(spi_device)                    [spi.c]
  │
  ├─► validates: mode & ~controller->mode_bits → -EINVAL
  │
  ├─► calls controller->setup(spi_device)      [e.g. pl022.c]
  │     │
  │     ├─► reads spi->mode & SPI_CPOL → set CPOL bit in hw reg
  │     ├─► reads spi->mode & SPI_CPHA → set CPHA bit in hw reg
  │     ├─► reads spi->mode & SPI_CS_HIGH → invert CS polarity
  │     └─► writes to hardware SSPCR0/SSPCR1 registers
  │
  └─► returns 0 on success
```

#### الـ SPI_READY flag (flow control)

```
spi_device.mode |= SPI_READY
      │
      ▼
controller starts transfer
      │
      ├─► slave asserts MISO low = "I'm not ready"
      │
      ▼
controller HW detects MISO low (if SPI_READY supported)
      │
      ├─► pauses clock (stretches transfer)
      │
      ▼
slave deasserts MISO = "ready now"
      │
      ▼
controller resumes clocking data
```

#### الـ SPI_CS_WORD flag

```
normal mode (no SPI_CS_WORD):
  CS low ─────────────────────────── CS high
          [word1][word2][word3]

SPI_CS_WORD mode:
  CS low─CS high  CS low─CS high  CS low─CS high
          [word1]          [word2]          [word3]
```

---

### Step 5: الـ Locking Strategy

الـ UAPI header نفسه مالوش لوكات — بس الـ structs اللي بتستخدم الـ flags دي بيطبقوا locking strategy واضحة:

#### جدول الـ Locks في الـ SPI Subsystem

| الـ Lock | نوعه | بيحمي إيه | ملاحظات |
|----------|-------|-----------|---------|
| `spi_controller->bus_lock_mutex` | `struct mutex` | الـ bus بالكامل في الـ `spi_sync_locked()` | للـ exclusive bus access |
| `spi_controller->io_mutex` | `struct mutex` | الـ message queue أثناء التنفيذ | بيضمن message واحدة بس شغالة |
| `spi_device->controller_state` | بيدار بالـ controller driver | الـ per-device hardware state | كل controller بيختار طريقة الـ locking بتاعته |
| `spi_statistics->syncp` | `struct u64_stats_sync` | الـ per-cpu statistics fields | seqcount لـ 32-bit systems |

#### قاعدة الـ mode field

الـ `spi_device->mode` بيتكتب **مرة واحدة بس** في وقت الـ setup (قبل أي transfer). بعد كده بيتقرأ بس من الـ concurrent transfers — فمحتاجش lock أثناء الـ transfers.

```
setup time (single thread):
  spi_setup() → controller->setup() → mode committed to HW

runtime (concurrent reads, no writes):
  spi_sync() reads mode → no lock needed (read-only after setup)
```

#### ترتيب الـ Locks (Lock Ordering) — لمنع الـ deadlock

```
1. bus_lock_mutex          (الأعلى — lock الـ bus كله)
   └─► 2. io_mutex         (lock الـ active transfer)
          └─► 3. spinlock  (في بعض الـ controllers للـ IRQ context)
```

لازم دايمًا تاخد الـ locks بالترتيب ده من فوق لتحت، وإلا deadlock.

---

### ملاحظة على الـ UAPI / Kernel Boundary

```
include/uapi/linux/spi/spi.h          include/linux/spi/spi.h
┌────────────────────────┐            ┌──────────────────────────┐
│ SPI_CPHA     bit 0     │            │ SPI_NO_TX    bit 31      │
│ SPI_CPOL     bit 1     │            │ SPI_NO_RX    bit 30      │
│ ...                    │            │ SPI_TPM_HW_FLOW bit 29   │
│ SPI_MOSI_IDLE_HIGH b18 │            │ ...                      │
│                        │            │                          │
│ SPI_MODE_USER_MASK     │            │ SPI_MODE_KERNEL_MASK     │
│   = bits 0..18         │            │   = bits 29..31          │
│   visible to userspace │            │   kernel-only            │
└────────────────────────┘            └──────────────────────────┘
              ↑                                    ↑
              └─── static_assert: no overlap ──────┘
```

الـ design ده بيضمن إن الـ userspace applications زي `spidev` مش ممكن تعمل conflict مع الـ kernel-internal flags لما بتحدد الـ mode عن طريق `ioctl(SPI_IOC_WR_MODE32)`.
## Phase 4: شرح الـ Functions

> الملف `include/uapi/linux/spi/spi.h` هو **UAPI header** بحت — مش فيه functions ولا structs، بس فيه **macro constants** بتشكّل الـ SPI mode flags الرسمية اللي بتتشارك بين kernel-space وuser-space (عبر `ioctl` وسواه). الـ "API" هنا هي الـ macros نفسها.

---

### ملخص كل الـ Macros — Cheatsheet

#### مجموعة الـ Clock Configuration

| Macro | Bit | القيمة | الوصف |
|---|---|---|---|
| `SPI_CPHA` | 0 | `0x0001` | Clock Phase — متى يتأخذ السامبل |
| `SPI_CPOL` | 1 | `0x0002` | Clock Polarity — الحالة الـ idle للـ clock |

#### مجموعة الـ Standard Modes

| Macro | القيمة | CPOL | CPHA |
|---|---|---|---|
| `SPI_MODE_0` | `0x0000` | 0 | 0 |
| `SPI_MODE_1` | `0x0001` | 0 | 1 |
| `SPI_MODE_2` | `0x0002` | 1 | 0 |
| `SPI_MODE_3` | `0x0003` | 1 | 1 |
| `SPI_MODE_X_MASK` | `0x0003` | — | mask للـ 4 modes |

#### مجموعة الـ Bus Behavior Flags

| Macro | Bit | القيمة | الوصف |
|---|---|---|---|
| `SPI_CS_HIGH` | 2 | `0x0004` | Chip Select active-high |
| `SPI_LSB_FIRST` | 3 | `0x0008` | LSB يتبعت الأول |
| `SPI_3WIRE` | 4 | `0x0010` | MOSI/MISO على نفس السلك |
| `SPI_LOOP` | 5 | `0x0020` | Loopback mode |
| `SPI_NO_CS` | 6 | `0x0040` | مفيش chip select |
| `SPI_READY` | 7 | `0x0080` | Slave يلوّع low عشان يوقف |

#### مجموعة الـ Multi-Wire (Wide Bus) Flags

| Macro | Bit | القيمة | الوصف |
|---|---|---|---|
| `SPI_TX_DUAL` | 8 | `0x0100` | TX على سلكين |
| `SPI_TX_QUAD` | 9 | `0x0200` | TX على 4 أسلاك |
| `SPI_RX_DUAL` | 10 | `0x0400` | RX على سلكين |
| `SPI_RX_QUAD` | 11 | `0x0800` | RX على 4 أسلاك |
| `SPI_TX_OCTAL` | 13 | `0x2000` | TX على 8 أسلاك |
| `SPI_RX_OCTAL` | 14 | `0x4000` | RX على 8 أسلاك |

#### مجموعة الـ Advanced/Special Flags

| Macro | Bit | القيمة | الوصف |
|---|---|---|---|
| `SPI_CS_WORD` | 12 | `0x1000` | Toggle CS بعد كل word |
| `SPI_3WIRE_HIZ` | 15 | `0x8000` | High-Z turnaround في الـ 3-wire |
| `SPI_RX_CPHA_FLIP` | 16 | `0x10000` | قلب CPHA في RX-only transfers |
| `SPI_MOSI_IDLE_LOW` | 17 | `0x20000` | MOSI يفضل low وقت الـ idle |
| `SPI_MOSI_IDLE_HIGH` | 18 | `0x40000` | MOSI يفضل high وقت الـ idle |

#### الـ User-Space Boundary Mask

| Macro | القيمة | الوصف |
|---|---|---|
| `SPI_MODE_USER_MASK` | `0x7FFFF` (bits 0–18) | كل الـ bits المسموح بيها من user-space |

---

### Category 1: الـ Clock Polarity & Phase (CPOL/CPHA)

الـ SPI protocol بيحدد الـ clock behavior بـ bitين أساسيين — **CPOL** و**CPHA** — وده اللي بيفرق بين الـ 4 standard modes.

---

#### `SPI_CPHA` — Clock Phase

```c
#define SPI_CPHA    _BITUL(0)   /* clock phase */
```

بيحدد **متى** بيتأخذ السامبل على الـ data line بالنسبة للـ clock edge.
- `CPHA = 0`: السامبل يتأخذ على الـ **leading edge** (أول edge).
- `CPHA = 1`: السامبل يتأخذ على الـ **trailing edge** (تاني edge).

**الـ bit:** 0 (أقل bit).
**الاستخدام:** بيتحط مباشرة في `spi_device->mode` أو في `SPI_IOC_WR_MODE32` ioctl.

---

#### `SPI_CPOL` — Clock Polarity

```c
#define SPI_CPOL    _BITUL(1)   /* clock polarity */
```

بيحدد **الحالة الـ idle** للـ clock line.
- `CPOL = 0`: الـ clock idle على **low**.
- `CPOL = 1`: الـ clock idle على **high**.

**الـ bit:** 1.
**الاستخدام:** نفس `SPI_CPHA`، بيتحط في `mode` field.

---

### Category 2: الـ Standard SPI Modes (0–3)

الـ 4 modes الأساسية اللي كل SPI device بيحتاج واحد منها.

---

#### `SPI_MODE_0`

```c
#define SPI_MODE_0  (0 | 0)     /* CPOL=0, CPHA=0 — original MicroWire */
```

الـ mode الأكتر شيوعاً. الـ clock idle على low، والسامبل على الـ rising edge.
ده الـ default لأغلب flash chips زي W25Qxx وكتير من الـ ADCs.

---

#### `SPI_MODE_1`

```c
#define SPI_MODE_1  (0 | SPI_CPHA)  /* CPOL=0, CPHA=1 */
```

الـ clock idle على low، لكن السامبل على الـ falling edge.
بيستخدمه بعض الـ sensors زي ADXL345 في بعض الـ configurations.

---

#### `SPI_MODE_2`

```c
#define SPI_MODE_2  (SPI_CPOL | 0)  /* CPOL=1, CPHA=0 */
```

الـ clock idle على high، والسامبل على الـ falling edge (اللي هو الـ leading edge هنا).

---

#### `SPI_MODE_3`

```c
#define SPI_MODE_3  (SPI_CPOL | SPI_CPHA)  /* CPOL=1, CPHA=1 */
```

الـ clock idle على high، والسامبل على الـ rising edge (الـ trailing edge).
بيستخدمه مثلاً MAX31855 thermocouple digitizer.

---

#### `SPI_MODE_X_MASK`

```c
#define SPI_MODE_X_MASK (SPI_CPOL | SPI_CPHA)  /* = 0x3 */
```

**Mask** بيعزل bits الـ CPOL وCPHA من الـ mode word. بيتستخدم في الـ kernel لما بيعمل:

```c
/* extract just the clock mode bits */
u32 clock_mode = device->mode & SPI_MODE_X_MASK;
```

---

### Category 3: الـ Bus Behavior Flags

الـ flags دي بتتحكم في سلوك الـ bus نفسه — الـ chip select، اتجاه الـ bits، وبعض الـ special modes.

---

#### `SPI_CS_HIGH` — Active-High Chip Select

```c
#define SPI_CS_HIGH _BITUL(2)   /* chipselect active high? */
```

بشكل افتراضي، الـ CS active-low (بيروح low عشان يفعّل الـ device). الـ flag ده بيعكس ده — الـ CS بيروح **high** عشان يفعّل.
بيتستخدم مع بعض الـ legacy devices أو اللي بتشتغل بـ positive logic.

**Key detail:** الـ SPI controller driver لازم يدعم ده، غير كده بيرجع error في `spi_setup()`.

---

#### `SPI_LSB_FIRST` — LSB-First Bit Order

```c
#define SPI_LSB_FIRST   _BITUL(3)   /* per-word bits-on-wire */
```

بشكل افتراضي، الـ SPI بيبعت **MSB first**. الـ flag ده بيخلي كل word يتبعت من الـ **LSB**.
بعض الـ controllers مش بيدعموه hardware، فبيعمل software bit-reversal.

---

#### `SPI_3WIRE` — Bidirectional (Shared SI/SO)

```c
#define SPI_3WIRE   _BITUL(4)   /* SI/SO signals shared */
```

بدل ما يكون MOSI وMISO خطين منفصلين، بيستخدموا **خط واحد مشترك** (half-duplex). اتجاه الخط بيتغير بين الـ TX وRX phases.
مفيد لما تكون الـ pins شحيحة زي في بعض الـ microcontrollers الصغيرة.

---

#### `SPI_LOOP` — Loopback Mode

```c
#define SPI_LOOP    _BITUL(5)   /* loopback mode */
```

بيوصّل MOSI بـ MISO internally داخل الـ controller — بيتستخدم للـ **self-test** فقط. الـ hardware controller لازم يدعمه.

---

#### `SPI_NO_CS` — No Chip Select

```c
#define SPI_NO_CS   _BITUL(6)   /* 1 dev/bus, no chipselect */
```

لما يكون في device واحد بس على الـ bus ومش محتاج CS أصلاً، أو لما الـ CS راجع من software خارج الـ SPI subsystem. بيخلي الـ kernel ميلمسش أي CS line.

---

#### `SPI_READY` — Slave Flow Control

```c
#define SPI_READY   _BITUL(7)   /* slave pulls low to pause */
```

الـ slave device عنده output line إضافية بيلوّعها low لما مش ready لاستقبال data — نوع من الـ **hardware flow control**. نادر في الـ modern devices.

---

### Category 4: الـ Multi-Wire Wide Bus Flags

دي الـ flags الخاصة بـ **Dual SPI**، **Quad SPI (QSPI)**، و**Octal SPI** — المستخدمة كتير مع الـ NOR flash (زي ISSI، Micron، Winbond) والـ PSRAM.

---

#### `SPI_TX_DUAL` / `SPI_RX_DUAL`

```c
#define SPI_TX_DUAL _BITUL(8)   /* transmit with 2 wires */
#define SPI_RX_DUAL _BITUL(10)  /* receive with 2 wires */
```

الـ TX أو RX بيحصل على **سلكين في نفس الوقت** — بيضاعف الـ throughput.
في الـ NOR flash protocol، الأمر (command) بيتبعت standard (1-wire)، لكن الـ data بيتبعت/يتستقبل Dual.
الـ combination ده بيسمّى `1-1-2` أو `1-2-2` حسب الـ flash protocol.

---

#### `SPI_TX_QUAD` / `SPI_RX_QUAD`

```c
#define SPI_TX_QUAD _BITUL(9)   /* transmit with 4 wires */
#define SPI_RX_QUAD _BITUL(11)  /* receive with 4 wires */
```

الـ TX أو RX بيحصل على **4 أسلاك في نفس الوقت** — مثلاً W25Q flash في Quad mode. بيوصل سرعة القراءة لـ 4x مقارنة بالـ standard SPI. اللي بيبعت الأوامر هو SPI_NOR driver في الـ `drivers/mtd/spi-nor/`.

---

#### `SPI_TX_OCTAL` / `SPI_RX_OCTAL`

```c
#define SPI_TX_OCTAL    _BITUL(13)  /* transmit with 8 wires */
#define SPI_RX_OCTAL    _BITUL(14)  /* receive with 8 wires */
```

الـ TX أو RX على **8 أسلاك** — الـ Octal SPI (xSPI standard). بيستخدمه الـ high-speed flash زي Macronix MX25UM51345G وبعض الـ HyperBus devices. بيوصل throughput عالي جداً بـ DDR.

**ملاحظة مهمة:** الـ bits 8, 9 للـ TX و 10, 11 للـ RX — لاحظ الـ gap: bit 12 هو `SPI_CS_WORD` وmit 13, 14 هما TX/RX Octal.

---

### Category 5: الـ Advanced Special-Case Flags

---

#### `SPI_CS_WORD` — Per-Word Chip Select Toggle

```c
#define SPI_CS_WORD _BITUL(12)  /* toggle cs after each word */
```

بعد كل **word** (مش بعد كل transfer)، الـ CS بيتـ toggle. بيتستخدم مع بعض الـ legacy protocols اللي بتتطلع كل word على حدة.

**Key detail:** محتاج support من الـ hardware controller. لو مش مدعوم، الـ `spi_setup()` بترجع `-EINVAL`.

---

#### `SPI_3WIRE_HIZ` — High-Impedance Turnaround

```c
#define SPI_3WIRE_HIZ   _BITUL(15)  /* high impedance turnaround */
```

في الـ 3-wire mode، لما بنعمل switch من TX لـ RX، السلك المشترك لازم يروح **high-impedance** لفترة قبل ما الـ slave يبدأ يبعت. ده بيمنع الـ contention على الـ bus.

---

#### `SPI_RX_CPHA_FLIP` — CPHA Flip for RX-only Transfers

```c
#define SPI_RX_CPHA_FLIP    _BITUL(16)  /* flip CPHA on Rx only xfer */
```

بعض الـ devices (خصوصاً في الـ Dual/Quad mode) بتتطلب إن الـ CPHA يتعكس في الـ RX-only phase مقارنة بالـ TX phase. ده work-around لـ quirk في بعض الـ flash controllers وSPI NOR devices.

---

#### `SPI_MOSI_IDLE_LOW` / `SPI_MOSI_IDLE_HIGH`

```c
#define SPI_MOSI_IDLE_LOW   _BITUL(17)  /* leave MOSI line low when idle */
#define SPI_MOSI_IDLE_HIGH  _BITUL(18)  /* leave MOSI line high when idle */
```

بيحدد الحالة اللي MOSI line بيفضل عليها لما ما فيش transfer جاري. افتراضياً مش محدد (controller-dependent).
- `SPI_MOSI_IDLE_LOW`: MOSI يفضل `0` في الـ idle.
- `SPI_MOSI_IDLE_HIGH`: MOSI يفضل `1` في الـ idle.

**لازم واحد بس:** الاتنين مع بعض منطقياً متعارضين. الـ kernel بيعمل validate في `spi_setup()`.

---

### Category 6: الـ Boundary Mask

---

#### `SPI_MODE_USER_MASK`

```c
#define SPI_MODE_USER_MASK  (_BITUL(19) - 1)   /* = 0x7FFFF, bits 0–18 */
```

ده الـ **mask الرسمي** اللي بيفصل بين الـ user-space flags والـ kernel-internal flags.

**الـ design:**
- الـ bits من 0 لـ 18 (الـ 19 bit الأدنى) = user-space flags (المعرّفة في الملف ده).
- الـ bits من أعلى = `SPI_MODE_KERNEL_MASK` (معرّفة في `include/linux/spi/spi.h`).
- الاتنين **لازم ما يتداخلوش** — في `static_assert` في الـ kernel بيضمن ده.

**الاستخدام في الـ kernel:**

```c
/* in spi_setup() — reject any unknown user-space bits */
if (spi->mode & ~SPI_MODE_USER_MASK)
    return -EINVAL;
```

**الاستخدام من user-space (عبر ioctl):**

```c
uint32_t mode = SPI_MODE_0 | SPI_RX_QUAD;
ioctl(fd, SPI_IOC_WR_MODE32, &mode);
```

---

### الـ _BITUL Macro — الأساس اللي كل حاجة بُنيت عليه

كل الـ macros فوق بتستخدم `_BITUL(x)` اللي متعرّفة في `include/uapi/linux/const.h`:

```c
#define _UL(x)      (_AC(x, UL))          /* cast to unsigned long */
#define _BITUL(x)   (_UL(1) << (x))       /* bit x as unsigned long */
```

**ليه `_BITUL` بدل `(1 << x)`؟**
- `1 << x` في C بيكون `int` — ممكن يعمل UB لو `x >= 31` على 32-bit.
- `_BITUL(x)` بيضمن الـ type هو `unsigned long` — آمن على 32-bit و64-bit.
- مهم في الـ UAPI لأن الـ header بيتستخدم في user-space كمان.

---

### علاقة الـ UAPI Header بـ Kernel Header

```
include/uapi/linux/spi/spi.h          ← الملف ده (user-visible bits 0–18)
        ↕ (لازم ما يتداخلوش)
include/linux/spi/spi.h               ← SPI_MODE_KERNEL_MASK (high bits)
```

الـ kernel header بيعرّف flags إضافية للـ kernel-internal use (زي `SPI_NO_CS_INTERNAL` أو flags خاصة بالـ controller) في الـ bits العالية، بعيداً عن الـ `SPI_MODE_USER_MASK`.

---

### سيناريو عملي — ضبط Quad SPI NOR Flash

```c
/* User-space: open /dev/spidev0.0 and configure for Quad-read */
int fd = open("/dev/spidev0.0", O_RDWR);

uint32_t mode = SPI_MODE_0      /* CPOL=0, CPHA=0          */
              | SPI_RX_QUAD;    /* receive on 4 wires       */

/* Validate against user mask — optional but good practice */
if (mode & ~SPI_MODE_USER_MASK) {
    /* shouldn't happen here, but defensive check */
    return -EINVAL;
}

ioctl(fd, SPI_IOC_WR_MODE32, &mode);
```

في الـ kernel (`spi_setup()`):

```c
/* Kernel validates the mode before passing to controller driver */
if (spi->mode & ~(SPI_MODE_USER_MASK | SPI_MODE_KERNEL_MASK))
    return -EINVAL;

/* Controller driver checks if it supports quad rx */
if ((spi->mode & SPI_RX_QUAD) && !controller_supports_quad(ctlr))
    return -EINVAL;
```
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. مدخلات الـ debugfs

الـ SPI subsystem بيكتب statistics ومعلومات runtime في `/sys/kernel/debug/spi/`.

```bash
# اعرض كل الـ SPI controllers الموجودة
ls /sys/kernel/debug/spi/

# اقرأ statistics الـ controller (مثلاً spi0)
cat /sys/kernel/debug/spi/spi0/statistics

# اقرأ statistics الـ device المتصل (مثلاً spi0.0)
cat /sys/kernel/debug/spi/spi0/spi0.0/statistics
```

**تفسير الـ output:**

```
messages:            1024     # عدد الـ spi_message اللي اتبعتت
transfers:           2048     # عدد الـ spi_transfer
errors:              3        # errors أثناء النقل — لازم تبقى صفر
timedout:            0        # timeout — لو مش صفر فيه مشكلة hardware
spi_sync:            1020     # كام مرة استخدم الـ driver spi_sync()
spi_sync_immediate:  800      # منها كام مرة اتنفذت فوراً بدون queue
spi_async:           4        # استخدامات spi_async()
bytes:               65536    # إجمالي bytes
bytes_tx:            32768
bytes_rx:            32768
```

الـ `struct spi_statistics` بتتعرّف في `include/linux/spi/spi.h` وبتتحدّث per-CPU بـ `u64_stats_sync` لتجنب الـ race conditions.

---

#### 2. مدخلات الـ sysfs

```bash
# اعرض كل الـ SPI devices المسجّلة
ls /sys/bus/spi/devices/

# SPI mode flags لـ device معينة (القيمة hex)
cat /sys/bus/spi/devices/spi0.0/spi-dev/spidev0.0/device/of_node/spi-max-frequency

# اقرأ الـ modalias عشان تعرف الـ driver اللي بيخدم الـ device
cat /sys/bus/spi/devices/spi0.0/modalias

# max_speed_hz المضبوطة
cat /sys/bus/spi/devices/spi0.0/max_speed_hz 2>/dev/null ||
  grep -r "max_speed_hz" /sys/bus/spi/devices/spi0.0/

# bits_per_word
cat /sys/bus/spi/devices/spi0.0/bits_per_word 2>/dev/null

# اعرض الـ driver المرتبط بالـ device
ls -la /sys/bus/spi/devices/spi0.0/driver

# اعرض الـ chipselect GPIO إن وُجد
cat /sys/bus/spi/devices/spi0.0/cs_gpio 2>/dev/null
```

**الـ sysfs المهمة لكل controller:**

```bash
# اقرأ mode الـ controller
cat /sys/class/spi_master/spi0/statistics 2>/dev/null

# عدد الـ chipselects المدعومة
cat /sys/class/spi_master/spi0/num_chipselect 2>/dev/null
```

---

#### 3. الـ ftrace — Tracepoints والـ Events

الـ SPI subsystem عنده tracepoints معرّفة في `include/trace/events/spi.h`.

```bash
# اعرض كل الـ SPI trace events المتاحة
ls /sys/kernel/tracing/events/spi/

# الـ events الأساسية:
# spi_controller_busy / spi_controller_idle
# spi_message_submit / spi_message_start / spi_message_done
# spi_transfer_start / spi_transfer_stop

# فعّل كل الـ SPI events دفعة واحدة
echo 1 > /sys/kernel/tracing/events/spi/enable

# أو فعّل event محدد بس
echo 1 > /sys/kernel/tracing/events/spi/spi_transfer_start/enable
echo 1 > /sys/kernel/tracing/events/spi/spi_transfer_stop/enable
echo 1 > /sys/kernel/tracing/events/spi/spi_message_done/enable

# شغّل الـ tracer
echo 1 > /sys/kernel/tracing/tracing_on

# نفّذ العملية اللي بتحاول تـ debug فيها...

# اقرأ الـ trace
cat /sys/kernel/tracing/trace

# وقّف الـ tracing
echo 0 > /sys/kernel/tracing/tracing_on
echo 0 > /sys/kernel/tracing/events/spi/enable
```

**مثال على output الـ ftrace:**

```
     kworker/0:2-85    [000] .....  123.456789: spi_message_submit: spi0.0
     kworker/0:2-85    [000] .....  123.456800: spi_transfer_start: spi0.0 len=4 speed=1000000
     kworker/0:2-85    [000] .....  123.456850: spi_transfer_stop:  spi0.0 len=4
     kworker/0:2-85    [000] .....  123.456855: spi_message_done:   spi0.0 status=0
```

لو `status` مش صفر — فيه error في الـ transfer.
الفرق بين `spi_transfer_start` و `spi_transfer_stop` هو الـ latency الفعلي.

---

#### 4. الـ printk والـ Dynamic Debug

```bash
# فعّل dynamic debug لكل الـ SPI core
echo "file drivers/spi/spi.c +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل debug لـ SPI controller معين (مثلاً spi-bcm2835)
echo "file drivers/spi/spi-bcm2835.c +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل debug لكل الـ SPI drivers دفعة واحدة
echo "module spi* +p" > /sys/kernel/debug/dynamic_debug/control

# اعرض الـ dynamic debug entries الـ SPI المفعّلة
grep spi /sys/kernel/debug/dynamic_debug/control | grep "=p"

# تابع الـ kernel log في real-time
dmesg -w | grep -i spi

# أو بـ loglevel مرفوع
echo 8 > /proc/sys/kernel/printk
dmesg -w
```

**الـ SPI mode flags كـ hex — سهّل قراءتها:**

```bash
# SPI_MODE_0 = 0x00, SPI_MODE_3 = 0x03
# SPI_CS_HIGH = 0x04, SPI_LSB_FIRST = 0x08
# SPI_TX_DUAL = 0x100, SPI_TX_QUAD = 0x200
# SPI_RX_DUAL = 0x400, SPI_RX_QUAD = 0x800
# SPI_MODE_USER_MASK = 0x7FFFF (bits 0-18)
# SPI_MODE_KERNEL_MASK = 0xE0000000 (bits 29-31)
```

---

#### 5. كونفيج الـ Kernel للـ Debugging

| Config Option | الوظيفة |
|---|---|
| `CONFIG_SPI_DEBUG` | يفعّل `dev_dbg()` في SPI core وبعض الـ controllers |
| `CONFIG_DEBUG_SPI_STUB` | يوفر stub controller للتيست بدون hardware |
| `CONFIG_DYNAMIC_DEBUG` | لازم يكون مفعّل عشان dynamic debug يشتغل |
| `CONFIG_TRACING` | لازم للـ ftrace tracepoints |
| `CONFIG_SPI_LOOPBACK_TEST` | module بيعمل loopback test على الـ controller |
| `CONFIG_FAULT_INJECTION` | لاختبار error paths في SPI |
| `CONFIG_LOCKDEP` | يكشف deadlocks في الـ SPI locks |
| `CONFIG_PROVE_LOCKING` | تحقق من صحة الـ locking في الـ SPI transfers |
| `CONFIG_KASAN` | يكشف memory corruption في الـ SPI buffers |
| `CONFIG_KCSAN` | يكشف data races في per-CPU statistics |

```bash
# تحقق إن الـ config مفعّل في الـ kernel الحالي
zcat /proc/config.gz | grep -E "CONFIG_(SPI|DEBUG_SPI|SPI_DEBUG)"
```

---

#### 6. الـ spidev وأدوات الـ Userspace

```bash
# spidev_test — أداة kernel للاختبار
# موجودة في tools/spi/spidev_test.c

# بناء الأداة
cd /path/to/kernel && make -C tools/spi

# اختبار loopback (وصّل MOSI بـ MISO)
./spidev_test -D /dev/spidev0.0 -v -p "Hello\x00"

# ضبط السرعة والـ mode
./spidev_test -D /dev/spidev0.0 -s 500000 -m 0 -v -p "\xAA\x55\xFF\x00"

# قراءة وكتابة بـ Python
python3 -c "
import spidev
spi = spidev.SpiDev()
spi.open(0, 0)
spi.max_speed_hz = 1000000
spi.mode = 0b00  # SPI_MODE_0
resp = spi.xfer2([0xAA, 0x55, 0x00, 0x00])
print('Response:', [hex(x) for x in resp])
spi.close()
"

# اقرأ الـ SPI mode الحالي عبر ioctl
python3 -c "
import fcntl, struct
SPI_IOC_RD_MODE32 = 0x80046b05
fd = open('/dev/spidev0.0', 'rb')
mode = struct.unpack('I', fcntl.ioctl(fd, SPI_IOC_RD_MODE32, b'\x00'*4))[0]
print(f'SPI mode: 0x{mode:08X}')
fd.close()
"
```

---

#### 7. جدول رسائل الـ Error الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `spi_master spi0: setup: invalid mode bits` | بتضبط bits من الـ `SPI_MODE_KERNEL_MASK` من الـ userspace | شيل الـ bits المحجوزة (29-31) من الـ mode |
| `spi0.0: transfer failed` | الـ controller رجّع error من `transfer_one` | شيك على الـ clock، الـ CS، والـ power |
| `spi0: message rejected` | الـ message جالها error قبل ما تبدأ | تحقق من `spi_message_init()` والـ transfer buffers |
| `spi0.0: timeout` | الـ device ما ردّش في الوقت | شيك على الـ READY line (SPI_READY)، أو زوّد الـ timeout |
| `SPI_MODE_USER_MASK & SPI_MODE_KERNEL_MASK must not overlap` | static_assert failed — conflict في الـ mode bits | حدّث `SPI_MODE_USER_MASK` أو `SPI_MODE_KERNEL_MASK` |
| `spi0: Unable to set bits per word` | الـ controller مش داعم الـ bits_per_word المطلوبة | تحقق من `bits_per_word_mask` في الـ controller |
| `spi0.0: can't use DMA` | الـ DMA buffers مش aligned أو الـ DMA channel مش متاح | استخدم `spi_transfer.cs_change = 1` أو عدّل الـ alignment |
| `spi0: failed to get CS GPIO` | الـ DT بيشاور على GPIO مش موجود | تحقق من الـ Device Tree وعدد pins المتاحة |
| `spi0.0: chipselect not configured` | الـ `chip_select` value أكبر من `num_chipselect` | تحقق من الـ DT `reg` property |
| `spi0: transfer list is empty` | بعتّ `spi_message` بدون `spi_transfer` | أضف `spi_message_add_tail()` قبل `spi_sync()` |

---

#### 8. أماكن استخدام dump_stack() و WARN_ON()

```c
/* في spi_setup() — تحقق من صحة الـ mode bits */
static int spi_setup(struct spi_device *spi)
{
    /* تحقق إن الـ user mode bits مش بتتداخل مع kernel bits */
    WARN_ON(spi->mode & SPI_MODE_KERNEL_MASK);

    /* تحقق إن SPI_MOSI_IDLE_LOW و SPI_MOSI_IDLE_HIGH مش متفعّلين مع بعض */
    if ((spi->mode & SPI_MOSI_IDLE_LOW) && (spi->mode & SPI_MOSI_IDLE_HIGH)) {
        dev_err(&spi->dev, "contradictory MOSI idle state\n");
        WARN_ON(1);
        return -EINVAL;
    }
    /* ... */
}

/* في transfer_one_message() — تحقق من الـ transfer */
static int spi_transfer_one_message(struct spi_controller *ctlr,
                                    struct spi_message *msg)
{
    struct spi_transfer *xfer;
    list_for_each_entry(xfer, &msg->transfers, transfer_list) {
        /* لو الـ buffer NULL والـ len مش صفر — مشكلة */
        WARN_ON(!xfer->tx_buf && !xfer->rx_buf && xfer->len);

        /* تحقق إن SPI_TX_QUAD و SPI_TX_DUAL مش مضبوطين مع بعض */
        WARN_ON((xfer->tx_nbits & SPI_TX_QUAD) &&
                (xfer->tx_nbits & SPI_TX_DUAL));
    }
}

/* dump_stack() في حالة unexpected failure */
static void spi_finalize_current_message(struct spi_controller *ctlr)
{
    struct spi_message *mesg = ctlr->cur_msg;
    if (mesg->status && mesg->status != -EINPROGRESS) {
        dev_err(&ctlr->dev, "unexpected status %d\n", mesg->status);
        dump_stack(); /* اعرض call stack للـ debugging */
    }
}
```

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State بيطابق الـ Kernel State

```bash
# تحقق إن الـ SPI clock polarity صح (CPOL)
# SPI_MODE_0: CPOL=0 → الـ clock idle LOW
# SPI_MODE_2: CPOL=1 → الـ clock idle HIGH
cat /sys/bus/spi/devices/spi0.0/modalias

# قارن الـ max_speed_hz في الـ DT مع اللي بيطلبه الـ driver
cat /proc/device-tree/spi@7e204000/spi-max-frequency 2>/dev/null | od -An -tx4

# تحقق إن الـ chip select polarity صح
# SPI_CS_HIGH (bit 2) = CS active HIGH
# الـ default = CS active LOW
python3 -c "
mode_hex = 0x04  # مثلاً
print('CS_HIGH:', bool(mode_hex & 0x04))
print('CPOL:', bool(mode_hex & 0x02))
print('CPHA:', bool(mode_hex & 0x01))
"
```

---

#### 2. Register Dump

```bash
# اقرأ registers الـ SPI controller مباشرة (مثال BCM2835 - Raspberry Pi)
# Base address للـ SPI0: 0x3F204000 (Pi 3) أو 0xFE204000 (Pi 4)

# استخدم devmem2 لو متاح
devmem2 0xFE204000 w   # CS register
devmem2 0xFE204004 w   # FIFO register
devmem2 0xFE204008 w   # CLK register
devmem2 0xFE20400C w   # DLEN register
devmem2 0xFE204010 w   # LTOH register
devmem2 0xFE204014 w   # DC register

# أو عبر /dev/mem (بيحتاج root)
python3 -c "
import mmap, struct
BASE = 0xFE204000
SIZE = 0x1000
with open('/dev/mem', 'rb') as f:
    mem = mmap.mmap(f.fileno(), SIZE, offset=BASE,
                    prot=mmap.PROT_READ, flags=mmap.MAP_SHARED)
    for offset, name in [(0,'CS'), (4,'FIFO'), (8,'CLK'), (12,'DLEN')]:
        val = struct.unpack_from('<I', mem, offset)[0]
        print(f'{name} (0x{BASE+offset:08X}): 0x{val:08X}')
    mem.close()
"

# استخدم io command لو موجود
io -4 0xFE204000
```

**تفسير الـ BCM2835 SPI CS Register:**

```
Bit 0-1: CS (chip select)
Bit 2:   CPHA
Bit 3:   CPOL
Bit 4:   CLEAR_TX (write 1 to clear TX FIFO)
Bit 5:   CLEAR_RX (write 1 to clear RX FIFO)
Bit 6:   CSPOL (CS polarity)
Bit 7:   TA (Transfer Active)
Bit 8:   DMAEN
Bit 9:   INTD (interrupt on done)
Bit 10:  INTR (interrupt on RX)
Bit 11:  ADCS (auto deassert CS)
Bit 16:  DONE (transfer done)
Bit 17:  RXD (RX FIFO not empty)
Bit 18:  TXD (TX FIFO has space)
```

---

#### 3. نصائح الـ Logic Analyzer والـ Oscilloscope

**إعداد الـ Logic Analyzer:**

```
Channels المطلوبة:
  CH0 → SCLK    (الـ clock)
  CH1 → MOSI    (Master Out Slave In)
  CH2 → MISO    (Master In Slave Out)
  CH3 → CS/SS   (Chip Select)
  CH4 → READY   (لو الـ device بيستخدم SPI_READY)

Trigger: على falling edge الـ CS (CS active LOW)
Sample Rate: على الأقل 10x أكتر من الـ SPI clock
             مثال: SPI @ 1MHz → sample بـ 10MHz على الأقل
```

**ما تدور عليه:**

```
1. SPI_MODE_0: CPOL=0, CPHA=0
   - الـ clock idle LOW
   - البيانات بتتقرأ على rising edge
   - بتتحط على falling edge

2. SPI_MODE_3: CPOL=1, CPHA=1
   - الـ clock idle HIGH
   - البيانات بتتقرأ على falling edge

3. SPI_CS_HIGH: الـ CS بيبقى HIGH أثناء الـ transfer
   الـ default هو LOW

4. SPI_LSB_FIRST: الـ bit الأقل أهمية بييجي الأول
   ابحث عن reversed bit pattern في الـ analyzer

5. SPI_TX_DUAL/QUAD: كل clock edge بتنتقل فيه 2 أو 4 bits
   هتشوف MOSI وSIO1 بيتحركوا مع بعض

6. SPI_MOSI_IDLE_LOW/HIGH: حالة MOSI بين الـ transfers
```

**للـ oscilloscope:**

```
- قيس الـ rise time للـ SCLK: لازم يكون أقل من 10% من الـ clock period
- تحقق من الـ voltage levels: SCLK, MOSI, MISO لازم يوصلوا لـ VDD الـ device
- قيس الـ CS-to-SCLK delay: الـ cs_setup delay
- قيس الـ SCLK-to-CS delay: الـ cs_hold delay
- شيك على الـ ringing والـ reflections لو الـ SPI clock فوق 10MHz
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماط الـ Kernel Log

| مشكلة الـ Hardware | نمط الـ Kernel Log | التشخيص |
|---|---|---|
| غلط في الـ SPI mode (CPOL/CPHA) | بيتقرأ بيانات غلط بدون error | قارن الـ mode في الـ DT مع datasheet الـ device |
| الـ clock بطيء جداً | `timedout` في الـ statistics | ارفع الـ max_speed_hz أو شيك الـ pull-up resistors |
| الـ CS مش بيشتغل | الـ device ما بيرد | تحقق من `SPI_CS_HIGH` flag وقطبية الـ GPIO |
| الـ MISO مفصول / floating | بيقرأ `0xFF` أو `0x00` دايماً | تحقق من التوصيل وإن الـ device مش في reset |
| نقص في الـ power | transfers ناجحة أحياناً وفاشلة أحياناً | قيس VDD أثناء الـ transfer على الـ oscilloscope |
| الـ SPI_READY line مش شغّال | `timedout` متكرر مع devices بطيئة | فعّل `SPI_READY` flag في الـ mode أو استخدم delays |
| تداخل مع SPI_3WIRE | MISO وMOSI مش بيتشاركوا صح | تحقق من `SPI_3WIRE` flag والـ turnaround timing |
| Octal SPI بيفشل | errors في كل transfer | تحقق إن الـ controller داعم `SPI_TX_OCTAL` و`SPI_RX_OCTAL` |

---

#### 5. Device Tree Debugging

```bash
# اعرض الـ DT node للـ SPI controller
cat /proc/device-tree/spi@7e204000/compatible 2>/dev/null
# مثال output: brcm,bcm2835-spi

# تحقق من الـ SPI child devices في الـ DT
ls /proc/device-tree/spi@7e204000/

# اقرأ spi-max-frequency من الـ DT مباشرة
hexdump -C /proc/device-tree/spi@7e204000/spidev@0/spi-max-frequency

# استخدم dtc لتحويل الـ DTB لصيغة قابلة للقراءة
dtc -I dtb -O dts /sys/firmware/fdt 2>/dev/null | grep -A 20 "spi@"

# تحقق من الـ DT overlays المحمّلة
ls /proc/device-tree/chosen/overlays/ 2>/dev/null
cat /proc/device-tree/chosen/bootargs | tr '\0' '\n'

# مقارنة الـ DT مع الـ kernel state
echo "DT max-freq:"
od -An -tu4 /proc/device-tree/spi@7e204000/spidev@0/spi-max-frequency 2>/dev/null
echo "Kernel max-freq:"
cat /sys/bus/spi/devices/spi0.0/max_speed_hz 2>/dev/null
```

**DT properties المهمة للـ SPI:**

```dts
&spi0 {
    status = "okay";
    cs-gpios = <&gpio 8 GPIO_ACTIVE_LOW>;  /* CS active LOW = default */

    flash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;                          /* chip select index */
        spi-max-frequency = <50000000>;     /* max 50MHz */
        spi-tx-bus-width = <4>;             /* SPI_TX_QUAD */
        spi-rx-bus-width = <4>;             /* SPI_RX_QUAD */
        /* spi-cs-high;                     لو CS active HIGH */
        /* spi-lsb-first;                   لو LSB first */
        /* spi-3wire;                        لو 3-wire mode */
    };
};
```

---

### Practical Commands

#### أوامر جاهزة للنسخ

```bash
#!/bin/bash
# SPI Debug Toolkit — انسخ واستخدم مباشرة

SPI_DEV="/dev/spidev0.0"
SPI_BUS="spi0"
SPI_DEVICE="spi0.0"

# ===== 1. تحقق سريع من الـ SPI subsystem =====
echo "=== SPI Devices ===" && ls /sys/bus/spi/devices/
echo "=== SPI Masters ===" && ls /sys/class/spi_master/

# ===== 2. اقرأ statistics الـ SPI =====
echo "=== Controller Stats ==="
cat /sys/kernel/debug/spi/${SPI_BUS}/statistics 2>/dev/null || \
  echo "debugfs not mounted. Run: mount -t debugfs none /sys/kernel/debug"

echo "=== Device Stats ==="
cat /sys/kernel/debug/spi/${SPI_BUS}/${SPI_DEVICE}/statistics 2>/dev/null

# ===== 3. فعّل كل SPI tracing =====
echo "Enabling SPI tracing..."
echo 1 > /sys/kernel/tracing/events/spi/enable
echo 1 > /sys/kernel/tracing/tracing_on
sleep 5   # شغّل تطبيقك هنا
echo 0 > /sys/kernel/tracing/tracing_on
echo "=== Trace Output ==="
cat /sys/kernel/tracing/trace | grep spi | head -50
echo 0 > /sys/kernel/tracing/events/spi/enable

# ===== 4. فعّل dynamic debug للـ SPI core =====
echo "file drivers/spi/spi.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/spi/spi-bcm2835.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "Dynamic debug enabled. Watch: dmesg -w | grep spi"

# ===== 5. اختبر الـ SPI loopback (MOSI متوصل بـ MISO) =====
if [ -c "$SPI_DEV" ]; then
    echo "=== Loopback Test ==="
    # spidev_test من tools/spi/
    spidev_test -D $SPI_DEV -v -l -s 500000 -p "\xAA\x55\x0F\xF0\xFF\x00"
else
    echo "spidev not available. Add spidev to DT or enable CONFIG_SPI_SPIDEV"
fi

# ===== 6. فحص SPI mode الحالي =====
echo "=== Current SPI Mode ==="
python3 -c "
import os, fcntl, struct
SPI_IOC_RD_MODE32 = 0x80046b05
SPI_IOC_RD_MAX_SPEED_HZ = 0x80046b04
SPI_IOC_RD_BITS_PER_WORD = 0x80016b03

dev = '$SPI_DEV'
if not os.path.exists(dev):
    print(f'{dev} not found')
    exit(1)

with open(dev, 'rb') as f:
    mode = struct.unpack('I', fcntl.ioctl(f, SPI_IOC_RD_MODE32, b'\x00'*4))[0]
    speed = struct.unpack('I', fcntl.ioctl(f, SPI_IOC_RD_MAX_SPEED_HZ, b'\x00'*4))[0]
    bpw = struct.unpack('B', fcntl.ioctl(f, SPI_IOC_RD_BITS_PER_WORD, b'\x00'))[0]

print(f'Mode: 0x{mode:08X}')
print(f'  CPHA (SPI_MODE bit0): {bool(mode & 0x01)}')
print(f'  CPOL (SPI_MODE bit1): {bool(mode & 0x02)}')
print(f'  CS_HIGH:              {bool(mode & 0x04)}')
print(f'  LSB_FIRST:            {bool(mode & 0x08)}')
print(f'  3WIRE:                {bool(mode & 0x10)}')
print(f'  LOOP:                 {bool(mode & 0x20)}')
print(f'  NO_CS:                {bool(mode & 0x40)}')
print(f'  TX_DUAL:              {bool(mode & 0x100)}')
print(f'  TX_QUAD:              {bool(mode & 0x200)}')
print(f'  RX_DUAL:              {bool(mode & 0x400)}')
print(f'  RX_QUAD:              {bool(mode & 0x800)}')
print(f'  CS_WORD:              {bool(mode & 0x1000)}')
print(f'  TX_OCTAL:             {bool(mode & 0x2000)}')
print(f'  RX_OCTAL:             {bool(mode & 0x4000)}')
print(f'  3WIRE_HIZ:            {bool(mode & 0x8000)}')
print(f'  RX_CPHA_FLIP:         {bool(mode & 0x10000)}')
print(f'  MOSI_IDLE_LOW:        {bool(mode & 0x20000)}')
print(f'  MOSI_IDLE_HIGH:       {bool(mode & 0x40000)}')
print(f'Max Speed: {speed/1e6:.3f} MHz')
print(f'Bits/Word: {bpw}')
"
```

---

#### مثال على output وتفسيره

**output من ftrace:**

```
# tracer: nop
#
#           TASK-PID   CPU#  ||||    TIMESTAMP  FUNCTION
          python3-1234 [000] .....  456.789012: spi_message_submit: spi0.0
          python3-1234 [000] .....  456.789015: spi_controller_busy: spi0
       kworker/0:1-56  [000] .....  456.789020: spi_message_start: spi0.0
       kworker/0:1-56  [000] .....  456.789025: spi_transfer_start: spi0.0 len=4 speed_hz=1000000 bpw=8 mode=0x0 cs_change=0 delay_usecs=0
       kworker/0:1-56  [000] .....  456.789145: spi_transfer_stop:  spi0.0 len=4
       kworker/0:1-56  [000] .....  456.789150: spi_message_done:   spi0.0 status=0 frame=1 actual=4
      python3-1234  [000] .....  456.789155: spi_controller_idle: spi0
```

**تفسير:**
- من `spi_transfer_start` لـ `spi_transfer_stop`: **120 µs** لـ transfer بحجم 4 bytes عند 1MHz — طبيعي (32 µs للبيانات + overhead)
- لو الفرق ده أكتر من 10x من الوقت المتوقع — في مشكلة في الـ hardware أو الـ driver بيعمل busy-wait
- `status=0` يعني success — أي قيمة تانية فيها مشكلة
- `actual=4` = 4 bytes اتنقلت فعلاً

**output من statistics:**

```bash
$ cat /sys/kernel/debug/spi/spi0/statistics
messages:            10240
transfers:           20480
errors:              0        # ممتاز
timedout:            2        # دي مشكلة — شيك hardware
spi_sync:            10235
spi_sync_immediate:  8192     # 80% بتشتغل في نفس الـ context = جيد
spi_async:           5
bytes:               1048576
bytes_tx:            524288
bytes_rx:            524288
```

الـ `timedout: 2` يعني إن الـ device تأخر مرتين — دور على `SPI_READY` line أو زوّد الـ cs_hold delay في الـ DT.
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Flash NOR على Industrial Gateway بـ RK3562 — بيانات فاسدة بعد الكتابة

#### العنوان
SPI NOR Flash يرجع بيانات غلط بعد write على industrial gateway.

#### السياق
شركة بتبني industrial gateway بـ RK3562 للـ SCADA systems. الـ gateway بيخزن configuration في SPI NOR Flash (Winbond W25Q128). الـ flash متوصل على SPI0 بـ Quad mode عشان الـ throughput. بعد كل firmware update، الـ configuration data بيتقرأ غلط.

#### المشكلة
الـ driver بيكتب البيانات بـ `SPI_TX_QUAD` لكن بيقرأها بـ standard SPI. النتيجة: بيانات متشوهة لأن الـ flash device في حالة quad read بيتوقع `SPI_RX_QUAD` في الـ read transaction.

```c
/* الكود الغلط في الـ driver */
struct spi_ioc_transfer xfer = {
    .tx_buf = (unsigned long)write_buf,
    .len    = len,
    .speed_hz = 50000000,
    .bits_per_word = 8,
    /* SPI_TX_QUAD موجود للكتابة */
};
/* لكن في الـ read: */
struct spi_ioc_transfer rxfer = {
    .rx_buf = (unsigned long)read_buf,
    .len    = len,
    /* SPI_RX_QUAD ناقص! */
};
```

#### التحليل
الفايل `include/uapi/linux/spi/spi.h` بيعرّف:

```c
#define SPI_TX_QUAD   _BITUL(9)   /* transmit with 4 wires */
#define SPI_RX_QUAD   _BITUL(11)  /* receive with 4 wires */
```

الـ bits مستقلة تمامًا — `SPI_TX_QUAD` على bit 9 و`SPI_RX_QUAD` على bit 11. لما الـ flash يدخل quad mode بعد write command، بيتوقع إن الـ host يقرأ بـ 4 wires. لو الـ `spi_ioc_transfer.bits_per_word` أو الـ `mode` field مش شايل `SPI_RX_QUAD`، الـ controller بيبعت على wire واحدة بس، والـ flash بيرد على 4 وايرات — فالبيانات بتتداخل.

#### الحل
```c
/* userspace ioctl fix */
struct spi_ioc_transfer rxfer = {
    .rx_buf       = (unsigned long)read_buf,
    .len          = len,
    .speed_hz     = 50000000,
    .bits_per_word = 8,
    /* لازم تضاف في spi_device->mode قبل open */
};

/* في device tree */
/*
&spi0 {
    flash@0 {
        compatible = "winbond,w25q128";
        spi-max-frequency = <50000000>;
        spi-rx-bus-width = <4>;   // يفعّل SPI_RX_QUAD في الكيرنل
        spi-tx-bus-width = <4>;   // يفعّل SPI_TX_QUAD
    };
};
*/
```

لو الـ userspace بيتحكم مباشرة بـ ioctl:
```bash
# تحقق من الـ mode الحالي
python3 -c "
import spidev
spi = spidev.SpiDev()
spi.open(0, 0)
print(hex(spi.mode))  # لازم يشمل SPI_RX_QUAD = bit 11
"
```

#### الدرس المستفاد
**الـ `SPI_TX_QUAD` و`SPI_RX_QUAD` bits مستقلة تمامًا**. لازم تحدد الاتجاهين بشكل منفصل. Flash NOR في quad mode بيتوقع quad read بعد كل quad write — لو نسيت `SPI_RX_QUAD` في الـ mode، البيانات بتتخرب بصمت.

---

### السيناريو 2: Android TV Box بـ Allwinner H616 — شاشة بتفلش مع SPI display

#### العنوان
SPI-connected display controller يعطي flicker على Android TV box.

#### السياق
Android TV box بـ Allwinner H616 بيستخدم small SPI display كـ status screen (128x64 OLED). الـ display controller (SSD1306) محتاج SPI_MODE_0. المطوّر بدّل الـ mode لـ SPI_MODE_3 ظنًا منه إن ده أسرع.

#### المشكلة
الشاشة بتفلش وبتعرض garbage data. الـ SSD1306 sensitive جدًا لـ clock phase وpolarity.

#### التحليل
من الفايل:
```c
#define SPI_CPHA    _BITUL(0)  /* clock phase */
#define SPI_CPOL    _BITUL(1)  /* clock polarity */

#define SPI_MODE_0  (0|0)             /* CPOL=0, CPHA=0 */
#define SPI_MODE_3  (SPI_CPOL|SPI_CPHA) /* CPOL=1, CPHA=1 */
```

الـ SSD1306 datasheet بيقول بيشتغل على CPOL=0, CPHA=0 فقط — يعني `SPI_MODE_0`. لما المطوّر غيّر لـ `SPI_MODE_3`، الـ clock edge اللي بيأخذ عليها الـ device البيانات اتغيرت، فالـ display بيسمع bit غلط في كل clock cycle.

```c
/* الكود الغلط */
uint8_t mode = SPI_MODE_3;  /* CPOL=1, CPHA=1 — غلط للـ SSD1306 */
ioctl(fd, SPI_IOC_WR_MODE, &mode);

/* الكود الصح */
uint8_t mode = SPI_MODE_0;  /* CPOL=0, CPHA=0 — صح */
ioctl(fd, SPI_IOC_WR_MODE, &mode);
```

#### الحل
```bash
# تحقق من الـ mode الحالي
cat /sys/bus/spi/devices/spi0.0/modalias

# في الـ device tree
# &spi0 {
#     display@0 {
#         compatible = "solomon,ssd1306";
#         spi-cpol;    // غلط — ازيله
#         spi-cpha;    // غلط — ازيله
#         spi-max-frequency = <10000000>;
#     };
# };
```

```c
/* الكود الصح في userspace */
uint8_t mode = SPI_MODE_0;  /* bit 0 و bit 1 كلاهم = 0 */
if (ioctl(fd, SPI_IOC_WR_MODE32, &mode) < 0)
    perror("SPI_IOC_WR_MODE32");
```

#### الدرس المستفاد
**`SPI_MODE_X_MASK`** = `(SPI_CPOL | SPI_CPHA)` = bits 0 و1 بس. الـ 4 modes مش بالضرورة متكافئة في الأداء — كل device ليه mode محدد في datasheet لازم تحترمه. SPI_MODE_3 مش "أسرع" من SPI_MODE_0.

---

### السيناريو 3: IoT Sensor Node بـ STM32MP1 — IMU بيرجع zeros دايمًا

#### العنوان
ICM-42688 IMU على STM32MP1 بيرجع zeros في كل القراءات.

#### السياق
IoT sensor node لقياس الاهتزاز في industrial equipment. الـ sensor هو ICM-42688-P بيتكلم SPI. الـ board bring-up على STM32MP1. المطوّر شاغل `SPI_CS_HIGH` في الـ mode.

#### المشكلة
الـ ICM-42688 chip select **active low** (الافتراضي). لما `SPI_CS_HIGH` يكون set، الـ CS line بتبقى high طول فترة الـ transaction — فالـ sensor مش بيتفعّل أصلًا، وكل القراءات zeros.

#### التحليل
من الفايل:
```c
#define SPI_CS_HIGH  _BITUL(2)  /* chipselect active high? */
```

الافتراضي في كل SPI devices إن CS **active low** — يعني لما CS=0 الجهاز بيشتغل. لو set `SPI_CS_HIGH` على device active-low، الـ controller بيـinvert المنطق ويرفع الـ CS أثناء الـ transaction بدل ما يخفضه.

```c
/* الكود الغلط */
uint32_t mode = SPI_MODE_0 | SPI_CS_HIGH;  /* CS_HIGH غلط للـ ICM-42688 */
ioctl(fd, SPI_IOC_WR_MODE32, &mode);

/* الكود الصح */
uint32_t mode = SPI_MODE_0;  /* CS active low — default */
ioctl(fd, SPI_IOC_WR_MODE32, &mode);
```

#### الحل
```bash
# فحص السيناريو بـ logic analyzer على CS line
# لو CS بيروح HIGH أثناء الـ transfer — المشكلة SPI_CS_HIGH

# فحص الـ DT
grep -r "spi-cs-high" /sys/firmware/devicetree/

# إزالة spi-cs-high من DT:
# &spi1 {
#     imu@0 {
#         compatible = "invensense,icm42688";
#         // spi-cs-high;  <-- ازيل السطر ده
#         spi-max-frequency = <24000000>;
#     };
# };
```

```c
/* التحقق من الـ mode بعد التصحيح */
uint32_t mode;
ioctl(fd, SPI_IOC_RD_MODE32, &mode);
/* تأكد إن bit 2 (SPI_CS_HIGH) = 0 */
printf("CS_HIGH bit: %d\n", (mode >> 2) & 1);  /* لازم يطبع 0 */
```

#### الدرس المستفاد
**`SPI_CS_HIGH` bit 2** بيعكس منطق الـ chip select. أي device جديد، ابدأ بالافتراض إن CS active-low وحدد `SPI_CS_HIGH` بس لو الـ datasheet صراحةً بيقول CS active-high. always قرأ الـ timing diagrams في الـ datasheet قبل ما تكتب حرف.

---

### السيناريو 4: Custom Board Bring-up بـ i.MX8 — SPI Flash بتقبل write بس مش بتحفظ

#### العنوان
SPI NOR Flash على i.MX8 بتاخد الـ write command بس البيانات مش بتتحفظ.

#### السياق
custom SBC بـ i.MX8MM للـ automotive ECU testing. الـ SPI Flash (MX25L25645G) متوصلة على FlexSPI. المطوّر بيستخدم raw SPI ioctl من userspace لتطوير الـ flashing tool.

#### المشكلة
الـ write operations بتـreturn success لكن verify بعدها بيلاقي البيانات لسه القديمة. السبب: `SPI_NO_CS` مضبوط، فالـ CS line مش بتتحرك أصلًا.

#### التحليل
من الفايل:
```c
#define SPI_NO_CS  _BITUL(6)  /* 1 dev/bus, no chipselect */
```

`SPI_NO_CS` مصمم للحالات اللي فيها جهاز واحد على الـ bus ومفيش CS line فعلية (CS مربوط بـ GND مباشرة). لو set على device بيحتاج CS، الـ controller مش بيـtoggle الـ CS line — فالـ flash مش بيلاقي الـ falling edge اللي بتـtrigger الـ SPI transaction جواه، فبيـignore كل الـ commands.

```c
/* فحص الـ mode المضبوط */
uint32_t mode;
ioctl(fd, SPI_IOC_RD_MODE32, &mode);

if (mode & SPI_NO_CS) {
    printf("SPI_NO_CS is SET — CS won't toggle!\n");
    mode &= ~SPI_NO_CS;  /* إزالة SPI_NO_CS */
    ioctl(fd, SPI_IOC_WR_MODE32, &mode);
}
```

#### الحل
```bash
# فحص مع logic analyzer:
# channel 0: CLK
# channel 1: MOSI
# channel 2: CS
# لو CS مش بيتحرك مع كل transaction — SPI_NO_CS مضبوط

# من الـ sysfs
cat /sys/bus/spi/devices/spi0.0/modalias
```

```c
/* الـ write sequence الصحيح للـ MX25L25645G */
uint32_t mode = SPI_MODE_0;  /* بدون SPI_NO_CS */
ioctl(fd, SPI_IOC_WR_MODE32, &mode);

/* Write Enable command أولًا */
uint8_t wren = 0x06;
struct spi_ioc_transfer t = {
    .tx_buf       = (unsigned long)&wren,
    .len          = 1,
    .speed_hz     = 80000000,
    .bits_per_word = 8,
    /* CS بيتـtoggle تلقائيًا لأن SPI_NO_CS = 0 */
};
ioctl(fd, SPI_IOC_MESSAGE(1), &t);
```

#### الدرس المستفاد
**`SPI_NO_CS`** مش "أنا مش محتاج chip select" — معناها "الـ CS line مربوطة fixed ومش محتاج الـ controller يـtoggle فيها". استخدمه فقط لو الـ hardware فعلًا كده. أي NOR/NAND flash محتاج CS toggling لكل command.

---

### السيناريو 5: AM62x Automotive ECU — SPI بـ 3-Wire Mode وdata corruption

#### العنوان
SPI EEPROM على AM62x يـreturn data ناقصة في 3-Wire mode.

#### السياق
automotive ECU بـ TI AM62x للـ body control module. الـ EEPROM (AT25256B) متوصل بـ 3-wire SPI (SI/SO على نفس الـ pin) لتوفير PCB space. المطوّر ضاف `SPI_3WIRE` لكن البيانات اللي بترجع ناقصة أو متشوهة.

#### المشكلة
في الـ 3-wire mode، الـ line بتبدّل اتجاهها بين TX وRX. لو الـ timing غلط أو الـ `SPI_3WIRE_HIZ` مش مضبوط في الوقت المناسب، الـ line بتبقى تـdrive نفسها أثناء الـ reception فبيحصل bus conflict.

#### التحليل
من الفايل:
```c
#define SPI_3WIRE      _BITUL(4)   /* SI/SO signals shared */
#define SPI_3WIRE_HIZ  _BITUL(15)  /* high impedance turnaround */
```

الـ `SPI_3WIRE` وحده مش كفاية. لما الـ turnaround من TX لـ RX، الـ controller لازم يحط الـ line في **high-impedance** قبل ما الـ slave يبدأ يـdrive. `SPI_3WIRE_HIZ` بيقول للـ controller: "خلي الـ line floating في لحظة التحويل".

```c
/* الكود الناقص */
uint32_t mode = SPI_MODE_0 | SPI_3WIRE;
/* بدون SPI_3WIRE_HIZ — bus conflict في الـ turnaround */
ioctl(fd, SPI_IOC_WR_MODE32, &mode);

/* الكود الصح */
uint32_t mode = SPI_MODE_0 | SPI_3WIRE | SPI_3WIRE_HIZ;
ioctl(fd, SPI_IOC_WR_MODE32, &mode);
```

```bash
# فحص الـ mode المضبوط
python3 -c "
import fcntl, struct
SPI_IOC_RD_MODE32 = 0x80047b05
fd = open('/dev/spidev0.0', 'rb')
buf = struct.pack('I', 0)
result = fcntl.ioctl(fd, SPI_IOC_RD_MODE32, buf)
mode = struct.unpack('I', result)[0]
SPI_3WIRE     = 1 << 4
SPI_3WIRE_HIZ = 1 << 15
print('3WIRE:', bool(mode & SPI_3WIRE))
print('3WIRE_HIZ:', bool(mode & SPI_3WIRE_HIZ))
fd.close()
"
```

#### الحل
```c
/* sequence صح للقراءة من AT25256B في 3-wire mode */
uint32_t mode = SPI_MODE_0 | SPI_3WIRE | SPI_3WIRE_HIZ;
ioctl(fd, SPI_IOC_WR_MODE32, &mode);

/* command + address في TX، ثم بيانات في RX */
uint8_t cmd[3] = {0x03, (addr >> 8) & 0xFF, addr & 0xFF};
uint8_t rx[len];

struct spi_ioc_transfer xfers[2] = {
    {
        /* TX phase: command + address */
        .tx_buf       = (unsigned long)cmd,
        .len          = 3,
        .speed_hz     = 5000000,
        .bits_per_word = 8,
        .cs_change    = 0,
    },
    {
        /* RX phase: data — line في HIZ أثناء التحويل */
        .rx_buf       = (unsigned long)rx,
        .len          = len,
        .speed_hz     = 5000000,
        .bits_per_word = 8,
    },
};
ioctl(fd, SPI_IOC_MESSAGE(2), xfers);
```

#### الدرس المستفاد
**`SPI_3WIRE`** وحده مش كافي في كل الـ hardware implementations. **`SPI_3WIRE_HIZ`** ضروري لتجنب bus conflict في لحظة الـ turnaround. دايمًا اقرأ الـ controller datasheet عشان تعرف هو بيدعم الـ HIZ turnaround ولا لأ — وتأكد إن الـ kernel driver للـ controller بيدعم الـ flag ده قبل ما تعتمد عليه في production.
## Phase 7: مصادر ومراجع

### الـ Official Kernel Documentation

الـ documentation الرسمية للـ SPI subsystem موجودة في عدة أماكن داخل kernel tree:

| المسار | المحتوى |
|--------|---------|
| `Documentation/spi/spi-summary.rst` | نظرة عامة شاملة على الـ SPI framework |
| `Documentation/spi/spidev.rst` | الـ userspace API وكيفية استخدام `/dev/spidevX.Y` |
| `include/uapi/linux/spi/spi.h` | الـ UAPI header — تعريفات الـ mode flags |
| `include/linux/spi/spi.h` | الـ kernel-internal header — `struct spi_device`, `struct spi_controller` |
| `drivers/spi/` | كل الـ controller drivers والـ SPI core |

الـ online version:
- [Serial Peripheral Interface (SPI) — The Linux Kernel documentation](https://docs.kernel.org/driver-api/spi.html)
- [SPI userspace API — The Linux Kernel documentation](https://docs.kernel.org/spi/spidev.html)
- [Overview of Linux kernel SPI support](https://docs.kernel.org/spi/spi-summary.html)

---

### الـ LWN.net Articles

دي أهم المقالات التاريخية اللي غطّت تطوير الـ SPI subsystem في Linux:

| المقال | الوصف |
|--------|-------|
| [SPI core (2005)](https://lwn.net/Articles/138037/) | أول patch للـ SPI core — David Brownell |
| [SPI core -- revisited (2005)](https://lwn.net/Articles/141186/) | مراجعة ثانية للـ SPI core بعد feedback المجتمع |
| [simple SPI framework (2006)](https://lwn.net/Articles/154425/) | إعادة تصميم الـ framework لتبسيط الـ API |
| [SPI core refresh (2006)](https://lwn.net/Articles/162268/) | تحديثات على الـ core بعد أول merge |
| [spi (2006)](https://lwn.net/Articles/146581/) | نقاشات مبكرة حول الـ SPI subsystem |
| [Add SPI over GPIO driver (2009)](https://lwn.net/Articles/290068/) | driver بيخلي GPIO pins تشتغل كـ SPI bus |
| [spi: Add slave mode support (2017)](https://lwn.net/Articles/723440/) | إضافة الـ SPI slave mode للـ kernel |
| [Introduction to SPI NAND framework (2017)](https://lwn.net/Articles/719733/) | framework مخصوص للـ SPI NAND flash chips |

---

### الـ Kernel Source على GitHub

- [include/uapi/linux/spi/spi.h — torvalds/linux](https://github.com/torvalds/linux/blob/master/include/uapi/linux/spi/spi.h) — الـ UAPI header الرئيسي
- [include/linux/spi/spi.h — torvalds/linux](https://github.com/torvalds/linux/blob/master/include/linux/spi/spi.h) — الـ kernel-internal header
- [include/uapi/linux/spi/spidev.h — torvalds/linux](https://github.com/torvalds/linux/blob/master/include/uapi/linux/spi/spidev.h) — الـ ioctl definitions للـ userspace

لرؤية تاريخ الـ UAPI header:
```bash
git log --oneline include/uapi/linux/spi/spi.h
git log --oneline include/linux/spi/spi.h
```

---

### الـ Mailing List

الـ SPI subsystem بيتناقش على:

- **القائمة البريدية**: `linux-spi@vger.kernel.org`
- **الـ Patchwork**: [Linux SPI core/device drivers — Patchwork](https://patchwork.kernel.org/project/spi-devel-general/list/)
- **الـ spi-devel-general**: [SourceForge Archive](https://sourceforge.net/p/spi-devel/mailman/spi-devel-general/)

المشرف الحالي على الـ SPI tree هو **Mark Brown** — كل الـ patches بتعدي عليه قبل ما تروح mainline.

---

### الـ KernelNewbies.org

صفحات الـ changelogs على kernelnewbies بتوثّق التغييرات في كل إصدار:

| الإصدار | الرابط |
|---------|--------|
| Linux 2.6.24 (أول SPI/MMC support) | [kernelnewbies.org/Linux_2_6_24](https://kernelnewbies.org/Linux_2_6_24) |
| Linux 3.15 | [kernelnewbies.org/Linux_3.15-DriversArch](https://kernelnewbies.org/Linux_3.15-DriversArch) |
| Linux 3.18 | [kernelnewbies.org/Linux_3.18-DriversArch](https://kernelnewbies.org/Linux_3.18-DriversArch) |
| Linux 4.18 | [kernelnewbies.org/Linux_4.18](https://kernelnewbies.org/Linux_4.18) |
| Linux 6.5 | [kernelnewbies.org/Linux_6.5](https://kernelnewbies.org/Linux_6.5) |
| Linux 6.6 | [kernelnewbies.org/Linux_6.6](https://kernelnewbies.org/Linux_6.6) |
| Linux 6.11 | [kernelnewbies.org/Linux_6.11](https://kernelnewbies.org/Linux_6.11) |
| Linux 6.12 | [kernelnewbies.org/Linux_6.12](https://kernelnewbies.org/Linux_6.12) |

---

### الـ eLinux.org

مصادر عملية للـ embedded Linux developers:

| الصفحة | المحتوى |
|--------|---------|
| [RPi SPI](https://elinux.org/RPi_SPI) | إعداد SPI على Raspberry Pi — الـ BCM2835 controller |
| [BeagleBoard/SPI](https://elinux.org/BeagleBoard/SPI) | الـ mcspi driver وإعداد الـ pin mux على BeagleBoard |
| [ECE497 SPI Project](https://elinux.org/ECE497_SPI_Project) | مثال عملي: ربط LED strand بالـ SPI عبر sysfs |
| [Tests:MSIOF-SPI-CS-GPIO](https://elinux.org/Tests:MSIOF-SPI-CS-GPIO) | اختبار MSIOF SPI مع GPIO chip selects على Renesas |

---

### الكتب الموصى بها

#### Linux Device Drivers, 3rd Edition (LDD3)
- **المؤلفون**: Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصل المهم**: Chapter 13 — USB Drivers (نفس نمط driver model المستخدم في SPI)
- **ملاحظة**: الكتاب قديم (kernel 2.6.10) بس الـ concepts الأساسية ثابتة
- **متاح مجاناً**: [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development, 3rd Edition — Robert Love
- **الفصول المهمة**:
  - Chapter 14: The Block I/O Layer (نفس مفاهيم الـ queuing)
  - Chapter 17: Devices and Modules (الـ device model الأساسي)
- **أهميته**: بيشرح الـ driver model اللي الـ SPI بيبني عليه

#### Embedded Linux Primer — Christopher Hallinan
- **الفصل**: Chapter 10 — MTD Subsystem (بيغطي SPI Flash)
- **أهميته**: بيربط الـ SPI hardware بالـ Linux driver stack في سياق embedded systems

#### Essential Linux Device Drivers — Sreekrishnan Venkateswaran
- **الفصل**: Chapter 8 — I2C and SPI subsystems
- **أهميته**: أحدث كتاب بيغطي SPI بشكل مباشر ومفصّل

---

### الـ Search Terms للبحث عن معلومات أكثر

للبحث في الـ kernel mailing list archives على `lore.kernel.org`:

```
SPI_MODE_USER_MASK split kernel user
SPI_MOSI_IDLE_LOW HIGH flag addition
SPI_CS_WORD toggle chip select
SPI TX_OCTAL RX_OCTAL octal SPI mode
SPI 3WIRE_HIZ high impedance
spi: add SPI_RX_CPHA_FLIP
Mark Brown SPI maintainer
spi-devel-general linux-spi vger
```

للبحث على `git.kernel.org`:
```bash
# تتبع تاريخ الـ UAPI flags
git log --all --oneline --follow -- include/uapi/linux/spi/spi.h

# البحث عن commit أضاف flag معين
git log --all --oneline -S "SPI_MOSI_IDLE_LOW"
git log --all --oneline -S "SPI_CS_WORD"
git log --all --oneline -S "SPI_MODE_USER_MASK"
```

للبحث على الـ web:
- `site:lore.kernel.org SPI_MODE_USER_MASK`
- `site:patchwork.kernel.org spi MOSI_IDLE`
- `linux kernel SPI octal mode OSPI`
- `linux spidev ioctl SPI_IOC_WR_MODE32`
## Phase 8: Writing simple module

### الفكرة

**الـ** `include/uapi/linux/spi/spi.h` بيعرّف الـ mode flags بتاعة SPI زي `SPI_CPHA`, `SPI_CPOL`, `SPI_CS_HIGH`, وغيرهم. كل ما device بيتضبط، بيتعمل call لـ `spi_setup()` اللي exported في kernel. هنعمل **kprobe** على `spi_setup` عشان نطبع معلومات الـ device — الـ mode flags، الـ speed، الـ bits_per_word — وده مفيد جداً في debugging مشاكل SPI على embedded systems.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * spi_setup_probe.c
 * kprobe on spi_setup() — prints SPI device configuration on each setup call
 */

/* --- Includes --- */
#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit          */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe()          */
#include <linux/spi/spi.h>      /* struct spi_device, SPI_* mode flags       */
#include <linux/printk.h>       /* pr_info()                                 */
#include <uapi/linux/spi/spi.h> /* SPI_CPHA, SPI_CPOL, SPI_CS_HIGH, ...     */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("SPI Watcher <kernel@example.com>");
MODULE_DESCRIPTION("kprobe on spi_setup() to log SPI device configuration");

/* ------------------------------------------------------------------ */
/* helper: بيحوّل الـ mode بتاع SPI لـ string مقروء                   */
/* ------------------------------------------------------------------ */
static void decode_spi_mode(u32 mode, char *buf, size_t len)
{
    snprintf(buf, len,
             "MODE%u%s%s%s%s%s%s%s",
             (mode & SPI_MODE_X_MASK),                  /* CPOL|CPHA number */
             (mode & SPI_CS_HIGH)    ? "|CS_HIGH"  : "",
             (mode & SPI_LSB_FIRST)  ? "|LSB_FIRST": "",
             (mode & SPI_3WIRE)      ? "|3WIRE"    : "",
             (mode & SPI_LOOP)       ? "|LOOP"     : "",
             (mode & SPI_TX_QUAD)    ? "|TX_QUAD"  : "",
             (mode & SPI_RX_QUAD)    ? "|RX_QUAD"  : "",
             (mode & SPI_NO_CS)      ? "|NO_CS"    : "");
}

/* ------------------------------------------------------------------ */
/* الـ pre-handler: بيتنفذ قبل ما spi_setup تشتغل                      */
/* ------------------------------------------------------------------ */
static int spi_setup_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * الـ argument الأول لـ spi_setup هو (struct spi_device *spi).
     * على x86_64: RDI = arg0, على ARM64: x0 = arg0.
     * الـ kprobe infrastructure بيوفر pt_regs بحيث نقدر نجيب الـ arg.
     */
    struct spi_device *spi = (struct spi_device *)regs_get_register(regs, 0);
    char mode_str[128];

    if (!spi)
        return 0;

    decode_spi_mode(spi->mode, mode_str, sizeof(mode_str));

    pr_info("spi_setup: dev=%s modalias=%s mode=0x%08x (%s) speed=%u Hz bits=%u\n",
            dev_name(&spi->dev),   /* اسم الـ device في sysfs مثلاً spi0.0  */
            spi->modalias,         /* اسم الـ driver المطلوب مثلاً "spidev" */
            spi->mode,             /* raw mode bitmask                        */
            mode_str,              /* decoded human-readable mode             */
            spi->max_speed_hz,     /* أقصى سرعة clock مدعومة بالـ device     */
            spi->bits_per_word);   /* عدد bits في الـ word الواحد             */

    return 0; /* return 0 = kernel يكمل تنفيذ spi_setup طبيعي             */
}

/* ------------------------------------------------------------------ */
/* تعريف الـ kprobe struct                                              */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "spi_setup",   /* الـ function اللي هنحط عليها الـ hook  */
    .pre_handler = spi_setup_pre, /* الـ callback اللي بيتنفذ قبلها         */
};

/* ------------------------------------------------------------------ */
/* module_init: بيسجل الـ kprobe وقت تحميل الـ module                 */
/* ------------------------------------------------------------------ */
static int __init spi_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("spi_setup_probe: failed to register kprobe: %d\n", ret);
        return ret;
    }

    pr_info("spi_setup_probe: kprobe planted on spi_setup @ %p\n", kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit: بيلغي تسجيل الـ kprobe عشان ميحصلش crash بعد الـ rmmod */
/* ------------------------------------------------------------------ */
static void __exit spi_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("spi_setup_probe: kprobe removed from spi_setup\n");
}

module_init(spi_setup_init);
module_exit(spi_probe_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `<linux/kprobes.h>` | بيوفر `struct kprobe`, `register_kprobe()`, `regs_get_register()` |
| `<linux/spi/spi.h>` | بيعرّف `struct spi_device` وفيه fields زي `mode`, `max_speed_hz` |
| `<uapi/linux/spi/spi.h>` | بيعرّف الـ mode flags: `SPI_CPHA`, `SPI_CS_HIGH`, `SPI_TX_QUAD` ...إلخ |
| `<linux/printk.h>` | بيوفر `pr_info()`, `pr_err()` للـ logging في kernel |

**الـ** UAPI header مهم لأن الـ mode flags المعرّفة فيه هي اللي بيستخدمها الـ userspace والـ kernel معاً.

---

#### الـ `decode_spi_mode()` helper

الدالة دي بتحوّل الـ bitmask بتاع الـ mode لـ string مقروء باستخدام الـ flags المعرّفة في `uapi/linux/spi/spi.h`. مثلاً لو الـ mode = `SPI_CPOL | SPI_CS_HIGH` هيطلع `"MODE2|CS_HIGH"` بدل `0x00000006` اللي مش واضح.

---

#### الـ `spi_setup_pre()` — الـ pre-handler

الـ handler ده بيتنفذ **قبل** ما `spi_setup()` تشتغل فعلاً. الـ `pt_regs` بيحتوي على قيم الـ registers وقت الـ call، ومنهم الـ argument الأول (الـ `spi_device *`). بنطبع:

- **`dev_name`**: اسم الـ device في sysfs زي `spi0.0` (الـ controller رقم 0، الـ chipselect رقم 0).
- **`modalias`**: اسم الـ driver المطلوب مثلاً `"ads7846"` أو `"spidev"`.
- **`mode`**: الـ bitmask الخام + الـ decoded string بتاعه.
- **`max_speed_hz`**: السرعة القصوى اللي الـ device يدعمها — مهم لـ debugging مشاكل الـ timing.
- **`bits_per_word`**: عدد الـ bits في كل word، الافتراضي 8.

---

#### الـ `kprobe` struct

```c
static struct kprobe kp = {
    .symbol_name = "spi_setup",
    .pre_handler = spi_setup_pre,
};
```

الـ kernel بيحل اسم `"spi_setup"` لـ address تلقائياً وقت `register_kprobe()`. مش محتاج نحدد الـ address يدوياً.

---

#### الـ `module_init` و `module_exit`

- **`register_kprobe`** بيحط breakpoint على أول instruction في `spi_setup` — لو فشل (مثلاً لو الـ function مش exported أو الـ kprobes مش enabled في kernel config) بيرجع error ونرجع فوراً.
- **`unregister_kprobe`** في الـ exit **ضروري جداً** — لو عملنا `rmmod` من غير ما نشيل الـ kprobe، أي call على `spi_setup` بعدها هيجمب على code اتمسح من الـ memory وده kernel panic فوري.

---

### كيفية التجربة

```bash
# بناء الـ module (محتاج Makefile بسيط)
make -C /lib/modules/$(uname -r)/build M=$PWD modules

# تحميل الـ module
sudo insmod spi_setup_probe.ko

# شوف الـ output
sudo dmesg | grep spi_setup

# مثال على output لو في SPI device اتضبط:
# [ 42.123456] spi_setup_probe: kprobe planted on spi_setup @ ffffffffc0123456
# [ 43.001234] spi_setup: dev=spi0.0 modalias=spidev mode=0x00000000 (MODE0) speed=1000000 Hz bits=8
# [ 43.005678] spi_setup: dev=spi1.0 modalias=ads7846 mode=0x00000001 (MODE1) speed=2000000 Hz bits=8

# إزالة الـ module
sudo rmmod spi_setup_probe
```

---

### ليه `spi_setup` تحديداً؟

لأنها **نقطة مركزية** — كل SPI device في النظام لازم يمر بيها مرة واحدة على الأقل عشان يتضبط. ده بيديك رؤية كاملة لكل الـ SPI devices الموجودة وإعداداتها بدون ما تلمس أي driver أو device. مفيد جداً في:

- التحقق إن الـ mode flags الصح بتتبعت (مثلاً `SPI_MODE_3` بدل `SPI_MODE_0`).
- مقارنة الـ speed المطلوبة مع الـ speed الفعلية.
- معرفة أي drivers بتعمل `spi_setup` أكتر من مرة.
