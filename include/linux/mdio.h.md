## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الـ `include/linux/mdio.h` جزء من subsystem اسمه **ETHERNET PHY LIBRARY** في الـ kernel، اللي بتشوفه في `MAINTAINERS` تحت اسم "ETHERNET PHY LIBRARY" — المسؤولون عنه Andrew Lunn و Heiner Kallweit.

---

### المشكلة اللي بيحلها — القصة الكاملة

تخيل عندك كارت شبكة (NIC) على اللاب توب، وجوزه chip صغير اسمه **PHY** (Physical Layer Transceiver). الـ PHY ده هو اللي بيتكلم مع الكابل الفعلي — بيحول البيتات الرقمية لإشارات كهربية أو ضوئية وبيرجعها تاني.

**السؤال:** كيف الـ CPU يقول للـ PHY "شيل السرعة لـ 1Gbps" أو "اعمل autonegotiation"؟

الجواب: عن طريق بروتوكول اسمه **MDIO** (Management Data Input/Output)، وده بروتوكول serial بسيط جداً — سلكين بس: MDC (clock) و MDIO (data). عن طريقيهم بتقدر تقرا وتكتب في registers جوه الـ PHY.

```
[ CPU / MAC ]
      |
      | MDIO bus (MDC + MDIO wires)
      |
  ----+----+----+----
  |        |        |
[PHY 0] [PHY 1] [PHY 2]    ← كل PHY ليه address من 0 لـ 31
```

---

### Clause 22 vs Clause 45 — ليه في ملف اسمه `mdio.h` منفصل؟

البروتوكول الأصلي اتعرّف في **Clause 22** من معيار IEEE 802.3 — بيدّي لكل PHY 32 register بس. كده كافي للـ 10/100/1000 Mbps التقليدية.

لما جه الـ 10 Gigabit والأسرع، الـ 32 register خلصت. فاتعمل نسخة جديدة اسمها **Clause 45** — بتفصل الـ PHY لأجزاء منطقية اسمها **MMDs** (MDIO Manageable Devices)، وكل MMD ليه مساحة registers منفصلة:

| MMD | الاسم | الوظيفة |
|-----|-------|---------|
| 1 | PMA/PMD | التعامل مع الوسيط الفيزيائي |
| 3 | PCS | Physical Coding Sublayer |
| 7 | AN | Auto-Negotiation |
| 30 | VEND1 | Vendor specific |

الملف `include/linux/mdio.h` ده هو **الـ header المركزي** اللي بيعرّف كل حاجة خاصة بالـ Clause 45 وبيوفر API موحد لكل الـ drivers.

---

### الـ `mdio.h` بيعمل إيه بالظبط؟

#### 1 - بيعرّف `struct mdio_device`

```c
struct mdio_device {
    struct device dev;        /* Linux device model integration */
    struct mii_bus *bus;      /* the MDIO bus this device sits on */
    int addr;                 /* bus address 0-31 */
    struct gpio_desc *reset_gpio;   /* optional reset GPIO */
    struct reset_control *reset_ctrl;
    /* ... */
};
```

ده الـ object اللي بيمثل أي device على الـ MDIO bus — مش PHY بس، ممكن يكون أي chip اتكلم عن طريق MDIO (زي بعض الـ switches).

#### 2 - بيعرّف `struct mdio_driver`

```c
struct mdio_driver {
    struct mdio_driver_common mdiodrv;
    int  (*probe)(struct mdio_device *mdiodev);
    void (*remove)(struct mdio_device *mdiodev);
    void (*shutdown)(struct mdio_device *mdiodev);
};
```

نفس نموذج الـ Linux driver model العادي — كل driver بيسجّل نفسه وبيشتغل لما تتطابق الـ device معاه.

#### 3 - بيعرّف `struct mdio_if_info`

```c
struct mdio_if_info {
    int prtad;           /* port address */
    u32 mmds;            /* bitmask of available MMDs */
    unsigned mode_support; /* C22, C45, or emulate C22 */
    struct net_device *dev;
    int (*mdio_read)(...);
    int (*mdio_write)(...);
};
```

ده interface بيستخدمه الـ NIC driver اللي بيعمل Clause 45 بنفسه من غير ما يمر بالـ PHY library الكاملة (زي بعض كروت 10G القديمة).

#### 4 - بيوفر Bus Access Functions

```c
/* بدون lock — لازم تعمل lock بإيدك */
int __mdiobus_read(struct mii_bus *bus, int addr, u32 regnum);
int __mdiobus_write(struct mii_bus *bus, int addr, u32 regnum, u16 val);

/* مع lock تلقائي */
int mdiobus_read(struct mii_bus *bus, int addr, u32 regnum);
int mdiobus_write(struct mii_bus *bus, int addr, u32 regnum, u16 val);

/* Clause 45 specifically */
int mdiobus_c45_read(struct mii_bus *bus, int addr, int devad, u32 regnum);
int mdiobus_c45_write(struct mii_bus *bus, int addr, int devad, u32 regnum, u16 val);
```

#### 5 - بيوفر Helper Functions للـ EEE و Auto-Negotiation

**EEE** هو Energy-Efficient Ethernet — بيخلي الـ PHY يدخل في وضع توفير طاقة لما مفيش data. الـ `mdio.h` فيه functions بتترجم بين register values وبين الـ `linkmode` bitmap اللي الـ kernel بيستخدمه:

```c
/* ترجمة EEE capability register → linkmode bits */
static inline void mii_eee_cap1_mod_linkmode_t(unsigned long *adv, u32 val);

/* ترجمة Clause 73 advertisement (بتاع الـ backplane 40G/100G) → linkmode */
static inline void mii_c73_mod_linkmode(unsigned long *adv, u16 *lpa);
```

---

### الـ Mutex Lock Classes — مشكلة الـ Nested Locking

الـ MDIO bus بتشيل mutex. المشكلة تيجي لما يكون عندك:

```
MDIO bus (normal)
    └── MDIO MUX (multiplexer chip, يتكلم بنفسه عبر MDIO)
            └── MDIO bus (behind mux)
                    └── PHY device
```

لو حاولت تعمل lock على الـ MUX وهو بالفعل جوه lock على الـ bus الرئيسي — هيحصل **deadlock** أو هينط الـ lockdep.

عشان كده الـ `mdio.h` بيعرّف:

```c
enum mdio_mutex_lock_class {
    MDIO_MUTEX_NORMAL,   /* الـ bus العادي */
    MDIO_MUTEX_MUX,      /* الـ MUX layer */
    MDIO_MUTEX_NESTED,   /* nested DSA layer */
};
```

---

### ASCII Diagram — الصورة الكاملة

```
Kernel Space
┌─────────────────────────────────────────────────────────┐
│                   Network Stack                         │
│                        │                               │
│              ┌─────────▼──────────┐                    │
│              │    NIC Driver      │  (e.g. igb, ixgbe) │
│              └─────────┬──────────┘                    │
│                        │  uses mii_bus                 │
│              ┌─────────▼──────────┐                    │
│              │   PHY Library      │  drivers/net/phy/  │
│              │  (phy_device.c)    │                    │
│              └─────────┬──────────┘                    │
│                        │  mdiobus_read/write           │
│              ┌─────────▼──────────┐                    │
│              │   mii_bus          │  struct mii_bus    │
│              │   (mdio bus core)  │  include/linux/phy.h│
│              └─────────┬──────────┘                    │
│                        │                               │
│         ┌──────────────┼──────────────┐                │
│         │              │              │                │
│  ┌──────▼────┐  ┌──────▼────┐  ┌─────▼─────┐         │
│  │mdio_device│  │mdio_device│  │mdio_device│         │
│  │ (PHY 0)   │  │ (PHY 1)   │  │(Switch)   │         │
│  └───────────┘  └───────────┘  └───────────┘         │
└─────────────────────────────────────────────────────────┘
                     MDIO Bus (hardware)
                  ┌──────┬──────┬──────┐
                 PHY0  PHY1  PHY2  Switch
```

---

### الفرق بين `mdio.h` و `mii.h` و `phy.h`

| الملف | بيعرّف إيه |
|-------|-----------|
| `include/linux/mii.h` | الـ Clause 22 القديم — `struct mii_if_info`، helper functions بسيطة |
| `include/uapi/linux/mdio.h` | register addresses وconstants معيارية (shared مع userspace) |
| `include/linux/mdio.h` | الـ file ده — Clause 45 structs، bus API، EEE helpers |
| `include/linux/phy.h` | الـ higher level — `struct phy_device`، `struct mii_bus` |

---

### الملفات المرتبطة اللي المفروض تعرفها

**Headers:**
- `include/linux/phy.h` — الـ `struct mii_bus` و `struct phy_device` اللي الكل بيستخدمهم
- `include/uapi/linux/mdio.h` — الـ register definitions المعيارية (shared مع userspace tools)
- `include/linux/mii.h` — الـ Clause 22 legacy interface
- `include/linux/linkmode.h` — الـ bitmaps اللي الـ EEE helpers بتحولّها

**Core Implementation:**
- `drivers/net/phy/phy_device.c` — الـ PHY device core، بيستخدم الـ mdio bus
- `drivers/net/mdio/of_mdio.c` — بيسجّل الـ MDIO bus من devicetree
- `drivers/net/mdio/mdio-bitbang.c` — software bitbang implementation للـ MDIO protocol

**Bus Controller Drivers (أمثلة):**
- `drivers/net/mdio/mdio-gpio.c` — MDIO عبر GPIO
- `drivers/net/mdio/mdio-bcm-unimac.c` — Broadcom UNIMAC MDIO controller
- `drivers/net/mdio/mdio-mux.c` — MDIO multiplexer support

**PHY Drivers (أمثلة):**
- `drivers/net/phy/broadcom.c` — Broadcom PHY driver
- `drivers/net/phy/marvell.c` — Marvell PHY driver

**Tracing:**
- `include/trace/events/mdio.h` — tracepoints للـ MDIO bus operations
## Phase 2: شرح الـ MDIO Framework

---

### المشكلة — ليه الـ MDIO Framework موجود؟

في أي نظام Ethernet، الـ **MAC** (المتحكم في الشبكة) محتاج يتكلم مع الـ **PHY** (الشريحة اللي بتوصل الكابل الفعلي). الكلام ده مش data — ده management: إيه السرعة؟ في link ولا لأ؟ enable auto-negotiation؟

قبل الـ MDIO framework، كل driver كان بيتعامل مع الـ PHY بطريقته الخاصة — كل واحد كتب read/write routines من الصفر. المشاكل كانت:

1. **تكرار الكود** — كل MAC driver بيعمل نفس الحاجة.
2. **لا يوجد device model** — الـ PHY مش represented في الـ kernel device tree صح.
3. **الـ MUX والـ DSA** — في بعض الأنظمة في MUX بين الـ MAC والـ PHY، أو Switch chip فيه PHYs جوه PHYs. من غير framework محدش يعرف يتعامل مع الـ nesting.
4. **Clause 45 vs Clause 22** — الـ IEEE 802.3 عنده نوعين من MDIO: القديم (22 register فقط) والجديد (address space كبير جداً مقسم بـ MMDs). محتاج abstraction يخبي الفرق ده.

---

### الحل — الـ MDIO Framework

الـ kernel حل المشكلة بإنه عمل **bus abstraction كامل** للـ MDIO، زي ما عمل مع I2C وSPI تماماً:

- الـ **`mii_bus`** = الـ bus نفسه (الأسلاك + الـ controller اللي بيتحكم فيها).
- الـ **`mdio_device`** = أي device موصول على الـ bus (PHY أو switch أو غيره).
- الـ **`mdio_driver`** = الـ driver اللي بيتعامل مع device معين.
- الـ **`mdio_if_info`** = واجهة بسيطة للـ MAC لما مش محتاج يشتغل مع الـ full bus model.

---

### التشبيه الواقعي — شبكة USB بس للـ PHY

تخيل إن الـ MDIO زي الـ USB:

| الـ MDIO | الـ USB |
|---|---|
| `mii_bus` | الـ USB Host Controller (xHCI) |
| `mdio_device` (addr 0-31) | USB device بـ address معين |
| `mdio_driver` | USB device driver |
| Clause 22 register (0-31) | USB control endpoint |
| Clause 45 MMD + register | USB interface + endpoint |
| `mdio_lock` mutex | USB bus arbitration |
| `mdio_map[32]` | الـ device table في الـ host controller |

**التعمق في التشبيه:**
- زي ما الـ USB host controller عنده `read_transfer()` و`write_transfer()` — الـ `mii_bus` عنده `->read()` و`->write()` function pointers اللي بينفذهم الـ MAC driver.
- زي ما في USB mux chips، في MDIO مشبوك عليه `MDIO_MUTEX_MUX` class لإن الـ locking بيتعمل على أكتر من level.
- الـ probe/remove lifecycle في `mdio_driver` طبق على طبق زي الـ USB driver.

---

### الصورة الكبيرة — مكان الـ MDIO في الـ kernel

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Space                              │
│              ethtool / ip link / network tools                  │
└───────────────────────────┬─────────────────────────────────────┘
                            │ syscalls
┌───────────────────────────▼─────────────────────────────────────┐
│                      Kernel - Networking                        │
│  ┌──────────────┐   ┌──────────────┐   ┌─────────────────────┐ │
│  │  net_device  │   │   ethtool    │   │  phylib (phy.h)     │ │
│  │  (MAC side)  │   │  subsystem   │   │  phy_device / state │ │
│  └──────┬───────┘   └──────┬───────┘   └──────────┬──────────┘ │
│         │                  │                       │            │
│         └──────────────────▼───────────────────────┘            │
│                    ┌───────────────────┐                        │
│                    │  mdio_if_info     │  ← واجهة مبسطة للـ MAC │
│                    │  (clause 45 API)  │                        │
│                    └────────┬──────────┘                        │
└─────────────────────────────┼───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│                    MDIO Bus Framework                           │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                struct mii_bus                           │   │
│   │  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐  │   │
│   │  │ ->read() │  │->write() │  │  mdio_map[0..31]     │  │   │
│   │  │->read_c45│  │->write_c45  │  (mdio_device ptrs)  │  │   │
│   │  └──────────┘  └──────────┘  └──────────────────────┘  │   │
│   │  mdio_lock (mutex)   stats[32]   reset_gpiod            │   │
│   └─────────────────────────────────────────────────────────┘   │
│          │                │                   │                  │
│   ┌──────▼──────┐  ┌──────▼──────┐   ┌───────▼──────┐          │
│   │mdio_device  │  │mdio_device  │   │ mdio_device  │          │
│   │  addr=0     │  │  addr=1     │   │  addr=N      │          │
│   │  (PHY)      │  │  (Switch)   │   │  (other)     │          │
│   └──────┬──────┘  └──────┬──────┘   └──────────────┘          │
│          │                │                                      │
│   ┌──────▼──────┐  ┌──────▼──────┐                              │
│   │phy_driver   │  │dsa_driver   │                              │
│   │(mdio_driver)│  │(mdio_driver)│                              │
│   └─────────────┘  └─────────────┘                              │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│                    Hardware Layer                               │
│                                                                 │
│   MAC Controller (e.g. stmmac, fec, bcmgenet)                  │
│   ┌──────────────────────────────────┐                          │
│   │ MDIO Controller (inside MAC)     │                          │
│   │  MDC clock pin ──────────────────┼──► PHY MDC              │
│   │  MDIO data pin ◄─────────────────┼──► PHY MDIO             │
│   └──────────────────────────────────┘                          │
└─────────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المركزية

الفكرة الجوهرية هي إن الـ **MDIO bus هو bus حقيقي في الـ kernel device model**، مش مجرد helper functions.

ده معناه:
- كل device عليه بيتسجل في الـ sysfs.
- الـ driver binding بيتعمل تلقائياً عن طريق `mdio_bus_type`.
- الـ power management و reset بيتديروا automatically.
- الـ nesting (MUX على MUX على bus) بيتعالج بـ `mdio_mutex_lock_class`.

---

### الـ Structs المحورية — تفاصيل

#### 1. `struct mii_bus` — الـ Bus نفسه

```c
struct mii_bus {
    /* الـ MAC driver بيملاهم عشان يقدر يقرأ/يكتب على الـ hardware */
    int (*read)(struct mii_bus *bus, int addr, int regnum);
    int (*write)(struct mii_bus *bus, int addr, int regnum, u16 val);

    /* Clause 45: لازم تحدد الـ MMD (devnum) قبل الـ register */
    int (*read_c45)(struct mii_bus *bus, int addr, int devnum, int regnum);
    int (*write_c45)(struct mii_bus *bus, int addr, int devnum, int regnum, u16 val);

    struct mutex mdio_lock;          /* bus arbitration — واحد بس بيتكلم في وقت واحد */
    struct mdio_device *mdio_map[PHY_MAX_ADDR]; /* 32 slot للـ devices */
    struct device dev;               /* kernel device model integration */
    u32 phy_mask;                    /* أي addresses تتجاهلهم في الـ probe */
};
```

**ملاحظة مهمة:** الـ `mdio_lock` بيتأخذ بـ `mdiobus_read()` / `mdiobus_write()` (الـ locked versions). الـ `__mdiobus_read()` / `__mdiobus_write()` (بالـ double underscore) بيشتغلوا من غير lock — للي معاه الـ lock بالفعل من برا.

#### 2. `struct mdio_device` — أي Device على الـ Bus

```c
struct mdio_device {
    struct device dev;           /* وراثة من kernel device model */
    struct mii_bus *bus;         /* مين بوسه على الـ bus ده */
    int addr;                    /* عنوانه على الـ bus (0-31) */
    int flags;                   /* MDIO_DEVICE_FLAG_PHY لو PHY */

    /* Reset management */
    struct gpio_desc *reset_gpio;
    struct reset_control *reset_ctrl;
    unsigned int reset_assert_delay;   /* microseconds */
    unsigned int reset_deassert_delay; /* microseconds */

    /* callbacks بيعملهم الـ bus code */
    int (*bus_match)(struct device *dev, const struct device_driver *drv);
    void (*device_free)(struct mdio_device *mdiodev);
    void (*device_remove)(struct mdio_device *mdiodev);
};
```

كل PHY في الـ kernel هو `mdio_device` قبل ما يبقى `phy_device`. الـ `phy_device` بيعمل `container_of` على الـ `mdio_device`.

#### 3. `struct mdio_driver` — الـ Driver لأي MDIO Device

```c
struct mdio_driver {
    struct mdio_driver_common mdiodrv;  /* فيه struct device_driver + flags */

    int (*probe)(struct mdio_device *mdiodev);    /* device اتلاقى */
    void (*remove)(struct mdio_device *mdiodev);  /* device اترفع */
    void (*shutdown)(struct mdio_device *mdiodev);/* system shutdown */
};
```

الـ `mdio_driver_common` عنده `flag = MDIO_DEVICE_FLAG_PHY` لو الـ driver ده خاص بـ PHY — ده بيخلي الـ `phylib` يتعامل معاه بشكل مختلف.

#### 4. `struct mdio_if_info` — الـ Clause 45 Helper للـ MAC

```c
struct mdio_if_info {
    int prtad;        /* Port address — MDIO_PRTAD_NONE لو مجهول */
    u32 mmds;         /* bitmask: أي MMDs موجودة في الـ PHY */
    unsigned mode_support; /* MDIO_SUPPORTS_C22 | MDIO_SUPPORTS_C45 | MDIO_EMULATE_C22 */

    struct net_device *dev;
    int (*mdio_read)(struct net_device *dev, int prtad, int devad, u16 addr);
    int (*mdio_write)(struct net_device *dev, int prtad, int devad, u16 addr, u16 val);
};
```

ده بيُستخدم في الـ MAC drivers القديمة أو اللي بتتعامل مع 10G PHYs مباشرةً من غير ما تستخدم الـ phylib الكاملة. الـ `MDIO_EMULATE_C22` مهم جداً: لو الـ PHY C45 بس والـ software بيطلب C22 registers، الـ framework بيعمل translation تلقائي.

---

### العلاقة بين الـ Structs

```
struct mii_bus
    │
    ├── mdio_map[0]  ──►  struct mdio_device  (addr=0)
    │                         │
    │                         ├── struct device dev
    │                         │       └── (parent = mii_bus->dev)
    │                         ├── struct mii_bus *bus  (back-pointer)
    │                         ├── int addr = 0
    │                         └── reset_gpio / reset_ctrl
    │
    ├── mdio_map[1]  ──►  struct mdio_device  (addr=1)
    │                         │
    │                     [phy_device embeds mdio_device]
    │                     struct phy_device {
    │                         struct mdio_device mdio;  ← أول field
    │                         ...
    │                     }
    │
    └── mdio_lock  (mutex protecting all reads/writes)


struct mdio_driver
    └── struct mdio_driver_common mdiodrv
            └── struct device_driver driver
                    └── (registered in mdio_bus_type)
```

---

### Clause 22 vs Clause 45 — الفرق الجوهري

**الـ Clause 22** (القديم — IEEE 802.3, 1998):
- 32 device address (PRTAD: bits 4:0).
- 32 register لكل device.
- مجموع: 1024 register فقط.
- المشكلة: 10G PHYs محتاجة الاف الـ registers.

**الـ Clause 45** (الجديد — IEEE 802.3ae, 2002):
- نفس 32 device address.
- كل device عنده **16 MMD** (DEVAD: device address داخل الـ PHY).
- كل MMD عنده **65536 register** (16-bit address).
- المجموع: أكتر من مليون register.

```
Clause 22 addressing:
  [PRTAD: 5 bits] [REGNUM: 5 bits]
         │               │
   PHY address 0-31   Register 0-31

Clause 45 addressing:
  [PRTAD: 5 bits] [DEVAD: 5 bits] [REGNUM: 16 bits]
         │               │                │
   PHY address 0-31   MMD 0-31      Register 0-65535
```

**الـ MMDs الأساسية:**
| DEVAD | الاسم | الوظيفة |
|-------|-------|---------|
| 1 | PMA/PMD | Physical Medium Attachment — التحكم في الـ analog layer |
| 2 | WIS | WAN Interface Sublayer — للـ SONET framing |
| 3 | PCS | Physical Coding Sublayer — encoding/decoding |
| 4 | PHY XS | PHY Extender Sublayer — للـ XAUI |
| 7 | AN | Auto-Negotiation — التفاوض على السرعة |
| 30 | Vendor 1 | خاص بالـ vendor |
| 31 | Vendor 2 | خاص بالـ vendor |

---

### الـ Locking Model — المستويات المتداخلة

ده من أصعب الأجزاء لأن الـ MDIO bus ممكن يكون فيه nesting:

```
Normal case:
  MAC driver → mdiobus_read() → takes mdio_lock → reads PHY

MUX case (e.g. mux chip between MAC and multiple PHYs):
  MAC driver → mdiobus_read() → takes mdio_lock (MDIO_MUTEX_NORMAL)
                                    → MUX driver → takes mdio_lock (MDIO_MUTEX_MUX)
                                                      → reads PHY

DSA case (switch chip with internal PHYs):
  MAC → mii_bus (MDIO_MUTEX_NORMAL)
    → DSA switch (MDIO_MUTEX_MUX)
      → internal PHY (MDIO_MUTEX_NESTED)
```

الـ `enum mdio_mutex_lock_class` بيخلي الـ kernel lockdep يفهم إن ده valid nested locking مش deadlock.

---

### الـ API Layers — من الأعلى للأسفل

#### Layer 1: Device-aware API (الأسهل للاستخدام)

```c
/* بيستخدم mdiodev->bus و mdiodev->addr تلقائياً */
int mdiodev_read(struct mdio_device *mdiodev, u32 regnum);
int mdiodev_write(struct mdio_device *mdiodev, u32 regnum, u16 val);

/* Clause 45 */
int mdiodev_c45_read(struct mdio_device *mdiodev, int devad, u16 regnum);
int mdiodev_c45_write(struct mdio_device *mdiodev, u32 devad, u16 regnum, u16 val);
```

#### Layer 2: Bus-level API (محتاج تحدد الـ addr بنفسك)

```c
/* Locked versions — الـ normal case */
int mdiobus_read(struct mii_bus *bus, int addr, u32 regnum);
int mdiobus_write(struct mii_bus *bus, int addr, u32 regnum, u16 val);

/* Unlocked versions — لما عندك الـ lock بالفعل */
int __mdiobus_read(struct mii_bus *bus, int addr, u32 regnum);
int __mdiobus_write(struct mii_bus *bus, int addr, u32 regnum, u16 val);
```

#### Layer 3: Modify helpers (read-modify-write atomic)

```c
/* بيعمل: read → (val & ~mask) | set → write */
int mdiobus_modify(struct mii_bus *bus, int addr, u32 regnum, u16 mask, u16 set);

/* نفسه بس بيرجع 1 لو الـ value اتغير فعلاً */
int mdiobus_modify_changed(struct mii_bus *bus, int addr, u32 regnum, u16 mask, u16 set);
```

---

### الـ EEE Helpers — Translation Layer

الـ **EEE** (Energy Efficient Ethernet) محتاج تترجم بين 3 أنظمة:
1. **MDIO registers** (Clause 45, register 3.20, 7.60, 7.61, etc.)
2. **ethtool legacy** (`SUPPORTED_*` و `ADVERTISED_*` بيتتعاملوا بـ u32 bitmask)
3. **linkmode** (الجديد — `unsigned long *` bitmap مع `ETHTOOL_LINK_MODE_*_BIT` constants)

الـ `mdio.h` بيوفر helpers للـ translation دي في الاتجاهين:

```c
/* من MDIO register → ethtool legacy */
u32 mmd_eee_cap_to_ethtool_sup_t(u16 eee_cap);

/* من MDIO register → linkmode */
void mii_eee_cap1_mod_linkmode_t(unsigned long *adv, u32 val);
void mii_eee_cap2_mod_linkmode_sup_t(unsigned long *adv, u32 val);

/* من linkmode → MDIO register */
u32 linkmode_to_mii_eee_cap1_t(unsigned long *adv);
u32 linkmode_to_mii_eee_cap2_t(unsigned long *adv);
```

**مثال واقعي — driver بيقرأ EEE capabilities من 10G PHY:**

```c
u16 cap_reg;
unsigned long adv[__ETHTOOL_LINK_MODE_MASK_NBITS / BITS_PER_LONG] = {};

/* اقرأ register 3.20 (EEE capability) */
cap_reg = mdiodev_c45_read(mdiodev, MDIO_MMD_PCS, MDIO_PCS_EEE_ABLE);

/* حوّله لـ linkmode bits */
mii_eee_cap1_mod_linkmode_t(adv, cap_reg);

/* دلوقتي adv فيه ETHTOOL_LINK_MODE_1000baseT_Full_BIT etc. */
```

---

### ما بيملكه الـ Framework وما بيفوضه

| الـ Framework يملكه | يفوضه للـ Driver |
|---|---|
| Device registration في sysfs | تنفيذ `->read()` و`->write()` على الـ hardware |
| Bus arbitration (mdio_lock) | إزاي بالظبط بتبعت MDC/MDIO pulses |
| Driver binding (probe/remove) | معنى كل register في الـ PHY |
| Reset sequencing (gpio/reset_ctrl) | Autonegotiation policy |
| Statistics per device | Interrupt handling |
| Clause 45 emulation of C22 | Vendor-specific extensions |
| Translation helpers (EEE, linkmode) | phy_device state machine (ده phylib) |

**ملاحظة:** الـ MDIO framework **لا يفهم** معنى الـ registers. هو بس بيوصّل. من يفهمهم هو الـ `phylib` (الـ PHY library) اللي بيشتغل فوق الـ MDIO framework.

---

### مثال كامل — إزاي الـ MAC Driver بيسجل الـ MDIO Bus

```c
/* في probe() الخاص بالـ MAC driver */
static int my_mac_probe(struct platform_device *pdev)
{
    struct mii_bus *bus;

    /* allocate the bus struct */
    bus = mdiobus_alloc();

    /* الـ MAC driver بيملي الـ function pointers */
    bus->read    = my_mac_mdio_read;   /* bitbangs or uses MMIO registers */
    bus->write   = my_mac_mdio_write;
    bus->name    = "my-mac-mdio";
    bus->parent  = &pdev->dev;

    snprintf(bus->id, MII_BUS_ID_SIZE, "%s-mdio", pdev->name);

    /* الـ framework بيعمل probe لكل address 0-31 */
    mdiobus_register(bus);  /* هنا بيلاقي الـ PHYs */

    /* بعد ده الـ phylib بياخد الأمور */
}

/* الـ framework بيعمل نفسه: */
static void mdiobus_scan(struct mii_bus *bus, int addr)
{
    /* بيحاول يقرأ PHY ID registers */
    /* لو لاقى device، بيعمل mdio_device_create() وbيسجله */
    /* الـ phylib بعدين بيعمل phy_device_create() فوقيه */
}
```

---

### ملخص سريع

```
المشكلة:  كل MAC driver بيتعامل مع PHY بطريقته — لا توحيد، لا device model
الحل:     MDIO bus framework — زي I2C بالظبط بس للـ PHY management
الأجزاء:  mii_bus (الـ bus) + mdio_device (الـ device) + mdio_driver (الـ driver)
الـ C22:  32 reg فقط — للـ 10/100/1000M PHYs القديمة
الـ C45:  16 MMDs × 65K reg — للـ 10G+ PHYs
الـ EEE:  helpers لتحويل capabilities بين MDIO regs و ethtool و linkmode
الـ Lock:  mdio_lock mutex + lock classes للـ nested MUX/DSA scenarios
```
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags، Enums، وخيارات الـ Configuration — Cheatsheet

#### الـ `mdio_mutex_lock_class` enum

| القيمة | المعنى |
|--------|--------|
| `MDIO_MUTEX_NORMAL` | الـ lock العادي للـ bus |
| `MDIO_MUTEX_MUX` | الـ lock لطبقة الـ MUX |
| `MDIO_MUTEX_NESTED` | الـ lock للـ nested layers (مثلاً DSA) |

#### الـ `mode_support` flags في `mdio_if_info`

| الـ Flag | القيمة | المعنى |
|----------|--------|--------|
| `MDIO_SUPPORTS_C22` | `1` | يدعم Clause 22 مباشرةً |
| `MDIO_SUPPORTS_C45` | `2` | يدعم Clause 45 |
| `MDIO_EMULATE_C22` | `4` | يترجم Clause 22 registers لـ Clause 45 تلقائياً |

#### الـ `MDIO_DEVICE_FLAG_PHY` flag

| الـ Flag | القيمة | المعنى |
|----------|--------|--------|
| `MDIO_DEVICE_FLAG_PHY` | `1` | يشير لأن الـ driver هو PHY driver |

#### الـ PHY ID Encoding (Clause 45)

| الـ Macro | الـ Bits | المعنى |
|-----------|----------|--------|
| `MDIO_PHY_ID_C45` | `bit 15` | يشير لأن الـ ID هو Clause 45 |
| `MDIO_PHY_ID_PRTAD` | `bits 9:5` | الـ Port Address (0-31) |
| `MDIO_PHY_ID_DEVAD` | `bits 4:0` | الـ Device Address (MMD) |

#### الـ MMD (Manageable Device) Addresses

| الـ Macro | القيمة | الطبقة |
|-----------|--------|--------|
| `MDIO_MMD_PMAPMD` | `1` | Physical Medium Attachment/Dependent |
| `MDIO_MMD_WIS` | `2` | WAN Interface Sublayer |
| `MDIO_MMD_PCS` | `3` | Physical Coding Sublayer |
| `MDIO_MMD_PHYXS` | `4` | PHY Extender Sublayer |
| `MDIO_MMD_AN` | `7` | Auto-Negotiation |
| `MDIO_MMD_VEND1` | `30` | Vendor Specific 1 |
| `MDIO_MMD_VEND2` | `31` | Vendor Specific 2 |

#### الـ EEE Capability Bits (registers 3.20 / 7.60 / 7.61)

| الـ Flag | المعنى |
|----------|--------|
| `MDIO_EEE_100TX` | EEE عند 100BASE-TX |
| `MDIO_EEE_1000T` | EEE عند 1000BASE-T |
| `MDIO_EEE_10GT` | EEE عند 10GBASE-T |
| `MDIO_EEE_1000KX` | EEE عند 1000BASE-KX |
| `MDIO_EEE_10GKX4` | EEE عند 10GBASE-KX4 |
| `MDIO_EEE_10GKR` | EEE عند 10GBASE-KR |
| `MDIO_EEE_2_5GT` | EEE عند 2.5GBASE-T (register 3.21) |
| `MDIO_EEE_5GT` | EEE عند 5GBASE-T (register 3.21) |

#### الـ `mdio_device.reset_state` القيم المتوقعة

الـ `reset_state` بيتتبع حالة الـ GPIO reset (1 = asserted، 0 = deasserted). ده بيُستخدم داخلياً لتجنب تكرار العمليات.

---

### 1. الـ Structs الأساسية

#### `struct mdio_device`

**الغرض:** يمثّل أي device موصول على الـ MDIO bus — سواء كان PHY أو switch أو غيره. هو الـ base object اللي بيقدر ينتمي لأي driver على الـ bus.

```c
struct mdio_device {
    struct device dev;          /* embedded kernel device — بيربطه بالـ device model */

    struct mii_bus *bus;        /* الـ bus اللي الـ device موصول عليه */

    /* callbacks خاصة بالـ bus matching والـ lifecycle */
    int  (*bus_match)(struct device *dev, const struct device_driver *drv);
    void (*device_free)(struct mdio_device *mdiodev);
    void (*device_remove)(struct mdio_device *mdiodev);

    int  addr;                  /* عنوان الـ device على الـ bus (0–31) */
    int  flags;                 /* flags عامة */
    int  reset_state;           /* الحالة الحالية لـ reset line */

    struct gpio_desc    *reset_gpio;         /* GPIO الخاص بالـ reset */
    struct reset_control *reset_ctrl;        /* reset controller بديل للـ GPIO */
    unsigned int reset_assert_delay;         /* مدة الانتظار بعد assert بالـ µs */
    unsigned int reset_deassert_delay;       /* مدة الانتظار بعد deassert */
};
```

**الربط مع structs أخرى:**
- بيحتوي على `struct device` embedded — ده بيربطه بالـ Linux device model بالكامل.
- بيشاور على `struct mii_bus` — الـ bus الأم.
- الـ `struct phy_device` بيكون أكبر struct بيحتوي `struct mdio_device` embedded فيه.

---

#### `struct mdio_driver_common`

**الغرض:** الـ base struct المشترك بين كل أنواع الـ MDIO drivers (سواء generic أو PHY).

```c
struct mdio_driver_common {
    struct device_driver driver;   /* embedded kernel driver */
    int flags;                     /* MDIO_DEVICE_FLAG_PHY لو ده PHY driver */
};
```

**الربط:** كل `struct mdio_driver` وكل `struct phy_driver` بيحتوي على `mdio_driver_common` embedded.

---

#### `struct mdio_driver`

**الغرض:** يمثّل الـ driver لأي MDIO device غير PHY (مثلاً Ethernet switches، clock chips، وغيرها على الـ MDIO bus).

```c
struct mdio_driver {
    struct mdio_driver_common mdiodrv;   /* الـ base — لازم يكون أول field */

    int  (*probe)   (struct mdio_device *mdiodev);   /* اكتشاف وتهيئة الـ device */
    void (*remove)  (struct mdio_device *mdiodev);   /* تحرير الموارد */
    void (*shutdown)(struct mdio_device *mdiodev);   /* إيقاف الـ device عند الـ shutdown */
};
```

**الربط:** بيستخدم `to_mdio_driver()` macro للتحويل من `device_driver` pointer للـ `mdio_driver`.

---

#### `struct mdio_if_info`

**الغرض:** واجهة الـ MDIO الخاصة بالـ Ethernet NIC controller — بتوصل الـ NIC driver بالـ PHY عبر Clause 45. مختلفة عن `mdio_device` لأنها بتمثّل طريقة تواصل الـ MAC مع الـ PHY مش الـ device نفسه.

```c
struct mdio_if_info {
    int  prtad;        /* Port address (MDIO_PRTAD_NONE = -1 لو مجهول) */
    u32  mmds;         /* bitmask للـ MMDs الموجودة في الـ PHY */
    unsigned mode_support; /* MDIO_SUPPORTS_C22 | C45 | EMULATE_C22 */

    struct net_device *dev;   /* الـ NIC المرتبطة */

    /* function pointers للـ read/write على الـ hardware */
    int (*mdio_read) (struct net_device *dev, int prtad, int devad, u16 addr);
    int (*mdio_write)(struct net_device *dev, int prtad, int devad, u16 addr, u16 val);
};
```

**الربط:**
- بيشاور على `struct net_device` — الـ NIC الأم.
- بيُستخدم مع `mdio45_*` و `mdio_mii_ioctl()` API.
- مستقل تماماً عن `struct mii_bus` — ده abstraction مختلف لنفس الـ MDIO bus من منظور الـ MAC/NIC.

---

### 2. رسم العلاقات بين الـ Structs (ASCII Diagram)

```
Linux Device Model
══════════════════════════════════════════════════════════

  struct bus_type (mdio_bus_type)
       │
       │  يحتوي devices وdrivers
       │
  ┌────┴──────────────────────────────────────────────┐
  │                                                    │
  ▼                                                    ▼
struct mii_bus                           struct mdio_driver_common
  │  (defined in linux/phy.h)              │  .driver → device_driver
  │  .mdio_map[32] ──────────────────────► │
  │  .mutex (mdio_mutex_lock_class)        │
  │                                        │     ┌──────────────────┐
  │  bيحمل array من:                       │     │ struct mdio_driver│
  │                                        │     │  .mdiodrv ◄──────┤
  │  ┌──────────────────────────────┐      │     │  .probe()         │
  │  │     struct mdio_device       │      │     │  .remove()        │
  │  │       .dev (device)          │      │     │  .shutdown()      │
  │  │       .bus ──────────────────┘      │     └──────────────────┘
  │  │       .addr (0-31)           │
  │  │       .reset_gpio            │
  │  │       .reset_ctrl            │      مثال: phy_device يمتد منه:
  │  │       .flags                 │
  │  │       .reset_state           │      ┌──────────────────────────┐
  │  └──────────────┬───────────────┘      │    struct phy_device      │
  │                 │                      │      .mdio ◄── embedded   │
  │                 │ container_of         │      .drv                 │
  │                 ▼                      │      .attached_dev        │
  │          struct phy_device             └──────────────────────────┘
  │          (أكبر struct بيحتوي mdio_device)
  │
  └──────────────────────────────────────────────────────


NIC/MAC منظور (مستقل):

  struct net_device
       │
       │ (NIC driver بيحمل هذا الـ struct)
       ▼
  struct mdio_if_info
       │  .prtad, .mmds, .mode_support
       │  .dev ──────────────────────► struct net_device (back pointer)
       │  .mdio_read()   ─────────────► hardware registers مباشرة
       │  .mdio_write()
       │
       │ بيُستخدم في:
       ▼
  mdio45_probe()
  mdio45_links_ok()
  mdio45_nway_restart()
  mdio45_ethtool_ksettings_get()
  mdio_mii_ioctl()
```

---

### 3. رسم دورة الحياة (Lifecycle Diagrams)

#### دورة حياة `mdio_device`

```
                mdio_device Lifecycle
═══════════════════════════════════════════════════════

  [Discovery / Probe Phase]
           │
           ▼
  mdio_device_create(bus, addr)
    → kzalloc(struct mdio_device)
    → dev_set_name()
    → device_initialize()
    → .bus    = bus
    → .addr   = addr
           │
           ▼
  mdio_device_register(mdiodev)
    → mdiobus_register_device()   ← بيضيفه على الـ bus map
    → device_add()                ← بيعلن عنه للـ kernel device model
           │
           │  الـ bus بيبحث عن driver مناسب
           ▼
  bus_match() → driver->probe(mdiodev)
    → mdiodev_set_drvdata()        ← بيخزن private data
           │
           │  [Runtime]
           ▼
  mdiodev_read() / mdiodev_write()
  mdiodev_modify() / mdiodev_c45_read() ...
           │
           │  [Teardown]
           ▼
  mdio_device_remove(mdiodev)
    → device_remove()
    → mdiobus_unregister_device()
           │
           ▼
  mdio_device_free(mdiodev) / mdio_device_put()
    → put_device()
    → kfree()
```

#### دورة حياة `mdio_driver`

```
  [Module Load]
       │
       ▼
  mdio_module_driver(my_driver)
    → module_init → mdio_driver_register(&my_driver)
         → driver_register()
         → .driver.bus = &mdio_bus_type
       │
       │  [Per Device]
       ▼
  bus يطابق driver مع device
    → mdio_driver.probe(mdiodev)
       │
       │  [Module Unload]
       ▼
  mdio_driver_unregister(&my_driver)
    → driver_unregister()
    → لكل device مرتبط: driver.remove(mdiodev)
```

#### دورة حياة `mdio_if_info` (NIC side)

```
  [NIC Driver Init]
       │
       ▼
  يملأ struct mdio_if_info:
    .prtad = <phy port address>
    .mmds  = <bitmask of MMDs>
    .mode_support = MDIO_SUPPORTS_C45 | MDIO_EMULATE_C22
    .dev   = netdev
    .mdio_read  = my_hw_read
    .mdio_write = my_hw_write
       │
       ▼
  mdio45_probe(mdio, prtad)
    → يتحقق من وجود الـ MMDs المتوقعة
       │
       ▼
  [Runtime]
  mdio45_links_ok()
  mdio45_nway_restart()
  mdio45_ethtool_ksettings_get()
  mdio_mii_ioctl()
       │
       ▼
  [NIC Driver Remove]
  لا توجد teardown خاصة — بس الـ struct بيتحرر مع الـ NIC
```

---

### 4. Call Flow Diagrams

#### قراءة register عبر `mdiodev_read()`

```
Driver Code
    │
    ▼
mdiodev_read(mdiodev, regnum)
    │  inline wrapper — بيجيب bus وaddr من الـ struct
    ▼
mdiobus_read(bus, addr, regnum)
    │  يأخذ bus->mutex (MDIO_MUTEX_NORMAL)
    ▼
__mdiobus_read(bus, addr, regnum)
    │  يتحقق من الـ bus state
    ▼
bus->read(bus, addr, regnum)
    │  هي الـ hardware callback المسجلة في struct mii_bus
    ▼
[Hardware MDIO Controller]
    → يرسل MDIO frame على السلك
    → ينتظر الـ response
    → يرجع الـ 16-bit register value
```

#### قراءة Clause 45 register عبر `mdiodev_c45_read()`

```
Driver Code
    │
    ▼
mdiodev_c45_read(mdiodev, devad, regnum)
    │  inline — بيجيب bus وaddr
    ▼
mdiobus_c45_read(bus, addr, devad, regnum)
    │  يأخذ bus->mutex
    ▼
__mdiobus_c45_read(bus, addr, devad, regnum)
    │  يبني الـ C45 address من devad + regnum
    ▼
bus->read_c45(bus, addr, devad, regnum)
    │  أو يحول لـ C22 عبر address extension لو الـ hardware لا يدعم C45 نيتيف
    ▼
[Hardware]
```

#### modify register (read-modify-write atomic)

```
mdiodev_modify(mdiodev, regnum, mask, set)
    │
    ▼
mdiobus_modify(bus, addr, regnum, mask, set)
    │  يأخذ bus->mutex — لازم يكون atomic
    ▼
__mdiobus_read()      ← قرأ القيمة الحالية
    │
    ▼
new_val = (old_val & ~mask) | (set & mask)
    │
    ▼
__mdiobus_write()     ← كتب القيمة الجديدة
    │  تحت نفس الـ mutex
    ▼
يرجع 0 لو نجح
```

#### NIC Driver ioctl flow (ethtool / SIOCGMIIREG)

```
userspace (ethtool)
    │
    ▼
kernel: dev_ioctl() → .ndo_eth_ioctl()
    │
    ▼
mdio_mii_ioctl(mdio_if_info, mii_ioctl_data, cmd)
    │  بيتحقق من الـ cmd (SIOCGMIIREG / SIOCSMIIREG)
    │  بيتحقق من الـ mode_support
    ▼
mdio->mdio_read(dev, prtad, devad, addr)
    │  أو mdio->mdio_write() حسب الـ cmd
    ▼
NIC hardware registers (مباشرة، مش عبر mii_bus)
```

#### EEE capability translation flow

```
PHY Driver يقرأ register 3.20
    │
    ▼
val = mdiodev_c45_read(mdiodev, MDIO_MMD_PCS, MDIO_PCS_EEE_ABLE)
    │
    ▼
mii_eee_cap1_mod_linkmode_t(adv_linkmode_bitmap, val)
    │  بيترجم كل bit من الـ register لـ ETHTOOL_LINK_MODE_xxx bit
    ▼
linkmode bitmap جاهز للـ phylib / ethtool
```

---

### 5. استراتيجية الـ Locking

الـ MDIO bus هو shared resource — كل الـ devices على نفس الـ bus بيشتركوا في نفس السلك، فلازم يكون الوصول متسلسل.

#### الـ Mutex في `struct mii_bus`

الـ `mii_bus` بيحتوي على `mutex` من نوع `mutex` بيتحكم في كل الوصول للـ bus:

| النوع | الـ Function | الـ Lock |
|-------|-------------|---------|
| **Locked** (عادي) | `mdiobus_read()` | يأخذ `bus->mutex` |
| **Locked** (عادي) | `mdiobus_write()` | يأخذ `bus->mutex` |
| **Locked** (عادي) | `mdiobus_modify()` | يأخذ `bus->mutex` طول الـ RMW |
| **Unlocked** | `__mdiobus_read()` | لا يأخذ lock — المتصل مسؤول |
| **Unlocked** | `__mdiobus_write()` | لا يأخذ lock — المتصل مسؤول |
| **Nested** | `mdiobus_read_nested()` | `mutex_lock_nested(MDIO_MUTEX_NESTED)` |
| **Nested** | `mdiobus_write_nested()` | `mutex_lock_nested(MDIO_MUTEX_NESTED)` |

#### متى تستخدم الـ Nested variants؟

الـ MDIO topology أحياناً بتكون متداخلة:

```
MAC
 └─► mii_bus (real hardware)
       └─► MDIO MUX device    ← بيتحكم في اختيار الـ sub-bus
             ├─► sub-bus A
             │     └─► PHY 1
             └─► sub-bus B
                   └─► PHY 2   ← DSA switch ports
```

لما الـ MUX driver بيحاول يكتب على الـ sub-bus، هو بالفعل داخل lock على الـ parent bus. لو استخدم `mdiobus_read()` العادي هيحصل deadlock. الحل هو:

```c
/* داخل MUX driver — هو بالفعل hold الـ parent bus lock */
mdiobus_read_nested(child_bus, addr, reg);
/* بيستخدم mutex_lock_nested(MDIO_MUTEX_MUX) لتجنب الـ deadlock */
```

#### الـ `__mdiobus_*` (Unlocked) — متى تستخدمها؟

لما تكون أنت بالفعل عارف إن الـ bus lock محجوز (مثلاً جوه `bus->read()` callback نفسه، أو في سياق يضمن exclusive access). الـ `mdiobus_modify()` نفسه بيستخدمهم داخلياً:

```c
/* داخل mdiobus_modify() — بعد ما أخد الـ mutex */
int mdiobus_modify(struct mii_bus *bus, int addr, u32 regnum, u16 mask, u16 set)
{
    mutex_lock(&bus->mutex);
    ret = __mdiobus_read(bus, addr, regnum);   /* unlocked version */
    new = (ret & ~mask) | (set & mask);
    __mdiobus_write(bus, addr, regnum, new);   /* unlocked version */
    mutex_unlock(&bus->mutex);
}
```

#### ترتيب الـ Locks (Lock Ordering)

```
لو عندك MUX + DSA + normal layers:

parent_bus->mutex  (MDIO_MUTEX_NORMAL)
    └─► mux_bus->mutex (MDIO_MUTEX_MUX)
          └─► nested_bus->mutex (MDIO_MUTEX_NESTED)

دايماً من الـ parent للـ child — عكس الترتيب ده بيسبب deadlock.
```

#### الـ `mdiodev_*` wrappers والـ Locking

الـ `mdiodev_read()` وأخواته (الـ non-`__` versions) كلهم بيأخذوا الـ lock تلقائياً. الـ `__mdiodev_*` versions بيمرروا للـ `__mdiobus_*` بدون lock:

```c
/* locked — آمن من أي context */
static inline int mdiodev_read(struct mdio_device *mdiodev, u32 regnum)
{
    return mdiobus_read(mdiodev->bus, mdiodev->addr, regnum);
}

/* unlocked — المتصل مسؤول عن الـ lock */
static inline int __mdiodev_read(struct mdio_device *mdiodev, u32 regnum)
{
    return __mdiobus_read(mdiodev->bus, mdiodev->addr, regnum);
}
```

#### جدول ملخص الـ Locking

| الـ Function | الـ Lock | متى تستخدمه |
|-------------|---------|-------------|
| `mdiobus_read/write` | يأخذ mutex عادي | الاستخدام الافتراضي |
| `mdiobus_read/write_nested` | يأخذ mutex nested | داخل MUX/DSA callbacks |
| `__mdiobus_read/write` | لا lock | أنت عارف إن الـ lock محجوز |
| `mdiobus_modify` | يأخذ mutex ويعمل RMW atomic | تعديل bits بأمان |
| `mdiobus_modify_changed` | نفس modify + يرجع إذا تغيرت القيمة | optimization |
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### مجموعة: Device Lifecycle

| Function | النوع | الغرض |
|---|---|---|
| `mdio_device_create()` | exported | ينشئ `mdio_device` جديد على bus معين |
| `mdio_device_register()` | exported | يسجل الـ device في kernel device model |
| `mdio_device_remove()` | exported | يزيل الـ device من الـ bus |
| `mdio_device_free()` | exported | يحرر memory الـ device |
| `mdio_device_get()` | inline | يزيد الـ refcount |
| `mdio_device_put()` | inline | يخفض الـ refcount (يستدعي `free`) |
| `mdio_device_reset()` | exported | يتحكم في reset GPIO/reset_control |

#### مجموعة: Driver Registration

| Function | النوع | الغرض |
|---|---|---|
| `mdio_driver_register()` | exported | يسجل الـ `mdio_driver` في الـ subsystem |
| `mdio_driver_unregister()` | exported | يلغي تسجيل الـ driver |
| `mdio_module_driver()` | macro | shortcut لـ `module_init`/`module_exit` |

#### مجموعة: Driver Data Helpers

| Function | النوع | الغرض |
|---|---|---|
| `mdiodev_set_drvdata()` | inline | يحفظ driver-private pointer في الـ device |
| `mdiodev_get_drvdata()` | inline | يسترجع driver-private pointer |

#### مجموعة: PHY ID Decoding (Clause 45)

| Function | النوع | الغرض |
|---|---|---|
| `mdio_phy_id_is_c45()` | inline | يتحقق إذا الـ ID هو clause 45 |
| `mdio_phy_id_prtad()` | inline | يستخرج الـ PRTAD من الـ phy_id |
| `mdio_phy_id_devad()` | inline | يستخرج الـ DEVAD من الـ phy_id |

#### مجموعة: Clause 45 Bus I/O (Unlocked)

| Function | النوع | الغرض |
|---|---|---|
| `__mdiobus_read()` | exported | قراءة register بدون lock |
| `__mdiobus_write()` | exported | كتابة register بدون lock |
| `__mdiobus_modify()` | exported | read-modify-write بدون lock |
| `__mdiobus_modify_changed()` | exported | modify + يرجع إذا تغيرت القيمة |

#### مجموعة: Clause 45 Bus I/O (Locked)

| Function | النوع | الغرض |
|---|---|---|
| `mdiobus_read()` | exported | قراءة مع mutex lock |
| `mdiobus_write()` | exported | كتابة مع mutex lock |
| `mdiobus_modify()` | exported | modify مع mutex lock |
| `mdiobus_modify_changed()` | exported | modify + flag التغيير مع lock |
| `mdiobus_read_nested()` | exported | قراءة مع nested lock |
| `mdiobus_write_nested()` | exported | كتابة مع nested lock |

#### مجموعة: Clause 45 (C45) Explicit Access

| Function | النوع | الغرض |
|---|---|---|
| `__mdiobus_c45_read()` | exported | C45 قراءة بدون lock |
| `__mdiobus_c45_write()` | exported | C45 كتابة بدون lock |
| `mdiobus_c45_read()` | exported | C45 قراءة مع lock |
| `mdiobus_c45_write()` | exported | C45 كتابة مع lock |
| `mdiobus_c45_read_nested()` | exported | C45 قراءة nested |
| `mdiobus_c45_write_nested()` | exported | C45 كتابة nested |
| `mdiobus_c45_modify()` | exported | C45 modify مع lock |
| `mdiobus_c45_modify_changed()` | exported | C45 modify + flag |

#### مجموعة: Per-Device Wrappers (`mdiodev_*`)

| Function | النوع | الغرض |
|---|---|---|
| `__mdiodev_read()` | inline | unlocked قراءة من device مباشرة |
| `__mdiodev_write()` | inline | unlocked كتابة من device |
| `__mdiodev_modify()` | inline | unlocked modify من device |
| `__mdiodev_modify_changed()` | inline | unlocked modify+change flag |
| `mdiodev_read()` | inline | locked قراءة من device |
| `mdiodev_write()` | inline | locked كتابة من device |
| `mdiodev_modify()` | inline | locked modify |
| `mdiodev_modify_changed()` | inline | locked modify+change flag |
| `__mdiodev_c45_read()` | inline | C45 unlocked read |
| `__mdiodev_c45_write()` | inline | C45 unlocked write |
| `mdiodev_c45_read()` | inline | C45 locked read |
| `mdiodev_c45_write()` | inline | C45 locked write |
| `mdiodev_c45_modify()` | inline | C45 locked modify |
| `mdiodev_c45_modify_changed()` | inline | C45 locked modify+change |

#### مجموعة: Bus Registry Helpers

| Function | النوع | الغرض |
|---|---|---|
| `mdiobus_register_device()` | exported | يسجل device في bus map |
| `mdiobus_unregister_device()` | exported | يزيله من bus map |
| `mdiobus_is_registered_device()` | exported | يتحقق وجود device على address معين |
| `mdiobus_get_phy()` | exported | يجيب `phy_device` من bus بالـ address |

#### مجموعة: Clause 45 mdio_if_info (Ethernet Controller Interface)

| Function | النوع | الغرض |
|---|---|---|
| `mdio45_probe()` | exported | يكتشف MMDs الموجودة في PHY |
| `mdio_set_flag()` | exported | يضبط/يمسح bit في register |
| `mdio45_links_ok()` | exported | يتحقق الـ link status من MMDs |
| `mdio45_nway_restart()` | exported | يعيد تشغيل auto-negotiation |
| `mdio45_ethtool_ksettings_get_npage()` | exported | يملأ `ethtool_link_ksettings` |
| `mdio45_ethtool_ksettings_get()` | inline | wrapper بدون next page |
| `mdio_mii_ioctl()` | exported | يخدم SIOCGMIIPHY/SIOCSMIIREG من userspace |

#### مجموعة: EEE / Linkmode Translators

| Function | النوع | الغرض |
|---|---|---|
| `mmd_eee_cap_to_ethtool_sup_t()` | inline | MMD EEE cap → ethtool supported |
| `mmd_eee_adv_to_ethtool_adv_t()` | inline | MMD EEE adv → ethtool adv |
| `ethtool_adv_to_mmd_eee_adv_t()` | inline | ethtool adv → MMD EEE register |
| `mii_eee_cap1_mod_linkmode_t()` | inline | EEE cap1 register → linkmode |
| `mii_eee_cap2_mod_linkmode_sup_t()` | inline | EEE cap2 (3.21) → linkmode |
| `mii_eee_cap2_mod_linkmode_adv_t()` | inline | EEE adv2 (7.62/7.63) → linkmode |
| `linkmode_to_mii_eee_cap1_t()` | inline | linkmode → EEE adv1 register |
| `linkmode_to_mii_eee_cap2_t()` | inline | linkmode → EEE adv2 register |

#### مجموعة: 10GBT / BASE-T1 / C73 Translators

| Function | النوع | الغرض |
|---|---|---|
| `linkmode_adv_to_mii_10gbt_adv_t()` | inline | linkmode → 10GBT AN CTRL bits |
| `mii_10gbt_stat_mod_linkmode_lpa_t()` | inline | 10GBT AN STATUS → linkmode LPA |
| `mii_t1_adv_l_mod_linkmode_t()` | inline | BASE-T1 ADV_L register → linkmode |
| `mii_t1_adv_m_mod_linkmode_t()` | inline | BASE-T1 ADV_M register → linkmode |
| `linkmode_adv_to_mii_t1_adv_l_t()` | inline | linkmode → BASE-T1 ADV_L |
| `linkmode_adv_to_mii_t1_adv_m_t()` | inline | linkmode → BASE-T1 ADV_M |
| `mii_10base_t1_adv_mod_linkmode_t()` | inline | 10BASE-T1 AN STATUS → linkmode |
| `linkmode_adv_to_mii_10base_t1_t()` | inline | linkmode → 10BASE-T1 AN CTRL |
| `mii_c73_mod_linkmode()` | inline | Clause 73 adv array → linkmode |

---

### Group 1: Device Lifecycle

هي الـ functions المسؤولة عن إنشاء، تسجيل، وتدمير الـ `mdio_device`. الـ `mdio_device` هو أي device متصل على MDIO bus سواء كان PHY أو أي MDIO peripheral تاني. الـ lifetime يبدأ بـ `create` وينتهي بـ `free`.

---

#### `mdio_device_create()`

```c
struct mdio_device *mdio_device_create(struct mii_bus *bus, int addr);
```

بيعمل allocate لـ `struct mdio_device` جديد وبيربطه بالـ `mii_bus` المحدد. بيضبط `dev.bus` على `&mdio_bus_type` ويحدد الـ `addr` (0-31). الـ device في البداية مش مسجل في الـ kernel device model.

- **`bus`**: الـ `mii_bus` اللي هيتضاف عليه الـ device.
- **`addr`**: MDIO bus address بتاع الـ device (0-31).
- **Return**: pointer للـ `mdio_device` الجديد، أو `ERR_PTR()` في حالة فشل الـ allocation.
- **Key details**: بيستدعي `device_initialize()` داخليًا. الـ device مش visible في `/sys` لحد ما `mdio_device_register()` يتستدعى.
- **Caller context**: process context فقط. بيتستدعى عادةً من bus scan code أو من driver probe function لما بيعمل child devices.

---

#### `mdio_device_register()`

```c
int mdio_device_register(struct mdio_device *mdiodev);
```

بيضيف الـ device للـ kernel device model عن طريق `device_add()`. بعد الاستدعاء، الـ `uevent` بيتبعت، والـ driver core بيحاول يـ match driver مناسب. أي device matching تتم عن طريق `mdiodev->bus_match` callback.

- **`mdiodev`**: الـ device المُنشأ بـ `mdio_device_create()`.
- **Return**: `0` success، أو error code سالب (`-EBUSY`, `-ENOMEM`, etc.).
- **Key details**: بعد نجاح الاستدعاء، الـ device مرئي في `/sys/bus/mdio_bus/devices/`. الـ refcount بيتضاف عن طريق الـ device core.
- **Caller context**: process context فقط (بينام).

---

#### `mdio_device_remove()`

```c
void mdio_device_remove(struct mdio_device *mdiodev);
```

بيعمل `device_del()` لإزالة الـ device من الـ bus وإلغاء أي driver binding موجود. الـ `remove` callback بتاع الـ driver بتتستدعى قبل الفصل. مش بيحرر الـ memory، لازم `mdio_device_free()` بعده.

- **`mdiodev`**: الـ device المراد إزالته.
- **Return**: void.
- **Key details**: الاستدعاء آمن حتى لو الـ device مش registered. بيشغل `device->device_remove` callback اللي موجودة في الـ struct.
- **Caller context**: process context.

---

#### `mdio_device_free()`

```c
void mdio_device_free(struct mdio_device *mdiodev);
```

بيخفض الـ reference count للـ device. لما الـ count يوصل صفر، الـ memory بتتحرر عن طريق `put_device()`. الـ `device_free` callback الموجودة في الـ struct بتتستدعى للـ cleanup.

- **`mdiodev`**: الـ device المراد تحريره.
- **Return**: void.
- **Key details**: لا تستدعيها مباشرة إلا لو مؤكد إن الـ device مش registered. الـ standard path هو `mdio_device_put()`.
- **Caller context**: أي context (بيُستدعى من finalizer).

---

#### `mdio_device_get()` / `mdio_device_put()`

```c
static inline void mdio_device_get(struct mdio_device *mdiodev);
static inline void mdio_device_put(struct mdio_device *mdiodev);
```

**الـ `get`** بيزيد الـ reference count عن طريق `get_device()`. **الـ `put`** بيستدعي `mdio_device_free()` مباشرةً — انتبه: الـ `put` هنا مش هو `put_device()` العادي، هو مجرد alias للـ `free`.

- **Key details**: الـ design ده بيعكس إن الـ `mdio_device` بيشيل lifetime تاعه في `mdio_device_free()` مش في generic `put_device()`.

---

#### `mdio_device_reset()`

```c
void mdio_device_reset(struct mdio_device *mdiodev, int value);
```

بيتحكم في hardware reset للـ device. لو الـ device عنده `reset_ctrl`، بيستخدم الـ reset framework. لو عنده `reset_gpio`، بيستخدم GPIO. بيراعي الـ `reset_assert_delay` و`reset_deassert_delay` delays.

```c
/* pseudocode */
mdio_device_reset(dev, 1):  // assert
    if reset_ctrl  -> reset_control_assert()
    elif reset_gpio -> gpiod_set_value(gpio, 1)
    msleep(reset_assert_delay)

mdio_device_reset(dev, 0):  // deassert
    if reset_ctrl  -> reset_control_deassert()
    elif reset_gpio -> gpiod_set_value(gpio, 0)
    msleep(reset_deassert_delay)
```

- **`value`**: `1` = assert reset، `0` = deassert reset.
- **Key details**: الـ `reset_state` field بيتتبع الحالة الحالية لتجنب تكرار نفس الـ operation.
- **Caller context**: process context (بينام بسبب delays).

---

### Group 2: Driver Registration

بتتعامل مع تسجيل الـ `mdio_driver` في الـ MDIO bus subsystem. الـ `mdio_driver` بيكون مسؤول عن non-PHY MDIO devices (الـ PHYs بتتسجل بـ `phy_driver_register`).

---

#### `mdio_driver_register()`

```c
int mdio_driver_register(struct mdio_driver *drv);
```

بيسجل الـ `mdio_driver` في kernel driver model عن طريق `driver_register()` على `mdio_bus_type`. بعده، الـ bus بيعمل probe لأي device مُسجَّل يناسبه.

- **`drv`**: الـ `mdio_driver` اللي `mdiodrv.driver.name` بتاعه لازم تكون محددة.
- **Return**: `0` success، أو error code.
- **Key details**: بيستدعي `mdiodrv.driver.bus = &mdio_bus_type` قبل التسجيل. الـ `flags` field في `mdio_driver_common` بيحدد النوع (`MDIO_DEVICE_FLAG_PHY` لو PHY).
- **Caller context**: process context، عادةً من `module_init()`.

---

#### `mdio_driver_unregister()`

```c
void mdio_driver_unregister(struct mdio_driver *drv);
```

بيزيل الـ driver من الـ bus، وبيستدعي `remove` callback لكل device مربوط بيه.

- **Return**: void.
- **Caller context**: process context، من `module_exit()`.

---

#### `mdio_module_driver()` (macro)

```c
#define mdio_module_driver(_mdio_driver) \
    module_driver(_mdio_driver, mdio_driver_register, mdio_driver_unregister)
```

الـ macro ده shortcut نظيف لأي MDIO driver بسيط. بيولّد `module_init` و`module_exit` functions أوتوماتيك.

**مثال استخدام:**
```c
static struct mdio_driver my_mdio_drv = {
    .mdiodrv.driver.name = "my-mdio-dev",
    .probe  = my_probe,
    .remove = my_remove,
};
mdio_module_driver(my_mdio_drv);
```

---

### Group 3: Driver Data Helpers

ده pattern standard في الـ Linux kernel لتخزين driver-private data مع الـ device.

---

#### `mdiodev_set_drvdata()` / `mdiodev_get_drvdata()`

```c
static inline void mdiodev_set_drvdata(struct mdio_device *mdio, void *data);
static inline void *mdiodev_get_drvdata(struct mdio_device *mdio);
```

مجرد wrappers على `dev_set_drvdata()` / `dev_get_drvdata()` بتاخد `mdio_device` بدل `device` مباشرةً. بتستخدم `&mdio->dev` كالـ key في الـ device's private data pointer.

- **`data`**: أي pointer لـ driver-specific structure.
- **Key details**: zero overhead — fully inlined. لا locking لأن بتتستدعى فقط في `probe` context.

**مثال:**
```c
/* في probe */
struct my_priv *priv = kzalloc(sizeof(*priv), GFP_KERNEL);
mdiodev_set_drvdata(mdiodev, priv);

/* في أي function تانية */
struct my_priv *priv = mdiodev_get_drvdata(mdiodev);
```

---

### Group 4: PHY ID Decoding (Clause 45)

الـ Clause 45 بيستخدم عنونة مختلفة عن Clause 22. الـ phy_id في Clause 45 بيضم كل من الـ **PRTAD** (Port Address، 5 bits) والـ **DEVAD** (Device Address، 5 bits) مضغوطين في integer واحد.

```
phy_id layout (C45):
  bit[5]    : MDIO_PHY_ID_C45 flag
  bits[4:0] : DEVAD (device address / MMD)
  bits[9:5] : PRTAD (port address)
```

---

#### `mdio_phy_id_is_c45()`

```c
static inline bool mdio_phy_id_is_c45(int phy_id);
```

بيتحقق إن الـ `phy_id` هو clause 45 identifier صحيح: لازم يكون `MDIO_PHY_ID_C45` bit مضبوط، وما فيش bits خارج `MDIO_PHY_ID_C45_MASK`.

- **Return**: `true` لو C45، `false` لو Clause 22 أو invalid.
- **Caller context**: بيتستدعى من `phy_device` creation code عشان يعرف الـ addressing mode.

---

#### `mdio_phy_id_prtad()`

```c
static inline __u16 mdio_phy_id_prtad(int phy_id);
```

بيستخرج الـ PRTAD (bits 9:5) بعد shift يمين 5. الـ PRTAD هو الـ physical port address على الـ MDIO bus (0-31).

---

#### `mdio_phy_id_devad()`

```c
static inline __u16 mdio_phy_id_devad(int phy_id);
```

بيستخرج الـ DEVAD (bits 4:0) عن طريق mask بـ `MDIO_PHY_ID_DEVAD`. الـ DEVAD بيحدد الـ MMD (مثلًا `MDIO_MMD_PCS = 3`، `MDIO_MMD_AN = 7`).

---

### Group 5: Bus I/O — Unlocked (double underscore prefix)

كل الـ functions اللي بتبدأ بـ `__` بتفترض إن الـ caller مسك الـ `mii_bus->mdio_lock` بالفعل. دي بتستخدم في contexts أو scenarios لازم فيها تعمل atomic sequence من operations.

---

#### `__mdiobus_read()`

```c
int __mdiobus_read(struct mii_bus *bus, int addr, u32 regnum);
```

بيقرأ register من device على الـ MDIO bus مباشرةً بدون أخذ الـ mutex. بيستدعي `bus->read()` op بتاع الـ bus controller.

- **`bus`**: الـ `mii_bus` instance.
- **`addr`**: MDIO address (0-31).
- **`regnum`**: رقم الـ register. للـ Clause 22 ده 5-bit value. للـ C45 بيكون encoded.
- **Return**: قيمة الـ register (16-bit في int)، أو قيمة سالبة لو error.
- **Locking**: **caller لازم يمسك `bus->mdio_lock`**.
- **Caller context**: يُستدعى من within locked MDIO transactions، مثلًا من `phy_read()` بعد ما الـ bus اتأمن.

---

#### `__mdiobus_write()`

```c
int __mdiobus_write(struct mii_bus *bus, int addr, u32 regnum, u16 val);
```

نفس `__mdiobus_read()` بس للكتابة. بيستدعي `bus->write()`.

- **`val`**: الـ 16-bit value المراد كتابته.
- **Return**: `0` success، أو error code.
- **Locking**: نفس الـ locked requirement.

---

#### `__mdiobus_modify()`

```c
int __mdiobus_modify(struct mii_bus *bus, int addr, u32 regnum, u16 mask, u16 set);
```

بيعمل read-modify-write بدون lock. بيقرأ الـ register، بيمسح الـ bits المحددة بـ `mask`، وبيضبط الـ bits الجديدة بـ `set`.

```c
/* pseudocode */
val = __mdiobus_read(bus, addr, regnum);
if (val < 0) return val;
new_val = (val & ~mask) | (set & mask);
return __mdiobus_write(bus, addr, regnum, new_val);
```

- **`mask`**: الـ bits اللي هتتمسح (0 = ignore، 1 = clear).
- **`set`**: الـ bits الجديدة اللي هتتكتب في مكان الـ mask.
- **Locking**: caller يمسك الـ lock.

---

#### `__mdiobus_modify_changed()`

```c
int __mdiobus_modify_changed(struct mii_bus *bus, int addr, u32 regnum, u16 mask, u16 set);
```

زي `__mdiobus_modify()` بالظبط بس بيرجع `1` لو القيمة اتغيرت فعلًا، `0` لو نفسها، أو error. مفيد لتجنب unnecessary writes وتتبع التغييرات.

- **Return**: `1` إذا تغيرت القيمة، `0` إذا لم تتغير، error code سالب للفشل.

---

### Group 6: Bus I/O — Locked

نفس الـ functions السابقة بس بتاخد وبترجع `bus->mdio_lock` أوتوماتيك. الـ standard path لأي driver عادي.

---

#### `mdiobus_read()` / `mdiobus_write()`

```c
int mdiobus_read(struct mii_bus *bus, int addr, u32 regnum);
int mdiobus_write(struct mii_bus *bus, int addr, u32 regnum, u16 val);
```

بيأخذوا الـ `bus->mdio_lock` mutex، بيعملوا العملية، وبيطلقوا الـ lock. الـ **standard API** لأي MDIO access من driver code.

- **Locking**: داخليًا بيستدعوا `mutex_lock(&bus->mdio_lock)` / `mutex_unlock()`.
- **Caller context**: process context — بينام على الـ lock.

---

#### `mdiobus_read_nested()` / `mdiobus_write_nested()`

```c
int mdiobus_read_nested(struct mii_bus *bus, int addr, u32 regnum);
int mdiobus_write_nested(struct mii_bus *bus, int addr, u32 regnum, u16 val);
```

نفس الـ locked versions بس بيستخدموا `mutex_lock_nested()` بـ `MDIO_MUTEX_NESTED` class. ضروريين لـ scenarios زي DSA أو MUX layer لما يكون في bus داخل bus وبيمسكوا نفس الـ mutex class في stack مختلف.

- **Key details**: بدون ده، الـ lockdep هيشكي من "possible circular locking dependency" في nested MDIO bus scenarios.

---

#### `mdiobus_modify()` / `mdiobus_modify_changed()`

```c
int mdiobus_modify(struct mii_bus *bus, int addr, u32 regnum, u16 mask, u16 set);
int mdiobus_modify_changed(struct mii_bus *bus, int addr, u32 regnum, u16 mask, u16 set);
```

الـ locked versions من `__mdiobus_modify()` و`__mdiobus_modify_changed()`. الأكثر استخدامًا في driver code لتعديل individual bits في register.

**مثال:**
```c
/* enable loopback bit بدون تأثير على باقي الـ bits */
mdiobus_modify(bus, addr, MDIO_CTRL1,
               0, MDIO_PCS_CTRL1_LOOPBACK);
```

---

### Group 7: Clause 45 Explicit Bus I/O

دي functions مخصصة صراحةً لـ Clause 45 addressing اللي بيستخدم `devad` (Device Address / MMD) كـ parameter إضافي. الـ Clause 22 بيخبي الـ devad داخل الـ address encoding، لكن C45 صريح.

---

#### `__mdiobus_c45_read()` / `__mdiobus_c45_write()`

```c
int __mdiobus_c45_read(struct mii_bus *bus, int addr, int devad, u32 regnum);
int __mdiobus_c45_write(struct mii_bus *bus, int addr, int devad, u32 regnum, u16 val);
```

بيقرأوا/يكتبوا C45 register بدون lock. الـ `devad` بيحدد الـ MMD (مثلًا `MDIO_MMD_AN = 7`). الـ `regnum` بيكون 16-bit register address داخل الـ MMD.

- **`devad`**: الـ MMD identifier (0-31). مثلًا `MDIO_MMD_PCS = 3`.
- **`regnum`**: register address داخل الـ MMD المختار.
- **Locking**: caller يمسك الـ lock.

---

#### `mdiobus_c45_read()` / `mdiobus_c45_write()`

```c
int mdiobus_c45_read(struct mii_bus *bus, int addr, int devad, u32 regnum);
int mdiobus_c45_write(struct mii_bus *bus, int addr, int devad, u32 regnum, u16 val);
```

نفس السابقين بس مع lock. الـ standard C45 access API.

**مثال:**
```c
/* قراءة EEE advertisement من MMD AN */
val = mdiobus_c45_read(bus, phy_addr,
                        MDIO_MMD_AN, MDIO_AN_EEE_ADV);
```

---

#### `mdiobus_c45_modify()` / `mdiobus_c45_modify_changed()`

```c
int mdiobus_c45_modify(struct mii_bus *bus, int addr, int devad,
                       u32 regnum, u16 mask, u16 set);
int mdiobus_c45_modify_changed(struct mii_bus *bus, int addr, int devad,
                               u32 regnum, u16 mask, u16 set);
```

الـ C45 equivalents لـ `mdiobus_modify()`. بيعملوا read-modify-write atomic operation على C45 register مع lock.

---

### Group 8: Per-Device Wrappers (`mdiodev_*`)

دي كلها `static inline` wrappers بتعمل نفس الـ `mdiobus_*` functions بس بتاخد `mdio_device *` بدل ما تاخد `mii_bus *` و`addr` منفصلين. مفيدة جدًا في driver code لأنها بتوفر boilerplate.

---

#### `__mdiodev_read()` / `__mdiodev_write()` / `__mdiodev_modify()` / `__mdiodev_modify_changed()`

```c
static inline int __mdiodev_read(struct mdio_device *mdiodev, u32 regnum);
static inline int __mdiodev_write(struct mdio_device *mdiodev, u32 regnum, u16 val);
static inline int __mdiodev_modify(struct mdio_device *mdiodev, u32 regnum, u16 mask, u16 set);
static inline int __mdiodev_modify_changed(struct mdio_device *mdiodev, u32 regnum, u16 mask, u16 set);
```

كلهم بيستدعوا `__mdiobus_*` المقابلة بـ `mdiodev->bus` و`mdiodev->addr`. **بدون lock** — caller لازم يمسك `mdiodev->bus->mdio_lock`.

---

#### `mdiodev_read()` / `mdiodev_write()` / `mdiodev_modify()` / `mdiodev_modify_changed()`

```c
static inline int mdiodev_read(struct mdio_device *mdiodev, u32 regnum);
static inline int mdiodev_write(struct mdio_device *mdiodev, u32 regnum, u16 val);
static inline int mdiodev_modify(struct mdio_device *mdiodev, u32 regnum, u16 mask, u16 set);
static inline int mdiodev_modify_changed(struct mdio_device *mdiodev, u32 regnum, u16 mask, u16 set);
```

نفسهم بس مع lock. الـ standard API في driver `probe`/`config` code.

**مثال من driver probe:**
```c
static int my_probe(struct mdio_device *mdiodev)
{
    /* قراءة PHY ID */
    int id = mdiodev_read(mdiodev, MII_PHYSID1);
    if (id < 0)
        return id;

    /* تعديل control register — clear low power bit */
    return mdiodev_modify(mdiodev, MDIO_CTRL1,
                          MDIO_CTRL1_LPOWER, 0);
}
```

---

#### C45 Per-Device Wrappers

```c
static inline int __mdiodev_c45_read(struct mdio_device *mdiodev, int devad, u16 regnum);
static inline int __mdiodev_c45_write(struct mdio_device *mdiodev, u32 devad, u16 regnum, u16 val);
static inline int mdiodev_c45_read(struct mdio_device *mdiodev, int devad, u16 regnum);
static inline int mdiodev_c45_write(struct mdio_device *mdiodev, u32 devad, u16 regnum, u16 val);
static inline int mdiodev_c45_modify(struct mdio_device *mdiodev, int devad, u32 regnum, u16 mask, u16 set);
static inline int mdiodev_c45_modify_changed(struct mdio_device *mdiodev, int devad, u32 regnum, u16 mask, u16 set);
```

نفس المنطق بس للـ Clause 45. بيستخدموا `mdiodev->bus` و`mdiodev->addr` مع الـ `devad` الإضافي.

---

### Group 9: Bus Registry Helpers

دي بتتعامل مع الـ address map الداخلي للـ `mii_bus` اللي بيتتبع أي device مسجل على أي address.

---

#### `mdiobus_register_device()`

```c
int mdiobus_register_device(struct mdio_device *mdiodev);
```

بيضيف الـ `mdiodev` لـ `bus->mdio_map[addr]`. الـ map ده بيستخدمه الـ bus عشان يعرف إيه الـ devices الموجودة. بيتستدعى تلقائيًا من `mdio_device_register()`.

- **Return**: `0` أو `-EBUSY` لو في device تاني على نفس الـ address.
- **Locking**: بيستخدم الـ bus spinlock داخليًا.

---

#### `mdiobus_unregister_device()`

```c
int mdiobus_unregister_device(struct mdio_device *mdiodev);
```

بيزيل الـ device من `bus->mdio_map`. بيتستدعى من `mdio_device_remove()`.

---

#### `mdiobus_is_registered_device()`

```c
bool mdiobus_is_registered_device(struct mii_bus *bus, int addr);
```

بيتحقق إن في `mdio_device` مسجل على الـ `addr` المحدد على الـ bus. مفيد لـ bus scanning code.

- **Return**: `true` لو مسجل، `false` لو لأ.

---

#### `mdiobus_get_phy()`

```c
struct phy_device *mdiobus_get_phy(struct mii_bus *bus, int addr);
```

بيجيب الـ `phy_device` المسجل على الـ `addr` على الـ bus. لو الـ device الموجود مش PHY (مش عنده `MDIO_DEVICE_FLAG_PHY`)، بيرجع `NULL`.

- **Return**: pointer لـ `phy_device` أو `NULL`.
- **Key details**: بيستخدم `mdio_map` لإيجاد الـ device، ثم `to_phy_device()` للـ cast.

---

### Group 10: Clause 45 `mdio_if_info` Interface

الـ `struct mdio_if_info` هو abstraction layer بيسمح للـ Ethernet controller drivers باستخدام MDIO Clause 45 devices بشكل موحد. الـ Ethernet controller بيوفر الـ `mdio_read`/`mdio_write` callbacks، وهذه الـ functions بتستخدمهم.

---

#### `mdio45_probe()`

```c
int mdio45_probe(struct mdio_if_info *mdio, int prtad);
```

بيكتشف الـ PHY على الـ `prtad` المحدد ويملأ `mdio->mmds` بالـ MMDs الموجودة. بيعمل ده عن طريق قراءة `MDIO_DEVS1` و`MDIO_DEVS2` registers من الـ PMA/PMD MMD.

```c
/* pseudocode */
devices1 = mdio_read(dev, prtad, MDIO_MMD_PMAPMD, MDIO_DEVS1);
devices2 = mdio_read(dev, prtad, MDIO_MMD_PMAPMD, MDIO_DEVS2);
mdio->mmds = devices1 | (devices2 << 16);
```

- **`prtad`**: Port address للـ PHY (0-31).
- **Return**: `0` لو الـ PHY استجاب، أو error.
- **Caller context**: بيتستدعى من NIC driver initialization عشان يفحص الـ PHY.

---

#### `mdio_set_flag()`

```c
int mdio_set_flag(const struct mdio_if_info *mdio,
                  int prtad, int devad, u16 addr, int mask, bool sense);
```

بيضبط أو يمسح bit معين في MDIO register. لو `sense = true` بيضبط الـ bits في `mask`، لو `false` بيمسحهم.

- **`mask`**: الـ bits المراد التحكم فيها.
- **`sense`**: `true` = set، `false` = clear.
- **Return**: `0` أو error.
- **Key details**: يستخدم الـ `mdio->mdio_read`/`mdio->mdio_write` callbacks.

---

#### `mdio45_links_ok()`

```c
int mdio45_links_ok(const struct mdio_if_info *mdio, u32 mmds);
```

بيتحقق إن الـ link up في الـ MMDs المطلوبة. بيقرأ `MDIO_STAT1` من كل MMD في الـ `mmds` bitmask ويتحقق من `MDIO_STAT1_LSTATUS` bit.

- **`mmds`**: bitmask للـ MMDs المراد التحقق منها.
- **Return**: `1` لو كل الـ MMDs لديها link، `0` لو لأ.
- **Caller context**: بيتستدعى من NIC `link_status` polling functions.

---

#### `mdio45_nway_restart()`

```c
int mdio45_nway_restart(const struct mdio_if_info *mdio);
```

بيعيد تشغيل auto-negotiation بضبط `MDIO_AN_CTRL1_RESTART` bit في الـ AN MMD. لو الـ PHY مش بيدعم AN، بيرجع `-EOPNOTSUPP`.

- **Return**: `0` success، `-EOPNOTSUPP` لو AN غير متاح.

---

#### `mdio45_ethtool_ksettings_get_npage()`

```c
void mdio45_ethtool_ksettings_get_npage(const struct mdio_if_info *mdio,
                                        struct ethtool_link_ksettings *cmd,
                                        u32 npage_adv, u32 npage_lpa);
```

بيملأ `ethtool_link_ksettings` struct من بيانات الـ PHY الحقيقية عن طريق قراءة MDIO registers. الـ `npage_adv` و`npage_lpa` بيسمحوا بإضافة next-page advertisement bits يدويًا.

- **Key details**: بيقرأ speed, duplex, pause capabilities من PMA/PMD, PCS, AN MMDs. بيترجم الـ MDIO register values لـ ethtool linkmode bits.

---

#### `mdio45_ethtool_ksettings_get()`

```c
static inline void mdio45_ethtool_ksettings_get(const struct mdio_if_info *mdio,
                                                 struct ethtool_link_ksettings *cmd);
```

بيستدعي `mdio45_ethtool_ksettings_get_npage()` بـ `npage_adv = 0` و`npage_lpa = 0`. مناسب للـ PHYs اللي مش بتستخدم next pages.

---

#### `mdio_mii_ioctl()`

```c
int mdio_mii_ioctl(const struct mdio_if_info *mdio,
                   struct mii_ioctl_data *mii_data, int cmd);
```

بيعالج الـ `SIOCGMIIPHY` و`SIOCSMIIREG` و`SIOCGMIIIREG` ioctl commands اللي بتيجي من userspace tools زي `mii-tool` و`ethtool`. بيترجم Clause 22 register accesses لـ C45 لو الـ PHY بيدعم ده.

```c
/* pseudocode */
switch (cmd) {
case SIOCGMIIPHY:
    mii_data->phy_id = mdio->prtad;
    return 0;

case SIOCGMIIIREG:
    devad = mdio_phy_id_devad(mii_data->phy_id);
    return mdio->mdio_read(dev, prtad, devad, mii_data->reg_num);

case SIOCSMIIREG:
    devad = mdio_phy_id_devad(mii_data->phy_id);
    return mdio->mdio_write(dev, prtad, devad, mii_data->reg_num, val);
}
```

- **`cmd`**: SIOCGMIIPHY / SIOCGMIIIREG / SIOCSMIIREG.
- **Return**: `0` أو error، `-EOPNOTSUPP` لو الـ command مش معروف.
- **Key details**: لو `mode_support` بيحتوي على `MDIO_EMULATE_C22`، بيحوّل الـ C22 register accesses الشائعة لـ C45 equivalents أوتوماتيك.
- **Caller context**: بيتستدعى من NIC driver `ioctl()` handler.

---

### Group 11: EEE Translators (Legacy ethtool u32 API)

دي group من الـ helpers بتترجم بين الـ MMD EEE registers وبين الـ ethtool legacy u32 advertisement format. موجودة للـ backward compatibility مع قديم ethtool API.

---

#### `mmd_eee_cap_to_ethtool_sup_t()`

```c
static inline u32 mmd_eee_cap_to_ethtool_sup_t(u16 eee_cap);
```

بيترجم قيمة الـ EEE Capability register (3.20) لـ ethtool `SUPPORTED_*` bitmask. كل bit في الـ register بيتترجم لـ `SUPPORTED_*` constant مقابل.

| MMD bit | ethtool constant |
|---|---|
| `MDIO_EEE_100TX` | `SUPPORTED_100baseT_Full` |
| `MDIO_EEE_1000T` | `SUPPORTED_1000baseT_Full` |
| `MDIO_EEE_10GT` | `SUPPORTED_10000baseT_Full` |
| `MDIO_EEE_1000KX` | `SUPPORTED_1000baseKX_Full` |
| `MDIO_EEE_10GKX4` | `SUPPORTED_10000baseKX4_Full` |
| `MDIO_EEE_10GKR` | `SUPPORTED_10000baseKR_Full` |

---

#### `mmd_eee_adv_to_ethtool_adv_t()`

```c
static inline u32 mmd_eee_adv_to_ethtool_adv_t(u16 eee_adv);
```

زي السابق بس للـ EEE Advertisement (7.60) و Link Partner Ability (7.61) registers ويرجع `ADVERTISED_*` bits.

---

#### `ethtool_adv_to_mmd_eee_adv_t()`

```c
static inline u16 ethtool_adv_to_mmd_eee_adv_t(u32 adv);
```

العكس — بيترجم ethtool `ADVERTISED_*` bits لـ MMD EEE register value مناسب للكتابة في register 7.60.

---

### Group 12: EEE Linkmode Translators (Modern API)

الـ modern ethtool API بيستخدم `linkmode` bitmaps بدل الـ legacy u32. هذه الـ functions بتحول بين الـ MMD EEE registers وبين الـ linkmode bitmap.

---

#### `mii_eee_cap1_mod_linkmode_t()`

```c
static inline void mii_eee_cap1_mod_linkmode_t(unsigned long *adv, u32 val);
```

بيترجم قيمة registers EEE cap1/adv1/lpa1 (3.20، 7.60، 7.61 — IEEE 802.3-2018) لـ linkmode bits. بيستخدم `linkmode_mod_bit()` عشان يضبط أو يمسح كل bit على حسب الـ register value.

- **`adv`**: الـ linkmode bitmap المراد تعديله (بيتغير in-place).
- **`val`**: قيمة الـ register (16 bits مُوسَّعة لـ u32).

---

#### `mii_eee_cap2_mod_linkmode_sup_t()` / `mii_eee_cap2_mod_linkmode_adv_t()`

```c
static inline void mii_eee_cap2_mod_linkmode_sup_t(unsigned long *adv, u32 val);
static inline void mii_eee_cap2_mod_linkmode_adv_t(unsigned long *adv, u32 val);
```

بيترجموا EEE cap2 (3.21) وEEE adv2/lpa2 (7.62/7.63 — IEEE 802.3-2022) لـ linkmode bits لـ 2.5G و5G. حاليًا الاثنين لهم نفس الـ implementation لأن الـ bits متطابقة، لكن انفصلوا anticipation للفروق المستقبلية في modes غير مدعومة حاليًا.

---

#### `linkmode_to_mii_eee_cap1_t()` / `linkmode_to_mii_eee_cap2_t()`

```c
static inline u32 linkmode_to_mii_eee_cap1_t(unsigned long *adv);
static inline u32 linkmode_to_mii_eee_cap2_t(unsigned long *adv);
```

العكس — بيحولوا linkmode bitmap لـ register value مناسب للكتابة في EEE advertisement registers (7.60 و7.62 على الترتيب).

---

### Group 13: 10GBT / BASE-T1 / Clause 73 Translators

---

#### `linkmode_adv_to_mii_10gbt_adv_t()`

```c
static inline u32 linkmode_adv_to_mii_10gbt_adv_t(unsigned long *advertising);
```

بيحول الـ linkmode advertisement لـ value مناسب للـ 10GBASE-T AN CONTROL register (7.32). بيدعم 2.5G، 5G، و10G speeds.

| Linkmode bit | Register bit |
|---|---|
| `2500baseT_Full` | `MDIO_AN_10GBT_CTRL_ADV2_5G` |
| `5000baseT_Full` | `MDIO_AN_10GBT_CTRL_ADV5G` |
| `10000baseT_Full` | `MDIO_AN_10GBT_CTRL_ADV10G` |

---

#### `mii_10gbt_stat_mod_linkmode_lpa_t()`

```c
static inline void mii_10gbt_stat_mod_linkmode_lpa_t(unsigned long *advertising, u32 lpa);
```

بيحول قيمة الـ 10GBASE-T AN STATUS register (7.33) لـ linkmode LPA bits. بيعدّل الـ linkmode bitmap in-place.

---

#### `mii_t1_adv_l_mod_linkmode_t()` / `mii_t1_adv_m_mod_linkmode_t()`

```c
static inline void mii_t1_adv_l_mod_linkmode_t(unsigned long *advertising, u32 lpa);
static inline void mii_t1_adv_m_mod_linkmode_t(unsigned long *advertising, u32 lpa);
```

بيترجموا BASE-T1 AN Advertisement registers المنخفضة (ADV_L = reg 514) والعالية (ADV_M = reg 515) لـ linkmode bits. الـ `_l` بتتعامل مع pause bits، الـ `_m` بتتعامل مع speed bits (10BaseT1L، 100BaseT1، 1000BaseT1).

---

#### `linkmode_adv_to_mii_t1_adv_l_t()` / `linkmode_adv_to_mii_t1_adv_m_t()`

```c
static inline u32 linkmode_adv_to_mii_t1_adv_l_t(unsigned long *advertising);
static inline u32 linkmode_adv_to_mii_t1_adv_m_t(unsigned long *advertising);
```

العكس — بيولّدوا register values من linkmode للكتابة في BASE-T1 AN advertisement registers.

---

#### `mii_10base_t1_adv_mod_linkmode_t()` / `linkmode_adv_to_mii_10base_t1_t()`

```c
static inline void mii_10base_t1_adv_mod_linkmode_t(unsigned long *adv, u16 val);
static inline u32 linkmode_adv_to_mii_10base_t1_t(unsigned long *adv);
```

خاصة بالـ 10BASE-T1 (IEEE 802.3cg-2019). الأولى بتحول AN STATUS register (7.527) → linkmode، والثانية بتحول linkmode → AN CONTROL register (7.526) value. بس بيدعموا `EEE_T1L` bit للـ 10BASE-T1L mode.

---

#### `mii_c73_mod_linkmode()`

```c
static inline void mii_c73_mod_linkmode(unsigned long *adv, u16 *lpa);
```

بيحول IEEE 802.3 Clause 73 (backplane autonegotiation) advertisement array لـ ethtool linkmode bits. الـ `lpa` هو array من 3 u16 values يمثلوا الـ AN base page words.

**التفاصيل:**
- `lpa[0]`: pause bits (Clause 73 word 0).
- `lpa[1]`: technology bits — 1GKX، 10GKX4، 40GKR4، 40GCR4، 100GKR4، 100GCR4، 25GR (KR+CR).
- `lpa[2]`: extended tech bits — 2500KX.

**ملاحظة مهمة:** الـ 25GBASE-R bit مشترك بين KR وCR، فالـ function بتضبط الاثنين (`25000baseKR_Full` و`25000baseCR_Full`) في آن واحد عشان تعكس الغموض ده في المعيار.

```c
/* مثال: decode C73 adv من register reads */
u16 lpa[3];
lpa[0] = mdiobus_c45_read(bus, addr, MDIO_MMD_AN, MDIO_AN_C73_0);
lpa[1] = mdiobus_c45_read(bus, addr, MDIO_MMD_AN, MDIO_AN_C73_1);
lpa[2] = mdiobus_c45_read(bus, addr, MDIO_MMD_AN, MDIO_AN_C73_2);
mii_c73_mod_linkmode(lp->advertising, lpa);
```

- **Caller context**: بيتستدعى من backplane PHY drivers أثناء AN result processing.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. مدخلات الـ debugfs

**الـ** MDIO subsystem بيكتب معلوماته تحت `/sys/kernel/debug/` عبر الـ PHY framework، مش مباشرةً من `mdio.h`، بس الـ mdio bus بيظهر تحت:

```bash
# اعرض كل الـ MDIO buses المسجّلة
ls /sys/kernel/debug/mdio-bus/

# اقرأ حالة PHY معيّن على bus معيّن (مثلاً bus اسمه "fixed-0")
cat /sys/kernel/debug/mdio-bus/fixed-0/fixed-0:00

# اقرأ الـ registers من phy device مباشرةً
cat /sys/kernel/debug/mdio-bus/<bus-name>/<phy-addr>
```

**الـ output بيبان كده:**
```
PHY ID: 0x001cc916
state: RUNNING
link: 1
speed: 1000
duplex: 1
```

---

#### 2. مدخلات الـ sysfs

```bash
# اعرض كل الـ MDIO devices المسجّلة
ls /sys/bus/mdio_bus/devices/

# اقرأ الـ modalias الخاص بـ device (مفيد للـ driver matching)
cat /sys/bus/mdio_bus/devices/<id>/modalias
# مثال output: mdio:0x001cc916

# driver المرتبط بالـ device
ls -la /sys/bus/mdio_bus/devices/<id>/driver

# phy_id للـ device
cat /sys/bus/mdio_bus/devices/<id>/phy_id

# اقرأ الـ addr (PRTAD)
cat /sys/bus/mdio_bus/devices/<id>/addr

# الـ net device المرتبط
ls /sys/bus/mdio_bus/devices/<id>/net/
```

جدول الـ sysfs attributes:

| Attribute | المسار | المعنى |
|-----------|--------|---------|
| `phy_id` | `/sys/bus/mdio_bus/devices/<id>/phy_id` | PHY ID register 1+2 combined |
| `addr` | `/sys/bus/mdio_bus/devices/<id>/addr` | MDIO bus address (0-31) |
| `modalias` | `/sys/bus/mdio_bus/devices/<id>/modalias` | driver matching string |
| `attached_dev` | `/sys/bus/mdio_bus/devices/<id>/attached_dev` | net interface name |

---

#### 3. الـ ftrace — tracepoints وأحداث مفيدة

```bash
# فعّل الـ tracepoints الخاصة بالـ MDIO/PHY
cd /sys/kernel/debug/tracing

# اعرض الـ events المتاحة
grep -r "mdio\|phy\|mii" available_events

# فعّل events الـ MDIO bus read/write
echo 1 > events/mdio/enable

# فعّل events الـ PHY state machine
echo 1 > events/net/phy_trigger_machine/enable 2>/dev/null || true

# ابدأ الـ tracing
echo 1 > tracing_on
cat trace_pipe &

# لو بتحتاج تتبّع mdiobus_read و mdiobus_write بالـ function tracer
echo function > current_tracer
echo mdiobus_read mdiobus_write mdiobus_c45_read mdiobus_c45_write > set_ftrace_filter
echo 1 > tracing_on
cat trace_pipe
```

**مثال output من trace_pipe:**
```
eth0-rx/0  [001] ....  123.456: mdiobus_read <-phy_read
eth0-rx/0  [001] ....  123.457: mdiobus_write <-phy_write_mmd
```

لتتبع الـ `mdio_device_reset` و state transitions:

```bash
echo mdio_device_reset mdiobus_register > set_ftrace_filter
echo 1 > tracing_on
```

---

#### 4. الـ printk والـ dynamic debug

```bash
# فعّل dynamic debug لكل الـ MDIO subsystem
echo "module libphy +p" > /sys/kernel/debug/dynamic_debug/control
echo "module mdio +p"   > /sys/kernel/debug/dynamic_debug/control

# فعّل لملف معيّن فقط
echo "file drivers/net/phy/mdio_bus.c +pflmt" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/net/phy/phy.c +pflmt"      > /sys/kernel/debug/dynamic_debug/control

# فعّل لـ function معيّنة
echo "func mdiobus_read +p" > /sys/kernel/debug/dynamic_debug/control

# تابع الـ kernel log
dmesg -w | grep -E "mdio|phy|mii"

# أو عبر journald
journalctl -k -f | grep -iE "mdio|phy"
```

مثال تفعيل verbose لـ PHY state machine:

```bash
echo "file drivers/net/phy/phy.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/net/phy/phy_device.c +p" > /sys/kernel/debug/dynamic_debug/control
```

---

#### 5. خيارات الـ kernel config للـ debugging

| Config | الوظيفة |
|--------|---------|
| `CONFIG_PHYLIB` | لازم يكون مفعّل — هو اللي بيسجّل الـ MDIO bus |
| `CONFIG_MDIO_DEVICE` | دعم الـ `struct mdio_device` |
| `CONFIG_MDIO_BUS` | الـ bus core |
| `CONFIG_NETDEV_DEBUG` | debugging عام للـ network devices |
| `CONFIG_DYNAMIC_DEBUG` | مطلوب للـ dynamic debug |
| `CONFIG_DEBUG_DRIVER` | verbose messages من الـ driver model |
| `CONFIG_PHYLIB_DISABLE_DELAY` | يلغي الـ delays في الـ AN — مفيد في الـ testing |
| `CONFIG_MDIO_I2C` | لو الـ MDIO over I2C |
| `CONFIG_MDIO_BCM_IPROC` | مثال driver — فعّل CONFIG_NET_VENDOR_BROADCOM |
| `CONFIG_NETCONSOLE` | ابعت الـ logs عبر network لو الـ serial مش متاح |
| `CONFIG_KASAN` | اكتشاف memory corruption في الـ MDIO drivers |
| `CONFIG_LOCKDEP` | يكتشف deadlocks في الـ `mdio_bus->mdio_lock` |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "CONFIG_(PHYLIB|MDIO|PHY_DEBUG)"
```

---

#### 6. أدوات خاصة بالـ subsystem

**ethtool** — الأداة الأساسية:

```bash
# اقرأ link settings (يستخدم mdio45_ethtool_ksettings_get داخليًا)
ethtool eth0

# اقرأ register dump (يستخدم mdiobus_read)
ethtool -d eth0

# اقرأ EEE capabilities (MDIO_AN_EEE_ADV register 7.60)
ethtool --show-eee eth0

# اقرأ الـ PHY statistics
ethtool -S eth0

# اعمل AN restart (يستخدم mdio45_nway_restart)
ethtool -r eth0

# اقرأ الـ phy_id
ethtool -i eth0
```

**phytool** — قراءة/كتابة MDIO registers مباشرةً:

```bash
# تثبيت
apt install phytool   # أو ابني من المصدر

# اقرأ register (Clause 22) — مثلاً BMCR (reg 0) على PHY addr 1
phytool read eth0/1/0

# اكتب register
phytool write eth0/1/0 0x3100

# اقرأ Clause 45 — MMD=3 (PCS), reg=20 (EEE Capability)
phytool read eth0/1/3:20
```

**mii-tool** (قديم، لكن مفيد):

```bash
mii-tool -v eth0
mii-tool --reset eth0
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ kernel log | المعنى | الحل |
|----------------------|--------|-------|
| `MDIO bus error; reconfigure` | فشل إرسال frame على الـ MDIO bus | افحص الـ clock، الـ MDC/MDIO lines، الـ pull-up resistors |
| `PHY xx not found` | الـ kernel ما لقاش PHY على الـ address المحدد | تحقق `PRTAD` في الـ DT أو الـ platform data، وافحص الـ reset GPIO |
| `mdiobus already has device at addr N` | جهازان على نفس الـ MDIO address | راجع الـ DT، وكل PHY له address خاص (0-31) |
| `Unable to find bus` | الـ mdio_device ما عندوش مؤشر لـ mii_bus | الـ probe ترتيب غلط، أو الـ DT phandle مش صح |
| `mii_bus: Could not get IRQ` | فشل تسجيل IRQ للـ bus | استخدم `PHY_POLL` كـ fallback |
| `Could not attach to PHY` | الـ driver ما يقدرش يعمل probe للـ PHY | افحص الـ phy_id والـ driver registration |
| `reset_control_deassert failed` | مشكلة في الـ reset controller | افحص الـ DT `resets` property والـ reset driver |
| `No compatible PHY driver found` | ما فيش driver يعمل match للـ phy_id | فعّل الـ driver الصح في الـ kernel config أو جهّز module |
| `MDIO read timeout` | الـ PHY ما رد في الوقت المحدد | افحص power، clock، الـ MDC frequency |
| `phylink: error -EIO` | فشل قراءة PCS/PHY registers | راجع الـ interface mode في الـ DT |

---

#### 8. أماكن استراتيجية لـ dump_stack() و WARN_ON()

الأماكن دي في الـ drivers اللي بتستخدم `mdio.h`:

```c
/* في mdio_device_reset() — تحقق من صحة الـ state */
void my_mdio_reset(struct mdio_device *mdiodev, int value)
{
    /* تحقق إن الـ bus محمول صح */
    WARN_ON(!mdiodev->bus);
    WARN_ON(mdiodev->addr < 0 || mdiodev->addr > 31);
    mdio_device_reset(mdiodev, value);
}

/* في الـ probe — تحقق من الـ phy_id */
int my_driver_probe(struct mdio_device *mdiodev)
{
    int id;
    id = mdiodev_read(mdiodev, MDIO_DEVID1);
    if (id < 0) {
        /* PHY ما ردش — الـ bus أو الـ reset غلط */
        WARN_ON_ONCE(1);
        dump_stack();
        return id;
    }
    /* تحقق إن الـ MMDs الأساسية موجودة */
    WARN_ON(!(mdiodev_read(mdiodev, MDIO_DEVS1) & MDIO_DEVS_PCS));
    ...
}

/* عند read/write فشل */
static int safe_c45_read(struct mdio_device *mdiodev, int devad, u16 reg)
{
    int val = mdiodev_c45_read(mdiodev, devad, reg);
    /* قيمة 0xFFFF تعني PHY مش موجود أو error */
    WARN_ON(val == 0xFFFF);
    return val;
}
```

---

### Hardware Level

#### 1. التحقق إن حالة الـ hardware تطابق حالة الـ kernel

```bash
# قارن ما يقرأه الـ kernel مع ما يُتوقع من الـ hardware datasheet

# 1. اقرأ الـ PHY ID من الـ kernel
ethtool -i eth0
# driver: ...
# version: ...

# 2. احسب الـ OUI من الـ PHY ID
# MDIO_DEVID1 (reg 2) و MDIO_DEVID2 (reg 3)
# OUI = bits[3:0] من reg2 shifted left 6 bits + bits[15:10] من reg3
# مثال: ID1=0x0141, ID2=0x0DD1 → OUI=0x005043 (Marvell)

# 3. تحقق من link status مباشرةً
phytool read eth0/1/1   # BMSR (Basic Mode Status Register)
# Bit 2 = link status

# 4. تحقق من الـ MDIO_STAT1 (= BMSR) للـ Clause 45
# MDIO_STAT1_LSTATUS = bit 2 = link up
phytool read eth0/1/3:1   # PCS Status 1 (Clause 45)
```

لمقارنة الـ speed المُعلنة مع الـ hardware:

```bash
# Clause 22: اقرأ BMCR
phytool read eth0/1/0
# Bit 6: Speed LSB, Bit 13: Speed MSB, Bit 8: Duplex

# Clause 45: اقرأ MMD=1 reg=0 (PMA/PMD Control 1)
phytool read eth0/1/1:0
```

---

#### 2. تقنيات الـ Register Dump

**باستخدام devmem2:**

```bash
# افترض إن الـ MDIO controller mapped على 0xFE300000
# (يختلف حسب الـ SoC — ارجع للـ datasheet)
apt install devmem2

# اقرأ MDIO control register
devmem2 0xFE300000 w

# اقرأ MDIO data register
devmem2 0xFE300004 w
```

**باستخدام /dev/mem (لو مفعّل في الـ kernel):**

```bash
# تحقق إن /dev/mem متاح
ls -la /dev/mem

# استخدم dd لقراءة block من الـ registers
dd if=/dev/mem bs=4 count=16 skip=$((0xFE300000/4)) 2>/dev/null | xxd
```

**باستخدام io tool من الـ busybox:**

```bash
# قراءة 32-bit register
io -4 -r 0xFE300000

# كتابة قيمة
io -4 -w 0xFE300000 0x00000001
```

**من داخل الـ kernel driver (للـ debugging المؤقت):**

```c
/* اطبع كل الـ MMD registers المهمة */
static void dump_mdio_regs(struct mdio_device *mdiodev)
{
    int i, val;
    /* PMA/PMD registers */
    for (i = 0; i <= 11; i++) {
        val = mdiodev_c45_read(mdiodev, MDIO_MMD_PMAPMD, i);
        pr_info("PMA reg[%d] = 0x%04x\n", i, val);
    }
    /* AN registers */
    val = mdiodev_c45_read(mdiodev, MDIO_MMD_AN, MDIO_AN_EEE_ADV);
    pr_info("EEE Adv (7.60) = 0x%04x\n", val);
    val = mdiodev_c45_read(mdiodev, MDIO_MMD_AN, MDIO_AN_EEE_LPABLE);
    pr_info("EEE LP  (7.61) = 0x%04x\n", val);
}
```

---

#### 3. نصائح الـ Logic Analyzer والـ Oscilloscope

الـ MDIO bus (Clause 22 و 45) بيشتغل على الـ MDC clock + MDIO data line:

```
MDC  _____|‾|_|‾|_|‾|_|‾|_|‾|_|‾|_|‾|_|‾|_|‾|_|‾|_|‾|_|‾|_|‾|_|‾|___
MDIO _____|  ST |OP |PHYA|REGA|TA|          DATA          |___________
          01/00  10   prtad devad  Z0   16-bit data
```

**نقاط قياس مهمة:**

| نقطة القياس | القيمة المتوقعة | لو غلط |
|-------------|----------------|---------|
| MDC frequency | عادةً 2.5 MHz (max 8 MHz per IEEE 802.3) | الـ MDIO reads بترجع 0xFFFF |
| MDC duty cycle | 50% ± 10% | بيانات غير موثوقة |
| MDIO high voltage | > 2.0V (3.3V system) | حروف مقلوبة |
| MDIO low voltage | < 0.8V | |
| Turnaround (TA) | 2 cycles: Z then 0 (read) | bus conflict |
| Preamble | 32 أو أكثر من 1's | بعض الـ PHYs بتحتاجه |

**إعداد الـ Logic Analyzer:**

```
Channel 0: MDC
Channel 1: MDIO
Sample rate: ≥ 25 MHz (10x oversampling)
Trigger: Rising edge على MDC
Decoder: MDIO (Clause 22 / Clause 45)
```

**اكتشاف مشكلة الـ bus contention:**
- لو MDIO بيبقى stuck على قيمة بعد read → أكثر من master على نفس الـ bus
- لو البيانات always 0xFFFF → PHY مش موجود أو الـ reset مش اتعمل

---

#### 4. المشاكل الشائعة في الـ Hardware وأنماط الـ kernel log

| المشكلة | النمط في الـ dmesg | السبب المحتمل |
|---------|-------------------|---------------|
| PHY ما بيردش | `mdio_bus mdio_bus: MDIO read error` | الـ PHY مش powered، أو الـ reset GPIO غلط |
| قراءات 0xFFFF دائمًا | `PHY ID: 0xffffffff` | الـ MDC/MDIO lines مش موصّلة، أو الـ pull-up مش موجود |
| AN ما بتكتملش | `Link is Down` بعد 30 ثانية | mismatch في الـ duplex/speed، أو الـ cable غلط |
| EEE مش شغّال | `eee local tx lpi disabled` | الـ register 7.60 بيرجع 0، الـ PHY مش داعم EEE |
| reset ما بيشتغلش | `deassert failed: -ENODEV` | الـ GPIO descriptor مش صح في الـ DT |
| Clause 45 timeouts | `c45_read timeout` | بعض الـ PHYs بتحتاج تأخير بعد MMD address write |

---

#### 5. الـ Device Tree Debugging

```bash
# اعرض كل الـ DT nodes الخاصة بالـ MDIO
find /proc/device-tree/ -name "mdio*" -o -name "*phy*" 2>/dev/null | head -20

# اقرأ الـ compatible string
cat /proc/device-tree/soc/mdio@*/compatible 2>/dev/null | strings

# اقرأ الـ reg (PRTAD)
cat /proc/device-tree/soc/mdio@*/phy@*/reg 2>/dev/null | xxd

# تحقق من الـ reset-gpios
cat /proc/device-tree/soc/mdio@*/phy@*/reset-gpios 2>/dev/null | xxd
```

**مثال DT صحيح لـ Clause 45 PHY:**

```dts
mdio0: mdio@fe300000 {
    compatible = "vendor,mdio-controller";
    reg = <0xfe300000 0x100>;
    #address-cells = <1>;
    #size-cells = <0>;
    clock-frequency = <2500000>;  /* MDC = 2.5 MHz */

    phy0: ethernet-phy@1 {
        compatible = "ethernet-phy-ieee802.3-c45";
        reg = <1>;                /* PRTAD = 1 */
        reset-gpios = <&gpio 10 GPIO_ACTIVE_LOW>;
        reset-assert-us = <10000>;    /* 10ms */
        reset-deassert-us = <50000>;  /* 50ms */
    };
};
```

**تحقق من توافق الـ DT مع الـ kernel:**

```bash
# هل الـ driver اكتشف الـ MDIO bus؟
dmesg | grep -i "mdio\|mii_bus"

# هل الـ PHY اتعرف؟
dmesg | grep -i "phy\|PHY ID"

# مثال output ناجح
# mdio_bus stmmac-0: MDIO bus registered at stmmac-0
# libphy: stmmac-0: probed
# stmmaceth fe300000.ethernet eth0: PHY [stmmac-0:00] driver [Marvell 88E1111] (irq=POLL)
```

---

### Practical Commands

#### جاهز للنسخ — كل التقنيات

**1. فحص سريع لحالة الـ MDIO bus:**

```bash
# اعرض كل الـ PHYs على الشبكة
for iface in $(ls /sys/class/net/ | grep -v lo); do
    echo "=== $iface ==="
    ethtool $iface 2>/dev/null | grep -E "Speed|Duplex|Link|Port"
    ethtool -i $iface 2>/dev/null | grep -E "driver|version|bus"
done
```

**2. قراءة كل registers الـ Clause 22 PHY:**

```bash
# يقرأ registers 0-31 من PHY على address 1
for reg in $(seq 0 31); do
    val=$(phytool read eth0/1/$reg 2>/dev/null)
    printf "reg[%2d] = %s\n" $reg "$val"
done
```

**مثال output:**
```
reg[ 0] = 0x1140    # BMCR: Auto-neg enabled, 1000M
reg[ 1] = 0x796d    # BMSR: Link up, AN complete
reg[ 2] = 0x0141    # PHYID1: OUI[23:8]
reg[ 3] = 0x0dd1    # PHYID2: OUI[7:2] + model + rev
reg[ 4] = 0x01e1    # ADVERTISE: 10/100/1000 Full
reg[ 5] = 0x45e1    # LPA: Link partner abilities
```

**3. قراءة EEE Clause 45 registers:**

```bash
# MMD 7 (AN) — EEE advertisement و link partner
echo "=== EEE AN registers ==="
phytool read eth0/1/7:60   # MDIO_AN_EEE_ADV   (7.60)
phytool read eth0/1/7:61   # MDIO_AN_EEE_LPABLE (7.61)
phytool read eth0/1/7:62   # MDIO_AN_EEE_ADV2   (7.62)
phytool read eth0/1/7:63   # MDIO_AN_EEE_LPABLE2 (7.63)

# MMD 3 (PCS) — EEE capability
phytool read eth0/1/3:20   # MDIO_PCS_EEE_ABLE  (3.20)
phytool read eth0/1/3:21   # MDIO_PCS_EEE_ABLE2 (3.21)
```

**مثال output وتفسيره:**
```
7:60 = 0x0006   # bits[1:2] = 100TX + 1000T EEE advertised
7:61 = 0x0006   # LP also supports EEE 100TX + 1000T
3:20 = 0x0006   # PHY hardware capable of EEE 100TX + 1000T
```

تفسير bits الـ EEE:

| Bit | Mask | المعنى |
|-----|------|--------|
| 1 | `MDIO_EEE_100TX` (0x0002) | EEE at 100BASE-TX |
| 2 | `MDIO_EEE_1000T` (0x0004) | EEE at 1000BASE-T |
| 3 | `MDIO_EEE_10GT` (0x0008) | EEE at 10GBASE-T |
| 4 | `MDIO_EEE_1000KX` (0x0010) | EEE at 1000BASE-KX |
| 5 | `MDIO_EEE_10GKX4` (0x0020) | EEE at 10GBASE-KX4 |
| 6 | `MDIO_EEE_10GKR` (0x0040) | EEE at 10GBASE-KR |

**4. تفعيل الـ ftrace لتتبع كل MDIO transactions:**

```bash
#!/bin/bash
cd /sys/kernel/debug/tracing
echo nop > current_tracer
echo 0 > tracing_on
echo > trace

# فعّل function tracing للـ mdio functions
echo function > current_tracer
echo "mdiobus_read
mdiobus_write
mdiobus_c45_read
mdiobus_c45_write
mdio_device_reset
__mdiobus_read
__mdiobus_write" > set_ftrace_filter

echo 1 > tracing_on
sleep 5   # خلّي الـ network تعمل transactions
echo 0 > tracing_on

cat trace | head -50
```

**مثال output:**
```
     kworker/0:2-123 [000] ....  456.789: mdiobus_read <-phy_read
     kworker/0:2-123 [000] ....  456.790: __mdiobus_read <-mdiobus_read
     kworker/0:2-123 [000] ....  456.792: mdiobus_write <-phy_write_mmd
```

**5. فحص الـ reset GPIO:**

```bash
# اعرض حالة الـ GPIO المرتبط بالـ PHY reset
# (لازم تعرف رقم الـ GPIO من الـ DT)
cat /sys/kernel/debug/gpio | grep -i "phy\|mdio\|reset"

# مثال output:
#  gpio-10  (phy-reset          ) out hi
```

**6. سكريبت كامل لتشخيص مشاكل MDIO:**

```bash
#!/bin/bash
# mdio_diag.sh - MDIO full diagnostic script

IFACE=${1:-eth0}
echo "=== MDIO Diagnostic for $IFACE ==="

echo "[1] Driver info:"
ethtool -i $IFACE

echo ""
echo "[2] Link status:"
ethtool $IFACE | grep -E "Speed|Duplex|Link detected|Auto"

echo ""
echo "[3] PHY registers (Clause 22):"
phytool print $IFACE

echo ""
echo "[4] EEE status:"
ethtool --show-eee $IFACE 2>/dev/null || echo "EEE not supported or phytool missing"

echo ""
echo "[5] Recent kernel messages:"
dmesg | tail -20 | grep -iE "mdio|phy|eth|link"

echo ""
echo "[6] sysfs MDIO bus:"
ls /sys/bus/mdio_bus/devices/ 2>/dev/null

echo ""
echo "[7] debugfs PHY state:"
cat /sys/kernel/debug/mdio-bus/*/$(ls /sys/kernel/debug/mdio-bus/ 2>/dev/null | head -1)/* 2>/dev/null | head -20
```

**مثال تشغيل واخراج:**

```bash
./mdio_diag.sh eth0

=== MDIO Diagnostic for eth0 ===
[1] Driver info:
driver: stmmac-platform
version: Dec  1 2023
bus-info: fe300000.ethernet

[2] Link status:
Speed: 1000Mb/s
Duplex: Full
Auto-negotiation: on
Link detected: yes

[3] PHY registers (Clause 22):
eth0/0:
    bmcr: 0x1140 (auto-neg enabled, 1G)
    bmsr: 0x796d (link up)
    id: 0x01410dd1

[5] Recent kernel messages:
[12345.678] stmmac-platform fe300000.ethernet eth0: Link is Up - 1Gbps/Full
```

**7. تحقق من الـ Clause 45 device presence:**

```bash
# اقرأ MDIO_DEVS1 و MDIO_DEVS2 (registers 5 و 6 في أي MMD)
# بيبيّن أي MMDs موجودة في الـ PHY
devs1=$(phytool read eth0/1/1:5 2>/dev/null)   # Devices in package 1
devs2=$(phytool read eth0/1/1:6 2>/dev/null)   # Devices in package 2

echo "DEVS1 (MMDs present low):  $devs1"
echo "DEVS2 (MMDs present high): $devs2"

# تفسير النتيجة:
# 0x00BE = bits: 1(PMAPMD) + 2(WIS) + 3(PCS) + 4(PHYXS) + 5(DTEXS) + 7(AN)
# كل bit يقابل MDIO_DEVS_PRESENT(n)
```

**8. إعادة تعيين PHY يدويًا:**

```bash
# أسلوب 1: عبر ethtool
ethtool -r eth0   # AN restart بس

# أسلوب 2: عبر sysfs
echo 0 > /sys/bus/mdio_bus/devices/<id>/reset_gpio   # assert reset
sleep 0.1
echo 1 > /sys/bus/mdio_bus/devices/<id>/reset_gpio   # deassert

# أسلوب 3: كتابة BMCR reset bit مباشرةً (Clause 22)
phytool write eth0/1/0 0x8000   # bit 15 = reset
sleep 0.5
phytool read eth0/1/0            # تحقق إن الـ bit اتصفّر تلقائيًا
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: PHY مش بيتعرف على Industrial Gateway بـ AM62x

#### العنوان
**MDIO bus scan فاشل — Industrial gateway على AM62x مع PHY خارجي**

#### السياق
شركة بتطور industrial gateway بـ Texas Instruments AM62x. البورد فيها Ethernet port عن طريق RGMII متوصل بـ Microchip KSZ9131 PHY. المنتج بيعمل على خط التصنيع ومحتاج يكون up في أقل من 3 ثواني بعد power-on.

#### المشكلة
بعد kernel boot، الـ Ethernet interface مش بتيجي up. الـ `ip link show` بيدي:
```
eth0: <NO-CARRIER,BROADCAST,MULTICAST,UP>
```
والـ dmesg بيقول:
```
mdio_bus stmmac-0: MDIO bus scan: could not find PHY at address 7
```

#### التحليل
المهندس بيبدأ يتبع الكود في `mdio.h`.

**أول خطوة** — الـ scan بيمر على `mdio_device_create()` لكل address من 0 لـ 31:
```c
struct mdio_device *mdio_device_create(struct mii_bus *bus, int addr);
```
الـ `addr` هنا هو physical address على الـ MDIO bus (0-31 زي ما موضح في التعليق في الـ struct):
```c
/* Bus address of the MDIO device (0-31) */
int addr;
```

**تاني خطوة** — بعد الـ create، kernel بيحاول يعمل `mdio_device_reset()`:
```c
void mdio_device_reset(struct mdio_device *mdiodev, int value);
```
الـ reset بيستخدم `reset_gpio` من الـ struct:
```c
struct gpio_desc *reset_gpio;
unsigned int reset_assert_delay;
unsigned int reset_deassert_delay;
```

المهندس راح فحص الـ Device Tree:
```dts
/* WRONG config */
&mdio0 {
    phy0: ethernet-phy@7 {
        reg = <7>;
        reset-gpios = <&gpio0 15 GPIO_ACTIVE_LOW>;
        reset-assert-us = <10>;      /* too short! */
        reset-deassert-us = <150>;   /* too short! */
    };
};
```

الـ KSZ9131 datasheet بيقول محتاج minimum 10ms assert و 100ms deassert. الـ delays في الـ DT قصيرة جداً، فالـ PHY مش بيطلع من الـ reset صح قبل ما الـ MDIO bus يحاول يقرأ الـ ID registers.

#### الحل
```dts
/* CORRECT config */
&mdio0 {
    phy0: ethernet-phy@7 {
        reg = <7>;
        reset-gpios = <&gpio0 15 GPIO_ACTIVE_LOW>;
        reset-assert-us = <10000>;    /* 10ms */
        reset-deassert-us = <150000>; /* 150ms */
    };
};
```

للتحقق بعد الـ fix:
```bash
# قبل boot: تأكد الـ PHY address صح
cat /sys/bus/mdio_bus/devices/stmmac-0\:07/phy_id

# أو باستخدام mdio-tool
mdio-tool stmmac-0 read 7 2   # PHYID1
mdio-tool stmmac-0 read 7 3   # PHYID2
```

#### الدرس المستفاد
الـ `reset_assert_delay` و `reset_deassert_delay` في `struct mdio_device` بيجوا مباشرة من الـ DT properties `reset-assert-us` و `reset-deassert-us`. دايماً ارجع للـ PHY datasheet وماتعتمدش على القيم الافتراضية. الـ industrial environments كمان فيها voltage ramp-up أبطأ وده بيزيد الـ timing requirements.

---

### السيناريو الثاني: EEE بيسبب link drops متكررة في Android TV Box بـ Allwinner H616

#### العنوان
**EEE (Energy-Efficient Ethernet) بيسبب instability — Android TV Box بـ Allwinner H616**

#### السياق
منتج Android TV box بيستخدم Allwinner H616 SoC متوصل بـ Realtek RTL8211F PHY عن طريق RGMII. المستخدمين بيشتكوا من video streaming بيتقطع كل بضع دقايق. الـ link بيوقع ويرجع في خلال ثانية تقريباً.

#### المشكلة
الـ logs بتظهر:
```
[  142.331] eth0: Link is Down
[  143.089] eth0: Link is Up - 1Gbps/Full - flow control off
```
الـ pattern ده بيتكرر بشكل منتظم تقريباً كل 3-7 دقايق أثناء الـ streaming.

#### التحليل
المهندس بيشك في EEE لأنه common مع links تحت low-traffic load متقطع.

**بيفحص الـ EEE capability** عن طريق الـ helpers الموجودين في `mdio.h`:
```c
/* reg 3.20 — EEE capability */
static inline void mii_eee_cap1_mod_linkmode_t(unsigned long *adv, u32 val)
{
    linkmode_mod_bit(ETHTOOL_LINK_MODE_100baseT_Full_BIT,
                     adv, val & MDIO_EEE_100TX);
    linkmode_mod_bit(ETHTOOL_LINK_MODE_1000baseT_Full_BIT,
                     adv, val & MDIO_EEE_1000T);
    ...
}
```

الـ EEE capability register (MMD 3, reg 20) و advertisement register (MMD 7, reg 60) بيتقروا باستخدام:
```c
int mdiobus_c45_read(struct mii_bus *bus, int addr, int devad, u32 regnum);
```

يعني `devad = MDIO_MMD_PCS (3)` و `regnum = MDIO_PCS_EEE_ABLE (20)`.

بيقرأ بـ ethtool:
```bash
ethtool --show-eee eth0
```
النتيجة:
```
EEE Settings for eth0:
    EEE status: enabled - active
    Tx LPI: 280
    EEE enabled: 1000baseT/Full
    EEE advertised: 1000baseT/Full
    LP advertised: 1000baseT/Full
```
الـ EEE شغال على 1GbE. الـ RTL8211F مع بعض الـ H616 GMAC implementations بيكون فيه timing issue في الـ LPI (Low Power Idle) exit sequence.

#### الحل
**الحل السريع** — إيقاف EEE:
```bash
ethtool --set-eee eth0 eee off
```

**الحل الدائم** — في kernel driver أو DT:
```dts
&mdio0 {
    phy0: ethernet-phy@0 {
        reg = <0>;
        /* Disable EEE advertisement */
        eee-broken-1000t;
    };
};
```

أو في الـ PHY driver عن طريق:
```c
/* translate back using mdio.h helper before writing */
u16 eee_adv = ethtool_adv_to_mmd_eee_adv_t(0); /* adv = 0 = disable all */
mdiobus_c45_write(bus, addr, MDIO_MMD_AN,
                  MDIO_AN_EEE_ADV, eee_adv);
```

الـ function `ethtool_adv_to_mmd_eee_adv_t()` من `mdio.h` هي بالظبط المكان الصح لتحويل ethtool flags لـ register value قبل الكتابة.

#### الدرس المستفاد
الـ EEE helpers في `mdio.h` زي `mmd_eee_cap_to_ethtool_sup_t()` و `ethtool_adv_to_mmd_eee_adv_t()` موجودين عشان تعمل الـ translation صح. أي حاجة بتكتبها مباشرة للـ EEE registers من غير ما تستخدم الـ helpers دي ممكن تطلع غلط. كمان EEE في بيئات streaming هو أول حاجة تشيلها لما تشوف link flapping.

---

### السيناريو الثالث: MDIO MUX deadlock في Custom Board بـ i.MX8MP

#### العنوان
**Deadlock في MDIO MUX — Custom board بـ i.MX8MP مع switch و PHY على نفس الـ bus**

#### السياق
custom board بتستخدم NXP i.MX8MP مع network switch (Microchip LAN9370) و standalone PHY (TI DP83867) متوصلين على نفس الـ MDIO bus عن طريق GPIO-controlled MUX. البورد بتستخدم في industrial automation.

#### المشكلة
الـ kernel بيـ hang أثناء boot بعد message:
```
INFO: task kworker/u4:2:57 blocked for more than 120 seconds.
...
schedule+0x58/0xc0
mutex_lock_nested+0x3c/0x50
mdiobus_read+0x24/0x70
```

الـ call stack بيبين deadlock في الـ MDIO bus mutex.

#### التحليل
الـ `mdio.h` بيعرّف الـ mutex lock classes:
```c
enum mdio_mutex_lock_class {
    MDIO_MUTEX_NORMAL,
    MDIO_MUTEX_MUX,
    MDIO_MUTEX_NESTED,
};
```

الـ comment في الكود بيشرح السبب:
```c
/* Multiple levels of nesting are possible. However typically this is
 * limited to nested DSA like layer, a MUX layer, and the normal
 * user. Instead of trying to handle the general case, just define
 * these cases.
 */
```

المشكلة: الـ switch driver (LAN9370 DSA) بيحاول يعمل `mdiobus_read()` من جوا callback بيتعمل call منه وهو شايل الـ bus mutex بـ `MDIO_MUTEX_NORMAL`. الـ MUX driver كمان محتاج ياخد نفس الـ mutex عشان يعمل switch.

الـ sequence:
```
DSA driver → mdiobus_read() → acquires MDIO_MUTEX_NORMAL
                             → MUX callback needed
                             → MUX tries mdiobus_read() → waits for MUTEX_NORMAL
                             → DEADLOCK
```

الحل المطلوب: الـ nested calls لازم تستخدم `mdiobus_read_nested()`:
```c
int mdiobus_read_nested(struct mii_bus *bus, int addr, u32 regnum);
int mdiobus_write_nested(struct mii_bus *bus, int addr, u32 regnum, u16 val);
```

أو بالنسبة للـ c45:
```c
int mdiobus_c45_read_nested(struct mii_bus *bus, int addr, int devad, u32 regnum);
int mdiobus_c45_write_nested(struct mii_bus *bus, int addr, int devad, u32 regnum, u16 val);
```

المهندس بيكتشف إن الـ MUX driver كان بيستخدم `mdiobus_read()` العادية بدل `mdiobus_read_nested()`.

#### الحل
في الـ MDIO MUX driver:
```c
/* WRONG — causes deadlock when called from within bus lock context */
static int mux_select(struct mii_bus *bus, int addr)
{
    return mdiobus_write(mux->parent_bus, mux_addr,
                         MUX_CTRL_REG, addr);  /* deadlock! */
}

/* CORRECT — use nested variant */
static int mux_select(struct mii_bus *bus, int addr)
{
    return mdiobus_write_nested(mux->parent_bus, mux_addr,
                                MUX_CTRL_REG, addr);  /* safe */
}
```

للتحقق إن lockdep كان بيصرخ:
```bash
dmesg | grep -i "possible circular locking"
dmesg | grep -i "MDIO"
```

#### الدرس المستفاد
الـ `MDIO_MUTEX_NESTED` و `MDIO_MUTEX_MUX` في `enum mdio_mutex_lock_class` مش مجرد documentation — هم بيحددوا الـ lock hierarchy. أي driver بيعمل MDIO access من جوا MUX callback أو DSA layer لازم يستخدم الـ `_nested` variants. الـ lockdep بيكشف ده بكير لو كان مفعّل في kernel config.

---

### السيناريو الرابع: Clause 45 vs Clause 22 confusion في Automotive ECU بـ STM32MP1

#### العنوان
**PHY مش بيتعرف على capabilities — Automotive ECU بـ STM32MP1 مع 100BASE-T1 PHY**

#### السياق
automotive ECU بتستخدم STM32MP1 مع Broadcom BCM89811 PHY وهو 100BASE-T1 PHY للـ in-vehicle networking. الـ PHY بيدعم Clause 45 بس. المهندس بيحاول يشغّل الـ link لأول مرة.

#### المشكلة
الـ PHY driver بيجيب:
```
libphy: stm32-dwmac-0:04: probed
stm32-dwmac-0:04: detected feature(s) not advertised
eth0: PHY [stm32-dwmac-0:04] driver [BCM89811] (irq=POLL)
eth0: No PHY found
```

الـ autonegotiation مش بتشتغل وlinkmode بيبقى فاضي.

#### التحليل
المهندس بيفحص إزاي الـ `mdio_if_info` struct بيتعامل مع الـ mode:
```c
struct mdio_if_info {
    int prtad;
    u32 mmds;
    unsigned mode_support;
    ...
};

#define MDIO_SUPPORTS_C22    1
#define MDIO_SUPPORTS_C45    2
#define MDIO_EMULATE_C22     4
```

الـ `mode_support` field بيحدد إيه اللي مدعوم. الـ BCM89811 هو Clause 45 only، بس الـ driver كان بيستخدم `MDIO_SUPPORTS_C22` فقط.

الـ C45 IDs بتتقرأ عن طريق:
```c
static inline bool mdio_phy_id_is_c45(int phy_id)
{
    return (phy_id & MDIO_PHY_ID_C45) && !(phy_id & ~MDIO_PHY_ID_C45_MASK);
}

static inline __u16 mdio_phy_id_prtad(int phy_id)
{
    return (phy_id & MDIO_PHY_ID_PRTAD) >> 5;
}

static inline __u16 mdio_phy_id_devad(int phy_id)
{
    return phy_id & MDIO_PHY_ID_DEVAD;
}
```

الـ PHY ID للـ C45 device بيتكون من `prtad` (port address) و `devad` (device address، المعروف بـ MMD). الكود كان بيعمل `mdio_phy_id_is_c45()` بس `mode_support` مش بيحتوي على `MDIO_SUPPORTS_C45`.

بعدين المهندس بيفحص الـ BASE-T1 advertisement helpers:
```c
/* للقراءة من PHY */
static inline void mii_t1_adv_m_mod_linkmode_t(unsigned long *advertising, u32 lpa)
{
    linkmode_mod_bit(ETHTOOL_LINK_MODE_100baseT1_Full_BIT,
                     advertising, lpa & MDIO_AN_T1_ADV_M_100BT1);
    ...
}

/* للكتابة للـ PHY */
static inline u32 linkmode_adv_to_mii_t1_adv_m_t(unsigned long *advertising)
{
    u32 result = 0;
    if (linkmode_test_bit(ETHTOOL_LINK_MODE_100baseT1_Full_BIT, advertising))
        result |= MDIO_AN_T1_ADV_M_100BT1;
    ...
    return result;
}
```

الـ helpers دي موجودة بس مش بتتاستخدم صح في الـ driver.

#### الحل
تصحيح الـ driver initialization:
```c
/* WRONG */
mdio->mode_support = MDIO_SUPPORTS_C22;

/* CORRECT for Clause 45 only PHY */
mdio->mode_support = MDIO_SUPPORTS_C45;
mdio->mmds = (1 << MDIO_MMD_PMAPMD) |
             (1 << MDIO_MMD_AN);

/* Probe the PHY using C45 */
ret = mdio45_probe(mdio, prtad);
```

وفي الـ Device Tree:
```dts
&mdio0 {
    phy0: ethernet-phy@4 {
        reg = <4>;
        /* Force C45 mode for automotive PHY */
        compatible = "ethernet-phy-ieee802.3-c45";
    };
};
```

للتحقق:
```bash
# قراءة MMD registers مباشرة
ethtool -d eth0

# أو باستخدام mii-tool مع C45 syntax
mdio-tool stmmac-0 c45_read 4 7 514  # devad=7 (AN MMD), reg=514 (T1_ADV_L)
```

#### الدرس المستفاد
الـ `mode_support` في `struct mdio_if_info` لازم تتضبط صح. `MDIO_EMULATE_C22` مفيدة للـ PHYs اللي C45 بس بتستجاوب على C22 registers، بس الـ automotive PHYs زي BCM89811 بتكون C45 only. دايماً اتحقق من الـ PHY datasheet قبل ما تبدأ. الـ helpers الخاصة بـ T1 زي `mii_t1_adv_m_mod_linkmode_t` موجودة تحديداً لـ automotive use cases.

---

### السيناريو الخامس: Custom MDIO Driver بيفشل في unregister بـ RK3562 IoT Sensor

#### العنوان
**Memory leak وكرش عند module unload — custom MDIO driver على RK3562 IoT node**

#### السياق
IoT sensor node بتستخدم Rockchip RK3562 مع custom magnetic field sensor بيتوصل عبر MDIO bus (MDIO كـ general-purpose low-speed bus). المهندس كتب MDIO driver مخصوص للـ sensor وشغّله كـ loadable kernel module.

#### المشكلة
عند تعمل `rmmod` لإزالة الـ module:
```
BUG: kernel NULL pointer dereference, address: 0000000000000018
...
mdio_driver_unregister+0x2c/0x60
sensor_mdio_exit+0x14/0x30
__do_sys_delete_module+0x158/0x280
```

#### التحليل
المهندس بيراجع الـ driver code بتاعه جنب `mdio.h`:

```c
/* الـ driver registration macro */
#define mdio_module_driver(_mdio_driver) \
    module_driver(_mdio_driver, mdio_driver_register, \
                  mdio_driver_unregister)
```

الـ `mdio_driver_register()` و `mdio_driver_unregister()` موجودين:
```c
int mdio_driver_register(struct mdio_driver *drv);
void mdio_driver_unregister(struct mdio_driver *drv);
```

الـ `struct mdio_driver` بتحتاج:
```c
struct mdio_driver {
    struct mdio_driver_common mdiodrv;
    int (*probe)(struct mdio_device *mdiodev);
    void (*remove)(struct mdio_device *mdiodev);
    void (*shutdown)(struct mdio_device *mdiodev);
};
```

المهندس كان شايل الـ `device_free` pointer مش متضبط صح:
```c
/* WRONG — missing device_free callback */
static int sensor_probe(struct mdio_device *mdiodev)
{
    struct sensor_priv *priv;
    priv = devm_kzalloc(&mdiodev->dev, sizeof(*priv), GFP_KERNEL);
    mdiodev_set_drvdata(mdiodev, priv);
    /* forgot to set mdiodev->device_free ! */
    return 0;
}
```

الـ `struct mdio_device` عنده:
```c
struct mdio_device {
    struct device dev;
    ...
    void (*device_free)(struct mdio_device *mdiodev);
    void (*device_remove)(struct mdio_device *mdiodev);
    ...
};
```

الـ `mdio_device_put()` بتعمل:
```c
static inline void mdio_device_put(struct mdio_device *mdiodev)
{
    mdio_device_free(mdiodev);
}
```

و `mdio_device_free()` بتستدعي `mdiodev->device_free()` — اللي كان NULL، فحصل NULL pointer dereference.

الـ `mdio_device_register()` و `mdio_device_remove()` لازم يتعملوا في order معين:
```c
int mdio_device_register(struct mdio_device *mdiodev);
void mdio_device_remove(struct mdio_device *mdiodev);
void mdio_device_free(struct mdio_device *mdiodev);
```

#### الحل
الـ driver الصح:
```c
static void sensor_free(struct mdio_device *mdiodev)
{
    /* devm handles memory, just do any hw cleanup */
    dev_info(&mdiodev->dev, "sensor freed\n");
}

static void sensor_remove(struct mdio_device *mdiodev)
{
    struct sensor_priv *priv = mdiodev_get_drvdata(mdiodev);
    /* stop sensor sampling */
    sensor_stop(priv);
}

static int sensor_probe(struct mdio_device *mdiodev)
{
    struct sensor_priv *priv;

    priv = devm_kzalloc(&mdiodev->dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    /* IMPORTANT: set callbacks before any other operation */
    mdiodev->device_free   = sensor_free;
    mdiodev->device_remove = sensor_remove;

    mdiodev_set_drvdata(mdiodev, priv);

    return sensor_hw_init(mdiodev);
}

static struct mdio_driver sensor_driver = {
    .mdiodrv.driver = {
        .name = "my-sensor",
        .of_match_table = sensor_of_match,
    },
    .probe    = sensor_probe,
    .remove   = sensor_remove,
    .shutdown = sensor_shutdown,
};

/* macro يعمل module_init و module_exit تلقائياً */
mdio_module_driver(sensor_driver);
```

للتحقق قبل الـ rmmod:
```bash
# تأكد ميـه اللي بيستخدم الـ module
lsmod | grep my_sensor

# تحقق من الـ device registration
ls /sys/bus/mdio_bus/devices/

# فحص الـ refcount
cat /sys/bus/mdio_bus/devices/mdio-0\:0a/uevent
```

للـ debug أثناء unregister:
```bash
echo 'file mdio_device.c +p' > /sys/kernel/debug/dynamic_debug/control
rmmod my_sensor 2>&1
dmesg | tail -20
```

#### الدرس المستفاد
الـ `device_free` و `device_remove` callbacks في `struct mdio_device` مش optional — لو مش بتستخدم `mdio_device_create()` اللي بتضبطهم تلقائياً، لازم تضبطهم يدوياً في `probe()`. الـ `mdio_module_driver()` macro بيوفر عليك كتابة `module_init/exit` بس مش بيوفر عليك صح initialization للـ device callbacks. دايماً اختبر `rmmod` مبكر في الـ development cycle مش بس وقت الـ release.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

الـ LWN.net هو المصدر الأساسي لمتابعة تطور MDIO subsystem في الـ kernel.

| المقال | الرابط | الأهمية |
|--------|--------|---------|
| MDIO and ethtool enhancements | [lwn.net/Articles/331581](https://lwn.net/Articles/331581/) | تغييرات مبكرة في الـ MDIO API |
| phylib: Add support for MDIO clause 45 | [lwn.net/Articles/384726](https://lwn.net/Articles/384726/) | إضافة Clause 45 لـ phylib — نقطة تحول |
| Support MDIO devices | [lwn.net/Articles/671456](https://lwn.net/Articles/671456/) | تمثيل non-PHY devices على الـ MDIO bus |
| enetc: Add mdio bus driver for the PCIe MDIO endpoint | [lwn.net/Articles/795095](https://lwn.net/Articles/795095/) | MDIO bus driver مستقل لـ PCIe |
| net: mdio: Add netlink interface | [lwn.net/Articles/925483](https://lwn.net/Articles/925483/) | واجهة netlink للوصول لـ MDIO من userspace |
| net: dsa: realtek: MDIO interface and RTL8367S | [lwn.net/Articles/880470](https://lwn.net/Articles/880470/) | تكامل MDIO مع DSA switches |

---

### التوثيق الرسمي للـ Kernel

الملفات دي موجودة جوه الـ `Documentation/` في source tree:

```
Documentation/networking/phy.rst          — PHY Abstraction Layer الكامل
Documentation/firmware-guide/acpi/dsd/phy.rst  — MDIO and PHYs in ACPI
Documentation/devicetree/bindings/net/    — DT bindings للـ MDIO
```

- **[PHY Abstraction Layer — kernel.org](https://www.kernel.org/doc/html/latest/networking/phy.html)**: الدليل الرسمي لـ `mii_bus`, `phy_device`, وكيفية التسجيل والاستخدام.
- **[MDIO bus and PHYs in ACPI — docs.kernel.org](https://docs.kernel.org/firmware-guide/acpi/dsd/phy.html)**: ربط الـ MDIO devices بـ ACPI firmware.
- **[phylink — kernel.org](https://www.kernel.org/doc/html/latest/networking/sfp-phylink.html)**: الـ phylink layer اللي بيجلس فوق MDIO/PHY layer.

---

### ملفات الـ Source المتعلقة مباشرةً

```
include/linux/mdio.h          — Clause 45 helpers والـ mdio_device struct
include/linux/phy.h           — mii_bus, phy_device, والـ PHY framework كله
drivers/net/phy/mdio_bus.c    — تنفيذ الـ mii_bus registration والـ locking
drivers/net/phy/mdio_device.c — تنفيذ mdio_device_create/register/remove
drivers/net/mdio/             — MDIO bus drivers (GPIO, mux, netlink, ...)
```

---

### نقاشات Mailing List المهمة

- **[Clause 45 and Clause 22 PHYs on one MDIO bus — lore.kernel.org](https://lore.kernel.org/netdev/YjjeMo2YjMZkPIYa@lunn.ch/t/)**: نقاش تقني عميق عن coexistence الـ C22 و C45 على نفس الـ bus.
- **[PATCH net-next: net: mdio: Add netlink interface](https://lore.kernel.org/lkml/20230306204517.1953122-1-sean.anderson@seco.com/)**: الـ patch الأصلي لإضافة netlink interface لـ MDIO.
- **[NET: Support clause 45 MDIO commands at the MDIO bus level](https://linux.kernel.narkive.com/gysqdpat/patch-net-support-clause-45-mdio-commands-at-the-mdio-bus-level)**: الـ patch التاريخي الأول لدعم C45 على مستوى الـ bus.
- **[mtk_eth_soc: implement Clause 45 MDIO access — patchwork](https://patchwork.kernel.org/project/linux-mediatek/patch/YcjepQ2fmkPZ2+pE@makrotopia.org/)**: مثال تطبيقي على إضافة C45 لـ MAC driver.

---

### مراجع Kernel Newbies حسب إصدار الـ Kernel

الـ MDIO subsystem اتطور عبر إصدارات كتير:

| الإصدار | التغيير | الرابط |
|---------|---------|--------|
| Linux 2.6.35 | دعم Clause 45 على مستوى MDIO bus | [kernelnewbies.org/Linux_2_6_35-DriversArch](https://kernelnewbies.org/Linux_2_6_35-DriversArch) |
| Linux 4.3 | دعم multiple MDIO buses في DSA | [kernelnewbies.org/Linux_4.3](https://kernelnewbies.org/Linux_4.3) |
| Linux 4.13 | تحسينات PHY و MDIO | [kernelnewbies.org/Linux_4.13](https://kernelnewbies.org/Linux_4.13) |
| Linux 4.19 | iProc mdio mux clock config | [kernelnewbies.org/Linux_4.19](https://kernelnewbies.org/Linux_4.19) |
| Linux 5.8 | IPQ40xx MDIO support | [kernelnewbies.org/Linux_5.8](https://kernelnewbies.org/Linux_5.8) |
| Linux 6.5 | regmap-based mdio driver | [kernelnewbies.org/Linux_6.5](https://kernelnewbies.org/Linux_6.5) |
| Linux 6.9 | mdio-bcm-unimac لـ ASP 2.2 | [kernelnewbies.org/Linux_6.9](https://kernelnewbies.org/Linux_6.9) |
| Linux 6.13 | تحديثات MDIO/PHY متعددة | [kernelnewbies.org/Linux_6.13](https://kernelnewbies.org/Linux_6.13) |

---

### أدوات Userspace مفيدة

- **[phytool — github.com/wkz/phytool](https://github.com/wkz/phytool)**: أداة command-line لقراءة وكتابة MDIO registers مباشرة من userspace. مفيدة جداً لـ debugging.
- **[mdio-tools — github.com/wkz/mdio-tools](https://github.com/wkz/mdio-tools/blob/master/README.md)**: مجموعة أدوات متقدمة تستخدم الـ netlink interface الجديد للتفاعل مع MDIO buses.
- **[mdio-proxy-module — github.com/nxp-qoriq](https://github.com/nxp-qoriq/mdio-proxy-module)**: kernel module من NXP للوصول لأي MDIO device.

---

### الكتب الموصى بها

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الفصل المهم**: Chapter 17 — Network Drivers
- بيغطي `net_device`, MII layer, وكيفية كتابة network driver بيتعامل مع PHY.
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل المهم**: Chapter 17 — Devices and Modules
- بيشرح device model وكيفية تسجيل devices على buses، وده أساسي لفهم `mdio_device` و `mii_bus`.

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل المهم**: Chapter 15 — Embedded Linux Networking
- بيتكلم عن PHY configuration في embedded systems وكيفية التعامل مع MDIO في بيئات الـ embedded.

#### Understanding Linux Network Internals — Christian Benvenuti
- بيغطي بعمق networking stack من الـ hardware layer لفوق، بما فيه PHY/MDIO management.

---

### مصطلحات البحث

لو عايز تلاقي معلومات أكتر، استخدم الـ search terms دي:

```
# للبحث في الـ mailing lists
linux-kernel mdio clause 45
netdev mdio_bus mii_bus
linux phy mdio_device driver

# للبحث عن التوثيق
linux kernel PHY Abstraction Layer
linux mdio_module_driver
linux mii_bus register

# للبحث عن الـ IEEE standard
IEEE 802.3 Clause 22 MDIO
IEEE 802.3 Clause 45 MDIO
IEEE 802.3ae 10G PHY management

# للبحث في الـ source code
mdio_device_create site:github.com torvalds
mdio45_probe linux kernel
mmd_eee_cap_to_ethtool
```

---

### مواقع مرجعية إضافية

- **[kernel.org/doc/html/latest/networking/phy.html](https://www.kernel.org/doc/html/latest/networking/phy.html)** — التوثيق الكامل لـ PHY layer مع أمثلة.
- **[lore.kernel.org/netdev](https://lore.kernel.org/netdev/)** — أرشيف كامل لمناقشات netdev mailing list، ابحث فيه عن `mdio` أو `phylib`.
- **[patchwork.kernel.org](https://patchwork.kernel.org/project/netdevbpf/list/)** — تتبع الـ patches الجديدة الخاصة بـ MDIO و PHY.
- **[bootlin.com/elixir](https://elixir.bootlin.com/linux/latest/source/include/linux/mdio.h)** — cross-reference للـ source code مع إمكانية تتبع كل function و struct.
## Phase 8: Writing simple module

### الفكرة

هنعمل kprobe على الدالة `mdio_device_register` — دي دالة exported موجودة في `drivers/net/phy/mdio_device.c` وبتتسجل كل مرة MDIO device (زي PHY chip) اتضاف على الـ bus. ده بيخلينا نشوف في real-time إيمتى الـ kernel بيكتشف PHY جديد وعلى أنهي bus وعلى أنهي address.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * mdio_register_probe.c
 * Traces every call to mdio_device_register() using a kprobe.
 * Prints the MDIO bus name and device address each time a new
 * MDIO device (e.g. a PHY chip) registers with the kernel.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>       /* pr_info */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe */
#include <linux/mdio.h>         /* struct mdio_device, struct mii_bus */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("MDIO Tracer");
MODULE_DESCRIPTION("kprobe on mdio_device_register to trace MDIO device hotplug");

/* ------------------------------------------------------------------ */
/* الـ pre-handler: بيتنادى قبل ما mdio_device_register تنفذ فعلاً    */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64 the first argument lives in rdi.
     * On arm64 it's in x0. The kprobes API gives us pt_regs,
     * so we use regs_get_kernel_argument() for portability.
     */
    struct mdio_device *mdiodev =
        (struct mdio_device *)regs_get_kernel_argument(regs, 0);

    if (!mdiodev)
        return 0;

    /*
     * mdiodev->addr  : MDIO bus address (0-31)
     * mdiodev->bus   : pointer to the mii_bus; its ->id field is a
     *                  human-readable string set by the bus driver
     *                  (e.g. "stmmac-0:00").
     */
    pr_info("mdio_probe: new MDIO device on bus='%s' addr=%d flags=0x%x\n",
            mdiodev->bus  ? mdiodev->bus->id : "<no-bus>",
            mdiodev->addr,
            mdiodev->flags);

    return 0; /* 0 means "let the probed function continue normally" */
}

/* ------------------------------------------------------------------ */
/* الـ post-handler: بيتنادى بعد ما mdio_device_register تخلص         */
/* ------------------------------------------------------------------ */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /* نطبع إن التسجيل خلص — الـ return value مش accessible هنا بسهولة */
    pr_info("mdio_probe: mdio_device_register() returned\n");
}

/* ------------------------------------------------------------------ */
/* تعريف الـ kprobe نفسه                                               */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "mdio_device_register", /* الدالة اللي هنـhook عليها */
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

/* ------------------------------------------------------------------ */
/* module_init                                                          */
/* ------------------------------------------------------------------ */
static int __init mdio_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("mdio_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("mdio_probe: kprobe planted on %s @ %p\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit                                                          */
/* ------------------------------------------------------------------ */
static void __exit mdio_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("mdio_probe: kprobe removed\n");
}

module_init(mdio_probe_init);
module_exit(mdio_probe_exit);
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|---|---|
| `linux/module.h` | ماكروهات `module_init` / `module_exit` / `MODULE_LICENSE` |
| `linux/kernel.h` | `pr_info` / `pr_err` |
| `linux/kprobes.h` | `struct kprobe` وكل الـ API الخاص بالـ kprobes |
| `linux/mdio.h` | تعريف `struct mdio_device` و `struct mii_bus` عشان نقرأ الـ fields |

**الـ** `linux/mdio.h` هو الـ header الأصلي للملف اللي بندرسه، بيعرّف `struct mdio_device` اللي بنقرأ منها الـ `addr` والـ `bus`.

---

#### الـ `handler_pre`

الـ pre-handler بيتنادى قبل ما الدالة المستهدفة تنفذ، فبنقدر نشوف الـ arguments الأصلية من غير ما يتغيروا. بنستخدم `regs_get_kernel_argument(regs, 0)` عشان نجيب أول argument (الـ `mdiodev`) بطريقة portable تشتغل على x86-64 وعلى ARM64 من غير ما نحتاج `#ifdef`.

---

#### الـ `handler_post`

**الـ** post-handler بيتنادى بعد ما الدالة المستهدفة ترجع، ومفيد لو عايزين نعمل timing أو نحسب عدد الـ calls. هنا بنكتفي بـ log بسيط يأكد إن التسجيل خلص.

---

#### الـ `struct kprobe`

```c
static struct kprobe kp = {
    .symbol_name = "mdio_device_register",
    ...
};
```

**الـ** `symbol_name` بيخلي الـ kernel يحل الـ address وقت `register_kprobe` من الـ symbol table مباشرةً، فمش محتاجين نكتب عنوان hard-coded. الدالة `mdio_device_register` exported من `drivers/net/phy/mdio_device.c` وموجودة في `linux/mdio.h` كـ declaration.

---

#### الـ `module_init`

بيسجّل الـ kprobe في الـ kernel، ولو فشل (مثلاً الـ symbol مش موجود أو الـ kprobes مش مفعّلة في الـ config) بيرجع error ويطبع الـ error code. الـ `kp.addr` بيتملى تلقائياً بعد `register_kprobe` بعنوان الدالة الفعلي في الذاكرة.

---

#### الـ `module_exit`

بيشيل الـ kprobe من الـ kernel عشان نضمن إن الـ callback مش هيتنادى بعد ما الـ module اتأزل. لو متسجلناش الـ unregister، ممكن يحصل use-after-free لما الـ kernel يحاول يناديه بعد ما الـ module اتحذف من الذاكرة.

---

### طريقة التشغيل

```bash
# بناء الـ module (مع Makefile بسيط)
make -C /lib/modules/$(uname -r)/build M=$PWD modules

# تحميل الـ module
sudo insmod mdio_register_probe.ko

# متابعة اللوج (ستشوف رسالة لكل PHY device يتسجل)
sudo dmesg -w | grep mdio_probe

# إزالة الـ module
sudo rmmod mdio_register_probe
```

**مثال على الـ output** لما PHY chip يتكشفه الـ kernel:

```
[  42.123456] mdio_probe: kprobe planted on mdio_device_register @ ffffffffc0a1b230
[  43.001234] mdio_probe: new MDIO device on bus='stmmac-0' addr=1 flags=0x0
[  43.001300] mdio_probe: mdio_device_register() returned
```

---

### Makefile

```makefile
obj-m += mdio_register_probe.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```
