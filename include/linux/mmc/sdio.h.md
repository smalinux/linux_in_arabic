## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ Subsystem ده؟

الملف `include/linux/mmc/sdio.h` جزء من **MULTIMEDIA CARD (MMC), SECURE DIGITAL (SD) AND SDIO SUBSYSTEM** — المسؤول عنه في الـ MAINTAINERS هو Ulf Hansson، وبيغطي كل حاجة في `drivers/mmc/` و `include/linux/mmc/`.

---

### الصورة الكبيرة — قصة بسيطة

تخيل إنك بتشتري **لاب توب** وعايز تضيف ليه WiFi card أو Bluetooth. بدل ما تفتح الجهاز وتلحم chip جديدة، بتحط **بطاقة صغيرة في slot** من جنب الجهاز وخلاص — ده بالظبط اللي **SDIO** بيعمله.

**SDIO (Secure Digital Input/Output)** هو امتداد لبروتوكول SD Cards العادي، بس بدل ما البطاقة بس تخزّن data، ممكن تكون فيها أجهزة كاملة زي:
- WiFi adapters (الأكثر شيوعاً في embedded systems)
- Bluetooth chips
- GPS receivers
- Cameras
- UART/SPI bridges

---

### المشكلة اللي بيحلها الملف ده

لما الـ kernel بيتكلم مع SDIO card، بيبعت **commands** رقمية على الـ bus (زي رسايل مشفرة). الـ card بتفهم الأرقام دي وبترد عليها. المشكلة إن:

1. لازم يكون في **قاموس مشترك** — الـ kernel يعرف إن الـ command رقم 52 معناه "اكتب/اقرأ register واحد مباشرة".
2. الـ card فيها **registers ثابتة** معروفة العناوين لكل SDIO card في العالم، محتاجين نعرف عناوينها.
3. الردود اللي بتيجي من الـ card لازم نعرف نفسّرها صح.

الملف `sdio.h` هو **القاموس ده بالظبط** — مجرد `#define` ثوابت بتمثل:
- أرقام الـ commands
- عناوين الـ registers
- بتات الـ status في الردود

---

### تفصيل محتوى الملف

#### 1. SDIO Commands

```c
#define SD_IO_SEND_OP_COND   5   /* سؤال: هل فيه SDIO card هنا؟ */
#define SD_IO_RW_DIRECT     52   /* اقرأ/اكتب register واحد (1 byte) */
#define SD_IO_RW_EXTENDED   53   /* اقرأ/اكتب block كامل من data */
```

الـ **CMD52** زي إنك بتطق باب أوضة واحدة — بتاخد بايت وحدة.
الـ **CMD53** زي إنك بتشيل صندوق كبير من أوضة — بتنقل blocks كاملة من data.

#### 2. R4 و R5 Response Bits

لما الـ card بتجاوب على الـ CMD5 (SD_IO_SEND_OP_COND)، بتبعت **R4 response**:

```c
#define R4_18V_PRESENT    (1<<24)  /* الكارت بتدعم 1.8V (UHS speed) */
#define R4_MEMORY_PRESENT (1<<27)  /* الكارت فيها memory part كمان (combo card) */
```

لما بتبعت CMD52 أو CMD53، الرد بيكون **R5** وفيه error bits:

```c
#define R5_COM_CRC_ERROR   (1<<15)  /* خطأ في CRC — البيانات اتبعتت غلط */
#define R5_ILLEGAL_COMMAND (1<<14)  /* أمر مش مفهوم */
#define R5_ERROR           (1<<11)  /* خطأ عام */
#define R5_FUNCTION_NUMBER (1<<9)   /* رقم function مش موجود */
#define R5_OUT_OF_RANGE    (1<<8)   /* عنوان register خارج النطاق */
```

#### 3. Card Common Control Registers (CCCR)

كل SDIO card في العالم لازم يكون فيها جزء ثابت اسمه **CCCR** — زي بطاقة التعريف للكارت. الـ CCCR بيبدأ من العنوان 0x00 وبيحتوي على:

| Register | عنوان | الوظيفة |
|----------|--------|---------|
| `SDIO_CCCR_CCCR` | 0x00 | إصدار الـ CCCR spec |
| `SDIO_CCCR_IOEx` | 0x02 | تفعيل/تعطيل كل function |
| `SDIO_CCCR_IORx` | 0x03 | هل الـ function جاهزة؟ |
| `SDIO_CCCR_IENx` | 0x04 | تفعيل interrupts لكل function |
| `SDIO_CCCR_INTx` | 0x05 | أي function بعتت interrupt؟ |
| `SDIO_CCCR_CAPS` | 0x08 | قدرات الكارت (multi-block, suspend…) |
| `SDIO_CCCR_SPEED` | 0x13 | سرعات الـ bus المدعومة (HS, UHS) |

#### 4. Bus Width وسرعات الـ interface

```c
#define SDIO_BUS_WIDTH_1BIT  0x00  /* بطيء — خط data واحد */
#define SDIO_BUS_WIDTH_4BIT  0x02  /* أسرع — 4 خطوط data */

#define SDIO_SPEED_SDR12   (0<<1)  /* 12.5 MB/s */
#define SDIO_SPEED_SDR25   (1<<1)  /* 25 MB/s (High-Speed) */
#define SDIO_SPEED_SDR50   (2<<1)  /* 50 MB/s */
#define SDIO_SPEED_SDR104  (3<<1)  /* 104 MB/s */
#define SDIO_SPEED_DDR50   (4<<1)  /* 50 MB/s DDR */
```

#### 5. Function Basic Registers (FBR)

كل **function** جوه الكارت (ممكن يكون في كارت واحدة 7 functions) عندها **FBR** خاص بيها من العنوان `function * 0x100`:

```c
#define SDIO_FBR_BASE(f)   ((f) * 0x100)  /* function 1 → 0x100, function 2 → 0x200 */
#define SDIO_FBR_STD_IF    0x00  /* نوع الـ function (WiFi? Bluetooth? Camera?) */
#define SDIO_FBR_CIS       0x09  /* مؤشر لـ Card Information Structure */
#define SDIO_FBR_BLKSIZE   0x10  /* حجم الـ block للنقل */
```

---

### قصة: كيف الـ kernel يتعرف على SDIO WiFi Card؟

**السيناريو:** حطّيت USB-to-SDIO dongle بفيه Marvell WiFi chip.

1. **MMC Host Controller** حسّ إن في كارت اتحطت — بعت **CMD5** (SD_IO_SEND_OP_COND).
2. الكارت ردّت بـ **R4** — الـ kernel شاف إن `R4_MEMORY_PRESENT = 0` يعني SDIO-only.
3. الـ kernel قرأ **CCCR** من العنوان 0x00 — عرف إن الكارت فيها كام function وبتدعم إيه.
4. قرأ **FBR** لكل function — عرف إن Function 1 نوعها `0x01` يعني WiFi (WLAN).
5. قرأ **CIS** (Card Information Structure) — عرف الـ vendor ID والـ device ID.
6. الـ kernel نادى الـ **sdio_bus** وقاله: "في device جديد vendor=Marvell, device=0x9129".
7. الـ SDIO bus دور على أي driver عنده `sdio_device_id` بيطابق ده — لقى `mwifiex_sdio`.
8. الـ driver اتحمل وبدأ يتكلم مع الـ WiFi chip عن طريق CMD52/CMD53.

---

### ليه الملف ده مهم؟

بدونه، كل driver هيضطر يكتب الأرقام الـ magic بنفسه — وهيبقى في inconsistency وأخطاء. الملف ده بيوفر **single source of truth** للـ SDIO spec constants.

---

### الملفات المرتبطة

#### Core Headers (include/linux/mmc/)

| ملف | الوظيفة |
|-----|---------|
| `sdio.h` | **الملف ده** — ثوابت الـ SDIO spec (commands, registers) |
| `sdio_func.h` | تعريف `struct sdio_func` و `struct sdio_driver` وكل الـ API |
| `sdio_ids.h` | vendor/device IDs لكل SDIO chips معروفة |
| `core.h` | `struct mmc_command`, `struct mmc_request` |
| `host.h` | `struct mmc_host` — الـ controller |
| `card.h` | `struct mmc_card` — الكارت نفسها |
| `mmc.h` | ثوابت MMC spec (مشابه لـ sdio.h بس للـ eMMC) |
| `sd.h` | ثوابت SD spec |

#### Core Implementation (drivers/mmc/core/)

| ملف | الوظيفة |
|-----|---------|
| `sdio.c` | initialization وdetection للـ SDIO cards |
| `sdio_ops.c` | تنفيذ CMD52/CMD53 على مستوى منخفض |
| `sdio_io.c` | الـ API العالي: `sdio_readb`, `sdio_writeb`, إلخ |
| `sdio_cis.c` | قراءة وتفسير الـ CIS tuples |
| `sdio_irq.c` | معالجة الـ interrupts من SDIO functions |
| `sdio_bus.c` | ربط الـ SDIO drivers بالـ devices (bus matching) |
| `sdio_uart.c` | مثال driver: UART function فوق SDIO |

#### Host Controllers (drivers/mmc/host/)

أمثلة: `sdhci.c` (أشهرهم — بيغطي أغلب hardware), `dw_mmc.c` (Synopsys DesignWare), `sdhci-msm.c` (Qualcomm).
## Phase 2: شرح الـ SDIO Framework

### المشكلة اللي الـ SDIO جاي يحلها

تخيل إن عندك slot واحد بس في الجهاز المدمج — الـ SD card slot — وعايز تحط فيه مش بس storage، لكن كمان WiFi أو Bluetooth أو GPS. المشكلة إن الـ SD slot اصلاً اتصمم للـ block storage، فالـ kernel في الأول ماعندوش أي abstraction يخلي device driver لـ WiFi يشتغل فوق SD interface.

**الـ SDIO** (Secure Digital Input/Output) هو امتداد لمعيار الـ SD اللي بيسمح بنقل I/O عام — مش بس block data — فوق نفس الـ physical interface. الكيرنل بيحتاج framework يفهم إن الكارد دي مش storage، ويعرف يـ enumerate الـ functions الجواها، ويوفر API موحدة للـ drivers اللي هتشتغل فوقها.

---

### الحل اللي بيقدمه الكيرنل

الكيرنل بيعامل الـ SDIO card كـ **multi-function device** — زي PCI تقريباً — كل function جواها ليها driver مستقل. الـ MMC/SD subsystem بيعمل enumeration للـ card، بيقرأ الـ **CCCR** و الـ **FBR** بتاع كل function، وبيخلق `sdio_func` device لكل function يـ probe الـ driver المناسب.

---

### التشبيه الحقيقي: USB Hub مع Devices

الـ SDIO card تقدر تتخيلها زي **USB hub صغير مدمج في كارد واحد**:

| مفهوم SDIO | مقابله في USB |
|---|---|
| الـ SDIO card نفسها | USB Hub |
| الـ CCCR (Card Common Control Registers) | USB Hub Descriptor |
| كل `sdio_func` | USB Device متوصل بالـ Hub |
| الـ FBR (Function Basic Registers) | USB Device Descriptor |
| الـ CIS (Card Information Structure) | USB Configuration Descriptor |
| `SD_IO_RW_DIRECT` (CMD52) | Control Transfer لـ USB |
| `SD_IO_RW_EXTENDED` (CMD53) | Bulk Transfer لـ USB |
| الـ `sdio_driver` | USB Driver |
| الـ `sdio_claim_host` / `sdio_release_host` | USB locking على الـ bus |
| الـ IRQ handler في `sdio_func` | USB interrupt endpoint |

**الفرق الجوهري عن USB**: الـ SDIO shared-bus يعني كل traffic بيمر على نفس الأسلاك، فمفيش concurrent access — عشان كده الـ `sdio_claim_host` ضرورية.

---

### Architecture الكامل في الكيرنل

```
┌──────────────────────────────────────────────────────────────┐
│                    SDIO Function Drivers                       │
│   brcmfmac (WiFi)   btmrvl (BT)   libertas (WiFi)  ...       │
│        │                 │                │                    │
│   sdio_driver        sdio_driver      sdio_driver             │
│   .probe()           .probe()         .probe()                │
└────────────────────────┬─────────────────────────────────────┘
                         │  sdio_func API
                         │  sdio_readb/writeb
                         │  sdio_memcpy_fromio/toio
                         │  sdio_claim_irq
                         ▼
┌──────────────────────────────────────────────────────────────┐
│                   SDIO Core (drivers/mmc/core/sdio*.c)        │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              struct mmc_card                          │    │
│  │  ┌─────────┐ ┌─────────┐ ┌──────────────────────┐   │    │
│  │  │sdio_cccr│ │sdio_cis │ │sdio_func[0..6]       │   │    │
│  │  │.sdio_vsn│ │.vendor  │ │  [0]=func1 (WiFi)    │   │    │
│  │  │.multi_  │ │.device  │ │  [1]=func2 (BT)      │   │    │
│  │  │  block  │ │.blksize │ │  ...                  │   │    │
│  │  └─────────┘ └─────────┘ └──────────────────────┘   │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                               │
│  SDIO Bus Driver (sdio_bus.c)                                 │
│  sdio_match_device() → يطابق sdio_func مع sdio_driver         │
└────────────────────────┬─────────────────────────────────────┘
                         │  mmc_request / mmc_command
                         │  CMD52 / CMD53
                         ▼
┌──────────────────────────────────────────────────────────────┐
│                   MMC Core (drivers/mmc/core/)                │
│                                                               │
│  mmc_wait_for_req()   mmc_wait_for_cmd()                      │
│  struct mmc_request → struct mmc_command → struct mmc_data    │
└────────────────────────┬─────────────────────────────────────┘
                         │  host ops (.request, .set_ios, ...)
                         ▼
┌──────────────────────────────────────────────────────────────┐
│               MMC Host Controller Driver                      │
│     sdhci-esdhc   sunxi-mmc   dw_mmc   omap_hsmmc  ...       │
│                                                               │
│   struct mmc_host  →  hardware registers (MMIO)              │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         ▼
               ┌─────────────────┐
               │  Physical SDIO  │
               │  Card (1/4-bit  │
               │  SD bus)        │
               │                 │
               │  ┌───────────┐  │
               │  │ CCCR      │  │ ← registers at address 0x0000-0x00FF
               │  │ FBR[1]    │  │ ← registers at address 0x0100-0x01FF
               │  │ FBR[2]    │  │ ← registers at address 0x0200-0x02FF
               │  │ CIS       │  │ ← tuple chain
               │  │ Fn1 Logic │  │
               │  │ Fn2 Logic │  │
               │  └───────────┘  │
               └─────────────────┘
```

---

### الـ Core Abstraction: Function كـ Device

الفكرة الأساسية في الـ SDIO Framework هي إن **كل function جوا الكارد = device مستقل في الـ Linux device model**.

الـ `struct sdio_func` هو الـ central object:

```c
struct sdio_func {
    struct mmc_card     *card;        /* الكارد الأم */
    struct device        dev;         /* Linux device — ده اللي بيـ register في sysfs */
    sdio_irq_handler_t  *irq_handler; /* callback لما interrupt يجي */
    unsigned int         num;         /* رقم الـ function (1-7) */

    unsigned char        class;       /* نوع الـ function (UART, BT, WiFi...) */
    unsigned short       vendor;      /* Vendor ID — زي USB */
    unsigned short       device;      /* Device ID */

    unsigned             max_blksize; /* أكبر block size تقدر تستخدمه */
    unsigned             cur_blksize; /* الـ block size الحالي */

    u8                  *tmpbuf;      /* DMA-safe scratch buffer */
    struct sdio_func_tuple *tuples;   /* CIS tuples خاصة بالـ function */
};
```

والـ `struct sdio_driver` هو الـ driver side:

```c
struct sdio_driver {
    char *name;
    const struct sdio_device_id *id_table; /* جدول الـ vendor/device IDs */

    int  (*probe) (struct sdio_func *, const struct sdio_device_id *);
    void (*remove)(struct sdio_func *);
    void (*shutdown)(struct sdio_func *);

    struct device_driver drv; /* base class — نفس نهج PCI و USB */
};
```

**ربط الـ driver بالـ device**: الـ `sdio_bus.c` بيعمل match بين الـ `sdio_func.vendor/device/class` وبين الـ `sdio_device_id` في الـ `id_table`. لو match حصل → بيتعمل `probe()`.

---

### هرم الـ Structs — إزاي كل حاجة مترابطة

```
struct mmc_card
│
├── struct mmc_host      *host          ← الـ controller HW
├── struct sdio_cccr      cccr          ← parsed من CCCR registers
│     ├── sdio_vsn                      ← SDIO spec version
│     ├── multi_block                   ← يدعم CMD53 multi-block؟
│     ├── high_speed                    ← يدعم HS mode؟
│     └── enable_async_irq              ← async interrupt mode
│
├── struct sdio_cis       cis           ← common Card Info Structure
│     ├── vendor                        ← من CISTPL_MANFID
│     ├── device
│     ├── blksize
│     └── max_dtr
│
└── struct sdio_func     *sdio_func[7]  ← max 7 functions
      │
      └── struct sdio_func (per function)
            ├── struct mmc_card  *card  ← pointer للأب
            ├── struct device     dev   ← Linux device object
            ├── num                     ← function number (1-7)
            ├── class/vendor/device     ← for matching
            ├── cur_blksize             ← negotiated block size
            ├── irq_handler             ← يتسجل بـ sdio_claim_irq()
            └── struct sdio_func_tuple *tuples  ← linked list of CIS tuples
```

---

### الـ CCCR — قلب الـ SDIO Card

الـ **Card Common Control Registers** هي أول 256 byte في address space الكارت (0x00000–0x000FF). الكيرنل بيقراها بـ CMD52 في أثناء الـ card initialization.

أهم الـ registers:

| Register | Offset | الغرض |
|---|---|---|
| `SDIO_CCCR_CCCR` | 0x00 | CCCR version + SDIO spec version |
| `SDIO_CCCR_IOEx` | 0x02 | Enable/disable كل function (bitmask) |
| `SDIO_CCCR_IORx` | 0x03 | هل الـ function ready؟ (read-only) |
| `SDIO_CCCR_IENx` | 0x04 | تفعيل الـ interrupt لكل function |
| `SDIO_CCCR_INTx` | 0x05 | إيه الـ function اللي رفعت interrupt؟ |
| `SDIO_CCCR_ABORT`| 0x06 | إلغاء عملية أو reset الكارت |
| `SDIO_CCCR_IF`   | 0x07 | bus width (1-bit vs 4-bit) |
| `SDIO_CCCR_CAPS` | 0x08 | capabilities (multi-block, suspend, ...) |
| `SDIO_CCCR_SPEED`| 0x13 | speed mode negotiation (SDR12→SDR104) |

```c
/* مثال: الكيرنل بيقرأ capabilities من CCCR */
#define SDIO_CCCR_CAP_SMB   0x02  /* supports multi-block transfers CMD53 */
#define SDIO_CCCR_CAP_SRW   0x04  /* supports read-wait protocol */
#define SDIO_CCCR_CAP_SBS   0x08  /* supports suspend/resume */
#define SDIO_CCCR_CAP_S4MI  0x10  /* interrupt during 4-bit CMD53 */
```

لو `SDIO_CCCR_CAP_SMB` مش set، الكيرنل بيبعت بس CMD52 (single byte) مش CMD53 (block/multi-byte). ده critical للـ performance.

---

### الـ FBR — بطاقة التعريف لكل Function

الـ **Function Basic Registers** موجودة لكل function عند offset `(function_number * 0x100)`:

```c
#define SDIO_FBR_BASE(f)   ((f) * 0x100)  /* function 1 → 0x100, function 2 → 0x200, ... */
```

| Register | Offset من الـ FBR Base | الغرض |
|---|---|---|
| `SDIO_FBR_STD_IF` | +0x00 | standard interface code (BT=0x02, WiFi=0x07) |
| `SDIO_FBR_POWER`  | +0x02 | power selection |
| `SDIO_FBR_CIS`    | +0x09 | pointer لـ CIS الخاص بالـ function (3 bytes) |
| `SDIO_FBR_BLKSIZE`| +0x10 | block size للـ function (2 bytes) |

الكيرنل بيقرأ `SDIO_FBR_STD_IF` عشان يحط الـ `sdio_func.class`، ومنه بيعمل match مع drivers زي `SDIO_DEVICE_CLASS(SDIO_CLASS_WLAN)`.

---

### الأوامر الثلاثة الأساسية

#### CMD5 — `SD_IO_SEND_OP_COND`
أول command يتبعت للـ SDIO card أثناء الـ enumeration. الـ card بترد بـ **R4 response** اللي بتقول:
- `R4_MEMORY_PRESENT` (bit 27): هل في memory function كمان؟ → ده الـ SD combo card
- `R4_18V_PRESENT` (bit 24): هل تقدر تشتغل على 1.8V؟

#### CMD52 — `SD_IO_RW_DIRECT`
نقل **byte واحد** read أو write. الـ argument format هو 32-bit:
```
[31]    R/W flag      (1=write, 0=read)
[30:28] function num  (0=CIA/CCCR, 1-7=functions)
[27]    RAW flag      (Read After Write — يرد بالقيمة الجديدة)
[25:9]  register address
[7:0]   data byte
```

ده اللي بيستخدمه `sdio_readb()` و `sdio_writeb()` من الـ API.

#### CMD53 — `SD_IO_RW_EXTENDED`
نقل **block أو stream** من البيانات. الـ argument format:
```
[31]    R/W flag
[30:28] function num
[27]    block mode    (1=block, 0=byte)
[26]    increment     (1=auto-increment address, 0=FIFO mode)
[25:9]  register address
[8:0]   count         (number of bytes or blocks)
```

ده اللي بيستخدمه `sdio_memcpy_fromio()` و `sdio_memcpy_toio()`.

**الفرق العملي بين increment=0 و increment=1**:
- `increment=1`: لنقل data من/إلى buffer في الكارت (memory-like)
- `increment=0`: لنقل data من/إلى FIFO register (زي RX FIFO في WiFi chip) — كل write/read بيبعت للنفس الـ address

---

### الـ R5 Response — إزاي تعرف اتصرف صح

الـ SDIO بيستخدم **R5 response** لـ CMD52 وCMD53:

```c
#define R5_COM_CRC_ERROR    (1 << 15)   /* CRC error في الـ command */
#define R5_ILLEGAL_COMMAND  (1 << 14)   /* command مش valid في الـ state ده */
#define R5_ERROR            (1 << 11)   /* general error */
#define R5_FUNCTION_NUMBER  (1 << 9)    /* function number غلط */
#define R5_OUT_OF_RANGE     (1 << 8)    /* address خارج نطاق الـ function */

#define R5_IO_CURRENT_STATE(x) ((x & 0x3000) >> 12)
/* states: 0=Dis, 1=CMD, 2=TRN, 3=RFU */
```

الكيرنل بيفحص `R5_STATUS(x)` بعد كل CMD52/CMD53 عشان يشوف في error ولا لا.

---

### الـ Speed Modes — من SDR12 لـ SDR104

الـ SDIO بيدعم طيف واسع من الـ speeds، الـ negotiation بيحصل عبر `SDIO_CCCR_SPEED`:

```
SDR12  → 25 MHz  → 12.5 MB/s  (default)
SDR25  → 50 MHz  → 25 MB/s    (High Speed)
SDR50  → 100 MHz → 50 MB/s    (UHS-I)
DDR50  → 50 MHz DDR → 50 MB/s (UHS-I)
SDR104 → 208 MHz → 104 MB/s   (UHS-I)
```

الكيرنل بيقرأ `SDIO_SPEED_SHS` من الـ CCCR، لو set يتفاوض على أعلى speed يدعمها الـ host controller.

---

### إيه اللي الـ SDIO Framework بيملكه وإيه اللي بيفوضه

```
┌─────────────────────────────────────────────────────────┐
│              SDIO Framework (Core) يمتلك:               │
│                                                         │
│  ✓ Card enumeration (قراءة CCCR, FBR, CIS)            │
│  ✓ Function device creation (sdio_func objects)         │
│  ✓ Bus matching (sdio_func ↔ sdio_driver)              │
│  ✓ IRQ multiplexing (من interrupt واحد → N handlers)   │
│  ✓ Host locking (sdio_claim_host / sdio_release_host)  │
│  ✓ Block size negotiation                               │
│  ✓ I/O API (sdio_readb, sdio_writeb, sdio_memcpy_*)   │
│  ✓ PM integration (sdio_set_host_pm_flags)             │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│           SDIO Function Driver يملك / يفوضه:            │
│                                                         │
│  ✓ معنى الـ registers الخاصة بالـ function              │
│  ✓ الـ firmware loading (لو WiFi مثلاً)                │
│  ✓ الـ upper-layer protocol (802.11, HCI, ...)          │
│  ✓ IRQ handling logic                                   │
│  ✓ Power management للـ function نفسها                  │
│  ✓ DMA buffer management                               │
└─────────────────────────────────────────────────────────┘
```

---

### مثال حقيقي: Driver لـ WiFi على SDIO

```c
/* مثال مبسط لـ SDIO WiFi driver */

static const struct sdio_device_id brcmf_sdmmc_ids[] = {
    SDIO_DEVICE(SDIO_VENDOR_ID_BROADCOM, SDIO_DEVICE_ID_BROADCOM_4329),
    { /* sentinel */ }
};

static int brcmf_sdio_probe(struct sdio_func *func,
                             const struct sdio_device_id *id)
{
    /* 1. خذ الـ host lock */
    sdio_claim_host(func);

    /* 2. فعّل الـ function في الكارت (يكتب في CCCR_IOEx) */
    sdio_enable_func(func);

    /* 3. اضبط الـ block size */
    sdio_set_block_size(func, 512);

    /* 4. سجّل IRQ handler */
    sdio_claim_irq(func, brcmf_sdio_isr);

    sdio_release_host(func);

    /* 5. load firmware وابدأ الـ WiFi stack */
    brcmf_fw_get_firmwares(func->card->host->parent, ...);

    return 0;
}

static void brcmf_sdio_isr(struct sdio_func *func)
{
    /* الـ SDIO core وصّل الـ interrupt لنا بعد ما قرأ CCCR_INTx */
    /* هنا نقرأ status registers الكارت ونعالج الـ packets */
    u8 status = sdio_readb(func, REG_INT_STATUS, &err);

    if (status & INT_RX_READY)
        brcmf_sdio_recv_pkt(func);
}

static struct sdio_driver brcmf_sdmmc_driver = {
    .name     = "brcmfmac",
    .id_table = brcmf_sdmmc_ids,
    .probe    = brcmf_sdio_probe,
    .remove   = brcmf_sdio_remove,
};

module_sdio_driver(brcmf_sdmmc_driver);
```

**الـ IRQ Flow** بالتفصيل:

```
Hardware SDIO interrupt (DAT1 line pulled low)
           │
           ▼
    MMC Host Controller
           │
           ▼
    mmc_signal_sdio_irq()   ← في interrupt context
           │
           ▼
    sdio_irq_thread()       ← kernel thread (process context)
           │
           ▼
    sdio_claim_host()
           │
           ▼
    قراءة SDIO_CCCR_INTx    ← بـ CMD52 عشان نعرف function# رفعت الـ interrupt
           │
           ▼
    استدعاء func->irq_handler(func)  ← handler الـ driver
           │
           ▼
    sdio_release_host()
```

---

### الـ Subsystems التانية اللي محتاج تعرفها

- **الـ MMC Core** (`drivers/mmc/core/`): المسؤول عن إرسال `mmc_request` للـ host controller. الـ SDIO core بيبني فوقه — الـ CMD52/CMD53 بيتحولوا لـ `mmc_command` وبيتبعتوا عبر `mmc_wait_for_cmd()`.

- **الـ Linux Device Model** (`include/linux/device.h`): الـ `sdio_func.dev` هو `struct device` عادي. الـ `sdio_bus_type` هو bus مسجل في الـ kernel، وبيستخدم `device_register()` / `driver_register()` زي أي bus تاني.

- **الـ CIS Parsing**: الـ **Card Information Structure** هي linked list من tuples (زي TLV) موجودة في address space الكارت. الـ SDIO core بيـ parse الـ CISTPL_MANFID و CISTPL_FUNCID من الـ CIS عشان يملأ `sdio_func.vendor/device/class`.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Enums والـ Config Options — Cheatsheet

#### SDIO Commands

| Macro | رقم | نوع | الـ Response | الوصف |
|---|---|---|---|---|
| `SD_IO_SEND_OP_COND` | 5 | bcr | R4 | بيتعرف على الـ card وبيجيب الـ OCR |
| `SD_IO_RW_DIRECT` | 52 | ac | R5 | قراءة/كتابة byte واحد مباشرة |
| `SD_IO_RW_EXTENDED` | 53 | adtc | R5 | نقل block أو stream من البيانات |

---

#### R4 Response Flags (بعد `SD_IO_SEND_OP_COND`)

| Flag | Bit | المعنى |
|---|---|---|
| `R4_18V_PRESENT` | 24 | الـ card بتدعم 1.8V |
| `R4_MEMORY_PRESENT` | 27 | في memory function جنب الـ SDIO |

---

#### R5 Response Flags (بعد CMD52/CMD53)

| Flag | Bits | النوع | Clear | المعنى |
|---|---|---|---|---|
| `R5_COM_CRC_ERROR` | 15 | er | b | CRC error في الـ command |
| `R5_ILLEGAL_COMMAND` | 14 | er | b | command غير مسموح بيه في الـ state ده |
| `R5_ERROR` | 11 | erx | c | general error |
| `R5_FUNCTION_NUMBER` | 9 | er | c | function number غلط |
| `R5_OUT_OF_RANGE` | 8 | er | c | العنوان خارج النطاق |
| `R5_STATUS(x)` | mask 0xCB00 | — | — | بيعزل بيتات الـ status |
| `R5_IO_CURRENT_STATE(x)` | [13:12] | s | b | الـ state الحالي للـ card |

---

#### CCCR Registers — Card Common Control Registers

| Register | Offset | المعنى |
|---|---|---|
| `SDIO_CCCR_CCCR` | 0x00 | إصدار الـ CCCR/FBR |
| `SDIO_CCCR_SD` | 0x01 | إصدار الـ SD Physical Spec |
| `SDIO_CCCR_IOEx` | 0x02 | تفعيل الـ I/O functions |
| `SDIO_CCCR_IORx` | 0x03 | جاهزية الـ I/O functions |
| `SDIO_CCCR_IENx` | 0x04 | تفعيل الـ interrupts |
| `SDIO_CCCR_INTx` | 0x05 | الـ pending interrupts |
| `SDIO_CCCR_ABORT` | 0x06 | إلغاء function / reset الـ card |
| `SDIO_CCCR_IF` | 0x07 | إعدادات الـ bus interface |
| `SDIO_CCCR_CAPS` | 0x08 | capabilities الـ card |
| `SDIO_CCCR_CIS` | 0x09 | مؤشر الـ Common CIS (3 bytes) |
| `SDIO_CCCR_SUSPEND` | 0x0C | Suspend control (لو SBS=1) |
| `SDIO_CCCR_SELx` | 0x0D | Function select for suspend |
| `SDIO_CCCR_EXECx` | 0x0E | Functions executing |
| `SDIO_CCCR_READYx` | 0x0F | Functions ready after resume |
| `SDIO_CCCR_BLKSIZE` | 0x10 | حجم الـ block للـ function 0 |
| `SDIO_CCCR_POWER` | 0x12 | Power control |
| `SDIO_CCCR_SPEED` | 0x13 | Speed mode selection |
| `SDIO_CCCR_UHS` | 0x14 | UHS-I support |
| `SDIO_CCCR_DRIVE_STRENGTH` | 0x15 | Drive strength |
| `SDIO_CCCR_INTERRUPT_EXT` | 0x16 | Extended interrupt support |

---

#### Bus Width Flags (`SDIO_CCCR_IF`)

| Flag | القيمة | المعنى |
|---|---|---|
| `SDIO_BUS_WIDTH_1BIT` | 0x00 | 1-bit bus |
| `SDIO_BUS_WIDTH_4BIT` | 0x02 | 4-bit bus |
| `SDIO_BUS_ECSI` | 0x20 | Enable continuous SPI interrupt |
| `SDIO_BUS_SCSI` | 0x40 | Support continuous SPI interrupt |
| `SDIO_BUS_ASYNC_INT` | 0x20 | Async interrupt support |
| `SDIO_BUS_CD_DISABLE` | 0x80 | تعطيل الـ pull-up على DAT3 |

---

#### Capabilities Flags (`SDIO_CCCR_CAPS`)

| Flag | Bit | المعنى |
|---|---|---|
| `SDIO_CCCR_CAP_SDC` | 0x01 | ممكن يبعت CMD52 وسط data transfer |
| `SDIO_CCCR_CAP_SMB` | 0x02 | بيدعم multi-block transfers |
| `SDIO_CCCR_CAP_SRW` | 0x04 | بيدعم read-wait protocol |
| `SDIO_CCCR_CAP_SBS` | 0x08 | بيدعم suspend/resume |
| `SDIO_CCCR_CAP_S4MI` | 0x10 | interrupt أثناء 4-bit CMD53 |
| `SDIO_CCCR_CAP_E4MI` | 0x20 | تفعيل interrupts أثناء 4-bit CMD53 |
| `SDIO_CCCR_CAP_LSC` | 0x40 | Low speed card |
| `SDIO_CCCR_CAP_4BLS` | 0x80 | 4-bit low speed card |

---

#### Speed Mode Flags (`SDIO_CCCR_SPEED`)

| Flag | القيمة | المعنى |
|---|---|---|
| `SDIO_SPEED_SHS` | 0x01 | Supports High-Speed |
| `SDIO_SPEED_SDR12` | bits[3:1]=0 | Default speed 12.5 MB/s |
| `SDIO_SPEED_SDR25` | bits[3:1]=1 | High-Speed 25 MB/s |
| `SDIO_SPEED_SDR50` | bits[3:1]=2 | UHS 50 MB/s |
| `SDIO_SPEED_SDR104` | bits[3:1]=3 | UHS 104 MB/s |
| `SDIO_SPEED_DDR50` | bits[3:1]=4 | DDR 50 MB/s |
| `SDIO_SPEED_EHS` | = SDR25 | Enable High-Speed (alias) |

---

#### UHS Support Flags (`SDIO_CCCR_UHS`)

| Flag | Bit | المعنى |
|---|---|---|
| `SDIO_UHS_SDR50` | 0x01 | بيدعم SDR50 |
| `SDIO_UHS_SDR104` | 0x02 | بيدعم SDR104 |
| `SDIO_UHS_DDR50` | 0x04 | بيدعم DDR50 |

---

#### Drive Strength Types (`SDIO_CCCR_DRIVE_STRENGTH`)

| Flag | المعنى |
|---|---|
| `SDIO_DRIVE_SDTA` | بيدعم Type A |
| `SDIO_DRIVE_SDTC` | بيدعم Type C |
| `SDIO_DRIVE_SDTD` | بيدعم Type D |
| `SDIO_DTSx_SET_TYPE_B` | اختيار Type B (default) |
| `SDIO_DTSx_SET_TYPE_A` | اختيار Type A |
| `SDIO_DTSx_SET_TYPE_C` | اختيار Type C |
| `SDIO_DTSx_SET_TYPE_D` | اختيار Type D |

---

#### Extended Interrupt Flags (`SDIO_CCCR_INTERRUPT_EXT`)

| Flag | Bit | المعنى |
|---|---|---|
| `SDIO_INTERRUPT_EXT_SAI` | 0 | Supports Async Interrupt |
| `SDIO_INTERRUPT_EXT_EAI` | 1 | Enable Async Interrupt |

---

#### FBR — Function Basic Registers

| Register | Offset من base | المعنى |
|---|---|---|
| `SDIO_FBR_BASE(f)` | f × 0x100 | بداية الـ FBR للـ function رقم f |
| `SDIO_FBR_STD_IF` | 0x00 | Standard Interface Code |
| `SDIO_FBR_STD_IF_EXT` | 0x01 | Extended Interface Code |
| `SDIO_FBR_POWER` | 0x02 | Power management |
| `SDIO_FBR_CIS` | 0x09 | CIS pointer للـ function (3 bytes) |
| `SDIO_FBR_CSA` | 0x0C | Code Storage Area pointer |
| `SDIO_FBR_CSA_DATA` | 0x0F | بوابة قراءة/كتابة الـ CSA |
| `SDIO_FBR_BLKSIZE` | 0x10 | حجم الـ block للـ function (2 bytes) |

| FBR Flag | المعنى |
|---|---|
| `SDIO_FBR_SUPPORTS_CSA` | الـ function بتدعم Code Storage Area |
| `SDIO_FBR_ENABLE_CSA` | تفعيل الوصول للـ CSA |
| `SDIO_FBR_POWER_SPS` | Supports Power Selection |
| `SDIO_FBR_POWER_EPS` | Enable low Power Selection |

---

#### CCCR/SDIO/SD Revision Values

| Macro | القيمة | الإصدار |
|---|---|---|
| `SDIO_CCCR_REV_1_00` | 0 | CCCR v1.00 |
| `SDIO_CCCR_REV_3_00` | 3 | CCCR v3.00 |
| `SDIO_SDIO_REV_1_00` | 0 | SDIO Spec v1.00 |
| `SDIO_SDIO_REV_3_00` | 4 | SDIO Spec v3.00 |
| `SDIO_SD_REV_1_01` | 0 | SD Physical v1.01 |
| `SDIO_SD_REV_3_00` | 3 | SD Physical v3.00 |

---

### 1. الـ Structs المهمة

#### `struct sdio_func` — الـ function device

**الغرض:** بتمثل كل function داخل الـ SDIO card. الـ SDIO card ممكن يكون فيها لغاية 7 functions (مثلاً: WiFi + Bluetooth في card واحدة).

| Field | النوع | الوصف |
|---|---|---|
| `card` | `struct mmc_card *` | الـ card الأم اللي الـ function ده جزء منها |
| `dev` | `struct device` | الـ device object للـ kernel device model |
| `irq_handler` | `sdio_irq_handler_t *` | الـ callback لما الـ interrupt بييجي |
| `num` | `unsigned int` | رقم الـ function (1–7) |
| `class` | `unsigned char` | Standard Interface Class (مثلاً: UART, BT, ...) |
| `vendor` | `unsigned short` | الـ vendor ID |
| `device` | `unsigned short` | الـ device ID |
| `max_blksize` | `unsigned` | أقصى حجم block مدعوم |
| `cur_blksize` | `unsigned` | الحجم الحالي للـ block |
| `enable_timeout` | `unsigned` | أقصى وقت (ms) لتفعيل الـ function |
| `state` | `unsigned int` | حالة الـ function (موجود في sysfs؟) |
| `tmpbuf` | `u8 *` | buffer مؤقت DMA-able للعمليات الصغيرة |
| `major_rev` | `u8` | رقم الـ major revision |
| `minor_rev` | `u8` | رقم الـ minor revision |
| `num_info` | `unsigned` | عدد الـ info strings |
| `info` | `const char **` | مصفوفة strings من الـ CIS |
| `tuples` | `struct sdio_func_tuple *` | قائمة الـ CIS tuples غير المعروفة للـ core |

---

#### `struct sdio_func_tuple` — CIS Tuple

**الغرض:** بتحفظ الـ tuples اللي الـ core ميعرفهاش من الـ Card Information Structure. كل function عندها linked list من الـ tuples دي.

| Field | النوع | الوصف |
|---|---|---|
| `next` | `struct sdio_func_tuple *` | التالي في الـ linked list |
| `code` | `unsigned char` | كود الـ tuple |
| `size` | `unsigned char` | حجم الـ data |
| `data[]` | `unsigned char[]` | الـ payload (flexible array) |

---

#### `struct sdio_driver` — الـ driver

**الغرض:** بيمثل الـ SDIO driver اللي بيدعم functions معينة. بيتسجل في الـ kernel bus ويـ probe الـ functions المناسبة.

| Field | النوع | الوصف |
|---|---|---|
| `name` | `char *` | اسم الـ driver |
| `id_table` | `const struct sdio_device_id *` | جدول الـ IDs المدعومة |
| `probe` | `int (*)(func, id)` | بيتستدعى لما الـ function يتطابق |
| `remove` | `void (*)(func)` | بيتستدعى لما الـ function يتفصل |
| `shutdown` | `void (*)(func)` | بيتستدعى عند الـ shutdown |
| `drv` | `struct device_driver` | الـ generic driver للـ bus model |

---

#### `struct mmc_card` (مرجع خارجي)

**الغرض:** بتمثل الـ card كاملة (hardware level). الـ `sdio_func` بيشير إليها. فيها معلومات الـ host والـ CID والـ capabilities.

---

### 2. مخطط علاقات الـ Structs

```
+------------------+
|   mmc_host       |  ← بيتحكم في الـ bus كله
|  (host controller)|
+--------+---------+
         |
         | hosts
         ▼
+------------------+         +----------------------+
|   mmc_card       |-------->|  mmc_request         |
|  +-card info     |  sends  |  +-cmd (mmc_command)  |
|  +-sdio_funcs[]  |         |  +-data (mmc_data)    |
+--------+---------+         +----------------------+
         |
         | owns (1..7)
         ▼
+------------------+       +--------------------+
|   sdio_func      |<------| sdio_driver        |
|  +-card *        | probe |  +-id_table         |
|  +-dev           |       |  +-probe()          |
|  +-irq_handler   |       |  +-remove()         |
|  +-tuples *------+--+    |  +-drv              |
|  +-num           |  |    +--------------------+
+------------------+  |
                       ▼
               +------------------+
               | sdio_func_tuple  |
               |  +-code          |
               |  +-size          |
               |  +-data[]        |
               |  +-next *--------+--> (next tuple ...)
               +------------------+
```

---

### 3. مخطط دورة الحياة — Lifecycle Diagram

```
[Power On / Card Insert]
         │
         ▼
  SD_IO_SEND_OP_COND (CMD5)
  → الـ card بترد بـ R4
  → بيتحدد: عدد الـ functions، دعم 1.8V، وجود memory
         │
         ▼
  قراءة CCCR (عبر CMD52)
  → بيتحدد: speed، bus width، capabilities
         │
         ▼
  قراءة FBR لكل function
  → vendor ID، device ID، class، CIS pointer
         │
         ▼
  قراءة CIS (Card Information Structure)
  → بيبني linked list من sdio_func_tuple
         │
         ▼
  sdio_func objects يتبنوا
  → mmc_card->sdio_func[i] = allocated sdio_func
         │
         ▼
  sdio_register_driver() ← من الـ driver module
         │
         ▼
  bus matching: sdio_func.vendor+device vs sdio_driver.id_table
         │
         ▼
  driver->probe(func, id) يتستدعى
  → sdio_enable_func()    ← CCCR_IOEx bit set
  → sdio_set_block_size() ← FBR_BLKSIZE write
  → sdio_claim_irq()      ← CCCR_IENx bit set + handler registration
         │
         ▼
  [Normal Operation]
  sdio_readb/writeb/memcpy_fromio/memcpy_toio
  IRQ → sdio_irq_handler_t() يتستدعى
         │
         ▼
  [Teardown]
  sdio_release_irq()
  sdio_disable_func()
  driver->remove(func)
  sdio_unregister_driver()
```

---

### 4. مخطط Call Flow

#### CMD52 — SD_IO_RW_DIRECT

```
driver calls sdio_readb(func, addr, &err)
  → sdio_io_rw_direct(func->card, 0, func->num, addr, 0, &val)
    → mmc_io_rw_direct_host(card->host, ...)
      → mmc_command.opcode = SD_IO_RW_DIRECT (52)
        → mmc_command.arg = [R/W|func_num|RAW|addr|data]
          → mmc_wait_for_cmd(host, cmd, retries)
            → host->ops->request(host, mrq)
              → hardware: CMD52 على الـ bus
                ← R5 response يرجع
              ← R5_STATUS() check
            ← error أو القيمة المقروءة
          ← *err_ret = error
        ← return val (u8)
```

#### CMD53 — SD_IO_RW_EXTENDED (Block Transfer)

```
driver calls sdio_memcpy_fromio(func, dst, addr, count)
  → sdio_io_rw_ext_helper(func, 0, addr, 1, dst, count)
    → [لو count > cur_blksize]: يقسم على blocks
    → sdio_io_rw_extended(func->card, 0, func->num, addr, 1, buf, blocks, blksz)
      → mmc_command.opcode = SD_IO_RW_EXTENDED (53)
        → mmc_command.arg = [R/W|func_num|block_mode|incr_addr|addr|count]
          → mmc_data: sg = buffer, blocks, blksz
            → mmc_wait_for_req(host, mrq)
              → host->ops->request(host, mrq)
                → hardware: CMD53 + data على الـ bus
                  ← data transfer + R5
```

#### IRQ Flow

```
[Hardware asserts interrupt line]
  → host controller interrupt handler
    → mmc_signal_sdio_irq(host)
      → schedule sdio_irq_thread
        → sdio_process_irqs(card)
          → CCCR_INTx بيتقرأ (CMD52)
            → لكل function فيها pending interrupt:
              func->irq_handler(func)
                → driver's handler يتشغل
                  → sdio_readb/writeb لقراءة الـ status وتصفية الـ interrupt
```

#### Driver Registration Flow

```
module_sdio_driver(my_driver)
  → sdio_register_driver(&my_driver, THIS_MODULE)
    → __sdio_register_driver(drv, module)
      → drv->drv.bus = &sdio_bus_type
        → driver_register(&drv->drv)
          → bus_for_each_dev(sdio_bus, sdio_bus_match)
            → لكل sdio_func موجود:
              sdio_bus_match(func->dev, drv->drv)
                → بيقارن func->class, vendor, device مع id_table
                  ← لو match: drv->probe(func, id) يتستدعى
```

---

### 5. استراتيجية الـ Locking

الـ `sdio.h` نفسه ما بيعرفش locks مباشرة، لكن كل الـ operations المبنية عليه بتتبع نظام واضح:

#### القفل الأساسي: `mmc_host->lock` (spinlock)

**بيحمي:** الـ host state الداخلي، الـ bus transactions الجارية.

#### الـ Claim Mechanism: `sdio_claim_host` / `sdio_release_host`

```c
sdio_claim_host(func);
/* critical section: CMD52/CMD53 */
val = sdio_readb(func, REG_STATUS, &err);
sdio_writeb(func, val | BIT(0), REG_CTRL, &err);
sdio_release_host(func);
```

**بيحمي:** كل الـ SDIO I/O operations. لازم يتعمل claim قبل أي CMD52/CMD53.

#### ترتيب الـ Locks (Lock Ordering)

```
1. sdio_claim_host()          ← أعلى مستوى (sleeping lock)
   └─ 2. mmc_host->lock       ← spinlock داخلي
         └─ 3. host HW regs   ← الـ register access
```

**قاعدة:** محظور تمامًا تعمل `sdio_claim_host` وأنت شايل أي lock آخر في الـ driver — هيحصل deadlock.

#### الـ IRQ Handler

الـ `sdio_irq_handler_t` بيتشتغل في سياق خاص (`sdio_irq_thread` — kernel thread)، مش في hard IRQ context. يعني:

- مسموح بـ sleeping
- مسموح بـ `sdio_claim_host` داخل الـ handler
- محظور الوصول لأي resource بيحتاج hard IRQ disabling

#### حماية الـ `sdio_func.state`

| Field | الـ Lock المسؤول |
|---|---|
| `state` (SDIO_STATE_PRESENT) | `device_lock(&func->dev)` من الـ device model |
| `irq_handler` | `mmc_host->lock` (spinlock) أثناء الـ claim/release |
| `cur_blksize` | `sdio_claim_host` (الـ caller مسؤول) |
| `tuples` | read-only بعد الـ initialization، مفيش لاك مطلوب |

#### الـ Suspend/Resume والـ CCCR_SUSPEND

لما `SDIO_CCCR_CAP_SBS` يكون set، الـ suspend/resume بيستخدم:
- `SDIO_CCCR_SUSPEND` / `SDIO_CCCR_SELx` / `SDIO_CCCR_EXECx` / `SDIO_CCCR_READYx`
- كل العمليات دي تحت `sdio_claim_host` + الـ pm_mutex بتاع الـ host
## Phase 4: شرح الـ Functions

> **ملاحظة:** الـ `include/linux/mmc/sdio.h` نفسه ما فيهوش functions — هو header خالص للـ macros بتعرّف commands وregisters الـ SDIO protocol (CCCR, FBR, R4/R5 response bits). الـ functions الفعلية للـ SDIO driver API موجودة في `include/linux/mmc/sdio_func.h` اللي بيعمل `#include` على `sdio.h` بشكل غير مباشر، وهو الـ public API الكامل للـ SDIO subsystem في الـ kernel. الشرح دا بيغطي الاتنين مع بعض.

---

### ملخص الـ Macros المهمة في `sdio.h`

#### SDIO Commands

| Macro | القيمة | الوصف |
|---|---|---|
| `SD_IO_SEND_OP_COND` | 5 | CMD5 — negotiation OCR أثناء initialization |
| `SD_IO_RW_DIRECT` | 52 | CMD52 — قراءة/كتابة single byte على أي register |
| `SD_IO_RW_EXTENDED` | 53 | CMD53 — نقل block/multi-byte data |

#### R4 Response Bits

| Macro | Bit | المعنى |
|---|---|---|
| `R4_18V_PRESENT` | 24 | الكارت بيدعم 1.8V switching |
| `R4_MEMORY_PRESENT` | 27 | الكارت فيه memory function (combo card) |

#### R5 Status Bits (CMD52/CMD53 Response)

| Macro | Bit | النوع | المعنى |
|---|---|---|---|
| `R5_COM_CRC_ERROR` | 15 | error | CRC error في الـ command |
| `R5_ILLEGAL_COMMAND` | 14 | error | command غير مسموح بيه في الـ state الحالية |
| `R5_ERROR` | 11 | error | general error |
| `R5_FUNCTION_NUMBER` | 9 | error | function number غير صالح |
| `R5_OUT_OF_RANGE` | 8 | error | address خارج نطاق الـ function |
| `R5_STATUS(x)` | mask | helper | بيعمل mask على bits الـ status المهمة (0xCB00) |
| `R5_IO_CURRENT_STATE(x)` | 13:12 | status | الـ state الحالية للكارت (Dis/Cmd/Tran/RFU) |

#### CCCR Registers (Card Common Control Registers)

| Macro | Offset | الوصف |
|---|---|---|
| `SDIO_CCCR_CCCR` | 0x00 | CCCR/SDIO spec version |
| `SDIO_CCCR_SD` | 0x01 | SD Physical Spec version |
| `SDIO_CCCR_IOEx` | 0x02 | I/O Enable register |
| `SDIO_CCCR_IORx` | 0x03 | I/O Ready register |
| `SDIO_CCCR_IENx` | 0x04 | Interrupt Enable |
| `SDIO_CCCR_INTx` | 0x05 | Interrupt Pending |
| `SDIO_CCCR_ABORT` | 0x06 | Function Abort / Card Reset |
| `SDIO_CCCR_IF` | 0x07 | Bus Interface Control |
| `SDIO_CCCR_CAPS` | 0x08 | Card Capabilities |
| `SDIO_CCCR_CIS` | 0x09 | Common CIS Pointer (3 bytes) |
| `SDIO_CCCR_BLKSIZE` | 0x10 | Block Size (function 0) |
| `SDIO_CCCR_POWER` | 0x12 | Power Control |
| `SDIO_CCCR_SPEED` | 0x13 | Bus Speed Select |
| `SDIO_CCCR_UHS` | 0x14 | UHS-I Support |
| `SDIO_CCCR_DRIVE_STRENGTH` | 0x15 | Drive Strength |
| `SDIO_CCCR_INTERRUPT_EXT` | 0x16 | Interrupt Extension (async IRQ) |

#### FBR Registers (Function Basic Registers)

| Macro | Offset نسبي | الوصف |
|---|---|---|
| `SDIO_FBR_BASE(f)` | `f * 0x100` | base address للـ function `f` |
| `SDIO_FBR_STD_IF` | 0x00 | Standard Interface Code |
| `SDIO_FBR_POWER` | 0x02 | Function Power Selection |
| `SDIO_FBR_CIS` | 0x09 | Function CIS Pointer (3 bytes) |
| `SDIO_FBR_CSA` | 0x0C | Code Storage Area Pointer |
| `SDIO_FBR_BLKSIZE` | 0x10 | Function Block Size (2 bytes) |

---

### ملخص الـ Functions في `sdio_func.h`

#### جدول شامل (Cheatsheet)

| Function | الفئة | الوصف المختصر |
|---|---|---|
| `sdio_register_driver()` | Registration | تسجيل SDIO driver |
| `sdio_unregister_driver()` | Registration | إلغاء تسجيل SDIO driver |
| `sdio_claim_host()` | Locking | أخذ ownership الـ host bus |
| `sdio_release_host()` | Locking | تحرير ownership الـ host bus |
| `sdio_enable_func()` | Function Control | تفعيل SDIO function |
| `sdio_disable_func()` | Function Control | تعطيل SDIO function |
| `sdio_set_block_size()` | Function Control | تعيين block size للـ function |
| `sdio_claim_irq()` | IRQ | تسجيل IRQ handler |
| `sdio_release_irq()` | IRQ | إلغاء تسجيل IRQ handler |
| `sdio_align_size()` | Helper | حساب حجم transfer متوافق |
| `sdio_readb()` | I/O | قراءة byte واحد |
| `sdio_readw()` | I/O | قراءة word (16-bit) |
| `sdio_readl()` | I/O | قراءة dword (32-bit) |
| `sdio_writeb()` | I/O | كتابة byte واحد |
| `sdio_writew()` | I/O | كتابة word (16-bit) |
| `sdio_writel()` | I/O | كتابة dword (32-bit) |
| `sdio_writeb_readb()` | I/O | CMD52 RAW — كتابة وقراءة atomic |
| `sdio_memcpy_fromio()` | Bulk I/O | قراءة block مع address increment |
| `sdio_readsb()` | Bulk I/O | قراءة block من fixed address (FIFO) |
| `sdio_memcpy_toio()` | Bulk I/O | كتابة block مع address increment |
| `sdio_writesb()` | Bulk I/O | كتابة block على fixed address (FIFO) |
| `sdio_f0_readb()` | Function 0 | قراءة من CCCR space |
| `sdio_f0_writeb()` | Function 0 | كتابة في CCCR space |
| `sdio_get_host_pm_caps()` | PM | استعلام PM capabilities |
| `sdio_set_host_pm_flags()` | PM | تعيين PM flags |
| `sdio_retune_crc_disable()` | Retuning | تعطيل CRC retuning trigger |
| `sdio_retune_crc_enable()` | Retuning | تمكين CRC retuning trigger |
| `sdio_retune_hold_now()` | Retuning | تجميد retuning فوراً |
| `sdio_retune_release()` | Retuning | تحرير retuning hold |

---

### فئة 1: Registration & Discovery

**الغرض:** تسجيل SDIO driver في الـ kernel driver model، وربطه بالـ `struct bus_type` الخاصة بالـ SDIO bus.

---

#### `sdio_register_driver` / `__sdio_register_driver`

```c
#define sdio_register_driver(drv) \
    __sdio_register_driver(drv, THIS_MODULE)

extern int __sdio_register_driver(struct sdio_driver *drv, struct module *owner);
```

الـ macro بتضيف `THIS_MODULE` أوتوماتيك عشان reference counting الـ module صح. الدالة بتسجّل الـ `sdio_driver` في الـ SDIO bus وبتعمل probe على الأجهزة الموجودة بالفعل اللي تتطابق مع الـ `id_table`.

**Parameters:**
- `drv` — pointer للـ `struct sdio_driver` المعرّف في الـ driver

**Return:** `0` عند النجاح، أو negative error code

**Key details:**
- بتشتغل في module init context
- الـ `id_table` لازم يكون `NULL`-terminated
- بتستدعي `driver_register()` داخلياً تحت الـ SDIO bus
- الـ `probe` callback بيتم استدعاؤه per matched function، مش per card

---

#### `sdio_unregister_driver`

```c
extern void sdio_unregister_driver(struct sdio_driver *drv);
```

بتعمل `remove` callback على كل الـ functions المرتبطة بالـ driver دا، وبتشيله من الـ SDIO bus.

**Parameters:**
- `drv` — نفس الـ pointer اللي اتسجّل بيه

**Return:** void

**Key details:**
- لازم تتعمل في module exit context
- بتستنى الـ `remove` callbacks تخلص قبل ما ترجع (synchronous)

---

#### `module_sdio_driver`

```c
#define module_sdio_driver(__sdio_driver) \
    module_driver(__sdio_driver, sdio_register_driver, \
                  sdio_unregister_driver)
```

Boilerplate eliminator — بتولّد `module_init` و`module_exit` تلقائياً للـ drivers اللي مش محتاجة initialization خاصة.

---

#### `struct sdio_driver` — الهيكل المركزي للـ Driver

```c
struct sdio_driver {
    char *name;
    const struct sdio_device_id *id_table;    /* matching table */

    int  (*probe)(struct sdio_func *, const struct sdio_device_id *);
    void (*remove)(struct sdio_func *);
    void (*shutdown)(struct sdio_func *);

    struct device_driver drv;    /* embedded kernel driver object */
};
```

| Field | الوصف |
|---|---|
| `name` | اسم الـ driver (يظهر في sysfs) |
| `id_table` | جدول matching — vendor/device/class |
| `probe` | بيتعمل call لما الكارت يتضاف ويتطابق |
| `remove` | بيتعمل call لما الكارت يتشال أو الـ driver يُـunregister |
| `shutdown` | بيتعمل call عند system shutdown |

---

#### `SDIO_DEVICE` / `SDIO_DEVICE_CLASS` macros

```c
#define SDIO_DEVICE(vend, dev) \
    .class = SDIO_ANY_ID, .vendor = (vend), .device = (dev)

#define SDIO_DEVICE_CLASS(dev_class) \
    .class = (dev_class), \
    .vendor = SDIO_ANY_ID, .device = SDIO_ANY_ID
```

بتُستخدم في تعريف `id_table` عشان تحدد الأجهزة اللي الـ driver بيدعمها. الـ `SDIO_DEVICE` بتتطابق على vendor+device ID، والـ `SDIO_DEVICE_CLASS` بتتطابق على standard interface class بس.

---

### فئة 2: Host Locking

**الغرض:** الـ SDIO bus مش thread-safe — كل access لازم يتم وإنت حاجز الـ host. ده mutex-like mechanism على مستوى الـ `mmc_host`.

---

#### `sdio_claim_host`

```c
extern void sdio_claim_host(struct sdio_func *func);
```

بتحجز الـ `mmc_host` exclusive لأي I/O operations جاية. الـ implementation بتستدعي `mmc_claim_host()` اللي بتعمل wait على `host->claim_wait` wait queue لو الـ host مشغول.

**Parameters:**
- `func` — الـ SDIO function اللي بيطلب الحجز؛ بتوصّل للـ `func->card->host`

**Return:** void

**Key details:**
- **Sleeping call** — ممنوع استدعاؤها من atomic context أو interrupt context
- **Non-recursive** — لو نفس الـ thread حجزه تاني مرة ممكن يحصل deadlock
- كل دوال الـ I/O تفترض إن الـ host محجوز مسبقاً (ما عدا حالات معينة)
- الـ host بيتحجز على مستوى الـ `mmc_host`، مش الـ function

---

#### `sdio_release_host`

```c
extern void sdio_release_host(struct sdio_func *func);
```

بتحرر الـ host وتوقّظ أي thread تاني waiting عليه. لازم تتعمل دايماً بعد `sdio_claim_host()`.

**Pattern المعتاد:**

```c
sdio_claim_host(func);

/* كل الـ I/O operations هنا */
sdio_writeb(func, value, addr, &err);
sdio_readb(func, addr, &err);

sdio_release_host(func);
```

---

### فئة 3: Function Enable/Disable

**الغرض:** التحكم في power state الـ SDIO function عن طريق الـ `SDIO_CCCR_IOEx` register.

---

#### `sdio_enable_func`

```c
extern int sdio_enable_func(struct sdio_func *func);
```

بتكتب في bit الـ function في `SDIO_CCCR_IOEx` عشان تشغّله، وبتستنى confirmation في `SDIO_CCCR_IORx` خلال timeout محدد بالـ `func->enable_timeout` (افتراضياً 100ms).

**Parameters:**
- `func` — الـ function المراد تفعيلها

**Return:** `0` نجاح، `-ETIMEDOUT` لو الكارت ما ردّش في الوقت المحدد، أو negative error code من CMD52

**Key details:**
- **يتطلب host claim** — لازم `sdio_claim_host()` قبله
- بتشتغل في probe context عادةً
- لو الـ function مش enabled، أي I/O عليها هيرجع error

**Pseudocode:**
```
sdio_enable_func(func):
    read SDIO_CCCR_IOEx via CMD52
    set bit (func->num) in the value
    write back to SDIO_CCCR_IOEx via CMD52

    timeout = func->enable_timeout (msec)
    loop:
        read SDIO_CCCR_IORx via CMD52
        if bit (func->num) is set → return 0
        sleep 1ms, decrement timeout
        if timeout == 0 → return -ETIMEDOUT
```

---

#### `sdio_disable_func`

```c
extern int sdio_disable_func(struct sdio_func *func);
```

بتعمل clear على bit الـ function في `SDIO_CCCR_IOEx` عشان تطفيها وتوقف power consumption بتاعها.

**Parameters:**
- `func` — الـ function المراد تعطيلها

**Return:** `0` نجاح، أو negative error code

**Key details:**
- **يتطلب host claim**
- بتتعمل في remove context عادةً قبل `sdio_release_host()`
- ما بتستناش confirmation (عكس enable)

---

#### `sdio_set_block_size`

```c
extern int sdio_set_block_size(struct sdio_func *func, unsigned blksz);
```

بتحدد الـ block size اللي بيتستخدم مع CMD53 block mode transfers. بتكتب في `SDIO_FBR_BLKSIZE` الخاص بالـ function وبتحدّث `func->cur_blksize`.

**Parameters:**
- `func` — الـ function
- `blksz` — الحجم المطلوب؛ لو `0` بيستخدم `func->max_blksize`

**Return:** `0` نجاح، `-EINVAL` لو الحجم أكبر من `func->max_blksize`

**Key details:**
- **يتطلب host claim**
- الـ block size بيأثر على performance بشكل كبير — قيمة كبيرة أكثر كفاءة للنقل الكبير
- بعض الـ hosts عندهم max block size خاص بيهم

---

### فئة 4: IRQ Management

**الغرض:** إدارة الـ in-band SDIO interrupts اللي بتيجي خلال الـ DAT1 line أثناء idle state.

---

#### `sdio_claim_irq`

```c
extern int sdio_claim_irq(struct sdio_func *func,
                           sdio_irq_handler_t *handler);
```

بتسجّل IRQ handler للـ function وبتفعّل interrupt في `SDIO_CCCR_IENx`. الـ SDIO core بيشغّل thread خاص (sdio_irq_thread) بيشوف `SDIO_CCCR_INTx` ويستدعي الـ handlers المناسبة.

**Parameters:**
- `func` — الـ function اللي محتاجة interrupt
- `handler` — callback: `void handler(struct sdio_func *func)`

**Return:** `0` نجاح، أو negative error code

**Key details:**
- **يتطلب host claim**
- الـ handler بيتشغّل في context الـ irq thread (ممكن يعمل sleep)
- بيعمل set على bit الـ master interrupt enable (bit 0 في `SDIO_CCCR_IENx`) أوتوماتيك
- الـ handler لازم يتعامل مع spurious interrupts

**الـ IRQ flow:**

```
DAT1 asserted (low) by card
    → host detects interrupt
    → sdio_irq_thread wakes up
    → reads SDIO_CCCR_INTx (CMD52)
    → for each set bit N:
        call func[N]->irq_handler(func[N])
```

---

#### `sdio_release_irq`

```c
extern int sdio_release_irq(struct sdio_func *func);
```

بتعمل clear على interrupt enable bit الخاص بالـ function وبتشيل الـ handler. لو مفيش functions تانية عندها interrupt، بتوقف الـ irq thread.

**Parameters:**
- `func` — الـ function

**Return:** `0` نجاح، أو negative error code

**Key details:**
- **يتطلب host claim**
- بتتعمل في remove context

---

### فئة 5: I/O Helpers

---

#### `sdio_align_size`

```c
extern unsigned int sdio_align_size(struct sdio_func *func,
                                     unsigned int sz);
```

بتحسب أصغر size ≥ `sz` يكون valid للـ CMD53 transfer بناءً على الـ host constraints وبناءً على الـ block size الحالي للـ function.

**Parameters:**
- `func` — الـ function
- `sz` — الحجم المطلوب

**Return:** الحجم المعدَّل المتوافق

**Key details:**
- الـ SDIO core لما بيستخدم block mode، الـ transfer size لازم يكون multiple للـ block size
- بيراعي الـ `host->max_blk_size` و`func->cur_blksize` مع بعض
- **مهم جداً** لأي driver بيعمل DMA transfers عشان ما يحصلش alignment fault

---

### فئة 6: Single-Byte I/O (CMD52 Based)

**الغرض:** قراءة/كتابة registers فردية في space الـ function. كل call = CMD52 واحد.

---

#### `sdio_readb`

```c
extern u8 sdio_readb(struct sdio_func *func,
                     unsigned int addr, int *err_ret);
```

بتقرأ byte واحد من register address `addr` في space الـ function. بتبعت CMD52 مع R/W=0 وfunction number=`func->num`.

**Parameters:**
- `func` — الـ function
- `addr` — الـ 17-bit register address (bits [25:9] في CMD52)
- `err_ret` — pointer لـ error code؛ ممكن `NULL` لو مش محتاجه

**Return:** القيمة المقروءة، أو `0xFF` عند الـ error (وبيسيب error code في `*err_ret`)

**Key details:**
- **يتطلب host claim**
- الداخل بيستخدم `func->card->host` لإرسال CMD52
- الـ `tmpbuf` في الـ `sdio_func` بيتستخدم كـ DMA-safe scratch buffer

---

#### `sdio_readw`

```c
extern u16 sdio_readw(struct sdio_func *func,
                      unsigned int addr, int *err_ret);
```

بتقرأ 16-bit value عن طريق إرسال CMD52 مرتين (little-endian): أول call على `addr` والتاني على `addr+1`.

**Parameters:** نفس `sdio_readb` بس الناتج `u16`

**Return:** الـ 16-bit value أو `0xFFFF` عند الـ error

---

#### `sdio_readl`

```c
extern u32 sdio_readl(struct sdio_func *func,
                      unsigned int addr, int *err_ret);
```

بتقرأ 32-bit value بأربع CMD52 calls (little-endian: addr, addr+1, addr+2, addr+3).

**Return:** الـ 32-bit value أو `0xFFFFFFFF` عند الـ error

---

#### `sdio_writeb`

```c
extern void sdio_writeb(struct sdio_func *func, u8 b,
                        unsigned int addr, int *err_ret);
```

بتكتب byte واحد على `addr` عن طريق CMD52 مع R/W=1.

**Parameters:**
- `b` — القيمة المراد كتابتها
- باقي الـ parameters نفس `sdio_readb`

---

#### `sdio_writew`

```c
extern void sdio_writew(struct sdio_func *func, u16 b,
                        unsigned int addr, int *err_ret);
```

بتكتب 16-bit value بـ CMd52 مرتين (little-endian).

---

#### `sdio_writel`

```c
extern void sdio_writel(struct sdio_func *func, u32 b,
                        unsigned int addr, int *err_ret);
```

بتكتب 32-bit value بأربع CMD52 calls.

---

#### `sdio_writeb_readb`

```c
extern u8 sdio_writeb_readb(struct sdio_func *func, u8 write_byte,
                             unsigned int addr, int *err_ret);
```

CMD52 RAW (Read After Write) operation — بتكتب `write_byte` وفي نفس الوقت بتقرأ القيمة القديمة **atomic** في نفس الـ command. الـ RAW bit (bit 27) في CMD52 argument بييجي مضبوط.

**Parameters:**
- `write_byte` — القيمة الجديدة المراد كتابتها
- `addr` — الـ register address
- `err_ret` — pointer لـ error code

**Return:** القيمة القديمة قبل الكتابة

**Key details:**
- مفيد جداً للـ atomic set/clear operations على registers زي interrupt masks
- الـ atomicity هنا على مستوى الـ bus command، مش software lock

---

### فئة 7: Bulk I/O (CMD53 Based)

**الغرض:** نقل blocks من البيانات. CMD53 أكفأ بكتير من CMD52 المتكررة.

---

#### `sdio_memcpy_fromio`

```c
extern int sdio_memcpy_fromio(struct sdio_func *func, void *dst,
                               unsigned int addr, int count);
```

بتقرأ `count` bytes من الـ function بدءاً من `addr` مع increment تلقائي للـ address (OP CODE=1). بتستخدم CMD53 في الـ byte mode أو block mode حسب الـ size والـ capabilities.

**Parameters:**
- `dst` — destination buffer (لازم يكون DMA-capable أو الـ core بيعمل bounce)
- `addr` — starting register address
- `count` — عدد الـ bytes

**Return:** `0` نجاح، أو negative error code

**Key details:**
- **يتطلب host claim**
- الـ core بيتعامل تلقائياً مع الـ block vs byte mode splitting
- لو الـ count مش aligned على الـ block size، بيستخدم `sdio_align_size` داخلياً
- الـ `dst` buffer لازم يكون DMA-safe (يُفضّل allocation من `kmalloc` مش stack)

---

#### `sdio_readsb`

```c
extern int sdio_readsb(struct sdio_func *func, void *dst,
                       unsigned int addr, int count);
```

زي `sdio_memcpy_fromio` بالظبط بس مع OP CODE=0 (fixed address — FIFO mode). بتقرأ `count` bytes كلها من نفس الـ `addr`.

**Return:** `0` نجاح، أو negative error code

**Use case:** قراءة من hardware FIFO registers زي WiFi/BT data pipes.

---

#### `sdio_memcpy_toio`

```c
extern int sdio_memcpy_toio(struct sdio_func *func, unsigned int addr,
                             void *src, int count);
```

بتكتب `count` bytes في الـ function بدءاً من `addr` مع address increment. CMD53 write direction.

**Parameters:**
- `addr` — destination register address
- `src` — source buffer
- `count` — عدد الـ bytes

**Return:** `0` نجاح، أو negative error code

---

#### `sdio_writesb`

```c
extern int sdio_writesb(struct sdio_func *func, unsigned int addr,
                        void *src, int count);
```

زي `sdio_memcpy_toio` بس FIFO mode (fixed address، OP CODE=0). بتكتب `count` bytes كلها على نفس الـ `addr`.

---

### فئة 8: Function 0 (CCCR) Access

**الغرض:** الـ Function 0 هي الـ meta-function اللي بتتحكم في الـ card نفسه (CCCR space). الـ access ليها مقيّد في الـ spec — الـ write مسموح فقط في vendor-specific area (0xF0-0xFF) إلا لو الـ quirk `MMC_QUIRK_LENIENT_FN0` مضبوط.

---

#### `sdio_f0_readb`

```c
extern unsigned char sdio_f0_readb(struct sdio_func *func,
                                    unsigned int addr, int *err_ret);
```

بتقرأ byte من CCCR address space (function 0) عن طريق CMD52 مع function number = 0.

**Parameters:**
- `func` — أي SDIO function (بتوصّل للكارت عشان تبعت CMD52 بـ fn=0)
- `addr` — الـ CCCR register address
- `err_ret` — optional error pointer

**Return:** القيمة المقروءة أو `0xFF` عند الـ error

---

#### `sdio_f0_writeb`

```c
extern void sdio_f0_writeb(struct sdio_func *func, unsigned char b,
                            unsigned int addr, int *err_ret);
```

بتكتب byte في CCCR space. الـ kernel بيرفض الكتابة خارج `0xF0-0xFF` إلا لو الكارت عنده quirk `MMC_QUIRK_LENIENT_FN0`.

**Key details:**
- `SDIO_CCCR_ABORT` (0x06) مثلاً بيتكتب فيه عشان تعمل function abort أو card reset
- الكتابة في `SDIO_CCCR_IF` (0x07) بتغير الـ bus width

---

### فئة 9: Power Management

**الغرض:** تسمح للـ SDIO driver يتحكم في suspend/resume behavior.

---

#### `sdio_get_host_pm_caps`

```c
extern mmc_pm_flag_t sdio_get_host_pm_caps(struct sdio_func *func);
```

بترجع الـ PM capabilities المدعومة من الـ host controller.

**Return:** bitmask من:
- `MMC_PM_KEEP_POWER` — الـ host قادر يحافظ على power أثناء suspend
- `MMC_PM_WAKE_SDIO_IRQ` — الـ host قادر يصحى على SDIO interrupt

---

#### `sdio_set_host_pm_flags`

```c
extern int sdio_set_host_pm_flags(struct sdio_func *func,
                                   mmc_pm_flag_t flags);
```

بتطلب من الـ host يفعّل PM features معينة أثناء الـ next suspend. الـ driver بيستدعيها في `suspend` callback بتاعه.

**Parameters:**
- `flags` — OR من `MMC_PM_KEEP_POWER` و/أو `MMC_PM_WAKE_SDIO_IRQ`

**Return:** `0` نجاح، `-EINVAL` لو طلب flag مش مدعوم من الـ host

**Example (WiFi driver suspend):**
```c
static int my_wifi_suspend(struct device *dev)
{
    struct sdio_func *func = dev_to_sdio_func(dev);
    mmc_pm_flag_t caps = sdio_get_host_pm_caps(func);

    if (caps & MMC_PM_KEEP_POWER)
        return sdio_set_host_pm_flags(func,
                    MMC_PM_KEEP_POWER | MMC_PM_WAKE_SDIO_IRQ);
    return -ENOSYS;
}
```

---

### فئة 10: Retuning Control

**الغرض:** الـ UHS-I modes (SDR50, SDR104) بتتطلب periodic retuning عشان تعوّض الـ voltage/temperature drift. أحياناً الـ driver محتاج يتحكم في timing الـ retuning.

---

#### `sdio_retune_crc_disable`

```c
extern void sdio_retune_crc_disable(struct sdio_func *func);
```

بتمنع الـ host من trigger الـ retuning لما يشوف CRC errors. مفيد لما الـ driver نفسه بيتعامل مع الـ errors ومش عايز الـ core يتدخل.

---

#### `sdio_retune_crc_enable`

```c
extern void sdio_retune_crc_enable(struct sdio_func *func);
```

بتستعيد السلوك الافتراضي — CRC error بيـtrigger retuning.

---

#### `sdio_retune_hold_now`

```c
extern void sdio_retune_hold_now(struct sdio_func *func);
```

بتجمّد الـ retuning فوراً. الـ host مش هيعمل retune حتى لو الـ timer خلص. بتتستخدم لما الـ driver عارف إن الـ tuning parameters لسه valid.

---

#### `sdio_retune_release`

```c
extern void sdio_retune_release(struct sdio_func *func);
```

بتحرر الـ retuning hold وتسمح للـ core يعمل retune مرة تانية. لازم تتعمل بعد كل `sdio_retune_hold_now()`.

---

### الـ Helper Macros في `sdio_func.h`

| Macro | الوصف |
|---|---|
| `sdio_func_present(f)` | يتحقق لو الـ function registered في sysfs |
| `sdio_func_set_present(f)` | بيضبط state الـ function كـ present |
| `sdio_func_id(f)` | بيرجع اسم الـ device في sysfs (e.g. `"mmc1:0001:1"`) |
| `sdio_get_drvdata(f)` | بيجيب الـ driver private data |
| `sdio_set_drvdata(f, d)` | بيحدد الـ driver private data |
| `dev_to_sdio_func(d)` | بيحوّل `struct device *` لـ `struct sdio_func *` |

---

### الـ Data Flow الكامل لـ SDIO Driver

```
module_init
    └── sdio_register_driver()
            └── driver_register() → matches id_table
                    └── probe(func, id)
                            ├── sdio_claim_host(func)
                            ├── sdio_enable_func(func)       → CMD52: IOEx
                            ├── sdio_set_block_size(func, N) → CMD52: FBR_BLKSIZE
                            ├── sdio_claim_irq(func, handler)→ CMD52: IENx
                            ├── sdio_release_host(func)
                            └── setup netdev/misc/etc.

IRQ path:
    card asserts DAT1
        → sdio_irq_thread
            ├── sdio_claim_host()
            ├── read SDIO_CCCR_INTx (CMD52)
            ├── call func->irq_handler(func)
            └── sdio_release_host()

Data TX:
    sdio_claim_host(func)
    sdio_memcpy_toio(func, addr, buf, len)  → CMD53 write
    sdio_release_host(func)

module_exit:
    └── sdio_unregister_driver()
            └── remove(func)
                    ├── sdio_claim_host(func)
                    ├── sdio_release_irq(func)    → CMD52: clear IENx bit
                    ├── sdio_disable_func(func)   → CMD52: clear IOEx bit
                    └── sdio_release_host(func)
```
## Phase 5: دليل الـ Debugging الشامل

### Software Level

---

#### 1. debugfs Entries

الـ **debugfs** بيوفر visibility كاملة على حالة الـ MMC/SDIO subsystem.

```bash
# mount debugfs لو مش mounted
mount -t debugfs none /sys/kernel/debug

# شوف كل entries خاصة بالـ MMC
ls /sys/kernel/debug/mmc*/
```

| Entry | المسار | المحتوى |
|-------|--------|---------|
| mmc0 directory | `/sys/kernel/debug/mmc0/` | كل معلومات الـ host controller |
| ios | `/sys/kernel/debug/mmc0/ios` | الـ I/O settings الحالية (clock, bus width, timing) |
| clock | `/sys/kernel/debug/mmc0/clock` | قيمة الـ clock الفعلية |
| reqs | `/sys/kernel/debug/mmc0/reqs` | قائمة الـ requests |
| err_stats | `/sys/kernel/debug/mmc0/err_stats` | إحصائيات الأخطاء |

```bash
# قرا الـ ios لمعرفة الـ bus width والـ timing mode
cat /sys/kernel/debug/mmc0/ios
# مثال على الـ output:
# clock:          50000000 Hz
# actual clock:   50000000 Hz
# vdd:            21 (3.3 ~ 3.4 V)
# bus mode:       push-pull (2)
# chip select:    don't care (0)
# power mode:     on (3)
# bus width:      4 bits (2)
# timing spec:    sd high-speed (2)
# signal voltage: 3.30 V (0)
# driver type:    default (0)

# شوف err_stats لمعرفة تاريخ الأخطاء
cat /sys/kernel/debug/mmc0/err_stats
```

---

#### 2. sysfs Entries

الـ **sysfs** بيعرض معلومات الـ SDIO function devices تحت `/sys/bus/sdio/`.

```bash
# اعرض كل الـ SDIO devices المتصلة
ls /sys/bus/sdio/devices/

# لكل device، اعرض بياناتها:
# mmc0:0001:1 = host0, card1, function1
ls /sys/bus/sdio/devices/mmc0:0001:1/

# قرا vendor/device IDs
cat /sys/bus/sdio/devices/mmc0:0001:1/vendor
cat /sys/bus/sdio/devices/mmc0:0001:1/device
cat /sys/bus/sdio/devices/mmc0:0001:1/class

# قرا block size الحالي للـ function
cat /sys/bus/sdio/devices/mmc0:0001:1/block_size

# شوف الـ driver المرتبط بالـ function
ls -la /sys/bus/sdio/devices/mmc0:0001:1/driver
```

| sysfs Attribute | المعنى |
|-----------------|--------|
| `vendor` | الـ Vendor ID من الـ FBR |
| `device` | الـ Device ID من الـ FBR |
| `class` | Standard Interface Class |
| `block_size` | الـ current block size للـ CMD53 |
| `modalias` | الـ driver alias للـ auto-loading |

---

#### 3. ftrace — Tracepoints وـ Events

الـ **ftrace** هو أقوى أداة لتتبع مسار تنفيذ الـ SDIO commands.

```bash
# enable الـ MMC tracepoints
cd /sys/kernel/debug/tracing

# شوف كل الـ available events خاصة بالـ MMC
grep -r "mmc" available_events

# enable حدث إرسال الـ SDIO command
echo 1 > events/mmc/mmc_request_start/enable
echo 1 > events/mmc/mmc_request_done/enable
echo 1 > events/mmc/mmc_cmd_rw_end/enable

# enable كل أحداث الـ MMC دفعة واحدة
echo 1 > events/mmc/enable

# ابدأ الـ tracing
echo 1 > tracing_on

# افعل العملية اللي بتحتاج تـ debug فيها (مثلاً load driver)

# وقف الـ tracing وقرا النتائج
echo 0 > tracing_on
cat trace
```

**مثال على output بعد CMD52 (SD_IO_RW_DIRECT):**

```
# tracer: nop
#
kworker/0:2-123   [000] .... 1234.567890: mmc_request_start: mmc0: start struct mmc_request[...]
kworker/0:2-123   [000] .... 1234.567920: mmc_cmd_rw_end:   mmc0: CMD52 args=0x00000000 resp=0x00001000 err=0
kworker/0:2-123   [000] .... 1234.567930: mmc_request_done: mmc0: end struct mmc_request[...]
```

```bash
# filter لـ function number معين (مثلاً function 1)
echo 'func==1' > events/mmc/mmc_request_start/filter

# استخدم function_graph tracer لتتبع call stack كامل
echo function_graph > current_tracer
echo sdio_io_rw_direct > set_graph_function
echo 1 > tracing_on
```

---

#### 4. printk والـ Dynamic Debug

تفعيل **dynamic debug** للـ MMC/SDIO subsystem بدون recompile.

```bash
# enable كل الـ debug messages في ملفات الـ SDIO
echo 'file sdio*.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file mmc*.c +p' > /sys/kernel/debug/dynamic_debug/control

# enable debug messages في module معين (مثلاً brcmfmac WiFi SDIO driver)
echo 'module brcmfmac +p' > /sys/kernel/debug/dynamic_debug/control

# enable مع stack trace لكل message
echo 'file sdio_ops.c +ps' > /sys/kernel/debug/dynamic_debug/control

# شوف القواعد الفعّالة حالياً
cat /sys/kernel/debug/dynamic_debug/control | grep mmc

# من kernel command line (في /etc/default/grub أو bootloader)
# dyndbg="file sdio.c +p; module mmc_core +p"

# set kernel log level لـ KERN_DEBUG
echo 8 > /proc/sys/kernel/printk
# أو
dmesg -n 7
```

**مثال على رسائل debug بعد التفعيل:**

```
[  12.345] mmc0: starting SDIO init
[  12.346] mmc0: CMD5 (SD_IO_SEND_OP_COND) OCR=0x00000000
[  12.350] mmc0: SDIO card detected, OCR=0x90ff8000
[  12.351] mmc0: CCCR revision 3.00, SDIO revision 3.00
[  12.352] mmc0: function 1: class=0, vendor=0x02d0, device=0x4330
```

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الوظيفة |
|---------------|---------|
| `CONFIG_MMC_DEBUG` | تفعيل debug messages في الـ MMC core |
| `CONFIG_DYNAMIC_DEBUG` | تفعيل dynamic debug infrastructure |
| `CONFIG_MMC_UNSAFE_RESUME` | للـ debugging مشاكل الـ suspend/resume |
| `CONFIG_MMC_SDHCI_IO_ACCESSORS` | عرض register access للـ debugging |
| `CONFIG_FAULT_INJECTION` | حقن أخطاء اصطناعية للـ testing |
| `CONFIG_FAIL_MMC_REQUEST` | حقن أخطاء في الـ MMC requests تحديداً |
| `CONFIG_LOCK_STAT` | statistics للـ mutex/spinlock |
| `CONFIG_DEBUG_SPINLOCK` | كشف deadlocks في الـ host lock |
| `CONFIG_KASAN` | كشف memory corruption في buffers الـ DMA |
| `CONFIG_SLUB_DEBUG` | debug allocator لكشف use-after-free |

```bash
# تحقق من الـ config في kernel حالي
zcat /proc/config.gz | grep -E "CONFIG_(MMC|SDIO|FAIL_MMC)"
# أو
grep -E "CONFIG_(MMC|SDIO|FAIL_MMC)" /boot/config-$(uname -r)
```

**تفعيل fault injection للـ SDIO testing:**

```bash
# بعد تفعيل CONFIG_FAIL_MMC_REQUEST
echo 1 > /sys/kernel/debug/mmc0/fail_mmc_request/probability
echo 1 > /sys/kernel/debug/mmc0/fail_mmc_request/times
echo 1 > /sys/kernel/debug/mmc0/fail_mmc_request/task-filter
```

---

#### 6. أدوات خاصة بالـ SDIO Subsystem

**الـ mmc-utils:**

```bash
# install
apt-get install mmc-utils

# قرا الـ CSD/CID/EXT_CSD
mmc info /dev/mmcblk0

# قرا CCCR register معين (مثلاً SDIO_CCCR_CAPS = 0x08)
# ده بيتعمل عبر CMD52 على function 0
mmc sdio_read /dev/mmcblk0 0x08

# ابدأ الـ high-speed mode
mmc extcsd read /dev/mmcblk0

# شوف الـ bus speed
cat /sys/kernel/debug/mmc0/ios | grep timing
```

**الـ devlink (للـ SDIO في context الـ networking):**

```bash
# لو الـ SDIO device هو network adapter (مثلاً WiFi)
devlink dev info
devlink health show
devlink health dump show pci/0000:01:00.0
```

**قراءة الـ CCCR registers مباشرة عبر sysfs:**

```bash
# بعض الـ drivers بيعرضوا registers عبر debugfs
ls /sys/kernel/debug/brcmfmac/
cat /sys/kernel/debug/brcmfmac/sdio/counters
```

---

#### 7. جدول رسائل الأخطاء الشائعة

| رسالة الخطأ | المعنى | الحل |
|-------------|--------|------|
| `mmc0: error -110 whilst initialising SDIO card` | Timeout أثناء CMD5 — الكارت مش بيرد | تحقق من الـ power supply وسلامة الاتصال الفيزيائي |
| `mmc0: CMD52 error -84 (R5_COM_CRC_ERROR)` | CRC error في الـ command — ضوضاء في الـ bus | قلل الـ clock speed، تحقق من جودة الكابلات |
| `mmc0: CMD52 error R5_ILLEGAL_COMMAND` | الكارت في state غلط أو الـ function مش enabled | تأكد من استدعاء `sdio_enable_func()` قبل أي I/O |
| `mmc0: CMD52 error R5_FUNCTION_NUMBER` | رقم الـ function في الأمر غير موجود | تحقق من `SDIO_FBR_BASE(f)` وعدد الـ functions الفعلي |
| `mmc0: CMD52 error R5_OUT_OF_RANGE` | عنوان الـ register خارج النطاق | تحقق من قيمة الـ address في bits [25:9] |
| `sdio: probe of mmc0:0001:1 failed with error -22` | الـ driver رفض الـ device (EINVAL) | تحقق من `sdio_device_id` table في الـ driver |
| `mmc0: tuning execution failed` | فشل الـ tuning لـ UHS modes | جرب `SDIO_SPEED_SDR25` بدل `SDR104` |
| `mmc0: Card stuck in wrong state!` | الكارت في state غير متوقع بعد error | ابعت `SDIO_CCCR_ABORT` (CMD52 write على 0x06) |
| `mmc0: SDIO: retry cmd 53 response error` | خطأ في CMD53 (RW_EXTENDED) | تحقق من الـ block size وـ DMA alignment |
| `mmc0: host does not support reading read-only switch` | الـ host controller محدودة capabilities | تحقق من `host->caps` flags |

---

#### 8. Strategic Points لـ dump_stack() وـ WARN_ON()

نقاط استراتيجية في الـ SDIO flow تنفع تحطها:

```c
/* في sdio_enable_func() — تأكد الـ function اتـ enable فعلاً */
int sdio_enable_func(struct sdio_func *func)
{
    /* بعد قراءة SDIO_CCCR_IORx */
    if (!(reg & (1 << func->num))) {
        WARN_ON(1); /* Function didn't enable after timeout */
        dump_stack();
        return -ETIMEDOUT;
    }
}

/* في sdio_claim_irq() — تأكد مفيش double claim */
int sdio_claim_irq(struct sdio_func *func, sdio_irq_handler_t *handler)
{
    WARN_ON(func->irq_handler); /* already claimed! */
    /* ... */
}

/* في sdio_set_block_size() — تحقق من الحدود */
int sdio_set_block_size(struct sdio_func *func, unsigned blksz)
{
    WARN_ON(blksz > func->card->host->max_blk_size);
    WARN_ON(blksz > func->max_blksize);
    /* ... */
}

/* في الـ IRQ handler — تأكد الـ function في الـ right state */
void my_sdio_irq_handler(struct sdio_func *func)
{
    if (WARN_ON(!sdio_func_present(func))) {
        /* Function removed but IRQ still firing */
        dump_stack();
        return;
    }
}
```

---

### Hardware Level

---

#### 1. التحقق من توافق الـ Hardware State مع الـ Kernel State

```bash
# تحقق من الـ bus width الفعلية مقارنة بما يقوله الـ kernel
cat /sys/kernel/debug/mmc0/ios
# إذا قال "4 bits" يجب أن تكون الـ DAT0-DAT3 كلها active على المنطق

# تحقق من الـ clock frequency الفعلية
cat /sys/kernel/debug/mmc0/clock
# قارن بـ actual clock في ios output

# تحقق من الـ signal voltage (3.3V vs 1.8V)
cat /sys/kernel/debug/mmc0/ios | grep "signal voltage"
# 3.30 V = عادي SD mode
# 1.80 V = UHS-I mode (SDR50/SDR104/DDR50)

# تحقق من أن الـ CCCR يعكس الـ bus width الحالية
# SDIO_CCCR_IF (0x07) bits [1:0]:
# 00 = 1-bit, 10 = 4-bit
```

**جدول التحقق من التوافق:**

| ما يقوله الـ kernel | ما تبحث عنه على الـ hardware |
|--------------------|-----------------------------|
| timing: sd high-speed | CLK يجب أن يكون ≤ 50 MHz |
| bus width: 4 bits | DAT0-DAT3 كلها تحمل بيانات |
| signal voltage: 1.80V | قياس VDD على الـ card = 1.8V |
| SDIO_SPEED_SDR104 | CLK يصل لـ 208 MHz |

---

#### 2. Register Dump Techniques

```bash
# استخدام devmem2 لقراءة registers الـ host controller
# (SDHCI على عنوان 0xFE300000 مثلاً على Raspberry Pi)
apt-get install devmem2

# قرا SDHCI_PRESENT_STATE register (offset 0x24)
devmem2 0xFE300024 w
# Output: 0xFE300024: 0x1FF0200 = DAT3-0 present, card inserted

# قرا SDHCI_CAPABILITIES register (offset 0x40)
devmem2 0xFE300040 w
# يعطيك الـ supported speeds والـ voltage ranges

# باستخدام /dev/mem مباشرة (تحتاج CONFIG_DEVMEM=y)
# قراءة CCCR register 0x07 (bus interface)
python3 -c "
import mmap, struct
f = open('/dev/mem', 'rb')
m = mmap.mmap(f.fileno(), 4096, offset=0xFE300000)
val = struct.unpack('<I', m[0x24:0x28])[0]
print(f'PRESENT_STATE: 0x{val:08x}')
m.close()
f.close()
"
```

**SDHCI Registers المهمة للـ Debugging:**

| Register | Offset | المعنى |
|----------|--------|--------|
| `SDHCI_PRESENT_STATE` | 0x24 | حالة الـ card والـ data lines |
| `SDHCI_HOST_CONTROL` | 0x28 | bus width وـ speed mode |
| `SDHCI_CLOCK_CONTROL` | 0x2C | الـ clock divisor |
| `SDHCI_INT_STATUS` | 0x30 | interrupt flags |
| `SDHCI_INT_ENABLE` | 0x34 | enabled interrupts |
| `SDHCI_CAPABILITIES` | 0x40 | قدرات الـ controller |

---

#### 3. Logic Analyzer وـ Oscilloscope Tips

**الـ SDIO bus signals:**

```
Signal    Pin#    ما تراقبه
-------   ----    ---------
CLK       5       الـ clock — يجب أن يكون نظيف بدون jitter
CMD       2       الـ commands والـ responses (half-duplex)
DAT0      7       البيانات + IRQ في 1-bit mode
DAT1      8       بيانات أو IRQ في multi-bit mode
DAT2      9       بيانات
DAT3      1       بيانات أو card detect
VDD       4       يجب 3.3V أو 1.8V حسب الـ mode
GND       3,6     ground reference
```

**إعداد الـ Logic Analyzer:**

```
Sample Rate: 10x الـ clock speed على الأقل
             مثلاً لـ SDR50 (50MHz) → 500MHz sample rate

Trigger: CMD line falling edge (بداية الـ command)
Protocol: اختار SDIO/SD protocol decoder

نقاط المراقبة:
1. CMD5 (SD_IO_SEND_OP_COND) → R4 response, تحقق من bit 27 (R4_MEMORY_PRESENT)
2. CMD52 على CCCR_IOEx (0x02) → تحقق من enable bits
3. CMD52 على CCCR_INTx (0x05) → تتبع الـ interrupts
4. CMD53 transactions → تحقق من الـ block count وـ address increment
```

**بالـ Oscilloscope:**

```
- CLK: قياس frequency دقيقة، rise/fall time < 1ns للـ high-speed
- VDD: تحقق من voltage ripple < 100mV خلال الـ burst transfers
- DAT lines: eye diagram لكشف الـ signal integrity issues
- تحقق من غياب الـ glitches على CMD خلال الـ data transfer
```

---

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة الـ hardware | Pattern في الـ kernel log | التشخيص |
|---------------------|--------------------------|---------|
| ضعف power supply | `mmc0: error -110 whilst initialising` + يختفي بعد reboot | قس VDD بالـ multimeter أثناء الـ init |
| سلك CMD مكسور | `mmc0: CMD52 error -84 (CRC_ERROR)` متكرر | افحص continuity بين الـ host وـ card |
| غياب pull-up على CMD | `mmc0: Timeout waiting for hardware interrupt` | راجع الـ R4-10kΩ على CMD line |
| EMI/noise | CRC errors عشوائية بدون pattern | فرّغ الـ board من مصادر noise، استخدم ferrite bead |
| مشكلة في الـ 1.8V regulator | `mmc0: failed to switch to 1.8V` | قس LDO output خلال voltage switch |
| DAT3 pull-up مفقود | Card detection fails | تحقق من `SDIO_BUS_CD_DISABLE` في CCCR_IF bit 7 |
| Clock too fast | `mmc0: tuning execution failed` | ابدأ بـ SDR12 وارفع تدريجياً |

---

#### 5. Device Tree Debugging

```bash
# تحقق من الـ DT compiled
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "mmc\|sdhci\|sdio"

# أو استخدم fdtdump
apt-get install device-tree-compiler
dtc -I dtb -O dts /boot/dtb/your-board.dtb | grep -A 30 "mmc"

# تحقق من الـ properties المهمة
cat /sys/firmware/devicetree/base/soc/mmc@fe300000/bus-width
# يجب أن يعطيك 4 لـ 4-bit mode

cat /sys/firmware/devicetree/base/soc/mmc@fe300000/max-frequency
# بالـ Hz مثلاً: 200000000 = 200MHz

# هل الـ SDIO interrupts enabled؟
cat /sys/firmware/devicetree/base/soc/mmc@fe300000/interrupts
```

**DT Properties الضرورية لـ SDIO:**

```dts
/* مثال DT node صح لـ SDIO WiFi */
mmc1: mmc@fe340000 {
    compatible = "brcm,bcm2835-sdhci";
    reg = <0xfe340000 0x100>;
    interrupts = <GIC_SPI 126 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&clocks BCM2835_CLOCK_EMMC>;
    bus-width = <4>;           /* يجب أن يطابق الـ SDIO card */
    max-frequency = <50000000>; /* SDR25 = 50MHz */
    non-removable;             /* للـ WiFi chips الثابتة */
    cap-sd-highspeed;
    cap-sdio-irq;              /* مهم جداً للـ SDIO interrupts */
    vmmc-supply = <&reg_3v3>;
    vqmmc-supply = <&reg_1v8>; /* للـ UHS-I */
};
```

```bash
# تحقق هل cap-sdio-irq موجودة
grep -r "cap-sdio-irq" /sys/firmware/devicetree/base/
# لو مش موجودة → الـ interrupts هتعمل polling بدل hardware IRQ

# تحقق هل non-removable set
grep -r "non-removable" /sys/firmware/devicetree/base/soc/
# مهم للـ embedded SDIO devices (WiFi, BT)

# مقارنة الـ DT parsed مع الـ kernel state
diff <(cat /sys/kernel/debug/mmc0/ios) \
     <(dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A5 "mmc@")
```

---

### Practical Commands

---

#### شيت الأوامر الجاهزة للنسخ

**1. فحص سريع لحالة الـ SDIO:**

```bash
#!/bin/bash
# sdio_quick_check.sh — فحص شامل سريع

echo "=== MMC/SDIO Devices ==="
ls /sys/bus/sdio/devices/ 2>/dev/null || echo "No SDIO devices found"

echo ""
echo "=== dmesg SDIO messages ==="
dmesg | grep -i "mmc\|sdio" | tail -30

echo ""
echo "=== IOS State ==="
for host in /sys/kernel/debug/mmc*/ios; do
    echo "--- $host ---"
    cat "$host" 2>/dev/null
done

echo ""
echo "=== SDIO Error Stats ==="
for stats in /sys/kernel/debug/mmc*/err_stats; do
    echo "--- $stats ---"
    cat "$stats" 2>/dev/null
done
```

**2. قراءة CCCR registers كلها:**

```bash
#!/bin/bash
# read_cccr.sh — قرا الـ 22 register الأول من الـ CCCR
# بيستخدم CMD52 عبر sysfs (بعض الـ drivers بيدعموا ده)

REGISTERS=(
    "0x00:CCCR_VERSION"
    "0x01:SD_VERSION"
    "0x02:IO_ENABLE"
    "0x03:IO_READY"
    "0x04:INT_ENABLE"
    "0x05:INT_PENDING"
    "0x06:IO_ABORT"
    "0x07:BUS_INTERFACE"
    "0x08:CAPABILITIES"
    "0x13:SPEED"
    "0x14:UHS_SUPPORT"
    "0x15:DRIVE_STRENGTH"
    "0x16:INTERRUPT_EXT"
)

for reg in "${REGISTERS[@]}"; do
    addr="${reg%%:*}"
    name="${reg##*:}"
    # قراءة عبر debugfs لو الـ driver دعم
    val=$(cat /sys/kernel/debug/mmc0/sdio_cccr_${addr} 2>/dev/null || echo "N/A")
    printf "CCCR[%s] %-20s = %s\n" "$addr" "$name" "$val"
done
```

**3. تفعيل complete SDIO debugging:**

```bash
#!/bin/bash
# enable_sdio_debug.sh

# Dynamic debug
echo 'file sdio*.c +pmf' > /sys/kernel/debug/dynamic_debug/control
echo 'file mmc_ops*.c +pmf' > /sys/kernel/debug/dynamic_debug/control
echo 'file core.c +p' > /sys/kernel/debug/dynamic_debug/control

# ftrace events
echo 1 > /sys/kernel/debug/tracing/events/mmc/enable

# kernel log level
echo 8 > /proc/sys/kernel/printk

echo "SDIO debugging enabled. Watch: dmesg -w"
```

**4. تشخيص مشكلة الـ IRQ:**

```bash
# تحقق هل الـ SDIO IRQ registered
cat /proc/interrupts | grep mmc

# تحقق من عدد الـ IRQs المُستقبلة
watch -n 1 'cat /proc/interrupts | grep mmc'
# لو العداد بيزيد = الـ IRQs بتوصل
# لو بيقف = مشكلة hardware أو الـ handler مش registered

# شوف الـ IRQ handler المسجل
cat /proc/irq/$(cat /proc/interrupts | grep mmc | awk '{print $1}' | tr -d ':')/actions
```

**5. إعادة تهيئة الـ SDIO card يدوياً:**

```bash
# echo 0 ثم 1 لـ reset الكارت
echo 0 > /sys/bus/platform/drivers/sdhci-iproc/fe340000.mmc/mmc_host/mmc1/power/control
sleep 1
echo 1 > /sys/bus/platform/drivers/sdhci-iproc/fe340000.mmc/mmc_host/mmc1/power/control

# أو unbind/bind الـ driver
DEVICE=$(ls /sys/bus/sdio/devices/ | head -1)
echo "$DEVICE" > /sys/bus/sdio/drivers/brcmfmac/unbind
sleep 2
echo "$DEVICE" > /sys/bus/sdio/drivers/brcmfmac/bind
```

**6. تتبع CMD52/CMD53 بالـ ftrace مع output مفصل:**

```bash
# script كامل لـ capture وـ analysis
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo > trace                          # clear buffer
echo 16384 > buffer_size_kb           # زود الـ buffer
echo 1 > events/mmc/mmc_request_start/enable
echo 1 > events/mmc/mmc_request_done/enable
echo 1 > tracing_on

# افعل العملية (مثلاً: rmmod + modprobe)
modprobe -r brcmfmac && modprobe brcmfmac

echo 0 > tracing_on
# احفظ الـ trace
cp trace /tmp/sdio_trace_$(date +%Y%m%d_%H%M%S).txt
echo "Saved trace to /tmp/"
# count الـ errors
grep "err=[^0]" /tmp/sdio_trace_*.txt | wc -l
```

**مثال output من الـ trace وكيف تقرأه:**

```
# بداية CMD52 لقراءة CCCR_IOEx
kworker-123  [000] 100.1: mmc_request_start: mmc0: cmd=52 arg=0x00001000 flags=0x95
#                                                         ^^^  ^^^^^^^^
#                                                         CMD52  arg: R/W=0(read), func=0, addr=0x08 (CAPS)

# الـ response
kworker-123  [000] 100.2: mmc_request_done: mmc0: cmd=52 resp=0x00001020 err=0
#                                                              ^^^^^^^^
#                                                              R5: status=0, state=Transfer, data=0x20 (E4MI set)
```

**تفسير R5 response:**
```
resp = 0x00001020
bits [15] = 0 → لا CRC error
bits [14] = 0 → لا illegal command
bits [13:12] = 01 → IO current state = Transfer
bits [11] = 0 → لا error
bits [9] = 0 → function number صح
bits [8] = 0 → in range
bits [7:0] = 0x20 → القيمة المقروءة من الـ register (SDIO_CCCR_CAP_E4MI)
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Wi-Fi SDIO مش بيتعرف على Android TV Box بـ Allwinner H616

#### العنوان
**Wi-Fi chip (RTL8189FTV)** مش بيطلع online على TV box بيشتغل بـ Android 11 فوق Allwinner H616.

#### السياق
شركة بتعمل Android TV box رخيصة بـ Allwinner H616، الـ Wi-Fi chip هو Realtek RTL8189FTV متوصل بـ SDIO. المنتج وصل للـ production line وبدأوا يلاقوا إن 20% من الوحدات مش بتعرف الـ Wi-Fi خالص بعد الـ power-on.

#### المشكلة
الـ kernel log بيطلع:

```
mmc1: error -110 whilst initialising SDIO card
mmc1: CMD5 timeout, retrying...
```

**الـ `CMD5`** ده هو `SD_IO_SEND_OP_COND` (اللي قيمته `5` في الهيدر) — أول command بيتبعت للـ SDIO card عشان تعرف نفسها. لو مجاوبتش يعني الـ card مش موجودة أو فيه مشكلة في power/clock.

#### التحليل

الهيدر بيحدد:
```c
#define SD_IO_SEND_OP_COND  5  /* bcr [23:0] OCR  R4 */
```

الـ response المتوقع هو **R4**، واللي بيحتوي على:
```c
#define R4_18V_PRESENT   (1 << 24)
#define R4_MEMORY_PRESENT (1 << 27)
```

الـ driver بيبعت CMD5 ويستنى R4 — لو الـ card مجاوبتش خالص (timeout)، يبقى المشكلة مش في الـ SDIO protocol نفسه، المشكلة في الـ physical layer.

فتحنا الـ schematic لقينا إن الـ SDIO VDD_IO بيجي من LDO بيطلع 1.8V، بس الـ RTL8189FTV محتاج 3.3V على الـ SDIO interface في الـ initialization phase. يعني الـ chip مش بتقدر ترد على CMD5 لأن الـ voltage غلط.

التحقق اللي أكّد الموضوع: الـ R4 response ضروري يحتوي على `R4_18V_PRESENT` بس لو الـ chip نفسها مش شايلة الـ VDD_IO صح، مش هتقدر ترد خالص.

#### الحل

في الـ Device Tree:
```dts
/* Before */
&mmc1 {
    vmmc-supply = <&reg_vcc3v3>;
    vqmmc-supply = <&reg_vcc1v8>;  /* غلط! */
};

/* After */
&mmc1 {
    vmmc-supply = <&reg_vcc3v3>;
    vqmmc-supply = <&reg_vcc3v3>;  /* صح — 3.3V للـ I/O */
    /* الـ UHS negotiation هتخلي الـ card تطلب 1.8V لو دعمت */
};
```

بعد كده الـ CMD5 نجح وظهر في الـ log:
```
mmc1: new SDIO card at address 0001
```

#### الدرس المستفاد
**الـ `SD_IO_SEND_OP_COND` (CMD5)** هو أول اختبار حقيقي للـ SDIO card. أي timeout فيه يبقى hardware/power مشكلة قبل ما تبص في الـ software. الفرق بين `vmmc` و`vqmmc` في الـ DT مهم جداً للـ SDIO cards.

---

### السيناريو الثاني: Industrial Gateway على RK3562 — الـ SDIO Wi-Fi بيوقف في الـ middle of transfer

#### العنوان
انقطاع متكرر في الـ SDIO Wi-Fi أثناء نقل بيانات كبيرة على industrial gateway.

#### السياق
Gateway صناعي بيشتغل على RK3562 بيجمع data من 50 حساس عبر RS-485 وبيبعتها على cloud بـ Wi-Fi (AP6256 متوصل SDIO). كل ساعتين تقريباً الـ Wi-Fi بيقطع وبيرجع تاني، بس في الـ production environment ده الانقطاع بيسبب data loss.

#### المشكلة
الـ kernel log بيطلع:
```
brcmfmac: brcmf_sdio_bus_txctl: ctrl_frame_stat == NULL
mmc0: Timeout waiting for hardware interrupt
sdio: CMD53 data error, R5 status: 0x00000800
```

**الـ `0x800`** في الـ R5 status ده `R5_ERROR` bit:
```c
#define R5_ERROR  (1 << 11)  /* erx, c */
```

#### التحليل

الـ `SD_IO_RW_EXTENDED` (CMD53) هو اللي بيُستخدم لنقل blocks:
```c
#define SD_IO_RW_EXTENDED  53  /* adtc [31:0] See below  R5 */
```

الـ argument format في الهيدر:
```
[31]    R/W flag
[30:28] Function number
[27]    Block mode
[26]    Increment address
[25:9]  Register address
[8:0]   Byte/block count
```

لما بيطلع `R5_ERROR` مع CMD53، يعني الـ card نفسها detect إن فيه مشكلة أثناء تنفيذ الـ command. شيلنا الـ logic analyzer على الـ SDIO bus لقينا إن الـ CLK بيتزعزع (jitter عالي) بعد ساعتين من التشغيل — السبب: الـ thermal throttling على الـ RK3562 بيخلي الـ SDIO clock source يتغير frequency.

الـ CCCR speed register مهم هنا:
```c
#define SDIO_CCCR_SPEED  0x13
#define  SDIO_SPEED_SHS  0x01  /* Supports High-Speed mode */
```

الـ driver كان بيحاول يشتغل بـ High-Speed (SDR25) بس الـ clock instability بتعمل errors.

#### الحل

حل أول: تخفيض الـ max frequency في الـ DT:
```dts
&sdmmc1 {
    max-frequency = <25000000>;  /* SDR12 بدل SDR25 */
    bus-width = <4>;
};
```

حل تاني (أفضل): تفعيل الـ `SDIO_CCCR_CAP_SRW` (read-wait protocol) عشان يقدر يوقف التransfer مؤقتاً لو فيه مشكلة بدل ما يفشل خالص:
```c
/* في الـ driver initialization */
if (card->cccr.cap & SDIO_CCCR_CAP_SRW) {
    /* enable read-wait for graceful recovery */
}
```

#### الدرس المستفاد
**الـ `R5_ERROR`** مع CMD53 في بيئات industrial يكون سببه الأول thermal/clock instability مش bug في الـ software. الـ CCCR capabilities registers (خصوصاً `SDIO_CCCR_CAPS`) لازم تتقرأ وتتحقق منها في الـ bring-up.

---

### السيناريو الثالث: IoT Sensor Board على STM32MP1 — الـ Interrupt مش بيوصل من الـ Wi-Fi

#### العنوان
الـ SDIO Wi-Fi interrupt مش شغال على IoT board بـ STM32MP1، الـ polling mode شغال بس بيرفع الـ CPU usage لـ 80%.

#### السياق
Board صغير لـ IoT بيشتغل على STM32MP157، بيستخدم RS9116 Wi-Fi/BT chip عبر SDIO. الـ driver شغال بـ polling mode لأن الـ interrupt لم يتكنفجر صح من الـ bring-up. الـ battery life بقت 6 ساعات بدل 24 ساعة.

#### المشكلة
حتى لما الـ interrupt line موصلة physically، الـ chip مش بتبعت interrupt. الـ dmesg بيطلع:
```
rs9116: interrupt mode requested but not working, falling back to polling
```

#### التحليل

الـ interrupt control في SDIO عبر CCCR registers:
```c
#define SDIO_CCCR_IENx  0x04  /* Function/Master Interrupt Enable */
#define SDIO_CCCR_INTx  0x05  /* Function Interrupt Pending */
```

عشان الـ interrupt يشتغل، لازم يتعملوا خطوتين:
1. تفعيل الـ master interrupt enable في bit 0 من `SDIO_CCCR_IENx`
2. تفعيل interrupt للـ function المعينة في bits 1-7

كمان لازم تبص على الـ CCCR capabilities:
```c
#define  SDIO_CCCR_CAP_S4MI  0x10  /* interrupt during 4-bit CMD53 */
#define  SDIO_CCCR_CAP_E4MI  0x20  /* enable ints during 4-bit CMD53 */
```

لو الـ bus عامل بـ 4-bit mode (وده الـ default)، لازم `E4MI` يكون set عشان الـ interrupts تشتغل أثناء الـ data transfer.

بعد قراءة الـ CCCR register على الـ target:
```bash
# قراءة CCCR_CAPS عبر debugfs
cat /sys/kernel/debug/mmc0/mmc0:0001/registers
```

لقينا إن `E4MI` مش set، والـ driver بتاع RS9116 مش بيتحقق من `S4MI` قبل ما يحاول يفعّل الـ 4-bit interrupt mode.

#### الحل

تعديل الـ driver initialization:
```c
/* في rs9116_sdio_probe() */
u8 caps;
sdio_readb(func, SDIO_CCCR_CAPS, &caps);

if (caps & SDIO_CCCR_CAP_S4MI) {
    /* card supports interrupts in 4-bit mode, enable it */
    u8 val = sdio_readb(func, SDIO_CCCR_CAPS, &ret);
    val |= SDIO_CCCR_CAP_E4MI;
    sdio_writeb(func, val, SDIO_CCCR_CAPS, &ret);
}

/* enable master interrupt + function 1 interrupt */
sdio_writeb(func, 0x03, SDIO_CCCR_IENx, &ret);
```

بعد الـ fix:
- الـ CPU usage نزل من 80% لـ 2%
- Battery life رجعت 22 ساعة

#### الدرس المستفاد
**الـ `SDIO_CCCR_CAP_S4MI`** و**`SDIO_CCCR_CAP_E4MI`** لازم يتحققوا منهم صراحة. الـ interrupt في 4-bit SDIO mode مش automatic — لازم الـ driver يفعّله بعد ما يتأكد إن الـ card بتدعمه.

---

### السيناريو الرابع: Automotive ECU على i.MX8 — الـ SDIO BT/Wi-Fi بيتهنج بعد suspend/resume

#### العنوان
الـ Bluetooth/Wi-Fi combo chip (88W8997) بيتهنج بعد الـ system suspend على automotive ECU بـ i.MX8QM.

#### السياق
ECU في سيارة كهربائية بيشتغل على i.MX8QM، بيستخدم Marvell 88W8997 (Wi-Fi + BT عبر SDIO) للاتصال بـ head unit. كل مرة الـ ECU يدخل في suspend وييجي من النوم، الـ BT/Wi-Fi بياخد وقت طويل يرجع (30 ثانية بدل 2 ثانية)، وأحياناً مبيرجعش خالص.

#### المشكلة
```
mmc2: error -110 whilst initialising SDIO card after resume
mmc2: All UHS modes disabled, trying CMD5 again
```

#### التحليل

الـ suspend/resume في SDIO بيمر بـ CCCR registers مخصصة:
```c
/* Following 4 regs are valid only if SBS is set */
#define SDIO_CCCR_SUSPEND  0x0c
#define SDIO_CCCR_SELx     0x0d
#define SDIO_CCCR_EXECx    0x0e
#define SDIO_CCCR_READYx   0x0f
```

وده مشروط بـ:
```c
#define  SDIO_CCCR_CAP_SBS  0x08  /* supports suspend/resume */
```

الـ flow اللي بيحصل:
1. عند الـ suspend، الـ kernel بيبعت لـ CCCR_SUSPEND عشان يوقف الـ function
2. عند الـ resume، المفروض يتحقق من `SDIO_CCCR_READYx` عشان يعرف الـ function وصحيت
3. بس الـ driver كان بيعمل re-initialization كاملة (CMD5 من الأول) بدل إنه يستخدم الـ SBS protocol

المشكلة الإضافية: الـ 88W8997 بعد الـ power cycle بتحتاج negotiation لـ UHS modes من الأول:
```c
#define SDIO_CCCR_UHS    0x14
#define  SDIO_UHS_SDR50  0x01
#define  SDIO_UHS_SDR104 0x02
#define  SDIO_UHS_DDR50  0x04
```

الـ driver كان بيقرأ `SDIO_CCCR_UHS` قبل ما الـ chip تخلص internal reset، فالـ values كانت invalid.

#### الحل

إضافة delay بعد الـ resume + استخدام `SDIO_CCCR_READYx` صح:
```c
static int mwifiex_sdio_resume(struct device *dev)
{
    /* wait for card to signal ready after power-on */
    int retry = 100;
    u8 ready;

    do {
        ready = sdio_readb(func, SDIO_CCCR_READYx, &err);
        if (ready & BIT(func->num))
            break;
        msleep(10);
    } while (--retry);

    if (!retry) {
        dev_err(dev, "card not ready after resume\n");
        return -ETIMEDOUT;
    }

    /* now safe to re-read UHS capabilities */
    u8 uhs = sdio_readb(func, SDIO_CCCR_UHS, &err);
    /* negotiate speed based on actual capabilities */
}
```

#### الدرس المستفاد
**الـ `SDIO_CCCR_READYx`** register مش بس للمعرفة — هو الـ synchronization mechanism الوحيد الموثوق للتأكد إن الـ SDIO function صحيت فعلاً بعد الـ suspend. في الـ automotive context ده أهم لأن الـ power cycles سريعة.

---

### السيناريو الخامس: Custom Board Bring-up على AM62x — الـ SDIO Card مش بتشتغل بـ High Speed

#### العنوان
Wi-Fi module (CYW43455) على custom board بـ Texas Instruments AM62x عامل بسرعة 25 Mbps بدل 150 Mbps المفروض.

#### السياق
Engineering team بتعمل custom board للـ industrial automation بـ AM62x، الـ Wi-Fi CYW43455 متوصل بـ SDIO. الـ basic connectivity شغالة بس الـ throughput منخفض جداً، الـ iperf بيطلع 25 Mbps max.

#### المشكلة
الـ kernel log بيطلع:
```
mmc1: new high speed SDIO card at address 0001
brcmfmac: F2 card ready, sd_bus_mode=0x1, sd_bus_width=0x2
```

`sd_bus_mode=0x1` ده SDR12 (12.5 MHz)، المفروض يكون على الأقل SDR25.

#### التحليل

الـ speed negotiation بتمر بـ CCCR_SPEED:
```c
#define SDIO_CCCR_SPEED     0x13
#define  SDIO_SPEED_SHS     0x01  /* Supports High-Speed mode */
#define  SDIO_SPEED_BSS_SHIFT  1
#define  SDIO_SPEED_BSS_MASK  (7<<SDIO_SPEED_BSS_SHIFT)
#define  SDIO_SPEED_SDR12   (0<<SDIO_SPEED_BSS_SHIFT)
#define  SDIO_SPEED_SDR25   (1<<SDIO_SPEED_BSS_SHIFT)
#define  SDIO_SPEED_SDR50   (2<<SDIO_SPEED_BSS_SHIFT)
#define  SDIO_SPEED_SDR104  (3<<SDIO_SPEED_BSS_SHIFT)
#define  SDIO_SPEED_DDR50   (4<<SDIO_SPEED_BSS_SHIFT)
```

الـ `SDIO_SPEED_EHS` (Enable High-Speed) هو نفسه `SDIO_SPEED_SDR25`:
```c
#define  SDIO_SPEED_EHS  SDIO_SPEED_SDR25  /* Enable High-Speed */
```

الـ flow المفروض:
1. الـ host يقرأ `SDIO_CCCR_SPEED` ويتحقق من `SDIO_SPEED_SHS` bit
2. لو الـ card بتدعم High Speed، الـ host يكتب `SDIO_SPEED_EHS` عشان يفعّله
3. بعد كده يرفع الـ clock frequency

بالـ debugging وجدنا إن الـ CYW43455 كانت بترد إن `SHS=1` (بتدعم High Speed) بس الـ driver مبكتبش الـ `EHS` bit صح.

السبب الجذري: في الـ DT، الـ SDIO controller على AM62x اتكنفجر بـ `sdhci-of-am654` driver بدون `keep-power-in-suspend` ومن غير تحديد الـ timing modes:

```dts
/* القديم — ناقص */
&sdhci1 {
    status = "okay";
    bus-width = <4>;
    vmmc-supply = <&vcc_sd>;
};
```

الـ driver لما مشافش capability flags في الـ DT، افترض SDR12 كـ safe default.

#### الحل

```dts
/* الجديد — صح */
&sdhci1 {
    status = "okay";
    bus-width = <4>;
    vmmc-supply = <&vcc_sd>;
    vqmmc-supply = <&vcc_1v8>;
    ti,otap-del-sel-sdr50 = <0x6>;   /* output tap delay for SDR50 */
    ti,otap-del-sel-sdr104 = <0x5>;  /* output tap delay for SDR104 */
    ti,itap-del-sel-sdr50 = <0x0>;
    mmc-uhs-sdr50;                    /* اعلن دعم SDR50 */
    keep-power-in-suspend;
};
```

بعد كده الـ negotiation اشتغلت:
```
mmc1: new ultra high speed SDR50 SDIO card at address 0001
```

والـ throughput وصل لـ 130 Mbps.

التحقق إضافي عبر الـ CCCR_DRIVE_STRENGTH:
```c
#define SDIO_CCCR_DRIVE_STRENGTH  0x15
#define  SDIO_DTSx_SET_TYPE_B     (0 << SDIO_DRIVE_DTSx_SHIFT)
#define  SDIO_DTSx_SET_TYPE_A     (1 << SDIO_DRIVE_DTSx_SHIFT)
```

على الـ custom board بـ trace lengths طويلة، غيرنا من Type B لـ Type A drive strength وده خفّض الـ signal integrity issues.

#### الدرس المستفاد
**الـ `SDIO_SPEED_SHS`** مجرد capability flag — الـ speed الفعلية بتتحدد لما الـ host يكتب **`SDIO_SPEED_EHS`** في نفس الـ register. في board bring-up، لازم تتحقق من كل الـ CCCR registers يدوياً عبر debugfs قبل ما تقول إن الـ driver "شغال".

```bash
# يدوياً: قراءة CCCR_SPEED من الـ card
cat /sys/kernel/debug/mmc1/mmc1:0001/registers | grep -A2 "CCCR"

# أو عبر mmcblk tool
mmc sdio-speed /dev/mmcblk1
```
## Phase 7: مصادر ومراجع

### توثيق الـ Kernel الرسمي

| المصدر | الوصف |
|--------|-------|
| [`Documentation/driver-api/mmc/`](https://www.kernel.org/doc/html/latest/driver-api/mmc/index.html) | الدليل الرسمي لـ MMC/SD/SDIO في الـ kernel — نقطة البداية الأهم |
| [`include/linux/mmc/sdio.h`](https://elixir.bootlin.com/linux/latest/source/include/linux/mmc/sdio.h) | الملف اللي بنشرحه — تعريفات CCCR وFBR وcommands |
| [`include/linux/mmc/sdio_func.h`](https://elixir.bootlin.com/linux/latest/source/include/linux/mmc/sdio_func.h) | الـ `sdio_func` struct وكل الـ API بتاع الـ driver |
| [`drivers/mmc/core/sdio.c`](https://elixir.bootlin.com/linux/latest/source/drivers/mmc/core/sdio.c) | الـ core SDIO initialization — بيقرا CCCR وFBR فعلياً |
| [`drivers/mmc/core/sdio_io.c`](https://elixir.bootlin.com/linux/latest/source/drivers/mmc/core/sdio_io.c) | تنفيذ `sdio_readb` / `sdio_writeb` وكل I/O functions |
| [`drivers/mmc/core/sdio_irq.c`](https://elixir.bootlin.com/linux/latest/source/drivers/mmc/core/sdio_irq.c) | إدارة الـ interrupts عبر `SDIO_CCCR_INTx` |

---

### مقالات LWN.net

دي أهم المقالات اللي بتغطي تطور الـ SDIO في الـ Linux kernel:

- **[SDIO support coming](https://lwn.net/Articles/242744/)** — مقال Pierre Ossman الأصلي لما أعلن إن الـ SDIO stack جاهز للـ kernel (2007). بيشرح الـ design decisions الأساسية.

- **[MMC updates](https://lwn.net/Articles/253888/)** — تغطية لـ 2.6.24 لما SDIO دخل رسمياً الـ mainline مع SPI support.

- **[What's in mmc.git for 2.6.24](https://lwn.net/Articles/251163/)** — تفاصيل الـ patchset اللي أضاف الـ SDIO framework.

- **[MMC/SD/SDIO card support — Kernel Docs](https://static.lwn.net/kerneldoc/driver-api/mmc/index.html)** — نسخة الـ kernel documentation على LWN.

- **[SDIO driver for Marvell SoCs](https://lwn.net/Articles/319814/)** — مثال real-world لـ SDIO host controller driver.

- **[StarFive SDIO/eMMC driver support](https://lwn.net/Articles/923377/)** — مثال حديث على إضافة SDIO support لـ SoC جديد.

- **[Add support for Meson MX SDIO MMC controller](https://lwn.net/Articles/734836/)** — مثال على Amlogic SoC SDIO controller.

---

### نقاشات الـ Mailing List

- **[sdio: parameterize SDIO FBR register defines](https://lkml.iu.edu/hypermail/linux/kernel/0707.3/3484.html)** — نقاش على LKML سنة 2007 بخصوص إعادة تنظيم الـ `SDIO_FBR_BASE` macros — مهم لفهم ليه الـ design اتعمل بالشكل ده.

- **[lore.kernel.org — MMC/SDIO discussions](https://lore.kernel.org/linux-mmc/)** — الأرشيف الكامل لـ `linux-mmc` mailing list — أهم مكان لمتابعة أي تغيير في الـ SDIO subsystem.

- **[LKML.ORG — Linux Kernel Mailing List Archive](https://lkml.org/)** — للبحث عن threads قديمة بخصوص CCCR وFBR وtiming issues.

---

### kernelnewbies.org

- **[Linux 2.6.24 — SDIO Introduction](https://kernelnewbies.org/Linux_2_6_24)** — الـ release اللي دخل فيه SDIO الـ mainline، مع شرح مبسّط للـ feature.

- **[Linux 2.6.32 — b43 SDIO support](https://kernelnewbies.org/Linux_2_6_32)** — أول driver كبير (b43 WiFi) بيستخدم الـ SDIO framework.

- **[Linux 5.10 — SDIO firmware coredump](https://kernelnewbies.org/Linux_5.10)** — إضافة coredump support للـ SDIO devices.

- **[Linux 6.4 — RTW88 SDIO support](https://kernelnewbies.org/Linux_6.4)** — مثال حديث على SDIO WiFi driver.

---

### elinux.org

- **[Tests:SDIO-with-UHS](https://elinux.org/Tests:SDIO-with-UHS)** — اختبار SDIO بسرعات UHS-I (SDR50/SDR104/DDR50) — مباشرة يرتبط بـ `SDIO_CCCR_UHS` و`SDIO_CCCR_SPEED`.

- **[Tests:SDIO-KS7010](https://elinux.org/Tests:SDIO-KS7010)** — اختبار SDIO WiFi adapter على hardware حقيقي.

- **[Tests:eMMC-HS](https://elinux.org/Tests:eMMC-HS)** — اختبار الـ High-Speed modes اللي بتتحكم فيها `SDIO_CCCR_SPEED`.

---

### Kernel Commits المهمة

| الـ Commit | الوصف |
|-----------|-------|
| [`b5b4b9a`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/include/linux/mmc/sdio.h) | التاريخ الكامل لتغييرات `sdio.h` على kernel.org git |
| [SDIO initial commit by Pierre Ossman](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b5b4b9a) | الإضافة الأصلية للـ SDIO framework في 2.6.24 |

للبحث عن commits بشكل صحيح:
```bash
git log --oneline -- include/linux/mmc/sdio.h
git log --oneline -- drivers/mmc/core/sdio.c
```

---

### كتب مُوصى بيها

#### Linux Device Drivers 3rd Edition (LDD3)
- الكتاب متاح مجاناً: [lwn.net/Kernel/LDD2](https://lwn.net/Kernel/LDD2/)
- **الفصول الأهم للـ SDIO:**
  - Chapter 8: Allocating Memory — لفهم DMA buffers في MMC transfers
  - Chapter 12: PCI Drivers — نفس الـ bus driver model اللي SDIO بتتبعه
  - Chapter 14: The Linux Device Model — الـ `sdio_func` بيتسجل كـ device في هذا الـ model

#### Linux Kernel Development — Robert Love (3rd Ed.)
- Chapter 17: Devices and Modules — الـ device model ومكانة SDIO فيه
- Chapter 11: Timers and Time Management — مهم لفهم الـ timing constraints في SDIO commands

#### Embedded Linux Primer — Christopher Hallinan (2nd Ed.)
- Chapter 12: Embedded Linux Devices — بيغطي SDIO في سياق embedded systems بشكل عملي
- بيشرح إزاي تختار بين USB وSDIO لـ wireless adapters في embedded designs

---

### SDIO Specification الرسمية

**الـ SD Association** هي المرجع الأساسي للـ spec:
- [SD Specifications Part E1 — SDIO Simplified Specification](https://www.sdcard.org/developers/sd-standard-overview/) — بتشرح بالتفصيل كل register في CCCR وFBR وكل command format.
- كل الـ `#define` في `sdio.h` مأخوذة حرفياً من هذه الـ specification.

---

### Elixir Cross-Reference

أداة لا غنى عنها لقراءة الـ kernel code:
- [elixir.bootlin.com — sdio.h](https://elixir.bootlin.com/linux/latest/source/include/linux/mmc/sdio.h) — بتشوف كل مكان في الـ kernel بيستخدم أي `#define` من الملف ده

---

### Search Terms للبحث عن معلومات أكتر

```
linux kernel SDIO CCCR registers
linux SDIO driver tutorial
sdio_func linux kernel
CMD52 CMD53 linux MMC
SDIO interrupt handling linux
linux mmc subsystem internals
Pierre Ossman SDIO kernel
linux SDIO UHS SDR104
sdio_readb sdio_writeb implementation
linux SDIO WiFi driver example
```
## Phase 8: Writing simple module

### الفكرة

الـ `sdio.h` بيعرّف constants بس — مفيش exported functions أو tracepoints جوّاه مباشرةً. بس الـ SDIO subsystem بيكشف functions زي `sdio_claim_irq` اللي بتتعامل مع الـ interrupt enable register (`SDIO_CCCR_IENx = 0x04`) اللي اتعرّف في الهيدر ده. هنعمل **kprobe** على `sdio_claim_irq` عشان نشوف امتى driver بيطلب SDIO interrupt ونطبع معلومات عن الـ function number والـ card.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * sdio_irq_probe.c — kprobe on sdio_claim_irq()
 *
 * sdio_claim_irq() is the function drivers call to register an IRQ handler
 * for a specific SDIO function. It writes to SDIO_CCCR_IENx (0x04) to
 * enable the interrupt for that function on the card.
 */

#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/mmc/sdio_func.h>   /* struct sdio_func */
#include <linux/mmc/card.h>        /* struct mmc_card, struct mmc_cid */
#include <linux/mmc/sdio.h>        /* SDIO_CCCR_IENx and friends */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("SDIO Probe Example");
MODULE_DESCRIPTION("kprobe on sdio_claim_irq to trace SDIO IRQ registrations");

/*
 * sdio_claim_irq signature:
 *   int sdio_claim_irq(struct sdio_func *func, sdio_irq_handler_t *handler);
 *
 * arg0 = struct sdio_func *func
 * arg1 = sdio_irq_handler_t *handler  (function pointer)
 */

static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * regs->di  = first arg  (struct sdio_func *func)   — x86_64 calling convention
     * regs->si  = second arg (sdio_irq_handler_t *handler)
     */
    struct sdio_func *func = (struct sdio_func *)regs->di;

    /* Guard against NULL — defensive check before dereferencing */
    if (!func || !func->card)
        return 0;

    /*
     * func->num  — SDIO function number (1..7), maps to the bit position
     *              that sdio_claim_irq will set in SDIO_CCCR_IENx (0x04).
     * func->card->cid.prod_name — product name from the Card Identification register.
     * func->card->host->class_dev.parent — pointer sanity only, not printed.
     */
    pr_info("sdio_irq_probe: sdio_claim_irq() called\n"
            "  card product : %.8s\n"
            "  manfid       : 0x%04x\n"
            "  func number  : %u  (will set bit %u in SDIO_CCCR_IENx=0x%02x)\n"
            "  irq handler  : %pS\n",
            func->card->cid.prod_name,
            func->card->cid.manfid,
            func->num,
            func->num,          /* bit N in IENx enables function N's IRQ */
            SDIO_CCCR_IENx,
            (void *)regs->si);  /* print symbol name of the IRQ handler */

    return 0;
}

/* kprobe structure — one probe, one symbol */
static struct kprobe kp = {
    .symbol_name = "sdio_claim_irq",
    .pre_handler = handler_pre,
};

static int __init sdio_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("sdio_irq_probe: register_kprobe failed, ret=%d\n", ret);
        return ret;
    }

    pr_info("sdio_irq_probe: kprobe planted on sdio_claim_irq at %p\n",
            kp.addr);
    return 0;
}

static void __exit sdio_probe_exit(void)
{
    /*
     * unregister_kprobe() must be called before module unload —
     * لو مسحناش الـ probe، الـ kernel هيحاول يرجع يجري الـ handler
     * في memory اتفرجت بالفعل → kernel panic.
     */
    unregister_kprobe(&kp);
    pr_info("sdio_irq_probe: kprobe removed from sdio_claim_irq\n");
}

module_init(sdio_probe_init);
module_exit(sdio_probe_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `<linux/kprobes.h>` | بيعرّف `struct kprobe`، `register_kprobe`، `unregister_kprobe` |
| `<linux/mmc/sdio_func.h>` | بيعرّف `struct sdio_func` اللي بيحمل رقم الـ function والـ card pointer |
| `<linux/mmc/card.h>` | بيعرّف `struct mmc_card` و`struct mmc_cid` اللي فيها اسم الكارت والـ manfid |
| `<linux/mmc/sdio.h>` | بيعرّف `SDIO_CCCR_IENx` (0x04) اللي هو الـ register اللي `sdio_claim_irq` بيكتب فيه |

**الـ `sdio.h`** هو بالظبط موضوع الدراسة — بنستخدم `SDIO_CCCR_IENx` منه جوّا الـ `pr_info` عشان نوضّح الرابط بين الـ probe وبين الـ register اللي الـ function دي بتتعامل معاه.

---

#### الـ `handler_pre` callback

```c
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
```

الـ kprobe بيستدعي الـ `pre_handler` **قبل** ما تتنفّذ الـ instruction الأولى في الدالة المستهدفة.
**الـ `pt_regs`** بيحمل حالة الـ registers وقت الدخول — على x86_64 الـ `rdi` = الأرجومنت الأول (`struct sdio_func *func`) والـ `rsi` = التاني (الـ handler pointer).

الكود بيطبع:
- اسم الكارت من `cid.prod_name` — بيجي من الـ CID register اللي الـ host قراه وقت initialization.
- رقم الـ function (`func->num`) — نفس الـ bit اللي هيتسيّت في `SDIO_CCCR_IENx` عشان يفعّل interrupt الـ function دي.
- رمز الـ handler باستخدام `%pS` — بيطبع اسم الـ symbol من الـ kallsyms بدل العنوان الخام.

---

#### الـ `struct kprobe`

```c
static struct kprobe kp = {
    .symbol_name = "sdio_claim_irq",
    .pre_handler = handler_pre,
};
```

الـ `.symbol_name` بيخلي الـ kernel يحلّ العنوان وقت `register_kprobe` — مش محتاجين نحطّ عنوان ثابت. ده أسلم لأن العنوان ممكن يتغيّر بين builds.

---

#### الـ `module_init` / `module_exit`

الـ `register_kprobe` بيكتب **breakpoint instruction** (int3 على x86) عند بداية `sdio_claim_irq` في الـ kernel text.
الـ `unregister_kprobe` لازم يتعمل في الـ `exit` عشان يرجع الـ instruction الأصلية — لو الـ module اتفرّج من الذاكرة والـ probe لسه موجود، أي interrupt تاني هيـcrash الـ kernel فوراً.

---

### Makefile بسيط للتجميع

```makefile
obj-m += sdio_irq_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### تشغيل وملاحظة الـ output

```bash
# تحميل الـ module
sudo insmod sdio_irq_probe.ko

# لما أي SDIO driver (wifi، bluetooth) يتـload ويـclaim interrupt
# هتشوف في dmesg:
sudo dmesg | grep sdio_irq_probe
# sdio_irq_probe: kprobe planted on sdio_claim_irq at ffffffffc0xxxxxx
# sdio_irq_probe: sdio_claim_irq() called
#   card product : RTL8821C
#   manfid       : 0x024c
#   func number  : 1  (will set bit 1 in SDIO_CCCR_IENx=0x04)
#   irq handler  : rtl8821cs_interrupt+0x0/0x40 [rtl8821cs]

# إزالة الـ module
sudo rmmod sdio_irq_probe
```

الـ output ده بيوضّح بالظبط العلاقة: الـ driver طلب enable interrupt للـ function 1، وده معناه إن `sdio_claim_irq` هتكتب `(1 << 1) | (1 << 0)` في الـ register عنوانه `SDIO_CCCR_IENx` = **0x04** — الـ bit 0 هو الـ master enable والـ bit 1 هو enable الـ function 1 نفسها.
