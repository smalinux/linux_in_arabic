## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الملف ده جزء من **ETHERNET PHY LIBRARY** — المسؤول عن إدارة الـ PHY chips (شرائح الـ Ethernet الفيزيائية) وبروتوكول الـ **MDIO** (Management Data Input/Output) اللي بيتكلم معاها.

---

### القصة من الأول

#### مشكلة: الـ PHY chip محتاج حد يتكلم معاه

لما بتصمم جهاز embedded (راوتر، كاميرا شبكية، switch صغير)، بيكون فيه chip مسؤول عن الجزء الفيزيائي من الـ Ethernet — بيتسمى **PHY chip**. الـ CPU محتاج يتكلم مع الـ PHY ده عشان يعرف:
- هل الكابل متوصل؟
- الـ speed كام؟ 10/100/1000 Mbps؟
- فيه أخطاء؟

الكلام ده بيحصل عبر بروتوكول اسمه **MDIO** — بروتوكول serial بسيط محتاج سلكين بس:
- **MDC** (Management Data Clock): ساعة التزامن
- **MDIO** (Management Data I/O): البيانات جاية ورايحة

---

#### الحل الأول: dedicated hardware

في معظم الحالات الـ SoC بيكون عنده وحدة hardware مخصصة للـ MDIO — بتولد الـ clock وبتبعت البيانات أوتوماتيك بدون تدخل الـ CPU.

#### الحل التاني: مفيش hardware — هنعمل "bitbang"

بس أحيانًا الـ SoC مبيكونش عنده MDIO hardware controller. الحل؟ **نستخدم pins عادية (GPIO) ونحرّك الـ clock والبيانات يدويًا في الـ software** — ده اللي بيتسمى **bitbanging**.

تخيل إنك بتكتب كل بت بإيدك: ارفع الـ clock — حط البيت — نزّل الـ clock — كرر 32 مرة. الـ CPU بيعمل ده بالكود بدل الـ hardware.

---

### دور الملف `mdio-gpio.h` (platform_data)

```c
struct mdio_gpio_platform_data {
    u32 phy_mask;           /* which PHY addresses to ignore (bitmask) */
    u32 phy_ignore_ta_mask; /* ignore turnaround bit for certain PHYs */
};
```

الملف ده صغير جدًا — بيعرّف struct واحدة بس. دوره هو نقل **إعدادات الـ platform** (اللي بيحددها الـ board developer) إلى الـ driver.

#### الـ `phy_mask` — مين هو؟

الـ MDIO bus ممكن يكون فيها لحد 32 PHY (عناوين 0-31). أحيانًا الـ board بيكون فيها عناوين فاضية أو محجوزة. الـ `phy_mask` بيقول للـ driver: "متحاولش تتكلم مع الـ PHYs دول" — كل bit set = عنوان يتجاهله.

**مثال:** لو الـ mask = `0x0000001E` يعني عناوين 1,2,3,4 متجاهَلة.

#### الـ `phy_ignore_ta_mask` — مين هو؟

في بروتوكول MDIO، بعد ما الـ master يبعت read request، لازم الـ PHY يرد بـ **turnaround bit** (بيت تحويل اتجاه الخط من output لـ input). بعض الـ PHY chips الرخيصة أو القديمة بتتجاهل الـ turnaround ده وبتبعت البيانات مباشرة. الـ `phy_ignore_ta_mask` بيقول للـ driver يتجاهل غياب الـ turnaround لعناوين معينة.

---

### الفرق بين نوعين من الـ platform data

| الطريقة | الملف |
|---|---|
| **Legacy platform_data** (C struct مباشرة) | `include/linux/platform_data/mdio-gpio.h` |
| **Device Tree** (DTS/YAML) | `Documentation/devicetree/bindings/net/mdio-gpio.yaml` |

على الأجهزة الحديثة، الإعدادات بتيجي من الـ Device Tree. على الأجهزة القديمة أو الـ boards اللي مبتدعمش DT، بيتم تمرير الـ struct دي مباشرة في الكود.

---

### رسم توضيحي للصورة الكبيرة

```
Board (no MDIO hardware)
         │
         ▼
  ┌─────────────┐
  │  GPIO pins  │  ← MDC pin + MDIO pin (+ optional MDO pin)
  └──────┬──────┘
         │  bitbang (software)
         ▼
  ┌─────────────────────┐
  │   mdio-gpio driver  │  ← drivers/net/mdio/mdio-gpio.c
  │                     │
  │  reads platform_data│  ← include/linux/platform_data/mdio-gpio.h
  │  (phy_mask, etc.)   │
  └──────────┬──────────┘
             │  uses
             ▼
  ┌─────────────────────┐
  │  mdio-bitbang layer │  ← drivers/net/mdio/mdio-bitbang.c
  │  (mii_bus ops)      │
  └──────────┬──────────┘
             │  registers
             ▼
  ┌─────────────────────┐
  │   PHY Library       │  ← drivers/net/phy/
  │   (phy_device)      │
  └──────────┬──────────┘
             │
             ▼
       Network Stack
```

---

### الملفات المهمة في الـ Subsystem

| الملف | الدور |
|---|---|
| `include/linux/platform_data/mdio-gpio.h` | الملف الحالي — struct الإعدادات للـ legacy platform |
| `include/linux/mdio-gpio.h` | ثوابت أرقام الـ GPIO pins (MDC=0, MDIO=1, MDO=2) |
| `include/linux/mdio-bitbang.h` | تعريف الـ `mdiobb_ctrl` و`mdiobb_ops` |
| `drivers/net/mdio/mdio-gpio.c` | الـ driver الرئيسي — بيربط الـ GPIO بالـ bitbang layer |
| `drivers/net/mdio/mdio-bitbang.c` | منطق الـ bitbang — بيحرك الـ bits فعليًا |
| `Documentation/devicetree/bindings/net/mdio-gpio.yaml` | الـ Device Tree binding |
| `drivers/net/phy/` | مكتبة الـ PHY كلها |
| `include/linux/phy.h` | تعريفات الـ `mii_bus` و`phy_device` |

---

### ملخص

الملف `mdio-gpio.h` صغير جدًا — بس دوره واضح: هو **وعاء الإعدادات** اللي بيتم تمريرها للـ driver اللي مسؤول عن تنفيذ بروتوكول MDIO باستخدام GPIO pins عادية بدل hardware مخصص. ده مهم في الأجهزة المدمجة البسيطة والقديمة اللي مبيكونش فيها MDIO controller.
## Phase 2: شرح الـ MDIO-GPIO Framework

---

### المشكلة — ليه الـ Framework ده موجود؟

في أي نظام embedded بيحتوي على Ethernet، لازم يكون في طريقة للـ MAC (Media Access Controller) إنه يتكلم مع الـ PHY chip (الـ Physical Layer chip اللي بيتعامل مع الكابل الفعلي). الطريقة القياسية دي اسمها **MDIO** (Management Data Input/Output)، وهي bus تسلسلي بيتعرف عليها IEEE 802.3 clause 22/45.

الـ MDIO bus محتاج على الأقل سلكين:
- **MDC** (Management Data Clock) — clock بيوليه الـ MAC
- **MDIO** (Management Data I/O) — بيانات ثنائية الاتجاه

**المشكلة الجوهرية:** مش كل SoC عنده hardware MDIO controller مدمج. في حالات كتير في embedded systems:

1. الـ SoC عنده MDIO controller بس متوصل بـ PHY على board تاني.
2. الـ SoC مالوش MDIO controller خالص (قديم أو بسيط).
3. المطوّر عايز يوصّل PHY على GPIOs عشان يوفّر flexibility.
4. في custom hardware الـ MDC/MDIO متوصلين على pins عشوائية من GPIO bank.

الحل؟ **bit-banging** — تنفيذ البروتوكول بالكامل software على GPIOs عادية. بس إزاي تخلي الـ kernel يعرف إن الـ MDIO bus ده مش hardware controller، إنما GPIOs؟ هنا بييجي الـ `mdio-gpio` subsystem.

---

### الحل — المنهج اللي اتبعه الـ Kernel

الـ kernel بيحل المشكلة بطبقات:

```
┌────────────────────────────────────────────┐
│        PHY Subsystem (phy.h / mii_bus)     │  <- لا يعرف حاجة عن hardware
├────────────────────────────────────────────┤
│     MDIO bit-bang layer (mdio-bitbang.h)   │  <- بيحول read/write لـ GPIO ops
├────────────────────────────────────────────┤
│     mdio-gpio driver (platform driver)     │  <- بيربط GPIO descriptor بالـ ops
├────────────────────────────────────────────┤
│     GPIO Subsystem (gpiolib)               │  <- يتعامل مع الـ hardware pin فعلاً
└────────────────────────────────────────────┘
```

الـ `mdio_gpio_platform_data` struct — اللي في الـ source file بتاعنا — هو الـ **configuration interface** اللي بيقول للـ driver:
- أنهي PHYs موجودين على الـ bus (`phy_mask`)
- أنهي PHYs نتجاهلهم لو مردوش acknowledge (`phy_ignore_ta_mask`)

---

### المعمار الكامل (Big Picture Architecture)

```
  +---------------------------------------------------------------+
  |                     User Space                                |
  |         ethtool / ip / network management tools               |
  +---------------------------+-----------------------------------+
                              | syscalls
  +---------------------------v-----------------------------------+
  |                   Network Stack (net/)                        |
  |    netdev / socket layer                                      |
  +---------------------------+-----------------------------------+
                              |
  +---------------------------v-----------------------------------+
  |              MAC Driver (e.g., fec, stmmac, mvneta)          |
  |   بيسجل mii_bus ويطلب registration مع PHY layer              |
  +----------+--------------------------------------------+------+
             | registers mii_bus                          | attaches to netdev
  +----------v-------------------------------+           |
  |          mii_bus (struct mii_bus)        |           |
  |  .read()  .write()  .reset()            |           |
  |  .phy_mask  .irq[]                      |           |
  +----------+-------------------------------+           |
             |                                  +--------v----------------+
     +-------+----------+                       |    PHY Driver           |
     |  Hardware MDIO   |                       |  (e.g., dp83867,       |
     |  Controller      |    OR                 |   ksz9031, genphy)     |
     |  (e.g., fec_     |                       +------------------------+
     |  mdio.c)         |
     +------------------+
             OR
  +--------------------------------------------------------------+
  |              mdio-gpio (platform driver)                     |
  |  +---------------------------------------------------+       |
  |  |  mdio_gpio_platform_data                          |       |
  |  |    .phy_mask           = 0xFFFFFFFE               |       |
  |  |    .phy_ignore_ta_mask = 0x00000002               |       |
  |  +---------------------------------------------------+       |
  |                                                              |
  |  GPIO[0] = MDC  ------------------------------------> PHY MDC|
  |  GPIO[1] = MDIO <----------------------------------> PHY MDIO|
  |  GPIO[2] = MDO  (optional, for unidirectional mode)         |
  +--------------------------------------------------------------+
             |
  +----------v---------------------------------------------------+
  |           mdio-bitbang layer (mdiobb_ctrl)                   |
  |  .set_mdc()       -> gpio_set_value(mdc_gpio, level)        |
  |  .set_mdio_dir()  -> gpio_direction_input/output(mdio_gpio) |
  |  .set_mdio_data() -> gpio_set_value(mdio_gpio, value)       |
  |  .get_mdio_data() -> gpio_get_value(mdio_gpio)              |
  +----------+---------------------------------------------------+
             |
  +----------v---------------------------------------------------+
  |              GPIO Subsystem (gpiolib)                        |
  |  gpiod_set_value() / gpiod_get_value()                      |
  |  بيتكلم مع الـ GPIO controller driver مباشرة               |
  +-------------------------------------------------------------+
```

---

### الـ Subsystems التانية اللي لازم تعرفها الأول

قبل ما تكمل، خد بالك من التلاتة دول:

- **GPIO Subsystem (gpiolib):** بيمثّل كل pin في الـ SoC كـ `gpio_desc`، وبيوفر API موحد (`gpiod_*`) للـ drivers.
- **PHY Subsystem (`struct mii_bus`, `struct phy_device`):** الـ abstraction layer اللي فوق الـ MDIO، بيكتشف الـ PHYs وبيشيلهم مع drivers مناسبة.
- **Platform Device/Driver model:** الـ kernel يربط `platform_device` بـ `platform_driver` عن طريق name أو device tree compatible string. الـ `platform_data` هو الـ private config pointer.

---

### تشبيه من الواقع — مع Mapping كامل

تخيل إنك عندك **مدير شبكة** (الـ MAC driver) محتاج يكلم **فني صيانة** (الـ PHY chip) عشان يعرف حالة الكابل.

بالعادة فيه **خط تليفون مخصص** (hardware MDIO controller) بينهم. بس في مشروعك الجديد، الخط الجاهز مش موجود.

الحل؟ المدير بيكلم الفني عن طريق **واكي توكي عادي** (GPIO bit-banging).

| التشبيه | الـ Kernel Equivalent |
|---------|----------------------|
| المدير (MAC driver) | `struct net_device` + MAC driver |
| الفني (PHY chip) | `struct phy_device` |
| الخط التليفوني المخصص | hardware MDIO controller |
| الواكي توكي | GPIO bit-banging (mdio-gpio) |
| قواعد الحوار (متى تتكلم، متى تسمع) | MDIO protocol timing (MDC pulses) |
| دليل أرقام الفنيين | `phy_mask` في `mdio_gpio_platform_data` |
| فني مش بيرد (TA error) | `phy_ignore_ta_mask` |
| السكرتير اللي بيوجه المكالمات | `mdiobb_ctrl` + `mii_bus` |
| شبكة التليفون التحتية | GPIO subsystem (gpiolib) |

**الـ mapping أعمق:**

- **دليل الأرقام (`phy_mask`):** زي ما في الدليل أرقام موجودة وأرقام غير مخصصة — الـ `phy_mask` بيقول للـ kernel "متحاولش تتواصل مع الـ PHY address التانية دي، مش موجودة على bus ده."
- **فني مش بيرد (`phy_ignore_ta_mask`):** في MDIO clause 22، بعد الـ read request لازم الـ PHY يحط الـ bus في High-Z (Turn-Around). لو PHY معين مش بيعمل كده (buggy hardware)، بدل ما الـ kernel يعتبره error، `phy_ignore_ta_mask` بيقوله "استحمل الـ PHY ده حتى لو مردش صح."

---

### الـ Core Abstraction — الفكرة المركزية

الـ core abstraction هو **`struct mii_bus`** — وده موجود في `include/linux/phy.h` وبيمثل الـ MDIO bus بالكامل بشكل مجرد:

```c
/* الـ bus اللي الـ PHYs بتتكلم عليه */
struct mii_bus {
    const char *name;
    char id[MII_BUS_ID_SIZE];

    /* function pointers - الـ actual implementation جوه الـ driver */
    int (*read)(struct mii_bus *bus, int addr, int regnum);
    int (*write)(struct mii_bus *bus, int addr, int regnum, u16 val);
    int (*reset)(struct mii_bus *bus);

    /* لو bit-bang: الـ mutex بيحمي الـ GPIO operations */
    struct mutex mdio_lock;

    struct device *parent;
    struct phy_device *phy_map[PHY_MAX_ADDR];  /* الـ PHYs المكتشفة */

    u32 phy_mask;           /* بيتاخد من platform_data */
    u32 phy_ignore_ta_mask; /* بيتاخد من platform_data */
    /* ... */
};
```

الـ mdio-gpio driver بياخد الـ `mdio_gpio_platform_data` ويحطها في `mii_bus.phy_mask` و `mii_bus.phy_ignore_ta_mask`.

---

### الـ Structs وعلاقتها ببعض

```
  mdio_gpio_platform_data          mdiobb_ctrl
  +----------------------+        +----------------------+
  | phy_mask       u32   |        | ops  *mdiobb_ops     |---> set_mdc()
  | phy_ignore_ta  u32   |        | override_op_c22 uint |     set_mdio_dir()
  +----------+-----------+        | op_c22_read    u8    |     set_mdio_data()
             |                    | op_c22_write   u8    |     get_mdio_data()
             | بيتحول             +----------+-----------+
             | في probe()                    |
             v                              | alloc_mdio_bitbang()
         mii_bus                            |
  +----------------------+                  |
  | phy_mask       u32 <-+<----------------+
  | phy_ignore_ta  u32   |  بيعمل mii_bus ويربطه
  | read()               |  بالـ mdiobb_ctrl
  | write()              |
  | phy_map[32]          |  <- بعد mdiobus_register()
  +----------------------+    بيتملى بـ phy_device
```

---

### الـ GPIO Pin Indices — المعرّفات الثابتة

```c
/* من include/linux/mdio-gpio.h */
#define MDIO_GPIO_MDC   0   /* أول GPIO resource = clock */
#define MDIO_GPIO_MDIO  1   /* تاني GPIO resource = data (bidirectional) */
#define MDIO_GPIO_MDO   2   /* تالت GPIO resource = data output فقط (اختياري) */
```

الـ `MDIO_GPIO_MDO` بييجي في حالة الـ unidirectional hardware — يعني الـ MDO والـ MDI منفصلين فيزيائياً (نادر، بس موجود في بعض الـ MAC chips القديمة).

---

### الـ Platform Data وإزاي بتتمرر

في الـ board file القديم (non-DT):

```c
/* board file - e.g., arch/arm/mach-xxx/board-xxx.c */
static struct mdio_gpio_platform_data my_mdio_pdata = {
    /* skip PHY address 0, scan only 1-31 */
    .phy_mask           = 0x00000001,

    /* PHY at address 1 has broken TA, ignore it */
    .phy_ignore_ta_mask = 0x00000002,
};

static struct platform_device my_mdio_gpio_dev = {
    .name = "mdio-gpio",
    .id   = 0,
    .dev  = {
        .platform_data = &my_mdio_pdata,  /* pointer للـ struct */
    },
    /* GPIOs معرّفة في resources */
};
```

في الـ Device Tree (الأشيع في ARM embedded):

```dts
/* device tree node */
mdio-gpio {
    compatible = "virtual,mdio-gpio";
    gpios = <&gpio0 10 0>,   /* MDC  */
            <&gpio0 11 0>;   /* MDIO */

    #address-cells = <1>;
    #size-cells = <0>;

    phy0: ethernet-phy@1 {
        reg = <1>;
    };
};
```

---

### الـ MDIO Protocol Timing — ليه الـ Bit-Bang محتاج Layer منفصلة؟

الـ MDIO Clause 22 Read Frame:

```
PRE      ST  OP   PHYAD  REGAD  TA        DATA
11...1   01  10   AAAAA  RRRRR  Z0   DDDDDDDDDDDDDDDD
(32 bits)                  (turnaround)   (16 bits)

PRE   = 32 ones (preamble — synchronization)
ST    = 01 (start of frame)
OP    = 10 = read  /  01 = write
PHYAD = 5-bit PHY address (0..31)
REGAD = 5-bit register address
TA    = Turnaround: master releases (Z), PHY drives 0
DATA  = 16-bit register value
```

كل bit في الـ frame ده محتاج:
1. تحديد اتجاه الـ MDIO pin (input/output)
2. كتابة أو قراءة الـ bit
3. رفع MDC
4. تنزيل MDC

يعني لكل 16-bit register read، بيحصل 60+ GPIO operation. الـ `mdiobb_ops` بتجرد ده كله في 4 functions بسيطة — ده هو سبب وجود الـ layer المنفصلة.

```c
/* الـ bitbang layer بيعمل read بالشكل ده تقريباً: */
int mdiobb_read_c22(struct mii_bus *bus, int phy, int reg)
{
    struct mdiobb_ctrl *ctrl = bus->priv;

    /* preamble: 32 ones */
    mdiobb_send_num(ctrl, 0xffffffff, 32);

    /* start + read opcode + phy addr + reg addr */
    mdiobb_send_num(ctrl, (0x1 << 10) | (0x2 << 8) |
                          (phy << 3) | reg, 14);

    /* turnaround: release bus, read TA bit from PHY */
    mdiobb_getbit(ctrl);

    /* read 16-bit data */
    return mdiobb_get_num(ctrl, 16);
}
```

---

### ملكية الـ Subsystem — إيه اللي بيشيله وإيه اللي بيفوّضه

| المهمة | مين المسؤول |
|--------|-------------|
| تعريف MDIO bus كـ `mii_bus` | PHY subsystem (`phy.h`) |
| bit-banging timing (MDC pulses, data setup/hold) | `mdio-bitbang` layer |
| تحريك الـ GPIO فعلاً (high/low) | GPIO subsystem (gpiolib) |
| تحديد أنهي addresses فيها PHYs | `mdio_gpio_platform_data.phy_mask` |
| التعامل مع broken PHY TA | `mdio_gpio_platform_data.phy_ignore_ta_mask` |
| اكتشاف الـ PHYs وربطها بـ drivers | PHY subsystem (auto-scan) |
| التفاوض على speed/duplex | PHY driver + phylink |
| قراءة/كتابة registers الـ PHY | `mdiobb_read_c22/c45` / `mdiobb_write_c22/c45` |

**الـ mdio-gpio driver بيملك:**
- ربط الـ GPIOs بالـ `mdiobb_ops`
- تمرير الـ platform data للـ `mii_bus`
- إدارة lifecycle الـ bus (alloc, register, unregister, free)

**بيفوّض لـ:**
- `mdio-bitbang`: الـ protocol timing بالكامل
- `gpiolib`: الـ actual pin manipulation
- PHY subsystem: الـ discovery والـ management
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

### الـ Flags والـ Masks — Cheatsheet

الملف بسيط جداً — ما فيهوش enums ولا config options معقدة. الـ fields الجوه الـ struct الوحيدة بتستخدم bitmasks:

| الـ Field | النوع | الوصف |
|---|---|---|
| `phy_mask` | `u32` | bitmask — كل bit بيمثل PHY address (0–31). لو الـ bit = 1 → الـ PHY ده **مش** موجود، يتجاهله |
| `phy_ignore_ta_mask` | `u32` | bitmask — الـ PHYs اللي لازم نتجاهل فيها الـ **TA** (turnaround) bit في الـ MDIO protocol |

**مثال عملي:**

```c
/* PHY addresses 0 and 3 موجودين بس، الباقي مش موجود */
/* phy_mask = ~((1 << 0) | (1 << 3)) = 0xFFFFFFF6 */

struct mdio_gpio_platform_data pdata = {
    .phy_mask         = 0xFFFFFFF6, /* ignore all except PHY 0 and PHY 3 */
    .phy_ignore_ta_mask = 0,        /* no TA issues on this board */
};
```

لو الـ `phy_ignore_ta_mask` بيت معين = 1 → الـ driver مش هيستنى الـ turnaround cycle من الـ PHY ده (بعض PHYs القديمة أو الغريبة مش بتعمل TA صح).

---

### الـ Structs

#### `mdio_gpio_platform_data`

**الغرض:** يحمل الـ board-specific configuration اللي بتتحطها الـ platform code (أو device tree) وبيقراها الـ `mdio-gpio` driver وقت الـ probe.

```c
struct mdio_gpio_platform_data {
    u32 phy_mask;           /* which PHY addresses to SKIP during bus scan */
    u32 phy_ignore_ta_mask; /* which PHYs have broken turnaround bit */
};
```

**الـ Fields بالتفصيل:**

- **`phy_mask`** — الـ MDIO bus scan بيجرب كل الـ 32 address (0 → 31). الـ driver بيعمل `if (phy_mask & (1 << addr)) skip;` — يعني لو الـ bit = 1، الـ address ده محجوب. ده بيمنع الـ ghost PHY detections على الـ bus.

- **`phy_ignore_ta_mask`** — الـ MDIO read cycle بيتطلب 2 bits turnaround بين الـ request والـ response. بعض الـ PHYs مش بتدي TA صح فبيحصل timeout. الـ bit ده بيقول للـ driver "اقرأ من غير ما تستنى TA".

**الارتباطات بالـ Structs التانية:**

الـ struct ده بيتحط في `platform_device.dev.platform_data` ولو بنستخدم Device Tree فالـ driver بيبني الـ struct ده من الـ DT properties تلقائياً.

```
platform_device
  └── dev
       └── platform_data  →  mdio_gpio_platform_data
                                ├── phy_mask
                                └── phy_ignore_ta_mask
```

---

### Struct Relationship Diagram

```
Board / DTS
    │
    │  (defines GPIO pins + phy_mask + phy_ignore_ta_mask)
    ▼
mdio_gpio_platform_data          ← هو الـ struct في الـ header ده
    │
    │  (passed via platform_device.dev.platform_data)
    ▼
mdio_gpio_bus  (في drivers/net/phy/mdio-gpio.c)
    ├── mdc_gpio          (clock GPIO pin)
    ├── mdio_gpio         (data GPIO pin)
    ├── phy_mask          ← مأخوذ من platform_data
    ├── phy_ignore_ta_mask← مأخوذ من platform_data
    └── mii_bus  ─────────────────────────────────────┐
                                                       ▼
                                                  mii_bus (في include/linux/phy.h)
                                                       ├── read()   → mdio_gpio_read()
                                                       ├── write()  → mdio_gpio_write()
                                                       ├── phy_mask ← copy هنا كمان
                                                       └── priv  →  mdio_gpio_bus
```

---

### Lifecycle Diagram

```
[Boot / Platform Init]
        │
        ▼
platform_device_register()
  └── dev.platform_data = &mdio_gpio_platform_data
        │
        ▼
mdio_gpio_probe()          ← في drivers/net/phy/mdio-gpio.c
  ├── dev_get_platdata()   → يجيب mdio_gpio_platform_data
  ├── alloc_mdio_bitbang() → يعمل mdio_gpio_bus + يجهز GPIO pins
  ├── new_bus->phy_mask    = pdata->phy_mask
  ├── new_bus->phy_ignore_ta_mask = pdata->phy_ignore_ta_mask
  └── mdiobus_register()  → يعمل scan للـ bus ويلاقي الـ PHYs
        │
        ▼
[Runtime — Network Stack يكلم الـ PHY]
  └── phy_read() / phy_write()
        → mii_bus->read/write
          → mdio_gpio_read/write
            → GPIO bit-bang على MDC + MDIO pins
        │
        ▼
[Driver Unload / Board Shutdown]
  └── mdio_gpio_remove()
        ├── mdiobus_unregister()
        ├── free_mdio_bitbang()
        └── GPIO pins released
```

---

### Call Flow Diagram

```
Network Stack يطلب PHY register read
    │
    ▼
phy_read(phydev, regnum)
    │
    ▼
mdiobus_read(bus, addr, regnum)          ← checks phy_mask here
    │  (if addr bit set in phy_mask → return -ENODEV immediately)
    ▼
bus->read(bus, addr, regnum)
    │
    ▼
mdio_gpio_read()                         ← الـ actual bit-bang
    ├── mdio_dir(bus, OUTPUT)            → GPIO direction set
    ├── mdio_out(bus, 1)                 → preamble (32 ones)
    ├── send_preamble()                  → clock MDC 32 times
    ├── send_read_cmd(addr, regnum)      → clock out ST+OP+PHYAD+REGAD
    ├── mdio_dir(bus, INPUT)             → turnaround — release bus
    │     (if addr in phy_ignore_ta_mask → skip TA check)
    ├── mdio_get(bus)                    → read 16 bits
    └── return data
```

---

### Locking Strategy

الـ struct `mdio_gpio_platform_data` نفسه **مش محتاج locking** لأنه read-only بعد الـ probe — بيتحط مرة واحدة وقت الـ initialization ومحدش بيعدله بعد كده.

الـ locking الحقيقي بيحصل في الـ `mii_bus`:

| الـ Lock | المكان | بيحمي إيه |
|---|---|---|
| `mii_bus->mdio_lock` (mutex) | `struct mii_bus` | الـ MDIO bus access — بيمنع concurrent read/write على نفس الـ bus |
| `mdio_gpio_bus` internal | في الـ driver | الـ GPIO state أثناء الـ bit-bang |

```
mdiobus_read()
    │
    mutex_lock(&bus->mdio_lock)    ← يمنع أي حد تاني يكلم الـ bus
    │
    bus->read()  →  GPIO bit-bang  (atomic من ناحية الـ bus)
    │
    mutex_unlock(&bus->mdio_lock)
```

مفيش locking داخل `mdio_gpio_platform_data` لأنه static configuration — زي ما الـ hardware registers بتتقرا مرة وقت الـ boot.
## Phase 4: شرح الـ Functions

### ملاحظة أولية

الملف `include/linux/platform_data/mdio-gpio.h` هو **header-only platform data file** — مش فيه functions خالص. الملف بيحتوي على struct واحد بس هو `mdio_gpio_platform_data`، اللي بيتبعت كـ platform data لـ driver الـ MDIO-over-GPIO (`drivers/net/phy/mdio-gpio.c`).

مفيش API calls، مفيش macros، مفيش inline functions — pure data structure definition.

---

### ملخص المحتوى (Cheatsheet)

| العنصر | النوع | الوصف |
|---|---|---|
| `mdio_gpio_platform_data` | `struct` | platform data للـ MDIO bus المبني على GPIO |
| `phy_mask` | `u32` | bitmask للـ PHY addresses المراد تجاهلها من الـ scan |
| `phy_ignore_ta_mask` | `u32` | bitmask للـ PHYs اللي بتتجاهل Turn-Around في الـ clause 22 |

---

### الـ Data Structure الوحيدة

#### `struct mdio_gpio_platform_data`

```c
struct mdio_gpio_platform_data {
    u32 phy_mask;           /* bitmask: skip these PHY addresses during bus scan */
    u32 phy_ignore_ta_mask; /* bitmask: PHYs that don't drive TA bit correctly */
};
```

##### الغرض

الـ `mdio_gpio_platform_data` هي الـ **platform data structure** اللي بيبعتها الـ board code أو الـ device tree glue code للـ driver `mdio-gpio` عن طريق `platform_device.dev.platform_data`. الـ driver بيعمل `dev_get_platdata()` ويقرأ منها.

---

#### الـ Fields بالتفصيل

##### `phy_mask` — `u32`

```c
u32 phy_mask;
```

- **الـ MDIO bus** بيدعم 32 عنوان (0–31) لـ PHY devices.
- الـ `phy_mask` هو **bitmask** — لو الـ bit رقم N مضروب، يبقى الـ bus scanner بيتجاهل الـ PHY عند العنوان N.
- بيتبعت مباشرة لـ `mdiobus_alloc()` ثم `bus->phy_mask = pdata->phy_mask;` جوا الـ driver.
- مثال: لو عندك PHY بس على العنوان 1، تعمل `phy_mask = ~BIT(1)` عشان تتجنب الـ scan للعناوين التانية وتوفر وقت الـ probe.

##### `phy_ignore_ta_mask` — `u32`

```c
u32 phy_ignore_ta_mask;
```

- في الـ **MDIO Clause 22 protocol**، بعد الـ address phase في الـ read cycle، في بيت اسمه **Turn-Around (TA)** المفروض الـ PHY يشيله (pull low). بعض الـ PHYs المعيبة أو الـ legacy chips مش بتعمل كده.
- الـ `phy_ignore_ta_mask` بيحدد الـ PHYs اللي لازم الـ bus controller يتجاهل الـ TA bit فيها عشان القراءة تشتغل صح.
- بيتبعت لـ `bus->phy_ignore_ta_mask` جوا الـ driver بنفس الطريقة.

---

### السياق الكامل — كيف بيتاستخدم الـ struct ده؟

الـ header ده جزء من منظومة أكبر:

```
Board Code / DTS
      │
      ▼
platform_device  ──► .dev.platform_data = &mdio_gpio_platform_data { ... }
      │
      ▼
mdio-gpio driver (drivers/net/phy/mdio-gpio.c)
      │
      ├── dev_get_platdata(dev)  → يجيب الـ struct
      ├── mdiobus_alloc()        → يخصص struct mii_bus
      ├── bus->phy_mask          ← من phy_mask
      ├── bus->phy_ignore_ta_mask ← من phy_ignore_ta_mask
      └── mdiobus_register()    → يسجل الـ bus في الـ kernel
```

**الـ GPIO lines** نفسها (MDC و MDIO) بتيجي من الـ device tree أو الـ GPIO lookup table — مش من الـ struct ده. الـ struct ده بس للـ PHY scanning configuration.

---

### ملاحظات للـ Embedded Developer

- لو بتستخدم **Device Tree** بدل platform data مباشرة، الـ `of_mdio` subsystem بيعمل parse للـ `phy-mask` property تلقائياً ومش محتاج الـ struct ده.
- الـ struct ده مهم بس في الـ **legacy board files** (`arch/*/mach-*/board-*.c`) اللي لسه مش migrate لـ DT.
- الـ `u32` بدل `unsigned int` عشان الـ MDIO bus عنده exactly 32 عنوان — مش implementation-defined width.
## Phase 5: دليل الـ Debugging الشامل

الـ subsystem ده هو **MDIO-GPIO bitbang driver** — بيبني **MDIO bus** (Management Data Input/Output) فوق GPIO pins عادية بدل hardware MDIO controller. كل pulse في الـ MDC clock وكل bit في الـ MDIO data بتتعمل يدوياً عبر `gpiod_set_value_cansleep()`. الـ platform data struct (`mdio_gpio_platform_data`) بتحتوي على حقلين فقط:

- **`phy_mask`**: bitmask للـ PHY addresses اللي الـ bus يتجاهلها أثناء الـ scan
- **`phy_ignore_ta_mask`**: bitmask للـ PHYs اللي بتتجاهل منها الـ turnaround bit error

أي bug ممكن يجي من: GPIO direction/value خاطئ، PHY مش بيرد، DT مش صح، أو phy_mask بيخبي PHY.

---

### Software Level

#### 1. debugfs entries

الـ driver نفسه ما بيكتبش entries في debugfs، لكن الـ GPIO subsystem والـ MDIO layer بيوفروا معلومات حيوية:

```bash
# أهم entry: حالة كل GPIO pins في النظام
cat /sys/kernel/debug/gpio

# مثال output صح:
# gpiochip0: GPIOs 0-53, parent: platform/fe200000.gpio
#  gpio-17 (mdc             |mdio-gpio        ) out lo
#  gpio-18 (mdio            |mdio-gpio        ) in
#  gpio-19 (mdo             |mdio-gpio        ) out hi

# الـ MDIO bus devices
ls /sys/bus/mdio_bus/devices/
# مثال: gpio-0:00  gpio-0:01
```

**الـ GPIO consumer name "mdio-gpio" في السطر ده هو الدليل إن الـ driver صح استولى على الـ pins.**

#### 2. sysfs entries

| المسار | المحتوى |
|--------|---------|
| `/sys/bus/mdio_bus/devices/gpio-0:XX/phy_id` | الـ PHY ID اللي اتقرأ من الـ hardware |
| `/sys/bus/mdio_bus/devices/gpio-0:XX/link` | حالة الـ link (0/1) |
| `/sys/bus/mdio_bus/devices/gpio-0:XX/speed` | سرعة الـ link بالـ Mbps |
| `/sys/bus/mdio_bus/devices/gpio-0:XX/duplex` | full=1 / half=0 |
| `/sys/bus/platform/drivers/mdio-gpio/` | الـ platform devices المربوطة بالـ driver |
| `/sys/class/gpio/gpioXX/direction` | اتجاه الـ GPIO pin (in/out) |
| `/sys/class/gpio/gpioXX/value` | القيمة الحالية للـ GPIO |

```bash
# تحقق إن الـ platform device اتبند بالـ driver
ls /sys/bus/platform/drivers/mdio-gpio/

# شوف PHY ID (0xFFFFFFFF = PHY غير موجود)
cat /sys/bus/mdio_bus/devices/gpio-0:01/phy_id

# شوف حالة الـ link
cat /sys/bus/mdio_bus/devices/gpio-0:01/link
```

#### 3. ftrace — tracepoints وـ events

```bash
# فعّل tracing للـ MDIO bus operations
echo 1 > /sys/kernel/debug/tracing/events/mdio/enable

# أو على event محدد
echo 1 > /sys/kernel/debug/tracing/events/mdio/mdio_access/enable

# فعّل GPIO events (الأهم للـ bitbang debugging)
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable

# فلتر على GPIO pins محددة بس
echo 'gpio >= 17 && gpio <= 19' > /sys/kernel/debug/tracing/events/gpio/gpio_value/filter
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_value/enable

# شوف النتيجة في real-time
cat /sys/kernel/debug/tracing/trace_pipe | grep -E "mdc|mdio|gpio-1[789]"

# أو snapshot
echo 1 > /sys/kernel/debug/tracing/tracing_on
ethtool eth0 > /dev/null 2>&1    # trigger MDIO activity
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

**مثال على output مفيد:**
```
# MDIO read ناجح:
mdio_access: bus=gpio-0 phy_id=1 read=1 regnum=1 val=0x796d

# val=0xFFFF → MDIO wire مش موصل أو PHY غايب
# val=0x0000 → PHY في reset أو power-down
```

#### 4. printk / dynamic debug

```bash
# فعّل dynamic debug للـ mdio-gpio driver
echo "module mdio_gpio +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل dynamic debug للـ bitbang layer
echo "module mdio_bitbang +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل للـ GPIO subsystem
echo "module gpiolib +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل مع line numbers وـ function names
echo "file drivers/net/mdio/mdio-gpio.c +pflmt" > /sys/kernel/debug/dynamic_debug/control

# شوف النتيجة
dmesg | grep -E "mdio|gpio|mdc|bitbang"

# زود الـ loglevel لشوف كل الـ messages
echo 8 > /proc/sys/kernel/printk
```

#### 5. Kernel Config options للـ debugging

| الـ Config | الغرض |
|-----------|-------|
| `CONFIG_MDIO_GPIO=m` | الـ driver — compile كـ module لتسهيل reload |
| `CONFIG_MDIO_BITBANG` | الـ bitbang core — لازم يكون on |
| `CONFIG_PHYLIB` | الـ PHY management layer — أساسي |
| `CONFIG_MDIO_DEVICE` | base class للـ MDIO devices |
| `CONFIG_DEBUG_GPIO` | يضيف WARN لو GPIO conflict أو wrong direction |
| `CONFIG_GPIO_SYSFS` | يعرض GPIO في sysfs للـ manual testing |
| `CONFIG_TRACING` | لتفعيل ftrace infrastructure |
| `CONFIG_DYNAMIC_DEBUG` | لتفعيل `dev_dbg` runtime |
| `CONFIG_KALLSYMS_ALL` | لـ stack traces أوضح في WARN_ON |
| `CONFIG_PROVE_LOCKING` | يكشف locking bugs في الـ bus access |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "CONFIG_MDIO|CONFIG_PHYLIB|CONFIG_DEBUG_GPIO|CONFIG_GPIO_SYSFS"
```

#### 6. أدوات خاصة بالـ subsystem

**`phytool`** — أداة userspace لقراءة/كتابة MDIO registers مباشرة:

```bash
# تثبيت
apt-get install phytool   # Debian/Ubuntu

# قراءة Basic Control Register (reg 0) من PHY عند address 1
phytool read eth0/1/0

# قراءة Basic Status (reg 1) — bit 2 = link up/down
phytool read eth0/1/1

# قراءة PHY Identifier (reg 2 و3)
phytool read eth0/1/2
phytool read eth0/1/3

# reset الـ PHY (bit 15 في reg 0)
phytool write eth0/1/0 0x8000

# scan كل الـ PHY addresses (0-31)
for addr in $(seq 0 31); do
    id=$(phytool read eth0/$addr/2 2>/dev/null)
    [ "$id" != "0xffff" ] && [ "$id" != "0x0000" ] && \
        echo "PHY at addr $addr: OUI=$(phytool read eth0/$addr/2):$(phytool read eth0/$addr/3)"
done
```

**`gpioget` / `gpioset`** — اختبار GPIO بدون kernel driver:

```bash
# اقرأ حالة MDIO pin
gpioget gpiochip0 18

# test MDC toggling يدوياً
gpioset gpiochip0 17=1
gpioset gpiochip0 17=0
```

**`ethtool`** للـ PHY info:
```bash
ethtool -i eth0          # معلومات الـ driver المرتبط
ethtool eth0             # link speed/duplex/status
ethtool -d eth0          # register dump (لو الـ driver بيدعمه)
```

#### 7. رسائل الـ Error الشائعة

| الـ Error Message | المعنى | الحل |
|------------------|--------|------|
| `mdio-gpio: failed to get alias id` | الـ DT ما فيهوش `aliases { mdio-gpio0 = ... }` | زود الـ aliases أو تجاهل — هيستخدم 0 كـ default |
| `devm_gpiod_get_index returned -ENOENT` | GPIO index مش محدد في الـ DT `gpios` property | الـ DT لازم يحدد 2 GPIOs (MDC + MDIO) على الأقل |
| `devm_gpiod_get_index returned -EPROBE_DEFER` | الـ GPIO controller لسه مش جاهز | الـ kernel هيعيد الـ probe تلقائياً — انتظر |
| `of_mdiobus_register failed` | فشل تسجيل الـ bus | تحقق من الـ DT صح وـ bus ID مش مكرر |
| `phy_id = 0xffffffff` في sysfs | الـ MDIO read بيرجع 0xFFFF — wire مش موصل أو PHY بلا power | افحص الـ hardware connections |
| `phy N not found on gpio bus` | الـ `phy_mask` بيخبي هذا الـ PHY address | اضبط `phy_mask = 0` للـ full scan |
| `gpio-XX: is already claimed` | الـ GPIO pin مستخدمة من driver تاني | تحقق من الـ DT مفيش overlap |
| `alloc_mdio_bitbang: failed` | memory allocation فشل | rare — تحقق من available memory |
| `MDIO read timed out` | الـ PHY مش بيرد على الـ frames | تحقق من MDC/MDIO wiring والـ PHY power |
| `mdiobus_register failed: bus ID in use` | الـ bus ID مكرر مع bus تاني | غيّر `bus_id` أو الـ DT alias |

#### 8. نقاط استراتيجية لـ `dump_stack()` / `WARN_ON()`

```c
/* نقطة 1: في mdio_gpio_get_data() — تحقق من GPIO acquisition */
static int mdio_gpio_get_data(struct device *dev,
                              struct mdio_gpio_info *bitbang)
{
    bitbang->mdc = devm_gpiod_get_index(dev, NULL, MDIO_GPIO_MDC,
                                        GPIOD_OUT_LOW);
    if (IS_ERR(bitbang->mdc)) {
        /* WARN + dump_stack لمعرفة call chain كاملاً */
        WARN(1, "mdio-gpio: MDC GPIO failed: %ld\n",
             PTR_ERR(bitbang->mdc));
        dump_stack();
        return PTR_ERR(bitbang->mdc);
    }
    ...
}

/* نقطة 2: في mdc_set() — تحقق من NULL dereference محتمل */
static void mdc_set(struct mdiobb_ctrl *ctrl, int what)
{
    struct mdio_gpio_info *bitbang =
        container_of(ctrl, struct mdio_gpio_info, ctrl);

    /* لو NULL هيطبع backtrace + kernel oops بدل silent corruption */
    WARN_ON(!bitbang->mdc);
    gpiod_set_value_cansleep(bitbang->mdc, what);
}

/* نقطة 3: في mdio_gpio_bus_init() — تحقق من bus allocation */
static struct mii_bus *mdio_gpio_bus_init(struct device *dev,
                                          struct mdio_gpio_info *bitbang,
                                          int bus_id)
{
    ...
    new_bus = alloc_mdio_bitbang(&bitbang->ctrl);
    if (WARN_ON(!new_bus))   /* early warning قبل NULL deref */
        return NULL;
    ...
}

/* نقطة 4: في mdio_dir() — تحقق من consistency */
static void mdio_dir(struct mdiobb_ctrl *ctrl, int dir)
{
    struct mdio_gpio_info *bitbang =
        container_of(ctrl, struct mdio_gpio_info, ctrl);

    /* لو MDO موجود لكن MDIO مش موجود — حالة غير منطقية */
    WARN_ON(bitbang->mdo && !bitbang->mdio);
    ...
}
```

---

### Hardware Level

#### 1. التحقق إن حالة الـ hardware تطابق الـ kernel state

الـ MDIO-GPIO driver بيعرف 3 pins:
- **MDC** (index 0 = `MDIO_GPIO_MDC`): clock — دايمًا output
- **MDIO** (index 1 = `MDIO_GPIO_MDIO`): data — bidirectional
- **MDO** (index 2 = `MDIO_GPIO_MDO`): separate data output — optional

```bash
# snapshot لحالة الـ GPIO pins
cat /sys/kernel/debug/gpio | grep -A10 "mdio-gpio"

# المتوقع أثناء idle:
# gpio-17 (mdc   |mdio-gpio) out lo    → MDC في الـ idle state
# gpio-18 (mdio  |mdio-gpio) in        → MDIO configured as input (ready for read)

# المتوقع أثناء write transaction:
# gpio-18 (mdio  |mdio-gpio) out hi    → MDIO switched to output

# لو MDC مش "out" → الـ probe فشل أو GPIO مش اتجاب
# لو MDIO دايمًا "out" ومش بيتغير → mdio_dir() مش بتشتغل صح
```

```
GPIO State Machine أثناء MDIO Read (Clause 22):
═══════════════════════════════════════════════
MDC:   ─┐ ┌─┐ ┌─┐ ┌─┐ ┌─   (toggling)
        └─┘ └─┘ └─┘ └─┘
MDIO:  ════OUT══════ ─Z─ ════IN════
       (preamble+header) (TA) (data from PHY)
```

#### 2. Register Dump Techniques

**عبر `devmem2` للـ GPIO controller registers:**
```bash
# مثال: Raspberry Pi 4 (BCM2711) GPIO base
# احصل على الـ base address من DT
cat /proc/device-tree/soc/gpio@7e200000/reg | xxd

# قرأ GPIO Function Select (GPFSEL) — يحدد direction كل pin
devmem2 0xFE200000 w   # GPFSEL0 — pins 0-9
devmem2 0xFE200004 w   # GPFSEL1 — pins 10-19

# قرأ GPIO Level registers — القيمة الفعلية
devmem2 0xFE200034 w   # GPLEV0 — pins 0-31
devmem2 0xFE200038 w   # GPLEV1 — pins 32-53

# تفسير GPFSEL: كل 3 bits لـ pin واحد
# 000 = Input, 001 = Output
# lو GPIO17 = bits [23:21] في GPFSEL1
# مثلاً: 0x00200000 = GPIO17 as Output
```

**بديل أأمن عبر `/sys/class/gpio`:**
```bash
GPIO_MDC=17
GPIO_MDIO=18

# تحقق من الـ direction
cat /sys/class/gpio/gpio${GPIO_MDC}/direction    # يجب أن يكون: out
cat /sys/class/gpio/gpio${GPIO_MDIO}/direction   # يتغير: in أو out

# تحقق من القيمة الحالية
cat /sys/class/gpio/gpio${GPIO_MDC}/value
cat /sys/class/gpio/gpio${GPIO_MDIO}/value
```

**عبر `/proc/iomem` لمعرفة الـ GPIO base:**
```bash
cat /proc/iomem | grep gpio
# fe200000-fe2000b3 : /soc/gpio@fe200000
```

#### 3. Logic Analyzer / Oscilloscope Tips

**إعداد الـ channels:**
```
CH0 → MDC  pin (GPIO17): Clock signal
CH1 → MDIO pin (GPIO18): Bidirectional data
CH2 → MDO  pin (GPIO19): Separate output (optional)
CH3 → PHY RESET pin: تأكد مش في reset

Trigger: Rising edge على CH0 (MDC)
Sample rate: >= 10 MHz (الـ bitbang عادةً 1-2 MHz)
Record length: >= 64 clocks لـ transaction واحدة
```

**MDIO Clause 22 frame structure للتحقق:**
```
Bit:  |<─── 32 ───>|ST|OP|<─PA─>|<─RA─>|TA|<──── DATA ────>|
      |111...111111|01|10|PPPPP |RRRRR |Z0|DDDDDDDDDDDDDDDD|
      (preamble)    (read)       (regad) (PHY drives)
```

**ما تدور عليه:**
- هل الـ MDC بيشتغل فعلاً بعد `of_mdiobus_register`؟ — لو لأ، الـ probe فشل
- هل preamble = 32 ones؟ — لو أقل، مشكلة في `mdiobb_cmd()`
- هل الـ PHY بيحط `0` على الـ TA bit أثناء read؟ — لو خلّاه `1`، استخدم `phy_ignore_ta_mask`
- هل الـ voltage levels صح؟ — 3.3V أو 1.8V حسب الـ PHY VDD
- هل الـ rise time < 20ns؟ — لو أبطأ، كابيسيتانس زيادة على الـ trace

**Logic analyzer software:**
```
Saleae Logic 2: Add Protocol → MDIO
PulseView (sigrok): mdio protocol decoder
```

#### 4. Hardware Issues الشائعة وعلاماتها في الـ kernel log

| Hardware Problem | الـ kernel log pattern | التشخيص والحل |
|----------------|----------------------|--------------|
| MDC pin مفتوح (open circuit) | `MDIO read timed out` مراراً | الـ PHY ما بيشوفش clock — افحص الـ PCB trace |
| MDIO pull-up مفقود | random `0xFFFF` أو بيانات عشوائية | أضف 1kΩ pull-up إلى VDD_IO |
| PHY بلا power | `phy_id = 0xffffffff` في sysfs | افحص power rail للـ PHY |
| PHY في reset دائم | `PHY not found at address X` | افحص الـ reset pin — active-low = يجب أن يكون HIGH |
| Wrong voltage level (3.3V vs 1.8V) | intermittent read errors | استخدم level shifter |
| MDO pin مطلوب بس مش موجود في DT | write fails، read شغال | أضف الـ GPIO الثالث في الـ DT |
| GPIO conflict مع driver تاني | `gpio-XX: is already claimed` | تحقق من الـ DT مفيش overlap |
| كابيسيتانس عالية على الـ MDIO line | signal distortion، bit errors | قصّر الـ PCB trace أو قلل الـ bitbang frequency |

#### 5. Device Tree Debugging

**قراءة الـ DT runtime:**
```bash
# شوف الـ compatible string للـ mdio node
cat /sys/bus/platform/devices/mdio-gpio/of_node/compatible

# شوف الـ GPIO mapping (أرقام raw phandle+index+flags)
xxd /proc/device-tree/mdio/gpios 2>/dev/null

# شوف الـ aliases
ls /proc/device-tree/aliases/ | grep mdio
cat /proc/device-tree/aliases/mdio-gpio0

# dump الـ DT node كاملاً بصيغة مقروءة
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | \
    awk '/virtual,mdio-gpio/{found=1} found{print; if(/^};$/){exit}}'
```

**DT صحيح للـ mdio-gpio:**
```dts
/ {
    aliases {
        mdio-gpio0 = &mdio0;
    };

    mdio0: mdio {
        compatible = "virtual,mdio-gpio";
        /* ترتيب الـ GPIOs: MDC, MDIO, ثم MDO (اختياري) */
        gpios = <&gpio 17 GPIO_ACTIVE_HIGH>,   /* MDC  — index 0 */
                <&gpio 18 GPIO_ACTIVE_HIGH>;   /* MDIO — index 1 */

        #address-cells = <1>;
        #size-cells = <0>;

        phy0: ethernet-phy@1 {
            reg = <1>;  /* PHY address على الـ MDIO bus */
        };
    };
};
```

**لـ Microchip SMI0 (compatible مختلف):**
```dts
mdio0: mdio {
    compatible = "microchip,mdio-smi0";
    /* بيستخدم op_c22_read=0, op_c22_write=0 تلقائياً */
    gpios = <&gpio 17 GPIO_ACTIVE_HIGH>,
            <&gpio 18 GPIO_ACTIVE_HIGH>;
    ...
};
```

**تحقق من الـ DT يطابق الـ hardware:**
```bash
# تحقق إن الـ GPIO numbers في DT تطابق الـ schematic
grep -r "mdio-gpio\|virtual,mdio" /proc/device-tree/ 2>/dev/null

# شوف PHY address في DT
find /proc/device-tree/ -name "reg" -path "*/ethernet-phy*" -exec xxd {} \;

# تحقق من الـ phy_mask لو بتستخدم platform data
# phy_mask=0 = scan all 32 addresses
# phy_mask=0xFFFFFFFE = scan address 0 فقط
```

---

### Practical Commands

#### Script شامل للـ diagnosis

```bash
#!/bin/bash
# mdio-gpio-debug.sh — run as root

echo "=== [1] Driver Loading ==="
lsmod | grep -E "mdio_gpio|mdio_bitbang" || echo "Modules not loaded — try: modprobe mdio_gpio"

echo ""
echo "=== [2] Platform Device Binding ==="
ls /sys/bus/platform/drivers/mdio-gpio/ 2>/dev/null || echo "Driver not bound to any device"

echo ""
echo "=== [3] MDIO Bus Devices ==="
ls /sys/bus/mdio_bus/devices/ 2>/dev/null || echo "No MDIO bus devices found"

echo ""
echo "=== [4] PHY Status ==="
for phy in /sys/bus/mdio_bus/devices/gpio-*/; do
    [ -d "$phy" ] || continue
    name=$(basename "$phy")
    phyid=$(cat "$phy/phy_id" 2>/dev/null || echo "N/A")
    link=$(cat "$phy/link" 2>/dev/null || echo "N/A")
    speed=$(cat "$phy/speed" 2>/dev/null || echo "N/A")
    echo "  $name: phy_id=$phyid  link=$link  speed=${speed}Mbps"
done

echo ""
echo "=== [5] GPIO State ==="
cat /sys/kernel/debug/gpio 2>/dev/null | grep -i "mdio\|mdc" || echo "No mdio/mdc GPIO labels found"

echo ""
echo "=== [6] Recent Kernel Messages ==="
dmesg | grep -E "mdio[-_]gpio|bitbang|phylib|phy.*gpio" | tail -20

echo ""
echo "=== [7] phy_mask Check ==="
echo "PHY scan — looking for responding PHYs on eth0:"
for addr in $(seq 0 7); do
    val=$(phytool read eth0/$addr/2 2>/dev/null)
    [ "$val" != "0xffff" ] && [ "$val" != "0x0000" ] && [ -n "$val" ] && \
        echo "  PHY at addr $addr: OUI_high=$val model=$(phytool read eth0/$addr/3)"
done
```

#### تفعيل الـ ftrace لتتبع MDIO transactions

```bash
# صفّر الـ trace buffer
echo 0 > /sys/kernel/debug/tracing/trace

# فعّل MDIO + GPIO events
echo 1 > /sys/kernel/debug/tracing/events/mdio/enable
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on

# trigger MDIO activity
ethtool eth0 > /dev/null 2>&1
phytool read eth0/1/1 > /dev/null 2>&1

echo 0 > /sys/kernel/debug/tracing/tracing_on

# اقرأ الـ trace
cat /sys/kernel/debug/tracing/trace

# إيقاف الـ trace
echo 0 > /sys/kernel/debug/tracing/events/mdio/enable
echo 0 > /sys/kernel/debug/tracing/events/gpio/enable
```

**تفسير الـ output:**
```
# Read ناجح:
kworker/0:1-42   [000] ....  5.123456: mdio_access: bus=gpio-0 phy_id=1 read=1 regnum=1 val=0x796d
#                                                                                         ^^^^^^
#                                       Basic Status: link up + auto-neg complete

# قيم شائعة للـ Basic Status (reg 1):
# 0x796d = link up، 100Mbps capable، auto-neg done
# 0x7949 = link down، auto-neg in progress
# 0xFFFF = لا PHY على هذا العنوان — GPIO disconnected أو PHY بلا power
# 0x0000 = PHY في reset
```

#### الـ phytool — أشمل أداة للـ MDIO debugging

```bash
# قراءة كل الـ standard registers من PHY 1
for reg in $(seq 0 15); do
    val=$(phytool read eth0/1/$reg 2>/dev/null)
    printf "Reg %2d (0x%02x): %s\n" $reg $reg "$val"
done

# تفسير أهم الـ registers:
# Reg  0: Basic Control — reset, speed, duplex, auto-neg enable
# Reg  1: Basic Status  — link status, auto-neg complete, capabilities
# Reg  2: PHY Identifier 1 (OUI bits [3:18])
# Reg  3: PHY Identifier 2 (OUI bits [19:24] + model + revision)
# Reg  4: Auto-neg Advertisement
# Reg  5: Auto-neg Link Partner Base Page Ability

# احسب الـ PHY OUI من reg 2 و3
oui_h=$(phytool read eth0/1/2)
oui_l=$(phytool read eth0/1/3)
echo "PHY OUI: $oui_h : $oui_l"
# مثال: 0x0022:0x1640 = Microchip KSZ8081
```

#### تحقق من الـ `phy_mask` و `phy_ignore_ta_mask`

```bash
# phy_mask هو bitmask — bit N=1 يعني skip PHY address N
# phy_mask=0x00000000 → scan all 32 addresses
# phy_mask=0xFFFFFFFE → scan address 0 فقط
# phy_mask=0xFFFFFFFC → scan addresses 0 و1 فقط

# لو PHY عند address 3 مش بيتكتشف:
# تحقق إن bit 3 في phy_mask = 0
# phy_mask=0xFFFFFFF7 → يتجاهل address 3 → خاطئ
# phy_mask=0x00000000 → scan الكل → صح للـ debugging

# phy_ignore_ta_mask: بعض الـ PHYs القديمة بتترك TA bit = '1' بدل '0'
# الـ kernel بيرفض الـ read لو TA خاطئ
# لو بتشوف corrupt data أو failed reads من PHY معين:
# set phy_ignore_ta_mask = BIT(addr)  مثلاً BIT(1) = 0x2 لـ PHY address 1

# هذين الحقلين بيتضبطوا في struct mdio_gpio_platform_data:
# struct mdio_gpio_platform_data pdata = {
#     .phy_mask = 0,           /* scan all */
#     .phy_ignore_ta_mask = 0, /* strict TA check */
# };
# وبيتنقلوا للـ mii_bus في mdio_gpio_bus_init():
# new_bus->phy_mask = pdata->phy_mask;
# new_bus->phy_ignore_ta_mask = pdata->phy_ignore_ta_mask;
```

#### إعادة تشغيل الـ driver للـ debugging

```bash
# لو mdio_gpio compiled كـ module
modprobe -r mdio_gpio
dmesg | tail -5    # تحقق من cleanup messages

modprobe mdio_gpio
dmesg | tail -10   # تحقق من probe ناجح

# مثال output probe ناجح:
# [   3.124] mdio-gpio mdio-gpio.0: GPIO Bitbanged MDIO bus
# [   3.156] libphy: GPIO Bitbanged MDIO: probed
# [   3.201] Generic PHY gpio-0:01: attached PHY driver (mii_bus:phy_addr=gpio-0:01, irq=POLL)
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — الـ PHY مش بيتعرف عليه

**العنوان:** MDIO-GPIO bus مش شايل الـ PHY في gateway صناعي

**السياق:**
شركة بتبني industrial Ethernet gateway على RK3562، الـ SoC مفيهوش MDIO controller جاهز متصل بالـ PHY chip اللي هي Microchip LAN8720. الفريق قرر يستخدم bitbang MDIO عن طريق GPIO pins.

**المشكلة:**
بعد boot، الـ PHY مش بيظهر في `ip link`، و dmesg بيقول:

```
mdio_gpio mdio_gpio.0: error: no phy found on bus
```

**التحليل:**
الـ driver بيقرأ `platform_data` من `struct mdio_gpio_platform_data`:

```c
struct mdio_gpio_platform_data {
    u32 phy_mask;        /* bitmask — bit N = skip PHY at address N */
    u32 phy_ignore_ta_mask; /* skip TA phase check for these PHYs */
};
```

الفريق كان بيجرب يخفي كل الـ PHYs وقت الـ debug وسبها كده:

```c
/* WRONG: masks ALL 32 PHY addresses — no PHY will be probed */
.phy_mask = 0xFFFFFFFF,
```

الـ driver بيعمل `mdiobus_scan()` وبيتخطى أي address فيها bit مضروب في `phy_mask`، فمفيش PHY بيتـ probe خالص.

**الحل:**
لازم `phy_mask` يكون `0` عشان يـ probe كل الـ addresses، أو بس يـ mask الـ addresses اللي فيها فعلاً مفيش PHY:

```c
static struct mdio_gpio_platform_data my_mdio_pdata = {
    /* PHY is at address 1 — unmask everything (probe all) */
    .phy_mask = 0,
    .phy_ignore_ta_mask = 0,
};
```

لو الـ PHY عنده مشكلة في الـ TA (turnaround) bit:

```c
static struct mdio_gpio_platform_data my_mdio_pdata = {
    .phy_mask = 0,
    /* LAN8720 at addr 1 sometimes needs TA ignored */
    .phy_ignore_ta_mask = BIT(1),
};
```

**الدرس المستفاد:**
الـ `phy_mask` بيعمل **blacklist** مش whitelist — bit مضروب = PHY متـ probe-وش. لازم تفهم الاتجاه صح من أول يوم.

---

### السيناريو 2: Android TV Box على Allwinner H616 — PHY عنده مشكلة في Turnaround

**العنوان:** `phy_ignore_ta_mask` بيخلي الـ bus يتعطل مع RTL8211F PHY

**السياق:**
منتج Android TV box على Allwinner H616 بيستخدم Realtek RTL8211F PHY. الفريق اكتشف إن الـ SoC MDIO controller بيتهنج أحياناً، قرروا يجربوا MDIO-GPIO كـ fallback.

**المشكلة:**
الـ PHY بيظهر أحياناً وأحياناً لأ، وفي الـ dmesg:

```
mdio_gpio: read timeout on PHY 0
```

**التحليل:**
الـ RTL8211F بيستخدم Clause 22 لكن بيدعم Clause 45 access عن طريق MMD. الفريق بالغلط ضرب bit 0 في `phy_ignore_ta_mask`:

```c
.phy_ignore_ta_mask = BIT(0), /* PHY is at address 0 */
```

الـ `phy_ignore_ta_mask` بيـ skip الـ turnaround phase في الـ MDIO frame — ده بيحصل مشكلة مع PHYs اللي بتتوقع الـ TA bit بشكل صحيح زي RTL8211F في بعض الـ register reads.

الـ struct تبسيطه:

```c
struct mdio_gpio_platform_data {
    u32 phy_mask;           /* skip probing these addresses */
    u32 phy_ignore_ta_mask; /* don't check TA bit for these PHYs */
};
```

الـ `phy_ignore_ta_mask` مخصص لـ PHYs قديمة أو buggy بس بتـ drive الـ bus لما المفروض تسيبه طليق — RTL8211F مش منها.

**الحل:**

```c
static struct mdio_gpio_platform_data tv_mdio_pdata = {
    .phy_mask = 0,
    /* RTL8211F is well-behaved — do NOT ignore TA */
    .phy_ignore_ta_mask = 0,
};
```

وعشان تتأكد:

```bash
# verify PHY is visible and registers readable
cat /sys/bus/mdio_bus/devices/*/phy_id
# should return 0x001cc916 for RTL8211F
```

**الدرس المستفاد:**
الـ `phy_ignore_ta_mask` مش "performance tweak" — ده workaround لـ broken PHYs بس. استخدامه مع PHY سليم بيعمل race condition على الـ MDIO bus.

---

### السيناريو 3: IoT Sensor Hub على STM32MP1 — تعارض بين PHY Addresses

**العنوان:** gateway عنده PHYs متعددة وواحد بيغلب على التاني

**السياق:**
IoT sensor hub على STM32MP157 بيجمع بيانات من sensors عن طريق Ethernet. الـ board عندها **MDIO-GPIO bus واحد** بس بيشيل **اتنين PHYs**: KSZ8081 عند address 1، وـ KSZ8081 تاني عند address 2.

**المشكلة:**
بس PHY واحد بيظهر في `ip link` رغم إن الاتنين متوصلين صح على الـ PCB.

**التحليل:**
الـ `phy_mask` field في `mdio_gpio_platform_data` بيتحكم في الـ addresses اللي بتتـ probe:

```c
struct mdio_gpio_platform_data {
    u32 phy_mask;           /* bitmask: bit N = don't probe addr N */
    u32 phy_ignore_ta_mask;
};
```

الكود الأصلي كان:

```c
static struct mdio_gpio_platform_data iot_mdio_pdata = {
    /* Intended to mask address 0 only */
    .phy_mask = ~BIT(1), /* WRONG: this masks everything EXCEPT bit 1 */
    .phy_ignore_ta_mask = 0,
};
```

المبرمج قلب المنطق — `~BIT(1)` = `0xFFFFFFFD` اللي بيـ mask كل حاجة ما عدا address 1. PHY عند address 2 اتمسك.

**الحل:**

```c
static struct mdio_gpio_platform_data iot_mdio_pdata = {
    /* Probe all addresses, let driver find both PHYs */
    .phy_mask = 0,
    .phy_ignore_ta_mask = 0,
};
```

أو لو عايز تـ mask address 0 فقط (لأنه فاضي):

```c
.phy_mask = BIT(0), /* skip address 0, probe 1..31 */
```

**الدرس المستفاد:**
الـ bit logic في `phy_mask` سهل ينعكس — دايماً اعمل sanity check بـ:

```bash
# after boot, see which PHY addresses were found
ls /sys/bus/mdio_bus/devices/
# expect: 0000:01, 0000:02 for two PHYs at addr 1 and 2
```

---

### السيناريو 4: Automotive ECU على i.MX8MP — PHY قديم محتاج TA Workaround

**العنوان:** PHY قديم في ECU بيـ drive الـ bus وقت الـ TA phase

**السياق:**
automotive ECU على i.MX8MP بيستخدم Broadcom BCM5481 PHY قديم (legacy design reuse). الـ MDIO مربوط عن طريق GPIO bitbang بسبب constraints في الـ PCB routing.

**المشكلة:**
الـ system بيـ boot لكن الـ link مش بيجي up. الـ oscilloscope بيـ show إن الـ MDIO data line بتبقى low وقت الـ TA bit — الـ PHY بيـ drive صفر بدل ما يسيب الـ line floating للـ MAC.

**التحليل:**
الـ Clause 22 MDIO read frame بيشوف كده:

```
PRE | ST | OP | PHYAD | REGAD | TA | DATA | IDLE
                                ^
                      PHY should release bus here
                      BCM5481 drives 0 instead (bug)
```

الـ MDIO-GPIO driver بيتفقد الـ TA bit بشكل default. لو الـ PHY بـ drive صفر، الـ driver بيشوف مشكلة وبيـ timeout.

الحل هو الـ `phy_ignore_ta_mask` field اللي موجود في:

```c
struct mdio_gpio_platform_data {
    u32 phy_mask;
    u32 phy_ignore_ta_mask; /* skip TA check for buggy PHYs */
};
```

**الحل:**
BCM5481 عند address 3 في الـ design ده:

```c
static struct mdio_gpio_platform_data ecu_mdio_pdata = {
    .phy_mask = 0,
    /* BCM5481 drives TA bit low — ignore TA check for addr 3 */
    .phy_ignore_ta_mask = BIT(3),
};
```

وتأكيد بـ:

```bash
# watch MDIO activity — no more timeout errors
dmesg | grep mdio
# check link state
ethtool eth0 | grep "Link detected"
```

**الدرس المستفاد:**
الـ `phy_ignore_ta_mask` موجود لأسباب حقيقية — PHYs قديمة كتير بتعمل كده. لما تشتغل على legacy hardware، اعرف الـ errata بتاعته.

---

### السيناريو 5: Custom Board Bring-up على AM62x — الـ platform_data مش بتوصل للـ Driver

**العنوان:** MDIO-GPIO driver مش شايل الـ platform_data في board bring-up

**السياق:**
فريق bring-up بيشغل AM62x-based custom board لأول مرة. الـ board عندها KSZ9031 PHY متوصل عن طريق GPIO MDIO. الفريق كتب platform device registration في الـ board file.

**المشكلة:**
الـ driver بيـ load لكن الـ PHY مش بيتـ probe، وما فيش error messages واضحة. الـ `phy_mask` المفروض تكون 0 بس الـ behavior بيبان زي ما كل الـ PHYs متمسكة.

**التحليل:**
الفريق سجل الـ platform device كده:

```c
static struct mdio_gpio_platform_data am62_mdio_pdata = {
    .phy_mask = 0,
    .phy_ignore_ta_mask = 0,
};

static struct platform_device am62_mdio_device = {
    .name = "mdio-gpio",
    .id   = 0,
    /* BUG: dev.platform_data not set! */
};
```

ونسيوا يربطوا الـ `platform_data`:

```c
/* Missing this line: */
am62_mdio_device.dev.platform_data = &am62_mdio_pdata;
```

الـ driver بيعمل `dev_get_platdata(dev)` وبيرجع `NULL`. لما الـ `platform_data` يكون NULL، الـ driver بيستخدم default values اللي ممكن تكون `phy_mask = 0xFFFFFFFF` أو بيـ fail بـ silent error.

الـ struct اللي المفروض تتمرر:

```c
struct mdio_gpio_platform_data {
    u32 phy_mask;           /* must be 0 to probe all PHYs */
    u32 phy_ignore_ta_mask;
};
```

**الحل:**

```c
static struct mdio_gpio_platform_data am62_mdio_pdata = {
    .phy_mask = 0,
    .phy_ignore_ta_mask = 0,
};

static struct platform_device am62_mdio_device = {
    .name = "mdio-gpio",
    .id   = 0,
    .dev  = {
        /* Correctly pass platform_data to driver */
        .platform_data = &am62_mdio_pdata,
    },
};
```

وللتحقق:

```bash
# after registering the platform device
ls /sys/bus/platform/devices/ | grep mdio
# then check PHY detection
ls /sys/bus/mdio_bus/devices/
```

**الدرس المستفاد:**
الـ `struct mdio_gpio_platform_data` صغيرة جداً (fieldين بس) لكن لو ما وصلتش للـ driver صح عن طريق `dev.platform_data`، الـ driver بيشتغل بشكل غير متوقع. في board bring-up، دايماً تأكد إن الـ `platform_data` pointer متضبط قبل ما تـ register الـ device.
## Phase 7: مصادر ومراجع

### مصادر LWN.net

الـ LWN.net أهم مصدر لمتابعة تطور الـ kernel — دي المقالات الأكتر صلة بـ MDIO و GPIO:

| المقالة | الأهمية |
|---------|---------|
| [MDIO bus multiplexer support](https://lwn.net/Articles/460763/) | تغطي إضافة driver للـ GPIO-controlled MDIO mux — مباشرة صلة بـ `mdio-gpio` |
| [phylib: Add support for MDIO clause 45](https://lwn.net/Articles/384726/) | توضح تطور الـ phylib و MDIO protocol أساسًا |
| [Support MDIO devices](https://lwn.net/Articles/671456/) | patch series بتتناول الـ MDIO/PHY subsystem بشكل شامل |
| [net: mdio: add ipq8064 mdio driver](https://lwn.net/Articles/813088/) | مثال حي لـ platform MDIO driver بـ platform_data |
| [net: mdio: Add netlink interface](https://lwn.net/Articles/925483/) | إضافة netlink للـ debug — بتفيد لو بتشتغل مع MDIO buses |
| [RTL8231 GPIO expander support](https://lwn.net/Articles/1042837/) | جهاز بيشتغل كـ MDIO device ومربوط بـ GPIO — نموذج عملي |

---

### توثيق الـ Kernel الرسمي

الـ `Documentation/` paths دي المرجع الأساسي جوه الـ kernel source:

```
Documentation/networking/phy.rst          ← PHY Abstraction Layer كامل
Documentation/driver-api/gpio/            ← GPIO subsystem بالكامل
Documentation/driver-api/gpio/driver.rst  ← كتابة GPIO drivers
Documentation/driver-api/gpio/board.rst   ← GPIO mappings و platform data
Documentation/firmware-guide/acpi/dsd/phy.rst  ← MDIO/PHY في ACPI
```

**روابط على kernel.org:**

- [PHY Abstraction Layer](https://www.kernel.org/doc/html/latest/networking/phy.html)
- [GPIO Driver Interface](https://docs.kernel.org/driver-api/gpio/driver.html)
- [GPIO Mappings](https://docs.kernel.org/driver-api/gpio/board.html)
- [General Purpose I/O Index](https://docs.kernel.org/driver-api/gpio/index.html)
- [MDIO bus and PHYs in ACPI](https://docs.kernel.org/firmware-guide/acpi/dsd/phy.html)

---

### الملفات المصدرية الأساسية في الـ Kernel

الملفات دي مباشرة مرتبطة بـ `mdio-gpio.h`:

```
include/linux/platform_data/mdio-gpio.h   ← الملف نفسه
drivers/net/phy/mdio-gpio.c               ← الـ driver اللي بيستخدم الـ platform_data
drivers/net/phy/mdio-bitbang.c            ← منطق الـ bitbanging الأساسي
include/linux/mdio-bitbang.h              ← واجهة الـ bitbang engine
drivers/net/phy/mdio_bus.c                ← الـ MDIO bus core
```

---

### Commits المهمة في تاريخ الـ Driver

**الـ patch الأصلي لإضافة `mdio-gpio`:**

- [PATCH phylib: add mdio-gpio bus driver (v3)](https://linux-arm-kernel.infradead.narkive.com/K0vtFa7V/patch-phylib-add-mdio-gpio-bus-driver-v3) — النقاش الأصلي على الـ mailing list لما اتضافت الـ platform_data

**الـ Kconfig entry:**

- [CONFIG_MDIO_GPIO على LKDDB](https://cateee.net/lkddb/web-lkddb/MDIO_GPIO.html) — بيوضح متى اتضاف الـ option وإيه الـ dependencies

**الـ MDIO bus mux عبر GPIO:**

- [CONFIG_MDIO_BUS_MUX_GPIO على LKDDB](https://cateee.net/lkddb/web-lkddb/MDIO_BUS_MUX_GPIO.html)

---

### نقاشات الـ Mailing List

| الرابط | الموضوع |
|--------|---------|
| [OpenFirmware GPIO MDIO bitbang driver](https://lists.ozlabs.org/pipermail/linuxppc-dev/2008-May/056942.html) | الـ patch الأول لـ OF-based mdio-gpio على linuxppc-dev |
| [mdio-gpio: Optionally ignore TA errors](https://patchwork.ozlabs.org/patch/471377/) | الـ patch اللي أضاف `phy_ignore_ta_mask` في الـ platform_data |
| [Port bitbanged MDIO code from Linux kernel](http://lists.infradead.org/pipermail/barebox/2016-January/025939.html) | مناقشة reuse الـ mdio-bitbang code في barebox — بتوضح الفصل بين الـ layers |
| [MDIO via GPIO - Toradex Community](https://community.toradex.com/t/mdio-via-gpio/10231) | مثال عملي على ربط PHY عبر GPIO في embedded board |

---

### kernelnewbies.org — تغييرات MDIO عبر الـ Kernel versions

| الإصدار | التغيير |
|---------|---------|
| [Linux 2.6.35](https://kernelnewbies.org/Linux_2_6_35-DriversArch) | إضافة clause 45 MDIO commands على مستوى الـ MDIO bus |
| [Linux 4.3](https://kernelnewbies.org/Linux_4.3) | دعم multiple MDIO buses |
| [Linux 4.13](https://kernelnewbies.org/Linux_4.13) | AVB MDIO و MII pins support |
| [Linux 4.19](https://kernelnewbies.org/Linux_4.19) | clock configuration في Broadcom iProc mdio mux |
| [Linux 6.3](https://kernelnewbies.org/Linux_6.3) | Amlogic GXL MDIO mux support |
| [Linux 6.5](https://kernelnewbies.org/Linux_6.5) | regmap-based mdio driver |
| [Linux 6.9](https://kernelnewbies.org/Linux_6.9) | تحسينات mdio-bcm-unimac driver |

---

### الكتب الموصى بيها

#### Linux Device Drivers 3rd Edition (LDD3)

- **الفصل 14** — The Linux Device Model: بيفسر `platform_device`, `platform_driver`, و `platform_data`
- **الفصل 17** — Network Drivers: أساس فهم الـ `mii_bus` وإزاي الـ PHY بيتكلم مع الـ MAC
- متاح مجانًا: [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)

- **الفصل 14** — The Block I/O Layer: مبادئ الـ bus abstraction
- **الفصل 17** — Devices and Modules: `platform_driver` وربطه بالـ hardware
- بيبني فهم قوي للـ device model اللي MDIO GPIO بيعتمد عليه

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)

- **الفصل 15** — Debugging Embedded Linux: بيتكلم على debugging الـ network interfaces والـ PHY
- **الفصل 10** — MTD Subsystem: بيوضح نموذج الـ platform_data في embedded context — نفس الـ pattern المستخدم في `mdio-gpio`

---

### مصادر تقنية إضافية

| المصدر | الرابط |
|--------|--------|
| mdio-gpio.c source (kernel 3.7 reference) | [docs.huihoo.com](https://docs.huihoo.com/doxygen/linux/kernel/3.7/mdio-gpio_8c_source.html) |
| cregit — annotated mdio-gpio.c | [cregit.linuxsources.org](https://cregit.linuxsources.org/code/4.7/drivers/net/phy/mdio-gpio.c.html) |
| MDIO MDC كـ GPIO pins — نقاش عملي | [linux-cirrus mailing list](https://linux-cirrus.freelists.narkive.com/0VG0nIkT/mdio-mdc-as-gpio-pins) |

---

### Search Terms للبحث عن معلومات أكتر

لو عايز تعمق أكتر، استخدم الـ search terms دي:

```
# بحث عام
"mdio-gpio" linux kernel driver
"mdio_gpio_platform_data" site:lore.kernel.org
"phy_mask" "phy_ignore_ta_mask" linux

# بحث في الـ mailing lists
site:lore.kernel.org mdio gpio platform_data
site:patchwork.ozlabs.org mdio-gpio

# بحث في الـ documentation
"mdio_bus" linux kernel documentation
"mii_bus" alloc phylib gpio
"mdio_bitbang" linux driver

# bitbang protocol
"MDIO bitbanging" linux embedded
"bb_info" mdio gpio kernel driver
```
## Phase 8: Writing simple module

### الهدف

**الـ** `mdio-gpio` driver بيسجّل platform device بيمثّل MDIO bus فوق GPIO. الـ struct اللي بيشغّل الكلام ده هو `mdio_gpio_platform_data` اللي عنده `phy_mask` و `phy_ignore_ta_mask`. أحسن حاجة نعملها هنا هي **kprobe** على `mdiobus_register` — الدالة اللي بتتسمى من الـ driver لما بيعمل register للـ MDIO bus، فنشوف مين بيسجّل buses وفي أي وقت.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on mdiobus_register — يتعقب كل bus MDIO بيتسجل في النظام
 * مفيد مع mdio-gpio لأن الـ driver بيستدعي mdiobus_register وقت probe
 */

#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe */
#include <linux/phy.h>         /* struct mii_bus — argument of mdiobus_register */
#include <linux/printk.h>      /* pr_info */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("mdio-gpio-watcher");
MODULE_DESCRIPTION("kprobe على mdiobus_register لمراقبة تسجيل MDIO buses");

/* ---- الـ kprobe handler ---- */

/*
 * pre_handler بيتنفذ قبل ما mdiobus_register تنفّذ أي كود.
 * الـ argument الأول للدالة (mii_bus *bus) موجود في RDI على x86-64.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * regs_get_kernel_argument(regs, 0) بيجيب أول argument
     * بطريقة portable عبر architectures مختلفة.
     */
    struct mii_bus *bus = (struct mii_bus *)regs_get_kernel_argument(regs, 0);

    if (!bus) {
        pr_info("[mdio-watch] mdiobus_register called with NULL bus!\n");
        return 0;
    }

    /*
     * bus->name و bus->id بيوصفوا الـ bus اللي بيتسجل.
     * phy_mask بيحدد أنهي PHY addresses الـ bus هيشوفها —
     * نطبعها بالـ hex عشان كل bit = PHY address.
     */
    pr_info("[mdio-watch] mdiobus_register: name='%s' id='%s' phy_mask=0x%08x parent='%s'\n",
            bus->name  ? bus->name  : "(null)",
            bus->id    ? bus->id    : "(null)",
            bus->phy_mask,
            (bus->parent && dev_name(bus->parent)) ? dev_name(bus->parent) : "(no parent)");

    return 0; /* 0 = كمّل تنفيذ الدالة الأصلية */
}

/* ---- تعريف الـ kprobe ---- */

static struct kprobe kp = {
    .symbol_name = "mdiobus_register", /* اسم الدالة في kernel symbol table */
    .pre_handler = handler_pre,
};

/* ---- init ---- */

static int __init mdio_watch_init(void)
{
    int ret;

    /*
     * register_kprobe بتحط breakpoint على mdiobus_register.
     * لو فشلت (مثلاً الدالة inline أو مش exported) بنرجع error.
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[mdio-watch] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[mdio-watch] kprobe planted on mdiobus_register @ %p\n", kp.addr);
    return 0;
}

/* ---- exit ---- */

static void __exit mdio_watch_exit(void)
{
    /*
     * unregister_kprobe ضروري في exit عشان نشيل الـ breakpoint
     * ونتجنب crash لو الـ handler اتنادى بعد ما الـ module اتفصل.
     */
    unregister_kprobe(&kp);
    pr_info("[mdio-watch] kprobe removed\n");
}

module_init(mdio_watch_init);
module_exit(mdio_watch_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | ماكروهات `module_init` / `module_exit` والـ metadata |
| `linux/kprobes.h` | `struct kprobe` و `register_kprobe` / `unregister_kprobe` |
| `linux/phy.h` | تعريف `struct mii_bus` عشان نقدر نقرأ fields زي `name` و `phy_mask` |
| `linux/printk.h` | `pr_info` / `pr_err` للـ logging |

#### الـ `pre_handler`

**الـ** `pre_handler` بيتنادى قبل ما الدالة المستهدفة (`mdiobus_register`) تشتغل، فنقدر نقرأ arguments قبل أي تعديل. بنستخدم `regs_get_kernel_argument(regs, 0)` بدل ما ناخد register مباشرةً عشان الكود يشتغل على x86 و ARM وغيرها بدون تعديل.

#### ليه `phy_mask`؟

**الـ** `phy_mask` هو نفس الـ field اللي في `mdio_gpio_platform_data` — كل bit فيه بيعطّل scanning لـ PHY address معيّن. طباعته بتكشف فوراً لو الـ driver اتهيأ صح من الـ platform data.

#### الـ `module_exit`

**الـ** `unregister_kprobe` في exit ضروري جداً — لو تركنا الـ breakpoint وفصلنا الـ module، الـ kernel هيحاول ينادي `handler_pre` اللي اتشالت من الذاكرة، وده kernel panic مضمون.

---

### Makefile

```makefile
obj-m += mdio_watch.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

---

### تشغيل وملاحظة النتيجة

```bash
# تحميل الـ module
sudo insmod mdio_watch.ko

# تحميل أي mdio-gpio device أو تشغيل test driver
# ثم نراقب الـ kernel log
sudo dmesg | grep mdio-watch
```

**مثال على output:**

```
[mdio-watch] kprobe planted on mdiobus_register @ ffffffffc0123456
[mdio-watch] mdiobus_register: name='mdio-gpio-0' id='gpio-mdio' phy_mask=0xfffffffe parent='soc:mdio'
```

**الـ** `phy_mask=0xfffffffe` معناه إن بس PHY address 0 هو المفعّل (bit 0 = 0)، وده طبيعي في embedded boards اللي عندها PHY واحد.
