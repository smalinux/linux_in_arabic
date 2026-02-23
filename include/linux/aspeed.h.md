## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ ASPEED؟

**ASPEED Technology** شركة تايوانية بتصنع chips اسمها **AST2400 / AST2500 / AST2600**. الـ chips دي بتتحط في السيرفرات كـ **BMC (Baseboard Management Controller)** — يعني زي "مدير صامت" جوه السيرفر بيشتغل حتى لو السيرفر الأساسي اتقفل أو عمله hang. بيتحكم في الطاقة، الحرارة، الـ fans، وبيوفر remote access للـ admins.

الـ ASPEED chips دي عندها core أساسي هو **ARM** بيشغّل Linux كـ BMC OS، وجنبيه كمان في **coprocessor** تاني أصغر هو **ColdFire** (أو نوع مماثل حسب الجيل).

---

### القصة: بصّ، عندنا مشكلة!

تخيّل معايا السيناريو ده:

```
┌─────────────────────────────────────────────────────────────┐
│              ASPEED BMC Chip                                │
│                                                             │
│   ┌─────────────┐          ┌───────────────────────────┐   │
│   │  ARM Core   │          │  ColdFire Coprocessor     │   │
│   │  (Linux)    │          │  (real-time firmware)     │   │
│   └──────┬──────┘          └────────────┬──────────────┘   │
│          │                              │                   │
│          └────────────┬─────────────────┘                   │
│                       │                                     │
│               ┌───────┴────────┐                            │
│               │  GPIO Hardware │                            │
│               │  (pins A, B, C │                            │
│               │   D, E, F ...) │                            │
│               └────────────────┘                            │
└─────────────────────────────────────────────────────────────┘
```

الـ ARM core بيشغّل Linux وبيتحكم في الـ **GPIO pins** عن طريق الـ kernel GPIO subsystem — ده الطبيعي.

لكن في نفس الوقت، الـ **ColdFire coprocessor** كمان محتاج يتحكم في بعض الـ GPIO pins دي في real-time عشان يتواصل مع IBM POWER servers عبر بروتوكول اسمه **FSI (Flexible Service Interface)** — بروتوكول قديم من IBM بيتكلم عبر GPIO bit-banging بسرعة عالية جداً ومش ممكن يتحملها Linux بسبب الـ scheduling latency.

**المشكلة:** إزاي يشتغلوا الاتنين على نفس الـ GPIO pins من غير ما كل واحد يلخبط على التاني؟

---

### الحل: نظام التحكيم (Arbitration)

الـ header file ده، `include/linux/gpio/aspeed.h`، هو الـ **contract** (العقد) بين:
- **GPIO driver** (`drivers/gpio/gpio-aspeed.c`) — المسؤول عن إدارة الـ GPIO pins
- **FSI master driver** (`drivers/fsi/fsi-master-ast-cf.c`) — اللي بيحتاج يسلّم الـ pins للـ ColdFire coprocessor

الـ header بيعرّف **API بسيط جداً** من 3 functions + struct واحدة:

```c
/* Callbacks يوفرهم driver الـ FSI عشان GPIO driver يتصل بيه */
struct aspeed_gpio_copro_ops {
    int (*request_access)(void *data);   /* ARM طالب الـ GPIO — ابقى جاهز */
    int (*release_access)(void *data);   /* ARM خلّص — ColdFire ممكن يرجع */
};

/* سجّل الـ callbacks دي مع GPIO driver */
int aspeed_gpio_copro_set_ops(const struct aspeed_gpio_copro_ops *ops, void *data);

/* قول للـ GPIO driver: الـ pin ده بتاع ColdFire */
int aspeed_gpio_copro_grab_gpio(struct gpio_desc *desc,
                                u16 *vreg_offset, u16 *dreg_offset, u8 *bit);

/* قول للـ GPIO driver: الـ pin ده رجع لـ ARM */
int aspeed_gpio_copro_release_gpio(struct gpio_desc *desc);
```

---

### الصورة الكبيرة: إيه اللي بيحصل فعلاً؟

```
  FSI Driver يشغّل                      GPIO Driver يتحكم
  ──────────────────                     ──────────────────
  aspeed_gpio_copro_set_ops()  ──────►  بيحفظ الـ callbacks

  aspeed_gpio_copro_grab_gpio() ──────► بيعلّم الـ bank كله "تحت سيطرة ColdFire"
                                        وبيرجع للـ FSI driver الـ register offsets

  ARM يحاول يكتب/يقرأ GPIO:
                               ──────► GPIO driver يكتشف إن الـ bank ده تبع ColdFire
                               ──────► بيكلّم request_access() (بيقول للـ FSI: "أنا عايز")
                               ──────► FSI driver يوقّف ColdFire مؤقتاً
                               ──────► ARM يعمل اللي هو عايزه
                               ──────► release_access() ← ColdFire يشتغل تاني
```

ده **handshaking protocol** بسيط وأنيق بين عالمين: عالم real-time (ColdFire) وعالم Linux (ARM).

---

### ليه ده مهم؟

في IBM POWER servers، الـ BMC بيحتاج يتواصل مع الـ POWER chip نفسه عبر **FSI** لأغراض زي:
- تشخيص المشاكل (debug)
- تحديث الـ firmware
- إعادة تشغيل أجزاء من الـ chip

الـ FSI بروتوكول timing-sensitive جداً — محتاج GPIO bit-banging بدقة microsecond. Linux مش قادر يضمن الـ timing ده بسبب الـ kernel scheduling. فالحل هو تسليم المهمة للـ ColdFire coprocessor اللي بيشتغل بدون OS ويقدر يلتزم بالـ timing.

لكن مش كل الوقت الـ ColdFire شاغل — أحياناً Linux محتاج يتحكم في نفس الـ GPIO pins لأسباب تانية. فالـ API ده بيوفر الـ **mutual exclusion** بين الاتنين.

---

### الـ Subsystem

الـ header ده بينتمي لـ **ARM/ASPEED MACHINE SUPPORT** subsystem في MAINTAINERS، وبيتوافق مع الـ GPIO subsystem اللي بيشمل:

| Section | المسؤول |
|---|---|
| ARM/ASPEED MACHINE SUPPORT | Joel Stanley, Andrew Jeffery |
| ASPEED PINCTRL DRIVERS | Andrew Jeffery |

---

### الملفات المهمة اللي لازم تعرفها

| الملف | الدور |
|---|---|
| `include/linux/gpio/aspeed.h` | الـ header ده — الـ public API للـ copro arbitration |
| `drivers/gpio/gpio-aspeed.c` | GPIO driver الأساسي — بينفّذ الـ API |
| `drivers/fsi/fsi-master-ast-cf.c` | المستخدم الوحيد للـ API ده — FSI/ColdFire driver |
| `Documentation/devicetree/bindings/gpio/aspeed,ast2400-gpio.yaml` | Device tree binding للـ GPIO |
| `Documentation/devicetree/bindings/gpio/aspeed,sgpio.yaml` | الـ SGPIO (Serial GPIO) variant |
| `arch/arm/mach-aspeed/` | الـ machine support للـ AST SoCs |
| `drivers/gpio/gpio-aspeed-sgpio.c` | driver الـ SGPIO المنفصل |
## Phase 2: شرح الـ ASPEED GPIO Co-Processor Framework

### المشكلة — ليه الـ Framework ده موجود؟

الـ **ASPEED AST2xxx** هي SoCs بتُستخدم في **BMC (Baseboard Management Controllers)** — يعني الـ chip اللي بتتحكم في السيرفر حتى لو الـ main CPU واقف. الـ ASPEED SoC فيها اتنين processors:

1. **ARM Cortex-A7** — الـ main CPU اللي بيشغّل Linux.
2. **Co-Processor (CoPro)** — microcontroller داخلي بيشتغل بشكل مستقل، غالباً بيتحكم في GPIO pins بسرعة عالية أو بـ real-time requirements.

المشكلة: الاتنين محتاجين يوصلوا نفس الـ GPIO hardware. لو الـ ARM Linux driver اتحكم في GPIO pin من غير ما يعرف إن الـ CoPro بيستخدمه في نفس اللحظة، هتحصل **race condition** على مستوى hardware — يعني اتنين بيكتبوا على نفس الـ register في نفس الوقت، والنتيجة undefined behavior أو damage للـ hardware state.

**السؤال**: إزاي Linux يشارك GPIO pins مع CoPro من غير ما يكسّر حاجة؟

---

### الحل — الـ Approach اللي الـ Kernel اتخده

الـ `include/linux/gpio/aspeed.h` بيعرّف **واجهة تنسيق صغيرة** بين الـ Linux GPIO driver والـ CoPro firmware. الفكرة بسيطة:

- الـ Linux driver عنده **lock protocol**: قبل ما يمس أي GPIO, بيسأل الـ CoPro "هل تسمحلي أوصل؟"
- الـ CoPro firmware بيرد بـ grant أو بـ wait.
- بعد ما Linux خلص، بيقول للـ CoPro "خلصت، الـ GPIO تبعك تاني."

ده مش mutex عادي في kernel — ده **cross-processor arbitration** على مستوى الـ firmware/hardware.

---

### Analogy — مقارنة بالواقع

تخيّل مبنى فيه **غرفة servers مشتركة** بين فريقين:
- **فريق أول (Linux ARM)**: بيدخل الغرفة من وقت لوقت عشان يعمل maintenance.
- **فريق تاني (CoPro)**: بيشتغل جوا الغرفة باستمرار بـ 24/7 وعنده مفاتيح.

القاعدة: لما الفريق الأول عايز يدخل، لازم يطلب الدخول من الـ security (CoPro). الـ security بيقفل الغرفة بشكل مؤقت، بيسمح للفريق الأول يدخل ويعمل شغله، وبعدين بيفتح تاني للفريق التاني.

الـ mapping على الـ kernel concepts:

| الـ Analogy | الـ Kernel Concept |
|---|---|
| فريق أول (Linux) | ARM Cortex-A7 running Linux GPIO driver |
| فريق تاني (CoPro) | ASPEED internal Co-Processor firmware |
| غرفة الـ servers | GPIO hardware registers (shared MMIO) |
| طلب الدخول | `aspeed_gpio_copro_grab_gpio()` |
| الـ security يسمح | `request_access()` callback → CoPro grants lock |
| الخروج وإعادة المفتاح | `aspeed_gpio_copro_release_gpio()` |
| الـ security يفتح تاني | `release_access()` callback → CoPro resumes |
| عنوان الغرفة | `vreg_offset`, `dreg_offset`, `bit` — exact register location |

---

### الـ Big Picture — Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        ASPEED AST2xxx SoC                       │
│                                                                  │
│  ┌──────────────────────┐      ┌──────────────────────────────┐ │
│  │  ARM Cortex-A7       │      │   Internal Co-Processor      │ │
│  │  (Linux)             │      │   (CoPro firmware)           │ │
│  │                      │      │                              │ │
│  │  aspeed_gpio_driver  │      │  Independent firmware loop   │ │
│  │        │             │      │         │                    │ │
│  │  aspeed_gpio_copro_  │      │  aspeed_gpio_copro_ops       │ │
│  │  grab_gpio()         │◄────►│  .request_access()           │ │
│  │  release_gpio()      │      │  .release_access()           │ │
│  └──────────┬───────────┘      └──────────┬───────────────────┘ │
│             │                             │                      │
│             └──────────┬──────────────────┘                      │
│                        ▼                                         │
│              ┌─────────────────────┐                             │
│              │   GPIO Hardware     │                             │
│              │   Registers (MMIO)  │                             │
│              │                     │                             │
│              │  Value Reg (vreg)   │ ← read/write GPIO state     │
│              │  Dir   Reg (dreg)   │ ← input/output direction    │
│              │  Bit offset         │ ← which pin exactly         │
│              └─────────────────────┘                             │
└─────────────────────────────────────────────────────────────────┘
```

**الـ aspeed_gpio_copro_set_ops()** بتربط الـ Linux driver بالـ CoPro firmware — بتسجّل الـ callbacks اللي الـ driver هيستدعيها لما يحتاج يتفاوض على access.

---

### مكان الـ Subsystem في الـ Kernel Stack

```
┌─────────────────────────────────────────────┐
│         Userspace / sysfs / libgpiod         │
├─────────────────────────────────────────────┤
│         GPIO Consumer API                    │
│   (gpiod_get, gpiod_set_value, ...)          │
├─────────────────────────────────────────────┤
│         GPIOLIB Core (gpiolib.c)             │
│   struct gpio_desc  ←  central abstraction   │
├─────────────────────────────────────────────┤
│         struct gpio_chip (driver ops)         │
│   .get(), .set(), .direction_input(), ...    │
├─────────────────────────────────────────────┤
│   ★  ASPEED GPIO Driver (gpio-aspeed.c)  ★  │
│      + aspeed_gpio_copro_* interface         │  ← نحن هنا
├─────────────────────────────────────────────┤
│         ASPEED GPIO Hardware Registers       │
│         (MMIO — Value/Dir Registers)         │
└─────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ header بيعرّف **ثلاثة concepts أساسية**:

#### 1. `struct aspeed_gpio_copro_ops`

```c
struct aspeed_gpio_copro_ops {
    /* called before Linux touches the GPIO register */
    int (*request_access)(void *data);

    /* called after Linux finishes with the GPIO register */
    int (*release_access)(void *data);
};
```

ده **vtable** — interface بالـ function pointers. الـ Linux driver مش عارف إيه الـ CoPro firmware بيعمل داخلياً. كل اللي يعرفه: "في حاجة بتحكم الـ CoPro، وأنا هتصل بيها قبل وبعد ما أمس الـ hardware."

الـ `void *data` ده **opaque context pointer** — بيتبعت للـ callback عشان يعرف يوصل لـ firmware state بتاعته من غير ما الـ kernel driver يعرف تفاصيله.

#### 2. `aspeed_gpio_copro_grab_gpio()`

```c
int aspeed_gpio_copro_grab_gpio(struct gpio_desc *desc,
                                u16 *vreg_offset,
                                u16 *dreg_offset,
                                u8  *bit);
```

دي الـ function اللي بتعمل اتنين حاجات في خطوة واحدة:
- بتستدعي `request_access()` عشان تأخذ إذن من الـ CoPro.
- بترجع **العنوان الدقيق** للـ GPIO register: `vreg_offset` (value register), `dreg_offset` (direction register), و`bit` (رقم الـ bit داخل الـ register).

ليه بترجع الـ offsets؟ لأن لما الـ Linux driver عايز يتعامل مع GPIO pin معين للـ CoPro، محتاج يعرف بالضبط أي register وأي bit — مش بس الـ `gpio_desc` abstractly.

#### 3. `aspeed_gpio_copro_set_ops()`

```c
int aspeed_gpio_copro_set_ops(const struct aspeed_gpio_copro_ops *ops,
                               void *data);
```

دي **registration function** — الـ CoPro firmware driver بيستدعيها عشان يسجّل الـ callbacks بتاعته مع الـ GPIO driver. بعدها الـ GPIO driver عنده طريقة يكلّم الـ CoPro.

---

### الـ struct Relationships

```
aspeed_gpio_copro_set_ops()
        │
        │  registers
        ▼
┌─────────────────────────────────┐
│  aspeed_gpio_copro_ops          │
│  ┌──────────────────────────┐   │
│  │ request_access(void*)    │   │  ← CoPro firmware implements this
│  │ release_access(void*)    │   │  ← CoPro firmware implements this
│  └──────────────────────────┘   │
│  + opaque void *data            │  ← CoPro context pointer
└─────────────────────────────────┘
        ▲
        │  called by
        │
aspeed_gpio_copro_grab_gpio(gpio_desc*, vreg*, dreg*, bit*)
        │
        │  takes/returns
        ▼
┌─────────────────────────────────┐
│  gpio_desc                      │  ← GPIOLIB abstraction (kernel core)
│  (opaque — defined in gpiolib)  │
└─────────────────────────────────┘
        │
        │  maps to
        ▼
┌─────────────────────────────────┐
│  Physical GPIO Register         │
│  vreg_offset: value register    │
│  dreg_offset: direction register│
│  bit:         pin bit index     │
└─────────────────────────────────┘
```

---

### الـ gpio_desc — الـ Dependency الأساسية

قبل ما تكمّل، لازم تفهم الـ **`struct gpio_desc`**:

الـ `gpio_desc` هو الـ **opaque handle** اللي الـ GPIOLIB بيستخدمه عشان يمثّل GPIO pin واحد. الـ consumer (driver) بياخد `gpio_desc*` من `gpiod_get()` ويتعامل معاه من غير ما يعرف التفاصيل الداخلية. الـ GPIOLIB Core (مش الـ ASPEED driver) هو اللي عارف الـ struct بالكامل.

---

### إيه اللي الـ Subsystem ده بيمتلكه vs. اللي بيفوّضه؟

| المسؤولية | مين بيعملها |
|---|---|
| تعريف protocol التفاوض على access | **ASPEED GPIO CoPro interface** (الـ header ده) |
| تنفيذ `request_access` / `release_access` | **CoPro firmware driver** (external) |
| التعامل مع الـ GPIO register فعلياً | **ASPEED GPIO driver** (`gpio-aspeed.c`) |
| تحويل `gpio_desc` لـ hardware info | **ASPEED GPIO driver** داخل `grab_gpio()` |
| إدارة الـ gpio_desc lifecycle | **GPIOLIB Core** |
| expose الـ GPIO لـ userspace | **GPIOLIB Core + sysfs/chardev** |
| الـ IRQ handling للـ GPIO pins | **ASPEED GPIO driver** مع `gpio_irq_chip` |

---

### مثال Real-World — Flow كامل

تخيّل إن الـ CoPro firmware بيتحكم في **fan control GPIO** (pin لـ PWM fan)، وفي نفس الوقت Linux محتاج يقرأ حالته:

```c
/* 1. CoPro firmware registers its arbitration callbacks */
static int copro_request_access(void *data) {
    /* pause CoPro's GPIO access, signal via shared mailbox register */
    writel(COPRO_PAUSE, mailbox_base + PAUSE_REG);
    return wait_for_completion_timeout(&copro_paused, HZ);
}

static int copro_release_access(void *data) {
    /* resume CoPro */
    writel(COPRO_RESUME, mailbox_base + PAUSE_REG);
    return 0;
}

static const struct aspeed_gpio_copro_ops fan_copro_ops = {
    .request_access = copro_request_access,
    .release_access = copro_release_access,
};

/* Called during CoPro driver probe */
aspeed_gpio_copro_set_ops(&fan_copro_ops, mailbox_data);

/* ──────────────────────────────────────────── */

/* 2. Linux GPIO driver wants to read the fan GPIO */
u16 vreg, dreg;
u8  bit;

/* grabs CoPro lock + gets register location */
ret = aspeed_gpio_copro_grab_gpio(fan_gpio_desc, &vreg, &dreg, &bit);
if (ret)
    return ret;

/* now safe to read — CoPro is paused */
val = readl(gpio_base + vreg) >> bit & 1;

/* 3. Release — CoPro resumes */
aspeed_gpio_copro_release_gpio(fan_gpio_desc);
```

الـ flow ده بيضمن إن Linux والـ CoPro ما بييجوش على نفس الـ register في نفس الوقت.

---

### ملاحظة على الـ Design

الـ header ده صغير جداً (19 سطر) لكنه بيمثّل **design pattern مهم**: بدل ما الـ kernel يحاول يتحكم في الـ CoPro firmware مباشرة (اللي مستحيل لأنه independent processor)، بيعمل **callbacks interface** بيسمح للـ CoPro firmware driver إنه يسجّل نفسه ويتحكم في التنسيق بنفسه. ده مثال نظيف على **Inversion of Control** في kernel driver design.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

> الملف المصدر: `include/linux/gpio/aspeed.h` + `drivers/gpio/gpio-aspeed.c`
> الـ header الصغير بيعرّف الـ public API للـ coprocessor، لكن الـ driver بيعرّف كل الـ structs الداخلية.

---

### الـ Flags والـ Enums والـ Defines — Cheatsheet

#### الـ G7 Control Register Bits (per-GPIO register في AST2700)

| Macro | Bit | الوظيفة |
|---|---|---|
| `GPIO_G7_CTRL_OUT_DATA` | 0 | output value / write latch |
| `GPIO_G7_CTRL_DIR` | 1 | direction: 1=output, 0=input |
| `GPIO_G7_CTRL_IRQ_EN` | 2 | enable interrupt |
| `GPIO_G7_CTRL_IRQ_TYPE0` | 3 | نوع الـ IRQ bit0 |
| `GPIO_G7_CTRL_IRQ_TYPE1` | 4 | نوع الـ IRQ bit1 |
| `GPIO_G7_CTRL_IRQ_TYPE2` | 5 | any-edge detect |
| `GPIO_G7_CTRL_RST_TOLERANCE` | 6 | persist through reset |
| `GPIO_G7_CTRL_DEBOUNCE_SEL1` | 7 | debounce timer select bit1 |
| `GPIO_G7_CTRL_DEBOUNCE_SEL2` | 8 | debounce timer select bit0 |
| `GPIO_G7_CTRL_INPUT_MASK` | 9 | mask input from sampling |
| `GPIO_G7_CTRL_IRQ_STS` | 12 | interrupt status (W1C) |
| `GPIO_G7_CTRL_IN_DATA` | 13 | actual sampled input value |

#### الـ G4 Register Offsets داخل كل bank

| Macro | Offset | الوظيفة |
|---|---|---|
| `GPIO_VAL_VALUE` | +0x00 | input value |
| `GPIO_VAL_DIR` | +0x04 | direction register |
| `GPIO_IRQ_ENABLE` | +0x00 | IRQ enable |
| `GPIO_IRQ_TYPE0` | +0x04 | IRQ type bit0 |
| `GPIO_IRQ_TYPE1` | +0x08 | IRQ type bit1 |
| `GPIO_IRQ_TYPE2` | +0x0c | any-edge |
| `GPIO_IRQ_STATUS` | +0x10 | pending IRQ status |
| `GPIO_DEBOUNCE_SEL1` | +0x00 | debounce sel1 |
| `GPIO_DEBOUNCE_SEL2` | +0x04 | debounce sel2 |
| `GPIO_CMDSRC_0` | +0x00 | command source register 0 |
| `GPIO_CMDSRC_1` | +0x04 | command source register 1 |

#### الـ Command Source Values

| Constant | Value | المعنى |
|---|---|---|
| `GPIO_CMDSRC_ARM` | 0 | الـ ARM CPU هو المتحكم |
| `GPIO_CMDSRC_LPC` | 1 | الـ LPC bus |
| `GPIO_CMDSRC_COLDFIRE` | 2 | الـ ColdFire coprocessor |
| `GPIO_CMDSRC_RESERVED` | 3 | محجوز |

#### الـ `enum aspeed_gpio_reg` — معرّفات سجلات الـ GPIO

| القيمة | يمثل |
|---|---|
| `reg_val` | قيمة الـ GPIO (input/output) |
| `reg_rdata` | read-back من write latch |
| `reg_dir` | اتجاه السطر |
| `reg_irq_enable` | تفعيل الـ interrupt |
| `reg_irq_type0/1/2` | تحديد نوع الـ trigger |
| `reg_irq_status` | حالة الـ interrupt (W1C) |
| `reg_debounce_sel1/2` | اختيار مؤقت الـ debounce |
| `reg_tolerance` | تحمّل الـ reset |
| `reg_cmdsrc0/1` | مصدر الأوامر |

#### IRQ Type Encoding (type2, type1, type0)

| type2 | type1 | type0 | النوع |
|---|---|---|---|
| 0 | 0 | 0 | falling edge |
| 0 | 0 | 1 | rising edge |
| 0 | 1 | 0 | low level |
| 0 | 1 | 1 | high level |
| 1 | 0 | 1 | both edges |

---

### الـ Structs الرئيسية

#### 1. `struct aspeed_gpio_copro_ops` — الـ Public API

الغرض: يعرّف callbacks يستخدمها الـ driver للتنسيق مع الـ coprocessor (ColdFire) لما يحتاج ARM يأخذ control على GPIO مؤقتاً.

```c
/* في include/linux/gpio/aspeed.h */
struct aspeed_gpio_copro_ops {
    /* وقّف الـ coprocessor وخلّي ARM يتحكم */
    int (*request_access)(void *data);
    /* رجّع التحكم للـ coprocessor */
    int (*release_access)(void *data);
};
```

| الحقل | النوع | الدور |
|---|---|---|
| `request_access` | function pointer | pause the coprocessor |
| `release_access` | function pointer | resume the coprocessor |

الاتصال بالـ structs التانية: الـ driver بيحفظ pointer لـ instance واحدة globally في `copro_ops` + `copro_data`.

---

#### 2. `struct aspeed_bank_props` — خصائص الـ Bank

الغرض: يحدد أي bits في bank معين تدعم input وأي bits تدعم output (مش كل GPIOs متماثلة في بعض الـ chips).

```c
struct aspeed_bank_props {
    unsigned int bank;   /* رقم الـ bank (0-based) */
    u32 input;           /* bitmask: الـ bits اللي تقدر تكون input */
    u32 output;          /* bitmask: الـ bits اللي تقدر تكون output */
};
```

المصفوفة بتنتهي بـ sentinel: `{ 0, 0, 0 }` — أي entry فيها `input == 0 && output == 0`.

---

#### 3. `struct aspeed_gpio_config` — إعداد الـ Chip

الغرض: وصف ثابت (const) لكل chip variant — بيتم تحديده من الـ device tree compatible string.

```c
struct aspeed_gpio_config {
    unsigned int nr_gpios;              /* العدد الكلي للـ GPIOs */
    const struct aspeed_bank_props *props; /* pointer لمصفوفة الـ bank props */
    const struct aspeed_gpio_llops *llops; /* vtable للعمليات الـ low-level */
    const int *debounce_timers_array;   /* offsets لـ debounce timer registers */
    int debounce_timers_num;            /* عدد الـ timers */
    bool require_dcache;                /* هل محتاج write-latch cache؟ */
};
```

| `require_dcache` | متى يكون true | السبب |
|---|---|---|
| AST2400/2500/2600 | دايماً | الـ G4 register layout بيحتاج read-modify-write من cache |
| AST2700 | false | كل GPIO له register مستقل، مفيش حاجة للـ cache |

---

#### 4. `struct aspeed_gpio_llops` — الـ Low-Level Operations (vtable)

الغرض: طبقة تجريد بين الـ core driver logic وبين الـ hardware register layout المختلف بين G4 (AST2400/2500/2600) وG7 (AST2700).

```c
struct aspeed_gpio_llops {
    /* اكتب bit واحد في register محدد لـ GPIO محدد */
    void (*reg_bit_set)(struct aspeed_gpio *gpio, unsigned int offset,
                        const enum aspeed_gpio_reg reg, bool val);
    /* اقرأ bit واحد */
    bool (*reg_bit_get)(struct aspeed_gpio *gpio, unsigned int offset,
                        const enum aspeed_gpio_reg reg);
    /* اقرأ register كامل (32-bit) للـ bank */
    int (*reg_bank_get)(struct aspeed_gpio *gpio, unsigned int offset,
                        const enum aspeed_gpio_reg reg);
    /* غيّر command source لـ GPIO معين */
    void (*privilege_ctrl)(struct aspeed_gpio *gpio, unsigned int offset, int owner);
    /* initialize كل GPIOs على ARM ownership */
    void (*privilege_init)(struct aspeed_gpio *gpio);
    /* اطلب تعليق الـ coprocessor قبل الوصول */
    bool (*copro_request)(struct aspeed_gpio *gpio, unsigned int offset);
    /* رجّع التحكم للـ coprocessor */
    void (*copro_release)(struct aspeed_gpio *gpio, unsigned int offset);
};
```

التنفيذات:
- **`aspeed_g4_llops`**: للـ AST2400/2500/2600 — bank-based registers، يدعم coprocessor handshake.
- **`aspeed_g7_llops`**: للـ AST2700 — per-GPIO control register، مفيش coprocessor support (`copro_*` = NULL).

---

#### 5. `struct aspeed_gpio_bank` — تخطيط Register الـ Bank (G4 فقط)

الغرض: بيعرّف الـ memory offsets لكل مجموعة registers في bank واحد من أصل 8 banks.

```c
struct aspeed_gpio_bank {
    uint16_t val_regs;       /* +0: read input; +4: direction */
    uint16_t rdata_reg;      /* read-back من write latch */
    uint16_t irq_regs;       /* +0: enable, +4: type0, +8: type1, +C: type2, +10: status */
    uint16_t debounce_regs;  /* +0: sel1, +4: sel2 */
    uint16_t tolerance_regs; /* reset tolerance */
    uint16_t cmdsrc_regs;    /* +0: src0, +4: src1 */
};
```

> مهم: `val_regs` عنده quirk — لما تقرأ منه بترجع الـ input المقروء من الـ line، مش الـ write latch. عشان كده الـ `rdata_reg` موجود للـ reliable read-back.

---

#### 6. `struct aspeed_gpio` — الـ Instance الرئيسية

الغرض: يمثل controller GPIO واحد — بيتخزن في `platform_device` data.

```c
struct aspeed_gpio {
    struct gpio_chip chip;       /* embedded gpio_chip — الـ Linux GPIO core object */
    struct device *dev;          /* الـ device */
    raw_spinlock_t lock;         /* يحمي كل register access */
    void __iomem *base;          /* base address بعد ioremap */
    int irq;                     /* رقم الـ parent IRQ */
    const struct aspeed_gpio_config *config; /* pointer للـ chip config */

    u8 *offset_timer;            /* [ngpio]: لكل GPIO، رقم الـ debounce timer (0=disabled) */
    unsigned int timer_users[4]; /* عداد المستخدمين لكل timer (index 0 مش مستخدم) */
    struct clk *clk;             /* clock لحساب debounce cycles */

    u32 *dcache;                 /* [banks]: cache للـ write latch (G4 فقط) */
    u8 *cf_copro_bankmap;        /* [ngpio/8]: عداد GPIOs assigned للـ ColdFire لكل byte-group */
};
```

| الحقل | الحجم | متى يُستخدم |
|---|---|---|
| `dcache` | `banks × 4` bytes | G4 فقط، read-modify-write للـ val register |
| `cf_copro_bankmap` | `ngpio/8` bytes | بس لما فيه coprocessor، عداد لكل group من 8 GPIOs |
| `offset_timer` | `ngpio` bytes | debounce tracking — index 0 يعني disabled |
| `timer_users[4]` | 4 × 4 bytes | reference counting للـ timers 1-3 |

---

### مخطط علاقات الـ Structs

```
                ┌─────────────────────────────────────┐
                │         struct aspeed_gpio           │
                │  (main instance, per controller)     │
                │                                      │
                │  .chip  ──────────────────────────► struct gpio_chip
                │  .config ─────────────────────────► struct aspeed_gpio_config
                │  .lock  (raw_spinlock_t)             │
                │  .base  (void __iomem *)             │
                │  .dcache (u32 *)                     │
                │  .offset_timer (u8 *)                │
                │  .cf_copro_bankmap (u8 *)            │
                └─────────────────────────────────────┘
                                │
                    .config points to
                                │
                                ▼
                ┌─────────────────────────────────────┐
                │      struct aspeed_gpio_config       │
                │  (const, one per chip variant)       │
                │                                      │
                │  .props ──────────────────────────► struct aspeed_bank_props[]
                │  .llops ──────────────────────────► struct aspeed_gpio_llops
                │  .nr_gpios                           │
                │  .debounce_timers_array (const int*)│
                │  .require_dcache (bool)              │
                └─────────────────────────────────────┘
                      │                   │
              .props  │           .llops  │
                      ▼                   ▼
        ┌─────────────────────┐  ┌──────────────────────────────┐
        │ aspeed_bank_props[] │  │  struct aspeed_gpio_llops    │
        │ (input/output masks │  │  (vtable: G4 or G7 impl)     │
        │  per bank)          │  │  .reg_bit_set()              │
        │ { bank, in, out }   │  │  .reg_bit_get()              │
        │ ...                 │  │  .reg_bank_get()             │
        │ { 0, 0, 0 } ← end  │  │  .privilege_ctrl()           │
        └─────────────────────┘  │  .copro_request()            │
                                 │  .copro_release()            │
                                 └──────────────────────────────┘

        Global (module-level):
        ┌──────────────────────────────────────────┐
        │  static const aspeed_gpio_copro_ops *copro_ops  │
        │  static void *copro_data                        │
        └──────────────────────────────────────────┘
                        │
                  set by aspeed_gpio_copro_set_ops()
                        │
                        ▼
        ┌──────────────────────────────────┐
        │  struct aspeed_gpio_copro_ops    │
        │  (public API, aspeed.h)          │
        │  .request_access()  ─► pause ColdFire  │
        │  .release_access()  ─► resume ColdFire │
        └──────────────────────────────────┘

        G4 only:
        ┌──────────────────────────────────────────┐
        │  static const aspeed_gpio_bank banks[8]  │
        │  (lookup table: bank index → reg offsets)│
        │  .val_regs, .rdata_reg, .irq_regs, ...  │
        └──────────────────────────────────────────┘
```

---

### مخطط الـ Lifecycle

```
Device Tree (compatible: "aspeed,ast2600-gpio")
        │
        ▼
aspeed_gpio_probe()
  ├── devm_kzalloc → alloc struct aspeed_gpio
  ├── devm_platform_ioremap_resource → gpio->base
  ├── raw_spin_lock_init(&gpio->lock)
  ├── device_get_match_data → gpio->config  (ast2600_config)
  ├── devm_clk_get_enabled → gpio->clk
  ├── setup gpio_chip callbacks (dir_in, dir_out, get, set, ...)
  ├── [if require_dcache]
  │     devm_kcalloc → gpio->dcache
  │     populate from hw (reg_bank_get rdata)
  ├── [if privilege_init] → switch all cmdsrc to ARM
  ├── setup gpio_irq_chip (parent_handler, parents, init_valid_mask)
  ├── devm_kzalloc → gpio->offset_timer
  └── devm_gpiochip_add_data → registered with GPIO core
              │
              ▼
        GPIO chip active
        │
        ├── consumer calls gpiod_get() → aspeed_gpio_request()
        │       └── pinctrl_gpio_request()
        │
        ├── consumer calls gpiod_direction_input/output()
        │       └── aspeed_gpio_dir_in/out()
        │               ├── acquire raw_spinlock_irqsave
        │               ├── [copro_request if bank owned by ColdFire]
        │               ├── reg_bit_set(reg_dir, 0/1)
        │               └── [copro_release]
        │
        ├── IRQ fires → aspeed_gpio_irq_handler()
        │       ├── chained_irq_enter
        │       ├── for each bank: reg_bank_get(reg_irq_status)
        │       ├── for each set bit: generic_handle_domain_irq
        │       └── chained_irq_exit
        │
        └── module unload / device remove
                └── devm cleanup (auto: ioremap, allocs, gpiochip)
```

---

### مخطط Call Flow — GPIO Set مع Coprocessor Arbitration

```
consumer calls: gpiod_set_value(desc, 1)
    │
    ▼
gpio_chip.set() → aspeed_gpio_set(gc, offset, val)
    │
    ├── guard(raw_spinlock_irqsave)(&gpio->lock)
    │
    ├── aspeed_gpio_copro_request(gpio, offset)
    │       │
    │       └── llops->copro_request(gpio, offset)  [G4 only]
    │               │
    │               ├── check cf_copro_bankmap[offset>>3] != 0?
    │               │   (هل الـ bank ده assigned للـ ColdFire؟)
    │               │
    │               ├── copro_ops->request_access(copro_data)
    │               │       └── [coprocessor firmware: pause ColdFire]
    │               │
    │               ├── aspeed_g4_privilege_ctrl(offset, GPIO_CMDSRC_ARM)
    │               │       └── write cmdsrc0/cmdsrc1 registers
    │               │
    │               └── gpio->dcache[bank] = reg_bank_get(reg_rdata)
    │                       └── sync cache with hw state
    │
    ├── __aspeed_gpio_set(gc, offset, val)
    │       │
    │       └── llops->reg_bit_set(gpio, offset, reg_val, val)
    │               │
    │               [G4]: read dcache → modify bit → write dcache + iowrite32
    │               [G7]: ioread32(ctrl_reg) → modify OUT_DATA bit → iowrite32
    │               │
    │               └── llops->reg_bit_get(reg_val)  ← flush write
    │
    └── aspeed_gpio_copro_release(gpio, offset)  [if copro was taken]
            │
            └── llops->copro_release(gpio, offset)
                    ├── aspeed_g4_privilege_ctrl(offset, GPIO_CMDSRC_COLDFIRE)
                    └── copro_ops->release_access(copro_data)
                            └── [coprocessor firmware: resume ColdFire]
```

---

### مخطط Call Flow — Debounce Setup

```
consumer calls: gpiod_set_debounce(desc, usecs)
    │
    ▼
gpio_chip.set_config() → aspeed_gpio_set_config(chip, offset, config)
    │
    ├── param == PIN_CONFIG_INPUT_DEBOUNCE ?
    │
    └── set_debounce(chip, offset, usecs)
            │
            ├── have_debounce(gpio, offset)?  ← فقط input-capable GPIOs
            │
            ├── usecs > 0 → enable_debounce()
            │       │
            │       ├── usecs_to_cycles(gpio, usecs)
            │       │       └── rate = clk_get_rate(gpio->clk)
            │       │           cycles = rate * usecs / 1_000_000
            │       │
            │       ├── guard(raw_spinlock_irqsave)
            │       │
            │       ├── if timer already allocated → unregister_allocated_timer()
            │       │
            │       ├── scan timers 1..3: find matching cycles in hw registers
            │       │
            │       ├── if none found: find timer with timer_users[j] == 0
            │       │       └── iowrite32(cycles, base + debounce_timers[j])
            │       │
            │       ├── register_allocated_timer(gpio, offset, i)
            │       │       ├── offset_timer[offset] = i
            │       │       └── timer_users[i]++
            │       │
            │       └── configure_timer(gpio, offset, i)
            │               └── reg_bit_set(reg_debounce_sel1/2, ...)
            │
            └── usecs == 0 → disable_debounce()
                    ├── unregister_allocated_timer()
                    │       └── timer_users[i]--; offset_timer[offset] = 0
                    └── configure_timer(gpio, offset, 0)
```

---

### مخطط Call Flow — Coprocessor GPIO Grab (Public API)

```
coprocessor driver calls:
aspeed_gpio_copro_grab_gpio(desc, &vreg_offset, &dreg_offset, &bit)
    │
    ├── gpiod_to_chip(desc) → chip
    ├── gpiochip_get_data(chip) → gpio
    ├── gpiod_hwgpio(desc) → offset
    ├── to_bank(offset) → bank
    │
    ├── aspeed_gpio_support_copro(gpio)?
    │       └── checks: copro_request && copro_release && privilege_ctrl && privilege_init
    │
    ├── alloc cf_copro_bankmap if not exists
    │
    ├── guard(raw_spinlock_irqsave)
    │
    ├── cf_copro_bankmap[offset>>3]++
    │
    ├── if count became 1 (first GPIO in this byte-group):
    │       aspeed_gpio_change_cmd_source(offset, GPIO_CMDSRC_COLDFIRE)
    │               └── privilege_ctrl(gpio, offset, COLDFIRE)
    │                       └── write cmdsrc0/cmdsrc1 registers
    │
    ├── *vreg_offset = bank->val_regs
    ├── *dreg_offset = bank->rdata_reg
    └── *bit = GPIO_OFFSET(offset)
```

---

### استراتيجية الـ Locking

#### القفل الوحيد: `gpio->lock` — الـ `raw_spinlock_t`

الـ driver يستخدم **قفل واحد فقط** هو `raw_spinlock_t lock` داخل `struct aspeed_gpio`.

| المحمي بالقفل | السبب |
|---|---|
| كل `reg_bit_set` / `reg_bit_get` operations | منع race على hardware registers |
| `gpio->dcache[]` | الـ G4 write cache مشترك |
| `gpio->offset_timer[]` | debounce timer allocation |
| `gpio->timer_users[]` | reference counting |
| `gpio->cf_copro_bankmap[]` | coprocessor bank tracking |
| تسلسل: `copro_request → hw write → copro_release` | atomicity للـ handshake |

#### ليه `raw_spinlock_t` وليس spinlock عادي؟

الـ `raw_spinlock_t` مفيش عليها `might_sleep()` checks ومتاحة حتى في الـ RT (realtime) kernel عكس `spinlock_t` العادية اللي بتتحول لـ mutex في RT. الـ GPIO driver يتعامل مع IRQ context فمحتاج raw spinlock.

#### ترتيب القفل (Lock Ordering)

مفيش nested locks — القفل الوحيد هو `gpio->lock`. الـ coprocessor callbacks (`copro_ops->request_access`) بتتنادى **من داخل** القفل، يعني:

```
gpio->lock acquired
    └── copro_ops->request_access()   ← لازم تكون atomic/non-sleeping
    └── hw register access
    └── copro_ops->release_access()
gpio->lock released
```

> تحذير: الكود اللي بيسجّل الـ `copro_ops` (باستخدام `aspeed_gpio_copro_set_ops`) لازم يضمن إن الـ callbacks لا تحاول تاخد أي lock تاني عشان متحصلش deadlock.

#### الـ IRQ Masking في الـ Lock

```c
guard(raw_spinlock_irqsave)(&gpio->lock);
```

بيستخدم `irqsave` يعني بيعطّل الـ IRQs على الـ CPU الحالية أثناء حيازة القفل — ده مهم لأن الـ IRQ handler نفسه (`aspeed_gpio_irq_handler`) بيتكلم عالـ hardware registers من غير ما يأخد القفل الرئيسي (بيتعامل مع `irq_status` read-only في الـ chained handler).
## Phase 4: شرح الـ Functions

### ملخص الـ API — Cheatsheet

| Function | Category | Return | Purpose |
|----------|----------|--------|---------|
| `aspeed_gpio_copro_grab_gpio` | Runtime | `int` | يحجز GPIO لـ co-processor ويرجّع offsets وبت الـ register |
| `aspeed_gpio_copro_release_gpio` | Runtime | `int` | يحرر GPIO من سيطرة الـ co-processor |
| `aspeed_gpio_copro_set_ops` | Registration | `int` | يسجّل callbacks الـ co-processor للـ access control |

---

### الـ Struct: `aspeed_gpio_copro_ops`

```c
struct aspeed_gpio_copro_ops {
    int (*request_access)(void *data);
    int (*release_access)(void *data);
};
```

الـ **`aspeed_gpio_copro_ops`** هو interface للـ co-processor (CoPro) اللي بيتحكم في GPIO على SoC الـ ASPEED AST2400/AST2500/AST2600.
الـ SoC دي فيها ARM core رئيسي وكمان microcontroller داخلي (co-processor) ممكن يحتاج exclusive access على GPIO lines معينة.
الـ struct ده بيعرّف callback pair بتسمح للـ GPIO driver يطلب من الـ co-processor يديه access أو يرجّعه.

| Field | النوع | الوصف |
|-------|-------|-------|
| `request_access` | `int (*)(void *)` | بيطلب من الـ CoPro يوقف تحكمه في الـ GPIO — returns 0 on success |
| `release_access` | `int (*)(void *)` | بيعلم الـ CoPro إن الـ ARM رجّع التحكم — returns 0 on success |

---

### Group 1: Registration

#### `aspeed_gpio_copro_set_ops`

```c
int aspeed_gpio_copro_set_ops(const struct aspeed_gpio_copro_ops *ops,
                               void *data);
```

**بيعمل إيه:**
بيسجّل الـ `aspeed_gpio_copro_ops` callbacks على مستوى الـ GPIO driver العام. الـ GPIO driver بيخزّن الـ `ops` و `data` في global/static state الخاصة بيه، وبالتالي الـ function دي لازم تتعمل call واحدة بس في الـ system (من driver الـ CoPro في الـ init path).

**Parameters:**

| Parameter | النوع | الوصف |
|-----------|-------|-------|
| `ops` | `const struct aspeed_gpio_copro_ops *` | pointer للـ ops struct اللي بيعرّف request/release callbacks — ممكن يبقى `NULL` لو حابب تلغي التسجيل |
| `data` | `void *` | opaque pointer بيتمرر كـ argument للـ callbacks — عادةً بيبقى pointer لـ private data الـ CoPro driver |

**Return Value:**
`0` في حالة النجاح. Error code سالب (مثلاً `-EBUSY`) لو في ops متسجلة بالفعل أو في error في الـ driver.

**Key Details:**
- الـ function دي بتعمل لـ **global registration** — مفيش per-GPIO-chip registration هنا.
- لازم تتعمل call قبل أي call لـ `aspeed_gpio_copro_grab_gpio`.
- الـ locking بتتعمل داخل الـ GPIO driver implementation.
- عادةً بيتعملها call من `probe()` الخاصة بـ driver الـ CoPro.

**Who calls it:**
الـ ASPEED co-processor driver (مثلاً `drivers/soc/aspeed/aspeed-lpc-ctrl.c` أو أي firmware interface driver) في الـ `probe()` بتاعه.

---

### Group 2: Runtime — GPIO Ownership Transfer

#### `aspeed_gpio_copro_grab_gpio`

```c
int aspeed_gpio_copro_grab_gpio(struct gpio_desc *desc,
                                 u16 *vreg_offset,
                                 u16 *dreg_offset,
                                 u8  *bit);
```

**بيعمل إيه:**
الـ function دي هي قلب الـ API. بتقوم بـ:
1. استدعاء `ops->request_access(data)` عشان تطلب من الـ CoPro يوقف تحكمه في الـ GPIO bus.
2. حساب وإرجاع الـ register offsets والـ bit position الخاصة بـ GPIO المطلوب، عشان الـ ARM يقدر يعمل direct register access.
3. بعد ما بتخلص، الـ CoPro ما يرجعش تحكمه تلقائياً — لازم تستدعي `aspeed_gpio_copro_release_gpio` بنفسك.

الـ ASPEED GPIO controller بيستخدم register layout منتظم: كل GPIO group فيه value register وdirection register وكل line بتتحدد ببت واحد.

**Parameters:**

| Parameter | النوع | الوصف |
|-----------|-------|-------|
| `desc` | `struct gpio_desc *` | الـ GPIO descriptor المطلوب — بيتجيب من `gpiod_get()` أو ما شابه |
| `vreg_offset` | `u16 *` | output — بيُملأ بـ offset الـ value/data register بالنسبة لـ base address الـ GPIO controller |
| `dreg_offset` | `u16 *` | output — بيُملأ بـ offset الـ direction register |
| `bit` | `u8 *` | output — رقم البت داخل الـ register (0–31 عادةً) |

**Return Value:**
`0` عند النجاح مع ملء الـ output parameters. Error code سالب في حالة:
- `ops->request_access()` فشل (الـ CoPro ما وافقش أو timeout).
- الـ `desc` مش GPIO تابع لـ ASPEED controller.
- ما فيش ops متسجلة (`-EOPNOTSUPP` أو مشابه).

**Key Details:**
- الـ function دي بتـ**implicitly يأخد lock** على الـ GPIO line بشكل منطقي من جهة الـ CoPro — لازم يتعملها `release` بعدين.
- الـ caller بيستخدم الـ offsets دي عشان يعمل `ioremap` أو direct MMIO write، متجنباً الـ GPIO subsystem overhead.
- مش thread-safe بالنسبة لـ concurrent callers على نفس الـ GPIO — الـ caller مسؤول عن أي serialization إضافية.
- الـ `desc` لازم يكون valid ومش released.

**Pseudocode Flow:**

```c
aspeed_gpio_copro_grab_gpio(desc, vreg_offset, dreg_offset, bit):
    // 1. Verify ops are registered
    if (!copro_ops)
        return -EOPNOTSUPP

    // 2. Request CoPro to yield GPIO control
    ret = copro_ops->request_access(copro_data)
    if (ret)
        return ret

    // 3. Translate gpio_desc → register layout
    gpio_chip = gpiod_to_chip(desc)
    gpio_index = gpio_chip_hwgpio(desc)  // hw GPIO number

    // ASPEED layout: groups of 8 GPIOs per register set
    // offset = base + group * stride
    *vreg_offset = calculate_value_reg_offset(gpio_index)
    *dreg_offset = calculate_dir_reg_offset(gpio_index)
    *bit         = gpio_index % 8   // or similar bit extraction

    return 0
```

**Who calls it:**
أي kernel driver محتاج يعمل direct MMIO access على GPIO بدون المرور بـ GPIO subsystem (لأسباب timing أو atomicity)، بعد ما يتأكد إن الـ CoPro مش بيستخدم نفس الـ line. مثال: firmware update drivers، أو IPMI/BMC interface drivers على ASPEED BMC SoCs.

---

#### `aspeed_gpio_copro_release_gpio`

```c
int aspeed_gpio_copro_release_gpio(struct gpio_desc *desc);
```

**بيعمل إيه:**
بيرجّع التحكم في الـ GPIO للـ CoPro عن طريق استدعاء `ops->release_access(data)`. لازم يتعمل call دايماً بعد `aspeed_gpio_copro_grab_gpio` ناجحة، حتى لو الـ MMIO operation فشلت، عشان الـ CoPro ما يفضلش محظور.

**Parameters:**

| Parameter | النوع | الوصف |
|-----------|-------|-------|
| `desc` | `struct gpio_desc *` | نفس الـ GPIO descriptor اللي اتعملت عليه `grab` — بيُستخدم للـ validation إنك بترجّع الصح |

**Return Value:**
`0` عند النجاح. Error code سالب لو:
- الـ `release_access()` callback رجّع error.
- ما فيش ops متسجلة.
- الـ `desc` مش valid أو مش owned.

**Key Details:**
- لازم يتعمل call في كل paths، success وerror على حد سواء — يُفضّل استخدامه في `goto cleanup` pattern.
- مش بيعمل GPIO value reset — بس بيفك الـ ownership من الـ ARM ويرجّعه للـ CoPro.
- الـ CoPro بعد الـ release ممكن يبدأ يتحكم في الـ GPIO line تاني فوراً.

**Pseudocode Flow:**

```c
aspeed_gpio_copro_release_gpio(desc):
    if (!copro_ops)
        return -EOPNOTSUPP

    // Optional: validate desc matches grabbed GPIO

    return copro_ops->release_access(copro_data)
```

**Who calls it:**
نفس الـ caller اللي استدعى `aspeed_gpio_copro_grab_gpio`، مباشرةً بعد انتهاء الـ MMIO operations المطلوبة.

---

### الـ Interaction Pattern الكامل

```
ARM Driver                          ASPEED GPIO Driver          CoPro Driver
    |                                      |                         |
    |--- aspeed_gpio_copro_grab_gpio() --->|                         |
    |                                      |-- request_access() ---->|
    |                                      |<-- 0 (ok) -------------|
    |<-- vreg_offset, dreg_offset, bit ----|                         |
    |                                      |                         |
    |--- direct MMIO write/read ---------->|  (bypassing subsystem)  |
    |                                      |                         |
    |--- aspeed_gpio_copro_release_gpio() ->|                        |
    |                                      |-- release_access() ---->|
    |                                      |<-- 0 (ok) -------------|
    |<-- 0 (ok) ---------------------------|                         |
```

---

### ملاحظات مهمة على الـ ASPEED GPIO Co-Processor Model

الـ ASPEED BMC SoCs (AST2400, AST2500, AST2600) فيها **microcontroller داخلي** (co-processor) بيشتغل على firmware منفصل وبيتحكم في GPIO lines معينة بشكل مستقل عن الـ ARM host. الـ API ده بيحل مشكلة **concurrent GPIO ownership** بين الـ ARM وهذا الـ CoPro.

بدون الـ API ده:
- لو الـ ARM عمل GPIO write وهو والـ CoPro بيكتبوا على نفس الـ register في نفس الوقت → **race condition** وcorrupted GPIO state.

باستخدام الـ API:
- الـ ARM بيطلب permission → بيعمل MMIO → بيرجّع permission.
- الـ CoPro بيوقف أثناء فترة الـ ARM access.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs

الـ ASPEED subsystem (LPC ctrl, P2A ctrl, GPIO, Video) مش بيسجّل entries كتير في debugfs بشكل تلقائي، لكن ممكن تلاقي معلومات مفيدة عبر regmap debugfs:

```bash
# تشوف كل الـ regmap entries المتاحة
ls /sys/kernel/debug/regmap/

# مثلاً لو الـ LPC controller اتعمله probe
ls /sys/kernel/debug/regmap/aspeed-lpc-v2*/
cat /sys/kernel/debug/regmap/aspeed-lpc-v2*/registers
```

**المخرجات المتوقعة:**
```
00000080: 00000500   # HICR5 - LPC FWH cycles enable state
00000084: 00020000   # HICR6 - FWH2AHB mode
00000088: fe000000   # HICR7 - BMC/Host LPC address mapping
0000008c: 00ff0000   # HICR8 - address mask
```

للـ GPIO co-processor:
```bash
ls /sys/kernel/debug/gpio
cat /sys/kernel/debug/gpio
```

---

#### 2. sysfs

```bash
# معلومات عامة عن الـ platform devices
ls /sys/bus/platform/devices/ | grep aspeed

# LPC ctrl device
ls /sys/bus/platform/devices/aspeed-lpc-ctrl*/
cat /sys/bus/platform/devices/aspeed-lpc-ctrl*/driver/module/parameters/* 2>/dev/null

# P2A ctrl device
ls /sys/bus/platform/devices/aspeed-p2a-ctrl*/

# GPIO
ls /sys/class/gpio/
cat /sys/class/gpio/gpiochip*/label
cat /sys/class/gpio/gpiochip*/ngpio
cat /sys/class/gpio/gpiochip*/base

# الـ clocks المستخدمة
cat /sys/kernel/debug/clk/clk_summary | grep -i aspeed

# الـ misc devices المسجّلة
ls -la /dev/aspeed-*
```

---

#### 3. ftrace — Tracepoints وأحداث مهمة

```bash
# تفعيل الـ function tracer لكل الـ aspeed functions
cd /sys/kernel/debug/tracing
echo function > current_tracer
echo 'aspeed_*' > set_ftrace_filter
echo 1 > tracing_on
sleep 5
echo 0 > tracing_on
cat trace | head -100

# تتبع الـ ioctl calls على الـ LPC ctrl
echo 'aspeed_lpc_ctrl_ioctl' > set_ftrace_filter
echo function > current_tracer
echo 1 > tracing_on

# تتبع الـ P2A bridge enable/disable
echo 'aspeed_p2a_enable_bridge aspeed_p2a_disable_bridge' > set_ftrace_filter

# تتبع الـ regmap writes (مهم لفهم register access)
echo 'regmap_write regmap_update_bits' >> set_ftrace_filter

# تتبع events الـ GPIO
echo 1 > events/gpio/enable

# تتبع الـ DMA events (للـ HACE crypto)
echo 1 > events/dma/enable

# تفعيل كل الـ syscon/regmap events
ls events/regmap/
echo 1 > events/regmap/enable
cat trace
```

---

#### 4. printk و Dynamic Debug

```bash
# تفعيل كل الـ dev_dbg messages في drivers/soc/aspeed/
echo 'file drivers/soc/aspeed/aspeed-lpc-ctrl.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/soc/aspeed/aspeed-p2a-ctrl.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/soc/aspeed/aspeed-lpc-snoop.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل كل الـ aspeed GPIO debug
echo 'file drivers/gpio/gpio-aspeed.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل بالـ module name
echo 'module aspeed_lpc_ctrl +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 'module aspeed_p2a_ctrl +pflmt' > /sys/kernel/debug/dynamic_debug/control

# مشاهدة فورية
dmesg -w | grep -i aspeed

# أو
journalctl -k -f | grep -i aspeed
```

الـ `+pflmt` بتفعّل: **print** + **function name** + **line number** + **module** + **timestamp**.

---

#### 5. Kernel Config Options للـ Debugging

| CONFIG Option | الوصف |
|---|---|
| `CONFIG_ASPEED_LPC_CTRL` | تفعيل LPC ctrl driver أصلاً |
| `CONFIG_ASPEED_P2A_CTRL` | تفعيل P2A bridge driver |
| `CONFIG_GPIO_ASPEED` | تفعيل ASPEED GPIO driver |
| `CONFIG_DEBUG_GPIO` | تفعيل GPIO debugging messages |
| `CONFIG_REGMAP_DEBUGFS` | تعرض register map في debugfs |
| `CONFIG_DEBUG_FS` | لازم يكون مفعّل علشان debugfs |
| `CONFIG_DYNAMIC_DEBUG` | تفعيل dynamic debug infrastructure |
| `CONFIG_TRACING` | تفعيل ftrace |
| `CONFIG_MFD_SYSCON` | الـ syscon/regmap backend |
| `CONFIG_ASPEED_BMC_MISC` | بعض الـ socinfo entries |
| `CONFIG_EDAC_ASPEED` | DRAM error detection على ASPEED |
| `CONFIG_ASPEED_XDMA_ENGINE` | لو محتاج debug الـ XDMA/P2A DMA |
| `CONFIG_DEBUG_MUTEXES` | debugging للـ mutex في P2A tracking |
| `CONFIG_PROVE_LOCKING` | لاكتشاف deadlocks في الـ mutex |

```bash
# تتحقق من الـ config الحالي
zcat /proc/config.gz | grep -i aspeed
zcat /proc/config.gz | grep CONFIG_REGMAP_DEBUGFS
```

---

#### 6. devlink وأدوات خاصة بالـ subsystem

الـ ASPEED LPC/P2A/GPIO drivers بيستخدموا **miscdevice** interface — مش devlink. الأدوات المناسبة:

```bash
# التحقق من الـ misc devices المسجّلة
cat /proc/misc | grep aspeed
# مثال على المخرجات:
# 122 aspeed-lpc-ctrl
# 123 aspeed-p2a-ctrl

# استخدام الـ ioctls مباشرة عبر أداة بسيطة
# للـ LPC ctrl — قراءة حجم الـ memory window:
python3 - <<'EOF'
import fcntl, struct, os

# ASPEED_LPC_CTRL_IOCTL_GET_SIZE = _IOWR(0xb2, 0x00, ...)
# struct aspeed_lpc_ctrl_mapping: u8 window_type, u8 window_id, u16 flags, u32 addr, u32 offset, u32 size
IOCTL_GET_SIZE = 0xc010b200
mapping = struct.pack('BBHIIi', 2, 0, 0, 0, 0, 0)  # WINDOW_MEMORY=2, id=0
fd = os.open('/dev/aspeed-lpc-ctrl', os.O_RDWR)
result = fcntl.ioctl(fd, IOCTL_GET_SIZE, bytearray(mapping))
wtype, wid, flags, addr, offset, size = struct.unpack('BBHIIi', result)
print(f'Memory window size: 0x{size:08x} ({size} bytes)')
os.close(fd)
EOF

# للـ P2A ctrl — قراءة الـ memory config:
python3 - <<'EOF'
import fcntl, struct, os

# ASPEED_P2A_CTRL_IOCTL_GET_MEMORY_CONFIG = _IOWR(0xb3, 0x01, ...)
IOCTL_GET_MEM = 0xc010b301
mapping = struct.pack('QII', 0, 0, 0)  # addr=u64, length=u32, flags=u32
fd = os.open('/dev/aspeed-p2a-ctrl', os.O_RDWR)
result = fcntl.ioctl(fd, IOCTL_GET_MEM, bytearray(mapping))
addr, length, flags = struct.unpack('QII', result)
print(f'P2A mem base: 0x{addr:016x}, size: 0x{length:08x}')
os.close(fd)
EOF

# الـ LPC snoop — قراءة POST codes من الـ BIOS
dd if=/dev/aspeed-lpc-snoop0 bs=1 count=32 2>/dev/null | xxd
# كل byte هو POST code من الـ host
```

---

#### 7. رسائل الخطأ الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `aspeed-lpc-ctrl: Couldn't get regmap` | الـ parent node مش syscon أو مش موجود | تحقق من الـ DT: الـ node لازم يكون child لـ `aspeed,ast2600-lpc-v2` |
| `aspeed-lpc-ctrl: unsupported LPC device binding` | الـ parent compatible مش من القائمة المدعومة | الـ compatible المقبولة: `aspeed,ast2400/2500/2600-lpc-v2` |
| `Reserved memory size must be a power of 2` | حجم الـ `memory-region` في DT غلط | لازم يكون 64K، 128K، 256K، ... إلخ |
| `Reserved memory must be naturally aligned` | الـ base address مش متوافق مع الـ size | مثلاً size=256K → address لازم يكون multiple of 256K |
| `aspeed-lpc-ctrl: couldn't find scu` | الـ AST2600 محتاج `aspeed,ast2600-scu` syscon | تأكد من وجود الـ SCU node في DT |
| `aspeed-lpc-ctrl: couldn't enable clock` | فشل في تفعيل الـ LPC clock | تحقق من `ASPEED_CLK_GATE_LCLK` في SCU |
| `aspeed-p2a-ctrl: Couldn't get regmap` | نفس مشكلة الـ LPC — parent syscon مفقود | الـ P2A محتاج parent بـ SCU regmap |
| `aspeed_gpio_copro_grab_gpio: failed` | الـ co-processor GPIO request فشل | تحقق من أن الـ GPIO مش مستخدم من kernel |
| `Didn't find host pnor flash node` | مش خطأ — الـ flash optional في DT | لو محتاج flash mapping، أضف `flash` phandle في DT |
| `Didn't find reserved memory` | الـ memory-region مش محدد في DT | أضف `memory-region` property مع reserved-memory node |

---

#### 8. أماكن استراتيجية لـ dump_stack() وـ WARN_ON()

```c
/* في aspeed_lpc_ctrl_ioctl() — تحقق من الـ map parameters */
static long aspeed_lpc_ctrl_ioctl(...)
{
    /* استراتيجي: قبل الـ regmap_write على HICR7/HICR8 */
    WARN_ON(!IS_ALIGNED(map.size, 0x10000));  /* size لازم تكون aligned لـ 64K */
    WARN_ON(map.addr & 0xffff);               /* low 16 bits لازم تكون zero */
}

/* في aspeed_p2a_enable_bridge() */
static void aspeed_p2a_enable_bridge(struct aspeed_p2a_ctrl *p2a_ctrl)
{
    /* WARN لو bridge اتفعّل من غير ما يكون فيه أي reader */
    WARN_ON(p2a_ctrl->readers == 0 &&
            !memchr_inv(p2a_ctrl->readerwriters,
                        0, sizeof(p2a_ctrl->readerwriters)));
}

/* في aspeed_p2a_release() — تحقق من الـ reference counting */
static int aspeed_p2a_release(...)
{
    /* WARN لو الـ readers count راح تحت الصفر */
    WARN_ON((int)(priv->parent->readers - priv->read) < 0);
}

/* في aspeed_gpio_copro_grab_gpio() */
int aspeed_gpio_copro_grab_gpio(struct gpio_desc *desc, ...)
{
    /* dump_stack لو طُلب GPIO مش موجود */
    if (WARN_ON(!desc)) {
        dump_stack();
        return -EINVAL;
    }
}
```

---

### Hardware Level

#### 1. التحقق من توافق حالة الـ Hardware مع الـ Kernel

```bash
# تحقق أن الـ LPC bridge مفعّل في الـ hardware
# HICR5 offset 0x80 من الـ LPC base — bit 10 (ENFWH) وbit 8 (ENL2H)
# قرأ الـ regmap عبر debugfs
cat /sys/kernel/debug/regmap/*/registers | awk '/^00000080:/ {print "HICR5:", $2}'

# تحقق من P2A bridge state — SCU180 bit 1
# الـ SCU base على ast2600 هو 0x1E6E2000
devmem2 0x1E6E2180 w  # SCU180

# تحقق من SCU2C — P2A region permissions
devmem2 0x1E6E202C w  # SCU2C — bits 22-25 تتحكم في الـ regions

# لو bit 1 في SCU180 = 1 → P2A bridge enabled
# لو bit 22 في SCU2C = 0 → FLASH region read-write
```

**مقارنة حالة الـ kernel مع الـ hardware:**

```bash
# الـ kernel يخبرك بعدد الـ readers
# مش في sysfs مباشرة، لكن ممكن تستنتجها من /proc/misc وـ /proc/$(pid)/fd
ls -la /proc/*/fd 2>/dev/null | grep aspeed-p2a-ctrl
```

---

#### 2. Register Dump Techniques

```bash
# تثبيت devmem2
apt-get install devmem2  # أو بناءه من المصدر

# AST2600 LPC Controller base: 0x1E789000
# قراءة الـ LPC HICR registers
devmem2 0x1E789080 w   # HICR5  — LPC FWH enable
devmem2 0x1E789084 w   # HICR6  — FWH2AHB mode
devmem2 0x1E789088 w   # HICR7  — Address mapping MSBs
devmem2 0x1E78908C w   # HICR8  — Address mask
devmem2 0x1E789090 w   # SNPWADR — Snoop address register
devmem2 0x1E789100 w   # HICRB  — extended control

# AST2500 LPC Controller base: 0x1E789000 (نفس العنوان)
# AST2400 LPC Controller base: 0x1E789000

# SCU registers (ast2600: 0x1E6E2000)
devmem2 0x1E6E202C w   # SCU2C  — Misc control (P2A region bits)
devmem2 0x1E6E2180 w   # SCU180 — PCIe config (P2A bridge enable)
devmem2 0x1E6E20D8 w   # SCU0D8 — Debug control (FWH2AHB enable)

# GPIO controller base: 0x1E780000
devmem2 0x1E780000 w   # GPIO A/B/C/D data
devmem2 0x1E780004 w   # GPIO A/B/C/D direction

# قراءة range كاملة من الـ LPC registers
/sbin/io -r -4 0x1E789080  # بديل لـ devmem2 على بعض الـ BMC distros

# استخدام /dev/mem مباشرة لو devmem2 مش موجود
python3 - <<'EOF'
import mmap, struct, os

def read_reg(base, offset):
    fd = os.open('/dev/mem', os.O_RDONLY | os.O_SYNC)
    m = mmap.mmap(fd, 4096, mmap.MAP_PRIVATE, mmap.PROT_READ,
                  offset=base & ~0xFFF)
    val = struct.unpack('<I', m[(base & 0xFFF) + offset:
                                 (base & 0xFFF) + offset + 4])[0]
    m.close()
    os.close(fd)
    return val

LPC_BASE = 0x1E789000
print(f'HICR5:  0x{read_reg(LPC_BASE, 0x80):08x}')
print(f'HICR6:  0x{read_reg(LPC_BASE, 0x84):08x}')
print(f'HICR7:  0x{read_reg(LPC_BASE, 0x88):08x}')
print(f'HICR8:  0x{read_reg(LPC_BASE, 0x8c):08x}')
EOF
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

**الـ LPC Bus (Host ↔ BMC):**

| Signal | الاستخدام |
|---|---|
| `LCLK` | LPC clock — 33 MHz عادةً، تحقق من الـ frequency |
| `LFRAME#` | بداية الـ LPC transaction — active low |
| `LAD[3:0]` | البيانات والعناوين — multiplexed |
| `LRESET#` | LPC reset |
| `SERIRQ` | Serial IRQ للـ interrupts |

```
Trigger: LFRAME# falling edge
Capture: LAD[3:0] × 8 cycles minimum
Protocol decoder: LPC (متوفر في PulseView/Sigrok)
Sample rate: >= 100 MHz (لـ 33 MHz LPC)
```

**الـ PCIe (للـ P2A bridge على ast2400/2500):**
- الـ P2A bridge بيستخدم الـ VGA BAR في الـ PCIe — تقدر تتحقق منه عبر `lspci -vvv` على الـ host

**GPIO Debugging:**
```
Trigger: أي GPIO edge على الـ pin المطلوب
Capture: 1 MHz sample rate كافي لـ GPIO
ابحث عن glitches في الـ signal لو الـ co-processor والـ kernel بيتنافسوا على نفس الـ GPIO
```

---

#### 4. Hardware Issues الشائعة وأنماطها في الـ Kernel Log

| المشكلة | نمط في الـ Kernel Log | التشخيص |
|---|---|---|
| LPC clock مش مفعّل | `couldn't enable clock` + `clk_prepare_enable failed` | تحقق من `ASPEED_CLK_GATE_LCLK` في SCU — bit 8 في SCU08 |
| P2A bridge مش بيتفعّل | لا يوجد log — الـ bridge يبدو enabled لكن الـ host يقرأ 0xFF | تحقق من SCU180 bit 1 بـ devmem2 |
| Reserved memory مش naturally aligned | `Reserved memory must be naturally aligned` | الـ BMC DRAM layout محتاج تعديل في DT |
| Host يكتب على wrong address | الـ host crash أو unexpected behavior | تحقق من HICR7/HICR8 — الـ address translation غلط |
| GPIO contention (kernel vs co-processor) | `aspeed_gpio_copro_grab_gpio: -EBUSY` | الـ GPIO مستخدم من Linux — حرّره أو اطلبه للـ co-processor |
| LPC snoop FIFO overflow | `kfifo_put failed` في snoop driver | الـ userspace مش بيقرأ بسرعة كافية من `/dev/aspeed-lpc-snoop0` |

---

#### 5. Device Tree Debugging

```bash
# التحقق من الـ DT المحمّل حالياً
dtc -I fs /proc/device-tree > /tmp/current.dts 2>/dev/null
grep -A 20 'lpc-ctrl' /tmp/current.dts

# التحقق من وجود الـ compatible strings الصحيحة
grep -r 'aspeed,ast' /proc/device-tree/ 2>/dev/null | head -20

# التحقق من الـ memory-region
cat /proc/device-tree/reserved-memory/*/reg | xxd
cat /proc/device-tree/reserved-memory/*/compatible 2>/dev/null

# التحقق من الـ LPC ctrl node
ls /proc/device-tree/ahb/lpc@*/
cat /proc/device-tree/ahb/lpc@*/compatible
cat /proc/device-tree/ahb/lpc@1e789000/lpc-ctrl@80/compatible

# مثال DT صحيح للـ ast2600 LPC ctrl
cat << 'DTEOF'
/* DT snippet للـ AST2600 */
lpc: lpc@1e789000 {
    compatible = "aspeed,ast2600-lpc-v2";
    reg = <0x1e789000 0x1000>;
    #address-cells = <1>;
    #size-cells = <1>;
    ranges = <0x0 0x1e789000 0x1000>;

    lpc_ctrl: lpc-ctrl@80 {
        compatible = "aspeed,ast2600-lpc-ctrl";
        reg = <0x80 0x80>;
        clocks = <&syscon ASPEED_CLK_GATE_LCLK>;
        memory-region = <&flash_memory>;
        flash = <&spi>;
    };
};

reserved-memory {
    #address-cells = <1>;
    #size-cells = <1>;
    ranges;

    flash_memory: region@98000000 {
        no-map;
        reg = <0x98000000 0x04000000>; /* 64MB, power-of-2, naturally aligned */
    };
};
DTEOF

# تحقق أن الـ driver اتعمله probe بنجاح
dmesg | grep -E '(aspeed-lpc-ctrl|aspeed-p2a-ctrl|aspeed-gpio)' | head -20

# لو الـ probe فشل، هتلاقي
# [    2.345678] aspeed-lpc-ctrl aspeed-lpc-ctrl: Couldn't get regmap
# أو
# [    2.345678] aspeed-lpc-ctrl: probe with driver aspeed-lpc-ctrl failed with error -19
```

**تحقق سريع من الـ DT vs Hardware:**

```bash
# عنوان reg في DT لازم يتطابق مع عنوان الـ register في الـ chip datasheet
# مثلاً lpc-ctrl@80 يعني offset 0x80 من base الـ LPC = 0x1E789080 = HICR5

# تحقق أن الـ clock handle صح
cat /proc/device-tree/ahb/lpc@1e789000/lpc-ctrl@80/clocks | xxd
# هتلاقي phandle + clock-id (ASPEED_CLK_GATE_LCLK = 8)
```

---

### Practical Commands — Summary

#### جاهز للنسخ

```bash
# === اكتشاف شامل للـ ASPEED subsystem ===

# 1. إيه الـ drivers اللي اتحمّلت؟
lsmod | grep aspeed
dmesg | grep -i aspeed | grep -v '\[' | head -30

# 2. الـ misc devices الموجودة
cat /proc/misc | grep aspeed

# 3. حالة الـ clocks
cat /sys/kernel/debug/clk/clk_summary 2>/dev/null | grep -E '(lclk|lpc|aspeed)'

# 4. الـ regmap registers
for d in /sys/kernel/debug/regmap/*/; do
    echo "=== $d ==="; cat "$d/registers" 2>/dev/null | head -20
done

# 5. تفعيل كل الـ debug logs لـ aspeed
for f in drivers/soc/aspeed drivers/gpio/gpio-aspeed.c; do
    echo "file $f +p" > /sys/kernel/debug/dynamic_debug/control 2>/dev/null
done
dmesg -C; dmesg -w &

# 6. قراءة مسجّلة من LPC registers مباشرة (ast2600)
for reg in 80 84 88 8c 90 94 100; do
    printf "0x1E7890%s = " $reg
    devmem2 "0x1E7890$reg" w 2>/dev/null | grep -o '0x[0-9A-Fa-f]*$'
done

# 7. قراءة P2A/SCU state
echo "SCU2C (P2A regions):"
devmem2 0x1E6E202C w 2>/dev/null
echo "SCU180 (P2A bridge enable):"
devmem2 0x1E6E2180 w 2>/dev/null

# 8. POST codes من الـ BIOS عبر LPC snoop
timeout 10 dd if=/dev/aspeed-lpc-snoop0 bs=1 2>/dev/null | xxd

# 9. ftrace snapshot سريع
echo function > /sys/kernel/debug/tracing/current_tracer
echo 'aspeed_*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
sleep 3
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace | grep aspeed

# 10. تنظيف الـ ftrace
echo nop > /sys/kernel/debug/tracing/current_tracer
echo > /sys/kernel/debug/tracing/set_ftrace_filter
```

#### تفسير المخرجات

```
# مثال على HICR5 = 0x00000500
# bit 10 (0x400) = ENFWH = 1 → FWH cycles مفعّلة ✓
# bit 8  (0x100) = ENL2H = 1 → L2H cycles مفعّلة ✓

# مثال على HICR7 = 0xfe000000 وHICR8 = 0x00ff0000
# HICR7 top 16 bits = 0xfe00 → BMC address MSBs = 0xfe000000
# HICR7 low 16 bits = 0x0000 → Host LPC address MSBs = 0x00000000
# HICR8 top 16 bits = 0x00ff → mask → bits 23:16 محمية
# → الـ host بيقرأ من LPC 0xFF000000 → BMC 0xFE000000

# مثال على SCU2C = 0x07800000
# bit 25 (0x02000000) = SCU2C_DRAM  → DRAM region write-protected ✓
# bit 24 (0x01000000) = SCU2C_SPI   → SPI region write-protected ✓
# bit 23 (0x00800000) = SCU2C_SOC   → SOC region write-protected ✓
# bit 22 (0x00400000) = SCU2C_FLASH → FLASH region write-protected ✓
# = كل الـ regions protected (الحالة الافتراضية بعد aspeed_p2a_disable_all)
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: OpenBMC على AST2600 — الـ coprocessor بيتعلق عند الـ GPIO handshake

#### العنوان
**GPIO co-processor access race على server BMC مبني على AST2600**

#### السياق
شركة بتبني server BMC مبني على **AST2600** باستخدام **OpenBMC**. الـ BMC بيتحكم في GPIOs خاصة بـ power sequencing للـ host processor. في نفس الوقت، في **co-processor** (ARM Cortex-M) داخل الـ AST2600 بيشتغل على firmware منفصل ومحتاج يتحكم في بعض الـ GPIO lines اللي بيتشاركها مع الـ Linux kernel.

#### المشكلة
بعد upgrade الـ OpenBMC image، الـ BMC بيـ hang عند boot في بعض الأحيان. الـ symptom هو إن الـ IPMI interface مش بيجاوب، والـ serial console بتظهر:

```
[  12.345678] aspeed-gpio 1e780000.gpio: copro access timeout waiting for release
[  12.345679] BUG: scheduling while atomic: kworker/0:1/42/0x00000002
```

#### التحليل
الـ `aspeed_gpio_copro_set_ops` بيسجّل الـ `ops` اللي بيستخدمها الـ GPIO driver عشان يطلب access من الـ co-processor أو يرجعه.

```c
/* aspeed.h — the ops struct the copro driver must implement */
struct aspeed_gpio_copro_ops {
    int (*request_access)(void *data);   /* called before touching shared GPIO */
    int (*release_access)(void *data);   /* called after finishing */
};

int aspeed_gpio_copro_set_ops(const struct aspeed_gpio_copro_ops *ops, void *data);
```

المشكلة إن الـ `request_access` callback بتاع الـ co-processor driver الجديد بيعمل `msleep()` جوا context اتعمل فيه `spin_lock_irqsave`، ده بيخلي kernel يـ panic بسبب sleep in atomic context.

الـ flow اللي بيحصل:

```
aspeed_gpio_set()
  └─► spin_lock_irqsave(&gpio->lock, flags)
        └─► copro_ops->request_access(data)   ← هنا بيعمل msleep() ← WRONG!
```

الـ `vreg_offset` و `dreg_offset` المرتجعين من `aspeed_gpio_copro_grab_gpio` بيحتاجوا إن الـ lock يكون ماسكه الـ co-processor driver طول فترة الـ access.

#### الحل
لازم الـ `request_access` callback ما يعملش أي blocking operation. الصح إنه يستخدم busy-wait أو semaphore صح:

```c
/* co-processor driver — WRONG implementation */
static int copro_request_access(void *data)
{
    struct copro_data *d = data;
    msleep(1);  /* BAD: sleep in atomic context */
    return 0;
}

/* co-processor driver — CORRECT implementation */
static int copro_request_access(void *data)
{
    struct copro_data *d = data;
    /* Signal co-processor via mailbox register and spin-wait */
    writel(COPRO_PAUSE_REQ, d->base + COPRO_CTRL_REG);
    /* Busy-wait is acceptable here — atomic context */
    return readl_poll_timeout_atomic(d->base + COPRO_STATUS_REG,
                                     val, val & COPRO_PAUSED,
                                     1, 500);  /* 1us interval, 500us timeout */
}
```

كمان لازم تتأكد إن الـ `aspeed_gpio_copro_grab_gpio` اتسمّاله بشكل صح:

```bash
# تحقق إن الـ copro ops اتسجلت
cat /sys/kernel/debug/gpio | grep aspeed
dmesg | grep "aspeed-gpio"
```

#### الدرس المستفاد
الـ callbacks المسجّلة عبر `aspeed_gpio_copro_set_ops` بتتنفذ جوا spinlock context. أي `request_access` أو `release_access` implementation لازم تكون non-blocking تماماً — ممنوع `msleep`, `wait_event`, أو أي blocking primitive.

---

### السيناريو الثاني: AST2500 BMC — LPC host interface فاشل بسبب GPIO ownership conflict

#### العنوان
**GPIO ownership conflict بين LPC driver والـ co-processor على AST2500**

#### السياق
منتج **IPMI BMC** مبني على **AST2500** بيستخدم بعض الـ GPIOs عشان يتحكم في الـ LPC bus signals للـ host. في نفس الوقت، الـ firmware بتاع الـ co-processor (SCI engine) محتاج access على نفس الـ GPIO group.

#### المشكلة
بعد تشغيل الـ system، الـ IPMI over LPC بيفشل بشكل متقطع. الـ host بيشتكي إن الـ BMC ما بيرد على IPMI requests. الـ logs بتظهر:

```
[  45.112233] aspeed-gpio: copro_grab_gpio failed for GPIO_B3, ret=-16
[  45.112240] aspeed-lpc-ctrl: failed to assert LPC_SCI line
```

الـ error code `-16` هو `EBUSY`.

#### التحليل
الدالة `aspeed_gpio_copro_grab_gpio` بترجع الـ register offsets والـ bit position اللي الـ co-processor محتاجها عشان يتحكم في الـ GPIO مباشرة:

```c
int aspeed_gpio_copro_grab_gpio(struct gpio_desc *desc,
                                u16 *vreg_offset,   /* value register offset */
                                u16 *dreg_offset,   /* direction register offset */
                                u8  *bit);          /* bit position 0-31 */
```

المشكلة إن الـ LPC driver بيعمل `gpio_request` على نفس الـ GPIO اللي الـ co-processor driver عمله `aspeed_gpio_copro_grab_gpio` عليه. الـ GPIO subsystem بيشوف الـ GPIO اتـ grab من الـ copro فبيرفض الـ request بـ `EBUSY`.

```
LPC driver                        CoPro driver
    │                                  │
    ▼                                  ▼
gpio_request(GPIO_B3)     aspeed_gpio_copro_grab_gpio(GPIO_B3, ...)
    │                                  │
    └──────► EBUSY ◄───────────────────┘
             (conflict!)
```

#### الحل
الـ ownership model واضح: الـ co-processor هو اللي بيتحكم في الـ GPIO مباشرة عبر register offsets. الـ Linux driver المفروض يستخدم `aspeed_gpio_copro_release_gpio` أول ما يخلص، وما يعملش `gpio_request` على نفس الـ GPIO في نفس الوقت.

```c
/* في الـ LPC driver — الحل الصح */
static int lpc_sci_assert(struct lpc_data *lpc)
{
    int ret;

    /* Release copro ownership first if needed */
    ret = aspeed_gpio_copro_release_gpio(lpc->sci_gpio);
    if (ret && ret != -ENOENT)
        return ret;

    /* Now safe to use standard GPIO API */
    gpiod_set_value(lpc->sci_gpio, 1);
    return 0;
}
```

وفي الـ Device Tree لازم يتضح مين المالك:

```dts
/* ast2500.dts */
&gpio {
    /* Mark GPIO_B3 as reserved for co-processor */
    gpio-reserved-ranges = <11 1>;  /* GPIO_B3 = bank B bit 3 = index 11 */
};
```

#### الدرس المستفاد
الـ `aspeed_gpio_copro_grab_gpio` و الـ standard `gpio_request` API متعارضين على نفس الـ GPIO. لازم تحدد ownership بشكل واضح في الـ DT وما تخلي اتنين drivers يتحاربوا على نفس الـ line.

---

### السيناريو الثالث: AST2600 OpenBMC — فشل في تسجيل الـ copro ops بسبب module load order

#### العنوان
**Module load order race — الـ GPIO copro ops مش متسجّلة وقت أول استخدام**

#### السياق
في **OpenBMC** على **AST2600**، في kernel module منفصل مسؤول عن تسجيل الـ co-processor ops عبر `aspeed_gpio_copro_set_ops`. ده الـ module بيخدم الـ iKVM و power control features. المشكلة ظهرت بعد إضافة feature جديدة بيحتاج الـ GPIO co-processor في مرحلة early boot.

#### المشكلة
الـ system بيـ boot تمام في معظم الأوقات، لكن بعد إضافة feature جديدة في الـ init sequence، الـ system بيـ crash بنسبة ~30% من الوقت:

```
[   3.221100] aspeed-gpio: copro ops not set, cannot grab GPIO_D5
[   3.221105] aspeed-pwm: failed to initialize fan control
[   3.221110] Unable to handle kernel NULL pointer dereference
```

#### التحليل
الـ `aspeed_gpio_copro_set_ops` بيعمل register للـ ops pointer في الـ GPIO driver:

```c
int aspeed_gpio_copro_set_ops(const struct aspeed_gpio_copro_ops *ops, void *data);
```

لما الـ co-processor module ما يكونش loadedبعد، الـ ops pointer بيكون NULL. أي محاولة لـ `aspeed_gpio_copro_grab_gpio` بتفشل لأن الـ driver بيعمل check على الـ ops قبل ما يكمل.

الـ race بيحصل هكذا:

```
Timeline A (crash):
t=0ms   aspeed-gpio module loads
t=1ms   aspeed-pwm probes, calls aspeed_gpio_copro_grab_gpio()
t=2ms   aspeed-copro module loads, calls aspeed_gpio_copro_set_ops()
        ← TOO LATE, pwm already failed

Timeline B (success):
t=0ms   aspeed-gpio module loads
t=1ms   aspeed-copro module loads, calls aspeed_gpio_copro_set_ops()
t=2ms   aspeed-pwm probes, calls aspeed_gpio_copro_grab_gpio()
        ← OK
```

#### الحل
الحل الصح هو استخدام deferred probe mechanism بدل ما تـ hard-code الـ init order:

```c
/* في الـ aspeed-pwm driver */
static int aspeed_pwm_probe(struct platform_device *pdev)
{
    int ret;

    ret = aspeed_gpio_copro_grab_gpio(fan_gpio, &vreg, &dreg, &bit);
    if (ret == -EPROBE_DEFER) {
        /* Copro ops not ready yet — try again later */
        return -EPROBE_DEFER;
    }
    if (ret) {
        dev_err(&pdev->dev, "failed to grab fan GPIO: %d\n", ret);
        return ret;
    }
    /* ... rest of probe ... */
}
```

وفي الـ Makefile أو `modules-load.d`:

```bash
# /etc/modules-load.d/aspeed-bmc.conf
# Load order matters — copro must come first
aspeed-gpio
aspeed-gpio-copro
aspeed-pwm
```

أو عبر DT `depends-on` equivalent في الـ driver:

```dts
&pwm_fan {
    /* Ensure co-processor is available */
    aspeed,copro = <&copro_node>;
    status = "okay";
};
```

#### الدرس المستفاد
الـ `aspeed_gpio_copro_set_ops` لازم تتعمل قبل أي driver يحاول `aspeed_gpio_copro_grab_gpio`. استخدم `-EPROBE_DEFER` pattern عشان تتجنب الـ race condition بين modules.

---

### السيناريو الرابع: AST2600 — IPMI power control بيكتب على GPIO غلط بسبب خطأ في الـ vreg_offset

#### العنوان
**خطأ في تفسير vreg_offset بيخلي الـ co-processor يكتب على GPIO group غلط**

#### السياق
شركة بتبني **1U server** وعملت custom firmware للـ co-processor داخل الـ AST2600 عشان يتحكم في الـ power rail sequencing مباشرة بدون تدخل الـ Linux kernel. الـ Linux بيستخدم `aspeed_gpio_copro_grab_gpio` عشان يدي الـ co-processor الـ register info اللي محتاجها.

#### المشكلة
بعد تشغيل الـ board، لقوا إن الـ 12V rail بيتشغّل في وقت غلط، والـ server بيـ shutdown فجأة. الـ host processor بيشوف power failure غير متوقعة.

#### التحليل
الـ API بيرجع ثلاث قيم:

```c
int aspeed_gpio_copro_grab_gpio(struct gpio_desc *desc,
                                u16 *vreg_offset,  /* offset من GPIO base للـ value reg */
                                u16 *dreg_offset,  /* offset للـ direction reg */
                                u8  *bit);         /* bit رقم 0-31 */
```

الـ co-processor firmware كان بيعامل `vreg_offset` كـ absolute address بدل offset من الـ GPIO base:

```c
/* firmware — WRONG */
void set_gpio_value(u16 vreg_offset, u8 bit, int val)
{
    /* BUG: treating offset as absolute address! */
    volatile u32 *reg = (u32 *)vreg_offset;
    if (val)
        *reg |= BIT(bit);
    else
        *reg &= ~BIT(bit);
}

/* firmware — CORRECT */
void set_gpio_value(u16 vreg_offset, u8 bit, int val)
{
    /* vreg_offset is relative to GPIO_BASE */
    volatile u32 *reg = (u32 *)(GPIO_BASE + vreg_offset);
    if (val)
        *reg |= BIT(bit);
    else
        *reg &= ~BIT(bit);
}
```

بسبب الخطأ ده، الـ co-processor كان بيكتب على address اتعمل فيها wraparound وبيوقع على GPIO register تاني خالص بيتحكم في الـ 12V rail.

للتحقق من الـ offsets الصح:

```bash
# تحقق من GPIO register map للـ AST2600
devmem2 0x1e780000 w  # GPIO_A value register
devmem2 0x1e780004 w  # GPIO_B value register
# vreg_offset for GPIO_A = 0x000, GPIO_B = 0x004, etc.
```

#### الحل
التوثيق الصريح في الـ interface بين Linux وـ co-processor firmware:

```c
/* shared header between Linux driver and co-processor firmware */
#define AST2600_GPIO_BASE  0x1E780000UL

struct copro_gpio_info {
    u32 abs_vreg_addr;  /* = GPIO_BASE + vreg_offset */
    u32 abs_dreg_addr;  /* = GPIO_BASE + dreg_offset */
    u8  bit;
};

/* Helper to convert kernel output to firmware-friendly struct */
static void fill_copro_info(struct gpio_desc *desc,
                             struct copro_gpio_info *info)
{
    u16 vreg, dreg;
    u8 bit;

    aspeed_gpio_copro_grab_gpio(desc, &vreg, &dreg, &bit);
    info->abs_vreg_addr = AST2600_GPIO_BASE + vreg;
    info->abs_dreg_addr = AST2600_GPIO_BASE + dreg;
    info->bit = bit;
}
```

#### الدرس المستفاد
الـ `vreg_offset` و `dreg_offset` هما **offsets نسبة لـ GPIO base**، مش absolute addresses. أي firmware بيستخدم الـ info ده لازم يضيف `GPIO_BASE` بنفسه. وثّق ده صراحةً في الـ shared header بين Linux وـ co-processor.

---

### السيناريو الخامس: AST2500 BMC — memory corruption بسبب استخدام gpio_desc بعد release

#### العنوان
**Use-after-free في GPIO co-processor driver بعد إعادة تشغيل الـ co-processor**

#### السياق
في **enterprise server BMC** على **AST2500**، عملية الـ firmware update بتاعة الـ co-processor بتعمل reset للـ co-processor في runtime بدون إعادة تشغيل الـ BMC. الـ Linux kernel لازم يتعامل مع ده بشكل آمن.

#### المشكلة
بعد عملية **co-processor firmware update** ناجحة، الـ BMC بيحتاج 30-60 ثانية قبل ما الـ IPMI يرجع يشتغل تاني، وأحياناً الـ BMC بيـ crash بالكامل:

```
[  234.556677] BUG: KASAN: use-after-free in aspeed_gpio_copro_release_gpio
[  234.556680] Read of size 8 at addr ffff000012345678 by task bmc-updater/423
```

#### التحليل
الـ update process كانت بتعمل الآتي:

```c
/* firmware updater — WRONG sequence */
void update_copro_firmware(void)
{
    /* 1. Stop co-processor */
    aspeed_copro_stop();

    /* 2. Update firmware */
    flash_new_firmware();

    /* 3. Deregister ops */
    aspeed_gpio_copro_set_ops(NULL, NULL);

    /* 4. Restart co-processor */
    aspeed_copro_start();

    /* 5. Re-register ops — but gpio_desc pointers might be stale! */
    aspeed_gpio_copro_set_ops(&my_ops, my_data);

    /* BUG: grab_gpio called again with same desc — but was already grabbed */
    aspeed_gpio_copro_grab_gpio(fan_gpio, &vreg, &dreg, &bit);
}
```

المشكلة إن `aspeed_gpio_copro_release_gpio` لازم يتعمل على كل GPIO اتـ grabbed قبل إعادة التسجيل. من غير ده، الـ GPIO driver عنده pointers على descriptors ممكن تكون stale أو في state غير consistent.

```c
int aspeed_gpio_copro_release_gpio(struct gpio_desc *desc);
/* لازم تتعمل على كل GPIO قبل aspeed_gpio_copro_set_ops(NULL, NULL) */
```

#### الحل
الـ teardown sequence الصحيحة:

```c
/* firmware updater — CORRECT sequence */
void update_copro_firmware(void)
{
    /* 1. Release ALL grabbed GPIOs first */
    aspeed_gpio_copro_release_gpio(fan_gpio);
    aspeed_gpio_copro_release_gpio(power_gpio);
    aspeed_gpio_copro_release_gpio(alert_gpio);

    /* 2. Deregister ops — now safe */
    aspeed_gpio_copro_set_ops(NULL, NULL);

    /* 3. Stop co-processor */
    aspeed_copro_stop();

    /* 4. Update firmware */
    flash_new_firmware();

    /* 5. Restart co-processor */
    aspeed_copro_start();

    /* 6. Re-register ops */
    aspeed_gpio_copro_set_ops(&my_ops, my_data);

    /* 7. Re-grab GPIOs — fresh, clean state */
    aspeed_gpio_copro_grab_gpio(fan_gpio, &vreg, &dreg, &bit);
    aspeed_gpio_copro_grab_gpio(power_gpio, &pwr_vreg, &pwr_dreg, &pwr_bit);
    aspeed_gpio_copro_grab_gpio(alert_gpio, &alt_vreg, &alt_dreg, &alt_bit);
}
```

للتحقق من الـ state بعد الـ update:

```bash
# تأكد إن الـ GPIOs اتـ release قبل الـ update
cat /sys/kernel/debug/gpio | grep "copro"

# بعد الـ update تأكد إن الـ ops اتسجلت تاني
dmesg | tail -20 | grep aspeed-gpio
```

#### الدرس المستفاد
عند أي عملية تعيد تهيئة الـ co-processor، لازم تتبع الترتيب: **release_gpio** لكل GPIO اتـ grabbed أولاً، بعدين **set_ops(NULL)**, بعدين الـ reset، بعدين إعادة الـ registration. أي ترتيب غير ده بيفتح الباب لـ use-after-free وـ memory corruption.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

**الـ** LWN.net هو المصدر الأهم لمتابعة تطور دعم Aspeed في kernel Linux.

| المقالة | الموضوع |
|---------|---------|
| [Introduce ASPEED AST27XX BMC SoC](https://lwn.net/Articles/984380/) | إضافة دعم جيل AST2700 (Cortex-A35) للـ kernel |
| [OpenBMC, a distribution for baseboard management controllers](https://lwn.net/Articles/683320/) | نظرة شاملة على OpenBMC ودور Aspeed فيه |
| [Add Aspeed SSIF BMC driver](https://lwn.net/Articles/856551/) | مناقشة driver الـ SSIF لـ IPMI من جانب الـ BMC |
| [Add SSIF BMC driver](https://lwn.net/Articles/909814/) | الإصدار النهائي من الـ patch بعد مراجعات متعددة |
| [Aspeed: Add SCU interrupt controller and XDMA engine drivers](https://lwn.net/Articles/804280/) | إضافة interrupt controller للـ SCU وكذلك XDMA engine |
| [soc: aspeed: Add XDMA engine driver](https://lwn.net/Articles/819546/) | driver الـ XDMA اللي بيسمح بـ DMA بين الـ host والـ BMC |
| [misc: aspeed: Add Aspeed UART routing control driver](https://lwn.net/Articles/839666/) | driver التحكم في توجيه الـ UART على AST2500/AST2600 |
| [spi: spi-mem: Add driver for Aspeed SMC controllers](https://lwn.net/Articles/886809/) | driver الـ SPI لـ flash controllers على Aspeed SoCs |
| [Static memory controllers for the Aspeed SoC](https://lwn.net/Articles/708880/) | دعم الـ static memory controllers على أجيال أقدم |
| [ipmi: Allow raw access to KCS devices](https://lwn.net/Articles/846806/) | مناقشة الوصول المباشر لواجهة KCS الـ IPMI |

---

### توثيق الـ kernel الرسمي

الملفات دي موجودة في الـ source tree تحت `Documentation/`:

#### Device Tree Bindings
```
Documentation/devicetree/bindings/arm/aspeed/aspeed.yaml
Documentation/devicetree/bindings/arm/aspeed/aspeed,sbc.yaml
Documentation/devicetree/bindings/mfd/aspeed,ast2x00-scu.yaml
Documentation/devicetree/bindings/mfd/aspeed-lpc.yaml
Documentation/devicetree/bindings/ipmi/aspeed,ast2400-ibt-bmc.yaml
Documentation/devicetree/bindings/ipmi/aspeed,ast2400-kcs-bmc.yaml
Documentation/devicetree/bindings/gpio/aspeed,ast2400-gpio.yaml
Documentation/devicetree/bindings/i2c/aspeed,i2c.yaml
Documentation/devicetree/bindings/spi/aspeed,ast2600-fmc.yaml
Documentation/devicetree/bindings/media/aspeed,video-engine.yaml
```

#### Hardware Monitor
```
Documentation/hwmon/aspeed-pwm-tacho.rst
Documentation/hwmon/aspeed-g6-pwm-tach.rst
```

#### Video Engine
```
Documentation/userspace-api/media/drivers/aspeed-video.rst
```

#### IPMI
```
Documentation/driver-api/ipmi.rst   ← شرح كامل لنظام IPMI في الـ kernel
```

---

### Header Files الأساسية في الـ Kernel

الـ Aspeed subsystem موزع على عدة headers بدل ملف واحد:

| الملف | الوظيفة |
|-------|---------|
| `include/linux/gpio/aspeed.h` | واجهة الـ co-processor للـ GPIO |
| `include/uapi/linux/aspeed-lpc-ctrl.h` | ioctl definitions لـ LPC window mapping |
| `include/uapi/linux/aspeed-p2a-ctrl.h` | ioctl definitions لـ PCIe-to-AHB bridge |
| `include/uapi/linux/aspeed-video.h` | V4L2 controls خاصة بـ Aspeed video engine |
| `include/dt-bindings/clock/aspeed-clock.h` | ثوابت الـ clock IDs |
| `include/dt-bindings/gpio/aspeed-gpio.h` | ثوابت الـ GPIO |
| `include/dt-bindings/interrupt-controller/aspeed-scu-ic.h` | interrupt IDs للـ SCU |

---

### Commits مهمة في تاريخ الـ Subsystem

#### إضافة AST2600 (Linux 5.4)
الـ AST2600 اتضاف في Linux 5.4 كـ Cortex-A7 dual-core BMC. المعلومة من kernelnewbies.org:
- [Linux 5.4 — kernelnewbies.org](https://kernelnewbies.org/Linux_5.4) — أول kernel فيه دعم كامل للـ AST2600 وعدد كبير من الـ OpenBMC boards زي Facebook Minipack و Wedge100.

#### إضافة دعم AST2700 (Linux 6.x)
- [Linux 6.17 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.17) — إضافة ASPEED AST27XX BMC boards جديدة.
- [Linux 5.16 — kernelnewbies.org](https://kernelnewbies.org/Linux_5.16) — إضافة TYAN S7106 و Inventec Transformers BMC.

#### Core SoC Support (Linux 4.9)
- [Linux 4.9 — kernelnewbies.org](https://kernelnewbies.org/Linux_4.9) — أول إضافة للـ Aspeed SoC core support مع `pinctrl-aspeed-g4` و `pinctrl-aspeed-g5`.

---

### Patchwork ومناقشات الـ Mailing List

| الرابط | الموضوع |
|--------|---------|
| [ASPEED KCS IPMI BMC driver — Patchwork](https://patchwork.ozlabs.org/project/openbmc/patch/1516810009-16353-1-git-send-email-haiyue.wang@linux.intel.com/) | أول نسخة من driver الـ KCS IPMI للـ AST2500 |
| [SSIF BMC driver v9 — Patchwork](https://patchwork.ozlabs.org/project/linux-aspeed/patch/20220929080326.752907-2-quan@os.amperecomputing.com/) | النسخة التاسعة من SSIF BMC driver بعد review طويل |
| [AST2600 GPIO driver — Patchwork](https://patchwork.kernel.org/project/linux-arm-kernel/patch/20190906063737.15428-1-rashmica.g@gmail.com/) | إضافة AST2600 details لـ Aspeed GPIO driver |
| [AST2600 I2C new mode — Patchwork ozlabs](https://patchwork.ozlabs.org/project/linux-aspeed/patch/20220323004009.943298-3-ryan_chen@aspeedtech.com/) | دعم register mode الجديد للـ I2C على AST2600 |
| [BMC GFX DRM driver MAINTAINERS — lore.kernel.org](https://lore.kernel.org/linux-arm-kernel/3c142c1e-e863-4f21-b313-3564e3a9d5d6@www.fastmail.com/) | إضافة Aspeed BMC GFX DRM driver لـ MAINTAINERS |

**الـ** mailing lists الرئيسية للمتابعة:
- `linux-aspeed@lists.ozlabs.org` — قائمة Aspeed المتخصصة
- `linux-arm-kernel@lists.infradead.org` — ARM kernel العام
- `openbmc@lists.ozlabs.org` — مجتمع OpenBMC

---

### QEMU Emulation لـ Aspeed BMC

للتجربة من غير hardware حقيقي، الـ QEMU بيدعم عدد كبير من Aspeed boards:
- [QEMU Aspeed family boards documentation](https://www.qemu.org/docs/master/system/arm/aspeed.html) — يشرح كيف تشغل OpenBMC image على AST2500/AST2600/AST2700 داخل QEMU

---

### Phoronix — أخبار Driver الجديدة

- [ASpeed AST2500 SSIF BMC Driver — Phoronix](https://www.phoronix.com/news/ASpeed-SSIF-BMC-Linux-Driver) — تغطية إضافة SSIF driver للـ kernel

---

### الكتب المرجعية

#### Linux Device Drivers (LDD3)
- **الفصل 9**: Communicating with Hardware — أساسيات الـ `ioremap` و `readl/writel` اللي كل driver في Aspeed بيستخدمها
- **الفصل 14**: The Linux Device Model — فهم `platform_device` و `of_device_id`
- **الفصل 18**: TTY Drivers — مفيد لفهم UART routing driver
- متاح مجاناً: [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 17**: Devices and Modules — كيف بيتسجل الـ platform driver
- **الفصل 13**: The Virtual Filesystem — مفيد لفهم `/dev` entries للـ BMC devices
- **الفصل 7**: Interrupts and Interrupt Handlers — أساسي لفهم VIC و SCU interrupt controllers

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 14**: Kernel Debugging Techniques — debugging على ARM embedded SoCs
- **الفصل 15**: Porting Linux to a Custom Platform — مفيد لفهم كيف اتضاف Aspeed للـ kernel

---

### مصادر إضافية

#### OpenBMC Project
- [OpenBMC GitHub](https://github.com/openbmc/openbmc) — التوزيعة الرسمية للـ BMC
- [OpenBMC kernel development guide](https://github.com/openbmc/docs/blob/master/kernel-development.md) — دليل إضافة patches للـ linux-aspeed

#### ASPEED Technology الرسمي
- [aspeedtech.com](https://www.aspeedtech.com) — datasheet و application notes للـ AST2400/2500/2600/2700

#### Linux Kernel IPMI Documentation
- [The Linux IPMI Driver — kernel.org](https://docs.kernel.org/driver-api/ipmi.html) — توثيق رسمي لكل interfaces الـ IPMI في الـ kernel

#### LKDDb — Kernel Driver Database
- [CONFIG_ASPEED_KCS_IPMI_BMC — lkddb](https://cateee.net/lkddb/web-lkddb/ASPEED_KCS_IPMI_BMC.html) — قائمة بكل Kconfig options الخاصة بـ Aspeed

---

### Search Terms للبحث عن مزيد من المعلومات

```
aspeed ast2600 linux kernel driver
aspeed BMC OpenBMC lore.kernel.org
linux-aspeed mailing list
aspeed LPC IPMI KCS BMC driver
aspeed GPIO co-processor
aspeed XDMA PCIe BMC
aspeed SCU system control unit
aspeed video engine V4L2
aspeed HACE crypto driver
aspeed FSI master IBM OpenPOWER
site:lwn.net aspeed
site:patchwork.ozlabs.org linux-aspeed
```
## Phase 8: Writing simple module

### الـ hook المختار: `aspeed_gpio_copro_set_ops`

**الـ function** دي هي الأهم في الـ API كلها — هي اللي بتسجّل الـ ops struct اللي بيحدد مين يتحكم في الـ GPIO access (الـ coprocessor ولا الـ ARM host). هنستخدم **kprobe** عشان نـ intercept أي call ليها ونطبع الـ ops pointer والـ data pointer.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_aspeed_copro.c
 * Intercepts aspeed_gpio_copro_set_ops() to log whenever
 * a coprocessor registers its GPIO access callbacks.
 */

#include <linux/module.h>       /* module_init / module_exit / MODULE_* macros  */
#include <linux/kernel.h>       /* pr_info / pr_err                              */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe, etc.          */
#include <linux/gpio/aspeed.h>  /* struct aspeed_gpio_copro_ops prototype        */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Doc Demo");
MODULE_DESCRIPTION("kprobe on aspeed_gpio_copro_set_ops to trace copro GPIO ops registration");

/* ------------------------------------------------------------------ *
 * kprobe pre-handler
 * Called right before aspeed_gpio_copro_set_ops() executes.
 *
 * On arm64:
 *   regs->regs[0]  → first  arg  (const struct aspeed_gpio_copro_ops *ops)
 *   regs->regs[1]  → second arg  (void *data)
 * ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Pull both arguments straight from the CPU registers */
    const struct aspeed_gpio_copro_ops *ops =
        (const struct aspeed_gpio_copro_ops *)regs->regs[0];
    void *data = (void *)regs->regs[1];

    if (!ops) {
        /* Passing NULL ops means "unregister" — log that too */
        pr_info("aspeed_copro_probe: copro ops CLEARED (data=%p)\n", data);
        return 0;
    }

    pr_info("aspeed_copro_probe: new ops registered\n");
    pr_info("  ops ptr        = %p\n", ops);
    pr_info("  request_access = %ps\n", ops->request_access);  /* %ps prints symbol name */
    pr_info("  release_access = %ps\n", ops->release_access);
    pr_info("  private data   = %p\n", data);

    return 0; /* 0 = let the original function run normally */
}

/* ------------------------------------------------------------------ *
 * kprobe descriptor
 * We attach by symbol name — no hard-coded address needed.
 * ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "aspeed_gpio_copro_set_ops",
    .pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ *
 * module_init
 * ------------------------------------------------------------------ */
static int __init aspeed_copro_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("aspeed_copro_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("aspeed_copro_probe: kprobe planted at %ps (%p)\n",
            kp.addr, kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ *
 * module_exit
 * ------------------------------------------------------------------ */
static void __exit aspeed_copro_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("aspeed_copro_probe: kprobe removed\n");
}

module_init(aspeed_copro_probe_init);
module_exit(aspeed_copro_probe_exit);
```

---

### شرح كل جزء

#### الـ includes

| Header | ليه موجود |
|--------|-----------|
| `linux/module.h` | الـ macros الأساسية لأي kernel module |
| `linux/kernel.h` | `pr_info` / `pr_err` للـ logging |
| `linux/kprobes.h` | الـ kprobe API كله — `struct kprobe`، `register_kprobe`، إلخ |
| `linux/gpio/aspeed.h` | تعريف `struct aspeed_gpio_copro_ops` عشان نقدر نـ cast وناخد الـ function pointers |

---

#### الـ `handler_pre`

الـ pre-handler بيتشغل **قبل** ما الـ function الأصلية تبدأ، فـ الـ arguments لسه موجودة في الـ registers.
بنـ cast `regs->regs[0]` لـ `const struct aspeed_gpio_copro_ops *` عشان نطبع الـ callbacks المسجَّلة باسمها الرمزي باستخدام `%ps`.

---

#### الـ `struct kprobe`

بنحدد `symbol_name` بدل عنوان ثابت عشان الكود يشتغل على أي kernel build من غير ما نحتاج `/proc/kallsyms` يدوياً.
الـ `pre_handler` هو الـ callback الوحيد المطلوب هنا — `post_handler` و`fault_handler` اختياريين.

---

#### الـ `module_init`

`register_kprobe` بيـ patch الـ instruction الأولى في `aspeed_gpio_copro_set_ops` بـ breakpoint (INT3 على x86، BRK على arm64).
لو فشل — مثلاً لو الـ symbol مش exported أو الـ kprobes مش enabled في الـ config — بنرجع الـ error مباشرة.

---

#### الـ `module_exit`

`unregister_kprobe` بيشيل الـ breakpoint ويستعيد الـ original instruction.
**لازم** يتعمل في الـ exit عشان لو الـ module اتـ unload والـ breakpoint فضل، أي call للـ function بعد كده هيـ crash الـ kernel.

---

### Makefile للتجربة

```makefile
obj-m += kprobe_aspeed_copro.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

```bash
# بناء وتحميل
make
sudo insmod kprobe_aspeed_copro.ko

# مشاهدة الـ log
sudo dmesg -w | grep aspeed_copro_probe

# إزالة
sudo rmmod kprobe_aspeed_copro
```

> **ملاحظة:** على نظام غير ASPEED (مثلاً x86 VM)، الـ symbol `aspeed_gpio_copro_set_ops` مش هيكون موجود، فـ `register_kprobe` هترجع `-ENOENT` والـ module هيفشل بشكل نظيف من غير crash.
