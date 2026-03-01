## Phase 1: الصورة الكبيرة ببساطة

### ما هو هذا الملف؟

**`phylib_stubs.h`** هو ملف header صغير لكنه يحل مشكلة معمارية كبيرة جداً في الـ Linux kernel — مشكلة **التبعية بين الـ built-in code والـ kernel modules**.

---

### القصة: المشكلة اللي بتحلها

تخيل عندك شركة كبيرة فيها قسمين:

- **القسم الأساسي (built-in)**: موجود دايماً من أول ما الشركة تفتح — ده زي الـ `net/core/` و `net/ethtool/` اللي بتتحمل مع الـ kernel مباشرةً.
- **قسم خارجي (module)**: ممكن يبقى موجود أو لأ حسب الاحتياج — ده زي الـ `CONFIG_PHYLIB` (PHY library) اللي ممكن يتحمل كـ module منفصل.

المشكلة: القسم الأساسي **مش مسموحله** يكلم القسم الخارجي مباشرةً في وقت الـ compile. لو الـ built-in code عمل `#include <linux/phy.h>` واستدعى دوال من الـ PHY library مباشرةً، لو الـ `CONFIG_PHYLIB=m` (module) هيبقى في linker error لأن الدوال دي مش موجودة في الـ built-in image.

**الـ kernel بيحل ده إزاي؟** باستخدام نمط اسمه **stubs pattern**.

---

### الـ Stubs Pattern — الفكرة الجوهرية

بدل ما الـ built-in code يكلم الـ PHY library مباشرةً، بنعمل **وسيط**:

```
Built-in code (net/core, ethtool)
         │
         ▼
   phylib_stubs  ◄──── pointer (NULL لو PHYLIB مش محمّل)
         │
         ▼
   PHY Library (drivers/net/phy/) ◄── loaded كـ module أو built-in
```

**`phylib_stubs`** هو pointer لـ struct بيحتوي على function pointers. لما الـ PHYLIB module يتحمّل، بيسجّل نفسه (`phylib_register_stubs`). لما يتفكّ تحميله، بيشيل نفسه (`phylib_unregister_stubs`). لو مش محمّل، الـ pointer بيبقى `NULL` والدوال بترجع `-EOPNOTSUPP`.

---

### الدوال اللي بيوفرها هذا الملف

| الدالة | الوظيفة |
|--------|---------|
| `phy_hwtstamp_get()` | بتجيب إعدادات الـ hardware timestamping من الـ PHY |
| `phy_hwtstamp_set()` | بتظبط إعدادات الـ hardware timestamping على الـ PHY |
| `phy_ethtool_get_phy_stats()` | بتجيب إحصائيات الـ PHY عبر الـ ethtool |
| `phy_ethtool_get_link_ext_stats()` | بتجيب إحصائيات الـ link الممتدة |

كل دالة بتعمل:
1. `ASSERT_RTNL()` — بتتأكد إن الـ RTNL lock ماسكه عشان thread-safe
2. بتتحقق إن `phylib_stubs != NULL`
3. لو موجود، بتفوّض للـ PHY library الحقيقية

---

### ليه الـ `RTNL lock` مهم هنا؟

**الـ RTNL (Routing Netlink Lock)** هو الـ big lock بتاع الـ networking subsystem. الـ `phylib_register_stubs()` و `phylib_unregister_stubs()` بيشتغلوا تحت نفس الـ lock، فبيضمنوا إن مفيش race condition بين:
- قراءة الـ `phylib_stubs` pointer
- تغييره لما الـ module بيتحمّل أو بيتفكّ

---

### الفرق بين الحالتين (`#if IS_ENABLED(CONFIG_PHYLIB)`)

```c
// لو PHYLIB موجود (built-in أو module):
// بيستخدم الـ stubs pointer الحقيقي
static inline int phy_hwtstamp_get(...) {
    ASSERT_RTNL();
    if (!phylib_stubs) return -EOPNOTSUPP;  // لو الـ module مش محمّل
    return phylib_stubs->hwtstamp_get(...);  // delegate للـ real implementation
}

// لو PHYLIB مش موجود خالص في الـ config:
// stubs فارغة بترجع EOPNOTSUPP فوراً
static inline int phy_hwtstamp_get(...) {
    return -EOPNOTSUPP;
}
```

---

### الـ Subsystem: ETHERNET PHY LIBRARY

الملف ده جزء من **ETHERNET PHY LIBRARY** subsystem اللي بيديره:
- Andrew Lunn و Heiner Kallweit
- الـ mailing list: `netdev@vger.kernel.org`

---

### الملفات المرتبطة اللي المفروض تعرفها

#### الملفات الأساسية للـ subsystem:

| الملف | الدور |
|-------|-------|
| `include/linux/phy.h` | الـ main header — بيعرّف `struct phy_device` وكل الـ PHY API |
| `include/linux/phylib_stubs.h` | **الملف ده** — الـ stubs interface |
| `include/linux/mii.h` | تعريفات الـ MII (Media Independent Interface) |
| `include/linux/mdio.h` | الـ MDIO bus interface |
| `drivers/net/phy/phy_device.c` | الـ core PHY device driver — بيسجّل الـ stubs الحقيقية |
| `drivers/net/phy/stubs.c` | بيعرّف وبيـexport الـ `phylib_stubs` pointer نفسه |
| `net/core/dev_ioctl.c` | بيستخدم `phy_hwtstamp_get/set` للـ ioctl interface |
| `net/ethtool/stats.c` | بيستخدم `phy_ethtool_get_phy_stats` |
| `net/ethtool/linkstate.c` | بيستخدم `phy_ethtool_get_link_ext_stats` |
| `drivers/net/phy/` | كل درايفرات الـ PHY المختلفة (Broadcom, Marvell, إلخ) |
| `drivers/net/mdio/` | درايفرات الـ MDIO bus |

#### هيكل الـ subsystem:

```
include/linux/
├── phy.h                 ← main PHY API
├── phylib_stubs.h        ← هذا الملف (stubs interface)
├── mii.h                 ← MII definitions
├── mdio.h                ← MDIO bus
└── phy_fixed.h           ← fixed PHY support

drivers/net/phy/
├── phy_device.c          ← PHY core + stubs registration
├── stubs.c               ← phylib_stubs pointer export
├── phy.c                 ← PHY state machine
├── mdio_bus.c            ← MDIO bus driver
└── [vendor PHY drivers]  ← broadcom.c, marvell.c, etc.

net/core/dev_ioctl.c      ← يستخدم hwtstamp stubs
net/ethtool/
├── stats.c               ← يستخدم phy_stats stubs
└── linkstate.c           ← يستخدم link_ext_stats stubs
```

---

### ملخص: ليه هذا الملف موجود؟

الهدف الوحيد من `phylib_stubs.h` هو **كسر الـ compile-time dependency** بين الـ core networking stack (built-in) وبين الـ PHY library (قد تكون module). بيحقق ده عن طريق:

1. تعريف **function pointers struct** (`struct phylib_stubs`)
2. توفير **inline wrappers** تتحقق من وجود الـ stubs قبل الاستخدام
3. توفير **fallback implementations** بترجع `-EOPNOTSUPP` لما الـ PHY library مش موجودة

ده نمط شائع جداً في الـ kernel اسمه **"stubs for optional modules"** وبتلاقيه في أماكن تانية كتير زي الـ `nf_conntrack_stubs.h` وغيره.
## Phase 2: شرح الـ phylib_stubs Framework

### المشكلة — ليه الـ stubs دي موجودة أصلاً؟

الـ Linux kernel بيتبنى بأشكال مختلفة جداً. في embedded systems كتير، `CONFIG_PHYLIB` بيتعمله compile كـ **loadable module** مش built-in. ده بيعمل مشكلة تصميمية حقيقية:

**الـ core networking stack** (زي `net/core/dev_ioctl.c` و `net/ethtool/stats.c`) بيتعمله compile كـ built-in دايماً جوه الـ kernel image. لكن الوظايف اللي بتتعامل مع الـ PHY hardware (زي hardware timestamping وجمع statistics) موجودة في `PHYLIB` اللي ممكن يبقى module.

الـ built-in code **مش بيقدر يعمل direct call** لـ symbol موجود في module — لأن وقت الـ build مفيش ضمان إن الـ module ده هيتحمّل أصلاً.

```
الـ kernel image (built-in):
  net/core/dev_ioctl.c ──→ يعمل call لـ phy_hwtstamp_set()
                                         ↓
  لكن الـ implementation موجودة في:
  drivers/net/phy/phy.c ──→ قد يكون MODULE !
```

لو الـ kernel بعمل direct link لـ symbol في module، هيفضل في حالة **unresolved symbol** لو الـ module مش محمّل، وده crash محقق.

---

### الحل — نمط الـ Stubs مع Function Pointer Table

الكيرنل بيحل المشكلة دي بنمط بسيط بس ذكي: بدل ما يعمل direct call للـ symbol، بيخزّن **pointer لـ struct بيحتوي function pointers** في مكان fixed built-in.

الفكرة:

1. **الـ built-in code** يعرف بس الـ stub functions — وهي inline functions خفيفة بتبص على pointer عالمي.
2. **الـ module** لما يتحمّل، بيسجّل نفسه بيملّي الـ pointer ده بـ implementation الحقيقية.
3. لو الـ module مش محمّل، الـ pointer بيبقى `NULL` والـ stubs بترجع `-EOPNOTSUPP`.

ده نمط موجود في أماكن تانية في الكيرنل زي `dsa_stubs` و `ethtool_phy_ops`.

---

### تشبيه من الواقع — مكتبة Plugin

تخيّل إنك بتبني نظام تشغيل موسيقى:

| التشبيه | المقابل الحقيقي |
|---------|----------------|
| نظام التشغيل الأساسي (Fixed) | الـ core networking stack (built-in) |
| زرار "Play على Bluetooth" | `phy_hwtstamp_get()` stub |
| لو مفيش Bluetooth driver محمّل | `phylib_stubs == NULL` → ترجع error |
| لما تحمّل الـ Bluetooth plugin | `phylib_register_stubs()` يملّي الـ pointer |
| الـ plugin نفسه | `drivers/net/phy/phy_device.c` module |

لكن الفرق المهم هنا: الـ "زرار" ده محمي بـ **RTNL lock** — لأن الكيرنل لازم يضمن إن الـ module مش بيتفكّ load وهو بيتنفّذ الـ function pointer. ده مش مجرد تشبيه سطحي — ده ضمان تزامن حقيقي.

---

### الصورة الكبيرة — Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    BUILT-IN KERNEL CODE                         │
│                                                                 │
│  ┌──────────────────────┐    ┌──────────────────────────────┐  │
│  │  net/core/dev_ioctl.c│    │  net/ethtool/stats.c         │  │
│  │  phy_hwtstamp_set()  │    │  phy_ethtool_get_phy_stats() │  │
│  │  phy_hwtstamp_get()  │    │  phy_ethtool_get_link_ext..  │  │
│  └──────────┬───────────┘    └──────────────┬───────────────┘  │
│             │                               │                   │
│             ▼                               ▼                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │         include/linux/phylib_stubs.h                    │   │
│  │   static inline wrappers → check phylib_stubs != NULL   │   │
│  │   then call: phylib_stubs->hwtstamp_get/set/...         │   │
│  └─────────────────────────┬───────────────────────────────┘   │
│                             │                                   │
│  ┌──────────────────────────▼──────────────────────────────┐   │
│  │  drivers/net/phy/stubs.c (built-in)                     │   │
│  │  const struct phylib_stubs *phylib_stubs;  ← global ptr │   │
│  └──────────────────────────┬───────────────────────────────┘  │
└─────────────────────────────│───────────────────────────────────┘
                              │
                              │  يتملّى لما الـ module يتحمّل
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    PHYLIB MODULE (قد يكون loadable)             │
│                                                                 │
│  drivers/net/phy/phy_device.c                                   │
│  ┌─────────────────────────────────────┐                       │
│  │ static const struct phylib_stubs    │                       │
│  │ __phylib_stubs = {                  │                       │
│  │   .hwtstamp_get  = __phy_hwtstamp_get,                      │
│  │   .hwtstamp_set  = __phy_hwtstamp_set,                      │
│  │   .get_phy_stats = __phy_ethtool_get_phy_stats,             │
│  │   .get_link_ext_stats = __phy_ethtool_get_link_ext_stats,   │
│  │ };                                  │                       │
│  │                                     │                       │
│  │ phylib_register_stubs() ────────────┼──→ phylib_stubs = &__ │
│  │ phylib_unregister_stubs() ──────────┼──→ phylib_stubs = NULL│
│  └─────────────────────────────────────┘                       │
│                                                                 │
│  drivers/net/phy/phy.c                                          │
│  ┌─────────────────────────────────────┐                       │
│  │  __phy_hwtstamp_get()               │                       │
│  │  __phy_hwtstamp_set()               │                       │
│  │  __phy_ethtool_get_phy_stats()      │                       │
│  │  __phy_ethtool_get_link_ext_stats() │                       │
│  └─────────────────────────────────────┘                       │
└─────────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ `struct phylib_stubs` هي **vtable** — جدول function pointers بيمثّل الـ interface بين الـ built-in core stack والـ PHY module.

```c
/* include/linux/phylib_stubs.h */
struct phylib_stubs {
    /* hardware timestamping: قراءة config الـ timestamp من الـ PHY */
    int (*hwtstamp_get)(struct phy_device *phydev,
                        struct kernel_hwtstamp_config *config);

    /* hardware timestamping: كتابة config جديد على الـ PHY */
    int (*hwtstamp_set)(struct phy_device *phydev,
                        struct kernel_hwtstamp_config *config,
                        struct netlink_ext_ack *extack);

    /* قراءة PHY statistics (IEEE standard + vendor-specific) */
    void (*get_phy_stats)(struct phy_device *phydev,
                          struct ethtool_eth_phy_stats *phy_stats,
                          struct ethtool_phy_stats *phydev_stats);

    /* قراءة link-level extended stats (مثلاً link down events) */
    void (*get_link_ext_stats)(struct phy_device *phydev,
                               struct ethtool_link_ext_stats *link_stats);
};
```

الـ pointer العالمي:

```c
/* drivers/net/phy/stubs.c — built-in */
const struct phylib_stubs *phylib_stubs;
EXPORT_SYMBOL_GPL(phylib_stubs);
```

---

### تدفق تنفيذ stub function — خطوة بخطوة

لنأخذ `phy_hwtstamp_get()` كمثال:

```c
/* include/linux/phylib_stubs.h */
static inline int phy_hwtstamp_get(struct phy_device *phydev,
                                   struct kernel_hwtstamp_config *config)
{
    /* 1. التأكد إن RTNL محجوز — ده يمنع race مع register/unregister */
    ASSERT_RTNL();

    /* 2. لو الـ module مش محمّل، phylib_stubs == NULL */
    if (!phylib_stubs)
        return -EOPNOTSUPP;

    /* 3. الـ module محمّل → استدعي الـ real implementation */
    return phylib_stubs->hwtstamp_get(phydev, config);
}
```

الـ implementation الحقيقية في الـ module:

```c
/* drivers/net/phy/phy.c */
int __phy_hwtstamp_get(struct phy_device *phydev,
                       struct kernel_hwtstamp_config *config)
{
    if (!phydev)
        return -ENODEV;

    /* phydev->mii_ts هو الـ MII timestamper interface */
    if (phydev->mii_ts && phydev->mii_ts->hwtstamp_get)
        return phydev->mii_ts->hwtstamp_get(phydev->mii_ts, config);

    return -EOPNOTSUPP;
}
```

---

### دور الـ RTNL Lock — ليه ضروري هنا؟

**الـ RTNL (Routing Netlink Lock)** هو mutex عالمي للـ networking stack. يحتاج فهمه قبل المتابعة:

> الـ RTNL هو global mutex بيحمي كل التغييرات على الـ network configuration. موجود في `include/linux/rtnetlink.h`.

السبب إن الـ stubs بتستخدمه هنا:

```
Thread 1 (user calls hwtstamp_get):        Thread 2 (rmmod phylib):
  rtnl_lock()                                rtnl_lock()  ← blocked
  ASSERT_RTNL() ✓
  phylib_stubs != NULL ✓
  call phylib_stubs->hwtstamp_get()          ...يستنى
  rtnl_unlock()
                                             phylib_unregister_stubs()
                                             → phylib_stubs = NULL
                                             rtnl_unlock()
```

بدون الـ RTNL:
```
Thread 1:                         Thread 2:
  phylib_stubs != NULL ✓
                                  phylib_stubs = NULL  ← race!
  call phylib_stubs->...          ← NULL pointer dereference crash!
```

الكود في `phy_device.c` بيؤكد إن الـ register/unregister بيحصلوا تحت نفس الـ lock:

```c
/* drivers/net/phy/phy_device.c */
static int __init phy_init(void)
{
    rtnl_lock();
    ethtool_set_ethtool_phy_ops(&phy_ethtool_phy_ops);
    phylib_register_stubs();   /* تحت rtnl_lock */
    rtnl_unlock();
    ...
}

static void __exit phy_exit(void)
{
    rtnl_lock();
    phylib_unregister_stubs(); /* تحت rtnl_lock */
    ethtool_set_ethtool_phy_ops(NULL);
    rtnl_unlock();
}
```

---

### الـ `#if IS_ENABLED(CONFIG_PHYLIB)` — الـ compile-time split

الملف بيقسم نفسه بـ preprocessor:

```
CONFIG_PHYLIB=y أو =m           │  CONFIG_PHYLIB غير موجود
────────────────────────────────┼──────────────────────────────
phylib_stubs global pointer     │  مفيش pointer
inline functions بتبص عليه      │  inline functions فارغة
ASSERT_RTNL() موجودة            │  ترجع -EOPNOTSUPP مباشرة
```

لما `CONFIG_PHYLIB` مش موجود خالص:

```c
/* الـ #else branch في phylib_stubs.h */
static inline int phy_hwtstamp_get(struct phy_device *phydev,
                                   struct kernel_hwtstamp_config *config)
{
    return -EOPNOTSUPP;  /* مباشرة، من غير أي runtime check */
}
```

ده بيضمن إن الـ code اللي بيستخدم الـ stubs مش محتاج يعمل `#ifdef` هو كمان — الـ abstraction شافّة تماماً.

---

### الـ Struct Relationships

```
phylib_stubs (global ptr)
    │
    └──→ struct phylib_stubs
              ├── hwtstamp_get ──────────→ __phy_hwtstamp_get()
              │                                │
              │                                └──→ phydev->mii_ts->hwtstamp_get()
              │                                         (struct mii_timestamper)
              │
              ├── hwtstamp_set ──────────→ __phy_hwtstamp_set()
              │                                │
              │                                └──→ phydev->mii_ts->hwtstamp_set()
              │
              ├── get_phy_stats ─────────→ __phy_ethtool_get_phy_stats()
              │                                │
              │                                └──→ phydev->drv->get_phy_stats()
              │                                         (struct phy_driver)
              │
              └── get_link_ext_stats ───→ __phy_ethtool_get_link_ext_stats()
                                               │
                                               ├──→ phydev->link_down_events
                                               └──→ phydev->drv->get_link_stats()
```

كل function في الـ vtable بتتعامل مع الـ `struct phy_device` اللي بيحتوي على:
- **`mii_ts`**: pointer لـ `struct mii_timestamper` — اللي بيوفّر الـ hardware timestamping
- **`drv`**: pointer لـ `struct phy_driver` — driver الـ PHY الفعلي

---

### الـ Subsystem بيمتلك إيه؟ وإيه اللي بيفوّضه؟

| الـ phylib_stubs يمتلك | بيفوّض لـ |
|------------------------|-----------|
| تعريف الـ interface (الـ vtable) | الـ `PHYLIB` module لتنفيذ الـ functions |
| الـ global pointer (`phylib_stubs`) | كل PHY driver لتقديم stats بصورة مختلفة |
| الـ safety check (NULL guard + RTNL) | الـ `phy_driver` لقراءة hardware registers |
| الـ compile-time fallback (`#else`) | الـ `mii_timestamper` لإدارة الـ PTP timestamps |

---

### مقارنة مع نمط مشابه — `dsa_stubs`

الـ DSA (Distributed Switch Architecture) بيستخدم نفس النمط تماماً:

```c
/* include/net/dsa_stubs.h */
struct dsa_stubs {
    int (*conduit_hwtstamp_validate)(struct net_device *dev,
                                    const struct kernel_hwtstamp_config *config,
                                    struct netlink_ext_ack *extack);
};
```

الفرق الوحيد: الـ `dsa_stubs` بيستخدم `netdev_uses_dsa()` كـ guard إضافي قبل ما يبص على الـ pointer. ده نمط kernel-wide معتمد لحل مشكلة built-in vs module dependencies.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### ملخص الـ Config Options والـ Flags

| العنصر | القيمة / النوع | المعنى |
|--------|---------------|---------|
| `CONFIG_PHYLIB` | `bool` kconfig | لو enabled، الكود الحقيقي يتحمّل — لو لا، كل الـ stubs بترجع `-EOPNOTSUPP` أو فراغ |
| `CONFIG_PROVE_LOCKING` | `bool` kconfig | بيفعّل `lockdep_rtnl_is_held()` الحقيقية بدل الـ stub اللي بترجع `true` دايماً |
| `CONFIG_DEBUG_NET_SMALL_RTNL` | `bool` kconfig | بيوفّر per-net RTNL بدل global RTNL |
| `ASSERT_RTNL()` | macro | بيـ`WARN` لو الـ caller مش شايل `rtnl_lock` — لو الـ lock مش مأخوذ الكود يوقف |
| `-EOPNOTSUPP` | `int` errno | بيترجعه لما `CONFIG_PHYLIB=n` أو لما `phylib_stubs == NULL` |

---

### الـ Structs المهمة

#### 1. `struct phylib_stubs`

**الغرض:** جدول function pointers — بمثابة vtable بسيطة تربط الكود اللي مش بيعرف phylib بالتنفيذ الحقيقي في `phy_device.c` لما الـ module يتحمّل.

```c
struct phylib_stubs {
    /* get hardware timestamp config من الـ PHY */
    int (*hwtstamp_get)(struct phy_device *phydev,
                        struct kernel_hwtstamp_config *config);

    /* set hardware timestamp config في الـ PHY */
    int (*hwtstamp_set)(struct phy_device *phydev,
                        struct kernel_hwtstamp_config *config,
                        struct netlink_ext_ack *extack);

    /* اقرأ إحصائيات الـ PHY layer */
    void (*get_phy_stats)(struct phy_device *phydev,
                          struct ethtool_eth_phy_stats *phy_stats,
                          struct ethtool_phy_stats *phydev_stats);

    /* اقرأ إحصائيات link ext زي link_down_events */
    void (*get_link_ext_stats)(struct phy_device *phydev,
                               struct ethtool_link_ext_stats *link_stats);
};
```

| الـ Field | النوع | الوظيفة |
|-----------|-------|---------|
| `hwtstamp_get` | `fn ptr` | بيجيب الـ PTP/hardware timestamp config الحالية من PHY |
| `hwtstamp_set` | `fn ptr` | بيضبط الـ hardware timestamping mode في PHY |
| `get_phy_stats` | `fn ptr` | بيقرأ IEEE 802.3 PHY stats + vendor stats |
| `get_link_ext_stats` | `fn ptr` | بيجيب `link_down_events` من مستوى الـ PHY |

**الـ Global Instance:** `const struct phylib_stubs *phylib_stubs` — متغير global عام بيتحمّل بالـ pointer لما `phylib` يتسجّل، وبيبقى `NULL` قبل كده أو بعد unregister.

---

#### 2. `struct phy_device` (مختصر — الـ fields المستخدمة في السياق ده)

**الغرض:** يمثّل جهاز PHY فيزيائي على الـ MDIO bus — هو المحور اللي كل عمليات الـ timestamping والـ stats بتتعمل عليه.

| الـ Field المهم | النوع | الوظيفة في السياق |
|----------------|-------|------------------|
| `mdio` | `struct mdio_device` | أساس الـ device على MDIO bus |
| `drv` | `const struct phy_driver *` | driver الـ PHY وفيه ops زي `hwtstamp_get/set` الحقيقية |
| `state` | `enum phy_state` | حالة الـ PHY (DOWN / UP / RUNNING إلخ) |
| `link` | `unsigned :1` | آخر link state معروف |
| `default_timestamp` | `unsigned :1` | الـ PHY ده هو مصدر الـ timestamp الافتراضي؟ |

---

#### 3. `struct kernel_hwtstamp_config`

**الغرض:** نسخة kernel-internal من `struct hwtstamp_config` للـ SIOCSHWTSTAMP/SIOCGHWTSTAMP — بتتجنب الـ UAPI مباشرة.

```c
struct kernel_hwtstamp_config {
    int flags;          /* SOF_TIMESTAMPING_* flags */
    int tx_type;        /* HWTSTAMP_TX_* */
    int rx_filter;      /* HWTSTAMP_FILTER_* */
    struct ifreq *ifr;  /* pointer للـ ioctl request الأصلي (legacy) */
    bool copied_to_user;/* legacy driver نسخ البيانات للـ user بالفعل */
    enum hwtstamp_source source;    /* netdev أو phylib PHY */
    enum hwtstamp_provider_qualifier qualifier; /* مؤهّل المزوّد */
};
```

---

#### 4. `struct ethtool_eth_phy_stats`

**الغرض:** إحصائيات IEEE 802.3 standard للـ PHY layer.

```c
struct ethtool_eth_phy_stats {
    enum ethtool_mac_stats_src src;   /* مصدر الإحصائية */
    struct_group(stats,
        u64 SymbolErrorDuringCarrier; /* أخطاء الـ symbol أثناء الـ carrier */
    );
};
```

---

#### 5. `struct ethtool_phy_stats`

**الغرض:** إحصائيات vendor-specific للـ PHY مش معرّفة في IEEE — packets، bytes، errors.

```c
struct ethtool_phy_stats {
    u64 rx_packets;  u64 rx_bytes;  u64 rx_errors;
    u64 tx_packets;  u64 tx_bytes;  u64 tx_errors;
};
```

---

#### 6. `struct ethtool_link_ext_stats`

**الغرض:** إحصائيات link-level موسّعة — حالياً بس `link_down_events`.

```c
struct ethtool_link_ext_stats {
    u64 link_down_events; /* عدد مرات انقطاع الـ PHY link فعلياً */
};
```

---

#### 7. `struct netlink_ext_ack`

**الغرض:** بيحمل رسائل خطأ moreعن الـ netlink — بيتمرر لـ `hwtstamp_set` عشان يبعت error message مفيد للـ user space.

```c
struct netlink_ext_ack {
    const char *_msg;            /* رسالة الخطأ */
    const struct nlattr *bad_attr; /* الـ attribute اللي عمل المشكلة */
    /* ... */
};
```

---

### علاقات الـ Structs — ASCII Diagram

```
                     ┌─────────────────────────────────────┐
                     │        phylib_stubs (global ptr)    │
                     │  const struct phylib_stubs *        │
                     └──────────────┬──────────────────────┘
                                    │ points to (under rtnl_lock)
                                    ▼
                     ┌─────────────────────────────────────┐
                     │       struct phylib_stubs           │
                     │  ┌──────────────────────────────┐   │
                     │  │ hwtstamp_get    (fn ptr) ────┼───┼──► __phy_hwtstamp_get()
                     │  │ hwtstamp_set    (fn ptr) ────┼───┼──► __phy_hwtstamp_set()
                     │  │ get_phy_stats   (fn ptr) ────┼───┼──► __phy_ethtool_get_phy_stats()
                     │  │ get_link_ext_stats (fn ptr) ─┼───┼──► __phy_ethtool_get_link_ext_stats()
                     │  └──────────────────────────────┘   │
                     └─────────────────────────────────────┘

كل fn ptr بياخد:
  ┌──────────────────┐      ┌────────────────────────────┐
  │  struct phy_device│      │ struct kernel_hwtstamp_config│
  │  ├─ mdio         │      │ ├─ flags                   │
  │  ├─ drv ─────────┼─────►│ ├─ tx_type                │
  │  ├─ state        │      │ ├─ rx_filter               │
  │  └─ link         │      │ └─ source                  │
  └──────────────────┘      └────────────────────────────┘
           │
           │ (get_phy_stats يكتب في)
           ▼
  ┌──────────────────────────┐   ┌─────────────────────────┐
  │ ethtool_eth_phy_stats    │   │ ethtool_phy_stats        │
  │ ├─ src                  │   │ ├─ rx_packets            │
  │ └─ SymbolErrorDuringCarr│   │ ├─ tx_packets            │
  └──────────────────────────┘   │ └─ tx/rx errors/bytes   │
                                 └─────────────────────────┘
           │ (get_link_ext_stats يكتب في)
           ▼
  ┌──────────────────────────┐
  │ ethtool_link_ext_stats   │
  │ └─ link_down_events      │
  └──────────────────────────┘
```

---

### Lifecycle Diagram — من التحميل للتنظيف

```
kernel boot
    │
    ▼
subsys_initcall(phy_init)           ← في phy_device.c
    │
    ├─► rtnl_lock()                 ← lock أول حاجة
    ├─► phylib_register_stubs()
    │       └─► phylib_stubs = &__phylib_stubs   ← global ptr يتعبّى
    ├─► rtnl_unlock()
    │
    ▼
[phylib module شغّال]
    │
    │   أي caller بيعمل:
    │   phy_hwtstamp_get() / phy_hwtstamp_set()
    │   phy_ethtool_get_phy_stats() / phy_ethtool_get_link_ext_stats()
    │       │
    │       ├─► ASSERT_RTNL()      ← caller لازم يكون شايل rtnl_lock
    │       ├─► if (!phylib_stubs) return -EOPNOTSUPP / return;
    │       └─► phylib_stubs->fn(phydev, ...)   ← dispatch
    │
    ▼
module unload OR init failure
    │
    ├─► rtnl_lock()
    ├─► phylib_unregister_stubs()
    │       └─► phylib_stubs = NULL              ← global ptr يتمسح
    └─► rtnl_unlock()
```

---

### Call Flow Diagrams

#### A. `phy_hwtstamp_get()` — قراءة الـ HW timestamp config

```
ethtool أو kernel code
    │
    └─► phy_hwtstamp_get(phydev, config)      [phylib_stubs.h — inline]
            │
            ├─► ASSERT_RTNL()                 ← كرّاسة السلامة
            │
            ├─► phylib_stubs == NULL?
            │       └─► return -EOPNOTSUPP    ← لو module مش محمّل
            │
            └─► phylib_stubs->hwtstamp_get(phydev, config)
                    │                         [dispatch عبر fn ptr]
                    └─► __phy_hwtstamp_get()  [phy_device.c]
                            │
                            └─► phydev->drv->hwtstamp_get(phydev, config)
                                    │         [PHY driver الفعلي]
                                    └─► hardware register read
```

#### B. `phy_hwtstamp_set()` — ضبط الـ HW timestamp

```
ethtool أو kernel code
    │
    └─► phy_hwtstamp_set(phydev, config, extack)  [phylib_stubs.h — inline]
            │
            ├─► ASSERT_RTNL()
            │
            ├─► phylib_stubs == NULL?
            │       └─► return -EOPNOTSUPP
            │
            └─► phylib_stubs->hwtstamp_set(phydev, config, extack)
                    │
                    └─► __phy_hwtstamp_set()  [phy_device.c]
                            │
                            ├─► validate config
                            └─► phydev->drv->hwtstamp_set(phydev, config, extack)
                                    │
                                    └─► hardware register write
                                        + NL_SET_ERR_MSG on error → extack
```

#### C. `phy_ethtool_get_phy_stats()` — جمع الإحصائيات

```
ethtool stats request
    │
    └─► phy_ethtool_get_phy_stats(phydev, phy_stats, phydev_stats)
            │
            ├─► ASSERT_RTNL()
            │
            ├─► phylib_stubs == NULL? → return (void)
            │
            └─► phylib_stubs->get_phy_stats(phydev, phy_stats, phydev_stats)
                    │
                    └─► __phy_ethtool_get_phy_stats()
                            │
                            ├─► يملأ ethtool_eth_phy_stats (IEEE 802.3)
                            └─► يملأ ethtool_phy_stats (vendor counters)
```

---

### استراتيجية الـ Locking

**القاعدة الأساسية:** كل عمليات الـ stub — register، unregister، وكل inline call — تتعمل تحت `rtnl_lock()`.

```
┌────────────────────────────────────────────────────────────┐
│                    rtnl_lock scope                         │
│                                                            │
│  phylib_register_stubs()   →  phylib_stubs = &stubs       │
│  phylib_unregister_stubs() →  phylib_stubs = NULL         │
│  phy_hwtstamp_get()        →  ASSERT_RTNL + read ptr      │
│  phy_hwtstamp_set()        →  ASSERT_RTNL + read ptr      │
│  phy_ethtool_get_phy_stats()    → ASSERT_RTNL + read ptr  │
│  phy_ethtool_get_link_ext_stats() → ASSERT_RTNL + read ptr│
└────────────────────────────────────────────────────────────┘
```

**ليه `rtnl_lock` وبس؟**

الـ `phylib_stubs` pointer مش protected بـ RCU أو spinlock — هو protected بـ `rtnl_lock` فقط. ده تصميم مقصود:
- الـ callers الحقيقيين (ethtool، netdev config) هم أصلاً شايلين `rtnl_lock` قبل ما يوصلوا لهنا.
- لا داعي لـ atomic أو RCU لأن الـ write (register/unregister) نادر جداً وبيحصل بس وقت module load/unload.

**ترتيب الـ Lock (lock ordering):**

```
rtnl_lock()          ← دايماً الأول
    └─► phylib_stubs->fn()
            └─► phydev->drv->hwtstamp_*()
                    └─► device-level lock (لو موجود في الـ driver)
```

`ASSERT_RTNL()` هو الـ enforcement mechanism — لو حد استدعى أي inline function من غير `rtnl_lock`، الـ kernel هيـprint `WARN` مع الـ file والـ line number.

---

### تصميم الـ Stub Pattern — ليه موجود؟

الهدف هو فصل dependency وقت compile-time بين:
- **الكود اللي بيستخدم PHY** (مثلاً ethtool، netdev ops) — مش عايز يـdepend على `CONFIG_PHYLIB` مباشرة.
- **الـ phylib module** نفسه — ممكن يكون module أو built-in.

```
بدون stubs:
  ethtool.c → #ifdef CONFIG_PHYLIB → phy_device.c  (tight coupling)

مع stubs:
  ethtool.c → phy_hwtstamp_get() [stub inline]
                    ↓ (runtime)
              phylib_stubs->hwtstamp_get()  ← لو محمّل
              -EOPNOTSUPP                   ← لو مش محمّل
```

ده نفس pattern الـ `tracepoints` و`ftrace` و`netfilter hooks` في الـ kernel — runtime pluggability بدون compile-time coupling.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

| Function | Return | RTNL Required | CONFIG_PHYLIB=n |
|---|---|---|---|
| `phy_hwtstamp_get()` | `int` | نعم (`ASSERT_RTNL`) | `-EOPNOTSUPP` |
| `phy_hwtstamp_set()` | `int` | نعم (`ASSERT_RTNL`) | `-EOPNOTSUPP` |
| `phy_ethtool_get_phy_stats()` | `void` | نعم (`ASSERT_RTNL`) | no-op |
| `phy_ethtool_get_link_ext_stats()` | `void` | نعم (`ASSERT_RTNL`) | no-op |

**الـ struct الوحيد في الملف:**

| Field | النوع | الوظيفة |
|---|---|---|
| `hwtstamp_get` | function pointer | قراءة hardware timestamp config من الـ PHY |
| `hwtstamp_set` | function pointer | كتابة hardware timestamp config على الـ PHY |
| `get_phy_stats` | function pointer | جلب إحصائيات Ethernet PHY layer |
| `get_link_ext_stats` | function pointer | جلب إحصائيات الـ link الموسّعة |

---

### المنطق العام للملف

**الـ `phylib_stubs.h`** يحل مشكلة الـ **optional module dependency** بين الـ network drivers والـ `CONFIG_PHYLIB`. لما تكون `CONFIG_PHYLIB=m` أو `=y`، الـ PHY library بتسجّل نفسها عن طريق `phylib_stubs` global pointer — وده نمط الـ **late binding via function pointer table**. لما تكون `CONFIG_PHYLIB=n`، كل الـ inline functions بترجع قيم افتراضية آمنة (stub behavior).

الـ pattern ده بيخلّي الـ Ethernet MAC drivers يستخدموا PHY features من غير ما يكونوا compile-time dependent على `phylib`.

**ليه RTNL؟** الـ `phylib_stubs` pointer نفسه بيتعدّل في `phylib_register_stubs()` و`phylib_unregister_stubs()` — الاتنين بياخدوا الـ `rtnl_lock()`. فـ الكود بيضمن إن مفيش race بين قراءة الـ pointer واستخدامه عن طريق إجبار الـ caller إنه يكون شايل الـ lock.

---

### Group 1: الـ Stub Table — `struct phylib_stubs`

```c
struct phylib_stubs {
    int (*hwtstamp_get)(struct phy_device *phydev,
                        struct kernel_hwtstamp_config *config);
    int (*hwtstamp_set)(struct phy_device *phydev,
                        struct kernel_hwtstamp_config *config,
                        struct netlink_ext_ack *extack);
    void (*get_phy_stats)(struct phy_device *phydev,
                          struct ethtool_eth_phy_stats *phy_stats,
                          struct ethtool_phy_stats *phydev_stats);
    void (*get_link_ext_stats)(struct phy_device *phydev,
                               struct ethtool_link_ext_stats *link_stats);
};
```

الـ `struct phylib_stubs` هي **vtable** صغيرة بتحوي كل العمليات اللي محتاجة `CONFIG_PHYLIB`. الـ `phylib` module بيملّي الـ struct ده ويحطّ عنوانه في الـ global pointer `phylib_stubs` وقت الـ module init. لما الـ module بيتفكّ تحميله، بيحطّ الـ pointer على `NULL` تاني.

الـ global pointer:
```c
extern const struct phylib_stubs *phylib_stubs;
```

**الـ locking model:** الـ pointer ده محمي بالـ `rtnl_lock()` — مش بـ RCU ومش بـ spinlock. ده معناه إن الـ read والـ write الاتنين بيحصلوا تحت الـ RTNL، وده يكفي لأن الـ PHY operations دي كلها slow path (ethtool / ioctl context).

---

### Group 2: Hardware Timestamping Functions

#### `phy_hwtstamp_get()`

```c
static inline int phy_hwtstamp_get(struct phy_device *phydev,
                                   struct kernel_hwtstamp_config *config);
```

بتقرأ الـ **hardware timestamping configuration** الحالية من الـ PHY device. الـ function دي بتكون entry point للـ `ethtool` أو الـ `ioctl` اللي بيعمل `SIOCGHWTSTAMP`. بتتحقق أولاً إن الـ `phylib_stubs` pointer مش NULL — لو NULL يعني الـ phylib مش loaded — فبترجع `-EOPNOTSUPP`. لو موجود، بتفوّض الاستدعاء للـ real implementation في الـ phylib module.

**Parameters:**
- `phydev` — الـ PHY device المراد قراءة الـ timestamping config بتاعته. الـ struct ده بيحتوي على كل حالة الـ PHY (state machine، driver pointer، إلخ).
- `config` — pointer لـ `struct kernel_hwtstamp_config` اللي بيتملّى بالـ config الحالية. الـ struct ده هو الـ kernel-internal version بعد ما اتحوّل من الـ userspace `hwtstamp_config`.

**Return value:** `0` في النجاح، أو error code سالب. الأكتر شيوعاً: `-EOPNOTSUPP` لو الـ PHY أو الـ phylib مش موجود.

**Key details:**
- **Locking:** `ASSERT_RTNL()` بيعمل `WARN_ONCE` لو الـ caller مش شايل الـ `rtnl_lock()`. الـ phylib registration functions كمان شايلة RTNL، فمفيش tearing للـ pointer.
- الـ fallback لـ `CONFIG_PHYLIB=n` بترجع `-EOPNOTSUPP` مباشرةً من غير أي checks.

**Caller context:** `ethtool_get_ts_info()` أو `dev_get_hwtstamp()` في networking stack، دايماً من process context تحت RTNL.

**Pseudocode flow:**
```
phy_hwtstamp_get():
    ASSERT_RTNL()                    // crash-warn if lock not held
    if phylib_stubs == NULL:
        return -EOPNOTSUPP           // module not loaded
    return phylib_stubs->hwtstamp_get(phydev, config)
                                     // delegate to real phylib impl
```

---

#### `phy_hwtstamp_set()`

```c
static inline int phy_hwtstamp_set(struct phy_device *phydev,
                                   struct kernel_hwtstamp_config *config,
                                   struct netlink_ext_ack *extack);
```

بتكتب **hardware timestamping configuration** جديدة على الـ PHY. دي الـ write counterpart لـ `phy_hwtstamp_get()`. بتُستدعى لما الـ userspace يعمل `SIOCSHWTSTAMP` أو ethtool set timestamp. نفس منطق الـ guard: RTNL check ثم NULL check على الـ `phylib_stubs`.

**Parameters:**
- `phydev` — الـ PHY device المستهدف.
- `config` — الـ config الجديدة اللي المفروض تتطبّق. الـ PHY driver الحقيقي ممكن يعدّلها لو الـ hardware مش بيدعم القيم المطلوبة بالظبط (nearest-match behavior).
- `extack` — **Extended ACK** — بيسمح للـ kernel يبعت error message نصي للـ userspace عن طريق netlink. بيُستخدم لما الـ PHY driver يرفض config معيّنة ويعايز يقول ليه.

**Return value:** `0` في النجاح، error code سالب في الفشل (مثلاً `-ERANGE`, `-EINVAL`, `-EOPNOTSUPP`).

**Key details:**
- **Locking:** نفس الـ RTNL requirement. التغيير في الـ timestamping config هو slow-path operation.
- الـ `extack` بيمر للـ underlying implementation — مش بيتعامل مع في الـ stub نفسه.
- الـ fallback لـ `CONFIG_PHYLIB=n` بترجع `-EOPNOTSUPP` وبتتجاهل `extack`.

**Caller context:** `ethtool_set_hwtstamp()` أو `dev_set_hwtstamp()` — process context تحت RTNL.

**Pseudocode flow:**
```
phy_hwtstamp_set():
    ASSERT_RTNL()
    if phylib_stubs == NULL:
        return -EOPNOTSUPP
    return phylib_stubs->hwtstamp_set(phydev, config, extack)
```

---

### Group 3: Statistics Collection Functions

#### `phy_ethtool_get_phy_stats()`

```c
static inline void phy_ethtool_get_phy_stats(
                        struct phy_device *phydev,
                        struct ethtool_eth_phy_stats *phy_stats,
                        struct ethtool_phy_stats *phydev_stats);
```

بتجمع **إحصائيات PHY layer** من نوعين مختلفين في نفس الاستدعاء. الـ `ethtool_eth_phy_stats` بيحتوي على الـ standard IEEE 802.3 PHY stats (الـ MIB counters)، والـ `ethtool_phy_stats` بيحتوي على stats driver-specific. بتُستدعى من ethtool core وقت `ETHTOOL_GPHYSTATS`.

**Parameters:**
- `phydev` — الـ PHY device المصدر للإحصائيات.
- `phy_stats` — buffer بيتملّى بالـ standard IEEE PHY statistics (مثلاً: symbol errors، false carrier events).
- `phydev_stats` — buffer للـ vendor-specific / driver-specific PHY stats.

**Return value:** `void` — لو الـ phylib مش loaded، الـ function بترجع بهدوء من غير تعديل في الـ buffers.

**Key details:**
- **Locking:** `ASSERT_RTNL()` — نفس النمط.
- لو الـ `phylib_stubs == NULL`، الـ buffers بتفضل زي ما هي (مملّيين بـ zeros من الـ caller الأعلى عادةً). ده سلوك آمن.
- الـ fallback لـ `CONFIG_PHYLIB=n` هو empty function — no-op كامل.

**Caller context:** `ethtool_get_phy_stats()` في `net/ethtool/stats.c` — process context تحت RTNL.

**Pseudocode flow:**
```
phy_ethtool_get_phy_stats():
    ASSERT_RTNL()
    if phylib_stubs == NULL:
        return                       // silent no-op, buffers untouched
    phylib_stubs->get_phy_stats(phydev, phy_stats, phydev_stats)
```

---

#### `phy_ethtool_get_link_ext_stats()`

```c
static inline void phy_ethtool_get_link_ext_stats(
                        struct phy_device *phydev,
                        struct ethtool_link_ext_stats *link_stats);
```

بتجمع **إحصائيات الـ link الموسّعة** من الـ PHY — زي link-down events، receiver errors، إلخ. دي إحصائيات مرتبطة بحالة الـ physical link بدل الـ MAC counters. بتُستدعى من ethtool core وقت استعلامات الـ link extended stats.

**Parameters:**
- `phydev` — الـ PHY device.
- `link_stats` — pointer لـ `struct ethtool_link_ext_stats` — بيتملّى بـ link-layer extended counters.

**Return value:** `void` — نفس السلوك الآمن لو الـ phylib مش موجود.

**Key details:**
- **Locking:** `ASSERT_RTNL()`.
- الـ fallback لـ `CONFIG_PHYLIB=n` هو empty body — no-op.
- الفرق بينها وبين `phy_ethtool_get_phy_stats()`: الأخيرة بتاخد buffer اتنين (standard + vendor)، الأولى بتاخد buffer واحد للـ link-specific stats.

**Caller context:** ethtool stats subsystem، process context تحت RTNL.

**Pseudocode flow:**
```
phy_ethtool_get_link_ext_stats():
    ASSERT_RTNL()
    if phylib_stubs == NULL:
        return
    phylib_stubs->get_link_ext_stats(phydev, link_stats)
```

---

### الـ Conditional Compilation — البنية الكاملة

الملف بيستخدم `#if IS_ENABLED(CONFIG_PHYLIB)` لتقسيم الـ implementations:

```
CONFIG_PHYLIB=y أو =m             CONFIG_PHYLIB=n
─────────────────────────────    ──────────────────────────
struct phylib_stubs { ... }      (مش موجود)
extern phylib_stubs *            (مش موجود)

phy_hwtstamp_get():              phy_hwtstamp_get():
  ASSERT_RTNL()                    return -EOPNOTSUPP
  check stubs != NULL
  call stubs->hwtstamp_get()

phy_ethtool_get_phy_stats():     phy_ethtool_get_phy_stats():
  ASSERT_RTNL()                    { } // empty
  check stubs != NULL
  call stubs->get_phy_stats()
```

الـ `IS_ENABLED()` بيشمل `=m` و`=y` معاً — وده صح هنا لأن الـ stub pointer بيتملّى وقت module init لو كانت `=m`، وبيتصفّر وقت module exit.

---

### نقاط تصميمية مهمة

**1. ليه مش RCU؟**
الـ `phylib_stubs` pointer محمي بـ RTNL مش RCU لأن:
- كل الـ callers (ethtool, ioctl) أصلاً شايلين RTNL.
- الـ register/unregister بياخدوا RTNL كمان.
- ده يُلغي الحاجة لـ `rcu_read_lock()` / `synchronize_rcu()` زيادة على الـ RTNL.

**2. ليه `inline` functions بدل macros؟**
الـ `static inline` بتضمن type checking كامل للـ parameters — مهم خصوصاً للـ pointer types المختلفة للـ stats structs.

**3. الـ `ASSERT_RTNL()` مش lock**
`ASSERT_RTNL()` هو `WARN_ONCE(!rtnl_is_locked(), ...)` — بس تحذير. ما بيعملش lock ولا بيوقف التنفيذ. الـ caller مسؤول إنه ياخد الـ lock قبل ما يستدعي أي function من دول.
## Phase 5: دليل الـ Debugging الشامل

الـ `phylib_stubs.h` بيوفر **stub interface** للـ PHY library — بيربط الـ networking core بالـ `CONFIG_PHYLIB` module عبر **vtable pointer** (`phylib_stubs`) محمي بالـ RTNL lock. الـ debugging هنا بيركز على: هل الـ stubs متسجلة؟ هل الـ RTNL محجوز صح؟ هل الـ PHY hardware بيرد؟

---

### Software Level

#### 1. debugfs Entries

الـ phylib بيسجّل entries جوه `/sys/kernel/debug/`:

```bash
# شوف كل الـ PHY devices المتسجلة
ls /sys/kernel/debug/mdio_bus/

# قرا state الـ PHY device معين (مثلاً: mdio0:01)
cat /sys/kernel/debug/mdio_bus/mdio0:01
```

**الـ output المتوقع:**

```
PHY 0:01  addr 1  connected to: eth0
  state: PHY_RUNNING (7)
  link: 1
  speed: 1000
  duplex: 1
```

| Entry | المعنى |
|-------|--------|
| `state` | حالة state machine الـ PHY |
| `link` | 1=up, 0=down |
| `speed` | السرعة بالـ Mbps |
| `duplex` | 1=full, 0=half |

```bash
# تحقق إن phylib_stubs pointer متسجل (عبر kallsyms)
grep phylib_stubs /proc/kallsyms
```

**output طبيعي:**

```
ffffffffc0a12480 D phylib_stubs	[phy]
```

لو الـ output فارغ → الـ module مش loaded أو `CONFIG_PHYLIB=n`.

---

#### 2. sysfs Entries

```bash
# كل الـ PHY devices في الـ system
ls /sys/bus/mdio_bus/devices/

# state الـ PHY الحالية
cat /sys/bus/mdio_bus/devices/mdio0\:01/phy_id
cat /sys/bus/mdio_bus/devices/mdio0\:01/phy_has_fixups

# إحصائيات الـ link (ethtool stats تمر عبر phylib_stubs)
cat /sys/class/net/eth0/statistics/rx_errors
cat /sys/class/net/eth0/statistics/tx_errors
```

---

#### 3. ftrace — Tracepoints/Events

```bash
# شوف الـ events المتاحة للـ net/phy subsystem
ls /sys/kernel/debug/tracing/events/net/
ls /sys/kernel/debug/tracing/events/mdio/

# فعّل tracing للـ mdio read/write (اللي بيستخدمه الـ PHY)
echo 1 > /sys/kernel/debug/tracing/events/mdio/enable

# فعّل function tracing للـ phy_hwtstamp_get و phy_hwtstamp_set
echo phy_hwtstamp_get phy_hwtstamp_set > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل الـ test ثم اقرا النتيجة
cat /sys/kernel/debug/tracing/trace

# reset
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo nop > /sys/kernel/debug/tracing/current_tracer
```

**example trace output:**

```
# TASK-PID   CPU#  TIMESTAMP  FUNCTION
  ethtool-1234 [000] 12345.678901: phy_hwtstamp_get <-ethtool_get_ts_info
  ethtool-1234 [000] 12345.678950: phy_hwtstamp_set <-ethtool_set_ts_info
```

```bash
# trace الـ rtnl_lock acquisition اللي بيحميه
echo rtnl_lock rtnl_unlock >> /sys/kernel/debug/tracing/set_ftrace_filter
```

---

#### 4. printk / Dynamic Debug

```bash
# فعّل dynamic debug لكل الـ phy module
echo "module phy +pflmt" > /sys/kernel/debug/dynamic_debug/control

# أو فعّله لملف معين
echo "file drivers/net/phy/phy.c +pflmt" > /sys/kernel/debug/dynamic_debug/control

# شوف الـ kernel log مباشرة
dmesg -w | grep -E "phy|mdio|phylib"

# فعّل الـ debug للـ mdio bus
echo "module mdio-bus +pflmt" > /sys/kernel/debug/dynamic_debug/control
```

**flags المستخدمة:**
- `p` = print the message
- `f` = include function name
- `l` = include line number
- `m` = include module name
- `t` = include thread ID

---

#### 5. Kernel Config Options للـ Debugging

```bash
# تحقق من الـ config الحالي
grep -E "CONFIG_PHYLIB|CONFIG_NET_PHY|CONFIG_MDIO|CONFIG_DEBUG_NET|CONFIG_PROVE_LOCKING|CONFIG_LOCKDEP" /boot/config-$(uname -r)
```

| Config | الغرض |
|--------|--------|
| `CONFIG_PHYLIB=y/m` | تفعيل الـ PHY library — **لازم يكون enabled** عشان `phylib_stubs` تتسجل |
| `CONFIG_PROVE_LOCKING=y` | تفعيل `lockdep_rtnl_is_held()` الحقيقية (مش الـ stub) |
| `CONFIG_LOCKDEP=y` | full lock dependency tracking |
| `CONFIG_DEBUG_NET_SMALL_RTNL=y` | fine-grained RTNL per-net locking debugging |
| `CONFIG_MDIO_DEVICE=y` | دعم الـ MDIO bus |
| `CONFIG_DEBUG_KERNEL=y` | تفعيل kernel debugging العام |
| `CONFIG_FRAME_WARN=1024` | كشف stack frames كبيرة |
| `CONFIG_NET_DROP_MONITOR=y` | مراقبة الـ packet drops |

---

#### 6. ethtool — الأداة الرئيسية للـ Subsystem

الـ `phylib_stubs` بتخدم الـ ethtool interface مباشرة:

```bash
# اقرا PHY stats (بتمر عبر phylib_stubs->get_phy_stats)
ethtool --phy-statistics eth0

# اقرا link extended stats (بتمر عبر phylib_stubs->get_link_ext_stats)
ethtool --statistics eth0

# اقرا hardware timestamping config (بتمر عبر phylib_stubs->hwtstamp_get)
ethtool -T eth0

# set hardware timestamping
ethtool -T eth0 rx-filter all

# اقرا PHY state كاملة
ethtool eth0

# اقرا التسجيلات من الـ PHY chip
ethtool --dump-module eth0
```

**ethtool -T output:**

```
Time stamping parameters for eth0:
Capabilities:
  hardware-transmit     (SOF_TIMESTAMPING_TX_HARDWARE)
  hardware-receive      (SOF_TIMESTAMPING_RX_HARDWARE)
PTP Hardware Clock: 0
Hardware Transmit Timestamp Modes:
  off                   (HWTSTAMP_TX_OFF)
  on                    (HWTSTAMP_TX_ON)
```

```bash
# تحقق إن الـ RTNL lock شغال صح
# (اي error هنا بيدل إن phylib_stubs مش متسجلة)
ip link show eth0
```

---

#### 7. Common Error Messages

| الـ Error Message | المعنى | الحل |
|-------------------|--------|------|
| `RTNL: assertion failed at phylib_stubs.h (38)` | **ASSERT_RTNL()** فشلت — `phy_hwtstamp_get` اتسمّاها بدون RTNL | تأكد إن الـ caller يحجز `rtnl_lock()` قبل الاستدعاء |
| `ethtool: phy: Operation not supported` | `phylib_stubs == NULL` — الـ `CONFIG_PHYLIB=n` أو module مش loaded | `modprobe phy` أو enable `CONFIG_PHYLIB=y` |
| `MDIO bus: PHY 0:01 not found` | الـ PHY مش متسجل على الـ MDIO bus | تحقق من الـ Device Tree أو module probing |
| `phy0: Could not attach to PHY` | مشكلة في ربط الـ MAC بالـ PHY | تحقق من `phy-handle` في الـ DT |
| `mdio_bus mdio0: MDIO read timeout` | الـ PHY chip مش بيرد | مشكلة hardware — تحقق من الـ power و الـ MDC/MDIO signals |
| `WARNING: CPU: 0 PID: X at include/linux/rtnetlink.h:56` | نفس مشكلة ASSERT_RTNL | راجع call stack بالكامل |
| `phylib: failed to register stubs` | تعارض في تسجيل الـ stubs | تأكد إن module واحد بس بيسجّل الـ stubs |

---

#### 8. WARN_ON() / dump_stack() Strategic Points

```c
/* النقاط الاستراتيجية لإضافة debugging */

/* 1. تحقق من phylib_stubs قبل الاستخدام */
static inline int phy_hwtstamp_get(struct phy_device *phydev,
                                   struct kernel_hwtstamp_config *config)
{
    ASSERT_RTNL();  /* هذا موجود فعلاً — بيعمل WARN_ONCE */

    /* أضف هنا لو محتاج تتبع أكثر */
    if (WARN_ON(!phylib_stubs)) {
        dump_stack();  /* يطبع الـ call trace كاملاً */
        return -EOPNOTSUPP;
    }

    return phylib_stubs->hwtstamp_get(phydev, config);
}

/* 2. تحقق من الـ phydev pointer */
WARN_ON_ONCE(!phydev);

/* 3. في phylib_register_stubs */
WARN_ON(phylib_stubs != NULL); /* تعارض في التسجيل */
```

---

### Hardware Level

#### 1. التحقق إن Hardware State يطابق Kernel State

```bash
# قرا الـ PHY state من الـ kernel
cat /sys/bus/mdio_bus/devices/mdio0\:01/phy_state 2>/dev/null
# أو
ethtool eth0 | grep -E "Speed|Duplex|Link"

# قرا registers الـ PHY مباشرة (Standard MII registers)
# Register 0: Control, Register 1: Status, Register 4-5: Auto-Negotiation
mii-tool -v eth0 2>/dev/null || \
ethtool -d eth0  # raw register dump لو driver يدعمه
```

**مقارنة الـ state:**

```bash
# الـ kernel state
ethtool eth0 | grep "Link detected"
# Link detected: yes

# الـ PHY chip register 1 (Status Register)
# bit 2 = Link Status
# تقدر تقراه عبر mdio-tools
mdio-util mdio0 phy 1 read 1
# Output: 0x786D  → bit 2 = 1 = Link UP
```

---

#### 2. Register Dump Techniques

```bash
# استخدام mdio-tools (أفضل طريقة للـ PHY registers)
# تثبيت: apt install mdio-tools || compile from source

# قرا register 0 (Control Register) من PHY address 1
mdio-util <mdio-bus-name> phy <phy-addr> read <reg>

# مثال: قرا كل الـ standard registers
for reg in 0 1 2 3 4 5 6 7; do
    echo -n "REG $reg: "
    mdio-util mdio0 phy 1 read $reg
done
```

**Standard MII Registers:**

| Register | الاسم | البتات المهمة |
|----------|-------|---------------|
| 0 | Control | bit 15=Reset, bit 12=AutoNeg Enable, bit 9=Restart AutoNeg |
| 1 | Status | bit 2=Link, bit 5=AutoNeg Complete |
| 2-3 | PHY Identifier | OUI + model number |
| 4 | AutoNeg Advertisement | ما يعلنه الـ PHY |
| 5 | AutoNeg Link Partner | ما يعلنه الطرف الآخر |

```bash
# لو ما فيه mdio-tools، استخدم /dev/mem (requires root + CONFIG_DEVMEM=y)
# اعرف physical address للـ MDIO controller من الـ DT أو datasheet
devmem2 0xFE540000 w  # قرا MDIO base register
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

الـ MDC/MDIO interface بيشتغل على:
- **MDC**: clock، عادة ≤ 2.5 MHz
- **MDIO**: data، bidirectional، open-drain

```
MDIO Frame Structure:
 PRE  ST  OP    PHYAD  REGAD  TA    DATA
[32×1][01][RD=10][5bit][5bit][Z0/10][16bit]
      [01][WR=01][5bit][5bit][10   ][16bit]
```

**نقاط القياس:**
- تحقق إن MDC clock منتظم ومش بيوقف
- الـ MDIO bus turnaround (`TA`) لازم يكون `Z` ثم `0` في الـ read
- timeout علامة إن الـ PHY مش بيرد أو في contention على الـ bus

```bash
# تفعيل MDIO bus tracing لتوليد traffic للقياس
echo 1 > /sys/kernel/debug/tracing/events/mdio/enable
ethtool eth0  # هذا بيعمل MDIO reads
```

---

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة | Pattern في dmesg | التشخيص |
|---------|-----------------|---------|
| PHY مش موجود | `PHY 0:01 not found` | تحقق من الـ power supply للـ PHY chip |
| MDIO timeout | `MDIO read/write timeout` | تحقق MDC clock، قد يكون الـ pull-up مفقود |
| Link instability | `eth0: Link is Down` متكررة | تحقق من كابل الـ Ethernet أو الـ magnetics |
| AutoNeg failure | `eth0: link up, 10Mbps` (بدل 1G) | جرب `ethtool -s eth0 autoneg off speed 1000` |
| HW timestamp failure | `timestamping not supported` | الـ PHY لا يدعم PTP أو الـ driver مش محدّث |

```bash
# مراقبة live
dmesg -w | grep -E "(mdio|phy|eth[0-9]|phylib)"
```

---

#### 5. Device Tree Debugging

```bash
# اقرا الـ DT المطبّق حالياً
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "mdio"

# تحقق من الـ phy-handle
cat /sys/firmware/devicetree/base/soc/ethernet@*/phy-handle 2>/dev/null | xxd

# تحقق من compatible string للـ PHY
find /sys/firmware/devicetree/base -name "compatible" -exec grep -l "ethernet-phy" {} \;

# قرا الـ phy address من الـ DT
cat /sys/firmware/devicetree/base/soc/mdio@*/phy@*/reg 2>/dev/null | xxd
```

**DT صحيح للـ PHY:**

```dts
/* مثال Device Tree node صحيح */
&mdio0 {
    phy0: ethernet-phy@1 {
        compatible = "ethernet-phy-ieee802.3-c22";
        reg = <1>;                /* PHY address على الـ MDIO bus */
        /* hwtstamp يحتاج هذا: */
        timestamping;
    };
};

&eth0 {
    phy-handle = <&phy0>;
    phy-mode = "rgmii-id";
};
```

```bash
# تحقق إن الـ phy-mode صح
cat /sys/class/net/eth0/phy_mode 2>/dev/null
# أو
ethtool -i eth0 | grep "bus-info"
```

---

### Practical Commands — Ready to Copy

#### فحص شامل للـ phylib_stubs

```bash
#!/bin/bash
# =============================================================
# phylib_stubs diagnostic script
# =============================================================

IFACE=${1:-eth0}

echo "=== PHY Interface: $IFACE ==="

# 1. تحقق إن PHYLIB module موجود
echo "[1] PHYLIB module status:"
lsmod | grep -E "phy$|phylib" || echo "  WARNING: phy module not loaded"

# 2. تحقق من phylib_stubs pointer
echo "[2] phylib_stubs symbol:"
grep phylib_stubs /proc/kallsyms | head -3

# 3. PHY device info
echo "[3] PHY device:"
ethtool $IFACE 2>/dev/null | grep -E "Speed|Duplex|Link|Port"

# 4. Hardware timestamp support (مرتبط مباشرة بالـ stubs)
echo "[4] HW Timestamping:"
ethtool -T $IFACE 2>&1

# 5. PHY statistics (عبر phylib_stubs->get_phy_stats)
echo "[5] PHY statistics:"
ethtool --phy-statistics $IFACE 2>&1 | head -20

# 6. Link extended stats (عبر phylib_stubs->get_link_ext_stats)
echo "[6] Link extended stats:"
ethtool --statistics $IFACE 2>&1 | grep -E "phy|link|error" | head -20

# 7. MDIO bus devices
echo "[7] MDIO bus devices:"
ls /sys/bus/mdio_bus/devices/ 2>/dev/null

# 8. kernel messages
echo "[8] Recent kernel PHY messages:"
dmesg | grep -E "(phy|mdio|phylib|RTNL)" | tail -20
```

#### تفعيل Full Tracing

```bash
#!/bin/bash
# فعّل كل الـ tracing المفيد للـ phylib_stubs debugging

# 1. mdio events
echo 1 > /sys/kernel/debug/tracing/events/mdio/enable

# 2. net events
echo 1 > /sys/kernel/debug/tracing/events/net/enable 2>/dev/null

# 3. function tracing للـ phy functions
echo "phy_hwtstamp_get phy_hwtstamp_set phy_ethtool_get_phy_stats phy_ethtool_get_link_ext_stats" > \
    /sys/kernel/debug/tracing/set_ftrace_filter

echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# 4. شغّل trigger
ethtool -T eth0
ethtool --phy-statistics eth0

# 5. اقرا النتائج
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace | grep -v "^#"

# 6. cleanup
echo nop > /sys/kernel/debug/tracing/current_tracer
echo > /sys/kernel/debug/tracing/set_ftrace_filter
```

#### تشخيص RTNL Lock Issues

```bash
# تحقق من lockdep warnings
dmesg | grep -E "RTNL|lockdep|deadlock|lock order" | tail -30

# لو CONFIG_PROVE_LOCKING=y، هتشوف:
# [  123.456] WARNING: CPU: 0 PID: 1234 at include/linux/rtnetlink.h:56
# [  123.456] RTNL: assertion failed at include/linux/phylib_stubs.h (38)

# تحقق من حالة الـ RTNL lock في /proc
cat /proc/locks | grep -i rtnl 2>/dev/null

# استخدم ss لشوف الـ netlink sockets
ss -nlp | grep rtnl
```

#### فحص الـ MDIO Registers يدوياً

```bash
#!/bin/bash
# قرا standard PHY registers
# المتطلبات: mdio-tools أو phytool

PHY_ADDR=1
MDIO_BUS="mdio0"

echo "=== PHY Register Dump (PHY addr: $PHY_ADDR) ==="
echo ""
echo "Control Register (0x00):"
phytool read $MDIO_BUS/$PHY_ADDR/0 2>/dev/null || \
    mdio-util $MDIO_BUS phy $PHY_ADDR read 0 2>/dev/null

echo "Status Register (0x01) — Link bit[2]:"
phytool read $MDIO_BUS/$PHY_ADDR/1 2>/dev/null

echo "PHY ID1 (0x02):"
phytool read $MDIO_BUS/$PHY_ADDR/2 2>/dev/null

echo "PHY ID2 (0x03):"
phytool read $MDIO_BUS/$PHY_ADDR/3 2>/dev/null
```

#### تحقق سريع من phylib_stubs Registration

```bash
# الأسرع لتشخيص "Operation not supported" من الـ stubs
python3 - <<'EOF'
import subprocess, re

# تحقق من الـ module
r = subprocess.run(['lsmod'], capture_output=True, text=True)
phy_loaded = any('phy' in line.split()[0] for line in r.stdout.splitlines())
print(f"PHY module loaded: {phy_loaded}")

# تحقق من الـ symbol
r = subprocess.run(['grep', 'phylib_stubs', '/proc/kallsyms'],
                   capture_output=True, text=True)
stubs_present = bool(r.stdout.strip())
print(f"phylib_stubs symbol: {'PRESENT' if stubs_present else 'MISSING'}")

if not phy_loaded:
    print("FIX: run 'modprobe phy' or enable CONFIG_PHYLIB=y")
if not stubs_present:
    print("FIX: phylib_register_stubs() not called — check phy module init")
EOF
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على AM62x — `CONFIG_PHYLIB` مش موجود في الـ config

#### العنوان
الـ `ethtool -T` بيرجع `EOPNOTSUPP` على gateway صناعي رغم إن الـ PHY بيدعم hardware timestamping

#### السياق
شركة بتبني **industrial IoT gateway** على **Texas Instruments AM62x** بيشتغل كـ PROFINET controller. الـ Ethernet PHY هو **DP83867** اللي بيدعم hardware timestamping (PTP/IEEE 1588). الـ product بيحتاج دقة تزامن أقل من microsecond عشان الـ real-time fieldbus.

#### المشكلة
المهندس بيشغّل الأمر ده:

```bash
ethtool -T eth0
```

النتيجة:
```
Time stamping parameters for eth0:
Capabilities:
        software-transmit
        software-receive
        software-system-clock
PTP Hardware Clock: none
Hardware Transmit Timestamp Modes: none
Hardware Receive Filter Modes: none
```

**الـ hardware timestamping مش شغّال خالص**. الـ `ethtool -T` مش بتظهر أي hardware capability.

#### التحليل
المهندس يدور في الكود — الـ flow بيمشي كده:

```c
/* ethtool core بيكلّم phy_hwtstamp_get() */
static inline int phy_hwtstamp_get(struct phy_device *phydev,
                                   struct kernel_hwtstamp_config *config)
{
    ASSERT_RTNL();

    if (!phylib_stubs)       /* <-- هنا المشكلة */
        return -EOPNOTSUPP;

    return phylib_stubs->hwtstamp_get(phydev, config);
}
```

**الـ `phylib_stubs` pointer بيساوي `NULL`** لأن `CONFIG_PHYLIB=n` في الـ kernel config. لما الـ `#else` branch بياخد:

```c
/* الـ stub الفاضي لما PHYLIB مش موجود */
static inline int phy_hwtstamp_get(struct phy_device *phydev,
                                   struct kernel_hwtstamp_config *config)
{
    return -EOPNOTSUPP;   /* دايماً برجع error */
}
```

الـ ethtool layer بيترجم الـ `-EOPNOTSUPP` إنه "no hardware capability".

المهندس يتأكد:

```bash
grep CONFIG_PHYLIB /boot/config-$(uname -r)
# CONFIG_PHYLIB is not set
```

تأكيد المشكلة.

#### الحل

```bash
# 1. فعّل PHYLIB في الـ kernel config
make menuconfig
# Device Drivers -> Network device support -> PHY Device support and infrastructure
# [*] PHY Device support and infrastructure (PHYLIB)
# [*]   Texas Instruments DP83867 PHY support

# 2. أعد بناء الكيرنل
make -j$(nproc) Image modules

# 3. تحقق بعد التفعيل
grep CONFIG_PHYLIB /boot/config-$(uname -r)
# CONFIG_PHYLIB=y

# 4. اختبر
ethtool -T eth0
# Time stamping parameters for eth0:
# Capabilities:
#         hardware-transmit
#         hardware-receive
#         hardware-raw-clock
# PTP Hardware Clock: 0
```

#### الدرس المستفاد
الـ `phylib_stubs.h` بيعمل **compile-time abstraction** — لما `CONFIG_PHYLIB=n`، كل دوال الـ PHY بتتحول لـ stubs فاضية بترجع `-EOPNOTSUPP` أو لا ترجع أي حاجة. **مفيش runtime error، مفيش kernel panic** — بس الـ feature مش شغّالة بصمت. لازم دايماً تتحقق من الـ Kconfig قبل ما تحكم إن الـ hardware عاطل.

---

### السيناريو 2: Android TV Box على Allwinner H616 — Race Condition في الـ `phylib_stubs`

#### العنوان
kernel warning عشوائي `RTNL: assertion failed` على TV box بيحصل عند الـ boot

#### السياق
**Android TV box** بيستخدم **Allwinner H616** مع **Realtek RTL8211F** PHY على Ethernet. الـ driver custom بيكلّم `phy_hwtstamp_get()` من context غلط.

#### المشكلة
الـ dmesg بيظهر:

```
WARNING: CPU: 2 PID: 847 at include/linux/rtnetlink.h:56
RTNL: assertion failed at include/linux/phylib_stubs.h (38)
Call Trace:
 phy_hwtstamp_get+0x1c/0x50
 custom_eth_ioctl+0x88/0x120 [h616_gmac]
 dev_ioctl+0x1a4/0x3c0
```

الـ warning بيحصل عشوائي عند الـ boot أو عند connect/disconnect الـ Ethernet cable.

#### التحليل
الـ `phy_hwtstamp_get()` في الـ file بيعمل:

```c
static inline int phy_hwtstamp_get(struct phy_device *phydev,
                                   struct kernel_hwtstamp_config *config)
{
    /* phylib_register_stubs() and phylib_unregister_stubs()
     * also run under rtnl_lock().
     */
    ASSERT_RTNL();   /* <-- بيتحقق إن rtnl_lock محجوز */

    if (!phylib_stubs)
        return -EOPNOTSUPP;

    return phylib_stubs->hwtstamp_get(phydev, config);
}
```

**الـ comment في الكود بيشرح السبب**: الـ `phylib_stubs` pointer ممكن يتغير في أي وقت لو `phylib_register_stubs()` أو `phylib_unregister_stubs()` اتشغّلوا. الـ لازم محجوزة قبل ما تقرأ الـ pointer ده.

الـ driver الـ custom بيعمل الغلط ده:

```c
/* custom_eth_ioctl في driver الـ H616 — كود غلط */
static int custom_eth_ioctl(struct net_device *ndev, struct ifreq *ifr, int cmd)
{
    struct phy_device *phydev = ndev->phydev;

    if (cmd == SIOCGHWTSTAMP) {
        struct kernel_hwtstamp_config config;
        /* المشكلة هنا: بيكلّم phy_hwtstamp_get بدون rtnl_lock */
        return phy_hwtstamp_get(phydev, &config);
    }
    return -EOPNOTSUPP;
}
```

#### الحل

```c
/* الكود الصح: لازم rtnl_lock قبل الاستدعاء */
static int custom_eth_ioctl(struct net_device *ndev, struct ifreq *ifr, int cmd)
{
    struct phy_device *phydev = ndev->phydev;
    int ret;

    if (cmd == SIOCGHWTSTAMP) {
        struct kernel_hwtstamp_config config;

        rtnl_lock();                              /* احجز الـ RTNL */
        ret = phy_hwtstamp_get(phydev, &config);  /* ASSERT_RTNL() هتنجح */
        rtnl_unlock();                            /* حرر الـ RTNL */

        if (ret)
            return ret;

        return copy_to_user(ifr->ifr_data, &config, sizeof(config))
               ? -EFAULT : 0;
    }
    return -EOPNOTSUPP;
}
```

```bash
# تحقق من الـ fix بـ lockdep
echo 1 > /proc/sys/kernel/prove_locking
# شغّل الاختبار وشوف لو في warnings
dmesg | grep -i "rtnl\|assertion"
```

#### الدرس المستفاد
الـ `ASSERT_RTNL()` في `phylib_stubs.h` مش مجرد debugging check — هو **contract** بيقول إن الـ function دي لازم تتنادى تحت `rtnl_lock`. السبب إن `phylib_stubs` pointer نفسه ممكن يتغير أثناء `phylib_register_stubs()` و `phylib_unregister_stubs()`. الـ comment في الكود بيقول كده صراحة.

---

### السيناريو 3: Automotive ECU على i.MX8 — الـ `get_phy_stats` بيرجع صفر دايماً

#### العنوان
monitoring system للـ automotive ECU مش بيشوف أي PHY errors رغم وجود noise على الكابل

#### السياق
**automotive ECU** على **NXP i.MX8M Plus** مع **NXP TJA1120** PHY (100BASE-T1 للـ in-vehicle networking). الـ quality assurance system بيستخدم `ethtool --phy-statistics` عشان يراقب الـ cable quality في خط التجميع.

#### المشكلة
```bash
ethtool --phy-statistics eth0
# NIC statistics:
#      phy_receive_errors: 0
#      phy_false_carrier: 0
# (كل الأرقام صفر حتى لو في noise على الكابل)
```

الـ oscilloscope بيظهر signal degradation واضح، بس الـ kernel ما بيشوفش أي errors.

#### التحليل
الـ call chain بتمشي كده:

```c
/* ethtool core بيكلّم */
static inline void phy_ethtool_get_phy_stats(struct phy_device *phydev,
                    struct ethtool_eth_phy_stats *phy_stats,
                    struct ethtool_phy_stats *phydev_stats)
{
    ASSERT_RTNL();

    if (!phylib_stubs)   /* لو phylib_stubs = NULL، بيرجع بدون عمل حاجة */
        return;

    phylib_stubs->get_phy_stats(phydev, phy_stats, phydev_stats);
}
```

لو `phylib_stubs->get_phy_stats` نفسه `NULL`؟ ده crash مؤكد. المهندس يفحص:

```bash
# فحص الـ phylib driver المحمّل
lsmod | grep phylib
# phylib               98304  4 tja1120,nxp_c45_tja11xx,fixed_phy,of_mdio

# فحص لو TJA1120 driver بيعمل implement الـ get_stats
grep -r "get_phy_stats\|phydev_stats" /sys/kernel/debug/
```

المشكلة إن `TJA1120` driver القديم ما بيعملش implement `get_phy_stats` في الـ `phy_driver` struct، فالـ `phylib`'s implementation بتكلّم `NULL` function pointer بالـ flow التالي:

```
phylib_stubs->get_phy_stats()
    -> phylib's internal get_phy_stats()
        -> phydev->drv->get_stats()  /* NULL لو الـ driver مش بيدعمه */
```

#### الحل

```c
/* في driver الـ TJA1120 — إضافة دعم الإحصائيات */
static void tja1120_get_phy_stats(struct phy_device *phydev,
                                   struct ethtool_phy_stats *stats)
{
    int val;

    /* اقرأ SQI (Signal Quality Indicator) register */
    val = phy_read_mmd(phydev, MDIO_MMD_VEND1, TJA1120_SQI_REG);
    if (val < 0)
        return;

    stats->stats[ETHTOOL_PHY_STAT_SQI] = val & TJA1120_SQI_MASK;

    /* اقرأ الـ receive error counter */
    val = phy_read_mmd(phydev, MDIO_MMD_VEND1, TJA1120_RX_ERR_CNT);
    if (val >= 0)
        stats->stats[ETHTOOL_PHY_STAT_RX_ERR] = val;
}

static struct phy_driver tja1120_driver = {
    /* ... */
    .get_stats      = tja1120_get_phy_stats,
    /* ... */
};
```

```bash
# بعد الـ fix
ethtool --phy-statistics eth0
# NIC statistics:
#      phy_receive_errors: 147
#      phy_sqi: 6
```

#### الدرس المستفاد
الـ `phylib_stubs.h` بيوفر **glue layer** بين الـ ethtool API وبين الـ phylib. لكن لو الـ PHY driver نفسه ما بيعملش implement الـ callbacks، الـ stubs هتنادي على NULL أو هتعمل no-op بصمت. لازم تتحقق من الـ chain كاملة: `phylib_stubs` → `phylib internal` → `phy_driver->get_stats`.

---

### السيناريو 4: Custom Board Bring-Up على RK3562 — الـ `phylib_stubs` بيترجّع `NULL` بعد `rmmod`

#### العنوان
kernel panic عند إعادة تحميل الـ PHY driver على RK3562 development board

#### السياق
مهندس بيعمل **bring-up** لـ custom board على **Rockchip RK3562** مع **Motorcomm YT8511** PHY. الـ workflow بيعمل `modprobe`/`rmmod` على الـ PHY driver كتير عشان debugging.

#### المشكلة
```
[ 1234.567890] BUG: kernel NULL pointer dereference, address: 0000000000000000
[ 1234.567891] #PF: supervisor read access in kernel mode
[ 1234.567892] Call Trace:
[ 1234.567893]  phy_hwtstamp_set+0x3c/0x58
[ 1234.567894]  ethnl_set_tsinfo+0x1a8/0x2c0
[ 1234.567895]  genl_family_rcv_msg_doit+0xd0/0x110
```

#### التحليل
الـ sequence بتمشي كده:

```
1. modprobe phylib              → phylib_register_stubs() بيشغّل
                                 → phylib_stubs = &real_stubs  (valid)

2. modprobe yt8511              → PHY driver محمّل

3. ethtool -T eth0 تشتغل       → phy_hwtstamp_set() بتشتغل تمام

4. rmmod phylib                 → phylib_unregister_stubs() بيشتغل
                                 → phylib_stubs = NULL

5. ethtool -T eth0 تشتغل تاني  → ???
```

لو الـ `ASSERT_RTNL()` مش موجود أو disabled، الـ code بيمشي:

```c
static inline int phy_hwtstamp_set(struct phy_device *phydev,
                                   struct kernel_hwtstamp_config *config,
                                   struct netlink_ext_ack *extack)
{
    ASSERT_RTNL();  /* بيطلع warning بس مش بيوقّف */

    if (!phylib_stubs)          /* المفروض هنا يوقف */
        return -EOPNOTSUPP;

    return phylib_stubs->hwtstamp_set(phydev, config, extack);
    /*                  ^^^^^^^^^^^^^ لو اتغيّر بين الفحص والاستدعاء = TOCTOU */
}
```

الـ race condition: بين `if (!phylib_stubs)` وبين `phylib_stubs->hwtstamp_set()`، ممكن `phylib_unregister_stubs()` يشتغل على CPU تاني ويعمل `phylib_stubs = NULL`. **ده لو مفيش RTNL lock!**

الحل في الكود نفسه: الـ `rtnl_lock` بيحمي من الـ race لأن `phylib_unregister_stubs()` كمان بيشتغل تحت `rtnl_lock`.

المهندس اكتشف إن الـ custom ethtool handler بتاعه بيعمل:

```c
/* كود غلط في custom handler */
void my_handler(void) {
    /* بيكلّم phy_hwtstamp_set بدون rtnl_lock */
    phy_hwtstamp_set(phydev, &config, NULL);
}
```

#### الحل

```c
/* الكود الصح */
void my_handler(void) {
    int ret;

    rtnl_lock();
    ret = phy_hwtstamp_set(phydev, &config, NULL);
    rtnl_unlock();

    if (ret == -EOPNOTSUPP) {
        pr_info("PHY timestamping not supported\n");
    }
}
```

```bash
# للتأكد إن phylib loaded قبل أي operation
cat /sys/bus/mdio_bus/devices/*/phy_id
# لو الملف موجود، الـ phylib شغّال

# تحقق من الـ stubs registration
cat /proc/kallsyms | grep phylib_stubs
```

#### الدرس المستفاد
الـ `phylib_stubs.h` بيعتمد على `rtnl_lock` كـ **synchronization primitive** بين الـ stubs pointer وبين التعديل عليه. الـ `ASSERT_RTNL()` في كل function مش warning زيادة — هو **mandatory locking requirement**. الـ race window بين `if (!phylib_stubs)` وبين الاستدعاء الفعلي ممكن يسبب NULL dereference لو اللاك مش محجوز.

---

### السيناريو 5: IoT Sensor على STM32MP1 — تقليل الـ kernel size بحذف PHYLIB وكسر الـ ethtool stats

#### العنوان
شركة embedded بتحاول تقليل الـ kernel size على STM32MP1 فتحذف `CONFIG_PHYLIB` وتكتشف إن الـ ethtool stats كسرت كلها

#### السياق
**IoT environmental sensor** على **STMicroelectronics STM32MP1** مع **LAN8720** PHY. الـ team بتعمل **size optimization** للـ production image عشان يتناسب مع 32MB flash. الـ network stack محتاجة بس basic connectivity — مفيش PTP، مفيش advanced stats.

#### المشكلة
بعد الـ optimization وحذف `CONFIG_PHYLIB`:

```bash
ethtool eth0
# Settings for eth0:
#         Link detected: yes
#         Speed: 100Mb/s
#         ...
# (تمام)

ethtool --phy-statistics eth0
# Cannot get PHY statistics: Operation not supported

ethtool -T eth0
# Cannot get time stamping information: Operation not supported

ethtool -S eth0
# no stats available
```

الـ monitoring dashboard اللي بيقرأ PHY error counters عشان يعرف لو في signal degradation بدأ يطلع alerts كاذبة لأنه بيفسّر الـ `-EOPNOTSUPP` كـ network failure.

#### التحليل
لما `CONFIG_PHYLIB=n`، الـ `#else` branch في `phylib_stubs.h` يشتغل:

```c
#else
/* كل الدوال بتبقى stubs فاضية */

static inline int phy_hwtstamp_get(struct phy_device *phydev,
                                   struct kernel_hwtstamp_config *config)
{
    return -EOPNOTSUPP;   /* دايماً */
}

static inline int phy_hwtstamp_set(struct phy_device *phydev,
                                   struct kernel_hwtstamp_config *config,
                                   struct netlink_ext_ack *extack)
{
    return -EOPNOTSUPP;   /* دايماً */
}

static inline void phy_ethtool_get_phy_stats(struct phy_device *phydev,
                    struct ethtool_eth_phy_stats *phy_stats,
                    struct ethtool_phy_stats *phydev_stats)
{
    /* no-op — مش بيعمل حاجة خالص */
}

static inline void phy_ethtool_get_link_ext_stats(struct phy_device *phydev,
                    struct ethtool_link_ext_stats *link_stats)
{
    /* no-op */
}

#endif
```

الـ design ده عن قصد — بيسمح للـ MAC driver إنه يعمل compile بدون PHYLIB، بس كل الـ PHY-specific features بتختفي بصمت.

**حجم الوفر من حذف PHYLIB:**

```bash
# قبل الحذف
size vmlinux | grep -i text
# text: 8,234,112

# بعد حذف CONFIG_PHYLIB
size vmlinux | grep -i text
# text: 7,891,456
# وفر ~340KB من الـ text segment
```

#### الحل

**Option A**: خلّي `CONFIG_PHYLIB=y` واحذف حاجات تانية أقل أهمية:

```bash
# حاجات ممكن تتحذف من STM32MP1 IoT image
# CONFIG_BLUETOOTH=n
# CONFIG_WIRELESS=n
# CONFIG_SOUND=n
# CONFIG_USB_GADGET=n
# بدل ما تحذف PHYLIB
```

**Option B**: لو الـ monitoring dashboard تحت سيطرتك، عالج الـ `-EOPNOTSUPP` صح:

```python
# monitoring script
import subprocess
import json

def get_phy_stats(iface):
    result = subprocess.run(
        ['ethtool', '--phy-statistics', iface],
        capture_output=True, text=True
    )
    if 'Operation not supported' in result.stderr:
        # PHYLIB مش موجود — مش error، feature غير متاحة
        return None, "phylib_disabled"
    return parse_stats(result.stdout), "ok"
```

**Option C**: لو محتاج basic PHY error counting بدون PHYLIB، اعمل custom MDIO reads مباشرة:

```c
/* custom PHY stats بدون phylib */
static void stm32_read_phy_errors(struct net_device *ndev)
{
    struct stm32_priv *priv = netdev_priv(ndev);

    /* اقرأ PHY register مباشرة عبر MDIO */
    int err_reg = mdiobus_read(priv->mii, priv->phy_addr,
                               LAN8720_SYMBLERR_CNT);
    if (err_reg > 0)
        priv->phy_errors += err_reg;
}
```

#### الدرس المستفاد
الـ `phylib_stubs.h` بيعمل **graceful degradation** — لما `CONFIG_PHYLIB=n`، الكود بيكمبايل ومفيش crash، بس كل الـ PHY-level features بتختفي وبترجع `-EOPNOTSUPP` أو no-op. هذا الـ design الصح لـ embedded systems، لكن لازم الـ userspace tools والـ monitoring systems تكون aware بالفرق بين "feature not supported by this hardware" و"feature disabled in kernel config". الوثيقة الـ proper لأي production image لازم تذكر صراحة أي `CONFIG_` flags اتحذفت وإيه تأثيرها على الـ functionality.
## Phase 7: مصادر ومراجع

### التوثيق الرسمي للـ Kernel

| المصدر | الرابط | الأهمية |
|--------|--------|----------|
| PHY Abstraction Layer — Kernel Docs | [docs.kernel.org/networking/phy.html](https://docs.kernel.org/networking/phy.html) | المرجع الأساسي لكل phylib |
| phylink & SFP — Kernel Docs | [docs.kernel.org/networking/sfp-phylink.html](https://docs.kernel.org/networking/sfp-phylink.html) | شرح الانتقال من phylib إلى phylink |
| PHY Link Topology — Kernel Docs | [docs.kernel.org/networking/phy-link-topology.html](https://docs.kernel.org/networking/phy-link-topology.html) | topology الشبكة والـ PHY |
| Netlink Interface for ethtool | [docs.kernel.org/networking/ethtool-netlink.html](https://docs.kernel.org/networking/ethtool-netlink.html) | واجهة netlink للـ ethtool وإحصاءات الـ PHY |
| Interface Statistics | [docs.kernel.org/networking/statistics.html](https://docs.kernel.org/networking/statistics.html) | إحصاءات الـ PHY عبر ethtool |

مسارات التوثيق داخل شجرة الـ kernel:

```
Documentation/networking/phy.rst
Documentation/networking/sfp-phylink.rst
Documentation/networking/phy-link-topology.rst
Documentation/networking/ethtool-netlink.rst
```

---

### مقالات LWN.net

الـ **LWN.net** هو المرجع التاريخي الأهم لمتابعة تطور phylib في الـ kernel.

| المقال | الرابط |
|--------|--------|
| RFC: PHY Abstraction Layer II — النقاش الأصلي لطبقة PHY | [lwn.net/Articles/127013/](https://lwn.net/Articles/127013/) |
| PHY Abstraction Layer III — الجولة الثالثة من النقاش | [lwn.net/Articles/144897/](https://lwn.net/Articles/144897/) |
| Phylink & SFP support — إضافة دعم phylink وSFP | [lwn.net/Articles/667055/](https://lwn.net/Articles/667055/) |
| phylib: Add support for MDIO clause 45 — دعم clause 45 | [lwn.net/Articles/384726/](https://lwn.net/Articles/384726/) |
| net: Make timestamping selectable — **الأهم لفهم phylib_stubs** | [lwn.net/Articles/947146/](https://lwn.net/Articles/947146/) |
| Add basic LED support for switch/phy | [lwn.net/Articles/927459/](https://lwn.net/Articles/927459/) |
| PAL: Support of the fixed PHY | [lwn.net/Articles/188485/](https://lwn.net/Articles/188485/) |
| PHY Abstraction Layer — كلاسيك توثيق LWN | [static.lwn.net/kerneldoc/networking/phy.html](https://static.lwn.net/kerneldoc/networking/phy.html) |

**الأهم** هو مقال [net: Make timestamping selectable](https://lwn.net/Articles/947146/) لأن patch series الخاص بـ `phylib_stubs` جاء مباشرةً من هذا السياق — الحاجة لتمرير `hwtstamp_get`/`hwtstamp_set` عبر stub آمن عندما `CONFIG_PHYLIB=m`.

---

### Patch Series والـ Mailing List

الـ `phylib_stubs.h` ظهر كجزء من جهود فصل الـ timestamping عن الـ PHY layer وجعله قابلاً للاختيار. أهم الـ threads:

| الموضوع | الرابط |
|---------|--------|
| net: Make timestamping selectable — v7 على lore.kernel.org | [lore.kernel.org — v7 00/16](https://lore.kernel.org/lkml/20231114-feature_ptp_netnext-v7-0-472e77951e40@bootlin.com/T/) |
| net: Make timestamping selectable — v12 على Patchwork | [patchwork.kernel.org — v12](https://patchwork.kernel.org/project/netdevbpf/cover/20240430-feature_ptp_netnext-v12-0-2c5f24b6a914@bootlin.com/) |
| ethtool tsinfo: Add support for reading tsinfo for hwtstamp provider | [patchwork.kernel.org — v17 12/14](https://patchwork.kernel.org/project/netdevbpf/patch/20240709-feature_ptp_netnext-v17-12-b5317f50df2a@bootlin.com/) |
| LKML: null-ptr-deref in generic_hwtstamp_ioctl_lower | [lkml.org/lkml/2025/10/29/1749](https://lkml.org/lkml/2025/10/29/1749) |
| PATCH RFC: net: phy: improve phylib aspects | [spinics.net/lists/netdev/msg489254.html](https://www.spinics.net/lists/netdev/msg489254.html) |

للبحث في أرشيف الـ mailing list مباشرةً:
```
https://lore.kernel.org/netdev/?q=phylib_stubs
https://www.spinics.net/lists/netdev/
```

---

### ملفات السورس الأساسية في الـ Kernel

الملفات دي هي المرجع المباشر لفهم `phylib_stubs.h` من جوه الـ kernel source tree:

```
include/linux/phylib_stubs.h       ← الملف الموثَّق نفسه
include/linux/phy.h                ← تعريف struct phy_device الكاملة
drivers/net/phy/phy_device.c       ← التسجيل: phylib_register_stubs()
drivers/net/phy/phy.c              ← منطق الـ PHY state machine
include/linux/rtnetlink.h          ← ASSERT_RTNL() المستخدم في الـ stubs
include/linux/ethtool.h            ← تعريفات ethtool_eth_phy_stats وغيرها
```

على GitHub:
- [torvalds/linux — include/linux/phy.h](https://github.com/torvalds/linux/blob/master/include/linux/phy.h)
- [torvalds/linux — drivers/net/phy/phy_device.c](https://github.com/torvalds/linux/blob/master/drivers/net/phy/phy_device.c)

---

### CONFIG_PHYLIB — معلومات الـ Kconfig

| الرابط | المحتوى |
|--------|---------|
| [cateee.net/lkddb — CONFIG_PHYLIB](https://cateee.net/lkddb/web-lkddb/PHYLIB.html) | قاعدة بيانات الـ kernel drivers لـ CONFIG_PHYLIB |
| [kernelconfig.io/config_phylib](https://www.kernelconfig.io/config_phylib) | تبعيات وخيارات الـ Kconfig للـ PHYLIB |

---

### Kernelnewbies.org

| الصفحة | الرابط |
|--------|--------|
| Linux 2.6.29 — أول دعم PHYLIB لـ IXP4xx Ethernet | [kernelnewbies.org/Linux_2_6_29](https://kernelnewbies.org/Linux_2_6_29) |
| Linux 6.15 — تحسينات phylib: reset randomization + adjustable polling | [kernelnewbies.org/Linux_6.15](https://kernelnewbies.org/Linux_6.15) |
| Linux 6.7 — تغييرات networking وphylib | [kernelnewbies.org/Linux_6.7](https://kernelnewbies.org/Linux_6.7) |
| Linux 6.18 — آخر تحديثات phylib | [kernelnewbies.org/Linux_6.18](https://kernelnewbies.org/Linux_6.18) |

---

### كتب مُوصى بيها

#### Linux Device Drivers, 3rd Edition (LDD3)
- المؤلفون: Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصل 17**: Network Drivers — يشرح بنية network driver ومفهوم abstraction layer
- متاح مجاناً: [lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)
- **ملاحظة**: LDD3 قديم نسبياً، phylib جاء بعده، لكن مبادئ الـ network driver ثابتة

#### Linux Kernel Development, 3rd Edition — Robert Love
- **الفصل 14**: The Block I/O Layer و**الفصل 15**: The Process Address Space
- الأهم هنا: فهم locking وrtnl_lock اللي بيستخدمه `phylib_stubs.h`
- يشرح مفهوم الـ module stubs وكيف الـ kernel بيتعامل مع optional subsystems

#### Embedded Linux Primer — Christopher Hallinan
- **الفصل 15**: Debugging Embedded Linux — تشمل debugging الـ PHY issues
- يغطي كيفية configure الـ network PHY في embedded systems

#### Understanding the Linux Kernel — Bovet & Cesati
- الفصول المتعلقة بـ module loading وsymbol resolution مهمة لفهم نمط الـ stubs

---

### مصطلحات بحث مفيدة

للبحث في Google أو lore.kernel.org أو LWN.net:

```
phylib_stubs hwtstamp kernel
phylib_register_stubs phy_device
CONFIG_PHYLIB=m stub pattern linux
phy_hwtstamp_get phylib kernel
ethtool_eth_phy_stats phy_device
kernel_hwtstamp_config phylib
ASSERT_RTNL rtnl_lock phy
phylink vs phylib linux kernel
linux kernel optional module stub pattern
netdev phylib timestamping selectable
```

---

### ملخص أولويات القراءة

| الأولوية | المصدر | السبب |
|----------|--------|-------|
| 1 | [docs.kernel.org/networking/phy.html](https://docs.kernel.org/networking/phy.html) | الأساس — لازم تقرأه الأول |
| 2 | [lwn.net/Articles/947146/](https://lwn.net/Articles/947146/) | السياق المباشر لـ phylib_stubs |
| 3 | `include/linux/phy.h` في الـ kernel source | لفهم struct phy_device |
| 4 | [lore.kernel.org — v7 patch series](https://lore.kernel.org/lkml/20231114-feature_ptp_netnext-v7-0-472e77951e40@bootlin.com/T/) | النقاش التقني الكامل |
| 5 | [docs.kernel.org/networking/sfp-phylink.html](https://docs.kernel.org/networking/sfp-phylink.html) | فهم العلاقة بين phylib وphylink |
## Phase 8: Writing simple module

### الفكرة

**الـ** `phylib_stubs` هو pointer عالمي بيحتفظ بـ vtable من نوع `struct phylib_stubs` — بيتسجل لما `phylib` يتحمّل، وبييجي `NULL` لما يتحذف. الفكرة هي استخدام **kprobe** على الـ `phy_ethtool_get_phy_stats` لأنه:

- بييتنادى كل ما `ethtool -S ethX` اتشغّل على interface فيه PHY.
- بيمرّر `struct phy_device *` كأول argument — فنقدر نطبع بيانات مفيدة زي الـ `phy_id` والـ `speed` والـ `link` state.
- safe تماماً لأنه read-only ومش بيعمل أي تعديل.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * phylib_spy.c — kprobe on phy_ethtool_get_phy_stats()
 *
 * Every time ethtool queries PHY stats, we print the PHY id,
 * the current link speed, and the link state.
 *
 * Build:  add to a Kbuild-based out-of-tree module, then:
 *   make -C /lib/modules/$(uname -r)/build M=$PWD modules
 *   insmod phylib_spy.ko
 *   ethtool -S eth0          <-- triggers the probe
 *   rmmod phylib_spy
 */

#include <linux/kernel.h>      /* pr_info, pr_err */
#include <linux/module.h>      /* MODULE_* macros, module_init/exit */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe */
#include <linux/phy.h>         /* struct phy_device, phy_interface_t */

/* ------------------------------------------------------------------ */
/*  kprobe handler — runs just before phy_ethtool_get_phy_stats()      */
/* ------------------------------------------------------------------ */

/*
 * phy_ethtool_get_phy_stats() prototype (from phylib_stubs.h):
 *
 *   void phy_ethtool_get_phy_stats(struct phy_device *phydev,
 *                                  struct ethtool_eth_phy_stats *phy_stats,
 *                                  struct ethtool_phy_stats *phydev_stats);
 *
 * On x86-64, the first argument (phydev) arrives in register RDI.
 * regs_get_kernel_argument(regs, 0) بيجيبه بطريقة portable.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * بنسحب أول argument (phydev) من registers الـ ABI.
     * بنعمل cast لـ unsigned long الأول لأن الـ helper بيرجع unsigned long.
     */
    struct phy_device *phydev =
        (struct phy_device *)regs_get_kernel_argument(regs, 0);

    if (!phydev)
        return 0;   /* لو NULL ما نعملش حاجة */

    /*
     * بنطبع:
     *   phy_id   — 32-bit OUI+model+revision من MII registers 2 & 3
     *   speed    — بالـ Mbps (10 / 100 / 1000 / ...)
     *   link     — 1 = up, 0 = down
     *   interface — اسم الـ MII mode (rgmii, sgmii, ...)
     */
    pr_info("phylib_spy: ethtool stats query — "
            "phy_id=0x%08x speed=%d link=%u iface=%s\n",
            phydev->phy_id,
            phydev->speed,
            phydev->link,
            phy_modes(phydev->interface));

    return 0;   /* 0 يعني "استمر في تنفيذ الدالة الأصلية" */
}

/* ------------------------------------------------------------------ */
/*  kprobe descriptor                                                   */
/* ------------------------------------------------------------------ */

/*
 * بنحط اسم الدالة كـ string — الـ kernel بيحلّه لـ address
 * وقت الـ register_kprobe() عن طريق kallsyms.
 * phy_ethtool_get_phy_stats هو الـ inline wrapper الظاهر للـ compiler،
 * لكن لما CONFIG_PHYLIB=y بيكون فعلياً call لـ phylib_stubs->get_phy_stats.
 * نستهدف الـ wrapper نفسه عشان نضمن الـ hook يشتغل حتى لو الـ stub اتغيّر.
 *
 * ملاحظة: لو الـ function اتعمل inline من الـ compiler وما ظهرتش
 * في الـ symbol table، الـ register_kprobe هيرجع -ENOENT — وده المتوقع.
 * في الحالة دي جرّب "phy_ethtool_get_link_ext_stats" بديلاً.
 */
static struct kprobe kp = {
    .symbol_name = "phy_ethtool_get_phy_stats",
    .pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ */
/*  init / exit                                                         */
/* ------------------------------------------------------------------ */

static int __init phylib_spy_init(void)
{
    int ret;

    /*
     * بنسجّل الـ kprobe — الـ kernel بيكتب breakpoint (INT3 على x86)
     * في أول byte من الدالة المستهدفة.
     * لو فشل (مثلاً الدالة inline أو محمية) بنطبع الـ error ونرجع.
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("phylib_spy: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("phylib_spy: hooked %s at %p\n",
            kp.symbol_name, kp.addr);
    return 0;
}

static void __exit phylib_spy_exit(void)
{
    /*
     * لازم نشيل الـ kprobe قبل ما الـ module يتحذف من الذاكرة،
     * لأن الـ handler pointer هيبقى dangling لو الـ code اتحذف
     * والـ breakpoint لسه موجود.
     */
    unregister_kprobe(&kp);
    pr_info("phylib_spy: unhooked %s\n", kp.symbol_name);
}

module_init(phylib_spy_init);
module_exit(phylib_spy_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe spy on phy_ethtool_get_phy_stats to log PHY state");
```

---

### شرح كل جزء

#### `#include`s

| Header | السبب |
|---|---|
| `<linux/kernel.h>` | `pr_info`, `pr_err` |
| `<linux/module.h>` | ماكروهات `MODULE_*` و `module_init`/`module_exit` |
| `<linux/kprobes.h>` | `struct kprobe`، `register_kprobe`، `regs_get_kernel_argument` |
| `<linux/phy.h>` | `struct phy_device`، `phy_modes()` |

**الـ** `phy.h` مش موجود في `phylib_stubs.h` مباشرةً — ده forward declaration بس — لكن الـ module محتاجه عشان يوصل لـ fields الـ `phy_device` زي `phy_id` و`speed`.

---

#### `handler_pre`

الـ `pre_handler` بيتشغّل **قبل** تنفيذ الدالة المستهدفة. استخدمنا `regs_get_kernel_argument(regs, 0)` بدل ما نقرأ الـ register مباشرةً (مثلاً `regs->di`) عشان الكود يشتغل على كل architectures مش بس x86.

الـ fields اللي بنطبعها:

| Field | المعنى |
|---|---|
| `phydev->phy_id` | 32-bit identifier من MDIO registers 2 & 3 — بيحدد vendor + model |
| `phydev->speed` | السرعة الحالية بالـ Mbps |
| `phydev->link` | `1` = link up، `0` = link down |
| `phy_modes(phydev->interface)` | string اسم الـ MII mode |

---

#### `struct kprobe`

الـ `symbol_name` يخلي الـ kernel يبحث عن الـ address بنفسه عن طريق `kallsyms` — أحسن من hardcode عنوان لأن العنوان بيتغيّر مع كل build. الـ `pre_handler` هو الـ callback اللي هيتنادى قبل التنفيذ.

---

#### `module_init` / `module_exit`

- `__init`: `register_kprobe` بيكتب الـ breakpoint في الذاكرة. لو الدالة `inline` أو محمية بـ `nokprobe_inline`، بيرجع error والـ module ما بيتحملش.
- `__exit`: `unregister_kprobe` **ضروري** قبل تحرير الـ module — لو ما اتعملش، الـ kernel ممكن يكال handler في address اتحذفت ويحصل kernel panic.

---

### Makefile للبناء

```makefile
obj-m += phylib_spy.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

---

### تجربة سريعة

```bash
# تحميل الـ module
sudo insmod phylib_spy.ko

# تشغيل ethtool على interface فيه PHY
sudo ethtool -S eth0

# مشاهدة الـ output
sudo dmesg | grep phylib_spy
# phylib_spy: hooked phy_ethtool_get_phy_stats at 0xffffffffc0a12340
# phylib_spy: ethtool stats query — phy_id=0x004dd072 speed=1000 link=1 iface=rgmii-id

# إزالة الـ module
sudo rmmod phylib_spy
# phylib_spy: unhooked phy_ethtool_get_phy_stats
```

**الـ** `phy_id=0x004dd072` مثلاً بيدل على Marvell 88E1111 — الـ OUI موجود في الـ bytes العليا.
