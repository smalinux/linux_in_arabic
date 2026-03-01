## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الـ file ده جزء من subsystem اسمه **NETWORKING [ETHTOOL PHY TOPOLOGY]**، المسؤول عنه Maxime Chevallier من Bootlin. الـ subsystem ده جزء من phylib (Physical Layer Library) — المكتبة اللي الـ Linux kernel بيستخدمها عشان يتعامل مع الـ PHY chips (الـ chips اللي بتشوف الإشارة الفيزيائية على الكابل).

---

### الصورة الكبيرة — القصة كاملة

#### المشكلة اللي بيحلها الـ file

تخيل عندك سيرفر بـ network card (الـ MAC). الـ MAC محتاج PHY chip عشان يتكلم مع الكابل الفيزيائي. في الحالة البسيطة:

```
+--------+   MII bus   +-----+   cable   +------+
|  MAC   | ----------- | PHY | --------- | Port |
+--------+             +-----+           +------+
net_device            phy_device
```

الـ Linux كان دايمًا بيفترض إن في **PHY واحد بس** لكل `net_device`. فيه field واحد `net_device.phydev` بيشاور عليه.

#### المشكلة مع SFP

الـ **SFP** هو معيار للـ transceivers القابلة للتبديل — الحاجة دي اللي بتشوفها في السيرفرات والسويتشات، الـ cage اللي بتحط فيها الـ module الزجاجي أو النحاسي.

بعض الـ SFP modules جوّاها PHY chip خاص بيها:

```
+-----+  SGMII   +------------------------+
| MAC | -------- | PHY جوا الـ SFP module |
+-----+          +------------------------+
```

هنا فيه PHY بس مش متعلق مباشرة بالـ MAC — هو جوا الـ module اللي ممكن تشيله وتحطه في أي وقت.

#### الحالة الأكثر تعقيدًا — PHY فوق PHY

بعض الـ MAC controllers مش قادرين يتكلموا مع SFP مباشرة (بيطلعوا RGMII مثلًا مش SGMII). الحل؟ بيحطوا **PHY وسيط (media converter)**:

```
+-----+ RGMII +-----------------------+ SGMII +--------------+
| MAC | ----- | PHY (media converter) | ----- | PHY (on SFP) |
+-----+       +-----------------------+       +--------------+
                  PHY_UPSTREAM_MAC               PHY_UPSTREAM_PHY
```

دلوقتي عندنا **PHYين على نفس الـ link**. الـ model القديم (`net_device.phydev` pointer واحد) مش كافي. مين هيتحكم فيهم؟ مين هيعرف إنهم موجودين أصلًا؟

#### الحل — الـ `phy_link_topology`

الـ file ده بيعرّف الـ framework اللي بيحل المشكلة دي:

- كل `net_device` دلوقتي عنده `link_topo` — pointer على **`struct phy_link_topology`**
- الـ `phy_link_topology` جوّاه **xarray** (نوع من الـ associative arrays في الـ kernel) بيخزن كل الـ PHYs المتصلة بالـ netdevice دي
- كل PHY بياخد **`phyindex`** — رقم unique زي الـ `ifindex` بالظبط بس للـ PHYs
- الـ userspace يقدر يعمل query على كل الـ PHYs دي عن طريق ethtool netlink

---

### الـ Structs الأساسية

#### `struct phy_link_topology`
```c
struct phy_link_topology {
    struct xarray phys;        /* xarray بيخزن كل الـ PHY nodes */
    u32 next_phy_index;        /* counter لتوليد الـ phyindex التالي */
};
```
دي الـ "topology map" — بتعيش في `net_device->link_topo`.

#### `struct phy_device_node`
```c
struct phy_device_node {
    enum phy_upstream upstream_type; /* MAC أو PHY تاني */

    union {
        struct net_device  *netdev;  /* لو upstream هو MAC */
        struct phy_device  *phydev;  /* لو upstream هو PHY تاني */
    } upstream;

    struct sfp_bus *parent_sfp_bus;  /* لو الـ PHY جوا SFP module */

    struct phy_device *phy;          /* الـ PHY نفسه */
};
```
كل node بتمثل PHY واحد وبتقول: "إيه اللي فوقي في الـ chain؟"

#### `enum phy_upstream`
```c
enum phy_upstream {
    PHY_UPSTREAM_MAC,  /* الـ PHY متصل بـ MAC مباشرة */
    PHY_UPSTREAM_PHY,  /* الـ PHY متصل بـ PHY تاني (media converter) */
};
```

---

### الـ API

| Function | الوظيفة |
|---|---|
| `phy_link_topo_add_phy()` | بيضيف PHY للـ topology، بيدّيه phyindex |
| `phy_link_topo_del_phy()` | بيشيل PHY من الـ topology |
| `phy_link_topo_get_phy()` | بيجيب الـ `phy_device` بالـ phyindex بتاعه |

الـ functions دي مش محتاج تناديها يدوياً في معظم الحالات — الـ phylib بيناديها أوتوماتيك جوا `phy_attach_direct()` و detach.

---

### الـ Flow بالكامل

```
phy_attach_direct()
    └─> phy_link_topo_add_phy(dev, phydev, PHY_UPSTREAM_MAC, dev)
            └─> netdev_alloc_phy_link_topology()  [لو أول PHY]
            └─> xa_alloc_cyclic() → phy->phyindex = 1, 2, 3, ...

SFP module يُدخَل → phylink يشوف PHY جوا الـ module
    └─> phy_link_topo_add_phy(dev, sfp_phy, PHY_UPSTREAM_PHY, converter_phy)

userspace يعمل ETHTOOL_MSG_PHY_GET DUMP
    └─> net/ethtool/phy.c يمشي على الـ xarray
    └─> يرجع كل الـ PHYs بالـ phyindex بتاعهم

userspace يعمل ETHTOOL_MSG_CABLE_TEST مع ETHTOOL_A_HEADER_PHY_INDEX=2
    └─> يوصل للـ PHY الخارجي مباشرة
```

---

### ليه ده مهم؟

قبل الـ framework ده، لو عايز تعمل cable test على الـ PHY اللي جوا الـ SFP module — مش على الـ PHY المتصل بالـ MAC — مكنتش تقدر. الـ ethtool كان بيشوف PHY واحد بس. دلوقتي تقدر تقول "اعمل cable test على PHY رقم 2" وتوصله مباشرة.

---

### الـ Files المكوّنة للـ Subsystem

| File | الدور |
|---|---|
| `include/linux/phy_link_topology.h` | الـ header الرئيسي — الـ structs والـ API |
| `drivers/net/phy/phy_link_topology.c` | الـ implementation — add/del logic |
| `net/ethtool/phy.c` | الـ userspace interface عبر ethtool netlink |
| `include/uapi/linux/ethtool.h` | تعريف `enum phy_upstream` و ethtool commands |
| `include/linux/phy.h` | تعريف `struct phy_device` و `phyindex` field |
| `include/linux/netdevice.h` | تعريف `link_topo` field في `struct net_device` |
| `drivers/net/phy/phy_device.c` | phylib core — بيستخدم الـ API في attach/detach |
| `Documentation/networking/phy-link-topology.rst` | التوثيق الرسمي |
## Phase 2: شرح الـ PHY Link Topology Framework

### المشكلة — ليه الـ Framework ده موجود أصلاً؟

في الشبكات الـ embedded، بين الـ MAC (Ethernet controller) والسلك الفعلي في
التوبولوجيا الحديثة مش بالضرورة بيبقى فيه PHY واحد بس — ممكن يبقى في سلسلة:

```
MAC ──MDIO──► PHY (media converter) ──MII──► SFP slot ──► SFP transceiver (with embedded PHY)
                      ↑                                           ↑
               PHY_UPSTREAM_MAC                           PHY_UPSTREAM_PHY
```

الكيرنل قبل الـ framework ده كان بيعرف الـ `net_device` يشوف PHY واحد بس — اللي
موجود في `dev->phydev`. لو في PHY تاني "اتفرع" من SFP module، الـ userspace مكانش
عنده طريقة يوصله أو يسأل عن capabilities بتاعته زي FEC أو EEE أو timestamping.

**المشكلة الجوهرية:** الـ `net_device` كان أعمى بالنسبة لأي PHY في السلسلة غير اللي
هو متكلم معاه مباشرة.

---

### الحل — نهج الكيرنل

الكيرنل حل المشكلة بـ **phy_link_topology**: قائمة مفهرسة بكل PHY devices اللي
جزء من link path بتاع الـ `net_device`، سواء كانوا متصلين مباشرة بالـ MAC أو
عبر PHY وسيط.

الفكرة: كل PHY بياخد **phyindex** فريد (شبه `ifindex` للـ interface)، وبيتخزن في
`xarray` جوه `phy_link_topology` المرتبطة بالـ `net_device`. الـ userspace بقى يقدر
يعمل enumerate للـ PHYs دي عبر ethtool وياخد capabilities كل واحد منهم.

---

### مثال واقعي — SFP على سويتش Marvell

تخيل بورد embedded فيه سويتش Marvell 88Q2110. بورد يعني:

```
Marvell MAC port ──MDIO──► 88Q2110 PHY (automotive)
                                │
                          SFP cage
                                │
                    SFP-T transceiver
                    (embedded PHY داخله)
```

قبل الـ framework: الكيرنل يعرف بس الـ 88Q2110. الـ SFP PHY؟ مجهول.
بعد الـ framework: الاتنين اتسجلوا في `phy_link_topology` وكل واحد عنده index.
الـ userspace يعمل `ethtool --show-phys eth0` ويشوف الاتنين ويقدر يتحكم فيهم.

---

### Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Userspace                            │
│              ethtool / nl80211 / custom tools               │
└──────────────────────────┬──────────────────────────────────┘
                           │  ethtool netlink API (phyindex)
┌──────────────────────────▼──────────────────────────────────┐
│                    Kernel Networking                         │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  struct net_device                                  │    │
│  │  ┌───────────────────────────┐                      │    │
│  │  │ *link_topo ───────────────┼──────────────────┐   │    │
│  │  │ *phydev  (direct PHY)     │                  │   │    │
│  │  │ *sfp_bus                  │                  │   │    │
│  │  └───────────────────────────┘                  │   │    │
│  └─────────────────────────────────────────────────┼───┘   │
│                                                     │        │
│  ┌──────────────────────────────────────────────────▼──┐    │
│  │  struct phy_link_topology                           │    │
│  │  ┌──────────────────────────────────────────────┐  │    │
│  │  │  xarray phys                                 │  │    │
│  │  │  ┌─────────────┐   ┌─────────────┐           │  │    │
│  │  │  │ index=1     │   │ index=2     │    ...    │  │    │
│  │  │  │ pdn (node)  │   │ pdn (node)  │           │  │    │
│  │  │  └──────┬──────┘   └──────┬──────┘           │  │    │
│  │  └─────────┼────────────────┼──────────────────┘  │    │
│  └────────────┼────────────────┼──────────────────────┘    │
│               │                │                            │
│  ┌────────────▼──────┐  ┌──────▼──────────┐                │
│  │ phy_device_node   │  │ phy_device_node  │                │
│  │ upstream=netdev   │  │ upstream=phydev  │                │
│  │ upstream_type=MAC │  │ upstream_type=PHY│                │
│  │ parent_sfp_bus=X  │  │ parent_sfp_bus=Y │                │
│  │ *phy ──────────┐  │  │ *phy ──────────┐ │                │
│  └────────────────┼──┘  └────────────────┼─┘                │
│                   │                       │                  │
│  ┌────────────────▼──────┐  ┌────────────▼───────────┐     │
│  │  struct phy_device    │  │  struct phy_device      │     │
│  │  (88Q2110 PHY)        │  │  (SFP embedded PHY)     │     │
│  │  phyindex = 1         │  │  phyindex = 2           │     │
│  │  is_on_sfp_module = 0 │  │  is_on_sfp_module = 1   │     │
│  └───────────────────────┘  └─────────────────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

---

### Analogy واقعية — سجل المستأجرين في عمارة

تخيل الـ `net_device` عمارة. الـ MAC هو الحارس. قبل الـ framework، الحارس
كان بيسجل اسم مستأجر واحد بس (اللي في الشقة 1). لو في مستأجر تاني اتضاف
(جه مع واحد تاني) مكانش متسجل في أي سجل.

**الـ phy_link_topology** هو **سجل رسمي للعمارة**:

| الـ Analogy | الـ Kernel Concept |
|---|---|
| العمارة | `struct net_device` |
| سجل المستأجرين | `struct phy_link_topology` |
| رقم الشقة (فريد) | `phyindex` (u32) |
| بطاقة بيانات كل مستأجر | `struct phy_device_node` |
| نوع صلة المستأجر بالعمارة | `enum phy_upstream` |
| "جاي معاه" أو "مستأجر مباشر" | `PHY_UPSTREAM_PHY` vs `PHY_UPSTREAM_MAC` |
| شركة إدارة العقارات اللي بيجي منها | `parent_sfp_bus` |
| نفس المستأجر | `struct phy_device *phy` |
| فهرس السجل (xarray) | البحث السريع بـ `xa_load()` |
| إعادة نفس رقم الشقة لو رجع نفس المستأجر | `phy->phyindex` persistence عبر `xa_insert` |

الحارس (الكيرنل) مش بيشيل السجل إلا لو العمارة اتهدت. المستأجر ممكن يمشي
ويرجع ويلاقي نفس رقم الشقة محجوز له.

---

### الـ Core Abstraction — الفكرة المحورية

الـ abstraction المركزية هي: **كل PHY في مسار الـ link يمثَّل بـ `phy_device_node` داخل `xarray` مرتبطة بالـ `net_device`**.

مش مجرد pointer لـ PHY — لكن **node** بيعرف:
1. **مين هو** (`*phy`)
2. **من فين جاي** (`upstream_type` + `upstream` union)
3. **عبر أيه** (`parent_sfp_bus`)

```c
struct phy_device_node {
    enum phy_upstream upstream_type;  /* MAC أو PHY */

    union {
        struct net_device  *netdev;   /* لو upstream هو MAC */
        struct phy_device  *phydev;   /* لو upstream هو PHY وسيط */
    } upstream;

    struct sfp_bus *parent_sfp_bus;   /* SFP bus اللي جاي منه لو موجود */

    struct phy_device *phy;           /* الـ PHY نفسه */
};
```

**الـ xarray** (ملاحظة: مش xarray تقليدي) — الـ xarray هو data structure في الكيرنل
بيجمع بين features الـ radix tree والـ array مع lock-free lookups. محتاج تفهمه؟
بالاختصار: `xa_load(xa, index)` = O(1) lookup بدون lock، و`xa_alloc_cyclic` =
تخصيص index فريد تلقائياً بالتسلسل الدوري. ده بيخلي البحث عن PHY بـ phyindex
سريع جداً.

---

### العلاقات بين الـ Structs

```
struct net_device
│
└─► *link_topo ──────────────────────────────────────────────┐
                                                              │
                                          struct phy_link_topology
                                          ┌────────────────────────┐
                                          │  xarray phys            │
                                          │  u32 next_phy_index     │
                                          └──────────┬─────────────┘
                                                     │
                              ┌──────────────────────┼──────────────────────┐
                              │                      │                      │
                    key=1     ▼          key=2       ▼         key=3       ▼
              ┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐
              │ phy_device_node   │ │ phy_device_node   │ │ phy_device_node   │
              │ upstream=netdev   │ │ upstream=phydev   │ │ upstream=phydev   │
              │ type=MAC          │ │ type=PHY          │ │ type=PHY          │
              │ sfp_bus=NULL      │ │ sfp_bus=sfp_bus_A │ │ sfp_bus=sfp_bus_B │
              │ *phy──────────┐   │ │ *phy──────────┐   │ │ *phy──────────┐   │
              └───────────────┼───┘ └───────────────┼───┘ └───────────────┼───┘
                              │                     │                     │
                   ┌──────────▼──────┐   ┌──────────▼──────┐   ┌─────────▼──────┐
                   │ phy_device      │   │ phy_device      │   │ phy_device     │
                   │ phyindex=1      │   │ phyindex=2      │   │ phyindex=3     │
                   │ (direct to MAC) │   │ (on SFP 1)      │   │ (on SFP 2)     │
                   └─────────────────┘   └─────────────────┘   └────────────────┘
```

---

### الـ Framework يمتلك إيه vs بيفوّض لمين؟

#### الـ Framework يمتلك:
- **تخصيص الـ `phy_link_topology`** — بيتعمل lazy (أول مرة يتضاف PHY)
- **تخصيص الـ `phy_device_node`** وملء بياناته
- **إدارة الـ xarray** (add/remove/lookup بـ phyindex)
- **تخصيص الـ phyindex** لكل PHY جديد عبر `xa_alloc_cyclic`
- **الـ API الثلاثي** (add / del / get)

#### الـ Framework يفوّض لـ:

| المهمة | من يتولاها |
|---|---|
| اكتشاف الـ PHY على MDIO bus | **PHYLIB** (`CONFIG_PHYLIB`) — الـ framework نفسه مشروط بيه |
| إدارة state machine الـ PHY (up/down/negotiation) | **PHYLIB** أيضاً |
| ربط الـ PHY بالـ SFP cage | **SFP subsystem** (`struct sfp_bus`) |
| نشر الـ topology للـ userspace | **ethtool netlink** (يستخدم `phy_link_topo_get_phy`) |
| تحديد `phy_upstream` الصحيح | الـ driver نفسه اللي بيستدعي `phy_link_topo_add_phy` |
| تدمير الـ `phy_link_topology` عند إلغاء الـ netdev | **net_device lifecycle** code |

---

### دورة حياة الـ Topology — من البداية للنهاية

```
Driver probe
     │
     ▼
phy_connect() / phylink_connect_phy()
     │
     ▼
phy_link_topo_add_phy(dev, phy, PHY_UPSTREAM_MAC, dev)
     │  ← lazy alloc phy_link_topology إذا مش موجودة
     │  ← alloc phy_device_node
     │  ← xa_alloc_cyclic → phyindex=1
     ▼
SFP module inserted
     │
     ▼
sfp_upstream notified → PHY on SFP detected
     │
     ▼
phy_link_topo_add_phy(dev, sfp_phy, PHY_UPSTREAM_PHY, parent_phy)
     │  ← alloc phy_device_node
     │  ← phyindex=2, parent_sfp_bus=dev->sfp_bus
     ▼
ethtool userspace query → phy_link_topo_get_phy(dev, 1 or 2)
     │
     ▼
SFP module removed
     │
     ▼
phy_link_topo_del_phy(dev, sfp_phy)
     │  ← xa_erase(phyindex=2), kfree(pdn)
     │  ← phy->phyindex=2 يفضل محفوظ (مش بيتمسح)
     ▼
Driver remove
     │
     ▼
phy_link_topo_del_phy(dev, phy)
     └─ phy_link_topology تفضل موجودة حتى يتحذف الـ net_device
```

**ملاحظة مهمة على الـ phyindex persistence:** لما بتمسح PHY من الـ topology، الكيرنل
**مش بيصفّر** `phy->phyindex`. ده عمد — لو نفس الـ PHY object اتضاف تاني
(مثلاً SFP اتنزع واتحط تاني)، هيلاقي نفس الـ index محجوز له، فالـ userspace
tools مش هتتلخبط وتشوفه بـ index جديد كل مرة.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Enums والـ Config Options — Cheatsheet

#### `enum phy_upstream` — (defined in `include/uapi/linux/ethtool.h`)

| القيمة | المعنى |
|---|---|
| `PHY_UPSTREAM_MAC` | الـ upstream هو MAC مباشر (controller أو switch port) |
| `PHY_UPSTREAM_PHY` | الـ upstream هو PHY تاني (media converter مثلاً) |

#### Config Options

| الـ Option | التأثير |
|---|---|
| `CONFIG_PHYLIB` | لو مش enabled، كل الـ API بتتحول لـ stubs بترجع `0` أو `NULL`، والـ topology بيتجاهل تماماً |

---

### الـ Structs المهمة

#### 1. `struct phy_link_topology`

**الغرض:** الـ container الرئيسي اللي بيحتفظ بكل الـ PHY devices المرتبطة بـ `net_device` واحد. بيتحجز lazily — يعني ما بيتعملش `alloc` غير لما أول PHY بيتضاف.

| الـ Field | النوع | الشرح |
|---|---|---|
| `phys` | `struct xarray` | الـ XArray اللي بيخزن فيه `phy_device_node*` مفهرسة بالـ `phyindex` |
| `next_phy_index` | `u32` | الـ counter اللي بيحدد الـ index الجاي عند `xa_alloc_cyclic`، بيبدأ من `1` |

- **بيتحجز** بـ `kzalloc` في `netdev_alloc_phy_link_topology()`.
- الـ XArray بيتبدأ بـ `XA_FLAGS_ALLOC1` عشان الـ indices تبدأ من `1` (القيمة `0` محجوزة تدل إن الـ PHY مش في أي topology).
- بيتخزن في `net_device->link_topo`.

---

#### 2. `struct phy_device_node`

**الغرض:** node واحدة في الـ topology بتمثل PHY واحد. بتربط الـ PHY بـ upstream بتاعه (MAC أو PHY تاني) وكمان بـ SFP bus لو موجود.

| الـ Field | النوع | الشرح |
|---|---|---|
| `upstream_type` | `enum phy_upstream` | هل الـ upstream MAC ولا PHY |
| `upstream.netdev` | `struct net_device *` | الـ MAC الـ upstream (لو `PHY_UPSTREAM_MAC`) |
| `upstream.phydev` | `struct phy_device *` | الـ PHY الـ upstream (لو `PHY_UPSTREAM_PHY`) |
| `parent_sfp_bus` | `struct sfp_bus *` | الـ SFP bus اللي الـ PHY موجود عليه لو `phy_on_sfp()` |
| `phy` | `struct phy_device *` | بوينتر للـ PHY device نفسه |

- الـ `upstream` union — بس field واحد فيهم بيكون valid حسب `upstream_type`.
- `parent_sfp_bus` بيتملى من `netdev->sfp_bus` أو `phydev->sfp_bus` حسب الـ upstream type.

---

#### 3. `struct phy_device` *(طرف ثالث — مُستخدَم)*

| الـ Field المهم | الشرح |
|---|---|
| `phyindex` | الـ unique id بتاع الـ PHY في الـ topology (مثل `ifindex`)، القيمة `0` = مش مسجل |
| `sfp_bus` | الـ SFP bus المرتبط بالـ PHY لو على SFP module |
| `is_on_sfp_module` | flag بيقرأه `phy_on_sfp()` |

---

#### 4. `struct net_device` *(طرف ثالث — anchor)*

| الـ Field المهم | الشرح |
|---|---|
| `link_topo` | بوينتر لـ `phy_link_topology*`، بيكون `NULL` لحد ما أول PHY بيتضاف |
| `phydev` | الـ primary PHY (الـ legacy field) |
| `sfp_bus` | الـ SFP bus المرتبط بالـ MAC مباشرة |

---

### Struct Relationship Diagram

```
struct net_device
┌─────────────────────────────────┐
│  link_topo ──────────────────────┼──► struct phy_link_topology
│  phydev    ──► struct phy_device │    ┌──────────────────────────────┐
│  sfp_bus   ──► struct sfp_bus    │    │  phys (xarray)               │
└─────────────────────────────────┘    │  ┌────────┬────────┬────────┐ │
                                        │  │ idx=1  │ idx=2  │ idx=3  │ │
                                        │  └───┬────┴───┬────┴───┬────┘ │
                                        │      │        │        │      │
                                        │  next_phy_index = 4          │
                                        └──────┼────────┼────────┼──────┘
                                               │        │        │
                                               ▼        ▼        ▼
                                       struct phy_device_node  (one per PHY)
                                       ┌────────────────────────────────┐
                                       │  phy ──────► struct phy_device │
                                       │  upstream_type                 │
                                       │  upstream ──► net_device  OR   │
                                       │               phy_device       │
                                       │  parent_sfp_bus ──► sfp_bus    │
                                       └────────────────────────────────┘

Case 1: PHY_UPSTREAM_MAC (simple single-PHY link)
  net_device ──────────────► phy_device_node ──► phy_device
  (upstream = net_device itself)

Case 2: PHY_UPSTREAM_PHY (chained PHYs / media converter + SFP PHY)
  net_device ──► node[1] ──► media_converter_phy
                               └─(upstream MAC)
  net_device ──► node[2] ──► sfp_phy
                               └─(upstream = media_converter_phy)
```

---

### Topology الـ SFP المركبة — مثال واقعي

```
السيناريو: MAC + media-converter PHY + SFP module يحتوي PHY

  +-------+  RGMII  +------------------+  SGMII  +----------------+
  |  MAC  |---------|  PHY (converter) |---------|  PHY (on SFP)  |
  +-------+         +------------------+         +----------------+
net_device          phy_device[1]                phy_device[2]
                    phyindex=1                   phyindex=2
                    upstream=MAC                 upstream=phy[1]
                    parent_sfp_bus=NULL          parent_sfp_bus=sfp_bus

  xarray layout:
  [1] → phy_device_node { phy=phy[1], type=MAC,  upstream=net_device }
  [2] → phy_device_node { phy=phy[2], type=PHY,  upstream=phy[1],
                           parent_sfp_bus=sfp_bus }
```

---

### Lifecycle Diagram

```
CREATION (lazy — عند أول phy_link_topo_add_phy)
─────────────────────────────────────────────────
net_device created
    │
    │  link_topo = NULL  (ما بيتحجزش مع الـ netdev)
    │
    ▼
phy_attach_direct(dev, phy, ...) or phy_sfp_attach_phy(...)
    │
    ▼
phy_link_topo_add_phy(dev, phy, upt, upstream)
    │
    ├─► [link_topo == NULL?]
    │       └─► netdev_alloc_phy_link_topology(dev)
    │               kzalloc(phy_link_topology)
    │               xa_init_flags(&topo->phys, XA_FLAGS_ALLOC1)
    │               topo->next_phy_index = 1
    │               dev->link_topo = topo
    │
    ├─► kzalloc(phy_device_node)
    │
    ├─► fill pdn fields (phy, upstream, parent_sfp_bus)
    │
    ├─► [phy->phyindex != 0?]
    │       ├─ YES → xa_insert(&topo->phys, phy->phyindex, pdn)
    │       └─ NO  → xa_alloc_cyclic(...)  ──► assigns phy->phyindex
    │
    └─► PHY is now in topology (visible via ethtool netlink)


USAGE
─────
phy_link_topo_get_phy(dev, phyindex)
    │
    ├─► topo = dev->link_topo  (NULL check)
    ├─► pdn = xa_load(&topo->phys, phyindex)
    └─► return pdn->phy


TEARDOWN (عند phy_detach أو phy_sfp_detach_phy)
────────────────────────────────────────────────
phy_link_topo_del_phy(dev, phy)
    │
    ├─► topo = dev->link_topo  (NULL check)
    ├─► pdn = xa_erase(&topo->phys, phy->phyindex)
    │       (phy->phyindex نفسه ما بيتصفرش — بيتحفظ لإعادة الاستخدام)
    └─► kfree(pdn)

    NOTE: phy_link_topology نفسها ما بتتحررش هنا — بتتحرر مع الـ net_device
```

---

### Call Flow Diagrams

#### إضافة PHY عبر `phy_attach_direct`

```
phy_attach_direct(dev, phydev, flags, interface)
  └─► phy_link_topo_add_phy(dev, phydev, PHY_UPSTREAM_MAC, dev)
        └─► [no topo yet?]
              └─► netdev_alloc_phy_link_topology(dev)
                    └─► xa_init_flags(&topo->phys, XA_FLAGS_ALLOC1)
        └─► kzalloc(phy_device_node)
        └─► pdn->upstream.netdev = dev
        └─► [phy_on_sfp(phy)?]
              └─► pdn->parent_sfp_bus = dev->sfp_bus
        └─► xa_alloc_cyclic(...)  →  assigns phy->phyindex
```

#### إضافة PHY على SFP خلف media-converter

```
PHY driver (media converter) calls:
  phy_sfp_attach_phy(upstream, sphy)          ← sphy = SFP module PHY
    └─► phy_link_topo_add_phy(
              dev    = phydev->attached_dev,  ← الـ net_device الأصلي
              phy    = sphy,
              upt    = PHY_UPSTREAM_PHY,
              upstr  = phydev               ← الـ media-converter PHY نفسه
        )
          └─► pdn->upstream.phydev = phydev
          └─► pdn->parent_sfp_bus = phydev->sfp_bus
          └─► xa_alloc_cyclic(...)  →  assigns sphy->phyindex
```

#### قراءة PHY من userspace عبر ethtool netlink

```
userspace: ethtool --show-phys eth0
  └─► ethtool netlink (kernel)
        └─► phy_link_topo_get_phy(dev, phyindex)
              └─► xa_load(&dev->link_topo->phys, phyindex)
                    └─► return pdn->phy
              └─► report phy info (id, speed caps, upstream_type, etc.)
```

---

### Locking Strategy

الـ header نفسه ما بيعرفش locks صريحة — الـ locking بيجي من الـ callers:

| الـ Lock | بيحمي إيه | التفاصيل |
|---|---|---|
| `RTNL lock` (`rtnl_lock`) | الـ `net_device->link_topo` pointer وكل الـ topology بشكل عام | كل الـ calls لـ `phy_link_topo_add_phy` و `phy_link_topo_del_phy` بتحصل تحت RTNL — زي `phy_attach_direct` و `phy_detach` |
| `xarray internal lock` | الـ `topo->phys` xarray نفسه | الـ XArray بـ lock داخلية spinlock، `xa_load` thread-safe، `xa_insert`/`xa_erase` protected automatically |
| **Lock Ordering** | RTNL → xarray internal lock | RTNL لازم يتاخد الأول (هو الـ outer lock)، الـ xarray lock بيتاخد جوا الـ xa_* calls |

**ملاحظة مهمة:** الـ `phy->phyindex` نفسه ما عندوش lock خاص بيه — بيتكتب جوا `xa_alloc_cyclic` تحت حماية الـ xarray lock، وبيتقرأ بعد كده read-only. طالما الـ RTNL شايل، الـ `dev->link_topo` pointer نفسه safe.

```
Lock Ordering:
  rtnl_lock()                   ← outer
    └─► xa_lock(&topo->phys)   ← inner (implicit inside xa_* calls)
          └─► [access/modify xarray entries]
        xa_unlock()
  rtnl_unlock()
```
## Phase 4: شرح الـ Functions

### ملخص سريع — Cheatsheet

| Function | Type | Description |
|---|---|---|
| `phy_link_topo_add_phy()` | exported, `EXPORT_SYMBOL_GPL` | يضيف PHY جديد لـ topology الـ netdev |
| `phy_link_topo_del_phy()` | exported, `EXPORT_SYMBOL_GPL` | يحذف PHY من الـ topology |
| `phy_link_topo_get_phy()` | static inline | يجيب `phy_device` بالـ index بتاعه |
| `netdev_alloc_phy_link_topology()` | static, internal | يعمل allocate للـ topology structure لأول مرة |

---

### التصنيفات

```
┌─────────────────────────────────────────────────────────┐
│  PHY Link Topology API                                  │
├──────────────────┬──────────────────┬───────────────────┤
│  Initialization  │  Registration    │  Lookup / Query   │
│  (internal)      │  (exported)      │  (inline)         │
├──────────────────┼──────────────────┼───────────────────┤
│ netdev_alloc_    │ phy_link_topo_   │ phy_link_topo_    │
│ phy_link_        │ add_phy()        │ get_phy()         │
│ topology()       │ phy_link_topo_   │                   │
│                  │ del_phy()        │                   │
└──────────────────┴──────────────────┴───────────────────┘
```

---

### Group 1: Initialization — الـ Lazy Allocation

هذه المجموعة مسؤولة عن إنشاء الـ `phy_link_topology` structure لأول مرة. الـ kernel بيعمل **lazy allocation** — يعني الـ topology مش بتتعمل وقت إنشاء الـ netdev، لكن بتتعمل أول ما بيتضاف PHY.

---

#### `netdev_alloc_phy_link_topology()`

```c
static int netdev_alloc_phy_link_topology(struct net_device *dev)
```

**بتعمل إيه:**
بتعمل `kzalloc` لـ `phy_link_topology` جديدة وتربطها بـ `dev->link_topo`. بتعمل initialize للـ xarray بـ `XA_FLAGS_ALLOC1` عشان الـ index يبدأ من 1 مش صفر، وبتشيل `next_phy_index = 1`.

**Parameters:**
- `dev` — الـ `net_device` اللي محتاج topology جديدة.

**Return value:**
- `0` عند النجاح.
- `-ENOMEM` لو فشل الـ `kzalloc`.

**Key details:**
- الـ flag `XA_FLAGS_ALLOC1` مهم: بيخلي الـ xarray يبدأ من index 1، عشان index 0 يكون reserved ويعكس "PHY مش مضاف لـ topology".
- الدالة static ومش exported — caller الوحيد هو `phy_link_topo_add_phy()`.

---

### Group 2: Registration — إضافة وحذف الـ PHYs

الـ APIs الأساسية اللي بيستخدمها الـ PHYLIB subsystem لتسجيل وإلغاء تسجيل الـ PHY devices في الـ topology.

---

#### `phy_link_topo_add_phy()`

```c
int phy_link_topo_add_phy(struct net_device *dev,
                          struct phy_device *phy,
                          enum phy_upstream upt, void *upstream);
```

**بتعمل إيه:**
بتضيف `phy_device` جديد لـ topology الـ netdev. بتعمل allocate لـ `phy_device_node` وبتملاه بمعلومات الـ upstream، وبعدين بتسجله في الـ xarray بـ index فريد. لو الـ PHY عنده `phyindex` موجود من قبل (يعني اتضاف قبل كده ومش أول مرة)، بتحاول تعيد استخدام نفس الـ index بدل ما تخصص جديد.

**Parameters:**
- `dev` — الـ `net_device` اللي بتتبع له الـ topology.
- `phy` — الـ `phy_device` المراد إضافته.
- `upt` — نوع الـ upstream component:
  - `PHY_UPSTREAM_MAC` — الـ PHY متصل مباشرة بـ MAC controller أو switch port.
  - `PHY_UPSTREAM_PHY` — الـ PHY متصل بـ PHY تاني (media converter scenario).
- `upstream` — pointer لـ `net_device` أو `phy_device` حسب قيمة `upt` — بيُعمل cast داخليًا.

**Return value:**
- `0` عند النجاح.
- `-ENOMEM` لو فشل الـ kzalloc.
- `-EINVAL` لو قيمة `upt` غير معروفة.
- error من `xa_insert()` أو `xa_alloc_cyclic()` لو فشل التسجيل في الـ xarray.

**Key details:**
- **Lazy topology init:** لو `dev->link_topo == NULL`، بتستدعي `netdev_alloc_phy_link_topology()` أولًا.
- **SFP bus tracking:** لو الـ PHY على SFP module (يعني `phy_on_sfp(phy)` صح)، بيسجل الـ `parent_sfp_bus` في الـ node — ده بيسمح بتتبع SFP chains.
- **Index reuse:** لو `phy->phyindex != 0` (يعني الـ PHY اشتغل قبل كده مع نفس الـ netdev)، بيستخدم `xa_insert()` لإعادة الـ index القديم. لو `phyindex == 0` بيستخدم `xa_alloc_cyclic()` لتخصيص index جديد تلقائيًا.
- الدالة exported بـ `EXPORT_SYMBOL_GPL` — بيستخدمها PHYLIB فقط.
- **Caller context:** بيستدعيها `phy_attach_direct()` و `phy_link_topo_add_phy()` في سياق process context مع RTNL lock محتمل.

**Pseudocode flow:**

```
phy_link_topo_add_phy(dev, phy, upt, upstream):
  if dev->link_topo == NULL:
      netdev_alloc_phy_link_topology(dev)  // lazy init

  pdn = kzalloc(phy_device_node)

  pdn->phy = phy
  switch upt:
    MAC:  pdn->upstream.netdev = upstream
          if phy_on_sfp(phy): pdn->parent_sfp_bus = netdev->sfp_bus
    PHY:  pdn->upstream.phydev = upstream
          if phy_on_sfp(phy): pdn->parent_sfp_bus = phydev->sfp_bus
    else: return -EINVAL

  pdn->upstream_type = upt

  if phy->phyindex != 0:
      xa_insert(&topo->phys, phy->phyindex, pdn)   // reuse old index
  else:
      xa_alloc_cyclic(&topo->phys, &phy->phyindex, pdn, ...) // alloc new

  return 0 (or error)
```

---

#### `phy_link_topo_del_phy()`

```c
void phy_link_topo_del_phy(struct net_device *dev, struct phy_device *phy);
```

**بتعمل إيه:**
بتحذف الـ `phy_device_node` الخاص بـ `phy` من الـ xarray وبتحرر الـ memory بتاعته. **مهم:** الدالة عمدًا مش بتصفي `phy->phyindex` — ده قرار تصميمي مقصود عشان لو الـ PHY رجع تاني يأخذ نفس الـ index.

**Parameters:**
- `dev` — الـ `net_device` اللي فيها الـ topology.
- `phy` — الـ `phy_device` المراد حذفه.

**Return value:**
- `void` — مفيش return value.

**Key details:**
- لو `dev->link_topo == NULL` (topology مش موجودة)، بترجع فورًا بدون عمل حاجة — safe to call unconditionally.
- `xa_erase()` بتشيل الـ entry من الـ xarray وبترجع الـ pointer القديم — بعدين `kfree()` بتحرر الـ `phy_device_node`.
- الـ `phy->phyindex` بيفضل بقيمته (مش بيُعاد لـ 0) — ده بيمكّن الـ index reuse في `phy_link_topo_add_phy()`.
- Exported بـ `EXPORT_SYMBOL_GPL`.
- **Caller context:** بيستدعيها `phy_detach()` و `phy_disconnect()` في process context.

---

### Group 3: Lookup — استعلام عن الـ PHYs

---

#### `phy_link_topo_get_phy()`

```c
static inline struct phy_device *
phy_link_topo_get_phy(struct net_device *dev, u32 phyindex);
```

**بتعمل إيه:**
بتجيب الـ `phy_device` المرتبط بـ index معين من الـ topology. الدالة inline وخفيفة جدًا — بس lookup في الـ xarray مع null check.

**Parameters:**
- `dev` — الـ `net_device` اللي فيها الـ topology.
- `phyindex` — الـ index الفريد للـ PHY (بيجي من `phy_device->phyindex`).

**Return value:**
- pointer لـ `phy_device` لو الـ index موجود.
- `NULL` لو الـ topology مش موجودة (`dev->link_topo == NULL`) أو الـ index مش موجود في الـ xarray.

**Key details:**
- **مفيش locking** داخلي — الـ caller مسؤول عن الـ synchronization المناسب (عادة RTNL lock).
- `xa_load()` آمنة تُستدعى من أي context بدون lock — لكن الـ returned pointer صلاحيته مرتبطة بوجود الـ PHY في الـ topology.
- الـ stub في حالة `!CONFIG_PHYLIB` بترجع `NULL` دايمًا — safe for non-PHY systems.
- بيستدعيها ethtool عند استجواب الـ topology عبر netlink.

---

### الـ Data Structures المستخدمة

#### `struct phy_link_topology`

```c
struct phy_link_topology {
    struct xarray phys;        /* xarray maps phyindex -> phy_device_node */
    u32 next_phy_index;        /* cyclic allocator hint, starts at 1 */
};
```

**الـ xarray** هنا بيعمل كـ sparse array — المفتاح هو الـ `phyindex` (u32 من 1 لـ `UINT_MAX`)، والقيمة هي pointer لـ `phy_device_node`. الـ `XA_FLAGS_ALLOC1` بيضمن إن الـ index 0 مش بيُستخدم (reserved كـ "no index").

#### `struct phy_device_node`

```c
struct phy_device_node {
    enum phy_upstream upstream_type;  /* MAC or PHY upstream */

    union {
        struct net_device  *netdev;   /* if upstream is MAC */
        struct phy_device  *phydev;   /* if upstream is PHY (media conv.) */
    } upstream;

    struct sfp_bus *parent_sfp_bus;   /* non-NULL if PHY is on SFP module */

    struct phy_device *phy;           /* the PHY itself */
};
```

الـ union بيوفر memory — في أي وقت في upstream واحد بس: إما MAC أو PHY. الـ `upstream_type` بيحدد أيهم valid.

#### `enum phy_upstream`

```c
enum phy_upstream {
    PHY_UPSTREAM_MAC,   /* PHY attached directly to MAC / switch port */
    PHY_UPSTREAM_PHY,   /* PHY attached to another PHY (media converter) */
};
```

---

### سيناريو عملي: PHY Chain مع SFP

```
┌─────────────────────────────────────────────────────────────┐
│  net_device (eth0)                                          │
│    └── link_topo                                            │
│           └── phys (xarray)                                 │
│                  ├── [index=1] phy_device_node              │
│                  │     upstream_type = PHY_UPSTREAM_MAC     │
│                  │     upstream.netdev = eth0               │
│                  │     parent_sfp_bus = eth0->sfp_bus       │
│                  │     phy = <media-converter PHY>          │
│                  │                                          │
│                  └── [index=2] phy_device_node              │
│                        upstream_type = PHY_UPSTREAM_PHY     │
│                        upstream.phydev = <conv. PHY>        │
│                        parent_sfp_bus = conv_phy->sfp_bus   │
│                        phy = <SFP transceiver PHY>          │
└─────────────────────────────────────────────────────────────┘
```

الـ call sequence:
```
phy_attach_direct(dev, media_conv_phy):
    phy_link_topo_add_phy(dev, media_conv_phy, PHY_UPSTREAM_MAC, dev)
        → index=1 assigned

phy_attach_direct(dev, sfp_phy):
    phy_link_topo_add_phy(dev, sfp_phy, PHY_UPSTREAM_PHY, media_conv_phy)
        → index=2 assigned
```

---

### CONFIG_PHYLIB Stubs

لو `CONFIG_PHYLIB` مش enabled، كل الـ functions بتبقى static inline stubs:
- `phy_link_topo_add_phy()` → returns `0` دايمًا.
- `phy_link_topo_del_phy()` → no-op.
- `phy_link_topo_get_phy()` → returns `NULL` دايمًا.

ده بيخلي الـ callers مش محتاجين `#ifdef` في كودهم.
## Phase 5: دليل الـ Debugging الشامل

### Software Level

#### 1. debugfs

الـ `phy_link_topology` ما بيعرّف entries مباشرة في debugfs، بس الـ PHYLIB subsystem بيوفر:

```bash
# mount debugfs لو مش موجود
mount -t debugfs none /sys/kernel/debug

# شوف كل entries موجودة
ls /sys/kernel/debug/mdio/

# كل bus موجود
ls /sys/kernel/debug/mdio/<bus-id>/
```

الـ kernel ما بيعرض `phy_link_topology` نفسه في debugfs، لكن ممكن نوصل للمعلومات عبر xarray بالـ ethtool netlink — الطريقة الصح هي ethtool/netlink اللي بيقرأ من `topo->phys` مباشرة.

---

#### 2. sysfs

الـ **sysfs** بيكشف حالة كل PHY على الـ MDIO bus:

```
/sys/class/mdio_bus/<bus-id>/<phy-addr>/
├── attached_dev          → symlink للـ net_device المرتبط
├── phy_id                → 32-bit PHY OUI+model
├── phy_interface         → rgmii, sgmii, 1000base-x, ...
├── phy_has_fixups        → 0 أو 1
├── phy_standalone        → يظهر لو PHY شغال بدون netdev
├── phy_dev_flags         → bitmask للـ flags من الـ MAC driver
└── statistics/
    ├── transfers         → عدد MDIO transactions الكلي
    ├── errors            → عدد الأخطاء
    ├── reads             → قراءات
    └── writes            → كتابات
```

```bash
# اقرأ معلومات PHY الأساسية
cat /sys/class/mdio_bus/stmmac-1/0:01/phy_id
cat /sys/class/mdio_bus/stmmac-1/0:01/phy_interface
cat /sys/class/mdio_bus/stmmac-1/0:01/attached_dev

# إحصائيات MDIO bus
cat /sys/class/mdio_bus/stmmac-1/0:01/statistics/errors
cat /sys/class/mdio_bus/stmmac-1/0:01/statistics/transfers
```

---

#### 3. ftrace — tracepoints وإيفينتس

الـ subsystem بيستخدم tracepoint واحد رئيسي:

**`mdio:mdio_access`** — بيتفعّل عند كل read/write على MDIO bus

```bash
# فعّل الـ tracepoint
echo 1 > /sys/kernel/debug/tracing/events/mdio/mdio_access/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل عملية (مثلاً ifup أو ethtool)
ip link set eth0 up

# اقرأ النتيجة
cat /sys/kernel/debug/tracing/trace
```

**مثال على output:**

```
# TASK-PID   CPU#  TIMESTAMP  FUNCTION
kworker/0:2-123   [000]  1234.56789: mdio_access: stmmac-1 read  phy:0x01 reg:0x01 val:0x796d
kworker/0:2-123   [000]  1234.56812: mdio_access: stmmac-1 read  phy:0x01 reg:0x00 val:0x1140
```

**التفسير:**
- `stmmac-1` → اسم الـ MDIO bus
- `phy:0x01` → عنوان PHY على الـ bus
- `reg:0x01` → رقم الـ register (0x01 = MII_BMSR - Basic Mode Status Register)
- `val:0x796d` → القيمة المقروءة

```bash
# فلتر على bus معين
echo 'busid == "stmmac-1"' > /sys/kernel/debug/tracing/events/mdio/mdio_access/filter

# فلتر على phy address
echo 'addr == 1' > /sys/kernel/debug/tracing/events/mdio/mdio_access/filter

# تابع live
cat /sys/kernel/debug/tracing/trace_pipe &
ip link set eth0 down && ip link set eth0 up
```

للـ **function tracing** مع الـ topology:

```bash
# trace دوال phy_link_topo_*
echo phy_link_topo_add_phy > /sys/kernel/debug/tracing/set_ftrace_filter
echo phy_link_topo_del_phy >> /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... trigger event ...
cat /sys/kernel/debug/tracing/trace
echo nop > /sys/kernel/debug/tracing/current_tracer
```

---

#### 4. printk و dynamic debug

```bash
# فعّل dynamic debug لكل drivers/net/phy/
echo "file drivers/net/phy/phy_link_topology.c +pflmt" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/net/phy/phy_device.c +pflmt" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/net/phy/phy.c +pflmt" > /sys/kernel/debug/dynamic_debug/control

# فعّل لكل PHY subsystem دفعة واحدة
echo "module phylib +pflmt" > /sys/kernel/debug/dynamic_debug/control

# اقرأ ما اتفعّل
cat /sys/kernel/debug/dynamic_debug/control | grep phy_link_topology

# شوف الـ kernel log
dmesg -w | grep -i phy
```

الخيارات بتاعة `+pflmt`:
- `p` → فعّل الطباعة
- `f` → اطبع اسم الدالة
- `l` → اطبع رقم السطر
- `m` → اطبع اسم الـ module
- `t` → اطبع الـ thread ID

---

#### 5. Kernel Config Options

| Config | الوصف |
|--------|-------|
| `CONFIG_PHYLIB` | لازم يكون مفعّل — الـ core library لكل PHY support |
| `CONFIG_PHYLINK` | لدعم PHY chaining وSFP upstream |
| `CONFIG_SFP` | لدعم SFP cages وتسجيل PHYs عليها في التوبولوجي |
| `CONFIG_MDIO_BUS` | الـ bus layer — لازم مفعّل |
| `CONFIG_DEBUG_KERNEL` | تفعيل debugging العام |
| `CONFIG_DYNAMIC_DEBUG` | لازم لاستخدام dynamic_debug |
| `CONFIG_TRACING` | لازم لاستخدام ftrace |
| `CONFIG_EVENT_TRACING` | لازم لـ mdio:mdio_access tracepoint |
| `CONFIG_LOCK_DEBUGGING_SUPPORT` | للـ lockdep مع MDIO bus locks |
| `CONFIG_PROVE_LOCKING` | لكشف deadlocks في MDIO locking |
| `CONFIG_SLUB_DEBUG` | لكشف memory corruption في `phy_device_node` allocations |
| `CONFIG_KASAN` | لكشف use-after-free في `pdn` بعد `xa_erase` |
| `CONFIG_PHYLIB_LEDS` | لو بتستخدم LED triggers مع PHY state |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "CONFIG_PHYLIB|CONFIG_PHYLINK|CONFIG_SFP|CONFIG_MDIO"
```

---

#### 6. ethtool — الأداة الرئيسية للـ topology

الـ `phy_link_topology` صُمِّم خصيصاً للتعامل معه عبر **ethtool netlink API**:

```bash
# اعرض كل PHYs على interface (يقرأ من topo->phys xarray)
ethtool --show-phys eth0

# مثال output لـ SFP scenario:
# PHY index: 1
#   Name: 0000:00:1f.6 (stmmac)
#   Driver: motorcomm
#   Upstream type: MAC
#
# PHY index: 2
#   Name: sfp-phy.0
#   Driver: generic
#   Upstream type: PHY
#   Upstream PHY index: 1
#   Parent SFP bus: sfp0

# استعلم عن PHY معين بـ phyindex
ethtool --show-phy eth0 --phy-index 2

# cable test على PHY خارجي (SFP PHY مثلاً)
ethtool --cable-test eth0 --phy-index 2

# إحصائيات من PHY معين
ethtool --show-phy-stats eth0 --phy-index 1

# PSE/PoE على PHY معين
ethtool --get-pse eth0 --phy-index 1

# اعرض معلومات netlink بالتفصيل
ethtool --debug 0x100 --show-phys eth0
```

**netlink dump مباشرة بـ `iproute2`:**

```bash
# عرض معلومات الـ link
ip link show eth0
ip -d link show eth0   # أكثر تفصيلاً

# عبر python-pyroute2
python3 -c "
from pyroute2 import IPRoute
with IPRoute() as ipr:
    idx = ipr.link_lookup(ifname='eth0')[0]
    print(ipr.get_links(idx))
"
```

**devlink للـ switch PHYs:**

```bash
# لو الـ PHY جزء من switch
devlink dev show
devlink port show
devlink port split <dev/port> count 4
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة في dmesg / kernel log | المعنى | الحل |
|-----------------------------|--------|-------|
| `phy_link_topo_add_phy: -ENOMEM` | فشل تخصيص `phy_device_node` أو `phy_link_topology` | تحقق من available memory — `cat /proc/meminfo` |
| `phy_link_topo_add_phy: -EINVAL` | `upstream_type` مش PHY_UPSTREAM_MAC ولا PHY_UPSTREAM_PHY | bug في الـ driver — الـ `upt` parameter غلط |
| `xa_insert: -EBUSY` | `phy->phyindex` موجود بالفعل في الـ xarray | مشكلة في lifecycle — PHY اتضاف مرتين بدون حذف |
| `phy phy-X:XX: Device is not supported` | الـ PHY ID ما لقاله driver | فعّل `CONFIG_GENERIC_PHY` أو ثبّت driver مناسب |
| `MDIO bus timeout` | الـ PHY ما ردّ على MDIO read/write | تحقق من الـ hardware — power, reset pin, pull-ups |
| `phy_attach: PHY is already attached` | محاولة attach PHY مرتبط بـ netdev تاني | `phy_detach()` الأول |
| `libphy: stmmac-1:00 - Link is Down` | link_state تغير لـ down | طبيعي عند unplug — لو مستمر راجع cable وSFP |
| `sfp sfp0: module is not supported` | SFP module ما في drivers ليه | أضف quirk أو تحقق من vendor ID |
| `net eth0: error -11 attaching PHY` | `-EAGAIN` — PHY مش ready لسه | retry من kernel — لو ظهر كثير راجع MDIO timing |
| `phy0: PHY reset timed out` | reset pin مش بيستجيب | تحقق من GPIO reset وتوقيت الـ power sequence |

---

#### 8. مواضع WARN_ON() و dump_stack()

أماكن استراتيجية لإضافة checks مؤقتة أثناء debugging:

```c
/* في phy_link_topo_add_phy() — تحقق إن topo اتعمل صح */
WARN_ON(!dev->link_topo);

/* تحقق إن PHY مش مسجل مرتين */
WARN_ON(xa_load(&topo->phys, phy->phyindex) != NULL && phy->phyindex != 0);

/* في phy_link_topo_del_phy() — تحقق إن الـ pdn موجود قبل الحذف */
WARN_ON_ONCE(!xa_load(&topo->phys, phy->phyindex));

/* في phy_link_topo_get_phy() — تحقق صحة الـ index */
WARN_ON(phyindex == 0); /* 0 محجوز كـ "not in topology" */

/* dump_stack عند ENOMEM لمتابعة من طلب الـ allocation */
if (!pdn) {
    dump_stack();
    return -ENOMEM;
}
```

للـ **xarray integrity check** في جلسات debugging:

```c
/* تحقق إن الـ xarray متسق بعد add/del */
#ifdef CONFIG_DEBUG_KERNEL
{
    unsigned long idx;
    struct phy_device_node *n;
    xa_for_each(&topo->phys, idx, n)
        WARN_ON(!n->phy);
}
#endif
```

---

### Hardware Level

#### 1. التحقق من حالة الـ Hardware تطابق حالة الـ Kernel

```bash
# حالة PHY في الـ kernel
ethtool eth0
ethtool --show-phys eth0

# اقرأ BMSR register مباشرة (reg 0x01)
# لو phytool مثبّت:
phytool read eth0/1 0/1    # bus=0, phy_addr=1, reg=1

# أو عبر ethtool registers dump
ethtool -d eth0
```

الـ **BMSR (register 0x01)** bits المهمة:

```
Bit 5 → Auto-Negotiation Complete
Bit 4 → Remote Fault
Bit 2 → Link Status
Bit 1 → Jabber Detect
```

```bash
# قارن حالة الـ link في kernel مع الـ PHY hardware
cat /sys/class/net/eth0/operstate          # up/down
cat /sys/class/net/eth0/carrier             # 1/0
cat /sys/class/net/eth0/speed               # 1000/100/10
cat /sys/class/net/eth0/duplex              # full/half

# لو في خلاف → مشكلة في PHY state machine
```

---

#### 2. Register Dump Techniques

**باستخدام `phytool` (الأفضل):**

```bash
# ثبّت phytool
apt install phytool   # أو build من source

# اقرأ registers أساسية
phytool read eth0/1 0/0    # BMCR - Basic Mode Control Register
phytool read eth0/1 0/1    # BMSR - Basic Mode Status Register
phytool read eth0/1 0/2    # PHY ID 1
phytool read eth0/1 0/3    # PHY ID 2
phytool read eth0/1 0/4    # ANAR - Auto-Neg Advertisement
phytool read eth0/1 0/5    # ANLPAR - Link Partner Ability
phytool read eth0/1 0/6    # ANER - Auto-Neg Expansion
```

**باستخدام `ethtool -d` (dump registers):**

```bash
# dump كل registers المتاحة
ethtool -d eth0

# مثال output:
# PHY register dump for eth0
# CTRL:    0x1140    AN_EN=1 DUPLEX=1 SPEED_SEL1=1
# STATUS:  0x796d    AN_COMPLETE=1 LINK=1
```

**باستخدام `mdio-tool` أو `/dev/mem` (للـ embedded):**

```bash
# تحذير: /dev/mem خطير — استخدم بحذر
# أفضل استخدام devmem2
devmem2 0xFEC00000 w    # عنوان MDIO controller register

# أو مباشرة عبر io utility
io -4 0xFEC00000
```

**باستخدام sysfs statistics للـ MDIO bus:**

```bash
# احصل على MDIO bus ID من sysfs
ls /sys/class/mdio_bus/

# اقرأ statistics
cat /sys/class/mdio_bus/stmmac-1/statistics/errors
cat /sys/class/mdio_bus/stmmac-1/statistics/transfers

# لكل PHY address
cat /sys/class/mdio_bus/stmmac-1/statistics/errors_1
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

الـ **MDIO** بروتوكول 2-wire (MDC clock + MDIO data):

```
MDIO Frame Structure:
PRE:  32 ones (preamble)
ST:   01 (start)
OP:   10=read, 01=write
PA5:  PHY address (5 bits)
RA5:  Register address (5 bits)
TA:   turnaround (10 for read, z0 for write)
DATA: 16 bits

Example read frame:
|11..1|01|10|00001|00001|10|ZZZZZZZZZZZZZZZZ|
 pream  st op  pa   ra   ta      data
```

**إعدادات Logic Analyzer:**

```
- Protocol: MDIO (IEEE 802.3)
- MDC frequency: عادة 2.5 MHz أو 25 MHz
- Voltage threshold: 1.8V أو 3.3V حسب الـ PHY
- Sample rate: × 10 على الأقل من MDC (25 MHz → 250 MHz sample)
- Trigger: على أول 32 ones (preamble)
```

**إعدادات Oscilloscope:**

```
- قِس MDC: تأكد من duty cycle 50% ± 10%
- قِس MDIO setup time: > 10 ns قبل rising edge of MDC
- قِس MDIO hold time: > 10 ns بعد rising edge of MDC
- قِس rise/fall time: < 25 ns على كابل طويل
- تحقق من pull-up resistors على MDIO: عادة 1.5kΩ - 10kΩ
```

**SFP I2C debugging:**

```bash
# الـ SFP module بيتواصل عبر I2C (A0h, A2h addresses)
i2cdetect -y 0           # اكتشف devices على I2C bus 0
i2cdump -y 0 0x50        # A0h = SFP EEPROM (module info)
i2cdump -y 0 0x51        # A2h = SFP DDM (diagnostics)

# اقرأ vendor name من SFP EEPROM
i2cget -y 0 0x50 0x14    # Vendor Name start byte
```

---

#### 4. مشاكل Hardware الشائعة وأنماطها في الـ kernel log

| المشكلة | النمط في dmesg | السبب المحتمل |
|---------|----------------|---------------|
| PHY ما اكتُشف | `libphy: bus_scan() failed` | MDIO pull-up مفقود، PHY address غلط في DT |
| Link oscillation سريع | `eth0: Link is Up\neth0: Link is Down` بتكرار | كابل رديء، SFP dirty، auto-neg مشكلة |
| فشل auto-negotiation | `AN restart failed` أو speed ثابت 10M | incompatible PHY capabilities، مشكلة في register 0x04 |
| SFP PHY ما اتسجل في topology | topology فيها PHY واحد بس رغم SFP | phylink ما فعّل SFP upstream ops |
| MDIO timeout مستمر | `MDIO bus stmmac-1: MDIO read timeout` | MDC clock مش وصل للـ PHY، power issue |
| PHY reset loop | `phy reset` يتكرر كل ثواني | GPIO reset pin عالق، timing خطأ |
| Duplex mismatch | throughput منخفض + collisions عالية | طرف fixed full، طرف auto → half duplex |
| Wrong speed after AN | `Link is Up - 100/Full` بينما cable يدعم 1G | AN advertisement مقيّد في register 0x04 |

---

#### 5. Device Tree Debugging

الـ `phy_link_topology` بيعتمد على DT لتحديد:
- أين الـ PHY على MDIO bus (PHY address)
- نوع الـ interface (rgmii, sgmii, etc.)
- وجود SFP cage

```bash
# اقرأ DT الفعلي في runtime
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "mdio"

# تحقق من PHY address في DT vs ما اكتُشف في sysfs
cat /sys/firmware/devicetree/base/soc/ethernet@*/mdio/ethernet-phy@*/reg
# يجب أن يطابق
ls /sys/class/mdio_bus/*/

# تحقق من phy-handle
cat /sys/firmware/devicetree/base/soc/ethernet@*/phy-handle 2>/dev/null

# تحقق من phy-mode
cat /sys/class/net/eth0/phydev/phy_interface
# يجب أن يطابق ما في DT
```

**DT صحيح لتوبولوجي PHY chained:**

```dts
/* MAC → PHY (media converter) → SFP PHY */
&mdio0 {
    converter_phy: ethernet-phy@1 {
        reg = <1>;
        sfp = <&sfp_cage0>;
        /* يجب على الـ driver استدعاء phy_sfp_attach_phy */
    };
};

&eth0 {
    phy-handle = <&converter_phy>;
    phy-mode = "rgmii-id";
};

&sfp_cage0 { /* SFP PHY يتسجل تلقائياً في topology */ };
```

```bash
# تحقق إن SFP ops متسجلة
dmesg | grep -i "sfp.*upstream\|sfp.*attach\|phylink.*sfp"

# تحقق من الـ topology result
ethtool --show-phys eth0
# توقع: PHY index 1 (converter) + PHY index 2 (SFP PHY)
```

---

### Practical Commands

#### دليل سريع — أوامر جاهزة للنسخ

**1. فحص شامل للـ topology:**

```bash
#!/bin/bash
IFACE=${1:-eth0}
echo "=== PHY Link Topology for $IFACE ==="
ethtool --show-phys $IFACE 2>/dev/null || echo "ethtool version قديم — لا يدعم --show-phys"

echo ""
echo "=== PHY sysfs info ==="
PHYDEV=$(readlink /sys/class/net/$IFACE/phydev 2>/dev/null)
if [ -n "$PHYDEV" ]; then
    BASE="/sys/class/mdio_bus"
    for bus in $BASE/*/; do
        for phy in $bus*/; do
            attached=$(readlink "$phy/attached_dev" 2>/dev/null | xargs basename 2>/dev/null)
            if [ "$attached" = "$IFACE" ]; then
                echo "PHY: $phy"
                echo "  ID:        $(cat $phy/phy_id 2>/dev/null)"
                echo "  Interface: $(cat $phy/phy_interface 2>/dev/null)"
                echo "  Fixups:    $(cat $phy/phy_has_fixups 2>/dev/null)"
                echo "  Flags:     $(cat $phy/phy_dev_flags 2>/dev/null)"
            fi
        done
    done
fi

echo ""
echo "=== Link State ==="
echo "  operstate: $(cat /sys/class/net/$IFACE/operstate)"
echo "  carrier:   $(cat /sys/class/net/$IFACE/carrier 2>/dev/null)"
echo "  speed:     $(cat /sys/class/net/$IFACE/speed 2>/dev/null)"
echo "  duplex:    $(cat /sys/class/net/$IFACE/duplex 2>/dev/null)"
```

**2. تفعيل MDIO tracing:**

```bash
#!/bin/bash
TRACE=/sys/kernel/debug/tracing

echo "nop" > $TRACE/current_tracer
echo 0 > $TRACE/tracing_on
echo "" > $TRACE/trace

echo 1 > $TRACE/events/mdio/mdio_access/enable
echo 1 > $TRACE/tracing_on

echo "Tracing MDIO for 5 seconds..."
sleep 5

echo 0 > $TRACE/tracing_on
echo 0 > $TRACE/events/mdio/mdio_access/enable

echo "=== MDIO Trace Results ==="
cat $TRACE/trace | grep -v "^#" | head -50
```

**3. مراقبة إضافة/حذف PHY من topology:**

```bash
#!/bin/bash
# تابع phy_link_topo_add_phy و del_phy بالـ function tracer
TRACE=/sys/kernel/debug/tracing

echo nop > $TRACE/current_tracer
echo phy_link_topo_add_phy > $TRACE/set_ftrace_filter
echo phy_link_topo_del_phy >> $TRACE/set_ftrace_filter
echo function > $TRACE/current_tracer
echo 1 > $TRACE/tracing_on

echo "Watching PHY topology changes (Ctrl+C to stop)..."
cat $TRACE/trace_pipe

# cleanup
echo nop > $TRACE/current_tracer
echo > $TRACE/set_ftrace_filter
```

**4. dump PHY registers:**

```bash
#!/bin/bash
# يحتاج phytool أو mdio-tool
IFACE=${1:-eth0}
PHY_ADDR=${2:-1}

if command -v phytool &>/dev/null; then
    echo "=== PHY Registers for $IFACE phy@$PHY_ADDR ==="
    for reg in 0 1 2 3 4 5 6 9 10 15; do
        printf "Reg[0x%02x] = 0x%04x\n" $reg \
            $(phytool read $IFACE/$PHY_ADDR 0/$reg 2>/dev/null || echo 0xFFFF)
    done
else
    echo "Using ethtool dump (less detail)..."
    ethtool -d $IFACE
fi
```

**5. تحقق من SFP topology:**

```bash
#!/bin/bash
IFACE=${1:-eth0}

echo "=== SFP Status ==="
ethtool -m $IFACE 2>/dev/null || echo "No SFP module or not supported"

echo ""
echo "=== SFP EEPROM (if i2c accessible) ==="
for bus in $(seq 0 5); do
    if i2cdetect -y $bus 2>/dev/null | grep -q "50"; then
        echo "SFP found on I2C bus $bus"
        echo "Vendor: $(i2cget -y $bus 0x50 0x14 2>/dev/null)"
        echo "Speed:  $(i2cget -y $bus 0x50 0x0C 2>/dev/null)"
    fi
done

echo ""
echo "=== All PHYs in topology ==="
ethtool --show-phys $IFACE 2>/dev/null
```

**6. فحص MDIO bus statistics:**

```bash
#!/bin/bash
echo "=== MDIO Bus Statistics ==="
for bus in /sys/class/mdio_bus/*/; do
    busname=$(basename $bus)
    stats="$bus/statistics"
    if [ -d "$stats" ]; then
        transfers=$(cat $stats/transfers 2>/dev/null)
        errors=$(cat $stats/errors 2>/dev/null)
        echo "Bus: $busname"
        echo "  Transfers: $transfers"
        echo "  Errors:    $errors"
        if [ "${errors:-0}" -gt 0 ]; then
            echo "  WARNING: MDIO errors detected!"
        fi
    fi
done
```

**7. فحص kernel config اللازمة:**

```bash
#!/bin/bash
echo "=== PHY Link Topology Kernel Config Check ==="
CONF=/proc/config.gz
[ -f "$CONF" ] || CONF=/boot/config-$(uname -r)

check_config() {
    local key=$1 expected=$2
    val=$(zcat $CONF 2>/dev/null | grep "^$key=" | cut -d= -f2)
    [ -z "$val" ] && val=$(grep "^$key=" $CONF 2>/dev/null | cut -d= -f2)
    if [ "$val" = "$expected" ] || [ "$val" = "m" -a "$expected" = "y" ]; then
        echo "  [OK]  $key=$val"
    else
        echo "  [!!]  $key=${val:-not set} (expected $expected)"
    fi
}

check_config CONFIG_PHYLIB y
check_config CONFIG_PHYLINK y
check_config CONFIG_SFP y
check_config CONFIG_MDIO_BUS y
check_config CONFIG_DYNAMIC_DEBUG y
check_config CONFIG_EVENT_TRACING y
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — SFP Module مش بيتعرف

#### العنوان
**SFP transceiver PHY invisible to ethtool على industrial Ethernet gateway**

#### السياق
شركة بتعمل industrial gateway بـ RK3562 فيه port Ethernet بـ SFP cage. المنتج بيستخدم SFP transceiver بيحتوي على PHY embedded جوّاه (مش مجرد optical module). البورد شغال، الـ link بييجي، بس الـ ethtool مش بيشوف غير PHY واحد (الـ on-chip MAC PHY) وبيتجاهل تماماً الـ PHY اللي جوّا الـ SFP.

#### المشكلة
الـ engineer بيعمل:
```bash
ethtool --show-phys eth0
```
بيلاقي PHY واحد بس — الـ inline media converter PHY مش موجود في القايمة. التطبيق الـ industrial بيعتمد على قراءة PHY diagnostics من الـ SFP المدمج لمراقبة جودة الـ link، وده مش شغال.

#### التحليل
الـ `phy_link_topology` هو الـ xarray اللي بيتتبع كل PHY devices في chain:

```c
struct phy_link_topology {
    struct xarray phys;       /* كل الـ PHYs المسجلين */
    u32 next_phy_index;       /* counter تلقائي للـ index */
};
```

لما بيتعمل `ethtool --show-phys`، الـ kernel بيمشي على الـ `topo->phys` xarray. المشكلة إن الـ SFP PHY ما اتضافش للـ topology أصلاً — يعني `phy_link_topo_add_phy()` ما اتاستدعتش بالشكل الصح لهذا الـ PHY.

الـ `phy_device_node` بتاع الـ SFP PHY المفروض يبان كده:
```c
struct phy_device_node {
    enum phy_upstream upstream_type; /* PHY_UPSTREAM_PHY — upstream هو PHY مش MAC */
    union {
        struct net_device *netdev;
        struct phy_device *phydev;   /* الـ media converter PHY هو الـ upstream */
    } upstream;
    struct sfp_bus *parent_sfp_bus;  /* ده المفتاح — ربط بالـ sfp_bus */
    struct phy_device *phy;          /* الـ SFP embedded PHY نفسه */
};
```

الـ `upstream_type` المفروض يكون `PHY_UPSTREAM_PHY` مش `PHY_UPSTREAM_MAC`، والـ `parent_sfp_bus` لازم يبقى non-NULL علشان يربط الـ PHY بالـ SFP cage.

المشكلة التحديدية: الـ SFP bus driver على RK3562 بيعمل `phy_link_topo_add_phy()` بـ `upstream = NULL` أو بـ upstream type غلط، فالـ node بيتضاف بس ببيانات ناقصة — أو ما بيتضافش خالص لو فيه early return.

#### الحل

**1. تشخيص — تأكد من وجود الـ topology:**
```bash
# شوف كل PHYs المسجلين
ethtool --show-phys eth0

# تحقق من الـ kernel log
dmesg | grep -i "phy.*topo\|sfp.*phy"

# شوف الـ sysfs
ls /sys/class/net/eth0/phydev/
```

**2. تحقق من الـ driver code — ابحث عن الاستدعاء الغلط:**
```c
/* خطأ شائع — upstream type غلط لما PHY خلف SFP */
phy_link_topo_add_phy(netdev, sfp_phy,
                      PHY_UPSTREAM_MAC,   /* غلط! */
                      netdev);

/* الصح — upstream هو الـ media converter PHY */
phy_link_topo_add_phy(netdev, sfp_phy,
                      PHY_UPSTREAM_PHY,   /* صح */
                      media_converter_phydev);
```

**3. تأكد من ربط الـ sfp_bus:**
```c
/* في SFP bus driver، بعد attach الـ PHY */
pdn = xa_load(&netdev->link_topo->phys, phy_index);
if (pdn)
    pdn->parent_sfp_bus = sfp_bus; /* ربط صريح */
```

#### الدرس المستفاد
الـ `phy_device_node.upstream_type` و `parent_sfp_bus` مش مجرد metadata — الـ ethtool userspace بيعتمد عليهم لبناء الـ topology tree. لما بيكون فيه SFP PHY، الـ upstream لازم يكون `PHY_UPSTREAM_PHY` مع pointer صريح للـ parent `phy_device`.

---

### السيناريو 2: Android TV Box على Allwinner H616 — Ethernet PHY بيختفي بعد suspend/resume

#### العنوان
**phy_link_topo corruption بعد system suspend على H616 Ethernet**

#### السياق
منتج Android TV box بـ Allwinner H616، فيه Ethernet 100M بـ internal PHY. بعد `adb shell input keyevent KEYCODE_SLEEP` وصحيان الجهاز، الـ Ethernet بييجي `No carrier` وبيفضل مش شغال. الـ reboot بيحل المشكلة. المشكلة بتظهر في 1 من كل 3 suspend cycles.

#### المشكلة
بعد resume، الـ ethtool بيشيل error:
```bash
ethtool -i eth0
# driver: sun8i-emac
# Cannot get driver information: No such device
```
الـ `phy_link_topo_get_phy()` بترجع `NULL` حتى مع index صحيح.

#### التحليل
الـ `phy_link_topo_get_phy()` بتبدأ بـ:

```c
static inline struct phy_device *
phy_link_topo_get_phy(struct net_device *dev, u32 phyindex)
{
    struct phy_link_topology *topo = dev->link_topo;
    struct phy_device_node *pdn;

    if (!topo)          /* <-- هنا الـ NULL check */
        return NULL;

    pdn = xa_load(&topo->phys, phyindex);
    if (pdn)
        return pdn->phy;

    return NULL;
}
```

الـ `dev->link_topo` بييجي `NULL` بعد resume. السبب: الـ suspend path في الـ `sun8i-emac` driver بيعمل `phy_link_topo_del_phy()` للـ PHY صح، بس الـ resume path بيعمل `phy_device_free()` ثم بيعيد allocate PHY جديد بدون ما يعيد يسجله في الـ topology عبر `phy_link_topo_add_phy()`.

النتيجة: الـ xarray فاضي (`topo->phys` فاضي لكن `topo` نفسه موجود، أو أحياناً `topo` نفسه NULL لو فيه race مع unregister).

الـ race condition: الـ resume بيشتغل قبل الـ MDIO bus يصحى، فـ `phy_connect()` بتفشل silently وبتسيب الـ topology فاضية.

#### الحل

**1. تحقق فين الـ del/add calls:**
```bash
# trace الـ function calls
echo 'phy_link_topo_add_phy' > /sys/kernel/tracing/set_ftrace_filter
echo 'phy_link_topo_del_phy' >> /sys/kernel/tracing/set_ftrace_filter
echo function > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/tracing_on
# عمل suspend/resume
cat /sys/kernel/tracing/trace | grep topo
```

**2. الـ fix في driver suspend/resume:**
```c
static int sun8i_emac_resume(struct device *dev)
{
    struct net_device *ndev = dev_get_drvdata(dev);
    struct phy_device *phydev;

    /* wake up MDIO bus أولاً */
    mdio_bus_resume(priv->mii_bus);

    /* reconnect PHY */
    phydev = phy_connect(ndev, priv->phy_node,
                         sun8i_emac_adjust_link,
                         PHY_INTERFACE_MODE_RMII);

    /* إعادة التسجيل في topology — ده اللي بيتنسى */
    phy_link_topo_add_phy(ndev, phydev,
                          PHY_UPSTREAM_MAC, ndev);
    return 0;
}
```

**3. defensive check في ethtool handler:**
```c
/* قبل أي topology query */
if (!dev->link_topo || !dev->link_topo->next_phy_index)
    return -EOPNOTSUPP;
```

#### الدرس المستفاد
كل `phy_link_topo_del_phy()` في suspend لازم يقابله `phy_link_topo_add_phy()` في resume — وده لازم يحصل **بعد** ما الـ MDIO bus يصحى. الـ `next_phy_index` counter ما بيترسيلش لو عملت del ثم add، فالـ index بيتغير — والـ userspace لازم يقرا الـ index الجديد.

---

### السيناريو 3: IoT Sensor على STM32MP1 — Dual PHY على نفس MAC، الـ second PHY مش بيتعرف

#### العنوان
**PHY chaining على STM32MP1 — media converter PHY مش ظاهر في topology**

#### السياق
تصميم IoT gateway industrial بـ STM32MP1 بيستخدم RTL8211F كـ copper PHY متصل بـ MAC، وقدامه VSC8514 كـ media converter بيحول لـ fiber SFP. الـ MAC بيتكلم مع RTL8211F بـ RGMII، والـ RTL8211F بيتكلم مع VSC8514 بـ SGMII، والـ VSC8514 بيطلع SFP.

#### المشكلة
```bash
ethtool --show-phys eth0
# Index 0: RTL8211F (PHY_UPSTREAM_MAC) ✓
# Index 1: يفترض VSC8514 — مش موجود ✗
```
الـ network management software بيحتاج يقرأ SFP DDM (Digital Diagnostic Monitoring) من الـ VSC8514، وده مش ممكن لأنه مش مسجل في topology.

#### التحليل
الـ `phy_link_topology` بيحتفظ بـ chain كاملة:

```
MAC → RTL8211F (index=0, upstream=MAC)
         ↓ MII
      VSC8514 (index=1, upstream=RTL8211F)
         ↓ SFP
      [SFP transceiver]
```

لكل node في الـ chain، الـ `phy_device_node` بيبقى:
```c
/* Node بتاع RTL8211F */
node_rtl = {
    .upstream_type = PHY_UPSTREAM_MAC,
    .upstream.netdev = eth0,
    .parent_sfp_bus = NULL,
    .phy = rtl8211f_phydev,
};

/* Node بتاع VSC8514 — ده اللي ناقص */
node_vsc = {
    .upstream_type = PHY_UPSTREAM_PHY,  /* upstream هو PHY مش MAC */
    .upstream.phydev = rtl8211f_phydev, /* الـ upstream هو RTL8211F */
    .parent_sfp_bus = sfp_bus_ptr,      /* مهم للـ SFP DDM */
    .phy = vsc8514_phydev,
};
```

المشكلة في الـ RTL8211F driver: بيعمل `phy_connect` للـ downstream PHY بس ما بيضيفوش للـ topology:
```c
/* في rtl8211f_probe() — الكود الناقص */
downstream_phy = phy_find_first(downstream_mdio_bus);
/* هنا ناسي يضيف الـ VSC8514 للـ topology */
/* phy_link_topo_add_phy() ما اتاستدعتش! */
```

#### الحل

**1. في RTL8211F driver، إضافة downstream PHY:**
```c
static int rtl8211f_sfp_probe(struct phy_device *phydev)
{
    struct net_device *netdev = phydev->attached_dev;
    struct phy_device *downstream;

    downstream = of_phy_get_and_connect(netdev,
                                        phydev->mdio.dev.of_node,
                                        rtl8211f_downstream_adjust);
    if (IS_ERR(downstream))
        return PTR_ERR(downstream);

    /* تسجيل الـ downstream PHY في topology */
    return phy_link_topo_add_phy(netdev, downstream,
                                 PHY_UPSTREAM_PHY,  /* upstream هو PHY */
                                 phydev);           /* الـ upstream phydev */
}
```

**2. تحقق بعد الـ fix:**
```bash
ethtool --show-phys eth0
# Index 0: RTL8211F  upstream=MAC
# Index 1: VSC8514   upstream=PHY[0]

# قراءة SFP diagnostics من خلال VSC8514
ethtool --show-module eth0
```

**3. Device Tree تعديل — ربط الـ PHYs:**
```dts
/* في DT بتاع STM32MP1 */
&mdio1 {
    phy0: ethernet-phy@0 {    /* RTL8211F */
        compatible = "ethernet-phy-id001c.c916";
        reg = <0>;
        sfp = <&sfp_cage>;
    };
};

&mdio2 {
    phy1: ethernet-phy@1 {    /* VSC8514 — downstream */
        compatible = "ethernet-phy-id0007.0750";
        reg = <1>;
    };
};
```

#### الدرس المستفاد
الـ PHY chaining مش automatic — كل PHY في الـ chain لازم يتضاف يدوياً بـ `phy_link_topo_add_phy()` مع الـ `upstream_type` الصح. الـ `PHY_UPSTREAM_PHY` مع pointer للـ parent phydev هو اللي بيخلي الـ kernel يبني الـ graph الصح.

---

### السيناريو 4: Automotive ECU على i.MX8 — ethtool topology query بتعمل kernel panic

#### العنوان
**NULL pointer dereference في `phy_link_topo_get_phy()` على i.MX8 automotive network**

#### السياق
ECU بـ i.MX8MQ في سيارة — بيستخدم `eth0` للـ vehicle CAN-to-Ethernet gateway. الـ CONFIG_PHYLIB معملهاش enable في الـ production kernel علشان الـ BSP vendor قرر انه مش محتاجها (بيستخدم fixed-link). الـ network monitoring daemon بيعمل periodic query بـ ethtool للـ PHY topology.

#### المشكلة
الـ system بيعمل kernel panic بعد شوية من الـ boot:
```
BUG: kernel NULL pointer dereference, address: 0000000000000008
RIP: phy_link_topo_get_phy+0x18
```

#### التحليل
الكود بيبص:

```c
/* عند CONFIG_PHYLIB=y */
static inline struct phy_device *
phy_link_topo_get_phy(struct net_device *dev, u32 phyindex)
{
    struct phy_link_topology *topo = dev->link_topo;

    if (!topo)          /* هذا الـ check بيحمينا */
        return NULL;

    pdn = xa_load(&topo->phys, phyindex); /* هنا بيحصل الـ panic */
    ...
}
```

المشكلة: الـ vendor BSP عمل `CONFIG_PHYLIB=n` بس فضّل الـ `phy_link_topo_get_phy()` calls في الـ ethtool glue code. مع `CONFIG_PHYLIB=n`، الكود اللي بيتستدعي هو النسخة الـ stub:

```c
/* عند CONFIG_PHYLIB=n — الـ stub */
static inline struct phy_device *
phy_link_topo_get_phy(struct net_device *dev, u32 phyindex)
{
    return NULL;  /* آمن تماماً */
}
```

بس المشكلة الحقيقية مش هنا. الـ vendor بنى kernel خاص بيعمل `#define IS_ENABLED(CONFIG_PHYLIB) 1` بالغلط في custom header — بيخلي الكود يستخدم النسخة الحقيقية مع `CONFIG_PHYLIB` مش مبنياً فعلاً، فالـ `dev->link_topo` بيبقى NULL وبدلاً من ما الـ NULL check يشتغل، فيه race condition بتعمل double dereference.

في الحقيقة: الـ `topo` مش NULL لكن الـ `topo->phys.xa_head` هو NULL لأن الـ xarray ما اتinitialized:
```c
/* الـ xarray محتاج xa_init() قبل xa_load() */
xa_init(&topo->phys);  /* ده بيحصل في phy_link_topo_add_phy */
/* لو ما حصلتش أي add، الـ xa_load بتتعمل على uninitialized memory */
```

#### الحل

**1. تشخيص:**
```bash
# تحقق من kernel config
zcat /proc/config.gz | grep PHYLIB
# CONFIG_PHYLIB=n → الـ stub المفروض يتاستخدم

# decode الـ panic address
addr2line -e vmlinux phy_link_topo_get_phy+0x18
```

**2. الـ fix في vendor BSP:**
```bash
# في الـ kernel .config
CONFIG_PHYLIB=y   # تفعيل صريح

# أو تفعيل fixed-link مع PHYLIB
CONFIG_FIXED_PHY=y
CONFIG_PHYLIB=y
```

**3. Defensive code في ethtool glue:**
```c
/* قبل استدعاء topology functions */
#if IS_ENABLED(CONFIG_PHYLIB)
    phy = phy_link_topo_get_phy(dev, phy_index);
    if (!phy)
        return -ENODEV;
#else
    return -EOPNOTSUPP;
#endif
```

**4. تأكد من xa_init:**
```c
/* في كود init الـ topology */
static int phy_link_topo_init(struct net_device *dev)
{
    dev->link_topo = kzalloc(sizeof(*dev->link_topo), GFP_KERNEL);
    if (!dev->link_topo)
        return -ENOMEM;
    xa_init(&dev->link_topo->phys);  /* لازم قبل أي xa_load */
    dev->link_topo->next_phy_index = 1;
    return 0;
}
```

#### الدرس المستفاد
الـ `#if IS_ENABLED(CONFIG_PHYLIB)` guard في `phy_link_topology.h` موجود لسبب مهم — ما تعملش override له. لو بناء الـ kernel بيستخدم `fixed-link` بدون PHYLIB، الـ topology functions بترجع stubs آمنة، وأي كود بيحاول يستخدم الـ real implementation هيـcrash.

---

### السيناريو 5: Custom Board Bring-up على AM62x — PHY مش بيتسجل في topology أثناء bring-up

#### العنوان
**PHY topology فاضية أثناء board bring-up على AM62x industrial Ethernet**

#### السياق
bring-up لبورد custom بـ TI AM62x (Sitara) للـ industrial automation. البورد فيه CPSW3G Ethernet بـ dual port — port0 بـ DP83867 PHY، port1 بـ DP83869 PHY مع SFP capability. المهندس الـ bring-up بيحاول يتحقق من الـ PHY topology عبر `ethtool` وكل قراياته بترجع فاضية.

#### المشكلة
```bash
ethtool --show-phys eth0
# 0 PHYs listed

ethtool --show-phys eth1
# 0 PHYs listed
```

رغم إن الـ link شايل والـ traffic ماشي.

#### التحليل
الـ `phy_link_topology` بيتبني فقط لما `phy_link_topo_add_phy()` تتاستدعى. في CPSW3G driver على AM62x، الـ PHY connection بتحصل في:

```c
/* في drivers/net/ethernet/ti/am65-cpsw-nuss.c */
static int am65_cpsw_nuss_init_port_ndev(...)
{
    /* الـ PHY بيتربط هنا */
    port->slave.phy = of_phy_connect(ndev, port->slave.phy_node, ...);

    /* المشكلة: phy_link_topo_add_phy() ما باستدعاتش بعد connect */
}
```

الـ `phy_connect()` الـ standard بتربط الـ PHY بالـ netdev، لكن **مش بتسجله في الـ topology تلقائياً**. التسجيل في الـ topology ده مسؤولية الـ driver نفسه. الـ DP83867 driver على الـ AM62x BSP القديم ما كانش بيعمل الـ topology registration.

الـ `next_phy_index` بيبقى على قيمته الابتدائية (صفر أو 1):
```c
struct phy_link_topology {
    struct xarray phys;       /* فاضي — ما فيش entries */
    u32 next_phy_index;       /* = 0 أو 1 لو ما اتضافش حاجة */
};
```

#### الحل

**1. تشخيص — تحقق من الـ phy_connect:**
```bash
# تحقق الـ PHY موجود في MDIO bus
cat /sys/bus/mdio_bus/devices/*/phy_id

# تحقق الـ link state
ip link show eth0
ethtool eth0 | grep "Link detected"

# trace topology registration
dmesg | grep -i "phy.*topo\|link.*topo"
```

**2. الـ fix — إضافة topology registration في AM62x driver:**
```c
static int am65_cpsw_nuss_port_phy_connect(struct am65_cpsw_port *port,
                                            struct net_device *ndev)
{
    struct phy_device *phy;
    int ret;

    phy = of_phy_connect(ndev, port->slave.phy_node,
                         am65_cpsw_nuss_adjust_link,
                         0, port->slave.phy_if);
    if (!phy)
        return -ENODEV;

    port->slave.phy = phy;

    /* تسجيل في topology — الـ fix */
    ret = phy_link_topo_add_phy(ndev, phy,
                                PHY_UPSTREAM_MAC,
                                ndev);
    if (ret) {
        phy_disconnect(phy);
        return ret;
    }

    return 0;
}
```

**3. لـ port1 مع SFP capability — ربط SFP bus:**
```c
/* لما بيكون فيه SFP على port1 */
ret = phy_link_topo_add_phy(ndev, sfp_phy,
                            PHY_UPSTREAM_PHY,
                            dp83869_phydev);
/* بعدين update الـ parent_sfp_bus */
pdn = xa_load(&ndev->link_topo->phys, sfp_phy_index);
if (pdn)
    pdn->parent_sfp_bus = port->sfp_bus;
```

**4. تحقق بعد الـ fix:**
```bash
ethtool --show-phys eth0
# Index 1: DP83867 [PHY_UPSTREAM_MAC]

ethtool --show-phys eth1
# Index 2: DP83869 [PHY_UPSTREAM_MAC]
# Index 3: SFP-PHY  [PHY_UPSTREAM_PHY, parent: DP83869]
```

**5. تحقق من الـ index ordering:**
```bash
# الـ next_phy_index بيزود مع كل add
# port0 PHY → index 1
# port1 PHY → index 2
# SFP PHY  → index 3
# التأكد بـ xa_for_each
```

#### الدرس المستفاد
الـ `phy_connect()` و `phy_link_topo_add_phy()` مستقلين — ربط الـ PHY بالـ MAC مش بيعمل تسجيل تلقائي في الـ topology. في كل driver جديد، لازم تعمل الاتنين صراحةً. أثناء الـ bring-up على AM62x أو أي SoC جديد، التأكد من الـ `ethtool --show-phys` خطوة لازم تكون في الـ checklist.
## Phase 7: مصادر ومراجع

### توثيق الـ Kernel الرسمي

| المصدر | الرابط | الوصف |
|--------|--------|-------|
| PHY link topology | [docs.kernel.org](https://docs.kernel.org/networking/phy-link-topology.html) | التوثيق الرسمي لـ `phy_link_topology` framework — نقطة البداية الأهم |
| PHY Abstraction Layer | [docs.kernel.org](https://www.kernel.org/doc/html/latest/networking/phy.html) | شرح كامل لـ PHYLIB و struct phy_device |
| phylink & SFP | [docs.kernel.org](https://docs.kernel.org/networking/sfp-phylink.html) | كيفية تكامل phylink مع SFP modules |
| ethtool Netlink interface | [docs.kernel.org](https://docs.kernel.org/networking/ethtool-netlink.html) | الـ `ETHTOOL_MSG_PHY_GET` command اللي بيكشف الـ topology لـ userspace |

**مسارات الـ Documentation داخل الـ kernel tree:**
```
Documentation/networking/phy-link-topology.rst
Documentation/networking/phy.rst
Documentation/networking/sfp-phylink.rst
Documentation/networking/ethtool-netlink.rst
```

---

### مقالات LWN.net

| المقال | الرابط | الأهمية |
|--------|--------|---------|
| Phylink & SFP support | [lwn.net/Articles/667055](https://lwn.net/Articles/667055/) | المقال الأساسي اللي شرح phylink لأول مرة — فهم السياق ضروري قبل الـ topology |
| Add support for SFP+ copper modules | [lwn.net/Articles/807091](https://lwn.net/Articles/807091/) | تفاصيل SFP copper و كيف بيتعامل معها الـ kernel |
| net: phy: Introduce PHY ports representation | [lwn.net/Articles/1041798](https://lwn.net/Articles/1041798/) | أحدث تطور — PHY ports representation فوق الـ topology framework |
| PHY Abstraction Layer III | [lwn.net/Articles/144897](https://lwn.net/Articles/144897/) | الجذور التاريخية لـ PHYLIB — فهم التطور |
| RFC: PHY Abstraction Layer II | [lwn.net/Articles/127013](https://lwn.net/Articles/127013/) | النقاشات المبكرة اللي شكّلت الـ PHY subsystem |
| Generic PHY Framework | [lwn.net/Articles/538966](https://lwn.net/Articles/538966/) | الـ generic PHY framework (مختلف عن ethernet PHY — مفيد للمقارنة) |

---

### نقاشات الـ Mailing List والـ Patches

الـ feature اتقدمت من **Maxime Chevallier** من Bootlin في سلسلة patches مرّت بـ 18+ version:

| Patch / Discussion | الرابط | الملاحظة |
|-------------------|--------|-----------|
| PATCH net-next v5 01/13 — Introduce ethernet link topology | [spinics.net](https://www.spinics.net/lists/netdev/msg961146.html) | أول version مكتملة وصلت netdev |
| PATCH net-next v10 00/13 — PHY listing and link_topology | [lore.kernel.org](https://lore.kernel.org/netdev/20240304151011.1610175-1-maxime.chevallier@bootlin.com/T/) | النسخة اللي اتناقشت فيها الـ xarray approach |
| PATCH net-next v17 00/14 — PHY listing and link_topology | [lore.kernel.org](https://lore.kernel.org/netdev/20240709063039.2909536-2-maxime.chevallier@bootlin.com/T/) | النسخة شبه النهائية — July 2024 |
| PATCH ethtool-next — PHY listing and targetting | [lore.kernel.org](https://lore.kernel.org/linux-arm-kernel/20240103142950.235888-4-maxime.chevallier@bootlin.com/T/) | الجانب الخاص بـ ethtool userspace tool |
| RFC phylink and SFP support | [spinics.net](https://www.spinics.net/lists/netdev/msg445767.html) | النقاش الأصلي اللي بدأ فكرة الـ topology representation |

> **ملاحظة:** للبحث في الـ lore.kernel.org عن أي نقاش جديد، ابحث بـ `phy_link_topology` أو `phy_link_topo_add_phy`.

---

### Kernelnewbies.org — Release Notes

| إصدار | الرابط | ما يخص الـ topology |
|-------|--------|---------------------|
| Linux 6.9 | [kernelnewbies.org/Linux_6.9](https://kernelnewbies.org/Linux_6.9) | الإصدار اللي دخلت فيه الـ phy_link_topology framework رسمياً |
| Linux 6.15 | [kernelnewbies.org/Linux_6.15](https://kernelnewbies.org/Linux_6.15) | آخر تحديثات الـ PHY subsystem |

---

### eLinux.org

| المصدر | الرابط | الفائدة |
|--------|--------|---------|
| Generic PHY Framework Overview (PDF) | [elinux.org](https://elinux.org/images/1/1c/Abraham--generic_phy_framework_an_overview.pdf) | شرح شامل لـ PHY framework architecture بشكل عام |
| Linux Kernel Resources | [elinux.org](https://elinux.org/Linux_Kernel_Resources) | مرجع عام للـ kernel subsystems والـ mailing lists |

---

### الملفات المصدرية الأساسية في الـ Kernel

```
include/linux/phy_link_topology.h   ← الـ header الرئيسي
drivers/net/phy/phy_link_topology.c ← الـ implementation
include/linux/phy.h                 ← struct phy_device و enum phy_upstream
include/linux/netdevice.h           ← struct net_device و حقل link_topo
include/linux/xarray.h              ← الـ xarray data structure المستخدمة للتخزين
drivers/net/phy/phylink.c           ← تكامل phylink مع الـ topology
net/ethtool/phy.c                   ← كشف الـ topology لـ userspace عبر ethtool
```

---

### كتب مرجعية

| الكتاب | الفصول ذات الصلة | الملاحظة |
|--------|-----------------|-----------|
| **Linux Device Drivers 3rd Ed. (LDD3)** — Corbet, Rubini, Kroah-Hartman | Chapter 17: Network Drivers | الأساس لفهم network driver model — متاح مجاناً على lwn.net |
| **Linux Kernel Development** — Robert Love (3rd Ed.) | Chapter 16: The Block I/O Layer + Network Stack chapters | فهم عام لبنية الـ kernel والـ subsystems |
| **Embedded Linux Primer** — Christopher Hallinan (2nd Ed.) | Chapter 15: Networking | يغطي PHY و MDIO في سياق embedded systems |
| **Understanding Linux Network Internals** — Christian Benvenuti | Part IV: Neighboring Subsystem | الأعمق في الـ Linux networking stack |

---

### مصطلحات البحث

للبحث عن معلومات إضافية، استخدم هذه الـ search terms:

```
phy_link_topology linux kernel
phy_link_topo_add_phy
struct phy_device_node kernel
phylib SFP chained PHY
ETHTOOL_MSG_PHY_GET netlink
phy_upstream enum linux
xarray kernel networking
phylink phy topology userspace
Maxime Chevallier PHY link topology bootlin
netdev mailing list phy topology 2024
```

---

### روابط سريعة للـ Bootlin

**الـ Bootlin** (الشركة اللي طوّرت الـ feature) عندها مصادر ممتازة:

| المصدر | الرابط |
|--------|--------|
| Bootlin blog — phylink tag | [bootlin.com/blog/tag/phylink](https://bootlin.com/blog/tag/phylink/) |
| Elixir cross-reference — phy_link_topology.h | [elixir.bootlin.com](https://elixir.bootlin.com/linux/latest/source/include/linux/phy_link_topology.h) |
| Elixir cross-reference — phy_link_topology.c | [elixir.bootlin.com](https://elixir.bootlin.com/linux/latest/source/drivers/net/phy/phy_link_topology.c) |

> **الـ Elixir cross-reference** هو أسرع طريقة لتتبع كل الأماكن اللي بتستخدم الـ `phy_link_topo_add_phy` أو `struct phy_link_topology` في الـ kernel.
## Phase 8: Writing simple module

### الفكرة

**`phy_link_topo_add_phy`** هي الدالة الأكثر إثارة للاهتمام في الملف — بتتاخد كل مرة PHY device بيتضاف لـ link topology بتاع net_device. باستخدام **kprobe** نقدر نعترض الـ call دي ونطبع اسم الـ interface، نوع الـ upstream، وعنوان الـ PHY device.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on phy_link_topo_add_phy()
 *
 * Intercepts every time a PHY is registered into a net_device's
 * link topology and prints device name + upstream type.
 */

/* --- Includes --- */
#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kprobes.h>      /* kprobe API */
#include <linux/netdevice.h>    /* struct net_device */
#include <linux/phy.h>          /* struct phy_device, phy_upstream enum */
#include <linux/phy_link_topology.h> /* PHY_UPSTREAM_MAC, PHY_UPSTREAM_PHY */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Study <study@example.com>");
MODULE_DESCRIPTION("kprobe demo: trace phy_link_topo_add_phy() calls");

/*
 * phy_link_topo_add_phy signature:
 *   int phy_link_topo_add_phy(struct net_device *dev,
 *                              struct phy_device *phy,
 *                              enum phy_upstream upt,
 *                              void *upstream);
 *
 * On x86-64 the arguments arrive in: di, si, dx, cx
 * regs->di  = dev
 * regs->si  = phy
 * regs->dx  = upt
 */

/* --- pre-handler: runs before the real function executes --- */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Pull arguments from CPU registers (x86-64 calling convention) */
    struct net_device  *dev = (struct net_device *)regs->di;
    struct phy_device  *phy = (struct phy_device  *)regs->si;
    enum phy_upstream   upt = (enum phy_upstream   )regs->dx;

    /* Safety: both pointers must be valid kernel addresses */
    if (!dev || !phy)
        return 0;

    pr_info("phy_topo_probe: netdev=\"%s\" phy_addr=%d upstream=%s\n",
            dev->name,
            phy->mdio.addr,
            upt == PHY_UPSTREAM_MAC ? "MAC" : "PHY");

    return 0;  /* 0 means "continue to the real function" */
}

/* --- kprobe struct: names the function we want to probe --- */
static struct kprobe kp = {
    .symbol_name = "phy_link_topo_add_phy",
    .pre_handler = handler_pre,
};

/* --- module_init: register the kprobe --- */
static int __init phy_topo_probe_init(void)
{
    int ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("phy_topo_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }
    pr_info("phy_topo_probe: planted at %s (%px)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* --- module_exit: unregister to avoid dangling hook --- */
static void __exit phy_topo_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("phy_topo_probe: removed\n");
}

module_init(phy_topo_probe_init);
module_exit(phy_topo_probe_exit);
```

---

### Makefile

```makefile
obj-m := phy_topo_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | الـ macros الأساسية لأي kernel module |
| `linux/kprobes.h` | تعريف `struct kprobe` وكل الـ API بتاعها |
| `linux/netdevice.h` | تعريف `struct net_device` عشان نوصل لـ `dev->name` |
| `linux/phy.h` | تعريف `struct phy_device` وحقل `mdio.addr` |
| `linux/phy_link_topology.h` | تعريف `enum phy_upstream` وقيمها |

**الـ** `linux/kprobes.h` ضروري عشان من غيره مفيش تعريف لـ `struct kprobe` ولا `register_kprobe`.

---

#### الـ `handler_pre`

الـ callback بيتنفذ **قبل** ما الـ function الأصلية تشتغل. بنسحب الـ arguments من الـ registers مباشرةً عشان الـ kprobe بيعمل intercept على مستوى الـ CPU قبل الـ stack frame يكتمل، وده الطريقة الصح على x86-64 (System V ABI: `rdi`, `rsi`, `rdx`, `rcx`).

الـ `pr_info` بتطبع اسم الـ interface (زي `eth0`)، الـ PHY address على الـ MDIO bus، ونوع الـ upstream (هل upstream هو MAC ولا PHY تاني).

---

#### الـ `struct kprobe`

**الـ** `symbol_name` هو الاسم الـ exported للدالة زي ما بيظهر في `/proc/kallsyms`. الـ kernel بيحل العنوان الفعلي وقت الـ `register_kprobe`. لو الدالة مش موجودة أو مش exported، الـ register هيفشل بـ `-EINVAL`.

---

#### الـ `module_exit` وأهمية الـ unregister

لو المودول اتحط من الذاكرة من غير ما الـ kprobe يتشال، الـ kernel هيحاول يكال callback في عنوان مش موجود → **kernel panic**. الـ `unregister_kprobe` بتشيل الـ breakpoint وبتضمن إن مفيش call جديد للـ handler بعد ما المودول اترفع.

---

### تجربة عملية

```bash
# بناء وتحميل المودول
make
sudo insmod phy_topo_probe.ko

# شغّل أي حاجة بتضيف PHY (مثلاً NetworkManager reset أو:)
sudo ip link set eth0 down && sudo ip link set eth0 up

# شوف الـ output
sudo dmesg | grep phy_topo_probe

# تفريغ المودول
sudo rmmod phy_topo_probe
```

**مثال على الـ output:**

```
[  123.456789] phy_topo_probe: planted at phy_link_topo_add_phy (ffffffffc0a12340)
[  125.001234] phy_topo_probe: netdev="eth0" phy_addr=1 upstream=MAC
[  130.998765] phy_topo_probe: removed
```
