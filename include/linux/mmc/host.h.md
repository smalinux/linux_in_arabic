## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي له الملف

**الـ MMC/SD/SDIO Subsystem** — المسؤول عنه: Ulf Hansson `<ulf.hansson@linaro.org>`، القائمة البريدية: `linux-mmc@vger.kernel.org`.

---

### الصورة الكبيرة — قبل أي كود

#### القصة من الأول

تخيل إنك عندك **لاب توب** وعايز تحط **SD card** فيه عشان تنقل صور. اللاب توب عنده **slot** للكارت، وده الـ **host controller**. الـ SD card نفسها هي الـ **card**. بين الاتنين في **بروتوكول** بيتكلموا بيه على **data lines** — ده هو الـ **MMC/SD protocol**.

الـ Linux Kernel بيحتاج يدعم آلاف الأجهزة المختلفة: Qualcomm Snapdragon، Raspberry Pi، Samsung Exynos، Rockchip — كل واحد منهم عنده **host controller مختلف**، بس كلهم بيتكلموا مع **SD/eMMC cards** بنفس البروتوكول.

المشكلة: لو كتبت driver لكل device من الصفر، هتكرر نفس الكود مئات المرات. الحل هو الـ **MMC Subsystem** — طبقة وسط **generic** بتوحد كل حاجة.

---

#### الـ `host.h` — إيه دوره تحديداً؟

الملف ده هو **عقد العمل بين الـ MMC Core وأي host controller driver**.

تخيل الـ MMC Core كمدير مشروع بيعمل أوامر زي:
- "ابعت الـ command ده للكارت"
- "غير الـ clock لـ 50 MHz"
- "شوف في كارت ولا لأ"

الـ host controller driver هو المقاول اللي بينفذ. الـ `host.h` هو الـ **interface** — القائمة الرسمية لكل حاجة المقاول لازم يعرف يعملها.

---

### الـ Structs والمفاهيم الأساسية

#### `struct mmc_ios` — إعدادات الحافلة الحالية

ده بيحدد **الحالة الفيزيائية الحالية** للـ bus:

```c
struct mmc_ios {
    unsigned int  clock;        /* clock rate — سرعة الساعة بالـ Hz */
    unsigned char bus_width;    /* 1-bit, 4-bit, 8-bit */
    unsigned char timing;       /* legacy, HS, UHS-I SDR104, HS400, ... */
    unsigned char power_mode;   /* OFF, UP, ON */
    unsigned char signal_voltage; /* 3.3V أو 1.8V أو 1.2V */
};
```

كل مرة الـ Core يعايز يغير سرعة أو voltage، بيعدّل الـ `mmc_ios` وبيبعته للـ driver عن طريق `set_ios()`.

---

#### `struct mmc_host_ops` — قائمة المهام الإلزامية على الـ Driver

ده الجزء الأهم في الملف. ده الـ **vtable** — الجدول اللي فيه كل function pointer لازم (أو ممكن) الـ driver يوفرها:

```c
struct mmc_host_ops {
    /* إرسال request للكارت — إلزامي */
    void (*request)(struct mmc_host *host, struct mmc_request *req);

    /* ضبط إعدادات الـ bus: clock, voltage, width */
    void (*set_ios)(struct mmc_host *host, struct mmc_ios *ios);

    /* هل في كارت في الـ slot؟ */
    int  (*get_cd)(struct mmc_host *host);

    /* هل الكارت read-only؟ */
    int  (*get_ro)(struct mmc_host *host);

    /* تبديل الـ voltage من 3.3V لـ 1.8V */
    int  (*start_signal_voltage_switch)(...);

    /* tuning لتحسين جودة الإشارة على سرعات عالية */
    int  (*execute_tuning)(struct mmc_host *host, u32 opcode);

    /* reset الكارت hardware */
    void (*card_hw_reset)(struct mmc_host *host);
};
```

كل driver زي `sdhci.c` أو `bcm2835.c` بيملي الجدول ده بـ functions خاصة بالـ hardware بتاعته.

---

#### `struct mmc_host` — القلب النابض

ده أكبر struct في الملف وهو **التمثيل الكامل لأي host controller** في الـ kernel:

```c
struct mmc_host {
    struct device        *parent;      /* الـ device في sysfs */
    const struct mmc_host_ops *ops;    /* ops table للـ driver */
    unsigned int         f_min, f_max; /* أقل وأعلى clock مدعوم */
    u32                  caps;         /* capabilities: 4-bit, UHS, DDR, ... */
    u32                  caps2;        /* more caps: HS200, HS400, CQE, crypto */
    struct mmc_ios       ios;          /* الإعدادات الحالية للـ bus */
    struct mmc_card      *card;        /* الكارت المتصل حالياً */
    struct mmc_supply    supply;       /* voltage regulators */
    /* ... والكثير */
    unsigned long        private[];    /* بيانات خاصة بالـ driver */
};
```

الـ `private[]` في الآخر ذكي جداً — الـ driver بيطلب مساحة إضافية عند الـ `mmc_alloc_host()` فتُحجز مباشرة بعد الـ struct، وبيوصلها بـ `mmc_priv(host)`.

---

#### `struct mmc_cqe_ops` — الـ Command Queue Engine

الـ **CQE** هو feature موجود في eMMC 5.1+ بيخلي الكارت يستقبل **32 request في وقت واحد** ويرتبهم هو بنفسه بدل ما تبعتهم واحدة واحدة. الـ `mmc_cqe_ops` هو الـ interface للـ hardware اللي بيدعم CQE:

```c
struct mmc_cqe_ops {
    int  (*cqe_enable)(struct mmc_host *host, struct mmc_card *card);
    int  (*cqe_request)(struct mmc_host *host, struct mmc_request *mrq);
    void (*cqe_recovery_start)(struct mmc_host *host);
    void (*cqe_recovery_finish)(struct mmc_host *host);
    /* ... */
};
```

---

#### الـ Capabilities Flags — إزاي الـ Core يعرف إيه اللي الـ Host يقدر يعمله

الـ driver بيـ set bits في `caps` و`caps2` لما بيـ register نفسه:

| Flag | المعنى |
|------|---------|
| `MMC_CAP_4_BIT_DATA` | يدعم 4-bit bus width |
| `MMC_CAP_8_BIT_DATA` | يدعم 8-bit (eMMC فقط) |
| `MMC_CAP_UHS_SDR104` | يدعم UHS-I SDR104 بـ 208 MB/s |
| `MMC_CAP_NONREMOVABLE` | كارت ثابت زي eMMC |
| `MMC_CAP2_HS200` | يدعم eMMC HS200 بـ 200 MB/s |
| `MMC_CAP2_HS400` | يدعم eMMC HS400 بـ 400 MB/s |
| `MMC_CAP2_CQE` | عنده Command Queue Engine |
| `MMC_CAP2_CRYPTO` | يدعم inline encryption |
| `MMC_CAP2_SD_UHS2` | يدعم SD UHS-II |

---

#### الـ Tuning — ليه موجود؟

على سرعات عالية زي HS200 و SDR104، الـ **timing** بتاع الـ signal بيبقى حساس جداً. تأخير بسيط في الـ clock phase ممكن يخلي الـ data تجي غلط. الـ **tuning** هو عملية بيبعت فيها الـ Core pattern معروف للكارت ويقرأه تاني، ويغير الـ clock phase شوية شوية لحد ما يلاقي أحسن نقطة. بعدين لو الـ temperature تغير أو حصل CRC error، بيعمل **re-tuning** تلقائي.

---

#### UHS-II — الجيل الجديد

**UHS-II** بروتوكول جديد خالص بيستخدم **2 lanes تفاضلية** بدل الـ parallel data lines التقليدية، بسرعات لحد **624 MB/s**. الـ struct `sd_uhs2_caps` بيحفظ capabilities الـ host والكارت في الـ mode ده. الـ `uhs2_control()` callback هو الطريقة الوحيدة اللي بيتكلم بيها الـ Core مع الـ host في الـ UHS-II mode.

---

### قصة الـ Stack بالكامل

```
┌─────────────────────────────────┐
│        User Application         │
│  (cp file.mkv /media/sdcard/)   │
└────────────┬────────────────────┘
             │
┌────────────▼────────────────────┐
│       Block Layer (VFS)         │
└────────────┬────────────────────┘
             │
┌────────────▼────────────────────┐
│    MMC Block Driver             │  ← drivers/mmc/core/block.c
│  (يحول block requests لـ MMC)   │
└────────────┬────────────────────┘
             │
┌────────────▼────────────────────┐
│       MMC Core                  │  ← drivers/mmc/core/core.c
│  (Protocol logic, init, PM)     │
└────────────┬────────────────────┘
             │  struct mmc_host_ops  ← هنا يبدأ دور host.h
┌────────────▼────────────────────┐
│   Host Controller Driver        │  ← مثلاً sdhci.c أو bcm2835.c
│  (يتكلم مع الـ hardware مباشرة) │
└────────────┬────────────────────┘
             │  Registers / DMA
┌────────────▼────────────────────┐
│  Physical Host Controller (HW)  │
│  + SD/eMMC/SDIO Card            │
└─────────────────────────────────┘
```

---

### الملفات المكوّنة للـ Subsystem

#### Headers — `include/linux/mmc/`

| الملف | الوصف |
|-------|-------|
| `host.h` | **هذا الملف** — تعريف `mmc_host`, `mmc_host_ops`, `mmc_ios` |
| `core.h` | تعريف `mmc_request`, `mmc_command`, `mmc_data` |
| `card.h` | تعريف `mmc_card` وكل بيانات الكارت (CID, CSD, EXT_CSD) |
| `mmc.h` | opcodes وأرقام commands لـ eMMC |
| `sd.h` | opcodes وأرقام commands لـ SD |
| `sdio.h` | تعريفات خاصة بـ SDIO |
| `sdio_func.h` | تعريف `sdio_func` للـ SDIO function drivers |
| `pm.h` | power management flags للـ MMC |
| `sd_uhs2.h` | تعريفات UHS-II |
| `slot-gpio.h` | GPIO-based card detect / write protect |

#### Core Logic — `drivers/mmc/core/`

| الملف | الوصف |
|-------|-------|
| `core.c` | منطق الـ initialization والـ request handling |
| `host.c` | `mmc_alloc_host()`, `mmc_add_host()`, `mmc_free_host()` |
| `block.c` | الـ block device interface |
| `sd.c` | إعداد SD cards |
| `mmc.c` | إعداد eMMC cards |
| `sdio.c` | إعداد SDIO cards (غير ظاهر هنا) |
| `regulator.c` | إدارة الـ voltage regulators |
| `crypto.c` | دعم inline encryption لـ eMMC |
| `pwrseq_emmc.c` | power sequence خاص بـ eMMC |

#### Host Drivers مهمة — `drivers/mmc/host/`

| الملف | الوصف |
|-------|-------|
| `sdhci.c` | الـ generic SDHCI driver — الأكثر استخداماً |
| `sdhci-pci-core.c` | SDHCI على PCI (لاب توب) |
| `sdhci-of-arasan.c` | SDHCI على Xilinx/Zynq |
| `sdhci-msm.c` | Qualcomm Snapdragon |
| `bcm2835.c` | Raspberry Pi |
| `dw_mmc.c` | DesignWare MMC — Rockchip/Exynos |
| `cqhci-core.c` | Command Queue Host Controller |
| `mmc_spi.c` | MMC عبر SPI |
## Phase 2: شرح الـ MMC/SD Host Framework

---

### المشكلة اللي بيحلها الـ Subsystem ده

في أي نظام embedded، في كارت storage — SD card أو eMMC أو SDIO —  بيتوصل بالـ SoC عن طريق **host controller** معين. كل controller ليه registers مختلفة، DMA مختلفة، وحتى timing مختلفة.

لو مفيش abstraction، كل driver عايز يكلم SD card هيتعامل مع hardware مباشرةً — يعني كود متكرر، fragile، ومستحيل يتنقل بين platforms.

**المشاكل تحديداً:**
1. **Hardware diversity** — SDHCI, Rockchip, i.MX, MediaTek، كل واحد controller مختلف تماماً.
2. **Card diversity** — SD, SDHC, SDXC, eMMC, SDIO، كل نوع بروتوكول مختلف.
3. **Multiple consumers** — block layer عايز يقرا/يكتب، Wi-Fi chip (SDIO) عايز IRQ handling، crypto subsystem عايز inline encryption.
4. **Power management** — voltage switching (3.3V → 1.8V)، runtime PM، وكمان card detection.

---

### الحل اللي بياخده الـ Kernel

الـ MMC subsystem بيحط **layer واحدة** بين:
- الـ **core logic** (بروتوكول التواصل مع الكارت نفسه)
- الـ **host driver** (hardware-specific implementation)

الـ core بيعرف ازاي يعمل card initialization، command sequencing، error recovery — بس مش بيعرف حاجة عن الـ registers.

الـ host driver بيعرف الـ registers بالظبط — بس مش بيفهم في بروتوكول eMMC.

الحلقة الوصل بينهم هي `struct mmc_host` + `struct mmc_host_ops`.

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Upper Layers (Consumers)                │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────┐ │
│  │  Block Layer │  │  SDIO Driver │  │  MMC Crypto Layer  │ │
│  │(mmc_blk.c)  │  │(sdio_bus.c)  │  │(inline encryption) │ │
│  └──────┬──────┘  └──────┬───────┘  └─────────┬──────────┘ │
└─────────┼────────────────┼──────────────────────┼───────────┘
          │                │                      │
          ▼                ▼                      ▼
┌─────────────────────────────────────────────────────────────┐
│                    MMC Core Layer                           │
│                                                             │
│   mmc_request ──► mmc_command ──► mmc_data ──► scatterlist │
│                                                             │
│   ┌──────────────────────────────────────────────────────┐ │
│   │              struct mmc_host                         │ │
│   │  ops ──► mmc_host_ops (callbacks to HW driver)       │ │
│   │  ios ──► mmc_ios     (current bus state)             │ │
│   │  card ──► mmc_card   (attached device)               │ │
│   │  cqe_ops ──► mmc_cqe_ops (Command Queue Engine)      │ │
│   └──────────────────────────────────────────────────────┘ │
└──────────────────────────┬──────────────────────────────────┘
                           │ ops callbacks
          ┌────────────────┼───────────────────┐
          ▼                ▼                   ▼
┌──────────────┐  ┌──────────────┐   ┌──────────────────┐
│  sdhci.c     │  │ mmc-bcm2835  │   │  sdhci-tegra.c   │
│ (generic     │  │ (RPi-specific│   │  (Tegra-specific) │
│  SDHCI)      │  │  controller) │   │                  │
└──────────────┘  └──────────────┘   └──────────────────┘
          │                │                   │
          ▼                ▼                   ▼
    ┌──────────────────────────────────────────┐
    │           Physical Hardware              │
    │    SD/eMMC/SDIO Card or Chip             │
    └──────────────────────────────────────────┘
```

---

### الـ Core Abstraction — إيه الفكرة المحورية؟

الفكرة المحورية هي: **الـ `struct mmc_host` هو العقد بين الـ core والـ hardware driver**.

الـ core بيقول "أنا عايز أبعت command"، ويملي `mmc_request`.
الـ host driver بياخد الـ request ده ويترجمه لـ register writes.
النتيجة ترجع في نفس الـ `mmc_request`.

كل الـ state الخاص بالبوص (voltage، clock، width) محفوظ في `mmc_ios` جوه الـ `mmc_host`.
لما الـ core يعمل تغيير، بيعدّل `ios` وبيكال `ops->set_ios()` — الـ driver هو اللي يطبّق التغيير فعلياً.

---

### Analogy عميقة — مطار دولي

تخيل مطار فيه:
- **Runway (المدرج)** = الـ physical bus (الأسلاك، الـ clock، الـ voltage)
- **Air Traffic Control (برج المراقبة)** = الـ MMC Core
- **Aircraft (الطيارة)** = الـ mmc_request
- **Ground Crew (طاقم الأرض)** = الـ host driver
- **Flight Plan** = الـ mmc_ios (إيه الـ speed mode، إيه الـ voltage، عرض الـ bus)
- **Passenger** = البيانات اللي في الـ scatterlist
- **Plane Model (بوينج/إيرباص)** = نوع الكارت (SD vs eMMC vs SDIO)

| عنصر المطار | المقابل في الكود |
|---|---|
| برج المراقبة بيقرر الـ runway | الـ core بيقرر الـ timing mode وبيعدّل `mmc_ios` |
| Ground crew بيحضّر المدرج | الـ host driver بيكتب الـ registers بناءً على `set_ios()` |
| الطيارة بتقلع/تهبط | الـ `mmc_request` بيتبعت عبر `ops->request()` |
| Flight plan مش بيتغير وهي جو | الـ `mmc_ios` ثابت أثناء الـ transfer |
| برج المراقبة مش بيلمس المحرك | الـ core مش بيلمس hardware registers أبداً |
| Ground crew مش بيقرر routes | الـ host driver مش بيفهم في بروتوكول eMMC |
| Tuning (تعديل المسار) قبل الهبوط | الـ `execute_tuning()` callback لتحسين الـ sampling window |

---

### الـ Structs الأساسية وعلاقاتها

```
struct mmc_host
│
├── const struct mmc_host_ops *ops        ← vtable للـ hardware driver
│     ├── request()                       ← إرسال mmc_request للهاردوير
│     ├── set_ios()                       ← تطبيق إعدادات mmc_ios
│     ├── get_cd()                        ← هل في كارت؟
│     ├── get_ro()                        ← read-only؟
│     ├── execute_tuning()                ← ضبط الـ sampling
│     ├── start_signal_voltage_switch()   ← 3.3V → 1.8V
│     └── card_hw_reset()                ← reset عبر RST_n pin
│
├── struct mmc_ios ios                    ← state الحالي للـ bus
│     ├── clock                           ← تردد الـ clock الحالي
│     ├── vdd                             ← voltage بيت mask
│     ├── bus_width                       ← 1/4/8 bits
│     ├── timing                          ← LEGACY/HS/HS200/HS400/UHS
│     ├── signal_voltage                  ← 3.3V / 1.8V / 1.2V
│     └── power_mode                      ← OFF / UP / ON
│
├── struct mmc_card *card                 ← الكارت المتوصل
│     ├── struct mmc_host *host           ← back-pointer للـ host
│     ├── struct mmc_cid cid              ← Card Identification Data
│     ├── struct mmc_csd csd              ← Card Specific Data
│     ├── struct mmc_ext_csd ext_csd      ← eMMC v4+ extended registers
│     └── type (MMC/SD/SDIO/SD_COMBO)
│
├── const struct mmc_cqe_ops *cqe_ops    ← Command Queue Engine ops
│     ├── cqe_enable() / cqe_disable()
│     ├── cqe_request()                  ← submit tagged request
│     ├── cqe_timeout()
│     └── cqe_recovery_start/finish()
│
├── struct mmc_supply supply             ← regulators
│     ├── vmmc   ← card power (3.3V)
│     └── vqmmc  ← IO voltage (3.3V or 1.8V)
│
└── unsigned long private[]             ← driver-private data (cacheline aligned)
```

```
struct mmc_request (mrq)
│
├── struct mmc_command *sbc     ← SET_BLOCK_COUNT (CMD23) - optional
├── struct mmc_command *cmd     ← الأمر الرئيسي
│     ├── opcode                ← رقم الـ command
│     ├── arg                   ← الـ argument
│     ├── resp[4]               ← الـ response من الكارت
│     └── flags                 ← نوع الـ response (R1/R1B/R2/R3...)
│
├── struct mmc_data *data       ← البيانات (لو في)
│     ├── blksz / blocks        ← حجم ونوع البلوكات
│     ├── flags (READ/WRITE)
│     ├── struct scatterlist *sg ← الـ memory buffers
│     └── bytes_xfered          ← كم بايت اتنقل فعلاً
│
├── struct mmc_command *stop    ← STOP_TRANSMISSION (CMD12) - optional
└── completion                  ← للـ synchronous waiting
```

---

### الـ mmc_ios بالتفصيل — إيه أهميته؟

الـ `mmc_ios` هو "إعدادات الـ link الفيزيائية" — مش بروتوكول، بس physical layer:

```c
struct mmc_ios {
    unsigned int  clock;         /* تردد الـ SDCLK بالـ Hz */
    unsigned short vdd;          /* bit mask لجهد الـ VDD */
    unsigned char bus_mode;      /* open-drain أو push-pull */
    unsigned char power_mode;    /* OFF / UP / ON */
    unsigned char bus_width;     /* 1-bit / 4-bit / 8-bit */
    unsigned char timing;        /* LEGACY حتى HS400 */
    unsigned char signal_voltage;/* 3.3V / 1.8V / 1.2V */
    unsigned char drv_type;      /* drive strength A/B/C/D */
    bool enhanced_strobe;        /* HS400ES mode */
};
```

**التسلسل الزمني لتغيير الـ timing mode (مثال: الترقي لـ HS200):**

```
Core:  ios.timing = MMC_TIMING_MMC_HS200
Core:  ios.clock  = 200000000
Core:  ios.signal_voltage = MMC_SIGNAL_VOLTAGE_180
Core:  ops->set_ios(host, &ios)
                │
                ▼
Driver: يكتب registers الـ controller
        - يضبط PLL للـ 200MHz
        - يخفض جهد الـ IO لـ 1.8V
        - يغير الـ DLL settings
```

---

### الـ Timing Modes — من الأبطأ للأسرع

| الـ Mode | الـ `timing` constant | أقصى سرعة | Bus Type |
|---|---|---|---|
| Legacy SD | `MMC_TIMING_LEGACY` | 25 MHz | SDR |
| SD High Speed | `MMC_TIMING_SD_HS` | 50 MHz | SDR |
| UHS-I SDR12 | `MMC_TIMING_UHS_SDR12` | 12.5 MHz | SDR |
| UHS-I SDR25 | `MMC_TIMING_UHS_SDR25` | 25 MHz | SDR |
| UHS-I SDR50 | `MMC_TIMING_UHS_SDR50` | 50 MHz | SDR |
| UHS-I SDR104 | `MMC_TIMING_UHS_SDR104` | 104 MHz | SDR |
| UHS-I DDR50 | `MMC_TIMING_UHS_DDR50` | 50 MHz | DDR |
| eMMC HS | `MMC_TIMING_MMC_HS` | 52 MHz | SDR |
| eMMC DDR52 | `MMC_TIMING_MMC_DDR52` | 52 MHz | DDR |
| eMMC HS200 | `MMC_TIMING_MMC_HS200` | 200 MHz | SDR |
| eMMC HS400 | `MMC_TIMING_MMC_HS400` | 400 MHz | DDR |
| UHS-II Speed A | `MMC_TIMING_UHS2_SPEED_A` | 1.56 Gbps | Packet |

---

### الـ Tuning — ليه موجود وبيشتغل ازاي؟

في الـ high-speed modes (HS200، SDR104)، الـ setup/hold times بيبقوا ضيقين جداً. التوقيت اللي بيقرا بيه الـ controller البيانات لازم يبقى في النص تماماً.

**الـ Tuning Process:**
1. الـ core بيطلب `ops->execute_tuning(host, opcode)`.
2. الـ driver بيبعت الـ tuning pattern command (CMD19 لـ SD، CMD21 لـ eMMC).
3. بيجرب phases مختلفة للـ sampling clock.
4. بيلاقي الـ window اللي فيها البيانات صح.
5. بيحفظ الـ optimal phase ويشتغل بيها.

لو الكارت اشتكى من CRC errors، الـ `mmc_retune_needed()` بتشتغل وبتخلي الـ `need_retune = 1` — وفي أول request جاي بيحصل re-tuning.

---

### الـ Command Queue Engine (CQE) — ليه مهم؟

الـ normal request model هو **synchronous** — بعت command، استنى، بعت تاني.

الـ **CQE** (موجود في eMMC 5.1+) بيخلي الكارت نفسه يـ queue لـ 32 request في وقت واحد ويرتبهم بكفاءة جوه (قريب من الـ NCQ في SATA).

الـ `mmc_cqe_ops` بتـ abstract هذا:
```c
struct mmc_cqe_ops {
    int  (*cqe_enable)(host, card);    /* فعّل الـ CQE */
    int  (*cqe_request)(host, mrq);    /* submit tagged I/O */
    void (*cqe_off)(host);             /* وقّف الـ CQE مؤقتاً للـ non-CQ commands */
    int  (*cqe_wait_for_idle)(host);   /* استنى الـ queue يفضى */
    void (*cqe_recovery_start)(host);  /* ابدأ error recovery */
    void (*cqe_recovery_finish)(host); /* خلّص recovery، استدعي mmc_cqe_request_done */
};
```

الـ `mmc_host` بيحتفظ بـ `cqe_qdepth` (عمق الـ queue) وبـ `cqe_private` للـ driver-specific state.

---

### الـ mmc_host Capabilities — الـ caps وـ caps2

الـ `caps` و`caps2` هما bitmask بيوصفوا قدرات الـ controller — مش الكارت:

**أهم الـ caps:**
| Flag | المعنى |
|---|---|
| `MMC_CAP_4_BIT_DATA` | يدعم الـ 4-bit bus mode |
| `MMC_CAP_8_BIT_DATA` | يدعم الـ 8-bit (eMMC فقط) |
| `MMC_CAP_NONREMOVABLE` | الكارت ثابت (eMMC على البورد) |
| `MMC_CAP_UHS_SDR104` | يدعم UHS-I SDR104 |
| `MMC_CAP_HW_RESET` | في RST_n pin للـ eMMC |
| `MMC_CAP_CMD23` | يدعم SET_BLOCK_COUNT قبل multiblock |

**أهم الـ caps2:**
| Flag | المعنى |
|---|---|
| `MMC_CAP2_HS200` | يدعم eMMC HS200 |
| `MMC_CAP2_HS400` | يدعم eMMC HS400 |
| `MMC_CAP2_CQE` | في Command Queue Engine |
| `MMC_CAP2_CRYPTO` | يدعم inline encryption |
| `MMC_CAP2_NO_SDIO` | لا ترسل SDIO commands (eMMC-only slot) |

الـ core بيعمل AND بين قدرات الـ host وقدرات الكارت، وبيختار أعلى mode مشترك.

---

### دورة حياة الـ Host Driver

```bash
# 1. التسجيل
mmc_alloc_host(sizeof(struct my_priv), &pdev->dev)
    # بيخصص mmc_host + private data بعده مباشرةً
    # mmc_priv(host) → يرجع pointer للـ private data

# 2. ملي القدرات
host->ops     = &my_host_ops;
host->f_min   = 400000;        # 400 kHz للـ initialization
host->f_max   = 200000000;     # 200 MHz للـ HS200
host->caps    = MMC_CAP_8_BIT_DATA | MMC_CAP_NONREMOVABLE;
host->caps2   = MMC_CAP2_HS200 | MMC_CAP2_HS400;
host->ocr_avail = MMC_VDD_32_33 | MMC_VDD_33_34;

# 3. Parse من Device Tree
mmc_of_parse(host);   # بيقرا mmc-ddr-1_8v, bus-width, إلخ

# 4. تسجيل مع الـ core
mmc_add_host(host);
    # الـ core بيبدأ card detection
    # بيبعت CMD0 (GO_IDLE), CMD1/ACMD41, CMD2, CMD3...
    # بيكشف نوع الكارت وبينشئ mmc_card
    # بيسجل الكارت في device model

# 5. عند الإزالة
mmc_remove_host(host);
mmc_free_host(host);
```

---

### الـ mmc_priv — الـ Pattern الخاص بالـ Private Data

```c
/* في الـ driver */
struct my_host {
    void __iomem *base;
    struct clk   *clk;
    int           irq;
};

/* allocation */
struct mmc_host *mmc = mmc_alloc_host(sizeof(struct my_host), dev);
struct my_host  *priv = mmc_priv(mmc);  /* pointer بعد mmc_host مباشرةً */

/* استرجاع */
static irqreturn_t my_irq(int irq, void *dev_id)
{
    struct mmc_host *host = dev_id;
    struct my_host  *priv = mmc_priv(host);
    /* ... */
}
```

الـ `private[]` في نهاية `mmc_host` هو **flexible array member** محاذي لحدود الـ cache line — بيخلي الـ driver data في نفس الـ cache line تقريباً.

---

### الـ Power Supplies — الـ Regulators

```
struct mmc_supply
│
├── vmmc   ← card VDD (عادةً 3.3V) — اللي بيشغّل الكارت نفسه
├── vqmmc  ← IO signaling voltage — (3.3V أو 1.8V للـ UHS)
└── vqmmc2 ← للـ PHY في بعض الـ controllers (UHS-II)
```

**لما الـ core يعمل voltage switch لـ UHS-I:**
1. بيكال `mmc_regulator_set_vqmmc()` لخفض الـ IO voltage لـ 1.8V.
2. بيكال `ops->start_signal_voltage_switch()` لإخبار الـ controller.
3. الـ controller بيستنى الـ stabilization time.
4. بيكال `ops->execute_tuning()` لإعادة المعايرة.

---

### إيه اللي بيمتلكه الـ Core مقابل اللي بيفوّضه للـ Driver

| المهمة | المسؤول |
|---|---|
| Card initialization sequence (CMD0, ACMD41, CMD2...) | **Core** |
| Error retry وـ timeout handling | **Core** |
| Re-tuning trigger logic | **Core** |
| Power management (runtime PM) | **Core** |
| Register writes للـ controller | **Host Driver** |
| DMA setup وـ teardown | **Host Driver** |
| Clock generation وـ phase calibration | **Host Driver** |
| Interrupt handling من الـ controller | **Host Driver** |
| Voltage switching (تحريك الـ regulator) | **Core يطلب، Driver ينفذ** |
| Tuning algorithm (send pattern, check, adjust) | **Driver** |
| Card detection (اكتشاف وجود الكارت) | **Core يتابع، Driver يُبلّغ** |

---

### الـ SDIO IRQ Path — مثال عملي

الـ SDIO cards (زي Wi-Fi chips) بتحتاج interrupt-driven model:

```
Wi-Fi Chip
  │ pulls DAT[1] low → interrupt signal
  ▼
Host Controller Hardware
  │ generates CPU interrupt
  ▼
Host Driver ISR
  mmc_signal_sdio_irq(host):
    ├── ops->enable_sdio_irq(host, 0)   ← وقّف الـ IRQ مؤقتاً
    ├── host->sdio_irq_pending = true
    └── wake_up_process(sdio_irq_thread)
              │
              ▼
       sdio_irq_thread (kernel thread)
              │
              ▼
       sdio_signal_irq(host)
              │
              ▼
       SDIO function driver handler
       (Wi-Fi driver يقرا البيانات)
```

---

### الخلاصة — Mental Model نهائي

```
[Block/SDIO/Crypto Consumer]
         │
         │  mmc_request (command + data)
         ▼
[MMC Core] ← يعرف بروتوكول، مش hardware
    │
    │  ops->request() / ops->set_ios()
    ▼
[mmc_host_ops vtable]
    │
    │  register writes + DMA
    ▼
[Host Controller Driver] ← يعرف hardware، مش بروتوكول
    │
    │  physical signals
    ▼
[SD/eMMC/SDIO Card]
```

الـ `mmc_host` هو النقطة الوحيدة اللي بتلتقي فيها الجهتين — فيه الـ state الكامل للـ bus وفيه الـ vtable اللي بيعبر الـ abstraction boundary.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options

#### `mmc_ios` — Bus Settings Defines

| Define | القيمة | المعنى |
|--------|--------|---------|
| `MMC_BUSMODE_OPENDRAIN` | 1 | الـ bus في وضع open-drain (تهيئة البطاقة) |
| `MMC_BUSMODE_PUSHPULL` | 2 | الـ bus في وضع push-pull (التشغيل الطبيعي) |
| `MMC_CS_DONTCARE` | 0 | SPI chip select — مش مهم |
| `MMC_CS_HIGH` | 1 | SPI chip select active-high |
| `MMC_CS_LOW` | 2 | SPI chip select active-low |
| `MMC_POWER_OFF` | 0 | الطاقة مقطوعة تماماً |
| `MMC_POWER_UP` | 1 | طور الإشعال — رامب-أب |
| `MMC_POWER_ON` | 2 | الطاقة شغالة، البطاقة جاهزة |
| `MMC_POWER_UNDEFINED` | 3 | حالة غير معروفة |
| `MMC_BUS_WIDTH_1` | 0 | خط بيانات واحد |
| `MMC_BUS_WIDTH_4` | 2 | 4 خطوط بيانات |
| `MMC_BUS_WIDTH_8` | 3 | 8 خطوط بيانات (eMMC فقط) |

#### `mmc_ios` — Timing Modes

| Define | القيمة | السرعة القصوى |
|--------|--------|---------------|
| `MMC_TIMING_LEGACY` | 0 | 25 MHz |
| `MMC_TIMING_MMC_HS` | 1 | 52 MHz |
| `MMC_TIMING_SD_HS` | 2 | 50 MHz |
| `MMC_TIMING_UHS_SDR12` | 3 | 25 MHz |
| `MMC_TIMING_UHS_SDR25` | 4 | 50 MHz |
| `MMC_TIMING_UHS_SDR50` | 5 | 100 MHz |
| `MMC_TIMING_UHS_SDR104` | 6 | 208 MHz |
| `MMC_TIMING_UHS_DDR50` | 7 | 50 MHz DDR |
| `MMC_TIMING_MMC_DDR52` | 8 | 52 MHz DDR |
| `MMC_TIMING_MMC_HS200` | 9 | 200 MHz |
| `MMC_TIMING_MMC_HS400` | 10 | 400 MHz DDR |
| `MMC_TIMING_SD_EXP` | 11 | SD Express via PCIe |
| `MMC_TIMING_SD_EXP_1_2V` | 12 | SD Express 1.2V |
| `MMC_TIMING_UHS2_SPEED_A` | 13 | UHS-II Speed A |
| `MMC_TIMING_UHS2_SPEED_A_HD` | 14 | UHS-II Speed A Half-Duplex |
| `MMC_TIMING_UHS2_SPEED_B` | 15 | UHS-II Speed B |
| `MMC_TIMING_UHS2_SPEED_B_HD` | 16 | UHS-II Speed B Half-Duplex |

#### `mmc_ios` — Signal Voltage

| Define | القيمة | الجهد |
|--------|--------|-------|
| `MMC_SIGNAL_VOLTAGE_330` | 0 | 3.3V (الافتراضي) |
| `MMC_SIGNAL_VOLTAGE_180` | 1 | 1.8V (UHS-I, HS200, HS400) |
| `MMC_SIGNAL_VOLTAGE_120` | 2 | 1.2V |

#### `mmc_host.caps` — Host Capabilities (32-bit bitmask)

| Flag | Bit | الوظيفة |
|------|-----|---------|
| `MMC_CAP_4_BIT_DATA` | 0 | دعم 4-bit data bus |
| `MMC_CAP_MMC_HIGHSPEED` | 1 | eMMC High Speed |
| `MMC_CAP_SD_HIGHSPEED` | 2 | SD High Speed |
| `MMC_CAP_SDIO_IRQ` | 3 | دعم SDIO interrupts |
| `MMC_CAP_SPI` | 4 | وضع SPI |
| `MMC_CAP_NEEDS_POLL` | 5 | polling لاكتشاف البطاقة |
| `MMC_CAP_8_BIT_DATA` | 6 | دعم 8-bit (eMMC) |
| `MMC_CAP_AGGRESSIVE_PM` | 7 | suspend عدواني لتوفير الطاقة |
| `MMC_CAP_NONREMOVABLE` | 8 | بطاقة غير قابلة للإزالة (eMMC) |
| `MMC_CAP_WAIT_WHILE_BUSY` | 9 | الـ host ينتظر busy signal |
| `MMC_CAP_3_3V_DDR` | 11 | DDR @ 3.3V |
| `MMC_CAP_1_8V_DDR` | 12 | DDR @ 1.8V |
| `MMC_CAP_1_2V_DDR` | 13 | DDR @ 1.2V |
| `MMC_CAP_POWER_OFF_CARD` | 14 | قطع الطاقة كاملاً |
| `MMC_CAP_UHS_SDR12..DDR50` | 16-20 | دعم UHS-I modes |
| `MMC_CAP_SYNC_RUNTIME_PM` | 21 | runtime PM متزامن |
| `MMC_CAP_CMD_DURING_TFR` | 29 | أوامر أثناء نقل البيانات |
| `MMC_CAP_CMD23` | 30 | دعم SET_BLOCK_COUNT |
| `MMC_CAP_HW_RESET` | 31 | reset عبر RST_n pin |

#### `mmc_host.caps2` — More Host Capabilities

| Flag | Bit | الوظيفة |
|------|-----|---------|
| `MMC_CAP2_BOOTPART_NOACC` | 0 | لا وصول لـ boot partitions |
| `MMC_CAP2_FULL_PWR_CYCLE` | 2 | دورة طاقة كاملة |
| `MMC_CAP2_HS200_1_8V_SDR` | 5 | HS200 @ 1.8V |
| `MMC_CAP2_HS200_1_2V_SDR` | 6 | HS200 @ 1.2V |
| `MMC_CAP2_SD_EXP` | 7 | SD Express via PCIe |
| `MMC_CAP2_SD_UHS2` | 9 | SD UHS-II |
| `MMC_CAP2_HS400_1_8V` | 15 | HS400 @ 1.8V |
| `MMC_CAP2_HS400_1_2V` | 16 | HS400 @ 1.2V |
| `MMC_CAP2_HS400_ES` | 20 | HS400 Enhanced Strobe |
| `MMC_CAP2_NO_SDIO` | 19 | تجاهل SDIO initialization |
| `MMC_CAP2_NO_SD` | 21 | تجاهل SD initialization |
| `MMC_CAP2_NO_MMC` | 22 | تجاهل eMMC initialization |
| `MMC_CAP2_CQE` | 23 | Command Queue Engine |
| `MMC_CAP2_CQE_DCMD` | 24 | CQE direct commands |
| `MMC_CAP2_CRYPTO` | 27 | inline encryption (يحتاج `CONFIG_MMC_CRYPTO`) |

#### `enum mmc_err_stat` — Error Statistics

| Enum | المعنى |
|------|--------|
| `MMC_ERR_CMD_TIMEOUT` | timeout على command |
| `MMC_ERR_CMD_CRC` | CRC error على command |
| `MMC_ERR_DAT_TIMEOUT` | timeout على data |
| `MMC_ERR_DAT_CRC` | CRC error على data |
| `MMC_ERR_AUTO_CMD` | خطأ في auto-CMD12/23 |
| `MMC_ERR_ADMA` | خطأ في ADMA (scatter-gather DMA) |
| `MMC_ERR_TUNING` | فشل tuning |
| `MMC_ERR_CMDQ_RED` | CQE Response Error Detected |
| `MMC_ERR_CMDQ_GCE` | CQE General Crypto Error |
| `MMC_ERR_CMDQ_ICCE` | CQE Invalid Crypto Config Error |
| `MMC_ERR_REQ_TIMEOUT` | request timeout |
| `MMC_ERR_CMDQ_REQ_TIMEOUT` | CQE request timeout |
| `MMC_ERR_ICE_CFG` | Inline Crypto Engine config error |
| `MMC_ERR_CTRL_TIMEOUT` | controller timeout |
| `MMC_ERR_UNEXPECTED_IRQ` | interrupt غير متوقع |
| `MMC_ERR_MAX` | عدد الـ counters |

#### `enum sd_uhs2_operation`

| Enum | المعنى |
|------|--------|
| `UHS2_PHY_INIT` | تهيئة الـ PHY layer |
| `UHS2_SET_CONFIG` | ضبط الـ UHS-II config |
| `UHS2_ENABLE_INT` | تفعيل الـ interrupts |
| `UHS2_DISABLE_INT` | تعطيل الـ interrupts |
| `UHS2_ENABLE_CLK` | تفعيل الـ clock |
| `UHS2_DISABLE_CLK` | تعطيل الـ clock |
| `UHS2_CHECK_DORMANT` | التحقق من Dormant state |
| `UHS2_SET_IOS` | ضبط الـ I/O settings |

#### VDD Voltage Defines (داخل `mmc_host`)

| Range | Mask | الجهد |
|-------|------|-------|
| `MMC_VDD_165_195` | `0x00000080` | 1.65–1.95V |
| `MMC_VDD_20_21` | `0x00000100` | 2.0–2.1V |
| `MMC_VDD_27_28` | `0x00008000` | 2.7–2.8V |
| `MMC_VDD_32_33` | `0x00100000` | 3.2–3.3V |
| `MMC_VDD_33_34` | `0x00200000` | 3.3–3.4V (الأشهر للـ SD) |
| `MMC_VDD_35_36` | `0x00800000` | 3.5–3.6V |

---

### الـ Structs المهمة

#### 1. `struct mmc_ios`

**الغرض:** تمثل الحالة الحالية للـ I/O bus — كل ما يخص الإشارات المادية.

```c
struct mmc_ios {
    unsigned int  clock;           /* تردد الـ clock بالـ Hz */
    unsigned short vdd;            /* رقم bit الجهد المختار من VDD defines */
    unsigned int  power_delay_ms;  /* انتظار استقرار الطاقة */
    unsigned char bus_mode;        /* OPENDRAIN أو PUSHPULL */
    unsigned char chip_select;     /* SPI: HIGH / LOW / DONTCARE */
    unsigned char power_mode;      /* OFF / UP / ON / UNDEFINED */
    unsigned char bus_width;       /* 1 / 4 / 8 bit */
    unsigned char timing;          /* LEGACY إلى UHS2_SPEED_B_HD */
    unsigned char signal_voltage;  /* 3.3V / 1.8V / 1.2V */
    unsigned char vqmmc2_voltage;  /* جهد إضافي للـ PHY */
    unsigned char drv_type;        /* A / B / C / D */
    bool enhanced_strobe;          /* HS400 Enhanced Strobe */
};
```

**الصلة بالـ structs الأخرى:** مخزنة داخل `mmc_host.ios`، تُمرر لـ `ops->set_ios()`.

---

#### 2. `struct mmc_clk_phase` و `struct mmc_clk_phase_map`

**الغرض:** ضبط phase shift للـ clock للتعامل مع الـ timing skew في الـ PCB.

```c
struct mmc_clk_phase {
    bool valid;    /* هل القيمة دي صالحة؟ */
    u16  in_deg;   /* زاوية phase للـ input clock بالدرجات */
    u16  out_deg;  /* زاوية phase للـ output clock بالدرجات */
};

/* مصفوفة بـ phase لكل timing mode من LEGACY لـ HS400 */
struct mmc_clk_phase_map {
    struct mmc_clk_phase phase[MMC_NUM_CLK_PHASES]; /* 11 entry */
};
```

**الاستخدام:** تُحمل من device tree عبر `mmc_of_parse_clk_phase()`.

---

#### 3. `struct sd_uhs2_caps`

**الغرض:** قدرات الـ host في وضع SD UHS-II (واجهة serial عالية السرعة بـ 2 lanes).

| الحقل | المعنى |
|-------|--------|
| `dap` | Data Access Pattern |
| `gap` | Gap parameter |
| `n_lanes` | عدد الـ lanes (1 أو 2) |
| `addr64` | دعم 64-bit addressing |
| `speed_range` | Speed A أو Speed B |
| `n_fcu` | عدد الـ Flow Control Units |
| `*_set` fields | القيم المفاوَض عليها مع البطاقة |

---

#### 4. `struct mmc_host_ops`

**الغرض:** جدول virtual functions — كل driver للـ host controller يملأ الـ function pointers دي.

```c
struct mmc_host_ops {
    /* طلبات البيانات */
    void (*pre_req)(host, req);          /* double-buffering: تحضير مسبق */
    void (*post_req)(host, req, err);    /* تنظيف بعد الطلب */
    void (*request)(host, req);          /* إرسال طلب (السياق العادي) */
    int  (*request_atomic)(host, req);   /* إرسال طلب في atomic context */

    /* ضبط الـ bus */
    void (*set_ios)(host, ios);          /* تطبيق إعدادات الـ bus */

    /* حالة الـ hardware */
    int  (*get_ro)(host);                /* 0=RW, 1=RO, -ENOSYS=مش موجود */
    int  (*get_cd)(host);                /* 0=فاضي, 1=موجود بطاقة */
    int  (*card_busy)(host);             /* هل DAT[0] low؟ */

    /* SDIO interrupts */
    void (*enable_sdio_irq)(host, en);
    void (*ack_sdio_irq)(host);

    /* إعادة الضبط والتهيئة */
    void (*init_card)(host, card);
    void (*card_hw_reset)(host);         /* RST_n pin */
    void (*card_event)(host);

    /* Voltage switching */
    int  (*start_signal_voltage_switch)(host, ios);

    /* Tuning — لمحاذاة timing */
    int  (*execute_tuning)(host, opcode);
    int  (*prepare_hs400_tuning)(host, ios);
    int  (*execute_hs400_tuning)(host, card);
    int  (*prepare_sd_hs_tuning)(host, card);
    int  (*execute_sd_hs_tuning)(host, card);

    /* HS400 lifecycle */
    int  (*hs400_prepare_ddr)(host);
    void (*hs400_downgrade)(host);
    void (*hs400_complete)(host);
    void (*hs400_enhanced_strobe)(host, ios);

    /* متفرقات */
    int  (*select_drive_strength)(card, max_dtr, host_drv, card_drv, *drv_type);
    int  (*multi_io_quirk)(card, direction, blk_size);
    int  (*init_sd_express)(host, ios);
    int  (*uhs2_control)(host, op);      /* إلزامي لـ UHS-II */
};
```

**الصلة:** محفوظة في `mmc_host.ops` كـ `const` pointer — الـ core يستدعيها، الـ driver يملأها.

---

#### 5. `struct mmc_cqe_ops`

**الغرض:** virtual functions خاصة بالـ Command Queue Engine (CQE) — يسمح بـ 32 طلب متوازي.

```c
struct mmc_cqe_ops {
    int  (*cqe_enable)(host, card);       /* تفعيل الـ CQE وتخصيص الموارد */
    void (*cqe_disable)(host);            /* تعطيل وتحرير موارد */
    int  (*cqe_request)(host, mrq);       /* إرسال طلب للـ queue */
    void (*cqe_post_req)(host, mrq);      /* تنظيف (DMA unmap إلخ) */
    void (*cqe_off)(host);                /* تحضير لقبول non-CQ commands */
    int  (*cqe_wait_for_idle)(host);      /* انتظار اكتمال كل الطلبات */
    bool (*cqe_timeout)(host, mrq, *recovery_needed); /* timeout handler */
    void (*cqe_recovery_start)(host);     /* بداية recovery */
    void (*cqe_recovery_finish)(host);    /* نهاية recovery */
};
```

---

#### 6. `struct mmc_slot`

**الغرض:** معلومات الـ slot المادي — card detection وغيره.

```c
struct mmc_slot {
    int  cd_irq;          /* IRQ رقم لكشف البطاقة، أو -EINVAL */
    bool cd_wake_enabled; /* هل يصحّي النظام لما تتغير حالة البطاقة؟ */
    void *handler_priv;   /* بيانات خاصة بـ driver الـ slot */
};
```

---

#### 7. `struct mmc_supply`

**الغرض:** تنظيم مصادر الطاقة الثلاثة للـ MMC controller.

```c
struct mmc_supply {
    struct regulator       *vmmc;     /* طاقة البطاقة (الأساسية) */
    struct regulator       *vqmmc;    /* طاقة الـ I/O interface (Vccq) */
    struct regulator       *vqmmc2;   /* طاقة PHY layer (اختيارية) */
    struct notifier_block   vmmc_nb;  /* notifier لأحداث vmmc */
    struct work_struct      uv_work;  /* work item لمعالجة under-voltage */
};
```

---

#### 8. `struct mmc_ctx`

**الغرض:** تحديد هوية الـ context اللي يمتلك الـ host حالياً.

```c
struct mmc_ctx {
    struct task_struct *task; /* الـ thread اللي عمل claim للـ host */
};
```

الفائدة: يمنع أي thread تاني من استخدام الـ host، ويتيح nesting للـ claim من نفس الـ context.

---

#### 9. `struct mmc_host` ← الـ struct الأساسي

**الغرض:** تمثل controller واحد كامل — الوعاء الرئيسي لكل حالة الـ host.

| الحقل | النوع | المعنى |
|-------|-------|---------|
| `parent` | `struct device *` | الـ device الأب (PCI/platform) |
| `class_dev` | `struct device` | الـ device في `/sys/class/mmc_hostX/` |
| `index` | `int` | رقم الـ host (mmc0, mmc1...) |
| `ops` | `const struct mmc_host_ops *` | جدول الـ driver operations |
| `pwrseq` | `struct mmc_pwrseq *` | Power sequence abstraction |
| `f_min/f_max/f_init` | `unsigned int` | حدود تردد الـ clock |
| `ocr_avail` | `u32` | الجهود المدعومة (bitmask) |
| `caps / caps2` | `u32` | قدرات الـ host (انظر الجداول أعلاه) |
| `max_seg_size` | `unsigned int` | أقصى حجم لـ DMA segment |
| `max_segs` | `unsigned short` | أقصى عدد segments |
| `max_req_size` | `unsigned int` | أقصى حجم طلب (bytes) |
| `max_blk_size` | `unsigned int` | أقصى حجم block |
| `max_blk_count` | `unsigned int` | أقصى عدد blocks في طلب |
| `lock` | `spinlock_t` | يحمي claim وbus ops |
| `ios` | `struct mmc_ios` | الحالة الحالية للـ bus |
| `claimed` | `bitfield:1` | هل الـ host محجوز؟ |
| `claimer` | `struct mmc_ctx *` | مَن حجز الـ host |
| `claim_cnt` | `int` | عمق الـ nesting |
| `card` | `struct mmc_card *` | البطاقة المتصلة |
| `bus_ops` | `const struct mmc_bus_ops *` | عمليات الـ bus driver الحالي |
| `supply` | `struct mmc_supply` | مصادر الطاقة |
| `cqe_ops` | `const struct mmc_cqe_ops *` | CQE operations |
| `cqe_enabled/on` | `bool` | حالة الـ CQE |
| `retune_timer` | `struct timer_list` | timer لإعادة الـ tuning |
| `err_stats[]` | `u32[MMC_ERR_MAX]` | إحصائيات الأخطاء |
| `crypto_profile` | `struct blk_crypto_profile` | inline encryption (إذا مفعّل) |
| `private[]` | `unsigned long[]` | بيانات خاصة بالـ driver (cacheline-aligned) |

---

#### 10. `struct mmc_request` (من core.h)

```c
struct mmc_request {
    struct mmc_command *sbc;        /* SET_BLOCK_COUNT (multiblock) */
    struct mmc_command *cmd;        /* الأمر الأساسي */
    struct mmc_data    *data;       /* بيانات الـ data phase */
    struct mmc_command *stop;       /* STOP_TRANSMISSION */
    struct completion   completion; /* للانتظار بعد اكتمال الطلب */
    void (*done)(struct mmc_request *); /* callback بعد الاكتمال */
    struct mmc_host    *host;
    bool                cap_cmd_during_tfr; /* أوامر أثناء النقل */
    int                 tag;        /* CQE task tag */
};
```

---

#### 11. `struct mmc_card` (من card.h)

```c
struct mmc_card {
    struct mmc_host   *host;      /* الـ host اللي البطاقة متصلة بيه */
    struct device      dev;
    unsigned int       type;      /* MMC / SD / SDIO / SD_COMBO */
    unsigned int       state;     /* حالة البطاقة */
    unsigned int       quirks;    /* workarounds لبطاقات معطوبة */
    struct mmc_cid     cid;       /* Card ID (manufacturer, serial...) */
    struct mmc_csd     csd;       /* Card Specific Data */
    struct mmc_ext_csd ext_csd;   /* Extended CSD (eMMC v4+) */
    struct sd_scr      scr;       /* SD Configuration Register */
    struct sdio_func  *sdio_func[SDIO_MAX_FUNCS]; /* SDIO functions */
    struct mmc_part    part[MMC_NUM_PHY_PARTITION]; /* physical partitions */
};
```

---

### مخططات العلاقات بين الـ Structs

```
                    ┌─────────────────────────────────────┐
                    │           struct mmc_host            │
                    │                                      │
                    │  .ops ──────────────────────────────►│struct mmc_host_ops
                    │  .cqe_ops ──────────────────────────►│struct mmc_cqe_ops
                    │  .ios ───────────────────────────────►│struct mmc_ios (embedded)
                    │  .card ─────────────────────────────►│struct mmc_card
                    │  .supply ────────────────────────────►│struct mmc_supply (embedded)
                    │  .slot ─────────────────────────────►│struct mmc_slot (embedded)
                    │  .claimer ──────────────────────────►│struct mmc_ctx
                    │  .ongoing_mrq ──────────────────────►│struct mmc_request
                    │  .uhs2_caps ────────────────────────►│struct sd_uhs2_caps (embedded)
                    │  .private[] ← driver private data    │
                    └─────────────────────────────────────┘
                               │
                               │ .parent
                               ▼
                    ┌─────────────────────┐
                    │   struct device     │  (platform/PCI/SDIO bus)
                    └─────────────────────┘

┌─────────────────┐          ┌─────────────────────────────┐
│  struct mmc_card│          │      struct mmc_request     │
│  .host ─────────┼─────────►│  .cmd ──►struct mmc_command │
│  .cid           │          │  .data ─►struct mmc_data    │
│  .csd           │          │  .stop ─►struct mmc_command │
│  .ext_csd       │          │  .sbc ──►struct mmc_command │
│  .sdio_func[]   │          │  .host ─►struct mmc_host    │
└─────────────────┘          └─────────────────────────────┘

┌─────────────────────────────────┐
│         struct mmc_supply       │
│  .vmmc ──►struct regulator      │
│  .vqmmc ─►struct regulator      │
│  .vqmmc2 ►struct regulator      │
│  .vmmc_nb ►struct notifier_block│
│  .uv_work ►struct work_struct   │
└─────────────────────────────────┘
```

---

### مخطط دورة حياة الـ mmc_host

```
Driver Probe
    │
    ▼
mmc_alloc_host(extra_size, parent_dev)
    │  يخصص: sizeof(mmc_host) + extra_size
    │  extra_size = بيانات الـ driver الخاصة
    │  يُرجع: struct mmc_host *
    │
    ▼
driver يملأ:
    host->ops        = &driver_ops
    host->f_min/max  = clock limits
    host->caps       = capabilities bitmask
    host->ocr_avail  = supported voltages
    host->max_blk_*  = DMA limits
    │
    ▼
mmc_add_host(host)
    │  يسجل الـ device في sysfs
    │  يشغّل mmc_start_host()
    │  يبدأ card detection
    │
    ▼
[Host Active — يخدم طلبات]
    │
    │  اكتُشفت بطاقة
    ▼
mmc_detect_change(host, delay)
    │  يجدوِل host->detect (delayed_work)
    │
    ▼
mmc_rescan() [في workqueue]
    │  يجرب: SDIO → SD → MMC
    │  يخصص mmc_card
    │  يشغل bus driver
    │
    ▼
[Card Active — يعالج I/O]
    │
    │  عند الإزالة أو shutdown
    ▼
mmc_remove_host(host)
    │  يوقف الـ rescan
    │  يُزيل البطاقة
    │  يُلغي تسجيل الـ device
    │
    ▼
mmc_free_host(host)
    │  put_device(&host->class_dev)
    │  يحرر الذاكرة في النهاية
    ▼
   END
```

---

### مخطط دورة حياة الـ CQE

```
mmc_cqe_ops->cqe_enable(host, card)
    │  تخصيص موارد CQE
    │  host->cqe_enabled = true
    │
    ▼
[CQE Active]
    │
    ├──► cqe_request(host, mrq)   ← طلبات متوازية (حتى 32)
    │       tag = mrq->tag
    │
    ├──► cqe_off(host)            ← قبل non-CQ commands
    │       host->cqe_on = false
    │       [non-CQ command]
    │       [cqe_request يُعيد تفعيله]
    │
    ├──► timeout scenario:
    │       cqe_timeout(host, mrq, &recovery_needed)
    │       إذا recovery_needed:
    │           cqe_recovery_start(host)
    │           [abort all tasks]
    │           cqe_recovery_finish(host)
    │           [mmc_cqe_request_done() على كل طلب]
    │
    ▼
cqe_wait_for_idle(host)
cqe_disable(host)
    │  host->cqe_enabled = false
    ▼
  END
```

---

### مخطط تدفق الطلبات العادية (non-CQE)

```
[Block Layer / SDIO Driver]
        │
        │ mmc_wait_for_req(host, mrq)
        ▼
[MMC Core: mmc_start_request()]
        │
        ├─► check retune needed?
        │       إذا host->need_retune:
        │           ops->execute_tuning(host, opcode)
        │
        ├─► ops->pre_req(host, mrq)   [اختياري — double buffering]
        │
        ├─► ops->request(host, mrq)
        │       ▼
        │   [Host Controller Driver]
        │       │ يبني DMA descriptors
        │       │ يكتب في registers
        │       │ يُطلق Transfer
        │       ▼
        │   [Hardware interrupt]
        │       │
        │       ▼
        │   mmc_request_done(host, mrq)
        │       │ host->ops->post_req(host, mrq, 0)
        │       │ mrq->done(mrq)
        │       ▼
        │   completion signaled
        │
        ▼
[Block Layer استلم النتيجة]
```

---

### مخطط تدفق تهيئة HS400

```
[MMC Core — hs400 init sequence]
        │
        ▼
1. ضبط HS200 @ 200MHz
        ops->set_ios(host, &ios)

        ▼
2. ops->execute_tuning(host, MMC_SEND_TUNING_BLOCK_HS200)
   ← يحدد أفضل phase للـ sampling

        ▼
3. ops->prepare_hs400_tuning(host, &ios)
   ← الـ driver يجهز الـ PHY للانتقال

        ▼
4. تخفيض السرعة لـ HS52 مؤقتاً
        ops->hs400_prepare_ddr(host)

        ▼
5. تفعيل DDR mode في البطاقة (CMD6)

        ▼
6. الانتقال لـ HS400 @ 200MHz DDR
        ops->set_ios(host, &ios)

        ▼
7. ops->execute_hs400_tuning(host, card)
   ← fine-tune للـ strobe

        ▼
8. ops->hs400_complete(host)
   ← الـ driver يُثبّت الإعدادات

[HS400 Active — 400 MB/s theoretical]
```

---

### مخطط دورة حياة الـ Voltage Switching (SD UHS-I)

```
[MMC Core]
        │
        ▼
1. بيرسل ACMD41 → يعرف البطاقة بتدعم 1.8V
        │
        ▼
2. بيبعت CMD11 (VOLTAGE_SWITCH)
        ops->request(host, mrq_cmd11)
        │
        ▼
3. بيوقف الـ clock
        ios.clock = 0
        ops->set_ios(host, &ios)
        │
        ▼
4. بيقول للـ driver يغير الجهد
        ops->start_signal_voltage_switch(host, &ios)
        ← الـ driver يبدل الـ regulator من 3.3V → 1.8V
        │
        ▼
5. بيشغل الـ clock تاني
        ios.clock = old_clock
        ios.timing = MMC_TIMING_UHS_SDR12  (أو أعلى لاحقاً)
        ops->set_ios(host, &ios)
        │
        ▼
6. بيتأكد DAT[0] بقى high
        ops->card_busy(host) → يرجع 0 (مش busy)
        │
        ▼
[UHS-I Mode Active]
```

---

### استراتيجية الـ Locking

#### الـ Locks الموجودة في `mmc_host`

| الـ Lock | النوع | يحمي إيه |
|----------|-------|-----------|
| `host->lock` | `spinlock_t` | `claimed`، `claim_cnt`، `claimer`، bus ops state |
| `host->wq` | `wait_queue_head_t` | ينتظر عليها الـ threads اللي عايزين يعملوا claim |

#### آلية الـ Claiming

الفكرة: بس thread واحد يقدر يكلم الـ host controller في نفس الوقت.

```c
/* احجز الـ host — يبلوك لو محجوز من حد تاني */
mmc_claim_host(host)
    spin_lock_irqsave(&host->lock, flags)
    while (host->claimed && host->claimer->task != current)
        wait on host->wq
    host->claimed = 1
    host->claimer = &ctx
    host->claim_cnt++
    spin_unlock_irqrestore(&host->lock, flags)

/* حرر الـ host */
mmc_release_host(host)
    spin_lock_irqsave(&host->lock, flags)
    if (--host->claim_cnt == 0)
        host->claimed = 0
        host->claimer = NULL
        wake_up(host->wq)
    spin_unlock_irqrestore(&host->lock, flags)
```

**الـ Nesting:** لو نفس الـ context عمل claim أكتر من مرة، الـ `claim_cnt` بيتزاد — محتاج نفس عدد الـ release.

#### ترتيب الـ Locks (Lock Ordering)

```
الترتيب الصحيح (من الخارج للداخل):
1. host->lock (spinlock) — أقصر lock
2. لا يوجد lock تاني بيتمسك جوا host->lock
   (كل العمليات الطويلة بتتعمل بعد release)

تحذير: ممنوع منع الـ IRQs لفترة طويلة جوا host->lock
        الـ ops callbacks (set_ios, request...) تتنادى بدون الـ lock
```

#### حماية الـ SDIO IRQ

```c
/* الحقول دي محمية بـ host->lock */
host->sdio_irqs          /* عدد الـ claimed IRQs */
host->sdio_irq_pending   /* علم وجود IRQ معلق */

/* الـ thread بيشتغل وهو ماسك host claim */
sdio_irq_thread:
    mmc_claim_host(host)
    sdio_run_irqs(host)
    mmc_release_host(host)
```

#### الـ Retune Timer

```
retune_timer (kernel timer, softirq context):
    │
    ▼
يضبط host->need_retune = 1
    │
    ▼
في أول request بعد كده:
    mmc_start_request() بيشوف need_retune
    بيعمل execute_tuning()
    بيصفّر need_retune
```

لا يوجد lock على `need_retune` — atomic read/write كافية هنا لأن worst case هو تأخير tuning بطلب واحد.

#### ملخص بصري للـ Locking

```
┌──────────────────────────────────────────────────────────┐
│                     Locking Map                          │
│                                                          │
│  host->lock (spinlock)                                   │
│  ├── claimed, claim_cnt, claimer                         │
│  └── sdio_irqs, sdio_irq_pending                        │
│                                                          │
│  mmc_claim_host() [mutex-like, sleeps on wq]             │
│  ├── bus operations (set_ios, request, etc.)             │
│  ├── card initialization                                 │
│  ├── tuning                                              │
│  └── all mmc_send_* functions                           │
│                                                          │
│  No lock needed (atomic or single-writer):               │
│  ├── need_retune (written by timer, read in request)     │
│  └── trigger_card_event                                  │
└──────────────────────────────────────────────────────────┘
```
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet لكل الـ APIs المهمة

#### Registration & Lifecycle

| Function | Category | Description |
|---|---|---|
| `mmc_alloc_host()` | Registration | allocate + init mmc_host struct |
| `devm_mmc_alloc_host()` | Registration | devres-managed version |
| `mmc_add_host()` | Registration | register host with MMC core |
| `mmc_remove_host()` | Cleanup | unregister + disconnect card |
| `mmc_free_host()` | Cleanup | free mmc_host struct |

#### Device Tree Parsing

| Function | Category | Description |
|---|---|---|
| `mmc_of_parse()` | DT Config | parse standard MMC DT properties |
| `mmc_of_parse_voltage()` | DT Config | parse voltage ranges from DT |
| `mmc_of_parse_clk_phase()` | DT Config | parse clock phase map from DT |

#### Runtime / Request Path

| Function | Category | Description |
|---|---|---|
| `mmc_detect_change()` | Runtime | schedule card detect work |
| `mmc_request_done()` | Runtime | complete a finished mmc_request |
| `mmc_command_done()` | Runtime | complete command part of request |
| `mmc_cqe_request_done()` | CQE Runtime | complete a CQE request |

#### Regulator Management

| Function | Category | Description |
|---|---|---|
| `mmc_regulator_get_supply()` | Power | lookup vmmc/vqmmc regulators |
| `mmc_regulator_set_ocr()` | Power | set card supply voltage via regulator |
| `mmc_regulator_set_vqmmc()` | Power | set I/O signalling voltage |
| `mmc_regulator_set_vqmmc2()` | Power | set PHY supply voltage |
| `mmc_regulator_enable_vqmmc()` | Power | enable vqmmc regulator |
| `mmc_regulator_disable_vqmmc()` | Power | disable vqmmc regulator |

#### Tuning Helpers

| Function | Category | Description |
|---|---|---|
| `mmc_send_tuning()` | Tuning | send CMD19/CMD21 tuning block |
| `mmc_send_abort_tuning()` | Tuning | abort tuning sequence |
| `mmc_retune_timer_stop()` | Tuning | stop periodic retune timer |
| `mmc_retune_needed()` | Tuning (inline) | mark host as needing retune |
| `mmc_can_retune()` | Tuning (inline) | query if retune is allowed |
| `mmc_doing_retune()` | Tuning (inline) | query if retune in progress |
| `mmc_doing_tune()` | Tuning (inline) | query init or re-tune in progress |

#### SD Helpers

| Function | Category | Description |
|---|---|---|
| `mmc_sd_switch()` | SD | send CMD6 switch/query command |
| `mmc_send_status()` | SD/eMMC | send CMD13 to read card status |
| `mmc_get_ext_csd()` | eMMC | read Extended CSD register |
| `mmc_read_tuning()` | SD/eMMC | read tuning block and compare |

#### SDIO IRQ

| Function | Category | Description |
|---|---|---|
| `sdio_signal_irq()` | SDIO IRQ | signal pending SDIO IRQ from threaded context |
| `mmc_signal_sdio_irq()` | SDIO IRQ (inline) | fast-path IRQ signal from ISR |
| `sdio_irq_claimed()` | SDIO IRQ (inline) | query if any SDIO IRQ is registered |

#### Accessor Inlines / Macros

| Function/Macro | Description |
|---|---|
| `mmc_priv()` | get host-driver private data pointer |
| `mmc_from_priv()` | get mmc_host from private pointer |
| `mmc_from_crypto_profile()` | get mmc_host from crypto_profile |
| `mmc_host_is_spi()` | check if host is SPI-only |
| `mmc_dev()` | get parent device |
| `mmc_classdev()` | get class_dev pointer |
| `mmc_hostname()` | get host name string |
| `mmc_card_is_removable()` | check if card is removable |
| `mmc_card_keep_power()` | check PM_KEEP_POWER flag |
| `mmc_card_wake_sdio_irq()` | check PM wake flag |
| `mmc_card_hs()` | check if card in HS timing |
| `mmc_card_uhs()` | check if card in UHS-I timing |
| `mmc_card_uhs2()` | check if host in UHS-II timing |
| `mmc_card_uhs2_hd_mode()` | check if UHS-II half-duplex mode |
| `mmc_get_dma_dir()` | get DMA direction from mmc_data flags |
| `mmc_debugfs_err_stats_inc()` | increment error counter in debugfs |

---

### Group 1: Registration & Lifecycle

الـ group ده مسؤول عن إنشاء الـ `mmc_host` struct وتسجيله مع الـ MMC core وإزالته. كل host driver لازم يمر بالترتيب: `alloc → populate ops/caps → add → (runtime) → remove → free`.

---

#### `mmc_alloc_host`

```c
struct mmc_host *mmc_alloc_host(int extra, struct device *dev);
```

بتعمل `kzalloc` للـ `mmc_host` مع إضافة `extra` bytes في نهايته (الـ `private[]` flexible array) عشان الـ host driver يخزن فيها data خاصة بيه. بتعمل init للـ locks والـ wait queues والـ work items الداخلية وبتربط الـ host بالـ parent device.

**Parameters:**
- `extra` — حجم البيانات الخاصة بالـ driver (bytes) — بتتوصل عن طريق `mmc_priv()`
- `dev` — الـ parent device (عادةً `&pdev->dev`)

**Return:** pointer للـ `mmc_host` المخصص، أو `NULL` لو فشل الـ allocation.

**Key details:**
- الـ `private[]` array بتتعمل align على cache line حسب `____cacheline_aligned`
- مش بتعمل registration مع الـ MMC core — ده شغل `mmc_add_host()`
- بتعمل init لـ `retune_timer`، `detect` delayed work، و`wq` wait queue

**Who calls it:** host drivers في `probe()` callback.

---

#### `devm_mmc_alloc_host`

```c
struct mmc_host *devm_mmc_alloc_host(struct device *dev, int extra);
```

نفس `mmc_alloc_host()` بالظبط لكن مع devres management — الـ host بيتـ free تلقائياً لما الـ device بتتـ detach. بتبسط error paths في الـ probe.

**Parameters:**
- `dev` — الـ device للـ devres binding
- `extra` — حجم private data

**Return:** pointer للـ `mmc_host` أو `ERR_PTR` on failure.

**Key details:**
- بتستخدم `devm_add_action_or_reset()` داخلياً لتسجيل cleanup
- لو `mmc_add_host()` اتسمى وفشل، لازم تعمل `mmc_remove_host()` يدوي قبل ما الـ devres يـ run

**Who calls it:** host drivers الحديثة في `probe()` — preferred على `mmc_alloc_host()`.

---

#### `mmc_add_host`

```c
int mmc_add_host(struct mmc_host *host);
```

بتكمل تسجيل الـ host مع الـ MMC core: بتعمل `device_add()` للـ `class_dev`، بتهيئ الـ debugfs entries، وبتشغل card detection. ده هو اللحظة اللي الـ MMC core بيبدأ فيها يتكلم مع الـ hardware.

**Parameters:**
- `host` — الـ host اللي اتخصص بـ `mmc_alloc_host()` وبالـ ops/caps المُعبأة

**Return:** `0` on success، negative errno on failure.

**Key details:**
- لازم الـ `host->ops` تتبع قبل الاستدعاء
- بتشغل `mmc_start_host()` اللي بتبدأ card scan
- الـ `rescan_disable` flag بيتـ clear هنا

**Who calls it:** host drivers في `probe()` بعد ما تكمل setup.

---

#### `mmc_remove_host`

```c
void mmc_remove_host(struct mmc_host *host);
```

بتعمل reverse لـ `mmc_add_host()`: بتوقف الـ host، بتفصل أي card متصل، وبتعمل `device_del()` للـ `class_dev`. بتنتظر أي pending work ينتهي.

**Parameters:**
- `host` — الـ host اللي هيتمسح

**Key details:**
- بتستدعي `mmc_stop_host()` اللي بتوقف الـ detect work وبتفصل الـ card
- blocking — بتنتظر الـ card removal تكتمل
- بعد ما بترجع، مفيش card access تانية ممكنة

**Who calls it:** host drivers في `remove()` أو `shutdown()` callbacks.

---

#### `mmc_free_host`

```c
void mmc_free_host(struct mmc_host *host);
```

بتحرر الـ `mmc_host` struct نفسه بعد `mmc_remove_host()`. بتعمل `put_device()` على الـ `class_dev` اللي بيـ trigger الـ kfree.

**Parameters:**
- `host` — الـ host اللي هيتـ free

**Key details:**
- لو استخدمت `devm_mmc_alloc_host()`، **متستدعيش** الفانكشن دي — الـ devres بيعملها تلقائي
- لازم `mmc_remove_host()` تتسمى قبلها

**Who calls it:** host drivers في `probe()` error path أو `remove()`.

---

### Group 2: Device Tree Parsing

الـ group ده بيقرأ الـ DT properties الـ standard اللي كل MMC host بيحتاجها — clock ranges، voltage masks، clock phases. بيوفر boilerplate parsing عشان الـ host drivers ميعيدوش اختراع العجلة.

---

#### `mmc_of_parse`

```c
int mmc_of_parse(struct mmc_host *host);
```

بتقرأ كل الـ standard MMC DT properties وبتعبي الـ `host->caps`، `host->caps2`، `host->f_min`، `host->f_max`، وغيرهم. دي الفانكشن الرئيسية للـ DT parsing في MMC subsystem.

**Parameters:**
- `host` — الـ host اللي هتتعبى فيه الـ caps

**Return:** `0` on success، negative errno on error.

**Key details:**

الـ properties اللي بتتقرأ تشمل:
- `bus-width` → `MMC_CAP_4_BIT_DATA` / `MMC_CAP_8_BIT_DATA`
- `cap-sd-highspeed` → `MMC_CAP_SD_HIGHSPEED`
- `cap-mmc-highspeed` → `MMC_CAP_MMC_HIGHSPEED`
- `sd-uhs-sdr12/25/50/104/ddr50` → `MMC_CAP_UHS_*`
- `mmc-hs200-1_8v/1_2v` → `MMC_CAP2_HS200_*`
- `mmc-hs400-1_8v/1_2v` → `MMC_CAP2_HS400_*`
- `non-removable` → `MMC_CAP_NONREMOVABLE`
- `no-sdio` → `MMC_CAP2_NO_SDIO`
- `no-sd` → `MMC_CAP2_NO_SD`
- `no-mmc` → `MMC_CAP2_NO_MMC`
- `max-frequency` → `host->f_max`

**Who calls it:** كل host driver في `probe()` — عادةً بعد `mmc_alloc_host()` مباشرة.

---

#### `mmc_of_parse_voltage`

```c
int mmc_of_parse_voltage(struct mmc_host *host, u32 *mask);
```

بتقرأ `voltage-ranges` property من الـ DT وبتحولها لـ OCR bitmask. مفيدة لو الـ host عنده voltage negotiation خاص.

**Parameters:**
- `host` — الـ host المعني
- `mask` — output: الـ OCR mask المُشتق من الـ DT

**Return:** عدد الـ voltage ranges المُقروءة، أو negative errno.

**Who calls it:** host drivers اللي بتحتاج explicit voltage parsing بدل الـ automatic اللي في `mmc_of_parse()`.

---

#### `mmc_of_parse_clk_phase`

```c
void mmc_of_parse_clk_phase(struct device *dev,
                             struct mmc_clk_phase_map *map);
```

بتقرأ الـ input/output clock phase angles (بالدرجات) من الـ DT لكل timing mode. بتعبي الـ `mmc_clk_phase_map` struct اللي بيستخدمها الـ host driver لضبط الـ clock phase tuning.

**Parameters:**
- `dev` — الـ device node للـ DT lookup
- `map` — output struct من `MMC_NUM_CLK_PHASES` entries (واحد لكل timing mode من `MMC_TIMING_LEGACY` لـ `MMC_TIMING_MMC_HS400`)

**Key details:**
- الـ DT property names بتكون على شكل `"mmc-ddr52-clk-phase-input-delay-deg"`
- لو الـ property مش موجودة، الـ corresponding entry بيبقى `valid = false`

**Who calls it:** host drivers اللي بتعمل manual clock phase calibration.

---

### Group 3: Runtime / Request Completion

الـ group ده هو heart الـ data transfer path. الـ host driver بيستدعي الـ functions دي من ISR أو DMA completion context عشان يبلغ الـ MMC core إن الـ request اتنهى.

---

#### `mmc_request_done`

```c
void mmc_request_done(struct mmc_host *host, struct mmc_request *mrq);
```

بتُعلم الـ MMC core إن الـ `mmc_request` اتنهى بالكامل (command + data + stop). بتشغل الـ `mrq->done()` callback وبتـ complete الـ `mrq->completion`. لو في retune مطلوب، بتضبطه.

**Parameters:**
- `host` — الـ host اللي نفّذ الـ request
- `mrq` — الـ request اللي اتنهى — لازم `cmd->error` و`data->error` يتبعوا قبل الاستدعاء

**Key details:**
- **لازم** تتسمى من context safe للـ scheduling (بتعمل `wake_up` على الـ wait queue)
- بتتحقق من الـ `MMC_CAP_DONE_COMPLETE` flag — لو مضبوطة، الـ upper layer عارف إن الـ request اتكمل جوا الـ interrupt
- بتـ handle LED activity toggling
- **لا تستدعيش** الفانكشن دي من CQE requests — استخدم `mmc_cqe_request_done()`

**Who calls it:** host drivers من ISR أو DMA completion tasklet/thread.

---

#### `mmc_command_done`

```c
void mmc_command_done(struct mmc_host *host, struct mmc_request *mrq);
```

بتُكمل الـ command phase بس من الـ `mmc_request` — مفيدة لو الـ host بيستخدم `MMC_CAP_CMD_DURING_TFR`. بتشغل `mrq->cmd_completion` للـ waiters على command completion بدون ما تنهي الـ data transfer.

**Parameters:**
- `host` — الـ host المعني
- `mrq` — الـ request اللي command phase بتاعه اتنهى

**Key details:**
- بتتعمل call قبل `mmc_request_done()` في الـ `CMD_DURING_TFR` scenario
- بتـ complete `mrq->cmd_completion` فقط — مش `mrq->completion`

**Who calls it:** host drivers اللي بتدعم concurrent commands during data transfer.

---

#### `mmc_cqe_request_done`

```c
void mmc_cqe_request_done(struct mmc_host *host, struct mmc_request *mrq);
```

نظير `mmc_request_done()` للـ Command Queue Engine. بتُعلم الـ MMC core بإتمام CQE request وبتشغل الـ `mrq->done()` callback.

**Parameters:**
- `host` — الـ host اللي عنده CQE
- `mrq` — الـ CQE request المنتهي

**Key details:**
- بتفترض إن الـ `mrq->tag` valid ومُعين لـ CQE task
- بتستدعيها الـ host driver من `cqe_ops->cqe_timeout()` أو completion interrupt

**Who calls it:** host drivers في CQE completion interrupt handler.

---

#### `mmc_detect_change`

```c
void mmc_detect_change(struct mmc_host *host, unsigned long delay);
```

بتـ schedule الـ `host->detect` delayed work بعد `delay` jiffies. الـ work بتعمل rescan للـ bus وتكتشف card insertion/removal.

**Parameters:**
- `host` — الـ host اللي في الـ bus بتاعه حصل change
- `delay` — تأخير (jiffies) قبل فحص الـ card — مفيد لـ debouncing

**Key details:**
- **آمنة للاستدعاء من interrupt context** — بتستخدم `schedule_delayed_work()`
- لو `rescan_disable == 1`، الـ work بتتـ schedule لكن مش بتعمل حاجة
- الـ `detect_change` flag بيتـ set لمنع الـ spurious rescans

**Who calls it:** ISR handlers لـ card detect GPIO، أو host drivers عند اكتشاف error.

---

### Group 4: SDIO IRQ Handling

الـ group ده بيدير الـ SDIO interrupt signalling — من الـ fast atomic path في ISR لحد الـ threaded handler.

---

#### `mmc_signal_sdio_irq` (inline)

```c
static inline void mmc_signal_sdio_irq(struct mmc_host *host);
```

**Fast path** لتبليغ الـ SDIO IRQ من ISR. بتعطل الـ SDIO IRQ على الـ hardware (لمنع الـ flood)، بتضبط `sdio_irq_pending = true`، وبتصحي الـ `sdio_irq_thread`.

**Key details:**
- بتستدعي `host->ops->enable_sdio_irq(host, 0)` — disable IRQ line
- **atomic-safe** — مش بتعمل lock
- الـ IRQ بيتـ re-enable من الـ `sdio_irq_thread` بعد ما يعالج الـ IRQ

**Who calls it:** host driver ISR لما يشوف SDIO interrupt pending.

---

#### `sdio_signal_irq`

```c
void sdio_signal_irq(struct mmc_host *host);
```

الـ non-atomic version للـ `mmc_signal_sdio_irq`. بتستخدم لما `MMC_CAP2_SDIO_IRQ_NOTHREAD` مضبوطة — بتشتغل من work/tasklet context بدل thread.

**Parameters:**
- `host` — الـ host اللي الـ SDIO IRQ جه منه

**Who calls it:** host drivers اللي بتستخدم `MMC_CAP2_SDIO_IRQ_NOTHREAD`.

---

#### `sdio_irq_claimed` (inline)

```c
static inline bool sdio_irq_claimed(struct mmc_host *host);
```

بترجع `true` لو في SDIO function واحدة على الأقل سجلت IRQ handler.

**Return:** `true` لو `host->sdio_irqs > 0`.

**Who calls it:** host drivers في `suspend()`/`resume()` لتحديد لو لازم يـ configure SDIO IRQ wakeup.

---

### Group 5: Regulator / Power Management

الـ group ده بيدير الـ voltage regulators للـ card power (vmmc) والـ I/O signalling (vqmmc).

---

#### `mmc_regulator_get_supply`

```c
int mmc_regulator_get_supply(struct mmc_host *mmc);
```

بتعمل `devm_regulator_get_optional()` للـ `vmmc` و`vqmmc` و`vqmmc2` من الـ DT. بتعبي `mmc->supply` وبتستنتج الـ `ocr_avail` من الـ regulator voltage range.

**Parameters:**
- `mmc` — الـ host المعني

**Return:** `0` on success، `EPROBE_DEFER` لو الـ regulator مش جاهز بعد، أو negative errno.

**Key details:**
- لو الـ `vmmc` regulator مش موجود (optional)، بيبقى `NULL` والـ function بترجع `0`
- بتشتغل مع الـ `notifier_block` الخاص بالـ undervoltage events

**Who calls it:** host drivers في `probe()` — عادةً بعد `mmc_of_parse()`.

---

#### `mmc_regulator_set_ocr`

```c
int mmc_regulator_set_ocr(struct mmc_host *mmc,
                           struct regulator *supply,
                           unsigned short vdd_bit);
```

بتحول الـ OCR bit number (من `mmc_ios.vdd`) لـ voltage microvolts وبتضبط الـ regulator. لو `vdd_bit == 0`، بتعطل الـ regulator (power off).

**Parameters:**
- `mmc` — الـ host
- `supply` — الـ regulator handle (عادةً `mmc->supply.vmmc`)
- `vdd_bit` — رقم الـ bit في الـ OCR register اللي بيمثل الـ voltage range المطلوب

**Return:** `0` on success، negative errno on failure.

**Key details:**
- الـ `MMC_VDD_*` macros بتعرف bit positions (7 = 1.65-1.95V، 23 = 3.3-3.4V)
- لو `CONFIG_REGULATOR` مش مُعرّف، بترجع `0` (stub)

**Who calls it:** MMC core في `set_ios()` flow لما بيغير الـ power state.

---

#### `mmc_regulator_set_vqmmc`

```c
int mmc_regulator_set_vqmmc(struct mmc_host *mmc, struct mmc_ios *ios);
```

بتضبط الـ vqmmc (I/O signalling) voltage بناءً على `ios->signal_voltage`. بتحول `MMC_SIGNAL_VOLTAGE_330/180/120` لـ microvolts وبتضبط الـ regulator.

**Parameters:**
- `mmc` — الـ host
- `ios` — الـ current I/O settings — بتقرأ `ios->signal_voltage` منه

**Return:** `0` on success، negative errno أو `-EINVAL` لو الـ regulator مش موجود.

**Key details:**
- بتتحقق إن الـ requested voltage في الـ regulator's supported range
- critical لـ UHS-I voltage switching sequence (3.3V → 1.8V)

**Who calls it:** MMC core في `mmc_set_signal_voltage()`.

---

#### `mmc_regulator_enable_vqmmc` / `mmc_regulator_disable_vqmmc`

```c
int mmc_regulator_enable_vqmmc(struct mmc_host *mmc);
void mmc_regulator_disable_vqmmc(struct mmc_host *mmc);
```

بتفعّل/بتعطّل الـ vqmmc regulator مع tracking للـ state في `host->vqmmc_enabled`. بتمنع double-enable/disable errors.

**Key details:**
- `enable` بترجع `0` on success أو negative errno
- الـ `host->vqmmc_enabled` flag بيتحقق منه قبل أي عملية

**Who calls it:** MMC core في power sequences، وhost drivers في suspend/resume.

---

### Group 6: Tuning & Re-tuning

الـ tuning ضروري للـ high-speed modes (UHS-I SDR50/SDR104، eMMC HS200/HS400) عشان يعوض الـ clock phase drift.

---

#### `mmc_send_tuning`

```c
int mmc_send_tuning(struct mmc_host *host, u32 opcode, int *cmd_error);
```

بتبعت tuning block request للـ card (CMD19 للـ SD، CMD21 للـ eMMC) وبتقارن الـ response مع الـ expected pattern. ده اللي بيحدد لو الـ current clock phase valid.

**Parameters:**
- `host` — الـ host
- `opcode` — `MMC_SEND_TUNING_BLOCK` (19) أو `MMC_SEND_TUNING_BLOCK_HS200` (21)
- `cmd_error` — output: error code للـ command phase (nullable)

**Return:** `0` لو الـ tuning block تطابق، negative errno لو في mismatch أو error.

**Pseudocode flow:**
```
allocate tuning_block buffer (64 or 128 bytes depending on bus width)
build mmc_request with DATA_READ
submit to host via mmc_wait_for_req()
compare received data with known-good tuning pattern
if match → return 0
else     → return -EIO
```

**Who calls it:** `mmc_host_ops->execute_tuning()` implementation في الـ host driver، عبر MMC core tuning loop.

---

#### `mmc_send_abort_tuning`

```c
int mmc_send_abort_tuning(struct mmc_host *host, u32 opcode);
```

بتبعت CMD12 (STOP_TRANSMISSION) عشان تـ abort tuning sequence لو فشلت. ضرورية عشان الـ card ترجع لـ transfer state.

**Parameters:**
- `host` — الـ host
- `opcode` — الـ tuning opcode اللي كان بيتنفذ

**Return:** `0` on success، negative errno on failure.

**Who calls it:** MMC core بعد tuning failure.

---

#### `mmc_retune_timer_stop`

```c
void mmc_retune_timer_stop(struct mmc_host *host);
```

بتوقف الـ `host->retune_timer` اللي بيـ trigger periodic re-tuning. لازم تتسمى قبل suspend أو لما بتعمل manual tuning.

**Who calls it:** MMC core في suspend path، وhost drivers لما بيعملوا explicit tuning.

---

#### `mmc_retune_needed` (inline)

```c
static inline void mmc_retune_needed(struct mmc_host *host);
```

بتضبط `host->need_retune = 1` لو الـ `can_retune` flag مضبوط. الـ MMC core بيشوف الـ flag ده عند الـ next request ويعمل re-tune قبل ما يكمل.

**Key details:**
- بتتسمى من ISR لما CRC error يحصل (إشارة إن الـ phase drift تجاوز الحد)
- **لا تضبط `need_retune` مباشرة** — استخدم الـ inline دي

**Who calls it:** host drivers لما يكتشفوا CRC errors.

---

#### `mmc_doing_tune` (inline)

```c
static inline bool mmc_doing_tune(struct mmc_host *host);
```

بترجع `true` لو في initial tuning أو re-tuning شغال حالياً. مهمة لمنع recursive tuning.

**Return:** `host->doing_retune || host->doing_init_tune`

---

### Group 7: SD/eMMC Command Helpers

---

#### `mmc_sd_switch`

```c
int mmc_sd_switch(struct mmc_card *card, bool mode, int group,
                  u8 value, u8 *resp);
```

بتبعت CMD6 (SWITCH_FUNC) للـ SD card. بتستخدم للـ query (mode=0) أو للـ switch (mode=1) لـ function group معين (bus speed، driver type، current limit).

**Parameters:**
- `card` — الـ SD card
- `mode` — `false` للـ query، `true` للـ switch
- `group` — رقم الـ function group (0-5)
- `value` — القيمة المطلوبة للـ group
- `resp` — buffer من 64 bytes للـ switch status response

**Return:** `0` on success، negative errno on failure.

**Who calls it:** MMC core في UHS-I initialization sequence.

---

#### `mmc_send_status`

```c
int mmc_send_status(struct mmc_card *card, u32 *status);
```

بتبعت CMD13 (SEND_STATUS) وبترجع الـ 32-bit card status register. بتستخدم للـ busy polling بعد write operations أو state verification.

**Parameters:**
- `card` — الـ card المستهدفة
- `status` — output: الـ R1 response (card status)

**Return:** `0` on success، `-ETIMEDOUT` أو `-EILSEQ` on failure.

**Key details:**
- الـ status bits المهمة: `R1_READY_FOR_DATA`، `R1_SWITCH_ERROR`، `R1_APP_CMD`
- بتستخدم كـ busy-wait mechanism بعد CMD6 switch

**Who calls it:** MMC core في post-write polling، partition switching، power management.

---

#### `mmc_get_ext_csd`

```c
int mmc_get_ext_csd(struct mmc_card *card, u8 **new_ext_csd);
```

بتقرأ الـ 512-byte Extended CSD register من الـ eMMC card عبر CMD8. بتـ allocate buffer وبترجع pointer ليه.

**Parameters:**
- `card` — الـ eMMC card
- `new_ext_csd` — output pointer — الـ caller مسؤول عن `kfree()` للـ buffer

**Return:** `0` on success، negative errno on failure.

**Key details:**
- الـ EXT_CSD بيحتوي على كل الـ advanced eMMC capabilities: HS200، CQE depth، RPMB size، إلخ
- الـ `card->ext_csd` struct بيتعبى من الـ raw data ده في `mmc_read_ext_csd()`

**Who calls it:** eMMC initialization في `mmc_init_card()`.

---

#### `mmc_read_tuning`

```c
int mmc_read_tuning(struct mmc_host *host, unsigned int blksz,
                    unsigned int blocks);
```

بتقرأ tuning blocks وبتقارنها مع الـ known-good pattern. أدق من `mmc_send_tuning()` — بتدعم multiple blocks.

**Parameters:**
- `host` — الـ host
- `blksz` — حجم الـ block (عادةً 64 أو 128 bytes حسب الـ bus width)
- `blocks` — عدد الـ blocks للقراءة والمقارنة

**Return:** `0` لو الـ pattern match، negative errno on mismatch.

**Who calls it:** custom tuning implementations في host drivers.

---

### Group 8: Accessor Inlines & Macros

الـ group ده صغير بس مهم — بيوفر type-safe وefficient access لأهم fields.

---

#### `mmc_priv` / `mmc_from_priv`

```c
static inline void *mmc_priv(struct mmc_host *host)
{
    return (void *)host->private;
}

static inline struct mmc_host *mmc_from_priv(void *priv)
{
    return container_of(priv, struct mmc_host, private);
}
```

الزوج ده بيوفر الـ host driver private data access. الـ `private[]` flexible array بيجيله extra space في الـ allocation عشان يخزن الـ driver-specific struct.

**مثال:**
```c
/* في probe */
struct my_host_data *priv = mmc_priv(host);
priv->base = devm_ioremap(...);

/* في interrupt handler */
struct my_host_data *priv = mmc_priv(host);
u32 status = readl(priv->base + STATUS_REG);
```

**Key details:**
- `mmc_from_priv()` بتستخدم `container_of` — safe لو الـ `priv` فعلاً من `host->private`
- الـ `private[]` متـ align على cache line لتحسين performance

---

#### `mmc_from_crypto_profile` (inline)

```c
#ifdef CONFIG_MMC_CRYPTO
static inline struct mmc_host *
mmc_from_crypto_profile(struct blk_crypto_profile *profile)
{
    return container_of(profile, struct mmc_host, crypto_profile);
}
#endif
```

بتوصل للـ `mmc_host` من الـ `blk_crypto_profile` — المسار الوحيد من inline encryption callbacks للـ host.

**Who calls it:** الـ MMC crypto implementation عند الـ key programming.

---

#### `mmc_get_dma_dir` (inline)

```c
static inline enum dma_data_direction mmc_get_dma_dir(struct mmc_data *data)
{
    return data->flags & MMC_DATA_WRITE ? DMA_TO_DEVICE : DMA_FROM_DEVICE;
}
```

بتحول الـ `MMC_DATA_WRITE/READ` flags لـ standard `dma_data_direction` enum اللي محتاجه الـ DMA mapping APIs.

**Who calls it:** host drivers في `pre_req()` لـ `dma_map_sg()`.

---

#### `mmc_debugfs_err_stats_inc` (inline)

```c
static inline void mmc_debugfs_err_stats_inc(struct mmc_host *host,
                                              enum mmc_err_stat stat)
{
    host->err_stats[stat] += 1;
}
```

بتزوّد العداد المقابل في `host->err_stats[]` array. الـ array بيتعرض في debugfs عشان الـ error tracking.

**Parameters:**
- `stat` — قيمة من `mmc_err_stat` enum:
  - `MMC_ERR_CMD_TIMEOUT` — command timeout
  - `MMC_ERR_CMD_CRC` — command CRC error
  - `MMC_ERR_DAT_TIMEOUT` — data timeout
  - `MMC_ERR_DAT_CRC` — data CRC error
  - `MMC_ERR_AUTO_CMD` — auto CMD12/23 error
  - `MMC_ERR_ADMA` — ADMA descriptor error
  - `MMC_ERR_TUNING` — tuning failure
  - `MMC_ERR_CMDQ_*` — CQE-specific errors
  - `MMC_ERR_ICE_CFG` — inline encryption config error
  - `MMC_ERR_CTRL_TIMEOUT` — controller internal timeout
  - `MMC_ERR_UNEXPECTED_IRQ` — spurious interrupt

**Who calls it:** host drivers في error interrupt handlers.

---

#### Timing Query Inlines

الـ inlines دول بتقرأ `host->ios.timing` بشكل type-safe:

```c
/* SD HS أو eMMC HS */
static inline int mmc_card_hs(struct mmc_card *card);

/* UHS-I: SDR12 → DDR50 */
static inline int mmc_card_uhs(struct mmc_card *card);

/* UHS-II: Speed A/B + HD variants */
static inline bool mmc_card_uhs2(struct mmc_host *host);

/* UHS-II Half-Duplex mode فقط */
static inline int mmc_card_uhs2_hd_mode(struct mmc_host *host);
```

**Key details:**
- `mmc_card_hs()` و`mmc_card_uhs()` بياخدوا `mmc_card *` — بيوصلوا للـ host عبر `card->host`
- `mmc_card_uhs2()` و`mmc_card_uhs2_hd_mode()` بياخدوا `mmc_host *` مباشرة

---

### Group 9: الـ `mmc_host_ops` Callbacks — شرح تفصيلي

الـ `mmc_host_ops` هي الـ vtable اللي كل host driver لازم يملاها. الـ MMC core بيستدعي الـ callbacks دي — مش host driver.

#### جدول الـ Callbacks

| Callback | Mandatory | Description |
|---|---|---|
| `request` | Yes | submit mmc_request للـ hardware |
| `set_ios` | Yes | ضبط clock/voltage/bus-width |
| `get_ro` | Optional | قراءة write-protect pin |
| `get_cd` | Optional | قراءة card detect pin |
| `enable_sdio_irq` | SDIO | enable/disable SDIO IRQ line |
| `ack_sdio_irq` | Cond. | ack IRQ (مع CAP2_SDIO_IRQ_NOTHREAD) |
| `request_atomic` | Optional | atomic request submission |
| `pre_req` | Optional | DMA pre-mapping لـ double buffering |
| `post_req` | Optional | DMA unmap بعد completion |
| `execute_tuning` | HS modes | run tuning algorithm |
| `start_signal_voltage_switch` | UHS | switch 3.3V→1.8V |
| `card_busy` | Optional | poll DAT0 low للـ busy detection |
| `prepare_hs400_tuning` | HS400 | ضبط ما قبل HS400 tuning |
| `execute_hs400_tuning` | HS400 | تنفيذ HS400 tuning |
| `hs400_prepare_ddr` | HS400 | switch لـ DDR في HS400 init |
| `hs400_downgrade` | HS400 | downgrade لـ HS200 مؤقتاً |
| `hs400_complete` | HS400 | إكمال HS400 selection |
| `hs400_enhanced_strobe` | HS400-ES | ضبط enhanced strobe |
| `select_drive_strength` | UHS/HS200/400 | اختيار driver type A/B/C/D |
| `card_hw_reset` | eMMC | RST_n hardware reset |
| `card_event` | Optional | card insertion/removal event |
| `multi_io_quirk` | Optional | limit multi-block transfers |
| `init_card` | Optional | HC-specific card init quirks |
| `init_sd_express` | SD Express | PCIe init للـ SD Express |
| `uhs2_control` | UHS-II | UHS-II protocol operations |
| `prepare_sd_hs_tuning` | SD HS | prepare SD HS tuning |
| `execute_sd_hs_tuning` | SD HS | run SD HS tuning |

---

#### أهم الـ Callbacks بالتفصيل

##### `request`

```c
void (*request)(struct mmc_host *host, struct mmc_request *req);
```

الـ main submission callback. الـ MMC core بيبعت الـ `mmc_request` للـ host driver ليحطه في الـ hardware queue. الـ function لازم **ترجع فوراً** (async) — الـ completion بتيجي عبر `mmc_request_done()`.

**Key details:**
- بتتسمى مع الـ `host->lock` **غير محجوز** — الـ driver مسؤول عن الـ locking
- الـ `req->sbc` لو موجود = SET_BLOCK_COUNT قبل الـ data command
- لو في error، بيحط الـ error code في `req->cmd->error` أو `req->data->error`

##### `set_ios`

```c
void (*set_ios)(struct mmc_host *host, struct mmc_ios *ios);
```

أهم callback بعد `request`. بتضبط:
- `ios->clock` — الـ clock frequency (0 = disable clock)
- `ios->bus_width` — 1/4/8 bit
- `ios->timing` — legacy/HS/UHS-I/HS200/HS400
- `ios->power_mode` — OFF/UP/ON
- `ios->signal_voltage` — 3.3V/1.8V/1.2V

**Key details:**
- ممكن تتسمى من sleepable context — مش من ISR
- `ios->clock = 0` لازم يوقف الـ SDCLK فعلاً (مش بس ضبط 0Hz)

##### `execute_tuning`

```c
int (*execute_tuning)(struct mmc_host *host, u32 opcode);
```

بتشغّل كامل tuning algorithm للـ host. الـ host driver لازم:
1. يدور على الـ valid clock phase window بتغيير الـ phase
2. يبعت `mmc_send_tuning()` عند كل phase
3. يختار الـ optimal phase في نص الـ valid window

**Return:** `0` لو tuning نجح، negative errno لو فشل.

##### `start_signal_voltage_switch`

```c
int (*start_signal_voltage_switch)(struct mmc_host *host, struct mmc_ios *ios);
```

بتنفّذ الـ 1.8V switching sequence:
1. بتوقف الـ SDCLK
2. بتخفض الـ vqmmc لـ 1.8V
3. بتستنى > 5ms
4. بتشغل الـ SDCLK تاني
5. بتتحقق إن الـ DAT/CMD lines ارتفعت (بـ 1.8V level)

**Return:** `0` on success، `-EAGAIN` لو الـ card مش ready للـ switch.

---

### Group 10: الـ `mmc_cqe_ops` Callbacks

الـ CQE (Command Queue Engine) بيسمح بـ out-of-order request execution — critical لـ eMMC performance في storage scenarios.

#### جدول الـ CQE Callbacks

| Callback | Description |
|---|---|
| `cqe_enable` | allocate CQE resources وتفعيله |
| `cqe_disable` | تعطيل CQE وتحرير الـ resources |
| `cqe_request` | submit request للـ CQE hardware queue |
| `cqe_post_req` | unmap DMA بعد request completion |
| `cqe_off` | suspend CQE لقبول non-CQ commands |
| `cqe_wait_for_idle` | انتظار إن الـ CQE queue تفضى |
| `cqe_timeout` | اكتشاف request timeout |
| `cqe_recovery_start` | بدء error recovery |
| `cqe_recovery_finish` | إنهاء recovery وإعادة الـ requests |

**Pseudocode للـ CQE error recovery flow:**

```
/* من MMC core */
cqe_ops->cqe_recovery_start(host)   /* stop all CQE HW activity */
    ↓
/* abort all queued tasks in HW */
/* send CMD48 (CMDQ_TASK_MGMT) to abort each outstanding task */
    ↓
cqe_ops->cqe_recovery_finish(host)  /* call mmc_cqe_request_done() for each task */
    ↓
/* upper layers (mmc_blk) handle retry */
```

---

### ملاحظة ختامية على الـ Locking Model

الـ MMC host locking بيعتمد على:

1. **`host->lock` (spinlock)** — يحمي `claimed`، `claimer`، `claim_cnt` وأي bus state
2. **`host->wq` (wait_queue)** — الـ `mmc_claim_host()` بتنتظر عليه
3. **الـ `mmc_ctx`** — بيسمح لـ nesting (نفس الـ task يـ claim الـ host أكتر من مرة)

الـ `request()` callback بيتسمى بعد `mmc_claim_host()` — يعني الـ host **محجوز دايماً** لما الـ request يتنفذ. لكن الـ `mmc_detect_change()` بتتسمى بدون claim — ليها مسار خاص عبر الـ `detect` work.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. debugfs Entries

الـ MMC subsystem بيعمل `debugfs_root` لكل host (`mmc_host->debugfs_root`) وكمان لكل card (`mmc_card->debugfs_root`).

```bash
# إظهار كل entries الـ MMC في debugfs
ls /sys/kernel/debug/mmc*/

# الـ entries الأساسية
/sys/kernel/debug/mmc0/
├── ios          # current mmc_ios state (clock, bus_width, timing, voltage...)
├── clock        # actual_clock بعد divider
├── err_stats    # counters لكل enum mmc_err_stat
├── state        # host state machine
└── caps         # host->caps + host->caps2 decoded
```

```bash
# قراءة الـ ios الحالية (bus_width, timing, signal_voltage, power_mode...)
cat /sys/kernel/debug/mmc0/ios

# output مثال:
# clock:          200000000 Hz
# actual clock:   200000000 Hz
# vdd:            21 (3.3 ~ 3.4 V)
# bus mode:       2 (push-pull)
# chip select:    0 (don't care)
# power mode:     2 (on)
# bus width:      3 (8 bits)
# timing spec:    9 (mmc HS200)
# signal voltage: 1 (1.80 V)
# driver type:    0 (driver type B)
```

```bash
# قراءة إحصائيات الأخطاء — بتقابل enum mmc_err_stat مباشرة
cat /sys/kernel/debug/mmc0/err_stats

# output مثال:
# err_stats:
#  CMD Timeout Cnt:    0
#  CMD CRC Err Cnt:    2
#  DAT Timeout Cnt:    0
#  DAT CRC Err Cnt:    1
#  Auto CMD Err Cnt:   0
#  ADMA Err Cnt:       0
#  Tuning Err Cnt:     0
#  CMDQ Red Err Cnt:   0
#  CMDQ GCE Err Cnt:   0
#  CMDQ ICCE Err Cnt:  0
#  Req Timeout Cnt:    0
#  CMDQ Req TO Cnt:    0
#  ICE Cfg Err Cnt:    0
#  CTRL TO Err Cnt:    0
#  Unexpected IRQ Cnt: 0
```

```bash
# قراءة caps المترجمة
cat /sys/kernel/debug/mmc0/caps
```

**الـ card debugfs:**

```bash
ls /sys/kernel/debug/mmc0/mmc0\:0001/
# ├── ext_csd       (raw 512 bytes من EXT_CSD لـ eMMC)
# ├── state         (card state: idle, ready, stby, tran...)
```

```bash
# dump الـ EXT_CSD كاملة (مهمة جداً لـ eMMC debugging)
cat /sys/kernel/debug/mmc0/mmc0\:0001/ext_csd | xxd | head -40
```

---

#### 2. sysfs Entries

```bash
# الـ host device في sysfs
ls /sys/class/mmc_host/mmc0/

# الـ card device
ls /sys/bus/mmc/devices/mmc0\:0001/

# أهم الـ attributes
cat /sys/bus/mmc/devices/mmc0\:0001/name        # product name من CID
cat /sys/bus/mmc/devices/mmc0\:0001/type        # MMC / SD / SDIO
cat /sys/bus/mmc/devices/mmc0\:0001/manfid      # manufacturer ID
cat /sys/bus/mmc/devices/mmc0\:0001/date        # manufacture date
cat /sys/bus/mmc/devices/mmc0\:0001/serial      # serial number
cat /sys/bus/mmc/devices/mmc0\:0001/fwrev       # firmware revision
cat /sys/bus/mmc/devices/mmc0\:0001/hwrev       # hardware revision
cat /sys/bus/mmc/devices/mmc0\:0001/pre_eol_info        # eMMC wear level
cat /sys/bus/mmc/devices/mmc0\:0001/life_time  # device life time estimate

# لو الكارت SDIO
cat /sys/bus/sdio/devices/mmc0\:0001\:1/class
cat /sys/bus/sdio/devices/mmc0\:0001\:1/vendor
cat /sys/bus/sdio/devices/mmc0\:0001\:1/device

# الـ power management
cat /sys/class/mmc_host/mmc0/power/runtime_status
```

---

#### 3. ftrace — Tracepoints والـ Events

**تفعيل tracing للـ MMC:**

```bash
# إظهار كل events المتاحة في الـ MMC subsystem
ls /sys/kernel/tracing/events/mmc/
# mmc_request_start  mmc_request_done  mmc_cmd_rw_start  mmc_cmd_rw_end
# mmc_blk_rw_start   mmc_blk_rw_end    mmc_blk_cmd_start mmc_blk_cmd_done

# تفعيل كل events الـ MMC
echo 1 > /sys/kernel/tracing/events/mmc/enable

# أو event محدد فقط (مثلاً تتبع الـ requests)
echo 1 > /sys/kernel/tracing/events/mmc/mmc_request_start/enable
echo 1 > /sys/kernel/tracing/events/mmc/mmc_request_done/enable

# تفعيل الـ tracing
echo 1 > /sys/kernel/tracing/tracing_on

# تشغيل عملية على الكارت ثم قراءة النتيجة
dd if=/dev/mmcblk0 of=/dev/null bs=4M count=10
cat /sys/kernel/tracing/trace | head -50

# إيقاف
echo 0 > /sys/kernel/tracing/tracing_on
echo 0 > /sys/kernel/tracing/events/mmc/enable
```

```bash
# تتبع function calls داخل الـ MMC core
echo mmc_start_request > /sys/kernel/tracing/set_ftrace_filter
echo mmc_request_done >> /sys/kernel/tracing/set_ftrace_filter
echo mmc_execute_tuning >> /sys/kernel/tracing/set_ftrace_filter
echo function > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/tracing_on
```

```bash
# تتبع tuning بشكل مخصص
echo 1 > /sys/kernel/tracing/events/mmc/mmc_execute_tuning/enable 2>/dev/null || true
# أو استخدام function_graph لرؤية call tree كامل
echo function_graph > /sys/kernel/tracing/current_tracer
echo mmc_execute_tuning > /sys/kernel/tracing/set_graph_function
```

---

#### 4. printk والـ Dynamic Debug

```bash
# تفعيل dynamic debug لكل الـ MMC subsystem
echo 'module mmc_core +p' > /sys/kernel/debug/dynamic_debug/control
echo 'module sdhci +p' > /sys/kernel/debug/dynamic_debug/control
echo 'module mmc_block +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل لملف محدد فقط
echo 'file drivers/mmc/core/core.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/mmc/host/sdhci.c +pmfl' > /sys/kernel/debug/dynamic_debug/control
# p = printk, m = module name, f = function name, l = line number

# تفعيل لـ function محددة
echo 'func mmc_select_timing +p' > /sys/kernel/debug/dynamic_debug/control
echo 'func mmc_execute_tuning +p' > /sys/kernel/debug/dynamic_debug/control

# رفع loglevel مؤقتاً لرؤية debug messages
echo 8 > /proc/sys/kernel/printk

# أو عبر dmesg
dmesg -n 8

# تابع الـ kernel log في real-time
dmesg -w | grep -i mmc
```

**تفعيل verbose logging عبر kernel cmdline:**

```
# في /etc/default/grub أو bootloader
GRUB_CMDLINE_LINUX="... mmc_core.use_spi_crc=1 sdhci.debug_quirks=0x40"
```

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الوصف |
|---|---|
| `CONFIG_MMC_DEBUG` | يفعّل `pr_debug()` calls في الـ MMC core |
| `CONFIG_FAIL_MMC_REQUEST` | يفعّل `fault_attr` لـ fault injection (`mmc_host->fail_mmc_request`) |
| `CONFIG_MMC_CRYPTO` | يفعّل inline encryption debugging |
| `CONFIG_DEBUG_FS` | شرط لظهور `/sys/kernel/debug/mmc*` |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل `pr_debug()` dynamic control |
| `CONFIG_MMC_SDHCI_PCI` | لو بتستخدم SDHCI على PCI |
| `CONFIG_LOCK_STAT` | لتتبع spinlock contentions على `mmc_host->lock` |
| `CONFIG_KASAN` | لكشف memory issues في host drivers |
| `CONFIG_PROVE_LOCKING` | يكشف deadlocks في claim/bus lock |

```bash
# التحقق من الـ configs المفعّلة في kernel الحالي
grep -E 'CONFIG_MMC|CONFIG_FAIL_MMC|CONFIG_SDHCI' /boot/config-$(uname -r)
zcat /proc/config.gz 2>/dev/null | grep -E 'CONFIG_MMC|FAIL_MMC'
```

**تفعيل fault injection (يحتاج `CONFIG_FAIL_MMC_REQUEST`):**

```bash
# إرغام الـ kernel على inject errors في requests المرسلة للـ mmc
echo 1 > /sys/kernel/debug/mmc0/fail_mmc_request/probability
echo 1 > /sys/kernel/debug/mmc0/fail_mmc_request/times
# بيستخدم mmc_host->fail_mmc_request مباشرة
```

---

#### 6. أدوات خاصة بالـ MMC Subsystem

```bash
# mmc-utils — الأداة الرسمية لـ eMMC management وdebugging
apt install mmc-utils   # أو yum install mmc-utils

# قراءة EXT_CSD مباشرة من user-space
mmc extcsd read /dev/mmcblk0

# قراءة معلومات الكارت
mmc info /dev/mmcblk0

# قراءة الـ CID (manufacturer info)
# output: يتضمن manfid, prod_name, fwrev مطابق لـ mmc_cid struct

# فحص life time estimation (يقابل ext_csd.device_life_time_est_typ_a/b)
mmc extcsd read /dev/mmcblk0 | grep -i "life time"
# EXT_CSD[268]: Device life time estimation type A: 0x01 (used 0-10%)
# EXT_CSD[269]: Device life time estimation type B: 0x02 (used 10-20%)

# فحص pre-EOL info (يقابل ext_csd.pre_eol_info)
mmc extcsd read /dev/mmcblk0 | grep -i "pre-eol"

# إرسال hardware reset عبر RST_n pin (يستدعي mmc_host_ops->card_hw_reset)
mmc hwreset enable /dev/mmcblk0
mmc hwreset send /dev/mmcblk0

# تفعيل/تعطيل cache (ext_csd.cache_ctrl)
mmc cache enable /dev/mmcblk0
mmc cache disable /dev/mmcblk0

# فحص write protect
mmc writeprotect get /dev/mmcblk0

# لـ SD cards: عرض تفاصيل UHS capabilities
cat /sys/bus/mmc/devices/mmc0\:*/ssr    # SD Status Register
```

```bash
# sdparm — لـ SD/MMC عبر SCSI emulation لو متاح
sdparm --page=0x3f /dev/sdb 2>/dev/null

# لينكس blktrace — لتتبع I/O على مستوى block layer
blktrace -d /dev/mmcblk0 -o - | blkparse -i -
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة في kernel log | المعنى | الحل |
|---|---|---|
| `mmc0: error -110 whilst initialising SD card` | timeout أثناء init (ETIMEDOUT) | تحقق من الطاقة/الكارت نفسه/السرعة |
| `mmc0: error -84 whilst initialising MMC card` | EILSEQ — CRC error في response | تحقق من PCB traces، جرب تخفيض clock |
| `mmc0: tuning execution failed` | فشل `execute_tuning` — مشكلة في signal quality | راجع clock phase، قلل frequency |
| `mmc0: CMD13 error -110` | SEND_STATUS timeout — الكارت مش بيرد | power cycle، تحقق VCCQ |
| `mmc0: data timeout` | DAT timeout — يقابل `MMC_ERR_DAT_TIMEOUT` | تحقق DMA config، clock، signal integrity |
| `mmc0: Got data interrupt 0x... even though no data operation was in progress` | `MMC_ERR_UNEXPECTED_IRQ` | bug في host driver أو hardware glitch |
| `mmc0: ADMA error` | `MMC_ERR_ADMA` — خطأ في ADMA descriptor | تحقق DMA buffer alignment، memory corruption |
| `mmc0: running CQE recovery` | CQE error — `MMC_ERR_CMDQ_RED/GCE/ICCE` | تحقق من eMMC firmware version |
| `mmc0: Card stuck in wrong state! (R1 0x...)` | الكارت في state غلط | hardware reset، تحقق power supply stability |
| `mmc0: switching to high-speed failed` | فشل CMD6 switch | تحقق VCC/VCCQ levels، تحقق mmc_ios.timing |
| `mmc0: Timeout waiting for hardware interrupt` | `MMC_ERR_CTRL_TIMEOUT` | bug في host driver ISR أو hardware hang |
| `mmc0: voltage regulator failed to switch` | فشل `start_signal_voltage_switch` | تحقق regulator vmmc/vqmmc في DT |
| `mmc0: card does not support SDIO` | لو `MMC_CAP2_NO_SDIO` مش مضبوط صح | حدد capabilities في DT أو platform data |
| `mmc0: re-tuning is needed but has been disabled` | retune_paused=1 ومحتاج retune | راجع host driver logic لـ retune_paused |

---

#### 8. نقاط استراتيجية لـ dump_stack() وWARN_ON()

```c
/* في host driver — التحقق من صحة الـ request قبل تنفيذه */
void my_host_request(struct mmc_host *host, struct mmc_request *mrq)
{
    /* تأكد إن الكارت موجود */
    WARN_ON(!host->card);

    /* تأكد إن الـ host مش في state غلط */
    WARN_ON(host->ios.power_mode == MMC_POWER_OFF);

    /* تحقق من الـ clock — لازم يكون > 0 عند transfer */
    if (mrq->data) {
        WARN_ON(host->ios.clock == 0);
    }
    ...
}

/* في set_ios — تحقق من قيم التوقيت */
void my_host_set_ios(struct mmc_host *host, struct mmc_ios *ios)
{
    /* التحقق من timing range */
    WARN_ON(ios->timing > MMC_TIMING_SD_EXP_1_2V);

    /* التحقق من bus_width consistency مع caps */
    if (ios->bus_width == MMC_BUS_WIDTH_8)
        WARN_ON(!(host->caps & MMC_CAP_8_BIT_DATA));
    ...
}

/* في CQE operations — تحقق من الـ depth */
int my_cqe_request(struct mmc_host *host, struct mmc_request *mrq)
{
    /* cqe_qdepth لازم يكون > 0 */
    WARN_ON(host->cqe_qdepth <= 0);

    /* التحقق إن CQE enabled قبل إرسال request */
    if (WARN_ON(!host->cqe_on)) {
        dump_stack(); /* رؤية من فين جاءت الـ call */
        return -EINVAL;
    }
    ...
}

/* في tuning — نقطة مهمة لتشخيص فشل الـ tuning */
int my_execute_tuning(struct mmc_host *host, u32 opcode)
{
    int ret;
    /* سجّل الـ phase map قبل الـ tuning */
    dev_dbg(mmc_dev(host), "Starting tuning, current timing=%d clock=%d\n",
            host->ios.timing, host->ios.clock);

    ret = do_tuning(host, opcode);
    if (ret) {
        /* اطبع dump لتتبع call chain لما يفشل الـ tuning */
        dev_err(mmc_dev(host), "Tuning failed at phase %d\n", current_phase);
        dump_stack();
    }
    return ret;
}

/* تحقق من الـ err_stats counter لما يتجاوز حد معين */
static void check_error_threshold(struct mmc_host *host)
{
    if (host->err_stats[MMC_ERR_DAT_CRC] > 100)
        WARN_ONCE(1, "mmc%d: excessive DAT CRC errors (%u), check signal integrity\n",
                  host->index, host->err_stats[MMC_ERR_DAT_CRC]);
}
```

---

### Hardware Level

---

#### 1. التحقق من أن الـ Hardware State يطابق Kernel State

```bash
# مقارنة الـ clock المضبوط في mmc_ios مع الـ actual_clock
cat /sys/kernel/debug/mmc0/ios | grep -E "clock|actual"
# clock:         200000000 Hz   ← ما طلبه الـ kernel (ios->clock)
# actual clock:  200000000 Hz   ← mmc_host->actual_clock

# الفرق بين الاثنين يعني الـ host controller عمل تقريب في الـ clock divider

# التحقق من bus_width — لازم يطابق عدد D lines المتوصلة
cat /sys/kernel/debug/mmc0/ios | grep "bus width"
# bus width: 3 (8 bits)   ← MMC_BUS_WIDTH_8

# التحقق من الـ voltage عبر regulator sysfs
cat /sys/class/regulator/*/name | grep -i mmc
ls /sys/class/regulator/
cat /sys/class/regulator/regulator.X/microvolts   # X = رقم الـ regulator

# التحقق من power mode
cat /sys/kernel/debug/mmc0/ios | grep power
# power mode: 2 (on)   ← MMC_POWER_ON

# للتحقق من الـ signal voltage switch نجح
# 1.8V switch → ios->signal_voltage == MMC_SIGNAL_VOLTAGE_180
cat /sys/kernel/debug/mmc0/ios | grep "signal voltage"
```

---

#### 2. Register Dump Techniques

```bash
# تحديد الـ base address من device tree أو /proc/iomem
cat /proc/iomem | grep -i sdhci
cat /proc/iomem | grep -i mmc
# output مثال:
# fe340000-fe3401ff : fe340000.mmc

# قراءة registers باستخدام devmem2
# (يحتاج تعطيل IOMMU أو direct memory access)
apt install devmem2

BASE=0xfe340000  # base address من /proc/iomem

# SDHCI standard register offsets
devmem2 $((BASE + 0x00)) w   # SDMA System Address
devmem2 $((BASE + 0x04)) w   # Block Size + Count
devmem2 $((BASE + 0x0C)) w   # Transfer Mode + Command
devmem2 $((BASE + 0x10)) w   # Response[0]
devmem2 $((BASE + 0x24)) w   # Present State Register
devmem2 $((BASE + 0x28)) b   # Host Control 1
devmem2 $((BASE + 0x29)) b   # Power Control
devmem2 $((BASE + 0x2C)) w   # Clock Control
devmem2 $((BASE + 0x30)) w   # Interrupt Status
devmem2 $((BASE + 0x34)) w   # Interrupt Enable
devmem2 $((BASE + 0x38)) w   # Signal Enable
devmem2 $((BASE + 0x3C)) w   # Auto CMD Error / Host Control 2
devmem2 $((BASE + 0x40)) w   # Capabilities (Low)
devmem2 $((BASE + 0x44)) w   # Capabilities (High)
```

```bash
# الطريقة الأسهل: io utility من package of util-linux
io -4 -r $((0xfe340000 + 0x24))   # Present State Register

# أو عبر /dev/mem مع python
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    base = 0xfe340000
    m = mmap.mmap(f.fileno(), 0x200, offset=base)
    ps = struct.unpack('<I', m[0x24:0x28])[0]
    print(f'Present State: 0x{ps:08X}')
    print(f'  CMD line:    {(ps>>0)&1}')
    print(f'  DAT[0..3]:   {(ps>>20)&0xF:04b}')
    print(f'  Card Present:{(ps>>16)&1}')
    print(f'  Card Stable: {(ps>>17)&1}')
    m.close()
"
```

**Present State Register — أهم register للـ debugging:**

```
Bit 0:  CMD line signal
Bit 1:  DAT[0] signal (busy detection)
Bit 2:  DAT[1] signal
Bit 3:  DAT[2] signal
Bit 4:  DAT[3] signal
Bit 16: Card Inserted
Bit 17: Card State Stable
Bit 18: Card Detect Pin Level
Bit 19: Write Protect Switch Pin Level
Bit 20-23: DAT[0-3] Line Signal Level
Bit 24: CMD Line Signal Level
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

**نقاط القياس الأساسية:**

```
الـ eMMC / SD Card Interface:
  - CLK  → قس على هذا الـ pin: يجب أن يكون نظيف بدون jitter
  - CMD  → bidirectional: command من host، response من card
  - DAT[0..3] أو DAT[0..7] → data lines
  - VCC  → 3.3V أو 1.8V حسب timing mode
  - VCCQ → I/O voltage: 3.3V (legacy) / 1.8V (UHS / HS200) / 1.2V
  - RST_n → active-low hardware reset (يقابل card_hw_reset callback)
```

**ما تبحث عنه في الـ Logic Analyzer:**

```
1. Voltage Switch Sequence:
   - CMD11 يُرسَل (قبل switch)
   - CLK يوقف
   - VDD تنزل لـ 1.8V
   - CLK يبدأ تاني
   - لو CLK ما وقفش → فشل switch = kernel log: "voltage switching failed"

2. Tuning Sequence (HS200/HS400):
   - CMD21 (eMMC) أو CMD19 (SD) تُرسَل مرات متعددة
   - الـ host بيغير phase وبيقيس response
   - ابحث عن CRC errors في responses لتحديد best phase
   - mmc_clk_phase_map بتخزن النتيجة

3. CMD Timeout:
   - بعد CMD يُرسَل، انتظر response window (NRC = 8 clocks)
   - لو ما فيش response = ETIMEDOUT
   - اقيس إذا DAT lines خارجة من الكارت أصلاً

4. DAT CRC Errors:
   - قيس شكل العين (eye diagram) على DAT lines بالـ oscilloscope
   - عند HS200: 200MHz، مطلوب eye opening >= 0.5UI
   - تحقق من impedance matching وطول الـ traces
```

**إعدادات Logic Analyzer موصى بها:**

```
- Sample Rate: × 4 من clock frequency على الأقل
  مثال: HS200 = 200MHz → sample rate >= 800MHz
- Voltage threshold: 0.9V لـ 1.8V signaling
- Protocol decoder: MMC/SD decoder لو متاح في Saleae/Sigrok
```

```bash
# sigrok / pulseview — open source logic analyzer tool
# تسجيل MMC protocol
sigrok-cli -d fx2lafw --config samplerate=24m \
    --channels D0=CMD,D1=CLK,D2=DAT0 \
    --time 500ms -o capture.sr
```

---

#### 4. Hardware Issues الشائعة وأنماط الـ Kernel Log

| المشكلة الهاردوير | نمط الـ kernel log | التشخيص |
|---|---|---|
| سوء توصيل DAT lines | `mmc0: error -84 ... DAT CRC` متكرر | قيس impedance، تحقق solder joints |
| ضعف الطاقة VCC | `mmc0: card init failed -110` عند cold boot | قيس VCC بـ oscilloscope أثناء init، تحقق من inrush current |
| VCCQ مش بيتحول لـ 1.8V | `mmc0: voltage switch failed` | تحقق regulator في DT، قيس VCCQ pin |
| CLK jitter عالي | Tuning fails — `mmc0: tuning execution failed` | قيس CLK jitter (يجب < 100ps لـ HS200) |
| RST_n مش شغال | Hang بعد `card_hw_reset` | تحقق GPIO direction وpolarity في DT |
| card detect broken | مش بيشوف الكارت أو false detects | تحقق cd_irq في `mmc_slot->cd_irq` وpolarity `MMC_CAP2_CD_ACTIVE_HIGH` |
| Write protect pin stuck | `mmc0: card is read-only` رغم إن مش read-only | تحقق RO pin، ممكن تعطّل بـ `MMC_CAP2_NO_WRITE_PROTECT` |
| ADMA descriptor corruption | `mmc0: ADMA error` مع DMA faults | تحقق memory alignment، IOMMU config |

---

#### 5. Device Tree Debugging

```bash
# عرض الـ DT node الخاص بالـ MMC host
# أولاً: إيجاد الـ node
find /proc/device-tree/ -name "*mmc*" -o -name "*sdhci*" 2>/dev/null

# قراءة الـ properties
cat /proc/device-tree/soc/mmc@fe340000/compatible | tr '\0' '\n'
cat /proc/device-tree/soc/mmc@fe340000/bus-width   | xxd
cat /proc/device-tree/soc/mmc@fe340000/max-frequency | xxd

# مقارنة DT values مع ما parse الـ kernel (mmc_of_parse)
# mmc_of_parse() بتملي host->f_max, caps, caps2 من الـ DT
# التحقق:
cat /sys/kernel/debug/mmc0/ios | grep clock
# vs ما في DT:
cat /proc/device-tree/soc/mmc@fe340000/max-frequency | \
    python3 -c "import sys; d=sys.stdin.buffer.read(); print(int.from_bytes(d,'big'))"
```

**الـ DT properties الأساسية التي تؤثر على `mmc_host`:**

```dts
/* مثال DT node صحيح لـ eMMC */
mmc0: mmc@fe340000 {
    compatible = "rockchip,rk3399-dw-mshc";
    reg = <0x0 0xfe340000 0x0 0x4000>;
    interrupts = <GIC_SPI 12 IRQ_TYPE_LEVEL_HIGH>;

    /* يُملي host->f_max = 150000000 */
    max-frequency = <150000000>;

    /* يضبط MMC_CAP_8_BIT_DATA */
    bus-width = <8>;

    /* يضبط MMC_CAP_NONREMOVABLE */
    non-removable;

    /* يضبط MMC_CAP2_HS200 */
    mmc-hs200-1_8v;

    /* يضبط MMC_CAP2_HS400 */
    mmc-hs400-1_8v;

    /* يضبط clock phases في mmc_clk_phase_map */
    mmc-hs400-enhanced-strobe;

    /* vmmc/vqmmc regulators — يُملي mmc_supply struct */
    vmmc-supply = <&vcc_sd>;
    vqmmc-supply = <&vccio_sd>;

    /* pinctrl للـ signal integrity */
    pinctrl-names = "default";
    pinctrl-0 = <&emmc_clk &emmc_cmd &emmc_bus8>;
};
```

```bash
# التحقق من clock phase parsing (mmc_of_parse_clk_phase)
# بيملي mmc_clk_phase_map->phase[timing].in_deg و out_deg
grep -r "phase" /proc/device-tree/soc/mmc@fe340000/ 2>/dev/null | xxd

# التحقق من الـ regulator binding صح
cat /proc/device-tree/soc/mmc@fe340000/vmmc-supply | xxd
# المرجع لازم يشاور على phandle صح لـ regulator node

# dtc لـ dump الـ DT كله بشكل readable
dtc -I fs /proc/device-tree/ 2>/dev/null | grep -A 20 "mmc@fe340000"
```

---

### Practical Commands

---

#### Script جاهز للتشخيص الشامل

```bash
#!/bin/bash
# mmc-debug.sh — comprehensive MMC subsystem debug script

MMC_HOST="${1:-mmc0}"
echo "=== MMC Debug Report for $MMC_HOST ==="
echo "Date: $(date)"
echo ""

echo "--- 1. Current IOS State ---"
cat /sys/kernel/debug/$MMC_HOST/ios 2>/dev/null || echo "debugfs not available"

echo ""
echo "--- 2. Error Statistics ---"
cat /sys/kernel/debug/$MMC_HOST/err_stats 2>/dev/null || echo "no err_stats"

echo ""
echo "--- 3. Host Capabilities ---"
cat /sys/kernel/debug/$MMC_HOST/caps 2>/dev/null || echo "no caps file"

echo ""
echo "--- 4. Card Info (sysfs) ---"
for f in /sys/bus/mmc/devices/${MMC_HOST}*/; do
    echo "Device: $f"
    for attr in name type manfid date serial fwrev hwrev pre_eol_info; do
        val=$(cat "$f/$attr" 2>/dev/null)
        [ -n "$val" ] && echo "  $attr: $val"
    done
done

echo ""
echo "--- 5. Recent Kernel Messages ---"
dmesg | grep -i "mmc\|sdhci\|mmcblk" | tail -30

echo ""
echo "--- 6. Block Device Info ---"
for dev in /dev/mmcblk[0-9]; do
    [ -b "$dev" ] && echo "$dev: $(blockdev --getsize64 $dev 2>/dev/null) bytes"
done

echo ""
echo "--- 7. Runtime PM Status ---"
for host in /sys/class/mmc_host/*/; do
    echo "$(basename $host): $(cat $host/power/runtime_status 2>/dev/null)"
done
```

```bash
# تشغيل الـ script
chmod +x mmc-debug.sh
./mmc-debug.sh mmc0
```

#### قراءة EXT_CSD وتفسيرها

```bash
# dump EXT_CSD كاملة وتفسير الـ bytes المهمة
mmc extcsd read /dev/mmcblk0 2>/dev/null | tee /tmp/extcsd.txt

# استخراج life time estimation
grep -i "life time\|pre.eol" /tmp/extcsd.txt

# output مثال:
# EXT_CSD[267]: Pre EOL information [PRE_EOL_INFO]: 0x01
#  i.e. Normal (< 80% reserved blocks used)
# EXT_CSD[268]: Device life time estimation type A: 0x02
#  i.e. Used from 10% to 20% of the reserved blocks
# EXT_CSD[269]: Device life time estimation type B: 0x01
#  i.e. Used from 0% to 10% of the reserved blocks
```

#### تفعيل وقراءة الـ ftrace بشكل عملي

```bash
#!/bin/bash
# ftrace-mmc.sh — تسجيل MMC events لمدة 5 ثواني

TRACE=/sys/kernel/tracing
echo nop > $TRACE/current_tracer
echo 0 > $TRACE/tracing_on
echo > $TRACE/trace

# فعّل events
echo 1 > $TRACE/events/mmc/enable

echo 1 > $TRACE/tracing_on
sleep 5
echo 0 > $TRACE/tracing_on

# احفظ النتيجة
cp $TRACE/trace /tmp/mmc-trace-$(date +%s).txt
echo "Trace saved to /tmp/mmc-trace-*.txt"

# عطّل events
echo 0 > $TRACE/events/mmc/enable
```

#### مقارنة الـ actual_clock مع المطلوب

```bash
# هذا يكشف إذا كان الـ clock divider صح
python3 - << 'EOF'
import subprocess
result = subprocess.run(['cat', '/sys/kernel/debug/mmc0/ios'],
                        capture_output=True, text=True)
lines = result.stdout.splitlines()
clocks = {l.split(':')[0].strip(): l.split(':')[1].strip()
          for l in lines if ':' in l}
req = clocks.get('clock', 'N/A')
act = clocks.get('actual clock', 'N/A')
print(f"Requested: {req}")
print(f"Actual:    {act}")
if req != 'N/A' and act != 'N/A':
    r = int(req.split()[0])
    a = int(act.split()[0])
    diff_pct = abs(r-a)/r*100 if r > 0 else 0
    print(f"Deviation: {diff_pct:.2f}%")
    if diff_pct > 5:
        print("WARNING: Large clock deviation — check clock divider")
EOF
```

#### تفسير Present State Register بسرعة

```bash
# قراءة وتفسير Present State Register (offset 0x24 في SDHCI)
BASE=$(grep -i sdhci /proc/iomem | head -1 | awk '{print $1}' | cut -d- -f1)
if [ -n "$BASE" ]; then
    PS=$(devmem2 $((16#$BASE + 0x24)) w 2>/dev/null | grep "Read at" | awk '{print $NF}')
    echo "Present State Register: $PS"
    PS_DEC=$((PS))
    echo "  CMD Inhibit (CMD): $(( (PS_DEC >> 0) & 1 ))"
    echo "  CMD Inhibit (DAT): $(( (PS_DEC >> 1) & 1 ))"
    echo "  Card Inserted:     $(( (PS_DEC >> 16) & 1 ))"
    echo "  Card State Stable: $(( (PS_DEC >> 17) & 1 ))"
    echo "  WP Switch Level:   $(( (PS_DEC >> 19) & 1 ))"
    echo "  DAT[0-3] levels:   $(( (PS_DEC >> 20) & 0xF ))"
else
    echo "Could not find SDHCI base address in /proc/iomem"
fi
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: eMMC على RK3562 — فشل التهيئة بسبب timing خاطئ في HS400

#### العنوان
**Industrial Gateway بيحاول يبوت من eMMC بس الـ kernel بيـ hang أثناء MMC init**

#### السياق
بورد industrial gateway مبنية على **RK3562** (Rockchip)، eMMC سعتها 32GB موصولة على `mmc0`. المنتج بيستخدم kernel مخصص، وفي أثناء bring-up الأولي الـ bootloader شغال تمام بس الـ Linux kernel بيـ panic أثناء init الـ eMMC.

#### المشكلة
الـ kernel بيطبع:
```
mmc0: error -110 whilst initialising MMC card
mmc0: CMD_TIMEOUT during HS400 tuning
```
الـ eMMC بتدعم `MMC_TIMING_MMC_HS400` بس الـ host driver مش بيكمّل الـ tuning sequence.

#### التحليل
بنفتح `host.h` ونتتبع:

```c
/* في struct mmc_host_ops */
int  (*prepare_hs400_tuning)(struct mmc_host *host, struct mmc_ios *ios);
int  (*execute_hs400_tuning)(struct mmc_host *host, struct mmc_card *card);
void (*hs400_downgrade)(struct mmc_host *host);
void (*hs400_complete)(struct mmc_host *host);
void (*hs400_enhanced_strobe)(struct mmc_host *host, struct mmc_ios *ios);
```

الـ core بيتوقع تسلسل محدد:
1. `prepare_hs400_tuning()` — تهيئة الـ PHY قبل الـ tuning
2. `execute_tuning()` — إرسال CMD21
3. `hs400_complete()` — تثبيت الـ strobe

الـ driver الخاص بالـ RK3562 كان `prepare_hs400_tuning` بيرجع 0 من غير ما يضبط الـ `mmc_clk_phase_map`:

```c
struct mmc_clk_phase_map {
    struct mmc_clk_phase phase[MMC_NUM_CLK_PHASES]; /* 11 phases */
};

struct mmc_clk_phase {
    bool valid;   /* لو false، الـ core بيتجاهل الـ phase دي */
    u16  in_deg;  /* input clock phase in degrees */
    u16  out_deg; /* output clock phase in degrees */
};
```

`MMC_NUM_CLK_PHASES` = `MMC_TIMING_MMC_HS400 + 1` = 11، يعني لازم كل phase من 0 لـ 10 تكون `valid = true` مع قيم صح. الـ driver كان بيـ zero-initialize الـ map من غير ما يحدد القيم، فـ`valid` بقت `false` لكل الـ phases.

```c
/* الكود الغلط في driver */
static int rk3562_mmc_prepare_hs400_tuning(struct mmc_host *host,
                                            struct mmc_ios *ios)
{
    struct rk3562_host *priv = mmc_priv(host); /* priv data via host->private[] */
    /* BUG: phase_map مش بتتحدد، valid = false */
    mmc_of_parse_clk_phase(&host->class_dev.parent->of_node, &priv->phase_map);
    return 0;
}
```

#### الحل
لازم نـ parse الـ clock phases من الـ Device Tree صح وعلامة `valid`:

```c
/* في .dts */
&mmc0 {
    clk-phase-legacy      = <0 90>;
    clk-phase-mmc-hs      = <0 90>;
    clk-phase-mmc-hs200   = <0 180>;
    clk-phase-mmc-hs400   = <90 180>;  /* in_deg=90, out_deg=180 */
    /* ... باقي الـ phases */
};
```

```c
/* الكود الصح في driver */
static int rk3562_mmc_prepare_hs400_tuning(struct mmc_host *host,
                                            struct mmc_ios *ios)
{
    struct rk3562_host *priv = mmc_priv(host);
    /* mmc_of_parse_clk_phase تملأ phase_map وتضبط valid=true */
    mmc_of_parse_clk_phase(host->parent, &priv->phase_map);
    return 0;
}
```

**أوامر debug مفيدة:**
```bash
# شوف الـ timing الحالي
cat /sys/kernel/debug/mmc0/ios

# شوف error stats
cat /sys/kernel/debug/mmc0/err_stats

# شوف actual_clock
cat /sys/class/mmc_host/mmc0/device/actual_clock
```

#### الدرس المستفاد
الـ `mmc_clk_phase_map` في `host.h` محتاجة كل الـ phases تكون `valid = true` مع قيم صح من الـ DT. لو حقل `valid` بقى `false`، الـ core بيتجاهل الـ phase دي خالص وممكن الـ HS400 tuning يفشل بـ timeout.

---

### السيناريو 2: SD Card على STM32MP1 — بطء شديد بسبب عدم تفعيل UHS

#### العنوان
**IoT sensor node بيقرأ SD card بـ 12 MB/s بدل 80+ MB/s**

#### السياق
جهاز IoT مبني على **STM32MP1** (STMicroelectronics)، بيسجّل بيانات على SD card بسرعة عالية. الـ card بتدعم UHS-I SDR104 (حتى 104 MHz)، بس الـ system بيتصرف كأنها SD HS فقط (50 MHz).

#### المشكلة
```bash
$ dmesg | grep mmc1
mmc1: new high-speed SDHC card at address 1234
```
الرسالة بتقول `high-speed` مش `ultra high speed`، يعني الـ UHS ما اتفعّلش.

#### التحليل
في `mmc_host`:
```c
u32 caps; /* Host capabilities */

#define MMC_CAP_UHS_SDR12   (1 << 16)
#define MMC_CAP_UHS_SDR25   (1 << 17)
#define MMC_CAP_UHS_SDR50   (1 << 18)
#define MMC_CAP_UHS_SDR104  (1 << 19)
#define MMC_CAP_UHS_DDR50   (1 << 20)
#define MMC_CAP_UHS         (MMC_CAP_UHS_SDR12 | MMC_CAP_UHS_SDR25 | \
                             MMC_CAP_UHS_SDR50 | MMC_CAP_UHS_SDR104 | \
                             MMC_CAP_UHS_DDR50)
```

الـ UHS بيحتاج كمان voltage switching:
```c
/* في struct mmc_ios */
unsigned char signal_voltage; /* MMC_SIGNAL_VOLTAGE_330 أو MMC_SIGNAL_VOLTAGE_180 */

/* في struct mmc_host_ops */
int (*start_signal_voltage_switch)(struct mmc_host *host, struct mmc_ios *ios);
```

الـ core بيتحقق من `MMC_CAP_UHS` في الـ `caps`، وبعدين بيحاول voltage switch من 3.3V لـ 1.8V عن طريق `start_signal_voltage_switch`. لو الـ driver ما ضبطش الـ `caps` أو ما نفّذش الـ callback صح، الـ core بيقع back على SD HS.

الـ supply في `mmc_host`:
```c
struct mmc_supply supply;
/* اللي فيها */
struct regulator *vqmmc; /* Optional Vccq supply — لازم يكون موجود للـ UHS */
```

المشكلة كانت في الـ DTS: الـ `vqmmc` regulator ما كانش متعرّف:

```dts
/* DTS غلط */
&sdmmc1 {
    vmmc-supply = <&v3v3>;
    /* vqmmc-supply ناقص! */
    bus-width = <4>;
};
```

بدون `vqmmc`، الـ `mmc_regulator_set_vqmmc()` بيرجع error فـ voltage switch بيفشل، والـ card بتشتغل على 3.3V فقط = لا UHS.

#### الحل
```dts
/* DTS صح */
&sdmmc1 {
    vmmc-supply  = <&v3v3>;
    vqmmc-supply = <&v1v8_sd>; /* regulator بيدعم 1.8V */
    bus-width = <4>;
    cap-sd-highspeed;
    sd-uhs-sdr104;
    sd-uhs-sdr50;
    sd-uhs-ddr50;
};
```

```c
/* في driver probe، تأكد من الـ caps */
host->caps |= MMC_CAP_UHS_SDR104 | MMC_CAP_UHS_SDR50 |
              MMC_CAP_UHS_DDR50  | MMC_CAP_4_BIT_DATA;
```

**تحقق بعد الإصلاح:**
```bash
$ dmesg | grep mmc1
mmc1: new ultra high speed SDR104 SDHC card at address 1234
$ cat /sys/kernel/debug/mmc1/ios | grep timing
timing spec:    6 (mmc-hs200)  # أو sd-uhs-sdr104
```

#### الدرس المستفاد
الـ `MMC_CAP_UHS_*` flags في `host->caps` مش كافية لوحدها — لازم يكون `vqmmc` regulator موجود في الـ DTS ومتعرّف صح. بدونه الـ `mmc_supply.vqmmc` بيبقى NULL وكل محاولة voltage switch بتفشل صامتة.

---

### السيناريو 3: eMMC على i.MX8 — data corruption بسبب HS400 Enhanced Strobe

#### العنوان
**Android TV Box بيعمل random filesystem corruption على eMMC بعد ساعات من الاستخدام**

#### السياق
Android TV box مبني على **i.MX8MM** (NXP)، eMMC جيل 5.1 بتدعم HS400 Enhanced Strobe. الجهاز شغال بشكل طبيعي في أول الشغل بس بعد 4-8 ساعات تقريباً بيظهر `EXT4-fs error` أو `mmc0: error -84` في الـ logcat.

#### المشكلة
```
mmc0: unrecognised response type 0x27
mmc0: DAT_CRC error during HS400ES operation
```

الـ eMMC بتشتغل على HS400ES بس الـ strobe calibration غلط.

#### التحليل
في `struct mmc_ios`:
```c
bool enhanced_strobe; /* hs400es selection — لازم تتعامل معاها صح */
```

في `struct mmc_host_ops`:
```c
void (*hs400_enhanced_strobe)(struct mmc_host *host, struct mmc_ios *ios);
```

الـ flow الصح لـ HS400ES حسب الـ spec:
```
HS200 tuning
    ↓
hs400_prepare_ddr()     ← تحضير الـ DDR
    ↓
hs400_complete()        ← تفعيل HS400
    ↓
hs400_enhanced_strobe() ← تفعيل enhanced strobe مع ios->enhanced_strobe = true
```

الـ driver كان بيـ call `hs400_enhanced_strobe()` قبل `hs400_complete()`، فالـ strobe timing كان بيتضبط على state مش مستقرة. النتيجة: في درجات حرارة تشغيل عالية (الـ TV box مغلق في حاوية) الـ timing drift بيتراكم لحد ما تبدأ أخطاء الـ CRC.

كمان `err_stats` في `mmc_host` كانت بتكشف المشكلة:
```c
u32 err_stats[MMC_ERR_MAX];
/* المهم هنا: */
// MMC_ERR_DAT_CRC    = index 3
// MMC_ERR_TUNING     = index 6
```

```bash
# قبل الإصلاح
$ cat /sys/kernel/debug/mmc0/err_stats
DAT_CRC: 1847
TUNING:  23
```

#### الحل
إصلاح ترتيب الـ callbacks في الـ driver:

```c
/* الترتيب الصح في مكان init الـ HS400ES */
static int imx8_mmc_init_hs400es(struct mmc_host *host, struct mmc_card *card)
{
    int err;

    /* 1. prepare DDR mode */
    err = host->ops->hs400_prepare_ddr(host);
    if (err)
        return err;

    /* 2. complete HS400 selection first */
    host->ops->hs400_complete(host);

    /* 3. then enable enhanced strobe */
    host->ios.enhanced_strobe = true;
    host->ops->hs400_enhanced_strobe(host, &host->ios);

    return 0;
}
```

كمان لازم تفعيل الـ re-tuning الدوري:
```c
/* في driver probe */
host->caps |= MMC_CAP2_HS400_ES;

/* الـ retune_period بالثواني — كل 5 دقايق */
host->retune_period = 300;
```

#### الدرس المستفاد
الـ `enhanced_strobe` flag في `mmc_ios` والـ `hs400_enhanced_strobe` callback لازم يتنفذوا **بعد** اكتمال الـ HS400 state machine. أي اختصار في الترتيب ممكن يدي نتائج صح في التجارب الابتدائية بس يفشل تحت ضغط حراري. الـ `err_stats[MMC_ERR_DAT_CRC]` هي أول مؤشر للمشكلة.

---

### السيناريو 4: SDIO Wi-Fi على AM62x — interrupt storms بتـ freeze الـ system

#### العنوان
**Automotive ECU على AM62x بيواجه kernel freeze بسبب SDIO IRQ loop**

#### السياق
وحدة ADAS مبنية على **TI AM62x**، بتستخدم Wi-Fi module متصل بـ SDIO (الـ chip هو TI WL1837). الـ module بيشتغل تمام بس تحت load عالي من الـ Wi-Fi traffic الـ system بتـ freeze تماماً ولازم reset يدوي.

#### المشكلة
```
INFO: rcu_sched detected stall on CPU 0!
mmc1: sdio_irq_thread stuck in tight loop
```

الـ SDIO interrupt thread بيدور في loop مستمرة.

#### التحليل
في `mmc_host`:
```c
unsigned int        sdio_irqs;           /* عدد الـ IRQs المسجّلة */
struct task_struct  *sdio_irq_thread;    /* الـ thread المسؤول */
struct work_struct  sdio_irq_work;       /* للـ NOTHREAD mode */
bool                sdio_irq_pending;    /* علامة pending IRQ */
atomic_t            sdio_irq_thread_abort; /* لإيقاف الـ thread */
```

الـ function الأساسية:
```c
static inline void mmc_signal_sdio_irq(struct mmc_host *host)
{
    host->ops->enable_sdio_irq(host, 0); /* disable IRQ أثناء المعالجة */
    host->sdio_irq_pending = true;
    if (host->sdio_irq_thread)
        wake_up_process(host->sdio_irq_thread);
}
```

المشكلة: الـ AM62x SDHCI driver كان `enable_sdio_irq()` مش بيـ mask الـ interrupt في الـ hardware صح. بيضبط bit في register بس من غير memory barrier، فالـ hardware بيـ fire الـ interrupt تاني قبل ما الـ thread يقرأ وينظف الـ status register في الـ Wi-Fi chip.

كمان الـ `MMC_CAP2_SDIO_IRQ_NOTHREAD` ما كانش متفعّل بينما الـ Wi-Fi driver كان بيعتمد على الـ `ack_sdio_irq` callback:
```c
/* في mmc_host_ops */
void (*enable_sdio_irq)(struct mmc_host *host, int enable);
/* Mandatory callback when using MMC_CAP2_SDIO_IRQ_NOTHREAD */
void (*ack_sdio_irq)(struct mmc_host *host);
```

#### الحل
```c
/* في الـ AM62x SDHCI driver */
static void am62x_sdhci_enable_sdio_irq(struct mmc_host *host, int enable)
{
    struct am62x_host *priv = mmc_priv(host);
    u32 reg;

    reg = readl(priv->base + SDHCI_INT_ENABLE);
    if (enable)
        reg |= SDHCI_INT_CARD_INT;
    else
        reg &= ~SDHCI_INT_CARD_INT;

    writel(reg, priv->base + SDHCI_INT_ENABLE);
    /* memory barrier ضروري قبل ما تكمّل */
    wmb();
}

static void am62x_sdhci_ack_sdio_irq(struct mmc_host *host)
{
    /* re-enable بعد ما الـ handler خلّص */
    am62x_sdhci_enable_sdio_irq(host, 1);
}
```

```c
/* في probe */
host->caps  |= MMC_CAP_SDIO_IRQ;
host->caps2 |= MMC_CAP2_SDIO_IRQ_NOTHREAD; /* استخدم الـ threaded mode الصح */
```

**التحقق:**
```bash
$ cat /proc/interrupts | grep mmc
# لو IRQ count بيزيد بشكل طبيعي مع الـ traffic = تمام
# لو بيزيد بملايين في ثانية = interrupt storm

# check sdio_irqs count
$ cat /sys/kernel/debug/mmc1/ios
```

#### الدرس المستفاد
الـ `enable_sdio_irq` callback لازم يضمن hardware-level masking مع proper memory barrier. استخدام `MMC_CAP2_SDIO_IRQ_NOTHREAD` بيتطلب تنفيذ `ack_sdio_irq` — من غيره الـ IRQ re-enable بيحصل في وقت غلط ويسبب storm. الـ `sdio_irq_thread` في `mmc_host` بيـ represent single-threaded processing لكل الـ SDIO functions.

---

### السيناريو 5: eMMC على Allwinner H616 — فشل bring-up بسبب power sequence خاطئة

#### العنوان
**Android TV Box رخيص على H616 مش بيلاقي الـ eMMC ومش بيبوت**

#### السياق
بورد TV box رخيصة مبنية على **Allwinner H616**، eMMC 16GB مدمجة. الـ bootloader (U-Boot) بيشتغل تمام لكن الـ Linux kernel مش بيشوف الـ eMMC خالص:
```
mmc0: no card present
mmc0: problem reading SD Status register
```

#### المشكلة
الـ eMMC موجودة فعلياً ومش removable، بس الـ kernel مش بيشوفها.

#### التحليل
في `mmc_host`:
```c
/* power-related في mmc_ios */
unsigned char power_mode; /* MMC_POWER_OFF / MMC_POWER_UP / MMC_POWER_ON */
unsigned int  power_delay_ms; /* waiting for stable power */

/* supply regulators */
struct mmc_supply supply;
/* اللي فيها */
struct regulator *vmmc;  /* Card power supply — الـ power الأساسي */
struct regulator *vqmmc; /* Optional Vccq supply */
```

الـ `mmc_pwrseq`:
```c
struct mmc_pwrseq *pwrseq; /* في struct mmc_host */
```

المشكلة كانت مركّبة:

**أولاً:** الـ DTS مش عنده `non-removable` property:
```c
/* في mmc_host */
#define MMC_CAP_NONREMOVABLE (1 << 8) /* Nonremovable e.g. eMMC */
```

بدون `non-removable`، الـ `mmc_card_is_removable()` بترجع true، والـ core بيحاول يعمل card detect polling عبر GPIO مش موصول.

**ثانياً:** الـ `rescan_disable` اتضبط من غير قصد:
```c
int rescan_disable; /* disable card detection */
```

كان في كود legacy في الـ driver بيضبط `rescan_disable = 1` في حالة معينة من الـ power state.

**ثالثاً:** الـ `power_mode` في `set_ios` ما اتنفّذش صح:
```c
/* في set_ios الخاص بالـ H616 driver */
static void h616_mmc_set_ios(struct mmc_host *host, struct mmc_ios *ios)
{
    switch (ios->power_mode) {
    case MMC_POWER_UP:
        /* BUG: power_delay_ms متجاهل */
        regulator_enable(host->supply.vmmc);
        break;
    case MMC_POWER_ON:
        /* بيبدأ الـ clock قبل ما الـ voltage تستقر */
        h616_set_clock(host, ios->clock);
        break;
    }
}
```

الـ `power_delay_ms` في `mmc_ios` موجود عشان الـ driver ينتظر استقرار الـ power قبل ما يبدأ الـ clock. تجاهله بيخلي CMD0 يتبعت وفولتية غير مستقرة فالـ eMMC مش بتستجيب.

#### الحل
```dts
/* DTS صح للـ H616 eMMC */
&mmc2 {
    pinctrl-names = "default";
    pinctrl-0 = <&mmc2_pins>;
    vmmc-supply  = <&reg_vcc_3v3>;
    vqmmc-supply = <&reg_vcc_1v8>;
    bus-width = <8>;
    non-removable;          /* مهم جداً */
    cap-mmc-highspeed;
    mmc-hs200-1_8v;
    mmc-hs400-1_8v;
    no-sdio;                /* MMC_CAP2_NO_SDIO */
    no-sd;                  /* MMC_CAP2_NO_SD */
    status = "okay";
};
```

```c
/* إصلاح set_ios */
static void h616_mmc_set_ios(struct mmc_host *host, struct mmc_ios *ios)
{
    switch (ios->power_mode) {
    case MMC_POWER_UP:
        regulator_enable(host->supply.vmmc);
        /* احترم power_delay_ms */
        if (ios->power_delay_ms)
            msleep(ios->power_delay_ms);
        break;
    case MMC_POWER_ON:
        /* شغّل الـ clock بعد ما الـ power استقر */
        h616_set_clock(host, ios->clock);
        break;
    case MMC_POWER_OFF:
        h616_set_clock(host, 0);
        regulator_disable(host->supply.vmmc);
        break;
    }
}
```

**Debug steps:**
```bash
# شوف الـ rescan state
$ cat /sys/bus/mmc/devices/mmc0\:0001/state

# force rescan
$ echo 1 > /sys/class/mmc_host/mmc0/rescan

# شوف الـ power mode
$ cat /sys/kernel/debug/mmc0/ios | grep power
power mode:     2 (on)  # 2 = MMC_POWER_ON

# لو مش موجود
$ dmesg | grep "mmc0:" | tail -20
```

#### الدرس المستفاد
الـ `power_mode` و `power_delay_ms` في `mmc_ios` مش decorative — الـ `set_ios` callback لازم ينفّذهم بدقة. الـ `MMC_CAP_NONREMOVABLE` لازم تتضبط لكل eMMC مدمجة (عبر `non-removable` في DTS أو hard-code في driver). نسيان أي خطوة من الـ power sequence ممكن يخلي الـ eMMC تبان invisible تماماً حتى لو موصولة فيزيائياً صح.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي بتغطي تطور الـ MMC subsystem في الـ Linux kernel:

| المقال | الرابط | الأهمية |
|--------|--------|---------|
| MMC layer — الـ patch الأصلي للـ framework | [lwn.net/Articles/82765](https://lwn.net/Articles/82765/) | أول implementation للـ MMC layer في الـ kernel |
| new driver: multimedia card (mmc) framework | [lwn.net/Articles/7176](https://lwn.net/Articles/7176/) | الـ patch الأصلي ضد kernel 2.4.19 |
| MMC updates — SDIO + SPI support | [lwn.net/Articles/253888](https://lwn.net/Articles/253888/) | إضافة SDIO و SPI للـ MMC layer |
| pwrseq: Add subsystem for power sequences | [lwn.net/Articles/602855](https://lwn.net/Articles/602855/) | الـ `mmc_pwrseq` اللي موجودة في `struct mmc_host` |
| mmc: Add Command Queue support | [lwn.net/Articles/738160](https://lwn.net/Articles/738160/) | الـ `mmc_cqe_ops` و CQE engine في الـ host |
| Add MMC software queue support | [lwn.net/Articles/803435](https://lwn.net/Articles/803435/) | الـ `hsq_enabled` و `hsq_depth` في `struct mmc_host` |
| Add support UHS-II for GL9755 | [lwn.net/Articles/944645](https://lwn.net/Articles/944645/) | الـ `sd_uhs2_caps` و `uhs2_control` callback |
| Add support UHS-II for GL9755 and GL9767 | [lwn.net/Articles/987626](https://lwn.net/Articles/987626/) | تطوير دعم UHS-II في الـ kernel 6.9 |
| mmc: sdhci-of-dwcmshc: Add CQE support | [lwn.net/Articles/965527](https://lwn.net/Articles/965527/) | مثال على تطبيق `mmc_cqe_ops` في controller حقيقي |
| mmc: Add host driver for OCTEON MMC | [lwn.net/Articles/496911](https://lwn.net/Articles/496911/) | مثال على كتابة host driver من الصفر |

---

### توثيق الـ Kernel الرسمي

الـ documentation الرسمي في `Documentation/` هو المرجع الأساسي لكتابة أي host driver:

#### ملفات التوثيق

```
Documentation/driver-api/mmc/index.rst          ← entry point للـ MMC subsystem docs
Documentation/driver-api/mmc/mmc-dev-attrs.rst  ← attributes الـ sysfs للـ mmc devices
Documentation/driver-api/mmc/mmc-dev-parts.rst  ← partitions في eMMC
Documentation/driver-api/mmc/mmc-tools.rst      ← mmc-utils و mmc_test
Documentation/driver-api/mmc/card-quirks.rst    ← quirks للـ cards المعروفة بمشاكل
```

الروابط على الإنترنت:
- [MMC/SD/SDIO card support — kernel docs](https://static.lwn.net/kerneldoc/driver-api/mmc/index.html)
- [MMC tools introduction](https://static.lwn.net/kerneldoc/driver-api/mmc/mmc-tools.html)
- [Device tree bindings: mmc-controller.yaml](https://mjmwired.net/kernel/Documentation/devicetree/bindings/mmc/mmc-controller.yaml)

#### ملفات الـ Source الأساسية

```
include/linux/mmc/host.h     ← struct mmc_host + mmc_host_ops + mmc_cqe_ops
include/linux/mmc/core.h     ← struct mmc_request + mmc_command + mmc_data
include/linux/mmc/card.h     ← struct mmc_card
include/linux/mmc/sd_uhs2.h  ← UHS-II specific definitions
drivers/mmc/core/            ← MMC core logic (bus, queue, block, sdio)
drivers/mmc/host/            ← host controller drivers (sdhci, dw_mmc, ...)
```

---

### Commits مهمة في تاريخ الـ MMC Subsystem

| الـ Feature | الـ Commit / السياق |
|------------|---------------------|
| إضافة `mmc_cqe_ops` للـ CQE | kernel 4.20 — Adrian Hunter (Intel) |
| إضافة `MMC_CAP2_CQE` و CQHCI driver | kernel 4.20 |
| إضافة `MMC_CAP2_HS400_ES` — enhanced strobe | kernel 4.7 |
| إضافة software queue (`hsq`) | kernel 5.7 |
| إضافة `MMC_CAP2_SD_UHS2` و UHS-II framework | kernel 6.8 – 6.9 |
| إضافة `MMC_CAP2_CRYPTO` / inline encryption | kernel 5.9 |
| إضافة `mmc_pwrseq` subsystem | kernel 4.0 |

للبحث عن commits محددة:
```bash
git log --oneline drivers/mmc/ -- include/linux/mmc/host.h
git log --oneline --grep="CQE\|cqe" drivers/mmc/
git log --oneline --grep="UHS2\|uhs2" drivers/mmc/
```

---

### نقاشات Mailing List

الـ mailing list الرئيسية للـ MMC subsystem هي `linux-mmc@vger.kernel.org`:

- **أرشيف Spinics**: [spinics.net/lists/linux-mmc](https://www.spinics.net/lists/linux-mmc/)
- **أرشيف mail-archive**: [mail-archive.com — linux-mmc@vger.kernel.org](https://www.mail-archive.com/linux-mmc@vger.kernel.org/)
- **نقاش مهم عن eMMC tuning failure**: [mail-archive.com/msg33823](https://www.mail-archive.com/linux-mmc@vger.kernel.org/msg33823.html)
- **نقاش CQE patches V9**: [linux.kernel.narkive.com](https://linux.kernel.narkive.com/VrkQk3df/patch-v9-05-15-mmc-mmc-enable-cqe-s)
- **نقاش HS400 tuning و presets**: [spinics.net/lists/linux-mmc/msg60173](https://www.spinics.net/lists/linux-mmc/msg60173.html)

للاشتراك أو البحث في الـ mailing list:
```
To: linux-mmc@vger.kernel.org
Subject: [PATCH] mmc: ...
```

---

### Kernelnewbies.org — تغييرات الـ MMC per kernel version

الصفحات دي بتوثق التغييرات في كل إصدار kernel وبيكون فيها section عن MMC:

| إصدار الـ Kernel | الرابط | أهم تغيير في MMC |
|-----------------|--------|-----------------|
| Linux 2.6.24 | [kernelnewbies.org/Linux_2_6_24](https://kernelnewbies.org/Linux_2_6_24) | SDIO + SPI support |
| Linux 3.9 | [kernelnewbies.org/Linux_3.9](https://kernelnewbies.org/Linux_3.9) | تحسينات الـ MMC core |
| Linux 4.0 | [kernelnewbies.org/Linux_4.0-DriversArch](https://kernelnewbies.org/Linux_4.0-DriversArch) | `mmc_pwrseq` subsystem |
| Linux 5.7 | [kernelnewbies.org/Linux_5.7](https://kernelnewbies.org/Linux_5.7) | MMC software queue (HSQ) |
| Linux 6.7 | [kernelnewbies.org/Linux_6.7](https://kernelnewbies.org/Linux_6.7) | تحسينات UHS-II |
| Linux 6.8 | [kernelnewbies.org/Linux_6.8](https://kernelnewbies.org/Linux_6.8) | UHS-II framework |

---

### eLinux.org — Embedded Linux و MMC

- **eMMC 8-bit width tests**: [elinux.org/Tests:eMMC-8bit-width](https://elinux.org/Tests:eMMC-8bit-width) — اختبار `MMC_BUS_WIDTH_8` و `MMC_CAP_8_BIT_DATA`
- **SDIO driver tests**: [elinux.org/Tests:SDIO-KS7010](https://elinux.org/Tests:SDIO-KS7010) — اختبار SDIO interrupts و `enable_sdio_irq`
- **MMC init error troubleshooting**: [elinux.org — mmc0: error -110](https://elinux.org/Thread:Talk:R-Car/Boards/M3SK/Booting_of_kernel_stops_at_23.047503_mmc0:_error_-110_whilst_initialising_MMC_card_(5)) — debug مشاكل الـ host driver

---

### كتب مرجعية

#### Linux Device Drivers, 3rd Edition (LDD3)
- **المؤلفون**: Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الرابط المجاني**: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)
- **الفصول ذات الصلة**:
  - Chapter 15: Memory Mapping and DMA — فهم `DMA_TO_DEVICE` / `DMA_FROM_DEVICE` المستخدمة في `mmc_get_dma_dir()`
  - Chapter 14: The Linux Device Model — فهم `struct device` و `class_dev` في `struct mmc_host`
  - Chapter 10: Interrupt Handling — فهم `enable_sdio_irq` و SDIO IRQ thread

#### Linux Kernel Development, 3rd Edition — Robert Love
- **الفصول ذات الصلة**:
  - Chapter 7: Interrupts and Interrupt Handlers — الـ `sdio_irq_thread` و `sdio_irq_work`
  - Chapter 11: Timers and Time Management — الـ `retune_timer` في `struct mmc_host`
  - Chapter 13: The Virtual Filesystem — فهم block layer اللي بيتعامل معاها الـ MMC
  - Chapter 16: The Page Cache and Page Writeback — performance context للـ eMMC CQE

#### Embedded Linux Primer, 2nd Edition — Christopher Hallinan
- **الفصول ذات الصلة**:
  - Chapter 8: Device Drivers — كتابة host driver باستخدام `mmc_alloc_host()` و `mmc_add_host()`
  - Chapter 9: File Systems — استخدام MMC/SD كـ storage في embedded systems

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- Chapter 6: Device Drivers — block device layer و علاقتها بالـ MMC subsystem

---

### مصادر تقنية إضافية

#### JEDEC Standards
- **JESD84-B51** (eMMC 5.1): يعرف الـ Command Queue, HS400, HS200
  - المرجع للـ `MMC_CAP2_CQE`, `MMC_TIMING_MMC_HS400`, `mmc_cqe_ops`
- **JESD84-B50** (eMMC 5.0): يعرف HS200 و Enhanced Strobe
  - المرجع للـ `MMC_CAP2_HS200`, `MMC_CAP2_HS400_ES`, `enhanced_strobe`

#### SD Association Specifications
- **Physical Layer Simplified Spec v9.00**: يعرف UHS-I (SDR12/25/50/104, DDR50)
  - المرجع للـ `MMC_CAP_UHS_*` flags
- **SD UHS-II Simplified Addendum**: يعرف UHS-II Speed A/B
  - المرجع للـ `sd_uhs2_caps`, `MMC_TIMING_UHS2_SPEED_*`

#### Micron White Paper
- [Linux Storage System Analysis for eMMC With Command Queuing](https://assets.micron.com/adobe/assets/urn:aaid:aem:c03e48e7-8379-4557-aee9-7e248468027f/renditions/original/as/linux-storage-system-analysis-emmc-command-queuing.pdf) — تحليل أداء CQE على Linux

---

### Search Terms للبحث عن مزيد من المعلومات

للبحث في الـ mailing list و الـ kernel history:

```
# في LWN.net
site:lwn.net "mmc_host_ops"
site:lwn.net "mmc_cqe_ops"
site:lwn.net "UHS-II" linux kernel

# في Patchwork
site:patchwork.kernel.org mmc host controller

# في LKML
site:lkml.iu.edu "struct mmc_host"
site:lkml.kernel.org mmc "mmc_host_ops"

# في Bootlin Elixir (cross-reference)
https://elixir.bootlin.com/linux/latest/source/include/linux/mmc/host.h
https://elixir.bootlin.com/linux/latest/source/drivers/mmc/core/

# Git log مباشرة
git log --follow -p include/linux/mmc/host.h
git log --oneline --grep="mmc_host_ops\|mmc_cqe_ops\|UHS2"
```

#### Bootlin Elixir Cross-Reference — أهم أداة للمطور
**الـ** `struct mmc_host` ومن يستخدمها:
```
https://elixir.bootlin.com/linux/latest/ident/mmc_host
https://elixir.bootlin.com/linux/latest/ident/mmc_host_ops
https://elixir.bootlin.com/linux/latest/ident/mmc_cqe_ops
```

#### TI Processor SDK Documentation
- [MMC/SD — TI Processor SDK Linux](https://software-dl.ti.com/processor-sdk-linux/esd/docs/06_02_00_81/linux/Foundational_Components/Kernel/Kernel_Drivers/Storage/MMC-SD.html) — مثال عملي لاستخدام الـ MMC host driver على embedded SoC
## Phase 8: Writing simple module

### الفكرة

هنعمل module بيعمل **kprobe** على الدالة `mmc_detect_change()` اللي بتتقال من host driver لما يحصل تغيير في وجود الـ card (إدخال أو إخراج). ده نقطة مثيرة جداً لأنها entry point لكل event insertion/removal في الـ MMC stack.

الدالة معرفة في `host.h`:
```c
void mmc_detect_change(struct mmc_host *, unsigned long delay);
```

بتاخد الـ `mmc_host` و الـ `delay` بالـ jiffies. هنطبع اسم الـ host و قيمة الـ delay لما تتنادى.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * mmc_detect_kprobe.c
 *
 * Hooks mmc_detect_change() via kprobe and logs the host name + delay.
 * Useful for tracing SD/eMMC card insertion and removal events.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>       /* pr_info */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe */
#include <linux/mmc/host.h>     /* struct mmc_host, mmc_hostname() */

/* kprobe target: mmc_detect_change(struct mmc_host *host, unsigned long delay) */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64:
     *   rdi = first arg  => struct mmc_host *host
     *   rsi = second arg => unsigned long delay (jiffies)
     *
     * On ARM64:
     *   x0 = host, x1 = delay
     *
     * kernel_pt_regs helpers abstract this per-arch.
     */
#ifdef CONFIG_X86_64
    struct mmc_host *host = (struct mmc_host *)regs->di;
    unsigned long    delay = (unsigned long)regs->si;
#elif defined(CONFIG_ARM64)
    struct mmc_host *host = (struct mmc_host *)regs->regs[0];
    unsigned long    delay = (unsigned long)regs->regs[1];
#else
    /* fallback: skip printing, still counts the hit */
    pr_info("mmc_detect_kprobe: mmc_detect_change() called (arch unsupported for arg decode)\n");
    return 0;
#endif

    /* mmc_hostname() returns dev_name(&host->class_dev) — safe to call here */
    pr_info("mmc_detect_kprobe: mmc_detect_change() on host=%s delay=%lu jiffies\n",
            mmc_hostname(host), delay);

    return 0; /* 0 = let the original function continue normally */
}

/* kprobe descriptor — only need .symbol_name and .pre_handler */
static struct kprobe kp = {
    .symbol_name = "mmc_detect_change",
    .pre_handler = handler_pre,
};

static int __init mmc_detect_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("mmc_detect_kprobe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("mmc_detect_kprobe: planted at %s (%px)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

static void __exit mmc_detect_probe_exit(void)
{
    /* Must unregister before module unloads to avoid executing freed code */
    unregister_kprobe(&kp);
    pr_info("mmc_detect_kprobe: removed from %s\n", kp.symbol_name);
}

module_init(mmc_detect_probe_init);
module_exit(mmc_detect_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Explorer");
MODULE_DESCRIPTION("kprobe on mmc_detect_change() to trace MMC card detect events");
```

---

### Makefile للـ out-of-tree build

```makefile
obj-m += mmc_detect_kprobe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ Includes

```c
#include <linux/kprobes.h>
#include <linux/mmc/host.h>
```

الـ `kprobes.h` هو اللي بيعرف `struct kprobe` و `register_kprobe()`. الـ `mmc/host.h` محتاجينه عشان نـ cast الـ argument لـ `struct mmc_host *` و نستخدم `mmc_hostname()`.

#### الـ `handler_pre`

```c
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
```

ده الـ callback اللي بيتنفذ **قبل** الـ instruction الأولى في `mmc_detect_change()`. الـ `pt_regs` بيحمل state الـ CPU registers وقت الـ intercept، ومنه بنجيب الـ arguments حسب الـ calling convention للـ architecture.

#### استخراج الـ Arguments

```c
struct mmc_host *host = (struct mmc_host *)regs->di;
unsigned long    delay = (unsigned long)regs->si;
```

على x86-64 الـ System V ABI بتحط أول argument في `rdi` وتاني في `rsi`. الـ `pt_regs` members `di` و `si` بيعكسوا ده مباشرة. على ARM64 بنستخدم `regs[0]` و `regs[1]` بدل كده.

#### الـ `mmc_hostname()` macro

```c
pr_info("... host=%s ...", mmc_hostname(host), delay);
```

الـ macro معرف في `host.h`:
```c
#define mmc_hostname(x) (dev_name(&(x)->class_dev))
```

بيرجع اسم الـ device زي `mmc0` أو `mmc1` وده آمن تماماً يتنادى في kprobe handler.

#### الـ `module_init` و `module_exit`

```c
ret = register_kprobe(&kp);
```

الـ init بيزرع الـ probe في الـ kernel بعد ما يحل الـ symbol. لو الـ symbol مش موجود أو الـ kernel مش مـ compile بـ `CONFIG_KPROBES=y` هيرجع error.

```c
unregister_kprobe(&kp);
```

الـ exit لازم يشيل الـ probe **قبل** ما الـ module يتـ unload. لو متشلش، أي call لـ `mmc_detect_change()` بعد كده هيـ execute كود في memory اتحررت وهيحصل kernel panic.

---

### تجربة الـ Module

```bash
# بناء الـ module
make

# تحميله
sudo insmod mmc_detect_kprobe.ko

# إدخال أو إخراج SD card أو تشغيل:
echo 1 > /sys/bus/mmc/devices/mmc0:0001/remove   # أو أي trigger

# مشاهدة الـ output
sudo dmesg | grep mmc_detect_kprobe

# تفريغه
sudo rmmod mmc_detect_kprobe
```

**مثال output متوقع:**

```
[  123.456789] mmc_detect_kprobe: planted at mmc_detect_change (ffffffffc0ab1234)
[  145.001234] mmc_detect_kprobe: mmc_detect_change() on host=mmc0 delay=0 jiffies
[  201.774412] mmc_detect_kprobe: mmc_detect_change() on host=mmc0 delay=20 jiffies
[  201.891234] mmc_detect_kprobe: removed from mmc_detect_change
```

الـ `delay=0` معناه instant detection (زي IRQ-based)، و `delay=20` معناه الـ driver بيستنى 20 jiffies (حوالي 200ms على HZ=100) قبل ما يبدأ الـ rescan عشان يتجنب bounce في الـ card detect line.
