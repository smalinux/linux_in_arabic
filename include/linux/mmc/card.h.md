## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي ليه الملف

الملف `include/linux/mmc/card.h` جزء من الـ **MMC/SD/SDIO Subsystem** — اللي اسمه الرسمي في الـ MAINTAINERS هو:

> MULTIMEDIA CARD (MMC), SECURE DIGITAL (SD) AND SDIO SUBSYSTEM

---

### القصة من الأول

تخيل إنك عندك موبايل وعايز تحط فيه **كارت SD** عشان توسع التخزين. الموبايل بيلتقي بالكارت ده عبر سلوت صغير — بس كيف الـ kernel بيعرف مين هو الكارت ده؟ هو SD عادي؟ eMMC مدمج في اللوحة؟ SDIO (زي كارت WiFi أو Bluetooth)؟

المشكلة إن في أنواع كتير جداً من الكروت:

| النوع | المثال |
|-------|--------|
| **MMC** (MultiMediaCard) | eMMC المدمج في الهواتف والـ SBCs |
| **SD** (Secure Digital) | كروت الكاميرا وأجهزة الـ dashcam |
| **SDIO** | كروت WiFi/BT القديمة زي Broadcom |
| **SD Combo** | كارت بيدعم الاتنين: storage + IO |

وكل واحد من دول بيحتاج الـ kernel يعرف عنه معلومات مختلفة — ايه سرعته، ايه قدرته، ايه الـ quirks اللي فيه. هنا بيجي دور `card.h`.

---

### ايه هو `card.h` بالظبط؟

**الـ `card.h`** هو الـ header الأساسي اللي بيعرّف الـ `struct mmc_card` — يعني الـ "بطاقة تعريف" الكاملة لأي كارت MMC/SD/SDIO في الـ kernel.

تخيل إن الـ kernel شغّال زي موظف استقبال في فندق:
- لما كارت جديد بيتوصل، الموظف بيعمله **ملف كامل** بكل بياناته.
- الملف ده هو `struct mmc_card`.
- كل المعلومات اللي بتتقرأ من الكارت وقت الـ **initialization** بتتحط في الـ fields دي.

---

### الـ Structs الأساسية اللي في الملف

#### 1. `struct mmc_cid` — Card Identification Data
بيانات التعريف الأساسية للكارت — المصنّع، اسم المنتج، رقم السيريال، تاريخ الإنتاج. الـ MMC بيبعت الـ CID لما بيبدأ يتكلم مع الـ host.

```c
struct mmc_cid {
    unsigned int  manfid;       /* manufacturer ID */
    char          prod_name[8]; /* product name */
    unsigned int  serial;       /* serial number */
    unsigned short year;        /* year of manufacture */
    /* ... */
};
```

#### 2. `struct mmc_csd` — Card Specific Data
المواصفات التقنية للكارت: سرعة القراءة، سرعة الكتابة، الـ capacity، الـ erase unit size. ده زي "الكتالوج التقني" للكارت.

#### 3. `struct mmc_ext_csd` — Extended CSD (eMMC فقط)
الـ CSD الموسّع — موجود في الـ eMMC بس (الـ MMCv4 وما فوق). بيوصل لـ 512 byte من المعلومات الإضافية: الـ partitions، الـ cache، الـ HPI، الـ RPMB، الـ Command Queue، الـ firmware version، وعمر الجهاز.

#### 4. `struct sd_scr` — SD Configuration Register
خاص بالـ SD cards — بيحدد bus widths المتاحة والـ SD spec version والـ commands المدعومة.

#### 5. `struct sd_switch_caps` — UHS-I Speed Capabilities
الـ SD cards الحديثة بتدعم UHS-I بسرعات مختلفة (SDR12, SDR25, SDR50, SDR104, DDR50). الـ struct ده بيحفظ ايه الـ speeds المتاحة.

#### 6. `struct sd_uhs2_config` — UHS-II Config
الجيل الجديد من الـ SD interface — بيدعم سرعات لغاية ~1.2 Gb/s عبر interface مختلف تماماً.

#### 7. `struct sdio_cccr` / `struct sdio_cis` — SDIO Card Info
خاص بكروت الـ SDIO — بيصف الـ Common Control Register والـ Card Information Structure.

#### 8. `struct mmc_part` — Physical Partition
الـ eMMC بيقسم نفسه لـ partitions فيزيائية: boot1، boot2، RPMB، وأربع General Purpose partitions.

#### 9. `struct mmc_card` — البطاقة الرئيسية
ده الـ struct الأهم في الملف كله. بيجمع كل حاجة:

```c
struct mmc_card {
    struct mmc_host    *host;       /* من أنا موصّل فيه */
    struct device       dev;        /* Linux device model */
    unsigned int        type;       /* MMC / SD / SDIO / SD_COMBO */
    unsigned int        quirks;     /* مشاكل معروفة في الكارت ده */
    struct mmc_cid      cid;        /* بيانات التعريف */
    struct mmc_csd      csd;        /* المواصفات التقنية */
    struct mmc_ext_csd  ext_csd;    /* extended info (eMMC) */
    struct sd_scr       scr;        /* SD config */
    struct sdio_func   *sdio_func[SDIO_MAX_FUNCS]; /* SDIO functions */
    struct mmc_part     part[MMC_NUM_PHY_PARTITION]; /* partitions */
    /* ... */
};
```

---

### الـ Quirks — مشاكل الكروت الحقيقية

جزء مهم جداً في `card.h` هو الـ **quirks flags** — وده بيحكي قصة حقيقية: مش كل الكروت بتشتغل صح مع الـ standard. فيه كروت من مصنّعين بعينهم بيها bugs:

- **`MMC_QUIRK_INAND_CMD38`** — الـ iNAND devices بتعملها CMD38 بشكل غلط.
- **`MMC_QUIRK_BLK_NO_CMD23`** — بعض الكروت بتفشل لو الـ kernel بعتلها CMD23.
- **`MMC_QUIRK_BROKEN_HPI`** — الـ High Priority Interrupt مش شغال في بعض الكروت.
- **`MMC_QUIRK_BROKEN_SD_CACHE`** — الـ SD cache support مكسور في كروت معينة.

ده بيوضّح إن الـ driver مش بس بيطبّق الـ spec — هو بيتعامل مع واقع صعب فيه hardware بتاعت vendors مختلفين كل واحد بيحتاج معاملة خاصة.

---

### الصورة الكبيرة: ازاي الـ Subsystem بيشتغل

```
+------------------+       +------------------+       +------------------+
|   Block Layer    |       |   SDIO Drivers   |       |   User Space     |
| (reads/writes)   |       | (WiFi, BT, etc.) |       |   (apps, fs)     |
+--------+---------+       +--------+---------+       +--------+---------+
         |                          |                           |
         +-------------+------------+---------------------------+
                       |
              +--------v---------+
              |    MMC Core      |  <-- drivers/mmc/core/
              |  core.c, sd.c,   |
              |  mmc.c, sdio.c   |
              +--------+---------+
                       |
              +--------v---------+
              |   struct mmc_card |  <-- include/linux/mmc/card.h
              |  (هنا بنحفظ كل   |
              |   معلومات الكارت) |
              +--------+---------+
                       |
              +--------v---------+
              |   struct mmc_host |  <-- include/linux/mmc/host.h
              |  (الـ controller)  |
              +--------+---------+
                       |
              +--------v---------+
              |   Host Drivers   |  <-- drivers/mmc/host/
              |  sdhci.c, dw_mmc |
              |  sdhci-msm.c ... |
              +------------------+
```

الـ `card.h` بيقع في المنتصف — هو الجسر اللي بيربط بيانات الـ hardware (اللي الـ host driver بيجيبها) بالـ software اللي فوق (core logic, block driver, SDIO functions).

---

### الملفات المهمة اللي المفروض تعرفها

#### Headers (include/linux/mmc/)
| الملف | الدور |
|-------|-------|
| `card.h` | تعريف `struct mmc_card` وكل الـ card data structs |
| `host.h` | تعريف `struct mmc_host` — الـ controller نفسه |
| `core.h` | الـ core API: `mmc_request`، `mmc_command`، `mmc_data` |
| `mmc.h` | تعريف MMC commands، الـ CSD/EXT_CSD bit fields |
| `sd.h` | تعريف SD-specific commands وبيانات الـ registers |
| `sdio.h` | SDIO commands وتعريفات الـ CCCR |
| `sdio_func.h` | تعريف `struct sdio_func` — كل function في كارت SDIO |

#### Core Driver (drivers/mmc/core/)
| الملف | الدور |
|-------|-------|
| `core.c` | نقطة البداية — الـ main MMC core logic |
| `mmc.c` | initialization لكروت الـ eMMC |
| `sd.c` | initialization لكروت الـ SD |
| `sdio.c` | initialization لكروت الـ SDIO |
| `block.c` | الـ block device interface (ازاي الـ FS تقرأ وتكتب) |
| `bus.c` | الـ MMC bus في الـ Linux device model |

#### Host Drivers (drivers/mmc/host/)
| الملف | الدور |
|-------|-------|
| `sdhci.c` | الـ generic SDHCI driver (أشهر host controller spec) |
| `sdhci-msm.c` | Qualcomm Snapdragon host |
| `dw_mmc.c` | DesignWare MMC host (Rockchip, HiSilicon, ...) |
| `sdhci-of-arasan.c` | Arasan SDHCI (Xilinx, ...) |
## Phase 2: شرح الـ MMC/SD/SDIO Framework

### المشكلة اللي بيحلها الـ Subsystem ده

في الأنظمة المدمجة، عندك أنواع كتير من storage cards وـ wireless peripherals بتتكلم كلها على نفس الـ physical bus (الـ MMC bus):

- **eMMC** — flash storage مدمج على الـ PCB، موجود في كل Android phone و embedded SoC
- **SD card** — removable storage قابل للنزع في أي وقت
- **SDIO card** — مش storage خالص، ده peripheral بيستخدم نفس الـ bus عشان يوصل devices زي WiFi chips وـ Bluetooth modules وـ GPS receivers

المشكلة الحقيقية: كل نوع من الأنواع دي ليه:
1. **Protocol مختلف** للـ initialization وـ negotiation
2. **Register map مختلف** (CID, CSD, EXT_CSD, SCR, CCCR...)
3. **Capabilities مختلفة** (speed modes, bus widths, partitioning, command queuing)
4. **Hardware quirks** — devices معينة بيكسر الـ spec في حاجات معينة

لو كل driver كتب الـ low-level protocol logic بتاعه من الأول، هيبقى عندك كود متكرر وـ bugs مكررة.

---

### الحل — Architecture الـ MMC Framework

الـ kernel بيحل المشكلة دي بـ **3-layer separation**:

```
+--------------------------------------------------+
|          Block Layer / SDIO Bus Driver           |  ← Consumers (مثلاً mmc_block, cfg80211)
+--------------------------------------------------+
|                  MMC Core Layer                  |  ← المنطق المشترك، الـ state machine
|   card detection | initialization | power mgmt   |
+--------------------------------------------------+
|              MMC Host Controller Driver          |  ← Provider (SDHCI, i.MX USDHC, Arasan...)
+--------------------------------------------------+
|               Hardware (MMC/SD Bus)              |  ← الـ physical signals
+--------------------------------------------------+
```

كل layer بتعمل حاجة محددة:

| Layer | المسؤولية |
|-------|-----------|
| Host Controller Driver | بيتكلم مع الـ hardware registers، بيرسل وبيستقبل الـ bits |
| MMC Core | بيعمل card detection، initialization، power management، وبيبني الـ `mmc_card` struct |
| Block/SDIO Driver | بيستخدم الـ card كـ block device أو كـ SDIO bus device |

---

### التشبيه الحقيقي — نظام الجوازات في المطار

تخيل مطار بيستقبل مسافرين من جنسيات مختلفة:

```
المسافر (SD card / eMMC / SDIO card)
   ↓
موظف الاستقبال (MMC Core)
   ↓ يقرأ الجواز (CID/CSD) ويسجل البيانات
   ↓ يحدد نوع التأشيرة (MMC_TYPE_MMC / SD / SDIO)
   ↓ يملأ ملف الشخص (mmc_card struct)
   ↓
يوجهه للمنطقة الصح (Block Device / SDIO Bus)
```

الآن بالتفصيل الكامل:

| التشبيه | الكود الفعلي |
|---------|-------------|
| جواز السفر | `struct mmc_cid` — manufacturer ID, product name, serial number |
| شروط الدخول (ما تقدر تعمله) | `struct mmc_csd` — capacity, max data rate, block sizes |
| التأشيرة الموسعة (خواص إضافية) | `struct mmc_ext_csd` — eMMC v4+ features, partitions, command queue |
| نوع الجنسية | `card->type` = `MMC_TYPE_MMC / SD / SDIO / SD_COMBO` |
| ملف المسافر الكامل | `struct mmc_card` — الـ struct الرئيسي |
| المطار نفسه (البنية التحتية) | `struct mmc_host` — الـ host controller |
| الموظف الأمني | quirks system — بيتعامل مع "المسافر المشاغب" اللي بيكسر القواعد |
| صالة الوصول | `struct device dev` — تسجيل الـ card في الـ Linux device model |

---

### الـ Core Abstraction — الـ `mmc_card`

الـ **`struct mmc_card`** هو المحور الرئيسي للـ subsystem. ده الـ object اللي بيمثل الـ physical card بعد ما الـ core يعمل لها initialization كامل.

```c
struct mmc_card {
    struct mmc_host     *host;    /* مين بيتحكم فيها؟ */
    struct device        dev;     /* تسجيلها في device model */
    unsigned int         type;    /* MMC / SD / SDIO / COMBO */
    unsigned int         state;   /* حالتها الحالية */
    unsigned int         quirks;  /* exceptions للـ spec */

    /* --- Raw registers من الـ hardware مباشرة --- */
    u32  raw_cid[4];
    u32  raw_csd[4];
    u32  raw_scr[2];
    u32  raw_ssr[16];

    /* --- Parsed versions من نفس الـ registers --- */
    struct mmc_cid       cid;
    struct mmc_csd       csd;
    struct mmc_ext_csd   ext_csd;   /* eMMC فقط */
    struct sd_scr        scr;        /* SD فقط */
    struct sd_ssr        ssr;        /* SD فقط */
    struct sd_switch_caps sw_caps;   /* SD CMD6 */

    /* --- SDIO-specific --- */
    unsigned int         sdio_funcs;
    struct sdio_cccr     cccr;
    struct sdio_cis      cis;
    struct sdio_func    *sdio_func[SDIO_MAX_FUNCS];

    /* --- Physical partitions (eMMC) --- */
    struct mmc_part      part[MMC_NUM_PHY_PARTITION];
    unsigned int         nr_parts;
};
```

---

### الـ Structs وعلاقتها ببعض

```
struct mmc_host (host controller)
       │
       │  1:N
       ▼
struct mmc_card (logical card representation)
       │
       ├──── struct mmc_cid       (Who are you? Manufacturer, Serial, Product Name)
       │
       ├──── struct mmc_csd       (What can you do? Capacity, Speed, Block sizes)
       │
       ├──── struct mmc_ext_csd   (eMMC Advanced features: partitions, HPI, CmdQ)
       │
       ├──── struct sd_scr        (SD: bus width support, spec version)
       ├──── struct sd_ssr        (SD: status register - AU size, erase info)
       ├──── struct sd_switch_caps (SD: HS/UHS speed modes, driver strength)
       ├──── struct sd_ext_reg    (SD v6+: Power mgmt & Perf enhancement)
       ├──── struct sd_uhs2_config (UHS-II: new high-speed serial interface)
       │
       ├──── struct sdio_cccr     (SDIO: Common Card Capability Register)
       ├──── struct sdio_cis      (SDIO: Card Information Structure)
       └──── struct sdio_func[7]  (SDIO: up to 7 function devices per card)
                    │
                    └── struct sdio_func (WiFi func, BT func, etc.)
                               │
                               └── struct device dev  (registered in sysfs)
```

---

### شرح كل Struct بالتفصيل

#### `struct mmc_cid` — Card Identification Register

الـ **CID** ده register بـ 128-bit موجود في كل card. بيُقرأ مرة واحدة بس أثناء الـ initialization باستخدام CMD2.

```c
struct mmc_cid {
    unsigned int  manfid;        /* Manufacturer ID — مين صنعها (Samsung=0x15, Kingston=0x41) */
    char          prod_name[8];  /* Product name — مثلاً "HAG4a2" */
    unsigned char prv;           /* Product revision — BCD format */
    unsigned int  serial;        /* Serial number */
    unsigned short oemid;        /* OEM/Application ID */
    unsigned short year;         /* Manufacturing year */
    unsigned char hwrev;         /* Hardware revision */
    unsigned char fwrev;         /* Firmware revision */
    unsigned char month;         /* Manufacturing month */
};
```

الـ `raw_cid[4]` في الـ `mmc_card` هو الـ 128 bit كما جاء من الـ hardware. الـ MMC core بيعمل parse له ويحطه في الـ `cid` struct المنظم.

#### `struct mmc_csd` — Card Specific Data

الـ **CSD** بيحتوي على المعلومات التشغيلية للـ card. بيُقرأ بـ CMD9.

```c
struct mmc_csd {
    unsigned char  structure;      /* CSD version (v1 or v2 for SD) */
    unsigned short cmdclass;       /* supported command classes (bitmask) */
    unsigned int   c_size;         /* raw capacity encoding */
    unsigned int   max_dtr;        /* max data transfer rate في Hz */
    unsigned int   erase_size;     /* erase unit size in sectors */
    unsigned int   read_blkbits;   /* log2 of read block size */
    unsigned int   write_blkbits;  /* log2 of write block size */
    sector_t       capacity;       /* total capacity in sectors */
    unsigned int   read_partial:1; /* supports partial read? */
    unsigned int   dsr_imp:1;      /* DSR register implemented? */
    /* ... */
};
```

الـ `cmdclass` بيمثل الـ command classes اللي الـ card بتدعمها. مثلاً Class 2 = block read, Class 4 = block write, Class 6 = write protection.

#### `struct mmc_ext_csd` — Extended CSD (eMMC فقط)

الـ **EXT_CSD** ده register بـ 512 bytes خاص بالـ eMMC (MMC v4+). بيُقرأ بـ CMD8 (SEND_EXT_CSD). ده أهم struct في الـ eMMC ecosystem.

```c
struct mmc_ext_csd {
    /* Speed modes */
    unsigned int  hs_max_dtr;      /* HS52: 52 MHz */
    unsigned int  hs200_max_dtr;   /* HS200: 200 MHz — للـ eMMC 5.0 */

    /* Partitioning */
    u8   part_config;              /* active partition (boot1/boot2/user) */
    bool partition_setting_completed;
    u8   raw_partition_support;   /* byte 160 في الـ EXT_CSD table */

    /* RPMB — Replay Protected Memory Block */
    u8   raw_rpmb_size_mult;      /* RPMB partition size = mult * 128KB */

    /* Background Operations */
    bool bkops;                   /* hardware supports BKOPS? */
    bool auto_bkops_en;           /* auto BKOPS enabled? */

    /* High Priority Interrupt */
    bool hpi_en;                  /* HPI enabled? */
    unsigned int hpi_cmd;         /* CMD12 or CMD13 لإلغاء operation */

    /* Command Queuing */
    bool cmdq_support;            /* hardware supports CmdQ? */
    bool cmdq_en;                 /* CmdQ enabled right now? */
    unsigned int cmdq_depth;      /* queue depth (max 32) */

    /* Cache */
    unsigned int cache_size;      /* cache size in KB */
    u8   cache_ctrl;              /* cache flush control */

    /* Device health */
    u8   pre_eol_info;                    /* 0=unknown, 1=normal, 2=warning, 3=urgent */
    u8   device_life_time_est_typ_a;      /* type A wear (0-11 scale) */
    u8   device_life_time_est_typ_b;      /* type B wear */

    /* Firmware upgrade */
    bool ffu_capable;             /* Field Firmware Update supported? */
    u8   fwrev[MMC_FIRMWARE_LEN]; /* current firmware version */
};
```

**ملاحظة مهمة**: كل field في الـ `raw_*` prefix (زي `raw_pwr_cl_52_195`) هو القيمة الخام كما جاءت من الـ register في offset معين. الـ parsed values (زي `hs_max_dtr`) هي اللي الـ core بيحسبها من الـ raw values. ده pattern ثابت في الـ MMC core.

#### `struct sd_scr` — SD Configuration Register

الـ **SCR** خاص بالـ SD cards (مش eMMC). بيُقرأ عن طريق ACMD51.

```c
struct sd_scr {
    unsigned char sda_vsn;     /* SD spec version */
    unsigned char sda_spec3;   /* supports SD spec 3.0? */
    unsigned char sda_spec4;   /* supports SD spec 4.0? */
    unsigned char bus_widths;  /* SD_SCR_BUS_WIDTH_1 | SD_SCR_BUS_WIDTH_4 */
    unsigned char cmds;        /* CMD20/23/48/58 support flags */
};
```

الـ `bus_widths` هنا بيحدد إذا كانت الـ card تدعم 4-bit bus mode (أسرع من 1-bit).

#### `struct sd_switch_caps` — Speed Mode Capabilities

ده بيتملى بعد قراءة الـ SD Function Switch (CMD6) بـ mode 0 (query).

```c
struct sd_switch_caps {
    unsigned int hs_max_dtr;     /* High Speed max: 50 MHz */
    unsigned int uhs_max_dtr;    /* UHS-I max: up to 208 MHz */
    unsigned int sd3_bus_mode;   /* supported UHS bus modes (bitmask) */
    unsigned int sd3_drv_type;   /* driver strength types A/B/C/D */
    unsigned int sd3_curr_limit; /* max current: 200/400/600/800 mA */
};
```

الـ UHS-I modes (SDR12, SDR25, SDR50, DDR50, SDR104) كلها مُعرّفة هنا. الـ core بيختار أعلى mode بيدعمه كل من الـ host والـ card.

#### `struct sdio_cccr` — Common Card Capability Register

الـ **CCCR** خاص بالـ SDIO cards (زي WiFi chips). بيحتوي على capabilities مشتركة لكل الـ functions على الـ card.

```c
struct sdio_cccr {
    unsigned int sdio_vsn;           /* SDIO spec version */
    unsigned int sd_vsn;             /* SD spec version */
    unsigned int multi_block:1;      /* supports multi-block transfers? */
    unsigned int high_speed:1;       /* supports High Speed? */
    unsigned int wide_bus:1;         /* supports 4-bit bus? */
    unsigned int enable_async_irq:1; /* async interrupt supported? */
    unsigned int disable_cd:1;       /* disable card detect pull-up? */
};
```

#### `struct sdio_func` — Function Device

في الـ SDIO، الـ card الواحدة ممكن يكون فيها **حتى 7 functions مستقلة**. كل function هي device منفصلة في الـ Linux device model.

```c
struct sdio_func {
    struct mmc_card    *card;        /* الـ card اللي الـ func دي جزء منها */
    struct device       dev;         /* device في sysfs */
    unsigned int        num;         /* function number (1-7) */
    unsigned char       class;       /* SDIO standard class (BT, WiFi, ...) */
    unsigned short      vendor;
    unsigned short      device;
    unsigned            max_blksize;
    unsigned            cur_blksize;
    sdio_irq_handler_t *irq_handler; /* callback لما الـ func تطلع interrupt */
};
```

مثال حقيقي: chip زي **Marvell SD8787** بيحتوي على function 1 = WiFi وfunction 2 = Bluetooth، كلهم على نفس الـ SDIO card، تحت نفس الـ `mmc_card`.

---

### الـ Quirks System — التعامل مع الـ Non-Compliant Devices

الـ **quirks** هم flags بيوصف devices اللي بتكسر الـ spec في حاجات معينة. الـ MMC core بيعمل match على الـ CID ويضيف الـ quirks المناسبة.

```c
#define MMC_QUIRK_INAND_CMD38          (1<<6)   /* iNAND: CMD38 مكسور */
#define MMC_QUIRK_BLK_NO_CMD23         (1<<7)   /* بعض الكروت: CMD23 بيعمل مشاكل */
#define MMC_QUIRK_BROKEN_HPI           (1<<13)  /* الـ HPI feature مش شغالة صح */
#define MMC_QUIRK_BROKEN_SD_CACHE      (1<<15)  /* SD cache feature مكسورة */
#define MMC_QUIRK_BROKEN_CACHE_FLUSH   (1<<16)  /* flush بقدر تعمله بس بعد الكتابة */
#define MMC_QUIRK_NO_UHS_DDR50_TUNING  (1<<18)  /* DDR50 tuning بيفشل */
```

الـ `quirk_max_rate` بيسمح بتحديد سقف للـ clock rate لـ card معينة بدل ما تشتغل بأعلى سرعة وتفشل.

---

### الـ Physical Partitions في eMMC

```
eMMC Flash
┌─────────────────────────────────────────────────┐
│  Boot Partition 1  │  Boot Partition 2           │  ← 2 boot areas (read-only by default)
├────────────────────┴─────────────────────────────┤
│  RPMB (Replay Protected Memory Block)            │  ← secure storage للـ Widevine keys
├──────────────────────────────────────────────────┤
│  GP1 │ GP2 │ GP3 │ GP4                           │  ← General Purpose partitions (optional)
├──────┴─────┴─────┴───────────────────────────────┤
│  User Data Area (الجزء الأكبر)                   │  ← الـ /data, /system, etc.
└──────────────────────────────────────────────────┘
```

كل partition بـ `struct mmc_part`:

```c
struct mmc_part {
    u64          size;        /* حجم الـ partition بالـ bytes */
    unsigned int part_cfg;    /* رقم الـ partition في EXT_CSD */
    char         name[20];    /* "boot0", "boot1", "rpmb", "gp0"... */
    bool         force_ro;    /* boot partitions بتبقى RO بـ default */
    unsigned int area_type;   /* MMC_BLK_DATA_AREA_BOOT / GP / RPMB / MAIN */
};
```

الـ `MMC_NUM_PHY_PARTITION = 7` = 2 boot + 4 GP + 1 RPMB.

---

### الـ Inline Helpers وأهميتها

```c
/* بدل ما تكتب card->ext_csd.data_sector_size == 4096 في كل مكان */
static inline bool mmc_large_sector(struct mmc_card *card) {
    return card->ext_csd.data_sector_size == 4096;
}

/* type checking macros */
#define mmc_card_mmc(c)     ((c)->type == MMC_TYPE_MMC)
#define mmc_card_sd(c)      ((c)->type == MMC_TYPE_SD)
#define mmc_card_sdio(c)    ((c)->type == MMC_TYPE_SDIO)
```

الـ large sector (4KB بدل 512 bytes) مهم لأن الـ eMMC الحديثة بتستخدم 4KB sectors، ولو الـ driver بعتلها request بـ 512-byte boundary هيفشل.

---

### الـ Request Pipeline — إزاي البيانات بتتنقل

> **ملاحظة**: الـ subsystem ده بيتعامل مع `struct mmc_request` المعرّف في `mmc/core.h`. الـ block layer بيبني الـ request ده وبيديه للـ MMC core اللي بيوديه للـ host controller.

```
Block Layer (bio)
      ↓
mmc_blk_mq_queue_rq()
      ↓ بيبني mmc_request
struct mmc_request {
    struct mmc_command *sbc;   /* CMD23: SET_BLOCK_COUNT (optional) */
    struct mmc_command *cmd;   /* CMD17/18/24/25: read/write */
    struct mmc_data    *data;  /* scatter-gather list للـ data */
    struct mmc_command *stop;  /* CMD12: STOP_TRANSMISSION */
    void (*done)(mrq);         /* callback عند الانتهاء */
}
      ↓
mmc_wait_for_req() → host->ops->request(host, mrq)
      ↓
Host Controller Driver (SDHCI / Arasan / i.MX USDHC)
      ↓
Physical MMC Bus
```

---

### ما يملكه الـ Framework vs ما يفوّضه

| الـ MMC Core يملك | يفوّض للـ Host Driver |
|------------------|----------------------|
| Card type detection | إرسال الـ commands على الـ bus |
| CID/CSD/EXT_CSD parsing | DMA setup والـ scatter-gather handling |
| Power state machine | Clock rate control |
| Quirks matching وتطبيقها | Voltage switching (1.8V / 3.3V) |
| Partition management | Tuning procedure (للـ HS200/HS400) |
| Command Queue management | الـ timing وـ signal integrity |
| SDIO IRQ routing | الـ hardware resets |

---

### الـ Device Model Integration

الـ `struct device dev` داخل الـ `mmc_card` هو اللي بيربط الـ card بالـ Linux Device Model. ده يعني:

- الـ card بتظهر في `/sys/bus/mmc/devices/`
- الـ udev بيشوفها ويعمل hotplug events
- الـ power management (runtime PM, system suspend) بيشتغل أوتوماتيك
- الـ debugfs entries (`card->debugfs_root`) بتتعمل تحت `/sys/kernel/debug/mmc*/`

الـ `complete_wq` (private workqueue) بيُستخدم لإكمال الـ async operations بدل الـ system workqueue عشان يتجنب الـ deadlocks في الـ storage path.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Macros والـ Enums — Cheatsheet

#### نوع الـ Card (`mmc_card.type`)

| Macro | قيمة | المعنى |
|---|---|---|
| `MMC_TYPE_MMC` | 0 | كارد eMMC/MMC عادي |
| `MMC_TYPE_SD` | 1 | كارد SD عادي |
| `MMC_TYPE_SDIO` | 2 | كارد SDIO (I/O فقط) |
| `MMC_TYPE_SD_COMBO` | 3 | SD Combo (I/O + Memory) |

#### الـ Quirks (`mmc_card.quirks`) — workarounds للكروت المعيبة

| Macro | Bit | المشكلة اللي بتحلها |
|---|---|---|
| `MMC_QUIRK_LENIENT_FN0` | 0 | تسمح بـ write على SDIO FN0 خارج نطاق VS CCCR |
| `MMC_QUIRK_BLKSZ_FOR_BYTE_MODE` | 1 | يستخدم `cur_blksize` في byte mode |
| `MMC_QUIRK_NONSTD_SDIO` | 2 | كارت SDIO مش standard — missing CIA registers |
| `MMC_QUIRK_NONSTD_FUNC_IF` | 4 | interfaces غير standard للـ functions |
| `MMC_QUIRK_DISABLE_CD` | 5 | يفصل مقاومة CD/DAT[3] |
| `MMC_QUIRK_INAND_CMD38` | 6 | أجهزة iNAND فيها bug في CMD38 |
| `MMC_QUIRK_BLK_NO_CMD23` | 7 | يتجنب CMD23 في multiblock عادي |
| `MMC_QUIRK_BROKEN_BYTE_MODE_512` | 8 | يتجنب إرسال 512 byte في byte mode |
| `MMC_QUIRK_LONG_READ_TIME` | 9 | وقت قراءة البيانات أطول مما تقوله الـ CSD |
| `MMC_QUIRK_SEC_ERASE_TRIM_BROKEN` | 10 | يتجنب secure erase/trim |
| `MMC_QUIRK_BROKEN_IRQ_POLLING` | 11 | polling على SDIO_CCCR_INTx يولد IRQ وهمي |
| `MMC_QUIRK_TRIM_BROKEN` | 12 | يتجنب trim كلياً |
| `MMC_QUIRK_BROKEN_HPI` | 13 | يعطل HPI المكسور |
| `MMC_QUIRK_BROKEN_SD_DISCARD` | 14 | يعطل SD discard المكسور |
| `MMC_QUIRK_BROKEN_SD_CACHE` | 15 | يعطل SD cache المكسور |
| `MMC_QUIRK_BROKEN_CACHE_FLUSH` | 16 | ما يفلش الـ cache قبل ما الـ write يخلص |
| `MMC_QUIRK_BROKEN_SD_POWEROFF_NOTIFY` | 17 | يعطل poweroff notify المكسور |
| `MMC_QUIRK_NO_UHS_DDR50_TUNING` | 18 | يعطل DDR50 tuning |

#### سرعات الـ eMMC (`mmc_ext_csd` macros)

| Macro | قيمة | الوضع |
|---|---|---|
| `MMC_HIGH_26_MAX_DTR` | 26 MHz | High Speed 26 |
| `MMC_HIGH_52_MAX_DTR` | 52 MHz | High Speed 52 |
| `MMC_HIGH_DDR_MAX_DTR` | 52 MHz | DDR52 |
| `MMC_HS200_MAX_DTR` | 200 MHz | HS200/HS400 |

#### سرعات الـ SD (`sd_switch_caps` macros)

| Macro | قيمة | الوضع |
|---|---|---|
| `DEFAULT_SPEED_MAX_DTR` | 25 MHz | SD Default |
| `HIGH_SPEED_MAX_DTR` | 50 MHz | SD HS |
| `UHS_SDR25_MAX_DTR` | 50 MHz | UHS SDR25 |
| `UHS_SDR50_MAX_DTR` | 100 MHz | UHS SDR50 |
| `UHS_SDR104_MAX_DTR` | 208 MHz | UHS SDR104 |
| `UHS_DDR50_MAX_DTR` | 50 MHz | UHS DDR50 |

#### أنواع الـ Physical Partitions في eMMC

| Macro | Bit | الـ Partition |
|---|---|---|
| `MMC_BLK_DATA_AREA_MAIN` | 0 | الـ main user area |
| `MMC_BLK_DATA_AREA_BOOT` | 1 | boot partition (1 أو 2) |
| `MMC_BLK_DATA_AREA_GP` | 2 | General Purpose Partition |
| `MMC_BLK_DATA_AREA_RPMB` | 3 | Replay Protected Memory Block |

#### SD Extension Register Features

| Macro | وظيفة | نوع |
|---|---|---|
| `SD_EXT_POWER_OFF_NOTIFY` | إشعار عند إيقاف التشغيل | Power Mgmt |
| `SD_EXT_POWER_SUSTENANCE` | إبقاء الطاقة | Power Mgmt |
| `SD_EXT_POWER_DOWN_MODE` | وضع إيقاف الطاقة | Power Mgmt |
| `SD_EXT_PERF_FX_EVENT` | Fixed Event | Performance |
| `SD_EXT_PERF_CARD_MAINT` | Card Maintenance | Performance |
| `SD_EXT_PERF_HOST_MAINT` | Host Maintenance | Performance |
| `SD_EXT_PERF_CACHE` | Cache دعم | Performance |
| `SD_EXT_PERF_CMD_QUEUE` | Command Queue دعم | Performance |

---

### كل الـ Structs المهمة

---

#### 1. `struct mmc_cid`

**الغرض:** تخزين بيانات هوية الكارد — الـ **Card Identification Register** — اللي بيتبعت لما الـ host يبعت CMD2.

| Field | نوع | المعنى |
|---|---|---|
| `manfid` | `unsigned int` | Manufacturer ID — معرّف الشركة الصانعة |
| `prod_name[8]` | `char[]` | اسم المنتج (ASCII، 8 حروف لـ MMC، 5 لـ SD) |
| `prv` | `unsigned char` | Product Revision (major.minor نيبل) |
| `serial` | `unsigned int` | Serial Number — 32-bit فريد |
| `oemid` | `unsigned short` | OEM/Application ID |
| `year` | `unsigned short` | سنة التصنيع |
| `hwrev` | `unsigned char` | Hardware Revision |
| `fwrev` | `unsigned char` | Firmware Revision (MMC فقط) |
| `month` | `unsigned char` | شهر التصنيع |

**الارتباط:** `mmc_card.cid` يحمل نسخة parsed منها، و`mmc_card.raw_cid[4]` يحمل الـ raw bits.

---

#### 2. `struct mmc_csd`

**الغرض:** الـ **Card Specific Data** — بيانات هيكل ذاكرة الكارد: السعة، سرعة القراءة/الكتابة، أحجام الـ blocks.

| Field | المعنى |
|---|---|
| `structure` | إصدار هيكل الـ CSD (v1 أو v2) |
| `mmca_vsn` | MMC spec version |
| `cmdclass` | الـ command classes المدعومة (bitmask) |
| `taac_clks` / `taac_ns` | Time Access — زمن الوصول (clocks + ns) |
| `c_size` | حجم الكارد (مُرمَّز) |
| `r2w_factor` | نسبة وقت الكتابة لوقت القراءة |
| `max_dtr` | أقصى Data Transfer Rate |
| `erase_size` | حجم وحدة المسح (بالـ sectors) |
| `wp_grp_size` | حجم مجموعة الـ Write Protect |
| `read_blkbits` / `write_blkbits` | log2 لحجم الـ block |
| `capacity` | السعة الكلية (مُحسوبة كـ `sector_t`) |
| `read_partial:1` | يدعم قراءة partial block |
| `read_misalign:1` | الـ block يمكن يتقاطع مع حدود سجلات |
| `write_partial:1` | يدعم كتابة partial block |
| `dsr_imp:1` | هل الـ DSR register موجود |

**الارتباط:** `mmc_card.csd` + `mmc_card.raw_csd[4]`.

---

#### 3. `struct mmc_ext_csd`

**الغرض:** الـ **Extended CSD** — 512 byte خاصة بـ eMMC v4+ — تحتوي على إعدادات متقدمة جداً: partitioning، HPI، BKOPS، Command Queue، cache، وغيرها.

أهم الفيلدات:

| Field | المعنى |
|---|---|
| `rev` | إصدار الـ EXT_CSD |
| `part_config` | تهيئة الـ partition النشطة حالياً |
| `cache_ctrl` | تفعيل/تعطيل الـ cache |
| `rst_n_function` | هل RST_n pin مفعل |
| `hs_max_dtr` | أقصى سرعة في HS mode |
| `hs200_max_dtr` | أقصى سرعة في HS200 |
| `sectors` | إجمالي عدد الـ sectors |
| `hc_erase_size` | حجم وحدة المسح في HC mode |
| `partition_setting_completed` | هل تم تطبيق الـ partitioning نهائياً |
| `enhanced_area_offset` / `enhanced_area_size` | نطاق الـ Enhanced Area |
| `cache_size` | حجم الـ cache الداخلي (KB) |
| `hpi_en` / `hpi` / `hpi_cmd` | دعم وتفعيل الـ HPI (High Priority Interrupt) |
| `bkops` / `man_bkops_en` / `auto_bkops_en` | Background Operations |
| `cmdq_en` / `cmdq_support` / `cmdq_depth` | Command Queue Engine |
| `fwrev[8]` | نسخة الـ Firmware |
| `pre_eol_info` | مؤشر اقتراب نهاية العمر الافتراضي |
| `device_life_time_est_typ_a/b` | تقدير العمر المتبقي للـ flash |
| `feature_support` | features flag — حالياً `MMC_DISCARD_FEATURE` |
| `raw_*` fields | القيم الخام مباشرة من الـ register قبل التحويل |

**الارتباط:** `mmc_card.ext_csd` — موجود فقط لو `mmc_card_mmc(card)` صح.

---

#### 4. `struct sd_scr`

**الغرض:** الـ **SD Configuration Register** — 64-bit register في SD يعطي إمكانيات الكارد.

| Field | المعنى |
|---|---|
| `sda_vsn` | SD spec version |
| `sda_spec3/4/x` | دعم spec versions الأحدث |
| `bus_widths` | bus widths المدعومة (`SD_SCR_BUS_WIDTH_1/4`) |
| `cmds` | الـ commands الاختيارية المدعومة (CMD20/23/48/58) |

---

#### 5. `struct sd_ssr`

**الغرض:** الـ **SD Status Register** — 512-bit — معلومات أداء وخصائص الكارد.

| Field | المعنى |
|---|---|
| `au` | Allocation Unit size (بالـ sectors) |
| `erase_timeout` | زمن المسح (ms) |
| `erase_offset` | offset زمن المسح (ms) |

---

#### 6. `struct sd_switch_caps`

**الغرض:** نتيجة CMD6 (SWITCH_FUNC) — يخبر الـ host بالـ bus modes وأنواع الـ driver وحدود التيار.

| Field | المعنى |
|---|---|
| `hs_max_dtr` | أقصى سرعة في HS mode |
| `uhs_max_dtr` | أقصى سرعة في UHS mode |
| `sd3_bus_mode` | bitmask لـ modes المدعومة (SDR12 حتى DDR50) |
| `sd3_drv_type` | أنواع الـ driver المدعومة (A/B/C/D) |
| `sd3_curr_limit` | حد التيار الأقصى المدعوم (200 حتى 800 mA) |

---

#### 7. `struct sd_ext_reg`

**الغرض:** تمثيل Extension Register في SD spec — لكل function (Power Management أو Performance Enhancement).

| Field | المعنى |
|---|---|
| `fno` | رقم الـ function |
| `page` | رقم الـ page في register space |
| `offset` | الـ offset داخل الـ page |
| `rev` | revision الـ function |
| `feature_enabled` | bitmask للـ features المفعّلة |
| `feature_support` | bitmask للـ features المدعومة |

**الاستخدام:** `mmc_card.ext_power` للـ PM، و`mmc_card.ext_perf` للأداء.

---

#### 8. `struct sd_uhs2_config`

**الغرض:** إعدادات واجهة **UHS-II** (الجيل الجديد من SD اللي بيستخدم serial link بدل parallel bus).

| الفيلدات | المعنى |
|---|---|
| `node_id` | معرّف العقدة في الـ UHS-II topology |
| `n_fcu` | عدد الـ FCU (Flow Control Units) |
| `maxblk_len` | أقصى طول للـ block |
| `n_lanes` | عدد الـ lanes (1 أو 2) |
| `can_hibernate` | يدعم hibernate mode |
| `n_lss_sync/dir` | عدد LSS frames |
| `speed_range_set` | Speed Range المضبوط فعلاً |
| `*_set` fields | القيم المطبّقة فعلاً بعد negotiation |

---

#### 9. `struct sdio_cccr`

**الغرض:** الـ **Common Card Control Register** — أول حاجة الـ host بيقراها من كارت SDIO — بيخبره بإمكانيات الـ card-level.

| Field | المعنى |
|---|---|
| `sdio_vsn` | إصدار SDIO spec |
| `sd_vsn` | إصدار SD spec |
| `multi_block:1` | يدعم multi-block transfer |
| `low_speed:1` | كارت low-speed |
| `wide_bus:1` | يدعم 4-bit bus |
| `high_power:1` | يدعم high-power mode |
| `high_speed:1` | يدعم high-speed mode |
| `disable_cd:1` | CD pin معطّل |
| `enable_async_irq:1` | يدعم Async IRQ (SDIO 3.0+) |

---

#### 10. `struct sdio_cis`

**الغرض:** الـ **Common Information Structure** — tuple من الـ CIS الخاصة بالـ card ككل (وليس function بعينها).

| Field | المعنى |
|---|---|
| `vendor` | Vendor ID |
| `device` | Device ID |
| `blksize` | حجم الـ block الافتراضي |
| `max_dtr` | أقصى Data Transfer Rate |

---

#### 11. `struct mmc_part`

**الغرض:** تمثيل **Physical Partition** واحدة في eMMC — boot/GP/RPMB/main.

| Field | المعنى |
|---|---|
| `size` | حجم الـ partition (bytes) |
| `part_cfg` | نوع الـ partition (encoding من EXT_CSD) |
| `name[20]` | اسم وصفي (مثلاً "boot0") |
| `force_ro` | هل يُجبَر على Read-Only (للـ boot partitions) |
| `area_type` | نوع المنطقة (MAIN/BOOT/GP/RPMB) |

---

#### 12. `struct mmc_card` — الـ Struct الرئيسي

**الغرض:** يمثّل **كارد MMC/SD/SDIO واحد** بالكامل — مركز كل المعلومات الخاصة بالكارد.

| Field | النوع | المعنى |
|---|---|---|
| `*host` | `struct mmc_host *` | الـ host controller اللي الكارد متصل بيه |
| `dev` | `struct device` | Linux device model — الكارد كـ device |
| `ocr` | `u32` | OCR value الحالي — الـ voltage range المُختار |
| `rca` | `unsigned int` | Relative Card Address |
| `type` | `unsigned int` | نوع الكارد (MMC/SD/SDIO/COMBO) |
| `state` | `unsigned int` | حالة الكارد الحالية |
| `quirks` | `unsigned int` | workarounds المفعّلة للكارد ده |
| `quirk_max_rate` | `unsigned int` | حد سرعة مفروض بسبب quirk |
| `written_flag` | `bool` | هل الـ eMMC اتكتب منذ آخر power-on |
| `reenable_cmdq` | `bool` | يحتاج إعادة تفعيل CQE |
| `erase_size` | `unsigned int` | حجم وحدة المسح (sectors) |
| `erase_shift` | `unsigned int` | log2 لحجم المسح (لو power-of-2) |
| `pref_erase` | `unsigned int` | حجم المسح المفضّل (sectors) |
| `erase_arg` | `unsigned int` | الـ argument المستخدم (erase/trim/discard) |
| `erased_byte` | `u8` | القيمة اللي بتملأ الـ sectors بعد المسح |
| `raw_cid[4]` | `u32[]` | بيانات CID خام |
| `raw_csd[4]` | `u32[]` | بيانات CSD خام |
| `raw_scr[2]` | `u32[]` | بيانات SCR خام (SD) |
| `raw_ssr[16]` | `u32[]` | بيانات SSR خام (SD) |
| `cid` | `struct mmc_cid` | هوية الكارد (parsed) |
| `csd` | `struct mmc_csd` | بيانات الكارد (parsed) |
| `ext_csd` | `struct mmc_ext_csd` | Extended CSD (eMMC فقط) |
| `scr` | `struct sd_scr` | SCR (SD فقط) |
| `ssr` | `struct sd_ssr` | SSR (SD فقط) |
| `sw_caps` | `struct sd_switch_caps` | نتيجة CMD6 (SD فقط) |
| `ext_power` | `struct sd_ext_reg` | Extension Register للطاقة (SD 4.x) |
| `ext_perf` | `struct sd_ext_reg` | Extension Register للأداء (SD 4.x) |
| `uhs2_config` | `struct sd_uhs2_config` | إعدادات UHS-II |
| `sdio_funcs` | `unsigned int` | عدد SDIO functions |
| `sdio_funcs_probed` | `atomic_t` | عدد الـ functions اللي اتـ probe |
| `cccr` | `struct sdio_cccr` | Common Control Register (SDIO) |
| `cis` | `struct sdio_cis` | Common Info Structure (SDIO) |
| `sdio_func[7]` | `struct sdio_func *[]` | مصفوفة الـ SDIO functions (max 7) |
| `sdio_single_irq` | `struct sdio_func *` | الـ function الوحيدة لو IRQ واحد |
| `info` / `tuples` | متعدد | معلومات CIS إضافية |
| `sd_bus_speed` | `unsigned int` | Bus Speed Mode الحالي |
| `mmc_avail_type` | `unsigned int` | modes المدعومة من host وcard معاً |
| `drive_strength` | `unsigned int` | driver strength لـ UHS-I/HS200/HS400 |
| `debugfs_root` | `struct dentry *` | entry في debugfs |
| `part[7]` | `struct mmc_part[]` | الـ partitions الفيزيائية (eMMC) |
| `nr_parts` | `unsigned int` | عدد الـ partitions الفعلية |
| `complete_wq` | `struct workqueue_struct *` | workqueue خاص بإكمال الـ requests |

---

#### 13. `struct sdio_func` (من `sdio_func.h`)

**الغرض:** يمثّل **SDIO Function** واحدة — كل function جهاز مستقل (مثلاً WiFi chip فيها function للـ WLAN وأخرى للـ Bluetooth).

| Field | المعنى |
|---|---|
| `*card` | الكارد الأب |
| `dev` | Linux device |
| `irq_handler` | دالة معالجة IRQ |
| `num` | رقم الـ function (1–7) |
| `class/vendor/device` | تعريف الـ function |
| `max_blksize/cur_blksize` | حجم الـ block |
| `state` | `SDIO_STATE_PRESENT` |
| `*tmpbuf` | scratch buffer قابل لـ DMA |
| `*tuples` | قائمة linked list من CIS tuples |

---

### مخطط علاقات الـ Structs (ASCII)

```
                    ┌─────────────────────────────────────────┐
                    │            struct mmc_host               │
                    │  (linux/mmc/host.h)                     │
                    │  .ops → struct mmc_host_ops              │
                    │  .card ──────────────────────────────┐  │
                    │  .ios → struct mmc_ios                │  │
                    │  .cqe_ops → struct mmc_cqe_ops        │  │
                    │  .supply → struct mmc_supply          │  │
                    └─────────────────────────────────────────┘
                                                          │
                                                          ▼
                    ┌─────────────────────────────────────────────────────────┐
                    │                  struct mmc_card                        │
                    │                 (linux/mmc/card.h)                      │
                    │                                                         │
                    │  .host ──────────────────────────────── (back pointer) │
                    │  .dev  (struct device)                                  │
                    │                                                         │
                    │  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐ │
                    │  │ mmc_cid  │  │ mmc_csd  │  │    mmc_ext_csd       │ │
                    │  │  .cid    │  │  .csd    │  │    .ext_csd          │ │
                    │  └──────────┘  └──────────┘  │  (eMMC only, 512B)  │ │
                    │                               └──────────────────────┘ │
                    │  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐ │
                    │  │  sd_scr  │  │  sd_ssr  │  │   sd_switch_caps     │ │
                    │  │  .scr    │  │  .ssr    │  │   .sw_caps           │ │
                    │  └──────────┘  └──────────┘  └──────────────────────┘ │
                    │  (SD only)     (SD only)      (SD only)               │
                    │                                                         │
                    │  ┌────────────────┐  ┌────────────────┐               │
                    │  │  sd_ext_reg    │  │  sd_ext_reg    │               │
                    │  │  .ext_power    │  │  .ext_perf     │               │
                    │  └────────────────┘  └────────────────┘               │
                    │                                                         │
                    │  ┌──────────────────────┐                              │
                    │  │   sd_uhs2_config     │                              │
                    │  │   .uhs2_config       │                              │
                    │  └──────────────────────┘                              │
                    │                                                         │
                    │  ┌───────────┐  ┌──────────┐                          │
                    │  │sdio_cccr  │  │ sdio_cis │  (SDIO only)             │
                    │  │  .cccr    │  │  .cis    │                          │
                    │  └───────────┘  └──────────┘                          │
                    │                                                         │
                    │  .sdio_func[0..6] ──────┐                             │
                    │  .sdio_single_irq ───┐  │                             │
                    │                      │  │                             │
                    │  .part[0..6] ──┐     │  │                             │
                    │  (mmc_part)    │     │  │                             │
                    └────────────────┼─────┼──┼─────────────────────────────┘
                                     │     │  │
                                     ▼     ▼  ▼
                              ┌──────────┐  ┌──────────────────┐
                              │ mmc_part │  │   sdio_func      │
                              │  [×7]    │  │    [×7 max]      │
                              └──────────┘  │  .card ──────────┼──► mmc_card
                                            │  .tuples ────────┼──► sdio_func_tuple
                                            │                  │    (linked list)
                                            └──────────────────┘
```

---

### مخطط دورة حياة الـ Card (Lifecycle)

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                       CARD LIFECYCLE                             │
  └──────────────────────────────────────────────────────────────────┘

  [Power Up / Card Insert]
         │
         ▼
  mmc_rescan() ─── detect_change flag ─── delayed_work on host->detect
         │
         ▼
  mmc_alloc_card()
    └─► kzalloc(sizeof(struct mmc_card))
    └─► card->host = host
    └─► device_initialize(&card->dev)
         │
         ▼
  ┌──── Card Identification ────┐
  │  CMD0  → IDLE state         │
  │  CMD8  → SD v2 check        │
  │  ACMD41/CMD1 → OCR nego     │  card->ocr مُعيَّن
  │  CMD2  → CID read           │  card->raw_cid[] + card->cid
  │  CMD3  → RCA assign         │  card->rca
  └─────────────────────────────┘
         │
         ▼
  ┌──── Card Specific Data ─────┐
  │  CMD9  → CSD read           │  card->raw_csd[] + card->csd
  │  CMD7  → SELECT card        │
  │  ACMD51 → SCR (SD only)     │  card->scr
  │  CMD6  → switch caps (SD)   │  card->sw_caps
  │  CMD8  → EXT_CSD (eMMC)     │  card->ext_csd
  └─────────────────────────────┘
         │
         ▼
  ┌──── Bus Configuration ──────┐
  │  Set bus width (4/8 bit)    │
  │  Set timing (HS/HS200/etc)  │  host->ios تتغير
  │  Tuning if needed           │
  │  Partitions scan (eMMC)     │  card->part[] + card->nr_parts
  └─────────────────────────────┘
         │
         ▼
  mmc_add_card()
    └─► device_add(&card->dev)   ←── الكارد بقى visible في sysfs
    └─► blk layer registers
         │
         ▼
  ┌──── SDIO Probe (if SDIO) ───┐
  │  Read CCCR                  │  card->cccr
  │  Read common CIS            │  card->cis
  │  For each function:         │
  │    sdio_alloc_func()        │  card->sdio_func[n]
  │    device_add(func->dev)    │
  │    sdio_bus_probe()         │  matches sdio_driver
  └─────────────────────────────┘
         │
         ▼
  [Card in USE — I/O requests flowing]
         │
         ▼
  [Card Remove / Power Down]
         │
         ▼
  mmc_remove_card()
    └─► device_del(&card->dev)
    └─► (SDIO) sdio_remove_func() for each func
    └─► mmc_free_card()
          └─► kfree(card)
```

---

### مخطط تدفق الـ I/O Request (Call Flow)

```
  Block Layer (bio/request)
         │
         ▼
  mmc_blk_mq_queue_rq()           ← block driver
         │
         ▼
  mmc_blk_issue_rq()
    └─► mmc_start_request()
         │
         ▼
  host->ops->request(host, mrq)   ← يستدعي الـ host driver (مثلاً sdhci_request)
         │
         ├── [CQE enabled?]
         │     └─► cqe_ops->cqe_request(host, mrq)
         │           └─► hardware CQE registers
         │
         └── [normal path]
               └─► سيتم إرسال CMD + DATA على البص الفيزيائي
                     │
                     ▼
               hardware interrupt fires
                     │
                     ▼
               mmc_request_done(host, mrq)
                     │
                     ▼
               blk_mq_complete_request()
                     │
                     ▼
               bio_endio()   ← الـ user process يستيقظ
```

---

### مخطط تدفق SDIO IRQ

```
  SDIO Card Interrupt (hardware line)
         │
         ▼
  host->ops->enable_sdio_irq(host, 0)   ← يعطل الـ interrupt في الـ hardware
         │
         ▼
  mmc_signal_sdio_irq()
    └─► host->sdio_irq_pending = true
    └─► wake_up_process(host->sdio_irq_thread)
         │
         ▼
  sdio_irq_thread (kernel thread)
    └─► sdio_run_irqs(host)
          └─► for each func in sdio_func[]:
                if func->irq_handler:
                  func->irq_handler(func)   ← driver's handler (e.g., WiFi ISR)
         │
         ▼
  host->ops->enable_sdio_irq(host, 1)   ← يُعيد تفعيل الـ interrupt
```

---

### مخطط تدفق اكتشاف الكارد (Card Detection Flow)

```
  Card Insert (GPIO IRQ or polling)
         │
         ▼
  mmc_detect_change(host, delay)
    └─► host->detect_change = 1
    └─► schedule_delayed_work(&host->detect, delay)
         │
         ▼
  mmc_rescan() [work queue context]
    │
    ├─► [host already has card?]
    │     └─► mmc_bus_ops->detect()   ← check if still there
    │
    └─► [no card] try enumerate:
          ├─► mmc_attach_sdio()      ← CMD5 probe
          ├─► mmc_attach_sd()        ← ACMD41 probe
          └─► mmc_attach_mmc()       ← CMD1 probe
```

---

### استراتيجية الـ Locking

#### الـ Locks المستخدمة

| Lock | موقعه | يحمي ماذا |
|---|---|---|
| `host->lock` (spinlock) | `struct mmc_host` | `claimed`، bus ops، إرسال requests |
| `host->claimer` (ctx) | `struct mmc_host` | من "يملك" الـ bus الآن |
| `host->wq` (wait queue) | `struct mmc_host` | انتظار تحرير الـ bus |
| `sdio_claim_host()` | API function | exclusive access للـ SDIO bus |

#### ترتيب الـ Locking

```
  sdio_claim_host(func)         ← يمسك الـ bus عن طريق claim count
       │
       └─► mmc_claim_host(card->host)
             └─► spin_lock_irqsave(&host->lock)
             └─► host->claimed = 1
             └─► host->claimer = current_ctx
             └─► spin_unlock_irqrestore(&host->lock)

  [do I/O operations]

  sdio_release_host(func)
       └─► mmc_release_host(card->host)
             └─► spin_lock_irqsave(&host->lock)
             └─► host->claimed = 0
             └─► wake_up(&host->wq)
             └─► spin_unlock_irqrestore(&host->lock)
```

**القاعدة الأساسية:**
- الـ `host->lock` spinlock بس للـ claim state — مش للـ I/O نفسه.
- الـ I/O requests محمية بـ "bus claim" (mutex-like عبر wait queue).
- الـ `sdio_func` data (مثل `cur_blksize`) محمية ضمنياً بالـ bus claim.
- مفيش lock مباشر على `mmc_card` fields — الـ card مش تُعدَّل بعد الـ initialization إلا في سياق الـ bus claim.

#### تسلسل الـ Locking الآمن

```
  صح:   claim_host → access card fields → release_host
  غلط:  access card fields بدون claim (race condition!)
  صح:   spinlock فقط لتعديل claim state
  غلط:  spinlock مع I/O (ممنوع sleep تحت spinlock)
```
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### الـ Inline Functions والـ Macros

| الاسم | النوع | الغرض |
|---|---|---|
| `mmc_large_sector()` | `static inline bool` | بيتحقق لو الـ card بتشتغل بـ 4KB sector size |
| `mmc_card_enable_async_irq()` | `static inline int` | بيرجع قيمة `enable_async_irq` من الـ CCCR |
| `mmc_card_is_blockaddr()` | `bool` (extern) | بيتحقق لو الـ card بتستخدم block addressing |
| `mmc_card_mmc(c)` | macro | نوع الـ card = MMC |
| `mmc_card_sd(c)` | macro | نوع الـ card = SD |
| `mmc_card_sdio(c)` | macro | نوع الـ card = SDIO |
| `mmc_card_sd_combo(c)` | macro | نوع الـ card = SD combo (IO+mem) |

---

### التصنيف حسب الـ Category

```
┌─────────────────────────────────────────────┐
│  card.h Functions & Macros                  │
│                                             │
│  ┌──────────────────┐  ┌─────────────────┐  │
│  │  Type Query      │  │  Feature Query  │  │
│  │  Macros          │  │  Inline Funcs   │  │
│  │  mmc_card_mmc()  │  │  mmc_large_     │  │
│  │  mmc_card_sd()   │  │   sector()      │  │
│  │  mmc_card_sdio() │  │  mmc_card_      │  │
│  │  mmc_card_sd_    │  │   enable_       │  │
│  │   combo()        │  │   async_irq()   │  │
│  └──────────────────┘  └─────────────────┘  │
│                                             │
│  ┌──────────────────────────────────────┐   │
│  │  Address Mode Query (extern)         │   │
│  │  mmc_card_is_blockaddr()             │   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

---

### Group 1: Type Query Macros

الـ macros دي بتعمل type-check على `struct mmc_card` بسرعة. كلها بتقارن الـ `type` field بالثوابت المعرفة (`MMC_TYPE_*`). الـ kernel بيستخدمها في كل حتة عشان يتصرف حسب نوع الـ card من غير ما يعمل switch/case طويل.

---

#### `mmc_card_mmc(c)`

```c
#define mmc_card_mmc(c)     ((c)->type == MMC_TYPE_MMC)
```

**الـ card هي eMMC/MMC plain card.** بيرجع non-zero لو `type == 0`. الـ callers زي `mmc_blk_probe()` بيعملوا diverge في الـ initialization path بناءً على النتيجة.

| Parameter | الوصف |
|---|---|
| `c` | pointer لـ `struct mmc_card` |

**Return:** boolean expression — non-zero لو MMC، صفر غير كده.

**Key details:** macro مش function — مفيش function call overhead، بس كمان مفيش type safety تامة. اللي بيستعمله لازم يتأكد إن `c` مش NULL.

---

#### `mmc_card_sd(c)`

```c
#define mmc_card_sd(c)      ((c)->type == MMC_TYPE_SD)
```

**الـ card هي SD card standard (memory فقط، مش combo).** بيرجع non-zero لو `type == 1`.

| Parameter | الوصف |
|---|---|
| `c` | pointer لـ `struct mmc_card` |

**Return:** boolean — non-zero لو SD.

**Key details:** الـ SD combo card (IO+mem) مش بترجع true هنا — تحتاج `mmc_card_sd_combo()`. الـ callers زي `mmc_sd_init_card()` بيستخدموها للـ SD-specific initialization.

---

#### `mmc_card_sdio(c)`

```c
#define mmc_card_sdio(c)    ((c)->type == MMC_TYPE_SDIO)
```

**الـ card هي SDIO-only (IO فقط، مفيش memory function).** بيرجع non-zero لو `type == 2`. الـ WiFi/BT chips زي BCM4329 بتظهر كـ `MMC_TYPE_SDIO`.

| Parameter | الوصف |
|---|---|
| `c` | pointer لـ `struct mmc_card` |

**Return:** boolean — non-zero لو SDIO pure.

**Key details:** الـ card دي مفيهاش `mmc_blk` partition — الـ `sdio_func` array هو اللي بيتعمل probe عليه بس.

---

#### `mmc_card_sd_combo(c)`

```c
#define mmc_card_sd_combo(c)    ((c)->type == MMC_TYPE_SD_COMBO)
```

**الـ card بتجمع SD memory + SDIO IO functions في card واحدة.** بيرجع non-zero لو `type == 3`. نادر في الـ hardware الحديث.

| Parameter | الوصف |
|---|---|
| `c` | pointer لـ `struct mmc_card` |

**Return:** boolean — non-zero لو SD combo.

**Key details:** الـ core بيعمل probe على الـ memory part وعلى الـ SDIO functions في نفس الوقت. الـ `sdio_funcs` field بيكون non-zero حتى لو `mmc_card_sd_combo()` is true.

---

### Group 2: Feature Query Inline Functions

الـ functions دي بتعرض queries عن features محددة على مستوى الـ card. مكتوبة كـ `static inline` عشان الـ compiler يـ inline them ويزيل الـ overhead.

---

#### `mmc_large_sector()`

```c
static inline bool mmc_large_sector(struct mmc_card *card)
{
    return card->ext_csd.data_sector_size == 4096;
}
```

**بيتحقق لو الـ eMMC card بتستخدم 4KB physical sector size بدل الـ 512-byte الـ standard.** الـ eMMC v5.0+ ممكن يشتغل بـ 4KB native sectors عشان يحسن performance وlifetime على الـ NAND الكبيرة.

| Parameter | الوصف |
|---|---|
| `card` | pointer لـ `struct mmc_card` — الـ card اللي هيتعمل عليها الـ query |

**Return:** `true` لو `ext_csd.data_sector_size == 4096`، `false` لو 512.

**Key details:**
- الـ `ext_csd` بيتبقى valid بس بعد ما `mmc_read_ext_csd()` تتنفذ ناجحة أثناء الـ card initialization.
- الـ `mmc_blk` layer بيستخدم النتيجة لتحديد الـ `logical_block_size` للـ request queue عن طريق `blk_queue_logical_block_size()`.
- على SD cards الـ function دي مش relevant لأن `ext_csd` بتكون فاضية — بس مش بتعمل crash لأن الـ comparison بيكون false.

**Pseudocode:**
```
mmc_large_sector(card):
    return (card->ext_csd.data_sector_size == 4096)
    // no locking needed — read-only after init
```

**Caller context:** `mmc_blk_probe()`, `mmc_init_card()`, أي كود بيبني الـ block device parameters.

---

#### `mmc_card_enable_async_irq()`

```c
static inline int mmc_card_enable_async_irq(struct mmc_card *card)
{
    return card->cccr.enable_async_irq;
}
```

**بيرجع قيمة الـ `enable_async_irq` bit من الـ SDIO CCCR register.** لو enabled، الـ SDIO card بتقدر ترسل interrupts حتى لو الـ data lines في الـ active state — ده بيقلل الـ IRQ latency في الـ SDIO WiFi drivers.

| Parameter | الوصف |
|---|---|
| `card` | pointer لـ `struct mmc_card` — لازم تكون SDIO card عشان الـ `cccr` تكون populated |

**Return:** `int` — non-zero لو async IRQ مدعوم ومتفعل، صفر غير كده.

**Key details:**
- الـ `cccr.enable_async_irq` بيتحدد أثناء `sdio_read_cccr()` اللي بتتنفذ في `mmc_sdio_init_card()`.
- الـ callers زي `sdio_run_irqs()` بيستخدموا الـ flag ده عشان يحددوا إزاي يعملوا schedule للـ IRQ handling.
- مفيش locking — الـ field ده بيتكتب مرة واحدة وقت الـ init وبعدين read-only.

**Caller context:** SDIO IRQ path، `mmc_sdio_enable_async_irq()`.

---

### Group 3: Address Mode Query (Extern Function)

---

#### `mmc_card_is_blockaddr()`

```c
bool mmc_card_is_blockaddr(struct mmc_card *card);
```

**بيتحقق لو الـ card بتستخدم block addressing (SDHC/SDXC/high-density eMMC) بدل byte addressing.** الـ low-capacity cards (< 2GB) بتستخدم byte addressing — أي command بيبعت byte offset. الـ high-capacity cards بتستخدم block (512-byte) addressing — الـ offset بيكون block number.

| Parameter | الوصف |
|---|---|
| `card` | pointer لـ `struct mmc_card` — أي نوع card |

**Return:** `true` لو block-addressed، `false` لو byte-addressed.

**Key details:**
- الـ implementation مش في الـ header — موجودة في `drivers/mmc/core/card.c` أو ما شابه.
- الـ `mmc_card` بتحمل الـ state في `card->state` flags (مش ظاهر هنا مباشرة، بس الـ function بتقراه من هناك عبر `MMC_STATE_BLOCKADDR`).
- **critical للـ block layer**: الـ `mmc_blk_issue_rq()` لازم تعرف هتبعت address كـ bytes ولا كـ blocks في الـ CMD17/18/24/25.
- الـ SD cards بتبقى block-addressed لو `CSD.CSD_STRUCTURE == 1` (SD 2.0+) أو لو الـ card أكبر من 2GB.
- للـ eMMC، الـ `ext_csd.sectors > 0` بيعني block-addressed.

**Caller context:** `mmc_blk_issue_rq()`, `mmc_send_cmd()`, أي كود بيبني الـ read/write commands.

---

### ملاحظات على الـ Structs المهمة في الـ File

الـ header ده في الجوهر هو **data structure definitions** للـ `struct mmc_card` وكل الـ sub-structs اللي بتحتويها. الـ functions قليلة لأن الـ behavioral logic موزعة على ملفات الـ core (`core/`, `core/sd.c`, `core/sdio.c`). الـ header بيوفر:

| الـ Struct | الغرض |
|---|---|
| `struct mmc_cid` | Card Identification Data — manufacturer, serial, product name |
| `struct mmc_csd` | Card Specific Data — capacity, timing, block sizes |
| `struct mmc_ext_csd` | eMMC Extended CSD — partitions, HS200/400, RPMB, HPI, BKOPS, CmdQ |
| `struct sd_scr` | SD Configuration Register — bus widths, supported commands |
| `struct sd_ssr` | SD Status Register — AU size, erase timing |
| `struct sd_switch_caps` | CMD6 switch capabilities — UHS-I modes, driver types, current limits |
| `struct sd_ext_reg` | SD Extension Registers (SD 4.x) — PM, PERF functions |
| `struct sd_uhs2_config` | UHS-II physical layer config — lanes, speed ranges, FCU |
| `struct sdio_cccr` | SDIO Common Card Control Register — SDIO version, bus caps |
| `struct sdio_cis` | SDIO Common Information Structure — vendor/device IDs |
| `struct mmc_part` | Physical partition descriptor (eMMC) — boot/GP/RPMB |
| `struct mmc_card` | الـ top-level card object — يجمع كل حاجة فوق |

---

### العلاقة بين الـ Functions والـ Card State Machine

```
Power-on / Detection
        │
        ▼
mmc_sdio_init_card() / mmc_sd_init_card() / mmc_init_card()
        │
        ├── يقرأ CID → struct mmc_cid
        ├── يقرأ CSD → struct mmc_csd
        ├── يقرأ ext_csd (eMMC) → struct mmc_ext_csd
        │       └── mmc_large_sector() بيشتغل هنا صح
        ├── يقرأ CCCR (SDIO) → struct sdio_cccr
        │       └── mmc_card_enable_async_irq() بيشتغل هنا صح
        └── يحدد block addressing → mmc_card_is_blockaddr() بيشتغل هنا

Runtime (block I/O)
        │
        ├── mmc_card_mmc() / mmc_card_sd() → تحديد الـ command set
        ├── mmc_large_sector() → logical_block_size للـ queue
        └── mmc_card_is_blockaddr() → byte vs block address في كل CMD
```
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. مدخلات الـ debugfs المتعلقة بالـ MMC/SD

الـ `struct mmc_card` بيحتوي على `struct dentry *debugfs_root` — ده هو الـ mount point لكل entries الـ card في debugfs.

**المسار الافتراضي:**

```
/sys/kernel/debug/mmc<N>/
```

| Entry | الوصف | طريقة القراءة |
|---|---|---|
| `mmc<N>/ios` | الـ IOS الحالية (clock, bus width, timing, voltage) | `cat` |
| `mmc<N>/state` | حالة الـ host | `cat` |
| `mmc<N>/err_stats` | عداد الأخطاء per-error-type (`mmc_err_stat`) | `cat` |
| `mmc<N>/<card_id>/ext_csd` | الـ Extended CSD الـ raw bytes كاملة (eMMC فقط) | `xxd` |
| `mmc<N>/<card_id>/ssr` | الـ SD Status Register | `cat` |

```bash
# اقرأ الـ IOS الحالية للـ host الأول
cat /sys/kernel/debug/mmc0/ios

# اقرأ الـ Extended CSD كاملة (eMMC)
xxd /sys/kernel/debug/mmc0/mmc0:0001/ext_csd | head -32

# شوف إحصائيات الأخطاء
cat /sys/kernel/debug/mmc0/err_stats
```

**مثال على output الـ ios:**

```
clock:          200000000 Hz
actual clock:   200000000 Hz
vdd:            21 (3.3 ~ 3.4 V)
bus mode:       2 (push-pull)
chip select:    0 (don't care)
power mode:     2 (on)
bus width:      3 (8 bits)
timing spec:    9 (mmc HS200)
signal voltage: 1 (1.80 V)
driver type:    0 (driver type B)
```

---

#### 2. مدخلات الـ sysfs

الـ `struct mmc_card` بيـ embed `struct device dev` — ده اللي بيعمل الـ sysfs entries.

**المسار:** `/sys/bus/mmc/devices/mmc<N>:<CARD_ID>/`

| Entry | المحتوى | المصدر في الكود |
|---|---|---|
| `type` | MMC / SD / SDIO / SD_COMBO | `card->type` |
| `name` | اسم المنتج من الـ CID | `card->cid.prod_name` |
| `manfid` | Manufacturer ID | `card->cid.manfid` |
| `oemid` | OEM/Application ID | `card->cid.oemid` |
| `serial` | Serial number | `card->cid.serial` |
| `hwrev` | Hardware revision | `card->cid.hwrev` |
| `fwrev` | Firmware revision | `card->cid.fwrev` |
| `date` | تاريخ التصنيع (month/year) | `card->cid.month/year` |
| `erase_size` | حجم الـ erase unit بالـ sectors | `card->erase_size` |
| `preferred_erase_size` | الحجم المفضل للـ erase | `card->pref_erase` |
| `cid` | الـ CID الـ raw (4×u32) | `card->raw_cid` |
| `csd` | الـ CSD الـ raw | `card->raw_csd` |
| `scr` | الـ SCR الـ raw (SD فقط) | `card->raw_scr` |
| `ssr` | الـ SSR الـ raw (SD فقط) | `card->raw_ssr` |
| `ocr` | الـ OCR الحالية | `card->ocr` |
| `rca` | الـ Relative Card Address | `card->rca` |
| `quirks` | الـ quirks bitmap | `card->quirks` |
| `ffu_capable` | هل بيدعم Firmware Update؟ | `card->ext_csd.ffu_capable` |
| `cmdq_en` | هل الـ Command Queue شغال؟ | `card->ext_csd.cmdq_en` |
| `pre_eol_info` | حالة صحة الـ eMMC (EOL) | `card->ext_csd.pre_eol_info` |
| `life_time` | عمر الـ eMMC (Type A + B) | `device_life_time_est_typ_a/b` |

```bash
# اقرأ نوع الكارت
cat /sys/bus/mmc/devices/mmc0:0001/type

# اقرأ اسم المنتج وبيانات المصنع
cat /sys/bus/mmc/devices/mmc0:0001/name
cat /sys/bus/mmc/devices/mmc0:0001/manfid
cat /sys/bus/mmc/devices/mmc0:0001/date

# اقرأ حالة صحة الـ eMMC
cat /sys/bus/mmc/devices/mmc0:0001/pre_eol_info
# القيم: 0x00=undefined, 0x01=normal, 0x02=warning, 0x03=urgent

cat /sys/bus/mmc/devices/mmc0:0001/life_time
# القيم per nibble: 0=undefined, 1=0-10%, 2=10-20%, ..., B=100%+

# شوف كل الـ quirks بتاعة الكارت
cat /sys/bus/mmc/devices/mmc0:0001/quirks
```

---

#### 3. الـ ftrace — Tracepoints والـ Events

```bash
# شوف كل events المتاحة في subsystem الـ mmc
ls /sys/kernel/tracing/events/mmc/

# Events الأهم:
# mmc_request_start   — بداية كل request
# mmc_request_done    — نهاية كل request مع الـ error status
# mmc_cmd_rw_start    — إرسال command
# mmc_cmd_rw_end      — انتهاء command
# mmc_blk_rw_start    — بداية block I/O
# mmc_blk_rw_end      — نهاية block I/O
# mmc_blk_cmdq_req_start / mmc_blk_cmdq_req_done  — CmdQ requests

# فعّل كل events الـ mmc
echo 1 > /sys/kernel/tracing/events/mmc/enable

# أو event محدد
echo 1 > /sys/kernel/tracing/events/mmc/mmc_request_done/enable

# فعّل الـ tracing
echo 1 > /sys/kernel/tracing/tracing_on

# شغّل workload (مثلاً dd)
dd if=/dev/mmcblk0 of=/dev/null bs=1M count=100

# وقف وشوف النتيجة
echo 0 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace | head -100
```

**مثال على output:**

```
kworker/u4:2-85    [001] ....  1234.567890: mmc_request_start: mmc0: cmd=18, arg=0x00000800, flags=0x000000b5
kworker/u4:2-85    [001] ....  1234.568123: mmc_request_done:  mmc0: cmd=18, err=0
```

**الـ err=0 يعني نجاح — أي قيمة تانية هي error code.**

---

#### 4. الـ printk والـ Dynamic Debug

```bash
# فعّل dynamic debug لكل ملفات الـ mmc
echo 'module mmc_core +p' > /sys/kernel/debug/dynamic_debug/control
echo 'module mmc_block +p' > /sys/kernel/debug/dynamic_debug/control

# أو لـ file محدد (مثلاً core.c)
echo 'file drivers/mmc/core/core.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل الـ timing info مع الـ messages
echo 'module mmc_core +pt' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ messages في dmesg
dmesg -w | grep -i mmc

# ضبط loglevel لـ MMC (level 7 = debug)
echo 7 > /proc/sys/kernel/printk

# للـ SDIO بالذات
echo 'module cfg80211 +p' > /sys/kernel/debug/dynamic_debug/control
```

**الـ pr_debug() في الـ MMC subsystem محتاج `CONFIG_DYNAMIC_DEBUG=y` أو `-DDEBUG` في compile time.**

---

#### 5. خيارات الـ Kernel Config للـ Debugging

| Config | الوظيفة | متى تفعّله |
|---|---|---|
| `CONFIG_MMC_DEBUG` | يفعّل `pr_debug()` في كل MMC core | دايماً في dev environment |
| `CONFIG_DYNAMIC_DEBUG` | يسمح بتفعيل debug messages runtime | أساسي |
| `CONFIG_MMC_BLOCK_BOUNCE` | تفعيل bounce buffers — يسهل tracking الـ DMA | عند مشاكل DMA |
| `CONFIG_FAIL_MMC_REQUEST` | fault injection للـ MMC requests | اختبار error paths |
| `CONFIG_LOCK_STAT` | إحصائيات الـ locks (spinlocks في MMC core) | عند مشاكل deadlock |
| `CONFIG_PROVE_LOCKING` | كشف deadlocks بالـ lockdep | dev environment |
| `CONFIG_DEBUG_BLOCK_EXT_DEVT` | extended device numbers للـ debug | عند confusion في الـ devices |
| `CONFIG_CRYPTO_TEST` | اختبار الـ encryption (eMMC inline crypto) | مشاكل التشفير |
| `CONFIG_BLK_DEBUG_FS` | debugfs entries للـ block layer | I/O debugging |
| `CONFIG_SCHED_DEBUG` | debug الـ workqueue اللي بيستخدمه `complete_wq` | مشاكل الـ scheduling |

```bash
# تحقق من الـ config الحالية
zcat /proc/config.gz | grep -E 'MMC_DEBUG|FAIL_MMC|DYNAMIC_DEBUG'
```

---

#### 6. أدوات خاصة بالـ Subsystem

##### mmc-utils

```bash
# اقرأ الـ EXT_CSD كاملة وفسّرها
mmc extcsd read /dev/mmcblk0

# اقرأ معلومات الكارت
mmc info /dev/mmcblk0

# اختبر الـ RPMB (Replay Protected Memory Block)
mmc rpmb read-counter /dev/mmcblk0rpmb

# فعّل/عطّل الـ BKOPS
mmc bkops-enable /dev/mmcblk0

# اعمل Firmware Update (FFU) — لو ffu_capable=true
mmc ffu <firmware.img> /dev/mmcblk0
```

##### mmcli (لـ ModemManager في حالة SDIO modem)

```bash
# شوف حالة الـ SDIO functions المتصلة
cat /sys/bus/sdio/devices/mmc0:0001:1/vendor
cat /sys/bus/sdio/devices/mmc0:0001:1/device
```

##### مراقبة الـ CmdQ (Command Queue)

```bash
# تحقق من حالة الـ Command Queue
cat /sys/bus/mmc/devices/mmc0:0001/cmdq_en
# 1 = enabled, 0 = disabled

# مراقبة queue depth
cat /sys/kernel/debug/mmc0/err_stats | grep -i cmdq
```

---

#### 7. جدول رسائل الأخطاء الشائعة

| رسالة الـ kernel log | المعنى | الحل |
|---|---|---|
| `mmc0: error -110 whilst initialising MMC card` | timeout أثناء الـ init — الكارت مش بيرد | افحص الـ power supply والـ voltage، جرب `max_hz` أقل |
| `mmc0: Controller never released inhibit bit(s).` | الـ host controller مش بيكمل الـ command | reset الـ controller، افحص الـ clock gating |
| `mmc0: error -84 transferring data, sector ...` | CRC error في الـ data | مشكلة hardware (نويزات على الـ bus)، جرب تقليل السرعة |
| `mmc0: Too many retries` | فشل بعد كل المحاولات | مشكلة في الكارت نفسه أو الـ bus integrity |
| `mmc0: unrecognised CSD structure version ...` | الـ CSD format غريب | الكارت مش standard، هيحتاج quirk |
| `mmc0: Tuning failed` | فشل tuning في HS200/HS400 | قلّل السرعة، افحص الـ PCB layout والـ impedance |
| `mmc0: ADMA error` | خطأ في الـ ADMA descriptor table | مشكلة DMA mapping، افحص الـ IOMMU settings |
| `mmc0: cache flush error -110` | timeout في flush الـ cache | الكارت مش بيستجيب — فعّل `MMC_QUIRK_BROKEN_CACHE_FLUSH` |
| `mmc0: HPI transfer failed` | فشل الـ High Priority Interrupt | فعّل `MMC_QUIRK_BROKEN_HPI` |
| `mmc0: card is slow to respond to CMD0` | الكارت بطيء في الـ reset | زوّد timeout أو أضف delay بعد power-on |
| `mmcblk0: error -84 sending status command` | CRC error في status | مشكلة mechanical في الـ card slot أو oxidation |
| `mmc0: Unknown chip select interrupt` | unexpected IRQ في SDIO | فعّل `MMC_QUIRK_BROKEN_IRQ_POLLING` |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في core.c — عند timeout في أي request */
static int mmc_wait_for_req_done(struct mmc_host *host,
                                  struct mmc_request *mrq)
{
    /* Strategic point: إذا الـ completion ما جتش في الوقت المحدد */
    if (!wait_for_completion_timeout(&mrq->completion,
                                      msecs_to_jiffies(MMC_CMD_TIMEOUT))) {
        WARN_ON(1); /* print stack trace */
        dump_stack();
        return -ETIMEDOUT;
    }
    return 0;
}

/* في card init — عند قراءة EXT_CSD */
static int mmc_read_ext_csd(struct mmc_card *card)
{
    /* WARN لو rev غير متوقع */
    WARN_ON(card->ext_csd.rev > 8);
    ...
}

/* في mmc_blk.c — عند اكتشاف حالة غير consistent */
if (WARN_ON(!card->ext_csd.cmdq_support && card->ext_csd.cmdq_en)) {
    /* CmdQ enabled but not supported? impossible */
    card->ext_csd.cmdq_en = false;
}
```

**نقاط مقترحة للـ WARN_ON:**

| الموقع | الشرط |
|---|---|
| بعد قراءة الـ CSD | `card->csd.capacity == 0` |
| في `mmc_select_timing()` | timing مش في المدى المعروف |
| في الـ RPMB operations | `raw_rpmb_size_mult == 0` |
| في الـ partition init | `nr_parts > MMC_NUM_PHY_PARTITION` |
| عند تفعيل الـ CmdQ | `cmdq_depth == 0 && cmdq_en` |

---

### Hardware Level

#### 1. التحقق من تطابق الـ hardware state مع الـ kernel state

```bash
# تحقق أن الـ voltage اللي الـ kernel شايفه هو نفسه على الـ hardware
# kernel بيقول:
cat /sys/kernel/debug/mmc0/ios | grep voltage
# output: signal voltage: 1 (1.80 V)

# قارنه بقياس فعلي على pin VQMMC بالـ multimeter أو scope
# لو مش متطابق — مشكلة في الـ voltage regulator أو الـ DT

# تحقق من الـ clock frequency
cat /sys/kernel/debug/mmc0/ios | grep clock
# actual clock يجب يتطابق مع اللي بتشوفه على الـ oscilloscope على CLK pin

# تحقق من عدد الـ SDIO functions
cat /sys/bus/mmc/devices/mmc0:0001/sdio_funcs
# قارنه بالـ datasheet للـ SDIO device
```

---

#### 2. Register Dump Techniques

```bash
# لو عارف base address الـ MMC controller (من الـ DT)
# مثلاً على Raspberry Pi: EMMC2 base = 0xfe340000

# باستخدام devmem2
devmem2 0xfe340000 w   # اقرأ الـ register الأول (SDHCI_DMA_ADDRESS)
devmem2 0xfe340004 w   # SDHCI_BLOCK_SIZE
devmem2 0xfe34002C w   # SDHCI_PRESENT_STATE

# SDHCI_PRESENT_STATE bits المهمة:
# bit 0: CMD inhibit
# bit 1: DAT inhibit
# bit 2: DAT line active
# bit 3: RE-TUNING request
# bit 16-19: DAT[3:0] line signal level
# bit 20: CMD line signal level

# باستخدام /dev/mem (محتاج CONFIG_DEVMEM=y)
python3 -c "
import mmap, os, struct
fd = os.open('/dev/mem', os.O_RDONLY | os.O_SYNC)
mem = mmap.mmap(fd, 0x100, offset=0xfe340000)
val = struct.unpack('<I', mem[0x24:0x28])[0]  # SDHCI_PRESENT_STATE
print(f'PRESENT_STATE: 0x{val:08x}')
print(f'CMD inhibit: {val & 1}')
print(f'DAT inhibit: {(val>>1) & 1}')
print(f'Card inserted: {(val>>16) & 1}')
mem.close()
os.close(fd)
"
```

**SDHCI Standard Registers الأهم للـ Debug:**

| Offset | الاسم | البيتات المهمة |
|---|---|---|
| `0x24` | `PRESENT_STATE` | CMD/DAT inhibit، card detect، write protect |
| `0x2C` | `HOST_CONTROL1` | DMA select، bus width، high speed |
| `0x30` | `POWER_CONTROL` | bus power، voltage select |
| `0x34` | `BLOCK_GAP` | stop at block gap |
| `0x3C` | `INT_STATUS` | آخر interrupt status |
| `0x40` | `INT_ENABLE` | الـ interrupts المفعّلة |
| `0x4E` | `HOST_CONTROL2` | UHS mode، driver strength، tuning |

---

#### 3. Logic Analyzer و Oscilloscope Tips

**الـ Pins المهمة للـ Probing:**

```
eMMC / SD Pins:
┌─────────────────────────────────────────┐
│  CLK  ──── probe here for clock quality │
│  CMD  ──── bidirectional command line   │
│  DAT0 ──── data (1-bit mode)            │
│  DAT1 ──── data + SDIO IRQ              │
│  DAT2 ──── data                         │
│  DAT3 ──── data + card detect in SD     │
│  DAT4-7 ── data (8-bit eMMC only)       │
│  VCC  ──── 3.3V supply                  │
│  VCCQ ──── I/O supply (1.8V or 3.3V)   │
└─────────────────────────────────────────┘
```

**إعدادات الـ Logic Analyzer:**

```
- Sample rate: مينيمم 4× أعلى clock frequency
  - HS200 (200MHz) → محتاج 800MHz sample rate
  - HS52 (52MHz) → محتاج 208MHz sample rate
  - للـ UHS-I (104MHz) → محتاج 416MHz

- Trigger: rising edge على CLK
- Protocol decode: MMC/SD في أي LA بيدعم JESD84
- انتبه لـ:
  - Glitches على CLK (بيسبب CRC errors)
  - Slow rise/fall time > 1ns في high-speed modes
  - Ringing بعد transitions
  - CMD line stuck low (card busy)
```

**Oscilloscope — Eye Diagram:**

```bash
# لازم تشوف eye diagram على DAT lines في HS200/HS400
# - Eye opening > 70% يعتبر ok
# - أقل من 50% → مشكلة في PCB impedance أو termination
# - الـ impedance المثالي: 50Ω differential
```

---

#### 4. Hardware Issues الشائعة وأنماطها في الـ kernel log

| المشكلة الـ Hardware | Pattern في الـ dmesg |
|---|---|
| **الكارت مش متصل صح** (loose contact) | `mmc0: error -110` بيجي وبيروح intermittently |
| **الـ VCC مش مستقر** (نويز في الـ power) | `CRC error` في أوقات الـ high load |
| **PCB trace طويلة** (stub reflections) | Tuning failures في HS200، بيتعمل retune كتير |
| **الـ VCCQ مش بيتحول لـ 1.8V** | `mmc0: failed to switch to 1.8V signalling` |
| **Clock jitter** | `mmc0: Tuning failed after N attempts` |
| **Oxidized contacts** | `mmcblk0: error -84` بشكل متكرر |
| **Wrong pull-up resistors** | `mmc0: card init failed` في أول boot |
| **Missing RST_N pull-up** | eMMC بيتعمله hard reset بدون سبب |
| **DAT line stuck low** | `mmc0: Controller never released inhibit bit` |
| **SDIO interrupt stuck** | `unexpected interrupt` مع SDIO cards |

---

#### 5. Device Tree Debugging

**تحقق أن الـ DT بيتطابق مع الـ hardware الفعلي:**

```bash
# اقرأ الـ DT المستخدم فعلاً في الـ runtime
ls /sys/firmware/devicetree/base/mmc*/

# شوف الـ properties
cat /sys/firmware/devicetree/base/mmc@fe340000/clock-frequency | xxd
# يجب تطابق مع الـ max-frequency في الـ DT

# قارن الـ compatible string
cat /sys/firmware/devicetree/base/mmc@fe340000/compatible

# شوف الـ bus-width المضبوطة في الـ DT
cat /sys/firmware/devicetree/base/mmc@fe340000/bus-width | xxd
# 01 00 00 04 = 4-bit, 01 00 00 08 = 8-bit

# شوف الـ voltages المدعومة
cat /sys/firmware/devicetree/base/mmc@fe340000/mmc-ddr-3_3v
# لو الـ file موجود → DDR 3.3V مفعّل

# تحقق من pinctrl
cat /sys/firmware/devicetree/base/mmc@fe340000/pinctrl-names
```

**النقاط الحرجة في الـ DT للـ MMC:**

```dts
/* مثال DT node لـ eMMC */
&mmc0 {
    status = "okay";
    bus-width = <8>;              /* يجب يتطابق مع عدد الـ pins الفعلي */
    max-frequency = <200000000>;  /* يجب ما يتعدى حد الـ PCB */
    cap-mmc-highspeed;
    mmc-hs200-1_8v;
    mmc-hs400-1_8v;
    non-removable;                /* لو eMMC مسودّر */
    vmmc-supply = <&vcc_3v3>;
    vqmmc-supply = <&vcc_1v8>;   /* مهم جداً للـ 1.8V signalling */
};
```

```bash
# لو الـ DT مش صح، بتشوف في dmesg:
# "mmc0: no vmmc regulator found" → vqmmc-supply ناقصة
# "mmc0: failed to get regulator" → الـ regulator node غلط
# "mmc0: tried to set unsupported voltage" → mismatch بين DT والـ hardware

# overlay debugging
dtc -I fs /sys/firmware/devicetree/base > /tmp/current_dt.dts
grep -A 20 'mmc@' /tmp/current_dt.dts
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للـ Copy-Paste

##### شخّص مشكلة eMMC من الصفر

```bash
#!/bin/bash
# eMMC Full Diagnostic Script

DEVICE="mmc0"
CARD="mmc0:0001"
BLK_DEV="/dev/mmcblk0"

echo "=== Card Info ==="
cat /sys/bus/mmc/devices/${CARD}/type
cat /sys/bus/mmc/devices/${CARD}/name
cat /sys/bus/mmc/devices/${CARD}/manfid
cat /sys/bus/mmc/devices/${CARD}/date

echo ""
echo "=== Current IOS ==="
cat /sys/kernel/debug/${DEVICE}/ios

echo ""
echo "=== Health Status ==="
PRE_EOL=$(cat /sys/bus/mmc/devices/${CARD}/pre_eol_info 2>/dev/null)
echo "Pre-EOL info: $PRE_EOL"
case $PRE_EOL in
    0x01) echo "  => Normal" ;;
    0x02) echo "  => WARNING: Approaching EOL" ;;
    0x03) echo "  => CRITICAL: EOL exceeded!" ;;
    *)    echo "  => Unknown/Not available" ;;
esac

LIFE_A=$(cat /sys/bus/mmc/devices/${CARD}/life_time 2>/dev/null | cut -d' ' -f1)
LIFE_B=$(cat /sys/bus/mmc/devices/${CARD}/life_time 2>/dev/null | cut -d' ' -f2)
echo "Life time Type A: $LIFE_A"
echo "Life time Type B: $LIFE_B"

echo ""
echo "=== Error Statistics ==="
cat /sys/kernel/debug/${DEVICE}/err_stats

echo ""
echo "=== EXT_CSD Critical Bytes ==="
mmc extcsd read ${BLK_DEV} 2>/dev/null | grep -E 'rev|Cache|BKOPS|CmdQueue|Firmware'
```

##### فعّل full tracing وشوف الـ performance

```bash
#!/bin/bash
# MMC Performance Trace

# نظّف
echo 0 > /sys/kernel/tracing/tracing_on
echo > /sys/kernel/tracing/trace

# فعّل events
for event in mmc_request_start mmc_request_done mmc_blk_rw_start mmc_blk_rw_end; do
    echo 1 > /sys/kernel/tracing/events/mmc/${event}/enable 2>/dev/null
done

# فعّل الـ tracing
echo 1 > /sys/kernel/tracing/tracing_on

# شغّل workload
dd if=/dev/mmcblk0 of=/dev/null bs=4K count=1000 2>/dev/null
sync

# وقف
echo 0 > /sys/kernel/tracing/tracing_on

# حلّل النتائج
echo "=== Requests with errors ==="
grep 'mmc_request_done' /sys/kernel/tracing/trace | grep -v 'err=0'

echo ""
echo "=== Request timing summary ==="
# احسب الـ average latency
python3 -c "
import re
starts = {}
latencies = []
with open('/sys/kernel/tracing/trace') as f:
    for line in f:
        m_start = re.search(r'(\d+\.\d+): mmc_request_start.*cmd=(\d+)', line)
        m_done  = re.search(r'(\d+\.\d+): mmc_request_done.*cmd=(\d+)', line)
        if m_start:
            starts[m_start.group(2)] = float(m_start.group(1))
        if m_done and m_done.group(2) in starts:
            lat = float(m_done.group(1)) - starts[m_done.group(2)]
            latencies.append(lat * 1000)  # to ms

if latencies:
    print(f'Count: {len(latencies)}')
    print(f'Avg latency: {sum(latencies)/len(latencies):.3f} ms')
    print(f'Max latency: {max(latencies):.3f} ms')
"
```

##### شخّص مشكلة SDIO

```bash
# شوف كل الـ SDIO devices المتصلة
ls /sys/bus/sdio/devices/

# لكل function
for f in /sys/bus/sdio/devices/mmc*; do
    echo "=== $f ==="
    echo -n "vendor: "; cat $f/vendor 2>/dev/null
    echo -n "device: "; cat $f/device 2>/dev/null
    echo -n "class:  "; cat $f/class  2>/dev/null
done

# شوف interrupt stats للـ SDIO
cat /proc/interrupts | grep -i mmc

# فعّل debugging للـ SDIO IRQ
echo 'file drivers/mmc/core/sdio_irq.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/mmc/core/sdio.c +p'     > /sys/kernel/debug/dynamic_debug/control
```

##### تحقق من صحة الـ eMMC بـ mmc-utils

```bash
# قرأ وفسّر EXT_CSD كامل
mmc extcsd read /dev/mmcblk0 2>&1 | tee /tmp/extcsd_dump.txt

# نقاط مهمة في الـ output:
grep -E 'BKOPS|CMDQ|CACHE|RPMB|PARTITION|FFU|EOL|Life' /tmp/extcsd_dump.txt

# اعمل discard test
blkdiscard /dev/mmcblk0p1

# قيس sequential read speed
dd if=/dev/mmcblk0 of=/dev/null bs=1M count=512 status=progress

# قيس random read (محتاج fio)
fio --name=rand-read --filename=/dev/mmcblk0 \
    --rw=randread --bs=4k --numjobs=1 \
    --iodepth=32 --runtime=30 --time_based \
    --ioengine=libaio --output-format=normal \
    --section=rand-read
```

##### مراقبة الأخطاء real-time

```bash
# شاشة مراقبة MMC errors
watch -n 1 'echo "=== MMC Error Stats ===" && cat /sys/kernel/debug/mmc0/err_stats && echo "" && echo "=== dmesg tail ===" && dmesg | grep -i mmc | tail -10'

# أو باستخدام udevadm لمراقبة card events
udevadm monitor --subsystem-match=mmc

# راقب الـ kernel messages المتعلقة بالـ MMC بشكل مباشر
dmesg -w | grep -E 'mmc|mmcblk|sdhci'
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: eMMC مش بيشتغل بـ HS200 على RK3562 — Industrial Gateway

#### العنوان
eMMC يرجع لـ Default Speed بعد boot على **RK3562**-based industrial IoT gateway

#### السياق
شركة بتصنع industrial gateway بتستخدم RK3562 مع eMMC 5.1 chip (Kingston EMMC04G-M627). الـ product بيتشغل نظام Ubuntu على eMMC. أثناء stress testing لاحظوا إن الـ I/O throughput بيوصل بس 45 MB/s بدل 250+ MB/s المتوقعة من HS200.

#### المشكلة
الـ eMMC بيتنيغوشييت على **Default Speed (26 MHz)** بدل HS200 (200 MHz). الـ dmesg بيظهر:

```bash
mmc0: new MMC card at address 0001
mmc0: mmc_avail_type = 0x2   # only HS supported, HS200 missing
```

#### التحليل
في `struct mmc_card` في `card.h`:

```c
unsigned int    mmc_avail_type; /* supported device type by both host and card */
```

الـ field ده بيتحسب من intersection بين capabilities الـ host و capabilities الـ card اللي بتيجي من `ext_csd.raw_card_type` (byte 196 في EXT_CSD). لما بنعمل debug:

```bash
# قراءة raw EXT_CSD byte 196
cat /sys/kernel/debug/mmc0/mmc0:0001/ext_csd | cut -c 393-394
# output: 06  → HS + HS200 supported by card
```

الـ card بتدعم HS200 — المشكلة في الـ host. بنروح لـ `mmc_avail_type` ونلاقي إن الـ host flag `MMC_CAP2_HS200_1_8V_SDR` مش متحطة في DT.

```c
/* mmc_avail_type is AND of host caps and card caps */
/* if host doesn't advertise HS200, ext_csd.hs200_max_dtr stays unused */
unsigned int    hs200_max_dtr;   /* from struct mmc_ext_csd */
#define MMC_HS200_MAX_DTR   200000000
```

#### الحل
في device tree الخاص بـ RK3562:

```dts
/* قبل */
&sdhci {
    bus-width = <8>;
    non-removable;
    status = "okay";
};

/* بعد */
&sdhci {
    bus-width = <8>;
    non-removable;
    mmc-hs200-1_8v;       /* تفعيل HS200 على 1.8V */
    vmmc-supply = <&vcc_3v3>;
    vqmmc-supply = <&vcc_1v8>;
    status = "okay";
};
```

بعد kإعادة compile وflash:

```bash
dmesg | grep mmc0
# mmc0: new ultra high speed MMC card at address 0001
# mmc0: mmc_avail_type = 0x6   # HS + HS200 now set

dd if=/dev/mmcblk0 of=/dev/null bs=1M count=100
# 100 MB copied in 0.38s = 263 MB/s
```

#### الدرس المستفاد
الـ `mmc_avail_type` في `struct mmc_card` هو الـ **intersection** — حتى لو الـ card بتدعم HS200، لازم الـ host driver يعلن عن capability دي عبر DT. دايمًا check الـ field ده في debugfs قبل ما تحكم إن الـ card نفسها المشكلة.

---

### السيناريو الثاني: Android TV Box على Allwinner H616 — SD Card يعمل corruption

#### العنوان
بيانات SD card بتتخرب على **Allwinner H616** Android TV box عند UHS-I DDR50

#### السياق
TV box بيستخدم Allwinner H616 مع slot SD خارجي للمستخدم. العملاء بيشكوا من filesystem corruption على SD cards بعد ساعات من الاستخدام. المشكلة بتظهر بس مع بعض cards (SanDisk Ultra Class 10 A1).

#### المشكلة
الـ corruption بيحصل أثناء large sequential writes. الـ kernel log بيظهر:

```
mmc1: tuning execution failed
mmc1: falling back to lower transfer mode
mmcblk1: error -84 transferring data
```

#### التحليل
بنفحص `struct sd_switch_caps` في `card.h`:

```c
struct sd_switch_caps {
    unsigned int    hs_max_dtr;
    unsigned int    uhs_max_dtr;
    unsigned int    sd3_bus_mode;
#define UHS_DDR50_BUS_SPEED     4
#define SD_MODE_UHS_DDR50       (1 << UHS_DDR50_BUS_SPEED)
    unsigned int    sd3_drv_type;
    unsigned int    sd3_curr_limit;
};
```

الـ SanDisk card دي بتدعم DDR50 (`sd3_bus_mode` فيها `SD_MODE_UHS_DDR50` set). الـ H616 host بيحاول يشتغل على DDR50 لكن الـ tuning بيفشل. بنشوف أيضًا:

```c
#define MMC_QUIRK_NO_UHS_DDR50_TUNING   (1<<18)
```

الـ quirk ده موجود في `mmc_card.quirks` field بالظبط لحالات زي دي. المشكلة إن الـ card مش معندهاش quirk entry في kernel quirks table.

```bash
# نشوف الـ card CID
cat /sys/class/mmc_host/mmc1/mmc1:aaaa/cid
# manfid=0x000003 (SanDisk), prod_name=SA64G
# يطابق الـ struct mmc_cid:
#   unsigned int  manfid;
#   char          prod_name[8];
```

#### الحل
إضافة quirk لهذه الـ card في `drivers/mmc/core/quirks.h` أو باستخدام sysfs مؤقتًا:

```c
/* في quirks table */
{
    .name = "SA64G",
    .manfid = 0x000003,
    .oemid = 0x5344,
    .quirk = MMC_QUIRK_NO_UHS_DDR50_TUNING,
},
```

أو كـ workaround في DT للـ host:

```dts
&mmc1 {
    no-mmc-hs400;
    sd-uhs-ddr50;       /* السماح بـ DDR50 */
    max-frequency = <50000000>;  /* تقييد الـ frequency */
};
```

الأسلم: تعطيل DDR50 بالكامل للـ host:

```dts
&mmc1 {
    /* حذف sd-uhs-ddr50 من capabilities */
    sd-uhs-sdr50;
    sd-uhs-sdr25;
    sd-uhs-sdr12;
    /* DDR50 محذوف — سيتفاوض على SDR50 بدلاً منها */
};
```

#### الدرس المستفاد
الـ `sd_switch_caps.sd3_bus_mode` بيقول إيه الـ modes الـ card بتدعمها، لكن مش كل card بتنفذهم صح. الـ `quirks` field في `struct mmc_card` موجود لهذا السبب بالظبط — دايمًا فحص quirks table لكل card بيعمل مشكلة.

---

### السيناريو الثالث: IoT Sensor على STM32MP1 — SDIO WiFi مش بيتعرف

#### العنوان
**RTL8723DS** SDIO WiFi chip مش بيظهر على STM32MP157-based IoT gateway

#### السياق
IoT sensor hub بيستخدم STM32MP157 مع RTL8723DS WiFi/BT combo chip موصل على SDIO. أثناء bring-up على board جديدة، الـ chip مش بيتعرف خالص. الـ dmesg بيوقف عند:

```
mmc2: new SDIO card at address 0001
```

بس مفيش driver يـ probe.

#### المشكلة
الـ `sdio_funcs` في `struct mmc_card` بيتحسب غلط — الـ chip بيتعرف كـ 1 function بس بدل 2.

#### التحليل
بنشوف `struct mmc_card`:

```c
unsigned int    sdio_funcs;          /* number of SDIO functions */
atomic_t        sdio_funcs_probed;   /* number of probed SDIO funcs */
struct sdio_cccr    cccr;            /* common card info */
struct sdio_cis     cis;             /* common tuple info */
struct sdio_func    *sdio_func[SDIO_MAX_FUNCS]; /* up to 7 functions */
```

الـ `sdio_cccr` بيتقرأ من الـ CCCR register على الـ card. بنستخدم debugfs:

```bash
cat /sys/kernel/debug/mmc2/mmc2:0001/sdio_cccr
# IOE: 0x02   # function 1 enabled only
# IOR: 0x00
```

المشكلة في الـ power-up sequence — الـ RTL8723DS محتاج WIFI_REG_ON pin يتـ assert قبل الـ SDIO initialization. الـ pin مش متحط صح في DT.

```c
/* الـ cccr.multi_block يتأثر بالـ power state */
struct sdio_cccr {
    unsigned int    sdio_vsn;
    unsigned int    sd_vsn;
    unsigned int    multi_block:1,
                    low_speed:1,
                    wide_bus:1,
                    high_power:1,
                    high_speed:1,
                    disable_cd:1,
                    enable_async_irq:1;
};
```

#### الحل
في DT:

```dts
/* قبل */
&sdmmc2 {
    bus-width = <4>;
    non-removable;
    status = "okay";
};

/* بعد */
wifi_pwrseq: wifi-pwrseq {
    compatible = "mmc-pwrseq-simple";
    reset-gpios = <&gpioh 4 GPIO_ACTIVE_LOW>;  /* WIFI_REG_ON */
    post-power-on-delay-ms = <200>;
};

&sdmmc2 {
    bus-width = <4>;
    non-removable;
    mmc-pwrseq = <&wifi_pwrseq>;
    #address-cells = <1>;
    #size-cells = <0>;
    status = "okay";

    rtl8723ds: wifi@1 {
        reg = <1>;
        interrupt-parent = <&gpiod>;
        interrupts = <3 IRQ_TYPE_LEVEL_LOW>;
    };
};
```

بعد التعديل:

```bash
dmesg | grep mmc2
# mmc2: new high speed SDIO card at address 0001
# mmc2: sdio_funcs = 2  # WiFi + BT

cat /sys/bus/sdio/devices/mmc2:0001:1/vendor
# 0x024c  (Realtek)
```

#### الدرس المستفاد
الـ `sdio_funcs` و `sdio_func[]` array في `struct mmc_card` بيتملوا بس لو الـ card اتـ powered بشكل صح. الـ SDIO chip اللي محتاج external power-enable pin لازم يتوصف في DT كـ `mmc-pwrseq` — الـ kernel مش بيعرف يعمله automatically.

---

### السيناريو الرابع: Automotive ECU على i.MX8M — eMMC RPMB تشفير بيفشل

#### العنوان
كتابة Secure Boot keys في **RPMB** partition بتفشل على i.MX8M automotive ECU

#### السياق
Automotive ECU بيستخدم i.MX8M Plus مع eMMC 5.1 لتخزين secure keys في الـ RPMB partition. أثناء production provisioning، الـ `rpmb_write` operation بتفشل على بعض الـ units.

#### المشكلة
الـ RPMB writes بيرجعوا `-EACCES`. البعض من الـ boards بتشتغل والبعض لأ.

#### التحليل
الـ RPMB partition بيتوصف في `struct mmc_part`:

```c
struct mmc_part {
    u64          size;       /* partition size (in bytes) */
    unsigned int part_cfg;   /* partition type */
    char         name[MAX_MMC_PART_NAME_LEN];
    bool         force_ro;   /* to make boot parts RO by default */
    unsigned int area_type;
#define MMC_BLK_DATA_AREA_RPMB  (1<<3)   /* RPMB area type */
};
```

وفي `struct mmc_ext_csd`:

```c
u8    raw_rpmb_size_mult;   /* byte 168 — RPMB partition size multiplier */
u8    sec_feature_support;  /* security features */
u8    rst_n_function;       /* RST_n signal behavior */
```

بنفحص:

```bash
# على board فاشلة
mmc extcsd read /dev/mmcblk0 | grep -A2 "RPMB"
# RPMB Size: 0x00  ← مشكلة! الـ RPMB مش متهيئ

# على board شغالة
# RPMB Size: 0x10  ← 512KB — صح
```

الـ `raw_rpmb_size_mult = 0` معناه إن الـ RPMB partition مش اتـ configure. الـ provisioning script محتاج يعمل `mmc rpmb` setup قبل ما يكتب.

بنفحص أيضًا:

```c
bool    enhanced_rpmb_supported; /* from mmc_ext_csd */
```

الـ boards الفاشلة عندها eMMC من vendor مختلف — بيدعم Enhanced RPMB بس الـ provisioning script مش بيتعامل معاه.

#### الحل
```bash
# خطوة 1: تأكد إن الـ RPMB partition موجود
mmc extcsd read /dev/mmcblk0 | grep "RPMB SIZE"

# خطوة 2: لو zero، set الـ partition config
# الـ raw_rpmb_size_mult byte 168 لازم يتكتب
mmc rpmb write-key /dev/mmcblk0rpmb /dev/mmcblk0 < key.bin

# خطوة 3: verify إن الـ part_cfg اتعمل صح
# struct mmc_part.part_cfg بيعكس EXT_CSD[179]
mmc extcsd read /dev/mmcblk0 | grep "PARTITION_CONFIG"
```

في الـ provisioning code:

```c
/* قبل RPMB write، check struct mmc_part */
struct mmc_card *card = ...;
int i;

for (i = 0; i < card->nr_parts; i++) {
    if (card->part[i].area_type & MMC_BLK_DATA_AREA_RPMB) {
        /* تحقق إن الـ size مش zero */
        if (card->part[i].size == 0) {
            /* RPMB not provisioned */
            return -ENODEV;
        }
    }
}
```

#### الدرس المستفاد
الـ `mmc_part[]` array في `struct mmc_card` بيحتوي على الـ physical partitions. الـ `MMC_BLK_DATA_AREA_RPMB` flag لوحده مش كافي — لازم تـ check الـ `size` field كمان. الـ eMMC من الـ factory ممكن يجي بـ RPMB غير configured.

---

### السيناريو الخامس: Custom Board Bring-up على AM62x — eMMC بطيء جداً في الكتابة

#### العنوان
eMMC write performance سيئة جداً على **TI AM62x** custom board — 2 MB/s بدل 150 MB/s

#### السياق
فريق bring-up بيشتغل على custom board بـ TI AM62x SoC (مستخدم في industrial HMI). الـ eMMC Kingston 32GB موصل على الـ main MMC interface. الـ read سريع (130 MB/s) لكن الـ write بطيء جداً (2 MB/s) — بيخلي flashing العملية تاخد ساعات.

#### المشكلة
الـ write بطيء بشكل غير طبيعي. الـ dmesg نظيف — مفيش errors.

#### التحليل
أول حاجة: نفحص الـ background operations. في `struct mmc_ext_csd`:

```c
bool    bkops;           /* background operations support */
bool    man_bkops_en;    /* manual bkops enable bit */
bool    auto_bkops_en;   /* auto bkops enable bit */
u8      raw_bkops_status; /* byte 246 */
```

```bash
mmc extcsd read /dev/mmcblk0 | grep -i bkops
# BKOPS_EN: 0x01   # manual bkops enabled
# BKOPS_STATUS: 0x03  # ← critical! الـ eMMC محتاج bkops بشدة
```

الـ `raw_bkops_status = 3` معناه الـ chip في حالة critical background operations. هو بيخصص معظم وقته في internal garbage collection بدل الـ host writes.

ثانيًا: نفحص الـ `cache` state:

```c
u8              cache_ctrl;   /* cache control byte */
unsigned int    cache_size;   /* Units: KB */
bool            hpi_en;       /* HPI enable bit */
bool            hpi;          /* HPI support bit */
unsigned int    hpi_cmd;      /* cmd used as HPI */
```

```bash
mmc extcsd read /dev/mmcblk0 | grep -i cache
# CACHE_SIZE: 4096 KB   # 4MB cache موجود
# CACHE_CTRL: 0x00      # ← الكاش معطل!
```

الـ 4MB cache في الـ eMMC معطل — كل write بيروح مباشرة للـ NAND.

ثالثًا: نفحص الـ `erase_group_def`:

```c
u8    erase_group_def;   /* erase group definition */
unsigned int    hc_erase_size;    /* In sectors */
unsigned int    hc_erase_timeout; /* In milliseconds */
```

```bash
mmc extcsd read /dev/mmcblk0 | grep -E "ERASE_GROUP|HC_ERASE"
# ERASE_GROUP_DEF: 0x00   # ← HC erase group معطل
# HC_ERASE_GRP_SIZE: 0x04  # 512KB erase group
```

الـ kernel بيستخدم الـ legacy erase group (512 bytes) بدل الـ HC erase group (512KB) — ده بيخلي كل write يحتاج erase overhead هائل.

#### الحل
**خطوة 1**: تفعيل الـ HC erase group وكاش الـ eMMC:

```bash
# تفعيل HC erase group
mmc extcsd write /dev/mmcblk0 175 0x01   # ERASE_GROUP_DEF = 1

# تفعيل الكاش
mmc extcsd write /dev/mmcblk0 33 0x01    # CACHE_CTRL = 1

# إجبار الـ bkops الآن عشان ننهي الـ garbage collection
mmc extcsd write /dev/mmcblk0 164 0x01   # BKOPS_START = 1
sleep 30
```

**خطوة 2**: في DT لـ AM62x:

```dts
&sdhci1 {
    pinctrl-names = "default";
    pinctrl-0 = <&emmc_pins_default>;
    ti,driver-strength-ohm = <50>;
    disable-wp;
    bus-width = <8>;
    non-removable;
    mmc-hs200-1_8v;     /* HS200 لأقصى أداء */
    no-1-8-v;           /* حذف لو الـ voltage regulator جاهز */
    status = "okay";
};
```

**خطوة 3**: التحقق من نتيجة الـ `erase_group_def` في kernel:

```c
/* الـ mmc_card.ext_csd.erase_group_def يؤثر على mmc_card.erase_size */
/* بعد SET_EXTCSD(ERASE_GROUP_DEF, 1): */
/* card->ext_csd.hc_erase_size يُستخدم بدل card->csd.erase_size */
```

```bash
# بعد التعديل
dd if=/dev/zero of=/dev/mmcblk0 bs=1M count=100 oflag=direct
# 100 MB/s+ ← improvement هائل

# فحص الحالة الجديدة
cat /sys/kernel/debug/mmc0/mmc0:0001/ext_csd | \
  python3 -c "
import sys
data = sys.stdin.read().strip()
bytes_ = [int(data[i:i+2],16) for i in range(0,len(data),2)]
print(f'ERASE_GROUP_DEF (175): {bytes_[175]}')
print(f'CACHE_CTRL (33): {bytes_[33]}')
print(f'BKOPS_STATUS (246): {bytes_[246]}')
"
```

#### الدرس المستفاد
الـ `mmc_ext_csd` struct في `card.h` هو mirror كامل لـ EXT_CSD register. ثلاثة fields بتأثر على write performance بشكل درامي: `erase_group_def` (HC vs legacy erase groups)، `cache_ctrl` (write cache)، و `raw_bkops_status` (internal GC state). أثناء bring-up، دايمًا افحص الثلاثة دول قبل ما تحكم إن الـ hardware بطيء.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

**الـ** LWN.net هو المرجع الأساسي لمتابعة تطور subsystem الـ MMC في الـ Linux kernel.

| المقال | الوصف |
|--------|-------|
| [new driver: MMC framework (2002)](https://lwn.net/Articles/7176/) | أول patch أضافت الـ MMC framework للـ kernel، اتعملت اختبارها على iPAQ H3800 |
| [1/4 MMC layer (2003)](https://lwn.net/Articles/82765/) | الـ patch اللي أضافت core support لـ MMC interfaces في الـ embedded devices زي PXA وARM MMCI |
| [Secure Digital (SD) support](https://lwn.net/Articles/126098/) | إضافة دعم الـ SD cards وفرق parsing الـ CSD بين SD وMMC |
| [MMC updates — SDIO & SPI](https://lwn.net/Articles/253888/) | التحديثات اللي جابت دعم SDIO وSPI للـ MMC layer |
| [Add MMC erase and secure erase V4](https://lwn.net/Articles/395147/) | إضافة عمليات الـ erase والـ secure erase للـ MMC |
| [mmc: core: add SD4.0 support](https://lwn.net/Articles/642336/) | إضافة دعم SD 4.0 protocol |
| [Replay Protected Memory Block (RPMB) subsystem](https://lwn.net/Articles/682276/) | المناقشة الأولى لإضافة RPMB subsystem — تخزين محمي للـ TEE |
| [char:rpmb: RPMB subsystem (v2)](https://lwn.net/Articles/705850/) | الـ RPMB subsystem بيشمل eMMC وUFS وNVMe بنفس الـ frame layout |
| [mmc: Add Command Queue support](https://lwn.net/Articles/738160/) | إضافة الـ Command Queue للـ eMMC مع دعم blk-mq |
| [Add MMC software queue support](https://lwn.net/Articles/803435/) | الـ software queue كبديل للـ CQHCI hardware queue |
| [mmc: introduce multi-block read gap tuning](https://lwn.net/Articles/1030167/) | تحسين أداء الـ MMC بضبط الـ gap في قراءة الـ multi-block |

---

### التوثيق الرسمي في الـ Kernel

الملفات دي موجودة داخل شجرة الـ kernel في `Documentation/driver-api/mmc/`:

```
Documentation/driver-api/mmc/index.rst          -- نقطة الدخول الرئيسية
Documentation/driver-api/mmc/mmc-tools.rst      -- شرح أدوات mmc-utils
Documentation/driver-api/mmc/mmc-async-req.rst  -- الـ async request interface
Documentation/driver-api/mmc/mmc-dev-attrs.rst  -- device attributes في sysfs
Documentation/driver-api/mmc/mmc-dev-parts.rst  -- physical partitions في eMMC
```

**الرابط الكامل للتوثيق:**
- [MMC/SD/SDIO card support — kernel.org docs](https://docs.kernel.org/driver-api/mmc/index.html)
- [MMC tools introduction](https://docs.kernel.org/driver-api/mmc/mmc-tools.html)

**الـ header files الأساسية في الـ source tree:**

```
include/linux/mmc/card.h     -- struct mmc_card وكل الـ CID/CSD/EXT_CSD structs
include/linux/mmc/host.h     -- struct mmc_host
include/linux/mmc/mmc.h      -- MMC commands وregister definitions
include/linux/mmc/sd.h       -- SD-specific definitions
include/linux/mmc/sdio.h     -- SDIO definitions
drivers/mmc/core/            -- MMC core layer
drivers/mmc/host/            -- host controller drivers
```

---

### Commits مهمة في الـ Kernel History

| الـ Commit / Patchwork | الوصف |
|------------------------|-------|
| [RPMB partition registration with RPMB subsystem](https://linux.kernel.narkive.com/7zfKwQDD/patch-v7-08-11-mmc-block-register-rpmb-partition-with-the-rpmb-subsystem) | ربط الـ eMMC RPMB partition بالـ RPMB subsystem |
| [rpmb: add RPMB subsystem — Patchwork v4](https://patchwork.kernel.org/project/linux-mmc/patch/1464817292-5407-2-git-send-email-tomas.winkler@intel.com/) | الـ patch الرابعة للـ RPMB subsystem |
| [eMMC Command Queue — Patchwork](https://patchwork.kernel.org/patch/5419421/) | إضافة Command Queue support في الـ mmc block layer |

**للبحث عن commits بنفسك:**
```bash
git log --oneline drivers/mmc/core/
git log --oneline include/linux/mmc/card.h
git log --grep="RPMB" --oneline drivers/mmc/
git log --grep="cmdq" --oneline drivers/mmc/
```

---

### نقاشات الـ Mailing List

**الـ** mailing list الرسمي هو `linux-mmc@vger.kernel.org`.

| الرابط | الموضوع |
|--------|---------|
| [linux-mmc@vger.kernel.org — mail-archive](https://www.mail-archive.com/linux-mmc@vger.kernel.org/) | أرشيف كامل لنقاشات الـ MMC mailing list |
| [RPMB partition access discussion](https://www.mail-archive.com/linux-mmc@vger.kernel.org/msg27797.html) | نقاش حول منع وصول RPMB partition للقراءة/الكتابة العادية |
| [eMMC 5.1 Command Queue discussion](https://www.spinics.net/lists/linux-mmc/msg52635.html) | نقاش تقني حول Command Queue support في eMMC 5.1 |
| [Linux 6.12 RPMB Subsystem — Phoronix](https://www.phoronix.com/news/Linux-6.12-RPMB-MMC) | إضافة الـ RPMB subsystem في kernel 6.12 |

---

### KernelNewbies — تحديثات الـ MMC عبر الإصدارات

**الـ** [kernelnewbies.org](https://kernelnewbies.org) بيوثق التغييرات في كل إصدار kernel.

| الإصدار | الرابط | أهم تغييرات MMC |
|---------|--------|----------------|
| 2.6.24 | [Linux_2_6_24](https://kernelnewbies.org/Linux_2_6_24) | إضافة SDIO وSPI support — تحول جذري في الـ MMC layer |
| 3.9 | [Linux_3.9](https://kernelnewbies.org/Linux_3.9) | تحسينات الـ eMMC والـ HS200 mode |
| 4.0 | [Linux_4.0-DriversArch](https://kernelnewbies.org/Linux_4.0-DriversArch) | MMC power sequences وتحسين hardware support |
| 5.7 | [Linux_5.7](https://kernelnewbies.org/Linux_5.7) | MMC software queue support |
| 6.7 | [Linux_6.7](https://kernelnewbies.org/Linux_6.7) | تحديثات MMC subsystem |
| 6.8 | [Linux_6.8](https://kernelnewbies.org/Linux_6.8) | تحديثات MMC subsystem |

---

### eLinux.org — موارد الـ Embedded

| الرابط | المحتوى |
|--------|---------|
| [Tests:eMMC-8bit-width](https://elinux.org/Tests:eMMC-8bit-width) | اختبار الـ eMMC بـ 8-bit bus width |
| [Tests:SDIO-KS7010](https://elinux.org/Tests:SDIO-KS7010) | اختبار الـ SDIO wireless driver |
| [SD/MMC High-Speed Support (PDF)](https://www.elinux.org/File:Clement-sd-mmc-high-speed-support-in-linux-kernel.pdf) | ورقة بحثية عن الـ High-Speed SD/MMC support في الـ Linux kernel |

---

### كتب موصى بيها

#### Linux Device Drivers 3 (LDD3)
- **الفصل 14** — The Linux Device Model: فاهم ازاي الـ `struct device` (اللي موجودة جوه `struct mmc_card`) بتشتغل
- متاح مجانًا: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 1, 17** — Block devices وازاي الـ MMC block driver بيتعامل مع الـ block layer
- الكتاب بيشرح الـ request queues والـ I/O scheduler اللي بيأثر على أداء الـ eMMC

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 9** — File Systems: بيشمل الـ MTD وازاي الـ SD/MMC cards بتتهيأ في الـ embedded systems
- **الفصل 15** — Porting Linux: بيشمل إضافة support لـ new MMC host controllers

#### Essential Linux Device Drivers — Sreekrishnan Venkateswaran
- **الفصل 14** — بيتكلم عن الـ MMC/SD driver architecture بشكل تفصيلي

---

### المواصفات الرسمية (Standards)

| المواصفة | المصدر |
|----------|--------|
| JEDEC eMMC Standard (JESD84-B51) | [jedec.org](https://www.jedec.org/standards-documents/docs/jesd84-b51) — eMMC 5.1 spec |
| SD Physical Layer Specification | [sdcard.org](https://www.sdcard.org/developers/sd-standard-overview/) — SD/SDHC/SDXC/SDUC |
| SDIO Specification | [sdcard.org](https://www.sdcard.org/developers/sd-standard-overview/) |

---

### Search Terms للبحث عن معلومات أكتر

للبحث في الـ kernel source:
```bash
# ابحث عن كل الـ quirks المعرّفة
grep -r "MMC_QUIRK_" drivers/mmc/ include/linux/mmc/

# ابحث عن استخدام mmc_card في الـ drivers
grep -r "struct mmc_card" drivers/mmc/core/

# ابحث عن الـ EXT_CSD registers
grep -r "EXT_CSD_" drivers/mmc/core/mmc.c

# ابحث عن الـ RPMB operations
grep -r "rpmb" drivers/mmc/ --include="*.c"
```

**Search terms مفيدة للـ web:**
```
linux kernel mmc_card struct explanation
eMMC EXT_CSD registers Linux driver
SDIO function driver Linux tutorial
MMC quirks list Linux kernel
linux mmc command queue CQHCI driver
eMMC RPMB Linux trusted execution environment
SD UHS-II Linux kernel support
mmc_host mmc_card relationship Linux
linux mmc block driver internals
```

**للبحث في الـ mailing list:**
```
site:lore.kernel.org mmc card quirk
site:patchwork.kernel.org linux-mmc
```
## Phase 8: Writing simple module

### الفكرة

**`mmc_card_is_blockaddr()`** هي الدالة الوحيدة المُعلنة exported في الملف (مش `static inline`). بتبص على الـ card وبتقول هل هي **block-addressed** (SDHC/SDXC/eMMC ≥ 2 GB) أو byte-addressed (SD قديمة). ده معلومة مهمة لأن كل قراءة/كتابة على الكارت بتتأثر بيها.

هنستخدم **kprobe** على `mmc_card_is_blockaddr` عشان نطبع معلومات عن الـ card في كل مرة بتتاكل الدالة دي — زي نوع الكارت (MMC/SD/SDIO)، manufacturer ID، اسم المنتج، وسعة السيكتورات.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * mmc_card_probe.c
 *
 * kprobe on mmc_card_is_blockaddr() — prints card identity info
 * every time the kernel checks whether a card is block-addressed.
 */

#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit */
#include <linux/kprobes.h>     /* kprobe, register_kprobe, ... */
#include <linux/mmc/card.h>    /* struct mmc_card, mmc_card_mmc(), ... */
#include <linux/printk.h>      /* pr_info() */

/* ------------------------------------------------------------------ */
/* pre-handler: runs BEFORE mmc_card_is_blockaddr() executes          */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * mmc_card_is_blockaddr(struct mmc_card *card)
     * On x86-64: first argument lives in RDI register.
     * On ARM64:  first argument lives in X0  register.
     * regs_get_kernel_argument() abstracts this for us.
     */
    struct mmc_card *card = (struct mmc_card *)regs_get_kernel_argument(regs, 0);

    if (!card)
        return 0;   /* defensive: never dereference NULL */

    /* Determine human-readable card type string */
    const char *type_str =
        mmc_card_mmc(card)      ? "eMMC/MMC" :
        mmc_card_sd(card)       ? "SD"        :
        mmc_card_sdio(card)     ? "SDIO"      :
        mmc_card_sd_combo(card) ? "SD-combo"  : "unknown";

    pr_info("[mmc_probe] mmc_card_is_blockaddr() called\n");
    pr_info("[mmc_probe]   type       : %s\n",         type_str);
    pr_info("[mmc_probe]   manfid     : 0x%06x\n",     card->cid.manfid);
    pr_info("[mmc_probe]   prod_name  : %.8s\n",       card->cid.prod_name);
    pr_info("[mmc_probe]   serial     : 0x%08x\n",     card->cid.serial);
    pr_info("[mmc_probe]   year/month : %u/%02u\n",    card->cid.year,
                                                        card->cid.month);
    pr_info("[mmc_probe]   sectors    : %u  (~%llu MB)\n",
            card->ext_csd.sectors,
            (unsigned long long)card->ext_csd.sectors >> 11); /* /2048 */
    pr_info("[mmc_probe]   cmdq       : %s\n",
            card->ext_csd.cmdq_en ? "enabled" : "disabled");

    return 0; /* 0 = continue normal execution of the probed function */
}

/* ------------------------------------------------------------------ */
/* kprobe descriptor                                                   */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "mmc_card_is_blockaddr", /* function to probe      */
    .pre_handler = handler_pre,             /* our callback           */
};

/* ------------------------------------------------------------------ */
/* module_init                                                         */
/* ------------------------------------------------------------------ */
static int __init mmc_card_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[mmc_probe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[mmc_probe] planted kprobe at %s (%p)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit                                                         */
/* ------------------------------------------------------------------ */
static void __exit mmc_card_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("[mmc_probe] kprobe removed from %s\n", kp.symbol_name);
}

module_init(mmc_card_probe_init);
module_exit(mmc_card_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe demo: log MMC card info on mmc_card_is_blockaddr()");
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `<linux/module.h>` | ضروري لأي module — بيوفر `module_init`, `module_exit`, `MODULE_LICENSE` |
| `<linux/kprobes.h>` | بيعرّف `struct kprobe` وكل API الـ kprobes |
| `<linux/mmc/card.h>` | بيجيب `struct mmc_card` والماكروهات `mmc_card_mmc()` وغيرها |
| `<linux/printk.h>` | بيوفر `pr_info()` و `pr_err()` |

#### الـ `handler_pre` callback

الـ handler بياخد `struct pt_regs *regs` اللي فيه حالة الـ registers لحظة الـ probe. بنستخدم `regs_get_kernel_argument(regs, 0)` عشان نجيب الـ argument الأول للدالة (الـ `card`) بطريقة portable على x86-64 وARM64 من غير ما نكتب assembly. بعدين بنطبع `cid` (Card Identification Data) اللي فيها manufacturer ID واسم المنتج والـ serial والتاريخ — ده بيساعدنا نعرف بالظبط إيه الكارت اللي الكيرنل بيتعامل معاها.

#### الـ `kp` struct

**`symbol_name`** بيقول للكيرنل هيـ probe على أنهي دالة بالاسم — الكيرنل بيبحث في الـ kallsyms بنفسه ويملي `kp.addr`. **`pre_handler`** بيحدد الـ callback اللي هيتشغل *قبل* تنفيذ الدالة، وده المناسب هنا لأننا بس عايزين نقرأ الـ arguments.

#### `module_init` / `module_exit`

**`register_kprobe`** بتزرع الـ breakpoint في الكيرنل live من غير إعادة compile. لازم نتأكد من الـ return value لأن الدالة ممكن تفشل (مثلاً لو الـ symbol مش موجود أو متعلمش probe عليه). في الـ exit، **`unregister_kprobe`** ضرورية عشان تشيل الـ breakpoint قبل ما الـ module يتشال من الذاكرة — لو متشلتش، الكيرنل هيكرش لأنه هيحاول يجري handler بعد ما اتحذف من الذاكرة.

---

### كيفية البناء والتشغيل

```bash
# Makefile بسيط جنب mmc_card_probe.c
cat > Makefile <<'EOF'
obj-m += mmc_card_probe.o
KDIR  ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
EOF

make
sudo insmod mmc_card_probe.ko
# شغّل أي عملية تقرأ MMC/SD أو افحص الكارت
dmesg | grep mmc_probe
sudo rmmod mmc_card_probe
```

### مثال على الـ Output المتوقع

```
[mmc_probe] planted kprobe at mmc_card_is_blockaddr (ffffffffc0a12340)
[mmc_probe] mmc_card_is_blockaddr() called
[mmc_probe]   type       : SD
[mmc_probe]   manfid     : 0x000003
[mmc_probe]   prod_name  : SR32G
[mmc_probe]   serial     : 0xdeadbeef
[mmc_probe]   year/month : 2022/04
[mmc_probe]   sectors    : 62333952  (~30441 MB)
[mmc_probe]   cmdq       : disabled
```

الـ `manfid = 0x000003` يعني **SanDisk** — ده standard JEDEC manufacturer ID موثق في مواصفات SD.
