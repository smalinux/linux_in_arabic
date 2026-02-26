## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الـ file ده جزء من **MMC/SD/SDIO Subsystem** في Linux kernel — المسؤول عن كل بطاقات الذاكرة والـ eMMC chips اللي جوه الموبايلات واللاب توب والـ embedded systems.

الـ maintainer: Ulf Hansson — الـ mailing list: `linux-mmc@vger.kernel.org`

---

### الصورة الكبيرة — ليه الملف ده موجود؟

تخيل إنك بتكلم حد بلغة مش عارفها — محتاج قاموس. الـ `mmc.h` ده هو **القاموس الرسمي** للغة الـ MMC protocol. كل أمر بتبعته للكارد، وكل رد بتستقبله، وكل register في الكارت — كلها معرّفة هنا كـ macros بأرقامها الرسمية من الـ JEDEC specification.

---

### المشكلة اللي بتحلها

الـ MMC card (أو eMMC اللي جوه تليفونك) هي في الأساس **جهاز مستقل** بيفهم بروتوكول خاص بيه. الـ CPU بيتكلم معاه عن طريق أوامر رقمية — زي مثلاً: "ابعت لي الـ CID"، "إقرأ block رقم 1000"، "غير الـ bus width لـ 8-bit".

المشكلة: مين اللي بيحفظ إن الأمر رقم `17` معناه "اقرأ block واحد"؟ ومين اللي بيعرف إن الـ bit رقم `31` في الـ R1 response معناه "out of range error"؟

**الإجابة: الملف ده.**

بدله كل developer كان هيحفظ أرقام غريبة أو يكتب comments مش واضحة. الملف ده بيجمع كل الـ constants دي في مكان واحد.

---

### القصة الكاملة — رحلة أمر Read بسيط

```
تطبيق المستخدم
     ↓
block layer (kernel)
     ↓
drivers/mmc/core/block.c  ← بيحول request لـ MMC command
     ↓
drivers/mmc/core/mmc_ops.c ← بيبني الأمر باستخدام macros من mmc.h
     ↓
drivers/mmc/host/sdhci.c   ← بيبعت الأمر على الـ bus الفعلي
     ↓
MMC Card Hardware
     ↓
R1 Response  ← الـ core بيفسره بالـ R1_* macros من mmc.h
```

في كل خطوة في الرحلة دي، الكود بيرجع لـ `mmc.h` عشان يعرف الأرقام الصح.

---

### محتوى الملف — إيه اللي جوّاه فعلاً؟

#### 1. أوامر الـ MMC (Command Set)

الـ MMC standard بيقسم الأوامر لـ **Classes** — كل class بيغطي وظيفة:

| Class | الوظيفة | أمثلة |
|-------|---------|-------|
| Class 0 | Basic | `CMD0` reset, `CMD1` init, `CMD7` select |
| Class 2 | Block Read | `CMD17` read single, `CMD18` read multi |
| Class 4 | Block Write | `CMD24` write single, `CMD25` write multi |
| Class 5 | Erase | `CMD35/36/38` erase groups |
| Class 6 | Write Protect | `CMD28/29/30` |
| Class 7 | Lock | `CMD42` lock/unlock |
| Class 11 | Command Queue | `CMD44/45/46/47/48` CMDQ |

```c
#define MMC_GO_IDLE_STATE        0   /* bc  — broadcast, no response */
#define MMC_SEND_OP_COND         1   /* bcr — broadcast, R3 response */
#define MMC_READ_SINGLE_BLOCK   17   /* adtc — addressed data transfer */
#define MMC_WRITE_BLOCK         24   /* adtc — write one block */
```

#### 2. الـ R1 Response Bits — حالة الكارد

لما بتبعت أمر، الكارد بيرد بـ **R1 status register** — 32 bit كل bit بيقول حاجة:

```c
#define R1_OUT_OF_RANGE    (1 << 31)  /* طلبت address برا نطاق الكارد */
#define R1_CARD_IS_LOCKED  (1 << 25)  /* الكارد محمي بـ password */
#define R1_COM_CRC_ERROR   (1 << 23)  /* الأمر وصل فيه تلف */
#define R1_ILLEGAL_COMMAND (1 << 22)  /* أمر مش موجود أو مش في الوقت الصح */
#define R1_READY_FOR_DATA  (1 <<  8)  /* الكارد جاهز يستقبل/يبعت بيانات */
```

وبيحدد **states** الكارد — الكارد مش بيقبل أوامر معينة غير في state معين:

```c
#define R1_STATE_IDLE  0  /* بعد الـ reset */
#define R1_STATE_TRAN  4  /* Transfer state — الحالة الطبيعية للـ r/w */
#define R1_STATE_PRG   7  /* Programming — الكارد بيكتب جوّاه */
```

#### 3. الـ EXT_CSD — "قاموس" الكارد

الـ **Extended CSD** هو register بحجم 512 byte — فيه كل إعدادات الكارد وقدراته وحالته. كل byte ليه رقم (offset) وليه صلاحيات (RO/RW/W).

```c
#define EXT_CSD_HS_TIMING      185  /* R/W — اختار الـ timing mode */
#define EXT_CSD_BUS_WIDTH      183  /* R/W — اختار عدد الـ data lines */
#define EXT_CSD_CARD_TYPE      196  /* RO  — الكارد يدعم أنهي سرعات */
#define EXT_CSD_REV            192  /* RO  — إصدار الـ EXT_CSD نفسه */
#define EXT_CSD_SEC_CNT        212  /* RO  — إجمالي عدد الـ sectors */
```

**مثال حقيقي:** لما الـ kernel بيحاول يشغل الكارد بـ HS400 (أسرع mode)، بيقرأ `EXT_CSD_CARD_TYPE` (offset 196) ويشوف لو الـ `EXT_CSD_CARD_TYPE_HS400` bit متحطة — لو آه، يكمل. لو لأ، يرجع لـ HS200 أو أبطأ.

```c
#define EXT_CSD_CARD_TYPE_HS400_1_8V (1<<6)  /* 200MHz DDR at 1.8V */
#define EXT_CSD_CARD_TYPE_HS400_1_2V (1<<7)  /* 200MHz DDR at 1.2V */
#define EXT_CSD_CARD_TYPE_HS400      (EXT_CSD_CARD_TYPE_HS400_1_8V | \
                                      EXT_CSD_CARD_TYPE_HS400_1_2V)
```

#### 4. الـ CCC — Card Command Classes

الـ **CSD register** بيحتوي على **CCC field** — 12-bit بكل bit بيقول "الكارد ده بيدعم الـ class دي":

```c
#define CCC_BLOCK_READ   (1<<2)  /* الكارد يعرف يقرأ blocks */
#define CCC_BLOCK_WRITE  (1<<4)  /* الكارد يعرف يكتب blocks */
#define CCC_ERASE        (1<<5)  /* الكارد يعرف يمسح */
#define CCC_LOCK_CARD    (1<<7)  /* الكارد يعرف يتقفل بـ password */
```

#### 5. الـ Erase/Trim/Discard Arguments

الـ MMC بيدعم مستويات مختلفة من "المسح":

```c
#define MMC_ERASE_ARG          0x00000000  /* مسح عادي */
#define MMC_TRIM_ARG           0x00000001  /* trim — زي TRIM في SSD */
#define MMC_DISCARD_ARG        0x00000003  /* discard — أسرع من trim */
#define MMC_SECURE_ERASE_ARG   0x80000000  /* مسح آمن — للبيانات الحساسة */
```

#### 6. الـ BKOPS — Background Operations

الـ eMMC لما بيكون عنده "تنظيف داخلي" يعمله (زي الـ garbage collection في الـ SSD):

```c
#define EXT_CSD_BKOPS_SUPPORT  502  /* هل الكارد بيدعم BKOPS */
#define EXT_CSD_BKOPS_EN       163  /* تفعيل/تعطيل الـ BKOPS */
#define EXT_CSD_BKOPS_START    164  /* ابدأ BKOPS دلوقتي */
#define EXT_CSD_URGENT_BKOPS   BIT(0) /* الكارد محتاج BKOPS فوري */
```

#### 7. الـ RPMB — Replay Protected Memory Block

جزء من الكارد مخصص للتخزين الآمن (بيستخدمه مثلاً TEE/TrustZone):

```c
#define EXT_CSD_RPMB_MULT  168  /* حجم الـ RPMB partition بالـ 128KB */
```

#### 8. الـ CMDQ — Command Queuing

زي الـ NCQ في الـ SATA أو الـ NVMe queuing — تقدر تبعت 32 أمر في نفس الوقت:

```c
#define EXT_CSD_CMDQ_SUPPORT  308  /* هل الكارد بيدعم CMDQ */
#define EXT_CSD_CMDQ_DEPTH    307  /* أقصى عدد أوامر متزامنة */
#define MMC_QUE_TASK_PARAMS   44   /* CMD44 — حدد parameters الأمر */
#define MMC_QUE_TASK_ADDR     45   /* CMD45 — حدد address البيانات */
#define MMC_EXECUTE_READ_TASK 46   /* CMD46 — نفّذ قراءة */
#define MMC_EXECUTE_WRITE_TASK 47  /* CMD47 — نفّذ كتابة */
```

---

### Inline Helper Functions

الملف فيه 3 functions بسيطة:

```c
/* هل الأمر ده multi-block transfer؟ */
static inline bool mmc_op_multi(u32 opcode) { ... }

/* هل الأمر ده tuning sequence؟ */
static inline bool mmc_op_tuning(u32 opcode) { ... }

/* هل الكارد جاهز فعلاً للبيانات؟ (بيتحقق من حالتين مش حالة واحدة) */
static inline bool mmc_ready_for_data(u32 status) { ... }
```

---

### الـ SPI Mode

الـ MMC ممكن يتشغل فوق **SPI bus** بدل الـ native MMC interface — في الحالة دي الـ responses بتختلف:

```c
#define R1_SPI_IDLE            (1 << 0)  /* الكارد في idle state */
#define R1_SPI_ILLEGAL_COMMAND (1 << 2)  /* أمر غير قانوني */
#define MMC_SPI_READ_OCR       58        /* قرأ الـ OCR register في SPI mode */
```

---

### الملفات ذات الصلة

| الملف | الدور |
|-------|-------|
| `include/linux/mmc/mmc.h` | **الملف ده** — MMC protocol constants |
| `include/linux/mmc/sd.h` | نفس الفكرة لبروتوكول الـ SD cards |
| `include/linux/mmc/sdio.h` | constants لبروتوكول الـ SDIO (WiFi/BT) |
| `include/linux/mmc/host.h` | تعريف struct الـ host controller |
| `include/linux/mmc/card.h` | تعريف struct الـ mmc_card |
| `include/linux/mmc/core.h` | الـ core API للتعامل مع الكروت |
| `drivers/mmc/core/mmc.c` | initialization وإدارة الـ MMC cards |
| `drivers/mmc/core/mmc_ops.c` | تنفيذ الأوامر المعرّفة هنا |
| `drivers/mmc/core/block.c` | ربط الـ MMC بالـ block device layer |
| `drivers/mmc/core/core.c` | القلب الرئيسي للـ subsystem |
| `drivers/mmc/host/sdhci.c` | أشهر host controller driver |

---

### ملخص — الملف ده بيعمل إيه بالضبط؟

الملف **لا بيحتوي على logic ولا بيفعل حاجة بنفسه.** ده ملف تعريفات بحت (constants + 3 inline helpers). قيمته في إنه:

1. **ترجمة** الـ JEDEC MMC specification لـ C constants يقدر الكود يستخدمها
2. **توحيد** اللغة بين كل مكونات الـ MMC subsystem
3. **مرجع** للـ developers لفهم معنى كل رقم في الـ protocol

بدونه، كل driver كان هيكتب الأرقام يدوياً — وبيؤدي لـ bugs وعدم توافق.
## Phase 2: شرح الـ MMC/SD Framework

### المشكلة اللي الـ Subsystem بيحلها

تخيل إنك بتكتب driver لـ eMMC chip على board معينة. من غير framework، هتضطر تعيد كتابة كل منطق:
- إزاي تعمل card initialization (CMD0 → CMD1 → CMD2 → CMD3 → CMD7)
- إزاي تفهم الـ CSD و CID registers
- إزاي تعمل block read/write
- إزاي تتعامل مع الـ error states
- إزاي تضيف support للـ filesystem فوق الكارت

وكمان لو عندك SD card driver تاني، هيعيد نفس الكلام من الأول. **الـ MMC/SD Framework** موجود عشان يمنع التكرار ده ويفصل بين:

1. **منطق البروتوكول** — أوامر الكارت، state machine، initialization sequences (هذا مشترك بين كل hardware)
2. **منطق الـ Hardware** — إزاي الـ controller بيبعت الـ clock وبيستقبل الـ data (ده بيختلف من chip لـ chip)

---

### الحل — نهج الـ Kernel

الـ kernel قسّم المشكلة لـ **3 طبقات واضحة**:

```
┌─────────────────────────────────────────┐
│         Block Device Layer              │  ← الـ filesystem بيشوف هنا
│         (mmc_blk driver)                │
├─────────────────────────────────────────┤
│         MMC Core Layer                  │  ← المنطق المشترك كله هنا
│  (card init, protocol, state machine)   │
├──────────────┬──────────────────────────┤
│  Card Ops    │    Host Ops              │
│  (mmc/sd/    │  (set_ios, request,      │
│   sdio)      │   get_cd, tuning...)     │
├──────────────┴──────────────────────────┤
│         Host Controller Driver          │  ← الـ hardware-specific هنا
│  (sdhci, dw_mmc, omap_hsmmc, ...)       │
├─────────────────────────────────────────┤
│         Actual Hardware (eMMC/SD/SDIO)  │
└─────────────────────────────────────────┘
```

---

### الـ Big Picture Architecture

```
User Space
    │
    ▼
┌──────────────────────────────────────────────────────┐
│                  VFS / Filesystem                    │
│              (ext4, fat32, f2fs, ...)                │
└────────────────────────┬─────────────────────────────┘
                         │  block I/O requests
                         ▼
┌──────────────────────────────────────────────────────┐
│              Block Layer (bio / blk-mq)              │
│   الـ subsystem ده مش جزء من MMC لكن بيستخدمه       │
└────────────────────────┬─────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────┐
│            mmc_blk (Block Driver for MMC)            │
│  - يحول block requests لـ mmc_request structs        │
│  - يتعامل مع partitions (boot0, boot1, RPMB, GP)    │
└────────────────────────┬─────────────────────────────┘
                         │  struct mmc_request
                         ▼
┌──────────────────────────────────────────────────────┐
│                    MMC Core                          │
│                                                      │
│  ┌─────────────┐  ┌──────────┐  ┌────────────────┐  │
│  │  mmc_ops    │  │  sd_ops  │  │   sdio_ops     │  │
│  │ (eMMC init) │  │(SD init) │  │ (SDIO init)    │  │
│  └─────────────┘  └──────────┘  └────────────────┘  │
│                                                      │
│  - Card detection & initialization                   │
│  - Power management                                  │
│  - Tuning / re-tuning                                │
│  - Error recovery                                    │
│  - Command queuing (CQE)                             │
└────────────────────────┬─────────────────────────────┘
                         │  struct mmc_host_ops (callbacks)
                         ▼
┌──────────────────────────────────────────────────────┐
│              Host Controller Driver                  │
│                                                      │
│   sdhci.c          dw_mmc.c         omap_hsmmc.c    │
│   sdhci-pci.c      sdhci-esdhc.c    bcm2835_mmc.c   │
│                                                      │
│  - DMA setup                                         │
│  - Clock management                                  │
│  - Voltage switching (3.3V ↔ 1.8V)                  │
│  - Signal timing / tuning hardware                   │
└────────────────────────┬─────────────────────────────┘
                         │  physical signals
                         ▼
┌──────────────────────────────────────────────────────┐
│            Physical Card / Chip                      │
│         (eMMC, SD Card, SDIO WiFi chip, ...)         │
└──────────────────────────────────────────────────────┘
```

---

### التشبيه الواقعي — المطار والطيارة

تخيل **شركة طيران بتشتغل على مطارات مختلفة**:

| عنصر المطار | المقابل في الـ MMC Framework |
|---|---|
| **الطيار** | الـ `mmc_blk` driver — بيحدد الـ destination (read/write request) |
| **برج المراقبة (ATC)** | الـ MMC Core — بيدير كل الـ protocol، بيقرر إيمتى تقلع وإيمتى تنزل |
| **لغة الطيران الموحدة (ICAO)** | الـ MMC commands (CMD0, CMD1...) — نفس اللغة بغض النظر عن المطار |
| **المطار نفسه (البنية التحتية)** | الـ Host Controller — SDHCI, DW-MMC, HostController التاني |
| **الـ Runway + الأجهزة** | الـ struct `mmc_host_ops` — الـ callbacks اللي كل مطار بيعملها بطريقته |
| **الطيارة** | الـ `mmc_card` struct — بتحمل كل معلومات الكارت (CID, CSD, EXT_CSD) |
| **جواز السفر / Flight manifest** | الـ CID (Card IDentification) — مين الكارت ده؟ من مين صنعه؟ |
| **قوانين الجمارك لكل بلد** | الـ quirks — كل كارت ممكن يكون عنده سلوك غريب محتاج handling خاص |

**الفكرة الجوهرية**: برج المراقبة (Core) ما بيعرفش تفاصيل كل مطار، لكن بيعرف إزاي يتكلم مع الطيارة بلغة موحدة. وكل مطار (Host Controller) بيعمل نفس الـ interface بطريقته الخاصة.

---

### الـ Core Abstraction — الفكرة المحورية

الـ MMC Framework بيدور حول **3 abstractions رئيسية**:

#### 1. `struct mmc_host` — الـ Controller

```c
struct mmc_host {
    const struct mmc_host_ops *ops;  /* vtable للـ hardware callbacks */
    struct mmc_ios ios;              /* الـ bus state الحالي (clock, voltage, width) */
    struct mmc_card *card;           /* الكارت اللي على الـ bus دلوقتي */
    unsigned int caps;               /* capabilities: 4-bit, 8-bit, DDR, UHS... */
    unsigned int f_min, f_max;       /* نطاق الـ clock المسموح بيه */
    u32 ocr_avail;                   /* الـ voltages المدعومة */
    /* ... */
};
```

الـ `mmc_host` هو **الـ Software representation للـ SDMMC Controller**. كل controller على الـ SoC بيكون له instance منه. الـ host driver بيعمل `alloc` وبيملى الـ `ops`.

#### 2. `struct mmc_card` — الكارت

```c
struct mmc_card {
    struct mmc_host *host;     /* على أي controller موصول */
    unsigned int type;         /* MMC_TYPE_MMC / SD / SDIO */
    unsigned int rca;          /* Relative Card Address — زي عنوان الكارت */
    struct mmc_cid cid;        /* Card IDentification — مين صنعه وإيه الـ serial */
    struct mmc_csd csd;        /* Card Specific Data — السعة والـ timing */
    struct mmc_ext_csd ext_csd;/* eMMC v4+ — الـ extended registers (512 bytes) */
    struct mmc_part part[7];   /* physical partitions: boot0/1, RPMB, GP0-3 */
    /* ... */
};
```

الـ `mmc_card` بيتعمل بعد الـ initialization ويُمثّل الكارت اللي اتشاف على الـ bus.

#### 3. `struct mmc_request` — الطلب

```c
struct mmc_request {
    struct mmc_command *sbc;   /* optional: SET_BLOCK_COUNT قبل الـ transfer */
    struct mmc_command *cmd;   /* الأمر الأساسي */
    struct mmc_data    *data;  /* البيانات المصاحبة (لو في data transfer) */
    struct mmc_command *stop;  /* STOP_TRANSMISSION بعد الـ transfer */
    void (*done)(struct mmc_request *); /* callback لما الـ request يخلص */
};
```

والـ `mmc_command` بيحمل الأمر نفسه:

```c
struct mmc_command {
    u32 opcode;        /* رقم الأمر: CMD17 = read, CMD24 = write, etc. */
    u32 arg;           /* الـ argument: عنوان الـ block أو أي parameter */
    u32 resp[4];       /* الـ response من الكارت (R1/R2/R3/R6/R7) */
    unsigned int flags;/* نوع الـ response المتوقع (MMC_RSP_R1, R2, ...) */
    int error;         /* نتيجة التنفيذ */
};
```

---

### كيف الـ Structs بترتبط ببعض

```
struct mmc_host
    │
    ├──► ops ──► struct mmc_host_ops (vtable)
    │                 ├── request()          ← بيبعت mmc_request للـ hardware
    │                 ├── set_ios()          ← بيغير clock/voltage/bus width
    │                 ├── get_cd()           ← card detected?
    │                 ├── execute_tuning()   ← calibrate timing
    │                 └── card_busy()        ← DAT0 still low?
    │
    ├──► ios ──► struct mmc_ios
    │                 ├── clock             ← الـ clock rate الحالي
    │                 ├── bus_width         ← 1/4/8 bit
    │                 ├── timing            ← LEGACY/HS/DDR52/HS200/HS400
    │                 └── signal_voltage    ← 3.3V / 1.8V / 1.2V
    │
    └──► card ──► struct mmc_card
                      ├── host ──────────────────► (back pointer)
                      ├── cid ──► struct mmc_cid   (manufacturer, serial)
                      ├── csd ──► struct mmc_csd   (capacity, timing class)
                      ├── ext_csd ► struct mmc_ext_csd (HS200/HS400 caps, RPMB size)
                      └── part[] ► struct mmc_part  (boot0, boot1, RPMB, GP0-3)
```

```
مسار الـ I/O Request:
──────────────────────────────────────────────────────

mmc_blk (block driver)
    │
    │  يبني struct mmc_request
    ▼
mmc_wait_for_req(host, mrq)     ← Core API
    │
    │  يستدعي
    ▼
host->ops->request(host, mrq)   ← Hardware callback
    │
    │  DMA + hardware registers
    ▼
Physical Card
    │
    │  response + data
    ▼
host driver يستدعي mmc_request_done(host, mrq)
    │
    ▼
mrq->done(mrq)                  ← يرجع للـ mmc_blk
```

---

### الـ Command Classes — تفاصيل بروتوكول الـ MMC

الأوامر في الملف مقسّمة لـ **classes** حسب الوظيفة، وده تصميم من الـ MMC specification نفسها:

| Class | الوظيفة | أمثلة من الكود |
|---|---|---|
| 0 (Basic) | إدارة الـ bus وعنونة الكارت | `MMC_GO_IDLE_STATE (CMD0)`, `MMC_ALL_SEND_CID (CMD2)` |
| 2 (Block Read) | قراءة البيانات | `MMC_READ_SINGLE_BLOCK (CMD17)`, `MMC_READ_MULTIPLE_BLOCK (CMD18)` |
| 4 (Block Write) | كتابة البيانات | `MMC_WRITE_BLOCK (CMD24)`, `MMC_WRITE_MULTIPLE_BLOCK (CMD25)` |
| 5 (Erase) | مسح البيانات | `MMC_ERASE_GROUP_START (CMD35)`, `MMC_ERASE (CMD38)` |
| 6 (Write Protect) | حماية الكتابة | `MMC_SET_WRITE_PROT (CMD28)` |
| 10 (Switch) | تغيير الـ mode (HS, DDR, ...) | `MMC_SWITCH (CMD6)` |
| 11 (CmdQ) | Command Queuing (أوامر متوازية) | `MMC_QUE_TASK_PARAMS (CMD44)`, `MMC_EXECUTE_READ_TASK (CMD46)` |

#### الـ R1 Status Register — إزاي تقرأ حالة الكارت

بعد أي أمر، الكارت بيرد بـ **R1 response** وهو 32-bit status word:

```c
/* أهم bits في الـ R1 response */
#define R1_OUT_OF_RANGE     (1 << 31)  /* عنوان برا نطاق الكارت */
#define R1_ADDRESS_ERROR    (1 << 30)  /* عنوان misaligned */
#define R1_COM_CRC_ERROR    (1 << 23)  /* CRC error في الأمر */
#define R1_ILLEGAL_COMMAND  (1 << 22)  /* أمر مش مسموح في الـ state الحالي */
#define R1_CURRENT_STATE(x) ((x & 0x00001E00) >> 9)  /* 4 bits للـ state */
#define R1_READY_FOR_DATA   (1 << 8)   /* الكارت جاهز لـ transfer */
```

الـ `mmc_ready_for_data()` في الملف بيتحقق من bit 8 **و** إن الكارت في state **TRAN** (Transfer State):

```c
static inline bool mmc_ready_for_data(u32 status)
{
    /* بعض الكروت بتعمل misreport للـ status bits */
    return status & R1_READY_FOR_DATA &&
           R1_CURRENT_STATE(status) == R1_STATE_TRAN;
}
```

#### الـ Card State Machine

```
            CMD0
              │
              ▼
         ┌─────────┐
         │  IDLE   │◄──────────────────────────────────┐
         └────┬────┘                                   │
              │ CMD1 (MMC) / ACMD41 (SD) — send OCR    │
              ▼                                        │
         ┌─────────┐                                   │
         │  READY  │                                   │
         └────┬────┘                                   │
              │ CMD2 — send CID                        │
              ▼                                        │
         ┌─────────┐                                   │
         │  IDENT  │                                   │
         └────┬────┘                                   │
              │ CMD3 — set RCA                         │
              ▼                                        │
         ┌─────────┐    CMD7 (deselect)                │
         │  STBY   │──────────────────────────────────►│
         └────┬────┘                                   │
              │ CMD7 (select)                          │
              ▼                                        │
         ┌─────────┐    CMD0 or power cycle            │
         │  TRAN   │──────────────────────────────────►┘
         └────┬────┘
         ┌────┘────┐
    CMD18│         │CMD24/25
         ▼         ▼
    ┌─────────┐ ┌─────────┐
    │  DATA   │ │   RCV   │
    └─────────┘ └─────────┘
```

---

### الـ EXT_CSD — الجوهرة المخفية في eMMC

**الـ EXT_CSD** register هو 512-byte block من الـ configuration والـ status، موجود **بس في eMMC** (مش في SD). الـ MMC Core بيقرأه بـ CMD8 (`MMC_SEND_EXT_CSD`) بعد الـ initialization.

أهم fields موجودة في الملف:

```c
/* Timing — إيه أعلى speed مدعوم */
#define EXT_CSD_CARD_TYPE_HS200_1_8V  (1<<4)  /* 200 MHz SDR */
#define EXT_CSD_CARD_TYPE_HS400_1_8V  (1<<6)  /* 200 MHz DDR */
#define EXT_CSD_CARD_TYPE_HS400ES     (1<<8)  /* HS400 + Enhanced Strobe */

/* Bus Width — يتكتب بـ CMD6 (MMC_SWITCH) */
#define EXT_CSD_BUS_WIDTH_1   0
#define EXT_CSD_BUS_WIDTH_4   1
#define EXT_CSD_BUS_WIDTH_8   2
#define EXT_CSD_DDR_BUS_WIDTH_4  5
#define EXT_CSD_DDR_BUS_WIDTH_8  6

/* Command Queuing */
#define EXT_CSD_CMDQ_SUPPORT  308  /* هل الكارت يدعم CQ? */
#define EXT_CSD_CMDQ_DEPTH    307  /* عمق الـ queue (max 32) */
```

#### الـ MMC_SWITCH (CMD6) — إزاي بتغير الـ Mode

```c
/*
 * MMC_SWITCH argument format:
 *  [25:24] Access Mode: WRITE_BYTE (0x03) الأكثر استخداماً
 *  [23:16] Index: رقم الـ byte في EXT_CSD
 *  [15:08] Value: القيمة الجديدة
 *  [02:00] Command Set: عادةً 0
 */
#define MMC_SWITCH_MODE_WRITE_BYTE  0x03
```

مثلاً عشان تحوّل الكارت لـ HS200:
```
CMD6 arg = (0x03 << 24) | (EXT_CSD_HS_TIMING << 16) | (EXT_CSD_TIMING_HS200 << 8)
```

---

### إيه اللي الـ Framework بيمتلكه مقابل اللي بيفوّضه

#### الـ Core Framework يمتلك:
- **Card initialization sequence**: من CMD0 → CMD7 وكل خطواته
- **Card identification**: قراءة CID, CSD, EXT_CSD وتحليلها
- **Bus speed negotiation**: اختيار الأعلى speed مشترك بين الـ host والـ card
- **Power management**: suspend/resume, power-off notification
- **Tuning logic**: متى تعمل re-tune وكيف تدير الـ tuning state
- **Error recovery**: retry logic, reset sequences
- **Command queuing scheduler**: ترتيب الـ CQE requests

#### يُفوّض للـ Host Driver:
- **الـ DMA setup**: إزاي تعمل scatter-gather DMA
- **الـ Clock generation**: بتعمل `set_ios()` بالـ clock المطلوب
- **الـ Voltage switching**: تنفيذ الـ 3.3V → 1.8V switch
- **الـ Tuning algorithm**: `execute_tuning()` — كل controller عنده طريقته
- **الـ Card detect**: `get_cd()` — من GPIO ولا من register؟
- **الـ HS400 sequence**: `prepare_hs400_tuning()`, `hs400_complete()`

---

### نظرة عملية — Lifecycle كارت eMMC على i.MX8

```bash
# لما الـ kernel يبدأ:
# 1. الـ imx-esdhc driver يعمل probe
#    → يستدعي mmc_alloc_host()
#    → يملى host->ops بـ callbacks تبعه
#    → يستدعي mmc_add_host()

# 2. الـ MMC Core يبدأ mmc_rescan() في workqueue
#    → يضغط على الـ power
#    → يبعت CMD0 (GO_IDLE)
#    → يبعت CMD1 (SEND_OP_COND) يتعرف إنه eMMC
#    → يبعت CMD2 يجيب الـ CID → يبني struct mmc_cid
#    → يبعت CMD3 يديه RCA
#    → يبعت CMD9 يجيب الـ CSD → يبني struct mmc_csd
#    → يبعت CMD7 يعمل SELECT
#    → يبعت CMD8 يجيب الـ EXT_CSD → يبني struct mmc_ext_csd

# 3. الـ Core يقرر الأعلى speed:
#    → host supports HS400? + card supports HS400? → HS400 sequence

# 4. يعمل mmc_add_card() → يظهر /dev/mmcblk0
#    → الـ partition table اتقرأت
#    → /dev/mmcblk0boot0, mmcblk0boot1, mmcblk0rpmb ظهروا
```
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

> الملف `include/linux/mmc/mmc.h` هو **header-only** بالكامل — مفيش structs فيه، بس فيه كميّة ضخمة من الـ `#define` macros والـ `static inline` functions اللي بتحدد بروتوكول MMC/eMMC كامل. الفهم الحقيقي بييجي من استيعاب العلاقات بين المجموعات دي.

---

### 0. الـ Flags، Enums، والـ Config Options — Cheatsheet

#### أ. أوامر MMC القياسية (Command Classes)

| Class | Commands | الوظيفة |
|-------|----------|---------|
| 0 (Basic) | CMD0,1,2,3,4,7,9,10,12,13,15 | init, select, status |
| 1 (Stream Read) | CMD11 | stream read |
| 2 (Block Read) | CMD16,17,18 | block read |
| 3 (Stream Write) | CMD20 | stream write |
| 4 (Block Write) | CMD16,24,25,26,27 | block write |
| 5 (Erase) | CMD35,36,38 | erase groups |
| 6 (Write Protect) | CMD28,29,30 | WP blocks |
| 7 (Lock) | CMD42 | lock/unlock card |
| 8 (App Spec) | CMD55,56 | app commands |
| 9 (I/O) | CMD39,40 | fast I/O, IRQ state |
| 10 (Switch) | CMD6 | high-speed switch |
| 11 (CmdQ) | CMD44,45,46,47,48 | command queuing |

#### ب. أرقام الأوامر المهمة

| Macro | رقم | النوع | الرد | الوظيفة |
|-------|-----|-------|------|---------|
| `MMC_GO_IDLE_STATE` | 0 | bc | — | reset الكارت |
| `MMC_SEND_OP_COND` | 1 | bcr | R3 | negotiate voltage/OCR |
| `MMC_ALL_SEND_CID` | 2 | bcr | R2 | اقرأ الـ CID |
| `MMC_SET_RELATIVE_ADDR` | 3 | ac | R1 | assign RCA |
| `MMC_SWITCH` | 6 | ac | R1b | غيّر EXT_CSD byte |
| `MMC_SELECT_CARD` | 7 | ac | R1 | select/deselect |
| `MMC_SEND_EXT_CSD` | 8 | adtc | R1 | اقرأ EXT_CSD كامل |
| `MMC_SEND_CSD` | 9 | ac | R2 | اقرأ CSD register |
| `MMC_STOP_TRANSMISSION` | 12 | ac | R1b | وقّف transfer |
| `MMC_SEND_STATUS` | 13 | ac | R1 | poll card state |
| `MMC_READ_SINGLE_BLOCK` | 17 | adtc | R1 | اقرأ block واحد |
| `MMC_READ_MULTIPLE_BLOCK` | 18 | adtc | R1 | multi-block read |
| `MMC_WRITE_BLOCK` | 24 | adtc | R1 | اكتب block واحد |
| `MMC_WRITE_MULTIPLE_BLOCK` | 25 | adtc | R1 | multi-block write |
| `MMC_ERASE` | 38 | ac | R1b | نفّذ erase |
| `MMC_QUE_TASK_PARAMS` | 44 | ac | R1 | CmdQ: set task params |
| `MMC_EXECUTE_READ_TASK` | 46 | adtc | R1 | CmdQ: execute read |
| `MMC_EXECUTE_WRITE_TASK` | 47 | adtc | R1 | CmdQ: execute write |

**أنواع الأوامر:**
- `bc` = broadcast, no response
- `bcr` = broadcast with response
- `ac` = addressed command, no data
- `adtc` = addressed data transfer command

#### ج. بتات الـ R1 Status Register

| Bit Macro | Bit | النوع | الوظيفة |
|-----------|-----|-------|---------|
| `R1_OUT_OF_RANGE` | 31 | er,c | address خارج النطاق |
| `R1_ADDRESS_ERROR` | 30 | erx,c | خطأ في العنوان |
| `R1_BLOCK_LEN_ERROR` | 29 | er,c | block length غلط |
| `R1_ERASE_SEQ_ERROR` | 28 | er,c | تسلسل erase خاطئ |
| `R1_WP_VIOLATION` | 26 | erx,c | انتهاك write protection |
| `R1_CARD_IS_LOCKED` | 25 | sx,a | الكارت مقفول |
| `R1_LOCK_UNLOCK_FAILED` | 24 | erx,c | فشل lock/unlock |
| `R1_COM_CRC_ERROR` | 23 | er,b | CRC error في الأمر |
| `R1_ILLEGAL_COMMAND` | 22 | er,b | أمر غير صالح |
| `R1_CARD_ECC_FAILED` | 21 | ex,c | ECC failure |
| `R1_CC_ERROR` | 20 | erx,c | خطأ في card controller |
| `R1_ERROR` | 19 | erx,c | general error |
| `R1_READY_FOR_DATA` | 8 | sx,a | الكارت جاهز لبيانات |
| `R1_SWITCH_ERROR` | 7 | sx,c | فشل MMC_SWITCH |
| `R1_EXCEPTION_EVENT` | 6 | sr,a | BKOPS/urgent event |
| `R1_APP_CMD` | 5 | sr,c | الكارت في app mode |

**أنواع الـ bits:**
- `e` = error bit, `s` = status bit, `r` = set بعد command مباشرة, `x` = set أثناء execution
- `a` = cleared حسب card state, `b` = cleared بأمر جديد, `c` = cleared by read

#### د. حالات الكارت (Card States) في R1

| Macro | قيمة | الحالة |
|-------|------|--------|
| `R1_STATE_IDLE` | 0 | بعد power-on أو CMD0 |
| `R1_STATE_READY` | 1 | بعد CMD1 (OCR negotiation) |
| `R1_STATE_IDENT` | 2 | بعد CMD2 (CID identification) |
| `R1_STATE_STBY` | 3 | له RCA، مش selected |
| `R1_STATE_TRAN` | 4 | **transfer state** — نقطة التشغيل الأساسية |
| `R1_STATE_DATA` | 5 | جاري data transfer |
| `R1_STATE_RCV` | 6 | جاري استقبال بيانات |
| `R1_STATE_PRG` | 7 | جاري programming |
| `R1_STATE_DIS` | 8 | deselected أثناء programming |

#### هـ. EXT_CSD — أهم الـ Byte Offsets

| Macro | Offset | Access | الوظيفة |
|-------|--------|--------|---------|
| `EXT_CSD_CMDQ_MODE_EN` | 15 | R/W | تفعيل Command Queuing |
| `EXT_CSD_CACHE_CTRL` | 33 | R/W | تحكم في الـ cache |
| `EXT_CSD_POWER_OFF_NOTIFICATION` | 34 | R/W | إشعار قبل power-off |
| `EXT_CSD_PART_CONFIG` | 179 | R/W | partition access config |
| `EXT_CSD_BUS_WIDTH` | 183 | R/W | عرض الـ bus |
| `EXT_CSD_HS_TIMING` | 185 | R/W | high-speed timing mode |
| `EXT_CSD_REV` | 192 | RO | EXT_CSD revision |
| `EXT_CSD_CARD_TYPE` | 196 | RO | speed modes المدعومة |
| `EXT_CSD_SEC_CNT` | 212 | RO | إجمالي الـ sectors (4 bytes) |
| `EXT_CSD_BKOPS_EN` | 163 | R/W | تفعيل background operations |
| `EXT_CSD_RST_N_FUNCTION` | 162 | R/W | تحكم في RST_n pin |
| `EXT_CSD_RPMB_MULT` | 168 | RO | حجم RPMB partition |
| `EXT_CSD_BOOT_WP` | 173 | R/W | boot partition write protect |
| `EXT_CSD_CACHE_SIZE` | 249 | RO | حجم cache (4 bytes) |
| `EXT_CSD_PRE_EOL_INFO` | 267 | RO | pre-end-of-life indicator |
| `EXT_CSD_DEVICE_LIFE_TIME_EST_TYP_A` | 268 | RO | عمر النوع A |
| `EXT_CSD_DEVICE_LIFE_TIME_EST_TYP_B` | 269 | RO | عمر النوع B |
| `EXT_CSD_CMDQ_DEPTH` | 307 | RO | عمق queue |
| `EXT_CSD_CMDQ_SUPPORT` | 308 | RO | هل CmdQ مدعوم |

#### و. EXT_CSD_CARD_TYPE — أوضاع السرعة

| Bit Macro | Value | السرعة | الجهد |
|-----------|-------|--------|-------|
| `EXT_CSD_CARD_TYPE_HS_26` | bit 0 | 26 MHz | — |
| `EXT_CSD_CARD_TYPE_HS_52` | bit 1 | 52 MHz | — |
| `EXT_CSD_CARD_TYPE_DDR_1_8V` | bit 2 | 52 MHz DDR | 1.8V أو 3V |
| `EXT_CSD_CARD_TYPE_DDR_1_2V` | bit 3 | 52 MHz DDR | 1.2V |
| `EXT_CSD_CARD_TYPE_HS200_1_8V` | bit 4 | 200 MHz SDR | 1.8V |
| `EXT_CSD_CARD_TYPE_HS200_1_2V` | bit 5 | 200 MHz SDR | 1.2V |
| `EXT_CSD_CARD_TYPE_HS400_1_8V` | bit 6 | 200 MHz DDR | 1.8V |
| `EXT_CSD_CARD_TYPE_HS400_1_2V` | bit 7 | 200 MHz DDR | 1.2V |
| `EXT_CSD_CARD_TYPE_HS400ES` | bit 8 | HS400 Enhanced Strobe | — |

#### ز. EXT_CSD_BUS_WIDTH — قيم عرض الـ Bus

| Macro | قيمة | الوضع |
|-------|------|-------|
| `EXT_CSD_BUS_WIDTH_1` | 0 | 1-bit SDR |
| `EXT_CSD_BUS_WIDTH_4` | 1 | 4-bit SDR |
| `EXT_CSD_BUS_WIDTH_8` | 2 | 8-bit SDR |
| `EXT_CSD_DDR_BUS_WIDTH_4` | 5 | 4-bit DDR |
| `EXT_CSD_DDR_BUS_WIDTH_8` | 6 | 8-bit DDR |
| `EXT_CSD_BUS_WIDTH_STROBE` | BIT(7) | Enhanced Strobe (HS400ES) |

#### ح. EXT_CSD_HS_TIMING — قيم التوقيت

| Macro | قيمة | الوضع |
|-------|------|-------|
| `EXT_CSD_TIMING_BC` | 0 | Backwards Compatibility |
| `EXT_CSD_TIMING_HS` | 1 | High Speed (52 MHz) |
| `EXT_CSD_TIMING_HS200` | 2 | HS200 (200 MHz SDR) |
| `EXT_CSD_TIMING_HS400` | 3 | HS400 (200 MHz DDR) |

#### ط. MMC_SWITCH Access Modes

| Macro | قيمة | الوظيفة |
|-------|------|---------|
| `MMC_SWITCH_MODE_CMD_SET` | 0x00 | غيّر command set |
| `MMC_SWITCH_MODE_SET_BITS` | 0x01 | set bits محددة |
| `MMC_SWITCH_MODE_CLEAR_BITS` | 0x02 | clear bits محددة |
| `MMC_SWITCH_MODE_WRITE_BYTE` | 0x03 | **اكتب byte كامل** (الأكثر استخداماً) |

#### ي. Erase/Trim/Discard Arguments

| Macro | قيمة | الوظيفة |
|-------|------|---------|
| `MMC_ERASE_ARG` | 0x00000000 | erase عادي |
| `MMC_SECURE_ERASE_ARG` | 0x80000000 | secure erase |
| `MMC_TRIM_ARG` | 0x00000001 | trim (hint للـ controller) |
| `MMC_DISCARD_ARG` | 0x00000003 | discard blocks |
| `MMC_SECURE_TRIM1_ARG` | 0x80000001 | المرحلة الأولى من secure trim |
| `MMC_SECURE_TRIM2_ARG` | 0x80008000 | المرحلة الثانية من secure trim |

#### ك. BKOPS و Exception Events

| Macro | الوظيفة |
|-------|---------|
| `EXT_CSD_URGENT_BKOPS` | BIT(0) — يجب تنفيذ BKOPS فوراً |
| `EXT_CSD_DYNCAP_NEEDED` | BIT(1) — dynamic capacity جديد مطلوب |
| `EXT_CSD_SYSPOOL_EXHAUSTED` | BIT(2) — system resources نفدت |
| `EXT_CSD_BKOPS_LEVEL_2` | 0x2 — مستوى critical لـ BKOPS |
| `EXT_CSD_MANUAL_BKOPS_MASK` | 0x01 — manual BKOPS |
| `EXT_CSD_AUTO_BKOPS_MASK` | 0x02 — automatic BKOPS |

#### ل. CSD Structure Versions

| Macro | قيمة | الـ Spec |
|-------|------|---------|
| `CSD_STRUCT_VER_1_0` | 0 | spec 1.0–1.2 |
| `CSD_STRUCT_VER_1_1` | 1 | spec 1.4–2.2 |
| `CSD_STRUCT_VER_1_2` | 2 | spec 3.1–4.1 |
| `CSD_STRUCT_EXT_CSD` | 3 | version في EXT_CSD |

#### م. Power Off Notification Values

| Macro | قيمة | المعنى |
|-------|------|--------|
| `EXT_CSD_NO_POWER_NOTIFICATION` | 0 | مفيش إشعار |
| `EXT_CSD_POWER_ON` | 1 | الكارت شغال |
| `EXT_CSD_POWER_OFF_SHORT` | 2 | power-off سريع |
| `EXT_CSD_POWER_OFF_LONG` | 3 | power-off مع flush كامل |

---

### 1. الـ Structs المهمة في الملف

الملف `mmc.h` **لا يحتوي على structs**، لكنه يعرّف المعطيات البروتوكولية الخام اللي بتُملأ فيها الـ structs الموجودة في ملفات MMC الأخرى. الـ structs الرئيسية اللي بتستخدم هذه التعريفات هي:

#### `struct mmc_command` (من `include/linux/mmc/core.h`)
```c
struct mmc_command {
    u32         opcode;    /* رقم الأمر — من macros في mmc.h */
    u32         arg;       /* argument — e.g. MMC_SWITCH_MODE_WRITE_BYTE */
    u32         resp[4];   /* response — R1 bits يُحلَّل بـ R1_* macros */
    unsigned int flags;
    /* ... */
};
```

#### `struct mmc_card` (من `include/linux/mmc/card.h`)
بتخزن:
- `ext_csd` — struct فيه parsed values من EXT_CSD offsets
- `csd` — CSD fields
- `cid` — CID fields
- `state` — بيتغير حسب R1_STATE_* macros

#### `struct mmc_ios` (من `include/linux/mmc/host.h`)
بتتحكم في:
- `bus_width` — بيتعيّن بـ `EXT_CSD_BUS_WIDTH_*` values
- `timing` — بيتعيّن بـ `EXT_CSD_TIMING_*` values

---

### 2. رسائل ASCII لعلاقات المفاهيم

#### أ. تسلسل بيانات MMC_SWITCH (CMD6)

```
MMC_SWITCH argument (32-bit):
┌──────────┬────────────┬────────────┬───────────┬───────────┐
│ [31:26]  │  [25:24]   │  [23:16]   │  [15:08]  │  [02:00]  │
│  Always  │ Access     │ EXT_CSD    │  Value    │ Cmd Set   │
│    0     │  Mode      │  Offset    │  Byte     │           │
│          │ 0x00..0x03 │  0..511    │  0..255   │           │
└──────────┴────────────┴────────────┴───────────┴───────────┘
              ↑                ↑            ↑
   MMC_SWITCH_MODE_*    EXT_CSD_*      القيمة الجديدة
```

#### ب. بنية R1 Status Register

```
R1 Status Register (32-bit):
Bit: 31  30  29  28  27  26  25  24  23  22  21  20  19  18  17  16
     ↓   ↓   ↓   ↓   ↓   ↓   ↓   ↓   ↓   ↓   ↓   ↓   ↓   ↓   ↓   ↓
     OOR AER BLE ESE EP  WPV CIL LUF CCE IC  ECC CCE ERR UN  OVR CID

Bit: 15  14  13  12  11  10   9   8   7   6   5   4   3   2   1   0
     WES EED ERS  -   -   -   - RFD SWE EXE APC  -   -   -   -   -
     ↑               ↑           ↑   ↑   ↑   ↑
   WP Erase      Reserved    Ready Switch Exc App
   Skip                     4Data Error Event Cmd

Card State (bits 12:9) = R1_STATE_IDLE..R1_STATE_DIS
```

#### ج. علاقة EXT_CSD بالـ Macros

```
EXT_CSD (512 bytes)
┌─────────────────────────────────────────────────┐
│ Byte 15:  CMDQ_MODE_EN  ← EXT_CSD_CMDQ_MODE_EN │
│ Byte 33:  CACHE_CTRL    ← EXT_CSD_CACHE_CTRL   │
│ Byte 163: BKOPS_EN      ← EXT_CSD_BKOPS_EN     │
│ Byte 179: PART_CONFIG   ← EXT_CSD_PART_CONFIG  │
│ Byte 183: BUS_WIDTH     ← EXT_CSD_BUS_WIDTH    │
│ Byte 185: HS_TIMING     ← EXT_CSD_HS_TIMING    │
│ Byte 192: REV           ← EXT_CSD_REV          │
│ Byte 196: CARD_TYPE     ← EXT_CSD_CARD_TYPE    │
│ Byte 212: SEC_CNT[3:0]  ← EXT_CSD_SEC_CNT      │ ← 4 bytes
│ Byte 249: CACHE_SIZE[3:0]← EXT_CSD_CACHE_SIZE  │ ← 4 bytes
│ Byte 267: PRE_EOL_INFO  ← EXT_CSD_PRE_EOL_INFO │
│ Byte 307: CMDQ_DEPTH    ← EXT_CSD_CMDQ_DEPTH   │
│ Byte 308: CMDQ_SUPPORT  ← EXT_CSD_CMDQ_SUPPORT │
└─────────────────────────────────────────────────┘
       ↑ تُقرأ كلها بـ MMC_SEND_EXT_CSD (CMD8)
       ↑ يُعدَّل كل byte بـ MMC_SWITCH (CMD6)
```

---

### 3. دورة حياة الكارت — Lifecycle Diagram

```
Power ON / CMD0 (RESET)
         │
         ▼
   ┌─────────────┐
   │  IDLE State │  R1_STATE_IDLE = 0
   └─────────────┘
         │ CMD1 (SEND_OP_COND) — negotiate OCR/voltage
         ▼
   ┌─────────────┐
   │ READY State │  R1_STATE_READY = 1
   └─────────────┘
         │ CMD2 (ALL_SEND_CID) — read 128-bit CID
         ▼
   ┌─────────────┐
   │ IDENT State │  R1_STATE_IDENT = 2
   └─────────────┘
         │ CMD3 (SET_RELATIVE_ADDR) — assign RCA
         ▼
   ┌─────────────┐
   │ STBY State  │  R1_STATE_STBY = 3  ← deselected
   └─────────────┘
         │ CMD7 (SELECT_CARD) — select by RCA
         ▼
   ┌─────────────┐
   │ TRAN State  │  R1_STATE_TRAN = 4  ← نقطة التشغيل الطبيعية
   └─────────────┘
    │         │         │           │
    ▼         ▼         ▼           ▼
 CMD8      CMD6      CMD17/18    CMD24/25
SEND_    SWITCH    READ_BLOCK  WRITE_BLOCK
EXT_CSD  (R/W)
    │         │         │           │
    │         │    DATA State    RCV State
    │         │    R1_STATE=5   R1_STATE=6
    │         │         │           │
    │         │    CMD12 STOP   CMD12 STOP
    │         │         │           │
    │         │         └─────┬─────┘
    │         │               ▼
    │         │          TRAN State ← (back)
    │         │
    │    (write to NAND)
    │         │
    │    PRG State (R1_STATE_PRG = 7)
    │         │
    │    poll CMD13 until R1_READY_FOR_DATA
    │         │
    │    TRAN State ← (back)
    │
    ▼
 parse EXT_CSD bytes
    │
    ├─ EXT_CSD_CARD_TYPE[196] → choose speed mode
    ├─ EXT_CSD_BUS_WIDTH[183] → set via CMD6
    ├─ EXT_CSD_HS_TIMING[185] → set via CMD6
    └─ EXT_CSD_CMDQ_SUPPORT[308] → enable CmdQ if supported
```

---

### 4. Call Flow Diagrams

#### أ. تسلسل تهيئة الكارت (Initialization Flow)

```
mmc_init_card()
  │
  ├─► CMD0 (MMC_GO_IDLE_STATE)
  │     → card resets
  │
  ├─► CMD1 (MMC_SEND_OP_COND, arg=OCR)
  │     → R3 response: OCR register
  │     → check MMC_CARD_BUSY bit (bit31)
  │     → loop until card not busy
  │
  ├─► CMD2 (MMC_ALL_SEND_CID)
  │     → R2 response: 128-bit CID
  │     → parse: manufacturer, product name, serial
  │
  ├─► CMD3 (MMC_SET_RELATIVE_ADDR)
  │     → R1 response: assigned RCA
  │
  ├─► CMD9 (MMC_SEND_CSD, arg=RCA<<16)
  │     → R2 response: CSD register
  │     → parse: capacity, block size, speed
  │     → check CSD_STRUCT_VER_* for EXT_CSD support
  │
  ├─► CMD7 (MMC_SELECT_CARD, arg=RCA<<16)
  │     → R1: card enters TRAN state
  │
  └─► CMD8 (MMC_SEND_EXT_CSD)
        → adtc: 512-byte data transfer
        → parse all EXT_CSD_* offsets
        → determine: CARD_TYPE, CMDQ_SUPPORT, etc.
```

#### ب. تسلسل رفع السرعة (Speed Upgrade Flow)

```
mmc_select_timing()
  │
  ├─ Read EXT_CSD_CARD_TYPE[196]
  │   ├─ EXT_CSD_CARD_TYPE_HS400? → go HS400 path
  │   ├─ EXT_CSD_CARD_TYPE_HS200? → go HS200 path
  │   └─ EXT_CSD_CARD_TYPE_HS?    → go HS path
  │
  ├─► CMD6 (MMC_SWITCH)
  │     arg = [MMC_SWITCH_MODE_WRITE_BYTE << 24]
  │           | [EXT_CSD_HS_TIMING << 16]
  │           | [EXT_CSD_TIMING_HS200 << 8]
  │     → R1b: wait for programming done
  │     → check R1_SWITCH_ERROR bit
  │
  ├─► CMD6 (MMC_SWITCH) — set bus width
  │     arg uses EXT_CSD_BUS_WIDTH[183]
  │     value = EXT_CSD_BUS_WIDTH_8 or EXT_CSD_DDR_BUS_WIDTH_8
  │
  ├─► mmc_execute_tuning()  [HS200/HS400]
  │     → CMD21 (MMC_SEND_TUNING_BLOCK_HS200)
  │     → adjust tap delays until no errors
  │
  └─► CMD13 (MMC_SEND_STATUS)
        → R1: verify R1_SWITCH_ERROR is 0
        → verify R1_CURRENT_STATE == R1_STATE_TRAN
```

#### ج. تسلسل قراءة بيانات (Read Flow)

```
mmc_read_blocks(card, buf, sector, count)
  │
  ├─ count > 1 ?
  │   ├─ YES → CMD18 (MMC_READ_MULTIPLE_BLOCK)
  │   └─ NO  → CMD17 (MMC_READ_SINGLE_BLOCK)
  │
  ├─► Send command
  │     arg = sector address (byte-addressed for <2GB, sector for ≥2GB)
  │     response = R1
  │     check: R1_OUT_OF_RANGE, R1_ADDRESS_ERROR, R1_BLOCK_LEN_ERROR
  │
  ├─► DMA/PIO data transfer
  │     card enters R1_STATE_DATA
  │
  ├─► [if multi-block] CMD12 (MMC_STOP_TRANSMISSION)
  │     → R1b: wait for card to finish
  │
  └─► CMD13 (MMC_SEND_STATUS)
        → poll until mmc_ready_for_data(status) == true
          i.e.: R1_READY_FOR_DATA && R1_STATE_TRAN
```

#### د. تسلسل Command Queue (CmdQ Flow)

```
cmdq_issue_task(tag, sector, count, is_write)
  │
  ├─► CMD44 (MMC_QUE_TASK_PARAMS)
  │     arg[20:16] = task_id
  │     arg[30] = is_write
  │     arg[15:0] = block_count
  │     → R1
  │
  ├─► CMD45 (MMC_QUE_TASK_ADDR)
  │     arg[31:0] = sector_address
  │     → R1
  │
  ├─► CMD46/47 (EXECUTE_READ/WRITE_TASK)
  │     arg[20:16] = task_id
  │     → R1 + data transfer
  │
  └─► [if abort needed] CMD48 (MMC_CMDQ_TASK_MGMT)
        arg[20:16] = task_id
        → R1b
```

#### هـ. تسلسل Erase/Trim

```
mmc_erase(card, from, nr, arg)
  │
  ├─► CMD35 (MMC_ERASE_GROUP_START)
  │     arg = start_sector
  │     → R1
  │
  ├─► CMD36 (MMC_ERASE_GROUP_END)
  │     arg = end_sector
  │     → R1
  │
  └─► CMD38 (MMC_ERASE)
        arg = one of:
          MMC_ERASE_ARG          → normal erase
          MMC_TRIM_ARG           → trim (map as free)
          MMC_DISCARD_ARG        → discard (lazy free)
          MMC_SECURE_ERASE_ARG   → crypto erase
          MMC_SECURE_TRIM1_ARG   → secure trim phase 1
          MMC_SECURE_TRIM2_ARG   → secure trim phase 2
        → R1b (long busy)
        → poll CMD13 until TRAN state
```

---

### 5. استراتيجية الـ Locking

الملف `mmc.h` نفسه **لا يعرّف locks**، لكنه يُقدّم المعلومات اللازمة لفهم ليه الـ locking ضروري في الـ MMC subsystem.

#### أ. ليه Locking ضروري؟

```
Multi-threaded access scenario:
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Block Layer  │    │  RPMB Driver │    │  SDIO Driver │
│ (read/write) │    │  (security)  │    │  (WiFi/BT)   │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                   │                   │
       └───────────────────┴───────────────────┘
                           │
                    ┌──────▼──────┐
                    │  MMC Host   │  ← one physical bus
                    │  Controller │    يسمح بأمر واحد فقط
                    └─────────────┘
```

الكارت في أي وقت يكون في حالة واحدة فقط (IDLE/TRAN/DATA/...) — لازم يكون فيه lock يمنع الـ concurrent access.

#### ب. تسلسل الـ Locks في MMC Core

```
struct mmc_host.lock (spinlock)
  └─ يحمي: ios state, clock, power state
       ↓ (يُطلق قبل command)

struct mmc_host.claim_mutex أو mmc_bus_get()
  └─ يحمي: ownership of the bus
       ↓ (يُأخذ أولاً)

Per-command: R1b response + polling CMD13
  └─ card-level serialization
     (الكارت نفسه يرفض أوامر جديدة وهو في PRG state)
```

#### ج. ترتيب الـ Locks (Lock Ordering)

```
مهم: لازم تاخد الـ locks بالترتيب ده عشان تتجنب deadlock

1. mmc_host->claim_mutex     (أعلى مستوى — bus ownership)
       ↓
2. mmc_host->lock  (spinlock — ios/power state)
       ↓
3. card->ext_csd state (implicit — under bus ownership)

عكس الترتيب ده = deadlock محتمل
```

#### د. EXT_CSD Write Protection Strategy

```
EXT_CSD_BOOT_WP[173] بيتحكم في 4 أوضاع:

┌──────────────────────────────────────────┐
│  EXT_CSD_BOOT_WP_B_PWR_WP_EN  (0x01)    │ ← power-cycle WP (مؤقت)
│  EXT_CSD_BOOT_WP_B_PERM_WP_EN (0x04)    │ ← permanent WP (دائم! لا رجعة)
│  EXT_CSD_BOOT_WP_B_PWR_WP_DIS (0x40)    │ ← disable power WP
│  EXT_CSD_BOOT_WP_B_PERM_WP_DIS (0x10)   │ ← disable perm WP (مرة واحدة)
└──────────────────────────────────────────┘

تحذير: PERM_WP_EN بيُحرق OTP bits في الكارت — مش reversible!
Kernel بيتحقق من الـ flags دي قبل أي write لـ boot partition.
```

#### هـ. RPMB Security Locking

```
RPMB (Replay Protected Memory Block):
  EXT_CSD_PART_CONFIG_ACC_RPMB = 0x3

للوصول لـ RPMB:
1. خذ bus lock
2. CMD6: switch partition to RPMB (EXT_CSD_PART_CONFIG = 0x3)
3. نفّذ RPMB frame (HMAC-SHA256 signed)
4. CMD6: switch back to user partition
5. حرر bus lock

الـ lock هنا مش بس ضد race condition — ده security protocol
لأن الـ RPMB بيعد الـ write counter ويرفض replay attacks
```

---

### 6. الـ Inline Functions

#### `mmc_op_multi(u32 opcode)`
```c
/* Returns true if opcode is a multi-block operation */
static inline bool mmc_op_multi(u32 opcode)
{
    return opcode == MMC_WRITE_MULTIPLE_BLOCK ||  /* CMD25 */
           opcode == MMC_READ_MULTIPLE_BLOCK;     /* CMD18 */
}
```
**بتُستخدم لـ:** تحديد هل لازم CMD12 (STOP_TRANSMISSION) بعد الأمر ولا لا.

#### `mmc_op_tuning(u32 opcode)`
```c
/* Returns true if opcode is a tuning command */
static inline bool mmc_op_tuning(u32 opcode)
{
    return opcode == MMC_SEND_TUNING_BLOCK ||         /* CMD19 — HS200 */
           opcode == MMC_SEND_TUNING_BLOCK_HS200;     /* CMD21 — HS200 alt */
}
```
**بتُستخدم لـ:** تخطي الـ error checking العادي لأن tuning commands بتُرسل بيانات عشوائية مقصودة.

#### `mmc_ready_for_data(u32 status)`
```c
/* Returns true only when card is truly ready for data in TRAN state */
static inline bool mmc_ready_for_data(u32 status)
{
    return status & R1_READY_FOR_DATA &&
           R1_CURRENT_STATE(status) == R1_STATE_TRAN;
}
```
**تحذير مهم:** بعض الكروت بتضرب بشكل خاطئ في الـ status bits — ولكده الـ kernel بيتحقق من **الاتنين معاً**: الـ ready bit والـ state field.

```
R1_CURRENT_STATE(x) = (x & 0x00001E00) >> 9  ← 4 bits فقط
```

---

### 7. ملخص — خريطة المعرفة الكاملة

```
mmc.h
  │
  ├─── MMC Commands (CMD0..CMD48)
  │         ↓
  │    struct mmc_command.opcode  (في core.h)
  │
  ├─── R1 Status Bits (R1_*)
  │         ↓
  │    struct mmc_command.resp[0]  → يُحلَّل بعد كل أمر
  │         ↓
  │    mmc_ready_for_data()  → polling loop
  │
  ├─── EXT_CSD Offsets (EXT_CSD_*)
  │         ↓
  │    CMD8 (SEND_EXT_CSD) → 512 bytes
  │         ↓
  │    struct mmc_ext_csd  (في card.h) → parsed fields
  │         ↓
  │    CMD6 (SWITCH) + MMC_SWITCH_MODE_* → write changes back
  │
  ├─── Card Type Bits (EXT_CSD_CARD_TYPE_*)
  │         ↓
  │    struct mmc_ios.timing  → EXT_CSD_TIMING_*
  │    struct mmc_ios.bus_width → EXT_CSD_BUS_WIDTH_*
  │
  ├─── Erase Args (MMC_*_ARG)
  │         ↓
  │    CMD35/36/38 sequence
  │
  └─── CCC Bits (CCC_*)
            ↓
       struct mmc_card.csd.cmdclass → feature detection
```
## Phase 4: شرح الـ Functions

> **الملف:** `include/linux/mmc/mmc.h`
> الملف ده header بحت — مفيهوش functions كتير، أغلب محتواه macros وconstants. الـ functions الموجودة هي `static inline` utilities بتشتغل على الـ MMC command opcodes وstatus registers. الشرح هيتناول كل function بالتفصيل، وكل macro group بالسياق.

---

### جدول الـ Functions والـ Macros — Cheatsheet

| الاسم | النوع | الغرض |
|---|---|---|
| `mmc_op_multi()` | `static inline bool` | بتشيك لو الـ opcode هو multi-block read/write |
| `mmc_op_tuning()` | `static inline bool` | بتشيك لو الـ opcode هو tuning command |
| `mmc_ready_for_data()` | `static inline bool` | بتشيك لو الكارت جاهز لاستقبال data (TRAN state + R1_READY_FOR_DATA) |
| `mmc_driver_type_mask(n)` | `macro` | بتوّلد mask للـ driver strength type رقم n |
| `R1_STATUS(x)` | `macro` | بتعزل error/status bits من الـ R1 response |
| `R1_CURRENT_STATE(x)` | `macro` | بتستخرج الـ card state الحالية (4 bits) من الـ R1 response |

---

### ### Group 1: Command Opcode Helpers

الـ group ده بيوفر abstractions فوق الـ raw command numbers. الـ MMC protocol بيستخدم opcodes (CMDn) مرقمة، والـ host driver محتاج يعرف يصنّف الـ opcode قبل ما يبعت الـ request عشان يحدد behavior زي الـ multi-block mode أو الـ tuning sequence.

---

#### `mmc_op_multi()`

```c
static inline bool mmc_op_multi(u32 opcode)
{
    return opcode == MMC_WRITE_MULTIPLE_BLOCK ||
           opcode == MMC_READ_MULTIPLE_BLOCK;
}
```

**ايه اللي بتعمله:**
بتشيك لو الـ opcode هو CMD18 (`MMC_READ_MULTIPLE_BLOCK`) أو CMD25 (`MMC_WRITE_MULTIPLE_BLOCK`). ده مهم لأن الـ multi-block transfers بتتطلب `MMC_STOP_TRANSMISSION` (CMD12) في الآخر أو `MMC_SET_BLOCK_COUNT` (CMD23) قبلها، بخلاف الـ single-block commands.

**Parameters:**
- `opcode` — الـ `u32` اللي بيحمل رقم الـ MMC command

**Return value:**
- `true` لو الـ opcode هو read/write multiple block
- `false` في أي حالة تانية

**Key details:**
- لا locking، لا side effects — pure predicate
- بتتستخدم في الـ `mmc_request` path قبل إرسال الـ command عشان يقرر الـ host driver يضيف STOP_TRANSMISSION ولا لأ
- الـ `MMC_SEND_TUNING_BLOCK` (CMD19) عنده نفس رقم `MMC_BUS_TEST_W` في التعريفات، ده intentional overlap في الـ spec لأن HS200 tuning مختلف

**Who calls it:**
`mmc_blk_mq_issue_rq()` وغيرها في `drivers/mmc/core/block.c` لتحديد نوع الـ transfer قبل issue.

---

#### `mmc_op_tuning()`

```c
static inline bool mmc_op_tuning(u32 opcode)
{
    return opcode == MMC_SEND_TUNING_BLOCK ||
           opcode == MMC_SEND_TUNING_BLOCK_HS200;
}
```

**ايه اللي بتعمله:**
بتحدد لو الـ command هو tuning command — إما CMD19 (`MMC_SEND_TUNING_BLOCK`) لـ SDR50/SDR104، أو CMD21 (`MMC_SEND_TUNING_BLOCK_HS200`) لـ eMMC HS200. الـ tuning commands ليها data phase خاصة — بترجع fixed 64/128-byte pattern والـ host controller بيستخدمها لمعايرة الـ sampling point.

**Parameters:**
- `opcode` — رقم الـ command

**Return value:**
- `true` لو هو tuning command
- `false` غير كده

**Key details:**
- الـ tuning sequence بتترسل بدون DMA في الغالب — الـ host controller بيتعامل معاها internally
- لازم الـ host controller يدعم `MMC_CAP_HW_RESET` و tuning capability
- الـ caller بيستخدم الـ return عشان يعطّل features زي الـ `sdhci_send_tuning()` في حالة الـ tuning requests عشان ميبعتش STOP_TRANSMISSION

**Who calls it:**
`mmc_execute_tuning()` في `drivers/mmc/core/mmc.c` وكمان الـ SDHCI host driver في `drivers/mmc/host/sdhci.c`.

---

### ### Group 2: Card Status Helpers (R1 Response)

الـ R1 response هي 32-bit card status بيرجعها الكارت بعد كل command. فيها error bits وstatus bits وcard state machine. الـ group ده بيوفر helpers عشان تفسر الـ R1 بشكل صحيح.

---

#### `mmc_ready_for_data()`

```c
static inline bool mmc_ready_for_data(u32 status)
{
    /*
     * Some cards mishandle the status bits, so make sure to check both the
     * busy indication and the card state.
     */
    return status & R1_READY_FOR_DATA &&
           R1_CURRENT_STATE(status) == R1_STATE_TRAN;
}
```

**ايه اللي بتعمله:**
بتتحقق إن الكارت جاهز لاستقبال أو إرسال data. الشرط المزدوج ده مهم لأن بعض الكروت بتعمل `R1_READY_FOR_DATA` set حتى لما تكون في state مش TRAN، وده behavior خاطئ من الكارت بس موجود في الواقع. الـ function بتتجنب الوقوع في الفخ ده بفرض التحقق من الحالتين مع بعض.

**Parameters:**
- `status` — الـ 32-bit R1 status word المستخرج من الـ MMC response

**Return value:**
- `true` لو `R1_READY_FOR_DATA` set وكمان الكارت في `R1_STATE_TRAN` (state=4)
- `false` لو أي شرط منهم مش متحقق

**Key details:**
- `R1_READY_FOR_DATA` = bit 8 في الـ R1 response
- `R1_CURRENT_STATE(x)` = `(x & 0x00001E00) >> 9` — بتعطي 4-bit state value
- `R1_STATE_TRAN` = 4 — ده الـ Transfer state اللي الكارت بيكون فيه جاهز للـ data transfer
- لو الكارت في `R1_STATE_PRG` (programming) أو `R1_STATE_RCV` (receiving)، الـ function بترجع false حتى لو `R1_READY_FOR_DATA` set

**Pseudocode flow:**
```
mmc_ready_for_data(status):
    ready_bit  = (status >> 8) & 1         // R1_READY_FOR_DATA
    card_state = (status >> 9) & 0xF       // R1_CURRENT_STATE
    return (ready_bit == 1) AND (card_state == 4)  // 4 = TRAN
```

**Who calls it:**
`mmc_poll_for_busy()` في `drivers/mmc/core/mmc_ops.c` بتـ poll الكارت بعد operations زي erase أو SWITCH عشان تتأكد إن الكارت رجع ready. كمان بتتستخدم في الـ data transfer completion path.

---

#### `R1_STATUS(x)` — Macro

```c
#define R1_STATUS(x)  (x & 0xFFF9A000)
```

**ايه اللي بيعمله:**
بيعزل bits الـ error والـ status من الـ R1 response بـ masking. الـ mask `0xFFF9A000` بيحتفظ بالـ bits اللي هي error indicators (bits 13-31 بس مش كلهم) ويشيل الـ bits الخاصة بالـ state والـ ready flags.

**Key details:**
- بيتستخدم للتحقق السريع إن مفيش error في الـ R1 response
- لو `R1_STATUS(status) != 0`، فيه error أو warning محتاج investigation
- الـ bits المحتفظ بيها تشمل: `R1_OUT_OF_RANGE`, `R1_ADDRESS_ERROR`, `R1_BLOCK_LEN_ERROR`, `R1_ERASE_SEQ_ERROR`, وغيرها

---

#### `R1_CURRENT_STATE(x)` — Macro

```c
#define R1_CURRENT_STATE(x)  ((x & 0x00001E00) >> 9)  /* sx, b (4 bits) */
```

**ايه اللي بيعمله:**
بيستخرج الـ 4-bit card state من bits [12:9] في الـ R1. الـ state machine للـ MMC card بتتحرك بين الـ states دي:

| Value | State | المعنى |
|---|---|---|
| 0 | `R1_STATE_IDLE` | بعد power-up أو CMD0 |
| 1 | `R1_STATE_READY` | بعد CMD1 (OCR accepted) |
| 2 | `R1_STATE_IDENT` | Identification mode |
| 3 | `R1_STATE_STBY` | Standby — مش selected |
| 4 | `R1_STATE_TRAN` | Transfer — جاهز للـ data |
| 5 | `R1_STATE_DATA` | بيبعت data |
| 6 | `R1_STATE_RCV` | بيستقبل data |
| 7 | `R1_STATE_PRG` | Programming/busy |
| 8 | `R1_STATE_DIS` | Disconnect |

---

### ### Group 3: Driver Type Mask Macro

---

#### `mmc_driver_type_mask(n)` — Macro

```c
#define mmc_driver_type_mask(n)  (1 << (n))
```

**ايه اللي بيعمله:**
بيولّد bitmask للـ driver strength type رقم `n`. الـ eMMC spec بتعرّف 4 driver strength types (Type 0–3) موجودة في `EXT_CSD_DRIVER_STRENGTH` (offset 197). الـ host بيستخدم الـ mask ده لمطابقة الـ driver type المدعوم من الكارت مع اللي بيدعمه الـ host controller.

**Parameters:**
- `n` — رقم الـ driver type (0 إلى 3 عادةً)

**Return value:**
- `(1 << n)` — integer bitmask

**Key details:**
- Type 0: 50 ohm pull-up، الـ default
- Type 1: 33 ohm — أقوى drive
- Type 2: 66 ohm — أضعف drive
- Type 3: 100 ohm — الأضعف
- بيتستخدم في `mmc_select_driver_type()` في `drivers/mmc/core/mmc.c` لتحديد أنسب driver type بناءً على قدرات الكارت والـ host

---

### ### Group 4: Command Classes Overview

الـ header بيعرّف الـ MMC commands مقسّمة لـ classes حسب الـ MMC spec v4.x. ده مهم للـ driver لأن الـ `CSD` بيحتوي على `CCC` field (Card Command Classes) اللي بيقول إيه الـ classes المدعومة من الكارت.

| Class | Macro | Commands الرئيسية |
|---|---|---|
| 0 — Basic | `CCC_BASIC` | CMD0,1,2,3,4,7,9,10,12,13,15 |
| 1 — Stream Read | `CCC_STREAM_READ` | CMD11 |
| 2 — Block Read | `CCC_BLOCK_READ` | CMD16,17,18 |
| 3 — Stream Write | `CCC_STREAM_WRITE` | CMD20 |
| 4 — Block Write | `CCC_BLOCK_WRITE` | CMD16,24,25,26,27 |
| 5 — Erase | `CCC_ERASE` | CMD35,36,38 |
| 6 — Write Protect | `CCC_WRITE_PROT` | CMD28,29,30 |
| 7 — Lock Card | `CCC_LOCK_CARD` | CMD42 |
| 8 — App Specific | `CCC_APP_SPEC` | CMD55,56 |
| 9 — I/O Mode | `CCC_IO_MODE` | CMD39,40 |
| 10 — Switch | `CCC_SWITCH` | CMD6 |
| 11 — Command Queue | — | CMD44,45,46,47,48 |

---

### ### Group 5: EXT_CSD Field Index — Reference

الـ `EXT_CSD` هو 512-byte register في الكارت بيتقرأ بـ CMD8. كل byte فيه له offset معرّف بـ macro. ده أهم الـ fields:

| Macro | Offset | Access | الوظيفة |
|---|---|---|---|
| `EXT_CSD_CMDQ_MODE_EN` | 15 | R/W | تفعيل Command Queue |
| `EXT_CSD_FLUSH_CACHE` | 32 | W | تفريغ الـ cache إلى الـ media |
| `EXT_CSD_CACHE_CTRL` | 33 | R/W | تفعيل/تعطيل الـ cache |
| `EXT_CSD_POWER_OFF_NOTIFICATION` | 34 | R/W | إشعار الكارت بالـ power off |
| `EXT_CSD_PART_CONFIG` | 179 | R/W | اختيار الـ partition النشطة |
| `EXT_CSD_BUS_WIDTH` | 183 | R/W | تحديد عرض الـ bus |
| `EXT_CSD_HS_TIMING` | 185 | R/W | تحديد timing mode (HS/HS200/HS400) |
| `EXT_CSD_REV` | 192 | RO | إصدار الـ EXT_CSD spec |
| `EXT_CSD_CARD_TYPE` | 196 | RO | الـ speed modes المدعومة |
| `EXT_CSD_SEC_CNT` | 212 | RO | عدد الـ sectors (4 bytes) |
| `EXT_CSD_BKOPS_EN` | 163 | R/W | تفعيل Background Operations |
| `EXT_CSD_RST_N_FUNCTION` | 162 | R/W | تفعيل RST_n pin |
| `EXT_CSD_RPMB_MULT` | 168 | RO | حجم الـ RPMB partition |
| `EXT_CSD_ERASE_GROUP_DEF` | 175 | R/W | استخدام HC erase group size |
| `EXT_CSD_HC_ERASE_GRP_SIZE` | 224 | RO | حجم الـ HC erase group |
| `EXT_CSD_CMDQ_DEPTH` | 307 | RO | عمق الـ command queue (max tasks) |
| `EXT_CSD_CMDQ_SUPPORT` | 308 | RO | دعم الـ CMDQ |
| `EXT_CSD_PRE_EOL_INFO` | 267 | RO | مؤشر اقتراب نهاية العمر |
| `EXT_CSD_DEVICE_LIFE_TIME_EST_TYP_A` | 268 | RO | تقدير عمر الـ MLC cells |
| `EXT_CSD_DEVICE_LIFE_TIME_EST_TYP_B` | 269 | RO | تقدير عمر الـ TLC/SLC cells |
| `EXT_CSD_FIRMWARE_VERSION` | 254 | RO | firmware version (8 bytes) |

---

### ### Group 6: Card Type Speed Modes

الـ `EXT_CSD_CARD_TYPE` (offset 196) بيحدد الـ speed modes اللي الكارت بيدعمها. الـ kernel بيـ read القيمة دي وبيعمل AND مع قدرات الـ host عشان يختار أعلى mode مدعوم.

```c
/* التسلسل الهرمي للـ speed modes من الأبطأ للأسرع */
EXT_CSD_CARD_TYPE_HS_26        // 26 MHz SDR
EXT_CSD_CARD_TYPE_HS_52        // 52 MHz SDR
EXT_CSD_CARD_TYPE_DDR_52       // 52 MHz DDR (1.8V أو 1.2V)
EXT_CSD_CARD_TYPE_HS200        // 200 MHz SDR (HS200)
EXT_CSD_CARD_TYPE_HS400        // 200 MHz DDR (HS400)
EXT_CSD_CARD_TYPE_HS400ES      // HS400 + Enhanced Strobe
```

**ASCII diagram — negotiation flow:**

```
Host capabilities
        |
        v
   AND with EXT_CSD_CARD_TYPE
        |
        v
   ┌────────────────────────────────┐
   │  HS400ES supported by both?   │──yes──> Select HS400ES
   └────────────────────────────────┘
        |no
        v
   ┌─────────────────────────────┐
   │  HS400 supported by both?  │──yes──> Select HS400
   └─────────────────────────────┘
        |no
        v
   ┌──────────────────────────────┐
   │  HS200 supported by both?  │──yes──> Select HS200
   └──────────────────────────────┘
        |no
        v
   ┌─────────────────────────────────┐
   │  DDR52 supported by both?   │──yes──> Select DDR52
   └─────────────────────────────────┘
        |no
        v
        HS52 or HS26
```

---

### ### Group 7: MMC_SWITCH Access Modes

الـ `MMC_SWITCH` (CMD6) هو أهم command في الـ eMMC management — بيغير أي byte في الـ EXT_CSD. الـ access mode بيتحدد في bits [25:24] من الـ argument.

| Macro | Value | المعنى |
|---|---|---|
| `MMC_SWITCH_MODE_CMD_SET` | 0x00 | تغيير الـ command set (deprecated عملياً) |
| `MMC_SWITCH_MODE_SET_BITS` | 0x01 | set الـ bits اللي قيمتها 1 في الـ value |
| `MMC_SWITCH_MODE_CLEAR_BITS` | 0x02 | clear الـ bits اللي قيمتها 1 في الـ value |
| `MMC_SWITCH_MODE_WRITE_BYTE` | 0x03 | كتابة الـ byte كامل بالقيمة المطلوبة |

**مثال — تفعيل الـ cache عبر CMD6:**
```c
/* Write 1 to EXT_CSD_CACHE_CTRL[0] */
mmc_switch(card,
           EXT_CSD_CMD_SET_NORMAL,
           EXT_CSD_CACHE_CTRL,
           1,                          /* value */
           card->ext_csd.generic_cmd6_time);
/*
 * mmc_switch internally builds CMD6 argument:
 * arg = (MMC_SWITCH_MODE_WRITE_BYTE << 24) |
 *       (EXT_CSD_CACHE_CTRL << 16)         |
 *       (1 << 8)                            |
 *       EXT_CSD_CMD_SET_NORMAL
 */
```

---

### ### Group 8: Erase/Trim/Discard Arguments

الـ eMMC بيدعم أنواع مختلفة من الـ erase operations، كلها بتتبعت كـ argument لـ CMD38:

| Macro | Value | الوظيفة |
|---|---|---|
| `MMC_ERASE_ARG` | `0x00000000` | Conventional erase — NAND erase |
| `MMC_TRIM_ARG` | `0x00000001` | Trim — mark blocks as unused (TRIM) |
| `MMC_DISCARD_ARG` | `0x00000003` | Discard — hint for GC, data may survive |
| `MMC_SECURE_ERASE_ARG` | `0x80000000` | Secure erase — cryptographic wipe |
| `MMC_SECURE_TRIM1_ARG` | `0x80000001` | Secure trim phase 1 |
| `MMC_SECURE_TRIM2_ARG` | `0x80008000` | Secure trim phase 2 |
| `MMC_SECURE_ARGS` | `0x80000000` | Mask للتحقق من الـ secure flag |
| `MMC_TRIM_OR_DISCARD_ARGS` | `0x00008003` | Mask للتمييز بين trim وdiscard |

**Key details:**
- الـ `MMC_SECURE_ARGS` mask بيتستخدم للتحقق لو الـ erase argument هو secure operation قبل السماح بيه
- الـ `SEC_FEATURE_SUPPORT` في `EXT_CSD` لازم يكون set قبل استخدام الـ secure variants
- `MMC_DISCARD_ARG` مختلف عن `TRIM` — الـ discard مش guaranteed يمسح الداتا

---

### ### Group 9: BKOPS و Command Queue Flags

**الـ Background Operations (BKOPS):**
الـ eMMC controller الداخلي بتاع الكارت بيعمل garbage collection وwear leveling في الخلفية. الـ kernel يقدر يتحكم فيها:

```c
EXT_CSD_BKOPS_SUPPORT   // 502: الكارت بيدعم BKOPS
EXT_CSD_BKOPS_EN        // 163: تفعيل manual BKOPS
EXT_CSD_AUTO_BKOPS_MASK // 0x02: تفعيل automatic BKOPS
EXT_CSD_BKOPS_START     // 164: إطلاق BKOPS manually (W)
EXT_CSD_BKOPS_STATUS    // 246: مستوى urgency (0-3)
EXT_CSD_BKOPS_LEVEL_2   // 0x2: مستوى critical — لازم يتعالج فوراً
```

**الـ Command Queue (CMDQ):**
ميزة eMMC 5.1 بتسمح بـ queue حتى 32 request في نفس الوقت:

```c
EXT_CSD_CMDQ_SUPPORT    // 308: الكارت بيدعم CMDQ
EXT_CSD_CMDQ_DEPTH      // 307: عمق الـ queue (max - 1)
EXT_CSD_CMDQ_DEPTH_MASK // GENMASK(4,0) — 5 bits للعمق
EXT_CSD_CMDQ_MODE_EN    // 15:  تفعيل/تعطيل CMDQ
EXT_CSD_CMDQ_MODE_ENABLED // BIT(0) — bit تفعيل الـ CMDQ
EXT_CSD_CMDQ_SUPPORTED  // BIT(0) — bit الدعم في EXT_CSD[308]
```

---

### ### Group 10: Power Management Constants

```c
EXT_CSD_NO_POWER_NOTIFICATION  // 0: مش بيبعت notification
EXT_CSD_POWER_ON               // 1: الكارت powered on ومتهيئ
EXT_CSD_POWER_OFF_SHORT        // 2: power off قصير (<= POWER_OFF_LONG_TIME)
EXT_CSD_POWER_OFF_LONG         // 3: power off طويل
```

الـ kernel بيكتب القيمة دي في `EXT_CSD_POWER_OFF_NOTIFICATION` قبل ما يقطع الطاقة عن الكارت عشان يديه وقت يعمل internal flush. لو الكارت شافه `EXT_CSD_POWER_OFF_LONG`، بيعمل full flush للـ write buffer وبيحفظ state.

---

### ### Group 11: SPI Mode Status Bits

لما الـ MMC/SD بيشتغل على SPI بدل native mode، الـ R1 response بيتغير لـ single byte بدل 32 bits، والـ R2 ممكن يتبعه:

| Macro | Bit | المعنى |
|---|---|---|
| `R1_SPI_IDLE` | bit 0 | الكارت في idle state |
| `R1_SPI_ERASE_RESET` | bit 1 | erase sequence interrupted |
| `R1_SPI_ILLEGAL_COMMAND` | bit 2 | command غير معروف |
| `R1_SPI_COM_CRC` | bit 3 | CRC error في الـ command |
| `R1_SPI_ERASE_SEQ` | bit 4 | خطأ في تسلسل الـ erase |
| `R1_SPI_ADDRESS` | bit 5 | address error |
| `R1_SPI_PARAMETER` | bit 6 | parameter error |
| `R2_SPI_CARD_LOCKED` | bit 8 | الكارت مقفول |
| `R2_SPI_WP_VIOLATION` | bit 13 | write protect violation |
| `R2_SPI_OUT_OF_RANGE` | bit 15 | address out of range / CSD overwrite |

**ملاحظة:** في SPI mode، bit 7 دايماً zero، والـ R2 byte يجي بعد R1 في response لـ `MMC_SEND_STATUS` بس.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. الـ debugfs entries

الـ MMC subsystem بيعرض معلومات كتير عن طريق debugfs، عادةً تحت `/sys/kernel/debug/mmc0/` (أو `mmc1`, `mmc2` إلخ حسب الـ host).

| المسار | المحتوى |
|--------|---------|
| `/sys/kernel/debug/mmc0/ios` | الـ I/O settings الحالية (clock, bus_width, timing, voltage) |
| `/sys/kernel/debug/mmc0/err_stats` | إحصائيات الأخطاء مقسّمة حسب نوعها (CMD timeout, CRC, ADMA…) |
| `/sys/kernel/debug/mmc0/ext_csd` | raw dump للـ EXT_CSD كامل (512 بايت) |
| `/sys/kernel/debug/mmc0/clock` | الـ clock الفعلي اللي بيستخدمه الـ host |
| `/sys/kernel/debug/mmc0/regs` | register dump للـ host controller (لو الـ driver دعمه) |

```bash
# اقرأ الـ ios الحالي
cat /sys/kernel/debug/mmc0/ios

# اقرأ err_stats لمعرفة نوع الأخطاء
cat /sys/kernel/debug/mmc0/err_stats

# dump كامل للـ EXT_CSD
cat /sys/kernel/debug/mmc0/ext_csd
```

**مثال output للـ ios:**
```
clock:          200000000 Hz
actual clock:   200000000 Hz
vdd:            21 (3.3 ~ 3.4 V)
bus mode:       2 (push-pull)
chip select:    0 (don't care)
power mode:     2 (on)
bus width:      3 (8 bits)
timing spec:    10 (mmc HS400)
signal voltage: 1 (1.80 V)
driver type:    0 (driver type B)
```

**تفسير الـ output:** الكارت شغال بـ HS400 على 200 MHz، 8-bit bus، 1.8V — ده الـ mode الأسرع لـ eMMC.

---

#### 2. الـ sysfs entries

| المسار | المحتوى |
|--------|---------|
| `/sys/bus/mmc/devices/mmc0:0001/` | directory الكارت الأول على host 0 |
| `/sys/bus/mmc/devices/mmc0:0001/cid` | الـ Card Identification Register |
| `/sys/bus/mmc/devices/mmc0:0001/csd` | الـ Card Specific Data |
| `/sys/bus/mmc/devices/mmc0:0001/date` | تاريخ التصنيع (من الـ CID) |
| `/sys/bus/mmc/devices/mmc0:0001/fwrev` | firmware revision |
| `/sys/bus/mmc/devices/mmc0:0001/hwrev` | hardware revision |
| `/sys/bus/mmc/devices/mmc0:0001/manfid` | manufacturer ID |
| `/sys/bus/mmc/devices/mmc0:0001/name` | اسم المنتج |
| `/sys/bus/mmc/devices/mmc0:0001/oemid` | OEM ID |
| `/sys/bus/mmc/devices/mmc0:0001/preferred_erase_size` | حجم الـ erase الأمثل |
| `/sys/bus/mmc/devices/mmc0:0001/serial` | الرقم التسلسلي |
| `/sys/bus/mmc/devices/mmc0:0001/type` | نوع الكارت (MMC/SD/SDIO) |
| `/sys/class/block/mmcblk0/queue/rotational` | يجب أن يكون 0 لـ eMMC |
| `/sys/class/block/mmcblk0/queue/discard_max_bytes` | الـ discard/TRIM support |

```bash
# اعرض معلومات الكارت
for f in /sys/bus/mmc/devices/mmc0:0001/*; do
    echo "$(basename $f): $(cat $f 2>/dev/null)";
done

# تحقق من الـ erase size
cat /sys/bus/mmc/devices/mmc0:0001/preferred_erase_size
```

---

#### 3. الـ ftrace — tracepoints وأحداث المراقبة

الـ MMC subsystem فيه tracepoints جاهزة تحت `mmc`:

```bash
# اعرض كل الـ tracepoints المتاحة
ls /sys/kernel/debug/tracing/events/mmc/

# أهم الـ events:
# mmc_request_start  - بداية كل request
# mmc_request_done   - نهاية كل request مع الـ error code
# mmc_cmd_rw_start   - بداية تنفيذ أمر
# mmc_cmd_rw_end     - نهاية تنفيذ أمر
# mmc_blk_rw_start   - block read/write start
# mmc_blk_rw_end     - block read/write end

# فعّل كل أحداث الـ MMC
echo 1 > /sys/kernel/debug/tracing/events/mmc/enable

# أو event محدد
echo 1 > /sys/kernel/debug/tracing/events/mmc/mmc_request_done/enable

# شغّل الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# اقرأ النتائج
cat /sys/kernel/debug/tracing/trace

# وقف الـ tracing
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo 0 > /sys/kernel/debug/tracing/events/mmc/enable
```

**مثال output:**
```
kworker/0:1H-282  [000] ....  123.456789: mmc_request_start: mmc0: start struct mmc_request[0x...]: cmd_opcode=25 cmd_arg=0x00200000 cmd_flags=0x95 cmd_retries=0 stop_opcode=12 sbc_opcode=23
kworker/0:1H-282  [000] ....  123.457890: mmc_request_done: mmc0: end struct mmc_request[0x...]: cmd_opcode=25 cmd_err=0 cmd_resp=0x900 stop_opcode=12 stop_err=0
```

**تفسير:** `cmd_opcode=25` ده `MMC_WRITE_MULTIPLE_BLOCK`، `cmd_err=0` يعني نجح، `cmd_resp=0x900` يعني الكارت في state TRAN (bits 11:9 = 4).

---

#### 4. الـ printk والـ dynamic debug

```bash
# فعّل الـ dynamic debug لكل الـ MMC core
echo 'module mmc_core +p' > /sys/kernel/debug/dynamic_debug/control

# أو لـ file محددة
echo 'file drivers/mmc/core/mmc.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل debug للـ block driver
echo 'module mmc_block +p' > /sys/kernel/debug/dynamic_debug/control

# وقف الـ debug
echo 'module mmc_core -p' > /sys/kernel/debug/dynamic_debug/control

# رفع الـ loglevel للـ kernel مؤقتاً (7 = DEBUG)
echo 7 > /proc/sys/kernel/printk

# أو عن طريق dmesg
dmesg -n 7
```

**لتتبع الـ EXT_CSD commands تحديداً:**
```bash
echo 'func mmc_send_ext_csd +pflmt' > /sys/kernel/debug/dynamic_debug/control
```

الـ flags: `p` = print، `f` = function name، `l` = line number، `m` = module، `t` = thread ID.

---

#### 5. الـ kernel config options للـ debugging

| الـ Config | الوصف |
|-----------|-------|
| `CONFIG_MMC_DEBUG` | يفعّل الـ debug messages في الـ MMC subsystem كلها |
| `CONFIG_MMC_UNSAFE_RESUME` | للـ testing فقط — استئناف بدون فحص الكارت |
| `CONFIG_FAIL_MMC_REQUEST` | fault injection لاختبار error handling |
| `CONFIG_DEBUG_FS` | لازم مفعّل عشان debugfs يشتغل |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل الـ pr_debug() و dev_dbg() |
| `CONFIG_TRACING` | لازم للـ ftrace events |
| `CONFIG_MMC_BLOCK_BOUNCE` | bounce buffers — disable لو بتشخص DMA مشاكل |
| `CONFIG_BLK_DEV_THROTTLING` | لو بتشخص مشاكل في الـ I/O throughput |

```bash
# تحقق من الـ config الحالي
grep -E 'CONFIG_MMC|CONFIG_FAIL_MMC' /boot/config-$(uname -r)

# أو من الـ kernel config المدمج
zcat /proc/config.gz | grep -E 'CONFIG_MMC'
```

---

#### 6. أدوات الـ subsystem المتخصصة

```bash
# mmc-utils — الأداة الأساسية لـ eMMC
# اقرأ الـ EXT_CSD كاملاً وفسّره
mmc extcsd read /dev/mmcblk0

# اعرض معلومات الكارت
mmc info /dev/mmcblk0

# قرأ الـ RPMB key
mmc rpmb read-counter /dev/mmcblk0rpmb

# فحص الـ BKOPS status
mmc bkops-en /dev/mmcblk0  # تفعيل manual BKOPS

# فحص حالة الـ write protection
mmc writeprotect boot get /dev/mmcblk0

# تفعيل الـ cache
mmc cache enable /dev/mmcblk0

# اقرأ الـ life time estimation
mmc extcsd read /dev/mmcblk0 | grep -A1 "Life Time"

# أداة mmcli (لو متوفرة)
# iostat لمراقبة الـ throughput
iostat -x mmcblk0 1

# blktrace للـ block-level tracing
blktrace -d /dev/mmcblk0 -o - | blkparse -i -
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---------------------|--------|------|
| `mmc0: error -110 whilst initialising MMC card` | timeout أثناء init — الكارت مش بيرد | تحقق من الكهرباء والـ clock، افحص hardware |
| `mmc0: CMD13 failed` | فشل قراءة الـ status register | غالباً مشكلة CRC أو signal integrity |
| `mmc0: Timeout waiting for hardware interrupt` | الـ host controller ما بعتش الـ interrupt | تحقق من الـ IRQ config والـ DMA |
| `mmc0: tuning execution failed` | فشل الـ tuning لـ HS200/HS400 | جرب تخفيض السرعة، افحص الـ impedance |
| `mmc0: Too large page read, retrying` | الـ page size أكبر من المتوقع | عادةً يُحل تلقائياً بـ retry |
| `mmc0: Card stuck in programming state!` | الكارت stuck في state PRG | power cycle للكارت، قد يكون damaged |
| `mmc0: Switch failed, status 0x...` | فشل `MMC_SWITCH` (CMD6) | افحص الـ EXT_CSD byte المستهدف، تحقق من timing |
| `mmc0: BKOPS handling failed` | فشل بدء الـ background operations | قد يكون الكارت مشغول، أعد المحاولة |
| `mmc0: cache flush error -5` | فشل flush الـ cache | I/O error، افحص الكارت بـ `fsck` |
| `mmc0: error -84 transferring data` | `EILSEQ` — خطأ CRC في البيانات | مشكلة signal integrity، افحص الـ bus |
| `mmc0: Reset called for DL error` | خطأ في data link | افحص الـ DMA config وحدود الذاكرة |
| `mmc0: ADMA error` | خطأ في الـ ADMA descriptor table | تحقق من الـ DMA addresses والـ alignment |
| `mmc0: req failed (CMD25): -110` | timeout في الـ write multiple blocks | بطيء جداً، زود الـ timeout أو افحص الكارت |
| `mmc0: card status error 0x...` | خطأ في الـ R1 response | فك bits الـ R1 وارجع للجدول أدناه |

**فك رموز الـ R1 status:**
```c
/* الـ bits المهمة في R1 response */
R1_OUT_OF_RANGE    = bit31  // الـ address خارج النطاق
R1_ADDRESS_ERROR   = bit30  // خطأ في الـ alignment
R1_BLOCK_LEN_ERROR = bit29  // حجم الـ block غلط
R1_COM_CRC_ERROR   = bit23  // CRC خاطئ في الـ command
R1_ILLEGAL_COMMAND = bit22  // الأمر مش مدعوم في الـ state الحالي
R1_CARD_ECC_FAILED = bit21  // فشل الـ ECC الداخلي
```

```bash
# فك أي R1 status value
STATUS=0x00900
echo "Current state: $(( ($STATUS & 0x1E00) >> 9 ))"
# 4 = TRAN (Transfer) — الـ state الطبيعي لعمليات القراءة/الكتابة
```

---

#### 8. نقاط استراتيجية لـ dump_stack() وـ WARN_ON()

```c
/* في drivers/mmc/core/mmc.c — بعد فشل MMC_SWITCH */
static int mmc_switch(struct mmc_card *card, u8 set, u8 index, u8 value,
                      unsigned int timeout_ms)
{
    int err = __mmc_switch(card, set, index, value, timeout_ms,
                           MMC_SEND_STATUS, true, true);
    /* نقطة مناسبة لـ WARN_ON */
    WARN_ON(err && err != -EBADMSG);
    return err;
}

/* في mmc_init_card — بعد قراءة EXT_CSD */
if (WARN_ON(card->ext_csd.rev < 3))
    return -EINVAL;  /* كارت قديم جداً */

/* تحقق من الـ state قبل أي command */
WARN_ON(R1_CURRENT_STATE(status) != R1_STATE_TRAN);

/* في الـ error path لـ tuning */
if (err) {
    dev_err(mmc_dev(host), "tuning failed: %d\n", err);
    dump_stack();  /* مهم لمعرفة من طلب الـ tuning */
}

/* فحص الـ clock range */
WARN_ON(host->ios.clock > host->f_max);
WARN_ON(host->ios.clock && host->ios.clock < host->f_min);
```

---

### Hardware Level

#### 1. التحقق من أن الـ hardware state مطابق للـ kernel state

```bash
# تحقق من الـ clock الفعلي
cat /sys/kernel/debug/mmc0/ios | grep "actual clock"

# تحقق من الـ bus width في الـ EXT_CSD (byte 183)
mmc extcsd read /dev/mmcblk0 | grep "BUS_WIDTH"
# يجب أن يطابق ما في /sys/kernel/debug/mmc0/ios

# تحقق من الـ timing mode
mmc extcsd read /dev/mmcblk0 | grep "HS_TIMING"
# 0=BC, 1=HS, 2=HS200, 3=HS400

# قارن الـ card type مع الـ timing المستخدم
mmc extcsd read /dev/mmcblk0 | grep "CARD_TYPE"
# Bit0=HS26, Bit1=HS52, Bit2=DDR52@1.8V, Bit4=HS200, Bit6=HS400
```

**جدول التطابق المطلوب:**

| الـ EXT_CSD HS_TIMING | الـ ios timing المتوقع | الـ bus width |
|----------------------|----------------------|--------------|
| 0 (BC) | `MMC_TIMING_LEGACY` | 1/4/8 bit |
| 1 (HS) | `MMC_TIMING_MMC_HS` | 4/8 bit |
| 2 (HS200) | `MMC_TIMING_MMC_HS200` | 4/8 bit |
| 3 (HS400) | `MMC_TIMING_MMC_HS400` | 8 bit فقط |

---

#### 2. تقنيات الـ Register Dump

```bash
# devmem2 — لقراءة registers الـ host controller مباشرة
# أولاً احصل على base address من device tree أو dmesg
dmesg | grep -i "sdhci\|mmc" | grep "mapped\|base"

# مثال لـ SDHCI controller على عنوان 0xFE340000
devmem2 0xFE340000 w   # Present State Register
devmem2 0xFE340004 w   # Host Control Register
devmem2 0xFE340024 w   # Normal Interrupt Status

# استخدام /dev/mem مع dd (للأنظمة القديمة)
dd if=/dev/mem bs=4 count=1 skip=$((0xFE340000/4)) 2>/dev/null | xxd

# io command (من package ioport)
io -4 -r 0xFE340000

# لو الـ driver عنده regmap يمكن القراءة عن طريق debugfs
ls /sys/kernel/debug/regmap/
cat /sys/kernel/debug/regmap/fe340000.mmc/registers
```

**الـ SDHCI registers الأهم للـ debugging:**

| Register | Offset | الأهمية |
|---------|--------|---------|
| Present State | 0x24 | هل الكارت موجود؟ هل البيانات جاهزة؟ |
| Normal Int Status | 0x30 | آخر interrupt حصل |
| Error Int Status | 0x32 | الأخطاء المكتشفة |
| Clock Control | 0x2C | الـ clock المضبوط |
| Capabilities | 0x40 | ما يدعمه الـ controller |

---

#### 3. Logic Analyzer / Oscilloscope Tips

**نقاط القياس على الـ eMMC:**

```
eMMC Pinout (BGA-153):
  CLK  → قس هنا التردد والـ duty cycle
  CMD  → قس الـ command/response timing
  DAT0-7 → قس عرض الـ bus والـ eye diagram
  RST_n  → تأكد أنها high أثناء التشغيل

ASCII: eMMC Bus Timing
         ___     ___     ___
CLK  ___|   |___|   |___|   |___
         ___________
CMD  ___|     CMD13    |________
                  _____________
DAT0 ____________| R1 response |__
```

```
نصائح للـ Logic Analyzer:
- Sample rate: 4× الـ clock على الأقل (HS400 = 200MHz → 800MHz sample rate)
- HS200: DDR على الـ CMD، SDR على الـ DATA
- HS400: DDR على الـ DATA أيضاً، استخدم الـ strobe (DS) للـ sampling
- فعّل الـ protocol decoder لـ eMMC/SD لو متوفر
- ابحث عن CRC errors في نهاية كل data block (16-bit CRC per data line)
```

**Oscilloscope:**
```bash
# ما تقيسه:
# 1. Rise/Fall time على CLK — يجب < 20% من الـ clock period
# 2. Signal voltage levels — 1.8V ±5% لـ HS200/HS400
# 3. Vref stability — ripple < 50mV
# 4. Power supply noise أثناء write operations
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في الـ kernel log

| المشكلة | Pattern في dmesg | السبب الغالب |
|--------|-----------------|-------------|
| Voltage switching failure | `mmc0: Signal voltage switch failed` | مشكلة في الـ PMIC أو LDO |
| Clock instability | `mmc0: tuning execution failed` متكرر | الـ PCB trace طويل أو غير مطابق |
| Data corruption | `mmc0: error -84` + `Buffer I/O error` | CRC error — افحص الـ eye diagram |
| Card not detected | `mmc0: no card present` | GPIO مش متصل، أو الـ card detect pin |
| Power issue | `mmc0: error -110 initialising` | الـ Vcc أو Vccq غير مستقر |
| eMMC worn out | `mmc0: error -5 transferring data` | الـ NAND blocks كلها bad |
| Bus contention | `mmc0: Got data interrupt 0x02000000` | مشكلة في الـ pull-up resistors |

```bash
# تحقق من الأخطاء المتكررة
dmesg | grep -c "mmc0: error"

# اعرض pattern الأخطاء مع timestamp
dmesg -T | grep "mmc0: error" | head -20

# تحقق من الـ BKOPS status (مهم لصحة الكارت)
mmc extcsd read /dev/mmcblk0 | grep -A2 "BKOPS_STATUS"
# 0=No ops needed, 1=Non-critical, 2=Performance impacted, 3=Critical!
```

---

#### 5. الـ Device Tree Debugging

```bash
# اعرض الـ DT node للـ MMC
cat /proc/device-tree/mmc@fe320000/compatible
# يجب أن يطابق الـ driver

# اعرض كل properties للـ MMC node
ls /proc/device-tree/mmc@fe320000/

# تحقق من الـ clock references
cat /proc/device-tree/mmc@fe320000/clocks | xxd

# تحقق من bus-width في الـ DT
cat /proc/device-tree/mmc@fe320000/bus-width | xxd
# يجب أن يكون 8 لـ eMMC

# اعرض الـ DT من dtc
dtc -I fs /proc/device-tree 2>/dev/null | grep -A 30 "mmc@"
```

**الـ properties المهمة في الـ DT للـ eMMC:**

```dts
/* مثال DT node لـ eMMC */
&mmc0 {
    compatible = "vendor,mmc-v5";
    reg = <0xFE340000 0x1000>;
    clocks = <&clk_mmc>;
    bus-width = <8>;            /* لازم 8 لـ HS400 */
    max-frequency = <200000000>; /* 200MHz لـ HS400 */
    non-removable;              /* eMMC مش بتُشال */
    mmc-hs400-1_8v;            /* دعم HS400 على 1.8V */
    mmc-hs200-1_8v;            /* دعم HS200 على 1.8V */
    mmc-ddr-1_8v;              /* دعم DDR52 على 1.8V */
    no-sdio;                   /* disable SDIO detection */
    no-sd;                     /* disable SD card detection */
    cap-mmc-highspeed;         /* دعم HS52 */
    vmmc-supply = <&vcc_emmc>; /* تحقق أن الـ regulator صح */
    vqmmc-supply = <&vcc1v8>;  /* Vccq = 1.8V */
};
```

```bash
# تحقق من أن الـ DT compatible موجود في الـ driver
grep -r "vendor,mmc-v5" /sys/bus/platform/drivers/

# تحقق من الـ regulator
cat /sys/class/regulator/regulator.X/voltage
# يجب أن يكون 3300000 لـ Vcc و 1800000 لـ Vccq في HS200/HS400

# تحقق من الـ pinctrl state
cat /sys/kernel/debug/pinctrl/pinctrl-handles
```

---

### Practical Commands

#### جاهز للنسخ — تشخيص شامل

```bash
#!/bin/bash
# eMMC Full Debug Script

DEV=/dev/mmcblk0
HOST=mmc0
DEBUGFS=/sys/kernel/debug

echo "=== MMC Host IOS ==="
cat $DEBUGFS/$HOST/ios

echo "=== Error Statistics ==="
cat $DEBUGFS/$HOST/err_stats

echo "=== Card Info ==="
for f in /sys/bus/mmc/devices/$HOST:0001/{manfid,name,serial,date,fwrev,type}; do
    echo "$(basename $f): $(cat $f 2>/dev/null)"
done

echo "=== EXT_CSD Key Fields ==="
mmc extcsd read $DEV 2>/dev/null | grep -E \
    "LIFE_TIME|EOL|BKOPS|CACHE|HS_TIMING|BUS_WIDTH|CMDQ|FIRMWARE"

echo "=== Block Device Queue ==="
cat /sys/class/block/mmcblk0/queue/rotational
cat /sys/class/block/mmcblk0/queue/discard_max_bytes

echo "=== Recent Errors ==="
dmesg | grep -i "mmc\|mmcblk" | tail -20
```

```bash
# تفعيل الـ full MMC tracing لثانية واحدة
echo 1 > /sys/kernel/debug/tracing/events/mmc/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
sleep 1
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace | grep -E "cmd_opcode|cmd_err" | head -30
echo 0 > /sys/kernel/debug/tracing/events/mmc/enable
```

```bash
# اختبار الـ read/write throughput
# Read test
dd if=/dev/mmcblk0 of=/dev/null bs=1M count=512 iflag=direct 2>&1 | tail -1

# Write test (خطير — هيمسح البيانات)
# dd if=/dev/zero of=/dev/mmcblk0 bs=1M count=512 oflag=direct 2>&1 | tail -1

# باستخدام fio
fio --name=read_test --filename=/dev/mmcblk0 --rw=read \
    --bs=512k --numjobs=1 --size=512m --direct=1 --group_reporting
```

```bash
# فحص الـ CRC errors بشكل مستمر
watch -n 1 'cat /sys/kernel/debug/mmc0/err_stats | grep -E "CRC|Timeout"'
```

```bash
# تفسير الـ R1 status response
decode_r1() {
    local status=$1
    echo "R1 Status: 0x$(printf '%08X' $status)"
    [ $((status & (1<<31))) -ne 0 ] && echo "  ERROR: OUT_OF_RANGE"
    [ $((status & (1<<30))) -ne 0 ] && echo "  ERROR: ADDRESS_ERROR"
    [ $((status & (1<<29))) -ne 0 ] && echo "  ERROR: BLOCK_LEN_ERROR"
    [ $((status & (1<<23))) -ne 0 ] && echo "  ERROR: COM_CRC_ERROR"
    [ $((status & (1<<22))) -ne 0 ] && echo "  ERROR: ILLEGAL_COMMAND"
    [ $((status & (1<<21))) -ne 0 ] && echo "  ERROR: CARD_ECC_FAILED"
    [ $((status & (1<<20))) -ne 0 ] && echo "  ERROR: CC_ERROR"
    [ $((status & (1<<19))) -ne 0 ] && echo "  ERROR: GENERAL_ERROR"
    local state=$(( (status & 0x1E00) >> 9 ))
    local states=("IDLE" "READY" "IDENT" "STBY" "TRAN" "DATA" "RCV" "PRG" "DIS")
    echo "  State: ${states[$state]}"
    [ $((status & (1<<8))) -ne 0 ] && echo "  READY_FOR_DATA: yes"
    [ $((status & (1<<7))) -ne 0 ] && echo "  WARNING: SWITCH_ERROR"
}

# استخدام
decode_r1 0x00000900  # TRAN state, ready for data
```

```bash
# تحقق من life time estimation
mmc extcsd read /dev/mmcblk0 | grep -E "PRE_EOL|LIFE_TIME"
# PRE_EOL_INFO:
#   0x00: Not defined
#   0x01: Normal (< 80% life used)
#   0x02: Warning (80-90% life used)
#   0x03: Urgent (> 90% life used — استبدل الكارت!)
#
# DEVICE_LIFE_TIME_EST_TYP_A/B:
#   0x00: Not defined
#   0x01: 0-10% used
#   0x02: 10-20% used
#   ...
#   0x0A: 90-100% used
#   0x0B: exceeded maximum estimated life time!
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: eMMC مش بيبوت على RK3562 — Industrial Gateway

#### العنوان
**eMMC KIOXIA HS400** بيفشل في initialization بسبب voltage mismatch على industrial gateway

#### السياق
شركة بتعمل industrial gateway مبني على **RK3562**، الـ eMMC هو KIOXIA 32GB HS400ES. البورد بتبوت من SPI NOR في المصنع، لكن بعد نقل الـ rootfs على الـ eMMC، الجهاز بيقف والـ kernel log بيطبع:

```
mmc0: error -110 whilst initialising MMC card
mmc0: CMD6 failed with status 0x00000080
```

#### المشكلة
الـ status `0x00000080` معناه:

```c
#define R1_SWITCH_ERROR  (1 << 7)  /* sx, c */
```

الـ driver بيبعت `MMC_SWITCH` (CMD6) عشان يدخل HS400، لكن الكارت بيرد بـ `R1_SWITCH_ERROR` — يعني قبل الـ command لكن فشل في التنفيذ.

#### التحليل
الـ MMC_SWITCH argument format موثّق في الملف:

```c
/*
 * MMC_SWITCH argument format:
 *
 *  [31:26] Always 0
 *  [25:24] Access Mode
 *  [23:16] Location of target Byte in EXT_CSD
 *  [15:08] Value Byte
 *  [07:03] Always 0
 *  [02:00] Command Set
 */
```

الـ driver بيكتب `EXT_CSD_TIMING_HS400` على byte `EXT_CSD_HS_TIMING` (185):

```c
#define EXT_CSD_HS_TIMING      185  /* R/W */
#define EXT_CSD_TIMING_HS400   3    /* HS400 */
```

لكن الكارت بيرفض لأن `EXT_CSD_CARD_TYPE` (byte 196) بيقول:

```c
#define EXT_CSD_CARD_TYPE_HS400_1_8V  (1<<6)  /* 200MHz DDR at 1.8V */
#define EXT_CSD_CARD_TYPE_HS400_1_2V  (1<<7)  /* 200MHz DDR at 1.2V */
```

الكارت يدعم HS400 فقط عند 1.8V، لكن الـ schematic فيه VCCQ موصّل على 3.3V LDO ثابت بدل switchable regulator.

الكارت بيقرأ الـ R1 response ويلاقي الـ state:

```c
#define R1_CURRENT_STATE(x)  ((x & 0x00001E00) >> 9)
#define R1_STATE_TRAN  4
```

الكارت في TRAN state ورافض HS400 — hardware issue مش software bug.

#### الحل

**أولاً: تأكيد بالـ debugfs**

```bash
# اقرأ EXT_CSD كامل
mmc extcsd read /dev/mmcblk0 | grep -E "CARD_TYPE|HS_TIMING"
# EXT_CSD[196] CARD_TYPE: 0xC7 يعني HS400 @ 1.8V فقط
```

**ثانياً: تعديل الـ Device Tree**

```dts
/* rk3562-gateway.dts */
&sdhci {
    bus-width = <8>;
    mmc-hs200-1_8v;         /* HS200 كافي وأكثر أماناً */
    /* حذف mmc-hs400-1_8v */
    non-removable;
    vmmc-supply  = <&vcc_3v3>;
    vqmmc-supply = <&vcc_1v8>;  /* لازم 1.8V للـ HS200 */
};
```

**ثالثاً: fallback لـ HS52 DDR**

لو الـ 1.8V LDO مش متاح أصلاً، الـ DDR52 يشتغل على 3.3V:

```dts
mmc-ddr-1_8v;  /* حذف وخليه على 3.3V DDR52 */
```

```c
#define EXT_CSD_CARD_TYPE_DDR_1_8V  (1<<2)
#define EXT_CSD_CARD_TYPE_DDR_52    (EXT_CSD_CARD_TYPE_DDR_1_8V | \
                                     EXT_CSD_CARD_TYPE_DDR_1_2V)
```

#### الدرس المستفاد
الـ `EXT_CSD_CARD_TYPE` defines في `mmc.h` بيحددوا القدرات بالـ voltage class. الـ `R1_SWITCH_ERROR` مش bug في الـ driver — الكارت بيقول صراحة "hardware مش متوافق". لازم تتحقق من الـ voltage rails قبل تفعيل أي speed mode.

---

### السيناريو 2: SDIO WiFi على AM62x بيختفي بعد Suspend/Resume

#### العنوان
**RTL8189FS** SDIO WiFi module على IoT sensor hub بيختفي بعد كل power-save cycle

#### السياق
IoT sensor hub مبني على **TI AM62x**، الـ WiFi module هو RTL8189FS موصّل على SDIO slot. الجهاز بيشتغل كويس من الأول، لكن بعد أول suspend/resume cycle، الـ WiFi مش بيشتغل:

```
mmc1: error -84 during resume
sdio: SDIO card mmc1:0001 removed
rtl8189fs: probe failed after resume
```

#### المشكلة
الـ error `-EIO` مع الـ status:

```c
#define R1_CC_ERROR  (1 << 20)  /* erx, c — Controller Communication Error */
```

الكارت مش بيرد على أي command بعد resume.

#### التحليل
الـ initialization sequence للـ SDIO بعد resume بتبدأ بـ:

```c
/* Step 1: reset الكارت */
#define MMC_GO_IDLE_STATE  0   /* bc — broadcast, no response expected */

/* Step 2: بعت CMD1 وانتظر OCR */
#define MMC_SEND_OP_COND   1   /* bcr [31:0] OCR  R3 */
#define MMC_CARD_BUSY      0x80000000  /* bit 31 — card still powering up */
```

الـ driver بيلوب على CMD1 لحد ما `MMC_CARD_BUSY` يتclear:

```c
/* داخل mmc_send_op_cond */
while (ocr & MMC_CARD_BUSY) {
    mdelay(10);
    /* retry CMD1 */
}
```

المشكلة: الـ AM62x PM driver بيقطع VDDIO في suspend ثم بيرجعه في resume، لكن الـ RTL8189FS محتاج 50ms على الأقل قبل ما يقبل CMD0. الـ driver بيبعت CMD0 قبل انتهاء الـ power-up sequence فالكارت مش بيرد وبييجي timeout.

بعد الـ `mmc_op_tuning()` check:

```c
static inline bool mmc_op_tuning(u32 opcode)
{
    return opcode == MMC_SEND_TUNING_BLOCK ||
           opcode == MMC_SEND_TUNING_BLOCK_HS200;
}
```

اتضح إن tuning operation كانت شغالة لحظة الـ suspend interrupt، فالـ module اتقطع في وسط sequence.

#### الحل

**في الـ Device Tree:**

```dts
&sdhci1 {
    /* SDIO WiFi slot */
    bus-width = <4>;
    cap-sd-highspeed;
    keep-power-in-suspend;         /* لا تقطع VDDIO في suspend */
    wakeup-source;
    vmmc-supply = <&wifi_3v3_reg>;
    post-power-on-delay-ms = <80>; /* انتظر 80ms بعد power-on */
};
```

**التأكد من الـ resume state:**

```bash
# شوف الـ MMC state machine بعد resume
cat /sys/bus/mmc/devices/mmc1:0001/state

# تابع الـ events
echo 1 > /sys/kernel/debug/tracing/events/mmc/enable
cat /sys/kernel/debug/tracing/trace | grep -E "CMD0|CMD1|BUSY"
```

**منع الـ interrupt أثناء tuning:**

```c
/* في الـ PM suspend callback */
if (mmc_op_tuning(host->mrq->cmd->opcode)) {
    /* انتظر انتهاء الـ tuning قبل suspend */
    mmc_wait_for_req_done(host, host->mrq);
}
```

#### الدرس المستفاد
الـ `MMC_CARD_BUSY` flag والـ `mmc_op_tuning()` inline function هم barriers مهمة في الـ PM path. أي SDIO module محتاج `post-power-on-delay-ms` كافي في الـ DT، ولازم تمنع الـ suspend وسط tuning sequence.

---

### السيناريو 3: RPMB على i.MX8 — Automotive ECU Rollback Counter Corruption

#### العنوان
**RPMB write counter** بيتتلف بعد firmware update على automotive ECU بيستخدم secure boot

#### السياق
Automotive ECU لنظام إدارة البطارية مبني على **NXP i.MX8M Plus**، eMMC 5.1 Micron automotive-grade. الـ OP-TEE بيستخدم RPMB لتخزين rollback counters. بعد OTA firmware update، الـ TEE فشل في تحديث الـ counter:

```
RPMB: write request failed, status=0x0002
RPMB: authentication failure
tee-supplicant: ioctl RPMB_WRITE_DATA returned -EPERM
```

#### المشكلة
الـ RPMB write counter اتضرر بسبب hard reset في منتصف write sequence.

#### التحليل
الوصول لـ RPMB partition بيتم عبر `MMC_SWITCH` على `EXT_CSD_PART_CONFIG`:

```c
#define EXT_CSD_PART_CONFIG          179  /* R/W */
#define EXT_CSD_PART_CONFIG_ACC_RPMB (0x3)
#define EXT_CSD_PART_CONFIG_ACC_MASK (0x7)
```

الـ access mode بيتغير بـ:

```c
#define MMC_SWITCH              6    /* ac [31:0] R1b */
#define MMC_SWITCH_MODE_WRITE_BYTE  0x03  /* Set target to value */
```

الـ RPMB write sequence:

```
CMD23 (SET_BLOCK_COUNT) + Reliable Write bit
    ↓
CMD25 (WRITE_MULTIPLE_BLOCK) — الـ RPMB frame مع MAC
    ↓
CMD13 (SEND_STATUS) — انتظر PRG → TRAN
```

```c
#define MMC_SET_BLOCK_COUNT       23
#define MMC_WRITE_MULTIPLE_BLOCK  25
#define MMC_SEND_STATUS           13
```

الـ R1 state check:

```c
#define R1_CURRENT_STATE(x)  ((x & 0x00001E00) >> 9)
#define R1_STATE_PRG   7   /* card is programming */
#define R1_STATE_TRAN  4   /* ready for next command */
```

الـ firmware update script عمل hardware reset والكارت كان في `R1_STATE_PRG`. ده أدى لـ partial RPMB write — counter اتزاد لكن الـ MAC مش محسوب صح.

الـ `EXT_CSD_WR_REL_PARAM` بيتحقق منه الـ driver:

```c
#define EXT_CSD_WR_REL_PARAM      166  /* RO */
#define EXT_CSD_WR_REL_PARAM_EN   (1<<2)
```

الكارت بيدعم reliable write — المشكلة في الـ reset timing مش في الـ card capability.

#### الحل

**أولاً: تشخيص**

```bash
# اقرأ الـ write counter الحالي
mmc rpmb read-counter /dev/mmcblk0rpmb

# حاول قراءة بالـ key القديم
mmc rpmb read-block /dev/mmcblk0rpmb 0 1 /tmp/out.bin <key_file>
```

**ثانياً: في الـ firmware update path، أرسل Power Off Notification أولاً**

```c
/* من mmc.h */
#define EXT_CSD_POWER_OFF_NOTIFICATION  34  /* R/W */
#define EXT_CSD_POWER_OFF_SHORT         2
#define EXT_CSD_POWER_OFF_LONG          3

/* قبل أي reset */
mmc_switch(card, EXT_CSD_CMD_SET_NORMAL,
           EXT_CSD_POWER_OFF_NOTIFICATION,
           EXT_CSD_POWER_OFF_SHORT,
           card->ext_csd.power_off_long_time);
```

**ثالثاً: بعد الـ notification، انتظر TRAN state**

```c
static inline bool mmc_ready_for_data(u32 status)
{
    return status & R1_READY_FOR_DATA &&
           R1_CURRENT_STATE(status) == R1_STATE_TRAN;
}
/* ابعت CMD13 في loop لحد ما mmc_ready_for_data() ترجع true */
```

**رابعاً: ارجع للـ user partition بعد RPMB**

```c
/* مهم جداً بعد أي RPMB access */
mmc_switch(card, EXT_CSD_CMD_SET_NORMAL,
           EXT_CSD_PART_CONFIG,
           0x0,  /* back to user partition */
           card->ext_csd.part_time);
```

#### الدرس المستفاد
الـ RPMB مش بيسمح بـ partial write. `EXT_CSD_POWER_OFF_NOTIFICATION` في `mmc.h` موجود لحماية من بالظبط هالسيناريو. كل firmware update path لازم يبعت هاد الـ notification وينتظر `R1_STATE_TRAN` قبل أي reset.

---

### السيناريو 4: SD Card على Allwinner H616 — Android TV Box، الكارت بيتعرف بحجم غلط

#### العنوان
SD card 32GB بيتعرف كـ 2GB على **Allwinner H616** بسبب خلل في CSD structure field

#### السياق
Android TV Box مبني على **Allwinner H616**. المصنع غيّر مورّد الـ SD card من SanDisk لـ Kingston في الشحنة الجديدة. الكروت الجديدة بتتعرف بـ 2GB بس:

```bash
$ lsblk
NAME       SIZE
mmcblk1      2G   # المفروض 32G
```

#### المشكلة
الـ kernel بيحسب الحجم غلط من الـ CSD register.

#### التحليل
الـ CSD parsing في الـ kernel بيعتمد على `CSD_STRUCTURE` field:

```c
#define CSD_STRUCT_VER_1_0  0   /* spec 1.0 - 1.2 — formula قديمة */
#define CSD_STRUCT_VER_1_1  1   /* spec 1.4 - 2.2 */
#define CSD_STRUCT_VER_1_2  2   /* spec 3.1 - 4.1 — EXT_CSD */
#define CSD_STRUCT_EXT_CSD  3   /* Version coded in EXT_CSD */
```

للـ SD cards:
- CSD v1 → حجم أقصاه 2GB، المعادلة: `(C_SIZE+1) × 2^(C_SIZE_MULT+2) × 2^READ_BL_LEN`
- CSD v2 → SDHC/SDXC، المعادلة: `(C_SIZE+1) × 512KB`

الكارت من Kingston بيرجع `CSD_STRUCTURE = 0` بدل `1` (firmware bug في المورّد). الكارت فعلاً 32GB لكن CSD بيقول v1 فالـ parser بيحسب 2GB.

يمكن التأكد عبر `EXT_CSD_SEC_CNT`:

```c
#define EXT_CSD_SEC_CNT  212  /* RO, 4 bytes — الحجم الحقيقي */
/* size_bytes = SEC_CNT * 512 */
```

الـ `CCC_BLOCK_READ` و`CCC_BLOCK_WRITE` في الـ CSD موجودين:

```c
#define CCC_BLOCK_READ   (1<<2)  /* CMD16,17,18 — support confirmed */
#define CCC_BLOCK_WRITE  (1<<4)  /* CMD16,24,25 */
```

وبيأكدوا إن الكارت supports standard block operations — المشكلة فقط في حساب الحجم.

#### الحل

**أولاً: اقرأ الـ CSD raw**

```bash
cat /sys/bus/mmc/devices/mmc1:0001/csd
# الـ bits [127:126] = CSD_STRUCTURE
# لو 00 → v1 مش صح لكارت 32GB
```

**ثانياً: تأكيد الحجم الحقيقي**

```bash
mmc extcsd read /dev/mmcblk1 | grep SEC_COUNT
# SEC_COUNT * 512 = الحجم بالبايت
# 62333952 * 512 = ~32GB
```

**ثالثاً: quirk للكارت المعيوب**

```c
/* في drivers/mmc/core/quirks.h */
{
    .name  = "Kingston SC32G",
    .manfid = 0x0041,
    .quirk = MMC_QUIRK_BROKEN_HIGHRES_SEC_COUNT,
    /* أو استخدام SEC_CNT من EXT_CSD مباشرة بدل CSD */
},
```

```bash
# force re-read للحجم من EXT_CSD
echo 1 > /sys/bus/mmc/devices/mmc1:0001/preferred_erase_size
```

#### الدرس المستفاد
الـ `CSD_STRUCT_VER_*` defines في `mmc.h` هم branching points في الـ size parser. الـ `EXT_CSD_SEC_CNT` هو المصدر الأكثر موثوقية للحجم الفعلي. لما تغيّر مورّد الـ SD، دايماً تحقق من `SEC_CNT` يدوياً قبل الإنتاج.

---

### السيناريو 5: eMMC على STM32MP1 — Data Logger بيوصل لـ EOL بسرعة غير طبيعية

#### العنوان
Industrial data logger على **STM32MP1** بيستنفذ عمر الـ eMMC في 6 أشهر بدل 5 سنوات بسبب خطأ في استخدام erase commands

#### السياق
data logger صناعي مبني على **STM32MP157** مع eMMC 8GB. الجهاز بيكتب sensor readings كل 5 ثواني 24/7. بعد 6 أشهر:

```
mmc0: error -5 (EIO) writing sector 0x1a4000
mmc0: CMD38 ERASE failed, R1 status 0x08000000
```

الـ status bit:

```c
#define R1_ERASE_PARAM  (1 << 27)  /* ex, c — invalid erase parameters */
```

#### المشكلة
الـ flash wear استنفذ بشكل مسرّع جداً.

#### التحليل
الـ application كانت تعمل **full erase** بعد كل write cycle "لتأمين البيانات":

```c
/* الكود القديم الغلط */
#define MMC_ERASE_GROUP_START  35  /* CMD35 */
#define MMC_ERASE_GROUP_END    36  /* CMD36 */
#define MMC_ERASE              38  /* CMD38 */

/* argument غلط */
cmd.arg = MMC_ERASE_ARG;  /* 0x00000000 — full physical erase */
```

```c
/* من mmc.h */
#define MMC_ERASE_ARG         0x00000000  /* Standard erase — P/E cycle كامل */
#define MMC_TRIM_ARG          0x00000001  /* Trim — يخلي FTL يقرر */
#define MMC_DISCARD_ARG       0x00000003  /* Discard — hint بس مش ضمان */
#define MMC_SECURE_ERASE_ARG  0x80000000  /* Secure erase — أبطأ وأكثر استهلاكاً */
```

الـ full erase (`MMC_ERASE_ARG`) بيعمل physical erase على الـ erase group كامل (~512KB) حتى لو كتبت sector واحد (512B). الـ write amplification = 512KB / 512B = **1024x**.

الأصح كان `MMC_TRIM_ARG` الذي يخبر الـ FTL (Firmware Translation Layer) بالـ blocks المحذوفة فيعمل wear leveling ذكي.

التحقق من دعم TRIM:

```c
#define EXT_CSD_SEC_FEATURE_SUPPORT  231  /* RO */
#define EXT_CSD_SEC_GB_CL_EN         BIT(4)  /* TRIM supported */
#define EXT_CSD_TRIM_MULT            232  /* RO — timeout multiplier */
```

الكارت كان يدعم TRIM لكن الكود ما استخدمه. إضافة لكده، الـ BKOPS كان معطّل:

```c
#define EXT_CSD_BKOPS_EN         163  /* R/W */
#define EXT_CSD_BKOPS_SUPPORT    502  /* RO — does card support it? */
#define EXT_CSD_AUTO_BKOPS_MASK  0x02 /* Auto BKOPS bit */
```

الـ BKOPS المعطّل معناه إن الـ eMMC مش بيعمل background garbage collection في الـ idle time.

كمان كان ممكن ينتبه مبكراً لو كان بيراقب:

```c
#define EXT_CSD_PRE_EOL_INFO                267  /* RO */
#define EXT_CSD_DEVICE_LIFE_TIME_EST_TYP_A  268  /* RO — SLC blocks */
#define EXT_CSD_DEVICE_LIFE_TIME_EST_TYP_B  269  /* RO — MLC blocks */
```

#### الحل

**أولاً: تغيير الـ erase argument**

```c
/* بدل MMC_ERASE_ARG استخدم TRIM */
cmd.opcode = MMC_ERASE_GROUP_START;
cmd.arg    = start_lba;
/* ... CMD36 ... */
cmd.opcode = MMC_ERASE;
cmd.arg    = MMC_TRIM_ARG;  /* 0x00000001 */
```

**ثانياً: تفعيل Auto BKOPS**

```bash
# تفعيل Auto BKOPS
mmc bkops-enable /dev/mmcblk0

# التحقق
mmc extcsd read /dev/mmcblk0 | grep -E "BKOPS_EN|BKOPS_SUPPORT"
```

**ثالثاً: مراقبة health دورية (cron job)**

```bash
#!/bin/bash
# /etc/cron.weekly/check-emmc-health
EXTCSD=$(mmc extcsd read /dev/mmcblk0)

PRE_EOL=$(echo "$EXTCSD" | grep PRE_EOL_INFO | awk '{print $NF}')
# 0x01 = Normal | 0x02 = Warning | 0x03 = Urgent

LIFE_A=$(echo "$EXTCSD" | grep DEVICE_LIFE_TIME_EST_TYP_A | awk '{print $NF}')
LIFE_B=$(echo "$EXTCSD" | grep DEVICE_LIFE_TIME_EST_TYP_B | awk '{print $NF}')

logger "eMMC health: PRE_EOL=$PRE_EOL LIFE_A=$LIFE_A LIFE_B=$LIFE_B"

if [ "$PRE_EOL" -ge "2" ]; then
    logger "WARNING: eMMC approaching end of life!"
fi
```

**رابعاً: إضافة fstrim دوري**

```bash
# systemd timer لـ fstrim
systemctl enable fstrim.timer
# أو يدوياً
fstrim -v /data
```

#### الدرس المستفاد
`MMC_ERASE_ARG` vs `MMC_TRIM_ARG` vs `MMC_DISCARD_ARG` في `mmc.h` ليسوا synonyms. الـ full erase بيقتل الـ flash في أقل من سنة في الأنظمة الكاتبة. في أي data logger أو industrial system، لازم تستخدم TRIM، تفعّل Auto BKOPS، وتراقب `EXT_CSD_PRE_EOL_INFO` كـ health indicator — ده موجود في `mmc.h` من كتير بس ناس مش بتستخدمه.
## Phase 7: مصادر ومراجع

### توثيق Kernel الرسمي

الـ Linux kernel نفسه فيه توثيق مباشر للـ MMC subsystem تحت `Documentation/`:

| المسار | المحتوى |
|--------|---------|
| `Documentation/driver-api/mmc/index.rst` | نقطة البداية للـ MMC/SD/SDIO driver API |
| `Documentation/driver-api/mmc/mmc-tools.rst` | شرح أدوات `mmc-utils` |
| `Documentation/driver-api/mmc/mmc-async-req.rst` | الـ asynchronous request framework |
| `Documentation/driver-api/mmc/mmc-dev-attrs.rst` | الـ sysfs attributes الخاصة بالكارت |
| `Documentation/driver-api/mmc/mmc-dev-parts.rst` | الـ partitions الخاصة بـ eMMC |

الـ header file المدروسة نفسها:

```
include/linux/mmc/mmc.h   — MMC commands, EXT_CSD fields, R1 status bits
include/linux/mmc/host.h  — host controller interface
include/linux/mmc/card.h  — card descriptor struct
include/linux/mmc/sdio.h  — SDIO-specific commands
```

الـ source الرئيسي للـ core:

```
drivers/mmc/core/mmc.c       — MMC card init & state machine
drivers/mmc/core/core.c      — request handling, bus ops
drivers/mmc/core/mmc_ops.c   — low-level MMC command wrappers
drivers/mmc/core/block.c     — block device interface
```

---

### مقالات LWN.net

الـ **LWN.net** هو المرجع الأول لمتابعة تطور الـ kernel — الروابط دي كلها حقيقية من نتائج البحث:

#### تطور الـ subsystem

- **[new driver: multimedia card (mmc) framework — 2002](https://lwn.net/Articles/7176/)**
  أول patch قدّم الـ MMC framework في الـ kernel — النواة الأولى للـ subsystem.

- **[1/4 MMC layer — 2004](https://lwn.net/Articles/82765/)**
  patch series أضاف الـ core MMC layer لدعم الـ embedded controllers زي Intel PXA وARM MMCI.

- **[Secure Digital (SD) support — 2005](https://lwn.net/Articles/126098/)**
  أول دعم لـ SD cards — كشف الفرق بين MMC وSD، قراءة الـ registers الإضافية، ودعم الـ 4-bit mode.

- **[MMC updates — 2008](https://lwn.net/Articles/253888/)**
  تحديثات شاملة أضافت SDIO وSPI support وأزالت الـ MMC-specific error codes.

- **[mmc: Add SD4.0 support — 2015](https://lwn.net/Articles/642336/)**
  إضافة دعم SD 4.0 للـ MMC subsystem.

#### ميزات متقدمة

- **[Add MMC erase and secure erase — 2011](https://lwn.net/Articles/391970/)**
  أول نسخة من patch الـ erase/secure-erase — يغطي `MMC_ERASE_ARG` و`MMC_SECURE_ERASE_ARG`.

- **[Add MMC erase and secure erase V4 — 2011](https://lwn.net/Articles/395147/)**
  النسخة المحسّنة من نفس الـ patch series بعد review cycles.

- **[eMMC High Priority Interrupt (HPI) Feature — 2012](https://lwn.net/Articles/493070/)**
  patch series يغطي BKOPS field offsets في EXT_CSD والـ HPI invocation لمعالجة الـ foreground requests بأولوية.

- **[Replay Protected Memory Block (RPMB) subsystem — 2016](https://lwn.net/Articles/694798/)**
  مناقشة تصميم الـ RPMB subsystem — مهم لفهم `EXT_CSD_RPMB_MULT` وحماية البيانات.

- **[char:rpmb — RPMB subsystem patches — 2016](https://lwn.net/Articles/705850/)**
  patches الـ RPMB layer الفعلية للـ kernel.

- **[mmc: Add Command Queue support — 2017](https://lwn.net/Articles/738160/)**
  الـ CMDQ feature من eMMC 5.1 — مباشرةً يغطي `EXT_CSD_CMDQ_SUPPORT` و`EXT_CSD_CMDQ_DEPTH` الموجودين في الـ header.

- **[eMMC inline encryption support — 2021](https://lwn.net/Articles/843454/)**
  دعم الـ inline encryption على مستوى الـ MMC layer.

---

### Kernel Commits المهمة

الـ commits دي أضافت أو غيّرت مفاهيم أساسية في `mmc.h`:

| الميزة | ما تبحث عنه في `git log` |
|--------|--------------------------|
| EXT_CSD initial definitions | `git log --oneline -- include/linux/mmc/mmc.h` |
| CMDQ support (`EXT_CSD_CMDQ_*`) | `git log --grep="cmdq" -- drivers/mmc/` |
| HS200/HS400 card types | `git log --grep="HS200\|HS400" -- include/linux/mmc/` |
| RPMB mult field | `git log --grep="RPMB" -- include/linux/mmc/mmc.h` |
| BKOPS support | `git log --grep="bkops" -- drivers/mmc/core/` |
| Sanitize feature | `git log --grep="sanitize\|SANITIZE" -- include/linux/mmc/` |

للبحث المباشر في الـ kernel git:

```bash
# تاريخ الـ header file كامل
git log --oneline include/linux/mmc/mmc.h

# الـ commits اللي أضافت EXT_CSD fields
git log --all -p --follow include/linux/mmc/mmc.h | grep -A2 "EXT_CSD_CMDQ"
```

---

### Mailing List — linux-mmc@vger.kernel.org

الـ mailing list الرسمي للـ MMC subsystem:

- **الأرشيف**: [https://linux-mmc.vger.kernel.narkive.com/](https://linux-mmc.vger.kernel.narkive.com/)
- **Patchwork**: [https://patchwork.kernel.org/project/linux-mmc/list/](https://patchwork.kernel.org/project/linux-mmc/list/)
- **LKML Archive**: [https://lkml.iu.edu/hypermail/linux/kernel/](https://lkml.iu.edu/hypermail/linux/kernel/)

مناقشات مهمة محددة:

- [BKOPS support patches (v9)](https://linux-mmc.vger.kernel.narkive.com/1ZFD1anD/patch-v9-mmc-support-bkops-feature-for-emmc)
- [BKOPS support patches (v12)](https://linux-mmc.vger.kernel.narkive.com/oFbbbtgD/patch-v12-mmc-support-bkops-feature-for-emmc)
- [CMDQ host controller patches](https://patchwork.kernel.org/project/linux-arm-msm/patch/20141202120147.GA27669@asutoshd-ics.in.qualcomm.com/)
- [RPMB HS400 fix](https://lkml.iu.edu/hypermail/linux/kernel/2107.1/10560.html)

---

### KernelNewbies.org

الـ [kernelnewbies.org](https://kernelnewbies.org) بيوثّق التغييرات في كل kernel version — الـ MMC اتطوّر عبر releases كتير:

| الـ Release | التغيير |
|-------------|---------|
| [Linux 2.6.12](https://kernelnewbies.org/Linux_2_6_12) | أول SD protocol support |
| [Linux 2.6.17](https://kernelnewbies.org/Linux_2_6_17) | SDHCI driver، AT91 MMC |
| [Linux 2.6.28](https://kernelnewbies.org/Linux_2_6_28) | SDIO high-speed، atmel-mci DMA |
| [Linux 2.6.31](https://kernelnewbies.org/Linux_2_6_31) | via-sdmmc driver، CB710 reader |
| [Linux 2.6.32](https://kernelnewbies.org/Linux_2_6_32) | dm365 MMC/SD support |
| [Linux 4.9](https://kernelnewbies.org/Linux_4.9) | Arria10 SD-MMC EDAC |
| [LinuxChanges](https://kernelnewbies.org/LinuxChanges) | قائمة شاملة بكل التغييرات |

---

### eLinux.org

الـ [elinux.org](https://elinux.org) مهم لـ embedded Linux وتشغيل MMC/SD على boards حقيقية:

- **[Tests:eMMC-HS](https://elinux.org/Tests:eMMC-HS)** — اختبار HS200 على eMMC فعلي مع boot log samples
- **[Tests:SDHI-CMD23](https://elinux.org/Tests:SDHI-CMD23)** — اختبار SDHI مع `CONFIG_MMC_SDHI` و`CONFIG_MMC_DEBUG`
- **[EVM Second MMC / SD](https://elinux.org/Second_MMC_/_SD)** — إضافة second MMC slot على TI EVM boards
- **[AM335x recovery](https://elinux.org/AM335x_recovery)** — recovery باستخدام SD card images

---

### الكتب الموصى بها

#### Linux Device Drivers (LDD3) — Corbet, Rubini, Kroah-Hartman
المرجع الكلاسيكي لكتابة الـ kernel drivers. الطبعة المجانية:
[https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

الفصول ذات الصلة:
- **Chapter 3**: Char Drivers — فهم الـ device model الأساسي
- **Chapter 13**: USB Drivers — نفس مفاهيم الـ bus/device/driver تنطبق على MMC
- **Chapter 14**: The Linux Device Model

#### Linux Kernel Development — Robert Love (3rd Edition)
- **Chapter 13**: The Virtual Filesystem — فهم كيف الـ block layer يتعامل مع الـ storage
- **Chapter 14**: The Block I/O Layer — مباشرةً متعلق بكيف الـ MMC block driver يرسل requests
- **Chapter 17**: Devices and Modules — device registration وـ sysfs

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **Chapter 9**: File Systems — eMMC partitioning وـ boot partition
- **Chapter 14**: Bootloaders — كيف الـ bootloader يتعامل مع MMC قبل الـ kernel

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- **Chapter 6**: Device Drivers — تغطية شاملة للـ driver model
- بيشرح الـ `struct bus_type`، `struct device_driver`، وكيف الـ MMC core يستخدمهم

---

### أدوات مساعدة

#### mmc-utils
الأداة الرسمية للتعامل مع MMC/eMMC من userspace:

```bash
# تثبيت
apt install mmc-utils  # Debian/Ubuntu

# قراءة EXT_CSD كاملة
mmc extcsd read /dev/mmcblk0

# معرفة نوع الكارت وسرعته
mmc status get /dev/mmcblk0

# تفعيل BKOPS
mmc bkops enable /dev/mmcblk0
```

- المصدر: [https://git.kernel.org/pub/scm/linux/kernel/git/cjb/mmc-utils.git](https://git.kernel.org/pub/scm/linux/kernel/git/cjb/mmc-utils.git)
- التوثيق الرسمي: [https://static.lwn.net/kerneldoc/driver-api/mmc/mmc-tools.html](https://static.lwn.net/kerneldoc/driver-api/mmc/mmc-tools.html)

#### JEDEC Standards
الـ specs الأصلية — مطلوبة لفهم كل field في الـ header:

| الـ Spec | المحتوى |
|----------|---------|
| JESD84-B51 | eMMC 5.1 — يغطي CMDQ، HS400، BKOPS |
| JESD84-A441 | eMMC 4.41 — الأساس |
| SD Physical Layer Simplified Specification | SD commands وـ registers |

---

### مصطلحات البحث

للبحث عن معلومات إضافية استخدم الـ search terms دي:

```
# للبحث في LWN
site:lwn.net "MMC" OR "eMMC" linux kernel

# للبحث في LKML
site:lkml.org mmc EXT_CSD linux

# للبحث عن specs
JESD84-B51 eMMC 5.1 specification

# للبحث عن الكود مباشرة
git log --oneline --follow -- include/linux/mmc/mmc.h
git grep "EXT_CSD_CMDQ" -- drivers/mmc/

# للبحث في Patchwork
site:patchwork.kernel.org "linux-mmc" EXT_CSD
```

الـ subsystem maintainer حالياً هو **Ulf Hansson** — ابحث عن patches باسمه على LKML لتتابع أحدث التطورات.
## Phase 8: Writing simple module

### الفكرة

الـ tracepoint الأنسب للـ hook هو **`mmc_request_start`** المعرَّف في `include/trace/events/mmc.h`.
ده tracepoint بيتطلق كل ما الـ MMC core بيبعت request للكارت (read/write/command)، ومعاه `struct mmc_host` و`struct mmc_request` اللي بيحتووا على كل التفاصيل: opcode, arg, blocks, blk_addr.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * mmc_tracer.c — Hook mmc_request_start tracepoint to log every MMC command.
 *
 * Tested on Linux 6.x with CONFIG_TRACEPOINTS=y and CONFIG_MMC=m/y.
 * Build with the Makefile below, then: insmod mmc_tracer.ko
 */

#include <linux/module.h>       /* MODULE_*, module_init/exit          */
#include <linux/kernel.h>       /* pr_info()                           */
#include <linux/tracepoint.h>   /* register_trace_*, tracepoint_probe_ */
#include <linux/mmc/core.h>     /* struct mmc_host, mmc_request, mmc_command, mmc_data */
#include <linux/mmc/mmc.h>      /* MMC_READ_SINGLE_BLOCK, MMC_WRITE_BLOCK, opcodes */

/* ------------------------------------------------------------------ *
 * Callback — called every time mmc_request_start fires               *
 * ------------------------------------------------------------------ */
static void probe_mmc_request_start(void *ignore,
                                    struct mmc_host *host,
                                    struct mmc_request *mrq)
{
    const char *dir = "NONE";
    u32  opcode   = 0;
    u32  arg      = 0;
    unsigned int blocks = 0;
    unsigned int blk_addr = 0;

    /* Extract command fields safely — cmd may be NULL for data-only requests */
    if (mrq->cmd) {
        opcode = mrq->cmd->opcode;
        arg    = mrq->cmd->arg;
    }

    /* Determine transfer direction from data flags */
    if (mrq->data) {
        blocks   = mrq->data->blocks;
        blk_addr = mrq->data->blk_addr;

        if (mrq->data->flags & MMC_DATA_READ)
            dir = "READ";
        else if (mrq->data->flags & MMC_DATA_WRITE)
            dir = "WRITE";
        else
            dir = "OTHER";
    }

    pr_info("mmc_tracer: host=%s opcode=CMD%u arg=0x%08x dir=%s "
            "blocks=%u blk_addr=0x%x\n",
            mmc_hostname(host),   /* e.g. "mmc0"                       */
            opcode,               /* e.g. 18 = READ_MULTIPLE_BLOCK      */
            arg,                  /* address or RCA depending on opcode */
            dir,
            blocks,
            blk_addr);

    /* Flag dangerous commands — erase on a production device */
    if (opcode == MMC_ERASE)
        pr_warn("mmc_tracer: *** ERASE command detected on %s! ***\n",
                mmc_hostname(host));
}

/* ------------------------------------------------------------------ *
 * module_init — register the tracepoint probe                        *
 * ------------------------------------------------------------------ */
static int __init mmc_tracer_init(void)
{
    int ret;

    /*
     * register_trace_mmc_request_start() macro-generated by TRACE_EVENT.
     * Associates our callback with the tracepoint.
     * NULL = no private data passed to the probe (we use the mrq directly).
     */
    ret = register_trace_mmc_request_start(probe_mmc_request_start, NULL);
    if (ret) {
        pr_err("mmc_tracer: failed to register tracepoint: %d\n", ret);
        return ret;
    }

    pr_info("mmc_tracer: loaded — watching all MMC requests\n");
    return 0;
}

/* ------------------------------------------------------------------ *
 * module_exit — mandatory unregister to avoid dangling callback ptr  *
 * ------------------------------------------------------------------ */
static void __exit mmc_tracer_exit(void)
{
    /*
     * unregister_trace_mmc_request_start() removes our callback.
     * tracepoint_synchronize_unregister() waits for any in-flight RCU
     * read-side sections that might still be executing the probe.
     * Skipping this can cause a use-after-free when the module is unloaded.
     */
    unregister_trace_mmc_request_start(probe_mmc_request_start, NULL);
    tracepoint_synchronize_unregister();
    pr_info("mmc_tracer: unloaded\n");
}

module_init(mmc_tracer_init);
module_exit(mmc_tracer_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("MMC Learner");
MODULE_DESCRIPTION("Trace every MMC request via mmc_request_start tracepoint");
```

---

### Makefile

```makefile
obj-m += mmc_tracer.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

```bash
# Build and load
make
sudo insmod mmc_tracer.ko

# Watch output
sudo dmesg -w | grep mmc_tracer

# Trigger some I/O on the eMMC/SD card
dd if=/dev/mmcblk0 of=/dev/null bs=4K count=32

# Unload
sudo rmmod mmc_tracer
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | ماكروهات `MODULE_*` و `module_init/exit` |
| `linux/tracepoint.h` | `register_trace_*` و `tracepoint_synchronize_unregister` |
| `linux/mmc/core.h` | تعريفات `struct mmc_host`, `struct mmc_request`, `struct mmc_data` |
| `linux/mmc/mmc.h` | ماكروهات الـ opcodes زي `MMC_ERASE`, `MMC_READ_MULTIPLE_BLOCK` و `mmc_hostname()` |

**الـ** `trace/events/mmc.h` مش بنضمّه مباشرة في الـ module — الـ kernel بيعمل `EXPORT_TRACEPOINT_SYMBOL_GPL` للـ tracepoint، والـ `register_trace_mmc_request_start` بيتوفر من الـ kernel اللي بنبني ضده.

---

#### الـ Probe Callback

```c
static void probe_mmc_request_start(void *ignore,
                                    struct mmc_host *host,
                                    struct mmc_request *mrq)
```

- **`void *ignore`** — الـ data pointer اللي بنديه لـ `register_trace_*`؛ احنا بعّتنا `NULL` فمش بنستخدمه.
- **`struct mmc_host *host`** — يمثل الـ controller نفسه (مثلاً `mmc0`)، بنجيب منه `mmc_hostname()` عشان نعرف على أنهي slot.
- **`struct mmc_request *mrq`** — الـ request الكامل اللي بيتبعت للكارت، بيتضمن `cmd` و `data` و `stop`؛ كلهم ممكن يكونوا `NULL` فلازم نعمل check قبل ما نوصل للـ fields.

الـ callback بيطبع: اسم الـ host، رقم الـ opcode (CMD18 = READ_MULTIPLE_BLOCK مثلاً)، الـ arg (عنوان البلوك)، الاتجاه (READ/WRITE)، وعدد البلوكات. لو الـ opcode كان `MMC_ERASE` بيطلع `pr_warn` تحذيري.

---

#### الـ module_init

```c
ret = register_trace_mmc_request_start(probe_mmc_request_start, NULL);
```

الـ macro ده بيضيف الـ callback في قائمة الـ probes الخاصة بالـ tracepoint باستخدام RCU. لو الـ tracepoint مش موجود في الـ kernel (مثلاً `CONFIG_TRACEPOINTS=n`) بيرجع error ومش بنكمل.

---

#### الـ module_exit

```c
unregister_trace_mmc_request_start(probe_mmc_request_start, NULL);
tracepoint_synchronize_unregister();
```

الـ `unregister_trace_*` بيشيل الـ callback من القائمة، لكن ممكن يكون في thread لسه بيشغّل الـ callback في نفس اللحظة (RCU grace period). الـ `tracepoint_synchronize_unregister()` بتستنى لحد ما كل الـ readers الحاليين يخلصوا — من غيرها الـ kernel هيـ call pointer لكود اتحذف وهيحصل kernel panic.
