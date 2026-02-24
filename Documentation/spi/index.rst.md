## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ SPI Subsystem؟

**SPI (Serial Peripheral Interface)** هو بروتوكول اتصال تسلسلي بسيط جداً، اخترعه موتورولا في الثمانينات، وبقي معيار "de facto" — يعني محدش رسمي بييه، بس الكل بيستخدمه. الـ Linux kernel عنده subsystem كامل لإدارة أجهزة SPI، واللي موجود في `Documentation/spi/` ده هو التوثيق الرسمي لكل حاجة في الـ subsystem ده.

---

### تخيل الموضوع كده

تخيل معايا إنك عندك **Arduino** أو microcontroller، وعايز تحط عليه:
- شاشة صغيرة
- بطاقة ذاكرة SD
- حساس درجة حرارة
- مبدل analog-to-digital

كل الأجهزة دي ممكن تتكلم بـ SPI. السؤال: **كيف؟**

SPI بيستخدم 4 أسلاك بس:

```
                    ┌─────────────────┐
                    │   CPU / SOC     │
                    │  (SPI Master)   │
                    └────┬────┬───┬───┘
                         │    │   │
              SCK ───────┘    │   └─── MOSI
              MISO ──────────┘
                         │
              nCS0 ───── Device 0 (مثلاً: Flash)
              nCS1 ───── Device 1 (مثلاً: ADC)
              nCS2 ───── Device 2 (مثلاً: touchscreen)
```

- **SCK (Clock)**: المعالج بيبعت نبضات clock لكل الأجهزة.
- **MOSI (Master Out Slave In)**: البيانات من المعالج للجهاز.
- **MISO (Master In Slave Out)**: البيانات من الجهاز للمعالج.
- **nCS (Chip Select)**: خط لكل جهاز، لما بيتفعّل بيقول للجهاز "إنت اللي بتتكلم دلوقتي".

الجميل إن SPI **full-duplex** — بيبعت ويستقبل في نفس الوقت!

---

### ليه الـ Linux محتاج Subsystem كامل لده؟

لأن المشكلة مش في الأسلاك، المشكلة في **التعقيد الحقيقي**:

1. **عندك مئات الأجهزة المختلفة** — كل واحد بروتوكوله الخاص، clock mode مختلف، word size مختلف، timing مختلف.
2. **عندك عشرات الـ controllers المختلفة** — Broadcom, Atmel, NXP, Qualcomm, Raspberry Pi, كل واحد بـ hardware registers مختلفة.
3. **محتاج تشغّل أكتر من جهاز على نفس الـ bus** — ومحتاج لـ locking وقوائم انتظار.
4. **بعض الأجهزة محتاجة DMA** — عشان تنقل بيانات كبيرة من غير ما تلخبط الـ CPU.

الحل اللي عمله Linux هو **فصل طبقتين**:

```
┌──────────────────────────────────────────────┐
│           Protocol Drivers (طبقة أعلى)        │
│   مثال: SPI Flash driver, touchscreen driver  │
│   بيعرف "إيه اللي مكتوب في الـ SPI message"  │
└────────────────────┬─────────────────────────┘
                     │  spi_message / spi_transfer
┌────────────────────▼─────────────────────────┐
│           SPI Core (القلب)                    │
│   drivers/spi/spi.c                           │
│   بيدير الـ queue والـ locking والـ DMA        │
└────────────────────┬─────────────────────────┘
                     │ hardware ops
┌────────────────────▼─────────────────────────┐
│        Controller Drivers (طبقة أدنى)         │
│   مثال: spi-bcm2835.c, spi-atmel.c            │
│   بيعرف "كيف تبعت البيانات على الـ hardware"  │
└──────────────────────────────────────────────┘
```

---

### الـ Documentation/spi/index.rst — إيه اللي بيعمله؟

الملف ده هو **فهرس التوثيق** للـ SPI subsystem بالكامل. مش كود — ده منظم كـ RST (reStructuredText) ويُبنى لـ Sphinx docs. بيجمع الملفات دي تحته:

| الملف | المحتوى |
|---|---|
| `spi-summary.rst` | الشرح الشامل للـ subsystem — البروتوكول، الـ API، كيف تكتب driver |
| `spidev.rst` | الـ userspace API — `/dev/spidevB.C` وإزاي تكلّم SPI من user space |
| `multiple-data-lanes.rst` | أجهزة SPI بأكتر من data lane (للـ ADCs والـ parallel flash) |
| `butterfly.rst` | مثال hardware: Butterfly evaluation board من Atmel |
| `spi-lm70llp` | مثال hardware: LM70 temperature sensor عبر SPI |
| `spi-sc18is602` | مثال hardware: SC18IS602 SPI-to-I2C bridge |

---

### القصة الكاملة: إزاي بيشتغل

**السيناريو**: عندك Raspberry Pi وعايز تقرأ من SPI Flash chip.

1. **Board init / Device Tree** بيعرّف إن في SPI controller رقم 0، وفيه flash chip على chip select 0:
   ```c
   static struct spi_board_info spi_board_info[] __initdata = {
   {
       .modalias    = "m25p80",       /* اسم الـ driver */
       .max_speed_hz = 10000000,      /* 10 MHz */
       .bus_num     = 0,
       .chip_select = 0,
       .mode        = SPI_MODE_0,
   },
   };
   ```

2. **Controller driver** (`spi-bcm2835.c`) بيسجّل نفسه كـ `spi_controller` ويعمل `spi_register_controller()`.

3. **SPI Core** بيشوف إن في `m25p80` مسجّل في `spi_board_info`، فبيبحث عن driver اسمه "m25p80" ويعمل له `probe()`.

4. **Protocol driver** (الـ m25p80 flash driver) جوا `probe()` بيبعت `spi_message` يطلب قراءة bytes من الـ flash.

5. **SPI Core** بياخد الـ message، بيحطه في queue، وبيوديه للـ controller driver.

6. **Controller driver** بيبعت البيانات فعلياً على الأسلاك بـ DMA أو PIO.

---

### الـ 4 Clock Modes — عشان ميتلخبطش

```
Mode | CPOL | CPHA | الـ Clock بيبدأ | بيقرأ عند
-----|------|------|----------------|----------
  0  |  0   |  0   | Low            | Rising edge
  1  |  0   |  1   | Low            | Falling edge
  2  |  1   |  0   | High           | Falling edge
  3  |  1   |  1   | High           | Rising edge
```

معظم الأجهزة بتستخدم **Mode 0** أو **Mode 3**.

---

### الـ Userspace API (spidev)

Linux بيسمح برامج الـ userspace تكلّم SPI مباشرة من غير ما تكتب kernel driver، عبر:

```bash
/dev/spidev0.0   # bus 0, chip select 0
/dev/spidev0.1   # bus 0, chip select 1
```

```c
/* مثال: قراءة byte واحد من SPI device */
int fd = open("/dev/spidev0.0", O_RDWR);

struct spi_ioc_transfer xfer = {
    .tx_buf = (unsigned long)tx_buf,
    .rx_buf = (unsigned long)rx_buf,
    .len    = 1,
    .speed_hz = 1000000,
    .bits_per_word = 8,
};

ioctl(fd, SPI_IOC_MESSAGE(1), &xfer);
```

---

### ملفات الـ Subsystem المهمة

#### الـ Core

| الملف | الوظيفة |
|---|---|
| `drivers/spi/spi.c` | قلب الـ subsystem — الـ queue، الـ API، التسجيل |
| `drivers/spi/internals.h` | تعريفات داخلية للـ core |
| `drivers/spi/spi-mem.c` | دعم SPI memory operations (Flash) |

#### الـ Headers

| الملف | الوظيفة |
|---|---|
| `include/linux/spi/spi.h` | التعريفات الأساسية: `spi_device`, `spi_controller`, `spi_message`, `spi_transfer` |
| `include/linux/spi/spi-mem.h` | واجهة SPI memory |
| `include/uapi/linux/spi/spidev.h` | الـ ioctl definitions للـ userspace |

#### الـ Controller Drivers (أمثلة)

| الملف | الـ Hardware |
|---|---|
| `drivers/spi/spi-bcm2835.c` | Raspberry Pi |
| `drivers/spi/spi-atmel.c` | Atmel ARM SOCs |
| `drivers/spi/spi-pl022.c` | ARM PrimeCell SSP |
| `drivers/spi/spi-imx.c` | NXP i.MX |
| `drivers/spi/spi-nxp-fspi.c` | NXP FlexSPI |
| `drivers/spi/spi-bitbang.c` | GPIO bitbang (أي pin يصير SPI) |

#### الـ Documentation

| الملف | المحتوى |
|---|---|
| `Documentation/spi/spi-summary.rst` | الشرح الشامل والـ API |
| `Documentation/spi/spidev.rst` | الـ userspace API |
| `Documentation/spi/multiple-data-lanes.rst` | أجهزة multi-lane |
| `Documentation/devicetree/bindings/spi/` | Device Tree bindings |

#### الـ Tools

| الملف | الوظيفة |
|---|---|
| `tools/spi/spidev_test.c` | أداة اختبار SPI من command line |
| `tools/spi/spidev_fdx.c` | مثال full-duplex transfer |

---

### المعتمد عليه في الـ MAINTAINERS

**الـ Maintainer**: Mark Brown `<broonie@kernel.org>`
**الـ Mailing List**: `linux-spi@vger.kernel.org`
**الـ Tree**: `git://git.kernel.org/pub/scm/linux/kernel/git/broonie/spi.git`

الـ SPI subsystem فيه أكتر من **180 controller driver** في `drivers/spi/`، مما يخليه من أكبر subsystems في الـ kernel من ناحية عدد الـ drivers.
## Phase 2: شرح الـ SPI Framework

### المشكلة اللي الـ SPI Subsystem بيحلها

في الـ embedded systems، عندك microcontroller أو processor بيحتاج يتكلم مع chips تانية: flash memory، ADC، touchscreen، codec. الـ hardware بيوفر **SPI controller** جوا الـ SoC — لكن لو كل driver كتبه من الصفر لازم يعرف كل تفاصيل الـ controller الخاصة بالـ SoC دي، هيبقى في كود مكرر وصعوبة في الـ portability.

المشكلة في نقطتين:
1. **التنوع في الـ controllers**: كل SoC ليه SPI controller مختلف (Broadcom، Freescale، Rockchip...) بـ registers مختلفة.
2. **التنوع في الـ devices**: كل device على الـ bus ليه protocol خاص بيه (flash بتخاطبه بطريقة، touchscreen بطريقة تانية).

من غير framework، كل مجموعة controller+device هتحتاج driver مكتوب خصيصاً لبعض.

---

### الحل: فصل الـ Controller عن الـ Protocol

الـ Linux SPI subsystem بيفصل المسؤوليات في طبقتين:

| الطبقة | المسؤولية | اسمها في الكود |
|--------|-----------|----------------|
| **Controller Driver** | يتكلم مع الـ hardware registers، يعمل DMA، يحرك الـ SCK | `spi_controller` |
| **Protocol Driver** | يعرف protocol الـ device — أوامر، بيانات، timing | `spi_driver` |

الـ core بيوقف في النص — بيدير الـ queue، بيعمل الـ matching بين الطبقتين، وبيوفر API موحد للـ protocol drivers.

---

### تشبيه من الواقع — البريد السريع

تخيل شركة شحن (DHL مثلاً):

| مكوّن الشحن | المقابل في SPI |
|-------------|----------------|
| شركة الشحن نفسها (DHL) | `spi_controller` — بتتحكم في الوسيلة الفعلية |
| الشاحنة/الطيارة | الـ hardware: SCK, MOSI, MISO |
| الـ waybill (بيان الشحن) | `spi_message` — atomic transaction |
| كل صندوق جوا الشحنة | `spi_transfer` — read/write buffer pair |
| العنوان على الصندوق | الـ chip select — بيحدد مين الـ target |
| المرسَل إليه | `spi_device` — الـ peripheral chip |
| محتوى الصندوق | بيانات الـ tx_buf / rx_buf |
| شركة الـ e-commerce اللي بتبعت الطرد | `spi_driver` (protocol driver) |

التشبيه أعمق من كده:
- زي ما DHL مش محتاجة تعرف جوا الصندوق إيه — الـ controller مش محتاج يعرف protocol الـ flash أو الـ codec.
- زي ما شركة الـ e-commerce مش محتاجة تعرف هل الشحن بطيارة أو شاحنة — الـ protocol driver مش محتاج يعرف هل الـ controller بيستخدم DMA أو PIO.
- لو DHL بدلها FedEx، الطرد يوصل نفس الطريقة — لو استبدلت الـ SoC بـ SoC تاني بيبقى نفس الـ protocol driver شغّال.
- الـ "قائمة الانتظار في المستودع" = الـ message queue جوا الـ `spi_controller`.

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Consumer Subsystems                            │
│    MTD (flash)    Input (touch)    ALSA (codec)    Net (eth)        │
└────────────┬────────────┬──────────────┬──────────────┬────────────┘
             │            │              │              │
┌────────────▼────────────▼──────────────▼──────────────▼────────────┐
│                   SPI Protocol Drivers (spi_driver)                 │
│   m25p80 (flash)   ads7846 (touch)   wm8731 (codec)  enc28j60      │
│                  each calls spi_async() / spi_sync()                │
└────────────────────────────────┬────────────────────────────────────┘
                                 │  spi_message  (list of spi_transfers)
┌────────────────────────────────▼────────────────────────────────────┐
│                        SPI Core (drivers/spi/spi.c)                 │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  spi_bus_type  ──  device/driver matching (modalias)        │    │
│  │  message queue  ── kthread pump (pump_messages kworker)     │    │
│  │  DMA mapping   ──  core handles map/unmap if can_dma()      │    │
│  │  spi_board_info ── static device registration               │    │
│  │  Device Tree   ──  of_register_spi_devices()                │    │
│  └─────────────────────────────────────────────────────────────┘    │
└────────────────────────────────┬────────────────────────────────────┘
                                 │  transfer_one_message() / transfer_one()
┌────────────────────────────────▼────────────────────────────────────┐
│                  SPI Controller Drivers (spi_controller)            │
│                                                                      │
│   spi-bcm2835.c    spi-imx.c    spi-rockchip.c    spi-bitbang.c    │
│   (Raspberry Pi)   (i.MX)       (RK3399)          (GPIO bitbang)    │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│                        Physical Hardware                             │
│                                                                      │
│   SCK ──────────────────────────────────────────────────────────    │
│   MOSI ─────────────────────────────────────────────────────────    │
│   MISO ─────────────────────────────────────────────────────────    │
│   nCS0 ─── Flash    nCS1 ─── Touch    nCS2 ─── Codec               │
└─────────────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction: الـ Message Queue Model

الفكرة المحورية في الـ SPI subsystem هي إن **التواصل مع أي SPI device بيتم عبر قائمة انتظار من الـ atomic transactions**.

الـ abstraction بيتكوّن من 3 levels متداخلة:

```
spi_message          ← atomic transaction (الـ CS ظل active طول المدة)
  └── spi_transfer   ← single read/write buffer pair
  └── spi_transfer   ← (يمكن يبقى فيه أكتر من transfer في message واحدة)
  └── spi_transfer
```

ليه ده مهم؟ لأن بعض الـ devices بيحتاج تبعت command وتقرأ response في **نفس العملية الواحدة** مع الـ CS ما اتفكش — زي `spi_w8r16()` اللي بتبعت 8 bits وترجع 16 bits في نفس الـ message.

---

### الـ Key Structs وعلاقتها ببعض

```
spi_controller
│
├── bus_num                   ← رقم الـ bus (مثلاً: SPI2)
├── num_chipselect            ← عدد الـ CS lines
├── mode_bits                 ← الـ modes المدعومة (CPOL, CPHA, ...)
├── min/max_speed_hz          ← حدود السرعة
│
├── [function pointers - what the controller MUST implement]
│   ├── setup()               ← configure device clock/mode
│   ├── transfer_one_message()← process one full atomic message    ┐ متبادلان
│   └── transfer_one()        ← process one transfer at a time     ┘
│
├── [function pointers - optional helpers]
│   ├── prepare_transfer_hardware()  ← wake up hardware
│   ├── unprepare_transfer_hardware()← sleep hardware
│   ├── set_cs()              ← toggle chip select GPIO/register
│   ├── can_dma()             ← can this transfer use DMA?
│   └── prepare_message()     ← DMA mapping setup
│
└── [internal queue state - owned by core]
    ├── kworker               ← kernel thread for message pump
    ├── queue (list_head)     ← pending messages
    └── cur_msg               ← currently executing message


spi_device
│
├── controller  ──────────────────────► spi_controller
├── max_speed_hz              ← per-device speed limit
├── mode                      ← SPI_MODE_0..3 + flags
├── bits_per_word             ← word size (default 8)
├── chip_select[]             ← physical CS index(es)
├── cs_gpiod[]                ← optional GPIO CS descriptors
└── modalias                  ← driver name for binding


spi_message
│
├── spi  ─────────────────────────────► spi_device
├── transfers (list_head)     ← linked list of spi_transfer
├── complete()                ← callback when done
├── status                    ← 0 = success
└── actual_length             ← bytes actually transferred


spi_transfer
│
├── tx_buf                    ← data to write (DMA-safe, or NULL)
├── rx_buf                    ← data to read  (DMA-safe, or NULL)
├── len                       ← bytes in this transfer
├── speed_hz                  ← override speed for this transfer
├── bits_per_word             ← override word size
├── cs_change                 ← toggle CS after this transfer?
├── delay                     ← post-transfer delay
└── transfer_list  ──────────► linked into spi_message.transfers
```

---

### الـ Clock Modes (CPOL/CPHA) بالتفصيل

ده مصدر ارتباك دايم. الـ SPI مفيهوش standard رسمي، فكل chip بتوثق الـ timing بطريقة مختلفة.

**CPOL** = Clock Polarity: قيمة الـ SCK وهو idle
- `CPOL=0` → الـ clock بيبدأ LOW
- `CPOL=1` → الـ clock بيبدأ HIGH

**CPHA** = Clock Phase: الـ edge اللي بيتأخذ فيه sample
- `CPHA=0` → sample على الـ leading edge (الـ edge الأولانية)
- `CPHA=1` → sample على الـ trailing edge (الـ edge التانية)

| Mode | CPOL | CPHA | Leading Edge | Trailing Edge |
|------|------|------|-------------|---------------|
| 0    | 0    | 0    | Sample (Rising) | Shift (Falling) |
| 1    | 0    | 1    | Shift (Rising) | Sample (Falling) |
| 2    | 1    | 0    | Sample (Falling) | Shift (Rising) |
| 3    | 1    | 1    | Shift (Falling) | Sample (Rising) |

Mode 0 و Mode 3 هم الأكتر شيوعاً في الواقع العملي — لأن معظم الـ devices بتأخذ sample على الـ rising edge مهما كان الـ polarity.

---

### الـ Device Registration Flow

الـ SPI devices مش بيتعملوا detect تلقائياً (مش زي USB). لازم تخبر الـ kernel إنهم موجودين. في 3 طرق:

#### 1. Device Tree (الأشيع في الـ embedded)
```dts
&spi1 {
    flash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;                /* chip select 0 */
        spi-max-frequency = <50000000>;
    };

    touchscreen@1 {
        compatible = "ti,ads7846";
        reg = <1>;                /* chip select 1 */
        spi-max-frequency = <2000000>;
        interrupt-parent = <&gpio>;
        interrupts = <31 IRQ_TYPE_EDGE_FALLING>;
    };
};
```

#### 2. Static Board Info (القديم، لا يزال موجود)
```c
static struct spi_board_info my_board_spi[] __initdata = {
    {
        .modalias       = "ads7846",          /* driver name */
        .platform_data  = &ads_info,
        .mode           = SPI_MODE_0,
        .irq            = GPIO_IRQ(31),
        .max_speed_hz   = 2000000,
        .bus_num        = 1,
        .chip_select    = 1,
    },
};
spi_register_board_info(my_board_spi, ARRAY_SIZE(my_board_spi));
```

#### 3. Dynamic (لـ MMC/SD over SPI)
الـ devices بتتعمل dynamically لما يتعمل hotplug.

---

### الـ Message Queue: كيف الـ Core بيشتغل

ده من أهم الحاجات اللي الـ core بيعملها، وبيوفر فيها على كل controller driver إنه يعيد اختراع العجلة.

```
Protocol Driver
    │
    │  spi_async(spi, msg)
    ▼
SPI Core
    │
    │  adds msg to ctlr->queue
    │  wakes kworker thread
    ▼
kthread: __spi_pump_messages()
    │
    ├── picks next msg from queue
    ├── calls ctlr->prepare_transfer_hardware()  [optional]
    ├── calls ctlr->prepare_message()            [optional - DMA map]
    ├── calls ctlr->transfer_one_message()       [mandatory OR]
    │         OR
    │   iterates transfers → ctlr->transfer_one() per transfer
    │
    ├── controller calls spi_finalize_current_message()
    ├── msg->complete(msg->context) callback fires
    └── picks next msg from queue, or calls unprepare_transfer_hardware()
```

**ملاحظة مهمة**: الـ `transfer_one_message()` و `transfer_one()` متبادلان حصرياً. لو implement الاتنين، الـ core بيستخدم بس `transfer_one_message()`.

---

### الـ I/O API من ناحية الـ Protocol Driver

```c
/* Async — يرجع فوراً، الـ callback بيتشغل لما يخلص */
int spi_async(struct spi_device *spi, struct spi_message *message);

/* Sync — بتنام لحد ما العملية تخلص، must not be called from IRQ context */
int spi_sync(struct spi_device *spi, struct spi_message *message);

/* Convenience wrappers — بيبنوا message+transfer داخلياً */
int spi_write(struct spi_device *spi, const void *buf, size_t len);
int spi_read(struct spi_device *spi, void *buf, size_t len);
int spi_write_then_read(struct spi_device *spi,
                        const void *txbuf, unsigned n_tx,
                        void *rxbuf, unsigned n_rx);

/* Common RPC pattern: write 8-bit command, read 16-bit response */
u16 val = spi_w8r16(spi, MY_READ_CMD);
```

**مثال عملي — قراءة temperature sensor:**
```c
static int read_temperature(struct spi_device *spi, s16 *temp)
{
    struct spi_transfer xfers[2];
    struct spi_message msg;
    u8 cmd = 0x03;    /* read command */
    u8 data[2];
    int ret;

    memset(xfers, 0, sizeof(xfers));
    spi_message_init(&msg);

    /* Transfer 1: send command (CS stays active) */
    xfers[0].tx_buf = &cmd;
    xfers[0].len    = 1;
    spi_message_add_tail(&xfers[0], &msg);

    /* Transfer 2: clock in the response */
    xfers[1].rx_buf = data;
    xfers[1].len    = 2;
    spi_message_add_tail(&xfers[1], &msg);

    ret = spi_sync(spi, &msg);
    if (ret)
        return ret;

    *temp = (data[0] << 8) | data[1];
    return 0;
}
```

---

### ما الـ SPI Core بيملكه vs ما بيفوّضه للـ Driver

| الـ Core بيملكه | الـ Controller Driver بيفوّضه |
|----------------|-------------------------------|
| الـ message queue وترتيبها | الـ hardware register access |
| الـ kthread pump | DMA channel management |
| Device/driver matching عبر modalias | Chip select toggling |
| DMA mapping (لو `can_dma()` رجعت true) | Clock rate configuration |
| الـ bus locking للـ exclusive access | الـ actual bit shifting |
| الـ sysfs entries | Power management callbacks |
| الـ Device Tree parsing | Error recovery في الـ hardware |
| الـ statistics (spi_statistics) | الـ hardware-specific quirks |

---

### الـ spi_bus_type — الـ Driver Model Integration

الـ SPI subsystem بيسجّل نفسه كـ **bus type** في الـ Linux driver model. ده مهم جداً لأنه بيربط كل حاجة ببعض.

> **الـ Driver Model**: subsystem جوا الـ kernel بيوفر abstraction موحدة لكل buses (USB، I2C، SPI...) — كل bus بيسجل `bus_type`، كل device بيسجل `device`، وكل driver بيسجل `device_driver`. الـ core بيعمل الـ matching.

```c
extern const struct bus_type spi_bus_type;
```

لما يتسجل `spi_device` على الـ bus، الـ kernel بيدور على `spi_driver` ليه عبر الـ `modalias`. لما يلاقيه، بيشغّل `probe()`. ده بيعني:
- الـ hotplug بيشتغل أوتوماتيك.
- الـ modules بتتحمّل بالـ `MODULE_DEVICE_TABLE`.
- الـ power management بيتعامل مع كل device لوحده.

---

### SPI في الـ sysfs

```
/sys/bus/spi/
    devices/
        spi1.0  →  ../../../devices/platform/soc/spi1/spi1.0   (flash)
        spi1.1  →  ../../../devices/platform/soc/spi1/spi1.1   (touchscreen)
    drivers/
        m25p80/
        ads7846/

/sys/class/spi_master/
    spi1  →  ../../devices/platform/soc/spi1              (bus 1)

/sys/devices/platform/soc/spi1/
    spi1.0/
        modalias   →  "spi-nor"
        of_node    →  link to DT node
```

الاسم `spi1.0` = bus رقم 1، chip select رقم 0.

---

### Extensions: Multi-lane SPI (QSPI / Octal SPI)

الـ standard SPI فيه MOSI وMISO واحدة. لكن الـ modern flash chips بتدعم:
- **Dual SPI**: 2 data lines (MISO + MOSI كلهم output في نفس الوقت)
- **Quad SPI**: 4 data lines
- **Octal SPI**: 8 data lines

الـ framework بيدعم ده عبر `tx_nbits` و `rx_nbits` في الـ `spi_transfer`:

```c
#define SPI_NBITS_SINGLE   0x01  /* standard */
#define SPI_NBITS_DUAL     0x02  /* 2 lines */
#define SPI_NBITS_QUAD     0x04  /* 4 lines */
#define SPI_NBITS_OCTAL    0x08  /* 8 lines */
```

وعبر `spi_controller_mem_ops` للـ controllers اللي عندها native memory interface للـ flash.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options

#### SPI Mode Bits (من `include/uapi/linux/spi/spi.h`)

| Flag | Bit | المعنى |
|------|-----|--------|
| `SPI_CPHA` | 0 | Clock phase — sample on trailing edge |
| `SPI_CPOL` | 1 | Clock polarity — clock starts high |
| `SPI_CS_HIGH` | 2 | Chip select active high |
| `SPI_LSB_FIRST` | 3 | إرسال الـ LSB أولاً بدل الـ MSB |
| `SPI_3WIRE` | 4 | MOSI و MISO على نفس السلك |
| `SPI_LOOP` | 5 | Loopback mode للتست |
| `SPI_NO_CS` | 6 | جهاز واحد على الـ bus بدون chip select |
| `SPI_READY` | 7 | الـ slave يسحب المنطق للـ low للـ pause |
| `SPI_TX_DUAL` | 8 | إرسال على 2 wires |
| `SPI_TX_QUAD` | 9 | إرسال على 4 wires |
| `SPI_RX_DUAL` | 10 | استقبال على 2 wires |
| `SPI_RX_QUAD` | 11 | استقبال على 4 wires |
| `SPI_CS_WORD` | 12 | toggle الـ CS بعد كل word |
| `SPI_TX_OCTAL` | 13 | إرسال على 8 wires |
| `SPI_RX_OCTAL` | 14 | استقبال على 8 wires |
| `SPI_3WIRE_HIZ` | 15 | High-Z turnaround للـ 3-wire |
| `SPI_RX_CPHA_FLIP` | 16 | قلب الـ CPHA على الـ Rx transfers فقط |
| `SPI_MOSI_IDLE_LOW` | 17 | MOSI تبقى low لما مفيش data |
| `SPI_MOSI_IDLE_HIGH` | 18 | MOSI تبقى high لما مفيش data |

#### SPI Modes الأساسية

| Mode | CPOL | CPHA | الـ Clock يبدأ | السامبل على |
|------|------|------|----------------|------------|
| `SPI_MODE_0` | 0 | 0 | Low | Rising edge |
| `SPI_MODE_1` | 0 | 1 | Low | Falling edge |
| `SPI_MODE_2` | 1 | 0 | High | Falling edge |
| `SPI_MODE_3` | 1 | 1 | High | Rising edge |

#### SPI Controller Flags (من `include/linux/spi/spi.h`)

| Flag | المعنى |
|------|--------|
| `SPI_CONTROLLER_HALF_DUPLEX` | الـ controller مش بيعمل full duplex |
| `SPI_CONTROLLER_NO_RX` | مش بيقدر يقرأ |
| `SPI_CONTROLLER_NO_TX` | مش بيقدر يكتب |
| `SPI_CONTROLLER_MUST_RX` | لازم يكون فيه rx buffer دايماً |
| `SPI_CONTROLLER_MUST_TX` | لازم يكون فيه tx buffer دايماً |
| `SPI_CONTROLLER_GPIO_SS` | الـ CS عن طريق GPIO |
| `SPI_CONTROLLER_SUSPENDED` | الـ controller suspended حالياً |
| `SPI_CONTROLLER_MULTI_CS` | بيقدر يـ assert أكتر من CS في نفس الوقت |

#### SPI Transfer Width (nbits)

| Constant | القيمة | المعنى |
|----------|--------|--------|
| `SPI_NBITS_SINGLE` | 0x01 | 1-bit transfer (الـ default) |
| `SPI_NBITS_DUAL` | 0x02 | 2-bit (DSPI) |
| `SPI_NBITS_QUAD` | 0x04 | 4-bit (QSPI) |
| `SPI_NBITS_OCTAL` | 0x08 | 8-bit (OSPI) |

#### SPI Transfer Error Flags

| Flag | المعنى |
|------|--------|
| `SPI_TRANS_FAIL_NO_START` | فشل في بداية الـ transfer |
| `SPI_TRANS_FAIL_IO` | I/O error أثناء الـ transfer |

#### SPI Multi-Lane Modes

| Constant | القيمة | المعنى |
|----------|--------|--------|
| `SPI_MULTI_LANE_MODE_SINGLE` | 0 | lane واحدة فقط |
| `SPI_MULTI_LANE_MODE_STRIPE` | 1 | كل data word على lane منفصلة |
| `SPI_MULTI_LANE_MODE_MIRROR` | 2 | نفس الـ word على كل الـ lanes |

#### SPI Delay Units

| Constant | المعنى |
|----------|--------|
| `SPI_DELAY_UNIT_USECS` (0) | الـ delay بالـ microseconds |
| `SPI_DELAY_UNIT_NSECS` (1) | الـ delay بالـ nanoseconds |
| `SPI_DELAY_UNIT_SCK` (2) | الـ delay بعدد clock cycles |

---

### الـ Structs المهمة

#### 1. `struct spi_device`

**الغرض:** الواجهة بين الـ controller driver والـ protocol driver — بيمثل جهاز SPI واحد على الـ bus.

```c
struct spi_device {
    struct device        dev;           // kernel device model
    struct spi_controller *controller;  // pointer للـ controller اللي عليه
    u32                  max_speed_hz;  // أقصى سرعة clock مسموحة
    u8                   bits_per_word; // حجم الـ word (default 8)
    u32                  mode;          // SPI_MODE_x + flags زي SPI_CS_HIGH
    int                  irq;           // رقم الـ interrupt (أو سالب)
    void                 *controller_state; // private state للـ controller
    void                 *controller_data;  // board-specific hints
    char                 modalias[SPI_NAME_SIZE]; // اسم الـ driver
    struct spi_statistics __percpu *pcpu_statistics;
    struct spi_delay     word_delay;    // delay بين الـ words
    struct spi_delay     cs_setup;      // delay بعد assert الـ CS
    struct spi_delay     cs_hold;       // delay قبل deassert الـ CS
    struct spi_delay     cs_inactive;   // delay بعد deassert الـ CS
    u8                   chip_select[SPI_DEVICE_CS_CNT_MAX]; // CS lines (max 4)
    u8                   num_chipselect;
    u32                  cs_index_mask; // الـ CS اللي الـ driver يستخدمها
    struct gpio_desc     *cs_gpiod[SPI_DEVICE_CS_CNT_MAX]; // GPIO CS
    u8                   tx_lane_map[8]; // mapping للـ tx lanes
    u8                   rx_lane_map[8]; // mapping للـ rx lanes
};
```

**الفيلد الأهم:** الـ `mode` بيحمل كل الـ SPI flags — الـ bits من 0 لـ 28 هي user flags، والـ bits من 29 لـ 31 هي kernel-only flags زي `SPI_NO_TX`, `SPI_NO_RX`, `SPI_TPM_HW_FLOW`.

---

#### 2. `struct spi_controller`

**الغرض:** بيمثل الـ SPI host/target controller نفسه — الـ hardware اللي بيتكلم مع الـ devices.

```c
struct spi_controller {
    struct device       dev;
    struct list_head    list;           // linked list لكل الـ controllers
    s16                 bus_num;        // رقم الـ bus (SPI0, SPI1, ...)
    u16                 num_chipselect; // عدد الـ CS lines
    u16                 num_data_lanes; // عدد الـ data lanes (default 1)
    u32                 mode_bits;      // الـ flags اللي الـ HW بيفهمها
    u32                 bits_per_word_mask; // الـ BPW القيم المدعومة
    u32                 min_speed_hz;
    u32                 max_speed_hz;
    u16                 flags;          // SPI_CONTROLLER_* flags

    /* Locking */
    struct mutex        io_mutex;       // للـ physical bus access
    struct mutex        add_lock;       // لمنع إضافة نفس الـ CS مرتين
    spinlock_t          bus_lock_spinlock;
    struct mutex        bus_lock_mutex;
    bool                bus_lock_flag;

    /* Callbacks إجبارية أو اختيارية */
    int  (*setup)(struct spi_device *spi);
    int  (*transfer)(struct spi_device *, struct spi_message *); // deprecated
    void (*cleanup)(struct spi_device *spi);
    bool (*can_dma)(struct spi_controller *, struct spi_device *, struct spi_transfer *);

    /* Queue-based transfer */
    bool                queued;
    struct kthread_worker *kworker;     // kernel thread للـ message pump
    struct kthread_work pump_messages;
    spinlock_t          queue_lock;
    struct list_head    queue;          // الـ message queue
    struct spi_message  *cur_msg;       // الـ message الحالي

    /* Hooks للـ queued controllers */
    int  (*prepare_transfer_hardware)(struct spi_controller *);
    int  (*transfer_one_message)(struct spi_controller *, struct spi_message *);
    int  (*unprepare_transfer_hardware)(struct spi_controller *);
    int  (*prepare_message)(struct spi_controller *, struct spi_message *);
    void (*set_cs)(struct spi_device *, bool enable);
    int  (*transfer_one)(struct spi_controller *, struct spi_device *, struct spi_transfer *);
    void (*handle_err)(struct spi_controller *, struct spi_message *);

    /* DMA */
    struct dma_chan     *dma_tx;
    struct dma_chan     *dma_rx;

    /* Statistics */
    struct spi_statistics __percpu *pcpu_statistics;
};
```

---

#### 3. `struct spi_transfer`

**الغرض:** بيمثل عملية read/write واحدة — الـ building block الأصغر في أي SPI transaction.

```c
struct spi_transfer {
    const void  *tx_buf;        // بيانات للإرسال (DMA-safe) أو NULL
    void        *rx_buf;        // buffer للاستقبال (DMA-safe) أو NULL
    unsigned    len;            // حجم الـ buffer بالـ bytes

    u16         error;          // SPI_TRANS_FAIL_* flags

    /* DMA scatterlists (يملأها الـ core) */
    struct sg_table tx_sg;
    struct sg_table rx_sg;
    dma_addr_t  tx_dma;
    dma_addr_t  rx_dma;

    /* Control bits */
    unsigned    dummy_data:1;   // transfer وهمية (للـ timing)
    unsigned    cs_off:1;       // نفذ الـ transfer والـ CS مـ activated
    unsigned    cs_change:1;    // toggle الـ CS بعد الـ transfer
    unsigned    tx_nbits:4;     // عدد الـ wires للـ TX
    unsigned    rx_nbits:4;     // عدد الـ wires للـ RX
    unsigned    multi_lane_mode:2; // SPI_MULTI_LANE_MODE_*
    bool        dtr_mode;       // Double Transfer Rate

    u8          bits_per_word;  // override الـ device default
    u32         speed_hz;       // override السرعة لهذا الـ transfer
    u32         effective_speed_hz; // السرعة الفعلية (يملأها الـ driver)

    /* Delays */
    struct spi_delay delay;           // delay بعد الـ transfer
    struct spi_delay cs_change_delay; // delay بين cs_change cycles
    struct spi_delay word_delay;      // delay بين الـ words

    /* PTP timestamping */
    struct ptp_system_timestamp *ptp_sts;
    unsigned int ptp_sts_word_pre;
    unsigned int ptp_sts_word_post;

    struct list_head transfer_list; // ربط داخل spi_message
};
```

---

#### 4. `struct spi_message`

**الغرض:** تسلسل atomic من الـ transfers — الـ SPI bus مش بيتقسم لأي message تانية طول ما المسج ده شغال.

```c
struct spi_message {
    struct list_head transfers;   // list من spi_transfer
    struct spi_device *spi;       // الجهاز المستهدف

    bool    pre_optimized;  // الـ peripheral driver عمل optimize مسبقاً
    bool    optimized;      // الـ core عمل __spi_optimize_message
    bool    prepared;       // spi_prepare_message اتنادى

    int     status;         // 0 = success, negative = error
    void    (*complete)(void *context); // callback لما الـ message يخلص
    void    *context;       // argument للـ complete()
    unsigned frame_length;  // إجمالي الـ bytes في المسج
    unsigned actual_length; // الـ bytes اللي اتنقلت فعلاً

    struct list_head queue;   // للـ controller driver internal use
    void    *state;           // state للـ controller driver
    void    *opt_state;       // state للـ optimize pass

    struct spi_offload *offload;    // offload instance (اختياري)
    struct list_head   resources;   // spi_res list للـ cleanup
};
```

---

#### 5. `struct spi_driver`

**الغرض:** الـ protocol driver اللي بيتكلم مع device معين — زي driver الـ Flash أو الـ Touchscreen.

```c
struct spi_driver {
    const struct spi_device_id *id_table; // جدول الـ devices المدعومة
    int  (*probe)(struct spi_device *spi);
    void (*remove)(struct spi_device *spi);
    void (*shutdown)(struct spi_device *spi);
    struct device_driver driver; // الـ base driver struct
};
```

---

#### 6. `struct spi_board_info`

**الغرض:** معلومات static عن الـ SPI device في الـ board — بتتحدد في init code أو Device Tree قبل ما الـ driver يتحمل.

```c
struct spi_board_info {
    char     modalias[SPI_NAME_SIZE]; // اسم الـ driver المطلوب
    const void *platform_data;        // بيانات خاصة بالـ device
    const struct software_node *swnode;
    void     *controller_data;        // hints للـ controller (DMA tuning إلخ)
    int      irq;
    u32      max_speed_hz;
    u16      bus_num;                 // رقم الـ SPI bus
    u16      chip_select;
    u32      mode;                    // SPI_MODE_x
};
```

---

#### 7. `struct spi_delay`

**الغرض:** تعريف delay بوحدة محددة — بيُستخدم في أكتر من مكان داخل spi_device و spi_transfer.

```c
struct spi_delay {
    u16 value; // القيمة العددية
    u8  unit;  // SPI_DELAY_UNIT_USECS / NSECS / SCK
};
```

---

#### 8. `struct spi_statistics`

**الغرض:** per-CPU counters للـ monitoring والـ debugging — موجود في كل من `spi_device` و `spi_controller`.

| Field | المعنى |
|-------|--------|
| `messages` | عدد الـ spi_messages |
| `transfers` | عدد الـ spi_transfers |
| `errors` | عدد الـ errors |
| `timedout` | عدد الـ timeouts |
| `spi_sync` | كام مرة استُخدم spi_sync |
| `spi_async` | كام مرة استُخدم spi_async |
| `bytes_tx` | إجمالي الـ bytes المرسلة |
| `bytes_rx` | إجمالي الـ bytes المستقبلة |
| `transfer_bytes_histo` | histogram لأحجام الـ transfers |

---

#### 9. `struct spi_res`

**الغرض:** resource management أثناء processing الـ spi_message — مستوحى من devres لكنه scoped على الـ message lifetime.

```c
struct spi_res {
    struct list_head entry;
    spi_res_release_t release; // callback وقت الـ cleanup
    unsigned long long data[]; // extra data مضمون alignment
};
```

---

### مخططات العلاقات بين الـ Structs

```
  Board Init Code
       |
       | spi_register_board_info()
       v
  spi_board_info[]
       |
       | (عند register الـ controller)
       v
  spi_device ──────────────► spi_controller
  ┌──────────────┐           ┌───────────────────────────┐
  │ dev          │           │ dev                       │
  │ controller ──┼──────────►│ list (global list)        │
  │ max_speed_hz │           │ bus_num                   │
  │ mode         │           │ queue ──► spi_message*    │
  │ chip_select[]│           │ cur_msg ──► spi_message*  │
  │ cs_gpiod[]   │           │ kworker                   │
  │ pcpu_stats   │           │ dma_tx / dma_rx           │
  └──────────────┘           │ ops: setup/transfer_one/  │
         ▲                   │       transfer_one_message │
         │                   └───────────────────────────┘
         │
  spi_driver
  ┌──────────────┐
  │ driver       │◄── kernel driver model
  │ probe()      │─── يستدعي spi_set_drvdata()
  │ remove()     │
  │ id_table     │
  └──────────────┘

  spi_message
  ┌────────────────────────┐
  │ spi ──────────────────►│ spi_device
  │ transfers (list_head)  │
  │   └─► spi_transfer ──► spi_transfer ──► NULL
  │        ├── tx_buf      ├── tx_buf
  │        ├── rx_buf      ├── rx_buf
  │        ├── len         ├── len
  │        └── transfer_list (linked)
  │ complete() callback    │
  │ resources (spi_res*)   │
  └────────────────────────┘
```

---

### دورة الحياة — Lifecycle Diagrams

#### أ. دورة حياة الـ SPI Controller Driver

```
  1. CREATION
     platform_driver_probe()
          │
          ▼
     spi_alloc_host(dev, sizeof(private))
     ──► kzalloc(sizeof(spi_controller) + private_size)
     ──► device_initialize(&ctlr->dev)

  2. CONFIGURATION
     ctlr->bus_num = N
     ctlr->num_chipselect = M
     ctlr->mode_bits = SPI_CPOL | SPI_CPHA | ...
     ctlr->transfer_one = my_hw_transfer
     ctlr->setup = my_hw_setup
          │
          ▼
  3. REGISTRATION
     spi_register_controller(ctlr)
     ──► alloc bus_num إذا كان سالب
     ──► device_add(&ctlr->dev)
     ──► of_register_spi_devices() أو spi_register_board_info devices
     ──► spi_new_device() لكل device في الجدول
          │
          ▼
  4. OPERATION (see transfer flow below)
          │
          ▼
  5. TEARDOWN
     spi_unregister_controller(ctlr)
     ──► device_del() لكل spi_device child
     ──► device_unregister(&ctlr->dev)
     ──► spi_controller_put() → kfree عند refcount = 0
```

#### ب. دورة حياة الـ SPI Protocol Driver

```
  1. REGISTRATION
     module_spi_driver(my_driver)
     ──► spi_register_driver()
     ──► driver_register()

  2. PROBE (kernel يربط الـ driver بالـ device)
     my_probe(spi_device *spi)
     ──► spi_get_device_match_data()
     ──► spi_setup(spi)          ← ضبط السرعة والـ mode
     ──► spi_set_drvdata(spi, priv)
     ──► بدء الـ I/O

  3. I/O (see message flow below)

  4. REMOVE
     my_remove(spi_device *spi)
     ──► إيقاف كل الـ I/O pending
     ──► تحرير الـ resources
```

#### ج. دورة حياة الـ spi_message

```
  PROTOCOL DRIVER:
  ┌────────────────────────────────────────┐
  │ 1. spi_message_init(&msg)              │
  │    أو spi_message_alloc(n, GFP_KERNEL) │
  │                                         │
  │ 2. spi_message_add_tail(&xfer, &msg)   │ ← ممكن أكتر من transfer
  │                                         │
  │ 3. msg.complete = my_callback          │
  │    msg.context = private_data          │
  │                                         │
  │ 4. spi_async(spi, &msg)               │ ← أو spi_sync()
  └────────────────────────────────────────┘
                    │
                    ▼
  SPI CORE:
  ┌────────────────────────────────────────┐
  │ 5. validate message                    │
  │ 6. __spi_optimize_message()            │
  │ 7. list_add_tail(&msg->queue,          │
  │                  &ctlr->queue)         │
  │ 8. kthread_queue_work(kworker, ...)    │
  └────────────────────────────────────────┘
                    │
                    ▼
  CONTROLLER DRIVER (message pump thread):
  ┌────────────────────────────────────────┐
  │ 9. prepare_transfer_hardware()         │
  │ 10. prepare_message()                  │
  │ 11. transfer_one_message()             │
  │     ──► set_cs(spi, true)             │
  │     ──► for each transfer:            │
  │         ─► can_dma? map buffers       │
  │         ─► transfer_one()             │
  │         ─► spi_finalize_current_      │
  │              transfer()              │
  │     ──► set_cs(spi, false)            │
  │ 12. spi_finalize_current_message()    │
  │ 13. unprepare_transfer_hardware()     │
  └────────────────────────────────────────┘
                    │
                    ▼
  PROTOCOL DRIVER:
  ┌────────────────────────────────────────┐
  │ 14. msg.complete(msg.context) called  │
  │ 15. check msg.status                  │
  │ 16. spi_message_free() إذا heap-alloc │
  └────────────────────────────────────────┘
```

---

### Call Flow Diagrams

#### الـ spi_async() Flow

```
protocol_driver calls spi_async(spi, msg)
  │
  ├─► validate: spi != NULL, msg->complete != NULL
  ├─► __spi_validate(spi, msg)
  │     ├─► check mode_bits compatibility
  │     ├─► check bits_per_word support
  │     └─► check speed_hz limits
  │
  ├─► ctlr->cur_msg = NULL? (queue empty fast path)
  │     └─► YES: spi_sync_immediate path (no queue)
  │
  └─► __spi_queued_transfer(spi, msg, need_pump)
        ├─► spin_lock_irqsave(&ctlr->queue_lock)
        ├─► list_add_tail(&msg->queue, &ctlr->queue)
        ├─► spin_unlock_irqrestore(&ctlr->queue_lock)
        └─► kthread_queue_work(ctlr->kworker, &ctlr->pump_messages)
              │
              ▼
        spi_pump_messages() [kernel thread]
              ├─► spi_get_next_queued_message()
              ├─► ctlr->prepare_transfer_hardware()
              ├─► __spi_pump_transfer_message()
              │     ├─► ctlr->prepare_message()
              │     ├─► spi_map_msg() [DMA mapping]
              │     └─► ctlr->transfer_one_message()
              │           ├─► ctlr->set_cs(spi, true)
              │           ├─► spi_transfer_one_message_noirq()
              │           │     └─► for each xfer:
              │           │           ctlr->transfer_one(ctlr, spi, xfer)
              │           │             → HW register write
              │           │           wait xfer_completion
              │           └─► ctlr->set_cs(spi, false)
              └─► spi_finalize_current_message()
                    ├─► spi_unmap_msg() [DMA unmap]
                    ├─► ctlr->unprepare_message()
                    └─► msg->complete(msg->context) ◄── الـ callback
```

#### الـ spi_sync() Fast Path

```
protocol_driver calls spi_sync(spi, msg)
  │
  ├─► init completion: msg->complete = spi_complete
  ├─► spi_async(spi, msg)
  │     └─► [إذا queue_empty == true]
  │           ─► __spi_sync() يشتغل في نفس الـ context
  │                 ─► ctlr->transfer_one_message() directly
  │                 ─► بدون kthread ← لأنه أسرع
  │
  └─► wait_for_completion(&msg->ctx.done)
        │
        └─► return msg->status
```

#### Controller Setup Flow

```
protocol_driver calls spi_setup(spi)
  │
  ├─► validate: no pending transfers
  ├─► ctlr->setup(spi)
  │     ├─► configure clock divider registers
  │     ├─► configure CPOL/CPHA bits
  │     ├─► configure word size
  │     └─► configure CS timing
  └─► return 0 or -errno
```

---

### استراتيجية الـ Locking

الـ SPI subsystem بيستخدم عدة locks لحماية resources مختلفة:

#### جدول الـ Locks

| Lock | النوع | بيحمي إيه | من بيمسكه |
|------|-------|-----------|-----------|
| `ctlr->queue_lock` | spinlock | الـ `ctlr->queue` list + `ctlr->running/busy` | spi_async, pump_messages |
| `ctlr->io_mutex` | mutex | الـ physical bus access (منع تداخل الـ messages) | spi_sync, pump loop |
| `ctlr->bus_lock_mutex` | mutex | exclusive bus locking للـ multi-message transactions | spi_bus_lock/unlock |
| `ctlr->bus_lock_spinlock` | spinlock | الـ `bus_lock_flag` | spi_async fast check |
| `ctlr->add_lock` | mutex | منع إضافة device على نفس الـ CS مرتين | spi_add_device |
| `spi_statistics.syncp` | seqcount | الـ per-CPU stats (32-bit systems) | SPI_STATISTICS_* macros |

#### Lock Ordering (الترتيب الصحيح لمنع الـ deadlock)

```
  bus_lock_mutex
       │
       └─► io_mutex
               │
               └─► queue_lock (spinlock — لا نفتح mutex جوّاه)
```

#### Bus Locking Pattern (للـ multi-message atomic transactions)

```c
/* لو الـ protocol driver محتاج يبعت أكتر من message بشكل atomic */
spi_bus_lock(ctlr);            // يمسك bus_lock_mutex
    spi_sync_locked(spi, &msg1);  // يستخدم io_mutex داخلياً
    spi_sync_locked(spi, &msg2);
spi_bus_unlock(ctlr);          // يرخي bus_lock_mutex
```

#### Queue Lock Usage Pattern

```c
/* إضافة message للـ queue */
spin_lock_irqsave(&ctlr->queue_lock, flags);
list_add_tail(&msg->queue, &ctlr->queue);
spin_unlock_irqrestore(&ctlr->queue_lock, flags);

/* الـ pump thread يجيب message */
spin_lock_irqsave(&ctlr->queue_lock, flags);
msg = list_first_entry(&ctlr->queue, struct spi_message, queue);
list_del_init(&msg->queue);
ctlr->cur_msg = msg;
spin_unlock_irqrestore(&ctlr->queue_lock, flags);
/* بعد كده يشتغل بدون lock */
```

#### الـ Per-CPU Statistics

الـ `spi_statistics` بيستخدم `u64_stats_sync` مع `get_cpu()/put_cpu()` عشان يضمن الـ atomicity على الـ 32-bit systems اللي فيها 64-bit counters محتاجة حماية إضافية، على الـ 64-bit systems الكتابة على 64-bit atomic بطبيعتها.

```c
/* الـ macro الصح للـ update */
SPI_STATISTICS_INCREMENT_FIELD(spi->pcpu_statistics, transfers);
/* اللي بيعمل: get_cpu → this_cpu_ptr → u64_stats_update_begin → inc → end → put_cpu */
```
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Registration & Lifecycle

| Function | Category | Context |
|---|---|---|
| `spi_register_driver()` | Protocol Driver Registration | sleepable |
| `spi_unregister_driver()` | Protocol Driver Cleanup | sleepable |
| `module_spi_driver()` | Macro — boilerplate init/exit | — |
| `spi_alloc_host()` | Controller Alloc | sleepable |
| `spi_alloc_target()` | Controller Alloc (slave) | sleepable |
| `devm_spi_alloc_host()` | devres Controller Alloc | sleepable |
| `spi_register_controller()` | Publish Controller | sleepable |
| `devm_spi_register_controller()` | devres Publish Controller | sleepable |
| `spi_unregister_controller()` | Teardown Controller | sleepable |
| `spi_register_board_info()` | Static Device Table | `__init` |
| `spi_new_device()` | Dynamic Device Creation | sleepable |
| `spi_alloc_device()` + `spi_add_device()` | Two-stage Device Registration | sleepable |
| `spi_unregister_device()` | Device Teardown | sleepable |
| `spi_setup()` | Configure Device Mode/Clock | sleepable |

#### I/O — Async & Sync

| Function | Blocking? | Context |
|---|---|---|
| `spi_async()` | No | any (IRQ/task) |
| `spi_sync()` | Yes | sleepable only |
| `spi_sync_locked()` | Yes | sleepable, bus locked |
| `spi_sync_transfer()` | Yes | sleepable |
| `spi_write()` | Yes | sleepable |
| `spi_read()` | Yes | sleepable |
| `spi_write_then_read()` | Yes | sleepable, small data only |
| `spi_w8r8()` | Yes | sleepable |
| `spi_w8r16()` | Yes | sleepable |
| `spi_w8r16be()` | Yes | sleepable, big-endian aware |

#### Message/Transfer Helpers

| Function | Purpose |
|---|---|
| `spi_message_init()` | zero + init lists |
| `spi_message_init_no_memset()` | init lists, skip memset |
| `spi_message_init_with_transfers()` | init + add array of transfers |
| `spi_message_add_tail()` | append spi_transfer to message |
| `spi_transfer_del()` | remove spi_transfer from message |
| `spi_message_alloc()` | alloc message + N transfers in one chunk |
| `spi_message_free()` | free that chunk |
| `spi_optimize_message()` | pre-optimize reusable message |
| `spi_unoptimize_message()` | release optimized state |

#### Controller Queue Callbacks (for driver authors)

| Callback | Who implements | Notes |
|---|---|---|
| `ctlr->setup()` | Controller driver | may sleep, avoid shared regs |
| `ctlr->cleanup()` | Controller driver | free per-device state |
| `ctlr->prepare_transfer_hardware()` | Controller driver | power-up HW |
| `ctlr->unprepare_transfer_hardware()` | Controller driver | power-down HW |
| `ctlr->transfer_one_message()` | Controller driver | mutually exclusive with transfer_one |
| `ctlr->transfer_one()` | Controller driver | mutually exclusive with transfer_one_message |
| `ctlr->set_cs()` | Controller driver | may be called from IRQ |
| `ctlr->handle_err()` | Controller driver | error handling hook |
| `ctlr->prepare_message()` | Controller driver | DMA mapping etc |
| `ctlr->unprepare_message()` | Controller driver | undo prepare |
| `spi_finalize_current_message()` | Controller driver | signal message done |
| `spi_finalize_current_transfer()` | Controller driver | signal transfer done |

#### Bus Locking & Utilities

| Function | Purpose |
|---|---|
| `spi_bus_lock()` / `spi_bus_unlock()` | exclusive bus access |
| `spi_controller_get_devdata()` / `spi_controller_set_devdata()` | controller private data |
| `spi_get_drvdata()` / `spi_set_drvdata()` | protocol driver data on spi_device |
| `spi_get_ctldata()` / `spi_set_ctldata()` | controller runtime state on spi_device |
| `spi_is_bpw_supported()` | check bits-per-word capability |
| `spi_bpw_to_bytes()` | convert bpw to in-memory byte size |
| `spi_max_transfer_size()` / `spi_max_message_size()` | HW size limits |
| `spi_controller_xfer_timeout()` | compute safe timeout value |
| `spi_split_transfers_maxsize()` / `spi_split_transfers_maxwords()` | split oversized transfers |

---

### Group 1: Protocol Driver Registration

الـ protocol driver بيتعامل مع SPI device من فوق — بيبعت messages من غير ما يلمس الـ hardware مباشرة. التسجيل بيربطه بالـ bus driver model.

---

#### `spi_register_driver()`

```c
/* Macro يلف __spi_register_driver مع THIS_MODULE */
#define spi_register_driver(driver) \
    __spi_register_driver(THIS_MODULE, driver)

int __spi_register_driver(struct module *owner, struct spi_driver *sdrv);
```

بيسجل الـ `spi_driver` مع الـ SPI bus type. الـ driver core بعد كده بيعمل match لأي `spi_device` عنده نفس الـ `modalias` وبيستدعي `probe()`. الـ macro موجودة علشان تمرر `THIS_MODULE` تلقائياً.

**Parameters:**
- `owner` — `THIS_MODULE`، بيمنع unload الـ module وهو شغال
- `sdrv` — pointer لـ `struct spi_driver` مليّانة callbacks

**Return:** `0` success، أو negative errno لو فشل.

**Key details:** بيعمل `driver_register()` داخلياً. بيستدعي `probe()` لكل `spi_device` متطابق موجود بالفعل على الـ bus. السياق: sleepable لأنه بيعمل memory allocation.

**Who calls it:** `module_init()` أو `module_spi_driver()` macro.

---

#### `spi_unregister_driver()`

```c
static inline void spi_unregister_driver(struct spi_driver *sdrv);
```

بتعكس الـ registration — بتستدعي `remove()` لكل device كانت مربوطة بالـ driver ده. بعدين بتشيل الـ driver من الـ bus.

**Parameters:**
- `sdrv` — نفس الـ pointer اللي اتسجل. لو NULL، بتتجاهل بأمان.

**Return:** void.

**Who calls it:** `module_exit()`.

---

#### `module_spi_driver()` — Macro

```c
#define module_spi_driver(__spi_driver) \
    module_driver(__spi_driver, spi_register_driver, spi_unregister_driver)
```

بيلغي الـ boilerplate — بدل ما تكتب `module_init` و`module_exit` يدوياً. كل module بيستخدمه مرة واحدة بس.

**مثال:**

```c
static struct spi_driver my_sensor_driver = {
    .driver = { .name = "my-sensor", .of_match_table = my_of_match },
    .probe  = my_sensor_probe,
    .remove = my_sensor_remove,
};
module_spi_driver(my_sensor_driver);
```

---

### Group 2: Controller Allocation & Registration

الـ controller driver هو اللي بيلمس الـ hardware registers، بيعمل DMA، وبيدير الـ queue. التسجيل بيعلن عن الـ bus للـ core.

---

#### `spi_alloc_host()` / `devm_spi_alloc_host()`

```c
static inline struct spi_controller *spi_alloc_host(struct device *dev,
                                                     unsigned int size);

static inline struct spi_controller *devm_spi_alloc_host(struct device *dev,
                                                          unsigned int size);
```

بيعمل allocate لـ `struct spi_controller` مع extra `size` bytes للـ driver-private data. الـ `devm_` نسخة بتتحرر تلقائياً لما الـ `dev` يتحرر.

**Parameters:**
- `dev` — الـ device اللي بيمتلك الـ controller (مثلاً `&pdev->dev`)
- `size` — حجم الـ driver-private struct اللي هيتلصق بعد الـ `spi_controller`

**Return:** pointer للـ `spi_controller` أو `NULL` لو فشل الـ alloc.

**Key details:** بيعمل `kzalloc` — الكل بيتصفّر. الـ driver-private data بتاخده بـ `spi_controller_get_devdata()`. الـ `devm_` نسخة بتضيف cleanup callback لـ devres. لا تستخدم `spi_alloc_host` مع `devm_spi_register_controller` وأنت مش محتاج تعمل cleanup يدوي.

**Who calls it:** `probe()` function للـ controller driver.

**Pseudocode:**

```
__spi_alloc_controller(dev, size, is_target=false):
    ctlr = kzalloc(sizeof(spi_controller) + size)
    ctlr->dev.parent = dev
    ctlr->target = false
    device_initialize(&ctlr->dev)
    return ctlr
```

---

#### `spi_alloc_target()` / `devm_spi_alloc_target()`

```c
static inline struct spi_controller *spi_alloc_target(struct device *dev,
                                                       unsigned int size);
```

نفس `spi_alloc_host` بالظبط، بس بيعمل `ctlr->target = true`. بيرجع `NULL` لو `CONFIG_SPI_SLAVE` مش enabled.

---

#### `spi_register_controller()` / `devm_spi_register_controller()`

```c
int spi_register_controller(struct spi_controller *ctlr);
int devm_spi_register_controller(struct device *dev,
                                  struct spi_controller *ctlr);
```

بيعلن عن الـ controller للـ SPI core وبيخلّيه visible في sysfs. بيعمل scan للـ board_info table علشان يخلق الـ `spi_device`s المُعلنة مسبقاً. الـ `devm_` نسخة بتعمل `spi_unregister_controller` تلقائياً على `dev_remove`.

**Parameters:**
- `ctlr` — الـ controller بعد ما تملا فيه كل الـ fields المطلوبة (bus_num، num_chipselect، callbacks...)
- `dev` — (الـ devm نسخة) الـ device المالك

**Return:** `0` لو نجح، negative errno لو فشل.

**Key details:**
- لازم تملا `ctlr->bus_num` قبل كده (أو سيب سالب للـ dynamic assignment).
- بيبدأ الـ message pump kthread.
- بيعمل match مع كل الـ board_info المسجلة ويخلق devices.
- لا يتعمل قبل ما تخلص من إعداد الـ `ctlr` callbacks.

**Who calls it:** `probe()` للـ controller driver، بعد `spi_alloc_host()`.

---

#### `spi_unregister_controller()`

```c
void spi_unregister_controller(struct spi_controller *ctlr);
```

بيعكس `spi_register_controller()`. بيوقف الـ message pump، بيحذف كل الـ `spi_device` children، وبيشيل الـ controller من sysfs.

**Who calls it:** `remove()` للـ controller driver. الـ `devm_` يستدعيها تلقائياً.

---

### Group 3: Static & Dynamic Device Registration

---

#### `spi_register_board_info()`

```c
int spi_register_board_info(struct spi_board_info const *info, unsigned n);
```

بيسجل جدول static من الـ SPI devices المعروفة على الـ board. بيتستدعى من `board_init()` أو `arch_initcall`. لما الـ controller بيتسجل لاحقاً، الـ core بيعمل match ويخلق الـ devices.

**Parameters:**
- `info` — array من `struct spi_board_info`
- `n` — عدد العناصر في الـ array (استخدم `ARRAY_SIZE`)

**Return:** `0` دايماً تقريباً (أو صفر لو `!CONFIG_SPI`).

**Key details:** بيعمل copy للـ info داخلياً — ممكن `__initdata`. اتستدعى مرة واحدة بس قبل الـ controller registration.

**مثال:**

```c
static struct spi_board_info board_spi_devs[] __initdata = {
    {
        .modalias     = "ads7846",     /* driver name */
        .max_speed_hz = 10000000,      /* 10 MHz */
        .bus_num      = 0,
        .chip_select  = 0,
        .mode         = SPI_MODE_0,
        .irq          = GPIO_IRQ(31),
    },
};

spi_register_board_info(board_spi_devs, ARRAY_SIZE(board_spi_devs));
```

---

#### `spi_new_device()`

```c
struct spi_device *spi_new_device(struct spi_controller *ctlr,
                                   struct spi_board_info *chip);
```

بيخلق `spi_device` جديد ديناميكياً (مش من جدول static). مفيد للـ hotplug أو للـ adapters (USB-to-SPI مثلاً).

**Parameters:**
- `ctlr` — الـ controller اللي هيتضاف عليه الـ device
- `chip` — الـ board_info اللي بتوصف الـ device

**Return:** pointer للـ `spi_device` الجديد، أو `NULL` لو فشل.

---

#### `spi_alloc_device()` + `spi_add_device()`

```c
struct spi_device *spi_alloc_device(struct spi_controller *ctlr);
int spi_add_device(struct spi_device *spi);
```

Two-stage registration: `spi_alloc_device` بيعمل allocate بس من غير ما يضيف للـ bus، وبعدين بتملا الـ fields يدوياً، وبعدين `spi_add_device` بيضيفه رسمياً. بيديك control أكتر من `spi_new_device`.

---

#### `spi_unregister_device()`

```c
void spi_unregister_device(struct spi_device *spi);
```

بيحذف الـ `spi_device` من الـ bus ويستدعي `remove()` للـ protocol driver المربوط. بيتستدعى عادةً تلقائياً من `spi_unregister_controller`.

---

### Group 4: Device Configuration

---

#### `spi_setup()`

```c
int spi_setup(struct spi_device *spi);
```

بيطبق إعدادات الـ `spi_device` (mode، bits_per_word، max_speed_hz) على الـ hardware عن طريق استدعاء `ctlr->setup()`. بيتستدعى عادةً مرة في `probe()` قبل أي I/O.

**Parameters:**
- `spi` — الـ device المراد إعداده. الـ fields المهمة: `spi->mode`، `spi->bits_per_word`، `spi->max_speed_hz`.

**Return:** `0` لو نجح، negative errno لو الـ controller رفض الـ mode.

**Key details:**
- ممكن يتستدعى أي وقت طالما مفيش message pending على الـ device ده.
- الـ controller driver لازم يفترض إن في transfers جارية لأجهزة تانية — ما يغيرش shared registers.
- `SPI_CS_HIGH`، `SPI_LSB_FIRST`، `SPI_3WIRE`، `SPI_MOSI_IDLE_HIGH` كلها بتتضبط من هنا.

**Who calls it:** protocol driver في `probe()`.

---

### Group 5: Message & Transfer Construction

الـ `spi_message` هو الوحدة الأساسية للـ I/O — sequence ذرية من transfers. الـ chipselect بيفضل active طول الـ message.

---

#### `spi_message_init()`

```c
static inline void spi_message_init(struct spi_message *m);
```

بيعمل `memset` صفر ثم يـ initialize الـ list heads. **لازم** تستدعيها قبل ما تستخدم أي message على stack أو embedded في struct.

---

#### `spi_message_init_no_memset()`

```c
static inline void spi_message_init_no_memset(struct spi_message *m);
```

بس بتعمل `INIT_LIST_HEAD` من غير memset. مفيدة لو الـ message اتخصصت بـ `kzalloc` بالفعل.

---

#### `spi_message_add_tail()`

```c
static inline void spi_message_add_tail(struct spi_transfer *t,
                                         struct spi_message *m);
```

بيضيف `spi_transfer` في نهاية قائمة الـ transfers في الـ message. التسلسل مهم — الـ transfers بتتنفذ بالترتيب.

---

#### `spi_message_init_with_transfers()`

```c
static inline void spi_message_init_with_transfers(struct spi_message *m,
                                                    struct spi_transfer *xfers,
                                                    unsigned int num_xfers);
```

Helper بيعمل `spi_message_init` ثم بيعمل loop ويضيف كل الـ transfers بـ `spi_message_add_tail`. للـ use cases البسيطة مع array ثابت.

---

#### `spi_message_alloc()` / `spi_message_free()`

```c
static inline struct spi_message *spi_message_alloc(unsigned ntrans, gfp_t flags);
static inline void spi_message_free(struct spi_message *m);
```

بيعمل allocate لـ message + N transfers في chunk واحد متراص في الذاكرة (zero-initialized). الـ `spi_message_free` مجرد `kfree` للـ chunk ده.

**مثال:**

```c
struct spi_message *msg = spi_message_alloc(2, GFP_KERNEL);
/* الـ transfers جاهزة ومضافة تلقائياً بعد الـ struct */
struct spi_transfer *t = (struct spi_transfer *)(msg + 1);
t[0].tx_buf = cmd_buf;  t[0].len = 1;
t[1].rx_buf = rx_buf;   t[1].len = 4;
spi_sync(spi, msg);
spi_message_free(msg);
```

---

#### `spi_optimize_message()` / `spi_unoptimize_message()`

```c
int spi_optimize_message(struct spi_device *spi, struct spi_message *msg);
void spi_unoptimize_message(struct spi_message *msg);
int devm_spi_optimize_message(struct device *dev, struct spi_device *spi,
                               struct spi_message *msg);
```

للـ messages اللي بتتبعت كتير (مثلاً في polling loop). بيعمل pre-allocate الـ DMA resources وbypasses الـ optimization step في كل إرسال، فكل استدعاء لـ `spi_sync` بيكون أسرع. `spi_unoptimize_message` بيحرر الـ resources دي. الـ `devm_` بتعمل `spi_unoptimize_message` تلقائياً لما الـ `dev` يتحرر.

**Key details:** الـ `msg->offload` لازم يتضبط قبل `spi_optimize_message` لو بتستخدم SPI offload.

---

### Group 6: I/O Functions — Async

---

#### `spi_async()`

```c
int spi_async(struct spi_device *spi, struct spi_message *message);
```

**الـ primitive الأساسية لكل الـ I/O** في SPI. بتضيف الـ message للـ queue وبتعود فوراً. الإتمام بيتبلّغ عبر `message->complete(message->context)`.

**Parameters:**
- `spi` — الـ target device
- `message` — الـ message المهيأ مع transfers وcompletion callback مضبوط

**Return:** `0` لو اتضافت للـ queue بنجاح، negative errno لو فيه مشكلة (مثلاً الـ controller suspended).

**Key details:**
- تقدر تستدعيها من أي context — IRQ handler، tasklet، workqueue، task.
- الـ memory للـ message والـ transfers والـ I/O buffers لازم تفضل valid لحد ما الـ completion callback يتستدعى — **ما تحطهاش على stack**.
- لو حصل error، الـ chip بيتـ deselect والـ message بيتلغى.
- الـ messages لنفس الـ `spi_device` بتتنفذ بـ FIFO order.

**Pseudocode:**

```
spi_async(spi, msg):
    validate msg fields
    msg->spi = spi
    spin_lock(ctlr->queue_lock)
    list_add_tail(&msg->queue, &ctlr->queue)
    spin_unlock(ctlr->queue_lock)
    if pump idle:
        kthread_queue_work(ctlr->kworker, &ctlr->pump_messages)
    return 0
```

---

### Group 7: I/O Functions — Synchronous

كلها built on top of `spi_async()`. كلها تحتاج sleepable context.

---

#### `spi_sync()`

```c
int spi_sync(struct spi_device *spi, struct spi_message *message);
```

بتبعت الـ message وبتـ block لحد ما يكتمل. داخلياً بتستخدم `spi_async` ثم `wait_for_completion`.

**Parameters:**
- `spi` — الـ target device
- `message` — الـ message (الـ complete callback بيتجاهله — الـ core بيحطه يدوياً)

**Return:** `message->status` — صفر لو نجح، negative errno لو فشل.

**Key details:**
- **Fast path**: لو الـ queue فاضي (`ctlr->queue_empty = true`)، بتنفذ الـ message في نفس الـ calling context من غير scheduling — أسرع بكتير.
- ما تستدعيهاش من IRQ context.
- لو الـ bus محتاج يتلوك، استخدم `spi_bus_lock` + `spi_sync_locked`.

---

#### `spi_sync_locked()`

```c
int spi_sync_locked(struct spi_device *spi, struct spi_message *message);
```

نفس `spi_sync` لكن بتفترض إن الـ caller حاجز الـ bus بـ `spi_bus_lock`. بتتستدعى بين `spi_bus_lock` و`spi_bus_unlock` لضمان إن مفيش driver تاني يتدخل.

---

#### `spi_bus_lock()` / `spi_bus_unlock()`

```c
int spi_bus_lock(struct spi_controller *ctlr);
int spi_bus_unlock(struct spi_controller *ctlr);
```

بيأخذ/بيحرر exclusive access على الـ SPI bus. ضروري للـ sequences اللي تحتاج multiple messages متتالية من غير أي interference. السياق: sleepable.

**مثال:**

```c
spi_bus_lock(spi->controller);
spi_sync_locked(spi, &msg1);  /* command */
spi_sync_locked(spi, &msg2);  /* data بدون أي device تاني يتدخل */
spi_bus_unlock(spi->controller);
```

---

#### `spi_sync_transfer()`

```c
static inline int spi_sync_transfer(struct spi_device *spi,
                                     struct spi_transfer *xfers,
                                     unsigned int num_xfers);
```

Helper بيعمل `spi_message_init_with_transfers` ثم `spi_sync`. للـ use cases البسيطة من غير ما تبني الـ message يدوياً.

---

#### `spi_write()`

```c
static inline int spi_write(struct spi_device *spi,
                              const void *buf, size_t len);
```

بتبعت `len` bytes من `buf` للـ device. بتبني `spi_transfer` بـ `tx_buf=buf` و`rx_buf=NULL`. **الـ buf لازم يكون DMA-safe** (heap أو free pages — مش stack).

---

#### `spi_read()`

```c
static inline int spi_read(struct spi_device *spi, void *buf, size_t len);
```

بتقرأ `len` bytes من الـ device في `buf`. بتبني `spi_transfer` بـ `rx_buf=buf` و`tx_buf=NULL` (الـ core بيبعت zeros).

---

#### `spi_write_then_read()`

```c
int spi_write_then_read(struct spi_device *spi,
                         const void *txbuf, unsigned n_tx,
                         void *rxbuf, unsigned n_rx);
```

**لأوامر الـ RPC النمطية**: بتبعت `n_tx` bytes، ثم بتقرأ `n_rx` bytes. بتعمل داخلياً copy للـ buffers لأنها بتستخدم kernel-internal DMA-safe buffer. **للبيانات الصغيرة فقط** (عشرات الـ bytes).

**Parameters:**
- `txbuf` — command bytes لإرسالها (ممكن stack)
- `n_tx` — عدد bytes الإرسال
- `rxbuf` — buffer لاستقبال الـ response (ممكن stack)
- `n_rx` — عدد bytes الاستقبال

**Return:** `0` لو نجح، negative errno لو فشل.

**Key details:** بيعمل mutex داخلي. الـ copy overhead موجود — ما تستخدمهاش لـ buffers كبيرة. مناسبة جداً لـ sensor register reads.

---

#### `spi_w8r8()` / `spi_w8r16()` / `spi_w8r16be()`

```c
static inline ssize_t spi_w8r8(struct spi_device *spi, u8 cmd);
static inline ssize_t spi_w8r16(struct spi_device *spi, u8 cmd);
static inline ssize_t spi_w8r16be(struct spi_device *spi, u8 cmd);
```

أبسط الـ helpers: بتبعت byte واحد (command)، وبتقرأ 8 أو 16 bit كـ response.

- **`spi_w8r8`**: بتبعت 1 byte، بتقرأ 1 byte → بترجع قيمة موجبة (الـ byte) أو errno سالب.
- **`spi_w8r16`**: بتبعت 1 byte، بتقرأ 2 bytes بـ wire order (big-endian محتمل).
- **`spi_w8r16be`**: زي `spi_w8r16` لكن بتعمل `be16_to_cpu()` تلقائياً.

**مثال:**

```c
/* قراءة register 0x10 من sensor */
int val = spi_w8r8(spi, 0x10 | 0x80); /* bit 7 = read flag */
if (val < 0)
    return val;
dev_dbg(&spi->dev, "reg=0x%02x\n", (u8)val);
```

---

### Group 8: Controller Driver Callbacks

الـ callbacks دي هي اللي الـ controller driver بيملاها في `struct spi_controller` قبل `spi_register_controller`.

---

#### `ctlr->setup()`

```c
int (*setup)(struct spi_device *spi);
```

بيضبط الـ hardware parameters للـ device: clock rate، SPI mode (CPOL/CPHA)، bits_per_word. الـ core بيستدعيه لما `spi_setup()` يتستدعى.

**Key details:**
- **تحذير مهم**: افترض دايماً إن الـ controller بيعمل transfer لـ device تاني حالياً. ما تغيرش أي shared register بطريقة تأثر على الـ transfers الجارية.
- الـ controller ممكن يحتفظ بـ per-device state في `spi->controller_state`.
- ممكن ينام (sleepable context).

---

#### `ctlr->cleanup()`

```c
void (*cleanup)(struct spi_device *spi);
```

بيحرر الـ per-device state اللي خزنه الـ controller في `spi->controller_state`. بيتستدعى لما الـ `spi_device` بيتحذف.

---

#### `ctlr->prepare_transfer_hardware()` / `ctlr->unprepare_transfer_hardware()`

```c
int (*prepare_transfer_hardware)(struct spi_controller *ctlr);
int (*unprepare_transfer_hardware)(struct spi_controller *ctlr);
```

- **`prepare`**: الـ core بيستدعيه قبل ما يبدأ يعالج الـ queue — فرصة لـ power-up الـ HW، enable clocks، إلخ.
- **`unprepare`**: الـ core بيستدعيه لما الـ queue تفضى — فرصة لـ power-down وتوفير الطاقة.

كلاهما ممكن ينام.

---

#### `ctlr->transfer_one_message()`

```c
int (*transfer_one_message)(struct spi_controller *ctlr,
                             struct spi_message *mesg);
```

الـ core بيديه message كاملة. الـ driver مسؤول عن تنفيذ كل الـ transfers فيها بالترتيب. لما يخلص **لازم** يستدعي `spi_finalize_current_message()`.

**Key details:**
- **مُحصر مع `transfer_one`**: لو الاثنين متضبطين، الـ core بيستخدم `transfer_one_message` بس ويتجاهل `transfer_one`.
- ممكن ينام.
- الـ chipselect بيفضل active طول الـ message إلا لو `cs_change` معمول تعديل.

---

#### `ctlr->transfer_one()`

```c
int (*transfer_one)(struct spi_controller *ctlr,
                    struct spi_device *spi,
                    struct spi_transfer *transfer);
```

أبسط من `transfer_one_message` — الـ core بيدير الـ message iteration والـ chipselect، والـ driver مسؤول بس عن تنفيذ transfer واحد.

**Return values:**
- `0` — التransfer اكتمل synchronously
- `1` — التransfer لسه جاري (async/DMA) — **لازم** يستدعي `spi_finalize_current_transfer()` لما يخلص
- negative errno — فشل

**لو فشل:**

```c
/* في interrupt/DMA completion handler */
xfer->error |= SPI_TRANS_FAIL_IO;
spi_finalize_current_transfer(ctlr);
```

---

#### `spi_finalize_current_message()` / `spi_finalize_current_transfer()`

```c
void spi_finalize_current_message(struct spi_controller *ctlr);
void spi_finalize_current_transfer(struct spi_controller *ctlr);
```

- **`spi_finalize_current_message`**: بيخبر الـ core إن الـ message الحالية اكتملت، فيبدأ يعالج التالية من الـ queue. **لازم** تستدعيها من `transfer_one_message`.
- **`spi_finalize_current_transfer`**: بيخبر الـ core إن الـ transfer الحالي اكتمل، فيكمل الـ message. **لازم** تستدعيها من `transfer_one` لو رجعت `1`.

**Who calls it:** controller driver من IRQ handler أو DMA completion callback — ممكن يتستدعى من atomic context.

---

#### `ctlr->set_cs()`

```c
void (*set_cs)(struct spi_device *spi, bool enable);
```

بيضبط logic level الـ chip select. الـ core بيستدعيه قبل وبعد كل message. `enable=true` يعني assert الـ CS (مع مراعاة `SPI_CS_HIGH` flag).

**Key details:** ممكن يتستدعى من IRQ context — **ما ينامش**.

---

#### `ctlr->prepare_message()` / `ctlr->unprepare_message()`

```c
int (*prepare_message)(struct spi_controller *ctlr,
                        struct spi_message *message);
int (*unprepare_message)(struct spi_controller *ctlr,
                          struct spi_message *message);
```

- **`prepare_message`**: بيتستدعى قبل `transfer_one_message` — فرصة لـ DMA mapping أو hardware configuration خاص بالـ message.
- **`unprepare_message`**: بيعكس الـ prepare بعد ما الـ message تكتمل.

كلاهما بيتستدعى من threaded context (ممكن ينام).

---

#### `ctlr->handle_err()`

```c
void (*handle_err)(struct spi_controller *ctlr,
                   struct spi_message *message);
```

بيتستدعى لو الـ core اكتشف error أثناء `transfer_one_message()`. بيدي الـ driver فرصة يعمل cleanup (reset HW مثلاً).

---

### Group 9: Controller Data Accessors

---

#### `spi_controller_get_devdata()` / `spi_controller_set_devdata()`

```c
static inline void *spi_controller_get_devdata(struct spi_controller *ctlr);
static inline void spi_controller_set_devdata(struct spi_controller *ctlr,
                                               void *data);
```

بيخزن/بيجيب الـ driver-private data pointer. مبنية على `dev_get_drvdata` / `dev_set_drvdata`. الـ data بتكون في الـ extra bytes اللي اتطلبت في `spi_alloc_host(dev, sizeof(*priv))`.

**مثال:**

```c
/* في probe() */
struct my_ctlr *priv = spi_controller_get_devdata(ctlr);
priv->base = devm_ioremap_resource(dev, res);
```

---

#### `spi_get_drvdata()` / `spi_set_drvdata()`

```c
static inline void spi_set_drvdata(struct spi_device *spi, void *data);
static inline void *spi_get_drvdata(const struct spi_device *spi);
```

الـ per-device driver data للـ protocol driver. بيتستدعى من `probe()` لحفظ الـ chip state.

---

#### `spi_get_ctldata()` / `spi_set_ctldata()`

```c
static inline void *spi_get_ctldata(const struct spi_device *spi);
static inline void spi_set_ctldata(struct spi_device *spi, void *state);
```

الـ controller driver بيستخدمهم لتخزين per-device runtime state (مثلاً DMA channel configuration، CS timing). مختلفة عن `drvdata` اللي للـ protocol driver.

---

#### `spi_get_chipselect()` / `spi_set_chipselect()`

```c
static inline u8 spi_get_chipselect(const struct spi_device *spi, u8 idx);
static inline void spi_set_chipselect(struct spi_device *spi, u8 idx,
                                       u8 chipselect);
```

بيجيب/بيضبط الـ physical chip select index في الـ `spi->chip_select[]` array. الـ `idx` هو الـ logical CS، والـ return هو الـ physical CS number.

---

#### `spi_is_csgpiod()`

```c
static inline bool spi_is_csgpiod(struct spi_device *spi);
```

بتتحقق إذا كان أي من الـ chip selects بيستخدم GPIO descriptor بدل native CS. مفيدة في الـ controller driver لمعرفة إذا كان محتاج يدير الـ CS بـ GPIO.

---

### Group 10: Utility & Capability Queries

---

#### `spi_is_bpw_supported()`

```c
static inline bool spi_is_bpw_supported(struct spi_device *spi, u32 bpw);
```

بتتحقق إذا كان الـ controller يدعم الـ bits-per-word المطلوبة. 8 bits دايماً supported. باقي القيم بتتفحص في `ctlr->bits_per_word_mask` عن طريق `SPI_BPW_MASK(bpw)`.

---

#### `spi_bpw_to_bytes()`

```c
static inline u32 spi_bpw_to_bytes(u32 bpw);
```

بتحوّل bits-per-word لحجم الـ word في الذاكرة (power-of-two bytes). مثلاً: 12-bit → 2 bytes، 20-bit → 4 bytes. مفيدة لحساب حجم الـ buffer.

| Input (bits) | Output (bytes) |
|---|---|
| 1–8 | 1 |
| 9–16 | 2 |
| 17–32 | 4 |
| 33–64 | 8 |

---

#### `spi_max_transfer_size()` / `spi_max_message_size()`

```c
static inline size_t spi_max_transfer_size(struct spi_device *spi);
static inline size_t spi_max_message_size(struct spi_device *spi);
```

بتجيب الـ hardware limits. لو الـ controller ما حددش `max_transfer_size`، الـ return هو `SIZE_MAX`. Protocol driver بيستخدمهم لتقسيم الـ buffers الكبيرة.

---

#### `spi_controller_xfer_timeout()`

```c
static inline unsigned int spi_controller_xfer_timeout(
        struct spi_controller *ctlr,
        struct spi_transfer *xfer);
```

بتحسب timeout مناسب بالـ milliseconds: `max(len * 8 * 2 / (speed_hz/1000), 500)`. بتضمن على الأقل 500ms لتجنب false positives على الأنظمة المحملة.

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

بيقسم الـ transfers اللي حجمها أكبر من الـ limit المسموح به. بيتستدعى عادةً من `prepare_message()` للـ controller driver. بيستخدم `spi_res` mechanism لعكس التعديل تلقائياً بعد اكتمال الـ message.

---

### Group 11: Delay Execution

---

#### `spi_delay_to_ns()`

```c
int spi_delay_to_ns(struct spi_delay *_delay, struct spi_transfer *xfer);
```

بيحول `struct spi_delay` (اللي ممكن تكون بـ microseconds أو nanoseconds أو SCK cycles) لنانوثانية. الـ `xfer` مطلوب لو الـ unit هو `SPI_DELAY_UNIT_SCK` علشان يعرف الـ clock speed.

**Return:** النانوثانية كـ int، أو negative errno.

---

#### `spi_delay_exec()`

```c
int spi_delay_exec(struct spi_delay *_delay, struct spi_transfer *xfer);
```

بينفذ الـ delay فعلياً (بينام بـ `ndelay`/`udelay`). بيتستدعى من الـ core بعد كل transfer لو فيه `delay` متضبط في الـ `spi_transfer`.

---

### Group 12: PTP Timestamping

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

للـ controller drivers اللي بتدعم `ptp_sts_supported = true`. بتاخد timestamps دقيقة حول اللحظة اللي بيتم فيها transfer الـ word المحدد. `progress` هو offset الـ word في الـ buffer. `irqs_off` بيخبر الـ core لو الـ IRQs كانت معطلة أثناء الـ timestamp.

---

### Group 13: PM Helpers للـ Controller

---

#### `spi_controller_suspend()` / `spi_controller_resume()`

```c
int spi_controller_suspend(struct spi_controller *ctlr);
int spi_controller_resume(struct spi_controller *ctlr);
```

- **`suspend`**: بيوقف الـ message pump kthread ويضبط `SPI_CONTROLLER_SUSPENDED` flag. لازم الـ controller driver يستدعيها في `pm_ops->suspend`.
- **`resume`**: بيشيل الـ flag ويعيد تشغيل الـ pump. لازم يستدعيها في `pm_ops->resume`.

---

### ملاحظات معمارية مهمة

```
Protocol Driver
     │  spi_async() / spi_sync()
     ▼
SPI Core (drivers/spi/spi.c)
     │  message queue (kworker pump)
     │  ctlr->prepare_transfer_hardware()
     │  ctlr->prepare_message()
     │  ctlr->transfer_one_message()  ←OR→  ctlr->transfer_one() [core iterates]
     │  spi_finalize_current_message()
     │  ctlr->unprepare_message()
     │  ctlr->unprepare_transfer_hardware()
     ▼
Controller Driver (HW registers / DMA)
     │  ctlr->set_cs()  [IRQ-safe]
     ▼
SPI Bus (SCLK / MOSI / MISO / nCS)
```

**قواعد DMA-safety الأساسية:**
- `tx_buf` و`rx_buf` في `spi_transfer` لازم يكونوا DMA-safe (heap أو page pool) — **مش stack، مش static**.
- الـ `spi_message` والـ `spi_transfer` structs نفسها ممكن يكونوا anywhere طالما مش جاري استخدامهم.
- استخدم `spi_write_then_read()` فقط للـ buffers الصغيرة — بتعمل copy داخلياً لـ DMA-safe region.
- بعد `spi_async()`، ما تلمسش الـ message أو الـ transfers لحد ما الـ completion callback يتستدعى.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs — المسارات والقراءة

الـ SPI subsystem مش بيسجّل entries كتير في debugfs بشكل افتراضي، لكن بعض الـ controllers بتضيف entries خاصة بيها.

```bash
# تأكد إن debugfs mounted
mount | grep debugfs
# لو مش موجود
mount -t debugfs none /sys/kernel/debug

# شوف كل حاجة SPI موجودة في debugfs
ls /sys/kernel/debug/spi*
ls /sys/kernel/debug/regmap/   # لو الـ controller بيستخدم regmap

# بعض الـ controllers زي spi-dw (DesignWare) بتضيف:
cat /sys/kernel/debug/spi0/registers   # register dump مباشر
```

**الـ `struct spi_statistics`** هي المصدر الرئيسي للمعلومات — بتتعرض عبر sysfs مش debugfs:

| Field | المعنى |
|---|---|
| `messages` | عدد الـ spi_messages اللي اتنفذت |
| `transfers` | عدد الـ spi_transfers |
| `errors` | عدد الأخطاء |
| `timedout` | timeout count |
| `bytes_tx` / `bytes_rx` | البايتات المرسلة/المستقبَلة |
| `transfers_split_maxsize` | transfers اتقسمت بسبب maxsize limit |

---

#### 2. sysfs — المسارات المهمة

```bash
# قائمة كل الـ SPI buses
ls /sys/class/spi_master/

# معلومات controller معين (مثلاً spi0)
ls /sys/class/spi_master/spi0/

# إحصائيات الـ controller نفسه
cat /sys/class/spi_master/spi0/statistics/messages
cat /sys/class/spi_master/spi0/statistics/errors
cat /sys/class/spi_master/spi0/statistics/timedout
cat /sys/class/spi_master/spi0/statistics/bytes
cat /sys/class/spi_master/spi0/statistics/bytes_rx
cat /sys/class/spi_master/spi0/statistics/bytes_tx
cat /sys/class/spi_master/spi0/statistics/transfers_split_maxsize

# قائمة الـ SPI devices على bus 0
ls /sys/bus/spi/devices/

# معلومات device معين (مثلاً spi0.0)
cat /sys/bus/spi/devices/spi0.0/modalias
cat /sys/bus/spi/devices/spi0.0/driver/module/version

# إحصائيات الـ device نفسه
cat /sys/bus/spi/devices/spi0.0/statistics/errors
cat /sys/bus/spi/devices/spi0.0/statistics/timedout

# الـ max_speed المتفق عليه
cat /sys/bus/spi/devices/spi0.0/max_speed_hz  # لو الـ driver بيعرضه

# الـ spi_slave support
cat /sys/class/spi_slave/spi0/slave   # اسم الـ target device

# تحقق من الـ driver المرتبط
ls -l /sys/bus/spi/devices/spi0.0/driver
```

---

#### 3. ftrace — الـ Tracepoints والـ Events

```bash
# شوف كل الـ SPI events المتاحة
ls /sys/kernel/tracing/events/spi/

# الـ events الأساسية في SPI subsystem:
# spi_controller_busy    - الـ controller بدأ شغل
# spi_controller_idle    - الـ controller خلص
# spi_message_start      - message بدأت
# spi_message_done       - message خلصت
# spi_transfer_start     - transfer بدأ
# spi_transfer_stop      - transfer خلص
# spi_set_cs             - chip select تغيّر

# فعّل كل الـ SPI events
echo 1 > /sys/kernel/tracing/events/spi/enable

# أو فعّل event واحد بس
echo 1 > /sys/kernel/tracing/events/spi/spi_message_done/enable
echo 1 > /sys/kernel/tracing/events/spi/spi_transfer_start/enable

# ابدأ التتبع
echo 1 > /sys/kernel/tracing/tracing_on

# اعمل العملية اللي بتحقق فيها
# مثلاً: اقرأ من SPI flash أو شغّل الـ driver

# اقرأ النتيجة
cat /sys/kernel/tracing/trace

# أو بشكل real-time
cat /sys/kernel/tracing/trace_pipe

# وقف التتبع
echo 0 > /sys/kernel/tracing/tracing_on
echo 0 > /sys/kernel/tracing/events/spi/enable
```

**مثال output من `trace_pipe`:**

```
      kworker/0:1-42    [000] ....  1234.567890: spi_message_start: spi0.0
      kworker/0:1-42    [000] ....  1234.567891: spi_transfer_start: spi0.0 len=4 tx=[0x9f 0x00 0x00 0x00] rx=[]
      kworker/0:1-42    [000] ....  1234.567910: spi_transfer_stop: spi0.0 len=4 tx=[] rx=[0xef 0x40 0x18 0x00]
      kworker/0:1-42    [000] ....  1234.567912: spi_message_done: spi0.0 status=0 frame=4 actual=4
```

**التفسير:** الـ `len=4` هو حجم الـ transfer، الـ `status=0` يعني نجاح، الـ `actual=4` هو الـ bytes اللي اتبعتت فعلاً.

---

#### 4. printk / Dynamic Debug

```bash
# فعّل الـ dynamic debug لكل الـ SPI core
echo 'file drivers/spi/spi.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل لـ driver معين (مثلاً spi-nor)
echo 'file drivers/mtd/spi-nor/core.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل لكل ملفات الـ SPI subsystem
echo 'file drivers/spi/* +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل مع stack trace لكل SPI message
echo 'file drivers/spi/spi.c +ps' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ dynamic debug entries الحالية
grep spi /sys/kernel/debug/dynamic_debug/control

# أو في الـ kernel cmdline لو محتاج من البداية
# dyndbg="file drivers/spi/spi.c +p"
```

في الـ driver نفسه، الـ `dev_dbg()` هو الأكثر استخداماً — بيحتاج `CONFIG_DYNAMIC_DEBUG=y` أو `pr_debug()` مع `-DDEBUG` في الـ Makefile.

---

#### 5. Kernel Config Options للـ Debugging

| Config | الغرض |
|---|---|
| `CONFIG_SPI_DEBUG` | يفعّل رسائل debug مفصّلة في SPI core |
| `CONFIG_DYNAMIC_DEBUG` | يتيح `dev_dbg()` / `pr_debug()` runtime |
| `CONFIG_SPI_LOOPBACK_TEST` | يختبر الـ SPI controller بـ loopback |
| `CONFIG_SPI_BITBANG` | يتيح software bitbang — مفيد للتشخيص |
| `CONFIG_DEBUG_SG` | يتحقق من صحة الـ scatter/gather buffers (مهم مع DMA) |
| `CONFIG_DMA_API_DEBUG` | يكتشف أخطاء الـ DMA mapping |
| `CONFIG_KASAN` | يكتشف memory corruption في الـ buffers |
| `CONFIG_LOCKDEP` | يكتشف deadlocks في الـ SPI locks |
| `CONFIG_FTRACE` | يتيح tracing infrastructure |
| `CONFIG_TRACING` | يتيح الـ SPI tracepoints |
| `CONFIG_SPI_SLAVE` | يتيح الـ SPI target mode |
| `CONFIG_SPI_MEM` | يتيح SPI memory layer (spi-mem) |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "CONFIG_SPI|CONFIG_DYNAMIC_DEBUG|CONFIG_DEBUG_SG"
# أو
cat /boot/config-$(uname -r) | grep -E "CONFIG_SPI"
```

---

#### 6. أدوات خاصة بالـ Subsystem

```bash
# spidev_test — أداة اختبار SPI من userspace (موجودة في kernel/tools/spi)
# بناء الأداة:
cd tools/spi && make

# اختبار loopback (MOSI → MISO مربوطين ببعض)
./spidev_test -D /dev/spidev0.0 -s 500000 -l -v

# إرسال bytes محددة
./spidev_test -D /dev/spidev0.0 -s 1000000 -p "\x9f\x00\x00\x00"

# قراءة JEDEC ID من SPI flash
./spidev_test -D /dev/spidev0.0 -s 1000000 -p "\x9f\x00\x00\x00" -v

# استخدام i2c-tools/spi-tools للفحص السريع
# (لو مثبّتة)
spitool -d /dev/spidev0.0 -c 0x9F -r 3

# فحص الـ MTD devices (SPI flash)
mtdinfo /dev/mtd0
cat /proc/mtd
```

---

#### 7. جدول رسائل الأخطاء الشائعة

| رسالة الـ Kernel Log | السبب | الحل |
|---|---|---|
| `spi_master spi0: failed to transfer one message` | فشل في transfer | شوف الـ statistics/errors وفعّل dynamic debug |
| `SPI transfer timed out` | الـ device مش بيستجيب أو clock مش وصله | تحقق من الـ wiring، الـ CS، والـ clock mode |
| `spi0.0: can't create new device` | الـ CS أو bus number غلط في DT/board info | صحّح `bus_num` و`chip_select` |
| `spi0.0: setup: unsupported mode bits` | الـ controller مش بيدعم الـ mode المطلوب | شوف `mode_bits` في الـ controller driver |
| `-EINVAL` من `spi_setup()` | bits_per_word أو speed خارج الحدود | تحقق من `min/max_speed_hz` و`bits_per_word_mask` |
| `DMA mapping failed` | buffer مش DMA-safe (stack أو static) | استخدم `kmalloc` أو `kzalloc` للـ buffer |
| `spi0.0: chipselect not activated` | مشكلة في GPIO أو pinmux | تحقق من الـ DT ومن الـ pinctrl |
| `spi0.0: transfer rejected, bus is busy` | محاولة transfer وهو في اللحظة دي occupied | استخدم `spi_bus_lock` بشكل صحيح |
| `spi0.0: message rejected: -ESHUTDOWN` | controller بيتوقف (suspend/remove) | تحقق من lifecycle management |
| `spi0.0: Failed to allocate DMA` | مفيش DMA channel متاح | شوف الـ DMA controller config وتوافق الـ channels |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في spi_transfer_one_message() — بعد كل transfer للتأكد من الحالة */
static int spi_transfer_one_message(struct spi_controller *ctlr,
                                    struct spi_message *msg)
{
    struct spi_transfer *xfer;
    int ret;

    list_for_each_entry(xfer, &msg->transfers, transfer_list) {
        /* تأكد إن الـ buffer DMA-safe */
        WARN_ON(xfer->tx_buf && object_is_on_stack(xfer->tx_buf));
        WARN_ON(xfer->rx_buf && object_is_on_stack(xfer->rx_buf));

        ret = ctlr->transfer_one(ctlr, msg->spi, xfer);
        if (ret < 0) {
            /* هنا ضيف dump_stack() لو الخطأ غير متوقع */
            dev_err(&msg->spi->dev,
                    "transfer failed: %d, len=%d\n",
                    ret, xfer->len);
            dump_stack();
            break;
        }
    }
    return ret;
}

/* في probe() للتأكد من إن الـ controller جاهز */
static int my_spi_probe(struct spi_device *spi)
{
    /* تأكد من الـ mode المدعوم */
    WARN_ON(spi->mode & ~(SPI_MODE_3 | SPI_CS_HIGH));

    /* تأكد من الـ speed */
    WARN_ON(spi->max_speed_hz > MY_MAX_FREQ);
}
```

أهم نقاط للـ `WARN_ON()` في الـ SPI:
- قبل كل DMA transfer: تأكد إن الـ buffer مش على الـ stack
- في `setup()`: تأكد إن مفيش transfer pending
- في `transfer_one()`: تأكد من الـ return value بشكل صريح
- في interrupt context: تأكد إن `spi_async()` هو اللي بيستخدم مش `spi_sync()`

---

### Hardware Level

#### 1. التحقق من حالة الـ Hardware مقابل الـ Kernel State

```bash
# شوف الـ mode اللي الـ kernel شايفه
cat /sys/bus/spi/devices/spi0.0/modalias

# بعض الـ controllers بتكشف الـ registers عبر sysfs
# مثلاً الـ BCM2835 (Raspberry Pi)
devmem2 0x3F204000 w   # SPI0_CS register
devmem2 0x3F204004 w   # SPI0_FIFO
devmem2 0x3F204008 w   # SPI0_CLK
devmem2 0x3F20400C w   # SPI0_DLEN

# قارن الـ clock المحسوب بالفعلي
# الـ max_speed_hz في kernel ≠ الـ clock الفعلي (بسبب التقريب للـ divider)
# استخدم oscilloscope لقياس الـ SCLK فعلياً
```

**جدول مقارنة حالة الـ Kernel والـ Hardware:**

| معلومة | Kernel Source | Hardware Verification |
|---|---|---|
| Clock mode (CPOL/CPHA) | `spi->mode & SPI_MODE_3` | Oscilloscope على SCLK وقت CS assert |
| Chip Select polarity | `spi->mode & SPI_CS_HIGH` | Multimeter أو scope على CS pin |
| Clock frequency | `spi->max_speed_hz` | Scope: measure SCLK period |
| Bus number | `spi->controller->bus_num` | تطابق مع datasheet الـ SOC |
| bits per word | `spi->bits_per_word` | عدّ الـ clock edges بين CS pulses |

---

#### 2. Register Dump Techniques

```bash
# devmem2 (الأكثر شيوعاً — يحتاج تثبيت)
apt install devmem2   # Debian/Ubuntu
# أو بناء من source

# مثال: SPI0 على BCM2835 (RPi)
BASE=0x3F204000
echo "=== SPI0 Registers ==="
devmem2 $((BASE + 0x00)) w   # CS: control & status
devmem2 $((BASE + 0x08)) w   # CLK: clock divider
devmem2 $((BASE + 0x0C)) w   # DLEN: data length

# عبر /dev/mem مباشرة (بيحتاج CONFIG_DEVMEM=y)
python3 -c "
import mmap, struct, os
fd = os.open('/dev/mem', os.O_RDONLY | os.O_SYNC)
mem = mmap.mmap(fd, 0x1000, offset=0x3F204000)
val = struct.unpack('<I', mem[:4])[0]
print(f'SPI_CS = 0x{val:08X}')
mem.close(); os.close(fd)
"

# io utility (من package iotools)
io -4 0x3F204000    # قرأ register 32-bit

# للـ DesignWare SPI controller (شائع في Intel SOCs)
BASE=0xFF120000   # مثال فقط — راجع datasheet
devmem2 $BASE w             # CTRLR0: control register 0
devmem2 $((BASE + 0x04)) w  # CTRLR1: control register 1
devmem2 $((BASE + 0x08)) w  # SSIENR: enable register
devmem2 $((BASE + 0x10)) w  # SER: slave enable register
devmem2 $((BASE + 0x14)) w  # BAUDR: baud rate register
devmem2 $((BASE + 0x2C)) w  # SR: status register
devmem2 $((BASE + 0x30)) w  # IMR: interrupt mask register
devmem2 $((BASE + 0x3C)) w  # RISR: raw interrupt status register

# script جاهز لـ dump كامل
for offset in 0x00 0x04 0x08 0x0C 0x10 0x14 0x18 0x1C 0x20 0x2C 0x30 0x3C; do
    val=$(devmem2 $((BASE + offset)) w 2>&1 | grep "Value" | awk '{print $NF}')
    printf "offset 0x%02X = %s\n" $offset "$val"
done
```

---

#### 3. Logic Analyzer / Oscilloscope

**الإعداد الأساسي:**

```
Channels needed:
  CH1 → SCLK
  CH2 → MOSI
  CH3 → MISO
  CH4 → CS (نSS)

Trigger: Rising/Falling edge على CS (CH4)

Sample Rate: >= 10x الـ SPI clock
  مثلاً لو clock = 1 MHz → sample rate >= 10 MHz
```

**ما تبحث عنه:**

```
SPI Mode 0 (CPOL=0, CPHA=0) — الـ Pattern الصحيح:

CS   ‾‾‾\___________________________/‾‾‾
          ↑ CS يهبط = بداية transfer
SCLK ____/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\____
           ↑ clock يبدأ بعد CS مباشرة
MOSI ----<D7><D6><D5><D4><D3><D2><D1><D0>----
          ↑ بيتغيّر على الـ falling edge
MISO ----<D7><D6><D5><D4><D3><D2><D1><D0>----
```

**Checklist بالـ Scope:**

| ما تقيسه | القيمة المتوقعة | لو مختلف |
|---|---|---|
| SCLK frequency | ≤ `max_speed_hz` | راجع clock divider في الـ driver |
| CS polarity | Low active (عادةً) | لو High، أضف `SPI_CS_HIGH` في mode |
| CPOL | مطابق لـ `spi->mode & SPI_CPOL` | غيّر الـ mode أو صحّح الـ DT |
| CPHA | مطابق لـ `spi->mode & SPI_CPHA` | same as above |
| CS setup time | ≥ `cs_setup` في `spi_device` | زوّد `cs_setup` delay |
| CS hold time | ≥ `cs_hold` في `spi_device` | زوّد `cs_hold` delay |
| Bus idle بين transfers | CS high, SCLK at CPOL level | لو CS مش بيرفع، مشكلة في cs_change flag |

---

#### 4. مشاكل الـ Hardware الشائعة ← Kernel Log Patterns

| المشكلة الـ Hardware | ما تشوفه في الـ Kernel Log |
|---|---|
| **مفيش voltage على VCC** | Driver يـ probe بس فيـ timeout في أول transfer |
| **MOSI/MISO مقلوبين** | الـ data المستقبَلة عكس المتوقعة — مفيش error لكن النتائج غلط |
| **CS مش واصل** | `spi transfer timed out` دايماً في كل message |
| **Clock mode غلط** | Data يتقرأ بـ 50% خطأ أو بشكل عشوائي — مفيش error في الـ log |
| **Speed عالي جداً** | بيظهر `EIO` أو بيانات خاطئة متقطعة |
| **Ground loop / noise** | Sporadic errors، الـ `errors` counter بيزيد تدريجياً |
| **DMA buffer alignment** | `DMA mapping failed` أو `cache coherency` errors |
| **Pull-up مفقود على MISO** | تقرأ `0xFF` دايماً حتى لو الـ device مش بيرد |

---

#### 5. Device Tree Debugging

```bash
# تحقق من الـ DT اللي اتحمّل فعلاً (مش الـ source)
dtc -I fs /sys/firmware/devicetree/base | grep -A 20 "spi@"

# أو بشكل أوضح مع fdtget
fdtget /sys/firmware/fdt /soc/spi@ff120000 clock-frequency
fdtget /sys/firmware/fdt /soc/spi@ff120000 compatible
fdtget /sys/firmware/fdt /soc/spi@ff120000 status

# شوف كل الـ SPI nodes
find /sys/firmware/devicetree/base -name "compatible" | xargs grep -l "spi" 2>/dev/null

# تحقق من الـ interrupts
cat /proc/interrupts | grep spi

# تحقق من الـ clocks المرتبطة بالـ SPI controller
cat /sys/kernel/debug/clk/clk_summary | grep spi

# تحقق من الـ pinctrl settings
cat /sys/kernel/debug/pinctrl/*/pinconf-groups | grep -A 5 "spi"
cat /sys/kernel/debug/pinctrl/*/pins | grep -i "spi\|mosi\|miso\|sclk"

# تحقق من الـ regulator
cat /sys/kernel/debug/regulator/regulator_summary | grep spi
```

**مثال DT صحيح مقابل غلط:**

```dts
/* صح — مع كل الـ properties المهمة */
&spi0 {
    status = "okay";
    #address-cells = <1>;
    #size-cells = <0>;

    flash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;                          /* chip select 0 */
        spi-max-frequency = <50000000>;     /* 50 MHz */
        spi-tx-bus-width = <1>;
        spi-rx-bus-width = <4>;             /* QSPI read */
        /* لازم يتطابق مع الـ hardware */
    };
};

/* غلط — missing cs أو speed عالي جداً للـ board */
flash@0 {
    spi-max-frequency = <100000000>;   /* ربما أعلى من قدرة الـ controller */
    /* مفيش reg = <0> → هيفشل في binding */
};
```

**للتحقق من تطابق الـ DT مع الـ Hardware:**

```bash
# قارن الـ cs المطلوب بما هو متاح
cat /sys/class/spi_master/spi0/of_node/spi-num-chipselects 2>/dev/null

# شوف الـ actual bus number
cat /sys/class/spi_master/spi0/of_node/reg

# تحقق إن الـ driver اتـ bind صح
ls /sys/bus/spi/devices/spi0.0/driver   # لازم يشاور على الـ driver
dmesg | grep "spi0.0"                   # شوف probe messages
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للـ Copy

```bash
#!/bin/bash
# === SPI Debug Toolkit ===

SPI_BUS=0
SPI_DEV=0
SPI_NODE="spi${SPI_BUS}.${SPI_DEV}"

echo "=== [1] SPI Bus Info ==="
ls /sys/class/spi_master/

echo -e "\n=== [2] Controller Statistics ==="
for stat in messages transfers errors timedout bytes bytes_tx bytes_rx transfers_split_maxsize; do
    val=$(cat /sys/class/spi_master/spi${SPI_BUS}/statistics/$stat 2>/dev/null || echo "N/A")
    echo "  $stat = $val"
done

echo -e "\n=== [3] Device Statistics ==="
for stat in messages transfers errors timedout; do
    val=$(cat /sys/bus/spi/devices/${SPI_NODE}/statistics/$stat 2>/dev/null || echo "N/A")
    echo "  $stat = $val"
done

echo -e "\n=== [4] Device Info ==="
echo "  modalias = $(cat /sys/bus/spi/devices/${SPI_NODE}/modalias 2>/dev/null)"
echo "  driver   = $(ls -l /sys/bus/spi/devices/${SPI_NODE}/driver 2>/dev/null | awk '{print $NF}')"

echo -e "\n=== [5] DT/OF Info ==="
cat /sys/bus/spi/devices/${SPI_NODE}/of_node/spi-max-frequency 2>/dev/null | od -An -tu4 | xargs echo "  max_speed_hz ="

echo -e "\n=== [6] Interrupts ==="
cat /proc/interrupts | grep -i "spi\|${SPI_NODE}"

echo -e "\n=== [7] Kernel Messages ==="
dmesg | grep -i "spi" | tail -30
```

```bash
# === ftrace one-liner ===
# تتبع SPI transfers لمدة 5 ثواني

echo 0 > /sys/kernel/tracing/tracing_on
echo > /sys/kernel/tracing/trace
echo 1 > /sys/kernel/tracing/events/spi/enable
echo 1 > /sys/kernel/tracing/tracing_on
sleep 5
echo 0 > /sys/kernel/tracing/tracing_on
echo 0 > /sys/kernel/tracing/events/spi/enable
cat /sys/kernel/tracing/trace | grep -v "^#" | head -50
```

```bash
# === Dynamic Debug one-liner ===
echo 'file drivers/spi/spi.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
# p = printk, f = function name, l = line number, m = module, t = thread ID
dmesg -w | grep "spi.c"
```

```bash
# === spidev loopback test ===
# يحتاج: MOSI و MISO مربوطين ببعض، وCONFIG_SPI_SPIDEV=y

# تحقق من وجود الـ device
ls /dev/spidev*

# اختبار بسيط
cat /dev/urandom | head -c 64 > /tmp/spi_tx.bin
./spidev_test -D /dev/spidev0.0 -s 1000000 --input-file /tmp/spi_tx.bin -v 2>&1

# لو الـ output = الـ input → الـ loopback شغال → الـ controller OK
diff /tmp/spi_tx.bin /tmp/spi_rx.bin && echo "LOOPBACK OK" || echo "LOOPBACK FAILED"
```

```bash
# === SPI Flash JEDEC ID Read ===
# يحتاج spidev أو يستخدم mtd layer

# عبر spidev_test
./spidev_test -D /dev/spidev0.0 -s 1000000 -p "\x9f\x00\x00\x00" -v
# Output مثال:
# TX | 9F 00 00 00
# RX | 00 EF 40 18   ← EF = Winbond، 40 = SPI NOR، 18 = 128Mb

# عبر flashrom
flashrom -p linux_spi:dev=/dev/spidev0.0,spispeed=4000 --identify

# عبر MTD
mtdinfo -a
cat /proc/mtd
```

#### مثال Output كامل مع التفسير

```
# dmesg | grep spi | head -20

[    1.234567] spi-bcm2835 3f204000.spi: could not get clk: -ENOENT
  ↑ مشكلة في الـ clock — تحقق من الـ DT وإن الـ clock driver اتـ load

[    1.345678] spi-bcm2835 3f204000.spi: SPI controller driver registered
  ↑ الـ controller اتسجّل بنجاح

[    1.456789] spi0.0: modalias spi-nor, cs0, bits 8, max 50000000 Hz, mode 0
  ↑ device اتـ probe: cs=0، 8-bit، 50MHz، SPI Mode 0 — كل حاجة OK

[    5.678901] spi0.0: SPI transfer timed out
  ↑ المشكلة هنا — راجع الـ CS، الـ clock، والـ MISO

# cat /sys/class/spi_master/spi0/statistics/errors
42
  ↑ 42 error — كتير — الـ hardware في مشكلة

# cat /sys/class/spi_master/spi0/statistics/timedout
42
  ↑ نفس العدد → كل الأخطاء timeouts → الـ device مش بيرد أصلاً

# الخطوة التالية: تحقق بالـ scope إن CS بينزل وإن الـ device واخد VCC
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: SPI Flash مش بيشتغل على Industrial Gateway بـ RK3562

#### العنوان
**SPI NOR Flash مش بييجي على RK3562 — بيطلع kernel panic في early boot**

#### السياق
شركة بتبني **industrial IoT gateway** بـ Rockchip RK3562. الـ gateway ده بيشتغل على Linux مع rootfs موجود على SPI NOR Flash (W25Q128). البورد بيعمل bring-up وكل حاجة ماشية في U-Boot، بس الـ kernel بيموت في اول ثانية وبيطلع panic بيقول "spi-nor: failed to get spi_device".

#### المشكلة
الـ SPI controller رقم 1 على الـ SoC معمول له DT node، لكن الـ SPI NOR Flash مش بيتعرف عليه. الـ kernel بيطلع:

```
[    2.341] spi-rockchip ff4e0000.spi: registered master spi1
[    2.342] spi spi1.0: modalias 'spi-nor' not matched
[    2.342] spi spi1.0: no driver found
```

#### التحليل
بناءً على `spi-summary.rst`، الـ SPI subsystem بيعتمد على الـ `modalias` لربط الـ protocol driver بالـ device. لما الـ DT بيعرف الـ device، الـ kernel بييجي يعمل match بين الـ `compatible` string والـ driver المسجل.

الخطأ هنا إن الـ DT node بيستخدم:
```
compatible = "spi-nor";
```
بدل ما يستخدم compatible string صح. زي ما بيشرح `spi-summary.rst`:

> *"The board_info should provide enough information to let the system work without the chip's driver being loaded."*

الـ `spi_board_info` أو الـ DT node لازم يحدد `compatible` و `reg` بشكل صحيح عشان الـ driver model يقدر يعمل bind.

بالإضافة لده، لما راجعنا الـ DT:
```dts
/* DT خاطئ */
&spi1 {
    status = "okay";
    spi-nor@0 {
        compatible = "spi-nor";   /* generic — مش مدعوم */
        reg = <0>;
        spi-max-frequency = <50000000>;
    };
};
```

#### الحل

**أولاً: تعديل الـ Device Tree**

```dts
/* DT صح */
&spi1 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&spi1_pins>;

    flash@0 {
        compatible = "winbond,w25q128", "jedec,spi-nor"; /* compatible صح */
        reg = <0>;
        spi-max-frequency = <80000000>;
        /* SPI mode 0 هو الـ default */
    };
};
```

**ثانياً: تأكيد إن الـ driver موجود في kernel config**

```bash
grep -i SPI_NOR /boot/config-$(uname -r)
# لازم يطلع: CONFIG_MTD_SPI_NOR=y
# و: CONFIG_SPI_ROCKCHIP=y
```

**ثالثاً: debug بعد التعديل**

```bash
# شوف الـ SPI devices المعروفة
ls /sys/bus/spi/devices/

# تأكيد الـ binding
cat /sys/bus/spi/devices/spi1.0/modalias

# اختبار الـ flash
mtd_debug info /dev/mtd0
```

#### الدرس المستفاد
الـ `compatible` في الـ DT لازم يكون specific للـ chip زي `"winbond,w25q128"` مش generic زي `"spi-nor"`. الـ SPI subsystem بيستخدم الـ modalias لعمل bind، وكما شرح `spi-summary.rst` الـ declaration لازم يكون كافي لتشغيل النظام حتى لو الـ driver مش loaded.

---

### السيناريو الثاني: SPI Touchscreen بيعمل Glitch على STM32MP1 Android Tablet

#### العنوان
**SPI Touchscreen (ADS7846) على STM32MP1 بيرجع قراءات غلط بسبب SPI clock mode خاطئ**

#### السياق
شركة بتعمل **Android tablet** بـ STM32MP157. الـ touchscreen controller هو ADS7846 متوصل على SPI2. الـ touch بيشتغل لكن بيطلع إن الـ cursor بيتحرك بشكل عشوائي في بعض الأوقات، وأحياناً بيتجمد. المشكلة بتظهر أكتر لما الـ CPU load بتزيد.

#### المشكلة
الـ ADS7846 بيشتغل على SPI Mode 0 (CPOL=0, CPHA=0). لكن بعض الـ SPI تحويلات بتحصل وفيه devices تانية على نفس الـ bus بتستخدم mode مختلف. الـ setup() بيتعمل call لكن مش في الوقت الصح.

```
[  234.112] spi2.0: Transfer error -EIO
[  234.113] ads7846: spi transfer failed, aborting
```

#### التحليل
زي ما شرح `spi-summary.rst` في قسم الـ Controller Methods:

> *"BUG ALERT: for some reason the first version of many spi_controller drivers seems to get this wrong. When you code setup(), ASSUME that the controller is actively processing transfers for another device."*

الـ `ctlr->setup()` بيتعمل call مع كل `spi_setup()` call، والـ controller المبني على STM32 بيغير الـ CPOL/CPHA registers في الحال، حتى لو فيه transfer جاري على device تانية على نفس الـ bus. ده بيخلي corruption في بيانات الـ device التانية.

كمان `spi-summary.rst` بيشرح الـ clock modes بالتفصيل:

```
CPOL=0, CPHA=0 → SPI Mode 0  (ADS7846 المطلوب)
CPOL=0, CPHA=1 → SPI Mode 1
CPOL=1, CPHA=0 → SPI Mode 2
CPOL=1, CPHA=1 → SPI Mode 3
```

الـ ADS7846 بيدعم Mode 0 و Mode 3 لأنه "doesn't care about polarity, and always clock data in/out on rising clock edges" — بس الـ driver محدد Mode 0 فقط في الـ DT.

#### الحل

**أولاً: DT صح مع chip-select handling**

```dts
/* STM32MP1 SPI2 DT */
&spi2 {
    status = "okay";
    cs-gpios = <&gpioz 3 GPIO_ACTIVE_LOW>;

    touchscreen@0 {
        compatible = "ti,ads7846";
        reg = <0>;
        spi-max-frequency = <1920000>; /* 3V board: 120000 * 16 */
        interrupts-extended = <&gpiof 14 IRQ_TYPE_EDGE_FALLING>;
        pendown-gpio = <&gpiof 14 GPIO_ACTIVE_LOW>;
        ti,x-plate-ohms = /bits/ 16 <580>;
        ti,y-plate-ohms = /bits/ 16 <410>;
        ti,vref-delay-usecs = /bits/ 16 <100>;
        spi-cpol;   /* اختار Mode 3 بدل Mode 0 لتجنب الـ edge cases */
        spi-cpha;
    };
};
```

**ثانياً: تأكد من isolation كل device**

```bash
# شوف إيه الـ devices الموجودة على SPI2
ls /sys/class/spi_master/spi2/

# اتحقق من الـ mode لكل device
cat /sys/bus/spi/devices/spi2.0/of_node/spi-cpol
cat /sys/bus/spi/devices/spi2.0/of_node/spi-cpha
```

**ثالثاً: debug باستخدام Logic Analyzer**

```bash
# شيل الـ timing باستخدام oscilloscope على SCLK pin
# تأكيد إن الـ clock idle state صح قبل CS assertion
# CPOL=0: clock idle LOW
# CPOL=1: clock idle HIGH
```

#### الدرس المستفاد
الـ SPI mode (CPOL/CPHA) لازم يتحدد بدقة في الـ DT. الـ ADS7846 بيدعم Mode 0 وMode 3 — استخدام Mode 3 ممكن يكون أكثر safety لأن الـ clock idle high بيفصل الـ device بشكل أوضح. والـ bug المذكور في الـ documentation حقيقي جداً في الـ STM32 SPI driver.

---

### السيناريو الثالث: i.MX8M Plus بـ SPI OLED Display — spidev مش بيشتغل من Userspace

#### العنوان
**Python script مش بيقدر يفتح `/dev/spidev0.0` على i.MX8M Plus — manufacturing tool معطل**

#### السياق
شركة بتعمل **IoT sensor node** بـ NXP i.MX8M Plus. فيه OLED display صغير متوصل على SPI1 بيتكلم معاه firmware updater tool مكتوب بـ Python. الـ script بيحتاج يفتح `/dev/spidev0.0` لكن الـ device مش موجود في الـ `/dev`.

#### المشكلة
الـ `/dev/spidev0.0` مش بيظهر حتى لو الـ SPI controller شغال. الـ dmesg بيطلع:

```
[    3.21] spi-imx 30820000.spi: registered master spi0
[    3.22] spi spi0.0: no driver found for spi0.0
```

#### التحليل
بناءً على `spidev.rst`، الـ spidev driver بيحتاج compatible string محدد في الـ DT. الـ documentation بيوضح:

> *"It used to be supported to define an SPI device using the 'spidev' name... But this is no longer supported by the Linux kernel and instead a real SPI device name as listed in one of the tables must be used."*

```
/* DT خاطئ — قديم */
&ecspi1 {
    oled@0 {
        compatible = "spidev";  /* مش بيشتغل في الكيرنل الحديث */
        reg = <0>;
        spi-max-frequency = <10000000>;
    };
};
```

الـ kernel بيطبع error ولا بيعمل probe. لازم الـ compatible يكون موجود في `spidev_dt_ids[]` table.

#### الحل

**أولاً: تعديل الـ DT باستخدام compatible مدعوم**

```dts
/* i.MX8M Plus ECSPI1 */
&ecspi1 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_ecspi1>;
    cs-gpios = <&gpio5 9 GPIO_ACTIVE_LOW>;

    oled_display@0 {
        compatible = "rohm,dh2228fv";  /* compatible مدعوم في spidev_dt_ids */
        reg = <0>;
        spi-max-frequency = <10000000>;
    };
};
```

**أو إضافة entry جديد في spidev driver (الطريقة الصح للـ upstream)**

```c
/* في drivers/spi/spidev.c */
static const struct of_device_id spidev_dt_ids[] = {
    /* ... entries موجودة ... */
    { .compatible = "my-company,oled-spi" },  /* entry جديد */
    {},
};
```

**ثانياً: بديل بدون تعديل DT — manual bind**

```bash
# بعد boot
echo spidev > /sys/bus/spi/devices/spi0.0/driver_override
echo spi0.0 > /sys/bus/spi/drivers/spidev/bind

# تأكيد
ls /dev/spidev*
# يطلع: /dev/spidev0.0
```

**ثالثاً: اختبار الـ Python script**

```python
import spidev

spi = spidev.SpiDev()
spi.open(0, 0)                    # bus=0, device=0
spi.max_speed_hz = 10_000_000     # 10 MHz
spi.mode = 0                      # SPI Mode 0

# إرسال command للـ OLED
resp = spi.xfer2([0xAE])          # Display OFF command
spi.close()
```

**رابعاً: debug الـ sysfs**

```bash
# تأكيد الـ spidev binding
ls /sys/class/spidev/
# يطلع: spidev0.0

# شوف الـ major/minor number
cat /sys/class/spidev/spidev0.0/dev
# يطلع: 153:0

# تأكيد الـ device node
ls -la /dev/spidev0.0
```

#### الدرس المستفاد
الـ `compatible = "spidev"` أتوقفت منذ زمن. لازم تستخدم compatible string حقيقي موجود في `spidev_dt_ids[]`، أو تضيف واحد جديد عن طريق patch. الـ manual bind بالـ `driver_override` هو حل مؤقت مناسب للـ development.

---

### السيناريو الرابع: AM62x Automotive ECU — SPI Full-Duplex مع ADC بيتقطع البيانات

#### العنوان
**SPI ADC readings على AM62x بتطلع corrupt في الـ high-speed full-duplex transfers**

#### السياق
شركة بتبني **automotive ECU** بـ TI AM62x للـ dashboard controls. فيه ADC خارجي (MCP3204) متوصل على McSPI0. الـ driver بيقرأ sensor data بـ full duplex (بيبعت command ويستقبل نتيجة في نفس الـ transfer). البيانات صح عند speeds منخفضة لكن عند 10MHz وفوق بتطلع قيم غلط.

#### المشكلة
الـ `spi_write_then_read()` بيستخدم stack buffer داخلياً وده بيسبب مشكلة مع DMA. الـ driver المكتوب بيستخدم:

```c
/* كود خاطئ — Stack buffer */
u8 tx[3], rx[3];  /* على الـ stack — خطير مع DMA */
tx[0] = 0x01;
tx[1] = (channel << 6);
tx[2] = 0x00;

spi_write_then_read(spi, tx, 3, rx, 3);
```

#### التحليل
`spi-summary.rst` بيحذر بوضوح:

> *"Follow standard kernel rules, and provide DMA-safe buffers in your messages. That way controller drivers using DMA aren't forced to make extra copies unless the hardware requires it."*

و:

> *"I/O buffers use the usual Linux rules, and must be DMA-safe. You'd normally allocate them from the heap or free page pool. **Don't use the stack**, or anything that's declared 'static'."*

الـ AM62x McSPI controller بيستخدم DMA بشكل aggressive. لما الـ buffer على الـ stack، الـ DMA ممكن يقرأ من address غلط أو بعد ما الـ stack frame اتمسح، وده بيدي readings غلط أو corrupt.

كمان `spi_write_then_read()` نفسه — زي ما بيشرح الـ documentation:

> *"The spi_write_then_read() call... should only be used with small amounts of data where the cost of an extra copy may be ignored."*

#### الحل

**أولاً: تصحيح الـ driver باستخدام DMA-safe buffers**

```c
struct mcp3204_data {
    struct spi_device *spi;
    /* DMA-safe buffers — جزء من الـ struct المتخصصة في heap */
    u8 tx_buf[3] ____cacheline_aligned;
    u8 rx_buf[3] ____cacheline_aligned;
};

static int mcp3204_read_channel(struct mcp3204_data *data, int channel)
{
    struct spi_transfer xfer = {
        .tx_buf = data->tx_buf,   /* heap buffer — DMA-safe */
        .rx_buf = data->rx_buf,   /* heap buffer — DMA-safe */
        .len    = 3,
    };
    struct spi_message msg;
    int ret;

    data->tx_buf[0] = 0x01;               /* start bit */
    data->tx_buf[1] = (channel << 6);     /* channel select */
    data->tx_buf[2] = 0x00;               /* don't care */

    spi_message_init(&msg);
    spi_message_add_tail(&xfer, &msg);

    ret = spi_sync(data->spi, &msg);      /* blocking — context that can sleep */
    if (ret)
        return ret;

    /* result is in bits 8:0 of the 12-bit ADC */
    return ((data->rx_buf[1] & 0x0F) << 8) | data->rx_buf[2];
}
```

**ثانياً: تأكيد الـ DMA في الـ DT**

```dts
/* AM62x McSPI0 */
&mcspi0 {
    status = "okay";
    ti,spi-num-cs = <4>;

    adc@0 {
        compatible = "microchip,mcp3204";
        reg = <0>;
        spi-max-frequency = <1800000>; /* MCP3204 max 1.8MHz at 2.7V */
        spi-cpha;
    };
};
```

**ثالثاً: verification**

```bash
# شوف إيه الـ DMA channels المستخدمة
cat /sys/kernel/debug/dmaengine/summary | grep spi

# قرأ قيم الـ ADC
cat /sys/bus/iio/devices/iio:device0/in_voltage0_raw
```

#### الدرس المستفاد
الـ stack buffers ممنوعة مع SPI DMA transfers. دايماً allocate الـ I/O buffers من الـ heap كـ part من الـ device driver data struct. الـ `spi_write_then_read()` مناسبة بس للـ small RPC requests زي ما بتشرح الـ documentation — مش للـ high-speed continuous ADC reads.

---

### السيناريو الخامس: Allwinner H616 Android TV Box — Quad-SPI Flash بيشتغل بـ Single Lane بدل Quad

#### العنوان
**SPI NOR Flash على Allwinner H616 بيشتغل بـ single-lane SPI بدل Quad-SPI — الـ boot بطيء جداً**

#### السياق
شركة بتبني **Android TV box** بـ Allwinner H616. الـ rootfs على QSPI NOR Flash (GD25Q128). الـ system بيشتغل لكن الـ boot time بياخد 45 ثانية بدل 8 ثانية المتوقعة. الـ profiling بيبين إن قراءة الـ kernel من الـ flash هي الـ bottleneck.

#### المشكلة
الـ Flash قادر يشتغل بـ Quad SPI (4 data lines) لكن الـ driver بيتعامل معاه كـ standard SPI (1 data line). النتيجة: throughput ربع المتوقع.

```bash
# ايه بيطلع في dmesg
[    1.23] spi-nor spi0.0: gd25q128 (16384 Kbytes)
[    1.24] spi-nor spi0.0: detected single SPI mode
```

#### التحليل
بناءً على `multiple-data-lanes.rst`، الـ SPI subsystem بيدعم multiple data lanes عن طريق:
- `spi-rx-bus-width` و `spi-tx-bus-width` properties في الـ DT
- الـ `multi_lane_mode` field في `spi_transfer`

لكن للـ QSPI NOR Flash، الآلية مختلفة شوية — الـ SPI NOR subsystem بيستخدم الـ `spi-rx-bus-width` من الـ DT لتحديد هل يستخدم 1x, 2x, أو 4x lines.

الـ DT الحالي:
```dts
/* DT خاطئ — missing quad mode */
&spi0 {
    flash@0 {
        compatible = "gigadevice,gd25q128", "jedec,spi-nor";
        reg = <0>;
        spi-max-frequency = <50000000>;
        /* لا يوجد spi-rx-bus-width ولا spi-tx-bus-width */
    };
};
```

#### الحل

**أولاً: تعديل الـ DT لتفعيل Quad Mode**

```dts
/* H616 SPI0 مع Quad-SPI */
&spi0 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&spi0_pins>;     /* تأكيد إن الـ pinmux شايل 4 data lines */

    flash@0 {
        compatible = "gigadevice,gd25q128", "jedec,spi-nor";
        reg = <0>;
        spi-max-frequency = <100000000>;   /* GD25Q128 max 100MHz */
        spi-rx-bus-width = <4>;            /* Quad SPI read */
        spi-tx-bus-width = <4>;            /* Quad SPI write */
    };
};
```

**ثانياً: تأكيد الـ pinmux يدعم 4 data lines**

```
H616 SPI0 pins الـ Quad mode:
  PC0 → CLK
  PC1 → CS0
  PC2 → MOSI (IO0)
  PC3 → MISO (IO1)
  PC4 → IO2   ← مهم للـ quad mode
  PC5 → IO3   ← مهم للـ quad mode
```

```dts
/* pinctrl لازم يشمل IO2 وIO3 */
spi0_pins: spi0-pins {
    pins = "PC0", "PC1", "PC2", "PC3", "PC4", "PC5";
    function = "spi0";
};
```

**ثالثاً: تحقق من الـ quad mode enabled في Flash register**

```bash
# بعد boot، تحقق إن الـ QE bit (Quad Enable) set في الـ flash status register
mtd_debug read /dev/mtd0 0 1 /tmp/test

# شوف الـ actual speed
dmesg | grep spi-nor
# المفروض يطلع: "gd25q128: detected Quad SPI mode"

# قياس الـ throughput
dd if=/dev/mtd0 of=/dev/null bs=1M count=8 2>&1 | grep MB
```

**رابعاً: إذا الـ controller مش بيدعم multi-lane**

استناداً لـ `multiple-data-lanes.rst`:

> *"To support multiple data lanes, SPI controller drivers need to set `struct spi_controller.num_data_lanes` to a value greater than 1."*

لو الـ H616 SPI controller driver مش بيدعم quad mode، لازم تضيف الـ support:

```c
/* في الـ controller driver */
ctlr->num_data_lanes = 4;   /* إعلان دعم 4 lanes */

/* في transfer_one() */
static int h616_spi_transfer_one(struct spi_controller *ctlr,
                                  struct spi_device *spi,
                                  struct spi_transfer *t)
{
    switch (t->multi_lane_mode) {
    case SPI_MULTI_BUS_MODE_SINGLE:
        /* standard single-lane transfer */
        break;
    case SPI_MULTI_BUS_MODE_STRIPE:
        /* quad-lane parallel transfer */
        h616_enable_quad_mode(ctlr);
        break;
    default:
        return -EINVAL;
    }
    /* ... باقي الـ implementation */
}
```

**خامساً: قياس التحسين**

```bash
# قبل Quad mode
time dd if=/dev/mtd0 of=/dev/null bs=1M count=16
# نتيجة: ~6.25 MB/s (50MHz / 8 bits)

# بعد Quad mode
time dd if=/dev/mtd0 of=/dev/null bs=1M count=16
# نتيجة: ~25 MB/s (50MHz × 4 bits / 8)

# بتحسن الـ boot time من 45s إلى ~10s
```

#### الدرس المستفاد
الـ `spi-rx-bus-width` و `spi-tx-bus-width` في الـ DT مش مجرد documentation — هما بيحددوا الـ actual electrical configuration. الـ Quad SPI بيدي 4x throughput على نفس الـ clock، وده حيوي للـ boot time من flash. تأكيد إن الـ pinmux بيشمل الـ IO2 وIO3 شرط أساسي — بدونهم الـ quad mode بيفشل صامت ويرجع لـ single lane.
## Phase 7: مصادر ومراجع

---

### مصادر رسمية — Kernel Documentation

الـ official kernel documentation للـ SPI subsystem موجودة في المسارات دي:

| المسار | المحتوى |
|--------|---------|
| `Documentation/spi/spi-summary.rst` | نظرة عامة شاملة على الـ SPI subsystem — معمارية، API، كيفية كتابة drivers |
| `Documentation/spi/spidev.rst` | الـ userspace API عبر `/dev/spidevX.Y` |
| `Documentation/spi/multiple-data-lanes.rst` | دعم الـ dual/quad SPI |
| `Documentation/spi/butterfly.rst` | مثال على driver حقيقي |
| `Documentation/spi/spi-lm70llp.rst` | driver للـ LM70 temperature sensor عبر SPI-over-GPIO |
| `Documentation/spi/spi-sc18is602.rst` | driver للـ SC18IS602 I2C-to-SPI bridge |

الـ rendered HTML على موقع kernel.org:
- [Serial Peripheral Interface (SPI) — kernel docs](https://docs.kernel.org/spi/spi-summary.html)
- [SPI userspace API (spidev)](https://docs.kernel.org/spi/spidev.html)

---

### LWN.net Articles

دي أهم المقالات التاريخية اللي وثّقت تطوّر الـ SPI subsystem في Linux:

#### المقالات الأساسية (تسلسل تاريخي)

| العنوان | الرابط | الأهمية |
|---------|--------|---------|
| **SPI core** | [lwn.net/Articles/138037](https://lwn.net/Articles/138037/) | أول patch series لإضافة الـ SPI core — David Brownell |
| **SPI core -- revisited** | [lwn.net/Articles/141186](https://lwn.net/Articles/141186/) | مراجعة وتحسين الـ API الأولي |
| **simple SPI framework** | [lwn.net/Articles/154425](https://lwn.net/Articles/154425/) | تبسيط الـ framework بعد النقاشات الأولى |
| **SPI core refresh** | [lwn.net/Articles/162268](https://lwn.net/Articles/162268/) | تحديث شامل للـ core بعد التجربة العملية |
| **spi** | [lwn.net/Articles/146581](https://lwn.net/Articles/146581/) | نقاش مفصّل حول الـ subsystem design |
| **Add SPI over GPIO driver** | [lwn.net/Articles/290068](https://lwn.net/Articles/290068/) | إضافة الـ `spi-gpio` driver — bitbanging SPI |
| **spi: Add slave mode support** | [lwn.net/Articles/723440](https://lwn.net/Articles/723440/) | إضافة دعم الـ SPI slave mode |
| **Introduction to SPI NAND framework** | [lwn.net/Articles/719733](https://lwn.net/Articles/719733/) | إطار عمل جديد للـ SPI NAND flash |
| **spi: dw: Add generic DW DMA controller support** | [lwn.net/Articles/821801](https://lwn.net/Articles/821801/) | إضافة دعم الـ DMA للـ DesignWare SPI controller |

الـ static kernel documentation على LWN:
- [SPI driver-api على static.lwn.net](https://static.lwn.net/kerneldoc/driver-api/spi.html)
- [spidev userspace API على static.lwn.net](https://static.lwn.net/kerneldoc/spi/spidev.html)

---

### kernelnewbies.org

الموقع بيوثّق التغييرات في كل إصدار kernel. الـ SPI بيظهر في كتير من الإصدارات:

| الإصدار | الرابط | ملاحظات |
|---------|--------|---------|
| Linux 2.6.24 | [kernelnewbies.org/Linux_2_6_24](https://kernelnewbies.org/Linux_2_6_24) | أول دعم MMC/SD over SPI |
| Linux 3.15 | [kernelnewbies.org/Linux_3.15-DriversArch](https://kernelnewbies.org/Linux_3.15-DriversArch) | تحسينات الـ SPI drivers |
| Linux 3.18 | [kernelnewbies.org/Linux_3.18-DriversArch](https://kernelnewbies.org/Linux_3.18-DriversArch) | إضافات drivers متعددة |
| Linux 6.5 | [kernelnewbies.org/Linux_6.5](https://kernelnewbies.org/Linux_6.5) | تحسينات حديثة |
| Linux 6.6 | [kernelnewbies.org/Linux_6.6](https://kernelnewbies.org/Linux_6.6) | تحسينات حديثة |
| Linux 6.12 | [kernelnewbies.org/Linux_6.12](https://kernelnewbies.org/Linux_6.12) | إضافات drivers جديدة |
| Linux 6.17 | [kernelnewbies.org/Linux_6.17](https://kernelnewbies.org/Linux_6.17) | Virtio SPI driver وغيره |

---

### elinux.org — Embedded Platforms

دي صفحات عملية لاستخدام SPI على platforms حقيقية:

| المنصة | الرابط | المحتوى |
|--------|--------|---------|
| **Raspberry Pi SPI** | [elinux.org/RPi_SPI](https://elinux.org/RPi_SPI) | BCM2835 SPI controllers، kernel config، spidev |
| **BeagleBoard SPI** | [elinux.org/BeagleBoard/SPI](https://elinux.org/BeagleBoard/SPI) | pinmux، mcspi driver، device tree |
| **ECE497 SPI Project** | [elinux.org/ECE497_SPI_Project](https://elinux.org/ECE497_SPI_Project) | مثال عملي على BeagleBone مع LED driver |
| **MSIOF SPI GPIO CS Testing** | [elinux.org/Tests:MSIOF-SPI-CS-GPIO](https://elinux.org/Tests:MSIOF-SPI-CS-GPIO) | اختبار GPIO chip selects على Renesas |

---

### Mailing List

الـ SPI subsystem بيستخدم القائمة البريدية الرسمية:

```
linux-spi@vger.kernel.org
```

للبحث في الأرشيف:
- [lkml.iu.edu — مثال على patch للـ spi-imx DMA](https://lkml.iu.edu/hypermail/linux/kernel/1410.2/02312.html)
- [spinics.net/lists/kernel](https://www.spinics.net/lists/kernel/) — أرشيف الـ LKML العام

---

### Kernel Source — ملفات مهمة للقراءة المباشرة

```
drivers/spi/
├── spi.c                  # الـ core — registration، message queue، transfers
├── spi-gpio.c             # SPI via GPIO bitbanging
├── spi-dw-core.c          # DesignWare SPI core driver
├── spi-bcm2835.c          # Raspberry Pi SPI driver
├── spi-omap2-mcspi.c      # Texas Instruments McSPI driver
└── spidev.c               # userspace /dev/spidevX.Y driver

include/linux/spi/
├── spi.h                  # spi_device, spi_controller, spi_message, spi_transfer
└── spi-mem.h              # SPI memory operations (NOR/NAND flash interface)
```

---

### Kernel Commits المهمة

| الحدث | الوصف |
|-------|-------|
| `v2.6.14` | أول إضافة للـ SPI subsystem بواسطة David Brownell |
| `v2.6.17` | تحسين الـ API وإضافة أول drivers حقيقية |
| `v3.4` | إضافة الـ `spi_async()` وتحسين الـ message queue |
| `v4.0` | دعم الـ DMA engine في الـ SPI core |
| `v4.9` | إضافة الـ SPI slave mode API |
| `v5.0` | الـ `spi-mem` interface للـ SPI NOR/NAND flash |
| `v5.14` | تحسين الـ `spi_controller_mem_ops` |

للبحث في الـ git log مباشرة:

```bash
# تاريخ الـ SPI core
git log --oneline drivers/spi/spi.c | head -30

# البحث عن commit معين
git log --all --oneline --grep="spi: Add slave"

# من أضاف الـ SPI core في البداية
git log --follow --oneline drivers/spi/spi.c | tail -5
```

---

### الكتب الموصى بها

#### Linux Device Drivers, 3rd Edition (LDD3)
- المؤلفون: Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصل 14**: The Linux Device Model — أساسي لفهم كيف الـ SPI devices بتتسجّل
- متاح مجانًا: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)
- ملاحظة: قديم نسبيًا (2005)، لكن الـ fundamentals لسه سارية

#### Linux Kernel Development, 3rd Edition — Robert Love
- **الفصل 14**: The Block I/O Layer — بيساعد على فهم الـ I/O stacks بشكل عام
- **الفصل 17**: Devices and Modules — فهم الـ driver model
- الـ SPI مش مغطّى بشكل مباشر، لكن الـ driver infrastructure أساسي

#### Embedded Linux Primer, 2nd Edition — Christopher Hallinan
- **الفصل 8**: Device Drivers — يشرح الـ SPI في سياق embedded systems
- يغطّي الـ device tree وكيف بتحدد SPI devices
- مناسب جدًا لمن بيشتغل على embedded Linux

#### Essential Linux Device Drivers — Sreekrishnan Venkateswaran
- **الفصل 8**: SPI/I2C — فصل مخصص للـ SPI و I2C
- أمثلة عملية وكود حقيقي
- أحدث من LDD3 في تغطية بعض الـ APIs

---

### مصادر إضافية على الإنترنت

- [kernel.org SPI docs v4.14](https://www.kernel.org/doc/html/v4.14/driver-api/spi.html) — نسخة قديمة مفيدة للمقارنة
- [kernel.org SPI docs v5.8](https://www.kernel.org/doc/html/v5.8/spi/spi-summary.html) — نسخة أحدث
- [Toradex: SPI Linux Guide](https://developer.toradex.com/linux-bsp/application-development/peripheral-access/spi-linux/) — دليل عملي من منظور embedded
- [GitHub: torvalds/linux — spi-summary.rst](https://github.com/torvalds/linux/blob/master/Documentation/spi/spi-summary.rst) — أحدث نسخة من الـ documentation

---

### Search Terms مفيدة

للبحث عن معلومات إضافية استخدم الـ keywords دي:

```
"linux spi subsystem" site:lwn.net
"spi_controller" OR "spi_master" linux kernel driver
"spi_transfer" "spi_message" linux kernel
linux spi slave mode driver
linux spi-mem NOR NAND flash interface
linux spi device tree bindings
"spi-gpio" bitbanging linux kernel
linux spidev userspace example
linux spi DMA async transfer
linux-spi@vger.kernel.org archive
```
## Phase 8: Writing simple module

### الفكرة

هنعمل kprobe على الدالة `spi_sync` — أكتر دالة SPI بتتنادى في الـ kernel لأنها الـ synchronous entry point لأي transfer.
كل ما device بيبعت أو يستقبل data عن طريق SPI بيعدي من `spi_sync`، فده بيخلينا نشوف اسم الـ device، الـ bus number، وعدد الـ transfers في كل message.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * spi_sync_probe.c
 *
 * kprobe on spi_sync() — logs every SPI synchronous transfer attempt:
 *   device name, max speed, number of transfers in the message.
 */

#include <linux/module.h>       /* MODULE_*, module_init/exit            */
#include <linux/kernel.h>       /* pr_info                               */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe        */
#include <linux/spi/spi.h>      /* struct spi_device, struct spi_message */

/* ------------------------------------------------------------------ */
/* Pre-handler: fires just before spi_sync() executes                  */
/* ------------------------------------------------------------------ */

/*
 * int spi_sync(struct spi_device *spi, struct spi_message *message);
 *
 * regs->di  = first arg  (spi)     on x86-64
 * regs->si  = second arg (message) on x86-64
 *
 * نستخدم regs_get_kernel_argument() اللي portable عبر الـ architectures.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    struct spi_device  *spi  = (struct spi_device  *)
                                regs_get_kernel_argument(regs, 0);
    struct spi_message *msg  = (struct spi_message *)
                                regs_get_kernel_argument(regs, 1);
    unsigned int        n    = 0;
    struct spi_transfer *t;

    /* Guard against NULL — kprobe يشتغل في أي context */
    if (!spi || !msg)
        return 0;

    /* Count transfers in the message linked list */
    list_for_each_entry(t, &msg->transfers, transfer_list)
        n++;

    pr_info("spi_sync_probe: dev=%s bus=%d cs=%u speed=%u Hz transfers=%u\n",
            dev_name(&spi->dev),          /* e.g. "spi0.0"                */
            spi->controller->bus_num,     /* SPI bus index                */
            spi->chip_select[0],          /* first (usually only) CS      */
            spi->max_speed_hz,            /* max clock configured         */
            n);                           /* how many spi_transfer items  */

    return 0; /* 0 = let spi_sync run normally; non-zero = skip it (dangerous) */
}

/* ------------------------------------------------------------------ */
/* kprobe descriptor                                                   */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "spi_sync",  /* kernel symbol to probe               */
    .pre_handler = handler_pre, /* called before the probed instruction */
};

/* ------------------------------------------------------------------ */
/* module_init / module_exit                                           */
/* ------------------------------------------------------------------ */
static int __init spi_probe_init(void)
{
    int ret;

    /*
     * register_kprobe() بتدور على الـ symbol في الـ kernel symbol table
     * وبتحط breakpoint افتراضي (int3 على x86) قبل أول instruction في الدالة.
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("spi_sync_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("spi_sync_probe: planted at %p\n", kp.addr);
    return 0;
}

static void __exit spi_probe_exit(void)
{
    /*
     * لازم نعمل unregister قبل ما الـ module يتشال من الـ memory،
     * لأن الـ handler بيشاور على كود ضوي الـ module —
     * لو الـ module اتشال والـ kprobe لسه شغال → kernel panic.
     */
    unregister_kprobe(&kp);
    pr_info("spi_sync_probe: removed\n");
}

module_init(spi_probe_init);
module_exit(spi_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on spi_sync — log every SPI synchronous transfer");
```

---

### Makefile

```makefile
obj-m += spi_sync_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل قسم

#### الـ includes

| Header | ليه موجود |
|--------|-----------|
| `<linux/module.h>` | ماكروهات `module_init`, `module_exit`, `MODULE_*` |
| `<linux/kprobes.h>` | `struct kprobe`, `register_kprobe`, `regs_get_kernel_argument` |
| `<linux/spi/spi.h>` | تعريف `struct spi_device` و`struct spi_message` عشان نعمل cast للـ args |

الـ includes دي بتديك كل الـ types اللي محتاجها من غير ما تـ include الـ kernel كله.

---

#### الـ `handler_pre` callback

الـ kprobe بيوقف التنفيذ **قبل** أول instruction في `spi_sync`، يعني الـ arguments لسه في الـ registers.
`regs_get_kernel_argument(regs, 0)` بيجيب الـ argument الأول بطريقة portable عبر x86, ARM64, RISC-V من غير ما تكتب assembly.

نعدّ الـ transfers بـ `list_for_each_entry` على `msg->transfers` — الـ linked list الداخلية اللي بيبني عليها الـ SPI core الـ message — وبكده بنعرف حجم الـ transaction قبل ما تتنفذ.

---

#### الـ `return 0` من الـ handler

القيمة `0` معناها "اكمّل الدالة الأصلية عادي".
لو رجعنا قيمة غير صفر، الـ kprobe framework بيـ skip الدالة ده خطير جداً في الـ production — بستخدمه بس في الـ testing.

---

#### الـ `kprobe` struct

حقل `symbol_name` بيخلي الـ framework يـ resolve العنوان وقت الـ `register_kprobe` بدل ما تـ hardcode address — portable عبر الـ kernel versions.
حقل `pre_handler` هو الـ function pointer للـ callback اللي هيتشغل.

---

#### `module_init` و `module_exit`

`register_kprobe` بتحط الـ breakpoint في الـ kernel text وبتسجل الـ handler.
`unregister_kprobe` في الـ exit **إجبارية** — الـ handler بيـ reference كود جوه الـ module، ولو الـ module اتـ unload والـ probe لسه active هينتج use-after-free وكراش.

---

### تشغيل وتجربة

```bash
# بناء الـ module
make

# تحميله
sudo insmod spi_sync_probe.ko

# شوف الـ output (لو فيه SPI devices شغالة)
sudo dmesg | grep spi_sync_probe

# مثال output على جهاز فيه SPI flash:
# spi_sync_probe: dev=spi0.0 bus=0 cs=0 speed=50000000 Hz transfers=2

# إزالة الـ module
sudo rmmod spi_sync_probe
```

لو مفيش SPI device حقيقي على الجهاز، ممكن تختبر على Raspberry Pi أو بـ `spi-stub` أو أي QEMU machine بيدعم SPI controller.
