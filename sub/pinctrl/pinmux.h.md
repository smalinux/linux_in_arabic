# شرح `pinmux.h` — `struct pinmux_ops` بالتفصيل

هذا الملف يُعرّف **vtable واحدة فقط**: `struct pinmux_ops`، وهي العقد بين الـ pinctrl core وأي driver يريد دعم الـ **pinmuxing**.

---

## أولاً: الـ Callbacks المتعلقة بالـ Pin Ownership

### `request(pctldev, offset)`

يُستدعى من الـ core **قبل** أي عملية mux، ليسأل الـ driver: "هل يمكنك إتاحة الـ pin رقم `offset` للاستخدام؟"

الـ driver هنا يملك حق الرفض بإرجاع error code سالب. مثال للاستخدام: بعض الـ pins محجوزة للـ power management ولا يجب أن يمسّها أحد.

```c
static int foo_request(struct pinctrl_dev *pctldev, unsigned int offset)
{
    if (offset == RESERVED_POWER_PIN)
        return -EBUSY;  // ارفض
    return 0;           // وافق
}
```

### `free(pctldev, offset)`

عكس `request()` تماماً. يُستدعى عندما يتخلى device عن الـ pin، فيتيح الـ driver للـ core إعادته للـ pool المتاح.

---

## ثانياً: الـ Callbacks المتعلقة بالـ Functions

### `get_functions_count(pctldev)`

بسيطة: ترجع عدد الـ functions المتاحة في هذا الـ driver. الـ core يستخدمها لتحديد نطاق الـ selector الصحيح (0 حتى count-1).

### `get_function_name(pctldev, selector)`

تُرجع اسم الـ function ذات الـ `selector` المعطى. الـ core يستخدمها لمطابقة ما هو موجود في الـ mapping table بما هو مسجّل في الـ driver.

مثال: لو الـ mapping table تقول `function = "spi0"` والـ driver عنده function باسم `"spi0"` عند selector=0، فالـ core يجد التطابق ويستخدم selector=0 في كل العمليات اللاحقة.

### `get_function_groups(pctldev, selector, **groups, *num_groups)`

تُرجع مصفوفة أسماء الـ groups التي يمكن استخدامها مع الـ function المحددة. الـ core يحتاج هذه المعلومات ليعرف:

- ما هي الخيارات المتاحة لتفعيل هذه الـ function؟
- أي group يجب أن يختار لو لم يحدد الـ user group معين؟

```c
static int foo_get_groups(struct pinctrl_dev *pctldev,
                          unsigned int selector,
                          const char *const **groups,
                          unsigned int *const num_groups)
{
    *groups     = foo_functions[selector].groups;
    *num_groups = foo_functions[selector].ngroups;
    return 0;
}
```

---

## ثالثاً: `function_is_gpio(pctldev, selector)`

هذا الـ callback مهم لفهم العلاقة بين الـ pinmux وsubsystem الـ GPIO.

المشكلة: الـ core أحياناً يحتاج أن يعرف "هل هذه الـ function هي GPIO function؟" ليقرر هل يسمح بتفعيلها مع أو بدون تعارض مع طلبات الـ GPIO.

متى يكون مفيداً؟ فقط على الـ controllers التي تستخدم **function واحدة مخصصة للـ GPIO**، وعادةً هذا يعني أن كل pin له group خاص به (one-group-per-pin approach). في هذه الحالة يمكن تفعيل GPIO على pin منفرد دون التأثير على غيره.

```c
static bool foo_function_is_gpio(struct pinctrl_dev *pctldev,
                                 unsigned int selector)
{
    return foo_functions[selector].flags & PINFUNCTION_FLAG_GPIO;
}
```

---

## رابعاً: `set_mux(pctldev, func_selector, group_selector)`

هذا هو **قلب الـ pinmux**. يُستدعى من الـ core لتفعيل function معينة على group معين فعلياً في الـ hardware.

نقطة مهمة جداً مذكورة في التعليق: **الـ driver لا يحتاج أن يتحقق من التعارضات**. الـ core تولّى هذا قبل استدعاء `set_mux()`. بمجرد وصول الـ selector إلى الـ driver، الـ core ضمن بالفعل أن هذه الـ pins غير مستخدمة من أي device أو GPIO أخرى.

```c
static int foo_set_mux(struct pinctrl_dev *pctldev,
                       unsigned int func_selector,
                       unsigned int group_selector)
{
    // الـ core ضمن لك عدم التعارض، فقط اكتب في الـ register
    u8 regbit = BIT(group_selector);
    writeb((readb(MUX) | regbit), MUX);
    return 0;
}
```

---

## خامساً: الـ Callbacks المتعلقة بالـ GPIO

### `gpio_request_enable(pctldev, range, offset)`

يُستدعى من الـ GPIO subsystem (عبر `pinctrl_gpio_request()`) عندما يطلب أحد تفعيل GPIO على pin معين.

الـ `offset` هو رقم الـ pin داخل الـ `gpio_range` المعطى، لا الرقم العالمي للـ GPIO.

تُنفَّذ هذه الدالة **فقط** إذا كان الـ controller قادراً على تحويل كل pin منفرداً إلى GPIO mode بشكل مستقل. لو الـ controller يتعامل مع الـ GPIO كـ function عادية، فلا داعي لتنفيذها.

### `gpio_disable_free(pctldev, range, offset)`

عكس `gpio_request_enable()` — يُفرج عن الـ GPIO muxing ويُعيد الـ pin لوضعه الافتراضي أو المتاح.

### `gpio_set_direction(pctldev, range, offset, input)`

بعض الـ controllers تحتاج إعداد مختلف في الـ pinmux registers بحسب اتجاه الـ GPIO (input أو output). مثلاً: بعض الـ SoCs عندها register للـ mux تحتوي على bit للاتجاه أيضاً.

الـ `input = true` تعني GPIO input، و`input = false` تعني GPIO output.

هذه الدالة تُستدعى من داخل `gpio_chip->direction_input()` و`gpio_chip->direction_output()` في الـ GPIO driver، وليس مباشرة من الـ device driver.

---

## سادساً: `strict` — الـ Flag المهم

هذا ليس callback بل **boolean flag** مباشر في الـ struct.

عندما يكون `strict = true`، يتحقق الـ core بصرامة قبل الموافقة على أي طلب pin: إذا كان الـ pin مستخدماً من GPIO (`gpio_owner`)، يرفض طلبات الـ pinmux عليه، والعكس صحيح.

متى تضعه؟ على الـ hardware من نوع تصميم (A) الذي شرحناه سابقاً — حيث GPIO و pinmux registers منفصلة ويمكنهما نظرياً التعارض.

```c
static struct pinmux_ops foo_pmxops = {
    .get_functions_count  = foo_get_functions_count,
    .get_function_name    = foo_get_fname,
    .get_function_groups  = foo_get_groups,
    .set_mux              = foo_set_mux,
    .gpio_request_enable  = foo_gpio_request_enable,
    .gpio_disable_free    = foo_gpio_disable_free,
    .gpio_set_direction   = foo_gpio_set_direction,
    .strict               = true,   // ← لا تسمح بـ GPIO و mux معاً
};
```

---

## الفرق بين طريقتين لدعم GPIO

الـ driver أمامه خياران لدعم GPIO:

**الطريقة الأولى:** تعريف GPIO كـ function عادية باسم مثل `"gpio"` في مصفوفة الـ functions، ويتعامل معها `set_mux()` كأي function أخرى. الـ core يبحث عن function باسم `"gpioN"` إذا لم يجد GPIO handler مخصص.

**الطريقة الثانية:** تنفيذ `gpio_request_enable()` / `gpio_disable_free()` مباشرة. هذا يتيح تفعيل GPIO على pin منفرد دون المرور بآلية الـ functions والـ groups. أسرع وأبسط للـ controllers التي تستطيع فعل هذا.

الفرق العملي: الطريقة الثانية مستقلة تماماً عن الـ function/group selectors، لكن الـ core مع ذلك يتأكد من عدم التعارض مع أي mux setting موجود على هذا الـ pin.

---

## ملخص كل الـ Callbacks

|الـ Callback|متى يُستدعى|إلزامي؟|
|---|---|---|
|`request`|قبل أي mux operation|اختياري|
|`free`|بعد تحرير الـ pin|اختياري|
|`get_functions_count`|عند التهيئة والـ debugfs|إلزامي|
|`get_function_name`|لمطابقة أسماء الـ functions|إلزامي|
|`get_function_groups`|لمعرفة groups الـ function|إلزامي|
|`function_is_gpio`|لو تحتاج تمييز GPIO function|اختياري|
|`set_mux`|عند تفعيل function على group|إلزامي|
|`gpio_request_enable`|عند طلب GPIO على pin|اختياري*|
|`gpio_disable_free`|عند تحرير GPIO على pin|اختياري*|
|`gpio_set_direction`|عند تغيير اتجاه الـ GPIO|اختياري|
|`strict` (flag)|يُضبط مرة واحدة عند التعريف|حسب الـ HW|

* إلزامي لو الـ controller يدعم per-pin GPIO muxing بدون function/group mechanism.

# شرح `pinmux.h` (Internal) — الـ Core Internal Interface

هذا الملف **داخلي** بحت — ليس للـ drivers بل للتواصل بين `core.c` و`pinmux.c` داخل الـ pinctrl subsystem نفسه. لاحظ أنه ليس في `include/linux/pinctrl/` بل في `drivers/pinctrl/`.

---

## أولاً: هيكل الملف العام

الملف مقسّم إلى ثلاثة أقسام بـ `#ifdef`:

```
#ifdef CONFIG_PINMUX
    ← الـ API الأساسية بين core و pinmux
#ifdef CONFIG_PINMUX && CONFIG_DEBUG_FS
    ← دوال الـ debugfs
#ifdef CONFIG_GENERIC_PINMUX_FUNCTIONS
    ← الـ generic function management
```

لو `CONFIG_PINMUX` مُعطَّل، كل الدوال تصبح **stubs** تُرجع 0 أو لا تفعل شيئاً — بدون أي `#ifdef` في كود الـ core نفسه.

---

## ثانياً: المجموعة الأولى — Validation و Checks

### `pinmux_check_ops(pctldev)`

يُستدعى من الـ core عند تسجيل الـ pin controller للتحقق أن الـ `pinmux_ops` المُسجَّلة كاملة وصحيحة. مثلاً: لو سجّلت `set_mux` بدون `get_functions_count` سيرفض. هذا الفحص يحدث مرة واحدة فقط عند الـ registration وليس في كل عملية.

### `pinmux_validate_map(map, i)`

يُستدعى لكل entry في الـ mapping table عند تسجيلها بـ `pinctrl_register_mappings()`. يتحقق أن الـ `PIN_MAP_TYPE_MUX_GROUP` entry تحتوي على function name على الأقل، وأن `ctrl_dev_name` موجود.

### `pinmux_can_be_used_for_gpio(pctldev, pin)`

سؤال مهم يطرحه الـ core قبل السماح لـ GPIO subsystem باستخدام pin: هل هذا الـ pin يمكن استخدامه كـ GPIO؟ يعتمد على الـ `strict` flag وحالة الـ pin الحالية.

```
لو strict=true:
  pin مستخدم في mux → returns false (لا يمكن)
  pin حر            → returns true

لو strict=false:
  دائماً returns true (التعارض مسموح)
```

---

## ثالثاً: المجموعة الثانية — GPIO Integration

هذه الدوال هي **الجسر** بين GPIO subsystem وبين الـ pinmux driver. تُستدعى من `gpiolib` وليس من الـ device drivers مباشرة.

### `pinmux_request_gpio(pctldev, range, pin, gpio)`

```
GPIO driver يستدعي gpiod_get()
        │
        ▼
gpiolib يستدعي gpio_chip->request()
        │
        ▼
gpiochip_generic_request()
        │
        ▼
pinmux_request_gpio()  ← هنا
        │
        ├─ يتحقق من strict flag وpin ownership
        ├─ يستدعي pinmux_ops->request() إذا موجودة
        └─ يستدعي pinmux_ops->gpio_request_enable() إذا موجودة
```

الـ `pin` هو رقم الـ pin في الـ controller، والـ `gpio` هو الرقم العالمي في Linux GPIO namespace.

### `pinmux_free_gpio(pctldev, pin, range)`

عكس `pinmux_request_gpio()` — يُفرج عن الـ pin ويستدعي `gpio_disable_free()` إذا كانت موجودة.

### `pinmux_gpio_direction(pctldev, range, pin, input)`

يُستدعى عند استدعاء `gpio_direction_input()` أو `gpio_direction_output()`. يمرر الطلب إلى `pinmux_ops->gpio_set_direction()` إذا كانت موجودة في الـ driver.

---

## رابعاً: المجموعة الثالثة — Setting Management

هذه المجموعة هي **قلب التفعيل الفعلي** — تتعامل مع `struct pinctrl_setting` التي تمثل state مُفعَّل حالياً.

### `pinmux_map_to_setting(map, setting)`

يُحوّل `pinctrl_map` entry (الوصف الثابت من الـ mapping table) إلى `pinctrl_setting` (كائن حي يمكن تفعيله وتعطيله).

```
pinctrl_map (ثابت، من الـ mapping table):
{
    dev_name = "foo-spi.0"
    type = MUX_GROUP
    function = "spi0"
    group = "spi0_grp"
}
        │
        ▼ pinmux_map_to_setting()
        │ يبحث عن function selector
        │ يبحث عن group selector
        ▼
pinctrl_setting (حي، يُخزَّن في pinctrl_state):
{
    type = PIN_MAP_TYPE_MUX_GROUP
    pctldev = pointer to controller
    data.mux.func  = 2  (selector رقمي)
    data.mux.group = 0  (selector رقمي)
}
```

تحويل الأسماء إلى selectors رقمية يحدث هنا مرة واحدة فقط — في `pinctrl_lookup_state()` وليس في كل `pinctrl_select_state()`.

### `pinmux_free_setting(setting)`

يُحرر الموارد المخصصة بـ `pinmux_map_to_setting()`.

### `pinmux_enable_setting(setting)`

هذا ما يُستدعى فعلياً عند `pinctrl_select_state()`:

```
pinctrl_select_state()
        │
        ▼
لكل setting في الـ state:
        │
        ├─ PIN_MAP_TYPE_MUX_GROUP → pinmux_enable_setting()
        │       │
        │       ├─ يتحقق من تعارضات الـ pins
        │       ├─ يُسجّل ownership للـ pins
        │       └─ يستدعي pinmux_ops->set_mux(func_sel, group_sel)
        │
        └─ PIN_MAP_TYPE_CONFIGS_* → pinconf_apply_setting()
```

### `pinmux_disable_setting(setting)`

عكس `pinmux_enable_setting()` — يُحرر الـ pin ownership ويمكن أن يضع الـ pins في وضع آمن.

---

## خامساً: المجموعة الرابعة — Debugfs

```c
void pinmux_show_map(struct seq_file *s, const struct pinctrl_map *map);
void pinmux_show_setting(struct seq_file *s, const struct pinctrl_setting *setting);
void pinmux_init_device_debugfs(struct dentry *devroot, struct pinctrl_dev *pctldev);
```

### `pinmux_init_device_debugfs(devroot, pctldev)`

ينشئ ملفات الـ debugfs داخل مجلد الـ controller:

```
/sys/kernel/debug/pinctrl/pinctrl-foo/
    ├── pinmux-functions  ← pinmux_show_map() لكل function
    ├── pinmux-pins       ← من يملك كل pin حالياً
    └── pinmux-select     ← كتابة فيه تُفعّل function يدوياً
```

### `pinmux_show_map()` و `pinmux_show_setting()`

تطبع معلومات الـ mapping والـ settings بشكل بشري مقروء في الـ debugfs. الفرق:

- `show_map` تطبع الوصف الثابت (الاسم، النوع)
- `show_setting` تطبع الحالة الحية (الـ selectors، الـ pins الفعلية)

---

## سادساً: المجموعة الخامسة — Generic Function Management

هذه المجموعة موجودة بـ `#ifdef CONFIG_GENERIC_PINMUX_FUNCTIONS` وتوفر **جدول functions مُدار تلقائياً** بدلاً من أن يدير كل driver مصفوفته يدوياً.

### `struct function_desc`

```c
struct function_desc {
    const struct pinfunction *func;  // البيانات العامة (name + groups)
    void *data;                      // بيانات خاصة بالـ driver
};
```

هذا الـ wrapper يجمع الـ generic part والـ driver-specific part في كائن واحد. لاحظناه في `pinctrl-single` حيث `function->data` هو `struct pcs_function *`.

### دوال الإدارة

```c
// إضافة function بأسماء groups
int pinmux_generic_add_function(pctldev, name, groups, ngroups, data);

// إضافة function من struct pinfunction جاهزة
int pinmux_generic_add_pinfunction(pctldev, func, data);

// حذف function بالـ selector
int pinmux_generic_remove_function(pctldev, selector);

// حذف كل الـ functions (عند unregister)
void pinmux_generic_free_functions(pctldev);
```

### دوال القراءة

```c
int         pinmux_generic_get_function_count(pctldev);
const char *pinmux_generic_get_function_name(pctldev, selector);
int         pinmux_generic_get_function_groups(pctldev, selector, **groups, *ngroups);

// إرجاع الـ function_desc كاملة (تُستخدم في pinctrl-single)
const struct function_desc *pinmux_generic_get_function(pctldev, selector);
```

### `pinmux_generic_function_is_gpio(pctldev, selector)`

يتحقق من `PINFUNCTION_FLAG_GPIO` في الـ function المحددة — يُستخدم من `pinmux_ops->function_is_gpio` الذي شرحناه سابقاً.

---

## الصورة الكاملة — كيف يرتبط هذا الملف بالباقي

```
pinctrl_select_state()   [consumer API]
        │
        ▼
core.c يمرر لـ:
        │
        ├─► pinmux_enable_setting()   ← من هذا الملف
        │       │
        │       └─► pinmux_ops->set_mux()   ← في driver الخاص بك
        │
        └─► pinconf_apply_setting()   ← من pinconf.h الداخلي

gpiod_get()   [GPIO API]
        │
        ▼
gpiolib يمرر لـ:
        │
        └─► pinmux_request_gpio()   ← من هذا الملف
                │
                └─► pinmux_ops->gpio_request_enable()  ← في driver الخاص بك

pinctrl_register_and_init()   [registration]
        │
        ├─► pinmux_check_ops()   ← تحقق من صحة الـ ops
        └─► pinmux_validate_map()  ← تحقق من صحة الـ mappings
```

الخلاصة: هذا الملف هو **interface داخلي** يحمي الـ core من تفاصيل الـ pinmux، ويحمي الـ pinmux من تفاصيل الـ core — Separation of Concerns داخل الـ subsystem نفسه.