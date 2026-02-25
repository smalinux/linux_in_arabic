## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem: MMC/SD/SDIO

الملف ده جزء من **MMC (MultiMediaCard) subsystem** — اللي بيتحكم في كل بطاقات التخزين اللي بتشوفها في الأجهزة: بطاقات SD في الكاميرا، وشرائح eMMC الجوّانية في الموبايلات والـ Raspberry Pi، وبطاقات SDIO للـ Wi-Fi والـ Bluetooth.

---

### الصورة الكبيرة — ليه الملف ده موجود؟

#### القصة من الأول

تخيّل إنك عندك **مكتبة (library)** فيها كُتب (data). عشان تجيب كتاب أو تحط واحد جديد، لازم:
1. تقول للمكتبجي **"عايز كتاب رقم كذا"** — ده الـ **command**.
2. المكتبجي يروح يجيب الكتاب — ده الـ **data transfer**.
3. ترد عليك وتقوله **"تمام، استلمت"** — دي الـ **response**.

الـ CPU والـ MMC card بيتكلموا بنفس المنطق. الـ CPU بيبعت **commands** للكارت، والكارت بيرد بـ **responses**، وأحيانًا في **data** بيتنقل في الاتجاهين.

الملف `include/linux/mmc/core.h` هو **العقد المشترك** — بيعرّف شكل الـ command وشكل الـ data وشكل الـ request اللي بيجمعهم مع بعض.

---

### ليه الملف ده مهم؟

كل حاجة في الـ MMC subsystem بتتكلم عن طريق الـ structures اللي في الملف ده:

- **الـ core layer** (اللي بيدير المنطق) بيبني objects من الـ structs دي ويبعتها.
- **الـ host driver** (driver الـ hardware اللي بيتحكم في الـ slot الفعلي) بياخد نفس الـ objects ويترجمها لـ signals على الـ bus.
- **الـ block driver** (اللي بيظهر للـ filesystem ككأنه disk) بيستخدمهم عشان يطلب قراءة وكتابة.

يعني الملف ده هو **لغة التواصل المشتركة** بين كل طبقات الـ subsystem.

---

### الـ Structures الأساسية وبتعمل إيه

#### 1. `struct mmc_command` — الأمر اللي بيتبعت للكارت

```c
struct mmc_command {
    u32  opcode;   /* رقم الأمر، زي CMD17 للقراءة */
    u32  arg;      /* الـ argument، زي رقم الـ block */
    u32  resp[4];  /* الرد اللي بييجي من الكارت (لحد 136 bit) */
    unsigned int flags; /* نوع الرد المتوقع: R1, R2, R3... */
    unsigned int retries;
    int  error;
    /* ... */
};
```

الـ `opcode` هو رقم الأمر حسب **SD/MMC specification** — مثلًا:
- `CMD17` = قرا block واحد
- `CMD18` = قرا blocks متعددة
- `CMD24` = اكتب block واحد

الـ `flags` بتقول للـ host driver هيستنى رد من الكارت إيه بالظبط. الـ standard بيعرّف أنواع ردود:

| Type | الوصف |
|------|-------|
| `MMC_RSP_NONE` | مفيش رد |
| `MMC_RSP_R1` | رد 48-bit فيه status |
| `MMC_RSP_R1B` | R1 + الكارت ممكن يبعت busy signal |
| `MMC_RSP_R2` | رد 136-bit (زي CSD/CID registers) |
| `MMC_RSP_R3` | رد 48-bit (زي OCR register) |

#### 2. `struct mmc_data` — الـ Data المصاحب للأمر

```c
struct mmc_data {
    unsigned int blksz;      /* حجم الـ block بالـ bytes */
    unsigned int blocks;     /* عدد الـ blocks */
    unsigned int flags;      /* READ أو WRITE؟ */
    struct scatterlist *sg;  /* الـ memory buffers (scatter-gather list) */
    unsigned int sg_len;
    unsigned int bytes_xfered; /* كام byte اتنقل فعلًا */
    int error;
};
```

الـ `scatterlist` مهم جدًا — بدل ما الـ data تكون في buffer واحد متواصل في الـ RAM (مش دايمًا متاح)، بتقدر تكون **مقطّعة على أجزاء متفرقة**. الـ DMA controller بيعرف يتعامل مع الـ scatter-gather list ده مباشرة.

#### 3. `struct mmc_request` — الـ Package الكامل

```c
struct mmc_request {
    struct mmc_command *sbc;  /* SET_BLOCK_COUNT قبل الكتابة المتعددة */
    struct mmc_command *cmd;  /* الأمر الأساسي */
    struct mmc_data    *data; /* الـ data (لو موجود) */
    struct mmc_command *stop; /* CMD12 لإيقاف الـ transfer */

    struct completion completion;     /* للـ blocking wait */
    void (*done)(struct mmc_request *); /* callback لما يخلص */
    struct mmc_host *host;
    bool cap_cmd_during_tfr; /* ممكن تبعت commands تانية أثناء الـ transfer */
    int  tag;                /* للـ Command Queue Engine */
};
```

الـ `mmc_request` هو اللي بيتبعت من أعلى لتحت — بيجمع الـ command والـ data والـ stop command في حاجة واحدة.

#### 4. `struct uhs2_command` — UHS-II Protocol

```c
struct uhs2_command {
    u16  header;
    u16  arg;
    __be32 payload[UHS2_MAX_PAYLOAD_LEN]; /* max 2 payloads */
    u8   uhs2_resp[UHS2_MAX_RESP_LEN];    /* response buffer */
};
```

الـ **UHS-II** (Ultra High Speed II) هو بروتوكول جيل تاني للـ SD cards بيستخدم **2-lane differential signaling** بدل الـ parallel bus القديم — بيوصل لسرعات أعلى بكتير. الـ struct ده بيعرّف شكل الـ packet في البروتوكول الجديد ده.

---

### القصة الكاملة — رحلة الـ Read Request

تخيّل إن الـ filesystem طلب يقرا 4KB من بطاقة SD:

```
Filesystem (ext4)
      │
      ▼
Block Layer (block/bio.c)
      │  بيعمل bio request
      ▼
MMC Block Driver (drivers/mmc/core/block.c)
      │  بيبني mmc_request + mmc_command (CMD18) + mmc_data
      ▼
MMC Core (drivers/mmc/core/core.c)
      │  mmc_wait_for_req() — بيبعت الـ request وبيستنى
      ▼
Host Driver (e.g., drivers/mmc/host/sdhci.c)
      │  بيترجم mmc_request لـ hardware registers
      ▼
SD Card Hardware
      │  بيرد بالـ data
      ▼
الـ completion بيتعمله complete()
      │
      ▼
الـ block driver بيرجع البيانات للـ filesystem
```

الـ `mmc_wait_for_req()` و `mmc_wait_for_cmd()` — اللي متعرفين في الملف ده — هم البوابة الرسمية اللي أي حد عايز يكلم الكارت بيعدي منها.

---

### الـ API Functions المعرّفة في الملف

```c
/* بعت request كامل (cmd + data) وستنى لحد ما يخلص */
void mmc_wait_for_req(struct mmc_host *host, struct mmc_request *mrq);

/* بعت command بسيط وستنى رده */
int mmc_wait_for_cmd(struct mmc_host *host, struct mmc_command *cmd, int retries);

/* عمل hardware reset للكارت */
int mmc_hw_reset(struct mmc_card *card);

/* عمل software reset للكارت */
int mmc_sw_reset(struct mmc_card *card);

/* احسب الـ timeout المناسب للـ data transfer بناءً على specs الكارت */
void mmc_set_data_timeout(struct mmc_data *data, const struct mmc_card *card);
```

---

### الفرق بين Native MMC/SD وبروتوكول SPI

الملف بيعرّف flags لـ **نوعين** من الـ responses:

**Native (parallel bus):**
```
MMC_RSP_R1  = MMC_RSP_PRESENT | MMC_RSP_CRC | MMC_RSP_OPCODE
```

**الـ SPI mode** (لما الكارت متوصل عن طريق SPI controller):
```
MMC_RSP_SPI_R1 = MMC_RSP_SPI_S1  /* byte واحد فقط */
```

في الـ SPI mode شكل الـ responses مختلف تمامًا — أصغر وأبسط — عشان الـ SPI protocol نفسه مختلف عن الـ parallel MMC bus.

---

### الملفات المكوّنة للـ Subsystem

#### الـ Headers (interface definitions)
| الملف | الدور |
|-------|-------|
| `include/linux/mmc/core.h` | **الملف ده** — تعريف command/data/request |
| `include/linux/mmc/host.h` | تعريف `mmc_host` و `mmc_ios` (الـ hardware abstraction) |
| `include/linux/mmc/card.h` | تعريف `mmc_card` (الكارت نفسه وصفاته) |
| `include/linux/mmc/mmc.h` | constants للـ MMC protocol (opcodes, register bits) |
| `include/linux/mmc/sd.h` | constants للـ SD protocol |
| `include/linux/mmc/sdio.h` | constants للـ SDIO protocol |
| `include/linux/mmc/sd_uhs2.h` | تعريفات UHS-II |
| `include/linux/mmc/pm.h` | Power management flags |
| `include/linux/mmc/slot-gpio.h` | GPIO helpers للـ card detect/write-protect |

#### الـ Core Layer (المنطق)
| الملف | الدور |
|-------|-------|
| `drivers/mmc/core/core.c` | **قلب الـ subsystem** — scheduling, error handling, mmc_wait_for_req |
| `drivers/mmc/core/block.c` | الـ block device driver (بيظهر الكارت كـ /dev/mmcblkX) |
| `drivers/mmc/core/mmc.c` | init/detection لـ eMMC cards |
| `drivers/mmc/core/sd.c` | init/detection لـ SD cards |
| `drivers/mmc/core/sdio.c` | init/detection لـ SDIO cards |
| `drivers/mmc/core/mmc_ops.c` | تنفيذ الـ MMC commands الأساسية |
| `drivers/mmc/core/sd_ops.c` | تنفيذ الـ SD commands |
| `drivers/mmc/core/host.c` | إدارة الـ host controllers |
| `drivers/mmc/core/queue.c` | إدارة الـ request queue |
| `drivers/mmc/core/bus.c` | الـ MMC bus على نظام الـ device model |
| `drivers/mmc/core/crypto.c` | دعم الـ inline encryption (لو CONFIG_MMC_CRYPTO) |

#### الـ Host Drivers (Hardware-specific)
| المجلد | مثال |
|--------|------|
| `drivers/mmc/host/sdhci.c` | الـ generic SDHCI host controller (أكتر واحد منتشر) |
| `drivers/mmc/host/dw_mmc.c` | Synopsys DesignWare controller |
| `drivers/mmc/host/omap_hsmmc.c` | Texas Instruments OMAP |
| `drivers/mmc/host/meson-gx-mmc.c` | Amlogic SoCs |

---

### ملفات مهمة يستحسن القارئ يعرفها

- **`drivers/mmc/core/core.c`** — اللي بيكمّل تنفيذ `mmc_wait_for_req()` اللي متعلنة في الملف ده
- **`include/linux/mmc/host.h`** — بيكمّل الصورة بتعريف `mmc_host` اللي الـ `mmc_request` بيشاور عليه
- **`include/linux/mmc/card.h`** — بيعرّف `mmc_card` اللي فيه specs الكارت المكتشفة (سرعة، حجم، capabilities)
- **`drivers/mmc/core/block.c`** — بيوضح إزاي الـ structs دي بتتبني فعلًا من طرف الـ block layer
## Phase 2: شرح الـ MMC/SD Framework

### المشكلة — ليه الـ subsystem ده موجود أصلاً؟

تخيل إن عندك embedded system فيها eMMC على SoC من Rockchip، وحاجة تانية فيها SD card على SDHCI controller من Texas Instruments، وحاجة تالتة فيها SDIO Wi-Fi chip على controller تانى. الـ hardware مختلف تماماً، بس الـ storage stack فوق (الـ block layer، الـ filesystem) محتاج يشتغل بنفس الطريقة مع الكل.

**المشكلة الجوهرية** إنك لو اشتغلت من غير framework:
- كل driver هيكرر نفس logic الـ card initialization (CMD0, CMD2, CMD3, ACMD41)
- كل driver هيعمل retry logic، error handling، power sequencing
- الـ upper layers مش هتعرف تتكلم مع أي controller من غير ما تعرف الـ hardware

ده بالظبط اللي بيحله الـ **MMC/SD subsystem**: يوفر **abstraction layer** واحد بين الـ card protocols (MMC/SD/SDIO) من فوق، وبين الـ host controllers المختلفة من تحت.

---

### الحل — الـ Kernel بيتعامل إزاى؟

الـ kernel بيقسم المشكلة لـ 3 طبقات:

1. **الـ Core Layer** — الـ logic المشترك: card detection, initialization, protocol state machines, request routing
2. **الـ Card Drivers** — بيعرف "إيه" اللي فوق: `mmc_block` للـ block storage، `sdio_*` للـ SDIO functions
3. **الـ Host Drivers** — بيعرف "إيه" اللي تحت: sdhci، dw_mmc، omap_hsmmc — كل واحد بـ hardware مختلف

الـ core layer بيعمل كـ **mediator** بين الطبقتين. الـ card drivers مش بيتكلموا مع الـ host drivers مباشرة أبداً.

---

### التشبيه الحقيقي — مكتب البريد

تخيل **مكتب البريد** (الـ MMC core) اللي بيستقبل طلبات من **العملاء** (الـ card drivers / block layer) ويوصّلها لـ **فروع مختلفة** (الـ host controllers).

| مكتب البريد | الـ MMC Subsystem |
|---|---|
| العميل بيقول "وصّل الطرد ده" | الـ `mmc_block` driver بيبعت `mmc_request` |
| مكتب البريد بيحدد الطريق | الـ MMC core بيرتب الـ requests وبيدير الـ queue |
| موظف الفرع بيعرف كيف يشغّل الماكينة | الـ host driver بـ `mmc_host_ops` callback |
| الطرد نفسه | الـ `mmc_data` مع الـ scatter list |
| رقم التتبع + إيصال الاستلام | الـ `mmc_command` + الـ response bytes |
| الفرع بيعمل "تسليم مباشر" أو "عن طريق وسيط" | الـ host بيدعم DMA أو PIO |
| الفرع بيعرف speed limits المنطقة | الـ `mmc_ios` بيحدد clock, bus width, voltage |
| لو الطرد اتأخر، مكتب البريد بيعمل retry | الـ `mmc_command.retries` + error recovery |
| بعض الفروع بتعمل عمليات مجمعة (batch) | الـ Command Queue Engine (CQE) في الـ eMMC |

الفرق بين التشبيه والواقع: في البريد المادي في وقت انتظار فيزيائي، في الـ MMC core الـ synchronization بيتعمل عبر `struct completion` (من الـ Linux completion API — وهو mechanism بيخلي thread يستنى event معين قبل ما يكمل).

---

### الصورة الكبيرة — Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    User Space / VFS                          │
└───────────────────────┬─────────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────────┐
│              Block Layer (bio / request queue)               │
│              (subsystem منفصل — يحول I/O requests           │
│               لـ block operations)                          │
└───────────────────────┬─────────────────────────────────────┘
                        │
        ┌───────────────┴──────────────┐
        │                              │
┌───────▼────────┐           ┌─────────▼──────────┐
│  mmc_block     │           │  sdio_func drivers  │
│  (block driver)│           │  (WiFi, BT, etc.)   │
└───────┬────────┘           └─────────┬────────────┘
        │  mmc_wait_for_req()          │
        └──────────────┬───────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                   MMC Core Layer                             │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────┐  │
│  │  Card Init  │  │  Request     │  │  Error Recovery    │  │
│  │  & Detect   │  │  Scheduling  │  │  & Retry Logic     │  │
│  └─────────────┘  └──────────────┘  └────────────────────┘  │
│                                                              │
│  Core Structs: mmc_request, mmc_command, mmc_data            │
└──────────────────────┬──────────────────────────────────────┘
                       │  mmc_host_ops callbacks
                       │  (request, set_ios, get_cd, ...)
        ┌──────────────┴──────────────────────┐
        │                                     │
┌───────▼────────┐                   ┌────────▼───────┐
│  sdhci-pltfm   │                   │  dw_mmc        │
│  (SDHCI host)  │                   │  (DesignWare)  │
└───────┬────────┘                   └────────┬───────┘
        │                                     │
┌───────▼────────┐                   ┌────────▼───────┐
│  Physical SDHC │                   │  Physical DW   │
│  Controller    │                   │  Controller    │
└───────┬────────┘                   └────────┬───────┘
        │                                     │
┌───────▼────────┐                   ┌────────▼───────┐
│  eMMC / SD     │                   │  SD Card       │
│  Card          │                   │                │
└────────────────┘                   └────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ abstraction المحورية في الـ MMC subsystem هي **الـ Transaction Model**: أي عملية I/O — سواء read/write/erase/tuning — بتتحول لـ `mmc_request` واحد بيتنقل عبر الـ layers.

```
┌─────────────────────────────────────────┐
│             mmc_request                  │
│                                          │
│  ┌──────────────┐   ┌──────────────┐    │
│  │  mmc_command │   │  mmc_command │    │
│  │    (sbc)     │   │    (cmd)     │    │
│  │ SET_BLOCK_   │   │ READ/WRITE   │    │
│  │ COUNT first  │   │ actual cmd   │    │
│  └──────────────┘   └──────┬───────┘    │
│                            │            │
│                     ┌──────▼───────┐    │
│                     │   mmc_data   │    │
│                     │              │    │
│                     │ sg[] ──────► scatter│
│                     │              │    list│
│                     │ blksz=512    │    │
│                     │ blocks=8     │    │
│                     └──────────────┘    │
│                                          │
│  ┌──────────────┐                        │
│  │  mmc_command │                        │
│  │    (stop)    │                        │
│  │ CMD12 STOP   │                        │
│  └──────────────┘                        │
│                                          │
│  completion  ◄── يستنى الـ thread هنا    │
│  done()  ◄── callback لما يخلص           │
└─────────────────────────────────────────┘
```

الـ `mmc_request` هو الـ **unit of work** — زي الـ `bio` في الـ block layer، أو الـ `urb` في الـ USB subsystem.

---

### شرح الـ Structs التفصيلي

#### `struct mmc_command` — الأمر الواحد

```c
struct mmc_command {
    u32  opcode;    /* رقم الـ command: CMD0=0, CMD17=17 (READ), CMD24=24 (WRITE) */
    u32  arg;       /* الـ argument: عادةً block address أو config bits */
    u32  resp[4];   /* الـ response من الـ card: R1=resp[0], R2=resp[0..3] */
    unsigned int flags; /* نوع الـ response المتوقع */

    unsigned int retries;      /* كام مرة يعيد لو فشل */
    int          error;        /* نتيجة التنفيذ: 0=نجح, -ETIMEDOUT, -EILSEQ */
    unsigned int busy_timeout; /* أقصى وقت ينتظر الـ card يخلص (ms) */

    struct mmc_data    *data;  /* لو الـ command معاه data transfer */
    struct mmc_request *mrq;   /* الـ request اللي ينتمي ليه */
    struct uhs2_command *uhs2_cmd; /* للـ UHS-II protocol */
};
```

الـ **flags** بتحدد نوع الـ response:

```
MMC_RSP_NONE  → CMD0 (GO_IDLE) — مفيش response خالص
MMC_RSP_R1    → CMD17 (READ_SINGLE_BLOCK) — 32-bit status
MMC_RSP_R1B   → CMD24 (WRITE_BLOCK) — R1 + busy signal على DAT0
MMC_RSP_R2    → CMD2 (ALL_SEND_CID) — 136-bit CID/CSD register
MMC_RSP_R3    → ACMD41 — OCR register
```

الـ host driver لازم يعرف نوع الـ response عشان يعرف يقرأ كام byte ويتحقق من CRC ولا لأ.

#### `struct mmc_data` — البيانات

```c
struct mmc_data {
    unsigned int timeout_ns;   /* الـ timeout بالـ nanoseconds */
    unsigned int blksz;        /* حجم الـ block (عادةً 512 bytes) */
    unsigned int blocks;       /* عدد الـ blocks */
    int          error;        /* -ETIMEDOUT أو -EILSEQ لو في مشكلة */
    unsigned int flags;        /* MMC_DATA_READ أو MMC_DATA_WRITE */

    unsigned int bytes_xfered; /* إيه اللي اتنقل فعلاً */

    struct mmc_command *stop;  /* CMD12 لإيقاف الـ multi-block transfer */
    struct scatterlist *sg;    /* الـ I/O scatter list */
    unsigned int       sg_len; /* عدد الـ segments */
    int                sg_count; /* عدد الـ mapped DMA entries */
    s32           host_cookie; /* الـ host driver يحط فيه private data */
};
```

الـ `scatterlist` مهم جداً — **الـ Scatter-Gather DMA**: بدل ما الـ data تكون في buffer واحد contiguous في الـ RAM، ممكن تكون في أجزاء متفرقة. الـ DMA controller بيقرأ الـ scatterlist ويجمّع الـ data من الأماكن دي. ده بيخلي الـ zero-copy I/O ممكن.

```
sg[0]: physical_addr=0x80001000, len=4096
sg[1]: physical_addr=0x80005000, len=4096
sg[2]: physical_addr=0x80003000, len=2048
         ↓
   DMA Controller
         ↓
   eMMC Card (continuous stream)
```

#### `struct mmc_request` — الـ Transaction الكامل

```c
struct mmc_request {
    struct mmc_command *sbc;  /* Optional: CMD23 SET_BLOCK_COUNT قبل الـ transfer */
    struct mmc_command *cmd;  /* الـ command الرئيسي */
    struct mmc_data    *data; /* البيانات (NULL للـ commands من غير data) */
    struct mmc_command *stop; /* CMD12 بعد الـ multi-block transfer */

    struct completion   completion;     /* الـ core يستنى هنا لما يخلص */
    struct completion   cmd_completion; /* للـ async command */

    void (*done)(struct mmc_request *); /* callback لما يخلص */
    void (*recovery_notifier)(struct mmc_request *); /* لو في error */

    struct mmc_host *host; /* الـ host اللي بيتنفذ عليه */
    bool  cap_cmd_during_tfr; /* هل ممكن تبعت commands أثناء الـ data transfer */
    int   tag; /* لـ Command Queue */
};
```

#### `struct mmc_host` — الـ Controller

ده الـ struct الأكبر والأهم. بيمثل الـ hardware controller:

```c
struct mmc_host {
    struct device        *parent;   /* الـ platform device */
    const struct mmc_host_ops *ops; /* vtable الـ callbacks */

    /* Hardware capabilities */
    unsigned int  f_min, f_max;     /* نطاق الـ clock المدعوم */
    u32           ocr_avail;        /* الـ voltages المدعومة */
    u32           caps, caps2;      /* feature flags */

    /* Current bus state */
    struct mmc_ios ios;             /* clock, voltage, bus width الحالية */

    /* Attached card */
    struct mmc_card *card;

    /* DMA limits */
    unsigned int  max_seg_size;
    unsigned short max_segs;
    unsigned int  max_req_size;
    unsigned int  max_blk_size;
    unsigned int  max_blk_count;

    /* CQE support */
    const struct mmc_cqe_ops *cqe_ops;
    bool  cqe_enabled, cqe_on;

    /* private data للـ host driver */
    unsigned long private[] ____cacheline_aligned;
};
```

الـ `private[]` في الآخر ده **flexible array member** — الـ host driver بيطلب extra bytes لما بيعمل `mmc_alloc_host(sizeof(struct my_host_priv), dev)`. الـ core ميعرفش إيه جوّاه، بس الـ host driver بيوصله بـ `mmc_priv(host)`.

#### `struct mmc_host_ops` — الـ Vtable

```c
struct mmc_host_ops {
    /* الأساسي — إرسال request */
    void (*request)(struct mmc_host *host, struct mmc_request *req);
    int  (*request_atomic)(struct mmc_host *host, struct mmc_request *req);

    /* Double buffering: جهّز request وأنت شغّال request تاني */
    void (*pre_req)(struct mmc_host *host, struct mmc_request *req);
    void (*post_req)(struct mmc_host *host, struct mmc_request *req, int err);

    /* ضبط الـ bus settings */
    void (*set_ios)(struct mmc_host *host, struct mmc_ios *ios);

    /* Card detection */
    int  (*get_cd)(struct mmc_host *host);  /* 1=present, 0=absent */
    int  (*get_ro)(struct mmc_host *host);  /* 1=read-only */

    /* Voltage switching للـ UHS */
    int  (*start_signal_voltage_switch)(struct mmc_host *host, struct mmc_ios *ios);

    /* Clock tuning للـ HS200/HS400 */
    int  (*execute_tuning)(struct mmc_host *host, u32 opcode);

    /* Hardware reset للـ eMMC */
    void (*card_hw_reset)(struct mmc_host *host);
};
```

---

### العلاقات بين الـ Structs

```
mmc_host
   │
   ├──► ops (mmc_host_ops) ── vtable للـ hardware operations
   │
   ├──► ios (mmc_ios) ── current bus state
   │         ├── clock: 400000 (400 KHz during init), 52000000 (52 MHz HS)
   │         ├── bus_width: MMC_BUS_WIDTH_4
   │         ├── timing: MMC_TIMING_MMC_HS200
   │         └── signal_voltage: MMC_SIGNAL_VOLTAGE_180
   │
   ├──► card (mmc_card) ── الـ card المتصلة
   │         ├── host ◄── back-pointer
   │         ├── type: MMC_TYPE_MMC / SD / SDIO
   │         ├── cid (مصنع، serial، اسم)
   │         ├── csd (capacity، timing)
   │         ├── ext_csd (eMMC v4+ features)
   │         └── part[] (physical partitions)
   │
   └──► cqe_ops (mmc_cqe_ops) ── للـ Command Queue Engine (optional)
```

```
mmc_request (الـ transaction)
   │
   ├──► sbc (mmc_command) ── CMD23 SET_BLOCK_COUNT (optional)
   │         └── opcode=23, arg=block_count, flags=MMC_RSP_R1
   │
   ├──► cmd (mmc_command) ── CMD18 READ_MULTIPLE_BLOCK
   │         ├── opcode=18, arg=start_block
   │         ├── flags=MMC_RSP_R1
   │         └──► data (mmc_data) ──► sg[] (scatter list)
   │                   ├── blksz=512
   │                   ├── blocks=8
   │                   └── flags=MMC_DATA_READ
   │
   └──► stop (mmc_command) ── CMD12 STOP_TRANSMISSION
             └── opcode=12, flags=MMC_RSP_R1B
```

---

### الـ Request Flow — من الـ Block Layer للـ Hardware

```
mmc_block driver
      │
      │  bio from filesystem
      ▼
  mmc_blk_mq_issue_rq()
      │
      │  يبني mmc_request + mmc_command + mmc_data
      ▼
  mmc_wait_for_req(host, mrq)          ← تعريفه في core.h
      │
      │  يضيف الـ request على queue
      ▼
  __mmc_start_request()
      │
      │  يتحقق من الـ retune، يجهز الـ DMA maps
      ▼
  host->ops->request(host, mrq)       ← الـ vtable call
      │
      │  (الـ host driver بيتولى من هنا)
      │  - يكتب الـ command registers
      │  - يفعّل الـ DMA
      │  - ينتظر interrupt
      ▼
  mmc_request_done(host, mrq)         ← الـ host driver بيناديها لما يخلص
      │
      │  يعمل complete() على الـ completion
      ▼
  mmc_wait_for_req() يرجع
      │
      ▼
  mmc_block driver يفحص cmd->error وdata->error
```

---

### إيه اللي الـ Core بيملكه مقابل إيه اللي بيفوّضه

#### الـ Core بيملك (يعمله بنفسه):
- **Card Detection & Initialization** — protocol state machine: CMD0 → CMD2 → CMD3 → CMD7
- **Error Recovery** — إعادة المحاولة بعد timeout أو CRC error
- **Request Queuing** — الـ serialization بين الـ upper layers
- **Bus Claiming** — `mmc_claim_host()` / `mmc_release_host()` (mutex على الـ bus)
- **Re-tuning** — بيحدد امتى يعمل tuning لو اكتشف CRC errors
- **Power Management** — runtime PM, suspend/resume sequence
- **UHS Negotiation** — بيتفاوض مع الـ card على أعلى speed مدعوم

#### الـ Core بيفوّض للـ Host Driver:
- **الـ Clock Generation** — `set_ios()` بيحدد الـ frequency، الـ hardware بيولّدها
- **الـ DMA Transfer** — الـ host driver بيعمل `dma_map_sg()` ويبرمج الـ ADMA/SDMA descriptors
- **الـ Interrupt Handling** — الـ host driver بيتعامل مع الـ hardware IRQ ويستدعي `mmc_request_done()`
- **الـ Voltage Switching** — `start_signal_voltage_switch()` بيتحكم في الـ regulator وبيخلّي الـ clock
- **الـ Tuning Execution** — `execute_tuning()` بيعمل الـ sampling window calibration
- **الـ Hardware Quirks** — كل حاجة hardware-specific بتروح في الـ host driver

#### الـ Card Driver (مثلاً mmc_block) بيفوّض للـ Core:
- بناء الـ requests بشكل صح
- استدعاء `mmc_wait_for_req()` أو `mmc_wait_for_cmd()`
- التعامل مع نتيجة الـ error codes

---

### الـ mmc_ios — حالة الـ Bus

```c
struct mmc_ios {
    unsigned int  clock;          /* الـ clock الحالية بالـ Hz */
    unsigned short vdd;           /* الـ voltage bit index */
    unsigned char bus_width;      /* 1, 4, أو 8 bits */
    unsigned char timing;         /* MMC_TIMING_* */
    unsigned char signal_voltage; /* 3.3V أو 1.8V أو 1.2V */
    unsigned char power_mode;     /* OFF / UP / ON */
    unsigned char drv_type;       /* drive strength A/B/C/D */
};
```

الـ `set_ios()` callback هو أكتر function بيتناديها في طول initialization. في كل مرحلة من مراحل negotiation، الـ core بيغيّر الـ ios struct وبيستدعي `set_ios()` عشان الـ hardware يطبّق الإعدادات الجديدة.

مثال على sequence الـ eMMC initialization:

```
1. set_ios(clock=400KHz, width=1bit, voltage=3.3V, power=UP)
2. CMD0 → reset
3. CMD1 → negotiate voltage
4. set_ios(clock=400KHz, width=1bit, voltage=3.3V, power=ON)
5. CMD2 → get CID
6. CMD3 → assign RCA
7. CMD7 → select card
8. CMD8 → get EXT_CSD
9. set_ios(clock=52MHz, width=8bit, voltage=1.8V, timing=HS200)
10. CMD21 → tuning (optional for HS200)
```

---

### الـ Command Queue Engine (CQE)

في الـ modern eMMC (v5.1+)، الـ card نفسها عندها **command queue** بتقبل 32 request في نفس الوقت وتعمل reordering هي. الـ `mmc_cqe_ops` struct بيوفر interface منفصل للـ CQE:

```
Normal flow:        CQE flow:
mmc_request         mmc_request (tag=0..31)
     ↓                   ↓
host->ops->request()  cqe_ops->cqe_request()
     ↓                   ↓
wait completion      returns immediately
                         ↓
                    card executes in its own order
                         ↓
                    interrupt per completed tag
```

الـ CQE بيحتاج `MMC_CAP2_CQE` flag في الـ host وبيشتغل مع `MMC_DATA_QBR` و `MMC_DATA_PRIO` flags في الـ mmc_data.

---

### الـ UHS-II — الجيل الجديد

الـ `uhs2_command` struct (موجود في core.h) بيعكس إن الـ UHS-II بيستخدم protocol مختلف تماماً عن الـ UHS-I:

```
UHS-I:  CMD line (1 bit, half-duplex) + DAT lines (4/8 bits)
UHS-II: LVDS lanes — packet-based protocol (header + payload)
        يوصل لـ 624 MB/s
```

الـ `uhs2_command.header` و `uhs2_command.arg` بيمثلوا الـ UHS-II packet format، وهو مختلف كلياً عن الـ SD command format التقليدي.

---

### ملخص التسلسل الهرمي

```
VFS / User Space
      │ (file operations)
Block Layer
      │ (bio requests)
mmc_block driver          ← card driver: يعرف "إيه" اللي فوق
      │ (mmc_request)
MMC Core (core.h)         ← قلب الـ subsystem: protocol logic
      │ (mmc_host_ops callbacks)
Host Drivers (sdhci, etc) ← يعرف "إيه" اللي تحت
      │ (register reads/writes + DMA)
Hardware Controller
      │ (CMD/DAT/CLK lines)
MMC/SD/SDIO Card
```

كل struct في `core.h` بيخدم طبقة واحدة في الهرم ده. مفيش طبقة بتتجاوز الـ interface المحدد لها — ده هو جوهر الـ MMC framework.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. Cheatsheet: الـ Flags والـ Macros المهمة

#### أولاً: response flags في `mmc_command.flags`

| Flag | Bit | المعنى |
|---|---|---|
| `MMC_RSP_PRESENT` | bit 0 | الـ card هيرد بـ response |
| `MMC_RSP_136` | bit 1 | الـ response طوله 136 bit (R2) |
| `MMC_RSP_CRC` | bit 2 | في CRC في الـ response |
| `MMC_RSP_BUSY` | bit 3 | الـ card ممكن يبعت busy signal |
| `MMC_RSP_OPCODE` | bit 4 | الـ response فيه opcode رجع |
| `MMC_RSP_SPI_S1` | bit 7 | SPI: byte status واحد |
| `MMC_RSP_SPI_S2` | bit 8 | SPI: byte status تاني |
| `MMC_RSP_SPI_B4` | bit 9 | SPI: 4 bytes data |
| `MMC_RSP_SPI_BUSY` | bit 10 | SPI: card ممكن يبعت busy |

#### ثانياً: Response Types المركّبة (Native)

| Type | تركيبه | الاستخدام |
|---|---|---|
| `MMC_RSP_NONE` | `0` | مفيش response خالص |
| `MMC_RSP_R1` | `PRESENT + CRC + OPCODE` | أغلب الـ commands العادية |
| `MMC_RSP_R1B` | `R1 + BUSY` | commands زي STOP أو ERASE |
| `MMC_RSP_R1B_NO_CRC` | `PRESENT + OPCODE + BUSY` | R1B من غير CRC check |
| `MMC_RSP_R2` | `PRESENT + 136 + CRC` | CID وCSD registers |
| `MMC_RSP_R3` | `PRESENT` | OCR register (لا CRC ولا opcode) |
| `MMC_RSP_R4` | `PRESENT` | SDIO: I/O send op cond |
| `MMC_RSP_R5` | `PRESENT + CRC + OPCODE` | SDIO: I/O read/write direct |
| `MMC_RSP_R6` | `PRESENT + CRC + OPCODE` | SD: published RCA response |
| `MMC_RSP_R7` | `PRESENT + CRC + OPCODE` | SD: interface condition |

#### ثالثاً: Response Types المركّبة (SPI)

| Type | تركيبه |
|---|---|
| `MMC_RSP_SPI_R1` | `SPI_S1` |
| `MMC_RSP_SPI_R1B` | `SPI_S1 + SPI_BUSY` |
| `MMC_RSP_SPI_R2` | `SPI_S1 + SPI_S2` |
| `MMC_RSP_SPI_R3` | `SPI_S1 + SPI_B4` |
| `MMC_RSP_SPI_R4` | `SPI_S1 + SPI_B4` |
| `MMC_RSP_SPI_R5` | `SPI_S1 + SPI_S2` |
| `MMC_RSP_SPI_R7` | `SPI_S1 + SPI_B4` |

#### رابعاً: Command Types في `MMC_CMD_MASK` (bits 5:6)

| Macro | Value | المعنى |
|---|---|---|
| `MMC_CMD_AC` | `0 << 5` | Addressed Command (no data) |
| `MMC_CMD_ADTC` | `1 << 5` | Addressed Data Transfer Command |
| `MMC_CMD_BC` | `2 << 5` | Broadcast Command (no response) |
| `MMC_CMD_BCR` | `3 << 5` | Broadcast Command with Response |

#### خامساً: Data Flags في `mmc_data.flags`

| Flag | Bit | المعنى |
|---|---|---|
| `MMC_DATA_WRITE` | bit 8 | الـ transfer من الـ host للـ card |
| `MMC_DATA_READ` | bit 9 | الـ transfer من الـ card للـ host |
| `MMC_DATA_QBR` | bit 10 | CQE: queue barrier (flush قبله) |
| `MMC_DATA_PRIO` | bit 11 | CQE: high priority request |
| `MMC_DATA_REL_WR` | bit 12 | Reliable Write (power-loss safe) |
| `MMC_DATA_DAT_TAG` | bit 13 | Tagged request للـ CQE |
| `MMC_DATA_FORCED_PRG` | bit 14 | Forced programming |

#### سادساً: CMD23 Argument Flags

| Flag | Bit | المعنى |
|---|---|---|
| `MMC_CMD23_ARG_REL_WR` | bit 31 | Reliable Write للـ SET_BLOCK_COUNT |
| `MMC_CMD23_ARG_TAG_REQ` | bit 29 | Tagged request للـ SET_BLOCK_COUNT |

#### سابعاً: UHS2 Constants

| Macro | Value | المعنى |
|---|---|---|
| `UHS2_MAX_PAYLOAD_LEN` | 2 | أقصى عدد `__be32` في الـ payload |
| `UHS2_MAX_RESP_LEN` | 20 | أقصى طول للـ UHS2 native response (bytes) |

#### ثامناً: Error Codes المستخدمة

| errno | المعنى في سياق MMC |
|---|---|
| `ETIMEDOUT` | الـ card أخد وقت أطول من المسموح في الرد |
| `EILSEQ` | مشكلة في الـ format: CRC fail، opcode غلط، bad end bit |
| `EINVAL` | الـ request مش ممكن يتنفذ بسبب hardware/driver restrictions |
| `ENOMEDIUM` | الـ slot فاضي والـ host شايف كده ومش قادر يكمل |

---

### 1. الـ Structs المهمة

#### `struct uhs2_command`

**الغرض:** تمثّل command واحدة على بروتوكول **UHS-II** — وهو interface جديد للـ SD cards بيشتغل على 2 lanes بدل الـ single-wire القديم، ويوفر bandwidth أعلى بكتير.

| Field | Type | الشرح |
|---|---|---|
| `header` | `u16` | بيحتوي على IOADR ونوع الـ packet (CCMD/DCMD) |
| `arg` | `u16` | الـ argument الخاص بالـ UHS2 command |
| `payload[]` | `__be32[2]` | الـ payload بـ big-endian — لحد 2 كلمة 32-bit |
| `payload_len` | `u8` | الطول الفعلي للـ payload بالـ bytes |
| `packet_len` | `u8` | الطول الكلي للـ packet |
| `tmode_half_duplex` | `u8` | flag لو الـ transfer mode هو half-duplex |
| `uhs2_resp[]` | `u8[20]` | buffer للـ native UHS2 response |
| `uhs2_resp_len` | `u8` | طول الـ response الفعلي |

**ارتباطاته:** محتوى جوّه `mmc_command` (كـ pointer) وجوّه `mmc_request` (كـ embedded struct).

---

#### `struct mmc_command`

**الغرض:** تمثّل **command واحدة** بتتبعت للـ MMC/SD card — من أبسط CMD0 (GO_IDLE) لحد أعقد multiblock commands.

| Field | Type | الشرح |
|---|---|---|
| `opcode` | `u32` | رقم الـ command (مثلاً CMD17 = single block read) |
| `arg` | `u32` | الـ argument المرفق مع الـ command |
| `resp[4]` | `u32[4]` | بيخزن الـ response: لو R2 = 136bit يملا الأربعة، غيره يكفي `resp[0]` |
| `flags` | `unsigned int` | تحديد نوع الـ response المتوقع + نوع الـ command (AC/ADTC/BC/BCR) |
| `retries` | `unsigned int` | أقصى عدد محاولات إعادة في حالة الفشل |
| `error` | `int` | نتيجة تنفيذ الـ command (0 = نجاح، أو errno) |
| `busy_timeout` | `unsigned int` | مهلة انتظار الـ busy signal بالـ milliseconds |
| `data` | `*mmc_data` | pointer للـ data segment لو الـ command بتنقل بيانات |
| `mrq` | `*mmc_request` | back-pointer للـ request اللي بيحتويه |
| `uhs2_cmd` | `*uhs2_command` | pointer لـ UHS2 command descriptor لو الـ interface UHS2 |
| `has_ext_addr` | `bool` | لـ **SDUC** (Ultra Capacity): هل في extended address؟ |
| `ext_addr` | `u8` | الـ extended address bits للـ SDUC cards (>2TB) |

---

#### `struct mmc_data`

**الغرض:** تصف **الـ data segment** المرتبط بـ command — يعني الـ buffer والـ DMA mapping اللي هيتنقل فعلاً على الـ bus.

| Field | Type | الشرح |
|---|---|---|
| `timeout_ns` | `unsigned int` | مهلة الـ data transfer بالـ nanoseconds (أقصاه ~80ms) |
| `timeout_clks` | `unsigned int` | مهلة إضافية بعدد الـ clock cycles (بتتضاف للـ `timeout_ns`) |
| `blksz` | `unsigned int` | حجم الـ block الواحد بالـ bytes (عادةً 512) |
| `blocks` | `unsigned int` | عدد الـ blocks في الـ transfer |
| `blk_addr` | `unsigned int` | عنوان أول block |
| `error` | `int` | خطأ في الـ data phase (0 = تمام) |
| `flags` | `unsigned int` | اتجاه الـ transfer + CQE flags |
| `bytes_xfered` | `unsigned int` | عدد الـ bytes اللي اتنقلت فعلاً |
| `stop` | `*mmc_command` | الـ stop command (CMD12) المرتبط بالـ multiblock transfer |
| `mrq` | `*mmc_request` | back-pointer للـ request |
| `sg_len` | `unsigned int` | عدد العناصر في الـ scatter-gather list |
| `sg_count` | `int` | عدد الـ SG entries بعد الـ DMA mapping |
| `sg` | `*scatterlist` | الـ scatter-gather list الخاص بالـ DMA |
| `host_cookie` | `s32` | private data للـ host driver (pre-mapped DMA cookie) |

---

#### `struct mmc_request`

**الغرض:** الـ **top-level container** — بيجمع كل حاجة في request واحدة: الـ optional pre-command، الـ main command، الـ data، والـ stop command. ده اللي بيتبعت للـ `mmc_host`.

| Field | Type | الشرح |
|---|---|---|
| `sbc` | `*mmc_command` | SET_BLOCK_COUNT (CMD23) — optional، بيتبعت قبل الـ main command في multiblock |
| `cmd` | `*mmc_command` | الـ main command |
| `data` | `*mmc_data` | الـ data descriptor (NULL لو مفيش data) |
| `stop` | `*mmc_command` | STOP_TRANSMISSION (CMD12) — optional، بعد multiblock |
| `completion` | `struct completion` | الـ kernel completion للـ synchronous waiting |
| `cmd_completion` | `struct completion` | completion خاص بالـ command phase (في CMD + DATA منفصلين) |
| `done` | `fn pointer` | callback بيتعمله call لما الـ request يخلص |
| `recovery_notifier` | `fn pointer` | callback لإعلام الـ upper layers (مثلاً block driver) إن في error ومحتاج recovery |
| `host` | `*mmc_host` | الـ host اللي بينفذ الـ request |
| `cap_cmd_during_tfr` | `bool` | لو `true`: ممكن ييجي command تاني أثناء الـ data transfer أو busy wait |
| `tag` | `int` | رقم تعريفي للـ request (مستخدم مع CQE) |
| `crypto_ctx` | `*bio_crypt_ctx` | (ifdef `CONFIG_MMC_CRYPTO`) سياق الـ inline encryption |
| `crypto_key_slot` | `int` | (ifdef `CONFIG_MMC_CRYPTO`) رقم الـ key slot المستخدم في الـ hardware encryption |
| `uhs2_cmd` | `struct uhs2_command` | **embedded** UHS2 command (مش pointer — موجود دايماً في الـ struct) |

---

### 2. مخطط علاقات الـ Structs

```
┌─────────────────────────────────────────────────────────────────────┐
│                        mmc_request                                  │
│                                                                     │
│  *sbc ──────────────────────┐                                       │
│  *cmd ──────────────────────┼──► mmc_command                        │
│  *stop ─────────────────────┘    ├── opcode, arg, resp[4]           │
│                                  ├── flags (RSP_* | CMD_*)          │
│  *data ────────────────────────► mmc_data                │          │
│                                  │   ├── blksz, blocks   │          │
│                                  │   ├── flags (R/W/CQE) │          │
│                                  │   ├── *sg ──────────────► scatterlist[] │
│                                  │   ├── *stop ──────────────► mmc_command │
│                                  │   └── *mrq (back) ─────────────────┐   │
│                                  │                        │           │   │
│                                  ├── *data (→ mmc_data)  │           │   │
│                                  ├── *mrq (back) ─────────────────────┘   │
│                                  └── *uhs2_cmd ─────────► uhs2_command    │
│                                                                            │
│  completion ─────────────────────► struct completion (swait based)        │
│  cmd_completion ─────────────────► struct completion                      │
│  *host ───────────────────────────► mmc_host (defined elsewhere)          │
│  done() ──────────────────────────► callback fn                           │
│  recovery_notifier() ─────────────► callback fn                           │
│  uhs2_cmd (embedded) ─────────────► uhs2_command (inline in struct)       │
│                                                                            │
│  [CONFIG_MMC_CRYPTO]                                                       │
│  *crypto_ctx ─────────────────────► bio_crypt_ctx                         │
│  crypto_key_slot                                                           │
└────────────────────────────────────────────────────────────────────────────┘
```

**ملاحظة مهمة على الـ back-pointers:**
- الـ `mmc_command.mrq` بيشاور على الـ `mmc_request` اللي بيحتويه.
- الـ `mmc_data.mrq` بيشاور على نفس الـ `mmc_request`.
- الـ `mmc_data.stop` بيشاور على `mmc_command` (CMD12) — اللي هو نفس `mmc_request.stop`.

---

### 3. مخطط الـ Lifecycle

#### دورة حياة الـ `mmc_request`

```
[Allocation]
    │
    ▼
mmc_request + mmc_command + mmc_data يتخصصوا (stack أو kmalloc)
    │
    ▼
[Initialization]
    │
    ├── init_completion(&mrq->completion)
    ├── mrq->cmd = &cmd
    ├── mrq->data = &data         (لو في data)
    ├── mrq->stop = &stop_cmd     (لو multiblock)
    ├── mrq->sbc = &sbc_cmd       (لو CMD23 مطلوب)
    ├── cmd->opcode = CMD_NUMBER
    ├── cmd->arg = argument
    ├── cmd->flags = MMC_RSP_R1 | MMC_CMD_ADTC
    ├── data->blksz = 512
    ├── data->blocks = N
    ├── data->flags = MMC_DATA_READ
    └── data->sg = scatter_list
    │
    ▼
[Pre-DMA Mapping]  ← host driver يعمل pre_req() لو supported
    │   dma_map_sg(data->sg, ...)
    │   data->host_cookie = DMA_COOKIE
    ▼
[Submission]
    │
    ├── mmc_wait_for_req(host, mrq)     ← synchronous path
    │       أو
    └── mmc_start_request(host, mrq)    ← async path
    │
    ▼
[Execution — Host Driver]
    │
    ├── (optional) send sbc command
    ├── send main cmd → hardware register
    ├── transfer data via DMA ← mmc_data.sg
    └── (optional) send stop command
    │
    ▼
[Completion]
    │
    ├── cmd->error = 0 (أو errno)
    ├── data->error = 0 (أو errno)
    ├── data->bytes_xfered = actual bytes
    ├── cmd->resp[] filled
    └── mrq->done(mrq) called
          أو complete(&mrq->completion) لو synchronous
    │
    ▼
[Post-DMA Unmap]
    │   dma_unmap_sg(data->sg, ...)
    ▼
[Teardown]
    │
    └── kfree / stack unwind
```

---

### 4. مخطط الـ Call Flow

#### الـ Synchronous Path عبر `mmc_wait_for_req()`

```
block_driver / sdio_driver
  └── mmc_wait_for_req(host, mrq)
        │
        ├── __mmc_start_request(host, mrq)
        │     │
        │     ├── host->ops->request(host, mrq)    ← host driver entry
        │     │     │
        │     │     ├── [hardware: send CMD on bus]
        │     │     │     opcode → CMD line
        │     │     │     arg   → ARG register
        │     │     │
        │     │     ├── [wait for response]
        │     │     │     resp[0..3] ← hardware response register
        │     │     │     cmd->error = check_crc() ? 0 : -EILSEQ
        │     │     │
        │     │     ├── [if ADTC: start DMA]
        │     │     │     sg → DMA engine
        │     │     │     data->bytes_xfered updated
        │     │     │     data->error set
        │     │     │
        │     │     └── [if stop needed]
        │     │           send CMD12 (data->stop)
        │     │
        │     └── complete(&mrq->cmd_completion)   ← لو async cmd done
        │
        ├── wait_for_completion(&mrq->completion)  ← block here
        │
        └── [return to caller with cmd->error / data->error filled]
```

#### الـ Async Path عبر `done` callback

```
caller
  └── mrq->done = my_callback
      mmc_start_request(host, mrq)
            │
            └── [hardware ISR fires]
                  mrq->done(mrq)    ← called in interrupt or tasklet context
                        │
                        └── my_callback(mrq)
                              └── process results
```

#### الـ `mmc_wait_for_cmd()` — wrapper مبسّط

```
mmc_wait_for_cmd(host, cmd, retries)
  └── for each retry:
        mrq = {.cmd = cmd}
        mmc_wait_for_req(host, &mrq)
        if cmd->error == 0: break
  └── return cmd->error
```

---

### 5. استراتيجية الـ Locking

الـ header ده بحد ذاته ما بيعرّفش locks مباشرة، لكن الـ structs اللي بيعرّفها بتندرج تحت locking scheme واسع في الـ MMC core:

#### الـ Locks الأساسية المرتبطة بالـ Structs دي

| Lock | موجود في | بيحمي |
|---|---|---|
| `mmc_host.lock` | `struct mmc_host` | state الـ host، الـ clock، الـ power، وكل الـ fields المشتركة |
| `mmc_host.ios` protection | `mmc_host.lock` | الـ I/O settings (bus width, timing, etc.) |
| `completion` mechanism | `mmc_request.completion` | الـ synchronous wait بين thread الـ caller والـ interrupt handler |

#### مبدأ اللي بيشتغل عليه النظام

```
[Process context — caller thread]          [Interrupt context — host ISR]
        │                                            │
mmc_wait_for_req()                         hardware finishes transfer
        │                                            │
wait_for_completion(&mrq->completion)      mrq->done(mrq) or complete(...)
        │                                            │
        └───────────── synchronization ──────────────┘
              via struct completion (swait-based, lock-free fast path)
```

- **الـ `mmc_request` نفسه ما فيهوش lock** — لأنه owned بـ caller thread حتى لحظة الـ submission، وبعدين owned بالـ hardware/ISR لحد ما الـ completion يتعمله signal.
- **الـ `mmc_data.host_cookie`** بيتعمله set في الـ pre_req() وبيتقرأ في الـ host driver — التنسيق هنا عبر الـ MMC core sequencing مش locks.
- **الـ CQE (Command Queue Engine)** بيضيف complexity إضافية: الـ `tag` field في `mmc_request` بيتعمله assign من queue، والـ `recovery_notifier` callback بيتسمّى من context مختلف — الـ serialization هنا عبر locks في الـ CQE layer مش في الـ struct ده.

#### Lock Ordering (لما بتحتاج تمسك أكتر من lock)

```
mmc_host.lock
    └── (ممنوع تمسك lock تاني جوّه ده إلا locks الـ completion)

completion.wait (swait_queue_head)
    └── يتمسك آخر حاجة، وبيتحرر تلقائياً مع complete()
```

**قاعدة مهمة:** الـ `mmc_request` بياخد معناه من الـ ownership model — لما بتبعت الـ request للـ host، ما تلمسش أي field فيه إلا بعد ما الـ completion يرجع أو الـ `done` callback يتعمله invoke. ده implicit ownership transfer مش guarded بـ lock عشان يتجنب overhead.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### الـ Exported Functions

| Function | Signature (مختصرة) | الغرض |
|---|---|---|
| `mmc_wait_for_req` | `void (host, mrq)` | بيبعت request وبيستنى على إنهاءه synchronously |
| `mmc_wait_for_cmd` | `int (host, cmd, retries)` | بيبعت command واحد وبيستنى على الـ response |
| `mmc_hw_reset` | `int (card)` | بيعمل hardware reset للكارت |
| `mmc_sw_reset` | `int (card)` | بيعمل software reset للكارت |
| `mmc_set_data_timeout` | `void (data, card)` | بيحسب ويضبط timeout لـ data transfer |

#### الـ Helper Macros

| Macro | الغرض |
|---|---|
| `mmc_resp_type(cmd)` | بيرجع نوع الـ response بتاع الـ command |
| `mmc_spi_resp_type(cmd)` | بيرجع نوع الـ SPI response |
| `mmc_cmd_type(cmd)` | بيرجع نوع الـ command (AC, ADTC, BC, BCR) |

---

### المجموعة الأولى: Request Submission & Synchronous Wait

دي المجموعة الأهم — بتتعامل مع إرسال الـ MMC requests للـ host controller والانتظار على إنهاءها. الـ MMC core بيشتغل على مبدأ الـ `mmc_request` كوحدة عمل أساسية، وفيها optional sbc (SET_BLOCK_COUNT)، command، data، و stop command.

---

#### `mmc_wait_for_req`

```c
void mmc_wait_for_req(struct mmc_host *host, struct mmc_request *mrq);
```

**الـ function دي بتعمل إيه:**
بتبعت الـ `mrq` للـ host controller وبتستنى synchronously على الـ completion. بتستخدم `mrq->completion` كـ kernel completion object وبتعمل `wait_for_completion()` بعد ما تبعت الـ request للـ host عن طريق `__mmc_start_request()`. الـ function مش بترجع غير لما الـ host يشتغل خلص وينادي الـ `mrq->done` callback.

**الـ Parameters:**
- `host` — الـ `struct mmc_host *` بتاع الـ controller اللي هيبعت عليه الـ request
- `mrq` — الـ `struct mmc_request *` اللي فيه الـ command/data/stop chain كاملة

**الـ Return value:**
`void` — الـ error بيتحط في `mrq->cmd->error` و `mrq->data->error`

**Key details:**
- بتحتاج `mmc_host` يكون claimed قبل الاستخدام (عن طريق `mmc_get_card()` أو `mmc_claim_host()`)
- الـ `mrq->completion` لازم يتعمل initialize قبل الاستدعاء — الـ core بيعمل `init_completion()` تلقائياً في `mmc_pre_req()`
- لو `cap_cmd_during_tfr` مضبوطة في الـ mrq، ممكن يتبعتوا commands تانية أثناء الـ data transfer
- الـ function دي blocking — بتشغل في kernel thread context، مش في interrupt context

**Pseudocode flow:**

```
mmc_wait_for_req(host, mrq):
    init_completion(&mrq->completion)
    __mmc_start_request(host, mrq)   // kicks off to host->ops->request()
    wait_for_completion(&mrq->completion)
    // host driver calls mmc_request_done() when hardware finishes
    // which calls complete(&mrq->completion)
    return
```

---

#### `mmc_wait_for_cmd`

```c
int mmc_wait_for_cmd(struct mmc_host *host, struct mmc_command *cmd,
                     int retries);
```

**الـ function دي بتعمل إيه:**
Wrapper مبسطة فوق `mmc_wait_for_req` لإرسال command واحد بدون data. بتعمل `struct mmc_request` على الـ stack، بتربط فيه الـ command، وبتبعته. بتعيد `cmd->error` في الآخر. مفيدة جداً لـ control commands زي CMD0, CMD2, CMD3.

**الـ Parameters:**
- `host` — الـ `struct mmc_host *` اللي هيبعت عليه
- `cmd` — الـ `struct mmc_command *` المجهز كامل مع الـ opcode/arg/flags
- `retries` — عدد مرات إعادة المحاولة لو الـ command فشل (بيتحط في `cmd->retries`)

**الـ Return value:**
`int` — صفر لو نجح، أو errno سالب (`-ETIMEDOUT`, `-EILSEQ`, `-EINVAL`, `-ENOMEDIUM`)

**Key details:**
- الـ host لازم يكون claimed
- بتعمل clear لـ `cmd->error` قبل الإرسال
- بتستخدم `mmc_wait_for_req()` داخلياً — مش بيعمل retry loop هو نفسه بشكل مباشر؛ الـ retries field بيتحط في الـ cmd ويتعامل معاه على مستوى الـ host driver أو الـ core

**Pseudocode flow:**

```c
mmc_wait_for_cmd(host, cmd, retries):
    struct mmc_request mrq = {0}
    mrq.cmd = cmd
    cmd->data = NULL
    cmd->retries = retries
    mmc_wait_for_req(host, &mrq)
    return cmd->error
```

---

### المجموعة التانية: Reset Operations

دي مجموعة الـ reset — بتتعامل مع استعادة الكارت لو كان في حالة error أو hang. الفرق بين الاتنين مهم جداً في الـ embedded systems.

---

#### `mmc_hw_reset`

```c
int mmc_hw_reset(struct mmc_card *card);
```

**الـ function دي بتعمل إيه:**
بتعمل full hardware reset للكارت. بالنسبة لـ eMMC بتستخدم إما الـ RST_n pin (لو مدعوم ومفعّل في الـ EXT_CSD) أو بتعمل power cycle كامل. بالنسبة لـ SD بتعمل re-initialization sequence. الـ function دي destructive — بتمسح كل الـ state بتاع الكارت.

**الـ Parameters:**
- `card` — الـ `struct mmc_card *` اللي محتاج reset

**الـ Return value:**
`int` — صفر لو نجح الـ reset وإعادة الـ initialization، أو errno سالب لو فشل

**Key details:**
- بتعمل notify للـ block layer إنه في reset جاري (عن طريق `mmc_blk_reset()` callbacks)
- بعد الـ hardware reset، الكارت بيكون زي ما اتشغل لأول مرة — كل الـ settings (bus width, speed mode, partitions) لازم تتعمل إعادة تهيئة
- لو الـ host driver عنده `hw_reset` op، بيتنادى مباشرة
- بتحتاج الـ host يكون claimed

**Caller context:**
بتتنادى من الـ error recovery paths في `mmc_blk_reset()` و `mmc_blk_err_check()` — مش من interrupt context أبداً.

---

#### `mmc_sw_reset`

```c
int mmc_sw_reset(struct mmc_card *card);
```

**الـ function دي بتعمل إيه:**
بتعمل software reset للكارت عن طريق إرسال CMD0 (GO_IDLE_STATE) أو CMD52 لـ SDIO cards. أخف بكتير من الـ hardware reset لأن مش بيحتاج power cycle. بتستخدمها لو الكارت اتشال من حالة timeout بس ما محتاجش full re-init.

**الـ Parameters:**
- `card` — الـ `struct mmc_card *` المحتاج reset

**الـ Return value:**
`int` — صفر لو نجح، أو errno سالب

**Key details:**
- أسرع بكتير من `mmc_hw_reset` لكن ممكن ميكونش كافي في كل الحالات
- الـ bus configuration (width, timing) بتفضل نفسها — مش بيحتاج re-negotiation كامل
- لو الـ host driver عنده `sw_reset` op خاصة بيه، بيتنادى

---

### المجموعة التالتة: Data Timeout Configuration

---

#### `mmc_set_data_timeout`

```c
void mmc_set_data_timeout(struct mmc_data *data, const struct mmc_card *card);
```

**الـ function دي بتعمل إيه:**
بتحسب وبتضبط الـ `timeout_ns` و `timeout_clks` في الـ `struct mmc_data` بناءً على خصائص الكارت المحددة في الـ CSD register. الـ MMC standard بيحدد timeout بالـ nanoseconds و clock cycles معاً — الـ host driver بيختار الأنسب.

**الـ Parameters:**
- `data` — الـ `struct mmc_data *` اللي محتاج تضبط فيه الـ timeout
- `card` — الـ `const struct mmc_card *` اللي بناءً على specs بتاعته بيتحسب الـ timeout

**الـ Return value:**
`void`

**Key details:**
- الحساب بيفرق بين read و write timeouts:
  - **Read timeout**: بيتحسب من `CSD.TAAC` (time-dependent) + `CSD.NSAC` (clock-dependent) مضروبين في factor ثابت
  - **Write timeout**: نفس الحساب بس مضروب في `CSD.R2W_FACTOR` (عادة 4× أو 8× أبطأ)
- لـ eMMC بيستخدم `EXT_CSD.SA_TIMEOUT` للـ sanitize operations
- الـ function مهمة جداً — timeout صغير أوي بيسبب false errors، وكبير أوي بيخلي الـ error recovery بطيء

**Pseudocode flow:**

```c
mmc_set_data_timeout(data, card):
    // Read base timeout from CSD
    mult = 10 << card->csd.taac_ns  // time value multiplier
    ns = (taac_mant[card->csd.taac_mant] * mult + 9) / 10
    ns += card->csd.nsac * 100  // add clock cycles contribution

    // Apply safety margin (x10 per spec)
    ns *= 10

    if (data->flags & MMC_DATA_WRITE):
        ns <<= card->csd.r2w_factor  // writes are slower

    data->timeout_ns = ns
    data->timeout_clks = 0

    // Cap at hardware maximum
    if (ns > 80 * NSEC_PER_MSEC):
        data->timeout_ns = 80 * NSEC_PER_MSEC
        data->timeout_clks = 0
```

---

### المجموعة الرابعة: Helper Macros

---

#### `mmc_resp_type(cmd)`

```c
#define mmc_resp_type(cmd) \
    ((cmd)->flags & (MMC_RSP_PRESENT|MMC_RSP_136|MMC_RSP_CRC|MMC_RSP_BUSY|MMC_RSP_OPCODE))
```

**بيعمل إيه:**
بيعمل mask على الـ `cmd->flags` ويرجع الـ bits المتعلقة بنوع الـ response فقط (R1, R1B, R2, R3, etc.). بيُستخدم للمقارنة السريعة.

**مثال:**

```c
if (mmc_resp_type(cmd) == MMC_RSP_R1B) {
    /* wait for busy line to go low */
}
```

---

#### `mmc_spi_resp_type(cmd)`

```c
#define mmc_spi_resp_type(cmd) \
    ((cmd)->flags & \
     (MMC_RSP_SPI_S1|MMC_RSP_SPI_BUSY|MMC_RSP_SPI_S2|MMC_RSP_SPI_B4))
```

**بيعمل إيه:**
نفس `mmc_resp_type` بس للـ SPI mode — بيعمل mask على الـ SPI-specific response bits بدل native MMC bits. الـ SPI protocol عنده response format مختلف تماماً — byte-based مش bit-based.

---

#### `mmc_cmd_type(cmd)`

```c
#define mmc_cmd_type(cmd) ((cmd)->flags & MMC_CMD_MASK)
```

**بيعمل إيه:**
بيرجع الـ command type من الـ flags:

| Value | Constant | معناه |
|---|---|---|
| `0 << 5` | `MMC_CMD_AC` | Addressed Command — no data |
| `1 << 5` | `MMC_CMD_ADTC` | Addressed Data Transfer Command |
| `2 << 5` | `MMC_CMD_BC` | Broadcast Command — no response |
| `3 << 5` | `MMC_CMD_BCR` | Broadcast Command with Response |

---

### الـ Data Structures المرتبطة بالـ Functions

#### `struct mmc_command`

الـ struct الأساسي لأي MMC command. كل عملية على الكارت بدأت من هنا.

```c
struct mmc_command {
    u32          opcode;        /* CMD number (0-63) */
    u32          arg;           /* 32-bit argument */
    u32          resp[4];       /* response buffer (max 136 bits = 4 words) */
    unsigned int flags;         /* response type + command type */
    unsigned int retries;       /* max retry count on error */
    int          error;         /* result: 0 or -errno */
    unsigned int busy_timeout;  /* ms to wait for busy line */
    struct mmc_data    *data;   /* associated data (NULL for control cmds) */
    struct mmc_request *mrq;    /* back-pointer to parent request */
    struct uhs2_command *uhs2_cmd; /* UHS-II extension */
    bool has_ext_addr;          /* SDUC extended address flag */
    u8   ext_addr;              /* SDUC extended address value */
};
```

#### `struct mmc_data`

بتوصف الـ data transfer المرتبطة بـ ADTC command.

```c
struct mmc_data {
    unsigned int timeout_ns;    /* timeout in nanoseconds — set by mmc_set_data_timeout() */
    unsigned int timeout_clks;  /* timeout in clock cycles */
    unsigned int blksz;         /* block size in bytes */
    unsigned int blocks;        /* number of blocks to transfer */
    unsigned int blk_addr;      /* starting block address */
    int          error;         /* result after transfer */
    unsigned int flags;         /* MMC_DATA_READ/WRITE + CQE flags */
    unsigned int bytes_xfered;  /* actual bytes transferred */
    struct mmc_command *stop;   /* optional CMD12 stop command */
    struct mmc_request *mrq;    /* back-pointer */
    unsigned int sg_len;        /* scatter-gather list length */
    int          sg_count;      /* number of mapped DMA sg entries */
    struct scatterlist *sg;     /* DMA scatter-gather list */
    s32          host_cookie;   /* opaque cookie for host DMA pre-mapping */
};
```

#### `struct mmc_request`

الـ container الرئيسي — بيجمع كل عناصر العملية الواحدة.

```c
struct mmc_request {
    struct mmc_command *sbc;       /* optional CMD23 SET_BLOCK_COUNT */
    struct mmc_command *cmd;       /* main command */
    struct mmc_data    *data;      /* data phase (optional) */
    struct mmc_command *stop;      /* stop command CMD12 (optional) */
    struct completion  completion; /* used by mmc_wait_for_req() */
    struct completion  cmd_completion;
    void (*done)(struct mmc_request *);              /* called by host when done */
    void (*recovery_notifier)(struct mmc_request *); /* CQE error recovery */
    struct mmc_host    *host;
    bool               cap_cmd_during_tfr; /* allow cmds during data */
    int                tag;                /* CQE tag */
    struct uhs2_command uhs2_cmd;          /* embedded UHS-II command */
};
```

---

### الـ Error Codes المستخدمة

| Error | السبب |
|---|---|
| `-ETIMEDOUT` | الكارت ما ردش في الوقت المحدد |
| `-EILSEQ` | CRC error أو opcode mismatch في الـ response |
| `-EINVAL` | الـ request مش ممكن ينفذ (hardware limitation) |
| `-ENOMEDIUM` | الـ slot فاضي — مفيش كارت |

---

### الـ UHS-II Support

الـ structs دي بتدعم الجيل الجديد من بروتوكول SD:

#### `struct uhs2_command`

```c
struct uhs2_command {
    u16    header;                             /* IOADR or other header fields */
    u16    arg;                                /* command argument */
    __be32 payload[UHS2_MAX_PAYLOAD_LEN];      /* big-endian payload (max 8 bytes) */
    u8     payload_len;
    u8     packet_len;
    u8     tmode_half_duplex;                  /* half-duplex T-mode flag */
    u8     uhs2_resp[UHS2_MAX_RESP_LEN];       /* native UHS-II response buffer */
    u8     uhs2_resp_len;
};
```

الـ UHS-II بيستخدم packet-based protocol فوق 2-wire differential interface (LVDS) بدل الـ parallel bus التقليدي. الـ `payload` محدود بـ 8 bytes و الـ response بـ 20 bytes.

---

### Flow الـ Request الكامل

```
Caller (block driver / mmc_test)
    │
    ▼
mmc_wait_for_cmd() or mmc_wait_for_req()
    │
    ├── init_completion(&mrq->completion)
    │
    ▼
__mmc_start_request(host, mrq)
    │
    ├── mmc_mrq_pr_debug()        // tracing
    ├── mmc_retune_recheck()      // check if retuning needed
    │
    ▼
host->ops->request(host, mrq)    // host driver takes over
    │
    ▼  [hardware DMA / IRQ]
    │
    ▼
mmc_request_done(host, mrq)      // called from IRQ/tasklet
    │
    ├── cmd->error set
    ├── data->error set
    ├── mrq->done(mrq)            // optional callback
    └── complete(&mrq->completion)
    │
    ▼
mmc_wait_for_req() returns
    │
    ▼
Caller reads cmd->error / data->error
```
## Phase 5: دليل الـ Debugging الشامل

> **الـ subsystem المستهدف:** `linux/mmc/core.h` — الـ MMC/SD core layer، يشمل `mmc_command`، `mmc_data`، `mmc_request`، و `uhs2_command`.

---

### Software Level

#### 1. debugfs Entries

الـ MMC subsystem بيعمل expose لمعلومات تشخيصية كتير تحت `/sys/kernel/debug/mmc*/`:

| المسار | المحتوى |
|--------|---------|
| `/sys/kernel/debug/mmc0/ios` | الـ current IOS state: clock, bus_width, timing, signal_voltage |
| `/sys/kernel/debug/mmc0/state` | حالة الـ host: card present, claimed, bus dead |
| `/sys/kernel/debug/mmc0/err_stats` | عدادات الـ errors لكل نوع من `mmc_err_stat` |
| `/sys/kernel/debug/mmc0/caps` | الـ host capabilities flags |
| `/sys/kernel/debug/mmc0/caps2` | الـ extended capabilities |

```bash
# قراءة IOS الحالي — مهم لو بتشك في timing أو voltage
cat /sys/kernel/debug/mmc0/ios

# مثال الـ output:
# clock:          200000000 Hz
# actual clock:   200000000 Hz
# vdd:            21 (3.3 ~ 3.4 V)
# bus mode:       push-pull
# chip select:    don't care
# power mode:     on
# bus width:      4 bits
# timing spec:    mmc HS200
# signal voltage: 1.80 V
# driver type:    B

# قراءة error statistics
cat /sys/kernel/debug/mmc0/err_stats
# الـ output بيبين counters زي:
# MMC_ERR_CMD_TIMEOUT: 3
# MMC_ERR_DAT_CRC: 1
```

```bash
# dump كل debugfs entries للـ MMC
ls /sys/kernel/debug/mmc*/
find /sys/kernel/debug/mmc0/ -type f | xargs -I{} sh -c 'echo "=== {} ==="; cat {} 2>/dev/null'
```

---

#### 2. sysfs Entries

| المسار | المحتوى |
|--------|---------|
| `/sys/bus/mmc/devices/mmc0:0001/` | device attributes للـ card |
| `/sys/bus/mmc/devices/mmc0:0001/type` | نوع الكارت: MMC, SD, SDIO |
| `/sys/bus/mmc/devices/mmc0:0001/cid` | الـ Card Identification Register |
| `/sys/bus/mmc/devices/mmc0:0001/csd` | الـ Card Specific Data |
| `/sys/bus/mmc/devices/mmc0:0001/scr` | الـ SD Configuration Register (SD فقط) |
| `/sys/bus/mmc/devices/mmc0:0001/date` | تاريخ تصنيع الكارت |
| `/sys/bus/mmc/devices/mmc0:0001/preferred_erase_size` | أفضل حجم للـ erase |
| `/sys/class/mmc_host/mmc0/` | host attributes |

```bash
# معرفة نوع الكارت وتاريخه
cat /sys/bus/mmc/devices/mmc0:*/type
cat /sys/bus/mmc/devices/mmc0:*/date
cat /sys/bus/mmc/devices/mmc0:*/cid

# الـ CID بيكون hex — أول بايتين هم الـ manufacturer ID
# مثال: 0x150100 = Samsung
cat /sys/bus/mmc/devices/mmc0:0001/cid
# 150100524f535058315810b8015700

# قراءة الـ erase size
cat /sys/bus/mmc/devices/mmc0:*/preferred_erase_size
```

---

#### 3. ftrace — Tracepoints والـ Events

الـ MMC subsystem عنده tracepoints في `drivers/mmc/core/`:

```bash
# شوف كل الـ available MMC events
ls /sys/kernel/tracing/events/mmc/

# الـ events الأساسية:
# mmc_request_start   — لما mmc_request بييجي للـ host
# mmc_request_done    — لما mmc_request يخلص
# mmc_cmd_rw_start    — بداية command
# mmc_cmd_rw_end      — نهاية command مع الـ response
# mmc_blk_rw_start    — بداية block read/write
# mmc_blk_rw_end      — نهاية block read/write

# تشغيل تتبع الـ requests الكاملة
echo 1 > /sys/kernel/tracing/events/mmc/enable
echo 1 > /sys/kernel/tracing/tracing_on
# شغّل العملية اللي بتعمل مشكلة
echo 0 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace

# تتبع command + data فقط (بدون block layer)
echo 1 > /sys/kernel/tracing/events/mmc/mmc_request_start/enable
echo 1 > /sys/kernel/tracing/events/mmc/mmc_request_done/enable

# مثال output مفيد:
# kworker/0:1-45   [000] ....  123.456: mmc_request_start: mmc0: start struct mmc_request[ffff...]:
#   cmd_opcode=18 cmd_arg=0x10000 cmd_flags=0x25 cmd_retries=0 stop_opcode=12
#   sbc_opcode=23 sbc_arg=0x10 data_blksz=512 data_blocks=16 data_flags=0x200
```

```bash
# تتبع function calls داخل MMC core
echo 'mmc_wait_for_req' > /sys/kernel/tracing/set_ftrace_filter
echo 'mmc_wait_for_cmd' >> /sys/kernel/tracing/set_ftrace_filter
echo function > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/tracing_on
```

---

#### 4. printk / Dynamic Debug

```bash
# تفعيل dynamic debug للـ MMC core كله
echo 'module mmc_core +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug للـ block layer بس
echo 'module mmc_block +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل مع file محدد
echo 'file core.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file block.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل مع line number للـ command error handling
echo 'file core.c line 1234 +p' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ active dynamic debug entries
cat /sys/kernel/debug/dynamic_debug/control | grep mmc

# عن طريق kernel cmdline لو محتاج debug من البداية
# أضف في /etc/default/grub:
# GRUB_CMDLINE_LINUX="... dyndbg='module mmc_core +p; module mmc_block +p'"
```

```bash
# رفع لـ printk level عشان تشوف debug messages
echo 8 > /proc/sys/kernel/printk
# أو
dmesg -n 8

# تابع الـ messages live
dmesg -w | grep -i mmc
```

---

#### 5. Kernel Config Options للـ Debugging

| Config | الغرض |
|--------|--------|
| `CONFIG_MMC_DEBUG` | تفعيل debug messages في الـ MMC core |
| `CONFIG_MMC_UNSAFE_RESUME` | مفيد لـ debug مشاكل الـ resume |
| `CONFIG_FAIL_MMC_REQUEST` | fault injection في الـ MMC requests |
| `CONFIG_MMC_CRYPTO` | تفعيل الـ inline encryption support |
| `CONFIG_DEBUG_FS` | لازم يكون enabled عشان debugfs يشتغل |
| `CONFIG_DYNAMIC_DEBUG` | لازم يكون enabled لـ dynamic debug |
| `CONFIG_TRACING` | لازم يكون enabled لـ ftrace |
| `CONFIG_MMC_BLOCK_BOUNCE` | bounce buffer — ممكن يخبّي مشاكل DMA |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'CONFIG_MMC|CONFIG_FAIL_MMC'

# أو لو الـ config موجود كـ file
grep -E 'CONFIG_MMC|CONFIG_FAIL_MMC' /boot/config-$(uname -r)
```

---

#### 6. Fault Injection — FAIL_MMC_REQUEST

الـ kernel بيوفر fault injection mechanism لـ MMC requests عن طريق `CONFIG_FAIL_MMC_REQUEST`:

```bash
# تحقق إن الـ fault injection موجود
ls /sys/kernel/debug/mmc0/fail_mmc_request/

# إعداد الـ fault injection: يفشل كل 100 request
echo 100 > /sys/kernel/debug/mmc0/fail_mmc_request/interval
echo 1   > /sys/kernel/debug/mmc0/fail_mmc_request/probability
echo 1   > /sys/kernel/debug/mmc0/fail_mmc_request/times
echo 1   > /sys/kernel/debug/mmc0/fail_mmc_request/task-filter

# مشاهدة تأثير الـ fault injection
dmesg -w | grep -E 'mmc|blk_update_request'
```

```bash
# أدوات عامة مفيدة مع MMC
mmc extcsd read /dev/mmcblk0   # قراءة Extended CSD لـ eMMC
mmc status get /dev/mmcblk0    # حالة الكارت
mmc csd read /dev/mmcblk0      # قراءة CSD register
```

---

#### 7. جدول رسائل الـ Errors الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---------------------|--------|------|
| `mmc0: error -110 whilst initialising SD card` | ETIMEDOUT — الكارت ما ردش على CMD أو ACMD | تحقق من التوصيلات، الـ VCC، ومستوى الجهد |
| `mmc0: error -84 whilst initialising MMC card` | EILSEQ — CRC error في الـ response | مشكلة في الـ signal integrity، خفّض clock frequency |
| `mmc0: controller never released inhibit bit(s)` | الـ host controller stuck | reset الـ host، تحقق من الـ power supply |
| `mmc0: Timeout waiting for hardware interrupt` | الـ hardware ما رفعش interrupt في الوقت | تحقق من الـ IRQ routing والـ clock |
| `mmc0: card reports max AC 0mA` | مشكلة في قراءة CID/CSD | كارت تالف أو مشكلة في الـ voltage switch |
| `mmcblk0: error -110 transferring data` | timeout أثناء data transfer | `mmc_data.timeout_ns` انتهى قبل الـ transfer |
| `mmc0: CMD23 failed, error -5` | خطأ في SET_BLOCK_COUNT (CMD23) | الكارت مش supportted لـ reliable write |
| `mmc0: Tuning procedure failed, falling back` | فشل execute_tuning للـ HS200/HS400 | مشكلة في الـ clock phase، جرب قيم phase مختلفة |
| `mmc0: card stuck in programming state` | الكارت busy لفترة طويلة في R1B | `busy_timeout` انتهى، ممكن كارت بطيء أو تالف |
| `mmc0: cache flush error` | خطأ في تفريغ الـ write cache | تحقق من حالة الكارت بـ `mmc extcsd read` |
| `mmc0: CQE error, status 0x...` | خطأ في الـ Command Queue Engine | تحقق من `MMC_ERR_CMDQ_*` counters في debugfs |
| `mmc0: ADMA error` | خطأ في الـ Advanced DMA descriptor | مشكلة في الـ DMA mapping أو الـ sg list |

---

#### 8. أماكن استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في mmc_wait_for_req — للتحقق من صحة الـ mrq */
void mmc_wait_for_req(struct mmc_host *host, struct mmc_request *mrq)
{
    /* تحقق إن الـ host مش NULL */
    WARN_ON(!host);

    /* تحقق إن في cmd على الأقل */
    WARN_ON(!mrq->cmd && !mrq->data);

    /* تحقق إن الـ completion initialized */
    WARN_ON(!mrq->done && !mrq->completion.done);
    /* ... */
}

/* في mmc_wait_for_cmd — للتحقق من الـ error بعد الـ response */
int mmc_wait_for_cmd(struct mmc_host *host, struct mmc_command *cmd, int retries)
{
    /* dump_stack لو حصل error غير متوقع */
    if (cmd->error && cmd->error != -ETIMEDOUT) {
        dev_warn(mmc_dev(host), "unexpected error %d\n", cmd->error);
        dump_stack();  /* اعرف مين طلب الـ cmd ده */
    }
    /* ... */
}

/* في mmc_request completion callback */
static void mmc_request_done_notifier(struct mmc_request *mrq)
{
    /* تحقق إن الـ data error واضح */
    if (mrq->data) {
        WARN_ON(mrq->data->error && mrq->data->bytes_xfered > 0);
        /* ده مش منطقي: في error لكن في bytes اتنقلت */
    }
}
```

---

### Hardware Level

#### 1. التحقق من أن الـ Hardware State يطابق الـ Kernel State

```bash
# قارن الـ IOS بين ما يقوله الـ kernel وما هو على الـ hardware
cat /sys/kernel/debug/mmc0/ios
# لو الـ clock = 200MHz وأنت شايف على الـ oscilloscope 50MHz فيه مشكلة

# تحقق من voltage switching
# الـ kernel بيسجل:
# signal voltage: 1.80 V  →  يعني عمل voltage switch لـ 1.8V
# لو الـ multimeter بيقرأ 3.3V على VCCQ فيه مشكلة

# تحقق من bus width
cat /sys/kernel/debug/mmc0/ios | grep width
# bus width: 4 bits  →  يعني DAT0-DAT3 يلزمهم يكونوا مستخدمين

# تحقق من card detection
cat /sys/class/mmc_host/mmc0/mmc0\:*/type 2>/dev/null || echo "no card"
```

---

#### 2. Register Dump Techniques

```bash
# تحقق من الـ base address من الـ device tree أو /proc/iomem
cat /proc/iomem | grep -i mmc
# مثال output:
# fe340000-fe34ffff : fe340000.mmc

# قراءة registers بـ devmem2 (لازم تثبّته)
# devmem2 ADDRESS [b|h|w] [VALUE]
devmem2 0xfe340000 w    # قراءة أول register (host version / capabilities)
devmem2 0xfe340024 w    # Present State Register في SDHCI
devmem2 0xfe340028 w    # Host Control 1 Register
devmem2 0xfe34002C w    # Power Control Register
devmem2 0xfe340030 w    # Clock Control Register
devmem2 0xfe340036 h    # Interrupt Status Register (16-bit)

# SDHCI standard registers (offset من الـ base address)
# 0x00-0x0F: SDMA System Address / Block Size / Block Count
# 0x10-0x17: Argument 1 / Transfer Mode / Command
# 0x18-0x27: Response Registers (resp[0]-resp[3])
# 0x20-0x23: Buffer Data Port
# 0x24:      Present State Register  ← أهم register للـ debug
# 0x28:      Host Control 1
# 0x2E:      Clock Control
# 0x30:      Timeout Control
# 0x32:      Software Reset
# 0x34:      Normal Interrupt Status
# 0x36:      Error Interrupt Status

# dump كامل لـ SDHCI registers بـ /dev/mem
python3 - << 'EOF'
import mmap, struct
base = 0xfe340000
with open('/dev/mem', 'rb') as f:
    m = mmap.mmap(f.fileno(), 0x100, offset=base)
    for off in range(0, 0x80, 4):
        val = struct.unpack('<I', m[off:off+4])[0]
        print(f'  0x{off:02X}: 0x{val:08X}')
    m.close()
EOF
```

---

#### 3. Logic Analyzer / Oscilloscope

**نقاط القياس المهمة على الـ SD/MMC bus:**

```
SD Card Pinout (standard):
┌─────────────────────────────────────┐
│  Pin 1: DAT3/CD  ← card detect      │
│  Pin 2: CMD      ← command line      │
│  Pin 3: VSS1     ← GND              │
│  Pin 4: VDD      ← 3.3V power       │
│  Pin 5: CLK      ← clock            │
│  Pin 6: VSS2     ← GND              │
│  Pin 7: DAT0     ← data bit 0       │
│  Pin 8: DAT1     ← data bit 1       │
│  Pin 9: DAT2     ← data bit 2       │
└─────────────────────────────────────┘
```

**إعدادات الـ Logic Analyzer:**

| الإعداد | القيمة المناسبة |
|---------|----------------|
| Sample rate | 10x الـ max clock (مثلاً 2GHz لـ HS200) |
| Voltage threshold | 0.9V لـ 1.8V signaling، 1.65V لـ 3.3V |
| Trigger | على rising edge لـ CLK أو falling edge لـ CMD |
| Capture depth | 10K-100K samples كافية لـ single command |

```
نموذج CMD line لـ CMD18 (READ_MULTIPLE_BLOCK):
 Start  Dir    Opcode     Argument      CRC    End
  ┌─┐  ┌──┐  ┌────────┐  ┌──────────┐  ┌───┐  ┌─┐
  │0│  │ 1│  │010010  │  │00008000  │  │CRC│  │1│
  └─┘  └──┘  └────────┘  └──────────┘  └───┘  └─┘
       Host

نموذج Response R1:
  ┌─┐  ┌──┐  ┌────────┐  ┌──────────┐  ┌───┐  ┌─┐
  │0│  │ 0│  │010010  │  │Card State│  │CRC│  │1│
  └─┘  └──┘  └────────┘  └──────────┘  └───┘  └─┘
       Card
```

**نقاط المراقبة:**
- الـ CMD line يلزم يبقى HIGH في الـ idle state
- الـ CLK يوقف في end of transfer (host بيوقفه)
- الـ DAT0 يبقى LOW أثناء busy state بعد Write
- الـ voltage switch: لازم الـ CLK يوقف قبل الـ switch

---

#### 4. Hardware Issues الشائعة وأنماط الـ Kernel Log

| المشكلة | الأعراض على الـ Log | التحقق |
|---------|-------------------|--------|
| **ضعف الـ power supply** | timeouts عشوائية، `error -110` | قِس الـ VCC تحت load، يلزم يكون stable ±5% |
| **مشكلة signal integrity** | `error -84` (CRC errors)، tuning failure | oscilloscope على CLK+CMD، ابحث عن ringing أو noise |
| **voltage switch failure** | بيقف عند init على `CMD11` | multimeter على VCCQ، لازم ينزل من 3.3V لـ 1.8V |
| **missing pull-up resistors** | `card never left idle state` | تأكد من pull-up على CMD وDAT lines |
| **كارت overheating** | errors تزيد مع الوقت | thermometer، الـ eMMC يشتغل -25°C لـ 85°C |
| **DMA misalignment** | `ADMA error`، kernel warning | تحقق من `sg->offset` و `sg->length` تكون aligned |
| **clock too fast** | tuning failure، intermittent errors | خفّض الـ max-frequency في DT |
| **card not resetting** | `mmc0: error -110 whilst initialising` | تحقق من RST_n pin للـ eMMC |

---

#### 5. Device Tree Debugging

```bash
# عرض الـ DT المُحمّل للـ MMC controller
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 50 'mmc@\|sdhci@'

# أو بطريقة أبسط
cat /sys/firmware/devicetree/base/soc/mmc@fe340000/compatible 2>/dev/null
ls /sys/firmware/devicetree/base/soc/mmc@fe340000/

# مقارنة الـ DT properties بما يتوقعه الـ driver
# مثال: تحقق من الـ clock frequency
cat /sys/firmware/devicetree/base/soc/mmc@fe340000/max-frequency | xxd
# القيمة بتكون big-endian 32-bit
# مثال: 00 BC 61 4E = 12345678 Hz

# تحقق من bus-width
cat /sys/firmware/devicetree/base/soc/mmc@fe340000/bus-width | xxd
# 00 00 00 04 = 4-bit bus

# تحقق من الـ voltage regulators
ls /sys/firmware/devicetree/base/soc/mmc@fe340000/ | grep vmmc
```

**الـ DT properties المهمة لـ MMC core:**

```dts
/* مثال DT node صحيح لـ eMMC على Raspberry Pi 4 */
mmc0: mmc@fe340000 {
    compatible = "brcm,bcm2711-emmc2";
    reg = <0x0 0xfe340000 0x0 0x100>;
    clocks = <&clocks BCM2711_CLOCK_EMMC2>;
    interrupts = <GIC_SPI 126 IRQ_TYPE_LEVEL_HIGH>;

    /* هذه القيم يلزم تطابق الـ hardware */
    bus-width = <8>;                    /* 1, 4, أو 8 */
    max-frequency = <200000000>;        /* 200MHz لـ HS400 */
    non-removable;                      /* للـ eMMC المُثبّت */
    mmc-hs400-1_8v;                    /* يدعم HS400 بـ 1.8V */
    vmmc-supply = <&reg_3v3>;
    vqmmc-supply = <&reg_1v8>;
};
```

```bash
# مقارنة الـ DT مع ما يقوله الـ kernel عن الـ capabilities
diff <(cat /sys/kernel/debug/mmc0/caps) <(dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep mmc-hs)

# تحقق من الـ regulator state
cat /sys/class/regulator/regulator.*/name
cat /sys/class/regulator/regulator.*/microvolts
```

---

### Practical Commands

#### أوامر جاهزة للنسخ

```bash
# ===== Quick Health Check للـ MMC =====
echo "=== MMC IOS ===" && cat /sys/kernel/debug/mmc0/ios
echo "=== MMC State ===" && cat /sys/kernel/debug/mmc0/state
echo "=== Error Stats ===" && cat /sys/kernel/debug/mmc0/err_stats
echo "=== Card Type ===" && cat /sys/bus/mmc/devices/mmc0:*/type 2>/dev/null
echo "=== Recent MMC Messages ===" && dmesg | tail -50 | grep -i mmc
```

```bash
# ===== تتبع MMC request lifecycle =====
# 1. صفّر الـ trace buffer
echo > /sys/kernel/tracing/trace

# 2. فعّل الـ MMC events
echo 1 > /sys/kernel/tracing/events/mmc/enable

# 3. ابدأ الـ tracing
echo 1 > /sys/kernel/tracing/tracing_on

# 4. شغّل العملية (مثلاً read من الـ disk)
dd if=/dev/mmcblk0 of=/dev/null bs=512 count=100

# 5. وقّف الـ tracing
echo 0 > /sys/kernel/tracing/tracing_on

# 6. اقرأ النتائج
cat /sys/kernel/tracing/trace | grep -E 'mmc_request|mmc_cmd'
```

```bash
# ===== تشخيص CRC errors =====
# شوف عدد CRC errors على مدار الوقت
watch -n 1 'grep -E "CRC|TIMEOUT" /sys/kernel/debug/mmc0/err_stats'

# لو الـ CRC errors بتزيد باستمرار → مشكلة signal integrity
# لو بتظهر فجأة ثم توقف → ممكن thermal issue
```

```bash
# ===== تشخيص timeout في mmc_command =====
# فعّل dynamic debug مع timestamps
echo 'module mmc_core +pt' > /sys/kernel/debug/dynamic_debug/control
echo 'module mmc_block +pt' > /sys/kernel/debug/dynamic_debug/control
dmesg -w 2>&1 | grep -E 'mmc|timeout' | head -100
```

```bash
# ===== التحقق من حالة الـ eMMC بالتفصيل =====
# محتاج package: mmc-utils
mmc extcsd read /dev/mmcblk0 | grep -E 'DEVICE_LIFE|PRE_EOL|CACHE'
# مثال output:
# [B1]: EXT_CSD_PRE_EOL_INFO: 0x01             <- Normal (0x01=Normal, 0x02=Warning, 0x03=Urgent)
# [B2]: EXT_CSD_DEVICE_LIFE_TIME_EST_TYP_A: 0x01  <- نسبة الـ wear: 0-10%
# [B3]: EXT_CSD_DEVICE_LIFE_TIME_EST_TYP_B: 0x01
```

```bash
# ===== تتبع mmc_request errors مع stack trace =====
# استخدام kprobe لتتبع الـ errors
echo 'p:mmc_req_err mmc_request_done mrq=%di error=%si:s32' > /sys/kernel/tracing/kprobe_events
echo 1 > /sys/kernel/tracing/events/kprobes/mmc_req_err/enable
cat /sys/kernel/tracing/trace | grep mmc_req_err
```

```bash
# ===== dump الـ mmc_command response =====
# لو محتاج تشوف الـ raw response لكل command
# فعّل kernel debug level
echo 8 > /proc/sys/kernel/printk
echo 'file core.c +pmf' > /sys/kernel/debug/dynamic_debug/control
# الـ +m = module name, +p = print, +f = function name
dmesg -w | grep -A2 'req done'
```

```bash
# ===== اختبار الـ busy_timeout =====
# شوف الـ busy_timeout الحالي لعمليات الـ write
# (من خلال ftrace على mmc_set_data_timeout)
echo 'mmc_set_data_timeout' > /sys/kernel/tracing/set_ftrace_filter
echo function > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/tracing_on
dd if=/dev/zero of=/dev/mmcblk0p1 bs=4M count=1 oflag=direct
cat /sys/kernel/tracing/trace
```

```bash
# ===== تشخيص UHS2 commands (uhs2_command struct) =====
# تحقق من UHS2 capabilities
cat /sys/kernel/debug/mmc0/caps2 | grep -i uhs2
cat /sys/firmware/devicetree/base/soc/mmc@*/ 2>/dev/null | grep uhs2

# لو محتاج trace الـ UHS2 native commands
echo 1 > /sys/kernel/tracing/events/mmc/enable
cat /sys/kernel/tracing/trace | grep -i uhs2
```

```bash
# ===== تشخيص CQE (Command Queue Engine) =====
# تحقق من الـ CQE state
cat /sys/kernel/debug/mmc0/err_stats | grep CMDQ

# مثال output لو في مشكلة:
# MMC_ERR_CMDQ_RED: 5     <- Recovery Error Detected
# MMC_ERR_CMDQ_GCE: 0     <- General Crypto Error
# MMC_ERR_CMDQ_ICCE: 0    <- Invalid Crypto Config Error

# تعطيل CQE مؤقتاً للاختبار (محتاج recompile أو module param)
# بعض الـ drivers بيدعموا:
echo 0 > /sys/bus/mmc/devices/mmc0:0001/cmdq_en 2>/dev/null
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: eMMC مش بيرد على CMD1 في بورد صناعي بيشتغل على RK3562

#### العنوان
**Industrial Gateway على RK3562 — eMMC بيتجمد أثناء الـ initialization بسبب `busy_timeout` غلط**

#### السياق
شركة بتعمل industrial IoT gateway بتستخدم RK3562 مع eMMC 5.1 بـ 32GB من Kingston. البورد بتشتغل على Buildroot Linux. المنتج كان شغال تمام على الـ EVK الأصلي، لما جم يشغلوه على الـ custom PCB لقوا الـ boot بيتعلق كل شوية بعشوائية.

#### المشكلة
الـ kernel بيطلع في الـ dmesg:

```
mmc0: error -110 whilst initialising MMC card
mmc0: CMD1 timeout
```

الـ `-110` ده `ETIMEDOUT`. البورد ممكن تبوت 3 مرات متتالية وتتعلق في الرابعة.

#### التحليل
في `mmc_command`:

```c
unsigned int    busy_timeout;   /* busy detect timeout in ms */
int             error;          /* command error */
```

الـ `CMD1` (SEND_OP_COND) بيرجع `MMC_RSP_R3` — يعني response بدون CRC وبدون opcode echo:

```c
#define MMC_RSP_R3  (MMC_RSP_PRESENT)
```

الـ host driver بيستخدم `busy_timeout` عشان يعرف يستنى قد إيه الكارت يخلص الـ initialization. لو الـ eMMC احتاج وقت أطول من المتوقع عشان الـ power rail يستقر (حالة شائعة في custom PCB بسبب decoupling caps صغيرة)، الـ host بيقطع الانتظار قبل ما الكارت يرد وبيحط `-ETIMEDOUT` في `cmd->error`.

الـ flow:

```
mmc_send_op_cond()
  → mmc_wait_for_cmd(host, cmd, retries)
        → mmc_wait_for_req(host, mrq)
              → wait_for_completion(&mrq->completion)  ← بيستنى هنا
        → لو cmd->error == -ETIMEDOUT → يرجع error
```

الـ `mmc_request` بيحتوي على `completion` و `cmd_completion`:

```c
struct completion   completion;
struct completion   cmd_completion;
```

الـ host ISR بيعمل `complete(&mrq->completion)` لما الـ hardware ينهي أو يـ timeout. لو الـ `busy_timeout` في `mmc_command` كان أقل من وقت الـ power-up الفعلي للـ eMMC، الـ host controller هو اللي بيقطع قبل ما الكارت يرد.

#### الحل

**خطوة 1 — اتأكد من الـ power sequence بالـ oscilloscope:**

```bash
# شوف الـ regulator اللي بيغذي الـ eMMC
cat /sys/class/regulator/regulator.X/state
```

**خطوة 2 — زود الـ power-up delay في الـ DT:**

```dts
/* arch/arm64/boot/dts/rockchip/rk3562-gateway.dts */
&mmc0 {
    vmmc-supply = <&vcc_emmc>;
    post-power-on-delay-ms = <200>;  /* زِد من 10 لـ 200 */
    status = "okay";
};
```

**خطوة 3 — لو الـ driver بيسمح، زود الـ `busy_timeout` في الـ init path:**

```c
/* في drivers/mmc/core/mmc.c */
cmd.busy_timeout = 1000; /* ms — زِد من 500 لـ 1000 */
```

**خطوة 4 — تحقق بعد التعديل:**

```bash
dmesg | grep mmc0
# المتوقع: mmc0: new HS400 MMC card at address 0001
```

#### الدرس المستفاد
الـ `busy_timeout` في `mmc_command` مش بس hint — الـ host controller بيستخدمه فعلاً لو بيدعم hardware timeout. في custom PCB، دايماً قيس الـ power ramp-up time بالـ scope قبل ما تضبط أي timeout values.

---

### السيناريو 2: Data Corruption في SD Card على STM32MP1 بسبب `blksz` خاطئ

#### العنوان
**STM32MP1 IoT Sensor Node — filesystem corruption بعد كل power cycle**

#### السياق
شركة بتعمل IoT sensor nodes بتجمع بيانات بيئية. كل node بيستخدم STM32MP1 مع microSD card بـ ext4 filesystem. بعد كل قطع تيار مفاجئ، الـ filesystem بتعمل errors وبتحتاج `fsck`.

#### المشكلة
```
EXT4-fs error (device mmcblk0p2): ext4_find_entry
block bitmap and bg descriptor inconsistent
```

الـ corruption بيحصل حتى لو الـ card كانت idle تماماً وقت قطع التيار.

#### التحليل
في `mmc_data`:

```c
struct mmc_data {
    unsigned int    timeout_ns;     /* data timeout (in ns, max 80ms) */
    unsigned int    timeout_clks;   /* data timeout (in clocks) */
    unsigned int    blksz;          /* data block size */
    unsigned int    blocks;         /* number of blocks */
    unsigned int    blk_addr;       /* block address */
    int             error;          /* data error */
    unsigned int    flags;
    unsigned int    bytes_xfered;
    ...
};
```

الـ `mmc_set_data_timeout()` — المعرفة في `core.h` وتنفيذها في `drivers/mmc/core/core.c` — بتحسب الـ `timeout_ns` و `timeout_clks` بناءً على الكارت نفسه:

```c
void mmc_set_data_timeout(struct mmc_data *data, const struct mmc_card *card);
```

المشكلة كانت في الـ custom driver patch اللي عمله أحد المهندسين: بدّل `blksz` من 512 إلى 4096 عشان يحسن الـ performance، لكن الـ `blocks` فضل بيتحسب على أساس 512:

```c
/* BUG: blksz = 4096 لكن blocks = total_bytes / 512 */
data->blksz  = 4096;
data->blocks = request_size / 512;  /* خطأ! المفروض / 4096 */
```

ده خلى `bytes_xfered` يتحسب غلط لأن:

```
actual_transfer = blksz * blocks = 4096 * (size/512) = size * 8
```

الـ DMA بيكتب 8x البيانات المطلوبة، وده بيـ corrupt الـ blocks المجاورة على الـ card.

#### الحل

**تصحيح الـ driver:**

```c
/* الصح */
data->blksz  = 512;   /* standard MMC block size */
data->blocks = request_size / 512;

/* أو لو عايز 4096 */
data->blksz  = 4096;
data->blocks = request_size / 4096;  /* لازم يتطابق */
```

**تحقق بالـ tracing:**

```bash
# تفعيل MMC tracing
echo 1 > /sys/kernel/debug/tracing/events/mmc/enable
cat /sys/kernel/debug/tracing/trace | grep "blksz"
```

**مراقبة `bytes_xfered`:**

```bash
# لو bytes_xfered != blksz * blocks → في bug
# راجع drivers/mmc/host/stm32_sdmmc2.c
```

#### الدرس المستفاد
الـ `blksz` و `blocks` في `mmc_data` لازم يتطابقوا دايماً. الـ MMC core مش بيـ validate العلاقة دي — المسؤولية على الـ caller. أي تعديل على الـ block size لازم يتغير معاه الـ blocks calculation.

---

### السيناريو 3: Android TV Box على Allwinner H616 — eMMC بيـ hang بسبب CQE و `MMC_DATA_QBR`

#### العنوان
**Android TV Box على H616 — UI freezes عشوائية بسبب Command Queue misuse**

#### السياق
مصنع صيني بيعمل Android TV box بالـ Allwinner H616. الـ eMMC بـ 64GB بيشتغل بـ HS400. المستخدمين بيبلغوا عن UI freeze لمدة 2-5 ثواني بشكل عشوائي أثناء تشغيل الـ apps.

#### المشكلة
```
mmc0: CQE error: status 0x00000008
mmc0: cqhci: timeout waiting for halt
```

الـ Android `logcat` بيظهر:

```
StorageManager: waitForAsyncVold timeout
```

#### التحليل
الـ `mmc_data` بيحتوي على flags خاصة بالـ CQE (Command Queue Engine):

```c
/* Extra flags used by CQE */
#define MMC_DATA_QBR        BIT(10)  /* CQE queue barrier */
#define MMC_DATA_PRIO       BIT(11)  /* CQE high priority */
#define MMC_DATA_REL_WR     BIT(12)  /* Reliable write */
#define MMC_DATA_DAT_TAG    BIT(13)  /* Tag request */
#define MMC_DATA_FORCED_PRG BIT(14)  /* Forced programming */
```

الـ `MMC_DATA_QBR` (Queue Barrier) مهمته إنه يمنع الـ eMMC من reorder الـ commands اللي جاية بعده. لو الـ driver بعت barrier بشكل غلط أو متكرر على كل write، الـ eMMC بيعمل drain لكل الـ queue قبل ما يكمل — ده بيخلى throughput يوقع بشكل دراماتيكي.

في الـ `mmc_request`:

```c
bool    cap_cmd_during_tfr;  /* Allow other commands during data transfer */
int     tag;                 /* CQE task tag */
```

الـ `tag` ده هو index الـ task في الـ CQE hardware queue (0-31 عادةً). لو الـ `tag` اتعين غلط أو اتكرر لطلبين مختلفين في نفس الوقت، الـ CQE hardware بيـ stall.

الـ `mmc_request` كمان بيحتوي على:

```c
void (*recovery_notifier)(struct mmc_request *);
```

اللي الـ CQE بيستخدمه عشان يبلغ الـ upper layers إن في error محتاج recovery. لو الـ `recovery_notifier` اتنسي يتعين أو بيحصله null dereference، الـ recovery loop بيتعلق.

#### الحل

**خطوة 1 — تعطيل CQE مؤقتاً للتشخيص:**

```bash
# في الـ DT
&mmc0 {
    no-mmc-hs400;
    /delete-property/ supports-cqe;
};
```

**خطوة 2 — تحقق من الـ QBR frequency:**

```bash
echo 1 > /sys/kernel/debug/tracing/events/mmc/mmc_request_start/enable
cat /sys/kernel/debug/tracing/trace | grep "QBR" | wc -l
# لو كتير جداً بالنسبة لعدد الـ writes → في over-use للـ barrier
```

**خطوة 3 — تصحيح الـ driver (sunxi-mmc.c):**

```c
/* ضع QBR بس على الـ REL_WR requests مش على كل write */
if (data->flags & MMC_DATA_REL_WR)
    data->flags |= MMC_DATA_QBR;
```

#### الدرس المستفاد
الـ CQE flags في `mmc_data` خصوصاً `MMC_DATA_QBR` لها تأثير مباشر على الـ eMMC performance. الـ barrier مش مجاني — كل `QBR` بيعمل pipeline flush. استخدمه بس مع الـ reliable writes أو لما الـ ordering مهم فعلاً.

---

### السيناريو 4: i.MX8M Plus Automotive ECU — `mmc_hw_reset` بيودي في kernel panic

#### العنوان
**Automotive ECU على i.MX8M Plus — kernel panic أثناء eMMC error recovery**

#### السياق
شركة automotive بتعمل ECU (Engine Control Unit) بالـ i.MX8M Plus. الـ eMMC بيحتوي على الـ boot partition والـ data partition. في بيئة الـ testing، بيحاكوا power glitches عشان يتحققوا من الـ error recovery. لقوا إن الـ system بيعمل kernel panic بدل ما يعمل recover.

#### المشكلة
```
BUG: kernel NULL pointer dereference, address: 0000000000000070
Call trace:
 mmc_hw_reset+0x4c/0x120
 mmc_blk_reset+0x28/0x80
 mmc_blk_err_check+0x1f0/0x380
```

#### التحليل
`mmc_hw_reset` معرفة في `core.h`:

```c
int mmc_hw_reset(struct mmc_card *card);
```

وتنفيذها في `drivers/mmc/core/core.c`. الـ function بتعمل reset للـ card عن طريق الـ host controller. الـ panic بيجي من إن الـ `mmc_card` pointer في حالة معينة بيكون غير مكتمل التهيئة.

بالتفصيل: لما بيحصل error في `mmc_request`:

```c
struct mmc_request {
    struct mmc_command  *cmd;
    struct mmc_data     *data;
    struct mmc_command  *stop;
    void (*done)(struct mmc_request *);
    void (*recovery_notifier)(struct mmc_request *);
    struct mmc_host     *host;
    ...
};
```

لو الـ `recovery_notifier` اتعين على function بتستدعي `mmc_hw_reset` لكن الـ `host` pointer في `mmc_request` بيكون NULL (بيحصل لو الـ request اتبنى جزئياً قبل error)، الـ `mmc_hw_reset` بيـ dereference الـ NULL.

الـ power glitch في الـ automotive test كان بيتسبب في إن الـ host controller يعمل reset لنفسه قبل ما الـ software يخلص من الـ ongoing request — بيخلي `mrq->host` invalid.

الـ `completion` في `mmc_request`:

```c
struct completion   completion;
struct completion   cmd_completion;
```

بعد الـ hardware reset، الـ `complete()` ممكن ميتعملش، وده بيخلي أي thread بيستنى `wait_for_completion()` يتعلق للأبد.

#### الحل

**خطوة 1 — تأمين الـ recovery path:**

```c
/* في drivers/mmc/core/core.c */
int mmc_hw_reset(struct mmc_card *card)
{
    /* لازم نتأكد إن الـ host موجود */
    if (!card || !card->host)
        return -EINVAL;
    ...
}
```

**خطوة 2 — ضمان complete() حتى بعد الـ hardware reset:**

```c
/* في الـ host driver (imx-usdhc.c) */
static void usdhc_host_reset(struct mmc_host *host)
{
    struct mmc_request *mrq = host->ongoing_mrq;
    if (mrq) {
        mrq->cmd->error = -ETIMEDOUT;
        /* لازم نعمل complete عشان ميتعلقش */
        complete(&mrq->completion);
    }
    /* ثم نعمل hardware reset */
    ...
}
```

**خطوة 3 — في الـ DT، ضع reset GPIO:**

```dts
&usdhc3 {
    reset-gpios = <&gpio3 16 GPIO_ACTIVE_LOW>;
    reset-delay-us = <200>;
};
```

#### الدرس المستفاد
الـ `completion` في `mmc_request` لازم يتعمله `complete()` في كل الـ code paths — حتى error paths وحتى hardware reset paths. في الـ automotive context، الـ power glitch recovery لازم يكون deterministic ومضمون ما يعلقش.

---

### السيناريو 5: AM62x Custom Board Bring-up — SDIO WiFi مش بيتشاف بسبب `MMC_RSP_R5` flags

#### العنوان
**Custom Board Bring-up على AM62x — SDIO WiFi chip مش بيتعرف عليه**

#### السياق
مهندس بيعمل bring-up لـ custom board بالـ TI AM62x مع Realtek RTL8723DS WiFi/BT chip عبر SDIO. الـ chip موجودة على البورد وتم التحقق منها بالـ hardware team، لكن الـ Linux مش بيشوفها خالص.

#### المشكلة
```
mmc1: error -84 whilst initialising SDIO card
mmc1: SDIO CMD52 failed: error=-84
```

الـ `-84` ده `EILSEQ` — يعني مشكلة في الـ format أو الـ CRC.

#### التحليل
الـ SDIO بيستخدم CMD52 (IO_RW_DIRECT) وCMD53 (IO_RW_EXTENDED). كلاهم بيرجعوا `R5`:

```c
#define MMC_RSP_R5  (MMC_RSP_PRESENT|MMC_RSP_CRC|MMC_RSP_OPCODE)
```

يعني R5 بيحتوي على:
- `MMC_RSP_PRESENT` — في response
- `MMC_RSP_CRC` — الـ host لازم يتحقق من CRC
- `MMC_RSP_OPCODE` — الـ response لازم يحتوي على نفس الـ opcode

الـ `EILSEQ` بيعني إن الـ CRC check فشل أو الـ opcode في الـ response غلط. في custom board، الغالب السبب هو:

1. **مشكلة في الـ signal integrity**: الـ SDIO data lines فيها reflections بسبب PCB layout.
2. **voltage mismatch**: الـ RTL8723DS بيشتغل على 1.8V لكن الـ host بيبعت 3.3V.

بالنظر لـ `mmc_command`:

```c
u32     resp[4];        /* response buffer */
int     error;          /* command error */
```

والـ macros:

```c
#define MMC_RSP_CRC     (1 << 2)    /* expect valid crc */
#define MMC_RSP_OPCODE  (1 << 4)    /* response contains opcode */
```

الـ `mmc_resp_type()` macro:

```c
#define mmc_resp_type(cmd)  ((cmd)->flags & \
    (MMC_RSP_PRESENT|MMC_RSP_136|MMC_RSP_CRC|MMC_RSP_BUSY|MMC_RSP_OPCODE))
```

لو الـ host controller قرأ `resp[0]` وشاف إن الـ bits مش متوافقة مع `MMC_RSP_OPCODE`، بيحط `EILSEQ` في `cmd->error`.

**الفحص بالـ logic analyzer** أظهر إن الـ CMD line بيرجع response لكن الـ CRC byte الأخير فيه single-bit error — علامة واضحة على signal integrity problem.

#### الحل

**خطوة 1 — تأكد من الـ voltage في الـ DT:**

```dts
/* arch/arm64/boot/dts/ti/k3-am625-custom.dts */
&sdhci1 {
    vmmc-supply  = <&vcc_3v3>;   /* power supply للـ card */
    vqmmc-supply = <&vcc_1v8>;   /* I/O voltage — RTL8723DS يحتاج 1.8V */
    bus-width    = <4>;
    cap-sd-highspeed;
    keep-power-in-suspend;
    ti,driver-strength-ohm = <50>;  /* قلل الـ drive strength */
    status = "okay";
};
```

**خطوة 2 — اضبط الـ drive strength للـ PCB:**

```dts
/* في pinmux */
AM62X_IOPAD(0x023c, PIN_INPUT_PULLUP | MUX_MODE0) /* MMC1_DAT0 */
/* غير PULL_UP_20K إلى PULL_UP_10K لو في signal issues */
```

**خطوة 3 — تحقق من الـ EILSEQ statistics:**

```bash
# شوف عدد الـ CRC errors
cat /sys/kernel/debug/mmc1/ios
# راجع الـ voltage
cat /sys/bus/mmc/devices/mmc1\:0001/preferred_erase_size
```

**خطوة 4 — لو المشكلة في الـ opcode mismatch وليس CRC:**

```bash
# فعّل MMC debug
echo "file drivers/mmc/core/sdio.c +p" > /sys/kernel/debug/dynamic_debug/control
dmesg | grep "CMD52"
# شوف resp[0] اللي بيتقرأ
```

#### الدرس المستفاد
الـ `EILSEQ` على SDIO يعني إن الـ host وجد مشكلة في الـ CRC أو الـ opcode في الـ response — وده المحدد في `MMC_RSP_R5` flags. الـ `mmc_command.resp[4]` هو الـ raw response — دايماً log الـ raw value أثناء الـ bring-up عشان تفرق بين مشكلة hardware (PCB/voltage) ومشكلة software (flags غلطانة).
## Phase 7: مصادر ومراجع

### مقالات LWN.net

الـ LWN.net هو المرجع الأهم لتتبع تطور الـ MMC subsystem في الـ Linux kernel.

| المقال | الوصف |
|--------|-------|
| [new driver: MMC framework (2002)](https://lwn.net/Articles/7176/) | أول patch لإضافة الـ MMC framework للـ kernel 2.4.19 — نقطة البداية الحقيقية للـ subsystem |
| [1/4 MMC layer](https://lwn.net/Articles/82765/) | مراجعة الـ MMC layer في مرحلة مبكرة من التطوير |
| [Secure Digital (SD) support](https://lwn.net/Articles/126098/) | إضافة دعم SD cards — كشف الـ SCR register وتحليل الـ CSD بشكل مختلف حسب SD/MMC |
| [MMC updates](https://lwn.net/Articles/253888/) | تحديثات كبيرة شملت دعم SDIO وSPI وحذف الـ MMC-specific error codes |
| [mmc: Add SD4.0 support](https://lwn.net/Articles/642336/) | إضافة دعم SD 4.0 للـ core layer |
| [mmc: Add Command Queue support](https://lwn.net/Articles/738160/) | إضافة الـ CQE (Command Queue Engine) — أثّر مباشرةً على الـ `mmc_data` flags زي `MMC_DATA_QBR` و`MMC_DATA_PRIO` |
| [Add MMC software queue support](https://lwn.net/Articles/803435/) | دعم الـ software command queue بعمق 32 request |
| [pwrseq: Add subsystem for power sequences](https://lwn.net/Articles/602855/) | الـ power sequencing للـ MMC/SDIO devices |
| [mmc: Add OMAP SDHCI driver](https://lwn.net/Articles/732566/) | مثال عملي على host controller driver يستخدم الـ `mmc_request` |

---

### التوثيق الرسمي في الـ Kernel

الـ `Documentation/driver-api/mmc/` هو الموقع الرسمي للتوثيق داخل الـ kernel tree.

```
Documentation/
└── driver-api/
    └── mmc/
        ├── index.rst          ← نقطة البداية
        ├── mmc-async-req.rst  ← شرح pre_req/post_req وكيفية تصغير latency
        ├── mmc-dev-attrs.rst  ← block device attributes للـ SD/MMC
        ├── mmc-dev-parts.rst  ← partitions في eMMC
        └── mmc-tools.rst      ← استخدام mmc-utils من userspace
```

**روابط مباشرة:**

- [MMC/SD/SDIO card support — index](https://docs.kernel.org/driver-api/mmc/index.html)
- [MMC Asynchronous Request](https://docs.kernel.org/driver-api/mmc/mmc-async-req.html) — شرح مهم لفهم `mmc_start_req()` والعلاقة بين `pre_req` و`post_req`
- [SD and MMC Block Device Attributes](https://docs.kernel.org/driver-api/mmc/mmc-dev-attrs.html)
- [SD and MMC Device Partitions](https://docs.kernel.org/driver-api/mmc/mmc-dev-parts.html)
- [MMC tools introduction](https://docs.kernel.org/driver-api/mmc/mmc-tools.html)

**الملفات المصدرية الرئيسية في الـ kernel:**

```
include/linux/mmc/
├── core.h      ← mmc_command, mmc_data, mmc_request, uhs2_command
├── host.h      ← mmc_host, mmc_host_ops
├── card.h      ← mmc_card
├── mmc.h       ← MMC opcodes وregister definitions
└── sd.h        ← SD-specific opcodes

drivers/mmc/
├── core/
│   ├── core.c       ← mmc_wait_for_req(), mmc_wait_for_cmd()
│   ├── mmc.c        ← eMMC initialization
│   ├── sd.c         ← SD initialization
│   └── sdio.c       ← SDIO initialization
└── host/            ← host controller drivers
```

---

### Commits مهمة في الـ Kernel Git

**للبحث في تاريخ الملف مباشرةً:**

```bash
# تاريخ كامل لـ core.h
git log --oneline -- include/linux/mmc/core.h

# إيمتى اتضافت الـ CQE flags في mmc_data
git log --oneline -S "MMC_DATA_QBR" -- include/linux/mmc/core.h

# إيمتى اتضافت uhs2_command
git log --oneline -S "uhs2_command" -- include/linux/mmc/core.h

# إيمتى اتضافت SDUC (has_ext_addr)
git log --oneline -S "has_ext_addr" -- include/linux/mmc/core.h
```

**روابط مباشرة على cgit:**

- [include/linux/mmc/core.h — kernel.org cgit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/include/linux/mmc/core.h)
- [drivers/mmc/core/ — kernel.org cgit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/mmc/core)

---

### نقاشات الـ Mailing List

الـ linux-mmc mailing list هو القناة الرئيسية لمتابعة تطور الـ subsystem.

- **الأرشيف الرسمي:** [lore.kernel.org/linux-mmc](https://lore.kernel.org/linux-mmc/)
- **نقاش مبكر عن العلاقة بين mmc_request وmmc_data وmmc_command:** [LKML 2007](https://lkml.iu.edu/hypermail/linux/kernel/0712.3/0011.html)
- **patch لـ SD function extension registers (Ulf Hansson):** [lore.kernel.org](https://lore.kernel.org/all/20210504161222.101536-10-ulf.hansson@linaro.org/)
- **patchwork للـ linux-mmc:** [patchwork.kernel.org/project/linux-mmc](https://patchwork.kernel.org/project/linux-mmc/)

**المُشرف الحالي على الـ subsystem:** Ulf Hansson من Linaro — أغلب الـ patches بتعدي عليه.

---

### KernelNewbies — تطور الـ MMC عبر الإصدارات

| الإصدار | التغيير المهم |
|---------|--------------|
| [Linux 2.6.12](https://kernelnewbies.org/Linux_2_6_12) | أول دعم SD في الـ MMC subsystem |
| [Linux 2.6.17](https://kernelnewbies.org/Linux_2_6_17) | SDHCI driver وSHD controller interface |
| [Linux 2.6.21](https://kernelnewbies.org/Linux_2_6_21) | دعم SDHC cards (High Capacity) |
| [Linux 2.6.24](https://kernelnewbies.org/Linux_2_6_24) | SPI وSDIO support |
| [Linux 2.6.27](https://kernelnewbies.org/Linux_2_6_27) | SDIO IRQ وscatter-gather |
| [Linux 2.6.32](https://kernelnewbies.org/Linux_2_6_32) | power saving features في OMAP HSMMC |
| [Linux 6.6](https://kernelnewbies.org/Linux_6.6) | تحسينات حديثة في الـ MMC subsystem |

---

### eLinux.org

- [eMMC HS Tests](https://elinux.org/Tests:eMMC-HS) — اختبار HS200 وHS400 في الـ SDHI driver
- [SD/MMC High-Speed Support in Linux Kernel (PDF)](https://www.elinux.org/File:Clement-sd-mmc-high-speed-support-in-linux-kernel.pdf) — ورقة بحثية تشرح UHS-I وHS200
- [EVM Second MMC/SD](https://elinux.org/Second_MMC_/_SD) — مشاكل عملية في تحديد mmcblk0 vs mmcblk1

---

### كتب مُوصى بيها

#### Linux Device Drivers 3rd Edition (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **المجاني:** [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)
- **الفصول ذات الصلة:**
  - Chapter 15: Memory Mapping and DMA — فهم `scatterlist` المستخدمة في `mmc_data.sg`
  - Chapter 14: The Linux Device Model — فهم الـ bus/device/driver model اللي بيقوم عليه الـ MMC subsystem

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل المهم:** Chapter 17: Devices and Modules
- يشرح الـ block device layer اللي بيتعامل مع الـ MMC block driver

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل المهم:** Chapter 11: BusyBox وChapter 9: File System
- بيشرح كيف الـ eMMC وSD cards بتتعامل معاها في الـ embedded systems
- مفيد لفهم السياق العملي لـ `mmc_wait_for_req()` في الـ boot process

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- تفاصيل عميقة عن الـ block layer وكيف بيتكامل مع الـ MMC

---

### مصادر تقنية إضافية

- **JEDEC Standards (eMMC):** [jedec.org — JESD84 (eMMC)](https://www.jedec.org/standards-documents/docs/jesd84-b51) — المواصفة الرسمية اللي بيُنفَّذها الـ kernel driver
- **SD Association Specs:** [sdcard.org/developers](https://www.sdcard.org/developers/) — مواصفات SD Physical Layer
- **mmc-utils (userspace tool):** [git.kernel.org/mmc-utils](https://git.kernel.org/pub/scm/utils/mmc/mmc-utils.git/) — أداة للتعامل مع الـ eMMC من userspace

---

### Search Terms للبحث عن مزيد من المعلومات

```
# بحث في LWN.net
site:lwn.net "mmc" "mmc_request"
site:lwn.net "eMMC" "command queue"
site:lwn.net "SDIO" linux kernel

# بحث في الـ mailing list
site:lore.kernel.org mmc_command mmc_data
site:lore.kernel.org UHS2 linux mmc

# بحث تقني عام
linux kernel "struct mmc_request" "struct mmc_command" internals
linux mmc subsystem "mmc_wait_for_req" implementation
linux eMMC "command queue engine" CQHCI
linux MMC "scatter gather" DMA mmc_data
linux SD UHS2 "uhs2_command" kernel patch
linux SDUC "ext_addr" mmc core
linux mmc crypto "bio_crypt_ctx" inline encryption
```
## Phase 8: Writing simple module

### الفكرة

**الـ** `mmc_wait_for_req` هي الدالة المحورية اللي بتتنفذ كل request على كارت MMC/SD — كل read أو write بيمر منها. هنستخدم **kprobe** عشان نـhook عليها ونطبع معلومات عن الـ request اللي بيتبعت: الـ opcode بتاع الـ command، وعدد الـ blocks، واتجاه الـ data (read/write).

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * mmc_req_probe.c
 * Hooks mmc_wait_for_req() via kprobe to log every MMC request:
 * command opcode, data direction, and block count.
 */

#include <linux/module.h>       /* MODULE_*, module_init/exit         */
#include <linux/kprobes.h>      /* kprobe, register_kprobe, etc.      */
#include <linux/mmc/core.h>     /* mmc_request, mmc_command, mmc_data */
#include <linux/mmc/host.h>     /* mmc_host                           */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("kernel-doc demo");
MODULE_DESCRIPTION("kprobe on mmc_wait_for_req to trace MMC requests");

/* ------------------------------------------------------------------ */
/* pre_handler: called just before mmc_wait_for_req() executes         */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64: first arg (host)  → rdi
     *            second arg (mrq)  → rsi
     * On ARM64:  first arg         → x0
     *            second arg        → x1
     * pt_regs_param() هنا بنجيب الـ args من الـ registers مباشرةً
     */
#ifdef CONFIG_X86_64
    struct mmc_host    *host = (struct mmc_host *)regs->di;
    struct mmc_request *mrq  = (struct mmc_request *)regs->si;
#elif defined(CONFIG_ARM64)
    struct mmc_host    *host = (struct mmc_host *)regs->regs[0];
    struct mmc_request *mrq  = (struct mmc_request *)regs->regs[1];
#else
    /* unsupported arch — skip silently */
    return 0;
#endif

    /* Guard against NULL pointers before dereferencing */
    if (!mrq || !mrq->cmd)
        return 0;

    if (mrq->data) {
        /* Request carries a data phase (read or write) */
        pr_info("mmc_probe: host=%s opcode=CMD%u dir=%s blocks=%u blksz=%u\n",
                dev_name(host->parent),        /* e.g. "mmc0"             */
                mrq->cmd->opcode,              /* CMD17=single-read, etc. */
                (mrq->data->flags & MMC_DATA_READ) ? "READ" : "WRITE",
                mrq->data->blocks,             /* number of 512-byte blks */
                mrq->data->blksz);             /* bytes per block         */
    } else {
        /* Control command only, no data transfer */
        pr_info("mmc_probe: host=%s opcode=CMD%u (no data)\n",
                dev_name(host->parent),
                mrq->cmd->opcode);
    }

    return 0; /* 0 = let the real function continue normally */
}

/* ------------------------------------------------------------------ */
/* kprobe descriptor                                                   */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "mmc_wait_for_req", /* target function by name     */
    .pre_handler = handler_pre,        /* our callback runs before it */
};

/* ------------------------------------------------------------------ */
/* module_init                                                         */
/* ------------------------------------------------------------------ */
static int __init mmc_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("mmc_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("mmc_probe: planted kprobe at %s (%px)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit                                                         */
/* ------------------------------------------------------------------ */
static void __exit mmc_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("mmc_probe: kprobe removed\n");
}

module_init(mmc_probe_init);
module_exit(mmc_probe_exit);
```

---

### Makefile

```makefile
obj-m += mmc_req_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

```bash
# بناء وتحميل الموديول
make
sudo insmod mmc_req_probe.ko
sudo dmesg | grep mmc_probe

# لو عايز تجرب: اعمل أي I/O على كارت SD
dd if=/dev/mmcblk0 of=/dev/null bs=4096 count=16

# لإزالة الموديول
sudo rmmod mmc_req_probe
```

---

### شرح كل جزء

#### الـ includes

```c
#include <linux/kprobes.h>
#include <linux/mmc/core.h>
#include <linux/mmc/host.h>
```

الـ `kprobes.h` بيجيب تعريف `struct kprobe` و `register_kprobe`. الـ `mmc/core.h` بيجيب تعريف `mmc_request`، `mmc_command`، `mmc_data` — اللي هما بالظبط الـ structs اللي بنفك تعريفها جوا الـ handler. الـ `mmc/host.h` بيجيب `mmc_host` عشان نطبع اسم الجهاز.

#### الـ pre_handler وأرجومنتاته

```c
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
```

الكيرنل بيوفر `pt_regs` اللي فيها snapshot للـ CPU registers لحظة دخول الدالة. بنجيب منها الأرجومنتات حسب الـ ABI: على x86-64 الأول في `rdi` والتاني في `rsi`، وعلى ARM64 في `x0` و`x1`. المتغيرات `host` و`mrq` اللي بنجيبهم دول هما نفس اللي اتبعتولها الدالة `mmc_wait_for_req`.

#### فحص الـ data direction

```c
(mrq->data->flags & MMC_DATA_READ) ? "READ" : "WRITE"
```

الـ `flags` داخل `mmc_data` بيحدد اتجاه النقل. `MMC_DATA_READ` معناه الكارت بيبعت data للـ host، وعدمه معناه الـ host بيكتب على الكارت.

#### الـ return 0

الـ pre_handler لازم يرجع 0 عشان الكيرنل يكمل تنفيذ الدالة الأصلية normally. لو رجع قيمة تانية هيقول للكيرنل إن الـ kprobe handler هيتعامل مع الموضوع هو — ده بيُستخدم في حالات نادرة زي الـ fault injection.

#### الـ module_init

```c
ret = register_kprobe(&kp);
```

`register_kprobe` بتحل محل أول byte من `mmc_wait_for_req` بـ breakpoint instruction وبتسجل الـ handler. لو الدالة مش exported أو مش موجودة في الـ symbol table، بترجع `-EINVAL`.

#### الـ module_exit

```c
unregister_kprobe(&kp);
```

**لازم** ننفذ الـ unregister في الـ exit عشان الكيرنل يرجع الـ original instruction تاني. لو منعملهاش، أي استدعاء لـ `mmc_wait_for_req` بعد تفريغ الموديول هيودي على Oops لأن الـ handler address بقى invalid.

---

### مثال على output الـ dmesg

```
[  45.123456] mmc_probe: planted kprobe at mmc_wait_for_req (ffffffffc0a12340)
[  45.200001] mmc_probe: host=mmc0 opcode=CMD18 dir=READ  blocks=8  blksz=512
[  45.200015] mmc_probe: host=mmc0 opcode=CMD12 (no data)
[  45.201003] mmc_probe: host=mmc0 opcode=CMD25 dir=WRITE blocks=16 blksz=512
```

**الـ** CMD18 هو `READ_MULTIPLE_BLOCK`، والـ CMD25 هو `WRITE_MULTIPLE_BLOCK`، والـ CMD12 هو `STOP_TRANSMISSION` — وده بيوضح بشكل مباشر طبيعة الـ request اللي بتمر على الـ MMC stack.
