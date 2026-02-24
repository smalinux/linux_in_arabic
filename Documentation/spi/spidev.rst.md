## Phase 1: الصورة الكبيرة ببساطة

### ما هو SPI وليه موجود أصلاً؟

تخيل عندك Arduino أو microcontroller صغير وعايز تكلمه من الـ Raspberry Pi أو من أي Linux board. الـ **SPI (Serial Peripheral Interface)** هو بروتوكول تواصل سريع بين الـ processor والـ peripherals زي الـ sensors والـ flash chips والـ displays. بيشتغل على 4 أسلاك أساسية:

```
Master (CPU)          Slave (e.g., sensor)
   MOSI  ─────────────────►  MOSI   (data OUT from master)
   MISO  ◄─────────────────  MISO   (data IN  to master)
   SCLK  ─────────────────►  SCLK   (clock signal)
   CS    ─────────────────►  CS     (chip select - من تكلم)
```

الـ master هو اللي بيتحكم في الـ clock وبيختار مين يتكلم معاه باستخدام الـ **chip select (CS)**. الـ SPI **full-duplex** يعني بيبعت ويستقبل في نفس الوقت.

---

### المشكلة: الـ Kernel Driver مش دايمًا الحل

في الحالة العادية، لما عايز تتحكم في SPI device من Linux، بتكتب **kernel driver** — كود بيشتغل جوا الـ kernel بامتيازات كاملة. لكن ده فيه مشاكل:

1. **لو فيه bug → kernel panic** — الجهاز بيموت وانت بتطور.
2. **التطوير بطيء** — كل تغيير محتاج recompile وreload للـ module.
3. **بتطور protocol جديد مع microcontroller** — ممكن تغيره كل شوية.
4. **Prototyping** — عايز تجرب بسرعة من Python أو C في userspace.

الحل؟ **spidev** — driver بسيط جداً بيفتح نافذة بين الـ userspace وأي SPI device على الشبكة مباشرةً.

---

### الصورة الكبيرة: spidev.rst بتوثق إيه بالضبط؟

الـ `spidev.rst` هو توثيق الـ **userspace API** الخاصة بالـ `spidev` driver. الـ `spidev` ده driver خاص جداً: مش بيعمل حاجة تانية غير إنه **يعرض SPI device كـ character device** في `/dev/spidevB.C`. ومن هناك، أي برنامج userspace يقدر يعمل:

- `open("/dev/spidev0.0", ...)` — يفتح الجهاز
- `read()` / `write()` — half-duplex بسيط
- `ioctl(SPI_IOC_MESSAGE)` — full-duplex أو متعدد الـ transfers مرة واحدة
- `ioctl(SPI_IOC_WR_MODE)` / `SPI_IOC_RD_*` — ضبط الـ clock mode والسرعة وغيرها

---

### القصة من الأول للآخر

**الموقف:** أنت بتعمل board فيها Raspberry Pi و STM32 microcontroller مربوط بـ SPI. الـ STM32 بيقيس درجة حرارة وعايز ترسله أوامر وتاخد البيانات منه.

**الخطوات بدون spidev:**
1. تكتب kernel driver كامل.
2. تعرّف SPI protocol جوا الـ kernel.
3. كل تغيير في الـ protocol = recompile.

**الخطوات مع spidev:**
1. تضيف entry للـ STM32 في Device Tree أو `spi_board_info`.
2. الـ kernel يعمل `/dev/spidev0.0` أوتوماتيك.
3. تكتب Python script أو C program يفتح الـ device ويبعت ويستقبل.
4. لو غيرت الـ protocol → تعدّل الـ userspace code بس، مفيش kernel recompile.

```c
int fd = open("/dev/spidev0.0", O_RDWR);

// Set SPI mode
uint8_t mode = SPI_MODE_0;
ioctl(fd, SPI_IOC_WR_MODE, &mode);

// Full duplex transfer
struct spi_ioc_transfer tr = {
    .tx_buf = (unsigned long)tx_buf,
    .rx_buf = (unsigned long)rx_buf,
    .len    = 4,
    .speed_hz = 1000000,  // 1 MHz
};
ioctl(fd, SPI_IOC_MESSAGE(1), &tr);
```

---

### Architecture: فين spidev في طبقات الـ SPI Stack؟

```
  ┌─────────────────────────────────┐
  │   Userspace Application         │  ← Python / C program
  │   open("/dev/spidev0.0")        │
  └────────────────┬────────────────┘
                   │ read/write/ioctl
  ┌────────────────▼────────────────┐
  │   spidev driver                 │  ← drivers/spi/spidev.c
  │   (character device, major=153) │     (بيترجم syscalls لـ SPI messages)
  └────────────────┬────────────────┘
                   │ spi_sync() / spi_message
  ┌────────────────▼────────────────┐
  │   SPI Core Framework            │  ← drivers/spi/spi.c
  │   (spi_controller, spi_device)  │
  └────────────────┬────────────────┘
                   │
  ┌────────────────▼────────────────┐
  │   SPI Controller Driver         │  ← e.g. spi-bcm2835.c (Raspberry Pi)
  │   (hardware-specific)           │       spi-pl022.c (ARM PrimeCell)
  └────────────────┬────────────────┘
                   │ hardware registers
  ┌────────────────▼────────────────┐
  │   Physical SPI Bus + Device     │  ← الـ sensor / microcontroller
  └─────────────────────────────────┘
```

---

### الـ Device Naming: B وC بيعنوا إيه؟

الـ `/dev/spidevB.C`:
- **B** = رقم الـ SPI bus (controller رقم كام)
- **C** = رقم الـ chip select على هذا الـ bus

فمثلاً `/dev/spidev0.0` يعني: الـ bus الأول، الـ chip select الأول.

---

### أنواع الـ ioctl المتاحة

| ioctl Command | الوظيفة |
|---|---|
| `SPI_IOC_RD_MODE` / `SPI_IOC_WR_MODE` | قراءة/كتابة الـ SPI mode (0..3) |
| `SPI_IOC_RD_MODE32` / `SPI_IOC_WR_MODE32` | نفسه لكن 32-bit للـ flags الإضافية |
| `SPI_IOC_RD_LSB_FIRST` / `SPI_IOC_WR_LSB_FIRST` | اتجاه البيانات: MSB أو LSB أول |
| `SPI_IOC_RD_BITS_PER_WORD` / `SPI_IOC_WR_BITS_PER_WORD` | حجم الـ word بالـ bits |
| `SPI_IOC_RD_MAX_SPEED_HZ` / `SPI_IOC_WR_MAX_SPEED_HZ` | أقصى سرعة للـ clock |
| `SPI_IOC_MESSAGE(N)` | إرسال N transfer دفعة واحدة (full-duplex مدعوم) |

---

### الـ SPI Modes: CPOL وCPHA

الـ **SPI mode** بيحدد شكل الـ clock وتوقيت قراءة البيانات:

| Mode | CPOL | CPHA | الـ Clock idle | Sample edge |
|---|---|---|---|---|
| `SPI_MODE_0` | 0 | 0 | Low | Rising |
| `SPI_MODE_1` | 0 | 1 | Low | Falling |
| `SPI_MODE_2` | 1 | 0 | High | Falling |
| `SPI_MODE_3` | 1 | 1 | High | Rising |

الـ **CPOL** = قطبية الـ clock وقت السكون. الـ **CPHA** = على أي edge تقرأ البيانات.

---

### تحذيرات مهمة في spidev

- الـ spidev **synchronous فقط** — مفيش async I/O.
- حجم أقصى لكل transfer = صفحة واحدة من الـ memory (4KB افتراضياً).
- **مش تقدر تغير chip select polarity** من userspace — ده ممكن يخرب transfers لـ devices تانية على نفس الـ bus.
- لو الـ device مش موجود، الـ SPI مش بيعطي error واضح — لأن SPI مفيش acknowledgement في البروتوكول نفسه.
- لازم تستخدم **اسم حقيقي** للـ SPI device في الـ Device Tree أو `spi_board_info` — مش "spidev" اسم generic تاني مش مقبول.

---

### Device Binding: إزاي الـ Kernel يعرف يربط spidev بالجهاز؟

الـ spidev driver عنده 3 جداول:
1. **`spidev_spi_ids[]`** — لما الـ device متعرّف بـ `spi_board_info.modalias`
2. **`spidev_dt_ids[]`** — لما الـ device متعرّف في الـ **Device Tree** بـ `compatible` string
3. **`spidev_acpi_ids[]`** — لما الـ device متعرّف بـ **ACPI** بـ `_HID`

أو ممكن تعمل bind يدوي من sysfs:
```bash
echo spidev > /sys/bus/spi/devices/spi0.0/driver_override
echo spi0.0  > /sys/bus/spi/drivers/spidev/bind
```

---

### الملفات المكوّنة للـ SPI Subsystem

#### Core Framework
| الملف | الوظيفة |
|---|---|
| `drivers/spi/spi.c` | الـ core: إدارة الـ controllers والـ devices وقوائم الـ transfers |
| `drivers/spi/spidev.c` | الـ spidev driver: الـ character device interface |
| `include/linux/spi/spi.h` | الـ kernel-internal SPI API |
| `include/uapi/linux/spi/spidev.h` | الـ userspace API (ioctl definitions) |
| `drivers/spi/internals.h` | private core internals |

#### Hardware Controller Drivers (أمثلة)
| الملف | الـ Hardware |
|---|---|
| `drivers/spi/spi-bcm2835.c` | Raspberry Pi |
| `drivers/spi/spi-pl022.c` | ARM PrimeCell |
| `drivers/spi/spi-atmel.c` | Atmel AT91 |
| `drivers/spi/spi-amd.c` | AMD SoCs |
| `drivers/spi/spi-bitbang.c` | Software bitbang (generic) |

#### Documentation
| الملف | المحتوى |
|---|---|
| `Documentation/spi/spidev.rst` | **الملف الحالي** — userspace API |
| `Documentation/spi/spi-summary.rst` | الملخص الشامل للـ SPI subsystem |
| `Documentation/spi/spi-lm70llp.rst` | مثال عملي على spidev hardware |
| `Documentation/spi/multiple-data-lanes.rst` | Dual/Quad/Octal SPI |

#### Tools
| الملف | الوظيفة |
|---|---|
| `tools/spi/spidev_test.c` | test tool شامل |
| `tools/spi/spidev_fdx.c` | مثال full-duplex مذكور في الوثيقة |

#### Device Tree Bindings
| الملف | الوظيفة |
|---|---|
| `Documentation/devicetree/bindings/spi/` | DT bindings لكل الـ SPI controllers |

---

### ملفات يجب معرفتها بجانب spidev.rst

- **`Documentation/spi/spi-summary.rst`** — الصورة الكاملة للـ SPI subsystem من ناحية الـ kernel driver model.
- **`include/uapi/linux/spi/spidev.h`** — تعريفات الـ ioctl اللي بتستخدمها في الكود.
- **`drivers/spi/spidev.c`** — الكود الفعلي للـ driver.
- **`tools/spi/spidev_test.c`** — مثال عملي كامل.
## Phase 2: شرح الـ SPI Framework

### المشكلة — ليه الـ SPI Framework موجود أصلاً؟

تخيل إن عندك SoC فيه 3 SPI controllers، وعليه متصل:
- flash chip (W25Q128) على SPI0
- IMU sensor (ICM-42688) على SPI1
- DAC chip (MCP4922) على SPI1 نفسه (chipselect تاني)

**بدون framework**، كل driver هيعمل إيه؟ هيكتب كوده الخاص عشان:
- يتعامل مع hardware registers الـ SPI controller
- يدير الـ chipselect
- يحل مشكلة الـ bus sharing (لو اتنين drivers حاولوا يكلموا الـ bus في نفس الوقت؟)
- يدعم DMA أو لأ
- يتعامل مع كل SoC بطريقة مختلفة

النتيجة: **كود متكرر وبيتكسر مع كل SoC جديد.**

الـ SPI framework بييجي يحل ده بـ **layer of abstraction** بين:
- الـ **protocol drivers** (اللي بتكلم sensor أو flash — مش بيهمهم الـ hardware)
- الـ **controller drivers** (اللي بيتكلم مع hardware registers الـ SPI controller فعلاً)

---

### الحل — الـ Kernel's Approach

الـ kernel قسّم المسؤولية لـ 3 layers واضحة:

```
┌─────────────────────────────────────────────────┐
│            Protocol / Client Driver              │
│   (مش بيعرف ولا بيهتم إيه SoC بيشتغل عليه)    │
│   مثال: spidev, spi-nor, ads7846 touchscreen     │
└──────────────────────┬──────────────────────────┘
                       │  spi_sync() / spi_async()
                       ▼
┌─────────────────────────────────────────────────┐
│              SPI Core (kernel/spi/spi.c)         │
│  - إدارة الـ bus و الـ queue                    │
│  - Locking و arbitration                         │
│  - DMA mapping helpers                           │
│  - sysfs / device model integration              │
└──────────────────────┬──────────────────────────┘
                       │  transfer_one() / transfer_one_message()
                       ▼
┌─────────────────────────────────────────────────┐
│         SPI Controller Driver (Provider)         │
│   مثال: spi-bcm2835.c, spi-stm32.c, spi-imx.c  │
│   - بيكتب في hardware registers                 │
│   - بيتعامل مع DMA channels                     │
│   - بيدير الـ chipselect lines                  │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
              [ SPI Hardware على الـ SoC ]
```

الـ **spidev** driver هو حالة خاصة: هو protocol driver بس بدل ما يكون kernel-only، بيعمل **character device** (`/dev/spidevB.C`) يخلي userspace يتكلم SPI مباشرة.

---

### الـ Real-World Analogy — مكتب بريد في شركة كبيرة

تخيل شركة فيها **مكتب بريد مركزي** (SPI Core):

| المفهوم الحقيقي | الـ Analogy |
|---|---|
| **SPI Core** | مكتب البريد المركزي |
| **spi_controller** | شباك البريد الفيزيائي (hardware) |
| **spi_device** | موظف بيبعت خطابات لعنوان معين |
| **spi_message** | طرد/خطاب بيتبعت |
| **spi_transfer** | ورقة واحدة جوه الطرد |
| **chipselect** | عنوان المستلم (يحدد مين يستقبل) |
| **protocol driver** | الموظف اللي بيكتب الخطاب — مش محتاج يعرف شاحنات البريد شغالة ازاي |
| **controller driver** | سائق الشاحنة — بيعرف الطرق والحمولة بس مش محتاج يعرف محتوى الخطاب |
| **bus locking** | مكتب البريد مبيبعتش لاتنين في نفس الوقت من نفس الشباك |
| **DMA** | حزام ناقل أوتوماتيكي بدل ما الموظف يشيل الطرد بإيده |

**التفاصيل الدقيقة في الـ Analogy:**

- لو عندك **اتنين موظفين** (protocol drivers) بيبعتوا على نفس الشباك (controller) → مكتب البريد بيرتّب ويبعت واحد ورا التاني (bus locking + queue).
- الموظف بيقدر يبعت **طرود متعددة مع بعض** في نفس العملية (spi_message مع متعدد spi_transfers) والشاحنة مش هتوقف في النص.
- الـ **chipselect** زي إن الشاحنة بتقف قدام باب معين — لو عندك 3 أجهزة على نفس الـ bus، كل جهاز بيعرف إنه هو المقصود بس لما بابه بيتفتح.

---

### الـ Big Picture Architecture

```
User Space
──────────────────────────────────────────────────────────
    Application
        │
        │  open("/dev/spidev0.0") → read/write/ioctl
        ▼
Kernel Space
──────────────────────────────────────────────────────────

  ┌──────────────────────────────────────────────────┐
  │                  spidev driver                    │
  │  (character device — major 153)                  │
  │                                                   │
  │  spidev_ioctl() → spi_sync()                     │
  │  spidev_read()  → spi_sync()                     │
  │  spidev_write() → spi_sync()                     │
  └──────────────────────┬───────────────────────────┘
                         │
                         ▼  (spi_message + spi_transfer)
  ┌──────────────────────────────────────────────────┐
  │                   SPI Core                        │
  │          (drivers/spi/spi.c)                     │
  │                                                   │
  │  spi_sync()                                       │
  │    → __spi_queued_transfer()                     │
  │    → pump_messages() (kthread)                   │
  │    → spi_map_msg() (DMA mapping)                 │
  │    → ctlr->transfer_one_message()               │
  └──────────────────────┬───────────────────────────┘
                         │
                         ▼
  ┌──────────────────────────────────────────────────┐
  │           SPI Controller Driver                   │
  │   مثال: drivers/spi/spi-bcm2835.c              │
  │                                                   │
  │  .transfer_one() أو .transfer_one_message()      │
  │  ├── يكتب في FIFO registers                     │
  │  ├── يشغل DMA transfer                          │
  │  └── ينتظر interrupt أو polling                  │
  └──────────────────────┬───────────────────────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │   SPI Hardware       │
              │  MOSI / MISO / SCK  │
              │  CS0 / CS1 / CS2    │
              └──────────────────────┘
                    │       │
              ┌─────┘   ┌───┘
              ▼         ▼
         [Flash]    [Sensor]
```

---

### الـ Core Abstractions — الـ Structs الأساسية

#### 1. `struct spi_controller` — قلب الـ Framework

ده بيمثل الـ SPI controller hardware (مثلاً SPI1 على الـ SoC).

```c
struct spi_controller {
    struct device   dev;       /* device model integration */
    s16             bus_num;   /* رقم الـ bus (0، 1، 2...) */
    u16             num_chipselect; /* كام device ممكن يتوصل */

    u32             mode_bits;      /* إيه modes بيدعمها الـ controller */
    u32             max_speed_hz;   /* أقصى سرعة */
    u32             min_speed_hz;

    /* *** الـ hooks اللي بيكتبها controller driver *** */

    /* إعداد الـ device (clock, mode, bits_per_word) */
    int (*setup)(struct spi_device *spi);

    /* الطريقة البدائية: بتضيف message للـ queue */
    int (*transfer)(struct spi_device *spi, struct spi_message *mesg);

    /* الطريقة الحديثة (preferred): بتنفذ transfer واحدة */
    int (*transfer_one)(struct spi_controller *ctlr,
                        struct spi_device *spi,
                        struct spi_transfer *transfer);

    /* الـ chipselect control */
    void (*set_cs)(struct spi_device *spi, bool enable);

    /* DMA support */
    bool (*can_dma)(struct spi_controller *ctlr,
                    struct spi_device *spi,
                    struct spi_transfer *xfer);
    struct dma_chan *dma_tx;
    struct dma_chan *dma_rx;

    /* الـ internal queue (بيديرها الـ core) */
    struct kthread_worker   *kworker;    /* message pump thread */
    struct list_head         queue;      /* pending messages */
    struct spi_message      *cur_msg;    /* الـ message اللي بيتنفذ دلوقتي */
};
```

**ملاحظة مهمة:** `transfer_one` و `transfer_one_message` متعادلوش — الـ core بيستخدم `transfer_one_message` generic implementation اللي بتلف على الـ transfers وبتكلم `transfer_one` لكل واحدة.

---

#### 2. `struct spi_device` — الجهاز المتصل

كل chip متصل على الـ SPI bus بيتمثل في `spi_device`:

```c
struct spi_device {
    struct device       dev;
    struct spi_controller *controller;

    u32                 max_speed_hz;   /* أقصى سرعة للـ chip ده */
    u8                  bits_per_word;  /* عادةً 8 */
    u32                 mode;           /* SPI_MODE_0..3 + flags */
    int                 irq;

    char                modalias[SPI_NAME_SIZE]; /* اسم الـ driver */

    /* timing */
    struct spi_delay    cs_setup;    /* delay بعد assert الـ CS */
    struct spi_delay    cs_hold;     /* delay قبل deassert الـ CS */
    struct spi_delay    word_delay;  /* delay بين كل word */

    /* chipselect info */
    u8                  chip_select[SPI_DEVICE_CS_CNT_MAX];
    struct gpio_desc   *cs_gpiod[SPI_DEVICE_CS_CNT_MAX];
};
```

---

#### 3. `struct spi_message` + `struct spi_transfer` — وحدة التنفيذ

الـ **message** هو atomic transaction — مش ممكن جهاز تاني يقاطعه:

```c
struct spi_message {
    struct list_head    transfers;  /* linked list من spi_transfer */
    struct spi_device  *spi;
    int                 status;     /* 0 = success, errno = error */
    void (*complete)(void *context); /* callback لما الـ message يخلص */
    void               *context;
    unsigned            frame_length;   /* إجمالي عدد البايتات */
    unsigned            actual_length;  /* اللي اتنقل فعلاً */
};
```

الـ **transfer** هي خطوة واحدة جوه الـ message:

```c
struct spi_transfer {
    const void  *tx_buf;  /* بيانات الإرسال (NULL = send zeros) */
    void        *rx_buf;  /* buffer الاستقبال (NULL = discard) */
    unsigned     len;     /* عدد البايتات */

    u32          speed_hz;     /* override سرعة الـ device للـ transfer دي */
    u8           bits_per_word; /* override الـ word size */

    unsigned     cs_change:1;  /* deassert الـ CS بعد الـ transfer دي؟ */
    struct spi_delay delay;    /* انتظر قبل الـ transfer الجاية */

    /* DMA addresses (بيملاهم الـ core) */
    dma_addr_t   tx_dma;
    dma_addr_t   rx_dma;

    struct list_head transfer_list; /* الرابط في الـ message */
};
```

---

#### 4. `struct spi_ioc_transfer` — نسخة الـ Userspace

ده نفس `spi_transfer` بس بشكل safe للـ userspace (من `spidev.h`):

```c
struct spi_ioc_transfer {
    __u64   tx_buf;       /* pointer → userspace address */
    __u64   rx_buf;       /* pointer → userspace address */
    __u32   len;
    __u32   speed_hz;
    __u16   delay_usecs;
    __u8    bits_per_word;
    __u8    cs_change;
    __u8    tx_nbits;     /* 1=single, 2=dual, 4=quad */
    __u8    rx_nbits;
    __u8    word_delay_usecs;
    __u8    pad;
};
```

**لماذا `__u64` وليس pointer؟**
عشان الـ pointer size بيختلف بين 32-bit و 64-bit systems، استخدام `__u64` ثابت على الاتنين — بيضمن إن الـ struct layout متغيرش.

---

### كيف الـ Structs بتترابط ببعض

```
spi_controller
│
├── [spi_device 0]  (CS=0, flash W25Q128)
│       └── modalias = "jedec,spi-nor"
│
├── [spi_device 1]  (CS=1, IMU ICM-42688)
│       └── modalias = "invensense,icm42688"
│
└── [spi_device 2]  (CS=2, /dev/spidev0.2 عبر spidev driver)
        └── modalias = "rohm,dh2228fv"

عند تنفيذ spi_sync():

spi_message
├── spi_transfer [0]  { tx_buf="CMD", rx_buf=NULL, len=1 }
├── spi_transfer [1]  { tx_buf=NULL, rx_buf=data, len=256, cs_change=0 }
└── (CS stays asserted across all transfers)
```

---

### SPI Modes — CPOL و CPHA

ده مفهوم hardware أساسي لازم تفهمه قبل ما تكتب أي driver:

```
CPOL=0: الـ clock خامد على 0
CPOL=1: الـ clock خامد على 1

CPHA=0: sample على الـ leading edge
CPHA=1: sample على الـ trailing edge

MODE_0: CPOL=0, CPHA=0  ← الأكتر شيوعاً
MODE_1: CPOL=0, CPHA=1
MODE_2: CPOL=1, CPHA=0
MODE_3: CPOL=1, CPHA=1

      SCK (CPOL=0, CPHA=0 / MODE_0)
       __    __    __    __
      |  |  |  |  |  |  |  |
  ____    ____    ____    ____
  ↑ sample هنا (leading = rising)

MOSI: [  D7  ][  D6  ][  D5  ]...
```

الـ `mode` في `spi_device` بيحدد الـ mode للـ chip ده، والـ controller driver هو اللي بيطبق ده فعلاً في الـ hardware registers.

---

### الـ spidev Driver — الـ Userspace Bridge

**الـ spidev** هو protocol driver مخصوص: بدل ما يكون kernel subsystem، بيكشف `/dev/spidevB.C` character device لـ userspace.

#### إزاي بيشتغل؟

```
userspace app
    │
    │  ioctl(fd, SPI_IOC_MESSAGE(2), transfers)
    ▼
spidev_ioctl()
    │
    │  1. copy_from_user() → الـ spi_ioc_transfer array
    │  2. يبني spi_message + spi_transfer structs
    │  3. spi_sync(spi, &msg)
    ▼
SPI Core → Controller Driver → Hardware
```

#### الـ ioctls المتاحة:

| ioctl | الوظيفة |
|---|---|
| `SPI_IOC_RD_MODE` / `SPI_IOC_WR_MODE` | قراية/كتابة الـ SPI mode (8-bit) |
| `SPI_IOC_RD_MODE32` / `SPI_IOC_WR_MODE32` | نفسه بس 32-bit (يدعم flags زيادة) |
| `SPI_IOC_RD_MAX_SPEED_HZ` / `SPI_IOC_WR_MAX_SPEED_HZ` | السرعة القصوى |
| `SPI_IOC_RD_BITS_PER_WORD` / `SPI_IOC_WR_BITS_PER_WORD` | حجم الـ word |
| `SPI_IOC_RD_LSB_FIRST` / `SPI_IOC_WR_LSB_FIRST` | اتجاه البيتات |
| `SPI_IOC_MESSAGE(N)` | تنفيذ N transfers في message واحدة |

#### مثال عملي — قراية register من IMU sensor:

```c
#include <linux/spi/spidev.h>
#include <sys/ioctl.h>
#include <fcntl.h>

int fd = open("/dev/spidev0.1", O_RDWR);

/* إعداد mode */
uint8_t mode = SPI_MODE_0;
ioctl(fd, SPI_IOC_WR_MODE, &mode);

uint32_t speed = 8000000; /* 8 MHz */
ioctl(fd, SPI_IOC_WR_MAX_SPEED_HZ, &speed);

/* Full-duplex: ابعت command واستقبل response في نفس الوقت */
uint8_t tx[] = { 0x80 | 0x75, 0x00 }; /* read WHO_AM_I register */
uint8_t rx[2] = { 0 };

struct spi_ioc_transfer xfer = {
    .tx_buf        = (uintptr_t)tx,
    .rx_buf        = (uintptr_t)rx,
    .len           = 2,
    .speed_hz      = 8000000,
    .bits_per_word = 8,
};

ioctl(fd, SPI_IOC_MESSAGE(1), &xfer);
/* rx[1] دلوقتي فيه قيمة الـ WHO_AM_I */
```

---

### Device Creation — إزاي الـ spidev بيتربط بالـ device؟

الـ kernel بيدعم 3 طرق لتعريف SPI devices:

#### 1. Device Tree (الأكتر شيوعاً في embedded)

```dts
/* في الـ Device Tree */
&spi0 {
    status = "okay";
    cs-gpios = <&gpio 8 GPIO_ACTIVE_LOW>;

    sensor@0 {
        compatible = "rohm,dh2228fv";  /* لازم يكون في spidev_dt_ids[] */
        reg = <0>;                      /* chipselect رقم 0 */
        spi-max-frequency = <8000000>;
    };
};
```

الـ `compatible` string لازم يكون موجود في `spidev_dt_ids[]` جوه `drivers/spi/spidev.c`.

#### 2. sysfs Manual Binding

```bash
# لو الـ device موجود بس مفيش driver اتربط بيه أوتوماتيك
echo spidev > /sys/bus/spi/devices/spi0.0/driver_override
echo spi0.0  > /sys/bus/spi/drivers/spidev/bind
```

#### 3. Board Info (legacy — قديم)

```c
static struct spi_board_info my_spi_devices[] = {
    {
        .modalias     = "rohm,dh2228fv",
        .max_speed_hz = 8000000,
        .bus_num      = 0,
        .chip_select  = 0,
        .mode         = SPI_MODE_0,
    },
};
spi_register_board_info(my_spi_devices, ARRAY_SIZE(my_spi_devices));
```

---

### الـ Framework يمتلك إيه مقابل إيه بيفوّضه للـ Drivers؟

| السؤال | الـ SPI Core يمتلكه | الـ Controller Driver يمتلكه |
|---|---|---|
| **Queue management** | ✓ message pump thread | ✗ |
| **Bus locking** | ✓ mutex + spinlock | ✗ |
| **DMA mapping** | ✓ sg_table + dma_map | ✗ (بس بيقول can_dma) |
| **sysfs / device model** | ✓ | ✗ |
| **Clock polarity/phase** | ✗ (بس بيتحقق من mode_bits) | ✓ كتابة registers |
| **Chipselect timing** | ✗ | ✓ set_cs() callback |
| **FIFO management** | ✗ | ✓ |
| **Interrupt handling** | ✗ | ✓ |
| **Speed calculation** | ✗ (بيمرر speed_hz) | ✓ يحسب prescaler |
| **Half/Full duplex** | ✗ (بيمرر tx/rx bufs) | ✓ يقرر على حسب hardware |
| **Protocol semantics** | ✗ | ✗ (ده شغل الـ protocol driver) |

---

### قيود الـ spidev اللي لازم تعرفها

| القيد | السبب |
|---|---|
| **Synchronous فقط** | مفيش async I/O من userspace |
| **مش بيقدر يغير CS polarity** | هيأثر على devices تانية على نفس الـ bus |
| **حد أقصى للـ transfer size** | افتراضي PAGE_SIZE، قابل للتغيير بـ module parameter |
| **مفيش إشارة للـ actual bitrate المستخدم** | الـ controller ممكن يختار أقرب قيمة |
| **مفيش IRQ handling** | لو الـ device محتاج interrupt، لازم kernel driver |
| **لا يوجد SPI slave mode** | spidev هو دايماً master |
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options

#### SPI Mode Flags (من `include/uapi/linux/spi/spi.h`)

| Flag | Bit | المعنى |
|------|-----|--------|
| `SPI_CPHA` | 0 | clock phase — sample on trailing edge |
| `SPI_CPOL` | 1 | clock polarity — idle high |
| `SPI_CS_HIGH` | 2 | chipselect active high |
| `SPI_LSB_FIRST` | 3 | إرسال LSB أولاً |
| `SPI_3WIRE` | 4 | SI/SO مشتركين على wire واحد |
| `SPI_LOOP` | 5 | loopback mode |
| `SPI_NO_CS` | 6 | مفيش chipselect (device وحيد على البص) |
| `SPI_READY` | 7 | الـ slave يسحب MISO للـ low عشان يعمل pause |
| `SPI_TX_DUAL` | 8 | إرسال على 2 wires |
| `SPI_TX_QUAD` | 9 | إرسال على 4 wires |
| `SPI_RX_DUAL` | 10 | استقبال على 2 wires |
| `SPI_RX_QUAD` | 11 | استقبال على 4 wires |
| `SPI_CS_WORD` | 12 | toggle CS بعد كل word |
| `SPI_TX_OCTAL` | 13 | إرسال على 8 wires |
| `SPI_RX_OCTAL` | 14 | استقبال على 8 wires |
| `SPI_3WIRE_HIZ` | 15 | high impedance turnaround في 3-wire |
| `SPI_RX_CPHA_FLIP` | 16 | اعكس CPHA في Rx فقط |
| `SPI_MOSI_IDLE_LOW` | 17 | MOSI يفضل low وهو idle |
| `SPI_MOSI_IDLE_HIGH` | 18 | MOSI يفضل high وهو idle |

**الـ `SPI_MODE_USER_MASK`** = bits 0–18 (كل اللي فوق) — ده اللي يقدر userspace يغيره.

**الـ `SPI_MODE_KERNEL_MASK`** = bits 29–31 — للـ kernel بس (`SPI_NO_TX`, `SPI_NO_RX`, `SPI_TPM_HW_FLOW`).

#### SPI Mode Combinations (cheatsheet)

| الاسم | CPOL | CPHA | الوصف |
|-------|------|------|-------|
| `SPI_MODE_0` | 0 | 0 | SCK idle low، sample on rising edge |
| `SPI_MODE_1` | 0 | 1 | SCK idle low، sample on falling edge |
| `SPI_MODE_2` | 1 | 0 | SCK idle high، sample on falling edge |
| `SPI_MODE_3` | 1 | 1 | SCK idle high، sample on rising edge |

#### SPI Controller Flags (`spi_controller.flags`)

| Flag | Bit | المعنى |
|------|-----|--------|
| `SPI_CONTROLLER_HALF_DUPLEX` | 0 | مش بيعمل full duplex |
| `SPI_CONTROLLER_NO_RX` | 1 | مش بيقدر يقرأ |
| `SPI_CONTROLLER_NO_TX` | 2 | مش بيقدر يكتب |
| `SPI_CONTROLLER_MUST_RX` | 3 | لازم يكون في rx buffer |
| `SPI_CONTROLLER_MUST_TX` | 4 | لازم يكون في tx buffer |
| `SPI_CONTROLLER_GPIO_SS` | 5 | GPIO لازم يسلّك الـ CS |
| `SPI_CONTROLLER_SUSPENDED` | 6 | suspended حالياً |
| `SPI_CONTROLLER_MULTI_CS` | 7 | يقدر يعمل assert على أكتر من CS |

#### SPI Transfer nbits

| Constant | Value | المعنى |
|----------|-------|--------|
| `SPI_NBITS_SINGLE` | 0x01 | 1-bit (standard MOSI/MISO) |
| `SPI_NBITS_DUAL` | 0x02 | 2-bit (Dual SPI) |
| `SPI_NBITS_QUAD` | 0x04 | 4-bit (Quad SPI) |
| `SPI_NBITS_OCTAL` | 0x08 | 8-bit (Octal SPI) |

#### SPI_MODE_MASK في spidev (من `spidev.c`)

الـ driver بيسمح بتغيير الـ flags دي بس من userspace:

```c
#define SPI_MODE_MASK  (SPI_MODE_X_MASK | SPI_CS_HIGH | SPI_LSB_FIRST \
                       | SPI_3WIRE | SPI_LOOP | SPI_NO_CS | SPI_READY \
                       | SPI_TX_DUAL | SPI_TX_QUAD | SPI_TX_OCTAL \
                       | SPI_RX_DUAL | SPI_RX_QUAD | SPI_RX_OCTAL \
                       | SPI_RX_CPHA_FLIP | SPI_3WIRE_HIZ | SPI_MOSI_IDLE_LOW)
```

---

### الـ Structs المهمة

---

#### 1. `struct spidev_data` — القلب الحقيقي لـ spidev

ده الـ struct اللي بيمثل **device node واحد** (`/dev/spidevB.C`). بيتخلق في `spidev_probe()` ويتحذف في `spidev_remove()` أو `spidev_release()`.

```c
/* من drivers/spi/spidev.c */
struct spidev_data {
    dev_t           devt;         /* رقم الـ device (major:minor) */
    struct mutex    spi_lock;     /* يحمي الـ spi pointer من الحذف */
    struct spi_device *spi;       /* pointer للـ SPI device في الـ kernel */
    struct list_head device_entry;/* عنصر في الـ device_list العالمي */

    /* TX/RX buffers — NULL لحد ما حد يفتح الـ device */
    struct mutex    buf_lock;     /* يحمي الـ buffers والـ I/O */
    unsigned        users;        /* عدد الـ open() النشطين */
    u8             *tx_buffer;    /* bounce buffer للإرسال (bufsiz bytes) */
    u8             *rx_buffer;    /* bounce buffer للاستقبال (bufsiz bytes) */
    u32             speed_hz;     /* الـ speed الحالية للـ transfers */
};
```

**العلاقات:**
- بيشاور على `struct spi_device` عبر `spi`.
- مربوط في `device_list` العالمية (محمية بـ `device_list_lock`).
- الـ `file->private_data` بيشاور عليه في كل file operation.

---

#### 2. `struct spi_ioc_transfer` — وحدة النقل من userspace

ده الـ struct اللي userspace بيبعته لـ kernel في `SPI_IOC_MESSAGE(N)`. معناه "اعمل transfer بالتفاصيل دي".

```c
/* من include/uapi/linux/spi/spidev.h */
struct spi_ioc_transfer {
    __u64  tx_buf;          /* userspace pointer للـ TX data (أو NULL) */
    __u64  rx_buf;          /* userspace pointer للـ RX data (أو NULL) */
    __u32  len;             /* طول الـ buffer بالبايت */
    __u32  speed_hz;        /* override للـ speed (0 = استخدم الـ default) */
    __u16  delay_usecs;     /* delay بعد آخر bit قبل ما CS يتغير */
    __u8   bits_per_word;   /* override لحجم الـ word (0 = default) */
    __u8   cs_change;       /* 1 = deselect الـ chip بعد الـ transfer */
    __u8   tx_nbits;        /* عدد الـ lanes للإرسال */
    __u8   rx_nbits;        /* عدد الـ lanes للاستقبال */
    __u8   word_delay_usecs;/* delay بين كل word */
    __u8   pad;             /* padding لمحاذاة الـ struct */
};
```

**ملاحظة:** الـ `tx_buf` و`rx_buf` هنا `__u64` حتى في 32-bit userspace — ده متعمد عشان الـ struct يبقى بنفس الحجم على 32-bit و64-bit.

---

#### 3. `struct spi_transfer` — وحدة النقل داخل الـ kernel

ده اللي kernel بيستخدمه internally. بيتبني من `spi_ioc_transfer` في `spidev_message()`.

```c
/* من include/linux/spi/spi.h */
struct spi_transfer {
    const void    *tx_buf;       /* kernel virtual address للـ TX */
    void          *rx_buf;       /* kernel virtual address للـ RX */
    unsigned       len;          /* طول بالبايت */
    u16            error;        /* SPI_TRANS_FAIL_NO_START أو SPI_TRANS_FAIL_IO */
    unsigned       cs_change:1;  /* toggle chipselect بعد الـ transfer */
    unsigned       tx_nbits:4;   /* عدد TX lanes */
    unsigned       rx_nbits:4;   /* عدد RX lanes */
    u8             bits_per_word;
    struct spi_delay delay;       /* delay بعد الـ transfer */
    struct spi_delay word_delay;  /* delay بين الـ words */
    u32            speed_hz;
    u32            effective_speed_hz; /* الـ speed الفعلية اللي اتنفذت */
    struct list_head transfer_list;    /* ربطه بالـ spi_message */
    /* ... DMA fields وغيره */
};
```

---

#### 4. `struct spi_message` — مجموعة transfers اتومية

تسلسل من `spi_transfer` بيتنفذ كـ transaction واحدة — الـ CS بيفضل active طول الرسالة.

```c
/* من include/linux/spi/spi.h */
struct spi_message {
    struct list_head   transfers;      /* linked list من spi_transfer */
    struct spi_device *spi;            /* الـ target device */
    int                status;         /* 0 = نجاح، سالب = error */
    void (*complete)(void *context);   /* callback لما تخلص */
    void              *context;        /* argument للـ callback */
    unsigned           frame_length;   /* إجمالي الـ bytes المطلوبة */
    unsigned           actual_length;  /* إجمالي الـ bytes اللي اتنقلت فعلاً */
    struct list_head   queue;          /* للـ controller driver */
    /* ... */
};
```

---

#### 5. `struct spi_device` — تمثيل الـ SPI slave device في الـ kernel

```c
/* من include/linux/spi/spi.h */
struct spi_device {
    struct device       dev;           /* الـ device model base */
    struct spi_controller *controller; /* الـ host controller */
    u32                 max_speed_hz;  /* أقصى سرعة */
    u8                  bits_per_word; /* حجم الـ word الافتراضي */
    u32                 mode;          /* SPI mode flags */
    int                 irq;           /* رقم الـ IRQ */
    char                modalias[SPI_NAME_SIZE]; /* اسم الـ driver */
    struct spi_delay    word_delay;    /* delay بين الـ words */
    /* ... chipselect arrays وغيره */
};
```

---

#### 6. `struct spi_controller` — الـ host controller

بيمثل الـ SPI bus controller نفسه (مثلاً SPI0 على RPI). بيدير queue من الـ messages.

**الـ fields المهمة لـ spidev:**
- `setup(spi)` — بيتاخد بعد `SPI_IOC_WR_*` ioctls.
- `transfer(spi, msg)` — الـ entry point لإرسال message.
- `io_mutex` — mutex للـ physical bus access.
- `bus_lock_mutex` + `bus_lock_flag` — exclusive bus locking.

---

### رسم علاقات الـ Structs

```
  userspace                kernel space
  ─────────                ────────────

  /dev/spidev0.0
       │
       │ open()
       ▼
  struct file
  ┌─────────────────┐
  │ private_data ───┼──────────────────────────────┐
  └─────────────────┘                              │
                                                   ▼
                                         struct spidev_data
                                         ┌──────────────────────┐
                                         │ devt                 │
                                         │ spi_lock (mutex)     │
                                         │ buf_lock (mutex)     │
                                         │ users                │
                                         │ tx_buffer  ──────────┼──► kmalloc(bufsiz)
                                         │ rx_buffer  ──────────┼──► kmalloc(bufsiz)
                                         │ speed_hz             │
                                         │ device_entry ────────┼──► device_list (global)
                                         │ spi ─────────────────┼─┐
                                         └──────────────────────┘ │
                                                                   ▼
                                                        struct spi_device
                                                        ┌──────────────────┐
                                                        │ dev              │
                                                        │ max_speed_hz     │
                                                        │ bits_per_word    │
                                                        │ mode             │
                                                        │ controller ──────┼──► struct spi_controller
                                                        └──────────────────┘

  ioctl(SPI_IOC_MESSAGE(N))
       │
       │ memdup_user()
       ▼
  struct spi_ioc_transfer[N]  (kernel copy)
       │
       │ spidev_message()
       ▼
  struct spi_transfer[N]  (kcalloc)
       │
       │ spi_message_add_tail()
       ▼
  struct spi_message
  ┌──────────────────────┐
  │ transfers ◄──────────┼──► list of spi_transfer
  │ spi ─────────────────┼──► spi_device
  │ status               │
  │ actual_length        │
  └──────────────────────┘
       │
       │ spi_sync()
       ▼
  spi_controller->transfer() / transfer_one_message()
       │
       ▼
  Hardware SPI Registers
```

---

### دورة حياة الـ spidev Device

```
Module Load (spidev_init)
│
├── register_chrdev(153, "spi", &spidev_fops)
│       └── حجز major number 153
├── class_register(&spidev_class)
│       └── udev هيشوف الـ class ويعمل /dev nodes
└── spi_register_driver(&spidev_spi_driver)
        └── ربط الـ driver بالـ bus

                │
                │ Device Tree / ACPI / board_info يلاقي compatible match
                ▼

spidev_probe(spi)
│
├── kzalloc(spidev_data)
├── spidev->spi = spi
├── mutex_init(&spidev->spi_lock)
├── mutex_init(&spidev->buf_lock)
├── find_first_zero_bit(minors)  ← بيجيب minor number حر
├── MKDEV(153, minor)
├── device_create(...)  ← udev يشوف event ويعمل /dev/spidev0.0
├── set_bit(minor, minors)
├── list_add(&spidev->device_entry, &device_list)
└── spi_set_drvdata(spi, spidev)

                │
                │ userspace: open("/dev/spidev0.0")
                ▼

spidev_open(inode, filp)
│
├── mutex_lock(&device_list_lock)
├── بحث في device_list عن devt == inode->i_rdev
├── kmalloc(tx_buffer, bufsiz)  ← بس لو users كان 0
├── kmalloc(rx_buffer, bufsiz)
├── spidev->users++
├── filp->private_data = spidev
└── mutex_unlock(&device_list_lock)

                │
                │ read/write/ioctl calls
                ▼

  [استخدام عادي — مشروح تفصيلاً في call flows]

                │
                │ userspace: close(fd)
                ▼

spidev_release(inode, filp)
│
├── mutex_lock(&device_list_lock)
├── spidev->users--
├── لو users == 0:
│   ├── kfree(tx_buffer)
│   ├── kfree(rx_buffer)
│   └── لو spidev->spi == NULL (unbind حصل وانت شاغل):
│           kfree(spidev)  ← أنت المسؤول تحذفه
└── mutex_unlock(&device_list_lock)

                │
                │ sysfs unbind أو rmmod
                ▼

spidev_remove(spi)
│
├── mutex_lock(&device_list_lock)
├── mutex_lock(&spidev->spi_lock)
├── spidev->spi = NULL  ← أي ioctl جديد هيلاقي -ESHUTDOWN
├── mutex_unlock(&spidev->spi_lock)
├── list_del(&spidev->device_entry)
├── device_destroy(...)  ← udev يمسح /dev/spidev0.0
├── clear_bit(minor, minors)
├── لو users == 0: kfree(spidev)
│     لو users > 0: مش هيتحذف — spidev_release هيحذفه
└── mutex_unlock(&device_list_lock)

                │
                │ module unload
                ▼

spidev_exit()
│
├── spi_unregister_driver(...)
├── class_unregister(...)
└── unregister_chrdev(153, ...)
```

---

### Call Flow Diagrams

#### read() — Half-Duplex Read

```
userspace: read(fd, buf, count)
  │
  ▼
spidev_read(filp, buf, count)
  ├── count > bufsiz? → return -EMSGSIZE
  ├── spidev = filp->private_data
  ├── mutex_lock(&spidev->buf_lock)
  │
  ├── spidev_sync_read(spidev, count)
  │     ├── spi_transfer t = { .rx_buf = spidev->rx_buffer, .len = count }
  │     ├── spi_message_init(&m)
  │     ├── spi_message_add_tail(&t, &m)
  │     └── spidev_sync(spidev, &m)
  │           ├── mutex_lock(&spidev->spi_lock)
  │           ├── spi == NULL? → -ESHUTDOWN
  │           ├── spi_sync(spi, message)   ◄── blocking call للـ core
  │           │     └── controller->transfer_one_message()
  │           │           └── hardware registers
  │           ├── status = message->actual_length
  │           └── mutex_unlock(&spidev->spi_lock)
  │
  ├── copy_to_user(buf, spidev->rx_buffer, status)
  └── mutex_unlock(&spidev->buf_lock)
```

#### write() — Half-Duplex Write

```
userspace: write(fd, buf, count)
  │
  ▼
spidev_write(filp, buf, count)
  ├── mutex_lock(&spidev->buf_lock)
  ├── copy_from_user(spidev->tx_buffer, buf, count)
  ├── spidev_sync_write(spidev, count)
  │     ├── spi_transfer t = { .tx_buf = spidev->tx_buffer, .len = count }
  │     └── [نفس مسار spidev_sync]
  └── mutex_unlock(&spidev->buf_lock)
```

#### ioctl(SPI_IOC_MESSAGE(N)) — Full-Duplex Transfer

```
userspace: ioctl(fd, SPI_IOC_MESSAGE(N), mesg_array)
  │
  ▼
spidev_ioctl(filp, cmd, arg)
  ├── _IOC_TYPE(cmd) == 'k'?  (SPI_IOC_MAGIC)
  ├── mutex_lock(&spidev->spi_lock)
  ├── spi = spi_dev_get(spidev->spi)  ← يزود الـ refcount
  ├── spi == NULL? → mutex_unlock + return -ESHUTDOWN
  ├── mutex_lock(&spidev->buf_lock)
  │
  └── switch(cmd): default branch:
        │
        ├── spidev_get_ioc_message(cmd, u_ioc, &n_ioc)
        │     ├── تحقق من type/direction/magic
        │     ├── حساب n_ioc = IOC_SIZE / sizeof(spi_ioc_transfer)
        │     └── memdup_user(u_ioc, size)  ← copy من userspace
        │
        └── spidev_message(spidev, ioc, n_ioc)
              ├── spi_message_init(&msg)
              ├── kcalloc(n_xfers, sizeof(spi_transfer))
              │
              ├── [loop على كل transfer]:
              │     ├── حساب len_aligned = ALIGN(len, ARCH_DMA_MINALIGN)
              │     ├── rx_buf → k_tmp->rx_buf = rx_buf; rx_buf += len_aligned
              │     ├── tx_buf → copy_from_user(); tx_buf += len_aligned
              │     ├── نسخ باقي الـ fields (cs_change, speed_hz, ...)
              │     └── spi_message_add_tail(k_tmp, &msg)
              │
              ├── spidev_sync_unlocked(spidev->spi, &msg)
              │     └── spi_sync(spi, message)
              │           └── spi_controller queue
              │                 └── hardware
              │
              ├── [loop تاني — copy RX data]:
              │     └── copy_to_user(u_tmp->rx_buf, k_tmp->rx_buf, len)
              │
              ├── kfree(k_xfers)
              └── return total bytes
        │
  ├── mutex_unlock(&spidev->buf_lock)
  ├── spi_dev_put(spi)  ← ينزل الـ refcount
  └── mutex_unlock(&spidev->spi_lock)
```

#### ioctl(SPI_IOC_WR_MODE) — تغيير الـ Mode

```
userspace: ioctl(fd, SPI_IOC_WR_MODE, &mode_byte)
  │
  ▼
spidev_ioctl → case SPI_IOC_WR_MODE:
  ├── mutex_lock(&spi_lock)  [مش الـ buf_lock هنا — هو جزء منها]
  ├── mutex_lock(&buf_lock)  ← يوقف أي I/O
  ├── get_user(tmp, (u8*)arg)
  ├── tmp & ~SPI_MODE_MASK? → -EINVAL
  ├── save = spi->mode
  ├── spi->mode = (tmp | (spi->mode & ~SPI_MODE_MASK)) & SPI_MODE_USER_MASK
  ├── spi_setup(spi)  ← يبعت الإعدادات للـ controller
  │     └── controller->setup(spi)
  ├── فشل؟ → spi->mode = save  (rollback)
  ├── mutex_unlock(&buf_lock)
  └── mutex_unlock(&spi_lock)
```

#### compat ioctl (32-bit userspace على 64-bit kernel)

```
32-bit process: ioctl(fd, SPI_IOC_MESSAGE(2), mesg)
  │
  ▼
spidev_compat_ioctl(filp, cmd, arg)
  ├── هل ده SPI_IOC_MESSAGE؟
  │     YES → spidev_compat_ioc_message()
  │             ├── u_ioc = compat_ptr(arg)  ← تحويل 32-bit pointer
  │             ├── memdup_user()
  │             ├── [loop] ioc[n].rx_buf = compat_ptr(ioc[n].rx_buf)
  │             │          ioc[n].tx_buf = compat_ptr(ioc[n].tx_buf)
  │             └── spidev_message()  ← نفس الـ normal path
  │
  └── NO → spidev_ioctl(filp, cmd, compat_ptr(arg))
```

---

### Locking Strategy

#### الـ Locks الموجودة

| Lock | النوع | الموقع | اللي بيحمايه |
|------|-------|--------|------------|
| `device_list_lock` | `DEFINE_MUTEX` | global في `spidev.c` | الـ `device_list` وعمليات الـ open/probe/remove |
| `spidev->spi_lock` | `struct mutex` | داخل `spidev_data` | الـ `spidev->spi` pointer (منع الـ NULL dereference) |
| `spidev->buf_lock` | `struct mutex` | داخل `spidev_data` | الـ TX/RX buffers وكل I/O operations |

#### ترتيب الـ Locking (Lock Ordering)

لازم **دايماً** تاخد الـ locks بالترتيب ده عشان تتجنب deadlock:

```
1. device_list_lock    (الأعلى)
2. spidev->spi_lock
3. spidev->buf_lock    (الأدنى)
```

في `spidev_ioctl()` الترتيب هو:
```
lock(spi_lock) → lock(buf_lock) → ... → unlock(buf_lock) → unlock(spi_lock)
```

في `spidev_release()`:
```
lock(device_list_lock) → lock(spi_lock) → unlock(spi_lock) → ... → unlock(device_list_lock)
```

#### سيناريو الـ Hot Removal أثناء I/O

```
Thread A (ioctl جاري)         Thread B (device removal)
─────────────────────         ──────────────────────────
lock(spi_lock)
spi = spi_dev_get(spi)
lock(buf_lock)
... I/O يجري ...
                              mutex_lock(device_list_lock)
                              mutex_lock(spi_lock)  ← ينتظر Thread A
                              [blocked]
... I/O يخلص ...
unlock(buf_lock)
spi_dev_put(spi)
unlock(spi_lock)              ← Thread B يكمل
                              spidev->spi = NULL
                              unlock(spi_lock)
                              device_destroy(...)
                              unlock(device_list_lock)
```

**النتيجة:** أي `ioctl` جديد بعد الـ removal هيلاقي `spidev->spi == NULL` ويرجع `-ESHUTDOWN` مباشرة. الـ I/O اللي كان جاري خلص بأمان لأن `spi_dev_get()` رفع الـ refcount.

#### الـ `buf_lock` وظيفته الثلاثية

من كومنت في الكود:

```
/* use the buffer lock here for triple duty:
 *  - prevent I/O (from us) so calling spi_setup() is safe
 *  - prevent concurrent SPI_IOC_WR_* from morphing data fields
 *    while SPI_IOC_RD_* reads them
 *  - SPI_IOC_MESSAGE needs the buffer locked "normally"
 */
```

يعني الـ `buf_lock` بيحمي:
1. **الـ TX/RX bounce buffers** من concurrent access.
2. **الـ `spi->mode` وباقي settings** أثناء read/write.
3. **الـ spi_setup() calls** — بيضمن مفيش I/O جاري وهو بيغير الـ config.

#### Bounce Buffer وConcurrency

الـ tx/rx buffers مشتركة بين كل الـ file descriptors على نفس الـ device. لو اتنين process فتحوا `/dev/spidev0.0` في نفس الوقت:

```
Process A (read)            Process B (write)
────────────────            ─────────────────
lock(buf_lock)
copy_from_hw → rx_buffer
copy_to_user(...)
unlock(buf_lock)
                            lock(buf_lock)  ← ينتظر
                            copy_from_user → tx_buffer
                            send_to_hw(...)
                            unlock(buf_lock)
```

الـ `buf_lock` يضمن إن الـ bounce buffers ما تتلوطش.

---

### الـ Minor Numbers Management

```c
static DECLARE_BITMAP(minors, N_SPI_MINORS);  /* N_SPI_MINORS = 32 */
```

```
minors bitmap (32 bits):
bit 0 → spidev0.0 → /dev/spidev0.0
bit 1 → spidev0.1 → /dev/spidev0.1
bit 2 → spidev1.0 → /dev/spidev1.0
...
bit 31 → آخر device

probe:   find_first_zero_bit() → set_bit()
remove:  clear_bit()
```

الـ major number ثابت = **153** (assigned في LANANA). الـ minor dynamic.

---

### المعادلة الكاملة: من ioctl للـ Hardware

```
userspace app
    │
    │  ioctl(fd, SPI_IOC_MESSAGE(2), transfers)
    ▼
spidev_ioctl()
    │  [spi_lock + buf_lock]
    │  spidev_get_ioc_message() → memdup_user()
    ▼
spidev_message()
    │  kcalloc(spi_transfer array)
    │  copy_from_user(tx_buf → spidev->tx_buffer)
    │  spi_message_add_tail() × N
    ▼
spidev_sync_unlocked()
    │  [NO extra lock — caller holds spi_lock]
    ▼
spi_sync()    [kernel SPI core]
    │  adds to controller queue
    ▼
spi_controller->transfer_one_message()
    │  set_cs(active)
    │  للكل transfer: transfer_one(spi_transfer)
    │  set_cs(inactive)
    ▼
Hardware SPI registers (FIFO/DMA)
    │
    │ [interrupt أو polling]
    ▼
spi_finalize_current_message()
    │
    ▼
spi_sync() returns
    │
    ▼
spidev_message() → copy_to_user(rx data)
    │
    ▼
ioctl() returns total bytes transferred
```
## Phase 4: شرح الـ Functions

---

### ملخص عام — Cheatsheet

#### الـ File Operations (Userspace API)

| الـ Function | النوع | الوظيفة |
|---|---|---|
| `spidev_open` | `file_operations.open` | فتح الـ `/dev/spidevB.C`، تخصيص الـ TX/RX buffers |
| `spidev_release` | `file_operations.release` | إغلاق الـ fd، تحرير الـ buffers عند آخر مستخدم |
| `spidev_read` | `file_operations.read` | half-duplex RX من الـ SPI device |
| `spidev_write` | `file_operations.write` | half-duplex TX إلى الـ SPI device |
| `spidev_ioctl` | `file_operations.unlocked_ioctl` | التحكم في الإعدادات + full-duplex transfers |
| `spidev_compat_ioctl` | `file_operations.compat_ioctl` | نفس الـ ioctl لكن من 32-bit userspace على 64-bit kernel |
| `spidev_compat_ioc_message` | داخلية | معالجة `SPI_IOC_MESSAGE` في الـ compat path |

#### الـ Transfer Helpers

| الـ Function | الوظيفة |
|---|---|
| `spidev_sync` | wrapper حول `spi_sync` مع lock على الـ spi pointer |
| `spidev_sync_unlocked` | يستدعي `spi_sync` مباشرةً ويرجع `actual_length` |
| `spidev_sync_write` | يبني `spi_transfer` للـ TX ويستدعي `spidev_sync` |
| `spidev_sync_read` | يبني `spi_transfer` للـ RX ويستدعي `spidev_sync` |
| `spidev_message` | يحوّل مصفوفة `spi_ioc_transfer` من userspace إلى `spi_message` وينفّذها |
| `spidev_get_ioc_message` | validates الـ ioctl cmd ويعمل `memdup_user` للـ transfer array |

#### الـ Driver Lifecycle

| الـ Function | الوظيفة |
|---|---|
| `spidev_probe` | يُنشئ الـ `spidev_data`، يُخصص minor number، يُنشئ الـ device node |
| `spidev_remove` | يُلغي الـ device node، يُحرر الـ `spidev_data` إن لم يكن هناك users |
| `spidev_init` | يُسجّل الـ chrdev + class + spi_driver عند load الـ module |
| `spidev_exit` | يُلغي التسجيل بالترتيب العكسي عند unload الـ module |

#### الـ Validation Callbacks

| الـ Function | الوظيفة |
|---|---|
| `spidev_of_check` | يرفض أي DT node يستخدم `compatible = "spidev"` مباشرةً |
| `spidev_acpi_check` | يطبع warning لأن الـ ACPI entries للـ development فقط |

---

### Group 1: Module Initialization & Cleanup

هذه المجموعة مسؤولة عن تسجيل الـ driver في الـ kernel وإنشاء البنية التحتية اللازمة (character device + sysfs class) قبل أن يبدأ أي probe.

---

#### `spidev_init`

```c
static int __init spidev_init(void)
```

**الـ function دي بتعمل إيه:**
بتُسجّل الـ character device على الـ major number 153 المحجوز لـ SPI، وبعدين بتُسجّل الـ `spidev_class` عشان الـ udev/mdev يقدر يُنشئ الـ `/dev` nodes أوتوماتيك، وأخيراً بتُسجّل الـ `spi_driver` اللي هيتولى الـ probe/remove.

**ترتيب التسجيل مهم جداً:**
```
register_chrdev(153, "spi", &spidev_fops)
    ↓ نجح
class_register(&spidev_class)
    ↓ نجح
spi_register_driver(&spidev_spi_driver)
    ↓ فشل → unregister class + chrdev
```

**الـ Return:** `0` عند النجاح، error code سالب عند الفشل.

**الـ Error Path:** عند فشل `class_register`، بيعمل `unregister_chrdev` مباشرةً. عند فشل `spi_register_driver`، بيعمل `class_unregister` ثم `unregister_chrdev`. الترتيب العكسي ضروري لتجنب resource leaks.

---

#### `spidev_exit`

```c
static void __exit spidev_exit(void)
```

**الـ function دي بتعمل إيه:**
بتُلغي تسجيل الـ `spi_driver` الأول (عشان يحصل remove لكل الـ devices المرتبطة)، ثم الـ class، ثم الـ chrdev. الترتيب العكسي لـ `spidev_init` ضروري لضمان عدم وجود dangling references.

**ملاحظة:** الـ `spi_unregister_driver` هيُطلق `spidev_remove` على كل device مربوطة قبل ما يرجع، يعني عند دخول `class_unregister` كل الـ device nodes بتكون اتمسحت بالفعل.

---

### Group 2: Driver Probe & Remove

هذه المجموعة هي نقطة الربط بين الـ SPI device (المُعرَّف في DT/ACPI/board-info) والـ character device في `/dev`.

---

#### `spidev_probe`

```c
static int spidev_probe(struct spi_device *spi)
```

**الـ function دي بتعمل إيه:**
بتُنشئ `spidev_data` جديدة لكل SPI device يُرتبط بيها الـ driver، وبتُخصص لها minor number من الـ `minors` bitmap، وبتُنشئ الـ character device node في `/dev/spidevB.C`.

**الـ Parameters:**
- `spi`: الـ `spi_device` اللي الـ SPI core سلّمه بعد match الـ device ID.

**الـ Return:** `0` عند النجاح، أو:
- نتيجة الـ `match()` callback لو فشلت (مثلاً `-EINVAL` من `spidev_of_check`)
- `-ENOMEM` لو `kzalloc` فشل
- `-ENODEV` لو مفيش minor numbers متاحة (الـ bitmap ممتلئة)
- error من `device_create`

**الـ Flow الداخلي:**

```c
// 1. تحقق من الـ match data (DT/ACPI check callbacks)
match = device_get_match_data(&spi->dev);
if (match) { status = match(&spi->dev); ... }

// 2. تخصيص spidev_data
spidev = kzalloc(sizeof(*spidev), GFP_KERNEL);

// 3. تهيئة الـ fields + mutexes
spidev->spi = spi;
mutex_init(&spidev->spi_lock);
mutex_init(&spidev->buf_lock);

// 4. تحت device_list_lock: إيجاد minor number
minor = find_first_zero_bit(minors, N_SPI_MINORS);
spidev->devt = MKDEV(SPIDEV_MAJOR, minor);

// 5. إنشاء الـ device node
dev = device_create(&spidev_class, &spi->dev, spidev->devt,
                    spidev, "spidev%d.%d", bus_num, chipselect);

// 6. تسجيل في الـ global list
set_bit(minor, minors);
list_add(&spidev->device_entry, &device_list);

// 7. حفظ الـ driver data
spi_set_drvdata(spi, spidev);
spidev->speed_hz = spi->max_speed_hz;
```

**الـ Locking:** كل العمليات على `minors` bitmap و`device_list` تحت `device_list_lock`. الـ `spi_lock` و`buf_lock` بيتعملوا `init` هنا لكن مش بيتاخدوا.

**الـ TX/RX Buffers:** مش بيتخصصوا هنا. التخصيص الـ lazy في `spidev_open` عشان نوفر memory لو الـ device مش بتتفتحش.

---

#### `spidev_remove`

```c
static void spidev_remove(struct spi_device *spi)
```

**الـ function دي بتعمل إيه:**
بتُزيل الـ device node من `/dev`، وبتعمل NULL للـ `spidev->spi` pointer عشان أي operations جاية تُرجع `-ESHUTDOWN`، وبتُحرر الـ `spidev_data` لو مفيش file descriptors مفتوحة.

**الـ Parameters:**
- `spi`: الـ `spi_device` اللي الـ SPI core جاي يُزيله.

**الـ Locking Pattern الخاص:**
```c
mutex_lock(&device_list_lock);     // يمنع open() جديدة
mutex_lock(&spidev->spi_lock);     // يضمن atomicity للـ NULL assignment
spidev->spi = NULL;                // الـ racing ioctls/reads/writes هتشوف NULL
mutex_unlock(&spidev->spi_lock);
// ... بعد كده:
if (spidev->users == 0)
    kfree(spidev);                 // آمن لأن مفيش open fds
// لو users > 0، الـ spidev_release هيعمل kfree
```

**النقطة الحرجة:** الـ `spidev->spi = NULL` تحت الـ `spi_lock` يضمن أن أي thread حاليًا في `spidev_sync` وهيأخذ الـ lock هيلاقي NULL ويرجع `-ESHUTDOWN` بدل ما يستخدم pointer لـ device اتمسحت.

---

### Group 3: File Operations — Open & Release

---

#### `spidev_open`

```c
static int spidev_open(struct inode *inode, struct file *filp)
```

**الـ function دي بتعمل إيه:**
بتبحث في الـ `device_list` العالمية عن الـ `spidev_data` المقابلة للـ minor number، وبتُخصص الـ TX/RX bounce buffers عند أول فتح، وبتزيد عداد الـ `users`.

**الـ Parameters:**
- `inode`: بيحتوي على الـ `i_rdev` (major:minor) اللي بنستخدمه للبحث.
- `filp`: بنضع فيه `private_data = spidev` للاستخدام في باقي الـ operations.

**الـ Return:** `0` عند النجاح، أو:
- `-ENXIO` لو مفيش device بيطابق الـ minor number
- `-ENOMEM` لو `kmalloc` فشل في تخصيص أي من الـ buffers

**الـ Buffer Allocation:**

```c
// Lazy allocation: TX buffer
if (!spidev->tx_buffer) {
    spidev->tx_buffer = kmalloc(bufsiz, GFP_KERNEL);
    // bufsiz default = 4096 bytes، قابل للتعديل بـ module param
}

// RX buffer، مع cleanup لو فشل
if (!spidev->rx_buffer) {
    spidev->rx_buffer = kmalloc(bufsiz, GFP_KERNEL);
    // لو فشل: kfree(tx_buffer) ثم error
}
```

**الـ `stream_open`:** بيُعطّل الـ `f_pos` tracking (الـ SPI مش seekable). يضمن أن multiple concurrent reads/writes مش بتتشوّه بأي offset logic.

**الـ Sharing:** عدة processes ممكن يفتحوا نفس الـ device في نفس الوقت. الـ `users` counter بيتزيد لكل `open`. الـ buffers مشتركة بين الـ users ومحمية بـ `buf_lock`.

---

#### `spidev_release`

```c
static int spidev_release(struct inode *inode, struct file *filp)
```

**الـ function دي بتعمل إيه:**
بتنقص عداد الـ `users`، وعند الوصول لـ صفر بتُحرر الـ TX/RX buffers، وبتتحقق لو الـ SPI device اتمسحت (أثناء فترة الفتح) فتُحرر الـ `spidev_data` كمان.

**الـ Double-Free Protection:**
```c
mutex_lock(&spidev->spi_lock);
dofree = (spidev->spi == NULL);  // الـ device اتمسحت؟
mutex_unlock(&spidev->spi_lock);

spidev->users--;
if (!spidev->users) {
    kfree(spidev->tx_buffer);
    kfree(spidev->rx_buffer);
    if (dofree)
        kfree(spidev);           // spidev_remove ما قدرش يحررها
    else
        spidev->speed_hz = spidev->spi->max_speed_hz; // reset speed
}
```

**الـ `spi_target_abort`:** في حالة `CONFIG_SPI_SLAVE`، بيُلغي أي transfer جاري على الـ slave device عند الإغلاق.

---

### Group 4: Half-Duplex I/O (read/write)

هذه المجموعة تُغطي الـ POSIX read/write API. كل عملية هي transfer وحيدة (single `spi_transfer` داخل `spi_message` واحدة). الـ chipselect بيتفعّل في بداية الـ `spi_sync` ويتعطّل في نهايتها.

---

#### `spidev_read`

```c
static ssize_t
spidev_read(struct file *filp, char __user *buf, size_t count, loff_t *f_pos)
```

**الـ function دي بتعمل إيه:**
بتستقبل `count` bytes من الـ SPI device إلى الـ kernel bounce buffer (`rx_buffer`)، ثم بتعمل `copy_to_user` لنقلها لـ userspace.

**الـ Parameters:**
- `filp`: الـ `private_data` بيحتوي على `spidev_data`.
- `buf`: الـ userspace buffer الوجهة.
- `count`: عدد الـ bytes المطلوب قراءتها.
- `f_pos`: متجاهل (stream device).

**الـ Return:** عدد الـ bytes المقروءة فعلاً عند النجاح، أو:
- `-EMSGSIZE` لو `count > bufsiz`
- `-EFAULT` لو `copy_to_user` فشل كلياً

**الـ Partial Copy:**
```c
missing = copy_to_user(buf, spidev->rx_buffer, status);
if (missing == status)
    status = -EFAULT;            // ما نقلنا ولا بايت
else
    status = status - missing;   // نقلنا جزء منهم
```

**الـ TX:** مش بيبعت حاجة (الـ `tx_buf = NULL` في الـ transfer)، لكن في SPI الـ hardware ممكن يبعت zeros أو يُعيد الـ MOSI line في حالة idle.

---

#### `spidev_write`

```c
static ssize_t
spidev_write(struct file *filp, const char __user *buf,
             size_t count, loff_t *f_pos)
```

**الـ function دي بتعمل إيه:**
بتنقل `count` bytes من userspace إلى الـ `tx_buffer`، ثم بتبعتها عبر الـ SPI bus. الـ RX data أثناء الإرسال بيتاهل (الـ `rx_buf = NULL`).

**الـ Parameters:**
- `buf`: الـ userspace buffer المصدر.
- `count`: عدد الـ bytes المراد إرسالها.

**الـ Return:** عدد الـ bytes المُرسَلة فعلاً عند النجاح، أو `-EMSGSIZE`/`-EFAULT`.

**الـ Copy-Then-Send:**
```c
mutex_lock(&spidev->buf_lock);
missing = copy_from_user(spidev->tx_buffer, buf, count);
if (missing == 0)
    status = spidev_sync_write(spidev, count);
else
    status = -EFAULT;
mutex_unlock(&spidev->buf_lock);
```

**ملاحظة:** الـ `buf_lock` بيُغطي كل من الـ copy وإرسال الـ transfer عشان يمنع أي thread تاني من تعديل الـ `tx_buffer` في نفس الوقت.

---

### Group 5: Transfer Execution Pipeline

هذه المجموعة تُمثّل الطبقة الداخلية التي تبني وتُنفّذ الـ `spi_message`.

---

#### `spidev_sync`

```c
static ssize_t
spidev_sync(struct spidev_data *spidev, struct spi_message *message)
```

**الـ function دي بتعمل إيه:**
بتأخذ الـ `spi_lock` للتحقق من أن الـ `spi_device` pointer ما زال صالحاً (لم يُحذف بـ `spidev_remove`)، ثم بتُنفّذ الـ message.

**الـ Parameters:**
- `spidev`: الـ driver state.
- `message`: الـ message المُعدّة مسبقاً بالـ transfers.

**الـ Return:** عدد الـ bytes المُنقلة فعلاً، أو `-ESHUTDOWN` لو الـ device اتحذفت.

**الـ Race Protection:**
```c
mutex_lock(&spidev->spi_lock);
spi = spidev->spi;
if (spi == NULL)
    status = -ESHUTDOWN;  // spidev_remove سبقنا
else
    status = spidev_sync_unlocked(spi, message);
mutex_unlock(&spidev->spi_lock);
```

**لماذا نأخذ local copy من `spi`؟** لأن `spidev->spi` ممكن يبقى NULL في أي لحظة من thread تاني. بعد ما نأخذ الـ pointer تحت الـ lock، الـ SPI core مش هيقدر يُحذف الـ device إلا بعد ما نخلص من الـ `spi_sync`.

---

#### `spidev_sync_unlocked`

```c
static ssize_t
spidev_sync_unlocked(struct spi_device *spi, struct spi_message *message)
```

**الـ function دي بتعمل إيه:**
Thin wrapper حول `spi_sync`. بيُحوّل return value من `0` إلى `message->actual_length` عشان الـ callers يقدروا يعرفوا كام byte اتنقل فعلاً.

**ملاحظة:** `spi_sync` بتُرجع `0` عند النجاح بغض النظر عن عدد الـ bytes. الـ `actual_length` بيتحدد من الـ SPI controller بعد اكتمال الـ transfer.

---

#### `spidev_sync_write`

```c
static inline ssize_t
spidev_sync_write(struct spidev_data *spidev, size_t len)
```

**الـ function دي بتعمل إيه:**
بتبني `spi_transfer` TX-only من الـ `tx_buffer` وبتُنفّذها عبر `spidev_sync`.

```c
struct spi_transfer t = {
    .tx_buf   = spidev->tx_buffer,  // الـ bounce buffer المحمّل من userspace
    .len      = len,
    .speed_hz = spidev->speed_hz,   // الـ speed المحفوظة في الـ state (0 = default)
};
```

---

#### `spidev_sync_read`

```c
static inline ssize_t
spidev_sync_read(struct spidev_data *spidev, size_t len)
```

**الـ function دي بتعمل إيه:**
نفس `spidev_sync_write` لكن RX-only. بتبني `spi_transfer` فيها `rx_buf` فقط وبتُنفّذها.

---

### Group 6: Full-Duplex ioctl Pipeline

هذه المجموعة هي قلب الـ `SPI_IOC_MESSAGE(N)` ioctl. بتسمح بإرسال N transfers متتالية في message واحدة مع التحكم الكامل في كل transfer.

---

#### `spidev_get_ioc_message`

```c
static struct spi_ioc_transfer *
spidev_get_ioc_message(unsigned int cmd, struct spi_ioc_transfer __user *u_ioc,
                       unsigned *n_ioc)
```

**الـ function دي بتعمل إيه:**
بتتحقق من صحة الـ ioctl command (النوع والرقم والاتجاه)، وبتستخرج عدد الـ transfers من حجم الـ ioctl، وبتعمل `memdup_user` لنسخ الـ array كلها من userspace إلى kernel memory.

**الـ Parameters:**
- `cmd`: الـ ioctl command الكامل (بيتضمن الـ size في الـ bits العليا).
- `u_ioc`: الـ userspace pointer لمصفوفة الـ `spi_ioc_transfer`.
- `n_ioc`: output — عدد الـ transfers في المصفوفة.

**الـ Return:** pointer لـ kernel copy من المصفوفة، أو:
- `ERR_PTR(-ENOTTY)` لو الـ cmd مش SPI_IOC_MESSAGE
- `ERR_PTR(-EINVAL)` لو الحجم مش مضاعف لـ `sizeof(spi_ioc_transfer)`
- `NULL` لو `n_ioc == 0` (message فاضية، مقبول)
- error من `memdup_user` (مثلاً `-ENOMEM`, `-EFAULT`)

**الـ Validation Logic:**
```c
// _IOC_SIZE(cmd) = N * sizeof(spi_ioc_transfer)
// هو الطريقة الوحيدة لمعرفة N من الـ ioctl
tmp = _IOC_SIZE(cmd);
if ((tmp % sizeof(struct spi_ioc_transfer)) != 0)
    return ERR_PTR(-EINVAL);
*n_ioc = tmp / sizeof(struct spi_ioc_transfer);
```

**الـ `memdup_user`:** بيعمل `kmalloc` + `copy_from_user` في خطوة واحدة. الـ caller مسؤول عن `kfree` لاحقاً.

---

#### `spidev_message`

```c
static int spidev_message(struct spidev_data *spidev,
                          struct spi_ioc_transfer *u_xfers, unsigned n_xfers)
```

**الـ function دي بتعمل إيه:**
بتُحوّل مصفوفة `spi_ioc_transfer` (القادمة من userspace) إلى `spi_message` حقيقية مع `spi_transfer` array في kernel space، مع نسخ الـ TX data من userspace إلى الـ bounce buffer ومعالجة الـ DMA alignment.

**الـ Parameters:**
- `spidev`: الـ driver state (للوصول لـ `tx_buffer`, `rx_buffer`, `speed_hz`).
- `u_xfers`: الـ array المنسوخة من userspace (في kernel space بالفعل بعد `spidev_get_ioc_message`).
- `n_xfers`: عدد الـ transfers.

**الـ Return:** مجموع الـ bytes في كل الـ transfers عند النجاح، أو error code سالب.

**الـ Flow الكامل:**

```c
// 1. تخصيص kernel spi_transfer array
k_xfers = kcalloc(n_xfers, sizeof(*k_tmp), GFP_KERNEL);

// 2. Loop على كل transfer
for each (u_tmp, k_tmp):
    len_aligned = ALIGN(u_tmp->len, ARCH_DMA_MINALIGN);

    // حساب الـ totals والتحقق من تجاوز الـ bufsiz
    total += k_tmp->len;
    if (total > INT_MAX) → -EMSGSIZE

    // RX: تخصيص جزء من rx_buffer
    if (u_tmp->rx_buf):
        rx_total += len_aligned
        if (rx_total > bufsiz) → -EMSGSIZE
        k_tmp->rx_buf = rx_buf; rx_buf += len_aligned;

    // TX: نسخ من userspace إلى tx_buffer
    if (u_tmp->tx_buf):
        tx_total += len_aligned
        if (tx_total > bufsiz) → -EMSGSIZE
        copy_from_user(tx_buf, u_tmp->tx_buf, u_tmp->len)
        k_tmp->tx_buf = tx_buf; tx_buf += len_aligned;

    // نسخ الإعدادات الأخرى
    k_tmp->cs_change, tx_nbits, rx_nbits, bits_per_word,
    delay (usecs), speed_hz, word_delay

    spi_message_add_tail(k_tmp, &msg);

// 3. تنفيذ الـ message
status = spidev_sync_unlocked(spidev->spi, &msg);

// 4. نسخ RX data إلى userspace
for each (u_tmp, k_tmp):
    if (u_tmp->rx_buf):
        copy_to_user(u_tmp->rx_buf, k_tmp->rx_buf, u_tmp->len)

// 5. تحرير k_xfers
kfree(k_xfers);
return total;
```

**الـ DMA Alignment:** كل transfer تأخذ `ARCH_DMA_MINALIGN`-aligned slice من الـ bounce buffers عشان الـ DMA controller يقدر يشتغل عليها مباشرةً بدون extra copies.

**الـ `cs_change`:** لو `u_tmp->cs_change = 1`، الـ chipselect بيتعطّل ويتفعّل من جديد بين الـ transfers — مفيد لبروتوكولات الـ RPC اللي محتاجة separator بين الـ command والـ response.

**الـ Speed per Transfer:** لو `u_tmp->speed_hz = 0`، بيستخدم `spidev->speed_hz` كـ default. ده بيسمح بتغيير الـ clock في منتصف الـ message.

**ملاحظة الـ Locking:** الـ function دي بتُنفَّذ تحت `spidev->buf_lock` (اللي بيأخذه الـ caller). `spidev_sync_unlocked` بيستخدم الـ spi pointer مباشرةً لأن الـ ioctl handler بالفعل أخذ `spi_lock` وعمل `spi_dev_get`.

---

### Group 7: ioctl Dispatcher

---

#### `spidev_ioctl`

```c
static long
spidev_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
```

**الـ function دي بتعمل إيه:**
الـ main ioctl handler. بتُعالج كل الـ SPI_IOC_* commands: القراءة والكتابة لإعدادات الـ mode/speed/bits، وتنفيذ الـ full-duplex messages عبر `SPI_IOC_MESSAGE(N)`.

**الـ Locking Pattern:**
```c
mutex_lock(&spidev->spi_lock);      // حماية من spidev_remove
spi = spi_dev_get(spidev->spi);    // زيادة refcount
// ...
mutex_lock(&spidev->buf_lock);      // حماية الـ buffers + spi_setup safety

// ... معالجة الـ cmd ...

mutex_unlock(&spidev->buf_lock);
spi_dev_put(spi);                   // تنقيص الـ refcount
mutex_unlock(&spidev->spi_lock);
```

**الـ `buf_lock` بيحمي ثلاث أشياء في وقت واحد:**
1. يمنع الـ I/O أثناء استدعاء `spi_setup` (اللي بيغيّر إعدادات الـ controller)
2. يمنع الـ concurrent SPI_IOC_WR_* من تعديل الـ fields أثناء SPI_IOC_RD_*
3. يحمي الـ bounce buffers للـ `SPI_IOC_MESSAGE`

**جدول الـ Commands:**

| Command | الاتجاه | الـ Action |
|---|---|---|
| `SPI_IOC_RD_MODE` | ← kernel | `put_user(spi->mode & MASK, u8 *)` |
| `SPI_IOC_RD_MODE32` | ← kernel | `put_user(spi->mode & MASK, u32 *)` |
| `SPI_IOC_RD_LSB_FIRST` | ← kernel | `put_user(LSB_FIRST bit, u8 *)` |
| `SPI_IOC_RD_BITS_PER_WORD` | ← kernel | `put_user(spi->bits_per_word, u8 *)` |
| `SPI_IOC_RD_MAX_SPEED_HZ` | ← kernel | `put_user(spidev->speed_hz, u32 *)` |
| `SPI_IOC_WR_MODE` | → kernel | `get_user` + `spi_setup` |
| `SPI_IOC_WR_MODE32` | → kernel | `get_user` + `spi_setup` |
| `SPI_IOC_WR_LSB_FIRST` | → kernel | toggle `SPI_LSB_FIRST` + `spi_setup` |
| `SPI_IOC_WR_BITS_PER_WORD` | → kernel | set `bits_per_word` + `spi_setup` |
| `SPI_IOC_WR_MAX_SPEED_HZ` | → kernel | set `max_speed_hz` + `spi_setup` + restore |
| `SPI_IOC_MESSAGE(N)` | bidirectional | `spidev_get_ioc_message` + `spidev_message` |

**الـ WR_MODE — GPIO CS Handling:**
```c
// لو الـ controller بيستخدم GPIO للـ chipselect:
// userspace مش بتعرف إن الـ hardware بيعكس الـ polarity
// فالـ driver بيضيف SPI_CS_HIGH تلقائياً
if (ctlr->use_gpio_descriptors && spi_get_csgpiod(spi, 0))
    tmp |= SPI_CS_HIGH;
```

**الـ WR_MAX_SPEED_HZ — الـ Save/Restore Pattern:**
```c
save = spi->max_speed_hz;
spi->max_speed_hz = tmp;
retval = spi_setup(spi);
if (retval == 0)
    spidev->speed_hz = tmp;    // احفظ في driver state
spi->max_speed_hz = save;      // دايماً restore الـ spi_device field
```
الـ `speed_hz` في `spidev_data` هو الـ cap للـ userspace. الـ `spi->max_speed_hz` بيرجع لقيمته الأصلية دايماً عشان driver كودات تانية ما تتأثرش.

---

#### `spidev_compat_ioctl`

```c
static long
spidev_compat_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
```

**الـ function دي بتعمل إيه:**
Entry point للـ 32-bit processes على 64-bit kernel. بتُفرّق بين `SPI_IOC_MESSAGE` (محتاجة pointer conversion خاص) وباقي الـ commands (ممكن تتعالج بـ `spidev_ioctl` العادية بعد `compat_ptr`).

```c
if (SPI_IOC_MESSAGE ioctl)
    return spidev_compat_ioc_message(filp, cmd, arg);  // pointer fix
else
    return spidev_ioctl(filp, cmd, (unsigned long)compat_ptr(arg));
```

---

#### `spidev_compat_ioc_message`

```c
static long
spidev_compat_ioc_message(struct file *filp, unsigned int cmd,
                           unsigned long arg)
```

**الـ function دي بتعمل إيه:**
نفس منطق الـ SPI_IOC_MESSAGE في `spidev_ioctl` لكن مع تحويل الـ 32-bit pointers (الـ `rx_buf`/`tx_buf` في `spi_ioc_transfer`) باستخدام `compat_ptr` قبل تمريرها لـ `spidev_message`.

```c
// البـ ioc[n].rx_buf و ioc[n].tx_buf هم 32-bit values في struct
// من 32-bit process، محتاجين sign extension صح على 64-bit
for (n = 0; n < n_ioc; n++) {
    ioc[n].rx_buf = (uintptr_t) compat_ptr(ioc[n].rx_buf);
    ioc[n].tx_buf = (uintptr_t) compat_ptr(ioc[n].tx_buf);
}
```

**لماذا لازم نعمل كده؟** في الـ `spi_ioc_transfer` struct، الـ `rx_buf` و`tx_buf` هم `__u64` دايماً حتى في الـ 32-bit builds. لكن القيم الجاية من 32-bit process بيكونوا 32-bit pointers. الـ `compat_ptr` بيعمل الـ sign extension الصح لتحويل الـ 32-bit address لـ 64-bit kernel address.

---

### Group 8: Validation Callbacks

---

#### `spidev_of_check`

```c
static int spidev_of_check(struct device *dev)
```

**الـ function دي بتعمل إيه:**
بتتحقق من إن الـ DT node مش بيستخدم `compatible = "spidev"` مباشرةً، وترفض الـ probe لو بيعملها.

**السبب:** `compatible = "spidev"` كانت مدعومة قديماً لكن بتعني إن الـ DT مش بيوصف الـ hardware الحقيقي — هو بيوصف الـ driver implementation. ده ضد مبدأ الـ Device Tree. الكود الصحيح هو استخدام `compatible = "vendor,device-name"` حقيقية.

**الـ Return:** `0` (قبول) في كل الأحوال إلا لو `compatible = "spidev"` بالظبط حيث بيُرجع `-EINVAL`.

---

#### `spidev_acpi_check`

```c
static int spidev_acpi_check(struct device *dev)
```

**الـ function دي بتعمل إيه:**
بتطبع `dev_warn` إن الـ ACPI SPT000* devices للـ development والتجربة فقط وليس للإنتاج، وترجع `0` (قبول على أي حال).

**السبب:** الـ systems الإنتاجية المبنية على ACPI المفروض يكون عندها driver حقيقي للـ peripheral المتصل، مش يستخدموا الـ spidev generic driver.

---

### الـ Data Flow الكامل — ASCII Diagram

```
Userspace Process
    │
    ├─ open("/dev/spidevB.C")
    │       └─► spidev_open()
    │               ├─ find spidev_data via device_list (under device_list_lock)
    │               ├─ kmalloc(tx_buffer, bufsiz)
    │               └─ kmalloc(rx_buffer, bufsiz)
    │
    ├─ write(fd, data, len)
    │       └─► spidev_write()
    │               ├─ copy_from_user → tx_buffer   (under buf_lock)
    │               └─► spidev_sync_write()
    │                       └─► spidev_sync()        (under spi_lock)
    │                               └─► spi_sync()  → SPI Controller HW
    │
    ├─ read(fd, buf, len)
    │       └─► spidev_read()
    │               ├─► spidev_sync_read()
    │               │       └─► spidev_sync() → spi_sync() → HW
    │               └─ copy_to_user ← rx_buffer
    │
    ├─ ioctl(fd, SPI_IOC_WR_MODE, &mode)
    │       └─► spidev_ioctl()
    │               ├─ spi_dev_get()                (under spi_lock)
    │               ├─ get_user(mode)               (under buf_lock)
    │               └─ spi_setup(spi)
    │
    ├─ ioctl(fd, SPI_IOC_MESSAGE(N), transfers)
    │       └─► spidev_ioctl()
    │               └─► spidev_get_ioc_message()    → memdup_user
    │                       └─► spidev_message()    (under buf_lock)
    │                               ├─ copy_from_user → tx slices in tx_buffer
    │                               ├─ build spi_transfers with DMA alignment
    │                               ├─► spidev_sync_unlocked() → spi_sync() → HW
    │                               └─ copy_to_user ← rx slices in rx_buffer
    │
    └─ close(fd)
            └─► spidev_release()
                    ├─ users--
                    └─ if (users == 0): kfree(tx_buffer, rx_buffer)
                                        if (spi == NULL): kfree(spidev)
```

---

### الـ Locking Map

| الـ Mutex | من يأخذه | يحمي إيه |
|---|---|---|
| `device_list_lock` (global) | `open`, `release`, `probe`, `remove` | الـ `device_list` و`minors` bitmap |
| `spidev->spi_lock` | `sync`, `ioctl`, `release`, `remove` | الـ `spidev->spi` pointer (NULL check) |
| `spidev->buf_lock` | `read`, `write`, `ioctl` | الـ `tx_buffer`/`rx_buffer` + `spi_setup` safety |

**الـ Lock Ordering:** دايماً `device_list_lock` قبل `spi_lock` قبل `buf_lock`. تعكس الترتيب ده = deadlock محتمل.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. مداخل الـ debugfs المرتبطة بـ spidev

الـ `spidev` نفسه ما بيكتبش مداخل في `debugfs` بشكل مباشر، لكن الـ SPI controller بيعمل كده.

```bash
# mount debugfs لو مش موجود
mount -t debugfs none /sys/kernel/debug

# استعراض مداخل الـ SPI controller
ls /sys/kernel/debug/spi*/
# مثال: /sys/kernel/debug/spi0/

# قراءة آخر transfer على البص
cat /sys/kernel/debug/spi0/statistics
```

| الملف | المعنى |
|---|---|
| `/sys/kernel/debug/spi0/statistics` | عدد الـ transfers، الـ errors، الـ bytes المنقولة |
| `/sys/kernel/debug/regmap/spi0.*` | قراءة/كتابة registers الـ device لو بيستخدم regmap |

---

#### 2. مداخل الـ sysfs للـ spidev

```bash
# الـ device node الأساسي — bus B=0, chipselect C=0
ls -la /dev/spidev0.0
# الناتج: crw-rw---- 1 root spi 153, 0 ...
# major=153 ده SPIDEV_MAJOR الثابت في الكود

# معلومات الـ SPI device من sysfs
ls /sys/bus/spi/devices/spi0.0/
cat /sys/bus/spi/devices/spi0.0/modalias
cat /sys/bus/spi/devices/spi0.0/driver_override

# الـ class node
ls /sys/class/spidev/
# هتلاقي spidev0.0 -> ../../devices/.../spi0.0/spidev/spidev0.0

# الـ dev attribute (يحتوي على major:minor)
cat /sys/class/spidev/spidev0.0/dev
# 153:0

# max_speed_hz الحالية
cat /sys/bus/spi/devices/spi0.0/spi_device_info 2>/dev/null || \
  cat /sys/bus/spi/devices/spi0.0/of_node/spi-max-frequency 2>/dev/null

# binding/unbinding يدوي
echo spidev > /sys/bus/spi/devices/spi0.0/driver_override
echo spi0.0  > /sys/bus/spi/drivers/spidev/bind
echo spi0.0  > /sys/bus/spi/drivers/spidev/unbind
```

---

#### 3. الـ ftrace — tracepoints وأحداث الـ SPI

```bash
# عرض الـ SPI events المتاحة
ls /sys/kernel/debug/tracing/events/spi/

# الـ events المهمة لـ spidev
# spi_transfer_start   — بداية كل transfer
# spi_transfer_stop    — نهاية الـ transfer
# spi_message_start    — بداية الـ message كاملة
# spi_message_done     — انتهاء الـ message + actual_length

# تفعيل الـ tracing لـ SPI كله
echo 1 > /sys/kernel/debug/tracing/events/spi/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on

# تشغيل أمر الاختبار
python3 spi_test.py

# قراءة الناتج
cat /sys/kernel/debug/tracing/trace

# إيقاف + reset
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo > /sys/kernel/debug/tracing/trace

# تتبع function محدد (spidev_sync فقط)
echo function > /sys/kernel/debug/tracing/current_tracer
echo spidev_sync > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

مثال ناتج الـ `trace`:

```
kworker/0:1-42   [000] ....   123.456: spi_message_start: spi0.0
kworker/0:1-42   [000] ....   123.457: spi_transfer_start: spi0.0 len=4 tx=[0xAB 0xCD ...] rx=[-]
kworker/0:1-42   [000] ....   123.458: spi_transfer_stop:  spi0.0 len=4
kworker/0:1-42   [000] ....   123.459: spi_message_done:  spi0.0 actual=4
```

---

#### 4. الـ printk والـ dynamic debug

الـ `spidev.c` بيستخدم `dev_dbg()` — ده مش بيظهر إلا لو فعّلت الـ dynamic debug.

```bash
# تفعيل كل debug messages الخاصة بـ spidev.c
echo "file drivers/spi/spidev.c +p" > /sys/kernel/debug/dynamic_debug/control

# التحقق إنها اتفعّلت
grep spidev /sys/kernel/debug/dynamic_debug/control
# هتشوف: drivers/spi/spidev.c:439 [spidev]spidev_ioctl =p "spi mode %x\n"
#                                                                         ^^ p = enabled

# تفعيل لكل ملفات الـ SPI subsystem
echo "module spidev +p" > /sys/kernel/debug/dynamic_debug/control

# قراءة الرسائل في real-time
dmesg -w | grep spidev

# أو عبر kernel log level
echo 8 > /proc/sys/kernel/printk
```

الرسائل اللي هتظهر بعد التفعيل (من الكود مباشرة):

```
spidev spi0.0: spi mode 3
spidev spi0.0: 8 bits per word
spidev spi0.0: 1000000 Hz (max)
spidev spi0.0: msb first
```

لتفعيل الـ `VERBOSE` (يطبع كل transfer):

```bash
# لازم تعيد compile الـ module بـ VERBOSE مفعّل
# في drivers/spi/Makefile أضف:
# CFLAGS_spidev.o := -DVERBOSE
```

---

#### 5. Kernel config options للـ debugging

| الـ CONFIG | الوظيفة |
|---|---|
| `CONFIG_SPI_DEBUG` | يفعّل debug messages في الـ SPI core |
| `CONFIG_DEBUG_FS` | يتيح /sys/kernel/debug (شرط أساسي) |
| `CONFIG_DYNAMIC_DEBUG` | يتيح `dev_dbg()` بشكل انتقائي |
| `CONFIG_TRACING` | أساس الـ ftrace |
| `CONFIG_SPI_SPIDEV` | الـ driver نفسه (y أو m) |
| `CONFIG_SPI_LOOPBACK_TEST` | اختبار الـ SPI بدون hardware خارجي |
| `CONFIG_FAULT_INJECTION` | حقن أخطاء اصطناعية لاختبار error paths |
| `CONFIG_LOCK_STAT` | إحصائيات الـ mutex لكشف deadlock |
| `CONFIG_PROVE_LOCKING` | يكتشف lockdep violations |

```bash
# التحقق من الـ config الحالية
zcat /proc/config.gz | grep -E "SPI|CONFIG_DEBUG_FS|DYNAMIC_DEBUG"
# أو
grep -E "CONFIG_SPI|CONFIG_DEBUG_FS" /boot/config-$(uname -r)
```

---

#### 6. أدوات الـ subsystem

```bash
# أداة spi-tools لو موجودة
apt-get install spi-tools   # Debian/Ubuntu
spi-config -d /dev/spidev0.0 -q   # query current config

# اختبار أساسي بـ Python
python3 - <<'EOF'
import spidev
spi = spidev.SpiDev()
spi.open(0, 0)
spi.max_speed_hz = 500000
spi.mode = 0
resp = spi.xfer2([0xAB, 0xCD, 0xEF])
print("RX:", [hex(b) for b in resp])
spi.close()
EOF

# اختبار أساسي بـ C (spidev_test من kernel tools)
# يوجد في tools/spi/spidev_test.c
./spidev_test -D /dev/spidev0.0 -s 500000 -v

# قراءة max_speed_hz عبر ioctl بدون كتابة كود
# (ممكن تستخدم devmem2 للـ controller registers — راجع Hardware Level)

# لو بتستخدم systemd — اعرض udev events لـ spidev
udevadm monitor --kernel --subsystem-match=spidev

# لعرض الـ device attributes اللي udev بيشوفها
udevadm info /dev/spidev0.0
```

---

#### 7. جدول رسائل الـ error الشائعة

| رسالة الـ error | المعنى | الحل |
|---|---|---|
| `spidev listed directly in DT is not supported` | الـ DT فيه `compatible = "spidev"` | غيّر لـ compatible حقيقي من `spidev_dt_ids[]` |
| `no minor number available!` | وصل عدد الـ devices لـ 32 (`N_SPI_MINORS`) | قلّل عدد الـ devices أو زيّد `N_SPI_MINORS` واعيد البناء |
| `open: ENXIO (No such device)` | الـ minor number مش موجود في `device_list` | تأكد الـ driver probe نجح: `dmesg | grep spidev` |
| `ioctl: ENOTTY` | الـ ioctl command مش SPI command | تحقق من `SPI_IOC_MAGIC` في header ومن بناء الـ cmd صح |
| `ioctl: EINVAL` | حجم الـ transfer مش مضروب في `sizeof(spi_ioc_transfer)` أو speed=0 | راجع بناء الـ `SPI_IOC_MESSAGE(n)` وتأكد speed_hz != 0 |
| `ioctl: EMSGSIZE` | حجم الـ transfer أكبر من `bufsiz` (default 4096 bytes) | زيّد `bufsiz` بالـ module param أو قسّم الـ transfer |
| `ioctl: ESHUTDOWN` | الـ SPI device اتشال وقت الـ ioctl | تحقق اتصال الـ controller، أعد probe |
| `ioctl: EFAULT` | `copy_from_user` أو `copy_to_user` فشلت | pointer في userspace غلط أو NULL |
| `ioctl: ENOMEM` | `kcalloc` لـ k_xfers فشل | ضغط على الـ memory — راجع `free -h` |
| `do not use this driver in production systems!` | بتستخدم ACPI SPT000x device | للتطوير بس، مش للإنتاج |
| `write: EMSGSIZE` | `count > bufsiz` في `spidev_write` | قلّل الحجم أو زيّد `bufsiz` |
| `read: EFAULT` | فشل `copy_to_user` في `spidev_read` | buffer في userspace غلط |

---

#### 8. نقاط استراتيجية لـ dump_stack() وWARN_ON()

مناطق في الكود تستحق الـ instrumentation:

```c
/* في spidev_sync() — تحقق السبب لو spi == NULL */
if (spi == NULL) {
    WARN_ON(1); /* unexpected shutdown during ioctl */
    dump_stack();
    status = -ESHUTDOWN;
}

/* في spidev_message() — لو الـ transfer size غريب */
WARN_ON(total > bufsiz * 2);

/* في spidev_probe() — لو فاضت الـ minors */
WARN_ON(minor >= N_SPI_MINORS);

/* في spidev_open() — race condition check */
WARN_ON(!mutex_is_locked(&device_list_lock));
```

---

### Hardware Level

#### 1. التحقق إن حالة الـ hardware مطابقة لحالة الـ kernel

```bash
# قراءة الـ mode المبرمج في الـ kernel
python3 -c "
import spidev, struct, fcntl
fd = open('/dev/spidev0.0', 'rb+', buffering=0)
import array
mode = array.array('B', [0])
fcntl.ioctl(fd, 0x80016B01, mode)  # SPI_IOC_RD_MODE
speed = array.array('I', [0])
fcntl.ioctl(fd, 0x80046B04, speed) # SPI_IOC_RD_MAX_SPEED_HZ
bpw = array.array('B', [0])
fcntl.ioctl(fd, 0x80016B03, bpw)   # SPI_IOC_RD_BITS_PER_WORD
print(f'Mode: {mode[0]} (CPOL={(mode[0]>>1)&1} CPHA={mode[0]&1})')
print(f'Speed: {speed[0]} Hz')
print(f'Bits per word: {bpw[0]}')
fd.close()
"

# تأكيد من sysfs
cat /sys/bus/spi/devices/spi0.0/of_node/spi-max-frequency
```

الـ oscilloscope يجب يُظهر:
- **SPI_MODE_0**: CLK idle low، sample on rising edge
- **SPI_MODE_1**: CLK idle low، sample on falling edge
- **SPI_MODE_2**: CLK idle high، sample on falling edge
- **SPI_MODE_3**: CLK idle high، sample on rising edge

---

#### 2. Register Dump بـ devmem2

```bash
# تثبيت devmem2
apt-get install devmem2

# مثال: قراءة base address للـ SPI controller من DT أو datasheet
# RPi SPI0 base: 0x3F204000
devmem2 0x3F204000 w   # CS register
devmem2 0x3F204004 w   # FIFO register
devmem2 0x3F204008 w   # CLK register
devmem2 0x3F20400C w   # DLEN register
devmem2 0x3F204010 w   # LTOH register
devmem2 0x3F204014 w   # DC register

# للـ i.MX8 SPI (ECSPI1 base: 0x30820000)
devmem2 0x30820000 w   # ECSPI_RXDATA
devmem2 0x30820004 w   # ECSPI_TXDATA
devmem2 0x30820008 w   # ECSPI_CONREG
devmem2 0x3082000C w   # ECSPI_CONFIGREG

# عبر /dev/mem مع dd
dd if=/dev/mem bs=4 count=1 skip=$((0x3F204008/4)) 2>/dev/null | xxd
```

---

#### 3. Logic Analyzer / Oscilloscope

**الإعداد الأساسي:**

```
Signal    | GPIO Pin  | Color (convention)
----------|-----------|-------------------
SCLK      | CLK line  | Yellow
MOSI      | TX line   | Blue
MISO      | RX line   | Green
CS (SS)   | CS line   | Red / Orange
```

**نقاط المقياس على الـ oscilloscope:**

```
CS  ‾‾‾\___________________________/‾‾‾
         |                         |
CLK ____/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\____   (SPI_MODE_0)
         |
MOSI ----X---X---X---X---X---X---X------
MISO --------X---X---X---X---X---X------
```

**إعدادات الـ logic analyzer (مثال Saleae Logic):**

```
- Sample rate: على الأقل 4x تردد الـ SCLK
  مثال: SCLK=1MHz → sample rate >= 4MS/s
- Voltage threshold: 1.65V لـ 3.3V signals
- Protocol decoder: SPI
  - CPOL: حسب الـ mode المبرمج في kernel
  - CPHA: حسب الـ mode
  - Bit order: MSB first (الافتراضي في spidev)
```

**إشارات الـ trigger:**

```bash
# شغّل الـ trigger على الـ CS falling edge
# ثم ابعت transfer من userspace:
echo -ne '\xAB\xCD' > /dev/spidev0.0
```

---

#### 4. مشاكل الـ hardware الشائعة وأنماط الـ kernel log

| المشكلة | ما تشوفه في dmesg | التشخيص |
|---|---|---|
| CLK polarity غلط | بيانات garbage في RX، لا errors | قيس SCLK idle level: high=CPOL1, low=CPOL0 |
| CPHA خطأ | أول/آخر bit مش صح | شوف oscilloscope: sample edge مطابق للـ mode؟ |
| CS مش بينزل | transfer يرجع 0 bytes أو timeout | قيس CS line مع logic analyzer |
| speed عالية جداً | نبضات CLK مشوّهة | ارجع لـ spi_setup error في dmesg، قلّل speed |
| noise على MISO | RX data متغير مع نفس TX | أضف pull-down/pull-up، قصّر الأسلاك |
| الـ device مش بيرد | RX كلها 0xFF أو 0x00 | تحقق VCC وGND الـ device، تحقق الـ CS polarity |
| DMA alignment | crash أو corruption بعد كل 4096 bytes | راجع `ALIGN(u_tmp->len, ARCH_DMA_MINALIGN)` في الكود |

---

#### 5. Device Tree Debugging

```bash
# عرض الـ DT الحالي كـ text
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "spi@"

# تحقق من الـ compatible string
cat /sys/bus/spi/devices/spi0.0/of_node/compatible
# يجب يكون واحد من spidev_dt_ids[] في الكود
# مثلاً: rohm,dh2228fv

# تحقق من الـ spi-max-frequency
cat /sys/bus/spi/devices/spi0.0/of_node/spi-max-frequency
# يجب يرجع رقم مثل 10000000

# تحقق من الـ reg (chipselect number)
cat /sys/bus/spi/devices/spi0.0/of_node/reg

# لو الـ driver مش بيعمل probe — تحقق من الـ DT match
dmesg | grep -E "spidev|spi.*probe|OF.*match"

# مثال DT صح لـ spidev
cat << 'EOF'
/* في ملف .dts أو .dtsi */
&spi0 {
    #address-cells = <1>;
    #size-cells = <0>;
    status = "okay";

    my_device: spi-device@0 {
        compatible = "rohm,dh2228fv";   /* من spidev_dt_ids */
        reg = <0>;                       /* chipselect 0 */
        spi-max-frequency = <1000000>;  /* 1MHz */
        spi-cpol;                        /* CPOL=1 لو محتاج */
        spi-cpha;                        /* CPHA=1 لو محتاج */
    };
};
EOF

# بعد تعديل DT، تحقق من الـ overlays
ls /sys/firmware/devicetree/base/chosen/

# DT overlay لـ Raspberry Pi (مثال)
dtoverlay=spi0-1cs,cs0_pin=8
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للنسخ

**فحص وجود الـ device:**

```bash
# تحقق من وجود /dev/spidevX.Y
ls -la /dev/spidev*
# crw-rw---- 1 root spi 153, 0 Feb 23 10:00 /dev/spidev0.0
#                        ^^^  ^ major=153, minor=0

# تحقق من الـ driver مرتبط
cat /sys/bus/spi/devices/spi0.0/driver/module/name
# spidev

# لو مش موجود — اعرف الـ devices المتاحة
ls /sys/bus/spi/devices/
# spi0.0  spi0.1  spi1.0

# تحقق من الـ probe
dmesg | grep -E "spidev|spi[0-9]"
```

**اختبار read/write أساسي:**

```bash
# كتابة bytes وقراءة RX (full-duplex مع نفس الحجم)
python3 - <<'EOF'
import spidev

bus, device = 0, 0
spi = spidev.SpiDev()
spi.open(bus, device)
spi.max_speed_hz = 500_000
spi.mode = 0b00  # SPI_MODE_0

# xfer2 = full duplex
tx = [0xAB, 0xCD, 0xEF, 0x00]
rx = spi.xfer2(tx)
print(f"TX: {[hex(b) for b in tx]}")
print(f"RX: {[hex(b) for b in rx]}")

spi.close()
EOF
```

**قراءة وكتابة الـ ioctl parameters:**

```bash
python3 - <<'EOF'
import spidev

spi = spidev.SpiDev()
spi.open(0, 0)

# قراءة الإعدادات الحالية
print(f"Mode:          {spi.mode}")
print(f"Bits per word: {spi.bits_per_word}")
print(f"Max speed:     {spi.max_speed_hz} Hz")
print(f"LSB first:     {spi.lsbfirst}")
print(f"CS high:       {spi.cshigh}")

spi.close()
EOF
```

**تفعيل الـ ftrace لجلسة debugging كاملة:**

```bash
#!/bin/bash
# spi_trace.sh — تسجيل كل SPI activity

TRACE=/sys/kernel/debug/tracing

# reset
echo 0 > $TRACE/tracing_on
echo > $TRACE/trace
echo nop > $TRACE/current_tracer

# فعّل SPI events
echo 1 > $TRACE/events/spi/spi_message_start/enable
echo 1 > $TRACE/events/spi/spi_message_done/enable
echo 1 > $TRACE/events/spi/spi_transfer_start/enable
echo 1 > $TRACE/events/spi/spi_transfer_stop/enable

# فعّل الـ tracing
echo 1 > $TRACE/tracing_on
echo "Tracing started. Run your SPI operations now."
echo "Press ENTER to stop..."
read

echo 0 > $TRACE/tracing_on
echo "=== SPI Trace ==="
cat $TRACE/trace | grep -v "^#"
echo "================="
```

**تفعيل الـ dynamic debug:**

```bash
#!/bin/bash
# spi_debug.sh — تفعيل كل debug messages للـ SPI

DYNDBG=/sys/kernel/debug/dynamic_debug/control

# فعّل spidev debug
echo "file drivers/spi/spidev.c +pflmt" > $DYNDBG
# p = print, f = function name, l = line number, m = module name, t = thread ID

# فعّل SPI core debug
echo "file drivers/spi/spi.c +p" > $DYNDBG

# اعرض الرسائل
dmesg -wH &
DMESG_PID=$!

echo "Dynamic debug enabled. Run SPI operations..."
sleep 5

kill $DMESG_PID
```

**فحص bufsiz وتعديله:**

```bash
# قراءة الـ bufsiz الحالية
cat /sys/module/spidev/parameters/bufsiz
# 4096

# تعديل عند تحميل الـ module
modprobe spidev bufsiz=65536

# أو لو محمّل مسبقاً — unload وأعد التحميل
rmmod spidev
modprobe spidev bufsiz=65536

# تحقق
cat /sys/module/spidev/parameters/bufsiz
# 65536
```

**اختبار EMSGSIZE:**

```bash
# هذا هيفشل لو bufsiz=4096
python3 -c "
import spidev
spi = spidev.SpiDev()
spi.open(0,0)
spi.max_speed_hz = 1_000_000
try:
    rx = spi.xfer2([0x00]*8192)  # أكبر من bufsiz
except OSError as e:
    print(f'Error: {e}')  # [Errno 90] Message too long = EMSGSIZE
spi.close()
"
```

**تشغيل spidev_test من kernel tools:**

```bash
# بناء spidev_test
cd /workspace/external/linux
make -C tools/spi/

# تشغيل اختبار أساسي
./tools/spi/spidev_test -D /dev/spidev0.0 -s 1000000 -v -p "Hello\x00"

# full duplex test
./tools/spi/spidev_test -D /dev/spidev0.0 -s 500000 -v -p "\xAB\xCD\xEF"

# ناتج مثال:
# spi mode: 0x0
# bits per word: 8
# max speed: 1000000 Hz (1000 kHz)
# TX | AB CD EF __ __ __ __ __  __ __ __ __ __ __ __ __ | ..
# RX | AB CD EF __ __ __ __ __  __ __ __ __ __ __ __ __ | ..
# (لو loopback) أو قيم مختلفة من الـ device
```

**مراقبة errors في real-time:**

```bash
# راقب dmesg مع filter للـ SPI errors
dmesg -w | grep -E "spi|SPI|spidev" | grep -iE "error|fail|timeout|warn"

# أو بـ journalctl لو systemd
journalctl -f -k | grep -iE "spi"

# عدّ الـ transfer errors من statistics
watch -n1 'cat /sys/kernel/debug/spi0/statistics 2>/dev/null || echo "no stats"'
```

**تحليل binding مشاكل:**

```bash
# لماذا الـ driver مش بيعمل probe؟
dmesg | grep -A3 "spidev"

# الناتج لو compatible غلط:
# spi spi0.0: spidev listed directly in DT is not supported

# حل: غيّر الـ compatible في DT
# من: compatible = "spidev"
# إلى: compatible = "rohm,dh2228fv"  (أي entry من spidev_dt_ids[])

# تحقق من الـ modalias
cat /sys/bus/spi/devices/spi0.0/modalias
# spi:rohm-dh2228fv  ← يجب يطابق entry في spidev_spi_ids[]

# قراءة uevents
cat /sys/bus/spi/devices/spi0.0/uevent
# DRIVER=spidev
# MODALIAS=spi:rohm-dh2228fv
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: فشل الـ probe على بورد RK3562 بسبب compatible string غلط

#### العنوان
Industrial gateway على RK3562 — الـ `/dev/spidev1.0` مش بيظهر خالص

#### السياق
شركة بتعمل industrial IoT gateway بتستخدم RK3562 SoC. الـ firmware engineer كتب Device Tree node عشان يوصل لـ SPI flash خارجي عن طريق spidev من الـ userspace. البورد اتشحن للعميل وبعدين الـ application بدأ يطلع error `No such file or directory` لما بيعمل `open("/dev/spidev1.0", ...)`.

#### المشكلة
الـ engineer استخدم `compatible = "spidev"` في الـ DT node، وده كان صح في kernel قديم. لكن في الـ kernels الحديثة (من بعد ما تم إزالة الـ "spidev" generic name)، الـ spidev driver مش بيعمل probe للـ device وبيطلع error في الـ dmesg.

```dmesg
[    2.134567] spi_master spi1: spidev compatible is no longer supported
[    2.134599] spi1.0: no driver bound
```

#### التحليل
الـ documentation بتقول صريح:

> *"It used to be supported to define an SPI device using the "spidev" name... But this is no longer supported by the Linux kernel and instead a real SPI device name as listed in one of the tables must be used."*

الـ driver بيشوف الـ compatible string ضد `spidev_dt_ids[]` table — `"spidev"` مش موجود فيها، فالـ probe بيفشل.

#### الحل

**خطوة 1** — شوف الـ tables الموجودة في الـ kernel source:

```bash
grep -r "spidev_dt_ids" drivers/spi/spidev.c
```

**خطوة 2** — غير الـ DT node لاستخدام compatible موجود فعلاً في `spidev_dt_ids[]`، زي `"rohm,dh2228fv"` أو `"lineartechnology,ltc2488"`:

```dts
/* غلط — مش بيشتغل في kernel حديث */
&spi1 {
    status = "okay";
    spiflash@0 {
        compatible = "spidev";   /* X مش بيشتغل */
        reg = <0>;
        spi-max-frequency = <10000000>;
    };
};
```

```dts
/* صح — باستخدام compatible موجود في spidev_dt_ids[] */
&spi1 {
    status = "okay";
    spiflash@0 {
        compatible = "rohm,dh2228fv";   /* موجود في الجدول */
        reg = <0>;
        spi-max-frequency = <10000000>;
    };
};
```

**أو** لو الـ device اسمه مش في الجدول، استخدم الـ sysfs manual binding:

```bash
echo spidev > /sys/bus/spi/devices/spi1.0/driver_override
echo spi1.0 > /sys/bus/spi/drivers/spidev/bind
ls /dev/spidev*
# /dev/spidev1.0
```

#### الدرس المستفاد
الـ `compatible = "spidev"` اتشال من الـ kernel — أي بورد bring-up جديد لازم يستخدم اسم device حقيقي من `spidev_dt_ids[]`، أو يعمل manual bind عن طريق `driver_override`. متفترضش الـ DT من kernel قديم هيشتغل بدون تعديل.

---

### السيناريو الثاني: فشل الـ `SPI_IOC_MESSAGE` على STM32MP1 بسبب تجاوز حد الـ page size

#### العنوان
Android TV box على Allwinner H616 — الـ ioctl بيرجع `EMSGSIZE` لما الـ transfer أكبر من 4096 bytes

#### السياق
فريق يشتغل على Android TV box بيستخدم Allwinner H616. الـ application بتاعتهم بترسل firmware updates لـ microcontroller خارجي عن طريق SPI في الـ factory. الـ transfer size المطلوب هو 8192 bytes في مرة واحدة، لكن الـ ioctl بيفشل.

#### المشكلة
الـ application بيعمل `SPI_IOC_MESSAGE(1)` مع `len = 8192`، بيرجع `-1` وَ `errno = EMSGSIZE`.

```c
struct spi_ioc_transfer tr = {
    .tx_buf = (unsigned long)tx_buf,
    .rx_buf = (unsigned long)rx_buf,
    .len    = 8192,   /* بيفشل! */
    .speed_hz = 1000000,
};

ret = ioctl(fd, SPI_IOC_MESSAGE(1), &tr);
/* ret = -1, errno = EMSGSIZE */
```

#### التحليل
الـ documentation بتقول:

> *"There's a limit on the number of bytes each I/O request can transfer to the SPI device. It defaults to one page, but that can be changed using a module parameter."*

الـ default page size في Linux = 4096 bytes. الـ spidev driver بيرفض أي transfer أكبر منها، والـ kernel بيرجع `EMSGSIZE`.

#### الحل

**خطوة 1** — تحقق من الحد الحالي:

```bash
cat /sys/module/spidev/parameters/bufsiz
# 4096
```

**خطوة 2** — زود الـ buffer size عن طريق module parameter:

```bash
# مؤقت — في runtime
echo 65536 > /sys/module/spidev/parameters/bufsiz

# دايم — في /etc/modprobe.d/
echo "options spidev bufsiz=65536" > /etc/modprobe.d/spidev.conf
```

**خطوة 3** — أو قسّم الـ transfer لـ chunks في الـ application code:

```c
#define CHUNK_SIZE 4096

int spi_write_large(int fd, uint8_t *buf, size_t total_len)
{
    size_t offset = 0;
    while (offset < total_len) {
        size_t chunk = MIN(CHUNK_SIZE, total_len - offset);
        struct spi_ioc_transfer tr = {
            .tx_buf   = (unsigned long)(buf + offset),
            .len      = chunk,
            .speed_hz = 1000000,
        };
        if (ioctl(fd, SPI_IOC_MESSAGE(1), &tr) < 0)
            return -errno;
        offset += chunk;
    }
    return 0;
}
```

**ملاحظة مهمة**: لو بتقسم الـ transfer، الـ chipselect بيتشال بين كل chunk (لأن كل `ioctl` مستقل). لو الـ slave مش بيتحمل ده، لازم تزود الـ bufsiz بدل ما تقسم.

#### الدرس المستفاد
الـ spidev بيحكم على size الـ transfer بالـ page size افتراضياً. في الـ factory programming أو الـ bulk transfers، إما ازود الـ `bufsiz` module parameter، أو صمم الـ application يشتغل مع الحد الموجود. خليك دايماً تتحقق من `errno` بالتفصيل لما `ioctl` يفشل.

---

### السيناريو الثالث: مشكلة الـ chipselect بين transfers على i.MX8 في نظام multi-slave

#### العنوان
Automotive ECU على i.MX8MQ — الـ SPI ADC بيقرأ قيم خطأ بسبب deactivation الـ CS غير المتوقعة

#### السياق
شركة automotive بتبني ECU بتستخدم i.MX8MQ. الـ SPI bus فيه 3 devices: ADC، DAC، وـ EEPROM. الـ software team قرر يستخدم spidev للـ ADC عشان يبروتوتايپ بسرعة. البروتوكول بتاع الـ ADC بيحتاج: أول ترانزاكشن 3 bytes command، تاني ترانزاكشن 2 bytes قراءة — والـ CS لازم يفضل low بينهم.

#### المشكلة
لما بيستخدم `write()` وبعدين `read()` كلهم منفصلين، الـ ADC بيرجع قيم عشوائية. الـ oscilloscope بيورّي إن الـ CS بيعلى بين الـ write والـ read.

```c
/* ده الكود الغلط */
write(fd, cmd, 3);   /* CS بيعلى هنا! */
read(fd, resp, 2);   /* بيعمل CS جديد — ADC مش بيعرف انت في أنهي state */
```

#### التحليل
الـ documentation واضحة:

> *"Standard read() and write() operations are obviously only half-duplex, and **the chipselect is deactivated between those operations**. Full-duplex access, and composite operation without chipselect de-activation, is available using the `SPI_IOC_MESSAGE(N)` request."*

الـ `write()` بيعمل CS low → transfer → CS high. نفس الكلام للـ `read()`. مفيش طريقة تمنع ده غير `SPI_IOC_MESSAGE`.

#### الحل

استخدام `SPI_IOC_MESSAGE(2)` مع `cs_change = 0` عشان الـ CS يفضل low طول الـ composite transfer:

```c
#include <linux/spi/spidev.h>
#include <sys/ioctl.h>

int adc_read(int fd, uint8_t *cmd, uint8_t *resp)
{
    struct spi_ioc_transfer tr[2] = {
        {
            /* أول transfer: إرسال الـ command */
            .tx_buf        = (unsigned long)cmd,
            .rx_buf        = 0,
            .len           = 3,
            .speed_hz      = 1000000,
            .cs_change     = 0,   /* CS يفضل low بعد الـ transfer ده */
            .delay_usecs   = 0,
        },
        {
            /* تاني transfer: استقبال الـ response */
            .tx_buf        = 0,
            .rx_buf        = (unsigned long)resp,
            .len           = 2,
            .speed_hz      = 1000000,
            .cs_change     = 0,
        },
    };

    /* الاتنين بيتعملوا في kernel request واحد — CS مش بيتشال */
    return ioctl(fd, SPI_IOC_MESSAGE(2), tr);
}
```

للتحقق:

```bash
# شوف الـ CS polarity والـ mode
python3 -c "
import spidev
spi = spidev.SpiDev()
spi.open(0, 0)
print('mode:', spi.mode)
"
```

#### الدرس المستفاد
`read()` و`write()` المنفصلين دايماً بيعملوا deactivation للـ CS بينهم — ده مش bug، ده by design. أي بروتوكول بيحتاج الـ CS يفضل active طول transactionٍ مركب، لازم يستخدم `SPI_IOC_MESSAGE(N)`. اقرأ datasheet الـ slave device قبل ما تقرر أي API تستخدم.

---

### السيناريو الرابع: خطأ في الـ SPI mode على AM62x — الـ sensor بيرد بـ garbage

#### العنوان
IoT sensor node على AM62x — temperature sensor بيقرأ `0xFF` دايماً رغم الـ wiring الصح

#### السياق
Team بتشتغل على industrial IoT node بتستخدم TI AM62x SoC. البورد فيها MAX31855 thermocouple-to-digital converter متوصل على SPI0. الـ engineer كتب Python script بسيط بـ spidev، لكن كل قراءة بترجع `0xFFFFFFFF`.

#### المشكلة
الـ wiring سليم والـ VCC موجود، لكن الـ data غلط. السبب إن MAX31855 بيشتغل على **SPI Mode 0** (CPOL=0, CPHA=0) لكن الـ default لـ spidev مش مضمون إنه Mode 0.

#### التحليل
الـ documentation بتشرح:

> *"SPI_IOC_RD_MODE, SPI_IOC_WR_MODE ... pass a pointer to a byte which will return (RD) or assign (WR) the SPI transfer mode. Use the constants SPI_MODE_0..SPI_MODE_3; or if you prefer you can combine SPI_CPOL ... or SPI_CPHA ..."*

لو الـ mode مش متضبط صح، الـ clock polarity أو الـ sampling edge بيكون غلط، والـ slave مش بيعرف يقرأ الـ bits.

التحقق من الـ mode الحالي:

```bash
python3 -c "
import spidev
spi = spidev.SpiDev()
spi.open(0, 0)
print('Current mode:', spi.mode)
spi.close()
"
```

أو عن طريق C:

```c
uint8_t mode;
ioctl(fd, SPI_IOC_RD_MODE, &mode);
printf("SPI mode: %d\n", mode);
/* لو مش 0، الـ MAX31855 مش هيشتغل */
```

#### الحل

**في C**:

```c
#include <linux/spi/spidev.h>
#include <sys/ioctl.h>
#include <fcntl.h>
#include <stdint.h>

int setup_max31855(const char *dev)
{
    int fd = open(dev, O_RDWR);
    if (fd < 0) return -1;

    uint8_t  mode  = SPI_MODE_0;   /* CPOL=0, CPHA=0 */
    uint8_t  bits  = 8;
    uint32_t speed = 5000000;      /* MAX31855 يدعم حتى 5 MHz */

    ioctl(fd, SPI_IOC_WR_MODE,          &mode);
    ioctl(fd, SPI_IOC_WR_BITS_PER_WORD, &bits);
    ioctl(fd, SPI_IOC_WR_MAX_SPEED_HZ,  &speed);

    return fd;
}

int read_temp_raw(int fd)
{
    uint8_t rx[4] = {0};
    struct spi_ioc_transfer tr = {
        .tx_buf   = 0,
        .rx_buf   = (unsigned long)rx,
        .len      = 4,
        .speed_hz = 5000000,
    };
    ioctl(fd, SPI_IOC_MESSAGE(1), &tr);

    /* MAX31855: bits [31:18] = temperature in 0.25°C steps */
    int32_t raw = (rx[0] << 24) | (rx[1] << 16) | (rx[2] << 8) | rx[3];
    return raw >> 18;  /* sign-extended 14-bit value */
}
```

**في الـ DT** — تأكد إن الـ SPI controller configured بـ mode صح:

```dts
&spi0 {
    status = "okay";
    thermocouple@0 {
        compatible = "maxim,max31855";
        reg = <0>;
        spi-max-frequency = <5000000>;
        spi-cpol;   /* CPOL=0: لا تحط السطر ده */
        spi-cpha;   /* CPHA=0: لا تحط السطر ده */
        /* SPI Mode 0 = default، مش محتاج تحط حاجة */
    };
};
```

#### الدرس المستفاد
دايماً verify الـ SPI mode بـ `SPI_IOC_RD_MODE` قبل أي transfer. كتير من الـ bugs في sensor bring-up بتيجي من إن الـ mode غلط. ارجع للـ datasheet واتأكد من CPOL وCPHA المطلوبين، وبعدين اضبطهم صريح بدل ما تعتمد على الـ defaults.

---

### السيناريو الخامس: الـ spidev مش بيظهر بعد reboot على custom board — مشكلة الـ udev/mdev

#### العنوان
Custom Linux على i.MX8MM — `/dev/spidev0.0` بيظهر مرة واحدة وبيختفي بعد reboot

#### السياق
شركة startup بتبني custom embedded Linux system على i.MX8MM بـ Yocto. الـ system استخدام Busybox mdev بدل udev. الـ engineer لاحظ إن `/dev/spidev0.0` موجود لما يعمل bind manually، لكن بعد كل reboot لازم يعمل bind تاني.

#### المشكلة
الـ DT node موجود وصح، الـ spidev module بيتحمل، لكن الـ device file مش بيتعمل automatically. بعد reboot:

```bash
ls /dev/spidev*
# ls: cannot access '/dev/spidev*': No such file or directory
```

#### التحليل
الـ documentation بتشرح:

> *"When the spidev driver is bound to a SPI device, the sysfs node for the device will include a child device node with a 'dev' attribute that will be understood by udev or mdev."*

> *"Do not try to manage the /dev character device special file nodes by hand. That's error prone..."*

المشكلة في الـ mdev configuration — الـ mdev مش عنده rule يعمل `/dev/spidev*` automatically. الـ kernel بيعمل الـ sysfs node، لكن الـ device file في `/dev` بيتعمل بواسطة الـ userspace daemon.

التحقق:

```bash
# شوف لو الـ sysfs node موجود
ls /sys/class/spidev/
# spidev0.0

# شوف الـ major:minor
cat /sys/class/spidev/spidev0.0/dev
# 153:0

# الـ kernel بيعمل الـ sysfs — المشكلة في mdev
```

#### الحل

**خطوة 1** — أضف mdev rule لـ spidev:

```bash
# في /etc/mdev.conf
echo 'spidev[0-9]+\.[0-9]+  0:0 660' >> /etc/mdev.conf
```

**خطوة 2** — أو لو بتستخدم udev، أضف udev rule:

```bash
# في /etc/udev/rules.d/90-spidev.rules
cat > /etc/udev/rules.d/90-spidev.rules << 'EOF'
KERNEL=="spidev*", GROUP="spi", MODE="0660"
EOF
udevadm control --reload-rules
```

**خطوة 3** — لو بتستخدم manual binding وعايزه يحصل عند الـ boot، ضيفه في init script:

```bash
#!/bin/sh
# في /etc/init.d/spidev-setup

DEVICE="spi0.0"

echo spidev > /sys/bus/spi/devices/${DEVICE}/driver_override
echo ${DEVICE} > /sys/bus/spi/drivers/spidev/bind

# انتظر الـ mdev يعمل الـ device node
sleep 0.1
ls /dev/spidev* && echo "spidev ready" || echo "spidev failed"
```

**خطوة 4** — التحقق الكامل بعد الإصلاح:

```bash
# بعد reboot
ls -la /dev/spidev*
# crw-rw---- 1 root spi 153, 0 Feb 23 10:00 /dev/spidev0.0

# تأكيد الـ binding
cat /sys/bus/spi/devices/spi0.0/driver
# spidev

# اختبار بسيط
python3 -c "
import spidev
spi = spidev.SpiDev()
spi.open(0, 0)
print('OK: spidev0.0 accessible')
spi.close()
"
```

#### الدرس المستفاد
الـ spidev driver بيعمل الـ sysfs entries، لكن الـ `/dev` node تعتمد على الـ userspace daemon (udev أو mdev). في embedded systems الـ minimal زي Busybox-based systems، لازم تتأكد إن الـ mdev rules موجودة. استخدام `driver_override` و`bind` عبر sysfs مفيد في الـ debugging، لكن مش بديل عن الـ proper device manager configuration في production.
## Phase 7: مصادر ومراجع

### التوثيق الرسمي للـ Kernel

| المصدر | الرابط | الوصف |
|--------|--------|-------|
| **spidev — Official Kernel Docs** | [docs.kernel.org/spi/spidev.html](https://docs.kernel.org/spi/spidev.html) | التوثيق الرسمي لـ spidev userspace API |
| **SPI Driver API** | [static.lwn.net/kerneldoc/driver-api/spi.html](https://static.lwn.net/kerneldoc/driver-api/spi.html) | الـ SPI subsystem API الكاملة للـ kernel |
| **spidev.rst — kernel source** | [github.com/torvalds/linux — Documentation/spi/spidev.rst](https://github.com/torvalds/linux/blob/master/Documentation/spi/spidev.rst) | ملف التوثيق المصدري في الـ kernel tree |
| **spidev.c — kernel source** | [github.com/torvalds/linux — drivers/spi/spidev.c](https://github.com/torvalds/linux/blob/master/drivers/spi/spidev.c) | الكود المصدري للـ driver نفسه |

الـ paths داخل الـ kernel tree:

```
Documentation/spi/spidev.rst        ← التوثيق الرسمي
drivers/spi/spidev.c                ← الـ driver implementation
include/uapi/linux/spi/spidev.h     ← الـ userspace API headers
tools/spi/spidev_test.c             ← مثال عملي للاختبار
tools/spi/spidev_fdx.c              ← مثال full-duplex
```

---

### مقالات LWN.net

الـ LWN.net هو أهم مرجع لتطور الـ SPI subsystem في الـ kernel. الـ articles التالية تغطي التاريخ الكامل من البداية:

| المقال | الرابط | التاريخ التقريبي |
|--------|--------|-----------------|
| **SPI core** — أول نسخة من الـ SPI framework | [lwn.net/Articles/138037](https://lwn.net/Articles/138037/) | 2005 |
| **SPI core -- revisited** — مراجعة وتحسين أولي | [lwn.net/Articles/141186](https://lwn.net/Articles/141186/) | 2005 |
| **spi** — إضافة الـ spidev character device | [lwn.net/Articles/146581](https://lwn.net/Articles/146581/) | 2005 |
| **simple SPI framework** — الـ framework المبسّط | [lwn.net/Articles/154425](https://lwn.net/Articles/154425/) | 2006 |
| **SPI core refresh** — تحديث شامل للـ core | [lwn.net/Articles/162268](https://lwn.net/Articles/162268/) | 2006 |
| **Add SPI over GPIO driver** — SPI bitbanging عبر GPIO | [lwn.net/Articles/290068](https://lwn.net/Articles/290068/) | 2009 |
| **spi: Add slave mode support** — دعم SPI slave mode | [lwn.net/Articles/723440](https://lwn.net/Articles/723440/) | 2017 |

> الـ LWN userspace API documentation: [static.lwn.net/kerneldoc/spi/spidev.html](https://static.lwn.net/kerneldoc/spi/spidev.html)

---

### kernelnewbies.org — تغيرات spidev عبر إصدارات الـ Kernel

| الإصدار | التغيير | الرابط |
|---------|---------|-------|
| **Linux 3.15** | إضافة Dual/Quad SPI Transfers لـ spidev | [kernelnewbies.org/Linux_3.15-DriversArch](https://kernelnewbies.org/Linux_3.15-DriversArch) |
| **Linux 4.8** | تفعيل spidev على Intel Edison | [kernelnewbies.org/Linux_4.8](https://kernelnewbies.org/Linux_4.8) |
| **Linux 4.13** | تحسينات على الـ SPI subsystem | [kernelnewbies.org/Linux_4.13](https://kernelnewbies.org/Linux_4.13) |
| **Linux 4.14** | تحسينات على الـ SPI subsystem | [kernelnewbies.org/Linux_4.14](https://kernelnewbies.org/Linux_4.14) |
| **Linux 5.8** | دعم Octal mode transfers في spidev | [kernelnewbies.org/Linux_5.8](https://kernelnewbies.org/Linux_5.8) |
| **Linux 6.3** | إضافة Silicon Labs SI3210 لـ spidev tables | [kernelnewbies.org/Linux_6.3](https://kernelnewbies.org/Linux_6.3) |
| **Linux 6.9** | تحديثات على الـ SPI subsystem | [kernelnewbies.org/Linux_6.9](https://kernelnewbies.org/Linux_6.9) |

---

### elinux.org — تطبيقات عملية على Hardware حقيقي

| الصفحة | الرابط | المحتوى |
|--------|--------|---------|
| **RPi SPI** | [elinux.org/RPi_SPI](https://elinux.org/RPi_SPI) | تفعيل spidev على Raspberry Pi، تشغيل spidev_test |
| **BeagleBone Black Enable SPIDEV** | [elinux.org/BeagleBone_Black_Enable_SPIDEV](https://elinux.org/BeagleBone_Black_Enable_SPIDEV) | Device Tree overlay لتفعيل spidev على BeagleBone Black |
| **ECE497 SPI Project** | [elinux.org/ECE497_SPI_Project](https://elinux.org/ECE497_SPI_Project) | مشروع تعليمي كامل باستخدام SPI و spidev |

---

### الـ Mailing List الرسمي

الـ mailing list الرئيسي لكل ما يخص الـ SPI subsystem والـ spidev driver:

```
linux-spi@vger.kernel.org
```

- أي patch لإضافة device جديد لـ `spidev_spi_ids[]` أو `spidev_dt_ids[]` يُرسل هنا.
- الأرشيف متاح على: [lore.kernel.org/linux-spi/](https://lore.kernel.org/linux-spi/)

---

### مراجع إضافية مفيدة

| المرجع | الرابط | الفائدة |
|--------|--------|---------|
| **SPIdev — linux-sunxi.org** | [linux-sunxi.org/SPIdev](https://linux-sunxi.org/SPIdev) | تفاصيل عملية لـ Allwinner SoCs |
| **STM32 — How to use SPI from userland** | [wiki.st.com — How_to_use_SPI_from_Linux_userland_with_spidev](https://wiki.st.com/stm32mpu/wiki/How_to_use_SPI_from_Linux_userland_with_spidev) | مثال كامل على STM32 |
| **Peripheral Interaction Without a Kernel Driver** | [embeddedrelated.com/showarticle/1485.php](https://www.embeddedrelated.com/showarticle/1485.php) | شرح مفصل لاستخدام spidev بدون kernel driver |
| **Using spidev with Linux Device Tree** | [yurovsky.github.io/2016/10/07/spidev-linux-devices.html](https://yurovsky.github.io/2016/10/07/spidev-linux-devices.html) | كيفية ربط spidev بالـ Device Tree |
| **Toradex — SPI Linux** | [developer.toradex.com/linux-bsp/…/spi-linux](https://developer.toradex.com/linux-bsp/application-development/peripheral-access/spi-linux/) | دليل عملي شامل لـ embedded Linux |
| **spidev Python library** | [pypi.org/project/spidev](https://pypi.org/project/spidev/) | استخدام spidev من Python في الـ userspace |

---

### الكتب المرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
الكتاب المرجعي الأساسي لكتابة kernel drivers، متاح مجاناً:

- **Chapter 1**: Introduction to Device Drivers
- **Chapter 3**: Char Drivers — أساس فهم `/dev/spidevB.C` كـ character device
- **Chapter 6**: Advanced Char Driver Operations — الـ `ioctl()` بالتفصيل
- الكتاب كاملاً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **Chapter 17**: Devices and Modules — فهم الـ device model وكيف يتعامل الـ kernel مع الـ devices
- الكتاب يشرح الـ `sysfs`, `udev`, وآلية الـ driver binding التي يعتمدها spidev

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **Chapter 8**: Device Drivers — مقدمة عملية للـ character drivers
- **Chapter 14**: Embedded Linux Porting — يتناول الـ board support وإعداد الـ SPI على embedded platforms

---

### Search Terms للبحث عن معلومات إضافية

```
# البحث في الـ kernel source
grep -r "spidev" drivers/spi/
grep -r "SPI_IOC_MESSAGE" include/uapi/linux/spi/

# البحث في الـ mailing list
lore.kernel.org/linux-spi/?q=spidev

# مصطلحات البحث على الإنترنت
"spidev ioctl SPI_IOC_MESSAGE"
"linux spidev device tree compatible string"
"spidev_dt_ids modalias"
"linux SPI userspace half duplex full duplex"
"spidev major 153 character device"
"linux spi_board_info spidev"
"linux spidev driver_override bind sysfs"
```
## Phase 8: Writing simple module

### الـ Function المختارة: `spidev_sync`

**الـ `spidev_sync`** هي قلب كل عملية نقل بيانات في الـ spidev driver — كل `read()` وكل `write()` وكل `SPI_IOC_MESSAGE` ioctl بتمر عليها. hooking عليها بـ kprobe بيخلينا نشوف كل transfer بيعمله أي userspace program على أي SPI device، مع حجم البيانات والـ bus number.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * spidev_sync_probe.c
 *
 * kprobe on spidev_sync() — logs every userspace SPI transfer:
 * caller process, SPI bus/chip-select, and message byte count.
 *
 * Build: add to a kernel module Makefile, load with insmod.
 * Tested against Linux 6.x with CONFIG_KPROBES=y.
 */

#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/spi/spi.h>      /* struct spi_device, spi_message */
#include <linux/sched.h>        /* current, task_comm_len          */

/* ------------------------------------------------------------------ */
/* Forward-declare the target function signature so we can cast args   */
/* ------------------------------------------------------------------ */

/*
 * spidev_sync(struct spidev_data *spidev, struct spi_message *message)
 *
 * spidev_data is a private struct inside spidev.c — we only need the
 * first two fields (devt + spi_lock) to reach the embedded spi pointer,
 * but it is safer (and sufficient) to read from the spi_message itself.
 *
 * spi_message carries:
 *   - spi        : pointer to the SPI device (bus_num, chip_select)
 *   - transfers  : linked list of spi_transfer segments
 *   - frame_length: total bytes in the message (filled by spi_message_init)
 *
 * We read spi_message.spi directly — no need to touch the opaque
 * spidev_data struct at all.
 */

/* ------------------------------------------------------------------ */
/* kprobe pre-handler — runs just before spidev_sync executes          */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64:
     *   rdi = first  arg = spidev_data *spidev  (we skip this)
     *   rsi = second arg = spi_message  *message
     *
     * On ARM64:
     *   x0  = spidev, x1 = message
     *
     * pt_regs accessors (regs->di / regs->si on x86-64) are
     * arch-specific, so we use the portable kprobe_get_argument()
     * helper-style cast via regs_get_kernel_argument().
     */
#ifdef CONFIG_X86_64
    struct spi_message *msg =
        (struct spi_message *) regs->si;          /* 2nd arg = RSI */
#elif defined(CONFIG_ARM64)
    struct spi_message *msg =
        (struct spi_message *) regs->regs[1];     /* 2nd arg = x1  */
#else
    /* Generic fallback — may need porting for other arches */
    struct spi_message *msg = NULL;
#endif

    /* Safety: the message pointer must be valid kernel memory */
    if (!msg || !msg->spi)
        return 0;

    /*
     * spi_message.frame_length is the sum of all spi_transfer.len
     * values — computed by spi_message_add_tail() / spi_finalize_current_transfer().
     * It may be zero if the controller hasn't filled it yet, so we
     * walk the transfer list ourselves for accuracy.
     */
    struct spi_transfer *xfer;
    unsigned int total_bytes = 0;
    unsigned int n_transfers = 0;

    list_for_each_entry(xfer, &msg->transfers, transfer_list) {
        total_bytes += xfer->len;
        n_transfers++;
    }

    pr_info("spidev_probe: [%s pid=%d] SPI%d.%d — %u transfer(s), "
            "%u byte(s) total, speed=%u Hz\n",
            current->comm,                        /* process name   */
            task_pid_nr(current),                 /* process PID    */
            msg->spi->controller->bus_num,        /* SPI bus index  */
            spi_get_chipselect(msg->spi, 0),      /* chip-select    */
            n_transfers,                          /* # of segments  */
            total_bytes,                          /* total TX+RX    */
            msg->spi->max_speed_hz);              /* configured Hz  */

    return 0; /* 0 = continue normal execution, don't skip the function */
}

/* ------------------------------------------------------------------ */
/* kprobe post-handler — runs after spidev_sync returns                */
/* ------------------------------------------------------------------ */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * نطبع هنا إن الـ sync خلصت — مفيد لو حبينا نحسب الـ latency
     * لو ضفنا ktime_get() في الـ pre-handler وفرقنا هنا.
     * دلوقتي بنكتفي بـ log بسيط.
     */
    pr_info("spidev_probe: spidev_sync returned (pid=%d)\n",
            task_pid_nr(current));
}

/* ------------------------------------------------------------------ */
/* kprobe struct — connects symbol name to our handlers                */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "spidev_sync",   /* exact kernel symbol to hook    */
    .pre_handler  = handler_pre,    /* called before the function     */
    .post_handler = handler_post,   /* called after the function      */
};

/* ------------------------------------------------------------------ */
/* module_init — register the kprobe                                   */
/* ------------------------------------------------------------------ */
static int __init spidev_kprobe_init(void)
{
    int ret;

    /*
     * register_kprobe() يبحث عن الـ symbol في الـ kallsyms table
     * ويضع software breakpoint (int3 على x86) عند بداية الدالة.
     * بيرجع 0 لو نجح، أو error code لو الـ symbol مش موجود أو
     * الـ kprobes مش enabled في الـ config.
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("spidev_probe: register_kprobe failed: %d "
               "(is CONFIG_KPROBES=y and spidev loaded?)\n", ret);
        return ret;
    }

    pr_info("spidev_probe: kprobe planted on spidev_sync @ %p\n",
            kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit — unregister the kprobe before unloading               */
/* ------------------------------------------------------------------ */
static void __exit spidev_kprobe_exit(void)
{
    /*
     * لازم نعمل unregister_kprobe() قبل ما الـ module يتـ unload،
     * لأن لو فضلت الـ breakpoint موجودة في الـ code وبعدين الـ
     * handler اتشال من الـ memory، أي SPI transfer هيعمل kernel panic.
     */
    unregister_kprobe(&kp);
    pr_info("spidev_probe: kprobe removed from spidev_sync\n");
}

module_init(spidev_kprobe_init);
module_exit(spidev_kprobe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on spidev_sync — trace all userspace SPI transfers");
```

---

### شرح كل جزء

#### الـ Includes

```c
#include <linux/kprobes.h>
#include <linux/spi/spi.h>
#include <linux/sched.h>
```

**الـ `kprobes.h`** بيجيب `struct kprobe` وكل الـ API الخاصة بيها. **الـ `spi.h`** محتاجينه عشان نـ cast الـ pointer لـ `struct spi_message` ونوصل لـ `spi->controller->bus_num`. **الـ `sched.h`** بيجيب `current` اللي بيديك الـ `task_struct` للـ process الشغالة دلوقتي.

---

#### اختيار الـ Function

**الـ `spidev_sync`** مش exported بـ `EXPORT_SYMBOL` لكن ده مش مشكلة للـ kprobes — الـ `register_kprobe()` بيدور على الاسم في الـ `kallsyms` table اللي بتشمل كل الـ symbols حتى الـ static منها (لو الـ kernel اتبنى بـ `CONFIG_KALLSYMS_ALL=y`).

لو الـ symbol مش ظاهر، ممكن تتحقق بـ:
```bash
grep spidev_sync /proc/kallsyms
```

---

#### الـ `handler_pre` — قلب الـ Module

```c
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
```

**الـ `pt_regs`** بيحتفظ بـ snapshot من الـ CPU registers لحظة ما الـ breakpoint اتضرب. على x86-64، الـ `rsi` بيحمل الـ argument التاني (الـ `spi_message *message`) وعلى ARM64 الـ `x1`. بـ cast بسيط بنوصل للـ struct وبنقرأ منها:

| Field | المعنى |
|---|---|
| `msg->spi->controller->bus_num` | رقم الـ SPI bus (0, 1, 2...) |
| `spi_get_chipselect(msg->spi, 0)` | رقم الـ chip-select |
| `xfer->len` in loop | حجم كل segment |
| `current->comm` | اسم الـ process اللي عمل الـ transfer |

المشي على `msg->transfers` list بيدينا العدد الحقيقي للـ transfers والبايتات — أدق من `frame_length` اللي ممكن يكون zero في مرحلة مبكرة.

---

#### الـ `handler_post`

بيتنفذ بعد ما `spidev_sync` ترجع. مفيده لو حبينا نحسب الـ latency بـ `ktime_get()` في الـ pre ونطرح في الـ post، أو نشوف الـ return value في `regs->ax`.

---

#### الـ `module_init` / `module_exit`

`register_kprobe()` بيضع **software breakpoint** (int3 على x86) في أول instruction للدالة. لما تـ unload الـ module لازم تـ call `unregister_kprobe()` عشان الـ kernel يشيل الـ breakpoint ويرجع الـ original bytes — من غير كده أي call للدالة بعد الـ unload هيـ crash الـ system.

---

### Makefile

```makefile
obj-m += spidev_kprobe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### تجربة الـ Module

```bash
# بناء وتحميل
make
sudo insmod spidev_kprobe.ko

# تشغيل أي برنامج يفتح /dev/spidev0.0 ويكتب بيانات
python3 -c "
import spidev
spi = spidev.SpiDev()
spi.open(0, 0)
spi.xfer2([0xAB, 0xCD, 0xEF])
spi.close()
"

# قراءة الـ log
sudo dmesg | tail -20

# تفريغ الـ module
sudo rmmod spidev_kprobe
```

**مثال للـ output المتوقع:**

```
[  142.331204] spidev_probe: kprobe planted on spidev_sync @ ffffffffc08a1230
[  145.887601] spidev_probe: [python3 pid=3412] SPI0.0 — 1 transfer(s), 3 byte(s) total, speed=500000 Hz
[  145.887634] spidev_probe: spidev_sync returned (pid=3412)
```

---

### ملاحظات مهمة

- الـ module ده **read-only** — مش بيعدل في الـ transfer ولا في الـ data.
- لو `spidev_sync` اتـ inline من الـ compiler مش هتلاقيها في الـ kallsyms — في الكود الحالي هي `static` مش `inline` فمفيش مشكلة.
- على أنظمة بـ `CONFIG_KPROBES_ON_FTRACE=y` ممكن يستخدم الـ kernel الـ ftrace infrastructure بدل الـ int3 — أسرع وأقل overhead.
