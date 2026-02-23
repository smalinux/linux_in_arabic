## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ NVMEM Framework؟

تخيل إن عندك جهاز embedded — راوتر، موبايل، تلفزيون ذكي — وجوه chip صغيرة بتخزن معلومات مهمة جداً زي:
- الـ **MAC address** بتاع الـ Wi-Fi
- الـ **serial number** بتاع الجهاز
- إعدادات المصنع (factory calibration)
- مفاتيح التشفير
- نوع الـ CPU وخصائصه

الـ chip دي ممكن تكون EEPROM أو OTP (One-Time Programmable) أو eFuse أو SRAM battery-backed. كل hardware ليه طريقة مختلفة للقراءة والكتابة.

### المشكلة اللي بيحلها الـ NVMEM

**بدون** NVMEM framework، كل driver محتاج يعرف يكلم الـ hardware بتاعه مباشرة — يعني لو عندك 20 driver محتاجين يقروا MAC address، كل واحد فيهم هيكرر نفس الكود بطريقته الخاصة.

الـ **NVMEM framework** بيعمل طبقة وسيطة موحدة:

```
┌──────────────────────────────────────────┐
│           Consumer Drivers               │
│  (ethernet, wifi, thermal, cpu-freq...)  │
└──────────────┬───────────────────────────┘
               │  nvmem_cell_get() / nvmem_cell_read()
               ▼
┌──────────────────────────────────────────┐
│          NVMEM Framework Core            │
│         drivers/nvmem/core.c             │
└──────────────┬───────────────────────────┘
               │  reg_read() / reg_write() callbacks
               ▼
┌──────────────────────────────────────────┐
│         NVMEM Provider Drivers           │
│  (EEPROM, eFuse, OTP, SRAM drivers...)   │
│  qfprom.c, sunxi_sid.c, imx-ocotp.c...  │
└──────────────────────────────────────────┘
```

### دور الملف `nvmem-consumer.h`

الملف ده هو الـ **واجهة العامة للـ consumers** — يعني الـ drivers اللي *محتاجة تقرأ* بيانات من الـ NVMEM بدون ما تهتم بنوع الـ hardware.

الـ consumer مش driver EEPROM أو OTP — ده driver عايز *يستخدم* البيانات المخزنة فيهم، زي:
- Driver الـ Ethernet عايز MAC address
- Driver الـ thermal عايز calibration data
- Driver الـ Wi-Fi عايز board parameters

### مفهوم الـ Cell

الـ NVMEM بيقسم الـ storage لـ **cells** — كل cell ليها اسم وموقع (offset) وحجم (bytes) وأحياناً bit offset جوه البايت. بدل ما تقول "اقرالي من byte 0x10 لـ 0x16"، بتقول "اديني الـ cell اللي اسمها `mac-address`".

```c
/* consumer بيطلب cell باسمها */
struct nvmem_cell *cell = nvmem_cell_get(dev, "mac-address");
void *data = nvmem_cell_read(cell, &len);
```

### مستويان للوصول

الملف بيوفر **مستويين**:

#### 1. Cell-based interface (الأسهل والأشهر)
```c
/* احصل على cell بالاسم */
struct nvmem_cell *nvmem_cell_get(struct device *dev, const char *id);

/* اقرأ raw bytes */
void *nvmem_cell_read(struct nvmem_cell *cell, size_t *len);

/* اقرأ directly كـ u8/u16/u32/u64 */
int nvmem_cell_read_u32(struct device *dev, const char *cell_id, u32 *val);
```

#### 2. Direct device interface (للـ drivers اللي محتاجة وصول كامل)
```c
/* احصل على الـ device كله */
struct nvmem_device *nvmem_device_get(struct device *dev, const char *name);

/* اقرأ من offset معين */
int nvmem_device_read(struct nvmem_device *nvmem, unsigned int offset,
                      size_t bytes, void *buf);
```

### الـ `devm_` prefix

كل function موجودة في نسختين:
- `nvmem_cell_get()` — تحتاج تعمل `nvmem_cell_put()` يدوياً
- `devm_nvmem_cell_get()` — الـ kernel بيعمل cleanup تلقائياً لما الـ device يتشال (device-managed)

### الـ `nvmem_cell_lookup`

لما الـ Device Tree مش موجود (x86 أو ACPI platforms)، الـ board code ممكن يسجل جدول lookup يقول "الـ consumer ده عايز الـ cell دي من الـ provider ده":

```c
static struct nvmem_cell_lookup my_lookups[] = {
    {
        .nvmem_name = "qfprom0",      /* اسم الـ provider */
        .cell_name  = "mac-address",  /* اسم الـ cell */
        .dev_id     = "eth0",         /* الـ consumer */
        .con_id     = "mac-address",  /* connection id */
    },
};
nvmem_add_cell_lookups(my_lookups, ARRAY_SIZE(my_lookups));
```

### الـ Notifier System

الملف بيوفر نظام notifications للـ consumers عشان تعرف لما NVMEM device أو cell يتضاف أو يتشال:

```c
/* الأحداث الممكنة */
NVMEM_ADD          /* device جديد اتسجل */
NVMEM_REMOVE       /* device اتشال */
NVMEM_CELL_ADD     /* cell جديدة اتضافت */
NVMEM_CELL_REMOVE  /* cell اتشالت */
NVMEM_LAYOUT_ADD   /* layout جديد */
NVMEM_LAYOUT_REMOVE
```

### الـ `#if IS_ENABLED(CONFIG_NVMEM)`

طول الملف فيه نمط مهم — كل function ليها نسختين:
- لو NVMEM enabled في kernel config: الـ real implementation في `core.c`
- لو NVMEM disabled: `static inline` بترجع `-EOPNOTSUPP`

ده بيخلي الـ consumer drivers تشتغل على kernels مش فيها NVMEM من غير ما تكسر الـ build.

### الـ OF (Open Firmware / Device Tree) integration

الجزء التاني من الملف فيه `of_nvmem_cell_get()` و`of_nvmem_device_get()` — دول بيشتغلوا لما الـ consumer محدد في الـ Device Tree بـ property زي:

```dts
ethernet@1 {
    nvmem-cells = <&mac_address>;
    nvmem-cell-names = "mac-address";
};
```

### القصة الكاملة — مثال عملي

```
EEPROM chip على I2C bus
    → at24.c driver (provider) يسجل nvmem_device
        → فيه cells: mac-address @ offset 0, serial @ offset 6
    → ethernet driver (consumer) يطلب nvmem_cell_get(dev, "mac-address")
        → الـ framework يلاقي الـ cell
        → يطلب من الـ at24 provider يقرأ 6 bytes من offset 0
        → يرجع للـ ethernet driver الـ MAC بتاعه
```

### الملفات اللي المبرمج يعرفها

| الملف | الدور |
|-------|-------|
| `include/linux/nvmem-consumer.h` | الـ API للـ consumers (هذا الملف) |
| `include/linux/nvmem-provider.h` | الـ API للـ providers (hardware drivers) |
| `drivers/nvmem/core.c` | قلب الـ framework — التسجيل، الـ lookup، الـ read/write |
| `drivers/nvmem/layouts.c` | دعم الـ layouts (TLV وغيره) |
| `drivers/nvmem/internals.h` | structs داخلية مش للـ consumers |
| `drivers/nvmem/qfprom.c` | مثال provider: Qualcomm eFuses |
| `drivers/nvmem/sunxi_sid.c` | مثال provider: Allwinner SID |
| `drivers/nvmem/imx-ocotp.c` | مثال provider: NXP i.MX OTP |
| `drivers/nvmem/at24.c` | (في `drivers/misc/eeprom/`) EEPROM provider |
| `Documentation/devicetree/bindings/nvmem/` | DT bindings للـ NVMEM |
| `Documentation/ABI/stable/sysfs-bus-nvmem` | sysfs interface |
## Phase 2: شرح الـ Regulator Framework

### المشكلة اللي بيحلها الـ Framework

في أي SoC حديث زي Cortex-A53 أو i.MX8، في عندك عشرات الـ voltage rails — كل core, memory interface, PHY, وحتى الـ sensors عندها supply voltage خاص بيها. الـ PMIC (Power Management IC) زي AXP803 أو TPS65219 بيوفر كل الـ rails دي.

المشكلة بدون framework موحد:
- كل driver بيكلم الـ PMIC مباشرة عبر I2C/SPI — coupling كامل بين الـ consumer driver والـ hardware
- مفيش تنسيق: لو USB driver وـ LCD driver شغالين على نفس الـ rail، مين اللي بيطفيه؟ لو طفيه وحدة والتانية لسه محتاجاه؟
- مفيش **reference counting** — ممكن يتطفى supply وجهاز تاني بيشتغل عليه
- مفيش **voltage negotiation** — لو اتنين consumers على نفس الـ regulator وكل واحد محتاج voltage مختلف، مين الأولوية؟
- مفيش **suspend management** — كل driver محتاج يعرف هو يعمل إيه وقت suspend

### الحل — نهج الـ Kernel

الـ Regulator Framework بيعمل **abstraction layer** كامل بين consumer drivers والـ PMIC hardware:

1. **الـ consumer** (مثلاً USB driver) بيطلب supply باسمه فقط: `"vbus"` — مش عارف أي PMIC وأي register
2. **الـ core** بيعمل reference counting، voltage arbitration، وmode selection
3. **الـ provider** (PMIC driver) بيسجل نفسه وبيوفر `regulator_ops` — callbacks لكل operation

الـ framework بيملك الـ **policy**: إمتى تشغل، إمتى تطفي، أي voltage تختار لو في conflict.
الـ PMIC driver بيملك الـ **mechanism**: إزاي تكتب في الـ register الصح.

---

### الـ Big Picture Architecture

```
 ┌─────────────────────────────────────────────────────────────────┐
 │                      Consumer Drivers                           │
 │   USB Driver    LCD Driver    CPU cpufreq    Audio Codec        │
 │   "vbus-supply" "vdd-supply"  "cpu0-supply"  "avdd-supply"      │
 └────────┬───────────┬──────────────┬──────────────┬─────────────┘
          │           │              │              │
          │  regulator_get() / regulator_enable()   │
          ▼           ▼              ▼              ▼
 ┌─────────────────────────────────────────────────────────────────┐
 │               Regulator Core (drivers/regulator/core.c)         │
 │                                                                 │
 │  ┌──────────────┐  ┌────────────────┐  ┌─────────────────────┐  │
 │  │ struct       │  │  Voltage       │  │  Reference          │  │
 │  │ regulator    │  │  Arbitration   │  │  Counting           │  │
 │  │ (per-consumer│  │  (min/max      │  │  (enable_count)     │  │
 │  │  handle)     │  │   intersection)│  │                     │  │
 │  └──────┬───────┘  └────────────────┘  └─────────────────────┘  │
 │         │                                                        │
 │         └──────► struct regulator_dev (rdev) ◄───────────────── │
 │                  [owns: ops, constraints, list of consumers]     │
 └───────────────────────────┬─────────────────────────────────────┘
                             │
                    regulator_ops callbacks
                             │
          ┌──────────────────┼───────────────────────┐
          ▼                  ▼                       ▼
 ┌─────────────────┐ ┌──────────────────┐ ┌──────────────────────┐
 │  AXP803 Driver  │ │  TPS65219 Driver │ │  Fixed Voltage Reg   │
 │  (I2C PMIC)     │ │  (I2C PMIC)      │ │  (no ops needed)     │
 └────────┬────────┘ └────────┬─────────┘ └──────────────────────┘
          │                   │
          ▼                   ▼
 ┌─────────────────┐ ┌──────────────────┐
 │  Physical PMIC  │ │  Physical PMIC   │
 │  on the board   │ │  on the board    │
 └─────────────────┘ └──────────────────┘
```

---

### التشابه مع الواقع — Water Distribution Analogy

تخيل شبكة مياه في مبنى سكني:

| الـ Water System | الـ Regulator Framework |
|---|---|
| محطة ضخ المياه | الـ PMIC (المصدر الحقيقي للطاقة) |
| شبكة المواسير | الـ voltage rails على الـ PCB |
| صنبور المياه لكل شقة | الـ `struct regulator` — الـ handle اللي كل consumer عنده |
| عداد المياه الرئيسي | الـ `struct regulator_dev` — يتحكم في الـ rail كله |
| شركة المياه (السياسة) | الـ Regulator Core — بيقرر مين يشغل ومين يطفي |
| السباك (التنفيذ) | الـ PMIC driver — بيكتب في الـ hardware registers |
| مفيش فلوس = ماء مقطوع | enable_count = 0 → regulator disabled |
| كل ساكن بيفتح الصنبور | كل consumer بيعمل `regulator_enable()` |
| الماء بيتقطع لما آخر ساكن يسافر | `disable` بيحصل لما آخر consumer يعمل `regulator_disable()` |
| ضغط المياه | voltage level |
| كل شقة عندها ضغط مختلف | voltage constraints per consumer |

**التعمق في التشابه:**
- لو شقة بتحتاج ضغط 3 بار وتانية 5 بار على نفس الخط → شركة المياه بتختار 5 بار (أعلى minimum يرضي الكل) — ده بالظبط الـ voltage arbitration
- لو كل الشقق خرجت وفتحت المصدر الاحتياطي — الشركة بتطفي المضخة الرئيسية تلقائياً — ده الـ reference counting
- ضغط عالي في وقت الذروة، ضغط تاني وقت الفجر — ده الـ DRMS (Dynamic Regulator Mode Switching)

---

### الـ Core Abstraction — الفكرة المحورية

الـ central idea هي: **فصل الـ "من يحتاج الطاقة" عن "كيف تُعطى الطاقة"**

الـ abstraction المحورية هي `struct regulator` — مش هي الـ hardware، هي **علاقة** بين consumer معين وـ supply معين:

```c
/* consumer.h — ده الـ opaque handle اللي كل driver بيشتغل بيه */
struct regulator;  /* forward declaration فقط — التفاصيل في core.c */

/* كل consumer بياخد handle خاص بيه — حتى لو على نفس الـ rail */
struct regulator *vdd = regulator_get(dev, "vdd");
struct regulator *vbus = regulator_get(dev, "vbus");
```

الـ `struct regulator` مش هي الـ hardware — هي الـ **per-consumer view** للـ hardware. ممكن يكون 10 consumers على نفس الـ rail وكل واحد عنده `struct regulator` مختلف، بس كلهم بيشيروا لنفس `struct regulator_dev`.

---

### الـ Struct Relationships

```
                    ┌──────────────────────────────┐
                    │     struct regulator_dev      │
                    │  (rdev — الـ hardware object)  │
                    │                              │
                    │  desc → regulator_desc       │
                    │  ops  → regulator_ops        │
                    │  constraints                 │
                    │  enable_count                │
                    │  consumer_list ──────────────┼──┐
                    └──────────────────────────────┘  │
                                                       │
              ┌──────────────────┬───────────────────┐ │
              ▼                  ▼                   ▼ │
   ┌──────────────────┐ ┌──────────────────┐ ┌────────▼─────────┐
   │ struct regulator │ │ struct regulator │ │ struct regulator │
   │  (USB driver)    │ │  (LCD driver)    │ │  (Audio driver)  │
   │  min_uV=3000000  │ │  min_uV=3100000  │ │  min_uV=2800000  │
   │  enabled=true    │ │  enabled=true    │ │  enabled=false   │
   └──────────────────┘ └──────────────────┘ └──────────────────┘
         │                      │                     │
         └──────────────────────┴─────────────────────┘
                           rdev->enable_count = 2
                    (Audio disabled → مش محسوب)
```

---

### الـ Core Consumer API — تفصيل كل مجموعة

#### 1. الـ Get/Put API

```c
/* الطريقة الأساسية — بتربط consumer بـ supply من الـ DT */
struct regulator *regulator_get(struct device *dev, const char *id);
/*
 * dev   = الـ device اللي بتطلب الـ supply
 * id    = الاسم اللي في الـ DT: "vdd", "vbus", "avdd" ...
 * return: handle أو ERR_PTR في حالة error
 */

/* نفسها بس بتتحرر تلقائياً وقت device removal */
struct regulator *devm_regulator_get(struct device *dev, const char *id);

/* exclusive = مش هينفع consumer تاني يأخد نفس الـ supply */
struct regulator *regulator_get_exclusive(struct device *dev, const char *id);
/*
 * مهم لـ: voltage-controlled power domains حيث consumer واحد فقط
 * يحق له يغير الـ voltage
 */

/* optional = لو مش موجود في الـ DT مش هيرجع error */
struct regulator *regulator_get_optional(struct device *dev, const char *id);
/*
 * مثال: driver بيشتغل على بوردين — بورد فيها external regulator
 * وبورد تانية فيها fixed voltage — بيستخدم optional عشان يكمل
 * حتى لو مفيش regulator معرف
 */
```

**ازاي الـ DT بيربط الـ supply بالـ consumer:**

```dts
/* في device tree */
my_sensor: sensor@48 {
    compatible = "my,sensor";
    reg = <0x48>;
    vdd-supply = <&ldo3>;    /* "vdd" → ldo3 node */
    vbus-supply = <&buck2>;  /* "vbus" → buck2 node */
};

/* الـ ldo3 و buck2 معرفين في PMIC node */
pmic: axp803@34 {
    regulators {
        ldo3: LDO3 {
            regulator-min-microvolt = <700000>;
            regulator-max-microvolt = <3300000>;
        };
        buck2: DCDC2 {
            regulator-min-microvolt = <500000>;
            regulator-max-microvolt = <1300000>;
        };
    };
};
```

#### 2. الـ Enable/Disable API

```c
int regulator_enable(struct regulator *regulator);
/*
 * بيزود enable_count بمقدار 1
 * لو enable_count بقى 1 (أول consumer) → بيكلم ops->enable()
 * لو enable_count > 1 → بيرجع 0 فوراً (الـ hardware شغال خلاص)
 */

int regulator_disable(struct regulator *regulator);
/*
 * بيقلل enable_count بمقدار 1
 * لو enable_count بقى 0 → بيكلم ops->disable()
 * لو enable_count > 0 → بيرجع 0 فوراً (consumers تانيين لسه محتاجينه)
 */

int regulator_force_disable(struct regulator *regulator);
/*
 * بيطفي بالقوة بصرف النظر عن enable_count
 * خطير! بس مفيدة في error recovery paths
 */

int regulator_disable_deferred(struct regulator *regulator, int ms);
/*
 * بيعمل disable بعد ms milliseconds
 * مفيدة جداً لـ: WiFi/BT modules اللي محتاجة time للـ shutdown sequence
 * بيمنع voltage glitches من restart سريع
 */
```

#### 3. الـ Voltage Control API

```c
int regulator_set_voltage(struct regulator *regulator, int min_uV, int max_uV);
/*
 * Consumer بيقول: "أنا محتاج voltage بين min_uV و max_uV"
 * الـ core بيعمل intersection مع باقي consumers على نفس الـ rail
 * لو مفيش intersection → -EINVAL
 *
 * مثال: CPU بيعمل DVFS:
 *   regulator_set_voltage(cpu_vdd, 900000, 1100000);  // 0.9V-1.1V
 */

int regulator_get_voltage(struct regulator *regulator);
/*
 * بيرجع الـ actual voltage المضبوط على الـ rail دلوقتي (microvolts)
 */

int regulator_set_voltage_triplet(struct regulator *reg,
                                  int min_uV, int target_uV, int max_uV);
/*
 * بيحاول target_uV الأول، لو فشل بيجرب min_uV → max_uV
 * مفيدة لـ DVFS حيث target مثالي بس ليه fallback range
 */

static inline int regulator_set_voltage_tol(struct regulator *regulator,
                                             int new_uV, int tol_uV)
{
    /* helper: بيحول tolerance% لـ min/max range */
    if (regulator_set_voltage(regulator, new_uV, new_uV + tol_uV) == 0)
        return 0;
    return regulator_set_voltage(regulator, new_uV - tol_uV, new_uV + tol_uV);
}
```

**الـ Voltage Arbitration بالتفصيل:**

```
Consumer A: min=1000mV, max=1200mV  ──┐
Consumer B: min=1100mV, max=1300mV  ──┼─→ Core: intersection = [1100, 1200]
Consumer C: min=900mV,  max=1150mV  ──┘         → يضبط على 1100mV

لو Consumer D جه بـ min=1250mV, max=1400mV:
    intersection مع [1100, 1200] = فراغ → -EINVAL
```

#### 4. الـ Current Control API

```c
int regulator_set_current_limit(struct regulator *reg, int min_uA, int max_uA);
/*
 * للـ current-limited regulators (LDOs مع overcurrent protection)
 * مثال: USB VBUS regulator بـ 500mA limit:
 *   regulator_set_current_limit(vbus_reg, 0, 500000);
 */

int regulator_get_unclaimed_power_budget(struct regulator *regulator);
int regulator_request_power_budget(struct regulator *reg, unsigned int pw_req);
void regulator_free_power_budget(struct regulator *reg, unsigned int pw);
/*
 * Power Budget Management — جديدة نسبياً
 * Consumer بيعلن كام mW هو عايز، الـ core بيتحقق إن في capacity كافية
 * مفيدة لـ: USB PD adapters، thermal-limited systems
 */
```

#### 5. الـ Operating Mode API

```c
/*
 * REGULATOR_MODE_FAST    = 0x1 — يقدر يتعامل مع fast load transients
 *                                مثال: CPU راجع من idle لـ full load
 *
 * REGULATOR_MODE_NORMAL  = 0x2 — الوضع الاعتيادي
 *
 * REGULATOR_MODE_IDLE    = 0x4 — efficient mode لـ light loads
 *                                أكتر noise لكن أقل power
 *
 * REGULATOR_MODE_STANDBY = 0x8 — أقل كفاءة بس أقل طاقة
 *                                مفيد لـ deep sleep
 */
int regulator_set_mode(struct regulator *regulator, unsigned int mode);
unsigned int regulator_get_mode(struct regulator *regulator);

int regulator_set_load(struct regulator *regulator, int load_uA);
/*
 * Consumer بيخبر الـ core بـ expected load
 * الـ core بيستخدمها مع DRMS لاختيار الـ mode الأمثل تلقائياً
 * مثال: قبل DVFS frequency increase:
 *   regulator_set_load(cpu_vdd, 500000);  // 500mA
 */
```

#### 6. الـ Bulk API

```c
/*
 * لو driver محتاج عدة supplies — بيدي convenience API:
 */
static struct regulator_bulk_data supplies[] = {
    { .supply = "vdd",  .init_load_uA = 100000 },
    { .supply = "vbus", .init_load_uA = 0       },
    { .supply = "avdd", .init_load_uA = 50000   },
};

/* بياخد كلهم دفعة واحدة */
devm_regulator_bulk_get(dev, ARRAY_SIZE(supplies), supplies);

/* بيشغل كلهم بالترتيب */
regulator_bulk_enable(ARRAY_SIZE(supplies), supplies);

/* بيطفي كلهم */
regulator_bulk_disable(ARRAY_SIZE(supplies), supplies);
```

#### 7. الـ Notifier API

```c
/*
 * الـ Notifier Subsystem — لازم تعرف عنه:
 * الـ kernel بيوفر notifier chains كـ pub/sub mechanism
 * بيخلي code يسجل callback يتنادى لما event بيحصل
 */
int regulator_register_notifier(struct regulator *regulator,
                                struct notifier_block *nb);

/*
 * Events مهمة:
 * REGULATOR_EVENT_UNDER_VOLTAGE     — voltage انخفض عن الحد
 * REGULATOR_EVENT_OVER_CURRENT      — current عالي جداً
 * REGULATOR_EVENT_OVER_TEMP         — hardware over temperature
 * REGULATOR_EVENT_VOLTAGE_CHANGE    — voltage اتغير (بعد DVFS مثلاً)
 * REGULATOR_EVENT_PRE_VOLTAGE_CHANGE — قبل ما voltage يتغير (لـ preparation)
 * REGULATOR_EVENT_FAIL              — regulator fail كامل
 *
 * مثال: audio codec بيعمل mute قبل voltage change:
 */
static int audio_regulator_notifier(struct notifier_block *nb,
                                    unsigned long event, void *data)
{
    struct audio_priv *priv = container_of(nb, struct audio_priv, nb);

    if (event & REGULATOR_EVENT_PRE_VOLTAGE_CHANGE) {
        audio_mute(priv);  /* امنع pop noise */
    } else if (event & REGULATOR_EVENT_VOLTAGE_CHANGE) {
        audio_unmute(priv);
    } else if (event & REGULATOR_EVENT_FAIL) {
        audio_emergency_shutdown(priv);
    }
    return NOTIFY_OK;
}
```

#### 8. الـ Bypass Mode

```c
int regulator_allow_bypass(struct regulator *regulator, bool allow);
/*
 * Bypass mode = الـ input voltage بيعدي مباشرة للـ output بدون regulation
 * أعلى efficiency لكن مفيش regulation
 * مفيدة لـ: LDOs في low-dropout situations حيث input ≈ output
 *
 * الـ core بيسمح بـ bypass بس لما كل consumers يوافقوا
 */
```

---

### الـ Suspend Management

```c
int regulator_set_suspend_voltage(struct regulator *reg,
                                  int min_uV, int max_uV,
                                  suspend_state_t state);
/*
 * بيضبط voltage مختلف وقت suspend
 * suspend_state_t: PM_SUSPEND_MEM, PM_SUSPEND_STANDBY, PM_SUSPEND_FREEZE
 *
 * مثال: CPU voltage بيتخفض في suspend لتوفير طاقة:
 *   regulator_set_suspend_voltage(cpu_vdd, 800000, 900000, PM_SUSPEND_MEM);
 */
```

---

### ما بيملكه الـ Framework vs ما بيفوضه للـ Driver

| الـ Framework يملك | الـ PMIC Driver يملك |
|---|---|
| Reference counting (`enable_count`) | تسلسل تشغيل الـ hardware (`ops->enable`) |
| Voltage arbitration (intersection of ranges) | قراءة وكتابة الـ voltage registers |
| Constraint enforcement (min/max limits) | تحديد الـ voltage steps المتاحة (`list_voltage`) |
| DRMS — اختيار الـ mode الأمثل | تطبيق الـ mode على الـ hardware |
| Dependency tree (supply chains) | الـ ramp delay الحقيقي بعد voltage change |
| sysfs/debugfs exposure | Interrupt handling للـ hardware faults |
| Notifier chain dispatch | Regmap transactions على I2C/SPI |
| Suspend/resume sequencing | Suspend-specific hardware configuration |

---

### الـ Supply Chain — Regulator Cascading

```
Battery (Li-Ion 3.7-4.2V)
    │
    ▼
BUCK1 (3.3V main rail) ← regulator_dev with parent=NULL
    │
    ├──► LDO1 (1.8V) ← regulator_dev with supply_name="buck1-supply"
    │        │
    │        └──► SoC I/O domain (1.8V)
    │
    ├──► LDO2 (1.2V) ← regulator_dev with supply_name="buck1-supply"
    │        │
    │        └──► DDR PHY (1.2V)
    │
    └──► LDO3 (3.0V)
             │
             └──► SD Card VDD (3.0V)
```

الـ framework بيفهم الـ tree دي — لو BUCK1 اتطلب يتطفى وفي LDO يعتمد عليه مش محدش طلب يطفيه، الـ framework بيرفض.

---

### مثال عملي كامل — WiFi Driver على ARM SoC

```c
struct wifi_priv {
    struct regulator *vdd;
    struct regulator *vbus;
    struct notifier_block nb;
};

static int wifi_probe(struct platform_device *pdev)
{
    struct wifi_priv *priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);

    /* احصل على الـ supplies من الـ DT */
    priv->vdd  = devm_regulator_get(&pdev->dev, "vdd");   /* 3.3V */
    priv->vbus = devm_regulator_get(&pdev->dev, "vbus");  /* 1.8V */

    if (IS_ERR(priv->vdd) || IS_ERR(priv->vbus))
        return PTR_ERR(priv->vdd);

    /* اخبر الـ core بالـ load المتوقع (لـ DRMS) */
    regulator_set_load(priv->vdd,  150000);  /* 150mA max */
    regulator_set_load(priv->vbus, 50000);   /* 50mA max */

    /* سجل notifier علشان تتعرف لو voltage اتغير */
    priv->nb.notifier_call = wifi_voltage_notifier;
    regulator_register_notifier(priv->vdd, &priv->nb);

    /* شغل الـ supplies */
    regulator_enable(priv->vdd);
    regulator_enable(priv->vbus);

    /* دلوقتي الـ WiFi hardware powered وجاهز للـ init */
    return wifi_hw_init(priv);
}

static int wifi_remove(struct platform_device *pdev)
{
    struct wifi_priv *priv = platform_get_drvdata(pdev);

    regulator_unregister_notifier(priv->vdd, &priv->nb);
    regulator_disable(priv->vbus);
    regulator_disable(priv->vdd);
    /* devm بيعمل regulator_put تلقائياً */
    return 0;
}
```

```dts
/* في الـ DTS */
wifi: wifi@1c100000 {
    compatible = "my,wifi-chip";
    reg = <0x1c100000 0x1000>;
    vdd-supply  = <&buck2>;   /* "vdd"  → buck2 */
    vbus-supply = <&ldo4>;    /* "vbus" → ldo4  */
};
```
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### Flags وـ Defines — Cheatsheet

#### REGULATOR_MODE_* — أوضاع التشغيل

| Define | القيمة | متى تستخدمه |
|--------|--------|-------------|
| `REGULATOR_MODE_INVALID` | `0x0` | وضع غير صالح / غير مُعيَّن |
| `REGULATOR_MODE_FAST` | `0x1` | تغييرات سريعة في الحِمل — CPU frequency scaling |
| `REGULATOR_MODE_NORMAL` | `0x2` | الوضع الافتراضي لمعظم الـ drivers |
| `REGULATOR_MODE_IDLE` | `0x4` | حِمل خفيف — أكثر كفاءةً، أكثر ضوضاء |
| `REGULATOR_MODE_STANDBY` | `0x8` | حِمل خفيف جداً — sleep state |

القيم قابلة لـ OR لتكوين mask من الأوضاع المدعومة.

---

#### REGULATOR_ERROR_* — أخطاء الـ Hardware

| Define | الـ Bit | المعنى |
|--------|---------|--------|
| `REGULATOR_ERROR_UNDER_VOLTAGE` | `BIT(1)` | الجهد أقل من المطلوب |
| `REGULATOR_ERROR_OVER_CURRENT` | `BIT(2)` | التيار تجاوز الحد |
| `REGULATOR_ERROR_REGULATION_OUT` | `BIT(3)` | الجهد خرج من نطاق التنظيم |
| `REGULATOR_ERROR_FAIL` | `BIT(4)` | فشل كامل في الـ output |
| `REGULATOR_ERROR_OVER_TEMP` | `BIT(5)` | الحرارة زادت عن الحد |
| `REGULATOR_ERROR_UNDER_VOLTAGE_WARN` | `BIT(6)` | تحذير — الجهد منخفض لكن لم يفشل بعد |
| `REGULATOR_ERROR_OVER_CURRENT_WARN` | `BIT(7)` | تحذير — التيار مرتفع |
| `REGULATOR_ERROR_OVER_VOLTAGE_WARN` | `BIT(8)` | تحذير — الجهد مرتفع |
| `REGULATOR_ERROR_OVER_TEMP_WARN` | `BIT(9)` | تحذير — حرارة مرتفعة |

---

#### REGULATOR_EVENT_* — أحداث الـ Notifier (من uapi/regulator/regulator.h)

| Define | القيمة | الحدث |
|--------|--------|-------|
| `REGULATOR_EVENT_UNDER_VOLTAGE` | `0x01` | جهد منخفض |
| `REGULATOR_EVENT_OVER_CURRENT` | `0x02` | تيار زائد |
| `REGULATOR_EVENT_REGULATION_OUT` | `0x04` | خروج عن نطاق التنظيم |
| `REGULATOR_EVENT_FAIL` | `0x08` | فشل كامل |
| `REGULATOR_EVENT_OVER_TEMP` | `0x10` | حرارة زائدة |
| `REGULATOR_EVENT_FORCE_DISABLE` | `0x20` | إيقاف قسري |
| `REGULATOR_EVENT_VOLTAGE_CHANGE` | `0x40` | تغيّر الجهد — data = old_uV |
| `REGULATOR_EVENT_DISABLE` | `0x80` | تم الإيقاف |
| `REGULATOR_EVENT_PRE_VOLTAGE_CHANGE` | `0x100` | قبيل تغيير الجهد — data = `pre_voltage_change_data*` |
| `REGULATOR_EVENT_ABORT_VOLTAGE_CHANGE` | `0x200` | فشل تغيير الجهد |
| `REGULATOR_EVENT_PRE_DISABLE` | `0x400` | قبيل الإيقاف |
| `REGULATOR_EVENT_ABORT_DISABLE` | `0x800` | فشل الإيقاف |
| `REGULATOR_EVENT_ENABLE` | `0x1000` | تم التشغيل |
| `REGULATOR_EVENT_WARN_MASK` | `0x1E000` | mask لأحداث التحذير الأربعة |

---

#### Config Options المهمة

| Option | الأثر |
|--------|-------|
| `CONFIG_REGULATOR` | تفعيل الـ framework كله — بدونه كل الـ API تصبح stubs |
| `CONFIG_OF` | تفعيل الـ DT-based regulator lookup |
| `IS_ENABLED(CONFIG_OF) && IS_ENABLED(CONFIG_REGULATOR)` | تفعيل `of_regulator_get*` |

---

### الـ Structs المهمة

#### 1. `struct regulator` — الـ Consumer Handle

**الغرض:** handle غير شفاف يمثّل اتصال consumer واحد بـ regulator واحد. الـ consumer يحتفظ بمؤشر لهذا الـ struct ويستخدمه في كل العمليات.

**ملاحظة:** الـ struct نفسه مُعرَّف في `drivers/regulator/internal.h` — مخفي عن الـ consumers عمداً (information hiding). الـ consumer يرى فقط pointer معتم.

**ما يحتويه داخلياً (من implementation):**
```c
struct regulator {
    struct device              *dev;      /* consumer device */
    struct list_head            list;     /* قائمة consumers على نفس rdev */
    struct regulator_dev       *rdev;     /* الـ regulator الفعلي */
    struct dentry              *debugfs;  /* debugfs entry */
    char                       *supply_name;
    int                         uA_load;  /* الحِمل الحالي المطلوب */
    int                         enable_count; /* عدد enable calls */
    /* ... */
};
```

---

#### 2. `struct pre_voltage_change_data` — بيانات حدث تغيير الجهد

**الغرض:** يُرسَل مع حدث `REGULATOR_EVENT_PRE_VOLTAGE_CHANGE` ليعطي الـ consumers فرصة الاعتراض أو التحضير قبل تغيير الجهد.

```c
struct pre_voltage_change_data {
    unsigned long old_uV;   /* الجهد الحالي قبل التغيير (ميكروفولت) */
    unsigned long min_uV;   /* أدنى جهد مطلوب في التغيير القادم */
    unsigned long max_uV;   /* أقصى جهد مطلوب في التغيير القادم */
};
```

**مثال استخدام:** driver للـ CPU يسمع الحدث — إذا `min_uV` أقل من الحد الأدنى لتشغيله، يُوقف تشغيله قبل أن ينخفض الجهد.

---

#### 3. `struct regulator_bulk_data` — بيانات Bulk Operations

**الغرض:** تسهّل إدارة أجهزة تحتاج أكثر من مصدر طاقة واحد في وقت واحد — بدلاً من استدعاء `regulator_get` و`regulator_enable` لكل مصدر على حدة.

```c
struct regulator_bulk_data {
    const char      *supply;        /* اسم المصدر — يحدده الـ user */
    struct regulator *consumer;     /* handle يملؤه الـ bulk API تلقائياً */
    int              init_load_uA;  /* حِمل مبدئي يُطبَّق عند get */
    /* private: */
    int              ret;           /* نتيجة العملية على هذا المصدر تحديداً */
};
```

**مثال واقعي — codec صوتي يحتاج VDD_CORE و VDD_IO:**
```c
static struct regulator_bulk_data wm8958_supplies[] = {
    { .supply = "AVDD",  .init_load_uA = 5000 },
    { .supply = "DBVDD", .init_load_uA = 1000 },
    { .supply = "DCVDD", .init_load_uA = 1000 },
};

/* في probe: */
ret = devm_regulator_bulk_get(&pdev->dev,
                              ARRAY_SIZE(wm8958_supplies),
                              wm8958_supplies);

ret = regulator_bulk_enable(ARRAY_SIZE(wm8958_supplies),
                            wm8958_supplies);
```

---

### علاقات الـ Structs — ASCII Diagram

```
Consumer Driver (e.g., WiFi chip driver)
    │
    │  regulator_get(dev, "vdd-supply")
    ▼
┌─────────────────────────────────────────┐
│         struct regulator                │  ← consumer handle (opaque)
│  ┌────────────────────────────────────┐ │
│  │  supply_name: "vdd-supply"        │ │
│  │  uA_load: 50000                   │ │
│  │  enable_count: 1                  │ │
│  │  list ──────────────────────────┐ │ │
│  └────────────────────────────────┼─┘ │
│   dev ──► struct device (consumer) │   │
│   rdev ─────────────────────────┐  │   │
└─────────────────────────────────┼──┼───┘
                                  │  │
    ┌─────────────────────────────┘  │
    ▼                                │
┌─────────────────────────────────────────────┐
│         struct regulator_dev                │  ← core/provider side
│  ┌─────────────────────────────────────┐    │
│  │  desc ──► struct regulator_desc    │    │
│  │  constraints ──► regulation_constraints│ │
│  │  consumer_list ◄────────────────────┼──┘ │  (list of all consumers)
│  │  mutex / ww_mutex                  │    │
│  │  ops ──► struct regulator_ops      │    │
│  └─────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
              │
              │ supply_rdev
              ▼
┌─────────────────────────┐
│  struct regulator_dev   │  ← parent regulator (e.g., PMIC's LDO1)
│  (مصدر الطاقة الأعلى)    │
└─────────────────────────┘
```

---

### دورة الحياة — Lifecycle Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                    Consumer Driver Lifecycle                     │
└──────────────────────────────────────────────────────────────────┘

1. PROBE
   ─────
   devm_regulator_get(dev, "vdd")
     │
     ├─► core يبحث في DT عن "vdd-supply" property
     ├─► core يجد الـ regulator_dev المقابل
     ├─► core يخصص struct regulator جديد
     ├─► يضيفه لـ consumer_list في regulator_dev
     └─► يُرجع pointer ← يُخزَّن في driver private data

2. CONFIGURE (اختياري)
   ──────────────────
   regulator_set_load(reg, 50000)   /* أخبر الـ core بالحِمل المتوقع */
   regulator_set_voltage(reg, 1800000, 1800000)  /* اطلب جهداً محدداً */

3. ENABLE
   ──────
   regulator_enable(reg)
     │
     ├─► core يزيد enable_count
     ├─► إذا أول enable → يستدعي ops->enable(rdev)
     └─► HW يبدأ توليد الجهد

4. USE
   ───
   (جهاز يعمل على هذا الجهد)
   يمكن استدعاء regulator_set_voltage() خلال التشغيل (DVFS)

5. DISABLE
   ───────
   regulator_disable(reg)
     │
     ├─► core يُنقص enable_count
     └─► إذا وصل لـ 0 → ops->disable(rdev) → HW يُوقف الجهد

6. REMOVE
   ──────
   (devm تُنفّذ هذا تلقائياً عند remove)
   devm_regulator_put(reg)
     │
     ├─► يُزيل struct regulator من consumer_list
     └─► يُحرر الذاكرة
```

---

### Call Flow Diagrams

#### regulator_get() — الحصول على مصدر طاقة

```
consumer driver
  regulator_get(dev, "vdd")
    │
    └─► _regulator_get(dev, id, get_type=NORMAL_GET)
          │
          ├─► regulator_dev_lookup(dev, id)
          │     │
          │     ├─► بحث في supply aliases
          │     ├─► of_regulator_match() ← DT lookup
          │     └─► returns rdev or ERR_PTR(-EPROBE_DEFER)
          │
          ├─► create_regulator(rdev, dev, id)
          │     │
          │     ├─► kmalloc(struct regulator)
          │     ├─► regulator->rdev = rdev
          │     ├─► list_add(&regulator->list, &rdev->consumer_list)
          │     └─► debugfs_create_*()  ← لـ sysfs/debugfs
          │
          └─► returns struct regulator *
```

---

#### regulator_enable() — تشغيل مصدر الطاقة

```
consumer driver
  regulator_enable(regulator)
    │
    └─► _regulator_enable(rdev)
          │
          ├─► ww_mutex_lock(&rdev->mutex)  ← قفل
          │
          ├─► regulator->enable_count++
          │
          ├─► if (rdev->use_count == 0)    ← أول consumer يطلب enable
          │     │
          │     ├─► if (rdev->supply)      ← الـ regulator مُغذَّى من آخر؟
          │     │     └─► regulator_enable(rdev->supply)  ← recursive!
          │     │
          │     ├─► rdev->desc->ops->enable(rdev)
          │     │     └─► driver يكتب لـ hardware register
          │     │
          │     └─► rdev->use_count++
          │
          └─► ww_mutex_unlock(&rdev->mutex)
```

---

#### regulator_set_voltage() — تغيير الجهد

```
consumer driver
  regulator_set_voltage(reg, min_uV=1800000, max_uV=1800000)
    │
    └─► _regulator_set_voltage_unlocked(rdev, min_uV, max_uV)
          │
          ├─► regulator_check_voltage(rdev, &min_uV, &max_uV)
          │     └─► تحقق من constraints.min_uV ≤ min_uV ≤ max_uV ≤ constraints.max_uV
          │
          ├─► _notifier_call_chain(rdev,
          │       REGULATOR_EVENT_PRE_VOLTAGE_CHANGE,
          │       &pre_change_data)
          │     └─► كل consumers مسجّلين يستقبلون الحدث
          │
          ├─► ops->set_voltage(rdev, min_uV, max_uV, &selector)
          │     └─► driver يكتب الـ selector لـ HW register
          │
          ├─► udelay(rdev->desc->ramp_delay)  ← انتظر استقرار الجهد
          │
          └─► _notifier_call_chain(rdev, REGULATOR_EVENT_VOLTAGE_CHANGE, ...)
```

---

#### Notifier Registration وـ Event Flow

```
consumer driver
  regulator_register_notifier(reg, &nb)
    │
    └─► blocking_notifier_chain_register(&rdev->notifier, nb)
          └─► يُضاف nb لقائمة الـ notifiers على هذا rdev

(لاحقاً عند حدوث خطأ في HW)

IRQ handler (regulator driver)
  regulator_notifier_call_chain(rdev, REGULATOR_EVENT_OVER_CURRENT, NULL)
    │
    └─► blocking_notifier_call_chain(&rdev->notifier, event, data)
          │
          └─► كل nb->notifier_call(nb, event, data) يُستدعى
                └─► consumer driver يُوقف جهازه أو يُقلل الحِمل
```

---

### استراتيجية الـ Locking

#### القفل الرئيسي: `ww_mutex` (Wound/Wait Mutex)

**الـ `ww_mutex`** مختلف عن `mutex` عادي — مصمَّم خصيصاً لتجنب deadlock عند الحاجة للحصول على أقفال متعددة في وقت واحد (مثلاً: تعديل voltage chain تمر بأكثر من رمز regulator).

```
┌─────────────────────────────────────────────────────────┐
│  struct regulator_dev                                   │
│  ┌────────────────────────────────────────────────────┐ │
│  │  ww_mutex mutex;  ← يحمي:                         │ │
│  │    - use_count                                     │ │
│  │    - consumer_list                                 │ │
│  │    - voltage state (constraints)                   │ │
│  │    - bypass state                                  │ │
│  └────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

#### ترتيب الأقفال (Lock Ordering)

```
إذا كان لديك chain من regulators:
  rdev_parent → rdev_child

يجب أخذ الأقفال من الأعلى للأسفل:
  ww_mutex_lock(rdev_parent->mutex)
    └─► ww_mutex_lock(rdev_child->mutex)
          └─► العملية
          └─► ww_mutex_unlock(rdev_child->mutex)
    └─► ww_mutex_unlock(rdev_parent->mutex)

الـ ww_mutex يكشف تلقائياً إذا حدث deadlock ويُعيد -EDEADLK
ليُعيد المستدعي المحاولة بترتيب صحيح.
```

#### الـ `blocking_notifier` للأحداث

الأحداث تستخدم `blocking_notifier_chain` — يعني الـ notifier callbacks تعمل في **sleeping context**، وممكن تأخذ locks. هذا مختلف عن `atomic_notifier` الذي لا يسمح بالنوم.

```
┌──────────────────────────────────────────────────────────────┐
│  Lock Summary                                                │
├──────────────┬───────────────────────────────────────────────┤
│ ww_mutex     │ يحمي state الـ rdev — voltage, enable count   │
│ blocking_nb  │ يحمي notifier chain — قابل للنوم             │
│ regulator_list_mutex (global) │ يحمي قائمة كل الـ rdvs       │
└──────────────┴───────────────────────────────────────────────┘
```

---

### مخطط العلاقات الكاملة

```
 Board/DT
   │  "vdd-supply = <&ldo1>"
   │
   ▼
struct regulator_init_data       struct regulator_desc
   │  (constraints, supply_name)      │  (name, ops, n_voltages)
   └──────────────┬───────────────────┘
                  ▼
         struct regulator_dev  (rdev)
              │    │    │
              │    │    └── supply ──► parent rdev (chained supply)
              │    │
              │    └── consumer_list
              │              │
              │    ┌─────────┼──────────┐
              │    ▼         ▼          ▼
              │  reg[0]    reg[1]     reg[2]    ← struct regulator (per consumer)
              │  (WiFi)   (Camera)   (CPU)
              │
              └── desc->ops ──► struct regulator_ops
                                    │
                                    ├── enable()
                                    ├── disable()
                                    ├── set_voltage()
                                    ├── get_voltage()
                                    └── set_mode()
                                            │
                                            ▼
                                     Hardware Registers
                                     (I2C/SPI/MMIO to PMIC)
```
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### مجموعة: Acquire / Release

| Function | Signature (مختصرة) | الغرض |
|---|---|---|
| `regulator_get` | `regulator_get(dev, id)` → `struct regulator *` | الحصول على handle الـ regulator |
| `devm_regulator_get` | `devm_regulator_get(dev, id)` → `struct regulator *` | نفسه + managed lifetime |
| `regulator_get_exclusive` | `regulator_get_exclusive(dev, id)` → `struct regulator *` | exclusive ownership |
| `regulator_get_optional` | `regulator_get_optional(dev, id)` → `struct regulator *` | لو مش موجود → `ERR_PTR(-ENODEV)` |
| `regulator_put` | `regulator_put(reg)` → void | تحرير الـ handle |
| `devm_regulator_put` | `devm_regulator_put(reg)` → void | manual release لـ devm handle |

#### مجموعة: Enable / Disable

| Function | الغرض |
|---|---|
| `regulator_enable` | يشغّل الـ regulator (reference-counted) |
| `regulator_disable` | يطفّي (reference-counted) |
| `regulator_force_disable` | يطفّي فوراً بغض النظر عن الـ refcount |
| `regulator_disable_deferred` | يطفّي بعد N مللي ثانية |
| `regulator_is_enabled` | يسأل هل شغّال؟ |

#### مجموعة: Voltage Control

| Function | الغرض |
|---|---|
| `regulator_set_voltage` | ضبط جهد بين min و max |
| `regulator_get_voltage` | قراءة الجهد الحالي |
| `regulator_sync_voltage` | مزامنة الـ voltage في الهاردوير |
| `regulator_set_voltage_time` | وقت الـ ramp بين جهدين |
| `regulator_count_voltages` | عدد الجهود المتاحة |
| `regulator_list_voltage` | الجهد عند selector معين |
| `regulator_is_supported_voltage` | هل النطاق مدعوم؟ |
| `regulator_get_linear_step` | خطوة الجهد في Linear regulators |
| `regulator_set_voltage_triplet` | helper: target أو fallback |
| `regulator_set_voltage_tol` | helper: target ± tolerance |

#### مجموعة: Current & Power Budget

| Function | الغرض |
|---|---|
| `regulator_set_current_limit` | ضبط حد التيار |
| `regulator_get_current_limit` | قراءة حد التيار |
| `regulator_set_load` | إبلاغ الـ framework بالـ load (لـ DRMS) |
| `regulator_get_unclaimed_power_budget` | الـ power budget المتاح |
| `regulator_request_power_budget` | حجز power budget |
| `regulator_free_power_budget` | تحرير power budget |

#### مجموعة: Mode Control

| Function | الغرض |
|---|---|
| `regulator_set_mode` | ضبط وضع التشغيل (FAST/NORMAL/IDLE/STANDBY) |
| `regulator_get_mode` | قراءة الوضع الحالي |
| `regulator_allow_bypass` | السماح بـ bypass mode |

#### مجموعة: Bulk Operations

| Function | الغرض |
|---|---|
| `regulator_bulk_get` | الحصول على عدة regulators دفعة واحدة |
| `devm_regulator_bulk_get` | نفسه + managed |
| `devm_regulator_bulk_get_const` | const-friendly bulk get |
| `devm_regulator_bulk_get_exclusive` | exclusive bulk get |
| `regulator_bulk_enable` | تشغيل عدة regulators |
| `regulator_bulk_disable` | إطفاء عدة regulators |
| `regulator_bulk_force_disable` | إطفاء قسري |
| `regulator_bulk_free` | تحرير عدة regulators |
| `devm_regulator_bulk_get_enable` | get + enable مرة واحدة |
| `regulator_bulk_set_supply_names` | helper لملء أسماء الـ supply |

#### مجموعة: Notifier

| Function | الغرض |
|---|---|
| `regulator_register_notifier` | تسجيل callback لأحداث الـ regulator |
| `devm_regulator_register_notifier` | نفسه + managed |
| `regulator_unregister_notifier` | إلغاء تسجيل الـ callback |

#### مجموعة: Suspend

| Function | الغرض |
|---|---|
| `regulator_suspend_enable` | تشغيل في حالة suspend |
| `regulator_suspend_disable` | إطفاء في حالة suspend |
| `regulator_set_suspend_voltage` | ضبط جهد الـ suspend |

#### مجموعة: Hardware / Low-level

| Function | الغرض |
|---|---|
| `regulator_get_regmap` | الحصول على `regmap` الـ regulator |
| `regulator_get_hardware_vsel_register` | عنوان ومask الـ voltage selector register |
| `regulator_list_hardware_vsel` | تحويل selector → hardware vsel value |
| `regulator_hardware_enable` | تشغيل/إطفاء مباشرة في الهاردوير |
| `regulator_get_error_flags` | قراءة أعلام الأخطاء |

#### مجموعة: OF / DeviceTree

| Function | الغرض |
|---|---|
| `of_regulator_get` | get عبر DT node |
| `devm_of_regulator_get` | نفسه + managed |
| `of_regulator_get_optional` | optional get عبر DT |
| `devm_of_regulator_get_optional` | نفسه + managed |
| `of_regulator_bulk_get_all` | الحصول على كل regulators من DT node |

#### مجموعة: Supply Alias

| Function | الغرض |
|---|---|
| `regulator_register_supply_alias` | ربط supply name بـ alias device |
| `regulator_unregister_supply_alias` | إلغاء الربط |
| `regulator_bulk_register_supply_alias` | bulk version |
| `devm_regulator_register_supply_alias` | managed version |

---

### مجموعة 1: Acquire & Release

**الفكرة:** قبل ما consumer يستخدم regulator، لازم يحصل على `struct regulator *` handle. الـ framework يعمل reference counting — لو consumer واحد أخد الـ handle، الـ regulator core يشوف مين بيشغّل مين.

---

#### `regulator_get`

```c
struct regulator *__must_check regulator_get(struct device *dev,
                                              const char *id);
```

**الـ** `regulator_get` بيدور على الـ regulator المربوط بـ `dev` تحت الاسم `id` (زي `"vcc"`, `"vin"`). لو مش موجود في الـ device tree أو platform data، بيرجع dummy regulator (مش error) — ده يخلّي الـ driver يشتغل على boards من غير regulator control. الـ caller **لازم** يفحص بـ `IS_ERR()`.

**البارامترات:**
- `dev` — الـ device الطالبة (consumer). بيُستخدم للـ DT lookup عبر `dev->of_node`.
- `id` — اسم الـ supply زي `"vcc-io"`. بيتطابق مع الـ `supply-names` في الـ DT.

**القيمة المُرجعة:** pointer صالح أو `ERR_PTR` في حالة خطأ جسيم.

**تفاصيل مهمة:**
- يحصل على `regulator_list_mutex` داخلياً أثناء البحث.
- لو `CONFIG_REGULATOR` مش موجود، بيرجع `NULL` — والـ consumer code لازم يتعامل مع `NULL` كـ dummy.
- **الـ caller:** driver probe أو init context فقط (may sleep).

---

#### `devm_regulator_get`

```c
struct regulator *__must_check devm_regulator_get(struct device *dev,
                                                   const char *id);
```

**الـ** `devm_regulator_get` نفس `regulator_get` بس بيربط الـ lifetime بـ `dev` عبر devres. لما `dev` يتحرر، الـ `regulator_put` بيتعمل تلقائياً من غير ما الـ driver يحتاج يعمله يدوياً في الـ `remove` callback.

**البارامترات:** نفس `regulator_get`.

**القيمة المُرجعة:** نفس `regulator_get`.

**الـ caller:** الاستخدام الشائع في كل الـ modern drivers اللي بتستخدم managed resources.

---

#### `regulator_get_exclusive`

```c
struct regulator *__must_check regulator_get_exclusive(struct device *dev,
                                                        const char *id);
```

**الـ** `regulator_get_exclusive` بيطلب ownership حصري للـ regulator. لو فيه consumer تاني بيستخدمه، بيرجع `ERR_PTR(-EPERM)`. ده مهم لـ consumers محتاجة تتحكم في الـ voltage بحرية من غير تعارض.

**البارامترات:** نفس `regulator_get`.

**القيمة المُرجعة:** pointer صالح، `ERR_PTR(-EPERM)` لو محجوز، أو كود خطأ تاني.

**تفاصيل مهمة:**
- بيمنع أي consumer تاني يعمل `get` على نفس الـ regulator.
- حتى لو shared regulator، الـ exclusive owner بيقدر يعمل voltage changes بدون تفاوض.
- **الـ caller:** PMIC drivers أو subsystems تحتاج full control.

---

#### `regulator_get_optional`

```c
struct regulator *__must_check regulator_get_optional(struct device *dev,
                                                       const char *id);
```

**الـ** `regulator_get_optional` زي `regulator_get` بس لو مش موجود بيرجع `ERR_PTR(-ENODEV)` بدل dummy. مفيد لما الـ supply اختياري ومش وجوده مقبول.

**الفرق الجوهري:** الـ `regulator_get` → silent dummy لو مفيش. الـ `regulator_get_optional` → hard error لو مفيش. الـ caller يقرر يعمل إيه.

**القيمة المُرجعة:** pointer صالح أو `ERR_PTR(-ENODEV)` لو مش موجود.

---

#### `devm_regulator_get_enable`

```c
int devm_regulator_get_enable(struct device *dev, const char *id);
```

**الـ** `devm_regulator_get_enable` بيعمل get و enable دفعة واحدة في سطر واحد. بيفيد لما الـ regulator محتاج يبقى دايماً شغّال طول عمر الـ device ومفيش حاجة لـ dynamic enable/disable.

**البارامترات:**
- `dev` — الـ device الطالبة.
- `id` — اسم الـ supply.

**القيمة المُرجعة:** 0 نجاح، أو كود خطأ سالب.

**الـ caller:** driver probe لما لا توجد حاجة لـ dynamic enable/disable.

---

#### `devm_regulator_get_enable_optional`

```c
int devm_regulator_get_enable_optional(struct device *dev, const char *id);
```

**الـ** `devm_regulator_get_enable_optional` زي `devm_regulator_get_enable` بس لو الـ regulator مش موجود بيرجع 0 (مش error). الـ supply اختياري.

**القيمة المُرجعة:** 0 نجاح أو supply غير موجود، كود خطأ سالب لو فشل الـ enable.

---

#### `devm_regulator_get_enable_read_voltage`

```c
int devm_regulator_get_enable_read_voltage(struct device *dev, const char *id);
```

**الـ** `devm_regulator_get_enable_read_voltage` بيعمل get + enable + بيرجع الجهد الحالي بالميكروفولت. مفيد لما الـ driver محتاج يعرف الجهد المتاح لاتخاذ قرارات تشغيل.

**القيمة المُرجعة:** الجهد بالـ µV نجاح، أو `ERR_PTR` error code سالب.

---

#### `regulator_put`

```c
void regulator_put(struct regulator *regulator);
```

**الـ** `regulator_put` بيحرر الـ handle ويقلل الـ refcount. لو كان enabled، لازم يتعمل `regulator_disable` الأول وإلا سيحدث imbalance في الـ enable refcount.

**تفاصيل مهمة:** لا يجوز استخدام الـ `regulator` pointer بعد الـ put. يُستخدم مع `regulator_get` فقط (مش devm version).

---

### مجموعة 2: Enable / Disable

**الفكرة:** الـ regulator framework بيعمل **reference counting** على الـ enable. يعني لو 3 consumers عملوا `enable`، الـ regulator بيتطفّى بس لما الـ 3 يعملوا `disable`. ده يحمي devices من الـ power-cut المفاجئ.

```
Consumer A: enable()  → refcount=1 → HW ON
Consumer B: enable()  → refcount=2 → still ON
Consumer A: disable() → refcount=1 → still ON
Consumer B: disable() → refcount=0 → HW OFF
```

---

#### `regulator_enable`

```c
int __must_check regulator_enable(struct regulator *regulator);
```

**الـ** `regulator_enable` بيزود الـ enable refcount بـ 1. لو كان الـ refcount صفر، يشغّل الـ hardware فعلاً عبر الـ `ops->enable`. بيستنى لغاية ما الجهد يستقر (ramp delay) قبل ما يرجع.

**البارامترات:**
- `regulator` — الـ handle اللي رجع من `regulator_get`.

**القيمة المُرجعة:** 0 نجاح، أو `errno` سالب (مثلاً `-EIO` لو الـ PMIC ما ردّش).

**تفاصيل مهمة:**
- يحصل على `rdev->mutex` أثناء التنفيذ.
- لو في parent regulator (supply chain)، بيشغّله الأول recursively.
- **لازم** تفحص الـ return value — `__must_check`.
- **الـ caller:** process context (يمكن ينام لو الـ regulator ops تستدعي I2C/SPI).

---

#### `regulator_disable`

```c
int regulator_disable(struct regulator *regulator);
```

**الـ** `regulator_disable` بيقلل الـ refcount. لو وصل صفر، يطفّي الـ hardware. لو الـ regulator عليه constraints تمنع الإطفاء (مثلاً `always-on` في DT)، بيتجاهل الإطفاء بصمت.

**القيمة المُرجعة:** 0 نجاح، كود خطأ سالب لو حصل مشكلة في الـ hardware.

**تفاصيل مهمة:** الـ parent supply chain بيتطفّى تلقائياً لو مفيش consumer تاني يستخدمه.

---

#### `regulator_force_disable`

```c
int regulator_force_disable(struct regulator *regulator);
```

**الـ** `regulator_force_disable` بيطفّي الـ regulator فوراً بغض النظر عن الـ refcount. ده **خطر** في الأنظمة اللي فيها multiple consumers لأنه ممكن يطفّي supply device تانية محتاجاه.

**القيمة المُرجعة:** 0 نجاح، كود خطأ سالب.

**الاستخدام:** emergency shutdown أو error recovery فقط.

---

#### `regulator_disable_deferred`

```c
int regulator_disable_deferred(struct regulator *regulator, int ms);
```

**الـ** `regulator_disable_deferred` بيجدول `disable` بعد `ms` مللي ثانية. مفيد لما الـ hardware محتاج وقت للاستقرار قبل الإطفاء أو لـ debouncing في power sequencing.

**البارامترات:**
- `regulator` — الـ handle.
- `ms` — التأخير بالمللي ثانية قبل تنفيذ الـ disable.

**تفاصيل مهمة:** بيستخدم `delayed_work` داخلياً. لو اتعمل `enable` قبل ما الـ delay ينتهي، يُلغي الـ pending disable.

---

#### `regulator_is_enabled`

```c
int regulator_is_enabled(struct regulator *regulator);
```

**الـ** `regulator_is_enabled` يسأل هل الـ regulator شغّال فعلاً في الـ hardware عبر `ops->is_enabled`. مش بيشوف الـ software refcount بس بيسأل الـ hardware state الفعلي.

**القيمة المُرجعة:** 1 = مشغّل، 0 = متطفّي، كود خطأ سالب لو الـ ops مش متاحة.

---

### مجموعة 3: Voltage Control

**الفكرة:** الـ voltage control هو صميم الـ regulator framework. كل consumer بيقول "أنا محتاج جهد بين X و Y فولت" والـ framework بيحل الـ constraints لكل الـ consumers ويختار جهد مناسب للكل.

```
Consumer A: set_voltage(1700000, 1900000) → طلب 1.7V..1.9V
Consumer B: set_voltage(1800000, 2000000) → طلب 1.8V..2.0V
Framework: يختار 1.8V (يُرضي الاتنين)
```

---

#### `regulator_set_voltage`

```c
int regulator_set_voltage(struct regulator *regulator, int min_uV, int max_uV);
```

**الـ** `regulator_set_voltage` بيطلب من الـ framework ضبط الجهد في النطاق `[min_uV, max_uV]`. الـ framework بيجمع طلبات كل الـ consumers ويختار جهداً يُرضي الكل. لو التعارض مش قابل للحل، بيرجع error.

**البارامترات:**
- `min_uV` — الحد الأدنى للجهد المقبول بالميكروفولت.
- `max_uV` — الحد الأقصى للجهد المقبول بالميكروفولت.

**القيمة المُرجعة:** 0 نجاح، `-EINVAL` لو النطاق غير صالح أو متعارض، `-ENOSYS` لو الـ ops مش موجودة.

**تفاصيل مهمة:**
- بيحصل على `rdev->mutex` أثناء التنفيذ.
- بيمر بـ `_regulator_call_set_voltage` → `ops->set_voltage_sel` أو `ops->set_voltage`.
- **الـ caller:** process context (قد ينام لو I2C/SPI).

**مثال:**
```c
/* طلب 1.8V ± 50mV على بورد ARM */
ret = regulator_set_voltage(vcc_reg, 1750000, 1850000);
if (ret)
    dev_err(dev, "failed to set voltage: %d\n", ret);
```

---

#### `regulator_get_voltage`

```c
int regulator_get_voltage(struct regulator *regulator);
```

**الـ** `regulator_get_voltage` بيقرأ الجهد الحالي الفعلي. لو الـ regulator عنده `ops->get_voltage`، بيستدعيه مباشرة. لو لا، بيرجع الجهد المطلوب الأخير المخزّن في الـ framework.

**القيمة المُرجعة:** الجهد بالـ µV نجاح، كود خطأ سالب لو مش موجود.

---

#### `regulator_sync_voltage`

```c
int regulator_sync_voltage(struct regulator *regulator);
```

**الـ** `regulator_sync_voltage` بيُعيد ضبط الـ voltage في الـ hardware بناءً على القيمة المخزّنة في الـ framework. مفيد بعد الخروج من suspend لو الـ PMIC reset حالته وعايز تتأكد إن الـ voltage صح.

**القيمة المُرجعة:** 0 نجاح، كود خطأ سالب.

---

#### `regulator_set_voltage_time`

```c
int regulator_set_voltage_time(struct regulator *regulator,
                                int old_uV, int new_uV);
```

**الـ** `regulator_set_voltage_time` بيحسب وقت الـ ramp اللازم للانتقال من `old_uV` لـ `new_uV` بالميكروثانية. مفيد لـ CPU frequency scaling علشان تعرف كام تستنى بعد `set_voltage` قبل ما ترفع الـ frequency.

**البارامترات:**
- `old_uV` — الجهد الحالي.
- `new_uV` — الجهد المستهدف.

**القيمة المُرجعة:** الوقت بالـ µs، أو كود خطأ سالب.

---

#### `regulator_count_voltages`

```c
int regulator_count_voltages(struct regulator *regulator);
```

**الـ** `regulator_count_voltages` بيرجع عدد الجهود المتقطعة المتاحة (discrete voltage levels). مفيد قبل loop بـ `regulator_list_voltage`.

**القيمة المُرجعة:** عدد موجب، أو كود خطأ سالب.

---

#### `regulator_list_voltage`

```c
int regulator_list_voltage(struct regulator *regulator, unsigned selector);
```

**الـ** `regulator_list_voltage` بيرجع قيمة الجهد (µV) المقابلة لـ `selector` index. يُستخدم للاستعراض عبر loop من 0 لـ `count_voltages - 1` لمعرفة كل الجهود المتاحة.

**البارامترات:**
- `selector` — الـ index من 0 إلى `count_voltages - 1`.

**القيمة المُرجعة:** الجهد بالـ µV، أو 0 لو الـ selector مش صالح، أو كود خطأ سالب.

```c
/* استعراض كل الجهود المتاحة */
int count = regulator_count_voltages(reg);
for (int i = 0; i < count; i++) {
    int uV = regulator_list_voltage(reg, i);
    if (uV > 0)
        pr_info("Voltage[%d] = %d uV\n", i, uV);
}
```

---

#### `regulator_is_supported_voltage`

```c
int regulator_is_supported_voltage(struct regulator *regulator,
                                    int min_uV, int max_uV);
```

**الـ** `regulator_is_supported_voltage` بيفحص هل في جهد متاح في النطاق `[min_uV, max_uV]`. بيفيد قبل الـ `set_voltage` للتحقق وتجنب error handling معقد.

**القيمة المُرجعة:** 1 = مدعوم، 0 = مش مدعوم، كود خطأ سالب.

---

#### `regulator_get_linear_step`

```c
unsigned int regulator_get_linear_step(struct regulator *regulator);
```

**الـ** `regulator_get_linear_step` بيرجع الخطوة بين مستويات الجهد المتتالية (µV) في regulators ذات الجهد الخطي. مفيد لحساب الـ selector من الجهد المطلوب بدون loop.

**القيمة المُرجعة:** الخطوة بالـ µV، أو 0 لو مش linear.

---

#### `regulator_set_voltage_triplet` (inline helper)

```c
static inline int regulator_set_voltage_triplet(struct regulator *regulator,
                                                  int min_uV, int target_uV,
                                                  int max_uV);
```

**الـ** `regulator_set_voltage_triplet` بيحاول يضبط `target_uV..max_uV` الأول. لو فشل، بحاول `min_uV..max_uV`. ده pattern شائع جداً لـ CPU voltage scaling.

**Pseudocode Flow:**
```
if set_voltage(target_uV, max_uV) == 0:
    return 0
else:
    return set_voltage(min_uV, max_uV)
```

---

#### `regulator_set_voltage_tol` (inline helper)

```c
static inline int regulator_set_voltage_tol(struct regulator *regulator,
                                              int new_uV, int tol_uV);
```

**الـ** `regulator_set_voltage_tol` بيضبط الجهد على `new_uV ± tol_uV`. بيحاول الأول `new_uV .. new_uV + tol_uV`، لو فشل `new_uV - tol_uV .. new_uV + tol_uV`. مفيد لما الـ tolerance معروف مسبقاً.

---

#### `regulator_is_supported_voltage_tol` (inline helper)

```c
static inline int regulator_is_supported_voltage_tol(struct regulator *regulator,
                                                       int target_uV, int tol_uV);
```

**الـ** `regulator_is_supported_voltage_tol` بيفحص هل `target_uV ± tol_uV` مدعوم. ده wrapper بسيط على `regulator_is_supported_voltage`.

---

### مجموعة 4: Current & Power Budget

**الفكرة:** بعض الـ regulators بتقدر تحدد حد التيار الأقصى. وفيه مفهوم **power budget** — الـ driver بيقول "أنا هاخد X ميللي وات" والـ framework يتأكد إن المجموع ما يعداش الـ budget المتاح. مفيد في embedded SoCs بـ strict power envelope.

---

#### `regulator_set_current_limit`

```c
int regulator_set_current_limit(struct regulator *regulator,
                                  int min_uA, int max_uA);
```

**الـ** `regulator_set_current_limit` بيضبط حد التيار في النطاق `[min_uA, max_uA]`. مفيد لـ USB power delivery أو charging regulators اللي بتحدد كمية التيار المتاح لكل consumer.

**البارامترات:**
- `min_uA / max_uA` — النطاق بالميكرو أمبير.

**القيمة المُرجعة:** 0 نجاح، كود خطأ سالب.

---

#### `regulator_get_current_limit`

```c
int regulator_get_current_limit(struct regulator *regulator);
```

**الـ** `regulator_get_current_limit` بيقرأ حد التيار الحالي بالـ µA.

**القيمة المُرجعة:** التيار بالـ µA، أو كود خطأ سالب.

---

#### `regulator_set_load`

```c
int regulator_set_load(struct regulator *regulator, int load_uA);
```

**الـ** `regulator_set_load` بيُبلّغ الـ framework بالـ load المتوقع لهذا الـ consumer. الـ framework بيجمع كل الـ loads ويقرر وضع تشغيل الـ regulator (DRMS — Dynamic Regulator Mode Switching) الأكفأ.

**البارامترات:**
- `load_uA` — التيار المتوقع بالـ µA.

**مثال:**
```c
/* قبل الـ heavy IO */
regulator_set_load(vcc, 100000); /* 100mA → سيختار NORMAL mode */

/* بعد الانتهاء والدخول في idle */
regulator_set_load(vcc, 1000);   /* 1mA → سيختار IDLE mode */
```

**تفاصيل مهمة:** لا يُغيّر الجهد مباشرة، بس بيعطي hint للـ framework يختار الـ mode الأكفأ. الـ DRMS يحصل في الـ `regulator_set_voltage` أو بعده.

---

#### `regulator_get_unclaimed_power_budget`

```c
int regulator_get_unclaimed_power_budget(struct regulator *regulator);
```

**الـ** `regulator_get_unclaimed_power_budget` بيرجع كمية الـ power (mW) المتاحة غير المحجوزة.

**القيمة المُرجعة:** الـ power budget بالـ mW، أو `INT_MAX` لو مفيش حد مضبوط.

---

#### `regulator_request_power_budget`

```c
int regulator_request_power_budget(struct regulator *regulator,
                                     unsigned int pw_req);
```

**الـ** `regulator_request_power_budget` بيحجز `pw_req` mW من الـ power budget. لو مفيش كفاية، بيرجع `-ENOSPC` ولازم الـ consumer يقلل طلبه.

**القيمة المُرجعة:** 0 نجاح، `-ENOSPC` لو مفيش كفاية، `-EOPNOTSUPP` لو stub.

---

#### `regulator_free_power_budget`

```c
void regulator_free_power_budget(struct regulator *regulator,
                                   unsigned int pw);
```

**الـ** `regulator_free_power_budget` بيُحرر `pw` mW من الـ power budget المحجوز مسبقاً. لازم يتعمل بعد كل `regulator_request_power_budget` ناجح.

---

### مجموعة 5: Mode Control

**الفكرة:** الـ regulator يمكن يشتغل في modes مختلفة بكفاءات مختلفة. الـ DRMS بيسمح للـ framework يختار تلقائياً، لكن الـ driver تقدر تتحكم يدوياً كمان.

```
REGULATOR_MODE_FAST    (0x1) → عند load عالي جداً + high slew rate (CPU scaling)
REGULATOR_MODE_NORMAL  (0x2) → الوضع الافتراضي لمعظم الحالات
REGULATOR_MODE_IDLE    (0x4) → عند load خفيف (device في idle)
REGULATOR_MODE_STANDBY (0x8) → عند load خفيف جداً (device في sleep)
```

---

#### `regulator_set_mode`

```c
int regulator_set_mode(struct regulator *regulator, unsigned int mode);
```

**الـ** `regulator_set_mode` بيضبط وضع تشغيل الـ regulator يدوياً. الـ mode قيمة من `REGULATOR_MODE_*`.

**البارامترات:**
- `mode` — الوضع المطلوب من `REGULATOR_MODE_FAST/NORMAL/IDLE/STANDBY`.

**القيمة المُرجعة:** 0 نجاح، كود خطأ سالب.

**تفاصيل مهمة:** في الغالب الـ DRMS يتعامل مع ده تلقائياً بناءً على `set_load`. يُستخدم هذا الـ API للـ manual override فقط.

---

#### `regulator_get_mode`

```c
unsigned int regulator_get_mode(struct regulator *regulator);
```

**الـ** `regulator_get_mode` بيرجع الوضع الحالي. في حالة `!CONFIG_REGULATOR` بيرجع `REGULATOR_MODE_NORMAL` دايماً.

**القيمة المُرجعة:** قيمة `REGULATOR_MODE_*`.

---

#### `regulator_allow_bypass`

```c
int regulator_allow_bypass(struct regulator *regulator, bool allow);
```

**الـ** `regulator_allow_bypass` بيُعلم الـ framework إن هذا الـ consumer موافق على تشغيل الـ regulator في bypass mode (الجهد ييجي مباشرة من الـ input من غير تنظيم). الـ framework يشغّل الـ bypass بس لو كل الـ consumers وافقوا.

**البارامترات:**
- `allow` — `true` للموافقة على الـ bypass، `false` للرفض.

**القيمة المُرجعة:** 0 نجاح، كود خطأ سالب.

---

#### `regulator_get_error_flags`

```c
int regulator_get_error_flags(struct regulator *regulator,
                                unsigned int *flags);
```

**الـ** `regulator_get_error_flags` بيقرأ أعلام الأخطاء الحالية من الـ hardware ويضعها في `*flags`. الـ flags دي OR من قيم `REGULATOR_ERROR_*`.

**البارامترات:**
- `flags` — pointer لـ `unsigned int` يُملأ بالـ error flags.

**الـ flags الممكنة:**
```c
REGULATOR_ERROR_UNDER_VOLTAGE     /* جهد أقل من المطلوب */
REGULATOR_ERROR_OVER_CURRENT      /* تيار زيادة */
REGULATOR_ERROR_REGULATION_OUT    /* خروج عن التنظيم */
REGULATOR_ERROR_FAIL              /* فشل كامل */
REGULATOR_ERROR_OVER_TEMP         /* حرارة زيادة */
/* + warning versions بـ _WARN suffix */
```

**الاستخدام:** يُستخدم في interrupt handlers أو polling للتحقق من صحة الـ regulator.

---

### مجموعة 6: Bulk Operations

**الفكرة:** معظم الـ SoC drivers بتحتاج أكثر من supply واحد. الـ Bulk API بيوفر طريقة مريحة للتعامل مع عدة regulators معاً بدل loops يدوية مع error handling معقد.

**الـ `struct regulator_bulk_data`:**
```c
struct regulator_bulk_data {
    const char *supply;      /* اسم الـ supply — يملّاه الـ caller */
    struct regulator *consumer; /* يملّاه الـ framework */
    int init_load_uA;        /* يُستدعى set_load بعد الـ get */
    /* private: */
    int ret;
};
```

---

#### `regulator_bulk_get`

```c
int __must_check regulator_bulk_get(struct device *dev,
                                      int num_consumers,
                                      struct regulator_bulk_data *consumers);
```

**الـ** `regulator_bulk_get` بيعمل `regulator_get` على مصفوفة كاملة. لو واحد فيهم فشل، بيعمل rollback ويحرر كل اللي نجح — atomic من ناحية error handling.

**البارامترات:**
- `num_consumers` — عدد العناصر.
- `consumers` — مصفوفة `regulator_bulk_data` مملّاها الـ caller بـ `supply` names.

**استخدام نموذجي:**
```c
static struct regulator_bulk_data supplies[] = {
    { .supply = "vcc-core" },
    { .supply = "vcc-io"   },
    { .supply = "vcc-phy"  },
};

ret = devm_regulator_bulk_get(dev, ARRAY_SIZE(supplies), supplies);
if (ret)
    return ret;

ret = regulator_bulk_enable(ARRAY_SIZE(supplies), supplies);
```

---

#### `devm_regulator_bulk_get_const`

```c
int __must_check devm_regulator_bulk_get_const(
    struct device *dev, int num_consumers,
    const struct regulator_bulk_data *in_consumers,
    struct regulator_bulk_data **out_consumers);
```

**الـ** `devm_regulator_bulk_get_const` بيأخد مصفوفة `const` من الـ supply names ويعمل نسخة managed ديناميكية. مفيد لما تعريف الـ supplies في `const` section أو `static const`.

**Pseudocode:**
```
kmemdup(in_consumers) → mutable copy
devm_regulator_bulk_get(mutable copy)
*out_consumers = mutable copy
```

---

#### `devm_regulator_bulk_get_enable`

```c
int devm_regulator_bulk_get_enable(struct device *dev,
                                     int num_consumers,
                                     const char * const *id);
```

**الـ** `devm_regulator_bulk_get_enable` بيعمل get + enable لعدة regulators عن طريق array من الأسماء فقط. بيوفر أقصر طريقة ممكنة في الـ drivers الحديثة.

```c
static const char * const supplies[] = { "vcc", "vio", "vmem" };
ret = devm_regulator_bulk_get_enable(dev, ARRAY_SIZE(supplies), supplies);
```

---

#### `regulator_bulk_enable` / `regulator_bulk_disable`

```c
int __must_check regulator_bulk_enable(int num_consumers,
                                         struct regulator_bulk_data *consumers);
int regulator_bulk_disable(int num_consumers,
                            struct regulator_bulk_data *consumers);
```

**الـ** `regulator_bulk_enable` بيشغّل كل الـ regulators في المصفوفة. لو واحد فشل، بيوقف ويرجع الـ error. الـ `regulator_bulk_disable` عكسه.

**تفاصيل مهمة:** الـ rollback في حالة partial failure مش ضمون بالكامل — الـ error path يحتاج مراجعة في الـ driver.

---

#### `regulator_bulk_set_supply_names` (inline helper)

```c
void regulator_bulk_set_supply_names(struct regulator_bulk_data *consumers,
                                      const char *const *supply_names,
                                      unsigned int num_supplies);
```

**الـ** `regulator_bulk_set_supply_names` بيملّي الـ `supply` field في مصفوفة `regulator_bulk_data` من مصفوفة أسماء. بيوفر ملء يدوي متكرر.

---

### مجموعة 7: Notifier / Events

**الفكرة:** الـ regulators بتطلق أحداث عند مشاكل زي under-voltage أو over-current. الـ consumers تقدر تسجّل callbacks لمعالجة هذه الأحداث في real-time.

**أهم الـ events (من `uapi/regulator/regulator.h`):**

| Event | القيمة | المعنى |
|---|---|---|
| `REGULATOR_EVENT_UNDER_VOLTAGE` | `0x01` | جهد أقل من المطلوب |
| `REGULATOR_EVENT_OVER_CURRENT` | `0x02` | تيار زيادة |
| `REGULATOR_EVENT_FAIL` | `0x08` | فشل كامل |
| `REGULATOR_EVENT_VOLTAGE_CHANGE` | `0x40` | الجهد اتغيّر |
| `REGULATOR_EVENT_PRE_VOLTAGE_CHANGE` | `0x100` | قبل تغيير الجهد (data = `pre_voltage_change_data`) |
| `REGULATOR_EVENT_DISABLE` | `0x80` | الـ regulator اتطفّى |
| `REGULATOR_EVENT_ENABLE` | `0x1000` | الـ regulator اتشغّل |

---

#### `regulator_register_notifier`

```c
int regulator_register_notifier(struct regulator *regulator,
                                  struct notifier_block *nb);
```

**الـ** `regulator_register_notifier` بيسجّل `nb->notifier_call` كـ callback لأحداث الـ regulator. بيستخدم الـ kernel blocking notifier chain داخلياً.

**البارامترات:**
- `nb` — الـ `notifier_block` بـ `notifier_call` function pointer.

**مثال:**
```c
static int my_regulator_event(struct notifier_block *nb,
                               unsigned long event, void *data)
{
    if (event & REGULATOR_EVENT_UNDER_VOLTAGE) {
        /* استجابة طارئة: خفّض الـ performance */
        throttle_performance();
    }
    if (event & REGULATOR_EVENT_PRE_VOLTAGE_CHANGE) {
        struct pre_voltage_change_data *pvc = data;
        pr_info("voltage changing from %lu to %lu..%lu uV\n",
                pvc->old_uV, pvc->min_uV, pvc->max_uV);
    }
    return NOTIFY_OK;
}

static struct notifier_block my_nb = {
    .notifier_call = my_regulator_event,
};

/* في probe */
regulator_register_notifier(vcc, &my_nb);
```

---

#### `devm_regulator_register_notifier`

```c
int devm_regulator_register_notifier(struct regulator *regulator,
                                       struct notifier_block *nb);
```

**الـ** `devm_regulator_register_notifier` نفس `regulator_register_notifier` بس managed — بيتلغي تلقائياً لما الـ device يتحرر.

---

### مجموعة 8: Suspend Support

**الفكرة:** أثناء system suspend، بعض الـ regulators لازم يشتغلوا بجهد مختلف (suspend voltage) أو يتطفّوا. الـ framework بيوفر API مخصص لده. الـ `state` هنا بيعبّر عن السيناريو: `PM_SUSPEND_MEM` (S3/deep sleep) أو `PM_SUSPEND_STANDBY` (S1).

---

#### `regulator_set_suspend_voltage`

```c
int regulator_set_suspend_voltage(struct regulator *regulator, int min_uV,
                                   int max_uV, suspend_state_t state);
```

**الـ** `regulator_set_suspend_voltage` بيضبط الجهد اللي هيستخدمه الـ regulator في حالة الـ suspend المحددة. بيُخزّن الجهد فقط — التطبيق الفعلي بيحصل وقت الـ suspend عبر الـ PM core.

**البارامترات:**
- `state` — `PM_SUSPEND_MEM` أو `PM_SUSPEND_STANDBY`.

---

#### `regulator_suspend_enable` / `regulator_suspend_disable`

```c
int regulator_suspend_enable(struct regulator_dev *rdev, suspend_state_t state);
int regulator_suspend_disable(struct regulator_dev *rdev, suspend_state_t state);
```

**الـ** `regulator_suspend_enable` بيشغّل الـ regulator في حالة suspend محددة. **ملاحظة مهمة:** الـ parameter هنا `regulator_dev *` (الجانب provider) مش `regulator *` (الجانب consumer) — يُستدعى من الـ PM core وليس من الـ consumer مباشرة.

---

### مجموعة 9: Hardware / Low-level Access

**الفكرة:** بعض الـ advanced use cases (مثلاً AVS — Adaptive Voltage Scaling في الـ CPUs) محتاجة تكتب الـ voltage selector مباشرة في الـ hardware register من غير ما تمر بكل الـ framework layers. ده يقلل الـ latency في voltage changes الـ time-critical.

---

#### `regulator_get_regmap`

```c
struct regmap *regulator_get_regmap(struct regulator *regulator);
```

**الـ** `regulator_get_regmap` بيرجع الـ `regmap` المرتبط بالـ regulator. مفيد لو الـ consumer محتاج يعمل read/write مباشر على الـ PMIC registers لحاجة مش مُغطّاة بالـ API.

**القيمة المُرجعة:** pointer صالح أو `ERR_PTR(-EOPNOTSUPP)` لو مش متاح.

---

#### `regulator_get_hardware_vsel_register`

```c
int regulator_get_hardware_vsel_register(struct regulator *regulator,
                                           unsigned *vsel_reg,
                                           unsigned *vsel_mask);
```

**الـ** `regulator_get_hardware_vsel_register` بيعطي عنوان الـ register ومask اللي بيتكتب فيه الـ voltage selector مباشرة في الـ PMIC.

**البارامترات:**
- `vsel_reg` — OUT: عنوان الـ register.
- `vsel_mask` — OUT: الـ bitmask للـ voltage selector في الـ register.

**الاستخدام:** AVS implementations على بعض الـ SoCs بتكتب الـ vsel مباشرة في interrupt context بدون I2C overhead.

---

#### `regulator_list_hardware_vsel`

```c
int regulator_list_hardware_vsel(struct regulator *regulator, unsigned selector);
```

**الـ** `regulator_list_hardware_vsel` بيحوّل abstract voltage selector (0..N) إلى القيمة الفعلية اللي لازم تتكتب في الـ hardware register (hardware vsel value).

**القيمة المُرجعة:** قيمة الـ hardware register، أو كود خطأ سالب.

---

#### `regulator_hardware_enable`

```c
int regulator_hardware_enable(struct regulator *regulator, bool enable);
```

**الـ** `regulator_hardware_enable` بيُشغّل أو يُطفّي الـ regulator مباشرة في الـ hardware من غير أي reference counting أو constraint checking.

**تحذير:** ده bypass كامل للـ framework. يُستخدم فقط في حالات خاصة جداً زي early boot أو platform-specific power sequences.

---

### مجموعة 10: OF / DeviceTree

**الفكرة:** في الأنظمة اللي عندها Device Tree، الـ regulator connections بتُعرّف في الـ DT. الـ `of_regulator_*` functions بتسمح للـ drivers تعمل lookup عبر `device_node` مباشرة بدل الاعتماد على `dev->of_node` فقط.

---

#### `of_regulator_get` / `devm_of_regulator_get`

```c
struct regulator *__must_check of_regulator_get(struct device *dev,
                                                  struct device_node *node,
                                                  const char *id);
struct regulator *__must_check devm_of_regulator_get(struct device *dev,
                                                      struct device_node *node,
                                                      const char *id);
```

**الـ** `of_regulator_get` بيعمل regulator lookup من `node` المحدد بدل `dev->of_node`. مفيد لما الـ supply مرتبط بـ sub-node في الـ DT مش الـ device root node.

**البارامترات:**
- `node` — الـ DT node اللي فيه الـ supply phandle.
- `id` — اسم الـ supply.

---

#### `of_regulator_get_optional` / `devm_of_regulator_get_optional`

```c
struct regulator *__must_check of_regulator_get_optional(struct device *dev,
                                                           struct device_node *node,
                                                           const char *id);
```

**الـ** `of_regulator_get_optional` نفس `of_regulator_get` بس لو مش موجود بيرجع `ERR_PTR(-ENODEV)` مش dummy.

---

#### `of_regulator_bulk_get_all`

```c
int __must_check of_regulator_bulk_get_all(struct device *dev,
                                             struct device_node *np,
                                             struct regulator_bulk_data **consumers);
```

**الـ** `of_regulator_bulk_get_all` بيجمع **كل** الـ regulators المذكورة في الـ DT node `np` دفعة واحدة. بيعمل allocation للمصفوفة داخلياً ويرجعها في `*consumers`.

**القيمة المُرجعة:** عدد الـ regulators اللي اتجمعت (موجب)، أو كود خطأ سالب.

**تفاصيل مهمة:** يُستخدم في drivers اللي عدد الـ supplies بتاعتها غير ثابت ومعرّف ديناميكياً في الـ DT.

---

### مجموعة 11: Supply Alias

**الفكرة:** بعض الـ boards بتربط supply name من driver معين بـ regulator موجود على device تانية. الـ alias API بيسمح بعمل mapping ده من غير تعديل الـ driver code — مفيد جداً لـ board-specific configurations.

---

#### `regulator_register_supply_alias`

```c
int regulator_register_supply_alias(struct device *dev, const char *id,
                                      struct device *alias_dev,
                                      const char *alias_id);
```

**الـ** `regulator_register_supply_alias` بيقول للـ framework: "لما `dev` يطلب supply بـ id `id`، روح دور عليه في `alias_dev` تحت الاسم `alias_id`."

**مثال:** driver بيطلب `"vdd"` لكن البورد عنده الـ regulator تحت `"vmain"` في PMIC تاني — من غير alias، اللوكأب هيفشل.

```c
/* في board setup code */
regulator_register_supply_alias(&my_device.dev, "vdd",
                                  &pmic_device.dev, "vmain");
```

**القيمة المُرجعة:** 0 نجاح، كود خطأ سالب.

---

#### `regulator_unregister_supply_alias`

```c
void regulator_unregister_supply_alias(struct device *dev, const char *id);
```

**الـ** `regulator_unregister_supply_alias` بيُلغي الـ alias المسجّل. لازم يتعمل قبل الـ device removal.

---

#### `devm_regulator_register_supply_alias`

```c
int devm_regulator_register_supply_alias(struct device *dev, const char *id,
                                           struct device *alias_dev,
                                           const char *alias_id);
```

**الـ** `devm_regulator_register_supply_alias` managed version — الـ alias بيتلغي تلقائياً لما `dev` يتحرر.

---

### مجموعة 12: Driver Data Helpers

---

#### `regulator_get_drvdata` / `regulator_set_drvdata`

```c
void *regulator_get_drvdata(struct regulator *regulator);
void regulator_set_drvdata(struct regulator *regulator, void *data);
```

**الـ** `regulator_set_drvdata` بيخزّن pointer خاص بالـ consumer في الـ regulator handle. الـ framework مبيلمسش القيمة دي. الـ `regulator_get_drvdata` بيجيبها.

**الاستخدام:** نادر جداً. معظم الـ drivers بتخزن الـ state في `dev_priv` أو struct خاص.

---

#### `regulator_is_equal`

```c
bool regulator_is_equal(struct regulator *reg1, struct regulator *reg2);
```

**الـ** `regulator_is_equal` بيفحص هل `reg1` و`reg2` بيشيروا لنفس الـ physical regulator (نفس الـ `regulator_dev`).

**الاستخدام:** مفيد لو نفس الـ supply اتطلب مرتين من contexts مختلفين وعايز تتأكد إنهم نفس الجهاز لتجنب double-disable.

---

### ملاحظات على الـ Stub Functions والـ `!CONFIG_REGULATOR`

الـ header مليان بـ `static inline` stubs تحت `#else` اللي بتُعرّف لما `CONFIG_REGULATOR` مش مفعّل:

```c
#if defined(CONFIG_REGULATOR)
    /* الـ real kernel implementations */
#else
    /* stubs: معظمها بترجع 0 أو NULL */
    static inline int regulator_enable(...)  { return 0; }
    static inline int regulator_get_voltage(...) { return -EINVAL; }
    static inline struct regulator *regulator_get(...) { return NULL; }
#endif
```

**الفكرة:** الـ consumer drivers بتشتغل compile clean حتى على kernels من غير regulator support. الـ stubs بتحاكي "everything is OK" في معظم الحالات.

**قواعد مهمة في الـ stubs:**
- الـ `regulator_get` stub بترجع `NULL` مش `ERR_PTR` — لازم الـ consumer يتعامل مع `NULL` كـ dummy valid handle.
- الـ `regulator_get_voltage` stub بترجع `-EINVAL` (مش 0) لأن إرجاع 0 كجهد هيكون misleading ويسبب bugs.
- الـ `regulator_get_exclusive` stub بترجع `ERR_PTR(-ENODEV)` مش `NULL` لأن الـ exclusive API ما ينفعش يرجع dummy.
- الـ `regulator_is_enabled` stub بترجع 1 دايماً — الافتراض إن كل حاجة شغّالة لو مفيش control.
## Phase 5: دليل الـ Debugging الشامل

### Software Level

---

#### 1. مدخلات الـ debugfs

الـ regulator framework بيكتب بياناته تحت `/sys/kernel/debug/regulator/`.

```bash
# اعرض كل الـ regulators المتاحة
ls /sys/kernel/debug/regulator/

# مثال على regulator اسمه vdd_cpu
cat /sys/kernel/debug/regulator/vdd_cpu/microvolts
cat /sys/kernel/debug/regulator/vdd_cpu/enable_count
cat /sys/kernel/debug/regulator/vdd_cpu/min_microvolts
cat /sys/kernel/debug/regulator/vdd_cpu/max_microvolts
cat /sys/kernel/debug/regulator/vdd_cpu/microamps
cat /sys/kernel/debug/regulator/vdd_cpu/opmode
cat /sys/kernel/debug/regulator/vdd_cpu/state        # "enabled" أو "disabled"
cat /sys/kernel/debug/regulator/vdd_cpu/bypass
```

| المدخل | المعنى |
|---|---|
| `microvolts` | الجهد الحالي بالـ µV |
| `enable_count` | عدد الـ consumers اللي عملوا `regulator_enable()` |
| `opmode` | FAST / NORMAL / IDLE / STANDBY |
| `state` | هل الـ regulator شغال أو لأ |
| `min_microvolts` / `max_microvolts` | الحدود المسموح بيها |
| `bypass` | هل هو في وضع الـ bypass |

```bash
# اطبع كل الـ regulators وحالتها دفعة واحدة
cat /sys/kernel/debug/regulator/regulator_summary
```

الـ `regulator_summary` بيطبع شجرة كاملة: اسم الـ regulator، الجهد، الـ enable count، والـ consumers المرتبطين بيه.

---

#### 2. مدخلات الـ sysfs

```bash
# لكل regulator device تحت /sys/class/regulator/
ls /sys/class/regulator/

# مثال: regulator.0
cat /sys/class/regulator/regulator.0/name
cat /sys/class/regulator/regulator.0/type          # "voltage" أو "current"
cat /sys/class/regulator/regulator.0/microvolts
cat /sys/class/regulator/regulator.0/min_microvolts
cat /sys/class/regulator/regulator.0/max_microvolts
cat /sys/class/regulator/regulator.0/opmode
cat /sys/class/regulator/regulator.0/state
cat /sys/class/regulator/regulator.0/status        # "ok", "error", "under-voltage", "over-current"
cat /sys/class/regulator/regulator.0/num_users
cat /sys/class/regulator/regulator.0/bypass
cat /sys/class/regulator/regulator.0/requested_microvolts
```

**الـ `status`** مهم جداً — بيعكس قراءة الـ hardware مباشرةً عبر `regulator_ops.get_status`.

---

#### 3. الـ ftrace — Tracepoints والـ Events

الـ regulator framework عنده tracepoints جاهزة في `drivers/regulator/core.c`:

```bash
# اعرض كل events الـ regulator
ls /sys/kernel/tracing/events/regulator/

# المهمين:
# regulator_enable / regulator_enable_delay / regulator_enable_complete
# regulator_disable / regulator_disable_complete
# regulator_set_voltage / regulator_set_voltage_complete
# regulator_get_voltage

# شغّل كل events الـ regulator
echo 1 > /sys/kernel/tracing/events/regulator/enable
echo 1 > /sys/kernel/tracing/events/regulator/filter  # اختياري: صفّي على regulator بعينه

# أو فعّلهم كلهم دفعة واحدة
echo 1 > /sys/kernel/tracing/events/regulator/enable
for f in /sys/kernel/tracing/events/regulator/*/enable; do echo 1 > $f; done

# ابدأ الـ trace
echo 1 > /sys/kernel/tracing/tracing_on
# شغّل الكود اللي بتشك فيه
echo 0 > /sys/kernel/tracing/tracing_on

# اقرأ النتيجة
cat /sys/kernel/tracing/trace
```

مثال على output:

```
     kworker/0:2-123  [000]  1234.567890: regulator_enable: name=vdd_cpu
     kworker/0:2-123  [000]  1234.568012: regulator_enable_complete: name=vdd_cpu
     mydriver-456     [001]  1235.001000: regulator_set_voltage: name=vdd_cpu (min=900000 uV, max=1100000 uV)
     mydriver-456     [001]  1235.002500: regulator_set_voltage_complete: name=vdd_cpu, val=1000000 uV
```

---

#### 4. الـ printk والـ Dynamic Debug

```bash
# فعّل dynamic debug لكل ملفات الـ regulator
echo "file drivers/regulator/*.c +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل لـ core فقط
echo "file drivers/regulator/core.c +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل للـ driver بعينه (مثلاً pmic driver)
echo "file drivers/regulator/mt6358-regulator.c +p" > /sys/kernel/debug/dynamic_debug/control

# أضف timestamps وأسماء الوظائف
echo "file drivers/regulator/core.c +pflt" > /sys/kernel/debug/dynamic_debug/control

# شوف إيه اللي مفعّل حالياً
grep regulator /sys/kernel/debug/dynamic_debug/control
```

لو محتاج طباعة مبكرة في الـ boot قبل ما dynamic debug يشتغل:

```bash
# في kernel command line (bootloader أو grub)
regulator.debug=1
```

---

#### 5. الـ Kernel Config Options للـ Debugging

| الـ Config | الغرض |
|---|---|
| `CONFIG_REGULATOR_DEBUG` | يفعّل رسائل debug تفصيلية في الـ core |
| `CONFIG_DEBUG_FS` | شرط لظهور `/sys/kernel/debug/regulator/` |
| `CONFIG_TRACING` + `CONFIG_FTRACE` | شرط للـ tracepoints |
| `CONFIG_DYNAMIC_DEBUG` | يسمح بتفعيل `pr_debug()` وقت التشغيل |
| `CONFIG_REGULATOR` | الإطار الأساسي — بدونه كل الـ API stubs فارغة |
| `CONFIG_REGULATOR_USERSPACE_CONSUMER` | يسمح بالتحكم من الـ userspace مباشرة |
| `CONFIG_REGULATOR_VIRTUAL_CONSUMER` | regulator وهمي للتست |
| `CONFIG_REGULATOR_FIXED_VOLTAGE` | fixed regulator للـ DT-based debugging |
| `CONFIG_PROVE_LOCKING` | يكشف locking bugs في الـ regulator core |
| `CONFIG_DEBUG_OBJECTS` | يكشف use-after-free على الـ regulator objects |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "CONFIG_REGULATOR|CONFIG_DEBUG_FS"
```

---

#### 6. أدوات خاصة بالـ Subsystem

**الـ userspace consumer interface:**

```bash
# لو CONFIG_REGULATOR_USERSPACE_CONSUMER مفعّل
ls /sys/devices/platform/reg-userspace-consumer.*/

# تحكم في الـ regulator يدوياً من userspace
echo 1 > /sys/devices/platform/reg-userspace-consumer.0/state  # enable
echo 0 > /sys/devices/platform/reg-userspace-consumer.0/state  # disable
```

**الـ virtual consumer:**

```bash
# قراءة وكتابة الجهد مباشرة عبر sysfs
echo 1800000 > /sys/devices/platform/reg-virtual-consumer.0/microvolts_normal
cat /sys/devices/platform/reg-virtual-consumer.0/mode
```

**الـ Generic Netlink events** (من `include/uapi/regulator/regulator.h`):

```bash
# استقبل أحداث الـ regulator عبر netlink
# RAW_EVENT_GROUP = "reg_mc_group"، FAMILY = "reg_event"
# استخدم genl-ctrl-list للاستعلام
genl-ctrl-list | grep reg

# أو بـ python/libnl لاستقبال REGULATOR_EVENT_OVER_CURRENT وغيرها
```

**فحص الـ constraints:**

```bash
# اطبع شجرة الـ regulator مع الـ constraints
cat /sys/kernel/debug/regulator/regulator_summary
```

مثال على الخرج:

```
 regulator                      use open bypass  min_uV  max_uV  load_uA
 ----------------------------------------
 vdd_core                         1    1      0  900000 1100000    50000
  +--> vdd_cpu (alias)            2    1      0  800000 1200000   100000
 vcc_io                           0    0      0 1800000 1800000        0
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ Kernel Log | المعنى | الإصلاح |
|---|---|---|
| `regulator_get: vdd_cpu supply not found` | الـ consumer طلب regulator مش موجود في الـ DT أو `board_regulator_init` | تحقق من اسم الـ supply في الـ DT، تأكد من تطابق `regulator-name` مع ما يمرره الـ driver |
| `Failed to enable vdd_cpu: -ENODEV` | الـ regulator مش مسجّل لما الـ consumer حاول يشغّله | probe ordering — استخدم `deferred probe` أو تأكد من ترتيب الـ `depends-on` في الـ DT |
| `regulator_enable: vdd_cpu already enabled` | double enable بدون تناظر مع disable | مراجعة منطق الـ enable/disable، التأكد إن كل `regulator_enable` ليه `regulator_disable` مقابلة |
| `regulator: Failed to get vdd_core -EPROBE_DEFER` | الـ PMIC driver ما اتحملش بعد | طبيعي في الـ boot — لو بيستمر راجع الـ PMIC driver initialization |
| `UNDER_VOLTAGE event on vdd_cpu` | الجهد وقع تحت الحد المسموح | فحص الـ power supply، الـ load الزيادة، إعدادات الـ regulator constraints |
| `OVER_CURRENT event on vdd_io` | التيار تعدى الحد الأقصى | فحص الدائرة عن short circuit، راجع `regulator_set_current_limit()` |
| `regulator_set_voltage: vdd_cpu invalid range` | الجهد المطلوب خارج الـ `min_uV/max_uV` المعرّفة | راجع الـ `regulator-min-microvolt/max-microvolt` في الـ DT |
| `OVER_TEMP event on pmic_ldo1` | الـ regulator وصل لدرجة حرارة عالية | فحص التبريد، خفّض الـ load |
| `regulator_set_load: vdd_cpu: set_load not supported` | الـ driver مش بيدعم DRMS | معلوماتي فقط — مش error حقيقي |
| `regulator_disable: vdd_cpu already disabled` | disable على regulator معطّل | bug في الـ consumer driver — راجع الـ enable_count في debugfs |

---

#### 8. مواضع الـ dump_stack() والـ WARN_ON()

**في الـ consumer driver** لو بتشك في order الـ enable/disable:

```c
int my_driver_probe(struct platform_device *pdev)
{
    struct regulator *reg = devm_regulator_get(&pdev->dev, "vdd");
    if (IS_ERR(reg)) {
        /* WARN بدل panic — يطبع stack trace ويكمل */
        dev_warn(&pdev->dev, "regulator not found: %ld\n", PTR_ERR(reg));
        return PTR_ERR(reg);
    }

    ret = regulator_enable(reg);
    WARN_ON(ret); /* يطبع stack trace لو enable فشل */
    return ret;
}
```

**لو بتحقق من الجهد بعد الـ enable:**

```c
    regulator_enable(reg);
    uV = regulator_get_voltage(reg);
    if (WARN_ON(uV < 900000 || uV > 1100000))
        dump_stack(); /* مكان إضافي للـ backtrace التفصيلي */
```

**نقطة استراتيجية في الـ notifier:**

```c
static int my_regulator_event(struct notifier_block *nb,
                               unsigned long event, void *data)
{
    if (event & REGULATOR_EVENT_UNDER_VOLTAGE) {
        pr_err("CRITICAL: under-voltage detected!\n");
        dump_stack();  /* اعرف مين كان شغال وقت الحدث */
    }
    if (event & REGULATOR_EVENT_FAIL) {
        WARN(1, "Regulator FAIL event — system may be unstable\n");
    }
    return NOTIFY_OK;
}
```

---

### Hardware Level

---

#### 1. التحقق من حالة الـ Hardware مقابل الـ Kernel State

```bash
# قارن الجهد اللي الـ kernel شايفه مع القراءة الحقيقية
cat /sys/class/regulator/regulator.N/microvolts
# قيس بالـ multimeter على الـ test point المقابل

# تحقق من الـ enable state
cat /sys/class/regulator/regulator.N/state
# قيس الجهد على الـ EN pin أو output الـ LDO مباشرة
```

لو الـ kernel بيقول "enabled" بس الجهد صفر على الـ board:
- احتمال مشكلة في الـ `enable GPIO` — راجع `regulator_enable_gpio`
- احتمال الـ I2C write للـ PMIC فشلت صامتة — فعّل `CONFIG_REGULATOR_DEBUG`

---

#### 2. تقنيات الـ Register Dump

للـ PMICs الـ I2C/SPI، ممكن تقرأ الـ registers مباشرة عبر الـ regmap debugfs:

```bash
# الـ regmap debugfs
ls /sys/kernel/debug/regmap/
# كل PMIC ليه entry باسمه مثلاً "0-0068" (I2C addr)

cat /sys/kernel/debug/regmap/0-0068/registers
# يطبع كل الـ registers بقيمها

# أو قرأ register بعينه
echo "0x1A" > /sys/kernel/debug/regmap/0-0068/address
cat /sys/kernel/debug/regmap/0-0068/val
```

لو الـ PMIC memory-mapped:

```bash
# استخدم devmem2 لقراءة register hardware مباشرة
# مثال: PMIC base address = 0x10200000، register offset = 0x20
devmem2 0x10200020 w

# أو عبر /dev/mem (يحتاج CONFIG_DEVMEM=y)
dd if=/dev/mem bs=4 count=1 skip=$((0x10200020/4)) 2>/dev/null | xxd
```

```bash
# لو الـ I2C PMIC، استخدم i2cdump
i2cdump -y 0 0x68          # dump كل registers
i2cget -y 0 0x68 0x1A b    # اقرأ register واحد
i2cset -y 0 0x68 0x1A 0x02 # اكتب register (احتياط!)
```

---

#### 3. الـ Logic Analyzer والـ Oscilloscope

**السيناريوهات المهمة للقياس:**

| ما تقيسه | نقطة القياس | المتوقع |
|---|---|---|
| تأكيد الـ enable | EN pin للـ LDO/DCDC | Rising edge عند `regulator_enable()` |
| وقت الـ ramp-up | Vout للـ regulator | من 0 للجهد الهدف — قارن مع `regulator-ramp-delay` في الـ DT |
| الـ I2C transactions | SCL/SDA | تحقق من ACK/NACK، ابحث عن retries |
| VSEL pin | VSEL GPIO | بيتغير عند `regulator_set_voltage()` على regulators بـ GPIO voltage select |
| تموج الجهد (ripple) | Vout | في وضع IDLE/STANDBY المفروض يزيد التموج |

**نصائح عملية:**

- حط trigger على الـ EN pin واقرأ trace من `regulator_enable` في ftrace في نفس الوقت — طابق الـ timestamps.
- لو الـ regulator بيتحكم فيه I2C، حط trigger على الـ I2C ACK الأول بعد address الـ PMIC.
- DCDC converters: قيس switching frequency — لو اتغيرت مفاجأة معناها الـ mode اتغير (NORMAL → IDLE).

---

#### 4. مشاكل الـ Hardware الشائعة وأنماط الـ Kernel Log

| المشكلة الهاردوير | النمط في الـ Kernel Log | ملاحظة |
|---|---|---|
| EN pin مش مرتبط صح | `regulator_get: failed to get supply` بعد DT probe | الـ GPIO مش موجود أو الـ polarity عكسية |
| PMIC لا يرد على I2C | `i2c i2c-0: ... no ack from device` ثم `Failed to enable` | بص على الـ pull-ups، الـ address، الـ power sequence |
| جهد الـ output أقل من المطلوب | `REGULATOR_EVENT_UNDER_VOLTAGE` في الـ log | سواء overcurrent أو مشكلة في الـ feedback network |
| الـ soft-start بطيء جداً | timeout في driver: `regulator enable timed out` | راجع `regulator-enable-ramp-delay` في الـ DT |
| الـ regulator شغال قبل الـ driver | `enable_count` في debugfs = 2 مع consumer واحد | `always-on` أو `boot-on` في الـ DT بيعمل pre-enable |
| PGOOD pin مش بيصحى | لا event ولا log — الـ consumer هانج | راجع interrupt routing للـ PGOOD |

---

#### 5. الـ Device Tree Debugging

```bash
# تحقق من الـ DT compiled
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "vdd_cpu"

# أو قرأ الـ DT مباشرة
cat /sys/firmware/devicetree/base/regulators/vdd_cpu/regulator-name 2>/dev/null

# تحقق من الـ constraints
cat /sys/firmware/devicetree/base/regulators/vdd_cpu/regulator-min-microvolt
cat /sys/firmware/devicetree/base/regulators/vdd_cpu/regulator-max-microvolt
cat /sys/firmware/devicetree/base/regulators/vdd_cpu/regulator-always-on
cat /sys/firmware/devicetree/base/regulators/vdd_cpu/regulator-boot-on
```

**أخطاء DT شائعة وتأثيرها:**

| خطأ في DT | التأثير في الـ Kernel |
|---|---|
| اسم supply في consumer مختلف عن اسم الـ regulator في DT | `supply not found` — `devm_regulator_get()` ترجع `-ENODEV` |
| `regulator-min-microvolt` أكبر من `regulator-max-microvolt` | kernel warning + رفض الـ set_voltage |
| نسيان `regulator-name` | الـ regulator بييجي باسم افتراضي — بيصعّب الـ debugfs |
| `enable-gpios` بـ polarity عكسية | الـ regulator بيتشغل عكس المتوقع |
| مفيش `vin-supply` للـ LDO | سلسلة الـ supply مكسورة — تحذيرات في الـ log |

```bash
# تحقق من الـ DT source ضد الـ hardware manual
# مثال: تحقق إن regulator-always-on موجود لـ rails اللازم تكون on دايماً
grep -r "regulator-always-on" /sys/firmware/devicetree/base/
```

---

### Practical Commands — جاهز للنسخ

#### فحص شامل سريع

```bash
#!/bin/bash
echo "=== Regulator Summary ==="
cat /sys/kernel/debug/regulator/regulator_summary

echo "=== All Regulator States ==="
for d in /sys/class/regulator/regulator.*/; do
    name=$(cat "$d/name" 2>/dev/null || echo "unknown")
    state=$(cat "$d/state" 2>/dev/null || echo "N/A")
    uV=$(cat "$d/microvolts" 2>/dev/null || echo "N/A")
    users=$(cat "$d/num_users" 2>/dev/null || echo "N/A")
    printf "%-20s state=%-10s voltage=%-10s users=%s\n" "$name" "$state" "$uV" "$users"
done
```

#### تفعيل الـ ftrace على كل events الـ regulator

```bash
cd /sys/kernel/tracing
echo 0 > tracing_on
echo "" > trace
for e in events/regulator/*/enable; do echo 1 > "$e"; done
echo 1 > tracing_on
sleep 5   # أو شغّل الـ test هنا
echo 0 > tracing_on
cat trace | grep -E "regulator_(enable|disable|set_voltage)"
```

#### تفعيل dynamic debug لكل الـ regulator core

```bash
echo "file drivers/regulator/core.c +pflt" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/regulator/helpers.c +pflt" >> /sys/kernel/debug/dynamic_debug/control
dmesg -w | grep regulator
```

#### فحص error flags بـ script

```bash
# يحتاج driver يدعم get_error_flags
# استخدم الـ sysfs status بديلاً
for d in /sys/class/regulator/regulator.*/; do
    status=$(cat "$d/status" 2>/dev/null)
    if [ "$status" != "ok" ] && [ -n "$status" ]; then
        name=$(cat "$d/name" 2>/dev/null)
        echo "ERROR: $name => $status"
    fi
done
```

#### I2C PMIC register dump كامل

```bash
# اكتشف I2C bus وaddress للـ PMIC
i2cdetect -y 0         # bus 0

# dump كل الـ registers
i2cdump -y 0 0x68 b    # PMIC على address 0x68

# قارن VSEL register بالجهد الـ kernel بيقوله
cat /sys/kernel/debug/regmap/0-0068/registers | grep -i vsel
cat /sys/class/regulator/regulator.0/microvolts
```

#### netlink listener لـ regulator events

```bash
# بـ Python3 لاستقبال REGULATOR_EVENT_* عبر Generic Netlink
python3 - <<'EOF'
import socket, struct
# REG_GENL_FAMILY_NAME = "reg_event"
# REG_GENL_MCAST_GROUP_NAME = "reg_mc_group"
# استخدم pyroute2 أو libnl للاشتراك
from pyroute2 import NetlinkSocket
# مثال بسيط — الـ implementation الكاملة يحتاج genl resolve
print("Listening for regulator events via Generic Netlink...")
print("Family: reg_event, Group: reg_mc_group")
EOF
```

#### كشف regulator probe failures في boot

```bash
dmesg | grep -iE "(regulator|supply).*(fail|error|defer|not found|-ENODEV|-EPROBE_DEFER)"
dmesg | grep -i "Failed to enable"
dmesg | grep -i "OVER_CURRENT\|UNDER_VOLTAGE\|OVER_TEMP\|FAIL"
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: UART مش شغال على بورد RK3562 — Industrial Gateway

#### العنوان
**الـ UART console مش بتظهر output على industrial gateway بعد تحديث الـ DT**

#### السياق
بتشتغل على gateway صناعي بيستخدم **RK3562** من Rockchip. الـ gateway بيتحكم في sensors عبر UART0. بعد merge لـ DT جديد، الـ UART console بطّلت تشتغل خالص من أول boot.

#### المشكلة
الـ UART driver بيعمل `probe` بدون error، بس مفيش أي output على الـ console. بتشوف في dmesg:

```
serial 2000000.serial: failed to enable regulator
```

الـ driver بيستخدم `devm_regulator_get_enable()` لـ power supply الخاصة بالـ UART transceiver (3.3V LDO).

#### التحليل
**الـ `devm_regulator_get_enable()`** (السطر 165 في consumer.h) بتعمل حاجتين في خطوة واحدة:
1. بتعمل `devm_regulator_get()` للـ supply باسم `"vdd"`
2. بتعمل `regulator_enable()` فوراً

```c
/* من consumer.h — السطر 165 */
int devm_regulator_get_enable(struct device *dev, const char *id);
```

لما DT عنده مشكلة في اسم الـ supply، الـ framework بيرجع `-ENODEV` أو `-EPROBE_DEFER`. الـ driver بيفسّر أي error على إنه failure كامل وبيوقف الـ probe.

التتبع:
```
devm_regulator_get_enable("vdd")
  → regulator_get(dev, "vdd")         ← بيدور على "vdd-supply" في DT
  → ERR_PTR(-ENODEV)                  ← مش لاقيه في DT node
  → driver probe returns -ENODEV
  → UART device not registered
```

في الـ DT القديم كان في:
```dts
&uart0 {
    vdd-supply = <&ldo_3v3>;
};
```
بعد الـ merge اتمسح السطر ده عن طريق خطأ.

#### الحل

**أولاً:** تحقق من الـ DT:
```bash
# على الـ target
cat /proc/device-tree/serial@2000000/vdd-supply
# أو
dtc -I fs /proc/device-tree | grep -A5 "serial@2000000"
```

**تانياً:** رجّع السطر المحذوف في الـ DT:
```dts
&uart0 {
    status = "okay";
    vdd-supply = <&ldo_3v3>;   /* 3.3V for RS485 transceiver */
};
```

**تالتاً:** لو الـ supply اختياري وممكن يكون مش موجود، استخدم `devm_regulator_get_enable_optional()` بدل `devm_regulator_get_enable()`:

```c
/* في الـ driver: لو الـ supply مش إجباري */
ret = devm_regulator_get_enable_optional(dev, "vdd");
if (ret && ret != -ENODEV)
    return ret;
```

#### الدرس المستفاد
**الـ `devm_regulator_get_enable()`** بتفشّل الـ probe لو الـ supply مش موجود في DT. لو الـ hardware ممكن يشتغل بدون supply control (زي لو فيه power always-on)، استخدم `_optional` variant. دايماً check الـ DT supply names بعد أي merge.

---

### السيناريو الثاني: SPI Flash مش بيتعرف على STM32MP1 — IoT Sensor Node

#### العنوان
**الـ SPI NOR flash بيرجع garbage data على STM32MP1 بسبب voltage mismatch**

#### السياق
بورد IoT sensor بيستخدم **STM32MP1** مع W25Q128 SPI NOR flash. الـ flash بيحتاج VCC = 3.3V بالظبط. في الـ production run الجديد، تم تغيير الـ PMIC من ST33 إلى ST34 مع default voltage مختلف.

#### المشكلة
الـ flash بيـ enumerate بس الـ read data كلها 0xFF أو garbage. الـ `mtd` device موجود، بس أي flash read بيفشل. الـ kernel log بيقول:

```
m25p80 spi0.0: unrecognized JEDEC id bytes: ff ff ff
```

#### التحليل
الـ SPI flash driver بيستخدم `regulator_get_voltage()` لـ verification، وبيستخدم `regulator_set_voltage()` لو احتاج تعديل:

```c
/* من consumer.h — السطر 231 */
int regulator_get_voltage(struct regulator *regulator);

/* من consumer.h — السطر 228 */
int regulator_set_voltage(struct regulator *regulator, int min_uV, int max_uV);
```

السيناريو:
```
regulator_get_voltage(vcc)   → returns 1800000 (1.8V)  ← wrong!
W25Q128 needs 2.7V - 3.6V
→ SPI signals at wrong voltage levels
→ flash returns all 0xFF
```

الـ PMIC الجديد عنده default output = 1.8V بدل 3.3V. الـ DT عنده:

```dts
&spi_flash_reg {
    regulator-min-microvolt = <3300000>;
    regulator-max-microvolt = <3300000>;
};
```

بس الـ `regulator_set_voltage()` مش بيتكالم في الـ probe لأن الـ driver افترض إن الـ PMIC هيديله الـ default voltage الصح.

#### الحل

**أولاً:** تحقق من الـ voltage الفعلي:
```bash
# على الـ target
cat /sys/class/regulator/regulator.X/microvolts
# أو
grep -r "" /sys/class/regulator/*/name
```

**تانياً:** أضف explicit voltage set في الـ driver أو في الـ DT باستخدام `regulator-always-on` مع الـ voltage:

```c
/* في الـ SPI driver probe */
struct regulator *vcc = devm_regulator_get(dev, "vcc");
if (IS_ERR(vcc))
    return PTR_ERR(vcc);

/* Explicitly set voltage before enable */
ret = regulator_set_voltage(vcc, 3300000, 3300000);
if (ret) {
    dev_err(dev, "failed to set VCC to 3.3V: %d\n", ret);
    return ret;
}

ret = regulator_enable(vcc);
```

**تالتاً:** في الـ DT، تأكد من الـ regulator constraints:
```dts
&ldo4 {
    regulator-name = "vcc-spi-flash";
    regulator-min-microvolt = <3300000>;
    regulator-max-microvolt = <3300000>;
    regulator-boot-on;
    regulator-always-on;
};
```

#### الدرس المستفاد
عند تغيير الـ PMIC، الـ default voltages ممكن تختلف. دايماً call `regulator_set_voltage()` explicitly قبل `regulator_enable()` في الـ driver، ومتعتمدش على الـ PMIC default خصوصاً في الـ production board changes.

---

### السيناريو الثالث: I2C HDMI Bridge بيسحب over-current على i.MX8 — Android TV Box

#### العنوان
**الـ HDMI output بتوقف فجأة مع over-current warning على i.MX8MQ TV Box**

#### السياق
Android TV box مبني على **i.MX8MQ** مع IT6263 HDMI bridge متوصل بـ I2C. الـ bridge بياخد power من LDO regulator. في الـ runtime، بعد ربع ساعة من الـ playback، الـ HDMI بيتقطع. الـ dmesg بيظهر:

```
regulator-1: REGULATOR_EVENT_OVER_CURRENT
hdmi-bridge: disabling due to over-current condition
```

#### التحليل
الـ HDMI bridge driver بيستخدم **`regulator_register_notifier()`** عشان يتلقى events من الـ regulator framework:

```c
/* من consumer.h — السطر 259-266 */
int regulator_register_notifier(struct regulator *regulator,
                              struct notifier_block *nb);
int regulator_unregister_notifier(struct regulator *regulator,
                                struct notifier_block *nb);
```

الـ notifier callback بيشوف `REGULATOR_EVENT_OVER_CURRENT` (السطر 0x02 في regulator.h)، وبيعمل `regulator_disable()` حمايةً للـ hardware.

السبب الفعلي: الـ TV box بيشغّل 4K HDR content، الـ IT6263 بيرفع الـ current consumption لـ 850mA. الـ LDO current limit معمول على 800mA.

```c
/* error flag check */
regulator_get_error_flags(regulator, &flags);
/* flags & REGULATOR_ERROR_OVER_CURRENT → BIT(2) */
```

#### الحل

**أولاً:** تشوف الـ current limit الحالي:
```bash
cat /sys/class/regulator/regulator.X/requested_microamps
cat /sys/class/regulator/regulator.X/microamps
```

**تانياً:** استخدم `regulator_set_current_limit()` برفع الـ limit:

```c
/* في الـ driver probe */
ret = regulator_set_current_limit(hdmi_supply, 500000, 1000000);
/* min=500mA, max=1A — يديه مرونة */
if (ret)
    dev_warn(dev, "could not set current limit: %d\n", ret);
```

**تالتاً:** في DT، عدّل الـ regulator constraints:
```dts
&ldo_hdmi {
    regulator-name = "hdmi-bridge-vcc";
    regulator-min-microvolt = <1800000>;
    regulator-max-microvolt = <1800000>;
    regulator-min-microamp  = <500000>;
    regulator-max-microamp  = <1200000>;  /* was 800000 */
};
```

**رابعاً:** في الـ notifier، بدل الـ hard disable، حاول تعمل recovery:

```c
static int hdmi_regulator_event(struct notifier_block *nb,
                                unsigned long event, void *data)
{
    if (event & REGULATOR_EVENT_OVER_CURRENT) {
        /* log and schedule recovery, don't hard-disable */
        schedule_delayed_work(&priv->recovery_work, HZ);
        return NOTIFY_OK;
    }
    if (event & REGULATOR_EVENT_FAIL) {
        /* hard failure — disable */
        regulator_disable(priv->supply);
        return NOTIFY_OK;
    }
    return NOTIFY_DONE;
}
```

#### الدرس المستفاد
الـ `REGULATOR_ERROR_OVER_CURRENT` و `REGULATOR_EVENT_OVER_CURRENT` مختلفان — الأول للـ polling عبر `regulator_get_error_flags()`، التاني للـ async notification. عند design الـ HDMI/display drivers على Android TV boxes، احسب الـ peak current للـ 4K HDR وحطّه في الـ constraints مش الـ typical current.

---

### السيناريو الرابع: CPU DVFS بيتعلق على AM62x — Automotive ECU

#### العنوان
**الـ CPU frequency scaling بيتفريز عند الـ OPP transitions على AM62x في automotive ECU**

#### السياق
ECU للسيارة مبني على **TI AM62x** (Cortex-A53). الـ ECU بيشتغل مع DVFS (Dynamic Voltage and Frequency Scaling) للـ power management. في الـ stress test، لما الـ CPU بيحاول ينتقل من 800MHz إلى 1.4GHz، الـ system بيتفريز لثانية كاملة.

#### المشكلة
الـ cpufreq driver بيستخدم `regulator_set_voltage_time()` عشان يحسب وقت الـ transition:

```c
/* من consumer.h — السطر 229-230 */
int regulator_set_voltage_time(struct regulator *regulator,
                               int old_uV, int new_uV);
```

الـ function بترجع الـ time بالـ microseconds اللي المفروض ينتظرها الـ driver بعد الـ voltage change قبل ما يرفع الـ frequency.

#### التحليل

الـ PMIC المستخدم (TPS6521905) عنده `ramp_delay` معمول على 500 µV/µs في الـ regulator driver. الـ transition من 850mV إلى 1050mV = 200mV:

```
time = ΔV / ramp_rate = 200,000 µV / 500 µV/µs = 400 µs
```

بس الـ driver كان بيعمل `udelay(400)` = 400 µs صح. المشكلة إن الـ `regulator_set_voltage()` نفسها بتاخد وقت لأن الـ PMIC على I2C @ 100kHz:

```
I2C transaction time ≈ 9 bits × 3 bytes / 100kHz ≈ 2.7ms
+ ramp time 400µs
Total ≈ 3.1ms
```

بس الـ driver كان بيعمل الـ delay قبل الـ I2C transaction تخلص، فالـ CPU بيشتغل بـ voltage غلط لـ 2.7ms، وده بيسبب instability وأحياناً lock-up.

للتحقق:
```bash
# check ramp delay
cat /sys/class/regulator/regulator.X/microvolts
# قبل وبعد OPP switch — قيس الـ timing بـ oscilloscope على PMIC EN pin
```

#### الحل

**أولاً:** استخدم `regulator_set_voltage_time()` بشكل صح — انتظر بعد الـ `regulator_set_voltage()` تخلص:

```c
int old_uV = regulator_get_voltage(cpu_reg);

/* first: set the voltage (this includes I2C time) */
ret = regulator_set_voltage(cpu_reg, new_uV, new_uV);
if (ret)
    return ret;

/* then: get the settling time and wait */
delay = regulator_set_voltage_time(cpu_reg, old_uV, new_uV);
if (delay > 0)
    udelay(delay);

/* now safe to change frequency */
clk_set_rate(cpu_clk, new_freq);
```

**تانياً:** لو الـ cpufreq-dt بيستخدم `regulator_set_voltage_triplet()` (السطر 717):

```c
/* tries target_uV first, falls back to min_uV range */
ret = regulator_set_voltage_triplet(cpu_reg,
                                    min_uV,    /* floor */
                                    target_uV, /* preferred */
                                    max_uV);   /* ceiling */
```

**تالتاً:** رفع سرعة الـ I2C للـ PMIC لو الـ hardware يسمح:
```dts
&i2c0 {
    clock-frequency = <400000>;  /* Fast mode instead of 100kHz */
};
```

#### الدرس المستفاد
الـ `regulator_set_voltage_time()` بتحسب فقط وقت الـ ramp الداخلي للـ PMIC — مش وقت الـ I2C communication. في الـ DVFS على automotive ECUs، لازم تاخد في الاعتبار الـ total latency = (I2C time) + (ramp time). في الـ safety-critical systems، استخدم fast I2C أو SPI PMIC لتقليل الـ transition time.

---

### السيناريو الخامس: الـ suspend/resume بيعمل crash على Allwinner H616 — Android TV Stick

#### العنوان
**الـ system يعمل kernel panic عند الـ resume من suspend على Allwinner H616 TV Stick**

#### السياق
Android TV stick رخيص مبني على **Allwinner H616**. عند الـ screen-off وبعد الـ suspend timeout، الـ system بيحاول يعمل resume بس بيـ crash بـ kernel panic:

```
Unable to handle kernel NULL pointer dereference at 0x00000000
PC is at regulator_sync_voltage+0x24
```

#### المشكلة
الـ display driver بيستخدم `regulator_set_suspend_voltage()` عشان يحط الـ DDR supply في low-power voltage أثناء الـ suspend، وعند الـ resume بيستخدم `regulator_sync_voltage()`:

```c
/* من consumer.h — السطر 273-274 */
int regulator_set_suspend_voltage(struct regulator *regulator, int min_uV,
                                  int max_uV, suspend_state_t state);

/* من consumer.h — السطر 232 */
int regulator_sync_voltage(struct regulator *regulator);
```

#### التحليل

الـ crash بييجي من إن الـ `struct regulator *` pointer بقى dangling. التسلسل:

```
suspend path:
  display_suspend()
    → regulator_set_suspend_voltage(ddr_reg, 900000, 900000, PM_SUSPEND_MEM)
    → regulator_disable(ddr_reg)         ← usage count decrements
    → devm_regulator_put(ddr_reg)        ← BUG: called manually + devm!

resume path:
  display_resume()
    → regulator_sync_voltage(ddr_reg)    ← ddr_reg is dangling pointer → PANIC
```

المشكلة إن الـ driver بيخلط بين `devm_regulator_get()` (اللي بتتحرر أوتوماتيك) وبين manual `regulator_put()`. لما بيستخدم `devm_regulator_get()` وبعدين بيعمل `devm_regulator_put()` يدوياً، الـ pointer بيتحرر مرتين — مرة يدوي ومرة أوتوماتيك عند الـ device cleanup.

```c
/* consumer.h — السطر 155 */
struct regulator *__must_check devm_regulator_get(struct device *dev,
                                                  const char *id);
/* consumer.h — السطر 169 */
void devm_regulator_put(struct regulator *regulator);
```

الـ `devm_*` variants مش محتاجين manual put — الـ framework بيعمله تلقائياً عند الـ device removal.

#### الحل

**أولاً:** امسح الـ manual `devm_regulator_put()` من الـ suspend path:

```c
/* WRONG — don't mix devm_get with manual put */
static int display_suspend(struct device *dev)
{
    struct display_priv *priv = dev_get_drvdata(dev);

    regulator_set_suspend_voltage(priv->ddr_reg,
                                  900000, 900000,
                                  PM_SUSPEND_MEM);
    regulator_disable(priv->ddr_reg);
    devm_regulator_put(priv->ddr_reg);   /* BUG: remove this line */
    return 0;
}

/* CORRECT */
static int display_suspend(struct device *dev)
{
    struct display_priv *priv = dev_get_drvdata(dev);

    /* just set suspend voltage and disable — devm handles lifetime */
    regulator_set_suspend_voltage(priv->ddr_reg,
                                  900000, 900000,
                                  PM_SUSPEND_MEM);
    regulator_disable(priv->ddr_reg);
    return 0;
}

static int display_resume(struct device *dev)
{
    struct display_priv *priv = dev_get_drvdata(dev);

    regulator_enable(priv->ddr_reg);
    /* sync to make sure hardware reflects current software state */
    regulator_sync_voltage(priv->ddr_reg);
    return 0;
}
```

**تانياً:** لو عاوز تتحكم يدوياً في الـ lifetime، استخدم `regulator_get()` مع `regulator_put()` (مش devm variants):

```c
/* probe */
priv->ddr_reg = regulator_get(dev, "ddr-vcc");

/* remove */
regulator_put(priv->ddr_reg);
```

**تالتاً:** للـ debug، enable الـ regulator tracing:
```bash
echo 1 > /sys/kernel/debug/tracing/events/regulator/enable
cat /sys/kernel/debug/tracing/trace
```

#### الدرس المستفاد
**الـ `devm_regulator_get()`** و **`regulator_get()`** مش قابلين للتبديل مع الـ put counterparts. `devm_*` الـ framework مسؤول عن الـ cleanup — ممنوع تعمل manual put. خلط الـ two ownership models أكثر سبب شائع للـ use-after-free bugs في الـ regulator consumers، خصوصاً في الـ suspend/resume paths على الـ budget Android devices.
## Phase 7: مصادر ومراجع

### التوثيق الرسمي في الـ Kernel

| المسار | المحتوى |
|--------|---------|
| `Documentation/power/regulator/consumer.rst` | واجهة الـ consumer driver — الأساس |
| `Documentation/power/regulator/overview.rst` | نظرة عامة على الـ regulator framework |
| `Documentation/power/regulator/regulator.rst` | واجهة الـ regulator driver |
| `Documentation/power/regulator/machine.rst` | واجهة الـ machine driver |
| `Documentation/driver-api/regulator.rst` | الـ driver API المدمج |
| `include/linux/regulator/consumer.h` | الملف المصدر لهذا التوثيق |

**روابط مباشرة:**

- [Regulator Consumer Driver Interface — docs.kernel.org](https://docs.kernel.org/power/regulator/consumer.html)
- [Linux voltage and current regulator framework — docs.kernel.org](https://docs.kernel.org/power/regulator/overview.html)
- [Voltage and current regulator API — docs.kernel.org](https://docs.kernel.org/driver-api/regulator.html)

---

### مقالات LWN.net

الـ [LWN.net](https://lwn.net) هو المرجع الأول لمتابعة تطور الـ kernel.

| المقال | الأهمية |
|--------|---------|
| [regulator: Add regulator_get_exclusive() API](https://lwn.net/Articles/342907/) | إضافة API للـ exclusive access — مهم لفهم سلوك الـ consumer |
| [Device tree support for regulators](https://lwn.net/Articles/462397/) | كيف الـ device tree غيّر طريقة ربط الـ consumer بالـ supply |
| [Devm helpers for regulator get and enable](https://lwn.net/Articles/904557/) | إضافة `devm_regulator_get_enable()` — تبسيط كبير للـ consumer drivers |
| [Add coupled regulators mechanism](https://lwn.net/Articles/752549/) | الـ coupled regulators وتأثيرها على الـ consumer API |
| [Extend regulator notification support](https://lwn.net/Articles/852636/) | توسيع نظام الـ notifications للـ consumers |
| [Runtime power management](https://lwn.net/Articles/347573/) | السياق الأشمل للـ power management في الـ kernel |
| [Active state management of power domains](https://lwn.net/Articles/744047/) | علاقة الـ power domains بالـ regulators |
| [Voltage and current regulator API — static.lwn.net](https://static.lwn.net/kerneldoc/driver-api/regulator.html) | نسخة LWN من الـ kernel docs |

---

### مصادر eLinux.org

- [Regulators: Learning To Play With Others — PDF](https://elinux.org/images/d/da/Regulators-_Learning_To_Play_With_Others.pdf) — عرض تقديمي ممتاز يشرح الـ framework من الناحية العملية على الـ embedded systems
- [Debugging Embedded Linux Kernel Power Management — PDF](https://www.elinux.org/images/1/13/Debugging_Embedded_Linux_(Kernel)_Power_Management.pdf) — debugging الـ regulator consumers في الأنظمة المدمجة

---

### kernelnewbies.org

| الصفحة | ما تجد فيها |
|--------|------------|
| [Linux 2.6.31 — kernelnewbies.org](https://kernelnewbies.org/Linux_2_6_31) | أول إضافة لـ userspace-consumer driver |
| [Linux 3.15 DriversArch](https://kernelnewbies.org/Linux_3.15-DriversArch) | تغييرات الـ regulator subsystem في 3.15 |
| [Linux 3.17 DriversArch](https://kernelnewbies.org/Linux_3.17-DriversArch) | تغييرات في 3.17 |
| [Linux 3.18 DriversArch](https://kernelnewbies.org/Linux_3.18-DriversArch) | تغييرات في 3.18 |
| [Linux Regulator userspace-consumer driver binding — mailing list](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2016-June/016470.html) | نقاش عملي على الـ mailing list |

---

### مصادر خارجية مهمة

**الـ original design document من Liam Girdwood:**

- [Regulator Control Interface for the Linux Kernel — Liam Girdwood (CELF 2008)](https://www.celinux.org/elc08_presentations/regulator-api-celf.pdf) — العرض الأصلي لتصميم الـ framework، مرجع تاريخي أساسي

**Wolfson/SlimLogic:**

- [Linux Voltage & Current Regulator Subsystem — slimlogic.co.uk](https://slimlogic.co.uk/2009/02/linux-voltage-current-regulator-subsystem/) — مقال Liam Girdwood عن الـ subsystem

**توثيق kernel.org الرسمي:**

- [consumer.rst — kernel.org](https://www.kernel.org/doc/Documentation/power/regulator/consumer.rst)
- [regulator.rst — kernel.org](https://www.kernel.org/doc/Documentation/driver-api/regulator.rst)

---

### Commits مهمة في الـ Git History

للبحث في تاريخ الـ commits، استخدم:

```bash
# تاريخ الـ consumer.h
git log --oneline include/linux/regulator/consumer.h

# commits المتعلقة بـ devm_regulator_get_enable
git log --oneline --grep="devm_regulator_get_enable"

# commits المتعلقة بـ bulk API
git log --oneline --grep="regulator_bulk"

# commits المتعلقة بـ exclusive get
git log --oneline --grep="regulator_get_exclusive"
```

**commits محورية يمكن البحث عنها:**

| الميزة | كيف تبحث عنها |
|--------|--------------|
| أول إضافة للـ framework (2.6.26) | `git log --grep="regulator: Add regulator framework"` |
| إضافة `regulator_get_exclusive` | [LWN patch](https://lwn.net/Articles/342907/) |
| إضافة `devm_regulator_get_enable` | [LWN patch series](https://lwn.net/Articles/904557/) |
| دعم الـ device tree | [LWN article](https://lwn.net/Articles/462397/) |

---

### نقاشات الـ Mailing List

- **linux-regulator list:** `lore.kernel.org/linux-regulator/`
- **بحث مباشر:** [lore.kernel.org/all/?q=regulator+consumer](https://lore.kernel.org/all/?q=regulator+consumer)
- **نقاش قديم مهم:** [PATCH: regulator: Provide optional dummy regulator for consumers](https://linux-kernel.vger.kernel.narkive.com/88i49VMt/patch-regulator-provide-optional-dummy-regulator-for-consumers)
- **نقاش devm APIs:** [PATCH: regulator: Add devm_regulator_get()](https://linux.kernel.narkive.com/iqcXfWgH/patch-regulator-add-devm-regulator-get)

---

### كتب مرجعية

#### Linux Device Drivers, 3rd Edition (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصل:** Chapter 14 — The Linux Device Model
- **ملاحظة:** الكتاب قديم (2005) ولا يغطي الـ regulator framework بشكل مباشر، لكنه أساسي لفهم الـ device model الذي يبني عليه الـ regulator
- **متاح مجاناً:** [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development, 3rd Edition — Robert Love
- **الفصل ذو الصلة:** Chapter 11 — Timers and Time Management، وChapter 14 — The Block I/O Layer (للسياق)
- **الأهمية:** يشرح الـ kernel internals اللي يبنى فوقها الـ regulator framework (notifiers، devm، reference counting)

#### Embedded Linux Primer, 2nd Edition — Christopher Hallinan
- **الفصل:** Chapter 15 — Embedded Linux Power Management
- **الأهمية:** يغطي الـ regulator framework في سياق الـ embedded systems والـ SoC بشكل عملي

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- **الأهمية:** يغطي الـ power management subsystem بعمق أكبر من LDD3

---

### مصطلحات البحث

للبحث عن معلومات إضافية استخدم الـ keywords دي:

```
# بحث عام
linux kernel regulator consumer API
linux regulator framework driver development

# بحث متخصص
regulator_get_exclusive linux
devm_regulator_get_enable linux kernel
regulator_bulk_get example driver
linux DRMS Dynamic Regulator Mode Switching
regulator notifier block linux
regulator set_voltage uV linux driver

# بحث في mailing list
site:lore.kernel.org regulator consumer
site:lkml.org regulator_get_exclusive

# بحث في الكود
site:elixir.bootlin.com regulator_get
site:github.com torvalds linux consumer.h
```

**الـ Bootlin Elixir Cross-Referencer** مفيد جداً لتتبع استخدامات الـ API في الـ kernel:

- [regulator_get على Bootlin Elixir](https://elixir.bootlin.com/linux/latest/ident/regulator_get)
- [devm_regulator_get_enable على Bootlin Elixir](https://elixir.bootlin.com/linux/latest/ident/devm_regulator_get_enable)
- [regulator_bulk_get على Bootlin Elixir](https://elixir.bootlin.com/linux/latest/ident/regulator_bulk_get)

---

### خريطة المراجع حسب المستوى

```
المبتدئ
  └─► kernelnewbies.org + LDD3 chapter 14
  └─► docs.kernel.org/power/regulator/consumer.html

المتوسط
  └─► LWN.net articles (الروابط أعلاه)
  └─► Documentation/power/regulator/ كاملة
  └─► elinux.org PDF presentations

المتقدم
  └─► lore.kernel.org/linux-regulator/ (patch discussions)
  └─► git log على include/linux/regulator/consumer.h
  └─► drivers/regulator/core.c المصدر مباشرة
  └─► Embedded Linux Primer ch.15
```
## Phase 8: Writing simple module

### الفكرة

**الـ** `regulator_register_notifier()` بتخلّينا نتلقى أحداث زي voltage change وover-current وover-temp على أي regulator موجود في النظام. ده من أكتر الأشياء اللي بتحصل في embedded Linux بشكل صامت — بنكتب module يكشف الأحداث دي ويطبعها.

المثال ده بيعمل hook على regulator اسمه `"vdd_cpu"` (شائع في SoCs زي Raspberry Pi وi.MX) ويطبع نوع الحدث والجهد القديم لما يحصل `PRE_VOLTAGE_CHANGE`.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * regulator_notifier_demo.c
 *
 * Hooks into a regulator's notifier chain and logs voltage-change
 * and fault events via pr_info().
 *
 * Tested concept: regulator_register_notifier() /
 *                 regulator_unregister_notifier()
 *
 * Build: add to a Kbuild with obj-m := regulator_notifier_demo.o
 */

#include <linux/module.h>       /* MODULE_* macros, module_init/exit       */
#include <linux/kernel.h>       /* pr_info(), pr_err()                     */
#include <linux/notifier.h>     /* notifier_block, notifier chain API      */
#include <linux/regulator/consumer.h>  /* regulator_get/put, register_notifier */
#include <uapi/regulator/regulator.h>  /* REGULATOR_EVENT_* constants       */

/* Name of the regulator we want to observe — change to match your board */
#define TARGET_REGULATOR  "vdd_cpu"

/* We hold a reference so we can unregister in exit */
static struct regulator *watched_reg;

/* ------------------------------------------------------------------ */
/*  Notifier callback                                                  */
/* ------------------------------------------------------------------ */
static int reg_event_handler(struct notifier_block *nb,
                             unsigned long event,
                             void *data)
{
    /* event is a bitmask of REGULATOR_EVENT_* values */
    if (event & REGULATOR_EVENT_PRE_VOLTAGE_CHANGE) {
        struct pre_voltage_change_data *vd = data;
        /*
         * data points to pre_voltage_change_data when this event fires;
         * it carries the old voltage and the requested [min, max] range.
         */
        pr_info("reg_demo: PRE_VOLTAGE_CHANGE  old=%lu uV  new=[%lu, %lu] uV\n",
                vd->old_uV, vd->min_uV, vd->max_uV);
    }

    if (event & REGULATOR_EVENT_VOLTAGE_CHANGE) {
        /* data is the old voltage cast to (void *) — not a pointer to struct */
        unsigned long old_uV = (unsigned long)data;
        pr_info("reg_demo: VOLTAGE_CHANGE done  old=%lu uV\n", old_uV);
    }

    if (event & REGULATOR_EVENT_OVER_CURRENT)
        pr_info("reg_demo: WARNING — OVER_CURRENT detected!\n");

    if (event & REGULATOR_EVENT_OVER_TEMP)
        pr_info("reg_demo: WARNING — OVER_TEMP detected!\n");

    if (event & REGULATOR_EVENT_FAIL)
        pr_info("reg_demo: ERROR — regulator FAIL event!\n");

    if (event & REGULATOR_EVENT_ENABLE)
        pr_info("reg_demo: regulator ENABLED\n");

    if (event & REGULATOR_EVENT_DISABLE)
        pr_info("reg_demo: regulator DISABLED\n");

    /* NOTIFY_OK tells the chain: handled, continue to next listener */
    return NOTIFY_OK;
}

/* The notifier_block wires our function into the regulator's chain */
static struct notifier_block reg_nb = {
    .notifier_call = reg_event_handler,
};

/* ------------------------------------------------------------------ */
/*  init / exit                                                        */
/* ------------------------------------------------------------------ */
static int __init reg_demo_init(void)
{
    int ret;

    /*
     * regulator_get() takes a reference to the named supply.
     * We pass NULL as device because this is a standalone module,
     * not a real driver with a struct device.
     * On a real driver use devm_regulator_get(dev, id) instead.
     */
    watched_reg = regulator_get(NULL, TARGET_REGULATOR);
    if (IS_ERR(watched_reg)) {
        pr_err("reg_demo: cannot get regulator '%s': %ld\n",
               TARGET_REGULATOR, PTR_ERR(watched_reg));
        return PTR_ERR(watched_reg);
    }

    /*
     * Register our notifier_block on the regulator's notifier chain.
     * From this point on, every event fires reg_event_handler().
     */
    ret = regulator_register_notifier(watched_reg, &reg_nb);
    if (ret) {
        pr_err("reg_demo: register_notifier failed: %d\n", ret);
        regulator_put(watched_reg);
        return ret;
    }

    pr_info("reg_demo: watching regulator '%s'\n", TARGET_REGULATOR);
    return 0;
}

static void __exit reg_demo_exit(void)
{
    /*
     * Must unregister BEFORE releasing the regulator reference.
     * If we skip this, the notifier chain still holds a pointer to
     * reg_nb which lives in our module memory — use-after-free on
     * the next event after rmmod.
     */
    regulator_unregister_notifier(watched_reg, &reg_nb);

    /*
     * Drop the reference obtained in init.
     * After this the kernel may power-gate the supply if no other
     * consumer holds it.
     */
    regulator_put(watched_reg);

    pr_info("reg_demo: unloaded\n");
}

module_init(reg_demo_init);
module_exit(reg_demo_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Doc Demo");
MODULE_DESCRIPTION("Logs regulator events via notifier chain");
```

---

### شرح كل جزء

#### الـ includes

```c
#include <linux/regulator/consumer.h>
#include <uapi/regulator/regulator.h>
```

**الـ** `consumer.h` بيجيب `regulator_get/put` وكمان `register_notifier` — ده الـ API اللي الـ consumer بيشوفه.
**الـ** `uapi/regulator/regulator.h` بيجيب الـ `REGULATOR_EVENT_*` constants وـ`struct pre_voltage_change_data` — بدونه مش هنعرف نفسر الـ event bitmask.

---

#### الـ `notifier_block`

```c
static struct notifier_block reg_nb = {
    .notifier_call = reg_event_handler,
};
```

ده بيعمل entry في الـ linked-list بتاعة الـ regulator's notifier chain.
كل regulator عنده chain خاصة بيه — لو سجّلت في `vdd_cpu` مش هتتأثر بأحداث `vdd_io`.

---

#### الـ callback وقراءة `data`

```c
if (event & REGULATOR_EVENT_PRE_VOLTAGE_CHANGE) {
    struct pre_voltage_change_data *vd = data;
    ...
}
if (event & REGULATOR_EVENT_VOLTAGE_CHANGE) {
    unsigned long old_uV = (unsigned long)data;
    ...
}
```

الـ `data` pointer مش نوعه ثابت — بيتغير حسب الـ event:

| Event | نوع `data` |
|---|---|
| `PRE_VOLTAGE_CHANGE` | `struct pre_voltage_change_data *` |
| `VOLTAGE_CHANGE` | `unsigned long` (old voltage) cast to `void *` |
| باقي الأحداث | `NULL` عادةً |

لازم تعرف النوع الصح من documentation الـ event — أي cast غلط = kernel oops.

---

#### الـ `regulator_get(NULL, ...)` في standalone module

في driver حقيقي بنعدي `dev` عشان الـ framework يعرف يربط الـ supply بالـ device tree node.
هنا بنعدي `NULL` لأن المقصود هو demonstration — في production استخدم `devm_regulator_get(dev, id)` عشان الـ cleanup يتعمل أوتوماتيك لما الـ device يتـunbind.

---

#### الـ `module_exit` وترتيب الـ cleanup

```c
regulator_unregister_notifier(watched_reg, &reg_nb);
regulator_put(watched_reg);
```

الترتيب ده مهم جداً:
لو عملنا `regulator_put` الأول، الـ `reg_nb` ممكن يتـfire على حدث اللحظة دي ونظام يحاول يدخل `reg_event_handler` بعد ما الـ module اتشيل من الذاكرة — **use-after-free**.
لازم Unregister أولاً عشان نضمن إن مفيش حدث جديد هيجي بعد كده.

---

### كيفية البناء والتشغيل

```bash
# Kbuild
obj-m := regulator_notifier_demo.o

# Build
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# Load
sudo insmod regulator_notifier_demo.ko

# Watch output
sudo dmesg -w | grep reg_demo

# Trigger a voltage change (example: cpufreq scaling)
echo performance > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# Unload
sudo rmmod regulator_notifier_demo
```

**الـ** `dmesg` هيطبع كل حدث بالجهد القديم والجديد في real-time.
على boards زي i.MX8 أو RK3588، الـ `vdd_cpu` بيتغير مئات المرات في الثانية مع cpufreq — الـ module هيكشف ده بوضوح.
