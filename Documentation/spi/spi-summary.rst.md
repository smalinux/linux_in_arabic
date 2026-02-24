## Phase 1: الصورة الكبيرة ببساطة

### ما هو SPI؟

تخيل إنك عندك **مايكروكنترولر** (زي الـ CPU في embedded board) وعايزه يتكلم مع chip تانية — سواء كانت sensor بيقيس درجة الحرارة، أو flash memory بتخزن بيانات، أو شاشة صغيرة أو أي peripheral تاني.

**الـ SPI (Serial Peripheral Interface)** هو "لغة تواصل" بسيطة جداً بين الـ CPU وهذه الـ chips. اخترعته شركة Motorola في الثمانينيات ومفيش هيئة رسمية بتحكمه — كل vendor بيعمله كيفما شاء، لكن الأساس واحد.

الـ SPI بيستخدم **4 أسلاك فقط**:

```
   CPU (Host/Master)              Peripheral (Target/Slave)
   ┌──────────────┐               ┌──────────────┐
   │         SCLK ├──────────────►│ SCLK         │  clock
   │         MOSI ├──────────────►│ MOSI         │  data out (host→target)
   │         MISO │◄──────────────┤ MISO         │  data in  (target→host)
   │          nCS ├──────────────►│ nCS          │  chip select (active low)
   └──────────────┘               └──────────────┘
```

- **SCLK**: ساعة — الـ host بيولّدها، وبيها بيحصل التزامن
- **MOSI** (Master Out Slave In): الـ host بيبعت data للـ target
- **MISO** (Master In Slave Out): الـ target بيبعت data للـ host
- **nCS** (Chip Select): الـ host بيختار مين بيتكلم معه (active low يعني low = محدد)

الجميل إن نفس الـ SCLK/MOSI/MISO ممكن تتوصل بـ **أكتر من chip** في نفس الوقت، وبس الـ nCS بيختلف لكل chip.

```
   CPU
   ├── SCLK ──────┬──────────┬────────
   ├── MOSI ──────┼──────────┼────────
   ├── MISO ──────┼──────────┼────────
   ├── nCS0 ──── [Flash]     │
   └── nCS1 ──────────── [Sensor]
```

---

### ليه SPI مفيد؟

| الميزة | التفاصيل |
|--------|----------|
| **بساطة الـ hardware** | 4 أسلاك بس، مفيش عنوان، مفيش arbitration |
| **سرعة عالية** | عادةً من 1 MHz لـ 100+ MHz، أسرع بكتير من I2C |
| **Full duplex** | الإرسال والاستقبال في نفس اللحظة |
| **Low cost** | مش محتاج resistors pull-up زي I2C |
| **Flexible** | يدعم word sizes مختلفة: 8, 12, 16, 20 bit |

**أين يُستخدم؟**
- **Flash memory**: BIOS chips في الـ PC، DataFlash، NOR Flash
- **Sensors**: ADC، temperature sensors، accelerometers
- **Displays**: LCD controllers
- **Networking**: بعض Ethernet adapters
- **Audio**: codecs
- **Storage**: MMC/SD cards (بتدعم SPI mode كـ fallback)

---

### القصة الكاملة: كيف يعمل الـ SPI في Linux؟

تخيل عندك **embedded board** (زي Raspberry Pi أو ARM SoC) فيها:
- **SPI controller** مدمج في الـ CPU —ده chip بيعرف يوّلد الـ SCLK ويتحكم في الـ MOSI/MISO
- **Flash chip** موصلة على الـ SPI bus
- **Temperature sensor** موصلة على نفس الـ bus

المشكلة: إزاي تكتب **driver** يشتغل مع أي SPI device على أي SPI controller؟

Linux حلّها بـ **طبقتين**:

```
┌────────────────────────────────────────────────┐
│          Protocol Driver (SPI target driver)    │
│  مثلاً: driver لـ Flash chip أو Temperature    │
│  بيعرف "اللغة" الخاصة بالـ chip (الـ commands) │
├────────────────────────────────────────────────┤
│              SPI Core (spi.c)                   │
│        الوسيط — بيوصّل الطبقتين ببعض           │
├────────────────────────────────────────────────┤
│         Controller Driver (SPI host driver)     │
│  مثلاً: driver لـ BCM2835 SPI أو i.MX SPI     │
│  بيعرف يتعامل مع الـ hardware registers        │
└────────────────────────────────────────────────┘
```

**الـ spi-summary.rst** ده وثيقة المطور — بتشرح الـ big picture لكل اللي فوق.

---

### الـ 4 Clock Modes

الـ SPI مفيش spec رسمية، فكل vendor بيختار طريقة الـ clocking. اتفقوا على تركيبة من بتين:

| Mode | CPOL | CPHA | معناه |
|------|------|------|-------|
| 0 | 0 | 0 | clock يبدأ low، sample على rising edge |
| 1 | 0 | 1 | clock يبدأ low، sample على falling edge |
| 2 | 1 | 0 | clock يبدأ high، sample على falling edge |
| 3 | 1 | 1 | clock يبدأ high، sample على rising edge |

- **CPOL**: الـ clock في حالة idle بيكون high أو low؟
- **CPHA**: بنـ sample الـ data على أي edge؟

الأكثر شيوعاً في الـ real world هما **Mode 0** و **Mode 3**.

---

### مكونات الـ SPI Subsystem في Linux

#### الـ Core Files

| الملف | الدور |
|-------|-------|
| `drivers/spi/spi.c` | الـ SPI core — الـ bus type، الـ registration، الـ message queue |
| `drivers/spi/spidev.c` | Userspace interface لـ SPI devices عبر `/dev/spidevX.Y` |
| `drivers/spi/internals.h` | Internal structures يستخدمها الـ core فقط |

#### الـ Headers

| الملف | الدور |
|-------|-------|
| `include/linux/spi/spi.h` | **الأهم** — تعريف `struct spi_device`، `struct spi_controller`، `struct spi_message`، `struct spi_transfer`، وكل الـ API |
| `include/uapi/linux/spi/spi.h` | الـ mode bits المتاحة للـ userspace (SPI_MODE_0, SPI_CPHA, إلخ) |
| `include/linux/spi/spi-mem.h` | واجهة لـ SPI memory (QSPI, OSPI) |

#### الـ Hardware Drivers (أمثلة)

| الملف | الـ Hardware |
|-------|-------------|
| `drivers/spi/spi-bcm2835.c` | Raspberry Pi (BCM2835) |
| `drivers/spi/spi-imx.c` | NXP i.MX SoCs |
| `drivers/spi/spi-pl022.c` | ARM PrimeCell SSP |
| `drivers/spi/spi-bitbang.c` | GPIO bitbang (أي GPIO pins) |
| `drivers/spi/spi-omap2-mcspi.c` | Texas Instruments OMAP |

#### الـ Documentation

| الملف | المحتوى |
|-------|---------|
| `Documentation/spi/spi-summary.rst` | **هذا الملف** — Big picture overview |
| `Documentation/spi/spidev.rst` | الـ userspace API |
| `Documentation/spi/multiple-data-lanes.rst` | Multi-lane SPI (QSPI, OSPI) |

---

### الـ Data Flow: إزاي بتتبعت رسالة SPI؟

**سيناريو**: driver بيقرأ temperature من SPI sensor

```
Protocol Driver (e.g., LM70 temperature sensor)
  │
  │  1. بيبني spi_message يحتوي على spi_transfer(s)
  │     (buffer فيه الـ command + buffer لاستقبال الـ reading)
  │
  ▼
spi_async() / spi_sync()   ← SPI Core API
  │
  │  2. الـ core بيضيف الـ message لـ queue الـ controller
  │
  ▼
Controller Driver
  │
  │  3. بيبعت الـ bytes على الـ hardware (DMA or PIO)
  │  4. بيتلقى الـ response في نفس الوقت (full duplex)
  │
  ▼
Callback / spi_sync returns
  │
  │  5. الـ protocol driver بيقرأ النتيجة من الـ buffer
  ▼
Upper Layers (input subsystem, hwmon, IIO, MTD, etc.)
```

---

### الـ Key Data Structures

```c
/* يمثّل chip واحدة موصلة على الـ SPI bus */
struct spi_device {
    struct device       dev;
    struct spi_controller *controller;
    u32                 max_speed_hz;  /* أقصى clock speed للـ chip */
    u8                  bits_per_word; /* حجم الـ word عادةً 8 */
    u32                 mode;          /* SPI_MODE_0..3 + flags */
    int                 irq;
    /* ... */
};

/* يمثّل الـ SPI host controller (الـ hardware) */
struct spi_controller {
    struct device   dev;
    int             bus_num;   /* رقم الـ bus في /sys */
    /* الـ methods اللي الـ controller driver بيـ implement */
    int  (*setup)(struct spi_device *spi);
    int  (*transfer_one_message)(struct spi_controller *, struct spi_message *);
    /* ... */
};

/* packet من transfers — الوحدة الأساسية للـ I/O */
struct spi_message {
    struct list_head    transfers; /* قائمة من spi_transfer */
    struct spi_device  *spi;
    void (*complete)(void *context); /* callback لما تخلص */
    /* ... */
};

/* transfer واحد: send/receive buffer */
struct spi_transfer {
    const void  *tx_buf; /* بيانات بنبعتها (NULL = لأ) */
    void        *rx_buf; /* buffer بنستقبل فيه (NULL = لأ) */
    unsigned    len;     /* عدد الـ bytes */
    u32         speed_hz;
    struct spi_delay delay; /* تأخير بعد الـ transfer */
    /* ... */
};
```

---

### ملفات مهمة يجب معرفتها

- **`include/linux/spi/spi.h`** — ابدأ منه، فيه كل الـ API والـ structs
- **`drivers/spi/spi.c`** — الـ core implementation
- **`drivers/spi/spi-bitbang.c`** — أبسط مثال لـ controller driver (GPIO bitbang)
- **`drivers/spi/spidev.c`** — لو عايز تفهم الـ userspace interface
- **`Documentation/spi/spidev.rst`** — وثيقة الـ `/dev/spidevX.Y` interface
- **`include/uapi/linux/spi/spi.h`** — الـ SPI mode flags الـ user-visible
## Phase 2: شرح الـ SPI Framework

### المشكلة اللي الـ SPI Framework بيحلها

تخيل إن عندك SoC زي i.MX6 أو STM32MP1 فيه 3 كونترولرز SPI. على كل كونترولر ممكن يكون متوصل flash memory، touchscreen sensor، ADC، وغيرهم. كل device من دول بيتكلم بـprotocol مختلف فوق نفس الـwires.

**المشكلة الحقيقية ثلاثية**:

1. **Hardware Diversity**: كل SoC ليه SPI controller مختلف — registers مختلفة، DMA setup مختلف، clock gating مختلف. لو كل device driver اشتغل مباشرة مع الـhardware، هيبقى كود مكرر بشكل مجنون وغير قابل للـporting.

2. **Bus Arbitration**: أكتر من device driver ممكن يحتاج يبعت data في نفس الوقت على نفس الـbus. محتاج حاجة تنظم الدور وتضمن إن الـchipselect يفضل active طول الـmessage.

3. **Driver Layering**: الـflash driver مش المفروض يعرف حاجة عن الـDMA engine أو الـclock source للـSPI controller. ده شغل حد تاني.

**الـkernel حل ده بـSPI framework**: طبقة وسيطة مسؤولة عن الـbus abstraction، الـqueue management، الـdevice/driver binding، والـDMA mapping.

---

### الـSolution: الـTwo-Driver Model

الـkernel قسّم الموضوع لنوعين من الـdrivers:

| النوع | الاسم | المسؤولية |
|-------|-------|------------|
| **Controller Driver** | `spi_controller` | بيتكلم مع الـhardware registers، DMA، IRQ |
| **Protocol Driver** | `spi_driver` | بيتكلم مع الـdevice (flash, sensor, etc.) عبر messages |

الـframework نفسه هو الـglue اللي بيربطهم ببعض ويدير الـqueue بينهم.

---

### الـBig Picture: مكان الـFramework في الـKernel

```
┌─────────────────────────────────────────────────────────────────────┐
│                        User Space                                    │
│              (spidev, userspace tools via /dev/spidevX.Y)           │
└───────────────────────────┬─────────────────────────────────────────┘
                            │ syscalls
┌───────────────────────────▼─────────────────────────────────────────┐
│                      Kernel Space                                    │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────┐ │
│  │  MTD/Flash   │  │  Input Layer │  │  Network     │  │  ALSA   │ │
│  │  Subsystem   │  │  (touchscr.) │  │  Stack       │  │  Audio  │ │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └────┬────┘ │
│         │                 │                  │               │      │
│  ┌──────▼─────────────────▼──────────────────▼───────────────▼────┐ │
│  │              SPI Protocol Drivers  (spi_driver)                │ │
│  │   spi-nor.c    ads7846.c    enc28j60.c    wm8731.c  ...       │ │
│  └──────────────────────────┬───────────────────────────────────┘ │
│                             │  spi_async() / spi_sync()            │
│  ┌──────────────────────────▼───────────────────────────────────┐  │
│  │                   SPI Core Framework                         │  │
│  │   drivers/spi/spi.c                                          │  │
│  │                                                              │  │
│  │   • Bus registration (spi_bus_type)                         │  │
│  │   • Device/Driver matching (modalias)                       │  │
│  │   • Message Queue (kthread pump)                            │  │
│  │   • DMA mapping helpers                                     │  │
│  │   • sysfs entries (/sys/bus/spi/...)                        │  │
│  └──────────────────────────┬───────────────────────────────────┘  │
│                             │  transfer_one() / transfer_one_msg()  │
│  ┌──────────────────────────▼───────────────────────────────────┐  │
│  │              SPI Controller Drivers  (spi_controller)        │  │
│  │   spi-imx.c   spi-pl022.c   spi-bcm2835.c   spi-bitbang.c  │  │
│  └──────────────────────────┬───────────────────────────────────┘  │
│                             │  MMIO registers, DMA, IRQ             │
│  ┌──────────────────────────▼───────────────────────────────────┐  │
│  │                      Hardware                                │  │
│  │   SPI Controller IP (e.g., i.MX ECSPI, PL022 SSP)          │  │
│  │   Physical wires: SCLK, MOSI, MISO, nCS0..nCSn              │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

### الـReal-World Analogy: مطعم (والتفاصيل الكاملة)

تخيل مطعم فيه:

```
الـ "Customer"    ←→   SPI Protocol Driver (spi-nor.c, ads7846.c)
الـ "Waiter"      ←→   SPI Core Framework (spi.c)
الـ "Kitchen"     ←→   SPI Controller Driver (spi-imx.c)
الـ "Chef Tools"  ←→   Hardware Registers / DMA Engine
الـ "Order"       ←→   spi_message
الـ "Dish Items"  ←→   spi_transfer (كل item في الأوردر = transfer)
الـ "Table #"     ←→   chipselect number
```

**التفصيل العميق للأنالوجي**:

- الـ**Customer** (protocol driver) بيطلب أكل بدون ما يعرف أي chef هيعمله أو أي stove هيستخدم. هو بس بيكتب الـorder.
- الـ**Waiter** (core) بياخد الـorder، يحطها في queue (لو في orders تانية قبلها)، يوصّلها للـkitchen بالترتيب، ويرجع بالنتيجة.
- الـ**Kitchen** (controller driver) هي اللي بتعمل الشغل الفعلي: تشغيل الـstove (DMA)، استخدام الأدوات الصح (registers). هي مش بتعرف ولا تهتم إيه الـcustomer عايز، هي بس بتنفذ.
- الـ**Table Number** (chipselect) بيحدد أنهي customer الأوردر بتاعته — المطعم فيه أكتر من طاولة (أكتر من device على نفس الـbus).
- الـ**Order** (`spi_message`) هي atomic: لازم تتنفذ كلها كاملة قبل ما table تانية تاخد دور. مش ممكن تعمل نص order لعميل ونص لتاني.
- الـ**Dish Items** (`spi_transfer`) هي الخطوات داخل الـorder: مثلاً "ابعت command byte" ثم "استقبل response bytes" — ممكن يكون فيه أكتر من step في نفس الـmessage.

---

### الـCore Abstractions: الـStructs الأساسية

#### 1. الـ`spi_controller` — يمثل الـhardware controller

```c
struct spi_controller {
    struct device   dev;        /* embedded device — الـkernel device model */
    s16             bus_num;    /* رقم الـbus: spi0, spi1, ... */
    u16             num_chipselect; /* عدد الـCS lines */
    u32             mode_bits;  /* الـmodes اللي الـhardware بيدعمها */
    u32             min_speed_hz;
    u32             max_speed_hz;

    /* === الـHooks اللي الـcontroller driver بيملأها === */
    int  (*setup)(struct spi_device *spi);   /* configure clock/mode لـdevice معين */
    void (*set_cs)(struct spi_device *spi, bool enable); /* toggle chipselect */
    int  (*transfer_one)(struct spi_controller *ctlr,
                         struct spi_device *spi,
                         struct spi_transfer *transfer); /* نفّذ transfer واحد */
    int  (*transfer_one_message)(struct spi_controller *ctlr,
                                 struct spi_message *mesg); /* نفّذ message كاملة */
    int  (*prepare_transfer_hardware)(struct spi_controller *ctlr);
    int  (*unprepare_transfer_hardware)(struct spi_controller *ctlr);

    /* === الـQueue Management (managed by core) === */
    bool            queued;         /* هل بيستخدم الـgeneric queue؟ */
    struct kthread_worker *kworker; /* الـpump thread */
    struct list_head queue;         /* قائمة الـmessages الـpending */
    struct spi_message *cur_msg;    /* الـmessage اللي بتتنفذ دلوقتي */

    /* === الـDMA Support === */
    bool (*can_dma)(struct spi_controller *ctlr,
                    struct spi_device *spi,
                    struct spi_transfer *xfer);
    struct dma_chan  *dma_tx;
    struct dma_chan  *dma_rx;
};
```

**ملاحظة مهمة**: `transfer_one` و `transfer_one_message` متضادين — لو الـdriver ملأ `transfer_one`، الـcore هو اللي بيجمع الـtransfers في message ويديرها. لو ملأ `transfer_one_message`، الـdriver بياخد control كامل على الـmessage.

---

#### 2. الـ`spi_device` — يمثل device واحد على الـbus

```c
struct spi_device {
    struct device       dev;            /* الـparent = spi_controller */
    struct spi_controller *controller;
    u32                 max_speed_hz;   /* أقصى clock speed للـdevice ده */
    u8                  bits_per_word;  /* عادةً 8، ممكن 12 أو 16 */
    u32                 mode;           /* SPI_MODE_0..3 + flags */
    int                 irq;
    char                modalias[SPI_NAME_SIZE]; /* اسم الـdriver اللازم */

    /* CS timing delays */
    struct spi_delay    cs_setup;
    struct spi_delay    cs_hold;
    struct spi_delay    cs_inactive;

    u8                  chip_select[SPI_DEVICE_CS_CNT_MAX];
};
```

الـ`spi_device` هو الـbridge بين الـprotocol driver والـcontroller. الـprotocol driver بياخد pointer ليه في الـ`probe()` وبيستخدمه في كل I/O.

---

#### 3. الـ`spi_transfer` — الـatomic read/write operation

```c
struct spi_transfer {
    const void  *tx_buf;    /* بيانات هتتبعت (NULL = إرسل صفر) */
    void        *rx_buf;    /* buffer لاستقبال (NULL = تجاهل المستقبَل) */
    unsigned    len;        /* حجم الـbuffer بالـbytes */

    u32         speed_hz;   /* override الـspeed للـtransfer ده بس */
    u8          bits_per_word; /* override الـword size */

    unsigned    cs_change:1;  /* هل ترفع الـCS بعد الـtransfer ده؟ */
    unsigned    tx_nbits:4;   /* 1=single, 2=dual, 4=quad lanes */
    unsigned    rx_nbits:4;

    struct spi_delay    delay;          /* delay بعد الـtransfer */
    struct spi_delay    word_delay;     /* delay بين الـwords */

    /* DMA — managed by core, not by protocol driver */
    dma_addr_t  tx_dma;
    dma_addr_t  rx_dma;
    struct sg_table tx_sg;
    struct sg_table rx_sg;

    struct list_head transfer_list; /* ربطها في الـspi_message */
};
```

---

#### 4. الـ`spi_message` — تسلسل atomic من الـtransfers

```c
struct spi_message {
    struct list_head    transfers;  /* linked list من spi_transfer */
    struct spi_device   *spi;       /* الـdevice اللي الـmessage ليه */

    int                 status;     /* النتيجة: 0 = success */
    void (*complete)(void *context); /* callback لما تخلص */
    void                *context;   /* argument للـcallback */

    unsigned            frame_length;   /* إجمالي الـbytes */
    unsigned            actual_length;  /* اللي اتبعت فعلاً */

    struct list_head    queue;  /* للـcontroller driver internal use */
};
```

---

### العلاقة بين الـStructs

```
spi_controller
│
│  (bus owns multiple devices)
│
├──► spi_device (nCS0) ──► spi_driver (e.g., spi-nor)
│        │
│        └── modalias = "m25p80"
│
├──► spi_device (nCS1) ──► spi_driver (e.g., ads7846)
│        │
│        └── modalias = "ads7846"
│
└──► spi_device (nCS2) ──► spi_driver (e.g., enc28j60)

                    ▼  (when driver sends I/O)

spi_message  ─────────────────────────────────┐
  │                                           │
  ├── spi_transfer #1                         │
  │     tx_buf = [0x03, 0x00, 0x00, 0x00]    │ → spi_device.controller
  │     rx_buf = NULL                         │       │
  │                                           │       ▼
  └── spi_transfer #2                         │  spi_controller.queue
        tx_buf = NULL                         │       │
        rx_buf = [256-byte buffer]            │       ▼
                                              │  kthread pump
                                              │       │
                                              │       ▼
                                              └── transfer_one() / transfer_one_message()
                                                       │
                                                       ▼
                                                  Hardware Registers / DMA
```

---

### الـMessage Queue: كيف بيشتغل الـPump

ده مفهوم محوري — الـSPI core بيشغّل **kthread** اسمه الـ"message pump". الـprotocol driver مش بيتعامل مع الـhardware مباشرة، بيحط الـmessage في الـqueue وبيعدّي.

```
Protocol Driver                SPI Core (kthread)          Controller Driver
      │                               │                           │
      │──spi_async(msg)──────────────►│                           │
      │                        add to queue                       │
      │                               │                           │
      │                        (wake pump)                        │
      │                               │                           │
      │                               │──prepare_transfer_hw()───►│
      │                               │◄──────────────────────────│
      │                               │                           │
      │                               │──transfer_one_message()──►│
      │                               │        (or)               │
      │                               │──transfer_one()──────────►│
      │                               │                           │
      │                               │   [hardware sends data]   │
      │                               │                           │
      │                               │◄──spi_finalize_current_message()
      │                               │                           │
      │◄──complete(context)───────────│                           │
      │                               │                           │
      │                        next message?                      │
      │                               │                           │
      │                        [if queue empty]                   │
      │                               │──unprepare_transfer_hw()─►│
```

**ملاحظة مهمة**: الـ`spi_sync()` هي مجرد `spi_async()` + `wait_for_completion()` — مش فيه مسار مختلف في الـhardware، بس بتبلوك الـcalling thread.

---

### الـDriver Registration Flow

#### الـController Driver:

```c
/* 1. Allocate controller struct (+ private data appended after it) */
ctlr = spi_alloc_host(&pdev->dev, sizeof(struct my_spi_priv));

/* 2. Fill in hardware capabilities */
ctlr->bus_num        = pdev->id;  /* e.g., 2 for SPI2 */
ctlr->num_chipselect = 4;
ctlr->mode_bits      = SPI_CPOL | SPI_CPHA | SPI_CS_HIGH;
ctlr->max_speed_hz   = 50000000; /* 50 MHz */

/* 3. Wire up the hooks */
ctlr->setup          = my_spi_setup;
ctlr->transfer_one   = my_spi_transfer_one;  /* OR transfer_one_message */
ctlr->set_cs         = my_spi_set_cs;
ctlr->prepare_transfer_hardware   = my_spi_prepare_hw;
ctlr->unprepare_transfer_hardware = my_spi_unprepare_hw;

/* 4. Publish to the framework */
spi_register_controller(ctlr);
/* هنا الـcore بيعمل:
   - ينشئ /sys/class/spi_master/spiB
   - يبدأ الـkthread pump
   - يعمل bind للـdevices اللي كانت مُعلنة في spi_board_info
*/
```

#### الـProtocol Driver:

```c
static struct spi_driver ads7846_driver = {
    .driver = {
        .name = "ads7846",          /* بيتطابق مع modalias في spi_device */
        .of_match_table = ads7846_dt_ids,
    },
    .probe  = ads7846_probe,
    .remove = ads7846_remove,
};
module_spi_driver(ads7846_driver);
/* module_spi_driver هي macro بتعمل module_init/exit بشكل تلقائي */
```

---

### الـmode Flags: CPOL و CPHA بالتفصيل

ده من أكتر الحاجات اللي بتلخبط الناس. الجدول ده بيوضح العلاقة:

| Mode | CPOL | CPHA | الـClock يبدأ | الـSample يحصل عند |
|------|------|------|---------------|---------------------|
| 0    | 0    | 0    | Low           | Rising edge (leading) |
| 1    | 0    | 1    | Low           | Falling edge (trailing) |
| 2    | 1    | 0    | High          | Falling edge (leading) |
| 3    | 1    | 1    | High          | Rising edge (trailing) |

```
Mode 0 (CPOL=0, CPHA=0):

nCS  ‾‾‾\___________________________/‾‾‾
SCLK ________/‾\_/‾\_/‾\_/‾\________
             ↑  ↑  ↑  ↑  ↑
             sample points (rising edge)
MOSI ----[D7][D6][D5][D4][D3]--------

Mode 3 (CPOL=1, CPHA=1): نفس السلوك من الـdevice side
```

**لماذا كتير من الـdevices بتدعم Mode 0 و Mode 3 في نفس الوقت؟** لأن في الاتنين الـsampling بيحصل على الـrising edge — الفرق بس هو هل الـclock هادي على HIGH أو LOW لما الـCS مش active.

---

### الـDevice Declaration: الـBoard Info vs Device Tree

**الطريقة القديمة (board files)**:

```c
static struct spi_board_info my_spi_devices[] __initdata = {
    {
        .modalias     = "ads7846",    /* ده اللي بيحصل match مع driver name */
        .platform_data = &ads_info,  /* board-specific data */
        .mode          = SPI_MODE_0,
        .max_speed_hz  = 1920000,
        .bus_num       = 1,           /* SPI controller #1 */
        .chip_select   = 0,           /* nCS0 */
        .irq           = GPIO_IRQ(31),
    },
};

spi_register_board_info(my_spi_devices, ARRAY_SIZE(my_spi_devices));
```

**الطريقة الحديثة (Device Tree)**:

```
/* في الـDTS */
&spi1 {
    status = "okay";
    cs-gpios = <&gpio1 5 GPIO_ACTIVE_LOW>;

    touchscreen@0 {
        compatible = "ti,ads7846";   /* يتطابق مع of_match_table في الـdriver */
        reg = <0>;                   /* chip_select = 0 */
        spi-max-frequency = <1920000>;
        pendown-gpio = <&gpio1 31 GPIO_ACTIVE_LOW>;
    };
};
```

الـSPI core بيقرأ الـDT ويبني الـ`spi_device` automatically.

---

### ملاحظة على الـSubsystems التانية

الـSPI framework بيعتمد على:

- **الـDevice Model** (`drivers/base/`): إطار الـ`struct device`, `struct bus_type`, الـ`probe/remove` lifecycle — الـSPI framework بيبني فوقيه مباشرة (`spi_bus_type`).
- **الـDMA Engine** (`drivers/dma/`): الـSPI core بيستخدم الـDMAengine API (`dmaengine_prep_slave_sg`) لما الـcontroller driver يدعم DMA — لازم تفهم الـscatterlist و`dma_map_sg` عشان تفهم الـmapping اللي الـcore بيعمله.
- **الـpinctrl subsystem** (`drivers/pinctrl/`): قبل ما الـcontroller driver يشتغل، الـpins لازم يتعملوا configure كـSPI function — ده بيحصل عبر الـpinctrl/pinmux API.

---

### ما بيمتلكه الـFramework مقابل ما بيفوّضه للـDrivers

| المسؤولية | الـSPI Core Framework يمتلكه | الـDriver بيفوّضه ليه |
|-----------|------------------------------|----------------------|
| Bus registration في sysfs | ✔ | |
| Device/Driver matching (modalias) | ✔ | |
| Message queue (FIFO per device) | ✔ | |
| kthread pump lifecycle | ✔ | |
| DMA mapping (sg_table, dma_map) | ✔ | |
| spi_sync wrapper فوق spi_async | ✔ | |
| Clock rate configuration | | ✔ Controller Driver |
| Chipselect toggling (timing) | | ✔ Controller Driver |
| FIFO/DMA register programming | | ✔ Controller Driver |
| Protocol interpretation (commands) | | ✔ Protocol Driver |
| Upper layer integration (MTD, input) | | ✔ Protocol Driver |
| Board-specific platform data | | ✔ Board/DTS config |

---

### مثال عملي: قراءة byte من SPI Flash

```c
/* في spi-nor.c — protocol driver بيقرأ من flash */
static int spi_nor_read(struct spi_nor *nor, loff_t from,
                        size_t len, u8 *buf)
{
    struct spi_device *spi = nor->spimem->spi;

    /* Transfer #1: إرسال READ command + address (4 bytes) */
    struct spi_transfer t[2] = {
        {
            .tx_buf = nor->bouncebuf,  /* [0x03, addr_hi, addr_mid, addr_lo] */
            .len    = 4,
        },
        /* Transfer #2: استقبال البيانات */
        {
            .rx_buf = buf,
            .len    = len,
        },
    };

    struct spi_message m;
    spi_message_init(&m);
    spi_message_add_tail(&t[0], &m);
    spi_message_add_tail(&t[1], &m);

    /* الـcore بياخد الـmessage، يحطها في queue الـcontroller،
     * الـkthread pump ينفذها، وبعدين يرجع النتيجة */
    return spi_sync(spi, &m);
}
```

طول الـmessage كلها (الـtransfer التاني بعد التاني)، الـchipselect فاضل active — الـflash شايف الـmessage كـatomic operation واحدة.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Enums والـ Config Options

#### SPI Mode Bits (من `include/uapi/linux/spi/spi.h`)

| Flag | Value | المعنى |
|------|-------|--------|
| `SPI_CPHA` | bit 0 | clock phase — sample on trailing edge |
| `SPI_CPOL` | bit 1 | clock polarity — clock starts high |
| `SPI_MODE_0` | 0 | CPOL=0, CPHA=0 |
| `SPI_MODE_1` | CPHA | CPOL=0, CPHA=1 |
| `SPI_MODE_2` | CPOL | CPOL=1, CPHA=0 |
| `SPI_MODE_3` | CPOL\|CPHA | CPOL=1, CPHA=1 |
| `SPI_CS_HIGH` | bit 2 | chipselect active high (عكس الافتراضي) |
| `SPI_LSB_FIRST` | bit 3 | إرسال LSB أول |
| `SPI_3WIRE` | bit 4 | MOSI و MISO على سلك واحد (half-duplex) |
| `SPI_LOOP` | bit 5 | loopback mode للاختبار |
| `SPI_NO_CS` | bit 6 | جهاز وحيد على الـ bus، بدون chipselect |
| `SPI_READY` | bit 7 | الـ slave يسحب الخط منخفضاً للإيقاف المؤقت |
| `SPI_TX_DUAL` | bit 8 | إرسال على 2 أسلاك (DUAL SPI) |
| `SPI_TX_QUAD` | bit 9 | إرسال على 4 أسلاك (QUAD SPI) |
| `SPI_RX_DUAL` | bit 10 | استقبال على 2 أسلاك |
| `SPI_RX_QUAD` | bit 11 | استقبال على 4 أسلاك |
| `SPI_CS_WORD` | bit 12 | toggle CS بعد كل word |
| `SPI_TX_OCTAL` | bit 13 | إرسال على 8 أسلاك |
| `SPI_RX_OCTAL` | bit 14 | استقبال على 8 أسلاك |
| `SPI_3WIRE_HIZ` | bit 15 | high-impedance turnaround |
| `SPI_RX_CPHA_FLIP` | bit 16 | flip CPHA على Rx فقط |
| `SPI_MOSI_IDLE_LOW` | bit 17 | MOSI يبقى منخفضاً وقت idle |
| `SPI_MOSI_IDLE_HIGH` | bit 18 | MOSI يبقى مرتفعاً وقت idle |

#### Kernel-only Mode Bits (من `include/linux/spi/spi.h`)

| Flag | Value | المعنى |
|------|-------|--------|
| `SPI_TPM_HW_FLOW` | bit 29 | TPM hardware flow control |
| `SPI_NO_RX` | bit 30 | مفيش سلك استقبال |
| `SPI_NO_TX` | bit 31 | مفيش سلك إرسال |

#### spi_controller->flags (Controller Constraints)

| Flag | Bit | المعنى |
|------|-----|--------|
| `SPI_CONTROLLER_HALF_DUPLEX` | 0 | الـ controller مش قادر full-duplex |
| `SPI_CONTROLLER_NO_RX` | 1 | مفيش buffer read |
| `SPI_CONTROLLER_NO_TX` | 2 | مفيش buffer write |
| `SPI_CONTROLLER_MUST_RX` | 3 | لازم يكون في rx دايماً |
| `SPI_CONTROLLER_MUST_TX` | 4 | لازم يكون في tx دايماً |
| `SPI_CONTROLLER_GPIO_SS` | 5 | GPIO chipselect مطلوب |
| `SPI_CONTROLLER_SUSPENDED` | 6 | الـ controller في suspend |
| `SPI_CONTROLLER_MULTI_CS` | 7 | يقدر يشغّل أكتر من CS في نفس الوقت |

#### spi_delay Units

| Constant | Value | الوحدة |
|----------|-------|--------|
| `SPI_DELAY_UNIT_USECS` | 0 | microseconds |
| `SPI_DELAY_UNIT_NSECS` | 1 | nanoseconds |
| `SPI_DELAY_UNIT_SCK` | 2 | clock cycles |

#### Transfer Width (nbits)

| Constant | Value | المعنى |
|----------|-------|--------|
| `SPI_NBITS_SINGLE` | 0x01 | سلك واحد (standard SPI) |
| `SPI_NBITS_DUAL` | 0x02 | سلكين (Dual SPI) |
| `SPI_NBITS_QUAD` | 0x04 | 4 أسلاك (QSPI) |
| `SPI_NBITS_OCTAL` | 0x08 | 8 أسلاك (OSPI) |

#### Transfer Error Flags

| Flag | Bit | المعنى |
|------|-----|--------|
| `SPI_TRANS_FAIL_NO_START` | 0 | الـ transfer مبدأش أصلاً |
| `SPI_TRANS_FAIL_IO` | 1 | فشل أثناء التنفيذ |

#### Multi-lane Mode

| Constant | Value | المعنى |
|----------|-------|--------|
| `SPI_MULTI_LANE_MODE_SINGLE` | 0 | lane واحد فقط |
| `SPI_MULTI_LANE_MODE_STRIPE` | 1 | word واحد per lane |
| `SPI_MULTI_LANE_MODE_MIRROR` | 2 | نفس الـ word على كل الـ lanes |

---

### 1. الـ Structs المهمة

#### `struct spi_device`
**الغرض:** proxy بين الـ protocol driver والـ controller — بيمثّل جهاز SPI واحد متوصل بالـ bus.

| Field | Type | الوصف |
|-------|------|-------|
| `dev` | `struct device` | دمج في Linux device model |
| `controller` | `struct spi_controller *` | بويintr به على الـ controller اللي بيديره |
| `max_speed_hz` | `u32` | أقصى clock speed مسموح للجهاز ده |
| `bits_per_word` | `u8` | حجم الـ word بالـ bits (default: 8) |
| `mode` | `u32` | SPI mode bits (CPOL/CPHA + flags) |
| `irq` | `int` | رقم الـ interrupt للجهاز |
| `controller_state` | `void *` | state خاص بالـ controller لكل device |
| `controller_data` | `void *` | بيانات board-specific (DMA hints, etc.) |
| `modalias` | `char[]` | اسم الـ driver المطلوب (للـ hotplug) |
| `chip_select[]` | `u8[4]` | مصفوفة الـ CS الفيزيائية |
| `cs_gpiod[]` | `struct gpio_desc *[4]` | GPIO descriptors للـ CS |
| `word_delay` | `struct spi_delay` | delay بين الـ words |
| `cs_setup/hold/inactive` | `struct spi_delay` | timing حول الـ CS |
| `pcpu_statistics` | per-CPU | إحصائيات أداء الجهاز |
| `tx_lane_map/rx_lane_map` | `u8[]` | تعيين الـ lanes للـ multi-lane |

---

#### `struct spi_controller`
**الغرض:** يمثّل الـ hardware SPI controller — القلب اللي بيدير كل عمليات الـ bus.

**Fields الأساسية:**

| Field | Type | الوصف |
|-------|------|-------|
| `dev` | `struct device` | دمج في device model |
| `bus_num` | `s16` | رقم الـ bus (من Board/SoC) |
| `num_chipselect` | `u16` | عدد الـ CS المتاحة |
| `mode_bits` | `u32` | الـ mode flags اللي بيفهمها الـ controller |
| `bits_per_word_mask` | `u32` | bitmask لحجم الـ word المدعوم |
| `min/max_speed_hz` | `u32` | نطاق الـ clock المدعوم |
| `flags` | `u16` | قيود الـ controller (HALF_DUPLEX, etc.) |
| `target` | `bool` | هل ده SPI target مش host؟ |
| `io_mutex` | `struct mutex` | يحمي الـ physical bus access |
| `bus_lock_mutex` | `struct mutex` | يحمي exclusive bus lock |
| `queue` | `struct list_head` | قائمة الـ messages المنتظرة |
| `queue_lock` | `spinlock_t` | يحمي الـ queue |
| `cur_msg` | `struct spi_message *` | الـ message اللي شغّال دلوقتي |
| `kworker` | `struct kthread_worker *` | الـ thread اللي بيشغّل الـ message pump |
| `dma_tx/dma_rx` | `struct dma_chan *` | قنوات الـ DMA |
| `cs_gpiods` | `struct gpio_desc **` | GPIO chipselects |

**الـ Callbacks (vtable):**

| Callback | المتى يُستدعى |
|----------|--------------|
| `setup(spi)` | تهيئة clock/mode للجهاز |
| `cleanup(spi)` | تحرير controller_state |
| `transfer(spi, msg)` | deprecated — إضافة message للـ queue |
| `prepare_transfer_hardware(ctlr)` | قبل ما تبدأ الـ transfers |
| `unprepare_transfer_hardware(ctlr)` | لما الـ queue تفضى |
| `transfer_one_message(ctlr, msg)` | تشغيل message كاملة |
| `transfer_one(ctlr, spi, xfer)` | تشغيل transfer واحدة |
| `prepare_message(ctlr, msg)` | DMA mapping قبل الـ message |
| `unprepare_message(ctlr, msg)` | undo الـ prepare |
| `set_cs(spi, enable)` | تحكم في chipselect |
| `handle_err(ctlr, msg)` | معالجة الأخطاء |
| `optimize_message(msg)` | تحسين الـ message قبل التنفيذ |
| `can_dma(ctlr, spi, xfer)` | هل ممكن DMA لهذا الـ transfer؟ |

---

#### `struct spi_transfer`
**الغرض:** يمثّل عملية read/write واحدة — الـ unit الأساسية للنقل.

| Field | Type | الوصف |
|-------|------|-------|
| `tx_buf` | `const void *` | buffer الإرسال (DMA-safe, أو NULL) |
| `rx_buf` | `void *` | buffer الاستقبال (DMA-safe, أو NULL) |
| `len` | `unsigned` | حجم البيانات بالـ bytes |
| `speed_hz` | `u32` | clock خاص بهذا الـ transfer (override) |
| `bits_per_word` | `u8` | word size خاص بهذا الـ transfer (override) |
| `cs_change` | `unsigned:1` | toggle CS بعد الـ transfer |
| `cs_off` | `unsigned:1` | نفّذ الـ transfer وCS مش مفعّل |
| `tx_nbits/rx_nbits` | `unsigned:4` | عدد الأسلاك للإرسال/الاستقبال |
| `delay` | `struct spi_delay` | delay بعد الـ transfer |
| `cs_change_delay` | `struct spi_delay` | delay بعد toggle الـ CS |
| `word_delay` | `struct spi_delay` | delay بين الـ words |
| `tx_sg/rx_sg` | `struct sg_table` | scatterlist للـ DMA |
| `tx_dma/rx_dma` | `dma_addr_t` | عناوين الـ DMA |
| `error` | `u16` | كود الخطأ بعد التنفيذ |
| `effective_speed_hz` | `u32` | الـ speed الفعلي بعد التنفيذ |
| `transfer_list` | `struct list_head` | ربط الـ transfers في الـ message |
| `ptp_sts` | pointer | PTP timestamp للـ timing |

---

#### `struct spi_message`
**الغرض:** تسلسل atomic من الـ transfers — بيتنفّذوا كوحدة واحدة على الـ bus.

| Field | Type | الوصف |
|-------|------|-------|
| `transfers` | `struct list_head` | قائمة الـ spi_transfer |
| `spi` | `struct spi_device *` | الجهاز المستهدف |
| `complete` | `void (*)(void *)` | callback لما ينتهي |
| `context` | `void *` | argument للـ callback |
| `status` | `int` | نتيجة التنفيذ (0 أو errno) |
| `frame_length` | `unsigned` | إجمالي الـ bytes في الـ message |
| `actual_length` | `unsigned` | الـ bytes اللي اتنقلت فعلاً |
| `queue` | `struct list_head` | ربط في قائمة الـ controller |
| `state` | `void *` | state للـ controller أثناء التنفيذ |
| `opt_state` | `void *` | state للـ optimize/unoptimize |
| `resources` | `struct list_head` | spi_res list للـ cleanup |
| `prepared` | `bool` | هل اتعمله prepare_message؟ |
| `optimized` | `bool` | هل اتعمله optimize؟ |
| `offload` | `struct spi_offload *` | optional offload engine |

---

#### `struct spi_driver`
**الغرض:** يمثّل الـ protocol driver — اللي بيتعامل مع جهاز SPI محدد.

| Field | Type | الوصف |
|-------|------|-------|
| `driver` | `struct device_driver` | دمج في driver model |
| `id_table` | `const struct spi_device_id *` | قائمة الأجهزة المدعومة |
| `probe` | `int (*)(struct spi_device *)` | ربط الـ driver بالجهاز |
| `remove` | `void (*)(struct spi_device *)` | فك الربط |
| `shutdown` | `void (*)(struct spi_device *)` | إيقاف تشغيل النظام |

---

#### `struct spi_board_info`
**الغرض:** template ثابت من الـ board init code يصف جهاز SPI قبل ما الـ controller يُسجَّل.

| Field | Type | الوصف |
|-------|------|-------|
| `modalias` | `char[]` | اسم الـ driver |
| `platform_data` | `const void *` | بيانات device-specific |
| `controller_data` | `void *` | hints للـ controller (DMA, etc.) |
| `irq` | `int` | رقم الـ IRQ |
| `max_speed_hz` | `u32` | أقصى clock |
| `bus_num` | `u16` | رقم الـ SPI bus |
| `chip_select` | `u16` | رقم الـ CS |
| `mode` | `u32` | SPI mode bits |

---

#### `struct spi_delay`
**الغرض:** يعبّر عن مدة انتظار بوحدات مختلفة.

```c
struct spi_delay {
    u16 value; /* القيمة */
    u8  unit;  /* الوحدة: USECS=0, NSECS=1, SCK=2 */
};
```

---

#### `struct spi_statistics`
**الغرض:** إحصائيات per-CPU للـ spi_device والـ spi_controller.

| Field | الوصف |
|-------|-------|
| `messages` | عدد الـ messages اللي اتنفّذت |
| `transfers` | عدد الـ transfers |
| `errors` | عدد الأخطاء |
| `timedout` | عدد mرات الـ timeout |
| `spi_sync/spi_async` | إحصاء نوع الـ calls |
| `bytes/bytes_tx/bytes_rx` | الـ bytes المنقولة |
| `transfer_bytes_histo[]` | histogram لأحجام الـ transfers |

---

### 2. مخططات العلاقات بين الـ Structs

```
┌─────────────────────────────────────────────────────────────┐
│                    Linux SPI Subsystem                       │
│                                                              │
│  spi_board_info[]          spi_driver                       │
│  (board init code)         (protocol driver)                │
│       │                         │                           │
│       │ spi_register_board_info │ spi_register_driver       │
│       ▼                         ▼                           │
│  ┌─────────────────────────────────────────────────┐        │
│  │              SPI Core (drivers/spi/spi.c)        │        │
│  └─────────────────────────────────────────────────┘        │
│            │                         │                      │
│            ▼                         ▼                      │
│    ┌──────────────┐         ┌──────────────────┐           │
│    │ spi_controller│◄────────│   spi_device     │           │
│    │              │ .controller             │   │           │
│    │  .queue ──►──┤         │  .dev            │           │
│    │  .cur_msg    │         │  .mode           │           │
│    │  .kworker    │         │  .max_speed_hz   │           │
│    │  .dma_tx/rx  │         │  .chip_select[]  │           │
│    └──────────────┘         └──────────────────┘           │
│            │                         │                      │
│            │                         ▼                      │
│            │                ┌──────────────────┐           │
│            │                │   spi_message    │           │
│            │                │  .transfers ──►──┤           │
│            │                │  .complete()     │           │
│            └───►queue────►  │  .spi ──────────►│           │
│                             └──────────────────┘           │
│                                      │                      │
│                                      ▼                      │
│                             ┌──────────────────┐           │
│                             │  spi_transfer    │           │
│                             │  .tx_buf         │           │
│                             │  .rx_buf         │           │
│                             │  .len            │           │
│                             │  .transfer_list ─┤──► next   │
│                             └──────────────────┘           │
└─────────────────────────────────────────────────────────────┘
```

**تفاصيل العلاقات:**

```
spi_controller
    │
    ├──► struct device dev           (embedded في device model)
    ├──► struct list_head queue      (list of spi_message)
    ├──► struct spi_message *cur_msg (الـ message الشغّال)
    ├──► struct kthread_worker       (message pump thread)
    ├──► struct dma_chan *dma_tx/rx  (DMA channels)
    └──► struct gpio_desc **cs_gpiods

spi_device
    │
    ├──► struct device dev
    ├──► struct spi_controller *controller   (مين بيديره)
    ├──► struct gpio_desc *cs_gpiod[]        (CS GPIO pins)
    └──► struct spi_delay word_delay/cs_*

spi_message
    │
    ├──► struct spi_device *spi
    ├──► struct list_head transfers    (linked list of spi_transfer)
    ├──► struct list_head resources    (spi_res for cleanup)
    └──► struct spi_offload *offload   (optional)

spi_transfer
    │
    ├──► const void *tx_buf / void *rx_buf
    ├──► struct sg_table tx_sg/rx_sg   (for DMA)
    ├──► struct spi_delay delay/word_delay
    └──► struct list_head transfer_list  (ربط في message)
```

---

### 3. مخطط دورة الحياة (Lifecycle)

#### Controller Driver Lifecycle

```
Platform/OF/ACPI probes controller driver
          │
          ▼
    spi_alloc_host(dev, sizeof(private))
          │
          ▼
    تهيئة ctlr->fields:
      - bus_num
      - num_chipselect
      - mode_bits
      - setup/transfer_one_message/etc.
          │
          ▼
    spi_register_controller(ctlr)
          │
          ├──► يبدأ الـ kthread (message pump)
          ├──► يسجّل الـ device في sysfs
          └──► يربط spi_board_info بالأجهزة
                    │
                    ▼
              spi_device per chip
                    │
                    ▼
              driver_match → probe()
                    │
                    ▼
           [Controller يشتغل عادي]
                    │
          (عند الـ unload)
                    ▼
    spi_unregister_controller(ctlr)
          │
          ├──► يوقف الـ kthread
          ├──► يحذف كل spi_device children
          └──► يشيل الـ sysfs entries
```

#### Protocol Driver (spi_driver) Lifecycle

```
module_init / module_spi_driver(CHIP_driver)
          │
          ▼
    spi_register_driver(&CHIP_driver)
          │
          ▼
    driver core matches modalias
          │
          ▼
    CHIP_probe(struct spi_device *spi)
          │
          ├──► spi_setup(spi)          ← تهيئة clock/mode
          ├──► alloc private state
          ├──► spi_set_drvdata(spi, chip)
          └──► تسجيل في upper layer (input/MTD/ALSA/etc.)
                    │
                    ▼
           [Driver يشتغل، يرسل messages]
                    │
          (عند الـ unload/unbind)
                    ▼
    CHIP_remove(struct spi_device *spi)
          │
          ├──► إيقاف أي I/O pending
          └──► تحرير الـ resources
```

#### spi_message Lifecycle

```
Protocol driver يبني الـ message:

    spi_message_init(&msg)           ← تصفير + init list
          │
    spi_message_add_tail(&t, &msg)   ← ضمّ transfers
          │
          ▼
    spi_async(spi, &msg)             ← يضيف للـ queue
      أو
    spi_sync(spi, &msg)              ← ينتظر blocking
          │
          ▼
    [Controller queue receives msg]
          │
          ▼
    ctlr->prepare_transfer_hardware()
          │
          ▼
    ctlr->prepare_message(ctlr, msg)  ← DMA mapping
          │
          ▼
    ctlr->transfer_one_message()
      أو per-transfer:
    ctlr->transfer_one()  [لكل transfer في الـ list]
          │
          ▼
    spi_finalize_current_message()
          │
          ▼
    msg->complete(msg->context)       ← يُبلّغ صاحب الـ msg
          │
          ▼
    ctlr->unprepare_message()
          │
    [لو الـ queue فاضية:]
    ctlr->unprepare_transfer_hardware()
```

---

### 4. مخططات Call Flow

#### الـ Async I/O Path (المسار الرئيسي)

```
Protocol driver:
    spi_async(spi, msg)
        │
        ▼
    [SPI Core validates msg]
        │
        ▼
    __spi_queued_transfer()
        │  يضيف msg لـ ctlr->queue
        ▼
    kthread_queue_work(ctlr->kworker, &ctlr->pump_messages)
        │
        ▼
    [Message Pump Thread wakes up]
        │
        ▼
    spi_pump_messages()
        │
        ▼
    ctlr->prepare_transfer_hardware(ctlr)     [once per batch]
        │
        ▼
    ctlr->optimize_message(msg)               [optional]
        │
        ▼
    ctlr->prepare_message(ctlr, msg)          [DMA map]
        │
        ▼
    ctlr->transfer_one_message(ctlr, msg)
        │  ──► Hardware registers
        │      DMA engine
        │      IRQ fires when done
        ▼
    spi_finalize_current_message(ctlr)
        │
        ▼
    msg->complete(msg->context)               [callback]
        │
        ▼
    ctlr->unoptimize_message(msg)             [optional]
```

#### الـ Sync I/O Path (الـ wrapper)

```
Protocol driver:
    spi_sync(spi, msg)
        │
        ▼
    [Fast path check: queue_empty && !must_async?]
        │ YES                       │ NO
        ▼                           ▼
    __spi_sync()              spi_async() + wait_for_completion()
    (تنفيذ مباشر في السياق)
        │
        ▼
    spi_transfer_one_message()
        │
        ▼
    [نفس مسار الـ async من transfer_one_message فصاعداً]
```

#### الـ transfer_one Path (Generic Core)

```
ctlr->transfer_one_message(ctlr, msg)  [generic implementation in spi.c]
        │
        ▼
    for each xfer in msg->transfers:
        │
        ▼
        ctlr->set_cs(spi, true)        [assert CS]
            │
            ▼
        [DMA or PIO?]
        can_dma()? → dma_map_sg()
            │
            ▼
        ctlr->transfer_one(ctlr, spi, xfer)
            │  → writes HW TX FIFO
            │  → starts DMA
            │  → returns 0 (done) or 1 (in-progress)
            ▼
        wait_for_completion(&ctlr->xfer_completion)
            │  [IRQ handler calls spi_finalize_current_transfer()]
            ▼
        spi_transfer_delay_exec(xfer)  [optional delay]
            │
        [cs_change? → toggle CS]
            │
        next xfer ...
        │
        ▼ (after all xfers)
    ctlr->set_cs(spi, false)           [deassert CS]
        │
        ▼
    spi_finalize_current_message(ctlr)
```

#### مسار إنشاء spi_device من Board Info

```
spi_register_board_info(info[], n)
        │  يحفظهم في global list
        ▼
spi_register_controller(ctlr)
        │
        ▼
    of_register_spi_devices(ctlr)     [من Device Tree]
      أو
    spi_match_board_info(ctlr)        [من static table]
        │
        ▼
    spi_alloc_device(ctlr)
        │
        ▼
    spi_add_device(spi)
        │
        ▼
    device_add(&spi->dev)
        │
        ▼
    driver core → spi_driver->probe(spi)
```

---

### 5. استراتيجية الـ Locking

#### الـ Locks الموجودة وما يحميه

| Lock | النوع | ما يحميه |
|------|-------|---------|
| `ctlr->queue_lock` | `spinlock_t` | الـ `queue` list + الـ `running/busy` flags |
| `ctlr->io_mutex` | `mutex` | physical bus access عند تنفيذ الـ transfers |
| `ctlr->add_lock` | `mutex` | منع إضافة نفس الـ CS مرتين |
| `ctlr->bus_lock_mutex` | `mutex` | exclusive bus lock (لما driver يحتاج يملك الـ bus) |
| `ctlr->bus_lock_spinlock` | `spinlock_t` | `bus_lock_flag` نفسه |
| `spi_statistics->syncp` | `u64_stats_sync` (seqcount) | per-CPU statistics counters |

#### ترتيب الـ Locks (Lock Ordering)

```
الأعلى (يُمسك أول):
    bus_lock_mutex          ← exclusive bus ownership
        │
        ▼
    io_mutex                ← physical transfer
        │
        ▼
    queue_lock (spinlock)   ← queue manipulation
        │
        ▼
    (لا يُمسك lock آخر)
```

> **قاعدة مهمة:** `queue_lock` هو spinlock ممكن يتأخد من IRQ context، فأي كود بيمسكه **لازم** يعمل `spin_lock_irqsave` مش `spin_lock` فقط.

#### الـ bus_lock للـ Multi-message Transactions

```c
/* Driver يحتاج يرسل sequence من messages بدون تدخل */
spi_bus_lock(ctlr);         /* يمسك bus_lock_mutex */
    spi_sync_locked(spi, &msg1);
    spi_sync_locked(spi, &msg2);
spi_bus_unlock(ctlr);       /* يحرر */
```

بدون الـ lock، ممكن driver تاني يدخل بـ message بينهم.

#### الـ DMA و Locking

- لما `can_dma()` ترجع `true`، الـ core بيعمل `dma_map_sg()` **قبل** ما يناده `transfer_one()`.
- الـ DMA transfer بيخلص asynchronously — الـ IRQ handler بينادي `spi_finalize_current_transfer()` اللي بيعمل `complete()` على `ctlr->xfer_completion`.
- الـ `io_mutex` بيحمي إن مفيش اتنين بيعملوا transfer في نفس الوقت.

#### الـ Statistics Locking

```c
/* Per-CPU update — بدون global lock */
SPI_STATISTICS_INCREMENT_FIELD(pcpu_stats, messages);
/* داخلياً بيعمل:
   get_cpu()
   u64_stats_update_begin(&lstats->syncp)  ← seqcount write lock
   u64_stats_inc(...)
   u64_stats_update_end(&lstats->syncp)
   put_cpu()
*/
```

القراءة بتستخدم `u64_stats_fetch_begin/retry` للـ consistency بدون lock كامل.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### مجموعة Registration & Lifecycle

| Function | Where Used | Purpose |
|---|---|---|
| `spi_register_board_info()` | board init | يسجّل جدول الأجهزة الثابتة |
| `spi_alloc_host()` | controller driver probe | يخصص `spi_controller` لـ host |
| `spi_alloc_target()` | controller driver probe | يخصص `spi_controller` لـ target/slave |
| `devm_spi_alloc_host()` | controller driver probe | نفسه بـ devres |
| `spi_register_controller()` | controller driver probe | ينشر الـ controller على الـ bus |
| `devm_spi_register_controller()` | controller driver probe | نفسه بـ devres |
| `spi_unregister_controller()` | controller driver remove | يلغي تسجيل الـ controller |
| `spi_register_driver()` | protocol driver module_init | يسجّل الـ protocol driver |
| `spi_unregister_driver()` | protocol driver module_exit | يلغي تسجيله |
| `module_spi_driver()` | protocol driver | macro يستغني عن init/exit |

#### مجموعة Device Management

| Function | Purpose |
|---|---|
| `spi_alloc_device()` | يخصص `spi_device` جديد |
| `spi_add_device()` | يضيف device مخصص مسبقاً للـ bus |
| `spi_new_device()` | يجمع alloc + add في خطوة واحدة |
| `spi_unregister_device()` | يحذف device من الـ bus |
| `spi_setup()` | يضبط mode/clock/wordsize للـ device |

#### مجموعة I/O — Protocol Driver API

| Function | Blocking? | Context |
|---|---|---|
| `spi_async()` | No | any (IRQ, task, ...) |
| `spi_sync()` | Yes (sleep) | process context only |
| `spi_sync_locked()` | Yes | process, داخل bus lock |
| `spi_write()` | Yes | process |
| `spi_read()` | Yes | process |
| `spi_write_then_read()` | Yes | process, small data only |
| `spi_w8r8()` | Yes | process, write 1B read 1B |
| `spi_w8r16()` | Yes | process, write 1B read 2B |
| `spi_w8r16be()` | Yes | process, big-endian variant |
| `spi_sync_transfer()` | Yes | process, array of transfers |

#### مجموعة Message & Transfer Helpers

| Function | Purpose |
|---|---|
| `spi_message_init()` | يبدأ `spi_message` من الصفر |
| `spi_message_init_no_memset()` | init بدون صفر (للـ embedded structs) |
| `spi_message_init_with_transfers()` | init + إضافة array من transfers |
| `spi_message_add_tail()` | يضيف transfer لآخر الـ message |
| `spi_transfer_del()` | يحذف transfer من الـ message |
| `spi_message_alloc()` | يخصص message + N transfers من heap |
| `spi_message_free()` | يحرر ما خصصه `spi_message_alloc()` |
| `spi_optimize_message()` | pre-optimizes repeated messages |
| `spi_unoptimize_message()` | يلغي الـ optimization |

#### مجموعة Controller Queue Callbacks

| Function | Caller |
|---|---|
| `spi_finalize_current_message()` | controller driver (بعد نهاية message) |
| `spi_finalize_current_transfer()` | controller driver (بعد نهاية transfer) |
| `spi_get_next_queued_message()` | controller driver |
| `spi_controller_suspend()` | power management |
| `spi_controller_resume()` | power management |

#### مجموعة Helpers & Accessors

| Function | Purpose |
|---|---|
| `spi_set_drvdata()` / `spi_get_drvdata()` | driver private data |
| `spi_get_ctldata()` / `spi_set_ctldata()` | controller_state على الـ device |
| `spi_controller_get_devdata()` | private data الـ controller |
| `spi_controller_set_devdata()` | يضبطها |
| `spi_dev_get()` / `spi_dev_put()` | refcount على الـ spi_device |
| `spi_controller_get()` / `spi_controller_put()` | refcount على الـ controller |
| `spi_get_chipselect()` / `spi_set_chipselect()` | CS index helpers |
| `spi_is_csgpiod()` | يتحقق هل CS عبر GPIO |
| `spi_max_message_size()` | اقصى حجم message يقبله الـ controller |
| `spi_max_transfer_size()` | اقصى حجم transfer |
| `spi_is_bpw_supported()` | يتحقق هل الـ bits_per_word مدعوم |
| `spi_bpw_to_bytes()` | يحول bpw لـ bytes |
| `spi_controller_is_target()` | هل الـ controller في target mode |
| `spi_transfer_is_last()` | هل الـ transfer اخر واحد في الـ message |
| `spi_bus_lock()` / `spi_bus_unlock()` | exclusive bus access |
| `spi_split_transfers_maxsize()` | يقسم transfers تتعدى maxsize |
| `spi_split_transfers_maxwords()` | يقسم transfers تتعدى maxwords |

---

### المجموعة الاولى: Board-Level Registration

الهدف من هذه المجموعة هو اخبار الـ SPI core بالاجهزة الموجودة على الـ board قبل حتى ان يُحمَّل اي driver. دي الـ static configuration path الكلاسيكية.

---

#### `spi_register_board_info()`

```c
int spi_register_board_info(struct spi_board_info const *info, unsigned n);
```

بتاخد array من `spi_board_info` وتحفظها في قائمة global داخل الـ SPI core. لما اي `spi_controller` يتسجّل لاحقاً، الـ core بيمشي على القائمة دي وبينشئ الـ `spi_device` المناسبة تلقائياً. الـ function مش بتنشئ الـ devices فوراً — بس بتحجز المعلومات.

| Parameter | شرح |
|---|---|
| `info` | pointer لاول عنصر في الـ array، كل عنصر يوصف device واحد |
| `n` | عدد العناصر في الـ array، عادةً `ARRAY_SIZE(...)` |

**Return:** `0` دايماً عملياً (الـ implementation البسيطة).

**Key details:**
- لازم تتكلم منها في `arch_initcall` او `board_init` — قبل ما controller يتسجّل.
- مفيش مقابلها unregister — الـ board info بتفضل طول عمر الـ kernel.
- الـ `bus_num` في كل entry لازم يطابق `spi_controller.bus_num` اللي هيُنشئ الـ devices.

---

### المجموعة الثانية: Controller Allocation & Registration

دي الـ API اللي بيستخدمها controller driver في الـ `probe()` عشان يخصص ويسجّل الـ SPI host.

---

#### `spi_alloc_host()` / `spi_alloc_target()`

```c
static inline struct spi_controller *spi_alloc_host(struct device *dev,
                                                    unsigned int size);

static inline struct spi_controller *spi_alloc_target(struct device *dev,
                                                      unsigned int size);
```

الاتنين بيكلّموا `__spi_alloc_controller()` اللي بيعمل `kzalloc` لـ `spi_controller` + `size` bytes extra للـ driver private data. الـ private data بتيجي مباشرةً بعد الـ struct في الميموري فيُوصلها عبر `spi_controller_get_devdata()`.

| Parameter | شرح |
|---|---|
| `dev` | الـ parent device (عادةً `&pdev->dev`) |
| `size` | حجم الـ driver-private struct اللي هيُخصَّص مع الـ controller |

**Return:** pointer لـ `spi_controller` جديد، او `NULL` لو فشل الـ alloc.

**Key details:**
- الـ `spi_controller.dev` بيتضبط كـ child للـ `dev` المُمرَّر.
- `spi_alloc_target()` بترجع `NULL` تلقائياً لو `CONFIG_SPI_SLAVE` مش مفعّل.
- استخدم `devm_spi_alloc_host()` لو عايز الـ devres تتكفّل بالـ free تلقائياً.

---

#### `spi_register_controller()`

```c
int spi_register_controller(struct spi_controller *ctlr);
```

بعد ما تملّي الـ `spi_controller` struct بالـ callbacks والـ bus_num وغيرها، الـ function دي بتنشره على الـ `spi_bus_type`. من هنا:
1. بتنشئ الـ device nodes في sysfs.
2. بتمشي على الـ board_info المحفوظة وبتنشئ الـ `spi_device` المقابلة.
3. الـ driver model بيحاول يـ bind كل device بـ driver مناسب.

```
spi_register_controller()
    +-- device_add(&ctlr->dev)
    +-- of_register_spi_devices()     // DT devices
    +-- acpi_register_spi_devices()   // ACPI devices
    +-- scan_boardinfo()              // static board_info list
            +-- spi_new_device()      // per each match
```

| Parameter | شرح |
|---|---|
| `ctlr` | الـ controller المُخصَّص بـ `spi_alloc_host()` |

**Return:** `0` on success, negative errno on failure.

**Key details:**
- لو `ctlr->bus_num` سالب، الـ core بيعطيه رقم dynamic.
- لازم كل الـ callbacks الالزامية (`setup`, و`transfer_one` او `transfer_one_message`) تكون مضبوطة قبل الاستدعاء.
- الـ `devm_spi_register_controller()` تعمل نفس الشيء بـ automatic unregister عند الـ device removal.

---

#### `spi_unregister_controller()`

```c
void spi_unregister_controller(struct spi_controller *ctlr);
```

عكس `spi_register_controller()` تماماً. بتحذف كل الـ `spi_device` التابعة، بتفصل الـ drivers المرتبطة، وبتحذف الـ device من sysfs.

**Key details:**
- بتستدعيها في `remove()` الـ platform driver.
- لو استخدمت `devm_spi_register_controller()` مش محتاجها صراحةً.
- الـ `spi_controller` نفسه بيتحرر لما الـ refcount يوصل صفر عبر `spi_controller_put()`.

---

### المجموعة الثالثة: Protocol Driver Registration

---

#### `spi_register_driver()` / `spi_unregister_driver()`

```c
/* macro -- passes THIS_MODULE automatically */
#define spi_register_driver(driver) \
    __spi_register_driver(THIS_MODULE, driver)

static inline void spi_unregister_driver(struct spi_driver *sdrv);
```

بتسجّل الـ `spi_driver` في `spi_bus_type` عن طريق `driver_register()`. الـ driver model بيعمل matching بناءً على `modalias` او `id_table`.

**Key details:**
- `module_spi_driver(__spi_driver)` macro بيستغني عن كتابة `module_init`/`module_exit` كلياً — استخدمه دايماً لو مفيش special setup.
- الـ `probe()` callback هتتكلم بـ `spi_device *` جاهز ومرتبط.

---

#### `spi_setup()`

```c
int spi_setup(struct spi_device *spi);
```

بتستدعي `ctlr->setup(spi)` اللي بتطبّق الاعدادات الجديدة (mode, speed, bits_per_word) على الـ hardware. لازم تتكلم منها من `probe()` قبل اول I/O، او في اي وقت مفيش message pending.

**Return:** `0` او negative errno.

**Key details:**
- الـ controller driver لازم يفترض ان في transfers جارية لـ devices تانية — ممنوع يعدّل shared registers مباشرةً.
- الـ `spi_device.mode` flags (SPI_MODE_0..3, SPI_CS_HIGH, SPI_LSB_FIRST, ...) بتتحوّل لـ hardware config هنا.

---

### المجموعة الرابعة: Device Dynamic Management

---

#### `spi_alloc_device()`

```c
struct spi_device *spi_alloc_device(struct spi_controller *ctlr);
```

بتخصص `spi_device` جديد بـ `kzalloc` وبتربطه بالـ `ctlr`. مفيدة لما محتاج تملّي الـ struct بيدك قبل ما تضيفه للـ bus.

---

#### `spi_add_device()`

```c
int spi_add_device(struct spi_device *spi);
```

بتكمّل تسجيل الـ device المُخصَّص بـ `spi_alloc_device()`. بتعمل validation، بتعمل `device_add()`، والـ driver model بيحاول يـ bind driver.

---

#### `spi_new_device()`

```c
struct spi_device *spi_new_device(struct spi_controller *ctlr,
                                  struct spi_board_info *chip);
```

بتجمع `spi_alloc_device()` + تملئة البيانات من `spi_board_info` + `spi_add_device()` في call واحدة. للـ dynamic hotplug scenarios.

---

#### `spi_unregister_device()`

```c
void spi_unregister_device(struct spi_device *spi);
```

بتشيل الـ device من الـ bus وبتستدعي `remove()` على الـ driver المرتبط، وبعدين بتعمل `device_unregister()`.

---

### المجموعة الخامسة: Message & Transfer Construction

دي الـ API اللي protocol driver بيبني بيها الـ I/O requests قبل ما يبعتها.

---

#### `spi_message_init()`

```c
static inline void spi_message_init(struct spi_message *m);
```

بتعمل `memset(m, 0)` وبعدين `INIT_LIST_HEAD` للـ transfers list والـ resources list. لازم تتكلم منها على اي `spi_message` قبل استخدامه.

---

#### `spi_message_add_tail()`

```c
static inline void spi_message_add_tail(struct spi_transfer *t,
                                         struct spi_message *m);
```

بتضيف الـ `spi_transfer` لاخر قايمة الـ transfers في الـ message. الـ transfers بتتنفّذ بالترتيب اللي اتضافوا بيه.

---

#### `spi_message_init_with_transfers()`

```c
static inline void spi_message_init_with_transfers(struct spi_message *m,
                                                    struct spi_transfer *xfers,
                                                    unsigned int num_xfers);
```

بتعمل `spi_message_init()` وبعدين loop بتضيف كل الـ transfers. shortcut مريحة لو عندك array من transfers.

---

#### `spi_message_alloc()` / `spi_message_free()`

```c
static inline struct spi_message *spi_message_alloc(unsigned ntrans, gfp_t flags);
static inline void spi_message_free(struct spi_message *m);
```

الـ `alloc` بتخصص بـ `kzalloc` struct واحد بيحتوي الـ `spi_message` + `ntrans` عدد من `spi_transfer` وبتضيفهم كلهم للـ message. الـ `free` ببساطة `kfree`. مفيدة لما الـ message والـ transfers مش جزء من struct تاني.

| Parameter | شرح |
|---|---|
| `ntrans` | عدد الـ transfers المطلوبة |
| `flags` | GFP flags — عادةً `GFP_KERNEL` |

---

#### `spi_optimize_message()` / `spi_unoptimize_message()`

```c
int spi_optimize_message(struct spi_device *spi, struct spi_message *msg);
void spi_unoptimize_message(struct spi_message *msg);
```

لما protocol driver بيبعت نفس الـ message كتير (زي sensor polling)، الـ `optimize_message` بيخلي الـ controller يعمل DMA mapping واعدادات مسبقة مرة واحدة بدل ما يعيدها مع كل ارسال. لازم `spi_unoptimize_message()` تتكلم قبل تعديل الـ message او تحريره.

**Key details:**
- `devm_spi_optimize_message()` بتعمل الـ unoptimize تلقائياً عند الـ device removal.
- الـ `msg->pre_optimized` flag بيتضبط لو الـ driver عمل optimize يدوياً.

---

### المجموعة السادسة: I/O Submission — الـ Core API

---

#### `spi_async()`

```c
int spi_async(struct spi_device *spi, struct spi_message *message);
```

الـ **primitive الاساسية** لكل الـ SPI I/O. بتضيف الـ message لـ queue الـ controller وبترجع فوراً. الـ completion بيتم عبر `message->complete(message->context)` callback اللي بيتكلم من context مختلف (عادةً threaded IRQ او kworker).

**Pseudocode flow:**

```
spi_async(spi, msg)
    lock ctlr->bus_lock_spinlock / check bus_lock_flag
    if ctlr->queue_empty && !ctlr->must_async
        --> try spi_sync fast path (opportunistic)
    else
        --> ctlr->transfer(spi, msg)   // deprecated path
        or
        --> list_add_tail(msg, ctlr->queue)
           kthread_queue_work(ctlr->kworker, &pump_messages)
```

| Parameter | شرح |
|---|---|
| `spi` | الـ target device |
| `message` | الـ message المُعبَّا بالـ transfers — لازم يبقى valid لحد ما الـ complete callback يتكلم |

**Return:** `0` لو اتضاف للـ queue، negative errno لو في error فوري (validation fail).

**Key details:**
- الـ buffers لازم تكون DMA-safe (من heap او page pool — **مش** stack).
- بعد الـ call، ما تلمسش الـ message او الـ transfers حتى يجي الـ callback.
- لو حصل error في النص، الـ CS بيتشال والـ message بيوقف.

---

#### `spi_sync()`

```c
int spi_sync(struct spi_device *spi, struct spi_message *message);
```

wrapper synchronous فوق `spi_async()`. بتحط `completion` في الـ message وبتستنى `wait_for_completion()`. لازم تتكلم منها من process context بس — ممنوع من interrupt context.

**Return:** `message->status` — صفر لو نجح، negative errno لو فشل.

**Key details:**
- الـ "fast path": لو الـ queue فاضية والـ controller جاهز، ممكن الـ transfer يتنفذ inline بدون queuing (opportunistic sync).
- `spi_sync_locked()` للاستخدام بعد `spi_bus_lock()` عشان exclusive access.

---

#### `spi_write()` / `spi_read()`

```c
static inline int spi_write(struct spi_device *spi, const void *buf, size_t len);
static inline int spi_read(struct spi_device *spi, void *buf, size_t len);
```

الاتنين wrappers بتبني `spi_transfer` واحدة وبتكلموا `spi_sync_transfer()`. الـ `spi_write` بتضبط `tx_buf` بس (rx_buf=NULL => data shifted in يُرمى). الـ `spi_read` العكس.

**Key details:**
- **Process context فقط** — بيناموا.
- للـ full-duplex استخدم `spi_sync_transfer()` مباشرةً مع `tx_buf` و`rx_buf` الاتنين.

---

#### `spi_write_then_read()`

```c
int spi_write_then_read(struct spi_device *spi,
                         const void *txbuf, unsigned n_tx,
                         void *rxbuf, unsigned n_rx);
```

بتعمل write بعدها read في message واحدة (تسلسلية — مش full duplex). بتعمل **internal copy** للـ buffers لانها محتاجة DMA-safe memory، فمش مناسبة للـ data كبير — للـ RPC-style small transactions بس.

| Parameter | شرح |
|---|---|
| `txbuf` | البيانات اللي هتتبعت |
| `n_tx` | حجم الـ tx بالـ bytes |
| `rxbuf` | buffer يستقبل الرد |
| `n_rx` | حجم الـ rx بالـ bytes |

**Return:** `0` or negative errno.

---

#### `spi_w8r8()` / `spi_w8r16()` / `spi_w8r16be()`

```c
static inline ssize_t spi_w8r8(struct spi_device *spi, u8 cmd);
static inline ssize_t spi_w8r16(struct spi_device *spi, u8 cmd);
static inline ssize_t spi_w8r16be(struct spi_device *spi, u8 cmd);
```

Convenience wrappers فوق `spi_write_then_read()`. كلهم بيبعتوا byte واحد command وبيستقبلوا 1 او 2 bytes رد. الـ `spi_w8r16be` بيعمل `be16_to_cpu()` على الـ result.

**Return:** القيمة المقروءة كـ `ssize_t` (unsigned)، او negative errno لو فشل. الـ trick ان الـ negative values بتكتشف الـ error تلقائياً.

```c
/* مثال: قراءة temperature من sensor */
ssize_t temp_raw = spi_w8r16be(spi, READ_TEMP_CMD);
if (temp_raw < 0)
    return temp_raw; /* error */
int temp_celsius = (int)temp_raw >> 4; /* 12-bit result */
```

---

#### `spi_sync_transfer()`

```c
static inline int spi_sync_transfer(struct spi_device *spi,
                                     struct spi_transfer *xfers,
                                     unsigned int num_xfers);
```

بتعمل `spi_message_init_with_transfers()` وبعدين `spi_sync()`. لو عندك array من transfers جاهزة دي اسرع طريقة.

---

### المجموعة السابعة: Bus Locking

---

#### `spi_bus_lock()` / `spi_bus_unlock()`

```c
int spi_bus_lock(struct spi_controller *ctlr);
int spi_bus_unlock(struct spi_controller *ctlr);
```

بتامن exclusive access للـ SPI bus. بعد `spi_bus_lock()`، الـ bus متاح بس لـ `spi_sync_locked()` حتى `spi_bus_unlock()`. مفيد لو protocol driver محتاج يعمل sequence من messages ضامن ان محدش تاني يتدخل.

**Key details:**
- مش لازم تستخدمها في الحالة العادية — الـ message atomicity كافية.
- `bus_lock_flag` في الـ controller بيشير ان الـ bus محجوز.

---

### المجموعة الثامنة: Controller Callbacks — ما يُنفَّذه بالـ Controller Driver

دي الـ ops اللي controller driver **لازم** او **ممكن** يعرّفها في الـ `spi_controller` struct.

---

#### `ctlr->setup(struct spi_device *spi)`

بيتكلم لما protocol driver يكلم `spi_setup()`. مهمته يحفظ الـ config (speed, mode, wordsize) بشكل مناسب — مش يطبّقها على الـ hardware على طول. **ممنوع** يعدّل shared registers لان في transfers جارية لـ devices تانية.

---

#### `ctlr->transfer_one_message(struct spi_controller *ctlr, struct spi_message *mesg)`

الـ controller driver بينفّذ الـ message كاملة (loop على الـ transfers، CS management، DMA او PIO). لما يخلص **لازم** يكلم `spi_finalize_current_message()` عشان الـ core يُكمل الـ queue.

```
transfer_one_message(ctlr, msg)
    set CS active
    for each transfer in msg->transfers:
        --> do the actual TX/RX (DMA or PIO)
        --> handle cs_change flag
        --> handle delays
    set CS inactive
    spi_finalize_current_message(ctlr)
```

---

#### `ctlr->transfer_one(struct spi_controller *ctlr, struct spi_device *spi, struct spi_transfer *transfer)`

ابسط من `transfer_one_message` — الـ core بيتكفّل بالـ CS management والـ delays والـ message loop. الـ driver بينفّذ transfer واحدة بس.

**Return values:**
- `0` — Transfer انتهى synchronously.
- `1` — Transfer لسه شغّال (async DMA)، الـ driver هيكلم `spi_finalize_current_transfer()` لاحقاً.
- negative errno — فشل.

**Key detail:** `transfer_one` و`transfer_one_message` **mutually exclusive** — لو الاتنين متضبطين، الـ core يستخدم `transfer_one_message` بس.

---

#### `ctlr->prepare_transfer_hardware()` / `ctlr->unprepare_transfer_hardware()`

```c
int (*prepare_transfer_hardware)(struct spi_controller *ctlr);
int (*unprepare_transfer_hardware)(struct spi_controller *ctlr);
```

الـ `prepare` بيتكلم قبل اول message في الـ queue (مناسب لـ clock enable، power on). الـ `unprepare` بيتكلم لما الـ queue تفضى (مناسب لـ runtime PM put). ممكن يناموا.

---

#### `ctlr->set_cs(struct spi_device *spi, bool enable)`

بيضبط مستوى CS GPIO. الـ core بيكلمه قبل وبعد كل message لو الـ controller مش بيعمل CS management يدوياً. ممكن يتكلم من interrupt context.

---

#### `ctlr->cleanup(struct spi_device *spi)`

بيتكلم عند حذف الـ `spi_device`. لو controller driver بيخزن per-device state في `spi->controller_state`، لازم يحرره هنا.

---

#### `ctlr->can_dma(ctlr, spi, xfer)`

```c
bool (*can_dma)(struct spi_controller *ctlr,
                struct spi_device *spi,
                struct spi_transfer *xfer);
```

لو موجودة وبترجع `true`، الـ core بيعمل DMA mapping للـ transfer buffers قبل ما يكلم `transfer_one()`. الـ driver يلاقي `xfer->tx_dma` و`xfer->rx_dma` جاهزين.

---

### المجموعة التاسعة: Queue Management Internals

---

#### `spi_finalize_current_message()`

```c
void spi_finalize_current_message(struct spi_controller *ctlr);
```

**لازم** يكلمها الـ controller driver بعد ما يخلص من الـ message. بتعمل:
1. تنفّذ الـ `message->complete(context)` callback.
2. تعمل `complete()` على الـ `cur_msg_completion` لو في `spi_sync()` مستني.
3. تُحفّز الـ pump thread لمعالجة الـ message الجاية.

---

#### `spi_finalize_current_transfer()`

```c
void spi_finalize_current_transfer(struct spi_controller *ctlr);
```

نفس الفكرة لكن عند استخدام `transfer_one` — بتخبر الـ core ان الـ transfer الحالية خلصت فيكمل الـ loop على باقي الـ transfers في الـ message.

---

#### `spi_get_next_queued_message()`

```c
struct spi_message *spi_get_next_queued_message(struct spi_controller *ctlr);
```

للـ controller drivers اللي بتدير الـ queue بنفسها (غير شائع). بترجع الـ message الجاية من الـ queue من غير ما تشيله.

---

### المجموعة العاشرة: Helpers & Accessors

---

#### `spi_set_drvdata()` / `spi_get_drvdata()`

```c
static inline void spi_set_drvdata(struct spi_device *spi, void *data);
static inline void *spi_get_drvdata(const struct spi_device *spi);
```

Wrappers لـ `dev_set_drvdata` / `dev_get_drvdata` على `spi->dev`. للـ per-chip private state للـ protocol driver. يتكلموا في `probe()` وبعدين في اي callback.

```c
/* في probe() */
chip = devm_kzalloc(&spi->dev, sizeof(*chip), GFP_KERNEL);
spi_set_drvdata(spi, chip);

/* في اي callback بعدين */
chip = spi_get_drvdata(spi);
```

---

#### `spi_controller_get_devdata()` / `spi_controller_set_devdata()`

```c
static inline void *spi_controller_get_devdata(struct spi_controller *ctlr);
static inline void spi_controller_set_devdata(struct spi_controller *ctlr, void *data);
```

نفس الفكرة لكن للـ controller driver — يوصّل لـ private data المُخصَّص مع الـ `spi_alloc_host()`.

---

#### `spi_max_transfer_size()` / `spi_max_message_size()`

```c
static inline size_t spi_max_transfer_size(struct spi_device *spi);
static inline size_t spi_max_message_size(struct spi_device *spi);
```

بتستدعوا الـ callbacks الاختيارية في الـ `spi_controller`. لو مش موجودة بترجعوا `SIZE_MAX`. الـ `spi_max_transfer_size()` بترجع `min(transfer_max, message_max)` لان الـ transfer limit لازم ما تتعدّى الـ message limit.

---

#### `spi_split_transfers_maxsize()` / `spi_split_transfers_maxwords()`

```c
int spi_split_transfers_maxsize(struct spi_controller *ctlr,
                                 struct spi_message *msg,
                                 size_t maxsize);

int spi_split_transfers_maxwords(struct spi_controller *ctlr,
                                  struct spi_message *msg,
                                  size_t maxwords);
```

لو الـ controller عنده limit على حجم الـ transfer، الـ core (او الـ driver) ممكن يكلم الـ functions دي في `prepare_message()` عشان تقسّم اي transfer كبيرة لـ transfers اصغر. بتستخدموا `spi_res` mechanism عشان تعكس التقسيم بعد انتهاء الـ message.

**Return:** `0` on success, negative errno on failure.

---

#### `spi_controller_xfer_timeout()`

```c
static inline unsigned int spi_controller_xfer_timeout(struct spi_controller *ctlr,
                                                         struct spi_transfer *xfer);
```

بتحسب timeout مناسب بالـ milliseconds للـ transfer بناءً على حجمه وسرعته:

```
timeout = max(len * 8 * 2 / (speed_hz / 1000), 500)
```

الـ factor 2x هامش امان، والـ minimum 500ms عشان avoid false positives على الـ loaded systems.

---

#### `spi_is_bpw_supported()` / `spi_bpw_to_bytes()`

```c
static inline bool spi_is_bpw_supported(struct spi_device *spi, u32 bpw);
static inline u32 spi_bpw_to_bytes(u32 bpw);
```

الاولى بتشيك الـ `bits_per_word_mask` في الـ controller. الثانية بتحوّل bits لـ bytes بـ `roundup_pow_of_two(BITS_TO_BYTES(bpw))` — يعني 12-bit => 2 bytes، 20-bit => 4 bytes.

---

#### `spi_take_timestamp_pre()` / `spi_take_timestamp_post()`

```c
void spi_take_timestamp_pre(struct spi_controller *ctlr,
                             struct spi_transfer *xfer,
                             size_t progress, bool irqs_off);

void spi_take_timestamp_post(struct spi_controller *ctlr,
                              struct spi_transfer *xfer,
                              size_t progress, bool irqs_off);
```

للـ controllers اللي بتدعم PTP timestamping. الـ controller driver بيكلمهم قبل وبعد ارسال الـ word المحدد بـ `ptp_sts_word_pre` و`ptp_sts_word_post` في الـ `spi_transfer`. بتسجّل الـ timestamp في `xfer->ptp_sts`.

---

### مثال متكامل: Protocol Driver Probe + I/O

```c
static int my_sensor_probe(struct spi_device *spi)
{
    struct my_sensor *sensor;

    /* setup: 3.3V board => max 1MHz, SPI mode 1 */
    spi->max_speed_hz  = 1000000;
    spi->mode          = SPI_MODE_1;
    spi->bits_per_word = 8;
    if (spi_setup(spi))
        return -EINVAL;

    sensor = devm_kzalloc(&spi->dev, sizeof(*sensor), GFP_KERNEL);
    if (!sensor)
        return -ENOMEM;

    spi_set_drvdata(spi, sensor);
    return 0;
}

/* قراءة register: بنبعت command byte ونستقبل 2 bytes */
static int read_reg16(struct spi_device *spi, u8 reg, u16 *val)
{
    ssize_t ret = spi_w8r16be(spi, reg);  /* process context */
    if (ret < 0)
        return ret;
    *val = (u16)ret;
    return 0;
}

/* كتابة async مع callback */
static void write_done(void *ctx)
{
    struct completion *done = ctx;
    complete(done);
}

static int write_reg_async(struct spi_device *spi, u8 reg, u8 val)
{
    /* DMA-safe buffer -- heap allocated */
    u8 *buf = kmalloc(2, GFP_KERNEL);
    struct spi_transfer xfer = { .tx_buf = buf, .len = 2 };
    struct spi_message  msg;
    DECLARE_COMPLETION_ONSTACK(done);

    buf[0] = reg;
    buf[1] = val;

    spi_message_init(&msg);
    spi_message_add_tail(&xfer, &msg);
    msg.complete = write_done;
    msg.context  = &done;

    spi_async(spi, &msg);           /* non-blocking */
    wait_for_completion(&done);     /* نستنى من process context */
    kfree(buf);
    return msg.status;
}
```

---

### مثال متكامل: Controller Driver Probe

```c
static int my_spi_probe(struct platform_device *pdev)
{
    struct spi_controller *ctlr;
    struct my_ctlr_data   *cd;

    /* allocate controller + private data */
    ctlr = devm_spi_alloc_host(&pdev->dev, sizeof(*cd));
    if (!ctlr)
        return -ENOMEM;

    cd = spi_controller_get_devdata(ctlr);

    /* fill required fields */
    ctlr->bus_num            = pdev->id;
    ctlr->num_chipselect     = 4;
    ctlr->mode_bits          = SPI_CPOL | SPI_CPHA | SPI_CS_HIGH;
    ctlr->bits_per_word_mask = SPI_BPW_MASK(8) | SPI_BPW_MASK(16);
    ctlr->max_speed_hz       = 50000000; /* 50 MHz */

    /* provide one of these two (mutually exclusive) */
    ctlr->transfer_one                = my_transfer_one;
    ctlr->prepare_transfer_hardware   = my_prepare_hw;
    ctlr->unprepare_transfer_hardware = my_unprepare_hw;

    /* hardware init */
    cd->base = devm_platform_ioremap_resource(pdev, 0);

    return devm_spi_register_controller(&pdev->dev, ctlr);
}

/* ينفّذ transfer واحدة -- الـ core بيتكفّل بالـ CS والـ delays */
static int my_transfer_one(struct spi_controller *ctlr,
                            struct spi_device *spi,
                            struct spi_transfer *xfer)
{
    struct my_ctlr_data *cd = spi_controller_get_devdata(ctlr);

    /* setup hardware registers from xfer->speed_hz, bits_per_word */
    my_hw_set_speed(cd, xfer->speed_hz ?: spi->max_speed_hz);

    /* start DMA -- return 1 means "still in progress" */
    my_hw_start_dma(cd, xfer->tx_dma, xfer->rx_dma, xfer->len);
    return 1;  /* ISR will call spi_finalize_current_transfer() */
}
```
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. الـ debugfs entries الخاصة بـ SPI

الـ SPI subsystem بيكتب statistics في debugfs تحت `/sys/kernel/debug/spi/`.

| المسار | المحتوى |
|--------|---------|
| `/sys/kernel/debug/spi/spi0/` | الـ controller الأول (bus 0) |
| `/sys/kernel/debug/spi/spi0/statistics` | إحصائيات الـ messages والـ transfers والـ errors |
| `/sys/kernel/debug/spi/spi0.0/statistics` | إحصائيات الـ device المتصل بـ CS0 |

```bash
# اقرأ statistics للـ controller
cat /sys/kernel/debug/spi/spi0/statistics

# مثال على الـ output
messages:          1234
transfers:         5678
errors:               0
timedout:             0
spi_sync:          1100
spi_sync_immediate:  900
spi_async:          134
bytes:            45678
bytes_tx:         22839
bytes_rx:         22839
transfer_bytes_histo:0=0,1=10,2=100,...
transfers_split_maxsize: 0
```

الـ `errors` > 0 يعني في مشكلة في الـ hardware أو الـ driver.
الـ `timedout` > 0 يعني الـ device بطيء جداً أو مش بيرد.
الـ `transfers_split_maxsize` > 0 يعني الـ transfers بتتقسم بسبب حد الـ maxsize.

---

#### 2. الـ sysfs entries

**الـ SPI بيظهر في sysfs في أماكن متعددة:**

```bash
# شوف كل الـ SPI controllers
ls /sys/class/spi_master/
# spi0  spi1  spi2

# الـ bus number
cat /sys/class/spi_master/spi0/dev

# devices متصلة بـ bus 0
ls /sys/bus/spi/devices/
# spi0.0  spi0.1  spi1.0

# معلومات device معين
cat /sys/bus/spi/devices/spi0.0/modalias
# spidev  أو اسم الـ driver زي  ads7846

# الـ driver اللي بيشتغل مع الـ device
ls -l /sys/bus/spi/devices/spi0.0/driver
# -> ../../../../bus/spi/drivers/ads7846

# الـ controller mode (للـ SPI target controller)
cat /sys/class/spi_slave/spi0/
```

**الـ statistics لكل device:**

```bash
# قراءة statistics لـ spi0.0
find /sys/bus/spi/devices/spi0.0/ -name "statistics" | xargs cat
```

---

#### 3. الـ ftrace — tracepoints وكيفية تفعيلها

**الـ SPI subsystem عنده tracepoints محددة في `include/trace/events/spi.h`:**

| الـ Tracepoint | الحدث |
|----------------|-------|
| `spi:spi_controller_idle` | الـ controller خلص شغل وبقي idle |
| `spi:spi_controller_busy` | الـ controller بدأ يشتغل |
| `spi:spi_setup` | تم استدعاء `spi_setup()` لـ device |
| `spi:spi_set_cs` | تغيير حالة الـ chip select |
| `spi:spi_message_submit` | تم إرسال message للـ queue |
| `spi:spi_message_start` | بدأ تنفيذ message |
| `spi:spi_message_done` | انتهى تنفيذ message (يُظهر actual/frame length) |
| `spi:spi_transfer_start` | بدأ transfer واحد |
| `spi:spi_transfer_stop` | انتهى transfer واحد (يُظهر tx/rx data) |

```bash
# تفعيل كل tracepoints الـ SPI
echo 1 > /sys/kernel/debug/tracing/events/spi/enable

# أو tracepoint محدد بس
echo 1 > /sys/kernel/debug/tracing/events/spi/spi_message_done/enable
echo 1 > /sys/kernel/debug/tracing/events/spi/spi_transfer_start/enable
echo 1 > /sys/kernel/debug/tracing/events/spi/spi_transfer_stop/enable

# ابدأ الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل الـ application أو افعل الـ operation
# ...

# اقرأ الـ trace buffer
cat /sys/kernel/debug/tracing/trace

# وقف الـ tracing
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

**مثال على output الـ `spi_message_done`:**

```
spi0.0: spi_message_done: spi0.0 0xffff8880... len=8/8
#      bus.cs  event          pointer   actual/frame
```

**الـ `spi_transfer_stop` يطبع أول 64 byte من الـ tx/rx data** — مفيد جداً لتتبع بيانات الـ protocol.

```bash
# filter على bus/device معين
echo 'bus_num == 0 && chip_select == 0' > \
  /sys/kernel/debug/tracing/events/spi/spi_message_done/filter
```

---

#### 4. الـ printk والـ dynamic debug

**تفعيل الـ dynamic debug للـ SPI core:**

```bash
# تفعيل كل debug messages في SPI subsystem
echo "module spi_core +p" > /sys/kernel/debug/dynamic_debug/control
echo "module spi +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug في controller driver معين (مثلاً spi-bcm2835)
echo "module spi_bcm2835 +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug في ملف معين
echo "file drivers/spi/spi.c +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل مع line numbers وأسماء functions
echo "file drivers/spi/spi.c +pflm" > /sys/kernel/debug/dynamic_debug/control

# شوف كل الـ debug points المفعلة
cat /sys/kernel/debug/dynamic_debug/control | grep spi
```

**الـ printk levels المهمة:**
- `KERN_ERR` — فشل الـ transfer أو setup
- `KERN_WARNING` — timeout أو retry
- `KERN_DEBUG` — تفاصيل الـ messages (تحتاج dynamic debug)

```bash
# اتابع الـ kernel log مباشرة
dmesg -w | grep -i spi

# أو
journalctl -k -f | grep -i spi
```

---

#### 5. الـ Kernel Config options للـ Debugging

| الـ Config | الوصف |
|-----------|-------|
| `CONFIG_SPI_DEBUG` | تفعيل debug messages في SPI core |
| `CONFIG_DEBUG_SPI_STUB` | stub controller للـ testing بدون hardware |
| `CONFIG_SPI_SPIDEV` | userspace access عبر `/dev/spidevB.C` للـ testing |
| `CONFIG_SPI_LOOPBACK_TEST` | loopback test driver |
| `CONFIG_SPI_TEST` | unit tests للـ SPI subsystem |
| `CONFIG_DEBUG_KERNEL` | يفعّل checks عامة في الـ kernel |
| `CONFIG_DYNAMIC_DEBUG` | يسمح بـ dynamic debug activation |
| `CONFIG_TRACING` | أساس الـ ftrace support |
| `CONFIG_SPI_SLAVE` | دعم SPI target mode للـ testing |

```bash
# تحقق إذا الـ config مفعل
grep CONFIG_SPI_DEBUG /boot/config-$(uname -r)
zcat /proc/config.gz | grep -E "CONFIG_SPI|CONFIG_DEBUG_SPI"
```

---

#### 6. الأدوات الخاصة بالـ SPI

**الـ spidev — الأداة الأساسية للـ userspace testing:**

```bash
# تثبيت spi-tools
apt install spi-tools   # Debian/Ubuntu
# أو build من المصدر: https://github.com/msperl/spi-tools

# اقرأ معلومات الـ device
spi-config -d /dev/spidev0.0 -q

# اعمل loopback test (MOSI موصول بـ MISO)
spi-pipe -d /dev/spidev0.0 -s 1000000 < /dev/urandom | head -c 16 | xxd
```

**الـ spidev_test من kernel tree:**

```bash
# موجود في tools/spi/ في kernel source
cd linux/tools/spi/
make
./spidev_test -D /dev/spidev0.0 -v -p "hello"
# -D  device
# -v  verbose (يطبع tx/rx data)
# -p  pattern للـ transmit
```

**مثال على output:**

```
spi mode: 0x0
bits per word: 8
max speed: 500000 Hz (500 kHz)
TX | 68 65 6C 6C 6F __ __ __ __ __ __ __ __ __ __ __ | hello
RX | 68 65 6C 6C 6F __ __ __ __ __ __ __ __ __ __ __ | hello
```

لو الـ TX == الـ RX، الـ loopback شغال وبالتالي الـ hardware سليم.

**فحص الـ SPI devices المسجلة:**

```bash
# كل devices على الـ SPI bus
ls -la /sys/bus/spi/devices/

# الـ driver المرتبط
ls -la /sys/bus/spi/devices/spi0.0/driver

# unregister/register SPI target (في target mode)
echo "spidev" > /sys/class/spi_slave/spi0/slave   # register
echo "(null)" > /sys/class/spi_slave/spi0/slave   # unregister
```

---

#### 7. جدول رسائل الـ Error الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---------------------|--------|------|
| `spi_master spi0: failed to transfer one message from queue` | فشل تنفيذ message | تحقق من الـ clock speed و الـ mode |
| `spi0.0: timeout waiting for completion` | الـ device لم يستجب في الوقت | قلل `max_speed_hz`، تحقق من الـ wiring |
| `spi0.0: rejecting DMA map of vmalloc memory` | buffer مش DMA-safe | استخدم `kmalloc` بدل stack أو vmalloc |
| `spi_alloc_host: no memory for spi_controller` | نفد الـ memory | تحقق من الـ heap |
| `spi0.0: chip select 0 already in use` | تعارض على نفس الـ CS | تحقق من تكوين الـ Device Tree |
| `SPI controller driver 'X' has no transfer_one()` | الـ controller driver ناقص | الـ driver يحتاج تطوير |
| `spi0: transfer timeout` | تجاوز الـ timeout في الـ transfer | تحقق من الـ IRQ أو الـ DMA config |
| `spi0.0: unsupported mode bits X` | mode غير مدعوم من الـ controller | تحقق من `mode_bits` في الـ controller |
| `probe of spi0.0 failed with error -22` | `probe()` رجع `-EINVAL` | بيانات الـ platform_data أو DT غلط |
| `spi0: GPIO chip select 0 not found` | الـ GPIO للـ CS مش موجود | تحقق من DT: `cs-gpios` property |

---

#### 8. أماكن إضافة `dump_stack()` و `WARN_ON()`

```c
/* في controller driver — تحقق من الـ buffer alignment */
static int myctlr_transfer_one(struct spi_controller *ctlr,
                                struct spi_device *spi,
                                struct spi_transfer *xfer)
{
    /* WARN لو الـ buffer مش aligned لـ DMA */
    WARN_ON(!IS_ALIGNED((unsigned long)xfer->tx_buf, 4));
    WARN_ON(!IS_ALIGNED((unsigned long)xfer->rx_buf, 4));

    /* WARN لو الطول صفر */
    WARN_ON(xfer->len == 0);

    /* WARN لو السرعة أكبر من الـ max */
    WARN_ON(xfer->speed_hz > ctlr->max_speed_hz);
    ...
}

/* في protocol driver — تحقق من state */
static int chip_send_command(struct spi_device *spi, u8 cmd)
{
    /* dump_stack لتتبع من أين جاءت الـ call */
    if (unlikely(chip->suspended)) {
        dump_stack();
        return -ENODEV;
    }

    /* WARN_ON_ONCE لو تم الاستدعاء من context غلط */
    WARN_ON_ONCE(in_interrupt() && !spi->controller->can_dma);
    ...
}
```

**النقاط الاستراتيجية:**
- في `probe()` بعد `spi_setup()` — تحقق من الـ mode الراجع
- في `transfer_one_message()` عند timeout — قبل ما ترجع الـ error
- عند فشل الـ DMA mapping — لفهم من أين جاء الـ buffer
- في `cleanup()` — للتأكد إن الـ controller_state اتحر بشكل صحيح

---

### Hardware Level

#### 1. التحقق إن حالة الـ Hardware مطابقة لحالة الـ Kernel

```bash
# شوف الـ mode المبرمج للـ device
cat /sys/bus/spi/devices/spi0.0/modalias

# شوف الـ spi_statistics لفهم حجم الـ traffic الفعلي
cat /sys/kernel/debug/spi/spi0/statistics

# قارن الـ max_speed_hz في DT مع القيمة الفعلية
# (بعض الـ controllers بيطنشوا القيمة ويحطوا أقل منها)
# فعّل spi_setup tracepoint وشوف الـ Hz الفعلية
echo 1 > /sys/kernel/debug/tracing/events/spi/spi_setup/enable
# ثم أعد تهيئة الـ device وشوف الـ trace
cat /sys/kernel/debug/tracing/trace | grep spi_setup
# output:
# spi0.0 setup mode 0, 8 bits/w, 1000000 Hz max --> 0
```

---

#### 2. الـ Register Dump Techniques

**باستخدام `devmem2` أو `/dev/mem`:**

```bash
# تثبيت devmem2
apt install devmem2

# اقرأ register بـ physical address (مثال: BCM2835 SPI0 base = 0x3F204000)
devmem2 0x3F204000 w   # اقرأ Control register
devmem2 0x3F204004 w   # اقرأ FIFO register
devmem2 0x3F204008 w   # اقرأ Clock Divider
devmem2 0x3F20400C w   # اقرأ Data Length

# أو باستخدام /dev/mem مباشرة
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    mem = mmap.mmap(f.fileno(), 0x1000, offset=0x3F204000)
    cs_reg = struct.unpack('<I', mem[0:4])[0]
    print(f'CS register: 0x{cs_reg:08X}')
    # bit 0: CS active low
    # bit 7: TA (Transfer Active)
    # bit 8: DMAEN
    mem.close()
"
```

**باستخدام `io` tool (من busybox أو standalone):**

```bash
# اقرأ 16 registers من base address
io -4 -r 0x3F204000 16
```

**ملاحظة:** تحتاج `CONFIG_STRICT_DEVMEM=n` أو `iomem=relaxed` في الـ boot params.

---

#### 3. نصائح Logic Analyzer وـ Oscilloscope

**الإعدادات الأساسية للـ Logic Analyzer:**

| الإشارة | الوصف | ما تراقبه |
|---------|-------|----------|
| **SCK** | الـ clock line | التردد، الـ CPOL، عدد الـ pulses |
| **MOSI** | Master Out Slave In | البيانات المُرسلة |
| **MISO** | Master In Slave Out | البيانات المُستقبلة |
| **nCS** | Chip Select | التوقيت، الـ active level |

```
الـ SPI Timing Diagram الصحيح (Mode 0):

nCS  ‾‾‾\__________________________/‾‾‾
          ^                        ^
          CS setup time            CS hold time

SCK  ____/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\____
         ^                              ^
         first edge                     last edge

MOSI ____X====X====X====X====X====X====____
         ^ data valid on rising edge

MISO ____X====X====X====X====X====X====____
```

**نقاط المراقبة:**
- **setup time بعد CS assertion**: لازم يكون ≥ الـ `cs_setup` delay المبرمج في الـ DT
- **hold time قبل CS deassertion**: لازم يكون ≥ الـ `cs_hold` delay
- **inter-transfer delay**: الـ `cs_change` behavior بين الـ transfers
- **idle state للـ MOSI**: لو بتستخدم `SPI_MOSI_IDLE_HIGH`، تحقق إنه high لما CS inactive

**للـ Oscilloscope:**
- اضبط الـ trigger على الـ falling edge للـ nCS
- فعّل الـ protocol decode لو موجود (SPI decode)
- قيس الـ eye diagram للـ MOSI/MISO عند high speeds للتحقق من الـ signal integrity

---

#### 4. Common Hardware Issues وأنماطها في الـ Kernel Log

| المشكلة الـ Hardware | الـ Pattern في الـ Log | الحل |
|---------------------|----------------------|------|
| الـ clock polarity غلط (CPOL) | data corruption، قيم عشوائية في RX | غيّر `SPI_MODE_X` في DT أو الـ driver |
| الـ CS active-high والـ kernel يعتقد active-low | الـ device لا يستجب | أضف `SPI_CS_HIGH` في الـ mode flags |
| wire مفكوك (MISO) | `bytes_rx` صفر دايماً رغم الـ transfers | فحص بالـ multimeter، continuity test |
| الـ pull-up/pull-down غلط على MISO | garbage في الـ RX عند ما تتكلم الـ device | أضف 4.7kΩ pull-up على MISO |
| السرعة عالية جداً (signal integrity) | errors متقطعة، `errors++` في statistics | قلل `max-speed-hz` في DT بالتدريج |
| الـ power supply غير مستقر | timeout أو resets عشوائية | قس الـ VCC بالـ oscilloscope أثناء الـ transfer |
| مشاركة الـ bus بين devices بـ CS مختلف | device يرد على CS مش بتاعه | تحقق من الـ SPI_CS_HIGH/LOW لكل device |

---

#### 5. الـ Device Tree Debugging

**تحقق إن الـ DT مطابق للـ hardware:**

```bash
# اعرض الـ DT الحالي كـ readable format
dtc -I fs -O dts /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "spi@"

# مثال على DT صحيح لـ SPI controller:
# spi0: spi@3f204000 {
#     compatible = "brcm,bcm2835-spi";
#     reg = <0x3f204000 0x1000>;
#     interrupts = <2 22>;
#     clocks = <&clocks BCM2835_CORE_CLK>;
#     #address-cells = <1>;
#     #size-cells = <0>;
#     cs-gpios = <&gpio 8 GPIO_ACTIVE_LOW>;
#
#     flash@0 {
#         compatible = "jedec,spi-nor";
#         reg = <0>;                    /* chip select 0 */
#         spi-max-frequency = <1000000>;
#         spi-cpol;                     /* CPOL=1 */
#         spi-cpha;                     /* CPHA=1 → Mode 3 */
#     };
# };
```

```bash
# تحقق من الـ DT properties للـ SPI device
cat /sys/bus/spi/devices/spi0.0/of_node/spi-max-frequency
cat /sys/bus/spi/devices/spi0.0/of_node/compatible

# تحقق من الـ overlays المطبقة
ls /sys/firmware/devicetree/base/chosen/

# شوف الـ DT source بعد تطبيق الـ overlays
dtc -I fs -O dts /sys/firmware/devicetree/base 2>/dev/null > /tmp/current_dt.dts
grep -B2 -A30 "spi" /tmp/current_dt.dts
```

**الأخطاء الشائعة في DT:**

| الخطأ | الأعراض | الحل |
|-------|---------|------|
| `reg = <1>` بدل `<0>` | Device على CS1 بدل CS0 | صحح الـ reg value |
| `spi-max-frequency` مرتفع جداً | signal corruption | قللها لـ 1MHz للـ testing |
| ناسي `spi-cpol` أو `spi-cpha` | data غلط | راجع datasheet الـ chip |
| `cs-gpios` غلط | CS لا يتحكم فيه الـ kernel | صحح الـ GPIO number والـ polarity |
| `compatible` غلط | الـ driver مش بيـ probe | طابق الـ compatible مع الـ driver |

```bash
# تحقق إذا الـ driver probe نجح
dmesg | grep -E "spi[0-9]\.[0-9]|spi_master"
# ناجح:    spi0.0: ads7846 on spi0.0
# فاشل:   spi0.0: probe with driver ads7846 failed with error -22
```

---

### Practical Commands

#### 1. أوامر جاهزة للنسخ والتشغيل

**فحص سريع للـ SPI subsystem:**

```bash
#!/bin/bash
# spi_debug_quick.sh — فحص سريع لحالة الـ SPI

echo "=== SPI Controllers ==="
ls /sys/class/spi_master/ 2>/dev/null || echo "No SPI controllers found"

echo ""
echo "=== SPI Devices ==="
ls /sys/bus/spi/devices/ 2>/dev/null || echo "No SPI devices found"

echo ""
echo "=== SPI Statistics (all controllers) ==="
for ctrl in /sys/kernel/debug/spi/*/statistics; do
    echo "--- $ctrl ---"
    cat "$ctrl" 2>/dev/null
done

echo ""
echo "=== Recent SPI Kernel Messages ==="
dmesg | grep -i spi | tail -20
```

**تفعيل الـ full SPI tracing:**

```bash
#!/bin/bash
# spi_trace_start.sh

TRACE=/sys/kernel/debug/tracing

# تنظيف الـ buffer
echo "" > $TRACE/trace

# تفعيل كل SPI events
echo 1 > $TRACE/events/spi/enable

# زود الـ buffer size لـ 64MB
echo 65536 > $TRACE/buffer_size_kb

# ابدأ الـ tracing
echo 1 > $TRACE/tracing_on

echo "Tracing started. Run: echo 0 > $TRACE/tracing_on  to stop"
echo "Then read: cat $TRACE/trace | grep spi"
```

**تفعيل الـ dynamic debug للـ SPI:**

```bash
# فعّل كل debug في SPI
echo "module spi_core +pflm" > /sys/kernel/debug/dynamic_debug/control 2>/dev/null
echo "file drivers/spi/spi.c +pflm" > /sys/kernel/debug/dynamic_debug/control 2>/dev/null

# فعّل لـ controller driver محدد
echo "module spi_bcm2835 +pflm" > /sys/kernel/debug/dynamic_debug/control 2>/dev/null
echo "module spi_imx +pflm" > /sys/kernel/debug/dynamic_debug/control 2>/dev/null

# شوف الـ debug messages
dmesg -w
```

**اختبار الـ spidev بالـ loopback:**

```bash
# تحقق من وجود الـ spidev device
ls /dev/spidev*
# /dev/spidev0.0  /dev/spidev0.1

# اختبار بسيط بـ Python
python3 << 'EOF'
import spidev

spi = spidev.SpiDev()
spi.open(0, 0)          # bus=0, device=0
spi.max_speed_hz = 500000
spi.mode = 0

# إرسال واستقبال (loopback: MOSI موصول بـ MISO)
tx = [0xAA, 0x55, 0x0F, 0xF0]
rx = spi.xfer2(tx)

print(f"TX: {[hex(b) for b in tx]}")
print(f"RX: {[hex(b) for b in rx]}")

if tx == rx:
    print("LOOPBACK OK: hardware is working")
else:
    print("LOOPBACK FAILED: check wiring")

spi.close()
EOF
```

**اختبار بـ `spidev_test` من الـ kernel tools:**

```bash
# build
cd /usr/src/linux/tools/spi/
make

# اختبار verbose
./spidev_test -D /dev/spidev0.0 -v -s 500000 -p "\xAA\x55\x0F\xF0"

# اختبار بـ loop (خيار -l) — يُكرر الـ test عدة مرات
./spidev_test -D /dev/spidev0.0 -v -s 1000000 -l -p "\xDE\xAD\xBE\xEF"
```

**مراقبة الـ statistics في real-time:**

```bash
#!/bin/bash
# spi_stats_watch.sh — مراقبة statistics كل ثانية

DEVICE=${1:-spi0}
STATS_FILE="/sys/kernel/debug/spi/$DEVICE/statistics"

if [ ! -f "$STATS_FILE" ]; then
    echo "Error: $STATS_FILE not found"
    echo "Available: $(ls /sys/kernel/debug/spi/ 2>/dev/null)"
    exit 1
fi

while true; do
    clear
    echo "=== SPI Statistics: $DEVICE ($(date)) ==="
    cat "$STATS_FILE"
    sleep 1
done
```

---

#### 2. قراءة الـ Output وتفسيره

**تفسير output الـ `spi_message_done` trace event:**

```
<task>-<pid> [cpu] .... <timestamp>: spi_message_done: spi0.0 0xffff888003a12000 len=16/16

spi0       → bus number 0
.0         → chip select 0
0xffff...  → pointer لـ spi_message struct في الـ kernel memory
len=16/16  → actual_length=16, frame_length=16
            لو actual < frame → الـ transfer اتوقف قبل ما يكمل!
```

**تفسير output الـ `spi_transfer_stop`:**

```
spi_transfer_stop: spi0.0 0xffff... len=4 tx=[aa 55 0f f0] rx=[aa 55 0f f0]
                                          ^^^                  ^^^
                                          ما أرسلناه           ما استقبلناه
```
لو `rx` مش مطابق لـ `tx` في الـ loopback test → مشكلة في الـ hardware.

**تفسير statistics:**

```
errors:     5     ← خمس transfers فشلوا، شوف dmesg للتفاصيل
timedout:   2     ← اتنين timeout، الـ device بطيء أو مش بيرد
spi_async: 100    ← معظم الـ transfers async (طبيعي في الـ production)
spi_sync:   50    ← نص الـ transfers sync (من context بيقدر ينام)
transfers_split_maxsize: 10  ← ١٠ transfers اتقسموا بسبب الـ max size في الـ controller
                               → ممكن يأثر على الـ performance
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — بيانات SPI sensor بتيجي غلط

#### العنوان
**قراءات ADC عبر SPI بتدي قيم عشوائية على industrial gateway**

#### السياق
شركة بتبني gateway صناعي على **RK3562** لمراقبة ضغط الغاز في خطوط الأنابيب. الـ sensor هو **ADS1247** (ADC 24-bit من Texas Instruments) متوصل على **SPI1**. الـ device tree اتعمله صح، والـ driver بيشتغل، بس القراءات مش ثابتة وأحيانًا بتدي overflow.

#### المشكلة
الـ ADS1247 بيشتغل على **SPI Mode 1** (CPOL=0, CPHA=1) بس الـ developer كتب في الـ DT:

```c
/* Wrong: mode 0 instead of mode 1 */
ads1247@0 {
    compatible = "ti,ads1247";
    reg = <0>;
    spi-max-frequency = <1000000>;
    /* missing: spi-cpha */
};
```

الـ spi-summary يوضح إن CPOL و CPHA بيحددوا إمتى البيانات بتتسامبل. لما CPHA=0 (mode 0) تفترض إن البيانات مستقرة على الـ leading edge، بس الـ ADS1247 بيحط البيانات على الـ trailing edge (CPHA=1)، فالـ controller بياخد sample في الوقت الغلط تمامًا.

#### التحليل
الـ spi-summary بيشرح:

> *CPHA=0 says sample on the leading edge, CPHA=1 means the trailing edge.*

الـ `struct spi_device` بيحمل الـ `mode` field:

```c
struct spi_device {
    u32 max_speed_hz;
    u32 mode;   /* SPI_MODE_0 .. SPI_MODE_3 */
    u8  bits_per_word;
    /* ... */
};
```

لما الـ `spi_setup()` بتتكلى في `probe()`، بتبعت الـ mode للـ controller driver اللي بيضبط الـ hardware registers. لو الـ DT مفيهوش `spi-cpha`، الـ mode بيبقى **SPI_MODE_0** وده بيخلي الـ RK3562 SPI controller يسامبل على الـ rising edge بدل الـ falling edge.

#### الحل

**تصحيح الـ Device Tree:**

```dts
&spi1 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&spi1m0_pins>;

    ads1247: adc@0 {
        compatible = "ti,ads1247";
        reg = <0>;
        spi-max-frequency = <1000000>;
        spi-cpha;   /* CPHA=1 → Mode 1 */
        /* No spi-cpol → CPOL=0 */
    };
};
```

**تحقق من الـ mode بعد البوت:**

```bash
# تأكد إن الـ spi_device أخد الـ mode الصح
cat /sys/bus/spi/devices/spi1.0/modalias

# استخدم scope أو logic analyzer على SCK و MOSI وشوف
# البيانات بتتغير قبل أو بعد الـ rising edge
```

**في الـ driver نفسه لو بيعمل override:**

```c
static int ads1247_probe(struct spi_device *spi)
{
    /* Force correct mode regardless of DT */
    spi->mode = SPI_MODE_1;
    spi->max_speed_hz = 1000000;
    spi->bits_per_word = 8;

    return spi_setup(spi);  /* apply to hardware */
}
```

#### الدرس المستفاد
**الـ CPOL/CPHA غلطة مش بتدي error، بتدي بيانات غلط بهدوء.** دايمًا راجع datasheet الـ sensor وقارن الـ timing diagram بالـ mode اللي في الـ DT. الـ spi-summary بيوضح إن كتير chips بتـsupport كل من Mode 0 و Mode 3 لأنها مش بتهتم بالـ polarity — لكن chips زي ADCs الدقيقة لازم تاخد الـ mode الصح بالظبط.

---

### السيناريو 2: Android TV Box على Allwinner H616 — SPI Flash مش بتـboot

#### العنوان
**SPI NOR Flash على H616 بتتعرف بس الـ rootfs مش بيتـmount**

#### السياق
منتج Android TV box رخيص بيستخدم **Allwinner H616** مع **W25Q128** (SPI NOR Flash 128Mb من Winbond) عشان يخزن الـ bootloader والـ kernel. البيتـbuild بشكل صح، بس عند الـ bring-up البيت يقف عند mounting الـ rootfs.

#### المشكلة
الـ developer خلى `max_speed_hz = 100000000` (100 MHz) في الـ DT بينما الـ W25Q128 الموجود على الـ board ده version قديم بيدعم **50 MHz** maximum. فوق كده، الـ board traces طويلة شوية بتزيد الـ capacitance.

الـ spi-summary بيوضح:

> *The board_info should provide enough information to let the system work without the chip's driver being loaded.*

يعني الـ `max_speed_hz` في الـ `spi_board_info` (أو الـ DT) هو الـ upper limit اللي بيحترمه الـ controller. لو الـ flash بيتكلم بسرعة أعلى من طاقته، الـ bit errors بتظهر بشكل عشوائي في الـ MTD reads.

#### التحليل
الـ `struct spi_device.max_speed_hz` بيتحدد من الـ DT، وبعدين الـ SPI core بيمرره لـ `ctlr->setup()`:

```c
/* spi-summary: "ctlr->setup(struct spi_device *spi)
   This sets up the device clock rate, SPI mode, and word sizes." */

/* In spi.h: */
struct spi_device {
    u32 max_speed_hz;  /* controller must not exceed this */
    /* ... */
};
```

الـ `spi_transfer.speed_hz` ممكن يـoverride الـ device speed لكل transfer لو احتاجت. لو مفيش override، الـ controller بيستخدم الـ `spi_device.max_speed_hz`. على الـ H616 الـ SPI controller بيحاول يشغل الـ flash على 100 MHz، فالـ W25Q128 القديم بيفشل في قراءة بعض الـ pages.

#### الحل

**تصحيح سرعة الـ DT:**

```dts
&spi0 {
    status = "okay";

    w25q128: flash@0 {
        compatible = "winbond,w25q128", "jedec,spi-nor";
        reg = <0>;
        /* Safe speed for old W25Q128 + long traces */
        spi-max-frequency = <40000000>;
        spi-tx-bus-width = <1>;
        spi-rx-bus-width = <1>;
    };
};
```

**تشخيص من الـ kernel log:**

```bash
dmesg | grep -i spi
dmesg | grep -i "w25q\|nor\|mtd"

# لو في bit errors:
# [    2.345678] spi-nor: unrecognized JEDEC id bytes: ff ff ff
# ده معناه الـ flash مش بترد أصلًا → speed أو mode غلط
```

**التحقق بعد التصحيح:**

```bash
cat /proc/mtd
# لازم تشوف mtd0, mtd1... بأحجامها الصح

# اختبار read:
dd if=/dev/mtd0 of=/dev/null bs=4096 count=100
# لو ماشي بدون errors → السرعة صح
```

#### الدرس المستفاد
**الـ `max_speed_hz` هو ceiling مش target.** خليه conservative خصوصًا في production boards مع traces طويلة أو temperature range واسع. الـ spi-summary بينبه إن الـ board_info بيحدد القيود الـboard-specific زي `max sample rate at 3V` — ده مش بس للـ ADCs، ده ينطبق على أي chip بيتأثر speed بالـ voltage أو temperature.

---

### السيناريو 3: IoT Sensor على STM32MP1 — Deadlock في الـ SPI driver

#### العنوان
**الـ SPI transaction بتـhang لمدة 30 ثانية ثم timeout على STM32MP1**

#### السياق
IoT sensor node بيستخدم **STM32MP1** يقرأ من **BME680** (sensor حرارة/رطوبة/ضغط/gas عبر SPI) كل ثانية. في الـ production كل فترة الـ system بيـhang لمدة ~30 ثانية ثم يكمل.

#### المشكلة
الـ driver بيستخدم `spi_sync()` من داخل interrupt context (في IRQ handler لـ GPIO interrupt جاي من sensor آخر على نفس الـ board).

الـ spi-summary بيوضح بوضوح:

> *There are also synchronous wrappers like spi_sync()... These may be issued only in contexts that may sleep.*

الـ `spi_sync()` بتعمل `wait_for_completion()` داخليًا — ده blocking call مش مسموح في IRQ context. الـ kernel مش بيـcrash فورًا لأن الـ IRQ handler بيكمل، بس الـ SPI message queue بتتعطل لأن الـ completion event مش بيتحقق بشكل صح.

#### التحليل
الـ flow الغلط:

```
GPIO IRQ fires
  └─> irq_handler()
        └─> spi_sync(spi, &msg)       ← WRONG: may sleep!
              └─> wait_for_completion()  ← blocks IRQ context
                    └─> SPI transfer completes
                          └─> completion callback tries to wake task
                                └─> DEADLOCK or scheduler warning
```

الـ spi-summary بيقول:

> *The basic I/O primitive is spi_async(). Async requests may be issued in any context (irq handler, task, etc) and completion is reported using a callback provided with the message.*

يعني الحل الصح هو `spi_async()` مع callback.

#### الحل

**تحويل الـ driver لـ async:**

```c
struct bme680_data {
    struct spi_device *spi;
    struct spi_message msg;
    struct spi_transfer xfer;
    u8 tx_buf[2];
    u8 rx_buf[4];
    struct work_struct process_work;
};

/* Completion callback — called in any context */
static void bme680_spi_complete(void *context)
{
    struct bme680_data *data = context;
    /* Schedule work to process result in process context */
    schedule_work(&data->process_work);
}

/* Called from IRQ handler — safe because spi_async() doesn't sleep */
static irqreturn_t sensor_irq_handler(int irq, void *dev_id)
{
    struct bme680_data *data = dev_id;

    spi_message_init(&data->msg);

    data->xfer.tx_buf = data->tx_buf;
    data->xfer.rx_buf = data->rx_buf;
    data->xfer.len    = 4;

    data->msg.complete = bme680_spi_complete;
    data->msg.context  = data;

    spi_message_add_tail(&data->xfer, &data->msg);

    /* Non-blocking — safe in IRQ context */
    spi_async(data->spi, &data->msg);

    return IRQ_HANDLED;
}
```

**التحقق إن المشكلة اختفت:**

```bash
# قبل الحل:
dmesg | grep "BUG: sleeping function called from invalid context"

# بعد الحل: لازم الـ log ده يختفي
# وكمان:
cat /sys/bus/spi/devices/spi0.0/../statistics/errors
# المفروض يبقى 0
```

#### الدرس المستفاد
**`spi_sync()` = blocking، `spi_async()` = non-blocking.** الـ spi-summary بيفرق بينهم بوضوح. أي code بيشتغل في IRQ context أو atomic context لازم يستخدم `spi_async()` مع completion callback، وبعدين يعالج البيانات في workqueue أو tasklet.

---

### السيناريو 4: Custom Board Bring-up على i.MX8 — الـ SPI Device مش بيظهر في sysfs

#### العنوان
**`/dev/spidev0.0` مش موجود و `spi0.0` مش في `/sys/bus/spi/devices/`**

#### السياق
فريق bring-up بيشتغل على **i.MX8M Plus** custom board. الـ design يحتوي على **MCP2515** (CAN controller عبر SPI) متوصل على **ECSPI3**. بعد البوت الـ spidev والـ mcp251x driver مش بيشتغلوا.

#### المشكلة
الـ engineer كتب الـ DT node للـ MCP2515 تحت الـ SPI controller بس الـ status الـ controller نفسه `disabled`.

إضافة لكده، الـ `bus_num` في الـ spi_board_info (لو كان بيستخدم static registration) أو الـ SPI controller alias في الـ DT مش متطابق مع الـ hardware.

الـ spi-summary بيشرح:

> *Bus numbering is important, since that's how Linux identifies a given SPI bus... On SOC systems, the bus numbers should match the numbers defined by the chip manufacturer.*

#### التحليل
على الـ i.MX8M Plus، ECSPI3 هو SPI bus رقم 2 (zero-indexed). لو الـ DT aliases مش صح:

```dts
/* WRONG — missing or wrong alias */
aliases {
    /* ecspi3 not aliased → gets dynamic bus number */
};
```

الـ kernel بيدي bus number dynamic، فـ`spi2.0` ممكن يتحول لـ`spi5.0` حسب ترتيب الـ registration. وده بيكسر أي userspace script بيعتمد على اسم ثابت.

كمان لو الـ controller نفسه `status = "disabled"` في الـ base DTS ومفيش override في الـ board-level DTSI، الـ `spi_register_controller()` مش هيتكلى أصلًا.

#### الحل

**تصحيح الـ Device Tree:**

```dts
/* In board-level .dts */

/* Step 1: alias لـ consistent bus numbering */
aliases {
    spi2 = &ecspi3;
};

/* Step 2: enable the controller */
&ecspi3 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_ecspi3>;
    cs-gpios = <&gpio5 25 GPIO_ACTIVE_LOW>;

    /* Step 3: declare the target device */
    can0: mcp2515@0 {
        compatible = "microchip,mcp2515";
        reg = <0>;
        spi-max-frequency = <10000000>;
        clocks = <&can_osc 0>;
        interrupt-parent = <&gpio1>;
        interrupts = <9 IRQ_TYPE_EDGE_FALLING>;
    };
};
```

**التحقق بعد تطبيق الـ DT:**

```bash
# تأكد إن الـ controller اتسجل
dmesg | grep "spi.*ecspi3\|ecspi3.*spi"

# تأكد إن الـ device ظهر
ls /sys/bus/spi/devices/
# المفروض تشوف: spi2.0 → symlink لـ .../ecspi3/spi2.0

# تأكد إن الـ driver اتربط
cat /sys/bus/spi/devices/spi2.0/modalias
# المفروض: mcp2515

dmesg | grep mcp251x
# المفروض: mcp251x spi2.0 can0: MCP2515 successfully initialized.
```

**فحص الـ sysfs hierarchy زي ما بيشرح spi-summary:**

```bash
# من الـ spi-summary:
# /sys/devices/.../CTLR/spiB.C ... spi_device on bus "B", chipselect C
find /sys/devices -name "spi2.0" -type d
ls /sys/class/spi_master/spi2/
```

#### الدرس المستفاد
**الـ SPI controller لازم يكون `status = "okay"` قبل ما تحط تحته أي device.** وكمان الـ aliases بتحدد الـ bus number الثابت — من غيرها الـ number بيتغير بين builds وبيكسر الـ userspace. الـ spi-summary بيوضح إن الـ bus numbers لازم تتطابق مع chip manufacturer numbering على الـ SoC.

---

### السيناريو 5: Automotive ECU على AM62x — CS Timing مشكلة مع SPI EEPROM

#### العنوان
**كتابة الـ SPI EEPROM على AM62x بتفشل بشكل متقطع في الـ automotive ECU**

#### السياق
**Automotive ECU** بيستخدم **TI AM62x** مع **AT25256B** (SPI EEPROM 256Kb من Microchip) لتخزين calibration data. في اختبارات الـ functional safety ظهر إن ~0.1% من عمليات الكتابة بتفشل، وده فوق الـ threshold المسموح في ASIL-B.

#### المشكلة
الـ AT25256B بيحتاج **tCS ≥ 250ns** بعد ما الـ CS ينزل قبل ما الـ clock يبدأ (CS setup time). الـ AM62x SPI controller بيشتغل بسرعة وبيبدأ الـ clock فورًا. في درجات حرارة عالية (85°C) الـ EEPROM بيتأخر أكثر وتبدأ الـ failures تزيد.

الـ spi-summary بيذكر:

> *optionally defining short delays after transfers ... using the spi_transfer.delay.value setting*

وكمان بيذكر `ctrl->set_cs_timing()`:

> *This method allows SPI client drivers to request SPI host controller for configuring device specific CS setup, hold and inactive timing requirements.*

#### التحليل
الـ `struct spi_device` بتحتوي على CS timing fields:

```c
struct spi_device {
    /* CS delays */
    struct spi_delay    cs_setup;    /* delay after CS asserted */
    struct spi_delay    cs_hold;     /* delay before CS deasserted */
    struct spi_delay    cs_inactive; /* delay after CS deasserted */
    /* ... */
};
```

لو الـ DT أو الـ driver مش بيحدد الـ `cs_setup`، الـ controller بيبدأ الـ clock فورًا بعد CS assertion. على الـ AM62x بـ 50 MHz SPI clock، كل cycle = 20ns — يعني الـ controller ممكن يبدأ يرسل data بعد CS بـ 20-40ns فقط، أقل من الـ 250ns اللي الـ EEPROM يحتاجها.

#### الحل

**إضافة الـ CS timing في الـ DT:**

```dts
&spi1 {
    status = "okay";

    eeprom: at25256b@0 {
        compatible = "atmel,at25256b", "atmel,at25";
        reg = <0>;
        spi-max-frequency = <5000000>;   /* 5 MHz max for AT25256B */

        /* CS timing requirements from datasheet */
        /* tCS = 250ns minimum → use 500ns for margin */
        spi-cs-setup-delay-ns = <500>;
        spi-cs-hold-delay-ns  = <250>;

        pagesize = <64>;
        size     = <32768>;
        address-width = <16>;
    };
};
```

**لو الـ DT properties مش كافية، تعمله في الـ driver:**

```c
static int at25256b_probe(struct spi_device *spi)
{
    /* Set CS timing explicitly */
    spi->cs_setup.value = 500;
    spi->cs_setup.unit  = SPI_DELAY_UNIT_NSECS;

    spi->cs_hold.value  = 250;
    spi->cs_hold.unit   = SPI_DELAY_UNIT_NSECS;

    /* Apply settings */
    if (spi_setup(spi) < 0) {
        dev_err(&spi->dev, "spi_setup failed\n");
        return -EIO;
    }

    dev_info(&spi->dev, "CS setup=%uns, hold=%uns\n",
             spi->cs_setup.value, spi->cs_hold.value);

    return 0;
}
```

**أو باستخدام `set_cs_timing` مباشرة لو الـ controller يدعمه:**

```c
/* From spi-summary: ctrl->set_cs_timing() */
spi_set_cs_timing(spi,
    500,   /* setup_clk_cycles (converted from ns) */
    250,   /* hold_clk_cycles */
    0);    /* inactive_clk_cycles */
```

**تحقق من الـ timing بالـ scope:**

```bash
# Trigger on CS falling edge
# قِس الوقت من CS → first SCLK rising edge
# لازم يكون >= 250ns

# في الـ kernel stats:
cat /sys/bus/spi/devices/spi1.0/../statistics/timedout
# لو بيزيد → في مشكلة timing
```

**اختبار stress في الـ temperature chamber:**

```bash
# كتابة/قراءة متكررة مع verification
for i in $(seq 1 10000); do
    dd if=/dev/urandom of=/dev/mtd0 bs=64 count=1 seek=$((i % 512)) 2>/dev/null
    dd if=/dev/mtd0 of=/tmp/verify bs=64 count=1 skip=$((i % 512)) 2>/dev/null
    # compare...
done
echo "Error count: $?"
```

#### الدرس المستفاد
**الـ CS timing مش بس cosmetic — في الـ automotive وغيرها بيكون فارق بين نجاح وفشل.** الـ spi-summary بينبه على `cs_change` flag وdelays بين transfers، وكمان بيوضح إن `set_cs_timing()` موجود بالظبط لحالة زي دي. دايمًا راجع الـ datasheet لـ `tCS`, `tCSH`, `tCSS` وخلي عندك margin خصوصًا في temperature extremes.
## Phase 7: مصادر ومراجع

### توثيق الـ Kernel الرسمي

| المسار | الوصف |
|--------|-------|
| `Documentation/spi/spi-summary.rst` | الوثيقة الرئيسية للـ SPI subsystem — نظرة عامة شاملة على الـ API وطريقة كتابة الـ drivers |
| `Documentation/spi/spidev.rst` | الـ userspace API عبر `/dev/spidevB.C` |
| `Documentation/spi/pxa2xx.rst` | مثال controller driver لمعالجات Intel/Marvell PXA |
| `Documentation/spi/ep93xx_spi.rst` | controller driver لـ Cirrus Logic EP93xx |
| `include/linux/spi/spi.h` | التعريفات الأساسية: `spi_device`, `spi_controller`, `spi_message`, `spi_transfer` |
| `drivers/spi/` | مجلد كل الـ SPI controller drivers في الـ kernel |
| `drivers/spi/spi.c` | الـ SPI core — قلب الـ subsystem |
| `drivers/spi/spi-bitbang.c` | الـ bitbanging framework للـ GPIO-based controllers |

**الـ kernel API documentation** المُوَلَّدة تلقائياً من الـ kerneldoc comments:
- https://static.lwn.net/kerneldoc/driver-api/spi.html
- https://www.kernel.org/doc/html/latest/spi/spidev.html
- https://docs.kernel.org/driver-api/mtd/spi-nor.html

---

### مقالات LWN.net

دي أهم المقالات اللي غطّت تطوير الـ SPI subsystem من الأول:

| المقال | الرابط | الأهمية |
|--------|--------|---------|
| **simple SPI framework** — أول patch قدّمه David Brownell لإضافة الـ SPI framework للـ kernel | [lwn.net/Articles/154425](https://lwn.net/Articles/154425/) | تاريخي — بداية الـ subsystem |
| **SPI core** — المراجعة الأولى للـ SPI core قبل الـ merge | [lwn.net/Articles/138037](https://lwn.net/Articles/138037/) | أساسي لفهم التصميم الأوّلي |
| **spi** — نقاش مبكر حول الـ SPI API | [lwn.net/Articles/146581](https://lwn.net/Articles/146581/) | |
| **SPI core refresh** — تحديث الـ SPI core بعد التغذية الراجعة من المجتمع | [lwn.net/Articles/162268](https://lwn.net/Articles/162268/) | يشرح التحسينات على الـ API |
| **Add SPI over GPIO driver** — إضافة الـ `spi-gpio` driver للـ bitbanging | [lwn.net/Articles/290068](https://lwn.net/Articles/290068/) | مهم لفهم الـ bitbang controllers |
| **spi: Add slave mode support** — إضافة دعم وضع الـ target/slave للـ SPI | [lwn.net/Articles/723440](https://lwn.net/Articles/723440/) | حديث — SPI target mode |
| **drivers: spi: Add qspi flash controller** — إضافة الـ QSPI controller | [lwn.net/Articles/557211](https://lwn.net/Articles/557211/) | مهم لـ SPI flash |
| **spi: stm32: add spi slave mode** — مثال تطبيقي على slave mode في STM32 | [lwn.net/Articles/930471](https://lwn.net/Articles/930471/) | |
| **SPI LPC information kernel module** | [lwn.net/Articles/824829](https://lwn.net/Articles/824829/) | |

---

### Commits مهمة في تاريخ الـ SPI Subsystem

الـ commits دي شكّلت الـ SPI subsystem كما هي دلوقتي:

```bash
# أول إضافة للـ SPI framework — David Brownell, 2005
git log --oneline --all -- drivers/spi/spi.c | tail -20

# إضافة الـ SPI message queue الجديدة
# commit اللي حوّل الـ transfer() القديمة لـ transfer_one_message()
git log --oneline --grep="spi: add message queue" -- drivers/spi/

# إضافة الـ SPI slave/target mode
git log --oneline --grep="spi.*slave" -- drivers/spi/ include/linux/spi/
```

**الـ commits الأساسية على GitHub:**
- https://github.com/torvalds/linux/commits/master/drivers/spi/spi.c
- https://github.com/torvalds/linux/commits/master/include/linux/spi/spi.h

---

### Mailing List الرسمي

الـ SPI subsystem ليها mailing list مخصص:

- **القائمة البريدية:** `linux-spi@vger.kernel.org`
- **الـ maintainer:** Mark Brown (مسؤول الـ SPI subsystem منذ 2008 تقريباً)
- **أرشيف كامل على lore.kernel.org:** https://lore.kernel.org/linux-spi/
- **أرشيف LKML:** https://lkml.org/

للبحث في النقاشات القديمة:
```bash
# ابحث في الأرشيف عبر lore
# https://lore.kernel.org/linux-spi/?q=spi_transfer_one
```

---

### kernelnewbies.org

الموقع ده بيوثّق التغييرات في كل إصدار من الـ kernel، ومفيد لتتبع إضافات الـ SPI عبر الإصدارات:

| الرابط | ما يحتويه |
|--------|----------|
| [Linux_2_6_24](https://kernelnewbies.org/Linux_2_6_24) | أول دعم لـ MMC/SD over SPI |
| [Linux_3.15-DriversArch](https://kernelnewbies.org/Linux_3.15-DriversArch) | تحديثات الـ SPI drivers في 3.15 |
| [Linux_4.18](https://kernelnewbies.org/Linux_4.18) | تحسينات الـ SPI في 4.18 |
| [Linux_6.5](https://kernelnewbies.org/Linux_6.5) | آخر التحديثات في 6.5 |
| [Linux_6.6](https://kernelnewbies.org/Linux_6.6) | تحديثات الـ SPI في 6.6 |
| [Linux_6.11](https://kernelnewbies.org/Linux_6.11) | تحديثات الـ SPI في 6.11 |

---

### elinux.org

| الرابط | الوصف |
|--------|-------|
| [RPi SPI](https://elinux.org/RPi_SPI) | تفعيل وإعداد الـ SPI على Raspberry Pi — مثال عملي ممتاز |
| [BeagleBoard/SPI/Patch-2.6.37](https://elinux.org/BeagleBoard/SPI/Patch-2.6.37) | patch الـ SPI على BeagleBoard |
| [An Introduction to SPI-NOR Subsystem (PDF)](https://elinux.org/images/4/4e/An_Introduction_to_SPI-NOR_Subsystem_-_v3_0.pdf) | مقدمة متعمقة في الـ SPI-NOR subsystem |
| [SPI Memory Support in Linux and U-Boot (PDF)](https://elinux.org/images/7/73/Raynal-spi-memories.pdf) | مقارنة دعم الـ SPI memory في Linux وU-Boot |

---

### كتب مُوصى بها

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الفصل 6:** Data Types in the Kernel
- **الفصل 14:** The Linux Device Model — ضروري لفهم `spi_driver` و`spi_device` كـ `device` و`device_driver`
- الكتاب متاح مجاناً: https://lwn.net/Kernel/LDD3/

#### Linux Kernel Development — Robert Love
- **الفصل 17:** Devices and Modules — فهم الـ bus/device/driver model اللي الـ SPI subsystem مبني عليه
- **الفصل 13:** The Virtual Filesystem — مفيد لفهم الـ sysfs entries للـ SPI

#### Embedded Linux Primer — Christopher Hallinan
- **الفصل 9:** File Systems for Embedded Linux — يغطي الـ MTD و SPI flash storage
- بيشرح الفرق بين الـ NAND وSPI NOR flash في السياق العملي

#### Essential Linux Device Drivers — Sreekrishnan Venkateswaran
- فصل مخصص للـ SPI بيشرح كتابة protocol driver من الصفر
- مثال حقيقي لـ touchscreen driver على SPI

---

### مصادر إضافية للـ SPI-NOR

الـ SPI-NOR subsystem انفصل عن الـ MTD subsystem وأصبح له كود مستقل:

```
drivers/mtd/spi-nor/          ← SPI NOR flash subsystem
include/linux/mtd/spi-nor.h   ← API خاص بـ SPI NOR
```

- **توثيق الـ SPI-NOR framework:** https://docs.kernel.org/driver-api/mtd/spi-nor.html
- **مقدمة من Texas Instruments:** http://events17.linuxfoundation.org/sites/events/files/slides/An%20Introduction%20to%20SPI-NOR%20Subsystem%20-%20v3_0.pdf

---

### مصطلحات البحث

لو عايز تبحث عن معلومات إضافية:

```
# بحث في الـ kernel documentation
"linux spi subsystem driver development"
"spi_controller transfer_one_message"
"spi_message spi_transfer linux kernel"
"linux spi bitbang gpio"
"spi slave mode linux target"
"linux spidev userspace"
"SPI DMA kernel driver"
"linux spi board info device tree"
"spi_register_board_info platform_data"
"linux MTD SPI NOR driver"
```

للبحث في الكود مباشرة:
```bash
# في مصدر الـ kernel
grep -r "spi_register_driver"  drivers/spi/
grep -r "struct spi_driver"    include/linux/spi/spi.h
grep -r "spi_alloc_host"       drivers/spi/
cscope -d -f cscope.out        # للتنقل في الكود
```

للبحث عبر الـ Git log:
```bash
git log --oneline --since="2020-01-01" -- drivers/spi/ | head -50
git log --oneline --grep="spi:" --since="2023-01-01" | head -30
```
## Phase 8: Writing simple module

### الفكرة: Hook على tracepoint `spi_message_done`

**الـ** `spi_message_done` tracepoint بيتفعّل في kernel كل ما SPI message خلصت — سواء نجحت أو فشلت. ده أكتر نقطة مفيدة عشان تشوف إيه اللي بيتبعت على الـ bus، وكام byte اتبعت فعلاً مقارنة بالمطلوب.

بنستخدم `DEFINE_EVENT` اللي موجود في `include/trace/events/spi.h` — الـ proto بتاعته:

```c
TP_PROTO(struct spi_message *msg)
```

الـ `spi_message` بيحتوي على:
- `msg->spi` → pointer للـ `spi_device` (ومنه نعرف bus_num و chip_select)
- `msg->frame_length` → إجمالي الـ bytes المطلوبة
- `msg->actual_length` → الـ bytes اللي اتبعت فعلاً
- `msg->status` → 0 = نجاح، negative = error

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * spi_msg_probe.c
 *
 * Tracepoint hook on spi_message_done:
 * prints bus/CS, requested vs actual bytes, and status for every
 * completed SPI message.
 */

#define pr_fmt(fmt) "spi_msg_probe: " fmt

#include <linux/module.h>       /* MODULE_* macros, module_init/exit       */
#include <linux/kernel.h>       /* pr_info()                               */
#include <linux/tracepoint.h>   /* for_each_kernel_tracepoint, tracepoint_probe_register */
#include <linux/spi/spi.h>      /* struct spi_message, spi_get_chipselect  */

/* ------------------------------------------------------------------ */
/*  1. Tracepoint probe callback                                       */
/* ------------------------------------------------------------------ */

/*
 * called_from_tracepoint() — الـ kernel بيستدعيها كل ما spi_message_done
 * يتفعّل. الـ proto لازم يطابق TP_PROTO في trace/events/spi.h بالظبط.
 *
 * ignore_data: مش بنستخدمه هنا، بس محتاج يكون موجود عشان signature صح.
 */
static void probe_spi_message_done(void *ignore_data, struct spi_message *msg)
{
    int bus   = msg->spi->controller->bus_num;
    int cs    = spi_get_chipselect(msg->spi, 0);
    unsigned frame  = msg->frame_length;   /* total bytes requested      */
    unsigned actual = msg->actual_length;  /* bytes actually transferred */
    int status      = msg->status;         /* 0 = ok, -errno = error     */

    pr_info("spi%d.%d  requested=%u  actual=%u  status=%d\n",
            bus, cs, frame, actual, status);
}

/* ------------------------------------------------------------------ */
/*  2. Tracepoint lookup helper                                        */
/* ------------------------------------------------------------------ */

/*
 * struct tp_and_probe: بنستخدمها عشان نحتفظ بـ pointer للـ tracepoint
 * بعد ما نلاقيه بـ for_each_kernel_tracepoint().
 */
struct tp_and_probe {
    const char          *name;
    struct tracepoint   *tp;
};

static struct tp_and_probe tp_spi_msg_done = {
    .name = "spi_message_done",
    .tp   = NULL,
};

/*
 * find_tp() — callback بيتكالل لكل tracepoint في الـ kernel.
 * لو الاسم اتطابق نحفظ الـ pointer ونوقف البحث.
 */
static void find_tp(struct tracepoint *tp, void *priv)
{
    struct tp_and_probe *tpp = priv;

    if (strcmp(tp->name, tpp->name) == 0)
        tpp->tp = tp;
}

/* ------------------------------------------------------------------ */
/*  3. module_init                                                     */
/* ------------------------------------------------------------------ */

static int __init spi_msg_probe_init(void)
{
    int ret;

    /* ابحث عن الـ tracepoint بالاسم في كل الـ tracepoints المسجّلة */
    for_each_kernel_tracepoint(find_tp, &tp_spi_msg_done);

    if (!tp_spi_msg_done.tp) {
        pr_err("tracepoint '%s' not found — is CONFIG_SPI_DEBUG enabled?\n",
               tp_spi_msg_done.name);
        return -ENOENT;
    }

    /*
     * سجّل الـ probe على الـ tracepoint.
     * NULL الأخير = no extra data passed to the probe.
     */
    ret = tracepoint_probe_register(tp_spi_msg_done.tp,
                                    probe_spi_message_done,
                                    NULL);
    if (ret) {
        pr_err("tracepoint_probe_register failed: %d\n", ret);
        return ret;
    }

    pr_info("loaded — watching spi_message_done\n");
    return 0;
}

/* ------------------------------------------------------------------ */
/*  4. module_exit                                                     */
/* ------------------------------------------------------------------ */

static void __exit spi_msg_probe_exit(void)
{
    /*
     * لازم نشيل الـ probe قبل ما الكود يتشال من الذاكرة.
     * لو ما شلناهوش والـ kernel اتصل فيه بعد unload → kernel panic.
     * tracepoint_synchronize_unregister() بتضمن إن مفيش execution جارية
     * وقت الشيل.
     */
    tracepoint_probe_unregister(tp_spi_msg_done.tp,
                                probe_spi_message_done,
                                NULL);
    tracepoint_synchronize_unregister();

    pr_info("unloaded\n");
}

module_init(spi_msg_probe_init);
module_exit(spi_msg_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("SPI Observer");
MODULE_DESCRIPTION("Trace every completed SPI message via spi_message_done tracepoint");
```

---

### Makefile

```makefile
obj-m += spi_msg_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/module.h` | الـ macros الأساسية زي `module_init/exit`, `MODULE_LICENSE` |
| `linux/kernel.h` | `pr_info()` و `pr_err()` |
| `linux/tracepoint.h` | `tracepoint_probe_register`, `for_each_kernel_tracepoint` |
| `linux/spi/spi.h` | `struct spi_message`, `spi_get_chipselect()` |

**الـ** `pr_fmt` macro بيحط prefix تلقائي على كل رسائل الـ log عشان يسهل البحث في `dmesg`.

---

#### الـ Probe Callback

الـ `probe_spi_message_done` بياخد `struct spi_message *msg` اللي بيحتوي على كل معلومات الـ transfer بعد ما خلص. بنطبع الـ bus number والـ chip select عشان نعرف أي device، وبنقارن `frame_length` بـ `actual_length` عشان نكشف transfers ناقصة أو errors.

---

#### الـ Tracepoint Lookup

بدل ما نعمل `#define CREATE_TRACE_POINTS` وندخل في complexity الـ compile-time tracepoints، بنستخدم `for_each_kernel_tracepoint()` اللي بتلف على كل الـ tracepoints المسجّلة في الـ kernel وقت الـ runtime. ده أسلم لـ out-of-tree modules عشان ما نحتاجش الـ kernel source.

---

#### الـ module_init

بنبحث عن الـ tracepoint بالاسم أول حاجة، لو ما لقيناهوش نرجع `-ENOENT` بدل ما نـ crash. بعدين بنسجّل الـ probe بـ `tracepoint_probe_register()`. الـ `NULL` الأخير هو الـ `data` pointer اللي بيتبعت للـ callback — مش محتاجينه هنا.

---

#### الـ module_exit

**لازم** نشيل الـ probe في الـ exit قبل ما الـ kernel يشيل الكود من الذاكرة. لو ما عملناش ده والـ tracepoint اتفعّل بعد الـ unload، الـ kernel هيـ call عنوان كود مش موجود → **kernel panic**. الـ `tracepoint_synchronize_unregister()` بتعمل `synchronize_rcu()` داخلياً عشان تتأكد إن مفيش CPU لسه شغال في الـ probe وقت ما بنشيلها.

---

### تشغيل واختبار

```bash
# Build
make

# Load
sudo insmod spi_msg_probe.ko

# شوف الـ output
sudo dmesg | grep spi_msg_probe

# مثال output لو في SPI device على الجهاز:
# spi_msg_probe: spi0.0  requested=4  actual=4  status=0
# spi_msg_probe: spi0.0  requested=2  actual=2  status=0

# Unload
sudo rmmod spi_msg_probe
```

لو الجهاز ما عندوش SPI devices حقيقية، ممكن تشغّل kernel في QEMU مع `-device spi-flash` أو تستخدم `spi-loopback-test` module عشان تولّد traffic وتشوف الـ output.
