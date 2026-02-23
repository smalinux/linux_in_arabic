## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الـ file ده جزء من subsystem اسمه **"MULTIMEDIA CARD (MMC) ETC. OVER SPI"** — موجود في `MAINTAINERS` بحالة `Orphan`، يعني مفيش maintainer بيهتم بيه بشكل رسمي دلوقتي، بس الكود شغال ومتضمن في الـ kernel.

---

### الصورة الكبيرة — القصة من الأول

#### المشكلة

تخيل عندك **microcontroller صغير** — Arduino أو embedded board رخيص — ومحتاج تقرأ وتكتب على **SD card**. الـ board ده مفيش فيه **MMC/SD controller** حقيقي (الـ hardware الأصلي اللي موجود في اللاب توب مثلاً). بس فيه **SPI bus** — وده bus بسيط جداً ومنتشر في كل الـ embedded systems.

الـ SD/MMC cards بتدعم نوعين من الـ protocol:
- **SD/MMC Native mode** — بروتوكول خاص، بيحتاج controller مخصص.
- **SPI mode** — بروتوكول أبسط، أي SPI master يقدر يشتغل بيه.

#### الحل

الـ Linux kernel عمل **driver** اسمه `mmc_spi` — بيخلي أي **SPI master controller** يقدر يتكلم مع SD/MMC card وكأنه MMC host حقيقي. الـ kernel بيشوف الـ card عادي كـ block device (`/dev/mmcblk0`) بدون ما يفرق هل وراها native controller أو SPI.

---

### دور الـ File ده — `mmc_spi.h`

الـ file ده **header صغير** — مش فيه logic. وظيفته الوحيدة إنه يعرّف **`struct mmc_spi_platform_data`**، وهي الـ struct اللي الـ **board-specific platform code** بيحطها كـ `platform_data` على الـ SPI device، عشان يقول للـ driver:

| الحقل | الدور |
|-------|-------|
| `init` | function بتسجل الـ **card-detect IRQ** وتجهز الـ hardware |
| `exit` | cleanup عند remove الـ driver |
| `caps` / `caps2` | **capabilities** زي `MMC_CAP_NEEDS_POLL` لو مفيش IRQ لكشف الـ card |
| `detect_delay` | وقت الـ **debounce** بالـ milliseconds عند insert/remove الـ card |
| `powerup_msecs` | كم millisecond لازم ننتظر بعد ما نشغل الـ power للـ card |
| `ocr_mask` | الـ **voltage ranges** اللي الـ slot بيدعمها |
| `setpower` | function بتشغل/توقف طاقة الـ card (GPIO مثلاً) |

---

### ليه الـ Platform Data ده مهم؟

الـ driver `mmc_spi.c` بيتعامل مع الـ SPI protocol نفسه — بس **كل board** ليها طريقتها الخاصة في:
- كشف إن فيه card اتحطت (card detect pin)
- التحكم في الـ power بتاع الـ slot
- الـ voltage اللي بيقدر يدي

الـ `platform_data` هو الجسر اللي بيخلي الـ generic driver يشتغل صح على أي board من غير ما يبقى فيه board-specific code جوا الـ driver نفسه.

---

### القصة بالكامل — خطوة خطوة

```
Board Code (arch/arm/.../board-xyz.c)
  │
  │  يعرف struct mmc_spi_platform_data
  │  بيحدد: أي GPIO لـ card detect، أي voltage
  │
  ▼
SPI Device Registration
  │
  ▼
mmc_spi.c driver probes
  │
  ├── يجيب platform_data عبر mmc_spi_get_pdata()
  ├── يسجل card-detect IRQ عبر pdata->init()
  ├── يضبط الـ voltage عبر pdata->ocr_mask
  ├── يسجل mmc_host مع الـ MMC core
  │
  ▼
MMC Core
  │
  ▼
Block Device Layer → /dev/mmcblk0
```

---

### الفرق بين Native وSPI Mode

```
Native MMC Controller          SPI + mmc_spi driver
─────────────────────          ────────────────────
4-bit data bus                 1-bit SPI MOSI/MISO
Dedicated hardware             أي SPI master
أسرع بكتير                    أبطأ لكن أرخص
CMD/DAT lines منفصلة          كل حاجة على SPI lines
```

---

### الـ Functions المُعلنة

```c
/* جيب platform_data من الـ SPI device */
extern struct mmc_spi_platform_data *mmc_spi_get_pdata(struct spi_device *spi);

/* رجّع platform_data بعد الاستخدام */
extern void mmc_spi_put_pdata(struct spi_device *spi);
```

الاتنين دول بيتعملوا في `mmc_spi.c` عشان يجيب الـ config اللي الـ board code حطه.

---

### الملفات اللي تعرفها

| الملف | الدور |
|-------|-------|
| `include/linux/spi/mmc_spi.h` | **الـ file ده** — platform_data struct |
| `drivers/mmc/host/mmc_spi.c` | الـ driver الرئيسي — المنطق كله |
| `include/linux/mmc/host.h` | تعريف `struct mmc_host` — الـ MMC core abstraction |
| `include/linux/mmc/mmc.h` | constants زي `MMC_VDD_*` وresponse codes |
| `include/linux/spi/spi.h` | تعريف `struct spi_device` وAPI الـ SPI bus |
| `include/linux/mmc/slot-gpio.h` | helper functions لكشف الـ card عبر GPIO |
| `drivers/mmc/host/` | باقي MMC host drivers (native controllers) |
| `include/linux/mmc/sd.h` | SD-specific definitions |

---

### ملفات الـ Subsystem

**Core:**
- `drivers/mmc/core/` — الـ MMC core layer
- `drivers/mmc/host/mmc_spi.c` — الـ SPI host driver

**Headers:**
- `include/linux/spi/mmc_spi.h` — platform data (الـ file ده)
- `include/linux/mmc/host.h` — mmc_host struct
- `include/linux/mmc/mmc.h` — protocol constants
- `include/linux/spi/spi.h` — SPI bus API

**Hardware Drivers (Native):**
- `drivers/mmc/host/sdhci.c` — SDHCI standard controller
- `drivers/mmc/host/dw_mmc.c` — Synopsys DesignWare
- `drivers/mmc/host/omap_hsmmc.c` — TI OMAP
## Phase 2: شرح الـ MMC-over-SPI Framework

---

### المشكلة اللي الـ Subsystem بيحلها

الـ **MMC/SD cards** بيتكلموا عادةً عبر بروتوكول native بيستخدم controller مخصص — بيتحكم في الـ clock، الـ data lines (4-bit)، والـ command line بشكل parallel. لكن في عالم الـ embedded المحدود، مش دايماً يكون فيه SoC بـ MMC controller حقيقي. كتير من الـ microcontrollers الصغيرة عندها بس **SPI master** — وكارت الـ SD بيدعم SPI mode كـ fallback بروتوكول.

المشكلة هي إزاي تخلي الـ **MMC subsystem** اللي بيتكلم مع drivers عبر `struct mmc_host` يشتغل فوق **SPI subsystem** اللي بيتعامل مع الـ bus بطريقة تانية خالص؟

---

### الحل اللي الكيرنل اتبعه

الكيرنل عمل **glue driver** اسمه `mmc_spi` — بيقع في النص بين الـ MMC core والـ SPI core:

- من فوق: بيسجل نفسه كـ `mmc_host` عند الـ MMC core، فالـ block layer ما بتعرفش إنه بيشتغل على SPI.
- من تحت: بيتكلم مع الـ SPI bus عبر `spi_device`، بيرسل ويستقبل bytes بالترتيب.

الـ **platform_data** بيكون هو الوحيد اللي بيعرف تفاصيل الـ hardware الخاص بالبورد — زي الـ GPIO للـ card detect والـ power control. ده اللي الـ header `mmc_spi.h` بيعرّفه.

---

### Real-World Analogy

تخيل إنك عايز تشغّل طابعة بـ USB على جهاز عنده بس **Serial port (RS-232)**. محتاج:

1. **محوّل** (USB-to-Serial adapter) — ده هو الـ `mmc_spi` driver.
2. **إعدادات المحوّل** (baud rate، power، detect pin) — دي هي الـ `mmc_spi_platform_data`.
3. الـ OS لازم يشوف طابعة USB عادية — زي ما الـ MMC core بيشوف `mmc_host` عادي.

لكن خلينا نكمل الـ analogy بعمق:

| الـ Analogy | الـ Kernel Concept |
|---|---|
| الطابعة USB | SD/MMC card |
| RS-232 Serial port | SPI bus |
| محوّل USB-to-Serial | `mmc_spi` driver |
| إعدادات المحوّل (jumpers/config) | `mmc_spi_platform_data` |
| الـ OS driver stack للطابعة | MMC core + block layer |
| الـ Serial port driver | SPI controller driver |
| "طاقة التشغيل بعد الوصل" | `powerup_msecs` delay |
| "LED بيضوي لما الطابعة تتحط" | card detect IRQ |
| "مش كل طابعة تشتغل على serial" | SPI mode أبطأ من native MMC |

---

### Big Picture Architecture

```
 ┌─────────────────────────────────────────────────┐
 │               User Space                         │
 │   open("/dev/mmcblk0")  read/write               │
 └────────────────────┬────────────────────────────┘
                      │
 ┌────────────────────▼────────────────────────────┐
 │           Block Layer (generic)                  │
 │         bio → request_queue                      │
 └────────────────────┬────────────────────────────┘
                      │
 ┌────────────────────▼────────────────────────────┐
 │           MMC Core  (drivers/mmc/core/)          │
 │  mmc_host_ops:  .request()  .set_ios()           │
 │  يتعامل مع بروتوكول MMC/SD بشكل generic         │
 └────────────────────┬────────────────────────────┘
                      │  struct mmc_host
 ┌────────────────────▼────────────────────────────┐
 │    mmc_spi driver  (drivers/mmc/host/mmc_spi.c) │◄── mmc_spi_platform_data
 │                                                  │    (init, exit, caps,
 │  - يترجم mmc_host_ops لـ SPI transactions        │     setpower, ocr_mask,
 │  - يتعامل مع CRC7/CRC16 يدوياً                  │     detect_delay)
 │  - يدير card detect IRQ                          │
 └────────────────────┬────────────────────────────┘
                      │  struct spi_device
 ┌────────────────────▼────────────────────────────┐
 │           SPI Core  (drivers/spi/spi.c)          │
 │  spi_transfer, spi_message, spi_sync()           │
 └────────────────────┬────────────────────────────┘
                      │
 ┌────────────────────▼────────────────────────────┐
 │       SPI Controller Driver                      │
 │  (e.g. spi-pl022.c, spi-omap2-mcspi.c)          │
 └────────────────────┬────────────────────────────┘
                      │
               ┌──────▼──────┐
               │  SD/MMC Card │  (SPI mode)
               └─────────────┘
```

---

### الـ Core Abstraction

الفكرة المركزية هي **الفصل بين بروتوكول الكارت والـ transport layer**:

- الـ **MMC core** ما بيعرفش هو فوق SPI ولا فوق native controller — بيشوف بس `struct mmc_host` مع `mmc_host_ops`.
- الـ **SPI core** ما بيعرفش إنه بيشيل SD data — بيشوف بس bytes تتبعت وترجع.
- الـ **`mmc_spi` driver** هو الوحيد اللي بيعرف الاتنين، وبيترجم بينهم.

الـ `mmc_spi_platform_data` هو **configuration contract** بين الـ board (platform) والـ driver — بيقوله إيه resources الـ hardware المتاحة.

---

### شرح `struct mmc_spi_platform_data`

```c
struct mmc_spi_platform_data {

    /* (1) Lifecycle hooks — البورد بتقول للـ driver:
     *     عند الـ probe، استدعي init() عشان تسجل IRQ الـ card detect
     *     عند الـ remove، استدعي exit() عشان تحرر الـ resources
     *
     *     الـ irqreturn_t (*)(int, void *) هو handler الـ driver نفسه —
     *     البورد بتاخده وتسجله على الـ GPIO/interrupt المناسب
     */
    int (*init)(struct device *,
        irqreturn_t (*)(int, void *),  /* card-detect IRQ handler */
        void *);                        /* driver data */
    void (*exit)(struct device *, void *);

    /* (2) MMC Capabilities — flags بتتمرر للـ mmc_host
     *     مثلاً MMC_CAP_NEEDS_POLL لو مفيش IRQ للـ card detect
     *     وMMC_CAP_SPI للإعلان إن الـ host يشتغل بـ SPI mode
     */
    unsigned long caps;
    unsigned long caps2;

    /* (3) Card Detect Debounce — كم millisecond ننتظر بعد
     *     حدوث الـ IRQ قبل ما نعتبر الكارت اتحط أو اتشال؟
     *     بيتجنب false positives من الـ mechanical contact bounce
     */
    u16 detect_delay;

    /* (4) Power Management
     *     powerup_msecs: وقت الانتظار بعد تشغيل الطاقة للكارت
     *                    قبل ما نبدأ نتكلم معاه (max 250ms per spec)
     *     ocr_mask: الـ voltages المتاحة على البورد
     *               (OCR = Operating Conditions Register)
     *               مثلاً MMC_VDD_32_33 | MMC_VDD_33_34
     *     setpower: callback البورد لتشغيل/إيقاف الـ voltage regulator
     */
    u16 powerup_msecs;
    u32 ocr_mask;
    void (*setpower)(struct device *, unsigned int maskval);
};
```

---

### Struct Relationship Diagram

```
 Platform/Board Code (e.g. board-foo.c or DT + driver)
 ┌──────────────────────────────────────────┐
 │  static struct mmc_spi_platform_data pd  │
 │  {                                        │
 │    .init     = foo_mmc_init,              │
 │    .exit     = foo_mmc_exit,              │
 │    .setpower = foo_set_power,             │
 │    .ocr_mask = MMC_VDD_32_33,            │
 │    .caps     = MMC_CAP_NEEDS_POLL,       │
 │  };                                       │
 └──────────────┬───────────────────────────┘
                │  stored in spi_device->dev.platform_data
                │  or retrieved via mmc_spi_get_pdata()
                ▼
 ┌──────────────────────────────────────────┐
 │  struct spi_device  (SPI bus device)      │
 │  ├── dev.platform_data → mmc_spi_pdata   │
 │  ├── max_speed_hz                         │
 │  ├── chip_select                          │
 │  └── mode (SPI_MODE_0)                   │
 └──────────────┬───────────────────────────┘
                │  mmc_spi driver reads pdata at probe time
                ▼
 ┌──────────────────────────────────────────┐
 │  struct mmc_spi_host  (driver private)    │
 │  ├── mmc  → struct mmc_host              │
 │  ├── spi  → struct spi_device            │
 │  ├── pdata → mmc_spi_platform_data       │
 │  └── ... (DMA buffers, CRC state, etc.)  │
 └──────────────┬───────────────────────────┘
                │  registered via mmc_add_host()
                ▼
 ┌──────────────────────────────────────────┐
 │  struct mmc_host  (MMC core abstraction)  │
 │  ├── ops → mmc_host_ops                  │
 │  │    ├── .request()  → SPI transactions │
 │  │    ├── .set_ios()  → SPI clock/mode   │
 │  │    └── .get_cd()   → GPIO read        │
 │  ├── ocr_avail  ← من ocr_mask            │
 │  └── caps       ← من caps/caps2          │
 └──────────────────────────────────────────┘
```

---

### الـ API Functions

```c
/* الـ driver بيستخدمهم عشان يجيب الـ platform_data بـ type-safe طريقة */
extern struct mmc_spi_platform_data *mmc_spi_get_pdata(struct spi_device *spi);
extern void mmc_spi_put_pdata(struct spi_device *spi);
```

الـ `mmc_spi_get_pdata()` بترجع الـ `platform_data` المخزنة في الـ `spi_device`. في حالة الـ **Device Tree**، ممكن يكون فيه كود بيبني الـ `mmc_spi_platform_data` ديناميكياً من الـ DT properties وبيحتاج reference counting — ولذلك الـ `put_pdata()` موجودة عشان تحرر الـ memory لو اتعملت dynamically.

---

### الـ Subsystem بيمتلك إيه مقابل اللي بيفوّضه؟

| الـ `mmc_spi` driver بيمتلك | بيفوّض لـ |
|---|---|
| ترجمة MMC commands لـ SPI byte sequences | SPI core: إرسال الـ bytes فعلياً على الـ bus |
| حساب CRC7 (للـ commands) وCRC16 (للـ data) | MMC core: قرار إيه الـ command التالي |
| إدارة دورة الـ card detect IRQ | Board code: تسجيل الـ IRQ على الـ GPIO الصح |
| ضبط الـ SPI clock speed عند تغيير الـ ios | Board code: تشغيل الـ voltage regulator |
| التعامل مع الـ SPI CS line أثناء transaction | SPI controller driver: timing الـ clock والـ data |
| Polling على الـ busy token بعد write | MMC core: scheduling الـ requests من الـ block layer |

---

### مفاهيم مرتبطة تحتاج تعرفها

- **SPI subsystem**: الـ `spi_device` + `spi_controller` + `spi_transfer/message` — ده الـ transport layer اللي `mmc_spi` بيستخدمه من تحت.
- **MMC core**: موجود في `drivers/mmc/core/` — بيدير بروتوكول MMC/SD ويوفر `mmc_host` abstraction للـ host drivers.
- **OCR (Operating Conditions Register)**: register في كارت الـ SD بيحدد الـ voltage ranges اللي يقدر يشتغل بيها — الـ `ocr_mask` في الـ platform_data بيحدد اللي الـ hardware بيوفره.
- **IRQ debounce**: لما الكارت بيتحط، الـ mechanical contacts بتعمل noise — الـ `detect_delay` بيحل المشكلة دي بإنه ينتظر استقرار الـ signal قبل ما يعمل enumerate للكارت.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. Flags, Enums & Config Options — Cheatsheet

الـ `mmc_spi.h` نفسه صغير جداً ومش بيعرّف flags أو enums مستقلة، لكن بيعتمد على macros من الـ MMC core. الجدول التالي بيلخّص كل القيم اللي بتتحط في الـ `caps` و`caps2` fields:

#### `caps` field — MMC Capabilities (من `linux/mmc/host.h`)

| Macro | القيمة | المعنى |
|---|---|---|
| `MMC_CAP_NEEDS_POLL` | `(1 << 5)` | الـ driver لازم يعمل polling لـ card detect (مفيش IRQ) |
| `MMC_CAP_SPI` | `(1 << 21)` | الـ host بيشتغل على SPI bus |
| `MMC_CAP_4_BIT_DATA` | `(1 << 2)` | يدعم 4-bit data bus (مش مستخدم على SPI) |
| `MMC_CAP_SD_HIGHSPEED` | `(1 << 10)` | يدعم SD High Speed mode |
| `MMC_CAP_MMC_HIGHSPEED` | `(1 << 11)` | يدعم MMC High Speed mode |

> على SPI، الـ flag الأهم عملياً هو `MMC_CAP_NEEDS_POLL` لأن معظم الـ boards مبتعملش IRQ line منفصل لـ card detect.

#### `ocr_mask` field — Voltage Ranges

| Macro | Bit(s) | الجهد |
|---|---|---|
| `MMC_VDD_32_33` | bit 20 | 3.2–3.3 V |
| `MMC_VDD_33_34` | bit 21 | 3.3–3.4 V |
| `MMC_VDD_165_195` | bit 7 | 1.65–1.95 V |

> الـ `ocr_mask` بيُمرَّر مباشرةً للـ `mmc_host->ocr_avail`. لو الـ board بيدّي 3.3V بس، تحط `MMC_VDD_32_33 | MMC_VDD_33_34`.

---

### 1. الـ Struct الرئيسي: `mmc_spi_platform_data`

**الغرض:** بيحمل كل المعلومات الخاصة بالـ platform (الـ board) اللي محتاجها الـ `mmc_spi` driver عشان يشغّل SD/MMC card على SPI bus. بيتحط في `spi_device->dev.platform_data`.

```c
struct mmc_spi_platform_data {
    /* --- Card Detect & Init --- */
    int (*init)(struct device *,
                irqreturn_t (*)(int, void *),  /* IRQ handler من الـ driver */
                void *);                        /* handler data */
    void (*exit)(struct device *, void *);

    /* --- MMC Core Capabilities --- */
    unsigned long caps;     /* MMC_CAP_* flags */
    unsigned long caps2;    /* MMC_CAP2_* flags */

    /* --- Card Detect Debounce --- */
    u16 detect_delay;       /* بالـ msecs */

    /* --- Power Management --- */
    u16 powerup_msecs;      /* انتظر كام msec بعد power-on (max 250) */
    u32 ocr_mask;           /* الـ voltages المتاحة */
    void (*setpower)(struct device *, unsigned int maskval);
};
```

#### شرح كل field بالتفصيل

| Field | النوع | الشرح |
|---|---|---|
| `init` | function pointer | الـ platform code بيعمل setup للـ GPIO وبيسجّل الـ card-detect IRQ. الـ driver بيديه الـ IRQ handler عشان يستدعيه لما الـ card تتحرك. |
| `exit` | function pointer | عكس `init` — بتحرر الـ resources (GPIO, IRQ) لما الـ driver يتفك. |
| `caps` | `unsigned long` | بيتنسخ مباشرةً لـ `mmc_host->caps`. |
| `caps2` | `unsigned long` | بيتنسخ لـ `mmc_host->caps2`. |
| `detect_delay` | `u16` (msecs) | بعد ما الـ IRQ ييجي، الـ driver بيستنى كذا ms قبل ما يقرأ state الـ card (debounce). |
| `powerup_msecs` | `u16` (msecs) | الوقت اللازم بعد تشغيل الطاقة قبل ما الـ card تكون جاهزة (spec بيقول max 250ms). |
| `ocr_mask` | `u32` | bitmask للـ voltages اللي الـ board يقدر يوفّرها للـ card. |
| `setpower` | function pointer | الـ platform code بيتحكم في الـ voltage regulator أو power switch للـ card slot. `maskval=0` يعني power off. |

---

### 2. الـ Structs الخارجية المرتبطة

الـ header نفسه صغير، لكن الـ driver (`mmc_spi.c`) بيربط بين عدة structs رئيسية:

#### `spi_device` (من `linux/spi/spi.h`)
بيمثل الـ SD card كـ SPI device على الـ bus. الـ `mmc_spi_platform_data` بيتحط في `spi_device->dev.platform_data`.

| Field المهم | الشرح |
|---|---|
| `max_speed_hz` | أقصى clock speed للـ SPI (لـ SD: 25MHz عادةً) |
| `mode` | `SPI_MODE_0` أو `SPI_MODE_3` (SD بيدعم الاتنين) |
| `bits_per_word` | دايماً 8 على SPI-MMC |

#### `mmc_host` (من `linux/mmc/host.h`)
بيمثل الـ MMC/SD host controller في نظر الـ MMC core.

| Field المهم | الشرح |
|---|---|
| `ops` | pointer لـ `mmc_host_ops` — فيها `request`, `set_ios`, `get_cd` |
| `caps` | بيتملى من `mmc_spi_platform_data->caps` |
| `ocr_avail` | بيتملى من `mmc_spi_platform_data->ocr_mask` |
| `max_blk_size` | 512 bytes على SPI-MMC |

#### `mmc_host_ops`
الـ vtable اللي الـ MMC core بيستخدمها للتعامل مع الـ driver:

| Operation | ما بتعمله |
|---|---|
| `request` | بتنفذ MMC command/data transfers على SPI |
| `set_ios` | بتضبط الـ clock speed والـ bus width والـ power state |
| `get_cd` | بترجع هل في card موجودة ولا لأ |

---

### 3. ASCII Struct Relationship Diagram

```
  ┌─────────────────────────────────────────┐
  │           spi_device                    │
  │  .dev.platform_data ──────────────────► │──┐
  │  .max_speed_hz                          │  │
  │  .mode (SPI_MODE_0/3)                   │  │
  └─────────────────────────────────────────┘  │
                    │                           │
                    │ spi_get_drvdata()         ▼
                    ▼              ┌────────────────────────────┐
  ┌─────────────────────────┐     │  mmc_spi_platform_data     │
  │   mmc_spi (private)     │     │  .init()                   │
  │   (driver priv data)    │     │  .exit()                   │
  │   .spi ─────────────────┼────►│  .caps / .caps2            │
  │   .mmc ──────────┐      │     │  .detect_delay             │
  │   .pdata ────────┼──────┼────►│  .powerup_msecs            │
  └─────────────────────────┘     │  .ocr_mask                 │
                    │             │  .setpower()               │
                    │ (backpointer)└────────────────────────────┘
                    ▼
  ┌─────────────────────────────────────────┐
  │              mmc_host                   │
  │  .ops ──────────────────────────────►   │──┐
  │  .caps  (← from pdata->caps)           │  │
  │  .ocr_avail (← from pdata->ocr_mask)   │  │
  │  .max_blk_size = 512                    │  │
  └─────────────────────────────────────────┘  │
                                                ▼
                               ┌─────────────────────────────┐
                               │       mmc_host_ops          │
                               │  .request()                 │
                               │  .set_ios()                 │
                               │  .get_cd()                  │
                               └─────────────────────────────┘
```

---

### 4. Lifecycle Diagram

```
  BOOT / MODULE LOAD
  ──────────────────
  platform/DT code
    └─► يسجّل spi_device مع platform_data = &mmc_spi_platform_data

  PROBE (mmc_spi_probe)
  ──────────────────────
  spi_driver.probe() يتّسمّى
    ├─► mmc_alloc_host()          ← ينشئ struct mmc_host
    ├─► يملى mmc_host->caps من pdata->caps
    ├─► يملى mmc_host->ocr_avail من pdata->ocr_mask
    ├─► pdata->init(dev, irq_handler, data)
    │     └─► platform code يسجّل card-detect IRQ (أو يضبط polling)
    ├─► mmc_add_host()            ← يسجّل الـ host في MMC core
    └─► الـ card بتتكشف وبتتعرّف (card detection starts)

  RUNTIME
  ────────
  MMC core يطلب operation
    ├─► mmc_host_ops.request()    ← ترسل SPI transactions
    ├─► mmc_host_ops.set_ios()    ← تغيّر clock/power
    └─► card-detect IRQ يوصل
          ├─► debounce (detect_delay ms)
          └─► mmc_detect_change() ← تبلّغ MMC core

  REMOVE / UNBIND
  ────────────────
  spi_driver.remove()
    ├─► mmc_remove_host()         ← يفك تسجيل الـ host
    ├─► pdata->exit(dev, data)    ← platform code يحرر IRQ/GPIO
    └─► mmc_free_host()           ← يحرر memory
```

---

### 5. Call Flow Diagrams

#### 5.1 Card Insertion (IRQ-based)

```
  Hardware: card inserted
    → GPIO edge → IRQ fires
      → pdata->init() registered handler يشتغل
        → mmc_spi irq_handler()
          → schedule delayed_work (detect_delay msecs)
            → mmc_detect_change(mmc_host, detect_delay)
              → MMC core يبدأ card identification
                → mmc_host_ops.request() يتّسمّى
                  → spi_sync() / spi_async()
                    → SPI bus controller
                      → SD card
```

#### 5.2 Power Control (`set_ios`)

```
  MMC core يطلب power change
    → mmc_host_ops.set_ios(host, ios)
      → ios->power_mode == MMC_POWER_ON ?
          → pdata->setpower(dev, pdata->ocr_mask)
            → platform regulator/GPIO يشغّل power
              → msleep(pdata->powerup_msecs)   ← ينتظر الـ card
      → ios->power_mode == MMC_POWER_OFF ?
          → pdata->setpower(dev, 0)
            → platform يقطع الطاقة
      → ios->clock بيتغير
          → spi_setup(spi)  ← يضبط الـ SPI clock
```

#### 5.3 Data Transfer (R/W Block)

```
  userspace: read(fd, buf, 512)
    → block layer
      → MMC core: mmc_request
        → mmc_host_ops.request(host, mrq)
          → mmc_spi_request()
            → [للكتابة] spi_message لـ CMD24
              → spi_sync(spi, &msg)
                → data bytes على MOSI
                  → SD card تستقبل → ترد بـ data response token
            → [للقراءة] spi_message لـ CMD17
              → سامع على MISO
                → SD card ترسل data block (512 bytes + CRC)
          → mrq->done(mrq)  ← يبلّغ MMC core بالنتيجة
```

---

### 6. الـ API Functions

الـ header بيعرّف function اتنين بس:

```c
/* يجيب الـ platform_data من spi_device */
extern struct mmc_spi_platform_data *mmc_spi_get_pdata(struct spi_device *spi);

/* يرجع الـ platform_data (يحرر resources لو lazy-allocated) */
extern void mmc_spi_put_pdata(struct spi_device *spi);
```

**الـ `mmc_spi_get_pdata`**: في حالة الـ Device Tree، الـ `platform_data` مش بيتحط مباشرةً. فالـ driver بيعمل lazy allocation ويملى الـ struct من DT properties. الـ `get_pdata` بيتكفّل بكده.

**الـ `mmc_spi_put_pdata`**: بترجع الـ resources لو الـ get_pdata كانت عملت allocation.

```
  DT path:
  ────────
  spi_driver.probe()
    → mmc_spi_get_pdata(spi)
        → هل في platform_data جاهز؟
            نعم → return as-is
            لأ  → kzalloc(struct mmc_spi_platform_data)
                  → يقرأ DT properties:
                      "voltage-ranges" → ocr_mask
                      "interrupts"     → detect IRQ
                  → return allocated struct
    → [use pdata]
    → mmc_spi_put_pdata(spi)
        → لو كان allocated من DT → kfree()
```

---

### 7. Locking Strategy

الـ `mmc_spi.h` نفسه ما بيعرّفش locks، لكن الـ driver بيستخدم:

| Lock / Mechanism | بيحمي إيه | مين بيمسكه |
|---|---|---|
| `mmc_host->lock` (spinlock) | الـ host state وقت الـ card detect | MMC core |
| `spi_device` bus lock (ضمني) | الـ SPI bus أثناء الـ transfer | `spi_sync()` بيمسكه تلقائياً |
| `delayed_work` | الـ card detect debounce | kernel workqueue |

**مفيش locking خاص** في الـ `mmc_spi_platform_data` نفسه — الـ fields بتتقرأ مرة واحدة في `probe()` وبعدين ما بتتغيرش. الـ function pointers (`setpower`, `init`, `exit`) بتتسمّى من contexts معروفة وما بتحتاجش locks إضافية.

**Lock Ordering** (لو حصل تداخل):

```
  mmc_host->lock
    └─► SPI bus mutex
          └─► platform GPIO/regulator lock (internal to platform driver)
```

> دايماً ممنوع تعمل SPI transfer وانت شايل الـ `mmc_host->lock` — الـ SPI transfer ممكن تنام (blocking).
## Phase 4: شرح الـ Functions

### ملخص سريع — Cheatsheet

| الـ Symbol | النوع | الغرض |
|---|---|---|
| `mmc_spi_platform_data` | `struct` | Platform-specific configuration لـ MMC/SD slot متصل عبر SPI |
| `init` | function pointer | تهيئة الـ slot وتسجيل الـ card-detect IRQ |
| `exit` | function pointer | تنظيف وفك تسجيل الـ IRQ عند إيقاف الـ driver |
| `caps` / `caps2` | `unsigned long` | MMC core capabilities flags (مثل `MMC_CAP_NEEDS_POLL`) |
| `detect_delay` | `u16` | Debounce time بالـ milliseconds لإشارة card detect |
| `powerup_msecs` | `u16` | Delay بعد تشغيل الـ power (max 250 ms) |
| `ocr_mask` | `u32` | Bitmask للـ voltages المدعومة (OCR register) |
| `setpower` | function pointer | تحكم في الـ power supply للـ card |
| `mmc_spi_get_pdata()` | `extern` function | جلب الـ `mmc_spi_platform_data` من الـ `spi_device` |
| `mmc_spi_put_pdata()` | `extern` function | تحرير/إعادة الـ platform data بعد الاستخدام |

---

### تصنيف الـ Functions والـ Callbacks

#### التصنيف الأول: Platform Data Access Functions

الدالتان `mmc_spi_get_pdata` و `mmc_spi_put_pdata` هما الـ primary API اللي يستخدمها الـ `mmc_spi` driver للوصول للـ board-specific configuration. الفكرة إن بعض الأنظمة بتوفر الـ platform data من خلال الـ Device Tree أو من خلال static `platform_data` مباشرة، فالـ `get/put` pattern بيوفر abstraction layer موحدة.

---

#### التصنيف التاني: Platform Data Callbacks (Function Pointers داخل الـ struct)

دي الـ hooks اللي بيوفرها الـ board code للـ driver عشان يتحكم في الـ hardware-specific behavior: الـ IRQ setup، الـ power control، وتهيئة الـ slot.

---

### شرح الـ Functions بالتفصيل

---

#### `mmc_spi_get_pdata`

```c
extern struct mmc_spi_platform_data *mmc_spi_get_pdata(struct spi_device *spi);
```

**الوظيفة:**
بترجع مؤشر للـ `mmc_spi_platform_data` المرتبطة بالـ `spi_device`. لو الـ board بتستخدم Device Tree، الـ implementation بتبني struct مؤقت وتملاه من الـ DT properties. لو بتستخدم legacy platform data، بترجع `spi->dev.platform_data` مباشرة.

**البارامترات:**
- `spi` — مؤشر للـ `spi_device` اللي بيمثل الـ MMC/SD controller على الـ SPI bus.

**الـ Return Value:**
- مؤشر صالح لـ `mmc_spi_platform_data` عند النجاح.
- `NULL` لو مفيش platform data متاحة (الـ driver ممكن يشتغل بـ defaults).

**Key Details:**
- يُستدعى في `probe()` context قبل أي عملية على الـ MMC host.
- في بيئات DT، الـ implementation بتعمل `kzalloc` للـ struct، فلازم يتم تحريره لاحقاً بـ `mmc_spi_put_pdata`.
- في بيئات non-DT، الـ `put` بتكون no-op لأن الـ data مش dynamically allocated.
- لا يوجد locking صريح — يُفترض إنها تُستدعى في single-threaded `probe` context.

**Caller Context:**
الـ `mmc_spi` driver بيستدعيها في `mmc_spi_probe()` مباشرة بعد الحصول على الـ `spi_device`.

**Pseudocode Flow:**
```
mmc_spi_get_pdata(spi):
    if CONFIG_OF && spi->dev.of_node:
        pdata = kzalloc(sizeof(*pdata))
        // populate from DT: caps, ocr_mask, detect_delay, etc.
        return pdata
    else:
        return spi->dev.platform_data  // may be NULL
```

---

#### `mmc_spi_put_pdata`

```c
extern void mmc_spi_put_pdata(struct spi_device *spi);
```

**الوظيفة:**
بتحرر الـ resources اللي اتخصصت في `mmc_spi_get_pdata`. في بيئات Device Tree بتعمل `kfree` للـ struct المؤقت. في بيئات non-DT، بتكون no-op لأن الـ platform data ملكية الـ board code.

**البارامترات:**
- `spi` — نفس الـ `spi_device` اللي اتبعت لـ `get_pdata`.

**الـ Return Value:**
- `void` — لا ترجع قيمة.

**Key Details:**
- يُستدعى في `mmc_spi_remove()` (الـ `remove` callback للـ driver).
- بالنسبة للـ DT path، بتعمل `kfree(spi->dev.platform_data)` وبتصفر المؤشر لمنع double-free.
- آمنة للاستدعاء حتى لو `get_pdata` رجعت `NULL`.

**Caller Context:**
الـ `mmc_spi` driver بيستدعيها في `mmc_spi_remove()` كـ cleanup step.

---

### شرح الـ Function Pointers داخل `mmc_spi_platform_data`

---

#### `init` — Card Slot Initialization Callback

```c
int (*init)(struct device *dev,
            irqreturn_t (*detect_int)(int, void *),
            void *data);
```

**الوظيفة:**
الـ board code بتوفر الـ implementation دي عشان تعمل setup للـ GPIO المستخدم في card detection وتسجل الـ interrupt handler المقدم من الـ driver. الـ driver بيمرر `detect_int` كـ ISR جاهز، والـ board code بس بتعمل `request_irq` بيه.

**البارامترات:**
- `dev` — الـ `struct device` للـ SPI device، بيُستخدم للـ logging والـ IRQ registration.
- `detect_int` — مؤشر للـ ISR المقدم من الـ `mmc_spi` driver، بيتم تسجيله للـ card-detect GPIO IRQ.
- `data` — opaque pointer بيرجع للـ ISR كـ `dev_id`، عادةً بيكون مؤشر للـ `mmc_host`.

**الـ Return Value:**
- `0` عند النجاح.
- قيمة سالبة (error code) عند الفشل (مثلاً `request_irq` فشلت).

**Key Details:**
- لو الـ board مش بتدعم card-detect IRQ (مثلاً بس polling)، ممكن الـ `init` يكون `NULL` أو بيرجع `0` من غير تسجيل IRQ.
- الـ capability `MMC_CAP_NEEDS_POLL` بتتحدد في `caps` لو مفيش IRQ متاح.
- يُستدعى مرة واحدة في `mmc_spi_probe()`.

**مثال Board Code:**
```c
static int my_board_mmc_init(struct device *dev,
                              irqreturn_t (*detect_int)(int, void *),
                              void *data)
{
    return request_irq(gpio_to_irq(CARD_DETECT_GPIO),
                       detect_int,
                       IRQF_TRIGGER_RISING | IRQF_TRIGGER_FALLING,
                       "mmc-detect", data);
}
```

---

#### `exit` — Card Slot Cleanup Callback

```c
void (*exit)(struct device *dev, void *data);
```

**الوظيفة:**
معكوس الـ `init`، بيعمل `free_irq` للـ card-detect IRQ وأي cleanup تاني مرتبط بالـ slot. بيتم استدعاؤه في `mmc_spi_remove()`.

**البارامترات:**
- `dev` — الـ `struct device` للـ SPI device.
- `data` — نفس الـ opaque pointer اللي اتبعت في `init`.

**الـ Return Value:**
- `void`.

**Key Details:**
- لو `init` كان `NULL`، الـ driver مش بيستدعي `exit`.
- الـ board code لازم تتأكد إن `free_irq` بتتستدعى بنفس الـ `dev_id` اللي اتستخدم في `request_irq`.

---

#### `setpower` — Power Control Callback

```c
void (*setpower)(struct device *dev, unsigned int maskval);
```

**الوظيفة:**
بتتحكم في الـ power supply للـ MMC/SD card. الـ driver بيستدعيها لما يحتاج يغير الـ voltage أو يوقف/يشغل الـ power للـ card.

**البارامترات:**
- `dev` — الـ `struct device` للـ SPI device.
- `maskval` — الـ OCR bitmask للـ voltage المطلوب، أو `0` لإيقاف الـ power تماماً.

**الـ Return Value:**
- `void`.

**Key Details:**
- لو `setpower` كان `NULL`، الـ driver بيفترض إن الـ power دايماً شغال وثابت.
- الـ `maskval` بيتطابق مع الـ bits في `ocr_mask` — الـ board code بتحوله لـ voltage حقيقي (مثلاً `3.3V`).
- بعد تشغيل الـ power، الـ driver بينتظر `powerup_msecs` قبل ما يبدأ الـ initialization sequence.

**مثال Board Code:**
```c
static void my_board_mmc_setpower(struct device *dev, unsigned int maskval)
{
    if (maskval)
        gpio_set_value(CARD_VCC_GPIO, 1);  /* power on */
    else
        gpio_set_value(CARD_VCC_GPIO, 0);  /* power off */
}
```

---

### شرح الـ Configuration Fields

#### `caps` و `caps2`

```c
unsigned long caps;
unsigned long caps2;
```

الـ bitmasks دي بتتنقل مباشرة لـ `mmc_host->caps` و `mmc_host->caps2` في `mmc_spi_probe()`. الـ driver بيعمل OR بين الـ hardware capabilities المكتشفة وبين الـ caps دي. أهم example: `MMC_CAP_NEEDS_POLL` لما مفيش card-detect IRQ.

---

#### `detect_delay`

```c
u16 detect_delay;  /* milliseconds */
```

بيحدد مدة الـ debounce لإشارة الـ card detection. الـ driver بيستخدم `mmc_detect_change(host, msecs_to_jiffies(detect_delay))` لما الـ card-detect IRQ بييجي، عشان يتجنب الـ false triggers من bounce الـ connector.

---

#### `powerup_msecs`

```c
u16 powerup_msecs;  /* max 250 msec */
```

الـ delay اللازمة بعد تشغيل الـ VCC للـ card قبل ما تبدأ الـ SPI communication. الـ MMC spec بتحدد مدة minimum لتثبيت الـ voltage. القيمة `0` معناها استخدام الـ default الخاص بالـ driver.

---

#### `ocr_mask`

```c
u32 ocr_mask;
```

الـ OCR (Operating Conditions Register) bitmask بيحدد الـ voltages المدعومة من الـ board. بيتنقل لـ `mmc_host->ocr_avail`. لو كانت `0`، الـ driver بيستخدم `MMC_VDD_32_33 | MMC_VDD_33_34` كـ default.

---

### العلاقة بين الـ Components — ASCII Diagram

```
Board Code (platform_data)
        |
        | mmc_spi_get_pdata()
        v
 mmc_spi_platform_data
 ┌──────────────────────┐
 │ init()  ──────────────────►  request_irq(card_detect_gpio)
 │ exit()  ──────────────────►  free_irq(card_detect_gpio)
 │ setpower() ───────────────►  GPIO / regulator control
 │ caps / caps2 ─────────────►  mmc_host->caps
 │ ocr_mask ─────────────────►  mmc_host->ocr_avail
 │ detect_delay ─────────────►  mmc_detect_change(..., delay)
 │ powerup_msecs ────────────►  msleep(powerup_msecs)
 └──────────────────────┘
        |
        | mmc_spi_put_pdata()
        v
     cleanup / kfree (DT path)
```

---

### ملاحظات ختامية على الـ Design

الـ header ده صغير لكنه بيمثل نموذج كلاسيكي لـ **platform abstraction في الـ kernel**: الـ driver (`mmc_spi`) بيكون generic ومستقل عن الـ board، والـ board code بتوفر الـ hooks اللي بتتناسب مع الـ hardware الفعلي. الـ `get/put` pattern بيضمن التوافق بين الـ legacy `platform_data` approach والـ modern Device Tree approach من غير ما يكسر الـ existing board code.
## Phase 5: دليل الـ Debugging الشامل

الـ `mmc_spi` driver بيربط الـ MMC/SD card stack بالـ SPI bus — أي مشكلة ممكن تيجي من ٣ مستويات: الـ SPI bus نفسه، الـ MMC protocol فوقيه، أو الـ platform glue اللي بيوصلهم (`mmc_spi_platform_data`). الدليل ده بيغطي الـ ٣ مستويات.

---

### Software Level

#### 1. debugfs Entries

الـ MMC subsystem بيعمل entries تحت `/sys/kernel/debug/mmc*/`:

```bash
# اعرف أي MMC hosts موجودين
ls /sys/kernel/debug/

# مثال لو الـ SPI-attached card ظهر كـ mmc1
ls /sys/kernel/debug/mmc1/

# ios: بيعرض الـ current state (clock, power, bus width, timing mode)
cat /sys/kernel/debug/mmc1/ios

# err_stats: عداد الـ errors من الـ core
cat /sys/kernel/debug/mmc1/err_stats
```

**تفسير ios output:**

```
clock:          400000 Hz      ← initialization clock (25MHz max for SPI mode)
actual clock:   400000 Hz
vdd:            21 (3.3 ~ 3.4 V)
bus mode:       push-pull
chip select:    high            ← CS inactive (بين الـ transactions)
power mode:     on
bus width:      1 bits          ← SPI = always 1-bit
timing spec:    legacy
```

لو `power mode: off` بدون ما تطلب — اتحقق من `setpower()` callback وفولتية الـ OCR.

---

#### 2. sysfs Entries

```bash
# المسار الأساسي للـ SPI device اللي بيشتغل كـ MMC controller
ls /sys/bus/spi/devices/

# مثال: spi0.0 هو الـ mmc_spi device
cat /sys/bus/spi/devices/spi0.0/modalias
# ناتج متوقع: spi:mmc_spi

# الـ MMC host المرتبط بيه
ls /sys/class/mmc_host/
cat /sys/class/mmc_host/mmc1/device/uevent

# الـ card نفسه لو اتعرف عليه
ls /sys/class/mmc_host/mmc1/mmc1:*/

# power state
cat /sys/class/mmc_host/mmc1/power/runtime_status

# الـ caps اللي اتعملت register بها (من mmc_spi_platform_data.caps)
# مش exposed مباشرة، بس شوفها عن طريق
cat /sys/class/mmc_host/mmc1/*/type       # SD, MMC, SDIO
```

---

#### 3. ftrace — Tracepoints والـ Events

```bash
# اعرض الـ available events المتعلقة بالـ MMC والـ SPI
grep -r "mmc\|spi" /sys/kernel/tracing/available_events | head -40

# فعّل الـ MMC core events
echo 1 > /sys/kernel/tracing/events/mmc/enable

# أو events محددة أكثر دقة
echo 1 > /sys/kernel/tracing/events/mmc/mmc_request_start/enable
echo 1 > /sys/kernel/tracing/events/mmc/mmc_request_done/enable

# فعّل الـ SPI tracepoints
echo 1 > /sys/kernel/tracing/events/spi/enable

# شغّل الـ tracing
echo 1 > /sys/kernel/tracing/tracing_on

# اعمل الـ operation اللي عايز تتبعها (مثلاً mount الـ card)
mount /dev/mmcblk1p1 /mnt/card

# اقرأ الـ trace
cat /sys/kernel/tracing/trace | grep -E "mmc|spi"

# وقّف
echo 0 > /sys/kernel/tracing/tracing_on
```

**مثال ناتج ftrace يشرحك الـ flow:**

```
mmc_request_start: mmc1: start struct mmc_request[cmd_opcode=CMD0 ...]
spi_transfer: spi0.0: len=6 tx=[40 00 00 00 00 95] rx=[]
spi_transfer: spi0.0: len=1 tx=[] rx=[01]          ← R1 response: in idle state
mmc_request_done:  mmc1: end struct mmc_request[cmd_opcode=CMD0 err=0]
```

---

#### 4. printk و Dynamic Debug

```bash
# فعّل الـ dynamic debug لكل الـ mmc_spi driver
echo "module mmc_spi +p" > /sys/kernel/debug/dynamic_debug/control

# أو للـ SPI core
echo "module spi +p" > /sys/kernel/debug/dynamic_debug/control

# لو عايز file معين فقط
echo "file drivers/mmc/host/mmc_spi.c +pflmt" > /sys/kernel/debug/dynamic_debug/control
# +p = print, +f = function name, +l = line number, +m = module, +t = thread

# تابع الـ kernel messages
dmesg -w | grep -E "mmc_spi|mmc1|spi0"

# أو من الـ kernel log مع timestamps واضحة
journalctl -k -f | grep -iE "mmc|spi"
```

**أهم رسائل الـ debug اللي بتظهر لما تفعّل dynamic debug:**

```
mmc_spi: INIT: set SPI mode 0, 400kHz
mmc_spi: CMD0 response: R1=0x01 (idle)
mmc_spi: CMD8 response: R7=0x000001AA (v2 card, 2.7-3.6V)
mmc_spi: CMD58 (READ_OCR): OCR=0xC0FF8000
mmc_spi: card OCR=0x40000000, selected
```

---

#### 5. Kernel Config Options للـ Debugging

| Option | الغرض |
|--------|--------|
| `CONFIG_MMC_DEBUG` | يفعّل الـ `dev_dbg()` calls داخل الـ MMC stack |
| `CONFIG_MMC_SPI` | الـ driver نفسه (لازم يكون `=y` أو `=m`) |
| `CONFIG_SPI_DEBUG` | debug للـ SPI core layer |
| `CONFIG_DEBUG_FS` | بيخلي `/sys/kernel/debug/mmc*` موجودة |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل `echo "module mmc_spi +p"` |
| `CONFIG_SPI_SPIDEV` | يخليك تاخد raw access للـ SPI bus للاختبار |
| `CONFIG_MMC_UNSAFE_RESUME` | لو بتـ debug مشاكل الـ suspend/resume |
| `CONFIG_FAULT_INJECTION` | حقن أخطاء محاكاة للـ MMC errors |
| `CONFIG_LOCKDEP` | يكشف deadlocks في الـ SPI/MMC locks |
| `CONFIG_KASAN` | يكشف memory corruption في الـ buffers |

```bash
# اتحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "CONFIG_MMC|CONFIG_SPI" | grep -v "^#"
```

---

#### 6. أدوات خاصة بالـ Subsystem

**spidev للاختبار المباشر:**

```bash
# لو فيه spidev device (CONFIG_SPI_SPIDEV)
ls /dev/spidev*

# استخدم spi-pipe أو spidev_test من kernel tools
# send CMD0 (GO_IDLE_STATE) يدوياً:
# CMD0 = 0x40, arg=0x00000000, CRC=0x95
python3 -c "
import spidev
spi = spidev.SpiDev()
spi.open(0, 0)
spi.max_speed_hz = 400000
spi.mode = 0
# dummy clocks لـ card initialization
spi.xfer2([0xFF]*10)
# CMD0
resp = spi.xfer2([0x40,0x00,0x00,0x00,0x00,0x95,0xFF])
print('R1:', hex(resp[-1]))
spi.close()
"
```

**mmcli / mmc-utils:**

```bash
# معلومات عن الـ card
mmc info /dev/mmcblk1

# extcsd dump (لـ eMMC)
mmc extcsd read /dev/mmcblk1 | head -50

# اعمل read test
dd if=/dev/mmcblk1 of=/dev/null bs=512 count=100 status=progress
```

---

#### 7. جدول رسائل الـ Error الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|----------------------|--------|-------|
| `mmc_spi: no response to CMD0` | الـ card مش بترد على reset | تحقق من الـ SPI wiring والـ CS، تأكد إن فيه `FF` bytes قبل الـ CMD0 |
| `mmc_spi: bad CRC` | corruption في الـ data أو الـ command | قلّل الـ SPI clock، افحص الـ signal integrity، تحقق من الـ pull-up resistors |
| `mmc_spi: timeout waiting for card response` | الـ card مش بترسل R1 | مشكلة power (اتحقق من `setpower()` و `ocr_mask`) أو الـ card معطوبة |
| `mmc1: error -110 whilst initialising SD card` | timeout كامل أثناء الـ init | `detect_delay` طويل جداً، أو مفيش card فعلاً، أو مشكلة حلقة الـ interrupt |
| `mmc1: card never left idle state` | الـ card مش بتخرج من CMD0 idle | الـ OCR مش متوافق (`ocr_mask` غلط)، أو voltage مش صح |
| `mmc_spi: GPIO polling for cd_irq failed` | فشل detect الـ card عبر الـ IRQ | اتحقق من `init()` callback في `mmc_spi_platform_data` |
| `spi_master spi0: transfer timeout` | الـ SPI transaction علقت | مشكلة في الـ SPI controller driver أو الـ DMA |
| `mmc1: Timeout waiting for hardware interrupt` | الـ card مش بتكمل الـ data transfer | الـ card بطيئة جداً أو مشكلة SPI clock polarity/phase |
| `mmc_spi: unknown data error` | R2 response غير معروف | تعارض في الـ SPI mode (Mode 0 vs Mode 3) |
| `mmc1: card is not responding` | الـ card اتكسرت أو انفصلت | افحص الـ physical connection والـ power supply |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

ضيف دول في `drivers/mmc/host/mmc_spi.c` لما تكون بتـ debug مشكلة صعبة:

```c
/* في mmc_spi_initsequence() — لو الـ card مش بتدخل idle */
static int mmc_spi_initsequence(struct mmc_spi_host *host)
{
    int status;
    /* ... */
    status = mmc_spi_command_send(host, NULL, cmd, 1);
    if (status) {
        /* هنا الـ card فشلت CMD0 */
        WARN_ON(status == -ETIMEDOUT);  /* يطبع stack trace مرة واحدة */
        dev_err(&host->spi->dev,
            "CMD0 failed: status=%d, check SPI wiring\n", status);
    }
    return status;
}

/* في mmc_spi_setup_data_message() — لو الـ DMA buffer مش aligned */
WARN_ON(!IS_ALIGNED((unsigned long)data_buf, sizeof(u32)));

/* في mmc_spi_probe() — تحقق إن الـ platform_data صح */
if (pdata) {
    WARN_ON(pdata->ocr_mask == 0);           /* لازم يكون set */
    WARN_ON(pdata->powerup_msecs > 250);     /* spec: max 250ms */
}

/* لتتبع الـ CS toggle */
static void mmc_cs_off(struct mmc_spi_host *host)
{
    if (unlikely(host->cs_gpiod == NULL)) {
        dump_stack();  /* هيبيّن مين استدعى CS off بدون GPIO */
        return;
    }
    /* ... */
}
```

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State موافق لـ Kernel State

الـ kernel بيشوف الـ card من خلال الـ `mmc_spi_platform_data` — لو في اختلاف بين الـ real hardware والـ platform_data، هيفشل صامت:

```bash
# اتحقق إن الـ voltage اللي بتدّيه للـ card موجود في ocr_mask
# OCR bit 20 = 3.2-3.3V, bit 21 = 3.3-3.4V
# مثال: ocr_mask=0x00FF8000 = 2.7V-3.6V range

# اتحقق من الـ actual voltage على الـ VDD pin بالـ multimeter
# لازم يكون ثابت ≥ 2.7V أثناء الـ operation

# اتحقق من الـ SPI mode المستخدم — MMC/SD بيشتغل على Mode 0 أو Mode 3 فقط
# الـ CPOL=0, CPHA=0 (Mode 0) هو الأشيع
cat /sys/bus/spi/devices/spi0.0/of_node/spi-cpol  # لازم مش موجود (default Mode 0)
cat /sys/bus/spi/devices/spi0.0/of_node/spi-cpha  # نفس الكلام

# الـ max clock في SPI mode: 25MHz للـ SD, 20MHz للـ MMC (في init: max 400kHz)
cat /sys/bus/spi/devices/spi0.0/of_node/spi-max-frequency
```

---

#### 2. Register Dump Techniques

**عن طريق devmem2 (لو عندك physical address للـ SPI controller):**

```bash
# مثال: Raspberry Pi SPI0 base = 0x3F204000 (BCM2835)
# اقرأ الـ SPI CS register (offset 0x00)
devmem2 0x3F204000 w
# CS register bits: DONE, RXD, TXD, TA (Transfer Active), CSPOL, CLEAR, CPHA, CPOL, CS

# اقرأ الـ CLK register (offset 0x08)
devmem2 0x3F204008 w
# CDIV: clock divider — CDIV=64 → 250MHz/64 = ~3.9MHz

# اقرأ الـ DLEN (data length)
devmem2 0x3F204010 w

# الـ FIFO (TX/RX)
devmem2 0x3F204004 w
```

**عن طريق /dev/mem (لو CONFIG_STRICT_DEVMEM مش مفعّل):**

```bash
python3 -c "
import mmap, os, struct
fd = os.open('/dev/mem', os.O_RDONLY | os.O_SYNC)
SPI_BASE = 0x3F204000
mem = mmap.mmap(fd, 0x1000, offset=SPI_BASE)
cs_reg = struct.unpack('<I', mem[0:4])[0]
print(f'CS register: 0x{cs_reg:08X}')
mem.close()
os.close(fd)
"
```

**عن طريق io utility من package `iotools`:**

```bash
io -4 -r 0x3F204000   # قرا 32-bit register
```

---

#### 3. Logic Analyzer / Oscilloscope

**الإشارات اللي تراقبها:**

| Signal | الـ Pin | اللي تشوفه |
|--------|---------|------------|
| `SCLK` | SPI Clock | Idle low (Mode 0)، frequency ثابتة |
| `MOSI` | Master Out | الـ command bytes بتتبعت صح |
| `MISO` | Master In | الـ R1 response = `0x01` في idle |
| `CS` | Chip Select | Active low، لازم يبقى low طول الـ transaction |
| `VDD` | Power | ثابتة ±50mV، مش بتـ droop أثناء transfer |

**Setup لـ Logic Analyzer (مثلاً Saleae):**

```
Protocol Decoder: SPI
  → CPOL: 0 (clock idle low)
  → CPHA: 0 (sample on rising edge)
  → Bit order: MSB first
  → Bits per transfer: 8

ابحث عن هذا الـ pattern أثناء الـ init:
  1. 74+ rising edges على SCLK مع CS=HIGH (dummy clocks)
  2. CS goes LOW
  3. MOSI: 40 00 00 00 00 95  ← CMD0
  4. MISO: FF FF FF 01         ← R1 = 0x01 (idle)
  5. CS goes HIGH
```

**Oscilloscope tips:**

- قيس الـ rise time على SCLK — لازم يكون < 20ns عند 1MHz، لو أبطأ فيه مشكلة في الـ pull-up resistors
- فيه ringing أو overshoot فوق VDD؟ → قلّل طول الـ traces أو ضيف series resistor (22-47Ω)
- الـ MISO بيجي `0xFF` طول الوقت؟ → الـ card مش موجودة أو مش بتـ respond

---

#### 4. Hardware Issues الشائعة وـ Kernel Log Patterns

| المشكلة | الـ Pattern في dmesg | السبب المحتمل |
|---------|---------------------|---------------|
| Card مش بترد | `timeout waiting for card response` | CS مش واصل للـ card، أو VDD مش كافي |
| CRC errors متكررة | `bad CRC on CMD` أو `data CRC error` | Clock سريع جداً، trace طويلة، مفيش decoupling capacitor |
| Card بتتعرف ومش بتتعرف | `mmc1: card never left idle state` (intermittent) | Loose connection أو `detect_delay` قصير جداً |
| Voltage مش صح | `mmc1: card is not responding` بعد power on | الـ `ocr_mask` مش بيتطابق مع الـ actual VDD |
| Card بتتعرف بس read/write بتفشل | `mmc1: Timeout waiting for hardware interrupt` | SPI clock polarity/phase غلط (Mode 0 vs Mode 3) |
| Boot delay طويل جداً | الـ init بياخد ≥ 5 ثواني | `powerup_msecs` مبالغ فيها، أو الـ polling loop متأخرة |

---

#### 5. Device Tree Debugging

الـ `mmc_spi_platform_data` في الـ boards الحديثة بيتبني من الـ DT:

```bash
# اتحقق من الـ DT node الخاص بالـ mmc_spi
dtc -I fs -O dts /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "mmc-spi\|mmc_spi"

# مثال DT node صح:
# &spi0 {
#     mmc-slot@0 {
#         compatible = "mmc-spi-slot";
#         reg = <0>;
#         spi-max-frequency = <25000000>;
#         voltage-ranges = <3200 3400>;   ← بيتحول لـ ocr_mask
#         cd-gpios = <&gpio 17 GPIO_ACTIVE_LOW>;  ← card detect
#         wp-gpios = <&gpio 18 GPIO_ACTIVE_HIGH>; ← write protect
#     };
# };

# اتحقق من الـ cd-gpio state
gpioinfo | grep -A2 "GPIO 17"

# اتحقق من الـ voltage-ranges اتترجمت صح
# لازم يتطابق مع الـ capabilities في dmesg:
dmesg | grep "mmc1.*ocr\|mmc1.*volt" -i
```

**اتحقق إن الـ DT يتطابق مع الـ hardware:**

```bash
# قارن الـ spi-max-frequency في DT مع الـ actual clock
cat /sys/bus/spi/devices/spi0.0/of_node/spi-max-frequency | xxd
# لازم القيمة = 25000000 (0x017D7840) أو أقل

# اتحقق من الـ compatible string
cat /sys/bus/spi/devices/spi0.0/of_node/compatible
# المتوقع: mmc-spi-slot
```

---

### Practical Commands — Ready to Copy

#### الـ Quick Diagnosis Script

```bash
#!/bin/bash
# mmc_spi_debug.sh — تشخيص سريع لـ mmc_spi

echo "=== MMC Hosts ==="
ls /sys/class/mmc_host/ 2>/dev/null || echo "No MMC hosts found"

echo -e "\n=== SPI MMC Devices ==="
for d in /sys/bus/spi/devices/*/; do
    modal=$(cat "$d/modalias" 2>/dev/null)
    [[ "$modal" == *"mmc"* ]] && echo "$d -> $modal"
done

echo -e "\n=== MMC IOS State ==="
for ios in /sys/kernel/debug/mmc*/ios; do
    echo "--- $ios ---"
    cat "$ios" 2>/dev/null
done

echo -e "\n=== Recent MMC/SPI kernel messages ==="
dmesg | grep -E "mmc|spi" | tail -30

echo -e "\n=== MMC Error Stats ==="
for err in /sys/kernel/debug/mmc*/err_stats; do
    echo "--- $err ---"
    cat "$err" 2>/dev/null
done
```

```bash
chmod +x mmc_spi_debug.sh && ./mmc_spi_debug.sh
```

---

#### تفعيل الـ Full Debug Mode

```bash
# خطوة ١: فعّل dynamic debug
echo "module mmc_spi +pflmt" > /sys/kernel/debug/dynamic_debug/control
echo "module mmc_core +p" > /sys/kernel/debug/dynamic_debug/control

# خطوة ٢: فعّل ftrace events
echo 1 > /sys/kernel/tracing/events/mmc/enable
echo 1 > /sys/kernel/tracing/events/spi/enable
echo 1 > /sys/kernel/tracing/tracing_on

# خطوة ٣: اعمل eject وinsert للـ card (أو trigger الـ detection)
echo 0 > /sys/class/mmc_host/mmc1/power/runtime_suspended  # wake up

# خطوة ٤: اجمع الـ results
dmesg > /tmp/mmc_dmesg.txt
cat /sys/kernel/tracing/trace > /tmp/mmc_trace.txt
echo 0 > /sys/kernel/tracing/tracing_on

echo "Results in /tmp/mmc_dmesg.txt and /tmp/mmc_trace.txt"
```

---

#### تفسير الـ R1 Response Byte

```bash
# R1 response هو أول non-0xFF byte الـ card بتبعته
# لو R1 = 0x00 → card ready (normal operation)
# لو R1 = 0x01 → in idle state (during init, expected)
# لو R1 = 0x04 → illegal command
# لو R1 = 0x05 → CRC error
# لو R1 = 0x08 → erase sequence error
# لو R1 = 0xFF → card not responding (timeout)

python3 -c "
r1 = 0x05  # غيّر القيمة دي
bits = {
    0: 'in-idle-state',
    1: 'erase-reset',
    2: 'illegal-command',
    3: 'com-crc-error',
    4: 'erase-sequence-error',
    5: 'address-error',
    6: 'parameter-error',
}
print(f'R1=0x{r1:02X}:')
for bit, name in bits.items():
    if r1 & (1 << bit):
        print(f'  bit{bit}: {name} SET')
"
```

---

#### اختبار الـ setpower() Callback

```bash
# محاكاة cycle الـ power (يستدعي setpower() callback)
echo "Cycling MMC power..."

# power off
echo 0 > /sys/class/mmc_host/mmc1/power/control  # أو عن طريق runtime PM
sleep 1

# power on
echo "on" > /sys/class/mmc_host/mmc1/power/control
sleep 0.3  # powerup_msecs

# تحقق من الـ result
dmesg | tail -20 | grep -E "mmc|spi"
cat /sys/kernel/debug/mmc1/ios
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: SD Card مش بتتعرف على Industrial Gateway بـ STM32MP1

#### العنوان
**SD Card detection فاشلة على custom industrial gateway بيستخدم SPI bus**

#### السياق
شركة بتعمل industrial IoT gateway بـ STM32MP1. الـ SD card مربوطة على SPI1 بدل SDMMC لأن الـ pins الأصلية بتتعارض مع Ethernet controller. المنتج بيشتغل بـ Linux 6.1، والـ bootloader شغال تمام، بس بعد ما kernel بيبدأ، الـ card مش بتتعرف خالص.

#### المشكلة
الـ driver `mmc_spi` بيتحمل، بس ما بيلاقيش card. الـ dmesg بيقول:
```
mmc_spi spi1.0: SD card absent or removed
```
مع إن الـ card موجودة فعلاً.

#### التحليل
**الـ `mmc_spi_platform_data`** بتحدد كيفية setup الـ card detect. الـ struct فيه:

```c
struct mmc_spi_platform_data {
    int (*init)(struct device *,
        irqreturn_t (*)(int, void *),
        void *);
    /* ... */
    unsigned long caps;  /* MMC_CAP_NEEDS_POLL لو مفيش CD IRQ */
    u16 detect_delay;    /* debounce بالـ msecs */
};
```

الـ engineer راح يشوف إن الـ `caps` مفيهاش `MMC_CAP_NEEDS_POLL`، وفي نفس الوقت الـ `init` callback بـ `NULL` — يعني ما حصلش تسجيل لأي CD IRQ. النتيجة: الـ MMC core ما عمل poll ولا اتبعت IRQ، فالـ card اتعاملت كأنها غير موجودة.

#### الحل
في الـ platform data (أو device tree عن طريق `mmc-spi-slot`):

```c
/* في board file أو عن طريق DT */
static struct mmc_spi_platform_data my_mmc_spi_info = {
    .caps = MMC_CAP_NEEDS_POLL,  /* polling لأنه مفيش CD IRQ */
    .detect_delay = 100,         /* 100ms debounce */
    .ocr_mask = MMC_VDD_32_33 | MMC_VDD_33_34,
};
```

أو في DT:

```dts
&spi1 {
    mmc-slot@0 {
        compatible = "mmc-spi-slot";
        reg = <0>;
        spi-max-frequency = <25000000>;
        voltage-ranges = <3300 3300>;
        /* مفيش cd-gpios، فـ polling هيتفعل أوتوماتيك */
    };
};
```

الـ `mmc_spi` driver بيشوف لو `cd-gpios` مش موجود وبيضيف `MMC_CAP_NEEDS_POLL` تلقائياً في بعض versions، بس الأضمن تحديده صراحةً.

#### الدرس المستفاد
لو مفيش hardware CD line، لازم `MMC_CAP_NEEDS_POLL` في الـ `caps`. الـ `init` callback ضروري بس لو في IRQ-based detection فعلي — بدونه لازم polling.

---

### السيناريو الثاني: SD Card بتتعرف وبعدين بتختفي على Allwinner H616 TV Box

#### العنوان
**Card بتشتغل بعد boot وبعدين بتوقف في Android TV Box بـ Allwinner H616**

#### السياق
منتج Android TV box بـ Allwinner H616. الـ SDMMC1 الأصلي بيستخدمه eMMC، فالتصميم ربط microSD على SPI0. بعد boot بفترة حوالي 5-10 دقايق، الـ card بتختفي من النظام فجأة.

#### المشكلة
```
mmcblk1: error -110 transferring data
mmc1: card never left busy state
mmc_spi spi0.0: timeout error
```
بعدين:
```
mmc1: card removed
```

#### التحليل
الـ `mmc_spi_platform_data` فيها:

```c
u16 powerup_msecs;   /* delay بعد power up — max 250ms */
u32 ocr_mask;        /* الـ voltages المتاحة */
void (*setpower)(struct device *, unsigned int maskval);
```

الـ `setpower` callback مسؤول عن تشغيل وإيقاف الـ VCC للـ card. المشكلة: الـ regulator المربوط على `setpower` بيتوقف بعد فترة بسبب `regulator_autosuspend`. الـ `powerup_msecs` كانت `0` — يعني بعد ما regulator اشتغل تاني ما فيش delay كافي قبل أول command.

```c
/* المشكلة: powerup_msecs = 0 */
static struct mmc_spi_platform_data h616_mmc_spi = {
    .setpower    = h616_mmc_setpower,  /* بيتحكم في regulator */
    .powerup_msecs = 0,  /* خطأ! card محتاجة وقت تستقر */
    .ocr_mask    = MMC_VDD_32_33 | MMC_VDD_33_34,
};
```

#### الحل
```c
static struct mmc_spi_platform_data h616_mmc_spi = {
    .setpower      = h616_mmc_setpower,
    .powerup_msecs = 250,  /* max allowed — إدي الـ card وقت */
    .ocr_mask      = MMC_VDD_32_33 | MMC_VDD_33_34,
    .caps          = MMC_CAP_NEEDS_POLL,
    .detect_delay  = 200,
};
```

وكمان تعطيل `regulator_autosuspend` على الـ regulator اللي بيغذي الـ card:
```bash
# تحقق من الـ regulator state
cat /sys/class/regulator/regulator.X/state
```

#### الدرس المستفاد
الـ `powerup_msecs` مش مجرد تفصيلة — الـ SD spec بتطلب up to 250ms بعد Vcc قبل أول command. صيفرها بـ `0` وانتظر مشاكل intermittent صعبة تتصاد.

---

### السيناريو الثالث: Kernel Panic عند إزالة SD Card على IoT Sensor بـ AM62x

#### العنوان
**NULL pointer dereference لما card بتتشال وهي شغالة على AM62x IoT node**

#### السياق
IoT sensor node صناعي بـ TI AM62x. الـ MCU بيسجل sensor data على SD card عبر SPI2. المنتج المفروض يتحمل hot-plug للـ card. بس كل ما حد بيشيل الـ card والنظام شغال، الـ kernel بيعمل panic.

#### المشكلة
```
BUG: kernel NULL pointer dereference, address: 0000000000000000
RIP: mmc_spi_cleanup+0x23/0x40
```

#### التحليل
الـ `mmc_spi_platform_data` بتعرف:

```c
void (*exit)(struct device *, void *);
```

الـ `exit` callback المفروض يعمل cleanup للـ resources اللي `init` حجزها — تحديداً الـ IRQ line للـ card detect. في الـ board code:

```c
static int am62x_mmc_spi_init(struct device *dev,
                               irqreturn_t (*detect_func)(int, void *),
                               void *data)
{
    int ret;
    /* حجز IRQ للـ CD pin */
    ret = request_irq(gpio_to_irq(CD_GPIO), detect_func,
                      IRQF_TRIGGER_BOTH, "mmc-cd", data);
    return ret;
}

static void am62x_mmc_spi_exit(struct device *dev, void *data)
{
    /* المبرمج نسي يعمل free_irq هنا! */
}
```

لما card بتتشال، الـ IRQ بييجي، بس الـ `data` pointer اللي اتبعت لـ `request_irq` بقى stale. الـ `exit` ما عملتش `free_irq`، فـ IRQ لسه registered بـ pointer لـ memory اتحررت.

#### الحل
```c
static void am62x_mmc_spi_exit(struct device *dev, void *data)
{
    /* لازم free_irq بنفس الـ dev_id اللي اتبعتت في request_irq */
    free_irq(gpio_to_irq(CD_GPIO), data);
    /* لو في GPIO حجزناه، نحرره هنا */
    gpio_free(CD_GPIO);
}
```

والـ init:
```c
static struct mmc_spi_platform_data am62x_mmc_spi = {
    .init         = am62x_mmc_spi_init,
    .exit         = am62x_mmc_spi_exit,  /* لازم يكون symmetric مع init */
    .detect_delay = 100,
    .caps         = 0,  /* IRQ-based، مش polling */
    .ocr_mask     = MMC_VDD_32_33 | MMC_VDD_33_34,
};
```

#### الدرس المستفاد
الـ `init` والـ `exit` لازم يبقوا symmetric تماماً — كل resource اتحجزت في `init` لازم تتحرر في `exit`. الـ driver مش بيعمل cleanup تلقائي للـ resources اللي البورد code حجزها.

---

### السيناريو الرابع: SD Card شغالة بس بطيئة جداً على i.MX8 Automotive ECU

#### العنوان
**Throughput منخفض جداً لـ SD card على SPI في automotive ECU بـ i.MX8**

#### السياق
Automotive ECU بـ NXP i.MX8M Mini بيسجل vehicle data على microSD عبر SPI3 بسرعة `25 MHz`. الـ card بتشتغل تمام وبتتعرف، بس الـ write speed حوالي 200 KB/s بس — المفروض تكون أكتر من 1.5 MB/s على SPI بالسرعة دي.

#### المشكلة
الـ card شغالة، بس الـ performance مش مقبول للـ data logging application.

#### التحليل
الـ `mmc_spi_platform_data` فيها `caps` و `caps2`:

```c
unsigned long caps;   /* MMC capabilities — بيتبعت لـ mmc_host */
unsigned long caps2;  /* extended capabilities */
```

الـ engineer بيشوف إن الـ `caps` فيها `MMC_CAP_NEEDS_POLL` بس. مفيش:
- `MMC_CAP_SPI` — اتبعتلو تلقائياً من الـ mmc_spi driver
- `MMC_CAP_SD_HIGHSPEED`

بس المشكلة الحقيقية: الـ SPI controller configured بـ `spi-max-frequency = <1000000>` في DT — يعني 1 MHz بس، مش 25 MHz!

```dts
/* الخطأ في DT */
&spi3 {
    mmc-slot@0 {
        compatible = "mmc-spi-slot";
        reg = <0>;
        spi-max-frequency = <1000000>;  /* 1 MHz فقط — خطأ! */
        voltage-ranges = <3300 3300>;
    };
};
```

وبما إن الـ `ocr_mask` في الـ platform data مش بيحدد الـ frequency، والـ frequency بتيجي من `spi-max-frequency` في DT — تغييرها أهم من أي `caps`.

#### الحل
```dts
&spi3 {
    mmc-slot@0 {
        compatible = "mmc-spi-slot";
        reg = <0>;
        spi-max-frequency = <25000000>;  /* 25 MHz */
        voltage-ranges = <3300 3300>;
        cd-gpios = <&gpio4 5 GPIO_ACTIVE_LOW>;
    };
};
```

وفي الـ platform data لو في board file:
```c
static struct mmc_spi_platform_data imx8_mmc_spi = {
    .caps     = MMC_CAP_NEEDS_POLL,
    /* ocr_mask بيحدد الـ voltage، مش الـ speed */
    .ocr_mask = MMC_VDD_32_33 | MMC_VDD_33_34,
    .detect_delay = 100,
    .powerup_msecs = 100,
};
```

للتحقق:
```bash
# شوف الـ SPI clock frequency الفعلية
cat /sys/kernel/debug/spi/spi3/statistics
# أو
dmesg | grep "mmc_spi\|SPI"
```

#### الدرس المستفاد
الـ `mmc_spi_platform_data` مسؤولة عن capabilities والـ power، بس الـ speed بتيجي من الـ SPI device properties (`spi-max-frequency`). مشاكل الـ performance مش دايماً في الـ MMC platform data — ابدأ بالـ SPI clock config.

---

### السيناريو الخامس: Card Detection عاملة عكسي على RK3562 Custom Board

#### العنوان
**Inverted CD signal على custom RK3562 board بيخلي الـ card دايماً "absent"**

#### السياق
Custom embedded Linux board بـ Rockchip RK3562 للـ industrial HMI. الـ SD card مربوطة على SPI1، والـ CD pin مربوط على GPIO2_A3. الـ PCB designer ربط الـ CD pin بـ pull-up (active high لما card موجودة)، عكس الـ standard اللي بيكون active low.

#### المشكلة
```
mmc_spi spi1.0: no SD/MMC card
```
مع إن الـ card موجودة وتمام.

#### التحليل
الـ `init` callback في `mmc_spi_platform_data` بيسجل الـ CD IRQ:

```c
int (*init)(struct device *,
    irqreturn_t (*)(int, void *),  /* الـ detect handler */
    void *);
```

الـ handler اللي بيتبعت بيتكلم على `mmc_detect_change()` — بس الـ logic بتاعة "card present" بتعتمد على GPIO polarity. في الـ board code:

```c
static int rk3562_mmc_init(struct device *dev,
                            irqreturn_t (*detect)(int, void *),
                            void *data)
{
    /* GPIO2_A3: active HIGH لما card موجودة */
    return request_irq(gpio_to_irq(RK_GPIO(2, 3)),
                       detect,
                       IRQF_TRIGGER_RISING | IRQF_TRIGGER_FALLING,
                       "mmc-cd", data);
}
```

بس لو الـ driver بيقرأ الـ GPIO state مباشرةً عشان يحدد "card present"، والـ polarity معكوسة، هيعتقد إن الـ card absent.

الحل في الـ DT هو استخدام `GPIO_ACTIVE_HIGH` بدل `GPIO_ACTIVE_LOW`:

```dts
/* الخطأ */
cd-gpios = <&gpio2 3 GPIO_ACTIVE_LOW>;  /* غلط لو الـ HW active high */

/* الصح */
cd-gpios = <&gpio2 3 GPIO_ACTIVE_HIGH>;  /* يطابق الـ HW */
```

أو لو الـ detect logic في الـ `init` callback يدوي:

```c
static irqreturn_t rk3562_cd_handler(int irq, void *data)
{
    int val = gpio_get_value(RK_GPIO(2, 3));
    /* val == 1 يعني card موجودة (active high على هذا البورد) */
    /* val == 0 يعني card مشالة */
    /* بلغ الـ MMC core بالتغيير */
    mmc_detect_change(data, msecs_to_jiffies(200));
    return IRQ_HANDLED;
}
```

وفي الـ `mmc_spi_platform_data`:
```c
static struct mmc_spi_platform_data rk3562_mmc_spi = {
    .init         = rk3562_mmc_init,
    .exit         = rk3562_mmc_exit,
    .detect_delay = 200,  /* debounce مناسب */
    .caps         = 0,    /* IRQ-based، مش polling */
    .ocr_mask     = MMC_VDD_32_33 | MMC_VDD_33_34,
    .powerup_msecs = 150,
};
```

للتشخيص السريع:
```bash
# اقرأ الـ GPIO مباشرةً
gpioget gpiochip2 3
# لو بيرجع 1 وهي موجودة — active high، غيّر الـ DT polarity
```

#### الدرس المستفاد
الـ `mmc_spi_platform_data` بتفصل logic الـ detection عن الـ polarity. لما بورد غير standard، الـ `init` callback هو المكان الصح لتصحيح الـ polarity — أو من خلال `GPIO_ACTIVE_HIGH` في DT. دايماً افحص الـ schematic للـ CD pin قبل ما تشخص أي حاجة تانية.
## Phase 7: مصادر ومراجع

### مصادر رسمية — Kernel Documentation

| المصدر | الرابط | الموضوع |
|--------|--------|---------|
| **MMC/SD/SDIO card support** | [docs.kernel.org](https://docs.kernel.org/driver-api/mmc/index.html) | التوثيق الرسمي لـ MMC subsystem |
| **SPI Summary** | [docs.kernel.org](https://docs.kernel.org/next/spi/spi-summary.html) | نظرة عامة على SPI framework في الـ kernel |
| **mmc-spi-slot Device Tree binding** | [kernel.org](https://www.kernel.org/doc/Documentation/devicetree/bindings/mmc/mmc-spi-slot.txt) | الـ DT bindings الخاصة بـ `mmc_spi` driver |
| **SPI Kernel Driver API** | [kernel.org v4.14](https://www.kernel.org/doc/html/v4.14/driver-api/spi.html) | الـ API الكامل لـ SPI framework |
| **MMC Tools** | [docs.kernel.org](https://docs.kernel.org/driver-api/mmc/mmc-tools.html) | أدوات `mmc-utils` لاختبار بطاقات MMC/SD |
| **MMC Dev Attributes** | [docs.kernel.org](https://docs.kernel.org/driver-api/mmc/mmc-dev-attrs.html) | الـ sysfs attributes للأجهزة MMC/SD |

---

### مقالات LWN.net

دي أهم المقالات اللي غطّت تطوير الـ SPI وإضافة دعم MMC عليه:

#### SPI Framework

- **[SPI core](https://lwn.net/Articles/138037/)** — أول مقال غطّى الـ SPI core framework في الـ kernel، بيشرح الـ architecture الأساسية.
- **[simple SPI framework](https://lwn.net/Articles/154425/)** — مقال قديم (2005) بيتكلم عن الـ SPI framework وذكر صريح إن MMC/SD cards ممكن تتوصل عبر SPI.
- **[SPI core refresh](https://lwn.net/Articles/162268/)** — تحديثات جوهرية على الـ SPI core.
- **[spi](https://lwn.net/Articles/146581/)** — patch مبكرة لإضافة SPI support.

#### MMC/SD Subsystem

- **[new driver: MMC framework](https://lwn.net/Articles/7176/)** — الـ patch الأولى اللي أضافت الـ MMC framework للـ kernel (2002).
- **[1/4 MMC layer](https://lwn.net/Articles/82765/)** — تطوير الـ MMC layer لدعم الأجهزة embedded.
- **[Secure Digital (SD) support](https://lwn.net/Articles/126098/)** — إضافة دعم SD cards.
- **[MMC updates](https://lwn.net/Articles/253888/)** — pull request مهمة بيقول صريح: "mostly SDIO and SPI support" — وده اللي جاب `mmc_spi` driver للـ mainline.
- **[mmc: core: add SD4.0 support](https://lwn.net/Articles/642336/)** — إضافة دعم SD 4.0.

#### GPIO و SPI over GPIO

- **[Add dynamic MMC-over-SPI-GPIO driver](https://lwn.net/Articles/290066/)** — driver بيخلّيك تعمل MMC/SD interface على GPIO pins عن طريق SPI، بيكمّل فكرة `mmc_spi`.
- **[Add SPI over GPIO driver](https://lwn.net/Articles/290068/)** — الـ `spi_gpio` driver اللي بيوفر SPI bitbang على GPIO، كتير من الـ boards بتستخدمه مع `mmc_spi`.

---

### Source Files في الـ Kernel

الملفات الأساسية اللي لازم تقراها جنب `mmc_spi.h`:

```
include/linux/spi/mmc_spi.h          ← الـ header موضوع التوثيق ده
drivers/mmc/host/mmc_spi.c           ← الـ driver الرئيسي
drivers/mmc/host/of_mmc_spi.c        ← Device Tree support لـ mmc_spi
Documentation/devicetree/bindings/mmc/mmc-spi-slot.txt  ← DT binding
```

الـ GitHub links للملفات الحية:

- [`drivers/mmc/host/mmc_spi.c`](https://github.com/torvalds/linux/blob/master/drivers/mmc/host/mmc_spi.c)
- [`drivers/mmc/host/of_mmc_spi.c`](https://github.com/torvalds/linux/blob/master/drivers/mmc/host/of_mmc_spi.c)

---

### Kernel Newbies — نقاط مرجعية

- **[Linux 2.6.24](https://kernelnewbies.org/Linux_2_6_24)** — النسخة دي أضافت الـ `mmc_spi` driver للـ mainline kernel (تحت `drivers/mmc/host/mmc_spi.c`). الصفحة بتشرح إن المشكلة الوحيدة كانت الـ overhead المرتفع مقارنةً بـ native MMC controller، لكن الميزة إنه بيشتغل على أي SPI controller.
- **[Linux 2.6.31](https://kernelnewbies.org/Linux_2_6_31)** — تحسينات على الـ MMC subsystem.
- **[Linux 3.15 DriversArch](https://kernelnewbies.org/Linux_3.15-DriversArch)** — تحديثات على MMC/SPI drivers.

---

### eLinux.org — موارد Embedded Linux

- **[ZipIt MMC](https://elinux.org/ZipIt_MMC)** — مثال عملي على توصيل MMC card على SPI، بيشرح الـ signal names من منظور الـ MMC card (DO/DI/CLK/CS).
- **[BeagleBoard/SPI](https://elinux.org/BeagleBoard/SPI)** — تجربة عملية على BeagleBoard لتفعيل SPI ودعم MMC عليه.
- **[BeagleBoard Zippy](https://elinux.org/BeagleBoard_Zippy)** — expansion board بتوفر MMC/SD interface ثانية على SPI.
- **[RPi SPI](https://elinux.org/RPi_SPI)** — إعداد SPI على Raspberry Pi، مفيد لفهم الـ platform configuration.

---

### Mailing List Archives

للبحث في النقاشات التاريخية حول `mmc_spi` و `mmc_spi_platform_data`:

- **[lore.kernel.org — linux-mmc](https://lore.kernel.org/linux-mmc/)** — الأرشيف الرسمي لـ mailing list الخاصة بـ MMC subsystem.
- **[lkml.org](https://lkml.org/)** — الأرشيف العام لـ Linux Kernel Mailing List.
- **[GitHub: linux-mailinglist-archives/linux-mmc](https://github.com/linux-mailinglist-archives/linux-mmc.vger.kernel.org.0)** — أرشيف مرآة على GitHub.

**الـ search terms المقترحة للـ mailing lists:**
```
mmc_spi_platform_data init exit
mmc_spi ocr_mask setpower
MMC over SPI platform_data
mmc_spi_get_pdata of_mmc_spi
```

---

### كتب مقترحة

| الكتاب | الفصول ذات الصلة | الأهمية |
|--------|-----------------|---------|
| **Linux Device Drivers, 3rd Ed. (LDD3)** — Corbet, Rubini, Hartman | Chapter 14: The Linux Device Model, Chapter 6: Advanced Char Driver Operations | مرجع أساسي لفهم الـ `platform_data` و interrupt handling |
| **Linux Kernel Development — Robert Love** | Chapter 14: The Block I/O Layer, Chapter 7: Interrupts and Interrupt Handlers | لفهم الـ IRQ hookup في `mmc_spi_platform_data` |
| **Embedded Linux Primer — Christopher Hallinan** | Chapter 11: BusyBox, Chapter 15: Debugging Embedded Linux | أمثلة عملية على MMC/SD في الـ embedded systems |
| **Essential Linux Device Drivers — Sreekrishnan Venkateswaran** | Chapter 8: Serial Drivers, Chapter 9: PCMCIA and CF | تغطية أعمق للـ serial buses وبروتوكولات الاتصال |

---

### Search Terms للبحث عن معلومات إضافية

لو عايز تعمق أكتر في الموضوع، الـ search terms دي هتساعدك:

```
# الـ driver نفسه
mmc_spi Linux kernel driver
mmc_spi_platform_data init exit irq
MMC SD over SPI Linux embedded

# الـ platform data pattern
linux kernel platform_data SPI device
spi_device platform_data board file
of_mmc_spi device tree

# الـ MMC core integration
mmc_host MMC_CAP_NEEDS_POLL
mmc_add_host SPI MMC
ocr_mask voltage MMC SPI

# الـ card detect
mmc_spi card detect IRQ debounce
MMC_CAP_NEEDS_POLL polling card detect

# الـ power management
mmc_spi setpower powerup_msecs
MMC SPI voltage regulator embedded
```
## Phase 8: Writing simple module

### الفكرة

الـ `mmc_spi.h` بيعرّف دالتين exported:
- `mmc_spi_get_pdata(struct spi_device *spi)` — بترجع الـ `mmc_spi_platform_data` الخاصة بالـ SPI device.
- `mmc_spi_put_pdata(struct spi_device *spi)` — بتحرر الـ platform data.

هنعمل **kprobe** على `mmc_spi_get_pdata` عشان نشوف كل مرة بيحاول فيها الـ MMC-SPI driver يجيب الـ platform data، ونطبع معلومات الـ `spi_device` المرتبطة بيه.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * mmc_spi_probe.c — kprobe on mmc_spi_get_pdata()
 *
 * Logs every call to mmc_spi_get_pdata() with SPI device info.
 */

/* ---------- includes ---------- */
#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit        */
#include <linux/kernel.h>       /* pr_info, KERN_INFO                       */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe, ...      */
#include <linux/spi/spi.h>      /* struct spi_device, max_speed_hz, ...     */
#include <linux/spi/mmc_spi.h>  /* prototype of mmc_spi_get_pdata()         */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("kernel-doc-ar");
MODULE_DESCRIPTION("kprobe demo: trace mmc_spi_get_pdata() calls");

/* ---------- pre-handler ---------- */
/*
 * بيتنادى قبل ما mmc_spi_get_pdata تنفّذ.
 * regs بيحتوي على قيم الـ registers في لحظة الاستدعاء —
 * الـ argument الأول (spi) موجود في rdi على x86-64.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * على x86-64: argument[0] → rdi
     * على ARM64:  argument[0] → regs->regs[0]
     * نستخدم regs_get_kernel_argument() اللي portable عبر architectures.
     */
    struct spi_device *spi =
        (struct spi_device *)regs_get_kernel_argument(regs, 0);

    if (!spi)
        return 0;

    /* nطبع معلومات مفيدة عن الـ SPI device اللي طلب الـ pdata */
    pr_info("mmc_spi_probe: mmc_spi_get_pdata() called\n"
            "  device    : %s\n"
            "  modalias  : %s\n"
            "  max_speed : %u Hz\n"
            "  mode      : 0x%08x\n"
            "  bits/word : %u\n",
            dev_name(&spi->dev),
            spi->modalias,
            spi->max_speed_hz,
            spi->mode,
            spi->bits_per_word);

    return 0; /* 0 = امشي كمّل، -errno = abort execution */
}

/* ---------- post-handler ---------- */
/*
 * بيتنادى بعد ما الدالة ترجع — هنا نطبع قيمة الـ return value
 * عشان نعرف لو الـ pdata اتجابت فعلاً أو رجعت NULL.
 */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /* على x86-64 الـ return value في rax */
    void *ret = (void *)regs_return_value(regs);

    pr_info("mmc_spi_probe: mmc_spi_get_pdata() returned %s\n",
            ret ? "valid pdata" : "NULL (no pdata)");
}

/* ---------- kprobe struct ---------- */
/*
 * symbol_name بيحدد الدالة اللي هنحط فيها الـ probe —
 * الـ kernel بيحوّلها لعنوان وقت الـ register.
 */
static struct kprobe kp = {
    .symbol_name = "mmc_spi_get_pdata",
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

/* ---------- init ---------- */
static int __init mmc_spi_probe_init(void)
{
    int ret;

    /*
     * register_kprobe بيحجز الـ breakpoint في الـ kernel text —
     * لو الدالة مش موجودة أو محمية هيرجع error.
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("mmc_spi_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("mmc_spi_probe: kprobe planted on %s @ %px\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ---------- exit ---------- */
static void __exit mmc_spi_probe_exit(void)
{
    /*
     * لازم نشيل الـ kprobe قبل ما الـ module يتـunload —
     * لو سبناه والـ module اتمسح هيحصل kernel panic
     * لأن الـ breakpoint هيـjump لكود اتمسح من الذاكرة.
     */
    unregister_kprobe(&kp);
    pr_info("mmc_spi_probe: kprobe removed from %s\n", kp.symbol_name);
}

module_init(mmc_spi_probe_init);
module_exit(mmc_spi_probe_exit);
```

---

### شرح كل جزء

#### الـ includes

| Header | ليه موجود |
|--------|-----------|
| `linux/kprobes.h` | بيعرّف `struct kprobe` وكل API الـ kprobe |
| `linux/spi/spi.h` | بيعرّف `struct spi_device` اللي بنقرأ منه البيانات |
| `linux/spi/mmc_spi.h` | بيعرّف prototype الدالة اللي بنراقبها |

الـ includes دي ضرورية عشان الـ compiler يعرف أحجام وأوفست الـ structs اللي بنـaccess فيها.

---

#### الـ `handler_pre`

الـ pre-handler بيتنادى **قبل** تنفيذ الدالة الأصلية، فده وقتنا الصح إننا نقرأ الـ argument. بنستخدم `regs_get_kernel_argument(regs, 0)` بدل ما نـhardcode `regs->rdi` عشان الكود يشتغل على x86-64 وARM64 من غير تعديل.

---

#### الـ `handler_post`

الـ post-handler بيتنادى **بعد** ما الدالة تنفّذ وقبل ما تُرجع للـ caller. بنستخدم `regs_return_value(regs)` عشان نعرف لو الـ platform data اتوجدت فعلاً.

---

#### الـ `struct kprobe`

الـ `symbol_name` بيخلي الـ kernel يحوّل اسم الدالة لعنوانها تلقائياً وقت الـ `register_kprobe` — مش لازم ندور على العنوان يدوياً.

---

#### الـ `module_init` / `module_exit`

- الـ `init` بيسجّل الـ kprobe — لو فشل مش بيـload الـ module خالص.
- الـ `exit` لازم يشيل الـ kprobe **قبل** ما الـ module code يتمسح من الذاكرة، وإلا الـ breakpoint يبقى شايل على عنوان مش موجود ويحصل kernel panic.

---

### Makefile للـ build

```makefile
obj-m += mmc_spi_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### تشغيل وتتبع الـ output

```bash
# تحميل الـ module
sudo insmod mmc_spi_probe.ko

# متابعة الـ log (لو فيه MMC على SPI هتشوف الـ output)
sudo dmesg -w | grep mmc_spi_probe

# تفريغ الـ module
sudo rmmod mmc_spi_probe
```

لو الـ board مفيهاش MMC على SPI، ممكن نـtest يدوي عن طريق كتابة module تاني بيـcall `mmc_spi_get_pdata()` بـ NULL device — الـ kprobe هيشتغل وهيطبع التحذير.
