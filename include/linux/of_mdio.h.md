## Phase 1: الصورة الكبيرة ببساطة

### ما هو الموضوع ده؟

تخيل إن عندك طابعة (الـ PHY chip) وعندك كمبيوتر (الـ MAC) بيحتاج يتكلم معاها. الطابعة محتاجة كابل USB أو LPT تتوصل بيه. في عالم الـ networking، الـ PHY chip بيتوصل بالـ MAC عن طريق **MDIO bus** — ده الكابل اللي بيحملوا الأوامر والإعدادات.

لكن المشكلة: كل board مختلف. في board معين، الـ PHY ممكن يكون على address 1، وفي تاني على address 7، وممكن يكون في bus منفصل خالص. الـ board designer اللي بيعرف كل الحاجات دي.

**الـ Device Tree (DT)** هو الحل: ملف نصي بيوصف الـ hardware للـ kernel. فيه بيقولوا "الـ MDIO bus ده موجود في العنوان ده، وعليه PHY على address 3".

**الـ `of_mdio.h`** هو الـ header اللي بيوفر الـ API عشان الـ kernel يقرأ الـ Device Tree ويعمل setup للـ MDIO bus والـ PHY devices اللي عليه أوتوماتيك.

### القصة كاملة

زمان (قبل الـ DT)، كل driver كان فيه hardcoded setup: "الـ PHY على address 5، الـ bus اسمه كذا". ده معناه كود مكرر في كل driver، وصعوبة في support أكتر من board.

لما جه الـ Open Firmware (OF) / Device Tree، أصبح الـ hardware description منفصل عن الكود. الـ `of_mdio.h` بيوفر الـ bridge اللي يترجم الـ DT nodes لـ kernel objects (struct mii_bus و struct phy_device).

### ليه الملف ده مهم؟

- بيخلي أي Ethernet MAC driver يـ register الـ MDIO bus بتاعه من الـ DT بسطر واحد
- بيوفر lookup functions عشان تلاقي الـ PHY device من الـ DT node بتاعته
- بيدعم الـ **fixed-link** (لما ما فيش PHY حقيقي، بس الـ switch متوصل بـ fixed speed)
- بيتعامل مع الـ `CONFIG_OF_MDIO` flag — لو مفيش DT support، بيرجع لـ fallback

### الملفات المرتبطة

| الملف | الدور |
|-------|-------|
| `include/linux/of_mdio.h` | الـ API الرئيسي (الملف ده) |
| `drivers/net/mdio/of_mdio.c` | التنفيذ الفعلي للـ API |
| `include/linux/phy.h` | تعريف `mii_bus`, `phy_device`, `phy_interface_t` |
| `include/linux/mdio.h` | تعريف `mdio_device`, `mdio_driver` |
| `drivers/net/phy/` | درايفرات الـ PHY chips (Marvell, Realtek, etc.) |
| `drivers/net/mdio/` | درايفرات الـ MDIO controllers |
| `net/core/of_net.c` | OF helpers للـ networking بشكل عام |
| `include/linux/phy_fixed.h` | دعم الـ fixed-link PHYs |

### الـ Subsystem في MAINTAINERS

الملف ده جزء من **MDIO/PHY LIBRARY** subsystem، المسؤول عنه:
- **Andrew Lunn** `<andrew@lunn.ch>`
- **Heiner Kallweit** `<hkallweit1@gmail.com>`
- Mailing list: `netdev@vger.kernel.org`

الـ files المرتبطة بالـ subsystem:
- `drivers/net/mdio/` — كل الـ MDIO bus controllers
- `drivers/net/phy/` — كل الـ PHY drivers
- `include/linux/*mdio*.h` — كل الـ MDIO headers
- `Documentation/devicetree/bindings/net/mdio*` — الـ DT bindings

### ملخص

**الـ `of_mdio.h`** = جسر بين الـ Device Tree وبين subsystem الـ MDIO/PHY في الـ kernel. بدونه، كل MAC driver كان لازم يعمل الـ PHY setup يدوياً من الـ hardcoded data.
## Phase 2: شرح الـ OF/MDIO Framework

### المشكلة

كل Ethernet MAC controller بيحتاج:
1. **MDIO bus** عشان يتكلم مع الـ PHY chip (يقرأ/يكتب registers)
2. **PHY device** بيمثل الـ physical transceiver chip على الـ bus

الـ MDIO bus عنده addressing scheme: كل PHY ليه address من 0 لـ 31. في SoC معقد ممكن يكون فيه أكتر من MDIO bus، وكل bus عليه أكتر من PHY.

**المشكلة الأساسية**: إزاي الـ kernel يعرف:
- الـ MDIO bus ده موجود فين في الـ memory map؟
- الـ PHY address بتاعه كام؟
- الـ PHY ده متوصل بأيه MAC؟
- الـ interface بينهم RGMII ولا SGMII ولا غيره؟

### الحل: Open Firmware / Device Tree Integration

الـ kernel بيحل المشكلة دي عن طريق:
1. الـ board vendor يكتب ملف DT `.dts` بيوصف الـ hardware
2. الـ `of_mdio` subsystem بيقرأ الـ DT nodes ويعمل populate تلقائي

مثال DT node:

```dts
mdio0: mdio@10080000 {
    compatible = "snps,dwmac-mdio";
    #address-cells = <1>;
    #size-cells = <0>;
    reg = <0x10080000 0x100>;

    /* PHY on address 1 */
    phy0: ethernet-phy@1 {
        reg = <1>;              /* MDIO address */
        compatible = "ethernet-phy-ieee802.3-c22";
    };
};

ethernet@10070000 {
    phy-handle = <&phy0>;       /* pointer to the PHY */
    phy-mode = "rgmii-id";
};
```

### Architecture Diagram

```
  ┌─────────────────────────────────────────────────────────┐
  │                    User Space                           │
  │          ethtool / ip link / NetworkManager             │
  └────────────────────────┬────────────────────────────────┘
                           │ socket / netlink
  ┌────────────────────────▼────────────────────────────────┐
  │                   Kernel Network Stack                  │
  │           net_device / socket / protocols               │
  └────────────────────────┬────────────────────────────────┘
                           │
  ┌────────────────────────▼────────────────────────────────┐
  │            Ethernet MAC Driver (consumer)               │
  │   e.g. dwmac, fec, macb, stmmac, mvneta                │
  │                                                         │
  │  calls: of_mdiobus_register() / of_phy_connect()       │
  └────────────┬───────────────────────────────┬────────────┘
               │                               │
  ┌────────────▼──────────┐    ┌───────────────▼────────────┐
  │   of_mdio layer       │    │     phylink / phylib        │
  │  (of_mdio.h / .c)     │    │  (phy state machine)       │
  │                       │    │                             │
  │ Reads DT nodes        │    │ Manages link negotiation    │
  │ Registers mii_bus     │    │ Calls adjust_link callback  │
  │ Finds PHY devices     │    │                             │
  └────────────┬──────────┘    └─────────────────────────────┘
               │
  ┌────────────▼──────────────────────────────────────────┐
  │              MDIO Bus Core (mdiobus_register)         │
  │                    struct mii_bus                     │
  │    addr[0..31] → struct mdio_device / phy_device      │
  └────────────┬──────────────────────────────────────────┘
               │
  ┌────────────▼──────────────────────────────────────────┐
  │           MDIO Controller Driver (provider)           │
  │   e.g. mdio-gpio, mdio-mux, mdio-bcm-unimac          │
  │   implements: bus->read(), bus->write(), bus->reset() │
  └────────────┬──────────────────────────────────────────┘
               │  MDIO wire protocol (2-wire: MDC + MDIO)
  ┌────────────▼──────────────────────────────────────────┐
  │              Physical PHY Chip (hardware)             │
  │   e.g. Marvell 88E1111, Realtek RTL8211, TI DP83867  │
  └───────────────────────────────────────────────────────┘
               │
  ┌────────────▼──────────────────────────────────────────┐
  │           Device Tree (DTS / DTB)                     │
  │   mdio { phy@1 { reg = <1>; }; }                     │
  │   عنوان الـ OF node بيتترجم لـ kernel objects         │
  └───────────────────────────────────────────────────────┘
```

### الـ Analogy المرتبط بالـ Kernel

| الحياة العملية | الـ Kernel |
|---------------|------------|
| الـ USB hub | `struct mii_bus` |
| الجهاز على الـ USB | `struct mdio_device` / `struct phy_device` |
| الـ USB address | MDIO address (0-31) |
| دليل توصيل الجهاز | Device Tree node |
| اللي بيفسر الدليل | `of_mdio.c` |
| بروتوكول التواصل | MDIO (MDC clock + MDIO data) |

### Core Abstractions

**1. `struct mii_bus`** — الـ bus نفسه. فيه read/write function pointers لـ raw MDIO access، وarray of `mdio_device` pointers indexed by PHY address.

**2. `struct mdio_device`** — الجهاز على الـ bus. embedded في `struct phy_device`. عنده الـ address وpointer للـ bus.

**3. `struct phy_device`** — PHY chip محدد، مع كل معلوماته: الـ link state، الـ interface mode، الـ driver.

**4. `struct device_node`** — تمثيل الـ kernel لـ DT node. الـ of_mdio layer يستخدمه للـ lookup والـ binding.

### الـ OF/MDIO Layer يمتلك ماذا؟

| ما يمتلكه OF/MDIO | ما يفوّضه |
|-------------------|------------|
| قراءة `reg` property وتحويله لـ MDIO address | تنفيذ read/write (للـ controller driver) |
| إنشاء `phy_device` لكل child node | state machine الـ PHY (لـ phylib) |
| ربط `phy_device` بـ DT node handle | الـ network stack فوق ده |
| دعم `fixed-link` nodes | إدارة الـ interrupts |
| `phy_mask` لتجاهل addresses معينة | |
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

### جدول الـ Config Options المهمة

| Option | القيمة | التأثير |
|--------|--------|---------|
| `CONFIG_OF_MDIO` | bool | يفعّل كل الـ of_mdio API. لو مش مفعّل، كل الدوال بترجع stub |
| `CONFIG_PHYLIB` | bool | الـ PHY library الأساسية. شرط لـ MDIO |
| `CONFIG_PHY_PACKAGE` | bool | يفعّل الـ `shared` array في `mii_bus` لـ multi-PHY packages |
| `CONFIG_FIXED_PHY` | bool | يفعّل الـ fixed-link support (`of_phy_register_fixed_link`) |
| `CONFIG_MDIO_DEVICE` | bool | الـ base MDIO device infrastructure |

### جدول الـ Constants المهمة

| الثابت | القيمة | المعنى |
|--------|--------|--------|
| `PHY_MAX_ADDR` | 32 | أقصى عدد PHY devices على bus واحد (MDIO addressing limit) |
| `MII_BUS_ID_SIZE` | 61 | حجم الـ string ID للـ bus |
| `MDIO_MUTEX_NORMAL` | 0 | lock class عادي |
| `MDIO_MUTEX_MUX` | 1 | lock class للـ mux layers |
| `MDIO_MUTEX_NESTED` | 2 | lock class nested |

### الـ Structs الأساسية

#### `struct mii_bus` (من `include/linux/phy.h`)

```c
struct mii_bus {
    struct module *owner;          /* module owner for ref counting */
    const char *name;              /* human-readable bus name */
    char id[MII_BUS_ID_SIZE];      /* unique bus ID string */
    void *priv;                    /* driver private data */

    /* Operations - implemented by the MDIO controller driver */
    int (*read)(struct mii_bus *bus, int addr, int regnum);
    int (*write)(struct mii_bus *bus, int addr, int regnum, u16 val);
    int (*read_c45)(struct mii_bus *bus, int addr, int devnum, int regnum);
    int (*write_c45)(struct mii_bus *bus, int addr, int devnum, int regnum, u16 val);
    int (*reset)(struct mii_bus *bus);

    struct mdio_bus_stats stats[PHY_MAX_ADDR]; /* per-PHY statistics */
    struct mutex mdio_lock;        /* serializes all MDIO transactions */
    struct device *parent;         /* parent device (MAC controller) */

    enum {
        MDIOBUS_ALLOCATED = 1,
        MDIOBUS_REGISTERED,
        MDIOBUS_UNREGISTERED,
        MDIOBUS_RELEASED,
    } state;                       /* bus lifecycle state */

    struct device dev;             /* kernel device representation */
    struct mdio_device *mdio_map[PHY_MAX_ADDR]; /* devices by address */
    u32 phy_mask;                  /* bitmask: addresses to skip during scan */
    u32 phy_ignore_ta_mask;        /* ignore turnaround errors at these addrs */
    int irq[PHY_MAX_ADDR];         /* per-PHY IRQ numbers */
    int reset_delay_us;            /* GPIO reset pulse width */
    int reset_post_delay_us;       /* delay after reset deassertion */
    struct gpio_desc *reset_gpiod; /* optional reset GPIO */
    struct mutex shared_lock;      /* protects shared[] array */
    struct phy_package_shared *shared[PHY_MAX_ADDR]; /* for PHY packages */
};
```

#### `struct mdio_device` (من `include/linux/mdio.h`)

```c
struct mdio_device {
    struct device dev;             /* base kernel device */
    struct mii_bus *bus;           /* the MDIO bus this device is on */

    /* bus_match: called during driver binding */
    int (*bus_match)(struct device *dev, const struct device_driver *drv);
    void (*device_free)(struct mdio_device *mdiodev);
    void (*device_remove)(struct mdio_device *mdiodev);

    int addr;                      /* MDIO address (0-31) */
    int flags;                     /* MDIO_DEVICE_FLAG_PHY if it's a PHY */
    int reset_state;               /* current reset state */
    struct gpio_desc *reset_gpio;  /* optional per-device reset */
    struct reset_control *reset_ctrl;
    unsigned int reset_assert_delay;
    unsigned int reset_deassert_delay;
};
```

#### `struct phy_device` (من `include/linux/phy.h`)

```c
struct phy_device {
    struct mdio_device mdio;       /* MUST be first - inherits mdio_device */
    const struct phy_driver *drv;  /* bound PHY driver */
    u32 phy_id;                    /* PHY chip ID (read from registers 2&3) */
    phy_interface_t interface;     /* RGMII / SGMII / MII / etc. */
    unsigned link:1;               /* current link state */
    unsigned autoneg:1;            /* autoneg enabled? */
    int speed;                     /* current link speed */
    int duplex;                    /* current duplex */
    struct net_device *attached_dev; /* the MAC using this PHY */
    void (*adjust_link)(struct net_device *dev); /* link change callback */
    struct mutex lock;             /* PHY state machine lock */
    /* ... many more fields ... */
};
```

### Struct Relationship Diagram

```
  ┌─────────────────────────────────────────────────────┐
  │                  struct mii_bus                     │
  │  id[], name, priv                                   │
  │  read(), write(), reset() ◄── controller driver    │
  │  mdio_lock (mutex)                                  │
  │  mdio_map[32] ──────────────────────┐               │
  │  irq[32]                            │               │
  │  phy_mask                           │               │
  │  struct device dev                  │               │
  └─────────────────────────────────────┼───────────────┘
                                        │ [addr]
              ┌─────────────────────────▼──────────────┐
              │          struct mdio_device             │
              │  struct device dev                      │
              │  struct mii_bus *bus ◄──────────────────┤
              │  int addr (0-31)                        │
              │  int flags                              │
              │  reset_gpio, reset_ctrl                 │
              └─────────────────────────┬───────────────┘
                         ▲             │ embedded in
                         │             ▼
              ┌──────────┴──────────────────────────────┐
              │          struct phy_device               │
              │  struct mdio_device mdio ◄── (above)    │
              │  u32 phy_id                              │
              │  phy_interface_t interface               │
              │  const struct phy_driver *drv            │
              │  struct net_device *attached_dev ────────┼─► MAC
              │  void (*adjust_link)(net_device *)       │
              │  struct mutex lock                       │
              │  enum phy_state state                    │
              └──────────────────────────────────────────┘
                                        ▲
                         ┌──────────────┘
              ┌──────────┴──────────────────────────────┐
              │          struct device_node              │
              │  (Device Tree node)                      │
              │  "ethernet-phy@1"                        │
              │  reg = <1>                               │
              │  linked via of_node pointer              │
              └──────────────────────────────────────────┘
```

### Lifecycle: إنشاء → Registration → Usage → Teardown

```
CREATION PHASE
══════════════
  MAC driver probe()
       │
       ▼
  mdiobus_alloc()              ← تخصيص struct mii_bus
       │
       ▼
  bus->read = my_read          ← تسجيل الـ operations
  bus->write = my_write
  bus->name = "my-mdio"
       │
       ▼
REGISTRATION PHASE (of_mdio.h API)
═══════════════════════════════════
  of_mdiobus_register(bus, np) ← np = DT node of mdio controller
       │
       ▼
  __of_mdiobus_register()      ← الـ real implementation
       │
       ├─► of_property_read_u32(child, "reg", &addr)  ← read PHY address
       │
       ├─► of_mdiobus_child_is_phy(child)              ← is it a PHY?
       │
       ├─► mdiobus_register()                          ← register with kernel
       │       │
       │       └─► probe each address → create phy_device
       │
       └─► of_mdiobus_phy_device_register()            ← bind DT node to phy
               │
               └─► phydev->mdio.dev.of_node = child

USAGE PHASE
═══════════
  MAC driver:
  of_phy_connect(netdev, phy_np, adjust_link_cb, 0, PHY_INTERFACE_MODE_RGMII)
       │
       ├─► of_phy_find_device(phy_np)    ← lookup by DT node
       │
       └─► phy_connect_direct()          ← start PHY state machine

TEARDOWN PHASE
══════════════
  MAC driver remove():
  phy_disconnect(phydev)
       │
  mdiobus_unregister(bus)
       │
  mdiobus_free(bus)
```

### Locking Strategy

| الـ Lock | النوع | يحمي ماذا |
|----------|-------|------------|
| `mii_bus->mdio_lock` | `mutex` | جميع MDIO read/write transactions (serialized) |
| `phy_device->lock` | `mutex` | PHY state machine والـ config operations |
| `mii_bus->shared_lock` | `mutex` | الـ `shared[]` array للـ PHY packages |

**القاعدة الأساسية**: لا تعمل MDIO access بدون `mdio_lock`. الـ `mdiobus_read()` و `mdiobus_write()` بتاخد الـ lock تلقائياً.
## Phase 4: شرح الـ Functions

### جدول الـ Functions (Cheatsheet)

| الدالة | الفئة | الغرض الرئيسي |
|--------|-------|---------------|
| `of_mdiobus_register()` | Registration | تسجيل MDIO bus من DT node |
| `devm_of_mdiobus_register()` | Registration | نفسه لكن devres-managed |
| `__of_mdiobus_register()` | Registration | الـ real implementation (تمر بها THIS_MODULE) |
| `__devm_of_mdiobus_register()` | Registration | الـ real devres implementation |
| `of_mdio_find_bus()` | Lookup | إيجاد `mii_bus` من DT node |
| `of_mdio_find_device()` | Lookup | إيجاد `mdio_device` من DT node |
| `of_phy_find_device()` | Lookup | إيجاد `phy_device` من DT node |
| `of_phy_connect()` | Connection | إيجاد PHY وتوصيله بـ net_device |
| `of_phy_get_and_connect()` | Connection | قراءة `phy-handle` من DT ثم connect |
| `of_mdiobus_child_is_phy()` | Inspection | هل الـ child node ده PHY؟ |
| `of_mdiobus_phy_device_register()` | Registration | ربط PHY بـ DT node بعد الإنشاء |
| `of_phy_register_fixed_link()` | Fixed Link | تسجيل fixed-link PHY من DT |
| `of_phy_deregister_fixed_link()` | Fixed Link | إلغاء تسجيل fixed-link PHY |
| `of_phy_is_fixed_link()` | Fixed Link | هل الـ node ده fixed-link? |
| `of_mdio_parse_addr()` | Parsing | قراءة وتحقق من الـ `reg` property |

---

### فئة 1: Registration Functions

#### `of_mdiobus_register()`

```c
static inline int of_mdiobus_register(struct mii_bus *mdio,
                                      struct device_node *np)
{
    return __of_mdiobus_register(mdio, np, THIS_MODULE);
}
```

الدالة دي بتاخد `mii_bus` مُعد مسبقاً وـ DT node، وبتمرر `THIS_MODULE` للـ implementation الحقيقية. بتقرأ كل الـ child nodes من الـ `np`، وبتعمل probe لكل PHY موجود في الـ DT، وبتعمل bind بين الـ phy_device والـ DT node بتاعه.

| البارامتر | النوع | الوصف |
|-----------|-------|-------|
| `mdio` | `struct mii_bus *` | الـ bus المُعد (read/write/reset مُسجلين) |
| `np` | `struct device_node *` | الـ DT node للـ MDIO controller |

**Return**: 0 عند النجاح، كود خطأ سالب عند الفشل.

**Side effects**: يعمل register للـ bus مع الـ kernel، ويعمل probe لكل PHY، ويعمل bind الـ DT nodes.

**من يستدعيها**: MAC drivers عند الـ probe: `stmmac`, `fec`, `macb`, `mvneta`, إلخ.

---

#### `devm_of_mdiobus_register()`

```c
static inline int devm_of_mdiobus_register(struct device *dev,
                                           struct mii_bus *mdio,
                                           struct device_node *np)
{
    return __devm_of_mdiobus_register(dev, mdio, np, THIS_MODULE);
}
```

نسخة الـ devres: الـ unregister بتحصل أوتوماتيك لما الـ `dev` يتـ removed. مريحة في الـ modern drivers عشان مش محتاج `.remove` callback.

| البارامتر | النوع | الوصف |
|-----------|-------|-------|
| `dev` | `struct device *` | الـ device المسؤولة عن lifecycle الـ bus |
| `mdio` | `struct mii_bus *` | الـ bus المُعد |
| `np` | `struct device_node *` | الـ DT node للـ MDIO |

**Return**: 0 عند النجاح، كود خطأ سالب عند الفشل.

---

### فئة 2: Lookup Functions

#### `of_mdio_find_bus()`

```c
struct mii_bus *of_mdio_find_bus(struct device_node *mdio_np);
```

بتبحث عن `mii_bus` مُسجل بيمثل الـ DT node المحدد. مفيدة لما driver تاني (مش الـ MAC الأساسي) يحتاج يوصل لنفس الـ MDIO bus.

**Return**: pointer لـ `mii_bus` (مع زيادة reference count) أو NULL.

**تحذير**: لازم تعمل `put_device(&bus->dev)` بعد الاستخدام عشان تـ release الـ reference.

**من يستدعيها**: DSA switches، MDIO mux drivers، أي driver يحتاج يوصل لـ MDIO bus بالـ DT node بتاعه.

---

#### `of_phy_find_device()`

```c
struct phy_device *of_phy_find_device(struct device_node *phy_np);
```

بتبحث عن `phy_device` مُسجل ومرتبط بالـ DT node المحدد. بتستخدم الـ `of_node` pointer المخزن في `phy_device->mdio.dev.of_node`.

**Return**: pointer لـ `phy_device` (مع reference) أو NULL لو مش موجود أو مش متسجل لسه.

---

#### `of_mdio_find_device()`

```c
struct mdio_device *of_mdio_find_device(struct device_node *np);
```

أعم من `of_phy_find_device`: بتلاقي أي `mdio_device` (مش بس PHY) مرتبط بالـ DT node. مفيدة للـ non-PHY MDIO devices (زي switches أو DSA chips).

**Return**: pointer لـ `mdio_device` أو NULL.

---

### فئة 3: Connection Functions

#### `of_phy_connect()`

```c
struct phy_device *
of_phy_connect(struct net_device *dev, struct device_node *phy_np,
               void (*hndlr)(struct net_device *), u32 flags,
               phy_interface_t iface);
```

بتجمع `of_phy_find_device()` + `phy_connect_direct()` في استدعاء واحد. بتلاقي الـ PHY من الـ DT node وبتوصله بالـ `net_device` مع تسجيل الـ `adjust_link` callback.

| البارامتر | النوع | الوصف |
|-----------|-------|-------|
| `dev` | `struct net_device *` | الـ MAC's network device |
| `phy_np` | `struct device_node *` | الـ DT node للـ PHY |
| `hndlr` | function pointer | callback لما الـ link state يتغير |
| `flags` | `u32` | PHY flags (عادةً 0) |
| `iface` | `phy_interface_t` | الـ interface mode (RGMII, SGMII, etc.) |

**Return**: pointer لـ `phy_device` عند النجاح، NULL عند الفشل.

**Pseudocode Flow**:
```
of_phy_connect(dev, phy_np, handler, flags, iface):
    phydev = of_phy_find_device(phy_np)
    if !phydev → return NULL
    ret = phy_connect_direct(dev, phydev, handler, iface)
    if ret → put_device; return NULL
    return phydev
```

---

#### `of_phy_get_and_connect()`

```c
struct phy_device *
of_phy_get_and_connect(struct net_device *dev, struct device_node *np,
                       void (*hndlr)(struct net_device *));
```

الأكثر راحة: بتقرأ `phy-handle` property من الـ `np` (الـ MAC's DT node)، وبتلاقي الـ PHY، وبتقرأ `phy-mode` property وبتوصل. driver ما يحتاجش يعرف الـ PHY node أو الـ interface mode يدوياً.

**Return**: pointer لـ `phy_device` أو NULL.

---

### فئة 4: Fixed Link Functions

#### `of_phy_is_fixed_link()`

```c
bool of_phy_is_fixed_link(struct device_node *np);
```

بتشوف لو الـ DT node عنده `fixed-link` sub-node. الـ fixed-link بيُستخدم لما الـ MAC متوصل بـ switch أو SoC port بـ fixed speed بدون PHY negotiation.

**Return**: `true` لو الـ DT node عنده `fixed-link`.

---

#### `of_phy_register_fixed_link()`

```c
int of_phy_register_fixed_link(struct device_node *np);
```

بتقرأ الـ `fixed-link` sub-node من الـ DT وبتسجل **virtual PHY device** يمثّله. الـ virtual PHY ده بيوفر link state ثابتة للـ phylib state machine.

**Return**: 0 عند النجاح.

---

#### `of_phy_deregister_fixed_link()`

```c
void of_phy_deregister_fixed_link(struct device_node *np);
```

بتلغي تسجيل الـ virtual PHY اللي سبق تسجيله بـ `of_phy_register_fixed_link()`.

---

### فئة 5: Inspection/Parsing

#### `of_mdiobus_child_is_phy()`

```c
bool of_mdiobus_child_is_phy(struct device_node *child);
```

بتشوف لو الـ child node ده PHY أم لا. بتبحث عن `compatible = "ethernet-phy-ieee802.3-c22"` أو أي compatible string فيه "ethernet-phy". لو مفيش compatible، بتفترض إنه PHY.

**Return**: `true` لو الـ node ده PHY.

---

#### `of_mdio_parse_addr()` (inline)

```c
static inline int of_mdio_parse_addr(struct device *dev,
                                     const struct device_node *np)
{
    u32 addr;
    int ret;
    ret = of_property_read_u32(np, "reg", &addr);
    if (ret < 0) {
        dev_err(dev, "%s has invalid PHY address\n", np->full_name);
        return ret;
    }
    if (addr >= PHY_MAX_ADDR) {  /* PHY_MAX_ADDR = 32 */
        dev_err(dev, "%s PHY address %i is too large\n", np->full_name, addr);
        return -EINVAL;
    }
    return addr;
}
```

بتقرأ الـ `reg` property من الـ DT node وبتتحقق إن القيمة في المدى [0, 31]. لو الـ `reg` مش موجود أو أكبر من 31 بتطبع error وبترجع error code.

**Return**: الـ address (0-31) عند النجاح، كود خطأ سالب عند الفشل.

**من يستدعيها**: `__of_mdiobus_register()` لكل child node.

---

#### `of_mdiobus_phy_device_register()`

```c
int of_mdiobus_phy_device_register(struct mii_bus *mdio, struct phy_device *phy,
                                   struct device_node *child, u32 addr);
```

بتعمل bind بين `phy_device` مُنشأ مسبقاً وبين الـ DT node بتاعه. بتسجل الـ phy على الـ bus على الـ address المحدد.

**Return**: 0 عند النجاح، كود خطأ سالب عند الفشل.

**من يستدعيها**: `__of_mdiobus_register()` أثناء bus scanning.
## Phase 5: دليل الـ Debugging الشامل

### الـ Kernel Config Options للـ Debugging

| Option | الغرض |
|--------|-------|
| `CONFIG_PHYLIB` | الـ base PHY library (لازم يكون enabled) |
| `CONFIG_OF_MDIO` | تفعيل الـ OF/MDIO integration |
| `CONFIG_MDIO_DEVICE` | base MDIO device support |
| `CONFIG_DYNAMIC_DEBUG` | تفعيل `dev_dbg()` runtime |
| `CONFIG_NET_CORE` | networking core (شرط) |

---

### Dynamic Debug

تفعيل كل debug messages في ملف `of_mdio.c`:

```bash
# تفعيل كل pr_debug() و dev_dbg() في of_mdio.c
echo 'file of_mdio.c +p' > /sys/kernel/debug/dynamic_debug/control

# تحقق من الـ active rules
cat /sys/kernel/debug/dynamic_debug/control | grep mdio

# تفعيل كل debug messages في PHY subsystem
echo 'module phylib +p' > /sys/kernel/debug/dynamic_debug/control
```

---

### sysfs Entries

```bash
# قائمة كل MDIO buses المُسجلة
ls /sys/bus/mdio_bus/devices/

# مثال output:
# mdio_bus-0000  (اسم الـ bus)

# معلومات عن PHY device محدد
ls /sys/bus/mdio_bus/devices/stmmac-0:01/
# output: driver  modalias  of_node  power  subsystem  uevent

# الـ phy_id
cat /sys/bus/mdio_bus/devices/stmmac-0:01/phy_id
# output: 0x001cc916  (Realtek RTL8211F)

# الـ interface
cat /sys/class/net/eth0/phydev/interface
```

---

### debugfs Entries

```bash
# مسار الـ debugfs (لو mounted)
ls /sys/kernel/debug/

# بعض PHY drivers بتضيف entries:
ls /sys/kernel/debug/phy/
# mdio_bus-0:01/

# MDIO statistics (من mii_bus->stats[])
cat /sys/kernel/debug/mdio_bus/stmmac-0/statistics
```

---

### ethtool لـ PHY Debugging

```bash
# حالة الـ PHY (link, speed, duplex)
ethtool eth0

# إجبار speed معينة (bypass autoneg)
ethtool -s eth0 speed 100 duplex full autoneg off

# قراءة register من الـ PHY مباشرة
ethtool --get-phy-reg eth0 0    # reg 0 = Basic Control Register
ethtool --get-phy-reg eth0 1    # reg 1 = Basic Status Register
ethtool --get-phy-reg eth0 2    # reg 2 = PHY ID 1
ethtool --get-phy-reg eth0 3    # reg 3 = PHY ID 2

# مثال output:
# PHY #0 reg[0x0000] = 0x3100   (autoneg enabled, 1000Mbps)
# PHY #0 reg[0x0001] = 0x796d   (link up, autoneg complete)
```

---

### ftrace / Tracepoints

```bash
# تفعيل tracing للـ MDIO events
cd /sys/kernel/debug/tracing
echo 1 > events/mdio/enable

# أو بشكل انتقائي:
echo 1 > events/mdio/mdio_access/enable

# مشاهدة الـ trace
cat trace_pipe

# مثال output:
# eth0-irq/0-144  [000] ....  123.456789: mdio_access: bus=stmmac-0 \
#                 phy_id=1 is_write=0 regnum=1 val=0x796d
```

---

### جدول: Common Errors والحلول

| رسالة الخطأ | السبب | الحل |
|-------------|-------|------|
| `XXX has invalid PHY address` | الـ `reg` property مش موجود في DT | أضف `reg = <N>;` للـ PHY node |
| `XXX PHY address N is too large` | الـ `reg` value >= 32 | القيمة لازم تكون 0-31 |
| `of_phy_connect: Unable to find bus for phy` | الـ MDIO bus مش متسجل لسه | تأكد إن `of_mdiobus_register()` اتعمل قبل `of_phy_connect()` |
| `PHY N not found on slave M` | الـ PHY مش بيرد على الـ MDIO address | تحقق من الـ hardware connection وصحة الـ address في DT |
| `fwnode is not a PHY` | الـ node مش PHY node | تحقق من الـ compatible string في DT |
| `No PHY found` | مفيش PHY بيرد على أي address | تحقق من power supply للـ PHY وكابل MDIO |
| `mdiobus_register failed` | مشكلة في تسجيل الـ bus | تحقق من الـ bus id وتأكد مش متسجل قبل كده |
| `-EPROBE_DEFER` | الـ MDIO controller driver مش probe لسه | طبيعي — الـ kernel هيـ retry تلقائياً |

---

### Strategic WARN_ON / dump_stack Points

```c
/* في of_mdio_parse_addr - لو الـ addr مش صح */
WARN_ON(addr >= PHY_MAX_ADDR);

/* لو of_phy_find_device رجع NULL بعد of_mdiobus_register ناجح */
phydev = of_phy_find_device(phy_np);
if (WARN_ON(!phydev))
    return -ENODEV;

/* لو الـ bus مش في state MDIOBUS_REGISTERED */
WARN_ON(mdio->state != MDIOBUS_REGISTERED);
```

---

### Hardware Level Debugging

#### تحقق من الـ MDIO electrical signals

الـ MDIO bus عنده سلكين:
- **MDC**: clock (من الـ MAC)
- **MDIO**: data (bidirectional)

```
# لو عندك logic analyzer:
# 1. capture على MDC و MDIO
# 2. ابحث عن preamble: 32 bits of '1'
# 3. start frame: 01
# 4. opcode: 10 (read) أو 01 (write)
# 5. PHY address: 5 bits
# 6. register address: 5 bits
# 7. turnaround: Z0 (read) أو 10 (write)
# 8. data: 16 bits
```

#### DT Debugging

```bash
# تأكيد إن الـ DT node موجود
dtc -I fs /sys/firmware/devicetree/base | grep -A 20 "mdio"

# أو
cat /sys/firmware/devicetree/base/soc/ethernet@.../mdio@.../reg

# تحقق من الـ compatible
cat /sys/firmware/devicetree/base/soc/mdio@10080000/compatible

# طباعة كل الـ of_node للـ bus
ls -la /sys/bus/mdio_bus/devices/*/of_node
```

#### Register Dump من الـ PHY

```bash
# قراءة كل standard registers (0-31)
for i in $(seq 0 31); do
    val=$(ethtool --get-phy-reg eth0 $i 2>/dev/null)
    echo "reg[$i] = $val"
done

# مثال output لـ RTL8211F:
# reg[0] = 0x3100   # Control: AN enable, full duplex
# reg[1] = 0x796d   # Status: link up, AN complete
# reg[2] = 0x001c   # PHY ID high
# reg[3] = 0xc916   # PHY ID low
```

---

### Shell Commands الجاهزة للنسخ

```bash
# --- 1. شوف كل PHY devices على الـ MDIO bus ---
for dev in /sys/bus/mdio_bus/devices/*/; do
    echo "=== $(basename $dev) ==="
    cat "$dev/phy_id" 2>/dev/null && echo ""
done

# --- 2. تفعيل كامل للـ MDIO/PHY debugging ---
echo 'file of_mdio.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file phy.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file phy_device.c +p' > /sys/kernel/debug/dynamic_debug/control

# --- 3. مشاهدة MDIO tracepoints ---
echo 1 > /sys/kernel/debug/tracing/events/mdio/enable
cat /sys/kernel/debug/tracing/trace_pipe &
TRACE_PID=$!
sleep 5
kill $TRACE_PID
echo 0 > /sys/kernel/debug/tracing/events/mdio/enable

# --- 4. تحقق من الـ PHY link state ---
ethtool eth0 | grep -E "Link|Speed|Duplex|Auto"

# --- 5. إيجاد الـ MDIO bus المرتبط بـ interface ---
readlink /sys/class/net/eth0/phydev/bus

# مثال output:
# ../../../bus/mdio_bus
```
## Phase 6: سيناريوهات من الحياة العملية

---

### سيناريو 1: RK3562 Industrial Gateway — الـ PHY مش بيتشاف

**العنوان**: PHY address خاطئ في الـ Device Tree

**السياق**: شركة بتعمل Industrial IoT gateway على أساس RK3562 SoC. الـ Ethernet بتاع الـ board اتعمل spin عليه — الـ PHY تغير من Realtek RTL8211F (address 1) لـ Marvell 88E1111 (address 0) عشان supply chain issues.

**المشكلة**: بعد الـ hardware change، الـ Ethernet مش بيشتغل. الـ dmesg بيقول:
```
stmmac-eth ff540000.ethernet: No PHY found
stmmac-eth ff540000.ethernet eth0: no PHY configured
```

**التحليل**:

الـ DT قديم لسه فيه:
```dts
mdio {
    phy0: ethernet-phy@1 {
        reg = <1>;   /* ← المشكلة: address قديم */
        compatible = "ethernet-phy-ieee802.3-c22";
    };
};
```

لما الـ kernel بيعمل `of_mdiobus_register()`:
1. `__of_mdiobus_register()` بيمشي على كل child node
2. بيستدعي `of_mdio_parse_addr()` على الـ `ethernet-phy@1` node
3. `of_property_read_u32(np, "reg", &addr)` بيرجع `addr = 1`
4. الـ kernel بيعمل probe على MDIO address 1
5. الـ probe بيبعت MDIO read لـ registers 2 و 3 عشان يقرأ الـ phy_id
6. الـ chip الجديد على address 0 مش بيرد على address 1 → `0xFFFF` (no device)
7. `get_phy_device()` بيفشل → "No PHY found"

**الحل**:

```dts
/* بعد التعديل */
mdio {
    phy0: ethernet-phy@0 {
        reg = <0>;   /* ← العنوان الصح للـ chip الجديد */
        compatible = "ethernet-phy-ieee802.3-c22";
    };
};
ethernet {
    phy-handle = <&phy0>;
    phy-mode = "rgmii-id";
};
```

```bash
# تحقق من الـ address الصح بـ mdio-tool أو phytool
phytool read eth0/0  # جرب address 0
# output: reg[0x00] = 0x1140  ← الـ PHY بيرد!
```

**الدرس المستفاد**: أي تغيير hardware في الـ PHY address لازم يتعكس في الـ DT. استخدم `phytool` أو `ethtool` تحقق من أي address بيرد قبل تعديل الـ DT.

---

### سيناريو 2: STM32MP1 Custom Board — Fixed Link مع Switch

**العنوان**: لا PHY — direct connection لـ Gigabit Switch

**السياق**: Board بيستخدم STM32MP1 في medical device. الـ design عنده GMAC متوصل مباشرة بـ RGMII لـ managed switch (Microchip KSZ9897) بدون external PHY chip، الـ link speed ثابتة 1Gbps.

**المشكلة**: الـ kernel بيتحاول يعمل PHY discovery على الـ MDIO bus ومش بيلاقي PHY، وبيفشل:
```
dwmac.5c008000: No PHY found
```

**التحليل**:

المبرمج استخدم `of_mdiobus_register()` زي العادي، لكن الـ DT ما فيهوش PHY node. الـ kernel ما عندوش PHY يتوصل بيه.

الـ solution الصحيح هو `fixed-link`:

```dts
ethernet@5c008000 {
    compatible = "st,stm32mp1-dwmac";
    phy-mode = "rgmii";
    /* لا phy-handle هنا */

    fixed-link {
        speed = <1000>;
        full-duplex;
    };
};
```

لما الـ MAC driver بيعمل probe، بيستدعي `of_phy_is_fixed_link(np)`:
- بتشوف لو فيه `fixed-link` subnode
- ترجع `true`

فالـ driver بيستدعي `of_phy_register_fixed_link(np)`:
- بتقرأ `speed` و `full-duplex` من الـ `fixed-link` subnode
- بتسجل virtual PHY device بـ `fixed_phy_register()`
- الـ virtual PHY ده دايماً بيرجع link up بالـ speed المحددة

ثم `of_phy_connect()` بيوصل الـ MAC بالـ virtual PHY.

**الحل الكامل** في الـ MAC driver:

```c
/* في probe(): */
if (of_phy_is_fixed_link(np)) {
    ret = of_phy_register_fixed_link(np);
    if (ret < 0)
        return ret;
    phy_np = of_node_get(np); /* fixed-link's virtual node */
} else {
    phy_np = of_parse_phandle(np, "phy-handle", 0);
}
phydev = of_phy_connect(ndev, phy_np, &adjust_link, 0, iface);
```

**الدرس المستفاد**: الـ `fixed-link` هو الطريقة الصحيحة لـ direct MAC-to-switch connections. الـ `of_phy_is_fixed_link()` لازم تتفحص قبل محاولة PHY discovery.

---

### سيناريو 3: i.MX8 Android TV Box — MDIO Bus Sharing

**العنوان**: MDIO bus واحد لـ PHYين مختلفين

**السياق**: Android TV Box على أساس i.MX8MM. الـ board عنده:
- Ethernet port موصل على Realtek RTL8211F على address 1
- WiFi chip (Marvell 88W8987) بيستخدم نفس MDIO bus على address 2 لـ configuration

**المشكلة**: الـ WiFi driver بيحاول يعمل `mdiobus_register()` خاص بيه، فبيفشل لأن الـ bus متسجل بالفعل:
```
mdio_bus fixed-0: Cannot get bus 'fec.mdio': already registered
```

**التحليل**:

الـ DT لازم يوصف الـ sharing:

```dts
fec_mdio: mdio@30be0000 {
    /* shared MDIO bus */
    #address-cells = <1>;
    #size-cells = <0>;

    /* Ethernet PHY */
    ethphy0: ethernet-phy@1 {
        reg = <1>;
        compatible = "ethernet-phy-ieee802.3-c22";
    };

    /* WiFi chip MDIO device (non-PHY) */
    wifi_mdio: wifi@2 {
        reg = <2>;
        compatible = "marvell,sd8xxx-mdio";
    };
};

ethernet@30be0000 {
    phy-handle = <&ethphy0>;
    phy-mode = "rgmii-id";
    mdio = <&fec_mdio>;  /* يشير لنفس الـ bus */
};

wifi {
    mdio = <&fec_mdio>;
    /* WiFi driver يستخدم of_mdio_find_bus() */
};
```

الـ WiFi driver يستخدم:
```c
/* يلاقي الـ existing bus بالـ DT node */
bus = of_mdio_find_bus(wifi_mdio_np);
if (!bus)
    return -EPROBE_DEFER;
/* يعمل access مباشر على address 2 بدون تسجيل bus جديد */
mdiobus_read(bus, 2, WIFI_CTRL_REG);
put_device(&bus->dev);  /* ← مهم جداً */
```

**الدرس المستفاد**: `of_mdio_find_bus()` بتدي reference — لازم تعمل `put_device()` بعد الاستخدام. الـ MDIO bus واحد ممكن يخدم أنواع مختلفة من devices.

---

### سيناريو 4: AM62x Automotive ECU — EPROBE_DEFER Issue

**العنوان**: Race condition بين MDIO controller و MAC driver

**السياق**: Automotive ECU بيستخدم AM62x SoC. الـ Ethernet MAC driver بيعمل probe قبل الـ MDIO controller driver، فبيرجع error.

**المشكلة**:
```
cpsw_new 8000000.ethernet: Failed to find MII bus
cpsw_new 8000000.ethernet: probe deferred
```

**التحليل**:

الـ AM62x عنده MDIO controller منفصل (`ti,davinci_mdio`). الـ MAC driver بيستدعي `of_mdio_find_bus()`:

```c
/* في MAC driver probe(): */
mdio_np = of_parse_phandle(np, "mdio-bus", 0);
bus = of_mdio_find_bus(mdio_np);
if (!bus) {
    /* MDIO bus مش متسجل لسه */
    return -EPROBE_DEFER;  /* ← الـ kernel هيـ retry لاحقاً */
}
```

لما `of_mdio_find_bus()` بترجع NULL، معناها إن الـ MDIO controller لسه ما عملش probe. الـ `-EPROBE_DEFER` بيقول للـ kernel "جرب تاني بعد ما drivers تانية تـ probe".

الـ DT لازم يوضح الـ dependency:

```dts
mdio0: mdio@4f00000 {
    compatible = "ti,davinci_mdio";
    reg = <0x4f00000 0x100>;
    #address-cells = <1>;
    #size-cells = <0>;

    phy0: ethernet-phy@0 {
        reg = <0>;
    };
};

ethernet@8000000 {
    compatible = "ti,am642-cpsw-nuss";
    mdio-bus = <&mdio0>;  /* explicit dependency */
    phy-handle = <&phy0>;
    phy-mode = "rgmii-id";
};
```

**الحل**: لا يوجد bug — هذا هو الـ expected behavior. الـ `-EPROBE_DEFER` mechanism هو الحل الصحيح لـ initialization ordering. الـ MAC driver بيـ probe مرة تانية بعد ما MDIO driver يـ register.

**الدرس المستفاد**: الـ `of_mdio_find_bus()` بترجع NULL (مش error code) لو الـ bus مش موجود. الـ driver لازم يترجم ده لـ `-EPROBE_DEFER` وما يعملش print لـ error message عشان مش bug.

---

### سيناريو 5: Allwinner H616 Board Bring-up — PHY Vendor Data

**العنوان**: PHY يحتاج initialization خاصة من الـ DT

**السياق**: Custom board على H616 SoC (Allwinner). الـ PHY هو Motorcomm YT8531 وبيحتاج configuration خاصة لـ RGMII TX/RX delays من الـ DT.

**المشكلة**: الـ link بيطلع unstable: dropped packets بشكل متكرر، link بتـ drop وبتـ come back.

**التحليل**:

الـ RGMII timing delays لازم تتضبط. الـ PHY driver بيقرأها من DT:

```dts
mdio {
    phy0: ethernet-phy@1 {
        reg = <1>;
        compatible = "motorcomm,yt8531";

        /* TX delay: 0.3ns steps */
        motorcomm,tx-clk-delay = <0x1f>;
        /* RX delay */
        motorcomm,rx-clk-delay = <0x1f>;

        /* من الـ standard properties */
        rx-internal-delay-ps = <2000>;
        tx-internal-delay-ps = <2000>;
    };
};

ethernet@5020000 {
    phy-handle = <&phy0>;
    phy-mode = "rgmii-id"; /* "-id" = internal delays */
};
```

لما `of_mdiobus_register()` بيعمل populate:
1. بيقرأ الـ child node `ethernet-phy@1`
2. بيستدعي `of_mdio_parse_addr()` → address = 1
3. `of_mdiobus_child_is_phy()` → `true` (compatible "motorcomm,yt8531" فيه "phy" implicitly via the compatible matching)
4. `of_mdiobus_phy_device_register()` بيسجل الـ phy مع DT node pointer
5. لما PHY driver بيـ bind، بيقرأ الـ vendor properties من `phydev->mdio.dev.of_node`
6. بيكتبها في PHY registers عشان يضبط الـ timing

```c
/* في PHY driver probe callback: */
struct device_node *np = phydev->mdio.dev.of_node;
u32 tx_delay, rx_delay;

of_property_read_u32(np, "motorcomm,tx-clk-delay", &tx_delay);
of_property_read_u32(np, "motorcomm,rx-clk-delay", &rx_delay);

/* كتابة القيم في PHY registers */
phy_write(phydev, YT8531_RGMII_TX_DELAY_REG, tx_delay);
phy_write(phydev, YT8531_RGMII_RX_DELAY_REG, rx_delay);
```

**الحل**: إضافة الـ vendor-specific DT properties، وتأكد إن `phy-mode = "rgmii-id"` عشان الـ delays الداخلية تُفعَّل.

```bash
# تحقق من الـ timing بعد التعديل:
ethtool -S eth0 | grep error  # ← لازم يكون zero
ping -f 192.168.1.1 -c 1000   # ← ما يكونش في packet loss
```

**الدرس المستفاد**: الـ `of_mdiobus_phy_device_register()` بيربط الـ DT node بالـ `phy_device`، فالـ PHY driver يقدر يقرأ vendor properties. الـ timing mismatches في RGMII غالباً بتظهر كـ intermittent packet loss.
## Phase 7: مصادر ومراجع

### LWN.net Articles

| المقالة | الرابط | الأهمية |
|---------|--------|---------|
| Add common OF device tree support for MDIO busses | [lwn.net/Articles/326450/](https://lwn.net/Articles/326450/) | الـ original patch اللي أضاف of_mdio |
| Support MDIO devices | [lwn.net/Articles/670191/](https://lwn.net/Articles/670191/) | إضافة non-PHY MDIO device support |
| net: mdio: Add netlink interface | [lwn.net/Articles/925483/](https://lwn.net/Articles/925483/) | إضافة netlink لـ MDIO operations |
| Add support for Broadcom iProc MDIO | [lwn.net/Articles/659127/](https://lwn.net/Articles/659127/) | مثال على MDIO controller driver |
| net: macb: Add mdio driver | [lwn.net/Articles/650897/](https://lwn.net/Articles/650897/) | مثال على multi-PHY MDIO sharing |
| ELCE: Grant Likely on device trees | [lwn.net/Articles/414016/](https://lwn.net/Articles/414016/) | شرح مبسط للعلاقة بين DT وـ Ethernet PHY |

---

### الـ Kernel Documentation الرسمية

```
Documentation/networking/phy.rst
    الـ main documentation للـ PHY library وـ MDIO subsystem

Documentation/devicetree/bindings/net/mdio.yaml
    الـ base DT binding schema للـ MDIO bus

Documentation/devicetree/bindings/net/ethernet-phy.yaml
    الـ DT binding schema للـ PHY devices

Documentation/devicetree/bindings/net/ethernet-controller.yaml
    شامل: phy-handle, phy-mode, fixed-link properties

Documentation/ABI/testing/sysfs-class-net-phydev
    الـ sysfs attributes للـ PHY devices
```

---

### الـ Source Files المرجعية

| الملف | الدور |
|-------|-------|
| `drivers/net/mdio/of_mdio.c` | التنفيذ الكامل للـ `of_mdio.h` API |
| `drivers/net/mdio/fwnode_mdio.c` | نسخة ACPI/fwnode من نفس الـ API |
| `drivers/net/phy/phy_device.c` | `phy_device` lifecycle |
| `drivers/net/phy/phy.c` | PHY state machine |
| `drivers/net/phy/fixed_phy.c` | الـ fixed-link implementation |
| `net/core/of_net.c` | `of_get_phy_mode()` وغيره |
| `drivers/net/ethernet/stmicro/stmmac/stmmac_main.c` | مثال على استخدام of_mdio API |
| `drivers/net/ethernet/freescale/fec_main.c` | مثال آخر |

---

### Relevant Kernel Commits

```
# الـ original of_mdio commit (2009)
git log --oneline -- drivers/net/mdio/of_mdio.c | tail -20

# commit مهم: إضافة devm_of_mdiobus_register
# كلمة البحث: "devm_of_mdiobus_register"
git log --all --oneline --grep="devm_of_mdiobus_register"

# commit إضافة fixed-link support
git log --all --oneline --grep="of_phy_register_fixed_link"
```

---

### الكتب الموصى بها

| الكتاب | المؤلف | الأهمية للموضوع |
|--------|--------|-----------------|
| **Linux Device Drivers (LDD3)** | Rubini, Corbet, Hartman | الفصل الخاص بـ network drivers (أساسي) |
| **Linux Kernel Development** | Robert Love | فهم الـ device model وـ sysfs |
| **Embedded Linux Primer** | Christopher Hallinan | فصل كامل عن Device Tree وـ networking |
| **Professional Linux Kernel Architecture** | Mauerer | عمق في الـ network subsystem |

---

### Search Terms مفيدة

```
# للبحث في kernel mailing list
site:lore.kernel.org "of_mdiobus_register"
site:lore.kernel.org "of_phy_connect"
site:lore.kernel.org "fixed-link" ethernet

# للبحث في الـ documentation
site:kernel.org/doc "mdio device tree"
site:kernel.org/doc "phy-handle"

# للبحث في الـ bootlin elixir (code cross-reference)
elixir.bootlin.com/linux/latest/source/drivers/net/mdio/of_mdio.c

# منتديات وـ Q&A
site:stackoverflow.com "of_mdiobus_register" linux kernel
```

---

### الـ DT Bindings المرجعية (مسارات داخل الـ kernel)

```
# MDIO bus (الـ parent)
Documentation/devicetree/bindings/net/mdio.yaml

# PHY devices (الـ children)
Documentation/devicetree/bindings/net/ethernet-phy.yaml

# Fixed link
Documentation/devicetree/bindings/net/fixed-link.yaml

# Marvell PHYs
Documentation/devicetree/bindings/net/marvell-phy.txt

# Realtek PHYs
Documentation/devicetree/bindings/net/realtek,rtl82xx.yaml

# Texas Instruments CPSW MDIO
Documentation/devicetree/bindings/net/ti,davinci-mdio.yaml
```

---

### eLinux.org Resources

- [elinux.org/Main_Page](https://elinux.org/Main_Page) — الـ main page، يحتوي على أقسام للـ embedded networking
- [elinux.org/Hack_A10_devices](https://elinux.org/Hack_A10_devices) — مثال على Allwinner MDIO/Ethernet bringup
- [elinux.org/BeagleBoard_Zippy](https://elinux.org/BeagleBoard_Zippy) — مثال على external Ethernet على BeagleBoard
## Phase 8: Writing simple module

### الهدف

بنعمل kernel module بيستخدم **kprobe** على `__of_mdiobus_register()`. الدالة دي هي الـ real entry point لكل MDIO bus registration من الـ Device Tree. لما أي driver يسجل MDIO bus من DT، الـ module بتاعنا هيطبع معلومات عن الـ bus والـ DT node.

اخترنا `__of_mdiobus_register()` عشان:
- exported symbol ومفيش direct hook mechanism تاني ليه
- بتُستدعى مرة واحدة بس لكل bus → مناسبة للـ monitoring
- بتمر بـ `mii_bus` و `device_node` → فيها معلومات مفيدة

---

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * of_mdio_probe_monitor.c
 *
 * kprobe on __of_mdiobus_register() to log MDIO bus registration events.
 * Useful for debugging DT-based MDIO bus bringup.
 *
 * Author: Kernel Debug Example
 */

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/kprobes.h>
#include <linux/phy.h>         /* struct mii_bus */
#include <linux/of.h>          /* struct device_node */
#include <linux/of_mdio.h>     /* the subsystem we're monitoring */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Debug Example");
MODULE_DESCRIPTION("kprobe monitor for __of_mdiobus_register — logs MDIO bus DT bringup");

/* ───────────────────────────────────────────────────────────────
 * الـ kprobe struct — بيحدد الـ function اللي بنـ hook عليها.
 * بنستخدم symbol_name بدل address عشان portable على أي kernel.
 * ─────────────────────────────────────────────────────────────── */
static struct kprobe kp = {
    .symbol_name = "__of_mdiobus_register",
};

/*
 * pre_handler: بتتنفذ قبل ما __of_mdiobus_register تبدأ.
 * الـ regs بتحتوي على الـ function arguments على الـ architecture.
 *
 * signature الـ function الأصلية:
 *   int __of_mdiobus_register(struct mii_bus *mdio,
 *                             struct device_node *np,
 *                             struct module *owner);
 *
 * على x86_64: arg1 = rdi, arg2 = rsi, arg3 = rdx
 * على ARM64:  arg1 = x0,  arg2 = x1,  arg3 = x2
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* ───────────────────────────────────────────────────────────
     * استخراج الـ arguments من الـ registers.
     * استخدمنا regs_get_kernel_argument() لأنه portable
     * بين x86_64 وـ ARM64 وـ RISC-V.
     * ─────────────────────────────────────────────────────────── */
    struct mii_bus *mdio = (struct mii_bus *)regs_get_kernel_argument(regs, 0);
    struct device_node *np = (struct device_node *)regs_get_kernel_argument(regs, 1);

    /* ───────────────────────────────────────────────────────────
     * Safety checks: الـ pointers ممكن يكونوا NULL في edge cases.
     * ─────────────────────────────────────────────────────────── */
    if (!mdio || !np) {
        pr_info("of_mdio_monitor: __of_mdiobus_register called with NULL args!\n");
        return 0;
    }

    /* ───────────────────────────────────────────────────────────
     * طبع معلومات الـ bus والـ DT node.
     * الـ bus->name بيدي اسم human-readable للـ bus.
     * الـ bus->id بيدي الـ unique identifier.
     * الـ np->full_name بيدي المسار الكامل في الـ DT.
     * الـ np->name بيدي الاسم القصير.
     * ─────────────────────────────────────────────────────────── */
    pr_info("of_mdio_monitor: [ENTRY] __of_mdiobus_register called\n");
    pr_info("of_mdio_monitor:   mii_bus name  = '%s'\n",
            mdio->name ? mdio->name : "(null)");
    pr_info("of_mdio_monitor:   mii_bus id    = '%s'\n", mdio->id);
    pr_info("of_mdio_monitor:   DT node name  = '%s'\n",
            np->name ? np->name : "(null)");
    pr_info("of_mdio_monitor:   DT full_name  = '%s'\n",
            np->full_name ? np->full_name : "(null)");
    pr_info("of_mdio_monitor:   phy_mask      = 0x%08x\n", mdio->phy_mask);

    /* ───────────────────────────────────────────────────────────
     * طبع الـ parent device name لو موجود.
     * الـ parent هو الـ MAC controller اللي عمل alloc للـ bus.
     * ─────────────────────────────────────────────────────────── */
    if (mdio->parent)
        pr_info("of_mdio_monitor:   parent device = '%s'\n",
                dev_name(mdio->parent));
    else
        pr_info("of_mdio_monitor:   parent device = (none)\n");

    return 0; /* 0 = تابع التنفيذ الطبيعي */
}

/*
 * post_handler: بتتنفذ بعد ما __of_mdiobus_register ترجع.
 * بنطبع مجرد confirmation إن الـ registration اتمت.
 * الـ flags هي processor flags بعد الـ function.
 */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /* ───────────────────────────────────────────────────────────
     * الـ return value بيكون في رجستر الـ return value:
     * x86_64: rax, ARM64: x0
     * نستخدمه عشان نعرف هل الـ registration نجحت أم لا.
     * ─────────────────────────────────────────────────────────── */
    long ret = regs_return_value(regs);

    if (ret == 0)
        pr_info("of_mdio_monitor: [EXIT]  __of_mdiobus_register succeeded (ret=0)\n");
    else
        pr_info("of_mdio_monitor: [EXIT]  __of_mdiobus_register FAILED (ret=%ld)\n", ret);
}

/* ───────────────────────────────────────────────────────────────
 * module_init: بنسجل الـ kprobe هنا.
 * لو الـ symbol مش موجود (kernel config مش enabled),
 * register_kprobe بترجع error وبنـ bail out.
 * ─────────────────────────────────────────────────────────────── */
static int __init of_mdio_monitor_init(void)
{
    int ret;

    kp.pre_handler = handler_pre;
    kp.post_handler = handler_post;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("of_mdio_monitor: register_kprobe failed: %d\n", ret);
        pr_err("of_mdio_monitor: Is CONFIG_OF_MDIO enabled?\n");
        return ret;
    }

    pr_info("of_mdio_monitor: kprobe registered on __of_mdiobus_register at %p\n",
            kp.addr);
    pr_info("of_mdio_monitor: Waiting for MDIO bus registrations...\n");
    return 0;
}

/* ───────────────────────────────────────────────────────────────
 * module_exit: إلغاء تسجيل الـ kprobe.
 * لازم يتعمل unregister قبل unload الـ module عشان
 * ما يحصلش kernel crash لو الـ function اتسمعت بعد الـ unload.
 * ─────────────────────────────────────────────────────────────── */
static void __exit of_mdio_monitor_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("of_mdio_monitor: kprobe unregistered. nmissed=%lu\n", kp.nmissed);
}

module_init(of_mdio_monitor_init);
module_exit(of_mdio_monitor_exit);
```

---

### Makefile

```makefile
obj-m += of_mdio_probe_monitor.o

# استبدل /lib/modules/... بالـ kernel source اللي بتبني ضده
all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

---

### طريقة الاستخدام

```bash
# بناء الـ module
make

# تحميل الـ module
sudo insmod of_mdio_probe_monitor.ko

# مشاهدة الـ output
sudo dmesg -w | grep of_mdio_monitor

# لتفعيل MDIO registration (مثلاً بـ rebind):
echo -n "ff540000.ethernet" | sudo tee /sys/bus/platform/drivers/dwmac-rk/unbind
echo -n "ff540000.ethernet" | sudo tee /sys/bus/platform/drivers/dwmac-rk/bind

# مثال output متوقع:
# [  123.456] of_mdio_monitor: [ENTRY] __of_mdiobus_register called
# [  123.456] of_mdio_monitor:   mii_bus name  = 'stmmac'
# [  123.456] of_mdio_monitor:   mii_bus id    = 'stmmac-0'
# [  123.456] of_mdio_monitor:   DT node name  = 'mdio'
# [  123.456] of_mdio_monitor:   DT full_name  = '/soc/ethernet@ff540000/mdio@0'
# [  123.456] of_mdio_monitor:   phy_mask      = 0x00000000
# [  123.456] of_mdio_monitor:   parent device = 'ff540000.ethernet'
# [  123.457] of_mdio_monitor: [EXIT]  __of_mdiobus_register succeeded (ret=0)

# إزالة الـ module
sudo rmmod of_mdio_probe_monitor
```

---

### شرح كل جزء

**`struct kprobe kp`**: بيحدد الـ function اللي بنـ hook عليها بالاسم. الـ kernel بيحول الاسم لـ address أوتوماتيك عند الـ registration.

**`handler_pre`**: بتتنفذ قبل الـ function. بنستخرج الـ arguments من الـ registers باستخدام `regs_get_kernel_argument()` اللي portable على كل architectures.

**`handler_post`**: بتتنفذ بعد الـ function. بنقرأ الـ return value من `regs_return_value()` عشان نعرف هل الـ registration نجحت.

**`regs_get_kernel_argument(regs, N)`**: دالة kernel معمولها عشان تستخرج الـ Nth argument من الـ function call بشكل portable. أفضل من الوصول المباشر لـ `regs->rdi` أو `regs->x0`.

**`kp.nmissed`**: عدد المرات اللي الـ kprobe مكانتش ready لما الـ function اتسمعت. مفيدة لـ debugging الـ probe نفسه.
