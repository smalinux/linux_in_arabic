## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

**الـ `include/linux/phy.h`** ينتمي لـ subsystem اسمه **ETHERNET PHY LIBRARY** (اختصاراً **PHYLIB**). المشرفون عليه: Andrew Lunn و Heiner Kallweit. يغطي كل حاجة من headers وحتى drivers لكل الـ PHY chips الموجودة في الـ Linux kernel.

---

### المشكلة اللي بتحلّها PHYLIB — قصة واقعية

تخيّل عندك راوتر أو NAS أو embedded board فيه chip Ethernet. الـ chip دي بتتكون من جزئين:

```
[ CPU ]
   │
[ MAC — Media Access Controller ]   ← جزء الـ SoC أو PCIe card، بيعمل framing وبيبعت البيانات
   │ (RGMII / SGMII / MII / RMII)   ← السلك اللي بيربطهم (interface mode)
[ PHY — Physical Layer Transceiver ] ← chip صغيرة منفصلة بتتكلم مع الكابل الفعلي
   │
[ RJ45 / Fiber / SFP ]              ← المنفذ اللي بتحط فيه الكابل
```

الـ **MAC** بيفهم packets. الـ **PHY** بيفهم فولتات وإشارات analog على السلك. بينهم "لغة" مشتركة اسمها **MII** (Media Independent Interface) وإصداراتها (RMII, RGMII, SGMII...).

المشكلة القديمة قبل PHYLIB: كل driver Ethernet كان بيكتب كود خاص بيه يتحكم في الـ PHY — reset، autoneg، read link status... فجه الـ kernel بيضم مئات الـ drivers كلها بتكتب نفس الكود. Duplication رهيبة، وكل ما يجي PHY chip جديد لازم تعيد الكتابة.

**الحل؟** PHYLIB = طبقة وسطى موحّدة:

```
[ MAC Driver (e.g. stmmac, fec) ]
         │  phy_connect()  │  phy_start()
         ▼
[ PHYLIB — include/linux/phy.h ]
         │ mdiobus_read/write
         ▼
[ PHY Driver (e.g. marvell.c, micrel.c) ]
         │ register read/write via MDIO bus
         ▼
[ PHY Hardware Chip on MDIO bus ]
```

---

### الـ `phy.h` — إيه اللي بيعرّفه بالظبط؟

الملف ده هو **العقد الرسمي** بين ثلاث أطراف:

| الطرف | وظيفته |
|-------|--------|
| **MAC driver** | بيبعت packets، عايز يعرف الـ link speed والـ duplex |
| **PHY driver** | بيعرف يتكلم مع chip معينة (Marvell/Realtek/Micrel...) |
| **PHYLIB core** | الوسيط اللي بيربطهم ويشغّل الـ state machine |

---

### المكونات الأساسية في الملف

#### 1. `enum phy_interface_t` — نوع التوصيل بين MAC والـ PHY

أكتر من 35 mode مختلف — من البسيط للمعقد:

```c
PHY_INTERFACE_MODE_RMII      /* Reduced MII — 10/100 Mbps فقط */
PHY_INTERFACE_MODE_RGMII     /* Reduced Gigabit MII — الأكثر شيوعاً في boards */
PHY_INTERFACE_MODE_SGMII     /* Serial Gigabit — عبر SerDes */
PHY_INTERFACE_MODE_10GBASER  /* 10G BaseR — SFP+ */
PHY_INTERFACE_MODE_100GBASEP /* 100G — للـ data center */
```

ده بيحدد "إيه الكابل الرقمي" بين الـ MAC والـ PHY داخل الـ PCB.

#### 2. `struct mii_bus` — الـ MDIO Bus

الـ **MDIO** هو بروتوكول serial بسيط (زي I2C بس للـ PHYs). كل bus ممكن يضم 32 جهاز (address 0–31).

```c
struct mii_bus {
    int (*read)(struct mii_bus *bus, int addr, int regnum);
    int (*write)(struct mii_bus *bus, int addr, int regnum, u16 val);
    struct mutex mdio_lock;         /* bus يتكلم واحد في الوقت */
    struct mdio_device *mdio_map[PHY_MAX_ADDR]; /* 32 جهاز */
    ...
};
```

الـ MAC driver بيعمل `mdiobus_alloc()` ثم `mdiobus_register()` وبعدين الـ kernel بيعمل scan يلاقي كل الـ PHY chips الموصولة.

#### 3. `enum phy_state` — الـ State Machine

الـ PHY بيمر بحالات واضحة:

```
PHY_DOWN → PHY_READY → PHY_UP → PHY_RUNNING
                                    ↕
                                PHY_NOLINK
                                    ↕
                                PHY_HALTED
                                    ↕
                                PHY_ERROR
                                    ↕
                                PHY_CABLETEST
```

الـ `state_queue` (delayed_work) بيشغّل الـ state machine كل ثانية أو بناءً على interrupt.

#### 4. `struct phy_device` — الـ PHY Instance

ده أهم struct في الملف. كل PHY chip على الـ bus عنده `phy_device` واحد:

```c
struct phy_device {
    struct mdio_device mdio;    /* base device — address على الـ bus */
    const struct phy_driver *drv; /* الـ driver المناسب */
    u32 phy_id;                 /* رقم تعريف الـ chip (من registers 2 & 3) */
    enum phy_state state;       /* فين الـ PHY دلوقتي */
    phy_interface_t interface;  /* RGMII / SGMII / ... */
    int speed;                  /* 10 / 100 / 1000 / 10000 */
    int duplex;                 /* DUPLEX_FULL / DUPLEX_HALF */
    unsigned link:1;            /* هل في link دلوقتي؟ */
    unsigned autoneg:1;         /* autoneg شغال؟ */
    struct net_device *attached_dev; /* الـ network interface المربوط */
    struct delayed_work state_queue; /* الـ polling / state machine */
    ...
};
```

#### 5. `struct phy_driver` — الـ PHY Driver Interface

كل PHY vendor (Marvell, Realtek, Micrel, Broadcom...) بيكتب driver بيملّي الـ struct ده:

```c
struct phy_driver {
    u32 phy_id;               /* يتطابق مع phy_device->phy_id */
    u32 phy_id_mask;          /* mask لمطابقة family كاملة */
    int (*config_init)(struct phy_device *); /* إعداد الـ PHY بعد reset */
    int (*config_aneg)(struct phy_device *); /* إعداد autonegotiation */
    int (*read_status)(struct phy_device *); /* قراءة speed/duplex/link */
    int (*handle_interrupt)(struct phy_device *); /* IRQ handler */
    int (*cable_test_start)(struct phy_device *); /* TDR cable test */
    int (*get_sqi)(struct phy_device *);           /* Signal Quality */
    int (*led_brightness_set)(...);                /* LED control */
    int (*get_plca_cfg)(...);  /* PLCA للـ 10Base-T1S */
    ...
};
```

لو الـ PHY chip مجناش driver خاص، الـ kernel بيستخدم `genphy` (Generic PHY driver) اللي بيشتغل مع أي Clause 22 PHY.

---

### الـ Features المتقدمة في الملف

| Feature | الوصف |
|---------|-------|
| **EEE** (Energy Efficient Ethernet) | توفير طاقة في حالة low traffic — IEEE 802.3az |
| **WoL** (Wake-on-LAN) | إيقاظ الجهاز عن طريق magic packet |
| **MACsec** | تشفير Ethernet layer 2 — offload للـ PHY |
| **PTP / HW Timestamping** | دقة timing للـ IEEE 1588 عبر `mii_timestamper` |
| **SFP Bus** | دعم hot-pluggable fiber modules (SFP/SFP+) |
| **Cable Test (TDR)** | فحص الكابل Copper وتحديد مكان العطل بالسنتيمتر |
| **PLCA** | Physical Layer Collision Avoidance للـ 10Base-T1S (Multi-drop Ethernet) |
| **MSE/SQI** | Signal Quality Index لتشخيص جودة الوصلة |
| **LED Control** | التحكم في LEDs الموجودة على الـ PHY chip مباشرة |
| **Rate Matching** | لما الـ MAC والـ PHY بيشتغلوا بـ speeds مختلفة |
| **phylink** | ربط الـ PHY بالـ MAC بشكل dynamic مع دعم SFP |

---

### قصة: إيه اللي بيحصل لما تحط كابل Ethernet

```
1. الـ MAC driver يعمل mdiobus_alloc() + mdiobus_register()
   → الـ kernel يعمل scan على الـ MDIO bus (addresses 0-31)
   → يلاقي PHY chip على address معين

2. يقرأ registers 2 و 3 من الـ PHY → يجيب phy_id
   مثلاً: 0x0141 0xC912 = Marvell 88E1111

3. يعمل match مع كل الـ phy_drivers المسجّلين
   → يلاقي marvell.c driver

4. يعمل phy_device_register() ← الـ PHY بقى جهاز في الـ kernel

5. الـ MAC driver يعمل phy_connect(net_dev, bus_id, handler, RGMII)
   → الـ PHY اتربط بالـ network interface

6. phy_start() ← بيبدأ الـ state machine
   → يعمل soft reset
   → config_init()
   → config_aneg() ← يعلن capabilities للـ link partner
   → state يتحول لـ PHY_UP

7. لما تحط الكابل:
   → PHY chip يشعل interrupt (أو polling كل ثانية)
   → handle_interrupt() → read_status()
   → speed=1000, duplex=FULL, link=1
   → state: PHY_RUNNING
   → adjust_link() callback للـ MAC driver
   → الـ network interface بقى UP

8. لما تشيل الكابل:
   → link=0 → state: PHY_NOLINK
   → adjust_link() → MAC يوقف TX/RX
```

---

### الملفات المرتبطة — يجب معرفتها

#### Core Files (المحرك الأساسي)

| الملف | الوظيفة |
|-------|---------|
| `drivers/net/phy/phy_device.c` | إنشاء/تسجيل الـ `phy_device`، الـ probing |
| `drivers/net/phy/phy.c` | الـ state machine، autoneg، interrupts |
| `drivers/net/phy/phy-c45.c` | Clause 45 (10G+) generic functions |
| `drivers/net/phy/mdio_bus.c` | الـ MDIO bus driver، scan، lock |
| `drivers/net/phy/phylink.c` | الـ MAC/PHY/SFP glue layer الحديث |
| `drivers/net/phy/phy_caps.c` | قدرات الـ PHY وتفاوض الـ speeds |

#### Headers المهمة

| الملف | الوظيفة |
|-------|---------|
| `include/linux/phy.h` | **الملف ده** — العقد الرئيسي |
| `include/linux/mdio.h` | تعريف `struct mdio_device` و`struct mii_bus` |
| `include/linux/mii.h` | تعريفات Clause 22 registers |
| `include/linux/phylink.h` | الـ phylink API للـ MAC drivers الحديثة |
| `include/linux/phy_fixed.h` | دعم الـ fixed-link (لما مفيش PHY chip) |
| `include/linux/linkmode.h` | bitmap لتمثيل link modes (10M/100M/1G/...) |
| `include/uapi/linux/mdio.h` | Clause 45 MMD register definitions |
| `include/uapi/linux/mii.h` | MII register constants (standard IEEE) |

#### PHY Hardware Drivers (أمثلة)

| الملف | الـ Vendor / Chip |
|-------|-------------------|
| `drivers/net/phy/marvell.c` | Marvell 88E1xxx (الأكثر انتشاراً) |
| `drivers/net/phy/marvell10g.c` | Marvell 88X3310 (10G) |
| `drivers/net/phy/micrel.c` | Microchip/Micrel KSZ series |
| `drivers/net/phy/broadcom.c` | Broadcom BCM54xx |
| `drivers/net/phy/dp83867.c` | Texas Instruments DP83867 |
| `drivers/net/phy/nxp-tja11xx.c` | NXP TJA11xx (Automotive) |
| `drivers/net/phy/microchip_t1s.c` | Microchip 10Base-T1S |

#### MDIO Bus Adapters

| الملف | الوظيفة |
|-------|---------|
| `drivers/net/mdio/of_mdio.c` | MDIO bus من الـ Device Tree |
| `drivers/net/mdio/fwnode_mdio.c` | MDIO bus من ACPI / fwnode |
| `drivers/net/phy/fixed_phy.c` | PHY وهمي لما الـ MAC متوصل مباشرة بالـ switch |

---

### لماذا الملف ده مهم جداً؟

كل جهاز Linux عنده Ethernet — من الـ Raspberry Pi للـ data center server — بيمر بالـ code ده. الملف `phy.h` هو الـ "دستور" اللي على أساسه:

- كل MAC driver بيتعلم يتكلم مع أي PHY
- كل PHY driver بيتعلم يسجّل نفسه مع الـ kernel
- الـ state machine بتضمن إن الـ link دايماً في حالة consistent
- الـ userspace عبر `ethtool` يقدر يتحكم في كل التفاصيل (speed، duplex، autoneg، EEE، WoL...)
## Phase 2: شرح الـ PHY Framework

---

### المشكلة — ليه الـ PHY Framework موجود أصلاً؟

في أي نظام embedded بيتكلم على شبكة Ethernet، في جزئين أساسيين:

- **MAC** (Media Access Controller): الـ hardware اللي جوه الـ SoC، بيتعامل مع الـ frames وبروتوكولات الـ link layer.
- **PHY** (Physical Layer transceiver): الـ chip الخارجي (أو الداخلي) اللي بيحوّل الـ digital data لإشارات كهربية على السلك الفعلي.

المشكلة إن عندنا آلاف الـ PHY chips في السوق (Realtek، Marvell، Micrel، Broadcom، إلخ) كل واحد منهم بيتكلم نفس بروتوكول الـ MDIO/MII على مستوى الـ bus، لكن كل واحد عنده رجيسترات خاصة بيه، initialization sequence مختلف، interrupt behavior مختلف، quirks خاصة بيه.

لو مفيش framework، كل MAC driver كان هيضطر:
1. يعرف كل PHY chip بيتعامل معاه.
2. يحتوي على كود initialization وlink monitoring لكل PHY.
3. يتحمل تعقيد الـ state machine (link up/down/autoneg) بنفسه.

ده كان بيخلي كل الـ MAC drivers bloated وبيكرروا نفس الكود بشكل ضخم.

---

### الحل — إيه اللي عمله الـ kernel؟

الـ kernel عمل **PHY Abstraction Layer** عُرف بـ **phylib** (PHY Library).

الفكرة الأساسية:
- عرّف **struct phy_device** كـ abstraction موحدة لأي PHY chip.
- عرّف **struct phy_driver** كـ vtable بيحط فيها كل vendor الـ operations الخاصة بـ PHY بتاعه.
- الـ framework نفسه بيتولى الـ state machine، الـ polling/interrupt، الـ link monitoring، والتنسيق مع الـ MAC driver.
- الـ MAC driver بيتكلم مع الـ framework بـ API موحدة (`phy_connect`, `phy_start`, `adjust_link`) من غير ما يعرف أي PHY تحته.

---

### الـ Real-World Analogy — بالتعمق الكامل

تخيل إنك بتصمم **نظام توصيل طرود** (courier system).

| مفهوم الـ Analogy | المقابل الكرنل |
|---|---|
| **الطريق الفعلي** (السكة، الأسفلت) | الـ physical medium (الكابل، الألياف) |
| **عربية التوصيل** (شاحنة، موتوسيكل، طيارة) | الـ PHY chip (بيحدد السرعة ونوع الوصلة) |
| **السائق** (بيعرف عربيته بالظبط) | الـ PHY driver (`struct phy_driver`) |
| **مركز الشحن** (بيوزع الطرود ومش فارق معاه نوع العربية) | الـ MAC driver |
| **إدارة أسطول السيارات** (بتسجل العربايا، بتتابع حالتها، بتقول لمركز الشحن لو عربية اتعطلت) | الـ phylib framework |
| **رقم تسجيل العربية** | الـ PHY ID (32-bit OUI + model) |
| **نوع الطريق** (أوتوستراد، شارع عادي) | الـ phy_interface_t (RGMII, MII, SGMII, ...) |

**التعميق:** إدارة الأسطول (phylib) لا تعرف ولا تهتم إذا العربية شاحنة أو موتوسيكل — بتتعامل معاهم كلهم بنفس الـ API. لو العربية اتعطلت (link down)، إدارة الأسطول بتبلّغ مركز الشحن (MAC driver) فوراً عبر callback محدد (`adjust_link`). السائق (PHY driver) هو الوحيد اللي يعرف إزاي يشغل عربيته الخاصة (الـ register map الخاص بيه).

---

### البنية الكبيرة — أين يقع الـ PHY Framework؟

```
 ┌─────────────────────────────────────────────────────────────────────┐
 │                        User Space                                   │
 │              ip link / ethtool / NetworkManager                     │
 └────────────────────────────┬────────────────────────────────────────┘
                              │ syscall
 ┌────────────────────────────▼────────────────────────────────────────┐
 │                      Linux Kernel                                   │
 │                                                                     │
 │   ┌──────────────────────────────────────────────────────────────┐  │
 │   │              Network Stack (net/core, net/ipv4, ...)         │  │
 │   └────────────────────────┬─────────────────────────────────────┘  │
 │                            │ skb                                    │
 │   ┌────────────────────────▼─────────────────────────────────────┐  │
 │   │          Net Device Layer  (struct net_device)               │  │
 │   └────────────────────────┬─────────────────────────────────────┘  │
 │                            │ ndo_start_xmit / ndo_open              │
 │   ┌────────────────────────▼─────────────────────────────────────┐  │
 │   │    MAC Driver (e.g. stmmac, fec, mvneta, dwmac)              │  │
 │   │                                                              │  │
 │   │  phy_connect(dev, bus_id, handler, PHY_INTERFACE_MODE_RGMII) │  │
 │   │  phy_start(phydev)                                           │  │
 │   │  adjust_link(dev) <-- callback when link changes             │  │
 │   └────────────────────────┬─────────────────────────────────────┘  │
 │                            │                                        │
 │   ┌────────────────────────▼─────────────────────────────────────┐  │
 │   │        *** phylib - PHY Framework (CONSUMER API) ***         │  │
 │   │                                                              │  │
 │   │  - PHY State Machine  (DOWN->READY->UP->RUNNING->NOLINK->.)  │  │
 │   │  - Link Monitoring    (polling or interrupt-driven)          │  │
 │   │  - Autoneg Management (config_aneg / aneg_done)              │  │
 │   │  - Device Matching    (phy_id & phy_id_mask)                 │  │
 │   │  - genphy fallback    (Clause 22 standard register ops)      │  │
 │   └────────────────────────┬─────────────────────────────────────┘  │
 │                            │                                        │
 │   ┌────────────────────────▼─────────────────────────────────────┐  │
 │   │       PHY Drivers (PROVIDERS - vendor-specific)              │  │
 │   │                                                              │  │
 │   │  realtek.c   marvell.c   micrel.c   dp83867.c  ...          │  │
 │   │  (struct phy_driver with ops for each specific chip)         │  │
 │   └────────────────────────┬─────────────────────────────────────┘  │
 │                            │                                        │
 │   ┌────────────────────────▼─────────────────────────────────────┐  │
 │   │     MDIO Bus Layer  (struct mii_bus)                         │  │
 │   │     - mdiobus_read() / mdiobus_write()                       │  │
 │   │     - mutex-protected serial access to MDIO pins             │  │
 │   └────────────────────────┬─────────────────────────────────────┘  │
 │                            │                                        │
 └────────────────────────────┼────────────────────────────────────────┘
                              │ MDIO (2-wire: MDC + MDIO)
 ┌────────────────────────────▼────────────────────────────────────────┐
 │                 PHY Hardware (e.g. RTL8211F, KSZ9031)               │
 │                 Clause 22: addr[4:0], reg[4:0], 16-bit data         │
 │                 Clause 45: addr[4:0], devtype[4:0], reg[15:0]       │
 └─────────────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstractions — الـ Structs الرئيسية

#### 1. `struct mii_bus` — الـ MDIO Bus

**الـ MDIO bus** هو بروتوكول تسلسلي بسيط (2 أسلاك: MDC clock + MDIO data) بيستخدمه الـ CPU عشان يقرأ ويكتب في رجيسترات الـ PHY. تعريف الـ struct في الملف:

```c
struct mii_bus {
    struct module *owner;
    const char *name;
    char id[MII_BUS_ID_SIZE];   /* "platform:mdio0" مثلاً */
    void *priv;                 /* private data للـ bus driver */

    /* عمليات القراءة والكتابة — بيوفرهم الـ MAC driver */
    int (*read)(struct mii_bus *bus, int addr, int regnum);
    int (*write)(struct mii_bus *bus, int addr, int regnum, u16 val);
    int (*read_c45)(struct mii_bus *bus, int addr, int devnum, int regnum);
    int (*write_c45)(struct mii_bus *bus, int addr, int devnum,
                     int regnum, u16 val);
    int (*reset)(struct mii_bus *bus);

    struct mutex mdio_lock;           /* يضمن serial access على الـ bus */
    struct mdio_bus_stats stats[PHY_MAX_ADDR]; /* احصائيات per-device */
    struct mdio_device *mdio_map[PHY_MAX_ADDR]; /* خريطة الـ PHY devices (max 32) */
    u32 phy_mask;                     /* addresses يتجاهلها وقت discovery */
    int irq[PHY_MAX_ADDR];            /* IRQ لكل device على الـ bus */

    struct device dev;                /* integration مع kernel device model */
};
```

**نقطة مهمة:** الـ `mdio_lock` بيضمن إن في وقت معين بس thread واحد بيتكلم على الـ bus. ليه؟ لأن MDIO هو serial protocol — مفيش hardware arbitration.

```
MDIO Bus (addr space: 0..31)
┌──────┬──────┬──────┬──────┬──────┐
│ PHY0 │ PHY1 │ PHY2 │ ...  │PHY31 │
│ addr │ addr │ addr │      │ addr │
│  0   │  1   │  2   │      │  31  │
└──────┴──────┴──────┴──────┴──────┘
       ^ كل PHY ليه address من 5 bits
       ^ max 32 device على نفس الـ bus
```

---

#### 2. `struct phy_device` — الـ PHY Instance

ده الـ struct الأهم في الـ framework. بيمثل **instance واحدة من PHY chip محددة** على الـ bus.

```c
struct phy_device {
    struct mdio_device mdio;     /* الـ base class — يحتوي على bus pointer + addr */

    const struct phy_driver *drv; /* الـ driver الخاص بهذا الـ PHY */

    u32 phy_id;                  /* 32-bit OUI + model + revision (من register 2,3) */
    struct phy_c45_device_ids c45_ids; /* لو Clause 45 PHY */

    /* flags */
    unsigned is_c45:1;
    unsigned is_internal:1;      /* PHY داخل الـ SoC نفسه */
    unsigned suspended:1;
    unsigned autoneg:1;
    unsigned link:1;             /* current link state */
    unsigned autoneg_complete:1;

    enum phy_state state;        /* PHY state machine state */
    phy_interface_t interface;   /* MII/RGMII/SGMII/... */

    /* link parameters */
    int speed;                   /* SPEED_10/100/1000/2500/10000 */
    int duplex;                  /* DUPLEX_HALF / DUPLEX_FULL */
    bool pause, asym_pause;

    /* link modes bitmaps (ethtool format) */
    __ETHTOOL_DECLARE_LINK_MODE_MASK(supported);    /* ما يدعمه الـ PHY+MAC معاً */
    __ETHTOOL_DECLARE_LINK_MODE_MASK(advertising);  /* ما بيعلن عنه في autoneg */
    __ETHTOOL_DECLARE_LINK_MODE_MASK(lp_advertising); /* ما أعلن عنه الطرف التاني */

    /* EEE - Energy Efficient Ethernet */
    __ETHTOOL_DECLARE_LINK_MODE_MASK(supported_eee);
    __ETHTOOL_DECLARE_LINK_MODE_MASK(advertising_eee);

    int irq;                     /* PHY_POLL أو PHY_MAC_INTERRUPT أو رقم IRQ */
    struct delayed_work state_queue; /* workqueue للـ state machine */
    struct mutex lock;           /* serialization للـ PHY access */

    /* الوصلة بالـ MAC */
    struct net_device *attached_dev;
    void (*adjust_link)(struct net_device *dev); /* callback لتبليغ الـ MAC */

    struct phylink *phylink;     /* لو بيستخدم الـ phylink subsystem */
    struct sfp_bus *sfp_bus;     /* لو فيه SFP module */
    struct mii_timestamper *mii_ts; /* لو بيدعم PTP timestamping */
};
```

**العلاقة بين الـ structs:**

```
struct mii_bus
    |
    |  mdio_map[addr]
    v
struct mdio_device  <---- embedded in struct phy_device (mdio field)
    | .bus  --> back to mii_bus
    | .addr --> PHY address (0..31)
    |
struct phy_device
    | .drv         --> struct phy_driver
    | .attached_dev --> struct net_device (MAC's netdev)
    | .state       --> enum phy_state
    | .interface   --> phy_interface_t
    v
PHY chip registers (via MDIO bus)
```

---

#### 3. `struct phy_driver` — الـ Driver Vtable

كل vendor بيسجل `struct phy_driver` عشان يعرّف الـ operations الخاصة بـ PHY بتاعه:

```c
struct phy_driver {
    struct mdio_driver_common mdiodrv; /* base — يحتوي على struct device_driver */
    u32 phy_id;           /* مثلاً 0x001CC916 لـ RTL8211F */
    char *name;           /* "RTL8211F Gigabit Ethernet" */
    u32 phy_id_mask;      /* الـ bits المهمة في المقارنة */
    const unsigned long * const features; /* PHY_GBIT_FEATURES مثلاً */
    u32 flags;

    /* Lifecycle */
    int (*probe)(struct phy_device *phydev);
    int (*get_features)(struct phy_device *phydev);
    int (*config_init)(struct phy_device *phydev);
    void (*remove)(struct phy_device *phydev);
    int (*soft_reset)(struct phy_device *phydev);

    /* Power Management */
    int (*suspend)(struct phy_device *phydev);
    int (*resume)(struct phy_device *phydev);

    /* Autonegotiation */
    int (*config_aneg)(struct phy_device *phydev);
    int (*aneg_done)(struct phy_device *phydev);
    int (*read_status)(struct phy_device *phydev); /* reads speed/duplex/link */

    /* Interrupts */
    int (*config_intr)(struct phy_device *phydev);
    irqreturn_t (*handle_interrupt)(struct phy_device *phydev);

    /* Register Access Override */
    int (*read_mmd)(struct phy_device *dev, int devnum, u16 regnum);
    int (*write_mmd)(struct phy_device *dev, int devnum, u16 regnum, u16 val);

    /* Advanced Features */
    int (*set_loopback)(struct phy_device *dev, bool enable, int speed);
    int (*get_sqi)(struct phy_device *dev);        /* Signal Quality Index */
    int (*cable_test_start)(struct phy_device *dev);
    int (*cable_test_get_status)(struct phy_device *dev, bool *finished);
    int (*get_plca_cfg)(struct phy_device *dev, struct phy_plca_cfg *plca_cfg);

    /* LED Control */
    int (*led_brightness_set)(struct phy_device *dev,
                              u8 index, enum led_brightness value);
    int (*led_hw_control_set)(struct phy_device *dev,
                              u8 index, unsigned long rules);
};
```

**كل الـ callbacks اختيارية.** لو الـ driver مسجلش `config_aneg` أو `read_status`، الـ framework هيستخدم الـ **genphy** fallback اللي بيتعامل مع أي PHY بيلتزم بـ IEEE 802.3 Clause 22 standard registers.

---

#### 4. `enum phy_state` — الـ State Machine

الـ framework بيدير state machine كاملة لكل PHY:

```
          phy_probe()
              |
              v
         +---------+
         |  DOWN   |  driver not bound yet
         +----+----+
              | probe() completes
              v
         +---------+
         |  READY  |  PHY ready, MAC not started yet
         +----+----+
              | phy_start()
              v
         +---------+
         |   UP    |  monitoring started, waiting for link
         +----+----+
      +-------+--------+
      | link up        | link down
      v                v
 +----------+    +----------+
 | RUNNING  |<-->|  NOLINK  |  polling/IRQ بيراقب التغيير
 +----+-----+    +----+-----+
      |               |
      +-------+--------+
              | phy_stop()
              v
         +---------+
         | HALTED  |  stopped, no polling
         +---------+
              | phy_start() again
              v
             UP
```

الانتقالات بتحصل إما بـ **polling** (delayed_work بيشتغل كل HZ) أو **interrupt** من الـ PHY نفسه.

---

#### 5. `phy_interface_t` — نوع الواجهة بين MAC وPHY

ده enum بيحدد **الـ electrical/protocol interface** بين الـ MAC والـ PHY. مش نوع الكابل الخارجي، ده الـ bus الداخلي بين الـ chips:

| Mode | Speed | الوصف |
|---|---|---|
| `MII` | 10/100M | الأصل، 16 pin parallel |
| `RMII` | 10/100M | Reduced MII، 8 pins |
| `RGMII` | 10/100/1000M | Reduced GMII، 12 pins |
| `RGMII_ID` | 10/100/1000M | RGMII + internal TX+RX delay |
| `RGMII_RXID` | 10/100/1000M | RGMII + internal RX delay فقط |
| `RGMII_TXID` | 10/100/1000M | RGMII + internal TX delay فقط |
| `SGMII` | 10/100/1000M | Serial GMII، lane واحدة 1.25Gbps |
| `1000BASEX` | 1G | 802.3z fiber/backplane |
| `10GBASER` | 10G | XFI/SFI single-lane 10G |
| `USXGMII` | حتى 10G | Universal Serial XGMII |

**مثال عملي على ARM SoC:**

```
i.MX8 SoC                    RTL8211F PHY
+-----------------+          +-----------------+
|                 |  RGMII   |                 |
|   ENET MAC      |<-------->|   PHY core      |
|   (FEC driver)  | 4 tx +   |                 |
|                 | 4 rx +   |   Autoneg       |
|                 | clocks   |   1000BASE-T    |
+-----------------+          +--------+--------+
         |                            | RJ45
         | MDIO (MDC+MDIO)            | Ethernet Cable
         +----------------------------+
           .interface = PHY_INTERFACE_MODE_RGMII_ID
```

---

### الـ MDIO Clause 22 vs Clause 45

ده مفهوم مهم لازم تفهمه عشان تفهم `phy_c45_device_ids` و `read_mmd`/`write_mmd`.

**Clause 22** (القديم):
- Address: 5 bits (0..31)
- Register: 5 bits (0..31) — بس 32 register
- Data: 16 bits
- مناسب للـ Gigabit PHYs والأقل

**Clause 45** (الجديد، للـ 10G+):
- Address: 5 bits (0..31)
- Device Type (DEVAD): 5 bits (مثل PMA/PMD=1, PCS=3, AN=7)
- Register: 16 bits — 65536 register per device
- Data: 16 bits
- بيسمح بـ **MMD** (MDIO Manageable Device) متعددة في نفس الـ chip

```c
/* قراءة Clause 22 - مباشرة */
phy_read(phydev, MII_BMSR);          /* Basic Mode Status Register */

/* قراءة Clause 45 - عبر MMD */
phy_read_mmd(phydev, MDIO_MMD_AN,   /* device = Autoneg */
             MDIO_AN_ADVERTISE);     /* register */
```

الـ framework بيدعم كلاهما. لو الـ PHY هو Clause 45، `is_c45 = 1` وبيتخزن الـ IDs في `c45_ids`.

---

### الـ genphy — الـ Generic PHY Driver

ده مفهوم محوري في الـ framework: **لو الـ PHY بيلتزم بـ IEEE 802.3 standard registers، مش محتاج driver خاص.**

```c
/* كل vendor-specific PHY driver ممكن يعتمد على الـ genphy functions */
int genphy_read_abilities(struct phy_device *phydev); /* يقرأ MII_PHYSID1/2 */
int genphy_config_aneg(struct phy_device *phydev);    /* يكتب MII_ADVERTISE */
int genphy_read_status(struct phy_device *phydev);    /* يقرأ MII_BMSR, MII_LPA */
int genphy_soft_reset(struct phy_device *phydev);     /* يكتب BMCR_RESET */
int genphy_suspend(struct phy_device *phydev);        /* يكتب BMCR_PDOWN */
int genphy_resume(struct phy_device *phydev);
```

**مثال:** RTL8211F driver بيعرّف `config_init` لأن فيه رجيسترات خاصة بيه للـ LED وdelay tuning، لكن بيستخدم `genphy_read_status` للـ link state لأن الـ status registers standard.

---

### الـ Link Mode Bitmaps

الـ framework بيستخدم **bitmap** عشان يعبر عن capabilities الـ link بدلاً من integers بسيطة. ده عشان كل link mode هو combination من (speed + duplex + medium):

```c
/* في struct phy_device */
__ETHTOOL_DECLARE_LINK_MODE_MASK(supported);    /* OR بين PHY وMAC capabilities */
__ETHTOOL_DECLARE_LINK_MODE_MASK(advertising);  /* اللي بيتعلن عنه في autoneg */
__ETHTOOL_DECLARE_LINK_MODE_MASK(lp_advertising); /* اللي أعلن عنه الـ link partner */
```

```
supported bitmap (مثال):
bit 0  --> 10baseT_Half
bit 1  --> 10baseT_Full
bit 2  --> 100baseT_Half
bit 3  --> 100baseT_Full
bit 11 --> 1000baseT_Full
...
```

الـ framework بيعمل AND بين `supported` الـ MAC وبيلت الـ PHY عشان يوصل للـ link modes المشتركة.

---

### ما يملكه الـ Framework مقابل ما يفوّضه للـ Drivers

| المسؤولية | Framework (phylib) | PHY Driver |
|---|---|---|
| State machine (UP/DOWN/RUNNING) | نعم | لا |
| Polling / interrupt scheduling | نعم | لا |
| Device discovery (scan MDIO bus) | نعم | لا |
| ID matching (phy_id & mask) | نعم | بيحدد الـ phy_id فقط |
| API للـ MAC drivers | نعم | لا |
| Register initialization | لا | نعم |
| Autoneg configuration | genphy fallback | ممكن يـ override |
| Link status reading | genphy fallback | ممكن يـ override |
| Interrupt enable/clear | لا | نعم |
| Vendor-specific features (LED, PLCA, SQI) | بيعرّف الـ interface | بينفذ الـ ops |
| Cable diagnostics (TDR) | orchestration | التنفيذ |
| Power management (suspend/resume) | بيستدعي | بينفذ |
| EEE negotiation | جزئياً | جزئياً |
| SFP/module management | via sfp_bus | لا |

---

### Flow عملي — من Boot لـ Link Up

```
1. MAC driver probe()
       |
       v
2. mdiobus_alloc() + register read/write ops
   mdiobus_register()  --> scan: phy_device_create() for each found PHY
       |
       v
3. phy_connect(netdev, "bus_id", adjust_link_cb, RGMII)
       |
       +-> phy_attach_direct()
               +-> driver_match_device() --> compares phy_id & mask
                   +-> driver probe() --> config_init() --> get_features()
       v
4. phy_start(phydev)
       |
       +-> state: DOWN -> UP
       +-> schedule delayed_work (state_queue)
               |
               v (every 1 second or on interrupt)
5. phy_state_machine()
       |
       +-> genphy_update_link() --> reads BMSR register
       +-> if link up --> read_status() --> gets speed/duplex
       +-> state: UP -> RUNNING
       +-> adjust_link(netdev) --> MAC driver updates its config
               |
               v
6. MAC driver adjust_link():
       +-> phydev->speed, phydev->duplex, phydev->link
       +-> reconfigure MAC clocks/mode to match PHY
```

---

### الـ phylink — التطور التالي للـ Framework

**phylink** هو subsystem أحدث بيحل مشكلة إن بعض الـ MAC controllers بيدعموا multiple interface modes (SGMII + RGMII + 1000BaseX) وبيحتاجوا يـ switch بينهم dynamically حسب الـ PHY أو الـ SFP module المتصل.

الـ phylib التقليدي كان بيفترض إن الـ interface ثابت. الـ phylink جه عشان يتعامل مع الـ SFP modules وتغيير الـ interface dynamically. ده subsystem منفصل (`struct phylink` موجود كـ pointer في `struct phy_device`)، لكن كلاهما بيتعاملوا مع نفس الـ `struct phy_driver` من تحت.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Enums والـ Config Options — Cheatsheet

#### PHY Driver Flags (`phy_driver::flags` + standalone macros)

| Flag | Value | المعنى |
|---|---|---|
| `PHY_IS_INTERNAL` | `0x00000001` | الـ PHY داخلي مدمج مع الـ MAC |
| `PHY_RST_AFTER_CLK_EN` | `0x00000002` | يحتاج reset بعد تفعيل الـ clock |
| `PHY_POLL_CABLE_TEST` | `0x00000004` | يستخدم polling أثناء cable test |
| `PHY_ALWAYS_CALL_SUSPEND` | `0x00000008` | يُستدعى suspend دايماً حتى لو مش مشغّل |
| `MDIO_DEVICE_IS_PHY` | `0x80000000` | الـ MDIO device هو PHY فعلاً |

#### PHY Generic Dev Flags (`phy_device::dev_flags` upper bits)

| Flag | Value | المعنى |
|---|---|---|
| `PHY_F_NO_IRQ` | `0x80000000` | لا تستخدم interrupt |
| `PHY_F_RXC_ALWAYS_ON` | `0x40000000` | الـ RX clock يظل شغّال حتى أثناء الـ suspend |
| `PHY_F_KEEP_PREAMBLE_BEFORE_SFD` | `0x20000000` | يحتاج preamble كامل |

#### IRQ Mode Constants

| Constant | Value | المعنى |
|---|---|---|
| `PHY_POLL` | `-1` | لا يوجد interrupt، يستخدم polling |
| `PHY_MAC_INTERRUPT` | `-2` | الـ MAC هو اللي يتعامل مع الـ interrupt |

#### `enum phy_state` — حالات الـ State Machine

| State | الانتقال منها |
|---|---|
| `PHY_DOWN` | البداية، لسه ما probe اتعمل |
| `PHY_READY` | بعد probe، PHY جاهز لكن MAC مش جاهز |
| `PHY_UP` | MAC جاهز، interrupts تبدأ هنا |
| `PHY_NOLINK` | PHY شغّال بس السلك مش متصل |
| `PHY_RUNNING` | السلك متصل وبيتبعت/بيتستقبل بيانات |
| `PHY_CABLETEST` | جاري اختبار السلك (cable test) |
| `PHY_HALTED` | PHY موقوف، لا polling ولا interrupts |
| `PHY_ERROR` | حصل خطأ في الـ PHY |

#### `enum phy_interface_t` — أنواع الـ Interface المهمة

| Interface | الوصف |
|---|---|
| `PHY_INTERFACE_MODE_MII` | MII كلاسيكي، 10/100 Mbps |
| `PHY_INTERFACE_MODE_GMII` | Gigabit MII |
| `PHY_INTERFACE_MODE_RGMII` | Reduced Gigabit MII — الأكثر شيوعاً |
| `PHY_INTERFACE_MODE_RGMII_ID` | RGMII مع delay داخلي على RX+TX |
| `PHY_INTERFACE_MODE_RGMII_RXID` | RGMII مع delay داخلي على RX فقط |
| `PHY_INTERFACE_MODE_RGMII_TXID` | RGMII مع delay داخلي على TX فقط |
| `PHY_INTERFACE_MODE_SGMII` | Serial Gigabit MII |
| `PHY_INTERFACE_MODE_1000BASEX` | 1000Base-X، للـ fiber |
| `PHY_INTERFACE_MODE_10GBASER` | 10G BaseR / XFI / SFI |
| `PHY_INTERFACE_MODE_USXGMII` | Universal Serial 10GE MII |
| `PHY_INTERFACE_MODE_INTERNAL` | MAC+PHY مدمجين، ما فيش واجهة خارجية |

#### `enum phy_led_modes`

| Mode | المعنى |
|---|---|
| `PHY_LED_ACTIVE_HIGH` | الـ LED يضيء لما الإشارة `1` |
| `PHY_LED_ACTIVE_LOW` | الـ LED يضيء لما الإشارة `0` |
| `PHY_LED_INACTIVE_HIGH_IMPEDANCE` | مش متصل (High-Z) لما idle |

#### `enum phy_mse_channel` — قنوات قياس MSE

| Channel | المعنى |
|---|---|
| `PHY_MSE_CHANNEL_A..D` | قياس per-pair للـ pairs A-D |
| `PHY_MSE_CHANNEL_WORST` | أسوأ pair يرجعه الـ hardware |
| `PHY_MSE_CHANNEL_LINK` | قياس aggregate للـ link كله |

#### `enum link_inband_signalling`

| Value | المعنى |
|---|---|
| `LINK_INBAND_DISABLE` | ممكن تعطّل in-band signalling |
| `LINK_INBAND_ENABLE` | ممكن تفعّله بدون bypass |
| `LINK_INBAND_BYPASS` | ممكن تفعّله مع bypass |

#### PHY ID Match Masks

| Macro | Mask | يطابق |
|---|---|---|
| `PHY_ID_MATCH_EXACT` | `GENMASK(31,0)` | exact UID |
| `PHY_ID_MATCH_MODEL` | `GENMASK(31,4)` | نفس الموديل، أي revision |
| `PHY_ID_MATCH_VENDOR` | `GENMASK(31,10)` | نفس الـ vendor، أي موديل |

#### PHY MSE Capability Flags (`supported_caps`)

| Flag | المعنى |
|---|---|
| `PHY_MSE_CAP_CHANNEL_A..D` | يدعم قياس per-pair للـ channel المحدد |
| `PHY_MSE_CAP_WORST_CHANNEL` | يقدر يحدد أسوأ channel |
| `PHY_MSE_CAP_LINK` | قياس link-wide aggregate |
| `PHY_MSE_CAP_AVG` | يدعم متوسط MSE |
| `PHY_MSE_CAP_PEAK` | يدعم peak MSE |
| `PHY_MSE_CAP_WORST_PEAK` | latched worst-case peak MSE |

#### MDIO Bus States

| State | المعنى |
|---|---|
| `MDIOBUS_ALLOCATED` | تم تخصيص الذاكرة |
| `MDIOBUS_REGISTERED` | مسجّل في الـ kernel |
| `MDIOBUS_UNREGISTERED` | تمت إزالة التسجيل |
| `MDIOBUS_RELEASED` | تم تحرير الذاكرة |

---

### 1. كل الـ Structs المهمة

---

#### `struct mdio_device` (من `linux/mdio.h`)

**الغرض:** الـ base object لأي device على بص MDIO، PHY أو غيره.

| Field | النوع | المعنى |
|---|---|---|
| `dev` | `struct device` | kernel device object — يورثه كل device |
| `bus` | `struct mii_bus *` | البص اللي الجهاز متصل بيه |
| `addr` | `int` | عنوان الـ device على البص (0–31) |
| `flags` | `int` | فلاقز خاصة (مثلاً `MDIO_DEVICE_FLAG_PHY`) |
| `reset_gpio` | `struct gpio_desc *` | GPIO pin للـ reset |
| `reset_ctrl` | `struct reset_control *` | reset controller (بديل للـ GPIO) |
| `reset_assert_delay` | `uint` | delay بعد assert reset بالـ microseconds |
| `reset_deassert_delay` | `uint` | delay بعد deassert reset |

**يتصل بـ:** `mii_bus` (عن طريق `bus`)، وكل `phy_device` يـembed واحد منه.

---

#### `struct mdio_driver_common` (من `linux/mdio.h`)

**الغرض:** base مشترك لكل drivers على MDIO bus.

| Field | النوع | المعنى |
|---|---|---|
| `driver` | `struct device_driver` | kernel driver object |
| `flags` | `int` | `MDIO_DEVICE_FLAG_PHY` يميّز PHY driver |

---

#### `struct mdio_bus_stats`

**الغرض:** إحصائيات لكل device على البص لتتبع عدد العمليات والأخطاء.

| Field | النوع | المعنى |
|---|---|---|
| `transfers` | `u64_stats_t` | مجموع العمليات (reads + writes) |
| `errors` | `u64_stats_t` | عدد العمليات الفاشلة |
| `writes` | `u64_stats_t` | عدد عمليات الكتابة |
| `reads` | `u64_stats_t` | عدد عمليات القراءة |
| `syncp` | `struct u64_stats_sync` | synchronization لتحديث الإحصائيات atomically |

---

#### `struct mii_bus`

**الغرض:** يُمثّل بص MDIO كامل. الـ controller بيسجّل bus واحد، وعليه حتى 32 PHY.

| Field | النوع | المعنى |
|---|---|---|
| `owner` | `struct module *` | الـ module المالك |
| `name` | `const char *` | اسم البص |
| `id[61]` | `char` | unique ID للبص |
| `priv` | `void *` | بيانات خاصة للـ driver |
| `read` | `fn ptr` | قراءة register بـ Clause 22 |
| `write` | `fn ptr` | كتابة register بـ Clause 22 |
| `read_c45` | `fn ptr` | قراءة register بـ Clause 45 |
| `write_c45` | `fn ptr` | كتابة register بـ Clause 45 |
| `reset` | `fn ptr` | reset البص كله |
| `stats[32]` | `struct mdio_bus_stats` | إحصائيات per-device (32 device max) |
| `mdio_lock` | `struct mutex` | mutex يحمي الوصول للبص — مهم جداً |
| `parent` | `struct device *` | الـ device الأب (مثلاً MAC أو bridge) |
| `state` | `enum` | حالة البص: ALLOCATED/REGISTERED/... |
| `dev` | `struct device` | kernel device representation |
| `mdio_map[32]` | `struct mdio_device *` | قائمة كل الـ MDIO devices على البص |
| `phy_mask` | `u32` | bitmask يتجاهل addresses معينة أثناء الـ probe |
| `phy_ignore_ta_mask` | `u32` | يتجاهل TA/read failure لعناوين معينة |
| `irq[32]` | `int` | interrupt لكل PHY حسب عنوانه |
| `reset_delay_us` | `int` | عرض نبضة الـ GPIO reset بالـ microseconds |
| `reset_gpiod` | `struct gpio_desc *` | GPIO descriptor للـ reset |
| `shared_lock` | `struct mutex` | يحمي الـ shared state بين PHYs |
| `shared[32]` | `struct phy_package_shared *` | shared state لـ PHYs في نفس الـ package |

---

#### `struct phy_c45_device_ids`

**الغرض:** يخزّن معرّفات الـ MMDs (MDIO Manageable Devices) في PHY من نوع Clause 45.

| Field | النوع | المعنى |
|---|---|---|
| `devices_in_package` | `u32` | قيمة register "devices in package" |
| `mmds_present` | `u32` | bitmask الـ MMDs الموجودة فعلاً |
| `device_ids[32]` | `u32` | device ID لكل MMD |

---

#### `struct phy_device`

**الغرض:** الـ object المركزي — يُمثّل instance واحدة من PHY. ده القلب اللي كل الـ framework بيتمحور حواليه.

| Field | النوع | المعنى |
|---|---|---|
| `mdio` | `struct mdio_device` | base MDIO device — embeds dev+bus+addr |
| `drv` | `const struct phy_driver *` | الـ driver المرتبط بالـ PHY |
| `devlink` | `struct device_link *` | رابط بين الـ PHY والـ MAC إذا كانوا على modules مختلفة |
| `phyindex` | `u32` | unique ID عالمي للـ PHY (زي ifindex) |
| `phy_id` | `u32` | UID المقروء من registers الـ PHY |
| `c45_ids` | `struct phy_c45_device_ids` | IDs للـ MMDs في Clause 45 |
| `is_c45` | `unsigned:1` | يستخدم Clause 45 addressing |
| `is_internal` | `unsigned:1` | PHY مدمج داخل الـ MAC |
| `is_pseudo_fixed_link` | `unsigned:1` | switch أو fixed link مش PHY حقيقي |
| `suspended` | `unsigned:1` | في وضع suspended |
| `loopback_enabled` | `unsigned:1` | loopback مفعّل |
| `is_on_sfp_module` | `unsigned:1` | الـ PHY على وحدة SFP |
| `mac_managed_pm` | `unsigned:1` | الـ MAC هو المسؤول عن suspend/resume |
| `wol_enabled` | `unsigned:1` | Wake-on-LAN مفعّل |
| `autoneg` | `unsigned:1` | autonegotiation مفعّل |
| `link` | `unsigned:1` | حالة الـ link الحالية (1=up) |
| `interrupts` | `unsigned:1` | الـ interrupts مفعّلة |
| `irq_suspended` | `unsigned:1` | PHY suspended، ارجّأ معالجة الـ interrupt |
| `state` | `enum phy_state` | حالة الـ state machine |
| `dev_flags` | `u32` | flags خاصة بالـ driver |
| `interface` | `phy_interface_t` | نوع الـ interface مع الـ MAC |
| `possible_interfaces` | `bitmap` | bitmap لكل interfaces ممكنة |
| `speed` | `int` | السرعة الحالية بالـ Mbps |
| `duplex` | `int` | half/full duplex |
| `supported` | `linkmode_mask` | link modes المدعومة من MAC+PHY |
| `advertising` | `linkmode_mask` | الـ modes المعلن عنها حالياً |
| `lp_advertising` | `linkmode_mask` | ما أعلن عنه الطرف التاني |
| `supported_eee` | `linkmode_mask` | EEE modes المدعومة |
| `advertising_eee` | `linkmode_mask` | EEE modes المعلن عنها |
| `enable_tx_lpi` | `bool` | الـ MAC يبعت LPI للـ PHY |
| `eee_cfg` | `struct eee_config` | إعدادات EEE من المستخدم |
| `irq` | `int` | رقم الـ interrupt (-1 = لا يوجد) |
| `priv` | `void *` | بيانات خاصة بالـ driver |
| `shared` | `struct phy_package_shared *` | shared state مع PHYs في نفس الـ package |
| `state_queue` | `struct delayed_work` | الـ work queue للـ state machine |
| `lock` | `struct mutex` | mutex يحمي الـ PHY device نفسه |
| `sfp_bus` | `struct sfp_bus *` | بص SFP لو الـ PHY على fiber port |
| `phylink` | `struct phylink *` | phylink layer فوق الـ PHY |
| `attached_dev` | `struct net_device *` | الـ NIC المتصل بالـ PHY |
| `mii_ts` | `struct mii_timestamper *` | hardware timestamping callbacks |
| `ports` | `struct list_head` | قائمة الـ PHY ports |
| `link_down_events` | `uint` | عدد مرات انقطاع الـ link |
| `adjust_link` | `fn ptr` | callback للـ MAC لما يتغير الـ link state |
| `phy_link_change` | `fn ptr` | callback للـ phylink |
| `macsec_ops` | `const struct macsec_ops *` | offloading ops للـ MACsec |
| `oatc14_sqi_capability` | `struct phy_oatc14_sqi_capability` | SQI capability للـ 10Base-T1S |

---

#### `struct phy_driver`

**الغرض:** يُمثّل driver لنوع PHY معين. كل vendor يسجّل واحد أو أكتر. يحتوي على كل الـ ops اللي الـ framework بيستدعيها.

| Field | النوع | المعنى |
|---|---|---|
| `mdiodrv` | `struct mdio_driver_common` | base driver info |
| `phy_id` | `u32` | UID الـ PHY اللي الـ driver بيتعامل معاه |
| `name` | `char *` | اسم الـ driver |
| `phy_id_mask` | `u32` | الـ bits المهمة في UID المقارنة |
| `features` | `const unsigned long *` | link modes المدعومة |
| `flags` | `u32` | PHY_IS_INTERNAL وغيرها |
| `driver_data` | `const void *` | بيانات ثابتة خاصة بالـ driver |
| `soft_reset` | `fn ptr` | software reset |
| `config_init` | `fn ptr` | تهيئة الـ PHY بعد الـ reset |
| `probe` | `fn ptr` | يُستدعى أثناء الـ discovery |
| `get_features` | `fn ptr` | يستعلم عن capabilities الـ hardware |
| `config_aneg` | `fn ptr` | يضبط autonegotiation |
| `aneg_done` | `fn ptr` | يتحقق إذا انتهت الـ autonegotiation |
| `read_status` | `fn ptr` | يقرأ السرعة والـ duplex بعد الـ link |
| `config_intr` | `fn ptr` | يفعّل/يعطّل الـ interrupts |
| `handle_interrupt` | `fn ptr` | معالج الـ IRQ |
| `suspend` / `resume` | `fn ptr` | power management |
| `read_mmd` / `write_mmd` | `fn ptr` | قراءة/كتابة MMD registers |
| `cable_test_start` | `fn ptr` | يبدأ اختبار السلك |
| `cable_test_get_status` | `fn ptr` | يستعلم عن نتيجة الاختبار |
| `get_sqi` / `get_sqi_max` | `fn ptr` | Signal Quality Indicator |
| `get_mse_capability` | `fn ptr` | MSE measurement capabilities |
| `get_mse_snapshot` | `fn ptr` | MSE snapshot لقناة معينة |
| `get_plca_cfg` / `set_plca_cfg` | `fn ptr` | PLCA configuration |
| `led_brightness_set` | `fn ptr` | ضبط سطوع LED |
| `led_hw_control_set/get` | `fn ptr` | hardware LED control |
| `get_next_update_time` | `fn ptr` | polling interval ديناميكي |
| `attach_mii_port` | `fn ptr` | ربط MII port بالـ PHY |
| `attach_mdi_port` | `fn ptr` | ربط MDI port بالـ PHY |

---

#### `struct phy_led`

**الغرض:** يُمثّل LED واحد مرتبط بـ PHY.

| Field | النوع | المعنى |
|---|---|---|
| `list` | `struct list_head` | عنصر في قائمة `phy_device::leds` |
| `phydev` | `struct phy_device *` | الـ PHY المالك |
| `led_cdev` | `struct led_classdev` | LED class device للـ kernel |
| `index` | `u8` | رقم الـ LED |

---

#### `struct phy_tdr_config`

**الغرض:** إعدادات TDR (Time Domain Reflectometry) لاختبار السلك.

| Field | المعنى |
|---|---|
| `first` | أول نقطة قياس (بالـ cm) |
| `last` | آخر نقطة قياس (بالـ cm) |
| `step` | المسافة بين نقاط القياس (بالـ cm) |
| `pair` | bitmask الـ pairs المطلوب قياسها (`PHY_PAIR_ALL = -1`) |

---

#### `struct phy_plca_cfg`

**الغرض:** إعدادات PLCA (Physical Layer Collision Avoidance) للـ 10Base-T1S multi-drop networks.

| Field | المعنى |
|---|---|
| `version` | read-only، إصدار الـ PLCA register map |
| `enabled` | تفعيل/تعطيل الـ PLCA |
| `node_id` | معرّف العقدة المحلية (0–254، 255 = معطّلة) |
| `node_cnt` | عدد العقد الكلي (للـ coordinator فقط) |
| `to_tmr` | transmit opportunity timer بالـ bit-times |
| `burst_cnt` | عدد الـ frames الإضافية في كل TO |
| `burst_tmr` | انتظار MAC لإرسال frame جديد قبل إنهاء burst |

---

#### `struct phy_plca_status`

**الغرض:** يُبلّغ عن حالة PLCA.

| Field | المعنى |
|---|---|
| `pst` | `true` = BEACON نشط = PLCA شغّال |

---

#### `struct phy_mse_capability`

**الغرض:** يصف قدرات قياس MSE (Mean Square Error) للـ PHY في الـ link mode الحالي.

| Field | المعنى |
|---|---|
| `max_average_mse` | أقصى قيمة average MSE (للـ normalization) |
| `max_peak_mse` | أقصى قيمة peak MSE |
| `refresh_rate_ps` | معدل تحديث hardware بالـ picoseconds |
| `num_symbols` | عدد الـ symbols في كل sample |
| `supported_caps` | bitmask من `PHY_MSE_CAP_*` |

---

#### `struct phy_mse_snapshot`

**الغرض:** snapshot لقيم MSE من نافذة قياس واحدة.

| Field | المعنى |
|---|---|
| `average_mse` | متوسط MSE على نافذة القياس |
| `peak_mse` | أعلى قيمة MSE في النافذة |
| `worst_peak_mse` | latched high-water mark — يُصفّر عند القراءة |

---

#### `struct phy_oatc14_sqi_capability`

**الغرض:** SQI capability للـ OATC14 10Base-T1S PHY.

| Field | المعنى |
|---|---|
| `updated` | هل البيانات اتحدّثت من الـ hardware |
| `sqi_max` | أقصى مستوى SQI مدعوم |
| `sqiplus_bits` | عدد bits الـ SQI+ (0=غير مدعوم، 3-8=عدد المستويات) |

---

### 2. مخططات علاقات الـ Structs (ASCII)

```
                    +-----------------------------------------+
                    |           struct mii_bus                |
                    |  id[], name, priv                       |
                    |  read/write/read_c45/write_c45/reset    |
                    |  mdio_lock (mutex)                      |
                    |  shared_lock (mutex)                    |
                    |  mdio_map[32] ----------------+         |
                    |  stats[32]                    |         |
                    |  irq[32]                      |         |
                    |  shared[32] --------+         |         |
                    +----------+----------|---------+---------+
                               |          |         |
                    +----------+          |         |
                    v                     v         |
         +--------------------+  +-------------------+
         |  struct mdio_      |  | struct phy_       |
         |  device            |  | package_shared    |
         |  dev (device)      |  | (shared state for |
         |  bus (*mii_bus) ---+  |  same-pkg PHYs)   |
         |  addr (0-31)       |  +-------------------+
         |  flags             |
         |  reset_gpio        |
         +--------+-----------+
                  |  (embedded as first member)
                  v
    +--------------------------------------------------+
    |              struct phy_device                   |
    |  mdio  <-- embeds mdio_device above              |
    |  drv ------------------------------------------+ |
    |  phy_id, c45_ids                               | |
    |  state (phy_state)                             | |
    |  interface (phy_interface_t)                   | |
    |  speed, duplex, link                           | |
    |  supported/advertising (linkmode bitmaps)      | |
    |  lock (mutex)                                  | |
    |  state_queue (delayed_work)                    | |
    |  attached_dev -------------------+             | |
    |  phylink ---------------+        |             | |
    |  sfp_bus ----------+    |        |             | |
    |  mii_ts --------+  |    |        |             | |
    |  leds (list) ---|----|-----|------|-----------+| |
    |  ports (list)   |  |    |        |            || |
    |  shared --------|--|----|---------            || |
    +-----------------|--|----|---------            |+-+
                      |  |    |    |                |
         +------------+  |    |    |                |
         v               |    |    v                v
+-------------------+    |    |  +-----------+  +----------+
| struct mii_       |    |    |  |  struct   |  | struct   |
| timestamper       |    |    |  |  net_     |  | phy_led  |
| (PTP/hwtstamp)    |    |    |  |  device   |  | list     |
+-------------------+    |    |  +-----------+  | phydev * |
                         |    |                 +----------+
                         v    |
                  +----------+|
                  |  struct  ||
                  |  phylink ||
                  +----------+|
                              v
                     +--------------+
                     |  struct      |
                     |  sfp_bus     |
                     +--------------+

    +--------------------------------------------------+
    |              struct phy_driver                   |
    |  mdiodrv (mdio_driver_common)                   |
    |  phy_id, phy_id_mask, name                      |
    |  features, flags                                |
    |  soft_reset, config_init, probe                 |
    |  config_aneg, read_status                       |
    |  suspend, resume                                |
    |  get_mse_capability, get_mse_snapshot           |
    |  get_plca_cfg, set_plca_cfg                     |
    |  led_* ops                                      |
    +--------------------------------------------------+
                    ^
                    +------- phydev->drv (pointer back)
```

---

### 3. Lifecycle Diagrams

#### أ. Lifecycle للـ `mii_bus`

```
Driver Probe
    |
    v
mdiobus_alloc()  --> state = MDIOBUS_ALLOCATED
    |                phy_mask, irq[], priv تتعبّأ
    v
driver sets:
  bus->read/write/reset/name/id
    |
    v
mdiobus_register()
    |
    +-- يضبط الـ GPIO reset
    +-- يسجّل bus->dev في sysfs
    +-- يمسح العناوين 0-31 (probe)
    |      +-- mdiobus_scan_c22() لكل عنوان
    |              +-- get_phy_device() --> phy_device_register()
    +-- state = MDIOBUS_REGISTERED
    |
    v
  [يعمل -- PHYs بيتتصلوا]
    |
    v
mdiobus_unregister()
    +-- يشيل كل mdio_map entries
    +-- state = MDIOBUS_UNREGISTERED
    |
    v
mdiobus_free()
    +-- state = MDIOBUS_RELEASED
```

#### ب. Lifecycle للـ `phy_device`

```
mdiobus_scan_c22(bus, addr)
    |
    v
get_phy_device(bus, addr, false)
    +-- يقرأ phy_id من registers
    +-- phy_device_create() --> kmalloc + INIT
    |      state = PHY_DOWN
    v
phy_device_register(phydev)
    +-- يسجّل mdio_device في kernel
    +-- يضيف phydev في mdio_map[]
    v
phy_attach(netdev, bus_id, interface)
  او
phy_connect(netdev, bus_id, handler, interface)
    +-- يبحث عن driver مناسب (phy_id & mask)
    +-- phy_driver->probe(phydev)
    +-- phy_init_hw()
    |      +-- drv->soft_reset()
    |      +-- drv->config_init()
    +-- phydev->attached_dev = netdev
    +-- state = PHY_READY
    v
phy_start(phydev)
    +-- state = PHY_UP
    +-- phy_start_machine() --> schedule delayed_work
    +-- phy_request_interrupt() لو interrupt mode
    v
  phy_state_machine() تشتغل بشكل دوري
    +-- PHY_UP --> يقرأ link state
    |      link up  --> PHY_RUNNING
    |      no link  --> PHY_NOLINK
    +-- PHY_RUNNING --> يراقب التغييرات
    +-- PHY_NOLINK  --> ينتظر العودة
    v
phy_stop(phydev)
    +-- state = PHY_HALTED
    +-- يوقف الـ state machine
    v
phy_disconnect(phydev)
    +-- phy_detach()
    +-- attached_dev = NULL
    v
phy_device_remove(phydev)
    +-- phy_device_free() --> kfree
```

---

### 4. Call Flow Diagrams

#### أ. phy_connect — ربط الـ MAC بالـ PHY

```
Ethernet driver (MAC)
  calls phy_connect(netdev, bus_id, adjust_link_cb, interface)
    |
    v
  phy_attach(netdev, bus_id, interface)
    |
    +-> phy_find_next() -- يبحث في mii_bus
    |
    +-> phy_attach_direct(netdev, phydev, 0, interface)
    |       |
    |       +-> driver_match() -- يبحث عن phy_driver مطابق
    |       |        phy_id & phy_id_mask == drv->phy_id & drv->phy_id_mask
    |       |
    |       +-> drv->probe(phydev)
    |       |
    |       +-> phy_init_hw(phydev)
    |       |       +-> drv->soft_reset()
    |       |       +-> drv->config_init()
    |       |
    |       +-> phydev->attached_dev = netdev
    |
    +-> phydev->adjust_link = adjust_link_cb
```

#### ب. phy_state_machine — دورة الـ State Machine

```
delayed_work fires (phy_state_machine)
    |
    v
  mutex_lock(&phydev->lock)
    |
    +--[PHY_UP]---------> phy_check_link_status()
    |                         +-> drv->read_status(phydev)
    |                         |        (او genphy_read_status)
    |                         |        +-> mdiobus_read() via mdio_lock
    |                         +-> link changed?
    |                         |     yes --> phydev->adjust_link(netdev)
    |                         +-> state --> PHY_RUNNING or PHY_NOLINK
    |
    +--[PHY_RUNNING]----> phy_check_link_status()
    |                         +-> link lost? --> PHY_NOLINK
    |                               --> adjust_link(netdev) [link down]
    |
    +--[PHY_NOLINK]-----> phy_check_link_status()
    |                         +-> link up? --> PHY_RUNNING
    |                               --> adjust_link(netdev) [link up]
    |
    +--[PHY_CABLETEST]--> drv->cable_test_get_status()
    |                         +-> done? --> PHY_UP
    |
    +-> mutex_unlock(&phydev->lock)
         +-> reschedule delayed_work (HZ او get_next_update_time jiffies)
```

#### ج. Register Read — phy_read_mmd

```
phy driver
  calls phy_read_mmd(phydev, devad, regnum)
    |
    v
  mutex_lock(&phydev->mdio.bus->mdio_lock)  <-- bus-level lock
    |
    +-> phydev->is_c45?
    |     YES --> mii_bus->read_c45(bus, addr, devad, regnum)
    |     NO  --> (indirect C22 access)
    |           mii_bus->write(bus, addr, MII_MMD_CTRL, devad)
    |           mii_bus->write(bus, addr, MII_MMD_DATA, regnum)
    |           mii_bus->write(bus, addr, MII_MMD_CTRL, devad|MII_MMD_CTRL_NOINCR)
    |           mii_bus->read(bus, addr, MII_MMD_DATA)
    |
    v
  mutex_unlock(&phydev->mdio.bus->mdio_lock)
    |
    v
  return value (او negative errno)
```

#### د. تسجيل PHY Driver جديد

```
PHY vendor module init
    |
    v
module_phy_driver(my_phy_drivers[])
    +-- phy_drivers_register(drv_array, count, THIS_MODULE)
    |       |
    |       +-> for each driver:
    |               phy_driver_register(drv)
    |                   +-> drv->mdiodrv.flags |= MDIO_DEVICE_FLAG_PHY
    |                   +-> driver_register(&drv->mdiodrv.driver)
    |                           +-> kernel matches existing PHY devices
    v
  [driver جاهز -- kernel يستدعي probe لاي PHY UID يطابق]
    |
    v
module exit
    +-> phy_drivers_unregister(drv_array, count)
```

---

### 5. استراتيجية الـ Locking

الـ PHY framework بيستخدم **ثلاث طبقات من الـ locks** واضحة ومحددة المسؤوليات:

---

#### الطبقة الأولى: `mii_bus::mdio_lock` (mutex)

**يحمي:** الوصول الفيزيائي للبص MDIO — القراءة والكتابة في registers الـ PHY.

```c
/* كل قراءة او كتابة على البص تمر من هنا */
mutex_lock(&bus->mdio_lock);
bus->read(bus, addr, regnum);
mutex_unlock(&bus->mdio_lock);
```

- **Thread-safe:** منع أكتر من caller من الوصول للبص في نفس الوقت.
- **الـ** `phy_read()` و `phy_write()` بيأخذوا الـ lock داخلياً.
- **الـ** `__phy_read()` و `__phy_write()` (prefix `__`) = **لا تأخذ الـ lock** — caller مسؤول.
- ترتيب: لو محتاج الـ lock دا وlock تاني، لازم `mdio_lock` يتاخد **أولاً بعد** phydev lock.

---

#### الطبقة الثانية: `phy_device::lock` (mutex)

**يحمي:** الـ state machine وكل fields الـ `phy_device` اللي بتتغير.

```c
/* state machine وكل تغيير في state */
mutex_lock(&phydev->lock);
phydev->state = PHY_RUNNING;
mutex_unlock(&phydev->lock);
```

- الـ `phy_state_machine()` بتأخد الـ lock في البداية وتشيله في النهاية.
- `phy_start()` و `phy_stop()` بيأخدوه.
- **ترتيب (lock ordering):** `phydev->lock` أولاً، **ثم** `mdio_lock` لو احتجنا الاثنين.

---

#### الطبقة الثالثة: `mii_bus::shared_lock` (mutex)

**يحمي:** الـ `shared` state المشترك بين PHYs في نفس الـ package (مثلاً 4-port PHY chip).

```c
/* لما PHYs متعددة تحتاج تتشارك حالة */
mutex_lock(&bus->shared_lock);
shared_data->something = value;
mutex_unlock(&bus->shared_lock);
```

- مش موجود دايماً — يُستخدم فقط لو `CONFIG_PHY_PACKAGE` مفعّل.

---

#### ترتيب الـ Locks (Lock Ordering) — مهم جداً لتجنب deadlock

```
rtnl_lock (rtnl_lock)            <-- الخارجي -- يحمي net_device
     |
     v
phy_device::lock (mutex)         <-- يحمي PHY state
     |
     v
mii_bus::shared_lock (mutex)     <-- يحمي shared package state
     |
     v
mii_bus::mdio_lock (mutex)       <-- يحمي MDIO bus access
```

**قاعدة صارمة:** لا تأخد lock من مستوى أعلى وانت شايل lock من مستوى أدنى — deadlock مضمون.

---

#### ملاحظات إضافية على الـ Locking

| السياق | يجوز تستدعي |
|---|---|
| Interrupt context | **لا شيء** — كل ops ممنوعة من interrupt |
| أخذت `mdio_lock` | `__phy_read/write` فقط |
| أخذت `phydev->lock` | أي op على الـ state |
| مش أخذت lock | `phy_read/write` (تأخد lock بنفسها) |

**الـ** `sfp_bus` و `phylink` بيتعدّلوا تحت الـ `rtnl_lock` — مش تحت `phydev->lock`.
## Phase 4: شرح الـ Functions

---

### ملخص عام — Cheatsheet

#### فئة: MDIO Bus Management

| Function | Signature Summary | الغرض |
|---|---|---|
| `mdiobus_alloc` | `() → struct mii_bus *` | تخصيص bus بدون private data |
| `mdiobus_alloc_size` | `(size_t) → struct mii_bus *` | تخصيص bus مع private data |
| `mdiobus_register` | macro → `__mdiobus_register(bus, THIS_MODULE)` | تسجيل الـ bus في kernel |
| `devm_mdiobus_register` | macro → managed register | تسجيل مع devm cleanup |
| `mdiobus_unregister` | `(bus) → void` | إلغاء تسجيل الـ bus |
| `mdiobus_free` | `(bus) → void` | تحرير ذاكرة الـ bus |
| `devm_mdiobus_alloc_size` | `(dev, sizeof_priv) → struct mii_bus *` | تخصيص managed |
| `mdio_find_bus` | `(name) → struct mii_bus *` | البحث عن bus بالاسم |
| `mdiobus_scan_c22` | `(bus, addr) → struct phy_device *` | scan لـ C22 PHY |

#### فئة: PHY Device Lifecycle

| Function | الغرض |
|---|---|
| `phy_device_create` | إنشاء `phy_device` يدويًا |
| `get_phy_device` | قراءة PHY ID وإنشاء device |
| `phy_device_register` | تسجيل في device model |
| `phy_device_free` | تحرير ذاكرة PHY device |
| `phy_device_remove` | إزالة من device model |
| `phy_init_hw` | تهيئة HW (reset + config_init) |
| `fwnode_phy_find_device` | إيجاد PHY من firmware node |
| `fwnode_get_phy_node` | قراءة phy-handle من DT |

#### فئة: PHY Attach/Detach/Connect

| Function | الغرض |
|---|---|
| `phy_attach` | ربط PHY بـ net_device بالاسم |
| `phy_attach_direct` | ربط مباشر بـ pointer |
| `phy_connect` | attach + ربط handler + start |
| `phy_connect_direct` | connect مباشر |
| `phy_disconnect` | قطع الاتصال وإيقاف |
| `phy_detach` | فصل PHY عن الـ MAC |

#### فئة: PHY State Machine

| Function | الغرض |
|---|---|
| `phy_start` | تشغيل state machine (UP) |
| `phy_stop` | إيقاف (HALTED) |
| `phy_start_machine` | بدء workqueue polling |
| `phy_stop_machine` | إيقاف workqueue |
| `phy_state_machine` | حلقة state machine الفعلية |
| `phy_trigger_machine` | re-schedule state machine |
| `phy_mac_interrupt` | إشعار من MAC عند link change |
| `phy_error` | الدخول في PHY_ERROR state |

#### فئة: Register I/O

| Function | Bus Lock؟ | الغرض |
|---|---|---|
| `phy_read` | نعم (implicit) | قراءة C22 register |
| `phy_write` | نعم | كتابة C22 register |
| `__phy_read` | يجب أخذه قبلها | قراءة بدون lock |
| `__phy_write` | يجب أخذه قبلها | كتابة بدون lock |
| `phy_modify` | نعم | read-modify-write ذرية |
| `__phy_modify` | يجب أخذه | بدون lock |
| `phy_modify_changed` | نعم | modify + إرجاع هل تغيّر |
| `phy_set_bits` | نعم | set bits فقط |
| `phy_clear_bits` | نعم | clear bits فقط |
| `phy_read_mmd` | نعم | قراءة MMD register |
| `phy_write_mmd` | نعم | كتابة MMD register |
| `__phy_read_mmd` | يجب أخذه | قراءة MMD بدون lock |
| `__phy_write_mmd` | يجب أخذه | كتابة MMD بدون lock |
| `phy_modify_mmd` | نعم | modify MMD register |
| `phy_modify_mmd_changed` | نعم | modify MMD + changed flag |
| `phy_set_bits_mmd` | نعم | set bits في MMD |
| `phy_clear_bits_mmd` | نعم | clear bits في MMD |

#### فئة: Paged Register Access

| Function | الغرض |
|---|---|
| `phy_save_page` | حفظ current page + lock |
| `phy_select_page` | التبديل لـ page معين |
| `phy_restore_page` | استعادة page قديمة + unlock |
| `phy_read_paged` | قراءة register من page معينة |
| `phy_write_paged` | كتابة register في page معينة |
| `phy_modify_paged` | modify register في page معينة |

#### فئة: Autonegotiation

| Function | الغرض |
|---|---|
| `phy_config_aneg` | تهيئة autoneg أو forced mode |
| `phy_start_aneg` | بدء autoneg مع lock |
| `_phy_start_aneg` | بدء autoneg بدون lock (internal) |
| `phy_restart_aneg` | إعادة تشغيل autoneg |
| `phy_aneg_done` | التحقق هل اكتمل autoneg |
| `genphy_config_aneg` | generic C22 config aneg |
| `genphy_restart_aneg` | generic C22 restart aneg |
| `genphy_aneg_done` | generic C22 aneg done check |
| `genphy_c45_config_aneg` | generic C45 config aneg |
| `genphy_c45_restart_aneg` | generic C45 restart aneg |

#### فئة: Generic PHY (genphy)

| Function | الغرض |
|---|---|
| `genphy_read_abilities` | قراءة link modes المدعومة |
| `genphy_update_link` | تحديث link state |
| `genphy_read_status` | قراءة speed/duplex بعد autoneg |
| `genphy_setup_forced` | ضبط forced speed/duplex |
| `genphy_soft_reset` | soft reset عبر BMCR bit |
| `genphy_suspend` | وضع PHY في power-down |
| `genphy_resume` | إخراج PHY من power-down |
| `genphy_loopback` | ضبط loopback mode |
| `genphy_read_lpa` | قراءة link partner advertisement |
| `genphy_read_master_slave` | قراءة master/slave config |

#### فئة: EEE

| Function | الغرض |
|---|---|
| `phy_support_eee` | تفعيل EEE advertisement |
| `phy_disable_eee` | تعطيل EEE advertisement |
| `phy_init_eee` | تهيئة EEE بعد link up |
| `phy_ethtool_get_eee` | قراءة EEE config لـ ethtool |
| `phy_ethtool_set_eee` | ضبط EEE config من ethtool |
| `phy_eee_tx_clock_stop_capable` | التحقق من دعم TX clock stop |
| `phy_eee_rx_clock_stop` | تفعيل/تعطيل RX clock stop |

#### فئة: Ethtool Integration

| Function | الغرض |
|---|---|
| `phy_ethtool_ksettings_get` | قراءة link settings |
| `phy_ethtool_ksettings_set` | ضبط link settings |
| `phy_ethtool_nway_reset` | restart autoneg من ethtool |
| `phy_ethtool_get_strings` | أسماء الـ stats |
| `phy_ethtool_get_stats` | قيم الـ stats |
| `phy_ethtool_set_wol` | ضبط WoL |
| `phy_ethtool_get_wol` | قراءة WoL |

---

### Group 1: MDIO Bus Allocation & Registration

**الـ MDIO bus** هو القناة الفيزيائية (أو المنطقية) اللي بتربط الـ MAC بالـ PHY chips. الـ kernel بيمثلها بـ `struct mii_bus`. كل PHY driver محتاج أوّلًا يخصّص الـ bus ويسجّله قبل ما يقدر يعمل أي read/write للـ PHY registers.

---

#### `mdiobus_alloc_size`

```c
struct mii_bus *mdiobus_alloc_size(size_t size);
```

بتخصص `struct mii_bus` + private data بـ `size` bytes إضافية جنبه في نفس allocation. بتضبط الـ state على `MDIOBUS_ALLOCATED` وبتشيل `owner` وبترجع pointer للـ bus.

- **Parameters:**
  - `size`: حجم الـ private data المطلوبة (ممكن صفر)
- **Return:** pointer لـ `mii_bus` أو NULL لو فشل الـ allocation
- **Key details:** الـ private data بتكون accessible عبر `bus->priv` (pointer حساب offset). يُستخدم في probe context.

---

#### `mdiobus_alloc`

```c
static inline struct mii_bus *mdiobus_alloc(void)
{
    return mdiobus_alloc_size(0);
}
```

الـ wrapper الأبسط لحالات إن الـ driver مش محتاج private data في نفس allocation.

---

#### `devm_mdiobus_alloc_size`

```c
struct mii_bus *devm_mdiobus_alloc_size(struct device *dev, int sizeof_priv);
```

نفس `mdiobus_alloc_size` بس بتربط الـ resource بالـ device. لو الـ device اتحرر (device_unregister أو driver unbind)، الـ bus بيتحرر أوتوماتيك بدون ما تحتاج `mdiobus_free` يدوي.

- **Parameters:**
  - `dev`: الـ device الـ owner
  - `sizeof_priv`: حجم الـ private data
- **Caller context:** probe function فقط

---

#### `__mdiobus_register` / `mdiobus_register`

```c
int __mdiobus_register(struct mii_bus *bus, struct module *owner);
#define mdiobus_register(bus) __mdiobus_register(bus, THIS_MODULE)
```

بتسجّل الـ MDIO bus في الـ kernel:
1. بتتحقق الـ bus valid
2. بتعمل reset لو موجود (`bus->reset`)
3. بتسكان كل الـ 32 address وبتحاول تعمل probe لكل PHY موجود
4. بتسجّل `bus->dev` في device model

- **Return:** 0 لو نجح، أو negative errno
- **Locking:** بيأخذ `mdio_lock` أثناء الـ scan
- **Key detail:** ماكرو `mdiobus_register` بيمرر `THIS_MODULE` أوتوماتيك

---

#### `mdiobus_unregister`

```c
void mdiobus_unregister(struct mii_bus *bus);
```

عكس `mdiobus_register`: بتحرر كل PHY devices المتسجلة على الـ bus وبتزيل الـ bus من الـ device model. يجب استدعاؤها قبل `mdiobus_free`.

---

#### `mdiobus_free`

```c
void mdiobus_free(struct mii_bus *bus);
```

بتحرر الذاكرة المخصصة لـ `mii_bus`. يجب استدعاؤها بعد `mdiobus_unregister` فقط.

---

#### `mdio_find_bus`

```c
struct mii_bus *mdio_find_bus(const char *mdio_name);
```

بتبحث في الـ device model عن MDIO bus بالاسم المحدد. مفيدة لما driver يحتاج يوصل لـ bus مسجّل من driver تاني.

- **Return:** pointer للـ bus مع refcount زاد، أو NULL
- **Key detail:** يجب عمل `put_device` على الـ bus بعد الاستخدام

---

#### `mdiobus_scan_c22`

```c
struct phy_device *mdiobus_scan_c22(struct mii_bus *bus, int addr);
```

بتعمل probe لـ PHY بـ Clause 22 protocol على address محدد. بتقرأ PHYID1/PHYID2 registers وبتبني `phy_device` لو لقت PHY.

- **Return:** pointer لـ `phy_device` أو ERR_PTR
- **Caller context:** يُستخدم من `mdiobus_register` أثناء الـ scan

---

### Group 2: PHY Device Creation & Registration

هاي المجموعة بتمثل lifecycle بناء الـ `phy_device` من الصفر وتسجيله في الـ kernel. الـ core بيعملده أوتوماتيك أثناء bus scan، بس ممكن تعمله يدوي.

---

#### `phy_device_create`

```c
struct phy_device *phy_device_create(struct mii_bus *bus, int addr,
                                     u32 phy_id, bool is_c45,
                                     struct phy_c45_device_ids *c45_ids);
```

بتبني `phy_device` جديد وبتملى فيه الـ metadata الأساسي. بتخصص الذاكرة، بتضبط `phy_id`، `is_c45`، `mdio.addr`، وبتربطه بالـ `bus`.

- **Parameters:**
  - `bus`: الـ MDIO bus اللي عليه الـ PHY
  - `addr`: الـ MDIO address (0-31)
  - `phy_id`: الـ UID المقروء من الـ PHY registers
  - `is_c45`: true لو Clause 45
  - `c45_ids`: struct بـ C45 device IDs (nullable لو C22)
- **Return:** pointer أو ERR_PTR(-ENOMEM)
- **Key detail:** الـ device مش مسجّل بعد في الـ device model — محتاج `phy_device_register`

---

#### `get_phy_device`

```c
struct phy_device *get_phy_device(struct mii_bus *bus, int addr, bool is_c45);
```

بتقرأ PHY ID registers عبر MDIO وبتعمل `phy_device_create`. دي الـ high-level function اللي بيستدعيها الـ bus scan.

**Pseudocode flow:**
```
get_phy_device(bus, addr, is_c45):
    r = mdiobus_read(PHYID1)
    r |= mdiobus_read(PHYID2) << 16
    if r == 0xFFFFFFFF → return ERR_PTR(-ENODEV)
    if is_c45:
        ids = read all MMD device IDs
    return phy_device_create(bus, addr, r, is_c45, &ids)
```

---

#### `phy_device_register`

```c
int phy_device_register(struct phy_device *phy);
```

بتسجّل الـ `phy_device` في الـ Linux device model (وبالتالي بتعمل driver matching). بعد النجاح، الـ PHY يبدأ يدخل في probe للـ PHY driver المناسب.

- **Return:** 0 أو negative errno
- **Side effects:** بتعمل device_add، بتخلي phy driver bind لو موجود

---

#### `phy_device_free`

```c
void phy_device_free(struct phy_device *phydev);
```

بتخفّض الـ refcount على الـ `phy_device`. لو وصل صفر، بتحرر الذاكرة. يُستخدم في error paths بعد `phy_device_create` مباشرة.

---

#### `phy_device_remove`

```c
void phy_device_remove(struct phy_device *phydev);
```

عكس `phy_device_register`: بتشيل الـ PHY من device model وبتفصل الـ driver.

---

#### `phy_init_hw`

```c
int phy_init_hw(struct phy_device *phydev);
```

بتهيّئ الـ PHY hardware: بتنادي `soft_reset` لو موجود، تطبّق fixups، تنادي `config_init`. يُستدعى بعد attach وبعد كل resume.

**Pseudocode flow:**
```
phy_init_hw(phydev):
    if drv->soft_reset: drv->soft_reset(phydev)
    apply_fixups(phydev)
    if drv->config_init: drv->config_init(phydev)
    return 0
```

---

#### `fwnode_phy_find_device` / `fwnode_get_phy_node`

```c
struct phy_device *fwnode_phy_find_device(struct fwnode_handle *phy_fwnode);
struct fwnode_handle *fwnode_get_phy_node(const struct fwnode_handle *fwnode);
```

الأولى بترجع الـ `phy_device` المرتبط بـ firmware node (DT أو ACPI). الثانية بتقرأ `phy-handle` property من الـ fwnode وبترجع الـ node الخاص بالـ PHY.

---

### Group 3: PHY Attach, Connect, Detach

**الـ attach** بيربط PHY بـ `net_device` بدون ما يبدأ الـ state machine. **الـ connect** بيعمل attach + يسجّل `adjust_link` callback + يشغّل state machine.

---

#### `phy_attach`

```c
struct phy_device *phy_attach(struct net_device *dev, const char *bus_id,
                              phy_interface_t interface);
```

بتبحث عن الـ PHY بالاسم (`bus_id` بصيغة `"mdio_bus_id:phy_addr"`) وبتعمل `phy_attach_direct`. الأبسط لما الـ MAC driver يعرف bus ID من DT.

- **Return:** pointer لـ PHY أو ERR_PTR
- **Locking:** بتأخذ rtnl_lock داخليًا

---

#### `phy_attach_direct`

```c
int phy_attach_direct(struct net_device *dev, struct phy_device *phydev,
                      u32 flags, phy_interface_t interface);
```

الـ low-level attach:
1. بتربط `phydev->attached_dev = dev`
2. بتضبط `phydev->interface`
3. بتعمل `phy_init_hw` لو مش initialized
4. بتقرأ features عبر `get_features`

- **Parameters:**
  - `dev`: الـ `net_device`
  - `phydev`: PHY مباشر
  - `flags`: dev_flags اختيارية
  - `interface`: الـ MII mode
- **Locking:** بتأخذ rtnl_lock

---

#### `phy_connect`

```c
struct phy_device *phy_connect(struct net_device *dev, const char *bus_id,
                               void (*handler)(struct net_device *),
                               phy_interface_t interface);
```

الـ high-level API اللي معظم MAC drivers بيستخدموها. بتعمل:
1. `phy_attach`
2. تسجيل `adjust_link` callback
3. `phy_start_machine`

- **`handler`**: الـ callback اللي الـ MAC بيسجّله عشان يعمل update على link speed/duplex

---

#### `phy_connect_direct`

```c
int phy_connect_direct(struct net_device *dev, struct phy_device *phydev,
                       void (*handler)(struct net_device *),
                       phy_interface_t interface);
```

نفس `phy_connect` بس بتاخد `phy_device *` مباشر بدلًا من bus_id string.

---

#### `phy_disconnect`

```c
void phy_disconnect(struct phy_device *phydev);
```

بتوقف الـ state machine، بتلغي الـ `adjust_link`، وبتعمل `phy_detach`.

---

#### `phy_detach`

```c
void phy_detach(struct phy_device *phydev);
```

بتفصل الـ PHY عن الـ `net_device`: بتصفّر `attached_dev`، بتوقف SFP bus، بتحرر الـ device link reference.

---

### Group 4: PHY State Machine

الـ PHY في phylib ليه **state machine** تشتغل على `delayed_work`. الـ states هي: `PHY_DOWN → PHY_READY → PHY_UP → PHY_RUNNING/PHY_NOLINK → PHY_HALTED`.

```
PHY_DOWN
   │ probe
   ▼
PHY_READY
   │ phy_start()
   ▼
PHY_UP
   │ link detected
   ├──────────────► PHY_RUNNING
   │                    │ link lost
   │                    ▼
   └──────────────► PHY_NOLINK
                         │ phy_stop()
                         ▼
                    PHY_HALTED
```

---

#### `phy_start`

```c
void phy_start(struct phy_device *phydev);
```

بتانتقل بالـ PHY من `PHY_HALTED` أو `PHY_READY` لـ `PHY_UP`. بتفعّل الـ interrupts لو متاحة. بتعمل schedule للـ state machine workqueue.

- **Caller context:** يُستدعى من MAC driver عند `ndo_open`
- **Locking:** بيأخذ `phydev->lock`

---

#### `phy_stop`

```c
void phy_stop(struct phy_device *phydev);
```

بتحوّل الـ PHY لـ `PHY_HALTED`، بتعطّل الـ interrupts، بتلغي الـ scheduled work. الـ link بيعتبر down.

- **Caller context:** يُستدعى من MAC driver عند `ndo_stop`
- **Locking:** بيأخذ `phydev->lock`

---

#### `phy_start_machine`

```c
void phy_start_machine(struct phy_device *phydev);
```

بتعمل schedule للـ `state_queue` delayed_work. يُستدعى مرة واحدة بعد `phy_connect`. الـ work بيعيد جدولة نفسه بعدين.

---

#### `phy_stop_machine`

```c
void phy_stop_machine(struct phy_device *phydev);
```

بتلغي الـ delayed_work وبتنتظر إنه يخلص. آمنة تُستدعى بعد `phy_stop`.

---

#### `phy_state_machine`

```c
void phy_state_machine(struct work_struct *work);
```

الـ work handler الفعلي للـ state machine. بيشتغل في process context.

**Pseudocode flow:**
```
phy_state_machine(work):
    phydev = container_of(work, state_queue)
    mutex_lock(&phydev->lock)

    switch (phydev->state):
        PHY_UP:
            phydev->drv->read_status()
            if link: → PHY_RUNNING
            else:    → PHY_NOLINK

        PHY_RUNNING | PHY_NOLINK:
            phydev->drv->read_status()
            if link changed: notify adjust_link

        PHY_CABLETEST:
            drv->cable_test_get_status()
            if done: → PHY_UP

    mutex_unlock(&phydev->lock)
    reschedule after HZ
```

---

#### `phy_trigger_machine`

```c
void phy_trigger_machine(struct phy_device *phydev);
```

بتعمل re-schedule للـ state machine فورًا بدلًا من انتظار الـ timer. يُستدعى من interrupt handler أو لما الـ MAC يكتشف link change.

---

#### `phy_mac_interrupt`

```c
void phy_mac_interrupt(struct phy_device *phydev);
```

يُستدعى لما الـ MAC driver يكتشف link change وبيعمل interrupt (mode `PHY_MAC_INTERRUPT`). بيعمل trigger للـ state machine.

---

#### `phy_error`

```c
void phy_error(struct phy_device *phydev);
```

بتحوّل الـ PHY لـ `PHY_ERROR` state وبتعمل trigger للـ state machine. يُستدعى لما read/write فشل فشل حاد.

---

### Group 5: Register I/O — C22 (Clause 22)

**Clause 22** هو بروتوكول MDIO الأصلي: 5-bit PHY address، 5-bit register address، 16-bit data.

---

#### `phy_read` / `__phy_read`

```c
static inline int phy_read(struct phy_device *phydev, u32 regnum);
static inline int __phy_read(struct phy_device *phydev, u32 regnum);
```

**الـ locked version** (`phy_read`) بتأخذ `mdio_lock` وبترخيه أوتوماتيك عبر `mdiobus_read`.
**الـ unlocked version** (`__phy_read`) للمتصل اللي عنده الـ lock بالفعل.

- **Return:** قيمة الـ register (16-bit في int)، أو negative errno لو فشل
- **مهم جدًا:** ممنوع استدعاؤها من interrupt context

---

#### `phy_write` / `__phy_write`

```c
static inline int phy_write(struct phy_device *phydev, u32 regnum, u16 val);
static inline int __phy_write(struct phy_device *phydev, u32 regnum, u16 val);
```

نفس المنطق بس للكتابة. `val` 16-bit فقط.

---

#### `phy_modify` / `__phy_modify`

```c
int phy_modify(struct phy_device *phydev, u32 regnum, u16 mask, u16 set);
int __phy_modify(struct phy_device *phydev, u32 regnum, u16 mask, u16 set);
```

بتعمل:
```c
new_val = (old_val & ~mask) | set
```
ذري بمعنى إن الـ read-modify-write كله تحت الـ `mdio_lock`. لو الـ mask و set متداخلين، الـ `set` هو اللي بيغلب.

- **Return:** 0 لو نجح، negative errno لو فشل
- **Key detail:** مش هو اللي بيحدد هل في تغيير فعلي — استخدم `phy_modify_changed` لده

---

#### `phy_modify_changed` / `__phy_modify_changed`

```c
int phy_modify_changed(struct phy_device *phydev, u32 regnum, u16 mask, u16 set);
```

زي `phy_modify` بالظبط بس بيرجع `1` لو القيمة اتغيرت فعلًا، `0` لو ملاقتش فرق، أو negative errno.

---

#### `phy_set_bits` / `phy_clear_bits`

```c
static inline int phy_set_bits(struct phy_device *phydev, u32 regnum, u16 val);
static inline int phy_clear_bits(struct phy_device *phydev, u32 regnum, u16 val);
```

shorthand لحالات set/clear فقط. الأول بيعمل `phy_modify(phydev, regnum, 0, val)`، والثاني `phy_modify(phydev, regnum, val, 0)`.

---

#### `phy_read_poll_timeout` (macro)

```c
#define phy_read_poll_timeout(phydev, regnum, val, cond, \
                              sleep_us, timeout_us, sleep_before_read)
```

بتعمل poll على register حتى `cond` تكون true أو timeout. بتستخدم `read_poll_timeout` من `<linux/iopoll.h>`. لو حصل read error، بيرجع error مش timeout.

---

### Group 6: Register I/O — MMD (Clause 45)

**Clause 45 / MMD (MDIO Manageable Device)** بيستخدم 5-bit device address (devad) + 16-bit register address. بيتيح الوصول لـ 32 device (PMA, PCS, AN, etc.).

لـ Clause 22 PHYs، الـ kernel بيستخدم indirect access عبر registers 13/14.

---

#### `phy_read_mmd` / `__phy_read_mmd`

```c
int phy_read_mmd(struct phy_device *phydev, int devad, u32 regnum);
int __phy_read_mmd(struct phy_device *phydev, int devad, u32 regnum);
```

بتقرأ register من MMD. لو الـ driver عنده `read_mmd` callback، بيستخدمه. غير كده:
- لـ C45 PHY: بيعمل direct C45 read
- لـ C22 PHY: بيستخدم indirect access عبر reg 13/14

- **Parameters:**
  - `devad`: رقم الـ MMD device (مثلًا `MDIO_MMD_PCS = 3`)
  - `regnum`: رقم الـ register
- **Return:** 16-bit value أو negative errno

---

#### `phy_write_mmd` / `__phy_write_mmd`

```c
int phy_write_mmd(struct phy_device *phydev, int devad, u32 regnum, u16 val);
int __phy_write_mmd(struct phy_device *phydev, int devad, u32 regnum, u16 val);
```

نفس المنطق بس كتابة. الـ locked version بتأخذ `mdio_lock`.

---

#### `phy_modify_mmd` / `phy_modify_mmd_changed`

```c
int phy_modify_mmd(struct phy_device *phydev, int devad, u32 regnum,
                   u16 mask, u16 set);
int phy_modify_mmd_changed(struct phy_device *phydev, int devad, u32 regnum,
                           u16 mask, u16 set);
```

read-modify-write ذري لـ MMD registers. الـ `_changed` variant بيرجع 1 لو تغيّر.

---

#### `phy_set_bits_mmd` / `phy_clear_bits_mmd`

```c
static inline int phy_set_bits_mmd(struct phy_device *phydev, int devad,
                                   u32 regnum, u16 val);
static inline int phy_clear_bits_mmd(struct phy_device *phydev, int devad,
                                     u32 regnum, u16 val);
```

shorthand لـ MMD bit manipulation.

---

#### `phy_read_mmd_poll_timeout` (macro)

```c
#define phy_read_mmd_poll_timeout(phydev, devaddr, regnum, val, cond, \
                                  sleep_us, timeout_us, sleep_before_read)
```

نفس `phy_read_poll_timeout` بس للـ MMD registers. مفيد جدًا لانتظار completion bits زي `AN_COMPLETE`.

---

### Group 7: Paged Register Access

بعض الـ PHYs (مثلًا Realtek RTL8211) بتستخدم **register pages** لتوسيع address space. الـ phylib بيوفر helper functions آمنة لدي.

---

#### `phy_save_page`

```c
int phy_save_page(struct phy_device *phydev);
```

بتأخذ `mdio_lock` وبتقرأ وتحفظ الـ current page عبر `drv->read_page`. بترجع الـ page number أو negative errno.

---

#### `phy_select_page`

```c
int phy_select_page(struct phy_device *phydev, int page);
```

بتتحقق إن `mdio_lock` محجوز وبتكتب الـ page الجديدة عبر `drv->write_page`. بترجع الـ old page number.

---

#### `phy_restore_page`

```c
int phy_restore_page(struct phy_device *phydev, int oldpage, int ret);
```

بتعيد الـ page الأصلية وبتطلق `mdio_lock`. `ret` هو نتيجة الـ operation اللي اتعملت في الـ middle — بيرجعه مباشرة (أو error لو الـ restore فشل). دائمًا يُستدعى لإغلاق `phy_save_page`.

**Pattern النموذجي:**
```c
int oldpage = phy_save_page(phydev);
if (oldpage < 0)
    return oldpage;

/* read/write registers on target page */
ret = __phy_write(phydev, REG, val);

return phy_restore_page(phydev, oldpage, ret);
```

---

#### `phy_read_paged` / `phy_write_paged`

```c
int phy_read_paged(struct phy_device *phydev, int page, u32 regnum);
int phy_write_paged(struct phy_device *phydev, int page, u32 regnum, u16 val);
```

بيجمعوا save_page + select_page + read/write + restore_page في call واحدة. آمنة ومتاحة من process context.

---

#### `phy_modify_paged` / `phy_modify_paged_changed`

```c
int phy_modify_paged(struct phy_device *phydev, int page, u32 regnum,
                     u16 mask, u16 set);
```

read-modify-write على page معينة. الـ `_changed` variant بيرجع 1 لو تغيّر.

---

### Group 8: Autonegotiation

---

#### `phy_config_aneg`

```c
int phy_config_aneg(struct phy_device *phydev);
```

بتنادي `drv->config_aneg` لو موجود، غير كده `genphy_config_aneg`. بتضبط الـ advertisement حسب `phydev->advertising` وبتبدأ أو تمسح الـ autoneg حسب `phydev->autoneg`.

---

#### `phy_start_aneg` / `_phy_start_aneg`

```c
int phy_start_aneg(struct phy_device *phydev);
int _phy_start_aneg(struct phy_device *phydev);
```

الـ locked version بتأخذ `phydev->lock`. الـ unlocked للـ callers اللي عندهم الـ lock بالفعل (مثلًا من state machine). بتنادي `phy_config_aneg` وبتعمل trigger للـ machine.

---

#### `phy_aneg_done`

```c
int phy_aneg_done(struct phy_device *phydev);
```

بتنادي `drv->aneg_done` لو موجود، غير كده `genphy_aneg_done` اللي بتقرأ BMSR_ANEGCOMPLETE bit.

- **Return:** 1 لو اكتمل، 0 لو لسه، negative errno لو فشل

---

#### `genphy_config_aneg` / `__genphy_config_aneg`

```c
static inline int genphy_config_aneg(struct phy_device *phydev);
int __genphy_config_aneg(struct phy_device *phydev, bool changed);
```

**Generic Clause 22 autoneg config.** بتكتب الـ ANAR/1000BASET_CTRL registers بناءً على `phydev->advertising` وبتبدأ الـ autoneg لو `phydev->autoneg == AUTONEG_ENABLE`.

**Pseudocode:**
```
__genphy_config_aneg(phydev, changed):
    if autoneg disabled:
        genphy_setup_forced(phydev)
    else:
        adv = ethtool_adv_to_mii_adv_t(advertising)
        write ANAR register
        if 1G capable: write 1000BASET_CTRL
        if changed or restart needed:
            genphy_restart_aneg(phydev)
```

---

#### `genphy_read_status`

```c
int genphy_read_status(struct phy_device *phydev);
```

بتقرأ link state، speed، duplex بعد autoneg:
1. بتقرأ BMSR لمعرفة link/autoneg_complete
2. لو autoneg_complete: بتقرأ ANLPAR للـ LPA
3. بتحسب speed/duplex من LPA
4. بتحدث `phydev->speed`، `phydev->duplex`، `phydev->link`

---

#### `genphy_update_link`

```c
int genphy_update_link(struct phy_device *phydev);
```

بتقرأ BMSR مرتين (BMSR sticky bits) وبتحدث `phydev->link`. المرة الأولى للـ latch clear، الثانية للقراءة الفعلية.

---

#### `genphy_c45_config_aneg` / `genphy_c45_read_status`

الـ Clause 45 equivalent للـ C22 genphy functions. بيتعاملوا مع PCS/AN MMDs (device 3, 7) بدلًا من C22 registers.

---

### Group 9: Power Management

---

#### `phy_suspend`

```c
int phy_suspend(struct phy_device *phydev);
```

بتوقف الـ PHY وبتضع في low-power mode. بتنادي `drv->suspend` أو `genphy_suspend`. بتضبط `phydev->suspended = true`.

- **Key detail:** لو الـ WoL enabled (`wol_enabled`), الـ suspend بيتخطى عشان الـ PHY لازم يفضل صاحي

---

#### `phy_resume`

```c
int phy_resume(struct phy_device *phydev);
```

بتصحّي الـ PHY وبتعيد تهيئته. بتنادي `drv->resume` أو `genphy_resume`. بتعمل `phy_init_hw` لإعادة configuration.

---

#### `__phy_resume`

```c
int __phy_resume(struct phy_device *phydev);
```

unlocked version لـ `phy_resume`، للـ callers اللي عندهم `mdio_lock` بالفعل.

---

#### `genphy_suspend` / `genphy_resume`

```c
int genphy_suspend(struct phy_device *phydev);
int genphy_resume(struct phy_device *phydev);
```

بيعملوا set/clear لـ `BMCR_PDOWN` bit في الـ BMCR register.

---

### Group 10: Interface Mode Helpers

---

#### `phy_modes`

```c
static inline const char *phy_modes(phy_interface_t interface);
```

بتحوّل enum `phy_interface_t` لـ string مقروءة (مثلًا `PHY_INTERFACE_MODE_RGMII` → `"rgmii"`). مستخدمة في logging والـ device tree matching.

---

#### `rgmii_clock`

```c
static inline long rgmii_clock(int speed);
```

بتحوّل link speed لـ RGMII clock rate بالـ Hz:
- 10 Mbps → 2.5 MHz
- 100 Mbps → 25 MHz
- 1000 Mbps → 125 MHz

بتُستخدم لضبط clock provider عند link state change.

---

#### `phy_interface_mode_is_rgmii`

```c
static inline bool phy_interface_mode_is_rgmii(phy_interface_t mode);
```

بتتحقق إن الـ mode من أي variant من RGMII (plain, ID, RXID, TXID).

---

#### `phy_interface_mode_is_8023z`

```c
static inline bool phy_interface_mode_is_8023z(phy_interface_t mode);
```

بتتحقق إن الـ mode بيستخدم 802.3z negotiation word (1000BASE-X أو 2500BASE-X).

---

#### `phy_interface_set_rgmii`

```c
static inline void phy_interface_set_rgmii(unsigned long *intf);
```

بتضبط كل RGMII variants في bitmap واحدة دفعة واحدة. مفيدة لـ phylink advertised_interfaces.

---

#### `phy_fix_phy_mode_for_mac_delays`

```c
phy_interface_t phy_fix_phy_mode_for_mac_delays(phy_interface_t interface,
                                                 bool mac_txid, bool mac_rxid);
```

لو الـ MAC بيضيف TX/RX delay، بتعدّل الـ interface mode عشان الـ PHY ما يضيفش delay تاني. مثلًا: لو `RGMII` + mac_txid + mac_rxid → ترجع `RGMII_ID`.

---

### Group 11: PHY ID Comparison

---

#### `phy_id_compare`

```c
static inline bool phy_id_compare(u32 id1, u32 id2, u32 mask);
```

بتتحقق من تطابق PHY IDs مع mask:
```c
return !((id1 ^ id2) & mask);
```

الـ mask بيحدد الـ bits الهامة في المقارنة. يُستخدم في driver matching.

---

#### `phy_id_compare_vendor` / `phy_id_compare_model`

```c
static inline bool phy_id_compare_vendor(u32 id, u32 vendor_mask);
static inline bool phy_id_compare_model(u32 id, u32 model_mask);
```

Wrappers بـ masks محددة مسبقًا:
- `PHY_ID_MATCH_VENDOR_MASK` = bits [31:10] (OUI)
- `PHY_ID_MATCH_MODEL_MASK` = bits [31:4] (OUI + model)

---

#### `phydev_id_compare`

```c
static inline bool phydev_id_compare(struct phy_device *phydev, u32 id);
```

بتقارن `id` مع `phydev->phy_id` باستخدام mask من الـ driver المربوط حاليًا. أبسط من استدعاء `phy_id_compare` مباشرة.

---

### Group 12: EEE (Energy-Efficient Ethernet)

---

#### `phy_support_eee`

```c
void phy_support_eee(struct phy_device *phydev);
```

بتفعّل EEE advertisement. تُستدعى من `config_init` أو `probe` لتفعيل EEE على الـ PHY. بتضبط `advertising_eee` بكل الـ EEE modes المدعومة.

---

#### `phy_disable_eee`

```c
void phy_disable_eee(struct phy_device *phydev);
```

بتعطّل EEE تمامًا (بتمسح الـ `advertising_eee`). مفيدة لو الـ MAC مش بيدعم EEE.

---

#### `phy_disable_eee_mode`

```c
static inline void phy_disable_eee_mode(struct phy_device *phydev, u32 link_mode);
```

بتعطّل EEE لـ link mode واحد بالتحديد (مثلًا 100BASE-TX EEE فقط). بتضيفه لـ `eee_disabled_modes`.

- **Key detail:** يجب استدعاؤها قبل `phy_start` — لو الـ PHY شغّال يظهر WARN_ON

---

#### `phy_init_eee`

```c
int phy_init_eee(struct phy_device *phydev, bool clk_stop_enable);
```

بتهيّئ EEE بعد link up: بتتحقق من الـ LPI support، وبتفعّل TX LPI. يُستدعى من link_up callback في phylink أو adjust_link.

---

#### `genphy_c45_eee_is_active`

```c
int genphy_c45_eee_is_active(struct phy_device *phydev, unsigned long *lp);
```

بتتحقق هل EEE فعّال بالفعل على الـ link (بعد negotiation). بتقرأ PCS EEE capability وبتقارن مع link partner.

---

### Group 13: Ethtool Integration

---

#### `phy_ethtool_ksettings_get` / `phy_ethtool_ksettings_set`

```c
void phy_ethtool_ksettings_get(struct phy_device *phydev,
                               struct ethtool_link_ksettings *cmd);
int phy_ethtool_ksettings_set(struct phy_device *phydev,
                              const struct ethtool_link_ksettings *cmd);
```

بيملوا/بيطبقوا `ethtool_link_ksettings` من/على الـ `phy_device`. MAC drivers بيستدعوها من `ndo_get_link_ksettings` و `ndo_set_link_ksettings`.

**الـ `set`** بتتحقق من الـ advertised modes صحيحة، وبتنادي `phy_start_aneg` لو اتغير شيء.

---

#### `phy_ethtool_nway_reset`

```c
int phy_ethtool_nway_reset(struct net_device *ndev);
```

يُستدعى من `ndo_set_settings` أو من ethtool مباشرة لإعادة تشغيل autoneg. بيأخذ `rtnl_lock`.

---

#### `phy_mii_ioctl`

```c
int phy_mii_ioctl(struct phy_device *phydev, struct ifreq *ifr, int cmd);
```

بيتعامل مع `SIOCGMIIPHY`، `SIOCGMIIREG`، `SIOCSMIIREG` ioctls. بيسمح لـ userspace بقراءة وكتابة MDIO registers مباشرة (مثلًا `mii-tool`).

---

#### `phy_ethtool_get_plca_cfg` / `phy_ethtool_set_plca_cfg`

```c
int phy_ethtool_get_plca_cfg(struct phy_device *phydev,
                             struct phy_plca_cfg *plca_cfg);
int phy_ethtool_set_plca_cfg(struct phy_device *phydev,
                             const struct phy_plca_cfg *plca_cfg,
                             struct netlink_ext_ack *extack);
```

قراءة/كتابة إعدادات الـ PLCA (Physical Layer Collision Avoidance) للـ 10BASE-T1S networks. بيستخدم `drv->get_plca_cfg` و `drv->set_plca_cfg`.

---

### Group 14: Logging & Debug Helpers

```c
#define phydev_err(_phydev, format, args...)
#define phydev_err_probe(_phydev, err, format, args...)
#define phydev_info(_phydev, format, args...)
#define phydev_warn(_phydev, format, args...)
#define phydev_dbg(_phydev, format, args...)
```

Wrappers على `dev_err/info/warn/dbg` بتستخدم `phydev->mdio.dev` كـ device context. بيضمنوا إن كل رسائل الـ log بيظهر فيها اسم الـ PHY device صح.

---

```c
void phy_attached_print(struct phy_device *phydev, const char *fmt, ...);
void phy_attached_info(struct phy_device *phydev);
char *phy_attached_info_irq(struct phy_device *phydev) __malloc;
```

**الـ `phy_attached_info`** بتطبع رسالة معلومات standard لما PHY يتربط: اسمه، عنوانه، IRQ mode. الـ MAC drivers بيستدعوها من `adjust_link` أو من `probe` بعد `phy_connect`.

---

### Group 15: Interrupt Handling

---

#### `phy_request_interrupt`

```c
void phy_request_interrupt(struct phy_device *phydev);
```

بتعمل `request_threaded_irq` للـ PHY irq. الـ thread handler بيقرأ status وبيعمل trigger للـ state machine. يُستدعى من `phy_start` لو الـ PHY مش في polling mode.

---

#### `phy_free_interrupt`

```c
void phy_free_interrupt(struct phy_device *phydev);
```

بتحرر الـ IRQ وبتعطّل interrupts في الـ PHY. يُستدعى من `phy_stop`.

---

#### `phy_disable_interrupts`

```c
int phy_disable_interrupts(struct phy_device *phydev);
```

بتنادي `drv->config_intr` لتعطيل interrupts في الـ PHY hardware. بترجع 0 أو negative errno.

---

### Group 16: Speed, Duplex, Pause Helpers

---

#### `phy_speed_to_str` / `phy_duplex_to_str`

```c
const char *phy_speed_to_str(int speed);
const char *phy_duplex_to_str(unsigned int duplex);
```

بيحولوا speed/duplex int لـ human-readable string (مثلًا `1000` → `"1000Mbps"`).

---

#### `phy_set_max_speed`

```c
void phy_set_max_speed(struct phy_device *phydev, u32 max_speed);
```

بتمسح كل الـ link modes اللي فوق `max_speed` من الـ `supported` bitmap. مفيدة لو الـ MAC مش بيدعم غير 100M مثلًا.

---

#### `phy_remove_link_mode`

```c
void phy_remove_link_mode(struct phy_device *phydev, u32 link_mode);
```

بتمسح link mode واحد من `supported` و `advertising`. مفيدة لـ quirks.

---

#### `phy_support_sym_pause` / `phy_support_asym_pause`

```c
void phy_support_sym_pause(struct phy_device *phydev);
void phy_support_asym_pause(struct phy_device *phydev);
```

بتضيفوا Pause bits لـ `advertising`. يُستدعوا من `config_init` لو الـ MAC بيدعم flow control.

---

#### `phy_get_pause`

```c
void phy_get_pause(struct phy_device *phydev, bool *tx_pause, bool *rx_pause);
```

بتحسب هل TX/RX pause مفعّل بناءً على الـ negotiated link modes. يُستدعى من MAC بعد link up.

---

#### `phy_resolve_pause`

```c
void phy_resolve_pause(unsigned long *local_adv, unsigned long *partner_adv,
                       bool *tx_pause, bool *rx_pause);
```

بتحسب TX/RX pause status من local وpartner advertisements بدون الحاجة لـ `phy_device`. مفيدة لـ phylink.

---

#### `phy_speed_down` / `phy_speed_up`

```c
int phy_speed_down(struct phy_device *phydev, bool sync);
int phy_speed_up(struct phy_device *phydev);
```

**الـ `phy_speed_down`** بتحفظ الـ current advertising وبتحذف الـ speeds الأعلى ثم بتعيد بدء autoneg. مفيدة لـ power saving.

**الـ `phy_speed_up`** بتستعيد الـ advertising المحفوظة وبتعيد تشغيل autoneg.

---

### Group 17: Fixups

---

#### `phy_register_fixup_for_id`

```c
int phy_register_fixup_for_id(const char *bus_id,
                              int (*run)(struct phy_device *));
```

بتسجّل fixup function لـ PHY محدد بالاسم. الـ fixup بيتنفذ في `phy_init_hw` بعد الـ `config_init`. مفيدة لـ board-specific quirks.

---

#### `phy_register_fixup_for_uid`

```c
int phy_register_fixup_for_uid(u32 phy_uid, u32 phy_uid_mask,
                               int (*run)(struct phy_device *));
```

زي السابقة بس بتطابق بالـ UID بدلًا من الاسم. أكثر portable.

---

### Group 18: PHY Module Registration

---

#### `phy_drivers_register` / `phy_drivers_unregister`

```c
int phy_drivers_register(struct phy_driver *new_driver, int n,
                         struct module *owner);
void phy_drivers_unregister(struct phy_driver *drv, int n);
```

بيسجّلوا/بيلغوا تسجيل array من `phy_driver` structures. عادةً ما بيُستدعوا مباشرة — بدلًا من كده استخدم الـ macro.

---

#### `module_phy_driver` / `phy_module_driver`

```c
#define module_phy_driver(__phy_drivers)
#define phy_module_driver(__phy_drivers, __count)
```

الـ macros دي بتولّد `module_init` و `module_exit` أوتوماتيك. مثال:

```c
static struct phy_driver my_phy_drivers[] = {
    {
        PHY_ID_MATCH_MODEL(0x00221640),
        .name = "My PHY",
        .features = PHY_GBIT_FEATURES,
        .config_init = my_phy_config_init,
    }
};
module_phy_driver(my_phy_drivers);
```

الـ macro بيكافئ:
```c
static int __init my_init(void) {
    return phy_drivers_register(my_phy_drivers,
                                ARRAY_SIZE(my_phy_drivers),
                                THIS_MODULE);
}
module_init(my_init);
```

---

### Group 19: Timestamping (PTP)

---

#### `phy_has_hwtstamp` / `phy_hwtstamp`

```c
static inline bool phy_has_hwtstamp(struct phy_device *phydev);
static inline int phy_hwtstamp(struct phy_device *phydev,
                               struct kernel_hwtstamp_config *cfg,
                               struct netlink_ext_ack *extack);
```

**الأولى** بتتحقق إن الـ PHY عنده hardware timestamping capability. **الثانية** بتستدعي `mii_ts->hwtstamp_set` لضبط الـ timestamp config.

---

#### `phy_rxtstamp` / `phy_txtstamp`

```c
static inline bool phy_rxtstamp(struct phy_device *phydev, struct sk_buff *skb, int type);
static inline void phy_txtstamp(struct phy_device *phydev, struct sk_buff *skb, int type);
```

بيمرروا الـ skb للـ PHY timestamper لإضافة RX/TX timestamps. يُستدعوا من الـ MAC driver في RX/TX path.

---

#### `phy_is_default_hwtstamp`

```c
static inline bool phy_is_default_hwtstamp(struct phy_device *phydev);
```

بتتحقق إن الـ PHY هو المصدر الافتراضي للـ timestamps (بدلًا من الـ MAC). بيرجع true لو `has_hwtstamp && default_timestamp`.

---

### Group 20: Cable Test

---

#### `phy_start_cable_test`

```c
int phy_start_cable_test(struct phy_device *phydev,
                         struct netlink_ext_ack *extack);
```

بتبدأ cable test (TDR للـ open/short detection). بتنتقل بالـ PHY لـ `PHY_CABLETEST` state وبتنادي `drv->cable_test_start`. النتائج بتتقرأ بعدين عبر Netlink.

---

#### `phy_start_cable_test_tdr`

```c
int phy_start_cable_test_tdr(struct phy_device *phydev,
                             struct netlink_ext_ack *extack,
                             const struct phy_tdr_config *config);
```

نفس السابقة بس لـ TDR raw cable test مع configuration محددة (first/last/step distances).

---

### Group 21: Misc Inline Helpers

---

#### `phy_is_started`

```c
static inline bool phy_is_started(struct phy_device *phydev);
```

بترجع `true` لو `phydev->state >= PHY_UP`. بتُستخدم للتحقق سريعًا إن الـ PHY شغّال.

---

#### `phy_polling_mode`

```c
static inline bool phy_polling_mode(struct phy_device *phydev);
```

بترجع `true` لو الـ PHY في polling mode (مش interrupt-driven). True لو:
- `phydev->irq == PHY_POLL`, أو
- الـ PHY في `PHY_CABLETEST` مع `PHY_POLL_CABLE_TEST` flag، أو
- الـ driver عنده `update_stats`

---

#### `phy_interrupt_is_valid`

```c
static inline bool phy_interrupt_is_valid(struct phy_device *phydev);
```

بترجع `true` لو الـ `phydev->irq` هو IRQ number فعلي (مش `PHY_POLL` ومش `PHY_MAC_INTERRUPT`).

---

#### `phy_lock_mdio_bus` / `phy_unlock_mdio_bus`

```c
static inline void phy_lock_mdio_bus(struct phy_device *phydev);
static inline void phy_unlock_mdio_bus(struct phy_device *phydev);
```

Wrappers بسيطة على `mutex_lock/unlock(&phydev->mdio.bus->mdio_lock)`. للـ drivers اللي بتحتاج تعمل عمليات متعددة بدون lock overhead لكل operation.

---

#### `phy_get_internal_delay`

```c
s32 phy_get_internal_delay(struct phy_device *phydev, const int *delay_values,
                           int size, bool is_rx);
```

بتقرأ قيمة الـ RGMII internal delay من الـ DT (`rx-internal-delay-ps` أو `tx-internal-delay-ps`) وبترجع أقرب مدعوم value من الـ `delay_values` array. مفيدة جدًا في RGMII drivers.

---

#### `phy_can_wakeup` / `phy_may_wakeup`

```c
static inline bool phy_can_wakeup(struct phy_device *phydev);
bool phy_may_wakeup(struct phy_device *phydev);
```

الأولى بتتحقق إن الـ PHY device registered كـ wakeup source. الثانية بتتحقق إن الـ wakeup فعّال فعلًا (enabled). يُستخدمان لاتخاذ قرار الـ suspend.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs — المسارات والقراءة

**الـ MDIO/PHY subsystem** بيكتب في debugfs تحت `/sys/kernel/debug/`:

```bash
# شوف كل الـ mdio buses المسجّلة
ls /sys/kernel/debug/mdio-bus/

# اقرأ إحصائيات الـ MDIO bus (transfers/errors/reads/writes)
# كل bus بيبقى اسمه زي "0000:00:1f.6:00" أو "stmmac-0"
cat /sys/kernel/debug/mdio-bus/<bus_id>/stats

# مثال: bus اسمه "fc030000.ethernet-1"
cat /sys/kernel/debug/mdio-bus/fc030000.ethernet-1/0/stats
```

**تفسير الـ output:**

```
transfers: 1423      ← إجمالي العمليات (reads + writes)
errors:    3         ← عدد العمليات اللي فشلت
writes:    98
reads:     1325
```

**الـ `mdio_bus_stats` struct** (من `phy.h`) بيحتوي على:
- `transfers`, `errors`, `writes`, `reads` — كلها `u64_stats_t`
- `syncp` — للـ synchronization بين contexts

```bash
# cable test results (بتظهر لو الـ PHY بيدعم cable diagnostics)
ls /sys/kernel/debug/netdev/
```

---

#### 2. sysfs — المسارات المفيدة

```bash
# كل PHY device ظاهر في sysfs تحت mdio_bus
ls /sys/bus/mdio_bus/devices/

# PHY device path نموذجي: <bus_id>:<phy_addr>
# مثلاً: stmmac-0:00 أو fc030000.ethernet-1:01
PHY_DEV="stmmac-0:00"

# اقرأ الـ phy_id (OUI + model + revision)
cat /sys/bus/mdio_bus/devices/$PHY_DEV/phy_id

# اقرأ الـ interface type المستخدم (rgmii, sgmii, ...)
cat /sys/bus/mdio_bus/devices/$PHY_DEV/phy_interface

# لو الـ net device معروف (eth0 مثلاً):
ethtool eth0           # speed/duplex/link
ethtool -S eth0        # statistics
ethtool -i eth0        # driver info + PHY ID

# PHY state machine state (من kernel 5.x+)
cat /sys/class/net/eth0/carrier
cat /sys/class/net/eth0/operstate

# MDIO bus attributes
ls /sys/bus/mdio_bus/
cat /sys/bus/mdio_bus/devices/$PHY_DEV/modalias
```

---

#### 3. ftrace — الـ Tracepoints والأحداث

**تفعيل tracing لـ MDIO/PHY events:**

```bash
# شوف الـ events المتاحة للـ MDIO
grep -r "mdio\|phylib\|phy_" /sys/kernel/debug/tracing/available_events

# trace كل الـ MDIO read/write operations
cd /sys/kernel/debug/tracing

echo 1 > tracing_on
echo "mdio:*" > set_event        # كل أحداث MDIO subsystem

# أو بشكل منفرد:
echo "mdio:mdio_access" >> set_event

# ابدأ الـ trace ثم اقرأ النتيجة
cat trace | head -50

# filter على bus معين
echo 'bus_id == "stmmac-0"' > events/mdio/mdio_access/filter

# trace الـ net device link state changes
echo "net:net_dev_queue" >> set_event
echo "net:napi_poll" >> set_event

# function tracer لـ phylib functions
echo function > current_tracer
echo "phy_*" > set_ftrace_filter
echo "genphy_*" >> set_ftrace_filter
cat trace

# إيقاف
echo 0 > tracing_on
echo nop > current_tracer
echo > set_ftrace_filter
```

**مثال على output مفيد:**

```
          eth0-123 [001] .... phy_state_machine: phydev=stmmac-0:01 state=RUNNING->NOLINK
     kworker/0:2  [000] .... mdio_access: bus=stmmac-0 addr=0x01 regnum=0x01 val=0x796d is_write=0
```

---

#### 4. printk / Dynamic Debug

**تفعيل dynamic debug للـ phylib:**

```bash
# تفعيل كل debug messages في ملفات phylib
echo "file drivers/net/phy/phy.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/net/phy/phy_device.c +p" >> /sys/kernel/debug/dynamic_debug/control
echo "file drivers/net/phy/mdio_bus.c +p" >> /sys/kernel/debug/dynamic_debug/control
echo "file drivers/net/phy/phy-core.c +p" >> /sys/kernel/debug/dynamic_debug/control

# تفعيل للـ module كله
echo "module phylib +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل مؤقت عبر kernel cmdline (في bootargs):
# dyndbg="module phylib +p"

# متابعة الـ output
dmesg -w | grep -i "phy\|mdio"
journalctl -f -k | grep -i phy

# إيقاف
echo "module phylib -p" > /sys/kernel/debug/dynamic_debug/control
```

**الـ `phydev_dbg()` macro** (من `phy.h`) بيُستخدم في الـ drivers بدلاً من `pr_debug()` مباشرة، وبيحتاج dynamic debug مفعّل.

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الغرض |
|---|---|
| `CONFIG_PHYLIB` | الـ PHY framework الأساسي — لازم يكون مفعّل |
| `CONFIG_DEBUG_MDIO` | يُضيف extra validation لعمليات الـ MDIO bus |
| `CONFIG_MDIO_DEVICE` | MDIO device infrastructure |
| `CONFIG_NETWORK_PHY_TIMESTAMPING` | debugging الـ PTP timestamping عبر PHY |
| `CONFIG_LED_TRIGGER_PHY` | تفعيل LED triggers للـ PHY (link/speed) |
| `CONFIG_PHY_PACKAGE` | shared state بين PHYs في نفس الـ package |
| `CONFIG_NET_DSA` | لو الـ PHY بيشتغل مع distributed switch |
| `CONFIG_TRACEPOINTS` | لتفعيل ftrace events |
| `CONFIG_DYNAMIC_DEBUG` | لتفعيل `phydev_dbg()` dynamically |
| `CONFIG_LOCKDEP` | كشف deadlocks في `mdio_lock` و`phydev->lock` |
| `CONFIG_KCSAN` | كشف data races في الـ MDIO bus access |
| `CONFIG_PROVE_LOCKING` | التحقق من صحة locking order في PHY state machine |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "PHYLIB|MDIO|PHY_PACK|DEBUG_MDIO"
```

---

#### 6. أدوات subsystem-specific

```bash
# ethtool — الأداة الأساسية للـ PHY debugging
ethtool eth0                    # link speed/duplex/autoneg state
ethtool -a eth0                 # pause parameters
ethtool -c eth0                 # coalesce settings
ethtool -k eth0                 # offload features
ethtool -S eth0                 # driver statistics (يشمل PHY stats)
ethtool --phy-statistics eth0   # PHY-specific statistics (kernel 4.10+)
ethtool -t eth0 offline         # self-test
ethtool --cable-test eth0       # cable test (TDR)
ethtool --cable-test-tdr eth0 first 0 last 10000 step 100  # raw TDR
ethtool -P eth0                 # permanent MAC address
ethtool --get-plca-cfg eth0     # PLCA configuration (10BASE-T1S)
ethtool --get-plca-status eth0  # PLCA status

# mii-tool (قديم لكن مفيد للـ Clause 22)
mii-tool -v eth0

# phytool — قراءة/كتابة PHY registers مباشرة
# (محتاج تثبيته: apt install phytool)
phytool read  eth0/1/0    # MII_BMCR (Basic Mode Control Register)
phytool read  eth0/1/1    # MII_BMSR (Basic Mode Status Register)
phytool write eth0/1/0 0x3100  # force 100M full-duplex

# devlink (للـ switchdev/DSA scenarios)
devlink dev show
devlink port show
```

---

#### 7. جدول رسائل الأخطاء الشائعة

| رسالة الـ kernel log | المعنى | الحل |
|---|---|---|
| `PHY <addr> not found` | الـ PHY مش موجود على الـ MDIO bus | افحص الـ MDIO wiring، الـ address، والـ phy_mask |
| `Unable to find PHY bus/device` | الـ bus_id في DT غلط أو الـ bus مش متسجل بعد | راجع `&mdio0` phandle في DT، تأكد من الـ probe order |
| `MDIO bus busy` | الـ mdio_lock محجوز لفترة طويلة | كشف deadlock عبر `CONFIG_LOCKDEP`، افحص `mdio_bus.mdio_lock` |
| `phy_id 0xffffffff` | الـ MDIO bus بيقرأ 0xFFFF في كل registers | مشكلة hardware: power، pull-ups، أو reset |
| `phy_id 0x00000000` | كمان مشكلة hardware: MDIO مش بيرد | نفس الأسباب السابقة |
| `Link is Down` بشكل متكرر | الـ PHY state machine بيتنقل PHY_RUNNING->PHY_NOLINK | افحص الكابل، الـ link partner، والـ autoneg settings |
| `genphy_read_status: no attached_dev` | الـ PHY مش attached لـ net_device | تأكد من `phy_connect()` أو `phy_attach_direct()` |
| `phy c45 read failed` | MDIO Clause 45 transaction فشلت | افحص دعم الـ C45 في الـ MAC controller |
| `PHY Error` state | الـ state machine وصل `PHY_ERROR` | `phy_error()` اتست، افحص `dmesg` للـ root cause |
| `irq <n>: nobody cared` | الـ PHY interrupt مش بيتعالج | افحص `phydev->irq`، وفعّل `PHY_POLL` للتشخيص |
| `autonegotiation timed out` | الـ AN مش بيكتمل | force speed/duplex، افحص الـ link partner config |
| `RGMII timing issue` | تأخير غلط في TX/RX للـ RGMII | استخدم `rgmii-id`/`rgmii-rxid`/`rgmii-txid` في DT |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في phy_device.c — تحقق من الـ lock */
int phy_start(struct phy_device *phydev)
{
    /* لو حد بيستدعي phy_start بدون mutex */
    WARN_ON(!mutex_is_locked(&phydev->lock));
    ...
}

/* في driver probe — تحقق من الـ phy_id */
static int my_phy_probe(struct phy_device *phydev)
{
    WARN_ON(phydev->phy_id == 0xffffffff);
    WARN_ON(phydev->phy_id == 0x00000000);
    ...
}

/* تحقق من الـ state machine transitions */
static void my_state_check(struct phy_device *phydev)
{
    /* state shouldn't be PHY_ERROR in normal operation */
    if (WARN_ON(phydev->state == PHY_ERROR)) {
        dump_stack();
        phy_print_status(phydev);  /* يطبع speed/duplex/link */
    }
}

/* في MDIO read — تحقق من الـ bus lock */
int __phy_read(struct phy_device *phydev, u32 regnum)
{
    /* لازم الـ mdio_lock ممسوك قبل ما تستدعي __phy_read */
    WARN_ON_ONCE(!mutex_is_locked(&phydev->mdio.bus->mdio_lock));
    return __mdiobus_read(phydev->mdio.bus, phydev->mdio.addr, regnum);
}

/* تحقق من الـ interface consistency */
WARN_ON(phydev->interface == PHY_INTERFACE_MODE_NA &&
        phydev->attached_dev != NULL);
```

**نقاط تستحق `dump_stack()` في debugging:**
- عند الدخول لـ `PHY_ERROR` state
- عند فشل `mdiobus_read()` أكثر من 3 مرات متتالية
- عند تغيير الـ interface type بعد ما الـ PHY اتعمله `phy_start()`
- عند استدعاء `phy_stop()` من context غير متوقع

---

### Hardware Level

#### 1. التحقق أن الـ Hardware State يطابق الـ Kernel State

```bash
# 1. اقرأ الـ PHY Basic Mode Status Register (MII reg 1)
#    bit 2 = Link Status، bit 5 = Autoneg Complete
phytool read eth0/1/1   # BMSR

# 2. قارن مع ما يقوله الـ kernel
ethtool eth0 | grep -E "Link|Speed|Duplex"

# 3. اقرأ الـ PHY ID registers (reg 2 و 3)
phytool read eth0/1/2   # PHY ID high (OUI bits 3-18)
phytool read eth0/1/3   # PHY ID low  (OUI bits 19-24 + model + revision)
# OUI = (reg2 << 6) | (reg3 >> 10)

# 4. تحقق من الـ autoneg advertisement (reg 4)
phytool read eth0/1/4   # ADVERTISE register
# bit 8 = 100BaseTx-Full، bit 7 = 100BaseTx-Half، إلخ

# 5. اقرأ Link Partner Ability (reg 5)
phytool read eth0/1/5   # LPA register

# 6. للـ Gigabit: اقرأ 1000BASE-T Control (reg 9)
phytool read eth0/1/9   # CTRL1000

# 7. للـ Clause 45 PHY: اقرأ PMA/PMD status
# devnum=1 (PMA/PMD), regnum=1
phytool read eth0/1/0x10001   # C45 format: devad.regnum
```

---

#### 2. Register Dump Techniques

```bash
# dump كل الـ Clause 22 registers (0-31) لـ PHY على addr 1
for reg in $(seq 0 31); do
    val=$(phytool read eth0/1/$reg 2>/dev/null)
    printf "REG[%02d] = %s\n" $reg "$val"
done

# بديل باستخدام mii-tool
mii-tool -v eth0

# استخدام /dev/mem لقراءة MDIO controller registers (للـ embedded SoCs)
# لازم تعرف العنوان الفيزيائي للـ MDIO controller
# مثال: STM32 MDIO base address
devmem2 0x40028000 w  # اقرأ MDIO control register

# io command (من package iotools)
io -4 0x40028000      # قراءة 32-bit register

# لـ Clause 45 dump عبر ethtool
# devad 1 = PMA/PMD, 3 = PCS, 4 = PHY XS, 7 = AN
ethtool --get-phy-dump eth0  # لو الـ driver بيدعمها
```

---

#### 3. نصائح Logic Analyzer / Oscilloscope

**الـ MDIO protocol تفاصيل:**

```
MDIO frame (Clause 22 Write):
  Preamble: 32 bits HIGH
  Start:    01
  Op:       01 (write) / 10 (read)
  PHY addr: 5 bits (PHYAD)
  Reg addr: 5 bits (REGAD)
  TA:       10 (write) / Z0 (read)
  Data:     16 bits

MDIO clock (MDC): typically 2.5 MHz (max)
```

**نقاط القياس الأساسية:**

| Signal | ما تبصفيه | الأعراض الشائعة |
|---|---|---|
| `MDC` | Clock regular، max 2.5 MHz | clock stretch أو missing → MAC controller issue |
| `MDIO` | Data valid على rising edge | glitches → pull-up مش كافي (عادة 1.5kΩ) |
| `RESET#` | Low-active، min 10ms pulse | reset قصير جداً → PHY مش بيكمّل init |
| `RGMII_TXC` | 125 MHz for 1G، 25 MHz for 100M | jitter عالي → clock source مشكلة |
| `RGMII_RXC` | نفس الـ TXC | phase mismatch → timing violations |
| `INT#` | Active low، open-drain | مش بينزل → pull-up مفقود أو PHY frozen |

**للـ RGMII timing:**
- الـ setup time لـ data قبل rising edge: min 1 ns
- الـ hold time بعد rising edge: min 1 ns
- لو في مشكلة: استخدم `rgmii-id` في DT لتفعيل internal delay

---

#### 4. Hardware Issues → Kernel Log Patterns

| المشكلة الـ Hardware | الـ Kernel Log | التشخيص |
|---|---|---|
| MDIO pull-up مفقود | `phy_id 0xffffffff` | قيس الـ MDIO line بالـ scope: لازم ترتفع لـ VCC |
| PHY power غير مستقر | `phy probe failed -ENODEV` بعد random time | قيس Vcc للـ PHY (عادة 1.8V أو 3.3V) بالـ oscilloscope |
| RGMII TX delay غلط | packets transmitted لكن الـ link partner مش بيستقبل | قيس TXC/TXD skew |
| RGMII RX delay غلط | `eth0: receive error` أو CRC errors | قيس RXC/RXD skew |
| الكابل damaged | `autonegotiation failed`، link up/down متكرر | `ethtool --cable-test eth0` |
| SFP module مش متوافق | `sfp: module not supported` | `ethtool -m eth0` لقراءة EEPROM |
| PHY reset stuck | `phy not found` بعد boot | قيس RESET# signal |
| MDC too fast | `mdio: read timeout` | قيس MDC frequency، خفّض في DT |

---

#### 5. Device Tree Debugging

```bash
# 1. اطبع الـ DT الفعلي اللي الـ kernel شايله
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "mdio"

# 2. تحقق من الـ phy-mode
cat /proc/device-tree/soc/ethernet@*/phy-mode

# 3. تحقق من الـ phy-handle/phy phandle
ls /proc/device-tree/soc/mdio@*/ethernet-phy@*/

# 4. تحقق من الـ reg (PHY address)
cat /proc/device-tree/soc/mdio@*/ethernet-phy@*/reg | xxd

# 5. تحقق من الـ interrupts للـ PHY
cat /proc/device-tree/soc/mdio@*/ethernet-phy@*/interrupts | xxd

# 6. فعّل DT debug أثناء boot (kernel cmdline):
# earlycon loglevel=8 of_devtree.debug=1

# 7. تحقق من الـ compatible string
cat /proc/device-tree/soc/mdio@*/ethernet-phy@*/compatible
# مثال: "ethernet-phy-ieee802.3-c22" أو "vendor,model"

# 8. لو phy-reset-gpios موجود
cat /proc/device-tree/soc/mdio@*/ethernet-phy@*/reset-gpios | xxd
```

**DT example صح لـ RGMII PHY:**

```dts
/* نموذج DT للـ PHY مع RGMII-ID و reset GPIO */
&fec {
    phy-mode = "rgmii-id";
    phy-handle = <&ethphy0>;
    status = "okay";

    mdio {
        #address-cells = <1>;
        #size-cells = <0>;

        ethphy0: ethernet-phy@1 {
            compatible = "ethernet-phy-ieee802.3-c22";
            reg = <1>;              /* PHY MDIO address */
            reset-gpios = <&gpio1 9 GPIO_ACTIVE_LOW>;
            reset-assert-us = <10000>;   /* 10ms */
            reset-deassert-us = <20000>; /* 20ms after reset */
            interrupt-parent = <&gpio1>;
            interrupts = <10 IRQ_TYPE_EDGE_FALLING>;
        };
    };
};
```

---

### Practical Commands — جاهزة للنسخ

#### مجموعة 1: تشخيص سريع

```bash
#!/bin/bash
# quick_phy_debug.sh
IFACE="${1:-eth0}"

echo "=== PHY Info ==="
ethtool "$IFACE"

echo -e "\n=== PHY Statistics ==="
ethtool --phy-statistics "$IFACE" 2>/dev/null || echo "not supported"

echo -e "\n=== Kernel Messages (last 50 PHY-related) ==="
dmesg | grep -iE "phy|mdio|stmmac|fec" | tail -50

echo -e "\n=== MDIO Bus ==="
ls /sys/bus/mdio_bus/devices/ 2>/dev/null

echo -e "\n=== Link Down Events ==="
cat /sys/class/net/"$IFACE"/link_down_events 2>/dev/null || echo "N/A"
```

#### مجموعة 2: قراءة PHY Registers الأساسية

```bash
#!/bin/bash
# read_phy_regs.sh
IFACE="${1:-eth0}"
PHY_ADDR="${2:-1}"

MII_REGS=(
    "0:BMCR(Control)"
    "1:BMSR(Status)"
    "2:PHYID1"
    "3:PHYID2"
    "4:ADVERTISE"
    "5:LPA"
    "9:CTRL1000"
    "10:STAT1000"
)

for entry in "${MII_REGS[@]}"; do
    reg="${entry%%:*}"
    name="${entry##*:}"
    val=$(phytool read "$IFACE/$PHY_ADDR/$reg" 2>/dev/null)
    printf "REG[%02d] %-20s = %s\n" "$reg" "$name" "$val"
done
```

**مثال على output وتفسيره:**

```
REG[00] BMCR(Control)       = 0x1140
  bit 12 = 1 -> Autoneg Enabled
  bit 8  = 1 -> Full-Duplex
  bit 6  = 1 -> Speed MSB (=1000M مع bit 13)

REG[01] BMSR(Status)        = 0x796d
  bit 2  = 1 -> Link is UP
  bit 5  = 1 -> Autoneg Complete
  bits 11-14  -> supported modes

REG[02] PHYID1              = 0x0022  <- OUI bits [3:18]
REG[03] PHYID2              = 0x1561  <- OUI bits [19:24] + Model + Rev
  PHY ID = 0x00221561 -> Atheros AR8035
```

#### مجموعة 3: MDIO Bus Statistics

```bash
#!/bin/bash
# mdio_stats.sh
for bus in /sys/kernel/debug/mdio-bus/*/stats; do
    echo "=== Bus: $(dirname $bus | xargs basename) ==="
    cat "$bus" 2>/dev/null
    echo
done

# error rate per bus
for bus in /sys/kernel/debug/mdio-bus/*/; do
    name=$(basename "$bus")
    stats=$(cat "$bus/stats" 2>/dev/null)
    transfers=$(echo "$stats" | grep transfers | awk '{print $NF}')
    errors=$(echo "$stats" | grep errors | awk '{print $NF}')
    if [ -n "$transfers" ] && [ "$transfers" -gt 0 ]; then
        rate=$(echo "scale=2; $errors * 100 / $transfers" | bc)
        echo "Bus: $name | Transfers: $transfers | Errors: $errors | Rate: ${rate}%"
    fi
done
```

#### مجموعة 4: PHY State Monitoring

```bash
#!/bin/bash
# watch_phy_state.sh

echo "module phylib +p" > /sys/kernel/debug/dynamic_debug/control

dmesg -w | awk '
/phy.*state/ { print strftime("%H:%M:%S"), $0 }
/Link is (Up|Down)/ { print strftime("%H:%M:%S"), "*** LINK CHANGE:", $0 }
/MDIO.*error/ { print strftime("%H:%M:%S"), "!!! MDIO ERROR:", $0 }
'
```

#### مجموعة 5: ftrace PHY State Machine

```bash
#!/bin/bash
# ftrace_phy.sh
TRACE=/sys/kernel/debug/tracing

echo 0 > $TRACE/tracing_on
echo > $TRACE/trace
echo function > $TRACE/current_tracer

{
echo "phy_state_machine"
echo "phy_link_change_notify"
echo "phy_trigger_machine"
echo "phy_mac_interrupt"
echo "genphy_update_link"
} > $TRACE/set_ftrace_filter

echo 1 > $TRACE/tracing_on
sleep 10
echo 0 > $TRACE/tracing_on

echo "=== PHY State Machine Trace ==="
cat $TRACE/trace | grep -v "^#" | head -100

echo nop > $TRACE/current_tracer
echo > $TRACE/set_ftrace_filter
```

#### مجموعة 6: Cable Test و PLCA

```bash
# cable test بسيط (Open/Short/OK per pair)
ethtool --cable-test eth0

# TDR test مع distance measurements
# first=0cm, last=100m (10000cm), step=100cm
ethtool --cable-test-tdr eth0 first 0 last 10000 step 100

# النتيجة بتظهر في dmesg
dmesg | grep -i "cable\|tdr\|pair"

# PLCA status للـ 10BASE-T1S networks
ethtool --get-plca-status eth0
```

**تفسير نتيجة cable test:**

```
Pair A code         Open
Pair A len          22m      <- الكابل مفتوح بعد 22 متر
Pair B code         OK
Pair C code         OK
Pair D code         Short
Pair D len          5m       <- قصر بعد 5 متر
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: RK3562 — Industrial Gateway لا يرى الـ PHY على الـ MDIO Bus

#### العنوان
**PHY غير مكتشف أثناء bring-up على بورد RK3562-based industrial gateway**

#### السياق
بورد custom مبني على **RK3562** مع **Realtek RTL8211F** PHY متصل بـ RGMII. المنتج: industrial IoT gateway يُستخدم في خطوط تصنيع. المهندس يشغّل kernel جديد ويلاحظ إن الـ `eth0` مش ظاهر خالص.

#### المشكلة
```bash
$ ip link show
# لا يوجد eth0
$ dmesg | grep phy
rk3562-gmac: no PHY found
```
الـ MDIO bus متسجّل بس ما فيش devices متاكتشفت. الـ `phy_mask` في `struct mii_bus` ضبطها حد غلط.

#### التحليل
الكود في `phy.h` بيعرّف:
```c
struct mii_bus {
    ...
    u32 phy_mask;  /* PHY addresses to be ignored when probing */
    ...
};
```
لما الـ driver بيعمل `mdiobus_register()` والـ kernel يبدأ يعمل scan على العناوين من 0 لـ `PHY_MAX_ADDR - 1` (أي 31)، أي عنوان bit مضبوط في `phy_mask` بيتجاهله. المهندس كان ضبط:
```c
bus->phy_mask = 0xFFFFFFFF; /* خطأ — بيعمل mask على كل العناوين */
```
بدل ما يضب العنوان الصح للـ PHY:
```c
/* RTL8211F على العنوان 0 في هذا الـ design */
bus->phy_mask = ~BIT(0); /* صح — يكشف بس العنوان 0 */
```

الـ `mdiobus_scan_c22()` اللي اتعرّفت في الهيدر:
```c
struct phy_device *mdiobus_scan_c22(struct mii_bus *bus, int addr);
```
بتحاول القراءة من كل عنوان، لكن لما العنوان متمسكوط في `phy_mask` بترجع قبل ما تقرأ أي register.

#### الحل
```c
/* في driver initialization */
bus->phy_mask = 0;  /* امسح الـ mask خالص — اكشف كل العناوين */
```
أو في Device Tree:
```dts
&gmac1 {
    phy-handle = <&phy0>;
    mdio {
        #address-cells = <1>;
        #size-cells = <0>;
        phy0: ethernet-phy@0 {
            reg = <0>;  /* عنوان الـ PHY على الـ MDIO bus */
            compatible = "ethernet-phy-ieee802.3-c22";
        };
    };
};
```
```bash
# تحقق بعد الإصلاح
$ cat /sys/bus/mdio_bus/devices/stmmac-0:00/phy_id
0x001cc916  # RTL8211F
```

#### الدرس المستفاد
**الـ `phy_mask` في `struct mii_bus` بيتحكم في scan — قيمة `0xFFFFFFFF` بتعمي الـ driver عن كل الـ PHYs.** دايما افحص الـ mask قبل ما تبدأ troubleshooting في مستويات أعمق.

---

### السيناريو الثاني: STM32MP1 — Link يطلع وينزل باستمرار (PHY State Machine Flapping)

#### العنوان
**`PHY_RUNNING` → `PHY_NOLINK` → `PHY_RUNNING` بشكل متكرر على STM32MP1 مع KSZ8081**

#### السياق
بورد custom على **STM32MP1** (industrial SoC من STMicroelectronics) مع **Microchip KSZ8081** PHY متصل بـ RMII. المنتج: automotive ECU يتحكم في بيانات CAN-to-Ethernet. المهندس بيشوف الـ link بيقطع كل 3-5 ثواني.

#### المشكلة
```bash
$ dmesg -w | grep eth0
eth0: Link is Up - 100Mbps/Full - flow control off
eth0: Link is Down
eth0: Link is Up - 100Mbps/Full - flow control off
eth0: Link is Down
# يتكرر كل ~4 ثواني
```

#### التحليل
الـ state machine المعرّف في `phy.h`:
```c
enum phy_state {
    PHY_DOWN = 0,
    PHY_READY,
    PHY_HALTED,
    PHY_ERROR,
    PHY_UP,
    PHY_RUNNING,   /* PHY شغّال */
    PHY_NOLINK,    /* مفيش link */
    PHY_CABLETEST,
};
```
الانتقال بين `PHY_RUNNING` و`PHY_NOLINK` بيتحكم فيه إما interrupt أو polling. الـ `phy_polling_mode()`:
```c
static inline bool phy_polling_mode(struct phy_device *phydev)
{
    if (phydev->drv->update_stats)
        return true;

    return phydev->irq == PHY_POLL;  /* PHY_POLL = -1 */
}
```
المشكلة: الـ IRQ line للـ KSZ8081 متوصل بـ GPIO، لكن الـ DT فيه `interrupts` property مش صح، فالـ kernel بيعمل fallback لـ `PHY_POLL`. الـ polling كل `HZ` ثانية بيشوف الـ link state من register، لكن كمان فيه noise على سلك RMII بيسبّب `read_status` يرجع link=0 أحيانا.

بالإضافة، الـ `phydev->irq` اتضبط على `PHY_POLL` بدل الـ GPIO IRQ الصح:
```c
#define PHY_POLL    -1   /* في phy.h */
```

#### الحل
**الخطوة 1** — صلّح الـ DT:
```dts
&fec {
    phy-mode = "rmii";
    phy-handle = <&phy0>;
    mdio {
        phy0: ethernet-phy@0 {
            reg = <0>;
            /* ربط الـ interrupt GPIO الصح */
            interrupt-parent = <&gpioa>;
            interrupts = <7 IRQ_TYPE_LEVEL_LOW>;
            /* منع autoneg مشاكل RMII */
            max-speed = <100>;
        };
    };
};
```
**الخطوة 2** — تحقق إن الـ IRQ اتسجّل صح:
```bash
$ cat /proc/interrupts | grep eth
# يجب يظهر interrupt للـ phy
$ cat /sys/class/net/eth0/phydev/interrupts
1  # يعني interrupts enabled
```

#### الدرس المستفاد
**لما `phydev->irq == PHY_POLL`، الـ PHY بيتحكم فيه polling وليس interrupt، وده ممكن يخلي link state متحوّل بسبب عيّنات لحظية غلط.** دايما تحقق من الـ `irq` value في `phy_device` وتأكد إن الـ GPIO IRQ متوصل صح في الـ DT.

---

### السيناريو الثالث: i.MX8MM — RGMII Delay خاطئ بيخلّي الـ Link يقوم بس بدون Traffic

#### العنوان
**`ping` بيفشل رغم إن `ip link` بيقول link=UP على i.MX8MM مع AR8031**

#### السياق
بورد على **i.MX8M Mini** مع **Atheros AR8031** Gigabit PHY متصل بـ RGMII. المنتج: Android TV box. المهندس بيشتكي إن الـ Ethernet بيقوم بس مفيش packets بتعدي.

#### المشكلة
```bash
$ ip link show eth0
eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ... state UP
$ ping 192.168.1.1
PING 192.168.1.1: 56 data bytes
# لا ردود خالص — 100% packet loss
```

#### التحليل
الـ `phy_interface_t` المعرّف في `phy.h` عنده أربع variants للـ RGMII:
```c
typedef enum {
    ...
    PHY_INTERFACE_MODE_RGMII,       /* بدون delays */
    PHY_INTERFACE_MODE_RGMII_ID,    /* TX + RX delay داخلي في الـ PHY */
    PHY_INTERFACE_MODE_RGMII_RXID,  /* RX delay بس */
    PHY_INTERFACE_MODE_RGMII_TXID,  /* TX delay بس */
    ...
} phy_interface_t;
```
الـ `phy_fix_phy_mode_for_mac_delays()` المعرّف في الهيدر:
```c
phy_interface_t phy_fix_phy_mode_for_mac_delays(phy_interface_t interface,
                                                 bool mac_txid, bool mac_rxid);
```
بتعدّل الـ interface mode بناءً على delays في الـ MAC. المشكلة إن الـ DT ضابط `phy-mode = "rgmii"` بدون delays، لكن الـ AR8031 على هذا الـ board محتاج RX delay من الـ PHY نفسه عشان يعوّض propagation delay على الـ PCB.

بدون الـ delay الصح، الـ MAC بيقدر يستقبل الـ clock لكن البيانات بتوصل خارج نافذة الـ setup time — يعني الـ link يقوم بس البيانات كلها بتتالف.

```c
/* ما بيتّعمل حاليا — link بيقوم بدون delay */
phydev->interface = PHY_INTERFACE_MODE_RGMII;

/* اللي المفروض يتّعمل */
phydev->interface = PHY_INTERFACE_MODE_RGMII_RXID;
```

#### الحل
```dts
/* Device Tree fix */
&fec1 {
    phy-mode = "rgmii-rxid";  /* كان "rgmii" */
    phy-handle = <&ethphy0>;
    ...
};
```
للتحقق عبر `ethtool`:
```bash
$ ethtool eth0 | grep "Duplex\|Speed"
Speed: 1000Mb/s
Duplex: Full

$ ping -c 4 192.168.1.1
PING 192.168.1.1: 4 packets transmitted, 4 received, 0% packet loss
```
وإذا اضطررت تعمله programmatically من الـ driver:
```c
/* في config_init callback */
static int ar8031_config_init(struct phy_device *phydev)
{
    /* فرض RX delay على الـ PHY عبر register */
    phy_write(phydev, 0x1d, 0x05);
    phy_write(phydev, 0x1e, 0x0100); /* enable RX delay */
    return 0;
}
```

#### الدرس المستفاد
**اختيار الـ `phy_interface_t` الغلط (RGMII vs RGMII_RXID/TXID) بيخلي الـ link يقوم بدون traffic.** دايما ارجع لـ board schematic ومواصفات الـ SoC والـ PHY لتحديد مين فيهم بيوفر الـ TX/RX delay.

---

### السيناريو الرابع: AM62x — Custom PHY Driver مش بيشتغل بسبب `phy_id_mask` خاطئ

#### العنوان
**PHY driver مش بيتـbind لـ Microsemi VSC8541 على TI AM62x**

#### السياق
بورد custom على **Texas Instruments AM62x** (Sitara SoC) مع **Microsemi VSC8541** industrial PHY. المنتج: industrial automation controller. المهندس كتب driver جديد للـ VSC8541 بس الـ kernel بيستخدم `genphy` بدله.

#### المشكلة
```bash
$ dmesg | grep phy
MDIO Bus: probed
Generic PHY stmmac-1:01: attached PHY driver [Generic PHY]
# المفروض يكون: Microsemi VSC8541
```

#### التحليل
الـ matching mechanism في `phy.h` بتستخدم:
```c
struct phy_driver {
    u32 phy_id;       /* الـ ID المستهدف */
    u32 phy_id_mask;  /* الـ bits المهمة */
    ...
};
```
مع الـ macros:
```c
#define PHY_ID_MATCH_EXACT(id)  .phy_id = (id), .phy_id_mask = PHY_ID_MATCH_EXTACT_MASK
#define PHY_ID_MATCH_MODEL(id)  .phy_id = (id), .phy_id_mask = PHY_ID_MATCH_MODEL_MASK
#define PHY_ID_MATCH_VENDOR(id) .phy_id = (id), .phy_id_mask = PHY_ID_MATCH_VENDOR_MASK
```

الـ masks المعرّفة:
```c
#define PHY_ID_MATCH_EXTACT_MASK  GENMASK(31, 0)   /* كل الـ 32 bit */
#define PHY_ID_MATCH_MODEL_MASK   GENMASK(31, 4)   /* بدون آخر 4 bits (revision) */
#define PHY_ID_MATCH_VENDOR_MASK  GENMASK(31, 10)  /* vendor فقط */
```

الـ `phy_id_compare()` المستخدمة في الـ matching:
```c
static inline bool phy_id_compare(u32 id1, u32 id2, u32 mask)
{
    return !((id1 ^ id2) & mask);  /* يقارن الـ bits المعرّفة في mask فقط */
}
```

المهندس كتب:
```c
static struct phy_driver vsc8541_driver = {
    .phy_id      = 0x000704c1,  /* خطأ — الـ ID الحقيقي 0x000704c2 */
    .phy_id_mask = 0xffffffff,  /* exact match — أي فرق بيفشّل الـ match */
    ...
};
```
الـ PHY الفعلي برجع `phy_id = 0x000704c2` (revision مختلف)، فالـ matching فشلت.

#### الحل
```c
/* استخدام PHY_ID_MATCH_MODEL يتجاهل الـ revision (آخر 4 bits) */
static const struct phy_driver vsc8541_drivers[] = {
    {
        PHY_ID_MATCH_MODEL(0x000704c0), /* يغطي كل revisions */
        .name = "Microsemi VSC8541",
        .features = PHY_GBIT_FEATURES,
        .config_init = vsc8541_config_init,
        .config_aneg = genphy_config_aneg,
        .read_status = genphy_read_status,
        .suspend     = genphy_suspend,
        .resume      = genphy_resume,
    },
};
```
```bash
# تحقق بعد الإصلاح
$ dmesg | grep phy
Microsemi VSC8541 stmmac-1:01: attached PHY driver [Microsemi VSC8541]

# أو تحقق من الـ phy_id
$ cat /sys/bus/mdio_bus/devices/stmmac-1:01/phy_id
0x000704c2
```

#### الدرس المستفاد
**استخدم `PHY_ID_MATCH_MODEL` دايما في الـ drivers عشان تغطي كل الـ silicon revisions.** `PHY_ID_MATCH_EXACT` مفيد بس لو عايز تتحكم في revision محدد جدا. الـ `phy_id_compare()` بتوضح إن الـ mask هو اللي بيحدد البت المهمة.

---

### السيناريو الخامس: Allwinner H616 — EEE بيسبّب TCP Stalls على Android TV Box

#### العنوان
**TCP sessions بتتجمّد بشكل متقطع بعد idle periods على Allwinner H616 مع RTL8152B**

#### السياق
**Allwinner H616** Android TV box بيستخدم **RTL8152B** USB-to-Ethernet مع Gigabit PHY داخلي. المستخدمين بيشكوا إن الـ streaming بيتوقف لحظات بعد كل pause أو بعد idle.

#### المشكلة
```bash
$ ss -ti dst 1.2.3.4
State    Recv-Q  Send-Q
ESTAB    0       0      # يبدو طبيعي
# لكن بعد 30 ثانية idle:
ESTAB    32768   0      # الـ Recv-Q بيتراكم — stall
```
```bash
$ ethtool --show-eee eth0
EEE settings for eth0:
    EEE status: enabled - active
    Tx LPI: 0 us
    # EEE شغّال وبيسبّب المشكلة
```

#### التحليل
الـ EEE (Energy Efficient Ethernet) الـ fields في `struct phy_device`:
```c
struct phy_device {
    ...
    __ETHTOOL_DECLARE_LINK_MODE_MASK(supported_eee);    /* ما يدعمه الـ PHY */
    __ETHTOOL_DECLARE_LINK_MODE_MASK(advertising_eee);  /* ما بيعلن عنه */
    bool enable_tx_lpi;  /* لما true، MAC يبعت LPI للـ PHY */
    bool eee_active;     /* EEE تم negotiate عليه */
    struct eee_config eee_cfg;  /* user config */
    ...
};
```
وفي `phy.h`:
```c
static inline void phy_disable_eee_mode(struct phy_device *phydev, u32 link_mode)
{
    WARN_ON(phy_is_started(phydev));
    linkmode_set_bit(link_mode, phydev->eee_disabled_modes);
    linkmode_clear_bit(link_mode, phydev->advertising_eee);
}
```
المشكلة: الـ RTL8152B بيدخل **LPI (Low Power Idle)** mode بعد idle period، والـ MAC على الـ H616 بيستغرق وقت طويل جدا في الـ wake-up من LPI، مما بيسبّب TCP timeout وبعدين stall.

```
PHY state عند الـ idle:
RUNNING → [EEE LPI entered] → [MAC delayed wake] → TCP Stall
```

الـ `eee_active = true` يعني الطرفين negotiated EEE، وده بيخلي الـ PHY يدخل LPI بشكل تلقائي.

#### الحل
**الحل 1 — تعطيل EEE من الـ userspace:**
```bash
$ ethtool --set-eee eth0 eee off
$ ethtool --show-eee eth0
EEE status: disabled
```

**الحل 2 — تعطيل EEE programmatically في الـ driver قبل attach:**
```c
static int rtl8152_phy_config_init(struct phy_device *phydev)
{
    /* تعطيل كل EEE modes عشان MAC مش بيتعامل صح مع LPI wake */
    phy_disable_eee_mode(phydev, ETHTOOL_LINK_MODE_100baseT_Full_BIT);
    phy_disable_eee_mode(phydev, ETHTOOL_LINK_MODE_1000baseT_Full_BIT);
    return 0;
}
```

**الحل 3 — الـ DT approach (لو الـ MAC driver يدعم):**
```dts
&emac {
    /* بعض الـ MAC drivers بيدعم هذا الـ property */
    eee-broken-100tx;
    eee-broken-1000t;
};
```

**تحقق إن المشكلة اتحلّت:**
```bash
$ ethtool --show-eee eth0 | grep status
EEE status: disabled

# TCP stalls اتحلّت:
$ ping -i 60 -c 3 192.168.1.1  # بعد idle طويل
3 packets transmitted, 3 received, 0% packet loss
```

#### الدرس المستفاد
**الـ `eee_active` في `phy_device` يعني إن EEE متفعّل بين الـ PHY والـ link partner، لكن الـ MAC نفسه لازم يعرف يتعامل مع LPI wake-up بدون تأخير.** على الـ SoCs المحدودة زي H616، EEE ممكن يسبّب TCP stalls ويحتاج يتعطّل صراحة عبر `phy_disable_eee_mode()` أو من الـ `ethtool`.
## Phase 7: مصادر ومراجع

### التوثيق الرسمي لـ Linux Kernel

| المصدر | الرابط | الوصف |
|--------|--------|-------|
| **PHY Abstraction Layer** (kernel.org) | [docs.kernel.org/networking/phy.html](https://docs.kernel.org/networking/phy.html) | التوثيق الرسمي الكامل لـ PHY Abstraction Layer — يشرح `phy_connect`, `phy_start`, `mdiobus_register` وكل الـ lifecycle |
| **Generic PHY Framework** | [docs.kernel.org/driver-api/phy/index.html](https://docs.kernel.org/driver-api/phy/index.html) | الـ generic PHY framework للـ non-Ethernet PHYs (USB، SATA، PCIe) |
| **phylink documentation** | [docs.kernel.org/networking/sfp-phylink.html](https://docs.kernel.org/networking/sfp-phylink.html) | كيفية migrate من phylib لـ phylink مع SFP support |
| **MDIO bus & PHYs in ACPI** | [docs.kernel.org/firmware-guide/acpi/dsd/phy.html](https://docs.kernel.org/firmware-guide/acpi/dsd/phy.html) | تعريف الـ PHY nodes في ACPI DSD |
| **phy.txt (kernel source)** | `Documentation/networking/phy.rst` | النسخة الداخلية في source tree — ابدأ منها |

---

### مقالات LWN.net

**الـ** LWN.net هي أهم مصدر لمتابعة تطور الـ PHY subsystem في الـ kernel.

| المقال | الرابط | الأهمية |
|--------|--------|---------|
| **RFC: PHY Abstraction Layer II** | [lwn.net/Articles/127013](https://lwn.net/Articles/127013/) | أول patch series لـ Andy Fleming — الأساس التاريخي للـ PAL |
| **PHY Abstraction Layer III** | [lwn.net/Articles/144897](https://lwn.net/Articles/144897/) | التطور المبكر للـ framework قبل merge في mainline |
| **Gianfar PHY Layer Update** | [lwn.net/Articles/146253](https://lwn.net/Articles/146253/) | مثال حقيقي لـ driver يستخدم PAL مع Freescale Gianfar MAC |
| **Generic PHY Framework** (2013) | [lwn.net/Articles/538966](https://lwn.net/Articles/538966/) | مقترح الـ generic PHY framework لـ non-Ethernet |
| **Documentation/phy.txt (3.13)** | [lwn.net/Articles/573337](https://lwn.net/Articles/573337/) | snapshot للـ documentation في kernel 3.13 |
| **Phylink & SFP support** | [lwn.net/Articles/667055](https://lwn.net/Articles/667055/) | مقدمة لـ phylink وكيف حل مشاكل الـ SFP hot-plug |
| **10G PHY support** | [lwn.net/Articles/498257](https://lwn.net/Articles/498257/) | إضافة دعم الـ 10G PHYs في الـ kernel |
| **Rust abstractions for network PHY** | [lwn.net/Articles/948948](https://lwn.net/Articles/948948/) | تجربة كتابة PHY driver بـ Rust — مهم لمتابعة مستقبل الـ kernel |
| **PHY ports representation** | [lwn.net/Articles/1041798](https://lwn.net/Articles/1041798/) | work على multi-port PHY representation (2024) |
| **Introduce ethernet port representation** | [lwn.net/Articles/1009486](https://lwn.net/Articles/1009486/) | مقترح لـ abstraction جديد لـ ethernet ports مع multi-PHY |

---

### الـ Source Code المهم في الـ Kernel

```
drivers/net/phy/          ← كل PHY drivers + phylib core
  phy.c                   ← phy_connect, phy_start, state machine
  phy_device.c            ← struct phy_device lifecycle
  mdio_bus.c              ← mdiobus_register, MDIO bus scanning
  phylink.c               ← phylink implementation
  sfp.c                   ← SFP module support

include/linux/phy.h       ← الـ header الرئيسي — struct phy_device, phy_driver
include/linux/mdio.h      ← MDIO bus types وـ helpers
include/linux/phylink.h   ← phylink API
net/core/ethtool.c        ← ethtool integration مع PHY
```

---

### Commits مهمة في الـ Kernel History

| الحدث | الوصف |
|-------|-------|
| **Linux 2.6.12** | أول merge للـ PHY Abstraction Layer — Andy Fleming, Freescale |
| **Linux 4.2** | إضافة الـ Generic PHY Framework في `drivers/phy/` |
| **Linux 4.5** | بداية عمل Russell King على phylink |
| **Linux 4.11** | أول دمج لـ phylink في mainline |
| **Linux 5.0** | تحسينات الـ clause 45 PHY support |
| **Linux 5.10** | multilink support في Cadence Torrent PHY |
| **Linux 6.12** | PHY listing وـ link_topology tracking |
| **Linux 6.13** | master-slave config عبر device tree |

ابحث عن هذه الـ commits في:
```bash
git log --oneline drivers/net/phy/phy.c | head -30
git log --oneline include/linux/phy.h | head -20
```

---

### Mailing Lists والمجتمع

| القناة | الوصف |
|--------|-------|
| **netdev@vger.kernel.org** | القائمة الرئيسية لـ Linux networking — كل PHY patches تمر هنا |
| **linux-kernel@vger.kernel.org** | القائمة العامة للـ kernel |
| **Archives** | [lore.kernel.org/netdev](https://lore.kernel.org/netdev/) — أرشيف كامل قابل للبحث |

للبحث في الأرشيف:
```bash
# ابحث في lore.kernel.org
https://lore.kernel.org/netdev/?q=phy_connect
https://lore.kernel.org/netdev/?q=phylink+sfp
```

---

### الكتب الموصى بها

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الفصل 17**: Network Drivers — يشرح الـ `net_device`, `ndo_*` ops، والعلاقة بين MAC وـ PHY
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)
- تحذير: الكتاب قديم (2005) لكن المفاهيم الأساسية لا تزال صحيحة

#### Linux Kernel Development, 3rd Edition — Robert Love
- **الفصل 17**: Device Drivers — يعطي context عام للـ driver model
- مرجع ممتاز لفهم `struct device`, `bus_type`, `device_driver` اللي يبني عليها الـ PHY framework

#### Embedded Linux Primer, 2nd Edition — Christopher Hallinan
- **الفصل 15**: Ethernet and Network Subsystem — يتكلم عن الـ PHY في سياق embedded
- مفيد جداً لفهم الـ RGMII/RMII timing issues في real hardware

#### Understanding Linux Network Internals — Christian Benvenuti
- أعمق مرجع للـ networking stack — يشرح الـ MII/MDIO على مستوى الـ registers

---

### kernelnewbies.org — تغييرات الـ PHY عبر الإصدارات

| الإصدار | الرابط | التغيير المهم |
|---------|--------|---------------|
| **Linux 3.9** | [kernelnewbies.org/Linux_3.9_DriverArch](https://kernelnewbies.org/Linux_3.9_DriverArch) | تحسينات الـ Generic PHY Framework |
| **Linux 4.2** | [kernelnewbies.org/Linux_4.2-DriversArch](https://kernelnewbies.org/Linux_4.2-DriversArch) | دمج الـ Generic PHY Framework |
| **Linux 4.9** | [kernelnewbies.org/Linux_4.9](https://kernelnewbies.org/Linux_4.9) | تحسينات PHY لـ embedded platforms |
| **Linux 5.10** | [kernelnewbies.org/Linux_5.10](https://kernelnewbies.org/Linux_5.10) | multilink PHY + USB3+DP combo PHY |
| **Linux 6.2** | [kernelnewbies.org/Linux_6.2](https://kernelnewbies.org/Linux_6.2) | H616 USB PHY + TI PHY modules |
| **Linux 6.10** | [kernelnewbies.org/Linux_6.10](https://kernelnewbies.org/Linux_6.10) | PHY improvements |
| **Linux 6.12** | [kernelnewbies.org/Linux_6.12](https://kernelnewbies.org/Linux_6.12) | PHY listing + link_topology |
| **Linux 6.13** | [kernelnewbies.org/Linux_6.13](https://kernelnewbies.org/Linux_6.13) | master-slave config via DT |

---

### elinux.org

| المصدر | الرابط | الوصف |
|--------|--------|-------|
| **Generic PHY Framework Overview** (PDF) | [elinux.org/images/1/1c/Abraham--generic_phy_framework_an_overview.pdf](https://elinux.org/images/1/1c/Abraham--generic_phy_framework_an_overview.pdf) | presentation ممتاز من ELC عن الـ Generic PHY Framework |

---

### الـ Tools المساعدة

```bash
# فحص PHY state من userspace
ethtool eth0                    # link speed, duplex, autoneg
ethtool -s eth0 speed 100 duplex full   # forced mode

# MDIO register access
mdio-tool eth0 read 1 0         # قراءة register 0 من PHY address 1

# kernel debug
echo 'module phylib +p' > /sys/kernel/debug/dynamic_debug/control
cat /sys/class/net/eth0/phydev/  # PHY sysfs attributes
dmesg | grep -i phy
```

---

### Search Terms للبحث عن معلومات أكثر

```
linux kernel phy_connect phy_start state machine
phylib vs phylink comparison
MDIO clause 22 clause 45 linux
struct phy_driver .config_aneg .read_status
phy_interface_t RGMII SGMII linux
linux ethernet PHY EEE energy efficient
struct mii_bus operations linux
linux PHY LED trigger netdev
phy_loopback linux kernel
linux SFP module phylink sfp_bus
netdev mailing list phylib 2024
```
## Phase 8: Writing simple module

### الـ Function المختارة: `phy_start`

**الـ `phy_start`** هي الـ function الـ exported اللي بتشغّل الـ PHY state machine وبتحوّل الـ PHY من حالة `PHY_UP` للـ polling/interrupt. كل ما interface شبكية اتفتحت (مثلاً `ip link set eth0 up`) اتنادى عليها — فهي نقطة مثالية لمراقبة bring-up الـ network interfaces.

بنستخدم **kprobe** على `phy_start` عشان نطبع معلومات الـ PHY (speed, duplex, interface mode, phy_id) في لحظة التشغيل.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * phy_start_probe.c
 *
 * kprobe on phy_start() — prints PHY info every time a network
 * interface is brought up (e.g. "ip link set eth0 up").
 *
 * Test:
 *   insmod phy_start_probe.ko
 *   ip link set eth0 down && ip link set eth0 up
 *   dmesg | grep phy_start_probe
 *   rmmod phy_start_probe
 */

#include <linux/module.h>        /* MODULE_* macros, module_init/exit        */
#include <linux/kprobes.h>       /* struct kprobe, register_kprobe, etc.     */
#include <linux/phy.h>           /* struct phy_device, phy_modes(), etc.     */
#include <linux/netdevice.h>     /* struct net_device, netdev_name()         */
#include <linux/kernel.h>        /* pr_info()                                */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("PHY Subsystem Student");
MODULE_DESCRIPTION("kprobe on phy_start() to log PHY bring-up events");

/* ------------------------------------------------------------------ */
/*  pre-handler: runs just before phy_start() executes                */
/* ------------------------------------------------------------------ */

/*
 * phy_start() signature:
 *   void phy_start(struct phy_device *phydev);
 *
 * On x86-64, first argument lives in RDI → regs->di.
 * On arm64, first argument lives in X0  → regs->regs[0].
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    struct phy_device *phydev;
    struct net_device *netdev;
    const char *ifname = "unknown";

#ifdef CONFIG_X86_64
    /* x86-64: first arg in RDI */
    phydev = (struct phy_device *)regs->di;
#elif defined(CONFIG_ARM64)
    /* arm64: first arg in X0 */
    phydev = (struct phy_device *)regs->regs[0];
#else
    /* Generic fallback — may not work on all arches */
    phydev = (struct phy_device *)regs->regs[0];
#endif

    /* Validate pointer before dereferencing */
    if (!phydev)
        return 0;

    /* Get the attached network device name if available */
    netdev = phydev->attached_dev;
    if (netdev)
        ifname = netdev->name;

    /*
     * Print meaningful PHY diagnostics:
     *  - interface name (e.g. "eth0")
     *  - phy_id: vendor+model packed as 32-bit OUI
     *  - interface mode (e.g. "rgmii-id", "sgmii")
     *  - current speed and duplex (may be 0 before link-up)
     *  - PHY device address on the MDIO bus
     *  - autoneg flag
     */
    pr_info("phy_start_probe: [%s] phy_start called\n", ifname);
    pr_info("phy_start_probe:   phy_id=0x%08x  mdio_addr=%d\n",
            phydev->phy_id, phydev->mdio.addr);
    pr_info("phy_start_probe:   interface=%s  speed=%d  duplex=%d  autoneg=%u\n",
            phy_modes(phydev->interface),
            phydev->speed,
            phydev->duplex,
            phydev->autoneg);
    pr_info("phy_start_probe:   is_c45=%u  is_internal=%u  link_down_events=%u\n",
            phydev->is_c45,
            phydev->is_internal,
            phydev->link_down_events);

    return 0;   /* 0 = let phy_start() run normally */
}

/* ------------------------------------------------------------------ */
/*  kprobe descriptor                                                  */
/* ------------------------------------------------------------------ */

static struct kprobe kp = {
    .symbol_name = "phy_start",   /* symbol to hook — exported in phy.h */
    .pre_handler = handler_pre,   /* our callback runs before the function */
};

/* ------------------------------------------------------------------ */
/*  init / exit                                                        */
/* ------------------------------------------------------------------ */

static int __init phy_probe_init(void)
{
    int ret;

    /*
     * register_kprobe() finds phy_start() in the kernel symbol table,
     * patches a breakpoint at its entry, and registers our pre_handler.
     * Returns 0 on success or a negative errno.
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("phy_start_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("phy_start_probe: kprobe planted at %pS\n", kp.addr);
    return 0;
}

static void __exit phy_probe_exit(void)
{
    /*
     * unregister_kprobe() removes the breakpoint and frees the kprobe.
     * Must be called in exit to avoid a dangling handler after rmmod;
     * otherwise the kernel will call a handler that no longer exists,
     * causing a panic.
     */
    unregister_kprobe(&kp);
    pr_info("phy_start_probe: kprobe removed from %pS\n", kp.addr);
}

module_init(phy_probe_init);
module_exit(phy_probe_exit);
```

---

### Makefile للـ Module

```makefile
obj-m += phy_start_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

```bash
# تجميع وتحميل الـ module
make
sudo insmod phy_start_probe.ko

# تشغيل interface عشان تطلع الـ output
sudo ip link set eth0 down
sudo ip link set eth0 up

# مشاهدة الـ log
dmesg | tail -20

# إزالة الـ module
sudo rmmod phy_start_probe
```

---

### شرح كل جزء

#### الـ `#include`s

الـ `<linux/kprobes.h>` هو اللي بيجيب `struct kprobe` وكل الـ API الخاص بالـ kprobes. الـ `<linux/phy.h>` بيجيب `struct phy_device` و `phy_modes()` اللي بنحتاجها عشان نطبع interface mode بصيغة نص مقروء.

#### الـ `handler_pre`

بيتنادى عليه قبل ما `phy_start()` تشتغل فعلاً، فبيشوف الـ `phydev` في حالته الأولية (قبل أي تعديل). الـ `pt_regs` هو snapshot للـ registers وقت الـ breakpoint، وعلى x86-64 أول argument بيبقى في `rdi`.

#### الـ `struct kprobe kp`

الـ `.symbol_name` بيخلي الـ kernel يدور على الـ symbol تلقائياً بدل ما نحط عنوان يدوي، وده أكتر portability. الـ `.pre_handler` هو الـ callback اللي اخترناه — ما في `post_handler` لأننا ما محتاجين نشوف الـ return value.

#### الـ `module_init` / `module_exit`

الـ `register_kprobe` بيزرع breakpoint في `phy_start()`، والـ `unregister_kprobe` بيشيله لما نعمل `rmmod`. لازم نعمل unregister في exit عشان لو الـ module اتشال من الذاكرة والـ kprobe لسه مسجّل، أول call لـ `phy_start` بعد كده هتعمل kernel panic لأن الـ handler pointer باظ.

#### الـ output المتوقع

```
phy_start_probe: kprobe planted at phy_start+0x0/0x...
phy_start_probe: [eth0] phy_start called
phy_start_probe:   phy_id=0x004dd072  mdio_addr=1
phy_start_probe:   interface=rgmii-id  speed=0  duplex=255  autoneg=1
phy_start_probe:   is_c45=0  is_internal=0  link_down_events=0
```

الـ `speed=0` قبل link-up طبيعي لأن autoneg لسه ما خلصتش. لما الـ link يطلع بيتعدّل من الـ state machine.
