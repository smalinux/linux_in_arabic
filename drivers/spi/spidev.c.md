## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الـ `spidev.c` جزء من **SPI Subsystem** في الـ Linux kernel. المسؤول عنه هو Mark Brown، والـ mailing list هو `linux-spi@vger.kernel.org`، وكل الملفات تحت `drivers/spi/` تنتمي لنفس الـ subsystem.

---

### ما هو الـ SPI أصلاً؟

**SPI (Serial Peripheral Interface)** هو بروتوكول اتصال سريع بين الـ CPU وأجهزة الـ hardware الصغيرة زي:
- شاشات OLED صغيرة
- sensors (حرارة، ضغط، accelerometer)
- flash memory chips
- ADC/DAC converters
- شرائح wireless زي sx1301

الـ SPI يشتغل بـ 4 أسلاك:
```
CPU (Master)          Chip (Slave)
─────────────────────────────────
MOSI ─────────────────────────► (data out → in)
MISO ◄───────────────────────── (data in ← out)
SCLK ─────────────────────────► (clock)
CS   ─────────────────────────► (chip select, active low)
```

الميزة الأساسية للـ SPI إنه **full-duplex** — بيبعت ويستقبل في نفس اللحظة على عكس I2C أو UART.

---

### المشكلة اللي بتحلها spidev.c

تخيل السيناريو ده:

> عندك Raspberry Pi وعايز تتكلم مع sensor بيشتغل بـ SPI. عادةً محتاج تكتب kernel driver خاص بالـ sensor ده — كود C في الـ kernel، تعمله compile، تحمله module، وهكذا. ده كلام كتير لحاجة بسيطة.

الـ `spidev.c` بيحل المشكلة دي بطريقة مختلفة تماماً:

**بيخلق character device (`/dev/spidev0.0`) بيتعرض لـ userspace**، فالـ application العادية تقدر تتكلم مع أي SPI chip مباشرةً بدون ما تكتب kernel driver خالص.

```
Userspace App
     │
     │  open("/dev/spidev0.0")
     │  read() / write() / ioctl()
     ▼
[ spidev.c — character device driver ]
     │
     │  spi_sync()
     ▼
[ SPI Core — spi.c ]
     │
     ▼
[ SPI Controller Driver — e.g. spi-bcm2835.c ]
     │
     ▼
[ Physical SPI Hardware ]
     │
     ▼
[ Your Sensor/Chip ]
```

---

### القصة الكاملة: كيف بيشتغل؟

#### 1. عند تشغيل الـ Kernel
- الـ `spidev_init()` بتسجل **character device** برقم major = 153.
- بتسجل **class** اسمها `spidev` عشان `udev` يعرف يعمل `/dev` nodes تلقائياً.
- بتسجل **SPI driver** اسمه `spidev_spi_driver`.

#### 2. عند اكتشاف الـ Hardware
لما الـ kernel يلاقي SPI device في الـ Device Tree أو ACPI بـ compatible string زي `"rohm,dh2228fv"`، بيستدعي `spidev_probe()`:
- بيخصص struct `spidev_data` جديد في الـ memory.
- بيحجز **minor number** فريد من بيتماب `minors`.
- بيعمل device node اسمه `spidev<bus>.<chipselect>` زي `/dev/spidev0.0`.

#### 3. لما الـ App تفتح الـ Device
`spidev_open()` بتخصص **TX buffer** و **RX buffer** (الـ default 4096 bytes لكل واحد) في الـ kernel memory.

#### 4. التواصل الفعلي
للـ userspace عندها 3 طرق تتواصل:

| الطريقة | الاستخدام |
|---------|-----------|
| `write()` | إرسال بيانات فقط (half-duplex TX) |
| `read()` | استقبال بيانات فقط (half-duplex RX) |
| `ioctl(SPI_IOC_MESSAGE)` | إرسال واستقبال في نفس الوقت (full-duplex) |

الـ `ioctl` هو الأقوى — بيقبل **array of transfers**، كل transfer ممكن يبعت ويستقبل في نفس الوقت، وبيتدي control كامل على:
- الـ `speed_hz` لكل transfer
- الـ `bits_per_word`
- الـ `cs_change` (فصل الـ chip select بين transfers)
- الـ `delay_usecs`

#### 5. الـ Bounce Buffer وأمان الـ DMA
الـ kernel ما يقدرش يستخدم pointers الـ userspace مباشرةً مع الـ DMA. عشان كده `spidev_message()` بتعمل **bounce buffers**:
- بتنسخ TX data من userspace للـ kernel buffer.
- بتدي الـ SPI core pointers للـ kernel buffers.
- بعد الـ transfer، بتنسخ RX data من الـ kernel buffer للـ userspace.

---

### تحذيرات مهمة في الكود نفسه

الكود صريح جداً في التحذيرات:

1. **الـ ACPI devices** (`SPT0001`, `SPT0002`, `SPT0003`) موجودة للـ development بس، ومكتوب في الكود: `"do not use this driver in production systems!"`.

2. **الـ Device Tree**: ممنوع تكتب `compatible = "spidev"` مباشرةً في الـ DT — الكود يرفضها explictly في `spidev_of_check()`. لازم تستخدم الـ compatible string الخاصة بالـ chip الفعلي.

3. **السبب**: الـ spidev مش بديل عن الـ proper kernel drivers في الـ production. هو أداة للـ prototyping والـ testing.

---

### مثال عملي: Python + spidev

```python
import spidev

spi = spidev.SpiDev()
spi.open(0, 0)                    # /dev/spidev0.0
spi.max_speed_hz = 1000000        # 1 MHz

# Full-duplex transfer: send [0x01, 0x02] and read 2 bytes simultaneously
response = spi.xfer2([0x01, 0x02])
print(response)

spi.close()
```

ده بيشتغل بسبب `spidev.c` — الـ Python library بتستخدم `/dev/spidev0.0` وبتعمل `ioctl(SPI_IOC_MESSAGE)` في الخلفية.

---

### الـ ioctl Commands المتاحة لـ Userspace

| Command | الوظيفة |
|---------|---------|
| `SPI_IOC_RD_MODE` / `SPI_IOC_WR_MODE` | قراءة/كتابة الـ SPI mode (0-3) |
| `SPI_IOC_RD_MODE32` / `SPI_IOC_WR_MODE32` | نفس الفكرة بـ 32-bit |
| `SPI_IOC_RD_LSB_FIRST` / `SPI_IOC_WR_LSB_FIRST` | اتجاه الـ bits |
| `SPI_IOC_RD_BITS_PER_WORD` / `SPI_IOC_WR_BITS_PER_WORD` | حجم الـ word |
| `SPI_IOC_RD_MAX_SPEED_HZ` / `SPI_IOC_WR_MAX_SPEED_HZ` | السرعة القصوى |
| `SPI_IOC_MESSAGE(N)` | إرسال N transfers دفعة واحدة |

---

### الملفات المكونة للـ Subsystem

#### الـ Core
| الملف | الدور |
|-------|-------|
| `drivers/spi/spi.c` | قلب الـ SPI subsystem — الـ registration والـ scheduling والـ spi_sync |
| `drivers/spi/spidev.c` | الـ userspace interface (الملف الحالي) |
| `drivers/spi/spi-mem.c` | واجهة خاصة لـ SPI memory (NOR flash, NAND) |
| `drivers/spi/spi-offload.c` | hardware offload للـ high-speed transfers |

#### الـ Headers
| الملف | الدور |
|-------|-------|
| `include/linux/spi/spi.h` | تعريفات `spi_device`, `spi_controller`, `spi_transfer`, `spi_message` |
| `include/uapi/linux/spi/spidev.h` | الـ userspace API — تعريف `spi_ioc_transfer` وكل الـ ioctl constants |
| `include/uapi/linux/spi/spi.h` | الـ SPI mode flags المشتركة بين kernel وuserspace |

#### أمثلة على الـ Hardware Drivers
| الملف | الـ Hardware |
|-------|------------|
| `drivers/spi/spi-bcm2835.c` | Raspberry Pi SPI |
| `drivers/spi/spi-imx.c` | i.MX processors |
| `drivers/spi/spi-dw-core.c` | DesignWare SPI IP |
| `drivers/spi/spi-gpio.c` | Bit-bang SPI بـ GPIO |
| `drivers/spi/spi-pl022.c` | ARM PrimeCell SSP |

#### الـ Documentation
| الملف | المحتوى |
|-------|---------|
| `Documentation/spi/spidev.rst` | كيفية استخدام `/dev/spidevB.C` |
| `Documentation/spi/spi-summary.rst` | نظرة عامة على الـ SPI subsystem |
| `Documentation/devicetree/bindings/spi/` | الـ DT bindings لكل الـ controllers |
## Phase 2: شرح الـ SPI Framework

### المشكلة — ليه الـ SPI Framework موجود أصلاً؟

الـ **SPI (Serial Peripheral Interface)** بروتوكول hardware بسيط: 4 أسلاك (MOSI, MISO, SCK, CS)، full-duplex، synchronous. الـ SoC بتاعك (زي i.MX6، AM335x، BCM2835) فيه hardware controller بيتحكم في الـ SPI bus. على البص ده ممكن يكون فيه flash chip، ADC، display controller، sensor، إلخ.

**المشكلة الأولى:** كل SoC عنده SPI controller مختلف تمامًا — registers مختلفة، DMA setup مختلف، interrupt handling مختلف. لو كل device driver (زي driver الـ W25Q flash) اتكلم مع الـ hardware مباشرةً، كان هيبقى مربوط بـ SoC معين وهينكسر على أي hardware تاني.

**المشكلة التانية:** عندك أكتر من device على نفس البص (SPI bus shared). مين اللي بيتحكم في الـ chip select؟ مين اللي بيعمل arbitration لو اتنين drivers عايزين يتكلموا في نفس الوقت؟

**المشكلة التالتة:** الـ userspace applications مش قادرة تتكلم مع SPI devices من غير driver مخصص لكل device. في prototyping وفي testing، ده overhead كبير.

**الحل اللي الـ kernel اتبناه:**

الـ SPI framework بيعمل **فصل واضح** بين 3 طبقات:

1. **Controller driver** — بيعرف يتكلم مع الـ SPI hardware registers على SoC معين
2. **Protocol/device driver** — بيعرف بروتوكول الـ device (flash، sensor، إلخ) بس مش مهتم بالـ hardware
3. **SPI core** — الوسيط اللي بيربطهم ويعمل arbitration وقueueing

**الـ spidev driver** هو حالة خاصة: بدل ما يكتب protocol driver في kernel، بيعمل **character device** (`/dev/spidevB.C`) يخلي الـ userspace يتكلم مع أي SPI device مباشرةً عن طريق `read/write/ioctl`.

---

### Architecture — صورة كبيرة

```
┌─────────────────────────────────────────────────────────────────┐
│                        USERSPACE                                │
│                                                                 │
│   app.py          test_adc.c         spidev_test               │
│      │                │                    │                    │
│   open("/dev/spidev0.0")            open("/dev/spidev1.2")     │
└──────┼────────────────┼────────────────────┼────────────────────┘
       │ syscall        │ syscall             │ syscall
═══════╪════════════════╪═════════════════════╪═══════════════════
       │                │                    │
┌──────▼────────────────▼────────────────────▼────────────────────┐
│                    VFS Layer                                     │
│           (file_operations dispatch)                             │
│        spidev_fops: read/write/ioctl/open/release               │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                  spidev driver                                   │
│              (drivers/spi/spidev.c)                             │
│                                                                 │
│  struct spidev_data                                             │
│  ┌─────────────┐    bounce buffers    ┌──────────────────────┐  │
│  │  tx_buffer  │◄──────────────────── │ copy_from_user()     │  │
│  │  rx_buffer  │──────────────────────► copy_to_user()       │  │
│  └──────┬──────┘                      └──────────────────────┘  │
│         │ spi_message + spi_transfer                            │
└─────────┼───────────────────────────────────────────────────────┘
          │ spi_sync()
┌─────────▼───────────────────────────────────────────────────────┐
│                    SPI Core                                      │
│              (drivers/spi/spi.c)                                │
│                                                                 │
│  • Message queueing (FIFO per device)                           │
│  • Chip select management                                       │
│  • DMA mapping                                                  │
│  • Bus locking / arbitration                                    │
│  • spi_bus_type registration                                    │
└─────────┬───────────────────────────────────────────────────────┘
          │ transfer_one_message() / transfer_one()
┌─────────▼───────────────────────────────────────────────────────┐
│               SPI Controller Driver                              │
│    (e.g. drivers/spi/spi-imx.c  /  spi-bcm2835.c)             │
│                                                                 │
│  struct spi_controller                                          │
│  • setup()          — configure clock, mode, CS timing         │
│  • transfer_one()   — push one spi_transfer to hardware        │
│  • set_cs()         — assert/deassert chip select              │
│  • can_dma()        — DMA capability check                     │
└─────────┬───────────────────────────────────────────────────────┘
          │ MMIO / DMA / interrupt
┌─────────▼───────────────────────────────────────────────────────┘
│              SPI Hardware (SoC registers)                        │
│         SCK ──── MOSI ──── MISO ──── CS0 ── CS1 ── CS2         │
└─────────────────────────────────────────────────────────────────┘
          │            │           │
     ┌────┴───┐  ┌─────┴──┐  ┌────┴────┐
     │W25Q128 │  │ MAX3100 │  │ ADS1256 │
     │(flash) │  │ (UART)  │  │  (ADC)  │
     └────────┘  └────────┘  └─────────┘
```

---

### التشبيه الواقعي — مطعم كبير بـ kitchen واحدة

تخيل مطعم فيه:

| عنصر المطعم | المقابل في SPI Framework |
|---|---|
| الـ kitchen (طباخين + معدات) | `spi_controller` — الـ hardware + controller driver |
| الـ waiter | `spi core` — بياخد الطلبات ويرتبها ويوديها للـ kitchen |
| الـ menu item (طلب) | `spi_message` — transaction كامل |
| كل صنف في الطلب | `spi_transfer` — قراءة أو كتابة واحدة |
| الزبون في الصالة | `protocol driver` أو `spidev userspace app` |
| تذكرة الطلب | الـ `struct spi_ioc_transfer` — تفاصيل كل خطوة |
| رقم الطاولة | `chip select` — بيحدد أي device على البص |

**التعمق في التشبيه:**

- **الـ kitchen مش بتعرف إيه الأكل** — controller driver بيعرف يبعت bytes على السلك بس مش مهتم إيه معناها. زي الطباخ اللي بيطبخ recipes بدون ما يعرف مين الزبون.
- **الـ waiter بيمنع التزاحم** — لو اتنين زبائن طلبوا في نفس الوقت، الـ waiter بيرتبهم FIFO. الـ SPI core كذلك بيعمل `io_mutex` و `bus_lock` على `spi_controller`.
- **تذكرة الطلب بتتحول** — الزبون بيكتب طلبه على ورقة (userspace: `struct spi_ioc_transfer`)، الـ waiter بيحولها لصيغة المطبخ (kernel: `struct spi_transfer`). ده بالظبط اللي بيعمله `spidev_message()` — بيحول من `spi_ioc_transfer[]` لـ `spi_transfer[]`.
- **الطاولة لازم تتحجز** — chip select لازم يتأكد إنه low (أو high لو `SPI_CS_HIGH`) طول فترة الـ message. الـ core بيتحكم في ده.
- **الـ kitchen ممكن تستخدم dishwasher** — لو الطلب كبير، الـ controller ممكن يستخدم DMA بدل CPU. الـ core بيقرر ده بناءً على `can_dma()`.

---

### الـ Core Abstractions — الأفكار المحورية

الـ SPI framework مبني على **3 structs محورية** بتشكل حلقة متكاملة:

```
spi_message
    │
    │  transfers (linked list)
    │
    ├──► spi_transfer [0]   tx_buf/rx_buf, len, speed_hz, cs_change
    ├──► spi_transfer [1]   tx_buf/rx_buf, len, speed_hz, delay
    └──► spi_transfer [N]   tx_buf/rx_buf, len, bits_per_word

         كلهم بيروحوا على:

spi_device  ──────────────►  spi_controller
  (الـ "proxy" للـ chip)        (الـ hardware abstraction)
  max_speed_hz                   transfer_one_message()
  mode (CPOL/CPHA)               set_cs()
  bits_per_word                  can_dma()
  chip_select[]                  setup()
```

#### struct spi_transfer — وحدة النقل الأساسية

```c
struct spi_transfer {
    const void  *tx_buf;      /* بيانات الإرسال (DMA-safe) */
    void        *rx_buf;      /* بيانات الاستقبال (DMA-safe) */
    unsigned    len;          /* عدد الـ bytes */

    u8          bits_per_word;  /* override لـ word size */
    u32         speed_hz;       /* override لـ clock speed */
    unsigned    cs_change:1;    /* هل تعمل toggle للـ CS بعدها؟ */
    unsigned    tx_nbits:4;     /* عدد lanes للإرسال (1x/2x/4x/8x) */
    unsigned    rx_nbits:4;     /* عدد lanes للاستقبال */

    struct spi_delay  delay;        /* تأخير بعد الـ transfer */
    struct spi_delay  word_delay;   /* تأخير بين كل word */

    struct list_head  transfer_list; /* ربط في الـ spi_message */
};
```

**ملاحظة مهمة:** الـ `tx_buf` و `rx_buf` لازم يكونوا DMA-safe (يعني مش على الـ stack). لذلك الـ spidev بيستخدم **bounce buffers** مخصصة — بياخد البيانات من userspace بـ `copy_from_user()` في buffer في kernel memory أولاً.

#### struct spi_message — الـ Transaction الكامل

```c
struct spi_message {
    struct list_head  transfers;    /* linked list of spi_transfer */
    struct spi_device *spi;         /* الـ device المستهدف */

    int               status;       /* نتيجة التنفيذ */
    void (*complete)(void *context); /* callback لما يخلص */
    void              *context;

    unsigned          frame_length;  /* إجمالي الـ bytes */
    unsigned          actual_length; /* اللي اتنقل فعلاً */
};
```

الـ message هي **وحدة الـ atomicity** — طول ما الـ message شغالة، الـ chip select مثبت والبص محجوز. مفيش device تاني يقدر يستخدم البص.

#### struct spi_device — الـ Proxy للـ Chip

```c
struct spi_device {
    struct device       dev;          /* device model integration */
    struct spi_controller *controller; /* الـ controller اللي عليه */

    u32   max_speed_hz;    /* أقصى سرعة */
    u8    bits_per_word;   /* حجم الـ word */
    u32   mode;            /* CPOL, CPHA, CS_HIGH, LSB_FIRST... */
    int   irq;             /* interrupt line */

    u8    chip_select[SPI_DEVICE_CS_CNT_MAX]; /* physical CS lines */
};
```

#### struct spi_controller — الـ Hardware Abstraction

```c
struct spi_controller {
    struct device  dev;
    s16            bus_num;         /* رقم البص (SPI0, SPI1...) */
    u16            num_chipselect;  /* عدد الـ CS lines */
    u32            mode_bits;       /* الـ modes المدعومة */
    u32            min_speed_hz;
    u32            max_speed_hz;

    /* الـ callbacks اللي controller driver بيعملها */
    int  (*setup)(struct spi_device *spi);
    int  (*transfer_one)(struct spi_controller *ctlr,
                         struct spi_device *spi,
                         struct spi_transfer *transfer);
    void (*set_cs)(struct spi_device *spi, bool enable);
    bool (*can_dma)(struct spi_controller *ctlr,
                    struct spi_device *spi,
                    struct spi_transfer *xfer);

    /* الـ message queue */
    struct list_head    queue;
    struct spi_message *cur_msg;
    struct kthread_worker *kworker;   /* الـ pump thread */
};
```

---

### تدفق البيانات في spidev — خطوة بخطوة

#### السيناريو: userspace بتعمل `ioctl(fd, SPI_IOC_MESSAGE(2), transfers)`

```
userspace                kernel (spidev)           SPI core        controller
─────────                ───────────────           ────────        ──────────

ioctl(fd,
  SPI_IOC_MESSAGE(2),
  u_xfers)
        │
        └─────────────► spidev_ioctl()
                              │
                         spidev_get_ioc_message()
                         memdup_user(u_xfers)       ← copy from userspace
                              │
                         spidev_message()
                              │
                         for each u_xfer:
                           copy_from_user(
                             tx_buf,                ← copy TX data
                             u_tmp->tx_buf)
                           build spi_transfer
                           spi_message_add_tail()
                              │
                         spidev_sync_unlocked()
                              │
                              └──────────────────► spi_sync()
                                                        │
                                                   add to queue
                                                   wake kworker
                                                        │
                                                        └────────► transfer_one_message()
                                                                         │
                                                                    set_cs(low)
                                                                    transfer_one()
                                                                    ← wait completion
                                                                    set_cs(high)
                                                                         │
                                                   complete ◄────────────┘
                              │                        │
                         copy_to_user(                 │    ← copy RX data back
                           u_tmp->rx_buf,              │
                           k_tmp->rx_buf)              │
                              │
                         return total bytes
        │
        ◄─────────────── return to userspace
```

---

### الـ spidev_data struct — قلب الـ Driver

```c
struct spidev_data {
    dev_t            devt;         /* major:minor number */
    struct mutex     spi_lock;     /* يحمي الـ spi pointer من الـ removal */
    struct spi_device *spi;        /* الـ SPI device المرتبط */
    struct list_head  device_entry; /* في الـ global device_list */

    struct mutex     buf_lock;     /* يحمي TX/RX buffers */
    unsigned         users;        /* عدد الـ open file handles */
    u8              *tx_buffer;    /* bounce buffer للإرسال */
    u8              *rx_buffer;    /* bounce buffer للاستقبال */
    u32              speed_hz;     /* السرعة الحالية */
};
```

**ليه اتنين mutex؟**

- **`spi_lock`**: بيحمي من حالة لو الـ SPI device اتشال (device removal) بينما في عملية جارية. لما `spidev_remove()` بتتنادى، بتعمل `spidev->spi = NULL`. الـ `spidev_sync()` بتشيك الـ pointer دي تحت الـ `spi_lock` عشان تعرف تبعت `ESHUTDOWN-.`
- **`buf_lock`**: بيحمي الـ TX/RX buffers من concurrent access — سواء من `read()`/`write()` أو من `ioctl()`.

الاتنين mutex مش نفس الحاجة وتداخلهم ممكن يعمل deadlock لو الترتيب اتقلب، ولذلك الـ code ملتزم دايمًا بترتيب واحد: `spi_lock` أولاً، `buf_lock` تاني.

---

### الـ Bounce Buffers — ليه موجودة؟

الـ DMA hardware مش قادر يتكلم مع الـ userspace memory مباشرةً لأسباب:

1. **Virtual vs Physical addresses** — الـ userspace pointer هو virtual address، ممكن مش contiguous في physical memory
2. **Page faults** — الـ DMA ما بيعرفش يـhandle page faults
3. **Cache coherency** — لو الـ buffer في userspace مش cache-aligned، ممكن يحصل data corruption

**الحل في spidev:**

```
userspace buffer  ──copy_from_user()──►  tx_buffer (kernel, DMA-safe)
                                               │
                                          spi_transfer.tx_buf
                                               │
                                          DMA engine ──► SPI hardware

SPI hardware ──► DMA engine ──► rx_buffer (kernel, DMA-safe)
                                      │
                      copy_to_user() ──► userspace buffer
```

الـ `bufsiz` (default 4096 bytes) بيتحكم في حجم الـ bounce buffers دي. قابل للتغيير عن طريق `modparam`:

```bash
modprobe spidev bufsiz=65536
```

---

### الـ IOCTL Interface — الـ API الكامل

الـ `spidev` بيعرض 3 أنواع من الـ operations:

#### 1. Simple Read/Write

```c
/* write-only: single transfer, TX only */
write(fd, data, len);

/* read-only: single transfer, RX only */
read(fd, buf, len);
```

الاتنين بيستخدموا الـ speed_hz الحالي وبيعملوا transfer واحد.

#### 2. Configuration IOCTLs

```c
/* قراءة/كتابة الـ SPI mode */
ioctl(fd, SPI_IOC_RD_MODE,  &mode);   /* 8-bit */
ioctl(fd, SPI_IOC_WR_MODE,  &mode);   /* 8-bit: SPI_MODE_0..3 */
ioctl(fd, SPI_IOC_RD_MODE32, &mode);  /* 32-bit: كل الـ flags */
ioctl(fd, SPI_IOC_WR_MODE32, &mode);

/* قراءة/كتابة bits per word */
ioctl(fd, SPI_IOC_RD_BITS_PER_WORD, &bits);
ioctl(fd, SPI_IOC_WR_BITS_PER_WORD, &bits);

/* قراءة/كتابة max speed */
ioctl(fd, SPI_IOC_RD_MAX_SPEED_HZ, &speed);
ioctl(fd, SPI_IOC_WR_MAX_SPEED_HZ, &speed);

/* LSB first */
ioctl(fd, SPI_IOC_RD_LSB_FIRST, &lsb);
ioctl(fd, SPI_IOC_WR_LSB_FIRST, &lsb);
```

كل write IOCTL بيعمل `spi_setup(spi)` اللي بيوصل للـ controller driver عشان يحدث الـ hardware settings.

#### 3. Full-Duplex / Multi-Transfer Message

```c
struct spi_ioc_transfer xfers[2] = {
    {
        .tx_buf = (uint64_t)cmd_buf,
        .len    = 1,
        .cs_change = 0,    /* خلي CS active بين الـ transfers */
    },
    {
        .rx_buf = (uint64_t)rx_buf,
        .len    = 4,
        .speed_hz = 1000000,  /* override السرعة لهذا التransfer */
    },
};

ioctl(fd, SPI_IOC_MESSAGE(2), xfers);
```

الـ macro `SPI_IOC_MESSAGE(N)` بيحسب الـ ioctl command number بناءً على عدد الـ transfers:

```c
#define SPI_MSGSIZE(N) \
    ((((N)*(sizeof(struct spi_ioc_transfer))) < (1 << _IOC_SIZEBITS)) \
        ? ((N)*(sizeof(struct spi_ioc_transfer))) : 0)

#define SPI_IOC_MESSAGE(N) _IOW(SPI_IOC_MAGIC, 0, char[SPI_MSGSIZE(N)])
```

يعني الـ size بيتشفر جوه الـ ioctl command نفسه — الـ kernel بيستخرجه بـ `_IOC_SIZE(cmd)` ويقسمه على `sizeof(struct spi_ioc_transfer)` عشان يعرف عدد الـ transfers.

---

### Device Naming — إزاي بتعرف الـ /dev اسمه إيه؟

```c
/* في spidev_probe(): */
dev = device_create(&spidev_class, &spi->dev, spidev->devt,
                    spidev, "spidev%d.%d",
                    spi->controller->bus_num,        /* رقم الـ bus */
                    spi_get_chipselect(spi, 0));      /* رقم الـ CS */
```

**مثال:** SPI controller رقم 0، device على CS رقم 0:
```
/dev/spidev0.0
```

SPI controller رقم 1، device على CS رقم 2:
```
/dev/spidev1.2
```

الـ major number ثابت (153)، الـ minor number dynamic بيتتعين من bitmap:

```c
#define SPIDEV_MAJOR    153
#define N_SPI_MINORS    32

static DECLARE_BITMAP(minors, N_SPI_MINORS);
```

---

### Device Tree / ACPI / SPI IDs — إزاي بيتعرف على الـ device؟

الـ spidev driver عنده 3 طرق للـ matching:

**1. Device Tree (`spidev_dt_ids`):**

```dts
/* في الـ DTS file */
&spi0 {
    ltc2488: adc@0 {
        compatible = "lineartechnology,ltc2488";
        reg = <0>;          /* chip select 0 */
        spi-max-frequency = <1000000>;
    };
};
```

```c
static const struct of_device_id spidev_dt_ids[] = {
    { .compatible = "lineartechnology,ltc2488",
      .data = &spidev_of_check },
    /* ... */
};
```

الـ `spidev_of_check` بتمنع استخدام `compatible = "spidev"` مباشرةً في الـ DTS لأنه deprecated وgenerally خطر على production.

**2. SPI Device ID Table (`spidev_spi_ids`):**

```c
static const struct spi_device_id spidev_spi_ids[] = {
    { .name = "ltc2488" },
    { .name = "sx1301"  },
    /* ... */
};
```

**3. ACPI (`spidev_acpi_ids`):**

للـ x86/UEFI systems. الـ ACPI IDs (SPT0001..SPT0003) مخصصة للـ development فقط وبتعمل warning لما تستخدمها.

---

### الـ Lifecycle — من probe لـ remove

```
spi_register_driver()
        │
        ▼ (kernel matches device)
spidev_probe(spi)
        │
        ├─ kzalloc(spidev_data)
        ├─ spidev->spi = spi
        ├─ mutex_init(spi_lock, buf_lock)
        ├─ find_first_zero_bit(minors)   ← minor number
        ├─ MKDEV(153, minor)             ← devt
        ├─ device_create(spidev_class)   ← يخلي udev يعمل /dev/spidevB.C
        ├─ list_add(&device_entry)       ← في global device_list
        └─ spi_set_drvdata(spi, spidev)  ← ربط

        │
        ▼ (user opens /dev/spidev0.0)
spidev_open(inode, filp)
        │
        ├─ list_for_each_entry ← دور على spidev بالـ devt
        ├─ kmalloc(tx_buffer)
        ├─ kmalloc(rx_buffer)
        ├─ spidev->users++
        └─ filp->private_data = spidev

        │
        ▼ (user closes fd)
spidev_release(inode, filp)
        │
        ├─ spidev->users--
        ├─ if users == 0:
        │    kfree(tx_buffer)
        │    kfree(rx_buffer)
        │    if spi == NULL: kfree(spidev)  ← already removed
        └─ spi_target_abort() if SPI_SLAVE

        │
        ▼ (device removed/driver unloaded)
spidev_remove(spi)
        │
        ├─ spidev->spi = NULL   ← يخلي الـ open handles تاخد ESHUTDOWN
        ├─ list_del(&device_entry)
        ├─ device_destroy(spidev_class, devt)  ← udev يشيل /dev/spidevB.C
        ├─ clear_bit(minor, minors)
        └─ if users == 0: kfree(spidev)
```

**ملاحظة لطيفة:** لو الـ device اتشال وفيه file still open، الـ spidev_data مش بتتحذف فوراً — بتفضل موجودة لحد ما الـ last close. الـ `spidev->spi = NULL` بيعمل signal للـ ongoing operations إنها ترجع `ESHUTDOWN-`.

---

### الـ Framework ملكه إيه مقابل إيه بيـ Delegate للـ Drivers؟

| المسؤولية | صاحبها |
|---|---|
| **قبول requests من userspace** | spidev (VFS layer) |
| **Bounce buffer management** | spidev |
| **User/kernel memory copy** | spidev |
| **spi_message building** | spidev |
| **Message queueing وارbitration** | SPI core |
| **Chip select timing** | SPI core + controller driver |
| **DMA mapping** | SPI core (إذا `can_dma()` = true) |
| **Clock configuration** | Controller driver |
| **Hardware registers** | Controller driver |
| **Transfer execution** | Controller driver (`transfer_one()`) |
| **Protocol logic** | **لا أحد** — ده مسؤولية الـ userspace app |

الـ spidev بيتعمد **مش يفهم** البروتوكول — هو مجرد pipe من userspace للـ SPI framework. فهم البروتوكول (مثلاً: إزاي تقرأ من LTC2488 ADC) مسؤولية الـ application فوق.

---

### الفرق الجوهري: spi_transfer vs spi_ioc_transfer

الاتنين بيمثلوا نفس الفكرة (transfer واحد) بس في address spaces مختلفة:

```c
/* userspace-facing (uapi): fixed layout, 64-bit pointers */
struct spi_ioc_transfer {
    __u64  tx_buf;       /* userspace pointer كـ integer */
    __u64  rx_buf;
    __u32  len;
    __u32  speed_hz;
    __u16  delay_usecs;
    __u8   bits_per_word;
    __u8   cs_change;
    __u8   tx_nbits;
    __u8   rx_nbits;
    __u8   word_delay_usecs;
    __u8   pad;          /* explicit padding — no surprises */
};

/* kernel-facing: richer, DMA-aware */
struct spi_transfer {
    const void *tx_buf;  /* kernel pointer، DMA-safe */
    void       *rx_buf;
    unsigned    len;
    u32         speed_hz;
    u8          bits_per_word;
    unsigned    cs_change:1;
    /* + DMA scatterlist, PTP timestamps, delay structs... */
    struct list_head transfer_list;  /* ربطها في spi_message */
};
```

الـ `spidev_message()` هي اللي بتعمل الترجمة بين الاتنين — ودي نقطة حرجة لأنها كمان بتعمل الـ bounce buffer assignment وبتشيل الـ DMA alignment.

---

### الـ 32-bit Compat Layer

الـ kernel بيدعم running 32-bit userspace على 64-bit kernel. المشكلة: الـ pointers في `spi_ioc_transfer` هي `__u64` دايمًا (عشان الـ struct layout يبقى consistent)، بس لما الـ userspace 32-bit، الـ pointer values بتكون 32-bit.

```c
#ifdef CONFIG_COMPAT
static long spidev_compat_ioc_message(struct file *filp, ...)
{
    /* Convert 32-bit pointers to 64-bit */
    for (n = 0; n < n_ioc; n++) {
        ioc[n].rx_buf = (uintptr_t) compat_ptr(ioc[n].rx_buf);
        ioc[n].tx_buf = (uintptr_t) compat_ptr(ioc[n].tx_buf);
    }
    /* ثم اتعامل عادي */
    retval = spidev_message(spidev, ioc, n_ioc);
}
#endif
```

الـ `compat_ptr()` بيحول الـ 32-bit pointer المحفوظ في 64-bit field لـ kernel pointer صح.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options

#### SPI Mode Flags (من `uapi/linux/spi/spi.h`)

| Flag | Bit | المعنى |
|------|-----|--------|
| `SPI_CPHA` | 0 | Clock phase — بيحدد امتى البيانات تتسامبل |
| `SPI_CPOL` | 1 | Clock polarity — بيحدد الـ idle state للـ clock |
| `SPI_CS_HIGH` | 2 | الـ chip select بيكون active high بدل low |
| `SPI_LSB_FIRST` | 3 | بيبعت الـ LSB الأول بدل الـ MSB |
| `SPI_3WIRE` | 4 | الـ MOSI والـ MISO بيشتركوا في wire واحد |
| `SPI_LOOP` | 5 | Loopback mode للتست |
| `SPI_NO_CS` | 6 | مفيش chip select — جهاز واحد على الـ bus |
| `SPI_READY` | 7 | الـ slave بيشد الـ signal low عشان يوقف |
| `SPI_TX_DUAL` | 8 | إرسال على 2 wires |
| `SPI_TX_QUAD` | 9 | إرسال على 4 wires |
| `SPI_TX_OCTAL` | 13 | إرسال على 8 wires |
| `SPI_RX_DUAL` | 10 | استقبال على 2 wires |
| `SPI_RX_QUAD` | 11 | استقبال على 4 wires |
| `SPI_RX_OCTAL` | 14 | استقبال على 8 wires |
| `SPI_3WIRE_HIZ` | 15 | High-Z turnaround في الـ 3-wire mode |
| `SPI_RX_CPHA_FLIP` | 16 | عكس الـ CPHA على الـ RX فقط |
| `SPI_MOSI_IDLE_LOW` | 17 | MOSI بيبقى low وهو idle |
| `SPI_MODE_USER_MASK` | bits 0–18 | كل الـ bits اللي المستخدم مسموحله يعدلها |

#### SPI Mode الجاهزة

| Mode | CPOL | CPHA | الاستخدام الشائع |
|------|------|------|-----------------|
| `SPI_MODE_0` | 0 | 0 | الأكثر شيوعاً — MicroWire أصلي |
| `SPI_MODE_1` | 0 | 1 | بعض الـ ADCs |
| `SPI_MODE_2` | 1 | 0 | نادر |
| `SPI_MODE_3` | 1 | 1 | بعض الـ flash chips |

#### SPI_MODE_MASK (في spidev.c)

```c
#define SPI_MODE_MASK  (SPI_MODE_X_MASK | SPI_CS_HIGH
                      | SPI_LSB_FIRST | SPI_3WIRE | SPI_LOOP
                      | SPI_NO_CS | SPI_READY | SPI_TX_DUAL
                      | SPI_TX_QUAD | SPI_TX_OCTAL | SPI_RX_DUAL
                      | SPI_RX_QUAD | SPI_RX_OCTAL
                      | SPI_RX_CPHA_FLIP | SPI_3WIRE_HIZ
                      | SPI_MOSI_IDLE_LOW)
```

ده الـ mask اللي spidev بيسمح للـ userspace يعدله — بيحمي الـ kernel-only bits.

#### SPI Controller Flags

| Flag | المعنى |
|------|--------|
| `SPI_CONTROLLER_HALF_DUPLEX` | مينفعش full duplex |
| `SPI_CONTROLLER_NO_RX` | مفيش RX wire |
| `SPI_CONTROLLER_NO_TX` | مفيش TX wire |
| `SPI_CONTROLLER_MUST_RX` | لازم يبقى فيه RX buffer |
| `SPI_CONTROLLER_MUST_TX` | لازم يبقى فيه TX buffer |
| `SPI_CONTROLLER_GPIO_SS` | الـ CS بيتعمل عن طريق GPIO |
| `SPI_CONTROLLER_SUSPENDED` | الـ controller حالياً suspended |
| `SPI_CONTROLLER_MULTI_CS` | بيقدر يشغل أكتر من CS في نفس الوقت |

#### IOCTL Commands (من `uapi/linux/spi/spidev.h`)

| Command | Direction | Size | الوظيفة |
|---------|-----------|------|---------|
| `SPI_IOC_RD_MODE` | Read | u8 | اقرأ الـ mode الحالي (8-bit) |
| `SPI_IOC_WR_MODE` | Write | u8 | اكتب الـ mode (8-bit) |
| `SPI_IOC_RD_MODE32` | Read | u32 | اقرأ الـ mode كامل (32-bit) |
| `SPI_IOC_WR_MODE32` | Write | u32 | اكتب الـ mode كامل (32-bit) |
| `SPI_IOC_RD_LSB_FIRST` | Read | u8 | اقرأ اتجاه الـ bits |
| `SPI_IOC_WR_LSB_FIRST` | Write | u8 | اكتب اتجاه الـ bits |
| `SPI_IOC_RD_BITS_PER_WORD` | Read | u8 | اقرأ حجم الـ word |
| `SPI_IOC_WR_BITS_PER_WORD` | Write | u8 | اكتب حجم الـ word |
| `SPI_IOC_RD_MAX_SPEED_HZ` | Read | u32 | اقرأ أقصى سرعة |
| `SPI_IOC_WR_MAX_SPEED_HZ` | Write | u32 | اكتب أقصى سرعة |
| `SPI_IOC_MESSAGE(N)` | Write | variable | نفّذ N transfers دفعة واحدة |

#### SPI_DELAY_UNIT Values

| قيمة | المعنى |
|------|--------|
| `SPI_DELAY_UNIT_USECS` (0) | الـ delay بالـ microseconds |
| `SPI_DELAY_UNIT_NSECS` (1) | الـ delay بالـ nanoseconds |
| `SPI_DELAY_UNIT_SCK` (2) | الـ delay بعدد clock cycles |

#### Transfer Width (tx_nbits / rx_nbits)

| قيمة | المعنى |
|------|--------|
| `SPI_NBITS_SINGLE` (0x01) | 1-bit (standard SPI) |
| `SPI_NBITS_DUAL` (0x02) | 2-bit (Dual SPI) |
| `SPI_NBITS_QUAD` (0x04) | 4-bit (Quad SPI) |
| `SPI_NBITS_OCTAL` (0x08) | 8-bit (Octal SPI) |

#### Config Options المهمة

| Option | Default | المعنى |
|--------|---------|--------|
| `SPIDEV_MAJOR` | 153 | الـ major number المخصص لـ spidev |
| `N_SPI_MINORS` | 32 | أقصى عدد devices (max 256) |
| `bufsiz` (module param) | 4096 | حجم أكبر SPI message مدعوم بالـ bytes |
| `CONFIG_COMPAT` | depends | تفعيل دعم الـ 32-bit userspace على kernel 64-bit |
| `CONFIG_SPI_SLAVE` | depends | دعم وضع الـ SPI target/slave |

---

### الـ Structs المهمة

#### 1. `struct spidev_data` — القلب الأساسي

**الغرض:** بيمثل instance واحد من الـ `/dev/spidevB.C` — بيربط الـ character device بالـ SPI device الحقيقي.

```c
struct spidev_data {
    dev_t           devt;        /* major:minor للـ char device */
    struct mutex    spi_lock;    /* يحمي الـ spi pointer من الـ removal */
    struct spi_device *spi;      /* pointer للـ SPI device الفعلي */
    struct list_head device_entry; /* entry في الـ global device_list */

    struct mutex    buf_lock;    /* يحمي الـ TX/RX buffers */
    unsigned        users;       /* عدد الـ open() المفتوحة حالياً */
    u8             *tx_buffer;   /* bounce buffer للإرسال (NULL لو مش مفتوح) */
    u8             *rx_buffer;   /* bounce buffer للاستقبال (NULL لو مش مفتوح) */
    u32             speed_hz;    /* الـ speed الحالية — cached من spi->max_speed_hz */
};
```

**الحقول بالتفصيل:**

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `devt` | `dev_t` | بيخزن الـ `MKDEV(153, minor)` — بيستخدمه `spidev_open` عشان يطابق الـ inode |
| `spi_lock` | `mutex` | بيحمي الـ `spi` pointer — لو الـ device اتنزع، `spi` بيبقى NULL |
| `spi` | `*spi_device` | الـ link للـ hardware — بيتحول NULL في `spidev_remove` |
| `device_entry` | `list_head` | بيربطه بالـ global `device_list` |
| `buf_lock` | `mutex` | بيحمي الـ buffers وبيمنع concurrent ioctl |
| `users` | `unsigned` | reference count للـ open files — البفرات بتتحذف لما يوصل صفر |
| `tx_buffer` | `u8*` | بيتحجز في `spidev_open` بحجم `bufsiz` (default 4096 bytes) |
| `rx_buffer` | `u8*` | نفس tx_buffer |
| `speed_hz` | `u32` | بيتبدأ من `spi->max_speed_hz` — بيتعدل عبر ioctl `SPI_IOC_WR_MAX_SPEED_HZ` |

---

#### 2. `struct spi_ioc_transfer` — واجهة الـ userspace

**الغرض:** بتوصف transfer واحد في الـ ioctl — بيبعتها الـ userspace لـ `SPI_IOC_MESSAGE(N)`.

```c
struct spi_ioc_transfer {
    __u64   tx_buf;          /* pointer للـ TX data في الـ userspace */
    __u64   rx_buf;          /* pointer للـ RX buffer في الـ userspace */
    __u32   len;             /* حجم الـ transfer بالـ bytes */
    __u32   speed_hz;        /* override للسرعة (0 = استخدم default) */
    __u16   delay_usecs;     /* delay بعد الـ transfer */
    __u8    bits_per_word;   /* override لحجم الـ word */
    __u8    cs_change;       /* toggle chipselect بعد الـ transfer ده */
    __u8    tx_nbits;        /* عدد الـ TX wires */
    __u8    rx_nbits;        /* عدد الـ RX wires */
    __u8    word_delay_usecs;/* delay بين الـ words */
    __u8    pad;             /* padding للـ alignment */
};
```

**ملاحظة مهمة:** الـ `tx_buf` و`rx_buf` بيتخزنوا كـ `__u64` مش pointers — ده عشان يشتغل بشكل موحد على 32-bit و64-bit kernels. في الـ 32-bit compat code، بيتحولوا من `compat_ptr`.

---

#### 3. `struct spi_device` — الـ SPI Device

**الغرض:** بيمثل الـ chip الفيزيائي اللي متوصل على الـ SPI bus — بيتعمل من الـ kernel core.

الحقول المهمة اللي spidev بيستخدمها:

| Field | الاستخدام في spidev |
|-------|---------------------|
| `mode` | بيتقرى/بيتكتب عبر `SPI_IOC_RD/WR_MODE` |
| `max_speed_hz` | مصدر الـ `spidev_data.speed_hz` |
| `bits_per_word` | بيتقرى/بيتكتب عبر `SPI_IOC_RD/WR_BITS_PER_WORD` |
| `controller` | بيستخدمه `spidev_ioctl` للتحقق من `use_gpio_descriptors` |
| `dev` | للـ logging (`dev_dbg`) وللـ `device_create` |

---

#### 4. `struct spi_transfer` — الـ Kernel Transfer

**الغرض:** بيوصف read/write buffer pair واحد — مكونات الـ `spi_message`.

الحقول اللي spidev بيستخدمها في `spidev_message`:

| Field | المصدر |
|-------|--------|
| `tx_buf` | bounce buffer من `spidev->tx_buffer` |
| `rx_buf` | bounce buffer من `spidev->rx_buffer` |
| `len` | من `u_tmp->len` |
| `speed_hz` | من `u_tmp->speed_hz` أو `spidev->speed_hz` |
| `bits_per_word` | من `u_tmp->bits_per_word` |
| `cs_change` | من `u_tmp->cs_change` |
| `delay.value/unit` | من `u_tmp->delay_usecs` بالـ USECS unit |
| `word_delay.value/unit` | من `u_tmp->word_delay_usecs` |
| `tx_nbits` / `rx_nbits` | من `u_tmp->tx_nbits/rx_nbits` |
| `transfer_list` | بيضيفه لـ `spi_message` عبر `spi_message_add_tail` |

---

#### 5. `struct spi_message` — الـ Atomic Transaction

**الغرض:** بيجمع مجموعة `spi_transfer` في transaction واحدة atomic — الـ chipselect بيبقى active طول المدة.

الحقول المهمة:

| Field | الوظيفة |
|-------|---------|
| `transfers` | linked list من `spi_transfer` structs |
| `spi` | بيتحدد من الـ core لما يتبعت |
| `actual_length` | بتقرأه spidev بعد الـ sync كـ return value |
| `status` | 0 = نجاح، negative = error code |
| `complete` | callback — spidev بيستخدم sync فمش بيحتاجه |

---

#### 6. `struct spi_driver` — الـ Driver Registration

```c
static struct spi_driver spidev_spi_driver = {
    .driver = {
        .name           = "spidev",
        .of_match_table = spidev_dt_ids,     /* Device Tree matching */
        .acpi_match_table = spidev_acpi_ids, /* ACPI matching */
    },
    .probe   = spidev_probe,
    .remove  = spidev_remove,
    .id_table = spidev_spi_ids,              /* SPI device ID matching */
};
```

---

#### 7. `struct spi_delay` — الـ Timing Control

```c
struct spi_delay {
    u16  value;  /* قيمة الـ delay */
    u8   unit;   /* USECS=0, NSECS=1, SCK=2 */
};
```

الـ spidev بيملأه من `delay_usecs` و`word_delay_usecs` في الـ `spi_ioc_transfer`.

---

### رسوم العلاقات بين الـ Structs

```
┌─────────────────────────────────────────────────────────────────┐
│                      Global State                               │
│                                                                 │
│  device_list (LIST_HEAD) ──────────────────────────────────┐   │
│  device_list_lock (mutex)                                   │   │
│  minors[N_SPI_MINORS] (bitmap)                              │   │
└─────────────────────────────────────────────────────────────┘   │
                                                              │
        ┌─────────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────┐
│         struct spidev_data               │
│  ┌──────────────────────────────────┐    │
│  │ devt (major=153, minor=N)        │    │
│  │ spi_lock (mutex)                 │    │
│  │ spi ──────────────────────────────────────────────┐
│  │ device_entry (list_head) ◄── في device_list       │
│  │ buf_lock (mutex)                 │    │            │
│  │ users (ref count)                │    │            │
│  │ tx_buffer ──► [kmalloc bufsiz]   │    │            │
│  │ rx_buffer ──► [kmalloc bufsiz]   │    │            │
│  │ speed_hz                         │    │            │
│  └──────────────────────────────────┘    │            │
└──────────────────────────────────────────┘            │
                                                        │
                                                        ▼
                                          ┌─────────────────────────┐
                                          │    struct spi_device     │
                                          │  mode                    │
                                          │  max_speed_hz            │
                                          │  bits_per_word           │
                                          │  controller ─────────────────────┐
                                          │  dev                     │        │
                                          └─────────────────────────┘        │
                                                                              ▼
                                                               ┌──────────────────────┐
                                                               │  struct spi_controller│
                                                               │  bus_num              │
                                                               │  use_gpio_descriptors │
                                                               │  setup()              │
                                                               │  transfer()           │
                                                               └──────────────────────┘

Per ioctl call:
┌────────────────────────────────────────────────────────────────┐
│  struct spi_ioc_transfer[] (from userspace, copied to kernel)  │
│    tx_buf ─► userspace addr                                    │
│    rx_buf ─► userspace addr                                    │
└────────────────────────────────────────────────────────────────┘
           │ spidev_message() translates
           ▼
┌─────────────────────────────────────────────┐
│  struct spi_message (on stack)              │
│    transfers (list_head)                    │
│      │                                      │
│      ├──► struct spi_transfer [0]           │
│      │      tx_buf ─► spidev->tx_buffer     │
│      │      rx_buf ─► spidev->rx_buffer     │
│      │      transfer_list                   │
│      │                                      │
│      └──► struct spi_transfer [1]           │
│             ...                             │
└─────────────────────────────────────────────┘
           │ spi_sync()
           ▼
      SPI Controller Hardware
```

---

### دورة الحياة (Lifecycle Diagrams)

#### Module Lifecycle

```
insmod spidev.ko
     │
     ▼
spidev_init()
     │
     ├─► register_chrdev(153, "spi", &spidev_fops)
     │       └─ يحجز الـ major number 153 مع 256 minor
     │
     ├─► class_register(&spidev_class)
     │       └─ يعمل /sys/class/spidev/ — يخلي udev يشتغل
     │
     └─► spi_register_driver(&spidev_spi_driver)
             └─ يسجل مع الـ SPI bus driver
                  │
                  ▼ (لكل device متوافقة)
             spidev_probe(spi)
                  │
                  ├─► kzalloc(spidev_data)
                  ├─► spidev->spi = spi
                  ├─► mutex_init(spi_lock, buf_lock)
                  ├─► find_first_zero_bit(minors) → minor
                  ├─► MKDEV(153, minor) → spidev->devt
                  ├─► device_create(&spidev_class, ..., "spidev%d.%d")
                  │       └─ udev يعمل /dev/spidevB.C
                  ├─► set_bit(minor, minors)
                  ├─► list_add(&spidev->device_entry, &device_list)
                  └─► spi_set_drvdata(spi, spidev)
```

#### File Open/Close Lifecycle

```
open("/dev/spidev0.0", O_RDWR)
     │
     ▼
spidev_open(inode, filp)
     │
     ├─► mutex_lock(&device_list_lock)
     ├─► list_for_each_entry: ابحث عن devt == inode->i_rdev
     ├─► kmalloc(tx_buffer, bufsiz)  [لو users كان صفر]
     ├─► kmalloc(rx_buffer, bufsiz)  [لو users كان صفر]
     ├─► spidev->users++
     ├─► filp->private_data = spidev
     ├─► stream_open(inode, filp)    [يمنع الـ pread/pwrite/seek]
     └─► mutex_unlock(&device_list_lock)

          ↕ استخدام (read/write/ioctl)

close(fd)
     │
     ▼
spidev_release(inode, filp)
     │
     ├─► mutex_lock(&device_list_lock)
     ├─► spidev = filp->private_data
     ├─► filp->private_data = NULL
     ├─► mutex_lock(&spi_lock) → فحص لو spi == NULL (device اتنزع)
     ├─► spidev->users--
     ├─► [لو users == 0]:
     │     ├─► kfree(tx_buffer)
     │     ├─► kfree(rx_buffer)
     │     └─► [لو spi == NULL]: kfree(spidev)  ← delayed free
     │         [لو spi != NULL]: reset speed_hz
     └─► mutex_unlock(&device_list_lock)
```

#### Device Removal Lifecycle

```
rmmod / device unplug
     │
     ▼
spidev_remove(spi)
     │
     ├─► mutex_lock(&device_list_lock)   ← يمنع open() جديدة
     ├─► mutex_lock(&spidev->spi_lock)
     ├─► spidev->spi = NULL              ← الـ I/O الجارية هتشوف NULL وترجع -ESHUTDOWN
     ├─► mutex_unlock(&spidev->spi_lock)
     ├─► list_del(&spidev->device_entry) ← تشيله من القائمة
     ├─► device_destroy(&spidev_class, spidev->devt) ← udev يمسح /dev/spidevB.C
     ├─► clear_bit(MINOR(spidev->devt), minors)
     ├─► [لو users == 0]: kfree(spidev)  ← immediate free
     │   [لو users > 0]:  تسيبه — spidev_release هيعمل الـ free
     └─► mutex_unlock(&device_list_lock)
```

---

### Call Flow Diagrams

#### read() Flow

```
userspace: read(fd, buf, count)
     │
     ▼
spidev_read(filp, buf, count, f_pos)
     │
     ├─► [count > bufsiz] → return -EMSGSIZE
     ├─► spidev = filp->private_data
     ├─► mutex_lock(&spidev->buf_lock)
     │
     ▼
spidev_sync_read(spidev, count)
     │
     ├─► init spi_transfer: rx_buf=spidev->rx_buffer, len=count, speed_hz
     ├─► spi_message_init(&m)
     ├─► spi_message_add_tail(&t, &m)
     │
     ▼
spidev_sync(spidev, &m)
     │
     ├─► mutex_lock(&spidev->spi_lock)
     ├─► [spidev->spi == NULL] → return -ESHUTDOWN
     │
     ▼
spidev_sync_unlocked(spi, message)
     │
     └─► spi_sync(spi, message)  ← blocking call للـ SPI core
              │
              ▼
         spi_controller->transfer_one_message()
              │
              ▼
         Hardware SPI registers
              │
              ▼ (data comes back)
     status = message->actual_length
     mutex_unlock(&spidev->spi_lock)
     mutex_unlock(&spidev->buf_lock)
     copy_to_user(buf, spidev->rx_buffer, status)
     return status
```

#### ioctl SPI_IOC_MESSAGE Flow (full duplex)

```
userspace: ioctl(fd, SPI_IOC_MESSAGE(N), xfers)
     │
     ▼
spidev_ioctl(filp, cmd, arg)
     │
     ├─► [_IOC_TYPE(cmd) != SPI_IOC_MAGIC] → -ENOTTY
     ├─► mutex_lock(&spidev->spi_lock)
     ├─► spi = spi_dev_get(spidev->spi)  ← refcount++
     ├─► [spi == NULL] → -ESHUTDOWN
     ├─► ctlr = spi->controller
     ├─► mutex_lock(&spidev->buf_lock)
     │
     ▼
spidev_get_ioc_message(cmd, u_ioc, &n_ioc)
     │
     ├─► validate cmd type/number/direction
     ├─► n_ioc = _IOC_SIZE(cmd) / sizeof(spi_ioc_transfer)
     └─► memdup_user(u_ioc, size) → كوبي من userspace لـ kernel
     │
     ▼
spidev_message(spidev, ioc, n_ioc)
     │
     ├─► spi_message_init(&msg)
     ├─► k_xfers = kcalloc(n_xfers, sizeof(spi_transfer))
     │
     ├─► [لكل transfer]:
     │     ├─► ALIGN(len, ARCH_DMA_MINALIGN) → len_aligned
     │     ├─► [u_tmp->rx_buf] → k_tmp->rx_buf = rx_buf; rx_buf += len_aligned
     │     ├─► [u_tmp->tx_buf] → copy_from_user → k_tmp->tx_buf = tx_buf; tx_buf += len_aligned
     │     ├─► نقل باقي الحقول: cs_change, bits_per_word, delay, speed_hz
     │     └─► spi_message_add_tail(k_tmp, &msg)
     │
     ▼
spidev_sync_unlocked(spidev->spi, &msg)  ← مفيش lock هنا — buf_lock كافي
     │
     └─► spi_sync(spi, &msg) → SPI core → hardware
              │
              ▼ (data returns)
     ├─► [لكل transfer له rx_buf]:
     │     └─► copy_to_user(u_tmp->rx_buf, k_tmp->rx_buf, u_tmp->len)
     │
     ├─► kfree(k_xfers)
     ├─► kfree(ioc)
     ├─► mutex_unlock(&buf_lock)
     ├─► spi_dev_put(spi)  ← refcount--
     ├─► mutex_unlock(&spi_lock)
     └─► return total bytes
```

#### ioctl WR_MODE Flow

```
userspace: ioctl(fd, SPI_IOC_WR_MODE32, &mode)
     │
     ▼
spidev_ioctl → case SPI_IOC_WR_MODE32:
     │
     ├─► get_user(tmp, (u32*)arg)         ← قرأ من userspace
     ├─► [tmp & ~SPI_MODE_MASK] → -EINVAL  ← رفض الـ kernel-only bits
     ├─► [ctlr->use_gpio_descriptors && spi_get_csgpiod(spi,0)]:
     │     └─► tmp |= SPI_CS_HIGH         ← force CS_HIGH للـ GPIO descriptors
     ├─► save = spi->mode
     ├─► spi->mode = (tmp | (spi->mode & ~SPI_MODE_MASK)) & SPI_MODE_USER_MASK
     ├─► retval = spi_setup(spi)          ← بيطبق التغيير على الـ hardware
     ├─► [retval < 0]: spi->mode = save   ← rollback لو فشل
     └─► [retval == 0]: dev_dbg(...)
```

---

### استراتيجية الـ Locking

الـ spidev بيستخدم **3 مستويات من الـ locking** — كل واحد بيحمي scope مختلف:

#### الـ Locks الموجودة

| Lock | النوع | المحمي |
|------|-------|--------|
| `device_list_lock` | global mutex | الـ `device_list` كلها + الـ `minors` bitmap |
| `spidev->spi_lock` | per-device mutex | الـ `spidev->spi` pointer |
| `spidev->buf_lock` | per-device mutex | الـ `tx_buffer`، `rx_buffer`، `speed_hz`، وأثناء `spi_setup()` |

#### ترتيب الـ Lock Acquisition (Lock Ordering)

```
device_list_lock
     └──► spidev->spi_lock     (مسموح)
               └──► buf_lock   (مسموح)

أو:

spidev->spi_lock
     └──► buf_lock              (مسموح)
```

**ممنوع منعاً باتاً** عكس الترتيب ده — يسبب deadlock.

#### تفاصيل كل Lock

**`device_list_lock` (global):**
- بيتمسك في `spidev_open` عشان يدور على الـ device في القائمة
- بيتمسك في `spidev_release` عشان يعمل cleanup
- بيتمسك في `spidev_probe` عشان يضيف لـ `device_list` ويحجز minor
- بيتمسك في `spidev_remove` عشان يشيل من القائمة ويمسح الـ device node

**`spidev->spi_lock` (per-instance):**
- بيحمي الـ `spidev->spi` pointer من الـ race مع `spidev_remove`
- في `spidev_sync`: بيتمسك → بيقرأ `spidev->spi` → بيبعت للـ core → بيتفتح
- في `spidev_ioctl`: بيتمسك → `spi_dev_get()` → بيتفتح بعد ما يخلص كل حاجة
- في `spidev_remove`: بيتمسك → `spidev->spi = NULL` → بيتفتح

**`spidev->buf_lock` (per-instance):**
- بيحمي الـ bounce buffers من concurrent reads/writes
- في `spidev_ioctl` بيستخدمه لثلاث أغراض:
  1. يمنع I/O أثناء `spi_setup()` عشان `setup()` آمن
  2. يمنع concurrent `SPI_IOC_WR_*` مع `SPI_IOC_RD_*`
  3. يحمي الـ buffers أثناء `SPI_IOC_MESSAGE`

#### Locking في `spidev_message` — استثناء مهم

في `spidev_message`، الـ call هو `spidev_sync_unlocked` مش `spidev_sync`:

```c
/* spidev_message يمسك buf_lock من بره */
status = spidev_sync_unlocked(spidev->spi, &msg);
/*                   ^^^ مش بياخد spi_lock تاني */
```

ده صح لأن `spidev_ioctl` بالفعل ماسك `spi_lock` وعمل `spi_dev_get()` — فالـ `spidev->spi` مضمون مش هيتغير.

#### Race Condition الأساسي اللي بيتعالج

```
Thread 1 (I/O)          Thread 2 (device removal)
─────────────────────   ──────────────────────────
mutex_lock(spi_lock)
spi = spidev->spi    →  [blocked on spi_lock]
mutex_unlock(spi_lock)
spi_sync(spi, msg)      mutex_lock(spi_lock)
                        spidev->spi = NULL
                        mutex_unlock(spi_lock)
[I/O finishes safely]   device_destroy(...)
```

الـ `spi_dev_get()` في الـ ioctl path بيزود الـ refcount — حتى لو الـ device اتنزعت، الـ `struct spi_device` في الميموري لحد ما `spi_dev_put()` يترجع.

---

### ملاحظات تصميمية مهمة

**Bounce Buffers:** الـ spidev بيستخدم bounce buffers (tx_buffer/rx_buffer) لأن الـ userspace pointers مش DMA-safe. الـ `copy_from_user` بيجيب البيانات لـ kernel memory أولاً، وبعدين الـ SPI core بيقدر يعمل DMA منها.

**الـ DMA Alignment:** في `spidev_message` بيعمل `ALIGN(len, ARCH_DMA_MINALIGN)` على كل transfer عشان الـ next buffer في الـ bounce buffer يبدأ من address صح للـ DMA — حتى لو الـ transfer نفسه أصغر.

**الـ Delayed Free:** لو الـ device اتنزعت (`spi == NULL`) وفيه files مفتوحة (`users > 0`)، الـ `kfree(spidev)` بيتأجل لـ `spidev_release()` — ده يضمن إن الـ struct مش بيتمسح وفيه code لسه شايل pointer عليه.
## Phase 4: شرح الـ Functions

---

### ملخص عام — Cheatsheet

#### Category 1: Module Init / Cleanup

| Function | Signature (brief) | Purpose |
|---|---|---|
| `spidev_init` | `static int __init spidev_init(void)` | تسجيل chrdev + class + spi_driver |
| `spidev_exit` | `static void __exit spidev_exit(void)` | إلغاء التسجيل بالعكس |

#### Category 2: Driver Probe / Remove

| Function | Signature (brief) | Purpose |
|---|---|---|
| `spidev_probe` | `static int spidev_probe(struct spi_device *spi)` | allocate spidev_data، إنشاء device node |
| `spidev_remove` | `static void spidev_remove(struct spi_device *spi)` | نزع الـ device node وتحرير الـ struct |

#### Category 3: File Operations

| Function | Signature (brief) | Purpose |
|---|---|---|
| `spidev_open` | `static int spidev_open(struct inode *, struct file *)` | allocate TX/RX buffers، ربط spidev بالـ filp |
| `spidev_release` | `static int spidev_release(struct inode *, struct file *)` | free buffers، cleanup عند آخر close |
| `spidev_read` | `static ssize_t spidev_read(struct file *, char __user *, size_t, loff_t *)` | قراءة half-duplex من الـ SPI bus |
| `spidev_write` | `static ssize_t spidev_write(struct file *, const char __user *, size_t, loff_t *)` | كتابة half-duplex على الـ SPI bus |
| `spidev_ioctl` | `static long spidev_ioctl(struct file *, unsigned int, unsigned long)` | التحكم في الـ SPI mode/speed + full-duplex transfers |
| `spidev_compat_ioctl` | `static long spidev_compat_ioctl(struct file *, unsigned int, unsigned long)` | نفس spidev_ioctl لكن لـ 32-bit userspace على 64-bit kernel |

#### Category 4: Internal Transfer Helpers

| Function | Signature (brief) | Purpose |
|---|---|---|
| `spidev_sync` | `static ssize_t spidev_sync(struct spidev_data *, struct spi_message *)` | تنفيذ spi_sync مع حماية spi_lock |
| `spidev_sync_unlocked` | `static ssize_t spidev_sync_unlocked(struct spi_device *, struct spi_message *)` | تنفيذ spi_sync بدون lock (caller يمسك الـ lock) |
| `spidev_sync_write` | `static inline ssize_t spidev_sync_write(struct spidev_data *, size_t)` | بناء spi_message للـ TX-only وتنفيذه |
| `spidev_sync_read` | `static inline ssize_t spidev_sync_read(struct spidev_data *, size_t)` | بناء spi_message للـ RX-only وتنفيذه |

#### Category 5: IOCTL Helpers

| Function | Signature (brief) | Purpose |
|---|---|---|
| `spidev_get_ioc_message` | `static struct spi_ioc_transfer *spidev_get_ioc_message(unsigned int, struct spi_ioc_transfer __user *, unsigned *)` | validate + copy_from_user لمصفوفة الـ transfers |
| `spidev_message` | `static int spidev_message(struct spidev_data *, struct spi_ioc_transfer *, unsigned)` | تحويل user transfers → kernel spi_message وتنفيذها |

#### Category 6: Device ID / Validation Callbacks

| Function | Signature (brief) | Purpose |
|---|---|---|
| `spidev_of_check` | `static int spidev_of_check(struct device *)` | ترفض تسجيل generic "spidev" compatible string في DT |
| `spidev_acpi_check` | `static int spidev_acpi_check(struct device *)` | تحذر من استخدام ACPI spidev في production |

---

### Group 1: Module Initialization & Cleanup

الـ group ده مسؤول عن تسجيل الـ driver مع الـ kernel وإلغاء التسجيل. الترتيب فيه مهم: chrdev الأول، بعدين الـ class عشان udev يعرف يعمل nodes، وأخيراً الـ spi_driver نفسه.

---

#### `spidev_init`

```c
static int __init spidev_init(void)
```

بتسجّل الـ character device بـ major number ثابت (153)، بعدين بتسجّل الـ `spidev_class` عشان udev/mdev يعملوا الـ `/dev/spidevB.C` nodes تلقائياً، وفي الآخر بتسجّل الـ `spi_driver`. لو أي خطوة فشلت، بترجع للخلف وتلغي اللي عملته قبلها.

**Parameters:** لا يوجد

**Return:** `0` عند النجاح، أو error code سالب

**Key details:**
- الترتيب: `register_chrdev` → `class_register` → `spi_register_driver`
- لو `spi_register_driver` فشل، بيعمل `class_unregister` + `unregister_chrdev`
- الـ major number 153 assigned رسمياً في `Documentation/admin-guide/devices.txt`
- الـ minor numbers بتُخصَّص dynamically من bitmap `minors[N_SPI_MINORS]`

**Caller context:** `module_init` — بيتشغل عند `insmod` أو boot

```
spidev_init():
    register_chrdev(153, "spi", &spidev_fops)  → fail? return error
    class_register(&spidev_class)               → fail? unregister_chrdev; return error
    spi_register_driver(&spidev_spi_driver)     → fail? class_unregister + unregister_chrdev
    return status
```

---

#### `spidev_exit`

```c
static void __exit spidev_exit(void)
```

بتلغي تسجيل الـ spi_driver أولاً (يضمن استدعاء `spidev_remove` لكل device)، بعدين تلغي الـ class، وأخيراً تلغي الـ chrdev. الترتيب عكس الـ `spidev_init` تماماً.

**Parameters:** لا يوجد

**Return:** `void`

**Caller context:** `module_exit` — عند `rmmod`

---

### Group 2: Driver Probe & Remove

الـ probe بيتشغل لما الـ SPI subsystem يلاقي device بيطابق أي entry في `spidev_spi_ids` أو `spidev_dt_ids` أو `spidev_acpi_ids`. بيعمل الـ `spidev_data` struct ويربطه بالـ `spi_device`.

---

#### `spidev_probe`

```c
static int spidev_probe(struct spi_device *spi)
```

أول حاجة بيعملها هي استدعاء الـ match callback لو موجود (زي `spidev_of_check` أو `spidev_acpi_check`) عشان يتحقق من صحة الـ device. بعدين بيعمل `kzalloc` للـ `spidev_data`، بيحجز minor number من الـ bitmap، وبيعمل `device_create` اللي يخلّي udev ينشئ `/dev/spidevB.C`.

**Parameters:**
- `spi`: pointer للـ `spi_device` اللي الـ SPI core عامله bind

**Return:** `0` عند النجاح، أو error code

**Key details:**
- بيحجز الـ `device_list_lock` mutex أثناء إضافة الـ device للـ `device_list`
- الاسم في `/dev` بيكون `spidev{bus_num}.{chip_select_index}` مثلاً `spidev0.0`
- `spi_set_drvdata(spi, spidev)` بيخزن الـ pointer عشان `spidev_remove` يقدر يسترجعه
- الـ TX/RX buffers مش بتُخصَّص هنا — بيتأخر لـ `spidev_open`
- `spidev->speed_hz` بتتهيأ من `spi->max_speed_hz`

```
spidev_probe(spi):
    match = device_get_match_data(&spi->dev)
    if match: call match() → fail? return error

    spidev = kzalloc(sizeof(*spidev), GFP_KERNEL)
    init spidev: spi ptr, mutexes, list_head

    mutex_lock(device_list_lock)
    minor = find_first_zero_bit(minors, N_SPI_MINORS)
    if minor available:
        spidev->devt = MKDEV(153, minor)
        device_create(&spidev_class, ...)  → creates /dev/spidevB.C
        set_bit(minor, minors)
        list_add(&spidev->device_entry, &device_list)
    mutex_unlock(device_list_lock)

    spi_set_drvdata(spi, spidev)
    return status
```

---

#### `spidev_remove`

```c
static void spidev_remove(struct spi_device *spi)
```

بتُبطل الـ device بأمان: أول حاجة بتعمل null للـ `spidev->spi` تحت الـ `spi_lock` عشان أي I/O جاري هيلاقي الـ spi == NULL وهيرجع `-ESHUTDOWN`. بعدين بتشيل الـ device من الـ list، بتعمل `device_destroy`، وبتفرر الـ `spidev_data` لو مفيش users فاتحين الـ fd.

**Parameters:**
- `spi`: الـ `spi_device` اللي بيتشال

**Return:** `void`

**Key details:**
- لو `spidev->users > 0` (يعني في fd مفتوح)، الـ `spidev_data` مش بتتحرر هنا — بتتحرر في `spidev_release` بعدين
- الـ `spi_lock` mutex بيحمي الـ `spidev->spi` field ضد race مع I/O
- `clear_bit(MINOR(spidev->devt), minors)` بيرجع الـ minor للـ pool

```
spidev_remove(spi):
    spidev = spi_get_drvdata(spi)

    mutex_lock(device_list_lock)
        mutex_lock(spidev->spi_lock)
            spidev->spi = NULL          // signals ongoing I/O to abort
        mutex_unlock(spidev->spi_lock)

        list_del(&spidev->device_entry)
        device_destroy(&spidev_class, spidev->devt)
        clear_bit(MINOR(spidev->devt), minors)

        if spidev->users == 0:
            kfree(spidev)               // no open fds, free now
        // else: spidev_release() will free later
    mutex_unlock(device_list_lock)
```

---

### Group 3: File Operations

ده الـ group اللي بيواجه الـ userspace مباشرةً عبر الـ VFS. كل function موصولة بالـ `spidev_fops`.

---

#### `spidev_open`

```c
static int spidev_open(struct inode *inode, struct file *filp)
```

بتدور على الـ `spidev_data` المناسب في الـ `device_list` عن طريق مقارنة `devt` بالـ `inode->i_rdev`. لو لقيته، بتعمل `kmalloc` للـ TX/RX buffers (بحجم `bufsiz` = 4096 byte افتراضياً)، وبتزود الـ `users` counter، وبتحط الـ pointer في `filp->private_data`.

**Parameters:**
- `inode`: inode الـ device file
- `filp`: الـ file struct للـ open الجاري

**Return:** `0` عند النجاح، `-ENXIO` لو مفيش device، `-ENOMEM` لو فيه مشكلة في الـ allocation

**Key details:**
- الـ buffers بتتعمل lazy: بس لو `tx_buffer == NULL` (يعني أول open). لو أكتر من user فاتح نفس الـ device، بيشاركوا نفس الـ buffers وبيستخدموا `buf_lock` للتنسيق
- `stream_open(inode, filp)` بيتعمل عشان يلغي الـ `O_NONBLOCK` seeking semantics ويخليها stream
- الـ `device_list_lock` mutex بيُمسك طول العملية كلها عشان يحمي ضد `spidev_remove` concurrent

```
spidev_open(inode, filp):
    mutex_lock(device_list_lock)

    list_for_each_entry(iter, &device_list):
        if iter->devt == inode->i_rdev:
            spidev = iter; break

    if !spidev: goto err_find_dev  // -ENXIO

    if !spidev->tx_buffer:
        tx_buffer = kmalloc(bufsiz)  → fail? goto err_find_dev
    if !spidev->rx_buffer:
        rx_buffer = kmalloc(bufsiz)  → fail? free tx_buffer; goto err_find_dev

    spidev->users++
    filp->private_data = spidev
    stream_open(inode, filp)

    mutex_unlock(device_list_lock)
    return 0
```

---

#### `spidev_release`

```c
static int spidev_release(struct inode *inode, struct file *filp)
```

بتنزّل الـ `users` counter. لو وصل صفر (آخر close)، بتحرر الـ TX/RX buffers. لو الـ `spi_device` اتنزع من قبل (بعد `spidev_remove`)، بتحرر الـ `spidev_data` نفسه. في SPI slave mode، بتستدعي `spi_target_abort` عشان تكنسل أي pending transfer.

**Parameters:**
- `inode`, `filp`: standard VFS parameters

**Return:** `0` دايماً

**Key details:**
- الـ `dofree` flag بيتحدد بسرعة تحت الـ `spi_lock` عشان يتجنب race مع `spidev_remove`
- إعادة ضبط `speed_hz` من `spi->max_speed_hz` يضمن إن الـ state بيرجع للـ default بعد آخر close
- `filp->private_data = NULL` أول حاجة بتتعمل عشان تمنع double-free

```
spidev_release(inode, filp):
    mutex_lock(device_list_lock)
    spidev = filp->private_data
    filp->private_data = NULL

    mutex_lock(spidev->spi_lock)
    dofree = (spidev->spi == NULL)   // was device removed?
    mutex_unlock(spidev->spi_lock)

    spidev->users--
    if users == 0:
        kfree(tx_buffer); kfree(rx_buffer)
        if dofree: kfree(spidev)
        else: spidev->speed_hz = spi->max_speed_hz  // reset speed

    #ifdef CONFIG_SPI_SLAVE
    if !dofree: spi_target_abort(spidev->spi)
    #endif

    mutex_unlock(device_list_lock)
    return 0
```

---

#### `spidev_read`

```c
static ssize_t
spidev_read(struct file *filp, char __user *buf, size_t count, loff_t *f_pos)
```

بتنفذ half-duplex read: بتاخد بيانات من الـ SPI bus في `rx_buffer` الـ kernel-side، بعدين بتعمل `copy_to_user` للـ userspace buffer. الـ chipselect بيتكنترول تلقائياً من الـ SPI core طول الـ transfer.

**Parameters:**
- `filp`: الـ file handle اللي فيه الـ `spidev_data`
- `buf`: الـ userspace buffer اللي هيستلم البيانات
- `count`: عدد الـ bytes المطلوب قراءتها
- `f_pos`: لا يُستخدم فعلياً (stream device)

**Return:** عدد الـ bytes اللي اتقرأت فعلاً، أو error code سالب

**Key details:**
- لو `count > bufsiz`: فوراً بيرجع `-EMSGSIZE`
- الـ `buf_lock` بيتمسك طول العملية (I/O + copy_to_user)
- لو `copy_to_user` فشلت جزئياً، بيرجع الـ bytes اللي اتنقلت فعلاً مش error كامل (إلا لو صفر bytes اتنقلت)
- بيستخدم `spidev->speed_hz` المضبوطة في `spi_transfer.speed_hz`

---

#### `spidev_write`

```c
static ssize_t
spidev_write(struct file *filp, const char __user *buf,
             size_t count, loff_t *f_pos)
```

بتنفذ half-duplex write: بتعمل `copy_from_user` في `tx_buffer` الـ kernel-side، بعدين بتبعت على الـ SPI bus. مفيش RX في العملية دي.

**Parameters:**
- `filp`: الـ file handle
- `buf`: الـ userspace buffer اللي فيه البيانات اللي هتتبعت
- `count`: عدد الـ bytes
- `f_pos`: مش بيستخدم

**Return:** عدد الـ bytes اللي اتبعتت، أو error code

**Key details:**
- لو `copy_from_user` فشلت (أي byte مش accessible): بيرجع `-EFAULT` فوراً
- الـ `buf_lock` بيحمي الـ `tx_buffer` من concurrent writes
- الـ actual bytes transferred بيتحدد من `message->actual_length`

---

#### `spidev_ioctl`

```c
static long
spidev_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
```

ده أهم function في الـ driver. بيتعامل مع كل الـ control commands والـ full-duplex transfers. بيمسك `spi_lock` أولاً ويعمل `spi_dev_get` عشان يحمي ضد device removal، بعدين بيمسك `buf_lock` لثلاث أغراض في نفس الوقت.

**Parameters:**
- `filp`: file handle
- `cmd`: الـ ioctl command code (مشتق من `_IOR/_IOW` macros)
- `arg`: الـ user argument (قيمة أو pointer)

**Return:** `0` أو قيمة موجبة عند النجاح، error code سالب عند الفشل

**Key details:**

الـ `buf_lock` بيُمسك لثلاث أغراض:
1. يمنع I/O جاري خلال `spi_setup()` call
2. يمنع concurrent `SPI_IOC_WR_*` من تغيير الـ mode خلال `SPI_IOC_RD_*`
3. الـ `SPI_IOC_MESSAGE` يحتاج الـ buffer lock

**الـ IOCTL commands المدعومة:**

| Command | Direction | Type | Action |
|---|---|---|---|
| `SPI_IOC_RD_MODE` | Read | `u8` | قراءة الـ SPI mode (0-3 + flags) |
| `SPI_IOC_RD_MODE32` | Read | `u32` | نفس السابق بـ 32 bit |
| `SPI_IOC_WR_MODE` | Write | `u8` | ضبط الـ SPI mode + استدعاء `spi_setup` |
| `SPI_IOC_WR_MODE32` | Write | `u32` | نفس السابق بـ 32 bit |
| `SPI_IOC_RD_LSB_FIRST` | Read | `u8` | هل الـ bit order LSB first? |
| `SPI_IOC_WR_LSB_FIRST` | Write | `u8` | ضبط الـ bit order |
| `SPI_IOC_RD_BITS_PER_WORD` | Read | `u8` | قراءة word size |
| `SPI_IOC_WR_BITS_PER_WORD` | Write | `u8` | ضبط word size |
| `SPI_IOC_RD_MAX_SPEED_HZ` | Read | `u32` | قراءة الـ speed |
| `SPI_IOC_WR_MAX_SPEED_HZ` | Write | `u32` | ضبط الـ speed |
| `SPI_IOC_MESSAGE(N)` | Write | array | full-duplex segmented transfer |

**نقطة دقيقة في `SPI_IOC_WR_MAX_SPEED_HZ`:** بتضبط `spi->max_speed_hz` مؤقتاً، بتعمل `spi_setup`، وبعدين بترجعه للقيمة القديمة. الـ `spidev->speed_hz` (مش `spi->max_speed_hz`) هي اللي بتتحفظ وتُستخدم في الـ transfers الجاية.

**نقطة مهمة في `SPI_IOC_WR_MODE`:** لو الـ controller بيستخدم GPIO descriptors للـ chip select، الـ `SPI_CS_HIGH` flag بتتضاف تلقائياً لأن الـ GPIO framework بيعكس الـ polarity بنفسه. وعند القراءة (`SPI_IOC_RD_MODE`)، الـ flag دي بتتشال من القيمة المرجعة عشان ما تربكش الـ userspace.

```
spidev_ioctl(filp, cmd, arg):
    if IOC_TYPE != SPI_IOC_MAGIC: return -ENOTTY

    mutex_lock(spi_lock)
    spi = spi_dev_get(spidev->spi)   // increments refcount
    if spi == NULL: return -ESHUTDOWN

    mutex_lock(buf_lock)

    switch(cmd):
        RD_MODE/RD_MODE32:
            tmp = spi->mode & SPI_MODE_MASK
            if gpio_cs: tmp &= ~SPI_CS_HIGH
            put_user(tmp, arg)

        WR_MODE/WR_MODE32:
            get_user(tmp, arg)
            save = spi->mode
            spi->mode = tmp | (spi->mode & ~SPI_MODE_MASK)
            retval = spi_setup(spi)
            if fail: spi->mode = save  // rollback

        WR_MAX_SPEED_HZ:
            get_user(tmp, arg)
            save = spi->max_speed_hz
            spi->max_speed_hz = tmp
            spi_setup(spi)
            spidev->speed_hz = tmp  // persist in spidev, not spi
            spi->max_speed_hz = save  // restore spi device

        default (SPI_IOC_MESSAGE):
            ioc = spidev_get_ioc_message(cmd, arg, &n_ioc)
            spidev_message(spidev, ioc, n_ioc)
            kfree(ioc)

    mutex_unlock(buf_lock)
    spi_dev_put(spi)
    mutex_unlock(spi_lock)
```

---

#### `spidev_compat_ioctl`

```c
static long
spidev_compat_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
```

الـ compat ioctl handler لـ 32-bit userspace على 64-bit kernel. لو الـ command هو `SPI_IOC_MESSAGE`، بيحول الـ `arg` بـ `compat_ptr(arg)` ويستدعي `spidev_compat_ioc_message`. لغيره من الـ commands، بيحول الـ `arg` وبيستدعي `spidev_ioctl` مباشرةً.

**Parameters:** نفس `spidev_ioctl`

**Return:** نفس `spidev_ioctl`

**Key details:** المشكلة الرئيسية هنا إن الـ `tx_buf` و `rx_buf` في `spi_ioc_transfer` هما `__u64` في الـ struct، لكن في 32-bit userspace القيم المخزنة فيهم pointers بـ 32-bit. `compat_ptr()` بيحولهم لـ 64-bit kernel pointers صح.

---

### Group 4: Internal Transfer Helpers

---

#### `spidev_sync_unlocked`

```c
static ssize_t
spidev_sync_unlocked(struct spi_device *spi, struct spi_message *message)
```

بتستدعي `spi_sync` مباشرةً وبترجع `message->actual_length` عند النجاح بدل الـ 0.

**Parameters:**
- `spi`: الـ SPI device
- `message`: الـ message المُجهّز

**Return:** عدد الـ bytes المنقولة فعلاً، أو error code سالب

**Key details:**
- **Caller يجب يكون ماسكاً `spi_lock`** — الاسم "unlocked" يقصد إن الـ function نفسها مش بتمسك lock
- `spi_sync` blocking call — بتنام لحين انتهاء الـ transfer
- `message->actual_length` بتتملى من الـ SPI core بعد اكتمال الـ transfer

---

#### `spidev_sync`

```c
static ssize_t
spidev_sync(struct spidev_data *spidev, struct spi_message *message)
```

الـ wrapper الآمن حول `spidev_sync_unlocked`. بتمسك الـ `spi_lock`، بتتحقق إن `spidev->spi != NULL` (الـ device لسه موجود)، بعدين بتستدعي `spidev_sync_unlocked`.

**Parameters:**
- `spidev`: الـ driver private data
- `message`: الـ spi_message المُجهّز

**Return:** bytes transferred أو error code

**Key details:**
- لو `spidev->spi == NULL` (الـ device اتنزع): بترجع `-ESHUTDOWN`
- الـ `spi_lock` mutex بيحمي الـ `spidev->spi` pointer من التغيير أثناء الاستخدام

---

#### `spidev_sync_write`

```c
static inline ssize_t
spidev_sync_write(struct spidev_data *spidev, size_t len)
```

بتبني `spi_transfer` بـ `tx_buf` فقط (بدون `rx_buf`) وبتنفذها. ده الـ half-duplex TX path.

**Parameters:**
- `spidev`: الـ driver data (منها بياخد `tx_buffer` و `speed_hz`)
- `len`: عدد الـ bytes في الـ TX buffer

**Return:** bytes transferred أو error

**Key details:**
- الـ `spi_transfer.speed_hz` بتتحدد من `spidev->speed_hz` (مش `spi->max_speed_hz`)
- `spi_message_init` + `spi_message_add_tail` بيبنوا الـ message على الـ stack

---

#### `spidev_sync_read`

```c
static inline ssize_t
spidev_sync_read(struct spidev_data *spidev, size_t len)
```

نفس `spidev_sync_write` لكن بـ `rx_buf` فقط (بدون `tx_buf`). الـ SPI controller بيبعت zeros على MOSI وبيستقبل على MISO.

**Parameters:** نفس `spidev_sync_write`

**Return:** bytes transferred أو error

---

### Group 5: IOCTL Message Helpers

---

#### `spidev_get_ioc_message`

```c
static struct spi_ioc_transfer *
spidev_get_ioc_message(unsigned int cmd, struct spi_ioc_transfer __user *u_ioc,
                       unsigned *n_ioc)
```

بتـ validate الـ ioctl command وبتعمل `memdup_user` لمصفوفة الـ `spi_ioc_transfer` structures من الـ userspace.

**Parameters:**
- `cmd`: الـ ioctl command اللي بيتحقق منه
- `u_ioc`: الـ userspace pointer لمصفوفة الـ transfers
- `n_ioc`: output — عدد الـ transfers

**Return:** pointer لـ kernel copy من الـ transfers، أو `ERR_PTR(-ENOTTY)` / `ERR_PTR(-EINVAL)`، أو `NULL` لو `n_ioc == 0`

**Key details:**
- `_IOC_SIZE(cmd)` بيحتوي على الـ total size بالـ bytes، مش عدد الـ transfers
- التحقق من قسمة `tmp % sizeof(struct spi_ioc_transfer) == 0` ضروري عشان يتجنب misaligned access
- `memdup_user` بتعمل `kmalloc` + `copy_from_user` في خطوة واحدة — caller مسؤول عن `kfree`
- لو الـ command type غلط: `-ENOTTY` (not a typewriter — الـ standard ioctl error)
- لو الـ size مش مضروب صحيح: `-EINVAL`

---

#### `spidev_message`

```c
static int spidev_message(struct spidev_data *spidev,
                          struct spi_ioc_transfer *u_xfers, unsigned n_xfers)
```

ده الـ function الأكتر تعقيداً في الـ driver. بيحوّل مصفوفة من الـ `spi_ioc_transfer` (user-level structs) لـ `spi_transfer` (kernel-level structs) في `spi_message` واحد، وبيعمل bounce buffering للـ TX data، وبعد التنفيذ بيعمل `copy_to_user` للـ RX data.

**Parameters:**
- `spidev`: الـ driver data (بياخد منها الـ bounce buffers والـ speed)
- `u_xfers`: مصفوفة الـ transfers اللي اتعملت copy_from_user من قبل
- `n_xfers`: عدد الـ transfers

**Return:** total bytes transferred عند النجاح، أو error code سالب

**Key details:**

**الـ Bounce Buffer Layout:**
```
spidev->tx_buffer [bufsiz bytes]:
    [transfer_0_tx | aligned_pad | transfer_1_tx | aligned_pad | ...]

spidev->rx_buffer [bufsiz bytes]:
    [transfer_0_rx | aligned_pad | transfer_1_rx | aligned_pad | ...]
```

- الـ `ALIGN(u_tmp->len, ARCH_DMA_MINALIGN)` ضروري عشان كل transfer تبدأ على حد DMA-aligned في الـ buffer
- الـ overflow checks (total > INT_MAX) موجودة عشان الـ return value مش يتبهدل بـ sign
- لو أي `tx_total` أو `rx_total` عدى `bufsiz`: فوراً `-EMSGSIZE`
- `spidev_sync_unlocked` بتستدعي (مش `spidev_sync`) لأن caller (`spidev_ioctl`) ماسك الـ `spi_lock` بالفعل
- بعد النجاح، loop تانية تعمل `copy_to_user` لكل transfer ليها `rx_buf`

```
spidev_message(spidev, u_xfers, n_xfers):
    k_xfers = kcalloc(n_xfers, sizeof(spi_transfer), GFP_KERNEL)
    tx_buf = spidev->tx_buffer
    rx_buf = spidev->rx_buffer
    total = tx_total = rx_total = 0

    for each u_tmp in u_xfers:
        len_aligned = ALIGN(u_tmp->len, ARCH_DMA_MINALIGN)

        if u_tmp->rx_buf:
            rx_total += len_aligned
            if rx_total > bufsiz: goto done (-EMSGSIZE)
            k_tmp->rx_buf = rx_buf; rx_buf += len_aligned

        if u_tmp->tx_buf:
            tx_total += len_aligned
            if tx_total > bufsiz: goto done (-EMSGSIZE)
            copy_from_user(tx_buf, u_tmp->tx_buf, u_tmp->len)
            k_tmp->tx_buf = tx_buf; tx_buf += len_aligned

        copy u_tmp fields → k_tmp (speed, bits_per_word, cs_change, delays...)
        spi_message_add_tail(k_tmp, &msg)

    status = spidev_sync_unlocked(spidev->spi, &msg)

    for each (u_tmp, k_tmp) in xfers:
        if u_tmp->rx_buf:
            copy_to_user(u_tmp->rx_buf, k_tmp->rx_buf, u_tmp->len)

    status = total
done:
    kfree(k_xfers)
    return status
```

---

#### `spidev_compat_ioc_message`

```c
static long
spidev_compat_ioc_message(struct file *filp, unsigned int cmd,
                          unsigned long arg)
```

بتتعامل خصيصاً مع `SPI_IOC_MESSAGE` من 32-bit userspace. بعد `spidev_get_ioc_message`، بتحوّل الـ `rx_buf` و `tx_buf` في كل transfer بـ `compat_ptr()` قبل تمريرها لـ `spidev_message`.

**Parameters:** نفس `spidev_ioctl` (بس `arg` هو compat pointer)

**Return:** total bytes أو error

**Key details:**
- الـ struct `spi_ioc_transfer` layout متطابق في 32-bit و 64-bit userspace (الـ `__u64` fields هي fixed size)، لكن القيم المخزنة فيها كـ pointers بـ 32-bit تحتاج تحويل
- `compat_ptr(ioc[n].rx_buf)` بيحول الـ 32-bit address لـ 64-bit kernel virtual address

---

### Group 6: Device ID / Validation Callbacks

---

#### `spidev_of_check`

```c
static int spidev_of_check(struct device *dev)
```

بتتحقق إن الـ DT node مش بيستخدم `compatible = "spidev"` العامة. الـ spidev driver لازم يتربط بـ specific compatible strings زي `"rohm,dh2228fv"` مش بـ generic "spidev".

**Parameters:**
- `dev`: الـ device المراد التحقق منه

**Return:** `0` لو كل حاجة تمام (الـ compatible مش "spidev")، `-EINVAL` لو الـ DT غلط

**Key details:**
- `device_property_match_string` بترجع index موجب لو لقت "spidev" في الـ compatible list
- الهدف: منع المطورين من كتابة `compatible = "spidev"` في الـ DT مباشرةً — ده antipattern لأن الـ DT لازم يوصف الـ hardware مش الـ software driver

---

#### `spidev_acpi_check`

```c
static int spidev_acpi_check(struct device *dev)
```

بتطبع warning إن الـ spidev ACPI devices مش للاستخدام في production. دايماً بترجع 0 (بتسمح بالـ probe).

**Parameters:**
- `dev`: الـ ACPI device

**Return:** دايماً `0`

**Key details:**
- الـ ACPI IDs المدعومة (`SPT0001`, `SPT0002`, `SPT0003`) مخصصة للـ development والـ testing فقط
- الفرق عن `spidev_of_check`: دي بس تحذر ومش بتمنع، لأن الـ ACPI use case للـ testing مقبول أكتر من الـ DT

---

### الـ Locking Strategy — نظرة شاملة

```
device_list_lock (mutex):
    يحمي: device_list, minors bitmap
    يُمسك في: open, release, probe, remove

spidev->spi_lock (mutex):
    يحمي: spidev->spi pointer نفسه
    يُمسك في: spidev_sync, spidev_ioctl, spidev_release, spidev_remove

spidev->buf_lock (mutex):
    يحمي: tx_buffer, rx_buffer, وأي I/O
    يُمسك في: spidev_read, spidev_write, spidev_ioctl
```

**الـ Lock Ordering:** `device_list_lock` → `spi_lock` → `buf_lock`

الـ `spidev_ioctl` بيمسك `spi_lock` و `buf_lock` في نفس الوقت، لكن `spidev_remove` بيمسك `device_list_lock` و `spi_lock`. عشان كده الـ `spi_lock` في `spidev_remove` بيُمسك ويُفَك بسرعة (فقط لضبط `spidev->spi = NULL`) قبل أي عمل تاني.

### Data Flow — الـ Full-Duplex IOCTL Path

```
userspace app
    │  ioctl(fd, SPI_IOC_MESSAGE(2), mesg_array)
    ▼
spidev_ioctl()
    │  validates cmd type
    │  spi_dev_get()           ← increments spi_device refcount
    │  mutex_lock(spi_lock)
    │  mutex_lock(buf_lock)
    ▼
spidev_get_ioc_message()
    │  validates IOC_TYPE, IOC_NR, IOC_DIR
    │  memdup_user()           ← copies spi_ioc_transfer[] to kernel
    ▼
spidev_message()
    │  allocates k_xfers[] (kernel spi_transfer array)
    │  for each transfer:
    │    copy_from_user(tx_buffer, u_xfer->tx_buf)
    │    assign rx_buffer pointer
    │    build spi_transfer from spi_ioc_transfer
    │  spidev_sync_unlocked()
    │    └─ spi_sync()         ← submits to SPI controller, sleeps
    │  for each rx transfer:
    │    copy_to_user(u_xfer->rx_buf, rx_buffer)
    │  kfree(k_xfers)
    ▼
spidev_ioctl()
    │  mutex_unlock(buf_lock)
    │  spi_dev_put()           ← decrements spi_device refcount
    │  mutex_unlock(spi_lock)
    ▼
userspace app  ← returns total bytes transferred
```
## Phase 5: دليل الـ Debugging الشامل

### Software Level

#### 1. debugfs entries

**الـ** `spidev` driver مش بيسجّل entries في debugfs بشكل مباشر، لكن الـ SPI core بيعمل كده. الـ entries الأساسية:

```bash
# اعرض كل الـ SPI controllers المسجّلة
ls /sys/kernel/debug/spi*/

# لو الـ kernel اتبنى بـ CONFIG_SPI_DEBUG
cat /sys/kernel/debug/spi0/statistics
```

الـ SPI core بيوفّر statistics تحت `/sys/kernel/debug/spi-*/`:
- `transfers` — عدد الـ transfers اللي اتنفّذت
- `errors` — عدد الأخطاء
- `timedout` — الـ transfers اللي timeout فيها

---

#### 2. sysfs entries

```bash
# الـ device الأساسي (مثلاً spi0.0)
ls /sys/bus/spi/devices/spi0.0/

# قراءة الـ modalias عشان تتأكد من الـ match
cat /sys/bus/spi/devices/spi0.0/modalias
# output: spi:spidev   أو   spi:ltc2488

# الـ driver اللي bind على الـ device
cat /sys/bus/spi/devices/spi0.0/driver/module/srcversion

# الـ chip select number
cat /sys/bus/spi/devices/spi0.0/chip_select

# الـ max speed المضبوطة
cat /sys/bus/spi/devices/spi0.0/max_speed_hz

# الـ bits per word
cat /sys/bus/spi/devices/spi0.0/bits_per_word

# الـ mode (CPOL/CPHA)
cat /sys/bus/spi/devices/spi0.0/mode

# الـ character device المرتبط
ls -la /sys/class/spidev/
# مثال output: spidev0.0 -> ../../devices/platform/soc/...
```

---

#### 3. ftrace — Tracepoints/Events

**الـ** `trace/events/spi.h` بيعرّف الـ tracepoints دي:

| Tracepoint | الوظيفة |
|---|---|
| `spi:spi_controller_idle` | الـ controller خلص شغل |
| `spi:spi_controller_busy` | الـ controller بدأ شغل |
| `spi:spi_setup` | تغيير إعدادات الـ device |
| `spi:spi_set_cs` | تغيير حالة الـ chip select |
| `spi:spi_message_submit` | message اتبعتت للـ queue |
| `spi:spi_message_start` | الـ transfer بدأ فعلياً |
| `spi:spi_message_done` | الـ transfer خلص |
| `spi:spi_transfer_start` | بدأ transfer واحد |
| `spi:spi_transfer_stop` | خلص transfer واحد |

```bash
# فعّل كل events الـ SPI
echo 1 > /sys/kernel/debug/tracing/events/spi/enable

# أو event معين
echo 1 > /sys/kernel/debug/tracing/events/spi/spi_message_done/enable

# شوف الـ output
cat /sys/kernel/debug/tracing/trace

# فلتر على bus معين (spi0)
echo 'bus_num == 0' > /sys/kernel/debug/tracing/events/spi/spi_message_done/filter

# تتبع ioctl calls على /dev/spidev0.0
trace-cmd record -e spi:spi_message_submit -e spi:spi_message_done sleep 5
trace-cmd report
```

**مثال output لـ** `spi_message_done`:
```
spi-1234   [000]  1234.5: spi_message_done: spi0.0 0xffff888... len=8/8
```
يعني: الـ transfer على spi0.0 تبعت 8 bytes ووصّل 8 bytes — سليم.

---

#### 4. printk / dynamic debug

**الكود بيستخدم** `dev_dbg()` في أكثر من مكان — دي بتشتغل بس لو الـ dynamic debug أُفعِّل:

```bash
# فعّل كل debug messages لـ spidev
echo 'module spidev +p' > /sys/kernel/debug/dynamic_debug/control

# أو على مستوى ملف معين
echo 'file drivers/spi/spidev.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل مع stack trace
echo 'file drivers/spi/spidev.c +ps' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ messages
dmesg -w | grep spidev
```

**الـ messages اللي هتظهر** (من الكود نفسه):
```
spidev spi0.0: spi mode 3        ← بعد SPI_IOC_WR_MODE
spidev spi0.0: 8 bits per word   ← بعد SPI_IOC_WR_BITS_PER_WORD
spidev spi0.0: 1000000 Hz (max)  ← بعد SPI_IOC_WR_MAX_SPEED_HZ
spidev spi0.0: lsb first         ← بعد SPI_IOC_WR_LSB_FIRST
```

لتفعيل الـ verbose mode (في الكود `#ifdef VERBOSE`) لازم تعمل recompile مع:
```bash
make drivers/spi/spidev.o KCFLAGS="-DVERBOSE"
```

---

#### 5. Kernel Config Options للـ Debugging

| Config | الوظيفة |
|---|---|
| `CONFIG_SPI_DEBUG` | تفعيل debug messages في الـ SPI core |
| `CONFIG_DEBUG_FS` | مطلوب لـ debugfs access |
| `CONFIG_DYNAMIC_DEBUG` | تفعيل `dev_dbg()` و `pr_debug()` |
| `CONFIG_TRACING` | مطلوب للـ ftrace |
| `CONFIG_SPI_SPIDEV` | الـ driver نفسه (=m أو =y) |
| `CONFIG_SPI_SLAVE` | لتفعيل الـ slave mode و `spi_target_abort()` |
| `CONFIG_LOCKDEP` | كشف deadlocks في الـ mutex (spi_lock + buf_lock) |
| `CONFIG_KASAN` | كشف memory corruption في الـ bounce buffers |
| `CONFIG_DEBUG_MUTEXES` | debug الـ mutex operations |
| `CONFIG_FAULT_INJECTION` | محاكاة أخطاء الـ memory allocation |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "CONFIG_SPI|CONFIG_DEBUG_FS|CONFIG_DYNAMIC_DEBUG"
```

---

#### 6. أدوات خاصة بالـ Subsystem

**الـ spidev لا يدعم devlink**، لكن:

```bash
# أداة spi-tools (لو متوفرة)
spi-config --device=/dev/spidev0.0 --query

# strace لمراقبة الـ ioctl calls من userspace
strace -e ioctl -x ./your_spi_app

# hexdump على الـ transfers
strace -e read,write,ioctl -x -s 256 ./your_spi_app 2>&1 | grep -A3 ioctl

# بايثون بسيط لاختبار الـ device
python3 -c "
import spidev
spi = spidev.SpiDev()
spi.open(0, 0)
spi.max_speed_hz = 500000
resp = spi.xfer2([0x9F, 0x00, 0x00])
print(hex(resp[0]), hex(resp[1]), hex(resp[2]))
spi.close()
"

# busybox devmem لقراءة registers الـ controller مباشرة (مثال RPi)
devmem2 0x3F204000 w   # SPI0 CS register على Raspberry Pi
```

---

#### 7. رسائل الأخطاء الشائعة

| رسالة في dmesg | المعنى | الحل |
|---|---|---|
| `spidev: nothing for minor X` | الـ minor number مش موجود في الـ device_list | تأكد إن الـ probe نجح — شوف `dmesg \| grep spidev` |
| `spidev listed directly in DT is not supported` | الـ DT nodes بيستخدم `compatible = "spidev"` مباشرة | استخدم compatible محدد زي `rohm,dh2228fv` |
| `do not use this driver in production systems!` | الـ device match على ACPI SPT000* | بيجي من `spidev_acpi_check()` — طبيعي في dev boards |
| `no minor number available!` | الـ 32 minor numbers اتحجزوا كلهم | `N_SPI_MINORS=32` — مش هتحتاج أكتر في الغالب |
| `spi_setup failed` | الـ controller رفض الإعدادات | شوف قيم mode/speed_hz/bits_per_word |
| `-ESHUTDOWN` من ioctl | الـ SPI device اتشال أثناء العملية | الـ `spidev->spi == NULL` — إعد فتح الـ device |
| `-EMSGSIZE` | الـ transfer أكبر من الـ `bufsiz` | زوّد `bufsiz` module param أو قسّم الـ transfer |
| `-EFAULT` | فشل `copy_from_user` أو `copy_to_user` | pointer من userspace غلط أو NULL |
| `-EINVAL` من `SPI_IOC_WR_MODE` | الـ mode bits فيها bits خارج `SPI_MODE_MASK` | تأكد من الـ flags المدعومة |
| `-ENOTTY` | الـ ioctl command مش من نوع `SPI_IOC_MAGIC` | الـ cmd غلط أو بتبعت ioctl على FD غلط |

---

#### 8. Strategic Points للـ dump_stack() / WARN_ON()

النقاط الاستراتيجية في الكود:

```c
/* في spidev_sync — لو الـ spi pointer NULL أثناء transfer */
static ssize_t spidev_sync(struct spidev_data *spidev, struct spi_message *message)
{
    mutex_lock(&spidev->spi_lock);
    spi = spidev->spi;

    /* نقطة مهمة: لو وصلنا هنا والـ spi NULL، race condition محتملة */
    WARN_ON(!spi && message->actual_length > 0);
    ...
}

/* في spidev_message — لو الـ total تجاوز الـ bufsiz */
if (rx_total > bufsiz) {
    WARN_ON(rx_total > bufsiz * 2); /* تجاوز ضعف الـ limit = bug */
    status = -EMSGSIZE;
    goto done;
}

/* في spidev_release — لو users < 0 */
spidev->users--;
WARN_ON((int)spidev->users < 0); /* underflow = double close bug */

/* في spidev_remove — للتأكد من cleanup صح */
WARN_ON(spidev->users > 0); /* device اتشال وفيه open fds */
```

---

### Hardware Level

#### 1. التحقق من حالة الـ Hardware مقابل الـ Kernel State

```bash
# قارن الـ speed المضبوطة في kernel مع ما يدعمه الـ hardware
cat /sys/bus/spi/devices/spi0.0/max_speed_hz
# لو الـ controller ضبّطها أقل من اللي طلبته، شوف:
dmesg | grep "spi0.0"
# مثال: spi0.0: setup: speed 500000 Hz (max)

# تأكد من الـ chip select polarity
cat /sys/bus/spi/devices/spi0.0/modalias
# إذا الـ chip select GPIO-based:
cat /sys/kernel/debug/gpio | grep "spi"

# تحقق من binding الـ driver
ls -la /sys/bus/spi/devices/spi0.0/driver
# لو مفيش symlink = الـ probe فشل
```

---

#### 2. Register Dump Techniques

**مثال على Raspberry Pi (BCM2835 SPI0):**

```bash
# SPI0 base address على RPi: 0x3F204000 (RPi2/3) أو 0xFE204000 (RPi4)

# قراءة CS register (control و status)
devmem2 0x3F204000 w
# bit 0-1: CS — الـ chip select المفعّل
# bit 2: CPHA
# bit 3: CPOL
# bit 4-5: CLEAR (clear FIFOs)
# bit 7: TA (Transfer Active)
# bit 8: DMAEN
# bit 16: DONE

# قراءة CLK divider (الـ speed)
devmem2 0x3F204008 w
# CLK = Core_freq / CDIV

# قراءة DLEN (data length)
devmem2 0x3F20400C w

# بديل بدون devmem2:
dd if=/dev/mem bs=4 count=1 skip=$((0x3F204000/4)) 2>/dev/null | xxd
```

**لو مش عارف الـ base address:**

```bash
# من الـ device tree المُفعَّل
cat /proc/device-tree/soc/spi@7e204000/reg | xxd
# أو
dtc -I fs /proc/device-tree 2>/dev/null | grep -A5 "spi@"
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

```
                    SPI Transfer على Logic Analyzer:

    CS   ‾‾‾‾‾‾‾‾‾|___________________________|‾‾‾‾‾‾‾‾‾
    CLK  __________|‾|_|‾|_|‾|_|‾|_|‾|_|‾|_|‾|________
    MOSI ~~~~~~~~~~|D7|D6|D5|D4|D3|D2|D1|D0|~~~~~~~~~~~
    MISO ~~~~~~~~~~|Q7|Q6|Q5|Q4|Q3|Q2|Q1|Q0|~~~~~~~~~~~
                   ^                           ^
              CS goes LOW                 CS goes HIGH
```

**نقاط مهمة للـ measurement:**

| ما تقيسه | الدلالة |
|---|---|
| CS setup time (CS↓ → CLK أول edge) | لازم > 50ns للأجهزة الكتير |
| CLK frequency | قارنه بـ `max_speed_hz` من sysfs |
| CPOL في idle | لازم يتطابق مع `spi->mode & SPI_CPOL` |
| CPHA (أول edge capture أو shift) | يتطابق مع `spi->mode & SPI_CPHA` |
| CS polarity في idle | الـ default LOW، لو MOSI stuck LOW تحقق |
| Byte gaps | لو فيه gap بين bytes مش متوقع، شوف `word_delay_usecs` |

**إعدادات Logic Analyzer:**
- Sample rate: على الأقل 10x من CLK speed (مثلاً CLK=1MHz → sample at 10MHz)
- Trigger: على falling edge من CS
- Decode: SPI protocol مع CPOL/CPHA حسب الـ mode المضبوط

---

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة | ما تشوفه في dmesg | السبب |
|---|---|---|
| MISO floating (مش متوصل) | بيانات عشوائية في rx_buf — مفيش error | مفيش pull resistor أو الـ device مش powered |
| CLK speed أعلى من اللي الـ slave يدعمه | `spi_setup failed: -EINVAL` أو بيانات خاطئة | قلّل `max_speed_hz` |
| CS polarity معكوسة | الـ device مش بيرد — timeout أو all 0xFF | فعّل/عطّل `SPI_CS_HIGH` في الـ mode |
| الـ device مش powered | `spi_sync returned -EIO` | تحقق من الـ power supply |
| noise على CLK | بيانات خاطئة بس كلام — `actual_length` صح | قلّل الـ speed أو استخدم decoupling caps |
| missing GND connection | transfers تنجح أحياناً وأحياناً لأ | شوف الـ common ground |
| MOSI/MISO معكوسين | كل response غلط أو `0x00` | swap الـ wires |

---

#### 5. Device Tree Debugging

```bash
# تحقق من الـ DT المُحمَّل فعلياً
dtc -I fs -O dts /proc/device-tree 2>/dev/null > /tmp/current_dt.dts
grep -A20 "spidev\|spi@" /tmp/current_dt.dts

# مثال DT صحيح لـ spidev:
cat /proc/device-tree/soc/spi@7e204000/spidev@0/compatible
# output يجب أن يكون مثلاً: rohm,dh2228fv
# وليس: spidev (ده بيسبب error)

# تحقق من الـ reg (chip select number)
cat /proc/device-tree/soc/spi@7e204000/spidev@0/reg | xxd
# 00 00 00 00 = CS0

# تحقق من spi-max-frequency
cat /proc/device-tree/soc/spi@7e204000/spidev@0/spi-max-frequency | xxd
# 00 07 a1 20 = 500000 Hz

# تأكد إن الـ DT node اتفسّر صح
ls /sys/bus/spi/devices/
# لازم تلاقي spi0.0 موجود

# لو الـ device مش ظاهر في sysfs:
dmesg | grep -i "spi\|of_spi\|spidev" | head -30
```

**مثال DT صحيح:**

```dts
&spi0 {
    status = "okay";
    spidev@0 {
        compatible = "rohm,dh2228fv";   /* NOT "spidev" */
        reg = <0>;                       /* CS0 */
        spi-max-frequency = <500000>;
    };
};
```

---

### Practical Commands

#### جاهز للـ Copy — كل حاجة في مكان واحد

```bash
# ============================================================
# 1. تحقق إن الـ driver محمّل والـ device موجود
# ============================================================
lsmod | grep spidev
ls /dev/spidev*
ls -la /sys/class/spidev/

# ============================================================
# 2. معلومات الـ device
# ============================================================
SPIDEV=/dev/spidev0.0
SPI_SYS=/sys/bus/spi/devices/spi0.0

cat $SPI_SYS/modalias
cat $SPI_SYS/max_speed_hz
cat $SPI_SYS/bits_per_word
cat $SPI_SYS/chip_select

# ============================================================
# 3. تفعيل dynamic debug
# ============================================================
echo 'file drivers/spi/spidev.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'module spidev +p' > /sys/kernel/debug/dynamic_debug/control
dmesg -w | grep spidev &

# ============================================================
# 4. تفعيل SPI tracing
# ============================================================
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo > /sys/kernel/debug/tracing/trace
echo 1 > /sys/kernel/debug/tracing/events/spi/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل الـ SPI application هنا
# ./your_app

echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace

# ============================================================
# 5. strace على SPI ioctl
# ============================================================
# (على جهاز development فقط)
cat > /tmp/spi_test.py << 'EOF'
import fcntl, struct, os

SPI_IOC_MAGIC = ord('k')
SPI_IOC_RD_MODE = 0x80016b01
SPI_IOC_RD_MAX_SPEED_HZ = 0x80046b04

fd = os.open("/dev/spidev0.0", os.O_RDWR)
buf = struct.pack('B', 0)
result = fcntl.ioctl(fd, SPI_IOC_RD_MODE, buf)
print(f"SPI Mode: {result[0]:#04x}")
buf = struct.pack('I', 0)
result = fcntl.ioctl(fd, SPI_IOC_RD_MAX_SPEED_HZ, buf)
print(f"Max Speed: {struct.unpack('I', result)[0]} Hz")
os.close(fd)
EOF
strace -e ioctl python3 /tmp/spi_test.py 2>&1

# ============================================================
# 6. اختبار بسيط بالـ read/write
# ============================================================
# كتابة byte واحد (0xAA) وقراءة الرد
python3 -c "
import os, struct
fd = os.open('/dev/spidev0.0', os.O_RDWR)
# SPI_IOC_MESSAGE(1) — transfer واحد full duplex
SPI_IOC_MESSAGE_1 = 0x40206b00
tx_buf = (0xAA).to_bytes(1, 'little')
xfer = struct.pack('QQIIHBBBBBx',
    int.from_bytes(tx_buf, 'little'),  # tx_buf ptr — simplified
    0,      # rx_buf
    1,      # len
    500000, # speed_hz
    0,      # delay_usecs
    8,      # bits_per_word
    0,      # cs_change
    0,      # tx_nbits
    0,      # rx_nbits
    0,      # word_delay_usecs
)
print('Use spidev library for proper ioctl usage')
os.close(fd)
"

# ============================================================
# 7. تحقق من الـ minor numbers المستخدمة
# ============================================================
cat /proc/devices | grep "153"
# 153 spi  ← الـ major number الخاص بـ spidev

# ============================================================
# 8. تغيير الـ bufsiz parameter
# ============================================================
# لو عايز transfers أكبر من 4096 bytes:
modprobe spidev bufsiz=65536
# أو لو محمّل:
# مش ممكن تغييره runtime — محتاج rmmod وmodprobe من جديد
cat /sys/module/spidev/parameters/bufsiz

# ============================================================
# 9. تحقق من الـ DT matching
# ============================================================
dmesg | grep -E "spidev|spi.*probe|of.*spi"
# لو شايف: "spidev listed directly in DT is not supported"
# → غيّر الـ compatible في الـ DT

# ============================================================
# 10. فحص الـ mutex state لو في deadlock مشتبه
# ============================================================
# (محتاج CONFIG_LOCKDEP مفعّل)
cat /proc/lockdep_stats
echo w > /proc/sysrq-trigger   # dump blocked tasks
dmesg | tail -50
```

**تفسير output الـ trace:**

```
# مثال trace output بعد transfer:
#
#        TASK       PID    CPU  TIMESTAMP  FUNCTION
# spi_test-1234  [000]  100.1: spi_message_submit: spi0.0 0xffff888012345678
# spi_test-1234  [000]  100.1: spi_message_start:  spi0.0 0xffff888012345678
# spi_test-1234  [000]  100.2: spi_transfer_start: spi0.0 0xffff888... len=4 tx=[aa bb cc dd] rx=[]
# spi_test-1234  [000]  100.3: spi_transfer_stop:  spi0.0 0xffff888... len=4 tx=[] rx=[11 22 33 44]
# spi_test-1234  [000]  100.3: spi_message_done:   spi0.0 0xffff888... len=4/4
#
# len=4/4 → actual=4 من frame=4 → transfer ناجح بالكامل
# لو len=0/4 → مفيش بيانات اتبعتت = مشكلة في الـ controller
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — بيانات مشفرة غلط بسبب SPI Mode خاطئ

#### العنوان
**SPI Mode mismatch** بين userspace daemon وـ flash memory في industrial gateway

#### السياق
شركة بتبني industrial IoT gateway بـ RK3562. الـ gateway بيقرأ configuration من SPI NOR flash خارجي (W25Q64) عن طريق `/dev/spidev0.0`. الـ userspace daemon مكتوب بـ Python يستخدم `spidev` library.

#### المشكلة
بعد flashing الـ firmware الجديد، الـ daemon بدأ يرجع بيانات مشوهة من الـ flash. أحياناً أول byte صح وبعده garbage. الـ W25Q64 بيشتغل على **SPI Mode 0** (CPOL=0, CPHA=0) بس الكود القديم كان بيحدد Mode 3 بالغلط وكان شغال بالصدفة مع controller تاني.

#### التحليل
الـ `spidev_ioctl` في `spidev.c` بيتعامل مع `SPI_IOC_WR_MODE`:

```c
case SPI_IOC_WR_MODE:
    retval = get_user(tmp, (u8 __user *)arg);
    if (retval == 0) {
        u32 save = spi->mode;

        if (tmp & ~SPI_MODE_MASK) {
            retval = -EINVAL;
            break;
        }
        /* يحفظ الـ bits اللي مش في SPI_MODE_MASK من القيمة القديمة */
        tmp |= spi->mode & ~SPI_MODE_MASK;
        spi->mode = tmp & SPI_MODE_USER_MASK;
        retval = spi_setup(spi);
        if (retval < 0)
            spi->mode = save;  /* rollback لو فشل */
    }
```

لما الـ daemon بيعمل `SPI_IOC_WR_MODE` بقيمة `SPI_MODE_3` (=3)، الكود بيمرر `spi_setup()` بدون error لأن الـ RK3562 controller بيدعم Mode 3، لكن الـ W25Q64 المتوصل فعلياً بيتوقع Mode 0 — فكل clock edge بيقرأ data في اللحظة الغلط.

الـ `spidev_sync_read` بتعمل transfer واحد:
```c
static inline ssize_t
spidev_sync_read(struct spidev_data *spidev, size_t len)
{
    struct spi_transfer t = {
        .rx_buf   = spidev->rx_buffer,
        .len      = len,
        .speed_hz = spidev->speed_hz,  /* mode مش هنا، هو في spi->mode */
    };
    ...
}
```

الـ mode مش موجود في `spi_transfer` مباشرة — بيتاخد من `spi_device->mode` اللي اتضبط غلط.

#### الحل

**في الـ DT:**
```dts
/* arch/arm64/boot/dts/rockchip/rk3562-gateway.dts */
&spi0 {
    status = "okay";
    w25q64: flash@0 {
        compatible = "rohm,dh2228fv"; /* يستخدم spidev */
        reg = <0>;
        spi-max-frequency = <50000000>;
        spi-cpol;    /* Mode بدون هاتين يبقى Mode 0 */
        /* لا spi-cpha ولا spi-cpol = Mode 0 */
    };
};
```

**في الـ userspace daemon:**
```python
import spidev

spi = spidev.SpiDev()
spi.open(0, 0)

# اقرأ الـ mode الحالي أول
# SPI_IOC_RD_MODE → ioctl(fd, 0x80016B01, ...)
current_mode = spi.mode
print(f"Current mode: {current_mode}")

# صحح للـ Mode 0
spi.mode = 0  # يستدعي SPI_IOC_WR_MODE داخلياً
spi.max_speed_hz = 50_000_000

# قراءة
data = spi.readbytes(256)
```

**تحقق من الـ mode الفعلي في kernel:**
```bash
# اقرأ الـ mode من /sys
cat /sys/bus/spi/devices/spi0.0/modalias

# استخدم spi-tools
spi-config -d /dev/spidev0.0 -q
# Output: /dev/spidev0.0: mode=0, lsb=0, bits=8, speed=50000000, spiready=0
```

#### الدرس المستفاد
الـ `spi_setup()` في `spidev_ioctl` بترجع 0 حتى لو الـ mode مش مناسب للـ hardware المتوصل — الكرنل مش بيعرف إيه الـ IC على الطرف التاني. التحقق من الـ mode لازم يكون في الـ bring-up الأول عن طريق oscilloscope أو logic analyzer، مش بس بـ software.

---

### السيناريو 2: Android TV Box على Allwinner H616 — `/dev/spidev` مش بيظهر بعد boot

#### العنوان
الـ `spidev` device node مش بيتكون رغم أن الـ driver محمل

#### السياق
فريق بيعمل Android TV box بـ Allwinner H616. محتاجين يوصلوا SPI touch controller خارجي (XPT2046) عن طريق `/dev/spidev1.0` لأن الـ touch driver الخاص ببطئ. قرروا يستخدموا `spidev` كـ workaround مؤقت في userspace.

#### المشكلة
بعد إضافة الـ DT node، `dmesg` مفيش فيه errors، الـ module متحمل، بس `/dev/spidev1.0` مش موجود.

```bash
ls /dev/spidev*
# ls: cannot access '/dev/spidev*': No such file or directory

lsmod | grep spidev
# spidev   20480  0

dmesg | grep spidev
# مفيش أي رسالة probe
```

#### التحليل
الـ `spidev_probe` بيشتغل بس لو الـ compatible string موجود في `spidev_dt_ids[]` وبيعدي الـ `spidev_of_check`:

```c
static int spidev_of_check(struct device *dev)
{
    /* لو الـ compatible string هو "spidev" بالظبط → error */
    if (device_property_match_string(dev, "compatible", "spidev") < 0)
        return 0;

    dev_err(dev, "spidev listed directly in DT is not supported\n");
    return -EINVAL;
}

static const struct of_device_id spidev_dt_ids[] = {
    { .compatible = "rohm,dh2228fv", .data = &spidev_of_check },
    { .compatible = "semtech,sx1301", .data = &spidev_of_check },
    /* ... قائمة محدودة جداً من الـ compatible strings */
    {},
};
```

المشكلة: الـ engineer كتب في الـ DT:
```dts
xpt2046: touchscreen@0 {
    compatible = "xpt2046";  /* مش موجود في spidev_dt_ids */
    reg = <0>;
    spi-max-frequency = <2000000>;
};
```

الـ `xpt2046` مش في `spidev_dt_ids` ولا في `spidev_spi_ids` — فالـ probe مش بيتشغل أصلاً.

حتى لو حط `compatible = "spidev"`:
```c
/* spidev_of_check بترجع -EINVAL */
match = device_get_match_data(&spi->dev);
if (match) {
    status = match(&spi->dev);
    if (status)
        return status;  /* الـ probe بيفشل هنا */
}
```

#### الحل

استخدم واحد من الـ compatible strings الموجودة في `spidev_dt_ids`:

```dts
/* arch/arm64/boot/dts/allwinner/sun50i-h616-tvbox.dts */
&spi1 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&spi1_pins>;

    xpt2046: touchscreen@0 {
        /* استخدم compatible معروف يناسب spidev */
        compatible = "rohm,dh2228fv";
        reg = <0>;
        spi-max-frequency = <2000000>;
        spi-cpol;
        spi-cpha;  /* XPT2046 → Mode 3 */
    };
};
```

تحقق إن الـ probe شغال:
```bash
dmesg | grep -i spi
# [    2.345] spidev spi1.0: buggy DT: spidev listed directly...
# أو
# [    2.345] spi1.0: spidev

ls -la /dev/spidev*
# crw-rw---- 1 root spi 153, 0 Feb 25 10:00 /dev/spidev1.0
```

لو محتاج compatible مخصص، لازم تضيفه في kernel source:
```c
/* في spidev.c */
static const struct of_device_id spidev_dt_ids[] = {
    /* ... الموجودة */
    { .compatible = "ti,xpt2046", .data = &spidev_of_check },
    {},
};
```

#### الدرس المستفاد
الـ `spidev` driver عمداً محدود الـ compatible strings المقبولة — ده حماية من استخدامه على أي hardware عشوائي. فهم `spidev_dt_ids[]` وآلية `spidev_of_check` ضروري قبل ما تتوقع إن الـ probe هيشتغل.

---

### السيناريو 3: IoT Sensor Node على STM32MP1 — data corruption بسبب `bufsiz` صغير

#### العنوان
الـ `SPI_IOC_MESSAGE` بيرجع `-EMSGSIZE` لما تحاول ترسل 8KB دفعة واحدة

#### السياق
فريق يبني IoT sensor node بـ STM32MP1 (Linux side). السنسور هو IMU بـ SPI interface بيرجع 8192 bytes من الـ FIFO في قراءة واحدة (512 samples × 16 bytes). الكود بيستخدم `SPI_IOC_MESSAGE` لعمل full-duplex transfer.

#### المشكلة
الكود C التالي بيفشل بـ `errno = EMSGSIZE`:

```c
struct spi_ioc_transfer xfer = {
    .tx_buf = (unsigned long)tx_buf,
    .rx_buf = (unsigned long)rx_buf,
    .len    = 8192,   /* 8KB */
    .speed_hz = 10000000,
};

int ret = ioctl(fd, SPI_IOC_MESSAGE(1), &xfer);
// ret = -1, errno = EMSGSIZE (90)
```

#### التحليل
في `spidev_message`:

```c
if (u_tmp->rx_buf) {
    rx_total += len_aligned;  /* ALIGN(8192, ARCH_DMA_MINALIGN) */
    if (rx_total > bufsiz) {  /* bufsiz = 4096 افتراضياً */
        status = -EMSGSIZE;
        goto done;
    }
    k_tmp->rx_buf = rx_buf;
    rx_buf += len_aligned;
}
```

الـ `bufsiz` الافتراضي هو **4096 bytes** — معرف في:

```c
static unsigned bufsiz = 4096;
module_param(bufsiz, uint, S_IRUGO);
MODULE_PARM_DESC(bufsiz, "data bytes in biggest supported SPI message");
```

وفي `spidev_open`:
```c
spidev->tx_buffer = kmalloc(bufsiz, GFP_KERNEL);
/* ...  */
spidev->rx_buffer = kmalloc(bufsiz, GFP_KERNEL);
```

الـ buffers اتعملت بـ `kmalloc` بحجم `bufsiz` في وقت الـ open. المقارنة في `spidev_message` بتتحقق إن `rx_total` و `tx_total` مش بيتعدوا `bufsiz`. لو حجم الـ transfer (8192) أكبر من `bufsiz` (4096) → `-EMSGSIZE`.

#### الحل

**Option 1:** تمرير `bufsiz` كـ module parameter:

```bash
# في /etc/modprobe.d/spidev.conf
options spidev bufsiz=16384

# أو مؤقتاً
rmmod spidev
modprobe spidev bufsiz=16384

# تحقق
cat /sys/module/spidev/parameters/bufsiz
# 16384
```

**Option 2:** في الـ kernel command line:

```
spidev.bufsiz=16384
```

**Option 3:** تقسيم الـ transfer في userspace:

```c
#define CHUNK_SIZE 4096
#define TOTAL_SIZE 8192

int spi_read_imu_fifo(int fd, uint8_t *rx_buf) {
    int ret, offset = 0;

    while (offset < TOTAL_SIZE) {
        size_t chunk = MIN(CHUNK_SIZE, TOTAL_SIZE - offset);
        struct spi_ioc_transfer xfer = {
            .tx_buf   = (unsigned long)(tx_buf + offset),
            .rx_buf   = (unsigned long)(rx_buf + offset),
            .len      = chunk,
            .speed_hz = 10000000,
        };

        ret = ioctl(fd, SPI_IOC_MESSAGE(1), &xfer);
        if (ret < 0) return ret;
        offset += chunk;
    }
    return TOTAL_SIZE;
}
```

**تحذير:** تقسيم الـ transfer بيرفع الـ CS بين الـ chunks (CS toggle بين كل ioctl) — بعض الـ sensors بتعتبر ده reset للـ FIFO pointer.

#### الدرس المستفاد
الـ `bufsiz` module parameter هو single point of control لحجم الـ tx_buffer وـ rx_buffer اللي بيتعمل `kmalloc` في `spidev_open`. أي transfer أكبر من `bufsiz` هيتـ reject في `spidev_message` بـ `-EMSGSIZE`. لازم تحدده صح من أول bring-up بناءً على أكبر transfer ممكن في التطبيق.

---

### السيناريو 4: Automotive ECU على i.MX8 — race condition لما بتفصل الـ device وفيه transaction جارية

#### العنوان
Kernel panic أو use-after-free لما الـ SPI device بيتفصل (hot-unplug) وفيه `ioctl` شغال

#### السياق
automotive ECU بـ i.MX8MQ. الـ ECU بيتواصل مع peripheral module عن طريق SPI. الـ module ده بيتفصل ويتوصل dynamically (hot-swap). الـ userspace daemon شغال بـ continuous loop بيعمل `SPI_IOC_MESSAGE` كل millisecond.

#### المشكلة
عند فصل الـ module، بيظهر أحياناً kernel oops أو الـ daemon بيرجع `-ESHUTDOWN` بس أحياناً data مشوهة. في حالات نادرة: kernel panic بـ NULL pointer dereference.

#### التحليل
الـ locking mechanism في `spidev_ioctl`:

```c
static long spidev_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    /* ... */
    spidev = filp->private_data;
    mutex_lock(&spidev->spi_lock);
    spi = spi_dev_get(spidev->spi);  /* atomic reference count */
    if (spi == NULL) {
        mutex_unlock(&spidev->spi_lock);
        return -ESHUTDOWN;  /* الـ device اتفصل قبل الـ ioctl */
    }
    /* ... */
    mutex_unlock(&spidev->buf_lock);
    spi_dev_put(spi);            /* رجع الـ reference */
    mutex_unlock(&spidev->spi_lock);
    return retval;
}
```

وفي `spidev_remove`:
```c
static void spidev_remove(struct spi_device *spi)
{
    struct spidev_data *spidev = spi_get_drvdata(spi);

    mutex_lock(&device_list_lock);
    mutex_lock(&spidev->spi_lock);
    spidev->spi = NULL;   /* يصفر الـ pointer */
    mutex_unlock(&spidev->spi_lock);

    list_del(&spidev->device_entry);
    device_destroy(&spidev_class, spidev->devt);
    clear_bit(MINOR(spidev->devt), minors);
    if (spidev->users == 0)
        kfree(spidev);    /* يحذف البيانات لو مفيش users */
    mutex_unlock(&device_list_lock);
}
```

المشكلة الدقيقة: الـ `spidev_message` بيستدعي `spidev_sync_unlocked` **بعد** ما يشيل `spi_lock`:

```c
/* في spidev_message — بيشتغل مع buf_lock بس، مش spi_lock */
status = spidev_sync_unlocked(spidev->spi, &msg);
```

بس `spidev->spi` اتاخد من `spidev_sync`:
```c
static ssize_t spidev_sync(struct spidev_data *spidev, struct spi_message *message)
{
    ssize_t status;
    struct spi_device *spi;

    mutex_lock(&spidev->spi_lock);
    spi = spidev->spi;           /* قرأ الـ pointer */
    if (spi == NULL)
        status = -ESHUTDOWN;
    else
        status = spidev_sync_unlocked(spi, message);  /* استخدمه */
    mutex_unlock(&spidev->spi_lock);
    return status;
}
```

الـ `spidev_sync` صح — بتاخد `spi_lock` وبتحقق. لكن `spidev_message` بيستدعي `spidev_sync_unlocked` مباشرة مع `spidev->spi` بدون lock إضافي لأنه متوقع إن الـ caller (spidev_ioctl) بيمسك `spi_lock` طول الوقت عن طريق `spi_dev_get`. ده بيحمي في الغالب، بس لو الـ ref count logic فيه edge case مع معين kernel version...

#### الحل

**في الـ userspace daemon — handle -ESHUTDOWN بشكل صح:**

```c
#include <errno.h>
#include <string.h>

int spi_transfer_safe(int fd, uint8_t *tx, uint8_t *rx, size_t len) {
    struct spi_ioc_transfer xfer = {
        .tx_buf   = (unsigned long)tx,
        .rx_buf   = (unsigned long)rx,
        .len      = len,
        .speed_hz = 5000000,
    };

    int ret = ioctl(fd, SPI_IOC_MESSAGE(1), &xfer);
    if (ret < 0) {
        if (errno == 108) { /* ESHUTDOWN */
            syslog(LOG_WARNING, "SPI device removed, closing fd");
            close(fd);
            fd = -1;
            return -ESHUTDOWN;
        }
        syslog(LOG_ERR, "SPI ioctl error: %s", strerror(errno));
        return ret;
    }
    return ret;
}
```

**في الـ daemon loop:**
```c
while (running) {
    if (fd < 0) {
        /* انتظر ظهور الـ device من جديد */
        usleep(100000); /* 100ms */
        fd = open("/dev/spidev0.0", O_RDWR);
        continue;
    }
    spi_transfer_safe(fd, tx, rx, 64);
    usleep(1000); /* 1ms */
}
```

**استخدم `inotify` للـ device hotplug:**
```c
int ifd = inotify_init();
inotify_add_watch(ifd, "/dev", IN_CREATE | IN_DELETE);
/* راقب events لـ "spidev0.0" */
```

#### الدرس المستفاد
الـ `spidev_remove` بيصفر `spidev->spi` ويـ free الـ struct لو `users == 0`. الـ `spidev_ioctl` بيستخدم `spi_dev_get` كحماية. الـ userspace لازم يتعامل مع `-ESHUTDOWN` كـ signal إن الـ device اتفصل ويعمل re-open بعد ظهوره من جديد — مش يـ retry الـ ioctl على نفس الـ fd.

---

### السيناريو 5: Custom Board Bring-up على AM62x — 32-bit userspace على 64-bit kernel وـ compat ioctl مكسور

#### العنوان
الـ `SPI_IOC_MESSAGE` بيفشل بـ `-EFAULT` لما 32-bit application بيشتغل على 64-bit AM62x kernel

#### السياق
Texas Instruments AM62x (Cortex-A53 64-bit). الـ SDK الخاص بالعميل فيه 32-bit ARM toolchain قديم. الـ application C المترجمة بـ 32-bit بتحاول تتواصل مع SPI ADC (ADS8688) عن طريق `spidev`.

#### المشكلة
نفس الكود شغال تمام على Raspberry Pi 32-bit، بس على الـ AM62x 64-bit kernel:

```bash
./spi_adc_app
# ioctl SPI_IOC_MESSAGE failed: Bad address (EFAULT)
```

#### التحليل
المشكلة في تمثيل الـ pointers داخل `struct spi_ioc_transfer`:

```c
/* من include/uapi/linux/spi/spidev.h */
struct spi_ioc_transfer {
    __u64   tx_buf;   /* دايماً 64-bit في الـ kernel ABI */
    __u64   rx_buf;   /* دايماً 64-bit */
    __u32   len;
    __u32   speed_hz;
    __u16   delay_usecs;
    __u8    bits_per_word;
    __u8    cs_change;
    /* ... */
};
```

الـ struct محدد بـ `__u64` للـ pointers عشان يكون consistent بين 32-bit و64-bit userspace. بس 32-bit application لما بتعمل:

```c
/* 32-bit app */
uint8_t tx[4], rx[4];
struct spi_ioc_transfer xfer;
memset(&xfer, 0, sizeof(xfer));
xfer.tx_buf = (unsigned long)tx;  /* 32-bit: بيحط قيمة 32-bit في __u64 field */
xfer.rx_buf = (unsigned long)rx;
```

الـ `unsigned long` في 32-bit app هو 4 bytes، بس الـ `__u64` field هو 8 bytes. الـ upper 4 bytes بتتملى بـ garbage أو zero تبعاً للـ alignment. الكرنل في `spidev_message` بيحاول يعمل:

```c
if (copy_from_user(tx_buf, (const u8 __user *)
                (uintptr_t) u_tmp->tx_buf,  /* الـ pointer ده 64-bit مع garbage */
            u_tmp->len))
    goto done;  /* بيرجع -EFAULT */
```

الـ `spidev_compat_ioc_message` موجود لمعالجة ده:

```c
#ifdef CONFIG_COMPAT
static long spidev_compat_ioc_message(struct file *filp, unsigned int cmd,
        unsigned long arg)
{
    struct spi_ioc_transfer __user *u_ioc;
    /* ... */
    u_ioc = (struct spi_ioc_transfer __user *) compat_ptr(arg);
    /* ... */
    /* Convert buffer pointers */
    for (n = 0; n < n_ioc; n++) {
        ioc[n].rx_buf = (uintptr_t) compat_ptr(ioc[n].rx_buf);
        ioc[n].tx_buf = (uintptr_t) compat_ptr(ioc[n].tx_buf);
    }
    retval = spidev_message(spidev, ioc, n_ioc);
    /* ... */
}

static long spidev_compat_ioctl(struct file *filp, unsigned int cmd,
        unsigned long arg)
{
    if (_IOC_TYPE(cmd) == SPI_IOC_MAGIC
            && _IOC_NR(cmd) == _IOC_NR(SPI_IOC_MESSAGE(0))
            && _IOC_DIR(cmd) == _IOC_WRITE)
        return spidev_compat_ioc_message(filp, cmd, arg);

    return spidev_ioctl(filp, cmd, (unsigned long)compat_ptr(arg));
}
#endif
```

الـ `compat_ioctl` بيحول الـ 32-bit pointers صح. المشكلة: الكرنل بيوجه الـ ioctl لـ `compat_ioctl` تلقائياً **لو** الـ app اتعمل compile كـ 32-bit والكرنل 64-bit. بس لو `CONFIG_COMPAT` مش enabled في kernel config → `spidev_compat_ioctl` = NULL → بيروح لـ `spidev_ioctl` العادي بدون pointer conversion.

#### الحل

**تحقق من الـ kernel config:**
```bash
zcat /proc/config.gz | grep CONFIG_COMPAT
# CONFIG_COMPAT=y  ← لازم يكون موجود
# CONFIG_COMPAT_32=y
```

لو مش موجود، أضفه في الـ kernel defconfig:
```
# arch/arm64/configs/am62x_defconfig
CONFIG_COMPAT=y
CONFIG_COMPAT_BINFMT_ELF=y
```

**تحقق إن الـ app 32-bit فعلاً:**
```bash
file ./spi_adc_app
# spi_adc_app: ELF 32-bit LSB executable, ARM, ...

# شوف الـ ioctl calls
strace -e ioctl ./spi_adc_app 2>&1 | grep spi
# ioctl(3, SPI_IOC_MESSAGE(1) or SPI_IOC_MESSAGE_1, ...) = -1 EFAULT
```

**Fix في الـ userspace كـ workaround مؤقت:**

```c
/* 32-bit app — صريح في تهيئة الـ struct */
struct spi_ioc_transfer xfer;
memset(&xfer, 0, sizeof(xfer));

/* استخدم __u64 cast صريح عشان تضمن zero extension */
xfer.tx_buf = (__u64)(uintptr_t)tx_buf;
xfer.rx_buf = (__u64)(uintptr_t)rx_buf;
xfer.len    = transfer_len;
xfer.speed_hz = 5000000;
xfer.bits_per_word = 8;

int ret = ioctl(fd, SPI_IOC_MESSAGE(1), &xfer);
```

**الحل النهائي: compile الـ app كـ 64-bit:**
```bash
aarch64-linux-gnu-gcc -o spi_adc_app_64 spi_adc.c
./spi_adc_app_64
# شغال تمام
```

#### الدرس المستفاد
الـ `spi_ioc_transfer` struct استخدم `__u64` للـ pointers عشان يكون compatible بين 32/64-bit — بس ده يشتغل صح بس لو `CONFIG_COMPAT=y` في الكرنل والكرنل بيوجه الـ syscall لـ `spidev_compat_ioctl`. لما تعمل bring-up على AM62x أو أي SoC 64-bit جديد، تأكد من الـ compat config قبل ما تستخدم 32-bit toolchain على 64-bit kernel.
## Phase 7: مصادر ومراجع

### التوثيق الرسمي للـ Kernel

| المصدر | الوصف |
|--------|-------|
| [`Documentation/spi/spidev.rst`](https://docs.kernel.org/spi/spidev.html) | الـ official userspace API doc للـ spidev — شرح الـ ioctl commands والـ device node وتحذيرات الـ production |
| [`Documentation/spi/spi-summary.rst`](https://docs.kernel.org/spi/spi-summary.html) | نظرة عامة على الـ SPI subsystem — الـ bus model والـ controller والـ device registration |
| [`Documentation/spi/spi-ldisc.rst`](https://docs.kernel.org/spi/index.html) | فهرس كامل لتوثيق الـ SPI في الـ kernel |
| [`include/uapi/linux/spi/spidev.h`](https://github.com/torvalds/linux/blob/master/include/uapi/linux/spi/spidev.h) | تعريفات الـ ioctl و `spi_ioc_transfer` struct للـ userspace |
| [`include/linux/spi/spi.h`](https://github.com/torvalds/linux/blob/master/include/linux/spi/spi.h) | الـ core SPI API — `spi_device`، `spi_message`، `spi_transfer` |

---

### مقالات LWN.net

الـ LWN.net هو المرجع الأساسي لتتبع تطور الـ Linux kernel subsystems.

| المقال | الأهمية |
|--------|---------|
| [SPI userspace API — kernel docs mirror](https://static.lwn.net/kerneldoc/spi/spidev.html) | نسخة من الـ official kernel doc على LWN |
| [simple SPI framework (2006)](https://lwn.net/Articles/154425/) | الـ patch الأصلي اللي أدخل الـ SPI subsystem للـ kernel — David Brownell |
| [SPI redux — driver model support](https://lwn.net/Articles/149698/) | تطوير الـ SPI framework ودعم الـ driver model |
| [SPI core refresh](https://lwn.net/Articles/162268/) | إعادة كتابة الـ SPI core ودعم الـ message queue |
| [spi: Add slave mode support](https://lwn.net/Articles/723440/) | إضافة الـ SPI slave mode — يستخدم spidev في الـ testing |
| [Serial Peripheral Interface (SPI) — kernel driver API](https://static.lwn.net/kerneldoc/driver-api/spi.html) | الـ driver API الكامل للـ SPI على LWN |

---

### صفحات kernelnewbies.org

صفحات الـ kernel changelogs اللي بتوثق التغييرات المهمة في الـ spidev عبر الإصدارات:

| الإصدار | التغيير في spidev |
|---------|------------------|
| [Linux 3.15](https://kernelnewbies.org/Linux_3.15-DriversArch) | إضافة **Dual/Quad SPI Transfers** لـ spidev |
| [Linux 4.8](https://kernelnewbies.org/Linux_4.8) | تفعيل spidev على Intel Edison |
| [Linux 4.12](https://kernelnewbies.org/Linux_4.12) | تحسينات على الـ SPI subsystem |
| [Linux 4.14](https://kernelnewbies.org/Linux_4.14) | تحديثات الـ SPI framework |
| [Linux 5.8](https://kernelnewbies.org/Linux_5.8) | إضافة **Octal mode** data transfers لـ spidev |
| [Linux 6.3](https://kernelnewbies.org/Linux_6.3) | إضافة Silicon Labs SI3210 compatible لـ spidev |
| [Linux 6.9](https://kernelnewbies.org/Linux_6.9) | تحديثات على الـ SPI driver |

---

### صفحات elinux.org

أمثلة عملية لاستخدام spidev على الـ embedded boards الشائعة:

| الصفحة | المحتوى |
|--------|---------|
| [RPi SPI](https://elinux.org/RPi_SPI) | إعداد spidev على الـ Raspberry Pi — تجميع spidev_test وتشغيله على `/dev/spidev0.0` |
| [BeagleBone Black Enable SPIDEV](https://elinux.org/BeagleBone_Black_Enable_SPIDEV) | تفعيل spidev عبر الـ device tree على الـ BeagleBone Black |
| [ECE497 SPI Project](https://elinux.org/ECE497_SPI_Project) | مشروع تعليمي لاستخدام spidev — unregister/register الـ driver |

---

### Commits المهمة في الـ Kernel

```
# الـ commit الأصلي لـ spidev (kernel 2.6.19 — 2006)
git log --follow drivers/spi/spidev.c

# commits التاريخية على kernel.org
https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/drivers/spi/spidev.c
```

| التغيير | الوصف |
|---------|-------|
| إضافة spidev (2006) | Andrea Paterniani — الـ initial simple synchronous userspace SPI interface |
| Cleanup (2007) | David Brownell — تبسيط وتنظيف الكود |
| Dual/Quad SPI | إضافة الـ `SPI_TX_DUAL`، `SPI_TX_QUAD` flags |
| Octal SPI | إضافة الـ `SPI_TX_OCTAL`، `SPI_RX_OCTAL` flags |
| Device tree compatible list | ربط الـ spidev بالـ DT nodes بدل الاسم العام |

لعرض الـ commit history:

```bash
git log --oneline --follow drivers/spi/spidev.c
```

---

### نقاشات الـ Mailing List

**الـ mailing list الرئيسية للـ SPI subsystem:**

```
linux-spi@vger.kernel.org
```

للبحث في الأرشيف:

- [lore.kernel.org/linux-spi](https://lore.kernel.org/linux-spi/) — أرشيف كامل لنقاشات الـ SPI
- [marc.info/?l=linux-kernel&s=spidev](https://marc.info/?l=linux-kernel&s=spidev) — بحث في LKML عن spidev

**نقاشات مهمة:**
- تحذير الـ production use — النقاش حول إزالة الاسم العام `"spidev"` من الـ compatible list وإجبار استخدام أسماء حقيقية
- الـ security discussion حول `/dev/spidev*` permissions وخطر الوصول المباشر للـ hardware

---

### مصادر GitHub الرسمية

| الملف | الرابط |
|-------|-------|
| `drivers/spi/spidev.c` | [github.com/torvalds/linux](https://github.com/torvalds/linux/blob/master/drivers/spi/spidev.c) |
| `Documentation/spi/spidev.rst` | [github.com/torvalds/linux](https://github.com/torvalds/linux/blob/master/Documentation/spi/spidev.rst) |
| `tools/spi/spidev_test.c` | [github.com/torvalds/linux](https://github.com/torvalds/linux/blob/master/tools/spi/spidev_test.c) — الـ official test tool |

---

### كتب مرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصل المهم:** Chapter 15 — Memory Mapping and DMA (للـ buffer management)
- **الفصل المهم:** Chapter 3 — Char Drivers (الـ file operations اللي يستخدمها spidev)
- **متاح مجاناً:** [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love
- **الإصدار:** 3rd Edition
- **الفصول المرتبطة:**
  - Chapter 17 — Devices and Modules (الـ char device registration)
  - Chapter 5 — System Calls (الـ ioctl mechanism)
- الكتاب ده بيشرح الـ kernel internals اللي بيبني عليها spidev زي الـ mutex و list management

#### Embedded Linux Primer — Christopher Hallinan
- **الإصدار:** 2nd Edition
- **الفصول المرتبطة:**
  - Chapter 15 — Embedded Linux BSP (الـ SPI device configuration في الـ BSP)
  - الكتاب ده بيشرح إزاي تفعّل spidev في الـ device tree وتستخدمه في الـ userspace application

#### Mastering Embedded Linux Programming — Frank Vasquez & Chris Simmonds
- **الإصدار:** 3rd Edition
- بيشرح استخدام `/dev/spidev*` من الـ userspace مع أمثلة C عملية

---

### مصادر تقنية إضافية

| المصدر | الرابط |
|--------|-------|
| Kernel.org official spidev doc | [kernel.org/doc/Documentation/spi/spidev](https://www.kernel.org/doc/Documentation/spi/spidev) |
| Using spidev with Linux device tree | [yurovsky.github.io](https://yurovsky.github.io/2016/10/07/spidev-linux-devices.html) |
| Toradex SPI Linux guide | [developer.toradex.com](https://developer.toradex.com/linux-bsp/application-development/peripheral-access/spi-linux/) |
| linux-sunxi SPIdev | [linux-sunxi.org/SPIdev](https://linux-sunxi.org/SPIdev) |

---

### Search Terms للبحث عن معلومات أكثر

لو عايز تعمق أكتر في الـ spidev و SPI subsystem:

```
# للبحث في الـ kernel mailing list
site:lore.kernel.org spidev
site:lore.kernel.org "spi_ioc_transfer"

# للبحث عن implementations
site:github.com spidev userspace example
linux spidev ioctl SPI_IOC_MESSAGE example

# للبحث عن مشاكل وحلول
spidev "compatible" device tree warning production
spidev SPI_IOC_WR_MODE32 ioctl

# للبحث عن الـ security considerations
spidev "/dev/spidev" udev permissions security
linux spi userspace interface alternatives

# مصطلحات تقنية مرتبطة
SPI_MODE_0 SPI_MODE_3 CPOL CPHA
spi_sync spi_message spi_transfer kernel
full-duplex SPI transfer userspace
```
## Phase 8: Writing simple module

### الفكرة: Hook على `spidev_sync` باستخدام kprobe

**الـ** `spidev_sync` هي النقطة المركزية اللي بتمر منها كل عملية SPI transfer من userspace — سواء read أو write أو ioctl message. هنعمل **kprobe** عليها عشان نطبع معلومات عن كل transfer: اسم الـ SPI device، وعدد الـ bytes المطلوبة، والـ bus number.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * spidev_spy.c — kprobe on spidev_sync() to trace userspace SPI transfers
 *
 * Hooks into spidev_sync() which is the central path for all
 * read/write/ioctl transfers issued through /dev/spidevB.C
 */

#include <linux/module.h>      /* MODULE_* macros, module_init/exit */
#include <linux/kernel.h>      /* pr_info */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe */
#include <linux/spi/spi.h>     /* struct spi_device, struct spi_message */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Learner");
MODULE_DESCRIPTION("Trace spidev userspace transfers via kprobe on spidev_sync");

/*
 * spidev_sync signature (from drivers/spi/spidev.c):
 *
 *   static ssize_t spidev_sync(struct spidev_data *spidev,
 *                               struct spi_message *message);
 *
 * spidev_data is a static struct — we only need spi_message here,
 * which is a public kernel struct from <linux/spi/spi.h>.
 *
 * We read args from pt_regs:
 *   regs->di = arg0 = spidev  (struct spidev_data *)
 *   regs->si = arg1 = message (struct spi_message *)
 *
 * We cast spidev_data* to a pointer and walk its first field manually:
 * BUT since spidev_data is static/private, we only use spi_message
 * which is public and gives us actual_length + the spi device via spi_message.spi.
 */

/* pre-handler: called just BEFORE the probed function executes */
static int spidev_sync_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64 the calling convention:
     *   rdi = first arg  (spidev_data *)
     *   rsi = second arg (spi_message *)
     *
     * We grab the spi_message pointer from rsi.
     * spi_message.spi holds the pointer to the underlying spi_device,
     * giving us bus_num, chip_select, max_speed_hz, and modalias.
     */
    struct spi_message *msg = (struct spi_message *)regs->si;
    struct spi_device  *spi;
    unsigned int        total_len = 0;
    struct spi_transfer *xfer;

    /* Validate pointer before dereferencing — probe fires in kernel context */
    if (!msg)
        return 0;

    spi = msg->spi;   /* spi_message.spi is set by spi_message_init callers */

    /*
     * Walk the transfer list to compute total bytes requested.
     * spi_message.transfers is a list_head of spi_transfer structs.
     */
    list_for_each_entry(xfer, &msg->transfers, transfer_list) {
        total_len += xfer->len;
    }

    if (spi) {
        pr_info("spidev_spy: transfer on spidev%d.%d [%s] — %u bytes @ %u Hz\n",
                spi->controller->bus_num,        /* bus number, e.g. 0 */
                spi_get_chipselect(spi, 0),      /* chip select index */
                spi->modalias,                   /* driver/device name string */
                total_len,                       /* total bytes across all transfers */
                spi->max_speed_hz);              /* max clock speed in Hz */
    } else {
        /*
         * spi field not yet set (edge case during init) — print minimal info
         */
        pr_info("spidev_spy: transfer — %u bytes (spi_device not yet linked)\n",
                total_len);
    }

    return 0; /* 0 = do NOT alter execution; let spidev_sync run normally */
}

/* Define the kprobe struct targeting spidev_sync by symbol name */
static struct kprobe kp = {
    /*
     * Symbol name of the function to probe.
     * spidev_sync is static so it won't appear in /proc/kallsyms on
     * some configs — if missing, use spidev_sync_unlocked which IS
     * accessible since it's called from spidev_message() too.
     * Alternatively build with CONFIG_KALLSYMS_ALL=y.
     */
    .symbol_name = "spidev_sync",
    .pre_handler = spidev_sync_pre,
};

static int __init spidev_spy_init(void)
{
    int ret;

    /*
     * register_kprobe() patches the instruction at spidev_sync entry
     * with an INT3 (x86) so the pre_handler fires before the function runs.
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("spidev_spy: register_kprobe failed: %d\n", ret);
        pr_err("spidev_spy: try CONFIG_KALLSYMS_ALL=y or probe spidev_sync_unlocked\n");
        return ret;
    }

    pr_info("spidev_spy: kprobe planted at %p (spidev_sync)\n", kp.addr);
    return 0;
}

static void __exit spidev_spy_exit(void)
{
    /*
     * unregister_kprobe() restores the original instruction at the probe site
     * and waits for any in-flight handler to finish before returning.
     * Must be called in exit to avoid a dangling breakpoint after rmmod.
     */
    unregister_kprobe(&kp);
    pr_info("spidev_spy: kprobe removed from spidev_sync\n");
}

module_init(spidev_spy_init);
module_exit(spidev_spy_exit);
```

---

### Makefile للبناء

```makefile
obj-m += spidev_spy.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

```bash
# بناء وتحميل الـ module
make
sudo insmod spidev_spy.ko

# شوف الـ output
sudo dmesg -w | grep spidev_spy

# تفريغ الـ module
sudo rmmod spidev_spy
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|---|---|
| `<linux/kprobes.h>` | بيجيب `struct kprobe` و`register_kprobe` و`unregister_kprobe` |
| `<linux/spi/spi.h>` | بيجيب `struct spi_device` و`struct spi_message` و`struct spi_transfer` |
| `<linux/module.h>` | ضروري لأي kernel module — بيجيب `MODULE_*` macros و`module_init/exit` |

#### اختيار `spidev_sync` كـ probe target

**الـ** `spidev_sync` هي النقطة الوحيدة اللي بتمر بيها كل عمليات read وwrite وioctl message من userspace — يعني hook واحد يغطي الكل. الـ `spidev_sync_unlocked` بتتكلم منها مباشرة، لكن `spidev_sync` هي اللي بتعمل الـ locking وبتتحقق إن الـ device لسه موجود.

#### الـ `pre_handler` وقراءة الـ args

**الـ** `pt_regs` بيمثل حالة الـ registers لحظة تنفيذ الـ INT3 breakpoint. على x86-64 الـ calling convention بيحط أول argument في `rdi` وتاني argument في `rsi` — إحنا عايزين `spi_message *` اللي هو الـ `rsi`. **الـ** `spi_message` هو struct عام معرّف في `<linux/spi/spi.h>` فنقدر نوصله بدون مشاكل.

#### `list_for_each_entry` على الـ transfers

كل **spi_message** بيحتوي على list من `spi_transfer` structs — كل واحد بيمثل segment واحد من البيانات. بنلف عليهم ونجمع الـ `len` عشان نعرف إجمالي الـ bytes اللي الـ userspace طالب تتنقل.

#### `spi_get_chipselect(spi, 0)`

ده inline helper معرّف في `<linux/spi/spi.h>` بيرجع رقم الـ chip select الأول — أأمن من الوصول المباشر لـ `spi->chip_select[0]` لأنه بيعمل bounds check.

#### `module_init` / `module_exit`

- **`register_kprobe`** في الـ init بيزرع الـ breakpoint ويربط الـ handler — لو فشل (مثلاً الـ symbol مش موجود في kallsyms) بيرجع error ومبيحملش الـ module.
- **`unregister_kprobe`** في الـ exit ضروري لأنه بيشيل الـ INT3 ويستنى أي handler شغال خلص — من غيره لو عملت rmmod والـ kprobe شغالة هيحصل kernel panic أو memory corruption.

#### ليه `return 0` في الـ pre_handler؟

القيمة صفر بتقول للـ kprobe framework "خلي الـ function الأصلية تكمل عادي". لو رجعنا قيمة تانية ممكن يتخطى الـ function، وده مش اللي إحنا عايزينه هنا.

---

### مثال على الـ output

```
[  123.456] spidev_spy: kprobe planted at ffffffffc0a12340 (spidev_sync)
[  125.001] spidev_spy: transfer on spidev0.0 [ltc2488] — 4 bytes @ 1000000 Hz
[  125.890] spidev_spy: transfer on spidev0.0 [ltc2488] — 4 bytes @ 1000000 Hz
[  130.002] spidev_spy: transfer on spidev1.0 [sx1301] — 256 bytes @ 8000000 Hz
```

---

### ملاحظة عملية: لو `spidev_sync` مش في kallsyms

**الـ** `spidev_sync` دالة `static` — في بعض kernel configs ممكن تظهر في `/proc/kallsyms` بس لازم `CONFIG_KALLSYMS_ALL=y`. لو مش موجودة، غيّر الـ `symbol_name` لـ `"spidev_sync_unlocked"` اللي هي أيضاً مثيرة للاهتمام لأن `spidev_message()` بتناديها مباشرة:

```c
static struct kprobe kp = {
    .symbol_name = "spidev_sync_unlocked",  /* fallback if spidev_sync not exported to kallsyms */
    .pre_handler = spidev_sync_pre,
};
```
