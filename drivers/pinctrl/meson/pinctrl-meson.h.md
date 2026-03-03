## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الملف ده جزء من **ARM/Amlogic Meson SoC support** subsystem، اللي بيتضمن كل حاجة خاصة بـ SoCs من شركة **Amlogic** (زي اللي بتلاقيها في Raspberry Pi-like boxes وتليفزيونات Android وـ set-top boxes). المسؤولين عنه هم Neil Armstrong وKevin Hilman من Linaro وBayLibre.

---

### الصورة الكبيرة — تخيل المشكلة

تخيل عندك لوحة إلكترونية (SoC) فيها مئات الـ **pins** — زي أرجل الحشرة. كل رجل ممكن تشتغل بأكتر من طريقة:

- ممكن تكون **GPIO** (دخل أو خرج رقمي بسيط — زي ضغطة زرار أو إضاءة LED).
- ممكن تكون **UART TX** (إرسال بيانات سيريال).
- ممكن تكون **I2C SDA** (بيانات I2C).
- ممكن تكون **SPI CLK** (ساعة SPI).

الـ SoC بيحتاج **مدير** يقول لكل pin: "انت هتشتغل إزاي؟" — ده بالظبط شغل الـ **Pin Controller**.

---

### قصة المشكلة — من الواقع

تصور إنك بتصمم **Android TV box** بـ Amlogic Meson SoC:

1. الـ WiFi chip محتاج SDIO (بيستخدم pins معينة في bank X).
2. الـ HDMI محتاج I2C لـ DDC (بيستخدم pins في bank H).
3. في نفس الوقت، المطور عايز يكنترول LED بـ GPIO من bank A.

الـ kernel لازم يعرف:
- فين الـ register اللي يغير مود الـ pin ده؟
- إيه الـ bit اللي لازم يتسيت؟
- لو عايز pull-up يعمل إيه؟
- لو عايز يقرأ قيمة الـ pin يروح فين في الـ registers؟

هنا بييجي دور ملف `pinctrl-meson.h` — هو **العقد المشترك** (shared contract) اللي كل الـ hardware-specific drivers بتكلمه.

---

### الهدف من الملف

**الـ `pinctrl-meson.h`** هو الـ **header الأساسي** للـ pinctrl framework الخاص بـ Amlogic Meson SoCs. مش بيتنفذ حاجة بذاتها، لكنه بيعرّف:

| العنصر | الغرض |
|---|---|
| `struct meson_pmx_group` | مجموعة pins بتشتغل مع بعض لوظيفة واحدة |
| `struct meson_pmx_func` | وظيفة كاملة (زي UART) مكونة من مجموعات |
| `struct meson_reg_desc` | وصف register واحد (offset + bit) |
| `enum meson_reg_type` | أنواع الـ registers: pull، direction، output، input، drive strength |
| `enum meson_pinconf_drv` | قيم drive strength المدعومة (500µA لـ 4000µA) |
| `struct meson_bank` | بنك من الـ pins مع كل الـ registers بتاعته |
| `struct meson_pinctrl_data` | البيانات الكاملة لـ SoC معين (pins، groups، funcs، banks) |
| `struct meson_pinctrl` | الـ instance الكاملة للـ driver وقت التشغيل |
| Macros (`BANK`, `BANK_DS`, `FUNCTION`, `MESON_PIN`) | تسهيل تعريف البيانات في كل SoC driver |

---

### كيف بيشتغل النظام — بالتخيل

تخيل الـ SoC زي **فندق كبير**:

- الـ **Bank** = طابق في الفندق (مثلاً Bank A، Bank X، Bank AO).
- الـ **Pin** = أوضة في الطابق.
- الـ **Function** = خدمة معينة (WiFi، UART، I2C).
- الـ **Group** = مجموعة أوض بتقدم نفس الخدمة مع بعض.
- الـ **Register** = مفتاح كهربائي وراء الكاونتر — كل مفتاح بيتحكم في حاجة معينة (اتجاه، pull، قيمة).

الـ `meson_pinctrl_data` هو **دليل الفندق** — بيقول فيه كل طابق فيه كام أوضة وأوضة رقم كام فيها إيه.

لما الـ kernel يجي يقول "عايز أحول pin 42 لـ UART TX"، الـ driver يبص في الدليل، يلاقي الـ group المناسب، يكتب في الـ register الصح الـ bit الصح — وخلص.

---

### الـ AO Bank — حالة خاصة مهمة

الـ **AO (Always-On) bank** مختلف عن باقي البنوك لأنه في **power domain منفصل** — مش بيتقفل حتى لو الـ SoC دخل sleep mode. ده معناه إن registers بتاعته في عنوان مختلف في الـ memory map، وبعض الـ SoCs القديمة (قبل G12A) بتحتاج parsing خاصة من Device Tree — ولذلك في function منفصلة `meson8_aobus_parse_dt_extra()`.

---

### الـ Drive Strength

الـ `enum meson_pinconf_drv` بيحدد قوة الـ output driver:
- **500µA** — أضعف، لو الـ signal قصير ومفيش load كبير.
- **4000µA** — أقوى، لو في load أو الـ signal طويل.

ده بيأثر على سرعة الـ signal وجودته وإستهلاك الطاقة.

---

### الملفات اللي لازم تعرفها

#### الـ Core (مشترك بين كل الـ SoCs)

| الملف | الدور |
|---|---|
| `drivers/pinctrl/meson/pinctrl-meson.h` | **الملف ده** — الـ shared data structures |
| `drivers/pinctrl/meson/pinctrl-meson.c` | الـ core implementation: probe، GPIO ops، pinconf، pinmux |
| `drivers/pinctrl/meson/pinctrl-meson8-pmx.c` | pinmux logic للـ Meson8/8b القديم |
| `drivers/pinctrl/meson/pinctrl-meson8-pmx.h` | header الـ Meson8 pmx |
| `drivers/pinctrl/meson/pinctrl-meson-axg-pmx.c` | pinmux logic لـ AXG وما بعده |
| `drivers/pinctrl/meson/pinctrl-meson-axg-pmx.h` | header الـ AXG pmx |

#### الـ SoC-specific Drivers (كل SoC بيعرّف pins وbanks وfunctions خاصة بيه)

| الملف | الـ SoC |
|---|---|
| `pinctrl-meson8.c` | Meson8 (S802) |
| `pinctrl-meson8b.c` | Meson8b (S905 القديم) |
| `pinctrl-meson-gxbb.c` | GXBB (S905) |
| `pinctrl-meson-gxl.c` | GXL (S905X/D) |
| `pinctrl-meson-axg.c` | AXG (A113) |
| `pinctrl-meson-g12a.c` | G12A/G12B (S905X2/X3) |
| `pinctrl-meson-a1.c` | A1 |
| `pinctrl-meson-s4.c` | S4 |
| `pinctrl-amlogic-c3.c` | C3 |
| `pinctrl-amlogic-t7.c` | T7 |
| `pinctrl-amlogic-a4.c` | A4 |

#### الـ Kernel Framework Headers

| الملف | الدور |
|---|---|
| `include/linux/pinctrl/pinctrl.h` | الـ core pinctrl API — `pinctrl_pin_desc`، `pinctrl_dev` |
| `include/linux/gpio/driver.h` | الـ GPIO chip interface |
| `include/linux/regmap.h` | abstraction للـ register access |

#### الـ Device Tree Bindings

| الملف | الدور |
|---|---|
| `Documentation/devicetree/bindings/pinctrl/amlogic,meson*` | DT schema لكل SoC |
| `arch/arm64/boot/dts/amlogic/` | الـ DTS files الفعلية |
## Phase 2: شرح الـ Pinctrl Framework

### المشكلة — ليه الـ Pinctrl موجود؟

أي SoC حديث زي Amlogic Meson فيه مئات الـ **pins** على الـ chip. كل pin ممكن يشتغل في أكتر من وظيفة:
- يكون GPIO عادي (input/output)
- يكون جزء من واجهة UART
- يكون جزء من واجهة I2C أو SPI
- يكون data line لـ EMMC أو SDIO
- وهكذا...

قبل الـ pinctrl subsystem، كل driver كان بيتحكم في الـ pins بتاعته بنفسه — بيكتب مباشرة على registers الـ mux. النتيجة؟

- **تعارض**: درايفران يحاولوا يستخدموا نفس الـ pin في نفس الوقت.
- **تشتيت**: كود الـ pin configuration موزع على كل درايفر بشكل مختلف.
- **عدم قابلية النقل**: كل SoC عنده addresses مختلفة — محدش يعرف يعمل generic code.
- **غياب الـ policy**: مين الأولى بالـ pin لو اتطلب من جهتين؟

الـ **pinctrl subsystem** اتعمل عشان يحل الفوضى دي بـ centralized pin management.

---

### الحل — الـ Kernel بيتعامل إزاي مع الموضوع؟

الكيرنل قسّم المشكلة لـ 3 طبقات مستقلة:

| الطبقة | اسمها | المسؤولية |
|--------|-------|------------|
| 1 | **Pin Enumeration** | تسجيل كل pin باسم ورقم فريد |
| 2 | **Pin Muxing (pinmux)** | تحديد الـ function اللي كل pin بيشتغل فيه |
| 3 | **Pin Configuration (pinconf)** | ضبط خصائص الـ pin: pull-up/down, drive strength |

وعمل abstraction layer واحد بين الكلام ده كله والـ drivers الباقيين.

---

### الـ Big Picture — فين الـ Pinctrl في الكيرنل؟

```
+----------------------------------------------------------+
|                    Kernel Subsystems                     |
|  +----------+  +----------+  +---------+  +----------+  |
|  | UART drv |  | I2C drv  |  | SPI drv |  | GPIO drv |  |
|  +----+-----+  +----+-----+  +----+----+  +----+-----+  |
|       |              |            |             |         |
|       +------+-------+------------+-------------+         |
|              |                                            |
|       +------v---------+                                  |
|       |  pinctrl core  |  <-- kernel/drivers/pinctrl/     |
|       | (pinctrl.c,    |      subsystem core               |
|       |  pinmux.c,     |                                  |
|       |  pinconf.c)    |                                  |
|       +------+---------+                                  |
|              |                                            |
|    +---------v-----------+                                |
|    | pinctrl_desc (ops)  |  <-- الـ vtable اللي الدرايفر  |
|    | .pctlops            |      بيملأه                    |
|    | .pmxops             |                                |
|    | .confops            |                                |
|    +---------+-----------+                                |
|              |                                            |
|   +----------v----------------------------+               |
|   |  meson_pinctrl driver                |               |
|   |  (pinctrl-meson.c + pinctrl-meson.h) |               |
|   +----------+---------------------------+               |
|              |                                            |
|    +---------v----------+                                 |
|    |  regmap abstraction|  <-- بيتكلم مع الـ registers    |
|    +---------+----------+                                 |
|              |                                            |
|   +----------v-----------+                                |
|   | Meson SoC Pin HW     |                               |
|   | MUX regs, GPIO regs  |                               |
|   | PULL regs, DS regs   |                               |
|   +----------------------+                                |
+----------------------------------------------------------+
```

الـ **pinctrl core** في المنتصف — بيستقبل طلبات من الـ consumers (UART, I2C, ...) وبينفذها عبر الـ driver اللي بيعرف HW الحقيقي.

---

### Real-World Analogy — الـ Switchboard في شركة اتصالات

تخيل شركة فيها 200 خط تليفون وعندها:
- قسم مبيعات محتاج 20 خط
- قسم دعم فني محتاج 30 خط
- قسم إداري محتاج 10 خطوط

الـ **switchboard operator** (= pinctrl core) هو اللي:
- بيعرف كل الخطوط الموجودة وأرقامها (**pin enumeration**)
- بيوصّل الخط للقسم الصح وقت الطلب (**pinmux**)
- بيضبط جودة الخط: صوت عالي/منخفض، مع/بدون hold music (**pinconf**)
- بيمنع قسمين يحجزوا نفس الخط (**conflict resolution**)

| عنصر في المثل | المقابل في الكيرنل |
|--------------|-------------------|
| خط تليفون | `pin` |
| رقم الخط | `pin number` في `pinctrl_pin_desc` |
| القسم (مبيعات/دعم) | `function` في `meson_pmx_func` |
| مجموعة خطوط للقسم | `group` في `meson_pmx_group` |
| الـ switchboard | `pinctrl_dev` (الـ core instance) |
| لوحة التوصيلات الفيزيائية | الـ SoC HW registers |
| الـ operator | `pinctrl_ops` + `pinmux_ops` vtables |
| ضبط جودة الخط | `pinconf_ops` + `meson_reg_desc` |
| دفتر العناوين | `meson_pinctrl_data` (static table) |

---

### الـ Core Abstraction — الفكرة المحورية

الـ pinctrl subsystem قائم على فكرة **"function → groups → pins"**:

```
Function: "uart_a"
    |
    +-- Group: "uart_a_tx_group"  --> pins: [PIN_UART_TX]
    |
    +-- Group: "uart_a_rx_group"  --> pins: [PIN_UART_RX]

Function: "i2c_b"
    |
    +-- Group: "i2c_b_group"  --> pins: [PIN_I2C_SDA, PIN_I2C_SCL]
```

**الـ function** هو الوظيفة الوظيفية (UART, I2C, SPI...).
**الـ group** هو المجموعة الفيزيائية من الـ pins اللي لازم تتفعّل مع بعض.
**الـ pin** هو الوحدة الصغيرة — رقم فريد يمثّل pad فيزيائي على الـ chip.

اللي بيميّز Meson إن عنده كمان مفهوم **bank**: مجموعة من الـ pins بتتحكم فيها نفس مجموعة الـ registers — يعني bit offset بيتحسب relative لأول pin في الـ bank.

---

### تفصيل الـ Structs وعلاقتها ببعض

#### 1. `pinctrl_pin_desc` — الوحدة الأساسية

```c
struct pinctrl_pin_desc {
    unsigned int number;   /* رقم فريد globally */
    const char   *name;    /* اسم للـ debugfs والـ DT */
    void         *drv_data;/* بيانات خاصة بالدرايفر */
};
```

في Meson بيتعمل بالماكرو:
```c
#define MESON_PIN(x) PINCTRL_PIN(x, #x)
// مثال: MESON_PIN(GPIOAO_0) --> { .number = GPIOAO_0, .name = "GPIOAO_0" }
```

#### 2. `meson_pmx_group` — مجموعة الـ Pins

```c
struct meson_pmx_group {
    const char          *name;     /* "uart_a_tx_group" */
    const unsigned int  *pins;     /* array of pin numbers */
    unsigned int         num_pins; /* عدد الـ pins */
    const void          *data;     /* HW-specific data (reg/bit للـ mux) */
};
```

الـ `data` pointer ده generic — الـ SoC-specific driver هو اللي يفسّره. في بعض Meson SoCs بيبقى pointer على struct بيحدد الـ register والـ bit اللي لازم يتكتب عليه عشان تفعّل الـ function دي.

#### 3. `meson_pmx_func` — الـ Function

```c
struct meson_pmx_func {
    const char          *name;       /* "uart_a" */
    const char * const  *groups;     /* أسماء الـ groups المنتمية للـ function */
    unsigned int         num_groups;
};
```

ملاحظة: الـ func بيشير للـ groups بـ **أسماء** مش بـ pointers — الـ core بيعمل lookup بالاسم.

#### 4. `meson_reg_desc` — وصف الـ Register

```c
struct meson_reg_desc {
    unsigned int reg; /* offset في الـ regmap */
    unsigned int bit; /* رقم الـ bit */
};
```

ده أبسط struct في الملف لكنه الأساس — كل pin configuration (pull, direction, output, input, drive-strength) بيتعمل عبر write/read على `(reg, bit)` محددين.

#### 5. `meson_bank` — البنك

```c
struct meson_bank {
    const char          *name;
    unsigned int         first;    /* أول pin number في البنك */
    unsigned int         last;     /* آخر pin number */
    int                  irq_first;/* أول hwirq offset */
    int                  irq_last;
    struct meson_reg_desc regs[MESON_NUM_REG]; /* regs للـ pin الأول فقط! */
};
```

**نقطة مهمة جداً**: الـ `regs` array بتشاور على الـ pin الأول في البنك. لو عايز تتحكم في pin رقم N في البنك، بتضيف `(N - first)` على الـ bit offset. يعني:

```
bank "GPIOX": first=50, last=70
regs[MESON_REG_DIR] = { reg=0x10, bit=0 }

عشان تضبط direction الـ GPIOX_5 (pin 55):
   --> regmap_write(reg=0x10, bit=(0 + (55-50)) = bit 5)
```

#### 6. `meson_reg_type` — أنواع الـ Register

```c
enum meson_reg_type {
    MESON_REG_PULLEN,  /* هل الـ pull resistor مفعّل؟ */
    MESON_REG_PULL,    /* up أو down؟ */
    MESON_REG_DIR,     /* input أو output؟ */
    MESON_REG_OUT,     /* اكتب قيمة output */
    MESON_REG_IN,      /* اقرأ قيمة input */
    MESON_REG_DS,      /* drive strength */
    MESON_NUM_REG,     /* عدد الأنواع (للـ array sizing) */
};
```

كل pin في البنك ليه 6 registers مختلفة بتتحكم في 6 جوانب مختلفة من سلوكه.

#### 7. `meson_pinconf_drv` — قيم Drive Strength

```c
enum meson_pinconf_drv {
    MESON_PINCONF_DRV_500UA,   /* 0.5 mA */
    MESON_PINCONF_DRV_2500UA,  /* 2.5 mA */
    MESON_PINCONF_DRV_3000UA,  /* 3.0 mA */
    MESON_PINCONF_DRV_4000UA,  /* 4.0 mA */
};
```

الـ **drive strength** ده قدرة الـ pin على source/sink تيار — مهم لـ signals عالية السرعة زي EMMC أو لما فيه load كبير على الـ line.

#### 8. `meson_pinctrl_data` — الـ Static SoC Data

```c
struct meson_pinctrl_data {
    const char                  *name;
    const struct pinctrl_pin_desc *pins;     /* كل pins الـ SoC */
    const struct meson_pmx_group  *groups;   /* كل الـ groups */
    const struct meson_pmx_func   *funcs;    /* كل الـ functions */
    unsigned int                   num_pins;
    unsigned int                   num_groups;
    unsigned int                   num_funcs;
    const struct meson_bank       *banks;    /* كل الـ banks */
    unsigned int                   num_banks;
    const struct pinmux_ops       *pmx_ops;  /* vtable للـ mux operations */
    const void                    *pmx_data; /* بيانات خاصة بالـ mux impl */
    int (*parse_dt)(struct meson_pinctrl *pc); /* hook لـ DT parsing */
};
```

ده الـ **read-only static table** اللي بيتعرّف compile-time لكل SoC. كل Meson SoC (G12A, S905, A1, ...) عنده نسخته الخاصة من الـ struct ده.

#### 9. `meson_pinctrl` — الـ Runtime Instance

```c
struct meson_pinctrl {
    struct device            *dev;       /* الـ platform device */
    struct pinctrl_dev       *pcdev;     /* handle من الـ pinctrl core */
    struct pinctrl_desc       desc;      /* الـ descriptor المسجّل في الـ core */
    struct meson_pinctrl_data *data;     /* pointer على الـ static SoC data */
    struct regmap            *reg_mux;   /* regmap للـ mux registers */
    struct regmap            *reg_pullen;/* regmap لتفعيل الـ pull */
    struct regmap            *reg_pull;  /* regmap لاتجاه الـ pull */
    struct regmap            *reg_gpio;  /* regmap للـ GPIO (dir/in/out) */
    struct regmap            *reg_ds;    /* regmap للـ drive strength */
    struct gpio_chip          chip;      /* الـ GPIO chip المدمج */
    struct fwnode_handle     *fwnode;    /* firmware node (DT or ACPI) */
};
```

ده الـ **runtime state** للدرايفر — بيتعمل عند الـ probe ويعيش طول عمر الـ driver.

---

### العلاقة البصرية بين الـ Structs

```
meson_pinctrl (runtime)
├── data ──────────────────> meson_pinctrl_data (static, per-SoC)
│                             ├── pins[] ──> pinctrl_pin_desc[]
│                             ├── groups[] -> meson_pmx_group[]
│                             │               ├── pins[]
│                             │               └── data (HW mux info)
│                             ├── funcs[] --> meson_pmx_func[]
│                             │               └── groups[] (by name)
│                             ├── banks[] --> meson_bank[]
│                             │               └── regs[MESON_NUM_REG]
│                             │                   └── meson_reg_desc {reg, bit}
│                             └── pmx_ops -> pinmux_ops vtable
│
├── desc ────────────────── pinctrl_desc
│                             ├── pctlops  --> pinctrl_ops vtable
│                             ├── pmxops   --> pinmux_ops vtable
│                             └── confops  --> pinconf_ops vtable
│
├── pcdev ───────────────── pinctrl_dev (owned by pinctrl core)
│
├── reg_mux ─────────────── regmap (MUX registers)
├── reg_pullen ──────────── regmap (PULL ENABLE registers)
├── reg_pull ────────────── regmap (PULL DIR registers)
├── reg_gpio ────────────── regmap (GPIO DIR/IN/OUT registers)
├── reg_ds ──────────────── regmap (DRIVE STRENGTH registers)
│
└── chip ────────────────── gpio_chip (GPIO subsystem interface)
```

---

### الـ Regmap Subsystem — ليه خمس regmaps منفصلين؟

الـ **regmap** هو abstraction layer للـ register access — بيوفّر unified API للقراءة والكتابة سواء كانت الـ registers في MMIO أو I2C أو SPI. لازم تفهم ده قبل تكمل.

في Meson، الـ pin control registers مش في نفس الـ address space — الـ SoC بيقسّمهم على blocks مختلفة حسب وظيفتهم:

| regmap | ما بيتحكم فيه | سبب الفصل |
|--------|--------------|-----------|
| `reg_mux` | أيه function كل pin بيشتغل فيها | بيتحكم فيه الـ pinmux layer |
| `reg_pullen` | هل الـ pull resistor شغّال؟ | 1-bit per pin |
| `reg_pull` | up أو down؟ | 1-bit per pin |
| `reg_gpio` | direction + value | شايل DIR, OUT, IN registers |
| `reg_ds` | drive strength | 2-bit per pin (4 levels) |

الفصل ده بيعكس الـ HW design الحقيقي للـ Meson SoCs — كل مجموعة registers في offset منفصل في الـ address space.

---

### الـ BANK_DS Macro — كيف بيُعرَّف بنك كامل؟

```c
#define BANK_DS(n, f, l, fi, li, per, peb, pr, pb, dr, db, or, ob, ir, ib, dsr, dsb) \
{                                                    \
    .name      = n,     /* اسم البنك */              \
    .first     = f,     /* أول pin */                \
    .last      = l,     /* آخر pin */                \
    .irq_first = fi,    /* أول hwirq */              \
    .irq_last  = li,    /* آخر hwirq */              \
    .regs = {                                        \
        [MESON_REG_PULLEN] = { per, peb }, /* pull-enable reg, bit */ \
        [MESON_REG_PULL]   = { pr,  pb  }, /* pull dir reg, bit */    \
        [MESON_REG_DIR]    = { dr,  db  }, /* direction reg, bit */   \
        [MESON_REG_OUT]    = { or,  ob  }, /* output reg, bit */      \
        [MESON_REG_IN]     = { ir,  ib  }, /* input reg, bit */       \
        [MESON_REG_DS]     = { dsr, dsb }, /* drive-strength reg,bit */\
    },                                               \
}
```

مثال حقيقي من Meson G12A:
```c
BANK_DS("GPIOX", PIN_GPIOX_0, PIN_GPIOX_19,
        10, 29,        /* irq first=10, last=29 */
        0x13, 0,       /* PULLEN: reg=0x13, bit=0 */
        0x14, 0,       /* PULL:   reg=0x14, bit=0 */
        0x18, 0,       /* DIR:    reg=0x18, bit=0 */
        0x19, 0,       /* OUT:    reg=0x19, bit=0 */
        0x1a, 0,       /* IN:     reg=0x1a, bit=0 */
        0x27, 0);      /* DS:     reg=0x27, bit=0 */
```

الـ `bit=0` يعني الـ pin الأول (GPIOX_0) بيتحكم فيه بـ bit 0 — وكل pin بعده بيزيد واحد على الـ bit offset.

---

### الـ GPIO Chip — الدمج مع الـ GPIO Subsystem

الـ **GPIO subsystem** في الكيرنل هو subsystem منفصل (مش الـ pinctrl) — بيدي الـ userspace والـ drivers القدرة يقرأوا/يكتبوا على GPIO lines. لازم تعرف إن الاتنين مستقلين وبيتكاملوا مع بعض.

`meson_pinctrl` بيحمل `struct gpio_chip` مدمج — يعني نفس الدرايفر بيسجّل نفسه في الـ pinctrl core وفي الـ GPIO subsystem في نفس الوقت:

```
meson_pinctrl_probe()
    |
    +-- pinctrl_register_and_init()  --> registers with pinctrl core
    |
    +-- devm_gpiochip_add_data()     --> registers with GPIO subsystem
```

الـ `gpio_chip` بيحدد callbacks زي `gpio_get`, `gpio_set`, `gpio_direction_input`, `gpio_direction_output` — وكلهم في النهاية بيعملوا regmap read/write على `reg_gpio`.

---

### الـ parse_dt Hook — تكيّف مع الـ Device Tree

```c
int (*parse_dt)(struct meson_pinctrl *pc);
```

ده function pointer في الـ `meson_pinctrl_data` — بيُستدعى من `meson_pinctrl_probe` عشان يعمل DT parsing إضافي specific لكل SoC. مثلاً:

- `meson8_aobus_parse_dt_extra()` — للـ Always-On bus في الجيل القديم
- `meson_a1_parse_dt_extra()` — للـ A1 SoC اللي عنده layout مختلف

ده نمط **template method** — الـ framework بيحدد الـ skeleton والـ SoC-specific driver بيملأ التفاصيل.

---

### الـ Pinctrl Core يملك إيه؟ والدرايفر يتكلف بإيه؟

| الـ pinctrl core يملك | الـ meson driver يتكلف بـ |
|----------------------|--------------------------|
| الـ pin number space (global) | تعريف الـ pins بالاسم والرقم في `meson_pinctrl_data` |
| الـ function/group lookup | تعريف functions وgroups لكل SoC |
| الـ consumer API (`pinctrl_get`, `pinctrl_select_state`) | تنفيذ `pmx_ops` للكتابة الفعلية على الـ MUX registers |
| الـ conflict detection | معرفة الـ register address لكل pin في كل bank |
| الـ DT state machine (`pinctrl-0`, `pinctrl-1`) | الـ `parse_dt` hook لتفسير DT properties الخاصة |
| الـ debugfs integration | الـ `pin_dbg_show` callback |
| ربط الـ GPIO range بالـ pinctrl | إنشاء الـ `gpio_chip` وتسجيله |

---

### الـ FUNCTION Macro — بساطة في التعريف

```c
#define FUNCTION(fn)                                    \
{                                                       \
    .name       = #fn,          /* "uart_a" */          \
    .groups     = fn##_groups,  /* uart_a_groups[] */   \
    .num_groups = ARRAY_SIZE(fn##_groups),              \
}
```

بيفترض إن في كل مكان بتعمل `FUNCTION(uart_a)`، لازم يكون موجود بالفعل `static const char * const uart_a_groups[]` معرّف قبله.

الـ pattern الكامل في أي SoC-specific file:

```c
/* 1. اعرّف الـ pins */
static const struct pinctrl_pin_desc meson_g12a_pins[] = {
    MESON_PIN(GPIOX_0),
    MESON_PIN(GPIOX_1),
    /* ... */
};

/* 2. اعرّف الـ groups */
static const unsigned int uart_a_tx_pins[] = { PIN_UART_A_TX };
static const unsigned int uart_a_rx_pins[] = { PIN_UART_A_RX };

static const struct meson_pmx_group meson_g12a_groups[] = {
    { .name="uart_a_tx", .pins=uart_a_tx_pins, .num_pins=1, .data=&tx_data },
    { .name="uart_a_rx", .pins=uart_a_rx_pins, .num_pins=1, .data=&rx_data },
};

/* 3. اعرّف الـ functions */
static const char * const uart_a_groups[] = { "uart_a_tx", "uart_a_rx" };

static const struct meson_pmx_func meson_g12a_funcs[] = {
    FUNCTION(uart_a),
};

/* 4. اعرّف الـ banks */
static const struct meson_bank meson_g12a_banks[] = {
    BANK_DS("GPIOX", ...),
};

/* 5. اجمع كل ده في meson_pinctrl_data */
static struct meson_pinctrl_data meson_g12a_periphs_data = {
    .name      = "periphs-banks",
    .pins      = meson_g12a_pins,
    .groups    = meson_g12a_groups,
    .funcs     = meson_g12a_funcs,
    .banks     = meson_g12a_banks,
    .num_pins  = ARRAY_SIZE(meson_g12a_pins),
    /* ... */
    .pmx_ops   = &meson_g12a_pmx_ops,
};
```

---

### خلاصة — الفكرة في جملة واحدة

الـ pinctrl framework بيفصل بين **وصف الـ hardware** (مين الـ pins وإيه وظايفهم — static data في `meson_pinctrl_data`) و**التحكم فيه runtime** (مين يستخدم مين — عبر الـ core + vtables) — وده بالضبط اللي بيخلي نفس الـ framework يشتغل على مئات الـ SoCs المختلفة.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. Cheatsheet — الـ Enums والـ Macros المهمة

#### `enum meson_reg_type` — أنواع الـ Registers

| القيمة | الفهرس | وظيفتها |
|---|---|---|
| `MESON_REG_PULLEN` | 0 | تفعيل/تعطيل الـ pull resistor |
| `MESON_REG_PULL` | 1 | اتجاه الـ pull (up/down) |
| `MESON_REG_DIR` | 2 | اتجاه الـ pin (input/output) |
| `MESON_REG_OUT` | 3 | كتابة قيمة output |
| `MESON_REG_IN` | 4 | قراءة قيمة input |
| `MESON_REG_DS` | 5 | drive strength |
| `MESON_NUM_REG` | 6 | عدد الـ reg types (sentinel) |

#### `enum meson_pinconf_drv` — قيم الـ Drive Strength

| القيمة | التيار |
|---|---|
| `MESON_PINCONF_DRV_500UA` | 500 µA |
| `MESON_PINCONF_DRV_2500UA` | 2500 µA |
| `MESON_PINCONF_DRV_3000UA` | 3000 µA |
| `MESON_PINCONF_DRV_4000UA` | 4000 µA |

#### Macros المهمة

| الـ Macro | الغرض |
|---|---|
| `MESON_PIN(x)` | تعريف pin descriptor باسمه — يوسع لـ `PINCTRL_PIN(x, #x)` |
| `FUNCTION(fn)` | تعريف `meson_pmx_func` من اسم function واحد |
| `BANK(n,f,l,fi,li,...)` | تعريف bank بدون drive-strength register |
| `BANK_DS(n,f,l,fi,li,...,dsr,dsb)` | تعريف bank مع drive-strength register |

---

### 1. الـ Structs المهمة

#### `struct meson_reg_desc`

**الغرض:** الوحدة الأصغر في النظام — تصف موقع bit واحد في regmap يتحكم في خاصية واحدة لـ pin.

```c
struct meson_reg_desc {
    unsigned int reg;  /* offset في الـ regmap */
    unsigned int bit;  /* رقم الـ bit داخل الـ register */
};
```

- كل pin خاصية (pull، direction، drive) لها `meson_reg_desc` واحد.
- الـ `bit` هو base bit للـ pin الأول في الـ bank — الـ pins التانية تحسب بالـ offset من أول pin.

---

#### `struct meson_bank`

**الغرض:** مجموعة pins متجاورة تتشارك نفس الـ registers للتحكم فيها. مثلاً bank "GPIOX" يضم GPIOX_0 لـ GPIOX_19.

```c
struct meson_bank {
    const char *name;           /* مثلاً "GPIOX" */
    unsigned int first;         /* رقم أول pin في الـ bank */
    unsigned int last;          /* رقم آخر pin */
    int irq_first;              /* أول hwirq number لهذا الـ bank */
    int irq_last;               /* آخر hwirq number */
    struct meson_reg_desc regs[MESON_NUM_REG]; /* مصفوفة 6 عناصر */
};
```

- الـ `regs[]` مفهرسة بـ `meson_reg_type` — يعني `regs[MESON_REG_PULL]` يعطي offset+bit للـ pull register.
- لو الـ `irq_first == irq_last == -1` ← الـ bank ده مش له interrupts.
- لو الـ SoC مش عنده DS، الـ `BANK()` macro يحط `regs[MESON_REG_DS] = {0, 0}`.

---

#### `struct meson_pmx_group`

**الغرض:** مجموعة pins تعمل مع بعض لتأدية function معينة. مثلاً "i2c0_sda" group فيها pin واحد، لكن "uart0" group ممكن تضم TX+RX+CTS+RTS.

```c
struct meson_pmx_group {
    const char *name;           /* مثلاً "uart_a_tx" */
    const unsigned int *pins;   /* مصفوفة pin numbers */
    unsigned int num_pins;      /* عدد الـ pins */
    const void *data;           /* driver-specific data — بيشير لـ meson_pmx_bank أو meson_axg_pmx_data */
};
```

- الـ `data` pointer generic — الـ driver الـ specific بيعرف يفسره حسب الـ SoC.
- كل group بيتسجل في الـ pinctrl core وله selector رقمي.

---

#### `struct meson_pmx_func`

**الغرض:** وظيفة كاملة (مثلاً "uart_a") تتكون من مجموعة groups. الـ consumer بيختار الـ function اسمياً.

```c
struct meson_pmx_func {
    const char *name;               /* مثلاً "uart_a" */
    const char * const *groups;     /* مصفوفة أسماء الـ groups */
    unsigned int num_groups;        /* عددها */
};
```

- الـ macro `FUNCTION(uart_a)` يتوقع وجود مصفوفة اسمها `uart_a_groups[]` في ملف الـ SoC.

---

#### `struct meson_pinctrl_data`

**الغرض:** الـ static data الخاصة بكل SoC — كل SoC له instance منها مخزّن في flash (const). ده هو "شخصية" الـ SoC.

```c
struct meson_pinctrl_data {
    const char *name;                          /* اسم الـ controller */
    const struct pinctrl_pin_desc *pins;       /* مصفوفة كل الـ pins */
    const struct meson_pmx_group *groups;      /* مصفوفة الـ groups */
    const struct meson_pmx_func *funcs;        /* مصفوفة الـ functions */
    unsigned int num_pins;
    unsigned int num_groups;
    unsigned int num_funcs;
    const struct meson_bank *banks;            /* مصفوفة الـ banks */
    unsigned int num_banks;
    const struct pinmux_ops *pmx_ops;          /* vtable الـ pinmux ops */
    const void *pmx_data;                      /* extra data للـ pmx ops */
    int (*parse_dt)(struct meson_pinctrl *pc); /* callback لـ DT parsing */
};
```

- بيتخزن كـ `platform_data` في جدول الـ `of_device_id` لكل SoC.
- الـ `pmx_ops` بتختلف: SoCs قديمة (Meson8) تستخدم ops مختلفة عن الجديدة (AXG/G12A).
- الـ `parse_dt` callback بيسمح لكل SoC يضيف regmaps إضافية (مثلاً AO domain).

---

#### `struct meson_pinctrl`

**الغرض:** الـ runtime state للـ driver — بيتخلق في `meson_pinctrl_probe()` وبيعيش طول فترة عمل الـ driver.

```c
struct meson_pinctrl {
    struct device *dev;               /* الـ platform device */
    struct pinctrl_dev *pcdev;        /* handle من الـ pinctrl core */
    struct pinctrl_desc desc;         /* descriptor مسجّل في الـ core */
    struct meson_pinctrl_data *data;  /* pointer للـ SoC-specific data */
    struct regmap *reg_mux;           /* regmap للـ mux registers */
    struct regmap *reg_pullen;        /* regmap لتفعيل الـ pull */
    struct regmap *reg_pull;          /* regmap لاتجاه الـ pull */
    struct regmap *reg_gpio;          /* regmap لـ GPIO (dir/in/out) */
    struct regmap *reg_ds;            /* regmap لـ drive strength */
    struct gpio_chip chip;            /* الـ GPIO chip المدمج */
    struct fwnode_handle *fwnode;     /* firmware node (DT/ACPI) */
};
```

- الـ regmaps ممكن تكون منفصلة أو نفس الـ regmap (حسب الـ SoC).
- `reg_ds` بيكون NULL على SoCs قديمة مش عندها DS control.
- الـ `gpio_chip` embedded مش pointer — يعني مفيش dynamic allocation منفصلة.

---

### 2. مخططات العلاقات بين الـ Structs

```
┌─────────────────────────────────────────────────────────────────┐
│                    meson_pinctrl  (runtime)                     │
│                                                                 │
│  dev ──────────────────────────────► struct device              │
│  pcdev ─────────────────────────────► struct pinctrl_dev        │
│  desc ──────────────────────────────► struct pinctrl_desc       │
│         (embedded, contains ptr to pins[], pmxops, confops)     │
│  data ──────────────────────────────► struct meson_pinctrl_data │
│  reg_mux ───────────────────────────► struct regmap             │
│  reg_pullen ────────────────────────► struct regmap             │
│  reg_pull ──────────────────────────► struct regmap             │
│  reg_gpio ──────────────────────────► struct regmap             │
│  reg_ds ────────────────────────────► struct regmap (nullable)  │
│  chip ──────────────────────────────► struct gpio_chip (embed)  │
│  fwnode ────────────────────────────► struct fwnode_handle      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ data pointer
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                meson_pinctrl_data  (const, SoC-specific)        │
│                                                                 │
│  pins ──────────────────────────────► pinctrl_pin_desc[]        │
│                                        [pin_number, "name"]     │
│  groups ────────────────────────────► meson_pmx_group[]         │
│                                         │                       │
│                                         │ pins[] ──► uint[]     │
│                                         │ data  ──► pmx_data    │
│  funcs ─────────────────────────────► meson_pmx_func[]          │
│                                         │                       │
│                                         │ groups[] ──► char*[]  │
│  banks ─────────────────────────────► meson_bank[]              │
│                                         │                       │
│                                         │ regs[6] ─► meson_reg_desc
│  pmx_ops ───────────────────────────► pinmux_ops (vtable)       │
│  parse_dt() ────────────────────────► callback fn               │
└─────────────────────────────────────────────────────────────────┘
```

#### علاقة الـ Bank بالـ Reg Descriptors

```
meson_bank ("GPIOX")
├── first = 50, last = 69
├── irq_first = 0, irq_last = 19
└── regs[6]:
    ├── [MESON_REG_PULLEN] = { reg=0x0C, bit=0 }  ← GPIOX_0 pull-en bit
    ├── [MESON_REG_PULL]   = { reg=0x3A, bit=0 }
    ├── [MESON_REG_DIR]    = { reg=0x10, bit=0 }
    ├── [MESON_REG_OUT]    = { reg=0x11, bit=0 }
    ├── [MESON_REG_IN]     = { reg=0x12, bit=0 }
    └── [MESON_REG_DS]     = { reg=0x2C, bit=0 }

للـ pin N في الـ bank:
  bit_offset = regs[type].bit + (N - bank->first)
  → regmap_update_bits(reg_gpio, regs[MESON_REG_DIR].reg,
                       BIT(bit_offset), ...)
```

#### علاقة الـ GPIO chip بالـ pinctrl

```
meson_pinctrl
   └── chip (gpio_chip, embedded)
         ├── .request()     → meson_gpio_request()
         ├── .get()         → reads reg_gpio MESON_REG_IN
         ├── .set()         → writes reg_gpio MESON_REG_OUT
         ├── .direction_input()  → clears MESON_REG_DIR bit
         ├── .direction_output() → sets MESON_REG_DIR bit
         └── .to_irq()      → maps pin to hwirq via bank->irq_first

  للوصول لـ meson_pinctrl من gpio_chip:
  gpiochip_get_data(&chip) → meson_pinctrl *pc
```

---

### 3. مخطط دورة الحياة

```
platform_device probe
       │
       ▼
meson_pinctrl_probe(pdev)
       │
       ├── devm_kzalloc → alloc meson_pinctrl (pc)
       │
       ├── of_device_get_match_data → pc->data = SoC-specific data
       │
       ├── pc->data->parse_dt(pc)
       │     ├── meson8_aobus_parse_dt_extra()  [old SoCs: AO domain]
       │     └── meson_a1_parse_dt_extra()      [A1: unified regmap]
       │         → يحدد regmaps: reg_mux, reg_pullen, reg_pull,
       │                          reg_gpio, reg_ds
       │
       ├── بناء pinctrl_desc:
       │     ├── desc.pins  = data->pins
       │     ├── desc.npins = data->num_pins
       │     ├── desc.pctlops  = &meson_pctrl_ops
       │     ├── desc.pmxops   = data->pmx_ops
       │     └── desc.confops  = &meson_pinconf_ops
       │
       ├── devm_pinctrl_register_and_init(dev, &desc, pc, &pcdev)
       │     → pinctrl core يسجّل الـ controller
       │
       ├── meson_gpiolib_register(pc)
       │     ├── بناء gpio_chip (label, ngpio, callbacks)
       │     └── devm_gpiochip_add_data(dev, &pc->chip, pc)
       │           → gpiolib تسجّل الـ GPIO chip
       │           → irqdomain يتخلق لو irq_first != -1
       │
       └── pinctrl_enable(pcdev)
             → الـ core جاهز يقبل pinctrl_get/select requests
```

#### Teardown

```
platform_device remove  (أو driver error path)
       │
       ├── devm_* cleanup:
       │     ├── gpiochip_remove(chip)
       │     │     └── irqdomain_remove لو كان موجود
       │     └── pinctrl_unregister(pcdev)
       │           └── الـ core يحرر كل الـ maps والـ states
       │
       └── devm_kfree(pc)
```

---

### 4. مخططات تدفق الـ Calls

#### تطبيق Pinmux Function

```
consumer driver calls:
  pinctrl_select_state(pstate, "default")
    │
    └── pinctrl core iterates maps
          │
          └── pinmux_enable_setting()
                │
                └── pmx_ops->set_mux(pcdev, func_selector, group_selector)
                      │
                      └── مثلاً meson_pmx_set() [في .c file]
                            │
                            ├── الحصول على meson_pmx_group من group_selector
                            │
                            ├── قراءة group->data → meson_pmx_bank
                            │     (reg offset + bit للـ mux)
                            │
                            └── regmap_update_bits(pc->reg_mux,
                                                   bank->reg,
                                                   BIT(bank->bit),
                                                   val)
                                  │
                                  └── hardware register يتغير
```

#### قراءة GPIO

```
user calls: gpiod_get_value(desc)
    │
    └── gpiochip->get(gc, offset)
          │
          └── meson_gpio_get(gc, offset)
                │
                ├── gpiochip_get_data(gc) → meson_pinctrl *pc
                │
                ├── meson_calc_reg_and_bit(pc, offset,
                │                          MESON_REG_IN,
                │                          &reg, &bit)
                │     ├── البحث عن bank التي تحوي offset
                │     └── bit = bank->regs[MESON_REG_IN].bit
                │                + (offset - bank->first)
                │
                └── regmap_read(pc->reg_gpio, reg, &val)
                      └── return (val >> bit) & 1
```

#### إعداد Drive Strength

```
consumer: PIN_CONFIG_DRIVE_STRENGTH_UA via pinctrl DT
    │
    └── pinconf_apply_setting()
          │
          └── confops->pin_config_set(pcdev, pin, configs, nconfigs)
                │
                └── meson_pinconf_set()
                      │
                      ├── لكل config:
                      │   PIN_CONFIG_DRIVE_STRENGTH_UA:
                      │     ├── meson_calc_reg_and_bit(..., MESON_REG_DS)
                      │     ├── تحويل µA → MESON_PINCONF_DRV_* enum
                      │     └── regmap_update_bits(pc->reg_ds, reg,
                      │                            0x3 << bit, val << bit)
                      │         (DS يحتاج 2 bits لكل pin)
                      │
                      └── PIN_CONFIG_BIAS_PULL_UP/DOWN:
                            ├── regmap_update_bits(reg_pullen, ...) ← enable
                            └── regmap_update_bits(reg_pull, ...)   ← direction
```

#### GPIO Interrupt Flow

```
external interrupt على GPIO pin
    │
    └── GIC يستقبل الـ parent IRQ
          │
          └── gpio_irq_chip handler
                │
                ├── يقرأ interrupt status register
                │
                └── generic_handle_irq(virq)
                      │
                      └── consumer ISR يتنفّذ
```

---

### 5. استراتيجية الـ Locking

#### ما الذي يحمي ماذا؟

| المورد | الـ Lock | السبب |
|---|---|---|
| `regmap` operations | داخلي في `regmap` (spinlock أو mutex حسب الـ config) | الـ regmap framework يتعامل معاه |
| `gpio_chip` state | `spinlock` في `gpio_device` (داخل gpiolib) | fast IRQ context |
| `pinctrl_dev` maps/states | `mutex` في `pinctrl_dev` (داخل pinctrl core) | sleepable context |
| `meson_pinctrl` struct نفسه | لا يوجد lock خاص | كل الـ fields تتكتب مرة واحدة في probe |

#### ملاحظات مهمة:

**الـ `meson_pinctrl` struct** نفسه ما فيهوش explicit lock لأن:
- الـ fields الـ static (data, desc) تتكتب مرة واحدة في probe ومن بعدها read-only.
- الـ regmap pointers ثابتة بعد probe.
- الـ GPIO operations المتزامنة يتحكم فيها gpiolib بـ spinlock داخلي.

**ترتيب الـ Locks (لتجنب deadlock):**

```
1. gpio_device spinlock   (الأسرع، IRQ-safe)
      ↓
2. regmap lock            (تابع — بيتاخد جوا الـ gpio operations)
      ↓
3. pinctrl_dev mutex      (الأبطأ، sleeping — لـ mux/conf changes فقط)
```

**قاعدة:** كل كود يمشي في IRQ context (مثلاً `gpio_chip->get()`) لازم يستخدم الـ regmap المعمول بـ `regmap_config.fast_io = true` أو يتأكد إن الـ regmap لا ينام.

**الـ `reg_ds`:** لو `NULL` (SoC قديم)، الـ driver يرجع `-ENOTSUPP` بدل ما يعمل null dereference.
## Phase 4: شرح الـ Functions

---

### ملخص الـ Functions والـ APIs (Cheatsheet)

#### Registration & Probe

| Function | النوع | الغرض |
|---|---|---|
| `meson_pinctrl_probe` | exported | Entry point — بيعمل probe للـ driver كله |
| `meson_gpiolib_register` | static | بيسجل الـ `gpio_chip` في gpiolib |
| `meson_map_resource` | static | بيعمل ioremap وregmap لـ register range واحد |
| `meson_pinctrl_parse_dt` | static | بيقرأ الـ DT ويبني الـ regmaps |
| `meson8_aobus_parse_dt_extra` | exported | تعديل خاص بـ Meson8 AO bus |
| `meson_a1_parse_dt_extra` | exported | تعديل خاص بـ A1 SoC |

#### Pinctrl Group/Function Ops

| Function | النوع | الغرض |
|---|---|---|
| `meson_get_groups_count` | static | عدد الـ pin groups |
| `meson_get_group_name` | static | اسم الـ group بالـ index |
| `meson_get_group_pins` | static | pins الـ group |
| `meson_pin_dbg_show` | static | debug output في debugfs |
| `meson_pmx_get_funcs_count` | exported | عدد الـ mux functions |
| `meson_pmx_get_func_name` | exported | اسم الـ function |
| `meson_pmx_get_groups` | exported | groups الخاصة بـ function |

#### Pin Config (Set)

| Function | النوع | الغرض |
|---|---|---|
| `meson_pinconf_set` | static | dispatcher لـ pin config |
| `meson_pinconf_group_set` | static | نفس الـ set لكل pins في group |
| `meson_pinconf_disable_bias` | static | يعطّل الـ pull resistor |
| `meson_pinconf_enable_bias` | static | يفعّل pull-up أو pull-down |
| `meson_pinconf_set_output` | static | يضبط اتجاه الـ pin (in/out) |
| `meson_pinconf_set_drive` | static | يضبط قيمة output الـ pin |
| `meson_pinconf_set_output_drive` | static | يضبط الاتجاه والقيمة معاً |
| `meson_pinconf_set_drive_strength` | static | يضبط drive strength بالـ µA |

#### Pin Config (Get)

| Function | النوع | الغرض |
|---|---|---|
| `meson_pinconf_get` | static | يقرأ config للـ pin |
| `meson_pinconf_group_get` | static | دايما يرجع `-ENOTSUPP` |
| `meson_pinconf_get_pull` | static | يقرأ حالة الـ pull |
| `meson_pinconf_get_output` | static | يقرأ اتجاه الـ pin |
| `meson_pinconf_get_drive` | static | يقرأ قيمة output |
| `meson_pinconf_get_drive_strength` | static | يقرأ drive strength |

#### GPIO Chip Callbacks

| Function | النوع | الغرض |
|---|---|---|
| `meson_gpio_get_direction` | static | يقرأ اتجاه الـ GPIO |
| `meson_gpio_direction_input` | static | يضبط الـ GPIO كـ input |
| `meson_gpio_direction_output` | static | يضبط الـ GPIO كـ output بقيمة |
| `meson_gpio_get` | static | يقرأ input value |
| `meson_gpio_set` | static | يضبط output value |

#### Low-Level Helpers

| Function | النوع | الغرض |
|---|---|---|
| `meson_get_bank` | static | بيدور على الـ bank اللي فيها الـ pin |
| `meson_calc_reg_and_bit` | static | بيحسب offset الـ register والـ bit |
| `meson_pinconf_set_gpio_bit` | static | بيكتب bit واحد في reg_gpio |
| `meson_pinconf_get_gpio_bit` | static | بيقرأ bit واحد من reg_gpio |

---

### Group 1: Low-Level Register Helpers

الغرض من الـ group دي هو تجريد التعامل مع الـ register space. كل عملية على pin بتبدأ بإيجاد الـ bank اللي بيتبعها، وبعدين حساب الـ register offset والـ bit index الصح.

---

#### `meson_get_bank`

```c
static int meson_get_bank(struct meson_pinctrl *pc,
                          unsigned int pin,
                          const struct meson_bank **bank)
```

بتعمل linear scan على كل الـ banks المعرّفة في الـ SoC data وبتدوّر على البنك اللي يكون `first <= pin <= last`. لو لقته بترجع 0 وبتحط pointer للـ bank في `*bank`.

**Parameters:**
- `pc` — instance الـ pinctrl driver
- `pin` — رقم الـ pin الـ global
- `bank` — output pointer للـ bank المطلوبة

**Return:** 0 لو النجاح، `-EINVAL` لو الـ pin مش موجود في أي bank.

**Key details:** مفيش locking هنا لأن الـ banks array بتكون read-only static data بعد الـ probe. الـ caller مسؤول إنه يتعامل مع الـ NULL حالة الـ error.

---

#### `meson_calc_reg_and_bit`

```c
static void meson_calc_reg_and_bit(const struct meson_bank *bank,
                                   unsigned int pin,
                                   enum meson_reg_type reg_type,
                                   unsigned int *reg,
                                   unsigned int *bit)
```

بتحسب الـ byte offset للـ register والـ bit index داخل الـ register لـ pin معين. الـ Meson hardware بيحط كل pins الـ bank في registers متتالية، كل pin بياخد عدد من الـ bits حسب `meson_bit_strides[reg_type]` (1 bit لـ GPIO/pull، 2 bits للـ drive strength).

**الحساب:**
```
bit_offset = (desc->bit + pin - bank->first) * stride
reg = (desc->reg + (bit_offset / 32)) * 4   /* byte addressing */
bit = bit_offset & 0x1f                      /* bit within 32-bit word */
```

**Parameters:**
- `bank` — البنك اللي فيها الـ pin
- `pin` — رقم الـ pin الـ global
- `reg_type` — نوع الـ register: PULLEN, PULL, DIR, OUT, IN, DS
- `reg` — output: byte offset في الـ regmap
- `bit` — output: bit position جوه الـ 32-bit register

**Key details:** الـ `meson_bit_strides` array هي `{1,1,1,1,1,2,1}` — بس الـ DS register بياخد 2 bits لكل pin (لأن عنده 4 قيم ممكنة)، باقي الـ types كلهم 1 bit.

---

#### `meson_pinconf_set_gpio_bit` / `meson_pinconf_get_gpio_bit`

```c
static int meson_pinconf_set_gpio_bit(struct meson_pinctrl *pc,
                                      unsigned int pin,
                                      unsigned int reg_type,
                                      bool arg)

static int meson_pinconf_get_gpio_bit(struct meson_pinctrl *pc,
                                      unsigned int pin,
                                      unsigned int reg_type)
```

الاتنين بيعملوا wrapper حول `meson_get_bank` + `meson_calc_reg_and_bit` ثم `regmap_update_bits` أو `regmap_read` على `pc->reg_gpio`. الـ Set بيستخدم atomic `regmap_update_bits` عشان ما يأثرش على الـ bits التانية في نفس الـ register.

**Return للـ get:** 0 أو 1 لو النجاح، قيمة سالبة لو error.

---

### Group 2: Pinctrl Group & Function Ops

الـ pinctrl core بيتكلم مع الـ driver من خلال `pinctrl_ops` و`pinmux_ops`. الـ group دي بتنفذ الجزء الخاص بالـ `pinctrl_ops`.

---

#### `meson_get_groups_count`

```c
static int meson_get_groups_count(struct pinctrl_dev *pcdev)
```

بترجع `pc->data->num_groups`. الـ pinctrl core بيستخدمها عشان يعرف يعمل enumerate للـ groups.

**Caller context:** pinctrl core في سياق الـ `pinctrl_ops`.

---

#### `meson_get_group_name`

```c
static const char *meson_get_group_name(struct pinctrl_dev *pcdev,
                                        unsigned selector)
```

بترجع `pc->data->groups[selector].name`. الـ `selector` هو index مباشر في الـ static groups array.

---

#### `meson_get_group_pins`

```c
static int meson_get_group_pins(struct pinctrl_dev *pcdev,
                                unsigned selector,
                                const unsigned **pins,
                                unsigned *num_pins)
```

بتحط pointer للـ pins array والعدد في `*pins` و`*num_pins`. الـ pinctrl core بيستخدمهم لو عايز يعرف الـ pins اللي في كل group.

**Return:** دايماً 0.

---

#### `meson_pin_dbg_show`

```c
static void meson_pin_dbg_show(struct pinctrl_dev *pcdev,
                               struct seq_file *s,
                               unsigned offset)
```

بتطبع اسم الـ device في الـ debugfs file. بسيطة جداً — بس مفيدة لتشخيص الـ mux assignments.

---

### Group 3: Pinmux Function Ops (Exported)

الـ functions دي بتنفذ `pinmux_ops` القياسي. الـ SoC-specific mux ops (زي `meson_pmx_set`) بتيجي من ملفات تانية زي `pinctrl-meson-gxl.c` وغيره، لكن الـ query ops مشتركة هنا.

---

#### `meson_pmx_get_funcs_count`

```c
int meson_pmx_get_funcs_count(struct pinctrl_dev *pcdev)
```

بترجع عدد الـ mux functions المتاحة في الـ SoC. الـ pinctrl core بيستخدمها أول ما بيعمل enumerate للـ functions.

**Export:** `EXPORT_SYMBOL_GPL` — الـ SoC drivers بتستخدمها مباشرة.

---

#### `meson_pmx_get_func_name`

```c
const char *meson_pmx_get_func_name(struct pinctrl_dev *pcdev,
                                    unsigned selector)
```

بترجع اسم الـ function زي `"uart_a"` أو `"i2c_b"`. الـ selector هو index في `pc->data->funcs`.

**Export:** `EXPORT_SYMBOL_GPL`.

---

#### `meson_pmx_get_groups`

```c
int meson_pmx_get_groups(struct pinctrl_dev *pcdev,
                         unsigned selector,
                         const char * const **groups,
                         unsigned * const num_groups)
```

بتحط pointer للـ groups array والعدد بتاعها. كل function ممكن تكون فيها أكتر من group (مثلاً `uart_a` ممكن تكون فيها `uart_a_tx` و`uart_a_rx` كـ groups منفصلة).

**Return:** دايماً 0.

---

### Group 4: Pin Configuration (Set Path)

---

#### `meson_pinconf_set`

```c
static int meson_pinconf_set(struct pinctrl_dev *pcdev,
                             unsigned int pin,
                             unsigned long *configs,
                             unsigned num_configs)
```

الـ main dispatcher للـ pin configuration. بتلف على كل الـ configs المطلوبة وبتستدعي الـ helper المناسب لكل نوع.

**Pseudocode:**
```
for each config in configs[]:
    param = extract param
    arg   = extract arg (للـ params اللي محتاجة قيمة)
    switch(param):
        BIAS_DISABLE       → meson_pinconf_disable_bias()
        BIAS_PULL_UP       → meson_pinconf_enable_bias(pin, true)
        BIAS_PULL_DOWN     → meson_pinconf_enable_bias(pin, false)
        DRIVE_STRENGTH_UA  → meson_pinconf_set_drive_strength(pin, arg)
        OUTPUT_ENABLE      → meson_pinconf_set_output(pin, arg)
        LEVEL              → meson_pinconf_set_output_drive(pin, arg)
        default            → -ENOTSUPP
    if error: return immediately
return 0
```

**Parameters:**
- `pcdev` — الـ pinctrl device instance
- `pin` — رقم الـ pin
- `configs` — array من packed config values (param + arg في u64)
- `num_configs` — عدد الـ configs

**Key details:** الـ function بتوقف عند أول error. الـ generic pinconf framework بيستخدم `pinconf_to_config_param` و`pinconf_to_config_argument` لفك الـ packed value.

---

#### `meson_pinconf_disable_bias`

```c
static int meson_pinconf_disable_bias(struct meson_pinctrl *pc,
                                      unsigned int pin)
```

بتعطّل الـ pull resistor بالكامل. بتعمل clear لـ bit الـ PULLEN فقط — مش محتاجة تمس bit الـ PULL نفسه.

**الخطوات:**
1. `meson_get_bank` للـ pin
2. `meson_calc_reg_and_bit` لـ `MESON_REG_PULLEN`
3. `regmap_update_bits(pc->reg_pullen, reg, BIT(bit), 0)` — clear الـ enable bit

---

#### `meson_pinconf_enable_bias`

```c
static int meson_pinconf_enable_bias(struct meson_pinctrl *pc,
                                     unsigned int pin,
                                     bool pull_up)
```

بتفعّل الـ pull resistor وبتحدد الاتجاه (up أو down). محتاجة خطوتين منفصلتين:

1. تضبط bit الـ PULL (الاتجاه) في `reg_pull`
2. تفعّل enable bit الـ PULLEN في `reg_pullen`

**Key details:** الـ sequence مهمة — لازم تضبط الاتجاه الأول قبل تفعيل الـ enable عشان ما يحصلش glitch بيجي من default direction.

**Parameters:**
- `pull_up` — `true` = pull-up، `false` = pull-down

---

#### `meson_pinconf_set_output` / `meson_pinconf_set_drive` / `meson_pinconf_set_output_drive`

```c
static int meson_pinconf_set_output(struct meson_pinctrl *pc,
                                    unsigned int pin, bool out)

static int meson_pinconf_set_drive(struct meson_pinctrl *pc,
                                   unsigned int pin, bool high)

static int meson_pinconf_set_output_drive(struct meson_pinctrl *pc,
                                          unsigned int pin, bool high)
```

**الـ `set_output`:** بيضبط الـ DIR bit في `reg_gpio`. لاحظ إن `!out` بيتكتب لأن الـ hardware بيعتبر 0 = output و1 = input (active-low direction bit).

**الـ `set_drive`:** بيضبط قيمة الـ OUT register في `reg_gpio` — ده الـ output data register.

**الـ `set_output_drive`:** بيجمع الاتنين في sequence واحدة:
1. `set_output(pin, true)` — اتجاه output
2. `set_drive(pin, high)` — القيمة

**Caller:** `meson_gpio_direction_output` من gpiolib.

---

#### `meson_pinconf_set_drive_strength`

```c
static int meson_pinconf_set_drive_strength(struct meson_pinctrl *pc,
                                            unsigned int pin,
                                            u16 drive_strength_ua)
```

بيضبط الـ drive strength للـ pin بالـ microamps. الـ hardware بيدعم 4 قيم فقط: 500µA، 2500µA، 3000µA، 4000µA — كل قيمة محتاجة 2 bits في الـ DS register.

**Pseudocode:**
```
if !pc->reg_ds: return -ENOTSUPP
get bank, calc DS reg+bit
if   ua <= 500  → ds_val = 0 (500µA)
elif ua <= 2500 → ds_val = 1 (2500µA)
elif ua <= 3000 → ds_val = 2 (3000µA)
elif ua <= 4000 → ds_val = 3 (4000µA)
else            → warn, use 4000µA
regmap_update_bits(reg_ds, reg, 0x3 << bit, ds_val << bit)
```

**Key details:** الـ mask بيكون `0x3 << bit` عشان 2 bits للـ DS. الـ `dev_warn_once` بيُستخدم بدل panic لو الـ DT بعت قيمة أكبر من 4000µA.

---

#### `meson_pinconf_group_set`

```c
static int meson_pinconf_group_set(struct pinctrl_dev *pcdev,
                                   unsigned int num_group,
                                   unsigned long *configs,
                                   unsigned num_configs)
```

بتطبق نفس الـ pin configs على كل الـ pins في group واحدة. بتلف على `group->pins[]` وبتستدعي `meson_pinconf_set` لكل واحد.

**Key details:** الـ errors بيتم تجاهلها هنا (مش بتوقف عند أول error)، والـ function بترجع 0 دايماً — ده بيوضح إن الـ group config بـ best-effort approach.

---

### Group 5: Pin Configuration (Get Path)

---

#### `meson_pinconf_get`

```c
static int meson_pinconf_get(struct pinctrl_dev *pcdev,
                             unsigned int pin,
                             unsigned long *config)
```

بيقرأ الـ config الحالي للـ pin ويبنيه كـ packed value. الـ `*config` بييجي وفيه الـ param المطلوب، والـ function بترجع الـ value.

**الـ params المدعومة:**
- `BIAS_DISABLE/PULL_UP/PULL_DOWN` — بيقارن النتيجة بالـ param المطلوب، لو ما اتطابقوش يرجع `-EINVAL`
- `DRIVE_STRENGTH_UA` — بيقرأ من الـ DS register
- `OUTPUT_ENABLE` — بيتحقق إن الـ pin فعلاً output
- `LEVEL` — بيتحقق إن الـ pin output ويقرأ قيمة الـ OUT register

**Key details:** قيمة `arg = 60000` اللي بتتحط للـ bias params هي conventional value بتستخدمها الـ pinconf framework لتعبّر عن "enabled" — مش مهمة كـ pull resistance حقيقية.

---

#### `meson_pinconf_get_pull`

```c
static int meson_pinconf_get_pull(struct meson_pinctrl *pc,
                                  unsigned int pin)
```

بيقرأ حالة الـ pull للـ pin بخطوتين:
1. يقرأ الـ PULLEN register — لو الـ bit = 0: يرجع `PIN_CONFIG_BIAS_DISABLE`
2. لو الـ bit = 1: يقرأ الـ PULL register — لو = 1 يرجع `PULL_UP`، لو = 0 يرجع `PULL_DOWN`

**Return:** أحد قيم `PIN_CONFIG_BIAS_*` أو قيمة سالبة لو error.

---

#### `meson_pinconf_get_drive_strength`

```c
static int meson_pinconf_get_drive_strength(struct meson_pinctrl *pc,
                                            unsigned int pin,
                                            u16 *drive_strength_ua)
```

بيقرأ الـ 2-bit field من الـ DS register وبيحوّله لـ µA value حقيقية. بيستخدم `(val >> bit) & 0x3` عشان يعزل الـ 2 bits.

**Return:** 0 لو النجاح، `-ENOTSUPP` لو `reg_ds` مش موجود، `-EINVAL` لو القيمة مش recognized.

---

#### `meson_pinconf_group_get`

```c
static int meson_pinconf_group_get(struct pinctrl_dev *pcdev,
                                   unsigned int group,
                                   unsigned long *config)
```

دايماً بترجع `-ENOTSUPP`. الـ pinctrl framework مش بيستخدمها فعلياً بشكل إلزامي، لكن لازم تكون موجودة في الـ `pinconf_ops`.

---

### Group 6: GPIO Chip Callbacks

الـ group دي بتشكّل الـ GPIO interface اللي بيشوفها الـ kernel users عن طريق `/sys/class/gpio` أو `gpiod_*` API.

---

#### `meson_gpio_get_direction`

```c
static int meson_gpio_get_direction(struct gpio_chip *chip, unsigned gpio)
```

بيقرأ الـ DIR register ويرجع `GPIO_LINE_DIRECTION_OUT` أو `GPIO_LINE_DIRECTION_IN`. بيستخدم `meson_pinconf_get_output` اللي بيعكس الـ bit لأن الـ hardware بيستخدم active-low direction.

**Return:** `GPIO_LINE_DIRECTION_IN/OUT` أو قيمة سالبة.

---

#### `meson_gpio_direction_input`

```c
static int meson_gpio_direction_input(struct gpio_chip *chip, unsigned gpio)
```

wrapper مباشر حول `meson_pinconf_set_output(pc, gpio, false)` — بيعمل clear للـ DIR bit (اللي هو active-low بيكون 1 عشان input).

---

#### `meson_gpio_direction_output`

```c
static int meson_gpio_direction_output(struct gpio_chip *chip,
                                       unsigned gpio, int value)
```

wrapper حول `meson_pinconf_set_output_drive` — بيضبط الاتجاه والـ initial value في نفس الوقت.

---

#### `meson_gpio_get`

```c
static int meson_gpio_get(struct gpio_chip *chip, unsigned gpio)
```

بيقرأ الـ IN register (مش الـ OUT) — ده مهم لأن الـ IN register بيعكس الـ actual pin state على الـ pad، سواء كان input أو output.

**الخطوات:**
1. `meson_get_bank`
2. `meson_calc_reg_and_bit` لـ `MESON_REG_IN`
3. `regmap_read` من `pc->reg_gpio`
4. يرجع `!!(val & BIT(bit))`

**Key details:** الـ `!!` بتحوّل أي قيمة non-zero لـ 1 بشكل explicit.

---

#### `meson_gpio_set`

```c
static int meson_gpio_set(struct gpio_chip *chip, unsigned int gpio, int value)
```

wrapper حول `meson_pinconf_set_drive` — بيكتب في الـ OUT register.

**Key details:** بيستخدم النوع الجديد `int` للـ value بدل `bool` لأن الـ GPIO API بيمرر 0 أو non-zero.

---

### Group 7: Registration & Probe

---

#### `meson_gpiolib_register`

```c
static int meson_gpiolib_register(struct meson_pinctrl *pc)
```

بيبني وبيسجل `gpio_chip` في الـ gpiolib. بيملأ كل الـ fields المطلوبة ثم بيستدعي `gpiochip_add_data`.

**الـ chip fields المهمة:**
| Field | القيمة |
|---|---|
| `base` | `-1` (dynamic allocation) |
| `ngpio` | `pc->data->num_pins` |
| `can_sleep` | `true` (regmap يستخدم I2C/sleep-safe access) |
| `fwnode` | من الـ DT GPIO node |
| `request/free` | `gpiochip_generic_request/free` (pinctrl integration) |

**Key details:** `can_sleep = true` ضروري لأن الـ regmap MMIO على بعض platforms ممكن يكون non-atomic. استخدام `gpiochip_generic_request` بيضمن إن الـ pinctrl core بيتعمله notify لما GPIO بيتطلب.

---

#### `meson_map_resource`

```c
static struct regmap *meson_map_resource(struct meson_pinctrl *pc,
                                         struct device_node *node,
                                         char *name)
```

بيعمل 3 حاجات في sequence:
1. `of_property_match_string(node, "reg-names", name)` — بيلاقي index الـ `reg` entry
2. `devm_ioremap_resource` — بيعمل map للـ MMIO range
3. `devm_regmap_init_mmio` — بيبني regmap فوقه

الـ `meson_regmap_config` بيتعدّل قبل الاستدعاء عشان يضبط `max_register` بناءً على حجم الـ resource، وده بيمنع out-of-bounds writes.

**Return:** `struct regmap*` أو `NULL`/`ERR_PTR` لو فشل.

**Key details:** الـ `meson_regmap_config` هو `static` — ده يعني إن الـ function دي مش thread-safe لو اتنادت بالتوازي. لكن في الـ probe context ده مش مشكلة.

---

#### `meson_pinctrl_parse_dt`

```c
static int meson_pinctrl_parse_dt(struct meson_pinctrl *pc)
```

بيقرأ كل الـ register regions من الـ DT وبيبني الـ regmaps المقابلة.

**Pseudocode:**
```
chips = gpiochip_node_count(dev)
if chips != 1: return error

pc->fwnode = gpiochip_node_get_first(dev)
gpio_np = to_of_node(fwnode)

pc->reg_mux  = meson_map_resource("mux")   /* required */
pc->reg_gpio = meson_map_resource("gpio")  /* required */
pc->reg_pull = meson_map_resource("pull")  /* optional */
pc->reg_pullen = meson_map_resource("pull-enable") /* optional */
pc->reg_ds = meson_map_resource("ds")      /* optional */

if pc->data->parse_dt:
    return pc->data->parse_dt(pc)  /* SoC-specific fixups */
return 0
```

**Key details:**
- `reg_mux` و`reg_gpio` إلزاميين — الـ probe بيفشل لو مش موجودين
- الـ optional resources بتتعامل معها بـ `NULL` لو مش موجودة
- الـ `parse_dt` callback بيسمح للـ SoC-specific code يعمل fixup على الـ regmap assignments

---

#### `meson8_aobus_parse_dt_extra`

```c
int meson8_aobus_parse_dt_extra(struct meson_pinctrl *pc)
```

خاصة بـ Meson8 AO domain — في الـ SoC ده، نفس الـ register بيتحكم في كل من PULL direction وPULL enable. الـ function بتعمل `pc->reg_pullen = pc->reg_pull` عشان تعكس ده.

**Key details:** لو `reg_pull` مش موجود ترجع `-EINVAL`. الـ function دي بتتبعت كـ `parse_dt` callback في الـ `meson_pinctrl_data`.

---

#### `meson_a1_parse_dt_extra`

```c
int meson_a1_parse_dt_extra(struct meson_pinctrl *pc)
```

في الـ A1 SoC، الـ GPIO register بيعمل شغل الـ pull والـ drive strength كمان. الـ function بتعمل:
```c
pc->reg_pull   = pc->reg_gpio;
pc->reg_pullen = pc->reg_gpio;
pc->reg_ds     = pc->reg_gpio;
```

ده بيخلي كل الـ regmap operations تروح لنفس الـ base address لكن بـ offsets مختلفة.

---

#### `meson_pinctrl_probe`

```c
int meson_pinctrl_probe(struct platform_device *pdev)
```

الـ entry point الرئيسي للـ driver. بينفذ الـ standard pinctrl probe sequence.

**Pseudocode:**
```
pc = devm_kzalloc(sizeof(meson_pinctrl))
pc->dev  = &pdev->dev
pc->data = of_device_get_match_data(dev)   /* SoC-specific data */

meson_pinctrl_parse_dt(pc)   /* build regmaps */

/* fill pinctrl_desc */
pc->desc.name    = "pinctrl-meson"
pc->desc.pctlops = &meson_pctrl_ops
pc->desc.pmxops  = pc->data->pmx_ops       /* SoC-specific! */
pc->desc.confops = &meson_pinconf_ops
pc->desc.pins    = pc->data->pins
pc->desc.npins   = pc->data->num_pins

pc->pcdev = devm_pinctrl_register(dev, &desc, pc)

meson_gpiolib_register(pc)
```

**Key details:**
- `pmx_ops` بييجي من الـ SoC data — مش uniform زي باقي الـ ops
- كل الـ memory بتتعمل لها manage بـ `devm_*` — الـ cleanup automatic عند الـ disconnect
- الـ probe بيفشل early لو الـ DT ناقص الـ required registers

---

### الـ ops Tables المبنية في الـ Driver

```
meson_pctrl_ops:
    .get_groups_count → meson_get_groups_count
    .get_group_name   → meson_get_group_name
    .get_group_pins   → meson_get_group_pins
    .dt_node_to_map   → pinconf_generic_dt_node_to_map_all  /* generic */
    .dt_free_map      → pinctrl_utils_free_map
    .pin_dbg_show     → meson_pin_dbg_show

meson_pinconf_ops:
    .pin_config_get       → meson_pinconf_get
    .pin_config_set       → meson_pinconf_set
    .pin_config_group_get → meson_pinconf_group_get  /* always -ENOTSUPP */
    .pin_config_group_set → meson_pinconf_group_set
    .is_generic           → true
```

---

### Call Flow: تطبيق Pull-Up على Pin

```
DT: pinctrl-0 = <&uart_pins>;
        ↓
pinctrl_select_state()
        ↓
meson_pinconf_set(pcdev, pin, [BIAS_PULL_UP])
        ↓
meson_pinconf_enable_bias(pc, pin, true)
        ↓
  meson_get_bank(pc, pin, &bank)
        ↓
  meson_calc_reg_and_bit(bank, pin, MESON_REG_PULL, &reg, &bit)
  regmap_update_bits(pc->reg_pull, reg, BIT(bit), BIT(bit))   /* direction = up */
        ↓
  meson_calc_reg_and_bit(bank, pin, MESON_REG_PULLEN, &reg, &bit)
  regmap_update_bits(pc->reg_pullen, reg, BIT(bit), BIT(bit)) /* enable pull */
```

---

### Call Flow: GPIO Output

```
gpiod_direction_output(gpio, 1)
        ↓
meson_gpio_direction_output(chip, gpio, 1)
        ↓
meson_pinconf_set_output_drive(pc, gpio, true)
        ↓
  meson_pinconf_set_output(pc, gpio, true)
      → meson_pinconf_set_gpio_bit(pc, gpio, MESON_REG_DIR, false)
          → regmap_update_bits(reg_gpio, DIR_reg, BIT(bit), 0)  /* DIR=0 = output */
        ↓
  meson_pinconf_set_drive(pc, gpio, true)
      → meson_pinconf_set_gpio_bit(pc, gpio, MESON_REG_OUT, true)
          → regmap_update_bits(reg_gpio, OUT_reg, BIT(bit), BIT(bit)) /* OUT=1 */
```
## Phase 5: دليل الـ Debugging الشامل

### Software Level

---

#### 1. debugfs — الـ Entries المهمة

الـ pinctrl subsystem بيعرض معلومات تفصيلية جوا `/sys/kernel/debug/pinctrl/`.

```bash
# اعرف كل الـ pin controllers الموجودين
ls /sys/kernel/debug/pinctrl/

# مثال على Meson SoC (A311D مثلاً):
# /sys/kernel/debug/pinctrl/ff634400.bus:pinctrl@40-pinctrl-meson/
# /sys/kernel/debug/pinctrl/ff800014.bus:pinctrl@14-pinctrl-meson/
```

| الـ Entry | المحتوى | الأمر |
|-----------|---------|-------|
| `pins` | كل الـ pins وأرقامها | `cat /sys/kernel/debug/pinctrl/<dev>/pins` |
| `pingroups` | الـ groups وأعضاؤها | `cat /sys/kernel/debug/pinctrl/<dev>/pingroups` |
| `pinmux-pins` | إيه الـ function مربوطة بكل pin | `cat /sys/kernel/debug/pinctrl/<dev>/pinmux-pins` |
| `pinmux-functions` | كل الـ functions المتاحة | `cat /sys/kernel/debug/pinctrl/<dev>/pinmux-functions` |
| `pinconf-pins` | pull/drive-strength لكل pin | `cat /sys/kernel/debug/pinctrl/<dev>/pinconf-pins` |
| `pinconf-groups` | نفس الكلام على مستوى group | `cat /sys/kernel/debug/pinctrl/<dev>/pinconf-groups` |

```bash
# قراءة حالة الـ pinmux لكل pin:
cat /sys/kernel/debug/pinctrl/ff634400.bus:pinctrl@40-pinctrl-meson/pinmux-pins

# مثال على الـ output:
# pin 0 (GPIOX_0): device ff634400.bus:pinctrl@40 function gpio_periphs, group GPIOX_0
# pin 1 (GPIOX_1): device ff634400.bus:pinctrl@40 function uart_a, group uart_a_tx
```

---

#### 2. sysfs — الـ Entries المهمة

**الـ GPIO subsystem** بيكمل معلومات الـ pinctrl جوا `/sys/class/gpio/` و `/sys/kernel/debug/gpio`.

```bash
# تفاصيل كل GPIO chip موجود في النظام:
cat /sys/kernel/debug/gpio

# مثال output:
# gpiochip0: GPIOs 410-505, parent: platform/ff634400.bus:pinctrl@40, periphs-banks:
#  gpio-410 (GPIOX_0           ) out lo
#  gpio-411 (GPIOX_1           ) in  hi IRQ

# اعرف الـ gpiochip الخاص بالـ controller:
ls /sys/bus/platform/devices/ | grep pinctrl
cat /sys/bus/platform/devices/ff634400.bus:pinctrl@40/uevent
```

| الـ sysfs Path | ما يعرضه |
|----------------|---------|
| `/sys/kernel/debug/gpio` | حالة كل GPIO (direction, value, IRQ) |
| `/sys/class/gpio/gpiochipN/base` | أول رقم GPIO في الـ chip |
| `/sys/class/gpio/gpiochipN/ngpio` | عدد الـ GPIOs |
| `/sys/class/gpio/gpiochipN/label` | اسم الـ controller |
| `/sys/bus/platform/devices/<dev>/driver` | الـ driver المربوط |

---

#### 3. ftrace — الـ Tracepoints والـ Events

```bash
# شوف الـ events المتاحة للـ pinctrl:
ls /sys/kernel/debug/tracing/events/gpio/
ls /sys/kernel/debug/tracing/events/regmap/

# تفعيل تتبع الـ GPIO requests:
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable

# تفعيل تتبع كل الـ regmap reads/writes (مهم لـ Meson لأن الـ registers بتتعامل معها عن طريق regmap):
echo 1 > /sys/kernel/debug/tracing/events/regmap/regmap_reg_read/enable
echo 1 > /sys/kernel/debug/tracing/events/regmap/regmap_reg_write/enable

# تفعيل tracing وقراءة النتائج:
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... شغل العملية اللي عايز تراقبها ...
cat /sys/kernel/debug/tracing/trace

# تصفية على device معين بس:
echo 'name == "ff634400.bus:pinctrl@40"' > /sys/kernel/debug/tracing/events/regmap/regmap_reg_write/filter
```

```bash
# استخدام function_graph tracer لتتبع meson_pinctrl functions:
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo 'meson_*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe
```

---

#### 4. printk و Dynamic Debug

```bash
# تفعيل dynamic debug للـ pinctrl subsystem كله:
echo 'file drivers/pinctrl/meson/* +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل للـ core pinctrl أيضاً:
echo 'file drivers/pinctrl/core.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/pinctrl/pinmux.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/pinctrl/pinconf.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل كل debug messages في الـ gpio driver أيضاً:
echo 'file drivers/gpio/gpiolib.c +p' > /sys/kernel/debug/dynamic_debug/control

# تغيير loglevel وقت الـ boot لرؤية كل الـ messages:
# في kernel cmdline:
# loglevel=8 dyndbg="file drivers/pinctrl/meson/* +p"
```

```bash
# مشاهدة الـ kernel messages المباشرة:
dmesg -w | grep -i 'pinctrl\|meson.*pin\|gpio'

# أو من /dev/kmsg:
cat /dev/kmsg | grep pinctrl
```

---

#### 5. Kernel Config Options للـ Debugging

| الـ Config | الوظيفة | ملاحظة |
|-----------|---------|--------|
| `CONFIG_DEBUG_PINCTRL` | تفعيل debug messages في الـ pinctrl core | ضروري |
| `CONFIG_PINCTRL_MESON` | الـ driver نفسه | يجب أن يكون `=y` أو `=m` |
| `CONFIG_GPIOLIB` | دعم الـ GPIO subsystem | مطلوب |
| `CONFIG_DEBUG_GPIO` | debug messages لـ GPIO | مفيد جداً |
| `CONFIG_GPIO_SYSFS` | تصدير الـ GPIOs عبر sysfs | للتشخيص اليدوي |
| `CONFIG_REGMAP_DEBUGFS` | debugfs للـ regmap (مهم جداً لـ Meson) | يعرض كل الـ register values |
| `CONFIG_DEBUG_FS` | تفعيل debugfs | متطلب أساسي |
| `CONFIG_DYNAMIC_DEBUG` | تفعيل dynamic debug | للـ `+p` flags |
| `CONFIG_PINCTRL_SINGLE` | مقارنة مع driver أبسط | للـ debugging المقارن |
| `CONFIG_PROVE_LOCKING` | كشف deadlocks في الـ spinlocks | مفيد لـ GPIO IRQ |

```bash
# تحقق من الـ configs الحالية:
zcat /proc/config.gz | grep -E 'PINCTRL|GPIO_DEBUG|REGMAP_DEBUG'
```

---

#### 6. أدوات خاصة بالـ Subsystem

**الـ regmap debugfs** هو الأداة الأهم لـ Meson pinctrl لأن كل الـ register access بيتم عبر `regmap`.

```bash
# شوف كل الـ regmap instances:
ls /sys/kernel/debug/regmap/

# كل instance اسمه اسم الـ device:
# ff634400.bus:pinctrl@40-reg_mux
# ff634400.bus:pinctrl@40-reg_gpio
# ff634400.bus:pinctrl@40-reg_pull
# ff634400.bus:pinctrl@40-reg_pullen
# ff634400.bus:pinctrl@40-reg_ds

# اقرأ كل registers في regmap:
cat /sys/kernel/debug/regmap/ff634400.bus:pinctrl@40-reg_gpio/registers

# مثال output (offset: value):
# 00: 00000000
# 04: ff00ff00
# 08: 00000000

# اقرأ قيمة register معين:
cat /sys/kernel/debug/regmap/ff634400.bus:pinctrl@40-reg_gpio/access
```

```bash
# gpioinfo من package libgpiod (أداة userspace مفيدة جداً):
gpioinfo
# Output:
# gpiochip0 - 96 lines:
#     line   0:   "GPIOX_0"  unused   input  active-high
#     line   1:   "GPIOX_1"  "uart-rx" used   input  active-high [kernel]

# gpiodetect: اعرف كل الـ chips:
gpiodetect

# gpioget: اقرأ قيمة pin:
gpioget gpiochip0 5

# gpiomon: راقب تغيرات pin:
gpiomon gpiochip0 5
```

---

#### 7. جدول الـ Error Messages الشائعة

| الـ Message | المعنى | الحل |
|-------------|--------|------|
| `pinctrl_get() failed for state default` | الـ consumer device مش لاقي pinctrl state | تحقق من الـ DT: `pinctrl-0` و `pinctrl-names` موجودين |
| `could not get gpio` | رقم الـ GPIO مش موجود أو محجوز | تأكد من الـ gpio-ranges في DT ومن `gpio_request()` |
| `pin X is already requested` | الـ pin محجوز من device تاني | افحص `cat /sys/kernel/debug/pinctrl/.../pinmux-pins` |
| `meson-pinctrl: probe of ... failed with error -ENODEV` | مشكلة في الـ regmap init أو الـ clock | تحقق من الـ DT `reg` property وانتهاء الـ clock init |
| `pinmux_request_gpio() failed` | فشل تحويل الـ pin لـ GPIO mode | الـ pin مش في الـ GPIO bank أو محجوز |
| `Failed to apply pinctrl state` | فشل تطبيق الـ state | سبب الـ error في السطر اللي فوقيه في dmesg |
| `regmap_read failed` | مشكلة في قراءة الـ register | تحقق من الـ clock وتحقق من أن الـ regmap مش NULL |
| `invalid pin number` | رقم الـ pin خارج النطاق | تحقق من `num_pins` في `meson_pinctrl_data` |
| `pin does not support drive-strength` | الـ bank مش عنده `MESON_REG_DS` | الـ BANK macro (مش BANK_DS) مش بيدعم DS |
| `GPIO IRQ not mapped` | الـ `irq_first/irq_last` مش معمول لهم mapping | تحقق من الـ irq-controller في DT |

---

#### 8. أماكن استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في meson_pmx_set() — عند تغيير الـ mux */
static int meson_pmx_set(struct pinctrl_dev *pcdev,
                          unsigned func_selector,
                          unsigned group_selector)
{
    struct meson_pinctrl *pc = pinctrl_dev_get_drvdata(pcdev);
    /* نقطة مناسبة للـ WARN_ON إذا كان الـ regmap NULL */
    WARN_ON(!pc->reg_mux);
    ...
}

/* في meson_pinconf_set() — عند ضبط pull/drive-strength */
/* WARN_ON إذا كان الـ pin خارج نطاق الـ bank */
WARN_ON(pin < bank->first || pin > bank->last);

/* في meson_gpio_request() */
/* dump_stack لتتبع مين طلب الـ GPIO */
#ifdef DEBUG
    dump_stack();
#endif

/* WARN_ON على القيم غير المتوقعة من enum meson_pinconf_drv */
WARN_ON(drive > MESON_PINCONF_DRV_4000UA);
```

---

### Hardware Level

---

#### 1. التحقق من تطابق حالة الـ Hardware مع الـ Kernel

الـ Meson SoC بيستخدم **MMIO registers** للـ pinctrl، مقسمة لـ 5 regmaps:

- `reg_mux` — الـ function selection (PINMUX)
- `reg_gpio` — الـ direction (OEN) وقراءة/كتابة الـ pins
- `reg_pull` — pull-up/pull-down enable
- `reg_pullen` — pull enable/disable
- `reg_ds` — drive-strength

```bash
# قارن القيمة الـ hardware مع ما يعرفه الـ kernel:

# 1. اقرأ من regmap debugfs (kernel view):
cat /sys/kernel/debug/regmap/ff634400.bus:pinctrl@40-reg_gpio/registers

# 2. اقرأ مباشرة من الـ hardware (hardware view) باستخدام devmem2:
# مثال: base address الـ GPIO registers على Amlogic A311D = 0xff634440
devmem2 0xff634440 w   # OEN register للـ GPIOX
devmem2 0xff634444 w   # Output register
devmem2 0xff634448 w   # Input register

# لو القيمتين مختلفتين → مشكلة في الـ regmap cache أو الـ bus
```

---

#### 2. Register Dump Techniques

```bash
# تثبيت devmem2:
apt-get install devmem2
# أو build من source إذا مش متاح

# قراءة word من address معين:
devmem2 0xff634440 w

# كتابة قيمة (تأكد إنك تعرف إيه اللي بتعمله!):
devmem2 0xff634440 w 0x00000001

# /dev/mem بديل (محتاج CONFIG_STRICT_DEVMEM=n في الـ kernel):
# قراءة 4 bytes من address 0xff634440:
dd if=/dev/mem bs=4 count=1 skip=$((0xff634440/4)) 2>/dev/null | xxd

# استخدام io utility:
io -4 0xff634440         # قراءة 32-bit
io -4 0xff634440 0x0001  # كتابة
```

**الـ Base Addresses المهمة على Amlogic SoCs:**

| الـ SoC | الـ Periphs Base | الـ AO Base |
|---------|----------------|------------|
| Meson8/8b | `0xC8834000` | `0xC8100000` |
| Meson GXL/GXM | `0xC8834000` | `0xC8100000` |
| Meson G12A/G12B | `0xFF634400` | `0xFF800000` |
| Meson SM1 | `0xFF634400` | `0xFF800000` |
| Meson A1 | `0xFE004000` | — |

```bash
# dump كل الـ pinctrl registers دفعة واحدة باستخدام /proc/iomem:
cat /proc/iomem | grep -i 'pinctrl\|periphs\|gpio'
# يعرض الـ address ranges المعروفة للـ kernel
```

---

#### 3. نصائح الـ Logic Analyzer / Oscilloscope

**نقاط القياس:**

```
مثال: تشخيص UART TX pin مش شتغال

1. حدد الـ pin على الـ schematic (مثلاً: GPIOX_0 = UART_A_TX)
2. بالـ oscilloscope:
   - راقب الـ pin قبل تطبيق الـ pinctrl state
   - راقب بعد تطبيقه
   - المتوقع: الـ pin يتغير من floating/low لـ UART waveform

3. بالـ logic analyzer:
   - Protocol: UART، بجد صح Baud Rate
   - يفيد تأكيد إن الـ data صح مش بس الـ signal موجود
```

**اختبار الـ GPIO بشكل مباشر:**

```bash
# export GPIO وتبديل قيمته وراقب بالـ oscilloscope:
# رقم GPIO = chip_base + pin_offset
# مثلاً GPIOX_0 على G12A = 410 + 0 = 410
echo 410 > /sys/class/gpio/export
echo out > /sys/class/gpio/gpio410/direction
echo 1 > /sys/class/gpio/gpio410/value
# شوف الـ pin بالـ oscilloscope — لازم يطلع high
echo 0 > /sys/class/gpio/gpio410/value
# لازم يرجع low
```

**مشاكل شائعة بالـ logic analyzer:**

| الأعراض | التفسير |
|---------|---------|
| الـ pin دايماً low حتى بعد `echo 1` | الـ OEN register (direction) مش اتضبط صح → الـ pin لسه input |
| الـ pin مش بيتغير خالص | الـ pull-down قوي جداً أو short circuit |
| signal غريب oscillating | conflict بين driver تاني وبين الـ GPIO |
| UART signal موجود بس corrupt | الـ baud rate أو الـ mux function غلط |

---

#### 4. المشاكل الشائعة وـ kernel log patterns

| المشكلة الـ Hardware | الـ Pattern في الـ Kernel Log |
|---------------------|------------------------------|
| الـ power domain للـ pin controller مش شغال | `pinctrl-meson: probe deferred` أو `regmap init failed` |
| clock للـ bus مش enabled | `regmap_read: failed` مع `-ETIMEDOUT` |
| عنوان الـ register غلط في DT | `unable to ioremap` أو `devm_ioremap_resource failed` |
| الـ GPIO interrupt line مش متوصل | `irq: no irq domain found` |
| تعارض مع bootloader config | pin يعمل بشكل عكسي — bootloader غير الـ pull |
| الـ pin مش موجود في الـ SoC version ده | `invalid pin number` في dmesg |

---

#### 5. Device Tree Debugging

**التحقق من الـ DT:**

```bash
# اقرأ الـ compiled DT من الـ running kernel:
ls /sys/firmware/devicetree/base/soc/bus@ff600000/pinctrl@40/
# يعرض كل الـ nodes والـ properties

# اقرأ property معينة:
xxd /sys/firmware/devicetree/base/soc/bus@ff600000/pinctrl@40/reg
# المفروض يطلع base address وsize

# تحويل من binary لـ DTS مقروء:
dtc -I fs /sys/firmware/devicetree/base > /tmp/running.dts
grep -A 20 'pinctrl@40' /tmp/running.dts

# قارن مع الـ DTS المصدر:
dtc -I dtb -O dts /boot/dtbs/amlogic/meson-g12a-your-board.dtb | grep -A 20 'pinctrl'
```

**الـ DT Properties المهمة للتحقق:**

```dts
/* مثال DT صحيح لـ Meson G12A periphs pinctrl */
periphs_pinctrl: pinctrl@40 {
    compatible = "amlogic,meson-g12a-periphs-pinctrl";
    #address-cells = <2>;
    #size-cells = <2>;
    reg = <0x0 0x40 0x0 0x4>;  /* mux registers */
    ranges;

    gpio: gpio@4000 {
        reg = <0x0 0x4000 0x0 0x200>,  /* gpio */
              <0x0 0x4800 0x0 0x200>,  /* pull-enable */
              <0x0 0x4c00 0x0 0x200>,  /* pull */
              <0x0 0x4e00 0x0 0x200>;  /* drive-strength */
        reg-names = "gpio", "pullen", "pull", "ds";
        gpio-controller;
        #gpio-cells = <2>;
        gpio-ranges = <&periphs_pinctrl 0 0 100>;
        interrupt-controller;
        #interrupt-cells = <2>;
        interrupts = <GIC_SPI 64 IRQ_TYPE_EDGE_RISING>, ...;
    };
};
```

```bash
# تحقق من الـ gpio-ranges:
cat /sys/firmware/devicetree/base/soc/bus@ff600000/pinctrl@40/gpio@4000/gpio-ranges | xxd

# تحقق من إن الـ pinctrl consumer صح:
# مثلاً UART device:
cat /sys/firmware/devicetree/base/soc/bus@ffd24000/serial@24000/pinctrl-0 | xxd
# المفروض يشاور على phandle موجود في الـ periphs_pinctrl node
```

---

### Practical Commands

---

#### مجموعة أوامر شاملة جاهزة للنسخ

```bash
#!/bin/bash
# === Meson Pinctrl Full Debug Script ===

PINCTRL_DEV=$(ls /sys/kernel/debug/pinctrl/ | head -1)
echo "=== Using pinctrl device: $PINCTRL_DEV ==="

echo ""
echo "=== [1] All pins and their current mux state ==="
cat /sys/kernel/debug/pinctrl/$PINCTRL_DEV/pinmux-pins

echo ""
echo "=== [2] All available functions ==="
cat /sys/kernel/debug/pinctrl/$PINCTRL_DEV/pinmux-functions

echo ""
echo "=== [3] Pin groups ==="
cat /sys/kernel/debug/pinctrl/$PINCTRL_DEV/pingroups

echo ""
echo "=== [4] GPIO state ==="
cat /sys/kernel/debug/gpio

echo ""
echo "=== [5] Regmap registers (GPIO bank) ==="
REGMAP=$(ls /sys/kernel/debug/regmap/ | grep 'reg_gpio' | head -1)
[ -n "$REGMAP" ] && cat /sys/kernel/debug/regmap/$REGMAP/registers

echo ""
echo "=== [6] Recent pinctrl/gpio kernel messages ==="
dmesg | grep -E 'pinctrl|meson.*pin|gpio' | tail -30
```

```bash
# تفعيل dynamic debug وتتبع probe:
echo 8 > /proc/sys/kernel/printk
echo 'file drivers/pinctrl/meson/* +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/pinctrl/core.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# إعادة bind الـ driver لرؤية الـ probe messages:
DEVICE="ff634400.bus:pinctrl@40"
echo $DEVICE > /sys/bus/platform/drivers/pinctrl-meson/unbind
echo $DEVICE > /sys/bus/platform/drivers/pinctrl-meson/bind
dmesg | tail -50
```

```bash
# تفعيل ftrace لـ regmap writes بس:
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo > trace
echo 1 > events/regmap/regmap_reg_write/enable
echo 1 > events/regmap/regmap_reg_read/enable
echo 1 > tracing_on

# شغل العملية اللي بتتشخص (مثلاً: set pinmux)
echo 0 > tracing_on
cat trace | grep -E 'reg_(mux|gpio|pull)'
```

---

#### كيفية تفسير الـ Output

**مثال: pinmux-pins output:**

```
pin 72 (GPIOX_0): device ff634400.bus:pinctrl@40 function uart_a, group uart_a_tx
pin 73 (GPIOX_1): device ff634400.bus:pinctrl@40 function uart_a, group uart_a_rx
pin 74 (GPIOX_2): UNCLAIMED
```

- `UNCLAIMED` = الـ pin مش محجوز من أي driver — ممكن يكون مشكلة لو المفروض يكون configured
- `function uart_a` = الـ mux اتضبط صح للـ UART
- `device ...` = الـ pin controller اللي بيتحكم فيه

**مثال: regmap registers output:**

```
00: 00000003   ← bits 0,1 = 1: GPIOX_0 و GPIOX_1 في mode "function" مش GPIO
04: 00000000   ← output register: كل القيم = 0
08: 00000005   ← input register: bits 0,2 = 1 يعني GPIOX_0 و GPIOX_2 قارين high
```

**مثال: dmesg أثناء probe ناجح:**

```
[    2.341] meson-pinctrl ff634400.bus:pinctrl@40: probing periphs-banks with 100 pins
[    2.342] meson-pinctrl ff634400.bus:pinctrl@40: registered pinctrl driver
[    2.350] meson-pinctrl ff634400.bus:pinctrl@40: initialized GPIO chip with 100 lines
```

**مثال: dmesg أثناء مشكلة:**

```
[    2.341] meson-pinctrl ff634400.bus:pinctrl@40: failed to get regmap for gpio: -ENODEV
```
→ يعني الـ `reg-names = "gpio"` مش موجود في الـ DT أو الـ node نفسه غلط.
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: UART مش شغال على Android TV Box بـ Amlogic S905X3

#### العنوان
**UART debug console صامت تماماً بعد تشغيل الجهاز**

#### السياق
بتشتغل على Android TV box بيستخدم **Amlogic S905X3**. الـ board بيستخدم UART_A كـ debug console على pins GPIOAO_0 وGPIOAO_1. بعد flash الـ kernel الجديد، الـ serial console بطلت تشتغل — مفيش output خالص حتى في early boot.

#### المشكلة
الـ UART_A function مش بيتطبق على الـ pins الصح. الـ pin controller بـ probe عادي ومفيش kernel panic، لكن الـ UART صامت.

#### التحليل
الـ driver بيستخدم `meson_pinctrl_data` عشان يعرف الـ groups والـ functions:

```c
struct meson_pinctrl_data {
    const char *name;
    const struct meson_pmx_group *groups;   /* array of all pin groups */
    const struct meson_pmx_func  *funcs;    /* array of all functions  */
    ...
};
```

كل `meson_pmx_group` بيوصف group من الـ pins اللي بتعمل وظيفة معينة:

```c
struct meson_pmx_group {
    const char       *name;
    const unsigned int *pins;     /* which physical pins */
    unsigned int      num_pins;
    const void        *data;      /* mux register/bit info */
};
```

الـ `FUNCTION` macro بيربط function باسمها بالـ groups المرتبطة بيها:

```c
#define FUNCTION(fn)                            \
    {                                           \
        .name   = #fn,                          \
        /* expands fn##_groups array name */    \
        .groups = fn ## _groups,                \
        .num_groups = ARRAY_SIZE(fn ## _groups),\
    }
```

لما بفتح ملف `pinctrl-meson-s4.c` (S905X3 pinctrl SoC file) لاقيت إن الـ group المسؤول عن UART_A هو `uart_a_tx_aobus_group` بس الـ `FUNCTION(uart_a)` بيشاور على array اسمها `uart_a_groups` — لو في mistype أو الـ group ناقصة من الـ array، الـ function بيُسجَّل بدون الـ group الصح.

راحت افتح `/sys/kernel/debug/pinctrl/`:

```bash
cat /sys/kernel/debug/pinctrl/pinctrl-meson-aobus/pinmux-functions
```

لاقيت `uart_a` موجودة بس `num_groups = 0` — يعني الـ `uart_a_groups[]` array فاضية أو فيها compile-time error صامت.

#### الحل
في الـ SoC-specific file، تأكدت إن الـ `uart_a_groups` array موجودة ومتعرَّفة صح قبل الـ `FUNCTION(uart_a)`:

```c
/* في ملف pinctrl-meson-s4.c — تأكد إن دي موجودة */
static const char * const uart_a_groups[] = {
    "uart_a_tx", "uart_a_rx", "uart_a_cts", "uart_a_rts",
};

/* بعدين */
static const struct meson_pmx_func meson_s4_aobus_funcs[] = {
    FUNCTION(gpio_aobus),
    FUNCTION(uart_a),   /* دلوقتي بيلاقي uart_a_groups صح */
    ...
};
```

بعد rebuild وflash، الـ UART console رجعت تشتغل.

#### الدرس المستفاد
الـ `FUNCTION` macro بيعمل string concatenation للـ array name — لو الـ array غلط اسمها أو مش معرَّفة، compiler بياخد error بس أحياناً linker بيقبلها لو في forward declaration. دايماً اتحقق من `pinmux-functions` في debugfs قبل ما تبدأ تشتغل على hardware.

---

### السيناريو الثاني: I2C بتشتغل slow جداً على Industrial Gateway بـ Amlogic A311D

#### العنوان
**I2C sensor data بتيجي متأخرة — drive strength غلط على الـ SDA/SCL pins**

#### السياق
بتعمل industrial IoT gateway بـ **Amlogic A311D** (Khadas VIM3 SoC). الـ gateway بيقرأ من 8 I2C sensors بـ 400 kHz (Fast Mode). العملاء بيشتكوا من latency عالية في readings. بالـ oscilloscope، شايف الـ rising edges على SDA وSCL بطيئة جداً — كأن الـ pull-up مش كافي.

#### المشكلة
الـ drive strength على I2C pins مضبوطة على أقل قيمة `MESON_PINCONF_DRV_500UA` بدل `MESON_PINCONF_DRV_2500UA`.

#### التحليل
الـ header بيعرّف enum الـ drive strength:

```c
enum meson_pinconf_drv {
    MESON_PINCONF_DRV_500UA,   /* 0.5 mA — أضعف */
    MESON_PINCONF_DRV_2500UA,  /* 2.5 mA */
    MESON_PINCONF_DRV_3000UA,  /* 3.0 mA */
    MESON_PINCONF_DRV_4000UA,  /* 4.0 mA — أقوى */
};
```

الـ `meson_bank` بيحتوي على `MESON_REG_DS` descriptor:

```c
struct meson_bank {
    const char *name;
    unsigned int first;
    unsigned int last;
    int irq_first;
    int irq_last;
    struct meson_reg_desc regs[MESON_NUM_REG]; /* بيشمل MESON_REG_DS */
};
```

والـ `BANK_DS` macro بيحدد offset وbit لـ drive strength register:

```c
#define BANK_DS(n, f, l, fi, li, per, peb, pr, pb, dr, db, or, ob, ir, ib, \
                dsr, dsb)                                                    \
    {                                                                        \
        ...                                                                  \
        .regs = {                                                            \
            ...                                                              \
            [MESON_REG_DS] = { dsr, dsb }, /* DS reg offset, bit */         \
        },                                                                   \
    }
```

الـ `meson_pinctrl` struct بيحتفظ بـ `reg_ds` regmap منفصل:

```c
struct meson_pinctrl {
    ...
    struct regmap *reg_ds;   /* separate regmap for drive strength */
    ...
};
```

راحت فتحت debugfs:

```bash
cat /sys/kernel/debug/pinctrl/pinctrl-meson-a311d/pinconf-pins | grep i2c_m0
```

Output أظهر:
```
pin 23 (GPIOA_14): drive-strength-microamp:500
pin 24 (GPIOA_15): drive-strength-microamp:500
```

#### الحل
في الـ device tree، زودت `drive-strength-microamp`:

```dts
&i2c0 {
    pinctrl-0 = <&i2c0_pins>;
    pinctrl-names = "default";
    status = "okay";
};

&pinctrl {
    i2c0_pins: i2c0-pins {
        mux {
            groups = "i2c0_sda", "i2c0_scl";
            function = "i2c0";
            drive-strength-microamp = <2500>;  /* بدل default 500 */
        };
    };
};
```

بعد reboot، الـ rising edges اتحسنت وI2C شغالة بـ 400 kHz صح.

#### الدرس المستفاد
الـ `MESON_REG_DS` field في `meson_bank` بيتحكم في drive strength على مستوى الـ register. الـ default value هو `MESON_PINCONF_DRV_500UA` اللي كافي للـ low-speed interfaces بس مش للـ Fast-Mode I2C مع long traces أو أكتر من 4 sensors.

---

### السيناريو الثالث: GPIO interrupt مش بيشتغل على IoT Sensor Board بـ Amlogic S905X3

#### العنوان
**GPIO edge interrupt من PIR motion sensor مش بيصحّي الـ system من suspend**

#### السياق
IoT security device بـ **Amlogic S905X3**. فيه PIR motion sensor متوصل بـ GPIOX_10. الـ system المفروض يصحى من suspend لما الـ PIR يشوف حركة. الـ GPIO interrupt شغال وهو يصحى بس بعد suspend مش بيصحّيش.

#### المشكلة
الـ `irq_first` و`irq_last` في `meson_bank` الخاص بـ GPIOX bank غلط، فالـ hwirq mapping بـ irq domain بيفشل بعد resume.

#### التحليل
الـ `meson_bank` struct بيخزن الـ IRQ range لكل bank:

```c
struct meson_bank {
    const char   *name;
    unsigned int  first;      /* first pin number in bank */
    unsigned int  last;       /* last pin number in bank  */
    int irq_first;            /* first hwirq in this bank */
    int irq_last;             /* last hwirq in this bank  */
    struct meson_reg_desc regs[MESON_NUM_REG];
};
```

الـ `BANK` macro بيمرر الـ IRQ params:

```c
#define BANK(n, f, l, fi, li, per, peb, pr, pb, dr, db, or, ob, ir, ib) \
    BANK_DS(n, f, l, fi, li, ...)
/* fi = irq_first, li = irq_last */
```

في ملف الـ S905X3 pinctrl، لاقيت:

```c
static const struct meson_bank meson_s905x3_periphs_banks[] = {
    /* BANK(name, first, last, irq_first, irq_last, ...) */
    BANK("X",  81, 96,  0,  15, ...),  /* GPIOX: pins 81-96, IRQs 0-15 */
};
```

المشكلة إن GPIOX_10 هو pin رقم 91 (81+10)، والـ hwirq المقابل هو 10 (0+10).

بالـ debugging:

```bash
cat /proc/interrupts | grep gpiox
# لاقيت الـ IRQ registered بس no wakeup capability
cat /sys/kernel/debug/gpio | grep gpiox_10
```

المشكلة الحقيقية: في الـ DT، الـ wakeup-source property ناقصة، والـ irq_first في bank X كان بـ قيمة -1 (غير initialized) في version قديمة من الـ BSP.

#### الحل
تأكدت إن الـ bank definition صح:

```c
/* في pinctrl-meson-s4.c أو equivalent S905X3 file */
BANK("X", 81, 96, 0, 15,
     /* PULLEN: */ 3, 0,
     /* PULL:   */ 4, 0,
     /* DIR:    */ 0, 0,
     /* OUT:    */ 1, 0,
     /* IN:     */ 2, 0),
```

وفي الـ DT:

```dts
pir_sensor: pir@0 {
    compatible = "pirmotion";
    interrupt-parent = <&gpio_intc>;
    interrupts = <10 IRQ_TYPE_EDGE_RISING>;  /* hwirq 10 = GPIOX_10 */
    wakeup-source;                            /* السطر المهم */
};
```

#### الدرس المستفاد
الـ `irq_first` وَ`irq_last` في `meson_bank` بيحددوا الـ hwirq offset. لو الـ bank definition فيها `-1` أو `0` غلط للـ irq range، الـ irqdomain mapping بيفشل بصمت ومفيش wakeup. دايماً اتحقق من `/sys/kernel/debug/irq/irqs/` بعد suspend/resume.

---

### السيناريو الرابع: SPI flash مش بيتقرأ على Custom Board Bring-up بـ Amlogic S905X3

#### العنوان
**SPI NOR flash مش بيظهر في `/dev/mtd*` — pinmux فشل في probe**

#### السياق
Custom industrial board bring-up. الـ SoC هو **Amlogic S905X3**. الـ SPI NOR flash (W25Q128) متوصل بـ GPIOH_4 إلى GPIOH_7 (CS, CLK, MOSI, MISO). الـ kernel يشتغل، بس `/dev/mtd0` مش موجود وkMSG فيها:

```
spi-meson: probe failed: pinctrl select error -22
```

#### المشكلة
اسم الـ function في الـ DT غلط — مش بيطابق الـ `meson_pmx_func.name` المعرَّف في الـ driver.

#### التحليل
الـ `meson_pmx_func` struct:

```c
struct meson_pmx_func {
    const char          *name;           /* function name — must match DT exactly */
    const char * const  *groups;         /* which pin groups implement it */
    unsigned int         num_groups;
};
```

الـ `FUNCTION` macro بيعمل stringify للاسم:

```c
#define FUNCTION(fn)                    \
    {                                   \
        .name = #fn,                    \ /* "spifc" مثلاً */
        .groups = fn ## _groups,        \
        ...                             \
    }
```

يعني لو الـ function اتعرّفت بـ `FUNCTION(spifc)` فالاسم هو `"spifc"` حرف بحرف. في الـ DT كاتب:

```dts
spi0_pins: spi0-pins {
    mux {
        groups = "spi0_mosi", "spi0_miso", "spi0_clk", "spi0_ss0";
        function = "spi0";  /* غلط! الاسم الصح هو "spifc" */
    };
};
```

التحقق:

```bash
cat /sys/kernel/debug/pinctrl/pinctrl-meson-periphs/pinmux-functions
# بيظهر: spifc, uart_a, i2c0, ...
# مفيش "spi0" خالص
```

#### الحل
صحّحت الـ DT:

```dts
spi0_pins: spi0-pins {
    mux {
        groups = "spifc_mosi", "spifc_miso",
                 "spifc_clk",  "spifc_cs0";
        function = "spifc";   /* الاسم الصح من meson_pmx_func.name */
    };
};
```

بعد rebuild DTB وreboot:

```bash
ls /dev/mtd*
# /dev/mtd0  /dev/mtd0ro
```

#### الدرس المستفاد
الـ `FUNCTION(fn)` macro بيحوّل الـ identifier مباشرة لـ string. أي typo أو اختلاف case بين الـ DT وبين اسم الـ function في الـ C code بيخلي الـ pinctrl core يرجع `-EINVAL` صامت. دايماً اعمل cross-check بين الـ DT `function = "..."` وبين output `pinmux-functions` في debugfs.

---

### السيناريو الخامس: PWM للـ Fan Control مش شغال على Automotive ECU بـ Amlogic A311D

#### العنوان
**PWM cooling fan مش بيستجوبش للـ thermal governor — pin stuck كـ GPIO**

#### السياق
Automotive ECU prototype بـ **Amlogic A311D**. الـ PWM_E متوصل بـ GPIOA_11 لتحكم fan. الـ thermal governor (step_wise) بيحاول يتحكم في الـ fan بس الـ fan شغال بـ full speed دايماً أو واقف — مفيش PWM duty cycle control.

#### المشكلة
الـ pin مش بيتحوّل من GPIO mode لـ PWM function. الـ `meson_pmx_group` للـ `pwm_e` group مش بتتطابق مع الـ pin المستخدم.

#### التحليل
الـ pinmux logic بيمر بـ `meson_pmx_group` عشان يعرف أي register/bit يكتب فيه:

```c
struct meson_pmx_group {
    const char        *name;
    const unsigned int *pins;    /* physical pin numbers in this group */
    unsigned int       num_pins;
    const void        *data;     /* points to mux register/bit — SoC specific */
};
```

الـ `data` pointer بيشاور على struct SoC-specific زي `meson_pmx_group_data` اللي بيحتوي على الـ register offset والـ bit للـ mux. لو الـ `pins[]` array في الـ `pwm_e` group بتحتوي على pin رقم تاني (مش GPIOA_11)، الـ mux بيتكتب على الـ pin الغلط.

بالـ debugfs:

```bash
cat /sys/kernel/debug/pinctrl/pinctrl-meson-a311d/pinmux-pins | grep GPIOA_11
# Output:
# pin 43 (GPIOA_11): GPIO GPIOA_11
# مش بيقول pwm_e
```

يعني الـ function موجودة بس الـ pin مش موجود في الـ group. فتحت `pinctrl-meson-a1.c` (closest to A311D in upstream) ولاقيت:

```c
static const unsigned int pwm_e_pins[] = { 42 }; /* GPIOA_10, مش 43 */
```

يعني الـ hardware engineer وصّل الـ PWM على GPIOA_11 بس الـ software config بتفترض GPIOA_10.

#### الحل
خيارين:

**1) غيّر الـ DT عشان تستخدم الـ pin الصح GPIOA_10:**

```dts
&pwm_e {
    pinctrl-0 = <&pwm_e_pins>;
    pinctrl-names = "default";
    status = "okay";
};

&pinctrl {
    pwm_e_pins: pwm-e-pins {
        mux {
            groups = "pwm_e";    /* يستخدم GPIOA_10 */
            function = "pwm_e";
        };
    };
};
```

**2) لو مستحيل تغيّر الـ hardware، أضف group جديدة في الـ SoC pinctrl file:**

```c
/* في pinctrl-meson-a311d.c */
static const unsigned int pwm_e_gpioa11_pins[] = { 43 }; /* GPIOA_11 */

static const struct meson_pmx_group meson_a311d_extra_groups[] = {
    {
        .name    = "pwm_e_alt",
        .pins    = pwm_e_gpioa11_pins,
        .num_pins = 1,
        .data    = &(struct meson_pmx_group_data){ .reg = X, .bit = Y },
    },
};
```

وتضيف `"pwm_e_alt"` لـ `pwm_e_groups[]` وتعمل rebuild.

للـ debugging السريع:

```bash
# تحقق من الـ thermal governor
cat /sys/class/thermal/cooling_device0/type
cat /sys/class/thermal/cooling_device0/cur_state

# اختبر الـ PWM يدوياً
echo 1 > /sys/class/pwm/pwmchip0/export
echo 1000000 > /sys/class/pwm/pwmchip0/pwm0/period
echo 500000  > /sys/class/pwm/pwmchip0/pwm0/duty_cycle
echo 1       > /sys/class/pwm/pwmchip0/pwm0/enable
```

#### الدرس المستفاد
الـ `meson_pmx_group.pins[]` array هي الـ source of truth للـ pin-to-function mapping. أي mismatch بين الـ hardware schematic والـ software group definition بيخلي الـ function تتكتب على الـ register الغلط بصمت — والـ pin الأصلي فاضل كـ GPIO. في الـ automotive projects، دايماً اعمل hardware/software pin matrix review قبل bring-up.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

#### الـ pinctrl subsystem — المقالات الأساسية

| المقال | الرابط |
|--------|--------|
| The pin control subsystem (مقالة تأسيسية) | [lwn.net/Articles/468759](https://lwn.net/Articles/468759/) |
| Documentation/pinctrl.txt — early kernel doc | [lwn.net/Articles/465077](https://lwn.net/Articles/465077/) |
| drivers: create a pin control subsystem | [lwn.net/Articles/463335](https://lwn.net/Articles/463335/) |
| pin controller subsystem v7 | [lwn.net/Articles/459190](https://lwn.net/Articles/459190/) |
| pinctrl: add a pin config interface | [lwn.net/Articles/471826](https://lwn.net/Articles/471826/) |
| pinctrl: add a generic pin config interface | [lwn.net/Articles/467269](https://lwn.net/Articles/467269/) |
| pinctrl: introduce the concept of a GPIO pin function category | [lwn.net/Articles/1031226](https://lwn.net/Articles/1031226/) |

#### الـ Meson/Amlogic pinctrl — patches وصلت mainline

| المقال | الرابط |
|--------|--------|
| Pinctrl driver for Amlogic Meson SoCs (الـ patch الأولية) | [lwn.net/Articles/616225](https://lwn.net/Articles/616225/) |
| Amlogic Meson pinctrl driver (revision) | [lwn.net/Articles/620822](https://lwn.net/Articles/620822/) |
| pinctrl: meson-a1: add pinctrl driver | [lwn.net/Articles/804174](https://lwn.net/Articles/804174/) |
| pinctrl: meson-s4: add pinctrl driver | [lwn.net/Articles/879912](https://lwn.net/Articles/879912/) |
| Add pinctrl driver support for Amlogic T7 SoCs | [lwn.net/Articles/945285](https://lwn.net/Articles/945285/) |
| Add support for Amlogic S7/S7D/S6 pinctrl | [lwn.net/Articles/1022709](https://lwn.net/Articles/1022709/) |
| irqchip: meson: add GPIO interrupt controller support | [lwn.net/Articles/703946](https://lwn.net/Articles/703946/) |

---

### توثيق الـ Kernel الرسمي

الـ documentation الرئيسية موجودة في شجرة المصدر تحت `Documentation/` وعلى الموقع الرسمي:

| الملف / الصفحة | الرابط |
|----------------|--------|
| `Documentation/driver-api/pin-control.rst` — التوثيق الكامل للـ pinctrl subsystem | [docs.kernel.org/driver-api/pin-control.html](https://docs.kernel.org/driver-api/pin-control.html) |
| PINCTRL subsystem (kernel v4.13 archive) | [kernel.org/doc/html/v4.13/driver-api/pinctl.html](https://www.kernel.org/doc/html/v4.13/driver-api/pinctl.html) |

**الملفات المرتبطة مباشرة بالكود:**

```
drivers/pinctrl/meson/pinctrl-meson.h     ← الملف المدروس
drivers/pinctrl/meson/pinctrl-meson.c     ← التنفيذ الأساسي
drivers/pinctrl/meson/pinctrl-meson8.c    ← بيانات Meson8 SoC
drivers/pinctrl/meson/pinctrl-meson-axg.c ← بيانات AXG family
drivers/pinctrl/meson/pinctrl-meson-g12a.c← بيانات G12A/G12B
include/linux/pinctrl/pinctrl.h           ← الـ API الأساسي
include/linux/pinctrl/pinmux.h            ← الـ pinmux ops
include/linux/pinctrl/pinconf.h           ← الـ pinconf ops
include/linux/gpio/driver.h               ← الـ GPIO chip integration
```

---

### الـ Device Tree Bindings

الـ bindings الخاصة بـ Amlogic Meson pinctrl موجودة في:

```
Documentation/devicetree/bindings/pinctrl/amlogic,meson-pinctrl.yaml
Documentation/devicetree/bindings/pinctrl/amlogic,meson-pinctrl-a1.yaml
```

---

### Commits مهمة في تاريخ الكود

| الوصف | الرابط |
|-------|--------|
| الـ commit الأصلي للـ pinctrl-meson driver (أكتوبر 2014) — بقلم Beniamino Galvani | [lkml.iu.edu — PATCH 1/3 pinctrl: add driver for Amlogic Meson SoCs](https://lkml.iu.edu/hypermail/linux/kernel/1410.0/04163.html) |
| إضافة دعم GPIO interrupts — patch v6 | [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-amlogic/patch/a86d38af-9103-7557-9986-2dc187815569@gmail.com/) |
| إضافة A1 pin controller compatible | [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1571050492-6598-2-git-send-email-qianggui.song@amlogic.com/) |
| callback جديد للـ SoC-specific fixup | [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1572004167-24150-3-git-send-email-qianggui.song@amlogic.com/) |
| إصلاح drive strength register calculation | [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-amlogic/patch/20200610041329.12948-1-hhk7734@gmail.com/) |
| إصلاح GPIO > GPIOZ_3 في meson8b | [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-amlogic/patch/20180124002738.32338-1-martin.blumenstingl@googlemail.com/) |

---

### نقاشات الـ Mailing List

- **linux-amlogic** — القائمة الرئيسية لتطوير Amlogic SoCs:
  - [patchwork.kernel.org/project/linux-amlogic](https://patchwork.kernel.org/project/linux-amlogic/)
- **linux-arm-kernel** — للـ patches المتعلقة بـ ARM architecture:
  - [patchwork.kernel.org/project/linux-arm-kernel](https://patchwork.kernel.org/project/linux-arm-kernel/)
- **LKML archive** لأول إعلان عن الـ driver:
  - [lkml.iu.edu/hypermail/linux/kernel/1910.1/00326.html](https://lkml.iu.edu/hypermail/linux/kernel/1910.1/00326.html)

---

### مشروع Linux for Amlogic Meson

مجتمع متخصص في دعم Amlogic SoCs في الـ mainline kernel:

| المصدر | الرابط |
|--------|--------|
| الموقع الرسمي | [linux-meson.com](https://linux-meson.com/) |
| تتبع الـ mainlining progress | [linux-meson.com/mainlining.html](https://linux-meson.com/mainlining.html) |

---

### kernelnewbies.org — تغييرات الـ pinctrl عبر الإصدارات

| الإصدار | الرابط |
|---------|--------|
| Linux 6.15 — Amlogic pinctrl driver updates | [kernelnewbies.org/Linux_6.15](https://kernelnewbies.org/Linux_6.15) |
| Linux 6.11 — pinctrl improvements | [kernelnewbies.org/Linux_6.11](https://kernelnewbies.org/Linux_6.11) |
| Linux 6.8 — pinctrl new drivers | [kernelnewbies.org/Linux_6.8](https://kernelnewbies.org/Linux_6.8) |
| Linux Changes (الأرشيف الكامل) | [kernelnewbies.org/LinuxChanges](https://kernelnewbies.org/LinuxChanges) |

---

### elinux.org — موارد مفيدة

| المصدر | الرابط |
|--------|--------|
| Introduction to pin muxing and GPIO control under Linux (PDF، ELC 2021) | [elinux.org — ELC-2021 pin muxing PDF](https://elinux.org/images/a/a7/ELC-2021_Introduction_to_pin_muxing_and_GPIO_control_under_Linux.pdf) |
| Linux Kernel Resources | [elinux.org/Linux_Kernel_Resources](https://elinux.org/Linux_Kernel_Resources) |
| i2c-demux-pinctrl testing | [elinux.org/Tests:i2c-demux-pinctrl](https://www.elinux.org/Tests:i2c-demux-pinctrl) |
| Device Trees (EBC) — أمثلة pinctrl في DT | [elinux.org/EBC_Device_Trees](https://elinux.org/EBC_Device_Trees) |

---

### كتب مُوصى بها

#### Linux Device Drivers (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصل المرتبط:** Chapter 9 — Communicating with Hardware (GPIO & register access)
- **ملاحظة:** الكتاب قديم نسبياً (kernel 2.6)، لكن المفاهيم الأساسية للـ `regmap` و GPIO لا تزال صالحة
- **رابط مجاني:** [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصول المرتبطة:**
  - Chapter 17 — Devices and Modules (فهم الـ platform_device وكيف يُسجَّل الـ driver)
  - Chapter 14 — The Block I/O Layer (نفس مبدأ الـ ops structure)
- **الأهمية:** يشرح كيف تعمل الـ `platform_device` / `platform_driver` التي يعتمد عليها `meson_pinctrl_probe()`

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل المرتبط:** Chapter 15 — Embedded Linux Development (GPIO, pinmux في السياق المدمج)
- **الأهمية:** يشرح كيف يتعامل الـ SoC مع الـ pin multiplexing من منظور practical embedded systems

#### Linux Device Driver Development — John Madieu (2nd Edition, 2022)
- **الأكثر حداثة** — يغطي الـ `regmap` API والـ `pinctrl` subsystem بشكل مباشر
- **الفصل المرتبط:** Chapter 15 — Pin Control and GPIO Subsystem
- يشرح `pinctrl_desc`, `pinmux_ops`, `pinconf_ops` وهي العناصر الأساسية في `pinctrl-meson.h`

---

### مصطلحات البحث الموصى بها

للبحث عن معلومات إضافية استخدم هذه المصطلحات:

```
# البحث في الكود المصدري
git log --oneline drivers/pinctrl/meson/
git log --follow drivers/pinctrl/meson/pinctrl-meson.h

# البحث في LKML
site:lkml.org "pinctrl meson"
site:lkml.org "meson_bank" OR "meson_pmx_group"
site:patchwork.kernel.org "pinctrl: meson"

# البحث في LWN
site:lwn.net "amlogic meson" pinctrl
site:lwn.net "meson_pinctrl"

# البحث العام
"pinctrl-meson" driver explanation
"meson_pinctrl_data" struct kernel
amlogic meson gpio pinmux regmap kernel driver
"BANK_DS" macro meson pinctrl
"meson_reg_type" PULLEN PULL DIR kernel
```

---

### روابط سريعة — Kernel Source على الإنترنت

| المصدر | الرابط |
|--------|--------|
| Elixir Cross-Reference — pinctrl-meson.h | [elixir.bootlin.com/linux/latest/source/drivers/pinctrl/meson/pinctrl-meson.h](https://elixir.bootlin.com/linux/latest/source/drivers/pinctrl/meson/pinctrl-meson.h) |
| Elixir — كل SoCs تستخدم `meson_pinctrl_data` | [elixir.bootlin.com/linux/latest/ident/meson_pinctrl_data](https://elixir.bootlin.com/linux/latest/ident/meson_pinctrl_data) |
| GitHub — torvalds/linux pinctrl/meson | [github.com/torvalds/linux/tree/master/drivers/pinctrl/meson](https://github.com/torvalds/linux/tree/master/drivers/pinctrl/meson) |
## Phase 8: Writing simple module

### الفكرة

هنعمل **kprobe** على الدالة `meson_pmx_get_func_name` — دي دالة exported من الـ pinctrl-meson driver بترجع اسم الـ pinmux function بالـ selector index. كل ما حد استعلم عن function name (مثلاً من sysfs أو من الـ pinctrl core أثناء الـ probing)، الـ kprobe بتاعتنا بتتشغل وبنطبع اسم الـ function وقيمة الـ selector.

ده مفيد عشان تتابع إيه الـ pinmux functions اللي بيستعلم عنها الـ kernel أثناء الـ boot أو تغيير الـ pinctrl state.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * kprobe on meson_pmx_get_func_name
 * Traces every pinmux function-name lookup on Amlogic Meson SoCs.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit       */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe, ...    */
#include <linux/printk.h>       /* pr_info                                */
#include <linux/pinctrl/pinctrl.h> /* struct pinctrl_dev (opaque, for cast) */

/* ------------------------------------------------------------------ */
/* Pre-handler: runs right before meson_pmx_get_func_name executes.   */
/* pt_regs holds the CPU registers at the probe site.                 */
/* On ARM64: x0 = pcdev, x1 = selector                               */
/* On x86-64: di = pcdev, si = selector                              */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
#if defined(CONFIG_ARM64)
    /* ARM64 — first arg in x0, second in x1 */
    unsigned int selector = (unsigned int)regs->regs[1];
#elif defined(CONFIG_X86_64)
    /* x86-64 System V ABI — second arg in RSI */
    unsigned int selector = (unsigned int)regs->si;
#else
    unsigned int selector = 0; /* fallback for other archs */
#endif

    /*
     * بنطبع الـ selector اللي الـ pinctrl core بيطلب اسم الـ function بتاعته.
     * الاسم الفعلي بيرجع من الدالة نفسها — مش محتاجين نمسكه هنا.
     */
    pr_info("meson_pmx: get_func_name called — selector=%u\n", selector);
    return 0; /* 0 = continue normal execution */
}

/* ------------------------------------------------------------------ */
/* Post-handler: runs after the probed function returns.              */
/* regs->ax (x86) / regs->regs[0] (ARM64) holds the return value     */
/* which is the const char * function name.                           */
/* ------------------------------------------------------------------ */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
#if defined(CONFIG_ARM64)
    /* x0 يحمل الـ return value على ARM64 */
    const char *func_name = (const char *)regs->regs[0];
#elif defined(CONFIG_X86_64)
    /* rax يحمل الـ return value على x86-64 */
    const char *func_name = (const char *)regs->ax;
#else
    const char *func_name = NULL;
#endif

    /*
     * بنطبع الاسم اللي رجعته الدالة — ده هو الـ pinmux function name
     * زي "uart_a", "i2c_b", "spi_c", إلخ.
     */
    if (func_name)
        pr_info("meson_pmx: get_func_name returned — name=\"%s\"\n",
                func_name);
}

/* ------------------------------------------------------------------ */
/* kprobe struct — بيحدد الدالة اللي هنعمل عليها probe              */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name    = "meson_pmx_get_func_name", /* اسم الـ symbol بالظبط  */
    .pre_handler    = handler_pre,               /* قبل تنفيذ الدالة        */
    .post_handler   = handler_post,              /* بعد رجوع الدالة         */
};

/* ------------------------------------------------------------------ */
/* module_init: بيسجل الـ kprobe في الـ kernel                       */
/* ------------------------------------------------------------------ */
static int __init meson_pmx_probe_init(void)
{
    int ret;

    /*
     * register_kprobe بتحط breakpoint على عنوان الـ symbol.
     * لو الدالة مش موجودة (السيستم مش Meson SoC) هترجع -ENOENT.
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("meson_pmx_probe: register_kprobe failed, ret=%d\n", ret);
        pr_err("meson_pmx_probe: is this an Amlogic Meson system?\n");
        return ret;
    }

    pr_info("meson_pmx_probe: kprobe planted on %s at %p\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit: بيشيل الـ kprobe وينظف كل حاجة                      */
/* ------------------------------------------------------------------ */
static void __exit meson_pmx_probe_exit(void)
{
    /*
     * لازم unregister قبل ما الـ module يتحمل من الذاكرة —
     * لو فضل الـ kprobe مسجل والـ handler code اتحذف هيحصل kernel panic.
     */
    unregister_kprobe(&kp);
    pr_info("meson_pmx_probe: kprobe removed from %s\n", kp.symbol_name);
}

module_init(meson_pmx_probe_init);
module_exit(meson_pmx_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe tracer for meson_pmx_get_func_name — Amlogic Meson pinctrl");
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | الـ macros الأساسية زي `module_init`, `MODULE_LICENSE` |
| `linux/kprobes.h` | الـ `struct kprobe` وكل API الخاص بالـ kprobes |
| `linux/printk.h` | الـ `pr_info` و`pr_err` |
| `linux/pinctrl/pinctrl.h` | عشان `struct pinctrl_dev` يكون معروف للـ compiler |

---

#### الـ `handler_pre`

**الـ pre-handler** بيتشغل قبل ما الدالة الأصلية تأخذ التنفيذ. بياخد الـ `selector` من الـ registers مباشرةً عشان يطبعه — وده مفيد لأن في الوقت ده الـ name لسه مرجعتش.

الفرق بين ARM64 وx86-64 في أرقام الـ registers بيتحل عن طريق `#ifdef` — عشان الكود يشتغل على النظامين.

---

#### الـ `handler_post`

**الـ post-handler** بيتشغل بعد رجوع الدالة، والـ return value (الـ `const char *` اسم الـ function) بيكون موجود في الـ register المناسب. بنطبعه لو مش NULL — وده بيكمّل الصورة: علمنا إيه الـ selector وإيه الاسم اللي اتبعتله.

---

#### الـ `struct kprobe`

الـ `symbol_name` بيخلي الـ kernel يبحث عن العنوان تلقائياً من الـ kallsyms بدل ما نحدده manually — أأمن وأسهل.

---

#### الـ `module_init` و`module_exit`

- الـ `init` بيسجل الـ kprobe وبيطبع عنوانه كتأكيد إنه اتحط صح.
- الـ `exit` بيشيله **قبل** تحرير الـ module من الذاكرة — لو متعملتش unregister والـ kernel حاول يكمّل تنفيذ الـ handler بعد تحرير الـ module memory هيحصل **use-after-free** وكراش مضمون.

---

### الـ Makefile وطريقة البناء

```makefile
# Makefile — ضيفه في نفس مجلد الـ .c file
obj-m += meson_pmx_kprobe.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

```bash
# بناء وتحميل الـ module
make
sudo insmod meson_pmx_kprobe.ko

# تابع الـ output
sudo dmesg -w | grep meson_pmx

# تفعيل استعلام عن الـ functions من sysfs لتوليد events
cat /sys/kernel/debug/pinctrl/*/pinmux-functions 2>/dev/null

# تحميل خارج
sudo rmmod meson_pmx_kprobe
```

---

### مثال على الـ output المتوقع

```
[  45.123] meson_pmx_probe: kprobe planted on meson_pmx_get_func_name at ffffffc0108a4d20
[  45.201] meson_pmx: get_func_name called — selector=0
[  45.201] meson_pmx: get_func_name returned — name="gpio_periphs"
[  45.202] meson_pmx: get_func_name called — selector=1
[  45.202] meson_pmx: get_func_name returned — name="uart_a"
[  45.203] meson_pmx: get_func_name called — selector=2
[  45.203] meson_pmx: get_func_name returned — name="i2c_b"
```

كل سطر بيوضح إن الـ pinctrl core بيمشي على الـ functions واحدة واحدة عشان يبني الـ internal tables — وده اللي بيحصل عادةً أثناء `meson_pinctrl_probe`.
