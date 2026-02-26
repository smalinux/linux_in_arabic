## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي بيتبعه الملف

الملف `include/linux/mmc/sd.h` جزء من **MMC/SD/SDIO Subsystem** — اللي بيتمنتنه Ulf Hansson على قائمة البريد `linux-mmc@vger.kernel.org`. الـ subsystem ده بيغطي كل حاجة من hardware drivers لحد الـ core logic اللي بتتعامل مع كروت الذاكرة.

---

### الصورة الكبيرة — قصة بسيطة

تخيل إنك عايز تتكلم مع حد بلغة معينة — لازم تعرف الكلمات والجمل الصح. كارت الـ **SD** (Secure Digital) اللي بتحطه في كاميرتك أو موبايلك هو جهاز مستقل ليه "لغته الخاصة": مجموعة من الـ **commands** والـ **registers** اللي لازم الـ host (يعني المعالج في الموبايل أو اللاب توب) يبعتهالها عشان تتفاهموا.

الـ **SD Specification** — اللي بتصدره مجموعة SD Association — هو القاموس الرسمي لهذه اللغة. فيه:
- أرقام الـ **commands** (CMD3, CMD6, CMD8, …)
- شكل الـ **arguments** اللي بتروح مع كل command
- نوع الـ **response** اللي بيرجع (R1, R2, R3, …)
- تعريف الـ **registers** زي OCR و SCR

الملف `sd.h` هو الترجمة الرسمية لهذا القاموس داخل الـ Linux kernel — مجرد `#define` constants بس كلها بتترجم بدقة الـ SD Specification.

---

### ليه الملف ده مهم؟

من غيره، أي كود في الـ kernel محتاج يكتب الأرقام hardcoded زي `mmc_send_cmd(host, 6, ...)` من غير ما يعرف إيه معنى الـ 6 ده. بوجود `sd.h`:

```c
/* بدل كتابة الرقم مباشرة */
mmc_send_cmd(host, 6, arg);

/* بنكتب اسم واضح من الـ spec */
mmc_send_cmd(host, SD_SWITCH, arg);
```

ده بيخلي الكود:
1. **Readable** — أي developer يفتح الـ spec ويلاقي نفس الاسم.
2. **Maintainable** — لو الـ spec اتغير، بنعدل في مكان واحد بس.
3. **Error-free** — مش هتكتب رقم غلط بالغلط.

---

### إيه اللي في الملف بالظبط؟

#### 1. SD Commands (CMD)

```c
#define SD_SEND_RELATIVE_ADDR     3   /* bcr                     R6  */
#define SD_SEND_IF_COND           8   /* bcr  [11:0] See below   R7  */
#define SD_SWITCH_VOLTAGE         11  /* ac                      R1  */
#define SD_SWITCH                 6   /* adtc [31:0] See below   R1  */
#define SD_ERASE_WR_BLK_START    32   /* ac   [31:0] data addr   R1  */
#define SD_ERASE_WR_BLK_END      33   /* ac   [31:0] data addr   R1  */
```

التعليق بيوضح:
- **نوع الـ command**: `bc` (broadcast)، `ac` (addressed)، `adtc` (data transfer)، `bcr` (broadcast with response).
- **شكل الـ argument**: مثلاً `[31:0]` يعني الـ 32-bit الكاملة، `[11:0]` يعني الـ 12 bit الأقل.
- **نوع الـ response**: R1 إلى R7 حسب الـ spec.

#### 2. Application Commands (ACMD)

```c
#define SD_APP_SET_BUS_WIDTH      6   /* ac   [1:0] bus width    R1  */
#define SD_APP_OP_COND           41   /* bcr  [31:0] OCR         R3  */
#define SD_APP_SEND_SCR          51   /* adtc                    R1  */
```

الـ **ACMD** هي commands خاصة بـ SD بس — لازم قبلها تبعت `CMD55` عشان الكارت يعرف إن اللي جاي بعدها ACMD مش CMD عادي.

#### 3. OCR Register Bits

**الـ OCR (Operating Conditions Register)** هو register بيقول لـ host الكارت بيشتغل على أنهي voltage وهل SDHC/SDXC/SDUC:

```c
#define SD_OCR_S18R    (1 << 24)  /* 1.8V switching request */
#define SD_OCR_XPC     (1 << 28)  /* SDXC power control */
#define SD_OCR_CCS     (1 << 30)  /* Card Capacity Status - SDHC/SDXC */
```

#### 4. SCR Version و Bus Width

**الـ SCR (SD Configuration Register)** بيكشف قدرات الكارت:

```c
#define SCR_SPEC_VER_0    0  /* SD spec 1.0 - 1.01 */
#define SCR_SPEC_VER_1    1  /* SD spec 1.10 */
#define SCR_SPEC_VER_2    2  /* SD spec 2.00 - 3.0X */

#define SD_BUS_WIDTH_1    0  /* 1-bit bus */
#define SD_BUS_WIDTH_4    2  /* 4-bit bus */
```

#### 5. SD_SWITCH Modes

**الـ CMD6 (SD_SWITCH)** بيعمل حاجتين: إما يـ **check** إن الكارت بيدعم mode معين، أو يـ **set** الـ mode فعلاً:

```c
#define SD_SWITCH_CHECK   0  /* بس بنتحقق — مش بنغير */
#define SD_SWITCH_SET     1  /* بنغير فعلاً */

#define SD_SWITCH_GRP_ACCESS   0  /* function group 1: access mode */
#define SD_SWITCH_ACCESS_DEF   0  /* Default speed (25 MHz) */
#define SD_SWITCH_ACCESS_HS    1  /* High speed (50 MHz) */
```

#### 6. Erase vs Discard

```c
#define SD_ERASE_ARG     0x00000000  /* Full erase */
#define SD_DISCARD_ARG   0x00000001  /* Discard — بيخلي الكارت يتصرف في البلوكات */
```

---

### قصة الـ SD Initialization — عشان تفهم السياق

لما بتحط كارت SD في جهازك، الـ kernel بيعمل التالي:

```
Host                         SD Card
  |                              |
  |--- CMD0 (GO_IDLE) --------->|   ← reset
  |                              |
  |--- CMD8 (SEND_IF_COND) ---->|   ← SD_SEND_IF_COND
  |<-- R7 (voltage check) ------|
  |                              |
  |--- ACMD41 (OP_COND) ------->|   ← SD_APP_OP_COND
  |<-- R3 (OCR register) -------|   ← SD_OCR_CCS, SD_OCR_S18R
  |                              |
  |--- CMD3 (REL_ADDR) -------->|   ← SD_SEND_RELATIVE_ADDR
  |<-- R6 (RCA assigned) -------|
  |                              |
  |--- ACMD51 (SEND_SCR) ------>|   ← SD_APP_SEND_SCR
  |<-- SCR register ------------|   ← SCR_SPEC_VER_*, SD_BUS_WIDTH_4
  |                              |
  |--- CMD6 (SWITCH to HS) ---->|   ← SD_SWITCH + SD_SWITCH_ACCESS_HS
  |<-- R1 (status) -------------|
```

كل اسم في الـ flow ده موجود في `sd.h` — الملف هو المرجع اللي بيربط الـ code بالـ spec.

---

### الملفات المرتبطة اللي لازم تعرفها

| الملف | الدور |
|-------|-------|
| `include/linux/mmc/sd.h` | **الملف ده** — SD constants و commands |
| `include/linux/mmc/mmc.h` | نفس الفكرة لـ MMC (eMMC) commands |
| `include/linux/mmc/sdio.h` | commands و registers خاصة بـ SDIO (WiFi/BT cards) |
| `include/linux/mmc/core.h` | الـ core structs للـ request/response |
| `include/linux/mmc/host.h` | الـ host controller interface |
| `include/linux/mmc/card.h` | الـ struct اللي بيمثل الكارت نفسه |
| `drivers/mmc/core/sd.c` | منطق الـ SD initialization اللي بيستخدم `sd.h` |
| `drivers/mmc/core/sd_ops.c` | الـ operations اللي بتبعت الـ commands الموجودة في `sd.h` |
| `drivers/mmc/core/core.c` | الـ MMC core — إدارة الـ bus |
| `drivers/mmc/core/block.c` | طبقة الـ block device فوق الـ MMC |

---

### ملفات الـ Subsystem الكاملة (Core + Headers + HW Drivers)

**Headers:**
- `include/linux/mmc/sd.h` ← الملف ده
- `include/linux/mmc/mmc.h`
- `include/linux/mmc/sdio.h`
- `include/linux/mmc/host.h`
- `include/linux/mmc/card.h`
- `include/linux/mmc/core.h`
- `include/linux/mmc/sd_uhs2.h`
- `include/linux/mmc/pm.h`
- `include/linux/mmc/slot-gpio.h`
- `include/linux/mmc/sdio_func.h`
- `include/linux/mmc/sdio_ids.h`
- `include/uapi/linux/mmc/ioctl.h`

**Core Logic:**
- `drivers/mmc/core/core.c`
- `drivers/mmc/core/sd.c`
- `drivers/mmc/core/sd_ops.c`
- `drivers/mmc/core/mmc.c`
- `drivers/mmc/core/mmc_ops.c`
- `drivers/mmc/core/sdio.c`
- `drivers/mmc/core/block.c`
- `drivers/mmc/core/host.c`
- `drivers/mmc/core/bus.c`

**Hardware Drivers (أمثلة):**
- `drivers/mmc/host/sdhci.c` — SDHCI standard host controller
- `drivers/mmc/host/dw_mmc.c` — DesignWare MMC
- `drivers/mmc/host/bcm2835.c` — Raspberry Pi
- `drivers/mmc/host/omap_hsmmc.c` — Texas Instruments OMAP
## Phase 2: شرح الـ MMC/SD Framework

### المشكلة اللي بيحلها الـ Framework ده

في أي نظام embedded، محتاج توصل بـ storage — سواء كانت SD card في كاميرا، أو eMMC chip على PCB، أو SDIO device زي WiFi module. المشكلة إن:

1. **الـ hardware متنوع جداً**: كل SoC عنده controller مختلف (Qualcomm، TI، Rockchip، ...).
2. **الـ card protocols متعددة**: MMC، SD، SDIO — كلهم بيتكلموا بـ command set مختلف شوية.
3. **الـ Linux block subsystem** محتاج interface موحد للـ read/write.

لو ماكنش فيه framework موحد، كل driver هيحتاج يعرف كل تفاصيل الـ card protocol والـ host controller في نفس الوقت — ده **spaghetti code** مضمون.

---

### الحل اللي اتبعه الـ Kernel

الـ kernel قسّم المسؤولية لـ **3 طبقات واضحة**:

```
┌─────────────────────────────────────────────┐
│           Block Layer (مستخدم الـ storage)   │
│         /dev/mmcblk0, /dev/mmcblk0p1         │
└────────────────┬────────────────────────────┘
                 │ read/write requests
┌────────────────▼────────────────────────────┐
│           MMC Core (القلب)                   │
│  drivers/mmc/core/                           │
│  - Card detection & initialization           │
│  - Protocol handling (SD, MMC, SDIO)         │
│  - Command building & response parsing       │
│  - Power management                          │
└──────┬──────────────────────────────┬────────┘
       │ mmc_host_ops callbacks        │ card-level structs
┌──────▼──────────┐          ┌────────▼────────┐
│  Host Driver    │          │  Card Driver    │
│  (platform-     │          │  (mmc_blk,      │
│  specific)      │          │   sdio_*, ...)  │
│  sdhci-pltfm    │          │                 │
│  meson-gx-mmc   │          │                 │
│  omap_hsmmc     │          │                 │
└─────────────────┘          └─────────────────┘
```

**الـ MMC Core** هو اللي بيتعامل مع الـ SD/MMC protocol commands بالظبط — أما الـ host driver فبيتعامل مع الـ silicon registers بس.

---

### التشبيه الحقيقي

تخيل **مطار دولي**:

| عنصر في المطار | المقابل في الـ MMC Framework |
|---|---|
| **المسافر** (Block Layer) | بيطلب وصول للبيانات |
| **موظف الـ check-in** (MMC Core) | بيترجم الطلب لـ SD commands |
| **الطيار + الطائرة** (Host Driver) | بيشغّل الـ hardware الفعلي |
| **بروتوكول الطيران** (SD Spec / JEDEC) | القواعد الثابتة اللي كلهم بيتبعوها |
| **الـ passport** (Card CID/CSD registers) | هوية الـ card وقدراتها |
| **Gate assignment** (mmc_ios) | إعدادات الاتصال: clock، voltage، bus width |

لكن الأهم: **الطيار مش محتاج يعرف وجهة المسافر**، وموظف الـ check-in مش محتاج يعرف إزاي بتشتغل الطائرة. كل طبقة بتتكلم مع اللي جنبها بس عن طريق interface محدود.

---

### الـ Architecture بالتفصيل

```
  ┌──────────────────────────────────────────────────────┐
  │                    User Space                        │
  │          open("/dev/mmcblk0") / ioctl / read         │
  └────────────────────┬─────────────────────────────────┘
                       │ VFS
  ┌────────────────────▼─────────────────────────────────┐
  │              Block Layer (blk-mq)                    │
  │  - Queue management, I/O scheduling                  │
  └────────────────────┬─────────────────────────────────┘
                       │ bio / request
  ┌────────────────────▼─────────────────────────────────┐
  │    mmc_blk driver  (drivers/mmc/card/block.c)        │
  │  - Maps block requests → mmc_request structs         │
  │  - Handles partitions (boot, rpmb, user)             │
  └────────────────────┬─────────────────────────────────┘
                       │ mmc_start_request()
  ┌────────────────────▼─────────────────────────────────┐
  │              MMC Core                                │
  │  drivers/mmc/core/                                   │
  │                                                      │
  │  ┌─────────────┐  ┌──────────────┐  ┌─────────────┐ │
  │  │  sd.c       │  │  mmc.c       │  │  sdio.c     │ │
  │  │ SD init &   │  │ eMMC init &  │  │ SDIO func   │ │
  │  │ card ops    │  │ EXT_CSD ops  │  │ enumeration │ │
  │  └──────┬──────┘  └──────┬───────┘  └──────┬──────┘ │
  │         └────────────────▼─────────────────┘        │
  │                    core.c / ops.c                    │
  │              mmc_wait_for_cmd()                      │
  │              mmc_set_ios()                           │
  └────────────────────┬─────────────────────────────────┘
                       │ mmc_host_ops function pointers
  ┌────────────────────▼─────────────────────────────────┐
  │         Host Controller Driver                       │
  │  drivers/mmc/host/                                   │
  │  sdhci.c / sdhci-pltfm.c / meson-gx-mmc.c / ...     │
  │  - request() : DMA setup, register writes           │
  │  - set_ios() : clock, voltage, bus width config     │
  │  - execute_tuning() : signal quality calibration    │
  └────────────────────┬─────────────────────────────────┘
                       │ Memory-mapped registers / DMA
  ┌────────────────────▼─────────────────────────────────┐
  │         Physical Hardware                            │
  │  SDHCI Controller ←→ SD/eMMC/SDIO Card              │
  │  (CLK, CMD, DAT0-3 or DAT0-7 lines)                 │
  └──────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction: الـ `mmc_host` و `mmc_card`

الـ MMC framework بيقوم على فكرة بسيطة: **الـ host** هو الـ controller في الـ SoC، والـ **card** هي اللي بتتوصل بيه. كل واحد فيهم عنده struct خاص بيه.

#### الـ `mmc_ios` — حالة الاتصال الحالية

**الـ `struct mmc_ios`** هو snapshot لإعدادات الـ bus في أي لحظة:

```c
struct mmc_ios {
    unsigned int  clock;          /* سرعة الـ clock بالـ Hz */
    unsigned short vdd;           /* voltage index */
    unsigned char bus_mode;       /* open-drain أو push-pull */
    unsigned char power_mode;     /* OFF / UP / ON */
    unsigned char bus_width;      /* 1-bit, 4-bit, 8-bit */
    unsigned char timing;         /* Legacy / HS / UHS-SDR104 / HS400 / ... */
    unsigned char signal_voltage; /* 3.3V أو 1.8V أو 1.2V */
    unsigned char drv_type;       /* A, B, C, D */
    bool enhanced_strobe;         /* HS400ES */
};
```

لما الـ MMC Core يحتاج يغير أي حاجة (مثلاً يرفع الـ clock من 400kHz لـ 50MHz بعد الـ init)، بيعدّل الـ `mmc_ios` وبيستدعي `host->ops->set_ios(host, &host->ios)` — والـ host driver هو اللي يكتب الـ registers الفعلية.

#### الـ `mmc_host_ops` — الـ vtable بتاع الـ host

```c
struct mmc_host_ops {
    void (*request)(struct mmc_host *host, struct mmc_request *req);
    void (*set_ios)(struct mmc_host *host, struct mmc_ios *ios);
    int  (*get_ro)(struct mmc_host *host);
    int  (*get_cd)(struct mmc_host *host);
    int  (*execute_tuning)(struct mmc_host *host, u32 opcode);
    int  (*start_signal_voltage_switch)(struct mmc_host *host,
                                        struct mmc_ios *ios);
    int  (*card_busy)(struct mmc_host *host);
    /* ... */
};
```

ده الـ **contract** بين الـ Core والـ host driver. الـ Core ما بيعرفش أي حاجة عن الـ hardware — بيتكلم بس عن طريق الـ function pointers دي.

#### الـ `mmc_card` — تمثيل الـ card

```c
struct mmc_card {
    struct mmc_host   *host;   /* الـ host اللي متوصلة بيه */
    struct device      dev;    /* Linux device model */
    unsigned int       type;   /* MMC_TYPE_SD, MMC_TYPE_MMC, ... */
    unsigned int       state;  /* present, readonly, suspended, ... */
    struct mmc_cid     cid;    /* Card Identification Data */
    struct mmc_csd     csd;    /* Card Specific Data */
    struct mmc_ext_csd ext_csd;/* Extended CSD (eMMC فقط) */
    struct sd_scr      scr;    /* SD Configuration Register */
    struct sd_ssr      ssr;    /* SD Status Register */
    struct sd_switch_caps sw_caps; /* نتيجة CMD6 */
    /* ... */
};
```

الـ **`mmc_cid`** بيحتوي على manufacturer ID، product name، serial number — ده اللي بيتقرأ بـ `CMD2` (ALL_SEND_CID) وقت الـ enumeration.

الـ **`mmc_csd`** (Card Specific Data) بيحتوي على:
- `max_dtr`: أقصى data transfer rate
- `cmdclass`: الـ command classes المدعومة (CCC bits)
- `capacity`: سعة الـ card بالـ sectors
- `read_blkbits` / `write_blkbits`: حجم الـ block

---

### الـ SD Commands — كيف بيشتغلوا

كل command في الـ SD protocol عنده:
- **رقم** (CMD0, CMD3, ACMD41, ...)
- **نوع**: `bc` (broadcast)، `bcr` (broadcast with response)، `ac` (addressed command)، `adtc` (addressed data transfer command)
- **argument**: 32-bit parameter
- **response type**: R1, R2, R3, R6, R7

```
Host ──CMD────────────────────────────────► Card
      (start bit + direction + index + arg + CRC)

Host ◄─────────────────────────────RESPONSE─ Card
      (R1: 48 bits status | R2: 136 bits CSD/CID)
```

#### تسلسل الـ SD Initialization (من `sd.h` و `sd.c`)

```
Power ON
   │
   ▼
CMD0  (GO_IDLE_STATE)  ← ماكنش فيه response
   │
   ▼
CMD8  (SEND_IF_COND)   ← SD_SEND_IF_COND
   │  arg: [11:8]=voltage, [7:0]=0xAA (check pattern)
   │  response R7: إذا رجع نفس الـ check pattern → SD v2+
   │
   ▼
ACMD41 (SD_APP_OP_COND) ← يتبعه CMD55 أولاً دايماً
   │  arg: [30]=HCS (Host Capacity Support), [24]=S18R (1.8V req)
   │  response R3: OCR register
   │  bit 31 (busy) = 0 → card لسه busy
   │  bit 30 (CCS)  = 1 → SDHC/SDXC (block addressing)
   │
   ▼
CMD2  (ALL_SEND_CID) ← R2 response = 136 bits
   │
   ▼
CMD3  (SD_SEND_RELATIVE_ADDR) ← الـ card تختار RCA
   │  response R6: RCA في bits [31:16]
   │
   ▼
CMD9  (SEND_CSD)  ← بنقرأ الـ CSD بالـ RCA
   │
   ▼
CMD7  (SELECT_CARD) ← الـ card تدخل Transfer State
   │
   ▼
ACMD51 (SEND_SCR) ← نقرأ الـ SCR (SD Configuration Register)
   │  بيعرفنا bus widths المدعومة وإصدار الـ spec
   │
   ▼
CMD6  (SD_SWITCH) ← نحاول نعمل switch لـ High Speed
   │  arg [31]=0 (check) أو 1 (switch)
   │  arg [3:0] = access mode function group
   │
   ▼
ACMD6 (SET_BUS_WIDTH) ← نوسّع لـ 4-bit
```

#### الـ Application Commands (ACMD)

الـ SD spec فيها concept مهم جداً: **الـ ACMD** — أي command بيتسبقه `CMD55` (APP_CMD). الـ card لما بتشوف `CMD55` بتعرف إن الـ command الجاي هو application-specific مش standard:

```c
/* من mmc.h */
#define MMC_APP_CMD  55   /* يخبر الـ card إن اللي جاي ACMD */

/* من sd.h */
#define SD_APP_SET_BUS_WIDTH   6   /* ACMD6  */
#define SD_APP_SD_STATUS      13   /* ACMD13 */
#define SD_APP_OP_COND        41   /* ACMD41 */
#define SD_APP_SEND_SCR       51   /* ACMD51 */
```

الـ Kernel بيبعتهم كده:

```c
/* مثال مبسط من drivers/mmc/core/sd_ops.c */
static int mmc_app_cmd(struct mmc_host *host, struct mmc_card *card)
{
    struct mmc_command cmd = {};
    cmd.opcode = MMC_APP_CMD;          /* CMD55 */
    cmd.arg    = card->rca << 16;      /* RCA في bits العليا */
    cmd.flags  = MMC_RSP_SPI_R1 | MMC_RSP_R1 | MMC_CMD_AC;
    return mmc_wait_for_cmd(host, &cmd, 0);
}
```

---

### الـ OCR Register — التفاوض على الـ Voltage

**الـ OCR (Operating Conditions Register)** هو 32-bit register بيصف الـ voltage ranges المدعومة. من `sd.h`:

```c
#define SD_OCR_S18R   (1 << 24)  /* 1.8V switching request */
#define SD_ROCR_S18A  SD_OCR_S18R /* 1.8V accepted */
#define SD_OCR_XPC    (1 << 28)  /* SDXC power control */
#define SD_OCR_CCS    (1 << 30)  /* Card Capacity Status: 1=SDHC/SDXC */
```

```
OCR bits layout:
Bit 31: Card busy (0=init in progress, 1=ready)
Bit 30: CCS — SDHC/SDXC flag
Bit 28: XPC — SDXC power control
Bit 27: CO2T — SDUC (>2TB) support
Bit 24: S18A — 1.8V accepted
Bits 23-0: Voltage windows (e.g., bit 21 = 3.3-3.4V)
```

الـ MMC Core بيبعت الـ OCR الخاص بالـ host (قدراته) في الـ ACMD41 argument، والـ card بتبعت الـ OCR بتاعها في الـ R3 response — لو فيه تقاطع في الـ voltage windows، الاتصال ممكن.

---

### الـ SCR — SD Configuration Register

**الـ SCR** بيتقرأ بـ ACMD51 وده الـ struct اللي بيمثله:

```c
struct sd_scr {
    unsigned char sda_vsn;      /* SD Memory Card Spec version */
    unsigned char sda_spec3;    /* spec version 3 flag */
    unsigned char sda_spec4;    /* spec version 4 flag */
    unsigned char sda_specx;    /* spec version X flag */
    unsigned char bus_widths;   /* supported bus widths */
    unsigned char cmds;         /* supported commands (CMD20/23/48/58) */
};
```

من `sd.h`:
```c
#define SCR_SPEC_VER_0  0  /* Spec 1.0–1.01 */
#define SCR_SPEC_VER_1  1  /* Spec 1.10    */
#define SCR_SPEC_VER_2  2  /* Spec 2.00–3.0X */

#define SD_BUS_WIDTH_1  0  /* 1-bit mode */
#define SD_BUS_WIDTH_4  2  /* 4-bit mode */
```

---

### الـ SD_SWITCH Command (CMD6) — تفعيل High Speed

**`SD_SWITCH`** هو من أهم commands في الـ SD spec — بيسمح للـ host يغير access mode الـ card:

```c
#define SD_SWITCH         6   /* adtc [31:0] */
#define SD_SWITCH_CHECK   0   /* [31]=0: just check, don't switch */
#define SD_SWITCH_SET     1   /* [31]=1: actually switch */
#define SD_SWITCH_GRP_ACCESS  0   /* Function Group 1: access mode */
#define SD_SWITCH_ACCESS_DEF  0   /* Default speed  */
#define SD_SWITCH_ACCESS_HS   1   /* High speed     */
```

**Argument format** (من التعليقات في `sd.h`):
```
Bit 31:    Mode — 0=check, 1=switch
Bits 30-24: Reserved
Bits 23-20: Function Group 6
Bits 19-16: Function Group 5
Bits 15-12: Function Group 4
Bits 11-8:  Function Group 3
Bits 7-4:   Function Group 2 (Command System)
Bits 3-0:   Function Group 1 (Access Mode)  ← الأهم
```

الـ Card بتبعت response 512-bit data block بيوضح:
- الـ functions المدعومة في كل group
- الـ function الحالية في كل group
- الـ maximum current consumption بعد الـ switch

---

### الـ Voltage Switching — من 3.3V لـ 1.8V

لما الـ card توافق على `S18R` في الـ ACMD41، الـ Core بيعمل:

```
1. يبعت CMD11 (SD_SWITCH_VOLTAGE)
2. يوقف الـ clock
3. يطلب من الـ host driver يخفض الـ signal voltage
   → host->ops->start_signal_voltage_switch()
4. يستنى 5ms
5. يرجّع الـ clock
6. يتحقق إن الـ DAT0 line = 1 (card ready)
```

ده بيفتح الطريق لـ UHS modes (SDR50، SDR104، DDR50) اللي بتشتغل بـ 1.8V بس.

---

### الـ SD_SWITCH_VOLTAGE (CMD11) vs SD_SWITCH (CMD6)

| | CMD11 | CMD6 |
|---|---|---|
| **الاسم** | SWITCH_VOLTAGE | SWITCH |
| **الهدف** | تغيير الـ signal voltage (3.3V→1.8V) | تغيير access mode / speed |
| **متى** | بعد ACMD41 مباشرة | بعد اختيار الـ card (CMD7) |
| **رجوع** | بيوقف الـ clock كجزء من البروتوكول | data block بالـ switch status |

---

### الـ Erase / Discard

```c
#define SD_ERASE_WR_BLK_START  32  /* تحديد بداية نطاق الـ erase */
#define SD_ERASE_WR_BLK_END    33  /* تحديد نهاية نطاق الـ erase */
/* ثم CMD38 (MMC_ERASE) بالـ argument: */
#define SD_ERASE_ARG    0x00000000  /* Erase عادي */
#define SD_DISCARD_ARG  0x00000001  /* Discard (hint للـ card أنها تتجاهل الـ data) */
```

الفرق بين **Erase** و **Discard**:
- **Erase**: الـ card مطلوبة تمسح الـ data فعلياً (بتكتب 0 أو 1 حسب `ERASED_MEM_CONT` في الـ CSD)
- **Discard**: مجرد hint للـ card إنها ممكن تتجاهل الـ data — أسرع بكتير، مفيد للـ wear leveling

---

### ملخص: ايه اللي الـ MMC Core بيمتلكه vs اللي بيفوّضه

```
┌─────────────────────────────────────────────────────────┐
│                    MMC Core يمتلك                       │
├─────────────────────────────────────────────────────────┤
│  • Card initialization state machine                    │
│  • Protocol command sequencing (CMD0→CMD2→CMD3→...)    │
│  • Response parsing (R1, R2, R3, R6, R7)               │
│  • OCR negotiation                                      │
│  • Bus width & speed mode selection logic               │
│  • Voltage switch sequencing                            │
│  • Error handling & retry logic                         │
│  • Card state tracking (idle/ready/ident/stby/tran)    │
│  • Power management (runtime PM, suspend/resume)        │
│  • mmc_card struct population (CID, CSD, SCR, EXT_CSD) │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                 MMC Core يفوّض للـ Host Driver          │
├─────────────────────────────────────────────────────────┤
│  • كتابة الـ hardware registers                        │
│  • DMA setup وإدارة الـ scatter-gather lists           │
│  • الـ clock generation وضبط الـ dividers              │
│  • الـ voltage regulator control                        │
│  • Tuning (تحسين timing على مستوى الـ signal)          │
│  • Card detect / write-protect GPIO pins               │
│  • SDHCI standard أو vendor-specific registers         │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                 MMC Core يفوّض للـ Card Driver          │
├─────────────────────────────────────────────────────────┤
│  • ترجمة block requests لـ SD commands (CMD17/18/24/25)│
│  • إدارة الـ partitions (mmc_blk)                      │
│  • SDIO function management (sdio_*)                   │
│  • Command Queue (CQE) scheduling                      │
└─────────────────────────────────────────────────────────┘
```

---

### العلاقة بين الـ Structs — رسم كامل

```
struct mmc_host
   ├── struct mmc_host_ops *ops       ← vtable للـ host driver
   ├── struct mmc_ios ios             ← حالة الـ bus الحالية
   ├── struct mmc_card *card          ← الـ card المتوصلة (أو NULL)
   ├── struct mmc_clk_phase_map clk_phase_map ← tuning phases
   └── u32 caps / caps2               ← قدرات الـ host (UHS, HS400, ...)

struct mmc_card
   ├── struct mmc_host *host          ← backpointer للـ host
   ├── struct device dev              ← Linux device model integration
   ├── struct mmc_cid cid             ← manufacturer، serial، ...
   ├── struct mmc_csd csd             ← timing، capacity، cmd classes
   ├── struct mmc_ext_csd ext_csd     ← eMMC فقط: partitions، HS200، CMDQ
   ├── struct sd_scr scr              ← SD فقط: spec version، bus widths
   ├── struct sd_ssr ssr              ← SD status: erase info
   ├── struct sd_switch_caps sw_caps  ← High Speed / UHS capabilities
   └── struct sd_ext_reg ext_power    ← SD 4.x Power Management function
              └── struct sd_ext_reg ext_perf ← Performance Enhancement function
```

---

### ملاحظة على SDIO

الـ `sd.h` بيغطي **SD Memory cards** بشكل أساسي، لكن الـ framework كمان بيدعم **SDIO** (زي WiFi، Bluetooth chips). الفرق:
- **SD Memory**: بيستخدم block read/write commands عن طريق `mmc_blk`
- **SDIO**: بيستخدم `CMD52` (IO_RW_DIRECT) و `CMD53` (IO_RW_EXTENDED) للـ register access — ده subsystem منفصل في `drivers/mmc/core/sdio*.c`

الـ `MMC_APP_CMD` (CMD55) بيشتغل بالظبط نفس الطريقة في الحالتين — لأن الـ application command mechanism جزء من الـ SD base spec.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الملف: `include/linux/mmc/sd.h`

الملف ده مش بيعرّف structs — هو **header للـ constants والـ macros** البتوصف بروتوكول SD (Secure Digital) على مستوى الـ wire protocol. كل اللي فيه هو:
- أرقام الـ commands (SD و ACMD)
- بتات الـ OCR register
- ثوابت الـ SCR version
- modes الـ SD_SWITCH

لكن الملف بيتعاشر مع structs في `core.h` — خصوصاً `mmc_command` و`mmc_data` و`mmc_request`. الـ phase ده هيغطي الاتنين مع بعض.

---

### 0. الـ Flags والـ Enums والـ Config Options — Cheatsheet

#### SD Commands (sd.h)

| Macro | رقم CMD | النوع | الـ Response | الوظيفة |
|---|---|---|---|---|
| `SD_SEND_RELATIVE_ADDR` | 3 | bcr | R6 | بيجيب الـ RCA من الكارت |
| `SD_SEND_IF_COND` | 8 | bcr | R7 | بيتحقق من voltage قبل init |
| `SD_SWITCH_VOLTAGE` | 11 | ac | R1 | بيطلب التحويل لـ 1.8V |
| `SD_ADDR_EXT` | 22 | ac | R1 | Extended address للـ SDUC |
| `SD_SWITCH` | 6 | adtc | R1 | Check أو Set speed mode |
| `SD_ERASE_WR_BLK_START` | 32 | ac | R1 | بداية نطاق الـ erase |
| `SD_ERASE_WR_BLK_END` | 33 | ac | R1 | نهاية نطاق الـ erase |
| `SD_READ_EXTR_SINGLE` | 48 | adtc | R1 | قراءة Extension Register |
| `SD_WRITE_EXTR_SINGLE` | 49 | adtc | R1 | كتابة Extension Register |

#### Application Commands — ACMD (بتجي بعد CMD55)

| Macro | رقم ACMD | الـ Response | الوظيفة |
|---|---|---|---|
| `SD_APP_SET_BUS_WIDTH` | 6 | R1 | تحديد عرض الـ bus (1-bit أو 4-bit) |
| `SD_APP_SD_STATUS` | 13 | R1 | بيجيب SD Status Register (512-bit) |
| `SD_APP_SEND_NUM_WR_BLKS` | 22 | R1 | عدد الـ blocks المكتوبة صح |
| `SD_APP_OP_COND` | 41 | R3 | بيبدأ عملية الـ initialization |
| `SD_APP_SEND_SCR` | 51 | R1 | بيجيب الـ SCR register |

#### OCR Bit Flags

| Macro | البت | المعنى |
|---|---|---|
| `SD_OCR_S18R` | bit 24 | الـ host بيطلب تشغيل على 1.8V |
| `SD_ROCR_S18A` | bit 24 | الكارت وافق على 1.8V (نفس القيمة) |
| `SD_OCR_2T` | bit 27 | الكارت بيدعم SDUC (فوق 2TB) |
| `SD_OCR_XPC` | bit 28 | الكارت بيدعم SDXC power control |
| `SD_OCR_CCS` | bit 30 | الكارت High Capacity (SDHC/SDXC) |

#### SCR Spec Versions

| Macro | القيمة | يمثّل |
|---|---|---|
| `SCR_SPEC_VER_0` | 0 | SD Spec 1.0 – 1.01 |
| `SCR_SPEC_VER_1` | 1 | SD Spec 1.10 |
| `SCR_SPEC_VER_2` | 2 | SD Spec 2.00 – 3.0x |

#### SD Bus Width

| Macro | القيمة | المعنى |
|---|---|---|
| `SD_BUS_WIDTH_1` | 0 | 1-bit bus (default) |
| `SD_BUS_WIDTH_4` | 2 | 4-bit bus (UHS/HS) |

#### SD_SWITCH Modes

| Macro | القيمة | المعنى |
|---|---|---|
| `SD_SWITCH_CHECK` | 0 | بس بيجرب من غير ما يغيّر |
| `SD_SWITCH_SET` | 1 | بيطبّق التغيير فعلاً |
| `SD_SWITCH_GRP_ACCESS` | 0 | Function Group 1 (Access Mode) |
| `SD_SWITCH_ACCESS_DEF` | 0 | Default Speed (25 MHz) |
| `SD_SWITCH_ACCESS_HS` | 1 | High Speed (50 MHz) |

#### Erase/Discard Arguments

| Macro | القيمة | المعنى |
|---|---|---|
| `SD_ERASE_ARG` | 0x00000000 | Erase عادي |
| `SD_DISCARD_ARG` | 0x00000001 | Discard (hint للكارت إنه يتجاهل البيانات) |

#### MMC Response Flags (core.h)

| Macro | البت | المعنى |
|---|---|---|
| `MMC_RSP_PRESENT` | bit 0 | في response |
| `MMC_RSP_136` | bit 1 | الـ response طوله 136-bit |
| `MMC_RSP_CRC` | bit 2 | فيه CRC يتحقق منه |
| `MMC_RSP_BUSY` | bit 3 | الكارت ممكن يبعت busy signal |
| `MMC_RSP_OPCODE` | bit 4 | الـ response بيحتوي على opcode |

#### Response Types المُجمَّعة

| Macro | يساوي | متى يُستخدم |
|---|---|---|
| `MMC_RSP_NONE` | 0 | أوامر من غير response |
| `MMC_RSP_R1` | PRESENT+CRC+OPCODE | معظم الـ commands العادية |
| `MMC_RSP_R1B` | R1 + BUSY | أوامر ممكن تستغرق وقت (erase) |
| `MMC_RSP_R2` | PRESENT+136+CRC | CID و CSD registers |
| `MMC_RSP_R3` | PRESENT فقط | OCR (no CRC, no opcode) |
| `MMC_RSP_R6` | PRESENT+CRC+OPCODE | RCA assignment |
| `MMC_RSP_R7` | PRESENT+CRC+OPCODE | IF_COND |

#### Command Type Flags

| Macro | البتات [6:5] | المعنى |
|---|---|---|
| `MMC_CMD_AC` | 00 | Addressed Command (no data) |
| `MMC_CMD_ADTC` | 01 | Addressed Data Transfer Command |
| `MMC_CMD_BC` | 10 | Broadcast Command (no response) |
| `MMC_CMD_BCR` | 11 | Broadcast Command with Response |

#### MMC Data Flags

| Macro | البت | المعنى |
|---|---|---|
| `MMC_DATA_WRITE` | bit 8 | عملية كتابة |
| `MMC_DATA_READ` | bit 9 | عملية قراءة |
| `MMC_DATA_QBR` | bit 10 | CQE Queue Barrier |
| `MMC_DATA_PRIO` | bit 11 | CQE High Priority |
| `MMC_DATA_REL_WR` | bit 12 | Reliable Write |
| `MMC_DATA_DAT_TAG` | bit 13 | Tagged request |
| `MMC_DATA_FORCED_PRG` | bit 14 | Forced programming |

---

### 1. الـ Structs المهمة

#### `struct mmc_command` (core.h)

**الغرض:** بيمثّل أمر واحد بيتبعت للكارت على الـ bus.

| Field | النوع | الوظيفة |
|---|---|---|
| `opcode` | u32 | رقم الأمر (مثلاً SD_SWITCH=6) |
| `arg` | u32 | الـ argument المصاحب للأمر |
| `resp[4]` | u32[4] | الـ response من الكارت (4 كلمات لـ R2، واحدة للباقي) |
| `flags` | unsigned int | نوع الـ response + نوع الأمر |
| `retries` | unsigned int | عدد مرات إعادة المحاولة عند الفشل |
| `error` | int | كود الخطأ (0 = نجاح، errno عند فشل) |
| `busy_timeout` | unsigned int | timeout بالـ milliseconds لـ busy detect |
| `data` | `*mmc_data` | لو الأمر بيتبع بيانات، بيشاور عليها |
| `mrq` | `*mmc_request` | الـ request الأم اللي الأمر ده جزء منه |
| `uhs2_cmd` | `*uhs2_command` | للأوامر على بروتوكول UHS-II |
| `has_ext_addr` | bool | للـ SDUC — لو فيه extended address |
| `ext_addr` | u8 | الـ extended address للـ SDUC (>2TB) |

---

#### `struct mmc_data` (core.h)

**الغرض:** بيوصف chunk البيانات اللي هتتنقل (read أو write).

| Field | النوع | الوظيفة |
|---|---|---|
| `timeout_ns` | unsigned int | timeout بالـ nanoseconds (max ~80ms) |
| `timeout_clks` | unsigned int | timeout بعدد الـ clock cycles |
| `blksz` | unsigned int | حجم الـ block بالـ bytes (عادةً 512) |
| `blocks` | unsigned int | عدد الـ blocks المطلوبة |
| `blk_addr` | unsigned int | عنوان أول block |
| `error` | int | كود خطأ البيانات |
| `flags` | unsigned int | READ أو WRITE + CQE flags |
| `bytes_xfered` | unsigned int | عدد الـ bytes اللي اتنقلت فعلاً |
| `stop` | `*mmc_command` | أمر STOP_TRANSMISSION بعد multiblock |
| `mrq` | `*mmc_request` | الـ request الأم |
| `sg_len` | unsigned int | عدد الـ scatter-gather entries |
| `sg_count` | int | عدد الـ entries المعمول ليها DMA map |
| `sg` | `*scatterlist` | الـ scatter list (بيوصف الـ memory buffers) |
| `host_cookie` | s32 | private data للـ host controller |

---

#### `struct mmc_request` (core.h)

**الغرض:** الـ "envelope" اللي بيجمع الأوامر والبيانات في طلب واحد متكامل.

| Field | النوع | الوظيفة |
|---|---|---|
| `sbc` | `*mmc_command` | أمر SET_BLOCK_COUNT قبل الـ multiblock |
| `cmd` | `*mmc_command` | الأمر الرئيسي |
| `data` | `*mmc_data` | البيانات المصاحبة (ممكن NULL) |
| `stop` | `*mmc_command` | أمر الإيقاف بعد النقل |
| `completion` | struct completion | للانتظار المتزامن على انتهاء الـ request |
| `cmd_completion` | struct completion | للانتظار على انتهاء الأمر تحديداً |
| `done` | function pointer | callback بيتنادى لما الـ request يخلص |
| `recovery_notifier` | function pointer | callback لإبلاغ الـ upper layers بالأخطاء (CQE) |
| `host` | `*mmc_host` | الـ host controller المسؤول |
| `cap_cmd_during_tfr` | bool | سماح بأوامر تانية خلال النقل |
| `tag` | int | معرّف للـ CQE (Command Queue Engine) |
| `crypto_ctx` | `*bio_crypt_ctx` | سياق التشفير (لو CONFIG_MMC_CRYPTO) |
| `crypto_key_slot` | int | رقم slot المفتاح (لو CONFIG_MMC_CRYPTO) |
| `uhs2_cmd` | struct uhs2_command | الأمر بصيغة UHS-II (embedded مش pointer) |

---

#### `struct uhs2_command` (core.h)

**الغرض:** بيمثّل أمر UHS-II — الجيل الجديد من SD اللي بيشتغل على packet-based protocol.

| Field | النوع | الوظيفة |
|---|---|---|
| `header` | u16 | UHS-II packet header |
| `arg` | u16 | argument مختلف عن SD عادي |
| `payload[2]` | __be32[2] | payload بالـ big-endian |
| `payload_len` | u8 | طول الـ payload بالـ bytes |
| `packet_len` | u8 | إجمالي طول الـ packet |
| `tmode_half_duplex` | u8 | هل الـ transfer نصف duplex؟ |
| `uhs2_resp[20]` | u8[20] | الـ response الخام من الكارت |
| `uhs2_resp_len` | u8 | طول الـ response |

---

### 2. علاقات الـ Structs

```
┌─────────────────────────────────────────────────────────────┐
│                      mmc_request                            │
│                                                             │
│  *sbc ──────────────► mmc_command (SET_BLOCK_COUNT)         │
│  *cmd ──────────────► mmc_command (الأمر الرئيسي)           │
│  *data ─────────────► mmc_data                              │
│  *stop ─────────────► mmc_command (STOP_TRANSMISSION)       │
│  *host ─────────────► mmc_host                              │
│  uhs2_cmd [embedded]─► uhs2_command                         │
│  done() callback                                            │
│  completion                                                 │
└─────────────────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────┐        ┌──────────────────────────┐
│    mmc_command       │        │       mmc_data            │
│                      │        │                           │
│  opcode (SD cmd #)   │        │  blksz, blocks            │
│  arg                 │        │  flags (READ/WRITE)       │
│  resp[4]             │        │  *sg (scatterlist)        │
│  flags               │        │  *stop ──► mmc_command    │
│  *data ─────────────►│        │  *mrq ──► mmc_request     │
│  *mrq ──────────────►│        │  host_cookie              │
│  *uhs2_cmd ─────────►│        └──────────────────────────┘
│  has_ext_addr        │
│  ext_addr (SDUC)     │
└──────────────────────┘
           │
           ▼
┌──────────────────────┐
│   uhs2_command       │
│                      │
│  header, arg         │
│  payload[2]          │
│  uhs2_resp[20]       │
└──────────────────────┘
```

---

### 3. Lifecycle Diagram — دورة حياة SD Command

```
[1] CARD DETECTION
    │
    ▼
[2] POWER ON SEQUENCE
    │  ─ CMD0 (GO_IDLE): كارت يرجع للـ idle
    │  ─ CMD8 / SD_SEND_IF_COND: check voltage
    │    arg: [11:8]=host voltage, [7:0]=0xAA (check pattern)
    │    response R7 لازم يرجع نفس الـ check pattern
    │
    ▼
[3] INITIALIZATION (ACMD41 / SD_APP_OP_COND)
    │  ─ CMD55 أولاً (يفتح باب ACMD)
    │  ─ ACMD41 مع OCR:
    │    SD_OCR_CCS=1       → نبيّن إننا بندعم SDHC/SDXC
    │    SD_OCR_S18R=1      → نطلب 1.8V لو الـ host بيدعم UHS
    │    SD_OCR_XPC=1       → نطلب SDXC power control
    │  ─ نكرر لحد ما الكارت يرد بـ OCR[31]=1 (ready)
    │
    ▼
[4] IDENTIFICATION
    │  ─ CMD2 (ALL_SEND_CID): بيجيب CID (R2, 136-bit)
    │  ─ CMD3 / SD_SEND_RELATIVE_ADDR: بيجيب RCA (R6)
    │
    ▼
[5] VOLTAGE SWITCH (لو UHS)
    │  ─ CMD11 / SD_SWITCH_VOLTAGE
    │  ─ الكارت بيسكت، الـ host بيسكّر الـ clock
    │  ─ الاتنين يسوّيوا switch لـ 1.8V
    │
    ▼
[6] CONFIGURATION
    │  ─ CMD9: SEND_CSD (R2) — يجيب Card Specific Data
    │  ─ CMD7: SELECT_CARD — يحط الكارت في Transfer State
    │  ─ ACMD51 / SD_APP_SEND_SCR: يجيب SCR
    │    بيعرف: spec version, bus width support
    │  ─ CMD6 / SD_SWITCH (CHECK mode أولاً):
    │    SD_SWITCH_GRP_ACCESS → SD_SWITCH_ACCESS_HS
    │    لو مدعوم → SD_SWITCH (SET mode)
    │  ─ ACMD6 / SD_APP_SET_BUS_WIDTH:
    │    SD_BUS_WIDTH_4 = 2 → 4-bit mode
    │
    ▼
[7] DATA TRANSFER
    │  ─ mmc_request بيتبنى
    │  ─ mmc_command (READ/WRITE cmd)
    │  ─ mmc_data (sg list, blksz, blocks)
    │  ─ mmc_wait_for_req() أو async
    │
    ▼
[8] ERASE/DISCARD (اختياري)
    │  ─ CMD32 / SD_ERASE_WR_BLK_START
    │  ─ CMD33 / SD_ERASE_WR_BLK_END
    │  ─ CMD38 / ERASE مع arg:
    │    SD_ERASE_ARG=0x0   → erase حقيقي
    │    SD_DISCARD_ARG=0x1 → discard (hint فقط)
    │
    ▼
[9] DESELECT / POWER OFF
```

---

### 4. Call Flow Diagrams

#### SD Initialization Flow

```
mmc_attach_sd()                          [drivers/mmc/core/sd.c]
  │
  ├─► mmc_send_if_cond()
  │     builds mmc_command: SD_SEND_IF_COND (8)
  │     arg = [11:8 voltage | 0xAA pattern]
  │     flags = MMC_RSP_R7 | MMC_CMD_BCR
  │     calls mmc_wait_for_cmd()
  │       → host->ops->request(host, mrq)
  │         → hardware sends CMD8 on bus
  │         → reads R7 response
  │
  ├─► mmc_send_app_op_cond()
  │     CMD55 + ACMD41
  │     arg includes SD_OCR_CCS | SD_OCR_S18R | SD_OCR_XPC
  │     loops until OCR[31]=1
  │
  ├─► mmc_sd_get_cid() → CMD2 (R2)
  │
  ├─► mmc_send_relative_addr() → SD_SEND_RELATIVE_ADDR (CMD3)
  │
  ├─► mmc_sd_switch_hs()
  │     builds mmc_command: SD_SWITCH (6)
  │     arg = [31]=0 (CHECK), [3:0]=SD_SWITCH_ACCESS_HS
  │     checks response data
  │     إذا supported → arg=[31]=1 (SET)
  │
  └─► mmc_app_set_bus_width()
        CMD55 + ACMD6
        arg = SD_BUS_WIDTH_4 (=2)
        flags = MMC_RSP_R1 | MMC_CMD_AC
```

#### Data Read Flow

```
block driver calls mmc_blk_mq_issue_rq()
  │
  ├─► builds mmc_request mrq
  │     mrq.cmd  = { opcode=READ_MULTIPLE_BLOCK, arg=blk_addr }
  │     mrq.data = { blksz=512, blocks=N, flags=MMC_DATA_READ, sg=... }
  │     mrq.stop = { opcode=STOP_TRANSMISSION }
  │
  └─► mmc_wait_for_req(host, mrq)
        │
        ├─► host->ops->pre_req(host, mrq)   [optional DMA prep]
        │
        ├─► host->ops->request(host, mrq)   [submit to HW]
        │     controller programs DMA from mrq.data.sg
        │     sends CMD18 on bus
        │     data flows: CARD → DMA → memory
        │     sends CMD12 (STOP)
        │
        ├─► wait_for_completion(&mrq.completion)
        │
        └─► host->ops->post_req(host, mrq)  [optional DMA cleanup]
              mrq.done() called
              → block layer notified
```

#### SD_SWITCH Argument Construction

```c
/* arg layout for CMD6 */
u32 arg = 0;
arg |= (SD_SWITCH_SET    << 31);  /* bit 31: 0=check, 1=set */
arg |= (0xF              << 16);  /* bits 23:20,19:16,15:12,11:8: keep other groups */
arg |= (0xF              <<  8);  /* group 3 */
arg |= (0xF              <<  4);  /* group 2 */
arg |= SD_SWITCH_ACCESS_HS;       /* bits 3:0: group 1 = High Speed */
```

#### ACMD Flow (كل ACMD بيحتاج CMD55 قبله)

```
send_app_cmd()
  │
  ├─► mmc_command cmd55 = {
  │     .opcode = MMC_APP_CMD,   /* 55 */
  │     .arg    = card->rca << 16,
  │     .flags  = MMC_RSP_R1 | MMC_CMD_AC
  │   }
  │   mmc_wait_for_cmd(host, &cmd55)
  │   check R1 response: bit 5 (APP_CMD) must be set
  │
  └─► mmc_command acmd = {
        .opcode = SD_APP_OP_COND,  /* أو أي ACMD تاني */
        .arg    = ocr_with_flags,
        .flags  = MMC_RSP_R3 | MMC_CMD_BCR
      }
      mmc_wait_for_cmd(host, &acmd)
```

---

### 5. ASCII Diagram — SD_SWITCH Argument Bit Layout

```
 31      24 23   20 19   16 15   12 11    8  7    4  3    0
┌──────────┬───────┬───────┬───────┬───────┬───────┬───────┐
│M│ reserv │ GRP6  │ GRP5  │ GRP4  │ GRP3  │ GRP2  │ GRP1  │
└──────────┴───────┴───────┴───────┴───────┴───────┴───────┘
 ▲                                                   ▲
 M=0: check                              0=Default Speed
 M=1: set                                1=High Speed (HS)
                                         2=SDR50
                                         3=SDR104
                                         4=DDR50
                                         0xF=no change
```

---

### 6. ASCII Diagram — OCR Register (ACMD41 response)

```
 31  30  29  28  27  26        24  23               0
┌───┬───┬───┬───┬───┬──────────┬───┬────────────────┐
│pwr│CCS│ 0 │XPC│CO2│ reserved │S18│  Voltage Range │
└───┴───┴───┴───┴───┴──────────┴───┴────────────────┘
  ▲   ▲       ▲   ▲              ▲
  │   │       │   │              └── SD_OCR_S18R: 1.8V accepted
  │   │       │   └── SD_OCR_2T: SDUC (>2TB) support
  │   │       └── SD_OCR_XPC: SDXC power control
  │   └── SD_OCR_CCS: 1=High Capacity (SDHC/SDXC/SDUC)
  └── power up status (1=ready)
```

---

### 7. استراتيجية الـ Locking

الملف `sd.h` نفسه ما بيعرّفش locks — الـ locking بيتعمل في الـ core layer. لكن بما إن الـ structs دي بتتستخدم في كل عمليات SD، مهم نعرف إزاي بتتحمى.

#### الـ Locks الرئيسية في MMC Core

| الـ Lock | نوعه | بيحمي إيه |
|---|---|---|
| `mmc_host.lock` | spinlock | الـ host state والـ bus operations |
| `mmc_host.claim_mutex` / `claimed` | mutex + atomic | الـ bus ownership (claim/release) |
| `mmc_card.lock` | spinlock (نادر) | بعض الـ card state fields |

#### قواعد الـ Locking

```
┌─────────────────────────────────────────────────────────┐
│                    Bus Ownership Model                  │
│                                                         │
│  caller → mmc_claim_host(host)                          │
│             └─ waits on host->wq if bus is busy         │
│             └─ sets host->claimed = 1                   │
│                                                         │
│  [CRITICAL SECTION — only one owner at a time]          │
│    mmc_wait_for_req() / mmc_wait_for_cmd()              │
│    builds mmc_request, submits to host->ops->request()  │
│    waits on mrq->completion                             │
│    (completion signaled from interrupt context via       │
│     host->ops->request done callback)                   │
│                                                         │
│  caller → mmc_release_host(host)                        │
│             └─ clears host->claimed                     │
│             └─ wakes up waiters on host->wq             │
└─────────────────────────────────────────────────────────┘
```

#### ترتيب الـ Locking (Lock Ordering)

```
مش مسموح تاخد أكتر من lock في نفس الوقت بالترتيب ده:

1. mmc_host.claim_mutex  (الأعلى — bus ownership)
         │
         ▼
2. mmc_host.lock         (spinlock — للـ state الداخلي)
         │
         ▼
3. (interrupt disabled)  (لما بنشتغل مع الـ DMA/IRQ callbacks)
```

- الـ `mmc_request` و`mmc_command` و`mmc_data` كلها بتتبنى على الـ stack أو بتتحجز بـ `kmalloc` من جانب الـ caller — مش بتتشارك بين threads في نفس الوقت لأن الـ bus claim بيضمن serialization.
- الـ `sg` (scatterlist) بيتعمله DMA map قبل الـ request وunmap بعده — ده اللي بيحميه `host_cookie`.
- لو الـ `cap_cmd_during_tfr` = true في `mmc_request`، ممكن تبعت أوامر تانية خلال النقل (مثلاً status polling) لكن بضمانات من الـ host controller.
## Phase 4: شرح الـ Functions

> **ملاحظة مهمة:** الـ `include/linux/mmc/sd.h` هو header خالص من الـ macros فقط — مفيش functions أو structs أو APIs. الـ Phase ده بيشرح كل macro بالعمق من منظور kernel developer محترف، موضحًا دورها في الـ SD subsystem.

---

### ملخص الـ Macros (Cheatsheet)

#### SD Commands

| Macro | Value | Command Type | Response | الوظيفة |
|---|---|---|---|---|
| `SD_SEND_RELATIVE_ADDR` | 3 | bcr | R6 | يطلب من الـ card إنه يولّد الـ RCA بتاعه |
| `SD_SEND_IF_COND` | 8 | bcr | R7 | يتحقق من توافق الـ voltage ويعرف SDHC/SDXC support |
| `SD_SWITCH_VOLTAGE` | 11 | ac | R1 | يطلب التحويل من 3.3V لـ 1.8V (UHS-I) |
| `SD_ADDR_EXT` | 22 | ac | R1 | يضبط الـ extended address للـ SDUC (class 2) |
| `SD_SWITCH` | 6 | adtc | R1 | يتحقق من أو يضبط الـ bus speed mode (class 10) |
| `SD_ERASE_WR_BLK_START` | 32 | ac | R1 | يحدد بداية نطاق الـ erase |
| `SD_ERASE_WR_BLK_END` | 33 | ac | R1 | يحدد نهاية نطاق الـ erase |

#### Application Commands (ACMD — يسبقها CMD55)

| Macro | Value | Command Type | Response | الوظيفة |
|---|---|---|---|---|
| `SD_APP_SET_BUS_WIDTH` | 6 | ac | R1 | يضبط عرض الـ bus (1-bit أو 4-bit) |
| `SD_APP_SD_STATUS` | 13 | adtc | R1 | يقرأ الـ SD Status Register (512-bit) |
| `SD_APP_SEND_NUM_WR_BLKS` | 22 | adtc | R1 | يقرأ عدد الـ blocks المكتوبة بنجاح |
| `SD_APP_OP_COND` | 41 | bcr | R3 | يرسل الـ OCR ويبدأ initialization |
| `SD_APP_SEND_SCR` | 51 | adtc | R1 | يقرأ الـ SD Configuration Register |

#### Extended Function Commands (Class 11)

| Macro | Value | Command Type | Response | الوظيفة |
|---|---|---|---|---|
| `SD_READ_EXTR_SINGLE` | 48 | adtc | R1 | يقرأ 512-byte من الـ extension register |
| `SD_WRITE_EXTR_SINGLE` | 49 | adtc | R1 | يكتب 512-byte في الـ extension register |

#### OCR Bit Definitions

| Macro | Bit | الوظيفة |
|---|---|---|
| `SD_OCR_S18R` | 24 | الـ host بيطلب التحويل لـ 1.8V |
| `SD_ROCR_S18A` | 24 | الـ card بيقبل التحويل لـ 1.8V (نفس الـ bit في الـ response) |
| `SD_OCR_2T` | 27 | الـ card بيدعم SDUC (capacity > 2TB) |
| `SD_OCR_XPC` | 28 | الـ host بيدعم SDXC power control (150mA+) |
| `SD_OCR_CCS` | 30 | Card Capacity Status — 1 = SDHC/SDXC/SDUC |

#### SCR Spec Versions

| Macro | Value | SD Spec Version |
|---|---|---|
| `SCR_SPEC_VER_0` | 0 | 1.0 – 1.01 |
| `SCR_SPEC_VER_1` | 1 | 1.10 |
| `SCR_SPEC_VER_2` | 2 | 2.00 – 3.0X |

#### Bus Width, Switch Mode & Groups

| Macro | Value | الوظيفة |
|---|---|---|
| `SD_BUS_WIDTH_1` | 0 | 1-bit bus width argument لـ ACMD6 |
| `SD_BUS_WIDTH_4` | 2 | 4-bit bus width argument لـ ACMD6 |
| `SD_SWITCH_CHECK` | 0 | CMD6 في mode الفحص فقط (بدون تغيير) |
| `SD_SWITCH_SET` | 1 | CMD6 في mode الضبط الفعلي |
| `SD_SWITCH_GRP_ACCESS` | 0 | Function Group 1 — Access Mode (speed) |
| `SD_SWITCH_ACCESS_DEF` | 0 | Default speed (25 MHz) |
| `SD_SWITCH_ACCESS_HS` | 1 | High speed (50 MHz) |

#### Erase Arguments

| Macro | Value | الوظيفة |
|---|---|---|
| `SD_ERASE_ARG` | 0x00000000 | Standard erase |
| `SD_DISCARD_ARG` | 0x00000001 | Discard — الـ card ممكن يحتفظ بالبيانات لو حبّ |

---

### Group 1: SD Standard Commands (CMD3, CMD8, CMD11, CMD22, CMD6, CMD32, CMD33)

الـ group ده بيغطي الـ commands الأساسية اللي بيبعتها الـ host للـ SD card عبر الـ CMD line. كل command ليه نوع محدد:

- **bcr** (broadcast with response): الـ host بيبعت لكل الـ cards على الـ bus.
- **ac** (addressed command): addressed لـ card معين بدون data transfer.
- **adtc** (addressed data transfer command): فيه data على الـ DAT lines.

---

#### `SD_SEND_RELATIVE_ADDR` (CMD3)

```c
#define SD_SEND_RELATIVE_ADDR     3   /* bcr                     R6  */
```

**الـ RCA (Relative Card Address)** هو عنوان 16-bit بيتولد أثناء الـ initialization. الـ host بيبعت CMD3 كـ broadcast، والـ card بترد بـ R6 اللي فيها الـ RCA الجديد. بعد كده، الـ host بيستخدم الـ RCA في كل أوامر الـ addressing. ده بيخلي أكتر من card ممكن تتعرف على الـ bus بعناوين مختلفة.

**في الكود:** الـ kernel بيستخدمه في `mmc_sd_get_cid()` و`mmc_sd_init_card()` جوه `drivers/mmc/core/sd.c`.

**الـ Response R6:**
```
[31:16] New RCA
[15:0]  Card Status bits (subset)
```

---

#### `SD_SEND_IF_COND` (CMD8)

```c
#define SD_SEND_IF_COND           8   /* bcr  [11:0] See below   R7  */
```

**CMD8** هو الأمر اللي بيميّز الـ SDHC/SDXC/SDUC عن الـ SD 1.x. لو الـ card مش بتفهم CMD8، يبقى ده SD 1.x وmmax capacity 2GB. لو ردّت بـ R7 صح، يبقى بتدعم الـ Physical Layer Spec 2.0+.

**الـ argument format:**
```
[31:12] Reserved (0)
[11:8]  VHS — Host Voltage Supply: 0b0001 = 2.7-3.6V
[7:0]   Check Pattern: 0xAA (بيتحقق منها في الـ response)
```

**الـ Response R7:**
```
[31:28] Command index (8)
[27:12] Reserved
[11:8]  Voltage accepted
[7:0]   Echo-back of check pattern
```

لو الـ check pattern مش متطابق، الـ card غالبًا عندها مشكلة.

---

#### `SD_SWITCH_VOLTAGE` (CMD11)

```c
#define SD_SWITCH_VOLTAGE         11  /* ac                      R1  */
```

بيطلب التحويل من **3.3V إلى 1.8V signaling** عشان يدعم UHS-I modes (SDR50, SDR104, DDR50). الـ sequence معقدة:

```
Host → CMD11 → Card responds R1
Host يوقف الـ CLK لـ 5ms
Host يخفض الـ VDD على الـ DAT lines
Host يرجع الـ CLK
Card تعمل مقارنة وترد بـ DAT[3:0] = 1111b
```

لو أي خطوة فشلت، لازم الـ host يعمل power cycle للـ card.

**في الكود:** `mmc_sd_init_uhs_card()` في `drivers/mmc/core/sd.c` هي اللي بتنظّم الـ voltage switch بالكامل.

---

#### `SD_ADDR_EXT` (CMD22) — Class 2

```c
#define SD_ADDR_EXT		 22   /* ac   [5:0]              R1  */
```

**SDUC** (SD Ultra Capacity) بيدعم capacity أكبر من 2TB. الـ addressing بيحتاج أكتر من 32-bit، فـCMD22 بيضبط الـ **upper 6 bits** من الـ address (bits 32-37) قبل ما الـ host يبعت أي read/write command. ده بيخلي الـ addressing كله 38-bit.

---

#### `SD_SWITCH` (CMD6) — Class 10

```c
#define SD_SWITCH                 6   /* adtc [31:0] See below   R1  */
```

**CMD6** هو الأساس في تغيير الـ bus speed mode. بيشتغل في mode اتنين:

```c
/* Check mode — فقط استعلام */
arg = (SD_SWITCH_CHECK << 31) | (0xFFFFFF & function_groups);

/* Set mode — تغيير فعلي */
arg = (SD_SWITCH_SET << 31) | (0xFFFFFF & function_groups);
```

**الـ argument breakdown:**
```
[31]    Mode: 0=Check, 1=Set
[30:24] Reserved
[23:20] Function Group 6 (Reserved)
[19:16] Function Group 5 (Reserved)
[15:12] Function Group 4 (Power Limit)
[11:8]  Function Group 3 (Drive Strength)
[7:4]   Function Group 2 (Command System)
[3:0]   Function Group 1 (Access Mode / Speed)
```

الرد عبارة عن 512-bit status structure على الـ DAT lines بتوضح الـ functions المدعومة والمفعّلة حاليًا.

**في الكود:** `mmc_sd_switch()` في `drivers/mmc/core/sd_ops.c`:

```c
/* Simplified flow of mmc_sd_switch() */
int mmc_sd_switch(struct mmc_card *card, int mode, int group, u8 value, u8 *resp)
{
    /* Build 32-bit argument */
    arg = mode << 31 | 0x00FFFFFF;
    arg &= ~(0xF << (group * 4));
    arg |= value << (group * 4);

    /* Send CMD6 with adtc — 64-byte response on DAT */
    return mmc_app_cmd(card->host, NULL) + mmc_wait_for_cmd(...);
}
```

---

#### `SD_ERASE_WR_BLK_START` / `SD_ERASE_WR_BLK_END` (CMD32 / CMD33)

```c
#define SD_ERASE_WR_BLK_START    32   /* ac   [31:0] data addr   R1  */
#define SD_ERASE_WR_BLK_END      33   /* ac   [31:0] data addr   R1  */
```

الـ erase sequence في SD بيتكون من 3 steps:
1. CMD32: تحديد start block address.
2. CMD33: تحديد end block address.
3. CMD38: تنفيذ الـ erase مع argument يحدد النوع (`SD_ERASE_ARG` أو `SD_DISCARD_ARG`).

للـ SDHC/SDXC، الـ addresses هي block addresses. للـ SD 1.x، هي byte addresses.

---

### Group 2: Application Commands (ACMD)

الـ ACMDs بتسبقها دايمًا **CMD55** اللي بتقول للـ card إن الأمر الجاي هو application-specific. الـ host بيبعت CMD55 مع الـ RCA، والـ card بترد بـ R1 وبيكون فيها `APP_CMD` bit = 1.

---

#### `SD_APP_SET_BUS_WIDTH` (ACMD6)

```c
#define SD_APP_SET_BUS_WIDTH      6   /* ac   [1:0] bus width    R1  */
```

بيضبط عرض الـ data bus بين الـ host والـ card:

| Argument Value | Bus Width | Macro |
|---|---|---|
| `0b00` | 1-bit | `SD_BUS_WIDTH_1` |
| `0b10` | 4-bit | `SD_BUS_WIDTH_4` |

لازم الـ host controller يعمل نفس الضبط على الـ hardware register بتاعه بعد نجاح الأمر. الـ 4-bit mode بيرفع الـ throughput x4 على نفس الـ clock frequency.

**في الكود:** `mmc_app_set_bus_width()` في `drivers/mmc/core/sd_ops.c`.

---

#### `SD_APP_SD_STATUS` (ACMD13)

```c
#define SD_APP_SD_STATUS         13   /* adtc                    R1  */
```

بيقرأ الـ **SD Status** وهو 512-bit (64 bytes) register بيوفر معلومات عن:
- الـ UHS speed grade
- الـ AU Size (Allocation Unit)
- عدد الـ erase operations المنتظرة
- الـ performance move info

الـ kernel بيستخدمه في `mmc_read_ssr()` في `drivers/mmc/core/sd.c` عشان يملأ الـ `struct sd_ssr`.

---

#### `SD_APP_SEND_NUM_WR_BLKS` (ACMD22)

```c
#define SD_APP_SEND_NUM_WR_BLKS  22   /* adtc                    R1  */
```

بعد كل write operation، الـ host ممكن يسأل الـ card: كام block اتكتب بنجاح فعلًا قبل ما يحصل error؟ ده مفيد جدًا في الـ error recovery عشان تعرف تكمل من الـ block الصح.

---

#### `SD_APP_OP_COND` (ACMD41)

```c
#define SD_APP_OP_COND           41   /* bcr  [31:0] OCR         R3  */
```

ده **أهم ACMD** في الـ initialization sequence. الـ host بيبعته في loop حتى الـ card تجهز:

```c
/* Simplified initialization loop */
do {
    mmc_send_app_op_cond(host, ocr, &rocr);
    /* Card busy → bit 31 of rocr = 0 */
} while (!(rocr & 0x80000000) && timeout--);
```

**الـ argument (OCR بيبعته الـ host):**
```
[31]    Reserved
[30]    HCS — Host Capacity Support (1 = supports SDHC/SDXC)
[28]    XPC — SDXC Power Control
[24]    S18R — 1.8V request
[23:0]  VDD Voltage Window
```

**الـ response R3 (OCR بيرجعه الـ card):**
```
[31]    Busy bit (0 = still initializing)
[30]    CCS — Card Capacity Status
[24]    S18A — 1.8V accepted
[23:0]  Supported voltage range
```

---

#### `SD_APP_SEND_SCR` (ACMD51)

```c
#define SD_APP_SEND_SCR          51   /* adtc                    R1  */
```

بيقرأ الـ **SCR (SD Configuration Register)** وهو 64-bit register بيحتوي على:

```
[63:60] SCR_STRUCTURE     — SCR structure version
[59:56] SD_SPEC           — SD Memory Card Spec Version
[55]    DATA_STAT_AFTER_ERASE
[54:52] SD_SECURITY
[51:48] SD_BUS_WIDTHS     — bit 0: 1-bit, bit 2: 4-bit supported
[47]    SD_SPEC3          — extends SD_SPEC for v3.0+
[46:43] EX_SECURITY
[42]    SD_SPEC4          — SD spec 4.xx support
[41:38] SD_SPECX          — SD spec 5.xx and above
[37:34] Reserved
[33:32] CMD_SUPPORT       — CMD23/CMD48/CMD49/CMD58/CMD59 support
[31:0]  Reserved
```

الـ kernel بيستخدم `mmc_app_send_scr()` في `drivers/mmc/core/sd_ops.c` ثم بيفسّرها في `mmc_decode_scr()`.

الـ `SCR_SPEC_VER_*` macros بتتقارن مع الـ `SD_SPEC` field:

```c
/* من drivers/mmc/core/sd.c */
if (card->scr.sda_vsn == SCR_SPEC_VER_2) {
    /* يدعم High Speed mode */
}
```

---

### Group 3: Extended Function Commands — Class 11 (CMD48, CMD49)

```c
#define SD_READ_EXTR_SINGLE      48   /* adtc [31:0]             R1  */
#define SD_WRITE_EXTR_SINGLE     49   /* adtc [31:0]             R1  */
```

الـ **Extension Registers** هي ميزة SD spec 5.1+، بتسمح لـ vendor-specific أو standard extensions إنهم يضيفوا functionality جديدة بدون تعديل الـ core spec. كل extension عنده register space 512-byte.

**الـ argument format:**
```
[31]    MIO — Memory/IO: 0=Memory, 1=IO
[30:27] FNO — Function Number (0-15)
[26]    BUS_ERR_INT — Enable bus error interrupt
[25:9]  ADDR — Register address within the function space
[8:0]   LEN — Number of bytes to transfer (1-512)
```

الـ kernel بيستخدمهم في دعم الـ **Performance Enhancement** extension والـ **Power Management** extension.

---

### Group 4: OCR Bit Macros

الـ **OCR (Operating Conditions Register)** بيتبادل بين الـ host والـ card أثناء الـ initialization عشان يتفقوا على الـ voltage والـ capabilities.

---

#### `SD_OCR_S18R` / `SD_ROCR_S18A` (Bit 24)

```c
#define SD_OCR_S18R     (1 << 24)   /* 1.8V switching request */
#define SD_ROCR_S18A    SD_OCR_S18R /* 1.8V switching accepted */
```

نفس الـ bit بس له معنيين حسب اتجاه الإرسال:
- الـ host بيبعته في ACMD41 argument كـ **S18R** = request.
- الـ card بترجعه في R3 response كـ **S18A** = accepted.

الـ UHS-I مش هيشتغل من غير الـ voltage switch ده. لو الـ card مش بتقبل، الـ host يفضل على 3.3V.

---

#### `SD_OCR_2T` (Bit 27)

```c
#define SD_OCR_2T       (1 << 27)   /* HO2T/CO2T - SDUC support */
```

**HO2T** (Host Over 2TB) و **CO2T** (Card Over 2TB) — نفس الـ bit كمان. الـ SDUC cards بتدعم capacity أكبر من 2TB وبتحتاج CMD22 لـ extended addressing.

---

#### `SD_OCR_XPC` (Bit 28)

```c
#define SD_OCR_XPC      (1 << 28)   /* SDXC power control */
```

الـ SDXC cards محتاجة طاقة أكتر. الـ host بيبعت الـ bit ده في ACMD41 عشان يعلن إنه قادر يوفر 150mA+. لو الـ bit ده 0، الـ card بتشتغل بـ power saving mode بس performance أقل.

---

#### `SD_OCR_CCS` (Bit 30)

```c
#define SD_OCR_CCS      (1 << 30)   /* Card Capacity Status */
```

**أهم bit في الـ OCR**. بعد ما الـ card تخلص initialization:
- `CCS = 0` → SD Standard Capacity (SDSC) — addressing بـ bytes — max 2GB.
- `CCS = 1` → SDHC/SDXC/SDUC — addressing بـ blocks (512 bytes each).

الـ kernel بيقرأه في `mmc_sd_init_card()`:

```c
/* من drivers/mmc/core/sd.c */
if (rocr & SD_OCR_CCS)
    mmc_card_set_blockaddr(card); /* يضبط FLAG_BLOCKADDR */
```

---

### Group 5: SD_SWITCH Macros (Mode & Function Groups)

---

#### `SD_SWITCH_CHECK` / `SD_SWITCH_SET`

```c
#define SD_SWITCH_CHECK     0
#define SD_SWITCH_SET       1
```

CMD6 بيشتغل في خطوتين دايمًا:
1. **Check first** (`SD_SWITCH_CHECK`): الـ card بترد بـ 64-byte status توضح الـ functions المدعومة. الـ host بيتحقق إن الـ function اللي عايزه موجودة.
2. **Set if supported** (`SD_SWITCH_SET`): الـ host بيبعت CMD6 تاني بالـ Set mode عشان يفعّل الـ function.

```c
/* Pattern شائع في الكود */
/* Step 1: Check */
err = mmc_sd_switch(card, SD_SWITCH_CHECK, SD_SWITCH_GRP_ACCESS,
                    SD_SWITCH_ACCESS_HS, status);
if (status[13] & 0x02) { /* HS supported */
    /* Step 2: Set */
    err = mmc_sd_switch(card, SD_SWITCH_SET, SD_SWITCH_GRP_ACCESS,
                        SD_SWITCH_ACCESS_HS, status);
}
```

---

#### `SD_SWITCH_GRP_ACCESS` / `SD_SWITCH_ACCESS_DEF` / `SD_SWITCH_ACCESS_HS`

```c
#define SD_SWITCH_GRP_ACCESS    0   /* Function Group 1 */
#define SD_SWITCH_ACCESS_DEF    0   /* Default Speed (25 MHz) */
#define SD_SWITCH_ACCESS_HS     1   /* High Speed (50 MHz) */
```

الـ **Function Group 1** هو الـ Access Mode — بيتحكم في الـ clock speed:

| Value | Mode | Max Clock | Max Throughput |
|---|---|---|---|
| 0 (`ACCESS_DEF`) | Default Speed | 25 MHz | 12.5 MB/s |
| 1 (`ACCESS_HS`) | High Speed | 50 MHz | 25 MB/s |
| 2 | SDR25 (UHS-I) | 50 MHz | 25 MB/s |
| 3 | SDR50 (UHS-I) | 100 MHz | 50 MB/s |
| 4 | SDR104 (UHS-I) | 208 MHz | 104 MB/s |
| 5 | DDR50 (UHS-I) | 50 MHz DDR | 50 MB/s |

الـ values 2-5 محددة في `include/linux/mmc/sd.h` الأوسع أو في `drivers/mmc/core/sd.c` مباشرة كـ literals.

---

### Group 6: Erase & Discard Arguments

```c
#define SD_ERASE_ARG       0x00000000
#define SD_DISCARD_ARG     0x00000001
```

الفرق الجوهري:

| | `SD_ERASE_ARG` | `SD_DISCARD_ARG` |
|---|---|---|
| Data after op | Undefined (usually 0xFF or 0x00) | May be preserved |
| Card action | Physical erase | Card decides |
| Speed | Slower (physical) | Faster (logical mark) |
| Use case | Secure wipe | Normal deletion |

الـ **Discard** مشابه للـ TRIM في SATA/NVMe — الـ flash translation layer في الـ card بتاخد قرارها متى تعمل الـ physical erase. ده بيحسّن الـ write performance لأن الـ card ممكن تعمل الـ erase في background.

في الكود، الـ `mmc_erase()` في `drivers/mmc/core/core.c` بتستخدم الـ arguments دي:

```c
/* من drivers/mmc/core/core.c — مبسّط */
int mmc_erase(struct mmc_card *card, unsigned int from,
              unsigned int nr, unsigned int arg)
{
    /* CMD32: Set erase start */
    mmc_send_cmd(host, SD_ERASE_WR_BLK_START, from, ...);

    /* CMD33: Set erase end */
    mmc_send_cmd(host, SD_ERASE_WR_BLK_END, from + nr - 1, ...);

    /* CMD38: Execute erase with arg = SD_ERASE_ARG or SD_DISCARD_ARG */
    mmc_send_cmd(host, MMC_ERASE, arg, ...);
}
```

---

### Initialization Flow — كيف بيتعاملوا مع بعض

```
Host Powers On Card
        │
        ▼
CMD0 (GO_IDLE_STATE)
        │
        ▼
CMD8 (SD_SEND_IF_COND)
  ├── No response → SD 1.x (SDSC only)
  └── Response OK → SD 2.0+ capable
        │
        ▼
ACMD41 (SD_APP_OP_COND) loop
  ├── arg[30] HCS=1 → Host supports SDHC/SDXC
  ├── arg[28] XPC → Host supports 150mA
  ├── arg[24] S18R → Host wants 1.8V
  └── Wait until card busy bit clears
        │
        ├── rocr[30] CCS=1 → SDHC/SDXC (block addressing)
        └── rocr[24] S18A=1 → Proceed to voltage switch
               │
               ▼
        CMD11 (SD_SWITCH_VOLTAGE)
               │
               ▼
        CMD2 (ALL_SEND_CID)
               │
               ▼
        CMD3 (SD_SEND_RELATIVE_ADDR) → Get RCA
               │
               ▼
        CMD9 (SEND_CSD)
               │
               ▼
        CMD7 (SELECT_CARD) → Transition to Transfer state
               │
               ▼
        ACMD51 (SD_APP_SEND_SCR) → Read capabilities
               │
               ▼
        CMD6 (SD_SWITCH) Check HS
               │
               ├── SD_SWITCH_ACCESS_HS supported?
               │     └── CMD6 Set HS → Switch to 50MHz
               │
               ▼
        ACMD6 (SD_APP_SET_BUS_WIDTH)
               └── SD_BUS_WIDTH_4 → Switch to 4-bit bus
```

---

### SCR Spec Version Macros في السياق

```c
/* من drivers/mmc/core/sd.c */
static int mmc_decode_scr(struct mmc_card *card)
{
    struct sd_scr *scr = &card->scr;

    /* الـ SD_SPEC field في الـ SCR */
    scr->sda_vsn = (resp[0] >> 24) & 0xf;

    if (scr->sda_vsn == SCR_SPEC_VER_2) {
        /* Check SD_SPEC3 bit for v3.0+ */
        scr->sda_vsn = ((resp[0] >> 15) & 0x1) ?
                       3 : SCR_SPEC_VER_2;
    }

    /* CMD23 support — SET_BLOCK_COUNT */
    if ((resp[0] >> 0) & 0x2)
        scr->cmds |= SD_SCR_CMD23_SUPPORT;
}
```

الـ `SCR_SPEC_VER_0/1/2` بيتستخدموا كـ comparison values عشان الـ driver يعرف مستوى الـ spec اللي بتدعمه الـ card وبناءً عليه يفعّل/يوقف features.
## Phase 5: دليل الـ Debugging الشامل

الـ subsystem ده هو **SD card protocol layer** في kernel — بيتحكم في الـ commands والـ OCR bits والـ SCR/SSR registers والـ bus speed switching (CMD6) اللي بتتعامل معاها الـ `sd.h` headers.

---

### Software Level

#### 1. debugfs Entries

الـ MMC/SD subsystem بيعمل entries جوه `/sys/kernel/debug/mmc*/` لكل controller.

```bash
# اعرض كل الـ debugfs entries المتاحة للـ MMC controllers
ls /sys/kernel/debug/mmc*/

# الـ entry الأهم: ios — بيعرض الـ I/O settings الحالية
cat /sys/kernel/debug/mmc0/ios
```

**مثال على output الـ ios:**

```
clock:          50000000 Hz
actual clock:   50000000 Hz
vdd:            21 (3.3 ~ 3.4 V)
bus mode:       2 (push-pull)
chip select:    0 (don't care)
power mode:     2 (on)
bus width:      2 (4 bits)
timing spec:    2 (sd high-speed)
signal voltage: 0 (3.30 V)
driver type:    0 (driver type B)
```

**التفسير:**
- `timing spec: 2` = الكارت شغال بـ High-Speed (50 MHz)
- `signal voltage: 0` = 3.3V → لو بتحاول تعمل UHS ولم تنزل لـ 1.8V، المشكلة في الـ `SD_SWITCH_VOLTAGE` (CMD11)
- `bus width: 2` = 4-bit — يتطابق مع `SD_BUS_WIDTH_4 = 2` في `sd.h`

```bash
# اقرأ الـ SCR register (SD Configuration Register)
cat /sys/kernel/debug/mmc0/scr
# output: مثلاً 02b54000 00000000
# البايت الأول [63:56] = SCR_STRUCTURE, البايت [59:56] = SD_SPEC version

# اقرأ الـ SSR (SD Status Register)
cat /sys/kernel/debug/mmc0/ssr
```

#### 2. sysfs Entries

**الـ path الأساسي:** `/sys/bus/mmc/devices/mmc0:*/`

```bash
# اعرض كل attributes الكارت
ls /sys/bus/mmc/devices/mmc0:*/

# معلومات الـ CID (Card Identification)
cat /sys/bus/mmc/devices/mmc0:*/name        # product name من CID.prod_name
cat /sys/bus/mmc/devices/mmc0:*/manfid      # manufacturer ID من CID.manfid
cat /sys/bus/mmc/devices/mmc0:*/serial      # serial number
cat /sys/bus/mmc/devices/mmc0:*/oemid       # OEM ID

# الـ CSD info
cat /sys/bus/mmc/devices/mmc0:*/csd         # raw CSD register

# الـ speed class
cat /sys/bus/mmc/devices/mmc0:*/speed_class
cat /sys/bus/mmc/devices/mmc0:*/uhs_speed_grade

# preferred erase size
cat /sys/bus/mmc/devices/mmc0:*/preferred_erase_size

# هل الكارت SDUC (SD Ultra Capacity) — بيتحقق من SD_OCR_2T
cat /sys/bus/mmc/devices/mmc0:*/ocr
```

**جدول الـ sysfs attributes المهمة:**

| Attribute | المصدر في الكود | المعنى |
|---|---|---|
| `name` | `mmc_card.cid.prod_name` | اسم المنتج |
| `manfid` | `mmc_card.cid.manfid` | manufacturer ID |
| `oemid` | `mmc_card.cid.oemid` | OEM ID |
| `serial` | `mmc_card.cid.serial` | رقم سيريال |
| `csd` | `mmc_card.raw_csd` | raw CSD |
| `scr` | `mmc_card.raw_scr` | raw SCR |
| `ocr` | `mmc_card.ocr` | OCR bits incl. `SD_OCR_CCS`, `SD_OCR_S18R` |
| `preferred_erase_size` | `mmc_card.pref_erase` | حجم الـ erase المفضل |
| `speed_class` | `mmc_card.ssr.au` | speed class من SSR |

#### 3. ftrace — Tracepoints والـ Events

```bash
# اعرض كل الـ tracepoints المتعلقة بالـ MMC
ls /sys/kernel/debug/tracing/events/mmc/

# الـ events الأهم للـ SD protocol debugging
echo 1 > /sys/kernel/debug/tracing/events/mmc/mmc_request_start/enable
echo 1 > /sys/kernel/debug/tracing/events/mmc/mmc_request_done/enable
echo 1 > /sys/kernel/debug/tracing/events/mmc/mmc_cmd_rw_end/enable

# تفعيل كل الـ MMC events دفعة واحدة
echo 1 > /sys/kernel/debug/tracing/events/mmc/enable

# ابدأ التتبع
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... اعمل العملية اللي بتـ debug فيها ...
cat /sys/kernel/debug/tracing/trace
```

**مثال على output:**

```
kworker/0:2-45    [000] ....  123.456789: mmc_request_start: mmc0: start struct mmc_request[0x...]:
    cmd_opcode=8 cmd_arg=0x000001aa cmd_flags=0x75 cmd_retries=0
    stop=NULL data=NULL
```

- **`cmd_opcode=8`** = `SD_SEND_IF_COND` — بيتحقق من الـ voltage range
- **`cmd_arg=0x000001aa`** = `[11:8]=0x1` (2.7-3.6V) + `[7:0]=0xAA` (check pattern)

```bash
# trace function calls في الـ SD driver
echo 'sd_*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

#### 4. printk والـ Dynamic Debug

```bash
# تفعيل الـ dynamic debug للـ MMC/SD subsystem بالكامل
echo 'module mmc_core +p' > /sys/kernel/debug/dynamic_debug/control
echo 'module sdhci +p' > /sys/kernel/debug/dynamic_debug/control
echo 'module sdhci_pci +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug لملف بعينه (مثلاً sd.c)
echo 'file drivers/mmc/core/sd.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل كل الـ debug messages في الـ MMC directory
echo 'file drivers/mmc/* +p' > /sys/kernel/debug/dynamic_debug/control

# عرض الـ control entries الحالية
cat /sys/kernel/debug/dynamic_debug/control | grep mmc

# تتبع من الـ boot (kernel command line)
# أضف في /etc/default/grub:
# GRUB_CMDLINE_LINUX="dyndbg='module mmc_core +p; module sdhci +p'"
```

**الـ log level المناسب:**

```bash
# رفع الـ log level مؤقتاً
echo 8 > /proc/sys/kernel/printk

# متابعة الـ kernel messages في real-time
dmesg -w | grep -E 'mmc|sdhci|sd[0-9]'
```

#### 5. Kernel Config Options للـ Debugging

```bash
# اعرض الـ options المفعّلة في الـ kernel الحالي
zcat /proc/config.gz | grep -E 'MMC.*DEBUG|DEBUG.*MMC'
```

| Config Option | الوظيفة |
|---|---|
| `CONFIG_MMC_DEBUG` | يفعّل الـ `pr_debug()` calls في الـ MMC core |
| `CONFIG_MMC_UNSAFE_RESUME` | تشخيص مشاكل الـ resume |
| `CONFIG_SDHCI_DEBUG_LOG` | سجل تفصيلي للـ SDHCI controller |
| `CONFIG_MMC_SDHCI_IPROC_DEBUG` | debug لـ Broadcom IPROC |
| `CONFIG_DEBUG_FS` | لازم يكون enabled عشان debugfs تشتغل |
| `CONFIG_DYNAMIC_DEBUG` | لازم عشان dynamic debug يشتغل |
| `CONFIG_TRACING` | لازم عشان ftrace يشتغل |
| `CONFIG_MMC_BLOCK_MINORS` | التحكم في عدد الـ minor devices |

```bash
# build kernel مع MMC debug
make menuconfig
# Device Drivers → MMC/SD/SDIO card support → MMC debugging
```

#### 6. أدوات خاصة بالـ MMC/SD Subsystem

```bash
# mmc-utils — الأداة الرسمية لتشخيص MMC/SD
apt install mmc-utils   # أو: yum install mmc-utils

# اعرض معلومات الكارت (يقرأ SCR, CSD, CID من sysfs)
mmc info /dev/mmcblk0

# اقرأ extended CSD (للـ eMMC فقط)
mmc extcsd read /dev/mmcblk0

# اختبار الـ bus speed modes
mmc setbuswidth /dev/mmcblk0 4   # 4-bit bus width

# التحقق من الـ erase group size
mmc erasesize /dev/mmcblk0

# hdparm للـ performance measurement
hdparm -tT /dev/mmcblk0

# iostat لمراقبة الـ I/O
iostat -x 1 10 mmcblk0
```

**الـ udev rules لمتابعة الـ events:**

```bash
# مراقبة الـ SD card insert/remove events
udevadm monitor --subsystem-match=mmc
```

#### 7. جدول أخطاء شائعة → المعنى → الحل

| رسالة الخطأ | المعنى | الحل |
|---|---|---|
| `mmc0: error -110 whilst initialising SD card` | timeout أثناء الـ init — CMD8 أو ACMD41 لم يرد | تحقق من power supply، جرب `SD_SEND_IF_COND` يدوياً |
| `mmc0: ACMD41 timeout, voltage mismatch or card is not SD` | الكارت مش بيرد على `SD_APP_OP_COND (ACMD41)` | الكارت تالف أو OCR mismatch، تحقق من `SD_OCR_CCS` |
| `mmc0: CMD6 failed` | فشل `SD_SWITCH` — الكارت مش بيقبل الـ speed switch | الكارت مش بيدعم High-Speed، تحقق من `sd_switch_caps.hs_max_dtr` |
| `mmc0: error -84 whilst initialising SD card` | CRC error في الـ response | مشكلة في الـ signal integrity، قصّر الكابل |
| `mmc0: tuning execution failed` | فشل الـ tuning لـ UHS-I/HS200 | مشكلة في الـ clock signal أو signal voltage (1.8V) |
| `mmc0: Switching to 1.8V signalling voltage failed` | `SD_SWITCH_VOLTAGE (CMD11)` فشل | الـ regulator مش بيقدر ينزل لـ 1.8V، تحقق من DT regulator config |
| `mmc0: card claims to support voltages below defined range` | OCR bits خاطئة | الكارت فيه quirk، ضيف `MMC_QUIRK_*` مناسب |
| `mmc0: Got data CRC error on read` | CRC error في الـ data | مشكلة في الـ bus integrity أو الكارت تالف |
| `mmc0: card is slow to respond, card removed?` | الكارت اتشال أو power issue | تحقق من الـ card detect pin وسلامة الاتصال |
| `mmc0: SD_SEND_IF_COND error -110` | CMD8 timeout — SD v1.0 card أو مش SD | يكون SD v1.0 (قبل spec 2.0)، الـ kernel بيـ fallback تلقائياً |
| `mmc0: SDHCI controller problem` | مشكلة في الـ host controller | تحقق من `SDHCI_INT_STATUS` register |
| `mmc0: ran out of retries making CMD` | الكارت مش بيرد بعد كذا retry | الكارت تالف أو power issue |
| `mmc0: discard error -110` | `SD_DISCARD_ARG (0x1)` في CMD32/33 فشل | ضيف `MMC_QUIRK_BROKEN_SD_DISCARD` للكارت |
| `mmc0: cache flush error` | فشل flush الـ cache | ضيف `MMC_QUIRK_BROKEN_CACHE_FLUSH` |

#### 8. نقاط استراتيجية لـ dump_stack() وWARN_ON()

**في `drivers/mmc/core/sd.c`:**

```c
/* نقطة 1: بعد فشل SD_SEND_IF_COND (CMD8) */
static int mmc_send_if_cond(struct mmc_host *host, u32 ocr)
{
    int err = mmc_wait_for_cmd(host, &cmd, 0);
    if (err) {
        /* هنا نعرف إن الكارت مش SD v2+ أو في مشكلة voltage */
        WARN_ON(err != -ETIMEDOUT); /* أي error غير timeout = غريب */
    }
}

/* نقطة 2: عند switch الـ voltage لـ 1.8V */
static int mmc_set_uhs_voltage(struct mmc_host *host, u32 ocr)
{
    /* بعد CMD11 = SD_SWITCH_VOLTAGE */
    if (!(ocr & SD_OCR_S18R)) {
        /* الكارت مش بيدعم 1.8V، دي مش مشكلة */
    } else {
        WARN_ON(!host->ops->start_signal_voltage_switch);
    }
}

/* نقطة 3: عند parsing الـ SCR */
static int mmc_decode_scr(struct mmc_card *card)
{
    struct sd_scr *scr = &card->scr;
    /* تحقق من SCR spec version */
    WARN_ON(scr->sda_vsn > SCR_SPEC_VER_2 + 10); /* version عالي جداً = corrupt */
}

/* نقطة 4: عند فشل SD_SWITCH (CMD6) للـ bus speed */
static int mmc_sd_switch(struct mmc_card *card, int mode, int group, u8 value)
{
    /* mode = SD_SWITCH_CHECK أو SD_SWITCH_SET */
    /* group = SD_SWITCH_GRP_ACCESS = 0 */
    /* value = SD_SWITCH_ACCESS_HS = 1 */
    if (err) {
        dump_stack(); /* عشان نشوف من فين اتنادى الـ switch */
    }
}
```

---

### Hardware Level

#### 1. التحقق من مطابقة Hardware State مع Kernel State

```bash
# الـ kernel يعتقد إن الكارت شغال بـ 4-bit bus width؟
cat /sys/kernel/debug/mmc0/ios | grep 'bus width'
# المفروض يطابق الـ hardware wiring

# تحقق من الـ clock speed المطلوب vs المحقق
cat /sys/kernel/debug/mmc0/ios
# "clock:" = المطلوب من الـ software
# "actual clock:" = اللي الـ controller فعلاً بيولده

# تحقق من الـ signal voltage
cat /sys/kernel/debug/mmc0/ios | grep 'signal voltage'
# 0 = 3.3V, 1 = 1.8V, 2 = 1.2V
# لازم يطابق اللي الـ regulator بيوليه فعلاً بالـ multimeter

# قارن OCR من الـ kernel مع الـ hardware spec
cat /sys/bus/mmc/devices/mmc0:*/ocr
# bit 24 = SD_OCR_S18R (1.8V switching accepted)
# bit 30 = SD_OCR_CCS (SDHC/SDXC card)
# bit 27 = SD_OCR_2T (SDUC support)
```

#### 2. Register Dump Techniques

**تحذير:** الـ `/dev/mem` access محتاج `CONFIG_DEVMEM=y` و `iomem=relaxed` في kernel cmdline.

```bash
# تثبيت devmem2
apt install devmem2   # أو build من source

# اقرأ SDHCI registers (مثال: SDHCI على عنوان 0xFE300000)
# لازم تعرف الـ base address من DT أو /proc/iomem
cat /proc/iomem | grep -i sdhci
cat /proc/iomem | grep -i mmc

# مثال: قراءة SDHCI_PRESENT_STATE register (offset 0x24)
devmem2 0xFE300024 w
# bit 16 = card inserted
# bit 17 = card stable
# bit 18 = card detect pin level

# قراءة SDHCI_CAPABILITIES (offset 0x40)
devmem2 0xFE300040 l
# bits [52:48] = max clock freq
# bit 37 = 1.8V support
# bit 36 = 3.0V support
# bit 35 = 3.3V support

# SDHCI_CLOCK_CONTROL (offset 0x2C)
devmem2 0xFE30002C w
# bits [15:8] = SDCLK frequency select
# bit 2 = SD Clock Enable
# bit 0 = Internal Clock Enable
```

**الـ SDHCI Standard Register Map (للـ reference):**

```
Offset  Register
0x00    SDHCI_DMA_ADDRESS
0x04    SDHCI_BLOCK_SIZE / SDHCI_BLOCK_COUNT
0x08    SDHCI_ARGUMENT
0x0C    SDHCI_TRANSFER_MODE / SDHCI_COMMAND
0x10    SDHCI_RESPONSE (16 bytes)
0x20    SDHCI_BUFFER
0x24    SDHCI_PRESENT_STATE        ← الأهم للـ debug
0x28    SDHCI_HOST_CONTROL
0x29    SDHCI_POWER_CONTROL
0x2A    SDHCI_BLOCK_GAP_CONTROL
0x2B    SDHCI_WAKE_UP_CONTROL
0x2C    SDHCI_CLOCK_CONTROL
0x2E    SDHCI_TIMEOUT_CONTROL
0x2F    SDHCI_SOFTWARE_RESET
0x30    SDHCI_INT_STATUS           ← تشوف منه الـ errors
0x34    SDHCI_INT_ENABLE
0x38    SDHCI_SIGNAL_ENABLE
0x3C    SDHCI_ACMD12_ERR (SDHCI v2) / SDHCI_HOST_CONTROL2 (v3)
0x40    SDHCI_CAPABILITIES
0x48    SDHCI_CAPABILITIES_1
```

```bash
# script شامل لقراءة أهم SDHCI registers
BASE=0xFE300000  # غيّر حسب الـ hardware

echo "=== SDHCI Register Dump ==="
echo "PRESENT_STATE:  $(devmem2 $((BASE+0x24)) w 2>/dev/null | awk '/Read/{print $NF}')"
echo "HOST_CONTROL:   $(devmem2 $((BASE+0x28)) b 2>/dev/null | awk '/Read/{print $NF}')"
echo "POWER_CONTROL:  $(devmem2 $((BASE+0x29)) b 2>/dev/null | awk '/Read/{print $NF}')"
echo "CLOCK_CONTROL:  $(devmem2 $((BASE+0x2C)) w 2>/dev/null | awk '/Read/{print $NF}')"
echo "INT_STATUS:     $(devmem2 $((BASE+0x30)) l 2>/dev/null | awk '/Read/{print $NF}')"
echo "CAPABILITIES:   $(devmem2 $((BASE+0x40)) l 2>/dev/null | awk '/Read/{print $NF}')"
```

#### 3. Logic Analyzer / Oscilloscope Tips

**الـ SD bus pins للقياس:**

```
SD Card Pinout (Standard SD):
Pin 1: CD/DAT3   ← data أو card detect
Pin 2: CMD       ← الـ command line - الأهم
Pin 3: VSS1      ← GND
Pin 4: VDD       ← قس الـ voltage هنا (3.3V أو 1.8V)
Pin 5: CLK       ← الـ clock - قيس التردد والـ jitter
Pin 6: VSS2      ← GND
Pin 7: DAT0      ← data
Pin 8: DAT1      ← data
Pin 9: DAT2      ← data
```

**نقاط القياس بالـ oscilloscope:**

```
1. CLK (Pin 5):
   - Default speed: 400 kHz (identification phase)
   - High-speed: 50 MHz
   - UHS-I SDR104: 208 MHz
   - قس الـ rise/fall time: لازم < 1ns لـ SDR104
   - قس الـ jitter: لازم < 0.5% من الـ period

2. CMD (Pin 2):
   - بعد كل command، انتظر الـ response
   - CMD8 (SD_SEND_IF_COND): يبعت ويستقبل R7 response
   - CMD11 (SD_SWITCH_VOLTAGE): بعد response، الـ CLK وقف والـ CMD/DAT يطلع high
   - لو مفيش response في 64 clock cycles = timeout

3. VDD (Pin 4):
   - أثناء SD_SWITCH_VOLTAGE: VDD لازم يقع من 3.3V لـ 1.8V خلال < 150ms
   - لو VDD مش بينزل → CMD11 هيفشل
```

**Logic Analyzer Protocol Decoder:**

```
- استخدم Sigrok/PulseView مع SD card decoder
- Set sample rate >= 10x الـ bus speed (مثلاً 500MHz sample لـ 50MHz bus)
- فعّل الـ "SD/MMC" decoder
- راقب:
  * الـ command index (6 bits)
  * الـ argument (32 bits)
  * الـ CRC (7 bits)
  * الـ response type (R1/R3/R6/R7)
```

#### 4. Hardware Issues الشائعة وأنماط الـ Kernel Log

| المشكلة الهاردوير | الكارنيل لوج | التحليل |
|---|---|---|
| **voltage regulator بطيء** | `mmc0: Timeout waiting for hardware interrupt` بعد CMD11 | الـ regulator بياخد وقت أطول من 150ms للتبديل لـ 1.8V |
| **clock signal noise** | `mmc0: Got data CRC error` or `mmc0: CMD CRC error` | قس الـ CLK بالـ scope، ابحث عن ringing |
| **سلك CMD طويل/ضعيف** | `mmc0: error -84` (EILSEQ) | الـ command CRC فشل، الـ impedance غلط |
| **power supply ripple** | `mmc0: card is slow to respond` متكرر | قس VDD بالـ scope، ابحث عن noise على الـ power rail |
| **SD socket oxidation** | `mmc0: error -110` (ETIMEDOUT) عند أي command | نظّف الـ contacts، جرب كارت تاني |
| **PCB trace capacitance** | tuning يفشل على بعض frequencies | الـ traces طويلة جداً → قصّر الـ PCB traces |
| **missing pull-up resistors** | `mmc0: ACMD41 no response` | CMD وDAT3 محتاجين pull-up 47kΩ |
| **card detect pin floating** | الـ kernel مش بيشوف الـ card | CD pin محتاج pull-up + debounce |

#### 5. Device Tree Debugging

**تحقق من مطابقة الـ DT مع الـ Hardware:**

```bash
# اعرض الـ DT node الخاص بالـ MMC controller
cat /proc/device-tree/mmc*/compatible
cat /proc/device-tree/mmc*/clock-frequency
cat /proc/device-tree/mmc*/bus-width

# أو باستخدام dtc لـ decompile
dtc -I fs /proc/device-tree/ 2>/dev/null | grep -A 30 'mmc@'

# تحقق من الـ regulator المربوط بالـ SD slot
cat /proc/device-tree/mmc*/vmmc-supply
cat /proc/device-tree/mmc*/vqmmc-supply  # 1.8V supply for UHS

# تحقق من الـ pinctrl settings
cat /proc/device-tree/mmc*/pinctrl-names
```

**الـ DT properties المهمة للـ SD (مرتبطة بـ sd.h constants):**

```dts
/* مثال DT node صحيح لـ SD slot يدعم UHS-I */
sdhci0: mmc@fe300000 {
    compatible = "brcm,bcm2835-sdhci";
    reg = <0xfe300000 0x100>;

    /* bus-width يتوافق مع SD_BUS_WIDTH_4 = 2 في sd.h */
    bus-width = <4>;

    /* الـ speeds المدعومة — بيتحكم في SD_SWITCH modes */
    cap-sd-highspeed;     /* SD_SWITCH_ACCESS_HS = 1 */
    sd-uhs-sdr50;         /* UHS_SDR50_BUS_SPEED = 2 */
    sd-uhs-sdr104;        /* UHS_SDR104_BUS_SPEED = 3 */
    sd-uhs-ddr50;         /* UHS_DDR50_BUS_SPEED = 4 */

    /* voltage switching لـ SD_SWITCH_VOLTAGE (CMD11) */
    vmmc-supply = <&reg_vmmc>;       /* 3.3V supply */
    vqmmc-supply = <&reg_vqmmc>;     /* 1.8V supply للـ UHS */

    /* الـ CD pin */
    cd-gpios = <&gpio 22 GPIO_ACTIVE_LOW>;
};
```

```bash
# تحقق من الـ regulator voltage ranges
cat /sys/class/regulator/regulator.*/name
cat /sys/class/regulator/regulator.*/min_microvolts
cat /sys/class/regulator/regulator.*/max_microvolts

# لو الـ vqmmc مش موجود، الـ 1.8V switching هيفشل
# والـ kernel هيلوج: "mmc0: Switching to 1.8V signalling voltage failed"

# تحقق من الـ gpio للـ card detect
gpioinfo | grep -A5 'card'
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للنسخ

**1. فحص شامل سريع للـ SD card:**

```bash
#!/bin/bash
# sd_debug_check.sh — فحص شامل للـ SD card

MMC_DEV=${1:-mmc0}
CARD_PATH="/sys/bus/mmc/devices/${MMC_DEV}:*"

echo "=== SD Card Debug Report ==="
echo "--- IOS State ---"
cat /sys/kernel/debug/${MMC_DEV}/ios

echo ""
echo "--- Card Info (CID) ---"
echo "Name:     $(cat ${CARD_PATH}/name 2>/dev/null)"
echo "Manf ID:  $(cat ${CARD_PATH}/manfid 2>/dev/null)"
echo "OEM ID:   $(cat ${CARD_PATH}/oemid 2>/dev/null)"
echo "Serial:   $(cat ${CARD_PATH}/serial 2>/dev/null)"
echo "Date:     $(cat ${CARD_PATH}/date 2>/dev/null)"

echo ""
echo "--- Capabilities ---"
echo "OCR:      $(cat ${CARD_PATH}/ocr 2>/dev/null)"
echo "SCR:      $(cat ${CARD_PATH}/scr 2>/dev/null)"
echo "Speed:    $(cat ${CARD_PATH}/speed_class 2>/dev/null)"

echo ""
echo "--- Recent Kernel Messages ---"
dmesg | grep -E "${MMC_DEV}|sdhci" | tail -20
```

**2. تفعيل logging كامل للـ SD init:**

```bash
# قبل ما تحشر الكارت
echo 8 > /proc/sys/kernel/printk
echo 'module mmc_core +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 'module sdhci +pflmt' > /sys/kernel/debug/dynamic_debug/control

# حشر الكارت، أو trigger re-init
echo 1 > /sys/bus/mmc/devices/mmc0/remove 2>/dev/null
echo 1 > /sys/bus/mmc/rescan

# اجمع اللوج
dmesg | grep -E 'mmc0|sdhci' > /tmp/sd_init_log.txt
cat /tmp/sd_init_log.txt
```

**3. اختبار الـ CMD6 (SD_SWITCH) يدوياً بـ mmc-utils:**

```bash
# اقرأ الـ switch capabilities (check mode = SD_SWITCH_CHECK = 0)
mmc switch /dev/mmcblk0

# اعرض bus speed mode الحالي
mmc extcsd read /dev/mmcblk0 | grep -E 'BUS_WIDTH|HS_TIMING|CARD_TYPE'
```

**4. تشخيص مشكلة الـ OCR:**

```bash
# اعرض الـ OCR raw value
OCR=$(cat /sys/bus/mmc/devices/mmc0:*/ocr)
echo "OCR: $OCR"

# تفسير الـ OCR bits (من sd.h)
python3 -c "
ocr = int('$OCR', 16)
print(f'OCR = 0x{ocr:08X}')
print(f'  SD_OCR_S18R (1.8V switch): {bool(ocr & (1<<24))}')   # SD_OCR_S18R
print(f'  SD_OCR_2T   (SDUC):        {bool(ocr & (1<<27))}')   # SD_OCR_2T
print(f'  SD_OCR_XPC  (SDXC power):  {bool(ocr & (1<<28))}')   # SD_OCR_XPC
print(f'  SD_OCR_CCS  (SDHC/SDXC):  {bool(ocr & (1<<30))}')   # SD_OCR_CCS
print(f'  Card busy:                 {bool(ocr & (1<<31))}')
"
```

**5. تفسير الـ SCR register:**

```bash
SCR=$(cat /sys/bus/mmc/devices/mmc0:*/scr 2>/dev/null || cat /sys/kernel/debug/mmc0/scr)
echo "Raw SCR: $SCR"

python3 -c "
import sys
scr_str = '$SCR'.replace(' ', '')
scr = int(scr_str, 16)
scr_hi = (scr >> 32) & 0xFFFFFFFF

struct_ver = (scr_hi >> 28) & 0xF
sd_spec    = (scr_hi >> 24) & 0xF
bus_widths = (scr_hi >> 16) & 0xF
sd_spec3   = (scr_hi >> 15) & 0x1
sd_spec4   = (scr_hi >> 10) & 0x1

print(f'SCR Structure version: {struct_ver}')
print(f'SD Spec version:       {sd_spec}  (0=1.0-1.01, 1=1.10, 2=2.0-3.x)')
print(f'SD Spec 3.0:           {bool(sd_spec3)}')
print(f'SD Spec 4.0:           {bool(sd_spec4)}')
print(f'Bus widths supported:')
print(f'  1-bit (SD_SCR_BUS_WIDTH_1): {bool(bus_widths & 0x1)}')
print(f'  4-bit (SD_SCR_BUS_WIDTH_4): {bool(bus_widths & 0x4)}')
"
```

**6. مراقبة الـ performance وتحليل الـ bottleneck:**

```bash
# قياس throughput
dd if=/dev/mmcblk0 of=/dev/null bs=4M count=256 iflag=direct
# إذا كان أبطأ من المتوقع → check bus speed mode

# فحص الـ I/O errors
cat /sys/block/mmcblk0/stat
# Column 1: reads completed, Col 9: I/Os in progress, Col 10: time in I/O ms

# مراقبة latency
ioping -c 20 /dev/mmcblk0

# تحقق من الـ queue depth
cat /sys/block/mmcblk0/queue/nr_requests
```

**7. Erase/Discard debugging:**

```bash
# تحقق من دعم الـ discard (SD_DISCARD_ARG = 0x1 في sd.h)
cat /sys/block/mmcblk0/queue/discard_granularity
cat /sys/block/mmcblk0/queue/discard_max_bytes

# لو discard_max_bytes = 0 → الكارت أو الـ driver مش بيدعم SD_DISCARD_ARG
# وممكن يكون بسبب MMC_QUIRK_BROKEN_SD_DISCARD

# اختبر الـ discard يدوياً (بحذر!)
blkdiscard -o 0 -l $((64*1024*1024)) /dev/mmcblk0p1

# راقب الـ kernel log
dmesg | grep -i discard
```

**8. متابعة الـ card detection events:**

```bash
# مراقبة realtime لـ hotplug events
udevadm monitor --subsystem-match=mmc --property

# مثال output عند حشر كارت:
# KERNEL[123.456] add      /devices/platform/sdhci.../mmc_host/mmc0/mmc0:1234 (mmc)
# ACTION=add
# DEVPATH=...
# SUBSYSTEM=mmc
# MMC_TYPE=SD
# MMC_NAME=SD32G   ← prod_name من CID
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: بطاقة SD مش بتشتغل على Android TV Box بـ Allwinner H616

#### العنوان
**SD card تفشل في الـ initialization على TV box بسبب مشكلة في `SD_SEND_IF_COND`**

#### السياق
شغال على منتج Android TV box مبني على **Allwinner H616** SoC. المنتج بيقبل بطاقات microSD لتوسعة الـ storage. العميل بيشتكي إن بطاقات **SDXC** معينة بتفشل في الـ detection بينما بطاقات SDHC القديمة بتشتغل تمام.

#### المشكلة
الـ kernel log بيظهر:
```
mmc0: error -110 whilst initialising SD card
mmc0: CMD8 timeout
```
الـ CMD8 هو `SD_SEND_IF_COND` — البطاقات الحديثة (SD 2.0+) بتحتاجه للـ negotiation.

#### التحليل
بنرجع لـ `sd.h`:

```c
#define SD_SEND_IF_COND  8   /* bcr  [11:0] See below   R7  */
```

الـ argument format الموثق في نفس الملف:
```
[31:12] Reserved (0)
[11:8]  Host Voltage Supply Flags
[7:0]   Check Pattern (0xAA)
```

الـ driver بيبعت CMD8 بـ argument `0x000001AA`:
- `[11:8] = 0x1` → 2.7–3.6V range
- `[7:0]  = 0xAA` → check pattern

لو البطاقة ما ردتش بنفس الـ pattern، الـ host بيعتبرها SD v1.x. البطاقات الحديثة SDXC بتستنى CMD8 صح عشان تدخل SD 3.0 mode. المشكلة: الـ H616 SDHC controller عنده **timing issue** في الـ CMD8 response window — الـ timeout value في الـ DT منخفض.

#### الحل
نفحص الـ DT node:
```dts
/* before */
&mmc2 {
    max-frequency = <150000000>;
    /* missing: no bus-width or vmmc config */
};
```

نصلح:
```dts
&mmc2 {
    vmmc-supply = <&reg_dldo1>;
    vqmmc-supply = <&reg_aldo1>;
    bus-width = <4>;
    max-frequency = <50000000>;  /* lower freq during bring-up */
    cd-gpios = <&pio 5 6 GPIO_ACTIVE_LOW>;
    data-inverted;
};
```

وكمان نتأكد إن الـ host driver بيبعت CMD8 صح:
```bash
# trace الـ MMC commands
echo 1 > /sys/kernel/debug/mmc0/ios
dmesg | grep -i "cmd8\|if_cond\|SD_SEND"
```

#### الدرس المستفاد
**الـ `SD_SEND_IF_COND` (CMD8)** هو الفاصل بين SD v1.x وSD v2.0+. أي مشكلة في الـ timing أو الـ voltage flags بتخلي الـ host يعتقد إن البطاقة قديمة وبيفوّت عليها الـ SDXC/UHS features.

---

### السيناريو 2: بطاقة SD بتيجي بـ 25MB/s بدل 104MB/s على Industrial Gateway بـ AM62x

#### العنوان
**فشل الـ `SD_SWITCH` لـ High-Speed mode على Texas Instruments AM62x**

#### السياق
Industrial IoT gateway بيشغل **Debian Linux** على **TI AM62x** SoC. الـ SD card المستخدمة هي SanDisk Industrial A1 Class 10. الـ datasheet بتقول المنتج المفروض يوصل **104MB/s (SDR104)**، لكن الـ benchmarks بتقف عند **25MB/s** فقط.

#### المشكلة
```bash
$ dd if=/dev/mmcblk0 of=/dev/null bs=4M count=100
100+0 records in → 26.2 MB/s
```

الكارت قادر على UHS-I SDR104، لكن الـ driver مش بيـswitch ليه.

#### التحليل
الـ speed switching بيمر بـ `SD_SWITCH` (CMD6):

```c
#define SD_SWITCH  6  /* adtc [31:0] See below   R1  */
```

الـ argument format:
```
[31]    Mode: SD_SWITCH_CHECK (0) or SD_SWITCH_SET (1)
[3:0]   Function group 1 → access mode
        SD_SWITCH_ACCESS_DEF (0) = default speed
        SD_SWITCH_ACCESS_HS  (1) = high speed
```

لكن UHS-I modes (SDR50, SDR104) بتحتاج أكتر من مجرد CMD6 — بتحتاج **voltage switching** من 3.3V لـ 1.8V عن طريق:

```c
#define SD_SWITCH_VOLTAGE  11  /* ac  R1 */
#define SD_OCR_S18R        (1 << 24)  /* 1.8V switching request */
#define SD_ROCR_S18A       SD_OCR_S18R
```

الـ sequence المفروض تكون:
1. `SD_APP_OP_COND` (ACMD41) مع `SD_OCR_S18R` set
2. لو الكارت رد بـ `SD_ROCR_S18A` → الكارت قبل الـ 1.8V
3. `SD_SWITCH_VOLTAGE` (CMD11) لتحويل الـ signaling
4. بعدين CMD6 بـ `SD_SWITCH_SET` + SDR104 function

الفحص في الـ DT:
```bash
cat /proc/device-tree/mmc@fa10000/no-1-8-v
# إذا موجود → هو المشكلة
```

#### الحل
نشيل `no-1-8-v` من الـ DT وبنتأكد إن الـ LDO بيدعم 1.8V output:

```dts
/* AM62x SK board DT fix */
&sdhci1 {
    pinctrl-names = "default";
    pinctrl-0 = <&main_mmc1_pins_default>;
    ti,driver-strength-ohm = <50>;
    vmmc-supply = <&vdd_mmc1>;
    vqmmc-supply = <&vdd_mmc1_sd>;  /* must support 1.8V */
    /* remove: no-1-8-v */
};
```

```bash
# تحقق بعد الـ fix
cat /sys/kernel/debug/mmc1/ios | grep timing
# يجيب: timing spec: 6 (sd uhs sdr104)
```

#### الدرس المستفاد
الـ `SD_SWITCH_VOLTAGE` (CMD11) و `SD_OCR_S18R` في `sd.h` هم مفتاح UHS-I. لو الـ DT فيه `no-1-8-v` أو الـ PMIC مش مضبوط على 1.8V، الـ kernel بيبقى في default 3.3V signaling وبيفوّت كل الـ high-speed modes.

---

### السيناريو 3: data corruption عشوائي على IoT Sensor بـ STM32MP1

#### العنوان
**مشكلة الـ Erase/Discard — إساءة استخدام `SD_ERASE_ARG` و `SD_DISCARD_ARG`**

#### السياق
IoT edge device بيشغل **Buildroot Linux** على **STM32MP157** بيحفظ sensor readings على microSD. بعد 3 أشهر من الاستخدام، الـ filesystem بيبقى corrupt وبيضطر الفريق يعمل reflash كل شوية.

#### المشكلة
الفحص الأولي:
```bash
dmesg | grep -i "mmc\|mmcblk"
# mmcblk0: error -84 transferring data
# EXT4-fs error: IO failure
```

الـ `error -84` هو `EILSEQ` — invalid data sequence.

#### التحليل
نراجع الـ erase definitions في `sd.h`:

```c
#define SD_ERASE_WR_BLK_START  32  /* ac [31:0] data addr  R1 */
#define SD_ERASE_WR_BLK_END    33  /* ac [31:0] data addr  R1 */

#define SD_ERASE_ARG    0x00000000  /* standard erase */
#define SD_DISCARD_ARG  0x00000001  /* discard (hint, data may remain) */
```

الـ Discard مجرد **hint** — البطاقة ممكن تبقّي البيانات أو تمسحها. المشكلة إن الـ userspace application كانت بتعمل `BLKDISCARD` ioctl وبعدين تقرأ نفس الـ blocks متوقعةً zeros، وبتاخد garbage data.

كمان، الـ STM32MP1 SDMMC controller عنده **bug** في الـ erase timeout calculation للـ SD cards بـ large AU size، مما بيخلي الـ erase ينتهي قبل ما البطاقة تخلص.

الـ chain:
1. App بتعمل `BLKDISCARD`
2. Kernel بيبعت CMD32+CMD33+CMD38 بـ `SD_DISCARD_ARG`
3. الـ controller بيـtimeout مبكر
4. البطاقة لسه بتعمل internal erase
5. الـ next write بتحصل فوق incomplete operation → corruption

#### الحل
نفرض standard erase بدل discard في الـ driver:

```c
/* in drivers/mmc/core/sd.c - temporary workaround */
static unsigned int sd_erase_arg(struct mmc_card *card)
{
    /* STM32MP1 quirk: discard causes corruption with some SD cards */
    if (card->host->quirks & MMC_QUIRK_BROKEN_SD_DISCARD)
        return SD_ERASE_ARG;

    if (card->erase_arg == SD_DISCARD_ARG)
        return SD_DISCARD_ARG;

    return SD_ERASE_ARG;
}
```

وفي الـ DT نضيف الـ quirk flag:
```dts
&sdmmc1 {
    st,neg-edge;
    broken-sd-discard;  /* custom quirk property */
};
```

أو نعطل الـ discard في الـ filesystem:
```bash
# fstab
/dev/mmcblk0p2  /  ext4  defaults,nodiscard  0 1
```

#### الدرس المستفاد
الـ `SD_DISCARD_ARG` مش مضمون — البطاقة ممكن تتجاهله أو تنفذه جزئياً. في environments الـ industrial اللي فيها data integrity مهمة، استخدم `SD_ERASE_ARG` الحقيقي أو عطل الـ discard خالص.

---

### السيناريو 4: SD card بتتعرف كـ 2GB بدل 128GB على Custom Board بـ RK3562

#### العنوان
**فشل الـ SDUC detection بسبب `SD_OCR_2T` و `SD_OCR_CCS` غير مضبوطين**

#### السياق
فريق hardware يعمل **custom board bring-up** على **RK3562** لجهاز POS (point-of-sale). بيحاولوا يشغلوا **128GB SDUC card** (SD Ultra Capacity — أكبر من 2TB capacity spec). البطاقة بتتعرف، لكن الـ size بيظهر 2GB بس.

#### المشكلة
```bash
lsblk
# mmcblk0    8:0   0   2G  0 disk
# المفروض يكون 128G
```

#### التحليل
في `sd.h` عندنا:

```c
#define SD_OCR_2T   (1 << 27)  /* HO2T/CO2T - SDUC support */
#define SD_OCR_CCS  (1 << 30)  /* Card Capacity Status */
```

الـ OCR negotiation بيحصل في `SD_APP_OP_COND` (ACMD41). الـ host بيبعت OCR argument بيطلب الـ voltage range ويحدد الـ capacity bits اللي بيدعمها. لو الـ host ما بعتش `SD_OCR_2T` في الـ request، البطاقة بتفترض إن الـ host مش عارف يتعامل مع SDUC وبترد بـ SDHC-compatible OCR.

الـ sequence المشكلة:
```
Host → ACMD41 arg: 0x40FF8000
       bit 30 (CCS) = 1  ✓
       bit 27 (CO2T) = 0  ✗ ← المشكلة

Card → OCR response: reports as SDHC (2GB visible)
```

الكود في `drivers/mmc/core/sd.c`:
```c
/* host should include SD_OCR_2T if it supports SDUC */
ocr = SD_OCR_CCS;
if (mmc_host_uhs(host))
    ocr |= SD_OCR_S18R;
/* BUG: SD_OCR_2T (SDUC) never set for RK3562 host driver */
```

الـ RK3562 MMC controller driver (`drivers/mmc/host/dw_mmc-rockchip.c`) مش بيset الـ `MMC_CAP2_SD_SDUC` capability.

#### الحل
نضيف الـ capability في الـ Rockchip DW MMC driver:

```c
/* drivers/mmc/host/dw_mmc-rockchip.c */
static void dw_mci_rk3562_set_ios(struct dw_mci *host, struct mmc_ios *ios)
{
    /* existing code... */

    /* Enable SDUC support */
    host->pdata->caps2 |= MMC_CAP2_SD_SDUC;
}
```

أو مؤقتاً في الـ DT:
```dts
&sdmmc {
    cap-sd-highspeed;
    sd-uhs-sdr104;
    /* force SDUC capability */
    rockchip,default-sample-phase = <90>;
    /* add custom capability via module param */
};
```

نتحقق:
```bash
cat /sys/kernel/debug/mmc0/ios
# يجيب capacity: SDUC

# أو
dmesg | grep "mmc0: new ultra high speed"
```

#### الدرس المستفاد
الـ `SD_OCR_2T` في `sd.h` هو الـ flag المسؤول عن SDUC detection. لو الـ host driver مش بيعلن عن الـ capability دي، البطاقات فوق 2TB هتبان أصغر. دايماً راجع الـ `MMC_CAP2_*` flags في الـ host driver لما تشتغل على بطاقات SDXC/SDUC جديدة.

---

### السيناريو 5: USB-SD card reader بياخد عناوين غلط على Automotive ECU بـ i.MX8

#### العنوان
**الـ `SD_ADDR_EXT` (CMD22) والـ extended addressing في automotive storage**

#### السياق
Automotive ECU مبني على **NXP i.MX8QM** بيشغل Linux مع **AUTOSAR** middleware. الجهاز بيستخدم **SD Express card** لتخزين الـ OTA update images. بعد تحديث الـ kernel من 5.15 لـ 6.1، الـ large OTA files (> 32GB) بتتكسر أثناء الـ write.

#### المشكلة
```bash
dmesg
# mmcblk0: error -5 writing sectors 67108864..67109887
# blk_update_request: I/O error, dev mmcblk0, sector 0x4000000
```

الـ sector `0x4000000` = `67,108,864` — بالظبط الـ 32GB boundary.

#### التحليل
نرجع لـ `sd.h`:

```c
#define SD_ADDR_EXT  22  /* ac [5:0]  R1 */
```

الـ `SD_ADDR_EXT` هو CMD22 — بيُستخدم لتمديد الـ addressing لما الـ card address تتجاوز 32-bit range (standard SDXC limit = 2TB، لكن بعض الـ SD Express cards بتستخدم extended addressing لـ use cases مختلفة).

الـ [5:0] argument بيحمل الـ **upper address bits** اللي بتتحط قبل CMD17/18/24/25.

الـ sequence الصح:
```c
/* Before any read/write beyond 32-bit address space */
/* CMD22: set address extension */
sd_addr_ext_cmd(card, upper_bits);  /* sets [37:32] of address */

/* Then CMD18/25 with lower 32 bits */
mmc_read_write_cmd(card, lower_32bits);
```

في kernel 6.1، تغيّر الـ `mmc_blk_rw_rq_prep()` وكسر الـ sequence للـ SD Express cards. الـ CMD22 بيتبعت، لكن الـ card state machine بتتـreset بعد error response بيخلي الـ upper address يتنسى.

#### الحل
نضيف re-send للـ `SD_ADDR_EXT` بعد أي command error:

```c
/* drivers/mmc/core/sd.c */
static int sd_send_addr_ext(struct mmc_card *card, u32 upper_addr)
{
    struct mmc_command cmd = {};

    /* CMD22 - SD_ADDR_EXT */
    cmd.opcode = SD_ADDR_EXT;
    cmd.arg    = upper_addr & 0x3F;  /* [5:0] only */
    cmd.flags  = MMC_RSP_R1 | MMC_CMD_AC;

    return mmc_wait_for_cmd(card->host, &cmd, 3);
}

static int sd_read_write_large(struct mmc_card *card, u64 sector,
                                void *buf, unsigned int blocks, int write)
{
    int err;
    u32 upper = (sector >> 32) & 0x3F;

    /* Always re-assert extended address before large transfers */
    if (upper) {
        err = sd_send_addr_ext(card, upper);
        if (err)
            return err;
    }

    /* now do the actual CMD18/CMD25 */
    return mmc_simple_transfer(card, (u32)sector, buf, blocks, write);
}
```

نتحقق:
```bash
# قبل وبعد الـ fix
echo 0 > /sys/kernel/debug/mmc0/clock_delay
mmcblk_test write 32G 1G  # يكتب فوق الـ 32GB boundary
dmesg | grep -v "^$" | tail -20
```

#### الدرس المستفاد
الـ `SD_ADDR_EXT` (CMD22) في `sd.h` هو الـ command الوحيد اللي بيمد الـ SD addressing لـ ما فوق 32 bits. في الـ automotive و industrial systems اللي بتتعامل مع storage كبير، أي regression في الـ CMD22 sequence هيتسبب في silent data corruption عند الـ 32GB boundary بالظبط.
## Phase 7: مصادر ومراجع

### مصادر LWN.net

**الـ** LWN.net من أهم المصادر لمتابعة تطور الـ MMC/SD subsystem في الـ Linux kernel:

| العنوان | الرابط | الأهمية |
|---------|--------|---------|
| MMC layer (early framework patch) | [lwn.net/Articles/82765](https://lwn.net/Articles/82765/) | أول patch لإضافة MMC framework |
| new driver: MMC framework (2.4.19) | [lwn.net/Articles/7176](https://lwn.net/Articles/7176/) | الأساس التاريخي للـ MMC layer |
| Secure Digital (SD) support | [lwn.net/Articles/126098](https://lwn.net/Articles/126098/) | أول دعم للـ SD cards: SCR register, 4-bit mode |
| mmc: core: add SD4.0 support | [lwn.net/Articles/642336](https://lwn.net/Articles/642336/) | إضافة دعم SD 4.0 والـ SDUC |
| MMC updates | [lwn.net/Articles/253888](https://lwn.net/Articles/253888/) | تحديثات SDIO و SPI support |
| Add MMC erase and secure erase V3 | [lwn.net/Articles/393535](https://lwn.net/Articles/393535/) | إضافة `SD_ERASE_ARG` و `SD_DISCARD_ARG` |
| mmc: introduce multi-block read gap tuning | [lwn.net/Articles/1030167](https://lwn.net/Articles/1030167/) | تحسينات حديثة للأداء |

---

### توثيق الـ Linux Kernel الرسمي

**الـ** kernel documentation المتعلقة مباشرةً بالـ MMC/SD subsystem:

```
Documentation/driver-api/mmc/index.rst
Documentation/driver-api/mmc/mmc-dev-attrs.rst
Documentation/driver-api/mmc/mmc-dev-parts.rst
Documentation/driver-api/mmc/mmc-tools.rst
Documentation/driver-api/mmc/core.rst
```

الروابط المباشرة على kernel.org:

- [MMC/SD/SDIO card support — The Linux Kernel documentation](https://docs.kernel.org/driver-api/mmc/index.html)
- [SD and MMC Device Partitions](https://docs.kernel.org/driver-api/mmc/mmc-dev-parts.html)
- [SD and MMC Block Device Attributes](https://docs.kernel.org/6.8/driver-api/mmc/mmc-dev-attrs.html)
- [MMC tools introduction](https://docs.kernel.org/driver-api/mmc/mmc-tools.html)

---

### الملفات المصدرية الأساسية في الـ Kernel

**الـ** header files والـ source files المرتبطة بالـ `include/linux/mmc/sd.h`:

```
include/linux/mmc/sd.h          ← الملف المدروس (SD commands, OCR, SCR)
include/linux/mmc/mmc.h         ← MMC-specific commands و registers
include/linux/mmc/host.h        ← host controller interface
include/linux/mmc/card.h        ← struct mmc_card definition
drivers/mmc/core/sd.c           ← SD card initialization و SD_SWITCH logic
drivers/mmc/core/sd_ops.c       ← تنفيذ SD_APP_* commands
drivers/mmc/core/mmc.c          ← MMC card initialization
drivers/mmc/host/sdhci.c        ← SDHCI host controller driver
```

---

### Commits مهمة في تاريخ الـ SD support

**الـ** commits التالية تمثل نقاط تحول في تطور الـ `sd.h`:

| التغيير | الوصف |
|---------|-------|
| إضافة SD 4.0 support | أضاف `SD_OCR_2T` لدعم SDUC (بطاقات > 2TB) |
| إضافة `SD_SWITCH_VOLTAGE` (CMD11) | لدعم الـ 1.8V signaling في UHS-I |
| إضافة `SD_ADDR_EXT` (CMD22) | Class 2 command للـ extended addressing |
| إضافة `SD_READ_EXTR_SINGLE`/`SD_WRITE_EXTR_SINGLE` | Class 11 commands (CMD48/CMD49) |
| إضافة `SD_DISCARD_ARG` | دعم الـ discard بجانب الـ erase |

للبحث في تاريخ الـ commits:

```bash
git log --oneline -- include/linux/mmc/sd.h
git log --oneline --grep="SD_SWITCH" -- drivers/mmc/core/
git log --oneline --grep="SDUC\|SD4.0\|sd.h" -- include/linux/mmc/
```

---

### نقاشات Mailing List

**الـ** mailing list الرسمي للـ MMC subsystem هو `linux-mmc@vger.kernel.org`.

للبحث في الأرشيف:

- [lore.kernel.org/linux-mmc](https://lore.kernel.org/linux-mmc/) — الأرشيف الرسمي لـ linux-mmc list
- [marc.info/?l=linux-kernel&s=MMC+SD](https://marc.info/?l=linux-kernel&s=MMC+SD) — LKML archive مفلتر

---

### kernelnewbies.org — تحديثات MMC/SD عبر إصدارات الـ Kernel

**الـ** kernelnewbies.org يوثق التغييرات في كل إصدار kernel:

| الإصدار | الرابط | ما يخص الـ MMC/SD |
|---------|--------|-------------------|
| Linux 2.6.17 | [kernelnewbies.org/Linux_2_6_17](https://kernelnewbies.org/Linux_2_6_17) | أول دعم لـ i.MX و AT91 MMC controllers |
| Linux 2.6.24 | [kernelnewbies.org/Linux_2_6_24](https://kernelnewbies.org/Linux_2_6_24) | إضافة SDIO و SPI support للـ MMC layer |
| Linux 2.6.30 | [kernelnewbies.org/Linux_2_6_30](https://kernelnewbies.org/Linux_2_6_30) | تحسينات على الـ MMC subsystem |
| Linux 2.6.31 | [kernelnewbies.org/Linux_2_6_31](https://kernelnewbies.org/Linux_2_6_31) | تحديثات MMC/SD drivers |
| Linux 2.6.33 | [kernelnewbies.org/Linux_2_6_33](https://kernelnewbies.org/Linux_2_6_33) | تحسينات إضافية |
| Linux 6.5 | [kernelnewbies.org/Linux_6.5](https://kernelnewbies.org/Linux_6.5) | أحدث تحديثات الـ MMC/SD/SDIO subsystem |
| Linux 6.6 | [kernelnewbies.org/Linux_6.6](https://kernelnewbies.org/Linux_6.6) | تحديثات MMC subsystem |
| Linux 6.7 | [kernelnewbies.org/Linux_6.7](https://kernelnewbies.org/Linux_6.7) | آخر التغييرات في الـ MMC/SD/SDIO |

---

### elinux.org — موارد Embedded Linux

**الـ** elinux.org يحتوي على موارد عملية للـ embedded systems:

- [EVM Second MMC/SD](https://elinux.org/Second_MMC_/_SD) — إضافة slot ثاني للـ MMC/SD على EVM boards
- [Tests:SDHI-DMA-Sync](https://elinux.org/Tests:SDHI-DMA-Sync) — اختبارات الـ DMA sync مع SD cards على R-Car H2
- [Tests:SD-SDHI-SDR104](https://elinux.org/Tests:SD-SDHI-SDR104) — نتائج اختبار SDR104 speed mode
- [Didj SD MMC Expansion](https://elinux.org/Didj_SD_MMC_Expansion) — مثال embedded على إضافة SD/MMC support
- [ZipIt MMC](https://elinux.org/ZipIt_MMC) — تفاصيل الـ wiring و kernel driver للـ embedded MMC

---

### الكتب المرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل المناسب**: Chapter 17 — Network Drivers (للمقارنة مع bus subsystems)
- **ملاحظة**: LDD3 لا يغطي الـ MMC/SD بشكل مباشر لكنه يشرح مبدأ الـ host/device driver model المستخدم في الـ MMC subsystem
- متاح مجاناً على: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل المناسب**: Chapter 13 — The Block I/O Layer
- الـ SD card يعمل كـ block device، فهذا الفصل يشرح كيفية وصول الـ MMC/SD layer إلى الـ block subsystem
- الـ `SD_READ_EXTR_SINGLE`/`SD_WRITE_EXTR_SINGLE` مفهومتان يربطهما هذا الفصل بالـ kernel architecture

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل المناسب**: Chapter 9 — File Systems و Chapter 14 — Kernel Debugging
- يشرح كيفية boot من SD card على embedded boards، وكيفية debugging الـ MMC driver issues
- مثالي لفهم `SD_SEND_IF_COND` (CMD8) و card initialization sequence في بيئة embedded

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- يغطي الـ bus subsystem بعمق، مفيد لفهم كيفية تسجيل الـ SD card كـ `mmc_bus_type`

---

### مواصفات SD الرسمية (SD Association)

**الـ** SD Physical Layer Specification هي المرجع الأساسي لكل الـ commands والـ registers المعرفة في `sd.h`:

- [SD Association Developer Page](https://www.sdcard.org/developers/)
- [Low Voltage Signaling Overview](https://www.sdcard.org/developers/sd-standard-overview/low-voltage-signaling/) — يشرح `SD_OCR_S18R` و CMD11

---

### مصادر إضافية

- [Linux Kernel Driver Database — MMC config](https://cateee.net/lkddb/web-lkddb/MMC.html) — قائمة شاملة بكل Kconfig options للـ MMC
- [Toradex Developer Center — SD/MMC Card](https://developer.toradex.com/linux-bsp/application-development/peripheral-access/sdmmc-card-linux/) — توثيق عملي من منظور embedded Linux BSP
- [STM32MPU MMC Overview](https://wiki.st.com/stm32mpu/wiki/MMC_overview) — مثال واقعي على استخدام الـ MMC driver على STM32MP SoCs

---

### Search Terms للبحث عن مزيد من المعلومات

للبحث في مصادر مختلفة، استخدم الـ keywords التالية:

```
# للبحث عن SD commands
"SD_SWITCH" linux kernel site:lore.kernel.org
"CMD6 SD card" linux driver
"SD_SEND_IF_COND CMD8" voltage check linux

# للبحث عن OCR و voltage
"SD_OCR_S18R" 1.8V switching linux kernel
"SD_OCR_CCS" SDHC SDXC capacity linux
"SDUC SD Ultra Capacity" linux kernel

# للبحث عن SCR register
"SCR register SD card" linux parsing
"SD_APP_SEND_SCR CMD51" linux kernel

# للبحث عن erase/discard
"SD_DISCARD_ARG" linux mmc
"SD erase vs discard" linux kernel

# للبحث عن extended commands (SD 4.0+)
"CMD48 CMD49 SD card" linux
"SD_READ_EXTR_SINGLE" linux kernel
"SD_ADDR_EXT CMD22" linux

# للبحث في الـ mailing list مباشرةً
site:lore.kernel.org/linux-mmc/ SD_SWITCH
site:lore.kernel.org/linux-mmc/ SDUC
```
## Phase 8: Writing simple module

### الفكرة

**الـ** `sd.h` بيعرّف الـ commands والـ constants اللي بيستخدمها الـ SD subsystem — من أهمهم `SD_SWITCH` اللي بيتبعت عشان تتحقق أو تغيّر الـ bus speed (HS, UHS). الدالة اللي بتبعت الـ command ده فعلياً هي `mmc_sd_switch()` الموجودة في `drivers/mmc/core/sd_ops.c` والمعلنة كـ `EXPORT_SYMBOL_GPL`.

هنعمل **kprobe** على `mmc_sd_switch` عشان نلتقط كل مرة بيحاول فيها الكرنل يبدّل أو يتحقق من الـ speed mode لكارت SD، وندي في الـ log اسم الكارت والـ mode والـ group والـ value.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * sd_switch_probe.c
 *
 * kprobe on mmc_sd_switch() — fires every time the kernel
 * sends an SD_SWITCH command to an SD card.
 *
 * mmc_sd_switch(struct mmc_card *card, bool mode,
 *               int group, u8 value, u8 *resp)
 *
 *   mode  : 0 = check, 1 = switch (SD_SWITCH_CHECK / SD_SWITCH_SET)
 *   group : function group (0 = access mode / bus speed)
 *   value : target function index (0=default, 1=HS, ...)
 */

#include <linux/kernel.h>      /* pr_info, pr_err */
#include <linux/module.h>      /* MODULE_* macros, module_init/exit */
#include <linux/kprobes.h>     /* kprobe struct, register/unregister */
#include <linux/mmc/card.h>    /* struct mmc_card, mmc_card_name() */
#include <linux/mmc/host.h>    /* mmc_sd_switch declaration */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Learning Kernel");
MODULE_DESCRIPTION("kprobe on mmc_sd_switch — trace SD SWITCH commands");

/* ------------------------------------------------------------------ */
/* pre_handler: runs just before mmc_sd_switch() executes             */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * Argument layout on x86-64 (System V ABI):
     *   rdi = arg0 → struct mmc_card *card
     *   rsi = arg1 → bool mode
     *   rdx = arg2 → int group
     *   rcx = arg3 → u8 value
     *   r8  = arg4 → u8 *resp   (not needed)
     */
    struct mmc_card *card = (struct mmc_card *)regs->di;
    bool   mode  = (bool)regs->si;
    int    group = (int)regs->dx;
    u8     value = (u8)regs->cx;

    /* mmc_hostname() returns the host controller name (e.g. "mmc0") */
    pr_info("sd_switch_probe: card=%s host=%s mode=%s group=%d value=%u\n",
            card->cid.prod_name,
            mmc_hostname(card->host),
            mode ? "SWITCH" : "CHECK",
            group,
            value);

    return 0; /* 0 = let the probed function execute normally */
}

/* ------------------------------------------------------------------ */
/* kprobe descriptor                                                   */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "mmc_sd_switch",   /* symbol exported by sd_ops.c */
    .pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ */
/* init / exit                                                         */
/* ------------------------------------------------------------------ */
static int __init sd_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("sd_switch_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("sd_switch_probe: planted at %pS\n", kp.addr);
    return 0;
}

static void __exit sd_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("sd_switch_probe: removed\n");
}

module_init(sd_probe_init);
module_exit(sd_probe_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/kprobes.h` | بيعرّف `struct kprobe` وكل الـ API الخاص بالـ kprobes |
| `linux/mmc/card.h` | عشان `struct mmc_card` و `card->cid.prod_name` |
| `linux/mmc/host.h` | عشان `mmc_hostname()` اللي بترجع اسم الـ host controller |

**الـ** `linux/mmc/sd.h` مش محتاجين نضمّه مباشرةً هنا لأن المعلومات الوحيدة اللي عايزينها هي `struct mmc_card` — بس من المفيد فهمه لأنه بيوضّح المعنى الدلالي للـ `mode` و `value` اللي بنطبعهم.

---

#### الـ `handler_pre` — الـ callback

الكرنل على x86-64 بيمرّر الـ arguments عن طريق registers: `rdi`, `rsi`, `rdx`, `rcx`. الـ `pt_regs` بيحفظ حالة الـ registers وقت الـ probe، فبنقدر نقرأ منه مباشرةً من غير ما نعدّل أي حاجة.

- **`card->cid.prod_name`**: الاسم التجاري للكارت زي `SD32G` — بييجي من الـ CID register اللي الكرنل بيقرأه لما الكارت يتعرف.
- **`mode`**: لو `0` معناها CHECK (نتحقق بس)، لو `1` معناها SWITCH (نغيّر فعلاً) — التعريفين `SD_SWITCH_CHECK` و `SD_SWITCH_SET` موجودين في `sd.h`.
- **`group`**: الـ function group — الـ group 0 هو الـ access mode (bus speed).
- **`value`**: الـ target mode — `0` = default, `1` = High Speed.
- **الـ return `0`**: لازم الـ `pre_handler` يرجّع `0` عشان الكرنل يكمّل تنفيذ الدالة الأصلية.

---

#### الـ `struct kprobe`

الـ `symbol_name` بيقول للكرنل يحط الـ breakpoint على `mmc_sd_switch` عن طريق الـ symbol table — أحسن من hardcode عنوان لأنه independent من الـ kernel version.

---

#### الـ `module_init` — التسجيل

**الـ** `register_kprobe()` بيحط الـ probe بشكل dynamic وقت الـ runtime. لو فشل (مثلاً لو الـ symbol مش موجود أو الـ kprobes مش enabled)، بنرجّع الـ error فوراً من غير ما نسيب الـ module loaded بحالة غلط.

---

#### الـ `module_exit` — إلغاء التسجيل

**الـ** `unregister_kprobe()` في الـ exit ضرورية عشان تزيل الـ breakpoint وتضمن إن الكرنل مش هيحاول يشغّل `handler_pre` بعد ما الـ module اتأزّل من الذاكرة — وده لو حصل هيودّي لـ use-after-free crash.

---

### كيفية البناء والتشغيل

```bash
# Makefile بسيط
obj-m += sd_switch_probe.o

# بناء الـ module
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# تحميل الـ module
sudo insmod sd_switch_probe.ko

# متابعة الـ log لما تحاول تحمّل كارت SD
sudo dmesg -w | grep sd_switch_probe

# مثال لـ output لما كارت UHS بيتحمّل:
# sd_switch_probe: card=SD32G host=mmc0 mode=CHECK group=0 value=1
# sd_switch_probe: card=SD32G host=mmc0 mode=SWITCH group=0 value=1

# إزالة الـ module
sudo rmmod sd_switch_probe
```
