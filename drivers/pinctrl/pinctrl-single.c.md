## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي الملف ده جزء منه

الملف `pinctrl-single.c` هو **driver** جنيرك تابع لـ **PIN CONTROLLER** subsystem في الـ Linux kernel. المسؤولون عنه هما Tony Lindgren وHaojian Zhuang، وبيخدم بشكل أساسي معالجات TI OMAP وAM3x/4x/5x/DRA7 وAM65x وJ7200، وكمان Marvell PXA1908.

---

### القصة من الأول: إيه المشكلة اللي بيحلها؟

تخيل عندك شريحة إلكترونية (SoC) فيها 200 طرف (pin) على الحافة بتاعتها. كل pin ممكن يشتغل بأكتر من وظيفة:

```
Pin رقم 5:
  ├── وظيفة A: UART TX (إرسال سيريال)
  ├── وظيفة B: SPI CLK (ساعة SPI)
  ├── وظيفة C: GPIO عادي
  └── وظيفة D: I2C SCL
```

الـ SoC بيتحكم في الاختيار ده عن طريق **register** واحد لكل pin (أو مجموعة pins). بتكتب قيمة معينة في الـ register ده، والـ pin بيتحول لوظيفة معينة.

**المشكلة:** كل SoC مختلف في:
- عرض الـ register (8 bit؟ 16؟ 32؟)
- أنهي bits هي اللي بتحدد الوظيفة
- عدد الـ pins في كل register

لو كتبنا driver خاص لكل SoC، هنحتاج عشرات الـ drivers المتشابهين. هدر تام!

---

### الحل: الـ `pinctrl-single` driver

**الفكرة الجوهرية:** لو كل pin عنده register واحد بيتحكم في وظيفته، يبقى ممكن نعمل driver واحد جنيرك يقرأ التفاصيل من الـ Device Tree ويشتغل على أي SoC بيتبع النمط ده.

```
Device Tree (DTS)         pinctrl-single driver           Hardware
─────────────────         ──────────────────────         ─────────
 register-width = 32  ──►  اقرأ الإعدادات         ──►  اكتب في الـ
 function-mask = 0x7  ──►  جهز جدول الـ pins              registers
 pins = <0x100 0x1>   ──►  نفذ الـ mux                   مباشرة
```

الـ DTS هو اللي بيقول للـ driver: "الـ register عرضه 32-bit، وbits 0-2 هي اللي بتحدد الوظيفة". الـ driver بيقرأ الكلام ده وينفذه بشكل جنيرك.

---

### ليه هو مفيد؟

| المشكلة | الحل |
|---------|------|
| عشرات SoC كلهم بنفس النمط (register per pin) | driver واحد جنيرك |
| كل SoC عنده عرض register مختلف | يقرأ `pinctrl-single,register-width` من DTS |
| التفريق بين GPIO و functions مختلفة | يقرأ `pinctrl-single,function-mask` من DTS |
| إضافة SoC جديد | بس تكتب DTS جديدة، من غير كود C |

---

### إيه اللي الملف بيعمله تحديدًا؟

الملف ده بيقدم **ثلاث خدمات أساسية** للكيرنل:

#### 1. الـ Pin Mux (تحديد وظيفة الـ pin)
لما تيجي تشغل UART مثلًا، الكيرنل بيطلب من الـ driver يحوّل الـ pin من GPIO لـ UART TX. الـ driver بيكتب القيمة الصح في الـ register المحدد في الـ DTS.

```c
/* الكود بيعمل ده بالضبط في pcs_set_mux() */
val = pcs->read(vals->reg);      /* اقرأ القيمة الحالية */
val &= ~mask;                    /* امسح bits الوظيفة */
val |= (vals->val & mask);       /* ضع القيمة الجديدة */
pcs->write(val, vals->reg);      /* اكتبها في الـ register */
```

#### 2. الـ Pin Configuration (إعدادات الـ pin)
غير الـ mux، كل pin ممكن يتضبط:
- **Pull-up / Pull-down** (مقاومة شد للأعلى أو الأسفل)
- **Drive strength** (قوة الإشارة)
- **Slew rate** (سرعة تغيير الإشارة)
- **Input Schmitt trigger** (تنعيم الإشارة)

#### 3. الـ Wake-up Interrupts (التنبيه من النوم)
بعض الـ pins ممكن يصحّوا الـ SoC من وضع الـ sleep لما يجي إشارة عليهم. الـ driver بيدعم ده عن طريق **irq_domain** مبني على الـ pin registers نفسها.

```
[SoC نايم] → [إشارة على pin] → [interrupt في register] → [الكيرنل يصحى]
```

---

### قصة الـ suspend/resume (مهمة جدًا لـ OMAP)

معالجات OMAP بتفقد محتوى الـ pin registers لما الجهاز يدخل وضع الـ deep sleep (power context loss). الـ driver عنده حل:

```
قبل النوم (suspend):
  pcs_save_context() → احفظ كل قيم الـ registers في الـ RAM

بعد الصحيان (resume):
  pcs_restore_context() → استعد القيم المحفوظة وارجع اكتبها
```

الـ flag `PCS_CONTEXT_LOSS_OFF` هو اللي بيفعّل السلوك ده.

---

### بنية الـ Driver من الداخل

```
pcs_probe()
├── اقرأ إعدادات DTS (register-width, function-mask, ...)
├── map الـ registers في الـ memory
├── pcs_allocate_pin_table()   → سجّل كل الـ pins في الـ framework
├── devm_pinctrl_register_and_init() → سجّل الـ driver في الـ pinctrl core
├── pcs_add_gpio_func()        → سجّل نطاقات GPIO
└── pcs_irq_init_chained_handler() → اعمل irq domain (لو في interrupt)

pcs_dt_node_to_map()           ← الكيرنل بيطلبها لما يعمل mux
├── pcs_parse_one_pinctrl_entry()   (لـ pin واحد per register)
└── pcs_parse_bits_in_pinctrl_entry() (لـ multi-pin per register)
```

---

### الـ SoCs المدعومة

| الـ compatible string | الـ SoC |
|----------------------|---------|
| `ti,omap3-padconf` | OMAP3 |
| `ti,omap4-padconf` | OMAP4 |
| `ti,omap5-padconf` | OMAP5 |
| `ti,am437-padconf` | AM437x |
| `ti,am654-padconf` | AM65x |
| `ti,dra7-padconf` | DRA7 |
| `ti,j7200-padconf` | J7200 |
| `marvell,pxa1908-padconf` | PXA1908 |
| `pinctrl-single` | جنيرك |
| `pinconf-single` | جنيرك مع pinconf |

---

### الملفات اللي لازم تعرفها

#### الملف الأساسي
| الملف | الدور |
|-------|-------|
| `drivers/pinctrl/pinctrl-single.c` | الـ driver الجنيرك كله |

#### الـ pinctrl framework (اللي الملف بيبني فوقيه)
| الملف | الدور |
|-------|-------|
| `drivers/pinctrl/core.c` | الـ pinctrl core، بيدير كل الـ drivers |
| `drivers/pinctrl/core.h` | تعريفات داخلية للـ core |
| `drivers/pinctrl/pinmux.c` | منطق الـ mux الجنيرك |
| `drivers/pinctrl/pinconf.c` | منطق الـ pin configuration |
| `drivers/pinctrl/pinconf-generic.c` | تنفيذ جنيرك لـ pinconf |
| `drivers/pinctrl/devicetree.c` | parsing الـ DTS للـ pinctrl |
| `drivers/pinctrl/pinctrl-generic.c` | helper functions جنيرك |

#### الـ Headers الأساسية
| الملف | الدور |
|-------|-------|
| `include/linux/pinctrl/pinctrl.h` | الـ API الأساسية (structs وops) |
| `include/linux/pinctrl/pinmux.h` | الـ mux operations API |
| `include/linux/pinctrl/pinconf.h` | الـ config operations API |
| `include/linux/pinctrl/pinconf-generic.h` | الـ generic config params |
| `include/linux/platform_data/pinctrl-single.h` | platform data لـ OMAP (legacy) |

#### الـ DTS Bindings (الوثائق)
| الملف | الدور |
|-------|-------|
| `Documentation/devicetree/bindings/pinctrl/pinctrl-single.yaml` | توثيق الـ DTS properties |
## Phase 2: شرح الـ Pinctrl Framework

---

### المشكلة اللي بيحلها الـ Subsystem

أي SoC ARM فيه عشرات أو مئات الـ **physical pins**. كل pin ممكن يشتغل بأكتر من دور — UART، SPI، GPIO، I2C، USB، إلخ. ده بيتحكم فيه عن طريق **hardware registers** جوا الـ SoC بتكتب فيها قيمة معينة تحدد الـ function.

المشكلة القديمة إن كل driver كان بيعمل `ioremap` وبيكتب في الـ mux registers بنفسه بدون تنسيق مع حد تاني. النتيجة:
- **تعارض** بين drivers على نفس الـ pin
- **تكرار كود** مرعب في كل SoC vendor
- مفيش طريقة موحدة تعرف "إيه اللي شغال دلوقتي على الـ pin ده؟"
- مفيش integration مع الـ device tree بشكل structured

---

### الحل: الـ Pinctrl Subsystem

الكيرنل حل ده بـ **pinctrl subsystem** — طبقة abstraction موحدة بتعمل الآتي:

1. **تسجيل كل الـ pins** في الـ SoC كـ array واحدة
2. **تعريف الـ functions** (UART0، SPI1، إلخ) وكل function بتأثر على مجموعة pins = **pin group**
3. **تطبيق الـ configuration** (pinmux + pinconf) من خلال driver واحد بيتكلم مع الـ hardware
4. **استهلاك الـ configuration من الـ device tree** — كل device بيقول "أنا عايز pins معينة بـ function معينة"

**الـ pinctrl-single.c** هو driver خاص بالـ SoCs اللي عندها pattern موحد: **register واحد لكل pin** أو **عدة pins packed في register واحد**، وكل اللي بتعمله هو تكتب القيمة الصح في الـ offset الصح. TI OMAP، AM3/4/5، DRA7، AM65x كلها بتستخدمه.

---

### Analogy: لوحة المفاتيح في الاستوديو

تخيل **استوديو تسجيل** فيه لوحة تحكم (mixing board) بمئات الـ knobs والـ faders.

| عنصر الاستوديو | المقابل في الـ kernel |
|---|---|
| الـ physical knob على اللوحة | الـ physical pin على الـ SoC |
| دور الـ knob (bass, treble, volume) | الـ mux function (UART TX, GPIO, SPI CLK) |
| مجموعة knobs لقناة واحدة | الـ pin group |
| الـ track اللي بيستخدم القناة | الـ consumer device (مثلاً UART driver) |
| مهندس الصوت اللي بيربط كل حاجة | الـ pinctrl core |
| دليل اللوحة (manual) | الـ device tree node |
| قاعدة "مينفعش كانال يتوصل لأكتر من track" | منع conflict بين drivers |
| مهندس الصوت بيقرأ الـ manual ويحرك الـ knobs | `pinctrl-single.c` بيقرأ الـ DT وبيكتب في الـ registers |

الـ **mixing board** (الـ SoC register space) هي الـ hardware. الـ **مهندس** (pinctrl-single) بيعرف كيف يتعامل معها. الـ **tracks** (devices) بيقولوا "أنا محتاج القناة دي" والمهندس هو اللي بيربط.

---

### Big Picture Architecture

```
  ┌──────────────────────────────────────────────────────────────┐
  │                     Consumer Devices                         │
  │   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
  │   │  UART    │  │  SPI     │  │  I2C     │  │  GPIO    │   │
  │   │  driver  │  │  driver  │  │  driver  │  │  driver  │   │
  │   └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
  └────────┼─────────────┼─────────────┼──────────────┼─────────┘
           │             │             │              │
           │   "أنا محتاج pinctrl state default"     │
           ▼             ▼             ▼              ▼
  ┌──────────────────────────────────────────────────────────────┐
  │                   Pinctrl Core (core.c)                      │
  │                                                              │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
  │  │  pinctrl_map │  │  pin_desc[]  │  │  state machine   │   │
  │  │  (DT→config) │  │  (all pins)  │  │  (default/sleep) │   │
  │  └──────────────┘  └──────────────┘  └──────────────────┘   │
  │                                                              │
  │  Calls ops:  pctlops / pmxops / confops                      │
  └──────────────────────┬───────────────────────────────────────┘
                         │
                         ▼
  ┌──────────────────────────────────────────────────────────────┐
  │              pinctrl-single driver (pcs_device)              │
  │                                                              │
  │  ┌─────────────────┐  ┌───────────────┐  ┌───────────────┐  │
  │  │  pinctrl_ops    │  │  pinmux_ops   │  │  pinconf_ops  │  │
  │  │  dt_node_to_map │  │  set_mux      │  │  config_get   │  │
  │  │  get_groups     │  │  gpio_request │  │  config_set   │  │
  │  └─────────────────┘  └───────────────┘  └───────────────┘  │
  │                                                              │
  │  pcs_device {                                                │
  │    base, size, width, fmask, fshift                          │
  │    read() / write() function pointers                        │
  │    pins[], gpiofuncs[], irqs[]                               │
  │  }                                                           │
  └──────────────────────┬───────────────────────────────────────┘
                         │
                         │  ioremap'd register access
                         ▼
  ┌──────────────────────────────────────────────────────────────┐
  │           SoC Pad Control Registers (MMIO)                   │
  │                                                              │
  │  Offset 0x000: PIN_0 mux register  [bits 2:0] = function    │
  │  Offset 0x004: PIN_1 mux register  [bits 2:0] = function    │
  │  Offset 0x008: PIN_2 mux register  [bits 2:0] = function    │
  │  ...                                                         │
  │  Offset 0x1FC: PIN_127 mux register                         │
  └──────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction: التراتبية الثلاثية

الـ pinctrl subsystem مبني على **3 مستويات abstraction** مترابطة:

#### 1. Pin — الوحدة الأصغر

**الـ `struct pinctrl_pin_desc`** بتمثل pin واحد:

```c
struct pinctrl_pin_desc {
    unsigned int number;   /* unique ID في نطاق الـ controller */
    const char  *name;     /* اسم اختياري للـ debugfs */
    void        *drv_data; /* بيانات خاصة بالـ driver */
};
```

في `pinctrl-single`، كل pin = register offset واحد (أو جزء من register لو `bits_per_mux`). الـ driver بيبني الـ array ده في `pcs_allocate_pin_table()` — بيحسب عدد الـ pins من `size / (width/8)`.

#### 2. Pin Group — مجموعة pins لوظيفة واحدة

**الـ pin group** هو مجموعة pins بتشكل معاً interface متكامل. مثلاً "uart0_pins" = { TX_pin, RX_pin, RTS_pin, CTS_pin }.

الـ pinctrl-single بيستخرجها من الـ DT node في `pcs_parse_one_pinctrl_entry()`:
```c
/* DT property: pinctrl-single,pins = <offset value> <offset value> ... */
/* كل pair = pin واحد + القيمة اللازمة في الـ register بتاعته */
```

#### 3. Function — الدور المنطقي

**الـ function** هو الاسم المجرد للوظيفة (مثلاً "uart0"). بتتكون من:
- اسم
- قائمة pin groups اللي تقدر تأدي الـ function دي

في `pinctrl-single`، الـ function name == الـ DT node name == الـ group name (تبسيط عملي).

---

### الـ Struct Relationships — تخطيط العلاقات

```
  pcs_device
  ├── struct pinctrl_desc desc          ← اللي بيتسجل عند الـ core
  │   ├── pinctrl_pin_desc *pins        ← الـ array بتاعة كل الـ pins
  │   ├── pinctrl_ops  *pctlops         ← get_groups, dt_node_to_map
  │   ├── pinmux_ops   *pmxops          ← set_mux, gpio_request_enable
  │   └── pinconf_ops  *confops         ← pin_config_get/set
  │
  ├── struct pinctrl_dev *pctl          ← handle اللي بيرجعه الـ core بعد التسجيل
  │
  ├── struct pcs_data pins              ← pcs's own copy of pin array
  │   └── pinctrl_pin_desc *pa[]
  │
  ├── struct list_head gpiofuncs        ← pcs_gpiofunc_range list
  │   └── pcs_gpiofunc_range {
  │       offset, npins, gpiofunc }
  │
  ├── struct list_head irqs             ← pcs_interrupt list (wake-up IRQs)
  │   └── pcs_interrupt {
  │       reg, hwirq, irq }
  │
  ├── void __iomem *base                ← الـ ioremap base address
  ├── unsigned width                    ← عرض الـ register: 8/16/32 bit
  ├── unsigned fmask                    ← mask للـ function bits
  ├── unsigned fshift                   ← shift للـ function bits
  │
  └── read() / write()                  ← function pointers (readb/w/l + writeb/w/l)


  pcs_function                          ← private data للـ function
  ├── const char *name
  ├── pcs_func_vals *vals[]             ← (reg ptr, value, mask) per pin
  │   └── pcs_func_vals {
  │       void __iomem *reg,
  │       unsigned val,
  │       unsigned mask }
  └── pcs_conf_vals *conf[]             ← pinconf settings
      └── pcs_conf_vals {
          enum pin_config_param param,
          val, enable, disable, mask }
```

---

### Three Ops Vtables — ازاي الـ Core بيكلم الـ Driver

الـ pinctrl core بيتكلم مع أي driver عن طريق **3 vtables**. الـ pinctrl-single بيملأهم كالآتي:

#### أولاً: `pinctrl_ops` — إدارة عامة للـ pins والـ groups

```c
static const struct pinctrl_ops pcs_pinctrl_ops = {
    /* delegated للـ generic layer — الـ core عنده بنية جاهزة */
    .get_groups_count = pinctrl_generic_get_group_count,
    .get_group_name   = pinctrl_generic_get_group_name,
    .get_group_pins   = pinctrl_generic_get_group_pins,

    /* خاص بالـ driver */
    .pin_dbg_show  = pcs_pin_dbg_show,    /* debugfs: اعرض قيمة الـ register */
    .dt_node_to_map = pcs_dt_node_to_map, /* اقرأ DT وحول لـ pinctrl_map */
    .dt_free_map    = pcs_dt_free_map,
};
```

#### ثانياً: `pinmux_ops` — التحكم في الـ muxing

```c
static const struct pinmux_ops pcs_pinmux_ops = {
    /* delegated للـ generic layer */
    .get_functions_count = pinmux_generic_get_function_count,
    .get_function_name   = pinmux_generic_get_function_name,
    .get_function_groups = pinmux_generic_get_function_groups,

    /* خاص بالـ driver — بيكتب في الـ register فعلاً */
    .set_mux             = pcs_set_mux,
    .gpio_request_enable = pcs_request_gpio,
};
```

#### ثالثاً: `pinconf_ops` — إعداد خصائص الـ pins

```c
static const struct pinconf_ops pcs_pinconf_ops = {
    .pin_config_get         = pcs_pinconf_get,
    .pin_config_set         = pcs_pinconf_set,
    .pin_config_group_get   = pcs_pinconf_group_get,
    .pin_config_group_set   = pcs_pinconf_group_set,
    .is_generic             = true,  /* بيستخدم الـ generic encoding */
};
```

---

### من الـ Device Tree للـ Register: الرحلة الكاملة

الـ device tree entry بتبدو كده لـ OMAP:

```dts
/* في الـ pinctrl node */
uart0_pins: pinmux_uart0_pins {
    pinctrl-single,pins = <
        0x170  0x00   /* UART0_RXD: offset 0x170, value 0 = MUX_MODE0 */
        0x174  0x00   /* UART0_TXD: offset 0x174, value 0 = MUX_MODE0 */
    >;
};

/* في الـ UART node */
uart0 {
    pinctrl-names = "default";
    pinctrl-0 = <&uart0_pins>;
};
```

**الرحلة خطوة بخطوة:**

```
1. UART driver يعمل probe
         │
         ▼
2. pinctrl core يشوف إن الـ device عنده pinctrl-0 property
         │
         ▼
3. pinctrl core يكال pcs_dt_node_to_map() مع node "uart0_pins"
         │
         ▼
4. pcs_parse_one_pinctrl_entry() تقرأ "pinctrl-single,pins":
   - Row 0: offset=0x170, val=0x00  → pin index = 0x170/4 = 92
   - Row 1: offset=0x174, val=0x00  → pin index = 0x174/4 = 93
         │
         ▼
5. pcs_add_function() تسجل function "uart0_pins" بـ vals[] array
   pinctrl_generic_add_group() تسجل group "uart0_pins" بـ pins[92,93]
         │
         ▼
6. ترجع pinctrl_map بنوع PIN_MAP_TYPE_MUX_GROUP
         │
         ▼
7. pinctrl core يطبق الـ map: يكال pcs_set_mux()
         │
         ▼
8. pcs_set_mux() تلف على vals[]:
   for each val:
       data  = pcs->read(val->reg)     // اقرأ الـ register الحالي
       data &= ~fmask                  // امسح bits الـ function
       data |= (val->val & fmask)      // ضع القيمة الجديدة
       pcs->write(data, val->reg)      // اكتب في الـ register
         │
         ▼
9. الـ pin دلوقتي مضبوط على UART function ✓
```

---

### الـ Two Modes: One-Register-Per-Pin vs Bits-Per-Mux

الـ driver بيدعم نموذجين مختلفين للـ hardware:

#### Mode 1: One register per pin (`bits_per_mux = false`)

```
Physical Layout:
  0x000: [  RESERVED  | PULL | INPUT_EN | MUX_MODE[2:0] ]  ← PIN_0
  0x004: [  RESERVED  | PULL | INPUT_EN | MUX_MODE[2:0] ]  ← PIN_1
  0x008: [  RESERVED  | PULL | INPUT_EN | MUX_MODE[2:0] ]  ← PIN_2

offset = pin_index * (width/8)
```

#### Mode 2: Multiple pins per register (`bits_per_mux = true`, property "pinctrl-single,bit-per-mux")

```
Physical Layout:
  0x000: [ PIN_7[3:0] | PIN_6[3:0] | PIN_5[3:0] | PIN_4[3:0] |
           PIN_3[3:0] | PIN_2[3:0] | PIN_1[3:0] | PIN_0[3:0] ]  ← 8 pins في 32-bit reg
  0x004: [ PIN_15     | ...         ...          | PIN_8      ]

offset  = (pin * bits_per_pin) / 8
shift   = (pin % (width/bits_per_pin)) * bits_per_pin
mask    = fmask << shift
```

في الـ DT بيستخدموا property مختلفة: `pinctrl-single,bits` بدل `pinctrl-single,pins`، وكل entry بتتكون من `<offset value mask>` بدل `<offset value>`.

---

### الـ Pinconf: إعداد الخصائص الكهربية

الـ **pinconf** (pin configuration) هو جزء منفصل عن الـ mux — بيتحكم في الخصائص الكهربية للـ pin:

| Property في الـ DT | الـ Parameter | عدد القيم |
|---|---|---|
| `pinctrl-single,bias-pullup` | `PIN_CONFIG_BIAS_PULL_UP` | 4 (val, enable, disable, mask) |
| `pinctrl-single,bias-pulldown` | `PIN_CONFIG_BIAS_PULL_DOWN` | 4 |
| `pinctrl-single,drive-strength` | `PIN_CONFIG_DRIVE_STRENGTH` | 2 (val, mask) |
| `pinctrl-single,slew-rate` | `PIN_CONFIG_SLEW_RATE` | 2 |
| `pinctrl-single,input-schmitt-enable` | `PIN_CONFIG_INPUT_SCHMITT_ENABLE` | 4 |

**الـ encoding** في الـ kernel: كل config value بيتخزن كـ `unsigned long` واحد:
- الـ bits العلوية = الـ parameter type (`enum pin_config_param`)
- الـ bits السفلية = القيمة

الـ helper macros:
```c
pinconf_to_config_packed(param, arg)  /* يعمل encode */
pinconf_to_config_param(config)       /* يستخرج الـ param */
pinconf_to_config_argument(config)    /* يستخرج الـ value */
```

---

### الـ IRQ Support: Wake-up من الـ Pins

ده feature اختياري موجود في بعض الـ SoCs (OMAP، DRA7). بعض الـ pin registers فيها bits للـ wake-up:

```
  OMAP pin register:
  Bit 14: WAKEUP_ENABLE  ← لو set، الـ pin يصحي الـ system
  Bit 15: WAKEUP_EVENT   ← الـ SoC يحطه لو الـ pin فعلاً صحّى
```

الـ driver بيسجل نفسه كـ **irq_chip** وبيعمل **irq_domain** — ده بيخلي الـ drivers التانية تقدر تطلب IRQs على الـ pins دي بشكل standard.

```
  Parent IRQ (SoC level)
       │
       ▼ pcs_irq_chain_handler() أو pcs_irq_handler()
       │   يقرأ كل register في قائمة pcs->irqs
       │   يشوف bit الـ WAKEUP_EVENT
       ▼
  irq_domain → generic_handle_domain_irq()
       │
       ▼
  Consumer (مثلاً driver بيستنى event على GPIO pin)
```

**ملاحظة مهمة:** الـ IRQ subsystem في الـ kernel (معالج الـ interrupts ومفهوم الـ irq_domain) موضوع منفصل — الـ irq_domain هو abstraction بيعمل mapping بين hardware interrupt numbers وـ Linux virtual IRQ numbers.

---

### ما يمتلكه الـ Subsystem مقابل ما يفوضه للـ Driver

| المسؤولية | المالك |
|---|---|
| منع conflict بين devices على نفس الـ pin | **Pinctrl Core** |
| إدارة الـ states (default, sleep, idle) | **Pinctrl Core** |
| ربط الـ consumer devices بالـ pin configurations | **Pinctrl Core** |
| تخزين الـ functions والـ groups (generic lists) | **Pinctrl Core** (generic helpers) |
| قراءة الـ DT وتحويله لـ register values | **pinctrl-single driver** |
| الكتابة الفعلية في الـ hardware registers | **pinctrl-single driver** |
| تحديد عرض الـ register (8/16/32) وطريقة الـ access | **pinctrl-single driver** |
| إدارة الـ IRQs الخاصة بالـ wake-up events | **pinctrl-single driver** |
| حفظ واسترجاع الـ context عند الـ suspend/resume | **pinctrl-single driver** |
| فهم الـ SoC-specific flags ومعالجتها | **pinctrl-single driver** |

---

### الـ probe() Function: كيف يتم الـ Initialization

```c
/* pcs_probe() — الخطوات بالترتيب */

1. of_device_get_match_data()      // اجلب الـ soc-specific flags (pcs_soc_data)
2. devm_kzalloc(pcs_device)        // allocate الـ main struct
3. of_property_read_u32("pinctrl-single,register-width")  // عرض الـ register
4. of_property_read_u32("pinctrl-single,function-mask")   // الـ mux bits mask
5. platform_get_resource()         // MMIO resource من الـ DT
6. devm_ioremap()                  // map الـ registers في الـ virtual space
7. اختار read()/write() functions حسب الـ width
8. pcs_allocate_pin_table()        // ابني array كل الـ pins
9. devm_pinctrl_register_and_init() // سجّل عند الـ pinctrl core
10. pcs_add_gpio_func()            // اقرأ gpio-range من الـ DT
11. pcs_irq_init_chained_handler() // setup الـ wake-up IRQs (اختياري)
12. pinctrl_enable()               // ابدأ تطبيق الـ configurations
```

---

### مثال حقيقي: TI AM335x (BeagleBone)

على الـ BeagleBone Black، الـ DT بيكون:

```dts
am33xx_pinmux: pinmux@44e10800 {
    compatible = "pinctrl-single";
    reg = <0x44e10800 0x0238>;        /* base address + size */
    #address-cells = <1>;
    #size-cells = <0>;
    pinctrl-single,register-width = <32>;
    pinctrl-single,function-mask = <0x7f>;  /* bits [6:0] */

    uart0_pins: pinmux_uart0_pins {
        pinctrl-single,pins = <
            0x170 0x30  /* RXD: MUX_MODE0 | PULL_UP | RX_ACTIVE */
            0x174 0x00  /* TXD: MUX_MODE0 */
        >;
    };
};
```

الـ `0x7f` mask = bits [6:0]:
- [2:0] = MUX_MODE (0-7)
- [3]   = PULL_TYPE_SEL
- [4]   = PULL_UP_DOWN_EN
- [5]   = INPUT_EN
- [6]   = SLOW_SLEW

القيمة `0x30` = `0b0110000` = INPUT_EN + PULL_UP_DOWN_EN = pin قادر يستقبل signal مع pull-up enabled.

ده اللي بيحصل جوا `pcs_set_mux()`:
```c
data  = readl(base + 0x170);   // اقرأ الـ register الحالي
data &= ~0x7f;                  // امسح الـ 7 bits الـ function
data |= (0x30 & 0x7f);         // ضع القيمة 0x30
writel(data, base + 0x170);    // اكتب
```
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Enums والـ Config Options

#### الـ `pcs_device.flags` — بيتات الـ Feature

| Flag | القيمة | المعنى |
|------|--------|--------|
| `PCS_FEAT_PINCONF` | `(1 << 0)` | الـ driver بيدعم pin configuration (pull-up, drive strength...) |
| `PCS_FEAT_IRQ` | `(1 << 1)` | فيه interrupt على الـ controller (wake-up events) |
| `PCS_QUIRK_SHARED_IRQ` | `(1 << 2)` | الـ IRQ متشارك بين أكتر من instance (OMAP) |
| `PCS_CONTEXT_LOSS_OFF` | `(1 << 3)` | السجلات بتتمسح عند الـ power-off، لازم نحفظها ونرجعها |

#### الـ Macros المشتقة

| Macro | الاستخدام |
|-------|-----------|
| `PCS_HAS_PINCONF` | بيتأكد إن الـ `PCS_FEAT_PINCONF` set |
| `PCS_HAS_IRQ` | بيتأكد إن الـ `PCS_FEAT_IRQ` set |
| `PCS_QUIRK_HAS_SHARED_IRQ` | بيتأكد إن الـ IRQ shared |
| `PCS_OFF_DISABLED` | قيمة `~0U` تعني الـ mux معطل (disabled) |

#### الـ `pin_config_param` المدعومة

| DTS Property | `pin_config_param` | عدد Arguments |
|---|---|---|
| `pinctrl-single,drive-strength` | `PIN_CONFIG_DRIVE_STRENGTH` | 2 |
| `pinctrl-single,slew-rate` | `PIN_CONFIG_SLEW_RATE` | 2 |
| `pinctrl-single,input-enable` | `PIN_CONFIG_INPUT_ENABLE` | 2 |
| `pinctrl-single,input-schmitt` | `PIN_CONFIG_INPUT_SCHMITT` | 2 |
| `pinctrl-single,low-power-mode` | `PIN_CONFIG_MODE_LOW_POWER` | 2 |
| `pinctrl-single,bias-pullup` | `PIN_CONFIG_BIAS_PULL_UP` | 4 |
| `pinctrl-single,bias-pulldown` | `PIN_CONFIG_BIAS_PULL_DOWN` | 4 |
| `pinctrl-single,input-schmitt-enable` | `PIN_CONFIG_INPUT_SCHMITT_ENABLE` | 4 |

> **الـ 2-parameter**: `<value mask>` — بيحسب الـ shift تلقائياً من الـ mask.
> **الـ 4-parameter**: `<value enable disable mask>` — لازم `value` يساوي `enable` أو `disable`.

#### الـ SoC Variants المعروفة

| Compatible String | SoC | Flags | IRQ Enable Bit | IRQ Status Bit |
|---|---|---|---|---|
| `ti,omap3-padconf` | OMAP3 | `SHARED_IRQ` | bit 14 | bit 15 |
| `ti,omap4-padconf` | OMAP4 | `SHARED_IRQ` | bit 14 | bit 15 |
| `ti,omap5-padconf` | OMAP5 | `SHARED_IRQ` | bit 14 | bit 15 |
| `ti,dra7-padconf` | DRA7 | — | bit 24 | bit 25 |
| `ti,am437-padconf` | AM437x | `SHARED_IRQ + CONTEXT_LOSS` | bit 29 | bit 30 |
| `ti,am654-padconf` | AM654 | `SHARED_IRQ + CONTEXT_LOSS` | bit 29 | bit 30 |
| `ti,j7200-padconf` | J7200 | `CONTEXT_LOSS` | — | — |
| `pinctrl-single` | Generic | — | — | — |
| `pinconf-single` | Generic+conf | `FEAT_PINCONF` | — | — |
| `marvell,pxa1908-padconf` | PXA1908 | `FEAT_PINCONF` | — | — |

---

### 1. الـ Structs المهمة

#### `struct pcs_device` — الـ Main Instance

ده الـ struct الأساسي اللي بيمثل الـ pinctrl-single controller كله. بيتخزن فيه كل حاجة من الـ register base لحد الـ IRQ domain.

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `res` | `struct resource *` | الـ memory resource من الـ platform device |
| `base` | `void __iomem *` | العنوان الافتراضي لأول register |
| `saved_vals` | `void *` | buffer لحفظ السجلات قبل الـ sleep (لو `CONTEXT_LOSS_OFF`) |
| `size` | `unsigned` | حجم الـ IO region بالـ bytes |
| `dev` | `struct device *` | الـ device الأصلي |
| `np` | `struct device_node *` | نود الـ device tree |
| `pctl` | `struct pinctrl_dev *` | handle للـ pinctrl core بعد التسجيل |
| `flags` | `unsigned` | بيتات الـ PCS_FEAT_xxx و PCS_QUIRK_xxx |
| `missing_nr_pinctrl_cells` | `struct property *` | workaround لـ legacy DTS مش فيه `#pinctrl-cells` |
| `socdata` | `struct pcs_soc_data` | نسخة embed من الـ SoC-specific data |
| `lock` | `raw_spinlock_t` | بيحمي الـ register read/write (IRQ context safe) |
| `mutex` | `struct mutex` | بيحمي الـ lists (gpiofuncs, irqs) |
| `width` | `unsigned` | عرض الـ register بالـ bits (8/16/32) |
| `fmask` | `unsigned` | الـ mask لبيتات الـ function في الـ register |
| `fshift` | `unsigned` | كام shift يمين عشان توصل لأول بيت في الـ fmask |
| `foff` | `unsigned` | القيمة اللي بتوقف الـ mux (disabled state) |
| `fmax` | `unsigned` | أكبر رقم function ممكن |
| `bits_per_mux` | `bool` | صح لو أكتر من pin في نفس الـ register |
| `bits_per_pin` | `unsigned` | عدد البيتات لكل pin (يتحسب من `fmask`) |
| `pins` | `struct pcs_data` | الـ array الـ static للـ pin descriptors |
| `gpiofuncs` | `struct list_head` | قائمة `pcs_gpiofunc_range` |
| `irqs` | `struct list_head` | قائمة `pcs_interrupt` |
| `chip` | `struct irq_chip` | الـ IRQ chip المدمج |
| `domain` | `struct irq_domain *` | الـ IRQ domain للـ virtual IRQs |
| `desc` | `struct pinctrl_desc` | الـ descriptor اللي بيتسجل في الـ pinctrl core |
| `read` | `fn ptr` | دالة قراءة السجل (8/16/32-bit) |
| `write` | `fn ptr` | دالة كتابة السجل (8/16/32-bit) |

---

#### `struct pcs_function` — تعريف الـ Mux Function

**الغرض**: بيمثل function واحدة (مجموعة pins بتتشبك بنفس البيرفيرال). اتخزن فيها الـ register values اللازمة لتفعيل الـ function دي.

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `name` | `const char *` | اسم الـ function (= اسم نود الـ DTS) |
| `vals` | `struct pcs_func_vals *` | array من الـ (register, value, mask) |
| `nvals` | `unsigned` | حجم الـ vals array |
| `conf` | `struct pcs_conf_vals *` | array من إعدادات الـ pinconf |
| `nconfs` | `int` | حجم الـ conf array |
| `node` | `struct list_head` | link في قائمة (قديم، دلوقتي بيستخدم generic framework) |

---

#### `struct pcs_func_vals` — قيمة Register واحدة

**الغرض**: الـ atomic unit للـ mux programming — عنوان register واحد + القيمة المطلوبة.

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `reg` | `void __iomem *` | العنوان الافتراضي للـ register |
| `val` | `unsigned` | القيمة المطلوبة كتابتها |
| `mask` | `unsigned` | الـ bits اللي المفروض نعدلها بس (في حالة `bits_per_mux`) |

---

#### `struct pcs_conf_vals` — إعداد الـ Pinconf

**الغرض**: بيصف إزاي نقرأ ونكتب خاصية معينة (pull-up, drive strength…) في الـ register.

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `param` | `enum pin_config_param` | نوع الإعداد (PULL_UP, DRIVE_STRENGTH...) |
| `val` | `unsigned` | القيمة المطلوبة من الـ user |
| `enable` | `unsigned` | البيتات اللي بتوضح إن الخاصية enabled |
| `disable` | `unsigned` | البيتات اللي بتوضح إن الخاصية disabled |
| `mask` | `unsigned` | الـ bits المعنية في الـ register |

> **الفرق بين 2-param و 4-param**: الـ 2-param مش بيستخدم `enable`/`disable`، بيحسب القيمة مباشرة من الـ `val` والـ `mask`. الـ 4-param بيستخدمهم للـ boolean properties (PULL_UP, PULL_DOWN).

---

#### `struct pcs_conf_type` — ربط الـ DTS Property بالـ Param

**الغرض**: lookup table بيربط اسم الـ DTS property بالـ `pin_config_param` المقابل. بيستخدمه `pcs_parse_pinconf()` وقت parsing.

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `name` | `const char *` | اسم الـ property في الـ DTS |
| `param` | `enum pin_config_param` | الـ enum value المقابل |

---

#### `struct pcs_gpiofunc_range` — نطاق GPIO Function

**الغرض**: بيعرف مجموعة pins ليها نفس قيمة الـ mux لما بنطلب GPIO. بيتحدد من `pinctrl-single,gpio-range` في الـ DTS.

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `offset` | `unsigned` | رقم أول pin في النطاق ده |
| `npins` | `unsigned` | عدد الـ pins في النطاق |
| `gpiofunc` | `unsigned` | قيمة الـ mux لتفعيل الـ GPIO |
| `node` | `struct list_head` | link في `pcs->gpiofuncs` |

---

#### `struct pcs_data` — Wrapper للـ Pin Array

**الغرض**: بيغلف الـ static array الخاصة بالـ pin descriptors وبيتابع عدد الـ pins المضافة.

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `pa` | `struct pinctrl_pin_desc *` | الـ array الـ dynamic المخصصة بـ `devm_kcalloc` |
| `cur` | `int` | index الـ pin الجاي اللي هيتضاف |

---

#### `struct pcs_soc_data` — بيانات خاصة بالـ SoC

**الغرض**: بيحدد السلوك الخاص بكل SoC، بيتجي من الـ `of_device_id.data` وبيتنسخ في `pcs->socdata`.

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `flags` | `unsigned` | الـ flags الـ initial للـ SoC (SHARED_IRQ, CONTEXT_LOSS...) |
| `irq` | `int` | رقم الـ interrupt (من DTS أو platform data) |
| `irq_enable_mask` | `unsigned` | الـ bit اللي بيـ enable الـ wake-up interrupt في الـ register |
| `irq_status_mask` | `unsigned` | الـ bit اللي بيوضح إن في wake-up event حصل |
| `rearm` | `void (*)(void)` | دالة اختيارية لإعادة تأهيل الـ interrupt (OMAP PRM) |

---

#### `struct pcs_interrupt` — تسجيل IRQ واحد

**الغرض**: بيمثل virtual IRQ واحد مرتبط بـ register معين في الـ pinctrl. بيتخلق في `pcs_irqdomain_map()`.

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `reg` | `void __iomem *` | عنوان الـ register الخاص بالـ interrupt ده |
| `hwirq` | `irq_hw_number_t` | رقم الـ HW IRQ (= register offset) |
| `irq` | `unsigned int` | رقم الـ virtual IRQ |
| `node` | `struct list_head` | link في `pcs->irqs` |

---

#### `struct pcs_pdata` — الـ Platform Data (Legacy)

**الغرض**: بيجي من الـ auxdata في بعض الـ OMAP boards القديمة اللي بتستخدم الـ PRM interrupts. لازم نستبدله بالـ DTS.

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `irq` | `int` | رقم الـ interrupt |
| `rearm` | `void (*)(void)` | دالة الـ rearm |

---

### 2. مخطط العلاقات بين الـ Structs

```
                        ┌──────────────────────────────────────────────┐
                        │              pcs_device                       │
                        │                                               │
                        │  base ──────────────→ [IO Registers in HW]  │
                        │  res ────────────────→ struct resource       │
                        │  dev ────────────────→ struct device         │
                        │  np  ────────────────→ struct device_node    │
                        │  pctl ───────────────→ struct pinctrl_dev ──→ [pinctrl core] │
                        │                                               │
                        │  desc: struct pinctrl_desc                   │
                        │    ├─ pins ──────────→ pcs_data.pa[]         │
                        │    ├─ pctlops ────────→ pcs_pinctrl_ops      │
                        │    ├─ pmxops ─────────→ pcs_pinmux_ops       │
                        │    └─ confops ─────────→ pcs_pinconf_ops     │
                        │                                               │
                        │  pins: pcs_data                              │
                        │    └─ pa[] ─────────→ pinctrl_pin_desc[]    │
                        │                                               │
                        │  socdata: pcs_soc_data                       │
                        │    ├─ irq_enable_mask                        │
                        │    ├─ irq_status_mask                        │
                        │    └─ rearm()                                 │
                        │                                               │
                        │  gpiofuncs list ─────→ pcs_gpiofunc_range   │
                        │    node ↔ node ↔ node (linked list)          │
                        │                                               │
                        │  irqs list ──────────→ pcs_interrupt        │
                        │    node ↔ node ↔ node (linked list)          │
                        │                                               │
                        │  chip: irq_chip ────→ [IRQ subsystem]       │
                        │  domain ────────────→ struct irq_domain      │
                        │                                               │
                        │  lock: raw_spinlock_t                        │
                        │  mutex: struct mutex                          │
                        └──────────────────────────────────────────────┘

  pcs_function ──────────────────────────────────────────────────────────
  │  name: "uart0_pins"
  │  vals[] ──→ pcs_func_vals { reg, val, mask }
  │              pcs_func_vals { reg, val, mask }
  │  conf[] ──→ pcs_conf_vals { param, val, enable, disable, mask }
  └─────────── يتخزن كـ function->data في الـ pinctrl generic framework

  الـ pinctrl core بيتعامل مع الـ driver عبر:
  pinctrl_dev → pinctrl_desc → {pctlops, pmxops, confops}
```

---

### 3. مخطط دورة الحياة (Lifecycle)

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                    PROBE (pcs_probe)                            │
  │                                                                 │
  │  1. devm_kzalloc(pcs_device)                                   │
  │  2. raw_spin_lock_init + mutex_init                             │
  │  3. of_property_read → width, fmask, fshift, fmax, foff        │
  │  4. platform_get_resource + devm_ioremap → base                │
  │  5. اختيار read/write functions (8/16/32-bit)                  │
  │  6. pcs_allocate_pin_table()                                    │
  │       └─ devm_kcalloc(pin_desc[])                              │
  │       └─ pcs_add_pin() × N                                     │
  │  7. devm_pinctrl_register_and_init()                           │
  │       └─ يسجل الـ desc مع الـ pinctrl core                    │
  │  8. pcs_add_gpio_func()                                         │
  │       └─ يبني pcs_gpiofunc_range list                          │
  │  9. irq_of_parse_and_map() → socdata.irq                       │
  │  10. pcs_irq_init_chained_handler()   (لو PCS_HAS_IRQ)        │
  │        └─ irq_domain_create_simple()                           │
  │        └─ irq_set_chained_handler_and_data()                   │
  │  11. pinctrl_enable()                                           │
  └─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │                  RUNTIME (consumer requests mux)                │
  │                                                                 │
  │  consumer driver: pinctrl_select_state()                        │
  │    → pinctrl core يبحث عن الـ map                              │
  │    → pcs_dt_node_to_map() [أول مرة فقط، يبني الـ map]         │
  │    → pcs_set_mux()                                              │
  │         └─ loop على func->vals[]                               │
  │         └─ raw_spin_lock_irqsave                               │
  │         └─ read-modify-write register                           │
  │         └─ raw_spin_unlock_irqrestore                           │
  └─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │                   SUSPEND (pinctrl_single_suspend_noirq)        │
  │                                                                 │
  │  لو PCS_CONTEXT_LOSS_OFF:                                      │
  │    pcs_save_context() → saved_vals[] = read(all registers)     │
  │  pinctrl_force_sleep(pctl)                                      │
  └─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │                   RESUME (pinctrl_single_resume_noirq)          │
  │                                                                 │
  │  لو PCS_CONTEXT_LOSS_OFF:                                      │
  │    pcs_restore_context() → write(all registers) من saved_vals  │
  │  pinctrl_force_default(pctl)                                    │
  └─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │                   REMOVE (pcs_remove)                           │
  │                                                                 │
  │  pcs_free_resources()                                           │
  │    └─ pcs_irq_free()                                           │
  │         └─ irq_domain_remove()                                  │
  │         └─ free_irq() أو irq_set_chained_handler(NULL)        │
  │  [devm_ resources يتحرروا تلقائياً]                            │
  └─────────────────────────────────────────────────────────────────┘
```

---

### 4. مخطط تدفق الـ Calls

#### 4.1 تسجيل الـ Driver وبناء الـ Pin Table

```
pcs_probe()
  ├─→ of_device_get_match_data()          [جيب الـ SoC data]
  ├─→ devm_kzalloc(pcs_device)
  ├─→ of_property_read_u32("pinctrl-single,register-width")   → pcs->width
  ├─→ of_property_read_u32("pinctrl-single,function-mask")    → pcs->fmask
  ├─→ devm_ioremap()                       → pcs->base
  ├─→ pcs_allocate_pin_table()
  │     ├─→ حساب nr_pins من pcs->size و pcs->width
  │     ├─→ devm_kcalloc(pinctrl_pin_desc[nr_pins])
  │     └─→ pcs_add_pin() × nr_pins
  │           └─→ [لو irq_enable_mask] clear wake-up bits
  │           └─→ pin->number = i; pcs->pins.cur++
  ├─→ devm_pinctrl_register_and_init()    → pcs->pctl
  ├─→ pcs_add_gpio_func()
  │     └─→ of_parse_phandle_with_args("pinctrl-single,gpio-range")
  │     └─→ devm_kzalloc(pcs_gpiofunc_range)
  │     └─→ list_add_tail(&range->node, &pcs->gpiofuncs)
  ├─→ pcs_irq_init_chained_handler()
  │     ├─→ INIT_LIST_HEAD(&pcs->irqs)
  │     ├─→ إعداد pcs->chip (irq_chip ops)
  │     ├─→ [SHARED_IRQ] request_irq(pcs_irq_handler)
  │     │   [else]        irq_set_chained_handler_and_data(pcs_irq_chain_handler)
  │     └─→ irq_domain_create_simple()    → pcs->domain
  └─→ pinctrl_enable(pcs->pctl)
```

#### 4.2 تحويل نود الـ DTS لـ Map (dt_node_to_map)

```
pcs_dt_node_to_map(pctldev, np_config)
  ├─→ devm_kcalloc(2 × pinctrl_map)
  ├─→ [bits_per_mux] pcs_parse_bits_in_pinctrl_entry()
  │     ├─→ pinctrl_count_index_with_args(np, "pinctrl-single,bits")
  │     ├─→ devm_kcalloc(pcs_func_vals[])
  │     ├─→ loop: pinctrl_parse_index_with_args()
  │     │     └─→ [offset, val, mask] → vals[].reg/val/mask
  │     │     └─→ pcs_get_pin_by_offset() → pin index
  │     ├─→ mutex_lock(&pcs->mutex)
  │     ├─→ pcs_add_function() → pinmux_generic_add_function()
  │     ├─→ pinctrl_generic_add_group()
  │     └─→ mutex_unlock(&pcs->mutex)
  │
  └─→ [else] pcs_parse_one_pinctrl_entry()
        ├─→ pinctrl_count_index_with_args(np, "pinctrl-single,pins")
        ├─→ devm_kcalloc(pcs_func_vals[])
        ├─→ loop: offset + val → vals[found].reg/val
        │     └─→ pcs_get_pin_by_offset() → pin index
        ├─→ mutex_lock(&pcs->mutex)
        ├─→ pcs_add_function()
        ├─→ pinctrl_generic_add_group()
        ├─→ [PCS_HAS_PINCONF] pcs_parse_pinconf()
        │     ├─→ count properties (prop2[] و prop4[])
        │     ├─→ devm_kcalloc(pcs_conf_vals[nconfs])
        │     ├─→ pcs_add_conf2() × 5 properties
        │     └─→ pcs_add_conf4() × 3 properties
        └─→ mutex_unlock(&pcs->mutex)
```

#### 4.3 تفعيل الـ Mux (set_mux)

```
consumer: pinctrl_select_state(state)
  → pinctrl core
    → pcs_set_mux(pctldev, fselector, group)
        ├─→ pinmux_generic_get_function(fselector) → function_desc
        ├─→ function->data → pcs_function *func
        └─→ for i in 0..func->nvals:
              vals = &func->vals[i]
              raw_spin_lock_irqsave(&pcs->lock)
              val = pcs->read(vals->reg)
              val &= ~mask
              val |= (vals->val & mask)
              pcs->write(val, vals->reg)
              raw_spin_unlock_irqrestore(&pcs->lock)
```

#### 4.4 طلب GPIO (gpio_request_enable)

```
gpio subsystem: gpiochip_request()
  → pinctrl: pinmux_gpio_request()
    → pcs_request_gpio(pctldev, range, pin)
        ├─→ [!fmask] return -ENOTSUPP
        └─→ list_for_each(pcs->gpiofuncs)
              frange = pcs_gpiofunc_range
              [pin in range?]
              offset = pcs_pin_reg_offset_get(pin)
              data = pcs->read(base + offset)
              [bits_per_mux]:
                pin_shift = pcs_pin_shift_reg_get(pin)
                data &= ~(fmask << pin_shift)
                data |= frange->gpiofunc << pin_shift
              [else]:
                data &= ~fmask
                data |= frange->gpiofunc
              pcs->write(data, base + offset)
```

#### 4.5 معالجة الـ IRQ (Wake-up Event)

```
  [HW pin detects wake-up event]
    → IRQ fires on pcs_soc->irq

  [SHARED_IRQ path]:
  pcs_irq_handler(irq, pcs_soc)
    → pcs_irq_handle(pcs_soc)

  [CHAINED path]:
  pcs_irq_chain_handler(desc)
    → chained_irq_enter(chip, desc)
    → pcs_irq_handle(pcs_soc)
    → chained_irq_exit(chip, desc)

  pcs_irq_handle(pcs_soc):
    pcs = container_of(pcs_soc, pcs_device, socdata)
    list_for_each(pcs->irqs):
      pcswi = pcs_interrupt
      raw_spin_lock(&pcs->lock)
      mask = pcs->read(pcswi->reg)
      raw_spin_unlock(&pcs->lock)
      [mask & irq_status_mask]:
        generic_handle_domain_irq(pcs->domain, pcswi->hwirq)
        count++
    return count   [0 = IRQ_NONE, >0 = IRQ_HANDLED]

  pcs_irqdomain_map(domain, irq, hwirq):
    devm_kzalloc(pcs_interrupt)
    pcswi->reg = base + hwirq
    pcswi->hwirq = hwirq
    pcswi->irq = irq
    mutex_lock → list_add_tail → mutex_unlock
    irq_set_chip_and_handler(irq, &pcs->chip, handle_level_irq)
```

---

### 5. استراتيجية الـ Locking

الـ driver بيستخدم **قفلين** بأهداف مختلفة تماماً:

#### الـ `raw_spinlock_t lock` — حماية الـ Registers

| الخاصية | التفاصيل |
|---------|----------|
| **النوع** | `raw_spinlock_t` (مش `spinlock_t` العادية) |
| **السبب** | بيتاخد في الـ IRQ context، ولازم يشتغل حتى لو الـ PREEMPT_RT kernel |
| **بيحمي** | كل `pcs->read()` و `pcs->write()` على الـ hardware registers |
| **يتاخد في** | `pcs_set_mux()`, `pcs_irq_set()`, `pcs_irq_handle()` |
| **الأماكن** | `raw_spin_lock_irqsave` في الـ thread context، `raw_spin_lock` في الـ IRQ context |

#### الـ `struct mutex mutex` — حماية الـ Lists

| الخاصية | التفاصيل |
|---------|----------|
| **النوع** | `struct mutex` (sleeping lock) |
| **بيحمي** | `pcs->gpiofuncs` list و `pcs->irqs` list |
| **يتاخد في** | `pcs_add_gpio_func()`, `pcs_parse_one_pinctrl_entry()`, `pcs_irqdomain_map()` |
| **مش بيتاخدش** | في أي IRQ context (مستحيل) |

#### ترتيب الـ Locking

```
  ممنوع: mutex → lock  (يسبب deadlock)
  مسموح: lock فقط (في IRQ context)
  مسموح: mutex فقط (في process context)
  مسموح: mutex أولاً ثم lock (لو محتاجين الاتنين)
```

#### الـ Lock Classes (Lockdep)

```c
static struct lock_class_key pcs_lock_class;      // للـ IRQ chip lock
static struct lock_class_key pcs_request_class;   // للـ IRQ request mutex
```

بيستخدمهم `irq_set_lockdep_class()` عشان الـ lockdep يعرف إن الـ pinctrl IRQ chip في category مختلفة عن أبوه، ويمنع الـ false recursion warnings.

---

### ملخص العلاقات الكاملة

```
platform_device
    │
    ▼
pcs_probe()
    │ ينشئ
    ▼
pcs_device [الـ core struct]
    ├──[embed]──→ pcs_soc_data     [SoC-specific: IRQ masks, flags]
    ├──[embed]──→ pcs_data         [pin table wrapper]
    │                └──→ pinctrl_pin_desc[]
    ├──[embed]──→ pinctrl_desc     [descriptor للـ pinctrl core]
    │                ├──→ pcs_pinctrl_ops
    │                ├──→ pcs_pinmux_ops
    │                └──→ pcs_pinconf_ops
    ├──[embed]──→ irq_chip         [IRQ chip للـ wake-up]
    ├──[ptr]───→ pinctrl_dev       [handle من الـ pinctrl core]
    ├──[ptr]───→ irq_domain        [virtual IRQ mapping]
    ├──[list]──→ pcs_gpiofunc_range[]   [GPIO mux ranges]
    └──[list]──→ pcs_interrupt[]        [registered virtual IRQs]

DTS parsing يبني:
    pcs_function
        ├──→ pcs_func_vals[]   [register/value pairs لتفعيل الـ mux]
        └──→ pcs_conf_vals[]   [pin config settings]
    يتسجل في الـ pinctrl core كـ:
        function_desc { name, groups, data=pcs_function }
```
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheat Sheet

#### I/O Helpers

| Function | الغرض |
|---|---|
| `pcs_readb/w/l` | قراءة register بعرض 8/16/32 bit |
| `pcs_writeb/w/l` | كتابة register بعرض 8/16/32 bit |
| `pcs_pin_reg_offset_get` | حساب offset الـ register للـ pin |
| `pcs_pin_shift_reg_get` | حساب shift داخل الـ register للـ pin |

#### Pinctrl Core Ops

| Function | الغرض |
|---|---|
| `pcs_pin_dbg_show` | طباعة معلومات pin في debugfs |
| `pcs_dt_node_to_map` | تحويل DT node → pinctrl_map |
| `pcs_dt_free_map` | تحرير الـ map المخصصة |
| `pcs_get_function` | جلب الـ pcs_function للـ pin الحالي |
| `pcs_set_mux` | تطبيق mux function على الـ registers |
| `pcs_request_gpio` | تحويل pin لـ GPIO mode |

#### Pinconf Ops

| Function | الغرض |
|---|---|
| `pcs_pinconf_get` | قراءة pin configuration |
| `pcs_pinconf_set` | كتابة pin configuration |
| `pcs_pinconf_group_get` | قراءة config لمجموعة pins |
| `pcs_pinconf_group_set` | كتابة config لمجموعة pins |
| `pcs_pinconf_clear_bias` | مسح bias settings للـ pin |
| `pcs_pinconf_bias_disable` | فحص إذا كان bias معطّل |

#### DT Parsing

| Function | الغرض |
|---|---|
| `pcs_parse_one_pinctrl_entry` | parse مدخل DT بـ one-pin-per-register |
| `pcs_parse_bits_in_pinctrl_entry` | parse مدخل DT بـ bits-per-mux mode |
| `pcs_parse_pinconf` | parse خصائص pinconf من DT node |
| `pcs_add_conf2` / `pcs_add_conf4` | إضافة config entries بـ 2 أو 4 parameters |
| `add_config` / `add_setting` | helper لملء pcs_conf_vals و settings array |

#### Pin Table

| Function | الغرض |
|---|---|
| `pcs_allocate_pin_table` | تخصيص وتهيئة جدول الـ pins كلها |
| `pcs_add_pin` | إضافة pin واحد للجدول |
| `pcs_add_function` | تسجيل function جديدة في framework |
| `pcs_get_pin_by_offset` | ترجمة register offset → pin index |
| `pcs_add_gpio_func` | parse وتخزين gpio-range entries من DT |

#### IRQ Subsystem

| Function | الغرض |
|---|---|
| `pcs_irq_set` | enable/disable interrupt في الـ register |
| `pcs_irq_mask` | mask interrupt (irq_chip callback) |
| `pcs_irq_unmask` | unmask interrupt (irq_chip callback) |
| `pcs_irq_set_wake` | toggle wake-up للـ suspend/resume |
| `pcs_irq_handle` | common interrupt handler منطق |
| `pcs_irq_handler` | handler للـ shared IRQ case |
| `pcs_irq_chain_handler` | handler للـ chained IRQ case |
| `pcs_irqdomain_map` | ربط hwirq بـ virq في الـ irq_domain |
| `pcs_irq_init_chained_handler` | إعداد الـ irq_chip والـ irq_domain |
| `pcs_irq_free` | تحرير موارد الـ interrupt |

#### Power Management

| Function | الغرض |
|---|---|
| `pcs_save_context` | حفظ قيم الـ registers قبل الـ suspend |
| `pcs_restore_context` | استعادة قيم الـ registers بعد الـ resume |
| `pinctrl_single_suspend_noirq` | suspend callback |
| `pinctrl_single_resume_noirq` | resume callback |

#### Driver Lifecycle

| Function | الغرض |
|---|---|
| `pcs_probe` | driver probe — كل التهيئة |
| `pcs_remove` | driver remove — تحرير الموارد |
| `pcs_free_resources` | cleanup الـ IRQ والـ DT properties |
| `pcs_quirk_missing_pinctrl_cells` | معالجة legacy DT binding |

---

### Group 1: I/O Access Helpers

الـ driver يتجنب استخدام regmap بسبب الـ performance overhead على OMAP حيث الـ mux registers تُكتب كثيراً (قبل وبعد الـ idle). بدلاً من ذلك، يحتفظ بـ function pointer مباشر (`pcs->read` / `pcs->write`) يُختار وقت الـ probe حسب `pcs->width`.

---

#### `pcs_readb` / `pcs_readw` / `pcs_readl`

```c
static unsigned int pcs_readb(void __iomem *reg);
static unsigned int pcs_readw(void __iomem *reg);
static unsigned int pcs_readl(void __iomem *reg);
```

**ما تفعله:** wrappers مباشرة فوق `readb/readw/readl`. الغرض الوحيد هو تطابق signature مع function pointer `unsigned (*read)(void __iomem *reg)` في `pcs_device`.

| Parameter | الوصف |
|---|---|
| `reg` | عنوان MMIO المقروء منه |

**Return:** قيمة الـ register cast إلى `unsigned int`.

**Key details:** لا يوجد locking هنا — الـ caller مسؤول عن الـ `raw_spin_lock_irqsave` إذا لزم.

---

#### `pcs_writeb` / `pcs_writew` / `pcs_writel`

```c
static void pcs_writeb(unsigned int val, void __iomem *reg);
static void pcs_writew(unsigned int val, void __iomem *reg);
static void pcs_writel(unsigned int val, void __iomem *reg);
```

**ما تفعله:** wrappers فوق `writeb/writew/writel`. تُعيّن في `pcs_probe` حسب قيمة `pcs->width` (8، 16، أو 32 bit).

---

#### `pcs_pin_reg_offset_get`

```c
static unsigned int pcs_pin_reg_offset_get(struct pcs_device *pcs,
                                           unsigned int pin);
```

**ما تفعله:** تحسب الـ byte offset للـ register الذي يحتوي على الـ pin المطلوب. في الـ mode العادي (one-register-per-pin) الـ offset = `pin × mux_bytes`. في الـ `bits_per_mux` mode، عدة pins تشترك في نفس الـ register فيُحسب بالـ alignment.

| Parameter | الوصف |
|---|---|
| `pcs` | الـ device instance |
| `pin` | رقم الـ pin (index) |

**Return:** الـ byte offset من `pcs->base`.

**Pseudocode:**
```c
mux_bytes = pcs->width / 8;
if bits_per_mux:
    byte_pos = (bits_per_pin * pin) / 8
    return align_down(byte_pos, mux_bytes)
else:
    return pin * mux_bytes
```

---

#### `pcs_pin_shift_reg_get`

```c
static unsigned int pcs_pin_shift_reg_get(struct pcs_device *pcs,
                                          unsigned int pin);
```

**ما تفعله:** في الـ `bits_per_mux` mode حيث عدة pins في نفس الـ register، تُحدد عدد الـ bits التي يجب الـ shift بها للوصول لـ field الـ pin داخل الـ register.

**Return:** عدد الـ bits للـ shift (`0` إذا كان pin الأول في الـ register).

---

### Group 2: Pinctrl Core Operations

تُشكّل هذه الـ functions الـ `pcs_pinctrl_ops` struct المُسجَّل مع الـ pinctrl framework.

---

#### `pcs_pin_dbg_show`

```c
static void pcs_pin_dbg_show(struct pinctrl_dev *pctldev,
                             struct seq_file *s,
                             unsigned pin);
```

**ما تفعله:** تطبع في debugfs القيمة الحالية للـ mux register للـ pin المحدد مع الـ physical address. مفيدة جداً للـ debugging لأنها تُظهر القيمة الفعلية في الهاردوير.

| Parameter | الوصف |
|---|---|
| `pctldev` | الـ pinctrl device handle |
| `s` | seq_file للـ debugfs output |
| `pin` | رقم الـ pin |

**الكالر:** kernel debugfs infrastructure عند قراءة `/sys/kernel/debug/pinctrl/*/pins`.

---

#### `pcs_dt_free_map`

```c
static void pcs_dt_free_map(struct pinctrl_dev *pctldev,
                            struct pinctrl_map *map, unsigned num_maps);
```

**ما تفعله:** تحرر الـ `pinctrl_map` array المُخصصة بـ `devm_kcalloc` في `pcs_dt_node_to_map`. بما إنها مُخصصة بـ `devm_*`، الـ `devm_kfree` كافية.

**الكالر:** pinctrl framework بعد الانتهاء من تطبيق الـ map.

---

#### `pcs_get_function`

```c
static int pcs_get_function(struct pinctrl_dev *pctldev, unsigned pin,
                            struct pcs_function **func);
```

**ما تفعله:** تُجلب الـ `pcs_function` المرتبطة بالـ pin عبر الـ `mux_setting` المحفوظ في الـ `pin_desc`. تستخدم `pinmux_generic_get_function` للوصول للـ `function_desc`، ثم تستخرج الـ `pcs_function` من `function->data`.

| Parameter | الوصف |
|---|---|
| `pctldev` | الـ pinctrl device |
| `pin` | رقم الـ pin |
| `func` | output pointer — يُملأ بالـ pcs_function |

**Return:** `0` نجاح، `-ENOTSUPP` إذا الـ pin مش مُعرّف في DTS أو مش مُفعَّل، `-EINVAL` إذا الـ function مش موجودة.

**Key details:** الـ pin يجب أن يكون له `mux_setting` صالح — أي يجب أن يكون قد مرّ بـ `pinctrl_select_state` قبل الاستدعاء.

---

#### `pcs_set_mux`

```c
static int pcs_set_mux(struct pinctrl_dev *pctldev, unsigned fselector,
                       unsigned group);
```

**ما تفعله:** الـ callback الأهم في الـ driver. تُطبّق الـ mux function على الهاردوير بكتابة كل الـ `pcs_func_vals` entries في الـ `pcs_function`. لكل register، تقرأ القيمة الحالية، تُعدّل الـ bits المطلوبة بالـ mask، ثم تكتب.

| Parameter | الوصف |
|---|---|
| `pctldev` | الـ pinctrl device |
| `fselector` | index الـ function في الـ framework |
| `group` | index الـ group (غير مستخدم مباشرة هنا) |

**Return:** `0` نجاح، `-EINVAL` إذا الـ function مش موجودة.

**Locking:** `raw_spin_lock_irqsave` لكل register write — ضروري لأن هذه الـ registers تُكتب من أي context.

**Pseudocode:**
```c
function = pinmux_generic_get_function(fselector);
func = function->data;
for each vals in func->vals:
    raw_spin_lock_irqsave(&pcs->lock);
    val = pcs->read(vals->reg);
    val = (val & ~mask) | (vals->val & mask);
    pcs->write(val, vals->reg);
    raw_spin_unlock_irqrestore(&pcs->lock);
```

**الكالر:** pinctrl framework عند `pinctrl_select_state()`.

---

#### `pcs_request_gpio`

```c
static int pcs_request_gpio(struct pinctrl_dev *pctldev,
                            struct pinctrl_gpio_range *range, unsigned pin);
```

**ما تفعله:** تُحوّل الـ pin لـ GPIO mode بكتابة الـ `gpiofunc` value في الـ register المناسب. تبحث في قائمة `pcs->gpiofuncs` عن الـ `pcs_gpiofunc_range` التي تحتوي على الـ pin.

| Parameter | الوصف |
|---|---|
| `pctldev` | الـ pinctrl device |
| `range` | الـ GPIO range (غير مستخدم مباشرة) |
| `pin` | رقم الـ pin المطلوب تحويله |

**Return:** `0` دائماً (حتى لو Pin مش في أي range — لا يعتبر error).

**الكالر:** `gpio_request()` / `gpiod_get()` path في الـ kernel.

---

### Group 3: Pinconf Operations

تُشكّل هذه الـ functions الـ `pcs_pinconf_ops` struct. مبنية على افتراض أن كل pin configuration مخزونة في نفس registers الـ mux أو registers مجاورة محددة في الـ DTS.

---

#### `pcs_pinconf_get`

```c
static int pcs_pinconf_get(struct pinctrl_dev *pctldev,
                           unsigned pin, unsigned long *config);
```

**ما تفعله:** تقرأ الـ configuration الحالية للـ pin من الهاردوير. تُحدد الـ `param` المطلوب من `*config`، تبحث في `func->conf[]` عن الـ entry المطابق، تقرأ الـ register، وتُعيد القيمة.

**معالجة خاصة لـ `PIN_CONFIG_BIAS_DISABLE`:** لا entry مخصص لها، بدلاً من ذلك تستدعي `pcs_pinconf_bias_disable` للتحقق إذا كانت PULL_UP وPULL_DOWN كلاهما inactive.

**معالجة `PIN_CONFIG_INPUT_SCHMITT`:** تتطلب التحقق من `INPUT_SCHMITT_ENABLE` أولاً في `func->conf[]` قبل إرجاع القيمة.

| Parameter | الوصف |
|---|---|
| `pctldev` | الـ pinctrl device |
| `pin` | رقم الـ pin |
| `config` | input: الـ param المطلوب؛ output: القيمة المقروءة |

**Return:** `0` نجاح، `-ENOTSUPP` إذا الـ param غير مدعوم أو الـ pin مش configured.

---

#### `pcs_pinconf_set`

```c
static int pcs_pinconf_set(struct pinctrl_dev *pctldev,
                           unsigned pin, unsigned long *configs,
                           unsigned num_configs);
```

**ما تفعله:** تطبّق array من الـ configurations على الـ pin. لكل `config` في الـ `configs[]`:
- إذا كان `PIN_CONFIG_BIAS_DISABLE` → تستدعي `pcs_pinconf_clear_bias` التي تُكتب disable bits لكل bias.
- للـ 2-parameter configs (drive-strength، slew-rate...) → تُطبّق الـ `arg` في الـ field بـ shift وmask.
- للـ 4-parameter configs (bias-pull-up، bias-pull-down، schmitt-enable) → تكتب `enable` أو `disable` bits.

**Locking:** لا يوجد spinlock هنا مباشرة — يُفترض استدعاؤها من context آمن (عادة mutex محمي من الـ pinctrl framework).

**Return:** `0` نجاح، `-ENOTSUPP` إذا param غير مدعوم في `func->conf`.

---

#### `pcs_pinconf_group_get` / `pcs_pinconf_group_set`

```c
static int pcs_pinconf_group_get(struct pinctrl_dev *pctldev,
                                 unsigned group, unsigned long *config);
static int pcs_pinconf_group_set(struct pinctrl_dev *pctldev,
                                 unsigned group, unsigned long *configs,
                                 unsigned num_configs);
```

**ما تفعلان:** iterates على كل الـ pins في الـ group وتطبّق `pcs_pinconf_get/set` على كل pin. في الـ get، تُتحقق أن كل الـ pins لها نفس القيمة (consistency check).

---

#### `pcs_pinconf_clear_bias`

```c
static void pcs_pinconf_clear_bias(struct pinctrl_dev *pctldev, unsigned pin);
```

**ما تفعله:** تمسح كل bias settings للـ pin بكتابة `0` لكل bias parameter في `pcs_bias[]` (PULL_DOWN وPULL_UP). تستدعي `pcs_pinconf_set` مباشرة.

**الكالر:** `pcs_pinconf_set` عند تطبيق `BIAS_DISABLE` أو عند تفعيل bias جديد (يجب مسح القديم أولاً).

---

#### `pcs_pinconf_bias_disable`

```c
static bool pcs_pinconf_bias_disable(struct pinctrl_dev *pctldev, unsigned pin);
```

**ما تفعله:** تفحص إذا كان كل bias types (PULL_DOWN، PULL_UP) غير مُفعّل. ترجع `true` فقط إذا فشلت `pcs_pinconf_get` للـ pin لكل bias types — أي لا bias مُفعَّل.

**Return:** `true` إذا bias مُعطَّل، `false` إذا أي bias مُفعَّل.

---

### Group 4: Device Tree Parsing

هذا الـ group هو قلب الـ driver — يُحوّل الـ DT entries لـ pinctrl maps قابلة للاستخدام من الـ framework.

---

#### `pcs_dt_node_to_map`

```c
static int pcs_dt_node_to_map(struct pinctrl_dev *pctldev,
                              struct device_node *np_config,
                              struct pinctrl_map **map, unsigned *num_maps);
```

**ما تفعله:** الـ entry point الرئيسي للـ DT parsing. تُخصص `pinctrl_map` array (حتى 2 entries: واحدة للـ mux وواحدة للـ pinconf). ثم تقرر أي parser تستدعي:
- إذا `pcs->bits_per_mux` → `pcs_parse_bits_in_pinctrl_entry`
- وإلا → `pcs_parse_one_pinctrl_entry`

| Parameter | الوصف |
|---|---|
| `pctldev` | الـ pinctrl device |
| `np_config` | الـ DT node للـ pinmux entry |
| `map` | output: array من pinctrl_map |
| `num_maps` | output: عدد الـ maps المُعبأة |

**Return:** `0` نجاح أو error code سلبي.

**الكالر:** pinctrl framework عند `pinctrl_select_state()`.

---

#### `pcs_parse_one_pinctrl_entry`

```c
static int pcs_parse_one_pinctrl_entry(struct pcs_device *pcs,
                                       struct device_node *np,
                                       struct pinctrl_map **map,
                                       unsigned *num_maps,
                                       const char **pgnames);
```

**ما تفعله:** تُحلّل الـ property `pinctrl-single,pins` من الـ DT node. كل entry في الـ property هي (offset, value) أو (offset, value1, value2) tuple. تُبني `pcs_func_vals[]` array وتُسجّل function وgroup في الـ pinctrl framework.

**Pseudocode flow:**
```
rows = count entries in "pinctrl-single,pins"
alloc vals[rows], pins[rows]

for each row:
    parse (offset, val) from DTS
    vals[i].reg = base + offset
    vals[i].val = parsed value
    pin = pcs_get_pin_by_offset(offset)
    pins[found++] = pin

mutex_lock(pcs->mutex)
fsel = pcs_add_function(name, vals, found, pgnames)
gsel = pinctrl_generic_add_group(name, pins, found)
map->type = PIN_MAP_TYPE_MUX_GROUP
map->data.mux = {group=name, function=name}

if PCS_HAS_PINCONF:
    pcs_parse_pinconf(func, map)  // adds second map entry
mutex_unlock(pcs->mutex)
```

**Locking:** `mutex_lock(&pcs->mutex)` يحمي تسجيل الـ functions والـ groups.

**Error paths:** cleanup كامل بـ `goto` labels: `free_pingroups → free_function → free_pins → free_vals`.

---

#### `pcs_parse_bits_in_pinctrl_entry`

```c
static int pcs_parse_bits_in_pinctrl_entry(struct pcs_device *pcs,
                                           struct device_node *np,
                                           struct pinctrl_map **map,
                                           unsigned *num_maps,
                                           const char **pgnames);
```

**ما تفعله:** مثل `pcs_parse_one_pinctrl_entry` لكن للـ `bits_per_mux` mode. الـ property هي `pinctrl-single,bits` وكل entry هي (offset, value, mask) حيث mask تُحدد أي pins في الـ register يتأثرون. تُحلّل الـ mask bit-by-bit لاستخراج كل pin على حدة.

**قيد:** لا يدعم pinconf (`PCS_HAS_PINCONF` يُسبّب error مباشرة).

**Loop logic للـ mask:**
```c
while (mask):
    bit_pos = __ffs(mask)           // أول bit مفعَّل
    pin_num = bit_pos / bits_per_pin
    mask_pos = fmask << bit_pos     // bits الخاصة بهذا الـ pin
    vals[found] = {reg, val & mask_pos, mask_pos}
    pins[found] = base_pin + pin_num
    mask &= ~mask_pos               // امسح bits هذا الـ pin
```

---

#### `pcs_parse_pinconf`

```c
static int pcs_parse_pinconf(struct pcs_device *pcs, struct device_node *np,
                             struct pcs_function *func,
                             struct pinctrl_map **map);
```

**ما تفعله:** تُحلّل خصائص الـ pinconf من الـ DT node وتملأ `func->conf[]`. تدعم نوعين:
- **prop2** (2-parameter): `drive-strength`, `slew-rate`, `input-enable`, `input-schmitt`, `low-power-mode` — كل منها (value, mask).
- **prop4** (4-parameter): `bias-pullup`, `bias-pulldown`, `input-schmitt-enable` — كل منها (value, enable, disable, mask).

تُنشئ أيضاً `settings[]` array بالـ `pinconf_to_config_packed` format وتُضيفها كـ `PIN_MAP_TYPE_CONFIGS_GROUP` map entry.

**Return:** `0` نجاح، `-ENOTSUPP` إذا لا يوجد أي pinconf properties، `-ENOMEM` إذا فشل الـ alloc.

---

#### `pcs_add_conf2` / `pcs_add_conf4`

```c
static void pcs_add_conf2(struct pcs_device *pcs, struct device_node *np,
                          const char *name, enum pin_config_param param,
                          struct pcs_conf_vals **conf, unsigned long **settings);

static void pcs_add_conf4(struct pcs_device *pcs, struct device_node *np,
                          const char *name, enum pin_config_param param,
                          struct pcs_conf_vals **conf, unsigned long **settings);
```

**ما تفعلان:** تقرأان property من الـ DT بـ `of_property_read_u32_array`، تملأان الـ `pcs_conf_vals` entry، وتُضيفان الـ setting المقابلة. الـ pointers تُزاد تلقائياً بعد كل entry.

---

#### `add_config` / `add_setting`

```c
static void add_config(struct pcs_conf_vals **conf, enum pin_config_param param,
                       unsigned value, unsigned enable, unsigned disable, unsigned mask);
static void add_setting(unsigned long **setting, enum pin_config_param param,
                        unsigned arg);
```

**ما تفعلان:** helper functions بسيطة جداً — تملأان الـ struct/value في الـ pointer الحالي ثم تُقدّمان الـ pointer للعنصر التالي. مصممة للاستخدام في loop.

---

### Group 5: Pin Table Management

---

#### `pcs_allocate_pin_table`

```c
static int pcs_allocate_pin_table(struct pcs_device *pcs);
```

**ما تفعله:** تحسب عدد الـ pins الكلي وتُخصص `pinctrl_pin_desc[]` array. العدد يُحسب إما بقسمة الـ IO size على `mux_bytes` (normal mode) أو بقسمة `size * 8 / bits_per_pin` (bits_per_mux mode). بعدها تستدعي `pcs_add_pin` لكل pin.

**Return:** `0` نجاح، `-ENOMEM` إذا فشل الـ alloc.

**Side effects:** تُعيّن `pcs->desc.pins` و `pcs->desc.npins`.

---

#### `pcs_add_pin`

```c
static int pcs_add_pin(struct pcs_device *pcs, unsigned int offset);
```

**ما تفعله:** تُضيف pin واحد للـ static array. إذا كان الـ SoC يدعم IRQ، تفحص إذا كانت الـ irq_enable_mask مضبوطة في الـ register (bootloader قد تركها مفتوحة) وتُصفّرها.

**Return:** الـ pin index نجاح، `-ENOMEM` إذا تجاوز الحد الأقصى.

---

#### `pcs_add_function`

```c
static int pcs_add_function(struct pcs_device *pcs,
                            struct pcs_function **fcn,
                            const char *name,
                            struct pcs_func_vals *vals,
                            unsigned int nvals,
                            const char **pgnames,
                            unsigned int npgnames);
```

**ما تفعله:** تُخصص `pcs_function` struct، تملأ `vals` و `nvals` و `name`، ثم تُسجّلها في الـ pinctrl framework بـ `pinmux_generic_add_function`. الـ `function->data` يُشير للـ `pcs_function` المُخصصة.

**Return:** الـ function selector (non-negative) نجاح، قيمة سلبية خطأ. عند الخطأ `*fcn = NULL`.

**المستدعي:** الـ caller يجب أن يحتفظ بـ `pcs->mutex`.

---

#### `pcs_get_pin_by_offset`

```c
static int pcs_get_pin_by_offset(struct pcs_device *pcs, unsigned offset);
```

**ما تفعله:** تعكس عملية `pcs_pin_reg_offset_get` — تحوّل register offset لـ pin index. تُتحقق أن الـ offset في نطاق الـ `pcs->size`.

**Return:** pin index نجاح، `-EINVAL` إذا offset خارج النطاق.

---

#### `pcs_config_match`

```c
static int pcs_config_match(unsigned data, unsigned enable, unsigned disable);
```

**ما تفعله:** تُقارن قيمة data مع enable وdisable bits. تُستخدم في `pcs_add_conf4` للتحقق من صحة الـ DTS values.

**Return:** `1` إذا == enable، `0` إذا == disable، `-EINVAL` إذا لا هذا ولا ذاك.

---

#### `pcs_add_gpio_func`

```c
static int pcs_add_gpio_func(struct device_node *node, struct pcs_device *pcs);
```

**ما تفعله:** تُحلّل property `pinctrl-single,gpio-range` من الـ DT. لكل entry، تُنشئ `pcs_gpiofunc_range` struct تحتوي على (offset, npins, gpiofunc) وتُضيفها لـ `pcs->gpiofuncs` list.

**Locking:** `mutex_lock(&pcs->mutex)` لحماية الـ list.

**Return:** `0` نجاح، `-ENOMEM` إذا فشل الـ alloc. نهاية الـ property تُعامل كـ end condition وليس كـ error.

---

### Group 6: IRQ Subsystem

الـ driver يدعم wake-up interrupts حيث كل pin register يحتوي على bit للـ wake-up enable وbit للـ wake-up event. يدعم حالتين: **shared IRQ** (عدة instances تشترك في IRQ واحد — كـ OMAP) و **chained IRQ** (IRQ مستقل لكل instance).

---

#### `pcs_irq_set`

```c
static inline void pcs_irq_set(struct pcs_soc_data *pcs_soc,
                               int irq, const bool enable);
```

**ما تفعله:** تبحث في `pcs->irqs` list عن الـ `pcs_interrupt` المطابق للـ `irq`، ثم تُعدّل الـ `irq_enable_mask` bit في الـ register. بعد الكتابة تقرأ مرة ثانية (flush posted write) لضمان وصول الكتابة للهاردوير.

**Locking:** `raw_spin_lock(&pcs->lock)` — لا `irqsave` هنا لأنها تُستدعى من interrupt context مسبقاً.

**Side effects:** إذا `pcs_soc->rearm != NULL`، تستدعيه بعد كل operation (للـ OMAP PRM).

---

#### `pcs_irq_mask` / `pcs_irq_unmask`

```c
static void pcs_irq_mask(struct irq_data *d);
static void pcs_irq_unmask(struct irq_data *d);
```

**ما تفعلان:** callbacks للـ `irq_chip`. تستخرجان `pcs_soc_data` من `irq_data_get_irq_chip_data(d)` وتستدعيان `pcs_irq_set` بـ `false`/`true`.

**الكالر:** IRQ core عند mask/unmask operations.

---

#### `pcs_irq_set_wake`

```c
static int pcs_irq_set_wake(struct irq_data *d, unsigned int state);
```

**ما تفعله:** toggle wake-up للـ suspend/resume. عند `state=1` → `pcs_irq_unmask`، عند `state=0` → `pcs_irq_mask`. للـ runtime PM الـ wake-up enabled بـ default.

**Return:** `0` دائماً.

---

#### `pcs_irq_handle`

```c
static int pcs_irq_handle(struct pcs_soc_data *pcs_soc);
```

**ما تفعله:** الـ common interrupt handling logic. تمشي على كل entries في `pcs->irqs`، تقرأ الـ register، وإذا كانت `irq_status_mask` bit مضبوطة تستدعي `generic_handle_domain_irq` لإيصال الـ interrupt لـ consumer الصح.

**Return:** عدد الـ interrupts التي تم تسليمها (يُستخدم للتمييز بين `IRQ_HANDLED` و `IRQ_NONE`).

---

#### `pcs_irq_handler`

```c
static irqreturn_t pcs_irq_handler(int irq, void *d);
```

**ما تفعله:** الـ handler للـ **shared IRQ** case. تُسجَّل بـ `request_irq` مع `IRQF_SHARED`. تستدعي `pcs_irq_handle` وترجع `IRQ_HANDLED` أو `IRQ_NONE` بناءً على النتيجة.

---

#### `pcs_irq_chain_handler`

```c
static void pcs_irq_chain_handler(struct irq_desc *desc);
```

**ما تفعله:** الـ handler للـ **chained IRQ** case. تُستخدم `chained_irq_enter` و `chained_irq_exit` لإدارة الـ parent chip. تُسجَّل بـ `irq_set_chained_handler_and_data`.

**الكالر:** IRQ core مباشرة عند وصول interrupt.

---

#### `pcs_irqdomain_map`

```c
static int pcs_irqdomain_map(struct irq_domain *d, unsigned int irq,
                             irq_hw_number_t hwirq);
```

**ما تفعله:** تُسمى من الـ irq_domain عند mapping hwirq → virq جديد. تُنشئ `pcs_interrupt` struct جديد (hwirq = register offset من `pcs->base`)، تُضيفه لـ `pcs->irqs`، وتُعيّن الـ irq_chip وhandler على الـ virq.

**Key details:**
- `irq_set_chip_and_handler(irq, &pcs->chip, handle_level_irq)` — يُستخدم level-triggered handling لأن wake-up interrupts عادة level.
- `irq_set_lockdep_class` و `irq_set_noprobe` لمنع lockdep false positives.

---

#### `pcs_irq_init_chained_handler`

```c
static int pcs_irq_init_chained_handler(struct pcs_device *pcs,
                                        struct device_node *np);
```

**ما تفعله:** الـ setup الكامل للـ interrupt infrastructure:
1. تُهيئ `pcs->chip` (irq_chip) بالـ callbacks.
2. إذا `PCS_QUIRK_SHARED_IRQ`: تستدعي `request_irq` بـ `IRQF_SHARED`.
3. وإلا: `irq_set_chained_handler_and_data`.
4. تُنشئ الـ `irq_domain` بـ `irq_domain_create_simple` — الـ hwirq هو الـ register offset مما يجعل الـ mapping مباشراً.

**Return:** `0` نجاح، `-EINVAL` إذا `irq_enable_mask` أو `irq_status_mask` = 0، أو إذا فشل إنشاء الـ domain.

---

#### `pcs_irq_free`

```c
static void pcs_irq_free(struct pcs_device *pcs);
```

**ما تفعله:** تُحرر موارد الـ interrupt بترتيب عكسي: تُزيل الـ irq_domain، ثم تُحرر الـ IRQ (shared أو chained حسب الـ flag).

---

### Group 7: Power Management

---

#### `pcs_save_context`

```c
static int pcs_save_context(struct pcs_device *pcs);
```

**ما تفعله:** تحفظ كل قيم الـ pinmux registers قبل الـ suspend في `pcs->saved_vals` buffer. تُستخدم فقط إذا `PCS_CONTEXT_LOSS_OFF` flag مضبوط (السيناريو الذي يفقد فيه الـ SoC context الـ registers وقت الـ off-mode). تُخصص الـ buffer أول مرة بـ `GFP_ATOMIC`.

**Switch على `pcs->width`:** يستخدم المؤشر المناسب (u64/u32/u16) لتجنب الـ type mismatch.

**Return:** `0` نجاح، `-ENOMEM` إذا فشل الـ alloc.

---

#### `pcs_restore_context`

```c
static void pcs_restore_context(struct pcs_device *pcs);
```

**ما تفعله:** تكتب القيم المحفوظة في `pcs->saved_vals` مجدداً لكل الـ registers بعد الـ resume. لا تُخصص ذاكرة — تفترض أن `pcs_save_context` قد مرّت أولاً.

---

#### `pinctrl_single_suspend_noirq`

```c
static int pinctrl_single_suspend_noirq(struct device *dev);
```

**ما تفعله:**
1. إذا `PCS_CONTEXT_LOSS_OFF`: تستدعي `pcs_save_context`.
2. تستدعي `pinctrl_force_sleep` لإجبار الـ pinctrl framework على تطبيق الـ sleep state.

**noirq context:** تُنفَّذ بعد تعطيل IRQs — آمن للوصول للـ registers بدون locks.

---

#### `pinctrl_single_resume_noirq`

```c
static int pinctrl_single_resume_noirq(struct device *dev);
```

**ما تفعله:**
1. إذا `PCS_CONTEXT_LOSS_OFF`: تستدعي `pcs_restore_context` لاستعادة الـ registers.
2. تستدعي `pinctrl_force_default` للرجوع للـ default state.

---

### Group 8: Driver Lifecycle

---

#### `pcs_probe`

```c
static int pcs_probe(struct platform_device *pdev);
```

**ما تفعله:** الـ entry point الرئيسي للـ driver. تتبع هذه الخطوات بالترتيب:

```
1. of_device_get_match_data()          // جلب SoC-specific data
2. devm_kzalloc(pcs_device)           // تخصيص main struct
3. raw_spin_lock_init + mutex_init    // تهيئة locks
4. of_property_read_u32(register-width, function-mask, function-off)
5. pcs_quirk_missing_pinctrl_cells()  // معالجة legacy DT
6. platform_get_resource + devm_request_mem_region + devm_ioremap
7. اختيار pcs->read/write حسب width
8. ملء pcs->desc (name, pctlops, pmxops, confops)
9. pcs_allocate_pin_table()           // بناء pin table
10. devm_pinctrl_register_and_init()  // تسجيل مع framework
11. pcs_add_gpio_func()               // parse gpio ranges
12. irq_of_parse_and_map() + dev_get_platdata()  // IRQ setup
13. pcs_irq_init_chained_handler()    // إعداد IRQ infrastructure
14. pinctrl_enable()                  // تفعيل الـ controller
```

**Error handling:** cleanup بـ `goto free` الذي يستدعي `pcs_free_resources`.

---

#### `pcs_remove`

```c
static void pcs_remove(struct platform_device *pdev);
```

**ما تفعله:** تستدعي فقط `pcs_free_resources`. باقي الموارد المُخصصة بـ `devm_*` تُحرَّر تلقائياً بالـ devres framework.

---

#### `pcs_free_resources`

```c
static void pcs_free_resources(struct pcs_device *pcs);
```

**ما تفعله:**
1. `pcs_irq_free()` — تحرير IRQ resources.
2. إذا `CONFIG_PINCTRL_SINGLE` مُضمَّن في الـ kernel (built-in): تُزيل الـ `missing_nr_pinctrl_cells` property من الـ DT بـ `of_remove_property`.

**ملاحظة:** في حالة module، لا حاجة لـ `of_remove_property` لأن الـ DT node يُدمَّر مع الـ module unload.

---

#### `pcs_quirk_missing_pinctrl_cells`

```c
static int pcs_quirk_missing_pinctrl_cells(struct pcs_device *pcs,
                                           struct device_node *np,
                                           int cells);
```

**ما تفعله:** معالجة legacy DT nodes التي لا تحتوي على `#pinctrl-cells` property (مطلوبة من الـ framework). تُنشئ `struct property` وتُضيفها للـ DT node ديناميكياً في الـ runtime. تُطبع warning تطلب من المطور تحديث الـ DTS.

**Return:** `0` إذا property موجودة أصلاً أو تمت إضافتها بنجاح، `-ENOMEM` إذا فشل الـ alloc.

---

### ملاحظات ختامية مهمة

#### الـ SoC Data Table

| Compatible String | SoC Variant | Flags | Notes |
|---|---|---|---|
| `ti,omap3/4/5-padconf` | OMAP wkup | `SHARED_IRQ` | WAKEUP_EN bit 14 |
| `ti,am437-padconf` | AM437x | `SHARED_IRQ + CONTEXT_LOSS_OFF` | Context save/restore |
| `ti,am654-padconf` | AM654 | `SHARED_IRQ + CONTEXT_LOSS_OFF` | WKUP bits 29/30 |
| `ti,dra7-padconf` | DRA7 | none | WAKEUPENABLE bits 24/25 |
| `ti,j7200-padconf` | J7200 | `CONTEXT_LOSS_OFF` | No shared IRQ |
| `pinctrl-single` | Generic | none | No IRQ, no pinconf |
| `pinconf-single` | Generic+conf | `FEAT_PINCONF` | No IRQ |

#### الـ Locking Strategy

```
raw_spinlock (pcs->lock):
    - pcs_set_mux() — كتابة mux registers
    - pcs_irq_set() — كتابة IRQ enable registers
    - pcs_irq_handle() — قراءة IRQ status registers
    (مناسب للـ atomic/interrupt context)

mutex (pcs->mutex):
    - pcs_parse_one_pinctrl_entry()
    - pcs_parse_bits_in_pinctrl_entry()
    - pcs_add_gpio_func()
    - pcs_irqdomain_map()
    (يحمي list modifications في sleepable context)
```
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs — المسارات والقراءة

الـ **pinctrl subsystem** بيعرض معلومات تفصيلية عبر **debugfs** في `/sys/kernel/debug/pinctrl/`.

```bash
# اعرف اسم الـ controller (يظهر بعد probe ناجح)
ls /sys/kernel/debug/pinctrl/

# مثال ناتج على OMAP/AM65x:
# pinctrl-single/   ← الـ directory الخاص بالـ driver

# اقرأ كل الـ pins مع قيمها وعناوينها
cat /sys/kernel/debug/pinctrl/pinctrl-single/pins

# مثال ناتج:
# pin 0 (44e10800) 44e10800 00000031 pinctrl-single
# pin 1 (44e10804) 44e10804 00000007 pinctrl-single
# ← العمود الأول: رقم الـ pin
# ← الثاني: العنوان الفيزيائي (من pcs->res->start + offset)
# ← الثالث: القيمة الحالية في الـ register (raw hex)
# ← الرابع: اسم الـ driver

# اعرض الـ pin groups المسجلة
cat /sys/kernel/debug/pinctrl/pinctrl-single/pingroups

# اعرض الـ functions (pinmux functions)
cat /sys/kernel/debug/pinctrl/pinctrl-single/pinmux-functions

# اعرض mapping (أي device يستخدم أي function)
cat /sys/kernel/debug/pinctrl/pinctrl-single/pinmux-pins

# مثال ناتج:
# pin 14: UNCLAIMED
# pin 15: gpio-leds.0 (GPIO UNCLAIMED) function uart0 group uart0

# اعرض pinconf للـ pins (لو PCS_FEAT_PINCONF مفعّل)
cat /sys/kernel/debug/pinctrl/pinctrl-single/pinconf-pins
cat /sys/kernel/debug/pinctrl/pinctrl-single/pinconf-groups
```

الدالة `pcs_pin_dbg_show()` في الكود هي اللي بتملأ ناتج `pins` — بتطبع العنوان الفيزيائي + القيمة الـ raw.

---

#### 2. sysfs — المسارات المفيدة

```bash
# تحقق من وجود الـ driver وأنه probe بنجاح
ls /sys/bus/platform/drivers/pinctrl-single/

# اعرض الـ device المربوط بالـ driver
ls /sys/bus/platform/devices/ | grep pinctrl

# مثال على AM335x:
ls /sys/bus/platform/devices/44e10800.pinmux/

# اقرأ معلومات الـ resource (base address, size)
cat /sys/bus/platform/devices/44e10800.pinmux/resource

# تحقق من الـ IRQ المخصص للـ controller
cat /sys/bus/platform/devices/44e10800.pinmux/irq   # لو موجود

# power management state
cat /sys/bus/platform/devices/44e10800.pinmux/power/runtime_status
```

---

#### 3. ftrace — الـ Tracepoints والـ Events

الـ **pinctrl subsystem** عنده tracepoints جاهزة:

```bash
# فعّل الـ tracing
mount -t tracefs tracefs /sys/kernel/tracing

# اعرض الـ events المتاحة للـ pinctrl
ls /sys/kernel/tracing/events/pinctrl/

# مثال events:
# pinctrl_single_set_mux
# pinctrl_pins_show  ← مش دايمًا موجودة
# pin_config_set
# pin_config_get

# فعّل كل events الـ pinctrl
echo 1 > /sys/kernel/tracing/events/pinctrl/enable

# أو فعّل event بعينه
echo 1 > /sys/kernel/tracing/events/pinctrl/pin_config_set/enable

# ابدأ الـ trace
echo 1 > /sys/kernel/tracing/tracing_on

# ... شغّل الـ device أو الـ scenario اللي بتختبره ...

# اقرأ النتيجة
cat /sys/kernel/tracing/trace

# مثال ناتج:
# systemd-1    [000] ....  12.345678: pin_config_set: dev=spi0, pin=18, config=0x00060001

# تتبع الـ function calls داخل pinctrl-single
echo 'pcs_set_mux' > /sys/kernel/tracing/set_ftrace_filter
echo 'pcs_pinconf_set' >> /sys/kernel/tracing/set_ftrace_filter
echo function > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace
```

---

#### 4. printk / dynamic debug

الكود يستخدم `dev_dbg()` و`dev_err()` و`dev_warn()` بكثرة. لتفعيل الـ **dynamic debug**:

```bash
# فعّل كل الـ debug messages من pinctrl-single.c
echo 'file pinctrl-single.c +p' > /sys/kernel/debug/dynamic_debug/control

# أو فعّل debug لـ module بأكمله
echo 'module pinctrl_single +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل مع line numbers وأسماء functions
echo 'module pinctrl_single +pflmt' > /sys/kernel/debug/dynamic_debug/control

# تحقق من الـ messages المفعّلة
cat /sys/kernel/debug/dynamic_debug/control | grep pinctrl-single
```

**رسائل `dev_dbg()` المهمة في الكود:**

| الرسالة | الموقع | المعنى |
|---------|--------|--------|
| `"allocating %i pins"` | `pcs_allocate_pin_table()` | عدد الـ pins عند الـ probe |
| `"enabling %s function%i"` | `pcs_set_mux()` | تفعيل function معينة |
| `"%pOFn index: 0x%x value: 0x%x"` | `pcs_parse_one_pinctrl_entry()` | قيم الـ DT entry |
| `"irq enabled at boot for pin at %lx"` | `pcs_add_pin()` | IRQ كان مفعّل قبل الـ driver |
| `"failed to match enable or disable bits"` | `pcs_add_conf4()` | pinconf value مش متطابقة |

```bash
# بعد تفعيل dynamic debug، شوف الـ output
dmesg | grep pinctrl-single
# أو في الـ real-time
dmesg -w | grep -i pinctrl
```

---

#### 5. Kernel Config Options للـ Debugging

```bash
# الـ options الأساسية
CONFIG_PINCTRL_SINGLE=y          # الـ driver نفسه (أو =m)
CONFIG_DEBUG_PINCTRL=y           # يفعّل extra validation في pinctrl core
CONFIG_PINMUX=y                  # دعم pinmux (لازم يكون enabled)
CONFIG_PINCONF=y                 # دعم pinconf (لازم للـ PCS_FEAT_PINCONF)

# Debugging عام
CONFIG_DYNAMIC_DEBUG=y           # يسمح بـ dev_dbg() dynamic activation
CONFIG_DEBUG_FS=y                # يفعّل debugfs
CONFIG_FTRACE=y                  # يفعّل ftrace
CONFIG_FUNCTION_TRACER=y         # function-level tracing
CONFIG_TRACING=y

# لكشف الـ locking bugs (spinlock/mutex)
CONFIG_PROVE_LOCKING=y           # lockdep
CONFIG_DEBUG_SPINLOCK=y
CONFIG_DEBUG_MUTEXES=y
CONFIG_LOCK_STAT=y               # إحصائيات الـ locking

# لكشف الـ memory bugs
CONFIG_DEBUG_SLAB=y
CONFIG_KASAN=y                   # AddressSanitizer — مفيد جداً
CONFIG_DEBUG_OBJECTS_WORK=y

# للـ IRQ debugging
CONFIG_GENERIC_IRQ_DEBUGFS=y     # يضيف /sys/kernel/debug/irq/
CONFIG_IRQ_DOMAIN_DEBUG=y
```

---

#### 6. أدوات خاصة بالـ Subsystem

**pinctrl-specific tools:**

```bash
# أداة pinctrl من package: libpinctrl-utils أو داخل kernel tools
# مش موجودة دايمًا — بنستخدم debugfs مباشرة

# devlink: مش مرتبط بـ pinctrl-single

# i2cdetect/gpiodetect للتحقق من الـ GPIO
gpiodetect            # يعرض كل gpio chips
gpioinfo gpiochip0    # يعرض pins وحالتها

# للتحقق من الـ IRQ domains
cat /sys/kernel/debug/irq/domains/  # لو CONFIG_GENERIC_IRQ_DEBUGFS=y
cat /proc/interrupts | grep pinctrl  # شوف لو الـ pinctrl IRQ بيتلقى

# تحقق من الـ DT properties المحملة
dtc -I fs /sys/firmware/devicetree/base > /tmp/current.dts 2>/dev/null
grep -A 20 'pinctrl-single' /tmp/current.dts
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---------------------|--------|------|
| `"register width not specified"` | الـ DT مش فيه `pinctrl-single,register-width` | أضف `pinctrl-single,register-width = <32>;` في الـ DT |
| `"could not get resource"` | مفيش `reg` property في الـ DT أو المورد محجوز | تحقق من `reg` في الـ DT node وتعارض العناوين |
| `"could not get mem_region"` | المنطقة محجوزة من driver تاني | شوف `/proc/iomem` وتعارض العناوين |
| `"could not ioremap"` | فشل الـ ioremap | مشكلة في الـ MMU أو العنوان غلط |
| `"too many pins, max %i"` | الـ DT محدد pins أكتر من الـ size | راجع `reg` و`register-width` في الـ DT |
| `"mux offset out of range: 0x%x (0x%x)"` | offset في الـ DT بره نطاق الـ controller | offset في `pinctrl-single,pins` غلط |
| `"Invalid number of rows: %d"` | الـ DT property فارغة أو مش موجودة | تحقق من صياغة `pinctrl-single,pins` |
| `"invalid args_count for spec: %i"` | عدد arguments في الـ DT entry غلط | لازم 2 أو 3 args لـ pinctrl-single,pins |
| `"could not add functions for %pOFn %ux"` | pin offset مش موجود في الـ table | تحقق من offset القيمة vs حجم الـ controller |
| `"could not register single pinctrl driver"` | فشل تسجيل الـ pinctrl device | شوف `dmesg` لأسباب أعمق (duplicate names, etc.) |
| `"please update dts to use #pinctrl-cells"` | DT قديم بدون `#pinctrl-cells` | أضف `#pinctrl-cells = <1>;` (أو 2 لو bits-per-mux) |
| `"pinconf not supported"` | محاولة استخدام pinconf مع bits-per-mux | الاتنين مش compatible في الكود الحالي |
| `"mask field of the property can't be 0"` | mask في pinconf property = 0 | الـ DT القيمة الرابعة في `pinctrl-single,bias-pullup` مش تكون 0 |
| `"initialized with no interrupts"` | IRQ init فشل لكن الـ probe كمل | راجع `irq_enable_mask` و`irq_status_mask` في SoC data |
| `"Invalid mask for %pOFn at 0x%x"` | mask في `pinctrl-single,bits` غير صالح | تحقق من alignment الـ mask مع `fmask` |

---

#### 8. نقاط استراتيجية لـ dump_stack() / WARN_ON()

```c
/* في pcs_set_mux() — تحقق إن الـ function data مش NULL قبل الكتابة */
WARN_ON(!func->vals);
WARN_ON(func->nvals == 0);

/* في pcs_irqdomain_map() — تحقق من صحة الـ hwirq */
WARN_ON(hwirq >= pcs->size);

/* في pcs_probe() — تحقق من الـ width المقبولة */
WARN_ON(pcs->width != 8 && pcs->width != 16 && pcs->width != 32);

/* في pcs_save_context() / pcs_restore_context() — context loss debugging */
if (!pcs->saved_vals) {
    dev_warn(pcs->dev, "saved_vals is NULL during restore\n");
    dump_stack();  /* لمعرفة المسار */
}

/* في pcs_irq_handle() — لو count = 0 رغم وجود interrupt */
WARN_ON_ONCE(count == 0 && (mask & pcs_soc->irq_status_mask));
```

---

### Hardware Level

#### 1. التحقق من تطابق حالة الـ Hardware مع الـ Kernel

```bash
# اقرأ القيمة الفعلية من debugfs (ما يعرفه الـ kernel)
cat /sys/kernel/debug/pinctrl/pinctrl-single/pins | grep "44e10800"
# مثال ناتج: pin 0 (44e10800) 44e10800 00000031 pinctrl-single
# القيمة: 0x00000031

# ثم اقرأ مباشرة من الـ hardware عبر /dev/mem
# (BASE_ADDRESS من cat /proc/iomem أو DT)
devmem2 0x44e10800 w   # w = 32-bit word
# مثال ناتج: Value at address 0x44E10800 (0x...) : 0x00000031
# المطابقة = الـ kernel محدّث صح
# عدم المطابقة = مشكلة في الـ read/write أو race condition
```

**القيم المتوقعة حسب الـ SoC:**

| SoC | Base Address (مثال) | Register Width | Function Bits |
|-----|---------------------|----------------|--------------|
| AM335x / OMAP3 | `0x44e10800` | 32-bit | bits [2:0] |
| OMAP4/5 | `0x4a002800` | 32-bit | bits [3:0] |
| DRA7x | `0x4a003400` | 32-bit | bits [3:0] |
| AM437x | `0x44e10800` | 32-bit | bits [5:0] |

---

#### 2. Register Dump Techniques

```bash
# devmem2 — اقرأ register واحد (32-bit)
devmem2 <PHYS_ADDR> w

# مثال: اقرأ PIN_CTRL_0 على AM335x
devmem2 0x44e10800 w

# اقرأ نطاق كامل من registers (bash loop)
BASE=0x44e10800
for i in $(seq 0 4 124); do
    ADDR=$(printf "0x%x" $((BASE + i)))
    VAL=$(devmem2 $ADDR w 2>/dev/null | awk '/Value/{print $NF}')
    PIN=$((i / 4))
    printf "PIN %3d @ 0x%08x = %s\n" $PIN $((BASE + i)) $VAL
done

# بديل باستخدام /dev/mem مباشرة (لو devmem2 مش موجود)
dd if=/dev/mem bs=4 count=32 skip=$((0x44e10800 / 4)) 2>/dev/null | \
    od -A x -t x4

# io utility (من package io)
io -4 -r 0x44e10800   # قراءة 32-bit
io -4 -w 0x44e10800 0x00000031  # كتابة (خطر! اعرف إيه بتكتبه)

# مقارنة نسخة الـ saved_vals مع الـ hardware (بعد suspend)
# مش ممكن مباشرة، لكن ممكن تقرأ قبل وبعد suspend
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

**للـ pinmux debugging:**

- قيس الـ pin اللي المفروض يتغير function بعد `pcs_set_mux()`.
- على logic analyzer: راقب الـ I2C/SPI/UART signal بعد الـ mux تغيّر — لو مفيش signal يبقى الـ mux لسه غلط.
- استخدم **trigger** على rising edge على الـ pin المعني.

**للـ IRQ wake-up debugging (OMAP_WAKEUP_EN):**

- الـ bit 14 (OMAP) أو bit 24 (DRA7) في الـ pin register هو الـ wakeup enable.
- راقب الـ wakeup signal على الـ oscilloscope: لو الـ pin اتحرك لكن الـ IRQ مستحدثش، يبقى الـ status mask مش بيتقرأ صح.
- قيس timing بين edge على الـ pin وبين الـ CPU wakeup line.

**Checklist عملي:**

```
[ ] تأكد الـ pin mode صح: GPIO vs Peripheral function
[ ] تأكد الـ pull-up/pull-down صح (بالـ multimeter: idle voltage)
[ ] تأكد الـ drive strength كافية (oscilloscope: edge slew rate)
[ ] تأكد مفيش bus contention (مترتين يكتبوا على نفس الـ pin)
```

---

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة الـ Hardware | ما يظهر في الـ Kernel Log | التشخيص |
|---------------------|--------------------------|---------|
| Pin stuck in GPIO mode | الـ peripheral (UART/SPI) مش بيشتغل، مفيش error | اقرأ الـ register مباشرة — bit[2:0] يرجع 0x7 (GPIO) بدل function |
| Pull-up مش موجود على I2C | `i2c i2c-0: No ACK from...` | multimeter: الـ idle voltage < 2V — غير قيمة pull في DT |
| Drive strength ضعيفة | أخطاء متقطعة على SPI/high-speed | oscilloscope: الـ edges مائلة جداً — زود `drive-strength` في DT |
| Wakeup pin مش بيصحّي | device لا يستيقظ من suspend | راجع `irq_enable_mask` وقرأ الـ register بعد suspend |
| عنوان الـ pinctrl register غلط في DT | `"could not get resource"` + probe fail | قارن `reg` في DT مع datasheet و`/proc/iomem` |
| تداخل بين SoC instances | kernel panic أو garbage values | `/proc/iomem` — تحقق من overlap في العناوين |

---

#### 5. Device Tree Debugging

```bash
# اعرض الـ DT الحالي اللي الـ kernel محمّله
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null > /tmp/live.dts

# ابحث عن الـ pinctrl node
grep -A 30 'pinctrl-single' /tmp/live.dts | head -60

# تحقق من الـ properties المطلوبة
grep -E 'register-width|function-mask|bit-per-mux|#pinctrl-cells' /tmp/live.dts

# تحقق من الـ compatible string
grep 'compatible.*pinctrl' /tmp/live.dts

# قارن مع الـ of_match table في الكود:
# "ti,am437-padconf", "ti,am654-padconf", "ti,dra7-padconf"
# "ti,omap3-padconf", "ti,omap4-padconf", "ti,omap5-padconf"
# "ti,j7200-padconf", "pinctrl-single", "pinconf-single"
# "marvell,pxa1908-padconf"

# تحقق من وجود الـ child nodes (functions)
grep -A 5 'pinctrl-single,pins' /tmp/live.dts | head -40

# تحقق إن #pinctrl-cells موجود (مطلوب لأي consumer)
grep '#pinctrl-cells' /tmp/live.dts

# تحقق من الـ IRQ configuration
grep -A 5 'interrupt' /tmp/live.dts | grep -A 5 'padconf\|pinctrl'

# لو الـ DT قديم وفيه warning عن #pinctrl-cells
dmesg | grep "please update dts"
# الحل: أضف #pinctrl-cells = <1>; أو <2>; حسب bits-per-mux
```

**مثال DT صحيح:**

```dts
/* AM335x مثال */
am33xx_pinmux: pinmux@44e10800 {
    compatible = "pinctrl-single";
    reg = <0x44e10800 0x0238>;
    #address-cells = <1>;
    #size-cells = <0>;
    #pinctrl-cells = <1>;
    pinctrl-single,register-width = <32>;
    pinctrl-single,function-mask = <0x7f>;

    uart0_pins: uart0-pins {
        pinctrl-single,pins = <
            0x170 0x0  /* offset=0x170, mux_mode=0 (UART0_RX) */
            0x174 0x0  /* offset=0x174, mux_mode=0 (UART0_TX) */
        >;
    };
};
```

---

### Practical Commands

#### Shell Commands جاهزة للنسخ

```bash
# ========== Setup ==========
PINCTRL_DIR="/sys/kernel/debug/pinctrl"
PINCTRL_NAME=$(ls $PINCTRL_DIR | grep -i pinctrl | head -1)
echo "Using: $PINCTRL_DIR/$PINCTRL_NAME"

# ========== عرض كل الـ pins مع قيمها ==========
cat "$PINCTRL_DIR/$PINCTRL_NAME/pins"

# ========== عرض mapping: من يستخدم ماذا ==========
cat "$PINCTRL_DIR/$PINCTRL_NAME/pinmux-pins"

# ========== عرض الـ functions المسجلة ==========
cat "$PINCTRL_DIR/$PINCTRL_NAME/pinmux-functions"

# ========== تفعيل dynamic debug ==========
echo 'module pinctrl_single +pflmt' > /sys/kernel/debug/dynamic_debug/control

# ========== ftrace لتتبع pcs_set_mux ==========
echo 0 > /sys/kernel/tracing/tracing_on
echo 'pcs_set_mux pcs_pinconf_set pcs_pinconf_get' > /sys/kernel/tracing/set_ftrace_filter
echo function > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/tracing_on
# ... شغّل الـ test ...
cat /sys/kernel/tracing/trace
echo 0 > /sys/kernel/tracing/tracing_on
echo nop > /sys/kernel/tracing/current_tracer

# ========== فحص الـ IRQ ==========
cat /proc/interrupts | grep -i pinctrl
# تحقق من الـ IRQ domain
ls /sys/kernel/debug/irq/domains/ 2>/dev/null | grep pinctrl

# ========== قراءة register مباشرة من الـ hardware ==========
# استخرج الـ base address من /proc/iomem
grep pinctrl /proc/iomem
# مثال ناتج: 44e10800-44e10a37 : 44e10800.pinmux

BASE=0x44e10800
# اقرأ PIN_CTRL[0] و PIN_CTRL[1]
devmem2 $BASE w
devmem2 $((BASE + 4)) w

# ========== dump كامل لـ pinctrl registers ==========
BASE=0x44e10800
SIZE=0x238
echo "Dumping pinctrl registers from $BASE, size $SIZE"
dd if=/dev/mem of=/tmp/pinctrl_dump.bin \
    bs=1 count=$((SIZE)) skip=$((BASE)) 2>/dev/null
od -A x -t x4 /tmp/pinctrl_dump.bin

# ========== مقارنة حالة pin قبل وبعد suspend ==========
cat "$PINCTRL_DIR/$PINCTRL_NAME/pins" > /tmp/pins_before_suspend.txt
# ... افعل suspend/resume ...
cat "$PINCTRL_DIR/$PINCTRL_NAME/pins" > /tmp/pins_after_resume.txt
diff /tmp/pins_before_suspend.txt /tmp/pins_after_resume.txt

# ========== تحقق من الـ DT live ==========
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | \
    grep -A 50 'pinctrl-single'

# ========== تحقق إن الـ consumer device مربوط بـ pinctrl ==========
# (مثلاً uart0)
cat /sys/kernel/debug/pinctrl/pinctrl-single/pinmux-pins | grep uart
```

#### تفسير الـ Output

**مثال قراءة `pins`:**

```
pin 0 (44e10800) 44e10800 00000031 pinctrl-single
         ^            ^       ^
         |            |       |
    physical addr   same   raw register value
    (pcs->res->start + offset)
```

- `0x00000031` = binary `0b00110001`
  - bits [2:0] = `001` → **mux_mode = 1** (مثلاً UART)
  - bit [5] = `1` → pull-up enabled (حسب الـ SoC)
  - bit [4] = `1` → receiver enabled

**مثال قراءة `pinmux-pins`:**

```
pin 2 (44e10808): uart0 (GPIO UNCLAIMED) function uart0 group uart0
                   ^         ^
                   |         |
              function name  GPIO controller claim status
```

**مثال قراءة `/proc/interrupts`:**

```
 45:         5      GIC  pinctrl
              ^
              |
         عدد الـ IRQs اللي اتعملت handle
         لو = 0 ومفروض فيه wake events → مشكلة في الـ irq_status_mask
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: AM62x Industrial Gateway — UART مش شغال بعد boot

#### العنوان
**UART3 صامت على AM62x في gateway صناعي بعد power-on**

#### السياق
شركة بتطور industrial IoT gateway بناءً على **AM62x** (Texas Instruments). الـ gateway بيتكلم مع sensors عبر UART3. في بيئة الـ production، الـ UART بيشتغل مع أول boot، بس بعد suspend/resume دوري (لتوفير الطاقة)، الـ UART بيبقى صامت خالص.

#### المشكلة
الـ UART3 TX pin مش بتشيل data بعد resume. الـ driver نفسه مش بيشتكي — بس الـ hardware level silent.

#### التحليل
الـ AM62x بيستخدم `ti,am654-padconf` اللي عنده `flags = PCS_QUIRK_SHARED_IRQ | PCS_CONTEXT_LOSS_OFF`:

```c
static const struct pcs_soc_data pinctrl_single_am654 = {
    .flags = PCS_QUIRK_SHARED_IRQ | PCS_CONTEXT_LOSS_OFF,
    .irq_enable_mask = (1 << 29),
    .irq_status_mask = (1 << 30),
};
```

الـ flag `PCS_CONTEXT_LOSS_OFF` معناه إن الـ pinmux registers بتتمسح لو الـ power domain اتقطع. الكود في:

```c
static int pinctrl_single_suspend_noirq(struct device *dev)
{
    struct pcs_device *pcs = dev_get_drvdata(dev);

    if (pcs->flags & PCS_CONTEXT_LOSS_OFF) {
        int ret;
        ret = pcs_save_context(pcs);  // بيحفظ كل registers
        if (ret < 0)
            return ret;
    }
    return pinctrl_force_sleep(pcs->pctl);
}

static int pinctrl_single_resume_noirq(struct device *dev)
{
    struct pcs_device *pcs = dev_get_drvdata(dev);

    if (pcs->flags & PCS_CONTEXT_LOSS_OFF)
        pcs_restore_context(pcs);  // بيرجّع الـ registers

    return pinctrl_force_default(pcs->pctl);
}
```

المشكلة: الـ DTS كانت بتستخدم `pinctrl-single` generic بدل `ti,am654-padconf`، فـ `pcs->flags` مكنش فيه `PCS_CONTEXT_LOSS_OFF`، وبالتالي `pcs_save_context` ما اتنادتش، والـ registers اتمسحت بعد power domain reset.

#### الحل
**تعديل الـ DTS:**

```dts
/* قبل — غلط */
pinctrl_uart3: pinctrl@11c000 {
    compatible = "pinctrl-single";
    ...
};

/* بعد — صح */
pinctrl_uart3: pinctrl@11c000 {
    compatible = "ti,am654-padconf";
    ...
};
```

**التحقق بعد الحل:**
```bash
# قبل suspend راجع إن الـ save اتعمل
echo mem > /sys/power/state
dmesg | grep pinctrl-single
# المفروض تشوف: "X pins, size Y" بعد resume من غير مشاكل
```

#### الدرس المستفاد
اختيار الـ `compatible` string صح في الـ DTS مش مجرد تسمية — هو اللي بيحدد الـ `pcs_soc_data` ومعاه الـ flags الحرجة زي `PCS_CONTEXT_LOSS_OFF`. أي مسمى غلط يعني context save/restore مش هيحصل، والـ pins هيتمسح بعد power domain loss.

---

### السيناريو 2: Allwinner H616 Android TV Box — I2C لا يُكتشف

#### العنوان
**I2C touchscreen مش بيتكشف على Android TV box بـ H616**

#### السياق
منتج Android TV box بناءً على **Allwinner H616**. الـ box فيه touch panel بتتوصل عبر I2C2. الـ kernel بيبوت بدون errors، بس الـ I2C device مش بيظهر في `i2cdetect`.

#### المشكلة
الـ I2C2 SDA/SCL pins مش configured كـ I2C function — بيتركوا على الـ default GPIO mode.

#### التحليل
في الـ DTS، الـ pinctrl entry كانت كالتالي:

```dts
i2c2_pins: i2c2-pins {
    pinctrl-single,pins = <
        0x48  0x3   /* SDA */
        0x4C  0x3   /* SCL */
    >;
};
```

الـ driver في `pcs_parse_one_pinctrl_entry()` بياخد كل entry وبيحسب الـ offset والـ value:

```c
offset = pinctrl_spec.args[0];   // 0x48
vals[found].reg = pcs->base + offset;
vals[found].val = pinctrl_spec.args[1];  // 0x3
```

المشكلة: الـ `pinctrl-single,register-width` في الـ DTS كانت `16` (16-bit registers)، بس الـ H616 pinmux registers عرضها 32-bit. فـ `pcs->read` و`pcs->write` كانوا بيشتغلوا بـ 16-bit:

```c
switch (pcs->width) {
case 16:
    pcs->read = pcs_readw;   // readw — 16 bit فقط!
    pcs->write = pcs_writew;
    break;
case 32:
    pcs->read = pcs_readl;   // readl — الصح للـ H616
    pcs->write = pcs_writel;
    break;
}
```

بيكتب value صح بس في عرض غلط، والنتيجة إن الـ upper bits الحاملة لـ function select مش بتتعدل.

#### الحل
**تعديل الـ DTS:**
```dts
pinctrl@300b000 {
    compatible = "pinctrl-single";
    reg = <0x0300b000 0x400>;
    /* تصحيح العرض من 16 لـ 32 */
    pinctrl-single,register-width = <32>;
    pinctrl-single,function-mask = <0x7>;
    ...
};
```

**للتحقق:**
```bash
# اقرأ الـ register مباشرة وتحقق إن function select صح
devmem 0x300b048 32
# المفروض تشوف القيمة 0x3 في الـ bits الصح
```

#### الدرس المستفاد
`pinctrl-single,register-width` لازم يتطابق مع عرض الـ hardware registers فعلاً. أي اختلاف بيخلي الـ read-modify-write يشتغل على نص الـ register، وده بيسبب bugs صعبة الاكتشاف لأن الـ driver ما بيرجعش error.

---

### السيناريو 3: STM32MP1 Industrial ECU — SPI corrupted data

#### العنوان
**SPI1 بيكتب data غلط على STM32MP1 في automotive ECU**

#### السياق
ECU بناءً على **STM32MP1** بيتكلم مع external EEPROM عبر SPI1. في الـ production، EEPROM بيقرأ data مش متسقة. الـ SPI clock والـ chip select شغالين، بس الـ MOSI signal corrupted.

#### المشكلة
الـ SPI1_MOSI pin configured بـ drive strength ضعيف جداً (2mA بدل 12mA المطلوب للـ trace الطويل على الـ PCB).

#### التحليل
الـ DTS كانت بتستخدم `pinconf-single` (اللي فيه `PCS_FEAT_PINCONF`):

```dts
spi1_mosi_pin: spi1-mosi-pin {
    pinctrl-single,pins = <0x1C 0x2>;
    pinctrl-single,drive-strength = <0x2 0xF>;  /* value=2mA, mask=0xF */
};
```

الـ driver في `pcs_add_conf2()` بيقرأ الـ property:

```c
static void pcs_add_conf2(struct pcs_device *pcs, struct device_node *np,
                          const char *name, enum pin_config_param param,
                          struct pcs_conf_vals **conf, unsigned long **settings)
{
    unsigned value[2], shift;
    int ret;

    ret = of_property_read_u32_array(np, name, value, 2);
    if (ret)
        return;
    /* set value & mask */
    value[0] &= value[1];   // value[0] = 0x2 & 0xF = 0x2
    shift = ffs(value[1]) - 1;  // ffs(0xF) - 1 = 0
    add_config(conf, param, value[0], 0, 0, value[1]);
    add_setting(settings, param, value[0] >> shift);  // 0x2 >> 0 = 2mA
}
```

المهندس حسب إن الـ mask `0xF` بيعني 4-bit field من bit 0، بس في STM32MP1 الـ drive strength field من bit 4 لـ bit 6 — المفروض الـ mask `0x70` والـ value `0x60` (12mA).

في `pcs_pinconf_set()`:
```c
case PIN_CONFIG_DRIVE_STRENGTH:
    shift = ffs(func->conf[i].mask) - 1;  // ffs(0x70)-1 = 4 (الصح)
    data &= ~func->conf[i].mask;
    data |= (arg << shift) & func->conf[i].mask;  // بيحط القيمة في المكان الصح
    break;
```

لأن الـ mask غلط، الـ bits اتكتبت في المكان الغلط في الـ register، والنتيجة drive strength = 2mA بدل 12mA.

#### الحل
```dts
/* تصحيح الـ mask ليطابق الـ bits الفعلية في STM32MP1 */
spi1_mosi_pin: spi1-mosi-pin {
    pinctrl-single,pins = <0x1C 0x2>;
    /* bit[6:4] = drive strength, 0x6 = 12mA */
    pinctrl-single,drive-strength = <0x60 0x70>;
};
```

**للتحقق:**
```bash
# اقرأ الـ pin register وتأكد من الـ bits
cat /sys/kernel/debug/pinctrl/pinctrl-single.0/pins
# أو
devmem 0x50002000 32  # عنوان pinmux base + offset
```

#### الدرس المستفاد
في `pinctrl-single,drive-strength` الـ parameter الثاني (mask) لازم يغطي بالظبط الـ bits المخصصة للـ feature في الـ hardware register. `pcs_add_conf2` بتعمل AND وبتحسب shift من الـ mask، فأي mask غلط = قيمة في مكان غلط.

---

### السيناريو 4: i.MX8 Custom Board Bring-up — GPIO لا يستجيب

#### العنوان
**GPIO toggle مش شغال على custom board بناءً على i.MX8**

#### السياق
مهندس بيعمل bring-up لـ custom board بناءً على **i.MX8MQ**. بيحاول يتحكم في LED عبر GPIO3_IO14. الـ `gpioset` بيرجع success بس LED مش بيشتغل.

#### المشكلة
الـ GPIO function مش configured — الـ pin لسه على الـ peripheral function الـ default (UART4 TX).

#### التحليل
الـ DTS فيه GPIO range معرّف:

```dts
pinctrl_gpio_led: gpio-led-grp {
    pinctrl-single,gpio-range = <&pcs_range 0x70 1 0x5>;
    /* offset=0x70, npins=1, gpiofunc=0x5 */
};
```

الـ driver في `pcs_add_gpio_func()` بيقرأ الـ range ويحطها في `pcs->gpiofuncs` list:

```c
range->offset = gpiospec.args[0];   // 0x70
range->npins = gpiospec.args[1];    // 1
range->gpiofunc = gpiospec.args[2]; // 0x5
list_add_tail(&range->node, &pcs->gpiofuncs);
```

لما GPIO subsystem بيطلب الـ pin، `pcs_request_gpio()` بتشتغل:

```c
list_for_each_safe(pos, tmp, &pcs->gpiofuncs) {
    frange = list_entry(pos, struct pcs_gpiofunc_range, node);
    if (pin >= frange->offset + frange->npins || pin < frange->offset)
        continue;

    offset = pcs_pin_reg_offset_get(pcs, pin);
    data = pcs->read(pcs->base + offset);
    data &= ~pcs->fmask;
    data |= frange->gpiofunc;   // بيكتب function 0x5
    pcs->write(data, pcs->base + offset);
    break;
}
```

المشكلة: المهندس حسب `pin` هنا هو الـ GPIO number (14)، بس فعلاً هو الـ pin index في الـ pinctrl table المحسوب من `offset / mux_bytes`. offset=0x70 مع register-width=32 (4 bytes) = pin index 28، مش 14.

الـ `frange->offset = 0x70` بيُقارن مع `pin` اللي قيمته 14، فالـ condition `pin < frange->offset` صح دايماً (14 < 112)، فـ range مش بيتطابق ومفيش حاجة بتتكتب.

#### الحل
الـ `pinctrl-single,gpio-range` المفروض تستخدم الـ pin index مش الـ byte offset:

```dts
/* pin index = offset / (register-width/8) = 0x70 / 4 = 28 */
pinctrl_gpio_led: gpio-led-grp {
    pinctrl-single,gpio-range = <&pcs_range 28 1 0x5>;
};
```

**للتحقق:**
```bash
cat /sys/kernel/debug/pinctrl/pinctrl-single.0/gpio-ranges
# المفروض تشوف الـ range صح بعد التعديل
gpioset gpiochip2 14=1  # بعد التعديل المفروض يشتغل
```

#### الدرس المستفاد
في `pcs_add_gpio_func()`، الـ `range->offset` بيتقارن مع الـ pin index (مش الـ byte offset). في الـ DTS لازم تحسب الـ pin index = byte_offset / (register_width_bits / 8) قبل ما تحطه في `gpio-range`.

---

### السيناريو 5: RK3562 IoT Sensor — Wake-up من الـ UART مش شغال

#### العنوان
**الـ system مش بيصحى من suspend عند وصول data على UART2 على RK3562**

#### السياق
IoT sensor node بناءً على **RK3562** بيستخدم UART2 كـ wake-up source لما يجي command من الـ cloud. الـ system بيدخل suspend ويصحى كل 30 ثانية بدل ما يصحى فور وصول data.

#### المشكلة
الـ wake-up interrupt من الـ UART2 RX pin مش بيصل للـ CPU أثناء suspend.

#### التحليل
الـ RK3562 بيستخدم `pinctrl-single` generic (مش TI-specific)، فـ `pcs_soc_data` فيها:

```c
static const struct pcs_soc_data pinctrl_single = {
};
/* irq_enable_mask = 0, irq_status_mask = 0 */
```

الـ `pcs_irq_init_chained_handler()` بتتحقق:

```c
if (!pcs_soc->irq_enable_mask || !pcs_soc->irq_status_mask) {
    pcs_soc->irq = -1;
    return -EINVAL;  // الـ IRQ handler ما اتسجلش!
}
```

فحتى لو الـ DTS فيه `interrupts` property، الـ IRQ subsystem مش هيشتغل لأن الـ masks صفر.

لو المهندس عاوز wake-up IRQ عبر `pinctrl-single`، محتاج يعمل custom SoC data مع الـ masks الصح، أو يستخدم الـ wake-up عبر GPIO irq chip منفصل.

**ترتيب الكود:**
```
pcs_probe()
  → pcs->socdata.irq = irq_of_parse_and_map(np, 0)
  → if (PCS_HAS_IRQ) → pcs_irq_init_chained_handler()
      → if (!irq_enable_mask) → return -EINVAL  ← هنا الفشل
  → dev_warn: "initialized with no interrupts"
```

الـ `dev_warn` ده بيتطبع في `dmesg` بس المهندسين كتير بيتجاهلوه.

#### الحل
**الخيار 1:** استخدم GPIO irq chip للـ wake-up بشكل منفصل عن `pinctrl-single`:

```dts
uart2_rx_gpio: uart2-rx-gpio {
    gpios = <&gpio3 12 GPIO_ACTIVE_HIGH>;
    interrupt-parent = <&gpio3>;
    interrupts = <12 IRQ_TYPE_EDGE_RISING>;
    wakeup-source;
};
```

**الخيار 2:** لو الـ SoC عنده wake-up bits في الـ pinmux registers (زي OMAP)، عرّف custom `pcs_soc_data`:

```c
/* في الـ platform driver أو DTS overlay */
static const struct pcs_soc_data rk3562_padconf = {
    .flags = PCS_QUIRK_SHARED_IRQ,
    .irq_enable_mask = BIT(X),   /* حسب الـ TRM */
    .irq_status_mask = BIT(Y),
};
```

**للتشخيص:**
```bash
dmesg | grep "pinctrl-single"
# لو شايف: "initialized with no interrupts" → الـ IRQ handler ما اتسجلش
cat /proc/interrupts | grep pinctrl
# لو مفيش سطر → مفيش IRQ متسجل
```

#### الدرس المستفاد
`pinctrl-single` كـ generic driver مش بيدعم wake-up IRQs إلا لو الـ SoC-specific `pcs_soc_data` فيها `irq_enable_mask` و`irq_status_mask` صح. الـ `dev_warn("initialized with no interrupts")` في `pcs_probe` هو early warning مهم — متتجاهلوش. للـ SoCs اللي ليها wake-up bits منفصلة في الـ pinmux register، الحل الصح هو تعريف custom `pcs_soc_data` مع الـ masks الصح.
## Phase 7: مصادر ومراجع

---

### مقالات LWN.net الأساسية

دي أهم المقالات اللي بتشرح نشأة الـ pinctrl subsystem وتطوره، معظمها كتبها Linus Walleij نفسه أو بيتكلم عن patches قدّمها:

| المقال | الأهمية |
|--------|---------|
| [drivers: create a pin control subsystem](https://lwn.net/Articles/463335/) | الـ patch الأساسية اللي أدخلت الـ pinctrl subsystem للكيرنل |
| [drivers: create a pin control subsystem v8](https://lwn.net/Articles/460768/) | الإصدار الثامن من الـ patch series — يوضح التطور قبل الـ merge |
| [pin controller subsystem v7](https://lwn.net/Articles/459190/) | مرحلة تطوير مبكرة — مفيد لفهم التصميم الأصلي |
| [The pin control subsystem](https://lwn.net/Articles/468759/) | مقال تعريفي بالـ subsystem بعد إدخاله — نقطة بداية ممتازة |
| [Documentation/pinctrl.txt](https://lwn.net/Articles/465077/) | الـ documentation الرسمية على LWN — تغطي الـ pinmux conventions |
| [pinctrl: add a pin config interface](https://lwn.net/Articles/471826/) | إضافة الـ pin configuration API فوق الـ pinmux |
| [pinctrl: add a generic pin config interface](https://lwn.net/Articles/467269/) | النسخة الأولى من الـ generic pinconf |
| [[PATCH] pinctrl: Add generic pinctrl-simple driver](https://lwn.net/Articles/496075/) | الـ patch اللي قدّمت `pinctrl-single` كـ generic driver للـ OMAP2+ padconf |
| [drivers/pinctrl: Add the concept of an "init" state](https://lwn.net/Articles/615322/) | إضافة الـ `init` state — مهمة لفهم دورة حياة الـ pin states |
| [Beaglebone black device tree](https://lwn.net/Articles/730224/) | مثال عملي على استخدام `pinctrl-single` في device tree حقيقي |
| [pinctrl: introduce the concept of a GPIO pin function category](https://lwn.net/Articles/1031226/) | تطور حديث في الـ GPIO/pinctrl integration |

---

### الـ Official Kernel Documentation

الملفات دي موجودة في شجرة الكيرنل مباشرةً:

```
Documentation/driver-api/pin-control.rst
    — التوثيق الرسمي الكامل للـ pinctrl subsystem
    — بيشرح: pin enumeration, pin groups, pinmux, pinconf, GPIO ranges

Documentation/devicetree/bindings/pinctrl/pinctrl-single.yaml
    — الـ DT binding الرسمية لـ pinctrl-single
    — بتشرح كل الـ properties: pinctrl-single,pins / ,bits / ,function-mask إلخ

Documentation/devicetree/bindings/pinctrl/pinctrl.yaml
    — الـ base binding اللي بيورثها أي pinctrl driver

include/linux/pinctrl/pinctrl.h
include/linux/pinctrl/pinmux.h
include/linux/pinctrl/pinconf.h
include/linux/pinctrl/pinconf-generic.h
    — الـ public API headers — أول حاجة تقرأها لو بتكتب driver جديد

include/linux/platform_data/pinctrl-single.h
    — الـ platform_data struct الخاصة بـ pinctrl-single

drivers/pinctrl/pinctrl-single.c
    — الـ source الرئيسي المدروس في هذه السلسلة

drivers/pinctrl/core.c / core.h
    — قلب الـ subsystem — registration, lookup, state machine
```

الـ online version على kernel.org:
- [PINCTRL subsystem — kernel.org v4.14](https://www.kernel.org/doc/html/v4.14/driver-api/pinctl.html)
- [pinctrl.txt — kernel.org](https://www.kernel.org/doc/Documentation/pinctrl.txt)

---

### كوميتات مهمة في تاريخ الـ Driver

| الكوميت | الوصف |
|---------|-------|
| `a8717d3` (approx. v3.4) | الإضافة الأولى لـ `pinctrl-single.c` — Tony Lindgren / TI |
| `pinctrl-single,bits` patch | [إضافة دعم multi-pin registers](https://linux.kernel.narkive.com/sgeALyM1/patch-v2-pinctrl-pinctrl-single-add-pinctrl-single-bits-type-of-mux) — Peter Ujfalusi |
| OMAP padconf macros | [نقل الـ macros لـ SoC-specific headers](https://patchwork.kernel.org/project/linux-omap/patch/1447390457-1158-2-git-send-email-javier@osg.samsung.com/) |
| OMAP dynamic pinctrl | [ARM: OMAP2+: dynamic pinctrl handling](https://linux-omap.vger.kernel.narkive.com/dUopo7KP/patch-0-3-arm-omap2-omap-device-add-dynamic-pinctrl-handling) |

للبحث في تاريخ الكوميتات مباشرة:
```bash
git log --oneline drivers/pinctrl/pinctrl-single.c
git log --oneline --follow -- drivers/pinctrl/pinctrl-single.c
```

---

### نقاشات Mailing List

| الرابط | الموضوع |
|--------|---------|
| [PATCH v2: pinctrl-single,bits type of mux](https://linux.kernel.narkive.com/sgeALyM1/patch-v2-pinctrl-pinctrl-single-add-pinctrl-single-bits-type-of-mux) | إضافة الـ `pinctrl-single,bits` لدعم registers بتتحكم في أكتر من pin |
| [PATCH: PINCTRL selectable by defconfig](https://lists.gt.net/linux/kernel/1839735) | نقاش جعل الـ PINCTRL قابلاً للتفعيل من الـ menuconfig |
| [linux-omap mailing list archive](https://www.spinics.net/lists/kernel/) | الـ archive العام لـ LKML للبحث عن نقاشات pinctrl-single |
| [LKML.ORG](https://lkml.org/) | الأرشيف الكامل لـ Linux Kernel Mailing List |

---

### KernelNewbies — تغييرات الـ Subsystem عبر الإصدارات

الصفحات دي بتوثق التغييرات في الـ pinctrl subsystem مع كل kernel release:

| الإصدار | الرابط |
|---------|--------|
| Linux 6.11 | [kernelnewbies.org/Linux_6.11](https://kernelnewbies.org/Linux_6.11) |
| Linux 6.13 | [kernelnewbies.org/Linux_6.13](https://kernelnewbies.org/Linux_6.13) |
| Linux 6.14 | [kernelnewbies.org/Linux_6.14](https://kernelnewbies.org/Linux_6.14) |
| Linux 6.15 | [kernelnewbies.org/Linux_6.15](https://kernelnewbies.org/Linux_6.15) |
| Linux 6.16 | [kernelnewbies.org/Linux_6.16](https://kernelnewbies.org/Linux_6.16) |

ابحث في الصفحة عن كلمة **pinctrl** عشان تطلع التغييرات المتعلقة بالـ subsystem في كل إصدار.

---

### eLinux.org — أمثلة عملية

| الصفحة | المحتوى |
|--------|---------|
| [EBC Device Trees](https://elinux.org/EBC_Device_Trees) | أمثلة device tree حقيقية بتستخدم `pinctrl-single,pins` لتعريف الـ GPIO |
| [AM335x recovery](https://elinux.org/AM335x_recovery) | ضبط الـ pinctrl على TI AM335x — نفس العائلة اللي اتصمم ليها الـ driver |
| [Pin Control & GPIO Update (PDF)](https://elinux.org/images/c/cd/Pincontrol-gpio-update.pdf) | عرض تقديمي شامل عن الـ pinctrl/GPIO subsystems — مفيد جداً للفهم البصري |
| [BeagleBoard Cape Compatibility](https://elinux.org/BeagleBoard/GSoC/2020_Projects/Cape_Compatibility) | استخدام `pinctrl-single` مع DRA7XX_CORE_IOPAD() macros |
| [i2c-demux-pinctrl Tests](https://www.elinux.org/Tests:i2c-demux-pinctrl) | اختبار الـ i2c demux مع pinctrl API |

---

### مصادر إضافية متخصصة

| المصدر | الرابط |
|--------|--------|
| Embedded.com — pinctrl subsystem overview | [linux-device-driver-development-the-pin-control-subsystem](https://www.embedded.com/linux-device-driver-development-the-pin-control-subsystem/) |
| STM32MPU Pinctrl Overview | [wiki.st.com/stm32mpu/wiki/Pinctrl_overview](https://wiki.st.com/stm32mpu/wiki/Pinctrl_overview) |
| Linux Device Tree Pinctrl Tutorial | [blog.modest-destiny.com](https://blog.modest-destiny.com/posts/linux-device-tree-pinctrl-tutorial/) |
| TI OMAP pinctrl header | [github.com/torvalds/linux — omap.h](https://github.com/torvalds/linux/blob/master/include/dt-bindings/pinctrl/omap.h) |

---

### الكتب المرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 9**: Data Types in the Kernel — بيتكلم عن الـ `__iomem` و memory-mapped registers اللي بيعتمد عليها `pinctrl-single` بشكل كامل.
- **الفصل 10**: Interrupt Handling — ضروري لفهم الـ interrupt chaining اللي بيستخدمه `pcs_irq_chain_handler`.
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 17**: Devices and Modules — يشرح الـ platform_device / platform_driver model اللي بيقوم عليه `pinctrl-single`.
- **الفصل 13**: The Virtual Filesystem — مفيد لفهم الـ `debugfs` integration في الـ pinctrl core.

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 14**: Kernel Initialization — بيشرح إزاي الـ board init code بيتكامل مع الـ pinctrl subsystem.
- **الفصل 15**: Bootstrapping Your Custom Board — مثال عملي على كتابة device tree مع `pinctrl-single`.

#### Mastering Embedded Linux Programming — Frank Vasquez & Chris Simmonds (3rd Edition)
- **الفصل 12**: Prototyping with Breakout Boards — بيشرح الـ pinctrl من منظور عملي مع BeagleBone.

---

### مصطلحات البحث

لو عايز تعمق أكتر، استخدم الكلمات دي في بحثك:

```
pinctrl-single driver linux kernel
pinctrl-single device tree binding
pinctrl subsystem linux Linus Walleij
OMAP padconf pinmux linux
pinctrl-single,pins property
pinctrl-single,bits multi-pin register
pinctrl generic pinconf
pin_config_param kernel
pcs_function pinctrl-single internals
TI AM335x AM437x pinctrl device tree
BeagleBone Black pinmux configuration
pinctrl irq chaining linux
platform_data pinctrl-single
debugfs pinctrl linux
```

---

### ملخص سريع للمصادر حسب الأولوية

```
للبداية السريعة:
  → "The pin control subsystem" على LWN: https://lwn.net/Articles/468759/
  → PDF عرض eLinux: https://elinux.org/images/c/cd/Pincontrol-gpio-update.pdf

لفهم الـ patch الأصلية:
  → https://lwn.net/Articles/496075/  (pinctrl-single introduction)
  → https://lwn.net/Articles/463335/  (pinctrl subsystem creation)

للتوثيق الرسمي:
  → Documentation/driver-api/pin-control.rst
  → Documentation/devicetree/bindings/pinctrl/pinctrl-single.yaml

لأمثلة device tree حقيقية:
  → https://elinux.org/EBC_Device_Trees
  → https://lwn.net/Articles/730224/  (BeagleBone Black DT)

للمتابعة المستمرة:
  → https://kernelnewbies.org  (ابحث عن pinctrl في كل release)
  → https://lkml.org           (تابع linux-gpio mailing list)
```
## Phase 8: Writing simple module

### الفكرة

**`pcs_set_mux`** هي الـ function الأساسية في `pinctrl-single` اللي بتكتب فعلياً في الـ hardware registers لما أي driver يطلب تفعيل function معينة على pin (مثلاً: تحويل pin من GPIO لـ UART TX). هنستخدم **kprobe** عشان نراقب كل مرة بيتعمل فيها mux switch ونطبع اسم الـ function والـ group اللي اتاختار.

---

### الـ Module

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on pcs_set_mux() — pinctrl-single mux hook
 *
 * Fires every time pinctrl-single programs a mux register,
 * printing the function selector and group number.
 */
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/kprobes.h>
#include <linux/pinctrl/pinctrl.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe monitor for pcs_set_mux in pinctrl-single");

/*
 * pcs_set_mux signature:
 *   static int pcs_set_mux(struct pinctrl_dev *pctldev,
 *                          unsigned fselector, unsigned group);
 *
 * pt_regs layout on x86-64:
 *   di = pctldev, si = fselector, dx = group
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Read args from CPU registers before the real function runs */
    unsigned long fselector = regs->si;   /* 2nd arg */
    unsigned long group     = regs->dx;   /* 3rd arg */

    pr_info("[pcs_mux_probe] set_mux called: fselector=%lu  group=%lu\n",
            fselector, group);

    return 0; /* 0 = let the original function execute normally */
}

/* kprobe struct — symbol_name points to the function we want to hook */
static struct kprobe kp = {
    .symbol_name = "pcs_set_mux",
    .pre_handler = handler_pre,
};

static int __init pcs_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[pcs_mux_probe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[pcs_mux_probe] kprobe planted at %p (%s)\n",
            kp.addr, kp.symbol_name);
    return 0;
}

static void __exit pcs_probe_exit(void)
{
    /* Must unregister before module unloads — otherwise kernel
     * will jump into freed memory when pcs_set_mux is called */
    unregister_kprobe(&kp);
    pr_info("[pcs_mux_probe] kprobe removed\n");
}

module_init(pcs_probe_init);
module_exit(pcs_probe_exit);
```

---

### شرح كل جزء

#### الـ Includes

```c
#include <linux/kprobes.h>
#include <linux/pinctrl/pinctrl.h>
```

**الـ** `kprobes.h` بيجيب تعريف `struct kprobe` وفانكشنز `register_kprobe` / `unregister_kprobe`. **الـ** `pinctrl/pinctrl.h` مش إلزامي هنا بس بيوضح الـ types المستخدمة في الـ function الأصلية للتوثيق.

---

#### الـ `handler_pre`

```c
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
```

بيتشغل **قبل** تنفيذ `pcs_set_mux` بالظبط. **الـ** `pt_regs` بيحتوي على state الـ CPU registers في لحظة الـ probe — ومنه بناخد `si` (fselector) و`dx` (group) اللي هما الـ argument التاني والتالت على x86-64 حسب الـ System V ABI.

الـ return value صفر معناه "اكمل تنفيذ الـ function الأصلية" — لو رجعنا قيمة تانية الـ kernel هيـ skip الـ function وده خطر.

---

#### الـ `struct kprobe`

```c
static struct kprobe kp = {
    .symbol_name = "pcs_set_mux",
    .pre_handler = handler_pre,
};
```

الـ `.symbol_name` بيخلي الـ kernel يحل الـ address وقت الـ register — مش محتاج نحط عنوان ثابت. لو الـ symbol مش exported أو مش موجود هيرجع error في `register_kprobe`.

---

#### الـ `module_init`

```c
ret = register_kprobe(&kp);
```

بيزرع الـ breakpoint في أول instruction من `pcs_set_mux`. من اللحظة دي كل call لـ `pcs_set_mux` في أي driver في الـ system هتعدي على الـ `handler_pre` الأول.

---

#### الـ `module_exit`

```c
unregister_kprobe(&kp);
```

ضروري جداً — لو الـ module اتـ unload من غير ما يتعمل unregister، الـ kernel هيكمل يقفز على عنوان الـ handler اللي بقى freed memory وده crash مضمون.

---

### Makefile للتجميع

```makefile
obj-m += pcs_mux_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

---

### تشغيل وملاحظة الـ output

```bash
# تحميل الـ module
sudo insmod pcs_mux_probe.ko

# مشاهدة الـ log في real-time
sudo dmesg -w | grep pcs_mux_probe

# تفريغ الـ module
sudo rmmod pcs_mux_probe
```

مثال على الـ output لما أي device يطلب تغيير الـ pin function (مثلاً عند boot أو عند فتح UART):

```
[pcs_mux_probe] kprobe planted at ffffffffc0a12340 (pcs_set_mux)
[pcs_mux_probe] set_mux called: fselector=0  group=0
[pcs_mux_probe] set_mux called: fselector=2  group=1
[pcs_mux_probe] set_mux called: fselector=5  group=3
[pcs_mux_probe] kprobe removed
```

كل سطر بيمثل عملية كتابة فعلية في الـ mux register على الـ SoC — الـ `fselector` بيشير لاسم الـ function في الـ pinctrl framework والـ `group` للـ pin group المتأثر.
