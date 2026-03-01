## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي بيتبع له الملف

الملف ده جزء من **BROADCOM GENET ETHERNET DRIVER** subsystem — المسؤول عن تشغيل شرائح الـ Ethernet الخاصة بـ Broadcom (زي BCM7xxx و BCM6xxx) في الـ Linux kernel.

---

### الـ MDIO إيه ده أصلاً؟

تخيل إن عندك كارت شبكة (Ethernet MAC) وعندك "كابلة" بتوصله بالسلك الفعلي — الكابلة دي اسمها **PHY** (Physical Layer chip). الـ PHY هو الشريحة الصغيرة اللي بتحول الـ bits الرقمية لإشارات كهربية على السلك.

طيب، اللي بيربط الـ MAC بالـ PHY إنه يتكلم معاه ويديه أوامر (فين المشكلة؟ هل الـ link شغال؟ بتشتغل بكام Mbps؟) — ده بيحصل عن طريق **MDIO bus** (Management Data Input/Output).

الـ MDIO زي كابل USB صغير بين الـ MAC والـ PHY — بيبعت ويستقبل أوامر إدارة وإعداد.

---

### الـ UniMAC إيه؟

**UniMAC** اختصار لـ Universal MAC — ده الـ Ethernet MAC controller بتاع Broadcom المدمج في شرائحهم (BCM7xxx وغيرها). الـ UniMAC فيه جوا controller خاص بيتحكم في الـ MDIO bus — واسمه **UniMAC MDIO controller**.

---

### الملف ده بيعمل إيه بالظبط؟

الملف `include/linux/platform_data/mdio-bcm-unimac.h` هو **platform data header** — يعني بيعرّف الـ struct اللي بتحمل الإعدادات اللي المحتاج تعديها لما بيشتغل الـ driver على platform مش بيستخدم Device Tree (أو لما الـ parent driver محتاج يمرر بيانات مخصوصة للـ child driver).

ببساطة: الـ driver بتاع الـ UniMAC MDIO (`mdio-bcm-unimac.c`) لازم حد يقوله:
- فين الـ PHYs الموجودة (mask)؟
- إيه الـ function اللي يستنى بيها لما MDIO transaction تخلص؟
- إيه اسم الـ bus؟
- إيه الـ clock؟

الـ struct `unimac_mdio_pdata` دي هي الجواب على الأسئلة دي.

---

### القصة الكاملة

**السيناريو:** شريحة BCM7xxx فيها UniMAC داخلي. الـ GENET driver (اللي بيشغل الـ Ethernet) محتاج يتحدث مع الـ PHY عن طريق MDIO.

**المشكلة:** الـ MDIO controller هو جزء من نفس الـ register space بتاع الـ UniMAC MAC، بس بيتعامل كـ platform device مستقل في الـ kernel عشان الـ separation of concerns.

**الحل:** الـ GENET driver (`bcmmii.c`) بينشئ platform device جديد للـ MDIO controller، وبيملأ struct من النوع `unimac_mdio_pdata` بالبيانات المطلوبة:

```c
/* bcmmii.c - GENET يملا البيانات ويمررها للـ MDIO driver */
struct unimac_mdio_pdata ppd;

ppd.wait_func      = bcmgenet_mii_wait;   // استنى interrupt بدل polling
ppd.wait_func_data = priv;                // بيانات الـ wait function
ppd.bus_name       = "bcmgenet MII bus";  // اسم الـ bus
ppd.clk            = priv->clk;           // الـ clock المشترك
```

**ليه الـ `wait_func` مهمة؟** لأن الـ GENET driver عنده interrupt بيجي لما MDIO transaction تخلص، فبدل ما الـ MDIO driver يعمل polling (busy-wait)، بيستخدم الـ `wait_func` اللي بيمررها الـ parent عشان يستنى الـ interrupt — أكفأ وأسرع.

---

### محتوى الملف بالتفصيل

```c
struct unimac_mdio_pdata {
    u32  phy_mask;            // mask للـ PHYs الموجودة على الـ bus
    int  (*wait_func)(void *data); // pointer لـ function بتستنى نهاية الـ transaction
    void *wait_func_data;     // الـ data اللي بتتبعت للـ wait_func
    const char *bus_name;     // اسم الـ MDIO bus (يظهر في /sys)
    struct clk *clk;          // الـ reference clock للـ MDIO controller
};

#define UNIMAC_MDIO_DRV_NAME  "unimac-mdio"  // اسم الـ platform driver
```

الـ struct صغيرة جداً لأن دورها واضح ومحدد: بس توصل configuration من الـ parent driver للـ child driver.

---

### العلاقة بين الملفات

```
bcmgenet.c / bcmmii.c          mdio-bcm-unimac.c
(GENET Ethernet Driver)   -->  (UniMAC MDIO Driver)
      |                               |
      | يملا unimac_mdio_pdata        | يقرأ platform_data
      | وينشئ platform_device        | ويشغل MDIO transactions
      |                               |
      +------- mdio-bcm-unimac.h ----+
               (الملف ده: struct + macro)
                        |
                        v
                    PHY chips
               (BCM54xx, BCM53125, ...)
```

---

### الملفات المهمة في الـ Subsystem

| الملف | الدور |
|-------|-------|
| `include/linux/platform_data/mdio-bcm-unimac.h` | تعريف `struct unimac_mdio_pdata` — الملف ده |
| `drivers/net/mdio/mdio-bcm-unimac.c` | الـ driver الفعلي للـ UniMAC MDIO controller |
| `drivers/net/ethernet/broadcom/genet/bcmgenet.c` | الـ GENET Ethernet driver الرئيسي |
| `drivers/net/ethernet/broadcom/genet/bcmmii.c` | الجزء المسؤول عن إعداد الـ MDIO/PHY في GENET |
| `drivers/net/ethernet/broadcom/genet/bcmgenet.h` | تعريفات الـ GENET driver الداخلية |
| `drivers/net/ethernet/broadcom/unimac.h` | register offsets للـ UniMAC MAC controller |
| `Documentation/devicetree/bindings/net/brcm,unimac-mdio.yaml` | Device Tree binding للـ MDIO controller |

---

### ملفات تانية مفيدة للقارئ

- **`include/linux/phy.h`** — تعريف `struct mii_bus` و PHY subsystem APIs
- **`include/linux/mdio.h`** — تعريفات الـ MDIO protocol نفسه (C22/C45)
- **`drivers/net/mdio/`** — كل الـ MDIO bus controllers في الـ kernel
## Phase 2: شرح الـ MDIO / UniMAC-MDIO Framework

### المشكلة — ليه الـ Subsystem ده موجود أصلاً؟

في أي SoC فيه Ethernet، لازم الـ MAC (Media Access Controller) يتكلم مع الـ PHY chip (الـ physical layer). الـ PHY هو الـ chip اللي بيحول الـ digital bits لـ signals كهربية على الكابل.

الـ **MDIO (Management Data Input/Output)** هو بروتوكول سيريال بسيط (2-wire: MDC clock + MDIO data) بيتيح للـ MAC إنه يقرا ويكتب في registers الـ PHY — زي يقرا link speed، يكتب auto-negotiation settings، يعرف في إيه error.

**المشكلة الفعلية:**
- كل SoC manufacturer بيعمل MDIO controller مختلف في الـ hardware.
- في نفس الوقت، الـ PHY drivers (زي `broadcom.c`, `marvell.c`) محتاجين يتكلموا مع الـ PHY بأوامر MDIO موحدة بصرف النظر عن الـ SoC.
- لو مفيش abstraction layer، كل PHY driver هيتكلم مع الـ hardware مباشرة ← chaos.

---

### الحل — ازاي الـ Kernel بيتعامل مع المشكلة؟

الـ kernel بيعمل **فصل واضح**:

| Layer | المسؤولية |
|-------|-----------|
| **MDIO Bus Driver** | بيتحكم في الـ hardware registers للـ MDIO controller |
| **`struct mii_bus`** | الـ abstraction اللي بتوحد كل الـ MDIO buses |
| **PHY Driver** | بيكلم `mii_bus` بدون ما يعرف إيه الـ hardware تحته |
| **Ethernet MAC Driver** | بيطلب scan للـ PHYs الموجودة على الـ bus |

الـ **UniMAC-MDIO** تحديداً هو الـ MDIO controller اللي Broadcom بيبنيه جوا الـ UniMAC (Unified MAC) IP block الموجود في chips زي BCM7xxx, BCM6xxx, وأشهرهم في embedded Linux هو الـ **BCM2711** (Raspberry Pi 4).

---

### Real-World Analogy — المقارنة بالواقع

تخيل شبكة مصنع فيها:

- **مشرف عام (MAC Driver)**: عايز يتحكم في آلات مختلفة.
- **هاتف داخلي موحد (mii_bus)**: كل الاتصالات بتمر عليه.
- **سنترال خاص بكل قسم (MDIO Controller / UniMAC-MDIO)**: بيعرف يبعت الأوامر للـ hardware بتاعه.
- **آلات (PHY chips)**: عندها addresses (0-31)، كل آلة بتستجب لأمر معين.

الـ `unimac_mdio_pdata` هي زي **بطاقة تعريف السنترال** — بتقول للـ unimac driver: إنت اسمك إيه، أنهي PHYs تتجاهلهم، وامتى تعتبر الـ transaction خلصت (wait function).

الـ mapping:
- السنترال الفيزيائي = الـ UniMAC MDIO hardware registers
- بطاقة التعريف = `struct unimac_mdio_pdata`
- رقم الخط الداخلي = `phy_mask` (أنهي addresses موجودة)
- إشارة "الخط فاضي" = `wait_func` (بتعرف امتى الـ bus ينهي transaction)
- نوع الكابل الداخلي = `clk` (clock source للتزامن)

---

### Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Kernel Space                             │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              PHY Subsystem (drivers/net/phy/)            │   │
│  │                                                          │   │
│  │   ┌────────────┐  ┌────────────┐  ┌──────────────────┐  │   │
│  │   │broadcom.c  │  │marvell.c   │  │ other PHY drivers│  │   │
│  │   └─────┬──────┘  └─────┬──────┘  └────────┬─────────┘  │   │
│  │         └───────────────┴───────────────────┘            │   │
│  │                         │  mdiobus_read/write()          │   │
│  └─────────────────────────┼────────────────────────────────┘   │
│                             │                                    │
│  ┌─────────────────────────▼────────────────────────────────┐   │
│  │              struct mii_bus  (MDIO Bus Abstraction)       │   │
│  │  .read()  .write()  .reset()  .phy_mask  .priv           │   │
│  └─────────────────────────┬────────────────────────────────┘   │
│                             │  function pointers                 │
│  ┌─────────────────────────▼────────────────────────────────┐   │
│  │          mdio-bcm-unimac.c  (Platform Driver)            │   │
│  │                                                          │   │
│  │  unimac_mdio_read()  ──► MDIO_CMD register               │   │
│  │  unimac_mdio_write() ──► MDIO_CMD register               │   │
│  │  unimac_mdio_reset() ──► dummy BMSR reads (workaround)   │   │
│  │  wait_func()         ──► poll or wait_event_timeout      │   │
│  │                                                          │   │
│  │  ◄── configured via struct unimac_mdio_pdata ──►         │   │
│  └─────────────────────────┬────────────────────────────────┘   │
│                             │  ioremap + readl/writel            │
└─────────────────────────────┼───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│                    Hardware (SoC)                                │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │   UniMAC IP Block                                        │   │
│  │   ┌─────────────────────┐  ┌─────────────────────────┐  │   │
│  │   │  MDIO_CMD  (0x00)   │  │  MDIO_CFG  (0x04)       │  │   │
│  │   │  [START_BUSY][RD/WR]│  │  [C22/C45][CLK_DIV]     │  │   │
│  │   │  [PHY_ADDR][REG][D] │  │  [SUPP_PREAMBLE]        │  │   │
│  │   └─────────────────────┘  └─────────────────────────┘  │   │
│  └──────────────────────┬───────────────────────────────────┘   │
│                          │  MDC + MDIO pins                      │
│  ┌───────────────────────▼─────────────────────────────────┐    │
│  │   PHY Chips on Board  (addr 0..31)                      │    │
│  │   [PHY@0]  [PHY@1]  ...  [PHY@N]                        │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

---

### مين بيستخدم الـ UniMAC-MDIO؟

**الـ consumers الأساسيون:**

1. **bcmgenet** (`drivers/net/ethernet/broadcom/genet/`) — الـ Gigabit Ethernet MAC في Raspberry Pi 4. بياخد الـ MDIO registers من جوا الـ UMAC register space وبيعمل child platform_device للـ unimac-mdio driver.

2. **bcm_sf2** (`drivers/net/dsa/bcm_sf2.c`) — الـ Broadcom StarFighter 2 switch، بيبحث عن `brcm,unimac-mdio` في Device Tree.

3. **bgmac** (`drivers/net/ethernet/broadcom/bgmac*`) — MIPS-based Broadcom Ethernet MACs.

---

### الـ Core Abstraction — الفكرة المركزية

الفكرة الجوهرية هي الـ **`struct mii_bus`** — هي الـ bus object اللي بيمثل قناة MDIO واحدة. كل حاجة في الـ MDIO subsystem بتدور حواليها.

```c
struct mii_bus {
    /* Identity */
    const char *name;          /* "unimac MII bus" */
    char id[MII_BUS_ID_SIZE];  /* unique ID like "unimac-mdio-0" */
    void *priv;                /* points to unimac_mdio_priv */

    /* Operations — filled by the low-level driver */
    int (*read)(struct mii_bus *bus, int addr, int regnum);
    int (*write)(struct mii_bus *bus, int addr, int regnum, u16 val);
    int (*reset)(struct mii_bus *bus);

    /* PHY management */
    u32 phy_mask;              /* bitmask of PHY addrs to SKIP */
    struct mdio_device *mdio_map[PHY_MAX_ADDR]; /* discovered devices */

    /* Synchronization */
    struct mutex mdio_lock;    /* one transaction at a time */
    struct device dev;         /* embedded kernel device */
};
```

الـ **`struct unimac_mdio_pdata`** هي الـ **platform data** — الطريقة التقليدية (قبل Device Tree) لتمرير configuration من الـ board code أو من الـ parent driver (زي bcmgenet) للـ platform driver. دي مش جزء من الـ kernel framework نفسه، دي specific configuration لـ UniMAC driver.

```c
struct unimac_mdio_pdata {
    u32 phy_mask;                    /* أنهي PHY addresses تتجاهلهم */
    int (*wait_func)(void *data);    /* ازاي تعرف إن الـ transaction خلصت */
    void *wait_func_data;            /* data تتبعت لـ wait_func */
    const char *bus_name;            /* اسم الـ bus في sysfs */
    struct clk *clk;                 /* clock للـ MDIO interface */
};
```

---

### الـ wait_func — ليه موجودة؟

ده أهم field في الـ struct ولازم نشرحه كويس.

الـ MDIO transaction بتاخد وقت (حوالي 25 microsecond). بعد ما تكتب الـ command في `MDIO_CMD` وتضرب `MDIO_START_BUSY`، لازم تستنى لحد ما الـ hardware يخلص.

**الاستنى ده ممكن يتعمل بطريقتين:**

**1. Polling (الـ default):**
```c
static int unimac_mdio_poll(void *wait_func_data)
{
    struct unimac_mdio_priv *priv = wait_func_data;
    /* delay 30us then poll every 2ms, timeout after 100ms */
    return read_poll_timeout(unimac_mdio_readl, val,
                             !(val & MDIO_START_BUSY),
                             2000, 100000, false, priv, MDIO_CMD);
}
```
الـ CPU يقعد يـ poll على الـ register. ده مناسب لما مفيش interrupt متاح.

**2. Interrupt-based wait (في bcmgenet):**
```c
/* من bcmmii.c */
static int bcmgenet_mii_wait(void *wait_func_data)
{
    struct bcmgenet_priv *priv = wait_func_data;
    /* ينام لحد ما الـ interrupt يصحيه */
    wait_event_timeout(priv->wq,
                       !(bcmgenet_umac_readl(priv, UMAC_MDIO_CMD)
                         & MDIO_START_BUSY),
                       HZ / 100);
    return 0;
}
```
الـ bcmgenet عنده interrupt بيتطلق لما الـ MDIO transaction تخلص، فبيستخدم `wait_event_timeout` بدل الـ polling. بيمرر الـ `wait_func` ده جوا الـ `unimac_mdio_pdata` عشان الـ unimac driver يستخدمه.

**الـ `wait_func_data`** هو الـ context pointer اللي بيتبعت لـ `wait_func` — في الحالة الأولى هو `unimac_mdio_priv*`، في الثانية هو `bcmgenet_priv*`.

---

### الـ phy_mask — اتجاهه عكسي مهم

لاحظ في الـ probe:

```c
/* pdata->phy_mask = 1 means "scan only PHY@0" */
bus->phy_mask = ~pdata->phy_mask;
/* مثال: pdata->phy_mask = 0x1 → bus->phy_mask = 0xFFFFFFFE */
/* يعني: تجاهل كل حاجة ماعدا addr=0              */
```

الـ `pdata->phy_mask` بيقول "أنهي PHYs تعمل scan عليهم" (set bit = include)، لكن `mii_bus->phy_mask` بيقول "أنهي PHYs تتجاهلهم" (set bit = skip). فالـ driver بيعمل NOT للـ mask عشان يحول بين التمثيلين.

---

### Platform Data vs Device Tree

الـ `unimac_mdio_pdata` بتُستخدم في حالتين:

```
┌─────────────────────────────────────────────────────────────────┐
│  Case 1: No Device Tree (MIPS boards, old code)                 │
│                                                                  │
│  board_init_code() {                                             │
│      struct unimac_mdio_pdata pd = {                            │
│          .phy_mask = 0xFF,  /* scan PHYs 0-7 */                 │
│          .bus_name = "my board MDIO",                           │
│      };                                                          │
│      platform_device_register_data(..., &pd, sizeof(pd));        │
│  }                                                               │
│  ↓                                                               │
│  unimac_mdio_probe() reads pdev->dev.platform_data               │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  Case 2: Parent driver creates child device (bcmgenet)          │
│                                                                  │
│  bcmgenet_mii_register() {                                       │
│      struct unimac_mdio_pdata ppd = {                            │
│          .wait_func = bcmgenet_mii_wait,  /* interrupt-based */  │
│          .wait_func_data = priv,                                  │
│          .bus_name = "bcmgenet MII bus",                         │
│          .clk = priv->clk,                                        │
│      };                                                          │
│      ppdev = platform_device_alloc(UNIMAC_MDIO_DRV_NAME, id);   │
│      platform_device_add_data(ppdev, &ppd, sizeof(ppd));         │
│      platform_device_add(ppdev);                                  │
│  }                                                               │
│  ↓                                                               │
│  unimac_mdio_probe() runs automatically by platform bus          │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  Case 3: Pure Device Tree (modern boards)                        │
│                                                                  │
│  DT node:                                                        │
│      mdio@... {                                                   │
│          compatible = "brcm,unimac-mdio";                        │
│          reg = <0x...>;                                           │
│          clock-frequency = <2500000>;                            │
│          #address-cells = <1>;                                   │
│          #size-cells = <0>;                                      │
│          phy@0 { ... };                                           │
│      };                                                          │
│                                                                  │
│  pdata = NULL → driver uses defaults (polling wait_func)         │
└─────────────────────────────────────────────────────────────────┘
```

---

### الـ struct unimac_mdio_priv — الـ Internal State

ده الـ struct الداخلي اللي بيعيش في الـ `mii_bus->priv`:

```c
struct unimac_mdio_priv {
    struct mii_bus  *mii_bus;       /* back pointer للـ bus */
    void __iomem    *base;          /* MDIO_CMD register base address */
    int (*wait_func)(void *data);   /* جه من pdata أو default polling */
    void            *wait_func_data;
    struct clk      *clk;           /* optional clock للـ gating */
    u32              clk_freq;      /* MDC target frequency */
};
```

العلاقة بين الـ structs:

```
platform_device
    └── pdev->dev.platform_data ──► struct unimac_mdio_pdata
                                        (input: من الـ board code أو parent driver)

platform_device
    └── platform_get_drvdata() ──► struct unimac_mdio_priv
                                        (internal: بيعيش طول عمر الـ driver)

struct unimac_mdio_priv
    └── .mii_bus ──► struct mii_bus
                         └── .priv ──► struct unimac_mdio_priv (back ptr)
                         └── .read  = unimac_mdio_read
                         └── .write = unimac_mdio_write
                         └── .reset = unimac_mdio_reset
```

---

### Hardware Registers — ازاي الـ MDIO Transaction بتتعمل

الـ UniMAC MDIO controller فيها 2 registers بس:

**MDIO_CMD (offset 0x00):**
```
Bit 29    : START_BUSY — اكتب 1 لبدء الـ transaction، بيتصفر لما يخلص
Bit 28    : READ_FAIL  — set by HW لو PHY مردش (no turn-around)
Bits 27-26: Operation — 10=READ, 01=WRITE
Bits 25-21: PHY address (0-31)
Bits 20-16: Register number (0-31)
Bits 15-0 : Data (write data أو read result)
```

**MDIO_CFG (offset 0x04):**
```
Bit 0     : C22/C45 mode — 1=Clause22, 0=Clause45
Bits 9-4  : CLK_DIV — MDIO clock = ref_clk / (2 * (DIV+1))
Bit 12    : SUPP_PREAMBLE — suppress 32-bit preamble
```

**مثال: قراءة register MII_BMSR من PHY@1:**
```c
/* 1. اكتب الأمر */
cmd = MDIO_RD | (1 << MDIO_PMD_SHIFT) | (MII_BMSR << MDIO_REG_SHIFT);
//  = (2 << 26) | (1 << 21) | (1 << 16)
unimac_mdio_writel(priv, cmd, MDIO_CMD);

/* 2. ابدأ الـ transaction */
reg = unimac_mdio_readl(priv, MDIO_CMD);
reg |= MDIO_START_BUSY;
unimac_mdio_writel(priv, reg, MDIO_CMD);

/* 3. استنى */
priv->wait_func(priv->wait_func_data);

/* 4. اقرا النتيجة */
cmd = unimac_mdio_readl(priv, MDIO_CMD);
result = cmd & 0xFFFF;   /* lower 16 bits = PHY register value */
```

---

### الـ BCM7xxx Workaround — مثال على Quirk Handling

في `unimac_mdio_reset()` فيه workaround مهم:

```c
static int unimac_mdio_reset(struct mii_bus *bus)
{
    /* بعض BCM7xxx PHYs بتفشل في أول transaction */
    /* الحل: اعمل dummy read من MII_BMSR قبل الـ real scan */
    for (addr = 0; addr < PHY_MAX_ADDR; addr++) {
        if (read_mask & 1 << addr)
            mdiobus_read(bus, addr, MII_BMSR);  /* dummy read */
    }
    return 0;
}
```

الـ `bus->reset` بيتتشال قبل `mdiobus_scan` جوا `mdiobus_register`. يعني قبل ما الـ kernel يبدأ يبحث عن PHYs، بيعمل warm-up read لكل PHY متوقع موجود.

---

### إيه اللي الـ Subsystem بيمتلكه vs اللي بيفوضه للـ Driver

| الـ MDIO Subsystem يمتلك | الـ UniMAC Driver يقرر |
|--------------------------|------------------------|
| `struct mii_bus` lifecycle (alloc/register/free) | كيفية الكتابة للـ hardware registers |
| PHY discovery (`mdiobus_scan`) | طريقة الانتظار (poll vs interrupt) |
| الـ mutex للـ bus serialization | الـ clock management |
| sysfs/devfs entries للـ bus والـ PHYs | الـ endianness handling (MIPS BE workaround) |
| PHY driver matching وـ probe | الـ BCM7xxx dummy read workaround |
| `phy_device` creation وـ registration | تحديد target MDC frequency من `clock-frequency` DT property |

---

### مفاهيم محتاج تعرفها قبل تكمل

- **`struct clk`** — الـ Clock Framework (CCF): الـ `clk` في `unimac_mdio_pdata` هو handle للـ reference clock. الـ UniMAC MDIO driver بيعمل `clk_prepare_enable` قبل كل transaction و`clk_disable_unprepare` بعدها — ده clock gating عشان توفير طاقة.

- **`platform_device` / `platform_driver`** — الـ Platform Bus: لما مفيش physical bus discoverable (زي PCI)، الـ kernel بيعمل virtual bus اسمه platform bus. الـ `UNIMAC_MDIO_DRV_NAME = "unimac-mdio"` هو الـ string اللي بيربط بين الـ device والـ driver على الـ platform bus.

- **`of_mdiobus_register`** — بياخد `mii_bus` وـ DT node، بيعمل scan للـ child nodes (`phy@N`) وبيعمل register `phy_device` لكل PHY لاقيه.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Macros — Cheatsheet

#### Hardware Register Map

| Register | Offset | الوصف |
|----------|--------|-------|
| `MDIO_CMD` | `0x00` | Command register — بيتحكم في العمليات |
| `MDIO_CFG` | `0x04` | Config register — إعداد clock والـ mode |

#### بتوع الـ MDIO_CMD (offset 0x00)

| Flag | Value | الوصف |
|------|-------|-------|
| `MDIO_START_BUSY` | `1 << 29` | Set = شغّل transaction، Busy = لسه بيشتغل |
| `MDIO_READ_FAIL`  | `1 << 28` | الـ PHY مردّش على read بشكل صح |
| `MDIO_RD`         | `2 << 26` | نوع العملية: Read |
| `MDIO_WR`         | `1 << 26` | نوع العملية: Write |
| `MDIO_PMD_SHIFT`  | `21`      | Shift لعنوان الـ PHY (bits 25:21) |
| `MDIO_PMD_MASK`   | `0x1F`    | Mask الـ 5 bits بتوع عنوان الـ PHY |
| `MDIO_REG_SHIFT`  | `16`      | Shift لرقم الـ register (bits 20:16) |
| `MDIO_REG_MASK`   | `0x1F`    | Mask الـ 5 bits بتوع رقم الـ register |
| data bits         | `0:15`    | الـ 16-bit data (read result أو write value) |

**مثال عملي** — build كلمة read command:
```c
cmd = MDIO_RD | (phy_id << 21) | (reg << 16);
/* للـ PHY addr=1, reg=0:
   MDIO_RD = 0x08000000
   phy    = 1 << 21 = 0x00200000
   reg    = 0 << 16 = 0x00000000
   cmd    = 0x08200000
*/
```

#### بتوع الـ MDIO_CFG (offset 0x04)

| Flag | Value | الوصف |
|------|-------|-------|
| `MDIO_C22`         | `1 << 0` | Enable Clause 22 (standard MII) |
| `MDIO_C45`         | `0`      | Enable Clause 45 (10G PHYs) |
| `MDIO_CLK_DIV_SHIFT` | `4`    | Shift لـ clock divider field |
| `MDIO_CLK_DIV_MASK`  | `0x3F` | 6-bit divider value |
| `MDIO_SUPP_PREAMBLE` | `1 << 12` | Suppress MDIO preamble bits |

**صيغة الـ clock divider:**
```
MDIO_CLK = ref_clk / (2 × (div + 1))
// لو ref = 250 MHz و MDIO_CLK = 2.5 MHz:
// div = (250_000_000 / (2 × 2_500_000)) - 1 = 49
```

#### الـ Driver Name Macro

| Macro | Value |
|-------|-------|
| `UNIMAC_MDIO_DRV_NAME` | `"unimac-mdio"` |

#### الـ of_device_id — Compatible Strings

| Compatible String | Hardware |
|-------------------|----------|
| `brcm,unimac-mdio` | UniMAC MDIO generic |
| `brcm,genet-mdio-v1..v5` | GENET Ethernet MAC embedded MDIO |
| `brcm,asp-v2.1/v2.2/v3.0-mdio` | ASP (Advanced Streaming Processor) |
| `brcm,bcm6846-mdio` | BCM6846 DSL SoC |

#### الـ mii_bus state enum

| State | المعنى |
|-------|--------|
| `MDIOBUS_ALLOCATED`   | تم alloc بس مش registered لسه |
| `MDIOBUS_REGISTERED`  | شغال وبيخدم requests |
| `MDIOBUS_UNREGISTERED`| اتفصل من الـ system |
| `MDIOBUS_RELEASED`    | الـ memory اتحررت |

---

### 1. الـ Structs المهمة

#### `struct unimac_mdio_pdata`
**الملف:** `include/linux/platform_data/mdio-bcm-unimac.h`

ده الـ **platform data** — الداتا اللي بيبعتها الـ board code أو الـ firmware للـ driver وقت الـ probe. بيُستخدم لما الـ driver بيُستخدم مع legacy platform devices (مش Device Tree).

| Field | Type | الوظيفة |
|-------|------|---------|
| `phy_mask` | `u32` | bitmask للـ PHY addresses اللي المفروض **تتعرف** (active = 1). الـ driver بيعمل complement ويحطه في `mii_bus->phy_mask` |
| `wait_func` | `int (*)(void *)` | callback بيانتظر اكتمال الـ MDIO transaction. لو NULL → الـ driver بيستخدم الـ built-in polling |
| `wait_func_data` | `void *` | الـ opaque pointer اللي بيتبعت للـ `wait_func` |
| `bus_name` | `const char *` | اسم friendly للـ MDIO bus (بيظهر في `/sys/bus/mdio_bus`) |
| `clk` | `struct clk *` | الـ clock handle. لو NULL → الـ driver بيجيب الـ clock بنفسه من الـ Device Tree |

**ملاحظة مهمة على `phy_mask`:** الـ pdata بيبعت mask للـ PHYs المطلوبة، بس الـ `mii_bus->phy_mask` هو الـ PHYs المتجاهلة — عشان كده الـ driver بيعمل:
```c
bus->phy_mask = ~pdata->phy_mask;  /* complement! */
```

---

#### `struct unimac_mdio_priv`
**الملف:** `drivers/net/mdio/mdio-bcm-unimac.c` (internal, مش exported)

ده الـ **private runtime state** للـ driver. بيتخزن في `mii_bus->priv`.

| Field | Type | الوظيفة |
|-------|------|---------|
| `mii_bus` | `struct mii_bus *` | back-pointer للـ generic MDIO bus object |
| `base` | `void __iomem *` | عنوان الـ hardware registers بعد الـ ioremap |
| `wait_func` | `int (*)(void *)` | نفس فكرة الـ pdata — إما `unimac_mdio_poll` أو custom من الـ platform |
| `wait_func_data` | `void *` | بيتبعت للـ wait_func، في الـ default case = `priv` نفسه |
| `clk` | `struct clk *` | الـ clock للـ MDIO controller — بيتـ enable/disable مع كل transaction |
| `clk_freq` | `u32` | الـ target MDIO clock frequency بالـ Hz (من `clock-frequency` في DT) |

---

#### `struct mii_bus`
**الملف:** `include/linux/phy.h`

ده الـ **core Linux MDIO bus abstraction**. كل MDIO controller في الـ system عنده instance منه.

| Field | Type | الوظيفة |
|-------|------|---------|
| `name` | `const char *` | اسم البص |
| `id[MII_BUS_ID_SIZE]` | `char[]` | unique ID، بيتبني من `pdev->name-pdev->id` |
| `priv` | `void *` | pointer على `unimac_mdio_priv` |
| `read` | `int (*)(bus, addr, reg)` | ← `unimac_mdio_read` |
| `write` | `int (*)(bus, addr, reg, val)` | ← `unimac_mdio_write` |
| `reset` | `int (*)(bus)` | ← `unimac_mdio_reset` |
| `mdio_lock` | `struct mutex` | **يحمي كل access للـ bus** — serialize الـ read/write |
| `parent` | `struct device *` | الـ platform device |
| `state` | `enum` | حالة الـ bus (ALLOCATED → REGISTERED ...) |
| `phy_mask` | `u32` | PHY addresses المتجاهلة في الـ scan |
| `phy_ignore_ta_mask` | `u32` | PHYs اللي بنتجاهل فيهم الـ READ_FAIL (workaround للـ BCM53125) |
| `mdio_map[]` | `struct mdio_device *[32]` | قائمة كل الـ PHY devices على البص |
| `irq[]` | `int[32]` | interrupt لكل PHY address |

---

### 2. رسم العلاقات بين الـ Structs

```
Platform Layer
═══════════════
  struct platform_device (pdev)
  ┌─────────────────────────────┐
  │ dev.platform_data ──────────┼──► struct unimac_mdio_pdata
  │ dev.of_node ────────────────┼──► Device Tree node
  └──────────────┬──────────────┘
                 │ probe()
                 ▼
Driver Runtime
═══════════════
  struct unimac_mdio_priv
  ┌──────────────────────────────────┐
  │ mii_bus ──────────────────────── ┼──► struct mii_bus
  │ base (void __iomem *)            │     ┌───────────────────────────┐
  │ wait_func()                      │     │ priv ──────────────────── ┼──► (back to priv)
  │ wait_func_data ──────────────────┼──►  │ read() = unimac_mdio_read │
  │ clk ─────────────────────────────┼──►  │ write()= unimac_mdio_write│
  │ clk_freq                         │     │ reset()= unimac_mdio_reset│
  └──────────────────────────────────┘     │ mdio_lock (mutex)         │
                                           │ mdio_map[32] ─────────────┼──► struct mdio_device[]
                                           │ parent ───────────────────┼──► &pdev->dev
                                           └───────────────────────────┘
Hardware Layer
═══════════════
  priv->base + 0x00 ──► MDIO_CMD register
  priv->base + 0x04 ──► MDIO_CFG register
```

---

### 3. دورة الحياة — Lifecycle Diagram

```
══════════════════════════════════════════════════════
                   DRIVER LIFECYCLE
══════════════════════════════════════════════════════

[Kernel Startup / Device Tree Probe]
         │
         ▼
unimac_mdio_probe(pdev)
         │
         ├─ devm_kzalloc → priv allocated
         │
         ├─ platform_get_resource → r (MMIO region)
         │
         ├─ devm_ioremap → priv->base mapped
         │
         ├─ of_property_read_u32("clock-frequency") → priv->clk_freq
         │
         ├─ mdiobus_alloc() → bus allocated
         │         State: MDIOBUS_ALLOCATED
         │
         ├─ [pdata path]              [DT path]
         │   bus->name = pdata->name   bus->name = "unimac MII bus"
         │   priv->wait_func = custom  priv->wait_func = unimac_mdio_poll
         │   priv->clk = pdata->clk   priv->clk = devm_clk_get_optional()
         │
         ├─ bus->read/write/reset = unimac_mdio_*
         │
         ├─ unimac_mdio_clk_set() → program MDIO_CFG divider
         │
         ├─ of_mdiobus_register(bus, np)
         │         │
         │         ├─ unimac_mdio_reset() ← dummy read workaround
         │         └─ mdiobus_scan → detect PHY devices
         │         State: MDIOBUS_REGISTERED
         │
         └─ platform_set_drvdata(pdev, priv)

[RUNTIME: repeated per network operation]
         │
         ▼
unimac_mdio_read() or unimac_mdio_write()
  (serialized by mii_bus->mdio_lock — held by caller mdiobus_read/write)
         │
         ├─ clk_prepare_enable(priv->clk)
         ├─ build cmd word
         ├─ unimac_mdio_writel(priv, cmd, MDIO_CMD)
         ├─ unimac_mdio_start() → set MDIO_START_BUSY
         ├─ priv->wait_func() → poll/wait until !BUSY
         └─ clk_disable_unprepare(priv->clk)

[SUSPEND/RESUME]
         │
  resume ▼
unimac_mdio_resume()
         └─ unimac_mdio_clk_set() → reprogram divider

[DRIVER UNLOAD / Device Removal]
         │
         ▼
unimac_mdio_remove(pdev)
         │
         ├─ mdiobus_unregister(priv->mii_bus)
         │         State: MDIOBUS_UNREGISTERED
         └─ mdiobus_free(priv->mii_bus)
                   State: MDIOBUS_RELEASED
                   (priv freed by devm automatically)
```

---

### 4. Call Flow Diagrams

#### 4.1 — Read Flow (الـ PHY driver بيعمل phy_read)

```
PHY driver
  phy_read(phydev, MII_BMSR)
    │
    ▼
phylib core
  mdiobus_read(bus, addr, reg)
    │  acquires bus->mdio_lock (mutex)
    ▼
  bus->read(bus, addr, reg)
    │
    ▼
unimac_mdio_read(bus, phy_id, reg)
    │
    ├─ clk_prepare_enable(priv->clk)        ← turn on MDIO clock
    │
    ├─ cmd = MDIO_RD | (phy_id<<21) | (reg<<16)
    │
    ├─ unimac_mdio_writel(priv, cmd, MDIO_CMD)   ← write command
    │         priv->base + 0x00 = cmd
    │
    ├─ unimac_mdio_start(priv)
    │         reg |= MDIO_START_BUSY         ← trigger hardware
    │         priv->base + 0x00 |= (1<<29)
    │
    ├─ priv->wait_func(priv->wait_func_data)
    │    └─ unimac_mdio_poll(priv)
    │         ├─ udelay(30)                  ← C22 = ~25us
    │         └─ read_poll_timeout(...)
    │               polls MDIO_CMD every 2ms
    │               timeout = 100ms
    │               until !(MDIO_CMD & MDIO_START_BUSY)
    │
    ├─ cmd = unimac_mdio_readl(priv, MDIO_CMD)
    │
    ├─ check MDIO_READ_FAIL (unless phy_ignore_ta_mask)
    │
    ├─ ret = cmd & 0xffff                    ← extract 16-bit data
    │
    └─ clk_disable_unprepare(priv->clk)      ← turn off clock

  mdiobus_read releases bus->mdio_lock
    │
    ▼
returns register value to phy_read caller
```

#### 4.2 — Write Flow

```
MAC driver
  mdiobus_write(bus, addr, reg, val)
    │  acquires bus->mdio_lock
    ▼
unimac_mdio_write(bus, phy_id, reg, val)
    │
    ├─ clk_prepare_enable()
    ├─ cmd = MDIO_WR | (phy_id<<21) | (reg<<16) | (val & 0xffff)
    ├─ writel cmd → MDIO_CMD
    ├─ unimac_mdio_start() → set START_BUSY
    ├─ wait_func() → poll until !BUSY
    └─ clk_disable_unprepare()
    (no READ_FAIL check — write لا بيرجع data)
```

#### 4.3 — Reset Workaround Flow

```
of_mdiobus_register(bus, np)
    │
    ├─ bus->reset(bus)          ← called BEFORE scan
    │       │
    │       ▼
    │  unimac_mdio_reset(bus)
    │       │
    │       ├─ [DT path]:
    │       │   for_each_available_child_of_node(np, child)
    │       │     addr = of_mdio_parse_addr(child)
    │       │     read_mask |= 1 << addr
    │       │
    │       ├─ [non-DT path]:
    │       │   read_mask = ~bus->phy_mask
    │       │
    │       └─ for addr in 0..PHY_MAX_ADDR:
    │               if (read_mask & (1<<addr))
    │                 mdiobus_read(bus, addr, MII_BMSR)  ← dummy read!
    │
    └─ mdiobus_scan() → proper PHY detection
```

#### 4.4 — Clock Setup Flow

```
unimac_mdio_probe()
    │
    └─ unimac_mdio_clk_set(priv)
            │
            ├─ if (!priv->clk_freq) → return 0  ← keep HW defaults
            │
            ├─ clk_prepare_enable(priv->clk)
            │
            ├─ rate = clk_get_rate(priv->clk)
            │   if (!rate) rate = 250_000_000   ← fallback
            │
            ├─ div = (rate / (2 * clk_freq)) - 1
            │
            ├─ if (div > 0x3F) → warn, skip      ← 6-bit overflow check
            │
            ├─ reg = readl(MDIO_CFG)
            │   reg &= ~(0x3F << 4)              ← clear old div
            │   reg |= div << 4                  ← set new div
            │   writel(reg, MDIO_CFG)
            │
            └─ clk_disable_unprepare(priv->clk)
```

---

### 5. استراتيجية الـ Locking

#### الـ mutex: `mii_bus->mdio_lock`

ده **الـ lock الوحيد** في هذا الـ driver، وهو مش بيتعمل lock فيه جوه الـ driver نفسه — ده مسؤولية الـ mdiobus core framework.

```
Thread A                           Thread B
  mdiobus_read(bus, 1, 0)           mdiobus_write(bus, 2, 0, val)
    mutex_lock(&bus->mdio_lock)       mutex_lock(&bus->mdio_lock)  ← BLOCKS
    → unimac_mdio_read()              ...waiting...
    → clk on, write cmd,
      start, poll, read result,
      clk off
    mutex_unlock(&bus->mdio_lock)
                                      ← unblocked
                                      → unimac_mdio_write()
                                      mutex_unlock()
```

**خلاصة الـ locking:**

| Resource | من بيحميه | نوع الـ lock |
|----------|-----------|-------------|
| Hardware registers (MDIO_CMD, MDIO_CFG) | `mii_bus->mdio_lock` | `struct mutex` |
| `priv->clk` (enable/disable) | محمي ضمنياً بالـ mdio_lock | — |
| `mii_bus->mdio_map[]` (PHY device list) | `mii_bus->mdio_lock` | `struct mutex` |
| `mii_bus->shared[]` | `mii_bus->shared_lock` | `struct mutex` (منفصل) |

#### ليه mutex وليه spinlock؟

الـ MDIO transactions بتاخد **~25-100 microseconds** (polling loop)، ده وقت طويل جداً لـ spinlock. الـ mutex بيخلي الـ thread ينام وإيده كده ما يضيعش CPU cycles.

#### لا يوجد lock ordering issues

الـ driver بسيط — lock واحد، ما فيش nested locking، ما فيش deadlock risk.

#### الـ clk و concurrency

`clk_prepare_enable` / `clk_disable_unprepare` هم thread-safe بذاتهم (الـ CCF بيعمل reference counting)، بس لأن الـ mdio_lock موجود، في practice الـ clk بيتـ enable من thread واحد في أي لحظة.
## Phase 4: شرح الـ Functions

الـ file ده (`mdio-bcm-unimac.h`) هو **platform data header** بحت — مش بيعرّف functions خالص، بيعرّف بس **struct** واحدة و **macro** واحدة بتمثلوا الـ platform data اللي بتتحط في الـ `platform_device` عشان الـ UniMAC MDIO driver يقدر يشتغل.

---

### ملخص الـ APIs والـ Data Structures

| العنصر | النوع | الغرض |
|--------|-------|--------|
| `struct unimac_mdio_pdata` | struct | الـ platform data اللي بتتدي للـ UniMAC MDIO driver |
| `phy_mask` | `u32` field | bitmask بيحدد الـ PHY addresses المستبعدة من الـ scan |
| `wait_func` | function pointer field | callback بيستنى الـ MDIO transaction تخلص |
| `wait_func_data` | `void *` field | الـ opaque data اللي بتتبعت لـ `wait_func` |
| `bus_name` | `const char *` field | اسم الـ MDIO bus |
| `clk` | `struct clk *` field | الـ clock handle بتاع الـ MDIO controller |
| `UNIMAC_MDIO_DRV_NAME` | macro | اسم الـ platform driver اللي بيتطابق مع الـ `platform_device` |

---

### Group 1: Platform Data Structure — `struct unimac_mdio_pdata`

الـ group ده عبارة عن struct واحدة بس، وده طبيعي في platform data headers — الفكرة إن الـ board code أو الـ Device Tree parsing code بتملا الـ struct دي وتحطها في `platform_device.dev.platform_data` قبل ما الـ driver يتشغّل.

---

#### `struct unimac_mdio_pdata`

```c
struct unimac_mdio_pdata {
    u32   phy_mask;                  /* bitmask: PHYs to exclude from bus scan */
    int (*wait_func)(void *data);    /* callback: poll/wait for MDIO done */
    void *wait_func_data;            /* opaque arg passed to wait_func */
    const char *bus_name;            /* human-readable MDIO bus name */
    struct clk *clk;                 /* optional clock for the controller */
};
```

**الغرض:**
الـ struct دي هي **الواجهة الوحيدة** بين الـ board/platform code والـ `unimac-mdio` driver. الـ driver بيجيبها عن طريق `dev_get_platdata()` أو من الـ Device Tree بعد parsing.

---

##### الـ Fields بالتفصيل

---

###### `u32 phy_mask`

- **النوع:** `u32` bitmask
- **الغرض:** بيحدد الـ PHY addresses اللي المفروض الـ MDIO bus scan **تتجاهلها**.
- **الميكانيزم:** الـ MDIO bus بيعمل scan على 32 address (0–31). لو الـ bit رقم N اتعمله set في `phy_mask`، الـ address N بيتعمله skip.
- **مثال:** لو عندك PHY على address 0 بس وعايز تتجاهل الباقي:

```c
/* Exclude all PHY addresses except address 0 */
pdata.phy_mask = ~BIT(0);  /* = 0xFFFFFFFE */
```

- **بيتبعت لـ:** `mdiobus_alloc()` ثم بيتحط في `mii_bus->phy_mask` جوه الـ driver.
- **لازم تعرف:** الـ value دي بتاتي من الـ Device Tree عادةً تحت property اسمها `phy-mask`، أو بتتعمل hardcode في الـ board file.

---

###### `int (*wait_func)(void *data)`

- **النوع:** function pointer بيرجع `int`
- **الغرض:** **الـ custom completion callback** — بيتكلمه الـ driver بعد ما يبعت MDIO transaction عشان يستنى الـ hardware يخلص.
- **السيناريو:** في بعض الـ SoCs، الـ UniMAC MDIO controller بيشتغل مع interrupt أو مع external synchronization mechanism. بدل ما الـ driver يعمل busy-wait hardcoded، بيديك hook تحط فيه الـ wait logic بتاعتك.
- **Return value:** `0` = نجح (transaction خلصت)، قيمة سلبية = error (timeout مثلاً).
- **مثال:**

```c
/* Custom wait: poll a register for completion */
static int my_mdio_wait(void *data)
{
    struct my_priv *priv = data;
    int timeout = 1000;

    while (timeout--) {
        if (readl(priv->base + MDIO_STAT) & MDIO_DONE)
            return 0;
        udelay(1);
    }
    return -ETIMEDOUT;
}
```

- **لو `NULL`:** الـ driver بيرجع للـ default polling logic الجوّاني بتاعه.
- **Caller context:** بيتكلّم من سياق الـ MDIO read/write operations، اللي ممكن تبقى في sleepable context (kernel thread) أو atomic حسب الـ `mdiobus` flags.

---

###### `void *wait_func_data`

- **النوع:** `void *` — opaque pointer
- **الغرض:** الـ data اللي بتتبعت كـ argument لـ `wait_func(data)`.
- **الاستخدام:** عادةً بيكون pointer لـ private struct بتاع الـ platform أو الـ controller، زي مثلاً الـ base address أو الـ completion struct.
- **مثال:**

```c
pdata.wait_func      = my_mdio_wait;
pdata.wait_func_data = &my_controller_priv;  /* passed as-is to wait_func */
```

- **ملاحظة:** الـ driver مش بيعمل dereference للـ pointer ده — بيبعته كـ opaque value. الـ type safety مسؤولية الـ board code.

---

###### `const char *bus_name`

- **النوع:** `const char *`
- **الغرض:** الاسم اللي بيتسجّل بيه الـ MDIO bus في الـ kernel (`mii_bus->name`).
- **الاستخدام:** بيظهر في `/sys/bus/mdio_bus/` وفي الـ kernel logs.
- **لو `NULL`:** الـ driver بيستخدم اسم default (عادةً `"unimac MII"` أو اسم مشتق من الـ device name).
- **مثال:**

```c
pdata.bus_name = "bcm-sf2-mdio";
```

---

###### `struct clk *clk`

- **النوع:** `struct clk *` — pointer للـ Common Clock Framework (CCF) clock handle
- **الغرض:** الـ clock بتاع الـ MDIO controller. الـ driver بيعمله `clk_prepare_enable()` قبل استخدام الـ hardware وبيعمله `clk_disable_unprepare()` في الـ cleanup.
- **لو `NULL`:** معناها الـ clock مش محتاج تحكم صريح (اتعمله enable قبل كده أو مش محتاج).
- **مثال:**

```c
pdata.clk = devm_clk_get(&pdev->dev, "mdio");
if (IS_ERR(pdata.clk))
    pdata.clk = NULL;  /* optional clock, continue without it */
```

- **لازم تعرف:** لو الـ driver اتبنى بدون CCF support، الـ field ده ممكن يتجاهله. الـ driver المسؤول هو `drivers/net/mdio/mdio-bcm-unimac.c`.

---

### Group 2: Driver Name Macro — `UNIMAC_MDIO_DRV_NAME`

```c
#define UNIMAC_MDIO_DRV_NAME    "unimac-mdio"
```

**الغرض:**
الـ macro ده هو **الرابط** بين الـ `platform_device` اللي بتسجّله الـ board code والـ `platform_driver` اللي موجود في الـ driver. الـ kernel بيعمل match بين `platform_device.name` و `platform_driver.driver.name` — لازم يبقوا متطابقين.

**الاستخدام في الـ board code:**

```c
static struct platform_device unimac_mdio_dev = {
    .name = UNIMAC_MDIO_DRV_NAME,   /* "unimac-mdio" */
    .id   = -1,
    .dev  = {
        .platform_data = &my_unimac_pdata,
    },
};
```

**الاستخدام في الـ driver:**

```c
/* In drivers/net/mdio/mdio-bcm-unimac.c */
static struct platform_driver unimac_mdio_driver = {
    .driver = {
        .name = UNIMAC_MDIO_DRV_NAME,  /* must match device name */
        ...
    },
    ...
};
```

**لازم تعرف:** استخدام الـ macro في الاتنين بدل hardcoding `"unimac-mdio"` بيضمن إن أي rename مستقبلي بيتعمل في مكان واحد بس.

---

### الـ Data Flow الكامل

```
Board/DT code
    │
    ├── fills struct unimac_mdio_pdata { ... }
    │       ├── phy_mask      ──► mdiobus->phy_mask
    │       ├── wait_func     ──► called after each MDIO R/W
    │       ├── wait_func_data──► passed to wait_func()
    │       ├── bus_name      ──► mii_bus->name
    │       └── clk           ──► clk_prepare_enable() at probe
    │
    └── registers platform_device with name = UNIMAC_MDIO_DRV_NAME
                                                     │
                                    kernel matches ──┘
                                                     │
                              platform_driver probe() called
                                    │
                                    ├── dev_get_platdata() → gets pdata
                                    ├── mdiobus_alloc()
                                    ├── applies phy_mask, bus_name, clk
                                    └── mdiobus_register()
```

---

### متى تستخدم الـ Header ده؟

| السيناريو | الاستخدام |
|-----------|-----------|
| Board file بيسجّل UniMAC MDIO device يدوياً | بيحتاج `#include` وبيملا الـ struct |
| Driver نفسه (`mdio-bcm-unimac.c`) | بيحتاج الـ struct عشان يقرا الـ platform data |
| SoC بيستخدم Device Tree بس | الـ driver بيبني الـ struct من DT properties — الـ header مش ضروري للـ board code |
| Custom wait mechanism مطلوب | بيستخدم `wait_func` + `wait_func_data` |
## Phase 5: دليل الـ Debugging الشامل

الـ subsystem اللي بنتكلم عنه هو **Broadcom UniMAC MDIO bus controller** — الـ driver اللي بيتحكم في الـ MDIO bus المدمج جوه الـ UniMAC Ethernet MAC في شرائح Broadcom (BCM7xxx, GENET, ASP, BCM6846). الـ platform data header (`mdio-bcm-unimac.h`) بيعرّف الـ `struct unimac_mdio_pdata` اللي بتحدد الـ `phy_mask`, الـ `wait_func`, والـ `clk`.

---

### Software Level

#### 1. debugfs

الـ MDIO subsystem مش بيسجّل entries في debugfs بشكل مباشر، لكن الـ **phy** و الـ **mii_bus** بيبانوا من خلال sysfs. مع ذلك، لو الـ kernel اتبنى بـ `CONFIG_DEBUG_FS`:

```bash
# شوف لو في debugfs مـ mount
mount | grep debugfs
# لو مش موجود:
mount -t debugfs none /sys/kernel/debug

# استعرض كل الـ entries المتعلقة بالـ mdio
ls /sys/kernel/debug/
```

مفيش entries مخصصة للـ unimac-mdio في debugfs، لكن الـ `mdiobus` نفسه بيظهر عن طريق:

```bash
ls /sys/kernel/debug/mdio_bus/
# مثال:
# unimac-mdio-0  unimac-mdio-1
cat /sys/kernel/debug/mdio_bus/unimac-mdio-0/mdio_bus_stats
```

#### 2. sysfs

```bash
# الـ mii_bus device
ls /sys/bus/mdio_bus/devices/
# مثال output:
# unimac-mdio-0:00  unimac-mdio-0:01  ...

# معلومات عن PHY معين (عنوان 0)
cat /sys/bus/mdio_bus/devices/unimac-mdio-0:00/phy_id
cat /sys/bus/mdio_bus/devices/unimac-mdio-0:00/phy_interface

# الـ platform device نفسه
ls /sys/bus/platform/devices/ | grep unimac
cat /sys/bus/platform/devices/unimac-mdio/driver_override

# الـ clock
cat /sys/kernel/debug/clk/clk_summary | grep mdio
```

#### 3. ftrace

```bash
# Enable tracing على الـ mdiobus operations
echo 1 > /sys/kernel/debug/tracing/events/mdio/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on

# اعمل أي operation على الـ PHY (مثلاً: ifconfig eth0 up)
cat /sys/kernel/debug/tracing/trace

# filter على device معين
echo 'bus_id == "unimac-mdio-0"' > /sys/kernel/debug/tracing/events/mdio/mdio_access/filter

# trace الـ clk events كمان
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_enable/enable
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_disable/enable
cat /sys/kernel/debug/tracing/trace
```

مثال output من tracing:

```
 ifconfig-234   [001]  1234.567890: mdio_access: unimac-mdio-0 phy=0 reg=1 val=0x796d write=0
```

#### 4. printk / dynamic debug

```bash
# تفعيل الـ dynamic debug للـ driver كله
echo "module mdio-bcm-unimac +p" > /sys/kernel/debug/dynamic_debug/control

# أو على مستوى الـ file
echo "file mdio-bcm-unimac.c +pflmt" > /sys/kernel/debug/dynamic_debug/control
# p = print, f = function name, l = line number, m = module, t = thread id

# تحقق إيه اللي اتفعّل
cat /sys/kernel/debug/dynamic_debug/control | grep unimac

# الـ debug message الموجودة في الكود:
# dev_dbg(&bus->dev, "Workaround for PHY @ %d\n", addr);
# بتظهر بعد تفعيل dynamic_debug

# تفعيل كل الـ mdio subsystem messages
echo "module of_mdio +p" > /sys/kernel/debug/dynamic_debug/control
echo "module libphy +p" > /sys/kernel/debug/dynamic_debug/control
```

#### 5. Kernel Config Options للـ Debugging

| Config Option | الهدف |
|---|---|
| `CONFIG_MDIO_BCM_UNIMAC` | الـ driver نفسه (يجب يكون built-in أو module) |
| `CONFIG_PHYLIB` | الـ PHY library الأساسية |
| `CONFIG_DEBUG_FS` | بيفعّل debugfs |
| `CONFIG_DYNAMIC_DEBUG` | بيفعّل dynamic printk |
| `CONFIG_OF_MDIO` | DT-based MDIO registration |
| `CONFIG_KALLSYMS` | لازم لـ stack traces مفهومة |
| `CONFIG_DEBUG_KERNEL` | يفعّل debugging عام |
| `CONFIG_PROVE_LOCKING` | lockdep للـ mii_bus->mdio_lock |
| `CONFIG_LOCKDEP` | تتبع الـ locking |
| `CONFIG_CLK_DEBUG` | debug الـ clock tree |
| `CONFIG_PM_DEBUG` | debug الـ resume/suspend |
| `CONFIG_REGMAP_DEBUGFS` | لو الـ MAC بيستخدم regmap |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "MDIO|PHYLIB|OF_MDIO"
# أو
grep -E "CONFIG_MDIO|CONFIG_PHYLIB" /boot/config-$(uname -r)
```

#### 6. أدوات خاصة بالـ Subsystem

```bash
# phytool: أداة لقراءة/كتابة الـ PHY registers مباشرة من userspace
# (يجب تثبيتها: apt install phytool أو بناءها من المصدر)

# قراءة BMSR (Basic Mode Status Register, reg=1)
phytool read eth0/0x00/0x01
# output مثال: 0x796d

# قراءة PHY ID
phytool read eth0/0x00/0x02   # PHYSID1
phytool read eth0/0x00/0x03   # PHYSID2

# كتابة على PHY register
phytool write eth0/0x00/0x00 0x1200  # software reset + auto-neg

# mii-tool لعرض حالة الـ link
mii-tool -v eth0

# ethtool لعرض معلومات الـ PHY
ethtool eth0
ethtool -d eth0    # register dump (لو supported)

# ioctl-based mdio access
# استخدم mdio-tools أو iproute2
devlink dev show
```

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `failed to remap register` | `devm_ioremap` فشل — الـ memory resource مش متاحة | تحقق من الـ DT `reg` property، وأن الـ address صح |
| `MDIO bus registration failed` | `of_mdiobus_register` رجع error | تحقق من الـ DT children nodes وأن الـ PHY addresses صح |
| `mdiobus_read` returns `-ETIMEDOUT` | `read_poll_timeout` انتهى timeout — الـ `MDIO_START_BUSY` ما اتصفرش | مشكلة hardware: الـ MDIO lines متعلقة أو الـ clock مش شغال |
| `Ignoring MDIO clock frequency request` | الـ `clock-frequency` المطلوب كبير أوي بالنسبة للـ reference clock | غيّر قيمة `clock-frequency` في الـ DT أو اتركها فاضية |
| `-EIO` من `unimac_mdio_read` | `MDIO_READ_FAIL` bit اتعمل set — الـ PHY مش بيرد | تحقق من الـ PHY address وأن الـ PHY powered ومتوصل |
| `devm_clk_get_optional` fails | الـ clock المحدد في DT مش موجود في الـ clock tree | تحقق من `clocks` property في DT وأن الـ clock provider registered |
| `Workaround for PHY @ N` | الـ BCM7xxx workaround بيتفعّل — dummy BMSR read | طبيعي لـ BCM7xxx PHYs، مش error |
| `of_mdio_parse_addr` returns negative | الـ child node في DT ملوش `reg` property صحيحة | أضف `reg = <PHY_ADDRESS>;` لكل PHY node |

#### 8. Strategic Points لـ dump_stack() و WARN_ON()

**النقاط المهمة للـ debug في الكود:**

```c
/* في unimac_mdio_probe — لو الـ resource مش موجودة */
r = platform_get_resource(pdev, IORESOURCE_MEM, 0);
if (!r) {
    WARN_ON(!r);  /* أضف هنا لو عايز stack trace */
    return -EINVAL;
}

/* في unimac_mdio_read — لما يرجع -EIO بسبب MDIO_READ_FAIL */
if (!(bus->phy_ignore_ta_mask & 1 << phy_id) && (cmd & MDIO_READ_FAIL)) {
    /* أضف هنا لتتبع مين طالب القراءة */
    pr_debug_ratelimited("MDIO read fail: phy=%d reg=%d cmd=0x%08x\n",
                         phy_id, reg, cmd);
    ret = -EIO;
    goto out;
}

/* في unimac_mdio_poll — لما الـ timeout يحصل */
ret = read_poll_timeout(...);
if (ret == -ETIMEDOUT) {
    dev_err(&priv->mii_bus->dev, "MDIO timeout: CMD=0x%08x\n",
            unimac_mdio_readl(priv, MDIO_CMD));
    dump_stack();  /* مفيد في early bring-up */
}

/* في unimac_mdio_clk_set — لما الـ divisor يتجاوز الـ mask */
if (div & ~MDIO_CLK_DIV_MASK) {
    WARN(1, "CLK div overflow: div=0x%x rate=%ld freq=%d\n",
         div, rate, priv->clk_freq);
}
```

---

### Hardware Level

#### 1. التحقق من حالة الـ Hardware

الـ UniMAC MDIO controller عنده register-ين أساسيين:
- **MDIO_CMD** (offset `0x00`): بيتحكم في الـ transactions
- **MDIO_CFG** (offset `0x04`): بيتحكم في الـ clock divider والـ protocol

```
MDIO_CMD register layout:
 31       29 28  27 26  25    21  20    16  15             0
 ┌──────────┬──┬────┬──────────┬──────────┬───────────────┐
 │START_BUSY│RF│RD/W│  PMD/PHY │  REGADDR │   DATA        │
 │  bit 29  │28│26-2│  [25:21] │  [20:16] │   [15:0]      │
 └──────────┴──┴────┴──────────┴──────────┴───────────────┘

MDIO_CFG register layout:
 15      13  12        4  3      1  0
 ┌────────┬───┬──────────┬────────┬──┐
 │reserved│SPP│ CLK_DIV  │reserved│C2│
 │        │ 12│  [9:4]   │        │  │
 └────────┴───┴──────────┴────────┴──┘
 C22=1 → Clause 22, C22=0 → Clause 45
```

```bash
# تحقق إن الـ driver اتحمّل وعرف الـ bus
dmesg | grep -i "unimac\|UniMAC\|MDIO"
# output متوقع:
# [    2.345678] unimac-mdio unimac-mdio: Broadcom UniMAC MDIO bus

# تحقق من الـ mii_bus registered
ls /sys/bus/mdio_bus/devices/
# يجب تلاقي: unimac-mdio-0:00, unimac-mdio-0:01, إلخ

# تحقق من الـ PHY تعرّف صح
cat /sys/bus/mdio_bus/devices/unimac-mdio-0:00/phy_id
# مثال output: 0x600d84a2  (Broadcom BCM54xx PHY)

# مقارنة بيتات MDIO_CFG الـ C22 bit
# لازم يكون set لو بتستخدم Clause 22 (معظم الـ PHYs العادية)
```

#### 2. Register Dump Techniques

```bash
# أولاً: اعرف الـ base address من dmesg أو /proc/iomem
cat /proc/iomem | grep -i mdio
# مثال:
# f0403c00-f0403c07 : unimac-mdio

# استخدام devmem2 لقراءة الـ MDIO_CMD register
# (افترض base = 0xf0403c00)
devmem2 0xf0403c00 w   # قراءة MDIO_CMD (32-bit)
devmem2 0xf0403c04 w   # قراءة MDIO_CFG

# مثال output لـ MDIO_CFG في حالة طبيعية:
# Value at address 0xF0403C04 (0x...): 0x00001031
# 0x1031 = C22 set, CLK_DIV=3, SUPP_PREAMBLE=0

# تفسير MDIO_CFG = 0x1031:
#   bit 0 (C22)       = 1  → Clause 22
#   bits[9:4] (CLK_DIV) = 3 → freq = 250MHz / (2*(3+1)) = ~31.25MHz
#   bit 12 (SUPP_PREAMBLE) = 0 → preamble sent

# استخدام /dev/mem مباشرة بـ Python
python3 -c "
import mmap, struct
with open('/dev/mem', 'r+b') as f:
    mm = mmap.mmap(f.fileno(), 0x10, offset=0xf0403c00)
    cmd = struct.unpack('<I', mm[0:4])[0]
    cfg = struct.unpack('<I', mm[4:8])[0]
    print(f'MDIO_CMD=0x{cmd:08x} MDIO_CFG=0x{cfg:08x}')
    mm.close()
"

# باستخدام io utility (لو متاحة)
io -4 -r 0xf0403c00
io -4 -r 0xf0403c04
```

#### 3. Logic Analyzer / Oscilloscope

الـ MDIO bus اتنين signals:
- **MDC** (Management Data Clock) — clock يولّده الـ MAC
- **MDIO** (Management Data I/O) — bidirectional data

```
MDIO Frame (Clause 22 Read):
  PRE    ST OP  PHYAD REGAD TA    DATA
  32×1   01  10 AAAAA RRRRR Z0  DDDDDDDDDDDDDDDD

  PRE = 32 bits preamble (all 1s)
  ST  = 01 (start of frame)
  OP  = 10 (read), 01 (write)
  TA  = turnaround (PHY drives 0 then data)

Clock frequency check:
  - Default: بيعتمد على الـ reference clock وقيمة CLK_DIV
  - عادةً: 2.5MHz إلى 25MHz
  - قيّس MDC بالـ oscilloscope — لازم تكون stable square wave

Common things to check:
  1. MDC: هل فيه clock signal؟ → لو لأ: مشكلة في الـ clk driver
  2. MDIO: هل الـ preamble (32 ones) بيتبعت؟ → CONFIG_MDIO_SUPP_PREAMBLE
  3. TA phase: هل الـ PHY بيرفع MDIO؟ → لو لأ: BCM53125 issue (phy_ignore_ta_mask)
  4. DATA: هل الـ 16-bit data صح؟ → قارن بـ phytool output
```

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة | Pattern في الـ Kernel Log | التشخيص |
|---|---|---|
| الـ PHY مش متوصل/مش مزوّد | تكرار `-EIO` من mdiobus_read في dmesg | شيك التوصيل الفيزيائي والـ power supply |
| الـ MDIO clock مش شغال | `clk_prepare_enable` returns error | شيك الـ `clocks` property في DT والـ clock provider |
| PHY address غلط | كل الـ reads بيرجعوا `0xFFFF` | راجع الـ hardware schematic لعنوان الـ PHY |
| الـ MDIO_START_BUSY مش بيتصفر | `-ETIMEDOUT` من `read_poll_timeout` | الـ MDIO controller مش بيشتغل — شيك الـ reset وتوصيل الـ clock |
| Pull-up ضعيف على MDIO | reads متقطعة وعشوائية | قيّس بالـ oscilloscope، زيد قيمة الـ pull-up resistor |
| الـ turn-around problem (BCM53125) | `-EIO` بس MDIO_READ_FAIL set | ضيف `phy-handle` مع `phy-ignore-ta` أو استخدم `phy_ignore_ta_mask` |

#### 5. Device Tree Debugging

```bash
# تحقق من الـ DT اللي اتحمّل فعلاً
# الـ DT المُجمَّع بيتظهر هنا:
ls /proc/device-tree/
find /proc/device-tree -name "compatible" | xargs -I{} sh -c 'echo "{}:"; cat "{}"' 2>/dev/null | grep -A1 unimac

# أو باستخدام dtc لتحويل الـ binary DT
dtc -I fs /proc/device-tree 2>/dev/null | grep -A 20 "unimac-mdio"

# تحقق من الـ reg property (لازم تتطابق مع الـ hardware memory map)
cat /proc/device-tree/soc/mdio@403c0/reg | xxd
# مثال output:
# 00000000: 0004 03c0 0000 0008                      ........
# يعني: base=0x403c0, size=0x8

# تحقق من الـ compatible string
cat /proc/device-tree/soc/mdio@403c0/compatible
# يجب يكون: brcm,unimac-mdio

# تحقق من الـ PHY children
ls /proc/device-tree/soc/mdio@403c0/
# يجب تلاقي: compatible  reg  #address-cells  #size-cells  ethernet-phy@0

# تحقق من clock-frequency لو محدد
cat /proc/device-tree/soc/mdio@403c0/clock-frequency | xxd
# مثال: 00 25 00 00 = 0x250000 = 2,424,832 Hz (~2.4MHz)

# استخدم of_node_put و of_find_compatible_node لـ runtime check
# أو من userspace:
find /sys/firmware/devicetree -name compatible -exec sh -c \
  'v=$(cat "$1" 2>/dev/null); case $v in *unimac*) echo "$1: $v";; esac' _ {} \;
```

---

### Practical Commands

#### Ready-to-Copy Shell Commands

```bash
# ============================================================
# 1. تحقق أن الـ driver اتحمّل صح
# ============================================================
dmesg | grep -E "unimac|mdio-bcm" | head -20

# expected output:
# [    2.123456] unimac-mdio unimac-mdio: Broadcom UniMAC MDIO bus

# ============================================================
# 2. اعرض كل الـ PHY devices المكتشفة
# ============================================================
ls /sys/bus/mdio_bus/devices/
# output مثال:
# unimac-mdio-0:00  unimac-mdio-0:01

# ============================================================
# 3. قرا PHY ID
# ============================================================
cat /sys/bus/mdio_bus/devices/unimac-mdio-0:00/phy_id
# output: 0x600d84a2

# ============================================================
# 4. فعّل dynamic debug للـ driver
# ============================================================
echo "module mdio-bcm-unimac +pflmt" > /sys/kernel/debug/dynamic_debug/control
dmesg -w &   # شاهد الـ output في real-time

# ============================================================
# 5. استخدم phytool لقراءة الـ PHY registers مباشرة
# ============================================================
phytool read eth0/0/1   # BMSR - Basic Mode Status Register
# output:
# 0x796d
# تفسير: bit 2 = link up, bit 5 = autoneg complete, إلخ

phytool print eth0/0    # اطبع كل الـ standard registers بشكل منظم

# ============================================================
# 6. فعّل MDIO tracing بـ ftrace
# ============================================================
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo > /sys/kernel/debug/tracing/trace
echo 1 > /sys/kernel/debug/tracing/events/mdio/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
# شغّل أي network operation
ip link set eth0 down && ip link set eth0 up
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace

# ============================================================
# 7. قرا الـ MDIO hardware registers مباشرة بـ devmem2
# ============================================================
# أول خطوة: اعرف الـ physical base address
cat /proc/iomem | grep mdio
# مثال output:
# f0403c00-f0403c07 : unimac-mdio

BASE=0xf0403c00
devmem2 $((BASE + 0x00)) w   # MDIO_CMD
devmem2 $((BASE + 0x04)) w   # MDIO_CFG

# ============================================================
# 8. اعمل manual MDIO transaction وتابع النتيجة
# ============================================================
# قرا MDSR من PHY عنوان 0 باستخدام ioctl
python3 - <<'EOF'
import socket, fcntl, struct

SIOCGMIIPHY = 0x8947
SIOCGMIIREG = 0x8948
SIOCSMIIREG = 0x8949

def mdio_read(iface, phy_addr, reg):
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    try:
        # pack: iface(16s) + phy_id(H) + reg_num(H) + val_out(H) + pad
        data = struct.pack('16sHHH', iface.encode(), phy_addr, reg, 0)
        data = data + b'\x00' * (40 - len(data))
        result = fcntl.ioctl(s, SIOCGMIIREG, data)
        return struct.unpack('16sHHH', result[:22])[3]
    finally:
        s.close()

phy_id1 = mdio_read('eth0', 0, 2)   # MII_PHYSID1
phy_id2 = mdio_read('eth0', 0, 3)   # MII_PHYSID2
bmsr    = mdio_read('eth0', 0, 1)   # MII_BMSR
print(f"PHY ID: 0x{phy_id1:04x}{phy_id2:04x}")
print(f"BMSR:   0x{bmsr:04x}")
print(f"  Link Up: {bool(bmsr & 0x04)}")
print(f"  Autoneg Complete: {bool(bmsr & 0x20)}")
EOF

# ============================================================
# 9. تحقق من الـ clock state
# ============================================================
cat /sys/kernel/debug/clk/clk_summary | grep -i mdio
# output مثال:
# mdio_clk       1  1  0  250000000  0     0  50000

# ============================================================
# 10. تحقق من الـ DT node اللي اتحمّل
# ============================================================
dtc -I fs /proc/device-tree 2>/dev/null | grep -B2 -A30 "unimac-mdio"
```

#### تفسير الـ Output

```
# تفسير MDIO_CMD register value = 0x20000000:
#   bit 29 (MDIO_START_BUSY) = 1  → transaction قيد التنفيذ أو ما كملتش
#   bit 28 (MDIO_READ_FAIL)  = 0  → ما فيش read failure
#   bits[27:26] = 00           → idle
# لو START_BUSY ضل set دايماً → مشكلة hardware (timeout حيحصل)

# تفسير MDIO_CFG register value = 0x1031:
#   bit 0  (C22)         = 1  → Clause 22 mode
#   bits[9:4] (CLK_DIV)  = 3  → MDIO clock = 250MHz / (2×4) = 31.25MHz
#   bit 12 (SUPP_PREAMBLE) = 1 → suppress preamble (بعض الـ PHYs مش بتحتاجه)

# تفسير phytool BMSR output = 0x796d:
#   bit 0: Extended capability     = 1
#   bit 2: Link status             = 1  ← Link UP
#   bit 3: Autoneg ability         = 1
#   bit 5: Autoneg complete        = 1
#   bit 6: MF Preamble Suppression = 1
#   bit 8: Extended status         = 0
#   bit 14: 100BASE-TX full duplex = 1
#   bit 15: 100BASE-TX half duplex = 1
```
## Phase 6: سيناريوهات من الحياة العملية

---

### سيناريو 1: Industrial Gateway على BCM58625 — الـ MDIO bus مش شايف أي PHY

#### العنوان
PHY scan فارغة رغم إن الـ PHY موصّل فعلاً على الـ board

#### السياق
شركة بتبني industrial Ethernet gateway بناءً على **Broadcom BCM58625** SoC. الـ board عندها **BCM54616S** Gigabit PHY متوصل عن طريق **UniMAC MDIO bus**. الـ driver بيتسجّل من غير مشاكل، بس `ip link` مش بيظهر أي Ethernet interface وكل probe بيفشل.

#### المشكلة
الـ `phy_mask` في الـ platform data بيـmask الـ PHY address اللي موجود فعلاً على الـ bus، فالـ MDIO bus scan بيتخطّى الـ PHY من غير ما يحاول يتكلم معاه.

#### التحليل
الـ struct اللي بتحكم الموضوع:

```c
struct unimac_mdio_pdata {
    u32 phy_mask;        /* bitmask — bit N = skip PHY at address N */
    int (*wait_func)(void *data);
    void *wait_func_data;
    const char *bus_name;
    struct clk *clk;
};
```

الـ driver بيمرر `phy_mask` للـ `mdiobus_alloc()` → `bus->phy_mask`. أي bit مضروب في الـ mask معناه "متعملش scan لـ PHY address ده". لو الـ PHY موجود على address 1 والـ mask هو `0xFFFFFFFE` (bit 1 set)، الـ scan هيتخطاه.

في الـ board file أو ACPI/DT glue code:

```c
/* خطأ: bit 1 مضروب = PHY address 1 متسكنش */
static struct unimac_mdio_pdata bcm58625_mdio_pdata = {
    .phy_mask  = 0xFFFFFFFE,  /* بيعمل skip لكل العناوين ما عدا 0 */
    .bus_name  = "unimac-mdio",
};
```

بينما الـ PHY فعلاً على address 1 مش address 0.

#### الحل

```c
/* صح: الـ PHY على address 1، يبقى كل العناوين التانية تتسكب */
static struct unimac_mdio_pdata bcm58625_mdio_pdata = {
    .phy_mask  = ~BIT(1),   /* scan بس address 1 */
    .bus_name  = "unimac-mdio",
};
```

أو للـ debug السريع:

```bash
# شوف الـ PHY addresses اللي الـ bus شايفها فعلاً
ls /sys/bus/mdio_bus/devices/

# لو فاضية، افحص الـ phy_mask
cat /sys/bus/platform/devices/unimac-mdio/*/of_node/phy-handle 2>/dev/null || \
  grep -r phy_mask /sys/bus/platform/devices/unimac-mdio/
```

#### الدرس المستفاد
**الـ `phy_mask` هو bitmask للـ skip مش للـ select** — الـ bit المضروب يعني "اتجنّب"، مش "استخدم". غلطة شائعة جداً لما بتنقل platform data من board تانية.

---

### سيناريو 2: Android TV Box على BCM7445 — الـ Ethernet بيقع بعد suspend/resume

#### العنوان
الـ MDIO bus بيـhang بعد الـ system resume من deep sleep

#### السياق
منتج Android TV box بيستخدم **Broadcom BCM7445** SoC مع **BCM54210** PHY على UniMAC MDIO. الجهاز بيشتغل تمام، بس بعد ما بيدخل في **S3 suspend** ويرجع، الـ Ethernet بيقع تماماً وكل MDIO read بيرجع `0xFFFF`.

#### المشكلة
الـ `clk` في الـ platform data — لما الـ system بيعمل suspend، الـ clock بيتوقف. لما بيرجع، الـ clock ما بيتشغّلش تاني صح لأن الـ driver مش بيعمل `clk_prepare_enable()` في الـ resume path.

#### التحليل

```c
struct unimac_mdio_pdata {
    u32 phy_mask;
    int (*wait_func)(void *data);
    void *wait_func_data;
    const char *bus_name;
    struct clk *clk;      /* ← الـ clock reference موجود هنا */
};
```

الـ driver في `mdio-bcm-unimac.c` بيعمل:

```c
/* في probe */
priv->clk = pdata ? pdata->clk : of_clk_get(dev, 0);
clk_prepare_enable(priv->clk);

/* في suspend — بيعمل disable */
clk_disable_unprepare(priv->clk);

/* في resume — المفروض يعمل enable تاني */
ret = clk_prepare_enable(priv->clk);
/* لو الـ clk pointer اتغيّر أو corrupt بعد suspend → hang */
```

لو الـ platform data اتعملها `kfree` أو اتعدّلت بعد الـ probe (في بعض الـ BSP implementations)، الـ `priv->clk` بيبقى dangling pointer.

#### الحل

```c
/* في الـ BSP code، خلّي الـ pdata static أو معمولها lifetime مناسبة */
static struct unimac_mdio_pdata bcm7445_mdio_pdata = {
    .phy_mask  = ~BIT(0),
    .bus_name  = "eth0-mdio",
    .clk       = NULL,  /* خليه NULL وسيب الـ driver يجيبه من DT */
};
```

```bash
# تحقق إن الـ clock شغّالة بعد resume
cat /sys/kernel/debug/clk/unimac_mdio_clk/clk_enable_count

# trace الـ resume
echo 1 > /sys/module/mdio_bcm_unimac/parameters/debug
dmesg | grep -i unimac | tail -20
```

#### الدرس المستفاد
لو بتمرر `struct clk *` في الـ platform data، لازم تضمن إن الـ lifetime بتاعتها تتجاوز الـ driver lifetime كلها بما فيها suspend/resume cycles. الأأمن إنك تسيب الـ `clk` بـ `NULL` وتخلّي الـ driver يجيبه من device tree.

---

### سيناريو 3: Custom Board Bring-up على BCM63138 — الـ `wait_func` بيسبب kernel panic

#### العنوان
Custom `wait_func` callback بيـcrash الـ kernel أثناء MDIO transaction

#### السياق
فريق bring-up بيشتغل على **BCM63138**-based DSL modem/router. بيحتاجوا custom synchronization لأن الـ UniMAC MDIO على الـ SoC ده بيحتاج wait خاص بسبب shared register space مع subsystem تاني. حطّوا `wait_func` في الـ platform data، وبعد كام ساعة من الـ testing، kernel panic.

#### المشكلة
الـ `wait_func` pointer في الـ platform data اتمسح من الـ memory لأن الـ function اتعرّفت في module اتـunload وهو الـ MDIO driver لسه بيستخدمها.

#### التحليل

```c
struct unimac_mdio_pdata {
    u32 phy_mask;
    int (*wait_func)(void *data);   /* ← function pointer */
    void *wait_func_data;           /* ← الـ data اللي بيتبعتله */
    const char *bus_name;
    struct clk *clk;
};
```

الـ driver بيستخدمها هكذا في كل MDIO operation:

```c
/* من mdio-bcm-unimac.c */
if (pdata && pdata->wait_func) {
    ret = pdata->wait_func(pdata->wait_func_data);
    if (ret)
        return ret;
}
```

الـ BSP كود عمل كده:

```c
/* في module A — اتـunload */
static int my_mdio_wait(void *data)
{
    /* check some register */
    return 0;
}

static struct unimac_mdio_pdata router_mdio_pdata = {
    .wait_func      = my_mdio_wait,   /* pointer لـ function في module هيتـunload */
    .wait_func_data = &my_shared_data,
    .phy_mask       = ~BIT(0),
};
```

لما module A اتـunload وجه MDIO transaction، الـ kernel call الـ function pointer على عنوان متحرّرة → panic.

#### الحل

```c
/* الـ wait_func لازم تكون في الـ core kernel أو في نفس module الـ MDIO driver */
/* أو استخدم completion/mutex بدل custom wait */

/* الأبسط: سيب wait_func = NULL وخلّي الـ driver يستخدم الـ default polling */
static struct unimac_mdio_pdata router_mdio_pdata = {
    .wait_func      = NULL,
    .wait_func_data = NULL,
    .phy_mask       = ~BIT(0),
    .bus_name       = "dsl-mdio",
};
```

```bash
# لو محتاج تتأكد من crash address
# بعد الـ panic، الـ call stack هيظهر العنوان المجهول
# استخدم addr2line أو kallsyms
echo "0xc1234567" > /proc/sys/kernel/perf_event_paranoid  # مش الحل
# الصح: افحص /proc/modules قبل الـ panic
cat /proc/modules | grep -v "^\[permanent\]"
```

#### الدرس المستفاد
الـ `wait_func` callback في `unimac_mdio_pdata` يجب إنها تكون إما `NULL` أو pointer لـ function في **permanent kernel code** (built-in). أي function في loadable module ممكن تتـunload وتسبب use-after-free.

---

### سيناريو 4: IoT Sensor Hub على BCM5301x — الـ bus_name بيتعارض مع MDIO bus تاني

#### العنوان
اتنين MDIO buses بنفس الاسم بيسبّبوا sysfs conflicts وـmatch غلط للـ PHY drivers

#### السياق
board بنيت على **BCM5301x** (Northstar) SoC لـ industrial IoT hub فيها **اتنين UniMAC** — واحد لـ WAN port وواحد لـ LAN ports عبر internal switch. الاتنين بيستخدموا `unimac-mdio` driver. بعد boot، بعض الـ PHYs بيتـbind للـ driver الغلط.

#### المشكلة
الاتنين platform data instances بيستخدموا نفس `bus_name` = `"unimac-mdio"`، فالـ MDIO bus registration بيتعارض وبعض الـ PHY lookups بتجيب الـ bus الغلط.

#### التحليل

```c
struct unimac_mdio_pdata {
    u32 phy_mask;
    int (*wait_func)(void *data);
    void *wait_func_data;
    const char *bus_name;   /* ← الاسم ده بيتحوّل لـ mdio bus ID */
    struct clk *clk;
};
```

الـ driver بيستخدم `bus_name` كـ identifier:

```c
/* في mdio-bcm-unimac.c */
bus->name = pdata ? pdata->bus_name : "unimac-mdio";
snprintf(bus->id, MII_BUS_ID_SIZE, "%s", bus->name);
```

لو الاتنين بنفس الاسم، الـ second `mdiobus_register()` بيفشل أو الـ sysfs entries بتـconflict.

#### الحل

```c
/* WAN MDIO */
static struct unimac_mdio_pdata bcm5301x_wan_mdio_pdata = {
    .phy_mask  = ~BIT(0),
    .bus_name  = "unimac-mdio-wan",   /* اسم فريد */
    .clk       = NULL,
};

/* LAN MDIO */
static struct unimac_mdio_pdata bcm5301x_lan_mdio_pdata = {
    .phy_mask  = 0xFFFFFFE0,  /* scan addresses 0-4 بس للـ switch ports */
    .bus_name  = "unimac-mdio-lan",   /* اسم فريد مختلف */
    .clk       = NULL,
};
```

```bash
# تحقق من الـ MDIO buses المسجّلة
ls /sys/bus/mdio_bus/devices/
# المفروض تشوف:
# unimac-mdio-wan:00
# unimac-mdio-lan:00
# ... إلخ

# لو فيه تعارض
dmesg | grep -i "mdio.*already\|mdio.*exist\|mdiobus_register"
```

#### الدرس المستفاد
كل `unimac_mdio_pdata` instance لازم يكون عنده `bus_name` **فريد**. الـ default `"unimac-mdio"` مناسب بس لما يبقى في الـ system bus واحد بس.

---

### سيناريو 5: Automotive ECU على BCM89610 — الـ PHY initialization بتتأخر وتسبّب link failure

#### العنوان
Race condition بين MDIO probe وPHY power-up بيسبّب فشل الـ link في الـ cold boot

#### السياق
ECU للسيارة بيستخدم **Broadcom BCM89610** (automotive Ethernet SoC) مع **BCM89811** 100BASE-T1 PHY (single-pair automotive). الـ ECU مشغّل Linux مع functional safety requirements. في الـ cold boot (~-20°C)، الـ Ethernet link بيفشل تقريباً 30% من الوقت.

#### المشكلة
الـ `wait_func` مش موجودة في الـ platform data (NULL)، فالـ driver بيستخدم default polling. في الـ cold boot، الـ PHY بياخد وقت أطول للـ power-up بسبب الـ slow capacitor charging، والـ MDIO probe بيحصل قبل ما الـ PHY يكون ready.

#### التحليل

```c
struct unimac_mdio_pdata {
    u32 phy_mask;
    int (*wait_func)(void *data);   /* NULL → default timeout polling */
    void *wait_func_data;
    const char *bus_name;
    struct clk *clk;
};
```

لما `wait_func` بيكون `NULL`، الـ driver بيعمل:

```c
/* default behavior — fixed timeout */
#define MDIO_TIMEOUT  100  /* ms */

static int unimac_mdio_wait_for_idle(void __iomem *base, bool result)
{
    u32 timeout = MDIO_TIMEOUT;
    while (timeout--) {
        if (/* bus idle */)
            return 0;
        udelay(1000);
    }
    return -ETIMEDOUT;
}
```

في الـ cold boot بالـ low temperature، الـ PHY بياخد أكتر من 100ms حتى يرد على MDIO queries.

#### الحل

```c
/* custom wait function تأخذ temperature في الاعتبار */
static int automotive_mdio_wait(void *data)
{
    struct bcm89610_mdio_data *priv = data;
    int temp = read_board_temperature(priv);
    int timeout_ms = (temp < 0) ? 500 : 100;  /* أطول في الـ cold */

    return wait_for_mdio_idle(priv->base, timeout_ms);
}

static struct bcm89610_mdio_data ecu_mdio_data = {
    .base = BCM89610_MDIO_BASE,
};

static struct unimac_mdio_pdata ecu_mdio_pdata = {
    .phy_mask       = ~BIT(3),            /* PHY على address 3 */
    .wait_func      = automotive_mdio_wait,
    .wait_func_data = &ecu_mdio_data,
    .bus_name       = "ecu-mdio",
    .clk            = NULL,
};
```

```bash
# debug الـ MDIO timing
echo 'file mdio-bcm-unimac.c +p' > /sys/kernel/debug/dynamic_debug/control
dmesg | grep unimac | grep -i "timeout\|wait\|idle"

# قياس الـ PHY response time
ethtool --phy-statistics eth0 2>/dev/null | grep -i timeout

# فحص الـ link state بعد الـ boot مباشرة
for i in $(seq 1 10); do
    ip link show eth0 | grep -o "state [A-Z]*"
    sleep 1
done
```

#### الدرس المستفاد
الـ `wait_func` callback في `unimac_mdio_pdata` مش مجرد optional feature — في الـ automotive و industrial environments، الـ PHY power-up timing بيختلف حسب الـ temperature والـ supply voltage. استخدام custom `wait_func` بيديك تحكم كامل في الـ MDIO transaction timing بدل ما تعتمد على الـ fixed default timeout.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي بتغطي الـ MDIO subsystem والـ Broadcom UniMAC بشكل مباشر أو غير مباشر:

| المقال | الموضوع |
|--------|---------|
| [Support MDIO devices](https://lwn.net/Articles/671456/) | تحويل الـ DSA switches لـ MDIO devices بدل PHY-only model |
| [Support MDIO devices (full thread)](https://lwn.net/Articles/670191/) | النقاش الكامل حول توسيع الـ MDIO bus لدعم non-PHY devices |
| [Add support for Broadcom iProc MDIO and Cygnus Ethernet PHY](https://lwn.net/Articles/659127/) | إضافة MDIO driver لـ iProc SoCs من Broadcom — قريب جداً لـ UniMAC architecture |
| [Add MDIO bus multiplexer support for iProc SoCs](https://lwn.net/Articles/690990/) | MDIO mux support على Broadcom iProc |
| [net: mdio: Add netlink interface](https://lwn.net/Articles/925483/) | إضافة netlink interface للـ MDIO bus لتسهيل debugging |
| [netdev/of/phy: MDIO bus multiplexer support](https://lwn.net/Articles/460763/) | الـ MDIO mux framework الأساسي في الـ kernel |
| [net: bcmasp: Add support for ASP2.0 Ethernet controller](https://lwn.net/Articles/870605/) | الـ BCM ASP 2.0 controller اللي بيستخدم `MDIO_BCM_UNIMAC` كـ dependency |
| [phylink — kernel docs](https://static.lwn.net/kerneldoc/networking/sfp-phylink.html) | الـ phylink layer اللي بيربط MAC بـ PHY عبر MDIO |

---

### توثيق الـ Kernel الرسمي

**الـ`Documentation/` paths الأساسية:**

```
Documentation/networking/phy.rst
    — شرح كامل لـ PHY subsystem وكيفية التسجيل على MDIO bus

Documentation/devicetree/bindings/net/brcm,unimac-mdio.yaml
    — الـ Device Tree bindings الرسمية لـ mdio-bcm-unimac driver

Documentation/networking/sfp-phylink.rst
    — كيفية ربط MAC بـ PHY عبر phylink/MDIO

Documentation/driver-api/phy/phy.rst
    — الـ PHY framework API للـ driver developers
```

**الـ driver الرئيسي في الـ kernel:**

```
drivers/net/phy/mdio-bcm-unimac.c   — التنفيذ الكامل
include/linux/platform_data/mdio-bcm-unimac.h  — الـ platform data struct
```

---

### Kernel Commits المهمة

| الـ Commit / Patch | الوصف |
|--------------------|-------|
| [Allow configuring MDIO clock divider](https://lore.kernel.org/lkml/20180921000542.26022-1-f.fainelli@gmail.com/) | Florian Fainelli يضيف دعم لضبط الـ clock divider في الـ UniMAC MDIO |
| [mdio-bcm-unimac: Add ASP v2.0 support](https://lore.kernel.org/netdev/1632519891-26510-5-git-send-email-justinpopo6@gmail.com/) | Justin Chen يضيف دعم Broadcom ASP 2.0 للـ driver |
| [Fix module autoload for OF platform driver](https://linux.kernel.narkive.com/zRwn4wCV/patch-1-2-net-phy-mdio-bcm-unimac-fix-module-autoload-for-of-platform-driver) | إصلاح الـ module alias لضمان تحميل الـ module تلقائياً |
| [stable queue: allow configuring MDIO clock (4.19)](https://kernel.googlesource.com/pub/scm/linux/kernel/git/stable/stable-queue/+/master/releases/4.19.85/net-phy-mdio-bcm-unimac-allow-configuring-mdio-clock.patch) | backport للـ stable kernel |
| [LKML: Correct rate fallback logic (2025)](https://lkml.org/lkml/2025/7/29/1180) | Florian Fainelli يصلح الـ clock rate fallback logic |

---

### نقاشات Mailing List

| النقاش | الموضوع |
|--------|---------|
| [GENET and UNIMAC MDIO probe issue](https://lore.kernel.org/all/4e230c8c-790a-2a87-ec72-c1d6e166719e@gmail.com/T/) | مشكلة probe ordering بين GENET و UniMAC MDIO على Raspberry Pi |
| [Add mdio-bcm-unimac soft dependency to bcmgenet](https://lore.kernel.org/netdev/162457560642.7534.18047903047924018273.git-patchwork-notify@kernel.org/) | إضافة `softdep` لضمان تحميل الـ MDIO driver قبل GENET |
| [MDIO_BCM_UNIMAC depend on HAS_IOMEM](https://linux.kernel.narkive.com/ZPK9YbIg/patch-drivers-net-phy-kconfig-let-mdio-bcm-unimac-depend-on-has-iomem) | إصلاح الـ Kconfig dependency لتفادي undefined symbol |
| [Rework GENET MDIO controller clocking](https://patchew.org/linux/20240219204053.471825-1-florian.fainelli@broadcom.com/) | إعادة هيكلة الـ clocking في الـ UniMAC MDIO (2024) |

---

### Kernelnewbies.org — تحديثات عبر الـ Kernel Versions

| الـ Kernel Version | التغيير المتعلق بـ MDIO |
|--------------------|------------------------|
| [Linux 2.6.35](https://kernelnewbies.org/Linux_2_6_35-DriversArch) | دعم clause 45 MDIO commands على مستوى الـ MDIO bus |
| [Linux 4.3](https://kernelnewbies.org/Linux_4.3) | دعم multiple MDIO buses في الـ DSA framework |
| [Linux 4.13](https://kernelnewbies.org/Linux_4.13) | إضافة group لـ AVB MDIO و MII pins |
| [Linux 6.5](https://kernelnewbies.org/Linux_6.5) | إضافة regmap-based MDIO driver |
| [Linux 6.9](https://kernelnewbies.org/Linux_6.9) | تحسينات على الـ MDIO subsystem |
| [Linux 6.17](https://kernelnewbies.org/Linux_6.17) | إضافة MDIO bus controller لـ Airoha AN7583 |

---

### Kernel Driver Database

**الـ** [CONFIG_MDIO_BCM_UNIMAC entry](https://cateee.net/lkddb/web-lkddb/MDIO_BCM_UNIMAC.html) **على** cateee.net بيوضح:
- الـ Kconfig option الكاملة
- الـ kernel versions اللي ظهر فيها الـ driver
- الـ dependencies والـ users

---

### كتب مُوصى بيها

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 17**: Network Drivers — بيشرح كيفية كتابة network device driver
- **الفصل 14**: The Linux Device Model — بيشرح الـ platform_device وكيف بتتربط بـ `struct unimac_mdio_pdata`
- متاح مجاناً على: https://lwn.net/Kernel/LDD3/

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 17**: Devices and Modules — بيغطي الـ device model والـ platform drivers
- **الفصل 13**: The Virtual Filesystem — مفيد لفهم كيف بيُعبّر عن الـ MDIO bus كـ abstraction layer

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 14**: Kernel Debugging Techniques — debugging الـ MDIO/PHY issues
- **الفصل 16**: Bootloader و Device Initialization — كيف بيبدأ الـ PHY initialization
- الـ [ECE497 listings](https://elinux.org/ECE497_Listings_for_Embedded_Linux_Primer_Chapter_2) من eLinux.org بتكمّل الكتاب بأمثلة عملية

#### Understanding Linux Network Internals — Christian Benvenuti
- بيغطي الـ net_device وكيف بتتكامل الـ PHY مع الـ MAC layer
- مرجع مهم لفهم الـ flow من الـ `mii_bus` لحد الـ network stack

---

### Search Terms للبحث عن مزيد من المعلومات

```
# للبحث في kernel mailing list (lore.kernel.org)
mdio-bcm-unimac
unimac_mdio_pdata
MDIO_BCM_UNIMAC
bcmgenet MDIO probe
UniMAC MDIO clock divider

# للبحث في kernel source (elixir.bootlin.com)
unimac_mdio_pdata
UNIMAC_MDIO_DRV_NAME
mdio-bcm-unimac

# للبحث العام
"Broadcom UniMAC MDIO" Linux driver
"platform_data MDIO" BCM kernel
GENET MDIO probe ordering issue
phylib MDIO bus registration Linux
```

**الـ** [Bootlin Elixir Cross-Referencer](https://elixir.bootlin.com/linux/latest/source/include/linux/platform_data/mdio-bcm-unimac.h) بيخليك تتتبع كل استخدام للـ struct في الـ kernel source بشكل interactive.
## Phase 8: Writing simple module

### الفكرة

الـ `unimac-mdio` driver بيسجّل `mdio_bus` عشان يتكلم مع PHY chips على شبكات Broadcom. أهم نقطة ممكن نعمل فيها hook هي لما الـ driver بيعمل **probe** — يعني لما الـ platform device بيتعرف ويبدأ تهيئة الـ MDIO bus. هنستخدم **kprobe** على `unimac_mdio_probe` عشان نطبع متى الـ driver اتشغّل ومين طلبه.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on unimac_mdio_probe:
 * Prints a message every time the Broadcom UniMAC MDIO driver
 * probes a platform device.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit   */
#include <linux/kernel.h>       /* pr_info                            */
#include <linux/kprobes.h>      /* kprobe struct, register_kprobe     */
#include <linux/platform_device.h> /* struct platform_device          */

/* ------------------------------------------------------------------ */
/* kprobe handler — runs BEFORE unimac_mdio_probe() executes          */
/* ------------------------------------------------------------------ */

/*
 * pre_handler يتنفذ قبل ما الـ function الأصلية تشتغل.
 * الـ regs بتحتوي على حالة الـ CPU registers في لحظة الـ probe.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * unimac_mdio_probe(struct platform_device *pdev)
     * على x86-64: أول argument بيكون في rdi
     * على ARM64:  أول argument بيكون في x0
     * نستخدم regs_get_kernel_argument() اللي portable عبر architectures
     */
    struct platform_device *pdev =
        (struct platform_device *)regs_get_kernel_argument(regs, 0);

    if (pdev)
        pr_info("[unimac-mdio kprobe] probe called for device: %s (id=%d)\n",
                pdev->name ? pdev->name : "unknown",
                pdev->id);
    else
        pr_info("[unimac-mdio kprobe] probe called (pdev=NULL?)\n");

    return 0; /* 0 = let the original function continue normally */
}

/*
 * post_handler بيتنفذ بعد ما الـ function الأصلية تخلص.
 * هنا بنطبع إننا رجعنا بسلام من الـ probe.
 */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    pr_info("[unimac-mdio kprobe] probe function returned.\n");
}

/* ------------------------------------------------------------------ */
/* kprobe descriptor                                                   */
/* ------------------------------------------------------------------ */

/*
 * بنحدد اسم الـ symbol اللي هنحطّ عليه الـ kprobe.
 * الـ kernel هيحوّل الاسم ده لعنوان في الذاكرة وقت الـ registration.
 */
static struct kprobe kp = {
    .symbol_name  = "unimac_mdio_probe",
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

/* ------------------------------------------------------------------ */
/* module_init                                                         */
/* ------------------------------------------------------------------ */

/*
 * register_kprobe بتحجز الـ hook وتضيف breakpoint افتراضي على الـ function.
 * لو الـ symbol مش موجود (الـ driver مش مكمبايل) بترجع -ENOENT.
 */
static int __init unimac_kprobe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[unimac-mdio kprobe] register_kprobe failed: %d\n", ret);
        pr_err("  (is CONFIG_MDIO_BCM_UNIMAC enabled and not built-in?)\n");
        return ret;
    }

    pr_info("[unimac-mdio kprobe] planted at %p (%s)\n",
            kp.addr, kp.symbol_name);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit                                                         */
/* ------------------------------------------------------------------ */

/*
 * unregister_kprobe ضروري في الـ exit عشان نشيل الـ breakpoint
 * ونمنع أي crash لو الـ module اتشال والـ handler لسه شايل pointer
 * على كود مش موجود.
 */
static void __exit unimac_kprobe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("[unimac-mdio kprobe] removed.\n");
}

module_init(unimac_kprobe_init);
module_exit(unimac_kprobe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe demo on unimac_mdio_probe — Broadcom UniMAC MDIO driver");
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/module.h` | ماكروهات `module_init` / `module_exit` / `MODULE_*` |
| `linux/kernel.h` | `pr_info` / `pr_err` |
| `linux/kprobes.h` | `struct kprobe`, `register_kprobe`, `regs_get_kernel_argument` |
| `linux/platform_device.h` | تعريف `struct platform_device` عشان نقرأ منه الاسم والـ id |

---

#### الـ `pre_handler`

الـ `pre_handler` بيتنفذ قبل ما `unimac_mdio_probe` تشتغل، فبنقدر نشوف مين الـ device اللي بيتعمل له probe قبل أي تعديل. استخدمنا `regs_get_kernel_argument(regs, 0)` عشان نجيب أول argument بطريقة portable على x86-64 وARM64 من غير ما نحتاج نكتب كود منفصل لكل architecture.

---

#### الـ `post_handler`

الـ `post_handler` بيأكد إن الـ probe function خلصت بدون ما تطلع exception. ده مفيد في الـ debugging عشان تعرف إذا الـ function اتنفذت كاملة ولا قفلت في النص.

---

#### الـ `struct kprobe`

الـ `symbol_name` بدل `addr` بيخلّي الـ kernel يحوّل الاسم لعنوان وقت الـ `register_kprobe` — ده أأمن لأن الـ address بيتغير مع كل كمبايل بسبب KASLR. لو الـ driver مش موجود في الـ kernel (مش `CONFIG_MDIO_BCM_UNIMAC=y`) الـ registration هترجع خطأ ونطلع بنظافة.

---

#### `module_init` و `module_exit`

**الـ `register_kprobe`** بتحجز الـ hook وقت تحميل الـ module، و**الـ `unregister_kprobe`** في الـ exit ضروري عشان لو الـ module اتأزل من الـ kernel وفي interrupt أو thread لسه شايل pointer على الـ `pre_handler`، ده هيعمل kernel panic — الـ unregister بيضمن إن كل الـ handlers خلصت وماحدش شايل pointer عليها قبل ما الكود يتشال من الذاكرة.

---

### كيفية التجربة

```bash
# بناء الـ module (مع Makefile بسيط)
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# تحميل الـ module
sudo insmod unimac_kprobe.ko

# مراقبة الـ output
sudo dmesg -w | grep unimac-mdio

# إزالة الـ module
sudo rmmod unimac_kprobe
```

**ملاحظة:** لو الـ machine مفيهاش Broadcom UniMAC hardware، الـ kprobe هيتسجل بنجاح لكن الـ handler مش هيتنفذ إلا لو اتعمل simulate لـ platform device أو شغّلت على hardware حقيقي.
