## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

**الـ `linkmode.h`** جزء من **Ethernet PHY Library** في الـ kernel — اللي بتتعامل مع الـ Physical Layer (PHY) chips اللي موجودة في كل كارت شبكة.

الـ maintainers موجودين في `MAINTAINERS` تحت:
```
ETHERNET PHY LIBRARY
F: include/linux/linkmode.h
F: drivers/net/phy/
F: include/linux/phy.h
```

---

### القصة: إزاي اتنين يتكلموا مع بعض؟

تخيل عندك تليفونين قديمين وعايزهم يتكلموا. كل واحد فيهم عنده قائمة بالإمكانيات اللي يعرف يشتغل بيها:

- التليفون الأول: "أنا بشتغل بـ 1000 Mbps أو 100 Mbps، وبفهم pause frames"
- التليفون التاني: "أنا بشتغل بـ 1000 Mbps أو 10 Mbps بس"

الاتنين لازم يتفقوا على: "هنشتغل بـ 1000 Mbps" — ده اللي بيتسمى **Auto-Negotiation**.

في عالم الشبكات، كل **NIC (Network Interface Card)** عندها **PHY chip** صغيرة بتعمل نفس الكلام — بتعلن عن إمكانياتها، بتسمع إعلانات الطرف التاني، وبتتفق على أحسن سرعة وإعدادات مشتركة.

السؤال: **إزاي الـ kernel يخزن إمكانيات كل جهاز دي؟**

---

### المشكلة اللي بتحلها الـ `linkmode.h`

الـ kernel محتاج يخزن **قائمة ضخمة من الـ capabilities** لكل PHY — زي:
- 10BASE-T Half/Full
- 100BASE-TX
- 1000BASE-T
- 10GBASE-T
- 100GBASE-SR4
- Pause support
- Autoneg support
- وأكتر من 100 capability تانية

أسهل طريقة؟ **Bitmap** — array من الـ bits، كل bit = capability واحدة.

الـ `linkmode` هو اسم الـ type اللي الـ kernel استخدمه لتسمية الـ bitmaps دي. بدل ما تكتب `bitmap_and(...)` مباشرة وتحتاج تتذكر الـ size في كل مرة، بتكتب `linkmode_and(...)` وهو بيعرف الـ size أوتوماتيك.

---

### الهدف من الملف ببساطة

**الـ `include/linux/linkmode.h`** هو مجرد **wrapper layer** فوق الـ bitmap API — بتوفر:

1. **Type safety**: كل العمليات بتاخد `unsigned long *` بس بـ size ثابت (`__ETHTOOL_LINK_MODE_MASK_NBITS`)
2. **Convenience**: بدل ما تكرر الـ size في كل مكان
3. **Semantic clarity**: الكود بقى واضح إنك بتتعامل مع link modes مش بيانات عشوائية

---

### الـ API: بسيطة جداً

```c
/* صفّر كل الـ bits */
linkmode_zero(dst);

/* امسح نسخة */
linkmode_copy(dst, src);

/* AND بين اتنين — بيطلع الـ capabilities المشتركة */
linkmode_and(dst, a, b);

/* OR — اجمع الـ capabilities */
linkmode_or(dst, a, b);

/* هل فيه حاجة مشتركة؟ */
linkmode_intersects(src1, src2);

/* set/clear/test bit واحد */
linkmode_set_bit(ETHTOOL_LINK_MODE_1000baseT_Full_BIT, advertising);
linkmode_test_bit(ETHTOOL_LINK_MODE_Pause_BIT, mask);
```

كلها بتتوسع لـ bitmap operations بـ `__ETHTOOL_LINK_MODE_MASK_NBITS` كـ size.

---

### قصة الـ Pause Negotiation

أعمق حاجة في الملف ده هي الـ **flow control (pause frames)** — وهي قصة لوحدها.

تخيل عندك **router** و**server** متوصلين ببعض:
- الـ server بيبعت بيانات بسرعة جنونية
- الـ router مش قادر يستوعب

الحل: الـ router يبعت **PAUSE frame** للـ server قوله "استنى شوية!"

بس السؤال: مين بيبعت PAUSE لمين؟ ده بيتحدد وقت الـ Auto-Negotiation عن طريق **Pause bit** و**Asym_Pause bit**.

الـ `linkmode.c` فيها دالتين بيحلوا ده:

```c
/* حول tx/rx preferences لـ advertisement bits حسب 802.3 */
void linkmode_set_pause(unsigned long *advertisement, bool tx, bool rx);

/* بعد الـ negotiation، قرر مين يبعت PAUSE ومين يستقبل */
void linkmode_resolve_pause(const unsigned long *local_adv,
                            const unsigned long *partner_adv,
                            bool *tx_pause, bool *rx_pause);
```

الـ logic دي مبنية على **IEEE 802.3 Annex 28B** — جدول معقد بيقول:

```
Local Pause | Local AsymDir | Partner Pause | Partner AsymDir | Result
     0      |      X        |      0        |       X         | Disabled
     0      |      1        |      1        |       0         | Disabled
     0      |      1        |      1        |       1         | TX only
     1      |      0        |      0        |       X         | Disabled
     1      |      X        |      1        |       X         | TX + RX
     1      |      1        |      0        |       1         | RX only
```

---

### ASCII Diagram: مسار الـ Link Mode في الـ Kernel

```
  User Space (ethtool)
        |
        | ioctl / netlink
        v
  +------------------+
  |   ethtool API    |  <-- include/uapi/linux/ethtool.h
  +------------------+
        |
        v
  +------------------+
  |  PHY Core Layer  |  <-- include/linux/phy.h
  |  (phy_device)    |      drivers/net/phy/phy_device.c
  +------------------+
        |
        |  uses linkmode bitmaps:
        |  phydev->supported    (ما يقدر عليه الـ PHY)
        |  phydev->advertising  (ما بيعلن عنه)
        |  phydev->lp_advertising (إعلانات الطرف التاني)
        v
  +------------------+
  |  linkmode API    |  <-- include/linux/linkmode.h
  |  (thin wrapper)  |      drivers/net/phy/linkmode.c
  +------------------+
        |
        v
  +------------------+
  |  bitmap API      |  <-- include/linux/bitmap.h
  +------------------+
```

---

### ليه ده مهم؟

كل مرة تشغل `ethtool eth0` وتشوف:
```
Supported ports: [ TP ]
Supported link modes:   1000baseT/Full
                        100baseT/Full
Advertised link modes:  1000baseT/Full
Link partner advertised link modes: 1000baseT/Full
```

ده كله بيتخزن كـ **linkmode bitmaps** في الـ `phy_device` struct — والـ `linkmode.h` هو الـ API اللي بتتعامل معاه.

---

### الملفات المرتبطة

| الملف | الدور |
|-------|-------|
| `include/linux/linkmode.h` | الـ header الرئيسي — wrapper فوق bitmap |
| `drivers/net/phy/linkmode.c` | implementation الـ pause functions |
| `include/uapi/linux/ethtool.h` | تعريف كل `ETHTOOL_LINK_MODE_*_BIT` enums |
| `include/linux/ethtool.h` | `__ETHTOOL_DECLARE_LINK_MODE_MASK` و structs |
| `include/linux/phy.h` | `phy_device` struct اللي فيه الـ linkmode bitmaps |
| `drivers/net/phy/phy_device.c` | الـ PHY core — بيستخدم linkmode بشكل مكثف |
| `drivers/net/phy/phylink.c` | الـ MAC-PHY glue layer — بيستخدم linkmode_resolve_pause |
| `include/linux/bitmap.h` | الـ bitmap API اللي linkmode بتلفها |
| `drivers/net/phy/` | كل drivers الـ PHY chips المختلفة |
## Phase 2: شرح الـ Linkmode Framework

### المشكلة — ليه الـ Framework ده موجود أصلاً؟

لما بيتكلم network interface مع الـ PHY chip، لازم الاتنين يتفقوا على:
- السرعة: 10M / 100M / 1G / 10G / 100G / ...
- الـ duplex: Half أو Full
- الـ medium: Copper (TP) أو Fibre أو Backplane
- الـ flow control (Pause): عايزين TX pause؟ RX pause؟
- الـ FEC mode: RS-FEC / BASE-R FEC / LLRS-FEC
- الـ Autoneg: بيشتغل أو لأ؟

ده معناه إن كل driver لازم يتكلم عن "مجموعة من الـ capabilities" في وقت واحد — مش قيمة واحدة.

**المشكلة القديمة:** الـ kernel كان بيستخدم `u32` واحد كـ bitmask — 32 bit فقط. ده اشتغل كويس لفترة، لكن لما السرعات اتضاعفت (25G, 40G, 100G, 400G, 1.6T...)، الـ 32 bit خلصت.

**المشكلة التانية:** الـ `u32` القديم ده كان بيتبعت للـ userspace مباشرة — يعني أي تغيير في الـ layout بيكسر الـ ABI.

---

### الحل — الـ Linkmode Bitmap

الـ kernel حل المشكلة بـ:

1. **تعريف `enum ethtool_link_mode_bit_indices`** — كل capability ليها رقم bit ثابت (من 0 لـ 124 حالياً).
2. **استخدام `DECLARE_BITMAP`** بدل الـ `u32` — بيعمل `unsigned long[]` بالحجم الكافي تلقائياً.
3. **الـ `linkmode.h`** يوفر API موحّد للتعامل مع الـ bitmaps دي.

**الـ `__ETHTOOL_LINK_MODE_MASK_NBITS`** هي آخر قيمة في الـ enum — تعمل كـ sentinel بتحدد عدد الـ bits الكلي.

```c
/* على 64-bit system مع ~125 bit:
 * DECLARE_BITMAP(name, 125)
 * => unsigned long name[2]   (2 × 64 = 128 bits, كافي لـ 125)
 */
#define __ETHTOOL_DECLARE_LINK_MODE_MASK(name) \
    DECLARE_BITMAP(name, __ETHTOOL_LINK_MODE_MASK_NBITS)
```

---

### الـ Big Picture — فين الـ Linkmode في الـ Kernel؟

```
┌──────────────────────────────────────────────────────────────────┐
│                        Userspace                                 │
│         ethtool -s eth0 speed 1000 duplex full autoneg on        │
└───────────────────────────┬──────────────────────────────────────┘
                            │ ioctl / netlink
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│                     Kernel: ethtool subsystem                    │
│   ethtool_link_ksettings  {                                      │
│       link_modes.supported      ← ما الـ HW يقدر يعمله         │
│       link_modes.advertising    ← إيه اللي بنعلن عنه في Autoneg │
│       link_modes.lp_advertising ← إيه اللي الـ partner عاوزه   │
│   }                                                              │
│   ← كل field فيها __ETHTOOL_DECLARE_LINK_MODE_MASK              │
└────────────┬──────────────────────┬──────────────────────────────┘
             │                      │
             ▼                      ▼
┌────────────────────┐   ┌──────────────────────────────────────┐
│   phylink layer    │   │         phy_device struct             │
│  (MAC ↔ PHY glue) │   │  supported[]    ← bitmap              │
│  state->advertising│   │  advertising[]  ← bitmap              │
│  state->lp_adv...  │   │  lp_advertising[] ← bitmap            │
└────────┬───────────┘   └────────────────┬─────────────────────┘
         │                                │
         ▼                                ▼
┌────────────────────────────────────────────────────────────────┐
│              linkmode API  (include/linux/linkmode.h)          │
│  linkmode_set_bit / linkmode_test_bit / linkmode_and /         │
│  linkmode_or / linkmode_subset / linkmode_resolve_pause ...    │
└────────────────────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────────────────────┐
│              bitmap API  (include/linux/bitmap.h)              │
│   bitmap_and / bitmap_or / bitmap_zero / bitmap_fill ...       │
│   كلهم بياخدوا  (dst, src, nbits)                             │
└────────────────────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────────────────────┐
│     PHY Drivers (e.g. drivers/net/phy/marvell.c, aquantia.c)  │
│     MAC Drivers (e.g. stmmac, bcmgenet, mvneta)               │
│  → يقرأوا/يكتبوا في الـ linkmode bitmaps عبر الـ API          │
└────────────────────────────────────────────────────────────────┘
```

---

### الـ Real-World Analogy — قائمة العروض في منيو المطعم

تخيل إنك بتفتح مطعم جديد وجارك بيفتح مطعم تاني. الاتنين عايزين يعملوا كومبو لمشترك — الأكل من مطعمك والمشروبات من مطعمه. لازم تتفقوا على إيه المتاح.

**المنيو** = الـ `enum ethtool_link_mode_bit_indices`
- كل item في المنيو ليه رقم ثابت (زي ETHTOOL_LINK_MODE_1000baseT_Full_BIT = 5)
- الأرقام دي ما بتتغيرش أبداً — لو غيّرتها كسرت كل الـ old menus (ABI break)

**القائمة بتاعتك** = الـ `supported` bitmap
- الـ checkbox اللي انت شايله = bit = 1 (عندي الطبق ده)
- الـ checkbox الفاضي = bit = 0 (مش موجود عندي)

**اللي بتعلنه للجار** = الـ `advertising` bitmap
- ممكن تكون عندك طبق لكن مش عاوز تقدمه دلوقتي — فبتشيله من الـ advertising حتى لو موجود في الـ supported

**قائمة الجار** = الـ `lp_advertising` bitmap (lp = link partner)
- بتيجيك عبر الـ Autoneg protocol (802.3 clause 28)

**الاتفاق النهائي** = `linkmode_and(result, advertising, lp_advertising)`
- التقاطع = الطبق اللي الاتنين عارضينه = الـ link mode اللي هيشتغل

**الـ Pause resolution** = بالظبط زي التفاوض على مين بيدفع الفاتورة:
- لو الاتنين قالوا "Pause" → TX + RX pause (كل واحد بيشيل عن التاني)
- لو الاتنين قالوا "Asym_Pause" بس → Disabled (مفيش اتفاق)
- لو أنا قلت "Pause + Asym" والجار قال "Pause فقط" → TX + RX
- لو أنا قلت "Pause + Asym" والجار قال "Asym فقط" → RX only عندي

---

### الـ Core Abstraction — إيه الفكرة الجوهرية؟

**الـ linkmode** هي abstraction طبقة فوق الـ raw bitmap API تقول:

> "عوض ما تتعامل مع `unsigned long[]` مباشرة وتحتاج تعرف عدد الـ bits في كل مكان، استخدم الـ `linkmode_*` functions اللي بتعرف `__ETHTOOL_LINK_MODE_MASK_NBITS` من الداخل."

يعني الـ consumer (driver) مش محتاج يعرف عدد الـ bits — الـ API بتتكفل بده.

```c
/* بدون linkmode — الـ driver لازم يعرف الـ NBITS */
bitmap_and(dst, a, b, __ETHTOOL_LINK_MODE_MASK_NBITS);

/* مع linkmode — أبسط وأوضح */
linkmode_and(dst, a, b);
```

ده زي الفرق بين `strlen(s)` وإنك تعدّ الـ chars بإيدك في كل دالة.

---

### الـ Struct Relationships

```
enum ethtool_link_mode_bit_indices
 ├── ETHTOOL_LINK_MODE_10baseT_Half_BIT   = 0
 ├── ETHTOOL_LINK_MODE_1000baseT_Full_BIT = 5
 ├── ETHTOOL_LINK_MODE_Autoneg_BIT        = 6
 ├── ETHTOOL_LINK_MODE_Pause_BIT          = 13
 ├── ETHTOOL_LINK_MODE_Asym_Pause_BIT     = 14
 ├── ETHTOOL_LINK_MODE_FEC_RS_BIT         = 50
 └── __ETHTOOL_LINK_MODE_MASK_NBITS       = 125  ← sentinel (current)

__ETHTOOL_DECLARE_LINK_MODE_MASK(name)
 └── DECLARE_BITMAP(name, 125)
      └── unsigned long name[2]   ← على 64-bit (2 × 64 = 128 ≥ 125)

struct phy_device {
    ...
    __ETHTOOL_DECLARE_LINK_MODE_MASK(supported);      ← ما الـ PHY يقدر يعمله
    __ETHTOOL_DECLARE_LINK_MODE_MASK(advertising);    ← اللي بنعلنه في Autoneg
    __ETHTOOL_DECLARE_LINK_MODE_MASK(lp_advertising); ← الـ link partner advertised
    __ETHTOOL_DECLARE_LINK_MODE_MASK(adv_old);        ← للـ change detection
    __ETHTOOL_DECLARE_LINK_MODE_MASK(supported_eee);  ← EEE capabilities
    __ETHTOOL_DECLARE_LINK_MODE_MASK(advertising_eee);
    ...
}

struct ethtool_link_ksettings {
    struct ethtool_link_settings base;   ← speed, duplex, port, ...
    struct {
        __ETHTOOL_DECLARE_LINK_MODE_MASK(supported);
        __ETHTOOL_DECLARE_LINK_MODE_MASK(advertising);
        __ETHTOOL_DECLARE_LINK_MODE_MASK(lp_advertising);
    } link_modes;
    u32 lanes;
}
```

---

### الـ linkmode API بالتفصيل

#### 1. الـ Basic Bitmap Operations

كل دالة هي wrapper بسيطة حول الـ `bitmap_*` API، بتمرر `__ETHTOOL_LINK_MODE_MASK_NBITS` تلقائياً:

```c
/* صفّر كل الـ bits */
static inline void linkmode_zero(unsigned long *dst) {
    bitmap_zero(dst, __ETHTOOL_LINK_MODE_MASK_NBITS);
}

/* اعمل AND بين بيتميتين — بتُستخدم لإيجاد التقاطع */
static inline void linkmode_and(unsigned long *dst,
                                const unsigned long *a,
                                const unsigned long *b) {
    bitmap_and(dst, a, b, __ETHTOOL_LINK_MODE_MASK_NBITS);
}

/* اعمل OR — لدمج capabilities من مصادر مختلفة */
static inline void linkmode_or(unsigned long *dst,
                               const unsigned long *a,
                               const unsigned long *b) {
    bitmap_or(dst, a, b, __ETHTOOL_LINK_MODE_MASK_NBITS);
}

/* هل الـ bitmap فاضية؟ — بيُستخدم للـ error checking */
static inline bool linkmode_empty(const unsigned long *src) {
    return bitmap_empty(src, __ETHTOOL_LINK_MODE_MASK_NBITS);
}

/* هل src1 subset من src2؟ — كل bit في src1 موجود في src2 */
static inline int linkmode_subset(const unsigned long *src1,
                                  const unsigned long *src2) {
    return bitmap_subset(src1, src2, __ETHTOOL_LINK_MODE_MASK_NBITS);
}

/* هل فيه تقاطع؟ — مفيد لاختبار compatibility */
static inline int linkmode_intersects(const unsigned long *src1,
                                      const unsigned long *src2) {
    return bitmap_intersects(src1, src2, __ETHTOOL_LINK_MODE_MASK_NBITS);
}
```

#### 2. الـ Bit-Level Operations — Macros مش Functions

```c
/* دي defines مباشرة على الـ bitops العادية — مش inline functions */
#define linkmode_test_bit   test_bit       /* atomic read */
#define linkmode_set_bit    __set_bit      /* non-atomic set */
#define linkmode_clear_bit  __clear_bit    /* non-atomic clear */
#define linkmode_mod_bit    __assign_bit   /* set or clear حسب value */
```

**ملاحظة مهمة:** الـ `__set_bit` و `__clear_bit` مش atomic — في الـ context اللي فيه shared access على الـ bitmap (مثلاً من interrupt context)، لازم تستخدم `set_bit` / `clear_bit` الـ atomic versions.

#### 3. الـ Bulk Set من Array

```c
/* بدل ما تعمل linkmode_set_bit لكل feature على حدة */
static inline void linkmode_set_bit_array(const int *array,
                                          int array_size,
                                          unsigned long *addr) {
    int i;
    for (i = 0; i < array_size; i++)
        linkmode_set_bit(array[i], addr);  /* set كل bit في الـ array */
}

/* مثال استخدام حقيقي من drivers/net/phy/phy_device.c */
static const int phy_basic_ports_array[] = {
    ETHTOOL_LINK_MODE_Autoneg_BIT,
    ETHTOOL_LINK_MODE_TP_BIT,
    ETHTOOL_LINK_MODE_MII_BIT,
};

linkmode_set_bit_array(phy_basic_ports_array,
                       ARRAY_SIZE(phy_basic_ports_array),
                       phy_basic_features);
```

---

### الـ Pause Resolution — أعقد جزء في الـ API

الـ **Pause** في الـ Ethernet هو flow control — لما الـ RX buffer بيتملى، الجهاز بيبعت PAUSE frame لوقف الـ transmitter التاني.

الـ **Asym_Pause** يعني إن الجهاز يقدر يتعامل مع pause في اتجاه واحد بس.

#### `linkmode_set_pause` — إزاي نعلن عن capabilities

```c
void linkmode_set_pause(unsigned long *advertisement, bool tx, bool rx)
{
    /* Pause bit = قادر أستقبل Pause frames */
    linkmode_mod_bit(ETHTOOL_LINK_MODE_Pause_BIT, advertisement, rx);

    /* Asym_Pause bit = فيه asymmetry (TX only أو RX only) */
    linkmode_mod_bit(ETHTOOL_LINK_MODE_Asym_Pause_BIT, advertisement,
                     rx ^ tx);   /* XOR: 1 لما الاتنين مختلفين */
}
```

**جدول التحويل (من 802.3 Annex 28B):**

| tx | rx | Pause bit | Asym_Pause bit | المعنى |
|----|----|-----------|----------------|--------|
| 0  | 0  | 0         | 0              | مفيش pause |
| 0  | 1  | 1         | 1              | RX only (أنا بس بستقبل) |
| 1  | 0  | 0         | 1              | TX only (أنا بس ببعت) |
| 1  | 1  | 1         | 0              | Full pause (TX + RX) |

#### `linkmode_resolve_pause` — إيه اللي اتفقنا عليه فعلاً

بعد الـ Autoneg، كل طرف عنده advertisement الطرف التاني. الدالة دي بتطبق الـ truth table بتاعة 802.3:

```c
void linkmode_resolve_pause(const unsigned long *local_adv,
                            const unsigned long *partner_adv,
                            bool *tx_pause, bool *rx_pause)
{
    __ETHTOOL_DECLARE_LINK_MODE_MASK(m);  /* bitmap مؤقتة على الـ stack */

    linkmode_and(m, local_adv, partner_adv);  /* التقاطع */

    if (linkmode_test_bit(ETHTOOL_LINK_MODE_Pause_BIT, m)) {
        /* الاتنين معلنين Pause → TX+RX */
        *tx_pause = true;
        *rx_pause = true;
    } else if (linkmode_test_bit(ETHTOOL_LINK_MODE_Asym_Pause_BIT, m)) {
        /* في asymmetry متفق عليه → نحتاج نشوف مين طالب إيه */
        *tx_pause = linkmode_test_bit(ETHTOOL_LINK_MODE_Pause_BIT,
                                      partner_adv);  /* الـ partner يقبل RX */
        *rx_pause = linkmode_test_bit(ETHTOOL_LINK_MODE_Pause_BIT,
                                      local_adv);    /* أنا أقبل RX */
    } else {
        *tx_pause = false;
        *rx_pause = false;
    }
}
```

**جدول الـ 802.3 Truth Table:**

```
Local        Partner
Pause AsymDir Pause AsymDir    Result
  0     X       0     X        Disabled
  0     1       1     0        Disabled
  0     1       1     1        TX only
  1     0       0     X        Disabled
  1     X       1     X        TX + RX
  1     1       0     1        RX only
```

---

### الـ Subsystem يمتلك إيه وبيفوّض إيه؟

```
┌─────────────────────────────────────────────────────────────┐
│              ما الـ Linkmode Framework بيمتلكه              │
├─────────────────────────────────────────────────────────────┤
│ • تعريف الـ bit indices (enum في uapi/linux/ethtool.h)      │
│ • الـ NBITS sentinel (آخر قيمة في الـ enum)                 │
│ • الـ __ETHTOOL_DECLARE_LINK_MODE_MASK macro                │
│ • كل الـ linkmode_* inline functions والـ macros            │
│ • منطق الـ Pause resolution (IEEE 802.3)                    │
│ • منطق الـ Pause advertisement encoding                     │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│         ما بيُفوّضه لـ Drivers والـ Subsystems التانية      │
├─────────────────────────────────────────────────────────────┤
│ • تحديد الـ supported bits (كل PHY driver يعرف hardware-ه) │
│ • قراءة الـ Autoneg results من الـ PHY registers (MII/MDIO) │
│ • تحويل الـ MII registers → linkmode bits                   │
│   (مثلاً: mii_adv_to_linkmode_adv_t في mii.h)              │
│ • الـ advertising policy (إيه اللي نعلنه وإيه اللي نخبيه)  │
│ • تطبيق الـ pause على الـ MAC hardware (TX/RX FIFO control) │
│ • الـ phylink layer: تنسيق الـ MAC مع الـ PHY               │
│ • الـ ethtool layer: تبعيت الـ bitmaps للـ userspace         │
└─────────────────────────────────────────────────────────────┘
```

---

### مثال عملي — PHY Driver بيستخدم الـ API

```c
/* مثال مبسط من drivers/net/phy/phy_device.c */

/* 1. PHY driver بيعلن capabilities الـ hardware */
void phy_advertise_supported(struct phy_device *phydev)
{
    /* صفّر أول */
    linkmode_zero(phydev->advertising);

    /* ضيف الـ speeds المدعومة */
    linkmode_set_bit(ETHTOOL_LINK_MODE_1000baseT_Full_BIT,
                     phydev->advertising);
    linkmode_set_bit(ETHTOOL_LINK_MODE_100baseT_Full_BIT,
                     phydev->advertising);

    /* ضيف الـ Pause support */
    linkmode_set_bit(ETHTOOL_LINK_MODE_Pause_BIT,
                     phydev->advertising);
    linkmode_set_bit(ETHTOOL_LINK_MODE_Asym_Pause_BIT,
                     phydev->advertising);
}

/* 2. بعد الـ Autoneg، نحسب الـ pause */
void phy_resolve_link(struct phy_device *phydev)
{
    bool tx_pause, rx_pause;

    /* advertising = اللي أنا عاوزه، lp_advertising = اللي الجار عاوزه */
    linkmode_resolve_pause(phydev->advertising,
                           phydev->lp_advertising,
                           &tx_pause, &rx_pause);

    /* بعدين نطبق النتيجة على الـ MAC hardware */
    mac_set_pause(phydev->attached_dev, tx_pause, rx_pause);
}

/* 3. التحقق من إن ما اتفقنا عليه ضمن الـ supported */
bool phy_validate_advertised(struct phy_device *phydev)
{
    /* advertising لازم يكون subset من supported */
    return linkmode_subset(phydev->advertising, phydev->supported);
}
```

---

### علاقة الـ Linkmode بالـ Subsystems التانية

**الـ ethtool subsystem** — لازم تعرفه: هو الـ interface الرئيسي بين الـ kernel وأدوات الـ userspace زي `ethtool`. بيستخدم `ethtool_link_ksettings` اللي فيها الـ linkmode bitmaps، وبيتكلم مع الـ drivers عبر `get_link_ksettings` / `set_link_ksettings` callbacks.

**الـ phylink subsystem** — لازم تعرفه: طبقة بين الـ MAC driver والـ PHY driver، بتجمع الـ capabilities من الاتنين وبتدير الـ state machine بتاع الـ link. بتستخدم الـ linkmode API بكثافة عشان تحسب التقاطع بين MAC capabilities وPHY capabilities.

**الـ MDIO/MII subsystem** — اللي بيقرأ الـ Autoneg registers من الـ PHY hardware وبيحولها لـ linkmode bits عبر helpers زي `mii_adv_to_linkmode_adv_t`.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. Flags, Enums, وـ Config Options — Cheatsheet

#### الـ `enum ethtool_link_mode_bit_indices` (في `uapi/linux/ethtool.h`)

كل bit في الـ linkmode bitmap يمثل قدرة واحدة للـ link. الـ enum ده هو الأساس اللي كل حاجة في الملف بتتعامل معاها.

| Range | أمثلة على الـ Bits | الفئة |
|-------|-------------------|-------|
| 0–5 | `10baseT_Half`, `100baseT_Half`, `1000baseT_Full` | سرعات Copper الكلاسيكية |
| 6–11 | `Autoneg`, `TP`, `AUI`, `MII`, `FIBRE`, `BNC` | خصائص الـ port |
| 13–14 | `Pause`, `Asym_Pause` | Flow control |
| 49–51, 74 | `FEC_NONE`, `FEC_RS`, `FEC_BASER`, `FEC_LLRS` | Forward Error Correction |
| 52–124 | `50000baseKR_Full` .. `1600000baseDR8_2_Full` | سرعات عالية حديثة |
| آخر entry | `__ETHTOOL_LINK_MODE_MASK_NBITS` | عدد الـ bits الكلي (الآن = 125) |

> **مهم:** الـ `__ETHTOOL_LINK_MODE_MASK_NBITS` مش قيمة ثابتة — بتزيد مع كل kernel version جديد بيضيف speeds جديدة. كل الـ bitmap operations في `linkmode.h` بتستخدمه كـ size.

#### الـ Macros الخاصة بالـ Pause Bits

| Bit | المعنى في الـ IEEE 802.3 |
|-----|--------------------------|
| `Pause_BIT` (13) | الجهاز يدعم استقبال وإرسال Pause frames |
| `Asym_Pause_BIT` (14) | الجهاز يدعم Asymmetric pause (اتجاه واحد فقط) |

#### جدول تحويل tx/rx لـ `linkmode_set_pause`

| tx | rx | Pause bit | Asym_Pause bit | النتيجة الفعلية |
|----|----|-----------|----------------|-----------------|
| 0 | 0 | 0 | 0 | Disabled |
| 0 | 1 | 1 | 1 | RX only (نظريًا) |
| 1 | 0 | 0 | 1 | TX only (نظريًا) |
| 1 | 1 | 1 | 0 | TX + RX |

> **ملاحظة:** التحويل ده فيه مشكلة documented في الكود نفسه — نتيجة الـ negotiation مش دايمًا بتطابق الـ tx/rx اللي اتطلب.

#### جدول نتيجة `linkmode_resolve_pause` (من IEEE 802.3)

| Local Pause | Local AsymDir | Partner Pause | Partner AsymDir | TX | RX |
|-------------|---------------|---------------|-----------------|----|----|
| 0 | X | 0 | X | 0 | 0 |
| 0 | 1 | 1 | 0 | 0 | 0 |
| 0 | 1 | 1 | 1 | 1 | 0 |
| 1 | 0 | 0 | X | 0 | 0 |
| 1 | X | 1 | X | 1 | 1 |
| 1 | 1 | 0 | 1 | 0 | 1 |

---

### 1. الـ Structs المهمة

#### `struct ethtool_link_ksettings` (في `linux/ethtool.h`)

**الغرض:** الـ struct الرئيسي اللي بيحمل كل settings وقدرات الـ network link داخل الـ kernel. ده هو اللي الـ PHY drivers والـ MAC drivers بيملوه وبيقروه.

```c
struct ethtool_link_ksettings {
    struct ethtool_link_settings base;   /* speed, duplex, autoneg, port... */
    struct {
        /* كل bitmap حجمه __ETHTOOL_LINK_MODE_MASK_NBITS bits */
        __ETHTOOL_DECLARE_LINK_MODE_MASK(supported);     /* ما يدعمه الـ HW */
        __ETHTOOL_DECLARE_LINK_MODE_MASK(advertising);   /* ما نُعلن عنه في AN */
        __ETHTOOL_DECLARE_LINK_MODE_MASK(lp_advertising);/* ما أعلنه الطرف الثاني */
    } link_modes;
    u32 lanes; /* عدد الـ lanes للـ high-speed links */
};
```

**الـ Fields:**
| Field | النوع | الوصف |
|-------|-------|-------|
| `base` | `struct ethtool_link_settings` | speed, duplex, port, autoneg, mdio_support |
| `link_modes.supported` | `DECLARE_BITMAP` | الـ bits اللي الـ hardware بيدعمها فعلًا |
| `link_modes.advertising` | `DECLARE_BITMAP` | الـ bits اللي بنعلن عنها في الـ auto-negotiation |
| `link_modes.lp_advertising` | `DECLARE_BITMAP` | الـ bits اللي الـ link partner أعلن عنها |
| `lanes` | `u32` | عدد الـ lanes (مهم لـ 100G/400G/800G) |

**الارتباط:** الـ `link_modes` هو بالضبط اللي الـ `linkmode_*` functions في `linkmode.h` بتتعامل معاه — بتاخد pointers لـ `unsigned long` arrays (الـ bitmaps).

---

#### `struct ethtool_link_settings` (في `uapi/linux/ethtool.h`)

**الغرض:** الـ base struct اللي بيحمل الـ scalar settings للـ link.

```c
struct ethtool_link_settings {
    __u32 cmd;
    __u32 speed;           /* Mbps */
    __u8  duplex;          /* DUPLEX_HALF / DUPLEX_FULL */
    __u8  port;            /* PORT_TP / PORT_AUI / PORT_MII / PORT_FIBRE... */
    __u8  phy_address;     /* MDIO address */
    __u8  autoneg;         /* AUTONEG_DISABLE / AUTONEG_ENABLE */
    __u8  mdio_support;    /* ETH_MDIO_SUPPORTS_C22 / C45 */
    __u8  eth_tp_mdix;     /* MDI-X status */
    __u8  eth_tp_mdix_ctrl;
    __s8  link_mode_masks_nwords; /* عدد الـ 32-bit words لكل bitmap */
    __u8  transceiver;
    __u8  master_slave_cfg;
    __u8  master_slave_state;
    __u8  rate_matching;
    __u32 reserved[7];
    __u32 link_mode_masks[]; /* variable-length — للـ userspace فقط */
};
```

---

#### `struct ethtool_keee` (في `linux/ethtool.h`)

**الغرض:** بيحمل معلومات الـ Energy Efficient Ethernet (EEE). بيستخدم نفس الـ linkmode bitmaps.

```c
struct ethtool_keee {
    __ETHTOOL_DECLARE_LINK_MODE_MASK(supported);   /* speeds تدعم EEE */
    __ETHTOOL_DECLARE_LINK_MODE_MASK(advertised);  /* speeds نُعلن عنها */
    __ETHTOOL_DECLARE_LINK_MODE_MASK(lp_advertised);
    u32  tx_lpi_timer;      /* وقت الانتظار قبل دخول Low Power Idle */
    bool tx_lpi_enabled;
    bool eee_active;        /* EEE شغال حاليًا على الـ link ؟ */
    bool eee_enabled;       /* EEE مفعّل في الـ config ؟ */
};
```

---

### 2. رسم علاقات الـ Structs

```
┌─────────────────────────────────────────────────────────────────┐
│                  ethtool_link_ksettings                         │
│                                                                  │
│  ┌──────────────────────────────────┐                           │
│  │    struct ethtool_link_settings  │ ← base                    │
│  │    (speed, duplex, autoneg, ...)  │                           │
│  └──────────────────────────────────┘                           │
│                                                                  │
│  link_modes:                                                     │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  supported    [unsigned long × N_LONGS]  ← bitmap[125]  │    │
│  │  advertising  [unsigned long × N_LONGS]  ← bitmap[125]  │    │
│  │  lp_advertising[unsigned long × N_LONGS] ← bitmap[125]  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                       ▲                                          │
│                       │                                          │
│           linkmode_*() functions في linkmode.h                   │
│           بتاخد pointer لأي bitmap من الـ 3 دول                 │
└─────────────────────────────────────────────────────────────────┘

enum ethtool_link_mode_bit_indices
┌──────────────────────────────────────┐
│  10baseT_Half_BIT  = 0               │
│  ...                                  │
│  Pause_BIT         = 13   ──────────┐ │
│  Asym_Pause_BIT    = 14   ──────────┼─┼── تستخدمهم
│  ...                                  │ │   linkmode_resolve_pause
│  FEC_RS_BIT        = 50              │ │   linkmode_set_pause
│  ...                                  │ │
│  __ETHTOOL_LINK_MODE_MASK_NBITS=125  │ │
└──────────────────────────────────────┘ │
                                         │
                    ┌────────────────────┘
                    ▼
         ┌─────────────────────┐
         │  linkmode bitmap    │
         │  (unsigned long[])  │
         │  bit 13 = Pause     │
         │  bit 14 = AsymPause │
         └─────────────────────┘
```

---

### 3. Lifecycle Diagrams

#### دورة حياة الـ linkmode bitmap في PHY Driver

```
Boot / probe
    │
    ▼
linkmode_zero(supported)
linkmode_zero(advertising)
    │
    ▼
PHY يكتشف قدراته:
linkmode_set_bit(ETHTOOL_LINK_MODE_1000baseT_Full_BIT, supported)
linkmode_set_bit(ETHTOOL_LINK_MODE_100baseT_Full_BIT,  supported)
linkmode_set_bit(ETHTOOL_LINK_MODE_Autoneg_BIT,        supported)
    │
    ▼
copy supported → advertising (الأول)
linkmode_copy(advertising, supported)
    │
    ▼
User/driver يعدّل advertising عبر ethtool:
linkmode_and(advertising, advertising, requested)
    │
    ▼
Auto-negotiation تبدأ:
(PHY hardware يقرأ advertising ويبعتها للطرف الثاني)
    │
    ▼
AN تخلص:
PHY يملأ lp_advertising من الـ registers
    │
    ▼
تحديد الـ pause:
linkmode_resolve_pause(advertising, lp_advertising, &tx, &rx)
    │
    ▼
Link up — driver يطبق tx_pause / rx_pause على الـ MAC
    │
    ▼
Link down / ifdown:
linkmode_zero(lp_advertising)  ← تنظيف
```

---

### 4. Call Flow Diagrams

#### تدفق `linkmode_resolve_pause`

```
PHY interrupt: link up
    │
    ▼
phy_link_up()  [drivers/net/phy/phy.c]
    │
    ├─► يقرأ lp_advertising من PHY registers
    │   phy_read_lpa() → يملأ phydev->lp_advertising
    │
    ▼
linkmode_resolve_pause(
    phydev->advertising,      ← local_adv
    phydev->lp_advertising,   ← partner_adv
    &tx_pause,
    &rx_pause
)
    │
    │  [داخل linkmode_resolve_pause]
    │
    ├─► linkmode_and(m, local_adv, partner_adv)
    │   /* m = bits مشتركة بين الطرفين */
    │
    ├─► if test_bit(Pause_BIT, m):
    │       tx=true, rx=true   ← كلاهما يدعمان Pause
    │
    ├─► elif test_bit(Asym_Pause_BIT, m):
    │       tx = test_bit(Pause_BIT, partner_adv)
    │       rx = test_bit(Pause_BIT, local_adv)
    │
    └─► else: tx=false, rx=false
    │
    ▼
phy_mac_interrupt() / adjust_link()
    │
    ▼
MAC driver يطبق النتيجة:
mac_dev->set_flow_ctrl(tx_pause, rx_pause)
    │
    ▼
Hardware registers updated
```

#### تدفق `linkmode_set_pause`

```
User runs: ethtool -A eth0 tx on rx on
    │
    ▼
ethtool_set_pauseparam()  [net/ethtool/ioctl.c]
    │
    ▼
driver->set_pauseparam(dev, &epause)
    │
    ▼
linkmode_set_pause(
    advertisement,    ← phydev->advertising
    epause.tx_pause,
    epause.rx_pause
)
    │
    │  [داخل linkmode_set_pause]
    │
    ├─► linkmode_mod_bit(Pause_BIT,      advertisement, rx)
    │   /* Pause bit = rx */
    │
    └─► linkmode_mod_bit(Asym_Pause_BIT, advertisement, rx ^ tx)
        /* AsymDir bit = rx XOR tx */
    │
    ▼
phy_start_aneg()  ← إعادة الـ AN بالـ advertisement الجديد
    │
    ▼
PHY hardware يبعت الـ advertisement الجديد للطرف الثاني
```

#### تدفق العمليات على الـ Bitmap

```
API المستخدم                   bitmap operation داخليًا
─────────────────────────────────────────────────────
linkmode_zero(dst)         →  bitmap_zero(dst, NBITS)
linkmode_fill(dst)         →  bitmap_fill(dst, NBITS)
linkmode_copy(dst, src)    →  bitmap_copy(dst, src, NBITS)
linkmode_and(dst, a, b)    →  bitmap_and(dst, a, b, NBITS)
linkmode_or(dst, a, b)     →  bitmap_or(dst, a, b, NBITS)
linkmode_andnot(d,s1,s2)   →  bitmap_andnot(d, s1, s2, NBITS)
linkmode_empty(src)        →  bitmap_empty(src, NBITS)
linkmode_equal(s1, s2)     →  bitmap_equal(s1, s2, NBITS)
linkmode_intersects(s1,s2) →  bitmap_intersects(s1, s2, NBITS)
linkmode_subset(s1, s2)    →  bitmap_subset(s1, s2, NBITS)
linkmode_set_bit(bit, dst) →  __set_bit(bit, dst)
linkmode_clear_bit(b, dst) →  __clear_bit(bit, dst)
linkmode_test_bit(b, dst)  →  test_bit(bit, dst)
linkmode_mod_bit(b, dst,v) →  __assign_bit(bit, dst, val)
```

---

### 5. Locking Strategy

الملف `linkmode.h` نفسه **مافيهوش أي locks** — وده intentional design decision.

#### ليه مفيش locks في الـ linkmode operations؟

الـ linkmode bitmaps موجودة في structs زي `struct phy_device` و`struct ethtool_link_ksettings`. الـ locking هو مسؤولية الـ caller مش الـ linkmode API نفسه.

#### من يحمي الـ bitmaps؟

| Context | الـ Lock المستخدم | من يحمي |
|---------|------------------|---------|
| PHY driver يعدّل `phydev->advertising` | `phydev->lock` (mutex) | كل الـ phydev fields |
| ethtool ioctl يقرأ/يكتب | `rtnl_lock()` | الـ net_device وكل ما يتعلق بيها |
| interrupt context (AN complete) | `phydev->lock` مأخوذ قبل الـ callback | تحديث lp_advertising |
| SFP/module hotplug | `rtnl_lock()` | تحديث supported modes |

#### الـ Macros المستخدمة في الكود

```c
/* non-atomic — يستخدموا في سياق protected بالـ lock */
#define linkmode_set_bit    __set_bit       /* non-atomic */
#define linkmode_clear_bit  __clear_bit     /* non-atomic */
#define linkmode_mod_bit    __assign_bit    /* non-atomic */

/* atomic — safe بدون lock */
#define linkmode_test_bit   test_bit        /* atomic read — safe */
```

> **مهم:** الـ `__set_bit` و`__clear_bit` مش atomic، يعني لازم الـ caller يحمي الـ bitmap بـ lock لو في concurrent access محتمل. أما `test_bit` فـ atomic read فـ safe تُقرأ بدون lock في بعض السياقات.

#### Lock Ordering (لو محتاج تاخد أكتر من lock)

```
rtnl_lock()          ← outer lock (network device level)
    └─► phydev->lock ← inner lock (PHY level)
```

ممنوع تعكس الترتيب ده — هيعمل deadlock.

---

### ملخص العلاقات النهائية

```
linkmode.h
    │
    ├── يعتمد على ──► bitmap.h  (bitmap_zero, bitmap_copy, ...)
    │
    ├── يعتمد على ──► ethtool.h
    │                    └── ethtool_link_ksettings
    │                             └── link_modes.{supported, advertising, lp_advertising}
    │
    └── يعتمد على ──► uapi/linux/ethtool.h
                         └── enum ethtool_link_mode_bit_indices
                                  ├── Pause_BIT (13)
                                  ├── Asym_Pause_BIT (14)
                                  └── __ETHTOOL_LINK_MODE_MASK_NBITS (125)

الـ linkmode API هو wrapper نظيف فوق bitmap API
بيوحّد الـ size (NBITS) ويخفيه عن كل الـ callers
```
## Phase 4: شرح الـ Functions

### جدول الـ Functions والـ Macros — Cheatsheet

| الاسم | النوع | الغرض |
|---|---|---|
| `linkmode_zero` | `static inline void` | تصفير كل الـ bits في الـ link mode mask |
| `linkmode_fill` | `static inline void` | ضبط كل الـ bits على 1 في الـ link mode mask |
| `linkmode_copy` | `static inline void` | نسخ الـ link mode mask من src إلى dst |
| `linkmode_and` | `static inline void` | AND بين mask-ين وتخزين النتيجة |
| `linkmode_or` | `static inline void` | OR بين mask-ين وتخزين النتيجة |
| `linkmode_empty` | `static inline bool` | فحص إذا كانت الـ mask فاضية (كل الـ bits = 0) |
| `linkmode_andnot` | `static inline bool` | `dst = src1 & ~src2` وترجع true لو النتيجة مش صفر |
| `linkmode_test_bit` | macro → `test_bit` | قراءة bit واحد |
| `linkmode_set_bit` | macro → `__set_bit` | ضبط bit واحد على 1 |
| `linkmode_clear_bit` | macro → `__clear_bit` | مسح bit واحد (ضبطه على 0) |
| `linkmode_mod_bit` | macro → `__assign_bit` | ضبط bit على قيمة معينة (0 أو 1) |
| `linkmode_set_bit_array` | `static inline void` | ضبط مجموعة bits من array |
| `linkmode_equal` | `static inline int` | مقارنة mask-ين هل هما متساويين |
| `linkmode_intersects` | `static inline int` | فحص هل في bit مشترك بين mask-ين |
| `linkmode_subset` | `static inline int` | فحص هل src1 subset من src2 |
| `linkmode_resolve_pause` | `void` (exported) | حساب اتجاه الـ pause flow control بعد الـ autoneg |
| `linkmode_set_pause` | `void` (exported) | ضبط الـ Pause وـ Asym_Pause bits في الـ advertisement |

---

### المفهوم الأساسي: ما هو الـ Link Mode Mask؟

الـ **link mode mask** هو `bitmap` بحجم `__ETHTOOL_LINK_MODE_MASK_NBITS` bits (حاليًا 125 bit). كل bit يمثل **capability** أو **link mode** معين زي `10baseT_Half`، `1000baseT_Full`، `Pause`، `Asym_Pause`، إلخ. الـ PHY layer بيستخدم 3 bitmaps أساسية:

- **`supported`**: اللي الـ hardware يدعمه فعلًا
- **`advertising`**: اللي بنعلن عنه في الـ autonegotiation
- **`lp_advertising`**: اللي الطرف التاني (link partner) أعلن عنه

كل functions في الـ header دي هي wrappers فوق الـ `bitmap_*` API بس بـ fixed size = `__ETHTOOL_LINK_MODE_MASK_NBITS`.

---

### Group 1: Bitmap Initialization & Copy

الغرض: تهيئة ونسخ الـ link mode masks. دي أساسيات الـ lifecycle لأي mask.

---

#### `linkmode_zero`

```c
static inline void linkmode_zero(unsigned long *dst)
{
    bitmap_zero(dst, __ETHTOOL_LINK_MODE_MASK_NBITS);
}
```

بتمسح كل الـ link mode bits في الـ mask وتضبطها على 0. الاستخدام الرئيسي هو تهيئة mask جديدة قبل ما تبدأ تضبط فيها bits. داخليًا بتعمل `memset` لو الـ nbits كبير.

- **Parameters**: `dst` — الـ bitmap المراد تصفيرها، لازم تكون حجمها `DECLARE_BITMAP(x, __ETHTOOL_LINK_MODE_MASK_NBITS)`
- **Return**: void
- **Caller context**: أي context — لا لوك، لا sleep
- **مثال**: `linkmode_zero(phydev->advertising)` قبل ضبط modes جديدة

---

#### `linkmode_fill`

```c
static inline void linkmode_fill(unsigned long *dst)
{
    bitmap_fill(dst, __ETHTOOL_LINK_MODE_MASK_NBITS);
}
```

بتضبط كل الـ bits على 1. مفيدة لو عايز تبدأ بـ "كل شاغل مدعوم" وتشطب منه. الـ `bitmap_fill` بتحترم الـ nbits وما بتضبطش bits تعدّيها.

- **Parameters**: `dst` — الـ bitmap المراد تعبئتها
- **Return**: void

---

#### `linkmode_copy`

```c
static inline void linkmode_copy(unsigned long *dst, const unsigned long *src)
{
    bitmap_copy(dst, src, __ETHTOOL_LINK_MODE_MASK_NBITS);
}
```

بتنسخ mask كاملة من `src` لـ `dst`. الاستخدام الشائع هو نسخ `supported` إلى `advertising` كنقطة بداية للـ autoneg advertisement.

- **Parameters**:
  - `dst` — الوجهة
  - `src` — المصدر (const)
- **Return**: void

---

### Group 2: Bitwise Operations

الغرض: العمليات المنطقية على الـ masks. الأكثر استخداماً في تقاطع الـ capabilities بين الـ local device والـ link partner.

---

#### `linkmode_and`

```c
static inline void linkmode_and(unsigned long *dst, const unsigned long *a,
                                const unsigned long *b)
{
    bitmap_and(dst, a, b, __ETHTOOL_LINK_MODE_MASK_NBITS);
}
```

بتحسب `dst = a & b` على كل الـ bits. النتيجة هي الـ capabilities المشتركة بين طرفين. مثلاً في `linkmode_resolve_pause` بيعملوا AND بين `local_adv` و`partner_adv` عشان يشوفوا إيه المتفق عليه.

- **Parameters**:
  - `dst` — بيتخزن فيها الناتج (ممكن تكون نفس a أو b)
  - `a`, `b` — الـ operands
- **Return**: void (لكن الـ `bitmap_and` الداخلية بترجع bool تدل إن النتيجة مش صفر — الـ wrapper بتتجاهلها)

---

#### `linkmode_or`

```c
static inline void linkmode_or(unsigned long *dst, const unsigned long *a,
                                const unsigned long *b)
{
    bitmap_or(dst, a, b, __ETHTOOL_LINK_MODE_MASK_NBITS);
}
```

بتحسب `dst = a | b`. مفيدة لدمج capabilities من مصادر مختلفة، مثلاً دمج الـ modes المدعومة من firmware مع المدعومة من hardware.

- **Parameters**: نفس `linkmode_and`
- **Return**: void

---

#### `linkmode_andnot`

```c
static inline bool linkmode_andnot(unsigned long *dst,
                                   const unsigned long *src1,
                                   const unsigned long *src2)
{
    return bitmap_andnot(dst, src1, src2, __ETHTOOL_LINK_MODE_MASK_NBITS);
}
```

بتحسب `dst = src1 & ~src2` — يعني بتشيل من src1 أي bits موجودة في src2. مفيدة لاستبعاد modes معينة، مثلاً شيل الـ half-duplex modes من الـ advertising. بترجع `true` لو النتيجة غير صفرية (في على الأقل bit واحد).

- **Parameters**:
  - `dst` — الناتج
  - `src1` — الـ base mask
  - `src2` — الـ mask اللي هتُنفى وتُطرح
- **Return**: `bool` — true لو النتيجة غير صفر

---

### Group 3: Single-Bit Operations (Macros)

الغرض: التعامل مع bit بعينه في الـ mask. دي aliases مباشرة لـ generic bit ops.

---

#### `linkmode_test_bit` → `test_bit`

```c
#define linkmode_test_bit   test_bit
```

بيقرأ قيمة bit معين في الـ mask. **يرجع non-zero لو الـ bit = 1**. آمن من ناحية concurrency على modern architectures لأن `test_bit` بيستخدم atomic read.

- **Parameters**: `(nr, addr)` — رقم الـ bit والـ pointer للـ bitmap
- **مثال**: `linkmode_test_bit(ETHTOOL_LINK_MODE_1000baseT_Full_BIT, phydev->supported)`

---

#### `linkmode_set_bit` → `__set_bit`

```c
#define linkmode_set_bit    __set_bit
```

بيضبط bit على 1. النسخة `__set_bit` هي **non-atomic** وبتفترض ان الكود عنده exclusive access أو في context مش محتاج atomic. لو في تعامل concurrent استخدم `set_bit` بدلها.

- **Parameters**: `(nr, addr)`

---

#### `linkmode_clear_bit` → `__clear_bit`

```c
#define linkmode_clear_bit  __clear_bit
```

بيمسح bit ويضبطه على 0. **non-atomic** زي `linkmode_set_bit`.

---

#### `linkmode_mod_bit` → `__assign_bit`

```c
#define linkmode_mod_bit    __assign_bit
```

بيضبط bit على قيمة معينة (0 أو 1) حسب الـ parameter. بيبسّط الكود اللي لازم يضبط bit حسب condition بدل ما يعمل if/else. **non-atomic**.

- **Parameters**: `(nr, addr, value)` — رقم الـ bit، الـ bitmap، والقيمة bool
- **مثال**: `linkmode_mod_bit(ETHTOOL_LINK_MODE_Pause_BIT, adv, rx)` — بيضبط الـ Pause bit حسب قيمة rx

---

#### `linkmode_set_bit_array`

```c
static inline void linkmode_set_bit_array(const int *array, int array_size,
                                          unsigned long *addr)
{
    int i;
    for (i = 0; i < array_size; i++)
        linkmode_set_bit(array[i], addr);
}
```

بتضبط مجموعة من الـ bits بـ loop بسيطة من array من أرقام الـ bits. مريحة لما يكون عندك list ثابتة من الـ modes اللي الـ driver بيدعمها، بدل ما تعمل `linkmode_set_bit` كل مرة لوحدها.

- **Parameters**:
  - `array` — مصفوفة من الـ bit indices (من الـ enum `ethtool_link_mode_bit_indices`)
  - `array_size` — عدد العناصر في الـ array
  - `addr` — الـ bitmap المراد التعديل فيها
- **Return**: void
- **مثال واقعي**:
```c
static const int phy_basic_ports_array[] = {
    ETHTOOL_LINK_MODE_10baseT_Half_BIT,
    ETHTOOL_LINK_MODE_10baseT_Full_BIT,
    ETHTOOL_LINK_MODE_100baseT_Half_BIT,
};
linkmode_set_bit_array(phy_basic_ports_array,
                       ARRAY_SIZE(phy_basic_ports_array),
                       phydev->supported);
```

---

### Group 4: Comparison & Predicate Functions

الغرض: مقارنة الـ masks واتخاذ قرارات بناءً عليها.

---

#### `linkmode_empty`

```c
static inline bool linkmode_empty(const unsigned long *src)
{
    return bitmap_empty(src, __ETHTOOL_LINK_MODE_MASK_NBITS);
}
```

بتفحص لو الـ mask فاضية تماماً (كل الـ bits = 0). مفيدة بعد عملية AND مثلاً عشان تتحقق هل في تقاطع أو لأ.

- **Parameters**: `src` — الـ mask المراد فحصها
- **Return**: `bool` — true لو كل الـ bits صفر

---

#### `linkmode_equal`

```c
static inline int linkmode_equal(const unsigned long *src1,
                                 const unsigned long *src2)
{
    return bitmap_equal(src1, src2, __ETHTOOL_LINK_MODE_MASK_NBITS);
}
```

بتقارن mask-ين وتشوف هل هما متطابقتين bit-by-bit. بترجع non-zero لو متساويين. الـ `phylink` بيستخدمها عشان يشوف هل في تغيير في الـ capabilities بعد التفاوض.

- **Parameters**: `src1`, `src2` — الـ masks المراد مقارنتها
- **Return**: `int` — non-zero لو متساويين، 0 لو مختلفين

---

#### `linkmode_intersects`

```c
static inline int linkmode_intersects(const unsigned long *src1,
                                      const unsigned long *src2)
{
    return bitmap_intersects(src1, src2, __ETHTOOL_LINK_MODE_MASK_NBITS);
}
```

بتفحص هل في على الأقل bit واحد مشترك (= 1 في الاثنين). بترجع non-zero لو في تقاطع. مفيدة قبل ما تبدأ التفاوض عشان تتأكد ان في modes مشتركة أصلاً.

- **Parameters**: `src1`, `src2`
- **Return**: `int` — non-zero لو في تقاطع

---

#### `linkmode_subset`

```c
static inline int linkmode_subset(const unsigned long *src1,
                                  const unsigned long *src2)
{
    return bitmap_subset(src1, src2, __ETHTOOL_LINK_MODE_MASK_NBITS);
}
```

بتفحص هل كل الـ bits اللي في src1 موجودة كمان في src2. يعني `src1 ⊆ src2`. مفيدة لو عايز تتأكد إن الـ advertising مش بتتجاوز الـ supported capabilities.

- **Parameters**: `src1` (المفروض يكون subset)، `src2` (الـ superset)
- **Return**: `int` — non-zero لو src1 فعلاً subset من src2

---

### Group 5: Pause Flow Control

ده الأهم من الناحية الـ semantic — بيحسم موضوع الـ **pause frames** في الـ Ethernet بعد الـ autonegotiation. الـ pause هو ميكانيزم لـ **flow control** في الـ Ethernet بيسمح للطرف المستقبِل يطلب من المُرسِل يوقف الإرسال مؤقتاً.

#### فهم الـ 802.3 Pause Advertisement

الـ 802.3 بيستخدم bit-ين في الـ autoneg:
- **`Pause`** (bit 13): الـ local device يدعم الاستقبال والإرسال لـ pause frames
- **`Asym_Pause`** (bit 14): الـ local device يدعم الـ asymmetric pause (اتجاه واحد بس)

جدول الـ 802.3 لحل الـ pause بعد الـ autoneg:

```
Local Pause | Local Asym | Partner Pause | Partner Asym | Result
     0      |     X      |      0        |      X       | Disabled
     0      |     1      |      1        |      0       | Disabled
     0      |     1      |      1        |      1       | TX only
     1      |     0      |      0        |      X       | Disabled
     1      |     X      |      1        |      X       | TX + RX
     1      |     1      |      0        |      1       | RX only
```

---

#### `linkmode_resolve_pause`

```c
void linkmode_resolve_pause(const unsigned long *local_adv,
                            const unsigned long *partner_adv,
                            bool *tx_pause, bool *rx_pause);
```

بتحل نتيجة الـ pause negotiation بين طرفين وفق الـ 802.3 spec. بتاخد الـ advertisement masks للطرفين وبترجع boolean لكل اتجاه (TX و RX).

- **Parameters**:
  - `local_adv` — الـ link mode mask للـ local device (advertising)
  - `partner_adv` — الـ link mode mask للـ link partner (lp_advertising)
  - `tx_pause` — output: هل لازم نفعّل إرسال الـ pause frames؟
  - `rx_pause` — output: هل لازم نفعّل استقبال الـ pause frames؟

- **Return**: void — النتيجة في `*tx_pause` و`*rx_pause`

- **Key details**: الـ function مش بتعمل لوك لأنها read-only على الـ bitmaps. الـ caller المسؤول عن الـ locking. متعملتش `EXPORT_SYMBOL` عادية بل `EXPORT_SYMBOL_GPL` — يعني module خارجي لازم يكون GPL.

- **Who calls it**: `phylink_resolve_flow_control()` و`phy_resolve_pause()` في `phy_device.c` بعد انتهاء الـ autoneg.

**Pseudocode Flow**:

```c
void linkmode_resolve_pause(local_adv, partner_adv, tx_pause, rx_pause):
    m = local_adv & partner_adv  // AND للـ masks

    if m has Pause_BIT:
        // كلانا يدعم Pause — TX + RX مفعّلين
        *tx_pause = true
        *rx_pause = true

    else if m has Asym_Pause_BIT:
        // كلانا يدعم Asym — نحسب كل اتجاه منفصل
        *tx_pause = partner_adv has Pause_BIT  // هو يستقبل، نحن نبعت
        *rx_pause = local_adv has Pause_BIT    // نحن نستقبل

    else:
        // ما فيش اتفاق
        *tx_pause = false
        *rx_pause = false
```

**مثال واقعي**: لو `local_adv` فيها `{Pause=1, Asym=1}` و`partner_adv` فيها `{Pause=0, Asym=1}`:
- الـ AND = `{Pause=0, Asym=1}` → دخل الـ `else if`
- `*tx_pause = test(partner_adv, Pause) = 0` → TX disabled
- `*rx_pause = test(local_adv, Pause) = 1` → RX enabled
- النتيجة: **RX only** — نستقبل pause frames بس ما نبعتش

---

#### `linkmode_set_pause`

```c
void linkmode_set_pause(unsigned long *advertisement, bool tx, bool rx);
```

بتترجم طلب المستخدم (tx/rx pause) لـ bits في الـ advertisement mask وفق الـ 802.3 encoding. الـ encoding:

```
tx | rx | Pause bit | Asym_Pause bit
 0 |  0 |     0     |      0
 0 |  1 |     1     |      1
 1 |  0 |     0     |      1
 1 |  1 |     1     |      0
```

الصيغة الرياضية: `Pause = rx`، `Asym_Pause = rx XOR tx`.

- **Parameters**:
  - `advertisement` — الـ mask اللي هيتعدل (عادةً `phydev->advertising`)
  - `tx` — من `ethtool_pauseparam.tx_pause`
  - `rx` — من `ethtool_pauseparam.rx_pause`

- **Return**: void

- **Key details**: في **تحذير مهم** في الكود نفسه — الـ encoding ده مش perfect. مثلاً طلب `tx=1, rx=1` بيضبط `Pause=1, Asym=0`، وده ممكن يأدي لـ disabled pause لو الطرف التاني يدعم `Asym` بس. الـ caller لازم يعرف هذا القيد.

- **Who calls it**: `phylink_ethtool_set_pauseparam()`، `phy_set_asym_pause()`، وـ HNS3 driver.

**Implementation**:

```c
void linkmode_set_pause(unsigned long *advertisement, bool tx, bool rx)
{
    // Pause bit = rx
    linkmode_mod_bit(ETHTOOL_LINK_MODE_Pause_BIT, advertisement, rx);

    // Asym_Pause bit = rx XOR tx
    linkmode_mod_bit(ETHTOOL_LINK_MODE_Asym_Pause_BIT, advertisement, rx ^ tx);
}
```

**مثال واقعي**: المستخدم بيعمل `ethtool -A eth0 tx on rx off`:
- `tx=true, rx=false`
- `Pause = false` → bit 13 = 0
- `Asym_Pause = false ^ true = true` → bit 14 = 1
- الـ partner اللي عنده `Pause=1` بس → AND في Asym = 1، مش Pause → `tx_pause = test(partner, Pause) = 1` → TX enabled

---

### ملاحظات تصميمية مهمة

1. **Fixed size wrappers**: كل الـ inline functions مجرد wrappers على `bitmap_*` API بـ fixed size = `__ETHTOOL_LINK_MODE_MASK_NBITS`. ده بيضمن consistency وبيسهّل التوسع مستقبلاً (لو زادوا modes جديدة، بس الـ NBITS اللي بيتغير).

2. **Non-atomic bit ops**: الـ `linkmode_set_bit`، `linkmode_clear_bit`، `linkmode_mod_bit` كلها non-atomic (`__set_bit` وليس `set_bit`). الـ link mode masks عادةً بتتعدل في contexts مضمونة (مثلاً تحت lock أو في initialization) فالـ non-atomic كافية وأسرع.

3. **GPL Export**: الـ `linkmode_resolve_pause` وـ `linkmode_set_pause` بس هم الـ functions غير inline، ومتعملهمش `EXPORT_SYMBOL` عادية بل `EXPORT_SYMBOL_GPL`، وده يعني الـ PHY و network drivers الـ out-of-tree اللي بتستخدمهم لازم تكون GPL-compatible.

4. **`__ETHTOOL_DECLARE_LINK_MODE_MASK(m)`**: الـ macro ده بيعمل `DECLARE_BITMAP(m, __ETHTOOL_LINK_MODE_MASK_NBITS)` — يعني بيعمل local bitmap على الـ stack. مستخدم في `linkmode_resolve_pause` كـ temporary AND result.
## Phase 5: دليل الـ Debugging الشامل

الـ `linkmode` هو abstraction فوق الـ `bitmap` بيمثّل مجموعة الـ link modes اللي بيدعمها الـ Ethernet interface — زي السرعة، الـ duplex، الـ pause، والـ FEC. الـ debugging هنا بيتمحور حول ethtool subsystem وحالة الـ PHY.

---

### Software Level

#### 1. debugfs entries

الـ `linkmode` نفسه ما لوش debugfs entries مباشرة، لكن الـ PHY layer اللي بيستخدمه بيكشف:

```bash
# mount debugfs لو مش متعمل
mount -t debugfs none /sys/kernel/debug

# PHY debugging — شوف الـ link modes للـ PHY device
ls /sys/kernel/debug/phy/

# مثال: قراءة حالة PHY معين
cat /sys/kernel/debug/phy/phy-0.0/state

# MDIO bus debugging
ls /sys/kernel/debug/mdio-bus/
cat /sys/kernel/debug/mdio-bus/eth0-mii/0
```

الـ output هيبيّن حالة الـ state machine للـ PHY: `UP`, `RUNNING`, `AN_COMPLETE` إلخ.

#### 2. sysfs entries

```bash
# الـ link modes اللي بيدعمها الـ interface
cat /sys/class/net/eth0/speed
cat /sys/class/net/eth0/duplex
cat /sys/class/net/eth0/carrier
cat /sys/class/net/eth0/carrier_changes

# الـ PHY المرتبط بالـ interface
ls -la /sys/class/net/eth0/phydev

# operstate
cat /sys/class/net/eth0/operstate
```

#### 3. ftrace — tracepoints وأحداث مهمة

```bash
# شوف الـ events المتاحة للـ net/ethtool
ls /sys/kernel/debug/tracing/events/net/
ls /sys/kernel/debug/tracing/events/ethtool/ 2>/dev/null

# فعّل tracing للـ link state changes
echo 1 > /sys/kernel/debug/tracing/events/net/net_dev_change_rx_flags/enable
echo 1 > /sys/kernel/debug/tracing/events/net/netdev_up/enable 2>/dev/null

# تتبع الـ phylink state machine
echo 1 > /sys/kernel/debug/tracing/events/net/phylink_mac_change/enable 2>/dev/null

# تشغيل الـ tracer
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo 'linkmode_resolve_pause' > /sys/kernel/debug/tracing/set_graph_function
echo 1 > /sys/kernel/debug/tracing/tracing_on

# قراءة النتيجة
cat /sys/kernel/debug/tracing/trace

# إيقاف
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo nop > /sys/kernel/debug/tracing/current_tracer
```

مثال output مفيد:
```
 0)               |  linkmode_resolve_pause() {
 0)   0.312 us    |    /* tx_pause=1, rx_pause=0 — asymmetric */
 0)   0.890 us    |  }
```

#### 4. printk / dynamic debug

```bash
# فعّل dynamic debug لكل الـ ethtool / phylink subsystem
echo 'module phylink +p' > /sys/kernel/debug/dynamic_debug/control
echo 'module libphy +p'  > /sys/kernel/debug/dynamic_debug/control

# تتبع دوال معينة
echo 'func linkmode_resolve_pause +p' > /sys/kernel/debug/dynamic_debug/control
echo 'func linkmode_set_pause +p'     > /sys/kernel/debug/dynamic_debug/control

# شوف الـ active debug statements
cat /sys/kernel/debug/dynamic_debug/control | grep linkmode

# تابع الـ kernel log
dmesg -w | grep -E 'linkmode|phylink|ethtool|pause'
```

لإضافة printk في الكود:
```c
/* في driver أو phylink code — طباعة الـ link modes */
pr_debug("linkmode supported: %*pb\n",
         __ETHTOOL_LINK_MODE_MASK_NBITS,
         pl->supported);

pr_debug("linkmode advertising: %*pb\n",
         __ETHTOOL_LINK_MODE_MASK_NBITS,
         pl->link_config.advertising);
```

#### 5. Kernel Config options للـ debugging

| Config | الوظيفة |
|--------|---------|
| `CONFIG_PHYLIB` | تفعيل PHY library — أساسي لـ linkmode |
| `CONFIG_PHYLINK` | الـ abstraction layer فوق PHY |
| `CONFIG_DEBUG_FS` | تفعيل debugfs |
| `CONFIG_DYNAMIC_DEBUG` | dynamic debug messages |
| `CONFIG_NET_SELFTESTS` | self-tests للـ network subsystem |
| `CONFIG_ETHTOOL_NETLINK` | ethtool netlink interface |
| `CONFIG_NETDEV_NOTIFIER_ERROR_INJECT` | inject errors لاختبار الـ error paths |
| `CONFIG_NET_DROP_MONITOR` | رصد الـ dropped packets |
| `CONFIG_PROVE_LOCKING` | كشف الـ locking bugs في الـ PHY |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'PHYLIB|PHYLINK|ETHTOOL|NET_SELF'
# أو
grep -E 'PHYLIB|PHYLINK|ETHTOOL' /boot/config-$(uname -r)
```

#### 6. ethtool — الأداة الرئيسية

**الـ ethtool هو الأداة المركزية لـ debugging الـ linkmode.** كل الـ linkmode bitmaps بتظهر من خلاله.

```bash
# عرض كل الـ link modes (supported / advertised / link-partner)
ethtool eth0

# عرض الـ pause settings (مرتبط مباشرة بـ linkmode_resolve_pause)
ethtool -a eth0

# ضبط الـ advertised modes يدوياً
ethtool -s eth0 advertise 0x0000000000ef

# عرض الـ ethtool stats
ethtool -S eth0 | grep -i pause

# ethtool netlink dump — أكثر تفصيلاً
ethtool --json eth0 2>/dev/null

# عرض تفاصيل الـ PHY
ethtool --show-phy eth0 2>/dev/null

# تشغيل self-test
ethtool -t eth0 online
```

مثال output لـ `ethtool eth0`:
```
Settings for eth0:
    Supported ports: [ TP ]
    Supported link modes: 10baseT/Half 10baseT/Full
                          100baseT/Half 100baseT/Full
                          1000baseT/Full
    Supported pause frame use: Symmetric Receive-only
    Supports auto-negotiation: Yes
    Advertised link modes:  1000baseT/Full
    Advertised pause frame use: Symmetric
    Advertised auto-negotiation: Yes
    Link partner advertised pause frame use: Symmetric Receive-only
    Speed: 1000Mb/s
    Duplex: Full
    Auto-negotiation: on
```

تفسير الـ pause:
- `Symmetric` = bit 13 (Pause_BIT) set
- `Receive-only` = bit 13 + bit 14 (Asym_Pause_BIT) set
- `Transmit-only` = bit 14 فقط

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ kernel log | المعنى | الحل |
|---|---|---|
| `phylink: no supported link modes` | الـ linkmode bitmap فاضي — ما في modes مشتركة | تحقق من `ethtool eth0` وتأكد الـ driver بيسجّل modes صح |
| `ETHTOOL_GLINKSETTINGS: bad nwords` | الـ userspace بعت bitmap size غلط | update ethtool binary |
| `phy: device attach failed` | الـ PHY ما اتعرّفش | تحقق من الـ MDIO address والـ PHY ID |
| `phylink: mac support ASYM_PAUSE but phy doesn't` | عدم تطابق الـ pause capabilities | تحقق من الـ PHY datasheet وalign الـ advertising |
| `unable to connect to phy` | مشكلة في MDIO bus | راجع الـ clock للـ MDIO والـ DT |
| `sfp: no support for any link modes` | SFP module ما بيدعمش أي mode مشترك | تحقق من الـ SFP compatibility |

#### 8. WARN_ON و dump_stack — أماكن استراتيجية

```c
/* في الـ driver — تحقق إن الـ supported modes مش فاضية قبل رفع الـ link */
WARN_ON(linkmode_empty(pl->supported));

/* تحقق إن الـ advertised هي subset من الـ supported */
WARN_ON(!linkmode_subset(config->advertising, pl->supported));

/* عند resolve الـ pause — تحقق من consistency */
if (!linkmode_test_bit(ETHTOOL_LINK_MODE_Pause_BIT, local_adv) &&
    !linkmode_test_bit(ETHTOOL_LINK_MODE_Asym_Pause_BIT, local_adv)) {
    /* ما في pause advertising خالص — هل ده متعمد؟ */
    WARN_ONCE(1, "no pause bits advertised on %s\n", dev->name);
}

/* dump_stack عند unexpected link mode negotiation */
if (linkmode_empty(negotiated)) {
    dump_stack();
    return -EINVAL;
}
```

---

### Hardware Level

#### 1. التحقق إن الـ hardware state يطابق الـ kernel state

الـ kernel بيخزّن الـ link modes في الـ bitmaps، لكن الـ PHY chip هو اللي بيعمل الـ negotiation فعلاً عبر الـ MDIO registers.

```bash
# قراءة PHY registers مباشرة (MDIO)
# MII_BMSR = register 1 — Basic Mode Status Register
phytool read eth0/0 MII_BMSR

# MII_ADVERTISE = register 4 — ما بيعلن عنه الـ local PHY
phytool read eth0/0 MII_ADVERTISE

# MII_LPA = register 5 — ما أعلن عنه الـ link partner
phytool read eth0/0 MII_LPA

# MII_CTRL1000 = register 9 — 1G advertisement
phytool read eth0/0 MII_CTRL1000

# مقارنة الـ hardware مع الـ kernel
ethtool eth0 | grep -A5 "Advertised link"
```

قارن الـ ADVERTISE register بيتوافق مع الـ linkmode bitmap المُعلن عنه في `ethtool eth0`.

#### 2. Register Dump

```bash
# قراءة MDIO registers عبر mii-tool
mii-tool -v eth0

# قراءة ذاكرة hardware مباشرة (MAC registers) — بتاج root
# مثال: لو الـ MAC على عنوان 0xFE300000
devmem2 0xFE300000 w      # قراءة 32-bit word
devmem2 0xFE300004 w      # الـ register التالي

# قراءة عبر /dev/mem (أقدم)
dd if=/dev/mem bs=4 count=1 skip=$((0xFE300000/4)) 2>/dev/null | xxd

# ethtool register dump (لو الـ driver يدعم)
ethtool -d eth0 | head -50

# قراءة EEPROM / PHY ID
ethtool -e eth0 | head -20
ethtool -i eth0    # driver + PHY version info
```

#### 3. Logic Analyzer / Oscilloscope

الـ MDIO bus يشتغل على **2.5MHz** (MDC clock). مراقبة الـ negotiation:

```
MDIO Bus Timing:
                 ___   ___   ___
MDC:  ______|   |   |   |   |...  2.5MHz max
MDIO: ----[PRE][ST][OP][PHYAD][REGAD][TA][DATA]----

PRE = 32 bits idle (logic 1)
ST  = 01 (Clause 22) أو 00 (Clause 45)
OP  = 10 (read) أو 01 (write)
```

نقاط المراقبة:
- **MDC**: يجب تكون clean بدون glitches
- **MDIO على الـ preamble**: يجب تكون HIGH لـ 32 clock cycles
- **turnaround bits**: يجب تكون Z (high-impedance) ثم 0

مؤشرات مشكلة hardware:
- MDC frequency أعلى من 2.5MHz → PHY ما بيرد
- MDIO signal مشوّش → مشكلة termination أو كابل طويل
- timeout على الـ read → PHY address غلط أو PHY مش شغّال

#### 4. Hardware Issues → Kernel Log Patterns

| المشكلة الـ hardware | Pattern في الـ kernel log |
|---|---|
| PHY ما بيرد على MDIO | `mdio_bus: MDIO read timeout` |
| Autoneg ما اكتملت | `phylink: autoneg timeout after 10s` |
| SFP module غلط / غير متوافق | `sfp: module is not supported` |
| كابل مش متوصّل صح | `eth0: Link is Down` بعد ثواني من Up |
| مشكلة في الـ 1000BASE-T pair | `phy: detected 1000Mbps but only 100Mbps stable` |
| Power issue على الـ PHY | `phy: lost link unexpectedly` بشكل متكرر |

```bash
# مراقبة مستمرة لـ link events
dmesg -w | grep -E 'eth0|phy|link|autoneg|mdio'

# عدد مرات تغيير الـ carrier
cat /sys/class/net/eth0/carrier_changes
```

#### 5. Device Tree Debugging

```bash
# عرض الـ DT node للـ MAC والـ PHY
cat /proc/device-tree/soc/ethernet@*/compatible 2>/dev/null
ls /proc/device-tree/soc/ethernet@*/

# تحقق من الـ PHY handle
cat /proc/device-tree/soc/ethernet@*/phy-handle 2>/dev/null | xxd

# عرض الـ mdio bus في الـ DT
ls /proc/device-tree/soc/mdio@*/

# تحقق من الـ phy-mode (rgmii, sgmii, etc.)
cat /proc/device-tree/soc/ethernet@*/phy-mode

# مقارنة مع ما يرى الـ kernel
cat /sys/class/net/eth0/phydev/phy_id 2>/dev/null
```

DT مثال صح:
```dts
ethernet0: ethernet@fe300000 {
    compatible = "vendor,mac-v2";
    phy-handle = <&phy0>;
    phy-mode = "rgmii-id";   /* يطابق الـ board design */

    mdio {
        phy0: ethernet-phy@1 {
            reg = <1>;       /* MDIO address = 1 */
            /* max-speed = <1000>; — اختياري للتقييد */
        };
    };
};
```

أخطاء DT شائعة:
- `phy-mode = "rgmii"` بدل `"rgmii-id"` → delay مش صح → packet corruption
- MDIO address غلط في `reg = <...>` → PHY ما بيتلاقاش
- Missing `clocks` property → MDIO clock ما بيشتغلش

---

### Practical Commands

#### جلسة debugging كاملة من البداية

```bash
# ===== الخطوة 1: معلومات أساسية =====
ethtool eth0
ethtool -i eth0
ethtool -a eth0   # pause settings

# ===== الخطوة 2: حالة الـ PHY من الـ kernel =====
cat /sys/class/net/eth0/carrier
cat /sys/class/net/eth0/speed
cat /sys/class/net/eth0/duplex
cat /sys/class/net/eth0/operstate

# ===== الخطوة 3: الـ PHY registers مباشرة =====
# (تأكد phytool مثبّت: apt install phytool)
phytool read eth0/1 MII_BMSR      # status
phytool read eth0/1 MII_ADVERTISE # local advertise
phytool read eth0/1 MII_LPA       # link partner advertise

# ===== الخطوة 4: تفعيل dynamic debug =====
echo 'module phylink +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 'module libphy +pflmt'  > /sys/kernel/debug/dynamic_debug/control
dmesg -c  # clear قديم

# ===== الخطوة 5: trigger link flap وراقب =====
ip link set eth0 down && sleep 2 && ip link set eth0 up
dmesg | tail -50

# ===== الخطوة 6: تحقق من الـ pause negotiation =====
ethtool -a eth0
# Expected output لو كلاهما بيدعم Symmetric:
# Pause parameters for eth0:
# Autonegotiate: on
# RX: on
# TX: on
```

#### فحص الـ linkmode bitmap يدوياً

```bash
# الـ advertised modes كـ hex mask (legacy 32-bit)
ethtool eth0 | grep "Advertised link" -A1

# الـ supported modes بالأرقام — script بسيط
python3 -c "
import subprocess, re
out = subprocess.check_output(['ethtool', 'eth0'], text=True)
for line in out.splitlines():
    if 'Supported link modes' in line or 'Advertised' in line:
        print(line)
"

# تحقق من bit معين — مثال: هل Pause_BIT (13) مضبوط في الـ advertised؟
ethtool eth0 | grep -i "Advertised pause"
```

#### فحص الـ ftrace على link negotiation

```bash
# script متكامل
#!/bin/bash
TRACE=/sys/kernel/debug/tracing

echo 0 > $TRACE/tracing_on
echo function > $TRACE/current_tracer
echo 'linkmode_*' > $TRACE/set_ftrace_filter 2>/dev/null || true
echo 'phylink_*'  >> $TRACE/set_ftrace_filter 2>/dev/null || true
echo 1 > $TRACE/tracing_on

# trigger الـ negotiation
ip link set eth0 down
sleep 1
ip link set eth0 up
sleep 5

echo 0 > $TRACE/tracing_on
cat $TRACE/trace | grep -E 'linkmode|phylink' | head -30
echo nop > $TRACE/current_tracer
```

#### تشخيص مشكلة الـ pause

```bash
# السيناريو: TX pause ما بيشتغلش رغم الـ negotiation

# 1. تحقق إن الـ negotiation حصلت صح
ethtool -a eth0
# إذا TX: off بس الـ hardware يدعم → مشكلة في linkmode_resolve_pause

# 2. فعّل debug وراقب الـ resolve logic
echo 'func linkmode_resolve_pause +p' > /sys/kernel/debug/dynamic_debug/control

# 3. قرأ الـ LPA register مباشرة
phytool read eth0/1 MII_LPA
# Bit 10 = Pause, Bit 11 = Asym_Pause في الـ link partner

# 4. قارن: لو الـ hardware بيقول pause لكن الـ ethtool لا
# → مشكلة في الـ driver — ما بيقرأش LPA صح
# → راجع الـ phy_get_pause() في الـ driver code
```

#### عرض الـ linkmode bits كـ human-readable

```bash
# python script لفك الـ bitmap
python3 << 'EOF'
import subprocess

MODES = {
    0:  "10baseT/Half",    1:  "10baseT/Full",
    2:  "100baseT/Half",   3:  "100baseT/Full",
    4:  "1000baseT/Half",  5:  "1000baseT/Full",
    6:  "Autoneg",         13: "Pause",
    14: "Asym_Pause",      47: "2500baseT/Full",
    48: "5000baseT/Full",  49: "FEC_NONE",
    50: "FEC_RS",          51: "FEC_BASER",
}

# مثال: bitmap كـ hex جاي من driver debug
bitmap_hex = "0x000000000000006f"  # ← ضع قيمتك هنا
val = int(bitmap_hex, 16)

print("Active link mode bits:")
for bit, name in sorted(MODES.items()):
    if val & (1 << bit):
        print(f"  bit {bit:3d}: {name}")
EOF
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — الـ Flow Control بيسبب packet loss عند تحميل عالي

#### العنوان
الـ pause frame negotiation غلط على RGMII بيخلي الـ Ethernet switch يطفح

#### السياق
**الـ board**: Custom industrial gateway مبني على RK3562، متوصل بـ managed gigabit switch عن طريق RGMII. المنتج بيشتغل كـ Modbus/TCP-to-PROFINET gateway في مصنع. الـ link speed: 1000BASE-T.

#### المشكلة
تحت تحميل عالي (حوالي 800 Mbps bidirectional)، الـ switch بيبدأ يرمي packets. الـ `ethtool -S eth0` بيظهر `rx_fifo_errors` بترتفع. الـ switch logs بتقول "port X: pause frames ignored".

#### التحليل
الـ driver بيستخدم `linkmode_set_pause()` عشان يحدد الـ advertisement. المشكلة إن الكود بيعمل:

```c
/* في الـ driver — tx=1, rx=1 */
linkmode_set_pause(advertisement, true, true);
```

الـ `linkmode_set_pause()` لما `tx=1, rx=1`:

```c
void linkmode_set_pause(unsigned long *advertisement, bool tx, bool rx)
{
    /* rx=1  → Pause_BIT = 1 */
    linkmode_mod_bit(ETHTOOL_LINK_MODE_Pause_BIT, advertisement, rx);
    /* rx ^ tx = 1^1 = 0 → Asym_Pause_BIT = 0 */
    linkmode_mod_bit(ETHTOOL_LINK_MODE_Asym_Pause_BIT, advertisement, rx ^ tx);
}
```

النتيجة: advertisement = `Pause=1, Asym=0`.

دلوقتي `linkmode_resolve_pause()` بتشتغل:

```c
linkmode_and(m, local_adv, partner_adv);
/* لو الـ switch بيعمل advertise: Pause=0, Asym=1 */
/* m = AND(local, partner):
   Pause_BIT:      1 & 0 = 0
   Asym_Pause_BIT: 0 & 1 = 0
   → m is empty → tx_pause=false, rx_pause=false */
```

الـ Flow Control اتعطل خالص رغم إن الـ switch كان بيطلبه. الـ switch بيبعت pause frames والـ NIC مش بتسمعهم، فـ buffer بيطفح.

#### الحل
المفروض نفهم احتياج الـ switch الأول، وبعدين نحدد الـ advertisement صح. لو الـ switch بيعمل advertise `Pause=0, Asym=1` (يعني: أنا عايز أستقبل pause بس مش بعت)، المحلي لازم يبعت `Pause=1, Asym=1` (rx=1, tx=0):

```c
/* الـ fix: المحلي يعمل advertise rx-only capability */
linkmode_set_pause(advertisement, false, true);
/* النتيجة:
   Pause_BIT = rx = 1
   Asym_Pause_BIT = rx ^ tx = 1 ^ 0 = 1
*/
```

دلوقتي `linkmode_resolve_pause()`:
```c
/* local:   Pause=1, Asym=1 */
/* partner: Pause=0, Asym=1 */
/* m = AND: Pause=0, Asym=1 */
/* Asym_Pause_BIT set → asymmetric case */
/* tx_pause = test_bit(Pause_BIT, partner_adv) = 0 → false */
/* rx_pause = test_bit(Pause_BIT, local_adv)   = 1 → true  */
```

الـ switch بيبعت pause frames والـ NIC بتسمعهم صح.

```bash
# تحقق بعد الـ fix
ethtool -a eth0
# Expected:
# Autonegotiate: on
# RX: on
# TX: off
```

#### الدرس المستفاد
الـ `linkmode_set_pause()` كودها فيه تعليق صريح "very problematical" — لازم تعرف الـ partner's capabilities قبل ما تحدد `tx/rx`. `tx=1, rx=1` مش دايمًا بيدي Full Duplex flow control.

---

### السيناريو 2: Android TV Box على Allwinner H616 — الـ 100M link بيرفض Autoneg

#### العنوان
الـ `linkmode_andnot()` بيشيل bits غلط من الـ supported mask فـ PHY مش بيعمل negotiate صح

#### السياق
**الـ board**: Android TV box رخيص على Allwinner H616، معاه Realtek RTL8211F PHY متوصل بـ RMII. المنتج المفروض يشتغل على 100BASE-TX. العميل بيشتكي إن أحيانًا الـ link بيقع بعد حوالي ساعة من الشغل.

#### المشكلة
الـ `dmesg` بيظهر:
```
r8169 0000:01:00.0 eth0: Link is Down
r8169 0000:01:00.0 eth0: Link is Up - 10Mbps/Full
```
الـ link بيرجع على 10M بدل 100M وبعدين بيقع.

#### التحليل
الـ PHY driver بيبني الـ `advertising` bitmap من خلال عمليات على الـ `supported` mask. فيه كود في الـ driver بيعمل:

```c
/* remove bits that aren't in supported */
linkmode_andnot(advertising, advertising, supported_not_in_hw);
```

بس فيه bug: `supported_not_in_hw` اتبنى غلط — فيه `ETHTOOL_LINK_MODE_100baseT_Full_BIT` اتضاف فيه بدل ما يتشال:

```c
/* Bug: بدل ما يحذف من supported، حاطه في المحذوفات */
linkmode_set_bit(ETHTOOL_LINK_MODE_100baseT_Full_BIT,
                 supported_not_in_hw);  /* wrong! */

/* النتيجة في linkmode_andnot: */
/* advertising = advertising AND NOT(supported_not_in_hw) */
/* بيشيل 100baseT_Full_BIT من الـ advertising */
```

`linkmode_andnot()` بتشتغل كالتالي:
```c
static inline bool linkmode_andnot(unsigned long *dst,
                                   const unsigned long *src1,
                                   const unsigned long *src2)
{
    return bitmap_andnot(dst, src1, src2,
                         __ETHTOOL_LINK_MODE_MASK_NBITS);
}
/* dst[i] = src1[i] & ~src2[i] */
/* advertising[100M_bit] = 1 & ~1 = 0 → اتشال! */
```

الـ PHY بيعمل autoneg بدون 100M في الـ advertisement، فـ switch بيختار 10M.

#### الحل
```c
/* الصح: ابني supported_not_in_hw صح */
/* احذف فقط الـ bits اللي الـ hardware مش بيدعمها فعلاً */
linkmode_zero(supported_not_in_hw);
/* مثلاً لو H616 مش بيدعم 1G على RMII */
linkmode_set_bit(ETHTOOL_LINK_MODE_1000baseT_Full_BIT,
                 supported_not_in_hw);
linkmode_set_bit(ETHTOOL_LINK_MODE_1000baseT_Half_BIT,
                 supported_not_in_hw);
/* كده 100M هيفضل في الـ advertising */
```

```bash
# للتحقق
ethtool eth0 | grep -A5 "Advertised link modes"
# المفروض تشوف: 100baseT/Full
```

#### الدرس المستفاد
`linkmode_andnot(dst, src1, src2)` بتشيل كل bit موجودة في `src2` من `src1`. أي bit خطأ في `src2` هيتشال من الـ result. دايمًا افحص محتوى الـ mask قبل ما تحطها كـ `src2`.

---

### السيناريو 3: IoT Sensor على STM32MP1 — الـ PHY مش بيعمل negotiate بعد reset

#### العنوان
`linkmode_zero()` بيتنسى قبل `linkmode_set_bit_array()` فـ stale bits بتخلي الـ negotiation يفشل

#### السياق
**الـ board**: Custom IoT sensor على STM32MP157C، معاه LAN8742A PHY، متوصل بـ industrial hub عبر 100BASE-TX. المنتج بيصحى من deep sleep كل 5 دقايق، يبعت sensor data، ويرجع ينام. بعد كذا cycle، الـ link بيبطل يعمل negotiate.

#### المشكلة
بعد ~20 wake/sleep cycle، الـ `ethtool eth0` بيرجع بـ:
```
Link detected: no
```
الـ dmesg:
```
stmmac eth0: PHY ID 0x0007c130 at addr 0
smsc 0.0:01: 1000baseT-Full advertised but not supported
```

#### التحليل
في دالة الـ wake handler، الـ driver بيعمل `linkmode_set_bit_array()` عشان يحدد الـ advertising bits:

```c
static const int stm32_phy_modes[] = {
    ETHTOOL_LINK_MODE_100baseT_Full_BIT,
    ETHTOOL_LINK_MODE_100baseT_Half_BIT,
    ETHTOOL_LINK_MODE_Autoneg_BIT,
    ETHTOOL_LINK_MODE_TP_BIT,
};

/* Bug: نسي linkmode_zero قبل set_bit_array */
/* advertising لسه فيها bits من الـ cycle السابق */
linkmode_set_bit_array(stm32_phy_modes,
                       ARRAY_SIZE(stm32_phy_modes),
                       phydev->advertising);
```

الـ `linkmode_set_bit_array()` فقط بتعمل set ولا بتعمل clear لأي حاجة:

```c
static inline void linkmode_set_bit_array(const int *array, int array_size,
                                          unsigned long *addr)
{
    int i;
    for (i = 0; i < array_size; i++)
        linkmode_set_bit(array[i], addr);  /* set only, no clear */
}
```

لو في cycle سابق اتعمل `linkmode_set_bit(ETHTOOL_LINK_MODE_1000baseT_Full_BIT, ...)` بسبب أي كود تاني، الـ bit هيفضل set. الـ LAN8742A PHY مش بيدعم 1G، فـ autoneg بيفشل.

#### الحل
```c
/* الصح: صفّر الـ mask الأول */
linkmode_zero(phydev->advertising);

linkmode_set_bit_array(stm32_phy_modes,
                       ARRAY_SIZE(stm32_phy_modes),
                       phydev->advertising);
```

أو للتحقق إن الـ advertising مش فيه bits مش supported:

```c
/* تأكد إن advertising هو subset من supported */
if (!linkmode_subset(phydev->advertising, phydev->supported)) {
    dev_warn(dev, "advertising has unsupported bits, fixing\n");
    linkmode_and(phydev->advertising,
                 phydev->advertising,
                 phydev->supported);
}
```

```bash
# Debug على الـ target
cat /sys/class/net/eth0/phydev/advertising
# أو
ethtool eth0
```

#### الدرس المستفاد
`linkmode_set_bit_array()` **additive فقط** — مش بتعمل reset. دايمًا `linkmode_zero()` الأول لو عايز تحدد الـ advertising من الصفر، خصوصًا في wake handlers.

---

### السيناريو 4: Automotive ECU على i.MX8M Plus — الـ 2.5G link مش بيتفاوض مع switch

#### العنوان
`linkmode_intersects()` بيكشف إن الـ local و partner advertisements مفيش بينهم تقاطع

#### السياق
**الـ board**: Automotive ECU على NXP i.MX8M Plus، معاه Marvell 88Q2112 PHY لـ 1000BASE-T1/2500BASE-T1. المنتج automotive gateway بيتوصل بـ managed switch داخل السيارة. المطلوب 2.5G link للـ high-bandwidth camera data.

#### المشكلة
الـ link بيطلع على 1G بدل 2.5G. الـ `ethtool eth0`:
```
Speed: 1000Mb/s
```
بينما الـ switch كونفيج على 2.5G. مفيش errors في الـ dmesg.

#### التحليل
الـ phylink في الـ kernel بيعمل `linkmode_intersects()` عشان يتأكد إن فيه speeds مشتركة قبل الـ negotiation:

```c
/* في phylink_resolve() */
if (!linkmode_intersects(link_state.advertising,
                         pl->link_config.advertising)) {
    /* no common modes, fall back */
}
```

المشكلة إن الـ i.MX8 MAC driver مش بيعمل set لـ `ETHTOOL_LINK_MODE_2500baseT_Full_BIT` في الـ supported mask:

```c
/* في الـ MAC driver — ناقص الـ 2.5G bit */
static const int imx8_mac_caps[] = {
    ETHTOOL_LINK_MODE_1000baseT_Full_BIT,
    ETHTOOL_LINK_MODE_100baseT_Full_BIT,
    /* ETHTOOL_LINK_MODE_2500baseT_Full_BIT — مش موجود! */
};
linkmode_set_bit_array(imx8_mac_caps, ARRAY_SIZE(imx8_mac_caps),
                       config->supported);
```

فـ `linkmode_intersects()` بين الـ PHY advertising (اللي فيه 2.5G) والـ MAC supported (اللي مفيهاش 2.5G) بترجع true بس على الـ 1G bits فقط، والـ negotiation بيختار 1G.

للتأكد من المشكلة:

```c
/* Debug code */
__ETHTOOL_DECLARE_LINK_MODE_MASK(common);
linkmode_and(common, phy_adv, mac_supported);
/* common فيها 1G بس → phylink يختار 1G */

/* لو عملنا linkmode_empty(common) كانت تكشف المشكلة الكاملة */
```

#### الحل
```c
/* أضف 2.5G للـ MAC supported capabilities */
static const int imx8_mac_caps[] = {
    ETHTOOL_LINK_MODE_2500baseT_Full_BIT,  /* added */
    ETHTOOL_LINK_MODE_1000baseT_Full_BIT,
    ETHTOOL_LINK_MODE_100baseT_Full_BIT,
};
```

أو من الـ Device Tree:

```dts
/* imx8mm.dtsi */
&fec1 {
    phy-mode = "rgmii-id";
    /* بعض الـ drivers بيقرأ max-speed */
    max-speed = <2500>;
};
```

```bash
# للتحقق
ethtool eth0 | grep "Supported link modes"
# المفروض تشوف: 2500baseT/Full
ethtool eth0 | grep Speed
# Expected: Speed: 2500Mb/s
```

#### الدرس المستفاد
`linkmode_intersects()` بترجع `true` حتى لو في bit واحد مشترك — مش معناها إن الأعلى speed مشتركة. لازم تتأكد إن الـ MAC supported mask كاملة قبل الـ phylink negotiation.

---

### السيناريو 5: Custom Board Bring-up على AM62x — الـ PHY مش بيتعرف وautoneg معطل

#### العنوان
`linkmode_equal()` بيكشف إن الـ advertised modes اتغيرت بشكل غير متوقع بعد `phy_start_aneg()`

#### السياق
**الـ board**: Custom AM62x (TI Sitara) industrial board، أول bring-up. الـ PHY: DP83867 متوصل بـ RGMII. المنتج PLC controller مع Ethernet للـ SCADA system. الـ engineer بيعمل bring-up لأول مرة.

#### المشكلة
الـ Ethernet الـ link بيطلع بس الـ autoneg مش بيشتغل صح. الـ `ethtool eth0` بيظهر:
```
Advertised link modes: Not reported
Auto-negotiation: off
```
رغم إن الكود بيعمل set للـ Autoneg_BIT.

#### التحليل
الـ engineer كان بيعمل debug بـ `linkmode_equal()` للتحقق من الـ state قبل وبعد `phy_start_aneg()`:

```c
__ETHTOOL_DECLARE_LINK_MODE_MASK(before);
__ETHTOOL_DECLARE_LINK_MODE_MASK(after);

/* save state before */
linkmode_copy(before, phydev->advertising);

phy_start_aneg(phydev);

/* check if advertising changed */
linkmode_copy(after, phydev->advertising);

if (!linkmode_equal(before, after)) {
    dev_warn(dev, "advertising changed during aneg start!\n");
    /* print both for comparison */
}
```

اكتشف إن `phy_start_aneg()` بتعمل `linkmode_and()` داخلياً:

```c
/* في phy_start_aneg → genphy_config_advert */
linkmode_and(phydev->advertising,
             phydev->advertising,
             phydev->supported);
```

المشكلة إن `phydev->supported` كانت فاضية (`linkmode_empty()` = true) لأن الـ PHY probe فشل بـ silent error بسبب MDIO address غلط في الـ DT:

```dts
/* Bug في DT */
&mdio {
    dp83867: ethernet-phy@1 {  /* Wrong! MDIO address should be 0 */
        reg = <1>;
    };
};
```

لما الـ PHY مش بيتلاقى، `supported` بيفضل empty، والـ AND بيخلي `advertising` فاضي كمان. الـ `linkmode_empty()` كانت ممكن تكشف ده:

```c
if (linkmode_empty(phydev->supported)) {
    dev_err(dev, "PHY supported modes are empty — check MDIO address\n");
    return -ENODEV;
}
```

#### الحل
**أولاً**: صلح الـ DT:
```dts
&mdio {
    dp83867: ethernet-phy@0 {  /* Correct MDIO address */
        reg = <0>;
        ti,rx-internal-delay = <DP83867_RGMIIDCTL_2_00_NS>;
        ti,tx-internal-delay = <DP83867_RGMIIDCTL_2_00_NS>;
        ti,fifo-depth = <DP83867_PHYCR_FIFO_DEPTH_4_B_NIB>;
    };
};
```

**ثانياً**: أضف sanity check في الـ driver:
```c
/* بعد phy_attach */
if (linkmode_empty(phydev->supported)) {
    dev_err(dev, "PHY has no supported link modes\n");
    return -EINVAL;
}

/* تأكد Autoneg_BIT موجود */
if (!linkmode_test_bit(ETHTOOL_LINK_MODE_Autoneg_BIT,
                       phydev->supported)) {
    dev_warn(dev, "PHY does not support autoneg\n");
}
```

```bash
# Debug على الـ target
# تحقق من MDIO scan
mdio-tool eth0 scan
# أو
dmesg | grep -i "phy\|mdio"

# بعد الـ fix
ethtool eth0 | grep -E "Auto-negotiation|Advertised"
# Expected:
# Auto-negotiation: on
# Advertised link modes: 1000baseT/Full 100baseT/Full
```

#### الدرس المستفاد
`linkmode_empty()` هي الـ first-line check في أي PHY bring-up — لو الـ `supported` mask فاضية معناه الـ PHY probe فشل بصمت. دايمًا افحصها بعد الـ `phy_attach()` مباشرة قبل ما تكمل الـ initialization.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي بتغطي الـ phylib والـ linkmode وكل اللي حواليهم:

| المقال | الأهمية |
|--------|---------|
| [Phylink & SFP support](https://lwn.net/Articles/667055/) | تقديم الـ phylink كـ abstraction layer فوق الـ phylib، وبيشرح ازاي الـ link modes بتتحدد |
| [PHY Abstraction Layer III](https://lwn.net/Articles/144897/) | تطور الـ phylib في النواة، وكيف اتعمل الـ link mode negotiation |
| [RFC: PHY Abstraction Layer II](https://lwn.net/Articles/127013/) | المقترح الأولاني للـ PHY Abstraction Layer مع دعم ethtool |
| [new ETHTOOL_GLINKSETTINGS/SLINKSETTINGS API](https://lwn.net/Articles/677122/) | الـ API الجديد اللي استبدل الـ ETHTOOL_GSET وفتح الطريق لـ linkmode bitmaps كبيرة |
| [ethtool netlink interface, part 1](https://lwn.net/Articles/783633/) | تقديم الـ netlink interface لـ ethtool، واللي بيستخدم linkmode bitmaps |
| [ethtool netlink interface, part 2](https://lwn.net/Articles/810618/) | تفاصيل تانية عن الـ netlink interface |
| [ethtool netlink interface, part 4](https://lwn.net/Articles/816137/) | بيغطي الـ pause، EEE، وcoalescing — مباشر متعلق بـ `linkmode_set_pause` |
| [Add phylib support for MV88X3310 10G phy](https://lwn.net/Articles/724633/) | مثال عملي على استخدام الـ linkmode مع PHY حقيقي |

---

### توثيق النواة الرسمي

الملفات دي موجودة في شجرة الـ kernel تحت `Documentation/`:

```
Documentation/networking/phy.rst
Documentation/networking/sfp-phylink.rst
Documentation/networking/ethtool-netlink.rst
```

**الـ online versions:**

- [PHY Abstraction Layer — kernel.org](https://www.kernel.org/doc/html/latest/networking/phy.html)
  - بيشرح الـ `phy_device`, الـ `link_mode`, والـ negotiation flow كامل

- [phylink — kernel.org](https://docs.kernel.org/networking/sfp-phylink.html)
  - بيشرح العلاقة بين الـ MAC driver والـ PHY وإزاي الـ linkmode bitmaps بتتبادل

- [Netlink interface for ethtool — kernel.org](https://docs.kernel.org/networking/ethtool-netlink.html)
  - بيشرح إزاي الـ link modes بتتعرض للـ userspace عبر netlink بـ compact أو bit-by-bit format

- [PHY link topology — kernel.org](https://docs.kernel.org/next/networking/phy-link-topology.html)
  - أحدث documentation بيشرح topology الـ PHY link

---

### سورس الكود المهم في النواة

الـ `linkmode.h` مش بيشتغل لوحده — الكود الفعلي للـ non-inline functions موجود هنا:

```
include/linux/linkmode.h          ← الـ header (الملف اللي بندرسه)
drivers/net/phy/linkmode.c        ← implementation لـ linkmode_resolve_pause و linkmode_set_pause
net/ethtool/linkmodes.c           ← الـ ethtool netlink handler للـ link modes
include/uapi/linux/ethtool.h      ← تعريف __ETHTOOL_LINK_MODE_MASK_NBITS والـ enums
include/linux/bitmap.h            ← الـ bitmap primitives اللي الـ linkmode بيبني عليها
```

**الـ GitHub links للكود:**

- [drivers/net/phy/linkmode.c](https://github.com/torvalds/linux/blob/master/drivers/net/phy/linkmode.c)
- [net/ethtool/linkmodes.c](https://github.com/torvalds/linux/blob/master/net/ethtool/linkmodes.c)

---

### Kernel Commits المهمة

الـ commits دي مهمة لفهم تاريخ الـ linkmode:

- **إضافة `linkmode.h` كـ header مستقل** — لما الـ bitmap operations اتفصلت من الـ phylib code وأتحطت في header منفصل
- **`linkmode_resolve_pause` و `linkmode_set_pause`** — اتأضيفوا لما الـ phylib محتاج logic موحد لحل الـ pause frames بناءً على الـ IEEE 802.3 Annex 28B

للبحث في الـ git history:

```bash
# داخل شجرة الـ kernel
git log --oneline -- include/linux/linkmode.h
git log --oneline -- drivers/net/phy/linkmode.c
git log --oneline -- include/uapi/linux/ethtool.h | grep -i "link_mode\|linkmode"
```

---

### نقاشات Mailing List

- [Re: RFC PATCH ethtool: Fix incompatibility between netlink and ioctl interfaces](https://www.mail-archive.com/netdev@vger.kernel.org/msg358528.html)
  - نقاش على netdev@vger.kernel.org عن التوافق بين الـ ioctl interface القديم والـ netlink الجديد فيما يخص الـ link modes

**للبحث في الـ mailing list archive:**

```
https://lore.kernel.org/netdev/?q=linkmode
https://lore.kernel.org/netdev/?q=linkmode_resolve_pause
https://lore.kernel.org/netdev/?q=phylib+linkmode
```

---

### Kernelnewbies

- [Linux 6.9 Changes](https://kernelnewbies.org/Linux_6.9)
  — بيذكر استخدام الـ linkmode bitmaps في الـ EEE netlink interface

- [Linux 6.13 Changes](https://kernelnewbies.org/Linux_6.13)
  — بيذكر تحويل الـ `eee_broken_modes` لـ linkmode bitmap مع accessor function جديدة

- [LinuxChanges](https://kernelnewbies.org/LinuxChanges)
  — الصفحة الرئيسية لمتابعة التغييرات في كل إصدار

---

### eLinux.org

- [Generic PHY Framework — An Overview (PDF)](https://elinux.org/images/1/1c/Abraham--generic_phy_framework_an_overview.pdf)
  — ورقة بحثية بتغطي الـ PHY framework من منظور embedded Linux، مفيدة جداً لفهم الـ context

---

### كتب موصى بيها

#### Linux Device Drivers 3rd Edition (LDD3)

> **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman

- **الفصل 17:** Network Drivers — بيشرح الـ `net_device`, الـ `ethtool_ops`, وكيفية التعامل مع الـ link state
- متاح مجاناً: [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

**ملاحظة:** LDD3 قديمة (2005) وما بتغطيش الـ phylib الحديث أو الـ linkmode bitmaps، بس أساسياتها لا تزال صالحة.

#### Linux Kernel Development — Robert Love (3rd Edition)

- **الفصل 17:** Devices and Modules — خلفية عامة عن الـ driver model
- بيفيد لفهم كيفية دمج الـ PHY drivers في الـ kernel infrastructure

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)

- **الفصل 15:** Embedded Ethernet — بيشرح الـ MII/RMII/SGMII interfaces وعلاقتها بالـ PHY layer
- مهم لفهم ليه الـ link modes موجودة أصلاً في embedded systems

---

### Search Terms للبحث عن معلومات أكثر

```
linux kernel linkmode bitmap
phylib link mode negotiation
ethtool ETHTOOL_GLINKSETTINGS link_mode_mask
__ETHTOOL_LINK_MODE_MASK_NBITS
linkmode_resolve_pause 802.3
phylink MAC driver link modes
linux ethtool netlink linkmodes
phy_resolve_pause kernel
ETHTOOL_A_LINKMODES_OURS netlink
```

**للبحث في Elixir (kernel cross-reference):**

```
https://elixir.bootlin.com/linux/latest/source/include/linux/linkmode.h
https://elixir.bootlin.com/linux/latest/source/drivers/net/phy/linkmode.c
https://elixir.bootlin.com/linux/latest/ident/linkmode_resolve_pause
```
## Phase 8: Writing simple module

### الفكرة

**`linkmode_resolve_pause`** هي الدالة الأنسب للـ hook — بيتم استدعاؤها كل ما الـ PHY driver بيحتاج يحسب إيه الـ pause mode المتفق عليه بين الـ local NIC والـ link partner عن طريق الـ auto-negotiation. الدالة exported بـ `EXPORT_SYMBOL_GPL` وبتاخد بيتات الـ advertisement وبترجع `tx_pause` و `rx_pause`. هنستخدم **kprobe** على entry الدالة عشان نطبع بيانات الـ advertisement قبل ما تتحسب النتيجة.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_linkmode_pause.c
 *
 * Hooks linkmode_resolve_pause() to log pause negotiation
 * advertisements whenever a PHY resolves flow control.
 */

#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe */
#include <linux/kernel.h>      /* pr_info */
#include <linux/ethtool.h>     /* ETHTOOL_LINK_MODE_Pause_BIT, __ETHTOOL_LINK_MODE_MASK_NBITS */
#include <linux/linkmode.h>    /* linkmode_test_bit */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on linkmode_resolve_pause to log pause advertisement bits");

/* -----------------------------------------------------------------------
 * pre_handler — بيتشغل مباشرةً قبل تنفيذ أول instruction في الدالة.
 * الـ regs بتحتوي على قيم الـ registers في لحظة الاستدعاء،
 * فنقدر نقرأ الـ arguments من الـ calling convention (x86-64: rdi, rsi).
 * ----------------------------------------------------------------------- */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * x86-64 calling convention:
     *   rdi = local_adv   (1st arg)
     *   rsi = partner_adv (2nd arg)
     *
     * نقرأ الـ pointers من الـ registers مباشرةً.
     */
    const unsigned long *local_adv   = (const unsigned long *)regs->di;
    const unsigned long *partner_adv = (const unsigned long *)regs->si;

    /* نفحص بيت الـ Pause وبيت الـ Asym_Pause في كل advertisement */
    bool local_pause  = linkmode_test_bit(ETHTOOL_LINK_MODE_Pause_BIT,      local_adv);
    bool local_asym   = linkmode_test_bit(ETHTOOL_LINK_MODE_Asym_Pause_BIT, local_adv);
    bool partner_pause = linkmode_test_bit(ETHTOOL_LINK_MODE_Pause_BIT,     partner_adv);
    bool partner_asym  = linkmode_test_bit(ETHTOOL_LINK_MODE_Asym_Pause_BIT,partner_adv);

    pr_info("kprobe[linkmode_resolve_pause]: "
            "local(Pause=%d AsymDir=%d) partner(Pause=%d AsymDir=%d)\n",
            local_pause, local_asym, partner_pause, partner_asym);

    return 0; /* 0 = تكمل تنفيذ الدالة الأصلية بشكل طبيعي */
}

/* -----------------------------------------------------------------------
 * struct kprobe — بتعرّف نقطة الـ hook.
 * الـ symbol_name بيخلي الـ kernel يعمل resolve للعنوان تلقائياً
 * من الـ symbol table بدل ما نحط عنوان hardcoded.
 * ----------------------------------------------------------------------- */
static struct kprobe kp = {
    .symbol_name = "linkmode_resolve_pause",
    .pre_handler = handler_pre,
};

/* -----------------------------------------------------------------------
 * module_init — بيسجّل الـ kprobe عند تحميل الـ module.
 * لو فشل التسجيل (مثلاً الـ symbol مش موجود أو الـ kprobes معطلة)
 * بيرجع error code والـ insmod بيفشل بشكل نظيف.
 * ----------------------------------------------------------------------- */
static int __init kprobe_lm_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("kprobe_linkmode: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("kprobe_linkmode: planted on %s at %px\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* -----------------------------------------------------------------------
 * module_exit — بيشيل الـ kprobe عند إزالة الـ module.
 * لازم نعمل unregister قبل ما الـ handler code يتشال من الذاكرة،
 * عشان منتجنبش kernel crash لو اتشغلت الدالة بعد الـ rmmod.
 * ----------------------------------------------------------------------- */
static void __exit kprobe_lm_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("kprobe_linkmode: removed from %s\n", kp.symbol_name);
}

module_init(kprobe_lm_init);
module_exit(kprobe_lm_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/module.h` | ماكرو `module_init` / `module_exit` والـ `MODULE_*` |
| `linux/kprobes.h` | `struct kprobe` وكل الـ API بتاع kprobes |
| `linux/kernel.h` | `pr_info` / `pr_err` |
| `linux/ethtool.h` | تعريف `ETHTOOL_LINK_MODE_Pause_BIT` و `ETHTOOL_LINK_MODE_Asym_Pause_BIT` |
| `linux/linkmode.h` | `linkmode_test_bit` عشان نفحص بيت معين في الـ bitmap |

الـ `linkmode.h` نفسه بيـ include الـ `ethtool.h` ضمنياً، لكن بنـ include الاتنين صراحةً عشان الـ code واضح ومش بيعتمد على تفاصيل implementation.

---

#### الـ `handler_pre`

الـ handler بياخد `struct pt_regs *regs` اللي فيها snapshot للـ CPU registers لحظة الاستدعاء. على x86-64 الـ calling convention بيحط أول argument في `rdi` وتاني argument في `rsi`، فـ `regs->di` = `local_adv` و `regs->si` = `partner_adv`.

بعدين بنستخدم `linkmode_test_bit` عشان نقرأ بيتين اتنين بس: **`Pause_BIT`** و **`Asym_Pause_BIT`** — ودول الاتنين اللي بيحددوا ناتج الـ `linkmode_resolve_pause` بالكامل حسب جدول 802.3.

---

#### الـ `struct kprobe`

بنبعت `symbol_name = "linkmode_resolve_pause"` بدل عنوان رقمي لأن الـ kernel بيعمل runtime lookup في الـ kallsyms table، وده أأمن وأسهل من hardcode. الـ `pre_handler` بيتشغل **قبل** الدالة الأصلية، فنشوف الـ input قبل ما يتغير أي حاجة.

---

#### الـ `module_init` / `module_exit`

`register_kprobe` بتحط **breakpoint instruction** (int3 على x86) في أول بايت من الدالة؛ لما الـ CPU يوصل عليه بيستدعي الـ handler بتاعنا. `unregister_kprobe` بتشيل الـ breakpoint وبتضمن إن مفيش استدعاء هيجي على handler بعد ما الـ module اتشال — ده ضروري عشان نتجنب **use-after-free** في الـ text segment.

---

### طريقة التجربة

```bash
# بناء الـ module (Makefile بسيط)
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# تحميل الـ module
sudo insmod kprobe_linkmode_pause.ko

# نحاكي auto-negotiation عن طريق إعادة تشغيل الـ link
sudo ip link set eth0 down && sudo ip link set eth0 up

# مراقبة الـ log
sudo dmesg | grep kprobe_linkmode

# إزالة الـ module
sudo rmmod kprobe_linkmode_pause
```

**مثال على output متوقع:**

```
kprobe_linkmode: planted on linkmode_resolve_pause at ffffffffc04a1230
kprobe[linkmode_resolve_pause]: local(Pause=1 AsymDir=0) partner(Pause=1 AsymDir=0)
kprobe_linkmode: removed from linkmode_resolve_pause
```

ده بيعني إن الـ local والـ partner الاتنين advertised الـ Pause بدون AsymDir، فالنتيجة هتكون TX+RX pause مفعّلين — وده متطابق مع الـ truth table في الـ linkmode.c source.
