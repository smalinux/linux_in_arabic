# شرح `omap.h` — OMAP Pinctrl Hardware Bindings

هذا الملف مختلف عن كل ما شرحناه سابقاً — بدلاً من الـ abstractions العامة، هذا **كود مرتبط بـ hardware حقيقي** من Texas Instruments (OMAP/AM33xx). يكشف كيف تُترجم مفاهيم الـ pinctrl إلى bits فعلية في registers.

---

## أولاً: بنية الـ OMAP Pad Control Register

قبل شرح الـ macros، يجب فهم بنية الـ register الفعلي في الـ hardware:

```
OMAP3/4/5 PAD CONTROL REGISTER (16-bit أو 32-bit حسب الجيل):

Bit:  15      14      13      12      11      10      9       8
    ┌───────┬───────┬───────┬───────┬───────┬───────┬───────┬───────┐
    │WAKEUP │WAKEUP │OFF_   │OFF_   │OFFOUT │OFFOUT │ OFF   │INPUT  │
    │_EVENT │_EN    │PULL_UP│PULL_EN│_VAL   │_EN    │_EN    │_EN    │
    └───────┴───────┴───────┴───────┴───────┴───────┴───────┴───────┘

Bit:  7       6       5       4       3       2     1     0
    ┌───────┬───────┬───────┬───────┬───────┬─────────────────────┐
    │  ---  │  ---  │ALTELEC│PULL_UP│PULL   │    MUX_MODE [2:0]   │
    │       │       │TRICAL │       │_ENA   │    (0..7)           │
    └───────┴───────┴───────┴───────┴───────┴─────────────────────┘
```

كل `#define` في الملف هو **bit mask** لهذا الـ register.

---

## ثانياً: MUX_MODE — اختيار الوظيفة

```c
#define MUX_MODE0   0   // 000b
#define MUX_MODE1   1   // 001b
#define MUX_MODE2   2   // 010b
#define MUX_MODE3   3   // 011b
#define MUX_MODE4   4   // 100b
#define MUX_MODE5   5   // 101b
#define MUX_MODE6   6   // 110b
#define MUX_MODE7   7   // 111b  ← عادةً GPIO على OMAP
```

الـ bits [2:0] في الـ register تختار من 8 وظائف ممكنة للـ pin. على OMAP، MUX_MODE7 هو دائماً تقريباً GPIO mode.

```
Pin "mcspi1_clk" مثلاً:
  MUX_MODE0 → SPI1_CLK      (الوظيفة الأساسية)
  MUX_MODE1 → SDI           (وظيفة بديلة)
  MUX_MODE4 → GPIO_171      (GPIO mode)
  MUX_MODE7 → SAFE_MODE     (وضع آمن - output disable)
```

---

## ثالثاً: الـ Bias و Input Bits — الخصائص الكهربائية الأساسية

### للجيل 24xx/34xx:

```c
#define PULL_ENA         (1 << 3)  // bit 3: تفعيل الـ pull resistor
#define PULL_UP          (1 << 4)  // bit 4: pull-up=1، pull-down=0
#define ALTELECTRICALSEL (1 << 5)  // bit 5: إعداد كهربائي بديل (نادر)
```

```
PULL_ENA=1, PULL_UP=0 → pull-down مُفعَّل
PULL_ENA=1, PULL_UP=1 → pull-up مُفعَّل
PULL_ENA=0            → بدون pull (floating)
```

### للجيل 34xx/4/5 (إضافات):

```c
#define INPUT_EN    (1 << 8)   // تفعيل الـ input buffer
```

على OMAP، الـ input buffer **مُعطَّل افتراضياً** لتوفير الطاقة. يجب تفعيله صراحةً لأي pin تقرأ منه. هذا يختلف عن معظم الـ SoCs الأخرى.

---

## رابعاً: OFF Mode Bits — وضع الإيقاف الكامل

هذه ميزة مميزة في OMAP لا توجد في كل الـ SoCs: القدرة على التحكم في حالة الـ pin حتى عندما تكون الـ logic الداخلية مُغلقة كلياً (off mode).

```c
#define OFF_EN      (1 << 9)   // تفعيل off mode control لهذا الـ pin
#define OFFOUT_EN   (1 << 10)  // في off mode: جعل الـ pin output
#define OFFOUT_VAL  (1 << 11)  // في off mode: قيمة الـ output (HIGH=1, LOW=0)
#define OFF_PULL_EN (1 << 12)  // في off mode: تفعيل pull resistor
#define OFF_PULL_UP (1 << 13)  // في off mode: pull-up=1، pull-down=0
```

```
السيناريو: UART TX pin عند دخول الـ system في off mode

بدون OFF_EN: الـ pin يطفو (floating) → يمكن أن يستهلك تياراً
مع OFF_EN + OFFOUT_EN + OFFOUT_VAL=0:
    → الـ pin يُثبَّت على LOW رغم إيقاف الـ core
    → استهلاك طاقة أقل + لا تشويش على الدارة المتصلة
```

### Wakeup Bits:

```c
#define WAKEUP_EN    (1 << 14)  // هذا الـ pin يمكنه إيقاظ النظام من off mode
#define WAKEUP_EVENT (1 << 15)  // قراءة فقط: هل هذا الـ pin هو من أيقظ النظام؟
```

`WAKEUP_EN` يجعل الـ pin يعمل كـ wakeup source — تغيير مستواه الكهربائي يُوقظ الـ SoC من أعمق حالات السكون.

---

## خامساً: الـ Active State Macros — التركيبات الجاهزة

هذه macros تجمع الـ bits السابقة في تركيبات شائعة:

```c
// Output بدون pull
#define PIN_OUTPUT           0
// Output + pull-up (للـ open-drain مثلاً)
#define PIN_OUTPUT_PULLUP    (PIN_OUTPUT | PULL_ENA | PULL_UP)
// Output + pull-down
#define PIN_OUTPUT_PULLDOWN  (PIN_OUTPUT | PULL_ENA)

// Input (يجب تفعيل INPUT_EN!)
#define PIN_INPUT            INPUT_EN
// Input + pull-up (أكثر استخداماً للـ buttons)
#define PIN_INPUT_PULLUP     (PULL_ENA | INPUT_EN | PULL_UP)
// Input + pull-down
#define PIN_INPUT_PULLDOWN   (PULL_ENA | INPUT_EN)
```

```
استخدام في الـ Device Tree:
&pinmux {
    uart3_pins: uart3-pins {
        pinctrl-single,pins = 
            OMAP3_CORE1_IOPAD(0x216e, PIN_INPUT_PULLUP | MUX_MODE0)  // RX
            OMAP3_CORE1_IOPAD(0x2170, PIN_OUTPUT       | MUX_MODE0)  // TX
        >;
    };
};
```

---

## سادساً: الـ Off Mode State Macros

```c
#define PIN_OFF_NONE          0                               // لا تحكم في off mode
#define PIN_OFF_OUTPUT_HIGH   (OFF_EN | OFFOUT_EN | OFFOUT_VAL)  // ثبّت HIGH
#define PIN_OFF_OUTPUT_LOW    (OFF_EN | OFFOUT_EN)               // ثبّت LOW
#define PIN_OFF_INPUT_PULLUP  (OFF_EN | OFFOUT_EN | OFF_PULL_EN | OFF_PULL_UP)
#define PIN_OFF_INPUT_PULLDOWN(OFF_EN | OFFOUT_EN | OFF_PULL_EN)
#define PIN_OFF_WAKEUPENABLE  WAKEUP_EN                      // يُوقظ النظام
```

```
مثال كامل على pin يعمل كـ UART TX وعند الـ off mode يُثبَّت LOW:

OMAP3_CORE1_IOPAD(0x2170, PIN_OUTPUT | MUX_MODE0 | PIN_OFF_OUTPUT_LOW)

القيمة النهائية في الـ register:
  bits [1:0] = 0 (MUX_MODE0 = UART TX)
  bit  [8]   = 0 (INPUT_EN  = off، هذا output)
  bit  [9]   = 1 (OFF_EN    = on)
  bit  [10]  = 1 (OFFOUT_EN = output في off mode)
  bit  [11]  = 0 (OFFOUT_VAL= LOW)
```

---

## سابعاً: الـ Address Macros — التحويل بين العناوين الفيزيائية والـ Offsets

هذا الجزء الأكثر إثارةً للاهتمام من الملف. يحل مشكلة عملية حقيقية.

### المشكلة:

الـ datasheet يعطيك **عنوان فيزيائي مطلق** للـ pad register (مثلاً `0x4800216E`)، لكن الـ pinctrl driver يعمل بـ **offset من base address**.

### الحل:

```c
#define OMAP_IOPAD_OFFSET(pa, offset)  (((pa) & 0xffff) - (offset))
```

يأخذ آخر 16-bit من العنوان الفيزيائي ويطرح منها الـ base offset ليُنتج الـ offset النسبي.

```
مثال لـ OMAP3:
  Base address: 0x48002000
  Base offset في الـ macro: 0x2030

  Pin register عند العنوان: 0x4800216E
  آخر 16-bit: 0x216E
  الـ offset:  0x216E - 0x2030 = 0x013E

#define OMAP3_CORE1_IOPAD(pa, val)  OMAP_IOPAD_OFFSET((pa), 0x2030) (val)
OMAP3_CORE1_IOPAD(0x4800216E, PIN_INPUT | MUX_MODE0)
→ ينتج: 0x013E  (PIN_INPUT | MUX_MODE0)
```

### كل SoC له base offset مختلف:

```
SoC             │  Base Offset
────────────────┼─────────────
OMAP2420        │  0x0030
OMAP2430        │  0x2030
OMAP3 Core1     │  0x2030
OMAP3430 Core2  │  0x25d8
OMAP3630 Core2  │  0x25a0
OMAP3 Wakeup    │  0x2a00
AM33xx / DM814x │  0x0800
```

### AM33xx له صيغة خاصة:

```c
#define AM33XX_IOPAD(pa, val)    OMAP_IOPAD_OFFSET((pa), 0x0800) (val) (0)
#define AM33XX_PADCONF(pa, conf, mux) OMAP_IOPAD_OFFSET((pa), 0x0800) (conf) (mux)
```

الـ `(0)` الإضافي في `AM33XX_IOPAD` لأن AM33xx يستخدم register بعرض مختلف أو له حقل إضافي.

---

## ثامناً: الـ UART RX Pin Defines

```c
#define OMAP3_UART1_RX  0x152
#define OMAP3_UART2_RX  0x14a
#define OMAP3_UART3_RX  0x16e
#define OMAP4_UART2_RX  0xdc
// ...
```

هذه مجرد offsets جاهزة للـ pins الشائعة الاستخدام. توفيراً للوقت والرجوع للـ datasheet في كل مرة.

---

## الصورة الكاملة — من الـ DTS للـ Register

```
Device Tree:
  OMAP3_CORE1_IOPAD(0x4800216E, PIN_INPUT_PULLUP | MUX_MODE0)
         │
         ▼
  offset = 0x216E - 0x2030 = 0x013E
  value  = (PULL_ENA | INPUT_EN | PULL_UP | MUX_MODE0)
         = (0x008   | 0x100    | 0x010   | 0x000)
         = 0x118
         │
         ▼
  pinctrl-single driver يكتب:
  writew(0x118, base + 0x013E)
         │
         ▼
  الـ Hardware Register عند 0x4800216E:
  ┌──────────────────────────────────┐
  │ bit8=1(INPUT_EN)                 │
  │ bit4=1(PULL_UP)                  │
  │ bit3=1(PULL_ENA)                 │
  │ bits[2:0]=0(MUX_MODE0=UART3_RX) │
  └──────────────────────────────────┘
```

هذا الملف يمثل **النهاية الأخيرة** لكل الـ abstraction layers التي شرحناها — هنا تتحول الـ concepts إلى bits حقيقية في hardware حقيقي.