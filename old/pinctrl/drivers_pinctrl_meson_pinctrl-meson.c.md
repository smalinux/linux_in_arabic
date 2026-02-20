# شرح `pinctrl-meson.c` — Real-World Driver Implementation

هذا أهم ملف شرحناه حتى الآن — ليس header أو interface، بل **driver حقيقي** يُطبّق كل المفاهيم التي تعلمناها على hardware فعلي من Amlogic (مستخدم في أجهزة Android TV وRaspberry Pi بديلة).

---

## أولاً: معمارية الـ Hardware — كيف تنظّم Amlogic الـ Pins

### الـ Banks

الـ pins مُنظَّمة في **banks** (مجموعات) بأحرف: A, B, C, X, Y, Z, AO, BOOT, CARD...

الـ **AO bank** خاص جداً — ينتمي لـ **Always-On power domain** لا يمكن إيقافه أبداً، ويستخدم registers مختلفة عن باقي الـ banks.

### الـ Register Ranges — 4 مجموعات registers لكل pin controller

```
┌─────────────────────────────────────────────────────────────┐
│             Meson Pin Controller Registers                  │
│                                                             │
│  reg_mux    ──► تحديد وظيفة الـ pin (MUX)                  │
│  reg_pullen ──► تفعيل/تعطيل الـ pull resistor              │
│  reg_pull   ──► اتجاه الـ pull (up أو down)                │
│  reg_gpio   ──► direction + output value + input value     │
│  reg_ds     ──► drive strength (اختياري، ليس في كل SoCs)  │
└─────────────────────────────────────────────────────────────┘

ملاحظة: في بعض الـ SoCs:
- reg_pull و reg_pullen نفس الـ register (meson8 AO bank)
- في Meson G12A: reg_gpio = reg_pull = reg_pullen (register واحد لكل شيء)
- في Meson A1: reg_gpio = reg_pull = reg_pullen = reg_ds
```

---

## ثانياً: `meson_bit_strides[]` — لماذا؟

```c
static const unsigned int meson_bit_strides[] = {
    1, 1, 1, 1, 1, 2, 1
};
```

معظم الـ configs تحتاج **bit واحد** per pin (0 أو 1). لكن `MESON_REG_DS` (drive strength) يحتاج **2 bits** per pin لأن له 4 قيم ممكنة (500µA, 2500µA, 3000µA, 4000µA).

الـ index يتوافق مع `enum meson_reg_type`:

```
index 0: REG_PULLEN  → stride 1
index 1: REG_PULL    → stride 1
index 2: REG_DIR     → stride 1
index 3: REG_OUT     → stride 1
index 4: REG_IN      → stride 1
index 5: REG_DS      → stride 2  ← 2 bits per pin
index 6: REG_MUX     → stride 1
```

---

## ثالثاً: `meson_calc_reg_and_bit()` — قلب الحسابات

هذه الدالة هي **الأهم في الـ driver** — تحوّل رقم الـ pin إلى عنوان register وبت محدد:

```c
static void meson_calc_reg_and_bit(const struct meson_bank *bank,
                                   unsigned int pin,
                                   enum meson_reg_type reg_type,
                                   unsigned int *reg,
                                   unsigned int *bit)
{
    const struct meson_reg_desc *desc = &bank->regs[reg_type];

    *bit = (desc->bit + pin - bank->first) * meson_bit_strides[reg_type];
    *reg = (desc->reg + (*bit / 32)) * 4;
    *bit &= 0x1f;
}
```

```
مثال: Bank X، pins 50..81، reg_type=PULL، desc={reg=2, bit=0}
نريد pin 55:

bit  = (0 + 55 - 50) * 1 = 5
reg  = (2 + (5 / 32)) * 4 = (2 + 0) * 4 = 8   ← byte offset = 0x08
bit  = 5 & 0x1f = 5

→ اكتب/اقرأ bit 5 في register عند offset 0x08 من reg_pull base

مثال مع DS (stride=2): pin 55
bit  = (0 + 55 - 50) * 2 = 10
reg  = (2 + (10 / 32)) * 4 = 8
bit  = 10 & 0x1f = 10      ← يشغل bits [11:10] (2 bits)
```

```
لماذا (* 4) في حساب reg؟
  الـ register أعراضها 32-bit = 4 bytes
  كل 32 bit = register واحد
  إذاً: byte_offset = register_index * 4

لماذا (& 0x1f) في حساب bit؟
  0x1f = 31 = max bit index في 32-bit register
  هذا يعطي الـ bit الفعلي داخل الـ register بعد تجاوز الـ 32
```

---

## رابعاً: `meson_pctrl_ops` — تنفيذ الـ pinctrl_ops

```c
static const struct pinctrl_ops meson_pctrl_ops = {
    .get_groups_count = meson_get_groups_count,  // من pc->data->num_groups
    .get_group_name   = meson_get_group_name,    // من pc->data->groups[sel].name
    .get_group_pins   = meson_get_group_pins,    // من pc->data->groups[sel].pins
    .dt_node_to_map   = pinconf_generic_dt_node_to_map_all, // ← generic مباشرة!
    .dt_free_map      = pinctrl_utils_free_map,             // ← utility جاهزة
    .pin_dbg_show     = meson_pin_dbg_show,
};
```

لاحظ: `dt_node_to_map` يستخدم `pinconf_generic_dt_node_to_map_all` مباشرة بدون wrapper — هذا مثال مثالي على استخدام الـ generic framework الذي شرحناه.

---

## خامساً: تسلسل الـ Pull Configuration

### `meson_pinconf_enable_bias()` — خطوتان لا خطوة واحدة

```c
static int meson_pinconf_enable_bias(struct meson_pinctrl *pc,
                                     unsigned int pin, bool pull_up)
{
    // الخطوة 1: اضبط الاتجاه في reg_pull
    meson_calc_reg_and_bit(bank, pin, MESON_REG_PULL, &reg, &bit);
    val = pull_up ? BIT(bit) : 0;
    regmap_update_bits(pc->reg_pull, reg, BIT(bit), val);

    // الخطوة 2: فعّل الـ pull في reg_pullen
    meson_calc_reg_and_bit(bank, pin, MESON_REG_PULLEN, &reg, &bit);
    regmap_update_bits(pc->reg_pullen, reg, BIT(bit), BIT(bit));
}
```

```
لماذا register منفصل للـ enable والاتجاه؟

reg_pull:   يحدد up أو down  (0=down, 1=up)
reg_pullen: يُفعّل أو يُعطّل  (0=disabled, 1=enabled)

الترتيب مهم: اضبط الاتجاه أولاً ثم فعّل
لتجنب لحظة يكون فيها الـ pull مُفعَّلاً بدون اتجاه محدد
```

### `meson_pinconf_get_pull()` — القراءة

```c
// 1. اقرأ reg_pullen: هل الـ pull مُفعَّل أصلاً؟
if (!(val & BIT(bit)))
    return PIN_CONFIG_BIAS_DISABLE;

// 2. لو مُفعَّل، اقرأ reg_pull: ما الاتجاه؟
if (val & BIT(bit))
    return PIN_CONFIG_BIAS_PULL_UP;
else
    return PIN_CONFIG_BIAS_PULL_DOWN;
```

---

## سادساً: Drive Strength — 2-bit Encoding

```c
static int meson_pinconf_set_drive_strength(struct meson_pinctrl *pc,
                                            unsigned int pin,
                                            u16 drive_strength_ua)
{
    if (!pc->reg_ds)
        return -ENOTSUPP;  // ليس كل SoCs تدعمه

    // تحويل القيمة بالـ µA إلى 2-bit encoding
    if      (drive_strength_ua <= 500)  ds_val = MESON_PINCONF_DRV_500UA;  // 0b00
    else if (drive_strength_ua <= 2500) ds_val = MESON_PINCONF_DRV_2500UA; // 0b01
    else if (drive_strength_ua <= 3000) ds_val = MESON_PINCONF_DRV_3000UA; // 0b10
    else if (drive_strength_ua <= 4000) ds_val = MESON_PINCONF_DRV_4000UA; // 0b11
    else {
        dev_warn_once(...); // تحذير مرة واحدة فقط لنفس الـ pin
        ds_val = MESON_PINCONF_DRV_4000UA; // fallback للأقصى
    }

    // الكتابة بـ mask محدد (2 bits)
    regmap_update_bits(pc->reg_ds, reg, 0x3 << bit, ds_val << bit);
}
```

```
reg_ds layout لـ pin 0..15 في bank واحد:

bit:  31..30  29..28  ...  3..2   1..0
     ┌──────┬──────┬─────┬──────┬──────┐
     │pin15 │pin14 │ ... │ pin1 │ pin0 │
     └──────┴──────┴─────┴──────┴──────┘
     2bits   2bits         2bits  2bits

mask = 0x3 << bit  ← يغطي الـ 2 bits المناسبين فقط
```

---

## سابعاً: `meson_pinconf_set()` و `meson_pinconf_get()` — الـ Main Callbacks

### الـ set — لاحظ التصميم المزدوج

```c
static int meson_pinconf_set(struct pinctrl_dev *pcdev, unsigned int pin,
                             unsigned long *configs, unsigned num_configs)
{
    for (i = 0; i < num_configs; i++) {
        param = pinconf_to_config_param(configs[i]);

        // Pass 1: استخرج الـ argument للـ params التي تحتاجه
        switch (param) {
        case PIN_CONFIG_DRIVE_STRENGTH_UA:
        case PIN_CONFIG_OUTPUT_ENABLE:
        case PIN_CONFIG_LEVEL:
            arg = pinconf_to_config_argument(configs[i]);
            break;
        default: break;
        }

        // Pass 2: نفّذ الـ config
        switch (param) {
        case PIN_CONFIG_BIAS_DISABLE:  → meson_pinconf_disable_bias()
        case PIN_CONFIG_BIAS_PULL_UP:  → meson_pinconf_enable_bias(true)
        case PIN_CONFIG_BIAS_PULL_DOWN:→ meson_pinconf_enable_bias(false)
        case PIN_CONFIG_DRIVE_STRENGTH_UA: → set_drive_strength(arg)
        case PIN_CONFIG_OUTPUT_ENABLE: → set_output(arg)
        case PIN_CONFIG_LEVEL:         → set_output_drive(arg)
        default: return -ENOTSUPP;
        }
    }
}
```

لماذا Pass مزدوج؟ لأن `BIAS_*` params لا تحتاج argument (الـ bit mask يُتجاهل قيمته)، بينما `DRIVE_STRENGTH_UA` و`LEVEL` تحتاجه. التصميم نظيف ويتجنب استدعاء `pinconf_to_config_argument` غير الضروري.

### الـ get — قيمة 60000 الغريبة

```c
case PIN_CONFIG_BIAS_PULL_UP:
    if (meson_pinconf_get_pull(pc, pin) == param)
        arg = 60000;  // ← لماذا 60000؟
    else
        return -EINVAL;
```

هذا يعني أن مقاومة الـ pull-up الداخلية في Meson هي تقريباً **60kΩ**. الـ generic framework يفسر الـ argument في `PIN_CONFIG_BIAS_PULL_UP` كقيمة بالـ Ohm. هذا ليس hardcoded عشوائياً — هو قيمة حقيقية من الـ datasheet.

---

## ثامناً: التكامل مع GPIO Subsystem

### الـ gpio_chip مُبنية على نفس الـ pinconf functions:

```c
// direction_input ← يستخدم set_output بـ false
meson_gpio_direction_input  → meson_pinconf_set_output(pc, gpio, false)

// direction_output ← يضبط direction + value معاً
meson_gpio_direction_output → meson_pinconf_set_output_drive(pc, gpio, value)

// set ← يكتب قيمة output فقط بدون تغيير direction
meson_gpio_set              → meson_pinconf_set_drive(pc, gpio, value)

// get ← يقرأ من reg_gpio (input value)
meson_gpio_get              → regmap_read(reg_gpio) للـ MESON_REG_IN
```

```
العلاقة بين pinconf و gpio في Meson:

reg_gpio contains:
  MESON_REG_DIR ← bit per pin: 0=output, 1=input
  MESON_REG_OUT ← bit per pin: output value
  MESON_REG_IN  ← bit per pin: input value (read-only من الـ hardware)

لاحظ: MESON_REG_DIR يُعكس (inverted)!
  meson_pinconf_set_output(out=true)  → يكتب 0 في DIR bit
  meson_pinconf_get_output()          → يقرأ DIR ثم يعكسه: return !ret
```

---

## تاسعاً: `meson_pinctrl_parse_dt()` — قراءة الـ Resources

```c
static int meson_pinctrl_parse_dt(struct meson_pinctrl *pc)
{
    // يجب أن يوجد gpio node واحد بالضبط
    chips = gpiochip_node_count(pc->dev);
    if (!chips || chips > 1) return -EINVAL;

    pc->fwnode = gpiochip_node_get_first(pc->dev);
    gpio_np    = to_of_node(pc->fwnode);

    // إلزامي: mux و gpio
    pc->reg_mux  = meson_map_resource(pc, gpio_np, "mux");   // must have
    pc->reg_gpio = meson_map_resource(pc, gpio_np, "gpio");  // must have

    // اختياري: pull و pull-enable
    pc->reg_pull   = meson_map_resource(pc, gpio_np, "pull");
    pc->reg_pullen = meson_map_resource(pc, gpio_np, "pull-enable");

    // اختياري: drive-strength (فقط بعض الـ SoCs)
    pc->reg_ds = meson_map_resource(pc, gpio_np, "ds");
    if (IS_ERR(pc->reg_ds)) pc->reg_ds = NULL;  // لا error لو غير موجود

    // SoC-specific customization
    if (pc->data->parse_dt)
        return pc->data->parse_dt(pc);
}
```

### الـ SoC Variants — كيف يتعامل الـ driver مع الاختلافات

```c
// meson8 AO bank: pull-enable = pull (نفس الـ register)
int meson8_aobus_parse_dt_extra(struct meson_pinctrl *pc) {
    pc->reg_pullen = pc->reg_pull;
}

// Meson A1: كل شيء في reg_gpio
int meson_a1_parse_dt_extra(struct meson_pinctrl *pc) {
    pc->reg_pull   = pc->reg_gpio;
    pc->reg_pullen = pc->reg_gpio;
    pc->reg_ds     = pc->reg_gpio;
}
```

هذا **النمط الصحيح** للتعامل مع hardware variants — بدلاً من `#ifdef` أو switch كبير، كل SoC يُعدّل الـ pointers في وقت الـ probe.

---

## عاشراً: `meson_pinctrl_probe()` — كل شيء معاً

```c
int meson_pinctrl_probe(struct platform_device *pdev)
{
    // 1. تخصيص الـ main struct
    pc = devm_kzalloc(dev, sizeof(*pc), GFP_KERNEL);
    pc->dev  = dev;
    pc->data = of_device_get_match_data(dev);  // بيانات SoC المحدد

    // 2. قراءة الـ DT و mapping الـ registers
    meson_pinctrl_parse_dt(pc);

    // 3. تعبئة الـ pinctrl_desc وتسجيله
    pc->desc.name    = "pinctrl-meson";
    pc->desc.pctlops = &meson_pctrl_ops;
    pc->desc.pmxops  = pc->data->pmx_ops;    // ← من الـ SoC-specific data
    pc->desc.confops = &meson_pinconf_ops;
    pc->desc.pins    = pc->data->pins;
    pc->desc.npins   = pc->data->num_pins;

    pc->pcdev = devm_pinctrl_register(pc->dev, &pc->desc, pc);

    // 4. تسجيل الـ GPIO chip
    return meson_gpiolib_register(pc);
}
```

---

## الصورة الكاملة للـ Driver

```
meson_pinctrl_probe()
         │
         ├─ meson_pinctrl_parse_dt()
         │       │
         │       ├─ meson_map_resource("mux")      → pc->reg_mux
         │       ├─ meson_map_resource("gpio")     → pc->reg_gpio
         │       ├─ meson_map_resource("pull")     → pc->reg_pull
         │       ├─ meson_map_resource("pull-enable")→ pc->reg_pullen
         │       └─ meson_map_resource("ds")       → pc->reg_ds
         │
         ├─ devm_pinctrl_register(&pc->desc, pc)
         │       │
         │       └─ الـ core يستدعي:
         │            meson_pctrl_ops.get_groups_count()
         │            meson_pctrl_ops.dt_node_to_map() ← generic
         │
         └─ meson_gpiolib_register()
                 │
                 └─ gpiochip_add_data(&pc->chip, pc)

عند طلب GPIO أو pinconf:
  pinctrl core
       │
       ▼
  meson_pinconf_set/get()
       │
       ▼
  meson_get_bank()    → أيّ bank يحتوي هذا الـ pin؟
       │
       ▼
  meson_calc_reg_and_bit() → أيّ register؟ أيّ bit؟
       │
       ▼
  regmap_update_bits(pc->reg_pull/pullen/gpio/ds, reg, mask, val)
       │
       ▼
  Hardware Register تُحدَّث
```

هذا الـ driver نموذج ممتاز للـ BSP driver الصحيح: يستخدم الـ generic framework، يتعامل مع الـ variants بنظافة، ويشارك الـ register logic بين pinconf و GPIO subsystem بدون تكرار.