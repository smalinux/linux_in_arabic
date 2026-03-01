## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الـ file ده جزء من **MDIO Regmap Driver** — subsystem صغير جوه networking stack في Linux kernel. بيتبع مجموعة drivers الموجودة في `drivers/net/mdio/`.

---

### الـ Big Picture — قبل أي كود

#### القصة الكاملة بالتخيل

تخيل إنك بتبني SoC (شريحة متكاملة) فيها Ethernet controller. الـ controller ده محتاج يتكلم مع **PHY** — الدائرة اللي بتحوّل الـ digital data لـ electrical signals على السلك.

الطريقة الطبيعية إن الـ PHY يكون chip منفصل على الـ PCB، والـ CPU يوصله عن طريق **MDIO bus** — بروتوكول serial بسيط (زي I2C بس للـ network PHYs).

بس في حالات تانية: الـ PHY أو الـ PCS (Physical Coding Sublayer) بيكون **embedded داخل الـ SoC نفسه**، ومش موجود كـ chip خارجي. في الحالة دي، الـ CPU بيوصل لسجلاته مباشرة عن طريق **MMIO (Memory-Mapped I/O)** — يعني بيكتب ويقرأ من عناوين في الـ memory مباشرة.

**المشكلة:** كل الـ PHY drivers في Linux مكتوبة وهي متوقعة إن في **MDIO bus** حقيقي. لو الـ PHY embedded وبيتوصله بـ MMIO، محتاج تعمل "جسر" يخلي الـ MDIO read/write تتحول لـ regmap read/write.

**الحل:** `mdio-regmap` — بيعمل **virtual MDIO bus** فوق أي `regmap` interface.

---

### ليه ده مفيد؟

```
بدونه:
    PHY driver ← MDIO bus ← PHY chip (خارجي)

معاه:
    PHY driver ← Virtual MDIO bus (mdio-regmap) ← regmap ← MMIO registers (internal PHY/PCS)
```

يعني نفس الـ PHY driver يشتغل من غير تعديل، سواء الـ PHY كان خارجي أو embedded جوه الـ SoC.

---

### الـ regmap هو إيه؟

**regmap** هو abstraction layer في Linux kernel بيوحّد طريقة القراءة والكتابة على registers سواء كانت:
- MMIO (Memory-Mapped)
- I2C
- SPI
- أي bus تاني

بدل ما كل driver يتعامل مع الـ hardware مباشرة، بيتعامل مع `struct regmap *` وخلاص.

---

### الـ MDIO هو إيه؟

**MDIO (Management Data Input/Output)** هو بروتوكول serial بسيط جداً — 2 أسلاك (MDC clock + MDIO data) — بيُستخدم لـ management وconfiguration الـ PHY chips. محدد في **IEEE 802.3**.

الـ **Clause 22** هو النوع القديم: address 5-bit (32 device) + register 5-bit (32 register).
الـ **Clause 45** هو الحديث: address أكبر، لـ 10G وما فوق.

`mdio-regmap` بيدعم Clause 22 فقط.

---

### الـ PCS هو إيه؟

**PCS (Physical Coding Sublayer)** هو جزء من network physical layer بيعمل encoding/decoding للبيانات (زي 8b/10b, 64b/66b). في SoCs حديثة زي Intel SocFPGA، الـ PCS بيكون embedded ومش PHY خارجي — ومحتاج نفس فكرة `mdio-regmap`.

---

### مثال حقيقي: Intel SocFPGA (STMMAC)

في `/workspace/external/linux/drivers/net/ethernet/stmicro/stmmac/dwmac-socfpga.c`:

```c
struct mdio_regmap_config mrc;

mrc.regmap = pcs_regmap;        // الـ regmap اللي بيوصل لـ PCS registers
mrc.parent = priv->device;
mrc.valid_addr = 0x0;           // الـ PHY address داخل الـ virtual bus
mrc.autoscan = false;

snprintf(mrc.name, MII_BUS_ID_SIZE, "%s-pcs-mii", dev_name(priv->device));

pcs_bus = devm_mdio_regmap_register(priv->device, &mrc);
// النتيجة: virtual MDIO bus فوق الـ MMIO registers
```

---

### هيكل الـ Header

الـ file `mdio-regmap.h` بيعرّف حاجتين بس:

#### 1. `struct mdio_regmap_config` — إعدادات الـ virtual bus

```c
struct mdio_regmap_config {
    struct device *parent;      // الـ device الأب (للـ devm management)
    struct regmap *regmap;      // الـ regmap اللي هيتحول لـ MDIO
    char name[MII_BUS_ID_SIZE]; // اسم الـ bus (للـ sysfs والـ debugging)
    u8 valid_addr;              // الـ PHY address الوحيد الصالح على الـ bus
    bool autoscan;              // هيعمل scan تلقائي للـ PHY ولا لأ؟
};
```

#### 2. `devm_mdio_regmap_register()` — الدالة الوحيدة

```c
struct mii_bus *devm_mdio_regmap_register(struct device *dev,
                                          const struct mdio_regmap_config *config);
```

بترجع `struct mii_bus *` — الـ virtual MDIO bus — وبتتعامل مع الـ cleanup تلقائياً عند الـ device removal (devm pattern).

---

### الـ valid_addr و autoscan

لأن الـ bus virtual ومفيش devices حقيقية، لازم تحدد إيه الـ address الصالح:

- **`valid_addr`**: الـ PHY address الوحيد اللي الـ bus هيرد عليه (الـ embedded PHY/PCS بيكون عادةً address 0).
- **`autoscan = true`**: الـ bus هيعمل probe للـ PHY على الـ valid_addr تلقائياً.
- **`autoscan = false`**: الـ driver الأب مسؤول هو إنه يعمل الـ probe يدوياً.

---

### الـ phy_mask trick

```c
if (config->autoscan)
    mii->phy_mask = ~BIT(config->valid_addr);  // unmask الـ valid address فقط
else
    mii->phy_mask = ~0;  // mask كل الـ addresses = مفيش autoscan
```

الـ `phy_mask` في `struct mii_bus` بيحدد أي addresses يتجاهلها الـ bus أثناء الـ scan. قيمة `~0` تعني تجاهل الكل.

---

### الـ Files المهمة

| الملف | الدور |
|-------|-------|
| `include/linux/mdio/mdio-regmap.h` | الـ public API — الـ header الحالي |
| `drivers/net/mdio/mdio-regmap.c` | الـ implementation |
| `include/linux/phy.h` | تعريف `struct mii_bus` و`MII_BUS_ID_SIZE` |
| `include/linux/regmap.h` | تعريف `struct regmap` والـ read/write API |
| `drivers/net/ethernet/stmicro/stmmac/dwmac-socfpga.c` | مثال استخدام حقيقي (Intel SocFPGA PCS) |
| `drivers/net/ethernet/altera/altera_tse_main.c` | مثال استخدام تاني |

---

### الـ Subsystem الأكبر: MDIO Drivers

`mdio-regmap` هو واحد من عائلة كبيرة من الـ MDIO bus drivers في `drivers/net/mdio/`:

- `mdio-bitbang.c` — MDIO عن طريق GPIO bit-banging
- `mdio-i2c.c` — MDIO فوق I2C
- `mdio-gpio.c` — MDIO عن طريق GPIO
- `mdio-regmap.c` — MDIO فوق regmap (MMIO) — الـ file ده

كلهم بيعملوا نفس الفكرة: بيوفروا `struct mii_bus` لـ PHY subsystem، بس من خلال transport layer مختلف.
## Phase 2: شرح الـ mdio-regmap Framework

### المشكلة اللي بيحلها الـ Framework ده

في الـ networking chips الحديثة، خصوصاً في الـ SoCs (زي Intel SoCFPGA أو Altera TSE)، بتلاقي إن الـ **PHY** أو الـ **PCS (Physical Coding Sublayer)** مش موجودين على bus خارجي زي MDIO منفصل — لكنهم **embedded جوه الـ IP block نفسه**، وبيتوصلوا عن طريق **MMIO registers** عادية في address space الـ SoC.

المشكلة: كل الـ kernel networking stack (phylink، phylib، إلخ) بيتكلم مع الـ PHY/PCS عن طريق الـ **MDIO bus abstraction** (`struct mii_bus`). لو الـ PHY عندك MMIO-mapped، ما فيش طريقة مباشرة تسجله في الـ kernel بدون ما تكتب bus driver كامل من الصفر.

**الـ mdio-regmap framework** بيحل مشكلة وحيدة: يخلي MMIO register space يظهر للـ kernel كأنه MDIO bus، عن طريق تحويل أي `regmap` handle لـ `mii_bus` صالح للاستخدام.

---

### الـ Solution — التفكير الكرنل

الـ kernel عنده subsystem اسمه **regmap** — هو abstraction layer يخليك تقرأ وتكتب registers بغض النظر عن الـ transport (MMIO، I2C، SPI، MDIO نفسه...). الـ regmap بيوفر `regmap_read()` و `regmap_write()` بشكل uniform.

الـ mdio-regmap بيعمل جسر بين عالمين:

```
عالم MDIO (mii_bus) ←──── mdio-regmap ────→ عالم regmap (MMIO/I2C/SPI)
```

بدل ما كل driver يكتب logic التحويل ده بنفسه، الـ framework بيعمل ده مرة واحدة وبيصدّر function واحدة بس: `devm_mdio_regmap_register()`.

---

### الـ Real-World Analogy — مع Mapping كامل

تخيل إن عندك **مصنع** (الـ SoC) بداخله **مهندس متخصص** (الـ PCS/PHY الداخلي). المهندس ده بيبعت ويستقبل أوامر عبر **لوحة تحكم داخلية** (MMIO registers) موصولة بكيبل مباشر.

لكن الـ management system (الـ kernel networking stack) بتتعامل فقط مع **الراديو اللاسلكي** (بروتوكول MDIO).

الـ mdio-regmap هو **مترجم** بيجلس بين الاتنين:
- لما الـ management system تقول "ابعت على قناة MDIO رقم 3 رجستر 0x10" → المترجم بيحول ده لـ "اكتب على عنوان 0x10 في الـ MMIO"
- لما يجيه رد من الـ MMIO → يحوله لـ MDIO reply

| الـ Analogy | الـ Kernel Concept |
|---|---|
| Management System | phylink / phylib / network stack |
| الراديو اللاسلكي (MDIO protocol) | `struct mii_bus` و `mii_bus->read/write` |
| المترجم | mdio-regmap driver |
| اللوحة الداخلية | regmap (MMIO) |
| المهندس الداخلي | PCS/PHY embedded في الـ SoC |
| قناة MDIO رقم X | `valid_addr` — العنوان الوحيد الصحيح على الـ bus |

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Linux Networking Stack                │
│          (stmmac / altera_tse / DSA framework)          │
└───────────────────────┬─────────────────────────────────┘
                        │ يطلب PCS/PHY operations
                        ▼
┌─────────────────────────────────────────────────────────┐
│                   phylink subsystem                     │
│         يتكلم مع الـ PCS عبر struct phylink_pcs         │
└───────────────────────┬─────────────────────────────────┘
                        │ phylink_pcs → mii_bus read/write
                        ▼
┌─────────────────────────────────────────────────────────┐
│              struct mii_bus  (MDIO Bus Layer)           │
│   .read  = mdio_regmap_read_c22()                       │
│   .write = mdio_regmap_write_c22()                      │
│   الـ id   = "socfpga-pcs-mii" مثلاً                    │
│   الـ priv = mdio_regmap_priv { regmap, valid_addr }    │
└───────────────────────┬─────────────────────────────────┘
                        │  regmap_read() / regmap_write()
                        ▼
┌─────────────────────────────────────────────────────────┐
│                  regmap subsystem                       │
│   backend: MMIO (devm_regmap_init_mmio)                 │
│   reg_bits=16, val_bits=16                              │
└───────────────────────┬─────────────────────────────────┘
                        │ readw() / writew()
                        ▼
┌─────────────────────────────────────────────────────────┐
│         SoC MMIO Address Space (PCS Registers)         │
│         مثلاً: base_addr + 0x0010 → MII_BMCR            │
└─────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

**الـ mdio-regmap بيعمل shim layer:** بيحول أي `struct regmap` لـ `struct mii_bus` من غير ما تكتب سطر كود زيادة.

الـ struct المحوري:

```c
/* mdio-regmap.h */
struct mdio_regmap_config {
    struct device *parent;   /* الـ device اللي بيمتلك الـ bus */
    struct regmap *regmap;   /* الـ regmap handle (MMIO/I2C/إلخ) */
    char name[MII_BUS_ID_SIZE]; /* اسم الـ bus في الـ sysfs */
    u8 valid_addr;           /* العنوان الوحيد الصحيح على الـ bus */
    bool autoscan;           /* هل الـ kernel يعمل PHY scan تلقائي؟ */
};
```

والـ struct الداخلي المحفوظ في `mii_bus->priv`:

```c
/* mdio-regmap.c */
struct mdio_regmap_priv {
    struct regmap *regmap;   /* نسخة من الـ regmap */
    u8 valid_addr;           /* العنوان المسموح بيه بس */
};
```

#### ليه `valid_addr`؟

الـ MDIO bus عادةً بيدعم لغاية 32 device (addresses 0–31). لكن الـ embedded PCS/PHY الـ MMIO-mapped غالباً PHY واحد بس. لو الـ kernel حاول يتكلم مع عنوان تاني، الـ driver بيرجع `-ENODEV` فوراً:

```c
static int mdio_regmap_read_c22(struct mii_bus *bus, int addr, int regnum)
{
    struct mdio_regmap_priv *ctx = bus->priv;

    /* ارفض أي عنوان غير الـ valid_addr */
    if (ctx->valid_addr != addr)
        return -ENODEV;

    /* حوّل الـ MDIO register read لـ regmap read */
    return regmap_read(ctx->regmap, regnum, &val);
}
```

#### ليه `autoscan`؟

لو `autoscan = true`، الـ framework بيضبط `phy_mask = ~BIT(valid_addr)` — يعني الـ kernel بيعمل scan على العنوان الصح بس ويكتشف الـ PHY تلقائياً من الـ Device Tree. لو `false`، الـ mask = `~0` (ما يشيلش حاجة، الـ driver بيعمل attach يدوي).

---

### العلاقة بين الـ Structs

```
mdio_regmap_config (input from driver)
    │
    ▼
devm_mdio_regmap_register()
    │
    ├─── devm_mdiobus_alloc_size()  → struct mii_bus
    │         │
    │         ├── .read  = mdio_regmap_read_c22
    │         ├── .write = mdio_regmap_write_c22
    │         ├── .priv  ──────────────────────────┐
    │         └── .phy_mask (من autoscan)          │
    │                                              ▼
    │                                   struct mdio_regmap_priv
    │                                       ├── .regmap    (→ MMIO)
    │                                       └── .valid_addr
    │
    └─── devm_mdiobus_register()  → تسجيل في الـ kernel
              │
              └── kernel يشوف mii_bus جاهز للاستخدام
```

---

### مثال حقيقي — Intel SoCFPGA (stmmac)

```c
/* drivers/net/ethernet/stmicro/stmmac/dwmac-socfpga.c */

static int socfpga_dwmac_pcs_init(struct stmmac_priv *priv)
{
    struct regmap_config pcs_regmap_cfg = {
        .reg_bits = 16,   /* register address: 16-bit */
        .val_bits = 16,   /* register value: 16-bit */
        .reg_shift = REGMAP_UPSHIFT(1),  /* word-aligned */
    };

    /* خطوة 1: حوّل الـ MMIO base address لـ regmap */
    pcs_regmap = devm_regmap_init_mmio(priv->device,
                                       dwmac->tse_pcs_base,
                                       &pcs_regmap_cfg);

    /* خطوة 2: اعمل mii_bus من الـ regmap */
    mrc.regmap    = pcs_regmap;
    mrc.parent    = priv->device;
    mrc.valid_addr = 0x0;    /* PHY واحد على عنوان 0 */
    mrc.autoscan  = false;   /* manual attach */

    pcs_bus = devm_mdio_regmap_register(priv->device, &mrc);

    /* خطوة 3: وصّل الـ bus بـ PCS driver (lynx_pcs) */
    pcs = lynx_pcs_create_mdiodev(pcs_bus, 0);
    priv->hw->phylink_pcs = pcs;
}
```

**الـ flow كامل:**
1. `devm_regmap_init_mmio()` → يحول `void __iomem *` لـ `struct regmap *`
2. `devm_mdio_regmap_register()` → يحول `struct regmap *` لـ `struct mii_bus *`
3. `lynx_pcs_create_mdiodev()` → يحول `struct mii_bus *` لـ `struct phylink_pcs *`
4. الـ `phylink` يتكلم مع الـ PCS بشكل موحد

---

### الـ Subsystems التانية المتعلقة

- **regmap subsystem**: لازم تفهمه الأول — هو الـ unified register I/O layer اللي بيدعم MMIO، I2C، SPI كـ backends مختلفة بنفس الـ API.
- **MDIO / phylib**: الـ `struct mii_bus` و `struct phy_device` — الـ bus model اللي الـ kernel بيستخدمه للتواصل مع PHYs.
- **phylink**: الـ layer فوق phylib، بيدير الـ MAC ↔ PCS ↔ PHY negotiation في الـ modern Ethernet drivers.

---

### الـ Ownership — مين بيملك إيه؟

| المسؤولية | الـ mdio-regmap Framework | الـ Driver اللي بيستخدمه |
|---|---|---|
| تخصيص `mii_bus` | نعم (devm) | لا |
| تسجيل الـ bus في الـ kernel | نعم | لا |
| تحويل MDIO read/write لـ regmap | نعم | لا |
| إنشاء الـ `regmap` (MMIO/I2C/SPI) | لا | نعم |
| تحديد `valid_addr` | لا (config) | نعم |
| وصل الـ `mii_bus` بـ PCS driver | لا | نعم |
| الـ `devm` cleanup عند remove | نعم | لا |

الـ framework صغير ومركّز جداً — أقل من 100 سطر كود — لأن فلسفته إنه يعمل شيء واحد بس ويعمله صح.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Config Options — Cheatsheet

| الثابت / الـ Field | النوع | القيمة / المعنى |
|---|---|---|
| `MII_BUS_ID_SIZE` | macro | 61 بايت — الحجم الأقصى لـ string اسم الـ bus |
| `autoscan` | `bool` في `mdio_regmap_config` | `true` → scan تلقائي للـ PHY على الـ bus |
| `valid_addr` | `u8` | الـ MDIO address الوحيد الصالح (0–31) |
| `phy_mask` | `u32` في `mii_bus` | bitmask للـ addresses اللي هتتجاهل أثناء الـ probe |
| `~BIT(valid_addr)` | تعبير | كل الـ addresses تتجاهل ماعدا `valid_addr` |
| `~0` | تعبير | كل الـ addresses تتجاهل (no autoscan) |
| `MDIOBUS_ALLOCATED` | enum state | الـ bus اتحجز بس مش مسجل |
| `MDIOBUS_REGISTERED` | enum state | الـ bus شغّال وجاهز |
| `MDIOBUS_UNREGISTERED` | enum state | اتفك التسجيل |
| `MDIOBUS_RELEASED` | enum state | اتحرر بالكامل |

---

### 1. الـ Structs المهمة

#### `struct mdio_regmap_config`
**الغرض:** الـ configuration object اللي بيتبعته الـ caller لـ `devm_mdio_regmap_register` — بيحدد فيه كل إعدادات الـ virtual MDIO bus.

```c
struct mdio_regmap_config {
    struct device  *parent;      /* الـ device الأب اللي هيمتلك الـ bus */
    struct regmap  *regmap;      /* الـ regmap المربوط بالـ MMIO region */
    char            name[MII_BUS_ID_SIZE]; /* اسم الـ bus (max 61 char) */
    u8              valid_addr;  /* MDIO address الصالح الوحيد (0-31) */
    bool            autoscan;   /* هل تعمل scan تلقائي للـ PHY؟ */
};
```

| الـ Field | الاستخدام الفعلي |
|---|---|
| `parent` | بيتحقق منه أول حاجة — لو `NULL` بيرجع `-EINVAL` |
| `regmap` | بيتنقل لـ `mdio_regmap_priv.regmap` اللي بيستخدمه الـ read/write |
| `name` | بيتنسخ لـ `mii_bus.id` عن طريق `strscpy` |
| `valid_addr` | الفلتر — أي read/write على address تاني بيرجع `-ENODEV` |
| `autoscan` | بيتحكم في `phy_mask` — لو `true` بيكشف الـ PHY تلقائياً |

---

#### `struct mdio_regmap_priv` (داخلي في الـ driver)
**الغرض:** الـ private context المخبي في `mii_bus.priv` — اللي بيشيله الـ read/write callbacks.

```c
struct mdio_regmap_priv {
    struct regmap *regmap;    /* مؤشر للـ regmap (من الـ config) */
    u8             valid_addr; /* الـ address الصالح للفلترة */
};
```

---

#### `struct mii_bus`
**الغرض:** يمثّل الـ MDIO bus الكامل في الـ kernel — ده الـ struct اللي الـ PHY subsystem بيشوفه.

| الـ Field المهم | الدور في سياق الـ mdio-regmap |
|---|---|
| `name` | ثابت = `"mdio-regmap"` |
| `id[MII_BUS_ID_SIZE]` | بيتملا من `config->name` |
| `priv` | بيشاور على `mdio_regmap_priv` |
| `read` | بيتربط بـ `mdio_regmap_read_c22` |
| `write` | بيتربط بـ `mdio_regmap_write_c22` |
| `parent` | نفس `config->parent` |
| `phy_mask` | بيتحكم فيه `autoscan` |
| `mdio_lock` | mutex داخلي يحمي الـ bus access |

---

#### `struct regmap` (مرجع خارجي)
**الغرض:** abstraction layer للـ register access — بيخفي تفاصيل الـ MMIO/I2C/SPI عن الـ driver.

الـ mdio-regmap بيستخدم فقط:
- `regmap_read(regmap, regnum, &val)` — قراءة register
- `regmap_write(regmap, regnum, val)` — كتابة register

---

### 2. مخطط علاقات الـ Structs

```
caller (e.g. platform driver)
        │
        │  fills
        ▼
┌─────────────────────────┐
│   mdio_regmap_config    │
│  ┌─────────────────┐   │
│  │ parent ─────────┼───┼──► struct device
│  │ regmap ─────────┼───┼──► struct regmap ──► MMIO registers
│  │ name[]          │   │
│  │ valid_addr      │   │
│  │ autoscan        │   │
│  └─────────────────┘   │
└─────────────────────────┘
        │
        │  passed to
        ▼
devm_mdio_regmap_register()
        │
        │  allocates
        ▼
┌─────────────────────────────────────────┐
│            struct mii_bus               │
│  name = "mdio-regmap"                   │
│  id   = config->name                   │
│  priv ──────────────────────────────►  │
│  read  = mdio_regmap_read_c22          │  ┌────────────────────┐
│  write = mdio_regmap_write_c22         │  │ mdio_regmap_priv   │
│  parent = config->parent               │  │ regmap ──► regmap  │
│  phy_mask (from autoscan)              │  │ valid_addr         │
│  mdio_lock (mutex)                     │  └────────────────────┘
│  mdio_map[] ── after register ──►      │
│    struct mdio_device[PHY_MAX_ADDR]    │
└─────────────────────────────────────────┘
        │
        │  registered via devm_mdiobus_register()
        ▼
   PHY subsystem
   phy_device instances per address
```

---

### 3. مخطط الـ Lifecycle

```
[CREATION]
    caller يجهّز mdio_regmap_config
    │
    ▼
devm_mdio_regmap_register(dev, config)
    │
    ├─ validate: config->parent != NULL
    │
    ├─ devm_mdiobus_alloc_size(parent, sizeof(mdio_regmap_priv))
    │     └─► mii_bus اتحجز + priv space في نفس الـ allocation
    │         state = MDIOBUS_ALLOCATED
    │
    ├─ ملء mii_bus:
    │     priv->regmap    = config->regmap
    │     priv->valid_addr = config->valid_addr
    │     mii->read       = mdio_regmap_read_c22
    │     mii->write      = mdio_regmap_write_c22
    │     mii->phy_mask   = autoscan ? ~BIT(addr) : ~0
    │
    ├─ devm_mdiobus_register(dev, mii)
    │     state = MDIOBUS_REGISTERED
    │     └─► PHY subsystem يبدأ probe على الـ addresses غير المحجوبة
    │
    └─► يرجع *mii_bus للـ caller

[USAGE]
    PHY driver يقرأ/يكتب عبر mii_bus
    │
    ├─ mdio_regmap_read_c22(bus, addr, regnum)
    │     ├─ addr != valid_addr ──► -ENODEV
    │     └─ regmap_read(regmap, regnum, &val) ──► MMIO read
    │
    └─ mdio_regmap_write_c22(bus, addr, regnum, val)
          ├─ addr != valid_addr ──► -ENODEV
          └─ regmap_write(regmap, regnum, val) ──► MMIO write

[TEARDOWN]
    devm cleanup عند remove الـ parent device
    │
    ├─ devm_mdiobus_register cleanup:
    │     mdiobus_unregister(mii)
    │     state = MDIOBUS_UNREGISTERED
    │
    └─ devm_mdiobus_alloc_size cleanup:
          mdiobus_free(mii)
          state = MDIOBUS_RELEASED
```

---

### 4. مخطط الـ Call Flow

#### قراءة register من PHY driver

```
PHY driver
  └─► phy_read(phydev, regnum)
        └─► mdiobus_read(mii_bus, addr, regnum)
              └─► mutex_lock(&bus->mdio_lock)          ← lock الـ bus
                    └─► bus->read(bus, addr, regnum)
                          └─► mdio_regmap_read_c22()
                                ├─ ctx = bus->priv      ← جيب الـ priv
                                ├─ if addr != valid_addr
                                │    return -ENODEV     ← فلترة الـ address
                                └─ regmap_read(ctx->regmap, regnum, &val)
                                      └─► MMIO read from hardware register
                                            return val
              └─► mutex_unlock(&bus->mdio_lock)         ← unlock
```

#### كتابة register من PHY driver

```
PHY driver
  └─► phy_write(phydev, regnum, val)
        └─► mdiobus_write(mii_bus, addr, regnum, val)
              └─► mutex_lock(&bus->mdio_lock)
                    └─► bus->write(bus, addr, regnum, val)
                          └─► mdio_regmap_write_c22()
                                ├─ if addr != valid_addr
                                │    return -ENODEV
                                └─ regmap_write(ctx->regmap, regnum, val)
                                      └─► MMIO write to hardware register
              └─► mutex_unlock(&bus->mdio_lock)
```

#### تسجيل الـ bus (Registration Flow)

```
devm_mdio_regmap_register(dev, config)
  │
  ├─► devm_mdiobus_alloc_size(config->parent, sizeof(*mr))
  │     └─► mdiobus_alloc_size()
  │           └─► kzalloc(sizeof(mii_bus) + sizeof_priv)
  │                 └─► returns mii_bus*  [MDIOBUS_ALLOCATED]
  │
  ├─► setup ops: read/write/name/id/parent/phy_mask
  │
  └─► devm_mdiobus_register(dev, mii)
        └─► __devm_mdiobus_register(dev, mii, THIS_MODULE)
              └─► __mdiobus_register(mii, owner)
                    ├─ device_register(&mii->dev)
                    ├─ of_mdiobus_register()  [if DT node exists]
                    │    └─► probe each child PHY from DT
                    └─► scan non-masked addresses
                          └─► mdiobus_scan_c22(bus, addr)  per addr
                                └─► get_phy_device() → phy_device_create()
                                      └─► phy_device registered in sysfs
```

---

### 5. استراتيجية الـ Locking

#### الـ Locks المستخدمة

| الـ Lock | النوع | موجود في | يحمي |
|---|---|---|---|
| `mii_bus.mdio_lock` | `struct mutex` | `struct mii_bus` | كل عمليات الـ read/write على الـ bus — يمنع التزامن |
| `mii_bus.shared_lock` | `struct mutex` | `struct mii_bus` | الـ `shared` state بين الـ PHYs (PHY package) |
| `regmap` internal lock | داخلي في الـ regmap | `struct regmap` | الـ MMIO access — بيتحكم فيه الـ regmap config |

#### ترتيب الـ Locking (Lock Ordering)

```
mdio_lock (mii_bus level)
    └─► regmap internal lock (register level)
```

لا يوجد lock في `mdio_regmap_read_c22` / `mdio_regmap_write_c22` مباشرة — الـ `mdio_lock` بياخده الـ `mdiobus_read/write` في الـ MDIO core قبل ما يستدعي الـ callbacks، والـ regmap بيدير الـ locking الداخلي الخاص بيه.

#### ملاحظة مهمة على `valid_addr`

الـ `valid_addr` مش محتاج lock لأنه **read-only بعد الـ init** — بيتكتب مرة واحدة في `devm_mdio_regmap_register` وبعدها بيتقرأ فقط.

#### نموذج عملي للـ locking

```c
/* mdiobus_read in mdio core — يشيل الـ lock قبل callback */
int mdiobus_read(struct mii_bus *bus, int addr, u32 regnum)
{
    int ret;

    mutex_lock(&bus->mdio_lock);       /* ← lock الـ bus */
    ret = bus->read(bus, addr, regnum);/* ← callback للـ mdio_regmap_read_c22 */
    mutex_unlock(&bus->mdio_lock);     /* ← unlock */

    return ret;
}
```

بكده الـ `mdio_regmap_read_c22` دايماً بتشتغل تحت الـ `mdio_lock`، ومش محتاجة تشيله تاني بنفسها.
## Phase 4: شرح الـ Functions

### ملخص الـ API — Cheatsheet

| Function | Category | الغرض |
|---|---|---|
| `devm_mdio_regmap_register()` | Registration | بتسجّل MDIO bus فوق regmap وبتربطها بـ device lifecycle |
| `mdio_regmap_read_c22()` | Runtime I/O (static) | بتقرأ register من الـ PHY عبر regmap — C22 clause |
| `mdio_regmap_write_c22()` | Runtime I/O (static) | بتكتب register في الـ PHY عبر regmap — C22 clause |

---

### Category 1: Registration

الـ registration هي الخطوة الوحيدة اللي بيشوفها الـ driver الخارجي. هي بتعمل allocate للـ `mii_bus`، بتربطه بالـ `regmap`، وبتسجّله في الـ MDIO subsystem — كل ده في call واحدة.

---

#### `devm_mdio_regmap_register`

```c
struct mii_bus *devm_mdio_regmap_register(struct device *dev,
                                          const struct mdio_regmap_config *config);
```

**بتعمل إيه:**
بتعمل allocate لـ `mii_bus` بـ managed memory (devres) فوق `config->parent`، بتعبّي الـ bus بـ `read`/`write` callbacks اللي بتشتغل عبر `regmap`، وبعدين بتسجّل الـ bus في kernel. لو `config->autoscan` تساوي true، بتضبط `phy_mask` عشان تسمح بـ scan للـ valid_addr بس.

**Parameters:**

| Parameter | النوع | الشرح |
|---|---|---|
| `dev` | `struct device *` | الـ device اللي بتتسجّل عليه الـ managed resources (devres owner) |
| `config` | `const struct mdio_regmap_config *` | الـ config struct اللي فيها الـ regmap، الـ parent، الـ address، والـ name |

**الـ `mdio_regmap_config` fields:**

| Field | النوع | الشرح |
|---|---|---|
| `parent` | `struct device *` | الـ parent device — لازم يكون موجود، لو NULL بترجع `-EINVAL` |
| `regmap` | `struct regmap *` | الـ regmap instance اللي فيه registers الـ PHY/PCS |
| `name` | `char[MII_BUS_ID_SIZE]` | الـ bus ID string — بيتعمله `strscpy` في `mii->id` |
| `valid_addr` | `u8` | الـ MDIO address الوحيد المقبول على الـ bus دي |
| `autoscan` | `bool` | لو true، بيضبط `phy_mask` عشان يسمح بـ scan للـ valid_addr تلقائياً |

**Return Value:**
- `struct mii_bus *` valid pointer لو نجح التسجيل
- `ERR_PTR(-EINVAL)` لو `config->parent == NULL`
- `ERR_PTR(-ENOMEM)` لو فشل الـ allocation
- `ERR_PTR(-Exxx)` لو `devm_mdiobus_register()` فشل

**Key Details:**
- الـ allocation بتتعمل على `config->parent` مش على `dev` — الفرق مهم: الـ devres cleanup بيتربط بالـ `dev` parameter مش الـ `parent`.
- بتستخدم `devm_mdiobus_alloc_size()` عشان تعمل alloc للـ bus مع private data (`mdio_regmap_priv`) في نفس allocation.
- بتستخدم `devm_mdiobus_register()` — يعني الـ unregister بيحصل تلقائي لما `dev` يتـ destroy.
- الـ `phy_mask`:
  - لو `autoscan = true`: `phy_mask = ~BIT(valid_addr)` → كل addresses محجوبة ماعدا الـ valid_addr.
  - لو `autoscan = false`: `phy_mask = ~0` → كل addresses محجوبة — مفيش scan خالص.
- بتطبع error message على `config->parent` (مش على `dev`) لو فشل الـ registration.

**Caller Context:**
- بتتكلم من probe context فقط.
- مش thread-safe لو اتكلمت concurrently على نفس الـ `dev`.

**Pseudocode Flow:**

```
devm_mdio_regmap_register(dev, config):
    if config->parent == NULL:
        return ERR_PTR(-EINVAL)

    mii = devm_mdiobus_alloc_size(config->parent, sizeof(mdio_regmap_priv))
    if !mii:
        return ERR_PTR(-ENOMEM)

    mr = mii->priv
    mr->regmap    = config->regmap
    mr->valid_addr = config->valid_addr

    mii->name   = "mdio-regmap"
    mii->id     = config->name   // via strscpy
    mii->parent = config->parent
    mii->read   = mdio_regmap_read_c22
    mii->write  = mdio_regmap_write_c22

    if config->autoscan:
        mii->phy_mask = ~BIT(config->valid_addr)  // scan valid_addr only
    else:
        mii->phy_mask = ~0                         // no autoscan at all

    rc = devm_mdiobus_register(dev, mii)
    if rc:
        dev_err(config->parent, ...)
        return ERR_PTR(rc)

    return mii
```

---

### Category 2: Runtime I/O (Static Internal Functions)

الـ functions دي static — ما بتظهرش في الـ header، بس هي جوهر الـ driver. بتشتغل كـ callbacks اللي بيسجّلهم `devm_mdio_regmap_register` في `mii->read` و`mii->write`. الـ MDIO core بيكلمهم وقت كل read/write operation.

---

#### `mdio_regmap_read_c22`

```c
static int mdio_regmap_read_c22(struct mii_bus *bus, int addr, int regnum);
```

**بتعمل إيه:**
بتتحقق إن الـ `addr` هو نفس `valid_addr` المسموح بيه، لو مش كده بترجع `-ENODEV`. لو تمام، بتعمل `regmap_read()` من الـ register رقم `regnum` وبترجع القيمة.

**Parameters:**

| Parameter | النوع | الشرح |
|---|---|---|
| `bus` | `struct mii_bus *` | الـ MDIO bus — منه بنجيب `priv` اللي فيه الـ regmap والـ valid_addr |
| `addr` | `int` | الـ MDIO PHY address اللي الـ core عايز يقرأ منه |
| `regnum` | `int` | رقم الـ register الـ C22 (0–31) |

**Return Value:**
- القيمة المقروءة (unsigned int مصبوح على int) لو نجح
- `-ENODEV` لو `addr != valid_addr`
- قيمة negative error من `regmap_read()` لو فشلت القراءة

**Key Details:**
- الـ locking بيتعمل في الـ MDIO core قبل ما يكلم الـ callback — مش محتاج lock جوّاها.
- الـ regmap نفسه ممكن يكون فيه locking داخلي حسب config بتاعته.
- الفرق عن C45: مفيش `devnum` هنا — C22 بيعنوّن بـ addr + regnum بس.

---

#### `mdio_regmap_write_c22`

```c
static int mdio_regmap_write_c22(struct mii_bus *bus, int addr, int regnum, u16 val);
```

**بتعمل إيه:**
بتتحقق من الـ `addr`، لو مش الـ valid_addr بترجع `-ENODEV`. لو تمام، بتعمل `regmap_write()` بالقيمة `val` في الـ register رقم `regnum`.

**Parameters:**

| Parameter | النوع | الشرح |
|---|---|---|
| `bus` | `struct mii_bus *` | الـ MDIO bus |
| `addr` | `int` | الـ MDIO PHY address |
| `regnum` | `int` | رقم الـ register |
| `val` | `u16` | القيمة المراد كتابتها — بتتوسّع ضمنياً لـ unsigned int عند تمريرها لـ regmap |

**Return Value:**
- `0` لو نجحت الكتابة
- `-ENODEV` لو `addr != valid_addr`
- قيمة negative error من `regmap_write()` لو فشلت

**Key Details:**
- الـ locking نفس الكلام — بيتعمل في الـ MDIO core.
- الـ `val` هو `u16` بس الـ `regmap_write` بياخد `unsigned int` — مفيش مشكلة في الـ zero-extension.
- مفيش read-modify-write — write مباشر للـ register.

---

### ملاحظات معمارية

```
Driver (e.g., stmmac, dwmac)
        |
        | calls devm_mdio_regmap_register()
        v
  mdio-regmap layer
        |         \
        |          \-- mdio_regmap_read_c22  \
        |           -- mdio_regmap_write_c22  \--> regmap --> MMIO registers
        |
        v
  mii_bus registered in MDIO subsystem
        |
        v
  PHY/PCS drivers use standard phy_read()/phy_write() API
```

الـ `valid_addr` فكرة مهمة: بما إن الـ IP block عنده PHY/PCS واحد mapped على MMIO، الـ bus بتوفّر address واحد صالح بس. أي access على address تاني بيرجع `-ENODEV` عشان يمنع الـ MDIO core من الـ scan على addresses وهمية.
## Phase 5: دليل الـ Debugging الشامل

الـ `mdio-regmap` driver بسيط جداً في تصميمه — هو بس layer وسطى بين `mii_bus` API والـ `regmap` API. لكن لما حاجة بتغلط، المشكلة ممكن تكون في أي مستوى: الـ regmap نفسه، الـ MDIO bus scan، أو الـ PHY/PCS اللي فوقيهم. الـ cheat sheet ده بيغطي كل المستويات.

---

### Software Level

#### 1. Debugfs Entries

الـ regmap تلقائياً بيعمل debugfs entries لو الـ kernel اتبنى بـ `CONFIG_REGMAP` (وهو كده دايماً). الـ entries دي بتسمحلك تقرأ وتكتب registers مباشرةً.

```bash
# اعرف اسم الـ regmap الخاص بالـ device
ls /sys/kernel/debug/regmap/

# مثال: device اسمه "ff200000.ethernet-pcs-mii"
ls /sys/kernel/debug/regmap/ff200000.ethernet-pcs-mii/
# هتلاقي: registers  access  cache_bypass  cache_only  cache_dirty  range

# اقرأ كل الـ registers المتاحة
cat /sys/kernel/debug/regmap/ff200000.ethernet-pcs-mii/registers
# output مثال:
# 00: 1140  <-- MII_BMCR: autoneg enabled, full-duplex, 100Mbps
# 01: 796d  <-- MII_BMSR: autoneg complete, link up
# 02: 0022  <-- MII_PHYSID1
# 03: 1622  <-- MII_PHYSID2
# 04: 01e1  <-- MII_ADVERTISE
# 05: 0000  <-- MII_LPA (no partner on MMIO PCS)

# اكتب register مباشرة (مفيد للـ forced reset)
echo "0x00 0x8000" > /sys/kernel/debug/regmap/ff200000.ethernet-pcs-mii/registers

# تفعيل cache_bypass لضمان قراءة hardware مباشرة
echo 1 > /sys/kernel/debug/regmap/ff200000.ethernet-pcs-mii/cache_bypass
cat /sys/kernel/debug/regmap/ff200000.ethernet-pcs-mii/registers
echo 0 > /sys/kernel/debug/regmap/ff200000.ethernet-pcs-mii/cache_bypass
```

**الـ entries المهمة:**

| Entry | الوظيفة |
|-------|----------|
| `registers` | قراءة/كتابة كل الـ registers |
| `cache_bypass` | تجاهل الـ cache واقرأ من hardware مباشرة |
| `cache_only` | اقرأ من cache فقط (بدون hardware access) |
| `cache_dirty` | شوف لو الـ cache مش sync مع hardware |
| `access` | شوف access permissions لكل register |
| `range` | الـ address range المدعوم |

---

#### 2. Sysfs Entries

الـ `mii_bus` بيتسجل كـ device على bus اسمه `mdio_bus` في sysfs.

```bash
# اعرف كل الـ MDIO buses المسجلة
ls /sys/bus/mdio_bus/devices/
# مثال output:
# ff200000.ethernet-pcs-mii:00

# معلومات الـ bus
cat /sys/bus/mdio_bus/devices/ff200000.ethernet-pcs-mii:00/uevent

# شوف الـ PHY device المرتبط بالـ bus
ls /sys/bus/mdio_bus/devices/ff200000.ethernet-pcs-mii:00/
# هتلاقي: driver  modalias  of_node  power  subsystem  uevent

# الـ driver المرتبط
cat /sys/bus/mdio_bus/devices/ff200000.ethernet-pcs-mii:00/driver
# رابط لـ /sys/bus/mdio_bus/drivers/intel-xpcs مثلاً

# تحقق من الـ PHY ID المكتشف
cat /sys/bus/mdio_bus/devices/ff200000.ethernet-pcs-mii:00/phy_id 2>/dev/null || echo "not a phy_device"
```

---

#### 3. Ftrace — الـ `mdio_access` Tracepoint

الـ tracepoint الوحيد المتاح هو `mdio_access` وده بيغطي كل قراءة/كتابة على الـ bus.

```bash
# تفعيل الـ tracepoint
echo 1 > /sys/kernel/debug/tracing/events/mdio/mdio_access/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل الـ application أو انتظر network activity
# مثلاً: force link detection
ip link set eth0 down && ip link set eth0 up

# اقرأ النتائج
cat /sys/kernel/debug/tracing/trace | grep mdio_access

# مثال output:
# kworker/0:2-45    [000] ..... 1234.567890: mdio_access: ff200000.ethernet-pcs-mii read  phy:0x00 reg:0x01 val:0x796d
# kworker/0:2-45    [000] ..... 1234.567891: mdio_access: ff200000.ethernet-pcs-mii write phy:0x00 reg:0x00 val:0x1140
```

**تفسير output الـ tracepoint:**

| Field | المعنى |
|-------|--------|
| `ff200000.ethernet-pcs-mii` | اسم الـ mii_bus id |
| `read`/`write` | نوع العملية |
| `phy:0x00` | الـ MDIO address = `valid_addr` في الـ config |
| `reg:0x01` | رقم الـ register (MII_BMSR = 1) |
| `val:0x796d` | القيمة |

**ملاحظة:** الـ tracepoint بيتفعل بس لو `err >= 0` — يعني لو بيرجع `-ENODEV` (لأن `addr != valid_addr`) مش هيظهر في trace.

```bash
# فلترة على bus معين بس
echo 'busid == "ff200000.ethernet-pcs-mii"' > /sys/kernel/debug/tracing/events/mdio/mdio_access/filter
```

---

#### 4. Printk / Dynamic Debug

```bash
# تفعيل dynamic debug للـ mdio-regmap module
echo "module mdio_regmap +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل dynamic debug للـ mdio_bus core
echo "module mdio_bus +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug لـ regmap operations على device معين
echo "file drivers/base/regmap/regmap.c +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل كل debug messages في subsystem الـ net
echo "module phy +p" > /sys/kernel/debug/dynamic_debug/control

# شوف الـ messages في dmesg
dmesg -w | grep -E "mdio|regmap|mii"
```

---

#### 5. Kernel Config Options للـ Debugging

| Config | الوظيفة |
|--------|---------|
| `CONFIG_MDIO_REGMAP` | الـ driver نفسه (tristate) |
| `CONFIG_REGMAP` | الـ regmap core — لازم يتفعل |
| `CONFIG_REGMAP_MMIO` | لو الـ backend هو MMIO |
| `CONFIG_REGMAP_MDIO` | لو الـ backend هو MDIO |
| `CONFIG_PHYLIB` | الـ PHY library — لازم للـ mii_bus |
| `CONFIG_MDIO_BUS` | الـ MDIO bus infrastructure |
| `CONFIG_DEBUG_KERNEL` | تفعيل general kernel debug |
| `CONFIG_DYNAMIC_DEBUG` | dynamic debug (`+p` في runtime) |
| `CONFIG_TRACING` | ftrace infrastructure |
| `CONFIG_KALLSYMS` | symbol names في stack traces |
| `CONFIG_DEBUG_LOCK_ALLOC` | debug الـ mutex اللي بيحمي الـ bus |
| `CONFIG_PROVE_LOCKING` | lockdep للـ mdiobus_mutex |

```bash
# تحقق من الـ config الحالي
grep -E "CONFIG_MDIO|CONFIG_REGMAP|CONFIG_PHYLIB" /boot/config-$(uname -r)
# أو
zcat /proc/config.gz | grep -E "CONFIG_MDIO|CONFIG_REGMAP"
```

---

#### 6. أدوات خاصة بالـ Subsystem

**الـ `mdio-netlink` / `ethtool`:**

```bash
# اقرأ PHY registers عبر ethtool (لو الـ PHY اتعرف)
ethtool -d eth0 2>/dev/null
# أو باستخدام raw register access
ethtool --phy-statistics eth0

# اقرأ register بـ ethtool عبر MDIO directly
# MII register 1 (BMSR) من PHY address 0
mii-tool -v eth0 2>/dev/null || echo "install mii-tool"

# phytool (أشمل)
phytool read eth0/0/1   # bus/addr/reg → قرأ BMSR
phytool write eth0/0/0 0x8000  # soft reset

# devlink لو PCS/MAC driver يدعمه
devlink dev show
devlink dev info pci/0000:00:00.0 2>/dev/null

# اقرأ regmap registers بـ script
for reg in $(seq 0 6); do
    val=$(cat /sys/kernel/debug/regmap/*/registers 2>/dev/null | awk -v r=$(printf "%02x" $reg) '$1==r":" {print $2}')
    echo "reg $reg = 0x$val"
done
```

---

#### 7. جدول Error Messages الشائعة

| Error / Log Message | السبب | الحل |
|---------------------|-------|------|
| `Cannot register MDIO bus![name] (-22)` | `config->parent == NULL` أو اسم الـ bus غلط | تأكد إن `config->parent` مش NULL قبل الاستدعاء |
| `Cannot register MDIO bus![name] (-12)` | ما فيش ذاكرة لـ `devm_mdiobus_alloc_size` | تحقق من memory pressure، قلل allocations |
| `Cannot register MDIO bus![name] (-16)` | الـ bus name متكرر (EBUSY) | غير `config->name` بحيث يكون unique |
| `mdio_regmap_read_c22: ENODEV` | `addr != valid_addr` | الـ PHY driver بيحاول يقرأ address غلط |
| `mdio_regmap_write_c22: ENODEV` | نفس السبب | تحقق من `valid_addr` في config |
| `regmap_read failed: -EIO` | الـ MMIO region مش accessible | تحقق من clock/power enable للـ IP |
| `of_mdiobus_register failed` | الـ DT node للـ MDIO مش صح | راجع DT bindings |
| `PHY xx not found` | `autoscan = false` و driver ما عملش explicit scan | إما فعّل `autoscan` أو اعمل `mdiobus_scan_c22` manually |

---

#### 8. Strategic Points لـ `dump_stack()` / `WARN_ON()`

```c
struct mii_bus *devm_mdio_regmap_register(struct device *dev,
					  const struct mdio_regmap_config *config)
{
	struct mdio_regmap_priv *mr;
	struct mii_bus *mii;
	int rc;

	/* نقطة 1: تحقق من الـ config قبل أي حاجة */
	if (WARN_ON(!config->parent))
		return ERR_PTR(-EINVAL);

	if (WARN_ON(!config->regmap))  /* regmap NULL = crash لاحقاً */
		return ERR_PTR(-EINVAL);

	mii = devm_mdiobus_alloc_size(config->parent, sizeof(*mr));
	if (!mii)
		return ERR_PTR(-ENOMEM);

	/* ... */

	rc = devm_mdiobus_register(dev, mii);
	if (rc) {
		/* نقطة 2: dump_stack هنا بيساعد تعرف من استدعى الـ register */
		dump_stack();
		dev_err(config->parent, "Cannot register MDIO bus![%s] (%d)\n",
			mii->id, rc);
		return ERR_PTR(rc);
	}

	return mii;
}

/* نقطة 3: في read/write، تحقق من الـ regmap مش NULL */
static int mdio_regmap_read_c22(struct mii_bus *bus, int addr, int regnum)
{
	struct mdio_regmap_priv *ctx = bus->priv;

	WARN_ON(!ctx->regmap);  /* لو NULL هنا، كان المفروض يتمسك في register */

	if (ctx->valid_addr != addr)
		return -ENODEV;
	/* ... */
}
```

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State يطابق الـ Kernel State

الـ `mdio-regmap` بيعمل map لـ registers مكتوبة في الـ MMIO space. الـ "PHY" هنا عبارة عن IP block (زي PCS) داخل الـ SoC.

```bash
# قرأ BMSR (register 1) — المفروض link bit = 1 لو كابل متوصل
# من خلال regmap debugfs
cat /sys/kernel/debug/regmap/*/registers | grep "^01:"
# output مثال: 01: 796d
# bit 2 = link status → 0x796d & 0x4 = 4 → link up

# تحقق من الـ kernel state للـ link
ip link show eth0 | grep -E "state|link"
cat /sys/class/net/eth0/carrier          # 1 = up, 0 = down
cat /sys/class/net/eth0/operstate        # up/down/unknown

# قارن BMCR (register 0) بين hardware وـ ethtool
phytool read eth0/0/0  # hardware read
ethtool eth0 | grep -E "Speed|Duplex|Auto"  # kernel view
```

---

#### 2. Register Dump Techniques

الـ IP block بيكون في الـ physical memory space — تقدر تقرأه بـ `devmem2` أو `/dev/mem`.

```bash
# اعرف الـ base address من DT أو من /proc/iomem
grep -i "pcs\|ethernet\|mdio" /proc/iomem | head -10
# مثال output:
# ff200000-ff20ffff : ff200000.ethernet

# قرأ BMCR (offset 0x00) — كل register هو 16-bit لكن aligned على 32-bit
# base = 0xff200000
devmem2 0xff200000 w    # register 0 (BMCR)
devmem2 0xff200004 w    # register 1 (BMSR)
devmem2 0xff200008 w    # register 2 (PHYSID1)
devmem2 0xff20000c w    # register 3 (PHYSID2)

# أو بـ /dev/mem (أبطأ وأقل أمناً)
dd if=/dev/mem bs=2 count=8 skip=$((0xff200000/2)) 2>/dev/null | xxd

# مقارنة شاملة للـ MII registers (0..6)
BASE=0xff200000
for i in 0 1 2 3 4 5 6; do
    addr=$(printf "0x%x" $((BASE + i*4)))
    val=$(devmem2 $addr w 2>/dev/null | awk '/Value at/ {print $NF}')
    echo "reg[$i] @ $addr = $val"
done
```

**ملاحظة:** الـ offset بين registers بيعتمد على `reg_bits` و `val_bits` في الـ `regmap_config` — تحقق من الـ driver المستخدم.

---

#### 3. Logic Analyzer / Oscilloscope Tips

الـ MMIO-mapped MDIO مش بيستخدم serial MDIO lines (MDC/MDIO) فيزيائياً — هو access لـ memory. لكن:

- **لو الـ IP بيتكلم مع PHY خارجي** عبر MDIO حقيقي، احتاج logic analyzer.
- **لو الـ PCS داخلي** (MMIO فقط)، الـ oscilloscope مش مفيد — استخدم debugfs.

```
Logic Analyzer Setup (للـ external MDIO):
- Channel 1: MDC (clock) — typical 2.5MHz
- Channel 2: MDIO (data) — bidirectional
- Trigger: rising edge على MDC

Protocol Decode:
- Preamble: 32 bits HIGH
- Start: 01
- Op: 10 (read) / 01 (write)
- PHY addr: 5 bits (= valid_addr)
- REG addr: 5 bits
- Turnaround: 2 bits
- Data: 16 bits
```

---

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة | Kernel Log Pattern | التشخيص |
|---------|-------------------|---------|
| Clock الـ IP مش enabled | `regmap_read failed` بعد كل قراءة | فعّل clock بـ `clk_prepare_enable` |
| Power domain مش مفتوح | `Cannot register MDIO bus! (-5)` EIO | فعّل power domain |
| Wrong base address في DT | القيم كلها `0xFFFF` أو `0x0000` | تحقق من `reg` property في DT |
| Reset لسه active | BMSR = 0x0000 أو 0xFFFF | release reset GPIO |
| Endianness مغلوط | قيم عكوسة، مثلاً PHYSID = 0x2200 بدل 0x0022 | عدّل `val_format_endian` في regmap_config |
| Address stride غلط | كل قراءة بترجع نفس القيمة | تحقق من `reg_stride` في regmap_config |

---

#### 5. Device Tree Debugging

```bash
# تحقق من الـ DT node المحمّل فعلاً
# افتراض: device على عنوان ff200000
ls /sys/firmware/devicetree/base/soc/ethernet@ff200000/ 2>/dev/null || \
    find /sys/firmware/devicetree/base -name "*.ethernet*" 2>/dev/null | head -5

# اقرأ الـ compatible string
cat /sys/firmware/devicetree/base/soc/ethernet@ff200000/compatible 2>/dev/null | tr '\0' '\n'

# تحقق من الـ reg property (base address)
xxd /sys/firmware/devicetree/base/soc/ethernet@ff200000/reg 2>/dev/null
# مثال: 00000000 ff200000 00000000 00010000
# يعني: base=0xff200000, size=0x10000

# استخدم dtc لتحويل الـ DTB لـ DTS قابل للقراءة
dtc -I dtb -O dts /sys/firmware/fdt 2>/dev/null | grep -A 20 "ethernet@ff200000"

# تحقق من الـ mdio subnode
dtc -I dtb -O dts /sys/firmware/fdt 2>/dev/null | grep -A 30 "mdio"

# تحقق من الـ pinmux (لو MDC/MDIO external)
cat /sys/kernel/debug/pinctrl/pinctrl-handles 2>/dev/null | grep -i mdio
```

**الـ DT المتوقع لـ mdio-regmap:**
```dts
/* مثال من dwmac-socfpga */
&gmac0 {
    compatible = "altr,socfpga-stmmac";
    reg = <0xff700000 0x2000>;

    /* الـ PCS registers في نفس الـ MMIO space */
    /* الـ driver بيعمل regmap على subregion */
};
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للنسخ

```bash
#!/bin/bash
# mdio-regmap full debug script

DEVICE_NAME=$(ls /sys/bus/mdio_bus/devices/ | head -1 | cut -d: -f1)
REGMAP_PATH=$(ls /sys/kernel/debug/regmap/ | grep "$DEVICE_NAME" | head -1)

echo "=== MDIO Bus Devices ==="
ls -la /sys/bus/mdio_bus/devices/

echo ""
echo "=== Regmap Registers (cache_bypass=1) ==="
if [ -n "$REGMAP_PATH" ]; then
    echo 1 > /sys/kernel/debug/regmap/$REGMAP_PATH/cache_bypass
    cat /sys/kernel/debug/regmap/$REGMAP_PATH/registers
    echo 0 > /sys/kernel/debug/regmap/$REGMAP_PATH/cache_bypass
else
    echo "No regmap found for $DEVICE_NAME"
fi

echo ""
echo "=== Ftrace mdio_access (5 seconds) ==="
echo 0 > /sys/kernel/debug/tracing/trace
echo 1 > /sys/kernel/debug/tracing/events/mdio/mdio_access/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
sleep 5
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo 0 > /sys/kernel/debug/tracing/events/mdio/mdio_access/enable
cat /sys/kernel/debug/tracing/trace | grep -v "^#" | tail -50

echo ""
echo "=== Link Status ==="
for iface in $(ls /sys/class/net/ | grep -v lo); do
    echo "$iface: carrier=$(cat /sys/class/net/$iface/carrier 2>/dev/null) operstate=$(cat /sys/class/net/$iface/operstate 2>/dev/null)"
done

echo ""
echo "=== dmesg mdio/regmap errors ==="
dmesg | grep -iE "mdio|regmap|mii_bus|pcs" | tail -30
```

---

#### قراءة MII Registers وتفسيرها

```bash
# قرأ BMSR وفسّره
BMSR=$(cat /sys/kernel/debug/regmap/*/registers | awk '/^01:/ {print "0x"$2}')
BMSR_DEC=$((BMSR))
echo "BMSR = $BMSR"
echo "  bit2  (Link Status)     = $(( (BMSR_DEC >> 2) & 1 ))"
echo "  bit3  (Autoneg Ability) = $(( (BMSR_DEC >> 3) & 1 ))"
echo "  bit5  (Autoneg Complete)= $(( (BMSR_DEC >> 5) & 1 ))"
echo "  bit11 (100BASE-TX FD)   = $(( (BMSR_DEC >> 11) & 1 ))"

# قرأ BMCR وفسّره
BMCR=$(cat /sys/kernel/debug/regmap/*/registers | awk '/^00:/ {print "0x"$2}')
BMCR_DEC=$((BMCR))
echo ""
echo "BMCR = $BMCR"
echo "  bit12 (Autoneg Enable)  = $(( (BMCR_DEC >> 12) & 1 ))"
echo "  bit13 (Speed 100)       = $(( (BMCR_DEC >> 13) & 1 ))"
echo "  bit15 (Soft Reset)      = $(( (BMCR_DEC >> 15) & 1 ))"
```

---

#### تشخيص مشكلة "ENODEV على كل access"

```bash
# المشكلة: كل regmap read/write بيرجع -ENODEV
# السبب: الـ PHY driver بيستخدم addr مختلف عن valid_addr

# خطوة 1: اعرف الـ valid_addr المضبوط
dmesg | grep -E "mdio-regmap|mii_bus|id=|bus.*registered"
# أو
cat /sys/bus/mdio_bus/devices/*/uevent | grep -i addr

# خطوة 2: شغّل ftrace بس لاحظ إن ENODEV مش هيظهر (الـ condition في tracepoint)
# استخدم dynamic debug بدل كده
echo "file drivers/net/mdio/mdio-regmap.c +p" > /sys/kernel/debug/dynamic_debug/control
dmesg -w &
ip link set eth0 down && ip link set eth0 up
# هتشوف الـ pr_debug messages لو اتضيفت

# خطوة 3: تحقق من الـ phy_mask
# لو phy_mask = ~BIT(0) → بس addr=0 مسموح (valid_addr=0)
# لو phy_mask = ~0 → autoscan معطّل، ما فيش scan
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Internal PCS مش بيظهر على MDIO bus — RK3562 Ethernet bring-up

#### العنوان
**الـ PCS الداخلي مش بيتعرف عليه kernel أثناء bring-up على RK3562**

#### السياق
بتشتغل على industrial gateway بيستخدم RK3562 مع Ethernet port واحد. الـ MAC موجود جوه الـ SoC ومتعلق بـ internal PCS بيتوصل من خلال MMIO registers مش عن طريق MDIO bus حقيقي على الـ pins.

#### المشكلة
عند boot، الـ kernel مش بيعمل `phylink` ناجح، وبتلاقي في dmesg:

```
rk3562-gmac: cannot find PHY
```

الـ DT فيه `phy-handle` بيشاور على node عادي، لكن مفيش PHY فعلي على wire.

#### التحليل
**الـ `mdio-regmap.h`** هو الحل هنا. الـ SoC بيعرض الـ PCS كـ registers جوه الـ MMIO address space. المشكلة إن الكود بيحاول يدور على PHY على MDIO bus تقليدي.

الـ struct المهم:

```c
struct mdio_regmap_config {
    struct device *parent;   /* الـ MAC device نفسه */
    struct regmap *regmap;   /* regmap بيشاور على MMIO region بتاع الـ PCS */
    char name[MII_BUS_ID_SIZE]; /* اسم الـ virtual bus */
    u8 valid_addr;           /* عنوان الـ PHY/PCS على الـ virtual bus (0-31) */
    bool autoscan;           /* لو true، بيعمل scan على كل العناوين */
};
```

الـ `devm_mdio_regmap_register()` بتعمل **virtual `mii_bus`** فوق الـ `regmap`. بعدين الـ phylink بيشوف bus عادي ويقدر يتكلم مع الـ PCS زي ما يتكلم مع أي PHY خارجي.

Flow:

```
MAC driver probe()
    └─> regmap_init_mmio(dev, base, &cfg)  ← regmap على MMIO
    └─> devm_mdio_regmap_register(dev, &mdio_cfg)
            └─> mdiobus_alloc()
            └─> bus->read  = mdio_regmap_read()   ← قراءة من regmap
            └─> bus->write = mdio_regmap_write()  ← كتابة في regmap
            └─> mdiobus_register(bus)
    └─> phylink_create(...) ← هيلاقي الـ virtual bus
```

#### الحل
في الـ MAC driver الخاص بـ RK3562، أضف:

```c
static int rk3562_mac_probe(struct platform_device *pdev)
{
    struct regmap *pcs_regmap;
    struct mdio_regmap_config mrc = {};
    struct mii_bus *mii;

    /* map PCS MMIO region via regmap */
    pcs_regmap = devm_regmap_init_mmio(&pdev->dev,
                                       pcs_base,
                                       &rk3562_pcs_regmap_cfg);

    mrc.parent    = &pdev->dev;
    mrc.regmap    = pcs_regmap;
    mrc.valid_addr = 0;          /* PCS على عنوان 0 */
    mrc.autoscan  = false;
    snprintf(mrc.name, MII_BUS_ID_SIZE, "rk3562-pcs-mdio");

    mii = devm_mdio_regmap_register(&pdev->dev, &mrc);
    if (IS_ERR(mii))
        return PTR_ERR(mii);

    /* ... باقي الـ probe */
}
```

في الـ DT:

```dts
&gmac {
    phy-mode = "rgmii";
    /* مش محتاج phy-handle لأن الـ PCS داخلي */
};
```

#### الدرس المستفاد
لما الـ PHY أو PCS يكون **MMIO-mapped** مش على MDIO bus حقيقي، الحل الصح هو `mdio-regmap` مش محاولة محاكاة bus خارجي أو عمل custom read/write functions من الصفر.

---

### السيناريو الثاني: عنوان خاطئ في `valid_addr` — STM32MP1 Ethernet

#### العنوان
**الـ PHY بيتعرف على عنوان غلط وبيديّ link flap على STM32MP1**

#### السياق
بتطور IoT gateway بيستخدم STM32MP1 مع internal RMII PHY. استخدمت `mdio-regmap` صح، لكن الـ link بيقوم ويقع كل 30 ثانية تقريبًا.

#### المشكلة
في الـ `dmesg`:

```
stm32-dwmac: PHY stm32-pcs-mdio:05 - Link is Up - 100/Full
stm32-dwmac: PHY stm32-pcs-mdio:05 - Link is Down
```

العنوان `05` مش صح. الـ PCS على هذا الـ SoC على عنوان `00`.

#### التحليل
الـ field الحاسم هو `valid_addr` في `mdio_regmap_config`:

```c
struct mdio_regmap_config {
    /* ... */
    u8 valid_addr;   /* ← ده بيحدد أي عنوان (0-31) صالح على الـ virtual bus */
    bool autoscan;   /* لو true، الـ bus scan هيلاقي أول device يرد */
};
```

لو `autoscan = true` والـ `valid_addr` مش متحدد صح، الـ MDIO bus scan ممكن يستجوب عناوين غلط وبعض registers بتحتوي على قيم شبيهة بـ PHY ID بالصدفة.

**الـ `mdio_regmap_read()`** في الـ implementation بتعمل:

```c
/* pseudo-code للـ driver implementation */
static int mdio_regmap_read(struct mii_bus *bus, int addr, int regnum)
{
    struct mdio_regmap *r = bus->priv;

    /* لو الـ addr مش valid_addr، return -ENODEV */
    if (addr != r->config.valid_addr)
        return 0xFFFF;

    return regmap_read(r->config.regmap, regnum, &val);
}
```

لو `valid_addr = 5` لكن الـ registers على عنوان `0`، كل read هيرجع `0xFFFF` وهيفسّره الـ phylib كـ PHY غايب أو فيه مشكلة.

#### الحل
```c
/* خطأ */
mrc.valid_addr = 5;
mrc.autoscan   = true;

/* صح */
mrc.valid_addr = 0;   /* من datasheet: STM32MP1 internal PHY على عنوان 0 */
mrc.autoscan   = false;
```

للتحقق:

```bash
# على الـ target
cat /sys/bus/mdio_bus/devices/stm32-pcs-mdio\:00/phy_id
# لازم يرجع قيمة صح مش 0xffffffff
```

#### الدرس المستفاد
**`valid_addr`** مش مجرد hint، ده بيحدد بالظبط أي MDIO address هو الصالح الوحيد على الـ virtual bus. راجع الـ SoC datasheet وحدد العنوان الصح قبل ما تعمل register.

---

### السيناريو الثالث: `devm_` مش بتنظف صح — i.MX8M Plus Android TV Box

#### العنوان
**crash عند rmmod بسبب الاعتقاد الخاطئ إن `devm_` بتعمل كل حاجة**

#### السياق
بتطور Android TV box على i.MX8M Plus. الـ Ethernet driver بيستخدم `mdio-regmap` للـ internal PCS. أثناء testing، بتعمل `rmmod` للـ driver وبتلاقي kernel panic.

#### المشكلة
```
BUG: unable to handle kernel NULL pointer dereference at 0000000000000018
Call trace:
  mdiobus_unregister+0x24/...
  device_release_driver+0x...
```

#### التحليل
الـ signature بتاع الـ function:

```c
struct mii_bus *devm_mdio_regmap_register(struct device *dev,
                                          const struct mdio_regmap_config *config);
```

الكلمة **`devm_`** معناها إن الـ `mii_bus` هيتحرر تلقائي لما الـ `dev` يتعمله detach. لكن المشكلة بتحصل لما:

1. الـ MAC driver بياخد الـ `mii_bus *` ويخزنه في struct خاص بيه.
2. يعمل `phylink_create()` بيستخدم الـ bus ده.
3. عند `remove()`: الـ `devm_` بتحرر الـ `mii_bus` الأول **قبل** ما `phylink_destroy()` يتعمل.

الترتيب الخاطئ:

```
driver remove()
    └─> devm cleanup (automatic) → mdiobus_unregister() → free mii_bus
    └─> phylink_destroy()  ← هنا بيحاول يوصل لـ mii_bus اللي اتحذف!
```

#### الحل
لازم تتأكد إن `phylink_destroy()` بيتعمل **قبل** ما الـ devm resources تتحرر:

```c
static int imx8mp_mac_remove(struct platform_device *pdev)
{
    struct imx8mp_mac *mac = platform_get_drvdata(pdev);

    /* أول حاجة: وقّف الـ phylink */
    phylink_stop(mac->phylink);
    phylink_destroy(mac->phylink);

    /* بعدين: الـ devm بيتكلم بقى على الـ mii_bus */
    /* لو محتاج manual unregister، عملها هنا قبل الـ return */
    return 0;
}
```

أو استخدم `devm_add_action_or_reset()` لضمان ترتيب التنظيف:

```c
static void imx8mp_phylink_destroy(void *data)
{
    struct imx8mp_mac *mac = data;
    phylink_stop(mac->phylink);
    phylink_destroy(mac->phylink);
}

/* في probe(): */
devm_add_action_or_reset(dev, imx8mp_phylink_destroy, mac);
/* ثم بعد كده: */
mii = devm_mdio_regmap_register(dev, &mrc);
```

الـ `devm` بيفكّ الـ resources بترتيب عكسي (LIFO)، فالـ phylink هيتحذف قبل الـ mii_bus.

#### الدرس المستفاد
الـ `devm_` بتضمن التحرير، لكن **الترتيب** مهم. استخدم `devm_add_action_or_reset()` لضبط ترتيب تنظيف الـ resources اللي بتعتمد على بعض.

---

### السيناريو الرابع: `name` collision بين buses — AM62x Multi-Port Switch

#### العنوان
**تعارض في أسماء الـ MDIO buses على AM62x بيخلي الـ sysfs يعمل overwrite**

#### السياق
بتشتغل على industrial gateway بيستخدم TI AM62x مع 3 Ethernet ports كل واحد فيه internal PCS خاص بيه. بتسجّل 3 `mdio-regmap` buses.

#### المشكلة
بعد boot، في `/sys/bus/mdio_bus/devices/` بتلاقي device واحد بس بدل تلاتة. الـ sysfs بيعرض آخر bus اتسجّل فقط:

```bash
ls /sys/bus/mdio_bus/devices/
# am62x-pcs-mdio:00   ← واحد بس!
```

#### التحليل
الـ field:

```c
struct mdio_regmap_config {
    /* ... */
    char name[MII_BUS_ID_SIZE];  /* ← MII_BUS_ID_SIZE = 61 */
    /* ... */
};
```

الـ `devm_mdio_regmap_register()` بتستخدم الـ `name` ده لتسجيل الـ `mii_bus`. الـ `mdiobus_register()` بتستخدم الـ name كـ ID في kernel. لو 3 buses بنفس الاسم، كل registration بتـ overwrite اللي قبلها أو بتفشل بصمت.

الكود الغلط:

```c
/* في loop على 3 ports */
for (i = 0; i < 3; i++) {
    mrc.parent    = &pdev->dev;
    mrc.regmap    = pcs_regmaps[i];
    mrc.valid_addr = 0;
    /* خطأ: نفس الاسم للتلاتة */
    snprintf(mrc.name, MII_BUS_ID_SIZE, "am62x-pcs-mdio");

    buses[i] = devm_mdio_regmap_register(&pdev->dev, &mrc);
}
```

#### الحل
```c
for (i = 0; i < 3; i++) {
    mrc.parent    = &pdev->dev;
    mrc.regmap    = pcs_regmaps[i];
    mrc.valid_addr = 0;
    /* صح: اسم فريد لكل bus */
    snprintf(mrc.name, MII_BUS_ID_SIZE, "am62x-pcs-mdio-%d", i);

    buses[i] = devm_mdio_regmap_register(&pdev->dev, &mrc);
    if (IS_ERR(buses[i]))
        return PTR_ERR(buses[i]);
}
```

للتحقق:

```bash
ls /sys/bus/mdio_bus/devices/
# am62x-pcs-mdio-0:00
# am62x-pcs-mdio-1:00
# am62x-pcs-mdio-2:00
```

#### الدرس المستفاد
`name` في `mdio_regmap_config` لازم يكون **globally unique** على مستوى الـ kernel. في حالة multi-instance، دايمًا ضيف index أو device address في الاسم.

---

### السيناريو الخامس: `autoscan = true` بيعمل false positives — Allwinner H616 Ethernet

#### العنوان
**الـ `autoscan` بيكتشف PHY وهمي على Allwinner H616 وبيعطل الـ network**

#### السياق
بتشتغل على Android TV box بيستخدم Allwinner H616. الـ SoC فيه internal 10/100 PHY على MMIO. استخدمت `autoscan = true` عشان ماتحتاجش تحدد العنوان يدويًا.

#### المشكلة
الـ kernel بيعمل scan ولاقي "PHY" على عناوين متعددة:

```
libphy: am7256-mdio: probed
am7256-mdio:00: attached PHY driver (UID 0x00441400)
am7256-mdio:07: attached PHY driver (UID 0x00441400)  ← مكررة!
```

الـ network بييجي up بس بيخبط في link negotiation conflicts.

#### التحليل
الـ field `autoscan`:

```c
struct mdio_regmap_config {
    /* ... */
    bool autoscan;  /* لو true: mdiobus_register بتعمل scan على عناوين 0-31 */
};
```

لما `autoscan = true`، الـ `mii_bus` بيعمل probe على العناوين من 0 لـ 31. الـ `mdio_regmap_read()` بيرجع قيمة من الـ regmap لكل عنوان. على H616، بعض الـ PCS registers بتعطي قيم غير zero على عناوين متعددة لأن الـ regmap بيعمل **address aliasing** (العنوان بييجي ignored في decode بتاع بعض الـ registers).

نتيجة: الـ phylib بيلاقي "PHY IDs" صالحة على أكتر من عنوان واحد.

الحل الصح هو استخدام `autoscan = false` مع تحديد `valid_addr` بالظبط:

```c
/* خطأ */
mrc.autoscan  = true;
/* valid_addr مش بيفرق هنا لأن autoscan بيتجاهله */

/* صح */
mrc.autoscan   = false;
mrc.valid_addr = 0;    /* من H616 datasheet: internal PHY على عنوان 0 */
```

عند `autoscan = false`، الـ `mdio_regmap_read()` بتـ reject أي عنوان مش بساوي `valid_addr` وبترجع error، فالـ bus scan مش هيلاقي غير device واحد.

للتحقق بعد الإصلاح:

```bash
# على الـ target
cat /sys/bus/mdio_bus/devices/h616-pcs-mdio\:00/phy_id
# لازم ييجي PHY ID صح مرة واحدة بس
ethtool eth0 | grep "Link detected"
# Link detected: yes
```

إذا لم تكن متأكد من العنوان:

```bash
# قبل ما تستخدم autoscan، اعمل manual scan بأمان:
for i in $(seq 0 31); do
    val=$(devmem2 $((PCS_BASE + i * 4)) w 2>/dev/null | grep "Read" | awk '{print $NF}')
    echo "addr $i = $val"
done
# وشوف أي عنوان بيرجع PHY ID منطقي (مش 0x0000 ومش 0xFFFF)
```

#### الدرس المستفاد
**`autoscan = true`** خطر على MMIO-mapped buses لأن الـ regmap مش بيشتكي من أي عنوان. دايمًا حدد `valid_addr` صح و`autoscan = false` في production code. الـ `autoscan` مفيد بس في debugging أو لما الـ PHY address فعلًا dynamic.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي بتغطي الـ mdio-regmap subsystem بشكل مباشر:

| المقالة | الأهمية |
|---------|---------|
| [Introduce a generic regmap-based MDIO driver](https://lwn.net/Articles/927209/) | RFC patch series بتاع Maxime Chevallier اللي أدخل الـ `mdio-regmap` للـ kernel — نقطة البداية الأساسية |
| [net: add a regmap-based mdio driver and drop TSE PCS](https://lwn.net/Articles/933508/) | الـ final merge series في Linux 6.5، بيوضح كيف تم التخلص من TSE PCS وتوحيد الـ driver |
| [regmap: Generic I2C and SPI register map library](https://lwn.net/Articles/451789/) | مقالة Mark Brown الأصلية عن الـ regmap subsystem — أساس مهم لفهم ليه الـ mdio-regmap اتبنى عليه |
| [net: mdio: Add netlink interface](https://lwn.net/Articles/925483/) | بيغطي تطوير الـ MDIO bus infrastructure بشكل عام |
| [RTL8231 GPIO expander support](https://lwn.net/Articles/859520/) | مثال عملي لـ device بيستخدم MDIO regmap provider |
| [regmap: mmio: Extending to support IO ports](https://lwn.net/Articles/904225/) | توسيع الـ regmap-mmio اللي الـ mdio-regmap بيعتمد عليه |

---

### الـ Kernel Documentation الرسمية

```
Documentation/networking/phy.rst          # PHY و MDIO bus framework
Documentation/driver-api/regmap.rst       # regmap API كامل مع أمثلة
Documentation/devicetree/bindings/net/   # DT bindings للـ MDIO buses
```

الـ `mdio-regmap` نفسه محتاج تقرا:

- `drivers/net/mdio/mdio-regmap.c` — الـ implementation
- `drivers/base/regmap/regmap-mdio.c` — طبقة الـ regmap-mdio اللي بتترجم MDIO access لـ regmap calls
- `include/linux/mdio/mdio-regmap.h` — الـ public API (الملف اللي بندرسه)

---

### Kernel Commits المهمة

**الـ RFC الأصلي (مارس 2023):**

> [RFC 0/7] Introduce a generic regmap-based MDIO driver
> <https://lore.kernel.org/all/20230324093644.464704-1-maxime.chevallier@bootlin.com/>

السيريز دي بتتضمن:
- إضافة `reg_shift` لـ regmap عشان يعمل address upshift (ضروري لأن MDIO stride = 1، لكن MMIO محتاج stride أكبر)
- إضافة `drivers/net/mdio/mdio-regmap.c`
- إضافة `include/linux/mdio/mdio-regmap.h`

**الـ regmap-mdio tag على kernel.googlesource:**

> <https://kernel.googlesource.com/pub/scm/linux/kernel/git/broonie/regmap/+/refs/tags/regmap-mdio>

---

### مناقشات الـ Mailing List

| الرابط | الموضوع |
|--------|---------|
| [lore.kernel.org — RFC series](https://lore.kernel.org/all/20230324093644.464704-1-maxime.chevallier@bootlin.com/) | النقاش الكامل على الـ RFC مع review من المشرفين |
| [patchew.org — reg_shift patch](https://patchew.org/linux/20230407152604.105467-1-maxime.chevallier@bootlin.com/) | patch الـ `reg_shift` في regmap اللي الـ mdio-regmap بيحتاجه |

---

### KernelNewbies

**الـ Linux 6.5 release notes** — بتذكر إضافة الـ regmap-based MDIO driver صراحةً:

> <https://kernelnewbies.org/Linux_6.5>

ابحث فيها عن قسم **Networking** أو **Drivers**.

---

### eLinux.org

مفيش صفحة مخصصة للـ mdio-regmap على eLinux، لكن الـ embedded networking references الموجودة:

> <https://elinux.org/Main_Page> → قسم **Networking**

الـ elinux.org مفيد أكتر للـ PHY/MDIO hardware-level documentation في context الـ embedded boards زي BeagleBone.

---

### Collabora Blog — Regmap Tutorial

مقالة عملية ممتازة بتشرح ازاي تحوّل driver تقليدي لـ regmap-based:

> [Using regmaps to make Linux drivers more generic](https://www.collabora.com/news-and-blog/blog/2020/05/27/using-regmaps-to-make-linux-drivers-more-generic/)

بتغطي:
- تحويل MMIO read/write لـ regmap calls
- regmap fields و register layouts
- نفس الـ pattern اللي بيستخدمه الـ mdio-regmap

---

### كتب موصى بيها

#### Linux Device Drivers (LDD3)
- **الكتاب:** *Linux Device Drivers, 3rd Edition* — Corbet, Rubini, Kroah-Hartman
- **الفصل المهم:** Chapter 17 — Network Drivers (بيغطي الـ MII/PHY layer)
- **متاح مجاناً:** <https://lwn.net/Kernel/LDD3/>

> ملحوظة: LDD3 قديم (2005) ومش بيغطي regmap أو mdio-regmap، لكنه أساس للـ network driver model.

#### Linux Kernel Development (Robert Love)
- **الطبعة:** الثالثة (2010)
- **الفصل المهم:** Chapter 17 — Devices and Modules, وChapter 14 — The Block I/O Layer (للـ bus abstraction patterns)
- الكتاب مش بيغطي regmap لأنه قبله، لكنه بيشرح الـ device model اللي الـ `devm_*` APIs بتبنى عليه

#### Mastering Linux Device Driver Development (Packt)
- **الكتاب:** *Mastering Linux Device Driver Development* — John Madieu
- **الفصل المهم:** Chapter 2 — Leveraging the Regmap API
  - <https://subscription.packtpub.com/book/iot-and-hardware/9781789342048/3/ch03lvl1sec10/introduction-to-regmap-and-its-data-structures-i2c-spi-and-mmio>
- بيغطي الـ regmap بشكل حديث ومناسب جداً لفهم الـ mdio-regmap

#### Linux Device Driver Development (Packt, 2nd Edition)
- **الفصل المهم:** Chapter 12 — Abstracting Memory Access: Introduction to the Regmap API
  - <https://subscription.packtpub.com/book/iot-and-hardware/9781803240060/15/ch15lvl1sec72/regmap-based-spi-driver-example-putting-it-all-together>

#### Embedded Linux Primer (Christopher Hallinan)
- **الطبعة:** الثانية
- **الفصل المهم:** Chapter 14 — Networking — بيغطي الـ PHY/MDIO في context الـ embedded boards

---

### Source Code References

```bash
# الـ implementation الرئيسي
drivers/net/mdio/mdio-regmap.c

# طبقة الـ regmap-mdio (الجزء الأدنى)
drivers/base/regmap/regmap-mdio.c

# مثال على driver بيستخدم mdio-regmap
drivers/net/pcs/pcs-lynx.c         # PCS Lynx driver
drivers/net/ethernet/altera/        # Altera TSE بعد migration

# الـ MDIO bus core
drivers/net/mdio/mdio-bus.c

# الـ regmap MMIO backend
drivers/base/regmap/regmap-mmio.c
```

---

### كودبراوزر — للاستكشاف Online

الـ source code كامل متاح online:

> [regmap-mdio.c على codebrowser.dev](https://codebrowser.dev/linux/linux/drivers/base/regmap/regmap-mdio.c.html)

---

### Search Terms للبحث عن مزيد من المعلومات

```
"mdio-regmap" site:lore.kernel.org
"devm_mdio_regmap_register" linux kernel
"mdio_regmap_config" driver example
"regmap mdio stride" linux
"MMIO mapped PHY linux kernel"
"PCS regmap phylink linux"
linux kernel "MII_BUS_ID_SIZE" mdio
"mdio_bus_alloc" regmap
```
## Phase 8: Writing simple module

### الفكرة

**الـ** `devm_mdio_regmap_register` هي الدالة الوحيدة المُصدَّرة في `mdio-regmap.h` — بتسجّل MDIO bus مبنية على regmap بدل من hardware MDIO controller حقيقي. هنعمل **kprobe** عليها عشان نشوف كل مرة بيتسجّل فيها MDIO bus جديد: مين الـ device اللي طلب، وإيه اسم الـ bus، وإيه الـ valid address المسموح بيها.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on devm_mdio_regmap_register
 * Logs every MDIO-regmap bus registration attempt
 */

#include <linux/module.h>       /* MODULE_* macros, module_init/exit */
#include <linux/kprobes.h>      /* kprobe API */
#include <linux/device.h>       /* struct device, dev_name() */
#include <linux/mdio/mdio-regmap.h> /* struct mdio_regmap_config */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Arabic Kernel Docs");
MODULE_DESCRIPTION("kprobe demo: trace devm_mdio_regmap_register calls");

/* ------------------------------------------------------------------ */
/* pre_handler: called just before devm_mdio_regmap_register executes  */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * x86_64 calling convention:
     *   rdi = 1st arg -> struct device *dev
     *   rsi = 2nd arg -> const struct mdio_regmap_config *config
     */
    struct device            *dev    = (struct device *)regs->di;
    struct mdio_regmap_config *cfg   = (struct mdio_regmap_config *)regs->si;

    if (!dev || !cfg)
        return 0;

    pr_info("mdio_regmap_probe: registering MDIO bus\n"
            "  parent device : %s\n"
            "  bus name      : %s\n"
            "  valid_addr    : 0x%02x\n"
            "  autoscan      : %s\n",
            dev_name(dev),
            cfg->name[0] ? cfg->name : "(empty)",
            cfg->valid_addr,
            cfg->autoscan ? "yes" : "no");

    return 0; /* 0 = let the real function run */
}

/* ------------------------------------------------------------------ */
/* kprobe struct                                                        */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "devm_mdio_regmap_register",
    .pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ */
/* module init                                                          */
/* ------------------------------------------------------------------ */
static int __init mdio_probe_init(void)
{
    int ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("mdio_regmap_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }
    pr_info("mdio_regmap_probe: kprobe planted at %ps\n", kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module exit                                                          */
/* ------------------------------------------------------------------ */
static void __exit mdio_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("mdio_regmap_probe: kprobe removed\n");
}

module_init(mdio_probe_init);
module_exit(mdio_probe_exit);
```

---

### Makefile

```makefile
obj-m += mdio_regmap_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|---|---|
| `linux/module.h` | لازم لكل module — بيوفر `module_init`, `module_exit`, `MODULE_*` |
| `linux/kprobes.h` | بيعرّف `struct kprobe` وكل API الـ probing |
| `linux/device.h` | عشان `struct device` و `dev_name()` اللي بنستخدمهم في الـ pre_handler |
| `linux/mdio/mdio-regmap.h` | عشان نقدر نـ cast الـ argument لـ `struct mdio_regmap_config` بشكل صح |

#### الـ `handler_pre`

الـ `pt_regs` بيحتوي على قيم الـ registers وقت الـ probe — على x86_64 الـ `rdi` هو أول argument والـ `rsi` هو تاني argument، فبنـ cast منهم مباشرة. الـ `pr_info` بيطبع اسم الـ parent device واسم الـ bus والـ `valid_addr` اللي بتحدد الـ PHY address المقبولة، وكمان `autoscan` اللي لو `true` بيعمل scan على كل الـ addresses.

#### الـ `struct kprobe`

بنحدد `symbol_name` بدل من عنوان hardcoded عشان الـ kernel يحل العنوان تلقائياً وقت `register_kprobe`. الـ `pre_handler` بيشتغل قبل ما الدالة تتنفذ، وده مناسب هنا لأننا بس عايزين نقرأ الـ arguments.

#### الـ `module_init`

**الـ** `register_kprobe` بيزرع الـ breakpoint في الـ kernel text وبيربط الـ handler بيه — لو فشل (مثلاً الدالة مش موجودة أو `CONFIG_KPROBES` مش enabled) بنرجّع الـ error ومبنكملش.

#### الـ `module_exit`

**الـ** `unregister_kprobe` ضروري في الـ exit عشان يشيل الـ breakpoint من الـ kernel text ويحرر أي resources — لو مشيناش الـ module من غير ما نعمل unregister هيبقى في الـ kernel كود مكسور في مكان ما.

---

### تشغيل وتجربة

```bash
# build
make

# load
sudo insmod mdio_regmap_probe.ko

# trigger: load any driver that calls devm_mdio_regmap_register
# مثلاً على SoC فيه internal PHY على regmap زي stmmac أو xgbe
sudo modprobe some_regmap_mdio_driver

# قرا اللوق
sudo dmesg | grep mdio_regmap_probe

# unload
sudo rmmod mdio_regmap_probe
```

**مثال على output متوقع:**

```
[  42.123456] mdio_regmap_probe: kprobe planted at devm_mdio_regmap_register+0x0
[  43.001234] mdio_regmap_probe: registering MDIO bus
                 parent device : f0000000.eth
                 bus name      : eth-mdio
                 valid_addr    : 0x01
                 autoscan      : no
```
