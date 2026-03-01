## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي ليه الملف

الـ `include/linux/mii.h` جزء من **ETHERNET PHY LIBRARY** — المكتبة المسئولة عن التعامل مع الـ **PHY chips** (الـ Physical Layer transceivers) في الكيرنل. المشرفين عليه هم Andrew Lunn وHeiner Kallweit، والـ mailing list هو `netdev@vger.kernel.org`.

---

### القصة من الأول: إيه هو الـ PHY وليه محتاجينه؟

تخيل إن جهازك بيتكلم مع الشبكة. جوه الجهاز فيه قطعتين:

```
[ CPU / OS ]
     │
[ MAC — Media Access Controller ]   ← بيتكلم "لغة رقمية" داخلية
     │
    MII Bus  ← سلك رفيع بيربط الاتنين
     │
[ PHY — Physical Layer Chip ]       ← بيحوّل الإشارات الرقمية لكهربا على السلك
     │
[ كابل الإيثرنت / الشبكة ]
```

الـ **MAC** بيعرف يبعت ويستقبل frames، لكنه مش بيعرف يتكلم مع الكابل مباشرة. الـ **PHY** هو اللي بيعمل التحويل الفيزيائي. الـ **MII** (Media Independent Interface) هو البروتوكول اللي بيربط الاتنين ده ببعض — زي لغة مشتركة بينهم.

الـ MII اتعرّف في **IEEE 802.3u** سنة 1995، والهدف منه إن الـ MAC تبقى مستقلة عن نوع الـ PHY. تقدر تحط أي PHY chip من أي شركة طالما بيتكلم MII.

---

### الـ MII Management Bus: MDIO

غير إن الـ MII بيبعت البيانات، فيه **bus تانية** اسمها **MDIO** (Management Data Input/Output) — زي serial bus صغير بسلكين فقط. من خلاله الـ MAC بتقرأ وتكتب **registers** جوه الـ PHY.

كل PHY عنده 32 register بحجم 16-bit كل واحدة. أهمها:

| Register | العنوان | الوظيفة |
|---|---|---|
| `BMCR` | 0x00 | Basic Mode Control — تشغيل/إيقاف autoneg، تحديد السرعة |
| `BMSR` | 0x01 | Basic Mode Status — هل الـ link موجود؟ |
| `MII_ADVERTISE` | 0x04 | إيه السرعات اللي أنا عارضها للطرف التاني |
| `MII_LPA` | 0x05 | إيه السرعات اللي الطرف التاني عارضها |
| `MII_CTRL1000` | 0x09 | إعلانات الـ 1000BASE-T |
| `MII_STAT1000` | 0x0a | نتيجة negotiation الـ 1G |

---

### القصة الكاملة: الـ Autonegotiation

لما بتوصّل كابل إيثرنت:

1. الـ PHY بتاعك بيبعت **إعلان** على السلك: "أنا أقدر أشتغل بـ 10/100/1000 Mbps، full أو half duplex".
2. الـ PHY التاني (في السويتش مثلاً) بيرد بإعلانه هو.
3. الاتنين بيتفقوا على **أعلى سرعة مشتركة** بينهم — ده اللي بيتسمى **autonegotiation** أو **N-Way**.
4. النتيجة بتتخزن في `MII_LPA` (Link Partner Ability).

الـ `mii.h` بيوفر الـ helper functions اللي بتقرأ الـ registers دي وبتحولها لـ format تاني يفهمه الـ ethtool أو الـ linkmode API.

---

### هدف الملف `include/linux/mii.h` تحديداً

الملف ده هو **الـ bridge بين عالمين**:

```
[ عالم MII registers — أرقام hex خام ]
           ↕
[ include/linux/mii.h ]
           ↕
[ عالم ethtool/linkmode — abstractions عالية المستوى ]
```

**ثلاث وظائف رئيسية:**

1. **تعريف `struct mii_if_info`** — الـ struct الأساسي اللي بيمثل PHY قديم يشتغل بالـ legacy MII interface. بيحتوي على function pointers لـ `mdio_read` و`mdio_write`.

2. **إعلان الـ external functions** — زي `mii_link_ok()`, `mii_check_media()`, `generic_mii_ioctl()` — المنفّذة في `drivers/net/mii.c`.

3. **مجموعة ضخمة من الـ inline conversion functions** — بتحوّل بين ثلاث تمثيلات مختلفة لنفس المعلومة (السرعات والـ duplex المدعومة):
   - **MII register bits** — القيم الخام في رجيستر الـ PHY
   - **ethtool legacy bits** — `ADVERTISED_xxx` flags القديمة
   - **linkmode bitmap** — الـ API الحديث اللي بيستخدم bitmaps

---

### ليه في ثلاث تمثيلات؟ (القصة الكاملة)

الكيرنل مرّ بتطور:

- **المرحلة الأولى:** الـ `ethtool` استخدم `u32` بيتس بسيطة (`ADVERTISED_10baseT_Half`, إلخ) — ده اللي اتسمى **legacy ethtool bits**.
- **المشكلة:** الـ `u32` اتملا! مع السرعات الجديدة (25G, 100G, إلخ) مكنش في مكان.
- **الحل:** الكيرنل حوّل لـ **linkmode** — `unsigned long` bitmap قابل للتوسع.
- **النتيجة:** المكتبة لازم تدعم الاتنين في نفس الوقت خلال فترة الـ migration.

فالـ `mii.h` بيوفر كل permutations التحويل:

```c
ethtool_adv_to_mii_adv_t()      /* ethtool legacy  → MII register */
mii_adv_to_ethtool_adv_t()      /* MII register    → ethtool legacy */
linkmode_adv_to_mii_adv_t()     /* linkmode        → MII register */
mii_adv_mod_linkmode_adv_t()    /* MII register    → linkmode */
```

---

### الفرق بين `_t` و`_x` في أسماء الـ functions

- **`_t`** = **1000BASE-T** — إيثرنت على كابل twisted pair (الكابل العادي في بيتك).
- **`_x`** = **1000BASE-X** — إيثرنت على فايبر أوبتيك أو SFP modules.

الـ registers نفسها بتتستخدم لكن bits معانيها مختلفة حسب الـ mode، فعشان كده في functions منفصلة للاتنين.

---

### الـ Flow Control

الـ **flow control** هو آلية بيقول فيها الـ switch للـ NIC "استنّى شوية، أنا مشغول" — زي traffic light على الشبكة. الـ `mii_advertise_flowctrl()` و`mii_resolve_flowctrl_fdx()` بيتعاملوا مع negotiation الـ pause frames حسب IEEE 802.3.

---

### ASCII: الصورة الكاملة

```
Driver قديم (legacy)
        │
        ▼
struct mii_if_info
  ├── phy_id
  ├── mdio_read()  ──────────────► MDIO Bus ──► PHY Register
  └── mdio_write() ◄──────────────              (BMCR, BMSR, ...)
        │
        ▼
  mii.h helper functions
        │
   ┌────┴─────────┐
   ▼              ▼
ethtool API    linkmode API
(legacy u32)   (modern bitmap)
        │              │
        └──────┬────────┘
               ▼
         userspace (ethtool cmd)
```

---

### الملفات المرتبطة اللي المفروض تعرفها

| الملف | الوظيفة |
|---|---|
| `include/uapi/linux/mii.h` | الـ register addresses والـ bit definitions — shared مع userspace |
| `drivers/net/mii.c` | التنفيذ الفعلي للـ functions المُعلنة في `mii.h` |
| `include/linux/phy.h` | الـ modern PHY framework — الـ `phy_device` struct والـ PHYLIB |
| `include/linux/linkmode.h` | الـ linkmode bitmap API المستخدم في الـ conversions |
| `drivers/net/phy/phy_device.c` | الـ core PHYLIB implementation |
| `drivers/net/phy/phy.c` | إدارة الـ PHY state machine |
| `drivers/net/phy/phy-core.c` | الـ helper functions للـ modern PHY framework |
| `drivers/net/mdio/` | الـ MDIO bus controllers (hardware specific) |
| `include/linux/mdio.h` | الـ MDIO/Clause 45 extensions |

---

### ملاحظة مهمة: Legacy vs Modern

الـ `mii.h` بالـ `struct mii_if_info` ده **الـ legacy path** — بيستخدمه الـ drivers القديمة اللي اتكتبت قبل PHYLIB. الـ drivers الحديثة بتستخدم `struct phy_device` من `include/linux/phy.h` بدلاً منه. لكن الـ conversion functions في `mii.h` لسه مستخدمة حتى في الـ PHYLIB نفسه لأنها بتتعامل مع MII register format الموحد.
## Phase 2: شرح الـ MII (Media Independent Interface) Framework

### المشكلة — ليه الـ MII موجود أصلاً؟

في الـ Ethernet، في قطعتين أساسيتين:

1. **MAC (Media Access Controller)** — اللي بيتحكم في الـ framing، الـ CSMA/CD، وإرسال الـ data. ده جزء من الـ SoC أو الـ NIC chip.
2. **PHY (Physical Layer Transceiver)** — اللي بيحول الـ digital bits لإشارات analog على السلك الفعلي (twisted pair, fiber, etc.).

المشكلة الأصلية: كل vendor كان بيعمل MAC وPHY يشتغلوا مع بعض بـ interface خاص بيه. يعني لو عملت driver لـ MAC معين، هو بيشتغل بس مع PHY معين من نفس الـ vendor.

**IEEE 802.3u** جه بـ MII كـ standard interface بين الـ MAC والـ PHY. الفكرة: أي MAC يشتغل مع أي PHY طالما الاتنين بيتكلموا MII.

لكن المشكلة امتدت للـ kernel: كل driver كان بيعيد نفس الكود من أول — قراءة الـ PHY registers، عمل autonegotiation، ترجمة السرعة والـ duplex. الـ kernel محتاج framework يـ abstract ده.

---

### الحل — الـ MII Helper Library

الـ `linux/mii.h` مش framework كامل زي الـ PHY subsystem (`linux/phy.h`). هو **helper library** بتقدم:

- **Struct واحدة** تـ encapsulate معلومات الـ PHY connection: `mii_if_info`
- **Function pointers** لـ read/write register بيخلي الكود generic
- **Inline helpers** لترجمة الـ register values لـ ethtool format والعكس
- **Autonegotiation logic** جاهزة (priority order، duplex resolution، flow control)

ده legacy API — الـ modern drivers بيستخدموا الـ `phylib` (PHY library في `linux/phy.h`). لكن الـ MII helpers لسه موجودة ومستخدمة في drivers قديمة وبسيطة.

---

### التشبيه الحقيقي — الـ Universal Remote Control

تخيل عندك تلفزيونات من brands مختلفة (Sony، Samsung، LG). كل تلفزيون ليه protocol خاص بيه.

**الـ Universal Remote** هو الـ MII:
- الـ **buttons (Volume+، Channel+)** = الـ standard MII registers (BMCR، BMSR، ADVERTISE، LPA)
- الـ **IR codes** اللي بيتبعت = محتوى الـ registers الفعلي (bit fields)
- الـ **remote itself** = `mii_if_info` struct — بيمسك الـ state
- **برمجة الـ remote للتلفزيون** = الـ `mdio_read`/`mdio_write` function pointers اللي بيبرمجها الـ driver
- **ترجمة "أنا عايز أرفع الصوت" لـ IR code** = الـ helper functions زي `ethtool_adv_to_mii_adv_t()`

الـ **ethtool** (user-space tool) بيتكلم مع الـ kernel بلغة عامة: "أنا عايز 100Mbps full-duplex". الـ helper functions بتترجم ده للـ bits الصح في الـ PHY registers.

---

### الـ Big Picture Architecture

```
User-space (ethtool tool)
         |
         | SIOCETHTOOL / SIOCxMIIxxx ioctl
         |
+--------v-----------------------------------------+
|              Network Stack (kernel)               |
|  +-----------+    +---------+    +------------+  |
|  | ethtool   |    |  ioctl  |    |  linkmode  |  |
|  | subsystem |    | handler |    |  bitmap    |  |
|  +-----+-----+    +----+----+    +-----+------+  |
|        |               |               |          |
|        +-------+-------+---------------+          |
|                |                                  |
|       +--------v---------+                        |
|       |   mii_if_info    |   <-- mii.h core      |
|       |  +-----------+   |                        |
|       |  | phy_id    |   |                        |
|       |  | mdio_read |---+--> driver-specific     |
|       |  | mdio_write|---+--> MDIO bus access     |
|       |  | full_duplex   |                        |
|       |  | force_media   |                        |
|       |  | supports_gmii |                        |
|       |  +-----------+   |                        |
|       +------------------+                        |
|                                                  |
+------------------+-------------------------------+
                   | MDIO bus (Management Data I/O)
         +---------v---------+
         |   PHY chip (e.g.  |
         |   Marvell 88E1111 |
         |   Micrel KSZ8081  |
         |   SMSC LAN8720    |
         +---------+---------+
                   |
                   | MII/RMII/RGMII/SGMII signals
         +---------v---------+
         |    Ethernet       |
         |    Cable/Fiber    |
         +-------------------+

Registers accessed via MDIO:
  Reg 0x00: BMCR  (Basic Mode Control)
  Reg 0x01: BMSR  (Basic Mode Status)
  Reg 0x02: PHYSID1
  Reg 0x03: PHYSID2
  Reg 0x04: ADVERTISE
  Reg 0x05: LPA   (Link Partner Ability)
  Reg 0x09: CTRL1000
  Reg 0x0A: STAT1000
```

---

### الـ Core Abstraction — `mii_if_info`

ده الـ central struct في الـ MII helper library:

```c
struct mii_if_info {
    int phy_id;           /* PHY address على الـ MDIO bus (0-31) */
    int advertising;      /* ايه اللي بنـ advertise (legacy ethtool bitmask) */
    int phy_id_mask;      /* mask لـ phy_id لما تعمل MDIO read/write */
    int reg_num_mask;     /* mask لرقم الـ register */

    unsigned int full_duplex   : 1; /* current duplex state */
    unsigned int force_media   : 1; /* autoneg disabled? (forced speed/duplex) */
    unsigned int supports_gmii : 1; /* هل الـ PHY بيدعم GMII (Gigabit)? */

    struct net_device *dev;   /* الـ net device المرتبط */

    /* function pointers — الـ driver بيحطهم */
    int  (*mdio_read) (struct net_device *dev, int phy_id, int location);
    void (*mdio_write)(struct net_device *dev, int phy_id, int location, int val);
};
```

الفكرة: الـ driver بيعبي الـ function pointers بكود بيعرف يتكلم مع الـ MDIO bus الخاص بيه. بعد كده، كل الـ generic logic (autoneg، link check، ethtool translation) بتتعمل على الـ `mii_if_info` من غير ما تعرف ايه الـ hardware.

---

### الـ Register Map — ازاي الـ PHY بيتكلم

**الـ MII protocol** بيتكلم عبر الـ **MDIO bus** — serial bus بيستخدم 2 pins: MDC (clock) وMDIO (data). كل PHY على الـ bus ليه address من 0 لـ 31.

```
MDIO Bus Structure:
  MAC/SoC                          PHY chips
  +-------+    MDC  ____________   +--------+  addr=1
  | MDIO  |----MDC-|            |--| PHY_1  |
  | ctrl  |        | MDIO bus   |  +--------+
  |       |--MDIO--| (shared)   |
  +-------+        |____________|  +--------+  addr=2
                                ---| PHY_2  |
                                   +--------+

                                   +--------+  addr=N
                                ---| PHY_N  |
                                   +--------+
```

الـ registers المهمة:

| Register | Address | الوظيفة |
|----------|---------|---------|
| BMCR | 0x00 | التحكم: enable autoneg، force speed، reset |
| BMSR | 0x01 | Status: link up/down، autoneg complete |
| PHYSID1/2 | 0x02-0x03 | PHY manufacturer OUI + model |
| ADVERTISE | 0x04 | ايه السرعات اللي بنقولها لـ link partner |
| LPA | 0x05 | ايه السرعات اللي الـ link partner بيعلن عنها |
| CTRL1000 | 0x09 | Gigabit advertisement |
| STAT1000 | 0x0A | Gigabit link partner status |

---

### الـ Autonegotiation Flow — خطوة بخطوة

الـ **autonegotiation** هو البروتوكول اللي بيخلي الطرفين يتفقوا على أحسن سرعة ودـ duplex مشتركين:

```
Local PHY                              Remote PHY (link partner)
    |                                          |
    | Write to ADVERTISE reg (0x04):           |
    | - ADVERTISE_10HALF  = 0x0020            |
    | - ADVERTISE_10FULL  = 0x0040            |
    | - ADVERTISE_100HALF = 0x0080            |
    | - ADVERTISE_100FULL = 0x0100            |
    |                                          |
    | Set BMCR: BMCR_ANENABLE | BMCR_ANRESTART|
    |                                          |
    |<-------- Fast Link Pulses (FLP) -------->|
    |     (exchange of ability registers)      |
    |                                          |
    | Read LPA reg (0x05) =                   |
    |   partner's ADVERTISE register           |
    |                                          |
    | negotiated = ADVERTISE & LPA             |
    |   (bitwise AND = common capabilities)   |
    |                                          |
    | mii_nway_result(negotiated):            |
    |   Priority: 100FULL > 100BASE4 >        |
    |             100HALF > 10FULL > 10HALF   |
    v                                          v
   Use highest common mode
```

الكود في `mii_nway_result()`:

```c
static inline unsigned int mii_nway_result(unsigned int negotiated)
{
    /* IEEE 802.3u priority order — أعلى سرعة + full duplex أولاً */
    if (negotiated & LPA_100FULL)
        return LPA_100FULL;
    else if (negotiated & LPA_100BASE4)
        return LPA_100BASE4;
    else if (negotiated & LPA_100HALF)
        return LPA_100HALF;
    else if (negotiated & LPA_10FULL)
        return LPA_10FULL;
    else
        return LPA_10HALF;  /* worst case */
}
```

---

### الـ Three Worlds of Speed Advertisement

في الـ MII framework تلت تمثيلات للـ link capabilities:

```
+------------------+     +------------------+     +------------------+
|  MII Registers   |     | ethtool legacy   |     | linkmode bitmap  |
|  (hardware)      |     | (u32 bitmask)    |     | (modern API)     |
+------------------+     +------------------+     +------------------+
| ADVERTISE_10HALF |<--->| ADVERTISED_      |<--->| ETHTOOL_LINK_    |
| = 0x0020        | (1) | 10baseT_Half     | (2) | MODE_10baseT_    |
|                 |     |                  |     | Half_BIT         |
| ADVERTISE_      |     | ADVERTISED_      |     | ETHTOOL_LINK_    |
| 100FULL         |<--->| 100baseT_Full    |<--->| MODE_100baseT_   |
| = 0x0100        |     |                  |     | Full_BIT         |
+------------------+     +------------------+     +------------------+

(1): ethtool_adv_to_mii_adv_t() / mii_adv_to_ethtool_adv_t()
(2): linkmode_adv_to_mii_adv_t() / mii_adv_to_linkmode_adv_t()
```

الـ **linkmode** هو الـ modern representation — bitmap كبير (ليس u32) بيقدر يمثل كل أنواع الـ links من 10Mbps لـ 400Gbps. الـ `linkmode.h` بيوفر macros زي `linkmode_test_bit()` و`linkmode_mod_bit()` اللي هي wrappers على الـ standard kernel `bitmap` operations.

---

### الـ Flow Control Resolution — جدول IEEE 802.3

الـ **flow control** في Ethernet بيستخدم PAUSE frames عشان يوقف الـ transmitter مؤقتاً لما الـ buffer بيـ overflow.

```c
static inline u8 mii_resolve_flowctrl_fdx(u16 lcladv, u16 rmtadv)
{
    u8 cap = 0;

    if (lcladv & rmtadv & ADVERTISE_PAUSE_CAP) {
        /* الاتنين بيدعموا symmetric pause */
        cap = FLOW_CTRL_TX | FLOW_CTRL_RX;
    } else if (lcladv & rmtadv & ADVERTISE_PAUSE_ASYM) {
        if (lcladv & ADVERTISE_PAUSE_CAP)
            cap = FLOW_CTRL_RX;     /* أنا بـ receive pause فقط */
        else if (rmtadv & ADVERTISE_PAUSE_CAP)
            cap = FLOW_CTRL_TX;     /* أنا بـ send pause فقط */
    }

    return cap;
}
```

بيطبق **Table 28B-3 من IEEE 802.3-2005**:

| lcl PAUSE | lcl ASYM | rmt PAUSE | rmt ASYM | TX | RX |
|-----------|----------|-----------|----------|----|----|
| 1 | X | 1 | X | YES | YES |
| 0 | 1 | 1 | 1 | YES | NO |
| 1 | 1 | 0 | 1 | NO | YES |
| else | | | | NO | NO |

---

### 1000BASE-X vs 1000BASE-T — نفس الـ Register، معنى مختلف

ده من أكثر النقاط اللي بتلخبط:

```
1000BASE-T (copper, 4 pairs twisted):
  - CTRL1000 register (0x09): ADVERTISE_1000HALF / ADVERTISE_1000FULL
  - STAT1000 register (0x0A): LPA_1000HALF / LPA_1000FULL
  - Functions: ethtool_adv_to_mii_ctrl1000_t(), mii_stat1000_to_ethtool_lpa_t()

1000BASE-X (fiber, SFP):
  - ADVERTISE register (0x04) بنفس bits لكن معناها مختلف:
    ADVERTISE_1000XFULL = 0x0020 (نفس value زي ADVERTISE_10HALF!)
    ADVERTISE_1000XHALF = 0x0040 (نفس value زي ADVERTISE_10FULL!)
  - Functions: ethtool_adv_to_mii_adv_x(), mii_adv_to_ethtool_adv_x()
```

نفس الـ register bits بتتفسر بشكل مختلف حسب نوع الـ PHY. الـ driver هو اللي يعرف هو بيتعامل مع copper ولا fiber ويستخدم الـ function الصح.

---

### SGMII — حالة خاصة للـ SoC-to-PHY

**SGMII (Serial Gigabit MII)** بيُستخدم بين الـ MAC في الـ SoC والـ external PHY عبر in-band autonegotiation على 4 wires بدل 18:

```
SoC                        External PHY
+----------+   SGMII      +----------+    RJ45
|   MAC    |<------------>|   PHY    |<-------->
| (in SoC) |  1Gbps serial|          |  twisted
+----------+              +----------+    pair

In-band autoneg via config_reg:
  Bits [12]: Full duplex
  Bits [11:10]: Speed (00=10M, 01=100M, 10=1G)
  Bit [15]: Link up
```

الـ SGMII LPA constants في `uapi/linux/mii.h`:
```c
#define LPA_SGMII_100FULL  0x1400  /* 100Mbps FD */
#define LPA_SGMII_1000FULL 0x1800  /* 1000Mbps FD */
#define LPA_SGMII_LINK     0x8000  /* PHY has link */
```

---

### الـ BMCR — التحكم المباشر في الـ PHY

الـ **Basic Mode Control Register (0x00)** هو أهم register في الـ PHY:

```
Bit 15: RESET        - software reset للـ PHY
Bit 14: LOOPBACK     - internal loopback للـ testing
Bit 13: SPEED100     - MSB للـ speed selection
Bit 12: ANENABLE     - enable autonegotiation
Bit 11: POWERDOWN    - low power mode
Bit 10: ISOLATE      - فصل الـ PHY عن الـ MII bus
Bit  9: ANRESTART    - restart autonegotiation
Bit  8: FULLDPLX     - full duplex (لما autoneg disabled)
Bit  7: CTST         - collision test
Bit  6: SPEED1000    - LSB للـ speed selection

Speed encoding:
  SPEED1000=1, SPEED100=0 => 1000Mbps
  SPEED1000=0, SPEED100=1 => 100Mbps
  SPEED1000=0, SPEED100=0 => 10Mbps
```

الكود في `mii_bmcr_encode_fixed()`:

```c
static inline u16 mii_bmcr_encode_fixed(int speed, int duplex)
{
    u16 bmcr;
    switch (speed) {
    case SPEED_2500:  /* 2500 بيتعمل encode كـ 1000 */
    case SPEED_1000:
        bmcr = BMCR_SPEED1000;  /* bit 6 = 1 */
        break;
    case SPEED_100:
        bmcr = BMCR_SPEED100;   /* bit 13 = 1 */
        break;
    default:
        bmcr = BMCR_SPEED10;    /* 0x0000 */
    }
    if (duplex == DUPLEX_FULL)
        bmcr |= BMCR_FULLDPLX;
    return bmcr;
}
```

---

### ايه اللي الـ MII بيملكه vs ايه اللي بيـ delegate للـ Driver

| الـ MII Framework يملك | الـ Driver يقدمه |
|------------------------|-----------------|
| منطق الـ autonegotiation priority | `mdio_read()` — physical register access |
| ترجمة بين register bits وethtool/linkmode | `mdio_write()` — physical register write |
| IEEE 802.3 flow control resolution | MDIO bus implementation (polling/interrupt) |
| `mii_link_ok()` — check الـ BMSR | PHY address على الـ bus |
| `mii_check_media()` — detect link changes | `net_device` struct |
| `generic_mii_ioctl()` — handle user ioctls | معرفة نوع الـ PHY (T vs X vs SGMII) |
| `mii_nway_restart()` — restart autoneg | تحديد `supports_gmii` |

---

### Real-World Example — Driver بيستخدم MII

الـ `drivers/net/ethernet/8390/ne2k-pci.c` وغيره من الـ simple NIC drivers:

```c
/* Driver initialization */
static int mynic_init(struct net_device *dev)
{
    struct mynic_priv *priv = netdev_priv(dev);

    /* عبي الـ mii_if_info */
    priv->mii.phy_id      = 1;        /* PHY address على MDIO bus */
    priv->mii.phy_id_mask = 0x1f;
    priv->mii.reg_num_mask= 0x1f;
    priv->mii.dev         = dev;
    priv->mii.mdio_read   = mynic_mdio_read;   /* hardware-specific */
    priv->mii.mdio_write  = mynic_mdio_write;  /* hardware-specific */
    priv->mii.full_duplex = 1;
    priv->mii.supports_gmii = 0;  /* 10/100 only */
    return 0;
}

/* الـ ethtool ops بتستخدم MII helpers مباشرة */
static void mynic_get_settings(struct net_device *dev,
                                struct ethtool_cmd *cmd)
{
    struct mynic_priv *priv = netdev_priv(dev);
    mii_ethtool_gset(&priv->mii, cmd);  /* كل الشغل هنا */
}

/* Link status check في الـ timer أو interrupt */
static void mynic_check_link(struct net_device *dev)
{
    struct mynic_priv *priv = netdev_priv(dev);
    mii_check_link(&priv->mii);  /* بيعمل netif_carrier_on/off */
}
```

---

### العلاقة مع الـ PHY Subsystem الحديث

الـ MII helper library هي **generation أول**. الـ modern approach هو **phylib** (`include/linux/phy.h`):

```
Legacy (mii.h):                    Modern (phy.h):

Driver                             Driver
  |                                  |
  | mii_if_info                      | phy_device *phydev
  | mdio_read/write (fn ptr)         |
  |                                  | phy_connect()
  v                                  v
MII helpers                        phylib framework
  |                                  |
  | direct register access           | MDIO bus abstraction
  v                                  v
MDIO bus                           MDIO bus
  |                                  |
  v                                  v
PHY chip                           PHY driver (separate)
                                     |
                                     v
                                   PHY chip
```

الـ phylib بيضيف: PHY drivers منفصلين، state machine للـ link management، interrupt support، وdeferrable work. الـ MII helpers لسه مفيدين في الـ cases البسيطة أو الـ drivers القديمة اللي مش عايزة كل الـ complexity دي.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Constants — Cheatsheet

#### MII Register Map (أهم الـ registers)

| Register | Offset | الوظيفة |
|---|---|---|
| `MII_BMCR` | `0x00` | Basic Mode Control — يتحكم في speed/duplex/autoneg |
| `MII_BMSR` | `0x01` | Basic Mode Status — يبيّن قدرات الـ PHY وحالة الـ link |
| `MII_PHYSID1/2` | `0x02/03` | معرّف الـ PHY (OUI + model) |
| `MII_ADVERTISE` | `0x04` | ما يعلنه الـ PHY في الـ autoneg |
| `MII_LPA` | `0x05` | ما يعلنه الـ link partner |
| `MII_EXPANSION` | `0x06` | إمكانيات إضافية للـ autoneg |
| `MII_CTRL1000` | `0x09` | التحكم في 1000BASE-T |
| `MII_STAT1000` | `0x0a` | حالة الـ link partner في 1000BASE-T |
| `MII_MMD_CTRL` | `0x0d` | الوصول لـ MDIO Manageable Device registers |
| `MII_MMD_DATA` | `0x0e` | بيانات الـ MMD |
| `MII_ESTATUS` | `0x0f` | Extended Status — يبيّن دعم 1000BASE-X/T |

---

#### BMCR Bits (Basic Mode Control Register)

| Bit | Mask | المعنى |
|---|---|---|
| `BMCR_RESET` | `0x8000` | reset الـ PHY لحالته الافتراضية |
| `BMCR_LOOPBACK` | `0x4000` | loopback mode للاختبار |
| `BMCR_SPEED100` | `0x2000` | اختيار 100 Mbps (مع bit 6 = 0) |
| `BMCR_ANENABLE` | `0x1000` | تفعيل الـ autonegotiation |
| `BMCR_PDOWN` | `0x0800` | power down للـ PHY |
| `BMCR_ISOLATE` | `0x0400` | عزل الـ PHY عن الـ MII bus |
| `BMCR_ANRESTART` | `0x0200` | إعادة تشغيل الـ autoneg |
| `BMCR_FULLDPLX` | `0x0100` | full duplex |
| `BMCR_SPEED1000` | `0x0040` | اختيار 1000 Mbps (MSB) |
| `BMCR_SPEED10` | `0x0000` | اختيار 10 Mbps |

> Speed encoding: `BMCR_SPEED100=1, BMCR_SPEED1000=0` → 100M | كلاهما 0 → 10M | `BMCR_SPEED1000=1` → 1000M

---

#### BMSR Bits (Basic Mode Status Register)

| Bit | Mask | المعنى |
|---|---|---|
| `BMSR_100BASE4` | `0x8000` | يدعم 100BASE-T4 |
| `BMSR_100FULL` | `0x4000` | يدعم 100M full-duplex |
| `BMSR_100HALF` | `0x2000` | يدعم 100M half-duplex |
| `BMSR_10FULL` | `0x1000` | يدعم 10M full-duplex |
| `BMSR_10HALF` | `0x0800` | يدعم 10M half-duplex |
| `BMSR_ESTATEN` | `0x0100` | يوجد Extended Status Register |
| `BMSR_ANEGCOMPLETE` | `0x0020` | الـ autoneg اكتملت |
| `BMSR_RFAULT` | `0x0010` | remote fault |
| `BMSR_ANEGCAPABLE` | `0x0008` | يدعم autoneg |
| `BMSR_LSTATUS` | `0x0004` | الـ link شغّال |

---

#### ADVERTISE / LPA Bits (MII_ADVERTISE & MII_LPA)

| Constant | Mask | المعنى |
|---|---|---|
| `ADVERTISE_10HALF` | `0x0020` | 10M half-duplex |
| `ADVERTISE_10FULL` | `0x0040` | 10M full-duplex |
| `ADVERTISE_100HALF` | `0x0080` | 100M half-duplex |
| `ADVERTISE_100FULL` | `0x0100` | 100M full-duplex |
| `ADVERTISE_100BASE4` | `0x0200` | 100BASE-T4 |
| `ADVERTISE_PAUSE_CAP` | `0x0400` | Symmetric Pause |
| `ADVERTISE_PAUSE_ASYM` | `0x0800` | Asymmetric Pause |
| `ADVERTISE_RFAULT` | `0x2000` | اكتشاف remote fault |
| `ADVERTISE_LPACK` | `0x4000` | تأكيد استلام إعلان الـ partner |
| `LPA_LPACK` | `0x4000` | الـ partner أكّد |
| `LPA_DUPLEX` | combo | مجموع bits الـ full-duplex |

---

#### ADVERTISE الخاصة بـ 1000BASE-X (على MII_ADVERTISE / MII_LPA)

| Constant | Mask | المعنى |
|---|---|---|
| `ADVERTISE_1000XFULL` | `0x0020` | 1000BASE-X full-duplex |
| `ADVERTISE_1000XHALF` | `0x0040` | 1000BASE-X half-duplex |
| `ADVERTISE_1000XPAUSE` | `0x0080` | 1000BASE-X pause |
| `ADVERTISE_1000XPSE_ASYM` | `0x0100` | 1000BASE-X asymmetric pause |

> ملاحظة: نفس bits الـ ADVERTISE_10HALF/FULL بس السياق مختلف (1000BASE-X)

---

#### MII_CTRL1000 / MII_STAT1000 Bits

| Constant | Mask | المعنى |
|---|---|---|
| `ADVERTISE_1000HALF` | `0x0100` | إعلان 1000BASE-T half-duplex |
| `ADVERTISE_1000FULL` | `0x0200` | إعلان 1000BASE-T full-duplex |
| `CTL1000_PREFER_MASTER` | `0x0400` | تفضيل دور الـ master |
| `CTL1000_AS_MASTER` | `0x0800` | محاولة أن يكون master |
| `CTL1000_ENABLE_MASTER` | `0x1000` | إجبار دور الـ master |
| `LPA_1000HALF` | `0x0400` | الـ partner يدعم 1000M half |
| `LPA_1000FULL` | `0x0800` | الـ partner يدعم 1000M full |
| `LPA_1000MSFAIL` | `0x8000` | فشل Master/Slave negotiation |

---

#### Flow Control Flags

| Constant | Value | المعنى |
|---|---|---|
| `FLOW_CTRL_TX` | `0x01` | إرسال PAUSE frames |
| `FLOW_CTRL_RX` | `0x02` | استقبال PAUSE frames |

---

#### MMD Access Control Bits

| Constant | Value | المعنى |
|---|---|---|
| `MII_MMD_CTRL_DEVAD_MASK` | `0x1f` | رقم الـ MMD device |
| `MII_MMD_CTRL_ADDR` | `0x0000` | Address mode |
| `MII_MMD_CTRL_NOINCR` | `0x4000` | بدون auto-increment |
| `MII_MMD_CTRL_INCR_RDWT` | `0x8000` | increment بعد read/write |
| `MII_MMD_CTRL_INCR_ON_WT` | `0xC000` | increment بعد write بس |

---

### الـ Structs المهمة

#### 1. `struct mii_if_info`

**الهدف:** الـ struct الرئيسي في الملف — بيمثّل واجهة الـ MII بين الـ network driver والـ PHY. الـ driver يحتفظ بنسخة منه ويمرّرها لكل functions الـ MII API.

```c
struct mii_if_info {
    int phy_id;          /* عنوان الـ PHY على الـ MDIO bus (0–31) */
    int advertising;     /* capabilities اللي الـ driver عايز يعلنها */
    int phy_id_mask;     /* mask لفلترة عنوان الـ PHY في الـ ioctl */
    int reg_num_mask;    /* mask لأرقام الـ registers المقبولة */

    unsigned int full_duplex : 1;   /* هل الـ link شغال full-duplex؟ */
    unsigned int force_media : 1;   /* هل autoneg معطّل (forced speed)؟ */
    unsigned int supports_gmii : 1; /* هل الـ PHY بيدعم GMII (1000M)؟ */

    struct net_device *dev;         /* الـ network device المرتبط */

    /* function pointers للـ MDIO read/write */
    int  (*mdio_read) (struct net_device *dev, int phy_id, int location);
    void (*mdio_write)(struct net_device *dev, int phy_id, int location, int val);
};
```

**الـ Fields الأساسية:**

| Field | النوع | الوظيفة |
|---|---|---|
| `phy_id` | `int` | عنوان الـ PHY على الـ MDIO bus |
| `advertising` | `int` | bitfield بقدرات الـ PHY المعلنة (ethtool style) |
| `phy_id_mask` | `int` | للتحقق من صلاحية الـ phy_id في الـ ioctl |
| `reg_num_mask` | `int` | للتحقق من صلاحية رقم الـ register |
| `full_duplex` | `1-bit` | حالة الـ duplex الحالية |
| `force_media` | `1-bit` | إذا كان `1` → autoneg معطّل |
| `supports_gmii` | `1-bit` | دعم 1000M (GMII) |
| `dev` | `struct net_device *` | الـ device المالك |
| `mdio_read` | `fn ptr` | callback لقراءة register من الـ PHY |
| `mdio_write` | `fn ptr` | callback لكتابة register في الـ PHY |

---

#### 2. `struct mii_ioctl_data`

**الهدف:** نقل بيانات الـ ioctl بين الـ userspace والـ kernel عند استخدام `SIOCGMIIPHY` / `SIOCGMIIREG` / `SIOCSMIIREG`.

```c
struct mii_ioctl_data {
    __u16 phy_id;   /* عنوان الـ PHY المستهدف */
    __u16 reg_num;  /* رقم الـ register */
    __u16 val_in;   /* القيمة المراد كتابتها */
    __u16 val_out;  /* القيمة المقروءة من الـ PHY */
};
```

| Field | الوظيفة |
|---|---|
| `phy_id` | عنوان الـ PHY (0–31) على الـ MDIO bus |
| `reg_num` | رقم الـ MII register (0x00–0x1f وما بعده) |
| `val_in` | القيمة اللي الـ userspace بيبعتها للكتابة |
| `val_out` | القيمة اللي الـ kernel بيرجّعها بعد القراءة |

---

### علاقات الـ Structs — ASCII Diagram

```
 Userspace (ethtool / ioctl)
          │
          │  struct ifreq
          ▼
 ┌─────────────────────────┐
 │    struct mii_ioctl_data│  ◄── if_mii(rq) بيحوّل الـ ifreq
 │  phy_id, reg_num,       │
 │  val_in, val_out        │
 └──────────┬──────────────┘
            │
            │  generic_mii_ioctl()
            ▼
 ┌──────────────────────────────────────────────┐
 │            struct mii_if_info                │
 │  phy_id  advertising  phy_id_mask            │
 │  full_duplex  force_media  supports_gmii     │
 │                                              │
 │  dev ──────────────► struct net_device       │
 │                                              │
 │  mdio_read() ──────► driver-specific func    │
 │  mdio_write() ─────► driver-specific func    │
 └──────────────────────────────────────────────┘
            │
            │  mdio_read/write callbacks
            ▼
 ┌──────────────────────┐
 │   MDIO Bus / PHY HW  │
 │  Registers 0x00-0x1f │
 └──────────────────────┘
```

---

### Lifecycle Diagram — دورة حياة الـ MII Interface

```
┌─────────────────────────────────────────────────────────────────┐
│                        Driver Probe                             │
│                                                                 │
│  1. allocate/embed mii_if_info inside driver private struct     │
│  2. mii.phy_id      = detect PHY address (scan MDIO bus)        │
│  3. mii.dev         = netdev                                    │
│  4. mii.mdio_read   = my_driver_mdio_read                       │
│  5. mii.mdio_write  = my_driver_mdio_write                      │
│  6. mii.supports_gmii = mii_check_gmii_support(&mii)           │
│  7. mii.advertising = ADVERTISED_10baseT_Half | ... (defaults)  │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Interface UP (ndo_open)                     │
│                                                                 │
│  8. mii_nway_restart(&mii)   ← يكتب BMCR_ANRESTART في BMCR    │
│     أو                                                          │
│     mii_bmcr_encode_fixed(speed, duplex) ← forced mode         │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Runtime / Link Monitoring                     │
│                                                                 │
│  9. Timer / interrupt يستدعي:                                   │
│     mii_check_link(&mii)    ← يتحقق من BMSR_LSTATUS            │
│     أو                                                          │
│     mii_check_media(&mii, 1, 0) ← يتحقق + يعدّل duplex/speed   │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                   ethtool / ioctl Calls                         │
│                                                                 │
│  mii_ethtool_gset / mii_ethtool_get_link_ksettings             │
│  mii_ethtool_sset / mii_ethtool_set_link_ksettings             │
│  generic_mii_ioctl   ← SIOCGMIIPHY / SIOCGMIIREG / SIOCSMIIREG │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Interface DOWN / Driver Remove                │
│                                                                 │
│  مفيش teardown خاص بـ mii_if_info —                            │
│  الـ struct جزء من الـ driver private data يتحرر معها           │
└─────────────────────────────────────────────────────────────────┘
```

---

### Call Flow Diagrams

#### 1. قراءة MII Register عبر ioctl (SIOCGMIIREG)

```
userspace: ioctl(sock, SIOCGMIIREG, &ifreq)
  │
  ▼
kernel: dev->ndo_do_ioctl(dev, ifreq, SIOCGMIIREG)
  │
  ▼
driver: generic_mii_ioctl(&mii_if, mii_data, SIOCGMIIREG, NULL)
  │
  ├── validate phy_id against mii_if->phy_id_mask
  ├── validate reg_num against mii_if->reg_num_mask
  │
  ▼
mii_if->mdio_read(mii_if->dev, mii_data->phy_id, mii_data->reg_num)
  │
  ▼
driver MDIO function → write to MDIO bus hardware registers
  │
  ▼
mii_data->val_out = result
  │
  ▼
userspace receives the register value in ifreq
```

---

#### 2. mii_check_media — كشف تغيّر الـ link

```
timer/interrupt fires
  │
  ▼
mii_check_media(&mii, ok_to_print=1, init_media=0)
  │
  ├── mii->mdio_read(dev, phy_id, MII_BMSR)
  │     → read BMSR once (لازم 2 reads لأن BMSR latched)
  ├── mii->mdio_read(dev, phy_id, MII_BMSR)  ← second read
  │
  ├── check BMSR_LSTATUS
  │     ├── link up?
  │     │     ├── mii->mdio_read(MII_ADVERTISE)
  │     │     ├── mii->mdio_read(MII_LPA)
  │     │     ├── negotiated = ADVERTISE & LPA
  │     │     ├── mii_nway_result(negotiated) → best speed
  │     │     ├── mii_duplex(force, negotiated) → full/half
  │     │     └── netif_carrier_on(dev) + printk if ok_to_print
  │     │
  │     └── link down?
  │           └── netif_carrier_off(dev)
  │
  └── return: did duplex change?
```

---

#### 3. mii_nway_result — أولوية IEEE 802.3u

```
negotiated = MII_ADVERTISE & MII_LPA  (bits مشتركة)
  │
  ▼
mii_nway_result(negotiated)
  │
  ├── LPA_100FULL set? → return LPA_100FULL   (أعلى أولوية)
  ├── LPA_100BASE4 set? → return LPA_100BASE4
  ├── LPA_100HALF set? → return LPA_100HALF
  ├── LPA_10FULL set? → return LPA_10FULL
  └── else → return LPA_10HALF               (أدنى أولوية)
```

---

#### 4. Flow Control Resolution — IEEE 802.3-2005 Table 28B-3

```
mii_resolve_flowctrl_fdx(lcladv, rmtadv)
  │
  ├── lcladv & rmtadv & PAUSE_CAP ?
  │     → TX + RX pause  (symmetric)
  │
  ├── lcladv & rmtadv & PAUSE_ASYM ?
  │     ├── lcladv & PAUSE_CAP ?
  │     │     → RX only  (local can receive, remote sends)
  │     └── rmtadv & PAUSE_CAP ?
  │           → TX only  (local sends, remote receives)
  │
  └── else → no flow control
```

---

#### 5. Advertisement Translation — ethtool ↔ MII ↔ linkmode

```
ethtool_adv_to_mii_adv_t(ethadv)
  │  ADVERTISED_10baseT_Half → ADVERTISE_10HALF
  │  ADVERTISED_100baseT_Full → ADVERTISE_100FULL
  │  ADVERTISED_Pause → ADVERTISE_PAUSE_CAP
  │  ...
  └── returns u32 for MII_ADVERTISE register

mii_adv_to_ethtool_adv_t(adv)      ← الاتجاه العكسي
  └── returns ethtool ADVERTISED_* bits

linkmode_adv_to_mii_adv_t(advertising)   ← من linkmode bitmap
  │  linkmode_test_bit(10baseT_Half_BIT) → ADVERTISE_10HALF
  │  ...
  └── returns u32 for MII_ADVERTISE register

mii_adv_to_linkmode_adv_t(adv)     ← لـ linkmode bitmap
  │  linkmode_zero(advertising)   ← clear أولاً
  └── mii_adv_mod_linkmode_adv_t() ← ثم set الـ bits
```

---

### Locking Strategy — استراتيجية الـ Locking

**الـ `mii.h` نفسه ما يعرّفش locks** — هو API خفيف جداً بدون state داخلي مُعقّد. مسؤولية الـ locking على الـ driver اللي بيستخدمه.

#### الـ Locking Rules العملية

| السياق | الحل المعتاد |
|---|---|
| قراءة/كتابة `mii_if_info` fields | `spin_lock_irqsave` أو `mutex` في الـ driver |
| استدعاء `mdio_read/write` | الـ driver مسؤول — ممكن يكون `mutex` لأن MDIO bus slow |
| `mii_check_link` من interrupt | الـ driver بيحمي بـ `spin_lock_irqsave` |
| `mii_check_media` من timer | الـ timer context → `spin_lock_bh` أو `spin_lock_irqsave` |
| الـ ethtool callbacks | الـ `rtnl_lock` بيكون مضمون من الـ ethtool core |
| الـ ioctl | الـ `rtnl_lock` بيكون مضمون من `dev_ioctl` |

#### مثال واقعي — driver يحمي MDIO access

```c
/* الـ driver يعرّف lock خاص بالـ MDIO bus */
struct my_driver_priv {
    struct mii_if_info mii;
    spinlock_t mdio_lock;   /* يحمي الـ mdio_read/write */
    /* ... */
};

static int my_mdio_read(struct net_device *dev, int phy_id, int reg)
{
    struct my_driver_priv *priv = netdev_priv(dev);
    unsigned long flags;
    int val;

    spin_lock_irqsave(&priv->mdio_lock, flags);
    /* access MDIO hardware */
    val = hw_read_mdio(phy_id, reg);
    spin_unlock_irqrestore(&priv->mdio_lock, flags);

    return val;
}
```

#### Lock Ordering (ترتيب الـ Locks عند التداخل)

```
rtnl_lock                    ← الأعلى (من ethtool/ioctl)
  └── driver->mdio_lock      ← الأدنى (من داخل الـ callbacks)

لا يجوز عكس الترتيب ← deadlock
```

#### ملاحظة على `mii_check_media`

**الـ `mii_check_media`** بتكتب `mii->full_duplex` → field مشترك بين الـ timer/interrupt وبين الـ ethtool. الـ driver لازم يحمي هذا الـ field بـ lock موحّد.

```
Timer context:
  spin_lock_irqsave(&priv->lock)
    mii_check_media(&priv->mii, ...)   ← يعدّل mii->full_duplex
  spin_unlock_irqrestore(&priv->lock)

ethtool gset:
  spin_lock_irqsave(&priv->lock)
    mii_ethtool_gset(&priv->mii, cmd)  ← يقرأ mii->full_duplex
  spin_unlock_irqrestore(&priv->lock)
```

---

### ملخص العلاقات الكاملة

```
┌──────────────┐     embeds      ┌────────────────────┐
│ driver priv  │────────────────►│  mii_if_info       │
│ struct       │                 │  .phy_id           │
│              │                 │  .advertising      │
│  mdio_lock   │◄── protects ───►│  .full_duplex      │
│  net_device  │◄── .dev ────────│  .mdio_read()      │
└──────────────┘                 │  .mdio_write()     │
                                 └────────┬───────────┘
                                          │
                          ┌───────────────┼────────────────┐
                          │               │                │
                          ▼               ▼                ▼
                   mii_check_link  mii_ethtool_*   generic_mii_ioctl
                          │               │                │
                          └───────────────┴────────────────┘
                                          │
                                          ▼
                               mii_if_info->mdio_read/write
                                          │
                                          ▼
                                  MDIO Bus Hardware
                                  PHY Registers (0x00-0x1f)
```
## Phase 4: شرح الـ Functions

---

### ملخص الـ Functions — Cheatsheet

#### Functions الـ Non-inline (exported من `mii.c`)

| Function | الغرض |
|---|---|
| `mii_link_ok()` | قراءة BMSR وتحديد إذا كان الـ link up |
| `mii_nway_restart()` | إعادة تشغيل الـ autonegotiation |
| `mii_ethtool_gset()` | تعبئة `ethtool_cmd` بالإعدادات الحالية (legacy) |
| `mii_ethtool_get_link_ksettings()` | تعبئة `ethtool_link_ksettings` (modern API) |
| `mii_ethtool_sset()` | تطبيق إعدادات ethtool على الـ PHY (legacy) |
| `mii_ethtool_set_link_ksettings()` | تطبيق إعدادات ethtool على الـ PHY (modern) |
| `mii_check_gmii_support()` | التحقق من دعم GMII / Gigabit |
| `mii_check_link()` | تحقق من حالة الـ link وإطلاق `netif_carrier_*` |
| `mii_check_media()` | تحقق كامل من الـ media وإرجاع تغيير الـ duplex |
| `generic_mii_ioctl()` | معالج عام لـ SIOCGMIIPHY/SIOCGMIIREG/SIOCSMIIREG |

#### Functions الـ Inline (helpers في الـ header)

| Function | الغرض |
|---|---|
| `if_mii()` | استخراج `mii_ioctl_data` من `ifreq` |
| `mii_nway_result()` | تحديد أعلى speed/duplex متفاوض عليه |
| `mii_duplex()` | هل الـ link full-duplex؟ |
| `ethtool_adv_to_mii_adv_t()` | ethtool → MII_ADVERTISE bits (100/10BaseT) |
| `linkmode_adv_to_mii_adv_t()` | linkmode → MII_ADVERTISE bits (100/10BaseT) |
| `mii_adv_to_ethtool_adv_t()` | MII_ADVERTISE → ethtool bits |
| `ethtool_adv_to_mii_ctrl1000_t()` | ethtool → MII_CTRL1000 bits (1000BaseT) |
| `linkmode_adv_to_mii_ctrl1000_t()` | linkmode → MII_CTRL1000 bits (1000BaseT) |
| `mii_ctrl1000_to_ethtool_adv_t()` | MII_CTRL1000 → ethtool bits |
| `mii_lpa_to_ethtool_lpa_t()` | MII_LPA → ethtool LP bits |
| `mii_stat1000_to_ethtool_lpa_t()` | MII_STAT1000 → ethtool LP bits |
| `mii_stat1000_mod_linkmode_lpa_t()` | MII_STAT1000 → linkmode (modify in-place) |
| `ethtool_adv_to_mii_adv_x()` | ethtool → MII_CTRL1000 bits (1000BaseX) |
| `mii_adv_to_ethtool_adv_x()` | MII_CTRL1000 → ethtool bits (1000BaseX) |
| `mii_adv_mod_linkmode_adv_t()` | MII_ADVERTISE → linkmode (modify in-place) |
| `mii_adv_to_linkmode_adv_t()` | MII_ADVERTISE → linkmode (zero + fill) |
| `mii_lpa_to_linkmode_lpa_t()` | MII_LPA → linkmode (zero + fill) |
| `mii_lpa_mod_linkmode_lpa_t()` | MII_LPA → linkmode (modify in-place) |
| `mii_ctrl1000_mod_linkmode_adv_t()` | MII_CTRL1000 → linkmode (modify in-place) |
| `linkmode_adv_to_lcl_adv_t()` | linkmode → pause bits فقط |
| `mii_lpa_mod_linkmode_x()` | MII_LPA (1000BaseX) → linkmode |
| `linkmode_adv_to_mii_adv_x()` | linkmode → MII_CTRL1000 bits (1000BaseX) |
| `mii_advertise_flowctrl()` | flow control caps → ADVERTISE pause bits |
| `mii_resolve_flowctrl_fdx()` | تحديد الـ TX/RX pause بعد autoneg |
| `mii_bmcr_encode_fixed()` | speed + duplex → BMCR register value |

---

### المجموعة 1: Link Status & Control

هذه المجموعة بتتعامل مع قراءة حالة الـ link وإعادة تشغيل الـ autoneg. كل function فيها بتقرأ أو بتكتب على الـ PHY registers عبر الـ `mdio_read`/`mdio_write` callbacks المخزنة في `mii_if_info`.

---

#### `mii_link_ok`

```c
extern int mii_link_ok(struct mii_if_info *mii);
```

**الـ function** بتقرأ الـ Basic Mode Status Register (BMSR) عبر `mdio_read` وبتبص على bit الـ `BMSR_LSTATUS`. لازم تقرأ الـ BMSR مرتين لأن الـ bit بيكون latched-low (لو انقطع الـ link حتى للحظة بيبقى 0 لحد ما تقراه). في بعض implementations بيقرأها مرة واحدة بس.

**Parameters:**
- `mii` — pointer على `mii_if_info` اللي فيه `mdio_read` callback والـ `phy_id`

**Return Value:**
- `1` إذا الـ link up
- `0` إذا الـ link down

**Key Details:**
- بتُستدعى من سياق sleeping (ممكن تأخذ وقت لو الـ MDIO bus slow)
- مش thread-safe بذاتها — الـ caller مسؤول عن أي locking على الـ MDIO bus
- بتُستدعى من `mii_check_link()` و `mii_check_media()`

---

#### `mii_nway_restart`

```c
extern int mii_nway_restart(struct mii_if_info *mii);
```

**الـ function** بتكتب على الـ BMCR وتـ set الـ `BMCR_ANRESTART` bit لإعادة تشغيل الـ autonegotiation. قبل كده بتتحقق إن الـ `BMCR_ANENABLE` موجود — لو الـ autoneg مش enabled أصلاً بترجع error.

**Parameters:**
- `mii` — pointer على `mii_if_info`

**Return Value:**
- `0` نجاح
- `-EINVAL` لو الـ autoneg مش enabled في الـ BMCR

**Key Details:**
- بتُستدعى من ethtool `nway_reset` handler
- الـ autoneg عملية غير synchronous — الـ function بترجع فوراً والـ hardware بيكمل في الخلفية
- التحقق من اكتمال الـ autoneg يكون عبر polling على `BMSR_ANEGCOMPLETE`

---

#### `mii_check_link`

```c
extern void mii_check_link(struct mii_if_info *mii);
```

**الـ function** بتستدعي `mii_link_ok()` وبناءً على النتيجة بتستدعي `netif_carrier_on()` أو `netif_carrier_off()` على الـ `net_device` المرتبط بالـ `mii_if_info`. بتُستخدم في الـ timer-based polling للـ link state.

**Parameters:**
- `mii` — pointer على `mii_if_info`، بيحتوي على `dev` pointer للـ `net_device`

**Return Value:** void

**Key Details:**
- لا تأخذ أي lock — الـ caller مسؤول
- مناسبة للـ drivers اللي بتعمل polling بدل interrupts
- لو الـ driver بيستخدم phylib، مش محتاج لـ `mii_check_link` لأن phylib بتتولى الموضوع

---

#### `mii_check_media`

```c
extern unsigned int mii_check_media(struct mii_if_info *mii,
                                    unsigned int ok_to_print,
                                    unsigned int init_media);
```

**الـ function** أشمل من `mii_check_link` — بتتحقق من الـ link status وبتقرأ نتيجة الـ autoneg (BMSR, MII_LPA, MII_CTRL1000 لو GMII) وبتحدّث `mii->full_duplex`. لو `init_media` مضبوط بتعمل update حتى لو الـ link state ما اتغيرش.

**Parameters:**
- `mii` — pointer على `mii_if_info`
- `ok_to_print` — لو non-zero بتطبع رسائل `pr_info` عن الـ speed/duplex
- `init_media` — لو non-zero بتعمل forced re-check حتى بدون تغيير في الـ link

**Return Value:**
- non-zero لو الـ duplex setting اتغير
- `0` لو ماحصلش تغيير

**Key Details:**
- بتقرأ `BMSR_ANEGCOMPLETE` — لو الـ autoneg مش خلص بتفضل على الـ current state
- بتحدّث `mii->full_duplex` كـ side effect
- بتستدعي `netif_carrier_on/off` هي كمان

**Pseudocode:**
```
mii_check_media(mii, ok_to_print, init_media):
    link_is_up = mii_link_ok(mii)
    if link_is_up:
        bmsr = mdio_read(BMSR)
        if BMSR_ANEGCOMPLETE in bmsr:
            lpa  = mdio_read(MII_LPA)
            adv  = mdio_read(MII_ADVERTISE)
            negotiated = lpa & adv
            if mii->supports_gmii:
                stat1000 = mdio_read(MII_STAT1000)
                /* check 1000T bits */
            new_full_duplex = mii_duplex(mii->force_media, negotiated)
        duplex_changed = (new_full_duplex != mii->full_duplex)
        mii->full_duplex = new_full_duplex
        netif_carrier_on(dev)
    else:
        netif_carrier_off(dev)
    return duplex_changed
```

---

#### `mii_check_gmii_support`

```c
extern int mii_check_gmii_support(struct mii_if_info *mii);
```

**الـ function** بتقرأ الـ `BMSR` وبتتحقق من `BMSR_ESTATEN` bit — لو موجود بتقرأ `MII_ESTATUS` وبتتحقق من `ESTATUS_1000_TFULL` أو `ESTATUS_1000_THALF` لتحديد دعم الـ Gigabit. بتحدّث `mii->supports_gmii`.

**Parameters:**
- `mii` — pointer على `mii_if_info`

**Return Value:**
- `1` لو الـ PHY يدعم GMII/1000BaseT
- `0` غير ذلك

**Key Details:**
- بتُستدعى مرة واحدة عند الـ probe/init
- نتيجتها بتُخزن في `mii->supports_gmii` لتحديد ما إذا كانت `mii_check_media` تقرأ `MII_CTRL1000`/`MII_STAT1000`

---

### المجموعة 2: ethtool Interface (Legacy `ethtool_cmd`)

هذه المجموعة بتربط الـ MII layer بالـ ethtool API القديم (`ETHTOOL_GSET`/`ETHTOOL_SSET`). الـ `ethtool_cmd` struct كانت الواجهة الأساسية قبل إدخال `ethtool_link_ksettings`.

---

#### `mii_ethtool_gset`

```c
extern void mii_ethtool_gset(struct mii_if_info *mii,
                              struct ethtool_cmd *ecmd);
```

**الـ function** بتقرأ الـ PHY registers وبتحول المعلومات لـ `ethtool_cmd` struct. بتملأ: supported modes، advertised modes، speed، duplex، port، transceiver، autoneg.

**Parameters:**
- `mii` — pointer على `mii_if_info`
- `ecmd` — pointer على `ethtool_cmd` اللي هيتمليء

**Return Value:** void

**Key Details:**
- legacy API — الـ drivers الجديدة المفروض تستخدم `mii_ethtool_get_link_ksettings()`
- بتقرأ `BMSR` لمعرفة الـ supported speeds
- بتستخدم `ethtool_cmd_speed_set()` لضبط الـ speed بشكل صح (32-bit field)
- لا تأخذ lock

---

#### `mii_ethtool_sset`

```c
extern int mii_ethtool_sset(struct mii_if_info *mii,
                             struct ethtool_cmd *ecmd);
```

**الـ function** بتاخد الـ `ethtool_cmd` المطلوب وبتكتب على الـ PHY registers. لو `ecmd->autoneg == AUTONEG_ENABLE` بتكتب على `MII_ADVERTISE` (و`MII_CTRL1000` لو GMII) وبتعمل autoneg restart. لو لا، بتحسب الـ BMCR المناسب وبتكتبه.

**Parameters:**
- `mii` — pointer على `mii_if_info`
- `ecmd` — pointer على `ethtool_cmd` المطلوب تطبيقه

**Return Value:**
- `0` نجاح
- `-EINVAL` لو الـ settings غير صحيحة (مثلاً speed غير مدعوم)

**Key Details:**
- legacy API
- بتُستدعى من `ethtool_set_settings()` في kernel
- الـ force_media flag في `mii_if_info` بيأثر على سلوكها

---

#### `mii_ethtool_get_link_ksettings`

```c
extern void mii_ethtool_get_link_ksettings(
    struct mii_if_info *mii,
    struct ethtool_link_ksettings *cmd);
```

**الـ function** النسخة الحديثة من `mii_ethtool_gset`. بتستخدم `ethtool_link_ksettings` اللي بيستخدم linkmode bitmaps بدل الـ u32 flags القديمة، وبالتالي بيدعم عدد أكبر بكثير من الـ link modes.

**Parameters:**
- `mii` — pointer على `mii_if_info`
- `cmd` — pointer على `ethtool_link_ksettings` اللي هيتمليء

**Return Value:** void

**Key Details:**
- الـ `ethtool_link_ksettings` بيحتوي على arrays من `unsigned long` لـ bitmaps
- بتستخدم helpers زي `mii_adv_to_linkmode_adv_t()` داخلياً
- الـ API الموصى بيه للـ drivers الجديدة

---

#### `mii_ethtool_set_link_ksettings`

```c
extern int mii_ethtool_set_link_ksettings(
    struct mii_if_info *mii,
    const struct ethtool_link_ksettings *cmd);
```

**الـ function** النسخة الحديثة من `mii_ethtool_sset`. بتقرأ الـ `ethtool_link_ksettings` وبتكتب على الـ PHY registers بالإعدادات المطلوبة.

**Parameters:**
- `mii` — pointer على `mii_if_info`
- `cmd` — pointer على `ethtool_link_ksettings` const المطلوب تطبيقه

**Return Value:**
- `0` نجاح
- `-EINVAL` لو الإعدادات غير صحيحة

**Key Details:**
- بتُستخدم مع `ethtool_ops->set_link_ksettings`
- تعامل مع `cmd->base.autoneg` لتحديد الـ mode

---

### المجموعة 3: IOCTL Handler

---

#### `generic_mii_ioctl`

```c
extern int generic_mii_ioctl(struct mii_if_info *mii_if,
                              struct mii_ioctl_data *mii_data,
                              int cmd,
                              unsigned int *duplex_changed);
```

**الـ function** معالج IOCTL عام للـ MII. بتتعامل مع الأوامر التالية:
- `SIOCGMIIPHY` — إرجاع الـ PHY ID
- `SIOCGMIIREG` — قراءة register من الـ PHY
- `SIOCSMIIREG` — كتابة register على الـ PHY (requires `CAP_NET_ADMIN`)

**Parameters:**
- `mii_if` — pointer على `mii_if_info`
- `mii_data` — struct فيه `phy_id`، `reg_num`، `val_in`، `val_out`
- `cmd` — رقم الـ IOCTL command
- `duplex_changed` — output، لو non-NULL بيُضبط لو الـ duplex اتغير نتيجة الكتابة

**Return Value:**
- `0` نجاح
- `-EOPNOTSUPP` لو الـ command مش مدعوم
- `-EPERM` لو محاولة كتابة بدون صلاحيات

**Key Details:**
- لو الـ driver كتب على `BMCR` أو `MII_ADVERTISE`، الـ function بتستدعي `mii_check_media()` وبتحدّث `*duplex_changed`
- بتُستدعى من `net_device_ops->ndo_do_ioctl` أو `ndo_eth_ioctl`
- الـ caller لازم يعمل `copy_from_user` على `mii_ioctl_data` قبل الاستدعاء

**Pseudocode:**
```
generic_mii_ioctl(mii_if, mii_data, cmd, duplex_changed):
    switch cmd:
        case SIOCGMIIPHY:
            mii_data->phy_id = mii_if->phy_id
            return 0

        case SIOCGMIIREG:
            mii_data->val_out = mdio_read(mii_data->phy_id,
                                           mii_data->reg_num)
            return 0

        case SIOCSMIIREG:
            if !capable(CAP_NET_ADMIN): return -EPERM
            mdio_write(mii_data->phy_id, mii_data->reg_num,
                       mii_data->val_in)
            if reg changed is BMCR or ADVERTISE:
                mii_check_media(mii_if, ...)
                if duplex_changed: *duplex_changed = ...
            return 0

        default:
            return -EOPNOTSUPP
```

---

### المجموعة 4: Inline Helpers — `if_mii` و autoneg results

---

#### `if_mii`

```c
static inline struct mii_ioctl_data *if_mii(struct ifreq *rq)
{
    return (struct mii_ioctl_data *) &rq->ifr_ifru;
}
```

**الـ function** بتعمل cast على الـ `ifr_ifru` union field في الـ `ifreq` struct لـ pointer على `mii_ioctl_data`. الـ `ifr_ifru` union كبير بما يكفي لاستيعاب `mii_ioctl_data` (4 x u16).

**Parameters:**
- `rq` — pointer على `ifreq` struct قادم من userspace

**Return Value:** pointer على `mii_ioctl_data` embedded في `ifreq`

**Key Details:**
- zero-overhead — مجرد cast
- بتُستخدم في الـ `ndo_eth_ioctl` handler قبل استدعاء `generic_mii_ioctl()`

---

#### `mii_nway_result`

```c
static inline unsigned int mii_nway_result(unsigned int negotiated)
```

**الـ function** بتاخد نتيجة AND بين `MII_ADVERTISE` و`MII_LPA` (الـ capabilities المشتركة) وبترجع أعلى mode متفاوض عليه حسب أولوية IEEE 802.3u.

**Parameters:**
- `negotiated` — `MII_ADVERTISE & MII_LPA` (الـ common capabilities)

**Return Value:** أحد الـ `LPA_xxx` constants للـ mode الأعلى أولوية:

```
LPA_100FULL > LPA_100BASE4 > LPA_100HALF > LPA_10FULL > LPA_10HALF
```

**Key Details:**
- لا تتعامل مع 1000BaseT — للـ Gigabit لازم تبص على `MII_STAT1000` منفصلاً
- ترتيب الأولوية مطابق لـ IEEE 802.3u table باستثناء أن 100BASE4 جاي بعد 100FULL وليس قبله

---

#### `mii_duplex`

```c
static inline unsigned int mii_duplex(unsigned int duplex_lock,
                                       unsigned int negotiated)
```

**الـ function** بتحدد إذا كان الـ link شغال full-duplex. لو `duplex_lock` مضبوط (يعني الـ autoneg معطل وهو forced full-duplex) بترجع 1 مباشرةً. غير كده بتستخدم `mii_nway_result()` وبتبص على `LPA_DUPLEX` mask.

**Parameters:**
- `duplex_lock` — non-zero لو الـ duplex forced (من `mii_if_info->full_duplex` مع `force_media`)
- `negotiated` — `ADVERTISE & LPA`

**Return Value:**
- `1` = full-duplex
- `0` = half-duplex

---

### المجموعة 5: Advertisement Translation — ethtool Legacy ↔ MII Registers

هذه المجموعة هي القلب التحويلي للـ header. الـ kernel لديه 3 representations للـ link capabilities:
1. **MII register bits** — الـ raw PHY register values
2. **ethtool `ADVERTISED_*` flags** — u32 bitmask قديم
3. **linkmode** — `unsigned long` bitmap حديث قابل للتوسع

الـ functions دي بتترجم بين الأشكال التلاتة.

---

#### `ethtool_adv_to_mii_adv_t`

```c
static inline u32 ethtool_adv_to_mii_adv_t(u32 ethadv)
```

**الـ function** بتحول الـ `ADVERTISED_*` flags للـ bits المناسبة في `MII_ADVERTISE` register لوضع 10/100BaseT.

**Parameters:**
- `ethadv` — u32 من `ADVERTISED_10baseT_Half`, `ADVERTISED_10baseT_Full`, `ADVERTISED_100baseT_Half`, `ADVERTISED_100baseT_Full`, `ADVERTISED_Pause`, `ADVERTISED_Asym_Pause`

**Return Value:** u32 يُكتب في `MII_ADVERTISE`

| Input bit | Output bit |
|---|---|
| `ADVERTISED_10baseT_Half` | `ADVERTISE_10HALF` |
| `ADVERTISED_10baseT_Full` | `ADVERTISE_10FULL` |
| `ADVERTISED_100baseT_Half` | `ADVERTISE_100HALF` |
| `ADVERTISED_100baseT_Full` | `ADVERTISE_100FULL` |
| `ADVERTISED_Pause` | `ADVERTISE_PAUSE_CAP` |
| `ADVERTISED_Asym_Pause` | `ADVERTISE_PAUSE_ASYM` |

---

#### `linkmode_adv_to_mii_adv_t`

```c
static inline u32 linkmode_adv_to_mii_adv_t(const unsigned long *advertising)
```

**الـ function** نفس `ethtool_adv_to_mii_adv_t` لكنها تقرأ من الـ linkmode bitmap بدل الـ u32 القديم. بتستخدم `linkmode_test_bit()` (وهو macro على `test_bit()`) لفحص كل bit.

**Parameters:**
- `advertising` — pointer على `unsigned long` bitmap (minimum size = `__ETHTOOL_LINK_MODE_MASK_NBITS` bits)

**Return Value:** u32 يُكتب في `MII_ADVERTISE`

---

#### `mii_adv_to_ethtool_adv_t`

```c
static inline u32 mii_adv_to_ethtool_adv_t(u32 adv)
```

**الـ function** عكس `ethtool_adv_to_mii_adv_t` — بتحول من `MII_ADVERTISE` register للـ `ADVERTISED_*` flags.

**Parameters:**
- `adv` — قيمة `MII_ADVERTISE` register

**Return Value:** u32 من `ADVERTISED_*` flags

---

#### `ethtool_adv_to_mii_ctrl1000_t`

```c
static inline u32 ethtool_adv_to_mii_ctrl1000_t(u32 ethadv)
```

**الـ function** بتحول الـ `ADVERTISED_1000baseT_*` flags للـ bits في `MII_CTRL1000` register. بتُستخدم لـ Gigabit PHYs اللي بتدعم 1000BaseT.

**Parameters:**
- `ethadv` — u32 من `ADVERTISED_1000baseT_Half` و`ADVERTISED_1000baseT_Full`

**Return Value:** u32 يُكتب في `MII_CTRL1000`

| Input | Output |
|---|---|
| `ADVERTISED_1000baseT_Half` | `ADVERTISE_1000HALF` |
| `ADVERTISED_1000baseT_Full` | `ADVERTISE_1000FULL` |

---

#### `linkmode_adv_to_mii_ctrl1000_t`

```c
static inline u32 linkmode_adv_to_mii_ctrl1000_t(const unsigned long *advertising)
```

**الـ function** نفس `ethtool_adv_to_mii_ctrl1000_t` لكن من linkmode bitmap. للـ 1000BaseT mode.

**Parameters:**
- `advertising` — linkmode bitmap pointer

**Return Value:** u32 يُكتب في `MII_CTRL1000`

---

#### `mii_ctrl1000_to_ethtool_adv_t`

```c
static inline u32 mii_ctrl1000_to_ethtool_adv_t(u32 adv)
```

**الـ function** عكس `ethtool_adv_to_mii_ctrl1000_t` — بتقرأ `MII_CTRL1000` وبترجع `ADVERTISED_1000baseT_*` flags.

**Parameters:**
- `adv` — قيمة `MII_CTRL1000` register

**Return Value:** u32 من `ADVERTISED_1000baseT_*` flags

---

### المجموعة 6: LPA (Link Partner Ability) Translation

الـ `MII_LPA` register بيحتوي على الـ capabilities اللي الـ link partner (الطرف التاني في الـ cable) أعلن عنها في الـ autoneg. الـ functions دي بتقرأ الـ LPA وبتحوله.

---

#### `mii_lpa_to_ethtool_lpa_t`

```c
static inline u32 mii_lpa_to_ethtool_lpa_t(u32 lpa)
```

**الـ function** بتحول `MII_LPA` register للـ ethtool LP advertisement flags. بتشمل كمان `ADVERTISED_Autoneg` لو الـ `LPA_LPACK` bit مضبوط (يعني الـ link partner ack الـ autoneg).

**Parameters:**
- `lpa` — قيمة `MII_LPA` register

**Return Value:** u32 من `ADVERTISED_*` flags تمثل capabilities الـ link partner

**Key Details:**
- داخلياً بتستدعي `mii_adv_to_ethtool_adv_t(lpa)` وبتضيف `ADVERTISED_Autoneg` لو `LPA_LPACK`
- الفرق بين الـ `mii_adv_*` و `mii_lpa_*` هو إضافة الـ `LPA_LPACK` handling

---

#### `mii_stat1000_to_ethtool_lpa_t`

```c
static inline u32 mii_stat1000_to_ethtool_lpa_t(u32 lpa)
```

**الـ function** بتحول `MII_STAT1000` register (اللي فيه الـ Gigabit LP capabilities) للـ ethtool LP flags.

**Parameters:**
- `lpa` — قيمة `MII_STAT1000` register

**Return Value:** u32 من `ADVERTISED_1000baseT_*`

| Input bit | Output |
|---|---|
| `LPA_1000HALF` | `ADVERTISED_1000baseT_Half` |
| `LPA_1000FULL` | `ADVERTISED_1000baseT_Full` |

---

#### `mii_stat1000_mod_linkmode_lpa_t`

```c
static inline void mii_stat1000_mod_linkmode_lpa_t(unsigned long *advertising,
                                                    u32 lpa)
```

**الـ function** بتحدث الـ linkmode bitmap بالـ 1000BaseT LP capabilities من `MII_STAT1000`. بتستخدم `linkmode_mod_bit` (وهو `__assign_bit`) — يعني بتضبط أو تمسح الـ bit حسب قيمة الـ `lpa`. لا تمسح الـ bits التانية في الـ bitmap.

**Parameters:**
- `advertising` — pointer على linkmode bitmap اللي هيتعدل
- `lpa` — قيمة `MII_STAT1000`

**Return Value:** void (in-place modification)

**Key Details:**
- الفرق بينها وبين `mii_adv_to_linkmode_adv_t` إنها بتـ modify in-place بدون `linkmode_zero()`
- مهمة لما بيكون عندك bits تانية في الـ advertising مش عايز تمسحها

---

### المجموعة 7: 1000Base-X Advertisement Translation

الـ 1000Base-X (fiber/SFP) بيستخدم register format مختلف عن 1000Base-T (copper). الـ bits في `MII_ADVERTISE` بيكون ليها معاني مختلفة في الـ X mode.

---

#### `ethtool_adv_to_mii_adv_x`

```c
static inline u32 ethtool_adv_to_mii_adv_x(u32 ethadv)
```

**الـ function** بتحول الـ ethtool advertisement flags للـ `MII_CTRL1000` register bits في وضع 1000Base-X. في الـ X mode، نفس register bits بيكون ليها معاني مختلفة:

| Input | Output |
|---|---|
| `ADVERTISED_1000baseT_Half` | `ADVERTISE_1000XHALF` |
| `ADVERTISED_1000baseT_Full` | `ADVERTISE_1000XFULL` |
| `ADVERTISED_Pause` | `ADVERTISE_1000XPAUSE` |
| `ADVERTISED_Asym_Pause` | `ADVERTISE_1000XPSE_ASYM` |

**Parameters:**
- `ethadv` — u32 من `ADVERTISED_*`

**Return Value:** u32 يُكتب في `MII_ADVERTISE` (في X mode)

---

#### `mii_adv_to_ethtool_adv_x`

```c
static inline u32 mii_adv_to_ethtool_adv_x(u32 adv)
```

**الـ function** عكس `ethtool_adv_to_mii_adv_x` — بتحول `MII_ADVERTISE` bits في 1000Base-X mode للـ ethtool flags.

**Parameters:**
- `adv` — قيمة `MII_ADVERTISE` في X mode

**Return Value:** u32 من `ADVERTISED_*`

---

#### `mii_lpa_mod_linkmode_x`

```c
static inline void mii_lpa_mod_linkmode_x(unsigned long *linkmodes,
                                           u16 lpa, int fd_bit)
```

**الـ function** بتحول الـ `MII_LPA` (config_reg) في 1000Base-X mode للـ linkmode bitmap. الـ `fd_bit` بيتحدد من الـ caller لأن الـ full-duplex bit بيختلف حسب نوع الـ X interface (1000BASE-X، SGMII، إلخ).

**Parameters:**
- `linkmodes` — pointer على linkmode bitmap اللي هيتعدل in-place
- `lpa` — قيمة الـ config_reg من الـ link partner (u16)
- `fd_bit` — رقم الـ bit في الـ linkmode bitmap اللي يمثل full-duplex لهذا الـ interface type

**Return Value:** void

**Key Details:**
- بتضبط: `Autoneg_BIT`، `Pause_BIT`، `Asym_Pause_BIT`، و `fd_bit`
- الـ `fd_bit` المرن يخلي الـ function صالحة لـ 1000BASE-X و2500BASE-X وغيرها
- بتستخدم `LPA_1000XPAUSE` و`LPA_1000XPAUSE_ASYM` و`LPA_1000XFULL`

---

#### `linkmode_adv_to_mii_adv_x`

```c
static inline u16 linkmode_adv_to_mii_adv_x(const unsigned long *linkmodes,
                                              int fd_bit)
```

**الـ function** عكس `mii_lpa_mod_linkmode_x` — بتحول من linkmode bitmap للـ MII config_reg bits في 1000Base-X mode. الـ `fd_bit` نفس مفهوم المرونة.

**Parameters:**
- `linkmodes` — pointer على linkmode bitmap const
- `fd_bit` — bit index في الـ linkmode للـ full-duplex

**Return Value:** u16 يُكتب في `MII_ADVERTISE` (X mode)

---

### المجموعة 8: Linkmode ↔ MII In-Place Modifiers

هذه المجموعة بتتعامل مع الـ linkmode representation الحديثة وبتعمل modify in-place بدل ما تزود الـ bitmap الكاملة.

---

#### `mii_adv_mod_linkmode_adv_t`

```c
static inline void mii_adv_mod_linkmode_adv_t(unsigned long *advertising,
                                               u32 adv)
```

**الـ function** بتحول `MII_ADVERTISE` bits للـ linkmode وبتعدل الـ bitmap in-place (بدون `linkmode_zero()`). بتضبط أو تمسح 6 bits: 10H, 10F, 100H, 100F, Pause, Asym_Pause.

**Parameters:**
- `advertising` — pointer على linkmode bitmap
- `adv` — قيمة `MII_ADVERTISE`

**Return Value:** void

**Key Details:**
- كل `linkmode_mod_bit` = `__assign_bit(bit, addr, value)` — بتضبط الـ bit لو value غير صفر وبتمسحه لو صفر
- مناسبة لما عايز تحدث بعض الـ bits وتسيب الباقي

---

#### `mii_adv_to_linkmode_adv_t`

```c
static inline void mii_adv_to_linkmode_adv_t(unsigned long *advertising,
                                              u32 adv)
```

**الـ function** زي `mii_adv_mod_linkmode_adv_t` لكن بتعمل `linkmode_zero(advertising)` الأول. يعني بتمسح الـ bitmap كامل وبعدين بتملأ الـ bits المناسبة.

**Parameters:**
- `advertising` — pointer على linkmode bitmap (هيتمسح أولاً)
- `adv` — قيمة `MII_ADVERTISE`

**Return Value:** void

**Key Details:**
- استخدمها لو عايز تعمل full reset للـ advertising من الـ PHY register
- استخدم `mii_adv_mod_linkmode_adv_t` لو عايز تحافظ على الـ bits الأخرى (زي الـ 1000BaseT bits)

---

#### `mii_lpa_to_linkmode_lpa_t`

```c
static inline void mii_lpa_to_linkmode_lpa_t(unsigned long *lp_advertising,
                                              u32 lpa)
```

**الـ function** بتحول `MII_LPA` للـ linkmode bitmap مع zero أول. بتشمل `Autoneg_BIT` لو `LPA_LPACK`. الـ full replacement.

**Parameters:**
- `lp_advertising` — pointer على linkmode bitmap (هيتمسح ويتملأ)
- `lpa` — قيمة `MII_LPA`

**Return Value:** void

---

#### `mii_lpa_mod_linkmode_lpa_t`

```c
static inline void mii_lpa_mod_linkmode_lpa_t(unsigned long *lp_advertising,
                                               u32 lpa)
```

**الـ function** نفس `mii_lpa_to_linkmode_lpa_t` لكن بدون zero — modify in-place. بتضبط أو تمسح الـ bits حسب قيمة `lpa`.

**Parameters:**
- `lp_advertising` — pointer على linkmode bitmap اللي هيتعدل
- `lpa` — قيمة `MII_LPA`

**Return Value:** void

---

#### `mii_ctrl1000_mod_linkmode_adv_t`

```c
static inline void mii_ctrl1000_mod_linkmode_adv_t(unsigned long *advertising,
                                                    u32 ctrl1000)
```

**الـ function** بتحدث الـ 1000BaseT bits في الـ linkmode bitmap من `MII_CTRL1000` register value. modify in-place.

**Parameters:**
- `advertising` — pointer على linkmode bitmap
- `ctrl1000` — قيمة `MII_CTRL1000`

**Return Value:** void

| Input bit | Linkmode bit |
|---|---|
| `ADVERTISE_1000HALF` | `1000baseT_Half_BIT` |
| `ADVERTISE_1000FULL` | `1000baseT_Full_BIT` |

---

### المجموعة 9: Flow Control

---

#### `linkmode_adv_to_lcl_adv_t`

```c
static inline u32 linkmode_adv_to_lcl_adv_t(const unsigned long *advertising)
```

**الـ function** بتستخرج فقط الـ pause bits من الـ linkmode وبتحولهم لـ `ADVERTISE_PAUSE_*` bits. مخصوصة للـ local pause capability extraction.

**Parameters:**
- `advertising` — linkmode bitmap

**Return Value:** u32 فيه `ADVERTISE_PAUSE_CAP` و/أو `ADVERTISE_PAUSE_ASYM`

**Key Details:**
- بتُستخدم قبل `mii_resolve_flowctrl_fdx()` لاستخراج الـ local capabilities

---

#### `mii_advertise_flowctrl`

```c
static inline u16 mii_advertise_flowctrl(int cap)
```

**الـ function** بتحسب الـ `ADVERTISE_PAUSE_*` bits المناسبة للإعلان عنها في `MII_ADVERTISE` بناءً على الـ flow control capabilities المطلوبة.

الـ encoding معقد شوية لأن الـ IEEE 802.3 بيستخدم combination من الـ PAUSE_CAP و PAUSE_ASYM bits لتمثيل 4 حالات:

| cap | PAUSE_CAP | PAUSE_ASYM | المعنى |
|---|---|---|---|
| RX only | 1 | 1 | يقبل TX من الطرف التاني |
| TX only | 0 | 1 | يعلن إنه عايز TX فقط |
| RX+TX | 1 | 0 | Symmetric pause |
| None | 0 | 0 | لا pause |

**Parameters:**
- `cap` — combination من `FLOW_CTRL_RX` (0x02) و `FLOW_CTRL_TX` (0x01)

**Return Value:** u16 من `ADVERTISE_PAUSE_*` bits

**Key Details:**
- XOR على `ADVERTISE_PAUSE_ASYM` لو `FLOW_CTRL_TX` — هذا يتبع الـ IEEE table بدقة
- بتُستدعى من الـ driver عند ملء الـ `MII_ADVERTISE` register

---

#### `mii_resolve_flowctrl_fdx`

```c
static inline u8 mii_resolve_flowctrl_fdx(u16 lcladv, u16 rmtadv)
```

**الـ function** بتحسب الـ TX/RX pause capabilities الفعلية بعد إتمام الـ autoneg. بتطبق IEEE 802.3-2005 Table 28B-3.

قواعد الـ resolution:
```
if (local PAUSE_CAP && remote PAUSE_CAP):
    cap = TX + RX
else if (local PAUSE_ASYM && remote PAUSE_ASYM):
    if (local PAUSE_CAP):   cap = RX only
    if (remote PAUSE_CAP):  cap = TX only
else:
    cap = 0 (no pause)
```

**Parameters:**
- `lcladv` — قيمة `MII_ADVERTISE` المحلي (local advertisement)
- `rmtadv` — قيمة `MII_LPA` من الـ link partner

**Return Value:** u8 من `FLOW_CTRL_TX` (0x01) و`FLOW_CTRL_RX` (0x02) أو 0

**Key Details:**
- فقط للـ full-duplex links (الـ function name ينتهي بـ `_fdx`)
- الـ half-duplex ما فيهوش pause في الـ standard
- نتيجتها بتُستخدم لبرمجة الـ MAC flow control registers

---

### المجموعة 10: BMCR Encoding

---

#### `mii_bmcr_encode_fixed`

```c
static inline u16 mii_bmcr_encode_fixed(int speed, int duplex)
```

**الـ function** بتحول speed و duplex values لـ `BMCR` register value مناسب لوضع الـ forced mode (بدون autoneg). بتُستخدم لما الـ driver عايز يضبط الـ PHY على speed معينة بالقوة.

**Parameters:**
- `speed` — `SPEED_2500`، `SPEED_1000`، `SPEED_100`، `SPEED_10`، أو أي قيمة أخرى
- `duplex` — `DUPLEX_FULL` أو `DUPLEX_HALF`

**Return Value:** u16 يُكتب مباشرةً في `MII_BMCR`

**Key Details:**
- `SPEED_2500` بيتـ encode كـ `BMCR_SPEED1000` لأن BMCR مش بيدعم 2500 صراحةً — الـ 2500Mbps بتتتحكم فيه عادةً عبر extended registers أو vendor-specific bits
- الـ speeds الغريبة كلها بتتـ fallback على 10Mbps
- `DUPLEX_FULL` الوحيد اللي بيـ set الـ `BMCR_FULLDPLX` bit — أي قيمة تانية بتيجي half-duplex

```c
/* Example: force 100Mbps full-duplex */
u16 bmcr = mii_bmcr_encode_fixed(SPEED_100, DUPLEX_FULL);
/* bmcr = BMCR_SPEED100 | BMCR_FULLDPLX */
mdio_write(dev, phy_id, MII_BMCR, bmcr);
```

---

### ملخص العلاقات بين الـ Groups

```
        userspace ethtool/ioctl
               |
    +----------+------------------+
    |                             |
mii_ethtool_gset/sset        generic_mii_ioctl
mii_ethtool_get/set               |
  _link_ksettings             mdio_read/write
    |                             |
    |      +----------------------+
    |      |
    v      v
  mii_check_media / mii_check_link
    |
    +-- mii_link_ok()          <- BMSR link bit
    +-- mii_nway_result()      <- best negotiated mode
    +-- mii_duplex()           <- full/half decision
    +-- netif_carrier_on/off() <- notifies TCP/IP stack

Translation Layer (all inline, zero overhead):
  ethtool <-> MII regs:    ethtool_adv_to_mii_adv_t / mii_adv_to_ethtool_adv_t
  linkmode <-> MII regs:   linkmode_adv_to_mii_adv_t / mii_adv_mod_linkmode_adv_t
  Flow control:            mii_advertise_flowctrl / mii_resolve_flowctrl_fdx
  1000BaseX:               *_adv_x / mii_lpa_mod_linkmode_x
```
## Phase 5: دليل الـ Debugging الشامل

### Software Level

---

#### 1. debugfs — المدخلات المتعلقة بـ MII/PHY

**الـ** debugfs مش بيعرض MII registers مباشرةً، لكن الـ PHY subsystem بيستخدمه بشكل غير مباشر.

```bash
# mount debugfs لو مش mounted
mount -t debugfs none /sys/kernel/debug

# PHY device debug info
ls /sys/kernel/debug/phy/
cat /sys/kernel/debug/phy/<phy-device>/state

# MDIO bus debug entries
ls /sys/kernel/debug/mdio_bus/
# buses زي: stmmac-0, fixed-0, mv643xx_eth-0

# stats للـ bus
cat /sys/kernel/debug/mdio_bus/<bus-name>/stats

# العثور على الـ buses المتاحة
ls /sys/kernel/debug/mdio-buses/ 2>/dev/null || ls /sys/kernel/debug/mdio_bus/
```

---

#### 2. sysfs — المدخلات المتعلقة بـ MII

```bash
# الـ PHY نفسه
ls /sys/bus/mdio_bus/devices/
# مثال: stmmac-0:00  stmmac-0:01

# حالة الـ PHY
cat /sys/bus/mdio_bus/devices/<phy-id>/state
# possible values: DOWN, READY, RUNNING, HALTED, ERROR

# السرعة والـ duplex والـ carrier من خلال الـ netdev
cat /sys/class/net/eth0/speed       # 10, 100, 1000
cat /sys/class/net/eth0/duplex      # full, half
cat /sys/class/net/eth0/carrier     # 1 = link up, 0 = link down
cat /sys/class/net/eth0/operstate   # up, down, unknown
cat /sys/class/net/eth0/phy_id      # مثال: 0x001cc915 = Realtek RTL8211
```

**تفسير الـ phy_id:**

| Bits | المحتوى |
|------|---------|
| [31:16] | OUI Bit 3-18 (PHYSID1 register) |
| [15:10] | OUI Bit 19-24 |
| [9:4] | Model number |
| [3:0] | Revision |

---

#### 3. ftrace — Tracepoints/Events للـ MII/PHY

```bash
# عرض events المتاحة للـ mdio
ls /sys/kernel/debug/tracing/events/mdio/
ls /sys/kernel/debug/tracing/events/net/

# تفعيل tracing للـ MDIO transactions
echo 1 > /sys/kernel/debug/tracing/events/mdio/enable

# تفعيل function tracing لـ mii functions
echo mii_check_link  > /sys/kernel/debug/tracing/set_ftrace_filter
echo mii_check_media >> /sys/kernel/debug/tracing/set_ftrace_filter
echo mii_link_ok     >> /sys/kernel/debug/tracing/set_ftrace_filter
echo mii_nway_restart >> /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe

# تفعيل graph tracing لرؤية call chain كامل
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo mii_nway_restart > /sys/kernel/debug/tracing/set_graph_function
cat /sys/kernel/debug/tracing/trace

# cleanup
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo nop > /sys/kernel/debug/tracing/current_tracer
echo > /sys/kernel/debug/tracing/set_ftrace_filter
```

**مثال على output من mdio tracepoint:**
```
kworker/0:2-45 [000] .... 1234.567890: mdio_access: eth0 phy:1 reg:0x01 val:0x796d write:0
# reg 0x01 = MII_BMSR
# val 0x796d → bit2=link up, bit5=autoneg complete, bit14=100FD capable
```

---

#### 4. printk و Dynamic Debug

```bash
# تفعيل dynamic debug لـ MII/PHY subsystem
echo 'module libphy +p'      > /sys/kernel/debug/dynamic_debug/control
echo 'module fixed_phy +p'  >> /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/net/mii.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/net/phy/mdio_bus.c +p' >> /sys/kernel/debug/dynamic_debug/control

# تفعيل لـ driver معين
echo 'file drivers/net/phy/realtek.c +p' > /sys/kernel/debug/dynamic_debug/control

# التحقق من الـ rules الفعالة
cat /sys/kernel/debug/dynamic_debug/control | grep -E 'mii|phy'

# رفع مستوى الـ printk لرؤية KERN_DEBUG
echo 8 > /proc/sys/kernel/printk

# متابعة live
dmesg -w | grep -i 'mii\|phy\|mdio\|autoneg\|link'
```

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الغرض |
|---------------|-------|
| `CONFIG_PHYLIB` | PHY library الأساسية — لازم يكون enabled |
| `CONFIG_MDIO_DEVICE` | MDIO bus device support |
| `CONFIG_NET_PHY_DEBUG` | تفعيل `dev_dbg` في phy drivers |
| `CONFIG_DYNAMIC_DEBUG` | dynamic debug activation بدون recompile |
| `CONFIG_FTRACE` | function tracing framework |
| `CONFIG_FUNCTION_TRACER` | تفعيل function-level tracing |
| `CONFIG_TRACEPOINTS` | لازم يكون enabled للـ mdio tracepoints |
| `CONFIG_FIXED_PHY` | fixed-link PHY للاختبار بدون PHY حقيقي |
| `CONFIG_DEBUG_KERNEL` | base debug support |
| `CONFIG_NET_DROP_MONITOR` | مراقبة الـ dropped packets |
| `CONFIG_LOCKDEP` | تتبع lock dependencies |

```bash
# فحص الـ config الحالي
zcat /proc/config.gz | grep -E 'CONFIG_(PHYLIB|MDIO|MII|PHY_DEBUG|DYNAMIC_DEBUG|FTRACE)'
# أو
grep -E 'CONFIG_(PHYLIB|MDIO|MII)' /boot/config-$(uname -r)
```

---

#### 6. ethtool — الأداة الرئيسية لـ MII Debugging

```bash
# عرض الإعدادات الحالية (BMCR + BMSR + autoneg result)
ethtool eth0

# عرض capabilities التفصيلية
ethtool -a eth0      # pause frames / flow control
ethtool -S eth0      # statistics (FCS errors, missed, etc.)
ethtool -i eth0      # driver + PHY info (phy-id, bus-info)

# إجبار restart لـ autoneg — يستدعي mii_nway_restart() داخلياً
ethtool -r eth0

# تغيير السرعة/duplex — force_media mode (يضبط mii_if_info.force_media)
ethtool -s eth0 speed 100 duplex full autoneg off

# رجوع لـ autoneg
ethtool -s eth0 autoneg on
ethtool -r eth0

# قراءة PHY registers مباشرة عبر SIOCGMIIREG ioctl
mii-tool -vvv eth0

# مراقبة مستمرة للتغييرات
mii-tool --watch eth0
```

**مثال output لـ ethtool ومعنى كل سطر:**
```
Settings for eth0:
    Supported ports: [ TP MII ]
    Supported link modes:   10baseT/Half 10baseT/Full
                            100baseT/Half 100baseT/Full
                            1000baseT/Full           ← من BMSR + ESTATUS + CTRL1000
    Supported pause frame use: Symmetric Receive-only
    Supports auto-negotiation: Yes                   ← BMSR_ANEGCAPABLE set
    Advertised link modes:  10baseT/Half 10baseT/Full
                            100baseT/Half 100baseT/Full
                            1000baseT/Full           ← من MII_ADVERTISE + MII_CTRL1000
    Advertised pause frame use: Symmetric            ← ADVERTISE_PAUSE_CAP set
    Link partner advertised link modes:  1000baseT/Full ← من MII_LPA + MII_STAT1000
    Speed: 1000Mb/s                                  ← نتيجة mii_nway_result()
    Duplex: Full                                     ← نتيجة mii_duplex()
    Auto-negotiation: on                             ← BMCR_ANENABLE set
    Link detected: yes                               ← mii_link_ok() → BMSR_LSTATUS=1
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---------------------|--------|------|
| `eth0: Link is Down` | `BMSR_LSTATUS` = 0 في MII_BMSR | فحص الكابل، تأكد `mii_link_ok()` يُستدعى صح |
| `MDIO timeout` / `MDIO read failed` | الـ PHY مش بيستجيب على الـ MDIO bus | فحص wiring، تأكد `phy_id` صح في `mii_if_info` |
| `PHY ID doesn't match` | `MII_PHYSID1`/`MII_PHYSID2` لا يطابق الـ driver | تحقق من `phy_id_mask` وـ `phy_id` |
| `no link during initialization` | `BMSR_ANEGCOMPLETE` ما اتضبطش | زود timeout الـ autoneg، فحص الـ ADVERTISE register |
| `autoneg failed` | `LPA_LPACK` = 0 في MII_LPA | تأكد الطرفين يدعموا mode مشترك في الـ ADVERTISE |
| `Master/Slave resolution failure` | `LPA_1000MSFAIL` set في MII_STAT1000 | غير `CTL1000_AS_MASTER` أو reset الـ PHY |
| `mii_check_gmii_support: GMII not supported` | `BMSR_ESTATEN` = 0 | الـ PHY مش 1000BASE-T، اضبط `supports_gmii = 0` |
| `PHY jabber detected` | `BMSR_JCD` set | فحص وجود loops أو مشكلة في الكابل |
| `Remote fault detected` | `BMSR_RFAULT` set | الطرف التاني يبلغ عن fault — فحص الـ remote end |
| `duplex mismatch` | تعارض duplex بين الطرفين | `mii_duplex()` يتعارض مع MAC config، فحص الـ LPA |
| `PHY reset timed out` | `BMCR_RESET` ظل set | مشكلة hardware أو MDC/MDIO clock |
| `1000BASE-T master/slave config failed` | `LPA_1000MSFAIL` في MII_STAT1000 | ضبط `CTL1000_PREFER_MASTER` أو reset الـ PHY |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في driver probe — تحقق من صحة mii_if_info setup */
static int mydrv_probe(struct net_device *dev)
{
    struct mydrv_priv *priv = netdev_priv(dev);
    struct mii_if_info *mii = &priv->mii;

    /* callbacks لازم تكون set قبل أي MDIO access */
    WARN_ON(!mii->mdio_read);
    WARN_ON(!mii->mdio_write);

    /* phy_id_mask لازم non-zero وإلا بنقرأ PHY غلط */
    WARN_ON(mii->phy_id_mask == 0);

    /* reg_num_mask لازم valid */
    WARN_ON(mii->reg_num_mask == 0);

    return 0;
}

/* في interrupt handler — BMSR يلازم يُقرأ مرتين (لأنه latched) */
static irqreturn_t mydrv_phy_irq(int irq, void *dev_id)
{
    struct mii_if_info *mii = dev_id;
    int bmsr;

    /* القراءة الأولى تمسح الـ latched bits */
    bmsr = mii->mdio_read(mii->dev, mii->phy_id, MII_BMSR);
    /* القراءة الثانية تعطي الحالة الحقيقية */
    bmsr = mii->mdio_read(mii->dev, mii->phy_id, MII_BMSR);

    if (WARN_ON(bmsr < 0)) {
        /* MDIO failed — dump للـ call stack لمعرفة الـ context */
        dump_stack();
        return IRQ_HANDLED;
    }

    mii_check_media(mii, 1 /* print */, 0 /* not init */);
    return IRQ_HANDLED;
}

/* في generic_mii_ioctl — تحقق من الـ register range */
int mydrv_ioctl(struct net_device *dev, struct ifreq *rq, int cmd)
{
    struct mii_if_info *mii = &((struct mydrv_priv *)netdev_priv(dev))->mii;
    struct mii_ioctl_data *mii_data = if_mii(rq);

    /* تحقق أن register number داخل المدى المسموح */
    WARN_ON(mii_data->reg_num > mii->reg_num_mask);

    return generic_mii_ioctl(mii, mii_data, cmd, NULL);
}

/* بعد force_media — تحقق أن الـ MAC اتعدل */
static void mydrv_link_change(struct net_device *dev)
{
    struct mydrv_priv *priv = netdev_priv(dev);
    unsigned int changed = mii_check_media(&priv->mii, 1, 0);

    if (changed) {
        /* تأكد إن الـ MAC duplex اتحدث فعلاً */
        WARN_ON(priv->mii.full_duplex != mydrv_get_mac_duplex(dev));
        mydrv_update_mac_duplex(dev, priv->mii.full_duplex);
    }
}
```

**النقاط الاستراتيجية الأهم:**
- في `mdio_read`/`mdio_write` callbacks عند القيم السالبة (MDIO timeout)
- في أول link-down transition غير متوقع
- عند تناقض بين `mii_if_info.full_duplex` وما قرأه `mii_duplex()` من الـ registers
- عند استدعاء `mii_nway_restart()` وـ `force_media = 1` (تناقض منطقي)

---

### Hardware Level

---

#### 1. التحقق من مطابقة Hardware State مع Kernel State

```
MAC <---MII Bus---> PHY <---Twisted Pair---> Remote PHY
       MDC/MDIO          RJ45 connector
```

| Kernel Variable | يقابله Register | طريقة التحقق |
|----------------|-----------------|--------------|
| `mii->full_duplex` | BMCR bit 8 (`BMCR_FULLDPLX`) | `ethtool eth0 \| grep Duplex` |
| `mii->force_media` | BMCR bit 12 (`BMCR_ANENABLE`) = 0 | `ethtool eth0 \| grep auto-neg` |
| Link up/down | BMSR bit 2 (`BMSR_LSTATUS`) | `cat /sys/class/net/eth0/carrier` |
| Autoneg done | BMSR bit 5 (`BMSR_ANEGCOMPLETE`) | `ethtool eth0 \| grep 'Auto-negot'` |
| Speed 1000 | BMCR bit 6 (`BMCR_SPEED1000`) | `cat /sys/class/net/eth0/speed` |
| 1000T LPA | MII_STAT1000 bits [11:10] | `phytool read eth0/1/10` |
| Advertised caps | `MII_ADVERTISE` reg 4 | `phytool read eth0/1/4` |

```bash
# برنامج Python لقراءة MII registers عبر ioctl مباشرة
python3 - <<'EOF'
import socket, struct, fcntl

SIOCGMIIREG = 0x8948
IFNAMSIZ = 16
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
iface = b'eth0'

regs = {0: 'BMCR', 1: 'BMSR', 4: 'ADVERTISE', 5: 'LPA', 9: 'CTRL1000', 10: 'STAT1000'}

for reg, name in regs.items():
    ifr = struct.pack('16sHHHH',
                      iface.ljust(IFNAMSIZ, b'\x00'),
                      1, reg, 0, 0)   # phy_id = 1
    result = fcntl.ioctl(sock, SIOCGMIIREG, ifr)
    _, _, _, _, val = struct.unpack('16sHHHH', result)
    print(f"REG 0x{reg:02x} ({name:12}) = 0x{val:04x}")

sock.close()
EOF
```

---

#### 2. Register Dump Techniques

**باستخدام `phytool` (الأسهل والأدق):**
```bash
# قراءة كل registers (0-31)
for reg in $(seq 0 31); do
    val=$(phytool read eth0/1/$reg 2>/dev/null)
    printf "REG[%02d] = %s\n" "$reg" "$val"
done

# تفسير BMSR مباشرة
python3 -c "
bmsr = 0x$(phytool read eth0/1/1 | sed 's/0x//')
print(f'BMSR = 0x{bmsr:04X}')
print(f'  Link:          {\"UP\" if bmsr & 0x0004 else \"DOWN\"}')
print(f'  Autoneg Done:  {bool(bmsr & 0x0020)}')
print(f'  10HD capable:  {bool(bmsr & 0x0800)}')
print(f'  10FD capable:  {bool(bmsr & 0x1000)}')
print(f'  100HD capable: {bool(bmsr & 0x2000)}')
print(f'  100FD capable: {bool(bmsr & 0x4000)}')
print(f'  Ext Status:    {bool(bmsr & 0x0100)}')
"
```

**باستخدام `devmem2` للـ MDIO controller registers:**
```bash
# مثال على STM32 EMAC (MDIO base = 0x40028C00)
devmem2 0x40028C10 w    # MACMIIDR — MDIO Data Register
devmem2 0x40028C14 w    # MACMIIAR — MDIO Address Register

# مثال على Allwinner (EMAC MDIO base = 0x01C0B000)
devmem2 0x01C0B060 w

# مثال على Marvell MDIO
devmem2 0xF1072000 w
```

**باستخدام `mii-tool`:**
```bash
# dump تفصيلي للـ MII registers
mii-tool -vvv eth0

# Output مثال:
# product info: vendor 00:1c:c4, model 3 rev 3  ← من MII_PHYSID1+MII_PHYSID2
# basic mode:   autonegotiation enabled           ← BMCR_ANENABLE
# basic status: autonegotiation complete, link ok ← BMSR_ANEGCOMPLETE | BMSR_LSTATUS
# capabilities: 1000baseT-FD 100baseTx-FD ...    ← من BMSR + ESTATUS
# advertising:  1000baseT-FD 100baseTx-FD ...    ← من MII_ADVERTISE + MII_CTRL1000
# link partner: 1000baseT-FD flow-control        ← من MII_LPA + MII_STAT1000
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

**الـ MDIO Bus Protocol على الـ wire:**
```
PRE (32×1) | ST(01) | OP(10=RD/01=WR) | PHYAD(5bits) | REGAD(5bits) | TA(Z→0) | DATA(16bits)

MDC  _____|‾|_|‾|_|‾|_|‾|_|‾|_|‾|_|‾|_|‾|_|‾|_|‾|_|‾|_|‾|___
MDIO ___IDLE__PRE(32)__ST__OP_PHYAD_REGAD__TA____DATA(16)______
```

**إعدادات الـ Logic Analyzer:**

| الإعداد | القيمة |
|---------|--------|
| Sample rate | ≥ 25 MSPS (10x لـ MDC بـ 2.5 MHz) |
| Trigger | Rising edge على أول bit من الـ preamble |
| Protocol decoder | MDIO/SMI (متوفر في Saleae Logic 2 و PulseView) |
| Voltage threshold | 1.65V (لـ 3.3V CMOS) |

**إعدادات الـ Oscilloscope:**

| Signal | Voltage | الـ Rise Time |
|--------|---------|--------------|
| MDC | 3.3V CMOS | < 10 ns |
| MDIO | 3.3V CMOS | < 10 ns |
| TX+/TX- | ±1V differential | حسب السرعة |

**ما تبحث عنه:**
- الـ preamble: 32 bit = 1 بالضبط قبل كل frame
- الـ turnaround (TA): bit أول = Hi-Z، bit ثاني = 0
- الـ MDIO يكون stable قبل rising edge للـ MDC بـ ≥ 10 ns
- الـ pull-up على MDIO يعطي rise time يدل على قيمته (1.5kΩ–4.7kΩ)

---

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة الهاردوير | Pattern في dmesg | التشخيص |
|------------------|------------------|---------|
| MDC/MDIO wire مقطوع | `PHY ID doesn't match` أو timeout | فحص continuity بـ multimeter |
| MDIO pull-up مفقود | Register values = `0xFFFF` دايماً | إضافة 1kΩ pull-up على MDIO إلى VDD |
| Power sequencing خطأ | `PHY not ready` عند boot | فحص timing بين VDD وRESET_N بالأوسيلوسكوب |
| Crystal 25MHz معطوب | Link مش بيطلع رغم PHY ID صح | فحص crystal بالأوسيلوسكوب |
| Impedance mismatch | Link flapping + CRC errors | فحص termination resistors (100Ω differential) |
| Magnetic transformer معطوب | `carrier lost` + physical errors | فحص inductance بالـ LCR meter |
| TX/RX polarity عكس | Link ميطلعش مع أي partner | عكس polarity في الـ DT أو driver |
| PHY address hardware strapping خطأ | `mii: unregistered phy id` | فحص PHYAD[2:0] pins على الـ PCB |
| Master/Slave 1000T conflict | `LPA_1000MSFAIL` في STAT1000 | ضبط `CTL1000_AS_MASTER` في CTRL1000 |

---

#### 5. Device Tree Debugging

**التحقق من DT يطابق الهاردوير:**
```bash
# عرض الـ DT المحمّل فعلاً
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 'mdio'

# فحص phy-mode (يجب أن يطابق الـ PCB connections)
cat /proc/device-tree/soc/ethernet@*/phy-mode | tr -d '\0'
# الناتج المتوقع: "rgmii-id" أو "mii" أو "sgmii" إلخ

# مقارنة مع ethtool
ethtool eth0 | grep Port

# فحص PHY address
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -B2 -A5 'phy@'
```

**DT entries المهمة لـ MII:**
```dts
/* مثال DT لـ interface يستخدم mii_if_info */
&fec {
    status = "okay";
    phy-mode = "mii";           /* يجب أن يطابق الهاردوير */
    phy-handle = <&phy0>;

    mdio {
        #address-cells = <1>;
        #size-cells = <0>;

        phy0: phy@1 {           /* address يجب يطابق PHYAD hardware pins */
            reg = <1>;          /* هذا هو phy_id في mii_if_info */
            max-speed = <100>;  /* تقييد السرعة لو الكابل لا يدعم 1G */
        };
    };
};
```

**أخطاء DT شائعة وحلها:**
```bash
# لو الـ phy reg لا يطابق الهاردوير
dmesg | grep 'mdio\|PHY'
# [2.34] stmmac-0: No PHY found
# الحل: فحص PHYAD strapping على الـ board وتصحيح reg في DT

# تحقق من PHYSID registers لمعرفة PHY الحقيقي
phytool read eth0/0/2   # PHYSID1 — OUI bits [22:7]
phytool read eth0/0/3   # PHYSID2 — OUI bits [6:1] + model + rev
# لو رجع 0xFFFF → address خطأ
# لو رجع 0x0000 → MDC/MDIO مقطوع
```

---

### Practical Commands

---

#### سكريبت شامل لـ MII Debugging

```bash
#!/bin/bash
# mii-debug.sh — comprehensive MII/PHY debug script
# Usage: ./mii-debug.sh [interface]
IFACE="${1:-eth0}"

echo "=== MII Debug Report for $IFACE ==="
echo "Timestamp: $(date)"
echo ""

echo "--- Basic Link State ---"
ip link show "$IFACE"
echo "Carrier: $(cat /sys/class/net/$IFACE/carrier 2>/dev/null) (1=up, 0=down)"
echo "Operstate: $(cat /sys/class/net/$IFACE/operstate 2>/dev/null)"
echo "Speed: $(cat /sys/class/net/$IFACE/speed 2>/dev/null) Mbps"
echo "Duplex: $(cat /sys/class/net/$IFACE/duplex 2>/dev/null)"
echo ""

echo "--- PHY Info ---"
ethtool -i "$IFACE" 2>/dev/null
echo ""

echo "--- Autoneg & Link Settings ---"
ethtool "$IFACE" 2>/dev/null
echo ""

echo "--- Flow Control (Pause Frames) ---"
ethtool -a "$IFACE" 2>/dev/null
echo ""

echo "--- Error Statistics ---"
ethtool -S "$IFACE" 2>/dev/null | grep -Ei 'error|drop|miss|fifo|crc|frame|collision'
echo ""

echo "--- Recent MII Kernel Messages ---"
dmesg --since "1 hour ago" 2>/dev/null | grep -i "$IFACE\|phy\|mdio\|mii\|autoneg" | tail -30
```

```bash
chmod +x mii-debug.sh
./mii-debug.sh eth0
```

---

#### قراءة MII Registers خطوة بخطوة

```bash
# الخطوة 1: تحديد PHY ID من الهاردوير
phytool read eth0/1/2   # PHYSID1 → OUI[22:7]
phytool read eth0/1/3   # PHYSID2 → OUI[6:1] + model + rev
# مثال: 0x001c + 0xc915 → RTL8211E (Realtek OUI = 00:1c:c4)

# الخطوة 2: فحص BMCR (reg 0) — mode حالي
phytool read eth0/1/0
# 0x1140 → bit12=ANENABLE, bit8=FULLDPLX, bit6=SPEED1000(MSB)

# الخطوة 3: فحص BMSR (reg 1) مرتين (latched link-down)
phytool read eth0/1/1   # القراءة الأولى تمسح latched bits
phytool read eth0/1/1   # القراءة الثانية = القيمة الحقيقية
# 0x796D → bit2=link up, bit5=autoneg complete, bit14=100FD, bit12=10FD

# الخطوة 4: ما نعلن عنه (reg 4 = MII_ADVERTISE)
phytool read eth0/1/4
# 0x05E1 → ADVERTISE_PAUSE_CAP + 100FD + 100HD + 10FD + 10HD + CSMA

# الخطوة 5: ما يدعمه الطرف الآخر (reg 5 = MII_LPA)
phytool read eth0/1/5
# 0x45E1 → LPA_LPACK + 100FD + 100HD + 10FD + 10HD

# الخطوة 6: advertisement للـ 1000T (reg 9 = MII_CTRL1000)
phytool read eth0/1/9
# 0x0300 → ADVERTISE_1000FULL + ADVERTISE_1000HALF

# الخطوة 7: partner 1000T status (reg 10 = MII_STAT1000)
phytool read eth0/1/10
# 0x3C00 → LPA_1000FULL + LPA_1000HALF + LPA_1000LOCALRXOK + LPA_1000REMRXOK
# لو bit15 (LPA_1000MSFAIL) = 1 → master/slave resolution failure!
```

---

#### تشخيص مشكلة Autoneg

```bash
# الخطوة 1: تأكد hardware يشوف carrier
cat /sys/class/net/eth0/carrier
# لو 0 مع كابل موصل → مشكلة في PHY أو الكابل نفسه

# الخطوة 2: فحص BMSR
BMSR=$(phytool read eth0/1/1 2>/dev/null)
python3 -c "
val = int('$BMSR', 16)
print(f'BMSR = 0x{val:04X}')
print(f'  Link Up:       {bool(val & 0x0004)}')
print(f'  Autoneg Done:  {bool(val & 0x0020)}')
print(f'  Autoneg Cap:   {bool(val & 0x0008)}')
print(f'  Remote Fault:  {bool(val & 0x0010)}')
print(f'  Jabber:        {bool(val & 0x0002)}')
"

# الخطوة 3: restart autoneg
ethtool -r eth0
sleep 3
ethtool eth0 | grep -E 'Speed|Duplex|Link'

# الخطوة 4: لو autoneg مش بتشتغل — force mode
ethtool -s eth0 speed 100 duplex full autoneg off
dmesg | tail -10

# الخطوة 5: فحص Master/Slave في 1000T
STAT1000=$(phytool read eth0/1/10 2>/dev/null)
python3 -c "
val = int('$STAT1000', 16)
print(f'STAT1000 = 0x{val:04X}')
print(f'  M/S Fail:      {bool(val & 0x8000)}')  # LPA_1000MSFAIL
print(f'  M/S Result:    {\"Master\" if val & 0x4000 else \"Slave\"}')
print(f'  Local RX OK:   {bool(val & 0x2000)}')
print(f'  Remote RX OK:  {bool(val & 0x1000)}')
print(f'  LP 1000FD:     {bool(val & 0x0800)}')
print(f'  LP 1000HD:     {bool(val & 0x0400)}')
"
```

---

#### تشخيص Flow Control

```bash
# فحص الإعداد الحالي (يعكس نتيجة mii_resolve_flowctrl_fdx)
ethtool -a eth0
# Pause parameters for eth0:
# Autonegotiated: on
# RX:             on
# TX:             off

# قراءة ADVERTISE pause bits مباشرة (reg 4)
ADV=$(phytool read eth0/1/4 2>/dev/null)
python3 -c "
val = int('$ADV', 16)
print(f'ADVERTISE = 0x{val:04X}')
print(f'  PAUSE_CAP:  {bool(val & 0x0400)}')   # ADVERTISE_PAUSE_CAP
print(f'  PAUSE_ASYM: {bool(val & 0x0800)}')   # ADVERTISE_PAUSE_ASYM
"

# قراءة LPA pause bits من الطرف الآخر (reg 5)
LPA=$(phytool read eth0/1/5 2>/dev/null)
python3 -c "
val = int('$LPA', 16)
print(f'LPA = 0x{val:04X}')
print(f'  PAUSE_CAP:  {bool(val & 0x0400)}')
print(f'  PAUSE_ASYM: {bool(val & 0x0800)}')
"

# IEEE 802.3 table 28B-3 — تطبيق mii_resolve_flowctrl_fdx منطقياً:
# lcl_PAUSE_CAP=1 AND rmt_PAUSE_CAP=1 → TX+RX enabled
# lcl_PAUSE_CAP=0, lcl_ASYM=1, rmt_PAUSE_CAP=1, rmt_ASYM=1 → TX only
# lcl_PAUSE_CAP=1, lcl_ASYM=1, rmt_PAUSE_CAP=0, rmt_ASYM=1 → RX only

# تعطيل flow control لو بيسبب مشاكل
ethtool -A eth0 rx off tx off
```

---

#### تفعيل MDIO Tracing الشامل

```bash
#!/bin/bash
# mdio-trace.sh — trace all MDIO transactions for N seconds
SECS="${1:-10}"
TRACE_DIR="/sys/kernel/debug/tracing"

echo "Enabling MDIO tracing for ${SECS}s..."

echo 0 > "$TRACE_DIR/tracing_on"
echo > "$TRACE_DIR/trace"

# enable MDIO events لو متاحة
echo 1 > "$TRACE_DIR/events/mdio/enable" 2>/dev/null

# function tracer لـ mii functions
echo function > "$TRACE_DIR/current_tracer"
{
    echo mii_check_link
    echo mii_check_media
    echo mii_link_ok
    echo mii_nway_restart
    echo mii_nway_result
    echo generic_mii_ioctl
} > "$TRACE_DIR/set_ftrace_filter"

echo 1 > "$TRACE_DIR/tracing_on"
sleep "$SECS"
echo 0 > "$TRACE_DIR/tracing_on"

echo "=== MDIO/MII Trace (first 100 lines) ==="
head -100 "$TRACE_DIR/trace"

# cleanup
echo nop > "$TRACE_DIR/current_tracer"
echo > "$TRACE_DIR/set_ftrace_filter"
echo 0 > "$TRACE_DIR/events/mdio/enable" 2>/dev/null
```

```bash
chmod +x mdio-trace.sh
./mdio-trace.sh 5
```

---

#### Health Check Script — للتشغيل الدوري

```bash
#!/bin/bash
# mii-health.sh — يشتغل كـ systemd timer أو cron كل دقيقة

IFACE="${1:-eth0}"
LOG="/var/log/mii-health.log"
TS=$(date '+%Y-%m-%d %H:%M:%S')

carrier=$(cat /sys/class/net/$IFACE/carrier 2>/dev/null)
speed=$(cat /sys/class/net/$IFACE/speed 2>/dev/null)
duplex=$(cat /sys/class/net/$IFACE/duplex 2>/dev/null)

echo "$TS iface=$IFACE carrier=$carrier speed=${speed}Mbps duplex=$duplex" >> "$LOG"

# فحص errors التراكمية
rx_err=$(ethtool -S $IFACE 2>/dev/null | grep -i 'rx.*error' | awk '{sum+=$2} END{print sum+0}')
tx_err=$(ethtool -S $IFACE 2>/dev/null | grep -i 'tx.*error' | awk '{sum+=$2} END{print sum+0}')

echo "$TS rx_errors=$rx_err tx_errors=$tx_err" >> "$LOG"

# تحذير في syslog
if [ "$carrier" = "0" ]; then
    logger -p kern.warn "MII health check: $IFACE link is DOWN"
fi
if [ "${rx_err:-0}" -gt 10000 ]; then
    logger -p kern.warn "MII health check: $IFACE excessive RX errors: $rx_err"
fi
if [ -n "$speed" ] && [ "$speed" -lt 1000 ] 2>/dev/null; then
    logger -p kern.notice "MII health check: $IFACE speed degraded to ${speed}Mbps"
fi
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — الـ PHY مش بيتفاوض صح

#### العنوان
Autonegotiation فاشل على Ethernet PHY في gateway صناعي بيشغّل Modbus/TCP

#### السياق
شركة بتبني industrial gateway بالـ RK3562 (Rockchip) لجمع بيانات من sensors عن طريق Modbus/TCP. الـ gateway بيتوصل بـ managed switch صناعي. في مرحلة الـ bring-up، الـ link بيجي بس throughput الـ Ethernet واطي جداً — قرايات الـ iperf بتوري 10Mbps بدل 100Mbps.

#### المشكلة
الـ driver بيستخدم `mii_if_info` ومش بيضبط الـ `supports_gmii` flag. نتيجة كده، الـ `mii_check_gmii_support()` بترجع 0 وبتخلي الـ driver يتجاهل الـ `MII_CTRL1000` register. الـ switch الصناعي بيعلن عن 1000BASE-T في الـ LPA بس الـ gateway مبيشوفوش.

#### التحليل
الـ `mii_check_gmii_support()` في `mii.c` بتتحقق من `MII_BMSR` أولاً:

```c
/* mii.c — mii_check_gmii_support */
int mii_check_gmii_support(struct mii_if_info *mii)
{
    int reg;

    reg = mii->mdio_read(mii->dev, mii->phy_id, MII_BMSR);
    if (reg < 0)
        return -1;

    /* BMSR_ESTATEN bit 0x0100 — يشوف لو في Extended Status */
    if (reg & BMSR_ESTATEN) {
        reg = mii->mdio_read(mii->dev, mii->phy_id, MII_ESTATUS);
        if (reg < 0)
            return -1;

        /* ESTATUS_1000_TFULL أو ESTATUS_1000_THALF */
        if (reg & (ESTATUS_1000_TFULL | ESTATUS_1000_THALF))
            return 1;
    }

    return 0;
}
```

الـ driver كان بيعمل:
```c
/* الكود الغلط في driver الـ RK3562 */
mii.supports_gmii = 0;  /* hard-coded صفر! */
```

بعد كده، في `mii_ethtool_get_link_ksettings()`:
```c
/* mii.c */
if (mii->supports_gmii) {
    /* ده مش بيتنفذ خالص لأن supports_gmii = 0 */
    adv = mii->mdio_read(dev, mii->phy_id, MII_CTRL1000);
    mii_ctrl1000_mod_linkmode_adv_t(cmd->link_modes.advertising, adv);
}
```

فبالتالي الـ `MII_ADVERTISE` register ما بيحطش `ADVERTISE_1000FULL` ولا `ADVERTISE_1000HALF`، والـ autonegotiation بيتم على 100Mbps كحد أقصى.

#### الحل

```c
/* الحل في probe() الخاص بـ driver */
static int rk3562_eth_probe(struct platform_device *pdev)
{
    struct rk3562_eth_priv *priv = ...;

    /* استخدم mii_check_gmii_support عشان تعرف GMII support */
    priv->mii.phy_id      = phy_addr;
    priv->mii.mdio_read   = rk3562_mdio_read;
    priv->mii.mdio_write  = rk3562_mdio_write;
    priv->mii.dev         = netdev;

    /* اسأل الـ PHY نفسه بدل ما تـ hard-code */
    priv->mii.supports_gmii = mii_check_gmii_support(&priv->mii);
    if (priv->mii.supports_gmii < 0) {
        dev_warn(&pdev->dev, "GMII check failed, defaulting to 0\n");
        priv->mii.supports_gmii = 0;
    }
    ...
}
```

بعد التعديل، تحقق:
```bash
# اقرأ MII_CTRL1000 register (reg 0x09) عشان تتأكد إن 1000T بيتعلن عنه
mii-tool -v eth0
ethtool eth0
# المتوقع: Speed: 1000Mb/s, Duplex: Full
```

#### الدرس المستفاد
لا تـ hard-code الـ `supports_gmii` في `mii_if_info`. دايماً استخدم `mii_check_gmii_support()` في الـ `probe()` عشان تسأل الـ PHY نفسه من الـ `BMSR` والـ `ESTATUS` registers. الـ PHY chip نفسه هو المصدر الصح.

---

### السيناريو 2: Android TV Box على Allwinner H616 — الـ link بيوقع بشكل متقطع

#### العنوان
الـ Ethernet بيوقع كل 5-10 دقايق في TV box بسبب `mii_check_link()` مش بتتكالل صح

#### السياق
منتج Android TV box بالـ Allwinner H616 SoC. الـ Ethernet port متوصل بـ RTL8211F PHY. المستخدمين بيشتكوا إن الـ streaming بيتقطع كل بضع دقايق والـ link indicator LED بيومض. الـ WiFi تمام، المشكلة في الـ Ethernet بس.

#### المشكلة
الـ driver بيستخدم `mii_check_link()` في الـ timer callback بس بيعمل كده كل 30 ثانية بدل كل ثانية. في نفس الوقت، `mii_link_ok()` بتقرأ `BMSR_LSTATUS` مرة واحدة بس، وده ما بيكفيش لأن الـ bit ده latch-low.

#### التحليل
`mii_check_link()` في `mii.c` بتعمل:

```c
void mii_check_link(struct mii_if_info *mii)
{
    int cur_link = mii_link_ok(mii);          /* بتقرأ BMSR_LSTATUS */
    int prev_link = netif_carrier_ok(mii->dev);

    if (cur_link && !prev_link)
        netif_carrier_on(mii->dev);
    else if (!cur_link && prev_link)
        netif_carrier_off(mii->dev);
}
```

و `mii_link_ok()` بتقرأ الـ `MII_BMSR` register:

```c
int mii_link_ok(struct mii_if_info *mii)
{
    /* BMSR_LSTATUS — bit 2 */
    if (mii->mdio_read(mii->dev, mii->phy_id, MII_BMSR) & BMSR_LSTATUS)
        return 1;
    return 0;
}
```

المشكلة: الـ `BMSR_LSTATUS` هو **latched-low** bit. يعني لو الـ link وقع ورجع قبل ما نقرأ، لازم نقرأ مرتين عشان نشيل الـ latch. الـ driver الخاطئ كان بيقرأ مرة واحدة بس فكانت بتصحى false-down events.

```c
/* الـ driver الخاطئ — قراية واحدة فقط */
static void h616_eth_timer(struct timer_list *t)
{
    struct h616_eth_priv *priv = from_timer(priv, t, link_timer);
    mii_check_link(&priv->mii);  /* BMSR_LSTATUS latched-low مش متعالجة! */
    mod_timer(&priv->link_timer, jiffies + 30 * HZ);  /* 30 ثانية تانية */
}
```

#### الحل

```c
/* الحل: قراية مزدوجة للـ BMSR عشان نشيل الـ latch */
static int h616_mii_link_ok(struct mii_if_info *mii)
{
    int bmsr;

    /* القراية الأولى تشيل الـ latch */
    bmsr = mii->mdio_read(mii->dev, mii->phy_id, MII_BMSR);
    /* القراية التانية هي الحقيقية */
    bmsr = mii->mdio_read(mii->dev, mii->phy_id, MII_BMSR);

    return (bmsr & BMSR_LSTATUS) ? 1 : 0;
}

/* وقلّل الـ timer interval لثانية واحدة */
static void h616_eth_timer(struct timer_list *t)
{
    struct h616_eth_priv *priv = from_timer(priv, t, link_timer);
    int cur_link = h616_mii_link_ok(&priv->mii);
    int prev_link = netif_carrier_ok(priv->mii.dev);

    if (cur_link && !prev_link)
        netif_carrier_on(priv->mii.dev);
    else if (!cur_link && prev_link)
        netif_carrier_off(priv->mii.dev);

    mod_timer(&priv->link_timer, jiffies + HZ); /* كل ثانية */
}
```

Debug commands:
```bash
# راقب carrier changes
watch -n 0.5 'cat /sys/class/net/eth0/carrier'
# اقرأ BMSR مباشرة
phytool read eth0/1/0x01   # MII_BMSR
```

#### الدرس المستفاد
الـ `BMSR_LSTATUS` في `MII_BMSR` هو latch-low register — لازم تقرأه مرتين متتاليتين عشان تاخد القيمة الصح. القراية الأولى بتشيل الـ latch، والتانية بتديك الحالة الفعلية دلوقتي. ده معروف في IEEE 802.3 spec.

---

### السيناريو 3: IoT Sensor على STM32MP1 — `ethtool -s` مبيشتغلش

#### العنوان
مستخدم مش قادر يـ force الـ speed لـ 100Mbps Full Duplex على STM32MP1 عن طريق `ethtool`

#### السياق
board بالـ STM32MP157 بيشغّل IIoT sensor aggregator. الـ network team محتاجة تـ force الـ link لـ 100BASE-TX Full Duplex بدون autonegotiation عشان يتوافق مع legacy PLC equipment. بيشغّلوا:
```bash
ethtool -s eth0 speed 100 duplex full autoneg off
```
بس الـ link بيفضل بـ autonegotiation ممكّن!

#### المشكلة
الـ driver بيعمل `generic_mii_ioctl()` للـ `SIOCETHTOOL` commands بس مش بيستخدم `mii_ethtool_set_link_ksettings()` صح. الـ `force_media` flag في `mii_if_info` مش بيتضبط.

#### التحليل
الـ flow المفروض:

```
ethtool -s eth0 speed 100 duplex full autoneg off
    → kernel ethtool ioctl
    → driver->ethtool_ops->set_link_ksettings()
    → mii_ethtool_set_link_ksettings()
    → mii_if_info.force_media = 1
    → يكتب BMCR بدون BMCR_ANENABLE
```

بس الـ driver كان بيعمل:
```c
/* driver الخاطئ */
static int stm32_eth_set_link_ksettings(struct net_device *dev,
                                         const struct ethtool_link_ksettings *cmd)
{
    struct stm32_priv *priv = netdev_priv(dev);
    /* بيكتب الـ BMCR مباشرة من غير ما يـ update force_media */
    u16 bmcr = mii_bmcr_encode_fixed(cmd->base.speed, cmd->base.duplex);
    priv->mii.mdio_write(dev, priv->mii.phy_id, MII_BMCR, bmcr);
    /* force_media مش بيتضبط! */
    return 0;
}
```

الـ `mii_bmcr_encode_fixed()` من `mii.h`:

```c
static inline u16 mii_bmcr_encode_fixed(int speed, int duplex)
{
    u16 bmcr;

    switch (speed) {
    case SPEED_1000:
        bmcr = BMCR_SPEED1000;
        break;
    case SPEED_100:
        bmcr = BMCR_SPEED100;  /* 0x2000 */
        break;
    default:
        bmcr = BMCR_SPEED10;   /* 0x0000 */
        break;
    }

    if (duplex == DUPLEX_FULL)
        bmcr |= BMCR_FULLDPLX; /* 0x0100 */

    return bmcr;
    /* لاحظ: BMCR_ANENABLE (0x1000) مش موجود = autoneg معطّل */
}
```

بس المشكلة أن `mii_check_link()` في الـ timer كانت بتستدعي `mii_nway_restart()` لأن `force_media == 0`:

```c
/* mii.c — mii_check_media */
unsigned int mii_check_media(struct mii_if_info *mii, ...)
{
    ...
    if (!mii->force_media) {
        /* بيعمل restart autoneg ويعيّد كل اللي اتعمل! */
        mii_nway_restart(mii);
    }
    ...
}
```

#### الحل

```c
/* الحل الصح: استخدم mii_ethtool_set_link_ksettings كاملاً */
static int stm32_eth_set_link_ksettings(struct net_device *dev,
                                         const struct ethtool_link_ksettings *cmd)
{
    struct stm32_priv *priv = netdev_priv(dev);
    int rc;

    spin_lock_irq(&priv->lock);
    rc = mii_ethtool_set_link_ksettings(&priv->mii, cmd);
    spin_unlock_irq(&priv->lock);

    return rc;
    /*
     * mii_ethtool_set_link_ksettings() بتضبط force_media
     * وبتكتب الـ BMCR الصح تلقائياً
     */
}
```

التحقق:
```bash
ethtool -s eth0 speed 100 duplex full autoneg off
ethtool eth0
# المتوقع:
# Speed: 100Mb/s
# Duplex: Full
# Auto-negotiation: off
phytool read eth0/1/0x00  # MII_BMCR
# المتوقع: 0x2100 = BMCR_SPEED100 | BMCR_FULLDPLX (بدون BMCR_ANENABLE)
```

#### الدرس المستفاد
لما بتعمل `force_media`، لازم تضبط `mii_if_info.force_media = 1` وإلا الـ `mii_check_media()` في الـ background timer هتعمل `mii_nway_restart()` وتفسد الإعداد. الأفضل تفوّض الموضوع كله لـ `mii_ethtool_set_link_ksettings()` بدل ما تعمل partial implementation.

---

### السيناريو 4: Automotive ECU على i.MX8 — Flow Control بيسبب jitter في CAN-over-Ethernet

#### العنوان
الـ Ethernet Pause frames بتسبب latency spikes في تطبيق CAN gateway على i.MX8M Plus

#### السياق
automotive ECU بالـ i.MX8M Plus بيعمل CAN-to-Ethernet gateway. الـ system بيحوّل CAN frames لـ UDP على شبكة داخلية. فريق الـ functional safety لاحظ latency spikes كل فترة وصلت لـ 50ms — فوق الحد المسموح بيه في AUTOSAR.

#### المشكلة
الـ Ethernet driver ممكّن الـ Pause frames (flow control) في الاتجاهين. لما الـ ECU بيبعت burst من الـ CAN frames، الـ switch بيبعتله Pause frame فبيوقف الـ TX لفترة وده بيسبب الـ latency spikes.

#### التحليل
الـ driver كان بيستخدم `mii_advertise_flowctrl()` لتوليد الـ advertisement:

```c
static inline u16 mii_advertise_flowctrl(int cap)
{
    u16 adv = 0;

    if (cap & FLOW_CTRL_RX)
        adv = ADVERTISE_PAUSE_CAP | ADVERTISE_PAUSE_ASYM;
    if (cap & FLOW_CTRL_TX)
        adv ^= ADVERTISE_PAUSE_ASYM;

    return adv;
}
```

الـ driver كان بيبعت:
```c
/* cap = FLOW_CTRL_TX | FLOW_CTRL_RX */
u16 flow_adv = mii_advertise_flowctrl(FLOW_CTRL_TX | FLOW_CTRL_RX);
/* النتيجة: ADVERTISE_PAUSE_CAP | ADVERTISE_PAUSE_ASYM ^ ADVERTISE_PAUSE_ASYM */
/* = ADVERTISE_PAUSE_CAP فقط — يعني symmetric pause */
```

بعد الـ autonegotiation، `mii_resolve_flowctrl_fdx()` بتحسب النتيجة:

```c
static inline u8 mii_resolve_flowctrl_fdx(u16 lcladv, u16 rmtadv)
{
    u8 cap = 0;

    /* لو الاتنين بيدعموا PAUSE_CAP → TX+RX pause ممكّن */
    if (lcladv & rmtadv & ADVERTISE_PAUSE_CAP) {
        cap = FLOW_CTRL_TX | FLOW_CTRL_RX;  /* ده اللي بيحصل */
    } else if (lcladv & rmtadv & ADVERTISE_PAUSE_ASYM) {
        if (lcladv & ADVERTISE_PAUSE_CAP)
            cap = FLOW_CTRL_RX;
        else if (rmtadv & ADVERTISE_PAUSE_CAP)
            cap = FLOW_CTRL_TX;
    }

    return cap;
}
```

النتيجة: `FLOW_CTRL_TX | FLOW_CTRL_RX` — يعني الـ ECU نفسه بيقبل Pause frames من الـ switch وبيوقف الـ TX.

#### الحل

للـ automotive application، محتاجين نمنع الـ TX pause (الـ ECU مش المفروض يوقف إرساله):

```c
/* i.MX8 ECU driver — نعلن عن RX pause بس، مش TX */
static void imx8_eth_setup_flowctrl(struct imx8_eth_priv *priv)
{
    u16 adv_reg;
    u16 lcladv, rmtadv;
    u8  fc_cap;

    /*
     * نعلن عن RX pause capability فقط
     * mii_advertise_flowctrl(FLOW_CTRL_RX):
     *   adv = ADVERTISE_PAUSE_CAP | ADVERTISE_PAUSE_ASYM
     *   cap & FLOW_CTRL_TX → false → لا XOR
     *   result = ADVERTISE_PAUSE_CAP | ADVERTISE_PAUSE_ASYM
     *
     * ده يخلي الـ resolve يطلع FLOW_CTRL_RX بس لو الـ switch يدعم PAUSE_CAP
     */
    adv_reg = mii_advertise_flowctrl(FLOW_CTRL_RX);

    /* اقرأ الـ MII_ADVERTISE الحالي وعدّل bits الـ pause بس */
    u16 current_adv = priv->mii.mdio_read(priv->mii.dev,
                                           priv->mii.phy_id,
                                           MII_ADVERTISE);
    current_adv &= ~(ADVERTISE_PAUSE_CAP | ADVERTISE_PAUSE_ASYM);
    current_adv |= adv_reg;

    priv->mii.mdio_write(priv->mii.dev, priv->mii.phy_id,
                          MII_ADVERTISE, current_adv);

    /* restart autoneg */
    mii_nway_restart(&priv->mii);
}

/* بعد اكتمال الـ autoneg، تحقق من النتيجة */
static void imx8_eth_check_fc(struct imx8_eth_priv *priv)
{
    u16 lcladv = priv->mii.mdio_read(priv->mii.dev,
                                      priv->mii.phy_id, MII_ADVERTISE);
    u16 rmtadv = priv->mii.mdio_read(priv->mii.dev,
                                      priv->mii.phy_id, MII_LPA);

    u8 fc = mii_resolve_flowctrl_fdx(lcladv, rmtadv);

    /* المتوقع: FLOW_CTRL_RX بس، مش TX */
    if (fc & FLOW_CTRL_TX)
        dev_warn(&priv->pdev->dev,
                 "TX pause unexpectedly enabled! fc=0x%x\n", fc);
}
```

```bash
# تحقق من الـ pause frames stats
ethtool -S eth0 | grep -i pause
# المتوقع:
# tx_pause_frames: 0   — مش بيبعت pause
# rx_pause_frames: N   — بيستقبل بس
```

#### الدرس المستفاد
الـ `mii_advertise_flowctrl()` و `mii_resolve_flowctrl_fdx()` بيطبّقوا IEEE 802.3-2005 Table 28B-3 للـ pause negotiation. في real-time applications (automotive, industrial)، غالباً بتحتاج RX pause فقط — يعني الـ link partner يقدر يطلب منك توقف، بس انت مش بتطلب منه. افهم الـ XOR logic في `mii_advertise_flowctrl` قبل ما تضبط الـ capabilities.

---

### السيناريو 5: Custom Board Bring-up على AM62x — 1000Base-X SFP مش بيتعرّف

#### العنوان
SFP module على AM62x بيظهر كـ 10/100 بدل 1000Base-X بسبب استخدام دوال الـ 1000Base-T بدل 1000Base-X

#### السياق
custom industrial board بالـ TI AM62x SoC وعليها SFP cage متوصل بـ 1000Base-X (Fiber) على MAC SGMII interface. الـ board engineer بيعمل bring-up وبيلاقي إن الـ `ethtool eth0` بيوري speed غلط — بيقول 100Mbps بدل 1000Mbps.

#### المشكلة
الـ engineer استخدم `mii_ctrl1000_mod_linkmode_adv_t()` لتحليل الـ LPA register بدل `mii_lpa_mod_linkmode_x()`. الأولى للـ 1000Base-T والتانية للـ 1000Base-X — registers مختلفة وتفسير bits مختلف.

#### التحليل
في 1000Base-T، الـ advertisement في `MII_CTRL1000` (reg 0x09):
```
Bit 9 (0x0200) = ADVERTISE_1000FULL
Bit 8 (0x0100) = ADVERTISE_1000HALF
```

في 1000Base-X، الـ advertisement في `MII_ADVERTISE` (reg 0x04) بس بـ تفسير مختلف للـ bits:
```
Bit 5 (0x0020) = ADVERTISE_1000XFULL   (نفس bit ADVERTISE_10HALF في T mode!)
Bit 6 (0x0040) = ADVERTISE_1000XHALF
Bit 7 (0x0080) = ADVERTISE_1000XPAUSE
Bit 8 (0x0100) = ADVERTISE_1000XPSE_ASYM
```

الكود الغلط في الـ driver:

```c
/* الغلط: استخدام دوال 1000Base-T مع 1000Base-X */
static void am62x_sfp_update_lpa(struct am62x_priv *priv)
{
    u32 stat1000 = priv->mii.mdio_read(priv->mii.dev,
                                        priv->mii.phy_id,
                                        MII_STAT1000); /* reg 0x0a */

    /* ده للـ 1000Base-T، مش للـ 1000Base-X! */
    mii_stat1000_mod_linkmode_lpa_t(priv->lp_advertising, stat1000);
    /* النتيجة: advertising هتبقى فاضية لأن الـ bits مش في MII_STAT1000 */
}
```

الكود الصح يستخدم `mii_lpa_mod_linkmode_x()`:

```c
static inline void mii_lpa_mod_linkmode_x(unsigned long *linkmodes, u16 lpa,
                                           int fd_bit)
{
    linkmode_mod_bit(ETHTOOL_LINK_MODE_Autoneg_BIT, linkmodes,
                     lpa & LPA_LPACK);              /* 0x4000 */
    linkmode_mod_bit(ETHTOOL_LINK_MODE_Pause_BIT, linkmodes,
                     lpa & LPA_1000XPAUSE);         /* 0x0080 */
    linkmode_mod_bit(ETHTOOL_LINK_MODE_Asym_Pause_BIT, linkmodes,
                     lpa & LPA_1000XPAUSE_ASYM);    /* 0x0100 */
    linkmode_mod_bit(fd_bit, linkmodes,
                     lpa & LPA_1000XFULL);          /* 0x0020 */
}
```

وللـ advertisement، الكود الغلط:
```c
/* الغلط */
u32 ctrl1000 = linkmode_adv_to_mii_ctrl1000_t(priv->advertising);
/* ctrl1000 بيكتب على reg 0x09 اللي مش مستخدم في 1000Base-X! */
```

#### الحل

```c
/* الصح: استخدم دوال الـ X mode للـ SFP/1000Base-X */
static void am62x_sfp_setup_autoneg(struct am62x_priv *priv)
{
    u16 adv;

    /*
     * للـ 1000Base-X: advertisement في MII_ADVERTISE (reg 0x04)
     * بس بـ X-mode bit mapping
     * fd_bit = ETHTOOL_LINK_MODE_1000baseX_Full_BIT
     */
    adv = linkmode_adv_to_mii_adv_x(priv->advertising,
                                      ETHTOOL_LINK_MODE_1000baseX_Full_BIT);

    priv->mii.mdio_write(priv->mii.dev, priv->mii.phy_id,
                          MII_ADVERTISE, adv);

    /* restart autoneg */
    mii_nway_restart(&priv->mii);
}

static void am62x_sfp_read_lpa(struct am62x_priv *priv)
{
    u16 lpa;

    /* في 1000Base-X، الـ LPA في MII_LPA (reg 0x05) */
    lpa = priv->mii.mdio_read(priv->mii.dev,
                               priv->mii.phy_id, MII_LPA);

    mii_lpa_mod_linkmode_x(priv->lp_advertising, lpa,
                            ETHTOOL_LINK_MODE_1000baseX_Full_BIT);
}
```

جدول المقارنة بين الـ modes:

| الـ Register | 1000Base-T | 1000Base-X |
|---|---|---|
| Advertisement | `MII_CTRL1000` (0x09) | `MII_ADVERTISE` (0x04) |
| LPA | `MII_STAT1000` (0x0a) | `MII_LPA` (0x05) |
| Full duplex bit | `ADVERTISE_1000FULL` (0x0200) | `ADVERTISE_1000XFULL` (0x0020) |
| الدالة المناسبة | `mii_ctrl1000_mod_linkmode_adv_t()` | `linkmode_adv_to_mii_adv_x()` |
| تحليل LPA | `mii_stat1000_mod_linkmode_lpa_t()` | `mii_lpa_mod_linkmode_x()` |

```bash
# Debug: اقرأ الـ registers مباشرة
phytool read eth0/0/0x04   # MII_ADVERTISE — هنا الـ advertisement في X mode
phytool read eth0/0/0x05   # MII_LPA — هنا الـ link partner ability في X mode
phytool read eth0/0/0x01   # MII_BMSR — هنا BMSR_ANEGCOMPLETE
```

#### الدرس المستفاد
الـ `mii.h` فيها مجموعتين منفصلتين من الدوال: واحدة للـ `1000Base-T` (بتستخدم `MII_CTRL1000` و `MII_STAT1000`) وواحدة للـ `1000Base-X` (بتستخدم `MII_ADVERTISE` و `MII_LPA` بـ تفسير bits مختلف). لما تيجي تدعم SFP أو SGMII fiber، استخدم دوال الـ `_x` suffix دايماً: `mii_lpa_mod_linkmode_x()` و `linkmode_adv_to_mii_adv_x()`. الخلط بينهم بيدي نتايج غلط بصمت بدون أي error messages.
## Phase 7: مصادر ومراجع

### توثيق رسمي في الـ kernel

| المصدر | الرابط | الوصف |
|--------|--------|-------|
| PHY Abstraction Layer | [docs.kernel.org/networking/phy.html](https://docs.kernel.org/networking/phy.html) | التوثيق الرئيسي لـ PHYLIB — يشرح الـ MDIO bus، الـ PHY driver، والـ autonegotiation |
| phylink | [kernel.org/.../sfp-phylink.html](https://www.kernel.org/doc/html/latest/networking/sfp-phylink.html) | الطبقة الحديثة اللي حلّت كتير من مشاكل الـ phylib القديمة |
| Netlink interface for ethtool | [static.lwn.net/.../ethtool-netlink.html](https://static.lwn.net/kerneldoc/networking/ethtool-netlink.html) | واجهة الـ netlink البديلة لـ ioctl في الـ ethtool |
| MDIO bus and PHYs in ACPI | [docs.kernel.org/.../acpi/dsd/phy.html](https://docs.kernel.org/firmware-guide/acpi/dsd/phy.html) | ربط الـ MDIO وPHY بالـ ACPI firmware |
| Networking index | [static.lwn.net/kerneldoc/networking/index.html](https://static.lwn.net/kerneldoc/networking/index.html) | فهرس كل توثيق الـ networking في الـ kernel |

**المسارات داخل الـ kernel source:**

```
Documentation/networking/phy.rst        — PHY Abstraction Layer الكامل
Documentation/networking/sfp-phylink.rst — دليل الهجرة من phylib لـ phylink
include/linux/mii.h                     — الـ header موضوع الدراسة
include/uapi/linux/mii.h                — ثوابت MII المكشوفة لـ userspace
drivers/net/phy/                        — كل PHY drivers
drivers/net/phy/phy.c                   — PHY core
drivers/net/phy/mdio_bus.c              — MDIO bus driver
```

---

### مقالات LWN.net

الـ **LWN.net** هو المرجع الأساسي لتاريخ قرارات الـ kernel:

| المقال | الرابط | الأهمية |
|--------|--------|---------|
| RFC: PHY Abstraction Layer II | [lwn.net/Articles/127013/](https://lwn.net/Articles/127013/) | أول RFC لبناء الـ PHYLIB — نقاش أصل المشكلة |
| PHY Abstraction Layer III | [lwn.net/Articles/144897/](https://lwn.net/Articles/144897/) | استكمال النقاش ودخول الـ phylib للـ mainline |
| Phylink & SFP support | [lwn.net/Articles/667055/](https://lwn.net/Articles/667055/) | ظهور الـ phylink كبديل أحدث للـ phylib |
| new ETHTOOL\_GLINKSETTINGS/SLINKSETTINGS API | [lwn.net/Articles/677122/](https://lwn.net/Articles/677122/) | إدخال `ethtool_link_ksettings` اللي بيستخدمه `mii.h` |
| RFC: new ETHTOOL\_GSETTINGS/SSETTINGS API | [lwn.net/Articles/631877/](https://lwn.net/Articles/631877/) | المناقشة الأولى لفكرة توسيع link mode masks لأكتر من 32 bit |
| ethtool netlink interface, part 1 | [lwn.net/Articles/783633/](https://lwn.net/Articles/783633/) | بداية استبدال ioctl بـ netlink في ethtool |
| Support MDIO devices | [lwn.net/Articles/670191/](https://lwn.net/Articles/670191/) | تعميم الـ MDIO bus لأجهزة غير PHY |
| MDIO and ethtool enhancements | [lwn.net/Articles/331581/](https://lwn.net/Articles/331581/) | تحسينات قديمة على الـ MDIO وethtool |
| Rust abstractions for network PHY drivers | [lwn.net/Articles/948948/](https://lwn.net/Articles/948948/) | كتابة PHY drivers بالـ Rust — مستقبل الـ subsystem |
| link\_ksettings API for virtual devices | [lwn.net/Articles/807789/](https://lwn.net/Articles/807789/) | helper functions للـ virtual network devices |
| Moving on from net-tools | [lwn.net/Articles/710533/](https://lwn.net/Articles/710533/) | لماذا انتهت حقبة mii-tool وجاء ethtool |

---

### kernelnewbies.org — تغييرات الـ kernel عبر الإصدارات

**الـ** kernelnewbies.org يوثّق التغييرات المهمة في كل إصدار:

| الإصدار | الرابط | التغيير المتعلق بـ MII/PHY |
|---------|--------|---------------------------|
| Linux 4.9 | [kernelnewbies.org/Linux_4.9](https://kernelnewbies.org/Linux_4.9) | تحسينات على الـ PHY subsystem |
| Linux 4.6 | [kernelnewbies.org/Linux_4.6](https://kernelnewbies.org/Linux_4.6) | تحديثات على الـ ethernet PHY drivers |
| Linux 4.3 | [kernelnewbies.org/Linux_4.3](https://kernelnewbies.org/Linux_4.3) | تحديثات networking stack |
| Linux 2.6.31 | [kernelnewbies.org/Linux_2_6_31](https://kernelnewbies.org/Linux_2_6_31) | إضافة non-MII PHY support لـ e100 |
| Linux 2.6.23 | [kernelnewbies.org/Linux_2_6_23](https://kernelnewbies.org/Linux_2_6_23) | إضافة ICPlus IP175C PHY driver، SGMII mode في m88e1111 |
| Linux 2.6.18 | [kernelnewbies.org/Linux_2_6_18](https://kernelnewbies.org/Linux_2_6_18) | إضافة SMSC LAN83C185 PHY driver |

---

### elinux.org — الـ Embedded Linux

الـ **elinux.org** مفيد للي بيشتغل على embedded systems مع PHY:

| الصفحة | الرابط | الفائدة |
|--------|--------|---------|
| BeagleBoard Zippy | [elinux.org/BeagleBoard_Zippy](https://elinux.org/BeagleBoard_Zippy) | مثال عملي لـ ethernet PHY على ARM board |
| Building BBB Kernel | [elinux.org/Building_BBB_Kernel](https://elinux.org/Building_BBB_Kernel) | بناء kernel للـ BeagleBone Black مع MDIO/PHY config |

---

### مناقشات الـ Mailing List ومصادر إضافية

```
# البحث في أرشيف lore.kernel.org
https://lore.kernel.org/netdev/?q=mii_if_info
https://lore.kernel.org/netdev/?q=linkmode+mii+adv
https://lore.kernel.org/netdev/?q=ethtool_link_ksettings

# أرشيف LKML القديم
https://lkml.org/lkml/2004/9/2/142   — أول إدخال phylib
```

**الـ mailing list الرئيسي:** `netdev@vger.kernel.org`
**أرشيف searchable:** [lore.kernel.org/netdev](https://lore.kernel.org/netdev/)

---

### man pages — الـ userspace tools

| الأداة | الرابط | الصلة |
|--------|--------|-------|
| mii-tool(8) | [man7.org/linux/man-pages/man8/mii-tool.8.html](https://man7.org/linux/man-pages/man8/mii-tool.8.html) | الأداة القديمة اللي تتكلم مع `mii_if_info` مباشرة |
| ethtool(8) | [man7.org/linux/man-pages/man8/ethtool.8.html](https://man7.org/linux/man-pages/man8/ethtool.8.html) | الأداة الحديثة البديلة — تستخدم `ethtool_link_ksettings` |

---

### الكتب المرجعية

#### Linux Device Drivers, 3rd Edition (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصل المهم:** Chapter 17 — Network Drivers
- **الرابط:** [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/) — متاح مجاناً
- **الصلة:** شرح بنية net_device، الـ NAPI، والـ mdio_read/mdio_write callbacks اللي في `mii_if_info`

#### Linux Kernel Development, 3rd Edition — Robert Love
- **الفصل:** Chapter 17 — Devices and Modules
- **الصلة:** فهم الـ driver model اللي بتشتغل عليه PHY drivers

#### Embedded Linux Primer, 2nd Edition — Christopher Hallinan
- **الفصل:** Chapter 15 — Embedded Drivers
- **الصلة:** إعداد الـ MDIO bus في embedded systems، device tree، وPHY configuration

#### Understanding Linux Network Internals — Christian Benvenuti
- **الأهمية:** الأعمق في تفاصيل الـ networking stack من MAC لـ PHY

---

### الـ Kernel Commits المهمة

```bash
# مشاهدة تاريخ mii.h بالكامل
git log --follow -- include/linux/mii.h

# أبرز commits:
# إضافة linkmode_adv_to_mii_adv_t وأخواتها (تحويل من ethtool_adv لـ linkmode)
git log --all --grep="linkmode.*mii" -- include/linux/mii.h

# إدخال mii_bmcr_encode_fixed
git log --all --grep="mii_bmcr_encode_fixed"

# إدخال mii_lpa_mod_linkmode_x
git log --all --grep="mii_lpa_mod_linkmode_x"
```

**الـ commits على kernel.org:**
- [git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/include/linux/mii.h)

---

### مصادر تقنية إضافية

| المصدر | الرابط | الوصف |
|--------|--------|-------|
| IEEE 802.3u Standard | ieee.org | المعيار الأصل لـ MII وقواعد الـ autonegotiation |
| TI E2E — PHY registers | [e2e.ti.com/.../1164499](https://e2e.ti.com/support/interface-group/interface/f/interface-forum/1164499/faq-how-to-read-and-write-ethernet-phy-registers-using-a-linux-terminal) | قراءة وكتابة PHY registers عملياً |
| Fun with ethtool | [linuxjournal.com/content/fun-ethtool](https://www.linuxjournal.com/content/fun-ethtool) | استخدام ethtool لمراقبة الـ MII registers |

---

### كلمات البحث

للاستزادة استخدم الكلمات دي في البحث:

```
# للـ kernel source
linux kernel MII PHY autonegotiation ethtool_link_ksettings
linux mii_if_info mdio_read mdio_write driver
linux linkmode advertisement PHY registers
linux BMCR BMSR MII registers kernel

# للـ hardware
IEEE 802.3u MII standard autonegotiation
MDIO bus SMI clause 22 clause 45
1000BASE-T SGMII RGMII PHY interface

# للـ mailing list
netdev mailing list mii linkmode
phylink sfp autonegotiation linux kernel
```
## Phase 8: Writing simple module

### الفكرة

**`mii_check_link`** هي دالة exported في `drivers/net/mii.c` — بتتحقق من state الـ PHY link عن طريق قراءة register الـ `MII_BMSR`. بنعمل لها kprobe عشان نشوف كل مرة kernel بيتحقق من link الـ network interface، ونطبع اسم الـ device وحالة الـ PHY ID.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * mii_check_link_probe.c
 * Attaches a kprobe to mii_check_link() to log every PHY link check.
 */
#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>      /* pr_info */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe */
#include <linux/mii.h>         /* struct mii_if_info */
#include <linux/netdevice.h>   /* struct net_device */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Explorer");
MODULE_DESCRIPTION("kprobe on mii_check_link to trace PHY link checks");

/*
 * pre_handler — runs just before mii_check_link() executes.
 *
 * الـ kprobe بيوفر الـ pt_regs اللي فيها arguments الدالة.
 * على x86-64: الـ argument الأول (struct mii_if_info *mii) موجود في rdi.
 */
static int mii_check_link_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * Retrieve the first argument: pointer to mii_if_info.
     * On x86-64 the first integer/pointer arg is in RDI.
     */
    struct mii_if_info *mii = (struct mii_if_info *)regs->di;

    /* Guard against NULL — defensive programming inside a probe */
    if (!mii || !mii->dev)
        return 0;

    /*
     * Print:
     *   - net_device name  (e.g. "eth0")
     *   - PHY ID           (address on the MDIO bus, e.g. 0 or 1)
     *   - full_duplex flag (1 = FDX, 0 = HDX)
     */
    pr_info("mii_probe: dev=%s phy_id=%d full_duplex=%u\n",
            mii->dev->name,
            mii->phy_id,
            mii->full_duplex);

    return 0; /* 0 = let the original function continue normally */
}

/* Define the kprobe structure */
static struct kprobe kp = {
    .symbol_name = "mii_check_link", /* kernel symbol to probe */
    .pre_handler = mii_check_link_pre,
};

static int __init mii_probe_init(void)
{
    int ret;

    /*
     * register_kprobe() patches one instruction in mii_check_link with a
     * breakpoint. When hit, pre_handler runs, then execution resumes.
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("mii_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("mii_probe: kprobe planted at %s (%px)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

static void __exit mii_probe_exit(void)
{
    /*
     * unregister_kprobe() removes the breakpoint and waits for any
     * running handler to finish — safe, no use-after-free.
     */
    unregister_kprobe(&kp);
    pr_info("mii_probe: kprobe removed\n");
}

module_init(mii_probe_init);
module_exit(mii_probe_exit);
```

---

### Makefile

```makefile
obj-m += mii_check_link_probe.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `<linux/module.h>` | الـ macros الأساسية لأي kernel module |
| `<linux/kprobes.h>` | `struct kprobe` وكل API الـ probing |
| `<linux/mii.h>` | `struct mii_if_info` — argument الدالة المستهدفة |
| `<linux/netdevice.h>` | `struct net_device` عشان نوصل لـ `dev->name` |

الـ includes دي ضرورية عشان نقدر نفسر الـ arguments بشكل صح من غير undefined behavior.

---

#### الـ `pre_handler`

الـ handler بياخد `struct pt_regs *regs` اللي بيمثل state الـ CPU عند نقطة الـ probe. على x86-64 الـ ABI بيحط الـ argument الأول في register الـ `rdi`، فبنعمل cast مباشر لـ `struct mii_if_info *`. الـ NULL check ضروري لأن الـ handler بيشتغل في kernel context وأي dereference خاطئ بيودي لـ kernel panic.

---

#### الـ `struct kprobe`

الـ `symbol_name` بيخلي الـ kernel يحل العنوان تلقائياً في وقت الـ `register_kprobe` من غير ما نحتاج نحط عنوان hardcoded. الـ `pre_handler` بيشتغل **قبل** أول instruction في الدالة، فبنشوف الـ arguments كاملة وسليمة.

---

#### الـ `module_exit` وأهمية الـ unregister

لو ما عملناش `unregister_kprobe` في الـ exit، الـ breakpoint بيفضل موجود في memory بس الـ handler code اتحذف مع الـ module — النتيجة: kernel crash فوري أول ما الدالة اتستدعت. الـ `unregister_kprobe` بتشيل الـ breakpoint وتستنى أي handler شغال يخلص قبل ما ترجع.

---

### تجربة عملية

```bash
# بناء وتحميل الـ module
make
sudo insmod mii_check_link_probe.ko

# لفرض link check: افصل وأعد توصيل cable أو استخدم ethtool
sudo ethtool -r eth0

# شوف الـ output
dmesg | grep mii_probe
# مثال على output:
# mii_probe: kprobe planted at mii_check_link (ffffffffc0a1b230)
# mii_probe: dev=eth0 phy_id=0 full_duplex=1

# إزالة الـ module
sudo rmmod mii_check_link_probe
dmesg | tail -3
# mii_probe: kprobe removed
```

---

### لماذا `mii_check_link` تحديداً؟

`mii_check_link` هي نقطة مركزية في stack الـ Ethernet — كل driver بيستخدم الـ generic MII layer بيناديها لما NAPI أو timer بيتحقق من حالة الـ link. Hooking عليها بيعطينا visibility على كل PHY في النظام في وقت واحد من غير ما نلمس أي driver.
