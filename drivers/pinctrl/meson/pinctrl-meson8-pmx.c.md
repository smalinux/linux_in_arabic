## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ Subsystem ده؟

الملفان `pinctrl-meson8-pmx.c` و `pinctrl-meson8-pmx.h` جزء من **ARM/Amlogic Meson SoC support** subsystem في الـ Linux kernel، تحت مسار `drivers/pinctrl/meson/`. المشرفون عليه مذكورون في `MAINTAINERS` تحت اسم **ARM/Amlogic Meson SoC support**، وبيشمل كل حاجة في `drivers/pinctrl/meson/`.

---

### القصة — تخيل معايا

تخيل إن عندك سلك واحد بس في البيت، وعايز توصله في أكتر من جهاز: مرة تليفون، ومرة سماعات، ومرة شاحن. السلك ده واحد بس مش ممكن يتوصل في كل حاجة في نفس الوقت. فبتحتاج **كبسة (مفتاح/switch)** تقدر بيها تقول "دلوقتي السلك ده يشتغل تليفون" أو "دلوقتي يشتغل سماعات".

الـ **SoC** (System on Chip) — زي Amlogic Meson8 اللي بيتلاقى في صناديق Android التلفزيونية القديمة — عنده مئات الأطراف (pins) خارجة منه. كل pin ممكن يشتغل بأكتر من طريقة:
- ممكن يكون GPIO عادي (بتضبطه إنت).
- ممكن يكون UART TX (للتواصل مع جهاز تاني).
- ممكن يكون SPI MOSI (لنقل بيانات سريع).
- ممكن يكون I2C SDA (للاتصال مع حساسات).

المشكلة: الـ hardware مش بيعرف إنت عايزه يعمل إيه. لازم يكون في كود في الـ kernel بيقوله "الـ pin ده دلوقتي هو UART مش GPIO". ده بالظبط هو شغل الـ **pinmux** (pin multiplexer).

---

### الـ Meson8 بالذات — ليه له كود خاص؟

شركة Amlogic عملت جيلين من الـ SoCs:

- **الجيل الأول** (Meson8، Meson8b، Meson8m2): بيتحكم في الـ pinmux بطريقة بسيطة جداً — **كل group عنده bit واحد في register**، لو الـ bit = 1 الـ function اتفعّل، لو = 0 الـ pin رجع GPIO.
- **الجيل الثاني** (AXG، G12A، S4، A4...): بيستخدم طريقة أكثر تعقيداً لتحديد الـ function (بيخزن رقم الـ function مش مجرد enable/disable).

الملفان دول بيتعاملوا مع **الجيل الأول بس**.

---

### إيه اللي بيعمله الكود ده بالظبط؟

#### الـ Header — `pinctrl-meson8-pmx.h`

بيعرّف:

```c
struct meson8_pmx_data {
    bool is_gpio;      /* هل الـ group ده GPIO خالص؟ */
    unsigned int reg;  /* رقم الـ register في الـ regmap */
    unsigned int bit;  /* رقم الـ bit جوه الـ register */
};
```

ده الـ "عنوان" لكل function group في الـ hardware. لو عايز تفعّل UART على مجموعة pins معينة، بتروح للـ register رقم X، وبتحط 1 في الـ bit رقم Y.

كمان بيوفر macro مهمين جداً:

```c
/* GROUP — لـ function group عادي (UART, SPI, إلخ) */
#define GROUP(grp, r, b) { .name = #grp, .pins = grp##_pins, ... PMX_DATA(r, b, false) }

/* GPIO_GROUP — لـ pin بيشتغل GPIO خالص */
#define GPIO_GROUP(gpio)  { .name = #gpio, .pins = (unsigned int[]){ gpio }, ... PMX_DATA(0, 0, true) }
```

الـ drivers زي `pinctrl-meson8.c` و `pinctrl-meson8b.c` بيستخدموا الـ macros دي عشان يبنوا جداول الـ groups بشكل مختصر.

#### الـ Source — `pinctrl-meson8-pmx.c`

فيه **3 وظائف أساسية** + **1 struct تُصدَّر**:

**1. `meson8_pmx_disable_other_groups()`**

لما عايز تفعّل function معين على pin، الـ pin ده ممكن يكون مشترك في أكتر من group. لازم تعطّل كل الـ groups التانية الأول. الدالة دي بتمشي على كل الـ groups، ولو لاقت group بيستخدم الـ pin المطلوب ومش هو الـ group المختار، بتعمل:

```c
regmap_update_bits(pc->reg_mux, pmx_data->reg * 4, BIT(pmx_data->bit), 0);
/* بتكتب 0 في الـ bit المخصص للـ group → يتعطل */
```

**2. `meson8_pmx_set_mux()`**

ده القلب. لما الـ kernel يقرر "عايز الـ UART يشتغل على الـ pins دي":
1. بيعطّل كل الـ groups التانية اللي بتستخدم نفس الـ pins (بيستدعي الدالة السابقة).
2. لو الـ function مش GPIO (func_num != 0)، بيكتب 1 في الـ bit المخصوص في الـ register:

```c
regmap_update_bits(pc->reg_mux, pmx_data->reg * 4,
                   BIT(pmx_data->bit), BIT(pmx_data->bit));
/* يفعّل الـ function المطلوب */
```

**3. `meson8_pmx_request_gpio()`**

لما حد يطلب pin كـ GPIO، بيعطّل كل الـ functions التانية عليه (بيستدعي `disable_other_groups` بـ `sel_group = -1`):

```c
meson8_pmx_disable_other_groups(pc, offset, -1);
/* -1 = مفيش group محدد = عطّل الكل */
```

**4. `meson8_pmx_ops`**

الـ struct اللي بيجمع الوظائف دي كلها في شكل operations table يقدر الـ pinctrl core يستخدمه:

```c
const struct pinmux_ops meson8_pmx_ops = {
    .set_mux             = meson8_pmx_set_mux,
    .get_functions_count = meson_pmx_get_funcs_count,   /* من pinctrl-meson.c */
    .get_function_name   = meson_pmx_get_func_name,     /* من pinctrl-meson.c */
    .get_function_groups = meson_pmx_get_groups,        /* من pinctrl-meson.c */
    .gpio_request_enable = meson8_pmx_request_gpio,
};
```

---

### الفرق بين الجيل الأول والتاني

| الميزة | Meson8 (الجيل الأول) | AXG/G12A (الجيل التاني) |
|---|---|---|
| طريقة التحكم | bit واحد per group (enable/disable) | field متعدد البتات (function selector) |
| الـ PMX file | `pinctrl-meson8-pmx.c` | `pinctrl-meson-axg-pmx.c` |
| التعقيد | بسيط جداً | أعقد قليلاً |
| الـ ops struct | `meson8_pmx_ops` | `meson_axg_pmx_ops` |

---

### رسم توضيحي للعلاقة بين الملفات

```
Linux pinctrl core
        │
        ▼
 pinctrl-meson.c         ← probe + GPIO control + pull config (مشترك لكل الأجيال)
        │
        ├── pinctrl-meson8-pmx.c  ← منطق الـ mux للجيل الأول (Meson8/8b)
        │         uses ──► pinctrl-meson8-pmx.h  ← struct + macros
        │
        ├── pinctrl-meson-axg-pmx.c ← منطق الـ mux للجيل الثاني (AXG+)
        │
        ├── pinctrl-meson8.c   ← جداول الـ pins/groups/funcs لشريحة Meson8
        ├── pinctrl-meson8b.c  ← جداول الـ pins/groups/funcs لشريحة Meson8b
        ├── pinctrl-meson-gxbb.c  ← جداول لـ GXbb
        ├── pinctrl-meson-gxl.c   ← جداول لـ GXl
        └── pinctrl-meson-g12a.c  ← جداول لـ G12A
```

---

### الملفات المكوّنة للـ Subsystem

| الملف | الدور |
|---|---|
| `pinctrl-meson.c` | الـ core المشترك: probe، GPIO، pull، pinconf |
| `pinctrl-meson.h` | الـ structs المشتركة: `meson_pinctrl`، `meson_bank`، `meson_pmx_group` |
| `pinctrl-meson8-pmx.c` | **منطق الـ mux للجيل الأول** (هذا الملف) |
| `pinctrl-meson8-pmx.h` | الـ struct `meson8_pmx_data` والـ macros `GROUP`، `GPIO_GROUP` |
| `pinctrl-meson8.c` | جدول الـ pins والـ groups والـ functions لشريحة Meson8/Meson8m2 |
| `pinctrl-meson8b.c` | نفس الفكرة لشريحة Meson8b |
| `pinctrl-meson-axg-pmx.c` / `.h` | نظير الجيل الثاني من الـ mux |
| `pinctrl-meson-g12a.c` | جداول شريحة G12A (الأحدث من الجيل الثاني) |
| `include/linux/pinctrl/pinmux.h` | تعريف `struct pinmux_ops` اللي بيتم تنفيذه هنا |
| `include/linux/pinctrl/pinctrl.h` | الـ pinctrl framework الأساسي |
| `include/linux/regmap.h` | واجهة الوصول للـ registers عبر `regmap_update_bits` |
## Phase 2: شرح الـ Pinctrl / Pinmux Framework

---

### المشكلة اللي الـ Subsystem ده بيحلها

أي SoC حديث — زي Amlogic Meson8 — فيه مئات الـ pins الجسدية. كل pin ممكن يشتغل في أكتر من وظيفة واحدة:

- نفس الـ pin ممكن يكون UART TX أو SPI MOSI أو GPIO عادي.
- الـ hardware بيتحكم في ده عن طريق **mux registers** — كل bit في register بيختار function معينة.

من غير subsystem موحد، كل driver كان لازم يعرف عناوين الـ registers دي ويكتبها بنفسه. النتيجة:

1. **تعارض**: درايفرين ممكن يحاولوا يعملوا mux لنفس الـ pin لفانكشنز مختلفة.
2. **تكرار كود**: كل درايفر بيعيد كتابة نفس منطق الـ register access.
3. **صعوبة الـ board bring-up**: مفيش مكان مركزي يعرف "مين شايل الـ pin ده دلوقتي".

---

### الحل اللي الـ Kernel بياخده

الـ kernel عنده **pinctrl subsystem** — طبقة وسطى بين الـ hardware drivers وبين الـ silicon registers. الـ subsystem ده بيوفر:

1. **تسجيل مركزي** لكل الـ pins والـ groups والـ functions.
2. **arbitration**: لو درايفرين طلبوا نفس الـ pin، الـ core يرفض واحد.
3. **واجهة موحدة**: أي درايفر يطلب function عن طريق `pinctrl_select_state()` بدون ما يعرف أي حاجة عن الـ registers.

الـ subsystem اتقسم لجزأين:
- **pinctrl core**: الـ generic logic، الـ arbitration، الـ DT parsing.
- **pin controller driver**: الكود الخاص بكل SoC — زي `pinctrl-meson8-pmx.c` — اللي بيعرف exactly أي bit في أي register بيعمل إيه.

---

### تشبيه الواقع (مع mapping كامل)

تخيل **لوحة توزيع كهربائي** في مبنى كبير:

| العنصر في التشبيه | المقابل في الـ Kernel |
|---|---|
| المبنى بكل أدواره | الـ SoC |
| الأسلاك الفيزيائية | الـ pins |
| مفاتيح التحويل في اللوحة | الـ mux register bits |
| إدارة المبنى (المشرف) | الـ pinctrl core |
| المستأجر اللي بيطلب كهرباء | الـ peripheral driver (UART, SPI, ...) |
| سجل المستأجرين عند المشرف | الـ pin ownership tracking في الـ core |
| دليل اللوحة (أي مفتاح بيوصل لأي حجرة) | الـ `meson8_pmx_data` (reg + bit) |
| إجراءات التشغيل للمشرف | الـ `pinmux_ops` vtable |

لما مستأجر (درايفر) عايز يشغّل حجرة (function)، بيقول للمشرف (core). المشرف يشوف السجل (ownership)، لو الخط متاح يقول للكهربائي (pin controller driver) يقلّب المفتاح الصح (يكتب في الـ register).

---

### الـ Big Picture Architecture

```
  ┌─────────────────────────────────────────────────────────┐
  │                  Consumer Drivers                        │
  │     (serial8250, spi-meson, i2c-meson, gpio-keys ...)   │
  └──────────────────────┬──────────────────────────────────┘
                         │  pinctrl_select_state()
                         ▼
  ┌─────────────────────────────────────────────────────────┐
  │               pinctrl CORE  (kernel/pinctrl/)           │
  │                                                          │
  │  - Pin ownership tracking (mux_owner / gpio_owner)      │
  │  - DT parsing  (pinctrl-bindings)                        │
  │  - State machine  (init / default / sleep / idle)        │
  │  - Conflict detection & arbitration                      │
  └───────────┬─────────────────────┬────────────────────────┘
              │ pinctrl_ops         │ pinmux_ops
              ▼                     ▼
  ┌───────────────────┐   ┌────────────────────────────────┐
  │  meson_pinctrl    │   │   meson8_pmx_ops               │
  │  (pinctrl-meson.c)│   │   (pinctrl-meson8-pmx.c)       │
  │                   │   │                                 │
  │  - groups/funcs   │   │  .set_mux()                    │
  │  - bank config    │   │  .gpio_request_enable()         │
  │  - DT extra parse │   │  .get_functions_count()         │
  └───────────────────┘   └──────────────┬─────────────────┘
                                         │ regmap_update_bits()
                                         ▼
  ┌─────────────────────────────────────────────────────────┐
  │              Hardware: Meson8 MUX Registers              │
  │                                                          │
  │   reg_mux regmap                                         │
  │   ┌────────┬────────┬────────┬────────┐                  │
  │   │ REG 0  │ REG 1  │ REG 2  │  ...   │                  │
  │   │[31..0] │[31..0] │[31..0] │        │                  │
  │   └────────┴────────┴────────┴────────┘                  │
  │    Each bit = enable one pinmux group                    │
  └─────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction: الثلاثي (Pin, Group, Function)

ده أهم مفهوم في الـ subsystem كله. الـ pinctrl framework بيبني نموذجه على ثلاث مستويات:

```
  Pin (الوحدة الصغرى)
   └── Group (مجموعة pins بتشتغل مع بعض كـ unit)
        └── Function (الـ peripheral اللي بيستخدم الـ group)
```

#### مثال حقيقي على Meson8:

```
Function: "uart_a"
  └── Group: "uart_a_tx_rx"
        └── Pins: [pin_45, pin_46]
              ├── pin_45 → TX
              └── pin_46 → RX

Function: "gpio"
  └── Group: "GPIOX_5"   (GPIO_GROUP macro)
        └── Pins: [pin_45]   (single pin, is_gpio = true)
```

---

### شرح الـ Structs وعلاقتها ببعض

```
struct meson_pinctrl_data
  ├── pins[]       ──► struct pinctrl_pin_desc[]   (رقم + اسم كل pin)
  ├── groups[]     ──► struct meson_pmx_group[]
  │                      ├── name
  │                      ├── pins[]    (أرقام الـ pins في الـ group)
  │                      └── data ──► struct meson8_pmx_data
  │                                      ├── reg   (رقم الـ register * 4 = byte offset)
  │                                      ├── bit   (رقم الـ bit في الـ register)
  │                                      └── is_gpio (هل ده GPIO group؟)
  ├── funcs[]      ──► struct meson_pmx_func[]
  │                      ├── name
  │                      └── groups[]  (أسماء الـ groups اللي الـ function دي بتشملها)
  ├── banks[]      ──► struct meson_bank[]
  │                      ├── first/last  (نطاق الـ pins)
  │                      └── regs[]  ──► struct meson_reg_desc (pull, dir, out, in ...)
  └── pmx_ops ──► &meson8_pmx_ops   (الـ vtable بتاع الـ mux)


struct meson_pinctrl   (الـ runtime instance)
  ├── pcdev   ──► struct pinctrl_dev   (الـ handle اللي الـ core بيديه)
  ├── desc    ──► struct pinctrl_desc  (الـ registration descriptor)
  ├── data    ──► struct meson_pinctrl_data  (الـ static SoC data)
  ├── reg_mux ──► struct regmap   (الـ mux registers)
  ├── reg_pull / reg_pullen / reg_gpio / reg_ds  ──► regmaps تانية
  └── chip    ──► struct gpio_chip  (لما الـ pins بتشتغل كـ GPIO)
```

---

### الـ Meson8 PMX Design: الـ One-Bit-Per-Group Model

الـ Meson8 بيستخدم نموذج بسيط جداً:

- كل **group** عنده bit واحدة في الـ mux registers.
- لو الـ bit = 1 → الـ group enabled (الـ peripheral شايل الـ pin).
- لو الـ bit = 0 → الـ group disabled.
- لو **كل** الـ groups اللي بتشمل pin معين = 0 → الـ pin بيتحول أوتوماتيك لـ GPIO mode.

ده عكس الـ modern chips اللي بتستخدم **multi-bit field** لاختيار من عدة functions (زي BCM2835 عندها 3 bits لكل pin).

```
  MUX_REG[N]
  ┌────┬────┬────┬────┬────┬────┐
  │ b7 │ b6 │ b5 │ b4 │ b3 │ b2 │ ...
  └──┬─┴──┬─┴────┴──┬─┴────┴────┘
     │    │         │
     │    │         └─ uart_a_tx_rx group enabled
     │    └─ i2c_b group enabled
     └─ spi_a_mosi group enabled
```

الـ constraint الأساسي: **pin واحد مش ممكن يكون في أكتر من group enabled في نفس الوقت** — ده اللي بيعمله `meson8_pmx_disable_other_groups()`.

---

### شرح الكود خطوة بخطوة

#### الـ Macros في الـ Header

```c
/* بيبني meson_pmx_group كاملة من اسم الـ group ورقم الـ register والـ bit */
#define GROUP(grp, r, b)
    {
        .name = #grp,
        .pins = grp ## _pins,          /* array معرّف في مكان تاني */
        .num_pins = ARRAY_SIZE(...),
        .data = (const struct meson8_pmx_data[]){
            PMX_DATA(r, b, false),     /* is_gpio = false */
        },
    }

/* GPIO group: pin واحد، is_gpio = true، الـ reg/bit مش مهمين */
#define GPIO_GROUP(gpio)
    {
        .name = #gpio,
        .pins = (const unsigned int[]){ gpio },
        .num_pins = 1,
        .data = (const struct meson8_pmx_data[]){
            PMX_DATA(0, 0, true),      /* is_gpio flag يقول للـ driver يتجاهله */
        },
    }
```

#### `meson8_pmx_disable_other_groups()`

```c
static void meson8_pmx_disable_other_groups(struct meson_pinctrl *pc,
                                            unsigned int pin, int sel_group)
{
    /* بيلف على كل الـ groups المعرّفة في الـ SoC */
    for (i = 0; i < pc->data->num_groups; i++) {
        group = &pc->data->groups[i];
        pmx_data = (struct meson8_pmx_data *)group->data;

        /* GPIO groups مش بيحتاج نعطّلها — و الـ selected group كمان */
        if (pmx_data->is_gpio || i == sel_group)
            continue;

        /* لو الـ group دي بتشمل الـ pin اللي بنتكلم عليه */
        for (j = 0; j < group->num_pins; j++) {
            if (group->pins[j] == pin) {
                /* اكتب 0 في الـ bit الخاص بالـ group دي */
                regmap_update_bits(pc->reg_mux,
                                   pmx_data->reg * 4,  /* byte offset */
                                   BIT(pmx_data->bit), /* mask */
                                   0);                 /* value = disable */
            }
        }
    }
}
```

**لماذا `reg * 4`؟** لأن الـ registers بتتعنون بـ word index (0, 1, 2, ...) لكن الـ regmap API بتاخد byte offset، فلازم نضرب في 4.

#### `meson8_pmx_set_mux()` — قلب الـ Driver

```c
static int meson8_pmx_set_mux(struct pinctrl_dev *pcdev,
                               unsigned func_num, unsigned group_num)
{
    /* الـ core بعت لنا index الـ function والـ group */
    func  = &pc->data->funcs[func_num];
    group = &pc->data->groups[group_num];
    pmx_data = (struct meson8_pmx_data *)group->data;

    /* الخطوة 1: عطّل كل الـ groups التانية اللي بتشارك أي pin من الـ group دي */
    for (i = 0; i < group->num_pins; i++)
        meson8_pmx_disable_other_groups(pc, group->pins[i], group_num);

    /* الخطوة 2: لو مش GPIO (func_num != 0)، فعّل الـ group */
    if (func_num)
        regmap_update_bits(pc->reg_mux,
                           pmx_data->reg * 4,
                           BIT(pmx_data->bit),
                           BIT(pmx_data->bit));  /* set the bit */
    return ret;
}
```

**لماذا func_num == 0 معناه GPIO؟** بالـ convention في الـ Meson driver، الـ function رقم 0 هي `gpio` — وبما إن الـ GPIO mode بيتفعل تلقائياً لما كل الـ bits = 0، مش محتاج نكتب أي حاجة.

#### `meson8_pmx_request_gpio()` — طلب GPIO مباشر

```c
static int meson8_pmx_request_gpio(struct pinctrl_dev *pcdev,
                                   struct pinctrl_gpio_range *range,
                                   unsigned offset)
{
    /* عطّل كل الـ groups اللي بتشمل الـ pin ده
       sel_group = -1 يعني: عطّل الكل بدون استثناء */
    meson8_pmx_disable_other_groups(pc, offset, -1);
    return 0;
}
```

---

### الـ vtable: `meson8_pmx_ops`

```c
const struct pinmux_ops meson8_pmx_ops = {
    /* الـ core بيسأل: "كام function عندك؟" */
    .get_functions_count = meson_pmx_get_funcs_count,

    /* الـ core بيسأل: "اسم الـ function رقم N إيه؟" */
    .get_function_name   = meson_pmx_get_func_name,

    /* الـ core بيسأل: "الـ function دي بتشمل أنهي groups؟" */
    .get_function_groups = meson_pmx_get_groups,

    /* الـ core بيقول: "فعّل الـ function دي على الـ group ده" */
    .set_mux             = meson8_pmx_set_mux,

    /* الـ GPIO subsystem بيطلب الـ pin ده كـ GPIO */
    .gpio_request_enable = meson8_pmx_request_gpio,
};
```

الفانكشنز الـ 3 الأوائل (`get_functions_count`, `get_function_name`, `get_function_groups`) متعرّفة في `pinctrl-meson.c` — مشتركة بين كل أجيال الـ Meson — لأن الـ data structures نفسها (الـ `funcs[]` array) واحدة.

---

### مين بيعمل إيه: الـ Core vs الـ Driver

| المسؤولية | الـ pinctrl Core | الـ Meson8 Driver |
|---|---|---|
| تتبع مين شايل الـ pin | نعم (mux_owner) | لا |
| الـ DT parsing | جزء منه | `parse_dt` callback |
| كتابة الـ MUX registers | لا | نعم (`regmap_update_bits`) |
| Conflict detection | نعم | لا |
| تعريف الـ pins/groups/funcs | لا (بس بيخزنها) | نعم (static arrays) |
| GPIO ↔ Peripheral switching | orchestration فقط | التنفيذ الفعلي |
| Pull-up / Pull-down config | callback فقط | التنفيذ في `pinconf_ops` |

---

### مفاهيم من Subsystems تانية لازم تعرفها

- **regmap**: طبقة abstraction فوق `ioremap` — بتوفر read/write للـ MMIO registers مع caching واختياري locking. الـ `regmap_update_bits(map, offset, mask, val)` بتعمل read-modify-write atomic.
- **GPIO subsystem**: الـ `struct gpio_chip` اللي جوا `meson_pinctrl` — بتسمح للـ pins تشتغل كـ GPIO عادي لما الـ mux function تتحول.
- **Device Tree**: الـ pinctrl states (`default`, `sleep`) بتتعرّف في الـ DT وبيعملها parse الـ core ويحوّلها لطلبات `set_mux`.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. Cheatsheet: الـ Macros، الـ Enums، والـ Config Options

#### الـ Enums

| Enum | القيم | الاستخدام |
|------|-------|-----------|
| `meson_reg_type` | `MESON_REG_PULLEN`, `MESON_REG_PULL`, `MESON_REG_DIR`, `MESON_REG_OUT`, `MESON_REG_IN`, `MESON_REG_DS`, `MESON_NUM_REG` | index في array الـ `regs[]` داخل `meson_bank` |
| `meson_pinconf_drv` | `MESON_PINCONF_DRV_500UA`, `MESON_PINCONF_DRV_2500UA`, `MESON_PINCONF_DRV_3000UA`, `MESON_PINCONF_DRV_4000UA` | قيم drive-strength المدعومة بالـ microamps |

#### الـ Macros (تعريف الـ static data)

| Macro | المعاملات | ما ينتجه |
|-------|-----------|---------|
| `PMX_DATA(r, b, g)` | reg, bit, is_gpio | `meson8_pmx_data` struct literal |
| `GROUP(grp, r, b)` | group name, reg, bit | `meson_pmx_group` مع `PMX_DATA` مدمج |
| `GPIO_GROUP(gpio)` | pin name | `meson_pmx_group` خاص بـ GPIO، `is_gpio=true`، reg=0، bit=0 |
| `FUNCTION(fn)` | function name | `meson_pmx_func` يربط الـ function بالـ groups array |
| `BANK(n,f,l,fi,li,...)` | name, first, last, irq range, reg/bit لكل وظيفة | `meson_bank` بدون DS (يستدعي `BANK_DS` بـ dsr=0، dsb=0) |
| `BANK_DS(n,f,l,fi,li,...)` | نفس BANK + dsr, dsb | `meson_bank` كامل مع drive-strength |
| `MESON_PIN(x)` | pin number | `PINCTRL_PIN(x, #x)` — يحوّل رقم الـ pin إلى descriptor |

#### الـ Register Access Pattern

| العملية | الـ regmap المستخدم | الصيغة |
|---------|-------------------|--------|
| تفعيل/تعطيل mux | `pc->reg_mux` | `reg * 4` (byte offset)، `BIT(bit)` |
| pull enable | `pc->reg_pullen` | عبر `meson_reg_desc` |
| pull up/down | `pc->reg_pull` | عبر `meson_reg_desc` |
| GPIO direction | `pc->reg_gpio` | عبر `meson_reg_desc` |
| drive strength | `pc->reg_ds` | عبر `meson_reg_desc` |

---

### 1. الـ Structs المهمة

#### `meson8_pmx_data` — (pinctrl-meson8-pmx.h)

**الغرض:** يحمل إحداثيات الـ bit الذي يتحكم في تفعيل مجموعة pinmux محددة داخل الـ mux register.

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `is_gpio` | `bool` | إذا كان `true` فالـ group هذا هو GPIO mode، مش محتاج تسجيل في الـ register |
| `reg` | `unsigned int` | رقم الـ register (يُضرب في 4 للحصول على byte offset) |
| `bit` | `unsigned int` | رقم الـ bit داخل الـ register اللي يفعّل/يعطّل الـ group |

**الارتباط:** يُخزَّن كـ `const void *data` داخل `meson_pmx_group`، ويُعمل له cast يدوي وقت الاستخدام.

---

#### `meson_pmx_group` — (pinctrl-meson.h)

**الغرض:** يمثّل مجموعة pins بتعمل مع بعض لتفعيل وظيفة معينة (مثلاً: UART TX/RX pins).

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `name` | `const char *` | اسم الـ group (مثلاً `"uart_a_tx"`) |
| `pins` | `const unsigned int *` | array بأرقام الـ pins في الـ group |
| `num_pins` | `unsigned int` | عدد الـ pins |
| `data` | `const void *` | pointer لـ `meson8_pmx_data` (أو نوع آخر في جيل تاني) |

---

#### `meson_pmx_func` — (pinctrl-meson.h)

**الغرض:** يمثّل وظيفة hardware كاملة (مثل UART A) وبيجمع كل الـ groups المرتبطة بيها.

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `name` | `const char *` | اسم الـ function (مثلاً `"uart_a"`) |
| `groups` | `const char * const *` | array بأسماء الـ groups المنتمية لهذه الـ function |
| `num_groups` | `unsigned int` | عدد الـ groups |

---

#### `meson_reg_desc` — (pinctrl-meson.h)

**الغرض:** يصف موقع bit واحد في الـ regmap لأي وظيفة (pull، direction، إلخ).

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `reg` | `unsigned int` | offset الـ register في الـ regmap |
| `bit` | `unsigned int` | رقم الـ bit |

---

#### `meson_bank` — (pinctrl-meson.h)

**الغرض:** يمثّل بنك GPIO — مجموعة pins متجاورة بتشارك نفس set من الـ registers للتحكم فيها.

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `name` | `const char *` | اسم البنك (مثلاً `"GPIOX"`) |
| `first` | `unsigned int` | أول pin number في البنك |
| `last` | `unsigned int` | آخر pin number في البنك |
| `irq_first` | `int` | أول hwirq رقمه في البنك |
| `irq_last` | `int` | آخر hwirq |
| `regs[MESON_NUM_REG]` | `meson_reg_desc[]` | array من 6 descriptors: pullen، pull، dir، out، in، ds |

---

#### `meson_pinctrl_data` — (pinctrl-meson.h)

**الغرض:** الـ static read-only data الخاصة بكل SoC — نوع من الـ "board file" للـ pinctrl. بتتعرّف مرة واحدة لكل SoC وبتشترك بينها.

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `name` | `const char *` | اسم الـ controller |
| `pins` | `const struct pinctrl_pin_desc *` | array بكل الـ pins |
| `groups` | `const struct meson_pmx_group *` | array بكل الـ groups |
| `funcs` | `const struct meson_pmx_func *` | array بكل الـ functions |
| `num_pins/groups/funcs` | `unsigned int` | أحجام الـ arrays |
| `banks` | `const struct meson_bank *` | array بكل الـ GPIO banks |
| `num_banks` | `unsigned int` | عدد البنوك |
| `pmx_ops` | `const struct pinmux_ops *` | pointer لـ ops الـ mux (في Meson8 = `meson8_pmx_ops`) |
| `pmx_data` | `const void *` | بيانات إضافية اختيارية للـ pmx layer |
| `parse_dt` | function pointer | دالة خاصة بكل SoC لتحليل device tree |

---

#### `meson_pinctrl` — (pinctrl-meson.h)

**الغرض:** الـ runtime instance للـ driver — بيتخلق مرة واحدة per device وبيحمل كل state الـ controller.

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `dev` | `struct device *` | الـ device المرتبط |
| `pcdev` | `struct pinctrl_dev *` | handle من الـ pinctrl core |
| `desc` | `struct pinctrl_desc` | الـ descriptor اللي بيتسجل مع الـ core |
| `data` | `struct meson_pinctrl_data *` | الـ static SoC-specific data |
| `reg_mux` | `struct regmap *` | regmap لـ registers التحكم في الـ mux |
| `reg_pullen` | `struct regmap *` | regmap لـ pull-enable |
| `reg_pull` | `struct regmap *` | regmap لـ pull up/down |
| `reg_gpio` | `struct regmap *` | regmap لـ GPIO direction/value |
| `reg_ds` | `struct regmap *` | regmap لـ drive strength |
| `chip` | `struct gpio_chip` | الـ GPIO chip المدمج |
| `fwnode` | `struct fwnode_handle *` | node الـ device tree |

---

### 2. مخططات علاقات الـ Structs (ASCII)

```
┌─────────────────────────────────────────────────────────────┐
│                    meson_pinctrl  (runtime)                  │
│                                                              │
│  dev ──────────────→ struct device                          │
│  pcdev ────────────→ struct pinctrl_dev  (pinctrl core)     │
│  chip ─────────────→ struct gpio_chip   (GPIO subsystem)    │
│  fwnode ───────────→ struct fwnode_handle (DT node)         │
│                                                              │
│  reg_mux ──────────→ struct regmap  [MUX registers]         │
│  reg_pullen ───────→ struct regmap  [Pull-enable regs]      │
│  reg_pull ─────────→ struct regmap  [Pull regs]             │
│  reg_gpio ─────────→ struct regmap  [GPIO regs]             │
│  reg_ds ───────────→ struct regmap  [Drive-strength regs]   │
│                                                              │
│  data ─────────────→ meson_pinctrl_data (static, per-SoC)  │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────┐
│                   meson_pinctrl_data  (static)               │
│                                                              │
│  pins[] ──────────→ pinctrl_pin_desc[]   (pin numbers/names)│
│  groups[] ─────────→ meson_pmx_group[]                      │
│  funcs[] ──────────→ meson_pmx_func[]                       │
│  banks[] ──────────→ meson_bank[]                           │
│  pmx_ops ──────────→ pinmux_ops  ← meson8_pmx_ops           │
│  parse_dt ─────────→ fn pointer                             │
└───────────┬─────────────────┬────────────────────────────────┘
            │                 │
            ▼                 ▼
┌───────────────────┐  ┌──────────────────────┐
│  meson_pmx_group  │  │   meson_pmx_func     │
│                   │  │                      │
│  name             │  │  name                │
│  pins[]           │  │  groups[] ──────────→│ (أسماء string)
│  num_pins         │  │  num_groups          │
│  data ────────────│→ │                      │
│  (const void *)   │  └──────────────────────┘
└─────────┬─────────┘
          │ cast to
          ▼
┌──────────────────────┐
│   meson8_pmx_data    │
│                      │
│  is_gpio (bool)      │
│  reg  (u32)          │
│  bit  (u32)          │
└──────────────────────┘

┌─────────────────────────────────────────┐
│            meson_bank                   │
│                                         │
│  name, first, last                      │
│  irq_first, irq_last                    │
│  regs[6]: meson_reg_desc[]              │
│    [PULLEN] → { reg, bit }              │
│    [PULL]   → { reg, bit }              │
│    [DIR]    → { reg, bit }              │
│    [OUT]    → { reg, bit }              │
│    [IN]     → { reg, bit }              │
│    [DS]     → { reg, bit }              │
└─────────────────────────────────────────┘
```

---

### 3. مخطط دورة الحياة (Lifecycle)

```
=== INIT PHASE ===

platform_device probe
        │
        ▼
meson_pinctrl_probe()
        │
        ├─→ allocate meson_pinctrl (devm_kzalloc)
        │
        ├─→ load meson_pinctrl_data from of_device_id
        │
        ├─→ init regmaps (reg_mux, reg_pullen, reg_pull, reg_gpio, reg_ds)
        │
        ├─→ call pc->data->parse_dt(pc)   ← SoC-specific DT parsing
        │
        ├─→ pinctrl_register() → returns pcdev
        │       └─ uses pc->desc which contains:
        │              ├─ .pins  = pc->data->pins
        │              ├─ .npins = pc->data->num_pins
        │              └─ .pmxops = pc->data->pmx_ops  ← meson8_pmx_ops
        │
        └─→ gpiochip_add_data() → registers GPIO chip

=== RUNTIME PHASE ===

   [see call flow diagrams below]

=== TEARDOWN PHASE ===

platform_device remove
        │
        ├─→ gpiochip_remove()
        └─→ pinctrl_unregister()
            └─ devm_* resources freed automatically
```

---

### 4. مخططات تدفق الاستدعاء (Call Flow)

#### 4.1 تفعيل وظيفة Pinmux (مثلاً UART)

```
consumer driver calls:
  pinctrl_select_state(pctldev, state)
        │
        ▼
  pinctrl core validates function + group
        │
        ▼
  meson8_pmx_ops.set_mux(pcdev, func_num, group_num)
  = meson8_pmx_set_mux()
        │
        ├─→ للكل pin في الـ group:
        │     meson8_pmx_disable_other_groups(pc, pin, group_num)
        │           │
        │           └─→ يلف على كل الـ groups في pc->data->groups[]
        │                 ├─ skip: is_gpio == true
        │                 ├─ skip: i == sel_group  (الـ group المختار)
        │                 └─ إذا الـ group بيستخدم نفس الـ pin:
        │                       regmap_update_bits(pc->reg_mux,
        │                                         pmx_data->reg * 4,
        │                                         BIT(pmx_data->bit),
        │                                         0)         ← disable
        │
        └─→ إذا func_num != 0  (مش GPIO):
              regmap_update_bits(pc->reg_mux,
                                pmx_data->reg * 4,
                                BIT(pmx_data->bit),
                                BIT(pmx_data->bit))          ← enable
```

#### 4.2 طلب GPIO من الـ pinctrl

```
gpio subsystem calls:
  gpio_request(gpio_num)
        │
        ▼
  pinctrl core calls:
  meson8_pmx_ops.gpio_request_enable(pcdev, range, offset)
  = meson8_pmx_request_gpio()
        │
        └─→ meson8_pmx_disable_other_groups(pc, offset, -1)
                  │
                  └─→ sel_group = -1  ← disable ALL groups for this pin
                        └─→ regmap_update_bits(..., 0) لكل group
                              ← pin يرجع لـ GPIO mode
```

#### 4.3 Query الـ Functions/Groups (من userspace أو DT binding)

```
pinctrl core queries:
        │
        ├─→ meson8_pmx_ops.get_functions_count(pcdev)
        │     = meson_pmx_get_funcs_count()
        │           └─→ return pc->data->num_funcs
        │
        ├─→ meson8_pmx_ops.get_function_name(pcdev, selector)
        │     = meson_pmx_get_func_name()
        │           └─→ return pc->data->funcs[selector].name
        │
        └─→ meson8_pmx_ops.get_function_groups(pcdev, selector, ...)
              = meson_pmx_get_groups()
                    └─→ return pc->data->funcs[selector].groups
                              + pc->data->funcs[selector].num_groups
```

---

### 5. استراتيجية الـ Locking

الـ driver ده بسيط جداً في موضوع الـ locking لأنه بيعتمد على طبقتين بتوفرا الحماية:

#### 5.1 الـ pinctrl core lock

**الـ pinctrl core** (في `drivers/pinctrl/core.c`) بيمسك `mutex` على مستوى الـ `pinctrl_dev` قبل ما يستدعي أي `pmx_ops`. يعني:
- `meson8_pmx_set_mux()` و`meson8_pmx_request_gpio()` بيتنفذوا وهم محميين بـ `pctldev->mutex`.
- الـ driver نفسه **مش محتاج** يمسك lock يدوي.

#### 5.2 الـ regmap lock

كل call لـ `regmap_update_bits()` هي **atomic** من ناحية الـ regmap layer:
- الـ `regmap` بيستخدم `spinlock` أو `mutex` داخلياً (حسب config) لحماية الـ read-modify-write.
- الـ driver مش بيشيل الـ register bits يدوي — بيترك ده للـ regmap.

#### 5.3 ترتيب الـ Locking

```
[pinctrl core mutex]
       │
       └─→ [regmap internal lock]  (يتمسك داخل regmap_update_bits)
```

لا يوجد lock يُمسك في عكس الترتيب ده — **مفيش خطر deadlock**.

#### 5.4 ملاحظة الـ Glitch Prevention

في `meson8_pmx_set_mux()` في comment صريح:
```c
/*
 * Disable groups using the same pin.
 * The selected group is not disabled to avoid glitches.
 */
```

يعني أثناء التبديل:
1. الـ groups التانية بتتعطّل الأول (بدون ما الـ group الجديدة تتفعّل لسه).
2. بعدين الـ group الجديدة بتتفعّل.

ده بيمنع حالة ما يكونش فيه أي group active خالص أثناء الـ transition، اللي ممكن يسبب glitch على الـ pin.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Functions & APIs

| Function | Category | Signature Summary | الوظيفة |
|---|---|---|---|
| `meson8_pmx_disable_other_groups` | Runtime Helper | `static void (pc, pin, sel_group)` | تعطيل كل الـ groups اللي بتستخدم pin معين ما عدا الـ selected |
| `meson8_pmx_set_mux` | Runtime Core | `static int (pcdev, func_num, group_num)` | تفعيل function معينة على group معينة عبر كتابة الـ mux register |
| `meson8_pmx_request_gpio` | Runtime Core | `static int (pcdev, range, offset)` | تحويل pin لـ GPIO mode بتعطيل كل الـ pinmux groups المرتبطة بيه |

#### Macros

| Macro | ملف | الوظيفة |
|---|---|---|
| `PMX_DATA(r, b, g)` | `.h` | ينشئ `meson8_pmx_data` initializer بالـ register، bit، و is_gpio flag |
| `GROUP(grp, r, b)` | `.h` | ينشئ `meson_pmx_group` لـ function group حقيقية مع ربطها بالـ mux register/bit |
| `GPIO_GROUP(gpio)` | `.h` | ينشئ `meson_pmx_group` خاصة بـ GPIO-only pin، `is_gpio=true` |

#### Exported Symbols

| Symbol | النوع | الوظيفة |
|---|---|---|
| `meson8_pmx_ops` | `const struct pinmux_ops` | الـ vtable الرئيسي للـ pinmux ops، يتسجل في `meson_pinctrl_data.pmx_ops` |

---

### تصنيف الـ Functions

```
┌─────────────────────────────────────────────────────┐
│              meson8 PMX Driver Layout                │
├─────────────────┬───────────────────────────────────┤
│  Core Runtime   │  meson8_pmx_set_mux               │
│                 │  meson8_pmx_request_gpio           │
├─────────────────┼───────────────────────────────────┤
│  Internal Helper│  meson8_pmx_disable_other_groups  │
├─────────────────┼───────────────────────────────────┤
│  Ops Table      │  meson8_pmx_ops (vtable)          │
├─────────────────┼───────────────────────────────────┤
│  Data Macros    │  PMX_DATA / GROUP / GPIO_GROUP    │
└─────────────────┴───────────────────────────────────┘
```

---

### Group 1: Internal Helpers

الـ helper دي مسؤولة عن enforcing قاعدة أساسية في الـ hardware: كل pin ممكن ينتمي لأكتر من group، لكن مينفعش يتفعل أكتر من group في نفس الوقت. الـ hardware model هنا **one-hot**: كل pin له مجموعة من bits في الـ mux registers، وأي bit يتفعل بيحوّل الـ pin لـ function معينة. لو اتنين bits اتفعلوا في نفس الوقت، السلوك undefined.

---

#### `meson8_pmx_disable_other_groups`

```c
static void meson8_pmx_disable_other_groups(struct meson_pinctrl *pc,
                                            unsigned int pin,
                                            int sel_group)
```

**ما بتعمله:**
بتمشي على كل الـ groups المعرّفة في الـ SoC، وبتشوف أي group بتحتوي على الـ `pin` المطلوب. أي group تلاقيها (وهي مش الـ `sel_group` المختارة وليست `is_gpio`) بتعمل لها `regmap_update_bits` تمسح الـ bit المقابل ليها في الـ mux register، يعني تعطّلها.

**البارامترات:**

| Parameter | النوع | الوصف |
|---|---|---|
| `pc` | `struct meson_pinctrl *` | الـ pin controller الرئيسي، فيه الـ `reg_mux` regmap وبيانات الـ SoC |
| `pin` | `unsigned int` | رقم الـ pin اللي عايزين نضمن إنه مش بيستخدمه غير group واحدة |
| `sel_group` | `int` | index الـ group المختارة اللي محتاجين نحافظ عليها (نمررها `-1` لو عايزين نعطل الكل) |

**Return Value:** `void` — مفيش return، الـ errors من `regmap_update_bits` بتتجاهل (الكود مش بيتعامل معاها، وده متعمد لأن الـ disable path مش critical path).

**Key Details:**

- **Locking:** الـ function نفسها مش بتاخد lock، لكن الـ callers (core pinctrl subsystem) بيمسكوا `pinctrl_dev->mutex` قبل ما ينادوا عليها.
- **`is_gpio` Skip:** الـ GPIO groups (اللي `pmx_data->is_gpio == true`) بتتخطى لأنها مش ليها mux bit حقيقي (`reg=0, bit=0`).
- **`sel_group` Skip:** الـ group المختارة بتتخطى **عمداً** لتجنب الـ glitch اللي بيحصل لو عطّلنا الـ group الحالية قبل ما نفعّل الجديدة.
- **Register Addressing:** الـ offset في الـ regmap بيتحسب كـ `pmx_data->reg * 4` (يعني word-addressed، كل register 32-bit).

**Pseudocode Flow:**

```
for each group i in pc->data->groups:
    pmx_data = group[i].data
    if pmx_data->is_gpio OR i == sel_group:
        skip
    for each pin j in group[i].pins:
        if group[i].pins[j] == pin:
            regmap_update_bits(reg_mux,
                               pmx_data->reg * 4,
                               BIT(pmx_data->bit),
                               0)          // clear the bit
```

**ASCII: مثال على pin مشترك بين groups**

```
Pin X مستخدم في:
  Group A  → reg=1, bit=5  ← يتعطل
  Group B  → reg=2, bit=3  ← يتعطل (sel_group)  ← يتحافظ عليه
  Group C  → reg=3, bit=1  ← يتعطل

نتيجة الـ regmap writes:
  reg 0x04: BIT(5) → 0
  reg 0x0C: BIT(1) → 0
  reg 0x08: BIT(3) → unchanged
```

---

### Group 2: Core Runtime (Pinmux Ops Implementations)

دول الـ functions اللي بتتجوزهم الـ pinctrl core من خلال الـ `pinmux_ops` vtable. الـ core بيناديهم لما user-space أو kernel driver يطلب تفعيل function معينة أو GPIO mode.

---

#### `meson8_pmx_set_mux`

```c
static int meson8_pmx_set_mux(struct pinctrl_dev *pcdev,
                               unsigned func_num,
                               unsigned group_num)
```

**ما بتعمله:**
دي الـ function الرئيسية للـ pinmux. لما driver يطلب تفعيل function معينة (زي UART أو I2C) على group معينة، الـ core بينادي على `set_mux`. هي بتعطّل الـ conflicting groups الأول، وبعدين لو الـ function مش GPIO (func_num != 0) بتكتب الـ bit في الـ mux register لتفعيل الـ function.

**البارامترات:**

| Parameter | النوع | الوصف |
|---|---|---|
| `pcdev` | `struct pinctrl_dev *` | الـ pinctrl device، بيتحول لـ `meson_pinctrl` عبر `pinctrl_dev_get_drvdata` |
| `func_num` | `unsigned` | index الـ function في `pc->data->funcs[]`، قيمة 0 = GPIO |
| `group_num` | `unsigned` | index الـ group في `pc->data->groups[]` |

**Return Value:** `int` — 0 عند النجاح، أو error code من `regmap_update_bits` في حالة فشل الكتابة.

**Key Details:**

- **GPIO Special Case:** لو `func_num == 0`، الـ function دي GPIO، ومش محتاجة تكتب أي bit في الـ mux register. بس لازم تعطّل الـ conflicting groups الأول. الـ GPIO mode هي الـ default لما مفيش mux bit متفعل.
- **Glitch Prevention:** الـ disable بيحصل **قبل** الـ enable. وكمان الـ `sel_group` بيتمرر لـ `disable_other_groups` عشان الـ selected group متتعطلش قبل ما نفعّلها.
- **Locking:** الـ pinctrl core بيمسك الـ mutex قبل ما ينادي على الـ function دي.
- **Per-pin iteration:** الـ disable loop بتمشي على كل الـ pins في الـ group، مش pin واحد، عشان group ممكن تحتوي على أكتر من pin (مثلاً SPI group فيها 4 pins).

**Pseudocode Flow:**

```
pc = get driver data from pcdev
func = pc->data->funcs[func_num]
group = pc->data->groups[group_num]
pmx_data = group->data

for each pin in group->pins:
    meson8_pmx_disable_other_groups(pc, pin, group_num)

if func_num != 0:  // not GPIO
    regmap_update_bits(reg_mux,
                       pmx_data->reg * 4,
                       BIT(pmx_data->bit),
                       BIT(pmx_data->bit))  // set the bit
return 0 or error
```

---

#### `meson8_pmx_request_gpio`

```c
static int meson8_pmx_request_gpio(struct pinctrl_dev *pcdev,
                                    struct pinctrl_gpio_range *range,
                                    unsigned offset)
```

**ما بتعمله:**
لما driver يطلب `gpio_request` على pin معين، الـ pinctrl core بينادي على `gpio_request_enable`. الـ function دي بتعمل حاجة واحدة بس: تعطّل كل الـ pinmux groups اللي بتستخدم الـ pin ده، وده بيخلّي الـ pin يرجع لـ GPIO mode تلقائياً (لأن GPIO هو الـ default state لما مفيش mux bit متفعل).

**البارامترات:**

| Parameter | النوع | الوصف |
|---|---|---|
| `pcdev` | `struct pinctrl_dev *` | الـ pinctrl device |
| `range` | `struct pinctrl_gpio_range *` | الـ GPIO range اللي الـ pin ينتمي ليها (مش بتتستخدم هنا، بس لازمة للـ vtable signature) |
| `offset` | `unsigned` | الـ pin number اللي محتاجين نحوّله لـ GPIO mode |

**Return Value:** `int` — دايماً 0 (الـ `disable_other_groups` مش بترجع error).

**Key Details:**

- الـ `sel_group` بتتمرر كـ `-1`، ده معناه "عطّل كل الـ groups حتى لو مفيش group محددة".
- `range` parameter مش بيتستخدم في الـ implementation دي لأن الـ Meson8 hardware مش محتاج معلومات الـ range لتفعيل GPIO mode.
- الـ caller هو `pinctrl_gpio_request` في `drivers/pinctrl/core.c`.

---

### Group 3: Ops Table (vtable)

---

#### `meson8_pmx_ops`

```c
const struct pinmux_ops meson8_pmx_ops = {
    .set_mux              = meson8_pmx_set_mux,
    .get_functions_count  = meson_pmx_get_funcs_count,
    .get_function_name    = meson_pmx_get_func_name,
    .get_function_groups  = meson_pmx_get_groups,
    .gpio_request_enable  = meson8_pmx_request_gpio,
};
EXPORT_SYMBOL_GPL(meson8_pmx_ops);
```

**الـ vtable دي** هي الـ interface بين الـ pinctrl core وبين الـ Meson8-specific implementation. الـ core بيستخدمها لـ:

| Op | Implementation | الوظيفة |
|---|---|---|
| `.set_mux` | `meson8_pmx_set_mux` | تفعيل function على group |
| `.get_functions_count` | `meson_pmx_get_funcs_count` | عدد الـ functions المتاحة (من `pinctrl-meson.c`) |
| `.get_function_name` | `meson_pmx_get_func_name` | اسم الـ function بالـ index (من `pinctrl-meson.c`) |
| `.get_function_groups` | `meson_pmx_get_groups` | الـ groups المرتبطة بـ function (من `pinctrl-meson.c`) |
| `.gpio_request_enable` | `meson8_pmx_request_gpio` | تحويل pin لـ GPIO mode |

**`EXPORT_SYMBOL_GPL`:** الـ symbol ده exported لأن الـ SoC-specific drivers (زي `pinctrl-meson8.c`) بيستخدموه في الـ `meson_pinctrl_data` الخاصة بيهم. الـ drivers دي module منفصلة عن الـ PMX driver.

---

### Group 4: Data Structures & Macros

---

#### `struct meson8_pmx_data`

```c
struct meson8_pmx_data {
    bool         is_gpio;   /* true = GPIO group, no real mux bit */
    unsigned int reg;       /* register index (word offset, *4 for byte) */
    unsigned int bit;       /* bit position within the register */
};
```

الـ struct دي هي الـ per-group private data. بتتخزن في `meson_pmx_group.data` كـ `const void *` وبتتكست manually لـ `struct meson8_pmx_data *` في الـ runtime.

**Hardware Mapping:**

```
MUX Register Space:
  reg_mux base address
       │
       ├── offset = reg * 4   (byte address)
       │        │
       │        └── bit = bit position
       │                 │
       │                 └── 1 = function enabled
       │                     0 = function disabled
```

---

#### `PMX_DATA(r, b, g)`

```c
#define PMX_DATA(r, b, g)   \
    {                       \
        .reg    = r,        \
        .bit    = b,        \
        .is_gpio = g,       \
    }
```

**Compound literal initializer** لـ `meson8_pmx_data`. بيستخدم في الـ `GROUP` و`GPIO_GROUP` macros.

---

#### `GROUP(grp, r, b)`

```c
#define GROUP(grp, r, b)                                \
    {                                                   \
        .name     = #grp,                              \
        .pins     = grp##_pins,                        \
        .num_pins = ARRAY_SIZE(grp##_pins),            \
        .data = (const struct meson8_pmx_data[]){      \
            PMX_DATA(r, b, false),                     \
        },                                             \
    }
```

بينشئ `meson_pmx_group` لـ function group حقيقية. بيفترض إن في array اسمها `grp##_pins` معرّفة قبليه في نفس الـ translation unit.

**مثال:**
```c
// الـ array الأول
static const unsigned int uart_tx_pins[] = { PIN_UART_TX };

// الـ GROUP macro
GROUP(uart_tx, 5, 11)
// ينتج:
// { .name="uart_tx", .pins=uart_tx_pins,
//   .num_pins=1, .data={.reg=5,.bit=11,.is_gpio=false} }
```

---

#### `GPIO_GROUP(gpio)`

```c
#define GPIO_GROUP(gpio)                                \
    {                                                   \
        .name     = #gpio,                             \
        .pins     = (const unsigned int[]){ gpio },    \
        .num_pins = 1,                                 \
        .data = (const struct meson8_pmx_data[]){      \
            PMX_DATA(0, 0, true),                      \
        },                                             \
    }
```

بينشئ group خاصة بـ GPIO pins. الـ `is_gpio=true` بيخلي `meson8_pmx_disable_other_groups` تتخطاها ولا تحاول تكتب في register 0 / bit 0 (اللي dummy values).

---

### Data Flow الكامل

```
gpio_request(pin N)
      │
      ▼
pinctrl_gpio_request()          [pinctrl core]
      │
      ▼
meson8_pmx_request_gpio()       [our driver]
      │
      └──► meson8_pmx_disable_other_groups(pc, N, -1)
                │
                ├── iterate all groups
                ├── find groups containing pin N
                └── regmap_update_bits(reg_mux, reg*4, BIT(bit), 0)


pinctrl_select_state("uart0")
      │
      ▼
pinmux_enable_setting()         [pinctrl core]
      │
      ▼
meson8_pmx_set_mux(pcdev, func=UART, group=uart_tx)
      │
      ├── for each pin in uart_tx group:
      │       meson8_pmx_disable_other_groups(pc, pin, uart_tx_idx)
      │
      └── regmap_update_bits(reg_mux, reg*4, BIT(bit), BIT(bit))
                                                ▲
                                         sets the mux bit
                                         → UART function active
```
## Phase 5: دليل الـ Debugging الشامل

الـ driver ده خاص بـ **pinmux** لجيل Meson8 من Amlogic — بيتحكم في تفعيل وتعطيل الـ pinmux groups عن طريق bits في registers الـ `reg_mux` regmap. الـ debugging هنا بيدور حول: هل الـ bit الصح اتكتب في الـ register الصح؟ هل الـ group اتفعّل/اتعطّل صح؟

---

### Software Level

#### 1. debugfs — المدخلات المهمة

الـ pinctrl subsystem بيكشف بيانات كتير عبر `/sys/kernel/debug/pinctrl/`.

| المسار | المحتوى | الاستخدام |
|--------|---------|-----------|
| `/sys/kernel/debug/pinctrl/` | قائمة كل الـ pinctrl devices | معرفة اسم الـ device |
| `/sys/kernel/debug/pinctrl/<dev>/pins` | كل الـ pins مع أرقامها | التحقق من وجود الـ pin |
| `/sys/kernel/debug/pinctrl/<dev>/pingroups` | كل الـ groups وpin list كل group | التحقق من تعريف الـ group |
| `/sys/kernel/debug/pinctrl/<dev>/pinmux-pins` | كل pin ومين بيملكه حالياً | أهم ملف للـ debug |
| `/sys/kernel/debug/pinctrl/<dev>/pinmux-functions` | الـ functions والـ groups المرتبطة بيها | التحقق من تعريف الـ function |
| `/sys/kernel/debug/gpio` | حالة كل GPIO | التحقق من ownership |

```bash
# اعرف اسم الـ pinctrl device الخاص بـ Meson8
ls /sys/kernel/debug/pinctrl/

# اقرأ حالة كل pin — هل متصل بـ function ولا GPIO؟
cat /sys/kernel/debug/pinctrl/pinctrl@c1109880/pinmux-pins

# اعرف الـ groups المتاحة
cat /sys/kernel/debug/pinctrl/pinctrl@c1109880/pingroups

# اعرف الـ functions وgroups كل واحدة
cat /sys/kernel/debug/pinctrl/pinctrl@c1109880/pinmux-functions
```

**مثال Output لـ `pinmux-pins`:**
```
pin 42 (GPIOX_10): pinmux_uart_a_tx_group (GPIO UNCLAIMED)
pin 43 (GPIOX_11): pinmux_uart_a_rx_group (GPIO UNCLAIMED)
pin 44 (GPIOX_12): (MUX UNCLAIMED) (GPIO UNCLAIMED)
```
لو الـ pin `MUX UNCLAIMED` يعني مفيش function طلبته — ابحث في الـ DT أو الـ driver.

---

#### 2. sysfs — المدخلات المهمة

| المسار | المحتوى |
|--------|---------|
| `/sys/bus/platform/devices/<dev>/` | الـ device attributes |
| `/sys/class/gpio/` | الـ GPIO chips المسجّلة |
| `/sys/kernel/debug/gpio` | تفاصيل كل GPIO مع direction وvalue |

```bash
# اعرف الـ gpio chips المسجّلة
cat /sys/kernel/debug/gpio

# اعرف الـ regmap registers (لو CONFIG_REGMAP_DEBUGFS=y)
ls /sys/kernel/debug/regmap/
cat /sys/kernel/debug/regmap/<dev>-mux/registers
```

---

#### 3. ftrace — الـ Tracepoints المهمة

```bash
# فعّل tracing للـ regmap writes — ده أهم حاجة لمتابعة كتابة الـ bits
echo 1 > /sys/kernel/debug/tracing/events/regmap/regmap_reg_write/enable
echo 1 > /sys/kernel/debug/tracing/events/regmap/regmap_reg_read/enable

# فعّل tracing للـ pinctrl
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/enable

# ابدأ الـ trace
echo 1 > /sys/kernel/debug/tracing/tracing_on

# افعل العملية اللي بتصحح فيها (مثلاً: device probe)
# ...

# اقرأ النتيجة
cat /sys/kernel/debug/tracing/trace

# وقّف
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

**مثال Output:**
```
kworker/0:1-42    [000] .... regmap_reg_write: pinctrl@c1109880 reg=0x2c val=0x00000004
```
هنا بنشوف: الـ `reg_mux` كتب في offset `0x2c` (= reg index 11 × 4) القيمة `0x4` (bit 2).

لتتبع `meson8_pmx_set_mux` تحديداً:
```bash
# استخدم function tracer
echo meson8_pmx_set_mux > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

---

#### 4. printk / Dynamic Debug

الـ driver بيستخدم `dev_dbg()` في `meson8_pmx_set_mux` — ده بيطلع بس لو فعّلت الـ dynamic debug:

```bash
# فعّل debug messages للـ pinctrl meson8 module
echo "module pinctrl-meson8-pmx +p" > /sys/kernel/debug/dynamic_debug/control

# أو لملف محدد
echo "file pinctrl-meson8-pmx.c +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل كل الـ debug في الـ pinctrl subsystem
echo "module pinctrl +p" > /sys/kernel/debug/dynamic_debug/control
```

**Output متوقع بعد التفعيل:**
```
pinctrl pinctrl@c1109880: enable function uart_a, group uart_a_tx
pinctrl pinctrl@c1109880: enable function uart_a, group uart_a_rx
```

لو مش شايف الرسالة دي — يعني إما `meson8_pmx_set_mux` مش اتنادى أو الـ func_num == 0 (GPIO mode).

**kernel boot parameter بديل:**
```
dyndbg="module pinctrl-meson8-pmx +p"
```

---

#### 5. Kernel Config Options للـ Debugging

| Config | الوظيفة | ملاحظة |
|--------|---------|--------|
| `CONFIG_PINCTRL_MESON` | الـ driver الأساسي | لازم يكون `y` أو `m` |
| `CONFIG_PINCTRL_MESON8` | الـ Meson8 SoC data | لازم يكون `y` أو `m` |
| `CONFIG_DEBUG_PINCTRL` | تفعيل debug في pinctrl core | بيضيف logging إضافي |
| `CONFIG_REGMAP_DEBUGFS` | كشف الـ regmap registers في debugfs | مهم جداً لرؤية قيم الـ MUX registers |
| `CONFIG_DYNAMIC_DEBUG` | تفعيل `dev_dbg` runtime | لازم لرسائل الـ dev_dbg |
| `CONFIG_GPIO_SYSFS` | كشف GPIO عبر sysfs | للتحقق اليدوي |
| `CONFIG_KALLSYMS` | أسماء symbols في stack traces | مهم لقراءة dump_stack |
| `CONFIG_PROVE_LOCKING` | كشف deadlocks | لو فيه مشاكل لocking |
| `CONFIG_DEBUG_GPIO` | debug GPIO core | مفيد مع gpio_request_enable |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "PINCTRL|REGMAP_DEBUG|DEBUG_GPIO"
```

---

#### 6. devlink وأدوات خاصة بالـ Subsystem

الـ meson8-pmx مش بيستخدم devlink — بس فيه أدوات pinctrl مباشرة:

```bash
# أداة pinctrl userspace (من package libpinctrl أو buildroot)
# بتتيح query مباشر للـ pinctrl

# لو مش متاحة، استخدم sysfs مباشرة:
# اقرأ حالة pin معين
cat /sys/kernel/debug/pinctrl/pinctrl@c1109880/pinmux-pins | grep "GPIOX_10"

# اعرف مين بيملك الـ GPIO حالياً
cat /sys/kernel/debug/gpio | grep -i "meson"

# regmap dump — أهم tool لرؤية قيم MUX registers فعلياً
cat /sys/kernel/debug/regmap/*/registers 2>/dev/null | head -60
```

---

#### 7. جدول Error Messages الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---------------------|--------|------|
| `could not get mux regmap` | فشل في الوصول لـ `reg_mux` regmap | تحقق من DT — هل `reg-names = "mux"` موجود؟ |
| `pin X already requested` | pin متطلوب من driver تاني | تحقق من تعارض في DT أو `pinctrl-0` |
| `could not request pin X` | فشل `pin_request` في الـ core | تحقق من `pinmux-pins` — مين ماسكه؟ |
| `invalid pin number X` | رقم pin خارج النطاق | تحقق من `num_pins` في `meson_pinctrl_data` |
| `pin X is not valid` | الـ pin مش معرّف في `pins` array | تحقق من `MESON_PIN()` macros |
| `unable to find group X` | اسم group غلط في DT | قارن الاسم بالـ `GROUP()` macro |
| `regmap_update_bits failed` | فشل كتابة الـ register | تحقق من الـ regmap config وحدود الـ register |
| `meson8_pmx_set_mux: enable function X, group Y` | معلوماتية — الـ mux بيتفعّل | طبيعي، يظهر مع dynamic debug |

---

#### 8. نقاط استراتيجية لـ dump_stack() وWARN_ON()

في `meson8_pmx_disable_other_groups()` — لو حابب تتحقق إن الـ group صح:

```c
static void meson8_pmx_disable_other_groups(struct meson_pinctrl *pc,
                                            unsigned int pin, int sel_group)
{
    const struct meson_pmx_group *group;
    struct meson8_pmx_data *pmx_data;
    int i, j;

    /* Debug: verify reg_mux is valid before any write */
    WARN_ON(!pc->reg_mux);

    for (i = 0; i < pc->data->num_groups; i++) {
        group = &pc->data->groups[i];
        pmx_data = (struct meson8_pmx_data *)group->data;

        /* Debug: catch NULL pmx_data */
        WARN_ON(!pmx_data);

        if (pmx_data->is_gpio || i == sel_group)
            continue;

        for (j = 0; j < group->num_pins; j++) {
            if (group->pins[j] == pin) {
                /* Debug: log which group is being disabled */
                dev_dbg(pc->dev, "disabling group %s (reg=%u bit=%u) for pin %u\n",
                        group->name, pmx_data->reg, pmx_data->bit, pin);
            }
        }
    }
}
```

في `meson8_pmx_set_mux()`:

```c
static int meson8_pmx_set_mux(struct pinctrl_dev *pcdev,
                               unsigned func_num, unsigned group_num)
{
    struct meson_pinctrl *pc = pinctrl_dev_get_drvdata(pcdev);
    const struct meson_pmx_func *func = &pc->data->funcs[func_num];
    const struct meson_pmx_group *group = &pc->data->groups[group_num];
    struct meson8_pmx_data *pmx_data =
        (struct meson8_pmx_data *)group->data;

    /* Debug: bounds check */
    WARN_ON(func_num >= pc->data->num_funcs);
    WARN_ON(group_num >= pc->data->num_groups);
    WARN_ON(pmx_data->reg > 15); /* sanity: MUX reg index shouldn't be huge */
    WARN_ON(pmx_data->bit >= 32);
    /* ... */
}
```

---

### Hardware Level

#### 1. التحقق إن حالة الـ Hardware بتطابق حالة الـ Kernel

الـ MUX registers في Meson8 بتبدأ من base address الـ `reg_mux` اللي جاي من DT. كل group عنده `reg` و`bit` — الـ bit ده لما يكون 1 يعني الـ function متفعّل.

```
MUX Register Layout:
Offset = pmx_data->reg * 4

Bit(pmx_data->bit) = 1  →  Function enabled
Bit(pmx_data->bit) = 0  →  Function disabled (GPIO mode)
```

للتحقق اليدوي:
```bash
# افرض الـ MUX base address = 0xC1109880
# وعندك group بـ reg=2, bit=5
# الـ register الفعلي = 0xC1109880 + (2 * 4) = 0xC1109888

devmem2 0xC1109888 w
# Output:
# /dev/mem opened.
# Memory mapped at address 0x...
# Value at address 0xC1109888 (0x...): 0x00000020
# 0x20 = bit 5 set → Group is ENABLED ✓
```

مقارنة مع الـ kernel:
```bash
cat /sys/kernel/debug/pinctrl/pinctrl@c1109880/pinmux-pins | grep <group-name>
```

لو الـ hardware قال bit = 1 والـ kernel قال GPIO mode — في race condition أو كتابة خارج الـ driver.

---

#### 2. Register Dump بـ devmem2 و/dev/mem

```bash
# تثبت devmem2 لو مش موجود
apt-get install devmem2  # أو ابنيه من المصدر

# dump أول 16 MUX register (64 bytes) من base الـ pinctrl
BASE=0xC1109880
for i in $(seq 0 15); do
    ADDR=$(printf "0x%X" $((BASE + i * 4)))
    VAL=$(devmem2 $ADDR w 2>/dev/null | grep "Value" | awk '{print $NF}')
    echo "REG[$i] @ $ADDR = $VAL"
done
```

**مثال Output:**
```
REG[0]  @ 0xC1109880 = 0x00000000
REG[1]  @ 0xC1109884 = 0x00000008
REG[2]  @ 0xC1109888 = 0x00000020
REG[3]  @ 0xC110988C = 0x00000000
```
بتحلل كل value: كل bit ممكن يكون group مختلف — راجع `GROUP()` macros في الـ SoC-specific file (زي `pinctrl-meson8.c`).

**بديل عبر /proc/iomem وio:**
```bash
# تحقق إن العنوان مسجّل
cat /proc/iomem | grep -i "pinctrl\|mux"

# استخدم io tool (من busybox)
io -4 -r 0xC1109888
```

---

#### 3. Logic Analyzer / Oscilloscope

لـ debug مشاكل الـ pinmux على مستوى الإشارة:

- **Logic Analyzer**: صِل على الـ pin المشكوك فيه وعلى الـ clock/data المتوقع
  - تحقق إن الـ pin بيتصرف كـ UART مثلاً (baud rate صح، polarity صح)
  - لو الـ pin مش بيتحرك بعد `pinmux_enable` → الـ bit مش اتكتب في الـ hardware

- **Oscilloscope**: لقياس drive strength — الـ Meson8 بيدعم 4 levels (500µA إلى 4mA)
  - rise time بطيء جداً → drive strength منخفض → تحقق من `MESON_REG_DS`

```
Checklist للـ Logic Analyzer:
┌─────────────────────────────────────┐
│ 1. هل الـ pin بيشتغل أصلاً؟        │
│ 2. هل الـ signal level صح (3.3V)?   │
│ 3. هل الـ baud rate / frequency صح? │
│ 4. هل فيه glitches لحظة الـ switch? │
└─────────────────────────────────────┘
```

الـ `meson8_pmx_disable_other_groups` بيتجنب disable الـ selected group أول ما بيشتغل — ده بيمنع glitches، لو لاحظت glitch في الـ logic analyzer يبقى الـ sequence بيتعمل wrong order.

---

#### 4. مشاكل Hardware شائعة وـ Kernel Log Patterns

| المشكلة | Pattern في الـ Log | التفسير |
|---------|--------------------|---------|
| UART مش بيشتغل بعد boot | لا يوجد output من UART | الـ pinmux مش اتفعّل — تحقق من DT `pinctrl-0` |
| Pin مش بيستجاب بعد كتابة GPIO | GPIO يقرأ wrong value | pin لسه في MUX mode — تحقق من `disable_other_groups` |
| Device probe فاشل | `could not request pin` | تعارض بين drivers على نفس الـ pin |
| Signal noise / interference | - | drive strength عالي جداً — قلل الـ DS |
| Pin عامل GPIO وهو المفروض function | `pin X: GPIO UNCLAIMED` | الـ consumer driver مش طالب الـ pinmux |

---

#### 5. Device Tree Debugging

الـ DT هو المصدر الأساسي لـ configuration الـ pinmux في Meson8.

```bash
# اقرأ الـ DT المحمّل فعلياً
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "pinctrl"

# تحقق من وجود الـ pinctrl node
ls /sys/firmware/devicetree/base/pinctrl@c1109880/

# اقرأ الـ reg property (MUX base address)
hexdump -C /sys/firmware/devicetree/base/pinctrl@c1109880/reg

# تحقق من الـ consumer device (مثلاً UART)
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 5 "uart" | grep "pinctrl-"
```

**DT صح لـ UART على Meson8:**
```dts
uart_a: serial@c11084c0 {
    compatible = "amlogic,meson-uart";
    pinctrl-0 = <&uart_a_pins>;
    pinctrl-names = "default";
    status = "okay";
};

&pinctrl_periphs {
    uart_a_pins: uart-a {
        mux {
            groups = "uart_tx_a", "uart_rx_a";
            function = "uart_a";
        };
    };
};
```

**نقاط التحقق:**
```
┌──────────────────────────────────────────────────────┐
│ 1. اسم الـ group في DT = اسمه في GROUP() macro؟      │
│ 2. اسم الـ function في DT = اسمه في FUNCTION() macro?│
│ 3. الـ pinctrl-names = "default" موجود؟               │
│ 4. الـ pinctrl-0 بيشاور على الـ node الصح؟            │
│ 5. الـ status = "okay" للـ consumer device؟           │
└──────────────────────────────────────────────────────┘
```

```bash
# تحقق إن الـ group name في kernel بيطابق الـ DT
cat /sys/kernel/debug/pinctrl/pinctrl@c1109880/pingroups | grep "uart_tx_a"
# لو مش موجود → الاسم في DT غلط أو الـ SoC data مش محمّل
```

---

### Practical Commands

#### سكريبت شامل للـ Debug

```bash
#!/bin/sh
# meson8-pinmux-debug.sh — debug script for Meson8 pinmux

DEV=$(ls /sys/kernel/debug/pinctrl/ | grep -i meson | head -1)
if [ -z "$DEV" ]; then
    echo "ERROR: No meson pinctrl device found in debugfs"
    exit 1
fi

echo "=== Meson8 Pinmux Debug Report ==="
echo "Device: $DEV"
echo ""

echo "--- Active Pinmux Assignments ---"
cat /sys/kernel/debug/pinctrl/$DEV/pinmux-pins
echo ""

echo "--- Available Functions ---"
cat /sys/kernel/debug/pinctrl/$DEV/pinmux-functions
echo ""

echo "--- GPIO State ---"
cat /sys/kernel/debug/gpio 2>/dev/null | grep -A 5 "meson\|gpiox\|gpioy" -i
echo ""

echo "--- Regmap MUX Registers ---"
if ls /sys/kernel/debug/regmap/ 2>/dev/null | grep -q "."; then
    for d in /sys/kernel/debug/regmap/*/; do
        echo "Regmap: $d"
        cat "$d/registers" 2>/dev/null | head -20
    done
else
    echo "REGMAP_DEBUGFS not enabled — enable CONFIG_REGMAP_DEBUGFS"
fi
```

```bash
# تفعيل ftrace لمتابعة كتابة MUX registers
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo "" > /sys/kernel/debug/tracing/trace
echo "function" > /sys/kernel/debug/tracing/current_tracer
echo "meson8_pmx_set_mux" > /sys/kernel/debug/tracing/set_ftrace_filter
echo "meson8_pmx_disable_other_groups" >> /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/events/regmap/regmap_reg_write/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on

# اعمل probe للـ device
echo "$(cat /sys/bus/platform/devices/*/uevent 2>/dev/null | grep uart | head -1)"

echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

**تفسير نتيجة الـ trace:**
```
# example trace output:
kworker/0:2  [000]  meson8_pmx_set_mux <-- func=1 (uart_a), group=5
kworker/0:2  [000]  meson8_pmx_disable_other_groups <-- pin=42, sel=5
kworker/0:2  [000]  regmap_reg_write: reg=0x08 val=0x00000000  ← disable old group
kworker/0:2  [000]  regmap_reg_write: reg=0x14 val=0x00000010  ← enable uart_a bit

# reg=0x14 = pmx_data->reg(5) * 4 = 20 = 0x14
# val=0x10 = BIT(4) → pmx_data->bit = 4
```

```bash
# dynamic debug — أسرع طريقة للـ trace بدون ftrace
echo "file pinctrl-meson8-pmx.c +pflmt" > /sys/kernel/debug/dynamic_debug/control
# +p = print, +f = function name, +l = line number, +m = module, +t = thread

# شغّل الـ device وراقب dmesg
dmesg -w | grep -i "meson8_pmx\|pinctrl\|mux"
```

```bash
# تحقق يدوي من MUX register بعد تفعيل UART
# افرض: uart_a_tx_group → reg=2, bit=5
# BASE من DT: reg = <0xc1109880 0x400>
devmem2 $((0xc1109880 + 2*4)) w
# لو bit 5 = 1 → UART TX مفعّل في الـ hardware
# لو bit 5 = 0 → الـ kernel كتب لكن الـ hardware مش بيستجاب

# تحقق من MUX regmap عبر debugfs
cat /sys/kernel/debug/regmap/c1109880.pinctrl-mux/registers 2>/dev/null
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: UART مش شغال على Android TV Box بـ Amlogic S905

#### العنوان
**UART console مش بيظهر عند boot على TV box تجاري**

#### السياق
board engineer بيعمل bring-up لـ Android TV box بـ Amlogic S905 (Meson GXBaby). الـ UART0 المفروض يكون هو الـ debug console على الـ header الجانبي. البورد وصلت من المصنع وفيها custom DT. بعد flash الـ firmware، ما فيش أي output على serial terminal.

#### المشكلة
الـ UART TX pin مش بيشتغل. الـ oscilloscope بيوريك إن الـ pin واقف على HIGH ومش بيتحرك. الـ UART driver اتحمل ومفيش kernel panic، بس الـ TX صامت.

#### التحليل
الـ pin اللي هو TX موجود في أكتر من group واحد في الـ pinmux table. لما الـ kernel بيعمل `pinctrl_select_state` على UART group، بيتنادى `meson8_pmx_set_mux`:

```c
static int meson8_pmx_set_mux(struct pinctrl_dev *pcdev, unsigned func_num,
                               unsigned group_num)
{
    /* ... */
    for (i = 0; i < group->num_pins; i++)
        meson8_pmx_disable_other_groups(pc, group->pins[i], group_num);

    if (func_num)
        ret = regmap_update_bits(pc->reg_mux, pmx_data->reg * 4,
                                 BIT(pmx_data->bit),
                                 BIT(pmx_data->bit));
    return ret;
}
```

الـ `meson8_pmx_disable_other_groups` بتمشي على كل الـ groups وبتعمل clear لأي group تاني بيستخدم نفس الـ pin. المشكلة إن في الـ custom DT، الـ TX pin اتعرّف كمان في group تاني (مثلاً SPI_MOSI أو PWM) وهو الأحدث في الـ probing order. فالـ SPI group اتفعّل بعد الـ UART، وعمل clear لـ bit الـ UART:

```c
static void meson8_pmx_disable_other_groups(struct meson_pinctrl *pc,
                                             unsigned int pin, int sel_group)
{
    for (i = 0; i < pc->data->num_groups; i++) {
        group = &pc->data->groups[i];
        pmx_data = (struct meson8_pmx_data *)group->data;
        if (pmx_data->is_gpio || i == sel_group)
            continue;
        for (j = 0; j < group->num_pins; j++) {
            if (group->pins[j] == pin) {
                regmap_update_bits(pc->reg_mux,
                                   pmx_data->reg * 4,
                                   BIT(pmx_data->bit), 0); /* clear! */
            }
        }
    }
}
```

#### الحل
**1- تشخيص:**
```bash
# اشوف state الـ pinctrl الحالية
cat /sys/kernel/debug/pinctrl/pinctrl-handles

# اشوف registers الـ mux مباشرة
devmem2 0xc1109880 w   # MUX register base للـ Meson8
```

**2- تصحيح الـ DT:** إزالة الـ pin المتعارض من الـ SPI group في الـ device tree:

```dts
/* خطأ: نفس الـ pin في SPI */
&spi_0 {
    pinctrl-0 = <&spi_pins>;  /* كان بياخد TX pin */
};

/* صح: disable الـ SPI أو استخدم pins تانية */
&spi_0 {
    status = "disabled";
};

&uart_A {
    pinctrl-0 = <&uart_a_pins>;
    pinctrl-names = "default";
    status = "okay";
};
```

#### الدرس المستفاد
الـ `meson8_pmx_disable_other_groups` مصمم عشان يضمن إن pin واحد ميستخدمش في أكتر من function في نفس الوقت. لازم تتأكد إن كل pin في الـ DT مش معرّف في أكتر من group نشط في نفس الوقت، لأن آخر `set_mux` بيعمل override على كل اللي قبله.

---

### السيناريو 2: I2C Sensor مش بيتعرف على Amlogic Meson8b Media Player

#### العنوان
**I2C temperature sensor على Meson8b بيرجع `-ENXIO` دايماً**

#### السياق
media player بـ Amlogic Meson8b (S812). في الـ board فيه temperature sensor (LM75) متوصل على I2C bus B. الـ driver بيتحمل وبيعمل probe بس كل read بيرجع `-ENXIO` (no device).

#### المشكلة
الـ I2C bus مش بتشتغل لأن الـ SDA/SCL pins فاضلة في GPIO mode بدل ما تكون في I2C function.

#### التحليل
لما بيتعمل `gpio_request` على pin معين (مثلاً من driver تاني أو من userspace عن طريق sysfs)، بيتنادى `meson8_pmx_request_gpio`:

```c
static int meson8_pmx_request_gpio(struct pinctrl_dev *pcdev,
                                    struct pinctrl_gpio_range *range,
                                    unsigned offset)
{
    struct meson_pinctrl *pc = pinctrl_dev_get_drvdata(pcdev);

    /* بيعمل disable لكل الـ groups اللي بتستخدم الـ pin دا */
    meson8_pmx_disable_other_groups(pc, offset, -1);

    return 0;
}
```

الـ `-1` في `sel_group` معناه "disable الكل بدون استثناء". في الكود:

```c
if (pmx_data->is_gpio || i == sel_group)
    continue;
/* sel_group = -1, يعني i == -1 مستحيل، فكل الـ groups هتتعطل */
```

فيه script في الـ init system بيعمل `echo "in" > /sys/class/gpio/gpio_X/direction` على pin اللي هو نفس SCL. ده بيعمل `gpio_request` → `meson8_pmx_request_gpio` → يعطّل I2C group.

#### الحل
**1- تحديد المشكلة:**
```bash
# شوف مين بيطلب الـ GPIO
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable
cat /sys/kernel/debug/tracing/trace | grep "gpio_X"

# شوف الـ pinctrl state بعد المشكلة
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep "i2c"
```

**2- تصحيح init script:** إزالة السطر اللي بيعمل export للـ GPIO المتعارض.

**3- حماية الـ pin في DT:**
```dts
/* اجعل الـ I2C مضبوط كـ default state */
&i2c_B {
    pinctrl-0 = <&i2c_b_pins>;
    pinctrl-names = "default";
    status = "okay";

    lm75: temperature-sensor@48 {
        compatible = "national,lm75";
        reg = <0x48>;
    };
};
```

**4- منع الـ userspace من المس بالـ pin:**
```bash
# في /etc/rc.local امنع export للـ pins دي
# أو استخدم pinctrl hog في DT
```

```dts
/* hog الـ pin عشان ميتحررش */
pinctrl {
    i2c_b_hog: i2c-b-hog {
        pins = "I2C_B_SDA", "I2C_B_SCL";
        function = "i2c_b";
    };
};
```

#### الدرس المستفاد
الـ `meson8_pmx_request_gpio` مع `sel_group = -1` بيعمل hard reset لكل الـ mux bits المتعلقة بالـ pin. أي `gpio_request` على pin بيستخدمه driver تاني بيدمر الـ function. لازم دايماً تعمل hog للـ pins اللي عايزها تفضل في function معينة.

---

### السيناريو 3: SPI Flash مش بيتقرأ على Industrial Gateway بـ Amlogic S905X

#### العنوان
**SPI NOR flash على Amlogic S905X industrial gateway بيفشل كل عملية read بعد warm reboot**

#### السياق
industrial gateway بيستخدم Amlogic S905X مع SPI NOR flash خارجي (W25Q128) عشان يخزن configuration data. بعد cold boot كل حاجة شغالة. بعد `reboot` command، الـ SPI flash مش بيتعرف، والـ driver بيطلع:
```
spi-nor spi0.0: unrecognized JEDEC id bytes: ff ff ff
```

#### المشكلة
بعد الـ reboot، الـ SPI pins بترجع لـ GPIO mode بدل SPI function بسبب ترتيب الـ driver init.

#### التحليل
عند الـ warm reboot، الـ kernel بيعمل `pinctrl_put` على الـ states القديمة. لما الـ SPI controller بيتعمل probe تاني، لازم يطلب الـ pins من جديد. لو فيه driver تاني بيتعمل init قبله وبيطلب نفس الـ pins كـ GPIO (حتى مؤقتاً)، الـ `meson8_pmx_request_gpio` بيعمل clear لـ bit الـ SPI:

```c
/* في meson8_pmx_disable_other_groups */
regmap_update_bits(pc->reg_mux,
                   pmx_data->reg * 4,
                   BIT(pmx_data->bit), 0);
/* ده بيعمل clear لـ bit الـ SPI في MUX register */
```

الـ `struct meson8_pmx_data` بيحدد بالضبط أي register وأي bit:

```c
struct meson8_pmx_data {
    bool is_gpio;      /* false للـ SPI group */
    unsigned int reg;  /* رقم الـ register */
    unsigned int bit;  /* رقم الـ bit */
};
```

الـ `GROUP` macro بيبني الـ data دي:
```c
#define GROUP(grp, r, b)   \
    {                       \
        .name = #grp,       \
        .pins = grp##_pins, \
        .num_pins = ARRAY_SIZE(grp##_pins), \
        .data = (const struct meson8_pmx_data[]){ \
            PMX_DATA(r, b, false),  /* is_gpio = false */ \
        },                  \
    }
```

لو فيه GPIO expander أو LED driver اتحمل قبل SPI وبيستخدم CS pin كـ GPIO output مؤقتاً، بيكسر الـ SPI mux.

#### الحل
**1- تشخيص:**
```bash
# شوف ترتيب الـ probe
dmesg | grep -E "(spi|gpio|pinctrl)" | head -50

# شوف الـ mux register مباشرة بعد reboot
devmem2 0xc1109880 w
devmem2 0xc1109884 w
```

**2- تعديل الـ DT عشان يحدد dependencies:**
```dts
&spi_0 {
    pinctrl-0 = <&spi_pins>;
    pinctrl-names = "default";
    status = "okay";
    /* اضمن إن الـ CS pin مش معرّف في أي gpio-hog */
};

/* امنع الـ LED driver من استخدام نفس الـ pin */
leds {
    compatible = "gpio-leds";
    /* استخدم pin مختلف */
};
```

**3- استخدام pinctrl sleep state:**
```dts
&spi_0 {
    pinctrl-0 = <&spi_pins>;
    pinctrl-1 = <&spi_pins_sleep>;
    pinctrl-names = "default", "sleep";
};
```

#### الدرس المستفاد
الـ `meson8_pmx_set_mux` محمي من الـ glitches لأنه بيعطل الـ groups التانية قبل ما يفعّل الـ target group. لكن الترتيب الزمني للـ driver probe مهم جداً. لازم تتأكد إن مفيش driver بيعمل `gpio_request` على الـ SPI pins في أي مرحلة من الـ boot.

---

### السيناريو 4: HDMI Hot-Plug Detection مش شغال على Set-Top Box بـ Amlogic Meson8

#### العنوان
**HDMI HPD interrupt مش بيتطلق على Amlogic Meson8 set-top box**

#### السياق
set-top box بيستخدم Amlogic Meson8 (S802). الـ HDMI hot-plug detection (HPD) المفروض يطلق interrupt لما تتوصل شاشة. الـ HDMI display بيشتغل زين، لكن disconnect/reconnect بيحتاج reboot عشان الـ display manager يشوفه.

#### المشكلة
الـ HPD pin معرّف في DT كـ GPIO interrupt. لكن الـ pin مش شغال كـ GPIO لأن الـ mux bit بتاعه محتل من function تانية.

#### التحليل
الـ HPD pin في Meson8 ممكن يشتغل في أكتر من mode. لما الـ HDMI driver بيعمل `gpio_request` على الـ HPD pin عشان يقرأه كـ interrupt:

```c
/* في meson8_pmx_request_gpio */
static int meson8_pmx_request_gpio(struct pinctrl_dev *pcdev,
                                    struct pinctrl_gpio_range *range,
                                    unsigned offset)
{
    struct meson_pinctrl *pc = pinctrl_dev_get_drvdata(pcdev);
    meson8_pmx_disable_other_groups(pc, offset, -1);
    return 0;
}
```

المشكلة إن `meson8_pmx_disable_other_groups` بيمشي على كل الـ groups وبيعمل clear لكل bit بيستخدم الـ pin، بما فيهم الـ HDMI DDC group اللي ممكن يكون share بعض الـ pins مع HPD في الـ Meson8 pinmux table.

الـ `GPIO_GROUP` macro بيحدد إن `is_gpio = true`:
```c
#define GPIO_GROUP(gpio)  \
    {                      \
        .name = #gpio,     \
        .pins = (const unsigned int[]){ gpio }, \
        .num_pins = 1,     \
        .data = (const struct meson8_pmx_data[]){ \
            PMX_DATA(0, 0, true),  /* is_gpio = true */ \
        },                 \
    }
```

الـ groups اللي `is_gpio = true` بيتخطاهم في `meson8_pmx_disable_other_groups`:
```c
if (pmx_data->is_gpio || i == sel_group)
    continue;  /* الـ GPIO groups بتتخطى */
```

لكن الـ non-GPIO groups اللي بتشارك الـ pin بتتعطل. لو الـ HPD pin بيشارك register مع HDMI DDC I2C، الـ DDC بيتعطل وده بيمنع الـ EDID reading.

#### الحل
**1- تشخيص:**
```bash
# شوف إيه الـ groups اللي بتستخدم HPD pin
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep -i "hpd\|hdmi"

# enable dynamic debug
echo "module pinctrl_meson8 +p" > /sys/kernel/debug/dynamic_debug/control
dmesg | grep "enable function"
```

**2- تعديل DT عشان الـ HPD يكون independent:**
```dts
&hdmi_tx {
    pinctrl-0 = <&hdmi_hpd_pins &hdmi_ddc_pins>;
    pinctrl-names = "default";

    /* HPD كـ interrupt */
    hpd-gpios = <&gpio GPIOH_0 GPIO_ACTIVE_HIGH>;
};
```

**3- التأكد إن الـ HDMI DDC I2C مش بيشارك pins مع HPD في الـ DT:**
```dts
/* DDC على I2C منفصل */
&i2c_A {
    pinctrl-0 = <&i2c_a_pins>;
    /* pins مختلفة تماماً عن HPD */
};
```

#### الدرس المستفاد
الـ `is_gpio` flag في `struct meson8_pmx_data` هو الفرق بين group بيتعطل و group بيتخطى. الـ GPIO groups دايماً بتتخطى عند الـ disable. لازم تفهم الـ Meson8 pinmux table كويس عشان تعرف أي الـ functional groups بتشارك pins مع الـ HPD.

---

### السيناريو 5: Custom Board Bring-Up — كل الـ Peripherals فشلت على Amlogic Meson8b

#### العنوان
**Bring-up لـ custom industrial board بـ Amlogic S812 — UART، I2C، SPI كلهم فاشلين**

#### السياق
hardware engineer سلّم بورد جديدة بـ Amlogic S812 (Meson8b) لـ embedded Linux engineer عشان يعمل bring-up. البورد فيها: UART0 للـ console، I2C1 لـ EEPROM، SPI0 لـ flash. الـ engineer نقل الـ DT من reference design قديم. كل الـ drivers بتظهر في dmesg بس مفيش peripheral بيشتغل.

#### المشكلة
الـ DT القديم مكتوب غلط — الـ `reg` و `bit` values في الـ GROUP macros مش مطابقة للـ Meson8b pinmux registers.

#### التحليل
الـ `meson8_pmx_set_mux` بيكتب في الـ register بناء على `pmx_data->reg` و `pmx_data->bit`:

```c
ret = regmap_update_bits(pc->reg_mux, pmx_data->reg * 4,
                         BIT(pmx_data->bit),
                         BIT(pmx_data->bit));
```

الـ `pmx_data` جاية من الـ `GROUP` macro:
```c
#define GROUP(grp, r, b)  \
    .data = (const struct meson8_pmx_data[]){ \
        PMX_DATA(r, b, false),  /* r = register offset, b = bit number */ \
    },
```

الـ `PMX_DATA` macro:
```c
#define PMX_DATA(r, b, g)  \
    {                       \
        .reg = r,           \
        .bit = b,           \
        .is_gpio = g,       \
    }
```

لو الـ `r` أو `b` غلط، الـ `regmap_update_bits` بيكتب في register غلط أو bit غلط. النتيجة: الـ peripheral المطلوب مش بيتفعّل، وممكن كمان يتفعّل peripheral تاني بالغلط (side effect).

**مثال على الـ bug:**
```c
/* غلط: r=2, b=4 بينما المفروض r=1, b=8 للـ UART0 في Meson8b */
GROUP(uart_0, 2, 4)  /* بيكتب في register تاني تماماً */

/* صح */
GROUP(uart_0, 1, 8)  /* الـ register والـ bit الصح من datasheet */
```

#### الحل
**1- تشخيص:**
```bash
# قارن الـ expected registers بالـ actual
# Meson8b UART0 MUX bit: register 1, bit 8 (من الـ datasheet)
devmem2 0xc1109884 w   # اقرأ MUX register 1

# شوف إيه اللي الـ driver بيكتبه
echo "file pinctrl-meson8-pmx.c +p" > /sys/kernel/debug/dynamic_debug/control
dmesg | grep "enable function"
```

**2- فتح الـ datasheet وتصحيح الـ GROUP definitions:**

```c
/* من pinctrl-meson8b.c — المفروض يكون صح */

/* UART0 على GPIOX_12 (TX) و GPIOX_13 (RX) */
static const unsigned int uart_tx_a_pins[] = { PIN(GPIOX_12, EE_OFF) };
static const unsigned int uart_rx_a_pins[] = { PIN(GPIOX_13, EE_OFF) };

/* من Meson8b datasheet: UART_A_TX على register 4, bit 24 */
static const struct meson_pmx_group meson8b_groups[] = {
    /* ... */
    GROUP(uart_tx_a, 4, 24),  /* r=4, b=24 — مطابق للـ datasheet */
    GROUP(uart_rx_a, 4, 23),  /* r=4, b=23 */
    /* ... */
};
```

**3- verification script:**
```bash
#!/bin/bash
# تحقق إن كل peripheral شغال

# UART
echo "Testing UART..."
echo "hello" > /dev/ttyAML0

# I2C
echo "Testing I2C..."
i2cdetect -y 1

# SPI
echo "Testing SPI..."
spi-test -D /dev/spidev0.0 -p "hello"
```

**4- قراءة الـ MUX registers بعد الـ fix:**
```bash
# اقرأ كل MUX registers
for i in 0 4 8 12 16 20; do
    addr=$((0xc1109880 + i))
    printf "MUX[%02d]: 0x%08x\n" $((i/4)) $(devmem2 $addr w | awk '/Read/ {print $NF}')
done
```

#### الدرس المستفاد
الـ `meson8_pmx_set_mux` بيثق بشكل كامل في الـ `reg` و `bit` اللي جايين من الـ `GROUP` macro. مفيش أي validation في الوقت الفعلي. لازم في أي bring-up جديد تفتح الـ Meson SoC datasheet وتتحقق يدوياً إن كل `GROUP(name, r, b)` مطابق للـ pinmux register table. الـ `regmap_update_bits` مع قيم غلط بيكتب في addresses عشوائية وممكن يكسر hardware تاني.
## Phase 7: مصادر ومراجع

### مصادر LWN.net

**الـ** LWN.net هو المرجع الأساسي لمتابعة تطور الـ Linux kernel، وفيما يلي أهم المقالات المتعلقة بالـ pinctrl subsystem والـ Meson SoCs:

| المقال | الوصف |
|--------|-------|
| [The pin control subsystem](https://lwn.net/Articles/468759/) | المقال التأسيسي الذي شرح فيه Linus Walleij فكرة الـ pinctrl subsystem لأول مرة |
| [drivers: create a pin control subsystem](https://lwn.net/Articles/463335/) | الـ patchset الأولي لإنشاء الـ pinctrl subsystem في الـ kernel |
| [pin controller subsystem v7](https://lwn.net/Articles/459190/) | النسخة السابعة من الـ patchset — تُظهر مسار تطور التصميم |
| [Documentation/pinctrl.txt](https://lwn.net/Articles/465077/) | أول نسخة من التوثيق الرسمي للـ pinctrl داخل الـ kernel |
| [pinctrl: add a pin config interface](https://lwn.net/Articles/471826/) | إضافة الـ pin configuration (pull-up, drive strength) فوق الـ pinmux |
| [pinctrl: add a generic pin config interface](https://lwn.net/Articles/467269/) | الـ generic pin config API التي يعتمد عليها كل driver لاحقاً |
| [Pinctrl driver for Amlogic Meson SoCs](https://lwn.net/Articles/616225/) | الـ patchset الأولي لدعم Meson SoCs — الجذر التاريخي لملفنا |
| [Amlogic Meson pinctrl driver](https://lwn.net/Articles/620822/) | النسخة المكتملة من الـ Meson pinctrl driver مع البنية التحتية المشتركة لكل SoCs |
| [pinctrl: meson-a1: add pinctrl driver](https://lwn.net/Articles/804174/) | امتداد الـ infrastructure إلى جيل أحدث (Meson-A1) |
| [pinctrl: meson-s4: add pinctrl driver](https://lwn.net/Articles/879912/) | دعم Meson-S4 — يُثبت استدامة التصميم الأصلي |
| [irqchip: meson: add support for the gpio interrupt controller](https://lwn.net/Articles/703946/) | إضافة GPIO interrupt controller مرتبطة مباشرة بالـ pinctrl data |
| [pinctrl: introduce the concept of a GPIO pin function category](https://lwn.net/Articles/1031226/) | أحدث تطور في الـ pinctrl API |

---

### التوثيق الرسمي داخل الـ kernel

```
Documentation/driver-api/pin-control.rst        # المرجع الرئيسي للـ pinctrl subsystem
Documentation/devicetree/bindings/pinctrl/      # كل bindings للـ pinctrl drivers
Documentation/devicetree/bindings/pinctrl/amlogic,meson-pinctrl.yaml  # binding الـ Meson
```

**الـ** online version على docs.kernel.org:

- [PINCTRL (PIN CONTROL) subsystem — The Linux Kernel documentation](https://docs.kernel.org/driver-api/pin-control.html)
- [PINCTRL v4.13 — kernel.org](https://www.kernel.org/doc/html/v4.13/driver-api/pinctl.html)

---

### الـ source code في الـ kernel الرسمي

الملفات الأساسية في `torvalds/linux`:

```
drivers/pinctrl/meson/pinctrl-meson8-pmx.c   # الـ pinmux logic للجيل الأول
drivers/pinctrl/meson/pinctrl-meson8-pmx.h   # struct meson8_pmx_data والـ macros
drivers/pinctrl/meson/pinctrl-meson8.c        # تعريف الـ pins والـ groups والـ banks
drivers/pinctrl/meson/pinctrl-meson.c         # البنية التحتية المشتركة لكل Meson SoCs
drivers/pinctrl/meson/pinctrl-meson.h         # struct meson_pinctrl وكل الـ shared types
include/linux/pinctrl/pinmux.h                # struct pinmux_ops — الـ interface الذي يُنفذه ملفنا
include/linux/pinctrl/pinctrl.h               # الـ pinctrl core API
```

**الـ** GitHub links للملفات مباشرة:

- [pinctrl-meson8.c على GitHub](https://github.com/torvalds/linux/blob/master/drivers/pinctrl/meson/pinctrl-meson8.c)
- [pinctrl-meson8-pmx.c على GitHub](https://github.com/intel/mainline-tracking/blob/master/drivers/pinctrl/meson/pinctrl-meson8-pmx.c)

---

### مناقشات الـ mailing list

| الرابط | الموضوع |
|--------|---------|
| [PATCH 1/3: pinctrl: add driver for Amlogic Meson SoCs](https://lkml.iu.edu/hypermail/linux/kernel/1410.0/04163.html) | الإرسال الأول للـ driver على LKML (أكتوبر 2014) — بقلم Beniamino Galvani |
| [PATCH v3 0/3: Amlogic Meson pinctrl driver](https://lkml.iu.edu/hypermail/linux/kernel/1411.2/00173.html) | النسخة الثالثة والأخيرة قبل الدمج في الـ mainline |
| [Re: PATCH RFC 2/3: pinctrl: Add driver support for Amlogic SoCs](https://www.spinics.net/lists/linux-gpio/msg107463.html) | نقاش على قائمة linux-gpio حول الـ RFC الأولي |
| [pinctrl: meson: allow gpio to request irq — Patchwork](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1476871709-8359-5-git-send-email-jbrunet@baylibre.com/) | Jerome Brunet يضيف دعم الـ GPIO IRQ (2016) |
| [pinctrl: meson: update pinctrl data with gpio irq data](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg1252733.html) | تحديث الـ pinctrl data لدعم الـ GPIO interrupts |

**الـ** Patchwork لمتابعة الـ patches الحالية:

- [linux-amlogic project on Patchwork](https://patchwork.kernel.org/project/linux-amlogic/)

---

### موقع linux-meson.com

الموقع الرسمي لمجتمع تطوير Linux على Amlogic Meson SoCs:

- [Linux for Amlogic Meson — الصفحة الرئيسية](https://linux-meson.com/)
- [Kernel Mainlining Progress](https://linux-meson.com/mainlining.html) — يتتبع حالة دعم كل hardware component بما فيها الـ pinctrl
- [Hardware Support](https://linux-meson.com/hardware.html)

---

### kernelnewbies.org

صفحات تتبع التغييرات في الـ pinctrl عبر إصدارات الـ kernel:

| الإصدار | الرابط |
|---------|--------|
| Linux 6.15 | [kernelnewbies.org/Linux_6.15](https://kernelnewbies.org/Linux_6.15) — أضاف Amlogic pinctrl driver جديد |
| Linux 6.17 | [kernelnewbies.org/Linux_6.17](https://kernelnewbies.org/Linux_6.17) |
| Linux 6.18 | [kernelnewbies.org/Linux_6.18](https://kernelnewbies.org/Linux_6.18) |
| Linux 4.9  | [kernelnewbies.org/Linux_4.9](https://kernelnewbies.org/Linux_4.9) |

---

### elinux.org

- [Introduction to pin muxing and GPIO control under Linux (PDF)](https://elinux.org/images/a/a7/ELC-2021_Introduction_to_pin_muxing_and_GPIO_control_under_Linux.pdf) — عرض من ELC 2021 يشرح الـ pinmux من الأساس
- [Linux Kernel Resources](https://elinux.org/Linux_Kernel_Resources) — روابط عامة لمصادر الـ kernel

---

### الكتب المرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
```
Chapter 14: The Linux Device Model
```
**الـ** LDD3 متاح مجاناً على: https://lwn.net/Kernel/LDD3/

يشرح الـ `struct device`، الـ `bus_type`، والـ `driver model` الذي يقوم عليه الـ pinctrl core.

#### Linux Kernel Development — Robert Love (3rd Edition)
```
Chapter 17: Devices and Modules    # يشرح الـ device model والـ driver registration
Chapter 14: The Block I/O Layer    # مفيد لفهم الـ regmap pattern
```

المفاهيم المباشرة لملفنا:
- `platform_driver` registration
- `module_init` / `EXPORT_SYMBOL_GPL`
- الـ `kobject` hierarchy

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
```
Chapter 15: Customizing the Kernel  # ذو صلة بالـ Kconfig وتفعيل الـ pinctrl
Chapter 16: Using Open Source Tools
```

يُفيد في فهم كيفية اختيار الـ `CONFIG_PINCTRL_MESON8` في الـ defconfig للبورد.

#### Mastering Embedded Linux Programming — Frank Vasquez & Chris Simmonds
```
Chapter 11: Interfacing with Device Drivers
```
يتناول الـ Device Tree والـ GPIO/pinctrl من منظور عملي على Yocto.

---

### الـ Kconfig entry للـ driver

```
CONFIG_PINCTRL_MESON8
```

- [Linux Kernel Driver DataBase: CONFIG_PINCTRL_MESON8](https://cateee.net/lkddb/web-lkddb/PINCTRL_MESON8.html)

يُستخدم لتفعيل الـ driver في الـ build:

```bash
# في menuconfig:
Device Drivers → Pin controllers → Amlogic Meson 8 pin controller
```

---

### search terms للبحث عن المزيد

استخدم هذه الكلمات المفتاحية للوصول لمزيد من المعلومات:

```
# على LWN.net
site:lwn.net "pinctrl" "meson"
site:lwn.net "pinmux" "amlogic"

# على lore.kernel.org
"pinctrl: meson" author:b.galvani@gmail.com
"pinctrl: meson" author:jbrunet@baylibre.com

# على Patchwork
project:linux-amlogic pinctrl meson8

# على GitHub
org:torvalds path:drivers/pinctrl/meson

# بحث عام
"meson8_pmx_ops" site:github.com
"meson8_pmx_data" linux kernel
"pinctrl-meson8-pmx" amlogic driver
"regmap_update_bits" pinctrl meson
```

---

### ملخص المراجع الأهم

| الأولوية | المرجع | لماذا؟ |
|----------|--------|--------|
| 1 | [docs.kernel.org/driver-api/pin-control.html](https://docs.kernel.org/driver-api/pin-control.html) | التوثيق الرسمي الكامل للـ pinctrl API |
| 2 | [lwn.net/Articles/468759](https://lwn.net/Articles/468759/) | المقال التأسيسي لفهم فلسفة الـ subsystem |
| 3 | [lwn.net/Articles/620822](https://lwn.net/Articles/620822/) | الـ Meson driver نفسه شُرح فيه |
| 4 | [linux-meson.com/mainlining.html](https://linux-meson.com/mainlining.html) | متابعة حالة الـ mainlining للـ Meson hardware |
| 5 | [lkml.iu.edu — PATCH 1/3](https://lkml.iu.edu/hypermail/linux/kernel/1410.0/04163.html) | الإرسال الأصلي للـ driver |
## Phase 8: Writing simple module

### الفكرة

**الـ** `meson8_pmx_set_mux` هي الدالة الأهم في الملف — بتتسمى كل ما الـ pinctrl core محتاج يحوّل pin من function لـ function تانية (مثلاً من UART لـ GPIO). هي `EXPORT_SYMBOL_GPL` مش موجودة مباشرة، لكن الـ `meson8_pmx_ops` struct اللي بيحتوي عليها هو المُصدَّر. الطريقة الأنسب للـ hook هنا هي **kprobe** على `meson8_pmx_set_mux` نفسها — بدون تعديل أي كود kernel، بنقدر نراقب كل عملية mux switching في real time.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * kprobe module: hook meson8_pmx_set_mux
 *
 * Intercepts every pinmux switch on Amlogic Meson8 SoCs and logs
 * the function selector and group selector to the kernel log.
 */

#include <linux/module.h>       /* MODULE_*, module_init/exit          */
#include <linux/kernel.h>       /* pr_info                             */
#include <linux/kprobes.h>      /* kprobe, register_kprobe, ...        */
#include <linux/pinctrl/pinctrl.h> /* pinctrl_dev_get_drvdata, pinctrl_dev */

/* ------------------------------------------------------------------
 * pre-handler: runs just BEFORE meson8_pmx_set_mux executes.
 *
 * Signature of the hooked function:
 *   int meson8_pmx_set_mux(struct pinctrl_dev *pcdev,
 *                           unsigned func_num,
 *                           unsigned group_num);
 *
 * The kernel calling convention on ARM (and most arches) passes the
 * first arguments in registers; pt_regs gives us access to them.
 * ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * regs->regs[0] → pcdev  (struct pinctrl_dev *)
     * regs->regs[1] → func_num
     * regs->regs[2] → group_num
     *
     * On x86_64:
     *   regs->di → pcdev
     *   regs->si → func_num
     *   regs->dx → group_num
     *
     * We use a generic approach: read the second and third arguments.
     * Adjust register names if cross-compiling for a different arch.
     */

#if defined(CONFIG_ARM) || defined(CONFIG_ARM64)
    unsigned int func_num  = (unsigned int)regs->regs[1];
    unsigned int group_num = (unsigned int)regs->regs[2];
#elif defined(CONFIG_X86_64)
    unsigned int func_num  = (unsigned int)regs->si;
    unsigned int group_num = (unsigned int)regs->dx;
#else
    /* fallback: just print that we were called */
    unsigned int func_num  = 0xDEAD;
    unsigned int group_num = 0xBEEF;
#endif

    pr_info("[meson8_pmx_probe] set_mux called: func=%u group=%u\n",
            func_num, group_num);

    return 0; /* 0 = let the real function continue normally */
}

/* ------------------------------------------------------------------
 * post-handler: runs just AFTER meson8_pmx_set_mux returns.
 * We use it to log the return value (stored in regs->ax / regs[0]).
 * ------------------------------------------------------------------ */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
#if defined(CONFIG_ARM) || defined(CONFIG_ARM64)
    long ret = (long)regs->regs[0];
#elif defined(CONFIG_X86_64)
    long ret = (long)regs->ax;
#else
    long ret = 0;
#endif

    pr_info("[meson8_pmx_probe] set_mux returned: %ld (%s)\n",
            ret, ret == 0 ? "OK" : "ERROR");
}

/* ------------------------------------------------------------------
 * kprobe descriptor — one struct per hooked symbol
 * ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "meson8_pmx_set_mux", /* exact kernel symbol name  */
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

/* ------------------------------------------------------------------
 * module_init: يسجّل الـ kprobe في الـ kernel
 * ------------------------------------------------------------------ */
static int __init meson8_pmx_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[meson8_pmx_probe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[meson8_pmx_probe] planted at %s (%p)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ------------------------------------------------------------------
 * module_exit: يشيل الـ kprobe عشان منحجبش أي call بعد الـ unload
 * ------------------------------------------------------------------ */
static void __exit meson8_pmx_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("[meson8_pmx_probe] removed from %s\n", kp.symbol_name);
}

module_init(meson8_pmx_probe_init);
module_exit(meson8_pmx_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe hook on meson8_pmx_set_mux — logs every pinmux switch");
```

---

### Makefile للبناء

```makefile
obj-m += meson8_pmx_probe.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|---|---|
| `linux/module.h` | ماكرو `module_init`, `module_exit`, `MODULE_*` |
| `linux/kernel.h` | `pr_info`, `pr_err` |
| `linux/kprobes.h` | `struct kprobe`, `register_kprobe`, `unregister_kprobe` |
| `linux/pinctrl/pinctrl.h` | تعريف `struct pinctrl_dev` عشان نفهم الـ arguments |

---

#### الـ `handler_pre` — ليه؟

**الـ** `pre_handler` بيتنفذ قبل ما الدالة الأصلية تشتغل، يعني بنشوف الـ arguments قبل أي تعديل. ده مهم لأن `meson8_pmx_set_mux` بتعدّل registers الـ hardware مباشرة عبر `regmap_update_bits`، فبنعرف أي function وأي group طُلب قبل ما الـ bit يتكتب فعلاً.

الـ `pt_regs` هو snapshot من registers الـ CPU لحظة الـ probe hit — منه بنستخرج الـ arguments الأول والتاني والتالت حسب calling convention الـ arch (ARM64 بيحط الـ args في `regs[0..n]`، x86_64 في `di/si/dx`).

---

#### الـ `handler_post` — ليه؟

**الـ** `post_handler` بيتنفذ بعد رجوع الدالة، والـ return value بيبقى في `regs[0]` (ARM64) أو `regs->ax` (x86). ده بيخلينا نعرف لو الـ mux switch نجح أو فشل بدون ما نغيّر أي حاجة في الـ kernel — مفيد جداً في debugging حالات الـ regmap failure.

---

#### الـ `struct kprobe kp`

**الـ** `symbol_name` هو الاسم بالظبط زي ما موجود في `/proc/kallsyms`. الـ kernel بيحوّله لـ address وقت الـ `register_kprobe`. لو الـ symbol مش موجود (مثلاً الـ driver مش مبني) الـ register هيرجّع `-ENOENT`.

---

#### الـ `module_init` — ليه؟

**الـ** `register_kprobe` بتزرع software breakpoint (INT3 على x86 أو BRK على ARM64) في الـ kernel text عند عنوان الدالة. أي call لـ `meson8_pmx_set_mux` بعدها هيتوقف عند الـ breakpoint ويشغّل handlers احنا كتبناهم.

---

#### الـ `module_exit` — ليه؟

**الـ** `unregister_kprobe` بيشيل الـ breakpoint ويرجّع الـ original instruction. لازم يتعمل في الـ exit عشان لو الـ module اتخلع والـ handler code اتحذف من الذاكرة، أي call تالية للدالة هتـ crash الـ kernel بـ bad address.

---

### كيفية الاختبار

```bash
# بناء وتحميل الـ module
make
sudo insmod meson8_pmx_probe.ko

# شوف اللوج (على جهاز Meson8 حقيقي أو QEMU)
sudo dmesg | grep meson8_pmx_probe

# أي عملية pinmux (مثلاً تشغيل UART أو تغيير GPIO direction) هتطلع في dmesg:
# [meson8_pmx_probe] set_mux called: func=1 group=3
# [meson8_pmx_probe] set_mux returned: 0 (OK)

# إزالة الـ module
sudo rmmod meson8_pmx_probe
sudo dmesg | grep meson8_pmx_probe
# [meson8_pmx_probe] removed from meson8_pmx_set_mux
```

> **ملاحظة:** الـ kprobe على `meson8_pmx_set_mux` بيشتغل فقط لو الـ `pinctrl-meson8-pmx` driver مبني كـ module أو built-in وموجود الـ symbol في `/proc/kallsyms`. افحص أولاً: `grep meson8_pmx_set_mux /proc/kallsyms`
