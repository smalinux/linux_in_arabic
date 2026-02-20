# شرح الـ Common Clock Framework 🕐

## إيه هو الموضوع ده أصلاً؟

تخيل إن الـ **SoC** (الشريحة) اللي في موبايلك أو في الـ board بتاعك زي مدينة كبيرة فيها مصانع كتير (peripherals زي USB, I2C, SPI). كل مصنع ده محتاج كهربا عشان يشتغل، صح؟

الـ **clock** في الإلكترونيات زي الكهربا دي بالظبط! كل **peripheral** محتاج **clock signal** عشان يشتغل - يعني إشارة بتنبض بسرعة معينة (زي tick-tock-tick-tock) عشان الدوائر تشتغل.

---

## ليه محتاجين الـ Framework ده؟

### المشكلة 🤔
قبل كده، كل **platform** (يعني كل شركة زي STM أو NXP أو Rockchip) كانت بتعمل الكود بتاعها بطريقتها الخاصة. يعني **code duplication** كتير جداً!

### الحل ✅
الـ **Common Clock Framework** جاء يقول: "يلا كلنا نستخدم نفس الطريقة!"

يعني بدل ما كل واحد يعمل العجلة من الأول، نعمل **standard interface** واحد للكل.

---

## إزاي الـ Framework ده شغال؟

### مقسوم لنصين (Two Halves):

#### **النص الأول: الـ Common Core** 🏛️
ده موجود في `drivers/clk/clk.c` وفيه:
- **struct clk_core**: البنية الأساسية اللي فيها معلومات عن الـ clock
  - اسم الـ clock
  - مين الـ **parent** بتاعه (الأب)
  - السرعة (rate)
  - عداد كام مرة اتشغل (enable_count)

تخيله زي **مدير المصنع** - بيمسك السجلات والحسابات.

#### **النص التاني: الـ Hardware-Specific** ⚙️
ده الكود اللي **انت** هتكتبه لـ platform بتاعك! فيه:
- **struct clk_ops**: مجموعة functions عشان تتحكم في الـ hardware الحقيقي
  - `.enable`: شغّل الـ clock
  - `.disable`: قفل الـ clock
  - `.set_rate`: غيّر السرعة
  - `.set_parent`: غيّر الأب

تخيله زي **العمال** اللي بيشتغلوا في المصنع فعلاً.

---

## مثال بسيط: إزاي تشغّل Clock؟

### 1. الـ Driver بيقول:
```c
clk_enable(clk);  // يا كود، شغّل الـ clock ده!
```

### 2. الـ Framework بيعمل:
```
clk_enable() → يدوّر على الـ ops المناسبة
  ↓
clk_gate_enable() → ده الكود بتاعك اللي بيشتغل الـ hardware
  ↓
clk_gate_set_bit() → بيروح يكتب في الـ register ويقلب الـ bit
```

تخيلها زي لما بتضغط على **مفتاح الكهربا** (enable)، الكهربا بتوصل للمصباح (hardware)!

---

## الـ Structures المهمة

### **struct clk_core** (المدير)
```c
struct clk_core {
    const char *name;           // اسم الـ clock (مثلاً "usb_clk")
    struct clk_core *parent;    // مين الأب؟
    unsigned long rate;         // السرعة (frequency)
    struct clk_ops *ops;        // الـ functions بتاعتك
};
```

### **struct clk_ops** (التعليمات)
```c
struct clk_ops {
    int (*enable)(struct clk_hw *hw);      // شغّل
    void (*disable)(struct clk_hw *hw);    // قفل
    unsigned long (*recalc_rate)(...);     // احسب السرعة
    int (*set_rate)(...);                  // غيّر السرعة
    int (*set_parent)(...);                // غيّر الأب
};
```

---

## الـ Locking (الأقفال) 🔒

عشان ما يحصلش **race condition** (اتنين يشتغلوا على نفس الـ clock في نفس الوقت)، فيه نوعين أقفال:

### 1. **Enable Lock** (Spinlock) 🏃
- **سريع جداً**
- يُستخدم مع `.enable` و `.disable`
- **ممنوع النوم** (can't sleep) داخله!
- تقدر تستخدمه في **atomic context**

تخيله زي قفل بسيط بتلفه بسرعة.

### 2. **Prepare Lock** (Mutex) 😴
- **يسمح بالنوم**
- يُستخدم مع باقي العمليات (set_rate, set_parent...)
- **لازم يكون في process context**

تخيله زي قفل أكبر ممكن تستنى عنده.

---

## ليه فيه Enable و Prepare؟

### **Prepare** (التحضير):
- بيجهز الـ clock (مثلاً يشغل الـ PLL)
- **ممكن ياخد وقت** (microseconds أو milliseconds)
- ممكن ينام

### **Enable** (التشغيل الفعلي):
- بيشغل الـ clock بسرعة
- **لازم يكون سريع جداً** (nanoseconds)
- **ممنوع ينام**!

تخيلها زي السيارة:
- **Prepare** = تسخين المحرك (ياخد وقت)
- **Enable** = تشغيل السيارة فعلياً (سريع)

---

## Power Management (توفير الطاقة) ⚡

### الفكرة:
لو في **peripheral** مش شغال، ليه نسيب الـ clock بتاعه شغال ويستهلك بطارية؟

### الحل:
```c
// لما تخلص شغل
clk_disable(clk);      // اقفل الـ clock
clk_unprepare(clk);    // اطفي الـ PLL كمان

// لما تحتاجه تاني
clk_prepare(clk);      // سخّن الـ PLL
clk_enable(clk);       // شغّل الـ clock
```

كده بتوفر طاقة! 💪

---

## إزاي تعمل Driver لـ Clock بتاعك؟

### 1. عرّف الـ Structure بتاعتك:
```c
struct clk_foo {
    struct clk_hw hw;           // لازم يكون موجود!
    void __iomem *reg;          // مكان الـ register
    u8 bit_idx;                 // رقم الـ bit
};
```

### 2. اكتب الـ Operations:
```c
struct clk_ops clk_foo_ops = {
    .enable  = &clk_foo_enable,
    .disable = &clk_foo_disable,
    .set_rate = &clk_foo_set_rate,
};
```

### 3. نفّذ الـ Functions:
```c
int clk_foo_enable(struct clk_hw *hw) {
    struct clk_foo *foo = to_clk_foo(hw);

    // اكتب في الـ register
    u32 reg = readl(foo->reg);
    reg |= BIT(foo->bit_idx);  // قلب الـ bit
    writel(reg, foo->reg);

    return 0;
}
```

### 4. سجّل الـ Clock:
```c
clk_register(...);  // قول للـ kernel: عندي clock جديد!
```

---

## الخلاصة 🎯

الـ **Common Clock Framework** هو:
- **مدير موحّد** لكل الـ clocks في النظام
- بيوفر **interface** سهل للـ drivers
- بيساعد في **power management**
- **مقسوم لنصين**: common code + hardware-specific code
- فيه **two-phase locking** (prepare/enable) عشان الكفاءة

تخيل الـ kernel زي مدينة فيها **مولد كهربا رئيسي** (root clock) و**محطات فرعية** (derived clocks)، والـ framework ده هو **لوحة التحكم** اللي بتتحكم في كل حاجة! 🏗️

---

# شرح أعمق للـ Common Clock Framework 🔍

## 1. الـ Clock Tree (شجرة الـ Clocks) 🌳

### تخيل الموضوع كده:

عندك **مولد كهربا رئيسي** (crystal oscillator) بيدي 24 MHz. منه بيتفرع كل حاجة في النظام.

```
                    Crystal (24 MHz)
                          |
                    +-----+-----+
                    |           |
                  PLL1        PLL2
                (800 MHz)   (600 MHz)
                    |           |
            +-------+-------+   +-------+
            |       |       |           |
         CPU_CLK  AHB_CLK APB_CLK    USB_CLK
        (400MHz) (200MHz) (100MHz)   (48MHz)
            |       |        |           |
          [CPU]  [DMA]   [UART]       [USB]
```

كل **clock** ليه **parent** (أب)، والأب ده بييجي منه الإشارة الأصلية.

---

## 2. الـ Parent-Child Relationship (علاقة الأب والابن)

### ليه مهمة؟

لو غيّرت سرعة **الأب**، لازم **الابن** يحسب سرعته تاني!

### مثال من الحياة الواقعية:

```c
struct clk_core {
    struct clk_core *parent;        // مين الأب؟
    struct clk_core **parents;      // لو عندي أكتر من أب محتمل
    u8 num_parents;                 // عدد الآباء المحتملين
    const char **parent_names;      // أسماء الآباء
};
```

**ليه أكتر من parent محتمل؟** 🤔

لأن بعض الـ clocks بتقدر **تختار** مصدرها! زي **multiplexer**:

```
      PLL1 ----\
                 \
      PLL2 -------> [MUX] ----> USB_CLK
                 /
      Crystal --/
```

الـ driver يقدر يقول: "عايز USB_CLK ييجي من PLL1 النهارده" عن طريق:
```c
clk_set_parent(usb_clk, pll1);
```

---

## 3. أنواع الـ Clocks المختلفة

### أ) **Fixed Rate Clock** (سرعة ثابتة) 🔒

أبسط نوع! زي الـ **crystal oscillator**.

```c
struct clk_fixed_rate {
    struct clk_hw hw;
    unsigned long fixed_rate;  // السرعة الثابتة (مثلاً 24000000)
    unsigned long flags;
};
```

**مافيش** set_rate ولا حاجة، السرعة ثابتة للأبد!

#### مثال:
```c
// Crystal 24 MHz
clk_register_fixed_rate(NULL, "osc24M", NULL, 0, 24000000);
```

---

### ب) **Gate Clock** (بوابة تشغيل/إيقاف) 🚪

بيشغل ويقفل الـ clock بس، **مابيغيرش السرعة**.

```c
struct clk_gate {
    struct clk_hw hw;
    void __iomem *reg;      // عنوان الـ register
    u8 bit_idx;             // رقم الـ bit اللي بيتحكم
    u8 flags;
    spinlock_t *lock;
};
```

#### إزاي بيشتغل؟

```c
static int clk_gate_enable(struct clk_hw *hw) {
    struct clk_gate *gate = to_clk_gate(hw);
    u32 val;

    // اقرا الـ register الحالي
    val = readl(gate->reg);

    // قلّب الـ bit المطلوب لـ 1
    val |= BIT(gate->bit_idx);

    // اكتب القيمة الجديدة
    writel(val, gate->reg);

    return 0;
}
```

**في الـ hardware:**
```
Register: 0x12340000
Bits:     [31....8][7][6][5][4][3][2][1][0]
                    ^
                    |
               USB clock gate (bit 7)

عايز تشغل USB؟ اكتب 1 في bit 7
عايز تقفله؟ اكتب 0 في bit 7
```

---

### ج) **Divider Clock** (مقسّم السرعة) ➗

بياخد clock من الأب **ويقسمه**!

```c
struct clk_divider {
    struct clk_hw hw;
    void __iomem *reg;
    u8 shift;           // من فين يبدأ في الـ register
    u8 width;           // كام bit للـ divider
    u8 flags;
    const struct clk_div_table *table;  // جدول القسمة
    spinlock_t *lock;
};
```

#### مثال:
```
Parent = 800 MHz
Divider = 4
Result = 800 / 4 = 200 MHz
```

#### في الـ hardware:
```
Register: 0x12340010
Bits [2:0] = divider value
  000 = divide by 1
  001 = divide by 2
  010 = divide by 4
  011 = divide by 8
  ...
```

```c
// عايز 200 MHz من 800 MHz؟
// يعني divider = 4
// يعني bits = 010
writel(0x2, divider_reg);
```

---

### د) **Mux Clock** (المُبدِّل) 🔀

بيختار **من مين** ييجي الـ clock!

```c
struct clk_mux {
    struct clk_hw hw;
    void __iomem *reg;
    u32 *table;         // جدول المصادر
    u32 mask;
    u8 shift;
    u8 flags;
    spinlock_t *lock;
};
```

#### مثال:
```
Sources:
  00 = Crystal (24 MHz)
  01 = PLL1 (800 MHz)
  10 = PLL2 (600 MHz)
  11 = Reserved

عايز تختار PLL1؟ اكتب 01 في الـ bits المخصصة
```

---

### هـ) **PLL Clock** (أهم نوع!) ⚙️

الـ **Phase-Locked Loop** - ده اللي بيضاعف السرعة!

```c
struct clk_pll {
    struct clk_hw hw;
    void __iomem *pll_base;
    u32 m;      // Multiplier (المضاعف)
    u32 n;      // Divider (المقسم)
    u32 p;      // Post divider (مقسم إضافي)
};
```

#### المعادلة السحرية:
```
Output = (Input × M) / (N × P)

مثال:
Input = 24 MHz
M = 100
N = 3
P = 2

Output = (24 × 100) / (3 × 2) = 2400 / 6 = 400 MHz
```

**ده إزاي بنحول 24 MHz لـ 800 MHz!** 🎯

---

## 4. الـ Rate Propagation (انتشار السرعة)

### السيناريو:
```
PLL (800 MHz)
    |
Divider (/2)
    |
APB_CLK (400 MHz)
    |
UART_CLK
```

لو **غيّرت** سرعة الـ PLL لـ 1000 MHz، إيه اللي هيحصل؟

### الـ Framework بيعمل كده:

1. **يبدأ من الأب** (PLL)
2. **يحسب السرعة الجديدة** (1000 MHz)
3. **ينزل للأبناء** (Divider)
4. **الـ Divider يحسب سرعته**: 1000 / 2 = 500 MHz
5. **ينزل لـ APB_CLK**: يبقى 500 MHz
6. **UART_CLK** كمان يتأثر!

#### الكود:

```c
unsigned long clk_recalc_rate(struct clk_core *core) {
    unsigned long parent_rate = 0;

    // جيب سرعة الأب
    if (core->parent)
        parent_rate = core->parent->rate;

    // احسب سرعتك بناءً على سرعة الأب
    if (core->ops->recalc_rate)
        return core->ops->recalc_rate(core->hw, parent_rate);

    return parent_rate;  // لو مافيش حاجة، خد سرعة الأب زي ما هي
}
```

---

## 5. الـ clk_hw Abstraction (الطبقة الوسيطة)

### ليه محتاجينه؟

عشان **نفصل** بين:
- الـ **common code** (clk_core)
- الـ **hardware code** (clk_gate, clk_divider...)

```c
struct clk_hw {
    struct clk_core *core;   // pointer للـ core
    struct clk *clk;         // pointer للـ consumer interface
    const struct clk_init_data *init;
};
```

### ازاي بنتنقل بينهم؟

```c
// من hw للـ hardware structure
#define to_clk_gate(_hw) container_of(_hw, struct clk_gate, hw)

// من clk_core للـ hw
struct clk_hw *hw = core->hw;

// من hw للـ clk_core
struct clk_core *core = hw->core;
```

**container_of** دي macro سحرية بتقولك: "أنا عندي عنوان الـ member، طلّعلي عنوان الـ struct كله!"

---

## 6. التسجيل الكامل (Full Registration)

### الخطوات التفصيلية:

#### 1. حضّر الـ init data:
```c
static const char *uart_parents[] = { "pll1", "pll2", "osc24M" };

struct clk_init_data init = {
    .name = "uart_clk",
    .ops = &clk_gate_ops,
    .parent_names = uart_parents,
    .num_parents = 3,
    .flags = CLK_SET_RATE_PARENT,  // لو غيرت سرعتي، غير الأب كمان
};
```

#### 2. املأ الـ hardware structure:
```c
struct clk_gate *gate;
gate = kzalloc(sizeof(*gate), GFP_KERNEL);

gate->reg = ioremap(0x12340000, 4);  // عنوان الـ register
gate->bit_idx = 7;                    // bit رقم 7
gate->hw.init = &init;
```

#### 3. سجّل في الـ framework:
```c
struct clk *clk;
clk = clk_register(NULL, &gate->hw);

if (IS_ERR(clk)) {
    pr_err("Failed to register uart_clk!\n");
    return PTR_ERR(clk);
}
```

#### 4. الـ framework بيعمل ايه جوه؟

```c
// drivers/clk/clk.c
struct clk *clk_register(struct device *dev, struct clk_hw *hw) {
    struct clk_core *core;

    // 1. اعمل clk_core جديد
    core = kzalloc(sizeof(*core), GFP_KERNEL);

    // 2. انسخ البيانات
    core->name = kstrdup(hw->init->name);
    core->ops = hw->init->ops;
    core->hw = hw;
    core->num_parents = hw->init->num_parents;

    // 3. ربط الـ hw بالـ core
    hw->core = core;

    // 4. دوّر على الـ parents
    for (i = 0; i < core->num_parents; i++) {
        core->parents[i] = clk_core_lookup(parent_names[i]);
    }

    // 5. ضيفه للـ clock tree
    clk_core_populate_parent_map(core);

    // 6. احسب السرعة الابتدائية
    if (core->ops->recalc_rate)
        core->rate = core->ops->recalc_rate(core->hw, parent_rate);

    return clk;
}
```

---

## 7. الـ Enable/Disable Reference Counting

### المشكلة:
لو **اتنين drivers** بيستخدموا نفس الـ clock، إيه اللي يحصل؟

```c
// Driver A
clk_enable(usb_clk);  // enable_count = 1

// Driver B
clk_enable(usb_clk);  // enable_count = 2

// Driver A خلص شغله
clk_disable(usb_clk); // enable_count = 1 (لسه شغال!)

// Driver B خلص شغله
clk_disable(usb_clk); // enable_count = 0 (دلوقتي يقفل)
```

### الكود:
```c
int clk_enable(struct clk *clk) {
    unsigned long flags;

    spin_lock_irqsave(&enable_lock, flags);

    if (clk->core->enable_count == 0) {
        // أول مرة نشغله
        clk->core->ops->enable(clk->core->hw);
    }

    clk->core->enable_count++;  // زوّد العداد

    spin_unlock_irqrestore(&enable_lock, flags);

    return 0;
}

void clk_disable(struct clk *clk) {
    unsigned long flags;

    spin_lock_irqsave(&enable_lock, flags);

    if (--clk->core->enable_count == 0) {
        // آخر واحد استخدمه، دلوقتي قفله
        clk->core->ops->disable(clk->core->hw);
    }

    spin_unlock_irqrestore(&enable_lock, flags);
}
```

---

## 8. الـ Rate Change Notification (إشعارات تغيير السرعة)

### ليه محتاجينها؟

بعض الـ drivers عايزة **تعرف** لما السرعة هتتغير!

**مثال:** الـ UART driver لازم يحسب الـ baud rate تاني لو الـ clock اتغير.

```c
struct notifier_block uart_clk_nb = {
    .notifier_call = uart_clk_notifier,
};

// سجّل نفسك عشان تعرف لما الـ clock يتغير
clk_notifier_register(uart_clk, &uart_clk_nb);

// الـ callback
static int uart_clk_notifier(struct notifier_block *nb,
                              unsigned long event, void *data) {
    struct clk_notifier_data *ndata = data;

    switch (event) {
    case PRE_RATE_CHANGE:
        // هيتغير دلوقتي!
        pr_info("Clock changing: %lu -> %lu\n",
                ndata->old_rate, ndata->new_rate);
        // استعد للتغيير
        break;

    case POST_RATE_CHANGE:
        // اتغير فعلاً!
        uart_update_baud_rate(ndata->new_rate);
        break;

    case ABORT_RATE_CHANGE:
        // التغيير اتلغى!
        break;
    }

    return NOTIFY_OK;
}
```

---

## 9. الـ Debugfs Interface (واجهة التصحيح)

### إزاي تشوف الـ clock tree؟

```bash
# اعرض كل الـ clocks
cat /sys/kernel/debug/clk/clk_summary

# Output:
   clock                         enable_cnt  prepare_cnt  rate
------------------------------------------------------------------------
 osc24M                          2           2            24000000
    pll1                         1           1            800000000
       cpu_clk                   1           1            400000000
       ahb_clk                   3           3            200000000
          apb_clk                5           5            100000000
             uart0               1           1            100000000
             uart1               0           0            100000000
             i2c0                1           1            100000000
    pll2                         1           1            600000000
       usb_clk                   1           1            48000000
```

### شوف تفاصيل clock معين:
```bash
cd /sys/kernel/debug/clk/uart0

cat clk_rate
100000000

cat clk_enable_count
1

cat clk_prepare_count
1

cat clk_parent
apb_clk
```

---

## 10. سيناريو واقعي كامل: USB Clock

### المطلوب:
عايزين نعمل USB clock بسرعة **48 MHz** من crystal **24 MHz**.

### الحل:

#### 1. عرّف الـ clocks:
```c
// Crystal الثابت
clk_register_fixed_rate(NULL, "osc24M", NULL, 0, 24000000);

// PLL بيضاعف × 20
// 24 MHz × 20 = 480 MHz
struct clk_pll *pll;
pll->m = 20;  // multiplier
pll->n = 1;   // divider
clk_register(NULL, "pll_usb", &pll->hw);

// Divider بيقسم ÷ 10
// 480 MHz / 10 = 48 MHz
struct clk_divider *div;
div->div = 10;
clk_register(NULL, "usb_pre_clk", &div->hw);

// Gate للتحكم
struct clk_gate *gate;
gate->bit_idx = 23;
clk_register(NULL, "usb_clk", &gate->hw);
```

#### 2. الـ Tree النهائي:
```
osc24M (24 MHz)
    |
pll_usb (480 MHz) = 24 × 20
    |
usb_pre_clk (48 MHz) = 480 / 10
    |
usb_clk (48 MHz) - with gate
```

#### 3. الـ USB driver يستخدمه:
```c
struct clk *clk;

clk = clk_get(dev, "usb_clk");
clk_prepare_enable(clk);  // شغّل الـ USB!

// استخدم الـ USB...

clk_disable_unprepare(clk);  // خلصنا، قفل
```

---

## 11. الـ CLK_SET_RATE_PARENT Flag

### المشكلة:
لو عندك:
```
PLL (800 MHz)
    |
Divider (/2)
    |
UART (400 MHz)
```

وعايز UART بسرعة **115200 baud** اللي محتاج **1.8432 MHz**.

لو قلت:
```c
clk_set_rate(uart_clk, 1843200);
```

الـ **divider** مش هيقدر يطلّع السرعة دي من 800 MHz!

### الحل:

استخدم **CLK_SET_RATE_PARENT**:

```c
struct clk_init_data init = {
    .name = "uart_clk",
    .flags = CLK_SET_RATE_PARENT,  // غيّر الأب لو محتاج!
};
```

دلوقتي لما تقول `clk_set_rate(uart_clk, 1843200)`:
1. الـ divider يحاول يطلّعها من 800 MHz → **مش قادر**
2. يقول للـ PLL: "يا أبويا، غيّر سرعتك!"
3. الـ PLL يغيّر لـ frequency مناسب
4. الـ divider يقسم ويطلّع الـ 1.8432 MHz

---

## 12. الـ Assigned Clocks (التعيين من Device Tree)

في الـ **Device Tree** تقدر تحدد السرعة مباشرة:

```dts
uart0: serial@12340000 {
    compatible = "vendor,uart";
    clocks = <&cru UART0_CLK>;
    clock-names = "uart";

    assigned-clocks = <&cru UART0_CLK>;
    assigned-clock-rates = <1843200>;     // السرعة المطلوبة
    assigned-clock-parents = <&cru PLL2>; // الأب المطلوب
};
```

الـ kernel لما يشوف ده، **تلقائياً** هيعمل:
```c
clk_set_parent(uart0_clk, pll2);
clk_set_rate(uart0_clk, 1843200);
```

---

## الخلاصة الشاملة 🎓

### الـ Common Clock Framework هو:

1. **نظام شجري** - كل clock ليه parent
2. **Reference counting** - عشان ما يقفلش clock حد تاني بيستخدمه
3. **Rate propagation** - لما تغير الأب، الأبناء تحسب نفسها تاني
4. **Multiple types** - fixed, gate, divider, mux, PLL
5. **Two-phase** - prepare (slow) + enable (fast)
6. **Notifications** - عشان الـ drivers تعرف لما حاجة تتغير
7. **Debugfs** - عشان تشوف إيه اللي بيحصل
8. **Device Tree integration** - تقدر تحدد كل حاجة من الـ DTS

**المعادلة السحرية:**
```
Clock Framework = Hardware abstraction + Power management + Rate calculation + Parent management
```

كل ده عشان **توفر طاقة** ✅ و **تدي كل peripheral السرعة اللي محتاجها** ✅!


---
# شرح تفصيلي لـ Clock API Functions 🔧

خليني أقسم الـ functions لمجموعات وأشرح كل واحدة بالتفصيل:

---

## 1️⃣ الـ Notifier Functions (الإشعارات) 📢

### إيه دي؟
لما الـ clock هيتغير، بعض الـ drivers عايزة **تعرف** عشان تستعد!

```c
int clk_notifier_register(struct clk *clk, struct notifier_block *nb);
int clk_notifier_unregister(struct clk *clk, struct notifier_block *nb);
```

### مثال من الحياة:
تخيل إن في **محطة كهربا** (clock)، وأنت عايز **إنذار** لما الكهربا هتتغير.

```c
// سجّل نفسك عشان تعرف لما الـ clock يتغير
static int my_clk_notifier(struct notifier_block *nb,
                           unsigned long event, void *data)
{
    struct clk_notifier_data *ndata = data;

    switch (event) {
    case PRE_RATE_CHANGE:
        // هيتغير دلوقتي! استعد!
        pr_info("Clock هيتغير من %lu لـ %lu\n",
                ndata->old_rate, ndata->new_rate);
        // وقّف الشغل اللي ممكن يتأثر
        my_device_pause();
        break;

    case POST_RATE_CHANGE:
        // خلاص اتغير! حدّث نفسك
        my_device_update_speed(ndata->new_rate);
        my_device_resume();
        break;

    case ABORT_RATE_CHANGE:
        // التغيير اتلغى! ارجع زي ما كنت
        my_device_resume();
        break;
    }

    return NOTIFY_OK;
}

struct notifier_block my_nb = {
    .notifier_call = my_clk_notifier,
};

// سجّل
clk_notifier_register(my_clk, &my_nb);

// لما تخلص
clk_notifier_unregister(my_clk, &my_nb);
```

### الـ devm variant:
```c
devm_clk_notifier_register(dev, clk, &nb);
// لما الـ device يتشال، يلغي التسجيل تلقائياً!
```

---

## 2️⃣ Clock Properties (خصائص متقدمة)

### أ) **Accuracy** (الدقة)
```c
long clk_get_accuracy(struct clk *clk);
```

**بيقيس** قد إيه الـ clock دقيق! بوحدة **ppb** (parts per billion).

```c
long accuracy = clk_get_accuracy(my_clk);
// لو طلع 0 = مثالي 100%
// لو طلع 100 = فيه غلطة 100 جزء من مليار
```

**مثال:** الـ crystal oscillator الرخيص ممكن يكون **±50 ppm** (50 جزء من مليون).

---

### ب) **Phase** (إزاحة الطور)
```c
int clk_set_phase(struct clk *clk, int degrees);
int clk_get_phase(struct clk *clk);
```

**إيه ده؟** بيزيح الإشارة **بالدرجات** (0-360°).

**ليه محتاجينه؟** في الـ **SD card** و **DDR memory**، لازم الـ clock يكون **متزامن** مع الـ data.

```c
// خلّي الـ clock متأخر 90 درجة
clk_set_phase(sdmmc_clk, 90);

// شوف الإزاحة الحالية
int phase = clk_get_phase(sdmmc_clk);
pr_info("Phase = %d degrees\n", phase);
```

**تخيلها كده:**
```
Clock A: ___/‾‾‾\___/‾‾‾\___
Clock B: _/‾‾‾\___/‾‾‾\___/   (shifted 45°)
```

---

### ج) **Duty Cycle** (نسبة التشغيل)
```c
int clk_set_duty_cycle(struct clk *clk, unsigned int num, unsigned int den);
int clk_get_scaled_duty_cycle(struct clk *clk, unsigned int scale);
```

**إيه ده؟** النسبة بين الـ **high** و **low** في الإشارة.

```c
// عايز 60% high, 40% low
clk_set_duty_cycle(my_clk, 60, 100);

// أو عايز 1:3 (25% high)
clk_set_duty_cycle(my_clk, 1, 4);
```

**شكلها:**
```
50% duty cycle (normal):
‾‾‾‾____‾‾‾‾____‾‾‾‾____

75% duty cycle:
‾‾‾‾‾‾‾_‾‾‾‾‾‾‾_‾‾‾‾‾‾‾_

25% duty cycle:
‾‾__‾‾__‾‾__
```

---

## 3️⃣ Rate Exclusivity (الحصرية) 🔐

### إيه المشكلة؟
لو driver عايز **يضمن** إن **حد تاني** ما يغيرش السرعة عليه!

```c
int clk_rate_exclusive_get(struct clk *clk);
void clk_rate_exclusive_put(struct clk *clk);
```

### السيناريو:
```c
// Camera driver محتاج السرعة تفضل ثابتة!
clk_rate_exclusive_get(camera_clk);
clk_set_rate(camera_clk, 96000000);

// دلوقتي لو driver تاني حاول يغير السرعة
// هيفشل! ❌

// استخدم الكاميرا...
camera_capture_photo();

// خلصت، سيّب الحصرية
clk_rate_exclusive_put(camera_clk);
// دلوقتي ناس تانية تقدر تغير ✅
```

**Managed variant:**
```c
devm_clk_rate_exclusive_get(dev, clk);
// لما الـ device يتشال، يسيّب الحصرية تلقائياً
```

---

## 4️⃣ Prepare/Unprepare (التحضير) ⏱️

### الفرق بين Prepare و Enable:

| Operation | Speed | Can Sleep? | Use Case |
|-----------|-------|------------|----------|
| **prepare** | بطيء (ms) | ✅ ينفع ينام | تشغيل PLL |
| **enable** | سريع جداً (ns) | ❌ ممنوع ينام | فتح gate |

```c
int clk_prepare(struct clk *clk);
void clk_unprepare(struct clk *clk);
```

### ليه اتنين مش واحد؟

**Prepare:** بيجهز الـ clock (مثلاً يسخن الـ PLL).
**Enable:** بيفتح الـ gate بسرعة.

```c
// في process context (يسمح بالنوم)
clk_prepare(usb_clk);        // ممكن ياخد 100 microseconds
                             // (بيسخن الـ PLL)

// في interrupt context (ممنوع النوم!)
clk_enable(usb_clk);         // أسرع من 1 microsecond
                             // (بس بيفتح gate)

// استخدم الـ USB...

clk_disable(usb_clk);        // سريع
clk_unprepare(usb_clk);      // بطيء (ممكن ينام)
```

### الـ Shortcut الشهير:
```c
// بدل ما تعمل prepare ثم enable
int clk_prepare_enable(struct clk *clk);

// بدل ما تعمل disable ثم unprepare
void clk_disable_unprepare(struct clk *clk);
```

### Bulk operations:
```c
// لو عندك مجموعة clocks
struct clk_bulk_data clks[] = {
    { .id = "usb" },
    { .id = "dma" },
    { .id = "uart" },
};

clk_bulk_prepare(3, clks);    // حضّرهم كلهم
clk_bulk_enable(3, clks);     // شغّلهم كلهم
```

---

## 5️⃣ Get/Put Functions (الحصول على Clock) 🎯

### الأساسيات:
```c
struct clk *clk_get(struct device *dev, const char *id);
void clk_put(struct clk *clk);
```

### إزاي بيشتغل؟

```c
// في الـ driver
static int my_probe(struct platform_device *pdev)
{
    struct clk *clk;

    // جيب الـ clock بالاسم
    clk = clk_get(&pdev->dev, "uart");
    if (IS_ERR(clk)) {
        dev_err(&pdev->dev, "Failed to get clock!\n");
        return PTR_ERR(clk);
    }

    // استخدمه...
    clk_prepare_enable(clk);

    return 0;
}

static int my_remove(struct platform_device *pdev)
{
    // ارجع الـ clock
    clk_put(clk);
    return 0;
}
```

---

### الـ devm Variants (الأذكى!) 🧠

**المشكلة:** لو نسيت `clk_put`، هيحصل **memory leak**!

**الحل:** استخدم **devm**!

```c
struct clk *devm_clk_get(struct device *dev, const char *id);
// لما الـ device يتشال، يعمل clk_put تلقائياً! ✨
```

#### Family كاملة:

```c
// 1. عادي (مش prepared ولا enabled)
struct clk *devm_clk_get(dev, "uart");

// 2. مع prepare
struct clk *devm_clk_get_prepared(dev, "uart");
// = devm_clk_get + clk_prepare

// 3. مع prepare + enable
struct clk *devm_clk_get_enabled(dev, "uart");
// = devm_clk_get + clk_prepare + clk_enable

// 4. Optional (لو مش موجود، يرجع NULL مش error)
struct clk *devm_clk_get_optional(dev, "uart");

// 5. Optional + prepared
struct clk *devm_clk_get_optional_prepared(dev, "uart");

// 6. Optional + enabled
struct clk *devm_clk_get_optional_enabled(dev, "uart");

// 7. Optional + enabled + set rate!
struct clk *devm_clk_get_optional_enabled_with_rate(dev, "uart", 48000000);
```

---

### Bulk Get (جيب مجموعة مرة واحدة):

```c
struct clk_bulk_data clks[] = {
    { .id = "ahb" },
    { .id = "apb" },
    { .id = "cpu" },
};

// جيبهم كلهم دفعة واحدة
int ret = clk_bulk_get(dev, ARRAY_SIZE(clks), clks);

// أو الـ managed version
devm_clk_bulk_get(dev, ARRAY_SIZE(clks), clks);

// شغّلهم كلهم
clk_bulk_prepare_enable(ARRAY_SIZE(clks), clks);
```

---

### Get All Clocks:
```c
struct clk_bulk_data *clks;
int num_clks;

// جيب **كل** الـ clocks اللي في الـ device tree
num_clks = clk_bulk_get_all(dev, &clks);

pr_info("Found %d clocks!\n", num_clks);
```

---

## 6️⃣ Enable/Disable (التشغيل/الإيقاف) 🔌

```c
int clk_enable(struct clk *clk);
void clk_disable(struct clk *clk);
```

### القواعد المهمة:

✅ **يُسمح** باستخدامهم في **atomic context**
✅ **سريعين جداً** (nanoseconds)
❌ **ممنوع ينام** داخلهم

```c
// في interrupt handler
static irqreturn_t my_interrupt(int irq, void *data)
{
    // ده safe! ✅
    clk_enable(dma_clk);

    // استخدم الـ DMA...

    clk_disable(dma_clk);
    return IRQ_HANDLED;
}
```

### Reference Counting مهم جداً! 🔢

```c
clk_enable(clk);   // count = 1 ✅ شغّل
clk_enable(clk);   // count = 2 (لسه شغال)
clk_enable(clk);   // count = 3 (لسه شغال)

clk_disable(clk);  // count = 2 (لسه شغال)
clk_disable(clk);  // count = 1 (لسه شغال)
clk_disable(clk);  // count = 0 ❌ دلوقتي قفل!
```

**ليه؟** عشان لو **3 drivers** بيستخدموا نفس الـ clock، ما يقفلوش على بعض!

---

## 7️⃣ Rate Control (التحكم في السرعة) 🏎️

### أ) Get Rate (اعرف السرعة):
```c
unsigned long clk_get_rate(struct clk *clk);
```

```c
unsigned long rate = clk_get_rate(cpu_clk);
pr_info("CPU running at %lu Hz (%lu MHz)\n",
        rate, rate / 1000000);
```

---

### ب) Set Rate (غيّر السرعة):
```c
int clk_set_rate(struct clk *clk, unsigned long rate);
```

```c
// عايز USB يشتغل على 48 MHz
int ret = clk_set_rate(usb_clk, 48000000);
if (ret) {
    pr_err("Failed to set rate!\n");
    return ret;
}

// تأكد
unsigned long actual = clk_get_rate(usb_clk);
pr_info("USB clock = %lu Hz\n", actual);
```

---

### ج) Round Rate (إيه أقرب سرعة ممكنة؟):
```c
long clk_round_rate(struct clk *clk, unsigned long rate);
```

**ليه محتاجينه؟** لأن مش كل سرعة ممكنة!

```c
// عايز 133 MHz
long rounded = clk_round_rate(my_clk, 133000000);

if (rounded != 133000000) {
    pr_info("Can't do 133 MHz, closest is %ld Hz\n", rounded);
    // Maybe: 125000000 Hz (125 MHz)
}

// خليها أقرب حاجة
clk_set_rate(my_clk, rounded);
```

---

### د) Set Range (حدد نطاق):
```c
int clk_set_rate_range(struct clk *clk, unsigned long min, unsigned long max);
int clk_set_min_rate(struct clk *clk, unsigned long rate);
int clk_set_max_rate(struct clk *clk, unsigned long rate);
```

```c
// الـ SD card محتاج بين 400 KHz و 50 MHz
clk_set_rate_range(sdmmc_clk, 400000, 50000000);

// أو حدد minimum بس
clk_set_min_rate(cpu_clk, 400000000); // مش أقل من 400 MHz

// أو maximum بس
clk_set_max_rate(cpu_clk, 1200000000); // مش أعلى من 1.2 GHz
```

---

### هـ) Set Rate Exclusive:
```c
int clk_set_rate_exclusive(struct clk *clk, unsigned long rate);
```

**ده اختصار لـ:**
```c
clk_rate_exclusive_get(clk);
clk_set_rate(clk, rate);
// لازم تعمل clk_rate_exclusive_put() بعدين
```

---

## 8️⃣ Parent Control (التحكم في الأب) 👨‍👦

### Get Parent (مين الأب؟):
```c
struct clk *clk_get_parent(struct clk *clk);
```

```c
struct clk *parent = clk_get_parent(usb_clk);
pr_info("USB clock parent: %s\n", __clk_get_name(parent));
// Output: USB clock parent: pll2
```

---

### Set Parent (غيّر الأب):
```c
int clk_set_parent(struct clk *clk, struct clk *parent);
```

```c
// عندنا مصدرين محتملين
struct clk *pll1 = clk_get(dev, "pll1");  // 800 MHz
struct clk *pll2 = clk_get(dev, "pll2");  // 600 MHz

// خلّي USB ييجي من PLL2
clk_set_parent(usb_clk, pll2);
```

---

### Check if Possible Parent:
```c
bool clk_has_parent(const struct clk *clk, const struct clk *parent);
```

```c
if (clk_has_parent(usb_clk, pll1)) {
    pr_info("PLL1 is a valid parent for USB\n");
    clk_set_parent(usb_clk, pll1);
} else {
    pr_err("Can't use PLL1 for USB!\n");
}
```

---

## 9️⃣ Device Tree Integration 🌳

### of_clk_get Functions:

```c
struct clk *of_clk_get(struct device_node *np, int index);
struct clk *of_clk_get_by_name(struct device_node *np, const char *name);
```

**من الـ Device Tree:**
```dts
uart0: serial@12340000 {
    compatible = "vendor,uart";
    clocks = <&cru UART_CLK>, <&cru APB_CLK>;
    clock-names = "uart", "apb";
};
```

**في الـ Driver:**
```c
struct clk *uart_clk, *apb_clk;

// طريقة 1: بالـ index
uart_clk = of_clk_get(np, 0);  // أول clock
apb_clk = of_clk_get(np, 1);   // تاني clock

// طريقة 2: بالاسم (أحسن!)
uart_clk = of_clk_get_by_name(np, "uart");
apb_clk = of_clk_get_by_name(np, "apb");
```

---

## 🔟 Context Save/Restore (للـ Suspend/Resume)

```c
int clk_save_context(void);
void clk_restore_context(void);
```

**ليه؟** لما النظام يدخل **deep sleep**، الـ registers بتتمسح!

```c
// في suspend
static int my_suspend(struct device *dev)
{
    // احفظ حالة كل الـ clocks
    clk_save_context();

    // Enter deep sleep...

    return 0;
}

// في resume
static int my_resume(struct device *dev)
{
    // ارجع حالة الـ clocks زي ما كانت
    clk_restore_context();

    return 0;
}
```

---

## 1️⃣1️⃣ Helper Functions (مساعدين)

### clk_is_match:
```c
bool clk_is_match(const struct clk *p, const struct clk *q);
```

**بيشوف** لو الـ 2 clocks دول نفس الـ hardware ولا لأ.

```c
struct clk *clk1 = clk_get(dev, "uart");
struct clk *clk2 = clk_get(dev, "uart");

if (clk_is_match(clk1, clk2)) {
    pr_info("Same hardware clock!\n");
}
```

---

### clk_drop_range:
```c
int clk_drop_range(struct clk *clk);
```

**بيلغي** أي range حطيته قبل كده.

```c
// كنت حاطط range
clk_set_rate_range(clk, 100000000, 200000000);

// دلوقتي عايز ألغيه
clk_drop_range(clk);
// = clk_set_rate_range(clk, 0, ULONG_MAX);
```

---

### clk_get_sys:
```c
struct clk *clk_get_sys(const char *dev_id, const char *con_id);
```

زي `clk_get` بس بياخد **اسم الـ device** مش الـ **device** نفسه.

```c
// بدل
struct clk *clk = clk_get(dev, "uart");

// تقدر
struct clk *clk = clk_get_sys("12340000.serial", "uart");
```

---

## مثال كامل واقعي: UART Driver 📝

```c
struct my_uart {
    void __iomem *base;
    struct clk *clk;
    unsigned long baud_rate;
};

static int my_uart_probe(struct platform_device *pdev)
{
    struct my_uart *uart;
    int ret;

    uart = devm_kzalloc(&pdev->dev, sizeof(*uart), GFP_KERNEL);

    // 1. جيب الـ clock (managed)
    uart->clk = devm_clk_get_enabled(&pdev->dev, "uart");
    if (IS_ERR(uart->clk))
        return PTR_ERR(uart->clk);

    // 2. اعرف السرعة الحالية
    unsigned long clk_rate = clk_get_rate(uart->clk);
    pr_info("UART clock = %lu Hz\n", clk_rate);

    // 3. سجّل notifier عشان تعرف لو السرعة اتغيرت
    uart->nb.notifier_call = uart_clk_notifier;
    devm_clk_notifier_register(&pdev->dev, uart->clk, &uart->nb);

    // 4. اضبط الـ baud rate
    uart_set_baud_rate(uart, 115200);

    return 0;
}

static int uart_clk_notifier(struct notifier_block *nb,
                             unsigned long event, void *data)
{
    struct my_uart *uart = container_of(nb, struct my_uart, nb);
    struct clk_notifier_data *ndata = data;

    if (event == POST_RATE_CHANGE) {
        // السرعة اتغيرت! احسب الـ baud rate تاني
        uart_recalc_baud_rate(uart, ndata->new_rate);
    }

    return NOTIFY_OK;
}

// لما الـ device يتشال
// كل حاجة هتتعمل automatically لأننا استخدمنا devm_*
```

---

## الخلاصة النهائية 🎓

### الـ Clock API مقسومة لـ:

1. **Lifecycle**: get/put, prepare/unprepare, enable/disable
2. **Configuration**: set_rate, set_parent, set_phase
3. **Query**: get_rate, get_parent, get_accuracy
4. **Constraints**: set_range, rate_exclusive
5. **Notifications**: register/unregister notifier
6. **Bulk operations**: عشان تعمل operation على مجموعة clocks
7. **Managed (devm)**: automatic cleanup

### القواعد الذهبية:

✅ استخدم **devm_*** variants عشان تتجنب leaks
✅ **prepare** في process context، **enable** في atomic
✅ بلّش الـ **enable/disable** calls
✅ استخدم **notifiers** لو محتاج تعرف بتغييرات
✅ استخدم **rate_exclusive** لو عايز تضمن stability

**الـ typical flow:**
```c
clk = devm_clk_get_enabled(dev, "my_clk");  // Get + prepare + enable
// Use the clock...
// (automatic cleanup when device removed)
```

كل دي الأدوات اللي محتاجها عشان **تتحكم في الـ clocks** في النظام بكفاءة! 💪

---

# Clock Subsystem Debugging Cheatsheet 🔧🐛

## 📋 Table of Contents
1. [Kernel Config & Build](#kernel-config)
2. [DebugFS Interface](#debugfs)
3. [SysFS Interface](#sysfs)
4. [Tracing & Logging](#tracing)
5. [Hardware Debugging](#hardware)
6. [Common Problems](#problems)
7. [Tools & Scripts](#tools)

---

## 1️⃣ Kernel Config & Build {#kernel-config}

### Enable Debugging Options في الـ `.config`:

```bash
# Essential options
CONFIG_COMMON_CLK=y
CONFIG_DEBUG_FS=y              # لازم للـ debugfs
CONFIG_COMMON_CLK_DEBUG=y      # معلومات أكتر في debugfs

# Advanced debugging
CONFIG_DEBUG_KERNEL=y
CONFIG_DYNAMIC_DEBUG=y         # عشان dynamic pr_debug
CONFIG_FTRACE=y                # للـ function tracing
CONFIG_KPROBES=y               # للـ kprobe tracing
CONFIG_PRINTK_TIME=y           # timestamps في dmesg

# Clock-specific debugging
CONFIG_CLK_SUNXI=y             # مثال: لو بتشتغل على Allwinner
CONFIG_CLK_ROCKCHIP=y          # مثال: لو بتشتغل على Rockchip
```

### Enable via menuconfig:

```bash
make menuconfig

# Navigate to:
Device Drivers --->
  Common Clock Framework --->
    [*] DebugFS representation of clock tree
    [*] Enable clock framework debugging

Kernel hacking --->
  [*] Debug Filesystem
  [*] Tracers --->
    [*] Kernel Function Tracer
```

### Build مع Debug Symbols:

```bash
# في الـ Makefile أو .config
CONFIG_DEBUG_INFO=y
CFLAGS += -g -O0    # disable optimization للـ debugging

# Build
make -j$(nproc) ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-
```

---

## 2️⃣ DebugFS Interface {#debugfs}

### Mount DebugFS:

```bash
# Check if mounted
mount | grep debugfs

# Mount manually
mount -t debugfs none /sys/kernel/debug

# Or add to /etc/fstab
echo "debugfs /sys/kernel/debug debugfs defaults 0 0" >> /etc/fstab
```

---

### 🌳 Clock Tree Summary

```bash
# عرض كل الـ clocks في النظام
cat /sys/kernel/debug/clk/clk_summary

# Output example:
#    clock                         enable_cnt  prepare_cnt  rate        accuracy phase
# ----------------------------------------------------------------------------------------
#  xin24m                          3           3            24000000    0        0
#     cpll                         1           1            1200000000  0        0
#        aclk_vop                  1           1            400000000   0        0
#        hclk_vop                  1           1            200000000   0        0
#     gpll                         2           2            1188000000  0        0
#        aclk_bus                  0           0            396000000   0        0
```

**Understanding the columns:**
- `enable_cnt`: كام مرة اتعمل enable (reference count)
- `prepare_cnt`: كام مرة اتعمل prepare
- `rate`: السرعة الحالية بالـ Hz
- `accuracy`: الدقة بالـ ppb
- `phase`: phase shift بالدرجات

---

### 🔍 Individual Clock Details

```bash
# Navigate to specific clock
cd /sys/kernel/debug/clk/uart0

# Available files:
ls -la
# clk_accuracy
# clk_duty_cycle
# clk_enable_count
# clk_flags
# clk_max_rate
# clk_min_rate
# clk_notifier_count
# clk_parent
# clk_phase
# clk_possible_parents
# clk_prepare_count
# clk_rate

# Read current rate
cat clk_rate
# 100000000

# Read parent
cat clk_parent
# apb_clk

# Read possible parents (if mux)
cat clk_possible_parents
# pll1 pll2 xin24m

# Read enable count
cat clk_enable_count
# 2

# Read flags
cat clk_flags
# 0x00000006  # CLK_SET_RATE_PARENT | CLK_IGNORE_UNUSED
```

---

### 📊 Clock Flags Decoder

```c
// من include/linux/clk-provider.h
#define CLK_SET_RATE_GATE       BIT(0)  // 0x01 - لازم يكون disabled عشان تغير rate
#define CLK_SET_PARENT_GATE     BIT(1)  // 0x02 - لازم يكون disabled عشان تغير parent
#define CLK_SET_RATE_PARENT     BIT(2)  // 0x04 - لو غيرت rate، غير الأب كمان
#define CLK_IGNORE_UNUSED       BIT(3)  // 0x08 - ما تقفلوش لو مش مستخدم
#define CLK_GET_RATE_NOCACHE    BIT(6)  // 0x40 - دايماً اقرا من hardware
#define CLK_SET_RATE_NO_REPARENT BIT(7) // 0x80 - ما تغيرش parent لما تغير rate
```

**Script لفك الـ flags:**
```bash
#!/bin/bash
FLAGS=$(cat /sys/kernel/debug/clk/uart0/clk_flags)
HEX_FLAGS=$((FLAGS))

echo "Flags: 0x$(printf '%08x' $HEX_FLAGS)"
[ $(($HEX_FLAGS & 0x01)) -ne 0 ] && echo "  - CLK_SET_RATE_GATE"
[ $(($HEX_FLAGS & 0x02)) -ne 0 ] && echo "  - CLK_SET_PARENT_GATE"
[ $(($HEX_FLAGS & 0x04)) -ne 0 ] && echo "  - CLK_SET_RATE_PARENT"
[ $(($HEX_FLAGS & 0x08)) -ne 0 ] && echo "  - CLK_IGNORE_UNUSED"
[ $(($HEX_FLAGS & 0x40)) -ne 0 ] && echo "  - CLK_GET_RATE_NOCACHE"
```

---

### 🔬 Clock Tree Visualization

```bash
# Print tree structure
cd /sys/kernel/debug/clk
tree -L 3

# Or custom script
#!/bin/bash
function print_clk_tree() {
    local CLK=$1
    local INDENT=$2

    echo "${INDENT}${CLK}"

    for CHILD in /sys/kernel/debug/clk/*/clk_parent; do
        PARENT=$(cat $CHILD)
        if [ "$PARENT" = "$CLK" ]; then
            CHILD_NAME=$(dirname $CHILD | xargs basename)
            print_clk_tree $CHILD_NAME "${INDENT}  "
        fi
    done
}

# Start from root clocks
for CLK in /sys/kernel/debug/clk/*; do
    NAME=$(basename $CLK)
    PARENT=$(cat $CLK/clk_parent 2>/dev/null)
    [ -z "$PARENT" ] && print_clk_tree $NAME ""
done
```

---

## 3️⃣ SysFS Interface {#sysfs}

### Device-Specific Clocks

```bash
# لكل device في النظام
cd /sys/devices/platform/

# Example: UART device
cd 12340000.serial

# Clock info
cat clk/clk_rate
cat clk/clk_enable_count

# أو
cd /sys/class/tty/ttyS0/device/clk/
```

---

### Assigned Clocks (من Device Tree)

```bash
# شوف الـ clocks المحددة في DT
cat /proc/device-tree/soc/uart@12340000/assigned-clocks
cat /proc/device-tree/soc/uart@12340000/assigned-clock-rates
```

---

## 4️⃣ Tracing & Logging {#tracing}

### A) Dynamic Debug (pr_debug)

```bash
# Enable all clock framework debug messages
echo "file drivers/clk/clk.c +p" > /sys/kernel/debug/dynamic_debug/control

# Enable for specific function
echo "func clk_set_rate +p" > /sys/kernel/debug/dynamic_debug/control

# Enable for specific module
echo "module clk_rockchip +p" > /sys/kernel/debug/dynamic_debug/control

# Disable
echo "file drivers/clk/clk.c -p" > /sys/kernel/debug/dynamic_debug/control

# View current settings
cat /sys/kernel/debug/dynamic_debug/control | grep clk
```

---

### B) Kernel Messages (dmesg)

```bash
# Watch clock messages in real-time
dmesg -w | grep -i "clk\|clock"

# Look for errors
dmesg | grep -E "clk.*error|clk.*fail|clk.*warn"

# Common patterns
dmesg | grep "clk_set_rate"
dmesg | grep "clk_enable"
dmesg | grep "CLK:"

# With timestamps
dmesg -T | grep clk
```

---

### C) Ftrace (Function Tracer)

#### Setup:
```bash
cd /sys/kernel/debug/tracing

# Enable function tracer
echo function > current_tracer

# Filter only clock functions
echo '*clk*' > set_ftrace_filter

# Or specific functions
echo clk_enable >> set_ftrace_filter
echo clk_disable >> set_ftrace_filter
echo clk_set_rate >> set_ftrace_filter

# Start tracing
echo 1 > tracing_on

# Do your operation...
# (e.g., modprobe driver, or cat /dev/ttyS0)

# Stop tracing
echo 0 > tracing_on

# View trace
cat trace | less

# Clear trace
echo > trace
```

#### Example Trace Output:
```
# tracer: function
#
#           TASK-PID   CPU#   TIMESTAMP  FUNCTION
#              | |       |       |         |
     kworker/0:1-123   [000]   100.123456: clk_prepare <-uart_probe
     kworker/0:1-123   [000]   100.123789: clk_enable <-uart_probe
     kworker/0:1-123   [000]   100.124001: clk_set_rate <-uart_set_baud
```

---

### D) Trace Events

```bash
cd /sys/kernel/debug/tracing

# List available clock events
ls events/clk/

# Example events:
# - clk_enable
# - clk_disable
# - clk_set_rate
# - clk_set_parent
# - clk_prepare
# - clk_unprepare

# Enable all clock events
echo 1 > events/clk/enable

# Or enable specific event
echo 1 > events/clk/clk_set_rate/enable

# Start tracing
echo 1 > tracing_on

# View trace
cat trace_pipe
# Or
cat trace

# Filter by specific clock
echo 'name == "uart0"' > events/clk/clk_set_rate/filter
```

#### Example Event Output:
```
<idle>-0     [000] d... 104.567890: clk_set_rate: uart0 100000000 -> 115200
<idle>-0     [000] d... 104.567901: clk_enable: uart0
```

---

### E) Kprobe Dynamic Tracing

```bash
cd /sys/kernel/debug/tracing

# Add probe on clk_set_rate function
echo 'p:myprobe clk_set_rate clk=%di rate=%si' > kprobe_events
# %di = first argument (RDI register)
# %si = second argument (RSI register)

# Enable the probe
echo 1 > events/kprobes/myprobe/enable

# View trace
cat trace_pipe

# Remove probe
echo 0 > events/kprobes/myprobe/enable
echo '-:myprobe' > kprobe_events
```

---

## 5️⃣ Hardware-Level Debugging {#hardware}

### A) Register Dumps

#### في الـ Driver Code:
```c
// Add to your clock driver
static void dump_clock_registers(void __iomem *base)
{
    pr_info("=== Clock Registers Dump ===\n");
    pr_info("CRU_CLKSEL_CON0  = 0x%08x\n", readl(base + 0x0000));
    pr_info("CRU_CLKSEL_CON1  = 0x%08x\n", readl(base + 0x0004));
    pr_info("CRU_CLKGATE_CON0 = 0x%08x\n", readl(base + 0x0200));
    pr_info("CRU_CLKGATE_CON1 = 0x%08x\n", readl(base + 0x0204));
    pr_info("CRU_PLL_CON0     = 0x%08x\n", readl(base + 0x0400));
    pr_info("============================\n");
}
```

---

#### Via devmem (من userspace):
```bash
# Install devmem2 tool
apt-get install devmem2

# Read register (32-bit)
devmem2 0x12340000
# Output: Value at address 0x12340000: 0xABCD1234

# Write register
devmem2 0x12340000 w 0x12345678

# Read multiple registers (bash script)
#!/bin/bash
BASE=0x12340000
for i in {0..15}; do
    ADDR=$(printf "0x%08x" $((BASE + i*4)))
    echo -n "[$ADDR] = "
    devmem2 $ADDR | grep Value
done
```

---

### B) Logic Analyzer / Oscilloscope

#### تجهيز الـ Hardware:

```
Signals to probe:
├── Clock Output Pin (CLKO)
├── Crystal/Oscillator (XIN/XOUT)
├── PLL Output (if accessible)
└── Peripheral Clock Input

Equipment needed:
├── Logic Analyzer (Saleae, cheaplogic, etc.)
├── Oscilloscope (100MHz+ for high-speed clocks)
└── Probes (10:1 for oscilloscope)
```

#### Measurements:

```bash
# 1. Frequency Measurement
# Expected: 24 MHz crystal
# Measure: Should see stable 24.000 MHz ± 50 ppm

# 2. Duty Cycle
# Expected: 50% (for most clocks)
# Measure: Time_high / Period

# 3. Jitter
# Expected: < 50 ps RMS (for good clock)
# Measure: Use oscilloscope's jitter analysis

# 4. Rise/Fall Time
# Expected: < 10% of period
# For 100 MHz: < 1 ns
```

---

### C) JTAG Debugging

```bash
# Connect OpenOCD
openocd -f interface/jlink.cfg -f target/stm32mp15x.cfg

# In OpenOCD console:
> halt
> mdw 0x50000000 0x100    # Memory Display Word (dump 256 registers)

# Read specific register
> mdw 0x50000000          # RCC_TZCR register
0x50000000: abcd1234

# Write register
> mww 0x50000000 0x12345678

# Resume
> resume
```

---

### D) Using /dev/mem (Direct Memory Access)

```c
// Example C program to read registers
#include <stdio.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <stdint.h>

#define CRU_BASE 0x12340000
#define CRU_SIZE 0x1000

int main() {
    int fd = open("/dev/mem", O_RDWR | O_SYNC);

    volatile uint32_t *cru = mmap(NULL, CRU_SIZE,
                                   PROT_READ | PROT_WRITE,
                                   MAP_SHARED, fd, CRU_BASE);

    printf("CRU_CLKSEL_CON0 = 0x%08x\n", cru[0x0000/4]);
    printf("CRU_CLKGATE_CON0 = 0x%08x\n", cru[0x0200/4]);

    munmap((void*)cru, CRU_SIZE);
    close(fd);
    return 0;
}

# Compile and run
gcc -o read_regs read_regs.c
./read_regs
```

---

## 6️⃣ Common Problems & Solutions {#problems}

### ❌ Problem 1: Clock Not Enabling

**Symptoms:**
```bash
# Enable count = 0 even after clk_enable()
cat /sys/kernel/debug/clk/usb_clk/clk_enable_count
# 0

dmesg | grep usb_clk
# usb_clk: failed to enable
```

**Debug Steps:**

```bash
# 1. Check if clock is gated by parent
cat /sys/kernel/debug/clk/usb_clk/clk_parent
cat /sys/kernel/debug/clk/apb_clk/clk_enable_count  # Should be > 0

# 2. Check power domain
cat /sys/kernel/debug/pm_genpd/pm_genpd_summary
# usb_pd should be "on"

# 3. Enable dynamic debug
echo "func clk_enable +p" > /sys/kernel/debug/dynamic_debug/control
echo "func clk_gate_enable +p" > /sys/kernel/debug/dynamic_debug/control

# 4. Check register directly
devmem2 0x12340200  # CRU_CLKGATE_CON
# Bit should be 0 (enabled)
```

**Solution:**
```c
// In driver code, add error checking:
ret = clk_prepare_enable(clk);
if (ret) {
    dev_err(dev, "Failed to enable clock: %d\n", ret);
    // Check parent
    struct clk *parent = clk_get_parent(clk);
    dev_err(dev, "Parent: %s, enabled: %d\n",
            __clk_get_name(parent),
            __clk_is_enabled(parent));
}
```

---

### ❌ Problem 2: Wrong Clock Rate

**Symptoms:**
```bash
# Expected: 48 MHz, Got: 47.923 MHz
cat /sys/kernel/debug/clk/usb_clk/clk_rate
# 47923076
```

**Debug Steps:**

```bash
# 1. Trace the clock tree
cd /sys/kernel/debug/clk/usb_clk
cat clk_parent     # pll2
cd ../pll2
cat clk_rate       # 1200000000
cat clk_parent     # xin24m
cd ../xin24m
cat clk_rate       # 24000000

# 2. Calculate expected rate
# PLL2 = 24M * (M/N) = 24M * (50/1) = 1200M
# USB = PLL2 / 25 = 1200M / 25 = 48M

# 3. Check divider register
devmem2 0x12340010  # USB_CLKDIV register
# Should be 25 (0x19)

# 4. Enable rate tracing
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_set_rate/enable
```

**Solution:**
```c
// Use clk_round_rate to check
long rounded = clk_round_rate(usb_clk, 48000000);
dev_info(dev, "Requested: 48000000, Rounded: %ld\n", rounded);

if (rounded != 48000000) {
    dev_warn(dev, "Can't achieve exact 48MHz\n");
}

clk_set_rate(usb_clk, rounded);
```

---

### ❌ Problem 3: Clock Glitches / Instability

**Symptoms:**
```bash
# UART receiving garbage data
# Devices resetting randomly
# Kernel crashes during clock change
```

**Debug Steps:**

```bash
# 1. Check for clock glitches with oscilloscope
# Look for:
#   - Frequency spikes
#   - Missing pulses
#   - Voltage drops

# 2. Check if CLK_SET_RATE_GATE is needed
cat /sys/kernel/debug/clk/uart_clk/clk_flags
# Should have 0x01 if gate required

# 3. Check notifiers
cat /sys/kernel/debug/clk/uart_clk/clk_notifier_count
# Should be > 0 if driver registered notifier

# 4. Use ftrace to catch the glitch
cd /sys/kernel/debug/tracing
echo 1 > events/clk/clk_set_rate/enable
echo 1 > events/irq/enable  # Check for spurious IRQs
cat trace_pipe
```

**Solution:**
```c
// Add CLK_SET_RATE_GATE flag
static struct clk_init_data uart_clk_init = {
    .name = "uart_clk",
    .ops = &clk_divider_ops,
    .flags = CLK_SET_RATE_GATE,  // Disable before changing
};

// Or register notifier
static int uart_clk_notifier(struct notifier_block *nb,
                             unsigned long event, void *data)
{
    if (event == PRE_RATE_CHANGE) {
        // Pause UART transfers
        uart_stop_tx(uart);
    } else if (event == POST_RATE_CHANGE) {
        // Recalculate baud rate and resume
        uart_update_baud(uart);
        uart_start_tx(uart);
    }
    return NOTIFY_OK;
}
```

---

### ❌ Problem 4: PLL Not Locking

**Symptoms:**
```bash
dmesg | grep PLL
# PLL2 failed to lock
# Timeout waiting for PLL

cat /sys/kernel/debug/clk/pll2/clk_rate
# 0   # PLL not running
```

**Debug Steps:**

```bash
# 1. Check PLL registers
devmem2 0x12340400  # PLL_CON0
devmem2 0x12340404  # PLL_CON1
devmem2 0x12340408  # PLL_CON2 (lock status)

# 2. Check if lock bit is set
# Bit 31 should be 1 when locked

# 3. Check PLL configuration
# M, N, P values must be in valid range

# 4. Verify input clock
cat /sys/kernel/debug/clk/xin24m/clk_rate
# Should be stable 24000000

# 5. Measure with oscilloscope
# Check if reference clock (XIN) is stable
```

**Solution:**
```c
// Add timeout and retry logic
static int wait_pll_lock(void __iomem *base)
{
    int timeout = 1000;
    u32 val;

    while (timeout--) {
        val = readl(base + PLL_CON2);
        if (val & PLL_LOCK_BIT)
            return 0;
        udelay(10);
    }

    pr_err("PLL failed to lock!\n");
    // Dump PLL registers for debug
    pr_err("PLL_CON0=0x%08x\n", readl(base + PLL_CON0));
    pr_err("PLL_CON1=0x%08x\n", readl(base + PLL_CON1));

    return -ETIMEDOUT;
}
```

---

### ❌ Problem 5: Unused Clock Disabled

**Symptoms:**
```bash
dmesg | grep "unused"
# Disabling unused clock: usb_clk

# Device not working because clock was disabled
```

**Debug Steps:**

```bash
# Check if CLK_IGNORE_UNUSED is set
cat /sys/kernel/debug/clk/usb_clk/clk_flags
# Should have 0x08 bit

# Check enable count at boot
dmesg | grep -A5 "clk_summary"
```

**Solution:**

**Option 1: Add flag in driver:**
```c
static struct clk_init_data usb_clk_init = {
    .name = "usb_clk",
    .ops = &clk_gate_ops,
    .flags = CLK_IGNORE_UNUSED,  // Don't disable at boot
};
```

**Option 2: Boot parameter:**
```bash
# Add to kernel command line
clk_ignore_unused

# In bootloader (U-Boot):
setenv bootargs "... clk_ignore_unused"
saveenv
```

**Option 3: Device Tree:**
```dts
&usb_clk {
    clock-ignore-unused;
};
```

---

## 7️⃣ Tools & Scripts {#tools}

### A) Clock Summary Script

```bash
#!/bin/bash
# clock_summary.sh - Enhanced clock summary

echo "=== Clock Tree Summary ==="
echo ""

for CLK in /sys/kernel/debug/clk/*; do
    NAME=$(basename $CLK)

    # Skip if not a directory
    [ ! -d "$CLK" ] && continue

    RATE=$(cat $CLK/clk_rate 2>/dev/null || echo "N/A")
    ENABLE=$(cat $CLK/clk_enable_count 2>/dev/null || echo "N/A")
    PARENT=$(cat $CLK/clk_parent 2>/dev/null || echo "ROOT")

    printf "%-30s %12s Hz  Enable:%s  Parent:%s\n" \
           "$NAME" "$RATE" "$ENABLE" "$PARENT"
done | sort

echo ""
echo "=== Enabled Clocks Only ==="
for CLK in /sys/kernel/debug/clk/*; do
    ENABLE=$(cat $CLK/clk_enable_count 2>/dev/null)
    if [ "$ENABLE" -gt 0 ] 2>/dev/null; then
        NAME=$(basename $CLK)
        RATE=$(cat $CLK/clk_rate)
        printf "%-30s %12s Hz  Count:%d\n" "$NAME" "$RATE" "$ENABLE"
    fi
done
```

---

### B) Clock Rate Converter

```bash
#!/bin/bash
# rate_converter.sh - Convert Hz to human-readable

convert_rate() {
    RATE=$1

    if [ $RATE -ge 1000000000 ]; then
        echo "$(awk "BEGIN {printf \"%.3f\", $RATE/1000000000}") GHz"
    elif [ $RATE -ge 1000000 ]; then
        echo "$(awk "BEGIN {printf \"%.3f\", $RATE/1000000}") MHz"
    elif [ $RATE -ge 1000 ]; then
        echo "$(awk "BEGIN {printf \"%.3f\", $RATE/1000}") KHz"
    else
        echo "$RATE Hz"
    fi
}

# Usage
for CLK in /sys/kernel/debug/clk/*; do
    [ -d "$CLK" ] || continue
    NAME=$(basename $CLK)
    RATE=$(cat $CLK/clk_rate 2>/dev/null)
    [ -z "$RATE" ] && continue

    printf "%-30s : %s\n" "$NAME" "$(convert_rate $RATE)"
done
```

---

### C) Monitor Clock Changes

```bash
#!/bin/bash
# clock_monitor.sh - Monitor clock rate changes

CLOCK=$1
INTERVAL=${2:-1}  # Default 1 second

echo "Monitoring clock: $CLOCK (Ctrl+C to stop)"
echo "Time                Rate           Change"
echo "================================================"

LAST_RATE=$(cat /sys/kernel/debug/clk/$CLOCK/clk_rate)

while true; do
    RATE=$(cat /sys/kernel/debug/clk/$CLOCK/clk_rate)
    TIME=$(date '+%H:%M:%S')

    if [ "$RATE" != "$LAST_RATE" ]; then
        CHANGE=$((RATE - LAST_RATE))
        printf "%s   %12d   %+d\n" "$TIME" "$RATE" "$CHANGE"
        LAST_RATE=$RATE
    fi

    sleep $INTERVAL
done

# Usage:
# ./clock_monitor.sh uart0 1
```

---

### D) PLL Calculator

```python
#!/usr/bin/env python3
# pll_calc.py - Calculate PLL parameters

def calculate_pll(ref_clk, target_freq, max_m=512, max_n=16, max_p=8):
    """
    Calculate PLL parameters: freq = (ref_clk * M) / (N * P)
    """
    best_diff = float('inf')
    best_params = None

    for m in range(1, max_m + 1):
        for n in range(1, max_n + 1):
            for p in range(1, max_p + 1):
                freq = (ref_clk * m) / (n * p)
                diff = abs(freq - target_freq)

                if diff < best_diff:
                    best_diff = diff
                    best_params = (m, n, p, freq)

                    if diff == 0:
                        return best_params

    return best_params

# Example
ref = 24_000_000  # 24 MHz crystal
target = 800_000_000  # 800 MHz target

m, n, p, actual = calculate_pll(ref, target)
print(f"Target: {target/1e6:.3f} MHz")
print(f"Actual: {actual/1e6:.3f} MHz")
print(f"M={m}, N={n}, P={p}")
print(f"Error: {abs(actual-target)} Hz ({abs(actual-target)/target*100:.4f}%)")

# Output:
# Target: 800.000 MHz
# Actual: 800.000 MHz
# M=100, N=3, P=1
# Error: 0 Hz (0.0000%)
```

---

### E) Register Bit Decoder

```bash
#!/bin/bash
# decode_reg.sh - Decode register bits

decode_clkgate_reg() {
    REG_VAL=$1

    echo "Clock Gate Register: 0x$(printf '%08x' $REG_VAL)"
    echo "Bit  Clock       Status"
    echo "=========================="

    CLOCKS=(
        "UART0" "UART1" "UART2" "UART3"
        "I2C0" "I2C1" "SPI0" "SPI1"
        "USB" "SDMMC" "EMMC" "GMAC"
        "GPU" "VPU" "VOPL" "VOPB"
    )

    for i in {0..15}; do
        BIT=$((REG_VAL & (1 << i)))
        if [ $BIT -eq 0 ]; then
            STATUS="ENABLED"
        else
            STATUS="DISABLED"
        fi
        printf "%2d   %-10s %s\n" $i "${CLOCKS[$i]}" "$STATUS"
    done
}

# Usage
REG=$(devmem2 0x12340200 | grep Value | awk '{print $NF}')
decode_clkgate_reg $REG
```

---

### F) Full System Clock Report

```bash
#!/bin/bash
# clock_report.sh - Generate comprehensive report

OUTPUT="clock_report_$(date +%Y%m%d_%H%M%S).txt"

{
    echo "==============================================="
    echo "Clock System Report"
    echo "Generated: $(date)"
    echo "Kernel: $(uname -r)"
    echo "==============================================="
    echo ""

    echo "=== Clock Summary ==="
    cat /sys/kernel/debug/clk/clk_summary
    echo ""

    echo "=== Enabled Clocks ==="
    for CLK in /sys/kernel/debug/clk/*; do
        EN=$(cat $CLK/clk_enable_count 2>/dev/null)
        if [ "$EN" -gt 0 ] 2>/dev/null; then
            NAME=$(basename $CLK)
            RATE=$(cat $CLK/clk_rate)
            PARENT=$(cat $CLK/clk_parent)
            echo "$NAME: $RATE Hz (enable_count=$EN, parent=$PARENT)"
        fi
    done
    echo ""

    echo "=== Recent Clock Messages ==="
    dmesg | grep -i clk | tail -50
    echo ""

    echo "=== Clock Drivers Loaded ==="
    lsmod | grep clk
    echo ""

    echo "=== Device Tree Clocks ==="
    find /proc/device-tree -name "*clock*" -type f 2>/dev/null | \
    while read F; do
        echo "$F: $(xxd -p $F | tr -d '\n')"
    done

} > "$OUTPUT"

echo "Report saved to: $OUTPUT"
```

---

## 🎯 Quick Reference Card

### Essential Commands:
```bash
# View all clocks
cat /sys/kernel/debug/clk/clk_summary

# Check specific clock
cat /sys/kernel/debug/clk/uart0/clk_rate
cat /sys/kernel/debug/clk/uart0/clk_enable_count

# Enable debug messages
echo "file drivers/clk/*.c +p" > /sys/kernel/debug/dynamic_debug/control

# Trace clock operations
echo 1 > /sys/kernel/debug/tracing/events/clk/enable
cat /sys/kernel/debug/tracing/trace_pipe

# Read register
devmem2 0x12340000

# Monitor dmesg
dmesg -w | grep -i clk
```

---

## 📚 References

```
Documentation:
├── Kernel Docs: Documentation/driver-api/clk.rst
├── Device Tree: Documentation/devicetree/bindings/clock/
├── Ftrace: Documentation/trace/ftrace.rst
└── DebugFS: Documentation/filesystems/debugfs.txt

Source Code:
├── Core: drivers/clk/clk.c
├── Providers: drivers/clk/clk-*.c
├── Platform: drivers/clk/<vendor>/
└── Headers: include/linux/clk*.h
```

---

## 🔥 Pro Tips

1. **Always enable CONFIG_COMMON_CLK_DEBUG** في development builds
2. **Use ftrace events** أسهل من printk debugging
3. **Check parent chain** لما السرعة غلط
4. **Verify hardware** مع oscilloscope للـ critical clocks
5. **Test with stress** - غير السرعة كتير وشوف stability
6. **Document findings** - اعمل script يطلع report
7. **Compare with working board** لو عندك reference
8. **Read TRM carefully** - الـ Technical Reference Manual فيه كل حاجة

---

الـ Cheatsheet ده يكفيك لتعمل debug لأي مشكلة في الـ clock subsystem! 🎓💪
