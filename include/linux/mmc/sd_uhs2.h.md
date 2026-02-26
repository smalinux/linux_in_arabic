## Phase 1: الصورة الكبيرة ببساطة

### ما هو هذا الملف؟

الملف `include/linux/mmc/sd_uhs2.h` هو **header** بتاع بروتوكول **UHS-II** (Ultra High Speed II) — الجيل الجديد من واجهة كروت SD اللي بتشتغل بسرعات عالية جداً.

---

### تشبيه بسيط

تخيل إن كرت SD القديم (UHS-I) زي أوتوستراد بـ lane واحدة. **UHS-II** جاء وعمل أوتوستراد بـ lanes متعددة وبروتوكول تواصل جديد خالص — مش مجرد "نفس الكابل بسرعة أعلى"، ده نظام حزم (packet-based) من الصفر.

الـ host (جهازك) والكرت بيتكلموا بـ **packets** منظمة بدل الـ commands البسيطة القديمة، وعندهم **طبقات** (LINK layer + TRANS layer) زي بروتوكول شبكات.

---

### ليه الملف ده موجود؟

الـ header ده بيعرّف:

| الفئة | المحتوى |
|---|---|
| **Packet Format** | تعريفات Header وArgument لكل نوع packet |
| **Packet Types** | CCMD, DCMD, RES, DATA, MSG |
| **Device Registers** | عناوين registers الكرت (PHY, LINK, Generic, CMD) |
| **I/O Addresses** | `IOADR_*` للوصول لإعدادات الكرت |
| **Clock Limits** | `UHS2_RCLK_MAX/MIN` للـ reference clock |

بدون الـ macros دي، driver مش قادر يبني أي packet يبعته للكرت ولا يقرأ capabilities بتاعته.

---

### القصة من الأول للآخر

```
[Host Controller] ──UHS-II bus (2 lanes)──► [SD Card]
      │
      │  1. يبعت CCMD (Control Command Packet) يسأل عن Capabilities
      │  2. الكرت يرد بـ RES packet
      │  3. Host يعمل Enumerate (يديه Node ID)
      │  4. يتفاوضوا على PHY speed (A أو B) وعدد الـ lanes
      │  5. يبدأ نقل البيانات بـ DCMD + DATA packets
      │  6. لو فيه interrupt → MSG packet من الكرت
```

الـ header ده بيغطي **Layer 1 و2** من البروتوكول ده (LINK + TRANS layers).

---

### Packet Header — بالتفصيل

```
 15  14  13  12 | 11 | 10   9   8 | 7 |  6   5   4  |  3   2   1   0
 SID[3:0]       | -- | TID[2:0]   | NP| TYP[2:0]    | DID[3:0]
```

- **DID**: Destination ID — مين المستقبل (الكرت)
- **TYP**: نوع الـ packet (CCMD / DCMD / RES / DATA / MSG)
- **NP**: Native Packet — يعني packet خاص بـ UHS-II نفسه مش SD command عادي
- **TID**: Transaction ID — رقم العملية
- **SID**: Source ID — مين المرسل (الـ Host = 0)

---

### أنواع الـ Packets

```c
#define UHS2_PACKET_TYPE_CCMD  (0 << 4)  // Control Command
#define UHS2_PACKET_TYPE_DCMD  (1 << 4)  // Data Command
#define UHS2_PACKET_TYPE_RES   (2 << 4)  // Response
#define UHS2_PACKET_TYPE_DATA  (3 << 4)  // Data Payload
#define UHS2_PACKET_TYPE_MSG   (7 << 4)  // Message (e.g. interrupt)
```

---

### Device Registers المهمة

```
0x000 → Generic Caps      (عدد الـ lanes، نوع التطبيق SD/SDIO)
0x002 → PHY Caps          (version، hibernate support)
0x004 → LINK/TRAN Caps    (FCU count، max block length)
0x008 → Generic Settings  (ضبط الـ lanes)
0x00A → PHY Settings      (Speed A = ~1.56 Gbps / Speed B = ~3.12 Gbps)
0x00C → LINK/TRAN Settings
0x040 → Preset Register
0x100 → Interrupt Register
0x180 → Status Register
0x200 → Command Registers (FULL_RESET, GO_DORMANT, DEVICE_INIT, ENUMERATE)
```

---

### الملفات المرتبطة

#### Core Files (المنطق الأساسي)

| الملف | الدور |
|---|---|
| `drivers/mmc/core/sd_uhs2.c` | تنفيذ initialization وenumeration وdata transfer |
| `drivers/mmc/core/sd.c` | الـ SD card driver العام |
| `drivers/mmc/core/core.c` | الـ MMC subsystem core |
| `drivers/mmc/core/sd_ops.c` | SD commands implementation |

#### Headers المهمة

| الملف | الدور |
|---|---|
| `include/linux/mmc/sd_uhs2.h` | **هذا الملف** — UHS-II constants وregister map |
| `include/linux/mmc/host.h` | يعرّف `struct mmc_host` وبيضم الـ UHS-II timing modes |
| `include/linux/mmc/core.h` | يعرّف `struct uhs2_command` و`struct mmc_command` |
| `include/linux/mmc/card.h` | يعرّف `struct mmc_card` للكرت نفسه |
| `include/linux/mmc/sd.h` | SD-specific constants (OCR, SCR, SSR) |

#### Host Controller Drivers

| الملف | الدور |
|---|---|
| `drivers/mmc/host/sdhci-uhs2.c` | SDHCI controller مع دعم UHS-II |
| `drivers/mmc/host/sdhci-uhs2.h` | Register definitions للـ SDHCI hardware |

---

### ملاحظة على الـ Clock

```c
#define UHS2_RCLK_MAX  52000000   // 52 MHz reference clock max
#define UHS2_RCLK_MIN  26000000   // 26 MHz reference clock min
```

الـ reference clock ده بيتحكم في timing الـ initialization phase — مختلف عن الـ data transfer speed اللي بتوصل لـ Gbps.
## Phase 2: شرح الـ SD UHS-II Framework

---

### 1. المشكلة — ليه الـ Subsystem ده موجود؟

الـ SD cards القديمة (UHS-I) بتشتغل على interface واحد: single-ended 1-bit أو 4-bit data bus بحد أقصى ~104 MB/s (SDR104). ده بقى ضيّق للتطبيقات الحديثة زي 4K video recording، industrial logging، وembedded AI inference اللي محتاجة throughput أعلى وlatency أقل.

**UHS-II** جه يحل المشكلة بـ:
- Interface جديد خالص: **2-lane differential serial link** (LVDS-based) بدل الـ parallel bus القديم.
- سرعة تصل لـ **~624 MB/s** (Speed Range B, Half Duplex).
- **Packet-based protocol** بدل الـ simple command/response.
- **Layered architecture** (LINK + TRANS layers) زي الـ PCIe أو USB.

لكن ده خلق مشكلة تانية: الـ kernel بيتعامل مع SD cards من خلال الـ `mmc` subsystem القديم. لازم framework جديد يـ**abstract** الـ UHS-II specifics من غير ما يكسر الـ existing `mmc_host` / `mmc_card` model.

---

### 2. الحل — الـ Kernel بيعمل إيه؟

الـ kernel أضاف **SD UHS-II layer** جوّا الـ existing MMC subsystem بدل ما يعمل subsystem منفصل. الـ approach:

| القرار | التفاصيل |
|--------|----------|
| **Extend, don't replace** | `mmc_command` اتضاف فيها `uhs2_cmd` pointer |
| **New capability flag** | `MMC_CAP2_SD_UHS2` في `mmc_host` |
| **New host op** | `uhs2_control()` callback في `mmc_host_ops` |
| **New struct** | `sd_uhs2_caps` لتخزين negotiated parameters |
| **New packet definitions** | `sd_uhs2.h` يعرّف كل الـ LINK/TRANS layer constants |
| **New timing modes** | `MMC_TIMING_UHS2_SPEED_A/B` في `mmc_ios` |

الـ framework بيـ**delegate** الـ physical layer operations (PHY init، clock control، IOS config) للـ host controller driver عن طريق الـ `uhs2_control()` callback.

---

### 3. الـ Big Picture Architecture

```
  ┌─────────────────────────────────────────────────────┐
  │              Block Device / Filesystem               │
  └────────────────────┬────────────────────────────────┘
                       │ read/write requests
  ┌────────────────────▼────────────────────────────────┐
  │              MMC Block Driver (mmc_blk)              │
  └────────────────────┬────────────────────────────────┘
                       │ mmc_request  (mmc_command + mmc_data)
  ┌────────────────────▼────────────────────────────────┐
  │              MMC Core (mmc/core/*.c)                 │
  │  ┌──────────────────────────────────────────────┐   │
  │  │          SD UHS-II Sublayer                  │   │
  │  │  • builds uhs2_command (header + arg +       │   │
  │  │    payload) from sd_uhs2.h macros            │   │
  │  │  • handles CCMD/DCMD packet construction     │   │
  │  │  • negotiates sd_uhs2_caps with card         │   │
  │  │  • calls uhs2_control() for PHY ops          │   │
  │  └──────────────────────────────────────────────┘   │
  └────────────────────┬────────────────────────────────┘
                       │ mmc_host_ops callbacks
                       │ uhs2_control(op) for UHS2 ops
  ┌────────────────────▼────────────────────────────────┐
  │         Host Controller Driver (e.g. sdhci-uhs2)    │
  │  • implements uhs2_control() enum operations        │
  │  • UHS2_PHY_INIT / UHS2_SET_CONFIG / UHS2_SET_IOS  │
  │  • UHS2_ENABLE/DISABLE_CLK / UHS2_CHECK_DORMANT    │
  └────────────────────┬────────────────────────────────┘
                       │ hardware registers / DMA
  ┌────────────────────▼────────────────────────────────┐
  │         UHS-II Host Controller (Silicon)             │
  │   [RCLK: 26–52 MHz ref clock]  [2-lane LVDS]        │
  └────────────────────┬────────────────────────────────┘
                       │  Lane 0 (D+/D-)  Lane 1 (D+/D-)
  ┌────────────────────▼────────────────────────────────┐
  │              SD UHS-II Card                          │
  │  Config Regs: 0x000–0x1FF  |  Int Regs: 0x100       │
  │  Status Regs: 0x180        |  CMD Regs: 0x200        │
  └─────────────────────────────────────────────────────┘

  Packet Types (sd_uhs2.h):
  CCMD (Control Cmd) | DCMD (Data Cmd) | RES (Response)
  DATA (Payload)     | MSG (Flow Ctrl / Interrupt)
```

---

### 4. الـ Real-World Analogy

تخيّل **بروتوكول التحدث مع موظف البنك عبر الإنترنت**:

| Analogy | Kernel Concept |
|---------|---------------|
| رقم العميل (DID) | `UHS2_DEST_ID` — Node ID of card |
| رقم المعاملة (TID) | `UHS2_TRANS_ID` — Transaction tracking |
| نوع الطلب (تحويل / استعلام) | Packet Type: `CCMD` vs `DCMD` |
| عنوان السجل المطلوب | `IOADR` — I/O Address in UHS-II space |
| حجم الـ payload المرفق | `PLEN` bits في الـ Argument field |
| رسالة "جهّز السطر" | `MSG` packet — Flow Control FCREQ/FCRDY |
| تأكيد أو رفض الطلب | `RES` packet — NACK bit + ECODE field |
| الموظف بيوافق على الاتصال أولاً | `UHS2_PHY_INIT` → `UHS2_DEV_INIT` sequence |

الـ `sd_uhs2.h` هو اللي بيعرّف "لغة التخاطب" دي كلها بالكامل.

---

### 5. الـ Core Abstraction

الـ abstraction المحورية هي **`uhs2_command`** struct (في `core.h`) مع الـ **packet macros** (في `sd_uhs2.h`):

```c
/* core.h — الـ packet structure اللي بتتبعت على الـ wire */
struct uhs2_command {
    u16  header;          /* DID + TYP + NP + TID + SID */
    u16  arg;             /* IOADR + PLEN + R/W  (CCMD)  */
                          /* TMODE + R/W          (DCMD)  */
    __be32 payload[2];    /* up to 16 bytes of data       */
    u8   payload_len;
    u8   packet_len;
    u8   tmode_half_duplex;
    u8   uhs2_resp[20];   /* raw response bytes           */
    u8   uhs2_resp_len;
};
```

**بتتضمّن في `mmc_command`:**
```c
struct mmc_command {
    /* ... existing SD/MMC fields ... */
    struct uhs2_command *uhs2_cmd;   /* NULL = legacy SD, !NULL = UHS-II */
};
```

يعني نفس الـ `mmc_request` pipeline بيشتغل، بس لو الـ `uhs2_cmd != NULL` الـ host driver بيبعت UHS-II packet بدل الـ legacy command.

**الـ `sd_uhs2_caps` struct** (في `host.h`) بتخزّن نتيجة الـ capability negotiation:

```c
struct sd_uhs2_caps {
    u8  n_lanes;        /* number of lanes negotiated       */
    u8  speed_range;    /* Speed A (0) or Speed B (1)       */
    u8  phy_rev;        /* PHY layer version                */
    u32 n_fcu;          /* Flow Control Units               */
    u32 maxblk_len;     /* max block length supported       */
    /* ... + _set variants for negotiated values ... */
};
```

---

### 6. الـ Framework بيمتلك إيه vs بيـDelegate إيه؟

| المسؤولية | مين بيتكفل بيها |
|-----------|----------------|
| تعريف packet format (header/arg bits) | **Framework** (`sd_uhs2.h` macros) |
| تعريف I/O Address space (IOADR) | **Framework** (`sd_uhs2.h` constants) |
| بناء الـ `uhs2_command` من SD commands | **Framework** (MMC core UHS-II sublayer) |
| negotiation الـ `sd_uhs2_caps` مع الكارت | **Framework** (MMC core) |
| إدارة الـ `UHS2_DEV_INIT` / `ENUM` sequence | **Framework** (MMC core) |
| تنفيذ الـ PHY initialization | **Driver** عبر `uhs2_control(UHS2_PHY_INIT)` |
| ضبط الـ clock وتشغيله/إيقافه | **Driver** عبر `uhs2_control(UHS2_ENABLE/DISABLE_CLK)` |
| ضبط الـ IOS (speed, lanes) على الـ HW | **Driver** عبر `uhs2_control(UHS2_SET_IOS)` |
| إدارة الـ Dormant state على الـ HW | **Driver** عبر `uhs2_control(UHS2_CHECK_DORMANT)` |
| تفسير الـ DMA والـ scatter-gather | **Driver** (existing `request()` callback) |
| الـ interrupt handling للـ MSG packets | **Driver** (HW-specific) |

**الخلاصة:** الـ framework بيمتلك الـ **protocol logic** (معرفة شكل الـ packets ومعنى كل bit)، والـ driver بيمتلك الـ **physical interface** (التحكم في الـ silicon).
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. Cheatsheet — الـ Flags والـ Enums والـ Macros

#### Packet Types (TYP field — bits [6:4] في الـ Header)

| Macro | قيمة | المعنى |
|---|---|---|
| `UHS2_PACKET_TYPE_CCMD` | `0 << 4` | Control Command packet |
| `UHS2_PACKET_TYPE_DCMD` | `1 << 4` | Data Command packet |
| `UHS2_PACKET_TYPE_RES`  | `2 << 4` | Response packet |
| `UHS2_PACKET_TYPE_DATA` | `3 << 4` | Data Payload packet |
| `UHS2_PACKET_TYPE_MSG`  | `7 << 4` | Message packet |

#### Header Field Masks

| Macro | Bits | المعنى |
|---|---|---|
| `UHS2_DEST_ID_MASK`   | [3:0]   | Destination Node ID |
| `UHS2_DEST_ID`        | `0x1`   | Default card DID |
| `UHS2_TRANS_ID_MASK`  | [10:8]  | Transaction ID |
| `UHS2_SRC_ID_MASK`    | [15:12] | Source ID (Host = 0) |
| `UHS2_NATIVE_PACKET`  | bit 7   | Native packet flag |

#### MSG Packet Categories

| Macro | قيمة | المعنى |
|---|---|---|
| `UHS2_MSG_CTG_LMSG` | `0x00` | Link message |
| `UHS2_MSG_CTG_INT`  | `0x60` | Interrupt |
| `UHS2_MSG_CTG_AMSG` | `0x80` | Application message |
| `UHS2_MSG_CTG_FCREQ`| `0x00` | Flow Control Request |
| `UHS2_MSG_CTG_FCRDY`| `0x01` | Flow Control Ready |
| `UHS2_MSG_CTG_STAT` | `0x02` | Status |

#### MSG Error Codes

| Macro | قيمة | المعنى |
|---|---|---|
| `UHS2_MSG_CODE_FC_UNRECOVER_ERR`   | `0x8` | Unrecoverable FC error |
| `UHS2_MSG_CODE_STAT_UNRECOVER_ERR` | `0x8` | Unrecoverable status error |
| `UHS2_MSG_CODE_STAT_RECOVER_ERR`   | `0x1` | Recoverable status error |

#### CCMD Argument (TRANS Layer)

| Macro | Bits | المعنى |
|---|---|---|
| `UHS2_NATIVE_CMD_WRITE`    | bit 7      | R/W = Write |
| `UHS2_NATIVE_CMD_READ`     | bit 7 = 0  | R/W = Read |
| `UHS2_NATIVE_CMD_PLEN_4B`  | `1 << 4`   | Payload = 4 bytes |
| `UHS2_NATIVE_CMD_PLEN_8B`  | `2 << 4`   | Payload = 8 bytes |
| `UHS2_NATIVE_CMD_PLEN_16B` | `3 << 4`   | Payload = 16 bytes |

#### DCMD Transfer Mode Flags (bits [6:3])

| Macro | Bit | المعنى |
|---|---|---|
| `UHS2_DCMD_2L_HD_MODE`      | bit 6 | 2-Lane Half-Duplex |
| `UHS2_DCMD_LM_TLEN_EXIST`   | bit 5 | TLEN field موجود |
| `UHS2_DCMD_TLUM_BYTE_MODE`  | bit 4 | TLEN unit = Bytes |
| `UHS2_NATIVE_DCMD_DAM_IO`   | bit 3 | Data Access Mode = IO |

#### Response Error Codes (RES packet)

| Macro | قيمة | المعنى |
|---|---|---|
| `UHS2_RES_NACK_MASK`   | bit 7 | NACK flag |
| `UHS2_RES_ECODE_COND`  | `1`   | Condition error |
| `UHS2_RES_ECODE_ARG`   | `2`   | Argument error |
| `UHS2_RES_ECODE_GEN`   | `3`   | General error |

#### Device Register Map (IOADR)

| Macro | عنوان | السجل |
|---|---|---|
| `UHS2_DEV_CONFIG_REG`           | `0x000` | Config region base |
| `UHS2_DEV_CONFIG_GEN_CAPS`      | `0x000` | Generic Capabilities |
| `UHS2_DEV_CONFIG_PHY_CAPS`      | `0x002` | PHY Capabilities |
| `UHS2_DEV_CONFIG_LINK_TRAN_CAPS`| `0x004` | LINK-TRAN Capabilities |
| `UHS2_DEV_CONFIG_GEN_SET`       | `0x008` | Generic Settings |
| `UHS2_DEV_CONFIG_PHY_SET`       | `0x00A` | PHY Settings |
| `UHS2_DEV_CONFIG_LINK_TRAN_SET` | `0x00C` | LINK-TRAN Settings |
| `UHS2_DEV_CONFIG_PRESET`        | `0x040` | Preset Values |
| `UHS2_DEV_INT_REG`              | `0x100` | Interrupt region |
| `UHS2_DEV_STATUS_REG`           | `0x180` | Status region |
| `UHS2_DEV_CMD_REG`              | `0x200` | Command region |

#### Device Command Register Offsets

| Macro | عنوان | الأمر |
|---|---|---|
| `UHS2_DEV_CMD_FULL_RESET`      | `0x200` | Full Reset |
| `UHS2_DEV_CMD_GO_DORMANT_STATE`| `0x201` | Go Dormant |
| `UHS2_DEV_CMD_DEVICE_INIT`     | `0x202` | Device Init |
| `UHS2_DEV_CMD_ENUMERATE`       | `0x203` | Enumerate |
| `UHS2_DEV_CMD_TRANS_ABORT`     | `0x204` | Transaction Abort |

#### Lane Configuration (N_LANES)

| Macro | قيمة | التوصيف |
|---|---|---|
| `UHS2_DEV_CONFIG_2L_HD_FD`  | `0x1` | 2-Lane HD/FD |
| `UHS2_DEV_CONFIG_2D1U_FD`   | `0x2` | 2D+1U Full Duplex |
| `UHS2_DEV_CONFIG_1D2U_FD`   | `0x4` | 1D+2U Full Duplex |
| `UHS2_DEV_CONFIG_2D2U_FD`   | `0x8` | 2D+2U Full Duplex |

#### PHY Speed Range

| Macro | قيمة | السرعة |
|---|---|---|
| `UHS2_DEV_CONFIG_PHY_SET_SPEED_A` | `0x0` | Range A (غير محدد) |
| `UHS2_DEV_CONFIG_PHY_SET_SPEED_B` | `0x1` | Range B |

#### Reference Clock Limits

| Macro | قيمة |
|---|---|
| `UHS2_RCLK_MAX` | 52 MHz |
| `UHS2_RCLK_MIN` | 26 MHz |

---

### 1. الـ Structs المهمة

#### `struct uhs2_command` — (linux/mmc/core.h)

**الغرض:** تمثيل حزمة UHS-II native command (CCMD أو DCMD) على مستوى الـ software قبل ما تتبعت للـ host controller.

| Field | Type | المعنى |
|---|---|---|
| `header` | `u16` | LINK layer header: DID, TYP, NP, TID, SID |
| `arg` | `u16` | TRANS layer argument: IOADR, R/W, PLEN |
| `payload[2]` | `__be32[2]` | Payload بحد أقصى 8 bytes (big-endian) |
| `payload_len` | `u8` | طول الـ payload الفعلي |
| `packet_len` | `u8` | إجمالي طول الحزمة |
| `tmode_half_duplex` | `u8` | Half-duplex transfer mode flag |
| `uhs2_resp[20]` | `u8[20]` | Buffer لاستقبال الـ native response |
| `uhs2_resp_len` | `u8` | طول الـ response الفعلي |

**ملاحظة مهمة:** الـ payload محدود بـ `UHS2_MAX_PAYLOAD_LEN = 2` entries × 4 bytes = 8 bytes، والـ response بـ `UHS2_MAX_RESP_LEN = 20` bytes.

---

#### `struct mmc_command` — (linux/mmc/core.h)

**الغرض:** تمثيل أي أمر MMC/SD/UHS2، يحمل pointer على `uhs2_command` لو الكارت UHS-II.

| Field | Type | المعنى |
|---|---|---|
| `opcode` | `u32` | رقم الأمر (CMD index) |
| `arg` | `u32` | SD/MMC argument |
| `resp[4]` | `u32[4]` | Response buffer (SD legacy) |
| `flags` | `unsigned int` | Response type flags (MMC_RSP_*) |
| `retries` | `unsigned int` | عدد مرات الإعادة |
| `error` | `int` | كود الخطأ |
| `busy_timeout` | `unsigned int` | timeout بالـ ms |
| `data` | `struct mmc_data *` | pointer على بيانات الـ transfer |
| `mrq` | `struct mmc_request *` | الـ request الأب |
| `uhs2_cmd` | `struct uhs2_command *` | UHS-II native command (لو موجود) |
| `has_ext_addr` | `bool` | Extended address (SDUC) |
| `ext_addr` | `u8` | Extended address value |

---

#### `struct mmc_request` — (linux/mmc/core.h)

**الغرض:** الوحدة الكاملة للتواصل بين الـ MMC core والـ host controller — تجمع الأوامر والبيانات مع الـ UHS2 command.

| Field | Type | المعنى |
|---|---|---|
| `sbc` | `struct mmc_command *` | SET_BLOCK_COUNT للـ multiblock |
| `cmd` | `struct mmc_command *` | الأمر الرئيسي |
| `data` | `struct mmc_data *` | بيانات الـ transfer |
| `stop` | `struct mmc_command *` | أمر الإيقاف |
| `completion` | `struct completion` | للـ synchronous waiting |
| `cmd_completion` | `struct completion` | لإكمال الـ command phase |
| `done` | `void (*)(mrq*)` | callback عند الانتهاء |
| `recovery_notifier` | `void (*)(mrq*)` | callback عند الحاجة للـ recovery |
| `host` | `struct mmc_host *` | الـ host المرتبط |
| `cap_cmd_during_tfr` | `bool` | يسمح بأوامر أثناء الـ transfer |
| `tag` | `int` | CQE tag |
| `uhs2_cmd` | `struct uhs2_command` | UHS2 command مضمّن مباشرة (not pointer) |

---

#### `struct sd_uhs2_config` — (linux/mmc/card.h)

**الغرض:** حفظ القدرات والإعدادات الحالية للكارت UHS-II بعد الـ enumeration — هي المرجع الرئيسي لكل ما قرأناه من device registers.

**قسم القدرات (Capabilities — من الكارت):**

| Field | Type | Register المصدر | المعنى |
|---|---|---|---|
| `node_id` | `u32` | Enumeration | Node ID للكارت |
| `n_fcu` | `u32` | LINK_TRAN_CAPS | عدد Flow Control Units |
| `maxblk_len` | `u32` | LINK_TRAN_CAPS | أقصى block length |
| `n_lanes` | `u8` | GEN_CAPS | عدد الـ lanes المدعومة |
| `dadr_len` | `u8` | GEN_CAPS | Data Address length |
| `app_type` | `u8` | GEN_CAPS | نوع التطبيق (SD_MEM=0x1) |
| `phy_minor_rev` | `u8` | PHY_CAPS | PHY minor revision |
| `phy_major_rev` | `u8` | PHY_CAPS | PHY major revision |
| `can_hibernate` | `u8` | PHY_CAPS | دعم Hibernation |
| `n_lss_sync` | `u8` | PHY_CAPS1 | عدد LSS sync symbols |
| `n_lss_dir` | `u8` | PHY_CAPS1 | عدد LSS dir symbols |
| `link_minor_rev` | `u8` | LINK_TRAN_CAPS | LINK minor revision |
| `link_major_rev` | `u8` | LINK_TRAN_CAPS | LINK major revision |
| `dev_type` | `u8` | LINK_TRAN_CAPS | نوع الجهاز |
| `n_data_gap` | `u8` | LINK_TRAN_CAPS1 | Data gap cycles |

**قسم الإعدادات الفعلية (Negotiated Settings):**

| Field | Type | المعنى |
|---|---|---|
| `n_fcu_set` | `u32` | FCU count تم تعيينه |
| `maxblk_len_set` | `u32` | Max block length تم تعيينه |
| `n_lanes_set` | `u8` | Lane count تم تعيينه |
| `speed_range_set` | `u8` | Speed range (A أو B) |
| `n_lss_sync_set` | `u8` | LSS sync تم تعيينه |
| `n_lss_dir_set` | `u8` | LSS dir تم تعيينه |
| `n_data_gap_set` | `u8` | Data gap تم تعيينه |
| `max_retry_set` | `u8` | Max retry count |

---

#### `struct mmc_card` — (linux/mmc/card.h) — الحقول المتعلقة بـ UHS2

**الغرض:** تمثيل الكارت الكامل. بيحمل `uhs2_config` كـ embedded struct.

```c
struct mmc_card {
    struct mmc_host      *host;         /* Host controller */
    struct sd_uhs2_config uhs2_config;  /* UHS-II capabilities & settings */
    /* ... باقي الحقول للـ SD/MMC/SDIO ... */
};
```

---

### 2. رسم علاقات الـ Structs

```
                    ┌─────────────────────────────────────┐
                    │          mmc_request                │
                    │  ┌─────────────────────────────┐   │
                    │  │  uhs2_command  (embedded)   │   │
                    │  │  header | arg | payload[]   │   │
                    │  │  uhs2_resp[]                │   │
                    │  └─────────────────────────────┘   │
                    │                                     │
                    │  *sbc ──► mmc_command               │
                    │  *cmd ──► mmc_command               │
                    │           └──► *uhs2_cmd ──► uhs2_command (pointer)
                    │  *data ──► mmc_data                 │
                    │  *stop ──► mmc_command               │
                    │  *host ──► mmc_host                 │
                    └─────────────────────────────────────┘

                    ┌──────────────────────────────────────┐
                    │            mmc_card                  │
                    │  *host ──────────────► mmc_host      │
                    │                                      │
                    │  uhs2_config (embedded):             │
                    │  ┌────────────────────────────────┐  │
                    │  │  sd_uhs2_config                │  │
                    │  │  node_id, n_lanes, n_fcu,      │  │
                    │  │  phy_*_rev, link_*_rev,        │  │
                    │  │  *_set fields (negotiated)     │  │
                    │  └────────────────────────────────┘  │
                    └──────────────────────────────────────┘
```

---

### 3. رسم Lifecycle — من الإنشاء للتدمير

#### Device Initialization Lifecycle

```
Power On
    │
    ▼
[UHS2_DEV_CMD_FULL_RESET @ 0x200]
    │  host sends CCMD to 0x200
    ▼
[UHS2_DEV_CMD_DEVICE_INIT @ 0x202]
    │  host sends broadcast CCMD (DID=SID=0)
    │  payload_len = UHS2_DEV_INIT_PAYLOAD_LEN (1)
    │  response_len = UHS2_DEV_INIT_RESP_LEN (6)
    │  repeat until UHS2_DEV_INIT_COMPLETE_FLAG (bit 11) set
    ▼
[UHS2_DEV_CMD_ENUMERATE @ 0x203]
    │  host assigns Node ID
    │  payload_len = UHS2_DEV_ENUM_PAYLOAD_LEN (1)
    │  response_len = UHS2_DEV_ENUM_RESP_LEN (8)
    │  → fills sd_uhs2_config.node_id
    ▼
[Read Device Registers via CCMD]
    │  Read GEN_CAPS    → n_lanes, dadr_len, app_type
    │  Read PHY_CAPS    → phy_rev, can_hibernate, n_lss_*
    │  Read LINK_CAPS   → link_rev, n_fcu, maxblk_len, dev_type
    │  → fills sd_uhs2_config capabilities fields
    ▼
[Negotiate & Write Settings via CCMD]
    │  Write GEN_SET    → n_lanes_set, GEN_SET_CFG_COMPLETE
    │  Write PHY_SET    → speed_range_set
    │  Write LINK_SET   → maxblk_len_set, max_retry_set
    │  → fills sd_uhs2_config *_set fields
    ▼
[Normal Operation — DCMD/CCMD transfers]
    │
    ▼
[Go Dormant: UHS2_DEV_CMD_GO_DORMANT_STATE @ 0x201]
    │  optional: UHS2_DEV_CMD_DORMANT_HIBER (bit 7) لو Hibernate
    ▼
[Resume or Full Reset]
```

---

#### uhs2_command Lifecycle (per transaction)

```
mmc_request allocated
    │
    ▼
uhs2_command embedded in mmc_request.uhs2_cmd
    │
    ▼
Build header:
    header  = UHS2_NATIVE_PACKET | UHS2_PACKET_TYPE_CCMD
    header |= (DID & UHS2_DEST_ID_MASK)
    header |= (TID << UHS2_TRANS_ID_POS)
    header |= (SID << UHS2_SRC_ID_POS)
    │
    ▼
Build arg:
    arg  = UHS2_NATIVE_CMD_WRITE or UHS2_NATIVE_CMD_READ
    arg |= (PLEN << UHS2_NATIVE_CMD_PLEN_POS)
    arg |= (IOADR MSB & UHS2_NATIVE_CCMD_MIOADR_MASK)
    arg |= (IOADR LSB << UHS2_NATIVE_CCMD_LIOADR_POS)
    │
    ▼
Set payload[] + payload_len + packet_len
    │
    ▼
Host controller transmits packet on UHS-II bus
    │
    ▼
Card sends RES packet back
    │
    ▼
uhs2_resp[] filled, uhs2_resp_len set
Check UHS2_RES_NACK_MASK → if set, read UHS2_RES_ECODE_MASK
    │
    ▼
mmc_request.done() callback called
    │
    ▼
mmc_request freed
```

---

### 4. رسم Call Flow

#### CCMD Write Flow (مثال: كتابة PHY_SET)

```
MMC Core (sd_uhs2.c)
        │
        │  uhs2_cmd->header = UHS2_NATIVE_PACKET | UHS2_PACKET_TYPE_CCMD
        │  uhs2_cmd->arg    = UHS2_NATIVE_CMD_WRITE | UHS2_NATIVE_CMD_PLEN_8B
        │                   | IOADR(UHS2_DEV_CONFIG_PHY_SET)
        │  uhs2_cmd->payload[0] = speed_range | (UHS2_DEV_CONFIG_PHY_SET_SPEED_B << 6)
        │  uhs2_cmd->payload_len = UHS2_CFG_WRITE_PAYLOAD_LEN (2)
        │
        ▼
mmc_wait_for_req(host, mrq)
        │
        ▼
host->ops->request(host, mrq)   ← host controller driver
        │
        ├── Serialize header(2B) + arg(2B) + payload(Nb) into TX buffer
        ├── Transmit on UHS-II lanes
        ├── Wait for RES packet
        ├── Fill uhs2_cmd->uhs2_resp[]
        └── Call mrq->done(mrq)
        │
        ▼
Check response:
    if uhs2_resp[N] & UHS2_RES_NACK_MASK:
        ecode = (uhs2_resp[N] >> UHS2_RES_ECODE_POS) & UHS2_RES_ECODE_MASK
        → UHS2_RES_ECODE_COND / ARG / GEN → return error
    else:
        → success, update sd_uhs2_config.*_set
```

#### MSG Interrupt Flow

```
Card detects error/event
        │
        ▼
Card sends MSG packet:
    header: TYP = UHS2_PACKET_TYPE_MSG
    body:   CTG = UHS2_MSG_CTG_INT (0x60)
            code = UHS2_MSG_CODE_STAT_RECOVER_ERR (0x1)
              or   UHS2_MSG_CODE_STAT_UNRECOVER_ERR (0x8)
        │
        ▼
Host interrupt fires
        │
        ▼
Host driver reads MSG body
        │
        ├── RECOVER_ERR → retry current transaction
        └── UNRECOVER_ERR → abort + full reset (UHS2_DEV_CMD_FULL_RESET)
```

#### Flow Control (FC) via MSG

```
Host wants to send data
        │
        ▼
Host sends MSG: CTG = UHS2_MSG_CTG_FCREQ (0x00)
        │
        ▼
Card checks its buffer (n_fcu credits)
        │
        ├── buffers available →
        │       Card sends MSG: CTG = UHS2_MSG_CTG_FCRDY (0x01)
        │       Host proceeds with DCMD
        │
        └── buffers full →
                Host waits for FCRDY
```

---

### 5. استراتيجية الـ Locking

الـ `sd_uhs2.h` header نفسه **مش بيعرّف أي locking primitives** — هو pure definitions (macros + constants). الـ locking بيحصل في طبقة أعلى:

| المستوى | الآلية | المسؤولية |
|---|---|---|
| **mmc_request** | `struct completion` (completion + cmd_completion) | Synchronous wait لإكمال الـ request |
| **mmc_host** | `host->lock` (spinlock في host.h) | حماية host state وqueueing |
| **mmc_card** | `host->claim_*` mechanism | منع تزامن الـ requests على نفس الكارت |
| **sd_uhs2_config** | محمي بـ host lock بشكل غير مباشر | بيتكتب مرة واحدة أثناء init ثم read-only |

**النمط العام:**
```c
/* الـ host lock بيحمي الـ request queue */
spin_lock_irqsave(&host->lock, flags);
    /* queue the mrq */
spin_unlock_irqrestore(&host->lock, flags);

/* الـ completion بتخلي الـ caller ينتظر */
wait_for_completion(&mrq->completion);
/* بعد هنا الـ uhs2_resp جاهز للقراءة بدون locking */
```

**الـ sd_uhs2_config بيتعبّى مرة واحدة أثناء:**
1. `DEVICE_INIT` → يحدد القدرات
2. `ENUMERATE` → يحدد الـ node_id
3. `CFG_WRITE` → يحدد الـ *_set fields

بعد انتهاء الـ initialization مفيش write على `sd_uhs2_config` — بيبقى read-only لباقي الـ session، فمحتاجش locking خاص.
## Phase 4: شرح الـ Functions

> **ملاحظة مهمة:** الملف `include/linux/mmc/sd_uhs2.h` هو **header-only macro file** — مش فيه functions أو APIs قابلة للاستدعاء. كل محتواه عبارة عن `#define` constants وbitmasks بتعرّف بروتوكول UHS-II على مستوى الـ LINK Layer والـ TRANS Layer وسجلات الجهاز. الـ functions الفعلية موجودة في `drivers/mmc/core/sd_uhs2.c`.
>
> الـ Phase ده هيشرح **كل الـ macros والـ constants** المعرّفة في الملف، مجمّعة بالـ categories، مع شرح دورها في البروتوكول.

---

## 0. Cheatsheet — الـ Macro Groups

| Group | Prefix | عدد الـ macros | الغرض |
|---|---|---|---|
| LINK Layer — Header | `UHS2_NATIVE_PACKET`, `UHS2_PACKET_TYPE_*`, `UHS2_DEST/SRC/TRANS_ID_*` | 12 | بناء الـ UHS2 packet header |
| MSG Packet | `UHS2_MSG_CTG_*`, `UHS2_MSG_CODE_*` | 9 | Flow control وError messages |
| TRANS Layer — CCMD Argument | `UHS2_NATIVE_CMD_*`, `UHS2_NATIVE_CCMD_*` | 9 | بناء الـ CCMD argument |
| TRANS Layer — DCMD Argument | `UHS2_DCMD_*` | 7 | بناء الـ DCMD argument |
| Response | `UHS2_RES_*` | 5 | تفسير الـ response packet |
| I/O Address Map | `UHS2_IOADR_*` | 8 | عناوين السجلات في الـ UHS2 I/O space |
| SD App Packets | `UHS2_SD_CMD_*` | 3 | تحديد SD commands فوق UHS2 |
| Device Registers | `UHS2_DEV_CONFIG_*`, `UHS2_DEV_INT_REG`, etc. | ~50 | خريطة كاملة لسجلات الكارت |
| Clock Limits | `UHS2_RCLK_MAX`, `UHS2_RCLK_MIN` | 2 | حدود الـ reference clock |
| Payload/Response Lengths | `UHS2_*_PAYLOAD_LEN`, `UHS2_*_RESP_LEN` | 9 | أطوال الـ packets لكل command نوع |

---

## 1. LINK Layer — Packet Header Macros

### الغرض
الـ UHS-II protocol بيبعت packets على الـ bus. كل packet عندها **2-byte header** بيتكوّن من حقول متعددة. الـ macros دي بتعرّف position وvalue لكل حقل في الـ header.

### بنية الـ Header (16 bits)

```
bit: [15:12]  [11]  [10:8]  [7]     [6:4]     [3:0]
      SID    Rsv    TID     NP     TYP(type)   DID
```

### الـ Macros تفصيلياً

```c
#define UHS2_NATIVE_PACKET_POS  7
#define UHS2_NATIVE_PACKET      (1 << UHS2_NATIVE_PACKET_POS)  /* = 0x80 */
```
- **`UHS2_NATIVE_PACKET`**: بيحدد إن الـ packet ده "native" — يعني صادر من الـ UHS2 layer نفسه (مش SD legacy command). الـ bit 7 في الـ header بيتسمى **NP (Native Packet)**.

```c
#define UHS2_PACKET_TYPE_POS    4
#define UHS2_PACKET_TYPE_CCMD   (0 << UHS2_PACKET_TYPE_POS)  /* 0x00 */
#define UHS2_PACKET_TYPE_DCMD   (1 << UHS2_PACKET_TYPE_POS)  /* 0x10 */
#define UHS2_PACKET_TYPE_RES    (2 << UHS2_PACKET_TYPE_POS)  /* 0x20 */
#define UHS2_PACKET_TYPE_DATA   (3 << UHS2_PACKET_TYPE_POS)  /* 0x30 */
#define UHS2_PACKET_TYPE_MSG    (7 << UHS2_PACKET_TYPE_POS)  /* 0x70 */
```
- الـ **TYP field** (bits [6:4]) بيحدد نوع الـ packet:
  - `CCMD` = Control Command: لقراءة/كتابة سجلات الكارت
  - `DCMD` = Data Command: لنقل البيانات (read/write blocks)
  - `RES`  = Response: رد الكارت على command
  - `DATA` = Data Payload: الـ data blocks الفعلية
  - `MSG`  = Message: flow control وerror notification

```c
#define UHS2_DEST_ID_MASK   0x0F
#define UHS2_DEST_ID        0x1
```
- **DID (Destination ID)** = bits [3:0]: الـ Node ID للكارت المقصود. الـ host دايماً بيبعت لـ DID=1 (أول كارت في الـ daisy chain).

```c
#define UHS2_SRC_ID_POS     12
#define UHS2_SRC_ID_MASK    0xF000
```
- **SID (Source ID)** = bits [15:12]: الـ Node ID للمرسل. الـ host دايماً SID=0.

```c
#define UHS2_TRANS_ID_POS   8
#define UHS2_TRANS_ID_MASK  0x0700
```
- **TID (Transaction ID)** = bits [10:8]: 3-bit counter بيعمل match بين الـ command والـ response المقابل له.

---

## 2. MSG Packet Macros

### الغرض
الـ **MSG packet** (type=0x70) هو آلية خاصة للـ flow control وإبلاغ الـ host بالأحداث (interrupts، errors). بيتبعت من الكارت للـ host.

```c
#define UHS2_MSG_CTG_POS    5
#define UHS2_MSG_CTG_LMSG   0x00   /* Link layer message */
#define UHS2_MSG_CTG_INT    0x60   /* Interrupt notification */
#define UHS2_MSG_CTG_AMSG   0x80   /* Application message */
```
- **CTG (Category)**: بيصنّف نوع الـ MSG. `LMSG` للـ link-level messages، `INT` للـ device interrupt للـ host، `AMSG` للـ application-level.

```c
#define UHS2_MSG_CTG_FCREQ  0x00   /* Flow Control Request */
#define UHS2_MSG_CTG_FCRDY  0x01   /* Flow Control Ready */
#define UHS2_MSG_CTG_STAT   0x02   /* Status message */
```
- الـ **Flow Control sub-types**: الكارت بيبعت `FCREQ` لما يحتاج إن الـ host يوقف الإرسال، والـ host بيرد بـ `FCRDY` لما يبقى جاهز يستقبل تاني.

```c
#define UHS2_MSG_CODE_POS                   8
#define UHS2_MSG_CODE_FC_UNRECOVER_ERR      0x8
#define UHS2_MSG_CODE_STAT_UNRECOVER_ERR    0x8
#define UHS2_MSG_CODE_STAT_RECOVER_ERR      0x1
```
- **Error codes**: `UNRECOVER_ERR` = خطأ مش ممكن الـ recovery منه (محتاج reset)، `RECOVER_ERR` = خطأ مؤقت ممكن إعادة المحاولة.

---

## 3. TRANS Layer — CCMD Argument Macros

### الغرض
الـ **CCMD (Control Command)** بيُستخدم لقراءة/كتابة سجلات الكارت في الـ UHS2 I/O space. الـ 2-byte argument بيحدد العنوان والـ payload size والاتجاه.

### بنية الـ CCMD Argument (16 bits)

```
bit: [15:8]      [7]    [6]   [5:4]     [3:0]
     LIOADR      R/W   Rsv   PLEN      MIOADR
```

```c
#define UHS2_NATIVE_CMD_RW_POS      7
#define UHS2_NATIVE_CMD_WRITE       (1 << UHS2_NATIVE_CMD_RW_POS)  /* 0x80 */
#define UHS2_NATIVE_CMD_READ        (0 << UHS2_NATIVE_CMD_RW_POS)  /* 0x00 */
```
- **R/W bit** (bit 7): يحدد اتجاه الـ control command — write أو read لسجل في الكارت.

```c
#define UHS2_NATIVE_CMD_PLEN_POS    4
#define UHS2_NATIVE_CMD_PLEN_4B     (1 << UHS2_NATIVE_CMD_PLEN_POS)   /* 0x10 */
#define UHS2_NATIVE_CMD_PLEN_8B     (2 << UHS2_NATIVE_CMD_PLEN_POS)   /* 0x20 */
#define UHS2_NATIVE_CMD_PLEN_16B    (3 << UHS2_NATIVE_CMD_PLEN_POS)   /* 0x30 */
```
- **PLEN (Payload Length)**: حجم الـ payload المرفق مع الـ CCMD — إما 4 أو 8 أو 16 bytes. القيمة 00b = لا يوجد payload.

```c
#define UHS2_NATIVE_CCMD_GET_MIOADR_MASK    0xF00
#define UHS2_NATIVE_CCMD_MIOADR_MASK        0x0F
#define UHS2_NATIVE_CCMD_LIOADR_POS         8
#define UHS2_NATIVE_CCMD_GET_LIOADR_MASK    0x0FF
```
- **IOADR (I/O Address)** = 12-bit address مقسوم على جزئين:
  - **MIOADR** (bits [3:0]): الـ MSB nibble من العنوان
  - **LIOADR** (bits [15:8]): الـ LSB byte من العنوان
  - الوحدة = 4 bytes، يعني IOADR=2 يعني byte offset = 8

```c
#define UHS2_CCMD_DEV_INIT_COMPLETE_FLAG    BIT(11)
```
- في الـ DEVICE_INIT command، الـ bit 11 في الـ argument بيشير إن الـ initialization اكتمل.

### أطوال الـ Packets

```c
#define UHS2_DEV_INIT_PAYLOAD_LEN       1   /* 4 bytes payload */
#define UHS2_DEV_INIT_RESP_LEN          6   /* 6 bytes response */
#define UHS2_DEV_ENUM_PAYLOAD_LEN       1
#define UHS2_DEV_ENUM_RESP_LEN          8
#define UHS2_CFG_WRITE_PAYLOAD_LEN      2   /* 8 bytes payload */
#define UHS2_CFG_WRITE_PHY_SET_RESP_LEN     4
#define UHS2_CFG_WRITE_GENERIC_SET_RESP_LEN 5
#define UHS2_GO_DORMANT_PAYLOAD_LEN     1
```
- القيم دي بتتحوّل لـ bytes بضربها في 4 (وحدة الـ payload). بتُستخدم في `drivers/mmc/core/sd_uhs2.c` لتحديد `packet_len` في `struct uhs2_command`.

---

## 4. TRANS Layer — DCMD Argument Macros

### الغرض
الـ **DCMD (Data Command)** بيُستخدم لنقل البيانات الفعلية (read/write blocks). الـ 2-byte argument بيحدد transfer mode والاتجاه.

### بنية الـ DCMD Argument

```
bit: [15:8]   [7]    [6]    [5]    [4]     [3]    [2:0]
     Rsv      R/W    DM     LM    TLUM    DAM    Rsv
```

```c
#define UHS2_DCMD_DM_POS        6
#define UHS2_DCMD_2L_HD_MODE    (1 << UHS2_DCMD_DM_POS)   /* 0x40 */
```
- **DM (Duplex Mode)**: لما يبقى 1 يعني الـ transfer في **2-lane Half-Duplex mode** بدل الـ Full-Duplex الافتراضي.

```c
#define UHS2_DCMD_LM_POS        5
#define UHS2_DCMD_LM_TLEN_EXIST (1 << UHS2_DCMD_LM_POS)   /* 0x20 */
```
- **LM (Length Mode)**: لما 1 يعني الـ DCMD packet فيها حقل TLEN (Transfer Length) صريح.

```c
#define UHS2_DCMD_TLUM_POS          4
#define UHS2_DCMD_TLUM_BYTE_MODE    (1 << UHS2_DCMD_TLUM_POS)  /* 0x10 */
```
- **TLUM (TLEN Unit Mode)**: لما 1 يعني وحدة قياس الـ TLEN هي **bytes**، لما 0 يعني **512-byte blocks**.

```c
#define UHS2_NATIVE_DCMD_DAM_POS    3
#define UHS2_NATIVE_DCMD_DAM_IO     (1 << UHS2_NATIVE_DCMD_DAM_POS)  /* 0x08 */
```
- **DAM (Data Access Mode)**: لما 1 يعني الـ data transfer هو I/O access، لما 0 يعني memory (block) access.

---

## 5. Response Macros

### الغرض
بتساعد الـ driver يفسّر الـ RES packet اللي بيرجعه الكارت بعد كل CCMD أو DCMD.

```c
#define UHS2_RES_NACK_POS   7
#define UHS2_RES_NACK_MASK  (0x1 << UHS2_RES_NACK_POS)   /* 0x80 */
```
- **NACK bit**: لما يكون 1 يعني الكارت رفض الـ command (Negative ACK). الـ driver لازم يتعامل مع الرفض ده.

```c
#define UHS2_RES_ECODE_POS      4
#define UHS2_RES_ECODE_MASK     0x7
#define UHS2_RES_ECODE_COND     1   /* Condition error */
#define UHS2_RES_ECODE_ARG      2   /* Argument error */
#define UHS2_RES_ECODE_GEN      3   /* General error */
```
- **ECODE (Error Code)**: لما NACK=1، الـ bits [6:4] بتحدد سبب الرفض:
  - `COND` = الـ command مش valid في الـ state الحالي
  - `ARG`  = الـ argument غلط أو out of range
  - `GEN`  = خطأ عام

---

## 6. I/O Address Map Macros

### الغرض
بتعرّف عناوين السجلات في الـ **UHS2 I/O space** اللي بيتم الوصول إليها عبر الـ CCMDs. الوحدة = 4 bytes.

```c
#define UHS2_IOADR_GENERIC_CAPS     0x00  /* Generic Capabilities (offset 0) */
#define UHS2_IOADR_PHY_CAPS         0x02  /* PHY Capabilities (offset 8) */
#define UHS2_IOADR_LINK_CAPS        0x04  /* Link-TRAN Capabilities (offset 16) */
#define UHS2_IOADR_RSV_CAPS         0x06  /* Reserved */
#define UHS2_IOADR_GENERIC_SETTINGS 0x08  /* Generic Settings (offset 32) */
#define UHS2_IOADR_PHY_SETTINGS     0x0A  /* PHY Settings (offset 40) */
#define UHS2_IOADR_LINK_SETTINGS    0x0C  /* Link-TRAN Settings (offset 48) */
#define UHS2_IOADR_PRESET           0x40  /* Preset register (offset 256) */
```

### خريطة الـ I/O Space

```
IOADR  Byte Offset  Register
0x00   0            Generic Capabilities (GEN_CAPS)
0x02   8            PHY Capabilities
0x04   16           Link-TRAN Capabilities
0x06   24           Reserved
0x08   32           Generic Settings
0x0A   40           PHY Settings
0x0C   48           Link-TRAN Settings
0x40   256          Preset Values
```

---

## 7. Device Config Registers — الـ Macro Groups

### 7a. Generic Capabilities (`GEN_CAPS` @ 0x000)

```c
#define UHS2_DEV_CONFIG_GEN_CAPS        (UHS2_DEV_CONFIG_REG + 0x000)
#define UHS2_DEV_CONFIG_N_LANES_POS     8
#define UHS2_DEV_CONFIG_N_LANES_MASK    0x3F
```
- **N_LANES**: عدد الـ data lanes المدعومة. 6 bits بتحدد التشكيلات الممكنة.

```c
#define UHS2_DEV_CONFIG_2L_HD_FD    0x1   /* 2-Lane Half/Full Duplex */
#define UHS2_DEV_CONFIG_2D1U_FD     0x2   /* 2D + 1U Full Duplex */
#define UHS2_DEV_CONFIG_1D2U_FD     0x4   /* 1D + 2U Full Duplex */
#define UHS2_DEV_CONFIG_2D2U_FD     0x8   /* 2D + 2U Full Duplex */
```
- الـ lane configurations: D = downstream (host→card)، U = upstream (card→host). الأرقام دي بتعمل bitmask في الـ N_LANES field.

```c
#define UHS2_DEV_CONFIG_DADR_POS    14
#define UHS2_DEV_CONFIG_DADR_MASK   0x1
```
- **DADR (Device Address)**: 1 bit بيحدد الـ node address للكارت في الـ daisy chain.

```c
#define UHS2_DEV_CONFIG_APP_POS     16
#define UHS2_DEV_CONFIG_APP_MASK    0xFF
#define UHS2_DEV_CONFIG_APP_SD_MEM  0x1
```
- **APP**: بيحدد نوع الـ application layer. `APP_SD_MEM=0x1` يعني SD Memory card.

### 7b. Generic Settings (`GEN_SET` @ 0x008)

```c
#define UHS2_DEV_CONFIG_GEN_SET             (UHS2_DEV_CONFIG_REG + 0x008)
#define UHS2_DEV_CONFIG_GEN_SET_N_LANES_POS 8
#define UHS2_DEV_CONFIG_GEN_SET_2L_FD_HD    0x0
#define UHS2_DEV_CONFIG_GEN_SET_2D1U_FD     0x2
#define UHS2_DEV_CONFIG_GEN_SET_1D2U_FD     0x3
#define UHS2_DEV_CONFIG_GEN_SET_2D2U_FD     0x4
#define UHS2_DEV_CONFIG_GEN_SET_CFG_COMPLETE BIT(31)
```
- **GEN_SET**: الـ host بيكتب فيه الـ lane configuration المختارة.
- **CFG_COMPLETE (bit 31)**: الـ host بيضرب الـ bit ده لما يخلص كتابة كل الـ settings، وبيديها للكارت إشارة إنه يبدأ يعمل بالـ configuration الجديدة.

### 7c. PHY Capabilities (`PHY_CAPS` @ 0x002)

```c
#define UHS2_DEV_CONFIG_PHY_CAPS        (UHS2_DEV_CONFIG_REG + 0x002)
#define UHS2_DEV_CONFIG_PHY_MINOR_MASK  0xF
#define UHS2_DEV_CONFIG_PHY_MAJOR_POS   4
#define UHS2_DEV_CONFIG_PHY_MAJOR_MASK  0x3
#define UHS2_DEV_CONFIG_CAN_HIBER_POS   15
#define UHS2_DEV_CONFIG_CAN_HIBER_MASK  0x1
```
- **PHY revision**: major.minor version للـ PHY spec المدعومة.
- **CAN_HIBER**: بيشير إن الكارت يقدر يدخل **Hibernate mode** (نوع من الـ low-power states في UHS2).

```c
#define UHS2_DEV_CONFIG_PHY_CAPS1       (UHS2_DEV_CONFIG_REG + 0x003)
#define UHS2_DEV_CONFIG_N_LSS_SYN_MASK  0xF
#define UHS2_DEV_CONFIG_N_LSS_DIR_POS   4
#define UHS2_DEV_CONFIG_N_LSS_DIR_MASK  0xF
```
- **N_LSS_SYN / N_LSS_DIR**: عدد الـ **Line Synchronization Sequences** المطلوبة. الـ host يلتزم بها لتوليد الـ clock correctly.

### 7d. PHY Settings (`PHY_SET` @ 0x00A)

```c
#define UHS2_DEV_CONFIG_PHY_SET             (UHS2_DEV_CONFIG_REG + 0x00A)
#define UHS2_DEV_CONFIG_PHY_SET_SPEED_POS   6
#define UHS2_DEV_CONFIG_PHY_SET_SPEED_A     0x0   /* Speed Range A: up to 1.44 Gbps */
#define UHS2_DEV_CONFIG_PHY_SET_SPEED_B     0x1   /* Speed Range B: up to 2.88 Gbps */
```
- الـ host بيكتب هنا الـ **speed range** المختار. Speed A = الحد الأقصى 1.44 Gbps، Speed B = 2.88 Gbps.

### 7e. Link-TRAN Capabilities (`LINK_TRAN_CAPS` @ 0x004)

```c
#define UHS2_DEV_CONFIG_LINK_TRAN_CAPS      (UHS2_DEV_CONFIG_REG + 0x004)
#define UHS2_DEV_CONFIG_LT_MINOR_MASK       0xF
#define UHS2_DEV_CONFIG_LT_MAJOR_POS        4
#define UHS2_DEV_CONFIG_LT_MAJOR_MASK       0x3
#define UHS2_DEV_CONFIG_N_FCU_POS           8
#define UHS2_DEV_CONFIG_N_FCU_MASK          0xFF
#define UHS2_DEV_CONFIG_DEV_TYPE_POS        16
#define UHS2_DEV_CONFIG_DEV_TYPE_MASK       0x7
#define UHS2_DEV_CONFIG_MAX_BLK_LEN_POS     20
#define UHS2_DEV_CONFIG_MAX_BLK_LEN_MASK    0xFFF
```
- **N_FCU**: عدد الـ **Flow Control Units** — كام packet يقدر الكارت يستقبل قبل ما يعمل flow control.
- **DEV_TYPE**: نوع الجهاز (memory، I/O، etc.).
- **MAX_BLK_LEN**: أقصى حجم block يقدر الكارت يتعامل معاه.

```c
#define UHS2_DEV_CONFIG_LINK_TRAN_CAPS1     (UHS2_DEV_CONFIG_REG + 0x005)
#define UHS2_DEV_CONFIG_N_DATA_GAP_MASK     0xFF
```
- **N_DATA_GAP**: عدد الـ clock cycles الـ gap المطلوب بين الـ data packets.

### 7f. Link-TRAN Settings (`LINK_TRAN_SET` @ 0x00C)

```c
#define UHS2_DEV_CONFIG_LINK_TRAN_SET           (UHS2_DEV_CONFIG_REG + 0x00C)
#define UHS2_DEV_CONFIG_LT_SET_MAX_BLK_LEN      0x200   /* 512 bytes */
#define UHS2_DEV_CONFIG_LT_SET_MAX_RETRY_POS    16
```
- الـ host بيكتب هنا الـ block length المختارة (عادةً 512 bytes) وعدد الـ retries الأقصى.

---

## 8. Device Command/Status/Interrupt Registers

```c
#define UHS2_DEV_CONFIG_REG         0x000   /* Config registers base */
#define UHS2_DEV_INT_REG            0x100   /* Interrupt registers base */
#define UHS2_DEV_STATUS_REG         0x180   /* Status registers base */
#define UHS2_DEV_CMD_REG            0x200   /* Command registers base */
```
- الخريطة العامة للـ UHS2 device register space مقسومة لـ 4 regions.

### Device Commands

```c
#define UHS2_DEV_CMD_FULL_RESET         (UHS2_DEV_CMD_REG + 0x000)
#define UHS2_DEV_CMD_GO_DORMANT_STATE   (UHS2_DEV_CMD_REG + 0x001)
#define UHS2_DEV_CMD_DORMANT_HIBER      BIT(7)
#define UHS2_DEV_CMD_DEVICE_INIT        (UHS2_DEV_CMD_REG + 0x002)
#define UHS2_DEV_INIT_COMPLETE_FLAG     BIT(11)
#define UHS2_DEV_CMD_ENUMERATE          (UHS2_DEV_CMD_REG + 0x003)
#define UHS2_DEV_CMD_TRANS_ABORT        (UHS2_DEV_CMD_REG + 0x004)
```

| Macro | Address | الوظيفة |
|---|---|---|
| `UHS2_DEV_CMD_FULL_RESET` | 0x200 | إعادة تعيين كاملة للكارت |
| `UHS2_DEV_CMD_GO_DORMANT_STATE` | 0x201 | الانتقال لـ Dormant (low-power) mode |
| `UHS2_DEV_CMD_DORMANT_HIBER` | BIT(7) | flag في DORMANT command لتفعيل Hibernate بدل Dormant |
| `UHS2_DEV_CMD_DEVICE_INIT` | 0x202 | بدء initialization sequence |
| `UHS2_DEV_INIT_COMPLETE_FLAG` | BIT(11) | flag في DEVICE_INIT command |
| `UHS2_DEV_CMD_ENUMERATE` | 0x203 | تعيين الـ Node ID للكارت |
| `UHS2_DEV_CMD_TRANS_ABORT` | 0x204 | إلغاء الـ transaction الحالية |

---

## 9. SD Application Packet Macros

```c
#define UHS2_SD_CMD_INDEX_POS   8
#define UHS2_SD_CMD_APP_POS     14
#define UHS2_SD_CMD_APP         (1 << UHS2_SD_CMD_APP_POS)   /* 0x4000 */
```
- لما الـ host بيبعت SD command (مش native UHS2 command) عبر UHS2:
  - الـ **CMD index** بيتحط في bits [13:8] من الـ DCMD argument.
  - **APP bit** (bit 14): بيميّز الـ ACMD (Application-Specific Command) عن الـ standard SD CMD.

---

## 10. Preset Register & Clock Limits

```c
#define UHS2_DEV_CONFIG_PRESET      (UHS2_DEV_CONFIG_REG + 0x040)   /* @ 0x040 */
```
- الـ **Preset register** بيحتوي على قيم configuration جاهزة للـ speed modes المختلفة.

```c
#define UHS2_RCLK_MAX   52000000   /* 52 MHz */
#define UHS2_RCLK_MIN   26000000   /* 26 MHz */
```
- **Reference Clock (RCLK)** حدوده 26 MHz ~ 52 MHz. الـ host بيستخدم القيم دي للتحقق من صحة الـ clock source قبل initializing the UHS2 interface.

---

## 11. العلاقة بين الـ Macros والـ Structs في `core.h`

الـ macros في `sd_uhs2.h` بتُستخدم لبناء الـ `struct uhs2_command`:

```c
/* من include/linux/mmc/core.h */
struct uhs2_command {
    u16     header;          /* LINK layer: DID + TYP + NP + TID + SID */
    u16     arg;             /* TRANS layer: IOADR + R/W + PLEN (CCMD) */
    __be32  payload[2];      /* config data لأقصى 8 bytes */
    u8      payload_len;     /* UHS2_*_PAYLOAD_LEN */
    u8      packet_len;      /* الحجم الكلي للـ packet */
    u8      tmode_half_duplex; /* UHS2_DCMD_2L_HD_MODE */
    u8      uhs2_resp[20];   /* الـ response buffer */
    u8      uhs2_resp_len;   /* UHS2_*_RESP_LEN */
};
```

### مثال: بناء DEVICE_INIT CCMD

```c
/* Pseudocode from sd_uhs2.c — building DEVICE_INIT command */

/* 1. Build header: native packet, CCMD type, broadcast (DID=SID=0) */
cmd->uhs2_cmd->header = UHS2_NATIVE_PACKET | UHS2_PACKET_TYPE_CCMD;

/* 2. Build argument: IOADR of DEVICE_INIT reg, write direction, 4-byte payload */
cmd->uhs2_cmd->arg = UHS2_DEV_CMD_DEVICE_INIT    /* IOADR */
                   | UHS2_NATIVE_CMD_WRITE         /* R/W=1 */
                   | UHS2_NATIVE_CMD_PLEN_4B;      /* payload=4 bytes */

/* 3. Set last init flag in payload if this is the last card */
cmd->uhs2_cmd->payload[0] = cpu_to_be32(UHS2_DEV_INIT_COMPLETE_FLAG);

/* 4. Set lengths */
cmd->uhs2_cmd->payload_len = UHS2_DEV_INIT_PAYLOAD_LEN;  /* =1 (×4 = 4 bytes) */
cmd->uhs2_cmd->uhs2_resp_len = UHS2_DEV_INIT_RESP_LEN;   /* =6 bytes expected */
```

---

## 12. الـ `uhs2_control` Callback (من `host.h`)

الـ callback الوحيد المرتبط مباشرة بالـ UHS2 operations في الـ host driver:

```c
int (*uhs2_control)(struct mmc_host *host, enum sd_uhs2_operation op);
```

- **الوظيفة**: نقطة الدخول الوحيدة للـ MMC core لتنفيذ أي operation خاصة بـ UHS2 على مستوى الـ host controller.
- **`host`**: الـ host controller المراد التحكم فيه.
- **`op`**: العملية المطلوبة من enum `sd_uhs2_operation`:

| Value | المعنى |
|---|---|
| `UHS2_PHY_INIT` | تهيئة الـ PHY layer |
| `UHS2_SET_CONFIG` | كتابة الـ configuration للكارت |
| `UHS2_ENABLE_INT` | تفعيل الـ interrupts |
| `UHS2_DISABLE_INT` | تعطيل الـ interrupts |
| `UHS2_ENABLE_CLK` | تفعيل الـ clock |
| `UHS2_DISABLE_CLK` | تعطيل الـ clock |
| `UHS2_CHECK_DORMANT` | التحقق من دخول الـ dormant mode |
| `UHS2_SET_IOS` | تحديث الـ I/O settings |

- **Return**: `0` للنجاح، أو `errno` سالب للخطأ.
- **Caller**: `drivers/mmc/core/sd_uhs2.c` — الـ core layer بيستدعيه في كل مراحل الـ initialization وتبديل الـ speed mode.
- **Key detail**: الـ callback ده **mandatory** لأي host يدعم `MMC_CAP2_SD_UHS2`، وبيتنفّذ في context مش atomic (ممكن يعمل sleep).
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs Entries

| Entry | Path | الوصف |
|-------|------|--------|
| `mmc0` host info | `/sys/kernel/debug/mmc0/` | معلومات الـ host controller |
| `ios` | `/sys/kernel/debug/mmc0/ios` | حالة الـ clock، bus width، timing |
| `state` | `/sys/kernel/debug/mmc0/state` | حالة الـ host state machine |

```bash
# اقرأ حالة الـ UHS-II interface
cat /sys/kernel/debug/mmc0/ios

# مثال على output:
# clock:          52000000 Hz
# actual clock:   52000000 Hz
# timing spec:    uhs2 (14)
# bus width:      2 (4 bits)
```

الـ `timing spec: uhs2 (14)` يأكد إن الـ host شغال بـ UHS-II mode. لو شفت `sd` أو `mmc` بدل كده، معناه الـ link negotiation فشل.

---

#### 2. sysfs Entries

| Entry | Path | الوصف |
|-------|------|--------|
| `type` | `/sys/bus/mmc/devices/mmc0:*/type` | نوع الكارت (SD, MMC) |
| `name` | `/sys/bus/mmc/devices/mmc0:*/name` | اسم الكارت |
| `date` | `/sys/bus/mmc/devices/mmc0:*/date` | تاريخ التصنيع |
| `hwrev` | `/sys/bus/mmc/devices/mmc0:*/hwrev` | Hardware revision |
| `preferred_erase_size` | `/sys/bus/mmc/devices/mmc0:*/preferred_erase_size` | حجم الـ erase المفضل |

```bash
# اقرأ معلومات الكارت
cat /sys/bus/mmc/devices/mmc0\:*/type
cat /sys/bus/mmc/devices/mmc0\:*/name

# تحقق من الـ UHS-II capability عبر الـ CID/CSD
ls /sys/bus/mmc/devices/mmc0\:*/
```

---

#### 3. ftrace — Tracepoints/Events

```bash
# فعّل الـ MMC tracepoints
cd /sys/kernel/debug/tracing
echo 1 > tracing_on

# فعّل كل أحداث الـ MMC
echo 1 > events/mmc/enable

# أو بشكل انتقائي — أهم الـ events لـ UHS-II:
echo 1 > events/mmc/mmc_request_start/enable
echo 1 > events/mmc/mmc_request_done/enable
echo 1 > events/mmc/mmc_cmd_rw_start/enable
echo 1 > events/mmc/mmc_cmd_rw_end/enable

# اقرأ الـ trace
cat trace

# مثال على output:
#   kworker/0:1-45  [000] ....  123.456: mmc_request_start: mmc0: start struct mmc_request
#   kworker/0:1-45  [000] ....  123.457: mmc_cmd_rw_start: mmc0: CMD5 [00000000] flags CLK_RAMP, CMD_AC
```

تتبع `CMD5` (IO_SEND_OP_COND) في الـ trace هو أول علامة على بداية UHS-II negotiation. غيابه يعني إن الـ driver مش شايف الكارت كـ UHS-II.

---

#### 4. printk / Dynamic Debug

```bash
# فعّل dynamic debug للـ MMC subsystem بالكامل
echo 'module mmc_core +p' > /sys/kernel/debug/dynamic_debug/control
echo 'module sdhci +p'    > /sys/kernel/debug/dynamic_debug/control
echo 'module sdhci_pci +p' > /sys/kernel/debug/dynamic_debug/control

# أو فعّل debug لملف محدد (مثلاً الـ UHS2 core)
echo 'file sd_uhs2.c +p'  > /sys/kernel/debug/dynamic_debug/control

# تحقق من الـ messages في الـ kernel log
dmesg -w | grep -i uhs2
dmesg -w | grep -i mmc

# مثال على messages مهمة:
# [  5.123] mmc0: UHS-II interface detected
# [  5.124] mmc0: UHS-II CCMD DEVICE_INIT sent
# [  5.125] mmc0: UHS-II card initialized, N_LANES=0x1
```

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الغرض |
|---------------|--------|
| `CONFIG_MMC_DEBUG` | يفعّل verbose logging في الـ MMC core |
| `CONFIG_MMC_SDHCI` | SDHCI host controller support (أساسي) |
| `CONFIG_MMC_SDHCI_PCI` | PCI-based SDHCI (لو الـ host على PCI) |
| `CONFIG_MMC_TEST` | وحدة test للـ MMC |
| `CONFIG_DYNAMIC_DEBUG` | لازم يكون مفعّل لـ dynamic debug |
| `CONFIG_FTRACE` | لتفعيل الـ tracepoints |
| `CONFIG_MMC_BLOCK_MINORS` | عدد الـ minor devices |
| `CONFIG_FAIL_MMC_REQUEST` | fault injection للـ MMC requests (للتست) |

```bash
# تحقق من التفعيل الحالي
zcat /proc/config.gz | grep -E 'CONFIG_MMC|CONFIG_SDHCI'
```

---

#### 6. Common Error Messages

| رسالة الـ Kernel | المعنى | الحل |
|------------------|--------|-------|
| `mmc0: error -110 whilst initialising SD card` | Timeout أثناء الـ init | تحقق من الـ power supply والـ clock |
| `mmc0: UHS-II DEVICE_INIT failed` | فشل CCMD DEVICE_INIT | تحقق من RCLK (26–52 MHz) |
| `mmc0: NACK received, ECODE=2` | ARG error في الـ CCMD | الـ IOADR أو الـ payload غلط (`UHS2_RES_ECODE_ARG`) |
| `mmc0: NACK received, ECODE=1` | Condition error | الـ card مش في الـ state الصح (`UHS2_RES_ECODE_COND`) |
| `mmc0: NACK received, ECODE=3` | General error | خطأ عام في الكارت (`UHS2_RES_ECODE_GEN`) |
| `mmc0: Unrecoverable error in MSG packet` | `UHS2_MSG_CODE_STAT_UNRECOVER_ERR` أو `UHS2_MSG_CODE_FC_UNRECOVER_ERR` | Reset كامل للكارت، احتمال عطل hardware |
| `mmc0: Recoverable error detected` | `UHS2_MSG_CODE_STAT_RECOVER_ERR` | إعادة محاولة الـ transaction |
| `mmc0: card removed during transfer` | Physical disconnect أو power issue | تحقق من الـ slot والكارت |
| `mmc0: timeout waiting for hardware interrupt` | الـ host controller مش بيرد | تحقق من الـ IRQ والـ clock |

---

#### 7. Strategic Points لـ `dump_stack()` / `WARN_ON()`

```c
/* 1. بعد بناء الـ UHS2 header — تحقق من الـ TYP و DID */
WARN_ON((header & UHS2_DEST_ID_MASK) > 0xF);
WARN_ON(((header >> UHS2_PACKET_TYPE_POS) & 0x7) > UHS2_PACKET_TYPE_MSG);

/* 2. قبل إرسال CCMD DEVICE_INIT */
WARN_ON(!(flags & UHS2_NATIVE_PACKET));

/* 3. عند تلقي NACK في الـ response */
if (resp & UHS2_RES_NACK_MASK) {
    pr_err("mmc%d: NACK! ECODE=%lu\n", host->index,
           (resp >> UHS2_RES_ECODE_POS) & UHS2_RES_ECODE_MASK);
    dump_stack();
}

/* 4. عند تلقي MSG من نوع unrecoverable error */
if (msg_code == UHS2_MSG_CODE_FC_UNRECOVER_ERR ||
    msg_code == UHS2_MSG_CODE_STAT_UNRECOVER_ERR) {
    WARN(1, "mmc%d: unrecoverable UHS-II error\n", host->index);
}

/* 5. عند الانتقال لـ GO_DORMANT — تحقق من الـ state */
WARN_ON(!mmc_card_uhs2(card));
```

---

### Hardware Level

#### 1. التحقق من Hardware State مقابل Kernel State

**الـ RCLK range (26 MHz – 52 MHz):**

```bash
# تحقق من الـ clock المستخدم
cat /sys/kernel/debug/mmc0/ios | grep clock

# المتوقع لـ UHS-II:
# clock: 52000000 Hz  ← UHS2_RCLK_MAX
# أو
# clock: 26000000 Hz  ← UHS2_RCLK_MIN
```

**Lane configuration:**

```
UHS2_DEV_CONFIG_N_LANES_MASK = 0x3F  (bits [13:8] من GEN_CAPS)
القيم المتوقعة:
  0x1 → 2L-HD (2 lanes, half-duplex)   ← UHS2_DEV_CONFIG_2L_HD_FD
  0x2 → 2D1U-FD (2 down, 1 up)
  0x4 → 1D2U-FD (1 down, 2 up)
  0x8 → 2D2U-FD (2 down, 2 up)
```

```bash
# مقارنة القيمة المقروءة من الكارت بما تم تطبيقه
dmesg | grep -i "n_lanes\|lanes\|uhs2"
```

**PHY Speed:**

```
UHS2_DEV_CONFIG_PHY_SET_SPEED_A = 0x0  ← Range A (up to ~1.8 Gbps)
UHS2_DEV_CONFIG_PHY_SET_SPEED_B = 0x1  ← Range B (up to ~3.6 Gbps)
```

---

#### 2. Register Dump Techniques

```bash
# استخدم devmem2 لقراءة SDHCI registers مباشرة (مثال لعنوان افتراضي)
# أولاً: اكتشف عنوان الـ SDHCI base من device tree أو lspci
lspci -v | grep -A 10 "SD Host"
cat /proc/iomem | grep -i sdhci

# بعد معرفة الـ base address (مثلاً 0xFE340000):
# UHS-II Vendor Specific Registers عادةً بتبدأ من offset 0x300

# قراءة UHS-II General Caps (IOADR=0x00)
devmem2 0xFE340300 w   # قراءة 32-bit

# قراءة PHY Caps (IOADR=0x02)
devmem2 0xFE340308 w

# قراءة LINK_TRAN Caps (IOADR=0x04)
devmem2 0xFE340310 w
```

```
UHS-II Device Register Map:
  0x000 → GEN_CAPS       (N_LANES, APP type)
  0x002 → PHY_CAPS       (PHY version, CAN_HIBER)
  0x004 → LINK_TRAN_CAPS (N_FCU, MAX_BLK_LEN)
  0x008 → GEN_SETTINGS
  0x00A → PHY_SETTINGS   (SPEED_A or SPEED_B)
  0x00C → LINK_TRAN_SET
  0x040 → PRESET
  0x100 → INT_REG
  0x180 → STATUS_REG
  0x200 → CMD_REG (FULL_RESET, GO_DORMANT, DEVICE_INIT, ENUMERATE)
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

**الـ UHS-II physical layer بيستخدم 2 lanes (D+ / D-):**

```
UHS-II Signal Lanes:
  ┌──────────────┐         ┌──────────────┐
  │   Host       │         │   Card       │
  │              │ D0+/D0- │              │
  │    Lane 0   ─┼────────→┼─ Lane 0      │
  │    Lane 1   ─┼────────→┼─ Lane 1      │ (HD mode)
  │              │←────────┼─             │ (return in FD)
  └──────────────┘         └──────────────┘
```

**ما تتبعه بالـ logic analyzer:**

| Signal | الوصف | القيمة المتوقعة |
|--------|--------|-----------------|
| `CLK` | Reference clock | 26 MHz أو 52 MHz |
| `D0+/D0-` | Lane 0 differential | LVDS levels (350mV swing) |
| `D1+/D1-` | Lane 1 differential | نفس Lane 0 في HD mode |

**نقاط المزامنة للـ capture:**

```
1. Capture عند DEVICE_INIT:
   - ابدأ الـ trigger على أول edge في الـ CLK بعد power-up
   - ابحث عن CCMD packet: النمط 0x00 في الـ DID و TYP=000b

2. Capture عند GO_DORMANT:
   - ابحث عن CMD byte = UHS2_DEV_CMD_GO_DORMANT_STATE (0x201)
   - لازم تشوف الـ DORMANT_HIBER bit لو بتدخل hibernate

3. Decode الـ packet:
   Byte 0-1: Header  [SID|TID|RSV|NP|TYP|DID]
   Byte 2-3: Argument [RW|PLEN|RSV|MIOADR|LIOADR]
   Byte 4+:  Payload
```

---

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة الـ Hardware | النمط في الـ Kernel Log | التشخيص |
|----------------------|------------------------|----------|
| ضعف الـ power supply | `mmc0: error -110 ... initialising` | قِس الـ VDD على الكارت، المفروض 1.8V |
| Clock jitter عالي | `mmc0: timeout ... interrupt` | قِس الـ RCLK بالأوسيلوسكوب |
| Lane polarity معكوسة | `mmc0: NACK received` مباشرة | اعكس D+/D- في الـ PCB |
| Impedance mismatch | CRC errors متكررة | تحقق من الـ trace length والـ termination |
| كارت مش UHS-II | `mmc0: falling back to legacy` | تحقق من الـ card spec |
| Hibernate مش شغال | `mmc0: GO_DORMANT failed` | تحقق من `CAN_HIBER` bit في PHY_CAPS |
| N_FCU صفر | Stalled transfers | تحقق من `UHS2_DEV_CONFIG_N_FCU_MASK` |

---

### Practical Commands

#### 1. Shell Commands جاهزة للنسخ

```bash
# ===== تشغيل الـ debugging الكامل لـ UHS-II =====

# 1. فعّل dynamic debug
echo 'module mmc_core +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 'module sdhci +pflmt'    > /sys/kernel/debug/dynamic_debug/control

# 2. فعّل ftrace events
echo 0 > /sys/kernel/debug/tracing/trace
echo 1 > /sys/kernel/debug/tracing/events/mmc/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on

# 3. إعادة تشغيل الكارت (eject + insert أو):
echo 1 > /sys/bus/mmc/devices/mmc0\:*/remove 2>/dev/null
echo 1 > /sys/bus/mmc/rescan

# 4. اقرأ الـ trace
cat /sys/kernel/debug/tracing/trace | grep mmc0

# 5. اقرأ الـ dmesg
dmesg | grep -E 'mmc0|uhs2|UHS' | tail -50

# 6. اقرأ حالة الـ ios
cat /sys/kernel/debug/mmc0/ios

# 7. وقف الـ tracing
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo 0 > /sys/kernel/debug/tracing/events/mmc/enable
```

```bash
# ===== تشخيص سريع لمشكلة NACK =====

dmesg | grep -i 'nack\|ecode\|uhs2\|ccmd\|dcmd' | tail -20

# ===== تحقق من الـ UHS-II lane config =====

dmesg | grep -i 'lane\|n_lanes\|2l\|fd\|hd' | tail -20

# ===== تحقق من الـ PHY speed negotiation =====

dmesg | grep -i 'speed\|phy\|range.a\|range.b' | tail -20

# ===== فحص الـ card capabilities عبر sysfs =====

for f in /sys/bus/mmc/devices/mmc0\:*/*; do
  echo "=== $f ==="; cat "$f" 2>/dev/null; echo
done
```

---

#### 2. Example Output وكيفية تفسيره

**مثال 1: `cat /sys/kernel/debug/mmc0/ios` بعد UHS-II init ناجح**

```
clock:		52000000 Hz         ← UHS2_RCLK_MAX — شغال بأقصى سرعة
actual clock:	52000000 Hz
vdd:		21 (3.3 ~ 3.4 V)
bus mode:	2 (push-pull)
chip select:	0 (don't care)
power mode:	2 (on)
bus width:	2 (4 bits)          ← قد يظهر هكذا، UHS-II بيستخدم lanes مش width بالمعنى التقليدي
timing spec:	14 (uhs2)           ← 14 = MMC_TIMING_UHS2 ← أهم مؤشر
signal voltage:	1 (1.80 V)         ← UHS-II بيشتغل على 1.8V
driver type:	0 (driver type B)
```

التفسير: كل القيم صح. `timing spec: 14` هو المؤشر الأساسي على نجاح UHS-II init.

---

**مثال 2: ftrace output لـ DEVICE_INIT**

```
# tracer: nop
#
kworker/u4:2-89 [001] d... 45.123001: mmc_request_start: mmc0: start struct mmc_request[0xffff888...]
kworker/u4:2-89 [001] d... 45.123050: mmc_cmd_rw_start: mmc0: CMD0 [00000000] flags RSP_NONE, CMD_BC
kworker/u4:2-89 [001] d... 45.123100: mmc_cmd_rw_end:   mmc0: CMD0 [00000000] flags RSP_NONE, CMD_BC; err=0
kworker/u4:2-89 [001] d... 45.124000: mmc_cmd_rw_start: mmc0: CMD8 [000001AA] flags RSP_R7, CMD_BCR
kworker/u4:2-89 [001] d... 45.124050: mmc_cmd_rw_end:   mmc0: CMD8 [000001AA] flags RSP_R7, CMD_BCR; err=0
```

التفسير:
- `CMD0` = GO_IDLE_STATE — reset
- `CMD8` = SEND_IF_COND — تحقق من الـ voltage range
- بعد كده المفروض تشوف الـ UHS-II specific commands (CCMD DEVICE_INIT)
- لو وقف عند `CMD8` وطلع `err=-110` → الكارت مش بيرد على الـ interface condition → مش UHS-II

---

**مثال 3: dmesg عند NACK مع ECODE**

```
[   45.200] mmc0: UHS-II CCMD sent to IOADR=0x00 (GEN_CAPS), RW=READ
[   45.201] mmc0: NACK received in response! ECODE=0x2
[   45.201] mmc0: Argument error — check IOADR alignment (must be 4-byte aligned)
```

التفسير:
- `ECODE=0x2` = `UHS2_RES_ECODE_ARG` — الـ argument (الـ IOADR أو الـ PLEN) غلط
- تحقق إن الـ IOADR مضروب في 4 (unit = 4 bytes)
- تحقق من الـ `UHS2_NATIVE_CMD_PLEN_*` اللي اتبعت

---

**مثال 4: dmesg عند Unrecoverable Error**

```
[   50.100] mmc0: UHS-II MSG received: CTG=INT, CODE=0x8
[   50.100] mmc0: Unrecoverable error — initiating full reset
[   50.101] mmc0: sending FULL_RESET to UHS2_DEV_CMD_FULL_RESET (0x200)
[   50.200] mmc0: UHS-II re-initialization after full reset
```

التفسير:
- `CODE=0x8` = `UHS2_MSG_CODE_STAT_UNRECOVER_ERR` أو `UHS2_MSG_CODE_FC_UNRECOVER_ERR`
- الكارت أرسل MSG packet بـ `UHS2_MSG_CTG_INT` (interrupt message)
- الـ driver يعمل full reset عبر `UHS2_DEV_CMD_FULL_RESET`
- لو بيتكرر كتير → مشكلة hardware في الـ signal integrity
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: فشل initialization على بورد AM62x صناعي

**العنوان:** Gateway صناعي بيرفض يشوف كارت UHS-II بعد cold boot

**السياق:**
بورد AM62x بيشتغل كـ industrial gateway في مصنع. الكارت SD هو industrial-grade UHS-II بيخزن logs. المنتج اتسلّم للعميل وبعد أسبوع بدأ يشكي إن الجهاز أحياناً مش بيشوف الكارت عند التشغيل.

**المشكلة:**
عند cold boot، الـ host بيبعت `DEVICE_INIT` command صح، بس الكارت مش بيرد بـ response صالح. الـ driver بيسجل timeout وبيفشل الـ initialization كلها.

**التحليل:**
الكود في `sd_uhs2.h` بيحدد:
```c
#define UHS2_DEV_CMD_DEVICE_INIT    (UHS2_DEV_CMD_REG + 0x002)
#define UHS2_DEV_INIT_COMPLETE_FLAG  BIT(11)
#define UHS2_DEV_INIT_RESP_LEN       6
```
الـ host لازم يقرأ `UHS2_DEV_INIT_RESP_LEN = 6` bytes في الـ response ويتحقق إن `BIT(11)` موجود. لو الـ host بيـcheck الـ flag قبل ما الكارت يكمّل init — والكارت الصناعي ده بياخد وقت أطول في بيئات low-temperature — الـ flag ممكن يبقى 0.

الـ header كمان بيحدد:
```c
#define UHS2_RCLK_MIN  26000000
#define UHS2_RCLK_MAX  52000000
```
لو الـ reference clock تحت الـ 26 MHz في بيئة تشغيل بارد، الـ timing كله بيتكسر.

**الحل:**
1. تحقق من الـ clock source في DT:
```dts
/* تأكد إن mmc clock مش بتتأثر بالحرارة */
&sdhci1 {
    clocks = <&k3_clks 57 0>;
    clock-frequency = <52000000>;
};
```
2. زوّد timeout الخاص بـ `DEVICE_INIT` في الـ driver:
```c
/* wait for UHS2_DEV_INIT_COMPLETE_FLAG with extra margin */
#define UHS2_DEVICE_INIT_TIMEOUT_MS  150  /* was 50ms, industrial needs more */
```
3. افحص الـ flag صح:
```c
if (resp[1] & UHS2_DEV_INIT_COMPLETE_FLAG)
    /* init done */
```

**الدرس المستفاد:**
`UHS2_DEV_INIT_COMPLETE_FLAG = BIT(11)` مش مجرد magic number — ده contract بين الـ host والكارت. في بيئات صناعية، الـ timing margins لازم تتضاعف.

---

### السيناريو الثاني: Android TV Box بـ Allwinner H616 — سرعة النقل وخّمت

**العنوان:** UHS-II بيشتغل بسرعة UHS-I بعد driver update

**السياق:**
TV box بـ Allwinner H616 وكارت SD UHS-II. بعد تحديث kernel من 5.15 لـ 6.1، سرعة الـ read انخفضت من ~280 MB/s لـ ~50 MB/s. المستخدمين بدأوا يشتكوا من تأخر في تحميل الـ content.

**المشكلة:**
الـ driver الجديد مش بيقدر ينفّذ Speed B mode، وبيفضل على Speed A.

**التحليل:**
الـ header بيحدد:
```c
#define UHS2_DEV_CONFIG_PHY_SET_SPEED_POS  6
#define UHS2_DEV_CONFIG_PHY_SET_SPEED_A    0x0
#define UHS2_DEV_CONFIG_PHY_SET_SPEED_B    0x1
```
الـ PHY settings register هو `UHS2_DEV_CONFIG_PHY_SET = 0x00A`. الـ driver لازم يكتب `(0x1 << 6)` في الـ register ده عشان يطلب Speed B.

كمان لازم يتحقق إن الكارت بيدعمه:
```c
/* read PHY Caps */
#define UHS2_DEV_CONFIG_PHY_CAPS  (UHS2_DEV_CONFIG_REG + 0x002)
/* major version at bits [5:4], minor at bits [3:0] */
#define UHS2_DEV_CONFIG_PHY_MAJOR_POS   4
#define UHS2_DEV_CONFIG_PHY_MAJOR_MASK  0x3
```
لو الـ major version 0 — يبقى Speed A بس. لو 1 فأكتر — يعني Speed B متاح.

في الـ driver الجديد، bug بسيط: الـ bit shift للـ `SPEED_B` اتكسر:
```c
/* Wrong: shifts speed value instead of position */
phy_set |= (UHS2_DEV_CONFIG_PHY_SET_SPEED_B << UHS2_DEV_CONFIG_PHY_SET_SPEED_POS);
/* Right: */
phy_set |= (1 << UHS2_DEV_CONFIG_PHY_SET_SPEED_POS);
```

**الحل:**
```c
/* في الـ PHY configuration function */
u32 phy_set = 0;
if (card->uhs2_state.speed_range == UHS2_SPEED_B)
    phy_set |= (UHS2_DEV_CONFIG_PHY_SET_SPEED_B << UHS2_DEV_CONFIG_PHY_SET_SPEED_POS);

uhs2_write_reg(host, UHS2_DEV_CONFIG_PHY_SET, phy_set);
```

**Debug command:**
```bash
# قرّا الـ PHY settings register مباشرة
mmc extcsd read /dev/mmcblk0
# أو
cat /sys/kernel/debug/mmc0/ios
```

**الدرس المستفاد:**
`SPEED_B = 0x1` مش هو القيمة اللي بتتكتب في الـ register. القيمة اللي بتتكتب هي `(0x1 << 6)`. الـ `_POS` macros موجودة عشان تُستخدم — متجاهلوهاش.

---

### السيناريو الثالث: i.MX8 Automotive ECU — تعليق عند GO_DORMANT

**العنوان:** ECU بيتعلق عند محاولة دخول power-saving mode

**السياق:**
automotive ECU بيعمل data logging على SD card UHS-II. المنتج محتاج يدخل في hibernation بين sessions. الـ engineers لاحظوا إن الـ system بيتعلق تماماً عند الـ suspend.

**المشكلة:**
الـ driver بيبعت `GO_DORMANT_STATE` command بـ hibernation flag، بس الكارت مش بيرد ومش بيدخل dormant. الـ driver بيـblock إلى الأبد.

**التحليل:**
الـ header بيحدد:
```c
#define UHS2_DEV_CMD_GO_DORMANT_STATE  (UHS2_DEV_CMD_REG + 0x001)
#define UHS2_DEV_CMD_DORMANT_HIBER     BIT(7)
#define UHS2_GO_DORMANT_PAYLOAD_LEN    1
```

الـ `GO_DORMANT` command payload طوله byte واحد بس. `BIT(7)` في الـ payload هو اللي بيقول للكارت "ادخل hibernation مش بس dormant".

المشكلة: الـ driver بيبعت الـ flag في الـ header مش في الـ payload:
```c
/* Wrong: putting HIBER flag in header field */
cmd.header |= UHS2_DEV_CMD_DORMANT_HIBER;

/* Right: flag goes in the 1-byte payload */
cmd.payload[0] = UHS2_DEV_CMD_DORMANT_HIBER; /* BIT(7) = 0x80 */
```

كمان لازم يتحقق إن الكارت بيدعم hibernation أصلاً:
```c
#define UHS2_DEV_CONFIG_CAN_HIBER_POS   15
#define UHS2_DEV_CONFIG_CAN_HIBER_MASK  0x1

/* check PHY caps before sending HIBER */
caps = uhs2_read_reg(host, UHS2_DEV_CONFIG_PHY_CAPS);
if (!((caps >> UHS2_DEV_CONFIG_CAN_HIBER_POS) & UHS2_DEV_CONFIG_CAN_HIBER_MASK)) {
    /* card doesn't support hibernation, use plain dormant */
    cmd.payload[0] = 0;
}
```

**الحل:**
```c
/* build GO_DORMANT command correctly */
cmd.arg = UHS2_DEV_CMD_GO_DORMANT_STATE;
cmd.payload_len = UHS2_GO_DORMANT_PAYLOAD_LEN;

/* only request hibernation if card supports it */
if (card->uhs2_caps.can_hibernate)
    cmd.payload[0] = UHS2_DEV_CMD_DORMANT_HIBER;
else
    cmd.payload[0] = 0x00;
```

**الدرس المستفاد:**
`UHS2_DEV_CMD_DORMANT_HIBER = BIT(7)` دي قيمة payload — مش header flag. الـ `GO_DORMANT_PAYLOAD_LEN = 1` بيقولك بالظبط فين تحط الـ flag.

---

### السيناريو الرابع: STM32MP1 IoT Sensor — Data Corruption بعد كتابة

**العنوان:** بيانات الـ sensor بتتلف أحياناً بعد write على UHS-II card

**السياق:**
IoT sensor node بيجمع بيانات بيئية وبيكتبها على SD card UHS-II. بعد أيام تشغيل، بيانات بعض الـ sessions بتبقى corrupt. المشكلة randomized ومش قابلة للتكرار بسهولة.

**المشكلة:**
الـ data corruption بيحصل عند الكتابة بـ block sizes أكبر من max المسموح بيه للكارت.

**التحليل:**
الـ header بيحدد حدود الـ block:
```c
#define UHS2_DEV_CONFIG_MAX_BLK_LEN_POS   20
#define UHS2_DEV_CONFIG_MAX_BLK_LEN_MASK  0xFFF
#define UHS2_DEV_CONFIG_LT_SET_MAX_BLK_LEN  0x200  /* 512 bytes default */
```

الـ driver بيقرأ `LINK_TRAN_CAPS` register:
```c
#define UHS2_DEV_CONFIG_LINK_TRAN_CAPS  (UHS2_DEV_CONFIG_REG + 0x004)
```

الـ max block length موجود في bits [31:20] من الـ register ده. لو الـ driver مش بيقرأ الـ caps صح وبيفترض الـ default 512 bytes، وكارت معين عنده max أصغر — بيحصل corruption.

كمان `N_FCU` (Number of Flow Control Units) مهم:
```c
#define UHS2_DEV_CONFIG_N_FCU_POS   8
#define UHS2_DEV_CONFIG_N_FCU_MASK  0xFF
```
الـ N_FCU بيحدد كام transaction ممكن يكون in-flight. تجاوزه بيسبب data loss.

الـ data gap بين packets:
```c
#define UHS2_DEV_CONFIG_N_DATA_GAP_MASK  0xFF
/* in LINK_TRAN_CAPS1 register at 0x005 */
```
لو الـ driver مش بيحترم الـ N_DATA_GAP — الكارت ممكن يفقد بيانات.

**الحل:**
```c
/* read and respect card's actual capabilities */
u32 lt_caps = uhs2_read_reg(host, UHS2_DEV_CONFIG_LINK_TRAN_CAPS);

card->uhs2_caps.max_blk_len =
    (lt_caps >> UHS2_DEV_CONFIG_MAX_BLK_LEN_POS) & UHS2_DEV_CONFIG_MAX_BLK_LEN_MASK;

card->uhs2_caps.n_fcu =
    (lt_caps >> UHS2_DEV_CONFIG_N_FCU_POS) & UHS2_DEV_CONFIG_N_FCU_MASK;

/* enforce in link settings */
uhs2_write_reg(host, UHS2_DEV_CONFIG_LINK_TRAN_SET,
    min(card->uhs2_caps.max_blk_len, UHS2_DEV_CONFIG_LT_SET_MAX_BLK_LEN));
```

**Debug:**
```bash
# شوف الـ capabilities اللي الـ driver اكتشفها
cat /sys/bus/mmc/devices/mmc0\:0001/caps2
dmesg | grep -i "uhs2\|mmc0"
```

**الدرس المستفاد:**
`UHS2_DEV_CONFIG_LT_SET_MAX_BLK_LEN = 0x200` هو الـ default اللي الـ host يكتبه، مش القيمة الوحيدة المقبولة. دايماً اقرأ الـ caps من الكارت أولاً.

---

### السيناريو الخامس: RK3562 Custom Board Bring-Up — فشل Enumeration

**العنوان:** UHS-II card مش بتيجي في الـ enumeration على بورد جديد

**السياق:**
engineer بيعمل bring-up لـ custom board بـ RK3562 وكارت UHS-II جديد من vendor تاني. الكارت بيتعرف عليها كـ UHS-I عادي مش UHS-II، رغم إن الكارت certified UHS-II.

**المشكلة:**
الـ enumeration sequence بتفشل عند الـ `ENUMERATE` command، فالـ host بيرجع لـ legacy mode.

**التحليل:**
الـ header بيحدد الـ enumeration:
```c
#define UHS2_DEV_CMD_ENUMERATE    (UHS2_DEV_CMD_REG + 0x003)
#define UHS2_DEV_ENUM_PAYLOAD_LEN  1
#define UHS2_DEV_ENUM_RESP_LEN     8
```

الـ response طوله 8 bytes. الـ host لازم يقرأ **بالظبط** 8 bytes مش أكتر مش أقل.

الـ packet structure بتاعت الـ CCMD header:
```c
/*
 * Header bits:
 * [3:0]  DID - Destination ID
 * [6:4]  TYP - Packet Type
 * [7]    NP  - Native Packet
 * [10:8] TID - Transaction ID
 * [15:12] SID - Source ID
 */
#define UHS2_NATIVE_PACKET      (1 << UHS2_NATIVE_PACKET_POS)
#define UHS2_PACKET_TYPE_CCMD   (0 << UHS2_PACKET_TYPE_POS)
#define UHS2_DEST_ID            0x1
```

في broadcast CCMD زي ENUMERATE، `DID = SID = 0`. لو الـ driver بيبعت `DID = UHS2_DEST_ID = 0x1` بدل 0 — الكارت مش هيرد لأنه بيـexpect broadcast.

ترتيب الـ IOADR في الـ argument:
```c
/* CCMD Argument: IOADR is split */
#define UHS2_NATIVE_CCMD_GET_MIOADR_MASK  0xF00  /* bits [11:8] = MSB of IOADR */
#define UHS2_NATIVE_CCMD_MIOADR_MASK      0x0F   /* lower nibble */
#define UHS2_NATIVE_CCMD_LIOADR_POS       8
#define UHS2_NATIVE_CCMD_GET_LIOADR_MASK  0x0FF  /* bits [7:0] = LSB of IOADR */
```

الـ IOADR بيتبعت MSB first. لو الـ driver عكس الترتيب — الكارت بيفتح register غلط وبيرد بـ NACK.

```c
/* NACK handling */
#define UHS2_RES_NACK_POS   7
#define UHS2_RES_NACK_MASK  (0x1 << UHS2_RES_NACK_POS)

/* error codes */
#define UHS2_RES_ECODE_COND  1  /* condition error */
#define UHS2_RES_ECODE_ARG   2  /* argument error  */
#define UHS2_RES_ECODE_GEN   3  /* general error   */
```

الـ `ECODE_ARG = 2` بيقولك إن الـ argument (الـ IOADR) غلط.

**الحل:**
```c
/* build ENUMERATE CCMD correctly */
/* broadcast: DID=0, SID=0 */
cmd.header = UHS2_NATIVE_PACKET | UHS2_PACKET_TYPE_CCMD;
/* DID=0 for broadcast, no UHS2_DEST_ID here */

/* IOADR for ENUMERATE = UHS2_DEV_CMD_ENUMERATE = 0x203 */
u16 ioadr = UHS2_DEV_CMD_ENUMERATE;
/* MSB nibble goes to arg bits [3:0] */
cmd.arg = (ioadr >> 8) & UHS2_NATIVE_CCMD_MIOADR_MASK;
/* LSB byte goes to arg bits [15:8] */
cmd.arg |= (ioadr & UHS2_NATIVE_CCMD_GET_LIOADR_MASK) << UHS2_NATIVE_CCMD_LIOADR_POS;
cmd.arg |= UHS2_NATIVE_CMD_WRITE | UHS2_NATIVE_CMD_PLEN_4B;

/* check response for NACK */
if (resp[0] & UHS2_RES_NACK_MASK) {
    u8 ecode = (resp[0] >> UHS2_RES_ECODE_POS) & UHS2_RES_ECODE_MASK;
    dev_err(host->dev, "ENUMERATE NACK, ecode=%d\n", ecode);
}
```

**Debug commands:**
```bash
# فعّل tracing للـ MMC commands
echo 1 > /sys/kernel/debug/tracing/events/mmc/enable
cat /sys/kernel/debug/tracing/trace

# شوف الـ UHS2 state
cat /sys/bus/mmc/devices/mmc0\:0001/uhs_speed_mode
```

**الدرس المستفاد:**
`UHS2_DEST_ID = 0x1` هو الـ normal DID للكارت بعد enumeration. أثناء الـ enumeration نفسها، الـ DID لازم يكون 0 (broadcast). الـ IOADR splitting بين MSB و LSB في الـ argument لازم تتعمل بالترتيب الصح وإلا الكارت هيرد بـ `ECODE_ARG`.
## Phase 7: مصادر ومراجع

---

### 1. مقالات LWN.net

| المقال | الرابط | الأهمية |
|--------|--------|---------|
| mmc: core: add SD4.0 support | [lwn.net/Articles/642336](https://lwn.net/Articles/642336/) | أول patch لدعم SD4.0/UHS-II في الـ kernel |
| Add support UHS-II for GL9755 | [lwn.net/Articles/809085](https://lwn.net/Articles/809085/) | patch series لـ Genesys Logic GL9755 controller |
| Linux 6.17-rc7 (UHS-II fixes) | [lwn.net/Articles/1038975](https://lwn.net/Articles/1038975/) | fixes لـ sdhci-uhs2 و GL9767 initialization |
| MMC/SD/SDIO card support (kernel docs) | [static.lwn.net/kerneldoc/driver-api/mmc/index.html](https://static.lwn.net/kerneldoc/driver-api/mmc/index.html) | توثيق رسمي للـ MMC subsystem |
| Secure Digital (SD) support | [lwn.net/Articles/126098](https://lwn.net/Articles/126098/) | تاريخ دعم SD في الـ kernel |
| MMC updates | [lwn.net/Articles/253888](https://lwn.net/Articles/253888/) | تحديثات MMC subsystem |

---

### 2. توثيق الـ Kernel الرسمي

الملفات دي موجودة في شجرة الـ kernel تحت `Documentation/driver-api/mmc/`:

```
Documentation/driver-api/mmc/index.rst          - فهرس MMC subsystem
Documentation/driver-api/mmc/mmc-async-req.rst  - الـ async request API
Documentation/driver-api/mmc/mmc-dev-attrs.rst  - device attributes
Documentation/driver-api/mmc/mmc-dev-parts.rst  - device partitions
Documentation/driver-api/mmc/mmc-test.rst       - test framework
Documentation/driver-api/mmc/mmc-tools.rst      - mmc-utils tools
```

روابط الـ docs الرسمية على kernel.org:

- [MMC/SD/SDIO card support — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/driver-api/mmc/index.html)
- [SD and MMC Device Partitions](https://www.kernel.org/doc/html/latest/driver-api/mmc/mmc-dev-parts.html)
- [MMC Test Framework](https://docs.kernel.org/driver-api/mmc/mmc-test.html)
- [MMC tools introduction](https://docs.kernel.org/driver-api/mmc/mmc-tools.html)

---

### 3. الـ Source Files ذات الصلة في الـ Kernel

الملفات الأساسية اللي بتشتغل مع `include/linux/mmc/sd_uhs2.h`:

```
include/linux/mmc/sd_uhs2.h          - تعريفات UHS-II (الـ header الرئيسي)
drivers/mmc/core/sd_uhs2.c           - UHS-II core logic
drivers/mmc/host/sdhci-uhs2.c        - SDHCI UHS-II host driver
drivers/mmc/host/sdhci-uhs2.h        - SDHCI UHS-II host header
```

---

### 4. Patchwork & Kernel Mailing List

أهم patch series للـ UHS-II:

| الـ Patch | الرابط |
|-----------|--------|
| V5 GL9755 UHS-II support (cover letter) | [patchwork.kernel.org/project/linux-mmc/cover/20221019110647.11076-1-victor.shih@genesyslogic.com.tw/](https://patchwork.kernel.org/project/linux-mmc/cover/20221019110647.11076-1-victor.shih@genesyslogic.com.tw/) |
| RFC v3.2 - sdhci: add UHS-II module | [patchwork.kernel.org/project/linux-mmc/patch/20210722040124.7573-10-jasonlai.genesyslogic@gmail.com/](https://patchwork.kernel.org/project/linux-mmc/patch/20210722040124.7573-10-jasonlai.genesyslogic@gmail.com/) |
| V10 - mmc: sdhci: add UHS-II module | [lore.kernel.org/linux-kernel/20230818100217.12725-9-victorshihgli@gmail.com/](https://lore.kernel.org/linux-kernel/20230818100217.12725-9-victorshihgli@gmail.com/) |
| V21 - mmc: sdhci: add UHS-II module (2024) | [lkml.indiana.edu/hypermail/linux/kernel/2409.0/08584.html](https://lkml.indiana.edu/hypermail/linux/kernel/2409.0/08584.html) |
| mmc: core: Prepare to support SD UHS-II cards | [patchwork.kernel.org/project/linux-mmc/patch/20220418115833.10738-3-jasonlai.genesyslogic@gmail.com/](https://patchwork.kernel.org/project/linux-mmc/patch/20220418115833.10738-3-jasonlai.genesyslogic@gmail.com/) |

---

### 5. kernelnewbies.org

| الصفحة | الرابط | المحتوى |
|--------|--------|---------|
| Linux 6.11 changelog | [kernelnewbies.org/Linux_6.11](https://kernelnewbies.org/Linux_6.11) | دعم UHS-II و SDUC أتضافوا في 6.11 |
| Linux 6.18 changelog | [kernelnewbies.org/Linux_6.18](https://kernelnewbies.org/Linux_6.18) | أحدث تحديثات MMC |
| Linux versions index | [kernelnewbies.org/LinuxVersions](https://kernelnewbies.org/LinuxVersions) | فهرس كل الإصدارات |

---

### 6. eLinux.org

| الصفحة | الرابط |
|--------|--------|
| Tests: SD-SDHI-SDR104 | [elinux.org/Tests:SD-SDHI-SDR104](https://elinux.org/Tests:SD-SDHI-SDR104) |
| Tests: SDIO-with-UHS | [elinux.org/Tests:SDIO-with-UHS](https://elinux.org/Tests:SDIO-with-UHS) |

> ملحوظة: eLinux.org مش عندها صفحة dedicated لـ UHS-II، لكن UHS-I testing pages فيها context مفيد.

---

### 7. كتب مقترحة

| الكتاب | الفصول ذات الصلة |
|--------|-----------------|
| **Linux Device Drivers, 3rd Ed.** (LDD3) — Rubini, Corbet, Kroah-Hartman | Ch. 13: USB Drivers (نفس مفاهيم الـ bus/host controller)، Ch. 14: The Linux Device Model |
| **Linux Kernel Development, 3rd Ed.** — Robert Love | Ch. 17: Devices and Modules، Ch. 13: The Virtual Filesystem |
| **Embedded Linux Primer, 2nd Ed.** — Christopher Hallinan | Ch. 11: BusyBox، Ch. 14: Embedded Linux Quick Start (storage subsystems) |
| **Professional Linux Kernel Architecture** — Wolfgang Mauerer | Ch. 6: Device Drivers، Ch. 9: Data Structures |

---

### 8. مصادر تقنية إضافية

- **SD Association Specification**: UHS-II Addendum Version 1.02 — المرجع الأساسي للـ packet format و register map اللي الـ header بيعرّفها
  - الـ spec مش مجانية لكن متاحة لأعضاء SD Association على [sdcard.org](https://www.sdcard.org/)

- **mmc-utils** (أداة userspace):
  ```
  git clone https://git.kernel.org/pub/scm/utils/mmc/mmc-utils.git
  ```

- **Tom's Hardware coverage** (Linux 6.11 UHS-II):
  [Linux update adds support for 128TB SD cards](https://www.tomshardware.com/software/linux/linux-update-adds-support-for-128-terabyte-sd-cards-sduc-and-uhs-ii-sd-cards-are-now-supported)

---

### 9. Search Terms للبحث عن مزيد من المعلومات

```
# للبحث في LKML
site:lore.kernel.org "sd_uhs2" OR "sdhci-uhs2" OR "UHS-II"

# للبحث في patchwork
site:patchwork.kernel.org "linux-mmc" "UHS-II"

# للبحث في LWN
site:lwn.net "UHS-II" OR "SD 4.0" mmc

# كلمات مفتاحية تقنية مفيدة
UHS-II CCMD DCMD packet  LINK layer MMC
sdhci-uhs2 driver GL9755 GL9767 Genesys Logic
SD UHS-II Addendum 1.02 register map IOADR
mmc_uhs2_cmd mmc_uhs2_data kernel patch series
CONFIG_MMC_SDHCI_UHS2 kernel config
```

---

### 10. ملخص المسار للـ Kernel Developer الجديد

```
1. إقرأ الـ spec: SD UHS-II Addendum v1.02 (LINK/TRANS layers)
2. إفهم الـ header: include/linux/mmc/sd_uhs2.h
3. إتبع الـ core logic: drivers/mmc/core/sd_uhs2.c
4. إتعلم الـ host side: drivers/mmc/host/sdhci-uhs2.c
5. اقرأ patch history على lore.kernel.org (V5 → V21)
6. اشتغل على kernelnewbies.org/Linux_6.11 للـ context
```
## Phase 8: Writing simple module

### الهدف

ملف `sd_uhs2.h` بيعرّف ثوابت وماكروهات بروتوكول **UHS-II** — مفيش فيه exported functions أو tracepoints خاصة بيه مباشرةً. بس الكود اللي بيستخدم الثوابت دي (زي `UHS2_DEV_CONFIG_N_LANES_MASK`، `UHS2_DEV_CONFIG_PHY_SET_SPEED_B`) بيتنفذ جوا MMC core عند بدء تشغيل الكارت. أفضل نقطة hook هي **tracepoint `mmc_request_start`** اللي بيتفعّل مع كل request بيتبعت للكارت، وبالتالي بيشمل أوامر init الخاصة بـ UHS-II.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * uhs2_trace_mod.c
 *
 * Kernel module that hooks the mmc_request_start tracepoint and
 * prints UHS-II-relevant packet information (opcode, arg, host name).
 * Useful for tracing DEVICE_INIT / ENUMERATE commands defined in sd_uhs2.h.
 */

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/tracepoint.h>
#include <linux/mmc/host.h>
#include <linux/mmc/mmc.h>
#include <linux/mmc/sd_uhs2.h>      /* UHS2_DEV_CMD_* constants */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Arabic Kernel Docs Project");
MODULE_DESCRIPTION("Trace MMC requests to observe UHS-II init commands");

/* ------------------------------------------------------------------ */
/* Tracepoint probe: called for every mmc_request_start event          */
/* ------------------------------------------------------------------ */
static void probe_mmc_request_start(void *data,
                                    struct mmc_host *host,
                                    struct mmc_request *mrq)
{
    u32 opcode = 0;
    u32 arg    = 0;

    /* Guard against a request with no command */
    if (!mrq || !mrq->cmd)
        return;

    opcode = mrq->cmd->opcode;
    arg    = mrq->cmd->arg;

    /*
     * UHS-II Device Init command lives at IOADR 0x202 (UHS2_DEV_CMD_DEVICE_INIT).
     * We flag it specially so it stands out in dmesg.
     */
    if (arg == UHS2_DEV_CMD_DEVICE_INIT)
        pr_info("uhs2_trace: [%s] UHS-II DEVICE_INIT detected! "
                "opcode=%u arg=0x%08x\n",
                mmc_hostname(host), opcode, arg);
    else
        pr_info("uhs2_trace: [%s] mmc_request_start "
                "opcode=%u arg=0x%08x\n",
                mmc_hostname(host), opcode, arg);
}

/* ------------------------------------------------------------------ */
/* Tracepoint lookup helper — kernel requires us to find the tp first  */
/* ------------------------------------------------------------------ */
static struct tracepoint *tp_mmc_request_start;

/* Called by for_each_kernel_tracepoint() for every registered tp */
static void find_mmc_tp(struct tracepoint *tp, void *priv)
{
    /* Compare by name to locate the right tracepoint */
    if (!strcmp(tp->name, "mmc_request_start"))
        tp_mmc_request_start = tp;
}

/* ------------------------------------------------------------------ */
/* Module init: register our probe on the tracepoint                   */
/* ------------------------------------------------------------------ */
static int __init uhs2_trace_init(void)
{
    int ret;

    /* Walk all registered tracepoints to find mmc_request_start */
    for_each_kernel_tracepoint(find_mmc_tp, NULL);

    if (!tp_mmc_request_start) {
        pr_err("uhs2_trace: mmc_request_start tracepoint not found\n");
        return -ENODEV;
    }

    /* Register our callback; NULL = no per-probe private data */
    ret = tracepoint_probe_register(tp_mmc_request_start,
                                    probe_mmc_request_start,
                                    NULL);
    if (ret) {
        pr_err("uhs2_trace: failed to register probe: %d\n", ret);
        return ret;
    }

    pr_info("uhs2_trace: module loaded, watching mmc_request_start\n");
    return 0;
}

/* ------------------------------------------------------------------ */
/* Module exit: cleanly unregister the probe                           */
/* ------------------------------------------------------------------ */
static void __exit uhs2_trace_exit(void)
{
    if (tp_mmc_request_start)
        tracepoint_probe_unregister(tp_mmc_request_start,
                                    probe_mmc_request_start,
                                    NULL);

    /* Ensure all RCU readers of the tracepoint have finished */
    tracepoint_synchronize_unregister();

    pr_info("uhs2_trace: module unloaded\n");
}

module_init(uhs2_trace_init);
module_exit(uhs2_trace_exit);
```

---

### شرح كل جزء

| الجزء | الغرض |
|-------|--------|
| `#include <linux/mmc/sd_uhs2.h>` | بيجيب ثوابت UHS-II زي `UHS2_DEV_CMD_DEVICE_INIT` اللي بنستخدمها في المقارنة |
| `probe_mmc_request_start()` | الـ callback اللي بيتنفذ مع كل request — بيطبع اسم الـ host والـ opcode والـ arg، وبيميّز أوامر UHS-II init بشكل خاص |
| `find_mmc_tp()` + `for_each_kernel_tracepoint()` | بيدوّر على الـ tracepoint بالاسم وقت runtime لأن الـ symbol مش دايمًا exported مباشرةً |
| `tracepoint_probe_register()` | بيربط الـ callback بالـ tracepoint بطريقة RCU-safe |
| `tracepoint_synchronize_unregister()` | بيضمن إن محدش لسه شغّال جوا الـ callback قبل ما المودول يتفك |
| `MODULE_LICENSE("GPL")` | لازم يكون GPL عشان الـ kernel يسمح بالوصول للـ tracepoint API |

---

### ملاحظات تشغيل

```bash
# بناء المودول (مع وجود Makefile مناسب)
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# تحميل
sudo insmod uhs2_trace_mod.ko

# مراقبة الـ output
sudo dmesg -w | grep uhs2_trace

# تفريغ
sudo rmmod uhs2_trace_mod
```

> لو الجهاز مفيش فيه كارت UHS-II فعلي، الـ probe هيتفعّل مع أي MMC/SD request عادي وهتشوف الـ opcode والـ arg في dmesg — ده في حد ذاته مفيد لفهم تسلسل أوامر الـ MMC.
