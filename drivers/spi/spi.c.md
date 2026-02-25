## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem: SPI SUBSYSTEM

الـ maintainer هو Mark Brown، والـ mailing list هو `linux-spi@vger.kernel.org`.

الملفات الأساسية للـ subsystem:
- `drivers/spi/spi.c` ← الملف اللي بندرسه — قلب الـ subsystem
- `drivers/spi/spi-mem.c` ← طبقة فوق SPI للتعامل مع Flash وذاكرة SPI
- `drivers/spi/spidev.c` ← يعطي userspace access للـ SPI bus
- `drivers/spi/spi-offload.c` ← تفويض نقل البيانات للـ hardware بدون CPU
- `drivers/spi/spi-mux.c` ← multiplexer للـ SPI bus
- `include/linux/spi/spi.h` ← تعريف كل الـ structs الأساسية
- `include/linux/spi/spi-mem.h` ← واجهة ذاكرة SPI
- `include/linux/spi/offload/` ← واجهة الـ offload
- `include/trace/events/spi.h` ← tracepoints للـ debugging

---

### ما هو SPI؟

**SPI** (Serial Peripheral Interface) بروتوكول تواصل بسيط جداً بين الـ CPU وأجهزة صغيرة زي:
- شريحة Flash فيها firmware الـ router
- حساس درجة الحرارة في اللوحة الأم
- شاشة LCD صغيرة
- دائرة RTC (ساعة) جنب الـ BIOS

---

### التشبيه البسيط (ELI5)

تخيل مطعم فيه:
- **طاولة SPI bus** مشتركة — 3 أسلاك: MOSI (ودن الطلبية للطباخ), MISO (ودن الجواب للزبون), SCLK (ساعة تنظّم الكلام)
- **كل كرسي = chip select** — الـ controller يختار مين يكلم بالضغط على "زر CS" الخاص بيه
- **الزبون = protocol driver** — يقول "أنا عايز أكتب/أقرأ من شريحة Flash"
- **Waiter = SPI Core (spi.c)** — يستقبل الطلب، يرتبه في queue، يوديه للمطبخ بالترتيب
- **المطبخ = SPI controller driver** — هو اللي يتعامل مع الـ hardware الفعلي (SoC SPI registers)

الـ `spi.c` هو الـ "waiter" ده بالظبط.

---

### المشكلة اللي بيحلها `spi.c`

#### القصة:

في أي embedded system عندك controller hardware واحد (مثلاً Raspberry Pi SoC) وعليه متصل 3 شرائح: Flash, ADC, وRTC. كل شريحة ليها driver منفصل. المشكلة:

1. **التزاحم** — الـ Flash driver عايز يقرأ في نفس الوقت اللي الـ RTC driver عايز يكتب. مين يتكلم الأول؟
2. **الاختلاف في الـ hardware** — كل SoC فيه controller مختلف. كيف تكتب Flash driver يشتغل على كل الـ SoCs؟
3. **الـ chip select** — اللي الـ CPU يضغط عليه عشان يقول "أنا باكلم الشريحة دي دلوقتي وبس"
4. **DMA** — نقل بيانات كبيرة بدون إشغال الـ CPU
5. **Power Management** — إيقاف الـ controller لما مفيش شغل

**الحل اللي `spi.c` بيقدمه:**

```
[ Flash Driver ]  [ RTC Driver ]  [ ADC Driver ]
        ↓               ↓               ↓
    spi_message     spi_message     spi_message
        ↘               ↓               ↙
         ====== SPI CORE (spi.c) ======
              Queue + Scheduling + CS
                       ↓
         [ SPI Controller Driver (HW specific) ]
                       ↓
         [ SPI Bus: MOSI / MISO / SCLK / CS ]
                       ↓
         [ Flash ]  [ RTC ]  [ ADC ]
```

---

### ما اللي بيعمله `spi.c` تحديداً؟

#### 1. تعريف الـ SPI bus كـ Linux bus

```c
const struct bus_type spi_bus_type = {
    .name    = "spi",
    .match   = spi_match_device,   /* يطابق device مع driver */
    .uevent  = spi_uevent,         /* hotplug */
    .probe   = spi_probe,
    .remove  = spi_remove,
};
```

الـ `spi_bus_type` يعمل Linux يعرف "spi" كـ bus حقيقية — يعني أي device يتسجل عليها وأي driver كمان.

#### 2. إدارة الـ spi_device

الـ **`struct spi_device`** يمثل شريحة واحدة متصلة بالـ bus (زي Flash chip). يتضمن:
- `max_speed_hz` — أقصى سرعة
- `mode` — SPI_MODE_0/1/2/3 (CPOL/CPHA)
- `chip_select[]` — رقم الـ CS
- `controller` — البوينتر للـ controller اللي عليه

```c
struct spi_device *spi_alloc_device(struct spi_controller *ctlr);
int spi_add_device(struct spi_device *spi);
void spi_unregister_device(struct spi_device *spi);
```

#### 3. تسجيل الـ spi_controller

الـ **`struct spi_controller`** هو الـ interface بين الـ core وـ hardware driver. أهم الـ callbacks:

```c
struct spi_controller {
    int  (*setup)(struct spi_device *spi);           /* ضبط الـ mode والـ clock */
    int  (*transfer_one)(struct spi_controller *,    /* نقل transfer واحد */
                         struct spi_device *,
                         struct spi_transfer *);
    void (*set_cs)(struct spi_device *, bool enable);/* تحكم في chip select */
    int  (*prepare_transfer_hardware)(struct spi_controller *);
    int  (*unprepare_transfer_hardware)(struct spi_controller *);
};
```

الـ hardware driver (مثلاً `spi-bcm2835.c`) يملى الـ struct ده ويعمل `spi_register_controller()`.

#### 4. نظام الـ Message Queue

الـ **`struct spi_message`** هو وحدة الشغل الأساسية — atomic sequence من الـ transfers:

```c
struct spi_message {
    struct list_head   transfers; /* قائمة من spi_transfer */
    struct spi_device *spi;
    void (*complete)(void *context); /* callback لما يخلص */
    int   status;
};
```

الـ **`struct spi_transfer`** هو قطعة بيانات واحدة:

```c
struct spi_transfer {
    const void *tx_buf;   /* بيانات للإرسال */
    void       *rx_buf;   /* buffer للاستقبال */
    unsigned    len;
    u32         speed_hz; /* سرعة هذا الـ transfer */
    u8          bits_per_word;
};
```

#### 5. الـ Message Pump (kthread)

الـ core يشغّل **kthread** اسمه `spi_pump_messages` لكل controller. لما يجي message جديد:

```
spi_async()  →  يضيف للـ queue
spi_sync()   →  يضيف وينتظر أو يعمل fast-path مباشر
                         ↓
              __spi_pump_messages() [kthread]
                         ↓
         __spi_pump_transfer_message()
                         ↓
            ctlr->transfer_one_message()
                         ↓
            spi_finalize_current_message()
```

الـ fast-path في `spi_sync()`: لو الـ queue فاضية يبعت المسج مباشرة بدون queue — يوفر latency.

#### 6. DMA Support

لو الـ controller يدعم DMA، الـ core يعمل:
- `spi_map_msg()` — يعمل DMA mapping للـ buffers قبل الإرسال
- `spi_unmap_msg()` — يفك الـ mapping بعد الانتهاء
- يدعم vmalloc buffers وكمان HIGHMEM

#### 7. Chip Select Management

الـ `spi_set_cs()` بتتحكم في الـ CS سواء كان:
- **Native CS** — الـ controller hardware نفسه بيتحكم فيه
- **GPIO CS** — pin عادي بيتحكم فيه الـ GPIO subsystem

مع دعم كامل للـ delays: `cs_setup`, `cs_hold`, `cs_inactive`.

#### 8. Device Registration من الـ Device Tree وـ ACPI

الـ core بيقرأ الـ Device Tree:
```
spi@fe000000 {
    flash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;  ← chip select 0
        spi-max-frequency = <50000000>;
    };
};
```
ويعمل تلقائياً `spi_device` لكل child node بدون ما الـ driver يعمل حاجة.

#### 9. Statistics (per-CPU)

الـ core يحسب per-device وper-controller إحصائيات مرئية في sysfs:
```
/sys/bus/spi/devices/spi0.0/statistics/
├── bytes_rx
├── bytes_tx
├── transfers
├── errors
├── transfer_bytes_histo_0-1
└── ...
```

#### 10. Transfer Splitting

لو الـ hardware مش بيقدر يعمل transfer أكبر من حجم معين، الـ `spi_split_transfers_maxsize()` بتقسمه تلقائياً لعدة transfers أصغر — شفاف تماماً للـ protocol driver.

---

### الـ Three Layers بالتفصيل

```
┌─────────────────────────────────────────────────────────┐
│  Protocol Drivers (spi-nor, mmc_spi, ads7846 ...)       │
│  يستخدموا: spi_sync(), spi_async(), spi_write_then_read │
├─────────────────────────────────────────────────────────┤
│           SPI CORE  (drivers/spi/spi.c)                 │
│  • bus registration   • message queue & pump            │
│  • device matching    • DMA mapping                     │
│  • CS management      • statistics                      │
│  • OF/ACPI enumeration • transfer splitting             │
├─────────────────────────────────────────────────────────┤
│  Controller Drivers (spi-bcm2835.c, spi-pl022.c ...)    │
│  يوفروا: transfer_one(), set_cs(), setup()              │
└─────────────────────────────────────────────────────────┘
              ↓ Physical SPI Bus (4 wires) ↓
         [ MOSI ]  [ MISO ]  [ SCLK ]  [ CS ]
```

---

### الملفات المهمة اللي المبرمج محتاج يعرفها

| الملف | الدور |
|-------|-------|
| `drivers/spi/spi.c` | قلب الـ subsystem — الملف ده |
| `drivers/spi/spi-mem.c` | واجهة للـ SPI Flash والـ EEPROM (NOR, NAND) |
| `drivers/spi/spidev.c` | يكشف SPI لـ userspace عبر `/dev/spidevX.Y` |
| `drivers/spi/internals.h` | shared internals بين ملفات الـ core |
| `drivers/spi/spi-offload.c` | hardware offload لتقليل CPU overhead |
| `include/linux/spi/spi.h` | تعريف كل الـ structs والـ API |
| `include/linux/spi/spi-mem.h` | API خاص بذاكرة SPI |
| `include/trace/events/spi.h` | tracepoints (ftrace debugging) |
| `Documentation/spi/spi-summary.rst` | وثائق رسمية للمطورين |
| `tools/spi/` | أدوات test لـ SPI من command line |

---

### مثال حقيقي: إيه اللي بيحصل لما Flash Driver يطلب قراءة

1. **Flash driver** يعمل `spi_message_init(&msg)` ويضيف `spi_transfer` للقراءة
2. يستدعي `spi_sync(spi_dev, &msg)`
3. الـ **core** يتحقق إن الـ queue فاضية → fast-path مباشر
4. يستدعي `spi_setup()` لو أول مرة
5. يعمل `DMA map` للـ buffer
6. يستدعي `ctlr->transfer_one_message()` → الـ hardware driver يبدأ النقل الفعلي
7. الـ **hardware** يضغط CS↓، يبعت SCLK pulses، يشوف MISO، يرفع CS↑
8. `spi_finalize_current_message()` → الـ core يكمل الـ completion
9. الـ `spi_sync()` بترجع للـ Flash driver بالبيانات
## Phase 2: شرح الـ SPI Framework

### المشكلة اللي بيحلها الـ SPI Subsystem

**SPI (Serial Peripheral Interface)** هو بروتوكول تواصل synchronous full-duplex بين الـ CPU والـ peripheral devices زي flash memories, ADCs, sensors, displays. البروتوكول نفسه بسيط: 4 أسلاك (MOSI, MISO, SCK, CS)، بس التحدي الحقيقي هو في إدارته من ناحية software.

**المشكلة الأولى**: كل chip بيعمل SPI controller له registers مختلفة وطريقة تشغيل مختلفة (SPI mode 0/1/2/3، speeds، chip selects). لو كل driver كتب كود SPI من الصفر، كان هيبقى عندنا code duplication هائلة وصعوبة في الصيانة.

**المشكلة الثانية**: في نفس الوقت ممكن يكون في أكتر من device على نفس الـ SPI bus. لازم يكون في arbitration صح: مين بيستخدم الـ bus امتى، وإزاي نضمن إن الـ transactions atomic (ما تتقطعش في النص).

**المشكلة الثالثة**: الـ protocol drivers (اللي بيتحدثوا مع flash chip مثلاً) ما المفروضش يعرفوا حاجة عن الـ hardware الـ SPI controller نفسه. محتاجين abstraction layer بينهم.

---

### الحل: الـ SPI Framework

الـ kernel حل الموضوع بتقسيم المسئوليات لـ 3 طبقات:

1. **Controller Driver** (الـ low-level): بيعرف يتكلم مع الـ SPI hardware مباشرة، بيملأ function pointers في struct `spi_controller`.
2. **SPI Core** (الـ `spi.c` اللي بندرسه): الـ middle layer اللي بيدير الـ bus، الـ queuing، الـ matching بين devices والـ drivers، والـ DMA mapping.
3. **Protocol Driver / Peripheral Driver**: بيعرف بس إنه شايل `spi_device` ويبعت `spi_message`. مش محتاج يعرف أي SPI controller موجود.

---

### الـ Big Picture Architecture

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                    Peripheral / Protocol Drivers                │
  │   (spi-nor, w25q128, mcp23s08, spidev, etc.)                    │
  │   يستخدموا: spi_sync(), spi_async(), spi_message, spi_transfer  │
  └──────────────────────────┬──────────────────────────────────────┘
                             │  spi_message submissions
                             ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │                      SPI Core (spi.c)                          │
  │                                                                 │
  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────────┐  │
  │  │  Bus Type     │  │  Message      │  │  DMA Mapping      │  │
  │  │  spi_bus_type │  │  Queue/Pump   │  │  spi_map_msg()    │  │
  │  │  matching     │  │  kthread_work │  │  sg_table setup   │  │
  │  └───────────────┘  └───────────────┘  └───────────────────┘  │
  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────────┐  │
  │  │  Device Reg.  │  │  CS Mgmt      │  │  Transfer Split   │  │
  │  │  DT/ACPI scan │  │  spi_set_cs() │  │  maxsize/maxwords │  │
  │  └───────────────┘  └───────────────┘  └───────────────────┘  │
  └──────────────────────────┬──────────────────────────────────────┘
                             │  calls function pointers
                             ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │                   SPI Controller Drivers                        │
  │   (spi-pl022, spi-imx, spi-bcm2835, spi-rockchip, etc.)        │
  │   بيملأوا: transfer_one(), set_cs(), setup(), prepare_message() │
  └──────────────────────────┬──────────────────────────────────────┘
                             │
                             ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │                    SPI Hardware (Registers)                     │
  │    ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐          │
  │    │ CS0  │  │ CS1  │  │ CS2  │  │ CS3  │  │ ...  │          │
  │    │Flash │  │ADC   │  │LCD   │  │Codec │  │      │          │
  │    └──────┘  └──────┘  └──────┘  └──────┘  └──────┘          │
  └─────────────────────────────────────────────────────────────────┘
```

---

### الـ Real-World Analogy: شبكة مطاعم بنفس المطبخ

تخيل إن عندك **مطبخ مركزي واحد** (= SPI Controller hardware) بيخدم **أكتر من كاونتر طلبات** (= SPI devices/chip selects).

| الـ SPI concept | الـ Analogy |
|---|---|
| `spi_controller` | المطبخ المركزي (الأوان، النار، كل الأدوات) |
| `spi_device` | كاونتر طلب معين (بيزا، سوشي، برجر) |
| `spi_driver` | الـ recipe / البرنامج الخاص بكل طبق |
| `spi_message` | طلب كامل من الزبون (كل الأكل المطلوب مرة واحدة) |
| `spi_transfer` | طبق واحد في الطلب (appetizer, main, dessert) |
| `spi_bus_type` | نظام الـ POS اللي بيوصل الطلبات للمطبخ |
| `chip select (CS)` | الـ "رقم طاولة" اللي بيحدد مين الزبون |
| Message Queue | قائمة الطلبات اللي لسه متنفذتش |
| `kthread (pump)` | الكاشير اللي بيبعت الطلبات للمطبخ واحدة واحدة |

**التعمق في الـ Analogy**:
- المطبخ (controller) بيعمل حاجة واحدة في الوقت (إزاي تطبخ لطاولتين بنفس الوقت في نفس الأوان؟)
- الطلب (message) "atomic": لو الزبون طلب سوشي وبعدين بيتزا، ما ينفعش حد تاني يقاطع في النص.
- الـ CS (رقم الطاولة): المطبخ لازم يعرف هيبعت الأكل لمين بالزبط.
- الـ kworker thread هو الـ runner اللي بياخد الطلبات من الـ queue ويوديها المطبخ ويرجع بالنتيجة.

---

### الـ Core Abstractions: الـ 5 Structs الأساسية

#### 1. `struct spi_controller` — المحور الرئيسي

ده أكبر struct في الـ framework وبيمثل الـ SPI hardware controller نفسه.

```c
struct spi_controller {
    struct device   dev;          /* kernel device model integration */
    struct list_head list;        /* global list of all controllers */
    s16             bus_num;      /* SPI bus number (e.g. spi0, spi1) */
    u16             num_chipselect; /* how many CS lines */

    /* ---- Function Pointers (filled by controller driver) ---- */
    int  (*setup)(struct spi_device *spi);         /* configure device */
    int  (*transfer_one)(struct spi_controller *,
                         struct spi_device *,
                         struct spi_transfer *);    /* do one transfer */
    void (*set_cs)(struct spi_device *, bool);      /* toggle CS line */
    int  (*prepare_message)(struct spi_controller *,
                            struct spi_message *);  /* DMA prep etc. */
    int  (*transfer_one_message)(struct spi_controller *,
                                 struct spi_message *); /* full message */

    /* ---- Queue Management (managed by core) ---- */
    struct kthread_worker  *kworker;     /* the pump thread */
    struct kthread_work     pump_messages; /* work item */
    spinlock_t              queue_lock;
    struct list_head        queue;       /* pending spi_messages */
    struct spi_message     *cur_msg;     /* currently executing */
    bool                    busy;
    bool                    running;
    bool                    queue_empty; /* optimization flag */

    /* ---- DMA Support ---- */
    bool (*can_dma)(struct spi_controller *, struct spi_device *,
                    struct spi_transfer *);
    struct dma_chan  *dma_tx;
    struct dma_chan  *dma_rx;
    void             *dummy_rx;   /* for MUST_RX controllers */
    void             *dummy_tx;   /* for MUST_TX controllers */
};
```

**تلاحظ** إن الـ controller driver بيملأ الـ function pointers، والـ core بيدير كل التاني.

الـ driver عنده خيارين للـ transfer:
- **الطريقة البسيطة**: يملأ `transfer_one` بس — الـ core بيتكفل بالـ CS management والـ delays والـ completion.
- **الطريقة المتقدمة**: يملأ `transfer_one_message` بالكامل — الـ core بس بيبعتله الـ message كاملة.

#### 2. `struct spi_device` — التمثيل الـ logical لكل chip

```c
struct spi_device {
    struct device       dev;           /* embedded kernel device */
    struct spi_controller *controller; /* pointer back to the controller */
    u32                 max_speed_hz;  /* max clock for this chip */
    u8                  bits_per_word; /* 8, 16, 32 bits etc. */
    u32                 mode;          /* SPI_CPHA, SPI_CPOL, SPI_CS_HIGH... */
    int                 irq;           /* optional IRQ line */
    char                modalias[SPI_NAME_SIZE]; /* driver name to match */

    /* CS management */
    u8                  chip_select[SPI_DEVICE_CS_CNT_MAX]; /* physical CS numbers */
    u8                  num_chipselect;
    struct gpio_desc   *cs_gpiod[SPI_DEVICE_CS_CNT_MAX];    /* GPIO-based CS */

    /* Timing */
    struct spi_delay    cs_setup;    /* delay after CS assert */
    struct spi_delay    cs_hold;     /* delay before CS deassert */
    struct spi_delay    cs_inactive; /* delay after CS deassert */
    struct spi_delay    word_delay;  /* inter-word delay */

    /* Per-CPU statistics */
    struct spi_statistics __percpu *pcpu_statistics;
};
```

الـ `spi_device` ما بيعرفش حاجة عن الـ hardware registers. هو بس بيقول "أنا عايز أتكلم بـ mode X بسرعة Y على CS رقم Z".

#### 3. `struct spi_transfer` — وحدة النقل الأساسية

```c
struct spi_transfer {
    const void  *tx_buf;    /* data to send (or NULL for rx-only) */
    void        *rx_buf;    /* buffer to receive into (or NULL for tx-only) */
    unsigned    len;        /* number of bytes */

    u32         speed_hz;       /* clock rate override (0 = use device default) */
    u8          bits_per_word;  /* word size override (0 = use device default) */

    /* Chip select control */
    unsigned    cs_change:1;   /* toggle CS after this transfer */
    unsigned    cs_off:1;      /* do this transfer with CS de-asserted */

    /* Lane control for QSPI/Octal SPI */
    unsigned    tx_nbits:4;    /* SPI_NBITS_SINGLE/DUAL/QUAD/OCTAL */
    unsigned    rx_nbits:4;

    /* DMA scatter-gather (filled by core) */
    struct sg_table tx_sg;
    struct sg_table rx_sg;
    bool        tx_sg_mapped;
    bool        rx_sg_mapped;

    /* Inter-transfer delays */
    struct spi_delay delay;          /* after transfer */
    struct spi_delay cs_change_delay; /* between CS toggle */
    struct spi_delay word_delay;     /* between words */

    struct list_head transfer_list;  /* link in spi_message.transfers */
};
```

الـ full-duplex طبيعة SPI بتظهر هنا: نفس الـ transfer بيبعت ويستقبل في نفس الوقت. لو `tx_buf = NULL`، الـ core بيبعت zeros. لو `rx_buf = NULL`، الـ received data بتتبور.

#### 4. `struct spi_message` — الـ atomic transaction

```c
struct spi_message {
    struct list_head  transfers;     /* list of spi_transfers */
    struct spi_device *spi;          /* device this message is for */

    /* Optimization state */
    bool    pre_optimized;   /* peripheral driver called spi_optimize_message() */
    bool    optimized;       /* core finished optimization */
    bool    prepared;        /* prepare_message() was called */

    /* Completion callback */
    int     status;                  /* 0=success, negative=error */
    void    (*complete)(void *ctx);  /* called when done */
    void    *context;

    unsigned frame_length;   /* total bytes in all transfers */
    unsigned actual_length;  /* bytes actually transferred */

    /* Queue management */
    struct list_head  queue;    /* link in ctlr->queue */

    /* Resources (for transfer splitting etc.) */
    struct list_head  resources;
};
```

الـ message هي الوحدة الـ "atomic" على الـ bus. طول ما الـ message شغالة، ما في device تاني يقدر يأخد الـ bus.

#### 5. `struct spi_driver` — الـ protocol driver

```c
struct spi_driver {
    const struct spi_device_id *id_table; /* list of supported devices */
    int   (*probe)(struct spi_device *);  /* device found, init it */
    void  (*remove)(struct spi_device *); /* device removed */
    void  (*shutdown)(struct spi_device *);
    struct device_driver driver;          /* embedded kernel driver */
};
```

---

### رسم العلاقات بين الـ Structs

```
spi_controller
│
├── dev (struct device) ─── registered as "spi0", "spi1" etc. in sysfs
│
├── queue (list_head) ──────── spi_message ──── spi_message ──── ...
│                               │                │
│                               ├── transfers ── spi_transfer
│                               │               └── spi_transfer
│                               ├── spi ──────► spi_device
│                               └── complete()
│
├── kworker ─────────────────── pump_messages (kthread_work)
│                               [reads from queue, calls transfer_one_message]
│
└── cs_gpiods[] ─────────────── gpio_desc (optional GPIO-based CS)


spi_device
├── dev (struct device) ─── registered as "spi0.0", "spi0.1" etc.
├── controller ──────────► spi_controller (back-pointer)
├── modalias ────────────── "w25q128" (used for driver matching)
└── mode, speed, BPW ────── per-device defaults


spi_bus_type
├── match() ─────────────── spi_match_device() [modalias/OF/ACPI]
├── probe() ─────────────── spi_probe() [calls sdrv->probe()]
└── uevent() ────────────── spi_uevent() [for hotplug]
```

---

### دورة حياة الـ Message: من `spi_sync()` للـ Hardware

ده أهم جزء. خلينا نتتبع الـ code path بالتفصيل:

#### الخطوة 1: الـ Peripheral Driver بيبعت الـ Message

```c
/* Protocol driver code (e.g. flash driver) */
struct spi_transfer xfer = {
    .tx_buf = cmd_buf,
    .rx_buf = data_buf,
    .len    = len,
};
struct spi_message msg;
spi_message_init(&msg);
spi_message_add_tail(&xfer, &msg);

/* Blocking call - caller sleeps until done */
ret = spi_sync(spi, &msg);
```

#### الخطوة 2: `spi_sync()` ← `__spi_sync()`

```c
static int __spi_sync(struct spi_device *spi, struct spi_message *message)
{
    /* Optimization: validate + split transfers once */
    status = spi_maybe_optimize_message(spi, message);

    /* Fast path: if queue is empty, skip kthread entirely */
    if (READ_ONCE(ctlr->queue_empty) && !ctlr->must_async) {
        __spi_transfer_message_noqueue(ctlr, message);
        return message->status;
    }

    /* Slow path: add to queue and wait */
    message->complete = spi_complete;  /* sets a completion */
    message->context  = &done;
    __spi_async(spi, message);         /* adds to ctlr->queue */
    wait_for_completion(&done);        /* sleep until callback */
    return message->status;
}
```

**الـ Fast Path مهم جداً**: لو الـ queue فاضية، الـ core بيشغل الـ message مباشرة في context المستدعي بدون ما يصحى الـ kthread. ده بيوفر latency كبيرة.

#### الخطوة 3: الـ Message Pump `__spi_pump_messages()`

```c
static void __spi_pump_messages(struct spi_controller *ctlr, bool in_kthread)
{
    spin_lock_irqsave(&ctlr->queue_lock, flags);

    if (ctlr->cur_msg)       /* already processing one */
        goto out_unlock;

    if (list_empty(&ctlr->queue)) /* nothing to do */
        goto out_unlock;

    /* Extract head of queue */
    msg = list_first_entry(&ctlr->queue, struct spi_message, queue);
    ctlr->cur_msg = msg;
    list_del_init(&msg->queue);
    ctlr->busy = true;
    spin_unlock_irqrestore(&ctlr->queue_lock, flags);

    /* Actually send the message */
    __spi_pump_transfer_message(ctlr, msg, was_busy);
    /* After done, re-queue work to check for more */
    kthread_queue_work(ctlr->kworker, &ctlr->pump_messages);
}
```

#### الخطوة 4: `__spi_pump_transfer_message()` — التنفيذ الفعلي

```c
static int __spi_pump_transfer_message(struct spi_controller *ctlr,
                                        struct spi_message *msg, bool was_busy)
{
    /* Power management: wake up the parent device */
    pm_runtime_get_sync(ctlr->dev.parent);

    /* Prepare hardware for this batch of transfers */
    ctlr->prepare_transfer_hardware(ctlr);

    /* Per-message preparation (e.g., DMA channel config) */
    ctlr->prepare_message(ctlr, msg);

    /* Map buffers to DMA if possible */
    spi_map_msg(ctlr, msg);

    /* Run the message through transfer_one_message() */
    ctlr->transfer_one_message(ctlr, msg);
    /* Either the driver's own, or core's spi_transfer_one_message() */
}
```

#### الخطوة 5: `spi_transfer_one_message()` — الـ Core Default

الـ core بيوفر implementation افتراضية للـ controllers اللي بتملأ `transfer_one` بس:

```c
static int spi_transfer_one_message(struct spi_controller *ctlr,
                                    struct spi_message *msg)
{
    /* Assert CS */
    spi_set_cs(msg->spi, true, false);

    list_for_each_entry(xfer, &msg->transfers, transfer_list) {
        /* Sync DMA buffers before device access */
        spi_dma_sync_for_device(ctlr, xfer);

        /* Call the driver's low-level transfer function */
        ret = ctlr->transfer_one(ctlr, msg->spi, xfer);

        if (ret > 0)  /* async: wait for completion */
            spi_transfer_wait(ctlr, msg, xfer);

        /* Sync DMA back to CPU after device access */
        spi_dma_sync_for_cpu(ctlr, xfer);

        /* Handle inter-transfer delays */
        spi_transfer_delay_exec(xfer);

        /* Handle cs_change: briefly deassert CS between transfers */
        if (xfer->cs_change && !list_is_last(...))
            spi_set_cs(msg->spi, false, false);
            /* delay */
            spi_set_cs(msg->spi, true, false);

        msg->actual_length += xfer->len;
    }

    /* Deassert CS when done */
    spi_set_cs(msg->spi, false, false);

    /* Signal completion */
    spi_finalize_current_message(ctlr);
}
```

---

### الـ Chip Select Management

الـ CS management في الـ SPI core معقد شوية لأن فيه 3 سيناريوهات:

```
  Scenario 1: Native CS (controller hardware)
  ─────────────────────────────────────────────
  ctlr->set_cs(spi, true/false)  ← driver controls its own CS register

  Scenario 2: GPIO-based CS
  ─────────────────────────────────────────────
  gpiod_set_value_cansleep(cs_gpiod, value)  ← GPIO library handles polarity

  Scenario 3: Hybrid (some controllers need both)
  ─────────────────────────────────────────────
  (SPI_CONTROLLER_GPIO_SS flag)
  GPIO CS + ctlr->set_cs()  ← both get called
```

الـ `spi_set_cs()` function في الـ core بتحسب الـ polarity بذكاء:

```c
static void spi_set_cs(struct spi_device *spi, bool enable, bool force)
{
    bool activate = enable;

    /* Optimization: skip if CS state hasn't changed */
    if (!force && enable == spi_is_last_cs(spi) && ...)
        return;

    /* Account for SPI_CS_HIGH mode (active high instead of active low) */
    if (spi->controller->last_cs_mode_high)
        enable = !enable;

    /* Apply hold delay before CS deassert */
    if (!activate)
        spi_delay_exec(&spi->cs_hold, NULL);

    if (spi_is_csgpiod(spi))
        spi_toggle_csgpiod(spi, idx, enable, activate);
    else if (ctlr->set_cs)
        ctlr->set_cs(spi, !enable);

    /* Apply setup/inactive delays */
    if (activate)
        spi_delay_exec(&spi->cs_setup, NULL);
    else
        spi_delay_exec(&spi->cs_inactive, NULL);
}
```

---

### الـ DMA Support: الـ Core بيعمل الـ Mapping

لما الـ controller driver بيعمل `can_dma` callback وبترجع true، الـ core بيتكفل بكل الـ DMA mapping قبل ما يبعت الـ message للـ driver.

```c
static int __spi_map_msg(struct spi_controller *ctlr, struct spi_message *msg)
{
    list_for_each_entry(xfer, &msg->transfers, transfer_list) {
        /* Ask driver if this transfer can use DMA */
        if (!ctlr->can_dma(ctlr, msg->spi, xfer))
            continue;

        /* Map TX buffer to DMA address */
        if (xfer->tx_buf)
            spi_map_buf_attrs(ctlr, tx_dev, &xfer->tx_sg,
                              xfer->tx_buf, xfer->len, DMA_TO_DEVICE, ...);

        /* Map RX buffer to DMA address */
        if (xfer->rx_buf)
            spi_map_buf_attrs(ctlr, rx_dev, &xfer->rx_sg,
                              xfer->rx_buf, xfer->len, DMA_FROM_DEVICE, ...);
    }
}
```

**ملاحظة مهمة**: الـ DMA mapping بيدعم vmalloc buffers وكمان highmem buffers عن طريق تقسيمها لـ scatter-gather entries.

الـ `dummy_rx` و `dummy_tx` مهمين للـ controllers اللي بيطلبوا `MUST_RX` أو `MUST_TX` — يعني hardware بيتطلب إن الـ lines الاتنين يكونوا active حتى لو الـ software مش محتاج الـ data التانية. الـ core بيخلق buffers وهمية وبيحطها مكان الـ NULL buffers.

---

### الـ Device Registration: إزاي الـ spi_device بتتعمل

الـ core عنده 3 طرق تعمل منها `spi_device`:

#### 1. من الـ Device Tree

```c
static void of_register_spi_devices(struct spi_controller *ctlr)
{
    /* Scan all child nodes of the controller node in DT */
    for_each_available_child_of_node(ctlr->dev.of_node, nc) {
        spi = spi_alloc_device(ctlr);
        of_spi_parse_dt(ctlr, spi, nc);  /* reads spi-max-frequency,
                                           spi-cpha, spi-cpol, reg, etc. */
        spi_add_device(spi);
    }
}
```

#### 2. من ACPI

```c
/* acpi_spi_add_resource() parses SPISerialBus() resources
   and fills struct acpi_spi_lookup with speed, mode, CS */
static void acpi_register_spi_devices(struct spi_controller *ctlr)
{
    acpi_walk_namespace(..., acpi_spi_add_device, ...);
}
```

#### 3. من الـ Board File (Legacy)

```c
/* Old-style static table - still used in some platforms */
spi_register_board_info(board_info_table, ARRAY_SIZE(table));
/* Core stores it in board_list and matches when controller registers */
```

---

### الـ Bus Type والـ Driver Matching

الـ `spi_bus_type` هو اللي بيربط الـ `spi_device` بالـ `spi_driver`.

```c
const struct bus_type spi_bus_type = {
    .name    = "spi",
    .match   = spi_match_device,  /* checks if driver handles this device */
    .probe   = spi_probe,         /* calls sdrv->probe() */
    .remove  = spi_remove,
};
```

الـ matching بيمشي بالترتيب ده:
1. لو في `driver_override` → يطابق اسم الـ driver بالضبط
2. الـ OpenFirmware/Device Tree matching (`compatible` string)
3. الـ ACPI matching (HID)
4. الـ `spi_device_id` table matching (modalias)
5. اسم الـ driver vs اسم الـ device

```c
static int spi_match_device(struct device *dev, const struct device_driver *drv)
{
    const struct spi_device *spi = to_spi_device(dev);
    const struct spi_driver *sdrv = to_spi_driver(drv);

    if (spi->driver_override)
        return strcmp(spi->driver_override, drv->name) == 0;

    if (of_driver_match_device(dev, drv))  return 1;
    if (acpi_driver_match_device(dev, drv)) return 1;
    if (sdrv->id_table)
        return !!spi_match_id(sdrv->id_table, spi->modalias);

    return strcmp(spi->modalias, drv->name) == 0;
}
```

---

### الـ Message Optimization: الـ Fast Path للـ Repeated Messages

الـ framework بيدعم **pre-optimization** للـ messages اللي بتتبعت أكتر من مرة (زي الـ SPI flash اللي بيقرأ نفس الـ command كتير):

```c
/* Peripheral driver calls this once per message setup */
spi_optimize_message(spi, &msg);
/* Now msg is validated, split if needed, and controller may
   pre-map DMA buffers. Subsequent spi_sync() calls are faster */

/* ... repeated calls ... */
spi_sync(spi, &msg);   /* skips validate + split every time */

/* When done with this message pattern */
spi_unoptimize_message(&msg);
```

الـ `spi_maybe_optimize_message()` في الـ core بيشوف لو الـ message اتعملت optimize قبل كده يعديها، غير كده يعمل الـ optimization فوراً.

---

### الـ Transfer Splitting

بعض الـ controllers عندها حد أقصى لحجم الـ transfer. الـ core بيتعامل مع ده أوتوماتيك:

```c
/* Core splits any transfer larger than controller's max into
   multiple smaller transfers using spi_replace_transfers() */
int spi_split_transfers_maxsize(struct spi_controller *ctlr,
                                struct spi_message *msg,
                                size_t maxsize)
{
    list_for_each_entry(xfer, &msg->transfers, transfer_list) {
        if (xfer->len > maxsize)
            __spi_split_transfer_maxsize(ctlr, msg, &xfer, maxsize);
    }
}
```

الـ split بيستخدم `spi_res` mechanism: الـ replacement transfers بتتخزن كـ resource في الـ message وبتتحذف automatically لما الـ message تخلص (زي الـ devres لكن على مستوى الـ message).

---

### إيه اللي الـ Core بيملكه vs اللي بيفوضه للـ Controller Driver

| المسئولية | SPI Core | Controller Driver |
|---|---|---|
| قبول الـ messages من الـ peripheral | ✓ | |
| الـ message queue management | ✓ | |
| الـ kthread pump | ✓ | |
| الـ message validation | ✓ | |
| الـ transfer splitting | ✓ | |
| الـ DMA mapping | ✓ | |
| الـ CS timing delays | ✓ | |
| تسجيل الـ devices (DT/ACPI) | ✓ | |
| الـ bus type / driver matching | ✓ | |
| Per-CPU statistics | ✓ | |
| قراءة/كتابة الـ hardware registers | | ✓ |
| تشغيل الـ FIFO/DMA hardware | | ✓ |
| ضبط الـ clock dividers | | ✓ |
| الـ interrupt handling | | ✓ |
| الـ CS signal logic (native) | | ✓ (عبر `set_cs`) |
| الـ CS GPIO logic | ✓ (عبر GPIO lib) | |
| الـ PM (prepare/unprepare HW) | يوصل الـ callbacks | ✓ (implement) |

---

### مثال عملي: الـ Flow في Controller Driver حقيقي

لو اتكتب driver لـ SPI controller جديد على ARM SoC، ده أدنى ما محتاج تعمله:

```c
static int my_spi_probe(struct platform_device *pdev)
{
    struct spi_controller *ctlr;

    /* 1. Allocate controller struct (core manages memory via devres) */
    ctlr = devm_spi_alloc_host(&pdev->dev, sizeof(struct my_priv));

    /* 2. Fill in hardware capabilities */
    ctlr->bus_num         = pdev->id;
    ctlr->num_chipselect  = 4;
    ctlr->mode_bits       = SPI_CPOL | SPI_CPHA | SPI_CS_HIGH;
    ctlr->max_speed_hz    = 50000000;  /* 50 MHz */
    ctlr->bits_per_word_mask = SPI_BPW_MASK(8) | SPI_BPW_MASK(16);

    /* 3. Provide function pointers (minimum needed) */
    ctlr->setup            = my_spi_setup;    /* configure clock/mode */
    ctlr->transfer_one     = my_spi_transfer; /* do one spi_transfer */
    ctlr->set_cs           = my_spi_set_cs;   /* toggle CS pin */

    /* Optional: DMA support */
    ctlr->can_dma          = my_spi_can_dma;
    ctlr->dma_tx           = dma_tx_chan;
    ctlr->dma_rx           = dma_rx_chan;

    /* 4. Register with the core - this triggers DT device enumeration */
    return devm_spi_register_controller(&pdev->dev, ctlr);
    /*
     * Core will:
     *   - assign bus_num
     *   - create kthread for message pump
     *   - scan DT children and create spi_devices
     *   - match spi_devices with spi_drivers
     */
}
```

**ملاحظة مهمة عن الـ Subsystems التانية**:
- **الـ Device Model** (عايز تفهمه أولاً): الـ `struct device` embedded في `spi_controller` و`spi_device` هو اللي بيخلي الـ sysfs entries وddriver binding يشتغلوا. الـ `spi_bus_type` هو implementation لـ `struct bus_type`.
- **الـ GPIO Subsystem**: الـ SPI core بيستخدمه للـ GPIO-based chip selects عبر `gpiod_get_index_optional()` و`gpiod_set_value_cansleep()`.
- **الـ DMA Engine**: الـ core بيتكلم مع الـ DMA engine عبر `struct sg_table` (scatter-gather tables) وبيعمل `dma_map_sgtable()` لتحويل الـ virtual addresses لـ physical DMA addresses.
- **الـ PM Runtime**: الـ core بيعمل `pm_runtime_get_sync()` على الـ parent device قبل أي transfer وبيعمل `pm_runtime_put_autosuspend()` لما الـ queue تفضى.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Enums والـ Config Options

#### أهم الـ mode bits لـ spi_device.mode

| Flag | Value | المعنى |
|---|---|---|
| `SPI_CPHA` | BIT(0) | Clock phase: البيانات تُأخذ على الـ edge التاني |
| `SPI_CPOL` | BIT(1) | Clock polarity: الـ clock يبدأ بـ high |
| `SPI_CS_HIGH` | BIT(2) | الـ chip select active high |
| `SPI_LSB_FIRST` | BIT(3) | الـ LSB يُرسل أول |
| `SPI_3WIRE` | BIT(4) | MOSI وMISO على نفس الـ wire |
| `SPI_LOOP` | BIT(5) | Loopback mode |
| `SPI_NO_CS` | BIT(6) | لا يوجد chip select |
| `SPI_READY` | BIT(7) | Device ترسل إشارة ready |
| `SPI_TX_DUAL` | BIT(8) | إرسال على 2 data lines |
| `SPI_TX_QUAD` | BIT(9) | إرسال على 4 data lines |
| `SPI_RX_DUAL` | BIT(10) | استقبال على 2 data lines |
| `SPI_RX_QUAD` | BIT(11) | استقبال على 4 data lines |
| `SPI_CS_WORD` | BIT(12) | toggle CS بعد كل word |
| `SPI_TX_OCTAL` | BIT(13) | إرسال على 8 data lines |
| `SPI_RX_OCTAL` | BIT(14) | استقبال على 8 data lines |
| `SPI_3WIRE_HIZ` | BIT(15) | High-Z بدل low في 3-wire |
| `SPI_MOSI_IDLE_LOW` | BIT(16) | MOSI idle = low |
| `SPI_MOSI_IDLE_HIGH` | BIT(17) | MOSI idle = high |
| `SPI_NO_TX` | BIT(31) | مفيش wire إرسال (kernel-only) |
| `SPI_NO_RX` | BIT(30) | مفيش wire استقبال (kernel-only) |
| `SPI_TPM_HW_FLOW` | BIT(29) | TPM hardware flow control |

#### أهم الـ controller flags (`spi_controller.flags`)

| Flag | Bit | المعنى |
|---|---|---|
| `SPI_CONTROLLER_HALF_DUPLEX` | 0 | Controller مش full-duplex |
| `SPI_CONTROLLER_NO_RX` | 1 | Controller مش بيقدر يقرأ |
| `SPI_CONTROLLER_NO_TX` | 2 | Controller مش بيقدر يكتب |
| `SPI_CONTROLLER_MUST_RX` | 3 | لازم يبقى فيه rx buffer دايمًا |
| `SPI_CONTROLLER_MUST_TX` | 4 | لازم يبقى فيه tx buffer دايمًا |
| `SPI_CONTROLLER_GPIO_SS` | 5 | الـ CS بيتعمل بـ GPIO |
| `SPI_CONTROLLER_SUSPENDED` | 6 | Controller معمله suspend |
| `SPI_CONTROLLER_MULTI_CS` | 7 | Controller بيدعم multi CS في نفس الوقت |

#### الـ SPI_NBITS (عدد الـ data lines لكل transfer)

| Macro | Value | المعنى |
|---|---|---|
| `SPI_NBITS_SINGLE` | 0x01 | 1 data line (standard) |
| `SPI_NBITS_DUAL` | 0x02 | 2 data lines |
| `SPI_NBITS_QUAD` | 0x04 | 4 data lines |
| `SPI_NBITS_OCTAL` | 0x08 | 8 data lines |

#### الـ spi_delay units

| Macro | Value | المعنى |
|---|---|---|
| `SPI_DELAY_UNIT_USECS` | 0 | التأخير بالـ microseconds |
| `SPI_DELAY_UNIT_NSECS` | 1 | التأخير بالـ nanoseconds |
| `SPI_DELAY_UNIT_SCK` | 2 | التأخير بعدد clock cycles |

#### الـ spi_transfer error bits

| Macro | Bit | المعنى |
|---|---|---|
| `SPI_TRANS_FAIL_NO_START` | 0 | Transfer فشل قبل ما يبدأ (fallback to PIO) |
| `SPI_TRANS_FAIL_IO` | 1 | Transfer فشل بـ I/O error |

#### الـ config options المهمة

| Config | التأثير |
|---|---|
| `CONFIG_SPI_SLAVE` | بيفعّل دعم target/slave mode |
| `CONFIG_SPI_DYNAMIC` | بيسمح بإضافة/إزالة devices بشكل ديناميكي |
| `CONFIG_HAS_DMA` | بيفعّل الـ DMA mapping في الـ core |
| `CONFIG_OF_DYNAMIC` | بيفعّل إضافة SPI devices من الـ DT بشكل ديناميكي |

---

### 1. الـ Structs المهمة

#### `struct spi_device`

**الغرض:** يمثل device واحد على الـ SPI bus — الجانب الـ controller-side proxy للـ target chip. كل chip مربوطة بـ chip select معين.

| Field | النوع | الشرح |
|---|---|---|
| `dev` | `struct device` | الـ device model — أساس الـ hierarchy |
| `controller` | `*spi_controller` | الـ controller اللي مربوط بيه |
| `max_speed_hz` | `u32` | أقصى سرعة clock للـ chip دي |
| `bits_per_word` | `u8` | حجم الـ word الافتراضي (8 bits إذا 0) |
| `rt` | `bool` | لو true → pump thread بيشتغل بـ realtime priority |
| `mode` | `u32` | الـ SPI mode flags (CPOL, CPHA, CS_HIGH، إلخ) |
| `irq` | `int` | رقم الـ IRQ الخاص بالـ device |
| `controller_state` | `void*` | حالة runtime خاصة بالـ controller |
| `controller_data` | `void*` | بيانات board-specific للـ controller |
| `modalias` | `char[]` | اسم الـ driver المطلوب (للـ hotplug) |
| `driver_override` | `const char*` | لو مكتوب اسم driver → الـ binding بيتقيد بيه |
| `pcpu_statistics` | `__percpu *` | إحصائيات per-CPU |
| `word_delay` | `spi_delay` | تأخير بين الـ words |
| `cs_setup` | `spi_delay` | تأخير بعد assert الـ CS |
| `cs_hold` | `spi_delay` | تأخير قبل deassert الـ CS |
| `cs_inactive` | `spi_delay` | تأخير بعد deassert الـ CS |
| `chip_select[]` | `u8[4]` | مصفوفة الـ physical chip select lines |
| `num_chipselect` | `u8` | عدد الـ CS المستخدمين |
| `cs_index_mask` | `u32:4` | bitmask للـ CS الفعالة |
| `cs_gpiod[]` | `*gpio_desc[4]` | GPIO descriptors للـ CS (أو NULL) |
| `tx_lane_map[]` | `u8[8]` | تعيين lanes الإرسال |
| `rx_lane_map[]` | `u8[8]` | تعيين lanes الاستقبال |

**العلاقات:**
- يحتوي على pointer لـ `spi_controller` (parent)
- الـ `dev.parent` = `&controller->dev`
- الـ `dev.bus` = `&spi_bus_type`

---

#### `struct spi_controller`

**الغرض:** يمثل الـ SPI host controller (أو target controller في slave mode). هو اللي بيدير الـ bus فعليًا، عنده الـ queue والـ kthread والـ ops.

| Field | النوع | الشرح |
|---|---|---|
| `dev` | `struct device` | الـ device model |
| `list` | `list_head` | للربط في الـ global `spi_controller_list` |
| `bus_num` | `s16` | رقم الـ SPI bus (-1 للـ dynamic allocation) |
| `num_chipselect` | `u16` | عدد الـ chip selects المتاحة |
| `num_data_lanes` | `u16` | عدد data lanes (default 1) |
| `mode_bits` | `u32` | الـ mode flags اللي الـ controller بيدعمها |
| `bits_per_word_mask` | `u32` | bitmask للـ BPW المدعومة |
| `min_speed_hz` | `u32` | أقل سرعة clock مدعومة |
| `max_speed_hz` | `u32` | أقصى سرعة clock مدعومة |
| `flags` | `u16` | controller constraints (HALF_DUPLEX، إلخ) |
| `target` | `bool` | true لو SPI target/slave controller |
| `devm_allocated` | `bool` | الـ allocation بيتعمل release تلقائي |
| `io_mutex` | `mutex` | يحمي الـ physical bus access |
| `add_lock` | `mutex` | يمنع إضافة نفس الـ CS مرتين |
| `bus_lock_spinlock` | `spinlock_t` | للـ bus locking الـ spinlock part |
| `bus_lock_mutex` | `mutex` | للـ bus locking الـ mutex part |
| `bus_lock_flag` | `bool` | لو true → الـ bus محجوز exclusive |
| `setup` | `fn ptr` | callback لتهيئة الـ device mode/clock |
| `transfer` | `fn ptr` | إضافة message للـ queue (unqueued drivers) |
| `cleanup` | `fn ptr` | تحرير controller-specific state |
| `can_dma` | `fn ptr` | بيقرر لو الـ transfer ممكن يتعمل بـ DMA |
| `queued` | `bool` | الـ driver بيستخدم الـ generic queue |
| `kworker` | `*kthread_worker` | الـ kthread اللي بيشيل الـ pump |
| `pump_messages` | `kthread_work` | الـ work item اللي بيشغل الـ pump |
| `queue_lock` | `spinlock_t` | يحمي الـ queue و الـ state |
| `queue` | `list_head` | قائمة الـ spi_message المنتظرة |
| `cur_msg` | `*spi_message` | الـ message اللي بيتشتغل دلوقتي |
| `cur_msg_completion` | `completion` | للـ synchronization على الـ cur_msg |
| `xfer_completion` | `completion` | للانتظار بعد `transfer_one` |
| `busy` | `bool` | الـ pump بيشتغل |
| `running` | `bool` | الـ queue شغّالة |
| `rt` | `bool` | الـ pump thread بـ realtime priority |
| `auto_runtime_pm` | `bool` | الـ core بيدير الـ PM تلقائيًا |
| `queue_empty` | `bool` | الـ queue فاضية (للـ fast path في spi_sync) |
| `must_async` | `bool` | بيعطّل الـ fast paths كلها |
| `fallback` | `bool` | fallback لـ PIO لو DMA فشل |
| `last_cs[]` | `s8[4]` | آخر CS اتفعّل (SPI_INVALID_CS = غير محدد) |
| `last_cs_mode_high` | `bool` | كان الـ mode بيستخدم CS_HIGH؟ |
| `max_dma_len` | `size_t` | أقصى حجم DMA transfer |
| `prepare_transfer_hardware` | `fn ptr` | callback قبل أول transfer |
| `unprepare_transfer_hardware` | `fn ptr` | callback بعد آخر transfer |
| `transfer_one_message` | `fn ptr` | بيعمل transfer لـ message كاملة |
| `transfer_one` | `fn ptr` | بيعمل transfer لـ spi_transfer واحد |
| `prepare_message` | `fn ptr` | setup قبل الـ message (e.g. DMA map) |
| `unprepare_message` | `fn ptr` | cleanup بعد الـ message |
| `optimize_message` | `fn ptr` | pre-optimization للـ message |
| `unoptimize_message` | `fn ptr` | تحرير موارد الـ optimization |
| `set_cs` | `fn ptr` | تحكم في الـ CS line |
| `set_cs_timing` | `fn ptr` | ضبط CS timing delays |
| `handle_err` | `fn ptr` | معالجة الـ error أثناء transfer |
| `mem_ops` | `*spi_controller_mem_ops` | SPI memory operations محسّنة |
| `cs_gpiods` | `**gpio_desc` | مصفوفة GPIO descriptors للـ CS |
| `use_gpio_descriptors` | `bool` | الـ core بيجيب الـ GPIO descriptors تلقائيًا |
| `dma_tx` | `*dma_chan` | الـ DMA TX channel |
| `dma_rx` | `*dma_chan` | الـ DMA RX channel |
| `dummy_rx` | `void*` | buffer وهمي للـ RX لما مفيش buffer |
| `dummy_tx` | `void*` | buffer وهمي للـ TX لما مفيش buffer |
| `pcpu_statistics` | `__percpu *` | إحصائيات per-CPU |
| `ptp_sts_supported` | `bool` | الـ driver بيدعم PTP timestamping |
| `irq_flags` | `unsigned long` | حالة الـ IRQ أثناء PTP timestamping |
| `dtr_caps` | `bool` | يدعم DDR (dual transfer rate) |
| `defer_optimize_message` | `bool` | تأجيل الـ optimization لوقت الـ transfer |

**العلاقات:**
- أبو كل الـ `spi_device` children
- مربوط في الـ global `spi_controller_list`
- عنده `kthread_worker` يشيل الـ `pump_messages` work

---

#### `struct spi_transfer`

**الغرض:** يمثل عملية قراءة/كتابة واحدة على الـ bus. الـ full-duplex يعني TX وRX بيحصلوا في نفس الوقت. الـ spi_message بتتكون من list من الـ transfers.

| Field | النوع | الشرح |
|---|---|---|
| `tx_buf` | `const void*` | بيانات الإرسال (DMA-safe) أو NULL |
| `rx_buf` | `void*` | buffer الاستقبال (DMA-safe) أو NULL |
| `len` | `unsigned` | حجم الـ buffers بالـ bytes |
| `error` | `u16` | error flags (FAIL_NO_START, FAIL_IO) |
| `tx_sg_mapped` | `bool` | الـ tx_sg اتعمله DMA map |
| `rx_sg_mapped` | `bool` | الـ rx_sg اتعمله DMA map |
| `tx_sg` | `sg_table` | Scatterlist للـ TX DMA |
| `rx_sg` | `sg_table` | Scatterlist للـ RX DMA |
| `tx_dma` | `dma_addr_t` | DMA address للـ TX |
| `rx_dma` | `dma_addr_t` | DMA address للـ RX |
| `cs_off` | `bit` | لو 1 → الـ CS بيكون off أثناء الـ transfer |
| `cs_change` | `bit` | toggle الـ CS بعد الـ transfer |
| `dummy_data` | `bit` | الـ transfer ده dummy bytes |
| `tx_nbits` | `4 bits` | عدد TX data lines (SINGLE/DUAL/QUAD/OCTAL) |
| `rx_nbits` | `4 bits` | عدد RX data lines |
| `bits_per_word` | `u8` | حجم الـ word لهذا الـ transfer |
| `speed_hz` | `u32` | سرعة الـ clock لهذا الـ transfer |
| `effective_speed_hz` | `u32` | السرعة الفعلية اللي اتنفّذت (بعد الـ transfer) |
| `delay` | `spi_delay` | تأخير بعد الـ transfer |
| `cs_change_delay` | `spi_delay` | تأخير بين deassert وassert عند cs_change |
| `word_delay` | `spi_delay` | تأخير بين الـ words |
| `offload_flags` | `unsigned int` | flags خاصة بالـ offload transfers |
| `ptp_sts_word_pre` | `unsigned int` | Word offset للـ PTP pre-timestamp |
| `ptp_sts_word_post` | `unsigned int` | Word offset للـ PTP post-timestamp |
| `ptp_sts` | `*ptp_system_timestamp` | Pointer لـ PTP timestamp structure |
| `timestamped` | `bit` | الـ transfer اتعمله timestamp |
| `dtr_mode` | `bool` | استخدام DDR mode |
| `transfer_list` | `list_head` | للربط في `spi_message.transfers` |

**العلاقات:**
- بتنتمي لـ `spi_message` عن طريق الـ `transfer_list`
- الـ `tx_sg` و`rx_sg` بيتبنوا أثناء `__spi_map_msg()`

---

#### `struct spi_message`

**الغرض:** يمثل transaction atomic كامل — مجموعة transfers بتتنفّذ بدون interruption. الـ chip select بيفضل active طول الـ message إلا لو `cs_change` اتستخدم.

| Field | النوع | الشرح |
|---|---|---|
| `transfers` | `list_head` | رأس قائمة الـ spi_transfer objects |
| `spi` | `*spi_device` | الـ device اللي الـ message مبعوتة ليه |
| `pre_optimized` | `bool` | الـ peripheral driver عمل optimize قبل الإرسال |
| `optimized` | `bool` | الـ core عمل optimize |
| `prepared` | `bool` | `prepare_message` اتعمل |
| `status` | `int` | 0 نجاح، negative error code للفشل |
| `complete` | `fn ptr` | callback لما الـ message تخلص |
| `context` | `void*` | argument الـ complete callback |
| `frame_length` | `unsigned` | إجمالي الـ bytes في الـ message |
| `actual_length` | `unsigned` | الـ bytes اللي اتنقلت فعليًا |
| `queue` | `list_head` | للربط في `spi_controller.queue` |
| `state` | `void*` | state خاص بالـ controller driver |
| `opt_state` | `void*` | state خاص بـ optimize/unoptimize |
| `offload` | `*spi_offload` | offload instance (اختياري) |
| `resources` | `list_head` | قائمة الـ spi_res المخصصة |

---

#### `struct spi_driver`

**الغرض:** يمثل الـ protocol driver — الطرف اللي بيتكلم مع الـ chip فوق طبقة الـ SPI.

| Field | النوع | الشرح |
|---|---|---|
| `id_table` | `*spi_device_id` | قائمة الـ device IDs المدعومة |
| `probe` | `fn ptr` | بيتستدعى لما device يتطابق مع الـ driver |
| `remove` | `fn ptr` | بيتستدعى لما الـ device يتفك |
| `shutdown` | `fn ptr` | بيتستدعى عند إيقاف النظام |
| `driver` | `device_driver` | الـ driver الـ base struct |

---

#### `struct spi_board_info`

**الغرض:** template مكتوب في الـ board code لوصف SPI device محدد. بيتحوّل لـ `spi_device` لما الـ controller يتسجّل.

| Field | النوع | الشرح |
|---|---|---|
| `modalias` | `char[]` | اسم الـ driver للـ device |
| `platform_data` | `const void*` | بيانات platform_data خاصة بالـ device |
| `swnode` | `*software_node` | software node للـ DT-less systems |
| `controller_data` | `void*` | hints للـ controller (مثلاً FIFO config) |
| `irq` | `int` | رقم الـ IRQ |
| `max_speed_hz` | `u32` | أقصى سرعة |
| `bus_num` | `u16` | رقم الـ SPI bus |
| `chip_select` | `u16` | رقم الـ CS |
| `mode` | `u32` | SPI mode flags |

---

#### `struct boardinfo` (داخلية في spi.c)

**الغرض:** wrapper داخلي لـ `spi_board_info` لتخزينها في الـ global `board_list`.

```c
struct boardinfo {
    struct list_head     list;       /* ربط في board_list */
    struct spi_board_info board_info; /* الـ board info الأصلي */
};
```

---

#### `struct spi_statistics`

**الغرض:** per-CPU counters لقياس الأداء وتتبع الأخطاء — تظهر في `/sys/bus/spi/devices/.../statistics/`.

| Field | النوع | الشرح |
|---|---|---|
| `syncp` | `u64_stats_sync` | seqcount للحماية على 32-bit systems |
| `messages` | `u64_stats_t` | عدد الـ messages |
| `transfers` | `u64_stats_t` | عدد الـ transfers |
| `errors` | `u64_stats_t` | عدد الـ errors |
| `timedout` | `u64_stats_t` | عدد الـ timeouts |
| `spi_sync` | `u64_stats_t` | عدد استدعاءات `spi_sync` |
| `spi_sync_immediate` | `u64_stats_t` | استدعاءات spi_sync اتنفّذت على طول بدون queue |
| `spi_async` | `u64_stats_t` | عدد استدعاءات `spi_async` |
| `bytes` | `u64_stats_t` | إجمالي البيانات المنقولة |
| `bytes_tx` | `u64_stats_t` | البيانات المرسلة |
| `bytes_rx` | `u64_stats_t` | البيانات المستقبَلة |
| `transfer_bytes_histo[17]` | `u64_stats_t[]` | histogram لأحجام الـ transfers |
| `transfers_split_maxsize` | `u64_stats_t` | عدد الـ transfers اللي اتقسمت |

---

#### `struct spi_delay`

**الغرض:** وصف تأخير زمني مع وحدة القياس.

```c
struct spi_delay {
    u16 value;  /* القيمة */
    u8  unit;   /* SPI_DELAY_UNIT_USECS / NSECS / SCK */
};
```

---

#### `struct spi_res`

**الغرض:** resource management مرتبط بـ `spi_message` — بيتحرر تلقائيًا لما الـ message تخلص (مشابه لـ devres بس لمدة حياة الـ message).

```c
struct spi_res {
    struct list_head        entry;    /* ربط في message->resources */
    spi_res_release_t       release;  /* callback عند التحرير */
    unsigned long long      data[];   /* بيانات extra (ULL-aligned) */
};
```

---

#### `struct spi_replaced_transfers`

**الغرض:** تسجيل عملية استبدال transfers داخل message — يُستخدم لـ splitting وlmax handling مع إمكانية الـ revert.

| Field | الشرح |
|---|---|
| `release` | extra release callback |
| `extradata` | pointer لبيانات extra |
| `replaced_transfers` | قائمة الـ transfers اللي اتشالت |
| `replaced_after` | نقطة الإعادة في القائمة |
| `inserted` | عدد الـ transfers الجديدة |
| `inserted_transfers[]` | مصفوفة الـ transfers البديلة |

---

#### `struct acpi_spi_lookup` (داخلية)

**الغرض:** مؤقتة لتجميع بيانات SPI device من الـ ACPI _CRS resources.

```c
struct acpi_spi_lookup {
    struct spi_controller *ctlr;
    u32 max_speed_hz;
    u32 mode;
    int irq;
    u8  bits_per_word;
    u8  chip_select;
    int n;      /* عداد الـ resources */
    int index;  /* الـ index المطلوب (-1 = أي) */
};
```

---

### 2. مخططات العلاقات بين الـ Structs

```
spi_bus_type
    │
    ├─► spi_controller (class: spi_master / spi_slave)
    │       │
    │       │  .dev.parent ───► platform_device / pci_device / etc.
    │       │
    │       ├─► spi_device  (dev.parent = &ctlr->dev)
    │       │       │
    │       │       │   .controller ──────────────────────────────┐
    │       │       │   .pcpu_statistics → spi_statistics (percpu) │
    │       │       │   .cs_gpiod[] → gpio_desc[]                  │
    │       │       └── .dev (bus=spi_bus_type)                    │
    │       │                                                       │
    │       ├─► spi_device (another child)                         │
    │       │                                                       │
    │       │   .queue ──► spi_message ──► spi_message ──► ...     │
    │       │                   │                                   │
    │       │                   │  .transfers ──► spi_transfer      │
    │       │                   │                      │            │
    │       │                   │                      └── .tx_sg   │
    │       │                   │                          .rx_sg   │
    │       │                   │  .spi ──────────────────────────► spi_device (back ref)
    │       │                   │  .resources ──► spi_res ──► spi_res
    │       │                   └── .queue (list_head in ctlr->queue)
    │       │
    │       ├── .kworker ──► kthread_worker
    │       │       └── .pump_messages (kthread_work)
    │       │
    │       ├── .dma_tx ──► dma_chan
    │       ├── .dma_rx ──► dma_chan
    │       └── .pcpu_statistics → spi_statistics (percpu)
    │
    └─► spi_driver
            │
            └── .driver (device_driver)
                    └── .bus = &spi_bus_type
```

```
board_list (global)
    │
    └── boardinfo ──► boardinfo ──► ...
            │
            └── .board_info (spi_board_info)
                        │
                        └── بيتحوّل لـ spi_device عند تسجيل controller

spi_controller_list (global)
    └── spi_controller ──► spi_controller ──► ...
              (مربوطين بـ board_lock mutex)

spi_controller_idr (global IDR)
    └── bus_num → spi_controller
```

---

### 3. دورة حياة الـ spi_controller

```
alloc stage:
  __spi_alloc_controller(dev, size, target)
      │
      ├── kzalloc(ctlr + driver_data)
      ├── device_initialize(&ctlr->dev)
      ├── INIT_LIST_HEAD(&ctlr->queue)
      ├── spin_lock_init(&ctlr->queue_lock)
      ├── mutex_init(&ctlr->io_mutex)
      ├── mutex_init(&ctlr->bus_lock_mutex)
      ├── mutex_init(&ctlr->add_lock)
      └── ctlr->bus_num = -1 (dynamic)

register stage:
  spi_register_controller(ctlr)
      │
      ├── spi_controller_check_ops()         ← التحقق من وجود ops مطلوبة
      ├── spi_controller_id_alloc()          ← حجز bus_num من IDR
      ├── init_completion(&ctlr->xfer_completion)
      ├── spi_get_gpio_descs()               ← جلب GPIO CS descriptors
      ├── device_add(&ctlr->dev)             ← تسجيل في sysfs
      ├── spi_controller_initialize_queue()  ← إنشاء kthread وبدء queue
      │       ├── ctlr->transfer = spi_queued_transfer
      │       ├── spi_init_queue()
      │       │       ├── kthread_run_worker() → ctlr->kworker
      │       │       └── kthread_init_work(&ctlr->pump_messages, spi_pump_messages)
      │       └── spi_start_queue()
      │               └── kthread_queue_work(ctlr->kworker, &ctlr->pump_messages)
      ├── spi_alloc_pcpu_stats()             ← إحصائيات
      ├── list_add_tail(&ctlr->list, &spi_controller_list)
      ├── spi_match_controller_to_boardinfo() ← تطابق مع board_info المسبق
      ├── of_register_spi_devices()          ← إنشاء devices من DT
      └── acpi_register_spi_devices()        ← إنشاء devices من ACPI

unregister stage:
  spi_unregister_controller(ctlr)
      │
      ├── device_for_each_child(..., __unregister) ← unregister كل الـ children
      ├── spi_destroy_queue()
      │       ├── spi_stop_queue()           ← انتظار خلوص الـ queue
      │       └── kthread_destroy_worker()
      ├── list_del(&ctlr->list)
      ├── device_del(&ctlr->dev)
      ├── idr_remove(&spi_controller_idr, id)
      └── put_device(&ctlr->dev)             ← إذا مش devm
```

---

### 4. دورة حياة الـ spi_device

```
alloc:
  spi_alloc_device(ctlr)
      │
      ├── spi_controller_get(ctlr)           ← ref++
      ├── kzalloc(spi_device)
      ├── spi_alloc_pcpu_stats()
      ├── spi->controller = ctlr
      ├── spi->dev.parent = &ctlr->dev
      ├── spi->dev.bus = &spi_bus_type
      ├── spi->dev.release = spidev_release
      └── device_initialize(&spi->dev)

configure & add:
  spi_add_device(spi)
      │
      ├── mutex_lock(&ctlr->add_lock)
      ├── __spi_add_device(spi)
      │       ├── validate CS range
      │       ├── spi_dev_check()            ← فحص تعارض الـ CS
      │       ├── spi_dev_set_name()         ← "spi0.1" مثلاً
      │       ├── assign cs_gpiods from ctlr->cs_gpiods
      │       ├── spi_setup(spi)             ← تهيئة الـ hardware
      │       └── device_add(&spi->dev)      ← تسجيل في sysfs → probe
      └── mutex_unlock(&ctlr->add_lock)

teardown:
  spi_unregister_device(spi)
      │
      ├── of_node_clear_flag() / acpi_device_clear_enumerated()
      ├── device_del(&spi->dev)
      ├── spi_cleanup(spi)                   ← ctlr->cleanup(spi)
      └── put_device(&spi->dev)
              └── spidev_release()
                      ├── spi_controller_put(spi->controller)
                      ├── kfree(spi->driver_override)
                      ├── free_percpu(spi->pcpu_statistics)
                      └── kfree(spi)
```

---

### 5. تدفق نقل البيانات

#### 5.1 الـ spi_sync path (المسار الشائع)

```
driver calls: spi_sync(spi, msg)
    │
    ├── mutex_lock(&ctlr->bus_lock_mutex)
    └── __spi_sync(spi, msg)
            │
            ├── __spi_check_suspended()      ← فحص suspend
            ├── spi_maybe_optimize_message() ← validate + split transfers
            │       └── __spi_optimize_message()
            │               ├── __spi_validate()       ← فحص bits/word, speed, nbits
            │               ├── spi_split_transfers()  ← تقسيم لو حجم > max
            │               └── ctlr->optimize_message() ← driver hook
            │
            ├── [fast path: queue_empty && !must_async]
            │       └── __spi_transfer_message_noqueue(ctlr, msg)
            │               └── __spi_pump_transfer_message(ctlr, msg, was_busy)
            │
            └── [slow path: queue not empty]
                    ├── message->complete = spi_complete
                    ├── __spi_async() → ctlr->transfer(spi, msg)
                    │       └── spi_queued_transfer() → adds to queue
                    │               └── kthread_queue_work(pump_messages)
                    └── wait_for_completion(&done)
```

#### 5.2 الـ pump loop (kthread)

```
spi_pump_messages(work)
    └── __spi_pump_messages(ctlr, in_kthread=true)
            │
            ├── mutex_lock(&ctlr->io_mutex)
            ├── spin_lock_irqsave(&ctlr->queue_lock)
            ├── [لو queue فاضية] → idle + unprepare_hw
            ├── msg = list_first_entry(&ctlr->queue)
            ├── ctlr->cur_msg = msg
            ├── spin_unlock_irqrestore
            │
            └── __spi_pump_transfer_message(ctlr, msg, was_busy)
                    │
                    ├── pm_runtime_get_sync()          ← wake up hardware
                    ├── ctlr->prepare_transfer_hardware()
                    ├── ctlr->prepare_message()
                    ├── spi_map_msg()                  ← DMA mapping
                    │       └── __spi_map_msg()
                    │               └── spi_map_buf_attrs() per transfer
                    │
                    ├── ctlr->transfer_one_message()   ← أو الـ default:
                    │       └── spi_transfer_one_message()
                    │               │
                    │               ├── spi_set_cs(spi, true)   ← assert CS
                    │               ├── [for each transfer]:
                    │               │       ├── spi_dma_sync_for_device()
                    │               │       ├── ctlr->transfer_one()    ← hardware I/O
                    │               │       ├── [لو async] wait_for_completion(xfer_completion)
                    │               │       ├── spi_dma_sync_for_cpu()
                    │               │       └── handle cs_change / delays
                    │               ├── spi_set_cs(spi, false)  ← deassert CS
                    │               └── spi_finalize_current_message()
                    │
                    └── spi_finalize_current_message(ctlr)
                            ├── spi_unmap_msg()
                            ├── ctlr->unprepare_message()
                            ├── spi_maybe_unoptimize_message()
                            ├── complete(&ctlr->cur_msg_completion)
                            └── msg->complete(msg->context)  ← callback
```

#### 5.3 الـ spi_async path

```
driver calls: spi_async(spi, msg)
    │
    ├── spi_maybe_optimize_message()
    ├── spin_lock_irqsave(&ctlr->bus_lock_spinlock)
    ├── [لو bus_lock_flag] → return -EBUSY
    └── __spi_async(spi, msg)
            │
            ├── SPI_STATISTICS_INCREMENT_FIELD(spi_async)
            └── ctlr->transfer(spi, msg)
                    └── spi_queued_transfer()
                            ├── spin_lock_irqsave(&ctlr->queue_lock)
                            ├── list_add_tail(&msg->queue, &ctlr->queue)
                            ├── ctlr->queue_empty = false
                            └── kthread_queue_work(pump_messages) [لو مش busy]
```

#### 5.4 تدفق الـ CS (Chip Select)

```
spi_set_cs(spi, enable, force)
    │
    ├── [check: same as last CS? skip]
    ├── trace_spi_set_cs()
    ├── update ctlr->last_cs[]
    │
    ├── [GPIO CS path]:
    │       spi_for_each_valid_cs(spi, idx)
    │           └── spi_toggle_csgpiod(spi, idx, enable, activate)
    │                   ├── gpiod_set_value_cansleep()
    │                   └── spi_delay_exec(cs_setup / cs_inactive)
    │
    └── [hardware CS path]:
            ctlr->set_cs(spi, !enable)
```

---

### 6. استراتيجية الـ Locking

#### جدول الـ Locks وما تحميه

| Lock | النوع | يحمي |
|---|---|---|
| `board_lock` | `mutex` (global) | `board_list`، `spi_controller_list`، `spi_controller_idr` |
| `ctlr->io_mutex` | `mutex` | الـ physical bus access — منع تداخل الـ transfers |
| `ctlr->add_lock` | `mutex` | منع إضافة نفس الـ CS مرتين |
| `ctlr->bus_lock_mutex` | `mutex` | الـ exclusive bus lock (`spi_bus_lock`) |
| `ctlr->bus_lock_spinlock` | `spinlock` | `ctlr->bus_lock_flag` — قابل للاستخدام من IRQ |
| `ctlr->queue_lock` | `spinlock` | `ctlr->queue`، `ctlr->cur_msg`، `ctlr->busy`، `ctlr->running`، `ctlr->queue_empty` |
| `spi_statistics.syncp` | `u64_stats_sync` | الـ statistics counters على 32-bit |

#### Lock Ordering (الترتيب الصحيح)

```
bus_lock_mutex
    └── io_mutex
            └── queue_lock (spinlock — irqsave)
```

> **القاعدة:** مش مسموح تمسك `queue_lock` وانت بتحاول تمسك `io_mutex` — ده هيعمل deadlock.
> الـ `bus_lock_spinlock` ممكن يتمسك من interrupt context.
> الـ `board_lock` مستقل تمامًا عن الـ transfer locks.

#### تفاصيل الـ locking أثناء الـ pump

```
__spi_pump_messages():
  1. mutex_lock(io_mutex)             ← منع تداخل transfers
  2. spin_lock_irqsave(queue_lock)    ← فحص وسحب من الـ queue
  3. spin_unlock_irqrestore(queue_lock)
  4. __spi_pump_transfer_message()    ← تنفيذ الـ transfer (io_mutex ممسوك)
  5. mutex_unlock(io_mutex)
```

#### الـ cur_msg_incomplete / cur_msg_need_completion (lock-free optimization)

الـ core بيستخدم `WRITE_ONCE` و `READ_ONCE` مع `smp_wmb()`/`smp_mb()` لتجنب استخدام completion إلا لما يكون في race فعلي:

```c
/* في __spi_pump_transfer_message: */
WRITE_ONCE(ctlr->cur_msg_incomplete, true);
WRITE_ONCE(ctlr->cur_msg_need_completion, false);
smp_wmb();
/* ... transfer ... */
WRITE_ONCE(ctlr->cur_msg_need_completion, true);
smp_mb();
if (READ_ONCE(ctlr->cur_msg_incomplete))
    wait_for_completion(&ctlr->cur_msg_completion);

/* في spi_finalize_current_message: */
WRITE_ONCE(ctlr->cur_msg_incomplete, false);
smp_mb();
if (READ_ONCE(ctlr->cur_msg_need_completion))
    complete(&ctlr->cur_msg_completion);
```

هذا الـ pattern بيضمن ordering صحيح بدون overhead الـ spinlock في الـ common case.

---

### ملخص العلاقات الجوهرية

```
spi_board_info ──(spi_register_board_info)──► boardinfo (in board_list)
                                                    │
                              (spi_match_controller_to_boardinfo)
                                                    │
                                                    ▼
spi_controller ──(alloc+register)──► spi_controller (in spi_controller_list)
       │                                    │
       │                            ┌───────┤
       │                            │       │
       ▼                            ▼       ▼
spi_device ◄──── of/acpi/board   queue   kthread
   │
   └──(spi_async/spi_sync)──► spi_message
                                   │
                                   └──► spi_transfer ──► spi_transfer ──► ...
                                              │
                                         tx_sg / rx_sg (DMA scatterlists)
```
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Group 1: Driver & Bus Registration

| Function | Signature (مختصرة) | الغرض |
|---|---|---|
| `__spi_register_driver` | `(module, spi_driver*) → int` | يسجل driver على الـ SPI bus |
| `spi_unregister_driver` | `(spi_driver*) → void` | يلغي تسجيل الـ driver |
| `spi_init` | `(void) → int` | يهيئ الـ SPI subsystem كله |

#### Group 2: Device Allocation & Registration

| Function | الغرض |
|---|---|
| `spi_alloc_device` | يخصص `spi_device` جديد |
| `spi_add_device` | يضيفه للـ bus |
| `spi_new_device` | alloc + add في خطوة واحدة |
| `spi_unregister_device` | يشيل الـ device من الـ bus |
| `spi_register_board_info` | يسجل board-level static devices |
| `spi_new_ancillary_device` | يسجل device ثانوي (نفس الـ chip) |

#### Group 3: Controller Allocation & Registration

| Function | الغرض |
|---|---|
| `__spi_alloc_controller` | يخصص `spi_controller` (host أو target) |
| `__devm_spi_alloc_controller` | نفسه لكن بـ devres |
| `spi_register_controller` | يسجل الـ controller على الـ system |
| `devm_spi_register_controller` | نفسه لكن بـ devres |
| `spi_unregister_controller` | يشيل الـ controller |

#### Group 4: Transfer / Message API

| Function | الغرض |
|---|---|
| `spi_setup` | يضبط mode/clock/bits للـ device |
| `spi_async` | يبعت message بشكل async |
| `spi_sync` | يبعت message بشكل sync (blocking) |
| `spi_sync_locked` | sync مع exclusive bus lock |
| `spi_bus_lock` / `spi_bus_unlock` | يقفل/يفتح الـ bus للاستخدام الحصري |
| `spi_write_then_read` | write ثم read في transaction واحد |

#### Group 5: Message Optimization

| Function | الغرض |
|---|---|
| `spi_optimize_message` | يعمل one-time setup للـ message |
| `spi_unoptimize_message` | يحرر resources الـ optimization |
| `devm_spi_optimize_message` | managed version |

#### Group 6: Transfer Splitting Helpers

| Function | الغرض |
|---|---|
| `spi_split_transfers_maxsize` | يكسر الـ transfers لـ chunks حسب bytes |
| `spi_split_transfers_maxwords` | يكسرها حسب عدد الـ words |

#### Group 7: Timing & CS Helpers

| Function | الغرض |
|---|---|
| `spi_delay_to_ns` | يحول `spi_delay` لـ nanoseconds |
| `spi_delay_exec` | ينفذ delay فعلي |
| `spi_transfer_cs_change_delay_exec` | ينفذ delay بعد CS change |
| `spi_finalize_current_transfer` | يبلّغ الـ core إن الـ transfer خلص |
| `spi_finalize_current_message` | يبلّغ إن الـ message خلص بالكامل |

#### Group 8: PTP Timestamping

| Function | الغرض |
|---|---|
| `spi_take_timestamp_pre` | يسجل timestamp قبل الـ TX |
| `spi_take_timestamp_post` | يسجل timestamp بعد الـ TX |

#### Group 9: ACPI Support

| Function | الغرض |
|---|---|
| `acpi_spi_count_resources` | يعد SpiSerialBus resources في ACPI |
| `acpi_spi_device_alloc` | يخصص `spi_device` من ACPI node |
| `acpi_spi_find_controller_by_adev` | يلاقي الـ controller من ACPI device |

#### Group 10: DMA Mapping

| Function | الغرض |
|---|---|
| `spi_map_buf` | يعمل DMA map لـ buffer |
| `spi_unmap_buf` | يعمل unmap |

#### Group 11: Queue Management (Internal)

| Function | الغرض |
|---|---|
| `spi_get_next_queued_message` | يجيب أول message في الـ queue |
| `spi_flush_queue` | ينهي كل الـ pending messages من الـ context الحالي |
| `spi_controller_suspend` / `spi_controller_resume` | suspend/resume للـ controller |

---

### Group 1: Driver & Bus Registration

الـ SPI subsystem بيعتمد على `spi_bus_type` اللي بيربط الـ `spi_device` بالـ `spi_driver`. الـ bus callbacks (`match`, `probe`, `remove`, `uevent`) بتتعامل مع matching عبر OF / ACPI / modalias.

---

#### `__spi_register_driver`

```c
int __spi_register_driver(struct module *owner, struct spi_driver *sdrv);
```

بيسجل الـ `spi_driver` على الـ `spi_bus_type`. بيعمل sanity check: لو الـ driver عنده `of_match_table` بس ماعندوش `spi_device_id` مقابل كل compatible string، بيطبع warning لأن module autoloading هيبقى بيشتغل بـ `spi:` modaliases مش `of:` modaliases.

**Parameters:**
- `owner`: الـ `THIS_MODULE` بتاع الـ driver
- `sdrv`: الـ `spi_driver` المراد تسجيله — لازم يكون فيه `driver.name` على الأقل

**Return:** 0 نجاح، error code سالب في الفشل

**Key details:**
- بيستدعي `driver_register(&sdrv->driver)` في النهاية
- الـ macro `spi_register_driver` بيمرر `THIS_MODULE` تلقائياً
- لا يحتاج locking لأن `driver_register` بيعمل ده جوّاه

**Caller context:** يُنادى من `module_init()` أو `module_spi_driver()` macro.

---

#### `spi_init`

```c
static int __init spi_init(void);
```

الـ `postcore_initcall` اللي بيهيئ الـ SPI subsystem كله:
1. يخصص `buf` (SPI_BUFSIZ bytes) لـ `spi_write_then_read`
2. يسجل `spi_bus_type`
3. يسجل `spi_controller_class`
4. لو `CONFIG_SPI_SLAVE` مفعّل، يسجل `spi_target_class`
5. يسجل OF/ACPI notifiers للـ dynamic device addition

**Return:** 0 أو negative error مع cleanup كامل على الـ error path.

---

### Group 2: Device Allocation & Registration

هنا بيتبنى الـ `spi_device` اللي بيمثّل peripheral واحد على الـ bus. الـ device بياخد refcount على الـ controller بتاعه.

---

#### `spi_alloc_device`

```c
struct spi_device *spi_alloc_device(struct spi_controller *ctlr);
```

بيخصص `spi_device` جديد وبيربطه بالـ controller. بيعمل:
- `kzalloc` للـ struct
- `spi_alloc_pcpu_stats(NULL)` لإنشاء per-CPU statistics
- يضبط `spi->dev.parent = &ctlr->dev`، `spi->dev.bus = &spi_bus_type`
- يرفع refcount على الـ controller بـ `spi_controller_get(ctlr)`
- `device_initialize` بدون `device_add` (الـ caller هو اللي يضيفه لما يجهّزه)

**Parameters:**
- `ctlr`: الـ controller اللي الـ device هيتربط بيه

**Return:** pointer للـ device الجديد، أو NULL لو فشل الـ alloc.

**Key details:**
- `spi->mode` بياخد قيمة `ctlr->buswidth_override_bits` كقيمة ابتدائية
- الـ caller مسؤول عن تعبئة باقي الـ fields ثم `spi_add_device()`
- لو مش هيضيف الـ device، لازم يعمل `spi_dev_put()` لتحرير الـ memory

**Caller context:** can sleep.

---

#### `__spi_add_device` (internal)

```c
static int __spi_add_device(struct spi_device *spi);
```

الـ internal core للـ device registration. بيعمل:

```
Validate num_chipselect <= SPI_DEVICE_CS_CNT_MAX
For each CS: validate cs < ctlr->num_chipselect
Check no duplicate CS values (internal self-check)
Initialize unused CS slots to SPI_INVALID_CS
Set device name (spi_dev_set_name)
bus_for_each_dev() → spi_dev_check() [check no conflict]
If ctlr->cs_gpiods: assign GPIO descriptors
Call spi_setup(spi)
device_add(&spi->dev)
```

بيُنادى دايماً من تحت الـ `ctlr->add_lock` mutex.

---

#### `spi_add_device`

```c
int spi_add_device(struct spi_device *spi);
```

الـ public wrapper حول `__spi_add_device`. بيمسك `ctlr->add_lock` mutex قبل ما يستدعي الـ internal function. الـ mutex ده بيمنع إضافة deviceين بنفس الـ chipselect في نفس الوقت.

**Return:** 0 نجاح، negative errno فشل.

---

#### `spi_new_device`

```c
struct spi_device *spi_new_device(struct spi_controller *ctlr,
                                  struct spi_board_info *chip);
```

الـ convenience function اللي بتعمل alloc + configure + add في خطوة واحدة من `spi_board_info`. بيُستخدم في board init code أو من drivers زي USB-to-SPI adapters.

**Parameters:**
- `ctlr`: الـ controller
- `chip`: descriptor بيحتوي على `modalias`, `chip_select`, `max_speed_hz`, `mode`, `irq`, `platform_data`, إلخ

**Return:** الـ `spi_device` الجديد أو NULL لو فشل.

**Key details:**
- لو `chip->swnode` موجود، بيعمل `device_add_software_node()` قبل التسجيل
- على error بيعمل cleanup للـ software node وبيعمل `spi_dev_put()`

---

#### `spi_unregister_device`

```c
void spi_unregister_device(struct spi_device *spi);
```

بيشيل الـ SPI device من الـ bus ويحرر resources:
1. لو OF node: `of_node_clear_flag(OF_POPULATED)` + `of_node_put`
2. لو ACPI: `acpi_device_clear_enumerated`
3. `device_remove_software_node`
4. `device_del` → unbind الـ driver
5. `spi_cleanup` → `ctlr->cleanup(spi)` لو موجود
6. `put_device` → قد يعمل `spidev_release` لو الـ refcount وصل صفر

**Caller context:** can sleep.

---

#### `spi_register_board_info`

```c
int spi_register_board_info(struct spi_board_info const *info, unsigned n);
```

بيسجل array من الـ SPI device descriptors في الـ global `board_list`. بيُنادى عادةً من `arch_initcall` أو board setup code.

**Logic:**
```
kcalloc(n, boardinfo)
For each entry:
    memcpy board_info
    mutex_lock(board_lock)
    list_add_tail to board_list
    For each existing controller: spi_match_controller_to_boardinfo()
    mutex_unlock
```

**Key details:**
- الـ `board_list` بيفضل موجود طول عمر الـ kernel — حتى لو الـ controller اتunload وأعاد تحميله، الـ devices هتتسجل تاني
- الـ `board_info` بيتعمله `memcpy` — لكن الـ embedded pointers (زي `platform_data`) بتتنسخ كـ raw pointers

---

### Group 3: Controller Allocation & Registration

الـ `spi_controller` هو الـ struct الرئيسي اللي بيمثل الـ SPI hardware. الـ allocation بياخد extra memory للـ driver-private data مباشرةً بعد الـ struct.

---

#### `__spi_alloc_controller`

```c
struct spi_controller *__spi_alloc_controller(struct device *dev,
                                              unsigned int size, bool target);
```

بيخصص `spi_controller` مع `size` bytes إضافية للـ driver-private data (cacheline-aligned). بيهيئ:
- جميع الـ locks: `queue_lock` (spinlock)، `bus_lock_spinlock`، `bus_lock_mutex`، `io_mutex`، `add_lock`
- `INIT_LIST_HEAD` للـ queue
- `ctlr->bus_num = -1` (سيتعيّن لاحقاً)
- `num_chipselect = 1`، `num_data_lanes = 1`
- الـ `dev.class` = `spi_controller_class` أو `spi_target_class` حسب `target`
- `pm_suspend_ignore_children` لمنع الـ PM من إيقاف الـ children قبل الـ controller

**Parameters:**
- `dev`: الـ physical device (platform device عادةً)
- `size`: حجم الـ driver-private data
- `target`: `true` لـ SPI target/slave mode

**Return:** pointer للـ controller، أو NULL.

**Key details:**
- الـ driver-private data accessible بـ `spi_controller_get_devdata()` / `spi_controller_set_devdata()`
- لا يُستخدم مباشرة — يُستخدم `spi_alloc_host()` أو `spi_alloc_target()` (macros تستدعيه)

---

#### `__devm_spi_alloc_controller`

```c
struct spi_controller *__devm_spi_alloc_controller(struct device *dev,
                                                   unsigned int size,
                                                   bool target);
```

نفس `__spi_alloc_controller` لكن بيستخدم `devm_add_action_or_reset` لتسجيل `devm_spi_release_controller` كـ cleanup action. لما الـ `dev` يتunbind، الـ controller بيتحرر تلقائياً. بيضبط `ctlr->devm_allocated = true` لمنع `spi_unregister_controller` من عمل `put_device` إضافية.

---

#### `spi_register_controller`

```c
int spi_register_controller(struct spi_controller *ctlr);
```

الأهم في الـ Group ده. بيسجل الـ controller على الـ system. الـ flow:

```
spi_controller_check_ops()  ← verify at least one transfer method
Determine bus_num:
    1. من OF alias ("spi0", "spi1", إلخ)
    2. أو static من ctlr->bus_num
    3. أو dynamic من idr_alloc()
init_completion(xfer_completion, cur_msg_completion)
dev_set_name("spi%u", ctlr->bus_num)
If use_gpio_descriptors: spi_get_gpio_descs()
Initialize last_cs[] = SPI_INVALID_CS
device_add(&ctlr->dev)
If transfer_one or transfer_one_message:
    spi_controller_initialize_queue()
spi_alloc_pcpu_stats() for controller
mutex_lock(board_lock)
list_add to spi_controller_list
Match against board_list entries
mutex_unlock
of_register_spi_devices()
acpi_register_spi_devices()
```

**Return:** 0 نجاح، negative errno + cleanup كامل (يشيل الـ bus_id، يدمر الـ queue) في الفشل.

**Key details:**
- الـ `spi_controller_check_ops` بتفحص: لو مفيش `mem_ops->exec_op`، لازم يكون في `transfer` أو `transfer_one` أو `transfer_one_message`
- **Locking**: بيمسك `board_lock` mutex لما يضيف للـ `spi_controller_list`
- بعد `device_add`، الـ userspace بيشوف الـ device

---

#### `spi_unregister_controller`

```c
void spi_unregister_controller(struct spi_controller *ctlr);
```

بيعكس `spi_register_controller`:

```
If CONFIG_SPI_DYNAMIC: lock add_lock
device_for_each_child() → spi_unregister_device() for all
If ctlr->queued: spi_destroy_queue()
mutex_lock(board_lock): list_del from spi_controller_list
device_del(&ctlr->dev)
idr_remove bus_num
If CONFIG_SPI_DYNAMIC: unlock add_lock
If !devm_allocated: put_device()
```

**Key details:**
- الـ `add_lock` بيمنع الـ race مع `spi_add_device` اللي ممكن يجري بشكل concurrent في `CONFIG_SPI_DYNAMIC`
- لو الـ controller اتخصص بـ `devm_spi_alloc_*`، مش محتاج تعمل `put_device` يدوياً

---

#### `spi_controller_suspend` / `spi_controller_resume`

```c
int spi_controller_suspend(struct spi_controller *ctlr);
int spi_controller_resume(struct spi_controller *ctlr);
```

**suspend**: بيوقف الـ queue بـ `spi_stop_queue()` (لو queued)، ثم بيعمل `__spi_mark_suspended()` اللي بتضيف `SPI_CONTROLLER_SUSPENDED` flag.

**resume**: بيشيل الـ flag ثم بيشغل الـ queue تاني بـ `spi_start_queue()`.

**Key details:**
- الـ `__spi_mark_suspended/resumed` بيمسكوا `bus_lock_mutex`
- الـ `__spi_check_suspended()` بيُنادى من `__spi_sync()` لـ early return بـ `-ESHUTDOWN`

---

### Group 4: Transfer / Message API

ده أهم group من ناحية الـ driver developers. بيوفر الـ API لإرسال البيانات.

---

#### `spi_setup`

```c
int spi_setup(struct spi_device *spi);
```

بيضبط الـ SPI mode والـ clock والـ bits_per_word للـ device. بيُنادى من الـ protocol driver في الـ `probe` أو لما يحتاج يغير الـ settings.

**Flow:**

```
Validate: no two of (DUAL, QUAD, NO_TX) together
Validate: no two of (DUAL, QUAD, NO_RX) together
Validate: SPI_3WIRE incompatible with multi-lane
Validate: MOSI_IDLE_LOW & MOSI_IDLE_HIGH not both set
bad_bits = mode & ~(ctlr->mode_bits | SPI_CS_HIGH | SPI_CS_WORD | ...)
ugly_bits = multi-lane bits in bad_bits → warn & strip (not fatal)
if remaining bad_bits: return -EINVAL
if !spi->bits_per_word: default = 8
Clamp spi->max_speed_hz to ctlr->max_speed_hz
mutex_lock(io_mutex)
ctlr->setup(spi)
spi_set_cs_timing(spi)
pm_runtime + spi_set_cs(spi, false, true)  ← deassert CS
mutex_unlock
If spi->rt && !ctlr->rt: spi_set_thread_rt(ctlr)
```

**Return:** 0 نجاح. الـ mutex بيتفك دايماً حتى على الـ error path.

**Key details:**
- بيمسك `io_mutex` — ده بيمنع concurrent setup مع active transfers
- `spi_set_cs(false, true)` — الـ `force=true` بيضمن deassert حتى لو CS مش اتغير
- الـ `ugly_bits` (multi-lane flags غير مدعومة) بتتشال بـ warning بدل ما يفشل — لأن الـ driver ممكن يكون generic

---

#### `spi_async`

```c
int spi_async(struct spi_device *spi, struct spi_message *message);
```

بيبعت الـ message بشكل asynchronous — بيرجع فوراً بدون انتظار.

**Flow:**

```
spi_maybe_optimize_message(spi, message)
spin_lock_irqsave(bus_lock_spinlock)
if ctlr->bus_lock_flag: return -EBUSY
else: __spi_async(spi, message)
spin_unlock_irqrestore
```

الـ `__spi_async` بداخله:
- بيرفع `spi_async` statistics
- trace `spi_message_submit`
- لو `!ctlr->ptp_sts_supported`: بيقرأ `ptp_read_system_prets` قبل الـ transfer
- `ctlr->transfer(spi, message)` — هنا بيضيف للـ queue

**Parameters:**
- `message`: لازم يكون فيه `complete` callback — هيُنادى من interrupt أو thread context عند الانتهاء

**Return:** 0 أو error. لو `ctlr->transfer` مش موجود: `-ENOTSUPP`.

**Key details:**
- يُستخدم في any context حتى interrupt context
- الـ completion callback بيُنادى من context مش ممكن ينام (can't sleep)
- لو الـ bus locked بـ `spi_bus_lock()`: بيرجع `-EBUSY`

---

#### `__spi_sync` (internal) و `spi_sync`

```c
int spi_sync(struct spi_device *spi, struct spi_message *message);
```

الـ blocking synchronous transfer. الـ internal `__spi_sync`:

```
Check !suspended
spi_maybe_optimize_message()
Increment spi_sync statistics
If queue_empty && !must_async:
    → Fast path: __spi_transfer_message_noqueue() (immediate execution)
    → return message->status
Else:
    → message->complete = spi_complete
    → message->context = &done (on-stack completion)
    → spin_lock: __spi_async(spi, message)
    → wait_for_completion(&done)
    → return message->status
```

**Key details:**
- الـ **fast path** هو الأهم للـ performance: لو الـ queue فاضية، الـ message بيتنفذ مباشرةً في الـ calling context بدون scheduling overhead
- الـ `spi_sync` بيمسك `bus_lock_mutex` — ده بيمنع concurrent `spi_sync` calls
- لو فيه async messages في الـ queue من نفس الـ context، بيتبعت للـ queue للـ ordering correctness
- لا يُستخدم في interrupt context أبداً

---

#### `spi_sync_locked`

```c
int spi_sync_locked(struct spi_device *spi, struct spi_message *message);
```

نفس `__spi_sync` بدون أخذ `bus_lock_mutex` — لأن الـ caller قبلها عمل `spi_bus_lock()` اللي مسك الـ mutex.

---

#### `spi_bus_lock` / `spi_bus_unlock`

```c
int spi_bus_lock(struct spi_controller *ctlr);
int spi_bus_unlock(struct spi_controller *ctlr);
```

**lock**: بيمسك `bus_lock_mutex` وبيضبط `bus_lock_flag = 1`. الـ `spi_async` بيرجع `-EBUSY` لأي caller تاني. الـ mutex بيفضل ممسوك لحد ما يجي `spi_bus_unlock`.

**unlock**: بيصفّر `bus_lock_flag` وبيحرر الـ mutex.

**Use case:** لما driver محتاج sequence من transfers متواصلة مش ينقطع بيها transfers من devices تانية — زي display drivers.

---

#### `spi_write_then_read`

```c
int spi_write_then_read(struct spi_device *spi,
                        const void *txbuf, unsigned n_tx,
                        void *rxbuf, unsigned n_rx);
```

بيعمل half-duplex MicroWire-style transaction: write N bytes ثم read M bytes. الـ data بيتنسخ لـ local buffer (DMA-safe) تلقائياً.

**Optimization:**
- لو `(n_tx + n_rx) <= SPI_BUFSIZ` (32 bytes default) وممكن يمسك `lock`: بيستخدم pre-allocated static `buf` لتجنب heap allocation
- غير كذلك: `kmalloc` بـ `GFP_DMA`

**Flow:**

```
Try trylock(lock) for static buf, else kmalloc
spi_message_init()
Add tx transfer (x[0]) if n_tx > 0
Add rx transfer (x[1]) if n_rx > 0
memcpy(local_buf, txbuf, n_tx)
x[0].tx_buf = local_buf
x[1].rx_buf = local_buf + n_tx
spi_sync()
if success: memcpy(rxbuf, x[1].rx_buf, n_rx)
```

**Key details:**
- **Portable code**: المتغير الـ `SPI_BUFSIZ` محدود بـ 32 bytes للـ portable code
- الـ static mutex موجود في الـ function — بيمنع concurrent calls على نفس الـ static buffer

---

### Group 5: Message Optimization Pipeline

الـ optimization system ده الجديد في الـ kernel اللي بيخلّي الـ driver يعمل expensive setup مرة واحدة بدل ما يعيده مع كل transfer.

---

#### `spi_optimize_message`

```c
int spi_optimize_message(struct spi_device *spi, struct spi_message *msg);
```

بيعمل one-time validation وsetup للـ message. يُستخدم من الـ peripheral driver لما بيعيد استخدام نفس الـ message (زي repetitive sensor reads).

**Internal flow (`__spi_optimize_message`):**

```
__spi_validate(spi, msg)     ← validate all transfers
spi_split_transfers(msg)     ← split if needed (CS_WORD, maxsize)
ctlr->optimize_message(msg) ← optional driver hook
msg->optimized = true
```

**Key details:**
- لما message يتعمله optimize، ممكن بس تغير الـ data في `tx_buf` — مش الـ pointer نفسه
- لو `ctlr->defer_optimize_message`: الـ optimization بتتأجل (زي spi-mux)
- بيضبط `msg->pre_optimized = true` عشان `spi_finalize_current_message` ما يعمل unoptimize تلقائياً

---

#### `spi_unoptimize_message`

```c
void spi_unoptimize_message(struct spi_message *msg);
```

بيحرر resources الـ optimization:
- بيستدعي `ctlr->unoptimize_message(msg)` لو موجود
- `spi_res_release()` لتحرير الـ `spi_res` resources
- `msg->optimized = false`، `msg->opt_state = NULL`

**Caller context:** can sleep.

---

#### `devm_spi_optimize_message`

```c
int devm_spi_optimize_message(struct device *dev, struct spi_device *spi,
                              struct spi_message *msg);
```

Managed version: بيسجل `spi_unoptimize_message` كـ devm action — بيتنادى تلقائياً لما الـ device يتunbind.

---

### Group 6: Internal Queue Management

الـ message pump هو الـ core الداخلي للـ async processing.

---

#### `spi_transfer_one_message` (default implementation)

```c
static int spi_transfer_one_message(struct spi_controller *ctlr,
                                    struct spi_message *msg);
```

الـ default implementation للـ `ctlr->transfer_one_message`. بتستخدمها الـ controllers اللي بتوفر `transfer_one` بدل `transfer_one_message`.

**Pseudocode:**

```
spi_set_cs(msg->spi, !first_xfer->cs_off, false)  ← assert CS
Increment messages stats

For each xfer in msg->transfers:
    trace_spi_transfer_start()
    update statistics
    if !ptp_sts_supported: ptp_read_system_prets()

    if xfer has data:
        reinit_completion(xfer_completion)
        spi_dma_sync_for_device()
        ret = ctlr->transfer_one(ctlr, msg->spi, xfer)
        if ret < 0:
            spi_dma_sync_for_cpu()
            if SPI_TRANS_FAIL_NO_START:
                fallback to PIO (goto fallback_pio)
            else: increment error stats, goto out
        if ret > 0:  ← async, need to wait
            spi_transfer_wait()
        spi_dma_sync_for_cpu()

    if !ptp_sts_supported: ptp_read_system_postts()
    trace_spi_transfer_stop()

    spi_transfer_delay_exec()
    Handle cs_change logic
    msg->actual_length += xfer->len

out:
spi_set_cs(msg->spi, false, false)  ← deassert CS
msg->status = ret
if msg->status: ctlr->handle_err()
spi_finalize_current_message()
```

**Key details:**
- الـ `fallback_pio` label: لو الـ DMA فشل بـ `SPI_TRANS_FAIL_NO_START`، بيعمل unmap ويعيد المحاولة بـ PIO
- `cs_change`: لو `true` على غير آخر transfer، بيعمل CS deassert/delay/reassert بين الـ transfers
- `cs_off`: بيسمح للـ transfer إنه يشتغل بدون CS assertion (rare use case)

---

#### `__spi_pump_messages`

```c
static void __spi_pump_messages(struct spi_controller *ctlr, bool in_kthread);
```

الـ message pump loop. بيشتغل من الـ kthread أو من `spi_sync` directly (fast path).

**Pseudocode:**

```
mutex_lock(io_mutex)
spin_lock_irqsave(queue_lock)

if cur_msg: goto out_unlock  ← already processing

if queue empty or !running:
    if !busy: goto out_unlock
    if !in_kthread:
        if simple idle: mark idle, queue_empty=true
        else: kthread_queue_work(pump_messages)
        goto out_unlock
    else (in kthread):
        kfree dummy_rx/tx
        unprepare_transfer_hardware()
        spi_idle_runtime_pm()
        queue_empty = true
        goto out_unlock

msg = list_first_entry(queue)
ctlr->cur_msg = msg
list_del(msg from queue)
was_busy = ctlr->busy; ctlr->busy = true
spin_unlock_irqrestore

__spi_pump_transfer_message(ctlr, msg, was_busy)
kthread_queue_work(pump_messages)  ← schedule next
ctlr->cur_msg = NULL
mutex_unlock(io_mutex)
cond_resched()
```

**Key details:**
- الـ `mutex_lock(io_mutex)` و`spin_lock_irqsave(queue_lock)` بيحققوا serialization
- الـ `in_kthread=false` path (من `spi_sync`): يتجنب `kfree` وخلافه عشان ممكن يكون في atomic context

---

#### `__spi_pump_transfer_message`

```c
static int __spi_pump_transfer_message(struct spi_controller *ctlr,
                                       struct spi_message *msg, bool was_busy);
```

بيُنفذ message واحد:

```
if !was_busy && auto_runtime_pm: pm_runtime_get_sync()
if !was_busy: ctlr->prepare_transfer_hardware()
ctlr->prepare_message()
spi_map_msg()  ← DMA mapping

WRITE_ONCE(cur_msg_incomplete, true)
WRITE_ONCE(cur_msg_need_completion, false)
reinit_completion(cur_msg_completion)
smp_wmb()

ctlr->transfer_one_message(ctlr, msg)

WRITE_ONCE(cur_msg_need_completion, true)
smp_mb()
if READ_ONCE(cur_msg_incomplete):
    wait_for_completion(cur_msg_completion)
```

**Key details:**
- الـ `cur_msg_incomplete` / `cur_msg_need_completion` flags هي optimization لتجنب الـ completion overhead في الـ common case (synchronous drivers)
- الـ `smp_wmb()` / `smp_mb()` ضروريين لـ memory ordering correctness

---

#### `spi_finalize_current_message`

```c
void spi_finalize_current_message(struct spi_controller *ctlr);
```

بيُنادى من الـ driver (أو من `spi_transfer_one_message`) لما الـ message خلص.

**Flow:**

```
mesg = ctlr->cur_msg
if !ptp_sts_supported && !transfer_one: read postts for all xfers
spi_unmap_msg()
if mesg->prepared: ctlr->unprepare_message()
mesg->prepared = false
spi_maybe_unoptimize_message()

WRITE_ONCE(cur_msg_incomplete, false)
smp_mb()
if READ_ONCE(cur_msg_need_completion):
    complete(cur_msg_completion)

trace_spi_message_done()
mesg->state = NULL
if mesg->complete: mesg->complete(mesg->context)
```

**Key details:**
- الـ `complete` callback بيتنادى من هنا — ده اللي بيصحّي الـ `spi_sync` أو الـ async caller
- لو `spi_maybe_unoptimize_message` هتعمل unoptimize: بس لو الـ message مش `pre_optimized` (أي الـ peripheral مش بيعيد استخدامه)

---

#### `spi_finalize_current_transfer`

```c
void spi_finalize_current_transfer(struct spi_controller *ctlr);
```

بيُنادى من interrupt-driven controllers بعد كل transfer واحد (مش الـ message كله). بيعمل `complete(&ctlr->xfer_completion)` اللي `spi_transfer_wait` بيستناه.

---

#### `spi_get_next_queued_message`

```c
struct spi_message *spi_get_next_queued_message(struct spi_controller *ctlr);
```

بيرجع أول message في الـ queue بدون ما يشيله. بيمسك `queue_lock` spinlock.

**Use case:** controllers بتعمل custom `transfer_one_message` وعايزة تبص على الـ next message قبل ما تخلص الـ current.

---

#### `spi_flush_queue`

```c
void spi_flush_queue(struct spi_controller *ctlr);
```

بيشغل `__spi_pump_messages(ctlr, false)` من الـ calling context — بيضمن إن كل الـ pending async messages خلصت قبل ما يكمل. بيُستخدم من الـ `spi-mem` code.

---

### Group 7: Transfer Splitting

---

#### `spi_split_transfers_maxsize`

```c
int spi_split_transfers_maxsize(struct spi_controller *ctlr,
                                struct spi_message *msg,
                                size_t maxsize);
```

بيكسر كل transfer في الـ message أكبر من `maxsize` لـ transfers أصغر. بيُستخدم من `optimize_message` callbacks.

**Internal via `__spi_split_transfer_maxsize`:**

```
count = DIV_ROUND_UP(xfer->len, maxsize)
spi_replace_transfers(msg, xfer, 1, count, ...)
    ← allocates spi_res, removes original, inserts 'count' copies
Fix up len, rx_buf, tx_buf offsets for each chunk
*xferp = &xfers[count-1]  ← skip newly inserted entries
Increment transfers_split_maxsize stats
```

**Key details:**
- الـ resources المخصصة هنا بتتحرر تلقائياً في `spi_res_release` عند الـ unoptimize
- الـ `spi_replace_transfers` mechanism بيستخدم `spi_res` lifecycle لضمان cleanup

---

#### `spi_split_transfers_maxwords`

```c
int spi_split_transfers_maxwords(struct spi_controller *ctlr,
                                 struct spi_message *msg,
                                 size_t maxwords);
```

نفس `spi_split_transfers_maxsize` لكن بيحسب `maxsize = maxwords * spi_bpw_to_bytes(xfer->bits_per_word)` لكل transfer — لأن الـ word size ممكن يختلف بين الـ transfers.

---

### Group 8: Timing & CS Management

---

#### `spi_set_cs` (internal)

```c
static void spi_set_cs(struct spi_device *spi, bool enable, bool force);
```

الـ central function للـ chip select control. بيعمل:

```
if !force && same CS state as last time: return  ← optimization
trace_spi_set_cs()
Update last_cs[], last_cs_index_mask, last_cs_mode_high

Handle SPI_CS_HIGH: invert 'enable' if needed

if !set_cs_timing (or GPIO): spi_delay_exec(cs_hold) on deactivate

if GPIO CS:
    if !SPI_NO_CS: spi_toggle_csgpiod() for each valid CS
    if SPI_CONTROLLER_GPIO_SS && set_cs: ctlr->set_cs(spi, !enable)
else if ctlr->set_cs:
    ctlr->set_cs(spi, !enable)

if GPIO or !set_cs_timing:
    on activate: spi_delay_exec(cs_setup)
    on deactivate: spi_delay_exec(cs_inactive)
```

**Key details:**
- الـ **polarity inversion**: الـ `enable=true` معناه "assert CS" — لو `SPI_CS_HIGH` فالـ HW يعمل HIGH، غير كذلك LOW
- الـ ACPI quirk: ACPI GPIOs دايماً Active High، فالـ logic بتتعكس manually

---

#### `spi_delay_to_ns`

```c
int spi_delay_to_ns(struct spi_delay *_delay, struct spi_transfer *xfer);
```

بيحول `spi_delay` لـ nanoseconds. بيعرف 3 units:
- `SPI_DELAY_UNIT_USECS`: بيضرب في `NSEC_PER_USEC`
- `SPI_DELAY_UNIT_NSECS`: بيرجعها as-is
- `SPI_DELAY_UNIT_SCK`: بيحسب `cycles * (NSEC_PER_SEC / speed_hz)` — يحتاج `xfer` صالح

**Return:** الـ delay بالـ nanoseconds، أو `<0` لو invalid.

---

#### `spi_delay_exec`

```c
int spi_delay_exec(struct spi_delay *_delay, struct spi_transfer *xfer);
```

بيحسب الـ delay بـ `spi_delay_to_ns` ثم بيعمله `_spi_transfer_delay_ns()`:
- `<= 1us`: `ndelay(ns)` — busy-wait دقيق
- `> 1us`: `fsleep(us)` — ممكن ينام (adaptive)

بيعمل `might_sleep()` لـ debug checking.

---

#### `spi_transfer_wait`

```c
static int spi_transfer_wait(struct spi_controller *ctlr,
                             struct spi_message *msg,
                             struct spi_transfer *xfer);
```

بينتظر إتمام transfer بدء من الـ `transfer_one()` callback:
- **Target mode**: `wait_for_completion_interruptible` بدون timeout
- **Host mode**: بيحسب timeout = `(8 * MSEC_PER_SEC * len / speed_hz) * 2 + 200ms`

لو timeout: يزيد statistics وبيرجع `-ETIMEDOUT`.

---

### Group 9: SPI Resource Management (spi_res)

الـ `spi_res` هو lifecycle-managed resource ملصق بـ `spi_message`. بيتحرر تلقائياً في `spi_res_release`.

---

#### `spi_res_alloc` / `spi_res_free` / `spi_res_add` / `spi_res_release`

```c
static void *spi_res_alloc(struct spi_device *spi, spi_res_release_t release,
                           size_t size, gfp_t gfp);
static void spi_res_free(void *res);
static void spi_res_add(struct spi_message *message, void *res);
static void spi_res_release(struct spi_controller *ctlr, struct spi_message *message);
```

- **alloc**: بيخصص `spi_res` + `size` bytes، بيضبط الـ `release` callback
- **free**: بيحرر `spi_res` (قبل ما يتضاف للـ message)
- **add**: بيضيف الـ resource لـ `message->resources` list
- **release**: بيمشي على الـ list بالـ reverse order، بينادي كل `res->release` callback، ثم `list_del` + `kfree`

**Use case:** `spi_replace_transfers` بيخصص `spi_replaced_transfers` كـ `spi_res` — لما الـ message يتفرغ، `__spi_replace_transfers_release` بيعيد الـ original transfers.

---

### Group 10: DMA Mapping

---

#### `spi_map_buf` / `spi_unmap_buf`

```c
int spi_map_buf(struct spi_controller *ctlr, struct device *dev,
                struct sg_table *sgt, void *buf, size_t len,
                enum dma_data_direction dir);
void spi_unmap_buf(struct spi_controller *ctlr, struct device *dev,
                   struct sg_table *sgt, enum dma_data_direction dir);
```

بيعمل/بيلغي DMA mapping لـ buffer. الـ internal `spi_map_buf_attrs` بيتعامل مع 3 حالات:
- **vmalloc buffer**: بيكسره لـ pages بـ `vmalloc_to_page()`
- **highmem kmap buffer**: نفس approach بـ `kmap_to_page()`
- **regular virtual address**: بيستخدم `sg_set_buf` مباشرةً

---

#### `__spi_map_msg` / `__spi_unmap_msg`

بيعمل DMA map/unmap لكل الـ transfers في الـ message اللي `ctlr->can_dma()` بترجعلها `true`. بيستخدم `DMA_ATTR_SKIP_CPU_SYNC` لأن الـ sync بيتعمل manually قبل/بعد كل transfer بـ `spi_dma_sync_for_device/cpu`.

---

#### `spi_map_msg`

```c
static int spi_map_msg(struct spi_controller *ctlr, struct spi_message *msg);
```

بيعمل:
1. لو `SPI_CONTROLLER_MUST_TX/RX`: بيخصص `dummy_tx/rx` buffers للـ transfers اللي ماعندهاش tx_buf أو rx_buf
2. ثم `__spi_map_msg()` للـ DMA mapping الفعلي

---

### Group 11: PTP Timestamping

---

#### `spi_take_timestamp_pre`

```c
void spi_take_timestamp_pre(struct spi_controller *ctlr,
                             struct spi_transfer *xfer,
                             size_t progress, bool irqs_off);
```

بيُنادى من الـ driver قبل ما يرسل الـ word المطلوب للـ timestamp. بيقرأ `ptp_read_system_prets()` مرة واحدة فقط (لما `progress <= ptp_sts_word_pre`).

لو `irqs_off=true`: بيعمل `local_irq_save` + `preempt_disable` لتقليل الـ jitter — مناسب للـ PIO drivers فقط.

---

#### `spi_take_timestamp_post`

```c
void spi_take_timestamp_post(struct spi_controller *ctlr,
                              struct spi_transfer *xfer,
                              size_t progress, bool irqs_off);
```

نظير `pre`: بيقرأ `ptp_read_system_postts()` وبيحرر الـ IRQs/preemption لو `irqs_off=true`. بيضبط `xfer->timestamped = 1` لمنع الـ timestamping مرتين.

---

### Group 12: ACPI & OF Device Discovery

---

#### `of_register_spi_devices`

```c
static void of_register_spi_devices(struct spi_controller *ctlr);
```

بيمشي على كل الـ available child nodes للـ controller في الـ Device Tree وبيعمل `of_register_spi_device()` لكل واحد. بيستخدم `of_node_test_and_set_flag(OF_POPULATED)` لمنع التسجيل المزدوج.

---

#### `of_spi_parse_dt`

```c
static int of_spi_parse_dt(struct spi_controller *ctlr, struct spi_device *spi,
                           struct device_node *nc);
```

بيقرأ من الـ DT node:
- `spi-cpha`, `spi-cpol`, `spi-3wire`, `spi-lsb-first`, `spi-cs-high` → mode bits
- `spi-tx-bus-width` / `spi-rx-bus-width` (1,2,4,8) → SINGLE/DUAL/QUAD/OCTAL mode bits
- `spi-tx-lane-map` / `spi-rx-lane-map` → physical lane mapping
- `reg` → chip select number(s)
- `spi-max-frequency` → max_speed_hz
- `spi-cs-setup-delay-ns`, `spi-cs-hold-delay-ns`, `spi-cs-inactive-delay-ns` → CS delays

---

#### `acpi_spi_device_alloc`

```c
struct spi_device *acpi_spi_device_alloc(struct spi_controller *ctlr,
                                         struct acpi_device *adev,
                                         int index);
```

بيخصص `spi_device` من ACPI `SpiSerialBus` resource. لو `ctlr=NULL`: بيلاقيه من الـ resource. لو `index=-1`: بياخد أول resource.

بيتعامل مع Apple machines special case عبر `acpi_spi_parse_apple_properties`.

---

#### `acpi_spi_count_resources`

```c
int acpi_spi_count_resources(struct acpi_device *adev);
```

بيعد عدد الـ `SpiSerialBus` resources في الـ `_CRS` للـ ACPI device. مفيد للـ drivers اللي بتبص على عدد الـ SPI interfaces قبل الـ allocation.

---

### Group 13: Validation & Internal Helpers

---

#### `__spi_validate`

```c
static int __spi_validate(struct spi_device *spi, struct spi_message *message);
```

بيـvalidate الـ message قبل التنفيذ:
- الـ `transfers` list مش فاضية
- Half-duplex / 3-wire: كل transfer ماعندهاش TX وRX معاً
- لكل transfer: validates `bits_per_word`، `speed_hz`، buffer alignment (`len % w_size == 0`)
- validates `tx_nbits` / `rx_nbits` مع الـ mode bits
- validates `dtr_mode` مع `ctlr->dtr_caps`
- validates `offload_flags` مع `msg->offload->xfer_flags`
- يضبط defaults: `bits_per_word=8`، `tx_nbits=SPI_NBITS_SINGLE` لو مش محدد

**Return:** 0 أو `-EINVAL`.

---

#### `spi_init_queue` / `spi_start_queue` / `spi_stop_queue` / `spi_destroy_queue`

الدورة الكاملة للـ message pump queue:

| Function | الهدف |
|---|---|
| `spi_init_queue` | ينشئ الـ kthread worker، يهيئ `pump_messages` work item |
| `spi_start_queue` | `ctlr->running = true`، يعمل queue initial kick |
| `spi_stop_queue` | ينتظر لحد ما الـ queue تفضى (max 500 iterations × 10ms = 5s) |
| `spi_destroy_queue` | `spi_stop_queue` ثم `kthread_destroy_worker` |

**Key detail في `spi_stop_queue`:** بيعمل polling بـ `usleep_range(10000, 11000)` — مش أجمل حل لكنه بيتجنب overhead الـ `wait_queue` في الـ common path.

---

#### `spi_set_thread_rt`

```c
static void spi_set_thread_rt(struct spi_controller *ctlr);
```

بيضبط الـ message pump kthread كـ `SCHED_FIFO` realtime priority بـ `sched_set_fifo()`. بيُنادى لو `ctlr->rt=true` أو لو أي `spi_device->rt=true` (من `spi_setup`). لو أي device على الـ bus محتاج RT، كل الـ controller بيشتغل بـ RT.

---

#### `spi_match_device`

```c
static int spi_match_device(struct device *dev, const struct device_driver *drv);
```

الـ bus `match` callback. بيتبع الأولوية:
1. `driver_override` — مباشرة بـ name matching
2. `of_driver_match_device` — DT compatible matching
3. `acpi_driver_match_device` — ACPI HID matching
4. `spi_match_id` على الـ `id_table`
5. `strcmp(modalias, drv->name)` — fallback

---

#### `spi_statistics_add_transfer_stats` (internal)

```c
static void spi_statistics_add_transfer_stats(struct spi_statistics __percpu *pcpu_stats,
                                              struct spi_transfer *xfer,
                                              struct spi_message *msg);
```

بيزيد per-CPU statistics لكل transfer. بيستخدم `fls(xfer->len)` لحساب histogram bucket (log2 approximation). الـ `u64_stats_update_begin/end` بيحمي من الـ inconsistency على 32-bit systems.

---

### ملاحظات ختامية مهمة

1. **Locking hierarchy** في الـ SPI core:
   ```
   board_lock (mutex)          ← static global, يحمي board_list و idr
     └── add_lock (mutex)      ← per-controller, يحمي device addition
           └── io_mutex        ← per-controller, يحمي actual I/O
                 └── queue_lock (spinlock) ← per-controller, يحمي queue
                       └── bus_lock_spinlock ← يحمي bus_lock_flag
   ```

2. **Memory ordering**: الـ `WRITE_ONCE`/`READ_ONCE` + `smp_wmb()`/`smp_mb()` في `__spi_pump_transfer_message` وfinalizer هي optimization صعبة — بتسمح بتجنب الـ completion في الـ common synchronous case.

3. **`spi_res` pattern**: كل الـ message transformations (split, replace transfers) بتستخدم الـ `spi_res` lifecycle — guaranteed cleanup عند الـ unoptimize بغض النظر عن الـ error paths.

4. **DMA fallback**: لو `transfer_one` فشل بـ `SPI_TRANS_FAIL_NO_START` وكان الـ buffer mapped، الـ core بيعمل unmap وبيعيد المحاولة بـ PIO عبر الـ `fallback_pio` label — transparent للـ protocol driver.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. مدخلات الـ debugfs

الـ SPI subsystem مش بيعرض entries مباشرة في debugfs بشكل standard، لكن لو عندك controller معين (زي `spi-rockchip` أو `spi-dw`) ممكن يعمل entries خاصة بيه. الـ core نفسه بيعتمد على **sysfs** أساساً للإحصائيات.

```bash
# شوف لو في أي SPI entry في debugfs
mount -t debugfs none /sys/kernel/debug 2>/dev/null || true
ls /sys/kernel/debug/ | grep -i spi

# بعض controllers بتعمل entries زي:
ls /sys/kernel/debug/regmap/        # لو controller بيستخدم regmap
```

---

#### 2. مدخلات الـ sysfs

الـ SPI core بيعمل **per-CPU statistics** لكل controller وكل device، وبتلاقيها في:

```
/sys/bus/spi/devices/spi<N>.<M>/statistics/
/sys/class/spi_master/spi<N>/statistics/
```

| الـ Entry | المعنى |
|-----------|--------|
| `messages` | عدد الـ `spi_message` اللي اتعالجت |
| `transfers` | عدد الـ `spi_transfer` اللي نفذت |
| `errors` | عدد الأخطاء |
| `timedout` | عدد مرات الـ timeout |
| `spi_sync` | استدعاءات `spi_sync()` |
| `spi_sync_immediate` | مرات نُفّذت فيها `spi_sync` مباشرة بدون قائمة |
| `spi_async` | استدعاءات `spi_async()` |
| `bytes` | إجمالي البايتات |
| `bytes_tx` | بايتات أُرسلت |
| `bytes_rx` | بايتات استُقبلت |
| `transfer_bytes_histo_0-1` ... `transfer_bytes_histo_65536+` | histogram للأحجام |
| `transfers_split_maxsize` | عدد الـ transfers اللي اتقسمت لحد `max_transfer_size` |

```bash
# قراءة إحصائيات controller رقم 0
cat /sys/class/spi_master/spi0/statistics/messages
cat /sys/class/spi_master/spi0/statistics/errors
cat /sys/class/spi_master/spi0/statistics/timedout

# قراءة إحصائيات device (مثلاً spi0.0)
cat /sys/bus/spi/devices/spi0.0/statistics/bytes_tx
cat /sys/bus/spi/devices/spi0.0/statistics/bytes_rx
cat /sys/bus/spi/devices/spi0.0/statistics/errors

# الـ modalias للتأكد من الـ driver المربوط
cat /sys/bus/spi/devices/spi0.0/modalias

# الـ driver_override لو محتاج تجبر device على driver معين
cat /sys/bus/spi/devices/spi0.0/driver_override
```

---

#### 3. الـ ftrace — Tracepoints والـ Events

الـ SPI core بيعرّف الـ tracepoints دي (معرّفة في `include/trace/events/spi.h`):

| الـ Tracepoint | بيتفعّل إمتى |
|----------------|-------------|
| `spi_controller_idle` | لما الـ controller يبقى idle (قائمة فاضية) |
| `spi_controller_busy` | لما الـ controller يبدأ يشتغل |
| `spi_setup` | عند استدعاء `spi_setup()` |
| `spi_set_cs` | عند تفعيل/إلغاء الـ chip select |
| `spi_message_submit` | لما message تتحط في الـ queue |
| `spi_message_start` | لما الـ controller يبدأ ينفّذ الـ message |
| `spi_message_done` | لما الـ message تخلص |
| `spi_transfer_start` | بداية كل `spi_transfer` |
| `spi_transfer_stop` | نهاية كل `spi_transfer` |

```bash
# تفعيل كل SPI events
cd /sys/kernel/debug/tracing
echo 1 > events/spi/enable

# أو حدد events معينة
echo 1 > events/spi/spi_transfer_start/enable
echo 1 > events/spi/spi_transfer_stop/enable
echo 1 > events/spi/spi_message_done/enable
echo 1 > events/spi/spi_set_cs/enable

# ابدأ التتبع
echo 1 > tracing_on

# نفّذ الكود اللي عايز تتبعه، تم اقرأ النتائج
cat trace

# مثال output:
# kworker/0:2-123 [000] .... spi_message_submit: spi0.0 0xffff...
# kworker/0:2-123 [000] .... spi_transfer_start: spi0.0 0xffff... len=4 tx=[de ad be ef]
# kworker/0:2-123 [000] .... spi_transfer_stop:  spi0.0 0xffff... len=4 rx=[01 02 03 04]
# kworker/0:2-123 [000] .... spi_message_done:   spi0.0 0xffff... len=4/4

# تعطيل بعد الانتهاء
echo 0 > events/spi/enable
echo 0 > tracing_on
```

**تفسير الـ output:**
- الـ `len=4/4` في `spi_message_done` معناها `actual_length/frame_length` — لو مختلفين في مشكلة.
- الـ `tx=[de ad be ef]` بيطبع أول 64 بايت بس.

---

#### 4. الـ printk والـ dynamic debug

الـ SPI core بيستخدم `dev_dbg()` في أماكن استراتيجية. تفعّلها بـ **dynamic debug**:

```bash
# تفعيل كل debug messages للـ SPI core
echo "file drivers/spi/spi.c +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل لـ SPI controller معين
echo "file drivers/spi/spi-rockchip.c +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل بالـ module
echo "module spi_core +p" > /sys/kernel/debug/dynamic_debug/control

# شوف كل الـ debug points المتفعّلة
grep "=p" /sys/kernel/debug/dynamic_debug/control | grep spi
```

**الـ dev_dbg messages المهمة في spi.c:**

```c
dev_dbg(dev, "registered child %s\n", dev_name(&spi->dev));
// تأكيد تسجيل device

dev_dbg(&spi->dev, "setup mode %lu, ... --> %d\n", ...);
// نتيجة spi_setup()

dev_dbg(&msg->spi->dev, "SPI transfer interrupted\n");
// transfer اتقاطع (target mode)
```

---

#### 5. كونفيج الـ Kernel للـ Debugging

| الـ Config | الغرض |
|-----------|--------|
| `CONFIG_SPI_DEBUG` | يفعّل كل `dev_dbg()` في الـ SPI subsystem |
| `CONFIG_DEBUG_SPI_LOOPBACK` | loopback test للـ controller |
| `CONFIG_SPI_SPIDEV` | userspace access لاختبار الـ transfers مباشرة |
| `CONFIG_SPI_SLAVE` | تفعيل target/slave mode |
| `CONFIG_SPI_DYNAMIC` | إضافة/حذف devices ديناميكياً |
| `CONFIG_LOCKDEP` | اكتشاف مشاكل الـ locking في الـ bus_lock_mutex وio_mutex |
| `CONFIG_PROVE_LOCKING` | يثبت صحة ترتيب الـ locks |
| `CONFIG_DEBUG_OBJECTS` | اكتشاف استخدام objects بعد تحريرها |
| `CONFIG_KASAN` | اكتشاف memory corruption في الـ transfer buffers |
| `CONFIG_KCSAN` | اكتشاف data races في الـ per-CPU statistics |

```bash
# تحقق من الكونفيج الحالي
zcat /proc/config.gz | grep -E "CONFIG_SPI|CONFIG_DEBUG_SPI"
```

---

#### 6. أدوات خاصة بالـ SPI Subsystem

**spidev — اختبار userspace مباشر:**

```bash
# التأكد من وجود spidev
ls /dev/spidev*

# اختبار loopback (قصّر MOSI على MISO)
# يحتاج spi-tools
apt install spi-tools   # Debian/Ubuntu
spi-config -d /dev/spidev0.0 -q   # اقرأ الكونفيج الحالي

# إرسال واستقبال بايتات (hex)
spi-pipe -d /dev/spidev0.0 -s 1000000 <<< $'\xde\xad\xbe\xef'
```

**spidev_test — أداة الـ kernel نفسها:**

```bash
# موجودة في tools/spi/ بسورس الـ kernel
cd /path/to/linux-source/tools/spi
make
./spidev_test -D /dev/spidev0.0 -s 500000 -v
# -s: speed in Hz
# -v: verbose (يطبع tx و rx)
```

**Output مثال:**

```
spi mode: 0x0
bits per word: 8
max speed: 500000 Hz (500 KHz)
TX | FF FF FF FF FF FF 40 00 00 00 00 95 FF FF FF FF ...
RX | FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF ...
```

**فحص الـ SPI bus من sysfs:**

```bash
# كل الـ SPI controllers المسجّلة
ls /sys/class/spi_master/

# كل الـ SPI devices
ls /sys/bus/spi/devices/

# الـ driver المربوط بكل device
for dev in /sys/bus/spi/devices/spi*.*; do
    echo -n "$dev -> driver: "
    readlink $dev/driver 2>/dev/null | xargs basename || echo "none"
done
```

---

#### 7. جدول رسائل الأخطاء الشائعة

| رسالة الـ Kernel | المعنى | الحل |
|-----------------|--------|------|
| `SPI transfer timed out` | الـ `wait_for_completion_timeout` خلص وقته | تحقق من IRQ للـ controller، والـ clock، والـ hardware |
| `SPI transfer failed: -EIO` | الـ `transfer_one()` رجع `-EIO` | مشكلة في الـ hardware أو الـ DMA |
| `chipselect N already in use` | device تاني بيستخدم نفس الـ CS | تحقق من الـ Device Tree أو board config |
| `can't setup %s, status -EINVAL` | `spi_setup()` فشلت | الـ mode bits مش supported أو speed خارج النطاق |
| `setup: unsupported mode bits %x` | bits في `spi->mode` مش في `ctlr->mode_bits` | حذف الـ bits غير المدعومة من الـ DT |
| `setup: can not select any two of dual, quad and no-rx/tx` | تضارب في وضع الـ bus | خليّ TX أو RX واحد بس (DUAL أو QUAD أو NO_TX) |
| `failed to prepare transfer hardware` | `prepare_transfer_hardware()` فشل | مشكلة في power أو clock للـ controller |
| `failed to transfer one message from queue` | `transfer_one_message()` رجع خطأ | مشكلة في الـ driver نفسه |
| `problem destroying queue` | `spi_stop_queue()` فاضت 500 محاولة | الـ kthread عالق أو transfer ما بتخلصش |
| `failed to add SPI device %s from ACPI` | `spi_add_device()` فشل عند enumerate الـ ACPI | تحقق من الـ ACPI tables والـ CS |
| `SPI driver %s has no spi_device_id for %s` | تحذير: driver بدون matching ID للـ DT compatible | أضف `spi_device_id` للـ driver |
| `cs%d >= max %d` | رقم الـ CS أكبر من `num_chipselect` | تصليح الـ DT أو زيادة `num_chipselect` |
| `Bufferless transfer has length %u` | `xfer->len != 0` لكن `tx_buf == rx_buf == NULL` | مشكلة في إعداد الـ transfer |
| `Attempted to sync while suspend` | استدعاء `spi_sync` في وقت الـ suspend | لازم تحمي الـ driver بـ runtime PM |

---

#### 8. نقاط استراتيجية لـ dump_stack() وWARN_ON()

من الكود نفسه، الـ WARN_ON موجودة في:

```c
/* spi_res_free: تحذير لو resource في قائمة لما بنحذفها */
WARN_ON(!list_empty(&sres->entry));

/* spi_res_add: تحذير لو resource اتضافت مرتين */
WARN_ON(!list_empty(&sres->entry));

/* spi_new_ancillary_device: لازم add_lock يكون مأخوذ */
WARN_ON(!mutex_is_locked(&ctlr->add_lock));

/* spi_finalize_current_message: PTP timestamp لازم يتعمل */
WARN_ON_ONCE(xfer->ptp_sts && !xfer->timestamped);
```

**نقاط تستفيد منها لو عملت debugging patches:**

```c
/* في __spi_pump_messages - تحقق من state الـ controller */
WARN_ON(ctlr->cur_msg && msg != ctlr->cur_msg);

/* في spi_transfer_one_message - تحقق من الـ CS قبل الـ transfer */
WARN_ON(!msg->spi);

/* في spi_finalize_current_message - تحقق من الـ completion */
WARN_ON(READ_ONCE(ctlr->cur_msg) != mesg);
```

---

### Hardware Level

---

#### 1. التحقق من أن حالة الـ Hardware متطابقة مع حالة الـ Kernel

```bash
# تحقق من أن الـ controller مسجّل بنفس الـ bus number في DT
cat /sys/class/spi_master/spi0/of_node   # symlink للـ DT node

# تحقق من الـ chip select الفعلي
cat /sys/bus/spi/devices/spi0.0/modalias  # مطابق لـ DT compatible
cat /sys/bus/spi/devices/spi0.0/uevent   # فيه MODALIAS=spi:...

# تحقق من أن الـ driver probe نجح
cat /sys/bus/spi/devices/spi0.0/driver   # symlink لاسم الـ driver

# فحص الـ GPIO chip selects
ls /sys/class/gpio/ | grep -i spi        # لو مستخدم GPIO-based CS
# أو
cat /sys/kernel/debug/gpio | grep -i "spi.*cs"
```

**ميزان الـ speed:**

```bash
# speed المطلوبة في الـ kernel
cat /sys/bus/spi/devices/spi0.0/of_properties/spi-max-frequency 2>/dev/null
# أو من DT overlay
fdtget /boot/dtb.dtb /spi@... spi-max-frequency

# الـ speed الفعلية (بعد negotiation مع الـ controller)
# بتلاقيها في dmesg بعد spi_setup مع CONFIG_SPI_DEBUG
dmesg | grep "spi.*setup mode"
```

---

#### 2. تقنيات الـ Register Dump

**devmem2 لقراءة registers الـ SPI controller مباشرة:**

```bash
# مثال: Raspberry Pi SPI0 base = 0x20204000
# (غيّر العنوان حسب SoC الخاص بك)
apt install devmem2

# قراءة CS register
devmem2 0x20204000 w

# قراءة CLK register
devmem2 0x20204008 w

# قراءة DLEN register
devmem2 0x2020400C w

# قراءة FIFO register (DATA)
devmem2 0x20204004 w

# Rockchip SPI0 مثال آخر
# قراءة status register
devmem2 0xFF1C0010 w   # SR register
```

**عبر /dev/mem:**

```c
/* مثال C program لقراءة block كامل */
#include <sys/mman.h>
#include <fcntl.h>
#define SPI_BASE 0xFF1C0000
#define SPI_SIZE 0x1000

int fd = open("/dev/mem", O_RDONLY);
void *base = mmap(NULL, SPI_SIZE, PROT_READ, MAP_SHARED, fd, SPI_BASE);
/* ثم اقرأ registers بـ offset */
uint32_t status = *(volatile uint32_t *)((char*)base + 0x10);
```

---

#### 3. Logic Analyzer وOscilloscope — نصائح عملية

**نقاط القياس الأساسية:**

```
     CS   ──┐                              ┌──
            └──────────────────────────────┘
    CLK   ──┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐
            └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘
   MOSI   ──D7──D6──D5──D4──D3──D2──D1──D0──
   MISO   ──D7──D6──D5──D4──D3──D2──D1──D0──
```

**نقاط يجب قياسها:**

| إشارة | اللي تدور عليه |
|-------|---------------|
| CS timing | الـ `cs_setup` و`cs_hold` و`cs_inactive` delays محتاج تطابق الـ datasheet |
| CLK polarity | CPOL=0: idle low / CPOL=1: idle high |
| CLK phase | CPHA=0: sample on leading edge / CPHA=1: sample on trailing edge |
| Clock speed | تأكد مش بتتجاوز الـ `max_speed_hz` للـ device |

**إعداد Logic Analyzer (Pulseview/Sigrok):**

```bash
# تثبيت sigrok
apt install sigrok sigrok-cli pulseview

# التقاط SPI بـ sigrok-cli
sigrok-cli -d fx2lafw \
  --config samplerate=24M \
  -C D0=CS,D1=CLK,D2=MOSI,D3=MISO \
  --time 100ms \
  -P spi:cs=D0:clk=D1:mosi=D2:miso=D3:cpol=0:cpha=0 \
  -A spi=mosi-data,spi=miso-data
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في الـ Kernel Log

| المشكلة | نمط الـ Log | الحل |
|---------|------------|------|
| CS لا يُفعَّل | لا يبدأ أي transfer رغم وجود messages | تحقق GPIO polarity في DT (`cs-gpios`) |
| CLK frequency خاطئة | data يوصل corrupt أو partial | تقليل `spi-max-frequency` في DT |
| CPOL/CPHA غلط | data كلها `0xFF` أو `0x00` | تصليح `spi-cpol`/`spi-cpha` في DT |
| DMA mapping فشل | `SPI transfer failed: -EIO` مع `DMA` errors | تحقق من alignment الـ buffers (يجب 4-byte aligned للـ DMA) |
| Timeout | `SPI transfer timed out` | IRQ مش واصل للـ kernel، أو controller مش يكمل الـ transfer |
| FIFO overflow | errors متكررة بسرعات عالية | قلّل الـ speed أو شغّل DMA بدل PIO |

---

#### 5. Device Tree Debugging

```bash
# قراءة الـ DT المطبّق فعلياً في الـ runtime
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "spi@"

# أو مباشرة من sysfs
cat /proc/device-tree/spi@1c68000/compatible
cat /proc/device-tree/spi@1c68000/clock-frequency

# التحقق من أن الـ node موجود وصحيح
ls /proc/device-tree/spi*/

# التحقق من الـ CS property
fdtget /boot/dtb /spi@1c68000 reg
fdtget /boot/dtb /spi@1c68000/device@0 spi-max-frequency
fdtget /boot/dtb /spi@1c68000/device@0 compatible

# التحقق من الـ clocks
cat /proc/device-tree/spi@1c68000/clocks
cat /sys/kernel/debug/clk/spi0/clk_rate    # الـ clock الفعلي
```

**DT properties المهمة للـ SPI device (من `of_spi_parse_dt()`):**

```dts
/* مثال DT node مكتمل للـ debugging */
&spi0 {
    status = "okay";
    my-device@0 {
        compatible = "vendor,chip";
        reg = <0>;                           /* chip select رقم 0 */
        spi-max-frequency = <1000000>;       /* 1 MHz */
        spi-cpol;                            /* CPOL=1 */
        spi-cpha;                            /* CPHA=1 */
        /* اختياري للـ timing fine-tuning */
        spi-cs-setup-delay-ns = <200>;
        spi-cs-hold-delay-ns = <200>;
        spi-cs-inactive-delay-ns = <500>;
    };
};
```

```bash
# لو DT غلط، الـ dmesg يظهر:
dmesg | grep -E "spi.*error|spi.*failed|spi.*invalid|OF: ERROR"

# أمثلة أخطاء DT شائعة:
# "spi0: %pOF has no valid 'reg' property"  → ناقص reg
# "cs0 >= max 1"                            → CS رقم أكبر من num_chipselect
# "cannot find modalias for %pOF"           → compatible مش مطابق لأي driver
```

---

### Practical Commands

---

#### أوامر جاهزة للنسخ

**فحص شامل سريع:**

```bash
#!/bin/bash
echo "=== SPI Controllers ==="
ls /sys/class/spi_master/

echo -e "\n=== SPI Devices ==="
for dev in /sys/bus/spi/devices/spi*.*; do
    modalias=$(cat $dev/modalias 2>/dev/null)
    driver=$(readlink $dev/driver 2>/dev/null | xargs basename 2>/dev/null || echo "unbound")
    echo "  $dev -> $modalias [$driver]"
done

echo -e "\n=== Statistics for spi0 ==="
for stat in messages transfers errors timedout bytes_tx bytes_rx; do
    val=$(cat /sys/class/spi_master/spi0/statistics/$stat 2>/dev/null)
    echo "  $stat: $val"
done

echo -e "\n=== Recent SPI dmesg ==="
dmesg | grep -i spi | tail -20
```

**تشغيل ftrace للـ SPI:**

```bash
#!/bin/bash
TRACING=/sys/kernel/debug/tracing

echo 0 > $TRACING/tracing_on
echo > $TRACING/trace
echo 1 > $TRACING/events/spi/enable
echo 1 > $TRACING/tracing_on

echo "Tracing SPI events... Press Ctrl+C to stop"
sleep 10   # غيّرها حسب الحاجة

echo 0 > $TRACING/tracing_on
cat $TRACING/trace | grep -v "^#" | head -100
```

**تفعيل dynamic debug للـ SPI:**

```bash
# تفعيل كل debug في spi.c
echo "file drivers/spi/spi.c +p" > /sys/kernel/debug/dynamic_debug/control

# تأكد من تفعيله
grep "spi.c" /sys/kernel/debug/dynamic_debug/control | grep "=p"

# شغّل الـ device operation، ثم
dmesg | grep -i spi | tail -50

# تعطيل
echo "file drivers/spi/spi.c -p" > /sys/kernel/debug/dynamic_debug/control
```

**اختبار loopback بـ spidev_test:**

```bash
# بناء الأداة من source
cd linux-source/tools/spi
make spidev_test

# اختبار loopback (يحتاج MOSI مربوط بـ MISO)
./spidev_test -D /dev/spidev0.0 -s 500000 -v -p "\xAA\xBB\xCC\xDD"

# Output المتوقع في loopback:
# TX | AA BB CC DD
# RX | AA BB CC DD   <- نفس الـ data = شغّال
# RX | FF FF FF FF   <- كلها high = مشكلة
# RX | 00 00 00 00   <- كلها low  = مشكلة CS أو polarity
```

**فحص transfer timing بـ perf:**

```bash
# قياس زمن الـ SPI transfers
perf trace -e spi:spi_transfer_start,spi:spi_transfer_stop \
    -- sleep 5 2>&1 | head -50

# قياس latency من submit لـ done
perf trace -e spi:spi_message_submit,spi:spi_message_done \
    -- sleep 5 2>&1 | awk '
    /spi_message_submit/ { start=$1 }
    /spi_message_done/   { printf "latency: %.3f ms\n", ($1-start) }
'
```

**فحص الـ DMA errors:**

```bash
# لو في DMA issues
dmesg | grep -E "dma|DMA" | grep -i spi
cat /proc/interrupts | grep -i spi

# فحص الـ DMA channels للـ controller
ls /sys/class/dma/ | grep spi
```

**تقرير كامل للـ debugging:**

```bash
#!/bin/bash
echo "=== SPI Debug Report $(date) ==="

echo -e "\n--- Controllers ---"
for ctrl in /sys/class/spi_master/spi*; do
    echo "Controller: $(basename $ctrl)"
    if [ -d $ctrl/statistics ]; then
        echo "  messages:  $(cat $ctrl/statistics/messages)"
        echo "  errors:    $(cat $ctrl/statistics/errors)"
        echo "  timedout:  $(cat $ctrl/statistics/timedout)"
        echo "  bytes_tx:  $(cat $ctrl/statistics/bytes_tx)"
        echo "  bytes_rx:  $(cat $ctrl/statistics/bytes_rx)"
    fi
done

echo -e "\n--- Devices ---"
for dev in /sys/bus/spi/devices/spi*.*; do
    echo "Device: $(basename $dev)"
    echo "  modalias: $(cat $dev/modalias 2>/dev/null)"
    echo "  driver:   $(readlink $dev/driver 2>/dev/null | xargs basename 2>/dev/null || echo 'unbound')"
    if [ -d $dev/statistics ]; then
        echo "  errors:   $(cat $dev/statistics/errors)"
        echo "  timedout: $(cat $dev/statistics/timedout)"
    fi
done

echo -e "\n--- Kernel Messages ---"
dmesg --level=err,warn | grep -i spi | tail -20

echo -e "\n--- GPIO CS State ---"
cat /sys/kernel/debug/gpio 2>/dev/null | grep -i "spi.*cs" || echo "No GPIO CS found"
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: SPI Flash مش شغال على بورد RK3562 — Industrial Gateway

#### العنوان
**SPI NOR Flash مش بيظهر في النظام بعد bring-up بورد جديد**

#### السياق
بورد industrial gateway مبنية على RK3562، فيها W25Q128 SPI NOR Flash متوصلة على SPI1. البورد المفروض تعمل كـ data logger. المهندس عمل DT وكمّل الـ bring-up، بس `ls /dev/mtd*` مفيش حاجة.

#### المشكلة
الـ SPI Flash device مش بتظهر. الـ kernel log بيقول:
```
spi-rockchip ff5b0000.spi: registered spi1
```
بس مفيش `spi1.0` ولا `mtd0`.

#### التحليل
**الـ flow في `spi.c`:**

1. `spi_register_controller()` (السطر 3393) اتشغل صح وسجّل الـ controller.
2. بعدها اتنادى `of_register_spi_devices()` (السطر 3503) اللي بتلف على child nodes في DT.
3. `of_register_spi_device()` (السطر 2602) بتنادي `of_spi_parse_dt()` (السطر 2354).
4. جوّا `of_spi_parse_dt()`، بيحاول يقرأ `reg` property:
```c
rc = of_property_read_variable_u32_array(nc, "reg", &cs[0], 1,
                                         SPI_DEVICE_CS_CNT_MAX);
if (rc < 0) {
    dev_err(&ctlr->dev, "%pOF has no valid 'reg' property (%d)\n",
            nc, rc);
    return rc;
}
```
5. لو `reg` غلط أو مش موجود، الـ device مش بيتسجّل خالص.

**التحقيق:**

```bash
# شوف الـ DT node
cat /sys/firmware/devicetree/base/spi@ff5b0000/flash@0/reg | xxd
# شوف الـ kernel messages
dmesg | grep spi
```

المهندس لاقى في DT:
```dts
/* غلط — reg مش موجود */
flash@0 {
    compatible = "winbond,w25q128";
    spi-max-frequency = <50000000>;
    /* نسي reg = <0>; */
};
```

#### الحل

**تصحيح DT:**
```dts
&spi1 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&spi1m0_pins>;
    cs-gpios = <&gpio0 RK_PB4 GPIO_ACTIVE_LOW>;

    flash@0 {
        compatible = "winbond,w25q128", "jedec,spi-nor";
        reg = <0>;  /* CS index */
        spi-max-frequency = <50000000>;
    };
};
```

**تحقق بعد الإصلاح:**
```bash
ls /sys/bus/spi/devices/
# spi1.0
cat /sys/bus/spi/devices/spi1.0/modalias
# spi:w25q128
ls /dev/mtd*
# /dev/mtd0  /dev/mtd0ro
```

#### الدرس المستفاد
**الـ `of_spi_parse_dt()`** صارم — الـ `reg` property إجباري لكل SPI device node وبدونه الـ device مش بيتسجّل نهائياً. أي DT node تحت SPI controller لازم يكون فيه `reg` بيحدد رقم الـ chip select.

---

### السيناريو الثاني: SPI transfer timeout على STM32MP1 — IoT Sensor Hub

#### العنوان
**`SPI transfer timed out` بشكل متكرر مع sensor SPI على STM32MP157**

#### السياق
منتج IoT sensor hub على STM32MP157F، بيشغّل BME280 sensor على SPI2 بسرعة 10 MHz. الـ driver بيعمل read كل ثانية. بعد ساعات من الشغل، الـ kernel بيلوج:
```
spi2.0: SPI transfer timed out
```
وبعدها الـ sensor بيتوقف عن الاستجابة.

#### المشكلة
الـ transfer بتعمل timeout بعد فترة، ومش بترجع تشتغل إلا بعد reboot.

#### التحليل
**الـ `spi_transfer_wait()` في السطر 1431:**

```c
static int spi_transfer_wait(struct spi_controller *ctlr,
                             struct spi_message *msg,
                             struct spi_transfer *xfer)
{
    u32 speed_hz = xfer->speed_hz;
    unsigned long long ms;

    /* حساب الـ timeout */
    ms = 8LL * MSEC_PER_SEC * xfer->len;
    do_div(ms, speed_hz);
    ms += ms + 200;  /* ضعف + 200ms margin */

    ms = wait_for_completion_timeout(&ctlr->xfer_completion,
                                     msecs_to_jiffies(ms));
    if (ms == 0) {
        SPI_STATISTICS_INCREMENT_FIELD(statm, timedout);
        SPI_STATISTICS_INCREMENT_FIELD(stats, timedout);
        dev_err(&msg->spi->dev, "SPI transfer timed out\n");
        return -ETIMEDOUT;
    }
    ...
}
```

الـ `xfer_completion` مش بيكتمل لأن `spi_finalize_current_transfer()` مش بيتنادى. السبب: الـ STM32 SPI driver بيعتمد على interrupt، ولو الـ interrupt ضاع (مثلاً بسبب IRQ storm من sensor تاني)، الـ completion مش بيحصل.

**التحقق:**

```bash
# شوف sysfs statistics
cat /sys/bus/spi/devices/spi2.0/statistics/timedout
cat /sys/bus/spi/devices/spi2.0/statistics/errors

# شوف الـ IRQ counters
cat /proc/interrupts | grep spi

# استخدم tracing
echo 1 > /sys/kernel/debug/tracing/events/spi/enable
cat /sys/kernel/debug/tracing/trace | grep spi_transfer
```

الـ trace بيوري إن `spi_transfer_start` بيظهر بس مفيش `spi_transfer_stop` — يعني الـ interrupt مش واصل.

#### الحل

**الحل على مستوى الـ DT — تحديد IRQ affinity:**
```dts
&spi2 {
    status = "okay";
    /* تأكيد إن الـ SPI interrupt منفصل */
    interrupts = <GIC_SPI 36 IRQ_TYPE_LEVEL_HIGH>;
};
```

**الحل في الـ driver — استخدام polling fallback أو زيادة الـ timeout للـ sensor:**
```c
/* في الـ BME280 driver */
struct spi_transfer xfer = {
    .tx_buf = cmd,
    .rx_buf = resp,
    .len = 2,
    .speed_hz = 1000000,  /* قلّل السرعة لـ 1MHz عشان أمان أكتر */
};
```

**الحل الأشمل — enable real-time thread للـ SPI:**
```c
/* في الـ sensor driver probe */
spi->rt = true;  /* يخلي الـ kworker يشتغل بـ SCHED_FIFO */
```
ده بيحرّك `spi_set_thread_rt()` اللي بتنادي `sched_set_fifo(ctlr->kworker->task)`.

#### الدرس المستفاد
الـ `spi_transfer_wait()` بتحسب timeout ديناميكياً على أساس الـ transfer size والسرعة. لو الـ interrupt ضاع، الـ timeout هو الـ safety net الوحيدة. في الأنظمة الـ real-time، استخدم `spi->rt = true` عشان تقلل الـ latency وتضمن تنفيذ الـ message pump بأولوية عالية.

---

### السيناريو الثالث: GPIO Chip Select مش شغال صح على i.MX8MQ — Android TV Box

#### العنوان
**SPI touchscreen controller مش بيستجوب — GPIO CS polarity مقلوبة**

#### السياق
Android TV box على i.MX8MQ، بيشغّل Goodix GT9271 touchscreen متوصل على SPI4 بـ GPIO chip select بدل الـ native CS. الـ device بيظهر في النظام بس الـ touches مش شغالة. الـ oscilloscope بيوري إن الـ CS line بتتحرك معكوسة.

#### المشكلة
الـ CS GPIO بيطلع HIGH وهو المفروض يكون active (assert)، بدل ما يطلع LOW.

#### التحليل
**`spi_toggle_csgpiod()` في السطر 1038:**

```c
static void spi_toggle_csgpiod(struct spi_device *spi, u8 idx,
                                bool enable, bool activate)
{
    if (is_acpi_device_node(dev_fwnode(&spi->dev)))
        /* ACPI: logic معكوسة */
        gpiod_set_value_cansleep(spi_get_csgpiod(spi, idx), !enable);
    else
        /* OF/DT: الـ GPIO library بتتعامل مع الـ polarity */
        gpiod_set_value_cansleep(spi_get_csgpiod(spi, idx), activate);
    ...
}
```

والـ `spi_set_cs()` في السطر 1062 بتتحكم في `enable` بناءً على `SPI_CS_HIGH`:

```c
spi->controller->last_cs_mode_high = spi->mode & SPI_CS_HIGH;
if (spi->controller->last_cs_mode_high)
    enable = !enable;
```

المشكلة: المهندس عرّف الـ GPIO في DT بدون `GPIO_ACTIVE_LOW` flag:

```dts
/* غلط */
cs-gpios = <&gpio3 RK_PC0 GPIO_ACTIVE_HIGH>;
```

والـ `spi_get_gpio_descs()` (السطر 3266) بتجيب الـ GPIO descriptors:

```c
cs[i] = devm_gpiod_get_index_optional(dev, "cs", i,
                                      GPIOD_OUT_LOW);
```

الـ `GPIOD_OUT_LOW` بتعني "unasserted" — لو الـ GPIO declared كـ `ACTIVE_HIGH` في DT، الـ "unasserted" = physical LOW. لما الـ core بتعمل `gpiod_set_value(cs, 1)` عشان تـ assert، الـ GPIO library بتحوّلها لـ physical HIGH — وده غلط لـ chip بيحتاج active-low CS.

#### الحل

**تصحيح DT:**
```dts
&ecspi4 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_ecspi4>;
    /* CS GPIO active-low صح */
    cs-gpios = <&gpio3 20 GPIO_ACTIVE_LOW>;

    touchscreen@0 {
        compatible = "goodix,gt9271";
        reg = <0>;
        spi-max-frequency = <10000000>;
        /* لا تضيف spi-cs-high هنا */
    };
};
```

**التحقق بـ sysfs:**
```bash
# شوف حالة الـ GPIO قبل وبعد transfer
cat /sys/kernel/debug/gpio | grep cs
# لازم يبقى "out lo" وهو unasserted (CS غير مفعّل)
# وlازم يبقى "out lo" وهو asserted عشان active-low
```

**للتأكيد من polarity بـ oscilloscope:**
```bash
# لف transfer واحد وراقب CS line
spi-pipe /dev/spidev4.0 --speed 1000000 < /dev/zero | head -c 4
```

#### الدرس المستفاد
الـ SPI core بيفوّض الـ polarity handling للـ GPIO library عندما تكون OF node (مش ACPI). الـ `GPIO_ACTIVE_LOW` في DT هي اللي بتحدد الـ physical polarity الصح. مش المفروض تضيف `spi-cs-high` للـ device node إلا لو الـ chip فعلاً بيحتاج CS active-high. الفرق بين الاتنين بيسبّب headache كبيرة في bring-up.

---

### السيناريو الرابع: SPI DMA failure على AM62x — Automotive ECU

#### العنوان
**بيانات corrupted من SPI ADC في automotive ECU على TI AM62x**

#### السياق
ECU على TI AM62x بيقرأ data من ADS8688 16-channel ADC بـ SPI. بيعمل high-speed sampling بـ 50 MHz، الـ buffer sizes كبيرة (4KB per read). في load عالي، الـ data بتيجي corrupted أو كلها zeros.

#### المشكلة
الـ DMA transfer شغّال بس الـ data في الـ rx buffer مش صح في بعض الأحيان.

#### التحليل
الـ flow في `spi.c` للـ DMA:

1. **`spi_map_msg()` (السطر 1379):** بتحضر الـ DMA buffers وبتشوف لو الـ controller محتاج `dummy_tx` أو `dummy_rx`.

2. **`__spi_map_msg()` (السطر 1222):** بتعمل الـ mapping الفعلي:
```c
list_for_each_entry(xfer, &msg->transfers, transfer_list) {
    unsigned long attrs = DMA_ATTR_SKIP_CPU_SYNC;

    if (!ctlr->can_dma(ctlr, msg->spi, xfer))
        continue;

    if (xfer->rx_buf != NULL) {
        ret = spi_map_buf_attrs(ctlr, rx_dev, &xfer->rx_sg,
                                xfer->rx_buf, xfer->len,
                                DMA_FROM_DEVICE, attrs);
    }
}
```

3. **المشكلة:** الـ `DMA_ATTR_SKIP_CPU_SYNC` بيعني إن الـ sync بيتعمل manually. الـ sync بيحصل في `spi_dma_sync_for_cpu()` (السطر 1325):

```c
static void spi_dma_sync_for_cpu(struct spi_controller *ctlr,
                                 struct spi_transfer *xfer)
{
    struct device *rx_dev = ctlr->cur_rx_dma_dev;

    if (xfer->rx_sg_mapped)
        dma_sync_sgtable_for_cpu(rx_dev, &xfer->rx_sg,
                                 DMA_FROM_DEVICE);
    ...
}
```

المشكلة الفعلية: الـ driver كان بيعمل `kmalloc` بدون `GFP_DMA` flag للـ rx buffer في بعض الأحيان، فالـ buffer كانت بتيجي في memory خارج نطاق الـ DMA controller على AM62x.

**`spi_map_buf_attrs()` (السطر 1121):**
```c
} else if (virt_addr_valid(buf)) {
    desc_len = min_t(size_t, max_seg_size, ctlr->max_dma_len);
    sgs = DIV_ROUND_UP(len, desc_len);
} else {
    return -EINVAL;  /* بيرجع error لو الـ buffer مش valid */
}
```

لو الـ buffer في vmalloc space مثلاً، الكود بيتعامل معاه بطريقة مختلفة. بس لو في physical space بس خارج الـ DMA mask، الـ `dma_map_sgtable` بيفشل بـ silent mode في بعض الـ platforms.

#### الحل

**في الـ ADC driver — تأكيد الـ DMA-safe allocation:**
```c
/* صح: allocate DMA-safe buffer */
rx_buf = kmalloc(transfer_size, GFP_KERNEL | GFP_DMA);
if (!rx_buf)
    return -ENOMEM;
```

**التحقق من الـ DMA stats:**
```bash
# شوف الـ transfers_split_maxsize
cat /sys/bus/spi/devices/spi0.0/statistics/transfers_split_maxsize
# لو عدد كبير، يعني الـ transfers بتتقسم بسبب DMA size limits

# شوف الـ errors
cat /sys/bus/spi/devices/spi0.0/statistics/errors
cat /sys/class/spi_master/spi0/statistics/errors
```

**تحقق من الـ DMA mask للـ AM62x:**
```bash
cat /sys/bus/platform/devices/20100000.spi/dma_mask
# لازم يكون 32-bit على الأقل
```

**إصلاح الـ controller DT لو لازم:**
```dts
&mcspi0 {
    status = "okay";
    dmas = <&edma 16 0>, <&edma 17 0>;
    dma-names = "tx", "rx";
    /* تأكد من الـ DMA channel config */
};
```

#### الدرس المستفاد
الـ `DMA_ATTR_SKIP_CPU_SYNC` بيعني إنك مسؤول عن الـ cache sync. الـ SPI core بتعمل ده automatically في `spi_dma_sync_for_cpu()` و `spi_dma_sync_for_device()`. المشكلة دايماً بتيجي من الـ buffer allocation — لازم دايماً تستخدم `GFP_DMA` أو `dma_alloc_coherent()` للـ high-speed SPI transfers على embedded platforms.

---

### السيناريو الخامس: SPI device conflict على Allwinner H616 — Custom Board Bring-up

#### العنوان
**`chipselect already in use` error عند تسجيل SPI devices على Allwinner H616**

#### السياق
custom board على Allwinner H616 (Orange Pi Zero 2 variant) بيشغّل SPI display (ST7789) وSPI flash (GD25Q64) على نفس الـ SPI0 controller بـ CS0 وCS1. الـ kernel log بيطلع:

```
spi0: chipselect 0 already in use
spi0: can't create new device for st7789v
```

#### المشكلة
الـ kernel مش بيقدر يسجّل الـ SPI display لأنه شايف إن CS0 متأخدة.

#### التحليل
**`spi_dev_check()` (السطر 644) و `spi_dev_check_cs()` (السطر 626):**

```c
static int spi_dev_check(struct device *dev, void *data)
{
    struct spi_device *spi = to_spi_device(dev);
    struct spi_device *new_spi = data;
    int status, idx;

    if (spi->controller == new_spi->controller) {
        for (idx = 0; idx < spi->num_chipselect; idx++) {
            status = spi_dev_check_cs(dev, spi, idx, new_spi, 0);
            if (status)
                return status;
        }
    }
    return 0;
}
```

و `spi_dev_check_cs()`:
```c
static inline int spi_dev_check_cs(struct device *dev,
                                   struct spi_device *spi, u8 idx,
                                   struct spi_device *new_spi, u8 new_idx)
{
    u8 cs, cs_new;
    u8 idx_new;

    cs = spi_get_chipselect(spi, idx);
    for (idx_new = new_idx; idx_new < new_spi->num_chipselect; idx_new++) {
        cs_new = spi_get_chipselect(new_spi, idx_new);
        if (cs == cs_new) {
            dev_err(dev, "chipselect %u already in use\n", cs_new);
            return -EBUSY;
        }
    }
    return 0;
}
```

`__spi_add_device()` (السطر 666) بتنادي `bus_for_each_dev()` لتتحقق من conflicts قبل ما تضيف الـ device:

```c
status = bus_for_each_dev(&spi_bus_type, NULL, spi, spi_dev_check);
if (status)
    return status;
```

**السبب الفعلي:** المهندس كان عنده DT قديم بيسجّل الـ flash على CS0 بردو في `board_info` static registration، وفي نفس الوقت الـ DT بيحاول يسجّل الـ display على CS0 كمان.

**التحقيق:**

```bash
# شوف الـ SPI devices الموجودة حالياً
ls /sys/bus/spi/devices/
# spi0.0  <--- ده موجود بالفعل

# شوف modalias بتاعه
cat /sys/bus/spi/devices/spi0.0/modalias
# spi:gd25q64  <--- الـ flash اتسجّل على CS0

# شوف الـ DT
dtc -I fs /sys/firmware/devicetree/base | grep -A10 "spi@"
```

المشكلة: الـ flash اتسجّل على CS0 من static `board_info`، والـ display في DT كمان على CS0.

#### الحل

**تصحيح DT — كل device لازم على CS مختلف:**
```dts
&spi0 {
    status = "okay";
    #address-cells = <1>;
    #size-cells = <0>;

    /* Flash على CS0 */
    flash@0 {
        compatible = "gigadevice,gd25q64", "jedec,spi-nor";
        reg = <0>;
        spi-max-frequency = <80000000>;
    };

    /* Display على CS1 */
    display@1 {
        compatible = "sitronix,st7789v";
        reg = <1>;
        spi-max-frequency = <30000000>;
        dc-gpios = <&pio 2 5 GPIO_ACTIVE_HIGH>;
        reset-gpios = <&pio 2 6 GPIO_ACTIVE_LOW>;
    };
};
```

**لو في static board_info قديم لازم يتشال:**
```c
/* في board file — احذف التسجيل القديم اللي بيتعارض */
static struct spi_board_info h616_spi_board_info[] = {
    /* احذف أو صحّح الـ chip_select */
    {
        .modalias       = "gd25q64",
        .max_speed_hz   = 80000000,
        .bus_num        = 0,
        .chip_select    = 0,  /* CS0 فقط للـ flash */
    },
    /* مش محتاج entry للـ display هنا لأنه في DT */
};
```

**التحقق بعد الإصلاح:**
```bash
ls /sys/bus/spi/devices/
# spi0.0  spi0.1

cat /sys/bus/spi/devices/spi0.0/modalias
# spi:gd25q64

cat /sys/bus/spi/devices/spi0.1/modalias
# spi:st7789v
```

**لفهم `__spi_add_device()` flow كامل:**
```
spi_add_device()
    └── mutex_lock(&ctlr->add_lock)
    └── __spi_add_device()
            ├── validate CS range (< ctlr->num_chipselect)
            ├── bus_for_each_dev() → spi_dev_check() → spi_dev_check_cs()
            ├── spi_setup()
            └── device_add()
```

#### الدرس المستفاد
الـ `spi_dev_check_cs()` بتمنع duplicate CS assignments على نفس الـ controller. في bring-up, تأكد دايماً إنه مفيش device مسجّل على نفس الـ CS من طريقين مختلفين (DT + static board_info). استخدم `ls /sys/bus/spi/devices/` أول ما يبدأ الـ boot عشان تشوف إيه اللي اتسجّل قبل ما تضيف devices جديدة.
## Phase 7: مصادر ومراجع

### توثيق Kernel الرسمي

| المسار | المحتوى |
|--------|---------|
| `Documentation/spi/spi-summary.rst` | نظرة عامة على الـ SPI subsystem — نقطة البداية المثالية |
| `Documentation/spi/spidev.rst` | الـ userspace API عبر `/dev/spidevX.Y` |
| `Documentation/spi/spi-lld.rst` | كتابة low-level SPI controller driver |
| `include/linux/spi/spi.h` | تعريفات `spi_device`, `spi_controller`, `spi_transfer`, `spi_message` |
| `include/linux/spi/spi-mem.h` | الـ spi-mem layer للـ flash memories |
| `drivers/spi/spi.c` | الـ core — كل ما تمت دراسته في هذا الملف |
| `drivers/spi/spi-bitbang.c` | مثال bitbang controller driver |

الـ online rendered version:
- [Serial Peripheral Interface (SPI) — driver-api](https://docs.kernel.org/driver-api/spi.html)
- [Overview of Linux kernel SPI support](https://docs.kernel.org/spi/spi-summary.html)
- [SPI userspace API (spidev)](https://www.kernel.org/doc/html/latest/spi/spidev.html)

---

### مقالات LWN.net

الـ **LWN.net** هو المرجع الأساسي لمتابعة تطور الـ kernel — الروابط دي حقيقية ومأخوذة من البحث مباشرة.

| المقال | الأهمية |
|--------|---------|
| [SPI core (2005)](https://lwn.net/Articles/138037/) | أول إعلان عن الـ SPI core في الـ kernel — بقلم David Brownell |
| [simple SPI framework (2005)](https://lwn.net/Articles/154425/) | الـ patch الأصلي لتبسيط الـ SPI framework |
| [SPI redux … driver model support (2005)](https://lwn.net/Articles/149698/) | تطور الـ driver model داخل الـ SPI subsystem |
| [spi (patch thread)](https://lwn.net/Articles/146581/) | thread مناقشة الـ patch الأولية |
| [spi: Add slave mode support (2017)](https://lwn.net/Articles/700433/) | إضافة دعم SPI slave mode لأول مرة |
| [spi: Add slave mode support — follow-up](https://lwn.net/Articles/723440/) | متابعة الـ slave mode implementation |
| [spi: spi-mem: Add a driver for NXP FlexSPI](https://lwn.net/Articles/763880/) | driver مبني على الـ spi-mem framework الجديد |
| [spi: spi-mem: NXP FlexSPI (revised)](https://lwn.net/Articles/768160/) | النسخة المُراجَعة من نفس الـ driver |
| [spi: Add the SPI daisy chain support](https://lwn.net/Articles/825229/) | إضافة دعم الـ daisy chain topology |
| [SPI LPC information kernel module](https://lwn.net/Articles/824829/) | استخدام SPI مع LPC bus |
| [spi: stm32: add spi slave mode](https://lwn.net/Articles/930471/) | تطبيق slave mode على الـ STM32 |

---

### Bootlin — مقالات تقنية متعمقة

**Bootlin** (سابقاً Free Electrons) من أفضل مصادر التعمق في الـ embedded Linux drivers.

- [spi-mem: bringing some consistency to the SPI memory ecosystem](https://bootlin.com/blog/spi-mem-bringing-some-consistency-to-the-spi-memory-ecosystem/)
  — شرح مفصّل ليه اتعمل الـ `spi-mem` layer وإزاي بيوحّد الـ SPI NOR و SPI NAND drivers

---

### eLinux.org — أمثلة على hardware حقيقي

مفيدة لفهم الـ SPI في سياق embedded boards:

| الصفحة | المحتوى |
|--------|---------|
| [RPi SPI](https://elinux.org/RPi_SPI) | إعداد SPI على Raspberry Pi، الـ `spi_bcm2708` driver، الـ device tree overlay |
| [BeagleBoard/SPI](https://elinux.org/BeagleBoard/SPI) | الـ `spi_board_info` struct والـ board file configuration |
| [ECE497 SPI Project](https://elinux.org/ECE497_SPI_Project) | مشروع تعليمي كامل لاستخدام SPI على BeagleBone |
| [Tests:MSIOF-SPI-CS-GPIO](https://elinux.org/Tests:MSIOF-SPI-CS-GPIO) | اختبار الـ chip-select عبر GPIO مع MSIOF controller |

---

### KernelNewbies.org — تتبع تطور الـ SPI عبر إصدارات الـ kernel

مفيد لمعرفة متى اتضاف كل feature:

| الصفحة | تغييرات SPI ذات صلة |
|--------|---------------------|
| [Linux 2.6.24](https://kernelnewbies.org/Linux_2_6_24) | إضافة SPI/SDIO MMC support — SPI أصبح متطلباً لبعض الـ MMC controllers |
| [Linux 3.0 DriverArch](https://kernelnewbies.org/Linux_3.0_DriverArch) | تغييرات معمارية في الـ SPI driver layer |
| [Linux 3.9 DriverArch](https://kernelnewbies.org/Linux_3.9_DriverArch) | تحسينات الـ SPI في دورة 3.9 |
| [Linux 4.12](https://kernelnewbies.org/Linux_4.12) | تحديثات الـ SPI subsystem |
| [Linux 6.8](https://kernelnewbies.org/Linux_6.8) | أحدث تغييرات الـ SPI في kernel 6.8 |
| [Linux 6.11](https://kernelnewbies.org/Linux_6.11) | تحسينات الـ SPI في 6.11 |
| [Linux 6.14](https://kernelnewbies.org/Linux_6.14) | تغييرات الـ SPI في 6.14 |
| [Linux 6.16](https://kernelnewbies.org/Linux_6.16) | أحدث إصدار موثق |

---

### Mailing List — متابعة التطوير الحالي

**الـ SPI mailing list الرسمي:**

```
linux-spi@vger.kernel.org
```

أرشيف الـ lore.kernel.org:
- [https://lore.kernel.org/linux-spi/](https://lore.kernel.org/linux-spi/)

كل الـ patches والنقاشات المتعلقة بالـ SPI subsystem بتعدي على هذا الـ list قبل الـ merge.

الـ maintainer الحالي: **Mark Brown** (`broonie@kernel.org`)
الـ git tree الخاص بيه: `git://git.kernel.org/pub/scm/linux/kernel/git/broonie/spi.git`

---

### Commits مهمة في تاريخ الـ SPI

| التغيير | التفاصيل |
|--------|---------|
| الإدخال الأولي للـ SPI core | David Brownell — Linux 2.6.14 (2005) |
| إضافة `spi-mem` layer | Boris Brezillon — Linux 4.18 (2018) |
| إضافة SPI slave mode | Geert Uytterhoeven — Linux 4.13 (2017) |
| إضافة `spi_offload` framework | David Lechner — Linux 6.13 (2024) — موجود في `include/linux/spi/offload/types.h` الذي يُضمَّن في الملف المدروس |

للبحث في تاريخ commit معيّن:
```bash
git log --oneline drivers/spi/spi.c | head -30
git log --oneline --follow include/linux/spi/spi.h | head -20
```

---

### كتب موصى بها

#### Linux Device Drivers 3rd Edition (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصل المهم:** Chapter 6 — *Advanced Char Driver Operations* (مفاهيم عامة مفيدة للـ bus drivers)
- **ملاحظة:** الكتاب قديم (kernel 2.6.10) لكن المفاهيم الأساسية للـ driver model لا تزال صالحة
- **متاح مجاناً:** [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصول المفيدة:**
  - Chapter 17: *Devices and Modules* — الـ device model وكيف يرتبط بيه الـ SPI
  - Chapter 13: *The Virtual Filesystem* — فهم الـ sysfs المستخدم في الـ SPI attributes
- **الميزة:** شرح واضح للـ `kobject`, `kset`, وبنية الـ bus في الـ kernel

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصول المفيدة:**
  - Chapter 8: *Device Drivers* — مقدمة للـ SPI في السياق الـ embedded
  - Chapter 15: *Embedded Linux Quick Start* — أمثلة على الـ board bring-up مع SPI devices
- **الميزة:** يشرح الـ `spi_board_info` والـ platform data بأمثلة حقيقية

#### Mastering Embedded Linux Programming — Frank Vasquez & Chris Simmonds (3rd Edition)
- فصل كامل عن الـ SPI وI2C في الـ embedded Linux
- أمثلة على BeagleBone وRaspberry Pi

---

### مصادر إضافية على الإنترنت

| المصدر | الرابط | ما ستجده |
|--------|--------|---------|
| توثيق kernel الرسمي | [docs.kernel.org/driver-api/spi.html](https://docs.kernel.org/driver-api/spi.html) | المرجع الكامل للـ SPI API |
| Bootlin Elixir (cross-reference) | [elixir.bootlin.com/linux/latest/source/drivers/spi/spi.c](https://elixir.bootlin.com/linux/latest/source/drivers/spi/spi.c) | قراءة الكود مع روابط للـ definitions |
| git.kernel.org (SPI tree) | [git.kernel.org/broonie/spi](https://git.kernel.org/pub/scm/linux/kernel/git/broonie/spi.git) | أحدث التغييرات قبل الـ merge |
| Analog Devices Wiki | [wiki.analog.com — SPI Driver](https://wiki.analog.com/resources/tools-software/linuxdsp/docs/linux-kernel-and-drivers/spi/spi) | شرح تطبيقي من شركة hardware |
| embetronicx | [SPI Device Driver Tutorial](https://embetronicx.com/tutorials/linux/device-drivers/linux-kernel-spi-device-driver-tutorial/) | tutorial خطوة بخطوة لكتابة SPI driver |

---

### Search Terms للبحث عن معلومات أكثر

للبحث في الـ mailing list:
```
site:lore.kernel.org linux-spi spi_message
site:lore.kernel.org linux-spi spi_controller
site:lore.kernel.org linux-spi offload
```

للبحث في الـ LWN.net:
```
site:lwn.net SPI controller driver
site:lwn.net spi-mem flash
site:lwn.net "struct spi_transfer"
```

للبحث العام:
```
linux kernel SPI subsystem internals
linux "spi_sync" vs "spi_async" implementation
linux SPI DMA scatter-gather transfer
linux SPI chip select GPIO
linux SPI device tree binding
"spi_controller_ops" linux kernel
```
## Phase 8: Writing simple module

### الفكرة: Hook على الـ tracepoint `spi_transfer_start`

الـ `spi_transfer_start` هو tracepoint معرَّف في `trace/events/spi.h` ومُصدَّر عبر `EXPORT_TRACEPOINT_SYMBOL` في `drivers/spi/spi.c`. بيتفعل في كل مرة بيبدأ فيها SPI transfer جديد — ده بيخليه أداة مثالية للـ monitoring لأنه بيديك اسم الـ bus، رقم الـ chip select، وحجم الـ transfer في نفس اللحظة اللي بيبدأ فيها.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * spi_probe_mod.c
 *
 * Hooks the spi_transfer_start tracepoint to log every SPI
 * transfer that starts anywhere on the system.
 *
 * Build with a Makefile that sets KDIR to your kernel build tree:
 *   obj-m += spi_probe_mod.o
 */

#include <linux/module.h>       /* MODULE_* macros, module_init/exit        */
#include <linux/kernel.h>       /* pr_info()                                 */
#include <linux/tracepoint.h>   /* for_each_kernel_tracepoint, tracepoint_probe_register */
#include <linux/spi/spi.h>      /* struct spi_message, struct spi_transfer   */
#include <trace/events/spi.h>   /* spi_transfer_start tracepoint declaration */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("SPI Watcher <example@kernel.org>");
MODULE_DESCRIPTION("Trace every SPI transfer_start event system-wide");

/* ------------------------------------------------------------------ */
/*  Tracepoint callback                                                 */
/* ------------------------------------------------------------------ */

/*
 * spi_transfer_start fires with (msg, xfer):
 *   msg  -> struct spi_message* — the parent message being processed
 *   xfer -> struct spi_transfer* — the individual transfer segment
 *
 * The third arg 'data' is the private pointer we pass at registration
 * (NULL here, so we ignore it).
 */
static void tp_spi_xfer_start(void *data,
                               struct spi_message  *msg,
                               struct spi_transfer *xfer)
{
    /* spi_get_chipselect() returns the physical CS index for slot 0.
     * msg->spi->controller->bus_num is the SPI bus number (e.g. 0 for spi0). */
    int  bus_num = msg->spi->controller->bus_num;
    int  cs      = spi_get_chipselect(msg->spi, 0);
    bool has_tx  = (xfer->tx_buf != NULL);
    bool has_rx  = (xfer->rx_buf != NULL);

    pr_info("spi_watch: spi%d.%d  len=%u  %s%s  speed=%u Hz\n",
            bus_num, cs,
            xfer->len,
            has_tx ? "TX " : "",
            has_rx ? "RX " : "",
            xfer->speed_hz);     /* 0 means "use device default" */
}

/* ------------------------------------------------------------------ */
/*  Module init / exit                                                  */
/* ------------------------------------------------------------------ */

static int __init spi_probe_init(void)
{
    int ret;

    /*
     * register_trace_spi_transfer_start() is the typed wrapper that the
     * DEFINE_EVENT macro generates for us.  It wires our callback into
     * the tracepoint's probe list so it will be called on every firing.
     */
    ret = register_trace_spi_transfer_start(tp_spi_xfer_start, NULL);
    if (ret) {
        pr_err("spi_watch: failed to register tracepoint probe (%d)\n", ret);
        return ret;
    }

    pr_info("spi_watch: loaded — watching all SPI transfer_start events\n");
    return 0;
}

static void __exit spi_probe_exit(void)
{
    /*
     * Must unregister before unloading.  If we skip this the tracepoint
     * subsystem will call a function pointer into freed module memory the
     * next time any SPI transfer starts → guaranteed kernel panic.
     */
    unregister_trace_spi_transfer_start(tp_spi_xfer_start, NULL);

    /* tracepoint_synchronize_unregister() waits for any currently running
     * probe invocation to finish before we return, making the unload safe. */
    tracepoint_synchronize_unregister();

    pr_info("spi_watch: unloaded\n");
}

module_init(spi_probe_init);
module_exit(spi_probe_exit);
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|---|---|
| `<linux/module.h>` | `module_init` / `module_exit` + الـ `MODULE_*` macros |
| `<linux/kernel.h>` | `pr_info()` / `pr_err()` |
| `<linux/tracepoint.h>` | `register_trace_*` و `tracepoint_synchronize_unregister` |
| `<linux/spi/spi.h>` | `struct spi_message`, `struct spi_transfer`, `spi_get_chipselect()` |
| `<trace/events/spi.h>` | التصريح بالـ tracepoint نفسه و wrapper functions اللي بييجوا منه |

**الـ `<trace/events/spi.h>`** مش بس declaration — لما بتحطه في module بدون `CREATE_TRACE_POINTS` بييجيب تعريفات `register_trace_spi_transfer_start` و`unregister_trace_spi_transfer_start` الجاهزة.

---

#### الـ callback `tp_spi_xfer_start`

الـ prototype لازم يطابق signature الـ tracepoint بالظبط:

```
TP_PROTO(struct spi_message *msg, struct spi_transfer *xfer)
```

الـ kernel بيضيف `void *data` في الأول كـ private cookie — محتاجين نحطه حتى لو مش بنستخدمه.

- **`bus_num`**: رقم الـ SPI bus — بيجي من `msg->spi->controller->bus_num`.
- **`cs`**: رقم الـ chip select الفيزيائي — بنجيبه عبر `spi_get_chipselect()` لأن الـ device ممكن يكون متعدد الـ CS.
- **`xfer->len`**: حجم الـ transfer بالبايت.
- **`xfer->speed_hz`**: السرعة المطلوبة للـ transfer ده تحديداً (ممكن تبطل default الـ device).

الـ pr_info ده بيطلع في `dmesg` وبيكون كافي لأي debugging أو تتبع.

---

#### الـ `module_init`

`register_trace_spi_transfer_start` بتضيف الـ callback في قائمة الـ probes الخاصة بالـ tracepoint. لو فشلت (مثلاً لأن الـ tracepoint مش موجود في الـ build)، بنرجع الـ error فوراً بدل ما نكمل.

---

#### الـ `module_exit`

المرحلتين الاتنين في الـ exit **إلزاميتين**:

1. **`unregister_trace_spi_transfer_start`** — بتشيل الـ callback من قائمة الـ probes.
2. **`tracepoint_synchronize_unregister`** — بتستنى على أي CPU تاني ممكن يكون في منتصف تنفيذ الـ probe دلوقتي. من غيرها ممكن يحصل use-after-free لما الـ kernel يحاول ينفذ callback في memory اتحررت بعد `rmmod`.

---

### Makefile للبناء

```makefile
obj-m += spi_probe_mod.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

### تشغيل وتجربة

```bash
# بناء الـ module
make KDIR=/path/to/kernel/build

# تحميل
sudo insmod spi_probe_mod.ko

# متابعة الـ output (في نظام فيه SPI devices)
sudo dmesg -w | grep spi_watch

# مثال output لـ sensor على spi0.0 بيتكلم
# spi_watch: spi0.0  len=4  TX RX  speed=1000000 Hz
# spi_watch: spi0.0  len=16 TX     speed=0 Hz

# تفريغ
sudo rmmod spi_probe_mod
```

> **ملاحظة:** على جهاز مفيهوش SPI hardware حقيقي (مثل VM عادية) الـ callback مش هيتنادى — الـ module هيتحمل بسلامة بس مش هيطبع غير رسالة الـ load. لو عايز تتجرب على VM، استخدم `spi-loopback-test` kernel driver أو أي SPI emulator.
