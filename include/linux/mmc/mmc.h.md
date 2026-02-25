## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي ليه الملف

الـ `mmc.h` جزء من **MMC/SD/SDIO Subsystem** في Linux kernel. المسؤول عنه Ulf Hansson، والـ mailing list هو `linux-mmc@vger.kernel.org`. الـ subsystem ده بيغطي كل الملفات في:

- `drivers/mmc/` — كور وهوست درايفرز
- `include/linux/mmc/` — الهيدرز
- `include/uapi/linux/mmc/` — الـ userspace API

---

### الصورة الكبيرة — إيه اللي بيحصل؟

#### تخيل المشكلة دي

عندك موبايل، وجوه **eMMC chip** — ده هو التخزين الداخلي. أو عندك كاميرا وفيها **SD card**. الـ OS محتاج يتكلم مع الـ chip دي عشان يقرأ ويكتب بيانات.

المشكلة إن الـ chip دي بتتكلم بـ **بروتوكول خاص** — مش USB ومش SATA. ده بروتوكول اسمه **MMC (MultiMediaCard protocol)**. بيتضمن:
- إرسال **commands** (أوامر) رقمية
- استقبال **responses** (ردود)
- نقل **data blocks**

بالظبط زي ما في لغة بشرية — محتاج تعرف الكلمات والجمل الصح عشان تتكلم مع الكارت.

#### دور الملف `mmc.h`

الملف ده هو **القاموس** أو **المعجم** اللي بيعرّف:

1. **كل الـ commands** اللي ممكن تبعتها للكارت — زي CMD0 (reset)، CMD17 (اقرأ block واحد)، CMD24 (اكتب block)، إلخ.
2. **الـ R1 status bits** — لما الكارت يرد، بيبعت 32-bit فيها flags بتقول "حصل error؟"، "الكارت شغال؟"، "جاهز للداتا؟"
3. **الـ EXT_CSD fields** — العنوان الرقمي لكل إعداد جوه الكارت (زي enable الـ cache، أو ضبط سرعة الباص).
4. **الـ Card Command Classes (CCC)** — تصنيف الأوامر لـ classes زي read، write، erase، lock.
5. **الـ speed modes** — HS، HS200، HS400 — كل mode ده بيدي سرعة أكبر.

---

### القصة الكاملة — رحلة أمر واحد

تخيل إن الـ kernel عايز يقرأ ملف من الـ eMMC:

```
Block Driver (block.c)
       |
       ↓ يطلب قراءة sector
MMC Core (core.c)
       |
       ↓ يختار الـ command الصح من mmc.h (MMC_READ_SINGLE_BLOCK = 17)
MMC Host Driver (مثلاً sdhci.c)
       |
       ↓ يحوّل الـ command لـ signals على الـ bus
eMMC Chip
       |
       ↓ بيبعت R1 response (status)
MMC Core يقرأ الـ R1
       |
       ↓ بيشيك على R1_READY_FOR_DATA و R1_CURRENT_STATE من mmc.h
Block Driver يستلم البيانات
```

الـ `mmc.h` هو الـ **shared dictionary** اللي كل طبقة في الرحلة دي بتستخدمه.

---

### أبرز محتويات الملف

#### 1. الـ Standard MMC Commands

```c
#define MMC_GO_IDLE_STATE         0   /* bc  — reset everything */
#define MMC_SEND_OP_COND          1   /* bcr — negotiate voltage */
#define MMC_ALL_SEND_CID          2   /* bcr — get card identity */
#define MMC_READ_SINGLE_BLOCK    17   /* adtc — read one block */
#define MMC_WRITE_BLOCK          24   /* adtc — write one block */
#define MMC_ERASE                38   /* ac   — erase blocks */
#define MMC_SWITCH                6   /* ac   — change EXT_CSD setting */
```

كل command ليه **نوع**:
- `bc` = broadcast — بيتبعت لكل الكروت
- `bcr` = broadcast with response
- `ac` = addressed command — لكارت معين
- `adtc` = addressed data transfer — بيشيل داتا

#### 2. الـ R1 Response Bits

```c
#define R1_OUT_OF_RANGE     (1 << 31)  /* Address out of range */
#define R1_ADDRESS_ERROR    (1 << 30)  /* Misaligned address */
#define R1_WP_VIOLATION     (1 << 26)  /* Write protected area */
#define R1_CARD_IS_LOCKED   (1 << 25)  /* Card is locked */
#define R1_READY_FOR_DATA   (1 << 8)   /* Card ready to transfer */
#define R1_CURRENT_STATE(x) ((x & 0x00001E00) >> 9)  /* 4-bit state */
```

الـ states الممكنة:
| State | معناه |
|-------|--------|
| `R1_STATE_IDLE` (0) | في البداية، الكارت مش متصل |
| `R1_STATE_TRAN` (4) | Transfer state — جاهز للنقل |
| `R1_STATE_DATA` (5) | بيبعت داتا |
| `R1_STATE_PRG` (7) | بيكتب (programming) |

#### 3. الـ EXT_CSD — سجل الإعدادات المتقدمة

الـ **EXT_CSD** هو جدول من 512 byte جوه الكارت — كل byte عنده رقم عنوان معرّف هنا:

```c
#define EXT_CSD_CACHE_CTRL      33   /* R/W — enable/disable cache */
#define EXT_CSD_BUS_WIDTH       183  /* R/W — set bus width (1/4/8 bit) */
#define EXT_CSD_HS_TIMING       185  /* R/W — set speed mode */
#define EXT_CSD_REV             192  /* RO  — EXT_CSD revision */
#define EXT_CSD_CARD_TYPE       196  /* RO  — supported speed modes */
#define EXT_CSD_SEC_CNT         212  /* RO  — total sector count (4 bytes) */
#define EXT_CSD_RPMB_MULT       168  /* RO  — RPMB partition size */
```

#### 4. الـ Speed Modes

```c
#define EXT_CSD_CARD_TYPE_HS_26     (1<<0)  /* 26 MHz  */
#define EXT_CSD_CARD_TYPE_HS_52     (1<<1)  /* 52 MHz  */
#define EXT_CSD_CARD_TYPE_DDR_1_8V  (1<<2)  /* 52 MHz DDR @1.8V */
#define EXT_CSD_CARD_TYPE_HS200_1_8V (1<<4) /* 200 MHz SDR */
#define EXT_CSD_CARD_TYPE_HS400_1_8V (1<<6) /* 200 MHz DDR */
```

التطور من 26 MHz → 52 MHz → 200 MHz SDR → 400 MB/s DDR — كل ده محتاج إعدادات مختلفة في الـ EXT_CSD.

#### 5. الـ Command Queue (CQ) — الميزة الحديثة

```c
#define MMC_QUE_TASK_PARAMS     44   /* queue a task with params */
#define MMC_QUE_TASK_ADDR       45   /* specify data address */
#define MMC_EXECUTE_READ_TASK   46   /* execute queued read */
#define MMC_EXECUTE_WRITE_TASK  47   /* execute queued write */
#define EXT_CSD_CMDQ_DEPTH      307  /* max queue depth */
```

الـ **Command Queuing** بيخلي الـ kernel يبعت 32 أمر دفعة واحدة بدل ما يستنى كل أمر يخلص — زي الـ NCQ في SATA.

---

### Inline Functions مهمة

```c
static inline bool mmc_op_multi(u32 opcode)
{
    /* هل الأمر ده multi-block transfer؟ */
    return opcode == MMC_WRITE_MULTIPLE_BLOCK ||
           opcode == MMC_READ_MULTIPLE_BLOCK;
}

static inline bool mmc_ready_for_data(u32 status)
{
    /* بيشيك إن الكارت في TRAN state وجاهز للداتا */
    return status & R1_READY_FOR_DATA &&
           R1_CURRENT_STATE(status) == R1_STATE_TRAN;
}
```

---

### الملفات اللي المفروض تعرفها

#### الهيدرز المرتبطة

| الملف | الدور |
|-------|-------|
| `include/linux/mmc/mmc.h` | **الملف ده** — commands وprotocol definitions |
| `include/linux/mmc/host.h` | تعريف `struct mmc_host` و`struct mmc_ios` |
| `include/linux/mmc/card.h` | تعريف `struct mmc_card`، `struct mmc_csd`، `struct mmc_ext_csd` |
| `include/linux/mmc/core.h` | تعريف `struct mmc_request`، `struct mmc_command` |
| `include/linux/mmc/sd.h` | SD-specific commands (مختلف عن MMC) |
| `include/linux/mmc/sdio.h` | SDIO commands للـ Wi-Fi وبلوتوث |

#### الكور درايفرز

| الملف | الدور |
|-------|-------|
| `drivers/mmc/core/mmc.c` | يقرأ EXT_CSD، يضبط الـ timing، يعمل init للكارت |
| `drivers/mmc/core/mmc_ops.c` | ينفذ الـ MMC commands (مثلاً `mmc_switch()`) |
| `drivers/mmc/core/core.c` | الـ core scheduler للـ requests |
| `drivers/mmc/core/block.c` | يربط الـ MMC بـ block device layer |

#### هوست درايفرز (أمثلة)

| الملف | الدور |
|-------|-------|
| `drivers/mmc/host/sdhci.c` | الـ standard SDHCI controller |
| `drivers/mmc/host/dw_mmc.c` | DesignWare MMC controller |
| `drivers/mmc/host/cqhci-core.c` | Command Queue Host Controller Interface |

---

### لماذا الملف ده مهم؟

كل كود بيتعامل مع MMC في الـ kernel — سواء كان هوست درايفر، كور، أو بلوك لاير — **لازم يعرف** رقم كل command، وشكل الـ response، وعنوان كل إعداد في EXT_CSD. الـ `mmc.h` هو المصدر الوحيد لكل المعلومات دي. من غيره، محدش يقدر يتكلم مع الكارت.
## Phase 2: شرح الـ MMC Framework

### المشكلة اللي الـ Framework ده بيحلها

تخيل إنك بتكتب driver لـ eMMC chip على board بـ ARM. من غير framework، هتضطر تكتب:
- كل logic الـ card initialization من الأول (CMD0 → CMD1 → CMD2 → CMD3 ...)
- كل state machine الـ card
- كل حاجة خاصة بـ SD vs MMC vs SDIO
- الـ block layer integration بنفسك

ولما تيجي تشتغل على board تاني بـ controller مختلف، هتعيد كتابة نفس الكود من الأول.

**المشكلة الأساسية**: عندنا طبقتين منفصلتين تماماً:
1. **الـ card protocol** — كلام MMC/SD standard، commands، responses، state machines. ده ثابت بغض النظر عن الـ hardware.
2. **الـ host controller hardware** — الـ SDHCI register map، الـ DMA setup، الـ clock gating. ده بيتغير من SoC للتاني.

الـ Linux MMC Framework بيفصل بينهم فصل تام.

---

### الحل — النهج اللي بياخده الـ Kernel

الـ kernel قسّم المسؤولية لثلاث طبقات:

```
┌─────────────────────────────────────┐
│         Block / Network Layer       │  ← مستخدم الـ storage
├─────────────────────────────────────┤
│         MMC Core (الـ framework)    │  ← المنطق المشترك
├─────────────────────────────────────┤
│   Card Driver   │   Host Driver     │  ← التخصص
│ (mmc/sd/sdio)   │ (sdhci/dw-mmc)   │
├─────────────────────────────────────┤
│         Hardware (eMMC / SDIO)      │
└─────────────────────────────────────┘
```

الـ **MMC Core** هو القلب — بيعمل:
- Card detection وإرسال initialization sequence
- Protocol negotiation (HS/HS200/HS400/DDR)
- Power management
- Error handling وretry logic
- تسجيل الـ card كـ block device

الـ **Host Driver** بيعمل:
- تسجيل نفسه مع الـ Core عبر `struct mmc_host`
- تنفيذ الـ ops (إرسال command، ضبط clock، ضبط voltage)

الـ **Card Driver** (e.g., `mmc_block`) بيعمل:
- تسجيل نفسه كـ block device
- ترجمة الـ bio requests لـ MMC commands

---

### الـ Big Picture Architecture

```
User Space (read/write /dev/mmcblk0)
              │
              ▼
┌─────────────────────────────┐
│      Block Layer (blk-mq)   │  ← subsystem تاني: بيدير الـ I/O queue
└────────────┬────────────────┘
             │  bio requests
             ▼
┌─────────────────────────────┐
│    mmc_block driver         │  ← CONSUMER: يطلب read/write
│  (drivers/mmc/card/block.c) │
└────────────┬────────────────┘
             │  struct mmc_request
             ▼
┌──────────────────────────────────────────────────┐
│                 MMC CORE                          │
│  ┌────────────────────────────────────────────┐  │
│  │  mmc_ops / sd_ops / sdio_ops               │  │  ← card-type logic
│  │  (drivers/mmc/core/mmc.c, sd.c, sdio.c)   │  │
│  ├────────────────────────────────────────────┤  │
│  │  core.c — request dispatch, error retry    │  │
│  │  bus.c  — card registration with sysfs     │  │
│  └────────────────────────────────────────────┘  │
└────────────────────┬─────────────────────────────┘
                     │  mmc_host_ops callbacks
          ┌──────────┴──────────┐
          ▼                     ▼
┌──────────────────┐   ┌──────────────────┐
│  SDHCI driver    │   │  DW-MMC driver   │  ← PROVIDERS
│ (sdhci.c)        │   │ (dw_mmc.c)       │
└────────┬─────────┘   └────────┬─────────┘
         │                      │
         ▼                      ▼
  ┌────────────┐        ┌─────────────┐
  │ eMMC chip  │        │  SD card    │
  └────────────┘        └─────────────┘
```

**الـ Subsystems التانية اللي لازم تعرفها:**
- **Block Layer (blk-mq)**: بيدير قوايم الـ I/O requests من الـ filesystem. الـ mmc_block بيتسجل فيه كـ block device.
- **Regulator Framework**: الـ `struct mmc_supply` بتحتوي على `vmmc` و`vqmmc` اللي هما regulator handles بتتحكم في الـ power supply للـ card.
- **Device Model / sysfs**: كل `mmc_host` و`mmc_card` هي `struct device` كاملة متسجلة في الـ device tree.
- **DMA Framework**: الـ `struct mmc_data` بتحتوي `struct scatterlist` اللي الـ host driver بيعمل لها DMA mapping.

---

### المثال الحقيقي — رحلة الـ read request

على Raspberry Pi 4 (SDHCI controller متصل بـ eMMC):

```
1. الـ filesystem بتطلب read من /dev/mmcblk0p2
2. blk-mq بيحول الطلب لـ bio
3. mmc_block بيحوّل الـ bio لـ struct mmc_request
   ├── struct mmc_command (CMD18: READ_MULTIPLE_BLOCK)
   ├── struct mmc_data (scatter list من الـ page cache)
   └── struct mmc_command stop (CMD12)
4. MMC Core بيبعت الـ request للـ host ops
5. SDHCI driver بيكتب في registers الـ controller
6. الـ controller بيتكلم مع الـ eMMC chip على الـ bus
7. البيانات بتيجي عبر DMA للـ page cache مباشرة
8. الـ completion callback بيشغّل
9. blk-mq بيكمّل الـ bio
```

---

### الـ Analogy الحقيقية — شركة شحن دولي

تخيل شركة شحن دولي زي DHL:

| الـ Analogy | الـ Kernel Concept |
|---|---|
| **العميل** (بيطلب شحن) | الـ `mmc_block` driver |
| **مكتب الـ DHL** (standard interface) | **MMC Core** |
| **بروتوكول الشحن الدولي** (CN23، ATA carnet) | **MMC/SD protocol** — الـ commands في `mmc.h` |
| **الطيارة/الشاحنة** (وسيلة النقل الفعلية) | **Host Controller Driver** (SDHCI, DW-MMC) |
| **المطار/الميناء** (hardware interface) | **الـ physical bus** (SDIO pins، PCIe) |
| **جواز السفر للبضاعة** (بيانات عن البضاعة) | `struct mmc_card` (CID, CSD, EXT_CSD) |
| **فاتورة الشحن** | `struct mmc_request` |
| **تفاصيل الفاتورة** (ماذا، كم، وين) | `struct mmc_command` + `struct mmc_data` |
| **إمكانيات المطار** (max weight، customs rules) | `struct mmc_host` capabilities (caps, caps2) |
| **حالة الجو** (بتأثر على الـ timing) | `struct mmc_ios` (clock, voltage, bus width) |

**التعمق في الـ Analogy:**

- لما DHL بتفتح فرع جديد في مدينة جديدة، مش بتغير إجراءات الشحن الدولي — كذلك لما بتضيف host controller جديد، الـ MMC protocol ما بيتغيرش.
- الطيارة مش محتاجة تعرف إيه اللي جوا الكرتونة — كذلك الـ SDHCI driver مش محتاج يفهم MMC commands، بس يبعتها على الـ wire.
- لو الطيارة اتأخرت (timeout)، مكتب DHL (Core) هو اللي بيقرر هنعيد الشحن ولا لأ (retry logic).

---

### الـ Core Abstraction — الفكرة المحورية

الفكرة المحورية في الـ MMC Framework هي:

> **الفصل التام بين "الـ protocol" و"الـ transport"**

الـ protocol (إيه الكوماند اللي هنبعته وإيه الـ response المتوقع) ده **ثابت** ومعرّف في الـ MMC/SD specification.

الـ transport (إزاي هنبعت الـ bits على الـ wire) ده **متغير** حسب الـ hardware.

الـ framework بيحقق ده عبر **vtable pattern** — الـ `struct mmc_host_ops`:

```c
/*
 * The host driver fills this table with its hardware-specific
 * implementations. The Core calls them without knowing the hardware.
 */
struct mmc_host_ops {
    /* Send a request to the hardware */
    void (*request)(struct mmc_host *host, struct mmc_request *req);

    /* Configure bus: clock, voltage, width, timing */
    void (*set_ios)(struct mmc_host *host, struct mmc_ios *ios);

    /* Is there a card inserted? */
    int  (*get_cd)(struct mmc_host *host);

    /* Is the card write-protected? */
    int  (*get_ro)(struct mmc_host *host);

    /* High-speed tuning: hardware-specific signal calibration */
    int  (*execute_tuning)(struct mmc_host *host, u32 opcode);

    /* ... more callbacks ... */
};
```

الـ Core بيكلّم الـ hardware دايماً عبر هذه الـ ops فقط — zero direct hardware access.

---

### الـ Central Structs وعلاقاتها

```
struct mmc_host                        struct mmc_card
┌─────────────────────────┐           ┌──────────────────────────┐
│ *parent (struct device) │◄──────────│ *host                    │
│ ops → mmc_host_ops      │           │ dev (struct device)      │
│ ios → mmc_ios           │           │ type (MMC/SD/SDIO)       │
│   ├ clock               │           │ rca (relative address)   │
│   ├ bus_width           │           │ cid → mmc_cid            │
│   ├ timing              │           │   ├ manfid               │
│   └ signal_voltage      │           │   └ prod_name            │
│ caps / caps2 (bitmasks) │           │ csd → mmc_csd            │
│ f_min / f_max           │           │   └ capacity             │
│ *card → mmc_card ───────┼──────────►│ ext_csd → mmc_ext_csd   │
│ ocr_avail               │           │   ├ hs_max_dtr           │
│ supply → mmc_supply     │           │   ├ cmdq_support         │
│   ├ *vmmc (regulator)   │           │   └ cache_size           │
│   └ *vqmmc (regulator)  │           │ part[] → mmc_part[7]    │
│ max_blk_size            │           │   (boot0/1, gp0-3, rpmb) │
│ max_blk_count           │           └──────────────────────────┘
│ *cqe_ops → mmc_cqe_ops  │
└─────────────────────────┘

struct mmc_request
┌────────────────────────┐
│ *sbc → mmc_command     │  ← CMD23: SET_BLOCK_COUNT (optional)
│ *cmd → mmc_command     │  ← الـ main command (e.g., CMD18)
│   ├ opcode             │
│   ├ arg                │
│   ├ resp[4]            │  ← 128-bit response buffer
│   └ flags (RSP type)   │
│ *data → mmc_data       │  ← بيانات القراءة/الكتابة
│   ├ blksz              │
│   ├ blocks             │
│   ├ *sg (scatterlist)  │  ← DMA scatter/gather list
│   └ flags (READ/WRITE) │
│ *stop → mmc_command    │  ← CMD12: STOP_TRANSMISSION
│ done() callback        │
└────────────────────────┘
```

---

### ما يملكه الـ Framework (Core) وما يفوّضه للـ Drivers

#### الـ Core بيمتلك ويتحكم فيه:

| المسؤولية | المكان |
|---|---|
| **Card initialization sequence** | `drivers/mmc/core/mmc.c` — CMD0→1→2→3→... |
| **Speed mode negotiation** | بيقرأ `EXT_CSD_CARD_TYPE` ويقارنه بـ `host->caps` |
| **Tuning trigger** | بيستدعي `host->ops->execute_tuning()` بعد الـ timing switch |
| **Error retry logic** | بيحاول تاني لو `cmd->error != 0` |
| **Power sequence** | بيطلب من الـ regulator framework رفع الـ vmmc |
| **Card detection** | `mmc_detect_change()` → delayed work |
| **sysfs/uevent** | تسجيل `mmc_card` كـ `struct device` |
| **Block count negotiation** | بيبعت CMD23 لو `MMC_CAP_CMD23` موجود |

#### الـ Core بيفوّض للـ Host Driver:

| المسؤولية | الـ ops callback |
|---|---|
| إرسال command على الـ wire | `->request()` |
| ضبط الـ clock frequency | `->set_ios()` |
| ضبط الـ bus width (1/4/8 bit) | `->set_ios()` |
| voltage switching (3.3V → 1.8V) | `->start_signal_voltage_switch()` |
| كشف وجود الـ card | `->get_cd()` |
| كشف الـ write protection | `->get_ro()` |
| signal tuning للـ HS200/HS400 | `->execute_tuning()` |
| hardware reset عبر RST_n | `->card_hw_reset()` |

#### الـ Core بيفوّض للـ Card Driver (mmc_block):

| المسؤولية | الطريقة |
|---|---|
| تسجيل الـ block device | `register_blkdev()` |
| ترجمة الـ bio لـ mmc_request | في `mmc_blk_mq_issue_rq()` |
| الـ partition management | عبر `mmc_part` array |

---

### الـ MMC Commands في `mmc.h` — لماذا هي هكذا؟

الـ commands المعرّفة في `mmc.h` مش عشوائية — هي مقسّمة لـ **Command Classes (CCC)**:

```
┌────────────────────────────────────────────────────────────┐
│  CSD Register: bits [95:84] = CCC (Command Class bitmask)  │
│                                                            │
│  Class 0 (Basic):     CMD0,1,2,3,4,7,9,10,12,13,15       │
│  Class 2 (Block R):   CMD16,17,18                         │
│  Class 4 (Block W):   CMD16,24,25,26,27                   │
│  Class 5 (Erase):     CMD35,36,38                         │
│  Class 6 (Write Prot):CMD28,29,30                         │
│  Class 7 (Lock):      CMD42                               │
│  Class 8 (App):       CMD55,56                            │
│  Class 10 (Switch):   CMD6                                │
│  Class 11 (CmdQ):     CMD44,45,46,47,48                  │
└────────────────────────────────────────────────────────────┘
```

الـ Core بيقرأ الـ CSD من الـ card ويعرف أي classes مدعومة قبل ما يبعت أي command.

---

### الـ EXT_CSD — قلب الـ eMMC Configuration

الـ `EXT_CSD` هو register بحجم **512 byte** موجود داخل الـ eMMC chip. مش موجود في الـ SD standard — eMMC حصري.

الـ Core بيقراه مرة واحدة أثناء الـ initialization بـ CMD8 (SEND_EXT_CSD) ويخزّنه في `mmc_card.ext_csd`.

**أهم الـ fields:**

```c
/*
 * EXT_CSD byte 196 — what speeds this card supports
 * The Core compares this with host->caps2 to pick the best mode
 */
#define EXT_CSD_CARD_TYPE_HS400_1_8V  (1<<6)  /* 200MHz DDR at 1.8V */
#define EXT_CSD_CARD_TYPE_HS200_1_8V  (1<<4)  /* 200MHz SDR at 1.8V */
#define EXT_CSD_CARD_TYPE_DDR_1_8V    (1<<2)  /* 52MHz DDR at 1.8V */
#define EXT_CSD_CARD_TYPE_HS_52       (1<<1)  /* 52MHz */

/*
 * EXT_CSD byte 185 — current timing mode
 * Written by Core via CMD6 (SWITCH) to change speed
 */
#define EXT_CSD_TIMING_HS400  3
#define EXT_CSD_TIMING_HS200  2
#define EXT_CSD_TIMING_HS     1

/*
 * EXT_CSD byte 308 — does this card support Command Queuing?
 * If yes, Core enables CQE path via mmc_cqe_ops
 */
#define EXT_CSD_CMDQ_SUPPORTED  BIT(0)
```

---

### الـ Card State Machine

الـ MMC spec بتعرّف states للـ card، والـ Core مسؤول إنه يتابعها:

```
                    ┌─────┐
         Power on   │     │ CMD0
    ────────────────► IDLE ◄──────────────────────────────┐
                    └──┬──┘                               │
                  CMD1 │ (SEND_OP_COND)                   │
                    ┌──▼──┐                               │
                    │READY│                               │
                    └──┬──┘                               │
                  CMD2 │ (ALL_SEND_CID)                   │
                    ┌──▼──┐                               │
                    │IDENT│                               │
                    └──┬──┘                               │
                  CMD3 │ (SET_RELATIVE_ADDR)              │
                    ┌──▼──┐                               │
         ┌──────────►STBY ◄─────────────────────┐        │
         │          └──┬──┘                      │        │
         │        CMD7 │ (SELECT_CARD)           │        │
         │          ┌──▼──┐                      │        │
         │    ┌─────►TRAN ◄────────────────────┐ │        │
         │    │     └──┬──┘  CMD12/error        │ │        │
         │    │  CMD18 │                        │ │        │
         │    │     ┌──▼──┐                     │ │        │
         │    │     │DATA │ ──── error ──────── │─┘        │
         │    │     └─────┘                     │          │
         │    │  CMD24/25                        │          │
         │    │     ┌─────┐                     │          │
         │    └─────┤ RCV │                     │          │
         │          └──┬──┘                     │          │
         │             │ CMD12                  │          │
         │          ┌──▼──┐                     │          │
         └──────────┤ PRG │ ─── write done ─────┘          │
                    └──┬──┘                                 │
                  CMD15│ (GO_INACTIVE)                      │
                    ┌──▼──┐                                 │
                    │ DIS │ ───────────────────────────────►┘
                    └─────┘

R1_CURRENT_STATE(status) بيقرأ الـ state من الـ card response
mmc_ready_for_data() بيتحقق إن الـ card في TRAN state وجاهزة
```

---

### الـ R1 Response وكيف الـ Core بيفسّره

كل command بيرجع response. الـ R1 هو الأكثر شيوعاً — 32 bit:

```
 Bit 31: OUT_OF_RANGE  — العنوان خارج حدود الـ card
 Bit 30: ADDRESS_ERROR — الـ address مش aligned
 Bit 29: BLOCK_LEN_ERROR
 Bit 23: COM_CRC_ERROR — الـ CRC للـ command غلط
 Bit 22: ILLEGAL_COMMAND — الـ command مش مدعوم في الـ state الحالية
 Bits 12:9 — CURRENT_STATE (4 bits: IDLE/READY/IDENT/STBY/TRAN/DATA/RCV/PRG/DIS)
 Bit  8: READY_FOR_DATA
 Bit  7: SWITCH_ERROR — CMD6 switch فشل

/* The Core uses this macro constantly to check card state */
#define R1_CURRENT_STATE(x)  ((x & 0x00001E00) >> 9)

/* And this function before any data transfer */
static inline bool mmc_ready_for_data(u32 status)
{
    return status & R1_READY_FOR_DATA &&
           R1_CURRENT_STATE(status) == R1_STATE_TRAN;
}
```

---

### الـ Bus Width وتأثيره على الـ Performance

```
Bus Width │ Max Throughput (HS) │ Max Throughput (HS400) │ الـ EXT_CSD Setting
──────────┼─────────────────────┼────────────────────────┼──────────────────────
1-bit     │  6.5 MB/s           │  N/A                   │ EXT_CSD_BUS_WIDTH_1
4-bit     │  26 MB/s            │  N/A                   │ EXT_CSD_BUS_WIDTH_4
8-bit     │  52 MB/s            │  400 MB/s              │ EXT_CSD_BUS_WIDTH_8
8-bit DDR │  104 MB/s           │  N/A                   │ EXT_CSD_DDR_BUS_WIDTH_8
```

الـ Core بيختار الأعلى اللي الـ host والـ card بيدعموه معاً:
- `host->caps & MMC_CAP_8_BIT_DATA` للـ host side
- `EXT_CSD_CARD_TYPE` للـ card side

---

### الـ Command Queue (CQE) — الـ Optimization المتقدم

الـ eMMC 5.1+ بيدعم **Command Queuing** — بدل ما تبعت command وتستنى، تقدر تبعت لغاية 32 command في نفس الوقت والـ card بتنفذهم internally:

```
بدون CQE:
  [CMD] → wait → [RESP] → [DATA] → [CMD] → wait → [RESP] → [DATA]

مع CQE:
  [CMD1] [CMD2] [CMD3] [CMD4]  →  [DATA1] [DATA3] [DATA2] [DATA4]
                                   (الـ card بترتبهم internally)
```

الـ framework بيدعم ده عبر:
```c
struct mmc_cqe_ops {
    int  (*cqe_enable)(struct mmc_host *host, struct mmc_card *card);
    int  (*cqe_request)(struct mmc_host *host, struct mmc_request *mrq);
    void (*cqe_recovery_start)(struct mmc_host *host);
    void (*cqe_recovery_finish)(struct mmc_host *host);
    /* ... */
};
```

الـ Core بيفعّله لو `EXT_CSD_CMDQ_SUPPORTED` و`MMC_CAP2_CQE` كلاهما موجودين.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Enums والـ Config Options — Cheatsheet

#### MMC Card States (R1_STATE_*)

| Constant | Value | المعنى |
|---|---|---|
| `R1_STATE_IDLE` | 0 | الكارت في idle، مش متحدد |
| `R1_STATE_READY` | 1 | جاهز للـ identification |
| `R1_STATE_IDENT` | 2 | بيتحدد (identification mode) |
| `R1_STATE_STBY` | 3 | standby، متحدد بس مش متاختار |
| `R1_STATE_TRAN` | 4 | **transfer state** — الحالة الطبيعية للـ I/O |
| `R1_STATE_DATA` | 5 | بيبعت data |
| `R1_STATE_RCV` | 6 | بياخد data |
| `R1_STATE_PRG` | 7 | بيبرمج (flash programming) |
| `R1_STATE_DIS` | 8 | disconnect state |

#### R1 Status Bits (error + status)

| Bit | Macro | Type | Clear | المعنى |
|---|---|---|---|---|
| 31 | `R1_OUT_OF_RANGE` | er | c | العنوان برا النطاق |
| 30 | `R1_ADDRESS_ERROR` | erx | c | عنوان غلط |
| 29 | `R1_BLOCK_LEN_ERROR` | er | c | طول البلوك غلط |
| 28 | `R1_ERASE_SEQ_ERROR` | er | c | تسلسل مسح غلط |
| 27 | `R1_ERASE_PARAM` | ex | c | بارامتر مسح غلط |
| 26 | `R1_WP_VIOLATION` | erx | c | انتهاك write protection |
| 25 | `R1_CARD_IS_LOCKED` | sx | a | الكارت محلوق |
| 24 | `R1_LOCK_UNLOCK_FAILED` | erx | c | فشل قفل/فتح |
| 23 | `R1_COM_CRC_ERROR` | er | b | CRC error في الأمر |
| 22 | `R1_ILLEGAL_COMMAND` | er | b | أمر غير مسموح |
| 21 | `R1_CARD_ECC_FAILED` | ex | c | فشل ECC |
| 20 | `R1_CC_ERROR` | erx | c | خطأ في الـ controller |
| 19 | `R1_ERROR` | erx | c | خطأ عام |
| 8 | `R1_READY_FOR_DATA` | sx | a | الكارت جاهز لاستقبال data |
| 7 | `R1_SWITCH_ERROR` | sx | c | خطأ في CMD6 switch |
| 6 | `R1_EXCEPTION_EVENT` | sr | a | حدث استثنائي (BKOPS مثلاً) |
| 5 | `R1_APP_CMD` | sr | c | الأمر الجاي هو ACMD |

> **Type key**: e=error, s=status, r=set in response, x=set during execution. **Clear key**: a=card state, b=next command, c=read to clear.

#### EXT_CSD_CARD_TYPE — Speed Modes

| Flag | Value | السرعة |
|---|---|---|
| `EXT_CSD_CARD_TYPE_HS_26` | bit0 | 26 MHz |
| `EXT_CSD_CARD_TYPE_HS_52` | bit1 | 52 MHz |
| `EXT_CSD_CARD_TYPE_DDR_1_8V` | bit2 | 52 MHz DDR @ 1.8V/3V |
| `EXT_CSD_CARD_TYPE_DDR_1_2V` | bit3 | 52 MHz DDR @ 1.2V |
| `EXT_CSD_CARD_TYPE_HS200_1_8V` | bit4 | 200 MHz SDR @ 1.8V |
| `EXT_CSD_CARD_TYPE_HS200_1_2V` | bit5 | 200 MHz SDR @ 1.2V |
| `EXT_CSD_CARD_TYPE_HS400_1_8V` | bit6 | 200 MHz DDR @ 1.8V |
| `EXT_CSD_CARD_TYPE_HS400_1_2V` | bit7 | 200 MHz DDR @ 1.2V |
| `EXT_CSD_CARD_TYPE_HS400ES` | bit8 | HS400 Enhanced Strobe |

#### Bus Width (EXT_CSD_BUS_WIDTH)

| Constant | Value | المعنى |
|---|---|---|
| `EXT_CSD_BUS_WIDTH_1` | 0 | 1-bit SDR |
| `EXT_CSD_BUS_WIDTH_4` | 1 | 4-bit SDR |
| `EXT_CSD_BUS_WIDTH_8` | 2 | 8-bit SDR |
| `EXT_CSD_DDR_BUS_WIDTH_4` | 5 | 4-bit DDR |
| `EXT_CSD_DDR_BUS_WIDTH_8` | 6 | 8-bit DDR |
| `EXT_CSD_BUS_WIDTH_STROBE` | BIT(7) | Enhanced strobe flag |

#### HS Timing (EXT_CSD_HS_TIMING)

| Constant | Value |
|---|---|
| `EXT_CSD_TIMING_BC` | 0 — backwards compatible |
| `EXT_CSD_TIMING_HS` | 1 — high speed |
| `EXT_CSD_TIMING_HS200` | 2 — HS200 |
| `EXT_CSD_TIMING_HS400` | 3 — HS400 |

#### MMC_SWITCH Access Modes

| Constant | Value | المعنى |
|---|---|---|
| `MMC_SWITCH_MODE_CMD_SET` | 0x00 | غير الـ command set |
| `MMC_SWITCH_MODE_SET_BITS` | 0x01 | اضبط bits محددة |
| `MMC_SWITCH_MODE_CLEAR_BITS` | 0x02 | صفر bits محددة |
| `MMC_SWITCH_MODE_WRITE_BYTE` | 0x03 | اكتب byte كامل |

#### Erase / Trim / Discard Arguments

| Constant | Value | المعنى |
|---|---|---|
| `MMC_ERASE_ARG` | 0x00000000 | مسح عادي |
| `MMC_SECURE_ERASE_ARG` | 0x80000000 | مسح آمن |
| `MMC_TRIM_ARG` | 0x00000001 | trim (mark as unused) |
| `MMC_DISCARD_ARG` | 0x00000003 | discard (hint to card) |
| `MMC_SECURE_TRIM1_ARG` | 0x80000001 | secure trim step 1 |
| `MMC_SECURE_TRIM2_ARG` | 0x80008000 | secure trim step 2 |

#### Card Command Classes (CCC)

| Macro | Bit | الأوامر |
|---|---|---|
| `CCC_BASIC` | 0 | CMD0,1,2,3,4,7,9,10,12,13,15 |
| `CCC_STREAM_READ` | 1 | CMD11 |
| `CCC_BLOCK_READ` | 2 | CMD16,17,18 |
| `CCC_STREAM_WRITE` | 3 | CMD20 |
| `CCC_BLOCK_WRITE` | 4 | CMD16,24,25,26,27 |
| `CCC_ERASE` | 5 | CMD32-39 |
| `CCC_WRITE_PROT` | 6 | CMD28,29,30 |
| `CCC_LOCK_CARD` | 7 | CMD16,42 |
| `CCC_APP_SPEC` | 8 | CMD55,56,ACMD* |
| `CCC_IO_MODE` | 9 | CMD5,39,40,52,53 |
| `CCC_SWITCH` | 10 | CMD6,34-37,50 |

#### مهم جداً — EXT_CSD Fields Index

| Field | Byte | Access | المعنى |
|---|---|---|---|
| `EXT_CSD_CMDQ_MODE_EN` | 15 | R/W | تفعيل Command Queue |
| `EXT_CSD_FLUSH_CACHE` | 32 | W | flush الـ cache |
| `EXT_CSD_CACHE_CTRL` | 33 | R/W | تحكم في الـ cache |
| `EXT_CSD_POWER_OFF_NOTIFICATION` | 34 | R/W | إشعار قبل قطع التيار |
| `EXT_CSD_PART_CONFIG` | 179 | R/W | إعداد الـ partition النشط |
| `EXT_CSD_BUS_WIDTH` | 183 | R/W | عرض الـ bus |
| `EXT_CSD_HS_TIMING` | 185 | R/W | timing mode |
| `EXT_CSD_REV` | 192 | RO | إصدار الـ EXT_CSD |
| `EXT_CSD_CARD_TYPE` | 196 | RO | speed modes المدعومة |
| `EXT_CSD_SEC_CNT` | 212 | RO | إجمالي عدد السيكتورات (4 bytes) |
| `EXT_CSD_BKOPS_EN` | 163 | R/W | تفعيل background ops |
| `EXT_CSD_BKOPS_START` | 164 | W | ابدأ background ops |
| `EXT_CSD_RPMB_MULT` | 168 | RO | حجم الـ RPMB بالـ 128KB |
| `EXT_CSD_FIRMWARE_VERSION` | 254 | RO | إصدار الـ firmware (8 bytes) |
| `EXT_CSD_PRE_EOL_INFO` | 267 | RO | مؤشر اقتراب نهاية العمر |
| `EXT_CSD_CMDQ_DEPTH` | 307 | RO | عمق الـ Command Queue |
| `EXT_CSD_CMDQ_SUPPORT` | 308 | RO | هل الكارت بيدعم CQ؟ |

---

### 1. الـ Structs المهمة

#### `struct mmc_command` (في `core.h`)

**الغرض**: يمثل أمر MMC/SD واحد بيتبعت للكارت.

| Field | Type | المعنى |
|---|---|---|
| `opcode` | `u32` | رقم الأمر (CMD0..CMD60) |
| `arg` | `u32` | الـ argument 32-bit |
| `resp[4]` | `u32[4]` | الـ response (يصل لـ 136-bit في R2) |
| `flags` | `unsigned int` | نوع الـ response + نوع الأمر |
| `retries` | `unsigned int` | عدد إعادات المحاولة |
| `error` | `int` | كود الخطأ (errno) |
| `busy_timeout` | `unsigned int` | timeout لـ busy (ms) |
| `data` | `*mmc_data` | البيانات المرتبطة بالأمر (لو ADTC) |
| `mrq` | `*mmc_request` | الـ request الأب |
| `uhs2_cmd` | `*uhs2_command` | امتداد لبروتوكول UHS-II |
| `has_ext_addr` | `bool` | SDUC extended address support |

**الـ flags المهمة في `mmc_command.flags`**:

```
MMC_RSP_PRESENT  bit0  -- في رد
MMC_RSP_136      bit1  -- رد 136-bit (R2: CID/CSD)
MMC_RSP_CRC      bit2  -- في CRC في الرد
MMC_RSP_BUSY     bit3  -- الكارت ممكن يبعت busy
MMC_RSP_OPCODE   bit4  -- الرد بيحتوي على opcode
MMC_CMD_AC       (0<<5) -- Addressed Command (no data)
MMC_CMD_ADTC     (1<<5) -- Addressed Data Transfer Command
MMC_CMD_BC       (2<<5) -- Broadcast Command (no response)
MMC_CMD_BCR      (3<<5) -- Broadcast Command with Response
```

**الـ response types المحددة سلفاً**:

```c
MMC_RSP_R1   = PRESENT | CRC | OPCODE          // معظم الأوامر
MMC_RSP_R1B  = PRESENT | CRC | OPCODE | BUSY   // أوامر بتشغل flash
MMC_RSP_R2   = PRESENT | 136 | CRC             // CID أو CSD
MMC_RSP_R3   = PRESENT                          // OCR register
```

---

#### `struct mmc_data` (في `core.h`)

**الغرض**: يمثل الـ data payload لأمر data-transfer (ADTC).

| Field | Type | المعنى |
|---|---|---|
| `timeout_ns` | `unsigned int` | data timeout بالـ nanoseconds |
| `timeout_clks` | `unsigned int` | data timeout بالـ clock cycles |
| `blksz` | `unsigned int` | حجم البلوك الواحد بالبايت |
| `blocks` | `unsigned int` | عدد البلوكات |
| `blk_addr` | `unsigned int` | عنوان البلوك الابتدائي |
| `error` | `int` | كود خطأ الـ data |
| `flags` | `unsigned int` | READ/WRITE + CQE flags |
| `bytes_xfered` | `unsigned int` | عدد البايتات اللي اتنقلت فعلاً |
| `stop` | `*mmc_command` | أمر CMD12 لإيقاف التحويل |
| `mrq` | `*mmc_request` | الـ request الأب |
| `sg_len` | `unsigned int` | عدد عناصر الـ scatter list |
| `sg_count` | `int` | عدد عناصر الـ SG المـ mapped |
| `sg` | `*scatterlist` | الـ I/O scatter-gather list |
| `host_cookie` | `s32` | private data للـ host controller |

**الـ data flags**:

```
MMC_DATA_WRITE    bit8   -- write to card
MMC_DATA_READ     bit9   -- read from card
MMC_DATA_QBR      bit10  -- CQE queue barrier
MMC_DATA_PRIO     bit11  -- CQE high priority request
MMC_DATA_REL_WR   bit12  -- Reliable Write (power-loss safe)
MMC_DATA_DAT_TAG  bit13  -- Data Tag (hint for garbage collection)
MMC_DATA_FORCED_PRG bit14 -- Force immediate program
```

---

#### `struct mmc_request` (في `core.h`)

**الغرض**: يجمع مجموعة من الأوامر في "request" واحد بيتبعت كوحدة.

| Field | Type | المعنى |
|---|---|---|
| `sbc` | `*mmc_command` | CMD23 (SET_BLOCK_COUNT) قبل الـ data |
| `cmd` | `*mmc_command` | الأمر الرئيسي |
| `data` | `*mmc_data` | الـ data (لو موجودة) |
| `stop` | `*mmc_command` | CMD12 لإنهاء الـ multiblock |
| `completion` | `struct completion` | للـ synchronous waiting |
| `cmd_completion` | `struct completion` | للانتظار بعد الأمر فقط |
| `done` | function pointer | callback لما الـ request ينتهي |
| `recovery_notifier` | function pointer | callback لما يحتاج recovery (CQE) |
| `host` | `*mmc_host` | الـ host اللي شغال عليه |
| `cap_cmd_during_tfr` | `bool` | يسمح بأوامر تانية أثناء الـ transfer |
| `tag` | `int` | CQE task ID |
| `uhs2_cmd` | `uhs2_command` | امتداد UHS-II مضمّن |

---

#### `struct mmc_cid` (في `card.h`)

**الغرض**: Card Identification — معلومات هوية الكارت.

| Field | المعنى |
|---|---|
| `manfid` | Manufacturer ID (8-bit) |
| `prod_name[8]` | اسم المنتج (ASCII) |
| `prv` | Product Revision |
| `serial` | الرقم التسلسلي (32-bit) |
| `oemid` | OEM/Application ID |
| `year` / `month` | تاريخ التصنيع |
| `hwrev` / `fwrev` | إصدار الـ HW والـ FW |

---

#### `struct mmc_csd` (في `card.h`)

**الغرض**: Card Specific Data — خصائص الكارت الجوهرية.

| Field | المعنى |
|---|---|
| `structure` | إصدار الـ CSD (0=v1.0, 1=v1.1, 2=v1.2, 3=EXT_CSD) |
| `mmca_vsn` | إصدار مواصفة MMC |
| `cmdclass` | الـ Command Classes المدعومة (CCC bitmask) |
| `taac_clks` / `taac_ns` | Access Time |
| `c_size` | Device Size |
| `max_dtr` | Maximum Data Transfer Rate |
| `erase_size` | وحدة المسح بالسيكتورات |
| `read_blkbits` / `write_blkbits` | log2 of block size |
| `capacity` | السعة الكلية (sector_t) |
| `read_partial` / `write_partial` | هل partial blocks مسموحة؟ |
| `dsr_imp` | هل الكارت بيعمل DSR؟ |

---

#### `struct mmc_ext_csd` (في `card.h`)

**الغرض**: Extended CSD Register — 512 byte من الـ config والـ capabilities في eMMC v4+. الأهم في الـ subsystem.

أهم الـ fields المُفسَّرة:

| Field | المعنى |
|---|---|
| `rev` | إصدار الـ EXT_CSD |
| `part_config` | الـ partition النشط حالياً (ACC_BOOT0/GP/RPMB) |
| `cache_ctrl` | هل الـ cache مفعّل؟ |
| `rst_n_function` | إعداد الـ hardware reset pin |
| `hs_max_dtr` | أقصى data rate في HS mode |
| `hs200_max_dtr` | أقصى data rate في HS200 |
| `sectors` | إجمالي عدد السيكتورات |
| `hc_erase_size` | حجم وحدة المسح الـ HC |
| `bkops` / `man_bkops_en` / `auto_bkops_en` | Background Operations |
| `cmdq_en` / `cmdq_support` / `cmdq_depth` | Command Queue |
| `fwrev[8]` | إصدار الـ firmware |
| `pre_eol_info` | مؤشر قرب نهاية العمر |
| `device_life_time_est_typ_a/b` | تقدير العمر المتبقي لكل نوع نانو-flash |
| `partition_setting_completed` | هل التقسيم اتأكد بشكل دائم؟ |
| `feature_support` | `MMC_DISCARD_FEATURE` وغيره |

---

#### `struct mmc_card` (في `card.h`)

**الغرض**: يمثل كارت MMC/SD/SDIO كامل — الـ struct المحوري في الـ subsystem.

| Field | Type | المعنى |
|---|---|---|
| `host` | `*mmc_host` | الـ host controller اللي الكارت متوصل بيه |
| `dev` | `struct device` | الكارت كـ device في الـ device model |
| `ocr` | `u32` | Operating Conditions Register |
| `rca` | `unsigned int` | Relative Card Address |
| `type` | `unsigned int` | MMC/SD/SDIO/SD_COMBO |
| `state` | `unsigned int` | حالة الكارت |
| `quirks` | `unsigned int` | bitmask لـ workarounds للكروت المعيوبة |
| `erase_size/shift/pref_erase` | — | بارامترات المسح |
| `raw_cid[4]` / `raw_csd[4]` | `u32[]` | raw registers |
| `cid` | `struct mmc_cid` | Card ID |
| `csd` | `struct mmc_csd` | Card Specific Data |
| `ext_csd` | `struct mmc_ext_csd` | Extended CSD (eMMC فقط) |
| `scr` / `ssr` / `sw_caps` | — | SD-specific info |
| `ext_power` / `ext_perf` | `struct sd_ext_reg` | SD Extension Registers |
| `uhs2_config` | `struct sd_uhs2_config` | UHS-II config |
| `sdio_funcs` | `unsigned int` | عدد الـ SDIO functions |
| `cccr` | `struct sdio_cccr` | SDIO Common Card Control |
| `cis` | `struct sdio_cis` | SDIO Common Information Structure |
| `sdio_func[7]` | `*sdio_func[]` | الـ SDIO functions (devices) |
| `part[7]` | `struct mmc_part[]` | الـ physical partitions (boot/GP/RPMB) |
| `complete_wq` | `*workqueue_struct` | private workqueue للكارت |
| `mmc_avail_type` | `unsigned int` | تقاطع قدرات الكارت والـ host |

---

#### `struct mmc_part` (في `card.h`)

**الغرض**: يمثل partition فيزيائي واحد في الكارت.

| Field | المعنى |
|---|---|
| `size` | الحجم بالبايت |
| `part_cfg` | إعداد الـ partition (ACC value) |
| `name[20]` | اسم الـ partition |
| `force_ro` | هل read-only إجبارياً؟ |
| `area_type` | MAIN/BOOT/GP/RPMB bitmask |

---

#### `struct uhs2_command` (في `core.h`)

**الغرض**: امتداد لـ `mmc_command` لبروتوكول UHS-II اللي بيستخدم packet-based interface بدل الـ legacy CMD/DAT lines.

| Field | المعنى |
|---|---|
| `header` / `arg` | الـ UHS2 packet header وargument |
| `payload[2]` | payload للأوامر UHS2-native |
| `tmode_half_duplex` | هل التحويل half-duplex؟ |
| `uhs2_resp[20]` | raw response |

---

### 2. مخطط العلاقات بين الـ Structs

```
                    ┌──────────────────────────────────────────┐
                    │             struct mmc_host               │
                    │  (host.h — الـ controller hardware)       │
                    └──────────┬───────────────────────────────┘
                               │ host->card  (1:1)
                               ▼
                    ┌──────────────────────────────────────────┐
                    │             struct mmc_card               │
                    │  .host  ──────────────────────────────►  │
                    │  .dev   (struct device)                   │
                    │  .cid   (struct mmc_cid)                  │
                    │  .csd   (struct mmc_csd)                  │
                    │  .ext_csd (struct mmc_ext_csd) [eMMC]     │
                    │  .scr / .ssr / .sw_caps  [SD]             │
                    │  .cccr / .cis            [SDIO]           │
                    │  .part[7] (struct mmc_part[])             │
                    │  .sdio_func[7] ──────────────────────►   │
                    └──────────────────────────────────────────┘
                         ▲
                         │ mrq->host
                    ┌──────────────────────────────────────────┐
                    │           struct mmc_request              │
                    │  .sbc  ──► struct mmc_command (CMD23)     │
                    │  .cmd  ──► struct mmc_command (main CMD)  │
                    │  .data ──► struct mmc_data                │
                    │  .stop ──► struct mmc_command (CMD12)     │
                    │  .completion (struct completion)          │
                    │  .done()  callback                        │
                    └──────────────────────────────────────────┘
                                      │
                    ┌─────────────────▼────────────────────────┐
                    │           struct mmc_command              │
                    │  .opcode / .arg / .resp[4]                │
                    │  .flags  (RSP type | CMD type)            │
                    │  .data ──► struct mmc_data                │
                    │  .uhs2_cmd ──► struct uhs2_command        │
                    └──────────────────────────────────────────┘
                                      │
                    ┌─────────────────▼────────────────────────┐
                    │            struct mmc_data                │
                    │  .sg ──► struct scatterlist               │
                    │  .stop ──► struct mmc_command (CMD12)     │
                    │  .blksz / .blocks / .flags                │
                    └──────────────────────────────────────────┘

struct mmc_card ──► struct mmc_part[7]
                     .area_type: MAIN | BOOT | GP | RPMB

struct mmc_card ──► struct mmc_ext_csd
                     يحتوي على كل الـ capabilities
                     (speed, partitions, cmdq, bkops, ...)
```

---

### 3. Lifecycle Diagrams

#### دورة حياة الكارت — من الاكتشاف لغاية الاستخدام

```
[Power On / Card Insert]
         │
         ▼
  mmc_rescan() triggered
  (host detects card)
         │
         ▼
  CMD0: GO_IDLE_STATE
  ─── card enters IDLE state ───
         │
         ▼
  CMD1 (eMMC) / CMD8 (SD): voltage check
  ─── card enters READY state ───
         │
         ▼
  CMD2: ALL_SEND_CID
  ─── card enters IDENT state ───
  [struct mmc_cid populated from R2 response]
         │
         ▼
  CMD3: SET_RELATIVE_ADDR (eMMC) / SEND_RELATIVE_ADDR (SD)
  ─── card.rca assigned, enters STBY state ───
         │
         ▼
  CMD9: SEND_CSD → parse into struct mmc_csd
  CMD10: SEND_CID (confirm)
         │
         ▼
  CMD7: SELECT_CARD (with card.rca)
  ─── card enters TRAN state ───
         │
         ▼
  CMD8: SEND_EXT_CSD (eMMC only)
  [struct mmc_ext_csd populated — 512 bytes]
         │
         ▼
  mmc_init_card():
    - set bus width (CMD6 → EXT_CSD_BUS_WIDTH)
    - set timing mode (CMD6 → EXT_CSD_HS_TIMING)
    - enable cache if supported
    - enable CMDQ if supported
    - enumerate partitions → struct mmc_part[]
         │
         ▼
  mmc_add_card():
    device_add(&card->dev)
    ─── block driver probes ───
         │
         ▼
  [Ready for I/O]
  mmc_blk_issue_rq()
    → struct mmc_request built
    → mmc_wait_for_req() called
         │
         ▼
  [Card Removed / Power Off]
  mmc_remove_card():
    device_del(&card->dev)
    mmc_card_set_dead()
    kfree(card)
```

#### دورة حياة الـ Request

```
[Block Layer: bio arrives]
         │
         ▼
  mmc_blk_issue_rw_rq()
    allocate struct mmc_request
    allocate struct mmc_command (cmd)
    allocate struct mmc_data
    set sg list from bio
         │
         ▼
  mmc_wait_for_req(host, mrq)
         │
         ├─ mrq->sbc = CMD23 (if reliable write or multi-block)
         ├─ mrq->cmd = CMD18/CMD25 (read/write multiple)
         ├─ mrq->data → sg → DMA
         └─ mrq->stop = CMD12 (stop transmission)
         │
         ▼
  host->ops->request(host, mrq)   ← host driver takes over
         │
         ▼
  [DMA transfer completes]
  mmc_request_done(host, mrq)
    → mrq->done(mrq) callback
    → complete(&mrq->completion)
         │
         ▼
  [Error handling if cmd->error or data->error]
  check R1 status bits:
    R1_OUT_OF_RANGE → -ERANGE
    R1_COM_CRC_ERROR → -EILSEQ
    timeout → -ETIMEDOUT
         │
         ▼
  [Request freed, next request]
```

---

### 4. Call Flow Diagrams

#### CMD6 — MMC_SWITCH (تغيير Bus Width مثلاً)

```
mmc_set_bus_width(card, MMC_BUS_WIDTH_8)
  │
  ▼
mmc_switch(card, EXT_CSD_CMD_SET_NORMAL,
           EXT_CSD_BUS_WIDTH,
           EXT_CSD_BUS_WIDTH_8,
           card->ext_csd.generic_cmd6_time)
  │
  ├─ build mmc_command:
  │    opcode = MMC_SWITCH (6)
  │    arg    = [access=WRITE_BYTE][index=183][value=2][cmdset=0]
  │    flags  = MMC_RSP_R1B | MMC_CMD_AC
  │
  ▼
mmc_wait_for_cmd(host, cmd, retries)
  │
  ▼
host->ops->request(host, mrq)
  │
  ▼ [hardware sends CMD6 on CMD line]
  │
  ▼ [card responds R1b — may pull DAT0 low (busy)]
  │
  ▼ [host waits for DAT0 to go high = card done]
  │
  ▼
check cmd->resp[0] for R1_SWITCH_ERROR (bit7)
  │
  ▼
mmc_set_timing(host, MMC_TIMING_MMC_HS)
host->ops->set_ios(host, &host->ios)
```

#### Read I/O Flow — HS200

```
submit_bio(READ, sector=X, len=512K)
  │
  ▼
mmc_blk_issue_rw_rq()
  │
  ├─ mrq.sbc: CMD23 arg=[blocks=1024 | REL_WR_off]
  ├─ mrq.cmd: CMD18 (READ_MULTIPLE_BLOCK) arg=X
  ├─ mrq.data: flags=MMC_DATA_READ, blocks=1024, blksz=512
  │             sg → kernel pages
  └─ mrq.stop: CMD12 (STOP_TRANSMISSION)
  │
  ▼
mmc_wait_for_req(host, mrq)
  │
  ▼
host->ops->request(host, mrq)
  │
  ├── [host sends CMD23 → card ACKs with R1]
  ├── [host sends CMD18 → card ACKs with R1]
  ├── [card starts streaming data on DAT[7:0]]
  ├── [ADMA/SDMA fills sg pages via DMA]
  └── [host auto-sends CMD12 after last block]
  │
  ▼
mmc_request_done()
  mrq->done(mrq)
  complete(&mrq->completion)
  │
  ▼
blk_mq_complete_request()
```

#### BKOPS Flow

```
[card sets R1_EXCEPTION_EVENT in status response]
  │
  ▼
mmc_check_bkops(card)
  │
  ├─ CMD13: SEND_STATUS → check R1_EXCEPTION_EVENT
  ├─ CMD8: SEND_EXT_CSD → read byte 54 (EXP_EVENTS_STATUS)
  └─ if EXT_CSD_URGENT_BKOPS set:
       │
       ▼
     mmc_start_bkops(card, is_synchronous)
       │
       ├─ CMD6: EXT_CSD_BKOPS_START = 1
       └─ if synchronous: wait for card to finish
          (poll CMD13 until R1_READY_FOR_DATA)
```

#### Command Queue (CQE) Flow

```
[Block layer: N requests queued]
  │
  ▼
cqhci_request() — CQE host driver
  │
  ├─ CMD44: QUE_TASK_PARAMS [task_id, block_count, flags]
  ├─ CMD45: QUE_TASK_ADDR  [sector address]
  │     (repeat for each task up to cmdq_depth)
  │
  ├─ CMD46/47: EXECUTE_READ/WRITE_TASK [task_id]
  │     ← data transfer on DAT lines
  │
  └─ [card may reorder execution internally]
  │
  ▼
[Completion interrupt per task]
cqhci_irq()
  │
  ├─ read task completion status
  ├─ call mmc_cqe_request_done(host, mrq)
  └─ mrq->done(mrq) or mrq->recovery_notifier(mrq)
```

---

### 5. Locking Strategy

الـ `mmc.h` نفسه header-only بيحتوي على constants وstatic inlines — مفيش locks فيه مباشرة. لكن الـ locking في الـ MMC subsystem بييجي من الـ structs المرتبطة:

#### الـ Locks الرئيسية في MMC Layer

```
┌─────────────────────────────────────────────────────────────┐
│  struct mmc_host (في host.h)                                 │
│                                                             │
│  .lock (spinlock_t)                                         │
│    ↳ يحمي: ios state, clock, bus state,                     │
│            كل تعديل على host->ios                           │
│    ↳ يُمسك: من host driver في الـ interrupt context         │
│                                                             │
│  .slot.lock (mutex — في بعض implementations)               │
│    ↳ يحمي: card detection state                             │
│                                                             │
│  claim_host / release_host (via mmc_claim_host)             │
│    ↳ هو mutex بيضمن إن مستخدم واحد بس يبعت requests        │
│    ↳ ممنوع يتمسك من interrupt context                       │
└─────────────────────────────────────────────────────────────┘
```

#### ترتيب الـ Locks (Lock Ordering)

```
mmc_claim_host()          ← الأعلى (blocking, can sleep)
  └─ host->lock            ← spinlock سريع
       └─ [hardware access]

NEVER: host->lock ثم mmc_claim_host()  ← deadlock!
```

#### الـ Context والـ Sleep

```
mmc_wait_for_req()
  ├── يستدعي host->ops->request() → IRQ context لو non-blocking
  ├── يستعمل wait_for_completion() → يتعطل (can sleep)
  └── يُستخدم فقط من process context (mmc_claim_host مُمسوك)

mmc_request_done()        ← يتستدعى من interrupt context (host IRQ)
  ├── يشغّل mrq->done()    ← يجب أن يكون atomic-safe
  └── complete(&mrq->completion) ← يصحّي الـ waiter
```

#### ما تحميه كل lock في السياق الكامل

| Lock | ما يحميه | Context |
|---|---|---|
| `host->lock` (spinlock) | `host->ios`, clock/bus state, queue head | IRQ + process |
| `mmc_claim_host()` (mutex/semaphore) | الـ host كاملاً — يمنع concurrent requests | process only |
| `card->ext_csd` fields | يُقرأ بعد init فقط (read-mostly, no lock needed) | process |
| `host->claimed` flag | يُحدَّث تحت `host->lock` | IRQ + process |

#### ملاحظة على الـ R1 Status و Locking

الـ status bits في `R1` (المُعرَّفة في `mmc.h`) بييجوا في response للأمر — مفيش lock بيحمي parsing الـ response لأنه بييحصل بعد ما الأمر يخلص وقبل ما الـ lock يتفك. الـ `mmc_ready_for_data()` inline function بتتقرأ من `cmd->resp[0]` اللي بيكون read-only بعد ما الـ hardware يكتبه.

```c
/* safe to call without lock — resp[] is written by HW, then read-only */
static inline bool mmc_ready_for_data(u32 status)
{
    return status & R1_READY_FOR_DATA &&
           R1_CURRENT_STATE(status) == R1_STATE_TRAN;
}
```
## Phase 4: شرح الـ Functions

---

### ملخص الـ Functions والـ Macros — Cheatsheet

#### Inline Functions

| Function | Signature | الغرض |
|---|---|---|
| `mmc_op_multi` | `bool mmc_op_multi(u32 opcode)` | يتحقق إذا كان الـ opcode عملية multi-block |
| `mmc_op_tuning` | `bool mmc_op_tuning(u32 opcode)` | يتحقق إذا كان الـ opcode عملية tuning |
| `mmc_ready_for_data` | `bool mmc_ready_for_data(u32 status)` | يتحقق إذا كانت الكارت جاهزة لاستقبال/إرسال بيانات |

#### Utility Macros

| Macro | الغرض |
|---|---|
| `R1_STATUS(x)` | يستخلص بتات الـ error status من الـ R1 response |
| `R1_CURRENT_STATE(x)` | يستخلص الـ current state (4 bits) من الـ R1 response |
| `mmc_driver_type_mask(n)` | يبني الـ bitmask الخاص بـ driver strength رقم n |

---

### تصنيف المحتوى

الملف `mmc.h` هو **header-only** — ما فيهوش functions قابلة للـ link بمعنى الكلمة، كل حاجة فيه إما:
1. **`#define` macros** — command indices, register bit fields, EXT_CSD offsets.
2. **`static inline` functions** — ثلاث functions بسيطة مدمجة في كل translation unit تستخدمها.

الـ grouping هيكون:
- **Group 1**: Command Opcode Helpers — `mmc_op_multi`, `mmc_op_tuning`
- **Group 2**: Status & State Helpers — `mmc_ready_for_data`, `R1_STATUS`, `R1_CURRENT_STATE`
- **Group 3**: Utility Macros — `mmc_driver_type_mask`

---

### Group 1: Command Opcode Helpers

الـ MMC core layer بيحتاج يعرف طبيعة الـ command اللي بيبعته — هل هو multi-block transfer أو tuning sequence؟ السبب إن لكل نوع معالجة مختلفة في الـ host controller وفي الـ error recovery path.

---

#### `mmc_op_multi`

```c
static inline bool mmc_op_multi(u32 opcode)
{
    return opcode == MMC_WRITE_MULTIPLE_BLOCK ||
           opcode == MMC_READ_MULTIPLE_BLOCK;
}
```

**بتعمل إيه:**
بتشيك إذا كان الـ `opcode` هو `CMD18` (READ_MULTIPLE_BLOCK) أو `CMD25` (WRITE_MULTIPLE_BLOCK). الـ multi-block commands بتحتاج `CMD12` (STOP_TRANSMISSION) أو مكانيزم الـ `SET_BLOCK_COUNT` (CMD23) عشان تنهيها، وبيتعامل معاها الـ host driver بشكل مختلف في الـ DMA setup.

**Parameters:**
- `opcode` — الـ command index بالـ `u32`، بيجي من `struct mmc_command::opcode`.

**Return value:**
- `true` لو الـ opcode هو multi-block read أو write.
- `false` في أي حالة تانية.

**Key details:**
- **No locking** — pure inline predicate، safe to call from any context.
- **Caller context:** بيتم استدعاؤها من `mmc_blk_rw_rq_prep()` وفي الـ host driver لتحديد هل محتاج `CMD12` أو لا، وكمان من `mmc_request_done()` في error recovery.

---

#### `mmc_op_tuning`

```c
static inline bool mmc_op_tuning(u32 opcode)
{
    return opcode == MMC_SEND_TUNING_BLOCK ||
           opcode == MMC_SEND_TUNING_BLOCK_HS200;
}
```

**بتعمل إيه:**
بتشيك إذا كان الـ command هو tuning sequence — إما `CMD19` (SEND_TUNING_BLOCK) المستخدم في HS200، أو `CMD21` (SEND_TUNING_BLOCK_HS200). الـ tuning procedure بيعملها الـ host controller عشان يحدد الـ sampling point الأمثل للـ clock في الـ high-speed modes.

**ملاحظة مهمة:** `CMD19` معرف مرتين في الملف — مرة كـ `MMC_BUS_TEST_W` ومرة كـ `MMC_SEND_TUNING_BLOCK`. ده مقصود — نفس الـ opcode number لكن في سياق مختلف (bus test vs. tuning)، والـ context بيحدد المعنى.

**Parameters:**
- `opcode` — الـ command index بالـ `u32`.

**Return value:**
- `true` لو كانت tuning command.
- `false` غير ذلك.

**Key details:**
- **No locking** — pure predicate.
- **Caller context:** بيتم استدعاؤها من الـ host driver tuning callback (مثلاً `sdhci_execute_tuning()`) وفي `mmc_execute_tuning()` في الـ core، عشان يتعامل مع الـ data direction والـ response type بشكل صح.

---

### Group 2: Status & State Helpers

الـ MMC R1 response هو 32-bit register بيحتوي على error bits وstate bits. الـ macros والـ inline functions دي بتسهل استخراج المعلومات منه بشكل type-safe.

---

#### `mmc_ready_for_data`

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

**بتعمل إيه:**
بتتحقق من إن الكارت في الحالة الصح لاستقبال data transfer command. بتجمع بين شرطين: إن الـ `READY_FOR_DATA` bit (bit 8) set، وإن الكارت في الـ `TRAN` state (state 4). الـ comment في الكود بيشرح السبب: بعض الكروت بتعمل mishandle للـ status bits، فالـ double check ضروري.

**الـ State Machine:**
```
IDLE → READY → IDENT → STBY → TRAN → DATA/RCV/PRG → TRAN
                                ↑_________________________|
```
الكارت لازم تكون في `TRAN` state عشان تقبل read/write command جديد.

**Parameters:**
- `status` — الـ 32-bit R1 response المقروء من CMD13 (SEND_STATUS) أو من أي command response.

**Return value:**
- `true` لو الكارت ready for data في الـ TRAN state.
- `false` لو مش ready (مثلاً: لسه شغالة PRG state، أو busy).

**Key details:**
- **No locking.**
- **Error paths:** لو رجعت `false`، الـ caller عادةً بيدخل polling loop ببعت CMD13 مرة تانية بعد delay.
- **Caller context:** بيتم استدعاؤها من `mmc_poll_for_busy()` وفي الـ block layer بعد write operations لضمان إن الكارت خلصت البرمجة.

---

#### `R1_STATUS(x)` — Macro

```c
#define R1_STATUS(x)  (x & 0xFFF9A000)
```

**بيعمل إيه:**
بيعمل mask على الـ R1 response ويستخرج بتات الـ error/status فقط، بيحذف الـ state field وبتات الـ flags التانية. الـ mask `0xFFF9A000` بيحتوي على كل الـ error bits من bit 13 لفوق (ما عدا الـ state bits من 9 لـ 12).

**Parameters:**
- `x` — الـ 32-bit R1 response.

**Return value:**
- الـ masked value — لو صفر يعني ما فيش errors في الـ status bits المعيّنة.

---

#### `R1_CURRENT_STATE(x)` — Macro

```c
#define R1_CURRENT_STATE(x)  ((x & 0x00001E00) >> 9)
```

**بيعمل إيه:**
بيستخرج الـ 4-bit current state field من الـ R1 response (bits 12:9). القيم المحتملة:

| Value | State | المعنى |
|---|---|---|
| 0 | `R1_STATE_IDLE` | Idle state |
| 1 | `R1_STATE_READY` | Ready state |
| 2 | `R1_STATE_IDENT` | Identification state |
| 3 | `R1_STATE_STBY` | Standby state |
| 4 | `R1_STATE_TRAN` | Transfer state — الحالة الطبيعية للعمل |
| 5 | `R1_STATE_DATA` | Sending data state |
| 6 | `R1_STATE_RCV` | Receiving data state |
| 7 | `R1_STATE_PRG` | Programming state — الكارت بتكتب في الـ flash |
| 8 | `R1_STATE_DIS` | Disconnect state |

**Parameters:**
- `x` — الـ 32-bit R1 response.

**Return value:**
- Integer من 0 لـ 15 يمثل الـ current state.

---

### Group 3: Utility Macros

---

#### `mmc_driver_type_mask(n)` — Macro

```c
#define mmc_driver_type_mask(n)  (1 << (n))
```

**بيعمل إيه:**
بيبني الـ bitmask الخاص بـ driver strength type رقم n. الـ eMMC spec بيحدد أربع driver strengths:

| n | Type | المعنى |
|---|---|---|
| 0 | Type B | Default (50 ohm) |
| 1 | Type A | 33 ohm |
| 2 | Type C | 66 ohm |
| 3 | Type D | 100 ohm |

بيتم استخدام الـ mask في المقارنة مع `EXT_CSD_DRIVER_STRENGTH` field (byte 197) من الـ EXT_CSD عشان تعرف هل الكارت بتدعم الـ driver strength المطلوب.

**Parameters:**
- `n` — رقم الـ driver strength type (0-3).

**Return value:**
- الـ bitmask المناسب: `1`, `2`, `4`, أو `8`.

**Caller context:** بيتم استخدامه في `mmc_select_hs400es()` وفي الـ tuning-related code لاختيار الـ driver strength المناسب للـ host.

---

### ملاحظات على الـ R1 Error Bits

الـ R1 response بيحتوي على bits كتير بتدل على errors مختلفة، دي أهمها:

| Bit | Macro | النوع | وقت الـ clear |
|---|---|---|---|
| 31 | `R1_OUT_OF_RANGE` | er,c | clear by read |
| 30 | `R1_ADDRESS_ERROR` | erx,c | clear by read |
| 29 | `R1_BLOCK_LEN_ERROR` | er,c | clear by read |
| 23 | `R1_COM_CRC_ERROR` | er,b | يتكلير مع أول command valid |
| 22 | `R1_ILLEGAL_COMMAND` | er,b | يتكلير مع أول command valid |
| 21 | `R1_CARD_ECC_FAILED` | ex,c | clear by read |
| 20 | `R1_CC_ERROR` | erx,c | clear by read |
| 19 | `R1_ERROR` | erx,c | general error |
| 7 | `R1_SWITCH_ERROR` | sx,c | يظهر بعد CMD6 |
| 6 | `R1_EXCEPTION_EVENT` | sr,a | exceptional event pending |

**أنواع الـ bits:**
- **e**: error bit — يدل على error حصل.
- **s**: status bit — يدل على حالة.
- **r**: يتعبّى في الـ actual command response.
- **x**: يتعبّى أثناء execution — الـ host لازم يـ poll بـ CMD13.

---

### ملاحظة على الـ SPI Mode

في SPI mode، الـ R1 format مختلف تماماً — byte واحد بدل 32 bits:

| Bit | Macro | المعنى |
|---|---|---|
| 0 | `R1_SPI_IDLE` | Card is in idle state |
| 2 | `R1_SPI_ILLEGAL_COMMAND` | Illegal command |
| 3 | `R1_SPI_COM_CRC` | CRC error |
| 6 | `R1_SPI_PARAMETER` | Parameter error |

الـ R2 byte (للـ SEND_STATUS فقط في SPI) بيحتوي على تفاصيل زيادة مثل `R2_SPI_CARD_LOCKED` و`R2_SPI_WP_VIOLATION`. الـ inline functions الثلاثة في الملف بتشتغل مع الـ native mode فقط — مش SPI.

---

### الـ EXT_CSD — Extended Card Specific Data

الـ EXT_CSD هو register بحجم 512 bytes بيتقرأ بـ CMD8 (SEND_EXT_CSD). الـ header بيحدد offsets لأهم الـ fields:

| Field | Offset | Access | الغرض |
|---|---|---|---|
| `EXT_CSD_REV` | 192 | RO | EXT_CSD revision |
| `EXT_CSD_CARD_TYPE` | 196 | RO | Supported speed modes |
| `EXT_CSD_HS_TIMING` | 185 | R/W | الـ timing mode الحالي |
| `EXT_CSD_BUS_WIDTH` | 183 | R/W | عرض الـ bus |
| `EXT_CSD_PART_CONFIG` | 179 | R/W | Active partition |
| `EXT_CSD_CMDQ_SUPPORT` | 308 | RO | هل Command Queue مدعومة |
| `EXT_CSD_CMDQ_MODE_EN` | 15 | R/W | تفعيل Command Queue |
| `EXT_CSD_CACHE_CTRL` | 33 | R/W | تفعيل الـ cache |
| `EXT_CSD_SEC_CNT` | 212 | RO | عدد الـ sectors (4 bytes) |
| `EXT_CSD_BKOPS_EN` | 163 | R/W | تفعيل Background Operations |
| `EXT_CSD_RPMB_MULT` | 168 | RO | حجم الـ RPMB partition |

تعديل الـ EXT_CSD بيتم عبر CMD6 (MMC_SWITCH) باستخدام الـ access modes:

```c
/* MMC_SWITCH argument format:
 * [25:24] = access mode (WRITE_BYTE = 0x03)
 * [23:16] = EXT_CSD byte index
 * [15:08] = value
 * [02:00] = command set (0 = normal)
 */
#define MMC_SWITCH_MODE_WRITE_BYTE  0x03
```

---

### الـ Erase/Trim/Discard Arguments

| Macro | Value | الغرض |
|---|---|---|
| `MMC_ERASE_ARG` | 0x00000000 | Erase عادي |
| `MMC_SECURE_ERASE_ARG` | 0x80000000 | Secure erase (overwrite then erase) |
| `MMC_TRIM_ARG` | 0x00000001 | Trim — إخبار الكارت بالـ unused blocks |
| `MMC_DISCARD_ARG` | 0x00000003 | Discard — أسرع من Trim، ما بيضمنش الـ physical erase |
| `MMC_SECURE_TRIM1_ARG` | 0x80000001 | المرحلة الأولى من الـ secure trim |
| `MMC_SECURE_TRIM2_ARG` | 0x80008000 | المرحلة التانية من الـ secure trim |

الـ `MMC_SECURE_ARGS` mask (0x80000000) بيستخدم للتحقق إذا كانت العملية secure أم لا.
الـ `MMC_TRIM_OR_DISCARD_ARGS` mask (0x00008003) بيستخدم للتمييز بين العمليات الـ trim/discard من الـ erase العادي.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. مدخلات الـ debugfs المتعلقة بـ MMC

الـ MMC subsystem بيعرض معلومات تفصيلية تحت `/sys/kernel/debug/mmc*`:

```bash
# اعرف كل الـ MMC hosts الموجودة
ls /sys/kernel/debug/

# اقرأ الـ EXT_CSD كامل (512 byte) — الأهم في debugging
cat /sys/kernel/debug/mmc0/mmc0:0001/ext_csd

# مثال: استخرج byte رقم 196 (EXT_CSD_CARD_TYPE)
hexdump -C /sys/kernel/debug/mmc0/mmc0:0001/ext_csd | awk 'NR==13'
```

**أهم الـ bytes في EXT_CSD للـ debugging:**

| Byte | الاسم | القيمة المتوقعة | ما يعنيه |
|------|-------|-----------------|----------|
| 192 | `EXT_CSD_REV` | 5–8 | إصدار الـ spec |
| 196 | `EXT_CSD_CARD_TYPE` | 0xFF → HS400ES | أقصى سرعة مدعومة |
| 185 | `EXT_CSD_HS_TIMING` | 0=BC, 1=HS, 2=HS200, 3=HS400 | الـ timing الحالي |
| 183 | `EXT_CSD_BUS_WIDTH` | 0=1bit, 1=4bit, 2=8bit, 6=DDR8 | عرض الـ bus |
| 179 | `EXT_CSD_PART_CONFIG` | 3 bits low | الـ partition النشطة |
| 246 | `EXT_CSD_BKOPS_STATUS` | 0=لا يحتاج, 2=عاجل | حالة الـ background ops |
| 267 | `EXT_CSD_PRE_EOL_INFO` | 1=طبيعي, 3=urgent | عمر الـ NAND |
| 268 | `EXT_CSD_DEVICE_LIFE_TIME_EST_TYP_A` | 1–10 (×10%) | استهلاك النوع A |
| 269 | `EXT_CSD_DEVICE_LIFE_TIME_EST_TYP_B` | 1–10 (×10%) | استهلاك النوع B |

```bash
# طريقة سريعة لقراءة byte معين من EXT_CSD
EXT_CSD=/sys/kernel/debug/mmc0/mmc0:0001/ext_csd
python3 -c "
data = open('$EXT_CSD','rb').read()
print(f'CARD_TYPE (196): 0x{data[196]:02x}')
print(f'HS_TIMING (185): 0x{data[185]:02x}')
print(f'BUS_WIDTH (183): 0x{data[183]:02x}')
print(f'REV       (192): 0x{data[192]:02x}')
print(f'PRE_EOL   (267): 0x{data[267]:02x}')
print(f'LIFE_A    (268): 0x{data[268]:02x}')
print(f'LIFE_B    (269): 0x{data[269]:02x}')
"
```

---

#### 2. مدخلات الـ sysfs

```bash
# معلومات الكارت الأساسية
cat /sys/class/mmc_host/mmc0/mmc0:0001/cid        # Card IDentification
cat /sys/class/mmc_host/mmc0/mmc0:0001/csd        # Card Specific Data
cat /sys/class/mmc_host/mmc0/mmc0:0001/date        # تاريخ التصنيع
cat /sys/class/mmc_host/mmc0/mmc0:0001/fwrev       # firmware version
cat /sys/class/mmc_host/mmc0/mmc0:0001/hwrev       # hardware revision
cat /sys/class/mmc_host/mmc0/mmc0:0001/manfid      # Manufacturer ID
cat /sys/class/mmc_host/mmc0/mmc0:0001/name        # اسم الكارت
cat /sys/class/mmc_host/mmc0/mmc0:0001/oemid       # OEM ID
cat /sys/class/mmc_host/mmc0/mmc0:0001/serial      # الرقم التسلسلي
cat /sys/class/mmc_host/mmc0/mmc0:0001/type        # MMC أو SD

# معلومات الـ block device
cat /sys/class/mmc_host/mmc0/mmc0:0001/block/mmcblk0/size   # الحجم بـ 512-byte sectors
cat /sys/class/mmc_host/mmc0/mmc0:0001/preferred_erase_size # مضاعفات الـ erase group

# حالة الـ host controller
cat /sys/class/mmc_host/mmc0/ios              # الـ I/O settings الحالية (clock, vdd, timing)
```

---

#### 3. الـ ftrace — tracepoints وأحداث الـ MMC

```bash
# اعرض كل events المتعلقة بـ MMC
ls /sys/kernel/tracing/events/mmc/

# Events الأهم:
# mmc_request_start   — بداية كل طلب
# mmc_request_done    — نهاية الطلب مع النتيجة
# mmc_cmd_rw_start    — إرسال command
# mmc_blk_rw_start    — بداية block I/O
# mmc_blk_rw_end      — نهاية block I/O

# فعّل تتبع كل عمليات MMC
cd /sys/kernel/tracing
echo 1 > events/mmc/enable
echo 1 > tracing_on
cat trace_pipe &
# شغّل العملية المشبوهة هنا
echo 0 > tracing_on

# تتبع command محدد (مثلاً CMD6 = MMC_SWITCH)
echo 'opcode==6' > events/mmc/mmc_cmd_rw_start/filter
echo 1 > events/mmc/mmc_cmd_rw_start/enable
echo 1 > tracing_on

# تتبع الـ errors فقط
echo 'error!=0' > events/mmc/mmc_request_done/filter
echo 1 > events/mmc/mmc_request_done/enable

# مثال output للتفسير:
# mmc0: start struct mmc_request[<addr>]: cmd_opcode=18 cmd_arg=0x800 ...
# → CMD18 = MMC_READ_MULTIPLE_BLOCK من عنوان sector 0x800
```

---

#### 4. الـ printk والـ dynamic debug

```bash
# فعّل dynamic debug لكل ملفات MMC core
echo 'file drivers/mmc/core/*.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل debug لـ MMC block layer
echo 'file drivers/mmc/core/block.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل debug لعمليات الـ switch (CMD6)
echo 'file drivers/mmc/core/mmc_ops.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل debug للـ host controller (مثلاً sdhci)
echo 'file drivers/mmc/host/sdhci.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل كل شيء في MMC دفعة واحدة
echo 'module mmc_core +p' > /sys/kernel/debug/dynamic_debug/control

# أعلى مستوى verbosity للـ kernel log
dmesg -n 8
# أو في kernel cmdline: loglevel=8 mmc_core.use_spi_crc=1
```

---

#### 5. خيارات الـ Kconfig للـ debugging

| CONFIG | الوظيفة |
|--------|---------|
| `CONFIG_MMC_DEBUG` | يفعّل pr_debug() في كل MMC core |
| `CONFIG_MMC_UNSAFE_RESUME` | يساعد في تشخيص مشاكل resume |
| `CONFIG_MMC_CLKGATE` | debugging لـ clock gating |
| `CONFIG_FAIL_MMC_REQUEST` | fault injection لتوليد errors اصطناعية |
| `CONFIG_DEBUG_FS` | لازم لـ debugfs entries |
| `CONFIG_DYNAMIC_DEBUG` | لازم لـ dynamic debug |
| `CONFIG_TRACING` | لازم لـ ftrace |
| `CONFIG_MMC_BLOCK_MINORS` | تحكم في عدد الـ minor numbers |
| `CONFIG_UBSAN` | يكشف undefined behavior في الـ bit manipulation |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'CONFIG_MMC|CONFIG_FAIL_MMC'
```

---

#### 6. أدوات خاصة بالـ MMC subsystem

```bash
# mmc-utils — الأداة الرسمية لـ eMMC
# اقرأ EXT_CSD بشكل مقروء
mmc extcsd read /dev/mmcblk0

# اقرأ حالة الـ BKOPS
mmc bkops-enable /dev/mmcblk0

# اعرف معلومات الكارت
mmc info /dev/mmcblk0

# شغّل sanitize
mmc sanitize /dev/mmcblk0

# اقرأ General Purpose Partition info
mmc extcsd read /dev/mmcblk0 | grep -A2 'EXT_CSD'

# mmcli (إن كان موجوداً في distro)
mmcli -m 0

# فحص الـ RPMB (Replay Protected Memory Block)
mmc rpmb read-counter /dev/mmcblk0rpmb

# فحص الـ write protect
mmc writeprotect boot get /dev/mmcblk0
```

---

#### 7. جدول رسائل الـ error الشائعة

| رسالة في dmesg | السبب | الحل |
|----------------|-------|------|
| `mmc0: error -110 whilst initialising MMC card` | timeout أثناء init (R1_READY_FOR_DATA لم يرتفع) | افحص الـ clock وتوصيل الـ VCC |
| `mmc0: CMD6 failed` | فشل MMC_SWITCH — غالباً R1_SWITCH_ERROR | اقرأ EXT_CSD[185] بعد الأمر مباشرة |
| `mmc0: response CRC error` | R1_COM_CRC_ERROR — ضوضاء على الـ CMD line | افحص impedance matching وطول الـ trace |
| `mmc0: card status error` | R1_ERROR bit مرفوع | أرسل CMD13 واقرأ الـ status كامل |
| `mmc0: card is busy` | R1_READY_FOR_DATA=0 أو state≠TRAN | انتظر وأعد المحاولة — قد يكون programming طويل |
| `mmc0: Timeout waiting for hardware interrupt` | الـ host controller لم يرسل interrupt | افحص الـ IRQ mapping والـ clock gate |
| `mmc0: tuning execution failed` | فشل HS200/HS400 tuning | خفّض السرعة مؤقتاً، افحص الـ signal integrity |
| `mmc0: card does not support the aggressive power saving mode` | الكارت رفض POWER_OFF_NOTIFICATION | طبيعي — الـ kernel يتجاهله |
| `mmc0: BKOPS error -5` | فشل EXT_CSD_BKOPS_START | الكارت مشغول — أعد المحاولة لاحقاً |
| `mmc0: Unhandled bounce limit` | مشكلة DMA alignment | افحص الـ dma-mapping config |
| `mmc0: card claims to support voltages below defined range` | الـ OCR غير منطقي | افحص الـ voltage regulator |

---

#### 8. أماكن استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في mmc_ops.c — بعد CMD6 (MMC_SWITCH) */
int mmc_switch(struct mmc_card *card, u8 set, u8 index, u8 value,
               unsigned int timeout_ms)
{
    u32 status;
    int err = __mmc_switch(card, set, index, value, timeout_ms,
                           MMC_TIMING_LEGACY, true, true, true);

    /* نقطة استراتيجية: تحقق من R1_SWITCH_ERROR */
    WARN_ON(err == -EBADMSG);  /* يطبع stack إن فشل الـ switch */
    return err;
}

/* في core/mmc.c — عند تغيير الـ timing */
static int mmc_select_hs400(struct mmc_card *card)
{
    /* تحقق أن الكارت يدعم HS400 فعلاً */
    WARN_ON(!(card->mmc_avail_type & EXT_CSD_CARD_TYPE_HS400));
    /* ... */
}

/* في core/block.c — عند تلقي R1_ERROR */
static int mmc_blk_err_check(struct mmc_card *card,
                              struct mmc_async_req *areq)
{
    u32 status = brq->stop.resp[0];  /* R1 response */
    if (status & R1_ERROR) {
        dev_err(&card->dev, "R1_ERROR: status=0x%08x\n", status);
        dump_stack();  /* اطبع الـ call stack */
    }
    WARN_ON(R1_CURRENT_STATE(status) == R1_STATE_PRG &&
            (status & R1_READY_FOR_DATA));
}
```

---

### Hardware Level

#### 1. التحقق من مطابقة الـ hardware state للـ kernel state

الـ kernel يتتبع الحالة في `struct mmc_card` و `struct mmc_host`. للتحقق:

```bash
# الحالة التي يعتقدها الـ kernel
cat /sys/class/mmc_host/mmc0/ios
# مثال output:
# clock:          200000000 Hz     ← الـ clock الحالي
# vdd:            21 (3.3 ~ 3.4 V)
# bus mode:       2 (push-pull)
# chip select:    0 (don't care)
# power mode:     2 (on)
# bus width:      3 (8 bits)       ← 0=1bit, 1=4bit, 2=8bit, 3=ddr
# timing spec:    9 (mmc HS400)    ← مطابق لـ EXT_CSD_TIMING_HS400=3

# قارن مع EXT_CSD الحقيقي
mmc extcsd read /dev/mmcblk0 | grep -E 'HS_TIMING|BUS_WIDTH|CARD_TYPE'
```

#### مقارنة الـ R1 states:

```
kernel يعتقد: R1_STATE_TRAN (4)
الكارت الحقيقي: أرسل CMD13 واقرأ bits [12:9]

CMD13 → R1[12:9] = current state
    0 = Idle     → مشكلة في initialization
    1 = Ready    → لم يكتمل الـ OCR negotiation
    2 = Ident    → لم يكتمل الـ CID read
    3 = Stby     → لم يتم SELECT_CARD (CMD7)
    4 = Tran     ← الحالة الطبيعية للـ data transfer
    5 = Data     → الكارت لا يزال يرسل بيانات
    6 = Rcv      → الكارت لا يزال يستقبل
    7 = Prg      → الكارت يبرمج (program) الـ NAND
    8 = Dis      → deselected
```

---

#### 2. Register Dump عبر devmem2

```bash
# مثال لـ SDHCI controller على عنوان 0xFE340000 (Raspberry Pi)
# اقرأ SDHCI registers الأساسية

# Present State Register (offset 0x24)
devmem2 0xFE340024 w
# bits: [20]=CMD_INHIBIT_DAT, [19]=DAT_LINE_ACTIVE, [0]=CMD_INHIBIT

# Host Control 1 (offset 0x28)
devmem2 0xFE340028 b
# bit[1]=4BIT_BUS, bit[2]=HS_ENABLE, bit[5]=8BIT_BUS

# Clock Control (offset 0x2C)
devmem2 0xFE34002C w
# bits[15:8]=SDCLK_FREQ, bit[2]=SD_CLK_EN, bit[0]=INTERNAL_CLK_EN

# Error Interrupt Status (offset 0x32)
devmem2 0xFE340032 w
# bit[4]=CURRENT_LIMIT_ERR, bit[3]=DATA_END_BIT_ERR
# bit[2]=DATA_CRC_ERR, bit[1]=DATA_TIMEOUT_ERR
# bit[0]=CMD_TIMEOUT_ERR

# طريقة بديلة — io utility
io -4 -r 0xFE340024   # قراءة 32-bit register
```

---

#### 3. Logic Analyzer / Oscilloscope نصائح

**الـ eMMC bus له 6 إشارات جوهرية:**

```
CMD  — bidirectional command/response
CLK  — unidirectional clock من الـ host
DAT0-DAT7 — bidirectional data (8-bit mode)
```

**إعدادات الـ Logic Analyzer:**

```
Protocol:    eMMC/MMC
Clock edge:  Rising edge (data sampled)
Voltage:     1.8V (HS200/HS400) أو 3.3V (legacy)
Sample rate: ≥ 4× الـ clock الفعلي
             - HS400: 200MHz DDR → sample بـ 1.6GHz+
             - HS200: 200MHz SDR → sample بـ 800MHz+
             - HS52:  52MHz      → sample بـ 208MHz+
```

**ما تبحث عنه:**

```
1. CMD response timing:
   - R1: 48 bits بعد آخر bit في الـ command
   - R1b (busy): CMD line ترتفع بعد انتهاء الـ programming

2. CRC errors:
   - الـ CRC7 في نهاية كل command/response
   - الـ CRC16 في نهاية كل data block

3. HS400 strobe:
   - DS (Data Strobe) line — الكارت يرسلها أثناء قراءة البيانات
   - غيابها = مشكلة في EXT_CSD_BUS_WIDTH_STROBE

4. Tuning pattern (HS200):
   - CMD21: الـ host يرسله ويستقبل 128 bytes كـ tuning block
   - الـ pattern المتوقع معروف — أي انحراف = signal integrity problem
```

---

#### 4. مشاكل الـ hardware الشائعة وأنماطها في dmesg

| المشكلة | النمط في dmesg | التشخيص |
|---------|----------------|---------|
| ضعف الـ VCC | `mmc0: error -110 whilst initialising` عند cold boot | قس الـ VCC بـ oscilloscope أثناء power-up sequence |
| تداخل الـ clock | `mmc0: response CRC error` عشوائي | افحص الـ PCB trace length — يجب ≤ 50mm لـ HS200 |
| مشكلة الـ pull-up على CMD | `mmc0: card does not respond to voltage select` | قس مقاومة الـ pull-up — يجب 10kΩ–50kΩ |
| flash NAND متعب | `mmc_read_ext_csd: PRE_EOL_INFO=3` | استبدل الـ eMMC — `LIFE_A/B` = 10 → 90%+ استهلاك |
| حرارة مرتفعة | `mmc0: CMD timeout` متكرر تحت حمل | قس درجة حرارة الـ eMMC — TJ_MAX عادة 85°C |
| layout مشكلة | tuning فاشل دائماً | أقصر الـ PCB traces أو أضف series resistor (10-22Ω) |

---

#### 5. Device Tree Debugging

```bash
# تحقق من الـ DT المُحمَّل فعلاً
cat /proc/device-tree/mmc@fe340000/bus-width     # يجب = <8> لـ eMMC
cat /proc/device-tree/mmc@fe340000/mmc-hs200-1_8v # وجود الـ property = HS200 مفعّل
cat /proc/device-tree/mmc@fe340000/mmc-hs400-1_8v # وجود الـ property = HS400 مفعّل
cat /proc/device-tree/mmc@fe340000/max-frequency   # أقصى clock مسموح

# فك تشفير الـ clock value
xxd /proc/device-tree/mmc@fe340000/max-frequency
# 0x0BEBC200 = 200000000 = 200MHz

# تحقق من الـ vmmc/vqmmc regulators
ls /proc/device-tree/mmc@fe340000/ | grep -E 'vmmc|vqmmc|reset'

# قارن مع الـ kernel log
dmesg | grep mmc0 | head -30
# mmc0: new HS400 MMC card at address 0001
# → الـ kernel اختار HS400 = الـ DT يدعمه والكارت يدعمه

# إن أردت تعطيل HS400 مؤقتاً لـ debugging بدون إعادة compile
# أضف في kernel cmdline:
# sdhci.debug_quirks2=0x20   (SDHCI_QUIRK2_NO_1_8_V)
# أو
# mmc_core.use_spi_crc=1
```

**مثال DT صحيح لـ eMMC HS400:**

```dts
&mmc0 {
    bus-width = <8>;          /* 8-bit bus = EXT_CSD_BUS_WIDTH_8 */
    mmc-hs200-1_8v;           /* يفعّل EXT_CSD_CARD_TYPE_HS200_1_8V */
    mmc-hs400-1_8v;           /* يفعّل EXT_CSD_CARD_TYPE_HS400_1_8V */
    mmc-hs400-enhanced-strobe;/* يفعّل EXT_CSD_BUS_WIDTH_STROBE */
    vmmc-supply = <&vcc3v3>;  /* 3.3V للـ eMMC core */
    vqmmc-supply = <&vcc1v8>; /* 1.8V لـ I/O */
    max-frequency = <200000000>;
    non-removable;            /* eMMC مُلحم = لا يُنزع */
    no-sdio;
    no-sd;
};
```

---

### Practical Commands

#### سكريبت شامل لأول تشخيص

```bash
#!/bin/bash
# eMMC Quick Diagnosis Script

CARD=/dev/mmcblk0
MMC_SYS=/sys/class/mmc_host/mmc0/mmc0:0001
DBG=/sys/kernel/debug/mmc0/mmc0:0001

echo "=== eMMC Info ==="
mmc info $CARD 2>/dev/null || echo "mmc-utils not installed"

echo ""
echo "=== sysfs: Card Identity ==="
for f in name manfid date fwrev type; do
    echo "$f: $(cat $MMC_SYS/$f 2>/dev/null)"
done

echo ""
echo "=== sysfs: I/O State ==="
cat /sys/class/mmc_host/mmc0/ios 2>/dev/null

echo ""
echo "=== EXT_CSD Critical Bytes ==="
if [ -f "$DBG/ext_csd" ]; then
    python3 - <<'PYEOF'
import sys
data = open('/sys/kernel/debug/mmc0/mmc0:0001/ext_csd','rb').read()
fields = {
    192: ('EXT_CSD_REV',         'Spec version'),
    196: ('EXT_CSD_CARD_TYPE',   'Speed capability'),
    185: ('EXT_CSD_HS_TIMING',   'Current timing: 0=BC 1=HS 2=HS200 3=HS400'),
    183: ('EXT_CSD_BUS_WIDTH',   'Bus width: 0=1b 1=4b 2=8b 6=DDR8'),
    179: ('EXT_CSD_PART_CONFIG', 'Active partition'),
    246: ('EXT_CSD_BKOPS_STATUS','0=ok 1=non-urgent 2=urgent'),
    267: ('EXT_CSD_PRE_EOL_INFO','1=normal 2=warning 3=urgent'),
    268: ('LIFE_EST_TYPE_A',     'Type A wear ×10%'),
    269: ('LIFE_EST_TYPE_B',     'Type B wear ×10%'),
    307: ('EXT_CSD_CMDQ_DEPTH',  'CmdQ depth'),
    308: ('EXT_CSD_CMDQ_SUPPORT','0=no 1=yes'),
}
for idx, (name, desc) in sorted(fields.items()):
    val = data[idx]
    print(f'  [{idx:3d}] {name:<35s} = 0x{val:02x} ({val:3d})  # {desc}')
PYEOF
else
    echo "debugfs not mounted. Try: mount -t debugfs none /sys/kernel/debug"
fi

echo ""
echo "=== Recent MMC errors in dmesg ==="
dmesg | grep -iE 'mmc.*error|mmc.*fail|mmc.*timeout|mmc.*CRC' | tail -20

echo ""
echo "=== BKOPS & Health ==="
mmc extcsd read $CARD 2>/dev/null | grep -E 'BKOPS|EOL|LIFE|FIRMWARE' || \
    echo "Run: apt install mmc-utils"
```

---

#### تفعيل الـ ftrace لتتبع I/O بالتفصيل

```bash
#!/bin/bash
# MMC full I/O trace

TR=/sys/kernel/tracing
echo nop > $TR/current_tracer
echo 0 > $TR/tracing_on

# فعّل events المهمة
for ev in mmc_request_start mmc_request_done mmc_cmd_rw_start mmc_blk_rw_start; do
    echo 1 > $TR/events/mmc/$ev/enable 2>/dev/null && echo "enabled: $ev"
done

# زود حجم الـ buffer
echo 65536 > $TR/buffer_size_kb

echo 1 > $TR/tracing_on
echo "Tracing for 5 seconds..."
sleep 5
echo 0 > $TR/tracing_on

# اعرض النتائج
cat $TR/trace | grep -v "^#" | head -100
```

**مثال output وتفسيره:**

```
# مثال trace output:
# kworker/0:2-47  [000] .... 1234.567890: mmc_request_start: mmc0: start struct mmc_request[0x...]:
#   cmd_opcode=18(READ_MULTIPLE_BLOCK) cmd_arg=0x00100000 cmd_flags=0x45 cmd_retries=0 stop_opcode=12(STOP_TRANSMISSION)
#
# التفسير:
# CMD18 = MMC_READ_MULTIPLE_BLOCK
# arg=0x00100000 = sector 0x800 = sector 2048 = 1MB offset
# بعد 0.3ms:
# mmc_request_done: cmd_opcode=18 cmd_err=0 → نجح بدون error
```

---

#### قراءة R1 response مباشرة بـ CMD13

```bash
# أرسل CMD13 (SEND_STATUS) واقرأ R1 مباشرة
# مع mmc-utils
mmc status get /dev/mmcblk0

# أو عبر سكريبت يفسّر الـ R1
python3 - <<'EOF'
# افرض أن R1 = 0x00000900 (حالة طبيعية)
# استبدل بالقيمة الحقيقية من mmc status get

R1 = 0x00000900

bits = {
    31: "R1_OUT_OF_RANGE",
    30: "R1_ADDRESS_ERROR",
    29: "R1_BLOCK_LEN_ERROR",
    28: "R1_ERASE_SEQ_ERROR",
    27: "R1_ERASE_PARAM",
    26: "R1_WP_VIOLATION",
    25: "R1_CARD_IS_LOCKED",
    24: "R1_LOCK_UNLOCK_FAILED",
    23: "R1_COM_CRC_ERROR",
    22: "R1_ILLEGAL_COMMAND",
    21: "R1_CARD_ECC_FAILED",
    20: "R1_CC_ERROR",
    19: "R1_ERROR",
    18: "R1_UNDERRUN",
    17: "R1_OVERRUN",
    16: "R1_CID_CSD_OVERWRITE",
    15: "R1_WP_ERASE_SKIP",
    14: "R1_CARD_ECC_DISABLED",
    13: "R1_ERASE_RESET",
    8:  "R1_READY_FOR_DATA",
    7:  "R1_SWITCH_ERROR",
    6:  "R1_EXCEPTION_EVENT",
    5:  "R1_APP_CMD",
}

states = {0:"IDLE",1:"READY",2:"IDENT",3:"STBY",4:"TRAN",
          5:"DATA",6:"RCV",7:"PRG",8:"DIS"}

state = (R1 >> 9) & 0xF
print(f"R1 = 0x{R1:08x}")
print(f"Current State: {state} = {states.get(state, 'UNKNOWN')}")
print(f"Ready for data: {bool(R1 & (1<<8))}")
print("\nError bits set:")
for bit, name in bits.items():
    if bit not in (8,7,6,5) and (R1 & (1 << bit)):
        print(f"  !! {name} (bit {bit})")
EOF
```

---

#### تشخيص مشكلة tuning فاشل

```bash
# 1. افحص الـ timing الحالي
cat /sys/class/mmc_host/mmc0/ios | grep timing

# 2. فعّل verbose logging للـ tuning
echo 'func mmc_execute_tuning +p' > /sys/kernel/debug/dynamic_debug/control

# 3. أعد تهيئة الكارت
echo 1 > /sys/class/mmc_host/mmc0/mmc0:0001/remove
sleep 1
echo 1 > /sys/class/mmc_host/mmc0/rescan

# 4. راقب الـ kernel log
dmesg -w | grep -E 'tuning|timing|HS200|HS400'

# 5. إن فشل HS400 — جرّب إجبار HS200
# في /etc/modprobe.d/mmc.conf:
# options sdhci debug_quirks2=0x40  (SDHCI_QUIRK2_NO_HS400)
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: eMMC على RK3562 — بورد صناعي بيرفض يبوت

#### العنوان
**Industrial Gateway على RK3562 — الـ eMMC مش بيبوت بعد تحديث الـ firmware**

#### السياق
شركة بتبني industrial gateway بيشغّل Linux على RK3562. الـ eMMC الـ 32GB موصّل على SDMMC2. بعد تحديث الـ firmware عن طريق OTA، الجهاز وقف في البوت وبدأ يطلع kernel panic.

#### المشكلة
الـ kernel بيطلع:

```
mmc1: error -110 whilst initialising MMC card
mmc1: CMD1 timeout
```

**الـ CMD1** هو `MMC_SEND_OP_COND = 1`، وده أول command بعد `MMC_GO_IDLE_STATE = 0`. الـ timeout بيدل إن الكارت مش بيرد.

#### التحليل
في `mmc.h`:
```c
#define MMC_GO_IDLE_STATE   0   /* bc                          */
#define MMC_SEND_OP_COND    1   /* bcr  [31:0] OCR         R3  */
```

الـ `MMC_SEND_OP_COND` بيتبعت مع **OCR** في bits [31:0]. الـ host driver بيحدد voltage range في الـ OCR argument. لو الـ OTA كتب EXT_CSD[`EXT_CSD_POWER_CLASS` = 187] بقيمة غلط، الكارت ممكن يرفض يعمل initialization بالـ voltage الحالي.

الـ sequence الصح:
1. `CMD0` → Reset
2. `CMD1` → Host يبعت OCR مع voltage window
3. الكارت يرد بـ R3 وبيتحقق من `MMC_CARD_BUSY = 0x80000000`

```c
#define MMC_CARD_BUSY   0x80000000  /* Card Power up status bit */
```

لو الكارت مش بيرفع الـ bit ده في الـ R3 response خلال timeout، الـ driver بيعتبره ميت.

#### الحل
```bash
# تحقق من EXT_CSD بعد ما الكارت يرجع يشتغل من بيئة rescue
mmc extcsd read /dev/mmcblk0 | grep -A2 "POWER_CLASS"

# تأكد إن الـ OTA script مش بيغير EXT_CSD[187] بقيمة عالية
# الكود في scripts/ota_update.sh:
# mmc switch 0 3 187 0  <-- تأكد القيمة 0 مش قيمة تانية
```

في الـ OTA script، كان في bug بيكتب `EXT_CSD_POWER_CLASS` بقيمة `0x0F` بدل `0x00`:
```c
/* Wrong: high power class that card doesn't support at 1.8V */
/* ext_csd[EXT_CSD_POWER_CLASS] = 0x0F; */

/* Correct: let card use default power class */
ext_csd[EXT_CSD_POWER_CLASS] = 0x00;
```

#### الدرس المستفاد
**الـ `MMC_CARD_BUSY` bit** في الـ R3 response هو أول علامة على إن الكارت جاهز. أي OTA يلمس EXT_CSD لازم يتحقق من `EXT_CSD_PWR_CL_52_360` و`EXT_CSD_PWR_CL_52_195` قبل ما يكتب `POWER_CLASS`.

---

### السيناريو 2: Android TV Box على Allwinner H616 — تشغيل بطيء جداً

#### العنوان
**Android TV Box على H616 — الـ eMMC شغّال بـ 26MHz بدل 200MHz**

#### السياق
منتج Android TV box بيستخدم Allwinner H616 مع eMMC 5.1 سعة 64GB. المستخدمين بيشكوا إن الجهاز بطيء جداً في تحميل التطبيقات. الـ spec بيقول المفروض يشتغل بـ HS400.

#### المشكلة
```bash
$ cat /sys/kernel/debug/mmc0/ios
clock:          26000000 Hz
bus mode:       push-pull
chip select:    don't care
power mode:     on
bus width:      1 bits
timing spec:    legacy
```

الجهاز شغّال بـ **legacy mode** على 26MHz بـ bus width 1 bit — أبطأ mode ممكن.

#### التحليل
الـ kernel بيقرأ `EXT_CSD_CARD_TYPE` (byte 196) عشان يحدد أعلى speed mode مدعوم. في `mmc.h`:

```c
#define EXT_CSD_CARD_TYPE           196     /* RO */

#define EXT_CSD_CARD_TYPE_HS_26     (1<<0)  /* Card can run at 26MHz */
#define EXT_CSD_CARD_TYPE_HS_52     (1<<1)  /* Card can run at 52MHz */
#define EXT_CSD_CARD_TYPE_HS200_1_8V (1<<4) /* Card can run at 200MHz */
#define EXT_CSD_CARD_TYPE_HS400_1_8V (1<<6) /* Card can run at 200MHz DDR, 1.8V */
```

الـ host driver بيعمل negotiation:
1. يقرأ `EXT_CSD_CARD_TYPE`
2. يقارنه بـ host capabilities
3. يبعت `MMC_SWITCH (CMD6)` لتغيير `EXT_CSD_HS_TIMING` (byte 185)

```c
#define MMC_SWITCH          6   /* ac   [31:0] See below   R1b */
#define EXT_CSD_HS_TIMING   185 /* R/W */
#define EXT_CSD_TIMING_HS400 3  /* HS400 */
```

الـ Device Tree للـ H616 كان فيه مشكلة:

```dts
/* Wrong DT - missing vqmmc regulator for 1.8V signaling */
&mmc0 {
    vmmc-supply = <&reg_vcc_3v3>;
    /* vqmmc-supply missing! */
    bus-width = <8>;
    max-frequency = <200000000>;
};
```

من غير `vqmmc`, الـ host مش قادر يعمل voltage switch لـ 1.8V، فبيفضل على legacy.

#### الحل
```dts
/* Correct DT */
&mmc0 {
    vmmc-supply = <&reg_vcc_3v3>;
    vqmmc-supply = <&reg_vcc_1v8>;  /* required for HS200/HS400 */
    bus-width = <8>;
    max-frequency = <200000000>;
    mmc-hs400-1_8v;
    non-removable;
};
```

بعد الـ fix:
```bash
$ cat /sys/kernel/debug/mmc0/ios
clock:          200000000 Hz
bus width:      8 bits
timing spec:    hs400
```

#### الدرس المستفاد
**`EXT_CSD_CARD_TYPE_HS400_1_8V`** بيحتاج إن الـ host يدعم voltage switching. لو `vqmmc` مش موجود في الـ DT، الـ driver بيرفض يعمل HS200/HS400 حتى لو الكارت يدعمها. دايماً تأكد من الـ `EXT_CSD_STROBE_SUPPORT` (byte 184) لو بتستخدم HS400ES.

---

### السيناريو 3: IoT Sensor على STM32MP1 — data corruption عند الكتابة

#### العنوان
**IoT Sensor على STM32MP1 — بيانات بتتخرّب عند الكتابة على eMMC بعد power loss**

#### السياق
جهاز IoT بيسجّل sensor data على eMMC صغير 4GB موصّل على SDMMC1 في STM32MP157. الجهاز بيتعرض لانقطاعات مفاجئة في الكهرباء. بعد كل انقطاع، الـ filesystem بيطلع corrupted.

#### المشكلة
بعد power loss، `fsck` بيلاقي:
```
ext4: journal commit I/O error
block 0x1234: expected checksum 0xABCD, got 0x0000
```

الكارت مش بيكمّل الـ write قبل ما الطاقة تنقطع.

#### التحليل
الـ eMMC بيدعم **Reliable Write** عن طريق `EXT_CSD_WR_REL_PARAM` (byte 166):

```c
#define EXT_CSD_WR_REL_PARAM        166     /* RO */
#define EXT_CSD_WR_REL_PARAM_EN     (1<<2)  /* Reliable write supported */
```

ولو مفعّل، الكارت بيضمن إن كل write operation إما بتكتمل أو بترجع للحالة القديمة عند power loss. الـ kernel بيفعّله عن طريق:

```c
#define MMC_SWITCH          6
/* MMC_SWITCH argument format:
 * [25:24] Access Mode = 0x03 (WRITE_BYTE)
 * [23:16] EXT_CSD byte index
 * [15:08] Value
 * [02:00] Command Set
 */
```

الـ driver كان محتاج يبعت:
```
CMD6 argument = (MMC_SWITCH_MODE_WRITE_BYTE << 24) |
                (EXT_CSD_WR_REL_SET << 16) |  /* byte 167 */
                (0x1F << 8)                     /* enable for all partitions */
```

لكن الـ DT ما كانش بيفعّل الـ `reliable-write`:

```dts
/* Missing in original DT */
/* no mmc-ddr-3_3v or any reliable write hint */
```

#### الحل
```dts
&sdmmc1 {
    arm,primecell-periphid = <0x10153180>;
    vmmc-supply = <&v3v3>;
    bus-width = <4>;
    non-removable;
    st,use-cqe;  /* Command Queue for ordered writes */
};
```

وفي الـ application level، استخدام `O_SYNC | O_DSYNC` مع كل write:
```c
int fd = open("/data/sensor.log", O_WRONLY | O_APPEND | O_SYNC);
```

للتحقق إن الـ reliable write اتفعّل:
```bash
mmc extcsd read /dev/mmcblk0 | grep -A1 "Write Reliability"
# Expected: WR_REL_PARAM[2] = 0x1
```

وتحقق من الـ R1 status بعد كل write:
```c
/* Check R1 response bits after CMD25 */
if (status & R1_WP_VIOLATION)
    pr_err("Write protection violation!\n");
if (status & R1_CC_ERROR)
    pr_err("Card controller error!\n");
/* R1_CC_ERROR = (1 << 20) */
```

#### الدرس المستفاد
**`EXT_CSD_WR_REL_PARAM_EN`** هو أساس data integrity على eMMC في بيئات power-unstable. دايماً تحقق منه في أي IoT أو industrial product، وتفعّل Reliable Write على boot partition وuser data partition.

---

### السيناريو 4: Automotive ECU على i.MX8 — RPMB authentication فشل

#### العنوان
**Automotive ECU على i.MX8QM — فشل في قراءة security keys من RPMB**

#### السياق
ECU لنظام ADAS بيستخدم i.MX8QM. الـ eMMC RPMB partition بيخزّن cryptographic keys لـ Secure Boot. بعد تبديل الـ eMMC chip بنفس الـ model من supplier تاني، النظام بدأ يرفض يبوت بسبب RPMB authentication failure.

#### المشكلة
```
tee-supplicant: RPMB: ioctl RPMB_F_READ failed: -EIO
optee: secure storage: authentication failed
```

#### التحليل
الـ RPMB (Replay Protected Memory Block) بيتحكم فيه `EXT_CSD_RPMB_MULT` (byte 168):

```c
#define EXT_CSD_RPMB_MULT   168     /* RO */
```

الـ RPMB size = `EXT_CSD_RPMB_MULT × 128KB`. الكارت الجديد كان نفس الـ model لكن revision مختلف، فـ RPMB size اتغير من 512KB إلى 4MB.

الوصول للـ RPMB بيتم عن طريق:
1. `MMC_SWITCH (CMD6)` لـ switch لـ RPMB partition
2. `EXT_CSD_PART_CONFIG` (byte 179) لاختيار الـ partition

```c
#define EXT_CSD_PART_CONFIG             179     /* R/W */
#define EXT_CSD_PART_CONFIG_ACC_RPMB    (0x3)   /* Access RPMB partition */
#define EXT_CSD_PART_CONFIG_ACC_MASK    (0x7)
```

الـ switch argument:
```c
arg = (MMC_SWITCH_MODE_WRITE_BYTE << 24) |
      (EXT_CSD_PART_CONFIG << 16) |
      (EXT_CSD_PART_CONFIG_ACC_RPMB << 8) |
      EXT_CSD_CMD_SET_NORMAL;
```

المشكلة إن الـ RPMB key كان مكتوب بـ write counter معيّن في الكارت القديم. الكارت الجديد عنده write counter = 0 (fresh chip) فالـ authentication بيفشل.

#### الحل
الحل في استخدام OP-TEE لإعادة provisioning الـ RPMB key:

```bash
# على النظام الجديد، provisioning الـ RPMB key
tee-supplicant &
# ثم من secure world:
optee_rpmb_write_key

# للتحقق من RPMB size الجديد:
mmc extcsd read /dev/mmcblk0 | grep "RPMB"
# RPMB Size: 4096 kB  (EXT_CSD_RPMB_MULT = 32)
```

وتحقق من status response بعد كل RPMB operation:
```c
/* After CMD13 (MMC_SEND_STATUS): */
u32 status;
/* R1_CARD_IS_LOCKED = (1 << 25) */
if (status & R1_CARD_IS_LOCKED)
    pr_err("Card is locked - RPMB provisioning needed\n");
```

#### الدرس المستفاد
**`EXT_CSD_RPMB_MULT`** لازم يتحقق منه في كل bring-up لكارت جديد. الـ RPMB key مرتبط بالـ chip المادي، ومش portable بين chips حتى لو نفس الـ model. دايماً في الـ automotive، خلّي RPMB provisioning جزء من الـ End-of-Line testing.

---

### السيناريو 5: Custom Board Bring-up على AM62x — eMMC بيدخل Sleep ومش بيصحى

#### العنوان
**Custom Board Bring-up على TI AM62x — eMMC بيدخل في Sleep mode ومش بيرجع**

#### السياق
فريق bring-up بيشتغل على custom board بيستخدم TI AM62x (Sitara) لـ industrial HMI panel. الـ eMMC 16GB موصّل على MMC0. الـ board بيشتغل تمام أول ما بيبوت، لكن بعد 30 ثانية من idle، كل الـ I/O على الـ eMMC بتفشل.

#### المشكلة
```
mmc0: CMD13 failed: -ETIMEDOUT
mmc0: error -110 whilst initialising MMC card
block layer: I/O error on mmcblk0
```

الـ error بييجي بالظبط بعد 30 ثانية — زي ما في timer.

#### التحليل
الـ eMMC يدعم **Sleep mode** عن طريق `CMD5` (`MMC_SLEEP_AWAKE`):

```c
#define MMC_SLEEP_AWAKE   5   /* ac   [31:16] RCA 15:flg R1b */
```

الـ bit 15 في الـ argument بيحدد:
- `1` → دخول Sleep
- `0` → Awake (خروج من Sleep)

الـ Sleep mode بيُفعَّل من الـ host driver لتوفير الطاقة. بعد ما يدخل Sleep، الكارت بيحتاج Awake command قبل أي operation تانية — لو الـ driver نسي يبعت Awake، كل الـ commands بعدها بتفشل.

الـ `EXT_CSD_S_A_TIMEOUT` (byte 217) بيحدد وقت الـ Sleep/Awake timeout:

```c
#define EXT_CSD_S_A_TIMEOUT   217   /* RO */
```

الـ actual timeout = `100ns × 2^EXT_CSD_S_A_TIMEOUT`. لو القيمة كبيرة، الـ Awake بياخد وقت طويل والـ driver بيعمل timeout.

المشكلة في الـ DT:

```dts
/* Wrong: aggressive power saving enabled */
&sdhci0 {
    keep-power-in-suspend;
    /* no cap-mmc-hw-reset */
};
```

الـ `mmc-pm-flags` كانت بتفعّل runtime PM بـ 30 ثانية timeout، والـ driver كان بيبعت sleep command لكن مش بيبعت Awake صح بسبب bug في الـ AM62x SDHCI driver.

#### التشخيص
```bash
# تحقق من R1 state machine
# R1_CURRENT_STATE(x) = ((x & 0x00001E00) >> 9)
# R1_STATE_DIS = 8  <-- disconnected (sleep)
# لو الكارت في state 8 ومش بيرجع → bug في Awake sequence

# اقرأ الـ EXT_CSD live
mmc extcsd read /dev/mmcblk0 | grep -E "S_A_TIMEOUT|SLEEP"

# فعّل MMC debug tracing
echo 1 > /sys/kernel/debug/mmc0/err_stats
echo 65535 > /sys/bus/platform/drivers/sdhci/fe18000.mmc/mmc_host/mmc0/
```

تتبع الـ CMD5 في kernel log:
```bash
echo 'mmc:*' > /sys/kernel/debug/tracing/set_event
cat /sys/kernel/debug/tracing/trace_pipe | grep -E "CMD5|SLEEP|AWAKE"
```

#### الحل
```dts
/* Fixed DT - disable aggressive sleep */
&sdhci0 {
    pinctrl-names = "default";
    pinctrl-0 = <&main_mmc0_pins_default>;
    ti,driver-strength-ohm = <50>;
    disable-wp;
    non-removable;
    cap-mmc-hw-reset;
    /* Remove keep-power-in-suspend to avoid sleep issues */
    /* Or increase sleep timeout to safe value */
};
```

أو في الـ kernel config:
```bash
# Disable MMC runtime PM during bring-up
echo 0 > /sys/bus/mmc/devices/mmc0:0001/power/autosuspend_delay_ms
echo on > /sys/bus/mmc/devices/mmc0:0001/power/control
```

للتحقق من الـ S_A_TIMEOUT value:
```bash
mmc extcsd read /dev/mmcblk0 | grep "Sleep/Awake"
# S_A_TIMEOUT: 0x11 → 100ns × 2^17 = ~13ms
# لو الـ SDHCI timeout أقل من كده → race condition
```

#### الدرس المستفاد
**`MMC_SLEEP_AWAKE (CMD5)`** هو double-edged sword — بيوفر طاقة لكنه بيتطلب Awake sequence صح قبل أي command. في الـ bring-up، افصل الـ runtime PM تماماً أول ما تتأكد إن الـ basic I/O شغّال، وراجع `EXT_CSD_S_A_TIMEOUT` عشان تضبط الـ host timeout أكبر منه. كمان `mmc_ready_for_data()` في `mmc.h` بتتحقق من `R1_CURRENT_STATE == R1_STATE_TRAN` وده مهم جداً — الكارت النايم بيكون في `R1_STATE_DIS` مش `R1_STATE_TRAN`.

```c
/* This function in mmc.h catches exactly this scenario */
static inline bool mmc_ready_for_data(u32 status)
{
    return status & R1_READY_FOR_DATA &&
           R1_CURRENT_STATE(status) == R1_STATE_TRAN;
    /* Card in sleep (DIS state) will return false here */
}
```
## Phase 7: مصادر ومراجع

---

### مقالات LWN.net

دي أهم المقالات اللي بتغطي الـ MMC subsystem في الـ Linux kernel:

| المقال | الوصف |
|--------|-------|
| [1/4 MMC layer](https://lwn.net/Articles/82765/) | أول patch series لـ MMC core support في الـ kernel |
| [new driver: MMC framework, patch against 2.4.19](https://lwn.net/Articles/7176/) | الـ MMC framework الأصلي — تاريخي ومهم لفهم الأساس |
| [MMC updates](https://lwn.net/Articles/253888/) | تحديثات الـ MMC layer زي SDIO و SPI support |
| [mmc: Add Command Queue support](https://lwn.net/Articles/738160/) | إضافة الـ HW Command Queue (CMDQ) للـ eMMC — بيديك 25-50% تحسن في الـ random I/O |
| [Add MMC software queue support](https://lwn.net/Articles/803435/) | الـ SW CMDQ بـ queue depth 32 — بديل للـ HW queue |
| [pwrseq: Add subsystem for power sequences](https://lwn.net/Articles/602855/) | الـ power sequence subsystem اللي بيخدم الـ MMC controllers |
| [Replay Protected Memory Block (RPMB) subsystem](https://lwn.net/Articles/975405/) | الـ RPMB كـ subsystem مستقل — مهم جداً للـ secure storage |
| [mmc: Add host driver for OCTEON MMC controller](https://lwn.net/Articles/496911/) | مثال real-world لـ MMC host controller driver |

---

### توثيق الـ Kernel الرسمي

الملفات دي في `Documentation/` هي المرجع الأول:

```
Documentation/driver-api/mmc/index.rst       # نقطة البداية
Documentation/driver-api/mmc/mmc-tools.rst   # mmc-utils tool
Documentation/driver-api/mmc/mmc-dev-parts.rst  # partitioning: boot, RPMB, GP
Documentation/driver-api/mmc/mmc-async-req.rst  # async request API
Documentation/driver-api/mmc/mmc-dev-attrs.rst  # sysfs attributes
```

**الـ online links:**

- [MMC/SD/SDIO card support — kernel.org](https://docs.kernel.org/driver-api/mmc/index.html)
- [SD and MMC Device Partitions](https://www.kernel.org/doc/html/latest/driver-api/mmc/mmc-dev-parts.html)
- [MMC tools introduction](https://docs.kernel.org/driver-api/mmc/mmc-tools.html)

---

### الـ Source Files الأساسية في الـ Kernel

الملفات دي هي جوهر الـ MMC subsystem:

```
include/linux/mmc/mmc.h      # commands, EXT_CSD fields, R1 status — الملف اللي بندرسه
include/linux/mmc/host.h     # struct mmc_host, OCR bits, capabilities
include/linux/mmc/card.h     # struct mmc_card, partitions, quirks
include/linux/mmc/sdio.h     # SDIO-specific commands
drivers/mmc/core/mmc.c       # MMC core logic: init, EXT_CSD parsing, speed selection
drivers/mmc/core/core.c      # request handling, power management
drivers/mmc/core/mmc_ops.c   # low-level operations: switch, send_status
drivers/mmc/core/queue.c     # block queue and CMDQ integration
```

---

### Commits مهمة في تاريخ الـ MMC Subsystem

| الـ Feature | المكان للبحث |
|-------------|--------------|
| HS200 support | `git log --all --grep="HS200" -- drivers/mmc/` |
| HS400 support | `git log --all --grep="HS400" -- drivers/mmc/` |
| HS400ES (Enhanced Strobe) | `git log --all --grep="HS400ES"` |
| CMDQ hardware support | `git log --all --grep="cmdq" -- drivers/mmc/` |
| RPMB partition support | `git log --all --grep="RPMB" -- drivers/mmc/` |
| BKOPS (Background Operations) | `git log --all --grep="BKOPS" -- drivers/mmc/` |

**Patchwork للـ linux-mmc mailing list:**
- [tmio: eMMC HS400 mode support patch](https://patchwork.kernel.org/patch/10175319/)
- [linux-commits-search (Typesense)](https://linux-commits-search.typesense.org/) — ابحث بـ "EXT_CSD" أو "mmc_select_hs400"

---

### Mailing List

**الـ linux-mmc mailing list** هو المكان الرسمي لمناقشات الـ MMC subsystem:

- **Email:** `linux-mmc@vger.kernel.org`
- **Archive:** [lore.kernel.org/linux-mmc](https://lore.kernel.org/linux-mmc/)
- **Maintainer حالي:** Ulf Hansson — اتبع patches بتاعته عشان تفهم اتجاه الـ subsystem

مناقشات مهمة اتعملت على الـ list:
- HS400 init sequence و علاقته بـ HS200
- CMDQ hardware vs software queue tradeoffs
- Power management و BKOPS scheduling
- RPMB كـ character device مستقل

---

### مصادر eLinux.org

- [Tests: eMMC 8-bit width](https://elinux.org/Tests:eMMC-8bit-width) — اختبار الـ 8-bit bus width للـ eMMC
- [Tests: SDIO KS7010](https://elinux.org/Tests:SDIO-KS7010) — مثال على SDIO driver testing
- [MMC initialization error troubleshooting](https://elinux.org/Thread:Talk:R-Car/Boards/M3SK/Booting_of_kernel_stops_at_23.047503_mmc0:_error_-110_whilst_initialising_MMC_card_(5)) — error -110 أثناء الـ MMC init — مفيد للـ debugging

---

### مصادر Kernel Newbies

**الـ kernel changelogs** بتوثق كل تغيير مهم في الـ MMC subsystem:

| الإصدار | ما اتغير في الـ MMC |
|---------|---------------------|
| [Linux 2.6.24](https://kernelnewbies.org/Linux_2_6_24) | SDIO support + MMC over SPI — تحول كبير في الـ subsystem |
| [Linux 3.9](https://kernelnewbies.org/Linux_3.9) | تحسينات الـ eMMC partitioning |
| [Linux 4.10](https://kernelnewbies.org/Linux_4.10) | HS400 enhanced strobe |
| [Linux 6.1](https://kernelnewbies.org/Linux_6.1) | MMC updates مستمرة |
| [Linux 6.5](https://kernelnewbies.org/Linux_6.5) | تحسينات الـ CMDQ والـ power management |
| [Linux 6.8](https://kernelnewbies.org/Linux_6.8) | MMC driver improvements |
| [Linux 6.12](https://kernelnewbies.org/Linux_6.12) | أحدث تحديثات الـ MMC subsystem |

---

### الكتب المرجعية

#### Linux Device Drivers, 3rd Edition (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصل المهم:** Chapter 17 — Network Drivers (للـ pattern)، وفهم block device layer
- **ملاحظة:** الـ mmc.h نفسه بيشكر Corbet و Rubini في الـ copyright header — دليل على تأثيرهم
- **متاح مجاناً:** [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development, 3rd Edition — Robert Love
- **الفصل:** Chapter 14 — The Block I/O Layer
- **الأهمية:** يشرح الـ block layer اللي الـ MMC بيتعامل معاه
- الـ `mmc_request`, الـ `mmc_data`, وربطهم بالـ `bio` structure

#### Embedded Linux Primer — Christopher Hallinan
- **الفصل:** Chapter 9 — File Systems
- **الأهمية:** بيغطي الـ flash storage وإزاي الـ MMC/eMMC بيتعامل مع الـ embedded systems
- الـ eMMC partitioning strategy وتهيئة الـ boot partition

#### Professional Embedded ARM Development
- بيتناول الـ MMC controller configuration في الـ ARM SoCs
- مفيد لفهم الـ host controller side بدل ما تبص بس على الـ card side

---

### أدوات عملية

#### mmc-utils
أداة userspace الرسمية للتعامل مع الـ eMMC:

```bash
# تثبيت
apt install mmc-utils        # Debian/Ubuntu
dnf install mmc-utils        # Fedora

# أمثلة عملية
mmc extcsd read /dev/mmcblk0              # قراءة كل الـ EXT_CSD
mmc status get /dev/mmcblk0               # R1 status
mmc bkops-en auto /dev/mmcblk0           # تفعيل Auto BKOPS
mmc sanitize /dev/mmcblk0                # تشغيل Sanitize
mmc bootpart enable 1 1 /dev/mmcblk0    # تفعيل boot partition

# Source code
git clone git://git.kernel.org/pub/scm/linux/kernel/git/cjb/mmc-utils.git
```

#### sysfs interface

```bash
# معلومات الـ card
cat /sys/class/mmc_host/mmc0/mmc0:0001/name
cat /sys/class/mmc_host/mmc0/mmc0:0001/type
cat /sys/class/mmc_host/mmc0/mmc0:0001/date

# EXT_CSD كـ raw hex
cat /sys/kernel/debug/mmc0/mmc0:0001/ext_csd

# الـ speed mode الحالي
cat /sys/class/mmc_host/mmc0/mmc0:0001/speed_class
```

---

### مصطلحات للبحث

لو عايز تعمق أكتر، استخدم الـ search terms دي:

```
# في kernel source
grep -r "EXT_CSD_CARD_TYPE" drivers/mmc/
grep -r "mmc_select_hs400" drivers/mmc/core/
grep -r "MMC_CMDQ" include/linux/mmc/

# في LWN.net
"MMC command queuing" site:lwn.net
"eMMC reliable write" site:lwn.net
"MMC power management BKOPS" site:lwn.net

# في mailing list archive
"EXT_CSD_CMDQ_SUPPORT" site:lore.kernel.org
"HS400ES enhanced strobe" site:lore.kernel.org
"MMC_SEND_TUNING_BLOCK" site:lore.kernel.org

# في patchwork
"mmc: core:" site:patchwork.kernel.org
```

---

### الـ JEDEC Specification

الـ source الأساسي لكل constants في `mmc.h` هو الـ JEDEC standard:

- **JESD84-B51** — eMMC 5.1 Specification (الأحدث المدعوم في الـ kernel)
- **JESD84-A441** — eMMC 4.41 (الأساس التاريخي)
- متاح من: [jedec.org](https://www.jedec.org/standards-documents/docs/jesd84-b51)
- **كل** `EXT_CSD_*` macro في `mmc.h` له رقم byte مقابل في الـ spec

---

### خلاصة تدرجية للتعلم

```
1. ابدأ بـ mmc.h نفسه — اقرأ الـ commands والـ EXT_CSD fields
   ↓
2. اقرأ JEDEC JESD84-B51 للـ spec الأصلي
   ↓
3. اتبع drivers/mmc/core/mmc.c لترى كيف يُستخدم كل macro
   ↓
4. استخدم mmc-utils عملياً على hardware حقيقي
   ↓
5. اقرأ LWN articles عن CMDQ وRPMB
   ↓
6. تابع linux-mmc@vger.kernel.org للتطورات الجديدة
```
## Phase 8: Writing simple module

### الفكرة

**الـ** `mmc_send_status` هي function مُصدَّرة بـ `EXPORT_SYMBOL_GPL` في `drivers/mmc/core/mmc_ops.c`. بتبعت **الـ** `MMC_SEND_STATUS` (CMD13) للكارت وبترجع الـ R1 status register — ده اللي بيحصل كل ما الـ kernel يحتاج يعرف حالة الكارت (أثناء write / erase / busy-poll). Hook على الـ function ده مفيد جداً لمراقبة نشاط الـ MMC/eMMC بدون أي تعديل في الـ MMC subsystem.

سنستخدم **kprobe** على `mmc_send_status` عشان نطبع اسم الـ host والـ RCA (Relative Card Address) لكل استعلام حالة.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * mmc_status_probe.c
 *
 * kprobe on mmc_send_status() — fires every time the kernel polls
 * an MMC/eMMC card's R1 status register (CMD13).
 */

#include <linux/kernel.h>      /* pr_info, pr_err                    */
#include <linux/module.h>      /* MODULE_* macros, module_init/exit  */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe()   */
#include <linux/mmc/card.h>    /* struct mmc_card, card->host, ->rca */
#include <linux/mmc/host.h>    /* struct mmc_host, mmc_hostname()    */
#include <linux/mmc/mmc.h>     /* MMC_SEND_STATUS, R1_CURRENT_STATE  */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("MMC Probe Demo");
MODULE_DESCRIPTION("kprobe on mmc_send_status to trace CMD13 card-status polls");

/*
 * pre_handler — runs just BEFORE mmc_send_status() executes.
 *
 * Prototype of the hooked function:
 *   int mmc_send_status(struct mmc_card *card, u32 *status)
 *
 * On x86-64:
 *   regs->di == first arg  (struct mmc_card *card)
 *   regs->si == second arg (u32 *status)          -- not used here
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Cast the first argument register to the expected pointer type */
    struct mmc_card *card = (struct mmc_card *)regs->di;

    if (!card || !card->host)
        return 0;   /* safety check — never deref a NULL in a probe */

    /*
     * mmc_hostname() returns the host's device name string (e.g. "mmc0").
     * card->rca is the 16-bit Relative Card Address assigned during
     * MMC_SET_RELATIVE_ADDR (CMD3).
     */
    pr_info("mmc_status_probe: [%s] CMD13 (SEND_STATUS) for RCA=0x%04x\n",
            mmc_hostname(card->host),
            card->rca);

    return 0; /* non-zero return would skip the original function — we don't want that */
}

/*
 * post_handler — runs just AFTER mmc_send_status() returns.
 *
 * Here we have access to the return value in regs->ax.
 * We only log errors (non-zero return means I/O error on CMD13).
 */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    long ret = regs_return_value(regs); /* portable way to read return val */

    if (ret)
        pr_warn("mmc_status_probe: mmc_send_status returned error %ld\n", ret);
}

/* kprobe descriptor — ties the symbol name to our handlers */
static struct kprobe kp = {
    .symbol_name = "mmc_send_status",
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

static int __init mmc_probe_init(void)
{
    int ret;

    /*
     * register_kprobe() looks up "mmc_send_status" in the kernel symbol
     * table, patches a breakpoint (int3 on x86) at the function entry,
     * and registers our handlers. Returns 0 on success.
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("mmc_status_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("mmc_status_probe: kprobe planted at %s (addr=%px)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

static void __exit mmc_probe_exit(void)
{
    /*
     * unregister_kprobe() MUST be called in exit — it removes the
     * breakpoint and ensures no CPU is still inside our handler
     * before the module pages are freed. Skipping this = kernel panic.
     */
    unregister_kprobe(&kp);
    pr_info("mmc_status_probe: kprobe removed from %s\n", kp.symbol_name);
}

module_init(mmc_probe_init);
module_exit(mmc_probe_exit);
```

---

### Makefile لبناء الـ Module

```makefile
obj-m += mmc_status_probe.o

KDIR  ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/kprobes.h` | يعرّف `struct kprobe`، `register_kprobe()`، `unregister_kprobe()` |
| `linux/mmc/card.h` | يعرّف `struct mmc_card` اللي فيه `->host` و`->rca` |
| `linux/mmc/host.h` | يعرّف `struct mmc_host` والـ macro الـ `mmc_hostname()` |
| `linux/mmc/mmc.h` | تعريفات الـ commands زي `MMC_SEND_STATUS` وبتات الـ R1 status |

الـ `mmc.h` هو ملفنا المحوري — من غيره مش هنعرف نفهم أرقام الـ opcodes أو بتات الـ response.

#### الـ `handler_pre`

الـ pre_handler بيتشغل قبل تنفيذ `mmc_send_status` مباشرةً. بنقرأ الـ `regs->di` (على x86-64 هو أول argument حسب ABI) وبنعامله كـ `struct mmc_card *` عشان نطلع منه اسم الـ host وقيمة الـ RCA — ده بيدّينا context واضح لكل استعلام CMD13.

#### الـ `handler_post`

الـ post_handler بيتشغل بعد ما الـ function ترجع. بنستخدم `regs_return_value()` بدل `regs->ax` مباشرةً عشان الكود يشتغل على أكتر من architecture. بنطبع error بس لو الـ return value مش صفر — ده مفيد للتشخيص بدون إغراق الـ dmesg.

#### الـ `struct kprobe`

**الـ** `.symbol_name` بيقول للـ kernel فين يحط الـ breakpoint بالاسم بدل العنوان — أسهل وأأمن. لو الـ symbol مش موجود أو مش قابل للـ probe، الـ `register_kprobe` بترجع error في init وبيمنع تحميل الـ module.

#### الـ `module_init` / `module_exit`

`register_kprobe` في الـ init بيزرع الـ probe في الـ kernel. `unregister_kprobe` في الـ exit **إجباري** — لو المودول اتأزّل من غيره، الـ CPU هيكمل تنفيذ بعد الـ breakpoint في كود مش موجود، وده kernel panic فوري.

---

### تجربة الـ Module

```bash
# بناء المودول
make

# تحميله
sudo insmod mmc_status_probe.ko

# مشاهدة الـ log (الـ CMD13 بيتعمل كتير أثناء أي I/O على eMMC)
sudo dmesg -w | grep mmc_status_probe

# إزالته
sudo rmmod mmc_status_probe
```

مثال على الـ output المتوقع في `dmesg`:

```
mmc_status_probe: kprobe planted at mmc_send_status (addr=ffffffffc0a1b340)
mmc_status_probe: [mmc0] CMD13 (SEND_STATUS) for RCA=0x0001
mmc_status_probe: [mmc0] CMD13 (SEND_STATUS) for RCA=0x0001
mmc_status_probe: [mmc0] CMD13 (SEND_STATUS) for RCA=0x0001
```

كل سطر بيقابل **busy-poll** واحدة — يعني الـ kernel كان بيستنّى الكارت يخلّص write أو erase ويرجع للـ `TRAN` state.
