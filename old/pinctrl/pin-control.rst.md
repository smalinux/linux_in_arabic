# شرح نظام Pinctrl بالعربي 🎛️

---

## ما هي المشكلة اللي بيحلها؟

تخيل إن عندك SoC زي RK3562 فيه مئات الـ **pins** (أرجل). كل رجل ممكن تشتغل بأكثر من طريقة:

```
Pin 42 → UART_TX
       → I2C_SDA
       → PWM
       → GPIO   ← الـ fallback الافتراضي
```

الـ Pinctrl هو اللي بيتحكم في كل ده.

---

## الـ 3 وظائف الأساسية

### 1. Pin Muxing — "إيه وظيفة الـ pin ده؟"

بيكتب في **mux registers** عشان يوصّل الـ pin لجهاز معين (UART، I2C، SPI…)

### 2. Pin Configuration — "إيه خصائصه الكهربية؟"

- **bias**: pull-up / pull-down / disable
- **drive-strength**: كام mA؟
- **open-drain** أو **push-pull**؟

### 3. GPIO Fallback — "لو مفيش حد عايزه، يبقى GPIO عادي"

أي pin مش محجوز بيرجع للـ GPIO subsystem.

---

## المصطلحات المهمة

|المصطلح|المعنى بالبسيط|
|---|---|
|**pin**|رجل فيزيائي واحد على الـ SoC|
|**pin group**|مجموعة pins بتشتغل مع بعض (مثلاً `uart2_pins` = TX + RX)|
|**function**|الوظيفة اللي بتتعمل على الـ group (مثلاً `uart2`)|
|**state**|صورة من الإعدادات ليها اسم: `default`، `sleep`، `idle`|
|**consumer**|أي driver بيطلب pins|
|**provider**|الـ SoC pinctrl driver اللي بيملك الـ hardware|
|**hog**|state بيتفعّل من أول ما الـ pinctrl يشتغل من غير ما حد يطلبه|

---

## ازاي بيشتغل؟ (Flow كامل)

```
uart_probe()
    │
    ▼
devm_pinctrl_get(dev)          ← "أنا محتاج pins"
    │  Pinctrl core بتقرأ الـ DT
    ▼
pinctrl_select_state("default")
    │  بتدور على الـ map entries
    ▼
set_mux() + pin_config_set()   ← Core بتكلم الـ SoC driver
    │
    ▼
SoC driver بيكتب في الـ registers على الـ hardware
```

الـ UART driver مش محتاج يعرف أي register — الـ Pinctrl بيعمل كل ده.

---

## الـ Device Tree — جانبين

### جانب Provider (في الـ SoC dtsi) — "تعريف الـ groups"

```dts
&pinctrl {
    uart2 {
        uart2m0_pins: uart2m0-pins {
            rockchip,pins =
                <1 RK_PB4 2 &pcfg_pull_up>,   /* RX */
                <1 RK_PB5 2 &pcfg_pull_none>;  /* TX */
        };
    };
};
```

### جانب Consumer (في الـ board dts) — "استخدام الـ groups"

```dts
&uart2 {
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&uart2m0_pins>;        /* default */
    pinctrl-1 = <&uart2m0_sleep_pins>;  /* sleep */
};
```

---

## قاعدة الـ Ownership

> **Pin واحد = consumer واحد بس في نفس الوقت**

لو driver تاني حاول يأخد نفس الـ pin → بياخد `**-EBUSY**` وخلاص.

---

## الـ States وإمتى بتتغير؟

|State|بيتفعّل إمتى؟|
|---|---|
|`default`|تلقائياً وقت الـ probe|
|`init`|قبل الـ probe (لو معرّفة)|
|`sleep`|عند runtime suspend|
|`idle`|manual من الـ driver|

```c
/* الـ driver بيغيّر manually */
pinctrl_pm_select_sleep_state(dev);   /* قبل النوم */
pinctrl_pm_select_default_state(dev); /* بعد الصحيان */
```

---

## Kernel Data Structures (بالبسيط)

```
pinctrl_desc          ← "الوصف الكامل للـ controller"
  ├── pinctrl_pin_desc[]   ← قائمة كل الـ pins
  ├── pinctrl_ops          ← get_groups, dt_node_to_map
  ├── pinmux_ops           ← set_mux  ← الأهم
  └── pinconf_ops          ← pin_config_set
```

---

## Debugfs — التشخيص

```bash
# كل الـ pins وحالتهم
cat /sys/kernel/debug/pinctrl/<ctrl>/pins

# مين شايل مين
cat /sys/kernel/debug/pinctrl/pinctrl-handles

# الـ mux assignments
cat /sys/kernel/debug/pinctrl/<ctrl>/pinmux-pins
```

---

## الأخطاء الشائعة

|الخطأ|السبب|الحل|
|---|---|---|
|`pin already requested`|نفس الـ pin في device تاني|راجع الـ DT|
|`-EPROBE_DEFER`|الـ pinctrl driver لسه مسجّلش|طبيعي، بيتري تاني|
|Pin مش بيتعمله config|`pin_config_set` بترجع `-ENOTSUPP`|أضف الـ `PIN_CONFIG_*` في driver|

---

**خلاصة:** الـ Pinctrl زي "مدير مرور" للـ pins — بيقرر مين يستخدم إيه، إمتى، وبأي إعدادات كهربية. الـ drivers مش محتاجة تعرف أي registers، بس تقول "أنا عايز الـ default state" والـ Pinctrl بيعمل الباقي.

# Linux Kernel Pinctrl Subsystem — الدليل الشامل

---

## 1. المفاهيم الأساسية — Core Definitions

### ما هو PIN CONTROLLER؟

هو **قطعة hardware** (عادةً مجموعة registers) تتحكم في الـ pins. تستطيع:

|القدرة|الوصف|
|---|---|
|**Multiplex**|تحديد وظيفة الـ pin (UART/SPI/GPIO...)|
|**Bias**|pull-up / pull-down / high impedance|
|**Drive strength**|قوة التيار الخارج من الـ pin|
|**Load capacitance**|سعة التحميل|

### ما هو PIN؟

أي خط دخل/خروج على الشريحة: pin, pad, ball, finger...  
كل pin له **رقم فريد** (unsigned integer) داخل **نطاق الـ controller** — النطاق محلي لكل controller.

---

## 2. مثال حقيقي: PGA Chip (8×8 = 64 pin)

```
        A   B   C   D   E   F   G   H
   8    o   o   o   o   o   o   o   o   ← pin 0..7
   7    o   o   o   o   o   o   o   o   ← pin 8..15
   6    o   o   o   o   o   o   o   o   ← pin 16..23
   5    o   o   o   o   o   o   o   o   ← pin 24..31
   4    o   o   o   o   o   o   o   o   ← pin 32..39
   3    o   o   o   o   o   o   o   o   ← pin 40..47
   2    o   o   o   o   o   o   o   o   ← pin 48..55
   1    o   o   o   o   o   o   o   o   ← pin 56..63
        │                           │
       pin 56 = "A1"            pin 63 = "H1"
       pin 0  = "A8"            pin 7  = "H8"
```

**الكود المقابل:**

```c
const struct pinctrl_pin_desc foo_pins[] = {
    PINCTRL_PIN(0,  "A8"),
    PINCTRL_PIN(1,  "B8"),
    ...
    PINCTRL_PIN(62, "G1"),
    PINCTRL_PIN(63, "H1"),
};

static struct pinctrl_desc foo_desc = {
    .name  = "foo",
    .pins  = foo_pins,
    .npins = ARRAY_SIZE(foo_pins),
    .owner = THIS_MODULE,
};

int __init foo_init(void) {
    struct pinctrl_dev *pctl;
    int error = pinctrl_register_and_init(&foo_desc, parent, NULL, &pctl);
    if (error) return error;
    return pinctrl_enable(pctl);
}
```

---

## 3. Pin Groups — تجميع الـ Pins

### المفهوم

الـ **group** هو مجموعة pins تعمل معاً لأداء وظيفة واحدة.  
مثال: SPI يحتاج {CLK, MISO, MOSI, CS} — لا معنى لتفعيل واحد منها دون الأخرى.

```
نفس الشريحة — مثال على مشاركة pin بين groups:

  pin 24 ──────────────────┐
                           ▼
  spi0_grp:  { 0,  8,  16, [24] }
  i2c0_grp:             { [24], 25 }
                           ▲
  pin 24 ──────────────────┘

  ⚠ لا يمكن تفعيل SPI و I2C في نفس الوقت لأنهما يشتركان في pin 24
```

### الكود

```c
static const unsigned int spi0_pins[] = { 0, 8, 16, 24 };
static const unsigned int i2c0_pins[] = { 24, 25 };

static const struct pingroup foo_groups[] = {
    PINCTRL_PINGROUP("spi0_grp", spi0_pins, ARRAY_SIZE(spi0_pins)),
    PINCTRL_PINGROUP("i2c0_grp", i2c0_pins, ARRAY_SIZE(i2c0_pins)),
};

static struct pinctrl_ops foo_pctrl_ops = {
    .get_groups_count = foo_get_groups_count,  // returns 2
    .get_group_name   = foo_get_group_name,    // "spi0_grp" or "i2c0_grp"
    .get_group_pins   = foo_get_group_pins,    // the actual pin arrays
};
```

---

## 4. Pinmux — ما هو؟

### المفهوم الكامل

**PINMUX** = تحديد الوظيفة الفعلية لكل pin على الـ hardware.  
السبب: pin واحد يمكن توصيله داخل الـ SoC بأكثر من دائرة silicon.

```
مثال من نفس الـ PGA 8×8:

Pins { A8, A7, A6, A5 } = { 0, 8, 16, 24 } ─── SPI port (CLK+RXD+TXD+FRM)
                                               OR
Pins { A5, B5 }         = { 24, 25 }        ─── I2C port (SCL+SDA)
                                               OR
نفس الـ SPI على pins أخرى:
Pins { G4, G3, G2, G1 } = { 38, 46, 54, 62 }─── SPI port أيضاً!

الصف السفلي:
Pins { A1..H1 } = { 56..63 }                ─── MMC bus (2/4/8 bit)
  - 2-bit: يستخدم { 56, 57 }
  - 4-bit: يستخدم { 56, 57, 58, 59 }
  - 8-bit: يستخدم { 56..63 } كاملاً
           ⚠ يحجب SPI البديل على { 38,46,54,62 }
```

---

## 5. Functions, Groups, Pins — العلاقة الثلاثية

```
┌─────────────────────────────────────────────────────────────┐
│                     FUNCTION "spi0"                         │
│                                                             │
│   groups: ┌─────────────────────────────────────────────┐  │
│           │  "spi0_0_grp"    │    "spi0_1_grp"          │  │
│           │  pins: 0,8,16,24 │    pins: 38,46,54,62     │  │
│           └─────────────────────────────────────────────┘  │
│           ↑ موقع A           ↑ موقع B (بديل)               │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                     FUNCTION "mmc0"                         │
│                                                             │
│   groups: ┌──────────┬──────────┬──────────────────────┐   │
│           │"mmc0_1"  │"mmc0_2"  │"mmc0_3"              │   │
│           │{56,57}   │{58,59}   │{60,61,62,63}         │   │
│           └──────────┴──────────┴──────────────────────┘   │
│           ↑ 2-bit    ↑ +2=4bit  ↑ +4=8bit total            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                     FUNCTION "i2c0"                         │
│   groups: ┌──────────────────────────────────────────────┐  │
│           │  "i2c0_grp"    pins: 24, 25                  │  │
│           └──────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### كيف تُنفَّذ في الكود

```c
// الـ functions مع groups المرتبطة بها
static const char *const spi0_groups[] = { "spi0_0_grp", "spi0_1_grp" };
static const char *const i2c0_groups[] = { "i2c0_grp" };
static const char *const mmc0_groups[] = { "mmc0_1_grp", "mmc0_2_grp", "mmc0_3_grp" };

static const struct pinfunction foo_functions[] = {
    PINCTRL_PINFUNCTION("spi0", spi0_groups, 2),
    PINCTRL_PINFUNCTION("i2c0", i2c0_groups, 1),
    PINCTRL_PINFUNCTION("mmc0", mmc0_groups, 3),
};

// set_mux: يكتب في الـ register المناسب
static int foo_set_mux(struct pinctrl_dev *pctldev,
                       unsigned int selector,  // index of function
                       unsigned int group)     // index of group
{
    u8 regbit = BIT(group);
    writeb((readb(MUX) | regbit), MUX);
    return 0;
}
```

### ماذا يحدث لو طلبت SPI و I2C معاً؟

```
Driver A يطلب spi0 (group: spi0_0_grp → pins 0,8,16,24)
Driver B يطلب i2c0 (group: i2c0_grp  → pins 24,25)

Pinmux core يفحص:
  pin 24 → مستخدم من spi0 ✗ conflict!
  يرفض طلب i2c0 → يُرجع -EBUSY

✓ الـ driver لا يحتاج أن يتحقق بنفسه — الـ core يحميه
```

---

## 6. Pin Configuration — ضبط الخصائص الكهربائية

### أنواع الـ configuration

```
Physical Pin
     │
     ├─ Pull-up   ── يربط الـ pin بـ VDD عبر resistor
     ├─ Pull-down ── يربط الـ pin بـ GND عبر resistor  
     ├─ Hi-Z      ── يفصل الـ pin (tristate / high impedance)
     ├─ Open-drain── يسمح فقط بالـ sink (بدون source)
     ├─ Drive strength ── قوة التيار (2mA / 4mA / 8mA...)
     └─ Slew rate  ── سرعة تغيير المستوى الكهربائي
```

### الكود

```c
static struct pinconf_ops foo_pconf_ops = {
    .pin_config_get        = foo_pin_config_get,        // قراءة config
    .pin_config_set        = foo_pin_config_set,        // كتابة config لـ pin واحد
    .pin_config_group_get  = foo_pin_config_group_get,  // قراءة لـ group
    .pin_config_group_set  = foo_pin_config_group_set,  // كتابة لـ group
};

// مثال على set:
static int foo_pin_config_set(struct pinctrl_dev *pctldev,
                              unsigned int offset,
                              unsigned long config)
{
    struct my_conftype *conf = (struct my_conftype *)config;
    switch (conf) {
        case PLATFORM_X_PULL_UP:
            // اكتب في register الـ pull-up
            break;
    }
}
```

---

## 7. GPIO Ranges — ربط GPIO بالـ Pins

### المشكلة

```
GPIO Subsystem يستخدم أرقام عالمية (global GPIO numbers)
Pin Controller يستخدم أرقام محلية (local pin numbers)

نحتاج mapping بينهما!
```

### مثال معقد: controller واحد يخدم GPIO chipين

```
Pin Controller "foo"
        │
        ├──── GPIO chip_a (16 pins)
        │         GPIO base=32, pin_base=32
        │
        └──── GPIO chip_b (8 pins)
                  GPIO base=48, pin_base=64
                  (offset مختلف!)

Mapping:
┌──────────────────────────────────────────────────────────────┐
│  GPIO namespace    │  Pin Controller namespace               │
├──────────────────────────────────────────────────────────────┤
│  gpio 32           │  pin 32  "A0"   ← chip_a               │
│  gpio 33           │  pin 33  "A1"                           │
│  ...               │  ...                                    │
│  gpio 47           │  pin 47  "A15"                          │
├──────────────────────────────────────────────────────────────┤
│  gpio 48           │  pin 64  "B0"   ← chip_b (offset!)     │
│  gpio 49           │  pin 65  "B1"                           │
│  ...               │  ...                                    │
│  gpio 55           │  pin 71  "B7"                           │
└──────────────────────────────────────────────────────────────┘

formula: pin_number = (gpio - gpio_base) + pin_base
chip_a:  pin = (gpio - 32) + 32 = gpio       (linear, no offset)
chip_b:  pin = (gpio - 48) + 64 = gpio + 16  (offset of 16)
```

### الكود

```c
static struct pinctrl_gpio_range gpio_range_a = {
    .name     = "chip a",
    .base     = 32,       // أول رقم GPIO
    .pin_base = 32,       // أول pin مقابل
    .npins    = 16,
    .gc       = &chip_a,
};

static struct pinctrl_gpio_range gpio_range_b = {
    .name     = "chip b",
    .base     = 48,       // أول رقم GPIO
    .pin_base = 64,       // أول pin (مختلف!)
    .npins    = 8,
    .gc       = &chip_b,
};

// Sparse mapping (pins غير متسلسلة):
static const unsigned int range_pins[] = { 14, 1, 22, 17, 10, 8, 6, 2 };
static struct pinctrl_gpio_range gpio_range_sparse = {
    .base  = 32,
    .pins  = &range_pins,   // تُستخدم بدل pin_base
    .npins = ARRAY_SIZE(range_pins),
};
```

---

## 8. GPIO Mode Pitfalls — فخ "GPIO Mode"

### المشكلة

```
Datasheet يقول: "ضع الـ pin في GPIO mode أثناء النوم"
المطور يفكر: "إذن أحتاج gpio API"
الحقيقة:    "هذا مجرد pin configuration، ليس GPIO حقيقي!"
```

### التصميمان الممكنان للـ Hardware

```
تصميم (A) — GPIO و peripherals مفصولان:
                    pin config regs
                         │
                         ▼            ┌── SPI
  Physical pins ──── pad ──── pinmux ─┼── I2C
                                      ├── MMC
                                      └── GPIO ← دائرة منفصلة

  GPIO orthogonal = يمكن قراءته حتى لو الـ pin في وضع SPI
  flag "strict" = يمنع تعارض GPIO و pinmux على نفس الـ pin

تصميم (B) — GPIO و peripherals مدمجان:
                    pin config regs
                         │
  Physical pins ──┬─ pad ──── pinmux ─┬── SPI
                  │              │    ├── I2C
                  └── GPIO       │    └── MMC
                      (دائماً    pin multiplex regs
                       متصلة)

  ⚠ GPIO يمكنه "التجسس" على SPI traffic
  ⚠ يمكن إفساد الـ traffic بالكتابة على GPIO
```

### الحل الصحيح: استخدم pinctrl states بدل GPIO API

```c
// ❌ الطريقة الخاطئة: التفكير أنك تحتاج gpio API
gpiod_direction_output(pin, 0);  // لا! هذا ليس GPIO حقيقي

// ✅ الطريقة الصحيحة: pinctrl states
struct pinctrl_state *pins_default;  // UART mode
struct pinctrl_state *pins_sleep;    // "GPIO mode" = LOW output

// في الـ device tree / mapping:
static unsigned long uart_sleep_mode[] = {
    PIN_CONF_PACKED(PIN_CONFIG_LEVEL, 0),  // output LOW
};

static struct pinctrl_map pinmap[] = {
    // Normal: UART function
    PIN_MAP_MUX_GROUP("uart", PINCTRL_STATE_DEFAULT,
                      "pinctrl-foo", "u0_group", "u0"),

    // Sleep: "GPIO mode" = just an electrical config
    PIN_MAP_MUX_GROUP("uart", PINCTRL_STATE_SLEEP,
                      "pinctrl-foo", "u0_group", "gpio-mode"),
    PIN_MAP_CONFIGS_PIN("uart", PINCTRL_STATE_SLEEP,
                        "pinctrl-foo", "UART_TX_PIN", uart_sleep_mode),
};

// في الـ UART driver:
pinctrl_select_state(pinctrl, pins_default);  // normal
pinctrl_select_state(pinctrl, pins_sleep);    // sleep/low
```

---

## 9. Board/Machine Configuration — ربط كل شيء معاً

### الـ Mapping Table

هي الجسر بين **device** و **function** و **group** على **pin controller** معين.

```
┌─────────────────────────────────────────────────────────────────┐
│                        pinctrl_map entry                        │
│                                                                 │
│  dev_name:      "foo-spi.0"     ← اسم الـ device               │
│  name:          "default"       ← اسم الـ state                 │
│  type:          MUX_GROUP       ← نوع الـ mapping               │
│  ctrl_dev_name: "pinctrl-foo"   ← أي pin controller            │
│  function:      "spi0"          ← أي function                  │
│  group:         "spi0_0_grp"    ← أي group (اختياري)           │
└─────────────────────────────────────────────────────────────────┘
```

### مثال كامل: I2C مع config

```c
static unsigned long i2c_grp_configs[] = {
    FOO_PIN_DRIVEN,       // push-pull output
    FOO_PIN_PULLUP,       // pull-up resistor
};

static unsigned long i2c_pin_configs[] = {
    FOO_OPEN_COLLECTOR,   // open-drain للـ I2C
    FOO_SLEW_RATE_SLOW,   // slew rate منخفض
};

static struct pinctrl_map mapping[] = {
    // 1. فعّل الـ mux لـ i2c0
    PIN_MAP_MUX_GROUP("foo-i2c.0", PINCTRL_STATE_DEFAULT,
                      "pinctrl-foo", "i2c0", "i2c0"),

    // 2. اضبط config للـ group كاملاً
    PIN_MAP_CONFIGS_GROUP("foo-i2c.0", PINCTRL_STATE_DEFAULT,
                          "pinctrl-foo", "i2c0", i2c_grp_configs),

    // 3. اضبط config لكل pin بشكل منفصل
    PIN_MAP_CONFIGS_PIN("foo-i2c.0", PINCTRL_STATE_DEFAULT,
                        "pinctrl-foo", "i2c0scl", i2c_pin_configs),
    PIN_MAP_CONFIGS_PIN("foo-i2c.0", PINCTRL_STATE_DEFAULT,
                        "pinctrl-foo", "i2c0sda", i2c_pin_configs),
};
```

---

## 10. Runtime Pinmuxing — تغيير الـ Mux أثناء التشغيل

### مثال: نقل SPI بين موقعين

```
موقع A: pins { 0, 8, 16, 24 }  → "spi0-pos-A"
موقع B: pins { 38, 46, 54, 62 }→ "spi0-pos-B"

 probe()                      runtime
    │                            │
    ├─ lookup "pos-A" → s1       ├─ select(s1) → SPI على موقع A
    ├─ lookup "pos-B" → s2       │
    │                            ├─ ... do something ...
    │                            │
    │                            └─ select(s2) → SPI على موقع B
```

```c
// في probe():
p  = devm_pinctrl_get(&device);
s1 = pinctrl_lookup_state(p, "pos-A");
s2 = pinctrl_lookup_state(p, "pos-B");

// في runtime:
pinctrl_select_state(p, s1);  // تفعيل موقع A
// ...
pinctrl_select_state(p, s2);  // تفعيل موقع B
```

---

## 11. Pin Control من منظور الـ Device Driver

### دورة حياة الـ Driver مع pinctrl

```
device probe() يُستدعى
       │
       ▼
devm_pinctrl_get(dev)
       │ يُخصّص handle لكل pinctrl info للـ device
       │ يقرأ الـ mapping table
       │ يُرجع -EPROBE_DEFER لو الـ pinctrl driver لم يُسجّل بعد!
       ▼
pinctrl_lookup_state(p, PINCTRL_STATE_DEFAULT)
       │ يبحث عن state بالاسم في الـ mapping table
       ▼
pinctrl_select_state(p, state)
       │ يُبرمج الـ hardware registers
       │ يستدعي set_mux() و pin_config_set() في الـ driver
       ▼
Device يعمل بشكل طبيعي
       │
       │ (عند السكون — power management)
       ▼
pinctrl_pm_select_sleep_state(dev)
       │
       ▼
pinctrl_pm_select_default_state(dev)
       │
       ▼
devm_pinctrl_put() ← تلقائي عند إزالة الـ device
```

### الـ PM States الجاهزة

```
State                │ متى تُستخدم
─────────────────────┼────────────────────────────────────
"default"            │ التشغيل العادي، تُفعَّل قبل probe()
"init"               │ تُفعَّل قبل probe() إذا وُجدت
                     │ (الـ default تُفعَّل بعد probe)
"sleep"              │ عند الـ system sleep
"idle"               │ عند الـ runtime idle
```

### مثال suspend/resume

```c
foo_suspend() {
    /* ... suspend device ... */
    pinctrl_pm_select_sleep_state(dev);  // تغيير الـ pins للوضع الاقتصادي
}

foo_resume() {
    pinctrl_pm_select_init_state(dev);   // إعادة التهيئة
    /* ... resume device ... */
    pinctrl_pm_select_default_state(dev); // العودة للوضع الطبيعي
}
```

---

## 12. System Hogging — الـ Pin Controller يحجز pins لنفسه

```
عند تسجيل الـ pin controller، يمكنه حجز functions معينة فوراً
(مثلاً: power management pins لا يجب أن يأخذها أي driver آخر)

المعيار: dev_name == ctrl_dev_name و state == "default"
```

```c
// هذا الـ entry يُفعَّل فوراً عند تسجيل "pinctrl-foo":
{
    .dev_name      = "pinctrl-foo",   // ← نفس اسم الـ controller!
    .name          = PINCTRL_STATE_DEFAULT,
    .ctrl_dev_name = "pinctrl-foo",
    .function      = "power_func",
},

// أو باستخدام الـ macro:
PIN_MAP_MUX_GROUP_HOG_DEFAULT("pinctrl-foo", NULL, "power_func")
```

---

## 13. Debugfs — أدوات التشخيص

```
/sys/kernel/debug/pinctrl/
│
├── pinctrl-devices     ← قائمة كل controllers مع دعم pinmux/pinconf
├── pinctrl-handles     ← handles المفعّلة حالياً
├── pinctrl-maps        ← كل الـ mappings المسجّلة
│
└── pinctrl-foo/        ← مجلد لكل controller
    ├── pins            ← كل pins مع محتوى الـ registers (اختياري)
    ├── gpio-ranges     ← ربط GPIO بالـ pins
    ├── pingroups       ← كل الـ groups
    ├── pinconf-pins    ← config لكل pin
    ├── pinconf-groups  ← config لكل group
    ├── pinmux-functions← كل الـ functions مع groups مرتبطة
    ├── pinmux-pins     ← من يستخدم كل pin (device أو GPIO)
    └── pinmux-select   ← كتابة فيه تُفعّل function يدوياً:
                          echo "spi0_0_grp spi0" > pinmux-select
```

---

## 14. الصورة الكاملة — Full Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              Linux Kernel                                    │
│                                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │SPI Driver│  │I2C Driver│  │MMC Driver│  │UART Driv.│  │ GPIO Driver  │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬───────┘  │
│       │              │              │              │              │           │
│       │  pinctrl_get/select_state() │              │      gpiod_get()        │
│       └──────────────┴──────────────┴──────────────┘              │           │
│                                  │                                │           │
│                     ┌────────────▼────────────┐    ┌─────────────▼─────────┐ │
│                     │    pinctrl CORE          │◄───│    GPIO Subsystem     │ │
│                     │                         │    │  pinctrl_gpio_request │ │
│                     │  - conflict detection   │    └───────────────────────┘ │
│                     │  - state management     │                              │
│                     │  - mapping table        │                              │
│                     └────────────┬────────────┘                              │
│                                  │ استدعاء الـ ops                           │
│                     ┌────────────▼────────────────────────────────────────┐  │
│                     │              pinctrl_desc (your BSP driver)         │  │
│                     │                                                     │  │
│                     │  pinctrl_ops     pinmux_ops        pinconf_ops      │  │
│                     │  ─────────────  ──────────────     ──────────────   │  │
│                     │  get_groups     get_functions      pin_config_get   │  │
│                     │  dt_node_to_map set_mux            pin_config_set   │  │
│                     │                gpio_set_direction                   │  │
│                     └────────────────────────┬────────────────────────────┘  │
│                                              │ register read/write            │
│                     ┌────────────────────────▼────────────────────────────┐  │
│                     │                  SoC Hardware                       │  │
│                     │                                                     │  │
│                     │  MUX registers  │  CONFIG registers  │  GPIO regs  │  │
│                     │  ──────────────────────────────────────────────     │  │
│                     │  pin0:spi_clk   │  pin0:pull-up      │  pin0:in   │  │
│                     │  pin1:spi_mosi  │  pin1:pull-up      │  pin1:out  │  │
│                     │  ...            │  ...               │  ...       │  │
│                     └──────────────────────────────────────────────────── ┘  │
│                                              │                               │
│                     ┌────────────────────────▼────────────────────────────┐  │
│                     │                Physical Pins                        │  │
│                     │     A8  B8  C8 ... G1  H1                          │  │
│                     └─────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 15. ملخص API المهمة للـ Driver Developer

```
Registration (BSP / Controller Driver):
  pinctrl_register_and_init() + pinctrl_enable()
  devm_pinctrl_register_and_init()          ← الأفضل دائماً

Consumer (Device Driver):
  devm_pinctrl_get(dev)                     ← احصل على handle
  pinctrl_lookup_state(p, "default")        ← ابحث عن state
  pinctrl_select_state(p, state)            ← فعّل الـ state
  ─ أو باختصار ─
  devm_pinctrl_get_select_default(dev)      ← الثلاثة في واحد

Power Management:
  pinctrl_pm_select_default_state(dev)
  pinctrl_pm_select_sleep_state(dev)
  pinctrl_pm_select_idle_state(dev)
  pinctrl_pm_select_init_state(dev)

GPIO Integration:
  pinctrl_add_gpio_range(pctl, range)
  pinctrl_find_gpio_range_from_pin(pctl, pin)
  pinctrl_get_group_pins(pctl, group, &pins, &npins)
```

---

## 16. نقاط مهمة يجب تذكّرها

| الموضوع                     | القاعدة                                                       |
| --------------------------- | ------------------------------------------------------------- |
| **Conflict detection**      | الـ core يمنع تعارض الـ pins تلقائياً                         |
| **-EPROBE_DEFER**           | لو الـ pinctrl لم يُسجّل بعد، أعد المحاولة                    |
| **GPIO API pitfall**        | "GPIO mode" في الـ datasheet ≠ `gpiod_get()` في الغالب        |
| **strict flag**             | امنع GPIO و pinmux على نفس الـ pin في نفس الوقت               |
| **devm_ prefix**            | استخدمها دائماً لتجنب memory leaks                            |
| **First-come first-served** | من يطلب الـ pin أولاً يحصل عليه                               |
| **Process context**         | `pinctrl_select_state()` ليس safe دائماً من interrupt context |