# شرح `pinctrl-single.c` — الـ Generic One-Register-Per-Pin Driver

هذا الـ driver مختلف جذرياً عن Meson — بدلاً من أن يكون مخصصاً لـ SoC معين، هو **driver عام** يعمل على أي hardware يتبع نمط "register واحد لكل pin". يُستخدم على OMAP، AM33xx، AM437x، DRA7، وغيرها.

---

## المجموعة الأولى: الـ Core Data Structures

### `struct pcs_func_vals`

```c
struct pcs_func_vals {
    void __iomem *reg;  // العنوان الـ virtual للـ register
    unsigned val;       // القيمة المراد كتابتها
    unsigned mask;      // الـ mask (مهم في bits_per_mux mode)
};
```

تُمثّل **عملية كتابة واحدة** في الـ hardware. function كاملة هي مصفوفة من هذه الـ entries — كل entry = "اكتب `val` في `reg` بعد تطبيق `mask`".

### `struct pcs_conf_vals`

```c
struct pcs_conf_vals {
    enum pin_config_param param;   // نوع الـ config
    unsigned val;                  // القيمة الحالية من الـ DT
    unsigned enable;               // pattern يعني "مُفعَّل"
    unsigned disable;              // pattern يعني "مُعطَّل"
    unsigned mask;                 // الـ bits ذات الصلة فقط
};
```

الفرق المهم بين هذه الـ struct وبين الـ Meson approach: هنا الـ driver لا يعرف أين الـ bits في الـ register — المستخدم يُخبره عبر DT بـ `enable` و`disable` patterns. هذا ما يجعل الـ driver "generic".

### `struct pcs_function`

```c
struct pcs_function {
    const char          *name;   // اسم الـ function
    struct pcs_func_vals *vals;  // مصفوفة reg/val pairs للـ mux
    unsigned             nvals;
    struct pcs_conf_vals *conf;  // مصفوفة config entries
    int                  nconfs;
    struct list_head     node;   // linked list
};
```

هذا تصميم مختلف عن Meson — الـ function تحمل كلاً من الـ mux values والـ conf values معاً، لأن في `pinctrl-single` كل function تُعرَّف في DT كـ node مستقل يحتوي على كليهما.

### `struct pcs_device` — الـ Main Instance

```c
struct pcs_device {
    void __iomem  *base;        // قاعدة registers الـ ioremapped
    void          *saved_vals;  // نسخة احتياطية للـ context loss
    unsigned       width;       // عرض الـ register: 8/16/32 bit
    unsigned       fmask;       // mask للـ function field
    unsigned       fshift;      // shift للـ function field
    unsigned       foff;        // قيمة "function off"
    unsigned       fmax;        // عدد الـ functions الأقصى
    bool           bits_per_mux;// هل register واحد يحتوي على عدة pins؟
    unsigned       bits_per_pin;// عدد bits لكل pin
    unsigned (*read)(void __iomem *reg);   // function pointer للقراءة
    void (*write)(unsigned, void __iomem *);// function pointer للكتابة
};
```

أهم شيء هنا: **function pointers للقراءة والكتابة**. بدلاً من `if (width==32) readl() else readw()`، يُحدَّد الـ function pointer مرة واحدة في الـ probe ثم يُستخدم في كل مكان. أكفأ وأنظف.

---

## المجموعة الثانية: Register Access — `pcs_readb/w/l` و `pcs_writeb/w/l`

```c
static unsigned int pcs_readb(void __iomem *reg)  { return readb(reg); }
static unsigned int pcs_readw(void __iomem *reg)  { return readw(reg); }
static unsigned int pcs_readl(void __iomem *reg)  { return readl(reg); }
static void pcs_writeb(unsigned int val, void __iomem *reg) { writeb(val, reg); }
static void pcs_writew(unsigned int val, void __iomem *reg) { writew(val, reg); }
static void pcs_writel(unsigned int val, void __iomem *reg) { writel(val, reg); }
```

لماذا لا `regmap` مثل Meson؟ التعليق يشرح صراحةً:

> "on omaps, some mux registers are performance critical as they may need to be remuxed every time before and after idle"

الـ regmap يُجري فحصاً لعرض الـ register في كل قراءة/كتابة. على OMAP هذا مكلف لأن الـ remux يحدث كثيراً. لذلك يستخدم function pointers مباشرة — overhead صفري.

---

## المجموعة الثالثة: Pin Offset و Shift Calculations

### `pcs_pin_reg_offset_get()`

```c
static unsigned int pcs_pin_reg_offset_get(struct pcs_device *pcs,
                                           unsigned int pin)
{
    unsigned int mux_bytes = pcs->width / BITS_PER_BYTE;

    if (pcs->bits_per_mux) {
        // وضع: عدة pins في register واحد
        unsigned int pin_offset_bytes;
        pin_offset_bytes = (pcs->bits_per_pin * pin) / BITS_PER_BYTE;
        return (pin_offset_bytes / mux_bytes) * mux_bytes;
    }

    // وضع: register واحد لكل pin (الأبسط)
    return pin * mux_bytes;
}
```

**الوضع العادي** (`bits_per_mux = false`):

```
pin 0 → offset 0
pin 1 → offset 4  (لو width=32bit = 4 bytes)
pin 2 → offset 8
...
```

**وضع `bits_per_mux`** — عدة pins في register واحد:

```
مثال: width=32bit، bits_per_pin=4، إذاً 8 pins في كل register

pin 0 → offset 0   (bits [3:0])
pin 1 → offset 0   (bits [7:4])
...
pin 7 → offset 0   (bits [31:28])
pin 8 → offset 4   (bits [3:0] في الـ register التالي)
```

### `pcs_pin_shift_reg_get()`

```c
static unsigned int pcs_pin_shift_reg_get(struct pcs_device *pcs,
                                          unsigned int pin)
{
    return (pin % (pcs->width / pcs->bits_per_pin)) * pcs->bits_per_pin;
}
```

يحسب **كم bit تُشيف** داخل الـ register للوصول لبيانات الـ pin:

```
مثال: width=32، bits_per_pin=4 → 8 pins per register

pin 0: shift = (0 % 8) * 4 = 0
pin 1: shift = (1 % 8) * 4 = 4
pin 3: shift = (3 % 8) * 4 = 12
pin 8: shift = (8 % 8) * 4 = 0  ← pin الأول في الـ register التالي
```

---

## المجموعة الرابعة: الـ pinctrl_ops و pinmux_ops

### `pcs_pinctrl_ops`

```c
static const struct pinctrl_ops pcs_pinctrl_ops = {
    .get_groups_count = pinctrl_generic_get_group_count, // ← generic!
    .get_group_name   = pinctrl_generic_get_group_name,  // ← generic!
    .get_group_pins   = pinctrl_generic_get_group_pins,  // ← generic!
    .pin_dbg_show     = pcs_pin_dbg_show,
    .dt_node_to_map   = pcs_dt_node_to_map,              // ← custom
    .dt_free_map      = pcs_dt_free_map,                 // ← custom
};
```

الـ get_groups_* تستخدم `pinctrl_generic_*` — يعني الـ core يُدير جدول الـ groups تلقائياً وهذا الـ driver يضيف إليه عبر `pinctrl_generic_add_group()` فقط. أكثر كسلاً من Meson (الذي يدير مصفوفته يدوياً) لكنه أبسط.

### `pcs_pinmux_ops`

```c
static const struct pinmux_ops pcs_pinmux_ops = {
    .get_functions_count = pinmux_generic_get_function_count, // ← generic
    .get_function_name   = pinmux_generic_get_function_name,  // ← generic
    .get_function_groups = pinmux_generic_get_function_groups,// ← generic
    .set_mux             = pcs_set_mux,           // ← custom
    .gpio_request_enable = pcs_request_gpio,      // ← custom
};
```

### `pcs_set_mux()` — تفعيل الـ Mux

```c
static int pcs_set_mux(struct pinctrl_dev *pctldev, unsigned fselector,
                       unsigned group)
{
    function = pinmux_generic_get_function(pctldev, fselector);
    func = function->data;  // ← هذا هو pcs_function المخصص

    for (i = 0; i < func->nvals; i++) {
        vals = &func->vals[i];

        raw_spin_lock_irqsave(&pcs->lock, flags);  // ← spinlock لأن قد تُستدعى من IRQ context

        val = pcs->read(vals->reg);           // اقرأ القيمة الحالية

        if (pcs->bits_per_mux)
            mask = vals->mask;      // mask مخصص لكل entry
        else
            mask = pcs->fmask;      // mask موحّد للـ function field

        val &= ~mask;               // امسح الـ bits
        val |= (vals->val & mask);  // ضع القيمة الجديدة
        pcs->write(val, vals->reg); // اكتب

        raw_spin_unlock_irqrestore(&pcs->lock, flags);
    }
}
```

لماذا `raw_spin_lock_irqsave` وليس `spin_lock`؟ لأن بعض الـ platforms تستدعي الـ remux من داخل interrupt context (مثلاً OMAP يعيد الـ mux قبل وبعد الـ idle).

### `pcs_request_gpio()` — تفعيل GPIO Mode

```c
static int pcs_request_gpio(struct pinctrl_dev *pctldev,
                             struct pinctrl_gpio_range *range, unsigned pin)
{
    // ابحث عن الـ range المناسب في قائمة gpiofuncs
    list_for_each_safe(pos, tmp, &pcs->gpiofuncs) {
        frange = list_entry(pos, struct pcs_gpiofunc_range, node);

        if (pin < frange->offset || pin >= frange->offset + frange->npins)
            continue;  // هذا الـ range لا يشمل هذا الـ pin

        offset = pcs_pin_reg_offset_get(pcs, pin);

        if (pcs->bits_per_mux) {
            // تعديل bits محددة فقط
            int pin_shift = pcs_pin_shift_reg_get(pcs, pin);
            data = pcs->read(pcs->base + offset);
            data &= ~(pcs->fmask << pin_shift);
            data |= frange->gpiofunc << pin_shift;
            pcs->write(data, pcs->base + offset);
        } else {
            // تعديل الـ function field كاملاً
            data = pcs->read(pcs->base + offset);
            data &= ~pcs->fmask;
            data |= frange->gpiofunc;
            pcs->write(data, pcs->base + offset);
        }
        break;
    }
}
```

---

## المجموعة الخامسة: الـ pinconf Operations

### فلسفة الـ Bias في `pinctrl-single`

الـ Meson يعرف مكان الـ pull bits. الـ `pinctrl-single` لا يعرف — المستخدم يُخبره عبر DT بـ patterns:

```dts
pinctrl-single,bias-pullup = <0x10 0x10 0x00 0x10>;
//                            val  enable disable mask
```

### `pcs_pinconf_clear_bias()`

```c
static void pcs_pinconf_clear_bias(struct pinctrl_dev *pctldev, unsigned pin)
{
    unsigned long config;
    for (i = 0; i < ARRAY_SIZE(pcs_bias); i++) {
        config = pinconf_to_config_packed(pcs_bias[i], 0);
        pcs_pinconf_set(pctldev, pin, &config, 1);
    }
}
```

`pcs_bias[]` = `{PIN_CONFIG_BIAS_PULL_DOWN, PIN_CONFIG_BIAS_PULL_UP}`. يُعطّل كلاهما بكتابة قيمة 0 (arg=0 يعني disable في هذا السياق).

### `pcs_pinconf_bias_disable()`

```c
static bool pcs_pinconf_bias_disable(struct pinctrl_dev *pctldev, unsigned pin)
{
    for (i = 0; i < ARRAY_SIZE(pcs_bias); i++) {
        config = pinconf_to_config_packed(pcs_bias[i], 0);
        if (!pcs_pinconf_get(pctldev, pin, &config))
            goto out;  // وجد أحد الـ bias مُفعَّلاً → ليس disabled
    }
    return true;  // لا pull-up ولا pull-down → bias مُعطَّل
out:
    return false;
}
```

### `pcs_pinconf_get()` — قراءة الـ Config

```c
static int pcs_pinconf_get(struct pinctrl_dev *pctldev,
                           unsigned pin, unsigned long *config)
{
    // 1. احصل على الـ function المرتبطة بهذا الـ pin
    pcs_get_function(pctldev, pin, &func);

    // 2. ابحث عن الـ conf entry المناسبة
    for (i = 0; i < func->nconfs; i++) {
        param = pinconf_to_config_param(*config);

        if (param == PIN_CONFIG_BIAS_DISABLE) {
            // خاص: تحقق من أن كل bias مُعطَّل
            if (pcs_pinconf_bias_disable(pctldev, pin)) {
                *config = 0;
                return 0;
            }
            return -ENOTSUPP;
        }

        if (param != func->conf[i].param) continue;

        // اقرأ الـ register
        offset = pin * (pcs->width / BITS_PER_BYTE);
        data = pcs->read(pcs->base + offset) & func->conf[i].mask;

        switch (func->conf[i].param) {
        // 4-parameter configs: enable/disable patterns
        case PIN_CONFIG_BIAS_PULL_DOWN:
        case PIN_CONFIG_BIAS_PULL_UP:
        case PIN_CONFIG_INPUT_SCHMITT_ENABLE:
            if ((data != func->conf[i].enable) ||
                (data == func->conf[i].disable))
                return -ENOTSUPP; // ليس في الوضع المطلوب
            *config = 0;
            break;

        // 2-parameter configs: قيمة عددية
        case PIN_CONFIG_DRIVE_STRENGTH:
        case PIN_CONFIG_SLEW_RATE:
        default:
            *config = data;
            break;
        }
    }
}
```

### `pcs_pinconf_set()` — كتابة الـ Config

```c
static int pcs_pinconf_set(struct pinctrl_dev *pctldev, unsigned pin,
                           unsigned long *configs, unsigned num_configs)
{
    for (j = 0; j < num_configs; j++) {
        param = pinconf_to_config_param(configs[j]);

        if (param == PIN_CONFIG_BIAS_DISABLE) {
            pcs_pinconf_clear_bias(pctldev, pin);
            continue;
        }

        // ابحث عن الـ conf entry المناسبة في func->conf
        for (i = 0; i < func->nconfs; i++) {
            if (param != func->conf[i].param) continue;

            offset = pin * (pcs->width / BITS_PER_BYTE);
            data = pcs->read(pcs->base + offset);
            arg = pinconf_to_config_argument(configs[j]);

            switch (param) {
            // 2-parameter: ضع القيمة العددية في الـ bits المحددة
            case PIN_CONFIG_DRIVE_STRENGTH:
            case PIN_CONFIG_SLEW_RATE:
            case PIN_CONFIG_INPUT_ENABLE:
                shift = ffs(func->conf[i].mask) - 1;  // أول bit في الـ mask
                data &= ~func->conf[i].mask;
                data |= (arg << shift) & func->conf[i].mask;
                break;

            // 4-parameter: enable أو disable pattern
            case PIN_CONFIG_BIAS_PULL_UP:
            case PIN_CONFIG_BIAS_PULL_DOWN:
                if (arg) {
                    pcs_pinconf_clear_bias(pctldev, pin); // أولاً أزل الـ bias الآخر
                    data = pcs->read(pcs->base + offset); // أعد القراءة
                }
                fallthrough;
            case PIN_CONFIG_INPUT_SCHMITT_ENABLE:
                data &= ~func->conf[i].mask;
                if (arg)
                    data |= func->conf[i].enable;   // فعّل
                else
                    data |= func->conf[i].disable;  // عطّل
                break;
            }
            pcs->write(data, pcs->base + offset);
        }
    }
}
```

لاحظ `fallthrough` — تفعيل PULL_UP يمر عبر نفس كود INPUT_SCHMITT_ENABLE لأن منطق الـ enable/disable bits هو نفسه.

### `pcs_pinconf_group_get()` — config لـ Group كامل

```c
static int pcs_pinconf_group_get(struct pinctrl_dev *pctldev,
                                  unsigned group, unsigned long *config)
{
    pinctrl_generic_get_group_pins(pctldev, group, &pins, &npins);

    for (i = 0; i < npins; i++) {
        pcs_pinconf_get(pctldev, pins[i], config);
        // تحقق أن كل pins الـ group لها نفس الـ config!
        if (i && (old != *config))
            return -ENOTSUPP;  // configs مختلفة → لا يمكن تمثيل الـ group
        old = *config;
    }
}
```

هذا شرط صارم: لو pins الـ group لها configs مختلفة، الـ group_get يفشل. هذا منطقي — الـ group config تعني config موحّدة لكل الـ group.

---

## المجموعة السادسة: الـ DT Parsing — `pcs_dt_node_to_map()`

هذا أعقد جزء في الـ driver. يُحوّل الـ DT nodes إلى `pinctrl_map` entries.

### `pcs_add_conf2()` و `pcs_add_conf4()`

```c
// للـ properties ذات قيمتين: [val, mask]
static void pcs_add_conf2(... const char *name, ...)
{
    of_property_read_u32_array(np, name, value, 2);
    // value[0] = قيمة الـ bits في الـ register
    // value[1] = mask للـ bits ذات الصلة

    value[0] &= value[1];  // تأكد أن القيمة داخل الـ mask
    shift = ffs(value[1]) - 1;

    add_config(conf, param, value[0], 0, 0, value[1]);
    add_setting(settings, param, value[0] >> shift);  // normalize للـ arg
}

// للـ properties ذات أربع قيم: [val, enable, disable, mask]
static void pcs_add_conf4(... const char *name, ...)
{
    of_property_read_u32_array(np, name, value, 4);
    // value[0] = القيمة الحالية
    // value[1] = pattern يعني "enabled"
    // value[2] = pattern يعني "disabled"
    // value[3] = mask

    // تحقق أن القيمة الحالية تطابق enable أو disable
    ret = pcs_config_match(value[0], value[1], value[2]);
    // ret=1: مُفعَّل الآن، ret=0: مُعطَّل الآن

    add_config(conf, param, value[0], value[1], value[2], value[3]);
    add_setting(settings, param, ret);  // 1=enabled, 0=disabled
}
```

**مثال DT:**

```dts
pinctrl-single,bias-pullup = <0x10 0x10 0x00 0x10>;
//  val=0x10 (currently enabled)
//  enable=0x10 (bit 4 set = pull-up enabled)
//  disable=0x00 (bit 4 clear = pull-up disabled)
//  mask=0x10 (only bit 4 matters)
```

### `pcs_parse_pinconf()` — DT → conf entries

```c
static const struct pcs_conf_type prop2[] = {
    { "pinctrl-single,drive-strength",  PIN_CONFIG_DRIVE_STRENGTH },
    { "pinctrl-single,slew-rate",       PIN_CONFIG_SLEW_RATE },
    { "pinctrl-single,input-enable",    PIN_CONFIG_INPUT_ENABLE },
    { "pinctrl-single,input-schmitt",   PIN_CONFIG_INPUT_SCHMITT },
    { "pinctrl-single,low-power-mode",  PIN_CONFIG_MODE_LOW_POWER },
};
static const struct pcs_conf_type prop4[] = {
    { "pinctrl-single,bias-pullup",           PIN_CONFIG_BIAS_PULL_UP },
    { "pinctrl-single,bias-pulldown",         PIN_CONFIG_BIAS_PULL_DOWN },
    { "pinctrl-single,input-schmitt-enable",  PIN_CONFIG_INPUT_SCHMITT_ENABLE },
};
```

الـ driver يقرأ هذه الـ properties من كل DT node ويبني `func->conf[]` منها.

### `pcs_parse_one_pinctrl_entry()` — المُحلّل الرئيسي

```c
static int pcs_parse_one_pinctrl_entry(pcs, np, map, num_maps, pgnames)
{
    // 1. اقرأ property "pinctrl-single,pins"
    rows = pinctrl_count_index_with_args(np, "pinctrl-single,pins");

    for (i = 0; i < rows; i++) {
        pinctrl_parse_index_with_args(np, name, i, &pinctrl_spec);

        // كل entry: offset + value (أو offset + value + mask لـ 3 args)
        offset = pinctrl_spec.args[0];
        vals[found].reg = pcs->base + offset;
        vals[found].val = pinctrl_spec.args[1];

        // حوّل الـ offset إلى pin number
        pin = pcs_get_pin_by_offset(pcs, offset);
        pins[found++] = pin;
    }

    // 2. أضف الـ function والـ group
    fsel = pcs_add_function(pcs, &function, np->name, vals, found, ...);
    gsel = pinctrl_generic_add_group(pcs->pctl, np->name, pins, found, pcs);

    // 3. أنشئ MUX_GROUP map entry
    (*map)->type = PIN_MAP_TYPE_MUX_GROUP;
    (*map)->data.mux.group = np->name;
    (*map)->data.mux.function = np->name;  // ← نفس الاسم! (DT node name)

    // 4. اختياري: parse الـ pinconf
    if (PCS_HAS_PINCONF)
        pcs_parse_pinconf(pcs, np, function, map);
}
```

لاحظ: function name = group name = اسم الـ DT node. هذا قرار تصميمي مقصود — الـ driver يتجنب فهم أسماء الـ pins والـ functions لتقليل الـ bloat على الـ systems الكبيرة.

### `pcs_parse_bits_in_pinctrl_entry()` — لوضع `bits_per_mux`

```c
// property: "pinctrl-single,bits"
// كل entry: offset + val + mask
// مثال: <0x100 0x200 0xf00>
//  register عند offset 0x100
//  اكتب 0x200 بـ mask 0xf00

while (mask) {
    bit_pos = __ffs(mask);  // أول bit مُعيَّن في الـ mask
    pin_num_from_lsb = bit_pos / pcs->bits_per_pin;

    vals[found].mask = submask;
    vals[found].reg  = pcs->base + offset;
    vals[found].val  = val_pos;

    pin = pcs_get_pin_by_offset(pcs, offset);
    pins[found++] = pin + pin_num_from_lsb;

    mask &= ~mask_pos;  // أزل هذا الـ pin من الـ mask
}
```

يُفسّر الـ mask bit-by-bit لاستخراج كل pin مشمول في هذا الـ entry.

---

## المجموعة السابعة: الـ Pin Table Allocation

### `pcs_allocate_pin_table()`

```c
static int pcs_allocate_pin_table(struct pcs_device *pcs)
{
    mux_bytes = pcs->width / BITS_PER_BYTE;

    if (pcs->bits_per_mux && pcs->fmask) {
        pcs->bits_per_pin = fls(pcs->fmask);  // عدد bits للـ function field
        nr_pins = (pcs->size * BITS_PER_BYTE) / pcs->bits_per_pin;
    } else {
        nr_pins = pcs->size / mux_bytes;  // register واحد = pin واحد
    }

    pcs->pins.pa = devm_kcalloc(dev, nr_pins, sizeof(pinctrl_pin_desc));

    for (i = 0; i < nr_pins; i++) {
        offset = pcs_pin_reg_offset_get(pcs, i);

        // تحقق: لو الـ IRQ مُفعَّل في الـ register أثناء الـ boot → أوقفه
        if (pcs_soc->irq_enable_mask) {
            val = pcs->read(pcs->base + offset);
            if (val & pcs_soc->irq_enable_mask) {
                val &= ~pcs_soc->irq_enable_mask;
                pcs->write(val, pcs->base + offset);
            }
        }

        pin->number = i;
        pcs->pins.cur++;
    }
}
```

الـ driver **يحسب عدد الـ pins تلقائياً** من حجم الـ memory region (`pcs->size`). لا يحتاج قائمة ثابتة مثل Meson. هذا سبب تسميته "generic".

---

## المجموعة الثامنة: الـ IRQ Support

هذا جزء غير موجود في Meson — الـ pin controller يدعم **wakeup interrupts**.

### التصميم

كل pin يمكنه أن يكون wakeup source. عند حدث كهربائي على الـ pin أثناء النوم، يُثار interrupt يُوقظ النظام.

```
Physical Pin
    │ level change during sleep
    ▼
Pin Controller Register
    │ bit WAKEUP_EVENT يُضبط تلقائياً
    ▼
IRQ line إلى CPU
    │
    ▼
pcs_irq_handle() يقرأ كل registers
    │
    ▼
generic_handle_domain_irq() يُثير الـ virtual IRQ المقابل
    │
    ▼
الـ device driver الذي يملك هذا الـ pin يستيقظ
```

### نوعا الـ IRQ Handler

```c
// 1. Chained (مُخصَّص): interrupt منفصل لكل controller
static void pcs_irq_chain_handler(struct irq_desc *desc)
{
    chained_irq_enter(chip, desc);
    pcs_irq_handle(pcs_soc);
    chained_irq_exit(chip, desc);
}

// 2. Shared: عدة controllers تشترك في نفس الـ interrupt (OMAP quirk)
static irqreturn_t pcs_irq_handler(int irq, void *d)
{
    return pcs_irq_handle(pcs_soc) ? IRQ_HANDLED : IRQ_NONE;
}
```

### `pcs_irq_set()` — تفعيل/تعطيل الـ Interrupt

```c
static inline void pcs_irq_set(struct pcs_soc_data *pcs_soc,
                                int irq, const bool enable)
{
    list_for_each(pos, &pcs->irqs) {
        pcswi = list_entry(pos, struct pcs_interrupt, node);
        if (irq != pcswi->irq) continue;

        raw_spin_lock(&pcs->lock);
        mask = pcs->read(pcswi->reg);
        if (enable)
            mask |= soc_mask;   // فعّل الـ WAKEUP_EN bit
        else
            mask &= ~soc_mask;  // عطّل الـ WAKEUP_EN bit
        pcs->write(mask, pcswi->reg);
        mask = pcs->read(pcswi->reg);  // ← flush posted write!
        raw_spin_unlock(&pcs->lock);
    }

    if (pcs_soc->rearm)
        pcs_soc->rearm();  // بعض الـ SoCs تحتاج خطوة rearm
}
```

`flush posted write` — قراءة بعد الكتابة تضمن أن الـ write وصل للـ hardware قبل العودة (مهم للـ memory-mapped registers).

---

## المجموعة التاسعة: Context Save/Restore

### المشكلة

بعض الـ SoCs (كـ AM437x بـ `PCS_CONTEXT_LOSS_OFF`) **تفقد محتوى registers** عند دخول أعمق حالات النوم لأن الـ power domain يُقطع.

### الحل

```c
static int pcs_save_context(struct pcs_device *pcs)
{
    // خصّص buffer لو لم يكن موجوداً (lazy allocation)
    if (!pcs->saved_vals)
        pcs->saved_vals = devm_kzalloc(dev, pcs->size, GFP_ATOMIC);

    // اقرأ كل الـ registers وخزّنها
    switch (pcs->width) {
    case 64:
        regsl = pcs->saved_vals;
        for (i = 0; i < pcs->size; i += mux_bytes)
            *regsl++ = pcs->read(pcs->base + i);
        break;
    case 32: /* نفس المنطق بـ u32 */ break;
    case 16: /* نفس المنطق بـ u16 */ break;
    }
}

static void pcs_restore_context(struct pcs_device *pcs)
{
    // أعد كتابة كل الـ registers من الـ buffer
    switch (pcs->width) {
    case 32:
        regsw = pcs->saved_vals;
        for (i = 0; i < pcs->size; i += mux_bytes)
            pcs->write(*regsw++, pcs->base + i);
        break;
    // ...
    }
}

static int pinctrl_single_suspend_noirq(struct device *dev)
{
    if (pcs->flags & PCS_CONTEXT_LOSS_OFF)
        pcs_save_context(pcs);  // احفظ قبل النوم

    return pinctrl_force_sleep(pcs->pctl);  // انتقل لـ sleep state
}

static int pinctrl_single_resume_noirq(struct device *dev)
{
    if (pcs->flags & PCS_CONTEXT_LOSS_OFF)
        pcs_restore_context(pcs);  // استعد بعد الاستيقاظ

    return pinctrl_force_default(pcs->pctl);  // انتقل لـ default state
}
```

---

## المجموعة العاشرة: `pcs_probe()` — التسلسل الكامل

```c
static int pcs_probe(struct platform_device *pdev)
{
    // 1. احصل على الـ SoC-specific data من of_match
    soc = of_device_get_match_data(&pdev->dev);

    // 2. اقرأ إعدادات الـ DT الأساسية
    of_property_read_u32(np, "pinctrl-single,register-width", &pcs->width);
    of_property_read_u32(np, "pinctrl-single,function-mask",  &pcs->fmask);
    of_property_read_u32(np, "pinctrl-single,function-off",   &pcs->foff);
    pcs->bits_per_mux = of_property_read_bool(np, "pinctrl-single,bit-per-mux");

    // 3. اختر الـ read/write functions المناسبة لعرض الـ register
    switch (pcs->width) {
    case 8:  pcs->read = pcs_readb; pcs->write = pcs_writeb; break;
    case 16: pcs->read = pcs_readw; pcs->write = pcs_writew; break;
    case 32: pcs->read = pcs_readl; pcs->write = pcs_writel; break;
    }

    // 4. ioremap الـ registers
    pcs->base = devm_ioremap(dev, res->start, pcs->size);

    // 5. احسب عدد الـ pins تلقائياً وأنشئ الـ table
    pcs_allocate_pin_table(pcs);

    // 6. سجّل الـ pin controller
    devm_pinctrl_register_and_init(dev, &pcs->desc, pcs, &pcs->pctl);

    // 7. أضف الـ GPIO function ranges
    pcs_add_gpio_func(np, pcs);

    // 8. إعداد الـ IRQ (اختياري)
    if (PCS_HAS_IRQ)
        pcs_irq_init_chained_handler(pcs, np);

    // 9. فعّل
    pinctrl_enable(pcs->pctl);
}
```

---

## مقارنة مع Meson Driver

```
الجانب              │ pinctrl-meson      │ pinctrl-single
────────────────────┼────────────────────┼───────────────────────
التخصص              │ Amlogic فقط        │ أي SoC one-reg-per-pin
Pin Table           │ static array       │ يُحسب من memory size
Function Names      │ موجودة ومعرّفة    │ = DT node name (لا معنى لها)
Register Access     │ regmap             │ function pointers مباشرة
GPIO Integration    │ كاملة مع gpio_chip │ عبر gpio_request_enable فقط
IRQ Support         │ لا                 │ نعم (wakeup)
Context Save        │ لا                 │ نعم (PCS_CONTEXT_LOSS_OFF)
Config Encoding     │ يعرف bit positions │ المستخدم يُخبره عبر DT
SoC Variants        │ parse_dt callback  │ pcs_soc_data flags
```

الـ `pinctrl-single` هو **أقصى درجات الـ generic** — حتى مواقع الـ bits لا يعرفها، المستخدم يُعرّف كل شيء في الـ DT. المقابل هو DT أطول وأعقد، لكن الـ driver نفسه لا يتغير من SoC لآخر.