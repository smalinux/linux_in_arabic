## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ SPI؟

**SPI (Serial Peripheral Interface)** هو بروتوكول اتصال تسلسلي بسيط جداً اخترعه Motorola في الثمانينيات. الفكرة أبسط من أي بروتوكول تاني: أسلاك قليلة، سرعة عالية، وتواصل مباشر بين المعالج وقطع الهاردوير.

تخيّل عندك **سبيكر بيتكلم مع مترجم**:
- الـ **Master** (المعالج): هو اللي بيتحكم في كل شيء.
- الـ **Slave/Target** (الجهاز): flash, ADC, سنسور، شاشة، إلخ.
- الأسلاك الأربعة الأساسية:
  - **MOSI** (Master Out Slave In): المعالج يبعت بيانات.
  - **MISO** (Master In Slave Out): الجهاز يرجع بيانات.
  - **SCK** (Clock): المعالج بيحدد سرعة الاتصال.
  - **CS/SS** (Chip Select): المعالج "ينادي" جهاز معين عشان يكلمه.

المميز في SPI إنه **full-duplex**: في نفس اللحظة بيبعت وبيستقبل. زي ما لو إنت بتتكلم وبتسمع في نفس الوقت.

---

### القصة: ليه محتاجين هذا الـ header؟

**المشكلة:** في لينكس في المئات من قطع الهاردوير بتشتغل بـ SPI (flash chips, OLED displays, ADCs, sensors, touch screens, وغيرها). كمان في عشرات من controllers (DesignWare, Freescale, Broadcom, Atmel, imx, إلخ). لو كل driver اشتغل بطريقته الخاصة، هيبقى عندنا فوضى تامة.

**الحل:** ملف `include/linux/spi/spi.h` هو **عقد الزواج** بين طرفين:
1. **Controller driver**: اللي يعرف كيف الهاردوير بيشغّل signals SCK, MOSI, MISO.
2. **Protocol driver**: اللي يعرف بروتوكول الجهاز المتصل (مثلاً: فلاشة NOR، شاشة ST7735، ADC ADS1220).

الـ header ده بيعرّف **الطبقة الوسطى (SPI core)** اللي بتحل المشكلة دي.

---

### تصور الطبقات

```
┌─────────────────────────────────────────────┐
│          Protocol Drivers (عملاء SPI)        │
│  (spi-nor, mmc_spi, ads7846, waveshare, ...) │
│       يستخدمون: spi_write(), spi_sync()      │
├─────────────────────────────────────────────┤
│              SPI Core (القلب)                │
│         include/linux/spi/spi.h              │
│   struct spi_message, spi_transfer,          │
│   spi_controller, spi_device                 │
│   drivers/spi/spi.c  ← التنفيذ الفعلي       │
├─────────────────────────────────────────────┤
│         Controller Drivers (هاردوير)         │
│  (spi-dw-core, spi-imx, spi-bcm2835, ...)   │
│  ينفّذون: transfer_one(), set_cs(), setup()  │
├─────────────────────────────────────────────┤
│             الهاردوير الفعلي                 │
│    (SPI Controller في SoC + الأجهزة)        │
└─────────────────────────────────────────────┘
```

---

### الأهداف الرئيسية للملف

**الـ `spi.h` بيحقق 4 أهداف:**

1. **تعريف الكائنات الأساسية** اللي بيشتغل بيها الكل:
   - `struct spi_device`: تمثيل الجهاز المتصل (فلاشة، سنسور، ...).
   - `struct spi_controller`: تمثيل الـ controller نفسه في الـ SoC.
   - `struct spi_transfer`: "طلب نقل" واحد (بيانات تُرسل أو تُستقبل).
   - `struct spi_message`: مجموعة transfers تنفَّذ بشكل atomic.

2. **تعريف الـ API** للـ protocol drivers:
   - `spi_sync()`, `spi_async()`: إرسال message.
   - `spi_write()`, `spi_read()`, `spi_w8r8()`: وظائف مساعدة.
   - `spi_setup()`: ضبط الـ mode والـ clock.

3. **تعريف الـ hooks** اللي لازم controller drivers تنفّذها:
   - `transfer_one()`, `transfer_one_message()`, `set_cs()`, `setup()`.

4. **تعريف آلية board info**:
   - `struct spi_board_info`: جدول الأجهزة المتصلة على البورد.

---

### قصة رحلة البيانات

لما driver للـ flash بيعمل `spi_write(spi, buf, len)`:

```
1. spi_write()
   └─► spi_sync_transfer()
       └─► spi_sync()
           └─► spi_async()
               └─► ctlr->transfer()         ← يضيف msg للـ queue
                   └─► pump_messages thread
                       └─► ctlr->prepare_message()
                           └─► ctlr->transfer_one_message()
                               └─► ctlr->transfer_one()  ← الـ HW فعلاً بيشتغل
                                   └─► spi_finalize_current_transfer()
                           └─► ctlr->unprepare_message()
               └─► spi_finalize_current_message()
                   └─► msg->complete()  ← callback للمتصل
```

**الـ `spi_message`** هو الوحدة الـ atomic: طول ما هي شغّالة، مش ممكن message تانية تستخدم نفس الـ bus.

---

### الـ SPI Modes: ليه مهمين؟

الـ clock بيشتغل بـ 4 modes حسب **polarity** و**phase**:

| Mode | CPOL | CPHA | الوصف |
|------|------|------|-------|
| 0    | 0    | 0    | Clock بيبدأ low، بيقرأ في leading edge |
| 1    | 0    | 1    | Clock بيبدأ low، بيقرأ في trailing edge |
| 2    | 1    | 0    | Clock بيبدأ high، بيقرأ في leading edge |
| 3    | 1    | 1    | Clock بيبدأ high، بيقرأ في trailing edge |

كل جهاز بيحدد في الـ datasheet بتاعه الـ mode المطلوب. الـ `spi_device.mode` بيحمل هذه القيمة.

---

### Multi-lane SPI: تطور البروتوكول

الـ header يدعم أشكال متقدمة:

| النوع | عدد الأسلاك للبيانات | الاستخدام |
|-------|---------------------|-----------|
| Standard SPI | 1 (MOSI/MISO) | الأجهزة العادية |
| Dual SPI | 2 | Flash أسرع |
| Quad SPI (QSPI) | 4 | Flash عالي السرعة |
| Octal SPI | 8 | Flash فائق السرعة |

الـ `tx_nbits` و`rx_nbits` في `spi_transfer` بيحددوا كام wire تُستخدم في كل transfer.

---

### ملفات المنظومة (SPI Subsystem)

**الـ maintainer** حسب `MAINTAINERS`: Mark Brown (`broonie@kernel.org`) — مسؤول عن SPI SUBSYSTEM.

#### الملفات الأساسية (Core):
| الملف | الدور |
|-------|-------|
| `include/linux/spi/spi.h` | التعريفات الأساسية (الملف ده) |
| `drivers/spi/spi.c` | تنفيذ الـ core API |
| `drivers/spi/spi-mem.c` | طبقة SPI memory |
| `drivers/spi/spi-offload.c` | offload للـ hardware |
| `include/uapi/linux/spi/spi.h` | الـ flags المشتركة مع userspace |

#### الـ Headers التكميلية:
| الملف | الدور |
|-------|-------|
| `include/linux/spi/spi-mem.h` | API لـ SPI memory (NOR Flash) |
| `include/linux/spi/spi_bitbang.h` | bit-bang SPI بالـ GPIO |
| `include/linux/spi/spi_gpio.h` | GPIO-based SPI |

#### أمثلة Controller Drivers:
| الملف | الهاردوير |
|-------|----------|
| `drivers/spi/spi-dw-core.c` | DesignWare SPI (Intel, Cavium) |
| `drivers/spi/spi-bcm2835.c` | Raspberry Pi |
| `drivers/spi/spi-imx.c` | Freescale i.MX |
| `drivers/spi/spi-atmel.c` | Atmel SAM |
| `drivers/spi/spi-bitbang.c` | bit-bang generic |

#### أمثلة Protocol Drivers (عملاء SPI):
| الملف | الجهاز |
|-------|-------|
| `drivers/mtd/spi-nor/` | NOR Flash |
| `drivers/mmc/host/mmc_spi.c` | SD Card فوق SPI |
| `drivers/input/touchscreen/ads7846.c` | touch controller |
| `drivers/net/ethernet/asix/ax88796c-spi.c` | Ethernet فوق SPI |

---

### الـ `spi_controller`: قلب التصميم

```c
struct spi_controller {
    struct device   dev;          /* device model integration */
    s16             bus_num;      /* رقم الـ bus (SPI0, SPI1, ...) */
    u16             num_chipselect; /* عدد الأجهزة المتصلة */

    /* الـ hooks اللي Controller Driver لازم ينفّذها */
    int  (*setup)(struct spi_device *spi);
    int  (*transfer_one)(struct spi_controller *ctlr,
                         struct spi_device *spi,
                         struct spi_transfer *transfer);
    void (*set_cs)(struct spi_device *spi, bool enable);

    /* message queue */
    struct list_head    queue;
    struct kthread_worker *kworker; /* thread مسؤول عن تنفيذ queue */
    /* DMA */
    struct dma_chan  *dma_tx;
    struct dma_chan  *dma_rx;
};
```

**الـ `spi_device`** يمثل جهاز واحد متصل على الـ bus:

```c
struct spi_device {
    struct device       dev;
    struct spi_controller *controller;
    u32                 max_speed_hz;  /* أقصى سرعة */
    u8                  bits_per_word; /* حجم الكلمة */
    u32                 mode;          /* SPI_MODE_0/1/2/3 + flags */
    u8    chip_select[SPI_DEVICE_CS_CNT_MAX]; /* أرقام CS */
};
```

---

### ملفات مهمة يجب معرفتها

- **`drivers/spi/spi.c`**: التنفيذ الكامل للـ core — هنا منطق الـ queue، الـ DMA mapping، الـ locking.
- **`include/linux/spi/spi-mem.h`**: امتداد للـ SPI Flash memory بيضيف مفهوم `spi_mem_op` (command + address + dummy + data).
- **`include/uapi/linux/spi/spi.h`**: الـ mode bits المشتركة مع userspace (`SPI_CPHA`, `SPI_CPOL`, `SPI_CS_HIGH`, `SPI_LSB_FIRST`, `SPI_3WIRE`, `SPI_LOOP`, `SPI_NO_CS`, إلخ).
- **`Documentation/spi/spi-summary.rst`**: الشرح الرسمي لكيفية كتابة controller وprotocol drivers.
- **`tools/spi/spidev_test.c`**: أداة اختبار لـ userspace من خلال `/dev/spidevX.Y`.
## Phase 2: شرح الـ SPI Framework

### المشكلة — ليه الـ SPI Framework موجود أصلاً؟

قبل ما يكون في framework موحد، كل driver بيتكلم مع SPI hardware كان بيعمل اللي يعجبه:
- بيكتب مباشرة على registers الـ controller
- بيدير الـ chip select بنفسه
- بيعمل busy-wait على الـ transfer
- مفيش قدر من reuse بين drivers مختلفة على نفس hardware

النتيجة؟ كل driver عنده نسخته الخاصة من نفس الكود، وأي تغيير في الـ hardware بيكسر عشرين driver. الـ Linux SPI framework اتعمل عشان يحل مشكلتين اتنين:

1. **Separation of concerns**: الـ driver اللي بيعرف بروتوكول الـ chip (زي sensor أو flash) مفروش ميعرفش إزاي الـ hardware controller بيشتغل.
2. **Reuse**: controller driver واحد (زي `spi-pl022.c` على ARM) يخدم مئات الـ peripheral drivers.

---

### الحل — النهج اللي الكرنل بياخده

الكرنل بيعمل **abstraction layer** بين:
- الـ **peripheral driver** (اللي بيعمل `spi_write`, `spi_read`) — مش محتاج يعرف أي controller موجود.
- الـ **controller driver** (اللي بيحط الـ bytes فعلاً على الـ wire) — مش محتاج يعرف أي device موصل.

التواصل بين الطرفين بيتم عبر **message queue**: الـ peripheral driver بيبني `spi_message` وبيديه للـ core، والـ core بيوصله للـ controller driver في الوقت المناسب.

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Kernel Upper Layers                       │
│         MTD / IIO / Networking / RTC / hwmon                │
└──────────────────────┬──────────────────────────────────────┘
                       │ uses
┌──────────────────────▼──────────────────────────────────────┐
│               SPI Protocol (Peripheral) Drivers             │
│    spi-nor.c   m25p80.c   mcp3208.c   spidev.c   enc28j60  │
│                                                             │
│    بيستخدموا:  spi_write()  spi_read()  spi_sync()         │
│                spi_async()  spi_message / spi_transfer      │
└──────────────────────┬──────────────────────────────────────┘
                       │ submits spi_message
┌──────────────────────▼──────────────────────────────────────┐
│                   SPI Core (spi.c)                          │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              spi_bus_type                           │   │
│  │   match() → probe() → driver binding               │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────┐    ┌────────────────────────────┐    │
│  │  Message Queue   │    │   Transfer Validation      │    │
│  │  (kthread pump)  │    │   (speed/bpw/mode checks)  │    │
│  └──────────────────┘    └────────────────────────────┘    │
│                                                             │
│  ┌──────────────────┐    ┌────────────────────────────┐    │
│  │  DMA Mapping     │    │   CS GPIO Management       │    │
│  │  (dmaengine)     │    │   (gpio/consumer API)      │    │
│  └──────────────────┘    └────────────────────────────┘    │
└──────────────────────┬──────────────────────────────────────┘
                       │ calls controller hooks
┌──────────────────────▼──────────────────────────────────────┐
│             SPI Controller (Host) Drivers                   │
│                                                             │
│   spi-pl022.c    spi-imx.c    spi-rockchip.c    spi-bcm2835│
│                                                             │
│   بيimplementoا:  transfer_one()  set_cs()  setup()        │
│                   prepare_transfer_hardware()               │
└──────────────────────┬──────────────────────────────────────┘
                       │ writes registers / triggers DMA
┌──────────────────────▼──────────────────────────────────────┐
│              SPI Hardware (SoC controller)                  │
│                                                             │
│   ┌──────┐   MOSI  ┌────────┐   MOSI  ┌──────────────┐    │
│   │ CPU  │─────────│SPI Ctrl│─────────│  SPI Device  │    │
│   │      │   MISO  │        │   MISO  │  (Flash/ADC) │    │
│   │      │─────────│        │─────────│              │    │
│   │      │   SCK   │        │   SCK   │              │    │
│   │      │─────────│        │─────────│              │    │
│   │      │   CS    │        │   CS    │              │    │
│   └──────┘─────────└────────┘─────────└──────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

---

### التشبيه الحقيقي — المطعم والكواتر

تخيل مطعم كبير:

| عنصر المطعم | المقابل في الـ SPI Framework |
|---|---|
| **الزبون** (بيطلب أكل) | الـ **upper layer** (MTD, IIO, networking) |
| **الـ waiter** (بيكتب الأوردر) | الـ **peripheral/protocol driver** |
| **الـ order slip** (ورقة الطلب) | الـ **`spi_message`** |
| **كل item في الأوردر** | الـ **`spi_transfer`** |
| **الـ kitchen queue** | الـ **message queue** في `spi_controller` |
| **رئيس الطهاة** (بيوزع الشغل) | الـ **SPI core** (spi.c) |
| **الطباخ المتخصص** (بيعرف الفرن) | الـ **controller driver** |
| **الفرن/الغاز** (الـ hardware) | الـ **SPI hardware registers** |

الزبون (upper layer) مش محتاج يعرف الطباخ شغال على غاز ولا كهرباء — بس بيقول "عايز كذا". الـ waiter (protocol driver) بيحول الطلب لـ order slip موحد (spi_message). الـ core هو اللي بياخد الـ slip ويوديه للمطبخ الصح. الطباخ (controller driver) هو بس اللي يعرف إزاي الفرن الـ hardware بيشتغل.

**النقطة العميقة**: الـ order slip (spi_message) موجود في memory الـ waiter (protocol driver) طول فترة التنفيذ — الـ framework مبيعملش copy، بس بيضمن إن الـ waiter مش يعدّل فيه لحد ما يجيه `complete()` callback.

---

### الـ Core Abstractions — الهياكل المحورية

#### 1. `struct spi_controller` — قلب الـ framework

```c
struct spi_controller {
    struct device   dev;          /* device model node — ده هو اللي بيظهر في sysfs */
    s16             bus_num;      /* رقم الـ bus (SPI0, SPI1, ...) */
    u16             num_chipselect;
    u32             mode_bits;    /* الـ modes اللي الـ controller يدعمها */
    u32             bits_per_word_mask;
    u32             min_speed_hz;
    u32             max_speed_hz;

    /* === الـ hooks اللي الـ controller driver بيimplementها === */
    int  (*setup)(struct spi_device *spi);          /* configure device */
    int  (*transfer)(struct spi_device *, struct spi_message *); /* low-level: add to queue */
    void (*set_cs)(struct spi_device *spi, bool enable);
    int  (*transfer_one)(struct spi_controller *, struct spi_device *, struct spi_transfer *);
    int  (*transfer_one_message)(struct spi_controller *, struct spi_message *);
    int  (*prepare_transfer_hardware)(struct spi_controller *);
    int  (*unprepare_transfer_hardware)(struct spi_controller *);

    /* === الـ message pump (managed by core) === */
    bool            queued;
    struct kthread_worker  *kworker;   /* dedicated kernel thread */
    struct kthread_work     pump_messages;
    struct list_head        queue;     /* pending messages */
    struct spi_message     *cur_msg;   /* currently executing message */

    /* === DMA support === */
    struct dma_chan *dma_tx;
    struct dma_chan *dma_rx;
    bool (*can_dma)(struct spi_controller *, struct spi_device *, struct spi_transfer *);
};
```

**الـ kthread worker**: الـ core بيشغّل thread منفصل (الـ "pump") عشان يـprocess الـ messages. الـ protocol driver بيعمل `spi_async()` من أي context، والـ pump thread هو اللي بيكلم الـ controller فعلياً. ده بيحل مشكلة إن الـ controller driver محتاج `can sleep`.

#### 2. `struct spi_device` — الـ proxy للـ peripheral

```c
struct spi_device {
    struct device        dev;          /* embedded device — بيتبع الـ spi_bus_type */
    struct spi_controller *controller; /* مين اللي بيتحكم فيه */
    u32                  max_speed_hz;
    u8                   bits_per_word;
    u32                  mode;         /* SPI_MODE_0/1/2/3, SPI_CS_HIGH, ... */
    int                  irq;
    void                *controller_state; /* private data للـ controller driver */
    char                 modalias[SPI_NAME_SIZE]; /* اسم الـ driver المطلوب */

    /* CS delays */
    struct spi_delay     cs_setup;
    struct spi_delay     cs_hold;
    struct spi_delay     cs_inactive;

    /* Multi-CS support */
    u8                   chip_select[SPI_DEVICE_CS_CNT_MAX];
    struct gpio_desc    *cs_gpiod[SPI_DEVICE_CS_CNT_MAX];
};
```

**`controller_state`**: ده pointer مهم. الـ controller driver بياخد memory إضافية عند إنشاء الـ `spi_controller` (عبر `spi_alloc_host(dev, sizeof(struct my_priv))`). لما بيحتاج يحفظ state خاص بكل device (زي pre-computed register values)، بيستخدم `controller_state` في الـ `spi_device`.

#### 3. `struct spi_transfer` — وحدة البيانات الأساسية

```c
struct spi_transfer {
    const void   *tx_buf;      /* NULL = أرسل zeros */
    void         *rx_buf;      /* NULL = throw away received data */
    unsigned      len;         /* بالـ bytes */
    u32           speed_hz;    /* 0 = استخدم الـ default من spi_device */
    u8            bits_per_word; /* 0 = استخدم الـ default */
    bool          cs_change;   /* deassert CS بعد الـ transfer ده؟ */

    /* DMA (managed by core, not protocol driver) */
    struct sg_table tx_sg;
    struct sg_table rx_sg;
    dma_addr_t    tx_dma;
    dma_addr_t    rx_dma;

    /* Multi-lane SPI (DSPI/QSPI) */
    unsigned      tx_nbits:4;  /* SPI_NBITS_SINGLE/DUAL/QUAD/OCTAL */
    unsigned      rx_nbits:4;

    /* Timing */
    struct spi_delay  delay;
    struct spi_delay  cs_change_delay;
    struct spi_delay  word_delay;

    struct list_head  transfer_list; /* linked into spi_message.transfers */
};
```

**النقطة**: الـ SPI hardware دايماً full-duplex — بيبعت وبيستقبل في نفس الوقت. لو `tx_buf = NULL`، الـ core بيبعت zeros. لو `rx_buf = NULL`، بيـdiscard البيانات الجايه. في الحالتين الـ hardware شغال.

#### 4. `struct spi_message` — الـ atomic transaction

```c
struct spi_message {
    struct list_head  transfers;   /* list of spi_transfer */
    struct spi_device *spi;        /* الـ target device */

    int    status;                 /* 0 = success, negative errno = error */
    void (*complete)(void *context); /* callback لما الـ message يخلص */
    void  *context;                /* argument للـ callback */

    unsigned frame_length;         /* total bytes */
    unsigned actual_length;        /* bytes actually transferred */

    bool pre_optimized;
    bool optimized;
    bool prepared;

    struct list_head queue;        /* لما تكون في controller's queue */
    void *state;                   /* controller's private per-message state */
    void *opt_state;               /* state for optimized messages */
};
```

**الـ atomicity**: الـ message بالنسبة للـ SPI bus هو atomic — طول ما بيتـexecute، مفيش message تانية تـaccess نفس الـ bus (إلا لو `spi_bus_lock`). الـ CS بتفضل active طول الـ message.

---

### الـ Struct Relationships — الرسم التفصيلي

```
spi_controller
│
├── dev (struct device)
│     └── bus_type = spi_bus_type
│
├── queue (list_head) ─────────────────────────┐
│                                              │
├── cur_msg ──────────► spi_message            │
│                       │                     │
│                       ├── spi ──────────► spi_device
│                       │                  │
│                       │                  ├── controller ──► (back to spi_controller)
│                       │                  ├── mode, speed_hz, bits_per_word
│                       │                  ├── cs_gpiod[]
│                       │                  └── controller_state (driver private)
│                       │
│                       ├── transfers (list) ─► spi_transfer [0]
│                       │                          ├── tx_buf, rx_buf, len
│                       │                          ├── tx_sg, rx_sg (DMA)
│                       │                          ├── speed_hz, bits_per_word
│                       │                          └── transfer_list ──► spi_transfer [1]
│                       │                                                 └── ...
│                       ├── complete() callback
│                       ├── status
│                       └── resources (list) ─► spi_res items
│
├── kworker (pump thread)
├── pump_messages (kthread_work)
│
├── /* controller hooks */
├── transfer_one()
├── set_cs()
├── prepare_transfer_hardware()
└── can_dma()
```

---

### طبقات الـ controller driver hooks — مين بيعمل إيه؟

الـ core بيوفر **مستويات مختلفة** من الـ integration للـ controller driver، من الأبسط للأكثر تحكماً:

#### المستوى الأول (الأبسط): `transfer_one`

الـ controller driver بيimplements فقط `transfer_one()` + `set_cs()`. الـ core بيتولى:
- إدارة الـ queue كاملة
- ترتيب الـ transfers داخل الـ message
- الـ CS timing والـ delays
- الـ DMA mapping لو `can_dma()` رجعت true

```c
/* مثال: controller driver بسيط */
static int my_spi_transfer_one(struct spi_controller *ctlr,
                               struct spi_device *spi,
                               struct spi_transfer *xfer)
{
    /* ابعت xfer->len bytes من xfer->tx_buf, استقبل في xfer->rx_buf */
    /* return 0 لو خلص synchronously */
    /* return 1 لو async — هتكمل بـ spi_finalize_current_transfer() */
}
```

#### المستوى التاني: `transfer_one_message`

الـ controller بياخد الـ message كاملة وبيتحكم في كل حاجة. مفيد لو الـ hardware عنده FIFO كبير أو DMA scatter-gather.

#### المستوى التالت (legacy): `transfer`

القديم — بيضيف الـ message للـ queue مباشرة. الـ core مش بيساعد بأي حاجة. بيتُستخدم في drivers قديمة.

---

### ما الـ framework بيمتلكه وما بيفوّضه

| المسؤولية | الـ SPI Core (Framework) | الـ Controller Driver | الـ Protocol Driver |
|---|---|---|---|
| إدارة الـ message queue | **يمتلك** | | |
| الـ kthread pump | **يمتلك** | | |
| DMA mapping | **يمتلك** (لو `can_dma`) | يحدد الـ capability | |
| CS GPIO management | **يمتلك** | يعطي `cs_gpiods[]` | |
| Transfer splitting (maxsize) | **يمتلك** | يحدد `max_transfer_size` | |
| Bus locking | **يمتلك** | | |
| Device registration/matching | **يمتلك** | | |
| Writing HW registers | | **يمتلك** | |
| CS timing (hardware) | يحسب | **يمتلك** | |
| Clock/mode setup | | **يمتلك** | |
| بناء الـ message | | | **يمتلك** |
| تحديد الـ protocol sequence | | | **يمتلك** |
| Retry logic | | | **يمتلك** |
| Error interpretation | | | **يمتلك** |

---

### الـ `spi_bus_type` والـ Device Model

> **prerequisite**: الـ Linux Device Model — كل subsystem بيعرّف `bus_type` يربط drivers بـ devices.

الـ SPI framework بيعرّف:

```c
extern const struct bus_type spi_bus_type;
```

ده بيعني:
- كل `spi_device` بيتسجل على `spi_bus_type`
- كل `spi_driver` بيتسجل على `spi_bus_type`
- الـ kernel بيعمل `match()` تلقائياً بين الاتنين عبر `modalias`/`of_match_table`

لما الـ controller driver بيعمل `spi_register_controller()`:
1. الـ core بيقرأ الـ device tree تحت نود الـ controller
2. بيبني `spi_device` لكل child node
3. بيسجلهم على الـ `spi_bus_type`
4. الـ bus بيعمل `match()` مع الـ protocol drivers المسجلة
5. لو match تم، بيكلم `probe()` في الـ `spi_driver`

---

### الـ Message Optimization — الـ Fast Path

الـ framework فيه optimization مهم: `spi_optimize_message()`.

لما protocol driver عنده message بيبعتها repetitively (زي ADC بيقرأ كل 1ms):

```c
/* Protocol driver بيعمل optimize مرة واحدة */
spi_optimize_message(spi, &msg);

/* Loop */
while (1) {
    spi_sync(spi, &msg);  /* fast path — skip re-validation */
    process_data(msg.transfers[0].rx_buf);
}

spi_unoptimize_message(&msg);
```

الـ core بيعمل:
1. Validate الـ message مرة واحدة
2. يعمل DMA mapping مرة واحدة
3. الـ controller driver بيحفظ الـ hardware state في `opt_state`

في الـ hot path، الـ `spi_sync()` بتـcheck `queue_empty` — لو الـ queue فاضية وفيه context مناسب، بتـexecute الـ message مباشرة بدون الـ kthread (الـ "opportunistic sync" path).

---

### الـ Statistics والـ Observability

```c
struct spi_statistics {
    struct u64_stats_sync syncp;  /* seqcount للـ per-cpu safety على 32-bit */

    u64_stats_t messages;
    u64_stats_t transfers;
    u64_stats_t errors;
    u64_stats_t timedout;
    u64_stats_t spi_sync;
    u64_stats_t spi_sync_immediate;  /* كام مرة الـ fast path اتنفذ */
    u64_stats_t spi_async;
    u64_stats_t bytes;
    u64_stats_t bytes_tx;
    u64_stats_t bytes_rx;

    u64_stats_t transfer_bytes_histo[17]; /* histogram للـ transfer sizes */
    u64_stats_t transfers_split_maxsize;  /* كام transfer اتقسمت */
};
```

الـ stats موجودة على مستويين: الـ `spi_controller` وكل `spi_device`. بتظهر في `/sys/class/spi_master/spiN/statistics/` وفي `/sys/bus/spi/devices/spiN.M/statistics/`.

الـ `u64_stats_sync` + `__percpu` pattern ده مهم: الـ stats بيتحدث per-CPU عشان يتجنب contention، والـ `syncp` (seqcount) بيضمن consistent read على 32-bit systems اللي 64-bit atomic مش guaranteed فيها.

---

### الـ SPI Modes — التفاصيل الحقيقية

| Mode | CPOL | CPHA | الـ Clock idle | Sample edge |
|---|---|---|---|---|
| SPI_MODE_0 | 0 | 0 | Low | Rising |
| SPI_MODE_1 | 0 | 1 | Low | Falling |
| SPI_MODE_2 | 1 | 0 | High | Falling |
| SPI_MODE_3 | 1 | 1 | High | Rising |

الـ `mode` field في `spi_device` بيحمل أكتر من CPOL/CPHA:

```
Bit 31..29: kernel-internal flags (SPI_NO_TX, SPI_NO_RX, SPI_TPM_HW_FLOW)
Bit 28..0:  user-visible flags (SPI_CS_HIGH, SPI_LSB_FIRST, SPI_3WIRE, ...)
```

الـ `SPI_MODE_KERNEL_MASK` و `SPI_MODE_USER_MASK` مش بياخدوش overlap — الـ `static_assert` في الـ header بيضمن ده وقت الـ compilation:

```c
static_assert((SPI_MODE_KERNEL_MASK & SPI_MODE_USER_MASK) == 0,
              "SPI_MODE_USER_MASK & SPI_MODE_KERNEL_MASK must not overlap");
```

---

### الـ Multi-Lane SPI (DSPI / QSPI / OSPI)

الـ framework بيدعم أكتر من wire واحدة للـ data:

```c
/* في spi_transfer */
unsigned tx_nbits:4;   /* SPI_NBITS_SINGLE=1, DUAL=2, QUAD=4, OCTAL=8 */
unsigned rx_nbits:4;
unsigned multi_lane_mode: 2; /* SINGLE, STRIPE, MIRROR */

/* في spi_device */
u8 tx_lane_map[SPI_DEVICE_DATA_LANE_CNT_MAX]; /* peripheral lane → controller lane */
u8 rx_lane_map[SPI_DEVICE_DATA_LANE_CNT_MAX];
```

**STRIPE mode**: كل word بيتبعت على lane منفصلة — throughput أعلى.
**MIRROR mode**: نفس الـ word على كل الـ lanes — للـ write-only devices.

ده مهم جداً للـ NOR flash على embedded: `spi-nor` framework فوق SPI بيستخدم QUAD mode عشان يوصل لـ 4x throughput.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options

#### SPI Mode Flags — `spi_device.mode` (u32)

| Flag | Bit | المعنى |
|------|-----|--------|
| `SPI_CPHA` | 0 | clock phase — بيتحدد امتى بيتقرأ/بيتكتب الداتا |
| `SPI_CPOL` | 1 | clock polarity — البداية high ولا low |
| `SPI_MODE_0` | CPOL=0, CPHA=0 | الـ mode الافتراضي (MicroWire) |
| `SPI_MODE_1` | CPOL=0, CPHA=1 | |
| `SPI_MODE_2` | CPOL=1, CPHA=0 | |
| `SPI_MODE_3` | CPOL=1, CPHA=1 | |
| `SPI_CS_HIGH` | 2 | الـ chip select بيكون active high |
| `SPI_LSB_FIRST` | 3 | بيبعت الـ LSB الأول |
| `SPI_3WIRE` | 4 | MOSI و MISO على نفس الـ pin |
| `SPI_LOOP` | 5 | loopback mode للاختبار |
| `SPI_NO_CS` | 6 | جهاز واحد على الـ bus، مفيش chip select |
| `SPI_READY` | 7 | الـ slave بيشد الـ line low للـ pause |
| `SPI_TX_DUAL` | 8 | إرسال بـ 2 wires |
| `SPI_TX_QUAD` | 9 | إرسال بـ 4 wires |
| `SPI_RX_DUAL` | 10 | استقبال بـ 2 wires |
| `SPI_RX_QUAD` | 11 | استقبال بـ 4 wires |
| `SPI_CS_WORD` | 12 | toggle الـ CS بعد كل word |
| `SPI_TX_OCTAL` | 13 | إرسال بـ 8 wires |
| `SPI_RX_OCTAL` | 14 | استقبال بـ 8 wires |
| `SPI_3WIRE_HIZ` | 15 | high-Z turnaround على الـ 3-wire |
| `SPI_RX_CPHA_FLIP` | 16 | عكس الـ CPHA على الـ RX فقط |
| `SPI_MOSI_IDLE_LOW` | 17 | MOSI يفضل low لما مفيش نقل |
| `SPI_MOSI_IDLE_HIGH` | 18 | MOSI يفضل high لما مفيش نقل |
| `SPI_NO_TX` | 31 | مفيش wire للإرسال (kernel-only) |
| `SPI_NO_RX` | 30 | مفيش wire للاستقبال (kernel-only) |
| `SPI_TPM_HW_FLOW` | 29 | TPM hardware flow control (kernel-only) |
| `SPI_MODE_USER_MASK` | bits 0–18 | الـ bits المتاحة لـ userspace |
| `SPI_MODE_KERNEL_MASK` | bits 29–31 | الـ bits المحجوزة للـ kernel |

#### SPI Controller Flags — `spi_controller.flags` (u16)

| Flag | Bit | المعنى |
|------|-----|--------|
| `SPI_CONTROLLER_HALF_DUPLEX` | 0 | مش قادر يعمل full duplex |
| `SPI_CONTROLLER_NO_RX` | 1 | مش قادر يقرأ |
| `SPI_CONTROLLER_NO_TX` | 2 | مش قادر يكتب |
| `SPI_CONTROLLER_MUST_RX` | 3 | لازم يكون في RX buffer دايمًا |
| `SPI_CONTROLLER_MUST_TX` | 4 | لازم يكون في TX buffer دايمًا |
| `SPI_CONTROLLER_GPIO_SS` | 5 | الـ CS بيتعمل عن طريق GPIO |
| `SPI_CONTROLLER_SUSPENDED` | 6 | الـ controller في حالة suspend |
| `SPI_CONTROLLER_MULTI_CS` | 7 | يقدر يـ assert أكتر من CS في نفس الوقت |

#### Transfer Error Flags — `spi_transfer.error` (u16)

| Flag | Bit | المعنى |
|------|-----|--------|
| `SPI_TRANS_FAIL_NO_START` | 0 | الـ transfer متبدأش خالص |
| `SPI_TRANS_FAIL_IO` | 1 | فشل I/O أثناء النقل |

#### Lane Width Constants — `spi_transfer.tx_nbits` / `rx_nbits`

| Constant | القيمة | المعنى |
|----------|--------|--------|
| `SPI_NBITS_SINGLE` | 0x01 | 1-bit (standard MOSI/MISO) |
| `SPI_NBITS_DUAL` | 0x02 | 2-bit (DUAL SPI) |
| `SPI_NBITS_QUAD` | 0x04 | 4-bit (QSPI) |
| `SPI_NBITS_OCTAL` | 0x08 | 8-bit (Octal SPI) |

#### Multi-Lane Mode Constants — `spi_transfer.multi_lane_mode`

| Constant | القيمة | المعنى |
|----------|--------|--------|
| `SPI_MULTI_LANE_MODE_SINGLE` | 0 | lane واحدة بس |
| `SPI_MULTI_LANE_MODE_STRIPE` | 1 | كل word بيتوزع على الـ lanes |
| `SPI_MULTI_LANE_MODE_MIRROR` | 2 | نفس الـ word بيتبعت على كل الـ lanes |

#### Delay Unit Constants — `spi_delay.unit`

| Constant | القيمة | المعنى |
|----------|--------|--------|
| `SPI_DELAY_UNIT_USECS` | 0 | microseconds |
| `SPI_DELAY_UNIT_NSECS` | 1 | nanoseconds |
| `SPI_DELAY_UNIT_SCK` | 2 | عدد clock cycles |

#### Helper Macros

| Macro | الاستخدام |
|-------|-----------|
| `SPI_BPW_MASK(bits)` | بيعمل mask لـ bits_per_word واحدة |
| `SPI_BPW_RANGE_MASK(min, max)` | range من min لـ max |
| `SPI_STATISTICS_ADD_TO_FIELD` | بيضيف count للـ per-CPU stats |
| `SPI_STATISTICS_INCREMENT_FIELD` | بيزود الـ counter بواحد |

---

### الـ Structs المهمة

#### 1. `struct spi_delay`

الـ **delay** بيتحدد بـ value ووحدة قياس. بيتستخدم في كذا حاجة: بين الـ words، قبل/بعد الـ CS، وبين الـ transfers.

```c
struct spi_delay {
    u16  value;  // القيمة العددية
    u8   unit;   // الوحدة: USECS / NSECS / SCK
};
```

**الاستخدام:** بيتضمَّن جوه `spi_device` (cs_setup, cs_hold, cs_inactive, word_delay) وجوه `spi_transfer` (delay, cs_change_delay, word_delay).

---

#### 2. `struct spi_statistics`

**إحصائيات** per-CPU للـ messages والـ transfers والـ bytes. موجود في كل من `spi_device` و`spi_controller`.

| Field | المعنى |
|-------|--------|
| `syncp` | seqcount لحماية الـ 64-bit counters على 32-bit CPUs |
| `messages` | عدد الـ spi_message اللي اتعالجت |
| `transfers` | عدد الـ spi_transfer |
| `errors` | عدد الأخطاء |
| `timedout` | عدد الـ timeouts |
| `spi_sync` | عدد calls لـ spi_sync |
| `spi_sync_immediate` | عدد المرات اللي اتنفذت فيها spi_sync فورًا |
| `spi_async` | عدد calls لـ spi_async |
| `bytes` / `bytes_tx` / `bytes_rx` | إجمالي الـ bytes |
| `transfer_bytes_histo[17]` | histogram لأحجام الـ transfers |
| `transfers_split_maxsize` | عدد الـ transfers اللي اتقسمت بسبب الـ maxsize |

---

#### 3. `struct spi_device`

الـ **proxy** اللي بيمثل جهاز SPI من ناحية الـ controller. كل جهاز فيزيائي (chip) بيكون ليه `spi_device` واحد.

| Field | النوع | المعنى |
|-------|-------|--------|
| `dev` | `struct device` | الـ device model embedding |
| `controller` | `*spi_controller` | الـ controller اللي بيتكلم معاه |
| `max_speed_hz` | u32 | أقصى سرعة للـ clock |
| `bits_per_word` | u8 | حجم الـ word الافتراضي (0 = 8 bits) |
| `mode` | u32 | SPI mode flags (CPOL, CPHA, إلخ) |
| `irq` | int | رقم الـ IRQ لو موجود |
| `controller_state` | `void*` | private state للـ controller driver |
| `controller_data` | `void*` | board-specific hints |
| `modalias[32]` | char | اسم الـ driver المطلوب |
| `driver_override` | `const char*` | إجبار binding لـ driver معين |
| `pcpu_statistics` | `__percpu *` | إحصائيات per-CPU |
| `word_delay` | `spi_delay` | تأخير بين الـ words |
| `cs_setup/hold/inactive` | `spi_delay` | تأخيرات الـ chip select |
| `chip_select[4]` | u8[] | الـ physical CS numbers |
| `num_chipselect` | u8 | عدد الـ CS المستخدمة |
| `cs_index_mask` | u32:4 | bitmask للـ CS النشطة |
| `cs_gpiod[4]` | `*gpio_desc[]` | GPIO descriptors للـ CS |
| `tx_lane_map[8]` | u8[] | mapping الـ TX lanes |
| `rx_lane_map[8]` | u8[] | mapping الـ RX lanes |
| `num_tx_lanes` / `num_rx_lanes` | u8 | عدد الـ data lanes |

---

#### 4. `struct spi_driver`

الـ **protocol driver** — الـ driver اللي بيكلم الجهاز عن طريق SPI messages.

| Field | المعنى |
|-------|--------|
| `id_table` | قائمة الأجهزة المدعومة |
| `probe` | بيتاستدعى لما بيتعمل bind |
| `remove` | بيتاستدعى لما بيتعمل unbind |
| `shutdown` | عند الـ system shutdown |
| `driver` | embedded `device_driver` |

---

#### 5. `struct spi_controller`

الـ struct الأضخم — بيمثل الـ SPI host controller (الـ IP block في الـ SoC). بيتحكم في الـ bus وبيدير queue الـ messages.

**القسم الأول — معلومات ثابتة:**

| Field | المعنى |
|-------|--------|
| `dev` | device model |
| `list` | في قائمة كل الـ controllers |
| `bus_num` | رقم الـ SPI bus (مثلاً SPI-2) |
| `num_chipselect` | عدد الـ CS المتاحة |
| `num_data_lanes` | عدد الـ data lanes |
| `dma_alignment` | محاذاة الـ DMA buffers |
| `mode_bits` | الـ mode flags اللي الـ controller بيفهمها |
| `bits_per_word_mask` | الـ word sizes المدعومة |
| `min/max_speed_hz` | نطاق السرعة |
| `flags` | قيود الـ controller |

**القسم الثاني — الـ callbacks:**

| Callback | المعنى |
|----------|--------|
| `setup` | إعداد mode وسرعة الجهاز |
| `set_cs_timing` | ضبط تأخيرات الـ CS |
| `transfer` | إضافة message للـ queue (legacy) |
| `cleanup` | تحرير resources الجهاز |
| `can_dma` | هل الـ transfer دي تقدر تتعمل بـ DMA؟ |
| `optimize_message` | تحسين الـ message للـ reuse |
| `prepare_transfer_hardware` | إعداد الـ hardware قبل النقل |
| `transfer_one_message` | نقل message كاملة (بديل لـ transfer_one) |
| `unprepare_transfer_hardware` | تحرير الـ hardware بعد النقل |
| `prepare_message` | إعداد message واحدة (DMA mapping مثلاً) |
| `unprepare_message` | عكس prepare_message |
| `set_cs` | تشغيل/إيقاف الـ chip select |
| `transfer_one` | نقل spi_transfer واحدة |
| `handle_err` | معالجة الأخطاء |
| `target_abort` | إيقاف النقل الجاري (target mode) |
| `get_offload` / `put_offload` | إدارة الـ offload instance |
| `fw_translate_cs` | ترجمة CS numbers من الـ firmware |

**القسم الثالث — الـ queue internals:**

| Field | المعنى |
|-------|--------|
| `queued` | هل الـ controller بيستخدم الـ generic queue؟ |
| `kworker` | الـ kthread اللي بيشغل الـ pump |
| `pump_messages` | kthread_work للـ scheduling |
| `queue_lock` | spinlock لحماية الـ queue |
| `queue` | قائمة الـ messages المنتظرة |
| `cur_msg` | الـ message الجاري تنفيذها |
| `cur_msg_completion` | completion للـ message الحالية |
| `busy/running/rt` | حالة الـ pump thread |
| `xfer_completion` | completion للـ transfer الحالية |
| `last_cs[4]` / `last_cs_index_mask` | آخر CS استُخدم |
| `queue_empty` | علامة إن الـ queue فاضية (fast path) |
| `must_async` | إيقاف كل الـ fast paths |
| `defer_optimize_message` | تأجيل الـ optimization |

---

#### 6. `struct spi_transfer`

**وحدة النقل الأساسية** — بيمثل read/write واحد على الـ bus. الـ message مكونة من سلسلة منهم.

| Field | المعنى |
|-------|--------|
| `tx_buf` / `rx_buf` | مؤشرات الـ data (DMA-safe) |
| `tx_dma` / `rx_dma` | عناوين الـ DMA (internal) |
| `tx_sg` / `rx_sg` | scatterlist للـ DMA |
| `tx_sg_mapped` / `rx_sg_mapped` | هل اتعمل DMA mapping؟ |
| `len` | حجم الـ data بالـ bytes |
| `speed_hz` | سرعة مخصوصة لهذا الـ transfer |
| `bits_per_word` | حجم الـ word لهذا الـ transfer |
| `dummy_data` | الـ transfer ده dummy (لا داتا حقيقية) |
| `cs_off` | نفّذ الـ transfer مع CS مطفي |
| `cs_change` | toggle الـ CS بعد الـ transfer |
| `tx_nbits` / `rx_nbits` | عدد الـ lanes للإرسال/الاستقبال |
| `multi_lane_mode` | طريقة توزيع الداتا على الـ lanes |
| `dtr_mode` | Double Transfer Rate |
| `delay` | تأخير بعد الـ transfer |
| `cs_change_delay` | تأخير بعد toggle الـ CS |
| `word_delay` | تأخير بين الـ words |
| `effective_speed_hz` | السرعة الفعلية اللي اتنفذت |
| `error` | error flags بعد التنفيذ |
| `offload_flags` | flags للـ SPI offload |
| `ptp_sts_word_pre/post` | word offsets للـ PTP timestamping |
| `ptp_sts` | مؤشر لـ PTP timestamp struct |
| `timestamped` | هل اتعمل timestamp؟ |
| `transfer_list` | الـ list_head اللي بيربطه بالـ message |

---

#### 7. `struct spi_message`

الـ **transaction الذري** — مجموعة من الـ transfers بتتنفذ مع بعض على الـ bus بدون انقطاع.

| Field | المعنى |
|-------|--------|
| `transfers` | list_head لقائمة الـ spi_transfers |
| `spi` | الجهاز المقصود |
| `pre_optimized` | الـ driver اللي فوق عمل optimize |
| `optimized` | الـ core عمل optimize |
| `prepared` | اتعمل prepare_message |
| `status` | 0 = success, negative = error |
| `complete` | callback لما تخلص |
| `context` | argument للـ complete callback |
| `frame_length` | إجمالي الـ bytes في الـ message |
| `actual_length` | الـ bytes اللي اتنقلت فعلاً |
| `queue` | list entry في قائمة الـ controller |
| `state` | private state للـ controller driver |
| `opt_state` | state الخاص بالـ optimize phase |
| `offload` | offload instance (اختياري) |
| `resources` | list of spi_res للـ resource management |

---

#### 8. `struct spi_board_info`

**قالب** لتسجيل جهاز SPI من board init code أو device tree. بيتحول لـ `spi_device` وقت التسجيل.

| Field | المعنى |
|-------|--------|
| `modalias[32]` | اسم الـ driver |
| `platform_data` | بيانات خاصة بالـ protocol driver |
| `swnode` | software node (device properties) |
| `controller_data` | hints للـ controller |
| `irq` | رقم الـ IRQ |
| `max_speed_hz` | أقصى سرعة |
| `bus_num` | رقم الـ SPI bus |
| `chip_select` | رقم الـ CS |
| `mode` | SPI mode flags |

---

#### 9. `struct spi_res`

**resource management** أثناء معالجة الـ message. مستوحى من devres لكنه مربوط بـ lifecycle الـ message مش الـ device.

```c
struct spi_res {
    struct list_head   entry;    // ربطه بـ message->resources
    spi_res_release_t  release;  // callback التحرير
    unsigned long long data[];   // extra data مضمونة التوافق
};
```

---

#### 10. `struct spi_replaced_transfers`

بيُستخدم لما الـ core بيحتاج يستبدل بعض الـ transfers في message (مثلاً عند الـ splitting) ويقدر يرجعهم بعدين.

| Field | المعنى |
|-------|--------|
| `release` | callback التحرير |
| `extradata` | بيانات إضافية |
| `replaced_transfers` | الـ transfers الأصلية اللي اتشالت |
| `replaced_after` | بعد أي transfer بيتحطوا |
| `inserted` | عدد الـ transfers الجديدة |
| `inserted_transfers[]` | الـ transfers الجديدة (flexible array) |

---

### رسائل العلاقات بين الـ Structs

```
                    ┌──────────────────────────────────────────────┐
                    │            spi_board_info                    │
                    │  (template — board init code)                │
                    └───────────────┬──────────────────────────────┘
                                    │ spi_new_device() / spi_add_device()
                                    ▼
┌──────────────────────────────────────────────────────────────────┐
│                        spi_controller                            │
│  dev ──► struct device                                           │
│  list ──► global spi list                                        │
│  queue ──► [spi_message] ──► [spi_message] ──► ...              │
│  cur_msg ──► spi_message (currently executing)                   │
│  cs_gpiods[] ──► gpio_desc[]                                     │
│  dma_tx/dma_rx ──► dma_chan                                      │
│  pcpu_statistics ──► spi_statistics (per-CPU)                    │
│  mem_ops ──► spi_controller_mem_ops                              │
│  mem_caps ──► spi_controller_mem_caps                            │
│  kworker ──► kthread_worker (message pump)                       │
└──────────────────────┬───────────────────────────────────────────┘
                       │ بيملك N جهاز
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                         spi_device                               │
│  dev ──► struct device                                           │
│  controller ──► spi_controller (parent)                          │
│  cs_gpiod[] ──► gpio_desc[]                                      │
│  pcpu_statistics ──► spi_statistics (per-CPU)                    │
│  word_delay / cs_setup / cs_hold / cs_inactive ──► spi_delay    │
└──────────────────────┬───────────────────────────────────────────┘
                       │ بيبعت إليه
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                         spi_message                              │
│  spi ──► spi_device                                              │
│  transfers ──► [spi_transfer] ──► [spi_transfer] ──► ...        │
│  resources ──► [spi_res] ──► [spi_res] ──► ...                  │
│  offload ──► spi_offload (optional)                              │
│  complete ──► callback function                                  │
└──────────────────────┬───────────────────────────────────────────┘
                       │ مكونة من
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                        spi_transfer                              │
│  tx_buf / rx_buf ──► memory buffers                              │
│  tx_sg / rx_sg ──► sg_table (DMA scatter-gather)                 │
│  ptp_sts ──► ptp_system_timestamp                                │
│  delay / cs_change_delay / word_delay ──► spi_delay             │
│  transfer_list ──► linked into spi_message.transfers             │
└──────────────────────────────────────────────────────────────────┘
```

---

### رسم ASCII للعلاقات الهرمية

```
spi_bus_type
    │
    ├── spi_controller [bus_num=0]
    │       │
    │       ├── spi_device [cs=0]  ◄── spi_driver (protocol driver)
    │       │       └── spi_statistics (per-cpu)
    │       │
    │       ├── spi_device [cs=1]  ◄── spi_driver
    │       │
    │       ├── spi_statistics (per-cpu)
    │       │
    │       └── queue:
    │               ├── spi_message ──► spi_device[cs=0]
    │               │       ├── spi_transfer[0]
    │               │       ├── spi_transfer[1]
    │               │       └── spi_res[]
    │               │
    │               └── spi_message ──► spi_device[cs=1]
    │
    └── spi_controller [bus_num=1]
            └── ...
```

---

### دورة حياة الـ Controller

```
┌─────────────────────────────────────────────────────────┐
│  CREATION                                               │
│                                                         │
│  spi_alloc_host(dev, sizeof(priv))                      │
│    └── __spi_alloc_controller(dev, size, false)         │
│          └── kzalloc(sizeof(*ctlr) + size)              │
│                └── device_initialize(&ctlr->dev)        │
└────────────────────────────┬────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────┐
│  SETUP                                                  │
│                                                         │
│  driver fills in ctlr fields:                           │
│    ctlr->bus_num         = N                            │
│    ctlr->num_chipselect  = M                            │
│    ctlr->mode_bits       = SPI_CPOL | SPI_CPHA | ...   │
│    ctlr->bits_per_word_mask = SPI_BPW_MASK(8)          │
│    ctlr->transfer_one    = my_transfer_one              │
│    ctlr->set_cs          = my_set_cs                    │
└────────────────────────────┬────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────┐
│  REGISTRATION                                           │
│                                                         │
│  spi_register_controller(ctlr)                          │
│    ├── بيـ parse الـ device tree/ACPI/board info        │
│    ├── بيـ create spi_device لكل entry                 │
│    ├── بيـ start الـ kthread message pump              │
│    └── device_add(&ctlr->dev) → sysfs entry            │
└────────────────────────────┬────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────┐
│  RUNTIME                                                │
│                                                         │
│  messages بتيجي من protocol drivers                     │
│  kthread pump بيشتغل:                                   │
│    prepare_transfer_hardware()                          │
│      ├── loop: transfer_one_message() أو transfer_one() │
│    unprepare_transfer_hardware()                        │
└────────────────────────────┬────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────┐
│  TEARDOWN                                               │
│                                                         │
│  spi_unregister_controller(ctlr)                        │
│    ├── بيوقف الـ kthread                               │
│    ├── بيـ unregister كل الـ spi_device children       │
│    └── device_del(&ctlr->dev)                           │
│                                                         │
│  spi_controller_put(ctlr)                               │
│    └── put_device() → كفاية refs تصفر → kfree          │
└─────────────────────────────────────────────────────────┘
```

---

### دورة حياة الـ spi_device

```
┌────────────────────────────────────────────────────────┐
│  CREATION                                              │
│  spi_alloc_device(ctlr)                                │
│    └── kzalloc(sizeof(spi_device))                     │
│          └── device_initialize(&spi->dev)              │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│  CONFIGURATION (من board_info أو DT)                   │
│    spi->max_speed_hz = ...                             │
│    spi->mode         = SPI_MODE_0                      │
│    spi->bits_per_word = 8                              │
│    spi->chip_select[0] = N                             │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│  REGISTRATION                                          │
│  spi_add_device(spi)                                   │
│    ├── ctlr->setup(spi) → validate & configure HW     │
│    └── device_add(&spi->dev) → driver probe()          │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│  RUNTIME                                               │
│  spi_driver.probe()                                    │
│    └── بيبعت messages عن طريق spi_sync / spi_async    │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│  TEARDOWN                                              │
│  spi_unregister_device(spi)                            │
│    ├── spi_driver.remove()                             │
│    └── device_del() → ctlr->cleanup(spi)              │
└────────────────────────────────────────────────────────┘
```

---

### دورة حياة الـ Message (Data Flow)

```
┌────────────────────────────────────────────────────────┐
│  PREPARATION (protocol driver)                         │
│                                                        │
│  spi_message_init(&msg)                                │
│  spi_message_add_tail(&xfer, &msg)                     │
│  msg.complete = my_callback                            │
│  msg.context  = my_data                                │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│  OPTIONAL OPTIMIZATION                                 │
│                                                        │
│  spi_optimize_message(spi, &msg)                       │
│    └── ctlr->optimize_message(&msg)                    │
│          (pre-map DMA, pre-compute timings, ...)       │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│  SUBMISSION                                            │
│                                                        │
│  spi_async(spi, &msg)          ← non-blocking         │
│    OR                                                  │
│  spi_sync(spi, &msg)           ← blocking             │
│    └── ctlr->transfer(spi, &msg)                       │
│          └── قائمة ctlr->queue                         │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│  PUMP EXECUTION (kthread)                              │
│                                                        │
│  spi_get_next_queued_message(ctlr)                     │
│    │                                                   │
│    ▼                                                   │
│  ctlr->prepare_transfer_hardware(ctlr)                 │
│    │                                                   │
│    ▼                                                   │
│  ctlr->prepare_message(ctlr, msg)                      │
│    │                                                   │
│    ▼                                                   │
│  [for each spi_transfer in msg.transfers]              │
│    ctlr->set_cs(spi, true)      → assert CS           │
│    ctlr->transfer_one(ctlr, spi, xfer)                 │
│      → HW register write/read                          │
│    spi_finalize_current_transfer(ctlr)                 │
│    [optional] ctlr->set_cs(spi, false) → deassert CS  │
│    │                                                   │
│    ▼                                                   │
│  ctlr->unprepare_message(ctlr, msg)                    │
│    │                                                   │
│    ▼                                                   │
│  spi_finalize_current_message(ctlr)                    │
│    └── msg->complete(msg->context)   ← callback       │
│                                                        │
│  ctlr->unprepare_transfer_hardware(ctlr)               │
└────────────────────────────────────────────────────────┘
```

---

### Call Flow Diagrams

#### الـ spi_sync Path (Synchronous Transfer)

```
protocol driver
  spi_sync(spi, msg)
    │
    ├─[fast path: queue_empty]──────────────────────────────┐
    │   __spi_pump_transfer_engine()                        │
    │     → ctlr->transfer_one_message()                    │
    │         → spi_finalize_current_message()              │
    │               → complete(&msg_done)                   │
    │                                                       │
    └─[normal path]─────────────────────────────────────────┤
        spi_async(spi, msg)                                 │
          ctlr->transfer(spi, msg)                          │
            list_add_tail(&msg->queue, &ctlr->queue)        │
            kthread_queue_work(ctlr->kworker, &pump)        │
                                                            │
        wait_for_completion(&done) ◄─────────────────────── ┘
```

#### الـ transfer_one_message Path (Generic Queue)

```
spi_pump_messages() [kthread]
  │
  ▼
ctlr->prepare_transfer_hardware(ctlr)    [optional]
  │
  ▼
ctlr->prepare_message(ctlr, msg)         [optional, e.g. DMA map]
  │
  ▼
spi_transfer_one_message(ctlr, msg)
  │
  ├── [foreach xfer in msg->transfers]
  │     │
  │     ▼
  │   __spi_validate_bits_per_word()
  │     │
  │     ▼
  │   spi_set_cs(spi, true)              → ctlr->set_cs(spi, enable)
  │     │
  │     ▼
  │   ctlr->transfer_one(ctlr, spi, xfer)
  │     │           └──► HW FIFO / DMA write
  │     │
  │     ├─[returns 1: async]
  │     │   wait_for_completion_timeout(&xfer_completion)
  │     │
  │     └─[returns 0: sync done]
  │         │
  │         ▼
  │       spi_delay_exec(&xfer->delay, xfer)
  │       [if cs_change] spi_set_cs(spi, false/true)
  │
  ▼
spi_set_cs(spi, false)                   [deassert CS]
  │
  ▼
spi_finalize_current_message(ctlr)
  └── msg->complete(msg->context)
```

#### الـ Driver Registration Flow

```
module_init / module_spi_driver
  │
  ▼
spi_register_driver(drv)
  └── __spi_register_driver(THIS_MODULE, drv)
        └── driver_register(&drv->driver)
              │
              ▼
            [for each spi_device on spi_bus_type]
              bus_type.match()
                └── spi_match_device()
                      ├── by id_table
                      ├── by device tree compatible
                      └── by modalias
                            │
                            ▼
                          drv->probe(spi_device)
```

---

### استراتيجية الـ Locking

#### الجدول الشامل

| Lock | النوع | بيحمي إيه | ملاحظات |
|------|-------|-----------|---------|
| `ctlr->queue_lock` | spinlock | `ctlr->queue`, `ctlr->busy`, `ctlr->running` | يتاستخدم من interrupt context |
| `ctlr->io_mutex` | mutex | الـ physical bus access الكامل | بيضمن sequential access |
| `ctlr->bus_lock_mutex` | mutex | الـ exclusive bus lock | بيتاستخدم مع `spi_bus_lock()` |
| `ctlr->bus_lock_spinlock` | spinlock | `ctlr->bus_lock_flag` | للـ fast check |
| `ctlr->add_lock` | mutex | إضافة spi_device لنفس الـ CS | بيمنع race في التسجيل |
| `spi_statistics.syncp` | seqcount | الـ 64-bit counters في الإحصائيات | على 32-bit systems بس |

#### ترتيب الـ Locking (Lock Ordering)

```
عشان تتجنب الـ deadlock، لازم تاخد الـ locks بالترتيب ده:

1. ctlr->bus_lock_mutex    (الأعلى)
       │
       ▼
2. ctlr->io_mutex
       │
       ▼
3. ctlr->queue_lock        (spinlock — أسرع)
       │
       ▼
4. spi_statistics.syncp    (seqcount — أضعف)
```

#### الـ Bus Lock API

```c
/* لما محتاج series من messages بدون تدخل من حد تاني */
spi_bus_lock(ctlr);       // بياخد bus_lock_mutex
    spi_sync_locked(spi, msg1);
    spi_sync_locked(spi, msg2);
spi_bus_unlock(ctlr);     // بيحرر bus_lock_mutex
```

#### حماية الـ Queue

```
[protocol driver context]          [kthread context]
spi_async()                        spi_pump_messages()
  spin_lock_irqsave(queue_lock)      spin_lock_irqsave(queue_lock)
  list_add_tail(msg, queue)          msg = list_first_entry(queue)
  spin_unlock_irqrestore(...)        ctlr->busy = true
  kthread_queue_work(pump)           spin_unlock_irqrestore(...)
                                     [process msg without lock]
                                     spin_lock_irqsave(queue_lock)
                                     ctlr->busy = false
                                     spin_unlock_irqrestore(...)
```

#### الـ Per-CPU Statistics

الـ `spi_statistics` بيتحدث بـ `u64_stats_update_begin/end` اللي بتستخدم seqcount على 32-bit وبتكون no-op على 64-bit. الـ `get_cpu()`/`put_cpu()` بتضمن إن الـ thread مش بيتحول لـ CPU تاني أثناء التحديث.

```c
/* مثال: تسجيل transfer */
SPI_STATISTICS_INCREMENT_FIELD(spi->pcpu_statistics, transfers);
/* هيـ expand لـ: */
get_cpu();
__lstats = this_cpu_ptr(spi->pcpu_statistics);
u64_stats_update_begin(&__lstats->syncp);
u64_stats_inc(&__lstats->transfers);
u64_stats_update_end(&__lstats->syncp);
put_cpu();
```
## Phase 4: شرح الـ Functions

---

## ملخص سريع — Cheatsheet

### Registration & Allocation

| Function | النوع | الوصف المختصر |
|---|---|---|
| `spi_register_driver(drv)` | macro | يسجل `spi_driver` مع `THIS_MODULE` |
| `__spi_register_driver(owner, drv)` | extern | الـ implementation الفعلية للتسجيل |
| `spi_unregister_driver(sdrv)` | inline | يلغي تسجيل الـ protocol driver |
| `module_spi_driver(drv)` | macro | boilerplate كامل لـ module_init/exit |
| `spi_alloc_host(dev, size)` | inline | يخصص `spi_controller` لـ host mode |
| `spi_alloc_target(dev, size)` | inline | يخصص `spi_controller` لـ target/slave mode |
| `devm_spi_alloc_host(dev, size)` | inline | نفسه بس devres-managed |
| `devm_spi_alloc_target(dev, size)` | inline | نفسه بس devres-managed |
| `__spi_alloc_controller(dev, size, target)` | extern | الـ backend الفعلي للتخصيص |
| `spi_register_controller(ctlr)` | extern | يسجل الـ controller في kernel bus |
| `devm_spi_register_controller(dev, ctlr)` | extern | نفسه بـ devres |
| `spi_unregister_controller(ctlr)` | extern | يلغي تسجيل الـ controller |

### Device Management

| Function | النوع | الوصف المختصر |
|---|---|---|
| `spi_alloc_device(ctlr)` | extern | يخصص `spi_device` جديد |
| `spi_add_device(spi)` | extern | يضيف الـ device للـ bus بعد تهيئته يدوياً |
| `spi_new_device(ctlr, bi)` | extern | يعمل alloc + add في خطوة واحدة |
| `spi_unregister_device(spi)` | extern | يزيل الـ device من الـ bus |
| `spi_new_ancillary_device(spi, cs)` | extern | يخصص device مساعد على نفس الـ controller |
| `spi_dev_get(spi)` | inline | يزيد الـ reference count |
| `spi_dev_put(spi)` | inline | يقلل الـ reference count |
| `spi_register_board_info(info, n)` | extern | يسجل static board-level device table |

### Transfer & Message

| Function | النوع | الوصف المختصر |
|---|---|---|
| `spi_setup(spi)` | extern | يهيئ clock/mode/bits للـ device |
| `spi_async(spi, msg)` | extern | يرسل message بشكل async (لا ينام) |
| `spi_sync(spi, msg)` | extern | يرسل message بشكل sync (ينام) |
| `spi_sync_locked(spi, msg)` | extern | sync مع bus lock مسبق |
| `spi_sync_transfer(spi, xfers, n)` | inline | sync helper لـ array من transfers |
| `spi_write(spi, buf, len)` | inline | write-only sync transfer |
| `spi_read(spi, buf, len)` | inline | read-only sync transfer |
| `spi_write_then_read(spi, tx, ntx, rx, nrx)` | extern | write ثم read في message واحد |
| `spi_w8r8(spi, cmd)` | inline | write byte واحد, read byte واحد |
| `spi_w8r16(spi, cmd)` | inline | write byte واحد, read 16-bit |
| `spi_w8r16be(spi, cmd)` | inline | نفسه مع big-endian conversion |

### Message Init & Manipulation

| Function | النوع | الوصف المختصر |
|---|---|---|
| `spi_message_init(m)` | inline | يصفر الـ message ويهيئ الـ lists |
| `spi_message_init_no_memset(m)` | inline | يهيئ الـ lists بدون memset |
| `spi_message_init_with_transfers(m, xfers, n)` | inline | init + add transfers دفعة واحدة |
| `spi_message_add_tail(t, m)` | inline | يضيف transfer لآخر القائمة |
| `spi_transfer_del(t)` | inline | يحذف transfer من القائمة |
| `spi_message_alloc(ntrans, flags)` | inline | يخصص message + transfers معاً |
| `spi_message_free(m)` | inline | يحرر message المخصص بـ spi_message_alloc |

### Controller Interaction

| Function | النوع | الوصف المختصر |
|---|---|---|
| `spi_finalize_current_message(ctlr)` | extern | يُعلم الـ core بانتهاء الـ message |
| `spi_finalize_current_transfer(ctlr)` | extern | يُعلم الـ core بانتهاء الـ transfer |
| `spi_get_next_queued_message(ctlr)` | extern | يسحب الـ message التالي من الـ queue |
| `spi_controller_suspend(ctlr)` | extern | يوقف الـ message pump |
| `spi_controller_resume(ctlr)` | extern | يُعيد تشغيل الـ message pump |
| `spi_take_timestamp_pre(ctlr, xfer, prog, irq)` | extern | يأخذ PTP snapshot قبل الـ word |
| `spi_take_timestamp_post(ctlr, xfer, prog, irq)` | extern | يأخذ PTP snapshot بعد الـ word |

### Optimization & Limits

| Function | النوع | الوصف المختصر |
|---|---|---|
| `spi_optimize_message(spi, msg)` | extern | يعمل pre-optimization للـ message |
| `spi_unoptimize_message(msg)` | extern | يلغي الـ optimization |
| `devm_spi_optimize_message(dev, spi, msg)` | extern | optimize مع devres |
| `spi_split_transfers_maxsize(ctlr, msg, max)` | extern | يقسم transfers تتجاوز الـ max size |
| `spi_split_transfers_maxwords(ctlr, msg, max)` | extern | يقسم transfers تتجاوز عدد words |
| `spi_max_message_size(spi)` | inline | يرجع max message size للـ device |
| `spi_max_transfer_size(spi)` | inline | يرجع max transfer size (أقل قيمة) |
| `spi_is_bpw_supported(spi, bpw)` | inline | يتحقق إذا كان bits_per_word مدعوم |
| `spi_bpw_to_bytes(bpw)` | inline | يحول bits_per_word لـ bytes (power-of-two) |
| `spi_controller_xfer_timeout(ctlr, xfer)` | inline | يحسب timeout مناسب للـ transfer |

### Helpers & Accessors

| Function | النوع | الوصف المختصر |
|---|---|---|
| `spi_get_ctldata(spi)` | inline | يقرأ `controller_state` |
| `spi_set_ctldata(spi, state)` | inline | يكتب `controller_state` |
| `spi_get_drvdata(spi)` | inline | يقرأ driver private data |
| `spi_set_drvdata(spi, data)` | inline | يكتب driver private data |
| `spi_get_chipselect(spi, idx)` | inline | يرجع CS رقم idx |
| `spi_set_chipselect(spi, idx, cs)` | inline | يضبط CS رقم idx |
| `spi_get_csgpiod(spi, idx)` | inline | يرجع GPIO descriptor لـ CS |
| `spi_set_csgpiod(spi, idx, gpiod)` | inline | يضبط GPIO descriptor لـ CS |
| `spi_is_csgpiod(spi)` | inline | يتحقق إذا كان أي CS هو GPIO |
| `spi_controller_get_devdata(ctlr)` | inline | يقرأ driver private data للـ controller |
| `spi_controller_set_devdata(ctlr, data)` | inline | يكتب driver private data للـ controller |
| `spi_controller_get(ctlr)` | inline | يزيد refcount للـ controller |
| `spi_controller_put(ctlr)` | inline | يقلل refcount للـ controller |
| `spi_controller_is_target(ctlr)` | inline | يتحقق إذا كان الـ controller في target mode |
| `spi_transfer_is_last(ctlr, xfer)` | inline | يتحقق إذا كان الـ transfer هو الأخير في الـ message |
| `spi_transfer_delay_exec(t)` | inline | ينفذ الـ delay المحدد في الـ transfer |
| `spi_delay_to_ns(delay, xfer)` | extern | يحول `spi_delay` لـ nanoseconds |
| `spi_delay_exec(delay, xfer)` | extern | ينفذ الـ delay (busy-wait أو sleep) |
| `spi_transfer_cs_change_delay_exec(msg, xfer)` | extern | ينفذ cs_change_delay |
| `spi_target_abort(spi)` | extern | يلغي transfer جاري في target mode |
| `spi_get_device_id(sdev)` | extern | يرجع `spi_device_id` المطابق |
| `spi_get_device_match_data(sdev)` | extern | يرجع الـ driver match data |

### ACPI & OF Finders

| Function | النوع | الوصف المختصر |
|---|---|---|
| `of_find_spi_controller_by_node(node)` | extern | يبحث عن controller بـ OF node |
| `acpi_spi_find_controller_by_adev(adev)` | extern | يبحث عن controller بـ ACPI device |
| `acpi_spi_device_alloc(ctlr, adev, idx)` | extern | يخصص `spi_device` من ACPI resource |
| `acpi_spi_count_resources(adev)` | extern | يعد SPI resources في ACPI device |

---

## Group 1: Driver & Controller Registration

### الغرض
هذه المجموعة تربط بين الـ protocol driver وبين الـ `spi_bus_type`. الـ SPI core يستخدم الـ Linux driver model — كل `spi_driver` يُسجَّل كـ `device_driver` على الـ `spi_bus_type`، وكل `spi_device` يُسجَّل كـ `device` على نفس الـ bus.

---

### `spi_register_driver` / `__spi_register_driver`

```c
/* Macro — يمرر THIS_MODULE تلقائياً */
#define spi_register_driver(driver) \
    __spi_register_driver(THIS_MODULE, driver)

extern int __spi_register_driver(struct module *owner, struct spi_driver *sdrv);
```

**الـ macro** يحمي من الخطأ الشائع بأن تمرر module غلط. الـ `__spi_register_driver` يفعل:
1. يضبط `sdrv->driver.bus = &spi_bus_type`
2. يستدعي `driver_register(&sdrv->driver)`
3. الـ bus يبدأ يـmatch الـ driver مع كل `spi_device` موجود

**Parameters:**
- `owner` — الـ module المالك (دايماً `THIS_MODULE`)
- `sdrv` — الـ `spi_driver` struct، لازم يكون `id_table` أو `driver.of_match_table` مضبوط

**Return:** 0 نجاح، negative errno فشل (مثلاً `-EBUSY` لو الاسم مكرر)

**Key details:** لازم يتنادى من `module_init()`. بعد النجاح، الـ bus سيستدعي `probe()` على كل device يطابق الـ id_table.

---

### `spi_unregister_driver`

```c
static inline void spi_unregister_driver(struct spi_driver *sdrv)
{
    if (sdrv)
        driver_unregister(&sdrv->driver);
}
```

يُعاكس `spi_register_driver`. الـ bus سيستدعي `remove()` على كل device مربوط بالـ driver قبل ما ينزل الـ driver.

**Context:** can sleep — لأن `driver_unregister` ينتظر الـ remove callbacks.

---

### `module_spi_driver`

```c
#define module_spi_driver(__spi_driver) \
    module_driver(__spi_driver, spi_register_driver, spi_unregister_driver)
```

يولّد `module_init()` و `module_exit()` تلقائياً. معظم الـ SPI drivers البسيطة تستخدمه بدل ما تكتب init/exit بنفسها.

**مثال:**
```c
/* بدل init/exit boilerplate */
module_spi_driver(my_spi_driver);
```

---

### `__spi_alloc_controller` / `spi_alloc_host` / `spi_alloc_target`

```c
extern struct spi_controller *__spi_alloc_controller(struct device *host,
                                                      unsigned int size,
                                                      bool target);

static inline struct spi_controller *spi_alloc_host(struct device *dev,
                                                     unsigned int size)
{
    return __spi_alloc_controller(dev, size, false);
}

static inline struct spi_controller *spi_alloc_target(struct device *dev,
                                                       unsigned int size)
{
    if (!IS_ENABLED(CONFIG_SPI_SLAVE))
        return NULL;
    return __spi_alloc_controller(dev, size, true);
}
```

يخصص `spi_controller` مع مساحة إضافية للـ driver private data مباشرةً بعده في الـ memory. الـ `__spi_alloc_controller` يعمل:
1. `kzalloc(sizeof(spi_controller) + size)` — الـ private data في `ctlr + 1`
2. يهيئ الـ `device` struct
3. يضبط `ctlr->target = target`
4. يهيئ الـ mutexes والـ spinlocks والـ lists

**Parameters:**
- `dev` — الـ parent device (عادةً platform device)
- `size` — حجم الـ private data للـ controller driver
- `target` — true لو slave/target mode

**Return:** pointer للـ controller أو NULL لو فشل الـ alloc

**Key details:** الـ private data يُقرأ بـ `spi_controller_get_devdata()`. لا تستخدم `kfree` مباشرة — الـ core يتحكم في دورة حياته عبر الـ device model.

---

### `devm_spi_alloc_host` / `devm_spi_alloc_target`

```c
static inline struct spi_controller *devm_spi_alloc_host(struct device *dev,
                                                          unsigned int size)
{
    return __devm_spi_alloc_controller(dev, size, false);
}
```

نفس `spi_alloc_host` لكن مربوط بـ devres — لما `dev` يُحرَّر، الـ controller يُحرَّر تلقائياً. **الخيار المفضل** للـ drivers الحديثة.

---

### `spi_register_controller` / `devm_spi_register_controller`

```c
extern int spi_register_controller(struct spi_controller *ctlr);
extern int devm_spi_register_controller(struct device *dev,
                                        struct spi_controller *ctlr);
```

يسجل الـ controller في الـ kernel ويجعله مرئياً للـ bus. الـ core سيفعل:

```
spi_register_controller(ctlr):
  ├── تحقق من صحة الـ fields (num_chipselect, etc.)
  ├── اطلب bus_num (أو خصص تلقائي لو سالب)
  ├── سجل الـ device: device_add(&ctlr->dev)
  ├── لو queued mode: أنشئ الـ kthread worker
  ├── لو use_gpio_descriptors: اجمع الـ GPIO CS descriptors
  └── scan board_info و of/acpi لإنشاء spi_devices
```

**Return:** 0 نجاح، negative errno فشل

**Context:** can sleep — يُستدعى من `probe()` للـ platform driver.

---

### `spi_unregister_controller`

```c
extern void spi_unregister_controller(struct spi_controller *ctlr);
```

يُعاكس `spi_register_controller`. يُزيل كل الـ `spi_device` children ثم يُزيل الـ controller نفسه من الـ bus. يوقف الـ kthread worker لو كان شغّال.

---

## Group 2: SPI Device Management

### الغرض
إنشاء وإدارة الـ `spi_device` objects — سواء من board tables أو DT/ACPI أو dynamically.

---

### `spi_alloc_device`

```c
extern struct spi_device *spi_alloc_device(struct spi_controller *ctlr);
```

يخصص `spi_device` جديد ويضبط:
- `spi->controller = ctlr`
- `spi->dev.parent = &ctlr->dev`
- `spi->dev.bus = &spi_bus_type`
- يخصص الـ `pcpu_statistics`

**Return:** pointer للـ device أو NULL

**من يستدعيه:** كود إنشاء الـ devices من DT أو ACPI أو manually.

---

### `spi_add_device`

```c
extern int spi_add_device(struct spi_device *spi);
```

يُكمل تسجيل الـ `spi_device` بعد ما ضبطت الـ fields يدوياً. يتحقق من:
- CS صالح
- لا يوجد device آخر على نفس الـ CS
- يستدعي `ctlr->setup(spi)` لو موجود

ثم يستدعي `device_add(&spi->dev)`.

**Return:** 0 أو negative errno

---

### `spi_new_device`

```c
extern struct spi_device *spi_new_device(struct spi_controller *ctlr,
                                          struct spi_board_info *chip);
```

يجمع `spi_alloc_device()` + ملء الـ fields من `spi_board_info` + `spi_add_device()` في خطوة واحدة.

```
spi_new_device():
  ├── spi_alloc_device(ctlr)
  ├── strscpy(spi->modalias, chip->modalias)
  ├── spi->max_speed_hz = chip->max_speed_hz
  ├── spi->mode = chip->mode
  ├── spi->irq = chip->irq
  └── spi_add_device(spi)
```

**Return:** pointer للـ device أو NULL

---

### `spi_unregister_device`

```c
extern void spi_unregister_device(struct spi_device *spi);
```

يستدعي `device_unregister(&spi->dev)` — الـ bus يستدعي `remove()` للـ driver أولاً لو كان مربوط.

---

### `spi_register_board_info`

```c
extern int spi_register_board_info(struct spi_board_info const *info, unsigned n);
```

يخزن static table من board-level device descriptions في قائمة global. لما `spi_register_controller()` يتنادى لاحقاً، يمشي على هذه القائمة ويخلق الـ devices اللي `bus_num` تاعتهم يطابق الـ controller.

**استخدام نموذجي في board file:**
```c
static struct spi_board_info board_spi_devices[] = {
    {
        .modalias    = "m25p80",
        .max_speed_hz = 20000000,
        .bus_num     = 0,
        .chip_select = 0,
        .mode        = SPI_MODE_0,
    },
};

/* في board_init() */
spi_register_board_info(board_spi_devices, ARRAY_SIZE(board_spi_devices));
```

---

## Group 3: Transfer & Message API

### الغرض
هذه المجموعة هي قلب الـ SPI API للـ protocol drivers. كل عملية نقل بيانات تمر من هنا.

---

### `spi_setup`

```c
extern int spi_setup(struct spi_device *spi);
```

يُطبّق إعدادات الـ device (mode, speed, bits_per_word) على الـ hardware. يستدعي `ctlr->setup(spi)`. مهم جداً: **يُستدعى فقط لو مفيش transfers جارية** على الـ device.

**Return:** 0 أو negative errno (مثلاً `-EINVAL` لو mode غير مدعوم)

**Context:** can sleep

---

### `spi_async`

```c
extern int spi_async(struct spi_device *spi, struct spi_message *message);
```

أساس كل الـ SPI transfers. يضيف الـ message للـ queue **بدون ما ينام**. الـ completion يتم عبر `message->complete(message->context)` callback.

**Parameters:**
- `spi` — الـ target device
- `message` — الـ message، لازم يكون `complete` callback مضبوط

**Return:** 0 لو اتضاف للـ queue، negative errno لو فشل قبل الـ queue

**Key details:**
- **لا ينام** — آمن في interrupt context لو الـ controller يدعم ذلك
- الـ memory (message + transfers + buffers) يجب أن يبقى صالحاً حتى يُستدعى الـ callback
- يأخذ `bus_lock_mutex` لو `bus_lock_flag` مضبوط

**Locking:** الـ message يُضاف تحت `queue_lock` spinlock

---

### `spi_sync`

```c
extern int spi_sync(struct spi_device *spi, struct spi_message *message);
```

يستدعي `spi_async()` ثم ينام حتى يكتمل الـ message. يدعم **fast path** لو `queue_empty && !must_async` — يُنفذ الـ message مباشرةً في calling context بدون queue.

```
spi_sync():
  ├── لو queue_empty: نفّذ مباشرة (spi_sync_immediate++)
  └── غير كذلك:
      ├── set message->complete = spi_complete
      ├── spi_async(spi, message)
      └── wait_for_completion(&done)
  └── return message->status
```

**Return:** 0 نجاح، negative errno فشل

**Context:** can sleep — **لا يُستدعى من interrupt context**

---

### `spi_sync_locked`

```c
extern int spi_sync_locked(struct spi_device *spi, struct spi_message *message);
```

مثل `spi_sync` لكن يفترض إن المستدعي يمتلك `bus_lock_mutex` مسبقاً (بعد `spi_bus_lock()`). يُستخدم لضمان atomic sequence من عدة messages.

---

### `spi_bus_lock` / `spi_bus_unlock`

```c
extern int spi_bus_lock(struct spi_controller *ctlr);
extern int spi_bus_unlock(struct spi_controller *ctlr);
```

يقفل الـ bus للاستخدام الحصري. بعد `spi_bus_lock()`:
- الـ `spi_sync` من threads أخرى ستُحجَب
- تستخدم `spi_sync_locked()` لإرسال messages

**مثال:**
```c
spi_bus_lock(ctlr);
spi_sync_locked(spi, &msg1);
spi_sync_locked(spi, &msg2);
spi_bus_unlock(ctlr);
```

---

### `spi_sync_transfer`

```c
static inline int spi_sync_transfer(struct spi_device *spi,
                                     struct spi_transfer *xfers,
                                     unsigned int num_xfers)
{
    struct spi_message msg;

    spi_message_init_with_transfers(&msg, xfers, num_xfers);
    return spi_sync(spi, &msg);
}
```

helper مريح لإرسال array من transfers بدون تهيئة يدوية للـ message. الـ message على الـ stack — آمن لأن `spi_sync` ينتظر الاكتمال.

---

### `spi_write` / `spi_read`

```c
static inline int spi_write(struct spi_device *spi, const void *buf, size_t len)
{
    struct spi_transfer t = { .tx_buf = buf, .len = len };
    return spi_sync_transfer(spi, &t, 1);
}

static inline int spi_read(struct spi_device *spi, void *buf, size_t len)
{
    struct spi_transfer t = { .rx_buf = buf, .len = len };
    return spi_sync_transfer(spi, &t, 1);
}
```

أبسط helpers للـ write-only وread-only. الـ rx_buf في `spi_write` يكون NULL — الـ core يُرسل dummy clocks ويتجاهل المستلم. لـ **الاستخدام البسيط فقط** — للأداء استخدم `spi_sync` مباشرة مع pre-allocated structures.

---

### `spi_write_then_read`

```c
extern int spi_write_then_read(struct spi_device *spi,
                                const void *txbuf, unsigned n_tx,
                                void *rxbuf, unsigned n_rx);
```

ينفذ write ثم read في message واحد بـ **two transfers**. ينسخ البيانات من/إلى kernel stack buffers — **للـ transfers الصغيرة فقط** (تحت حجم معين محدد kernel-internally).

```
spi_write_then_read():
  ├── xfer[0]: tx_buf = local_buf(txbuf), len = n_tx
  ├── xfer[1]: rx_buf = local_buf(rxbuf), len = n_rx
  ├── spi_message_init_with_transfers(&msg, xfer, 2)
  ├── spi_sync(spi, &msg)
  └── memcpy(rxbuf, local_buf, n_rx)
```

**Return:** 0 أو negative errno

**Key details:** يستخدم mutex داخلي (`lock`) لحماية الـ static buffers — **لا تستخدمه في hot paths**.

---

### `spi_w8r8` / `spi_w8r16` / `spi_w8r16be`

```c
static inline ssize_t spi_w8r8(struct spi_device *spi, u8 cmd)
{
    ssize_t status;
    u8 result;
    status = spi_write_then_read(spi, &cmd, 1, &result, 1);
    return (status < 0) ? status : result;
}
```

**الـ w8r8**: يبعت command byte ويستقبل byte واحد رداً. شائع جداً مع sensors وEEPROMs.

**الـ w8r16**: نفسه لكن يستقبل 16-bit في wire-order (قد يكون big-endian).

**الـ w8r16be**: نفسه لكن يعمل `be16_to_cpu()` على النتيجة — للأجهزة اللي ترد بـ big-endian explicitly.

**Return:** القيمة المقروءة (unsigned) أو negative errno لو فشل

---

## Group 4: Message & Transfer Initialization

### الغرض
تهيئة الـ `spi_message` والـ `spi_transfer` structures قبل الإرسال.

---

### `spi_message_init`

```c
static inline void spi_message_init(struct spi_message *m)
{
    memset(m, 0, sizeof *m);
    spi_message_init_no_memset(m);
}
```

يُصفّر الـ message الكامل ثم يهيئ الـ lists. **يُستخدم للـ messages على الـ stack أو المخصصة statically**. لو الـ message مخصص بـ `kzalloc` استخدم `spi_message_init_no_memset` لتوفير الـ memset.

---

### `spi_message_init_no_memset`

```c
static inline void spi_message_init_no_memset(struct spi_message *m)
{
    INIT_LIST_HEAD(&m->transfers);
    INIT_LIST_HEAD(&m->resources);
}
```

يهيئ الـ lists فقط بدون memset. يُستخدم لما تعرف إن الـ struct مُصفَّر مسبقاً (مثلاً بعد `kzalloc`).

---

### `spi_message_init_with_transfers`

```c
static inline void spi_message_init_with_transfers(struct spi_message *m,
                                                    struct spi_transfer *xfers,
                                                    unsigned int num_xfers)
{
    unsigned int i;
    spi_message_init(m);
    for (i = 0; i < num_xfers; ++i)
        spi_message_add_tail(&xfers[i], m);
}
```

يجمع init + add transfers في خطوة واحدة. مفيد لـ fixed-size transfer arrays.

---

### `spi_message_add_tail`

```c
static inline void spi_message_add_tail(struct spi_transfer *t,
                                         struct spi_message *m)
{
    list_add_tail(&t->transfer_list, &m->transfers);
}
```

يُضيف transfer لنهاية قائمة الـ message. الـ transfers تُنفَّذ بالترتيب الذي أُضيفت به. **لا تُضيف transfer بعد ما أرسلت الـ message**.

---

### `spi_transfer_del`

```c
static inline void spi_transfer_del(struct spi_transfer *t)
{
    list_del(&t->transfer_list);
}
```

يحذف transfer من القائمة قبل الإرسال. نادر الاستخدام في الكود العادي.

---

### `spi_message_alloc` / `spi_message_free`

```c
static inline struct spi_message *spi_message_alloc(unsigned ntrans, gfp_t flags)
{
    struct spi_message_with_transfers {
        struct spi_message m;
        struct spi_transfer t[];
    } *mwt;

    mwt = kzalloc(struct_size(mwt, t, ntrans), flags);
    if (!mwt) return NULL;

    spi_message_init_no_memset(&mwt->m);
    for (i = 0; i < ntrans; i++)
        spi_message_add_tail(&mwt->t[i], &mwt->m);

    return &mwt->m;
}
```

يخصص message + transfers في allocation واحدة متجاورة في الـ memory. الـ transfers جاهزة ومُضافة للقائمة فوراً. يُستخدم لما تحتاج dynamic allocation مع عدد معروف من transfers.

**الـ `spi_message_free`** يستدعي `kfree(m)` فقط — لأن الـ transfers جزء من نفس الـ allocation.

---

## Group 5: Controller-Side Callbacks (للـ controller drivers)

### الغرض
هذه الـ functions تُستدعى من الـ SPI core **نحو** الـ controller driver. الـ controller driver لا يستدعيها — بل يُنفّذها.

---

### `spi_finalize_current_message`

```c
extern void spi_finalize_current_message(struct spi_controller *ctlr);
```

يُستدعى من الـ controller driver لما ينهي معالجة الـ message الحالي. الـ core سيستدعي `message->complete()` callback ثم يسحب الـ message التالي من الـ queue.

**Key details:**
- **إلزامي** — لو لم تستدعه، الـ queue ستتجمّد
- يُستدعى من interrupt context أو thread context
- يتحقق من `cur_msg_need_completion` flag لتحسين الأداء

**Locking:** يأخذ `queue_lock` داخلياً

---

### `spi_finalize_current_transfer`

```c
extern void spi_finalize_current_transfer(struct spi_controller *ctlr);
```

للـ controllers التي تستخدم `transfer_one` callback بدل `transfer_one_message`. يُعلم الـ core بانتهاء الـ transfer الحالي حتى يمشي للتالي أو يُكمل الـ message.

**يُستدعى من:** interrupt handler أو completion callback في الـ controller driver.

---

### `spi_get_next_queued_message`

```c
extern struct spi_message *spi_get_next_queued_message(struct spi_controller *ctlr);
```

يُرجع الـ message التالي في الـ queue بدون إزالته. يُستخدم من controllers التي تُنفّذ `transfer_one_message` بنفسها وتحتاج peek للقائمة.

**Return:** pointer للـ message أو NULL لو القائمة فارغة

---

### `spi_controller_suspend` / `spi_controller_resume`

```c
extern int spi_controller_suspend(struct spi_controller *ctlr);
extern int spi_controller_resume(struct spi_controller *ctlr);
```

يوقف/يُعيد تشغيل الـ message pump. الـ `suspend` يضبط `SPI_CONTROLLER_SUSPENDED` flag ويوقف الـ kworker. الـ `resume` يُزيل الـ flag ويُعيد تشغيل الـ worker.

**من يستدعيهم:** الـ controller driver من `suspend`/`resume` callbacks.

---

## Group 6: PTP Timestamping

### الغرض
دعم **PTP (Precision Time Protocol)** لمزامنة الوقت عبر SPI. مثال: قراءة ساعة POSIX بدقة عالية.

---

### `spi_take_timestamp_pre` / `spi_take_timestamp_post`

```c
void spi_take_timestamp_pre(struct spi_controller *ctlr,
                             struct spi_transfer *xfer,
                             size_t progress, bool irqs_off);

void spi_take_timestamp_post(struct spi_controller *ctlr,
                              struct spi_transfer *xfer,
                              size_t progress, bool irqs_off);
```

يأخذ system timestamp قبل/بعد إرسال word محدد (محدد بـ `ptp_sts_word_pre`/`ptp_sts_word_post`).

**Parameters:**
- `ctlr` — الـ controller
- `xfer` — الـ transfer الجاري
- `progress` — عدد الـ words المُرسلة حتى الآن
- `irqs_off` — لو true، يستخدم `ptp_read_system_prets`/`postts` بدون تعطيل interrupts إضافي

**من يستدعيهم:** الـ controller driver من داخل الـ transfer loop.

**Key details:** لو `ctlr->ptp_sts_supported = false`، الـ core يأخذ snapshot تلقائياً عند تسليم الـ transfer للـ driver بدون استدعاء هذه الـ functions.

---

## Group 7: Transfer Optimization

### الغرض
تحسين الـ messages المتكررة عن طريق pre-computation وcaching لتقليل overhead في الـ hot path.

---

### `spi_optimize_message`

```c
extern int spi_optimize_message(struct spi_device *spi, struct spi_message *msg);
```

يُشغّل الـ `ctlr->optimize_message()` callback (لو موجود) لـ pre-compute أي حسابات ثقيلة (مثلاً DMA descriptor setup). يضبط `msg->pre_optimized = true`.

**الاستخدام المثالي:** للـ messages التي تُرسَل بشكل متكرر مع نفس البيانات (مثل polling).

**Return:** 0 أو negative errno

---

### `spi_unoptimize_message`

```c
extern void spi_unoptimize_message(struct spi_message *msg);
```

يُحرر الـ resources المخصصة بـ `spi_optimize_message`. **لازم يُستدعى** قبل تحرير الـ message.

---

### `devm_spi_optimize_message`

```c
extern int devm_spi_optimize_message(struct device *dev, struct spi_device *spi,
                                      struct spi_message *msg);
```

يربط `spi_unoptimize_message` بـ devres لـ `dev` — يُستدعى تلقائياً عند `device_remove`.

---

## Group 8: Transfer Splitting

### الغرض
بعض الـ controllers لها حد أقصى لحجم الـ transfer. هذه الـ functions تُقسّم الـ transfers الكبيرة تلقائياً.

---

### `spi_split_transfers_maxsize`

```c
extern int spi_split_transfers_maxsize(struct spi_controller *ctlr,
                                        struct spi_message *msg,
                                        size_t maxsize);
```

يمشي على كل transfer في الـ message — أي transfer حجمه أكبر من `maxsize` يُقسَّم لعدة transfers أصغر. يستخدم `spi_res` لتتبع الـ transfers المُضافة حتى يُرجعها للحالة الأصلية بعد اكتمال الـ message.

**من يستدعيها:** الـ `prepare_message` callback في الـ controller driver.

**Return:** 0 أو negative errno (مثلاً `-ENOMEM`)

---

### `spi_split_transfers_maxwords`

```c
extern int spi_split_transfers_maxwords(struct spi_controller *ctlr,
                                         struct spi_message *msg,
                                         size_t maxwords);
```

نفس `spi_split_transfers_maxsize` لكن الحد بالـ **words** (بحسب `bits_per_word`) بدل الـ bytes.

---

## Group 9: Limits & Capability Queries

### الغرض
queries لقدرات الـ controller وحدوده — يُستخدم من الـ protocol drivers للتحقق قبل الإرسال.

---

### `spi_max_message_size` / `spi_max_transfer_size`

```c
static inline size_t spi_max_message_size(struct spi_device *spi)
{
    struct spi_controller *ctlr = spi->controller;
    if (!ctlr->max_message_size)
        return SIZE_MAX;
    return ctlr->max_message_size(spi);
}

static inline size_t spi_max_transfer_size(struct spi_device *spi)
{
    /* returns min(max_transfer_size, max_message_size) */
}
```

يتحقق إذا كان الـ controller يُعرّف حداً أقصى. لو لا يوجد callback، يُرجع `SIZE_MAX` (لا حد). الـ `spi_max_transfer_size` يُرجع الحد الأدنى بين الـ transfer limit والـ message limit لأن transfer لا يمكن أن يتجاوز حجم الـ message.

---

### `spi_is_bpw_supported`

```c
static inline bool spi_is_bpw_supported(struct spi_device *spi, u32 bpw)
{
    u32 bpw_mask = spi->controller->bits_per_word_mask;
    if (bpw == 8 || (bpw <= 32 && bpw_mask & SPI_BPW_MASK(bpw)))
        return true;
    return false;
}
```

يتحقق إذا كان `bpw` مدعوماً. الـ 8-bit مدعوم دايماً (افتراضي). غيره يتحقق من `bits_per_word_mask` في الـ controller.

**استخدام من driver:**
```c
if (!spi_is_bpw_supported(spi, 16)) {
    dev_err(&spi->dev, "16-bit transfers not supported\n");
    return -EINVAL;
}
```

---

### `spi_bpw_to_bytes`

```c
static inline u32 spi_bpw_to_bytes(u32 bpw)
{
    return roundup_pow_of_two(BITS_TO_BYTES(bpw));
}
```

يُحوّل bits_per_word لعدد bytes لازم لكل word، مقرّباً لـ power of two. مثلاً: 9 bits → 2 bytes، 17 bits → 4 bytes.

---

### `spi_controller_xfer_timeout`

```c
static inline unsigned int spi_controller_xfer_timeout(struct spi_controller *ctlr,
                                                         struct spi_transfer *xfer)
{
    return max(xfer->len * 8 * 2 / (xfer->speed_hz / 1000), 500U);
}
```

يحسب timeout بالـ milliseconds:
- يحسب وقت الـ transfer النظري على خط واحد: `(len * 8 bits) / speed_hz`
- يضاعفه × 2 كـ safety margin
- minimum 500ms لتجنب false positives على systems محملة

**يُستخدم من:** controller drivers لضبط timeout للـ DMA أو interrupt-driven transfers.

---

## Group 10: Delay Handling

### الغرض
تنفيذ الـ delays المحددة في `spi_transfer` وبين الـ chip selects.

---

### `spi_delay_to_ns`

```c
extern int spi_delay_to_ns(struct spi_delay *_delay, struct spi_transfer *xfer);
```

يُحوّل `spi_delay` (الذي قد يكون بـ microseconds، nanoseconds، أو SCK cycles) لـ nanoseconds. لو الـ unit هو `SPI_DELAY_UNIT_SCK` يحتاج `xfer->effective_speed_hz` لحساب مدة الـ clock cycle.

**Return:** القيمة بالـ nanoseconds أو negative errno

---

### `spi_delay_exec`

```c
extern int spi_delay_exec(struct spi_delay *_delay, struct spi_transfer *xfer);
```

ينفذ الـ delay فعلياً — يستدعي `spi_delay_to_ns` ثم:
- لو < threshold: `ndelay()` (busy-wait)
- لو أكبر: `usleep_range()`

**Return:** 0 أو negative errno

---

### `spi_transfer_cs_change_delay_exec`

```c
extern void spi_transfer_cs_change_delay_exec(struct spi_message *msg,
                                               struct spi_transfer *xfer);
```

ينفذ الـ delay المناسب بعد تغيير الـ CS. يُجمع `xfer->cs_change_delay` مع `spi->cs_inactive` delay. يُستدعى من الـ core بعد deassert الـ CS.

---

## Group 11: Accessor Helpers

### الغرض
accessors بسيطة تُخفي التفاصيل الداخلية وتُوفّر API نظيف للـ controller وprotocol drivers.

---

### `spi_get_ctldata` / `spi_set_ctldata`

```c
static inline void *spi_get_ctldata(const struct spi_device *spi)
{ return spi->controller_state; }

static inline void spi_set_ctldata(struct spi_device *spi, void *state)
{ spi->controller_state = state; }
```

**يُستخدمان من:** الـ controller driver لحفظ per-device state (مثل DMA channel assignments، CS GPIO state). يُستدعيان من `setup()` و `cleanup()` callbacks.

---

### `spi_get_drvdata` / `spi_set_drvdata`

```c
static inline void spi_set_drvdata(struct spi_device *spi, void *data)
{ dev_set_drvdata(&spi->dev, data); }

static inline void *spi_get_drvdata(const struct spi_device *spi)
{ return dev_get_drvdata(&spi->dev); }
```

يُستخدمان من الـ protocol driver لحفظ driver private data. يُستدعيان من `probe()` (set) و باقي الـ driver functions (get).

---

### `spi_get_chipselect` / `spi_set_chipselect`

```c
static inline u8 spi_get_chipselect(const struct spi_device *spi, u8 idx)
{ return spi->chip_select[idx]; }
```

الـ `spi_device` يدعم حتى `SPI_DEVICE_CS_CNT_MAX` (4) chip selects. الـ idx هو logical CS index، القيمة المُرجعة هي physical CS number.

---

### `spi_get_csgpiod` / `spi_set_csgpiod` / `spi_is_csgpiod`

```c
static inline bool spi_is_csgpiod(struct spi_device *spi)
{
    u8 idx;
    for (idx = 0; idx < spi->num_chipselect; idx++) {
        if (spi_get_csgpiod(spi, idx))
            return true;
    }
    return false;
}
```

**الـ `spi_is_csgpiod`** يتحقق إذا كان أي CS على هذا الـ device مربوط بـ GPIO بدل الـ native CS. مهم لتحديد إذا كان الـ controller يحتاج يستخدم GPIO toggling.

---

### `spi_controller_get_devdata` / `spi_controller_set_devdata`

```c
static inline void *spi_controller_get_devdata(struct spi_controller *ctlr)
{ return dev_get_drvdata(&ctlr->dev); }
```

نفس مبدأ `spi_get_drvdata` لكن للـ controller driver نفسه. يُستخدم من الـ controller driver للوصول لـ hardware-specific state.

---

### `spi_controller_get` / `spi_controller_put`

```c
static inline struct spi_controller *spi_controller_get(struct spi_controller *ctlr)
{
    if (!ctlr || !get_device(&ctlr->dev))
        return NULL;
    return ctlr;
}
```

يزيد/يقلل الـ reference count للـ controller device. لو الـ refcount وصل صفر، الـ controller يُحرَّر. نادر الاستخدام خارج الـ core.

---

### `spi_controller_is_target`

```c
static inline bool spi_controller_is_target(struct spi_controller *ctlr)
{
    return IS_ENABLED(CONFIG_SPI_SLAVE) && ctlr->target;
}
```

يتحقق إذا كان الـ controller في target/slave mode. الـ `IS_ENABLED` يجعل الـ check يُحسم compile-time لو `CONFIG_SPI_SLAVE` مش مفعّل.

---

### `spi_transfer_is_last`

```c
static inline bool spi_transfer_is_last(struct spi_controller *ctlr,
                                         struct spi_transfer *xfer)
{
    return list_is_last(&xfer->transfer_list, &ctlr->cur_msg->transfers);
}
```

يُستخدم من الـ controller driver في `transfer_one` callback لتحديد إذا كان هذا آخر transfer في الـ message — مهم لقرار إبقاء أو رفع الـ CS.

---

## Group 12: Statistics Macros

### الغرض
تحديث per-CPU statistics بشكل آمن على الـ 32-bit systems التي تحتاج seqcount protection.

---

### `SPI_STATISTICS_INCREMENT_FIELD` / `SPI_STATISTICS_ADD_TO_FIELD`

```c
#define SPI_STATISTICS_INCREMENT_FIELD(pcpu_stats, field) \
    do { \
        struct spi_statistics *__lstats; \
        get_cpu(); \
        __lstats = this_cpu_ptr(pcpu_stats); \
        u64_stats_update_begin(&__lstats->syncp); \
        u64_stats_inc(&__lstats->field); \
        u64_stats_update_end(&__lstats->syncp); \
        put_cpu(); \
    } while (0)
```

يُحدّث حقل statistics بشكل atomic. الـ `get_cpu()/put_cpu()` يمنع الـ preemption بين الـ `this_cpu_ptr` والـ الكتابة. الـ `u64_stats_update_begin/end` يحمي من torn reads على 32-bit architectures.

**مثال:**
```c
SPI_STATISTICS_INCREMENT_FIELD(ctlr->pcpu_statistics, messages);
SPI_STATISTICS_ADD_TO_FIELD(ctlr->pcpu_statistics, bytes_tx, xfer->len);
```

---

## Group 13: ACPI & OF Support

### الغرض
البحث عن الـ controllers والـ devices عبر firmware layers.

---

### `of_find_spi_controller_by_node`

```c
extern struct spi_controller *of_find_spi_controller_by_node(struct device_node *node);
```

يبحث في قائمة الـ controllers المسجلة عن controller مرتبط بالـ OF node المحدد. يرفع الـ refcount — لازم تستدعي `spi_controller_put` بعد الاستخدام.

---

### `acpi_spi_find_controller_by_adev`

```c
extern struct spi_controller *acpi_spi_find_controller_by_adev(struct acpi_device *adev);
```

نفسه لكن بالـ ACPI device. متاح فقط لو `CONFIG_ACPI && CONFIG_SPI_MASTER`.

---

### `acpi_spi_device_alloc`

```c
extern struct spi_device *acpi_spi_device_alloc(struct spi_controller *ctlr,
                                                  struct acpi_device *adev,
                                                  int index);
```

يُنشئ `spi_device` من ACPI SPI resource descriptor. الـ `index` يحدد أي resource لو الـ ACPI device يملك أكثر من resource.

**Return:** pointer للـ device أو `ERR_PTR(-errno)`

---

### `acpi_spi_count_resources`

```c
extern int acpi_spi_count_resources(struct acpi_device *adev);
```

يعد عدد SPI resource descriptors في الـ ACPI device. يُستخدم قبل `acpi_spi_device_alloc` لمعرفة كم device لازم يُنشأ.

---

## Group 14: `spi_target_abort`

```c
extern int spi_target_abort(struct spi_device *spi);
```

يُلغي الـ transfer الجاري على controller في target/slave mode. يستدعي `ctlr->target_abort()` callback. مفيد لو الـ host side أوقف الـ transfer بشكل غير متوقع.

**Return:** 0 أو negative errno

**Context:** يُستخدم من protocol driver يعمل كـ SPI slave.

---

## Group 15: `spi_get_device_id` / `spi_get_device_match_data`

```c
extern const struct spi_device_id *spi_get_device_id(const struct spi_device *sdev);
extern const void *spi_get_device_match_data(const struct spi_device *sdev);
```

**الـ `spi_get_device_id`:** يُرجع الـ `spi_device_id` entry اللي طابقت هذا الـ device — يُستخدم من الـ probe لمعرفة أي variant من الـ supported devices يعمل معه.

**الـ `spi_get_device_match_data`:** يُرجع `driver_data` pointer المرتبط بالـ match — شائع مع chip-specific config structs.

**مثال:**
```c
static int my_probe(struct spi_device *spi)
{
    const struct my_chip_data *data = spi_get_device_match_data(spi);
    /* data->fifo_size, data->max_freq, etc. */
}
```

---

## ملاحظة ختامية — تسلسل الـ Typical SPI Transfer

```
Protocol Driver                    SPI Core                    Controller Driver
      │                               │                               │
      │ spi_message_init()            │                               │
      │ spi_message_add_tail()        │                               │
      │                               │                               │
      │──── spi_sync(spi, msg) ──────>│                               │
      │                               │──── ctlr->prepare_message() ─>│
      │                               │<─── return 0 ─────────────────│
      │                               │                               │
      │                               │──── ctlr->transfer_one() ────>│
      │                               │                (DMA / IRQ)    │
      │                               │<─── spi_finalize_current_     │
      │                               │     transfer() ───────────────│
      │                               │                               │
      │                               │──── ctlr->unprepare_message()─>│
      │                               │<─── return ───────────────────│
      │                               │                               │
      │                               │──── spi_finalize_current_     │
      │                               │     message() ────────────────│
      │<─── msg->complete() ──────────│                               │
      │     (or wake up spi_sync)     │                               │
```
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. الـ debugfs Entries

الـ SPI subsystem بيكتب statistics في debugfs. المسار الأساسي:

```
/sys/kernel/debug/spi<N>/
```

لو الـ controller مرقمه `spi0`:

```bash
# اقرأ statistics الـ controller
cat /sys/kernel/debug/spi0/statistics

# اقرأ statistics الـ device المتصل (spi0.0 = bus0, cs0)
cat /sys/kernel/debug/spi0.0/statistics
```

الـ `struct spi_statistics` بتطلع الحقول دي:

| الحقل | المعنى |
|---|---|
| `messages` | عدد الـ spi_message المكتملة |
| `transfers` | عدد الـ spi_transfer المكتملة |
| `errors` | عدد الأخطاء |
| `timedout` | عدد الـ timeouts |
| `spi_sync` | كام مرة استُخدم `spi_sync()` |
| `spi_sync_immediate` | مرات execute في نفس الـ context بدون queue |
| `spi_async` | كام مرة استُخدم `spi_async()` |
| `bytes` | إجمالي البايتات |
| `bytes_tx` | بايتات أُرسلت |
| `bytes_rx` | بايتات استُقبلت |
| `transfer_bytes_histo` | histogram لأحجام الـ transfers (17 bucket) |
| `transfers_split_maxsize` | كام transfer اتقسم بسبب maxsize |

---

#### 2. الـ sysfs Entries

```bash
# معلومات الـ SPI device
ls /sys/bus/spi/devices/spi0.0/

# الـ modalias (اسم الـ driver)
cat /sys/bus/spi/devices/spi0.0/modalias

# الـ driver المرتبط
ls -l /sys/bus/spi/devices/spi0.0/driver

# كل الـ SPI devices الموجودة
ls /sys/bus/spi/devices/

# الـ controller نفسه
ls /sys/class/spi_master/spi0/

# max_speed للـ device
cat /sys/bus/spi/devices/spi0.0/../of_node/spi-max-frequency 2>/dev/null
# أو عبر DT
cat /sys/bus/spi/devices/spi0.0/../of_node/clock-frequency 2>/dev/null

# runtime PM state
cat /sys/bus/spi/devices/spi0.0/power/runtime_status
```

---

#### 3. الـ ftrace

##### تفعيل الـ tracepoints الخاصة بالـ SPI

```bash
# شوف الـ events المتاحة
ls /sys/kernel/tracing/events/spi/

# الـ events المهمة
# spi_controller_busy / spi_controller_idle
# spi_message_start / spi_message_done
# spi_transfer_start / spi_transfer_stop
```

```bash
# تفعيل كل SPI events
echo 1 > /sys/kernel/tracing/events/spi/enable

# أو events محددة
echo 1 > /sys/kernel/tracing/events/spi/spi_message_start/enable
echo 1 > /sys/kernel/tracing/events/spi/spi_message_done/enable
echo 1 > /sys/kernel/tracing/events/spi/spi_transfer_start/enable
echo 1 > /sys/kernel/tracing/events/spi/spi_transfer_stop/enable

# ابدأ الـ tracing
echo 1 > /sys/kernel/tracing/tracing_on

# شغل الـ application اللي بتختبرها

# اقرأ النتيجة
cat /sys/kernel/tracing/trace
```

##### مثال output من ftrace

```
# TASK-PID   CPU  TIMESTAMP  FUNCTION
spi-flash-1234  [001]  1234.567: spi_message_start: spi0.0
spi-flash-1234  [001]  1234.568: spi_transfer_start: spi0.0 len=4 speed=10000000
spi-flash-1234  [001]  1234.571: spi_transfer_stop:  spi0.0 len=4 effective_speed=9600000
spi-flash-1234  [001]  1234.572: spi_message_done:  spi0.0 status=0
```

- الفرق بين `speed` و `effective_speed` يكشف لو الـ controller شاغل clock divider مختلف.
- `status != 0` في `spi_message_done` = خطأ.

---

#### 4. الـ printk / Dynamic Debug

```bash
# تفعيل dynamic debug لكل SPI core
echo 'module spi_core +p' > /sys/kernel/debug/dynamic_debug/control

# أو لـ driver معين (مثلاً spi-bcm2835)
echo 'module spi_bcm2835 +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل مع file محدد
echo 'file drivers/spi/spi.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل مع function محددة
echo 'func spi_transfer_one_message +p' > /sys/kernel/debug/dynamic_debug/control

# إلغاء التفعيل
echo 'module spi_core -p' > /sys/kernel/debug/dynamic_debug/control
```

لو محتاج تفعّل من command line الـ kernel:

```bash
# في /etc/default/grub أضف:
GRUB_CMDLINE_LINUX="dyndbg='module spi_core +p'"
```

---

#### 5. الـ Kernel Config Options للـ Debugging

| Config | الوظيفة |
|---|---|
| `CONFIG_SPI_DEBUG` | يفعّل dev_dbg() messages في SPI core |
| `CONFIG_SPI_LOOPBACK_TEST` | driver للاختبار بدون hardware حقيقي |
| `CONFIG_SPI_SLAVE` | يفعّل target/slave mode |
| `CONFIG_DEBUG_SPI` | (بعض versions) extra validation |
| `CONFIG_DYNAMIC_DEBUG` | يتيح dynamic_debug controls |
| `CONFIG_TRACING` | يتيح ftrace events |
| `CONFIG_FUNCTION_TRACER` | function-level tracing |
| `CONFIG_LOCKDEP` | يكشف deadlocks في bus_lock_mutex |
| `CONFIG_DEBUG_SPINLOCK` | يكشف مشاكل queue_lock |
| `CONFIG_PROVE_LOCKING` | يثبت correctness الـ locking |
| `CONFIG_DMA_API_DEBUG` | يكشف DMA mapping errors |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'CONFIG_SPI|CONFIG_DEBUG'
# أو
cat /boot/config-$(uname -r) | grep -E 'CONFIG_SPI|CONFIG_DEBUG'
```

---

#### 6. أدوات خاصة بالـ SPI Subsystem

##### spidev (userspace interface)

```bash
# لو مفعّل CONFIG_SPI_SPIDEV
ls /dev/spidev*

# تثبيت أدوات الاختبار
apt install spi-tools   # Debian/Ubuntu

# اختبار loopback (MOSI → MISO مربوطين)
spi-pipe -d /dev/spidev0.0 -s 1000000 -b 8 < /dev/urandom | head -c 64 | xxd

# أو باستخدام spidev_test من kernel sources
# tools/spi/spidev_test.c
./spidev_test -D /dev/spidev0.0 -s 500000 -v
```

##### spi-tools

```bash
# عرض معلومات الـ device
spi-config -d /dev/spidev0.0 --mode
spi-config -d /dev/spidev0.0 --speed
spi-config -d /dev/spidev0.0 --bits

# تغيير الإعدادات
spi-config -d /dev/spidev0.0 -m 0 -s 1000000 -b 8
```

##### فحص الـ message queue

```bash
# شوف الـ kthread الخاص بالـ SPI pump
ps aux | grep spi

# مثال output:
# root  123  0.0  0.0  0  0  ?  S  00:00  kworker/u4:0+spi0
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة في dmesg | المعنى | الحل |
|---|---|---|
| `spi_master spi0: failed to transfer one message` | فشل في إرسال message | تحقق من CS وال clock |
| `spi0.0: error -110 (ETIMEDOUT)` | timeout انتهت المهلة | الـ hardware بطيء أو مش شغال، افحص SCK |
| `spi0.0: error -22 (EINVAL)` | إعدادات غلط (mode/speed/bpw) | تحقق من `bits_per_word_mask` و `mode_bits` |
| `spi0.0: error -16 (EBUSY)` | الـ bus محجوز | deadlock في `bus_lock_flag` |
| `spi0: DMA init failed` | فشل تهيئة DMA | تحقق من DMA channels والـ alignment |
| `spi-bcm2835: could not get clk` | مفيش clock | تحقق من DT clock-names |
| `spi0: GPIO chip select not valid` | CS GPIO مش صح | تحقق من DT cs-gpios property |
| `spi_message submitted while idle` | bug في الـ driver | transfer submitted بعد unregister |
| `SPI transfer timed out` | `xfer_completion` expired | افحص interrupt handler |
| `spi0.0: bits_per_word 12 not supported` | BPW غير مدعوم | تحقق من `bits_per_word_mask` في الـ controller |
| `spi0: SPI transfer with invalid DMA address` | buffer مش DMA-safe | استخدم DMA-safe buffer (kmalloc بدل stack) |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في transfer_one callback — تحقق إن الـ len منطقي */
int my_spi_transfer_one(struct spi_controller *ctlr,
                        struct spi_device *spi,
                        struct spi_transfer *xfer)
{
    WARN_ON(!xfer->tx_buf && !xfer->rx_buf); /* لازم يكون في buffer */
    WARN_ON(xfer->len == 0);                 /* لازم يكون في data */

    if (xfer->speed_hz > ctlr->max_speed_hz) {
        dev_err(&spi->dev, "speed %u > max %u\n",
                xfer->speed_hz, ctlr->max_speed_hz);
        dump_stack(); /* اطبع call stack */
        return -EINVAL;
    }
    /* ... */
}

/* في complete callback — تحقق من status */
static void my_complete(void *context)
{
    struct spi_message *msg = context;
    WARN_ON(msg->status && msg->actual_length > 0); /* partial transfer مع error */
}

/* في probe — تحقق من الـ controller state */
static int my_probe(struct spi_device *spi)
{
    WARN_ON(!spi->controller);
    WARN_ON(spi->max_speed_hz == 0);
    /* ... */
}
```

---

### Hardware Level

#### 1. التحقق أن حالة الـ Hardware تطابق حالة الـ Kernel

```bash
# تحقق إن الـ controller registered
ls /sys/class/spi_master/

# تحقق من الـ bus_num
cat /sys/class/spi_master/spi0/device/of_node/reg  2>/dev/null

# تحقق من عدد الـ chipselects
cat /sys/class/spi_master/spi0/device/of_node/num-cs 2>/dev/null

# تحقق من السرعة القصوى
cat /sys/class/spi_master/spi0/device/of_node/spi-max-frequency 2>/dev/null

# compare مع الـ datasheet
# لو SPI flash مثلاً بتقرأ ID:
echo -ne '\x9f\x00\x00\x00' | spi-pipe -d /dev/spidev0.0 -s 1000000 | xxd
# الـ output لازم يطابق JEDEC ID في الـ datasheet
```

---

#### 2. Register Dump Techniques

لو الـ SPI controller في Raspberry Pi (BCM2835) على عنوان `0x3F204000`:

```bash
# تثبيت devmem2
apt install devmem2

# اقرأ SPI0 CS register (offset 0x00)
devmem2 0x3F204000 w

# اقرأ SPI0 FIFO register (offset 0x04)
devmem2 0x3F204004 w

# اقرأ SPI0 CLK register (offset 0x08)
devmem2 0x3F204008 w

# اقرأ SPI0 DLEN register (offset 0x0C)
devmem2 0x3F20400C w
```

للـ i.MX6 SPI (ECSPI1 على `0x02008000`):

```bash
devmem2 0x02008000 w   # ECSPI_RXDATA
devmem2 0x02008004 w   # ECSPI_TXDATA
devmem2 0x02008008 w   # ECSPI_CONREG
devmem2 0x0200800C w   # ECSPI_CONFIGREG
devmem2 0x02008010 w   # ECSPI_INTREG
devmem2 0x02008018 w   # ECSPI_STATREG
```

```bash
# بديل: io utility
io -4 0x3F204000   # قراءة 32-bit register
io -4 -r 0x3F204000 0x10  # قراءة 16 bytes من العنوان

# أو عبر /dev/mem مباشرة (بحذر شديد)
dd if=/dev/mem bs=4 count=8 skip=$((0x3F204000/4)) 2>/dev/null | xxd
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

##### الإشارات اللي لازم تراقبها

```
SPI Signals:
  SCLK  ─────╮  ╭─╮ ╭─╮ ╭─╮ ╭─╮ ╭─╮ ╭─╮ ╭─╮ ╭───
              ╰──╯ ╰─╯ ╰─╯ ╰─╯ ╰─╯ ╰─╯ ╰─╯ ╰─╯
  CS    ───╮                                    ╭───
           ╰────────────────────────────────────╯
  MOSI  ───X──────────────────────────────────────
  MISO  ──────────────────────────────────────────
```

**إعدادات Logic Analyzer:**

| الإعداد | القيمة المقترحة |
|---|---|
| Sample rate | 10× أو أكثر من SCLK |
| Trigger | CS falling edge |
| Protocol decoder | SPI (اختار CPOL/CPHA صح) |
| Voltage threshold | 1.8V أو 3.3V حسب الـ board |

**أشياء لازم تتحقق منها:**

```
1. CS setup time  = وقت بين CS assert وأول SCLK edge
   → لازم يطابق spi_device.cs_setup

2. CS hold time   = وقت بين آخر SCLK edge و CS deassert
   → لازم يطابق spi_device.cs_hold

3. inter-word delay = الوقت بين كل word
   → لازم يطابق spi_device.word_delay

4. SCLK frequency = قيسها بالـ oscilloscope
   → لازم تطابق effective_speed_hz في spi_transfer
```

**مثال sigrok/PulseView:**

```bash
# capture بـ sigrok-cli
sigrok-cli -d fx2lafw --config samplerate=24m \
  --channels D0=SCLK,D1=MOSI,D2=MISO,D3=CS \
  --trigger CS=f \
  --time 100ms \
  -P spi:clk=SCLK:mosi=MOSI:miso=MISO:cs=CS:cpol=0:cpha=0 \
  -o capture.sr

# analyze
sigrok-cli -i capture.sr -P spi --protocol-decoder-annotations spi
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في dmesg

| المشكلة | ما يظهر في dmesg / ftrace | الحل |
|---|---|---|
| **SCLK مش بيوصل للـ device** | `ETIMEDOUT` متكرر، `timedout` counter يزيد | افحص الـ PCB trace والـ clock enable register |
| **CS مش بيتفعّل (GPIO مش configured)** | `spi0.0: Failed to get CS GPIO` | تحقق من `cs-gpios` في DT |
| **MISO float (مش connected)** | البيانات المستقبلة عشوائية، CRC failures | افحص الـ pull-up/down على MISO |
| **Voltage mismatch (3.3V device على 1.8V bus)** | بيانات غلط أو لا response | افحص الـ level shifter |
| **Noise على SCLK (ringing)** | intermittent errors، `errors` counter يزيد | أضف series resistor 33Ω على SCLK |
| **غلط في CPOL/CPHA** | أول byte صح والباقي غلط | غيّر `SPI_MODE_x` في الـ DT أو driver |
| **DMA buffer على stack** | `spi0: invalid DMA address` | استخدم `kmalloc` بدل stack للـ buffers |
| **CS مش بيرجع HIGH** | الـ device مش بيرد على transfers تانية | افحص `cs_change` في الـ transfer |

---

#### 5. Device Tree Debugging

```bash
# اطبع الـ DT المحمّل فعلياً للـ SPI controller
cat /sys/firmware/devicetree/base/spi@7e204000/*/name 2>/dev/null

# أو بشكل أوضح
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 30 "spi@"

# تحقق من الـ compatible string
cat /sys/firmware/devicetree/base/spi@7e204000/compatible | tr '\0' '\n'

# تحقق من الـ CS pins
cat /sys/firmware/devicetree/base/spi@7e204000/cs-gpios 2>/dev/null | xxd

# تحقق من الـ clocks
cat /sys/firmware/devicetree/base/spi@7e204000/clocks 2>/dev/null | xxd

# مقارنة DT source مع الـ runtime
# lو عندك .dts الأصلي:
dtc -I dts -O dtb -o /tmp/check.dtb my_board.dts
fdtdump /tmp/check.dtb | grep -A 20 "spi"

# تحقق من spi-max-frequency
grep -r spi-max-frequency /sys/firmware/devicetree/base/ 2>/dev/null | \
  while read line; do
    echo "$(echo $line | awk -F: '{print $1}') → $(cat ${line%%:*} | xxd -p)"
  done
```

**الحقول اللازمة في DT لكل SPI device:**

```dts
/* مثال DT صح */
&spi0 {
    status = "okay";
    cs-gpios = <&gpio 8 GPIO_ACTIVE_LOW>;  /* CS via GPIO */

    flash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;                          /* chip select 0 */
        spi-max-frequency = <50000000>;     /* 50 MHz */
        spi-cpol;                           /* CPOL=1 */
        spi-cpha;                           /* CPHA=1 → MODE 3 */
        /* spi-tx-bus-width = <1>; default */
        /* spi-rx-bus-width = <1>; default */
    };
};
```

---

### Practical Commands

#### الـ Cheat Sheet الكامل

```bash
# ========== DISCOVERY ==========

# كل الـ SPI controllers
ls /sys/class/spi_master/

# كل الـ SPI devices
ls /sys/bus/spi/devices/

# driver مرتبط بـ device
ls -l /sys/bus/spi/devices/spi0.0/driver

# ========== STATISTICS ==========

# statistics الـ controller
cat /sys/kernel/debug/spi0/statistics

# statistics الـ device
cat /sys/kernel/debug/spi0.0/statistics

# ========== TRACING ==========

# تفعيل كل SPI events
echo 1 > /sys/kernel/tracing/events/spi/enable && \
echo 1 > /sys/kernel/tracing/tracing_on

# قرأ الـ trace
cat /sys/kernel/tracing/trace | grep spi

# إيقاف الـ tracing
echo 0 > /sys/kernel/tracing/tracing_on && \
echo 0 > /sys/kernel/tracing/events/spi/enable

# ========== DYNAMIC DEBUG ==========

# تفعيل debug لـ SPI core
echo 'module spi_core +pfml' > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug لـ driver معين
echo 'module spi_bcm2835 +pfml' > /sys/kernel/debug/dynamic_debug/control

# إلغاء كل debug
echo 'module spi_core -p' > /sys/kernel/debug/dynamic_debug/control

# ========== DMESG FILTERING ==========

# رسائل SPI فقط
dmesg | grep -i spi

# errors فقط
dmesg --level=err,warn | grep -i spi

# real-time monitoring
dmesg -w | grep -i spi

# ========== SPIDEV TESTING ==========

# اختبار loopback (MOSI ↔ MISO مربوطين)
./spidev_test -D /dev/spidev0.0 -s 500000 -v -l

# إرسال bytes محددة
python3 -c "
import spidev
spi = spidev.SpiDev()
spi.open(0, 0)
spi.max_speed_hz = 1000000
spi.mode = 0
response = spi.xfer2([0x9F, 0x00, 0x00, 0x00])
print('JEDEC ID:', [hex(b) for b in response])
spi.close()
"

# ========== REGISTER DUMP ==========

# BCM2835 SPI0 registers (RPi)
for offset in 0x00 0x04 0x08 0x0C 0x10 0x14; do
    echo "Offset $offset: $(devmem2 $((0x3F204000 + offset)) w 2>/dev/null | grep 'Read' | awk '{print $NF}')"
done

# ========== DEVICE TREE ==========

# اطبع DT الكامل بشكل مقروء
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -B2 -A 40 "spi@"

# تحقق من compatible
find /sys/firmware/devicetree/base -name "compatible" -exec sh -c \
  'path=$1; content=$(tr "\0" "\n" < "$path"); echo "$path: $content"' _ {} \; \
  2>/dev/null | grep -i spi

# ========== KERNEL OOPS ANALYSIS ==========

# لو عندك oops في SPI driver، ابحث عن الـ address
addr2line -e vmlinux <address_from_oops>

# أو
scripts/decode_stacktrace.sh vmlinux < /tmp/oops.txt
```

#### مثال على تفسير Statistics Output

```bash
$ cat /sys/kernel/debug/spi0.0/statistics
# Output مثال:
messages:           1024
transfers:          2048
errors:                3
timedout:              1
spi_sync:           1020
spi_sync_immediate:  800    ← 800/1020 = 78% executed directly, good!
spi_async:             4
bytes:             65536
bytes_tx:          32768
bytes_rx:          32768
transfer_bytes_histo:
  0:     0          ← transfers بـ 0 bytes (مفيش، كويس)
  1:   128          ← transfers بـ 1 byte
  2:   256          ← transfers بـ 2-3 bytes
  ...
  16:   12          ← transfers بـ 32KB+ (كبيرة)
transfers_split_maxsize: 5   ← 5 transfers اتقسمت بسبب maxsize limit
```

**التفسير:**
- **`errors: 3`** مع **`timedout: 1`** → في مشكلة hardware متقطعة، افحص الـ connections.
- **`spi_sync_immediate: 800`** من أصل `1020` → الـ fast path شغال كويس.
- **`transfers_split_maxsize: 5`** → بعض الـ transfers أكبر من `max_transfer_size()`، الـ core بيقسمها أوتوماتيك.
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — بيانات مش بتتزامن صح مع ADC خارجي

#### العنوان
**SPI mode mismatch** بيخلي قراءات الـ ADC على industrial gateway غلط تماماً

#### السياق
بنبني industrial gateway بيقيس analog signals من sensors صناعية عن طريق ADC خارجي (`ADS8688`) موصّل على `spi1` في RK3562. الـ gateway بيرسل البيانات لـ SCADA system. الـ production team بتشتكي إن القراءات عشوائية ومش مترابطة بالمنطق.

#### المشكلة
الـ ADC بيرجّع قيم غير منطقية — أحياناً `0xFFFF`، أحياناً بياناً مقلوب. الـ driver بيشتغل ومفيش error logs.

#### التحليل
الـ `ADS8688` بيستخدم **SPI Mode 1** (CPOL=0, CPHA=1). في الـ DT، المبرمج كتب:

```c
// spi_device.mode يتحدد من DT أو board_info
// الفيلد ده بيتحكم في CPOL/CPHA
u32 mode;
```

لما بنعمل `spi_setup(spi)` الـ core بيبعت الـ mode للـ controller driver. لو الـ mode غلط، الـ SCK polarity/phase هتبقى مختلفة عن اللي الـ ADC بيتوقعه، والبيانات هتتشال على edges غلط.

```bash
# نشوف الـ mode المسجّل للـ device
cat /sys/bus/spi/devices/spi1.0/modalias
# نشوف statistics هل في errors
cat /sys/bus/spi/devices/spi1.0/../../../statistics/errors
```

**الـ `spi_statistics`** في الـ header:
```c
struct spi_statistics {
    u64_stats_t errors;      // عدد الـ errors
    u64_stats_t timedout;    // عدد الـ timeouts
    u64_stats_t bytes_rx;    // bytes اتستقبلت
    u64_stats_t bytes_tx;    // bytes اتبعتت
    ...
};
```
الـ errors = 0 لأن الـ transfer نفسه نجح، بس البيانات غلط بسبب mode خاطئ.

#### الحل
```dts
/* DT node تاعت الـ ADC — قبل التصحيح */
ads8688: adc@0 {
    compatible = "ti,ads8688";
    reg = <0>;
    spi-max-frequency = <10000000>;
    /* spi-cpha كانت ناقصة! */
};

/* بعد التصحيح */
ads8688: adc@0 {
    compatible = "ti,ads8688";
    reg = <0>;
    spi-max-frequency = <10000000>;
    spi-cpha;  /* يضيف SPI_CPHA → Mode 1 */
};
```

بعد الـ fix:
```bash
# verify بعد reboot
dmesg | grep spi
# لازم يظهر: spi1.0: setup mode 1, 8 bits/w, 10000000 Hz max
```

#### الدرس المستفاد
**الـ `spi_device.mode`** هو أول حاجة لازم تتحقق فيها. الـ `SPI_CPOL` و`SPI_CPHA` bits لو غلطوا، الـ transfer هيكمل بدون errors لكن البيانات هتبقى nonsense. دايماً راجع datasheet الـ chip وتأكد من الـ DT bindings (`spi-cpol`, `spi-cpha`).

---

### السيناريو 2: Android TV Box على Allwinner H616 — Flash بطيء جداً والـ boot بياخد وقت طويل

#### العنوان
الـ `max_speed_hz` على SPI NOR Flash مش بيتطبق صح — الـ boot وقته ضعف المتوقع

#### السياق
Android TV box بيستخدم Allwinner H616، الـ SPI NOR Flash (`W25Q128`) هو مخزن الـ U-Boot والـ kernel. الـ datasheet بيقول الـ Flash يقدر يشتغل على 133 MHz، لكن الـ boot بياخد 8 ثواني بدل 3.5.

#### المشكلة
الـ Flash بيشتغل على 24 MHz بدل الـ maximum المسموح. الـ `spi_transfer.speed_hz` بيتجاهل والـ `spi_device.max_speed_hz` بيتاخد.

#### التحليل
الـ header بيوضح الـ priority:

```c
struct spi_transfer {
    /* لو speed_hz = 0 هنا، الـ core بياخد من spi_device */
    u32 speed_hz;
    u32 effective_speed_hz; /* الـ speed الفعلية اللي اتستخدمت */
    ...
};

struct spi_device {
    u32 max_speed_hz; /* الـ ceiling — الـ controller مش هيعدّيه */
    ...
};
```

الـ Allwinner H616 SPI controller بيحدد `max_speed_hz` في الـ driver. لو الـ DT فيه قيمة أقل، الـ `spi_setup()` بيطبق `min(device_max, controller_max)`.

```bash
# نشوف الـ effective speed
cat /sys/kernel/debug/spi1/spi1.0/statistics/
# نشوف الـ DT speed
dtc -I fs /sys/firmware/devicetree/base | grep -A5 "flash@0"
```

لقينا في الـ DT:
```dts
flash@0 {
    spi-max-frequency = <24000000>; /* 24 MHz فقط! */
};
```

بالإضافة لكده، الـ `transfers_split_maxsize` في الـ statistics كانت عالية — يعني الـ controller بياخد limit على transfer size وبيقسمها:

```c
u64_stats_t transfers_split_maxsize; /* كم مرة اتقسم الـ transfer */
```

#### الحل
```dts
/* H616 SPI controller يدعم حتى 100 MHz مع هذا الـ NOR */
flash@0 {
    compatible = "winbond,w25q128", "jedec,spi-nor";
    reg = <0>;
    spi-max-frequency = <100000000>; /* 100 MHz — أمان مع board traces */
    spi-rx-bus-width = <4>; /* Quad SPI للقراءة */
    spi-tx-bus-width = <1>;
};
```

```bash
# بعد الـ fix — نقيس boot time
time systemctl isolate multi-user.target
# نشوف effective_speed_hz
cat /sys/kernel/debug/spi-nor/spi1.0/params
```

#### الدرس المستفاد
**الـ `max_speed_hz`** في الـ `spi_device` هو الـ ceiling مش الـ target. لو الـ DT قيمته منخفضة، الـ core هيكرّم القيمة دي حتى لو الـ controller يقدر أسرع. راجع `spi_transfer.effective_speed_hz` بعد كل transfer لتأكيد الـ speed الفعلية.

---

### السيناريو 3: IoT Sensor Board على STM32MP1 — الـ IMU بيديّ corrupt data بعد 10 دقائق

#### العنوان
**`word_delay`** مش متظبطة صح على IMU (`ICM-42688`) فوق STM32MP1 — بيحصل data corruption متأخر

#### السياق
IoT sensor node بيقيس vibration باستخدام ICM-42688 IMU موصّل على SPI2 في STM32MP157. الـ node بيرسل data لـ edge server. بعد 10 دقائق تشغيل مستمر، بيبدأ يظهر data outliers — readings خارج النطاق المادي المعقول.

#### المشكلة
الـ ICM-42688 datasheet بيقول لازم يكون في minimum delay بين الـ SPI words (inter-word gap). من غير الـ delay ده، الـ internal state machine للـ IMU بتتعطل.

#### التحليل
الـ `spi_device` فيه `word_delay`:

```c
struct spi_device {
    struct spi_delay word_delay; /* inter-word delay — بين الـ words في نفس الـ transfer */
    ...
};

struct spi_delay {
#define SPI_DELAY_UNIT_USECS  0  /* microseconds */
#define SPI_DELAY_UNIT_NSECS  1  /* nanoseconds */
#define SPI_DELAY_UNIT_SCK    2  /* clock cycles */
    u16 value;
    u8  unit;
};
```

كمان `spi_transfer` فيه `word_delay` للـ per-transfer override:

```c
struct spi_transfer {
    struct spi_delay word_delay; /* override للـ spi_device.word_delay */
    ...
};
```

الـ STM32MP1 SPI controller بيدعم hardware-controlled inter-word delay. لما بيكون فيه burst transfers من 512 bytes، الـ SPI master بيبعتهم بسرعة. أول 10 دقائق تمام لأن الـ IMU buffer ما اتملاش، بعد كده الـ problem بيظهر.

```bash
# نشوف statistics
cat /sys/class/spi_master/spi2/statistics/errors
cat /sys/class/spi_master/spi2/statistics/bytes_rx

# نستخدم logic analyzer لنشوف الـ waveform
# أو نشوف الـ delay عبر spi debugfs
ls /sys/kernel/debug/spi2/
```

#### الحل
في الـ driver's `probe()`:

```c
static int icm42688_probe(struct spi_device *spi)
{
    /* ICM-42688 يحتاج >= 40 ns inter-word delay عند 10 MHz */
    spi->word_delay.value = 50; /* 50 ns */
    spi->word_delay.unit  = SPI_DELAY_UNIT_NSECS;

    /* apply الـ setup */
    ret = spi_setup(spi);
    if (ret)
        return ret;

    /* ... باقي الـ probe */
}
```

أو من DT:
```dts
icm42688: imu@0 {
    compatible = "invensense,icm42688p";
    reg = <0>;
    spi-max-frequency = <10000000>;
    spi-cs-setup-delay-ns = <10>;
    /* word_delay بيتحط programmatically في الـ driver */
};
```

#### الدرس المستفاد
الـ **`word_delay`** و**`cs_setup`** و**`cs_hold`** في `spi_device` مش decoration — هي timing requirements حقيقية. المشاكل اللي بتظهر بعد وقت (thermal، load-dependent) غالباً من timing violations. راجع `spi_delay_to_ns()` و`spi_delay_exec()` في الـ header.

---

### السيناريو 4: Automotive ECU على i.MX8 — الـ SPI bus بيتعلق لما شغّلنا SPI و CAN مع بعض

#### العنوان
**`bus_lock_flag`** و race condition في الـ `spi_controller` بيخلي الـ ECU يتعلق عند concurrent access

#### السياق
Automotive ECU على NXP i.MX8QM. الـ ECU بيشغّل إتنين drivers بيستخدموا نفس SPI bus: driver لـ external CAN transceiver (`MCP2518FD`) وdriver لـ safety watchdog chip (`TPS1HB08`). الـ watchdog لازم يتبعتله heartbeat كل 20ms. المشكلة: system بيتعلق (kernel hang) كل 2-3 دقائق.

#### المشكلة
الـ watchdog driver بيستخدم `spi_bus_lock()` لضمان exclusive access، لكن بيحصل deadlock مع الـ CAN driver اللي بيعمل `spi_sync()` من interrupt context.

#### التحليل
الـ header بيحدد الـ locking structure:

```c
struct spi_controller {
    spinlock_t   bus_lock_spinlock; /* للـ atomic operations */
    struct mutex bus_lock_mutex;    /* للـ sleeping lock */
    bool         bus_lock_flag;     /* الـ bus محجوز؟ */
    ...
};
```

الـ `spi_bus_lock()` بيأخد `bus_lock_mutex` — ده sleeping lock. لو الـ CAN driver بيعمل `spi_sync()` من interrupt context، والـ mutex محجوز، هيحاول ينام في interrupt context = **kernel BUG**.

```bash
# نشوف الـ kernel trace
dmesg | grep -E "BUG|sleeping|sched"
# أو نستخدم ftrace
echo 1 > /sys/kernel/debug/tracing/events/spi/enable
cat /sys/kernel/debug/tracing/trace
```

الـ `spi_statistics` هتكشف:
```c
u64_stats_t spi_async;    /* عدد الـ async calls */
u64_stats_t spi_sync;     /* عدد الـ sync calls */
```

لو الـ CAN driver بيعمل `spi_sync` من ISR، الـ `spi_sync` counter هيكون مرتفع والـ system سيكون unstable.

#### الحل
الـ CAN driver لازم يستخدم `spi_async()` مع completion callback بدل `spi_sync()`:

```c
/* غلط — من interrupt context */
static irqreturn_t can_irq_handler(int irq, void *data)
{
    /* spi_sync() ممنوعة هنا */
    spi_sync(priv->spi, &priv->msg); /* DEADLOCK RISK */
    return IRQ_HANDLED;
}

/* صح — نستخدم workqueue */
static irqreturn_t can_irq_handler(int irq, void *data)
{
    schedule_work(&priv->spi_work); /* defer للـ process context */
    return IRQ_HANDLED;
}

static void can_spi_work(struct work_struct *work)
{
    /* هنا تمام — process context */
    spi_sync(priv->spi, &priv->msg);
}
```

للـ watchdog driver لو محتاج exclusive access:
```c
/* الـ bus_lock لـ atomic sequences فقط */
spi_bus_lock(spi->controller);
spi_sync_locked(spi, &msg1);
spi_sync_locked(spi, &msg2);
spi_bus_unlock(spi->controller);
```

#### الدرس المستفاد
**`spi_sync()`** بتنام — ممنوعة من interrupt/atomic context. الـ `bus_lock_flag` و`bus_lock_mutex` في `spi_controller` هما للـ exclusive multi-message sequences، مش للـ protection من interrupt context. استخدم `spi_async()` أو defer لـ workqueue لأي SPI operation من ISR.

---

### السيناريو 5: Custom Board Bring-up على AM62x — الـ SPI device مش بيظهر في `/dev/spidev*`

#### العنوان
الـ **`cs_gpiod`** و`use_gpio_descriptors` مش متظبطين — الـ SPI CS مش بيشتغل على AM62x custom board

#### السياق
Custom industrial board بيستخدم TI AM62x (BeaglePlay-based). الـ board بيضم EEPROM (`AT25M02`) على SPI0. الـ hardware team وصّل الـ CS line على GPIO بدل الـ native CS للـ controller لأسباب layout. بعد bring-up، الـ EEPROM مش بيظهر وفيه error: `spi_device: no CS GPIO`.

#### المشكلة
الـ SPI controller على AM62x بيستخدم `use_gpio_descriptors = true`، والـ DT لازم يعرّف الـ CS GPIO بطريقة معينة. البرنامج كتب الـ DT بطريقة قديمة.

#### التحليل
الـ `spi_controller` فيه:

```c
struct spi_controller {
    struct gpio_desc **cs_gpiods;      /* array of GPIO CS descriptors */
    bool use_gpio_descriptors;         /* الـ core يقرأ GPIOs من DT تلقائياً */
    s8   unused_native_cs;             /* أول native CS مش مستخدم */
    s8   max_native_cs;                /* الـ ceiling للـ native CS numbers */
    ...
};
```

والـ `spi_device`:
```c
struct spi_device {
    struct gpio_desc *cs_gpiod[SPI_DEVICE_CS_CNT_MAX]; /* CS GPIO per device */
    u8 num_chipselect;
    ...
};
```

الـ helper functions:
```c
static inline struct gpio_desc *spi_get_csgpiod(const struct spi_device *spi, u8 idx)
{
    return spi->cs_gpiod[idx];
}

static inline bool spi_is_csgpiod(struct spi_device *spi)
{
    u8 idx;
    for (idx = 0; idx < spi->num_chipselect; idx++) {
        if (spi_get_csgpiod(spi, idx))
            return true;
    }
    return false; /* لو false = مفيش GPIO CS */
}
```

الـ AM62x SPI driver بيسمّي الـ GPIO CS property `cs-gpios` في الـ DT. المبرمج استخدم `gpio-cs` (format قديم).

```bash
# نشوف الـ probe error
dmesg | grep -i "spi0\|cs gpio\|chipselect"
# output:
# spi-omap2-mcspi 20100000.spi: unable to get CS GPIO

# نشوف الـ DT binding
dtc -I fs /sys/firmware/devicetree/base > /tmp/current.dts
grep -A20 "spi@20100000" /tmp/current.dts
```

#### الحل
```dts
/* DT قبل التصحيح — format قديم */
&spi0 {
    status = "okay";
    #address-cells = <1>;
    #size-cells = <0>;
    gpio-cs = <&gpio0 10 GPIO_ACTIVE_LOW>; /* خطأ */

    eeprom@0 {
        compatible = "atmel,at25";
        reg = <0>;
        spi-max-frequency = <5000000>;
    };
};

/* DT بعد التصحيح */
&spi0 {
    status = "okay";
    #address-cells = <1>;
    #size-cells = <0>;
    cs-gpios = <&gpio0 10 GPIO_ACTIVE_LOW>; /* صح */

    eeprom@0 {
        compatible = "atmel,at25";
        reg = <0>;
        spi-max-frequency = <5000000>;
        spi-max-cs-setup-ns = <100>;
    };
};
```

```bash
# بعد الـ fix، verify
ls /sys/bus/spi/devices/
# يظهر: spi0.0

# verify الـ CS GPIO اتأخد
cat /sys/bus/spi/devices/spi0.0/driver_override

# test بسيط
echo "at25" > /sys/bus/spi/devices/spi0.0/modalias
```

لو محتاج تعمل GPIO CS programmatically:
```c
/* في الـ controller driver probe() */
ctlr->use_gpio_descriptors = true; /* الـ core هيجيب الـ GPIOs تلقائياً */
/* الـ core هيملّي ctlr->cs_gpiods[] وكمان spi_device->cs_gpiod[] */
```

#### الدرس المستفاد
لما بتستخدم **GPIO CS** بدل native CS، الـ `spi_controller.use_gpio_descriptors` لازم يكون `true` والـ DT لازم يستخدم `cs-gpios` (مش `gpio-cs` أو أي format تاني). الـ SPI core بيستخدم `spi_get_csgpiod()` و`spi_is_csgpiod()` عشان يعرف يتعامل مع الـ CS. راجع الـ `spi_set_csgpiod()` لو محتاج تعمل manual assignment في الـ driver.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

| المقال | الوصف |
|--------|-------|
| [SPI core](https://lwn.net/Articles/138037/) | أول patch رسمي لـ SPI core في الـ kernel — نقطة البداية التاريخية |
| [SPI core -- revisited](https://lwn.net/Articles/141186/) | مراجعة تانية للـ SPI framework قبل الـ merge النهائي |
| [simple SPI framework](https://lwn.net/Articles/154425/) | الـ patch اللي وصف نموذج قايمة الـ messages الـ async — الأساس اللي بني عليه الـ subsystem |
| [SPI core refresh](https://lwn.net/Articles/162268/) | تحديث شامل للـ SPI core بعد التجارب الأولى |
| [spi](https://lwn.net/Articles/146581/) | نقاش الـ mailing list حول الـ SPI API design |
| [Add SPI over GPIO driver](https://lwn.net/Articles/290068/) | إضافة `spi-gpio` driver — مهم لفهم كيف الـ SPI بيتعمل على bitbanging |
| [spi: Add slave mode support](https://lwn.net/Articles/723440/) | إضافة دعم **SPI slave mode** — تحول مهم في الـ subsystem |
| [Introduction to SPI NAND framework](https://lwn.net/Articles/719733/) | الـ SPI NAND framework — layer فوق الـ SPI core لـ NAND flash |

---

### التوثيق الرسمي في الـ kernel

#### ملفات `Documentation/spi/`

```
Documentation/spi/
├── spi-summary.rst          # نظرة عامة على الـ subsystem — ابدأ منه
├── spi-lm70llp.rst          # مثال عملي على driver حقيقي
├── pxa2xx.rst               # مثال على controller-specific docs
└── spi-sc18is602.rst        # مثال على SPI-to-I2C bridge driver
```

الـ online version:
- [Serial Peripheral Interface (SPI) — The Linux Kernel documentation](https://www.kernel.org/doc/html/v4.14/driver-api/spi.html)
- [SPI userspace API (spidev)](https://static.lwn.net/kerneldoc/spi/spidev.html)
- [Driver API: SPI](https://static.lwn.net/kerneldoc/driver-api/spi.html)

#### الملف الأساسي

```
include/linux/spi/spi.h      # الـ header الرئيسي — كل الـ structs والـ APIs
drivers/spi/spi.c            # الـ SPI core implementation
drivers/spi/spi-bitbang.c    # الـ bitbang helper للـ GPIO-based controllers
```

---

### Kernel Commits مهمة

| الـ Commit | الوصف |
|-----------|-------|
| [`b885244`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b885244eb1c4be02e9f3ca7c5fe4c3c0c48419db) | الـ commit الأصلي لـ SPI subsystem من David Brownell |
| [SPI slave mode initial commit](https://lore.kernel.org/linux-spi/) | إضافة `SPI_SLAVE` flag وتغيير الـ `spi_master` struct |
| [spi: introduce `spi_controller`](https://lore.kernel.org/linux-spi/) | rename من `spi_master` إلى `spi_controller` لدعم الـ slave mode |

---

### Mailing List

الـ SPI subsystem له mailing list منفصلة:

- **الأرشيف**: [https://lore.kernel.org/linux-spi/](https://lore.kernel.org/linux-spi/)
- **الـ maintainer**: Mark Brown — بيراجع كل الـ patches الخاصة بـ SPI
- للبحث عن نقاش معين: `https://lore.kernel.org/linux-spi/?q=spi_transfer`

---

### eLinux.org

| الصفحة | المحتوى |
|--------|---------|
| [RPi SPI](https://elinux.org/RPi_SPI) | SPI على Raspberry Pi — مثال عملي ممتاز للـ `spidev` من userspace |
| [SPI Memory Support in Linux and U-Boot](https://elinux.org/images/7/73/Raynal-spi-memories.pdf) | PDF شامل عن الـ SPI memory layer (SPI NOR/NAND) |
| [An Introduction to SPI-NOR Subsystem](https://elinux.org/images/4/4e/An_Introduction_to_SPI-NOR_Subsystem_-_v3_0.pdf) | مقدمة تفصيلية لـ SPI-NOR subsystem |
| [BeagleBoard SPI Patch](https://elinux.org/BeagleBoard/SPI/Patch-2.6.37) | مثال على إضافة SPI support لـ board معين |

---

### kernelnewbies.org

التغييرات المهمة في الـ SPI عبر الإصدارات موثقة في:

| الإصدار | التغيير |
|---------|---------|
| [Linux 2.6.24](https://kernelnewbies.org/Linux_2_6_24) | أول دعم رسمي لـ SPI في الـ mainline |
| [Linux 3.15 — DriversArch](https://kernelnewbies.org/Linux_3.15-DriversArch) | تحسينات على الـ SPI core وإضافة drivers جديدة |
| [Linux 4.13](https://kernelnewbies.org/Linux_4.13) | تغييرات على الـ SPI slave support |
| [Linux 4.18](https://kernelnewbies.org/Linux_4.18) | تحسينات الـ SPI DMA handling |
| [Linux 6.5](https://kernelnewbies.org/Linux_6.5) | تحديثات حديثة على الـ SPI subsystem |
| [Linux 6.18](https://kernelnewbies.org/Linux_6.18) | Virtio SPI driver وـ Amlogic SPI flash support |

---

### كتب مرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل المهم**: Chapter 1 (Driver Model) + الـ bus/device/driver model بشكل عام
- الكتاب مجاني أونلاين: [LDD3 on LWN](https://lwn.net/Kernel/LDD2/)
- الـ SPI مش موجود بشكل مباشر في LDD3 لأنه أقدم من الـ subsystem، لكن الـ chapters عن الـ device model أساسية

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 14**: The Block I/O Layer — يفهمك الـ request queues (نفس المفهوم في SPI)
- **الفصل 17**: Devices and Modules — الـ device model الأساسي

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 10**: MTD Subsystem — مرتبط بـ SPI NOR/NAND
- **الفصل 12**: Embedded Development Environment — إعداد بيئة الـ SPI testing

#### Essential Linux Device Drivers — Sreekrishnan Venkateswaran
- **الفصل 8**: Serial Drivers — نفس مبادئ الـ serial protocols
- بيغطي الـ SPI بشكل مباشر مع أمثلة على `spi_driver` و`spi_board_info`

---

### أدوات وموارد تقنية إضافية

#### Userspace Testing
```bash
# تثبيت أدوات الـ spidev
apt-get install spi-tools

# اختبار SPI device من userspace
spi-config -d /dev/spidev0.0 --mode=0 --lsb=0 --bits=8 --speed=1000000
spi-pipe -d /dev/spidev0.0 < /dev/urandom | xxd | head
```

#### فحص الـ SPI devices في النظام
```bash
ls /sys/bus/spi/devices/          # كل الـ SPI devices المتاحة
cat /sys/bus/spi/devices/spi0.0/modalias  # الـ driver المرتبط
ls /sys/class/spidev/             # الـ spidev character devices
```

---

### Search Terms للبحث عن معلومات أكتر

```
linux kernel SPI subsystem internals
spi_controller linux driver implementation
linux spi_transfer DMA scatter-gather
linux spidev userspace interface
linux SPI slave mode implementation
Mark Brown linux SPI maintainer
linux SPI device tree binding
linux SPI cs-gpio chip select
linux SPI ACPI enumeration
linux mtd spi-nor framework
linux spi-nand subsystem
lore.kernel.org linux-spi patches
```

---

### ملخص المصادر حسب الأولوية

| الأولوية | المصدر | الهدف |
|---------|--------|-------|
| 1 | `Documentation/spi/spi-summary.rst` | فهم الـ subsystem بشكل عام |
| 2 | `include/linux/spi/spi.h` | الـ API الكاملة |
| 3 | [LWN: simple SPI framework](https://lwn.net/Articles/154425/) | فهم التصميم الأصلي والـ rationale |
| 4 | [lore.kernel.org/linux-spi](https://lore.kernel.org/linux-spi/) | متابعة التطوير الحالي |
| 5 | [kernel.org SPI docs](https://www.kernel.org/doc/html/v4.14/driver-api/spi.html) | المرجع الرسمي |
| 6 | [eLinux RPi SPI](https://elinux.org/RPi_SPI) | تطبيق عملي على hardware حقيقي |
## Phase 8: Writing simple module

### الفكرة

**`spi_setup()`** هي من أكثر الـ exported functions إثارة للاهتمام في SPI subsystem — كل SPI driver يستدعيها لضبط الـ clock، الـ mode، والـ bits_per_word قبل أي transfer. بـ **kprobe** على `spi_setup` نقدر نشوف كل device بيتـ configure في real-time: اسمه، الـ speed، الـ SPI mode، وعدد الـ bits.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * spi_setup_probe.c
 *
 * Attach a kprobe to spi_setup() to log every SPI device
 * configuration event: device name, speed, mode, bits-per-word.
 *
 * Build with a Makefile that sets:
 *   obj-m += spi_setup_probe.o
 */

#include <linux/module.h>       /* MODULE_*, module_init/exit         */
#include <linux/kernel.h>       /* pr_info()                          */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe()   */
#include <linux/spi/spi.h>      /* struct spi_device, SPI_MODE_*      */
#include <linux/device.h>       /* dev_name()                         */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("SPI Observer <observer@example.com>");
MODULE_DESCRIPTION("kprobe on spi_setup() to log SPI device configuration");

/* ------------------------------------------------------------------ */
/* pre-handler: fires just before spi_setup() executes                 */
/* ------------------------------------------------------------------ */

/*
 * spi_setup() prototype:
 *   int spi_setup(struct spi_device *spi);
 *
 * On x86-64, first argument lives in register RDI → regs->di
 * On ARM64,  first argument lives in register X0  → regs->regs[0]
 *
 * We use the portable pt_regs_read helper via a cast.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * Cast the first argument register to a pointer to spi_device.
     * On x86-64: regs->di holds the first argument.
     * This is architecture-specific but standard for kprobe usage.
     */
    struct spi_device *spi = (struct spi_device *)regs->di;

    /* Guard: validate the pointer is not NULL before dereferencing */
    if (!spi)
        return 0;

    /*
     * spi->mode contains both kernel-private bits (upper nibble)
     * and user-visible bits (lower bits like CPOL, CPHA, CS_HIGH …).
     * Mask with 0xFF to show only the standard SPI mode bits.
     */
    pr_info("[spi_probe] device=%-20s  bus=%d  cs=%d  "
            "speed=%u Hz  mode=0x%02x  bpw=%u\n",
            dev_name(&spi->dev),           /* e.g. "spi0.0"           */
            spi->controller->bus_num,      /* SPI bus number           */
            spi_get_chipselect(spi, 0),    /* primary chip-select      */
            spi->max_speed_hz,             /* max clock frequency      */
            spi->mode & 0xFF,              /* CPOL|CPHA|CS_HIGH|...    */
            spi->bits_per_word             /* word width (0 = 8 bits)  */
    );

    return 0; /* non-zero would abort the probed function — never do that */
}

/* Optional: post-handler fires right after spi_setup() returns */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * regs->ax on x86-64 holds the return value.
     * We just log whether the setup succeeded or failed.
     */
    long ret = (long)regs->ax;

    if (ret)
        pr_info("[spi_probe] spi_setup() returned error: %ld\n", ret);
}

/* ------------------------------------------------------------------ */
/* kprobe descriptor                                                    */
/* ------------------------------------------------------------------ */

static struct kprobe kp = {
    .symbol_name = "spi_setup",   /* kernel symbol to hook            */
    .pre_handler  = handler_pre,  /* called before the function runs  */
    .post_handler = handler_post, /* called after the function returns */
};

/* ------------------------------------------------------------------ */
/* module init / exit                                                   */
/* ------------------------------------------------------------------ */

static int __init spi_probe_init(void)
{
    int ret;

    /*
     * register_kprobe() patches the kernel text at spi_setup's
     * entry point with a breakpoint instruction (INT3 on x86).
     * When hit, the CPU transfers control to our handlers.
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[spi_probe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[spi_probe] kprobe planted at %p (spi_setup)\n", kp.addr);
    return 0;
}

static void __exit spi_probe_exit(void)
{
    /*
     * unregister_kprobe() restores the original instruction at the
     * patched address and waits for any in-flight handlers to finish.
     * Skipping this would leave a dangling breakpoint → kernel oops.
     */
    unregister_kprobe(&kp);
    pr_info("[spi_probe] kprobe removed from spi_setup\n");
}

module_init(spi_probe_init);
module_exit(spi_probe_exit);
```

---

### Makefile لبناء الـ Module

```makefile
# Place this Makefile next to spi_setup_probe.c
# Usage: make -C /lib/modules/$(uname -r)/build M=$PWD modules

obj-m += spi_setup_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `<linux/kprobes.h>` | بيعرّف `struct kprobe` وكل API اللي بنحتاجه لتسجيل الـ probe |
| `<linux/spi/spi.h>` | بيعرّف `struct spi_device` وكل الـ accessors زي `spi_get_chipselect()` |
| `<linux/device.h>` | بيعرّف `dev_name()` اللي بترجع اسم الـ device زي `"spi0.0"` |
| `<linux/module.h>` | لازم لأي kernel module — بيعرّف `MODULE_*` وماكروهات الـ init/exit |

---

#### الـ `handler_pre` — ليه كده؟

**الـ `pt_regs *regs`** بيحمل حالة الـ CPU registers لحظة ضرب الـ breakpoint. على x86-64 الـ calling convention بيحط أول argument في **`RDI`** واللي بيظهر كـ `regs->di` — ده بيسمحلنا نوصل لـ `struct spi_device *spi` بدون ما نغير أي حاجة في السلوك الأصلي.

**الـ `return 0`** ضروري — لو رجعنا قيمة غير صفر، الـ kprobe framework هيـ abort تنفيذ `spi_setup` نفسها وده مش اللي بنعمله هنا.

---

#### الـ `handler_post` — ليه مفيد؟

بيشتغل بعد ما `spi_setup()` ترجع فيعرفنا لو فيه controller رفض الـ configuration (مثلاً طلب `bits_per_word` مش مدعوم). ده مفيد في debugging السيناريوهات اللي فيها device بيـ fail بصمت.

---

#### الـ `struct kprobe kp`

**`symbol_name = "spi_setup"`** بيخلي الـ framework يحل العنوان تلقائياً بدل ما نحدد عنوان عددي ثابت — ده أكثر portability بين الـ kernel versions. الـ `kp.addr` بيتملى بعد `register_kprobe()` تلقائياً.

---

#### الـ `module_init` و `module_exit`

- **`register_kprobe()`** في الـ init بيـ patch الـ kernel text في العنوان المطلوب بـ breakpoint instruction (INT3). لو فشل بيرجع error code سلبي ولازم نتعامل معاه.
- **`unregister_kprobe()`** في الـ exit ضروري جداً — بيستعيد الـ instruction الأصلية ويضمن إن مفيش handler شغال وقت الـ removal. بدونه الـ kernel هيـ crash لما يحاول يتنفذ breakpoint في address مش موجودة في الـ module اتانزل.

---

### تجربة الـ Module

```bash
# build
make -C /lib/modules/$(uname -r)/build M=$PWD modules

# load
sudo insmod spi_setup_probe.ko

# watch output (any SPI driver probe will trigger it)
sudo dmesg -w | grep spi_probe

# example output on a Raspberry Pi with an MCP3204 ADC on spi0.0:
# [spi_probe] device=spi0.0               bus=0  cs=0  speed=1000000 Hz  mode=0x00  bpw=8

# trigger manually: re-bind a known SPI device
echo spi0.0 | sudo tee /sys/bus/spi/drivers/mcp320x/bind

# unload
sudo rmmod spi_setup_probe
```

---

### ملاحظة على الـ Portability

الكود فوق بيستخدم `regs->di` وده **x86-64 only**. عشان يكون portable:

```c
/* Portable way using kprobe's fetch_ops (kernel >= 5.8) */
#ifdef CONFIG_X86_64
    struct spi_device *spi = (struct spi_device *)regs->di;
#elif defined(CONFIG_ARM64)
    struct spi_device *spi = (struct spi_device *)regs->regs[0];
#elif defined(CONFIG_ARM)
    struct spi_device *spi = (struct spi_device *)regs->ARM_r0;
#else
#error "Unsupported architecture"
#endif
```

في الـ production استخدم `kretprobe` أو `fprobe` اللي بتقدم **`func_get_func_arg()`** للوصول للـ arguments بشكل portable.
