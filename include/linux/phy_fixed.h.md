## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

**الـ `phy_fixed.h`** جزء من **ETHERNET PHY LIBRARY** اللي بتتضمن كل كود التعامل مع الـ PHY chips في Linux kernel. المسؤولون عنها Andrew Lunn و Heiner Kallweit.

---

### الصورة الكبيرة — قبل أي كود

#### الـ PHY إيه أصلاً؟

الـ **PHY** (Physical Layer Transceiver) هو الـ chip اللي بيعمل الاتصال الفعلي بين الـ MAC (اللي جوه الـ CPU أو الـ SoC) وبين الكابل أو الـ fiber. هو اللي بيتحكم في:

- السرعة (10M / 100M / 1G / ...)
- الـ duplex (full / half)
- الـ link status (متصل ولا لأ)
- الـ pause frames

الـ MAC بيتكلم مع الـ PHY عبر **MDIO bus** — ده bus بسيط جداً (زي I2C بس لشبكات)، بيستخدم عنوان لكل PHY وبيقرأ/يكتب registers معيارية موثقة في **IEEE 802.3**.

#### المشكلة: مفيش PHY chip فعلي!

تخيل إنك بتبني **embedded system** — مثلاً:

- **SoC** زي Raspberry Pi اللي فيه Ethernet MAC جوّاه
- وبتربطه بـ **switch chip** زي DSA switch
- أو بتعمل **virtual network interface** في بيئة virtualization
- أو hardware بيتصل بـ fiber/copper بدون PHY chip حقيقي (الـ MAC بيتكلم مع الـ switch مباشرة عبر RGMII مثلاً)

في الحالة دي، الـ Linux networking stack مش هيشتغل — لأنه **بيفترض** إن في PHY حقيقي على MDIO bus عنده registers بيجي يقراها، ولو ما لقاش PHY، الـ link ما هيطلعش.

#### الحل: Fixed PHY — PHY وهمي بمعلومات ثابتة

الـ **Fixed PHY** هو emulation كاملة لـ PHY chip حقيقي، بس بدل ما يكون hardware chip على MDIO bus، هو **software object** جوه الـ kernel بيتظاهر إنه PHY.

بدل ما الـ kernel يقرأ registers من hardware، الـ fixed PHY بيرجع قيم ثابتة (أو ديناميكية) من software — speed كام، duplex إيه، الـ link up ولا down.

#### التشبيه

تخيل إنك عندك ماكينة قهوة بتتوقع إن فيه كوباية فيها علشان تشتغل. بس أنت عايزها تشتغل من غير كوباية (اختبار مثلاً). الحل: تحط **dummy cup** — كوباية وهمية بتخدع الـ sensor إن في كوباية. ده بالظبط الـ fixed PHY.

---

### السيناريوهات الحقيقية

| السيناريو | ليه محتاج Fixed PHY |
|---|---|
| DSA switch مربوط بـ CPU port | الـ CPU port مش فيه PHY — الـ link always up بـ 1Gbps |
| Virtual NIC في KVM/QEMU | مفيش hardware خالص |
| Embedded board بـ RGMII بدون PHY chip | الـ MAC متصل بالـ switch مباشرة |
| Loopback testing | محتاج interface شغال بدون hardware |

---

### دور الـ `phy_fixed.h` تحديداً

الـ header ده هو **الـ public API** اللي drivers الـ network بتستخدمها علشان:

1. **تسجّل Fixed PHY** — تقول للـ kernel "اعمل PHY وهمي بالخصائص دي"
2. **تلغي التسجيل** لما الـ driver بيتشال
3. **تغيّر الـ carrier state** (link up/down) ديناميكياً من الـ driver
4. **تحدد callback** لو الـ link status بيجي من hardware مش ثابت

```c
/* الـ struct الوحيد في الـ header — بيحدد خصائص الـ Fixed PHY */
struct fixed_phy_status {
    int speed;        /* 10, 100, 1000 Mbps */
    int duplex;       /* DUPLEX_HALF أو DUPLEX_FULL */
    bool link:1;      /* link up أو down */
    bool pause:1;     /* flow control enabled */
    bool asym_pause:1;/* asymmetric pause */
};
```

---

### كيف الـ Fixed PHY بيشتغل — القصة كاملة

```
+------------------+        MDIO bus (وهمي)        +-------------------+
|  Network Driver  |  ----registers read/write--->  |  Fixed PHY (sw)   |
|  (e.g. DSA, KVM) |                                |  fixed_phy.c      |
+------------------+                                +-------------------+
         |                                                    |
         | fixed_phy_register()                     بيرجع values
         | fixed_phy_set_link_update()              من fixed_phy_status
         v                                          مش من hardware
+------------------+
| phylib (phy.c)   |  <-- ده ما بيعرفش الفرق بين PHY حقيقي ووهمي
+------------------+
```

1. الـ driver بيعمل `fixed_phy_register()` بـ status (speed=1000, duplex=full, link=1)
2. الـ kernel بيعمل **MDIO bus وهمي** (`fmb_mii_bus`) وبيسجّل عليه الـ Fixed PHY
3. لو حد قرأ MDIO register، الـ `fixed_mdio_read()` بيرجع القيمة من الـ software state بدل hardware
4. الـ **phylib** (اللي فوق الـ Fixed PHY) مش شايف فرق — بيتعامل معاه زي أي PHY عادي
5. لو الـ driver عنده callback (عبر `fixed_phy_set_link_update`)، الـ kernel بيسأله كل poll cycle عن الـ link state الحقيقي

---

### الـ Compile-time Guard

```c
#if IS_ENABLED(CONFIG_FIXED_PHY)
    /* الـ real functions */
extern int fixed_phy_change_carrier(...);
struct phy_device *fixed_phy_register(...);
...
#else
    /* stub functions ترجع -ENODEV لو مش enabled */
static inline struct phy_device *fixed_phy_register(...) {
    return ERR_PTR(-ENODEV);
}
#endif
```

لو الـ `CONFIG_FIXED_PHY` مش enabled، الـ header بيوفر **inline stubs** تضمن إن الكود بيكمبايل من غير errors حتى لو الـ feature مش موجودة.

---

### الملفات المكونة للـ Subsystem

#### Core Files

| الملف | الدور |
|---|---|
| `include/linux/phy_fixed.h` | Public API + struct تعريفات (الملف ده) |
| `drivers/net/phy/fixed_phy.c` | Implementation — الـ Fixed MDIO bus والـ registration |
| `drivers/net/phy/swphy.c` | Software PHY register emulation — بيحسب قيم MDIO registers من status |
| `drivers/net/phy/swphy.h` | Header لـ swphy.c |

#### PHY Library Core

| الملف | الدور |
|---|---|
| `include/linux/phy.h` | الـ main PHY API — struct `phy_device`, `mii_bus`, phylib كله |
| `include/linux/mii.h` | MII register definitions + helpers |
| `include/linux/mdio.h` | MDIO bus definitions |
| `drivers/net/phy/phy.c` | phylib core — state machine، polling، events |
| `drivers/net/phy/mdio_bus.c` | MDIO bus registration والـ scanning |
| `drivers/net/mdio/of_mdio.c` | Device tree parsing لـ MDIO buses |

#### ملفات ذات صلة تستحق المعرفة

| الملف | ليه مهم |
|---|---|
| `include/linux/phylib_stubs.h` | Stubs لـ phylib لما مش built-in |
| `Documentation/networking/phy.rst` | توثيق كامل للـ PHY subsystem |
| `Documentation/devicetree/bindings/net/fixed-link.yaml` | Device tree binding لـ fixed PHY |

---

### خلاصة

الـ `phy_fixed.h` بيحل مشكلة بسيطة جداً بأناقة: **Linux networking بيفترض دايماً وجود PHY hardware**، لكن في hardware كتير الـ PHY chip مش موجود. الـ Fixed PHY بيعمل "PHY وهمي" في software بيشبع توقعات الـ phylib framework، وده بيخلي الـ driver code يشتغل بشكل موحد سواء كان في hardware PHY حقيقي أو لأ.
## Phase 2: شرح الـ Fixed PHY Framework

### المشكلة — ليه الـ Fixed PHY موجود أصلاً؟

الـ **PHY (Physical Layer transceiver)** في الشبكات هو chip منفصل بيتكلم مع الـ MAC عبر bus اسمه **MDIO (Management Data Input/Output)**. الكيرنل عنده PHY subsystem كامل بيعمل:

- Discovery للـ PHY chip عبر الـ MDIO bus
- قراءة registers منه (speed، duplex، link status)
- تشغيل state machine بتتعامل مع link up/down

المشكلة: **مش كل connection بين MAC وشبكة بيمر عبر PHY chip حقيقي بيدعم MDIO.**

الحالات دي شايعة جداً في embedded systems:

| الحالة | المثال |
|---|---|
| Switch chip متصل بـ MAC عبر RGMII ثابت | Marvell 88E6xxx ، Atheros AR8xxx |
| SFP module بدون negotiation | fiber link بـ 1G fixed |
| internal MAC-to-MAC connection | بين SoC cores |
| emulated network في FPGA أو simulator | QEMU virtio-net |
| industrial controller بـ fixed 100Mbps | لا يوجد MDIO على الإطلاق |

في كل الحالات دي الـ MAC driver بيحتاج يتكلم مع الـ PHY subsystem بنفس الـ API، لكن مفيش PHY chip حقيقي يتكلم معاه.

**الـ solution الساذجة**: المبرمج يعمل special case في الـ MAC driver ويتجاهل الـ PHY layer.
**المشكلة**: ده يكسر كل الـ generic network stack اللي بيفترض وجود `phy_device`.

---

### الـ Solution — الـ Fixed PHY كـ "Software PHY"

الـ Fixed PHY Framework بيحل المشكلة عن طريق **خلق PHY device وهمي (virtual)** بيسجّل نفسه في الـ PHY subsystem بس بدل ما بيقرأ registers من hardware، بيرجع status ثابت أو بيستدعي callback من الـ MAC driver علشان يعرف الـ link state.

بكده:
- الـ MAC driver بيتعامل مع `phy_device *` زي أي PHY حقيقي
- الـ PHY subsystem مش بيعرفش الفرق
- الـ `fixed_phy` بيوفر الـ "illusion" المطلوبة

---

### الـ Real-World Analogy — المقياس الكهربائي الوهمي

تخيل عندك نظام صناعي بيقيس التيار الكهربائي. النظام مصمم يقرأ من sensor حقيقي عبر I2C. بس في بيئة testing، مفيش sensor — بدله software بيرجع قيمة ثابتة.

| الـ Analogy | الـ Kernel Concept |
|---|---|
| نظام قياس التيار | الـ MAC driver (مثلاً `stmmac`) |
| الـ I2C bus | الـ MDIO bus |
| الـ sensor الحقيقي | الـ PHY chip (مثلاً Marvell 88E1111) |
| الـ software sensor | الـ `fixed_phy` device |
| القيمة المُعادة (ثابتة أو محسوبة) | الـ `fixed_phy_status` struct |
| الـ callback بيسأل "هل التيار ماشي؟" | الـ `link_update()` callback |
| الـ I2C address | الـ PHY address على الـ MDIO bus (وهمي) |

الفرق المهم: الـ fixed PHY مش بس "بيرجع قيمة ثابتة" — ممكن يكون ليه callback يتصل بيه كل ما الـ state machine تحتاج تعرف link status، وده بيخليه dynamic مش static خالص.

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Network Stack (L3/L4)                    │
└─────────────────────────┬───────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────┐
│              net_device  (struct net_device)                │
│          e.g. eth0  —  driven by MAC driver                 │
└──────────┬──────────────────────────────────────────────────┘
           │  phy_connect() / phy_attach()
           │
┌──────────▼──────────────────────────────────────────────────┐
│              PHY Subsystem  (drivers/net/phy/)              │
│                                                             │
│   ┌─────────────────────┐   ┌────────────────────────────┐ │
│   │   Real PHY Device   │   │   Fixed PHY Device         │ │
│   │   (e.g. 88E1111)    │   │   (fixed_phy.c)            │ │
│   │                     │   │                            │ │
│   │  reads MDIO regs    │   │  reads fixed_phy_status    │ │
│   │  real HW registers  │   │  OR calls link_update()    │ │
│   └────────┬────────────┘   └────────────┬───────────────┘ │
│            │                             │                  │
│   ┌────────▼─────────────────────────────▼───────────────┐ │
│   │            struct phy_device                         │ │
│   │  .speed, .duplex, .link, .state (PHY state machine)  │ │
│   └───────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
           │
           │  (Real PHY only)
┌──────────▼──────────────────────────────────────────────────┐
│              MDIO Bus  (struct mii_bus)                     │
│              read/write registers via MDIO protocol         │
└──────────┬──────────────────────────────────────────────────┘
           │
┌──────────▼──────────────────────────────────────────────────┐
│              PHY Chip  (silicon on PCB)                     │
│              e.g. Marvell 88E1111, Microchip LAN8720        │
└─────────────────────────────────────────────────────────────┘
```

الـ Fixed PHY بيعيش في نفس الـ layer زي الـ real PHY — وده هو جوهر الفكرة. الـ MAC driver ماشفش الفرق.

---

### الـ Core Abstractions

#### 1. `struct fixed_phy_status`

```c
struct fixed_phy_status {
    int speed;        /* SPEED_10 / SPEED_100 / SPEED_1000 */
    int duplex;       /* DUPLEX_HALF / DUPLEX_FULL */
    bool link:1;      /* is the link up? */
    bool pause:1;     /* PAUSE frame support */
    bool asym_pause:1;/* asymmetric PAUSE support */
};
```

ده الـ **snapshot** للـ link state اللي الـ fixed PHY بيشوفه. مفيش negotiation — القيم دي إما:
- ثابتة من وقت الـ registration (static config)
- أو بتتحدث عن طريق `link_update()` callback كل ما الـ PHY state machine تسأل

#### 2. الـ `link_update` Callback

```c
int (*link_update)(struct net_device *dev,
                   struct fixed_phy_status *status);
```

ده الـ **dynamic hook** — بيخلي الـ MAC driver يحكم على الـ link state بطريقته الخاصة. مثلاً:
- Switch driver بيسأل الـ switch chip "هل الـ port ده متصل؟"
- Fiber driver بيقرأ SFP module signal detect
- FPGA driver بيقرأ register من logic

الـ PHY state machine بتشتغل زي العادي بس بدل ما تقرأ MDIO registers، بتستدعي الـ callback ده.

---

### الـ API

#### `fixed_phy_register()`

```c
struct phy_device *fixed_phy_register(
    const struct fixed_phy_status *status,
    struct device_node *np           /* DT node, can be NULL */
);
```

بيعمل `phy_device` جديد، بيسجله في الـ PHY subsystem، وبيرجع pointer ليه. الـ MAC driver بعدين يعمل `phy_connect()` على الـ `phy_device` ده زي أي PHY عادي.

#### `fixed_phy_register_100fd()`

```c
struct phy_device *fixed_phy_register_100fd(void);
```

Convenience wrapper — بيعمل fixed PHY بـ 100Mbps Full Duplex link up. الأكثر استخداماً في بيئات simple embedded.

#### `fixed_phy_set_link_update()`

```c
int fixed_phy_set_link_update(
    struct phy_device *phydev,
    int (*link_update)(struct net_device *,
                       struct fixed_phy_status *)
);
```

بيربط الـ dynamic callback بعد الـ registration. الـ MAC driver بيستدعيه بعد ما يعمل `fixed_phy_register()`.

#### `fixed_phy_change_carrier()`

```c
int fixed_phy_change_carrier(struct net_device *dev, bool new_carrier);
```

بيغير الـ carrier state من خارج الـ PHY — مفيد لما الـ MAC driver عنده event خارجي (مثلاً interrupt من switch chip) ومحتاج يحدّث الـ link state فوراً بدون انتساب للـ polling cycle.

#### `fixed_phy_unregister()`

```c
void fixed_phy_unregister(struct phy_device *phydev);
```

بيشيل الـ virtual PHY من الـ subsystem عند الـ cleanup.

---

### كيف الـ Structs بتترابط

```
struct net_device (eth0)
    │
    │  .phydev ──────────────────────────────────────────────┐
    │                                                        │
    ▼                                                        ▼
MAC driver code                               struct phy_device
    │                                             │
    │  fixed_phy_register()                       │  .drv ──► struct phy_driver
    │  fixed_phy_set_link_update(cb)              │            (fixed_phy driver)
    │                                             │
    │                                             │  .mdio ──► struct mdio_device
    │                                             │             (virtual MDIO device
    │                                             │              on virtual mii_bus)
    │
    │  cb = link_update(net_dev, *status)
    │       └──► writes into fixed_phy_status
    │                 .speed   = 1000
    │                 .duplex  = DUPLEX_FULL
    │                 .link    = true
    │
    └──► PHY state machine reads fixed_phy_status
         instead of MDIO registers
```

**ملاحظة مهمة**: الـ `fixed_phy` بيعمل **virtual mii_bus** واحد بيستضيف كل الـ fixed PHY devices. الـ bus ده مش بيتعامل مع hardware — الـ read/write operations فيه بترجع values من الـ `fixed_phy_status` مباشرة.

---

### ما الـ Fixed PHY يملكه وما يفوّضه

| المسؤولية | مين بيتعامل معاها؟ |
|---|---|
| تعريف `phy_device` وتسجيله | Fixed PHY Framework |
| الـ PHY state machine (UP/DOWN/RUNNING) | PHY Subsystem العام (ما يتغيرش) |
| قراءة link status | إما `fixed_phy_status` ثابت أو `link_update()` callback من الـ MAC |
| MDIO register read/write | الـ virtual mii_bus بيرجع fake data |
| تحديث `net_device` carrier | PHY Subsystem عبر `netif_carrier_on/off()` |
| تحديد speed/duplex | MAC driver عبر `fixed_phy_status` |
| Auto-negotiation | **لا أحد** — مش موجودة في fixed link |
| Interrupt handling | **لا أحد** — بيستخدم polling دايماً (`PHY_POLL`) |

---

### مثال عملي — Switch Chip على RGMII

```c
/* MAC driver probe function */
static int my_mac_probe(struct platform_device *pdev)
{
    struct fixed_phy_status fps = {
        .link   = 1,
        .speed  = 1000,
        .duplex = DUPLEX_FULL,
    };

    /* Create virtual PHY — no MDIO, no real chip */
    phydev = fixed_phy_register(&fps, pdev->dev.of_node);
    if (IS_ERR(phydev))
        return PTR_ERR(phydev);

    /* Attach a dynamic callback — switch chip tells us real link status */
    fixed_phy_set_link_update(phydev, my_switch_link_update);

    /* Connect to net_device — same as with real PHY */
    phy_connect(ndev, phydev_name(phydev), my_mac_adjust_link,
                PHY_INTERFACE_MODE_RGMII);
    phy_start(phydev);
    return 0;
}

/* Called by PHY state machine to know actual link status */
static int my_switch_link_update(struct net_device *ndev,
                                  struct fixed_phy_status *status)
{
    /* Read real status from switch chip via SPI/I2C/memory-mapped */
    status->link   = switch_get_port_link(MY_PORT);
    status->speed  = switch_get_port_speed(MY_PORT);
    status->duplex = DUPLEX_FULL;
    return 0;
}
```

---

### الـ Kconfig Guard — `CONFIG_FIXED_PHY`

الـ header بيستخدم `IS_ENABLED(CONFIG_FIXED_PHY)` علشان يوفر stubs لما الـ feature مش مفعّلة:

```c
#if IS_ENABLED(CONFIG_FIXED_PHY)
extern struct phy_device *fixed_phy_register(...);
/* real implementations */
#else
static inline struct phy_device *fixed_phy_register(...)
{
    return ERR_PTR(-ENODEV);  /* stub — returns error */
}
#endif
```

ده pattern شايع في الكيرنل: الـ MAC driver بيقدر يستدعي `fixed_phy_register()` بدون `#ifdef` في الكود بتاعه — لو الـ config مش موجودة، بيرجع `-ENODEV` وبس.

`fixed_phy_change_carrier()` و `fixed_phy_set_link_update()` مش ليهم stubs عشان لو الـ config مش موجودة مش هيتم استدعاؤهم أصلاً (الـ `fixed_phy_register()` بيرجع error قبلهم).
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Config Options والـ Flags — Cheatsheet

#### Config Options

| Option | المعنى |
|---|---|
| `CONFIG_FIXED_PHY` | يفعّل دعم الـ fixed PHY في الكيرنل. لو مش موجود، كل الـ API بتتحول لـ stubs ترجع `ERR_PTR(-ENODEV)`. |

#### Flags في `struct fixed_phy_status`

| الـ Field | النوع | المعنى |
|---|---|---|
| `speed` | `int` | سرعة الـ link: `10`, `100`, `1000`, إلخ (Mbps) |
| `duplex` | `int` | `DUPLEX_FULL` أو `DUPLEX_HALF` |
| `link` | `bool:1` | الـ link موجود (`1`) أو لا (`0`) |
| `pause` | `bool:1` | الـ symmetric pause مفعّل |
| `asym_pause` | `bool:1` | الـ asymmetric pause مفعّل |

#### States في `enum phy_state` (المرتبطة بالـ fixed PHY)

| الـ State | القيمة | المعنى |
|---|---|---|
| `PHY_DOWN` | 0 | الـ PHY مش جاهز خالص، لسه في probe |
| `PHY_READY` | 1 | الـ driver جاهز، بس الـ link مش اتشغّل لسه |
| `PHY_HALTED` | 2 | الـ PHY موجود بس polling/interrupts متوقفة |
| `PHY_ERROR` | 3 | في error |
| `PHY_UP` | 4 | الـ PHY اشتغل، بيتحقق من الـ link |
| `PHY_RUNNING` | 5 | الـ link شغّال وبتتبادل packets |
| `PHY_NOLINK` | 6 | الـ PHY شغّال بس الـ link مقطوع |
| `PHY_CABLETEST` | 7 | بيعمل cable diagnostics |

---

### الـ Structs المهمة

#### 1. `struct fixed_phy_status`

**الغرض:** بتمثّل الحالة الثابتة (static) لـ PHY وهمي — PHY مش موجود فيزيائياً ومحتاجش MDIO bus حقيقي. بتحدد إيه سرعة الـ link وإيه الـ capabilities بتاعته.

```c
struct fixed_phy_status {
    int  speed;        /* link speed in Mbps: 10, 100, 1000 */
    int  duplex;       /* DUPLEX_FULL or DUPLEX_HALF */
    bool link:1;       /* 1 = link up, 0 = link down */
    bool pause:1;      /* symmetric flow control */
    bool asym_pause:1; /* asymmetric flow control */
};
```

**العلاقات:**
- بتتبعت كـ `const` pointer لـ `fixed_phy_register()` عشان تحدد specs الـ PHY وقت الـ registration.
- الـ `link_update` callback بيقدر يعدّل نسخة منها runtime لو الحالة اتغيرت.
- القيم بتتنسخ جوّا الـ `phy_device` في الـ fields: `speed`, `duplex`, `link`, `pause`, `asym_pause`.

---

#### 2. `struct phy_device` (من `include/linux/phy.h`)

**الغرض:** الـ struct الأساسي اللي بيمثّل أي PHY في الـ kernel، سواء حقيقي أو وهمي (fixed). الـ fixed PHY بيعمل instance منه بدون MDIO bus hardware حقيقي.

**الـ Fields الأهم في سياق fixed PHY:**

| الـ Field | النوع | الأهمية للـ fixed PHY |
|---|---|---|
| `mdio` | `struct mdio_device` | الـ base device، بيوّرثه الـ phy_device |
| `drv` | `const struct phy_driver *` | الـ driver بيكون `fixed_phy_driver` |
| `is_pseudo_fixed_link` | `unsigned:1` | بيتحدد `1` للـ fixed PHY |
| `speed` | `int` | السرعة الحالية، بتيجي من `fixed_phy_status` |
| `duplex` | `int` | الـ duplex الحالي |
| `link` | `unsigned:1` | حالة الـ link الحالية |
| `pause` / `asym_pause` | `bool:1` | الـ flow control |
| `attached_dev` | `struct net_device *` | الـ MAC اللي اتربط بيه |
| `state` | `enum phy_state` | الـ state machine الحالية |
| `lock` | `struct mutex` | بيحمي access للـ phy_device |
| `state_queue` | `struct delayed_work` | الـ polling work queue |
| `priv` | `void *` | بيشاور على `fixed_phy` struct الداخلي |
| `adjust_link` | `void (*)(struct net_device *)` | الـ callback للـ MAC لما الـ link يتغير |

---

#### 3. `struct device_node` (من `include/linux/of.h`)

**الغرض:** بتمثّل node في الـ Device Tree. الـ fixed PHY بياخد pointer ليها عشان يربط نفسه بـ DT node معين، وده بيخلي الـ MAC يلاقيه من خلال `of_phy_find_device()`.

```c
struct device_node {
    const char        *name;        /* node name in DT */
    const char        *full_name;   /* full path */
    struct fwnode_handle fwnode;    /* firmware node handle */
    struct property   *properties;  /* DT properties list */
    struct device_node *parent;     /* parent node */
    struct device_node *child;      /* first child */
    struct device_node *sibling;    /* next sibling */
    unsigned long      _flags;      /* internal flags */
};
```

**العلاقة بالـ fixed PHY:**
- الـ `np` parameter في `fixed_phy_register()` بيربط الـ PHY بـ DT node.
- لو `np == NULL` يعني مش محتاج ربط DT.
- الـ MAC driver بيعمل `of_phy_find_device(np)` ويلاقي الـ `phy_device` اللي اتسجّل بنفس الـ `np`.

---

### مخططات العلاقات بين الـ Structs

```
┌─────────────────────────────────────────────────────────────┐
│                    fixed_phy_register()                     │
│                     (caller passes)                         │
└──────────┬──────────────────────────┬───────────────────────┘
           │                          │
           ▼                          ▼
┌──────────────────┐        ┌─────────────────────┐
│ fixed_phy_status │        │   device_node (np)  │
│ ────────────────│        │ ───────────────────  │
│  speed           │        │  name               │
│  duplex          │        │  full_name          │
│  link:1          │        │  properties         │
│  pause:1         │        │  fwnode             │
│  asym_pause:1    │        └──────────┬──────────┘
└──────────────────┘                   │
           │                           │ np stored
           │ values copied             │ in mdio.dev
           ▼                           ▼
┌──────────────────────────────────────────────────────┐
│                   phy_device                         │
│ ──────────────────────────────────────────────────── │
│  mdio (struct mdio_device)                           │
│    └── dev (struct device)                           │
│         └── of_node ──────────────► device_node     │
│  drv ──────────────────────────────► phy_driver     │
│  is_pseudo_fixed_link = 1                            │
│  speed / duplex / link / pause / asym_pause          │
│  state (enum phy_state)                              │
│  lock (mutex)                                        │
│  state_queue (delayed_work)                          │
│  attached_dev ─────────────────────► net_device     │
│  adjust_link() callback                              │
│  priv ──────────────────────────────► fixed_phy     │
│                                       (internal)     │
└──────────────────────────────────────────────────────┘
           │
           │ fixed_phy_set_link_update() يحط
           ▼
┌──────────────────────────────────────────────────────┐
│  link_update callback                                │
│  int (*link_update)(struct net_device *,             │
│                     struct fixed_phy_status *)       │
│  ── بيتبعت من driver/MAC عشان يعدّل الـ status      │
│     بناءً على hardware state خاص (مثلاً switch)     │
└──────────────────────────────────────────────────────┘
```

---

### مخطط الـ Lifecycle

```
1. CREATION
───────────
Bootup / Driver probe
        │
        ▼
fixed_phy_register(status, np)
        │
        ├─► alloc phy_device
        ├─► copy status → phy_device fields
        ├─► set is_pseudo_fixed_link = 1
        ├─► attach to fake MDIO bus (fixed_mdio_bus)
        ├─► register with mdio subsystem
        └─► store np in dev.of_node
        │
        ▼
   phy_device ready (PHY_READY state)

2. ATTACHMENT (MAC connects to PHY)
────────────────────────────────────
MAC driver calls:
        phy_connect(netdev, phy_id, adjust_link, interface)
  or    of_phy_connect(netdev, np, adjust_link, 0, interface)
        │
        ├─► phy_device.attached_dev = netdev
        ├─► phy_device.adjust_link = callback
        └─► state → PHY_UP

3. OPERATION (link polling)
────────────────────────────
PHY state machine (state_queue delayed_work):
        │
        ├─► calls fixed driver's read_status()
        │     └─► calls link_update() if set
        │           └─► updates fixed_phy_status
        │     └─► copies status to phy_device fields
        │
        ├─► if link changed → calls adjust_link(netdev)
        │     └─► MAC driver updates netif carrier
        │
        └─► reschedule state_queue (polling interval)

4. CARRIER CHANGE (runtime)
────────────────────────────
fixed_phy_change_carrier(netdev, new_carrier)
        │
        └─► directly calls netif_carrier_on/off

5. TEARDOWN
────────────
Driver unload or device removal
        │
        ▼
fixed_phy_unregister(phydev)
        │
        ├─► phy_disconnect() — detach from netdev
        ├─► mdiobus_unregister_device()
        └─► phy_device_free() — release memory
```

---

### مخططات الـ Call Flow

#### تسجيل الـ PHY (Registration Flow)

```
driver/board code
  └─► fixed_phy_register(status, np)
        │
        ├─► fixed_phy_alloc()            [يعمل alloc للـ phy_device]
        │     └─► phy_device_create()
        │           └─► alloc + init phy_device
        │
        ├─► copy: speed/duplex/link/pause → phydev
        │
        ├─► phydev->is_pseudo_fixed_link = 1
        │
        ├─► mdiobus_register_device(&phydev->mdio)
        │     └─► registers on fixed_mdio_bus (fake bus)
        │
        └─► if np:
              phydev->mdio.dev.of_node = np
              └─► of_phy_find_device() will work later
```

#### تحديث الـ link_update callback

```
MAC driver (e.g., DSA switch port driver)
  └─► fixed_phy_set_link_update(phydev, my_link_update)
        │
        └─► stores callback in internal fixed_phy struct
              (accessible via phydev->priv)

Later, during PHY polling:
  state_queue work runs
    └─► fixed_phy_driver.read_status(phydev)
          └─► retrieves fixed_phy from phydev->priv
                └─► if link_update:
                      link_update(phydev->attached_dev,
                                  &status)
                        │
                        └─► MAC/switch driver reads
                            hardware state and fills
                            fixed_phy_status
                └─► copies status → phydev fields
```

#### تغيير الـ carrier مباشرة

```
kernel code (e.g., USB gadget, virtual interface)
  └─► fixed_phy_change_carrier(netdev, true/false)
        │
        ├─► finds phy_device from netdev->phydev
        └─► netif_carrier_on(netdev)  [if true]
            netif_carrier_off(netdev) [if false]
```

#### تسجيل سريع بـ 100FD

```
driver calls fixed_phy_register_100fd()
  └─► internally calls:
        fixed_phy_register(
          &(fixed_phy_status){
            .link   = 1,
            .speed  = 100,
            .duplex = DUPLEX_FULL,
          },
          NULL   /* no DT node */
        )
  └─► returns ready phy_device
```

---

### استراتيجية الـ Locking

#### الـ Locks المستخدمة

| الـ Lock | النوع | موجود في | بيحمي إيه |
|---|---|---|---|
| `phydev->lock` | `struct mutex` | `phy_device` | كل الـ state transitions، الـ read_status، الـ config |
| `rtnl_lock` | global semaphore | kernel net core | الـ `attached_dev`، الـ `sfp_bus_attached` |

#### ترتيب الـ Locking (Lock Ordering)

```
rtnl_lock
  └─► phydev->lock
        └─► [no deeper locks in fixed PHY path]
```

**قاعدة ثابتة:** لازم تاخد `rtnl_lock` الأول لو محتاج تعدّل في الـ `attached_dev` أو تعمل `phy_connect/disconnect`. بعدين تاخد `phydev->lock` لو محتاج تعدّل الـ state الداخلية.

#### ملاحظات خاصة بالـ Fixed PHY

- الـ fixed PHY **مش بيستخدم interrupts** — كل حاجة polling عبر `state_queue`.
- الـ `link_update` callback **بيتنادى من جوّا `phydev->lock`** — يعني الـ callback نفسه **ممنوع يحاول ياخد نفس الـ lock** (deadlock).
- الـ `fixed_phy_change_carrier()` بيشتغل من أي context (بما فيها atomic)، لأنه **مش بياخد أي lock**، بس بيشتغل على `netif_carrier_*` اللي thread-safe.
- الـ `fixed_mdio_bus` (الـ fake bus) بيحمي نفسه بـ `mdio_bus->mdio_lock` لو في access متزامن من ناحية الـ sysfs أو ethtool.
## Phase 4: شرح الـ Functions

---

### ملخص الـ API — Cheatsheet

| Function | Returns | الغرض الرئيسي |
|---|---|---|
| `fixed_phy_register()` | `struct phy_device *` أو `ERR_PTR` | تسجيل fixed PHY بـ status محدد |
| `fixed_phy_register_100fd()` | `struct phy_device *` أو `ERR_PTR` | shortcut لتسجيل 100Mbps full-duplex |
| `fixed_phy_unregister()` | `void` | إلغاء تسجيل وتحرير الـ fixed PHY |
| `fixed_phy_set_link_update()` | `int` | ربط callback لتحديث حالة الـ link |
| `fixed_phy_change_carrier()` | `int` | تغيير حالة الـ carrier من الـ MAC driver |

---

### الـ struct المركزي: `fixed_phy_status`

```c
struct fixed_phy_status {
    int  speed;        /* link speed: 10 / 100 / 1000 Mbps */
    int  duplex;       /* DUPLEX_HALF or DUPLEX_FULL */
    bool link:1;       /* is link up? */
    bool pause:1;      /* symmetric pause capable */
    bool asym_pause:1; /* asymmetric pause capable */
};
```

**الـ** `fixed_phy_status` هو الـ snapshot الكامل لحالة الـ link على الـ fixed PHY. مفيش MDIO register reads حقيقية — الـ kernel بيقرأ من الـ struct ده مباشرة عبر `swphy_read_reg`. الـ driver بيملاه إما statically وقت التسجيل أو ديناميكياً عبر الـ `link_update` callback.

---

### Group 1: Registration — تسجيل الـ Fixed PHY

الـ fixed PHY subsystem بيسمح لأي MAC driver يشتغل مع link ثابت (switch port، SFP module بدون PHY حقيقي، أو loopback) من غير ما يحتاج MDIO bus فعلي. الـ registration بتخلق `phy_device` وهمي على bus خاص داخلي اسمه `fixed_mdio_bus`.

---

#### `fixed_phy_register()`

```c
struct phy_device *fixed_phy_register(
    const struct fixed_phy_status *status,
    struct device_node *np
);
```

**ما بتعمله:**
دي الـ entry point الأساسي للـ subsystem. بتأخذ الـ initial link status، بتتحقق منه، بتحجز slot على الـ virtual MDIO bus، وبتسجّل `phy_device` كامل في الـ kernel. لو `np` مش NULL، بتربط الـ PHY بالـ device tree node عشان `of_phy_connect()` تلاقيه لاحقاً.

**الـ Parameters:**

| Parameter | النوع | الشرح |
|---|---|---|
| `status` | `const struct fixed_phy_status *` | الحالة الابتدائية للـ link — speed, duplex, pause flags |
| `np` | `struct device_node *` | الـ DT node للـ fixed-link — مقبول `NULL` |

**الـ Return Value:**
- `struct phy_device *` valid عند النجاح
- `ERR_PTR(-EINVAL)` لو الـ speed/duplex combination invalid (عبر `swphy_validate_state`)
- `ERR_PTR(-EPROBE_DEFER)` لو الـ virtual MDIO bus لسه متسجلش — ده مهم في probe ordering
- `ERR_PTR(-ENOSPC)` لو الـ 8 slots كلهم محجوزين
- `ERR_PTR(-ENOMEM)` من `get_phy_device` داخلياً

**Key Details:**
- الـ `link` field في الـ status بيتضبط على `true` تلقائياً داخل الـ register path — الـ fixed link دايماً up بعد التسجيل
- الـ `is_pseudo_fixed_link = true` بيخلي الـ phylib يتجاهل autoneg ويتعامل مع الـ PHY ده بشكل مختلف
- الـ slot allocation بتستخدم `test_and_set_bit` — thread-safe، ما تحتاجش external lock
- الـ caller مسؤول عن استدعاء `fixed_phy_unregister()` في الـ cleanup path بالظبط

**Caller Context:**
يُستدعى من driver `probe()` أو من `of_get_fixed_link_phydev()` في الـ phylink layer، في sleeping context.

**Pseudocode Flow:**

```
fixed_phy_register(status, np):
    swphy_validate_state(status)       → ERR_PTR(-EINVAL) if bad speed/duplex
    check fmb_mii_bus registered       → ERR_PTR(-EPROBE_DEFER) if not ready
    addr = fixed_phy_get_free_addr()   → ERR_PTR(-ENOSPC) if full
    fmb_fixed_phys[addr].status = *status
    fmb_fixed_phys[addr].status.link = true
    phy = get_phy_device(fmb_mii_bus, addr, false)
    phy->mdio.dev.of_node = np
    phy->is_pseudo_fixed_link = true
    phy_device_register(phy)           → on failure: cleanup all above
    return phy
```

---

#### `fixed_phy_register_100fd()`

```c
struct phy_device *fixed_phy_register_100fd(void);
```

**ما بتعمله:**
wrapper مختصر — بيبني `fixed_phy_status` ثابت بـ `SPEED_100` و`DUPLEX_FULL` وبيستدعي `fixed_phy_register()` مباشرة بدون DT node. مناسب جداً للـ loopback interfaces أو الـ virtual network devices اللي دايماً محتاجة fixed 100FD link.

**الـ Parameters:** لا يوجد.

**الـ Return Value:** نفس `fixed_phy_register()` — `phy_device *` أو `ERR_PTR`.

**Key Details:**
- الـ `pause` و`asym_pause` بياخدوا قيمة `false` بالـ default
- الـ stub في حالة `!CONFIG_FIXED_PHY` بترجع `ERR_PTR(-ENODEV)` مباشرة من غير overhead

**Caller Context:**
Drivers زي USB-to-Ethernet أو DSA ports أو virtual interfaces في الـ init path.

---

### Group 2: Cleanup — إلغاء التسجيل

---

#### `fixed_phy_unregister()`

```c
void fixed_phy_unregister(struct phy_device *phydev);
```

**ما بتعمله:**
بتعكس كل خطوة من خطوات `fixed_phy_register()` بالترتيب الصح — إزالة من الـ MDIO subsystem، تحرير الـ DT node reference، تحرير الـ bitmap slot، وأخيراً تحرير الـ memory.

**الـ Parameters:**

| Parameter | النوع | الشرح |
|---|---|---|
| `phydev` | `struct phy_device *` | الـ PHY اللي رجعته `fixed_phy_register()` |

**الـ Return Value:** `void` — مفيش error reporting.

**Key Details:**
- الترتيب: `phy_device_remove` → `of_node_put` → `fixed_phy_del` → `phy_device_free`
- لازم الـ MAC يعمل `phy_disconnect()` قبل الاستدعاء — لو الـ MAC لسه connected ممكن use-after-free
- الـ stub في حالة `!CONFIG_FIXED_PHY` هي empty inline — safe to call unconditionally

**Caller Context:**
`remove()` أو error path في `probe()` بعد `fixed_phy_register()` ناجح، sleeping context.

---

### Group 3: Runtime Configuration — ضبط السلوك أثناء التشغيل

---

#### `fixed_phy_set_link_update()`

```c
int fixed_phy_set_link_update(
    struct phy_device *phydev,
    int (*link_update)(struct net_device *,
                       struct fixed_phy_status *)
);
```

**ما بتعمله:**
بتركّب callback function على الـ fixed PHY. الـ callback ده بيتنادى في كل مرة الـ PHY state machine تعمل poll للـ link status (عبر `fixed_mdio_read`). جوا الـ callback، الـ driver يقدر يقرأ حالة الـ link الحقيقية من hardware (GPIO، I2C، SPI، switch register) ويحدّث `fixed_phy_status` accordingly.

**الـ Parameters:**

| Parameter | النوع | الشرح |
|---|---|---|
| `phydev` | `struct phy_device *` | الـ fixed PHY المستهدف |
| `link_update` | function pointer | callback — يأخذ `net_device *` وـ `fixed_phy_status *`، يعدّلها in-place |

**الـ Return Value:**
- `0` عند النجاح
- `-EINVAL` لو `phydev` أو الـ internal lookup فشل

**Key Details:**
- الـ callback بيتشغل في context الـ PHY state machine workqueue — يقدر يعمل sleeping operations زي I2C reads
- بيخزن كمان `phydev` في الـ internal `fixed_phy` struct عشان الـ callback يصل لـ `attached_dev`
- لو الـ callback رجع error، الـ `fixed_mdio_read` بيستكمل بالـ status القديم

```c
/* pattern نموذجي للـ callback */
static int my_link_update(struct net_device *dev,
                          struct fixed_phy_status *status)
{
    status->link   = gpio_get_value(LINK_GPIO);
    status->speed  = SPEED_1000;
    status->duplex = DUPLEX_FULL;
    return 0;
}
```

**Caller Context:**
بعد `fixed_phy_register()` وقبل `phy_connect()` في الـ probe path، مرة واحدة.

**Pseudocode Flow:**

```
fixed_phy_set_link_update(phydev, link_update):
    fp = fixed_phy_find(phydev->mdio.addr)
    if !fp → return -EINVAL
    fp->link_update = link_update
    fp->phydev = phydev
    return 0

/* لاحقاً، أثناء polling: */
fixed_mdio_read(bus, addr, reg):
    fp = fixed_phy_find(addr)
    if fp->link_update:
        fp->link_update(fp->phydev->attached_dev, &fp->status)
    return swphy_read_reg(reg, &fp->status)
```

---

#### `fixed_phy_change_carrier()`

```c
int fixed_phy_change_carrier(struct net_device *dev, bool new_carrier);
```

**ما بتعمله:**
بتسمح للـ MAC driver يغير حالة الـ carrier على الـ fixed PHY مباشرة من الـ `net_device` context. مفيدة لما الـ MAC نفسه قادر يكتشف carrier change (مثلاً interrupt خارجي) وعايز يعكسه على الـ fixed PHY بدون callback.

**الـ Parameters:**

| Parameter | النوع | الشرح |
|---|---|---|
| `dev` | `struct net_device *` | الـ network device اللي عليه الـ fixed PHY |
| `new_carrier` | `bool` | `true` = link up، `false` = link down |

**الـ Return Value:**
- `0` عند النجاح
- `-EINVAL` لو `dev->phydev` مش موجود أو مش fixed PHY

**Key Details:**
- التغيير بيحصل على `fp->status.link` مباشرة — في الـ poll cycle القادم، الـ phylib هيلاقي الـ status الجديد
- مفيش explicit lock — الـ `bool:1` write يُعتبر safe على الـ architectures الحديثة في الـ use case ده
- مختلفة عن `fixed_phy_set_link_update` — ده one-shot change، مش callback مستمر

**Caller Context:**
MAC driver interrupt handler أو workqueue، أو `ndo_*` callbacks، في أي context.

---

### ملاحظات معمارية عامة

```
MAC Driver
    │
    ├── fixed_phy_register(status, np)      ← probe time
    │       │
    │       └── creates phy_device on fixed_mdio_bus
    │
    ├── fixed_phy_set_link_update(phy, cb)  ← optional, probe time
    │
    ├── phy_connect(dev, phy, handler, iface)
    │       └── starts PHY state machine workqueue
    │
    │   [ Runtime: PHY workqueue polls ~every 1s ]
    │       └── mii_bus->read() → fixed_mdio_read()
    │               ├── calls link_update callback (if set)
    │               └── swphy_read_reg() → MII register value
    │                       └── phylib updates phydev state
    │                               └── MAC gets link change notification
    │
    ├── fixed_phy_change_carrier(dev, false) ← optional, runtime
    │       └── fp->status.link = false
    │
    ├── phy_disconnect(dev)
    │
    └── fixed_phy_unregister(phydev)        ← remove time
```

الـ `CONFIG_FIXED_PHY` guard بيضمن إن الـ stubs الـ `static inline` بتتكمبل دايماً حتى لو الـ subsystem مش enabled — بيخلي الـ driver code نظيف تماماً من `#ifdef` داخلية، والـ compiler بيحذف الـ dead code تلقائياً.
## Phase 5: دليل الـ Debugging الشامل

الـ **Fixed PHY** subsystem بيعمل emulation لـ MDIO bus وهمي — مفيش hardware PHY حقيقي، كل اللي بيحصل هو kernel يتظاهر إن في PHY عشان يكمل الـ network stack بشكل طبيعي. ده بيخلي الـ debugging مختلف: مفيش registers hardware تقراها، بس في state وهمي محتاج تتحقق منه.

---

### Software Level

#### 1. debugfs Entries

**الـ** `fixed-0` **هو الـ** MII bus ID المتسجل في الـ kernel:

```bash
# اعرف كل الـ MDIO buses المسجلة
ls /sys/bus/mdio_bus/devices/

# شوف الـ fixed PHY devices تحت الـ bus
ls /sys/bus/mdio_bus/devices/ | grep fixed

# PHY address بيكون 0-7 (NUM_FP=8)
# Example: fixed-0:00 يعني bus=fixed-0, addr=0
cat /sys/bus/mdio_bus/devices/fixed-0:00/uevent
```

```bash
# debugfs للـ MDIO (لو kernel compiled مع CONFIG_MDIO_DEVRES)
mount -t debugfs none /sys/kernel/debug 2>/dev/null || true
ls /sys/kernel/debug/mdio-bus/ 2>/dev/null
```

مفيش dedicated debugfs entries للـ fixed PHY نفسه — الـ state محفوظ في memory فقط (`fmb_fixed_phys[]` array).

#### 2. sysfs Entries

```bash
# حالة الـ PHY device الـ attached للـ net device
PHY_IFACE="eth0"   # غيّر للـ interface بتاعك

# الـ link state كما يراها الـ fixed PHY
cat /sys/class/net/${PHY_IFACE}/carrier
cat /sys/class/net/${PHY_IFACE}/speed
cat /sys/class/net/${PHY_IFACE}/duplex
cat /sys/class/net/${PHY_IFACE}/operstate

# معلومات الـ PHY device المربوط
cat /sys/class/net/${PHY_IFACE}/phydev/phy_id
cat /sys/class/net/${PHY_IFACE}/phydev/is_pseudo_fixed_link
# لو القيمة 1 → ده fixed PHY فعلاً

# الـ MDIO bus state
cat /sys/bus/mdio_bus/devices/fixed-0:00/state 2>/dev/null
```

```bash
# فحص كل الـ fixed PHY devices المسجلة
for dev in /sys/bus/mdio_bus/devices/fixed-0:*; do
    echo "=== $dev ==="
    cat "$dev/uevent" 2>/dev/null
done
```

#### 3. ftrace — Tracepoints والـ Events

```bash
# mount tracefs لو مش موجود
mount -t tracefs nodev /sys/kernel/tracing 2>/dev/null || true
cd /sys/kernel/tracing

# enable أحداث الـ MDIO bus (بيشمل fixed MDIO reads/writes)
echo 1 > events/mdio/enable

# تتبع calls على fixed_mdio_read و fixed_mdio_write
echo 'fixed_mdio_read' >> set_ftrace_filter
echo 'fixed_mdio_write' >> set_ftrace_filter
echo 'fixed_phy_change_carrier' >> set_ftrace_filter
echo function > current_tracer
echo 1 > tracing_on

# اعمل action (ping مثلاً) وبعدين اقرأ
cat trace | grep -E "fixed_mdio|fixed_phy"

# إيقاف
echo 0 > tracing_on
echo nop > current_tracer
```

```bash
# تتبع PHY state changes
echo 1 > events/net/netif_carrier_on/enable
echo 1 > events/net/netif_carrier_off/enable
echo 1 > tracing_on
cat trace_pipe &
# اعمل: ip link set eth0 up
```

#### 4. printk / Dynamic Debug

```bash
# تفعيل dynamic debug للـ fixed PHY module
echo 'module fixed_phy +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/net/phy/fixed_phy.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/net/phy/swphy.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug للـ PHY core
echo 'file drivers/net/phy/phy.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/net/phy/phy_device.c +p' > /sys/kernel/debug/dynamic_debug/control

# تابع الـ kernel log
dmesg -w | grep -iE "fixed.phy|mdio|phy_device"

# أو بـ journald
journalctl -k -f | grep -iE "fixed.phy|fmb|swphy"
```

```bash
# رفع loglevel مؤقتاً
echo 8 > /proc/sys/kernel/printk
# أو
dmesg -n 8
```

#### 5. Kernel Config Options للـ Debugging

| Config Option | الغرض |
|---|---|
| `CONFIG_FIXED_PHY` | لازم يكون `y` أو `m` عشان الـ subsystem يشتغل |
| `CONFIG_PHYLIB` | الـ base library — لازم موجود |
| `CONFIG_MDIO_DEVRES` | إضافة resource management للـ MDIO |
| `CONFIG_DYNAMIC_DEBUG` | تفعيل `pr_debug()` في runtime |
| `CONFIG_NET_PHY_SFPBUS` | لو في SFP modules متربطة |
| `CONFIG_NETDEVICES` | لازم للـ net_device integration |
| `CONFIG_DEBUG_KERNEL` | تفعيل كل debugging features |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "CONFIG_FIXED_PHY|CONFIG_PHYLIB|CONFIG_DYNAMIC_DEBUG"
# أو
grep -E "CONFIG_FIXED_PHY|CONFIG_PHYLIB" /boot/config-$(uname -r)
```

#### 6. أدوات خاصة بالـ Subsystem

```bash
# ethtool — الأداة الأساسية
ethtool eth0                    # speed, duplex, link
ethtool -i eth0                 # driver info + bus-info (هتشوف "fixed-0:00")
ethtool -S eth0                 # statistics
ethtool --show-phy-tunable eth0 # PHY tunables

# ip link
ip link show eth0
ip -s link show eth0

# mii-tool (legacy ولكن مفيد)
mii-tool -v eth0

# phytool (لو مثبت)
# phytool read eth0/0/1   # قراءة MII register من fixed PHY
# لاحظ: fixed MDIO write بيرجع 0 دايماً بدون effect
```

#### 7. جدول رسائل الخطأ الشائعة

| رسالة في dmesg / الـ return value | المعنى | الحل |
|---|---|---|
| `ERR_PTR(-ENODEV)` من `fixed_phy_register()` | `CONFIG_FIXED_PHY` مش enabled | فعّل الـ config وأعد build الـ kernel |
| `ERR_PTR(-EPROBE_DEFER)` | الـ `fmb_mii_bus` لسه متسجلش — race condition أثناء boot | الـ driver محتاج يعيد المحاولة لاحقاً — ده طبيعي |
| `ERR_PTR(-ENOSPC)` | كل الـ 8 slots الـ PHY ممتلئة (`NUM_FP=8`) | قلّل عدد الـ fixed PHYs أو زوّد `NUM_FP` |
| `ERR_PTR(-EINVAL)` | `swphy_validate_state()` رفض الـ status | تحقق إن speed=10/100/1000 و duplex=0/1 |
| `fixed_phy_find: -EINVAL` | phy_addr مش registered في `fixed_phy_ids` | الـ phydev مش من الـ fixed bus |
| `fixed_phy_find: -ENOENT` | address فيه bit set بس struct فاضي | bug في التسجيل — check `fixed_phy_ids` bitmap |
| `mdiobus_register failed` | فشل تسجيل الـ virtual MDIO bus | check dmesg لأسباب أعمق |
| `phy_device_register failed` | PHY device registration فشل | غالباً duplicate phy_addr |
| `netif_carrier_off` بدون سبب | الـ `link_update` callback بيرجع `link=false` | افحص الـ custom `link_update` function |

#### 8. Strategic Points لـ dump_stack() / WARN_ON()

نقاط استراتيجية لو بتعمل kernel debugging patch:

```c
/* في fixed_phy_register() — تحقق من الـ status قبل التسجيل */
static struct phy_device *fixed_phy_register(...)
{
    WARN_ON(!status);
    WARN_ON(status->speed != 10 &&
            status->speed != 100 &&
            status->speed != 1000);
    /* ... */
}

/* في fixed_mdio_read() — لو fp مش موجود وده مش متوقع */
static int fixed_mdio_read(struct mii_bus *bus, int phy_addr, int reg_num)
{
    struct fixed_phy *fp = fixed_phy_find(phy_addr);
    if (!fp) {
        /* ده طبيعي لأن phy_mask=~0 ممكن تجي reads لـ addresses مش registered */
        return 0xffff;
    }
    /* WARN لو link_update موجود بس attached_dev فاضي */
    WARN_ON(fp->link_update && !fp->phydev->attached_dev);
    /* ... */
}

/* في fixed_phy_change_carrier() — critical path */
int fixed_phy_change_carrier(struct net_device *dev, bool new_carrier)
{
    WARN_ON(!dev);
    if (!dev->phydev) {
        dump_stack(); /* مش المفروض يحصل بعد phy_connect */
        return -EINVAL;
    }
    /* ... */
}
```

---

### Hardware Level

#### 1. التحقق من State الـ Hardware مقابل الـ Kernel

الـ Fixed PHY مفيش ليه hardware حقيقي — هو **software emulation** خالص. لكن الـ "hardware" هنا هو:

- الـ MAC controller الـ attached للـ RGMII/SGMII interface
- السلك أو الـ SFP module المتصل فعلاً

```bash
# تحقق إن الـ MAC driver شايف الـ link
ethtool eth0 | grep -E "Link|Speed|Duplex"

# قارن مع ما الـ kernel يعتقده
cat /sys/class/net/eth0/carrier       # 1=up, 0=down
cat /sys/class/net/eth0/carrier_changes  # عدد مرات تغيير الـ link

# لو في SFP
ethtool eth0 | grep -i sfp
# أو
cat /sys/class/net/eth0/sfp_present 2>/dev/null
```

```bash
# تحقق إن الـ physical interface قادر يستقبل frames
ip -s link show eth0 | grep -A2 "RX\|TX"
# لو RX=0 مع link=1 → مشكلة hardware أو misconfiguration
```

#### 2. Register Dump Techniques

الـ Fixed PHY بيعمل emulate لـ MII registers في software عن طريق `swphy_read_reg()`. ممكن تقرأ القيم المحاكاة:

```bash
# قراءة MII registers الوهمية عن طريق ethtool
# PHY address = الـ slot number (0-7)
# مثال: PHY في slot 0 على bus fixed-0
ethtool -d eth0 2>/dev/null | head -40
# أو بـ mii-tool
mii-tool -v eth0

# phytool لقراءة MII registers مباشرة
# phytool read fixed-0/0/0    # BMCR register
# phytool read fixed-0/0/1    # BMSR register
# phytool read fixed-0/0/4    # ADVERTISE register
```

MII Registers المحاكاة من `swphy_read_reg()`:

| Register | رقم | القيمة المتوقعة |
|---|---|---|
| MII_BMCR | 0 | speed bits + duplex bit |
| MII_BMSR | 1 | `BMSR_LSTATUS` لو link=true |
| MII_PHYSID1 | 2 | `0` |
| MII_PHYSID2 | 3 | `0` |
| MII_ADVERTISE | 4 | pause bits + speed bits |
| MII_LPA | 5 | نفس MII_ADVERTISE (link partner = نفسه) |

```bash
# لو عندك /dev/mem والـ MAC له base address معروف
# devmem2 0xFE540000 w    # مثال لـ GMAC base address
# ده للـ MAC مش للـ fixed PHY نفسه
```

#### 3. Logic Analyzer / Oscilloscope Tips

الـ Fixed PHY بيعمل bypass للـ MDIO bus الحقيقية — مفيش MDIO signals على الـ wire:

```
+--------+      (no real MDIO)      +-----------+
|  MAC   | <-- software emulation -> | fixed PHY |
| driver |                           | (kernel)  |
+--------+                           +-----------+
    |
    | RGMII / SGMII / MII  <- ده اللي تقيسه فعلاً
    |
+--------+
| switch |  أو  | media converter |  أو  | SFP |
+--------+
```

**ما تقيسه بالـ oscilloscope:**

```
RGMII Interface:
- TXCLK: 125MHz (1Gbps) / 25MHz (100Mbps) / 2.5MHz (10Mbps)
- RXCLK: نفس الـ TXCLK من الطرف الثاني
- TXD[3:0] + TXEN: data في الـ TX direction
- RXD[3:0] + RXDV: data في الـ RX direction

لو RXCLK مش موجود -> المشكلة في الـ switch/media converter مش في الـ fixed PHY
لو TXCLK موجود بس RXCLK مش موجود -> الطرف الثاني مش شغال
```

```bash
# تحقق من الـ clock source في Device Tree
grep -r "clock-frequency\|clocks\|rgmii" /sys/firmware/devicetree/base/ 2>/dev/null | head -10
```

#### 4. مشاكل Hardware شائعة ← أنماط في الـ Kernel Log

| المشكلة الـ hardware | pattern في dmesg | التشخيص |
|---|---|---|
| الـ switch/converter مش powered | `netif_carrier_off eth0` عند boot | الـ link_update callback شايف link=false |
| Wrong speed negotiation | `eth0: renamed from...` ثم لا traffic | Speed في DT مختلف عن الـ physical interface |
| RGMII delay misconfiguration | `eth0: fifo errors` أو RX drops عالية | تحتاج `rgmii-id` أو `rgmii-rxid` في DT |
| الـ MAC مش بيبعث clocks | لا traffic رغم carrier=1 | افحص MAC driver initialization |
| SFP module غلط | `sfp: no support for SFP module` | الـ module مش في قائمة الـ supported modules |

```bash
# مراقبة errors في real-time
watch -n1 'ip -s link show eth0 | grep -A4 "RX\|TX"'

# تفاصيل أكثر
ethtool -S eth0 | grep -iE "error|drop|miss|fifo|overflow"
```

#### 5. Device Tree Debugging

الـ Fixed PHY بيعتمد على الـ Device Tree لربط الـ MAC بالـ PHY الوهمي:

```bash
# قراءة الـ DT للـ fixed PHY node
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "fixed-link\|fixed-phy"

# أو مباشرة
find /sys/firmware/devicetree/base -name "fixed-link" -exec sh -c \
    'echo "=== $1 ==="; ls "$1"; cat "$1/speed" 2>/dev/null | od -An -tu4' _ {} \; 2>/dev/null
```

**DT صح لـ fixed PHY:**

```dts
/* في الـ MAC node */
&gmac0 {
    phy-mode = "rgmii-id";

    fixed-link {
        speed = <1000>;    /* 10 / 100 / 1000 */
        full-duplex;       /* غيابه = half duplex */
        pause;             /* optional */
        asym-pause;        /* optional */
    };
};
```

```bash
# تحقق إن الـ speed في DT صح
speed_val=$(cat /sys/firmware/devicetree/base/soc/ethernet@.../fixed-link/speed 2>/dev/null | od -An -tu4 | tr -d ' ')
echo "DT speed: ${speed_val} Mbps"

# قارن مع ما الـ kernel يستخدمه
ethtool eth0 | grep Speed

# تحقق من full-duplex property
ls /sys/firmware/devicetree/base/soc/ethernet@.../fixed-link/ | grep duplex
```

```bash
# الـ of_node المسجل في الـ PHY device
ls -la /sys/bus/mdio_bus/devices/fixed-0:00/of_node 2>/dev/null
# لو symlink موجود → الـ DT node مربوط صح
# لو مش موجود → np=NULL أُعطي لـ fixed_phy_register()
```

---

### Practical Commands — جاهزة للنسخ

#### فحص سريع شامل

```bash
#!/bin/bash
# fixed-phy-debug.sh — سكريبت تشخيص سريع

IFACE="${1:-eth0}"
echo "=== Fixed PHY Debug for ${IFACE} ==="

echo -e "\n[1] Link State"
cat /sys/class/net/${IFACE}/carrier 2>/dev/null && \
    cat /sys/class/net/${IFACE}/speed 2>/dev/null && \
    cat /sys/class/net/${IFACE}/duplex 2>/dev/null

echo -e "\n[2] ethtool info"
ethtool ${IFACE} 2>/dev/null | grep -E "Speed|Duplex|Link|Port"
ethtool -i ${IFACE} 2>/dev/null | grep -E "driver|bus-info|version"

echo -e "\n[3] Fixed PHY devices registered"
ls /sys/bus/mdio_bus/devices/ 2>/dev/null | grep fixed

echo -e "\n[4] is_pseudo_fixed_link"
cat /sys/class/net/${IFACE}/phydev/is_pseudo_fixed_link 2>/dev/null || echo "N/A"

echo -e "\n[5] Recent dmesg"
dmesg | grep -iE "fixed.phy|${IFACE}|mdio" | tail -20

echo -e "\n[6] RX/TX stats"
ip -s link show ${IFACE} | grep -A2 "RX\|TX"

echo -e "\n[7] DT fixed-link"
find /sys/firmware/devicetree/base -name "fixed-link" 2>/dev/null | \
    while read d; do
        echo "  Found: $d"
        ls "$d/"
    done
```

#### تفعيل كامل للـ Tracing

```bash
#!/bin/bash
# enable-fixed-phy-trace.sh

T=/sys/kernel/tracing
echo 0 > $T/tracing_on
echo nop > $T/current_tracer
echo > $T/trace

# function tracer للـ fixed PHY functions
echo function > $T/current_tracer
echo 'fixed_mdio_read
fixed_mdio_write
fixed_phy_change_carrier
fixed_phy_set_link_update
fixed_phy_register
fixed_phy_unregister' > $T/set_ftrace_filter

# أحداث الـ network
echo 1 > $T/events/net/netif_carrier_on/enable
echo 1 > $T/events/net/netif_carrier_off/enable

echo 1 > $T/tracing_on
echo "Tracing enabled. Press Ctrl+C to stop."
cat $T/trace_pipe
```

#### قراءة MII Registers الوهمية

```bash
#!/bin/bash
# read-fixed-phy-regs.sh
# يحتاج phytool أو mii-tool

IFACE="${1:-eth0}"
BUS_INFO=$(ethtool -i ${IFACE} 2>/dev/null | grep bus-info | awk '{print $2}')
# BUS_INFO مثلاً: "fixed-0:00"
ADDR=$(echo $BUS_INFO | cut -d: -f2 | sed 's/^0*//')

echo "PHY bus: $BUS_INFO, addr: $ADDR"
echo ""
echo "MII Register Map (software-emulated):"
printf "%-20s %-6s %-8s\n" "Register" "Num" "Value"
printf "%-20s %-6s %-8s\n" "--------" "---" "-----"

# قراءة عبر ethtool -d
ethtool -d ${IFACE} 2>/dev/null | head -20
```

#### مراقبة carrier changes

```bash
# مراقبة مستمرة لتغيرات الـ link
IFACE="eth0"
PREV=$(cat /sys/class/net/${IFACE}/carrier_changes)
echo "Monitoring carrier changes on ${IFACE}..."
while true; do
    CURR=$(cat /sys/class/net/${IFACE}/carrier_changes 2>/dev/null)
    if [ "$CURR" != "$PREV" ]; then
        TS=$(date +"%T.%N")
        STATE=$(cat /sys/class/net/${IFACE}/carrier 2>/dev/null)
        echo "$TS: carrier_changes=$CURR, carrier=$STATE"
        PREV=$CURR
        dmesg | tail -5
    fi
    sleep 0.1
done
```

#### تفسير الـ Output

```
# مثال output من ethtool eth0 مع fixed PHY:
Settings for eth0:
    Supported ports: [ ]                  # <- فاضي = fixed PHY (مفيش hardware)
    Supported link modes:   Not reported  # <- مش بيعلن modes
    Speed: 1000Mb/s                       # <- من fixed_phy_status.speed
    Duplex: Full                          # <- من fixed_phy_status.duplex
    Port: MII                             # <- MII emulated
    PHYAD: 0                              # <- slot number في fmb_fixed_phys[]
    Transceiver: internal
    Link detected: yes                    # <- من fixed_phy_status.link

# مثال output من ethtool -i eth0:
driver: macb                             # <- اسم الـ MAC driver
version: ...
bus-info: fixed-0:00                     # <- "fixed-0" = الـ virtual MDIO bus
                                         #   "00" = PHY address (slot 0)
# لو شفت "fixed-0:XX" في bus-info -> ده يأكد إن الـ interface بتستخدم fixed PHY
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على AM62x — الـ Ethernet Switch مش بيظهر

#### العنوان
Ethernet port على industrial gateway مبني على AM62x مش بيرفع link بعد boot.

#### السياق
فريق تطوير بيعمل bring-up لـ industrial IoT gateway بيستخدم **TI AM62x** SoC. الـ board بيها internal switch متصل بـ MAC عن طريق **RGMII** لكن مفيش PHY chip فعلي — الـ switch بيدير الـ link negotiation هو نفسه. المنتج بيتباع في خطوط تصنيع وبيحتاج يبعت بيانات sensor فوراً بعد الـ boot.

#### المشكلة
بعد boot، الـ network interface `eth0` بتظهر في `ip link` لكن state بتبقى `NO-CARRIER` طول الوقت حتى لو الكابل متوصل:

```bash
$ ip link show eth0
2: eth0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default
    link/ether aa:bb:cc:dd:ee:ff brd ff:ff:ff:ff:ff:ff
```

الـ `dmesg` مفيش فيه أي log عن PHY.

#### التحليل
الـ driver بيحاول يعمل `of_phy_connect()` ويدور على PHY node في الـ Device Tree. لما بيفضل ملاقيش PHY حقيقي، الـ MAC مش بيعمل `netif_carrier_on()` أبداً.

الحل الصح: استخدام **fixed PHY** من `phy_fixed.h`. الـ `fixed_phy_register()` بتسجل pseudo-PHY بـ status ثابت:

```c
struct fixed_phy_status status = {
    .speed  = 1000,   /* 1Gbps */
    .duplex = DUPLEX_FULL,
    .link   = 1,      /* always up */
};
phydev = fixed_phy_register(&status, np);
```

بس المشكلة إن الـ DT node للـ MAC مش فيه `fixed-link` property خالص.

#### الحل

**الخطوة 1 — تعديل الـ Device Tree:**

```dts
/* am62x-industrial-gateway.dts */
&cpsw3g {
    status = "okay";

    ethernet-ports {
        #address-cells = <1>;
        #size-cells = <0>;

        cpsw_port1: port@1 {
            reg = <1>;
            ti,mac-only;
            label = "port1";
            phy-mode = "rgmii-id";

            fixed-link {
                speed = <1000>;
                full-duplex;
            };
        };
    };
};
```

**الخطوة 2 — التحقق بعد boot:**

```bash
$ dmesg | grep -i fixed
[    2.341] fixed-0:00: attached PHY driver [Fixed PHY] (mii_bus:phy_addr=fixed-0:00)

$ ip link show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP
```

#### الدرس المستفاد
لما يكون فيه MAC متصل بـ switch أو SFP module بدون PHY MDIO حقيقي، لازم تستخدم **fixed PHY** عن طريق `fixed_phy_register()` أو `fixed-link` DT property. الـ `phy_fixed.h` هو الـ interface الوحيد للـ kernel لتسجيل الـ pseudo-PHY دي.

---

### السيناريو 2: Android TV Box على Allwinner H616 — الـ 100Mbps Ethernet بتكسر

#### العنوان
الـ Ethernet على TV box بيشتغل لحظات وبعدين يوقف — المشكلة في wrong speed negotiation.

#### السياق
منتج **Android TV box** مبني على **Allwinner H616**. الـ board بيها **RTL8201** PHY متصل بـ RMII. بس في revision تانية من الـ PCB، المصمم قرر يوصل الـ MAC بـ **internal 100M switch** زي H3 بدون PHY chip خارجي، علشان يوفر تكلفة. الـ software team مش عارفين.

#### المشكلة
الـ Ethernet بيشتغل لأول 30 ثانية وبعدين الـ link بتسقط. الـ `ethtool` بيبين:

```bash
$ ethtool eth0
Speed: Unknown!
Duplex: Unknown!
Link detected: no
```

الـ `dmesg` بيبين:
```
sunxi-gmac 5020000.ethernet eth0: Link is Down
```

في اللوح الجديد مفيش MDIO signals بالأساس، فالـ driver بيحاول يـ poll PHY وبيفشل.

#### التحليل
الـ `sunxi-gmac` driver بيستخدم `of_phy_connect()` عادي. لما الـ PHY chip مش موجود على الـ MDIO bus، الـ connection بتتعمل أول مرة غلط أو بتفشل خالص، فبييجي timeout وبيعمل `netif_carrier_off()`.

الحل: استخدام `fixed_phy_register_100fd()` اللي موجودة في `phy_fixed.h`:

```c
/* from phy_fixed.h */
struct phy_device *fixed_phy_register_100fd(void);
```

الـ function دي بتسجل fixed PHY بـ:
- Speed: 100 Mbps
- Duplex: Full
- Link: always up

#### الحل

**الخطوة 1 — تعديل الـ DTS لـ H616:**

```dts
/* h616-tv-box-rev2.dts */
&emac0 {
    pinctrl-names = "default";
    pinctrl-0 = <&rmii_pins>;
    phy-mode = "rmii";
    allwinner,rx-delay-ps = <0>;
    allwinner,tx-delay-ps = <0>;
    status = "okay";

    fixed-link {
        speed = <100>;
        full-duplex;
    };
};
```

**الخطوة 2 — أو من الـ driver مباشرة لو DT مش كافي:**

```c
/* في الـ probe function */
if (!of_get_child_by_name(np, "fixed-link") &&
    !of_parse_phandle(np, "phy-handle", 0)) {
    /* no real PHY, register fixed 100Mbps full-duplex */
    priv->phydev = fixed_phy_register_100fd();
    if (IS_ERR(priv->phydev))
        return PTR_ERR(priv->phydev);
}
```

**التحقق:**

```bash
$ ethtool eth0
Speed: 100Mb/s
Duplex: Full
Link detected: yes
```

#### الدرس المستفاد
الـ `fixed_phy_register_100fd()` هي shortcut مريحة للـ 100BASE-TX Full Duplex اللي هو الـ common case في الـ embedded boards بدون PHY. لو الـ PCB revision اتغيرت وشيلت الـ PHY chip، الحل السريع هو الـ fixed PHY مش إعادة كتابة الـ driver.

---

### السيناريو 3: Automotive ECU على i.MX8 — الـ Link Status Callback محتاج

#### العنوان
ECU على i.MX8 محتاج يعرف link status من automotive PHY خارجي متوصل بـ SPI مش MDIO.

#### السياق
**Automotive ECU** بيستخدم **i.MX8M Plus** لمعالجة بيانات الـ camera والـ sensor fusion. في بعض variants، بيستخدموا **TJA1100** automotive PHY اللي بيتحكم فيه عن طريق SPI مش MDIO. المنتج لازم يكشف فصل الكابل في أقل من 100ms.

#### المشكلة
المهندس سجل fixed PHY وحط `link = true` ثابت. الـ interface بتفضل `UP` حتى لو الكابل متفصل. الـ stack العلوي مش بيعرف إن الـ link راح، وبيفضل يبعت packets في الفراغ:

```bash
$ ip link show eth0
eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>  # غلط — الكابل متفصل!
```

#### التحليل
الـ `phy_fixed.h` بتوفر:

```c
extern int fixed_phy_set_link_update(struct phy_device *phydev,
            int (*link_update)(struct net_device *,
                               struct fixed_phy_status *));
```

الـ function دي بتسمح للـ driver إنه يسجل **callback** بيتعمل call كل مرة الـ PHY state machine بيحتاج يعرف الـ current link status. الـ callback بياخد `struct fixed_phy_status *` ويملاه بالـ actual status اللي بيجي من SPI.

#### الحل

**الخطوة 1 — الـ DTS:**

```dts
/* imx8mp-automotive-ecu.dts */
&fec1 {
    phy-mode = "rgmii";
    status = "okay";

    fixed-link {
        speed = <100>;
        full-duplex;
    };
};
```

**الخطوة 2 — الـ link_update callback في الـ driver:**

```c
#include <linux/phy_fixed.h>
#include <linux/spi/spi.h>

/* read TJA1100 status register over SPI */
static int tja1100_link_update(struct net_device *dev,
                               struct fixed_phy_status *status)
{
    struct tja1100_priv *priv = netdev_priv(dev);
    u8 reg;

    reg = tja1100_spi_read(priv->spi, TJA1100_BASIC_STATUS);

    status->link   = !!(reg & TJA1100_LINK_STATUS);
    status->speed  = 100;
    status->duplex = DUPLEX_FULL;
    status->pause  = 0;

    return 0;
}

static int tja1100_probe(struct spi_device *spi)
{
    struct fixed_phy_status fstatus = {
        .speed  = 100,
        .duplex = DUPLEX_FULL,
        .link   = 1,
    };
    struct device_node *np = spi->dev.of_node;
    struct phy_device *phydev;

    /* register fixed PHY as placeholder */
    phydev = fixed_phy_register(&fstatus, np);
    if (IS_ERR(phydev))
        return PTR_ERR(phydev);

    /* override static link with dynamic SPI-based callback */
    fixed_phy_set_link_update(phydev, tja1100_link_update);

    priv->phydev = phydev;
    return 0;
}
```

**التحقق:**

```bash
# simulate link down via SPI register write
$ spi-tool write TJA1100_BASIC_STATUS 0x00

$ ip link show eth0
eth0: <BROADCAST,MULTICAST> state DOWN  # carrier dropped correctly
```

#### الدرس المستفاد
الـ `fixed_phy_set_link_update()` هي الـ bridge بين الـ fixed PHY abstraction والـ real hardware status. بدلها الـ fixed PHY يبقى دايماً `link=1` وده مش صح في automotive حيث detection الـ cable disconnect حيوي للـ safety.

---

### السيناريو 4: STM32MP1 IoT Sensor Hub — الـ CONFIG_FIXED_PHY مش enabled

#### العنوان
الـ build بيـ compile بدون error بس الـ network مش بيشتغل على custom STM32MP1 board.

#### السياق
فريق بيطور **IoT sensor hub** على **STM32MP157** بيجمع بيانات من عشرات الـ I2C sensors وبيبعتها على الـ cloud عن طريق Ethernet. الـ board مش فيها PHY chip — المصمم استخدم **RMII to fiber** module اللي بيدير هو الـ link. الـ `fixed-link` DT property موجودة صح.

#### المشكلة
الـ build بيكمل من غير أي error، بس على الـ target:

```bash
$ ip addr add 192.168.1.10/24 dev eth0
RTNETLINK answers: Network is down
```

والـ driver مش بيـ register أي PHY. الـ probe بيرجع `-ENODEV`.

#### التحليل
في الـ `phy_fixed.h` الـ guard الأساسي:

```c
#if IS_ENABLED(CONFIG_FIXED_PHY)
/* real implementations */
struct phy_device *fixed_phy_register(...);
#else
/* stub that always returns ERR_PTR(-ENODEV) */
static inline struct phy_device *
fixed_phy_register(const struct fixed_phy_status *status,
                   struct device_node *np)
{
    return ERR_PTR(-ENODEV);
}

static inline struct phy_device *fixed_phy_register_100fd(void)
{
    return ERR_PTR(-ENODEV);
}

static inline void fixed_phy_unregister(struct phy_device *phydev)
{
    /* intentional no-op */
}
#endif
```

لما `CONFIG_FIXED_PHY` مش enabled في الـ `.config`، كل الـ functions بتتحول لـ stubs بتـ return `-ENODEV`. الـ driver بياخد الـ error ده وبيفشل بصمت. الـ compile ينجح لأن الـ stubs موجودين.

#### الحل

**الخطوة 1 — check الـ config:**

```bash
$ grep CONFIG_FIXED_PHY /path/to/build/.config
# CONFIG_FIXED_PHY is not set   # هنا المشكلة
```

**الخطوة 2 — enable في الـ defconfig:**

```bash
$ cd linux/
$ make menuconfig
# Device Drivers -> Network device support -> PHY Device support
# [*] FIXED PHY (CONFIG_FIXED_PHY)

# أو مباشرة:
$ echo "CONFIG_FIXED_PHY=y" >> arch/arm/configs/stm32mp1_custom_defconfig
$ make stm32mp1_custom_defconfig
```

**الخطوة 3 — rebuild وتأكيد:**

```bash
$ grep CONFIG_FIXED_PHY .config
CONFIG_FIXED_PHY=y

$ dmesg | grep fixed
[    1.823] fixed-0:00: attached PHY driver [Fixed PHY]
```

**الخطوة 4 — منع المشكلة في المستقبل باستخدام Kconfig dependency:**

```kconfig
config STM32MP1_FIBER_ETH
    tristate "STM32MP1 RMII-to-Fiber Ethernet driver"
    depends on STMMAC_ETH
    select FIXED_PHY          /* force-enable عشان الـ driver يشتغل */
    help
      Driver for STM32MP1 boards with RMII-to-fiber module and no PHY chip.
```

#### الدرس المستفاد
الـ `#if IS_ENABLED(CONFIG_FIXED_PHY)` guard في `phy_fixed.h` مقصود — بيخلي الـ compile ينجح دايماً لكن الـ runtime بيفشل بشكل واضح. الحل الصح هو `select FIXED_PHY` في `Kconfig` للـ driver اللي يعتمد عليه، مش تتوقع إن الـ user يـ enable المهم ده manually.

---

### السيناريو 5: RK3562 Custom Board Bring-Up — Kernel Panic عند driver unload

#### العنوان
Kernel panic عند unload الـ module على RK3562 development board بسبب dangling PHY pointer.

#### السياق
فريق bring-up بيطور driver لـ custom network module على **RK3562** لجهاز HMI صناعي. الـ module بيشتغل على `rmii` بدون PHY خارجي. الـ driver بيتـ load وبيتـ unload كتير أثناء التطوير.

#### المشكلة
أول `rmmod` بعد اشتغال الـ network:

```
BUG: kernel NULL pointer dereference at 0000000000000058
Call trace:
  phy_state_machine+0x3c/0x1c0
  process_one_work+0x174/0x408
```

الـ PHY state machine لسه شغال على `phydev` اللي الـ driver مسح الـ resources بتاعته من غير ما يـ unregister الـ fixed PHY.

#### التحليل
الـ `phy_fixed.h` بتعرف:

```c
extern void fixed_phy_unregister(struct phy_device *phydev);
```

الـ function دي لازم تتعمل call في الـ `remove()` قبل ما الـ driver يعمل free لأي resource. لو اتعملتش، الـ PHY state machine (workqueue) بيفضل بيـ poll على pointer لـ struct اتحذف.

#### الحل

**الـ driver الغلط:**

```c
/* BAD: forgot to unregister fixed PHY */
static int rk3562_eth_remove(struct platform_device *pdev)
{
    struct rk3562_eth *priv = platform_get_drvdata(pdev);

    netif_stop_queue(priv->netdev);
    free_irq(priv->irq, priv);
    /* phydev dangling! PHY workqueue still running -> panic */
    return 0;
}
```

**الحل الصح مع الترتيب الصح:**

```c
#include <linux/phy_fixed.h>

static int rk3562_eth_remove(struct platform_device *pdev)
{
    struct rk3562_eth *priv = platform_get_drvdata(pdev);

    /* 1. stop network queue first */
    netif_stop_queue(priv->netdev);

    /* 2. disconnect PHY — stops the state machine workqueue */
    phy_disconnect(priv->phydev);

    /* 3. unregister fixed PHY — frees the pseudo-PHY resources */
    fixed_phy_unregister(priv->phydev);

    /* 4. now safe to free IRQ and other resources */
    free_irq(priv->irq, priv);
    unregister_netdev(priv->netdev);

    return 0;
}
```

**التحقق بدون panic:**

```bash
$ insmod rk3562_eth.ko
$ ip link set eth0 up
$ rmmod rk3562_eth     # بدون kernel panic

$ dmesg | tail -5
[  42.100] rk3562-eth: PHY disconnected
[  42.102] rk3562-eth: fixed PHY unregistered
[  42.104] rk3562-eth: removed successfully
```

#### الدرس المستفاد
الـ `fixed_phy_unregister()` مش optional. أي driver بيعمل `fixed_phy_register()` **لازم** يعمل `fixed_phy_unregister()` في الـ cleanup path. الترتيب الصح دايماً: `netif_stop_queue` ← `phy_disconnect` ← `fixed_phy_unregister` ← باقي الـ resources. أي حاجة تانية بتودي kernel panic في اللحظة اللي الـ PHY state machine بتحاول تـ access freed memory.
## Phase 7: مصادر ومراجع

### توثيق الـ Kernel الرسمي

| المصدر | الرابط | الوصف |
|--------|--------|-------|
| PHY Abstraction Layer | [docs.kernel.org](https://docs.kernel.org/networking/phy.html) | الوثيقة الرسمية للـ PAL — بيشرح فيها الـ fixed PHY وازاي بيتسجل |
| phylink documentation | [docs.kernel.org](https://docs.kernel.org/networking/sfp-phylink.html) | بيشرح الـ fixed link mode جوا phylink وعلاقته بالـ fixed PHY |
| PHY subsystem (driver-api) | [docs.kernel.org](https://docs.kernel.org/driver-api/phy/phy.html) | الـ generic PHY framework من جهة الـ driver API |
| phy.txt (قديم) | [kernel.org](https://www.kernel.org/doc/Documentation/networking/phy.txt) | النسخة القديمة من التوثيق، مفيدة لفهم التاريخ |

---

### مقالات LWN.net

**الـ** LWN هو المرجع الأهم لتتبع تطور الـ subsystems في الـ kernel.

| المقال | الرابط | الأهمية |
|--------|--------|---------|
| PAL: Support of the fixed PHY | [lwn.net/Articles/188485](https://lwn.net/Articles/188485/) | **أهم مقال** — بيشرح patch فيتالي بوردوج اللي أضاف الـ fixed PHY لأول مرة، وازاي بيخلي الـ boards اللي معنهاش hardware PHY تستخدم الـ PAL |
| PHY framework | [lwn.net/Articles/564188](https://lwn.net/Articles/564188/) | بيتكلم عن تطور الـ PHY framework بشكل عام |
| Generic PHY Framework | [lwn.net/Articles/544964](https://lwn.net/Articles/544964/) | الـ generic PHY framework — مختلف عن الـ fixed PHY بس مكمّله |
| drivers: phy: add generic PHY framework | [lwn.net/Articles/516636](https://lwn.net/Articles/516636/) | الـ patch الأصلية للـ generic PHY framework |
| net: phy: Add support for rate adaptation | [lwn.net/Articles/901880](https://lwn.net/Articles/901880/) | تطورات حديثة في الـ PHY layer |

---

### نقاشات الـ Mailing List

الـ mailing list بيكشف تاريخ التطور والقرارات التصميمية.

| النقاش | الرابط | الموضوع |
|--------|--------|---------|
| fixed_phy_unregister() patch | [lore.kernel.org](https://lore.kernel.org/lkml/55155C2B.9020507@list.ru/) | إضافة `fixed_phy_unregister()` كـ counterpart لـ `fixed_phy_register()` في 2015 |
| RFC: extend fixed driver with fixed_phy_register() | [patchwork.kernel.org](https://patchwork.kernel.org/patch/2854621/) | الـ patch الأصلي اللي عرّف `fixed_phy_register()` وربطها بالـ device tree |
| net: phy: export fixed_phy_register() | [patchwork.ozlabs.org](https://patchwork.ozlabs.org/comment/925513/) | نقاش تصدير الـ symbol للـ modules — ظهرت المشكلة مع bcmgenet |
| PHY fixed driver: rework release path | [lists.archive.carbon60.com](https://lists.archive.carbon60.com/linux/kernel/797805) | إصلاح مسار الـ release وتحديث ترميز الـ `phy_id` |
| fixed PHY Device Tree usage | [netdev.vger.kernel.narkive.com](https://netdev.vger.kernel.narkive.com/Qnet05PF/fixed-phy-device-tree-usage) | نقاش استخدام الـ fixed PHY مع الـ Device Tree |
| fixed link OF binding patch | [lists.infradead.org](http://lists.infradead.org/pipermail/linux-arm-kernel/2014-May/257227.html) | الـ binding الأصلي للـ fixed link في الـ device tree |

---

### الكود المصدري على GitHub

| الملف | الرابط | الوصف |
|-------|--------|-------|
| `include/linux/phy_fixed.h` | [github.com/torvalds/linux](https://github.com/torvalds/linux/blob/master/include/linux/phy_fixed.h) | الـ header الرئيسي — نقطة البداية |
| `drivers/net/phy/fixed_phy.c` | [github.com/torvalds/linux](https://github.com/torvalds/linux/blob/master/drivers/net/phy/fixed_phy.c) | الـ implementation الكاملة |
| `drivers/net/phy/phy_device.c` | [github.com/torvalds/linux](https://github.com/torvalds/linux/blob/master/drivers/net/phy/phy_device.c) | الـ phy_device lifecycle |

---

### مصادر kernelnewbies.org

بيفيد لمعرفة إيه اللي اتغير في كل إصدار kernel.

| الصفحة | الرابط | ما يخص الـ fixed PHY |
|--------|--------|---------------------|
| Linux 5.10 Changes | [kernelnewbies.org/Linux_5.10](https://kernelnewbies.org/Linux_5.10) | تغييرات في الـ networking/PHY layer |
| Linux 4.9 Changes | [kernelnewbies.org/Linux_4.9](https://kernelnewbies.org/Linux_4.9) | إصلاحات في الـ fixed PHY |
| Linux 5.14 Changes | [kernelnewbies.org/Linux_5.14](https://kernelnewbies.org/Linux_5.14) | تحسينات في الـ PHY subsystem |
| Linux 6.2 Changes | [kernelnewbies.org/Linux_6.2](https://kernelnewbies.org/Linux_6.2) | تطورات حديثة |

---

### elinux.org

| الصفحة | الرابط | الوصف |
|--------|--------|-------|
| Generic PHY Framework Overview (PDF) | [elinux.org](https://elinux.org/images/1/1c/Abraham--generic_phy_framework_an_overview.pdf) | عرض تقديمي شامل عن الـ PHY framework في الـ embedded Linux |

---

### كتب مرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصول المهمة**: Chapter 17 — Network Drivers
- بيشرح الـ `net_device` structure وحياة الـ network interface
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)
- **الأهمية للـ fixed PHY**: فهم الـ `net_device` اللي بيتمرر لـ `fixed_phy_set_link_update()` و`fixed_phy_change_carrier()`

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصول المهمة**: Chapter 17 — Devices and Modules
- بيشرح نموذج الـ device في الـ kernel وكيف تتعامل مع الـ subsystems
- **الأهمية**: فهم `IS_ENABLED(CONFIG_FIXED_PHY)` والـ Kconfig system

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصول المهمة**: Chapter 14 — Networking
- بيتكلم عن الـ embedded networking وحالات استخدام الـ fixed PHY بالتفصيل
- **الأهمية**: بيشرح السيناريوهات العملية — SoC مدمج بـ MAC بدون hardware PHY

#### Understanding Linux Network Internals — Christian Benvenuti
- الأعمق في شرح الـ networking stack
- Chapter 18 بيتكلم عن الـ device drivers وعلاقتها بالـ PHY layer

---

### مسارات التوثيق داخل الـ Kernel Source

```
Documentation/networking/phy.rst          ← الوثيقة الرئيسية للـ PHY layer
Documentation/networking/sfp-phylink.rst  ← phylink وعلاقته بالـ fixed PHY
Documentation/devicetree/bindings/net/
    ethernet-phy.yaml                     ← DT bindings للـ PHY بشكل عام
    fixed-link.yaml                       ← DT binding للـ fixed link
```

---

### كلمات البحث للمزيد

لو عايز تكمل بحثك، استخدم الكلمات دي:

```
# بحث عام
"fixed PHY" linux kernel
"fixed_phy_register" site:lore.kernel.org
"CONFIG_FIXED_PHY" linux kernel networking

# Device Tree
"fixed-link" linux device tree ethernet
"fixed-link" yaml ethernet DT binding linux

# phylink integration
"phylink" "fixed link" linux kernel
"pl_fixed_link" linux kernel source

# تاريخ التطور
"fixed_phy_add" linux kernel history
"Vitaly Bordug" fixed PHY linux patch
"MDIO bus" "fixed" linux embedded

# مشاكل شائعة
"fixed_phy_register" undefined module
"fixed PHY" carrier change linux
```

---

### ملخص أهم المراجع بالأولوية

| الأولوية | المرجع | السبب |
|----------|--------|-------|
| 1 | [LWN: PAL fixed PHY](https://lwn.net/Articles/188485/) | الأصل التاريخي — بيفسر "ليه" اتعمل |
| 2 | [docs.kernel.org/networking/phy.html](https://docs.kernel.org/networking/phy.html) | التوثيق الرسمي الحالي |
| 3 | [fixed_phy.c على GitHub](https://github.com/torvalds/linux/blob/master/drivers/net/phy/fixed_phy.c) | الـ implementation الكاملة |
| 4 | [lore.kernel.org: fixed_phy_unregister](https://lore.kernel.org/lkml/55155C2B.9020507@list.ru/) | فهم resource management |
| 5 | [phylink docs](https://docs.kernel.org/networking/sfp-phylink.html) | الاتجاه الحديث للـ fixed links |
## Phase 8: Writing simple module

### الـ function المختارة: `fixed_phy_change_carrier`

**الـ function دي** exported وبتتحكم في حالة الـ carrier (link up/down) لأي `net_device` مربوط بـ fixed PHY. ده مناسب لأننا نعمل عليها **kprobe** نشوف مين بيطلبها وبياخد أيه قرار.

---

### الـ module كامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * fixed_phy_carrier_probe.c
 * Intercept fixed_phy_change_carrier() calls via kprobe
 * and log the net_device name + requested carrier state.
 */
#include <linux/module.h>       /* module_init / module_exit / MODULE_* */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe, ... */
#include <linux/netdevice.h>    /* struct net_device, netdev_name() */
#include <linux/kernel.h>       /* pr_info() */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Arabic Kernel Docs Project");
MODULE_DESCRIPTION("kprobe on fixed_phy_change_carrier to trace carrier changes");

/* ------------------------------------------------------------------ */
/* kprobe pre-handler: called just before the probed function runs     */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * fixed_phy_change_carrier(struct net_device *dev, bool new_carrier)
     *
     * On x86-64, the first argument lands in RDI, second in RSI.
     * We cast them to recover the original parameters.
     */
    struct net_device *dev = (struct net_device *)regs->di;
    bool new_carrier        = (bool)regs->si;

    pr_info("fixed_phy_probe: dev=%-16s  carrier -> %s\n",
            dev ? netdev_name(dev) : "<null>",
            new_carrier ? "UP" : "DOWN");

    return 0; /* 0 = let the original function continue normally */
}

/* ------------------------------------------------------------------ */
/* kprobe descriptor                                                    */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "fixed_phy_change_carrier", /* target by symbol name */
    .pre_handler = handler_pre,                /* our callback          */
};

/* ------------------------------------------------------------------ */
/* module_init: register the kprobe                                    */
/* ------------------------------------------------------------------ */
static int __init fixed_phy_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("fixed_phy_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("fixed_phy_probe: planted at %pS\n", kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit: unregister the kprobe                                  */
/* ------------------------------------------------------------------ */
static void __exit fixed_phy_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("fixed_phy_probe: removed\n");
}

module_init(fixed_phy_probe_init);
module_exit(fixed_phy_probe_exit);
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | لازم لأي kernel module — بيعرّف `module_init`, `module_exit`, `MODULE_*` |
| `linux/kprobes.h` | بيجيب `struct kprobe` وكل API الـ kprobe subsystem |
| `linux/netdevice.h` | عشان نقدر نستخدم `struct net_device` و`netdev_name()` |
| `linux/kernel.h` | بيجيب `pr_info()` و`pr_err()` |

---

#### الـ `handler_pre` callback

**الـ handler ده** بيتنفذ في اللحظة اللي الـ CPU وصل فيها لأول instruction في `fixed_phy_change_carrier` قبل ما تتنفذ. الـ `struct pt_regs *regs` بيحتوي على قيم الـ registers في اللحظة دي.

- الـ `regs->di` هو `RDI` على x86-64 — اللي بيحمل الـ argument الأول (`struct net_device *dev`).
- الـ `regs->si` هو `RSI` — اللي بيحمل الـ argument التاني (`bool new_carrier`).

بنطبع اسم الـ device والحالة الجديدة للـ carrier عشان نعرف مين طلب تغيير الـ link state ومتى.

---

#### الـ `struct kprobe kp`

**الـ `.symbol_name`** بيخلّي الـ kernel يحل اسم الـ function لعنوانها وقت الـ `register_kprobe` تلقائياً — مش محتاجين نعرف العنوان يدوياً.
**الـ `.pre_handler`** هو الـ callback الوحيد اللي محتاجينه هنا لأننا بس عايزين نقرأ الـ arguments قبل تنفيذ الـ function الأصلية.

---

#### الـ `module_init`

**بنسجّل الـ kprobe** هنا وبنطبع العنوان الفعلي اللي انحط فيه الـ probe (`%pS` بيطبع symbol + offset) — مفيد للتأكد إن الـ kprobe اتحط صح.

---

#### الـ `module_exit`

**لازم** نعمل `unregister_kprobe` في الـ exit عشان الـ kernel يشيل الـ breakpoint instruction اللي حطه في الـ function. لو مشيناش الـ kprobe، الـ kernel هيعمل crash لأول ما يوصل لنقطة الـ probe بعد ما الـ module اتشال من الـ memory.

---

### Makefile وتشغيل الـ module

```makefile
obj-m += fixed_phy_carrier_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

```bash
# بناء الـ module
make

# تحميل الـ module
sudo insmod fixed_phy_carrier_probe.ko

# مشاهدة الـ output (مثلاً لو عندك virtual NIC بـ fixed PHY)
sudo dmesg -w | grep fixed_phy_probe

# مثال على output
# fixed_phy_probe: planted at fixed_phy_change_carrier+0x0/0x40
# fixed_phy_probe: dev=eth0              carrier -> UP
# fixed_phy_probe: dev=eth0              carrier -> DOWN

# إزالة الـ module
sudo rmmod fixed_phy_carrier_probe
```

> **ملحوظة:** `fixed_phy_change_carrier` بتتعمل في systems اللي فيها virtual أو emulated NICs زي QEMU virtio مع fixed PHY backend، أو أي driver بيستخدم `CONFIG_FIXED_PHY`. لو مفيش fixed PHY devices في النظام، الـ kprobe هيتسجل بنجاح لكن الـ handler مش هيتنفذ غير لما يتعمل carrier change فعلي.
