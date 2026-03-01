## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي ليه الملف

**الـ `include/linux/of_net.h`** جزء من **ETHERNET PHY LIBRARY** — المكتبة اللي بتدير كل حاجة بين الـ MAC (المتحكم في الشبكة داخل الـ SoC) والـ PHY (الشريحة المسؤولة عن الإشارة الفعلية على السلك).

---

### القصة: إزاي الـ Embedded Linux بيعرف يوصّل الـ Ethernet

تخيل معايا إنك بتصمم راوتر صغير أو كاميرا IP أو Raspberry Pi. الجهاز ده فيه:

- **MAC** (Media Access Controller): جزء من الـ SoC نفسه — بيتعامل مع الـ packets على مستوى البرنامج.
- **PHY chip**: شريحة خارجية صغيرة — بتحوّل الإشارات الرقمية لإشارات كهربائية على السلك الفعلي.

الاتنين دول محتاجين يتكلموا مع بعض، بس في مشكلتين:

**المشكلة الأولى — إيه الـ interface mode؟**
الـ MAC والـ PHY ممكن يتكلموا مع بعض بطرق مختلفة جداً: MII، RMII، RGMII، SGMII، وعشرات غيرهم. كل بورد ليها اختيار مختلف حسب التصميم. السؤال: إزاي الـ driver بيعرف المطلوب؟

**المشكلة التانية — إيه الـ MAC address؟**
كل device لازم يكون ليها MAC address فريد. في embedded boards كتير، الـ MAC address مش محفوظ في EEPROM زي ما بيحصل في laptops — أحياناً بيتحط في الـ Device Tree أو في NVMEM (flash storage).

**الحل هو الـ Device Tree (OF = Open Firmware):**
الـ Device Tree ملف نصي بيوصف الـ hardware للـ kernel. مثلاً:

```dts
ethernet@10000000 {
    compatible = "vendor,eth-mac";
    phy-mode = "rgmii-id";
    local-mac-address = [00 11 22 33 44 55];
    nvmem-cells = <&mac_address>;
    nvmem-cell-names = "mac-address";
};
```

الـ `of_net.h` بيقدّم helper functions تقرأ المعلومات دي من الـ Device Tree وتديها للـ driver بشكل نضيف.

---

### الهدف من الملف

**الـ `include/linux/of_net.h`** هو header بسيط (49 سطر بس) بيعلن عن **4 functions** هي الجسر بين الـ Device Tree والـ networking subsystem:

| Function | بتعمل إيه |
|---|---|
| `of_get_phy_mode()` | تقرأ `phy-mode` أو `phy-connection-type` من الـ DT وترجع `phy_interface_t` |
| `of_get_mac_address()` | تدوّر على MAC address في الـ DT (بترتيب أولوية) |
| `of_get_mac_address_nvmem()` | تجيب MAC address من NVMEM cell |
| `of_get_ethdev_address()` | wrapper بتجيب MAC وتحطه مباشرة في الـ `net_device` |
| `of_find_net_device_by_node()` | تلاقي الـ `net_device` المرتبط بـ DT node معين |

---

### ليه محتاجينه؟

**بدون الـ `of_net.h`:** كل driver Ethernet لازم يكتب كود خاص بيه يقرأ الـ Device Tree، يتعامل مع fallbacks، يتحقق من صحة الـ MAC address — كود متكرر وعرضة للأخطاء في كل driver.

**بعد الـ `of_net.h`:** driver كامل ممكن يتعامل مع الموضوع في سطرين:

```c
/* Get PHY interface type from Device Tree */
ret = of_get_phy_mode(np, &priv->phy_interface);

/* Get and set MAC address from Device Tree or NVMEM */
of_get_ethdev_address(np, netdev);
```

---

### سيناريو حقيقي — أولوية البحث عن MAC Address

`of_get_mac_address()` بتدوّر بالترتيب ده:

```
1. "mac-address"        ← آخر MAC اتحط (مثلاً U-Boot حدّثه)
        ↓ مش موجود؟
2. "local-mac-address"  ← الـ MAC الافتراضي في الـ DTS
        ↓ مش موجود؟
3. "address"            ← legacy property قديمة
        ↓ مش موجود؟
4. NVMEM cell "mac-address" ← محفوز في flash/EEPROM خارجي
        ↓ مش موجود؟
   ← error
```

ده بيحل مشكلة الـ embedded boards القديمة اللي U-Boot بتاعها كان بيحدّث property واحدة بس.

---

### الـ `#if defined(CONFIG_OF) && defined(CONFIG_NET)` Guard

الملف بيستخدم conditional compilation ذكي:

- لو الـ kernel مُفعّل فيه **CONFIG_OF** (Open Firmware / Device Tree support) **و** **CONFIG_NET** (networking) → بيوفّر الـ functions الحقيقية.
- غير كده → بيوفّر **stub functions** ترجع `-ENODEV`، عشان الكود اللي بيستخدمهم ميتكسرش على architectures من غير Device Tree.

---

### الملفات المرتبطة اللي المفروض تعرفها

#### Core Files
| الملف | الدور |
|---|---|
| `include/linux/of_net.h` | الملف اللي احنا بندرسه — declarations |
| `net/core/of_net.c` | الـ implementation الفعلي للـ functions |
| `include/linux/of.h` | الـ Device Tree API الأساسي (`device_node`, `of_property_read_string`) |
| `include/linux/phy.h` | تعريف `phy_interface_t` والـ PHY framework كله |

#### PHY & MDIO Layer
| الملف | الدور |
|---|---|
| `drivers/net/mdio/of_mdio.c` | OF helpers للـ MDIO bus (بروتوكول التحكم في الـ PHY) |
| `drivers/net/mdio/fwnode_mdio.c` | firmware-agnostic MDIO helpers |
| `drivers/net/phy/` | كل drivers الـ PHY chips (Broadcom, Marvell, إلخ) |

#### Documentation & Bindings
| الملف | الدور |
|---|---|
| `Documentation/devicetree/bindings/net/ethernet-phy.yaml` | الـ DT bindings الرسمية للـ PHY |
| `Documentation/devicetree/bindings/net/mdio*` | الـ DT bindings للـ MDIO bus |
| `Documentation/networking/phy.rst` | توثيق الـ PHY framework |

#### NVMEM Integration
| الملف | الدور |
|---|---|
| `drivers/nvmem/` | الـ NVMEM subsystem اللي بيقرأ الـ MAC من flash |
| `include/linux/nvmem-consumer.h` | API للوصول لـ NVMEM cells |

---

### ASCII Diagram — مكان الـ `of_net.h` في النظام

```
   Device Tree (.dts)
         │
         │  phy-mode = "rgmii-id"
         │  local-mac-address = [...]
         │
         ▼
   ┌─────────────────────┐
   │     of_net.h API    │  ← نحن هنا
   │  of_get_phy_mode()  │
   │  of_get_mac_address │
   └─────────┬───────────┘
             │
    ┌────────┴─────────┐
    │                  │
    ▼                  ▼
  MAC Driver         PHY Framework
 (net_device)      (phy_interface_t)
    │                  │
    └────────┬─────────┘
             │
             ▼
       MDIO Bus → PHY Chip → Ethernet Cable
```

الـ `of_net.h` هو الجسر الصغير اللي بيحوّل المعلومات الـ static في الـ Device Tree لـ runtime configuration تشتغل بيها الـ network drivers.
## Phase 2: شرح الـ OF-Net (Open Firmware Network Helpers) Framework

### المشكلة اللي بيحلها

في الـ embedded Linux، عندك Ethernet MAC controller جوه الـ SoC. المشكلة إن الـ driver بتاع الـ MAC محتاج يعرف معلومات hardware-specific مش موجودة في الـ register map:

1. **الـ PHY interface type** — إيه البروتوكول اللي بيربط الـ MAC بالـ PHY؟ RGMII؟ RMII؟ SGMII؟ الاختيار الغلط بيخلي الـ link مش شغال خالص.
2. **الـ MAC address** — مين صاحب الجهازده؟ ممكن تكون محفوظة في NVMEM (EEPROM)، أو في الـ Device Tree نفسه، أو في vendor-specific register.
3. **ربط الـ `net_device` بالـ `device_node`** — لو عندك أكتر من Ethernet interface، إزاي تلاقي الـ software object اللي يقابل node معين في الـ DT؟

من غير framework موحد، كل driver كان بيعمل code خاص بيه لقراءة الـ DT يدوياً — duplication، inconsistency، وبالتالي bugs.

---

### الحل اللي الـ kernel بياخده

الـ kernel بيقدم **OF-Net helpers** — طبقة رفيعة فوق الـ OF (Open Firmware / Device Tree) framework بتعمل abstraction لقراءة network-specific properties من الـ Device Tree. الـ API بسيطة وصغيرة عمداً لأن هدفها واحد بس: **ترجمة DT properties لـ C types بتستخدمها الـ networking stack**.

الفكرة الجوهرية: بدل ما كل driver يعرف فين الـ MAC address محفوظة وإزاي يقراها، الـ OF-Net helpers بيتعاملوا مع كل مصادر البيانات دي بالترتيب.

---

### التشابه من الواقع: بطاقة التعريف في المصنع

تخيل إن عندك خط إنتاج في مصنع، وكل ماكينة جديدة بتتركب فيه لازم:

- تعرف **الكابل اللي هتتوصل بيه** للشبكة الداخلية (ده الـ PHY interface mode — RGMII, RMII, إلخ).
- تعرف **رقم التسلسل الخاص بها** (ده الـ MAC address).
- يكون في **قائمة مركزية** بتربط كل ماكينة برقمها في خريطة المصنع (ده ربط الـ `net_device` بالـ `device_node`).

الـ OF-Net helpers هم الـ "مسؤول اللوجستي" اللي بيجيب كل المعلومات دي من **ملف تصميم المصنع** (الـ Device Tree) بدل ما كل ماكينة تدور عليها بنفسها.

التطابق التفصيلي:

| التشابه | المقابل الحقيقي |
|---|---|
| ملف تصميم المصنع | الـ Device Tree Blob (DTB) |
| مسؤول اللوجستي | دوال `of_get_*` في `of_net.h` |
| نوع الكابل (RJ45 / Fiber / إلخ) | `phy_interface_t` (RGMII, RMII, ...) |
| رقم التسلسل على اللوحة أو في safe | MAC address في DT أو NVMEM |
| قائمة الماكينات في المصنع | `of_find_net_device_by_node()` |
| الماكينة نفسها | `struct net_device` |
| موقع الماكينة في خريطة المصنع | `struct device_node` |

---

### الـ Big Picture Architecture

```
+----------------------------------------------------------+
|                    Ethernet Driver                       |
|  (مثلاً: stmmac, fec, mvneta, dwmac)                    |
|                                                          |
|   probe()  {                                             |
|     of_get_phy_mode(np, &iface);    <-- يقرأ DT         |
|     of_get_ethdev_address(np, dev); <-- يقرأ MAC        |
|   }                                                      |
+--------------------+-----+--------------------------------+
                     |     |
          +----------+     +----------+
          |                           |
+---------v---------+    +------------v-----------+
|   OF-Net Helpers  |    |   OF-Net Helpers        |
| (linux/of_net.h)  |    | (linux/of_net.h)        |
|                   |    |                          |
| of_get_phy_mode() |    | of_get_ethdev_address()  |
|                   |    | of_get_mac_address()     |
|                   |    | of_get_mac_address_nvmem()|
+---------+---------+    +------------+-------------+
          |                           |
+---------v---------+    +------------v-----------+
|   OF Framework    |    |   OF Framework +        |
| (linux/of.h)      |    |   NVMEM Subsystem       |
|                   |    |                          |
| device_node       |    | device_node              |
| struct property   |    | struct property          |
| of_property_read* |    | nvmem_device_read()      |
+---------+---------+    +------------+-------------+
          |                           |
+---------v---------------------------v-----------+
|              Device Tree (DTB)                  |
|                                                 |
|  ethernet@ff700000 {                            |
|    compatible = "snps,dwmac";                   |
|    phy-mode = "rgmii-id";         <-- phy mode  |
|    mac-address = [00 11 22 33 44 55]; <-- MAC   |
|    nvmem-cells = <&mac_addr>;     <-- or NVMEM  |
|  };                                             |
+-------------------------------------------------+
          |
+---------v---------+
|   PHY Subsystem   |
| (linux/phy.h)     |
|                   |
| phy_interface_t   |  <-- enum بيحدد نوع الـ interface
+-------------------+
```

---

### الـ Subsystems اللي لازم تفهمهم الأول

- **OF (Open Firmware) / Device Tree framework** — الـ `linux/of.h` هو الأساس. بيوفر `struct device_node` و `struct property` وكل دوال قراءة الـ DT. الـ OF-Net هو wrapper فوقيه بس.
- **NVMEM Subsystem** — بيوفر واجهة موحدة للوصول لـ Non-Volatile Memory (EEPROM, eFuses, إلخ). الـ `of_get_mac_address_nvmem()` بيستخدمه لما الـ MAC محفوظة في EEPROM خارجي.
- **PHY Subsystem** — الـ `linux/phy.h` بيعرف `phy_interface_t` اللي هو الـ enum الأساسي اللي الـ `of_get_phy_mode()` بيرجعه.

---

### الـ Core Abstraction: قراءة DT Properties بـ Type Safety

الـ abstraction الأساسية في `of_net.h` هي **ترجمة strings وbytes خام من الـ DT لـ typed C values** مفيدة للـ networking stack.

#### الـ Struct الأساسية

الـ OF-Net بيتعامل مع structين رئيسيين موروثين من الـ OF framework:

```c
/*
 * property: عقدة واحدة من خصائص الـ device node
 * (مثلاً: phy-mode = "rgmii-id" أو mac-address = [00 11 22 ...])
 */
struct property {
    char  *name;       /* اسم الـ property: "phy-mode", "mac-address" */
    int    length;     /* طول الـ value بالـ bytes */
    void  *value;      /* القيمة الخام: string أو binary */
    struct property *next; /* linked list للـ properties */
};

/*
 * device_node: node كاملة في الـ Device Tree
 * تمثل جهاز واحد (مثلاً: Ethernet controller واحد)
 */
struct device_node {
    const char       *name;        /* اسم الـ node */
    phandle           phandle;     /* رقم مرجعي فريد داخل الـ DT */
    const char       *full_name;   /* المسار الكامل: /soc/ethernet@ff700000 */
    struct fwnode_handle fwnode;   /* abstraction handle موحد */
    struct property  *properties;  /* linked list للخصائص */
    struct device_node *parent;    /* الـ node الأب */
    struct device_node *child;     /* أول ابن */
    struct device_node *sibling;   /* الأخ الجاي */
};
```

العلاقة بين الـ structs:

```
device_node  (ethernet@ff700000)
    |
    +---> properties (linked list)
    |         |
    |         +--> { name="compatible", value="snps,dwmac" }
    |         |         |
    |         |         v
    |         +--> { name="phy-mode", value="rgmii-id" }
    |         |         |
    |         |         v
    |         +--> { name="mac-address", value=[00 11 22 33 44 55] }
    |         |         |
    |         |         v
    |         +--> { name="nvmem-cells", value=<phandle> }
    |                   |
    |                   v (NULL = نهاية الـ list)
    |
    +---> parent  --> /soc node
    +---> child   --> (no children)
    +---> sibling --> next device in /soc
```

---

### الـ API بالتفصيل

#### 1. `of_get_phy_mode()`

```c
/*
 * بيقرأ "phy-mode" أو "phy-connection-type" من الـ DT
 * ويحوّلها من string لـ enum phy_interface_t
 * مثال: "rgmii-id" --> PHY_INTERFACE_MODE_RGMII_ID
 */
extern int of_get_phy_mode(struct device_node *np, phy_interface_t *interface);
```

الـ `phy_interface_t` هو enum بيحدد الـ electrical/protocol interface بين الـ MAC والـ PHY:

```c
typedef enum {
    PHY_INTERFACE_MODE_NA,         /* مش محدد */
    PHY_INTERFACE_MODE_RGMII,      /* RGMII بدون delay - الأشيع في ARM SoCs */
    PHY_INTERFACE_MODE_RGMII_ID,   /* RGMII + internal TX+RX delay */
    PHY_INTERFACE_MODE_RGMII_RXID, /* RGMII + internal RX delay فقط */
    PHY_INTERFACE_MODE_RGMII_TXID, /* RGMII + internal TX delay فقط */
    PHY_INTERFACE_MODE_RMII,       /* Reduced MII - أبطأ، أقل pins */
    PHY_INTERFACE_MODE_SGMII,      /* Serial GbE - شائع في SFP modules */
    PHY_INTERFACE_MODE_MII,        /* Classic MII */
    /* ... وكتير غيرهم */
} phy_interface_t;
```

**ليه الـ delay مهم؟** في RGMII، الـ clock والـ data لازم يوصلوا في نفس الوقت. التأخير الناتج عن PCB traces ممكن يسبب setup/hold violations. الـ `RGMII_ID` بيقول للـ MAC/PHY إنهم يضيفوا delay داخلي يعوض ده. لو اخترت النوع الغلط، الـ link بيقوم وبيوقع في نفس الوقت أو مش بيقوم خالص.

مثال استخدام في driver:

```c
static int dwmac_probe(struct platform_device *pdev)
{
    struct device_node *np = pdev->dev.of_node;
    phy_interface_t interface;
    int ret;

    /* اقرأ نوع الـ interface من الـ DT */
    ret = of_get_phy_mode(np, &interface);
    if (ret) {
        /* fallback لو مش موجودة */
        interface = PHY_INTERFACE_MODE_MII;
    }

    /* استخدم الـ interface في ضبط الـ MAC hardware */
    stmmac_set_mac_interface(priv, interface);
}
```

#### 2. `of_get_mac_address()` و `of_get_mac_address_nvmem()`

```c
/*
 * بيقرأ الـ MAC address من الـ DT مباشرةً
 * بيدور على properties بالترتيب:
 *   1. "mac-address"        (runtime address, بيتحدت بـ bootloader)
 *   2. "local-mac-address"  (static address مضمنة في الـ DT source)
 */
extern int of_get_mac_address(struct device_node *np, u8 *mac);

/*
 * بيقرأ الـ MAC address من NVMEM مرتبط بالـ DT node
 * بيستخدم "nvmem-cells" property عشان يلاقي الـ NVMEM cell
 */
extern int of_get_mac_address_nvmem(struct device_node *np, u8 *mac);
```

**الفرق بين المصدرين:**

| المصدر | الـ DT Property | الاستخدام |
|---|---|---|
| DT مباشر | `mac-address` | الـ bootloader كتبها قبل بدء الـ kernel |
| DT مباشر | `local-mac-address` | مكتوبة ثابتة في الـ DTS source |
| NVMEM | `nvmem-cells = <&mac_addr>` | EEPROM خارجي أو eFuses داخل الـ SoC |

مثال DT لكل مصدر:

```dts
/* مصدر 1: مكتوبة في الـ DT */
ethernet@ff700000 {
    local-mac-address = [00 0A 35 00 12 34];
};

/* مصدر 2: بيقرأ من EEPROM عبر NVMEM */
ethernet@ff700000 {
    nvmem-cells = <&mac_address>;
    nvmem-cell-names = "mac-address";
};

/* في مكان تاني في الـ DT */
eeprom@50 {
    compatible = "atmel,24c02";
    reg = <0x50>;
    #address-cells = <1>;
    #size-cells = <1>;

    mac_address: mac-address@FA {
        reg = <0xFA 6>;  /* 6 bytes من offset 0xFA */
    };
};
```

#### 3. `of_get_ethdev_address()`

```c
/*
 * wrapper بيجمع بين of_get_mac_address + eth_hw_addr_set
 * بيحط الـ MAC address مباشرةً في الـ net_device
 * أحدث وأوضح من استخدام of_get_mac_address() مباشرة
 */
int of_get_ethdev_address(struct device_node *np, struct net_device *dev);
```

الاستخدام المثالي في driver:

```c
/* الطريقة القديمة - ممل وفيها boilerplate */
u8 mac[ETH_ALEN];
ret = of_get_mac_address(np, mac);
if (!ret)
    eth_hw_addr_set(dev, mac);

/* الطريقة الجديدة - نظيفة ومباشرة */
ret = of_get_ethdev_address(np, dev);
if (ret)
    eth_hw_addr_random(dev); /* fallback: random MAC */
```

#### 4. `of_find_net_device_by_node()`

```c
/*
 * بيبحث في كل الـ net_devices المسجلة
 * ويرجع اللي مرتبطة بالـ device_node ده
 * مفيد لو عندك switch أو bridge محتاج يلاقي الـ uplink interface
 */
extern struct net_device *of_find_net_device_by_node(struct device_node *np);
```

---

### تصميم الـ Compile-time Guards

الـ `of_net.h` فيه pattern مهم جداً في الـ kernel:

```c
#if defined(CONFIG_OF) && defined(CONFIG_NET)
/* التعريفات الحقيقية */
extern int of_get_phy_mode(...);
extern int of_get_mac_address(...);
/* ... */
#else
/* stub functions بترجع error */
static inline int of_get_phy_mode(struct device_node *np,
                                   phy_interface_t *interface)
{
    return -ENODEV;  /* مفيش DT أو مفيش networking */
}
/* ... */
#endif
```

**ليه ده مهم؟**

- `CONFIG_OF` — بيتفعّل لما الـ platform بيستخدم Device Tree (ARM, RISC-V، إلخ). مش موجود في x86 عادةً.
- `CONFIG_NET` — بيتفعّل لما الـ networking stack متضمنة في الـ build.

لو أي منهم مش موجود، الـ compiler بيشتغل مع stub functions بدل ما يتعطل. الـ driver بيتعامل مع الـ `-ENODEV` return ويعمل fallback. ده بيضمن إن نفس الـ driver code يشتغل على أنظمة بـ DT وبدونه.

---

### إيه اللي الـ OF-Net بيمتلكه مقابل إيه اللي بيفوّضه للـ Drivers

| المسؤولية | تبع مين؟ | السبب |
|---|---|---|
| قراءة `"phy-mode"` string وتحويلها لـ enum | **OF-Net** | logic موحد لكل الـ drivers |
| تحديد أولوية المصادر للـ MAC address | **OF-Net** | الترتيب (mac-address ← local-mac-address ← nvmem) يجب أن يكون موحداً |
| تطبيق الـ MAC على الـ hardware registers | **Driver** | كل MAC controller بيه طريقته الخاصة |
| ضبط الـ PHY timing/clocking بناءً على الـ mode | **Driver** | hardware-specific تماماً |
| إنشاء ربط الـ `net_device` بالـ `device_node` | **Driver** عند الـ probe | الـ OF-Net بس بيبحث، مش بيربط |
| fallback لو مفيش MAC في الـ DT | **Driver** | قرار policy (random MAC؟ hardcoded؟) |
| التحقق من صحة الـ MAC (unicast؟ multicast؟) | **OF-Net + Networking core** | مشترك |

---

### مثال واقعي كامل: STM32 Ethernet Driver

الـ SoC: STM32MP1 — ARM Cortex-A7 فيه Ethernet MAC داخلي متوصل بـ PHY خارجي عبر RGMII.

**الـ DTS:**

```dts
/* arch/arm/boot/dts/stm32mp15xx-dhcor-som.dtsi */
&ethernet0 {
    status = "okay";
    pinctrl-0 = <&ethernet0_rgmii_pins_a>;
    phy-mode = "rgmii-id";                    /* <-- of_get_phy_mode */
    mac-address = [00 80 E1 42 05 81];        /* <-- of_get_mac_address */
    phy-handle = <&phy0>;

    mdio0 {
        phy0: ethernet-phy@0 {
            reg = <0>;
        };
    };
};
```

**الـ Driver (stmmac):**

```c
static int stmmac_dt_phy(struct plat_stmmacenet_data *plat,
                         struct device_node *np,
                         struct device *dev)
{
    phy_interface_t iface;

    /* 1. اقرأ الـ PHY mode */
    if (of_get_phy_mode(np, &iface) == 0)
        plat->phy_interface = iface;
        /* = PHY_INTERFACE_MODE_RGMII_ID */

    /* 2. اقرأ الـ MAC address */
    if (of_get_ethdev_address(np, ndev) < 0)
        eth_hw_addr_random(ndev); /* fallback */

    /* 3. ابحث عن PHY node */
    plat->phylink_node = of_parse_phandle(np, "phy-handle", 0);

    return 0;
}
```

---

### رسم العلاقة الكاملة بين الـ Structs

```
Device Tree في الـ Memory
─────────────────────────
device_node ["ethernet@ff700000"]
│
├── full_name = "/soc/ethernet@ff700000"
├── phandle   = 0x12
├── fwnode    ──────────────────────> fwnode_handle
│                                    (يربطه بالـ device model)
│
└── properties (linked list)
    │
    ├── property { name="compatible",  value="snps,dwmac-4.20a" }
    │       │
    ├── property { name="phy-mode",    value="rgmii-id" }
    │       │          ↑
    │       │     of_get_phy_mode() بيقرأ ده
    │       │     ويحوله لـ PHY_INTERFACE_MODE_RGMII_ID
    │       │
    ├── property { name="mac-address", value=[00 11 22 33 44 55] }
    │       │          ↑
    │       │     of_get_mac_address() بيقرأ ده
    │       │     of_get_ethdev_address() بيحطه في net_device
    │       │
    └── property { name="nvmem-cells", value=<phandle:0x25> }
                       ↑
                 of_get_mac_address_nvmem() بيتبع الـ phandle
                 للـ NVMEM node ويقرأ منه الـ MAC


net_device ["eth0"]
──────────────────
│  dev_addr[6] = { 00, 11, 22, 33, 44, 55 }   <-- من of_get_ethdev_address
│  dev.of_node ──────────────────────────────> device_node أعلاه
│  phydev ───────────────────────────────────> phy_device
│              │
│              └── interface = PHY_INTERFACE_MODE_RGMII_ID  <-- من of_get_phy_mode
```
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. Cheatsheet — الـ Enums والـ Config Options والـ Flags المهمة

#### Config Options التي تتحكم في الـ API

| Option | المعنى |
|---|---|
| `CONFIG_OF` | تفعيل دعم Open Firmware / Device Tree |
| `CONFIG_NET` | تفعيل networking stack |
| `CONFIG_OF && CONFIG_NET` | الشرط اللي يفعّل كل الـ API الحقيقي في `of_net.h` |
| `CONFIG_OF_DYNAMIC` | يتيح إضافة/حذف nodes ديناميكيًا (overlays) |
| `CONFIG_OF_KOBJ` | يربط كل `device_node` بـ kobject في sysfs |
| `CONFIG_OF_PROMTREE` | دعم PROM tree على SPARC |

عند غياب `CONFIG_OF` أو `CONFIG_NET`، كل دوال `of_net.h` بتتحول لـ **stub inline** بترجع `-ENODEV` أو `NULL` — مفيش panic، مفيش crash.

---

#### `device_node._flags` — Bit Flags

| Flag | Value (bit#) | المعنى |
|---|---|---|
| `OF_DYNAMIC` | 1 | الـ node والـ properties اتخصصت بـ `kmalloc` |
| `OF_DETACHED` | 2 | الـ node اتفصل عن الـ device tree |
| `OF_POPULATED` | 3 | الـ device اتعمل ليه بالفعل |
| `OF_POPULATED_BUS` | 4 | اتعمل platform bus لأبنائه |
| `OF_OVERLAY` | 5 | اتخصص لـ overlay |
| `OF_OVERLAY_FREE_CSET` | 6 | في overlay cset بيتحذف |

---

#### `phy_interface_t` — Enum كامل (cheatsheet)

**الـ** `phy_interface_t` هو enum بيوصف الـ interface بين الـ MAC والـ PHY.

| القيمة | المعنى العملي |
|---|---|
| `PHY_INTERFACE_MODE_NA` | مش محدد — متلمش |
| `PHY_INTERFACE_MODE_INTERNAL` | MAC وPHY مدمجين، مفيش wire |
| `PHY_INTERFACE_MODE_MII` | MII كلاسيك (4-bit، 25MHz، 100Mbps) |
| `PHY_INTERFACE_MODE_GMII` | Gigabit MII (8-bit، 125MHz) |
| `PHY_INTERFACE_MODE_SGMII` | Serial Gigabit MII |
| `PHY_INTERFACE_MODE_RMII` | Reduced MII (2-bit، 50MHz) |
| `PHY_INTERFACE_MODE_RGMII` | Reduced GMII (4-bit، 125MHz) |
| `PHY_INTERFACE_MODE_RGMII_ID` | RGMII مع internal RX+TX delay |
| `PHY_INTERFACE_MODE_RGMII_RXID` | RGMII مع internal RX delay بس |
| `PHY_INTERFACE_MODE_RGMII_TXID` | RGMII مع internal TX delay بس |
| `PHY_INTERFACE_MODE_XGMII` | 10G MII |
| `PHY_INTERFACE_MODE_XLGMII` | 40G MII |
| `PHY_INTERFACE_MODE_QSGMII` | Quad SGMII (4 ports over 1 serdes) |
| `PHY_INTERFACE_MODE_PSGMII` | Penta SGMII (5 ports) |
| `PHY_INTERFACE_MODE_10GBASER` | 10G BaseR (XFI/SFI) |
| `PHY_INTERFACE_MODE_25GBASER` | 25G BaseR |
| `PHY_INTERFACE_MODE_USXGMII` | Universal 10GE MII |
| `PHY_INTERFACE_MODE_50GBASER` | 50G BaseR |
| `PHY_INTERFACE_MODE_100GBASEP` | 100G BaseP |
| `PHY_INTERFACE_MODE_1000BASEX` | 1000Base-X (fiber) |
| `PHY_INTERFACE_MODE_2500BASEX` | 2.5G Base-X |
| `PHY_INTERFACE_MODE_MAX` | sentinel — مش interface حقيقي |

في الـ DTS، بتحط مثلًا:
```
phy-mode = "rgmii-id";
```
والـ kernel بيعمل lookup في string table ويرجع `PHY_INTERFACE_MODE_RGMII_ID`.

---

### 1. الـ Structs المهمة

#### `struct device_node`  — العمود الفقري

**الهدف:** يمثل node واحدة في الـ Device Tree (DT). كل hardware component في الـ DTS بيبقى له `device_node` في الـ kernel.

```c
struct device_node {
    const char *name;          /* اسم الـ node (e.g. "ethernet@3000000") */
    phandle phandle;           /* رقم unique بيستخدمه DT للـ cross-references */
    const char *full_name;     /* المسار الكامل e.g. "/soc/ethernet@3000000" */
    struct fwnode_handle fwnode; /* abstraction layer بين OF وبقية الـ kernel */

    struct property *properties; /* linked list بالـ properties */
    struct property *deadprops;  /* properties اتحذفت (للـ overlay rollback) */
    struct device_node *parent;  /* الـ node الأب */
    struct device_node *child;   /* أول ابن */
    struct device_node *sibling; /* الأخ التالي */

    struct kobject kobj;         /* (إذا CONFIG_OF_KOBJ) entry في /sys/firmware/devicetree/ */
    unsigned long _flags;        /* OF_DYNAMIC, OF_POPULATED, ... */
    void *data;                  /* platform-specific data */
};
```

**الاتصالات:**
- بيحتوي على linked list من `struct property`
- بيعمل embed لـ `struct fwnode_handle` —ده الـ bridge لـ `struct device` العادية
- الـ `of_net.h` API كله بياخد `device_node *np` كأول argument

---

#### `struct property` — الـ key-value لكل node

**الهدف:** يخزن property واحدة من الـ DTS (اسم + بيانات خام).

```c
struct property {
    char  *name;          /* e.g. "mac-address", "phy-mode" */
    int    length;        /* طول الـ value بالـ bytes */
    void  *value;         /* raw bytes (big-endian كما في DTS) */
    struct property *next; /* التالي في الـ linked list */
};
```

**مثال عملي:** لما `of_get_mac_address()` بتشتغل، بتدور على property اسمها `"mac-address"` أو `"local-mac-address"` أو `"address"` — كلهم `struct property` جوه الـ node.

---

#### `struct net_device` — (forward declaration فقط في `of_net.h`)

**الهدف:** يمثل الـ network interface في الـ kernel. `of_net.h` بيعمله forward declaration بس، لأنه محتاجه في `of_get_ethdev_address()` و`of_find_net_device_by_node()`.

**الحقل المهم من ناحية `of_net`:**
```c
/* جوه net_device */
unsigned char dev_addr[MAX_ADDR_LEN]; /* الـ MAC address */
```

`of_get_ethdev_address()` بتكتب الـ MAC من الـ DT مباشرة في `dev->dev_addr`.

---

#### `phy_interface_t` — (typedef enum)

مش struct، بس يُعامَل كـ output parameter:

```c
/* of_get_phy_mode بتكتب في *interface */
phy_interface_t interface;
of_get_phy_mode(np, &interface);
/* دلوقتي interface = PHY_INTERFACE_MODE_RGMII_ID مثلًا */
```

---

### 2. مخطط العلاقات بين الـ Structs

```
Device Tree (binary blob in RAM)
          │
          ▼
┌─────────────────────────────┐
│      struct device_node     │
│  name: "ethernet@3000000"  │
│  full_name: "/soc/eth..."  │
│  fwnode ──────────────────────────────────────────┐
│  properties ──┐                                    │
│  parent  ◄────┼── /soc node                        │
│  child   ────►│── mdio node                        │
│  _flags       │                                    │
└───────────────┼────────────────────────────────────┘
                │
                ▼
        ┌──────────────────┐     ┌──────────────────┐
        │  struct property │────►│  struct property │──► NULL
        │  name: "mac-address"  │  name: "phy-mode"│
        │  length: 6       │     │  length: varies  │
        │  value: [bytes]  │     │  value: "rgmii-id│
        └──────────────────┘     └──────────────────┘

                                    │ fwnode embed
                                    ▼
                         ┌─────────────────────┐
                         │  struct fwnode_handle│
                         │  ops: &of_fwnode_ops │
                         └──────────┬──────────┘
                                    │ container_of
                                    ▼
                         ┌─────────────────────┐
                         │    struct device     │
                         │  (platform device)   │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │  struct net_device   │
                         │  dev_addr[6]: MAC    │
                         └─────────────────────┘
```

---

### 3. مخطط الـ Lifecycle

#### الـ Device Tree Node — من الـ Boot لحد الـ Driver

```
Boot (DT blob في RAM)
       │
       ▼
of_core_init()
  └─► يبني شجرة device_node في الـ kernel
  └─► كل node: kmalloc + of_node_init() → refcount = 1

       │
       ▼
Platform driver يتسجل (e.g. dwmac)
  └─► kernel يعمل match بين driver وnode عن طريق "compatible"
  └─► يستدعي probe(dev) مع struct platform_device

       │
       ▼
داخل probe():
  np = dev->of_node  ← الـ device_node
  │
  ├─► of_get_phy_mode(np, &interface)
  │       → يقرأ property "phy-mode"
  │       → يحول string → phy_interface_t
  │
  ├─► of_get_mac_address(np, mac)
  │       → يجرب: "mac-address" → "local-mac-address" → "address"
  │       → لو مفيش: يجرب nvmem عن طريق of_get_mac_address_nvmem()
  │
  └─► of_get_ethdev_address(np, netdev)
          → يكتب الـ MAC في netdev->dev_addr

       │
       ▼
Driver يسجل الـ net_device:
  register_netdev(netdev)

       │
       ▼
Teardown (rmmod أو unplug):
  unregister_netdev(netdev)
  → of_node_put(np)  ← يخفض refcount
  → لو refcount = 0: of_node_release() → kfree
```

---

### 4. مخطط Call Flow

#### `of_get_phy_mode(np, &interface)`

```
driver probe()
  └─► of_get_phy_mode(np, &interface)
        └─► of_property_read_string(np, "phy-mode", &pm)
              └─► يدور في np->properties linked list
                    └─► لقى property "phy-mode"
                          └─► phy_get_interface_by_name(pm)
                                └─► strcmp loop على phy_modes[] table
                                      └─► بيرجع PHY_INTERFACE_MODE_RGMII_ID
                          └─► يكتب في *interface
        └─► return 0 (نجح) أو -ENODEV / -EINVAL
```

#### `of_get_mac_address(np, mac)`

```
driver probe()
  └─► of_get_mac_address(np, mac_buf)
        ├─► of_property_read_u8_array(np, "mac-address", mac_buf, 6)
        │     → نجح؟ → return 0
        │
        ├─► of_property_read_u8_array(np, "local-mac-address", mac_buf, 6)
        │     → نجح؟ → return 0
        │
        ├─► of_property_read_u8_array(np, "address", mac_buf, 6)
        │     → نجح؟ → return 0
        │
        └─► return -ENODEV (مفيش حاجة)
```

#### `of_get_ethdev_address(np, dev)`

```
driver probe()
  └─► of_get_ethdev_address(np, netdev)
        └─► of_get_mac_address(np, addr)
              → نجح؟
                └─► eth_hw_addr_set(netdev, addr)
                      └─► memcpy(netdev->dev_addr, addr, ETH_ALEN)
              → فشل؟ → return error
```

#### `of_find_net_device_by_node(np)`

```
caller
  └─► of_find_net_device_by_node(np)
        └─► يعمل iterate على كل registered net_devices
              └─► for_each_netdev(net, dev)
                    └─► dev->dev.of_node == np ?
                          → نعم: return dev (مع hold reference)
                          → لا: continue
        └─► مالقاش: return NULL
```

---

### 5. استراتيجية الـ Locking

ملف `of_net.h` نفسه مش بيعرّف locks، لكن الـ functions اللي بيعلن عنها بتعتمد على:

#### Locks المستخدمة داخليًا (في التنفيذ `of_net.c`)

| Lock | النوع | بيحمي إيه |
|---|---|---|
| `of_mutex` (داخل OF core) | `mutex` | قراءة/تعديل الـ `device_node` tree |
| `rtnl_lock` | `mutex` | `for_each_netdev` في `of_find_net_device_by_node` |

#### قواعد الـ Locking

**الـ** `of_get_phy_mode` و`of_get_mac_address`:
- **Read-only** على الـ properties — بيقروا بس.
- الـ properties بعد الـ boot بتبقى immutable في معظم الحالات.
- لو `CONFIG_OF_DYNAMIC` مفعّل (overlays)، الـ OF core بياخد internal `of_mutex` قبل قراءة الـ properties.
- الـ driver نفسه **مش محتاج** يمسك أي lock.

**الـ** `of_find_net_device_by_node`:
- المفروض يتنادى وأنت شايل **`rtnl_lock`** أو داخل context بيضمن إن الـ netdev مش بيتحذف.
- مثال صح:
```c
rtnl_lock();
dev = of_find_net_device_by_node(np);
if (dev) {
    /* استخدم dev هنا */
    dev_put(dev);
}
rtnl_unlock();
```

**الـ** `of_get_ethdev_address`:
- بيكتب في `netdev->dev_addr`.
- المفروض يتنادى من `probe()` فقط — قبل `register_netdev()` — عشان مفيش race مع قراءة الـ MAC من userspace أو الـ stack.
- بعد التسجيل، تغيير `dev_addr` محتاج `RTNL` و`eth_hw_addr_set()`.

#### ترتيب الـ Locks (Lock Ordering)

```
rtnl_lock
    └─► of_mutex  (OF internal)
            └─► property data (read-only بعد boot)
```

**ممنوع** تعكس الترتيب ده (OF mutex قبل RTNL) — هيتعمل deadlock.
## Phase 4: شرح الـ Functions

### ملخص الـ API — Cheatsheet

| Function | Signature Summary | Purpose |
|---|---|---|
| `of_get_phy_mode` | `(node, *interface) → int` | اقرأ `phy-mode` property من DT واحوّلها لـ `phy_interface_t` |
| `of_get_mac_address` | `(node, *mac) → int` | اجيب MAC address من DT properties أو nvmem |
| `of_get_mac_address_nvmem` | `(node, *mac) → int` | اجيب MAC address من nvmem cell فقط |
| `of_get_ethdev_address` | `(node, *dev) → int` | اجيب MAC وسجّله مباشرة في `net_device` |
| `of_find_net_device_by_node` | `(node) → *net_device` | ابحث عن `net_device` مرتبط بـ DT node |

> لو `CONFIG_OF` أو `CONFIG_NET` مش معرّف، كل الـ functions بترجع stub بـ `-ENODEV` أو `NULL`.

---

### التصنيف

```
of_net.h Functions
├── PHY Interface Query
│   └── of_get_phy_mode
├── MAC Address Retrieval
│   ├── of_get_mac_address
│   ├── of_get_mac_address_nvmem
│   └── of_get_ethdev_address
└── Device Lookup
    └── of_find_net_device_by_node
```

---

### فئة 1: PHY Interface Query

الهدف من الفئة دي هو استخراج **phy-mode** (نوع الـ interface بين MAC وPHY) من الـ Device Tree. في الـ embedded platforms زي Raspberry Pi أو NXP i.MX، الـ DTS بيحدد `phy-mode = "rgmii-id"` مثلاً، والـ driver بيحتاج يقرأها وقت الـ probe.

---

#### `of_get_phy_mode`

```c
extern int of_get_phy_mode(struct device_node *np, phy_interface_t *interface);
```

**بتعمل إيه:**
بتقرأ الـ `phy-mode` string property من الـ `device_node` المعطى، وبتحوّلها لقيمة `phy_interface_t` enum مستخدمةً `phy_get_interface_by_name()`. لو الـ property مش موجودة أو القيمة مش معروفة، بترجع error.

**Parameters:**

| Parameter | Type | الوصف |
|---|---|---|
| `np` | `struct device_node *` | الـ DT node اللي فيها `phy-mode` property — عادةً الـ ethernet node نفسه |
| `interface` | `phy_interface_t *` | output — بيتكتب فيه الـ enum value زي `PHY_INTERFACE_MODE_RGMII_ID` |

**Return value:**
- `0` عند النجاح، `*interface` بيتحدّث.
- `-ENODEV` لو مفيش `phy-mode` property.
- `-ENODEV` كمان لو القيمة مش معروفة.

**Key details:**
- الـ function مش بتعمل لوكينق — قراءة read-only من الـ DT tree.
- في `!CONFIG_OF` stub بترجع `-ENODEV` مباشرة بدون أي عمل.
- الـ caller context: driver probe — يُستدعى من `__init` أو `.probe` callback.
- الـ `phy_interface_t` enum معرّفة في `<linux/phy.h>` وبتشمل أكتر من 20 mode زي `MII`, `GMII`, `RGMII`, `SGMII`, `XGMII`, وغيرها.

**Pseudocode:**
```
of_get_phy_mode(np, interface):
    s = of_get_property(np, "phy-mode")
    if !s → return -ENODEV
    mode = phy_get_interface_by_name(s)
    if mode == PHY_INTERFACE_MODE_NA → return -ENODEV
    *interface = mode
    return 0
```

**مثال real-world:**
```c
/* في probe() لـ Ethernet MAC driver */
phy_interface_t iface;
ret = of_get_phy_mode(pdev->dev.of_node, &iface);
if (ret) {
    /* fallback to default MII */
    iface = PHY_INTERFACE_MODE_MII;
}
```

---

### فئة 2: MAC Address Retrieval

الفئة دي هدفها جلب الـ MAC address من مصادر متعددة مرتّبة بحسب الأولوية: static DT properties → nvmem cells → MAC from other sources.

في الـ embedded systems، الـ MAC address بيتخزّن في:
1. **DT property** زي `mac-address` أو `local-mac-address`
2. **nvmem cell** بيشاور على EEPROM أو OTP fuse
3. **reg** property أو `address-bits` (نادر)

---

#### `of_get_mac_address`

```c
extern int of_get_mac_address(struct device_node *np, u8 *mac);
```

**بتعمل إيه:**
بتدوّر على الـ MAC address في ترتيب أولوية محدد: أول حاجة بتجرّب الـ static DT properties (`mac-address` ثم `local-mac-address`)، وبعدين لو مش لاقية بتجرّب الـ nvmem cell. النتيجة بتتكتب في buffer المعطى.

**Parameters:**

| Parameter | Type | الوصف |
|---|---|---|
| `np` | `struct device_node *` | الـ DT node للـ ethernet interface |
| `mac` | `u8 *` | output buffer — لازم يكون 6 bytes على الأقل |

**Return value:**
- `0` عند النجاح، الـ `mac` buffer فيه MAC address صالح.
- `-ENODEV` لو مفيش property ومفيش nvmem.
- `-EINVAL` لو الـ MAC address موجود بس invalid (multicast bit set أو كله أصفار).

**Key details:**
- بتفضّل `mac-address` على `local-mac-address` — الأولى بيحددها الـ bootloader (زي U-Boot) ديناميكياً، والتانية بتيجي من الـ hardware layout.
- الـ nvmem fallback بيستدعي `of_get_mac_address_nvmem()` داخلياً.
- الـ validation بيتحقق إن الـ MAC مش multicast (`mac[0] & 0x01`) ومش كله أصفار.
- Caller context: driver probe فقط.

**Pseudocode:**
```
of_get_mac_address(np, mac):
    /* try static properties first */
    addr = of_get_property(np, "mac-address", &len)
    if addr && len == 6 && is_valid_ether_addr(addr):
        memcpy(mac, addr, 6)
        return 0

    addr = of_get_property(np, "local-mac-address", &len)
    if addr && len == 6 && is_valid_ether_addr(addr):
        memcpy(mac, addr, 6)
        return 0

    /* fallback to nvmem */
    return of_get_mac_address_nvmem(np, mac)
```

---

#### `of_get_mac_address_nvmem`

```c
extern int of_get_mac_address_nvmem(struct device_node *np, u8 *mac);
```

**بتعمل إيه:**
بتحاول تجيب الـ MAC address عن طريق **nvmem framework** بالبحث عن `nvmem-cells` property في الـ DT node المعطى. بتستخدم `nvmem_cell_get()` و`nvmem_cell_read()` لقراءة الـ cell، وبتتحقق من الـ validity.

**Parameters:**

| Parameter | Type | الوصف |
|---|---|---|
| `np` | `struct device_node *` | الـ DT node اللي فيها `nvmem-cells = <&mac_cell>` |
| `mac` | `u8 *` | output buffer — 6 bytes |

**Return value:**
- `0` عند النجاح.
- `-ENODEV` لو مفيش nvmem cell مسمّى "mac-address".
- error من nvmem subsystem لو القراءة فشلت.
- `-EINVAL` لو الـ MAC المقروء invalid.

**Key details:**
- بتدوّر على nvmem cell باسم `"mac-address"` بالتحديد.
- بتستخدم `nvmem_cell_get(dev, "mac-address")` — الـ `dev` بيتجاب من الـ `np` عن طريق `of_find_device_by_node()`.
- الـ cell بيتحرر بعد القراءة بـ `nvmem_cell_put()` — مفيش leak.
- DT مثال:
```dts
ethernet@e0000000 {
    nvmem-cells = <&mac_addr>;
    nvmem-cell-names = "mac-address";
};
```

---

#### `of_get_ethdev_address`

```c
int of_get_ethdev_address(struct device_node *np, struct net_device *dev);
```

**بتعمل إيه:**
Wrapper فوق `of_get_mac_address()` — بتجيب الـ MAC وبعدين بتسجّله مباشرة في الـ `net_device` باستخدام `eth_hw_addr_set()`. بتختصر على الـ driver خطوة اليدوية.

**Parameters:**

| Parameter | Type | الوصف |
|---|---|---|
| `np` | `struct device_node *` | الـ DT node للـ ethernet interface |
| `dev` | `struct net_device *` | الـ network device اللي هيتسجّل فيها الـ MAC |

**Return value:**
- `0` عند النجاح، `dev->dev_addr` اتحدّث.
- نفس error codes بتاعة `of_get_mac_address()`.

**Key details:**
- `eth_hw_addr_set()` بتأمن الـ write على `dev->dev_addr` مع الـ memory barriers المطلوبة.
- مش بتعمل `netif_addr_lock` — لازم تُستدعى قبل register_netdev.
- Caller context: probe فقط قبل ما الـ device يتسجّل.

**Pseudocode:**
```
of_get_ethdev_address(np, dev):
    ret = of_get_mac_address(np, addr)
    if ret:
        return ret
    eth_hw_addr_set(dev, addr)
    return 0
```

**مثال real-world:**
```c
/* في probe() — بدل ما تعمل الخطوتين يدوياً */
ret = of_get_ethdev_address(np, ndev);
if (ret) {
    dev_warn(&pdev->dev, "no MAC in DT, using random\n");
    eth_hw_addr_random(ndev);
}
```

---

### فئة 3: Device Lookup

---

#### `of_find_net_device_by_node`

```c
extern struct net_device *of_find_net_device_by_node(struct device_node *np);
```

**بتعمل إيه:**
بتدوّر في قائمة الـ registered network devices كلها عن طريق `for_each_netdev()` وبترجع أول `net_device` ربط نفسه بالـ DT node المعطى — بتقارن `dev->dev.of_node == np`.

**Parameters:**

| Parameter | Type | الوصف |
|---|---|---|
| `np` | `struct device_node *` | الـ DT node اللي بندوّر على الـ net_device المقابل ليه |

**Return value:**
- `struct net_device *` مع زيادة الـ reference count بـ `dev_hold()` لو لقى.
- `NULL` لو مفيش net_device مرتبط بالـ node ده.

**Key details:**
- الـ caller **لازم يعمل `dev_put()`** على النتيجة لما يخلص منها — عشان الـ `dev_hold()` اللي اتعمل جوّاها.
- بتاخد `rtnl_lock` أو بتستخدم RCU لحماية الـ net_device list — حسب الـ kernel version.
- الـ stub في `!CONFIG_OF` بترجع `NULL` مباشرة.
- Caller context: يمكن تُستدعى من sleepable context — مش IRQ safe.
- الاستخدام الشائع: DSA (Distributed Switch Architecture) أو من OF overlay code لربط الـ PHY بالـ MAC عبر phandle.

**Pseudocode:**
```
of_find_net_device_by_node(np):
    rtnl_lock()
    for each netdev in net_device_list:
        if netdev->dev.of_node == np:
            dev_hold(netdev)
            rtnl_unlock()
            return netdev
    rtnl_unlock()
    return NULL
```

**مثال real-world:**
```c
/* DSA switch driver يربط الـ CPU port بالـ master netdev */
struct device_node *cpu_port_np = of_get_child_by_name(sw_np, "cpu-port");
struct net_device *master = of_find_net_device_by_node(cpu_port_np);
if (!master) {
    return -EPROBE_DEFER; /* master مش probe اتعمله لسه */
}
/* ... استخدم master ... */
dev_put(master);
```

---

### الـ Stub Implementations

لما `CONFIG_OF` أو `CONFIG_NET` مش معرّفين، كل الـ functions بتتحوّل لـ `static inline` stubs:

```c
/* كل دي بترجع فشل فوراً بدون أي logic */
static inline int of_get_phy_mode(...)          { return -ENODEV; }
static inline int of_get_mac_address(...)        { return -ENODEV; }
static inline int of_get_mac_address_nvmem(...) { return -ENODEV; }
static inline int of_get_ethdev_address(...)     { return -ENODEV; }
static inline struct net_device *of_find_net_device_by_node(...) { return NULL; }
```

الـ stubs دي بتضمن إن الكود اللي بيستخدم الـ header ده بيـ compile بدون مشاكل على architectures أو configurations مش بتدعم OF أو networking.

---

### ترتيب الاستدعاء الطبيعي في Ethernet Driver

```
driver probe()
    │
    ├─► of_get_phy_mode(np, &iface)
    │       └── يحدد interface type: RGMII, SGMII, MII, إلخ
    │
    ├─► of_get_ethdev_address(np, ndev)
    │       └── يجيب MAC من DT أو nvmem → يسجّله في net_device
    │           (internally calls of_get_mac_address)
    │
    └─► of_find_net_device_by_node(master_np)   [اختياري - DSA/PHY binding]
            └── يرجع master netdev مع dev_hold
            └── CALLER يعمل dev_put() بعدين
```
## Phase 5: دليل الـ Debugging الشامل

الـ subsystem اللي بنتكلم عنه هو **`of_net`** — مجموعة الـ helpers اللي بتربط الـ **Device Tree (OF = Open Firmware)** بالـ network devices في الـ Linux kernel. الوظايف الأساسية:

- `of_get_phy_mode()` — بتجيب الـ PHY interface mode من الـ DT.
- `of_get_mac_address()` — بتجيب الـ MAC address من الـ DT property.
- `of_get_mac_address_nvmem()` — بتجيب الـ MAC من الـ NVMEM.
- `of_get_ethdev_address()` — بتجيب الـ address وتحطه في الـ `net_device`.
- `of_find_net_device_by_node()` — بتدور على الـ `net_device` المرتبط بـ DT node.

---

### Software Level

#### 1. debugfs

الـ `of_net` مش بيعمل debugfs entries خاصة بيه، لكن الـ OF core بيعرضهم:

```bash
# شوف الـ Device Tree nodes المتاحة
ls /sys/kernel/debug/device-tree/

# اقرأ node معين (مثلاً ethernet)
ls /sys/kernel/debug/device-tree/soc/ethernet@1234/

# اقرأ قيمة property زي phy-mode
xxd /sys/kernel/debug/device-tree/soc/ethernet@1234/phy-mode

# اقرأ local-mac-address كـ hex
xxd /sys/kernel/debug/device-tree/soc/ethernet@1234/local-mac-address
```

الـ output المتوقع لـ `local-mac-address`:

```
00000000: aabb ccdd eeff                           ......
```

يعني الـ MAC هو `aa:bb:cc:dd:ee:ff`.

---

#### 2. sysfs

```bash
# تحقق من الـ MAC address اللي اتسجل في الـ net_device
cat /sys/class/net/eth0/address

# شوف الـ PHY المرتبط
ls /sys/class/net/eth0/phydev/

# اقرأ phy_interface من sysfs (لو الـ driver بيعرضها)
cat /sys/class/net/eth0/phydev/phy_interface

# تحقق من الـ device node المرتبط
readlink /sys/class/net/eth0/device/of_node
# النتيجة المتوقعة:
# ../../../../firmware/devicetree/base/soc/ethernet@1234
```

---

#### 3. ftrace — Tracepoints والـ Events

```bash
# فعّل الـ tracing على الـ OF subsystem
echo 1 > /sys/kernel/debug/tracing/tracing_on

# تابع الـ function calls في of_net
cd /sys/kernel/debug/tracing
echo 'of_get_mac_address' > set_ftrace_filter
echo 'of_get_phy_mode'   >> set_ftrace_filter
echo 'of_get_ethdev_address' >> set_ftrace_filter
echo function > current_tracer
cat trace

# لو عايز تتابع كل الـ OF helpers
echo 'of_*' > set_ftrace_filter
echo function_graph > current_tracer
cat trace | head -50
```

مثال على الـ output:

```
 1)               |  of_get_phy_mode() {
 1)               |    of_property_read_string() {
 1)   0.312 us    |    }
 1)   1.105 us    |  }
```

---

#### 4. printk / Dynamic Debug

```bash
# فعّل الـ dynamic debug لكل ملفات of_net
echo 'file of_net.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل debug للـ of.c الأساسي (بيشمل of_property_read_*)
echo 'file of.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل debug للـ nvmem اللي بيستخدمه of_get_mac_address_nvmem
echo 'file nvmem-core.c +p' > /sys/kernel/debug/dynamic_debug/control

# شوف النتايج في dmesg
dmesg -w | grep -E 'of_net|mac_address|phy_mode'
```

لو عايز تفعل من الـ kernel cmdline:

```
dyndbg="file of_net.c +p; file of.c +p"
```

---

#### 5. Kernel Config Options للـ Debugging

| Config | الوظيفة |
|--------|---------|
| `CONFIG_OF` | يفعّل الـ Open Firmware / Device Tree support |
| `CONFIG_OF_DEBUG` | يضيف debug output للـ OF core |
| `CONFIG_OF_UNITTEST` | يشغّل unit tests للـ OF functions |
| `CONFIG_OF_DYNAMIC` | يسمح بتعديل الـ DT في runtime |
| `CONFIG_DEBUG_FS` | يفعّل الـ debugfs اللي بيعرض `device-tree/` |
| `CONFIG_PHYLIB` | الـ PHY library (مطلوب لـ `phy_interface_t`) |
| `CONFIG_NVMEM` | للـ `of_get_mac_address_nvmem` |
| `CONFIG_NET_CORE` | لـ `net_device` و `of_get_ethdev_address` |
| `CONFIG_DEBUG_INFO` | لازم يكون شغال عشان الـ kernel debugger يشتغل صح |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل `pr_debug()` و `dev_dbg()` ديناميكياً |

---

#### 6. أدوات الـ Subsystem

```bash
# fdtdump — اطبع الـ DTB كاملاً (على الـ host)
dtc -I dtb -O dts /boot/dtb-$(uname -r) -o /tmp/board.dts
grep -A 10 'ethernet' /tmp/board.dts

# فحص الـ NVMEM اللي بيخدم MAC address
ls /sys/bus/nvmem/devices/
cat /sys/bus/nvmem/devices/*/nvmem | xxd | head

# ethtool — تحقق من الـ PHY mode اللي اتعمل negotiate عليه
ethtool eth0 | grep -i 'speed\|duplex\|port\|transceiver'
ethtool -i eth0  # يعرض driver info

# ip link — تحقق من الـ MAC اللي اتسجل
ip link show eth0
```

---

#### 7. جدول رسايل الـ Error الشايعة

| رسالة الـ Kernel Log | المعنى | الحل |
|----------------------|--------|-------|
| `of_get_mac_address: invalid mac address` | الـ property موجودة بس قيمتها غلط (مش 6 bytes أو multicast) | صح الـ DT: `local-mac-address = [aa bb cc dd ee ff];` |
| `of_get_phy_mode: ENODEV` | مفيش property `phy-mode` في الـ DT node | ضيف `phy-mode = "rgmii-id";` للـ DT node |
| `of_get_phy_mode: invalid phy mode` | القيمة مش معروفة في `phy_modes[]` | استخدم قيمة صحيحة: `rgmii`, `sgmii`, `mii`, إلخ |
| `of_get_mac_address: failed to get mac nvmem` | الـ `nvmem-cells` مش موجودة أو الـ NVMEM driver مش loaded | تحقق من `CONFIG_NVMEM` وإن الـ NVMEM phandle صح |
| `of_find_net_device_by_node: NULL` | الـ net_device لم يتسجل بعد أو الـ node مش مربوط | تأكد إن الـ driver بيعمل `SET_NETDEV_DEV(dev, &pdev->dev)` |
| `eth0: failed to get MAC address` | فشل `of_get_ethdev_address` — ممكن يرجع للـ nvmem أو DT | شيل الـ DT أو فحص الـ NVMEM |
| `probe of ethernet@X failed with error -22` | `-EINVAL` من `of_get_phy_mode` أو `of_get_mac_address` | افحص الـ DT properties بـ `dtc` |

---

#### 8. أماكن استراتيجية لـ `dump_stack()` و`WARN_ON()`

```c
int of_get_mac_address(struct device_node *np, u8 *mac)
{
    int ret;

    ret = of_get_mac_addr_property(np, mac);
    if (ret) {
        /* نقطة استراتيجية: لو فشل التحقق من الـ nvmem fallback */
        WARN_ON(ret != -ENODATA && ret != -ENOENT);
        return ret;
    }

    /* تحقق إن الـ MAC مش multicast أو all-zeros */
    if (!is_valid_ether_addr(mac)) {
        dump_stack(); /* اعرف مين طلب MAC غلط */
        return -EINVAL;
    }
    return 0;
}
```

```c
int of_get_phy_mode(struct device_node *np, phy_interface_t *interface)
{
    /* لو رجع -ENODEV ومفيش fallback في الـ driver */
    if (ret == -ENODEV)
        WARN_ON_ONCE(!np); /* الـ node نفسه NULL ده مشكلة */
}
```

**الأماكن الأفضل للـ WARN_ON:**

- بعد `of_get_mac_address` لو `!is_valid_ether_addr(mac)`
- بعد `of_get_phy_mode` لو الـ mode رجع `PHY_INTERFACE_MODE_NA` بشكل غير متوقع
- في `of_find_net_device_by_node` لو الـ result NULL وده مش متوقع في سياق الـ driver

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State بيطابق الـ Kernel State

```bash
# قارن الـ MAC في الـ DT مع اللي بيقوله الـ kernel
# من الـ DT (raw):
xxd /sys/kernel/debug/device-tree/soc/ethernet@1234/local-mac-address

# من الـ kernel (processed):
cat /sys/class/net/eth0/address

# قارن الـ PHY mode في الـ DT مع اللي اتفاوض عليه فعلاً
xxd /sys/kernel/debug/device-tree/soc/ethernet@1234/phy-mode
ethtool eth0 | grep Port

# تحقق من الـ RGMII delay settings (مهمة جداً)
# بعض الـ SoCs بتحتاج phy-mode = "rgmii-id" مش "rgmii"
```

---

#### 2. Register Dump Techniques

```bash
# اقرأ الـ MAC controller registers (بتغير العنوان حسب الـ SoC)
# مثال: SoC Ethernet base = 0x10010000
devmem2 0x10010000 w   # MAC Configuration Register
devmem2 0x10010004 w   # MAC Frame Filter
devmem2 0x10010008 w   # MAC Hash Table High
devmem2 0x1001000c w   # MAC Hash Table Low

# لو بتستخدم /dev/mem
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    m = mmap.mmap(f.fileno(), 0x100, offset=0x10010000)
    val = struct.unpack('<I', m[0:4])[0]
    print(f'MAC_CFG: 0x{val:08x}')
"

# اقرأ الـ PHY registers عبر ethtool
ethtool -d eth0            # dump الـ registers
ethtool --phy-statistics eth0  # PHY statistics
```

---

#### 3. Logic Analyzer / Oscilloscope

**نقاط القياس الأساسية لـ RGMII:**

```
RGMII Interface Signals:
┌────────────┬──────────────┬─────────────────────────────┐
│  Signal    │  Voltage     │  ملاحظة                     │
├────────────┼──────────────┼─────────────────────────────┤
│ TXC        │ 1.8V / 3.3V │ TX clock — يجب يكون stable  │
│ TXD[3:0]  │ 1.8V / 3.3V │ TX data — 4 bits             │
│ TX_CTL     │ 1.8V / 3.3V │ TX enable / error            │
│ RXC        │ 1.8V / 3.3V │ RX clock من الـ PHY          │
│ RXD[3:0]  │ 1.8V / 3.3V │ RX data                      │
│ RX_CTL     │ 1.8V / 3.3V │ RX DV / error                │
└────────────┴──────────────┴─────────────────────────────┘

Timing (1Gbps RGMII):
- TXC period = 8ns
- Setup time  = 1ns minimum
- Hold time   = 1ns minimum
- RGMII-ID: MAC يعمل 2ns delay على TXC و RXC
```

**الـ MDIO bus للـ PHY:**

```
MDIO (Management Data I/O):
- MDC: clock — max 2.5MHz
- MDIO: data — bidirectional

راقب الـ MDIO traffic عشان تتأكد إن of_get_phy_mode
اتطبق صح على الـ PHY registers
```

---

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة الهاردوير | الـ Pattern في الـ Kernel Log |
|-------------------|-------------------------------|
| الـ RGMII delay غلط (phy-mode خاطئ في DT) | `eth0: Link is Down` رغم إن الكابل موصل، أو `NETDEV WATCHDOG: eth0 transmit timeout` |
| الـ MAC address في الـ EEPROM/NVMEM بايظ | `eth0: using random MAC address` |
| MDIO bus مش واصل للـ PHY | `MDIO read timeout` أو `PHY ID = 0xffffffff` |
| voltage mismatch (1.8V vs 3.3V) | لا PHY، لا MAC — silence في الـ boot log |
| الـ PHY مش محتاج reset | `phy: ... autonegotiation timed out` |
| wrong `phy-handle` phandle في DT | `of_mdio_parse_addr: missing reg property` |

---

#### 5. Device Tree Debugging

```bash
# على الـ target board، اطبع الـ DT المحمل فعلاً
dtc -I fs -O dts /sys/firmware/devicetree/base/ 2>/dev/null | \
    grep -A 20 'ethernet'

# تحقق من الـ phy-mode property
cat /sys/firmware/devicetree/base/soc/ethernet@1234/phy-mode
# المتوقع: rgmii-id (أو أي mode تاني)

# تحقق من الـ MAC address property
hexdump -C /sys/firmware/devicetree/base/soc/ethernet@1234/local-mac-address
# المتوقع: 6 bytes

# تحقق من الـ nvmem-cells للـ MAC
grep -r 'mac-address' /sys/firmware/devicetree/base/ 2>/dev/null

# قارن الـ DTB اللي على الـ disk بالـ live DT
dtc -I dtb -O dts /boot/board.dtb -o /tmp/boot.dts
dtc -I fs  -O dts /sys/firmware/devicetree/base/ -o /tmp/live.dts
diff /tmp/boot.dts /tmp/live.dts
```

**DT صحيح لـ Ethernet node:**

```dts
ethernet0: ethernet@10010000 {
    compatible = "vendor,eth";
    reg = <0x10010000 0x1000>;
    /* MAC address ثابتة في الـ DT */
    local-mac-address = [aa bb cc dd ee ff];
    /* أو من NVMEM */
    nvmem-cells = <&mac_address>;
    nvmem-cell-names = "mac-address";
    /* PHY mode — أهم property */
    phy-mode = "rgmii-id";
    phy-handle = <&phy0>;
};
```

---

### Practical Commands

#### سيناريو 1: تشخيص فشل `of_get_mac_address`

```bash
# 1. تحقق إن الـ property موجودة في الـ DT
hexdump -C /sys/firmware/devicetree/base/soc/ethernet@1234/local-mac-address

# 2. تحقق من الـ NVMEM fallback
ls /sys/bus/nvmem/devices/
for dev in /sys/bus/nvmem/devices/*/nvmem; do
    echo "=== $dev ==="; hexdump -C "$dev" | head -5
done

# 3. فعّل dynamic debug وشوف الـ log
echo 'file of_net.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 'file nvmem-core.c +pflmt' >> /sys/kernel/debug/dynamic_debug/control
dmesg -C && dmesg -w &
# شغّل الـ driver من تاني
echo 0x10010000.ethernet > /sys/bus/platform/drivers/vendor-eth/unbind
echo 0x10010000.ethernet > /sys/bus/platform/drivers/vendor-eth/bind
```

#### سيناريو 2: تشخيص فشل `of_get_phy_mode`

```bash
# 1. تحقق من الـ property
cat /sys/firmware/devicetree/base/soc/ethernet@1234/phy-mode
# المتوقع: rgmii-id, sgmii, mii, gmii, etc.

# 2. شوف القيم الصحيحة المدعومة في الـ kernel
grep 'PHY_INTERFACE_MODE_' \
    /workspace/external/linux/include/linux/phy.h | head -20

# 3. تابع الـ call باستخدام ftrace
echo 'of_get_phy_mode' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
# trigger الـ probe
cat /sys/kernel/debug/tracing/trace
```

#### سيناريو 3: التحقق من الـ PHY negotiation

```bash
# تحقق شامل من حالة الـ PHY
ethtool eth0
# المتوقع:
# Settings for eth0:
#   Speed: 1000Mb/s
#   Duplex: Full
#   Port: MII
#   Link detected: yes

# شوف الـ PHY registers مباشرة
ethtool -d eth0 2>/dev/null || mii-tool -v eth0

# تحقق من الـ phy interface اللي اختاره الـ kernel
grep 'phy_interface\|phy.mode' /sys/kernel/debug/tracing/trace
```

#### سيناريو 4: التحقق من تطابق الـ DT مع الـ Hardware

```bash
# اطبع كل الـ Ethernet-related nodes
dtc -I fs -O dts /sys/firmware/devicetree/base/ 2>/dev/null | \
    awk '/ethernet|mdio|phy/{found=1} found{print; if(/\}/) {found=0}}'

# تحقق من الـ phy-handle phandle يرجع لـ PHY node صح
dtc -I fs -O dts /sys/firmware/devicetree/base/ 2>/dev/null | \
    grep -E 'phy-handle|phandle|reg\s*='

# تحقق من الـ clock للـ MAC
cat /sys/firmware/devicetree/base/soc/ethernet@1234/clock-names
cat /sys/kernel/debug/clk/eth_clk/clk_rate  # لو الـ clk debugfs شغال
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Ethernet مش شغال على Industrial Gateway بـ RK3562

#### العنوان
**الـ PHY mode غلط في Device Tree بيخلي الـ Ethernet يموت من أول ما يبوت**

#### السياق
شركة بتبني industrial gateway بـ RK3562 — الجهاز بيتوصل بـ Modbus/TCP لـ PLC devices في factory floor. الـ board بيستخدم **RTL8211F PHY** متوصل على الـ MAC بـ RGMII interface. المنتج وصل للـ production وبعد الشحن الـ Ethernet مشغلتش على أي وحدة.

#### المشكلة
الـ kernel log بيقول:

```bash
rk_gmac-dwmac fe010000.ethernet eth0: no phy found
# أو
rk_gmac-dwmac fe010000.ethernet: PHY reset timed out
```

الـ `ip link show eth0` بيرجع `NO-CARRIER` حتى لو الكابل متوصل.

#### التحليل
الـ driver بيستدعي `of_get_phy_mode()` من `of_net.h` جوا `probe`:

```c
/* inside rk_gmac driver probe */
ret = of_get_phy_mode(dev->of_node, &plat->interface);
if (ret) {
    dev_err(dev, "missing phy-mode property\n");
    return ret;
}
```

`of_get_phy_mode()` بتفتش في الـ `device_node` عن property اسمها `"phy-mode"` أو `"phy-connection-type"`. لو مش موجودة بترجع `-ENOENT`. لو موجودة بس بـ typo أو قيمة مش recognized، بترجع `-ENODEV`.

في الـ DTS بتاع الـ production board:

```dts
/* WRONG — مكتوب rgmii بس المطلوب rgmii-id */
&gmac1 {
    phy-mode = "rgmii";
    /* RTL8211F needs internal delay handled by MAC side */
};
```

**الـ RTL8211F** ده PHY بيحتاج الـ TX delay يتعمل من جهة الـ MAC (RK3562 GMAC). الصح هو `"rgmii-id"` اللي بيقول لـ `of_get_phy_mode()` ترجع `PHY_INTERFACE_MODE_RGMII_ID`. لما بترجع `PHY_INTERFACE_MODE_RGMII` بدل كده، الـ MAC بيشتغل بدون internal delay فالـ clock skew بيعدي الـ setup/hold time وربط الـ PHY بيفشل.

#### الحل

```dts
/* rk3562-gateway.dts */
&gmac1 {
    assigned-clocks = <&cru SCLK_GMAC1>;
    assigned-clock-parents = <&gmac1_clkin>;
    clock_in_out = "input";

    /* FIX: use rgmii-id so MAC adds TX+RX delay */
    phy-mode = "rgmii-id";

    pinctrl-names = "default";
    pinctrl-0 = <&gmac1m1_miim
                 &gmac1m1_tx_bus2
                 &gmac1m1_rx_bus2
                 &gmac1m1_rgmii_clk
                 &gmac1m1_rgmii_bus>;

    phy-handle = <&rgmii_phy1>;
    status = "okay";
};
```

```bash
# للتحقق بعد التعديل
dmesg | grep -i "phy mode\|rgmii\|gmac"
ethtool eth0 | grep "Speed\|Duplex\|Link"
```

#### الدرس المستفاد
**الـ `of_get_phy_mode()`** بترجع enum من `phy_interface_t` — الفرق بين `RGMII` و`RGMII_ID` و`RGMII_RXID` و`RGMII_TXID` مش cosmetic، ده بيتحكم في الـ clock delay configuration. لازم تقرأ datasheet الـ PHY وتحدد مين المسؤول عن الـ delay: الـ MAC ولا الـ PHY ولا هما مع بعض.

---

### السيناريو الثاني: MAC Address بتتغير مع كل Boot على Android TV Box بـ Allwinner H616

#### العنوان
**الـ `of_get_mac_address()` مش لاقي الـ MAC في الـ DTS فبيولد random address**

#### السياق
مصنع بيعمل Android TV box بـ Allwinner H616. الـ box بتتباع بـ IPTV service تعتمد على MAC address ثابت للـ licensing. المستخدمين بيشتكوا إن الـ subscription بتنقطع بعد كل restart.

#### المشكلة
```bash
# على البورد
ip link show eth0
# 2: eth0: <BROADCAST,MULTICAST> mtu 1500
#     link/ether da:3f:8b:12:45:aa brd ff:ff:ff:ff:ff:ff
# كل boot بيدي MAC مختلف!
```

```bash
dmesg | grep -i "mac\|address"
# sunxi-gmac 5020000.ethernet: Using random MAC address
```

#### التحليل
الـ driver بيعمل:

```c
/* sunxi GMAC driver */
ret = of_get_mac_address(pdev->dev.of_node, addr);
if (ret == -ENODEV || ret == -EINVAL) {
    /* fallback: generate random */
    eth_hw_addr_random(ndev);
    dev_info(&pdev->dev, "Using random MAC address\n");
}
```

**`of_get_mac_address()`** بتدور على properties بالترتيب ده:
1. `"mac-address"` — عادةً بيتحط من bootloader
2. `"local-mac-address"` — static في DTS
3. بتروح لـ `of_get_mac_address_nvmem()` لو في `nvmem-cells` property

في الـ DTS الحالي:

```dts
/* h616-tvbox.dts — WRONG, missing mac address source */
&emac0 {
    phy-handle = <&ext_rgmii_phy>;
    phy-mode = "rgmii-id";
    /* لا mac-address ولا local-mac-address ولا nvmem-cells */
    status = "okay";
};
```

الـ U-Boot بيكتب `mac-address` في الـ DTS في الـ RAM لكن لو الـ boot arguments مش بتمرره صح، أو لو الـ SID (Security ID) fuses مش مقروية، النتيجة هي random MAC.

#### الحل

**Option 1: قراءة MAC من SID fuses عبر NVMEM:**

```dts
/* h616-tvbox.dts */
&sid {
    /* Allwinner SID contains factory-burned MAC */
    mac_addr_sid: mac-addr@0c {
        reg = <0x0c 0x06>; /* 6 bytes at offset 0x0c */
    };
};

&emac0 {
    phy-handle = <&ext_rgmii_phy>;
    phy-mode = "rgmii-id";
    nvmem-cells = <&mac_addr_sid>;
    nvmem-cell-names = "mac-address";
    status = "okay";
};
```

`of_get_mac_address()` لما تلاقي `nvmem-cells` بتفوض لـ `of_get_mac_address_nvmem()` اللي بتقرأ الـ SID عبر NVMEM framework.

**Option 2: U-Boot يكتب MAC في DTS:**

```bash
# في U-Boot env
setenv ethaddr "AA:BB:CC:DD:EE:FF"
saveenv
# U-Boot هيكتب mac-address property في chosen/ethernet node
```

```bash
# للتحقق
cat /sys/class/net/eth0/address
# لازم يكون ثابت عبر boots
```

#### الدرس المستفاد
**`of_get_mac_address()`** بتمشي سلسلة من fallbacks. لو الـ SoC عنده fuses زي Allwinner SID أو i.MX OTP، استخدم `nvmem-cells` في الـ DTS وده بيضمن MAC ثابت مرتبط بالـ hardware، مش بالـ bootloader configuration.

---

### السيناريو الثالث: Industrial IoT Sensor بـ STM32MP1 — الـ Ethernet Driver مش بيعمل Probe

#### العنوان
**`of_get_ethdev_address()` بيرجع error لأن `net_device` لسه محلاش**

#### السياق
Team بتطور IoT sensor hub بـ STM32MP1 (Cortex-A7 + M4). الجهاز بيجمع بيانات من 32 sensor عبر SPI وبيرسلها بـ Ethernet. Custom driver بيستخدم `of_get_ethdev_address()` بطريقة غلط أثناء `probe()`.

#### المشكلة
```bash
dmesg | grep stm32-eth
# stm32-eth 5800a000.ethernet: of_get_ethdev_address failed: -19
# stm32-eth 5800a000.ethernet: probe failed: -19
# (-19 = -ENODEV)
```

الـ driver مش بيعمل probe خالص وكل الـ sensors معزولة عن الـ network.

#### التحليل
`of_get_ethdev_address()` signature هو:

```c
int of_get_ethdev_address(struct device_node *np, struct net_device *dev);
```

بتقرأ الـ MAC من الـ DT وبتكتبها مباشرةً في `dev->dev_addr` عبر `eth_hw_addr_set()`. لكن ده بيفرض إن `net_device` يكون **allocated** قبل ما تتستدعي.

الـ driver الغلط:

```c
static int sensor_eth_probe(struct platform_device *pdev)
{
    struct sensor_eth_priv *priv;
    u8 mac[ETH_ALEN];

    /* BUG: ndev مش allocated لسه */
    ret = of_get_ethdev_address(pdev->dev.of_node, NULL); /* crash! */

    /* أو بيعمل alloc بعدين */
    ndev = alloc_etherdev(sizeof(*priv));
    /* ... */
}
```

الحل الصح هو إنه يعمل `alloc_etherdev()` الأول، أو يستخدم `of_get_mac_address()` اللي بترجع raw bytes من غير ما تحتاج `net_device`:

```c
/* of_get_mac_address — بترجع bytes فقط */
int of_get_mac_address(struct device_node *np, u8 *mac);
/* مش محتاج net_device */
```

#### الحل

```c
static int sensor_eth_probe(struct platform_device *pdev)
{
    struct net_device *ndev;
    struct sensor_eth_priv *priv;
    u8 mac[ETH_ALEN];
    int ret;

    /* STEP 1: allocate net_device first */
    ndev = alloc_etherdev(sizeof(*priv));
    if (!ndev)
        return -ENOMEM;

    SET_NETDEV_DEV(ndev, &pdev->dev);

    /* STEP 2: now of_get_ethdev_address is safe */
    ret = of_get_ethdev_address(pdev->dev.of_node, ndev);
    if (ret) {
        dev_warn(&pdev->dev,
                 "no MAC in DT (%d), using random\n", ret);
        eth_hw_addr_random(ndev);
    }

    /* rest of init... */
}
```

```dts
/* stm32mp1-sensor-hub.dts */
&ethernet0 {
    status = "okay";
    phy-mode = "rmii";
    /* RMII on STM32MP1 is common for cost-sensitive designs */
    local-mac-address = [AA BB CC DD EE FF];
    phy-handle = <&phy0>;
};
```

#### الدرس المستفاد
**`of_get_ethdev_address()`** هي wrapper فوق `of_get_mac_address()` بتكتب في `net_device` مباشرةً — لازم الـ `net_device` يكون allocated قبلها. لو محتاج الـ MAC قبل الـ alloc، استخدم `of_get_mac_address()` العادية اللي بترجع raw bytes في `u8 mac[6]`.

---

### السيناريو الرابع: Automotive ECU بـ i.MX8 — الـ `of_find_net_device_by_node()` بيرجع NULL في Wrong Context

#### العنوان
**محاولة إيجاد الـ `net_device` بـ `of_find_net_device_by_node()` من داخل early init بتفشل**

#### السياق
شركة automotive بتطور ECU بـ i.MX8QM للـ ADAS system. فيه custom kernel module مسؤول عن تحديد priority لـ network traffic بناءً على vehicle speed. الـ module بيحاول يجيب reference لـ `net_device` الخاصة بـ AVB (Audio Video Bridging) Ethernet interface عشان يطبق QoS rules.

#### المشكلة
```bash
# في dmesg عند load الـ module
imx8-avb-qos: of_find_net_device_by_node returned NULL for fec1
imx8-avb-qos: cannot apply QoS rules, aborting
```

الـ module بيشتغل بـ `module_init()` لكن الـ network device لسه مش registered في الوقت ده.

#### التحليل
`of_find_net_device_by_node()` بتعمل:

```c
struct net_device *of_find_net_device_by_node(struct device_node *np)
{
    struct device *dev;

    dev = class_find_device_by_of_node(&net_class, np);
    if (!dev)
        return NULL;

    return to_net_dev(dev);
}
```

بتدور في `net_class` على device مرتبطة بالـ `device_node`. لو الـ Ethernet driver لسه ما عملش `register_netdev()` —  اللي بيحصل أثناء deferred probe أو lazy init — الـ function هترجع `NULL`.

في الـ i.MX8QM، الـ FEC driver (Ethernet) بيعتمد على:
- Clock initialization من CCF
- Power domain من SoC power management
- IOMUXC pinmux configuration

كل ده ممكن يسبب deferred probe، وبالتالي الـ `net_device` ممكن تتـregister بعد الـ AVB QoS module.

#### الحل

```c
/* imx8-avb-qos.c */
#include <linux/of_net.h>
#include <linux/notifier.h>
#include <linux/netdevice.h>

static struct device_node *avb_np;

/* Use netdevice notifier instead of early lookup */
static int avb_netdev_event(struct notifier_block *nb,
                             unsigned long event, void *ptr)
{
    struct net_device *ndev = netdev_notifier_info_to_dev(ptr);
    struct net_device *target;

    if (event != NETDEV_REGISTER)
        return NOTIFY_DONE;

    /* now it's safe — device just registered */
    target = of_find_net_device_by_node(avb_np);
    if (target && target == ndev) {
        apply_avb_qos_rules(ndev);
        pr_info("AVB QoS applied to %s\n", ndev->name);
    }

    return NOTIFY_DONE;
}

static struct notifier_block avb_netdev_nb = {
    .notifier_call = avb_netdev_event,
};

static int __init avb_qos_init(void)
{
    avb_np = of_find_node_by_alias(NULL, "ethernet1");
    if (!avb_np)
        return -ENODEV;

    /* Register notifier — يشتغل لما net_device تتregister */
    return register_netdevice_notifier(&avb_netdev_nb);
}
```

#### الدرس المستفاد
**`of_find_net_device_by_node()`** بتبحث في `net_class` اللي بيُكتب فيها بس لما `register_netdev()` تتستدعي. في أي سيناريو فيه deferred probe أو async init، اللازم تستخدم **netdevice notifier** وتستدعي الـ function جواه عشان تضمن إن الـ device موجودة فعلاً.

---

### السيناريو الخامس: Custom Board Bring-up بـ AM62x — الـ Ethernet شغال في U-Boot بس مش في Linux

#### العنوان
**الـ `CONFIG_OF` و`CONFIG_NET` مش enabled فالـ `of_net.h` بيرجع stubs بـ `-ENODEV`**

#### السياق
Engineer بيعمل bring-up لـ custom board بـ TI AM62x (Sitara) لمشروع industrial HMI. الـ Ethernet شغالة تمام في U-Boot، بيقدر يعمل TFTP. لما بوّت الـ Linux kernel، `eth0` مش بيظهر خالص في `ip link`.

#### المشكلة
```bash
dmesg | grep -i eth
# (nothing — no ethernet driver even tried to probe)

ls /sys/class/net/
# lo
# (eth0 غايب تماماً)
```

```bash
zcat /proc/config.gz | grep -E "CONFIG_OF|CONFIG_NET|CONFIG_TI_CPSW"
# CONFIG_OF is not set
# CONFIG_NET=y
# CONFIG_TI_CPSW=y
```

#### التحليل
الـ `of_net.h` كله محاط بـ:

```c
#if defined(CONFIG_OF) && defined(CONFIG_NET)
/* real implementations */
extern int of_get_phy_mode(...);
extern int of_get_mac_address(...);
/* ... */
#else
/* stub implementations — all return -ENODEV or NULL */
static inline int of_get_phy_mode(...) { return -ENODEV; }
static inline int of_get_mac_address(...) { return -ENODEV; }
/* ... */
#endif
```

لما `CONFIG_OF` مش enabled، **كل الـ functions** بتبقى stubs ترجع `-ENODEV`. الـ TI CPSW driver بيعمل:

```c
ret = of_get_phy_mode(node, &interface);
if (ret < 0) {
    dev_err(dev, "failed to get phy-mode: %d\n", ret);
    return ret; /* probe fails completely */
}
```

الـ probe بيفشل قبل ما يحاول يعمل register. وده بيحصل لأن الـ custom kernel config نسيت تـenable `CONFIG_OF` اللي هو ضروري لأي ARM SoC بيستخدم Device Tree.

#### الحل

```bash
# في kernel menuconfig
make ARCH=arm64 menuconfig

# Device Drivers → Device Tree and Open Firmware support
#   [*] Support for device tree in /proc

# OR في .config مباشرةً:
CONFIG_OF=y
CONFIG_OF_ADDRESS=y
CONFIG_OF_IRQ=y
CONFIG_OF_NET=y   # بيـenable of_net.c implementations
```

```bash
# rebuild
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc) Image dtbs modules

# تحقق
zcat /proc/config.gz | grep CONFIG_OF
# CONFIG_OF=y
# CONFIG_OF_NET=y

dmesg | grep cpsw
# ti-cpsw-nuss 8000000.ethernet: initialized
# ti-cpsw-nuss 8000000.ethernet eth0: Link is Up - 1Gbps/Full
```

```bash
# لو عاوز تتأكد إن الـ DT parsed صح
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A5 "ethernet"
# لازم تشوف phy-mode و mac-address
```

#### الدرس المستفاد
الـ `#if defined(CONFIG_OF) && defined(CONFIG_NET)` في `of_net.h` مش مجرد guard — ده بيغير سلوك الـ functions جذرياً. على أي ARM SoC بيستخدم Device Tree، `CONFIG_OF` لازم يكون `y` وإلا كل الـ DT-based network configuration هتبقى dead stubs. في custom board bring-up، دايماً verify الـ kernel config من أول حاجة قبل ما تحقق في الـ DTS أو الـ driver.
## Phase 7: مصادر ومراجع

### مصادر LWN.net

دي أهم المقالات اللي بتغطي الـ device tree والـ networking في الـ kernel:

| المقال | الأهمية |
|--------|---------|
| [ELCE: Grant Likely on device trees](https://lwn.net/Articles/414016/) | مقدمة أساسية للـ device tree من مؤلفه |
| [Platform devices and device trees](https://lwn.net/Articles/448502/) | بيشرح إزاي الـ platform devices بتتعامل مع الـ DT |
| [KS2009: Generic device trees](https://lwn.net/Articles/357487/) | الجلسة الأصلية اللي Grant Likely شرح فيها الـ DT |
| [Device tree bindings](https://lwn.net/Articles/572114/) | بيشرح الـ bindings وإزاي بيتكتبوا |
| [Device trees II: The harder parts](https://lwn.net/Articles/573409/) | الجزء التقني الأعمق في تنفيذ الـ DT |
| [Add common OF device tree support for MDIO busses](https://lwn.net/Articles/326450/) | مباشراً عن دعم الشبكات في الـ DT |
| [Picking a MAC address for a FreedomBox](https://lwn.net/Articles/528112/) | نقاش عملي عن الـ MAC address في الـ embedded Linux |
| [Dynamic DT device nodes](https://lwn.net/Articles/872164/) | الـ DT nodes الديناميكية |
| [Linux and the Devicetree — usage model](https://static.lwn.net/kerneldoc/devicetree/usage-model.html) | الـ usage model الرسمي من الـ kernel docs |

---

### التوثيق الرسمي في الـ kernel

**الـ `Documentation/` paths** الأهم للـ `of_net.h`:

```
Documentation/devicetree/bindings/net/ethernet-controller.yaml
Documentation/devicetree/bindings/net/ethernet-phy.yaml
Documentation/devicetree/usage-model.rst
Documentation/driver-api/nvmem.rst
Documentation/networking/net_dev_features.rst
```

**الملفات المصدرية المباشرة:**

```
include/linux/of_net.h        ← الـ header موضوع البحث
net/ethernet/eth.c            ← of_get_ethdev_address implementation
drivers/of/of_net.c           ← تنفيذ كل الـ functions
include/linux/phy.h           ← تعريف phy_interface_t
include/linux/of.h            ← الـ OF core API
```

---

### كوميتات kernel مهمة

| الكوميت / الـ Patch | الوصف |
|---------------------|--------|
| [nvmem support for of_get_mac_address](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1556893635-18549-3-git-send-email-ynezz@true.cz/) | إضافة دعم الـ nvmem لجلب الـ MAC address |
| [net: add nvmem MAC address support](https://patchwork.kernel.org/project/linux-omap/patch/20180718161035.7005-5-brgl@bgdev.pl/) | دعم الـ nvmem في `eth_platform_get_mac_address()` |
| [net/smscx5xx: use device tree for MAC](https://lore.kernel.org/patchwork/patch/643640) | مثال عملي على استخدام الـ DT للـ MAC في driver |
| [davinci_emac: use nvmem for MAC](https://patchwork.kernel.org/project/linux-omap/patch/20180626102245.30711-8-brgl@bgdev.pl/) | تطبيق الـ nvmem MAC lookup في driver حقيقي |

---

### نقاشات الـ mailing list

- **[mac-address vs local-mac-address](https://linuxppc-dev.ozlabs.narkive.com/9gJb90pU/mac-address-vs-local-mac-address)** — نقاش تاريخي مهم عن الفرق بين الـ property الاتنين في الـ DT
- **[Passing MAC address via Device Tree Blob](https://community.element14.com/products/devtools/avnetboardscommunity/avnetboard-forums/f/general/36448/passing-mac-address-to-kernel-via-device-tree-blob)** — تطبيق عملي على board حقيقية
- **[U-Boot: Device tree and MAC address questions](http://u-boot.10912.n7.nabble.com/Device-tree-and-u-boot-passing-MAC-address-questions-td28275.html)** — إزاي الـ bootloader بيمرر الـ MAC للـ kernel عبر الـ DT

---

### مصادر elinux.org

| الصفحة | المحتوى |
|--------|---------|
| [Device Tree Reference](https://elinux.org/Device_Tree_Reference) | مرجع شامل لـ syntax والـ bindings |
| [Device Tree Mysteries](https://elinux.org/Device_Tree_Mysteries) | حل مشاكل الـ DT الشائعة |
| [Device Trees](https://elinux.org/Device_Trees) | صفحة البداية للـ DT على elinux |
| [Device tree plumbers 2016](https://elinux.org/Device_tree_plumbers_2016_etherpad) | نقاشات مؤتمر 2016 عن مستقبل الـ DT |
| [Device tree kernel summit 2017](https://elinux.org/Device_tree_kernel_summit_2017_etherpad) | قرارات مهمة عن الـ DT architecture |

---

### kernelnewbies.org

| الصفحة | ما تلاقيه فيها |
|--------|----------------|
| [Linux_4.3](https://kernelnewbies.org/Linux_4.3) | تغييرات الـ networking والـ DT في kernel 4.3 |
| [Linux_4.9](https://kernelnewbies.org/Linux_4.9) | تحسينات الـ network subsystem |
| [Linux_3.2](https://kernelnewbies.org/Linux_3.2) | أولى إضافات الـ OF network support |
| [KernelGlossary](https://kernelnewbies.org/KernelGlossary) | تعريفات مهمة: OF, DT, NVMEM, PHY |
| [Documents](https://kernelnewbies.org/Documents) | فهرس الوثائق التعليمية |

---

### الكتب الموصى بيها

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصول المهمة:** Chapter 14 (The Linux Device Model), Chapter 17 (Network Drivers)
- **الـ Chapter 17** بيشرح `net_device`, `register_netdev()`, وإزاي الـ driver بيضبط الـ MAC address
- متاح مجاناً: [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 5:** System Calls
- **الفصل 17:** Devices and Modules — بيغطي الـ device model والـ OF
- الأساس لفهم إزاي الـ kernel بيتعامل مع الـ hardware abstraction

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 7:** Bootloaders — بيشرح إزاي الـ U-Boot بيبني الـ DT ويمرره للـ kernel
- **الفصل 15:** Debugging Embedded Linux — بيشمل debugging الـ DT
- مهم جداً لفهم الـ `mac-address` property وإزاي الـ bootloader بيحدد قيمتها runtime

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- **الفصل 12:** Networks — تغطية شاملة للـ network stack من الـ kernel perspective

---

### مصادر إضافية مهمة

**الـ NVMEM Subsystem:**
- [NVMEM Subsystem — kernel docs](https://docs.kernel.org/driver-api/nvmem.html) — لفهم `of_get_mac_address_nvmem()`

**الـ PHY Layer:**
- [linux/phy.h documentation](https://docs.kernel.org/networking/phy.html) — لفهم `phy_interface_t` المستخدمة في `of_get_phy_mode()`

**الـ OF Core:**
- [Open Firmware and Device Tree — kernel docs](https://docs.kernel.org/devicetree/index.html) — الدليل الرسمي الكامل

**مثال عملي على board:**
- [Set unique eth0 MAC from EEPROM](https://www.acmesystems.it/acqua_at24mac402) — تطبيق real-world لـ `of_get_mac_address` مع AT24MAC402

---

### كلمات البحث

لو عايز تلاقي معلومات أكتر، ابحث بـ:

```
of_get_mac_address kernel implementation
of_get_phy_mode device tree PHY
of_find_net_device_by_node kernel
linux kernel nvmem mac address lookup
device tree local-mac-address property
of_net.c kernel source
linux ethernet device tree binding
phy_interface_t device tree mode
of_get_ethdev_address net_device
linux kernel MAC address NVMEM cell
```
## Phase 8: Writing simple module

### الفكرة

**`of_get_mac_address`** هي الدالة اللي هنعمل عليها kprobe — بتجيب الـ MAC address من الـ Device Tree لأي network device جديد بيتسجل. ده بيحصل وقت boot أو وقت ما driver جديد يتحمّل، يعني الـ hook هيطبع كل MAC بيتقرأ من الـ DT في اللحظة دي.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kprobes.h>      /* kprobe API */
#include <linux/of.h>           /* struct device_node, of_node_full_name() */
#include <linux/netdevice.h>    /* ETH_ALEN */
#include <linux/printk.h>       /* pr_info */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("KernelDoc");
MODULE_DESCRIPTION("kprobe on of_get_mac_address — log MAC reads from Device Tree");

/*
 * of_get_mac_address signature:
 *   int of_get_mac_address(struct device_node *np, u8 *mac);
 *
 * pt_regs on x86-64:
 *   regs->di = first arg  (np)
 *   regs->si = second arg (mac buffer — not filled yet at pre-handler)
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* grab the device_node pointer from the first argument register */
    struct device_node *np = (struct device_node *)regs->di;

    if (!np)
        return 0;

    /* of_node_full_name() returns the full DT path, e.g. /soc/ethernet@1c30000 */
    pr_info("of_net_probe: of_get_mac_address called for node: %s\n",
            of_node_full_name(np));

    return 0; /* 0 = let the real function continue normally */
}

/* post-handler: بعد ما الدالة ترجع، نطبع الـ MAC اللي اتحفظ في الـ buffer */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * At post-handler the function already returned.
     * regs->ax holds the return value (0 = success).
     * regs->si still points to the mac buffer that was filled.
     */
    u8 *mac = (u8 *)regs->si;
    long ret  = (long)regs->ax;

    if (ret == 0 && mac)
        pr_info("of_net_probe: MAC read OK => %pM\n", mac);
    else
        pr_info("of_net_probe: of_get_mac_address returned %ld (no MAC or error)\n",
                ret);
}

/* تعريف الـ kprobe نفسه */
static struct kprobe kp = {
    .symbol_name  = "of_get_mac_address", /* اسم الـ symbol في الـ kernel */
    .pre_handler  = handler_pre,          /* بيتشغل قبل الدالة */
    .post_handler = handler_post,         /* بيتشغل بعد الدالة */
};

static int __init of_net_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("of_net_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("of_net_probe: kprobe planted on %s at %px\n",
            kp.symbol_name, kp.addr);
    return 0;
}

static void __exit of_net_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("of_net_probe: kprobe removed\n");
}

module_init(of_net_probe_init);
module_exit(of_net_probe_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/module.h` | الـ macros الأساسية زي `MODULE_LICENSE` و `module_init` |
| `linux/kprobes.h` | تعريف `struct kprobe` وكل الـ API الخاصة بيه |
| `linux/of.h` | تعريف `struct device_node` ودالة `of_node_full_name()` |
| `linux/netdevice.h` | `ETH_ALEN` = 6 لو احتجنا نتحقق من طول الـ MAC |
| `linux/printk.h` | `pr_info` / `pr_err` للـ logging |

**الـ `of.h`** ضروري هنا عشان لو حاولنا نعمل dereference لـ `device_node` من غيره، الـ compiler هيشتكي من incomplete type.

---

#### الـ `handler_pre`

بيتشغل **قبل** تنفيذ `of_get_mac_address` مباشرة. بنجيب الـ `device_node *` من `regs->di` (الـ first argument على x86-64 ABI) وبنطبع مسار الـ node في الـ Device Tree. في اللحظة دي الـ `mac` buffer لسه فاضي، ده ليه بنستنى الـ post-handler.

---

#### الـ `handler_post`

بيتشغل **بعد** ما الدالة ترجع. الـ `regs->ax` فيه الـ return value (0 = success)، و`regs->si` لسه بيشاور على نفس الـ buffer اللي اتملى بالـ MAC. بنستخدم `%pM` اللي هو format specifier خاص بالـ kernel لطباعة الـ MAC بصيغة `aa:bb:cc:dd:ee:ff`.

---

#### الـ `struct kprobe`

- **`symbol_name`**: الاسم اللي الـ kernel هيدوّر عليه في الـ symbol table — مش محتاجين عنوان صريح.
- **`pre_handler` / `post_handler`**: الـ callbacks اللي عرّفناهم فوق.

---

#### الـ `module_init` و `module_exit`

- **`register_kprobe`** بتحط breakpoint افتراضي على الدالة المطلوبة وبتربط الـ handlers بيه.
- **`unregister_kprobe`** في الـ `exit` ضرورية عشان لو مسحناش الـ hook وفضل الـ callback بيشاور على كود اتنزل من الميموري، هيحصل kernel panic.

---

### Makefile للتجميع

```makefile
obj-m += of_net_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### تجربة الـ Module

```bash
# تحميل الـ module
sudo insmod of_net_probe.ko

# شوف الـ log (لو عندك network driver بيتحمّل أو عمل reload)
sudo dmesg | grep of_net_probe

# إزالة الـ module
sudo rmmod of_net_probe
```

مثال على الـ output المتوقع على جهاز ARM مثلاً:

```
of_net_probe: kprobe planted on of_get_mac_address at ffffffff81a3c210
of_net_probe: of_get_mac_address called for node: /soc/ethernet@1c30000
of_net_probe: MAC read OK => de:ad:be:ef:00:01
```

> **ملاحظة**: على ARM64 الـ first argument في `regs->regs[0]` مش `regs->di` — محتاج تعدّل الـ handler حسب الـ architecture.
