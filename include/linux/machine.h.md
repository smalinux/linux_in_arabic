## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ Regulator Framework؟

تخيّل إن عندك بيت فيه أجهزة كتير: تليفزيون، لابتوب، موبايل بيتشارج، راوتر. كل جهاز بيحتاج فولت مختلف — التليفزيون 12V، الموبايل 5V، الراوتر 9V. الكهرباء اللي جاية من الشبكة 220V. يعني لازم يكون في حاجة "تنظّم" الكهرباء وتوصّلها صح لكل جهاز.

في الـ SoC (System on Chip) — زي اللي في موبايلك — نفس المشكلة بالظبط. عندك CPU بيشتغل على 1.0V، RAM على 1.8V، WiFi chip على 3.3V، وكلهم داخل نفس الجهاز. الـ **PMIC (Power Management IC)** هو اللي بيعمل ده — بيجيب الكهرباء من البطارية أو الشبكة وبيقسّمها وينظّمها لكل مكون.

الـ **Voltage and Current Regulator Framework** في Linux Kernel هو الطبقة اللي بتتحكم في كل ده من ناحية البرنامج.

---

### الـ machine.h — إيه دورها؟

الـ framework عنده 3 وجهات نظر (perspectives):

| الطرف | الهيدر بتاعه | دوره |
|---|---|---|
| **Driver** (اللي بيشغّل الـ PMIC) | `driver.h` | بيعرّف ops زي `set_voltage`, `enable` |
| **Consumer** (الجهاز اللي محتاج الكهرباء) | `consumer.h` | بيطلب `regulator_get()`, `regulator_enable()` |
| **Machine/Board** (الـ platform setup) | `machine.h` | بيحدد القيود والحدود لكل الـ board |

**الـ `machine.h` هي واجهة الـ board/platform** — يعني بتقول للـ kernel: "في البورد دي، الـ CPU regulator ممنوع ينزل عن 900mV وممنوع يعلى عن 1200mV، ولازم يفضل شغّال حتى لو ما فيش حد طلبه."

---

### القصة الكاملة

#### المشكلة

عندك بورد ARM جديدة — مثلاً Raspberry Pi أو موبايل Samsung. المهندس بتاع الـ hardware حدد إن:

- الـ CPU voltage لازم يكون بين 900mV و 1200mV
- الـ DDR memory voltage ثابت على 1.8V ومش مسموح يتغير
- الـ WiFi regulator لازم يفضل شغّال حتى أثناء الـ suspend
- الـ camera regulator ممنوع يتفعّل إلا لما الكاميرا شغّالة

كل القيود دي مش في الـ PMIC driver ولا في الـ consumer driver — دي قرارات خاصة بالـ board design. وده بالظبط اللي `machine.h` بتحله.

#### الحل

الـ board file (أو device tree + platform code) بتبني `struct regulator_init_data` وتملّيها بـ `regulation_constraints`، وتوصف مين بيستهلك كل regulator عن طريق `regulator_consumer_supply`.

```c
/* Board file: مثلاً arch/arm/mach-omap2/board-zoom.c */
static struct regulator_consumer_supply zoom_vmmc1_supply[] = {
    REGULATOR_SUPPLY("vmmc", "omap_hsmmc.0"),  /* MMC controller يستهلك vmmc */
};

static struct regulation_constraints zoom_vmmc1_constraints = {
    .name           = "VMMC1",
    .min_uV         = 1850000,   /* 1.85V minimum */
    .max_uV         = 3150000,   /* 3.15V maximum */
    .valid_modes_mask = REGULATOR_MODE_NORMAL | REGULATOR_MODE_STANDBY,
    .valid_ops_mask  = REGULATOR_CHANGE_VOLTAGE | REGULATOR_CHANGE_MODE
                     | REGULATOR_CHANGE_STATUS,
};

static struct regulator_init_data zoom_vmmc1 = {
    .constraints    = zoom_vmmc1_constraints,
    .num_consumer_supplies = ARRAY_SIZE(zoom_vmmc1_supply),
    .consumer_supplies = zoom_vmmc1_supply,
};
```

---

### الـ Structures المهمة في machine.h

#### `struct regulation_constraints` — قلب الملف

دي البنية الأساسية. بتحدد:

- **حدود الفولت**: `min_uV` / `max_uV`
- **حدود التيار**: `min_uA` / `max_uA`
- **الـ flags الثابتة**:
  - `always_on` — لا تطفي الـ regulator أبدًا (للـ CPU مثلاً)
  - `boot_on` — الـ bootloader فعّله وسيبه شغّال
  - `apply_uV` — طبّق الفولت من أول ما الـ kernel يشتغل
- **حالات الـ suspend**:
  - `state_mem` — الـ regulator يعمل إيه لما الجهاز ييجي في RAM suspend
  - `state_disk` — لما الجهاز ييجي في hibernation
  - `state_standby` — لما يكون في standby

```c
struct regulation_constraints {
    const char *name;

    int min_uV;          /* أقل فولت مسموح بيه */
    int max_uV;          /* أعلى فولت مسموح بيه */
    int min_uA;          /* أقل تيار */
    int max_uA;          /* أعلى تيار */

    unsigned int valid_ops_mask;   /* العمليات المسموح بيها */
    unsigned int valid_modes_mask; /* الـ modes المسموح بيها */

    struct regulator_state state_disk;     /* حالة الـ hibernate */
    struct regulator_state state_mem;      /* حالة الـ suspend-to-RAM */
    struct regulator_state state_standby;  /* حالة الـ standby */

    unsigned always_on:1;   /* مش ممكن يتطفى خالص */
    unsigned boot_on:1;     /* شغّال من البداية */
    unsigned apply_uV:1;    /* طبّق الفولت فورًا */
    unsigned system_critical:1; /* لو اتطفى — النظام هيوقع */
    /* ... */
};
```

#### `struct regulator_state` — حالة الـ suspend

```c
struct regulator_state {
    int uV;           /* الفولت أثناء الـ suspend */
    int min_uV;       /* أقل فولت مسموح في الـ suspend */
    int max_uV;       /* أعلى فولت مسموح في الـ suspend */
    unsigned int mode;
    int enabled;      /* DISABLE_IN_SUSPEND أو ENABLE_IN_SUSPEND */
    bool changeable;
};
```

#### `struct regulator_consumer_supply` — مين بيستهلك إيه

```c
struct regulator_consumer_supply {
    const char *dev_name;  /* اسم الـ device اللي بيستهلك */
    const char *supply;    /* اسم الـ supply زي "vcc" أو "vmmc" */
};

/* Macro مريح للإنشاء */
#define REGULATOR_SUPPLY(_name, _dev_name) \
{ .supply = _name, .dev_name = _dev_name }
```

#### `struct regulator_init_data` — حزمة التهيئة الكاملة

```c
struct regulator_init_data {
    const char *supply_regulator;           /* مين بيغذّيه (regulator فوقيه) */
    struct regulation_constraints constraints; /* الحدود */
    int num_consumer_supplies;
    struct regulator_consumer_supply *consumer_supplies; /* المستهلكين */
    void *driver_data;  /* بيانات خاصة بالـ driver */
};
```

---

### الـ REGULATOR_CHANGE_* Flags

دي بتحدد إيه اللي مسموح للـ consumer يغيّره:

| الـ Flag | المعنى |
|---|---|
| `REGULATOR_CHANGE_VOLTAGE` | مسموح تغيير الفولت |
| `REGULATOR_CHANGE_CURRENT` | مسموح تغيير التيار |
| `REGULATOR_CHANGE_MODE` | مسموح تغيير الـ operating mode |
| `REGULATOR_CHANGE_STATUS` | مسموح تشغيل/تطفية الـ regulator |
| `REGULATOR_CHANGE_DRMS` | مسموح الـ Dynamic Mode Switching |
| `REGULATOR_CHANGE_BYPASS` | مسموح وضعه في bypass mode |

---

### مثال عملي — رحلة تسجيل الـ regulator

```
Board Init                        Regulator Core
    │                                   │
    │  regulator_init_data              │
    │  (constraints + consumers)        │
    │                                   │
    ├──► platform_device_register() ───►│
    │                                   │
    │                            regulator_register()
    │                                   │
    │                         يتحقق من الـ constraints
    │                         يربط المستهلكين بالـ supply
    │                                   │
Consumer Driver                         │
    │                                   │
    ├──► regulator_get("vmmc") ─────────┤
    │                         يدوّر على المستهلك في القائمة
    │                         يرجعله handle
    │
    ├──► regulator_set_voltage(1800000, 3300000)
    │         يتحقق: هل 1.8V-3.3V ضمن الـ constraints؟
    │         يتحقق: هل CHANGE_VOLTAGE في valid_ops_mask؟
    │         لو آه → ينفذ / لو لأ → -EPERM
```

---

### الملفات المهمة في الـ Subsystem

#### الـ Headers (المرتبطة مع machine.h)

| الملف | الدور |
|---|---|
| `include/linux/regulator/machine.h` | واجهة الـ board/platform — هذا الملف |
| `include/linux/regulator/consumer.h` | واجهة الـ consumer driver |
| `include/linux/regulator/driver.h` | واجهة كاتب الـ PMIC driver |
| `include/linux/regulator/of_regulator.h` | ربط الـ device tree بالـ framework |
| `include/linux/regulator/fixed.h` | بورد عنده fixed-voltage regulator |

#### الـ Core

| الملف | الدور |
|---|---|
| `drivers/regulator/core.c` | قلب الـ framework — `regulator_register()` إلخ |
| `drivers/regulator/fixed.c` | driver للـ fixed-voltage regulators |
| `drivers/regulator/of_regulator.c` | يقرأ الـ device tree ويبني `regulator_init_data` |
| `drivers/regulator/dummy.c` | stub لما الـ regulator framework متعملش compile |

#### Hardware Drivers (أمثلة)

| الملف | الشريحة |
|---|---|
| `drivers/regulator/axp20x-regulator.c` | X-Powers AXP PMICs (Allwinner SoCs) |
| `drivers/regulator/tps65910-regulator.c` | TI TPS65910 PMIC |
| `drivers/regulator/max77686-regulator.c` | Maxim MAX77686 (Samsung Exynos) |
| `drivers/regulator/bd718x7-regulator.c` | ROHM BD718x7 (NXP i.MX8) |
| `drivers/regulator/rpi-panel-attiny-regulator.c` | Raspberry Pi Panel |

#### الـ Documentation

| الملف | المحتوى |
|---|---|
| `Documentation/power/regulator/overview.rst` | نظرة عامة على الـ framework |
| `Documentation/power/regulator/consumer.rst` | كيف يستخدمه الـ consumer driver |
| `Documentation/power/regulator/machine.rst` | كيف يستخدمه الـ board code |
| `Documentation/power/regulator/driver.rst` | كيف تكتب PMIC driver |

---

### ليه machine.h مهمة جداً؟

الـ framework بيمنع أي consumer يعمل حاجة خارج الـ constraints. لو CPU driver حاول يرفع الفولت لـ 2.0V وهو مسموحه بس لـ 1.2V — الـ core بيرفض. ده بيحمي الـ hardware من التلف. الـ `machine.h` هي المكان اللي البورد engineer بيكتب فيه "الحدود الآمنة" اللي الـ software مش يعدّاها.

الـ function `regulator_has_full_constraints()` مهمة كمان — لما الـ board code يناديها بتقول للـ framework "أنا كتبت constraints لكل الـ regulators"، فالـ framework يقدر يطفي أي regulator ما فيش حد بيستخدمه بدل ما يسيبه شغّال خوفًا.
## Phase 2: شرح الـ Regulator Framework

### المشكلة اللي الـ Subsystem بيحلها

في أي SoC حديث — خد مثلاً Qualcomm SM8550 أو TI AM62x — فيه عشرات الـ power rails مختلفة:
- CPU core بياخد 0.6V → 1.1V (حسب الـ frequency)
- DDR بياخد 1.1V ثابت
- WiFi بياخد 1.8V و 3.3V
- Camera sensor بياخد 2.8V بس لما الكاميرا شتغالة

من غير framework موحد، كل driver هيتعامل مع الـ PMIC (Power Management IC) بشكل مباشر — يعني:
- كل driver لازم يعرف عنوان الـ PMIC على الـ I2C/SPI
- كل driver يعرف الـ register map الخاص بالـ PMIC
- لو غيّرت الـ PMIC من Maxim لـ Qualcomm → تعيد كتابة كل الـ drivers
- مفيش حماية: driver ممكن يضبط voltage غلط ويحرق الـ hardware

الـ kernel محتاج **abstraction layer** يفصل بين:
1. اللي *بيستهلك* الكهربا (WiFi driver, CPU freq driver, camera driver)
2. اللي *بيوفرها* (PMIC driver, LDO driver, buck converter driver)

---

### الحل: الـ Regulator Framework

الـ kernel بيقدم **Regulator Framework** — طبقة وسيطة بتعمل:

1. **Abstraction**: الـ consumer driver بيقول "أنا محتاج `vcc-wifi`" من غير ما يعرف إنه LDO3 في PMIC عنوانه 0x58 على I2C-2.
2. **Constraints enforcement**: الـ board/machine code بيحدد إن `vcc-wifi` تقدر تتغير بس بين 1.8V و 3.3V — الـ framework مش بيسمح لأي حد يطلب برا الحدول دي.
3. **Reference counting**: لو WiFi driver و Bluetooth driver الاتنين بيستخدموا نفس الـ rail، الـ framework مش بيطفيها لحد ما الاتنين يقولوا "خلصت".
4. **Suspend management**: لما الـ system بيدخل suspend، الـ framework بيعرف أي regulator يطفيه وأي واحد يفضل شغال.

---

### التشبيه الواقعي: شركة كهرباء في مدينة

تخيل مدينة فيها شركة كهرباء مركزية (PMIC) بتوصّل تيار لمناطق مختلفة:

| المدينة | الـ Kernel |
|---------|-----------|
| شركة الكهرباء المركزية | PMIC driver (الـ provider) |
| الخطوط التوصيل لكل منطقة | الـ regulator rails (VCC_CPU, VCC_IO, ...) |
| عقد إيجار + حدود استهلاك | `struct regulation_constraints` |
| المصنع اللي بيطلب كهربا | consumer driver (WiFi, GPU, ...) |
| قاعدة بيانات العناوين | `struct regulator_consumer_supply` + lookup tables |
| قارئ العداد المركزي | الـ framework core (reference counting) |
| لائحة حالات الطوارئ | `struct regulator_state` (suspend states) |

**التفاصيل الدقيقة للتشبيه:**

- المصنع مش بيتصل بشركة الكهرباء مباشرة — بيتصل بـ "مكتب الخدمات" (الـ framework API).
- المكتب عنده "عقد لكل منطقة" (`regulation_constraints`) يحدد: أقل ضغط، أعلى ضغط، هل مسموح تغيير الضغط أصلاً.
- لو المصنع طلب ضغط برا الحدود، المكتب بيرفض فوراً — مش بيوصّل الطلب للشركة.
- في حالة انقطاع الكهرباء (suspend)، كل منطقة عندها تعليمات مسبقة: إيه اللي يتطفى وإيه اللي يفضل.

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Consumer Drivers                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │
│  │ WiFi drv │  │ GPU drv  │  │ CPU freq │  │ Camera sensor drv│   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────────┬─────────┘   │
│       │             │              │                  │             │
│       └─────────────┴──────────────┴──────────────────┘            │
│                             │                                       │
│                    regulator_get() / regulator_enable()             │
│                    regulator_set_voltage() ...                      │
│                             │                                       │
├─────────────────────────────▼───────────────────────────────────────┤
│                   Regulator Framework Core                          │
│                    (drivers/regulator/core.c)                       │
│                                                                     │
│  ┌──────────────────────┐    ┌─────────────────────────────────┐   │
│  │  Consumer Lookup     │    │   Constraint Enforcement        │   │
│  │  (supply name →      │    │   (min_uV, max_uV, ops_mask)    │   │
│  │   regulator_dev)     │    │                                 │   │
│  └──────────────────────┘    └─────────────────────────────────┘   │
│                                                                     │
│  ┌──────────────────────┐    ┌─────────────────────────────────┐   │
│  │  Reference Counting  │    │   Suspend State Manager         │   │
│  │  (enable/disable)    │    │   (state_mem / state_disk)      │   │
│  └──────────────────────┘    └─────────────────────────────────┘   │
│                             │                                       │
├─────────────────────────────▼───────────────────────────────────────┤
│                   Regulator Provider API                            │
│              (struct regulator_ops + regulator_desc)                │
│                             │                                       │
│       ┌─────────────────────┴──────────────────────┐               │
│       │                                            │               │
│  ┌────▼──────────────┐                  ┌──────────▼───────────┐   │
│  │  PMIC Driver      │                  │  Fixed Regulator     │   │
│  │  (e.g. pmic5.c)   │                  │  (regulator-fixed.c) │   │
│  │  I2C/SPI          │                  │  GPIO-controlled     │   │
│  └────┬──────────────┘                  └──────────┬───────────┘   │
│       │                                            │               │
└───────┼────────────────────────────────────────────┼───────────────┘
        │                                            │
   ┌────▼───────────┐                        ┌───────▼────────┐
   │  Real PMIC HW  │                        │  GPIO + FET    │
   │  (Qualcomm PM) │                        │  (board-level) │
   └────────────────┘                        └────────────────┘

        ▲ Machine/Board Layer (machine.h)
        │
        └── regulator_init_data
              ├── regulation_constraints   ← حدود التشغيل
              ├── regulator_consumer_supply[] ← مين بيستهلك إيه
              └── supply_regulator          ← الـ parent supply
```

---

### الـ machine.h في المنظومة دي: فين بالظبط؟

الـ `machine.h` هو **واجهة الـ board code** — يعني اللي بيكتبه مهندس الـ BSP لما بيوصف الـ hardware بتاعه للـ framework.

الملف ده بيعرّف:

#### 1. `struct regulation_constraints` — عقد التشغيل

```c
struct regulation_constraints {
    const char *name;       /* اسم وصفي للـ rail مثلاً "vcc-wifi" */

    int min_uV;             /* أقل جهد مسموح (بالـ microvolts) */
    int max_uV;             /* أعلى جهد مسموح */
    int uV_offset;          /* تعويض voltage drop على الـ PCB traces */

    int min_uA;             /* أقل تيار مسموح */
    int max_uA;             /* أعلى تيار مسموح */
    int ilim_uA;            /* أعلى تيار للـ input */

    unsigned int valid_ops_mask;   /* العمليات المسموحة (CHANGE_VOLTAGE, etc.) */
    unsigned int valid_modes_mask; /* الـ modes المسموحة (FAST, NORMAL, IDLE) */

    /* حالات الـ suspend */
    struct regulator_state state_disk;    /* hibernate */
    struct regulator_state state_mem;     /* suspend-to-RAM */
    struct regulator_state state_standby; /* standby */

    /* flags مضغوطة في 1 bit لكل منها */
    unsigned always_on:1;         /* لا يُطفى أبداً (مثلاً: rail الـ CPU) */
    unsigned boot_on:1;           /* البوتلودر شغّله */
    unsigned apply_uV:1;          /* طبّق الـ voltage عند الـ init */
    unsigned system_critical:1;   /* لو اتطفى، النظام هيوقع */
    unsigned over_current_protection:1; /* auto-disable عند over-current */
    /* ... */
};
```

**الـ `valid_ops_mask`** هو المفتاح — لو `REGULATOR_CHANGE_VOLTAGE` مش موجود فيه، أي driver يحاول يغير الـ voltage هياخد `-EPERM` من الـ framework فوراً.

#### 2. `struct regulator_state` — حالة الـ suspend

```c
struct regulator_state {
    int uV;          /* الجهد خلال الـ suspend (لو مختلف عن العادي) */
    int min_uV;
    int max_uV;
    unsigned int mode;
    int enabled;     /* DO_NOTHING_IN_SUSPEND / DISABLE_IN_SUSPEND / ENABLE_IN_SUSPEND */
    bool changeable; /* هل ممكن يتغير dynamically بعدين؟ */
};
```

مثال: الـ DDR rail لازم يفضل enabled في `state_mem` (suspend-to-RAM) لأن الذاكرة لازم تتحفظ، بس ممكن يتطفى في `state_disk` (hibernate لأن البيانات اتحفظت على الـ disk).

#### 3. `struct notification_limit` — حدود الإشعارات

```c
struct notification_limit {
    int prot;   /* hard limit → auto-protect (ايقاف تلقائي) */
    int err;    /* error threshold → بلّغ بـ REGULATOR_EVENT_ERROR */
    int warn;   /* warning threshold → بلّغ بـ REGULATOR_EVENT_WARN */
};
```

الـ framework بيستخدمها مع over-current / over-voltage / under-voltage / over-temp.

#### 4. `struct regulator_consumer_supply` — خريطة الاستهلاك

```c
struct regulator_consumer_supply {
    const char *dev_name;  /* اسم الـ device المستهلك (من dev_name()) */
    const char *supply;    /* اسم الـ supply من وجهة نظر المستهلك */
};
```

مثال واقعي من board code:

```c
static struct regulator_consumer_supply vcc_wifi_consumers[] = {
    REGULATOR_SUPPLY("vddio", "sdhci-pltfm.1"),  /* WiFi SDIO */
    REGULATOR_SUPPLY("vcc",   "brcmf_pcie.0"),   /* WiFi PCIe */
};
```

يعني: الـ rail الفيزيائي `LDO3` بيغذّي `sdhci-pltfm.1` اللي بيطلبه بالاسم `vddio`، وكمان `brcmf_pcie.0` اللي بيطلبه بالاسم `vcc`.

#### 5. `struct regulator_init_data` — الحزمة الكاملة

```c
struct regulator_init_data {
    const char *supply_regulator;   /* الـ parent supply (chain of regulators) */

    struct regulation_constraints constraints; /* العقد */

    int num_consumer_supplies;
    struct regulator_consumer_supply *consumer_supplies; /* الخريطة */

    void *driver_data; /* بيانات إضافية للـ PMIC driver */
};
```

ده اللي بيتحط في الـ `platform_data` أو الـ device tree parse code لكل regulator.

---

### علاقة الـ Structs ببعض

```
struct regulator_init_data
├── supply_regulator ──────────────────────► اسم regulator_init_data آخر
│                                            (مثلاً BUCK1 → VSYS)
├── struct regulation_constraints
│   ├── min_uV, max_uV, valid_ops_mask
│   ├── struct regulator_state state_mem
│   │   └── enabled = ENABLE_IN_SUSPEND
│   ├── struct regulator_state state_disk
│   │   └── enabled = DISABLE_IN_SUSPEND
│   └── struct notification_limit over_curr_limits
│       └── prot=5000000, err=4000000, warn=3500000 (uA)
│
└── struct regulator_consumer_supply[]
    ├── { "vddio", "sdhci-pltfm.1" }
    └── { "vcc",   "brcmf_pcie.0"  }
```

---

### الـ Supply Chain: الـ Regulators بتتسلسل

في الـ hardware الحقيقي، مش كل regulator بياخد من الـ battery مباشرة:

```
Battery (VBAT ~3.7V)
    │
    ├──► BUCK1 (step-down) → 1.8V (VDD_CORE)
    │       └──► LDO1 (low-dropout) → 1.2V (VDDQ_DDR)
    │       └──► LDO2 → 1.05V (VDD_CPU)
    │
    └──► BUCK2 → 3.3V (VDD_IO)
            └──► LDO3 → 1.8V (VCC_WIFI)
```

الـ `supply_regulator` في `regulator_init_data` بيمثّل التسلسل ده — اللي بيخلي الـ framework يعرف إنه لو عايز يشغّل `VCC_WIFI` لازم أول ما يتأكد إن `VDD_IO` شغال.

---

### الـ Flags الـ 1-bit: ليه بيستخدموا Bitfields؟

```c
unsigned always_on:1;
unsigned boot_on:1;
unsigned apply_uV:1;
```

لأن كل `struct regulation_constraints` بيعيش في الـ flash/ROM كـ static data — الـ bitfields بتوفر memory footprint. في SoC فيه 30+ regulator، ده بيفرق.

---

### الـ `regulator_has_full_constraints()`: وظيفة مهمة لطيفة

```c
#ifdef CONFIG_REGULATOR
void regulator_has_full_constraints(void);
#endif
```

الـ board code بيستدعيها بعد ما يسجّل كل الـ regulators — بتقول للـ framework: "أنا كتبت الـ constraints لكل الـ rails، أي regulator مش متعرّف يعتبر disabled". من غير الدالة دي، الـ framework بيكون conservative ومش بيطفي حاجة مش معرّفة.

---

### الـ Framework بيملك إيه مقابل إيه بيفوّض للـ Drivers؟

| المسؤولية | الـ Framework يملكها | الـ Driver يتعامل معها |
|-----------|---------------------|----------------------|
| التحقق من الـ constraints | نعم | لا |
| Reference counting (enable/disable) | نعم | لا |
| Suspend state management | نعم | جزئياً (ينفّذ) |
| الـ voltage/current الفعلي على الـ hardware | لا | نعم (عبر `regulator_ops`) |
| الـ register writes للـ PMIC | لا | نعم |
| الـ I2C/SPI communication مع الـ PMIC | لا | نعم |
| notification للـ consumers عند events | نعم | لا |
| الـ sysfs entries (`/sys/class/regulator/`) | نعم | لا |

---

### الـ Subsystems الأخرى اللي الـ Regulator Framework بيتكامل معها

- **Device Tree / OF (Open Firmware)**: الـ `regulator-fixed` و PMIC drivers بيقروا الـ constraints من DT بدل الـ `regulator_init_data` — بس الـ `machine.h` structs هي نفسها اللي بتتملى.
- **PM (Power Management) / suspend**: الـ framework بيستخدم `suspend_state_t` من `<linux/suspend.h>` عشان يعرف هو في `mem` ولا `disk` ولا `standby`.
- **Notifier framework**: الـ consumers ممكن يسجّلوا `notifier_block` عشان يستقبلوا events (over-voltage, under-voltage, ...).
- **sysfs / debugfs**: الـ framework تلقائياً بيعمل entries لكل regulator.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

> الملف المصدر: `include/linux/regulator/machine.h`
> الهدف: واجهة machine/board لـ Linux Regulator Framework — بتحدد للكيرنل إيه القيود والإعدادات اللي المنظم الكهربي (voltage/current regulator) يشتغل بيها على الـ SoC.

---

### 0. الـ Flags والـ Enums والـ Config Options — Cheatsheet

#### REGULATOR_CHANGE_* — valid_ops_mask flags

| الـ Flag | القيمة | المعنى |
|---|---|---|
| `REGULATOR_CHANGE_VOLTAGE` | `0x1` | السوفتوير يقدر يغير الجهد |
| `REGULATOR_CHANGE_CURRENT` | `0x2` | السوفتوير يقدر يغير الحد الأقصى للتيار |
| `REGULATOR_CHANGE_MODE` | `0x4` | السوفتوير يقدر يغير وضع التشغيل |
| `REGULATOR_CHANGE_STATUS` | `0x8` | السوفتوير يقدر يفعّل/يوقف المنظم |
| `REGULATOR_CHANGE_DRMS` | `0x10` | Dynamic Regulator Mode Switching مسموح |
| `REGULATOR_CHANGE_BYPASS` | `0x20` | السوفتوير يقدر يدخل bypass mode |

#### Suspend behavior — enabled field في regulator_state

| الـ Constant | القيمة | المعنى |
|---|---|---|
| `DO_NOTHING_IN_SUSPEND` | `0` | اتركه زي ما هو (الافتراضي) |
| `DISABLE_IN_SUSPEND` | `1` | اقطع الكهرباء أثناء suspend |
| `ENABLE_IN_SUSPEND` | `2` | خليه شغال أثناء suspend |

#### enum regulator_active_discharge

| القيمة | المعنى |
|---|---|
| `REGULATOR_ACTIVE_DISCHARGE_DEFAULT` | اتبع الـ default للـ hardware |
| `REGULATOR_ACTIVE_DISCHARGE_DISABLE` | عطّل active discharge |
| `REGULATOR_ACTIVE_DISCHARGE_ENABLE` | فعّل active discharge (يسرّع بلوغ الجهد صفر بعد الإيقاف) |

#### REGULATOR_MODE_* — وضع تشغيل المنظم (من consumer.h)

| الـ Mode | القيمة | الاستخدام |
|---|---|---|
| `REGULATOR_MODE_FAST` | `0x1` | لأحمال سريعة التغير (CPU DVFS) |
| `REGULATOR_MODE_NORMAL` | `0x2` | التشغيل العادي — الأكثر استخداماً |
| `REGULATOR_MODE_IDLE` | `0x4` | حمل خفيف، كفاءة أعلى |
| `REGULATOR_MODE_STANDBY` | `0x8` | حمل خفيف جداً، أقل ضوضاء |

#### CONFIG_REGULATOR — حماية بالـ Kconfig

```c
#ifdef CONFIG_REGULATOR
void regulator_has_full_constraints(void);
#else
static inline void regulator_has_full_constraints(void) {}
#endif
```

**الـ** `CONFIG_REGULATOR` لو مش موجود، كل الـ API بتبقى stubs فارغة — الـ driver بيتكمبايل بدون ما يحتاج `#ifdef` في كل مكان.

#### REGULATOR_NOTIF_LIMIT_* — قيم خاصة للـ notification_limit

| الـ Constant | القيمة | المعنى |
|---|---|---|
| `REGULATOR_NOTIF_LIMIT_DISABLE` | `-1` | عطّل الإشعار |
| `REGULATOR_NOTIF_LIMIT_ENABLE` | `-2` | فعّل الإشعار بدون حد محدد |

#### REGULATOR_DEF_UV_LESS_CRITICAL_WINDOW_MS

```c
#define REGULATOR_DEF_UV_LESS_CRITICAL_WINDOW_MS  10
```

بعد حدث under-voltage حرج، عندك 10ms تعمل فيهم حاجات أقل خطورة (زي logging) قبل ما تعمل الحاجات الحرجة (منع تلف الـ hardware).

---

### 1. الـ Structs المهمة

#### struct notification_limit

```c
struct notification_limit {
    int prot;   /* protection threshold — هيعمل action تلقائي */
    int err;    /* error threshold — يبعت error notification */
    int warn;   /* warning threshold — يبعت warning notification */
};
```

**الـ** `notification_limit` بتحدد ثلاث مستويات للرد على تجاوز حد معين:
- **`prot`**: الأخطر — بيشغّل حماية تلقائية (مثلاً يوقف المنظم)
- **`err`**: بيبعت error event للـ consumers عبر notifier chain
- **`warn`**: إنذار مبكر فقط

**بيُستخدم في** `regulation_constraints` لأربع حالات: over-current، over-voltage، under-voltage، over-temperature.

---

#### struct regulator_state

```c
struct regulator_state {
    int uV;           /* الجهد المستهدف أثناء suspend */
    int min_uV;       /* أدنى جهد مسموح أثناء suspend */
    int max_uV;       /* أقصى جهد مسموح أثناء suspend */
    unsigned int mode; /* وضع التشغيل أثناء suspend */
    int enabled;      /* DO_NOTHING / DISABLE / ENABLE */
    bool changeable;  /* هل يُسمح بتغيير الـ state ديناميكياً؟ */
};
```

**الـ** `regulator_state` بتصف حالة المنظم أثناء **system-wide suspend**. الـ `regulation_constraints` بتحمل ثلاث نسخ منها:
- `state_disk` → suspend-to-disk (hibernate)
- `state_mem` → suspend-to-RAM
- `state_standby` → standby

**مثال عملي** — راوتر مدمج: أثناء suspend-to-RAM، VDD_CPU يقفل (`DISABLE_IN_SUSPEND`)، لكن VDD_DDR يفضل شغال بجهد أقل (`ENABLE_IN_SUSPEND`, uV=900000).

---

#### struct regulation_constraints

الـ struct الأهم في الملف — بتحدد **كل القيود** اللي على المنظم:

```c
struct regulation_constraints {
    const char *name;           /* اسم وصفي للـ constraints */

    /* حدود الجهد بالـ microvolts */
    int min_uV;                 /* أدنى جهد يطلبه الـ consumer */
    int max_uV;                 /* أقصى جهد يطلبه الـ consumer */
    int uV_offset;              /* تعويض لانخفاض الجهد على الأسلاك */

    /* حدود التيار بالـ microamperes */
    int min_uA;
    int max_uA;
    int ilim_uA;                /* الحد الأقصى للتيار الداخل (input current) */

    int pw_budget_mW;           /* ميزانية الطاقة الكلية */
    int system_load;            /* حمل غير مرصود بأي consumer */

    /* للمنظمات المتزاوجة (coupled regulators) */
    u32 *max_spread;            /* أقصى فرق جهد بين منظمين متزاوجين */
    int max_uV_step;            /* أقصى خطوة تغيير جهد واحدة */

    unsigned int valid_modes_mask;  /* الـ modes المسموح بها */
    unsigned int valid_ops_mask;    /* العمليات المسموح بها */

    int input_uV;               /* جهد المصدر (لو المصدر نفسه منظم) */

    /* حالات الـ suspend */
    struct regulator_state state_disk;
    struct regulator_state state_mem;
    struct regulator_state state_standby;

    /* حدود الإشعار */
    struct notification_limit over_curr_limits;
    struct notification_limit over_voltage_limits;
    struct notification_limit under_voltage_limits;
    struct notification_limit temp_limits;

    suspend_state_t initial_state; /* حالة suspend ابتدائية */
    unsigned int initial_mode;     /* وضع تشغيل عند البدء */

    /* توقيت التغييرات */
    unsigned int ramp_delay;          /* uV/us — سرعة تغيير الجهد */
    unsigned int settling_time;       /* us — وقت الاستقرار (غير خطي) */
    unsigned int settling_time_up;    /* us — وقت الاستقرار عند الرفع */
    unsigned int settling_time_down;  /* us — وقت الاستقرار عند الخفض */
    unsigned int enable_time;         /* us — وقت الإيقاظ من zero */
    unsigned int uv_less_critical_window_ms; /* ms بعد UV حرج */

    unsigned int active_discharge;    /* regulator_active_discharge enum */

    /* bitfield flags */
    unsigned always_on:1;             /* لا يُوقف أبداً */
    unsigned boot_on:1;               /* الـ bootloader فعّله مسبقاً */
    unsigned apply_uV:1;              /* طبّق الجهد عند الـ init لو min==max */
    unsigned ramp_disable:1;          /* عطّل ramp delay */
    unsigned soft_start:1;            /* ارفع الجهد ببطء */
    unsigned pull_down:1;             /* مقاومة سحب عند الإيقاف */
    unsigned system_critical:1;       /* حرج لاستقرار النظام */
    unsigned over_current_protection:1; /* أوقف تلقائياً عند over-current */
    unsigned over_current_detection:1;  /* أرسل إشعار فقط */
    unsigned over_voltage_detection:1;
    unsigned under_voltage_detection:1;
    unsigned over_temp_detection:1;
};
```

**الـ** `always_on:1` مثلاً بيُستخدم لمنظم VDD_RTC في كل SoC — لو اتقطع الـ RTC هيضيع الوقت والـ wake alarm.

---

#### struct regulator_consumer_supply

```c
struct regulator_consumer_supply {
    const char *dev_name;  /* اسم الجهاز المستهلك (dev_name()) */
    const char *supply;    /* اسم الـ supply زي "vcc" أو "vdd" */
};
```

**بتربط** اسم supply بجهاز معين. بيتم ملؤها بالـ macro:

```c
/* مثال: ربط "vcc" بجهاز SPI */
REGULATOR_SUPPLY("vcc", "spi0.0")
```

---

#### struct regulator_init_data

```c
struct regulator_init_data {
    const char *supply_regulator;         /* اسم المنظم الأب (أو NULL للـ system supply) */
    struct regulation_constraints constraints; /* كل القيود */
    int num_consumer_supplies;            /* عدد الـ consumers */
    struct regulator_consumer_supply *consumer_supplies; /* مصفوفة الـ consumers */
    void *driver_data;                    /* بيانات خاصة بالـ driver */
};
```

**الـ** `regulator_init_data` هي **نقطة الدخول** الكاملة — الـ board file أو Device Tree يحدد واحدة لكل منظم، وبيمرّها للـ `regulator_register()`.

---

### 2. مخطط العلاقات بين الـ Structs

```
regulator_init_data
├── supply_regulator ──────────────────→ (اسم string يشير لـ regulator_init_data أخرى)
│
├── regulation_constraints
│   ├── state_disk  ─────────────→ regulator_state
│   ├── state_mem   ─────────────→ regulator_state
│   ├── state_standby ───────────→ regulator_state
│   ├── over_curr_limits ────────→ notification_limit
│   ├── over_voltage_limits ─────→ notification_limit
│   ├── under_voltage_limits ────→ notification_limit
│   └── temp_limits ─────────────→ notification_limit
│
├── consumer_supplies[] ─────────→ regulator_consumer_supply[]
│                                      ├── dev_name → "spi0.0"
│                                      └── supply   → "vcc"
│
└── driver_data ─────────────────→ (void* — يديره الـ driver)
```

---

### 3. مخطط دورة الحياة (Lifecycle)

```
Board/DTS Init
     │
     ▼
[تعريف regulator_init_data في board file]
     │
     ▼
regulator_register(regulator_desc, regulator_config)
     │   config->init_data = &my_init_data
     │
     ▼
[Core يقرأ regulation_constraints]
     │   ├─ يتحقق من min_uV/max_uV
     │   ├─ يطبّق apply_uV لو min==max
     │   ├─ يفعّل المنظم لو boot_on=1
     │   └─ يبني sysfs entries
     │
     ▼
[Consumer يطلب regulator_get(dev, "vcc")]
     │
     ▼
[Core يبحث في consumer_supplies عن تطابق dev_name + supply]
     │
     ▼
[Consumer يستخدم: regulator_enable() / regulator_set_voltage()]
     │   Core يتحقق من valid_ops_mask قبل أي عملية
     │
     ▼
[System Suspend]
     │   Core يطبّق state_mem / state_disk
     │
     ▼
[Consumer يستدعي regulator_put()]
     │
     ▼
[regulator_unregister()]
     │   يتحقق إن كل الـ consumers عملوا put
     │
     ▼
[Free resources]
```

---

### 4. مخطط تدفق الاستدعاء (Call Flow)

#### 4.1 تسجيل المنظم من الـ Board File

```
board_init() أو pmic_probe()
  └─→ regulator_register(&desc, &config)
        └─→ _regulator_register()
              ├─→ set_machine_constraints()
              │     ├─→ يقرأ constraints->valid_ops_mask
              │     ├─→ لو apply_uV: _regulator_do_set_voltage()
              │     │     └─→ ops->set_voltage() [hardware]
              │     ├─→ لو boot_on: _regulator_enable()
              │     │     └─→ ops->enable() [hardware]
              │     └─→ لو always_on: marks rdev->always_on
              └─→ يسجّل الـ consumers من consumer_supplies[]
```

#### 4.2 طلب Consumer لتغيير الجهد

```
driver calls regulator_set_voltage(regu, min_uV, max_uV)
  └─→ _regulator_set_voltage_unlocked()
        ├─→ يتحقق: REGULATOR_CHANGE_VOLTAGE في valid_ops_mask ؟
        │     └─→ لأ → return -EPERM
        ├─→ يتحقق: min_uV >= constraints->min_uV ؟
        │         max_uV <= constraints->max_uV ؟
        │     └─→ خارج الحدود → return -EINVAL
        ├─→ لو ramp_delay: احسب وقت الانتظار
        ├─→ ops->set_voltage() أو ops->set_voltage_sel()
        │     └─→ hardware register write
        └─→ لو !ramp_disable: msleep/usleep_range(ramp_time)
```

#### 4.3 Suspend Flow

```
pm_suspend(state)
  └─→ regulator_suspend_prepare(state)
        └─→ لكل rdev:
              ├─→ لو state == PM_SUSPEND_MEM:
              │     يقرأ constraints->state_mem
              ├─→ لو state == PM_SUSPEND_DISK:
              │     يقرأ constraints->state_disk
              └─→ رسّل:
                    ├─→ لو enabled==DISABLE: ops->disable()
                    ├─→ لو enabled==ENABLE:  ops->enable()
                    └─→ لو uV محدد: ops->set_voltage()
```

#### 4.4 Over-Current Detection Flow

```
hardware interrupt (PMIC IRQ)
  └─→ pmic_irq_handler()
        └─→ regulator_notifier_call_chain(rdev, REGULATOR_EVENT_OVER_CURRENT, NULL)
              └─→ لكل consumer مسجّل:
                    └─→ notifier_call(nb, event, data)
                          ├─→ لو prot threshold تجاوز:
                          │     regulator_force_disable()
                          └─→ لو err threshold فقط:
                                log error + notify userspace
```

---

### 5. استراتيجية الـ Locking

الـ locking في regulator framework بيتم في **regulator core** (`drivers/regulator/core.c`) وليس في ملف `machine.h` نفسه — لكن الـ constraints اللي بيحددها `machine.h` بتُحمى بالـ locks دي:

#### الـ Locks الأساسية

| الـ Lock | النوع | ما يحميه |
|---|---|---|
| `rdev->mutex` | `ww_mutex` (wound-wait) | كل عمليات منظم واحد (enable/disable/set_voltage) |
| `regulator_list_mutex` | `mutex` | قائمة كل المنظمات المسجلة |
| `rdev->supply->mutex` | نفس النوع | المنظم الأب عند تغيير الجهد المتسلسل |

#### ترتيب الـ Locking (Lock Ordering)

```
لو منظم B يعتمد على منظم A (supply chain):

regulator_set_voltage(B)
  └─→ lock(B->mutex)           [1]
        └─→ lock(A->mutex)     [2] — دايماً الـ child قبل الـ parent
              └─→ ops->set_voltage(A)
              unlock(A->mutex)
        └─→ ops->set_voltage(B)
  unlock(B->mutex)
```

**الـ** `ww_mutex` (wound-wait) بيُستخدم عشان يتجنب deadlock لما في coupled regulators محتاجة تأخذ أكتر من lock في نفس الوقت — لو في تعارض، الـ transaction الأحدث "تنجرح" (wound) وتعيد المحاولة.

#### حماية الـ Constraints

الـ `regulation_constraints` بعد ما بيتعمل `set_machine_constraints()` وقت الـ registration:
- بتبقى **read-only** — ما فيش lock محتاج للقراءة منها
- الحد الوحيد اللي ممكن يتغير ديناميكياً هو `regulator_state.changeable = true`، وده بيتحمى بـ `rdev->mutex`

```
rdev->mutex
  └─→ يحمي:
        ├─→ rdev->use_count     (عدد الـ consumers المفعّلين)
        ├─→ rdev->open_count    (عدد الـ consumers الحاصلين على handle)
        ├─→ rdev->voltage[]     (الجهد الحالي لكل suspend state)
        └─→ rdev->constraints   (بس للكتابة — القراءة بدون lock OK)
```

---

### ملاحظة على ملفات machine.h الأخرى

بما إن `include/linux/machine.h` غير موجود كملف مباشر، الـ Linux kernel بيقسّم مفهوم "machine interface" على subsystem مستقل لكل منها:

| الملف | الـ Subsystem |
|---|---|
| `include/linux/regulator/machine.h` | Voltage/Current Regulators |
| `include/linux/gpio/machine.h` | GPIO Lookup Tables + Hogs |
| `include/linux/pinctrl/machine.h` | Pin Mux/Config Mapping |
| `include/linux/iio/machine.h` | IIO Channel Mapping |

كلهم بيتبعوا نفس الفلسفة: **الـ board file أو DTS بيحدد الربط بين الـ hardware والـ driver** عبر static lookup tables — بدل ما كل driver يعرف تفاصيل الـ hardware بشكل مباشر.
## Phase 4: شرح الـ Functions

> **ملاحظة معمارية:** الـ `linux/regulator/machine.h` ده الـ **board/machine-level API** للـ regulator framework. الهدف منه إنك تعرّف قيود التشغيل (constraints) للـ regulators على مستوى الـ platform data — يعني كل القيم الحدية للـ voltage وcurrent والـ modes وحالات الـ suspend، وكمان الربط بين الـ regulator وأجهزة الاستهلاك (consumers).

---

### جدول ملخص — Regulator Machine API

| الـ Function / الـ Macro | النوع | الغرض الأساسي |
|---|---|---|
| `regulator_has_full_constraints()` | Function | إعلام الـ core إن كل الـ constraints اتحددت |
| `regulator_suspend_prepare()` | Inline stub | تهيئة الـ regulators للـ suspend (stub حالياً) |
| `regulator_suspend_finish()` | Inline stub | إنهاء الـ suspend للـ regulators (stub حالياً) |
| `REGULATOR_SUPPLY()` | Macro | بناء `regulator_consumer_supply` entry |

### جدول ملخص — Key Structs & Constants

| الـ Struct / الـ Constant | الغرض |
|---|---|
| `struct regulation_constraints` | القيود الكاملة للـ regulator (voltage, current, modes, flags) |
| `struct regulator_state` | حالة الـ regulator أثناء system suspend states |
| `struct regulator_consumer_supply` | ربط supply باسم device |
| `struct regulator_init_data` | البيانات الأولية الكاملة للـ regulator على مستوى الـ platform |
| `struct notification_limit` | حدود الـ protection/warning/error للـ over-current, over-voltage... إلخ |
| `REGULATOR_CHANGE_*` | Flags بتحدد ما يُسمح للـ consumer بتغييره |
| `DO_NOTHING / DISABLE / ENABLE _IN_SUSPEND` | سلوك الـ regulator في حالة الـ suspend |
| `enum regulator_active_discharge` | تحكم في الـ active discharge عند إيقاف الـ regulator |

---

### Group 1: Registration & Initialization Functions

الـ group ده بيحتوي على الـ functions اللي بتخبر الـ regulator core بالحالة الكاملة للـ system ومتى يكون آمن يـ disable أي regulator مش محتاجه حد.

---

#### `regulator_has_full_constraints()`

```c
#ifdef CONFIG_REGULATOR
void regulator_has_full_constraints(void);
#else
static inline void regulator_has_full_constraints(void) {}
#endif
```

**الـ function دي بتعمل إيه:**
الـ platform code بتستدعيها مرة واحدة بعد ما تخلص تسجيل كل الـ `regulator_init_data` الخاصة بالـ board. بتعلم الـ regulator core إن الـ machine قالت "كل الـ regulators المتاحة اتعرّفت وإحنا عارفين كل الـ consumers". من بعدها الـ core يقدر يـ disable بأمان أي regulator مش محتاجه أي consumer بدل ما يفضل شايله enabled احتياطاً.

**Parameters:** لا يوجد.

**Return value:** `void`.

**Key details:**
- من غير استدعاء الـ function دي، الـ core بيفترض إن في regulators ممكن تتسجل في أي وقت وبيكون **conservative** — يعني ما بيـ disable الـ regulators اللي مش محتاجها حد لأنه خايف يكون في consumer مسجّلش لسه.
- بعد استدعائها، الـ core بيعمل pass على كل الـ regulators غير المستخدمة ويـ disable أي واحد فيها `always_on = 0`.
- بيتستدعى من `machine_init()` أو `board_init()` بعد تسجيل كل الـ `regulator_init_data`.
- في حالة `!CONFIG_REGULATOR` بتبقى empty inline.

**Caller context:** Early init — بعد `platform_device_register` لكل الـ regulators.

**Pseudocode flow:**
```
regulator_has_full_constraints()
  → sets global flag: has_full_constraints = true
  → triggers a pass over all registered regulators
  → for each regulator with use_count == 0 and always_on == 0:
      → regulator_disable()
```

---

### Group 2: Suspend Lifecycle Stubs

الـ functions دول موجودين كـ stubs حالياً — جسمهم فاضي ورجعوا `0` دايماً. كانوا بيعملوا حاجة في إصدارات قديمة من الـ kernel لكن الـ functionality اتنقلت للـ regulator core نفسه عن طريق الـ `regulator_state` في الـ constraints.

---

#### `regulator_suspend_prepare()`

```c
static inline int regulator_suspend_prepare(suspend_state_t state)
{
    return 0;
}
```

**الـ function دي بتعمل إيه:**
كانت بتجهّز الـ regulators للـ suspend state المحددة (disk/mem/standby) عن طريق تطبيق الـ `regulator_state` الموافق. حالياً بقت stub وجسمها `return 0` لأن الـ core handler بيتعامل مع الـ suspend states أوتوماتيك عبر الـ PM notifiers.

**Parameters:**
- `state`: نوع الـ suspend — `PM_SUSPEND_MEM`, `PM_SUSPEND_DISK`, `PM_SUSPEND_STANDBY`.

**Return value:** `0` دايماً.

**Key details:**
- الـ `regulator_state` fields في `regulation_constraints` (زي `state_mem`, `state_disk`) هي اللي بتتطبق فعلاً من الـ core تلقائياً عبر الـ `regulator_pm_notifier`.
- الاستدعاء المباشر من الـ platform code مش ضروري في الـ modern kernel.

**Caller context:** Suspend path — من `syscore_suspend()` أو platform-specific suspend code.

---

#### `regulator_suspend_finish()`

```c
static inline int regulator_suspend_finish(void)
{
    return 0;
}
```

**الـ function دي بتعمل إيه:**
كانت بترجّع الـ regulators لحالتها قبل الـ suspend بعد الـ resume. حالياً stub كمان — الـ core بيتعامل مع الـ resume path تلقائياً.

**Parameters:** لا يوجد.

**Return value:** `0` دايماً.

**Caller context:** Resume path.

---

### Group 3: Helper Macros

الـ macros دول بتسهّل بناء الـ static data structures في الـ board files — بدل ما تكتب struct initialization يدوي بتستخدم macros واضحة.

---

#### `REGULATOR_SUPPLY(_name, _dev_name)`

```c
#define REGULATOR_SUPPLY(_name, _dev_name)  \
{                                           \
    .supply   = _name,                      \
    .dev_name = _dev_name,                  \
}
```

**الـ macro دي بتعمل إيه:**
بتبني `struct regulator_consumer_supply` initializer. بتربط اسم الـ supply (زي `"vcc"`) باسم الـ device اللي بتستهلكه (زي `"spi0.0"`). الـ regulator core بيستخدم الـ `dev_name` ده للـ lookup لما الـ consumer driver يعمل `regulator_get()`.

**Parameters:**
- `_name`: اسم الـ supply من ناحية الـ consumer (زي `"vdd"`, `"avcc"`).
- `_dev_name`: نتيجة `dev_name()` للـ consumer device — الـ string اللي بيتعرف بيها الـ device في sysfs وداخل الـ kernel.

**مثال عملي:**
```c
static struct regulator_consumer_supply my_codec_supplies[] = {
    REGULATOR_SUPPLY("AVDD", "soc-audio.0"),
    REGULATOR_SUPPLY("DVDD", "soc-audio.0"),
};
```

**Key details:**
- الـ `dev_name` لازم يطابق تماماً الـ string اللي بيرجعه `dev_name(dev)` للـ consumer device.
- لو الـ device اتسجل على باص زي I2C واسمه يتولّد late، ممكن تستخدم الـ `dev_name` الـ expected أو تستخدم DT بدل الـ board file static lookup.

---

### Group 4: Core Data Structures (Machine-Level API)

الـ structs دول هي الـ API الأساسية اللي بيكتبها الـ board/machine developer عشان يعرّف سلوك كل regulator على الـ platform.

---

#### `struct regulator_state`

```c
struct regulator_state {
    int uV;          /* target voltage in suspend */
    int min_uV;      /* minimum allowed voltage in suspend */
    int max_uV;      /* maximum allowed voltage in suspend */
    unsigned int mode;   /* operating mode in suspend */
    int enabled;     /* DO_NOTHING / DISABLE / ENABLE _IN_SUSPEND */
    bool changeable; /* can be toggled between enabled/disabled */
};
```

**الـ struct ده بيعمل إيه:**
بيوصف حالة الـ regulator أثناء system-wide low power states (disk/mem/standby). الـ board file بتملّاه وبتحطه في `regulation_constraints` عشان الـ core يعرف يعمل إيه بالـ regulator لما الـ system تدخل suspend.

**Fields:**
- `uV`: الـ voltage المطلوب في الـ suspend. لو `0` يتجاهله الـ core.
- `min_uV` / `max_uV`: نطاق الـ voltage المسموح بيه في الـ suspend — بيدي مرونة للـ core.
- `mode`: الـ operating mode (من `REGULATOR_MODE_*`) — مفيد لو عايز تحوّل لـ STANDBY mode في الـ suspend لتوفير طاقة.
- `enabled`: واحد من:
  - `DO_NOTHING_IN_SUSPEND (0)`: اتركه زي ما هو.
  - `DISABLE_IN_SUSPEND (1)`: أوقفه.
  - `ENABLE_IN_SUSPEND (2)`: اشغّله لو مش شغّال.
- `changeable`: بيسمح للـ userspace/PM code بتغيير الـ enabled state ديناميكياً.

**مثال عملي:**
```c
/* PMIC rail: disable in mem suspend, enable in standby */
.state_mem = {
    .enabled = DISABLE_IN_SUSPEND,
},
.state_standby = {
    .uV      = 1000000,  /* 1V in standby */
    .mode    = REGULATOR_MODE_STANDBY,
    .enabled = ENABLE_IN_SUSPEND,
},
```

---

#### `struct notification_limit`

```c
struct notification_limit {
    int prot;  /* protection threshold — auto-disable */
    int err;   /* error threshold — send error notification */
    int warn;  /* warning threshold — send warning notification */
};
```

**الـ struct ده بيعمل إيه:**
بيحدد ثلاث مستويات للرد على حالات الخطر (over-current, over-voltage, under-voltage, over-temp). الـ core بيقارن القراءة الفعلية بالحدود دي ويتصرف بناءً على النتيجة.

**Fields:**
- `prot`: الحد الحرج — لو اتخطى، الـ core يـ disable الـ regulator أوتوماتيك (hardware protection).
- `err`: الحد الأدنى للخطر — بيبعت `REGULATOR_EVENT_*` notification للـ consumers.
- `warn`: حد التحذير المبكر — بيبعت warning notification قبل ما الوضع يتدهور.
- القيمة الخاصة `REGULATOR_NOTIF_LIMIT_DISABLE (-1)` تعطّل المستوى ده.
- القيمة الخاصة `REGULATOR_NOTIF_LIMIT_ENABLE (-2)` تفعّل المستوى ده بالإعداد الافتراضي.

**مثال عملي:**
```c
/* Notify on over-current, protect at 2A */
.over_curr_limits = {
    .prot = 2000000,  /* 2A — disable regulator */
    .err  = 1800000,  /* 1.8A — error event */
    .warn = 1500000,  /* 1.5A — warning event */
},
```

---

#### `struct regulation_constraints`

```c
struct regulation_constraints {
    const char *name;

    int min_uV, max_uV;
    int uV_offset;

    int min_uA, max_uA, ilim_uA;
    int pw_budget_mW;
    int system_load;

    u32 *max_spread;       /* for coupled regulators */
    int max_uV_step;       /* max step size for voltage changes */

    unsigned int valid_modes_mask;  /* allowed operating modes */
    unsigned int valid_ops_mask;    /* allowed operations */

    int input_uV;          /* input voltage from upstream regulator */

    /* suspend states */
    struct regulator_state state_disk;
    struct regulator_state state_mem;
    struct regulator_state state_standby;

    /* fault detection limits */
    struct notification_limit over_curr_limits;
    struct notification_limit over_voltage_limits;
    struct notification_limit under_voltage_limits;
    struct notification_limit temp_limits;

    suspend_state_t initial_state;
    unsigned int initial_mode;

    unsigned int ramp_delay;       /* uV/us */
    unsigned int settling_time;    /* us, non-linear total */
    unsigned int settling_time_up; /* us, for voltage increase */
    unsigned int settling_time_down;/* us, for voltage decrease */
    unsigned int enable_time;      /* us, turn-on time */
    unsigned int uv_less_critical_window_ms; /* ms after UV event */
    unsigned int active_discharge;

    /* bitfield flags */
    unsigned always_on:1;
    unsigned boot_on:1;
    unsigned apply_uV:1;
    unsigned ramp_disable:1;
    unsigned soft_start:1;
    unsigned pull_down:1;
    unsigned system_critical:1;
    unsigned over_current_protection:1;
    unsigned over_current_detection:1;
    unsigned over_voltage_detection:1;
    unsigned under_voltage_detection:1;
    unsigned over_temp_detection:1;
};
```

**الـ struct ده بيعمل إيه:**
ده قلب الـ machine API. بيوصف كل القيود والصلاحيات لـ regulator واحد على مستوى الـ board. الـ core بيستخدمه كـ policy: ما بيسمحش لأي consumer يخرج عن الحدود المحددة هنا.

**أهم الـ fields:**

| الـ Field | الوصف |
|---|---|
| `min_uV` / `max_uV` | نطاق الـ output voltage المسموح للـ consumers يطلبوه |
| `uV_offset` | تعويض للـ voltage drop على الـ PCB traces — بيُضاف على الطلب |
| `min_uA` / `max_uA` | نطاق الـ output current المسموح |
| `ilim_uA` | الـ input current limit |
| `pw_budget_mW` | الحد الأقصى للطاقة الكلية للـ regulator |
| `system_load` | حِمل ثابت (زي الـ PCB leakage) مش بتحسبه الـ consumer requests |
| `max_spread` | بيستخدم في الـ coupled regulators — الفرق الأقصى المسموح بين الـ outputs |
| `max_uV_step` | الخطوة القصوى لتغيير الـ voltage في لحظة واحدة |
| `valid_modes_mask` | OR من `REGULATOR_MODE_*` — الـ modes المسموح بيها |
| `valid_ops_mask` | OR من `REGULATOR_CHANGE_*` — العمليات اللي يقدر الـ consumer يعملها |
| `input_uV` | الـ voltage المُدخَل لو الـ regulator ده بياخد من regulator تاني |
| `state_disk/mem/standby` | حالة الـ regulator في كل suspend state |
| `initial_state` | الـ suspend state الابتدائي |
| `initial_mode` | الـ operating mode اللي يبدأ بيه الـ regulator |
| `ramp_delay` | وقت الانتظار بعد تغيير الـ voltage — بالـ uV/µs |
| `settling_time` | وقت الاستقرار الكلي لو الـ ramp غير خطي |
| `settling_time_up/down` | وقت الاستقرار لكل اتجاه على حدة |
| `enable_time` | وقت الـ turn-on بالـ microseconds |
| `uv_less_critical_window_ms` | النافذة الزمنية بعد UV event اللي فيها يقدر الـ system يعمل logging وحاجات خفيفة |
| `active_discharge` | هل تفعّل الـ active discharge circuit عند الإيقاف |

**أهم الـ bitfield flags:**

| الـ Flag | المعنى |
|---|---|
| `always_on` | الـ regulator ده ما بيتوقفش طول ما الـ system شغّالة |
| `boot_on` | الـ bootloader شغّله — الـ core ما يحاولش يوقفه |
| `apply_uV` | لو `min_uV == max_uV`، طبّق الـ voltage ده فوراً عند الـ init |
| `ramp_disable` | عطّل الـ ramp delay — تغيير سريع بدون انتظار |
| `soft_start` | رفع الـ voltage ببطء عند الـ enable لتجنب الـ inrush current |
| `pull_down` | فعّل مقاومة الـ pull-down عند الإيقاف لتسريع الـ discharge |
| `system_critical` | الـ regulator ده حيوي للـ system — أي fault فيه يعمل panic |
| `over_current_protection` | اوقف الـ regulator أوتوماتيك لو حصل over-current |
| `over_current_detection` | بعّت notification للـ consumers لو حصل over-current |
| `over_voltage_detection` | بعّت notification لو حصل over-voltage |
| `under_voltage_detection` | بعّت notification لو حصل under-voltage |
| `over_temp_detection` | بعّت notification لو حصل over-temperature |

**`valid_ops_mask` flags:**

| الـ Flag | المعنى |
|---|---|
| `REGULATOR_CHANGE_VOLTAGE (0x1)` | الـ consumer يقدر يطلب voltage مختلف |
| `REGULATOR_CHANGE_CURRENT (0x2)` | الـ consumer يقدر يطلب current limit مختلف |
| `REGULATOR_CHANGE_MODE (0x4)` | الـ consumer يقدر يغير الـ operating mode |
| `REGULATOR_CHANGE_STATUS (0x8)` | الـ consumer يقدر يـ enable/disable الـ regulator |
| `REGULATOR_CHANGE_DRMS (0x10)` | يفعّل الـ Dynamic Regulator Mode Switching |
| `REGULATOR_CHANGE_BYPASS (0x20)` | الـ consumer يقدر يحط الـ regulator في bypass mode |

**مثال كامل واقعي:**
```c
static struct regulation_constraints ldo3_constraints = {
    .name           = "LDO3",
    .min_uV         = 1800000,  /* 1.8V min */
    .max_uV         = 3300000,  /* 3.3V max */
    .valid_modes_mask = REGULATOR_MODE_NORMAL | REGULATOR_MODE_IDLE,
    .valid_ops_mask   = REGULATOR_CHANGE_VOLTAGE | REGULATOR_CHANGE_STATUS,
    .always_on      = 0,
    .boot_on        = 1,        /* bootloader enabled it */
    .apply_uV       = 1,        /* min == max? apply it */
    .ramp_delay     = 200,      /* 200 uV/us */
    .enable_time    = 150,      /* 150 us turn-on */
    .state_mem = {
        .enabled = DISABLE_IN_SUSPEND,
    },
    .state_disk = {
        .enabled = DISABLE_IN_SUSPEND,
    },
    .over_current_protection = 1,
    .system_critical = 0,
};
```

---

#### `struct regulator_consumer_supply`

```c
struct regulator_consumer_supply {
    const char *dev_name;  /* dev_name() of the consumer device */
    const char *supply;    /* supply name, e.g. "vcc", "avdd" */
};
```

**الـ struct ده بيعمل إيه:**
بيربط supply بـ device محدد. الـ core بيستخدمه للـ lookup لما الـ consumer driver يعمل `regulator_get(dev, "vcc")` — بيدوّر على entry فيها `dev_name` يطابق `dev_name(dev)` و`supply` يطابق الـ supply ID.

**Fields:**
- `dev_name`: النتيجة المتوقعة لـ `dev_name(consumer_dev)`.
- `supply`: اسم الـ supply من ناحية الـ consumer — زي `"vcc"`, `"iovdd"`.

**Key details:**
- لو `dev_name` كانت `NULL`، الـ supply بيبقى متاح لأي consumer يطلبه بالاسم ده.
- الـ lookup بيحصل في `regulator_get()` → `regulator_dev_lookup()` → `regulator_match()`.

---

#### `struct regulator_init_data`

```c
struct regulator_init_data {
    const char *supply_regulator;      /* parent regulator name, or NULL */
    struct regulation_constraints constraints;
    int num_consumer_supplies;
    struct regulator_consumer_supply *consumer_supplies;
    void *driver_data;                 /* opaque, core doesn't touch it */
};
```

**الـ struct ده بيعمل إيه:**
ده الـ top-level struct اللي الـ board file بيبعته للـ regulator driver وقت التسجيل. بيجمع الـ constraints + قائمة الـ consumers + بيانات الـ driver-specific في مكان واحد.

**Fields:**
- `supply_regulator`: اسم الـ parent regulator اللي بيغذّي الـ regulator ده (Power Supply Chain). لو `NULL` يعني الـ system supply مباشرة.
- `constraints`: الـ `regulation_constraints` كاملة.
- `num_consumer_supplies`: عدد الـ entries في مصفوفة `consumer_supplies`.
- `consumer_supplies`: مصفوفة من `regulator_consumer_supply` — الربط بين الـ supply والـ consumers.
- `driver_data`: بيانات خاصة بالـ driver — الـ core ما بيلمسها. بتوصل لـ driver عن طريق `regulator_get_drvdata()`.

**مثال كامل:**
```c
static struct regulator_consumer_supply ldo5_consumers[] = {
    REGULATOR_SUPPLY("vmmc", "sdhci-omap.0"),
    REGULATOR_SUPPLY("vmmc", "sdhci-omap.1"),
};

static struct regulator_init_data ldo5_data = {
    .supply_regulator  = "VSYS",   /* fed from system 5V rail */
    .constraints = {
        .name           = "LDO5",
        .min_uV         = 3000000,
        .max_uV         = 3000000, /* fixed 3V */
        .apply_uV       = 1,
        .valid_ops_mask = REGULATOR_CHANGE_STATUS,
        .always_on      = 0,
        .state_mem      = { .enabled = DISABLE_IN_SUSPEND },
    },
    .num_consumer_supplies = ARRAY_SIZE(ldo5_consumers),
    .consumer_supplies     = ldo5_consumers,
};
```

---

### Group 5: Coupling & Advanced Constraint Features

#### الـ Coupled Regulators (`max_spread`)

الـ field `max_spread` في `regulation_constraints` بيستخدم لما يكون في أكتر من regulator مرتبطين ببعض (coupled) — زي الـ CPUs اللي بتشتغل على voltages متقاربة في الـ SMP systems.

```c
/* Example: two rails must stay within 100mV of each other */
static u32 cpu_spread[] = { 100000 }; /* 100mV max spread */

.max_spread = cpu_spread,
```

الـ core بيراعي الـ `max_spread` لما يحسب الـ voltage requests من كل الـ consumers على الـ coupled rails — بيضمن إن الفرق بين أي اتنين rails مش بيتخطى الـ `max_spread`.

#### الـ UV Event Window (`uv_less_critical_window_ms`)

```c
.uv_less_critical_window_ms = REGULATOR_DEF_UV_LESS_CRITICAL_WINDOW_MS, /* 10ms */
```

لما يحصل under-voltage حرج، في نافذة زمنية (افتراضياً 10ms) بيقدر فيها الـ system يعمل حاجات غير حرجة زي الـ logging أو flush الـ cache. بعد انتهاء النافذة، لازم يروح لأكشن أحرج زي الـ emergency shutdown لحماية الـ hardware.

#### الـ Active Discharge (`enum regulator_active_discharge`)

```c
enum regulator_active_discharge {
    REGULATOR_ACTIVE_DISCHARGE_DEFAULT,  /* use hardware default */
    REGULATOR_ACTIVE_DISCHARGE_DISABLE,  /* disable active discharge */
    REGULATOR_ACTIVE_DISCHARGE_ENABLE,   /* enable active discharge */
};
```

الـ active discharge circuit بتسحب تيار من الـ output عند الإيقاف عشان تشيّل الـ voltage بسرعة. مفيد لو في capacitor كبير على الـ output ومحتاج يتفرغ سريع قبل ما تشغّل الـ rail تاني.

---

### الـ Data Flow الكاملة — كيف تتفاعل الـ Structs

```
Board/Machine File
        │
        ▼
struct regulator_init_data
  ├── supply_regulator ──────────────────── Parent Rail Name
  ├── constraints (regulation_constraints)
  │     ├── min_uV / max_uV / valid_ops_mask / ...
  │     ├── state_mem / state_disk / state_standby
  │     └── over_curr_limits / flags (always_on, etc.)
  ├── consumer_supplies[]  ─────────────── [REGULATOR_SUPPLY() entries]
  │     └── {dev_name, supply}
  └── driver_data ──────────────────────── Opaque to core
        │
        ▼
regulator_register() / platform_device_register()
        │
        ▼
Regulator Core
  ├── regulator_dev_lookup()  ── consumer_supplies → regulator_get()
  ├── regulator_check_voltage() ── min_uV/max_uV
  ├── regulator_check_current_limit() ── min_uA/max_uA
  ├── regulator_check_mode() ── valid_modes_mask
  └── regulator_suspend_set_state() ── state_mem/disk/standby
        │
        ▼
regulator_has_full_constraints()  ── disable unused rails
```
## Phase 5: دليل الـ Debugging الشامل

> **الـ scope**: الـ machine-level board configuration interfaces — GPIO lookup (`gpio/machine.h`)، pinctrl mappings (`pinctrl/machine.h`)، و regulator init data (`regulator/machine.h`). الثلاثة بيشتغلوا على نفس المبدأ: بيربطوا hardware resources بـ devices في وقت الـ boot من غير Device Tree.

---

### Software Level

#### 1. Debugfs Entries

الـ debugfs بيكشف حالة كل subsystem بالتفصيل.

**GPIO:**

```
/sys/kernel/debug/gpio
/sys/kernel/debug/gpio_lookup_tables   (لو موجود في kernel version بتاعك)
```

```bash
# اقرأ كل GPIO lines وحالتها
cat /sys/kernel/debug/gpio

# مثال output:
# gpiochip0: GPIOs 0-31, parent: platform/fe200000.gpio, pinctrl-bcm2835:
#  gpio-4   (                    |led0                ) out hi
#  gpio-17  (                    |button              ) in  lo IRQ ACTIVE_LOW
```

**Pinctrl:**

```
/sys/kernel/debug/pinctrl/
/sys/kernel/debug/pinctrl/<ctrl-name>/pinmux-functions
/sys/kernel/debug/pinctrl/<ctrl-name>/pinmux-pins
/sys/kernel/debug/pinctrl/<ctrl-name>/pinconf-pins
/sys/kernel/debug/pinctrl/<ctrl-name>/pingroups
/sys/kernel/debug/pinctrl/<ctrl-name>/pins
/sys/kernel/debug/pinctrl/<ctrl-name>/pinmux-select   (write only)
```

```bash
# شوف كل pin controllers المتاحة
ls /sys/kernel/debug/pinctrl/

# شوف الـ pin functions المسجلة
cat /sys/kernel/debug/pinctrl/fe200000.gpio/pinmux-functions

# شوف إيه pin بيعمل إيه دلوقتي
cat /sys/kernel/debug/pinctrl/fe200000.gpio/pinmux-pins

# شوف الـ groups
cat /sys/kernel/debug/pinctrl/fe200000.gpio/pingroups
```

**Regulator:**

```
/sys/kernel/debug/regulator/
/sys/kernel/debug/regulator/<name>/
/sys/kernel/debug/regulator/regulator_summary
```

```bash
# ملخص كل regulators في النظام
cat /sys/kernel/debug/regulator/regulator_summary

# output بيبين: name, enabled, use_count, voltage, current, mode
# REGULATOR          STATUS  USE_COUNT  MIN_UV  MAX_UV  CURRENT_UA
# 3p3-supply         enabled     2      3300000 3300000      0

# تفاصيل regulator معين
cat /sys/kernel/debug/regulator/3p3-supply/enable_count
cat /sys/kernel/debug/regulator/3p3-supply/min_microvolts
cat /sys/kernel/debug/regulator/3p3-supply/max_microvolts
```

---

#### 2. Sysfs Entries

**GPIO (عبر gpiod interface):**

```
/sys/class/gpio/                          (legacy — deprecated)
/sys/bus/gpio/devices/                    (جديد)
/sys/kernel/debug/gpio                    (الأهم للـ debug)
```

```bash
# export GPIO للـ userspace لو محتاج تتحقق (legacy)
echo 17 > /sys/class/gpio/export
cat /sys/class/gpio/gpio17/direction
cat /sys/class/gpio/gpio17/value
cat /sys/class/gpio/gpio17/active_low
echo 17 > /sys/class/gpio/unexport
```

**Pinctrl:**

```
/sys/bus/platform/devices/<dev>/pinctrl_state   (لو driver بيستخدمه)
/sys/bus/platform/drivers/<drv>/
```

**Regulator:**

```
/sys/class/regulator/
/sys/class/regulator/regulator.X/
    microvolts        — الـ voltage الحالي
    opmode            — الـ operating mode
    state             — enabled/disabled
    type              — voltage/current
    name
    num_users
    requested_microvolts
```

```bash
# شوف كل regulators
ls /sys/class/regulator/

# الحالة الحالية لـ regulator.0
cat /sys/class/regulator/regulator.0/state
cat /sys/class/regulator/regulator.0/microvolts
cat /sys/class/regulator/regulator.0/num_users
```

---

#### 3. Ftrace — Tracepoints والـ Events

```bash
# mount tracefs لو مش mounted
mount -t tracefs nodev /sys/kernel/tracing

cd /sys/kernel/tracing

# شوف GPIO events المتاحة
grep -r gpio available_events

# شوف regulator events
grep regulator available_events

# فعّل GPIO request/free events
echo 1 > events/gpio/enable

# فعّل regulator events (enable/disable/voltage set)
echo 1 > events/regulator/enable

# أو فعّل events بعينها
echo 1 > events/regulator/regulator_enable/enable
echo 1 > events/regulator/regulator_set_voltage/enable
echo 1 > events/regulator/regulator_set_voltage_complete/enable

# فعّل pinctrl events
echo 1 > events/pinctrl/enable

# ابدأ الـ tracing
echo 1 > tracing_on

# شغّل الـ device/operation اللي محتاج تتابعه
# ...

# اقرأ النتيجة
cat trace

# مثال output:
# regulator_enable: name=3p3-supply
# regulator_set_voltage: name=vcore min=900000 max=1100000
# regulator_set_voltage_complete: name=vcore val=1000000
```

**Function tracing لـ gpiod_get:**

```bash
echo 'gpiod_get*' > set_ftrace_filter
echo function > current_tracer
echo 1 > tracing_on
# ... trigger the GPIO request ...
cat trace
echo nop > current_tracer
```

---

#### 4. Printk و Dynamic Debug

**تفعيل dynamic debug لـ GPIO subsystem:**

```bash
# تفعيل كل debug messages في gpio core
echo 'file drivers/gpio/gpiolib.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/gpio/gpiolib-of.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug للـ gpio machine (lookup tables)
echo 'file drivers/gpio/gpiolib-acpi.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل كل gpio messages دفعة واحدة
echo 'module gpiolib +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل pinctrl debug
echo 'file drivers/pinctrl/core.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/pinctrl/pinmux.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/pinctrl/pinconf.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل regulator debug
echo 'file drivers/regulator/core.c +p' > /sys/kernel/debug/dynamic_debug/control

# أو بـ kernel command line
# dyndbg="file drivers/gpio/gpiolib.c +p; file drivers/pinctrl/core.c +p"
```

**تفعيل verbose logging بـ boot parameter:**

```bash
# في /boot/cmdline.txt أو GRUB
gpio.debug=1
regulator.debug=1
```

**printk level:**

```bash
# خفّض الـ log level لتشوف debug messages
echo 8 > /proc/sys/kernel/printk
# أو
dmesg -n 8
```

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الوظيفة |
|---|---|
| `CONFIG_DEBUG_GPIO` | يفعّل GPIO debug messages والـ sysfs exports |
| `CONFIG_GPIOLIB` | أساسي — بدونه مفيش GPIO subsystem |
| `CONFIG_GPIO_SYSFS` | يكشف GPIOs عبر `/sys/class/gpio` |
| `CONFIG_DEBUG_PINCTRL` | يفعّل pinctrl verbose logging |
| `CONFIG_PINCTRL` | أساسي للـ pinctrl subsystem |
| `CONFIG_REGULATOR_DEBUG` | يفعّل regulator debug messages |
| `CONFIG_REGULATOR` | أساسي للـ regulator framework |
| `CONFIG_DEBUG_KERNEL` | يفعّل kernel debugging عموماً |
| `CONFIG_DYNAMIC_DEBUG` | يسمح بـ dynamic debug activation |
| `CONFIG_FTRACE` | أساسي للـ function tracing |
| `CONFIG_TRACING` | يفعّل kernel tracing framework |
| `CONFIG_GPIO_MOCKUP` | GPIO driver وهمي للـ testing |
| `CONFIG_REGULATOR_VIRTUAL_CONSUMER` | virtual regulator consumer للـ testing |
| `CONFIG_REGULATOR_FIXED_VOLTAGE` | fixed voltage regulator (شائع في board configs) |

```bash
# تحقق من options في kernel الحالي
zcat /proc/config.gz | grep -E 'CONFIG_(DEBUG_GPIO|DEBUG_PINCTRL|REGULATOR_DEBUG)'

# أو
grep -E 'DEBUG_GPIO|DEBUG_PINCTRL|REGULATOR_DEBUG' /boot/config-$(uname -r)
```

---

#### 6. أدوات الـ Subsystem الخاصة

**gpioinfo / gpioset / gpioget (libgpiod):**

```bash
# اعرف كل GPIO chips والـ lines
gpioinfo

# مثال output:
# gpiochip0 - 54 lines:
#         line   0:      unnamed       unused   input  active-high
#         line   4:      unnamed        "led0"  output  active-high [used]
#         line  17:      unnamed      "button"   input   active-low [used]

# اقرأ قيمة GPIO
gpioget gpiochip0 17

# اضبط GPIO
gpioset gpiochip0 4=1

# مراقبة GPIO events
gpiomon --num-events=5 gpiochip0 17
```

**pinctrl tool (من kernel tools):**

```bash
# الـ pinctrl debug info كلها عبر debugfs — مافيش standalone tool عادةً
# لكن ممكن تستخدم:
cat /sys/kernel/debug/pinctrl/<ctrl>/pinmux-pins | grep -v "^pin [0-9]*: " | grep -v "^$"
```

**regulator tool:**

```bash
# مفيش standalone tool — كل حاجة عبر sysfs/debugfs
# script مفيد لمراقبة كل regulators:
for r in /sys/class/regulator/*/; do
    name=$(cat "$r/name" 2>/dev/null || echo "?")
    state=$(cat "$r/state" 2>/dev/null || echo "?")
    uV=$(cat "$r/microvolts" 2>/dev/null || echo "?")
    users=$(cat "$r/num_users" 2>/dev/null || echo "?")
    printf "%-20s state=%-10s voltage=%-10s users=%s\n" "$name" "$state" "$uV" "$users"
done
```

---

#### 7. جدول رسائل الـ Error الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `gpio-XX (device): gpiod_get: no GPIO consumer XX found` | الـ GPIO مش موجود في lookup table ولا DT | أضف `GPIO_LOOKUP` entry للـ device أو تحقق من DT property |
| `gpio-XX: requested but not yet claimed` | GPIO مش محجوز بـ `gpiod_get()` | تحقق إن الـ driver بيعمل `gpiod_get()` صح |
| `GPIO line XX is already in use` | GPIO محجوز من driver تاني | تحقق من الـ lookup tables — ممكن تضارب |
| `Could not get gpio XX` | فشل `gpiod_get()` | تحقق من الـ key و con_id في `gpiod_lookup_table` |
| `pin XX already requested` | تعارض في pinctrl — اتنين بيطلبوا نفس الـ pin | راجع الـ pinctrl mapping tables وتأكد مفيش overlap |
| `pin-group XX is not registered` | الـ group المكتوب في `PIN_MAP_MUX_GROUP` مش موجود في controller | تحقق من اسم الـ group في controller driver |
| `could not get GPIO XX: -16 (EBUSY)` | GPIO مستخدم من process تاني | `cat /sys/kernel/debug/gpio` تشوف مين حاجزه |
| `regulator: XX supply not found, using dummy regulator` | الـ supply المطلوب مش مسجل في `regulator_init_data` | تحقق من `.supply_regulator` و consumers list |
| `regulator XX: minimum voltage X uV below XX uV` | طلب voltage أقل من الـ `min_uV` في constraints | راجع `regulation_constraints.min_uV` |
| `regulator XX: invalid state` | حاولت تشغّل regulator في state مش معرّف | تحقق من `valid_ops_mask` |
| `Failed to get regulator XX` | `regulator_get()` فشل — الـ supply مش موجود | تحقق من `regulator_consumer_supply` table |
| `pinctrl: request() failed for pin XX` | فشل hardware-level pin request | تحقق من controller driver وحالة الـ hardware |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في gpiod_add_lookup_table() — تحقق من تسجيل table مكررة */
void gpiod_add_lookup_table(struct gpiod_lookup_table *table)
{
    mutex_lock(&gpio_lookup_lock);
    /* نقطة مناسبة للـ WARN لو table مسجلة قبل كده */
    WARN_ON(lookup_table_already_registered(table));
    list_add_tail(&table->list, &gpio_lookup_list);
    mutex_unlock(&gpio_lookup_lock);
}

/* في pinctrl_register_mappings() — تحقق من consistency */
int pinctrl_register_mappings(const struct pinctrl_map *maps, unsigned num)
{
    for (i = 0; i < num; i++) {
        /* WARN_ON لو type invalid */
        WARN_ON(maps[i].type == PIN_MAP_TYPE_INVALID);
        /* WARN_ON لو dev_name فاضية */
        WARN_ON(!maps[i].dev_name);
    }
}

/* في regulator_set_voltage() — تحقق من الـ constraints */
/* ضيف dump_stack() هنا لتعرف مين بيغير الـ voltage */
if (min_uV < rdev->constraints->min_uV) {
    dev_err(rdev_get_dev(rdev), "below minimum voltage\n");
    WARN_ON(1);  /* يطبع stack trace + يكمّل */
}
```

**نقاط مهمة للـ WARN_ON:**

```
- بعد كل gpiod_add_lookup_table() تحقق إن dev_id مش NULL
- في gpiod_get() لو عدد الـ lookups > عدد الـ GPIOs الفعلية في chip
- في pinctrl_register_mappings() لو num_maps = 0
- في regulation_constraints تحقق إن min_uV <= max_uV
- في regulator_init_data تحقق إن consumer_supplies != NULL لو num > 0
```

---

### Hardware Level

#### 1. التحقق إن Hardware State بيطابق Kernel State

**GPIO:**

```bash
# الـ kernel يشوف GPIO 17 إنها input active-low — تحقق من hardware:
cat /sys/kernel/debug/gpio | grep "gpio-17"
# gpio-17  (button) in lo ACTIVE_LOW

# لو الـ hardware pin فعلاً HIGH بس kernel شايفه LOW —
# معناه ACTIVE_LOW flag شغال صح (الـ inversion بتحصل في kernel)
# لو الـ hardware pin LOW وكمان kernel شايفه LOW (بدون ACTIVE_LOW) — ok
# لو في تناقض: تحقق من flags في gpiod_lookup table
```

**Regulator:**

```bash
# الـ kernel يقول regulator enabled ويديه 3.3V
cat /sys/class/regulator/regulator.2/state      # enabled
cat /sys/class/regulator/regulator.2/microvolts # 3300000

# تحقق من الـ hardware بـ oscilloscope أو multimeter على الـ rail
# لو مختلف: احتمال الـ regulator driver مش بيقرأ feedback صح
```

#### 2. Register Dump Techniques

```bash
# قراءة register بـ devmem2 (لو مثبت)
devmem2 0xFE200000 w    # قراءة GPIO base register على RPi4

# بدون devmem2 — بـ /dev/mem مباشرة (محتاج CONFIG_STRICT_DEVMEM=n)
python3 -c "
import mmap, os, struct
f = os.open('/dev/mem', os.O_RDONLY | os.O_SYNC)
m = mmap.mmap(f, 0x1000, offset=0xFE200000)
val = struct.unpack('<I', m.read(4))[0]
print(hex(val))
m.close()
os.close(f)
"

# قراءة GPIO function select registers على BCM2835/BCM2711
# GPFSEL0 @ 0xFE200000 — bits 0-29 بيتحكموا في GPIO 0-9
devmem2 0xFE200000 w   # GPFSEL0
devmem2 0xFE200004 w   # GPFSEL1 (GPIO 10-19)
devmem2 0xFE200008 w   # GPFSEL2 (GPIO 20-29)

# قراءة GPIO level registers
devmem2 0xFE200034 w   # GPLEV0 — GPIO 0-31 current levels
```

```bash
# io command (من package ioports أو الـ kernel tools)
# للـ x86 فقط — port-mapped I/O
io -4 -r 0x400   # قراءة 4 bytes من port 0x400
```

**Script لـ dump GPIO controller registers:**

```bash
#!/bin/bash
GPIO_BASE=0xFE200000  # غيّره حسب SoC بتاعك

echo "=== GPIO Register Dump ==="
for offset in 0x00 0x04 0x08 0x0C 0x10 0x14 0x34 0x38; do
    addr=$((GPIO_BASE + offset))
    val=$(devmem2 $(printf '0x%X' $addr) w 2>/dev/null | grep "Read" | awk '{print $NF}')
    printf "0x%08X: %s\n" $addr "$val"
done
```

#### 3. Logic Analyzer / Oscilloscope Tips

**للـ GPIO:**

```
- قيس الـ rise/fall time: لو بطيء جداً → احتمال pull-up مش متفعل
  صح: PULL_UP flag في gpiod_lookup أو hardware pull-up

- قيس الـ voltage level:
  HIGH: لازم ≥ 0.7 × VCC
  LOW:  لازم ≤ 0.3 × VCC
  لو في الوسط: floating pin — تحقق من PULL_UP/PULL_DOWN

- لو GPIO_OPEN_DRAIN: الـ HIGH بيجي من external pull-up فقط
  لو مفيش pull-up → الـ line بتفضل LOW دايماً

- قيس delay بين kernel gpiod_set_value() والـ physical change
  لو > 1µs: احتمال I2C/SPI GPIO expander (indirect GPIO)
```

**للـ Regulator:**

```
- قيس الـ voltage على الـ output rail مباشرة
- قيس الـ enable signal (EN pin)
- راقب الـ startup sequence: VDD_CPU لازم يجي قبل VDD_IO
- لو voltage بيرتجف (ripple) > 50mV: احتمال capacitor مشكلة
- قيس الـ voltage أثناء load change لتتحقق من الـ regulation
```

**للـ Pinctrl:**

```
- بعد ما kernel يعمل pinctrl_select_state():
  اقيس الـ pin function بـ logic analyzer
  تأكد الـ pin بيشتغل كـ UART/SPI/I2C مش كـ GPIO
- لو SPI clock مش بيظهر: الـ pin لسه configured كـ GPIO
  راجع الـ PIN_MAP_MUX_GROUP entry
```

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة | الـ Log Pattern | التشخيص |
|---|---|---|
| GPIO floating (لا pull-up ولا pull-down) | قراءات عشوائية في IRQ handler | أضف `GPIO_PULL_UP` أو `GPIO_PULL_DOWN` في lookup flags |
| OPEN_DRAIN بدون external pull-up | GPIO عالق على LOW دايماً | أضف external pull-up أو استخدم push-pull |
| Regulator مش بيوصل للـ target voltage | `regulator_set_voltage: X uV (requested Y uV)` | تحقق من hardware feedback resistors |
| Pin conflict — اتنين بيستخدموا نفس الـ pin | `pin XX already requested` | راجع الـ pinctrl map table للـ overlaps |
| GPIO expander مش بيرد | `gpio-XX: timed out waiting` | تحقق من I2C/SPI bus + clock |
| Regulator enable بيفشل | `regulator_enable: X failed: -EIO` | تحقق من EN pin connectivity + I2C address |
| Boot regulator sequence خاطئة | Kernel panic في early boot | راجع ترتيب `always_on` و `boot_on` في constraints |

#### 5. Device Tree Debugging

**تحقق إن الـ DT بيطابق الـ hardware:**

```bash
# اقرأ الـ compiled DT الحالي
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | less

# ابحث عن GPIO properties
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A5 "gpios"

# تحقق من regulator nodes
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A20 "regulator@"

# تحقق من pinctrl nodes
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A10 "pinctrl"
```

```bash
# قارن DT source مع الـ compiled:
# DT source:
cat /boot/overlays/your-overlay.dts

# Compiled (loaded):
dtc -I fs -O dts /sys/firmware/devicetree/base 2>/dev/null > /tmp/current.dts
diff /boot/overlays/your-overlay.dts /tmp/current.dts
```

**تحقق من GPIO assignment صح:**

```bash
# لو device بيستخدم DT — شوف properties
cat /proc/device-tree/soc/gpio@7e200000/gpios
# أو بـ Python:
python3 -c "
import os
base = '/proc/device-tree'
for root, dirs, files in os.walk(base):
    for f in files:
        if 'gpio' in f.lower():
            print(os.path.join(root, f))
"
```

**تحقق من pinctrl state:**

```bash
# شوف الـ pinctrl state الحالي للـ device
cat /sys/bus/platform/devices/MYDEV.0/pinctrl-state 2>/dev/null

# أو عبر debugfs
cat /sys/kernel/debug/pinctrl/MYCTRL/pinmux-pins | grep MYDEV
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للـ Copy

**فحص سريع للـ GPIO subsystem:**

```bash
#!/bin/bash
echo "=== GPIO Debug Report ==="
echo "--- GPIO Chips ---"
gpioinfo 2>/dev/null || cat /sys/kernel/debug/gpio

echo "--- GPIO Lookup Tables ---"
cat /sys/kernel/debug/gpio_lookup_tables 2>/dev/null || echo "Not available"

echo "--- GPIO IRQ mappings ---"
cat /proc/interrupts | grep -i gpio
```

**فحص سريع للـ Pinctrl:**

```bash
#!/bin/bash
CTRL=$(ls /sys/kernel/debug/pinctrl/ | head -1)
echo "=== Pinctrl Debug: $CTRL ==="
echo "--- Pins ---"
cat /sys/kernel/debug/pinctrl/$CTRL/pins | head -30
echo "--- Mux Functions ---"
cat /sys/kernel/debug/pinctrl/$CTRL/pinmux-functions
echo "--- Active Pin Mux ---"
cat /sys/kernel/debug/pinctrl/$CTRL/pinmux-pins
echo "--- Pin Groups ---"
cat /sys/kernel/debug/pinctrl/$CTRL/pingroups
```

**فحص سريع للـ Regulators:**

```bash
#!/bin/bash
echo "=== Regulator Debug Report ==="
echo "--- Summary ---"
cat /sys/kernel/debug/regulator/regulator_summary 2>/dev/null

echo "--- Per Regulator Status ---"
for r in /sys/class/regulator/*/; do
    name=$(cat "$r/name" 2>/dev/null || basename "$r")
    state=$(cat "$r/state" 2>/dev/null || echo "unknown")
    uV=$(cat "$r/microvolts" 2>/dev/null || echo "N/A")
    users=$(cat "$r/num_users" 2>/dev/null || echo "0")
    printf "%-25s %-10s %10s uV  %s users\n" "$name" "$state" "$uV" "$users"
done
```

**تفعيل كل الـ debugging دفعة واحدة:**

```bash
#!/bin/bash
# Dynamic debug للـ machine-level board config subsystems
for file in \
    "drivers/gpio/gpiolib.c" \
    "drivers/gpio/gpiolib-of.c" \
    "drivers/pinctrl/core.c" \
    "drivers/pinctrl/pinmux.c" \
    "drivers/pinctrl/pinconf.c" \
    "drivers/regulator/core.c"; do
    echo "file $file +p" > /sys/kernel/debug/dynamic_debug/control
    echo "Enabled debug for: $file"
done

# فعّل ftrace events
echo 1 > /sys/kernel/tracing/events/gpio/enable
echo 1 > /sys/kernel/tracing/events/regulator/enable
echo 1 > /sys/kernel/tracing/events/pinctrl/enable 2>/dev/null
echo 1 > /sys/kernel/tracing/tracing_on

echo "Debug enabled. Check dmesg and /sys/kernel/tracing/trace"
```

**مراقبة GPIO events بـ ftrace:**

```bash
# تابع الـ GPIO requests في real time
cd /sys/kernel/tracing
echo 0 > tracing_on
echo > trace
echo 1 > events/gpio/enable
echo 1 > tracing_on
sleep 5
cat trace | grep -E "gpio_direction|gpio_value|gpio_request"
echo 0 > tracing_on
```

**تشخيص مشكلة regulator_get() فاشل:**

```bash
# 1. تحقق من اسم الـ supply
dmesg | grep -i "supply\|regulator" | tail -20

# 2. شوف الـ regulator المسجلين
cat /sys/kernel/debug/regulator/regulator_summary

# 3. فعّل debug لـ regulator core وشوف الـ lookup process
echo 'file drivers/regulator/core.c +p' > /sys/kernel/debug/dynamic_debug/control
# أعد تشغيل الـ driver اللي بيفشل:
echo "your_driver" > /sys/bus/platform/drivers/your_driver/unbind 2>/dev/null
echo "your_device" > /sys/bus/platform/drivers/your_driver/bind 2>/dev/null
dmesg | tail -30
```

**تشخيص pin conflict:**

```bash
# 1. شوف مين حاجز الـ pin المشكلة
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep "pin XX"

# 2. شوف كل conflicts
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep -v "UNCLAIMED" | sort

# 3. تحقق من الـ mapping tables المسجلة
dmesg | grep -i "pinctrl\|pinmux" | head -30
```

#### مثال على Output وتفسيره

```bash
$ cat /sys/kernel/debug/gpio
gpiochip0: GPIOs 0-53, parent: platform/fe200000.gpio, pinctrl-bcm2711:
 gpio-4   (                    |led-red             ) out hi        # LED شغال
 gpio-5   (                    |led-green           ) out lo        # LED مطفي
 gpio-17  (                    |button-1            ) in  lo ACTIVE_LOW  # زرار مضغوط (logic 0 = pressed لأن ACTIVE_LOW)
 gpio-18  (                    |button-2            ) in  hi ACTIVE_LOW  # زرار مش مضغوط
 gpio-22  (                    |reset               ) out hi        # reset line مرفوعة (inactive)
```

```
gpio-4: out hi → kernel بيشوفه active (HIGH)
gpio-17: in lo ACTIVE_LOW → الـ physical pin = HIGH (لأن ACTIVE_LOW inversion)
          يعني الـ button مضغوط (active)
gpio-18: in hi ACTIVE_LOW → الـ physical pin = LOW
          يعني الـ button مش مضغوط (inactive)
```

```bash
$ cat /sys/kernel/debug/regulator/regulator_summary
 regulator                      use open bypass  min_uV  max_uV  voltage  current

 vddarm                           1    0      0  800000 1300000  1000000        0
  └──vddcore                      2    0      0  850000 1050000   950000        0
      └──vdd33                    3    0      0 3300000 3300000  3300000        0
          ├──vddio                1    0      0 1800000 3300000  3300000        0
          └──vddadc               0    0      0 3300000 3300000  3300000        0
```

```
الـ tree بيبين supply hierarchy:
- vddarm: 1 consumer، voltage = 1V (في الـ range 0.8-1.3V) ✓
- vddcore: 2 consumers، مربوط بـ vddarm كـ parent
- vddadc: 0 users → ممكن تعطله لو مش محتاجه (power saving)
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: GPIO Reset مش شغال على Industrial Gateway بـ AM62x

#### العنوان
**الـ Ethernet PHY مش بيتعمله reset صح على gateway صناعي**

#### السياق
شركة بتعمل industrial gateway بـ **TI AM62x** SoC. الـ gateway بيتحكم في خطوط إنتاج عن طريق Modbus over Ethernet. في مرحلة bring-up، الـ Ethernet interface مش بتظهر خالص — `ip link` مش بتشوف `eth0`.

#### المشكلة
الـ board مش بتستخدم Device Tree GPIO (لأن الـ BSP القديم platform data based)، فالمطور كتب lookup table في board file. بس الـ PHY driver بيطلب GPIO بـ `con_id = "reset"` وإيده فاضية.

```c
/* board file — AM62x platform */
static struct gpiod_lookup_table eth_phy_gpio_table = {
    .dev_id = "ethernet-phy",        /* wrong: driver uses different dev_id */
    .table = {
        GPIO_LOOKUP("gpio0", 14, "reset", GPIO_ACTIVE_LOW),
        {},
    },
};
```

#### التحليل
الـ `gpiod_lookup_table` بتتطابق على أساس `dev_id`. لما الـ PHY driver بيعمل `gpiod_get(dev, "reset", GPIOD_OUT_LOW)`، الكود بيدور في الـ list اللي اتضافت بـ `gpiod_add_lookup_table()`. بيقارن `dev_id` الـ table بـ `dev_name(dev)` — لو مش متطابقين تمام، مش هيلاقي الـ entry ده.

```c
struct gpiod_lookup_table {
    struct list_head list;
    const char *dev_id;      /* لازم يتطابق مع dev_name(dev) بالظبط */
    struct gpiod_lookup table[];
};
```

الـ `struct gpiod_lookup` اللي جواها:

```c
struct gpiod_lookup {
    const char *key;         /* "gpio0" — اسم الـ gpiochip */
    u16 chip_hwnum;          /* 14 — رقم الـ pin relative to chip */
    const char *con_id;      /* "reset" — لازم يتطابق مع ما بيطلبه الـ driver */
    unsigned int idx;        /* 0 */
    unsigned long flags;     /* GPIO_ACTIVE_LOW */
};
```

#### الحل
لازم تعرف الـ `dev_id` الصح بـ:

```bash
# شوف الـ device name اللي الـ PHY driver بيسجل بيه
dmesg | grep -i "phy\|ethernet"
ls /sys/bus/platform/devices/ | grep eth
```

لو الاسم طلع `30100000.ethernet-phy`:

```c
static struct gpiod_lookup_table eth_phy_gpio_table = {
    .dev_id = "30100000.ethernet-phy",  /* الاسم الصح من sysfs */
    .table = {
        GPIO_LOOKUP("gpio0", 14, "reset", GPIO_ACTIVE_LOW),
        {},
    },
};

static int __init board_init(void)
{
    gpiod_add_lookup_table(&eth_phy_gpio_table);
    return 0;
}
```

#### الدرس المستفاد
**الـ `dev_id` في `gpiod_lookup_table` لازم يتطابق حرفيًا مع `dev_name(dev)`** — مش اسم الـ driver، مش اسم الـ compatible string. دايمًا تتحقق من `/sys/bus/*/devices/` قبل ما تكتب الـ table.

---

### السيناريو 2: GPIO Hog بيأثر على SPI Flash على Android TV Box بـ Allwinner H616

#### العنوان
**الـ SPI NOR Flash مش بياخد commands صح بعد boot بسبب GPIO hog خاطئ**

#### السياق
Android TV box بـ **Allwinner H616** فيها SPI NOR flash بتخزن bootloader environment. بعد kernel boot، أي محاولة `mtd` read بتطلع `EIO`.

#### المشكلة
مطور تاني كتب GPIO hog لـ pin بيشارك مع SPI CS (chip select) وعمله output-high في الـ boot، فالـ flash مش بتتسيلكت خالص.

```c
/* hog table في board file أو arch init */
static struct gpiod_hog spi_hogs[] = {
    GPIO_HOG("pio", 68, "spi-cs-hog",
             GPIO_ACTIVE_HIGH,
             GPIOD_OUT_HIGH),   /* بيرفع الـ CS — يعني deselect دايم */
    {},
};
```

#### التحليل
الـ `struct gpiod_hog`:

```c
struct gpiod_hog {
    struct list_head list;
    const char *chip_label;  /* "pio" — Allwinner GPIO controller */
    u16 chip_hwnum;          /* 68 — نفس pin الـ SPI CS */
    const char *line_name;   /* اسم للـ hog */
    unsigned long lflags;    /* GPIO_ACTIVE_HIGH */
    int dflags;              /* GPIOD_OUT_HIGH — يعني بيرفعه دايم */
};
```

لما `gpiod_add_hogs()` بتتعمل call، الـ kernel بيعمل `gpiod_hog` على الـ line ده ويبقى owned بواسطة kernel — أي driver تاني بيحاول يطلبه بياخد `EBUSY`.

#### الحل

```bash
# تأكد إن الـ line متهوقة
gpioinfo | grep -i "spi\|hog\|68"
# هتلاقي: line 68: "spi-cs-hog" [kernel output-high]

# شيل الـ hog أو غير الـ pin number
```

الحل الصح: إما تشيل الـ hog ده خالص، أو لو محتاج GPIO hog لـ enable line تانية، تتأكد إنه pin مختلف:

```c
/* الـ hog الصح — pin 72 بدل 68 */
static struct gpiod_hog spi_hogs[] = {
    GPIO_HOG("pio", 72, "spi-power-en",
             GPIO_ACTIVE_HIGH,
             GPIOD_OUT_HIGH),
    {},
};

static int __init board_init(void)
{
    gpiod_add_hogs(spi_hogs);
    return 0;
}
```

#### الدرس المستفاد
**الـ GPIO hog بيعمل kernel takeover للـ line** — أي driver تاني مش هيقدر يطلبها. قبل ما تضيف `gpiod_hog`، تتأكد من pin mapping الـ SoC إن في H616 مفيش تعارض مع SPI/I2C/UART.

---

### السيناريو 3: GPIO_ACTIVE_LOW مفهوماش غلط على Reset Circuit في i.MX8 Automotive ECU

#### العنوان
**الـ CAN transceiver مش بيخرج من reset على ECU بـ i.MX8MP**

#### السياق
Automotive ECU بـ **i.MX8MP** بيكنترول CAN bus لـ body control module. الـ CAN transceiver بيحتاج active-low reset signal. المطور كتب lookup table وحدد `GPIO_ACTIVE_LOW`، بس الـ transceiver فاضي مش بيرسل frames.

#### المشكلة
المطور اعتقد إن `GPIO_ACTIVE_LOW` معناها إن الـ pin بيتحط low عند التشغيل. في الحقيقة، الـ flag بيأثر على `gpiod_set_value()` logic — لو حطيت value = 1 (assert)، الـ kernel بيحوله low على الحديدة.

```c
static struct gpiod_lookup_table can_gpio_table = {
    .dev_id = "3300000.flexcan",
    .table = {
        /* reset pin — active low على الـ schematic */
        GPIO_LOOKUP("gpio2", 5, "reset", GPIO_ACTIVE_LOW),
        {},
    },
};
```

الـ driver بيعمل:

```c
/* في الـ CAN driver */
gpiod = gpiod_get(dev, "reset", GPIOD_OUT_HIGH);
/* GPIOD_OUT_HIGH مع GPIO_ACTIVE_LOW = pin physically LOW = reset asserted */
```

بس المطور بعد كده عمل:

```c
gpiod_set_value(gpiod, 1); /* assert reset — صح */
msleep(10);
gpiod_set_value(gpiod, 0); /* deassert — المفروض يرفع الـ pin physically */
```

المشكلة إنه عمل `gpiod_get` بـ `GPIOD_OUT_HIGH` وابتدأ بـ reset asserted، بس الـ schematic محتاج الـ reset يتعمل deassert بعد power stable.

#### التحليل
الـ `flags` في `struct gpiod_lookup`:

```c
struct gpiod_lookup {
    const char *key;
    u16 chip_hwnum;
    const char *con_id;
    unsigned int idx;
    unsigned long flags;   /* GPIO_ACTIVE_LOW هنا بيعكس logic في gpiod_set_value */
};
```

الـ polarity inversion بتحصل في `gpiod` layer:
- `GPIO_ACTIVE_LOW` + `gpiod_set_value(gpiod, 1)` → pin = 0 (physically low)
- `GPIO_ACTIVE_LOW` + `gpiod_set_value(gpiod, 0)` → pin = 1 (physically high)

#### الحل

```bash
# تحقق من الـ pin state
gpioget gpiochip2 5
# لو 0 والـ chip في reset، يبقى الـ flag صح بس الـ timing غلط

# فحص الـ schematic: reset active low = chip في reset لما pin = 0
```

الحل بتغيير ترتيب الـ initialization:

```c
/* الـ driver code الصح */
gpiod = gpiod_get(dev, "reset", GPIOD_OUT_LOW);
/* GPIOD_OUT_LOW مع GPIO_ACTIVE_LOW = pin physically HIGH = no reset */
msleep(1);
gpiod_set_value(gpiod, 1); /* assert reset — pin goes LOW */
msleep(10);
gpiod_set_value(gpiod, 0); /* deassert — pin goes HIGH, chip operational */
```

#### الدرس المستفاد
**الـ `GPIO_ACTIVE_LOW` في الـ lookup table بيعكس كل قيم `gpiod_set_value()`** — الـ "1" معناه "asserting the signal" مش "voltage high". اقرأ الـ schematic، حدد الـ active level، وبعدين افهم كيف الـ `gpiod` API بيترجم ده.

---

### السيناريو 4: GPIO_LOOKUP_IDX لـ Multi-LED على IoT Sensor Board بـ STM32MP1

#### العنوان
**الـ LED driver بيضيء LED واحد بس من أصل 3 على IoT sensor board**

#### السياق
IoT sensor board بـ **STM32MP157** بيتحكم في 3 status LEDs (power, network, error). المطور كتب lookup table بـ `GPIO_LOOKUP` بدل `GPIO_LOOKUP_IDX`، فالـ driver بيلاقي الـ LED الأولى بس.

#### المشكلة

```c
/* كود خاطئ — كل entry بنفس الـ con_id بدون index */
static struct gpiod_lookup_table led_gpio_table = {
    .dev_id = "leds-gpio",
    .table = {
        GPIO_LOOKUP("gpioa", 13, "led", GPIO_ACTIVE_HIGH),
        GPIO_LOOKUP("gpioa", 14, "led", GPIO_ACTIVE_HIGH), /* مش هيتلاقي */
        GPIO_LOOKUP("gpiob", 2,  "led", GPIO_ACTIVE_HIGH), /* مش هيتلاقي */
        {},
    },
};
```

#### التحليل
الـ `GPIO_LOOKUP` macro:

```c
#define GPIO_LOOKUP(_key, _chip_hwnum, _con_id, _flags) \
    GPIO_LOOKUP_IDX(_key, _chip_hwnum, _con_id, 0, _flags)
```

لاحظ إن `idx = 0` دايمًا. الـ lookup code بيبحث بـ `{con_id, idx}` معًا، فالثلاث entries كلهم بنفس `{con_id="led", idx=0}` — بياخد الأول بس.

الـ struct الصح:

```c
struct gpiod_lookup {
    const char *key;
    u16 chip_hwnum;
    const char *con_id;
    unsigned int idx;    /* الفرق هنا — لازم 0, 1, 2 */
    unsigned long flags;
};
```

والـ macro الصح:

```c
/*
 * GPIO_LOOKUP_IDX — لما عندك أكتر من GPIO بنفس con_id
 */
#define GPIO_LOOKUP_IDX(_key, _chip_hwnum, _con_id, _idx, _flags)  \
(struct gpiod_lookup) {                                             \
    .key        = _key,                                             \
    .chip_hwnum = _chip_hwnum,                                      \
    .con_id     = _con_id,                                          \
    .idx        = _idx,         /* المهم: index مختلف لكل GPIO */  \
    .flags      = _flags,                                           \
}
```

#### الحل

```c
/* كود صح — باستخدام GPIO_LOOKUP_IDX */
static struct gpiod_lookup_table led_gpio_table = {
    .dev_id = "leds-gpio",
    .table = {
        GPIO_LOOKUP_IDX("gpioa", 13, "led", 0, GPIO_ACTIVE_HIGH), /* power LED */
        GPIO_LOOKUP_IDX("gpioa", 14, "led", 1, GPIO_ACTIVE_HIGH), /* network LED */
        GPIO_LOOKUP_IDX("gpiob", 2,  "led", 2, GPIO_ACTIVE_HIGH), /* error LED */
        {},
    },
};
```

في الـ driver:

```c
/* يطلب كل LED بـ index */
gpiod_power   = gpiod_get_index(dev, "led", 0, GPIOD_OUT_LOW);
gpiod_network = gpiod_get_index(dev, "led", 1, GPIOD_OUT_LOW);
gpiod_error   = gpiod_get_index(dev, "led", 2, GPIOD_OUT_LOW);
```

#### الدرس المستفاد
**`GPIO_LOOKUP` دايمًا بيحط `idx=0`** — لو عندك أكتر من GPIO بنفس الاسم (زي LED array أو channel array)، لازم `GPIO_LOOKUP_IDX` مع index مختلف لكل واحدة. الـ driver كمان لازم يستخدم `gpiod_get_index()`.

---

### السيناريو 5: إزالة Lookup Table وقت module unload على Custom Board بـ RK3562

#### العنوان
**kernel panic عند reload driver على custom RK3562 board بسبب lookup table مش اتشالت**

#### السياق
Custom industrial board بـ **RK3562** فيها USB hub يتكنترول بـ GPIO power enable. المطور عمل driver كـ loadable module. عند `rmmod` و`insmod` تاني، الـ kernel بيعمل panic أو بيرجع `EBUSY`.

#### المشكلة
الـ module `init` بتضيف الـ lookup table، بس الـ module `exit` منساش يشيلها. عند إعادة التحميل، الـ table بتتضاف تاني مع وجود entry القديمة على نفس list node — corruption في الـ linked list.

```c
/* كود ناقص — exit بدون gpiod_remove_lookup_table */
static struct gpiod_lookup_table usb_hub_gpio_table = {
    .dev_id = "xhci-hcd.0",
    .table = {
        GPIO_LOOKUP("gpio0", 28, "power", GPIO_ACTIVE_HIGH),
        {},
    },
};

static int __init usb_hub_gpio_init(void)
{
    gpiod_add_lookup_table(&usb_hub_gpio_table);
    return 0;
}

static void __exit usb_hub_gpio_exit(void)
{
    /* المشكلة: مفيش gpiod_remove_lookup_table */
}

module_init(usb_hub_gpio_init);
module_exit(usb_hub_gpio_exit);
```

#### التحليل
الـ `gpiod_lookup_table` بيتضاف لـ global linked list:

```c
struct gpiod_lookup_table {
    struct list_head list;   /* member في global lookup list */
    const char *dev_id;
    struct gpiod_lookup table[];
};
```

الـ `list_head` embedded في الـ struct. لما الـ module بيتشال، الذاكرة بتتحرر. لما بيتحط تاني، الـ struct بيتكتب على نفس address — بس الـ list القديم لسه فيه pointer لعنوان قديم. نتيجة: list corruption وkernel panic.

#### الحل

```c
static int __init usb_hub_gpio_init(void)
{
    gpiod_add_lookup_table(&usb_hub_gpio_table);
    return 0;
}

static void __exit usb_hub_gpio_exit(void)
{
    /* لازم دايمًا تشيل الـ table قبل unload */
    gpiod_remove_lookup_table(&usb_hub_gpio_table);
}

module_init(usb_hub_gpio_init);
module_exit(usb_hub_gpio_exit);
```

التحقق:

```bash
# قبل rmmod
cat /sys/kernel/debug/gpio  # شوف الـ GPIO states

# بعد rmmod وinsmod
dmesg | tail -20  # تأكد مفيش warnings
ls /sys/class/gpio/  # تأكد الـ GPIOs اتحررت
```

لو بتستخدم `gpiod_add_hogs()` كمان، نفس القاعدة:

```c
static void __exit board_exit(void)
{
    gpiod_remove_hogs(board_hogs);      /* لـ hogs */
    gpiod_remove_lookup_table(&table);  /* لـ lookup tables */
}
```

#### الدرس المستفاد
**كل `gpiod_add_lookup_table()` لازم يقابله `gpiod_remove_lookup_table()` في الـ exit path** — خصوصًا في loadable modules. نفس الكلام لـ `gpiod_add_hogs()` / `gpiod_remove_hogs()`. الـ `list_head` في الـ struct بيعمل tight coupling مع الذاكرة — لو الذاكرة اتحررت والـ node لسه في الـ list، kernel panic مضمون.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

الـ LWN.net هو المرجع الأساسي لمتابعة تطور kernel internals. المقالات دي بتغطي مباشرةً التحول من board files و `machine_desc` التقليدية لنظام device tree الحديث اللي بيشتغل معاه `mips_machine` و `machine_features`.

| المقال | الأهمية |
|--------|---------|
| [Supporting multi-platform ARM kernels](https://lwn.net/Articles/496400/) | شرح ازاي الـ ARM kernel بدأ يدعم أكتر من board في image واحدة عن طريق إزالة board files واستبدالها بـ device tree |
| [Basic ARM device tree support](https://lwn.net/Articles/423607/) | أول دعم حقيقي لـ DT على ARM، شرح كيفية matching بين `machine_desc` وـ `compatible` property |
| [Device-tree support for ARM](https://lwn.net/Articles/367752/) | أولى الباتشات اللي جابت DT support للـ ARM architecture |
| [Platform devices and device trees](https://lwn.net/Articles/448502/) | ازاي الـ `platform_device` الـ static بدأ يتزال ويتعوض بـ DT nodes |
| [One Kernel For All](https://lwn.net/Articles/900782/) | الرؤية الكاملة لـ multi-platform kernel اللي فيه الـ machine detection بيتم runtime |
| [ACPI for ARM?](https://lwn.net/Articles/574439/) | نقاش البديل الآخر لـ board files على ARM: ACPI vs Device Tree |

---

### توثيق الـ Kernel الرسمي

الملفات دي موجودة في شجرة الـ kernel تحت `Documentation/`:

```
Documentation/devicetree/usage-model.rst
    شرح كامل لنموذج استخدام device tree في Linux، بما فيه
    كيفية اختيار الـ machine_desc المناسب من compatible property.

Documentation/arch/mips/
    توثيق architecture MIPS بشكل عام.

Documentation/process/submitting-patches.rst
    مرجع لطريقة إرسال patches للـ MIPS subsystem.
```

الـ online version:
- [Linux and the Devicetree — kernel.org](https://docs.kernel.org/devicetree/usage-model.html)
- [CPU Architectures — kernel.org](https://static.lwn.net/kerneldoc/arch/index.html)

---

### Source Files الأساسية في الـ Kernel

الـ `machine.h` مش ملف واحد — في الـ kernel فيه عدة تنفيذات حسب الـ architecture:

```
arch/mips/include/asm/machine.h
    تعريف struct mips_machine، ماكرو MIPS_MACHINE()،
    وـ for_each_mips_machine() — ده الملف الرئيسي للموضوع.

arch/s390/include/asm/machine.h
    نهج مختلف: machine_features[] كـ bitmap لـ runtime feature detection
    على IBM mainframes.

arch/mips/include/asm/mach-loongson2ef/machine.h
    override للـ MIPS machine definitions لمعالجات Loongson.

include/linux/gpio/machine.h
    مرجع لنمط مشابه: machine-level GPIO lookups.

include/linux/regulator/machine.h
    machine-level constraints للـ voltage regulators.
```

---

### Commits المهمة

#### تقديم الـ MIPS Generic Machine Infrastructure

الـ patch series الأصلي اللي أنشأ `arch/mips/include/asm/machine.h`:

- **[PATCH v3 00/18] MIPS generic kernels, SEAD-3 & Boston support**
  Paul Burton — Imagination Technologies، 2016
  [lore.kernel.org](https://lore.kernel.org/all/4317168.PHWzNCbrtz@np-p-burton/T/)

  > ده الـ series اللي قدّم `struct mips_machine`، ماكرو `MIPS_MACHINE()`،
  > وـ section `.mips.machines.init` لأول مرة في الـ kernel.

- **[PATCH 22/26] MIPS: generic: Introduce generic DT-based board support**
  [lore.kernel.org](https://lore.kernel.org/lkml/20160826153725.11629-23-paul.burton@imgtec.com/)

---

### نقاشات الـ Mailing List

| الموضوع | الرابط |
|---------|--------|
| MIPS generic kernels v2 — 26 patches | [lkml.iu.edu](https://lkml.iu.edu/hypermail/linux/kernel/1608.3/03844.html) |
| MAINTAINERS: Update linux-mips mailing list | [lore.kernel.org](https://lore.kernel.org/lkml/20181130201120.15903-1-paul.burton@mips.com/) |
| MIPS: DTS: img: Marduk board device tree | [lore.kernel.org](https://lore.kernel.org/linux-devicetree/57FF3172.4010709@imgtec.com/) |

**الـ Mailing Lists النشطة:**
- `linux-mips@vger.kernel.org` — للـ MIPS architecture
- `devicetree@vger.kernel.org` — للـ device tree و FDT
- `linux-kernel@vger.kernel.org` — للنقاشات العامة

---

### كتب مُوصى بيها

#### Linux Device Drivers, 3rd Edition (LDD3)
المرجع المجاني الأشهر لكتابة drivers:

- الفصول المهمة: **Chapter 14** (The Linux Device Model) — بيشرح `bus_type`، `device`، `driver` وازاي بيترابطوا، وده الأساس اللي بُني عليه نظام machine detection.
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **Chapter 19**: Portability — بيتكلم عن اختلافات الـ architectures وازاي الـ kernel بيتعامل معاها.
- **Chapter 17**: Devices and Modules — بيشرح الـ device model اللي بيشتغل فوقيه الـ machine layer.

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **Chapter 4**: The Linux Kernel — بيشرح boot sequence وازاي الـ machine detection بيحصل من الـ bootloader.
- **Chapter 16**: Porting Linux — بيغطي نقل الـ kernel لـ hardware جديد وإنشاء board support package.

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- **Chapter 1**: Introduction and Overview — architecture overview مع تفاصيل فرق الـ platforms.

---

### KernelNewbies.org

الـ KernelNewbies بيوثق التغييرات في كل version، ومنه تتابع إضافة الـ machine support لكل kernel release:

- [Linux 5.19 — New ARM/MIPS machines](https://kernelnewbies.org/Linux_5.19) — أُضيف فيه دعم لـ 30+ board جديدة
- [Linux 6.13 — ARM CCA support](https://kernelnewbies.org/Linux_6.13) — دعم الـ Confidential Compute Architecture على ARM
- [Linux 6.15 — ARM Morello devicetree](https://kernelnewbies.org/Linux_6.15) — أُضيف الـ DT لـ Arm Morello platform
- [Linux 2.6.20 — Early ARM machine additions](https://kernelnewbies.org/Linux_2_6_20) — من أوائل الـ machine additions الـ documented

---

### eLinux.org

| الصفحة | المحتوى |
|--------|---------|
| [Device Tree Reference](https://elinux.org/Device_Tree_Reference) | مرجع شامل لصياغة DTS وـ compatible strings |
| [ARM Versatile](https://elinux.org/ARM_Versatile) | مثال حي على board قديمة بـ machine descriptor |
| [QEMUonARM](https://elinux.org/QEMUonARM) | تشغيل ARM machines في QEMU للتجربة بدون hardware |

---

### search terms للبحث عن معلومات أكثر

```
# للبحث في الـ kernel source
git log --all --oneline -- arch/mips/include/asm/machine.h
git log --all --oneline -- arch/s390/include/asm/machine.h
git log --grep="mips_machine" --oneline

# للبحث في الـ mailing list
site:lore.kernel.org "mips_machine"
site:lore.kernel.org "struct mips_machine"
site:lore.kernel.org "MIPS_MACHINE"

# كلمات البحث المفيدة
"mips generic kernel" device tree board
"machine_desc" ARM multiplatform kernel
"machine_features" s390 alternative patching
linux kernel "mips_machine_is_compatible" fdt
".mips.machines.init" linker section kernel
"for_each_mips_machine" linux kernel
"machine_has_" s390 linux feature detection
"apply_mips_fdt_fixups" device tree fixup
```

---

### جدول ملخص المراجع

| النوع | المصدر | الأهمية |
|-------|--------|---------|
| مقال | [LWN: Multi-platform ARM](https://lwn.net/Articles/496400/) | عالية جداً |
| مقال | [LWN: Basic ARM DT support](https://lwn.net/Articles/423607/) | عالية جداً |
| مقال | [LWN: Platform devices and DT](https://lwn.net/Articles/448502/) | عالية |
| commit | [MIPS generic kernels patch series](https://lore.kernel.org/all/4317168.PHWzNCbrtz@np-p-burton/T/) | عالية جداً |
| docs | [kernel.org: Linux and DT](https://docs.kernel.org/devicetree/usage-model.html) | عالية جداً |
| wiki | [eLinux: Device Tree Reference](https://elinux.org/Device_Tree_Reference) | متوسطة |
| wiki | [eLinux: ARM Versatile](https://elinux.org/ARM_Versatile) | متوسطة |
| كتاب | LDD3 — Chapter 14 | عالية |
| كتاب | Linux Kernel Development — Ch. 19 | متوسطة |
| كتاب | Embedded Linux Primer — Ch. 4, 16 | عالية للـ embedded |
## Phase 8: Writing simple module

### الفكرة

**الـ** `gpiod_add_lookup_table` هي دالة exported من `include/linux/gpio/machine.h` — بتضيف `gpiod_lookup_table` للـ global list اللي الـ GPIO subsystem بيبحث فيها لما device بتطلب GPIO بالاسم. هنعمل kprobe عليها عشان نطبع اسم الـ device اللي بتسجّل الـ lookup table وعدد الـ entries جوّاها.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_gpio_lookup.c
 *
 * Hooks gpiod_add_lookup_table() to log every platform device
 * that registers a GPIO lookup table at runtime.
 */

#include <linux/kernel.h>       /* pr_info, pr_err */
#include <linux/module.h>       /* MODULE_* macros, module_init/exit */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe */
#include <linux/gpio/machine.h> /* struct gpiod_lookup_table, struct gpiod_lookup */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Doc Project");
MODULE_DESCRIPTION("kprobe on gpiod_add_lookup_table — log GPIO lookup table registrations");

/* ------------------------------------------------------------------ */
/* pre_handler: runs just before gpiod_add_lookup_table() executes     */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64 the first argument lands in RDI.
     * On ARM64 it lands in X0.
     * kprobes gives us pt_regs; we cast the right register.
     */
#if defined(CONFIG_X86_64)
    struct gpiod_lookup_table *tbl =
        (struct gpiod_lookup_table *)regs->di;
#elif defined(CONFIG_ARM64)
    struct gpiod_lookup_table *tbl =
        (struct gpiod_lookup_table *)regs->regs[0];
#else
    /* Fallback — not supported on this arch, bail out silently */
    return 0;
#endif

    if (!tbl)
        return 0;

    /* Count valid entries (sentinel has key == NULL) */
    int count = 0;
    while (tbl->table[count].key != NULL)
        count++;

    pr_info("[gpio_lookup_probe] dev_id=\"%s\"  entries=%d\n",
            tbl->dev_id ? tbl->dev_id : "(null)",
            count);

    return 0; /* 0 = continue normal execution */
}

/* ------------------------------------------------------------------ */
/* kprobe descriptor                                                    */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "gpiod_add_lookup_table",
    .pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ */
/* module_init                                                          */
/* ------------------------------------------------------------------ */
static int __init gpio_lookup_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[gpio_lookup_probe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[gpio_lookup_probe] planted on %s @ %p\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit                                                          */
/* ------------------------------------------------------------------ */
static void __exit gpio_lookup_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("[gpio_lookup_probe] removed\n");
}

module_init(gpio_lookup_probe_init);
module_exit(gpio_lookup_probe_exit);
```

---

### Makefile

```makefile
obj-m += kprobe_gpio_lookup.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

```bash
make
sudo insmod kprobe_gpio_lookup.ko
dmesg | grep gpio_lookup_probe
sudo rmmod kprobe_gpio_lookup
```

---

### شرح كل جزء

#### الـ includes

```c
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe */
#include <linux/gpio/machine.h> /* struct gpiod_lookup_table */
```

**الـ** `kprobes.h` بيجيب كل الـ API الخاص بالـ dynamic instrumentation.
**الـ** `gpio/machine.h` محتاجينه عشان نعرف شكل الـ `gpiod_lookup_table` ونقدر نقرا الـ `dev_id` والـ `table[]` بأمان.

---

#### الـ `handler_pre`

```c
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
```

الـ kprobe بيشغّل الـ handler ده قبل ما الدالة الأصلية تتنفّذ — يعني إحنا بنشوف الـ arguments قبل ما أي تعديل يحصل جوّاها.
الـ `pt_regs` بيحمل state الـ CPU كامل في لحظة الاعتراض، وبنجيب الـ argument الأول (الـ pointer للـ table) من الـ register المناسب حسب الـ architecture.

---

#### قراءة الـ `pt_regs`

```c
#if defined(CONFIG_X86_64)
    struct gpiod_lookup_table *tbl = (struct gpiod_lookup_table *)regs->di;
#elif defined(CONFIG_ARM64)
    struct gpiod_lookup_table *tbl = (struct gpiod_lookup_table *)regs->regs[0];
#endif
```

الـ calling convention بيختلف بين الـ architectures — على x86-64 الأول argument في `RDI`، وعلى ARM64 في `X0`.
الـ `#if` ضروري عشان نفس الكود يشتغل صح على الاتنين من غير undefined behavior.

---

#### عدّ الـ entries

```c
while (tbl->table[count].key != NULL)
    count++;
```

الـ `gpiod_lookup_table` بتنتهي بـ sentinel entry فيها `key == NULL` — نفس فكرة الـ null-terminated string.
بنعدّ العناصر دي عشان نعطي context مفيد في الـ log — بنعرف فين الأجهزة اللي بتسجّل GPIOs كتير.

---

#### الـ `pr_info` output

```c
pr_info("[gpio_lookup_probe] dev_id=\"%s\"  entries=%d\n", ...);
```

**الـ** `pr_info` بيطبع في الـ kernel ring buffer — تتابعه بـ `dmesg`.
بنطبع الـ `dev_id` عشان نعرف اسم الـ device اللي بتسجّل الـ table، وعدد الـ entries عشان نفهم حجم الـ GPIO mapping بتاعتها.

---

#### الـ `register_kprobe` / `unregister_kprobe`

```c
ret = register_kprobe(&kp);   /* في init */
unregister_kprobe(&kp);       /* في exit */
```

`register_kprobe` بيكتب breakpoint instruction في الذاكرة عند عنوان الدالة المستهدفة — من غير أي تعديل في الـ source أو recompile.
`unregister_kprobe` في الـ exit ضروري جداً عشان يشيل الـ breakpoint ويرجّع الكود الأصلي — لو سبناه هيحصل undefined behavior أو kernel panic لما الـ handler يتنفّذ بعد ما الـ module اتشيل من الذاكرة.

---

### مثال على الـ output

```
[gpio_lookup_probe] planted on gpiod_add_lookup_table @ ffffffffc03a1200
[gpio_lookup_probe] dev_id="spi0.0"   entries=3
[gpio_lookup_probe] dev_id="leds"     entries=5
[gpio_lookup_probe] removed
```

**الـ** output ده بيوضّح بالاسم كل device سجّلت GPIO lookup table — مفيد لتشخيص conflicts وفهم الـ GPIO wiring على الـ board.
