# شرح `machine.h` — الـ Mapping Table Interface

هذا الملف هو **الجسر** بين الـ board/machine configuration وبين الـ pinctrl subsystem. يُعرّف الـ mapping table التي تربط كل device بالـ pins والـ functions والـ configs الخاصة به.

---

## أولاً: `enum pinctrl_map_type` — أنواع الـ Mapping Entries

```
pinctrl_map_type
│
├── PIN_MAP_TYPE_INVALID       ← قيمة خاصة، تعني "اكتشف النوع تلقائياً"
│                                (تُستخدم داخلياً في generic parser)
│
├── PIN_MAP_TYPE_DUMMY_STATE   ← entry فارغ — يُعلن وجود state بدون أي تأثير
│                                مهم لـ platforms لا تحتاج pinctrl
│
├── PIN_MAP_TYPE_MUX_GROUP     ← تفعيل function على group معين
│                                (يُحدد أي وظيفة تخرج من أي pins)
│
├── PIN_MAP_TYPE_CONFIGS_PIN   ← ضبط خصائص كهربائية على pin منفرد
│                                (pull-up، drive-strength، إلخ)
│
└── PIN_MAP_TYPE_CONFIGS_GROUP ← نفس السابق لكن على group كامل دفعة واحدة
```

---

## ثانياً: الـ Structs الداخلية

### `struct pinctrl_map_mux`

تُستخدم حصراً مع `PIN_MAP_TYPE_MUX_GROUP`:

```c
struct pinctrl_map_mux {
    const char *group;     // أي group من groups الـ controller؟
                           // NULL = خذ أول group مناسب تلقائياً
    const char *function;  // أي function تريد تفعيلها؟
};
```

الـ `group` اختياري لأن بعض الـ functions ترتبط بـ group واحد فقط، فلا داعي لتحديده صراحةً.

### `struct pinctrl_map_configs`

تُستخدم مع `PIN_MAP_TYPE_CONFIGS_PIN` و`PIN_MAP_TYPE_CONFIGS_GROUP`:

```c
struct pinctrl_map_configs {
    const char    *group_or_pin; // اسم الـ pin أو الـ group المستهدف
    unsigned long *configs;      // مصفوفة من الـ packed config values
    unsigned int   num_configs;  // عددها
};
```

الـ `configs` هي نفس الـ `PIN_CONF_PACKED()` values التي شرحناها في `pinconf-generic.h`.

---

## ثالثاً: `struct pinctrl_map` — الـ Entry الكاملة

هذه هي الـ struct المحورية في الملف كله:

```c
struct pinctrl_map {
    const char            *dev_name;      // اسم الـ device المالك
    const char            *name;          // اسم الـ state
    enum pinctrl_map_type  type;          // نوع الـ entry
    const char            *ctrl_dev_name; // أي pin controller يُنفّذ هذا
    union {
        struct pinctrl_map_mux     mux;     // بيانات الـ MUX_GROUP
        struct pinctrl_map_configs configs; // بيانات الـ CONFIGS_*
    } data;
};
```

```
struct pinctrl_map
┌───────────────────────────────────────────────────────┐
│ dev_name:      "foo-uart.0"                           │  ← من يطلب؟
│ name:          "default"                              │  ← أي state؟
│ type:          PIN_MAP_TYPE_MUX_GROUP                 │  ← ماذا يفعل؟
│ ctrl_dev_name: "pinctrl-foo"                          │  ← من يُنفّذ؟
│ data.mux:                                             │
│     .group    = "uart0_grp"                           │  ← أي group؟
│     .function = "uart0"                               │  ← أي function؟
└───────────────────────────────────────────────────────┘
```

### قاعدة الـ Hogging

لو `dev_name == ctrl_dev_name`، الـ core يُفعّل هذا الـ entry فوراً عند تسجيل الـ controller نفسه، دون انتظار أي device آخر.

---

## رابعاً: الـ Macros — شرح كامل لكل واحد

الملف يُوفّر 13 macro لتسهيل بناء الـ mapping table. يمكن فهمها بنظام من ثلاثة أبعاد:

```
البُعد الأول: النوع
  MUX_GROUP    → لتفعيل functions
  CONFIGS_PIN  → لضبط config على pin
  CONFIGS_GROUP→ لضبط config على group
  DUMMY_STATE  → entry وهمي

البُعد الثاني: الـ State
  (بدون لاحقة)   → تحدد الـ state يدوياً
  _DEFAULT        → يستخدم "default" state تلقائياً

البُعد الثالث: الـ Hogging
  (بدون HOG)     → entry عادي للـ device
  _HOG            → dev_name = ctrl_dev_name (يُفعَّل فوراً)
  _HOG_DEFAULT    → HOG + DEFAULT معاً
```

### جدول كل الـ Macros

|الـ Macro|النوع|State|Hog؟|
|---|---|---|---|
|`PIN_MAP_DUMMY_STATE`|DUMMY|يدوي|لا|
|`PIN_MAP_MUX_GROUP`|MUX|يدوي|لا|
|`PIN_MAP_MUX_GROUP_DEFAULT`|MUX|default|لا|
|`PIN_MAP_MUX_GROUP_HOG`|MUX|يدوي|نعم|
|`PIN_MAP_MUX_GROUP_HOG_DEFAULT`|MUX|default|نعم|
|`PIN_MAP_CONFIGS_PIN`|CONFIGS_PIN|يدوي|لا|
|`PIN_MAP_CONFIGS_PIN_DEFAULT`|CONFIGS_PIN|default|لا|
|`PIN_MAP_CONFIGS_PIN_HOG`|CONFIGS_PIN|يدوي|نعم|
|`PIN_MAP_CONFIGS_PIN_HOG_DEFAULT`|CONFIGS_PIN|default|نعم|
|`PIN_MAP_CONFIGS_GROUP`|CONFIGS_GROUP|يدوي|لا|
|`PIN_MAP_CONFIGS_GROUP_DEFAULT`|CONFIGS_GROUP|default|لا|
|`PIN_MAP_CONFIGS_GROUP_HOG`|CONFIGS_GROUP|يدوي|نعم|
|`PIN_MAP_CONFIGS_GROUP_HOG_DEFAULT`|CONFIGS_GROUP|default|نعم|

---

## خامساً: مثال mapping table كامل

```c
static unsigned long uart_default_configs[] = {
    PIN_CONF_PACKED(PIN_CONFIG_BIAS_DISABLE, 0),
    PIN_CONF_PACKED(PIN_CONFIG_DRIVE_STRENGTH, 4),   // 4mA
};

static unsigned long uart_sleep_configs[] = {
    PIN_CONF_PACKED(PIN_CONFIG_LEVEL, 0),            // force LOW
};

static const struct pinctrl_map board_mappings[] __initconst = {

    /* 1. UART: تفعيل الـ mux في الوضع الطبيعي */
    PIN_MAP_MUX_GROUP("foo-uart.0", PINCTRL_STATE_DEFAULT,
                      "pinctrl-foo", "uart0_grp", "uart0"),

    /* 2. UART: ضبط config لكل pin على حدة */
    PIN_MAP_CONFIGS_PIN_DEFAULT("foo-uart.0", "pinctrl-foo",
                                "UART0_TX", uart_default_configs),
    PIN_MAP_CONFIGS_PIN_DEFAULT("foo-uart.0", "pinctrl-foo",
                                "UART0_RX", uart_default_configs),

    /* 3. UART: state النوم - مجرد config بدون mux جديد */
    PIN_MAP_CONFIGS_PIN("foo-uart.0", PINCTRL_STATE_SLEEP,
                        "pinctrl-foo", "UART0_TX", uart_sleep_configs),

    /* 4. SPI: يدعم موقعين للـ runtime switching */
    PIN_MAP_MUX_GROUP("foo-spi.0", "pos-A",
                      "pinctrl-foo", "spi0_grp_a", "spi0"),
    PIN_MAP_MUX_GROUP("foo-spi.0", "pos-B",
                      "pinctrl-foo", "spi0_grp_b", "spi0"),

    /* 5. Hogging: الـ controller يحجز power pins لنفسه فوراً */
    PIN_MAP_MUX_GROUP_HOG_DEFAULT("pinctrl-foo", "power_grp", "power_func"),

    /* 6. Dummy: platform لا تحتاج pinctrl لكن الـ driver يتوقع state */
    PIN_MAP_DUMMY_STATE("foo-mmc.0", PINCTRL_STATE_DEFAULT),
};

/* التسجيل: */
pinctrl_register_mappings(board_mappings, ARRAY_SIZE(board_mappings));
```

---

## سادساً: الـ Registration Functions

```c
// تسجيل ثابت (static) عادةً في __init:
int pinctrl_register_mappings(const struct pinctrl_map *map,
                              unsigned int num_maps);

// تسجيل managed — يُلغى تلقائياً عند إزالة الـ device:
int devm_pinctrl_register_mappings(struct device *dev,
                                   const struct pinctrl_map *map,
                                   unsigned int num_maps);

// إلغاء التسجيل يدوياً:
void pinctrl_unregister_mappings(const struct pinctrl_map *map);

// يُخبر الـ core أن يتجاهل طلبات الـ pinctrl الفاشلة بدون error:
void pinctrl_provide_dummies(void);
```

### `pinctrl_provide_dummies()` — متى تستخدمها؟

عندما يكون لديك نفس الـ kernel يعمل على platforms مختلفة، بعضها يحتاج pinctrl وبعضها لا. بدلاً من ملء الـ mapping table بـ DUMMY_STATE entries لكل device، تستدعي هذه الدالة مرة واحدة وتقول للـ core: "لو لم تجد mapping لأي device، تجاهل الأمر بهدوء ولا ترفع error."

---

## سابعاً: الصورة الكاملة — كيف تتدفق المعلومات

```
Board/DTS                pinctrl_map[]              pinctrl core
─────────────────────────────────────────────────────────────────

"uart0 يحتاج            {dev="foo-uart.0",         يبحث عن
 uart0 function"    →    name="default",      →    device "foo-uart.0"
                         type=MUX_GROUP,            عند استدعاء
                         function="uart0"}          pinctrl_get()

                                                         │
                                                         ▼
                                              يستدعي get_function_name()
                                              في pinmux_ops حتى يجد
                                              function باسم "uart0"
                                                         │
                                                         ▼
                                              يستدعي set_mux()
                                              مع الـ selectors المناسبة
                                                         │
                                                         ▼
                                              الـ Hardware يُفعَّل ✓
```

الـ mapping table هي **قاموس** يربط الأسماء بعضها ببعض — أسماء devices، أسماء states، أسماء functions، أسماء groups — وكل هذه الأسماء يجب أن تتطابق بالضبط مع ما هو مسجَّل في الـ pinctrl driver.