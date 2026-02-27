## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem: Common Clock Framework (CCF)

الملف `clk-gate.c` جزء من الـ **Common Clock Framework (CCF)** — وده الـ subsystem المسؤول عن إدارة كل الـ clocks جوه الـ Linux kernel. الـ maintainers هما Michael Turquette و Stephen Boyd، والكود كله تحت `drivers/clk/`.

---

### المشكلة اللي بيحلها — بالقصة

تخيل معايا إنك عندك مبنى فيه ألف أوضة، وكل أوضة عندها كهرباء بتشتغل طول الوقت. ده بيبدد طاقة هائلة. الحل الذكي إنك تحط **مفتاح كهرباء** على كل أوضة — لما محدش فيها تقفله، ولما حد يدخل تفتحه. ده بالظبط اللي بيعمله `clk-gate.c`.

**الـ clock** في الـ hardware هو النبض الإيقاعي اللي بيخلي الـ chip يشتغل — زي نبضة القلب للـ CPU، الـ USB controller، الـ memory controller، إلخ. كل component محتاج clock عشان يشتغل. لو سبت كل الـ clocks شغالة طول الوقت، هتاكل battery بسرعة وهتسخن الـ chip من غير لازمة.

**الحل:** لكل component اللي مش بيُستخدم دلوقتي، بتـ"تقفل" الـ clock بتاعه — تقطع عنه النبضة — فيوقف يأكل طاقة. لما يُحتاج تاني، بتـ"تفتحه" تاني.

الـ `clk-gate.c` هو اللي بيعرف الـ Linux kernel يعمل الـ enable و disable دول بكتابة **bit واحد في register** في الـ hardware.

---

### الصورة الكبيرة — إيه اللي بيحصل بالظبط؟

```
[Driver بيطلب enable للـ clock]
         |
         v
    clk_enable()   <-- API عامة في include/linux/clk.h
         |
         v
  [CCF Framework في drivers/clk/clk.c]
         |
         v
   clk_gate_enable()  <-- في clk-gate.c
         |
         v
  [كتابة bit في register في الـ hardware]
  reg |= BIT(bit_idx)   --> الـ clock اتفتح
```

الـ Framework بيفصل بين:
- **من فوق:** الـ driver اللي مش محتاج يعرف إيه الـ hardware
- **من تحت:** الـ hardware اللي عنده register محدد في عنوان محدد

---

### إيه اللي بيعمله `clk-gate.c` بالتفصيل

الملف ده بيوفر **نوع واحد بسيط من الـ clocks**: clock ممكن تفتحه أو تقفله بكتابة bit واحد في memory-mapped register.

```
struct clk_gate:
┌─────────────────────────────┐
│  hw       → ربط بالـ CCF   │
│  reg      → عنوان الـ reg  │
│  bit_idx  → رقم الـ bit    │
│  flags    → سلوك خاص       │
│  lock     → spinlock للحماية│
└─────────────────────────────┘
```

الـ flags المهمة:
| Flag | المعنى |
|------|---------|
| `CLK_GATE_SET_TO_DISABLE` | عكوس — بدل ما 1 = enable، بقى 1 = disable |
| `CLK_GATE_HIWORD_MASK` | الـ bits العليا في الـ register هي الـ mask للكتابة |
| `CLK_GATE_BIG_ENDIAN` | الـ register بيُقرأ بـ big-endian |

---

### الـ Logic الأساسي — `clk_gate_endisable`

المنطق جوا الكود بيعتمد على XOR بسيط:

```
enable XOR set2dis = result
  1    XOR    0    =   1   --> set the bit (enable)
  0    XOR    0    =   0   --> clear the bit (disable)
  1    XOR    1    =   0   --> clear the bit (enable with inverted logic)
  0    XOR    1    =   1   --> set the bit (disable with inverted logic)
```

ده بيخلي الكود يدعم الـ hardware اللي بيعتبر 0 = enable و 1 = disable بدون if/else تانية.

---

### الـ HIWORD_MASK — ليه موجود؟

بعض الـ hardware (زي Rockchip) عندها registers بيشتغلوا بطريقة خاصة: الـ 16-bit العليا هي الـ "write mask" والـ 16-bit الدنيا هي القيمة الفعلية. يعني لو عايز تغير bit 5، لازم تكتب:

```
bit 21 = 1  (في الـ upper half — قول للـ hardware "أنا بغير bit 5")
bit 5  = 1  (في الـ lower half — القيمة الجديدة)
```

ده بيمنع الـ race conditions لأن كتابة واحدة بتعمل الـ mask والـ value مع بعض.

---

### الـ devm — إدارة الـ memory أوتوماتيك

الملف بيوفر نسختين من الـ registration:
- **`__clk_hw_register_gate`** — عادية، لازم تـ unregister يدوياً
- **`__devm_clk_hw_register_gate`** — الـ framework بيتكفل بالـ cleanup لما الـ device يتشال

---

### الـ Big Endian Support

الـ SoCs المختلفة بتختلف في ترتيب الـ bytes في الـ registers. لذلك في:
- `readl` / `writel` — للـ little-endian (الأغلبية)
- `ioread32be` / `iowrite32be` — للـ big-endian (زي بعض الـ MIPS و PowerPC)

---

### ملفات الـ Subsystem اللي المفروض تعرفها

#### الـ Core (القلب)
| الملف | الدور |
|-------|-------|
| `drivers/clk/clk.c` | قلب الـ CCF — بيدير كل الـ clock tree |
| `drivers/clk/clk-gate.c` | **ملفنا** — الـ gatable clock type |
| `drivers/clk/clk-divider.c` | clock بيقسم التردد |
| `drivers/clk/clk-mux.c` | clock بيختار من مصادر متعددة |
| `drivers/clk/clk-fixed-rate.c` | clock بتردد ثابت |
| `drivers/clk/clk-fixed-factor.c` | clock بمعامل ضرب/قسمة ثابت |
| `drivers/clk/clk-composite.c` | clock مركب من gate + divider + mux |
| `drivers/clk/clk-devres.c` | الـ devm helpers |

#### الـ Headers
| الملف | الدور |
|-------|-------|
| `include/linux/clk-provider.h` | الـ API للي بيكتب clock drivers — فيه `struct clk_gate`، `struct clk_ops`، `struct clk_hw` |
| `include/linux/clk.h` | الـ API للي بيستخدم الـ clocks من الـ drivers العادية |
| `include/linux/clkdev.h` | ربط الـ clocks بالـ devices |

#### الـ Hardware Drivers (أمثلة واقعية بتستخدم clk-gate)
| الملف | الـ SoC |
|-------|---------|
| `drivers/clk/rockchip/` | Rockchip RK3399, RK3588 (Raspberry Pi 5 variant) |
| `drivers/clk/samsung/` | Exynos SoCs |
| `drivers/clk/bcm/clk-bcm2835.c` | Raspberry Pi |
| `drivers/clk/qcom/` | Qualcomm Snapdragon |
| `drivers/clk/mediatek/` | MediaTek SoCs |
| `drivers/clk/meson/` | Amlogic (Android TV boxes) |

---

### ملفات تانية مهمة للقارئ

- `scripts/gdb/linux/clk.py` — سكريبت GDB للـ debug بتاع الـ clock tree وقت الـ runtime
- `Documentation/devicetree/bindings/clock/` — كيف تعرّف الـ clocks في الـ device tree
- `drivers/clk/clk-gate_test.c` — الـ unit tests للـ gate clock
## Phase 2: شرح الـ Common Clock Framework (CCF) — Gate Clock Subsystem

---

### المشكلة اللي الـ Subsystem بيحلها

أي SoC حديث — خد مثلاً Allwinner H3 أو Rockchip RK3399 — فيه عشرات الـ clocks: clock للـ CPU، clock للـ USB، clock للـ UART، clock للـ DRAM controller، إلخ. كل clock ممكن:

- يتـ**enable/disable** (gate)
- يتغير الـ **rate** بتاعته (divider/PLL)
- يتغير الـ **parent** بتاعه (mux)

قبل الـ CCF (قبل kernel 3.4)، كل platform كانت بتعمل الـ clock management بطريقتها الخاصة — كود مكرر، inconsistent API، مستحيل تعمل cross-platform clock sharing.

**الـ CCF** اتعمل عشان يوحّد ده كله في framework واحد: API موحّد للـ consumers، وabstraction layer واضح للـ providers (الـ hardware-specific drivers).

**الـ clk-gate.c** تحديداً بيحل مشكلة واحدة بسيطة وشائعة جداً: clock بتـ enable/disable بـ **bit واحد في register** في الـ SoC. ده أكتر نوع clock موجود في أي SoC.

---

### الحل: الـ Layered Abstraction

الـ CCF بيقسم الـ clock management لطبقتين:

1. **Generic Framework Layer** (`drivers/clk/clk.c`): بيتعامل مع الـ enable counting، dependency tracking، debugfs، notifiers — كود مشترك بين كل الـ clocks.
2. **Hardware-Specific Layer** (`drivers/clk/clk-gate.c` وأمثاله): بيعرف بس إزاي يعمل enable/disable للـ hardware الفعلي.

التواصل بينهم بيحصل عن طريق `struct clk_ops` — table of function pointers — بالظبط زي الـ VFS و `file_operations`.

---

### التشبيه الواقعي: كهربائي في فيلا

تخيل فيلا كبيرة فيها لوحة كهربائية رئيسية (الـ CCF core). كل أوضة فيها قاطع (circuit breaker) بيتحكم في الكهربا بتاعتها.

| التشبيه | الـ Kernel Concept |
|---|---|
| لوحة الكهربا الرئيسية | `drivers/clk/clk.c` — الـ CCF core |
| القاطع بتاع كل أوضة | `struct clk_gate` — hardware gate descriptor |
| رقم القاطع في اللوحة | `bit_idx` في الـ register |
| عنوان اللوحة الفرعية | `void __iomem *reg` — عنوان الـ MMIO register |
| الكهربائي اللي بيلف على القواطع | `clk_gate_ops` — الـ ops table |
| الـ safety rule: متقطعش أثناء التحميل | `spinlock_t *lock` — mutual exclusion |
| النوع العكسي: القاطع Up = Off | `CLK_GATE_SET_TO_DISABLE` flag |
| اللوحة بـ big-endian addressing | `CLK_GATE_BIG_ENDIAN` flag |
| الساكن اللي بيطلب النور | driver يعمل `clk_enable()` |
| عداد الـ residents في الأوضة | الـ enable reference count في CCF core |

الكهربائي (ops) ما بيعرفش مين الساكن — ده شغل اللوحة الرئيسية. هو بس بيعرف إزاي يشغل القاطع رقم X في اللوحة الفرعية Y.

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Consumer Drivers                         │
│         (USB driver, UART driver, GPU driver, ...)              │
│                                                                 │
│   clk_enable(clk)  clk_disable(clk)  clk_set_rate(clk, rate)   │
└────────────────────────────┬────────────────────────────────────┘
                             │  Public Consumer API (include/linux/clk.h)
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    CCF Core (clk.c)                             │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ enable_count │  │ prepare_count│  │  clk_core tree       │  │
│  │  reference   │  │  reference   │  │  (parent/child links)│  │
│  └──────┬───────┘  └──────────────┘  └──────────────────────┘  │
│         │                                                       │
│  calls ops->enable() / ops->disable() / ops->is_enabled()      │
└─────────────────────────────┬───────────────────────────────────┘
                              │  struct clk_ops (provider API)
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              Hardware-Specific Implementations                  │
│                                                                 │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │ clk-gate.c  │  │ clk-divider.c│  │  clk-mux.c             │ │
│  │ (enable bit)│  │ (rate divider│  │  (parent selector)     │ │
│  └──────┬──────┘  └──────────────┘  └────────────────────────┘ │
│         │                                                       │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │         Platform Clock Drivers                              ││
│  │  (drivers/clk/sunxi/, drivers/clk/rockchip/, ...)          ││
│  │  بيبنوا clk_gate + clk_divider + clk_mux مع بعض           ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      SoC Hardware                               │
│                                                                 │
│   Clock Control Unit (CCU) — block of MMIO registers           │
│   ┌────────┬────────┬────────┬────────┬────────────────────┐   │
│   │ reg+0  │ reg+4  │ reg+8  │ reg+C  │ ...                │   │
│   │ [b0]=USB│[b2]=SPI│[b0]=I2C│        │                    │   │
│   │ gate   │ gate  │ gate   │        │                    │   │
│   └────────┴────────┴────────┴────────┴────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction: الـ `clk_hw` كـ Embedded Handle

الفكرة المحورية في الـ CCF هي الـ **embedded handle pattern**. بدل ما الـ framework يعمل pointer لـ hardware struct، الـ hardware struct بتـ embed الـ handle جوّاها:

```c
/* الـ generic gate descriptor */
struct clk_gate {
    struct clk_hw hw;    /* ← الـ handle مـ embedded جوا الـ hardware struct */
    void __iomem *reg;   /* عنوان الـ gate register في الـ MMIO space */
    u8 bit_idx;          /* رقم الـ bit اللي بيتحكم في الـ gate */
    u8 flags;            /* CLK_GATE_SET_TO_DISABLE | CLK_GATE_HIWORD_MASK | CLK_GATE_BIG_ENDIAN */
    spinlock_t *lock;    /* بيحمي الـ read-modify-write من الـ race conditions */
};

/* الـ generic framework handle */
struct clk_hw {
    struct clk_core *core;        /* pointer للـ CCF internal state */
    struct clk *clk;              /* per-user handle للـ consumers */
    const struct clk_init_data *init; /* بيتـ NULL بعد الـ registration */
};
```

**الـ `to_clk_gate()` macro** بيعمل الحركة العكسية: من `clk_hw *` للـ `clk_gate *` الـ containing struct:

```c
#define to_clk_gate(_hw) container_of(_hw, struct clk_gate, hw)
```

ده بالظبط نفس pattern الـ `container_of` المستخدم في `list_head`، `kobject`، وكتير من kernel subsystems.

---

### العلاقة بين الـ Structs

```
┌─────────────────────────────────┐
│        struct clk_gate          │
│ ┌─────────────────────────────┐ │
│ │      struct clk_hw  hw      │ │◄── CCF core يشوف بس الـ clk_hw
│ │  ┌──────────────────────┐   │ │
│ │  │ struct clk_core *core│   │ │
│ │  │ struct clk *clk      │   │ │
│ │  │ clk_init_data *init  │   │ │
│ │  └──────────────────────┘   │ │
│ └─────────────────────────────┘ │
│ void __iomem *reg  ─────────────┼──► MMIO register في الـ CCU
│ u8 bit_idx         ─────────────┼──► رقم الـ bit (0-31 أو 0-15)
│ u8 flags           ─────────────┼──► SET_TO_DISABLE, HIWORD_MASK, BIG_ENDIAN
│ spinlock_t *lock   ─────────────┼──► shared lock مع registers تانية ممكن
└─────────────────────────────────┘
           ▲
           │ container_of(hw, struct clk_gate, hw)
           │
    ┌──────┴──────┐
    │ clk_gate_ops│  (enable, disable, is_enabled)
    └─────────────┘
```

---

### الـ `clk_ops` Table: عقد بين الـ Framework والـ Hardware

```c
const struct clk_ops clk_gate_ops = {
    .enable    = clk_gate_enable,    /* يكتب 1 في الـ gate bit */
    .disable   = clk_gate_disable,   /* يكتب 0 في الـ gate bit */
    .is_enabled = clk_gate_is_enabled, /* يقرأ الـ gate bit */
    /* لا recalc_rate ← rate بييجي من الـ parent مباشرة */
    /* لا set_parent  ← parent ثابت */
    /* لا round_rate  ← مفيش rate manipulation */
};
```

الـ gate clock عمداً بيـ implement بس 3 ops. ده مش نقص — ده تصميم. الـ CCF core بيفهم إن غياب `recalc_rate` معناه "خد الـ rate من الـ parent بدون تغيير".

---

### منطق الـ Enable/Disable: الـ XOR Trick

الـ `clk_gate_endisable()` بتحل مشكلة الـ polarity بأسلوب أنيق:

```c
static void clk_gate_endisable(struct clk_hw *hw, int enable)
{
    struct clk_gate *gate = to_clk_gate(hw);
    /* set = 1 لو الـ hardware بيـ disable بالـ set (inverted logic) */
    int set = gate->flags & CLK_GATE_SET_TO_DISABLE ? 1 : 0;
    u32 reg;

    /* XOR: لو enable=1 و set=0  → set=1 (اكتب 1 لتشغيل)
             لو enable=1 و set=1  → set=0 (امسح 1 لتشغيل — inverted)
             لو enable=0 و set=0  → set=0 (امسح 1 لإيقاف)
             لو enable=0 و set=1  → set=1 (اكتب 1 لإيقاف — inverted) */
    set ^= enable;

    /* ...spinlock... */

    if (gate->flags & CLK_GATE_HIWORD_MASK) {
        /* بعض الـ SoCs زي Rockchip: الـ high 16-bit هي الـ write-enable mask
           بيحدد أي bits في الـ low 16 مسموح تتغير — atomic write بدون RMW */
        reg = BIT(gate->bit_idx + 16);  /* enable the mask bit */
        if (set)
            reg |= BIT(gate->bit_idx);  /* set the gate bit */
    } else {
        /* الـ normal case: read-modify-write */
        reg = clk_gate_readl(gate);
        if (set)
            reg |= BIT(gate->bit_idx);
        else
            reg &= ~BIT(gate->bit_idx);
    }

    clk_gate_writel(gate, reg);
    /* ...spinlock... */
}
```

#### ليه الـ HIWORD_MASK مهم؟

في Rockchip SoCs، بعض الـ clock registers بتدعم الـ "write-with-mask" pattern: الـ high 16-bit بيحدد أي bits من الـ low 16-bit هتتغير. ده بيخلي الـ write **atomic** — مش محتاج read-modify-write cycle، وده بيحل مشكلة الـ race condition بدون spinlock في الـ hardware نفسه.

```
Register (32-bit):
┌────────────────────┬────────────────────┐
│   High 16-bit      │   Low 16-bit       │
│   (Write Mask)     │   (Actual Values)  │
│  bit 16 = mask     │  bit 0 = gate val  │
│  for bit 0         │                    │
└────────────────────┴────────────────────┘

Write 0x00010001 → يغير bit 0 إلى 1 (mask bit 16 مفتوح)
Write 0x00010000 → يغير bit 0 إلى 0
Write 0x00000001 → NO-OP (mask bit مش مفتوح) — الـ register مش هيتغير
```

---

### الـ Registration Flow: من الـ Driver للـ CCF

```
SoC Clock Driver (e.g., drivers/clk/sunxi-ng/ccu-h3.c)
    │
    │  clk_hw_register_gate(dev, "usb-clk", "pll-periph",
    │                        flags, reg, bit_idx, gate_flags, lock)
    │
    ▼
__clk_hw_register_gate()                [clk-gate.c]
    │
    ├── kzalloc(struct clk_gate)         ← allocate hardware descriptor
    ├── fill gate->reg, bit_idx, flags, lock
    ├── fill clk_init_data { name, ops=&clk_gate_ops, parent_names, flags }
    ├── gate->hw.init = &init
    │
    ├── clk_hw_register(dev, &gate->hw)  ← دخل للـ CCF core
    │       │
    │       └── clk_core_alloc()         ← الـ CCF بيعمل struct clk_core
    │           clk_core_link_to_parent() ← بيبني الـ clock tree
    │           clk_register debugfs
    │
    └── return &gate->hw                 ← الـ driver بيحتفظ بالـ hw pointer
```

بعد الـ registration، الـ `init` pointer بيتـ set لـ NULL — الـ CCF نقل كل المعلومات اللي محتاجها للـ `struct clk_core` الداخلية.

---

### الـ devm Pattern: Automatic Cleanup

```c
struct clk_hw *__devm_clk_hw_register_gate(struct device *dev, ...)
{
    /* allocate a pointer slot managed by devres */
    struct clk_hw **ptr = devres_alloc(devm_clk_hw_release_gate,
                                       sizeof(*ptr), GFP_KERNEL);

    hw = __clk_hw_register_gate(...);  /* الـ actual registration */

    if (!IS_ERR(hw)) {
        *ptr = hw;
        devres_add(dev, ptr);  /* ربط الـ cleanup بالـ device lifetime */
    }
    return hw;
}

/* لما الـ device يتـ remove، kernel بيكال ده automatically */
static void devm_clk_hw_release_gate(struct device *dev, void *res)
{
    clk_hw_unregister_gate(*(struct clk_hw **)res);
}
```

ده بيضمن إن الـ clock بيتـ unregister وبيتمسح الـ memory لما الـ driver يتـ unbound — zero manual cleanup في الـ driver نفسه.

---

### ما الـ Framework بيملكه VS ما بيفوّضه للـ Driver

| المهمة | المسؤول | الكود |
|---|---|---|
| عدّ الـ enable references (reference counting) | CCF Core | `clk.c` |
| إدارة الـ clock tree (parent/child) | CCF Core | `clk.c` |
| propagate rate changes للـ children | CCF Core | `clk.c` |
| debugfs entries (clk_summary) | CCF Core | `clk.c` |
| notifier chain عند تغيير الـ rate | CCF Core | `clk.c` |
| كتابة الـ enable bit في الـ register | clk-gate driver | `clk-gate.c` |
| قراءة حالة الـ gate من الـ hardware | clk-gate driver | `clk-gate.c` |
| تحديد endianness الـ register | clk-gate driver | `clk-gate.c` |
| التعامل مع الـ HIWORD_MASK hardware quirk | clk-gate driver | `clk-gate.c` |
| الـ spinlock لحماية الـ RMW | shared (driver يوفره) | `clk-gate.c` |

---

### مثال واقعي: USB Clock على Allwinner H3

```c
/* من drivers/clk/sunxi-ng/ccu-h3.c */
static SUNXI_CCU_GATE(bus_usb0_clk, "bus-usb0", "ahb1",
                      0x060, BIT(24), 0);

/* اللي بيحصل تحت: */
// reg    = 0x060 (offset في CCU base)
// bit    = 24
// parent = "ahb1" clock
// flags  = 0 (normal polarity: set bit = enable)
```

لما driver يعمل `clk_enable(usb_clk)`:
1. CCF core بيزوّد الـ enable_count
2. لو أول enable، بيكال `clk_gate_ops.enable`
3. `clk_gate_enable` → `clk_gate_endisable(hw, 1)`
4. بيقرأ الـ register عند offset `0x060`
5. بيعمل set للـ bit 24
6. بيكتب الـ register تاني

النتيجة: الـ USB controller دلوقتي شايف clock signal.

---

### ملخص: ليه clk-gate.c موجود كـ Separate File؟

الـ CCF بيطبّق مبدأ الـ **separation of concerns** بشكل صارم:

- **الـ policy** (متى تـ enable، إيه الـ dependencies) → CCF core
- **الـ mechanism** (إزاي تكتب في الـ hardware) → الـ gate/divider/mux drivers

الـ `clk-gate.c` هو أبسط implementation ممكن لأبسط نوع clock: bit واحد، register واحد، ثلاث operations فقط. ده بيخليه template مثالي لأي SoC driver محتاج gate clock — بدل ما يكتب نفس الكود من الأول، بيستخدم `clk_hw_register_gate()` ويخلص.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums — Cheatsheet

#### الـ CLK_GATE flags (خاصة بـ clk_gate، موجودة في `clk-provider.h`)

| Flag | Value | المعنى |
|------|-------|--------|
| `CLK_GATE_SET_TO_DISABLE` | `BIT(0)` | بدل ما bit=1 يعني enable، هنا bit=1 يعني disable — عكس السلوك الافتراضي |
| `CLK_GATE_HIWORD_MASK` | `BIT(1)` | الـ register بيستخدم الـ upper 16-bit كـ write-enable mask — شائع في Rockchip |
| `CLK_GATE_BIG_ENDIAN` | `BIT(2)` | اقرأ/اكتب الـ register بـ big-endian بدل little-endian |

#### الـ framework-level CLK flags (بتتحط في `clk_init_data.flags`)

| Flag | Value | المعنى |
|------|-------|--------|
| `CLK_SET_RATE_GATE` | `BIT(0)` | لازم تعمل gate قبل ما تغير الـ rate |
| `CLK_SET_PARENT_GATE` | `BIT(1)` | لازم تعمل gate قبل تغيير الـ parent |
| `CLK_SET_RATE_PARENT` | `BIT(2)` | وصّل طلب تغيير الـ rate للـ parent |
| `CLK_IGNORE_UNUSED` | `BIT(3)` | متوقفش الـ clock حتى لو محدش بيستخدمه |
| `CLK_GET_RATE_NOCACHE` | `BIT(6)` | اسأل الـ hardware دايماً، متستخدمش الـ cached rate |
| `CLK_IS_CRITICAL` | `BIT(11)` | لا تعمل gate أبداً، حتى لو مفيش مستخدمين |
| `CLK_OPS_PARENT_ENABLE` | `BIT(12)` | فعّل الـ parent قبل أي gate/rate operation |

---

### الـ Structs المهمة

#### 1. `struct clk_gate`

**الغرض:** بيمثل clock واحد بيقدر يتفعّل ويتوقف عن طريق bit واحد في register معين في الـ hardware.

```c
struct clk_gate {
    struct clk_hw hw;       // الحلقة اللي بتربطه بالـ CCF — لازم يكون أول field
    void __iomem  *reg;     // عنوان الـ register في الـ MMIO space
    u8            bit_idx;  // رقم الـ bit اللي بيتحكم في الـ gate (0-15 في حالة HIWORD_MASK)
    u8            flags;    // CLK_GATE_* flags
    spinlock_t    *lock;    // lock خارجي بيحمي الـ register (ممكن يكون NULL)
};
```

| Field | النوع | الشرح |
|-------|-------|-------|
| `hw` | `struct clk_hw` | الجسر بين الـ hardware-specific code والـ CCF — محتوي على `core` و`clk` و`init` |
| `reg` | `void __iomem *` | الـ MMIO address للـ register اللي فيه الـ gate bit |
| `bit_idx` | `u8` | رقم الـ bit (0-31 عادةً، أو 0-15 لو `CLK_GATE_HIWORD_MASK` مفعّل) |
| `flags` | `u8` | خليط من `CLK_GATE_SET_TO_DISABLE`, `CLK_GATE_HIWORD_MASK`, `CLK_GATE_BIG_ENDIAN` |
| `lock` | `spinlock_t *` | pointer لـ spinlock خارجي — الـ driver بيديه، مش الـ gate نفسه. ممكن `NULL` لو الـ register ليه atomic write (HIWORD_MASK) |

---

#### 2. `struct clk_hw`

**الغرض:** الـ handle اللي بيربط الـ hardware-specific struct (زي `clk_gate`) بالـ Common Clock Framework.

```c
struct clk_hw {
    struct clk_core *core;           // CCF internal representation
    struct clk      *clk;            // per-user clk handle
    const struct clk_init_data *init; // init data — بيتمسح بعد التسجيل
};
```

الـ macro `to_clk_gate(_hw)` بيستخدم `container_of` عشان يرجع من `clk_hw *` للـ `clk_gate *`.

---

#### 3. `struct clk_init_data`

**الغرض:** بيحمل كل المعلومات اللازمة لتسجيل الـ clock في الـ CCF — بيتمسح من `clk_hw` بعد ما يتسجّل.

```c
struct clk_init_data {
    const char              *name;         // اسم الـ clock
    const struct clk_ops    *ops;          // pointer للـ ops table
    const char * const      *parent_names; // طريقة 1: أسامي الـ parents
    const struct clk_parent_data *parent_data; // طريقة 2: parent data (أحدث)
    const struct clk_hw    **parent_hws;   // طريقة 3: مؤشرات مباشرة
    u8                       num_parents;  // عدد الـ parents (0 أو 1 في الـ gate)
    unsigned long            flags;        // CLK_* framework flags
};
```

---

#### 4. `struct clk_ops`

**الغرض:** جدول الـ callbacks اللي الـ CCF بيستدعيها — الـ gate بيملي 3 فقط منها.

```c
// الـ gate بيستخدم بس:
const struct clk_ops clk_gate_ops = {
    .enable     = clk_gate_enable,      // اشغّل الـ clock
    .disable    = clk_gate_disable,     // وقّف الـ clock
    .is_enabled = clk_gate_is_enabled,  // اسأل الـ hardware
};
// باقي الـ ops (set_rate، set_parent، إلخ) = NULL → الـ gate لا يدعمها
```

---

#### 5. `struct clk_parent_data`

**الغرض:** بيوصف الـ parent clock بطريقة أكثر مرونة من مجرد الاسم.

```c
struct clk_parent_data {
    const struct clk_hw *hw;       // pointer مباشر (لو internal)
    const char          *fw_name;  // اسم في الـ device tree
    const char          *name;     // اسم global كـ fallback
    int                  index;    // index في الـ DT
};
```

---

### رسم العلاقات بين الـ Structs

```
┌─────────────────────────────────────────────────────┐
│                  struct clk_gate                    │
│  ┌──────────────┐                                   │
│  │ struct clk_hw│◄── to_clk_gate() ──────────────── │
│  │   .core ─────┼──────────────► struct clk_core   │
│  │   .clk  ─────┼──────────────► struct clk         │
│  │   .init ─────┼──► struct clk_init_data           │
│  │              │      .ops ──────► struct clk_ops  │
│  │              │      .parent_names/data/hws        │
│  └──────────────┘                                   │
│  .reg  ──────────────────────────► MMIO register   │
│  .bit_idx (0-15/31)                                 │
│  .flags (CLK_GATE_*)                                │
│  .lock ──────────────────────────► spinlock_t       │
└─────────────────────────────────────────────────────┘
         │
         │ clk_hw_register() / of_clk_hw_register()
         ▼
┌─────────────────────────────────────────────────────┐
│              CCF (Common Clock Framework)            │
│   struct clk_core (internal, hidden from drivers)   │
│   - enable_count, prepare_count                     │
│   - parent, children list                           │
│   - cached rate                                     │
└─────────────────────────────────────────────────────┘
```

---

### Lifecycle Diagram — دورة حياة الـ clk_gate

```
[Driver/Platform Code]
        │
        │ 1. يجهّز: reg، bit_idx، flags، lock
        │
        ▼
__clk_hw_register_gate()
        │
        ├── kzalloc(sizeof(clk_gate))     ← تخصيص ذاكرة
        │
        ├── يملي clk_init_data:
        │     .name, .ops=&clk_gate_ops
        │     .flags, .parent_*
        │     .num_parents
        │
        ├── يملي clk_gate fields:
        │     .reg, .bit_idx, .flags, .lock
        │     .hw.init = &init
        │
        ├── if (dev || !np)  → clk_hw_register(dev, hw)
        │   else if (np)     → of_clk_hw_register(np, hw)
        │        │
        │        └── CCF: يخزّن الـ clock، يعمل clk_core
        │                  ويمسح hw->init = NULL
        │
        ▼
     [REGISTERED — جاهز للاستخدام]
        │
        │ consumer calls: clk_prepare() → clk_enable()
        │                     ↓               ↓
        │              prepare_lock      enable_lock
        │              (mutex)           (spinlock)
        │                             clk_gate_enable()
        │                             clk_gate_endisable(hw, 1)
        │                             → spin_lock_irqsave
        │                             → writel/iowrite32be
        │                             → spin_unlock_irqrestore
        │
        │ consumer calls: clk_disable() → clk_unprepare()
        │                     ↓
        │              clk_gate_disable()
        │              clk_gate_endisable(hw, 0)
        │
        ▼
clk_hw_unregister_gate(hw)     OR    clk_unregister_gate(clk)
        │
        ├── clk_hw_unregister(hw)   ← CCF بيحذف الـ clk_core
        └── kfree(gate)             ← تحرير الذاكرة
```

**الـ devm variant:**
```
__devm_clk_hw_register_gate()
        │
        ├── devres_alloc(devm_clk_hw_release_gate, ...)
        ├── __clk_hw_register_gate(...)
        └── devres_add(dev, ptr)
                │
                └── عند device release تلقائياً:
                    devm_clk_hw_release_gate()
                    → clk_hw_unregister_gate()
                    → kfree(gate)
```

---

### Call Flow Diagrams

#### تفعيل الـ clock (clk_enable)

```
consumer: clk_enable(clk)
  → CCF: clk_core_enable()
    → acquires enable_lock (spinlock, irq-safe)
      → ops->enable(hw)                      [= clk_gate_enable]
        → clk_gate_endisable(hw, enable=1)
          │
          ├── to_clk_gate(hw)                [container_of]
          ├── set = (CLK_GATE_SET_TO_DISABLE) ? 1 : 0
          ├── set ^= 1                       [XOR logic]
          ├── spin_lock_irqsave(gate->lock)
          │
          ├── [CLK_GATE_HIWORD_MASK?]
          │     YES: reg = BIT(bit_idx + 16)  ← enable mask bit
          │           if set: reg |= BIT(bit_idx)
          │           → single atomic write (no read needed)
          │
          │     NO:  reg = clk_gate_readl()   ← read-modify-write
          │           if set: reg |= BIT(bit_idx)
          │           else:   reg &= ~BIT(bit_idx)
          │
          ├── clk_gate_writel(gate, reg)
          │     [BIG_ENDIAN?] iowrite32be() : writel()
          │
          └── spin_unlock_irqrestore(gate->lock)
```

#### استعلام الحالة (clk_is_enabled)

```
consumer: clk_is_enabled(clk)
  → CCF: clk_core_is_enabled()
    → ops->is_enabled(hw)                    [= clk_gate_is_enabled]
      → clk_gate_readl(gate)                 [مراعاة BIG_ENDIAN]
      → if CLK_GATE_SET_TO_DISABLE:
            reg ^= BIT(bit_idx)              [اعكس المنطق]
      → reg &= BIT(bit_idx)
      → return reg ? 1 : 0
```

#### تسجيل الـ clock

```
driver: clk_hw_register_gate(dev, name, parent, flags, reg, bit, gflags, lock)
  → macro expands to __clk_hw_register_gate(dev, NULL, ...)
    → validate: if HIWORD_MASK && bit_idx > 15 → EINVAL
    → kzalloc(struct clk_gate)
    → fill clk_init_data{name, ops, flags, parent_*, num_parents}
    → fill clk_gate{reg, bit_idx, flags, lock, hw.init}
    → if (dev || !np): clk_hw_register(dev, &gate->hw)
      else if (np):    of_clk_hw_register(np, &gate->hw)
        → CCF: __clk_register()
          → allocates clk_core
          → links parent
          → sets hw->init = NULL
          → adds to clock tree
    → return &gate->hw
```

---

### استراتيجية الـ Locking

الـ clk_gate بيتعامل مع مستويين من الـ locking:

#### 1. الـ prepare_lock (mutex — في الـ CCF)

- بيحمي `prepare_count` وكل العمليات اللي ممكن تنام
- الـ gate ما بيشوفهوش مباشرةً — الـ CCF هو اللي بيمسك المطرح

#### 2. الـ enable_lock / gate->lock (spinlock — irq-safe)

| السيناريو | الـ Lock المستخدم | المكان |
|-----------|-----------------|-------|
| تعديل الـ register | `gate->lock` (spinlock) | `clk_gate_endisable()` |
| الـ register يدعم HIWORD_MASK | `gate->lock` ممكن يكون NULL | single atomic write → لا حاجة للـ read-modify-write |
| قراءة الحالة فقط | لا lock | `clk_gate_is_enabled()` |

**ترتيب الـ locks:**
```
prepare_mutex (CCF global)
    └── enable_lock (CCF global spinlock)
            └── gate->lock (per-register spinlock)
```

**ملحوظة مهمة:** لو `gate->lock == NULL` الكود بيستخدم `__acquire/__release` وهي annotations فقط للـ sparse checker، مش actual locking — معناها إن الـ caller بيتكفّل بالـ synchronization (مثلاً في حالة `CLK_GATE_HIWORD_MASK` اللي بيعمل atomic write بدون حاجة لـ RMW).

#### جدول: مين بيحمي إيه

| Resource | Lock | السبب |
|----------|------|-------|
| `gate->reg` read-modify-write | `gate->lock` spinlock + `irqsave` | ممكن يتشارك مع interrupt handlers |
| `gate->reg` HIWORD_MASK write | لا lock (أو NULL) | الـ hardware نفسه بيعمل atomic update |
| `clk_core` internals | CCF prepare_mutex / enable_lock | مش شغل الـ gate driver |
| `devres` list | device lock (داخل devres subsystem) | تلقائي |
## Phase 4: شرح الـ Functions

---

### ملخص الـ Functions والـ APIs — Cheatsheet

#### Runtime Ops (clk_ops callbacks)

| Function | Signature المختصرة | الغرض |
|---|---|---|
| `clk_gate_enable` | `int (struct clk_hw *)` | تفعيل الـ gate clock عبر set bit |
| `clk_gate_disable` | `void (struct clk_hw *)` | تعطيل الـ gate clock عبر clear bit |
| `clk_gate_is_enabled` | `int (struct clk_hw *)` | قراءة الـ hardware state مباشرة |

#### Internal Helpers

| Function | النوع | الغرض |
|---|---|---|
| `clk_gate_endisable` | `static void` | الـ core logic لتغيير حالة الـ gate |
| `clk_gate_readl` | `static inline u32` | قراءة الـ register مع دعم big-endian |
| `clk_gate_writel` | `static inline void` | كتابة الـ register مع دعم big-endian |

#### Registration APIs

| Function/Macro | المستخدم الموصى به | الغرض |
|---|---|---|
| `__clk_hw_register_gate` | internal / platform drivers | تسجيل gate clock بـ `clk_hw` interface |
| `clk_hw_register_gate` (macro) | platform drivers — الأشيع | wrapper على `__clk_hw_register_gate` |
| `clk_hw_register_gate_parent_hw` (macro) | عند معرفة الـ parent بـ `clk_hw*` | wrapper بـ parent_hw |
| `clk_hw_register_gate_parent_data` (macro) | عند استخدام `clk_parent_data` | wrapper بـ parent_data |
| `clk_register_gate` | legacy drivers | تسجيل بـ `struct clk*` القديم |
| `__devm_clk_hw_register_gate` | devm-based drivers | نفس التسجيل لكن مع resource management |
| `devm_clk_hw_register_gate` (macro) | الأفضل في الكود الجديد | wrapper على `__devm_clk_hw_register_gate` |

#### Cleanup APIs

| Function | الغرض |
|---|---|
| `clk_hw_unregister_gate` | إلغاء تسجيل والـ kfree — الحديث |
| `clk_unregister_gate` | إلغاء تسجيل legacy interface |
| `devm_clk_hw_release_gate` | devres callback — لا يُستدعى مباشرة |

---

### Group 1: الـ I/O Helpers

هذه الـ functions الـ inline هي abstraction layer بسيطة فوق `readl`/`writel` القياسيين، هدفها الوحيد هو دعم الـ hardware التي تستخدم big-endian register layout بدلاً من little-endian.

---

#### `clk_gate_readl`

```c
static inline u32 clk_gate_readl(struct clk_gate *gate)
```

بتقرأ الـ 32-bit gate register. لو الـ `CLK_GATE_BIG_ENDIAN` flag مضبوط بتستخدم `ioread32be`، غير كده بتستخدم `readl` العادي.

**Parameters:**
- `gate` — pointer لـ `struct clk_gate` اللي فيه الـ `reg` وكمان الـ `flags`

**Return:** قيمة الـ register كـ `u32`

**Key details:**
- الـ `CLK_GATE_BIG_ENDIAN` flag بيتحكم في الـ byte-swap تلقائياً
- الفصل ده مهم على SoCs زي بعض MIPS/PowerPC platforms

---

#### `clk_gate_writel`

```c
static inline void clk_gate_writel(struct clk_gate *gate, u32 val)
```

بتكتب الـ 32-bit value على الـ gate register. نفس منطق الـ endianness زي `clk_gate_readl`.

**Parameters:**
- `gate` — pointer لـ `struct clk_gate`
- `val` — القيمة الكاملة للـ register اللي هتتكتب

**Return:** void

**Key details:**
- بتُستدعى دايماً بعد تعديل الـ bit value في `clk_gate_endisable` — مش بتعمل read-modify-write بنفسها إلا لو الـ caller جهّز القيمة

---

### Group 2: الـ Core Logic (Enable/Disable)

الـ function الجوهرية في الملف كله. كل حاجة تانية بتتفرع منها.

---

#### `clk_gate_endisable`

```c
static void clk_gate_endisable(struct clk_hw *hw, int enable)
```

بتعمل enable أو disable للـ gate clock عن طريق تعديل bit واحد في الـ hardware register. بتتعامل مع ثلاث حالات مختلفة: الـ standard gate، الـ inverted gate (SET_TO_DISABLE)، والـ HIWORD_MASK gate.

**Parameters:**
- `hw` — الـ `clk_hw` handle اللي بنتحول منه لـ `clk_gate` عبر `to_clk_gate()`
- `enable` — 1 = enable, 0 = disable

**Return:** void

**Key details — Locking:**
- لو `gate->lock != NULL` → بيستخدم `spin_lock_irqsave` / `spin_unlock_irqrestore` (IRQ-safe)
- لو `gate->lock == NULL` → بيستخدم `__acquire` / `__release` لـ lockdep annotation بس بدون فعلي locking — ده مقبول لو الـ caller بيضمن exclusive access

**Key details — Logic:**

الـ XOR formula هي:
```
set = (CLK_GATE_SET_TO_DISABLE ? 1 : 0) XOR enable
```

| CLK_GATE_SET_TO_DISABLE | enable | set (bit action) |
|---|---|---|
| 0 | 1 (enable) | 1 → set bit |
| 0 | 0 (disable) | 0 → clear bit |
| 1 | 1 (enable) | 0 → clear bit |
| 1 | 0 (disable) | 1 → set bit |

**Key details — HIWORD_MASK:**

لو `CLK_GATE_HIWORD_MASK` مضبوط، الـ register مقسوم: lower 16-bit للـ gate data، وupper 16-bit للـ write-mask. مش بيعمل read-modify-write هنا، بيبني القيمة من الأول:
```c
reg = BIT(gate->bit_idx + 16);   // set the mask bit in high word
if (set)
    reg |= BIT(gate->bit_idx);   // optionally set the data bit in low word
```
ده pattern شائع في Rockchip و HiSilicon SoCs.

**Pseudocode flow:**

```
clk_gate_endisable(hw, enable):
    gate = to_clk_gate(hw)
    set = (SET_TO_DISABLE flag ? 1 : 0) XOR enable

    lock(gate->lock)  // spin_lock_irqsave أو __acquire

    if HIWORD_MASK:
        reg = BIT(bit_idx + 16)         // mask high word
        if set: reg |= BIT(bit_idx)     // data low word
    else:
        reg = readl(gate->reg)
        if set:  reg |=  BIT(bit_idx)
        else:    reg &= ~BIT(bit_idx)

    writel(gate->reg, reg)
    unlock(gate->lock)
```

**Who calls it:** فقط `clk_gate_enable` و`clk_gate_disable` — كلاهم thin wrappers.

---

### Group 3: الـ clk_ops Callbacks

الـ three functions دول هما اللي بتتسجّل في `clk_gate_ops` وبيستدعيهم الـ CCF (Common Clock Framework) مباشرة.

---

#### `clk_gate_enable`

```c
static int clk_gate_enable(struct clk_hw *hw)
```

**الـ clk_ops.enable** callback للـ gate clock. بتستدعي `clk_gate_endisable(hw, 1)` وبترجع 0 دايماً.

**Parameters:**
- `hw` — الـ hardware handle من الـ CCF

**Return:** 0 دايماً (no failure path)

**Key details:**
- بتُستدعى من الـ CCF بعد `clk_prepare` بـ enable_lock مضبوط (spinlock)
- **يجب ألا تنام** — atomic context
- الـ CCF بيعمل enable counting، فمش بتتصل إلا لو العدد كان صفر قبلها

**Who calls it:** الـ CCF عند `clk_enable()` API call من consumer

---

#### `clk_gate_disable`

```c
static void clk_gate_disable(struct clk_hw *hw)
```

**الـ clk_ops.disable** callback. بتستدعي `clk_gate_endisable(hw, 0)`.

**Parameters:**
- `hw` — الـ hardware handle

**Return:** void

**Key details:**
- atomic context — no sleep
- الـ CCF بيستدعيها بس لما enable count يوصل لصفر

**Who calls it:** الـ CCF عند `clk_disable()` API call من consumer

---

#### `clk_gate_is_enabled`

```c
int clk_gate_is_enabled(struct clk_hw *hw)
```

بتقرأ الـ hardware register مباشرة وبترجع الحالة الفعلية للـ gate. بتتعامل مع الـ SET_TO_DISABLE polarity عن طريق XOR.

**Parameters:**
- `hw` — الـ hardware handle

**Return:** 1 لو الـ clock enabled، 0 لو disabled

**Key details:**
```c
reg = clk_gate_readl(gate);
if (CLK_GATE_SET_TO_DISABLE)
    reg ^= BIT(bit_idx);   // flip polarity
reg &= BIT(bit_idx);       // isolate the bit
return reg ? 1 : 0;
```

- مش بتاخد lock لأنها read-only — ده مقبول طالما الـ read أتومية على 32-bit aligned register
- `EXPORT_SYMBOL_GPL` — متاحة خارج الـ module لو محتاجها platform driver مباشرة
- الـ CCF بتستدعيها عند `clk_is_enabled()` وكمان عند boot-time لتحديد الـ initial state

**Who calls it:** الـ CCF runtime + platform drivers اللي بتعمل direct status check

---

### Group 4: الـ clk_gate_ops Table

```c
const struct clk_ops clk_gate_ops = {
    .enable     = clk_gate_enable,
    .disable    = clk_gate_disable,
    .is_enabled = clk_gate_is_enabled,
};
EXPORT_SYMBOL_GPL(clk_gate_ops);
```

الـ ops table ده هو الـ interface الرسمي بين الـ gate implementation والـ CCF. لاحظ إنه **مش بيحدد** `.recalc_rate` أو `.set_rate` أو `.set_parent` — الـ gate clock بيورث كل ده من parent بدون تعديل. الـ `EXPORT_SYMBOL_GPL` بيخلي الـ platform drivers تقدر تستخدمه مباشرة لو هي بتعمل custom struct بس نفس الـ gate logic.

---

### Group 5: الـ Registration Functions

---

#### `__clk_hw_register_gate`

```c
struct clk_hw *__clk_hw_register_gate(struct device *dev,
        struct device_node *np, const char *name,
        const char *parent_name, const struct clk_hw *parent_hw,
        const struct clk_parent_data *parent_data,
        unsigned long flags,
        void __iomem *reg, u8 bit_idx,
        u8 clk_gate_flags, spinlock_t *lock)
```

الـ canonical registration function للـ gate clock. بتعمل `kzalloc` للـ `struct clk_gate`، بتملّي الـ `clk_init_data`، وبتستدعي `clk_hw_register` أو `of_clk_hw_register` حسب الـ context.

**Parameters:**
- `dev` — الـ `struct device*` للـ driver (ممكن NULL لو OF-only)
- `np` — الـ `device_node*` للـ DT registration (ممكن NULL)
- `name` — اسم الـ clock في الـ framework
- `parent_name` — اسم الـ parent كـ string (legacy way)
- `parent_hw` — pointer مباشر للـ parent `clk_hw` (أفضل للـ internal clocks)
- `parent_data` — `struct clk_parent_data*` (الطريقة الحديثة للـ mixed parents)
- `flags` — الـ framework-level flags (مثل `CLK_SET_RATE_PARENT`)
- `reg` — الـ MMIO address للـ gate register
- `bit_idx` — رقم الـ bit (0-based) المسؤول عن الـ gating
- `clk_gate_flags` — gate-specific flags (`CLK_GATE_SET_TO_DISABLE`, `CLK_GATE_HIWORD_MASK`, `CLK_GATE_BIG_ENDIAN`)
- `lock` — shared `spinlock_t*` للـ registers المشتركة بين أكثر من clock (ممكن NULL)

**Return:** `struct clk_hw*` أو `ERR_PTR(errno)` في حالة الخطأ

**Key details:**
- لو `CLK_GATE_HIWORD_MASK` و `bit_idx > 15` → بترجع `ERR_PTR(-EINVAL)` فوراً لأن الـ high-word mask بس بيغطي bits 0–15
- بس **واحد** من `parent_name / parent_hw / parent_data` المفروض يكون غير NULL — الثلاثة مع بعض ممكن يسبب undefined behavior في الـ CCF
- لو `dev` موجود → `clk_hw_register(dev, hw)` (device-managed internally via devres)
- لو `np` موجود و `dev` NULL → `of_clk_hw_register(np, hw)` (DT-based)
- عند الفشل بتعمل `kfree(gate)` وبترجع الـ ERR_PTR

**Pseudocode flow:**

```
__clk_hw_register_gate(...):
    if HIWORD_MASK && bit_idx > 15: return ERR_PTR(-EINVAL)

    gate = kzalloc(sizeof(*gate))
    if !gate: return ERR_PTR(-ENOMEM)

    init.name = name
    init.ops  = &clk_gate_ops
    init.flags = flags
    // set exactly one of: parent_names, parent_hws, parent_data
    init.num_parents = (any parent given ? 1 : 0)

    gate->reg = reg
    gate->bit_idx = bit_idx
    gate->flags = clk_gate_flags
    gate->lock  = lock
    gate->hw.init = &init

    if dev || !np:  ret = clk_hw_register(dev, &gate->hw)
    else:           ret = of_clk_hw_register(np, &gate->hw)

    if ret: kfree(gate); return ERR_PTR(ret)
    return &gate->hw
```

**Who calls it:** الـ public macros (`clk_hw_register_gate`, `clk_hw_register_gate_parent_hw`, `clk_hw_register_gate_parent_data`) و`__devm_clk_hw_register_gate`

---

#### `clk_register_gate`

```c
struct clk *clk_register_gate(struct device *dev, const char *name,
        const char *parent_name, unsigned long flags,
        void __iomem *reg, u8 bit_idx,
        u8 clk_gate_flags, spinlock_t *lock)
```

الـ **legacy** registration API اللي بترجع `struct clk*` بدل `struct clk_hw*`. موجودة للـ backward compatibility مع الكود القديم.

**Parameters:** نفس `__clk_hw_register_gate` بس بدون `np`، `parent_hw`، `parent_data`

**Return:** `struct clk*` أو `ERR_PTR`

**Key details:**
- بتستدعي `clk_hw_register_gate` داخلياً وبتجيب `hw->clk`
- **لا تُستخدم في الكود الجديد** — الـ `clk_hw` API هو الصح دلوقتي
- `EXPORT_SYMBOL_GPL`

**Who calls it:** legacy platform drivers اللي لسه مش migrate لـ `clk_hw` API

---

#### `__devm_clk_hw_register_gate`

```c
struct clk_hw *__devm_clk_hw_register_gate(struct device *dev,
        struct device_node *np, const char *name,
        const char *parent_name, const struct clk_hw *parent_hw,
        const struct clk_parent_data *parent_data,
        unsigned long flags,
        void __iomem *reg, u8 bit_idx,
        u8 clk_gate_flags, spinlock_t *lock)
```

نفس `__clk_hw_register_gate` بالظبط لكنها بتضيف **devres** management — يعني الـ cleanup بيحصل أوتوماتيك لما الـ `device` يتعمله `remove` أو `unbind`.

**Parameters:** نفس `__clk_hw_register_gate` تماماً

**Return:** `struct clk_hw*` أو `ERR_PTR`

**Key details:**
- بتعمل `devres_alloc(devm_clk_hw_release_gate, ...)` الأول عشان تخزّن الـ `clk_hw*`
- لو التسجيل نجح → `devres_add(dev, ptr)` يربط الـ resource بالـ device lifetime
- لو التسجيل فشل → `devres_free(ptr)` فوراً
- الـ cleanup callback هو `devm_clk_hw_release_gate` اللي بيستدعي `clk_hw_unregister_gate`

**Pseudocode flow:**

```
__devm_clk_hw_register_gate(...):
    ptr = devres_alloc(devm_clk_hw_release_gate, sizeof(clk_hw*))
    if !ptr: return ERR_PTR(-ENOMEM)

    hw = __clk_hw_register_gate(...)   // do actual registration

    if !IS_ERR(hw):
        *ptr = hw
        devres_add(dev, ptr)
    else:
        devres_free(ptr)

    return hw
```

**Who calls it:** `devm_clk_hw_register_gate` macro وكل macros الـ devm variants (parent_hw, parent_data)

---

### Group 6: الـ Cleanup Functions

---

#### `clk_hw_unregister_gate`

```c
void clk_hw_unregister_gate(struct clk_hw *hw)
```

بتعمل unregister للـ gate clock من الـ CCF وبتحرر الـ `struct clk_gate` allocation.

**Parameters:**
- `hw` — الـ `clk_hw*` اللي اترجع من التسجيل

**Return:** void

**Key details:**
- بتستدعي `clk_hw_unregister(hw)` الأول عشان يتشال من الـ CCF tree
- بعدين `kfree(gate)` عن طريق `to_clk_gate(hw)` — **لازم** الترتيب ده لأن بعد الـ kfree الـ gate pointer يبقى dangling
- **لا تُستدعى** مباشرة لو استخدمت `devm_clk_hw_register_gate` — الـ devres بيعملها أوتوماتيك

**Who calls it:** platform drivers في `remove()` callback، أو `devm_clk_hw_release_gate` callback

---

#### `clk_unregister_gate`

```c
void clk_unregister_gate(struct clk *clk)
```

الـ legacy cleanup المقابل لـ `clk_register_gate`.

**Parameters:**
- `clk` — الـ `struct clk*` legacy handle

**Return:** void

**Key details:**
- بتجيب الـ `clk_hw` عبر `__clk_get_hw(clk)` — لو NULL بترجع بهدوء
- بتستدعي `clk_unregister(clk)` ثم `kfree(gate)`

**Who calls it:** legacy platform drivers في `remove()`

---

#### `devm_clk_hw_release_gate`

```c
static void devm_clk_hw_release_gate(struct device *dev, void *res)
```

**devres release callback** — مش للاستخدام المباشر. بتُستدعى أوتوماتيك من الـ devres framework لما الـ device يتحرر.

**Parameters:**
- `dev` — الـ device (unused في الـ implementation)
- `res` — pointer لـ `struct clk_hw*` المخزّنة في الـ devres

**Return:** void

**Key details:**
- بتعمل dereference للـ `res` كـ `struct clk_hw**` وبتستدعي `clk_hw_unregister_gate`
- الـ devres framework بيتعامل مع الـ `free` للـ `res` نفسه بعدين

---

### ملاحظات عامة على التصميم

**الـ struct clk_gate — مراجعة سريعة:**

```c
struct clk_gate {
    struct clk_hw hw;       // MUST be first — to_clk_gate() depends on it
    void __iomem *reg;      // gate control register
    u8 bit_idx;             // which bit (0–31, or 0–15 for HIWORD_MASK)
    u8 flags;               // CLK_GATE_SET_TO_DISABLE | HIWORD_MASK | BIG_ENDIAN
    spinlock_t *lock;       // optional shared lock
};
```

**الـ flags المتاحة:**

| Flag | Value | المعنى |
|---|---|---|
| `CLK_GATE_SET_TO_DISABLE` | BIT(0) | الـ bit=1 يعني disabled (inverted polarity) |
| `CLK_GATE_HIWORD_MASK` | BIT(1) | الـ high 16-bit = write mask (Rockchip style) |
| `CLK_GATE_BIG_ENDIAN` | BIT(2) | الـ register يُقرأ/يُكتب بـ big-endian |

**تدفق الـ call chain كامل:**

```
Consumer: clk_enable(clk)
    └─> CCF: clk_core_enable()
          └─> clk_gate_enable(hw)
                └─> clk_gate_endisable(hw, 1)
                      ├─> spin_lock_irqsave()
                      ├─> clk_gate_readl()   [or skip for HIWORD]
                      ├─> set/clear bit
                      ├─> clk_gate_writel()
                      └─> spin_unlock_irqrestore()
```
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. مدخلات الـ debugfs

الـ **CCF (Common Clock Framework)** بيعمل directory تلقائيًا في `/sys/kernel/debug/clk/` لكل clock مسجّل، بما فيها الـ gate clocks.

```bash
# شوف كل الـ gate clocks المتاحة
ls /sys/kernel/debug/clk/

# اقرأ معلومات clock بعينه (استبدل <clk_name> باسم الـ clock)
cat /sys/kernel/debug/clk/<clk_name>/clk_enable_count
cat /sys/kernel/debug/clk/<clk_name>/clk_prepare_count
cat /sys/kernel/debug/clk/<clk_name>/clk_flags
cat /sys/kernel/debug/clk/<clk_name>/clk_rate
cat /sys/kernel/debug/clk/<clk_name>/clk_parent
```

| المدخل | المعنى |
|---|---|
| `clk_enable_count` | عدد مرات `clk_enable()` بدون مقابلها `clk_disable()` — لو > 0 يبقى مفعّل |
| `clk_prepare_count` | نفس الفكرة لكن للـ prepare layer |
| `clk_flags` | الـ flags المضبوطة على الـ clock (مثلاً `CLK_IGNORE_UNUSED`) |
| `clk_rate` | معدل التردد المورث من الـ parent (gate لا تغيّره) |

```bash
# طباعة شجرة كل الـ clocks بالتسلسل الهرمي
cat /sys/kernel/debug/clk/clk_summary

# مثال على الإخراج المتوقع:
#   clock              enable_cnt  prepare_cnt  protect_cnt  rate
#   my_gate_clk                1            1            0  100000000
```

---

#### 2. مدخلات الـ sysfs

الـ sysfs لا توفر نفس تفاصيل الـ debugfs للـ clocks، لكن لو الـ clock مرتبط بـ device:

```bash
# شوف الـ clocks التي يستخدمها device معين
ls /sys/bus/platform/devices/<device_name>/

# لو الـ driver يستخدم devm_clk_hw_register_gate، الـ device resource بتظهر
cat /sys/bus/platform/devices/<device_name>/of_node/clocks
```

---

#### 3. الـ ftrace — Tracepoints والـ Events

**الـ CCF** فيه tracepoints مباشرة للـ enable/disable:

```bash
# فعّل الـ tracing للـ clock events
cd /sys/kernel/debug/tracing

# شوف الـ events المتاحة
grep -r "clk" available_events

# فعّل event الـ enable
echo 1 > events/clk/clk_enable/enable
echo 1 > events/clk/clk_enable_complete/enable
echo 1 > events/clk/clk_disable/enable
echo 1 > events/clk/clk_disable_complete/enable
echo 1 > events/clk/clk_prepare/enable
echo 1 > events/clk/clk_unprepare/enable

# ابدأ الـ trace
echo 1 > tracing_on
# شغّل الـ device أو الـ operation اللي بتتابعها
echo 0 > tracing_on

# اقرأ النتيجة
cat trace
```

مثال إخراج:
```
# TASK-PID    CPU  TIMESTAMP  FUNCTION
  kworker/0-42  [000]  123.456  clk_enable: my_gate_clk
  kworker/0-42  [000]  123.457  clk_enable_complete: my_gate_clk
```

```bash
# تتبع دالة clk_gate_endisable تحديدًا باستخدام function tracer
echo function > current_tracer
echo clk_gate_enable > set_ftrace_filter
echo clk_gate_disable >> set_ftrace_filter
echo clk_gate_is_enabled >> set_ftrace_filter
echo 1 > tracing_on
cat trace
```

---

#### 4. الـ printk والـ Dynamic Debug

```bash
# فعّل dynamic debug لكل ملفات الـ clk subsystem
echo "file drivers/clk/* +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل لـ clk-gate.c تحديدًا
echo "file drivers/clk/clk-gate.c +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل مع line numbers وأسماء الدوال
echo "file drivers/clk/clk-gate.c +pflT" > /sys/kernel/debug/dynamic_debug/control

# شوف الـ dynamic debug الحالي
cat /sys/kernel/debug/dynamic_debug/control | grep clk-gate
```

لو محتاج تضيف `pr_debug()` يدوي في `clk_gate_endisable()`:

```c
static void clk_gate_endisable(struct clk_hw *hw, int enable)
{
    struct clk_gate *gate = to_clk_gate(hw);
    /* Temporary debug — remove before upstream submission */
    pr_debug("clk_gate_endisable: name=%s enable=%d bit=%d reg_before=0x%08x\n",
             clk_hw_get_name(hw), enable, gate->bit_idx,
             clk_gate_readl(gate));
    /* ... rest of function ... */
}
```

```bash
# شوف رسائل الـ kernel
dmesg | grep -E "clk_gate|clk-gate"
journalctl -k | grep clk
```

---

#### 5. الـ Kernel Config Options للـ Debugging

| الـ Config | الغرض |
|---|---|
| `CONFIG_COMMON_CLK` | لازم يكون مفعّل — هو اللي يجيب CCF |
| `CONFIG_CLK_DEBUG` | يفعّل الـ debugfs entries للـ CCF كاملة |
| `CONFIG_DEBUG_FS` | شرط لوجود `/sys/kernel/debug/clk/` |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل `pr_debug()` runtime |
| `CONFIG_KALLSYMS` | يخلي stack traces مفهومة |
| `CONFIG_LOCKDEP` | يكشف lockdep violations في `spinlock_t *lock` |
| `CONFIG_DEBUG_SPINLOCK` | يكشف spinlock misuse في `clk_gate_endisable()` |
| `CONFIG_DEBUG_ATOMIC_SLEEP` | يكشف لو حد نام في atomic context أثناء enable |
| `CONFIG_PROVE_LOCKING` | فوق LOCKDEP — يثبت صحة ترتيب الـ locking |
| `CONFIG_KASAN` | يكشف memory errors في `kzalloc` / `kfree` للـ gate struct |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "CONFIG_(COMMON_CLK|CLK_DEBUG|DEBUG_FS|DYNAMIC_DEBUG|LOCKDEP|DEBUG_SPINLOCK)"
```

---

#### 6. أدوات خاصة بالـ Subsystem

```bash
# clk_summary — أشمل أداة لرؤية حالة كل الـ clocks
cat /sys/kernel/debug/clk/clk_summary | column -t

# تصفية gate clocks فقط (بتبص على clk_enable_count > 0)
awk 'NR==1 || $2 > 0' /sys/kernel/debug/clk/clk_summary

# لو الـ platform يستخدم devlink (مثلاً DSA/network)، مش متعلق بالـ gate clock مباشرة
# لكن لو في reset controller مرتبط:
cat /sys/kernel/debug/reset/reset_summary 2>/dev/null

# لو تستخدم regmap-based gate clock، اقرأ الـ register مباشرة
cat /sys/kernel/debug/regmap/<device>/registers | grep <offset_hex>
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `gate bit exceeds LOWORD field` | `bit_idx > 15` مع `CLK_GATE_HIWORD_MASK` | اجعل `bit_idx <= 15` أو لا تستخدم `CLK_GATE_HIWORD_MASK` |
| `clk: couldn't get clock` | الـ clock لم يُسجَّل بعد | تأكد من ترتيب `probe()` وأن الـ parent clock مسجّل قبله |
| `clk: failed to register ...` | `clk_hw_register()` فشلت | تحقق من `dmesg` للـ errno — غالبًا `-EEXIST` (اسم مكرر) أو `-ENOMEM` |
| `WARNING: ... prepare_count > 0` | `clk_prepare` بدون `clk_unprepare` | الـ driver لم يتوازن في استدعاءاته |
| `clk: ... is enabled but unused` | الـ clock مفعّل لكن ما فيش consumer | أضف `CLK_IGNORE_UNUSED` مؤقتًا للـ debug، ثم ابحث عن الـ consumer المفقود |
| `-EPROBE_DEFER` | الـ parent clock مش جاهز | الـ framework بيعيد المحاولة تلقائيًا، تأكد من `init order` |
| `Bad spinlock magic` | الـ spinlock خراب — `gate->lock` corrupt | `CONFIG_DEBUG_SPINLOCK` + تحقق من الـ allocation |

---

#### 8. نقاط استراتيجية لـ dump_stack() / WARN_ON()

```c
static void clk_gate_endisable(struct clk_hw *hw, int enable)
{
    struct clk_gate *gate = to_clk_gate(hw);

    /* Warn if called with invalid bit index */
    WARN_ON(gate->bit_idx >= 32);

    /* Warn if HIWORD_MASK used with bit_idx > 15 */
    WARN_ON((gate->flags & CLK_GATE_BIG_ENDIAN) && !IS_ALIGNED((unsigned long)gate->reg, 4));

    /* Debug: dump stack when unexpected enable/disable happens */
    if (unlikely(enable == clk_gate_is_enabled(hw))) {
        pr_warn("clk_gate: redundant %s on %s\n",
                enable ? "enable" : "disable",
                clk_hw_get_name(hw));
        dump_stack();
    }
    /* ... */
}

int clk_gate_is_enabled(struct clk_hw *hw)
{
    struct clk_gate *gate = to_clk_gate(hw);

    /* Verify register is mapped */
    WARN_ON(!gate->reg);
    /* ... */
}
```

---

### Hardware Level

#### 1. التحقق من تطابق حالة الـ Hardware مع حالة الـ Kernel

```bash
# اقرأ clk_enable_count من debugfs
ENABLE_CNT=$(cat /sys/kernel/debug/clk/<clk_name>/clk_enable_count)
echo "Kernel thinks clock is: $([ $ENABLE_CNT -gt 0 ] && echo ENABLED || echo DISABLED)"

# قارن بقراءة الـ register المباشر (أنظر القسم التالي)
# لو الـ hardware قال disabled والـ kernel قال enabled → مشكلة sync
```

الـ `clk_gate_is_enabled()` بتقرأ الـ register من الـ MMIO مباشرة في كل مرة تُستدعى، فمش بتستخدم cache — ده بيسهّل التحقق.

---

#### 2. تقنيات الـ Register Dump

```bash
# باستخدام devmem2 (لازم تثبّته على الـ target)
# استبدل PHYS_ADDR بالعنوان الفيزيائي للـ gate register
devmem2 0x<PHYS_ADDR> w

# مثال: لو الـ gate register على عنوان 0xFF760000
devmem2 0xFF760000 w
# الإخراج: 0xFF760000: 0x00000001  ← البت 0 = 1 → clock enabled

# باستخدام /dev/mem (لازم CONFIG_DEVMEM=y وlazy_irqfixup أو devmem_is_allowed)
python3 -c "
import mmap, struct
with open('/dev/mem', 'r+b') as f:
    m = mmap.mmap(f.fileno(), 4096, offset=0xFF760000 & ~0xFFF)
    offset = 0xFF760000 & 0xFFF
    val = struct.unpack('<I', m[offset:offset+4])[0]
    print(f'Register value: 0x{val:08X}')
    m.close()
"

# باستخدام io utility (من package devmem أو busybox)
io -4 -r 0xFF760000
```

تفسير القراءة بناءً على الـ flags:

```
CLK_GATE_SET_TO_DISABLE = 0, bit_idx = 3:
  Value = 0x00000008 → bit 3 = 1 → clock ENABLED

CLK_GATE_SET_TO_DISABLE = 1, bit_idx = 3:
  Value = 0x00000008 → bit 3 = 1 → clock DISABLED (منعكس)

CLK_GATE_HIWORD_MASK, bit_idx = 3:
  Write: upper 16 = mask bits, lower 16 = value bits
  Read: بص بس على bit 3 في الـ lower 16
```

---

#### 3. Logic Analyzer / Oscilloscope

لو بتـ debug gate clock على signal فعلي:

- **Oscilloscope**: وصّل probe على output الـ clock signal.
  - لو الـ gate مفعّل: شوف signal نظيف بالتردد المتوقع.
  - لو الـ gate معطّل: شوف DC level (0 أو VDD حسب الـ pull).

- **Logic Analyzer**: فيد أكثر للـ digital clocks.
  - راقب الـ register write transaction على الـ bus (AHB/APB).
  - فعّل trigger على العنوان الفيزيائي للـ gate register.
  - قارن التوقيت بين الـ write للـ register والـ clock output.

```
نصيحة: استخدم ftrace لتسجيل timestamp الـ write، ثم قارنه
        بالـ logic analyzer capture لنفس اللحظة.
```

---

#### 4. أعطال الـ Hardware الشائعة وأنماط الـ Log

| عطل الـ Hardware | ما يظهر في الـ Kernel Log | التحقيق |
|---|---|---|
| الـ clock لا يُعطّل فعليًا بعد `clk_disable()` | لا تظهر رسائل خطأ — الـ kernel يحسبها معطّلة | قرأ الـ register بـ devmem2، وصّل oscilloscope |
| خطأ في الـ polarity (SET_TO_DISABLE مضبوط غلط) | الـ device لا يعمل بعد `clk_enable()` | تحقق من الـ datasheet لتأكيد bit polarity |
| الـ register يحتاج HIWORD_MASK لكن مش مضبوط | write على الـ register بيخرب bits تانية | قرأ الـ register قبل وبعد الـ write |
| Endianness خاطئ (BIG_ENDIAN flag ناقص) | الـ gate لا يستجيب، قراءة `is_enabled` دايمًا 0 | قارن قراءة `devmem2` مع قراءة الـ kernel |
| power domain للـ gate register معطّل | `readl()` يرجع 0xDEADBEEF أو يسبب Data Abort | تفعيل الـ power domain قبل register access |

---

#### 5. تـ Debugging الـ Device Tree

```bash
# تحقق من الـ DT node المسجّل للـ clock controller
find /sys/firmware/devicetree/base -name "compatible" -exec grep -l "clk\|clock" {} \;

# اقرأ الـ reg property للـ clock controller (العنوان الفيزيائي)
cat /sys/firmware/devicetree/base/soc/clock-controller@FF760000/reg | od -An -tx4

# تحقق من clocks property في الـ consumer device
cat /sys/firmware/devicetree/base/soc/<consumer_device>/clocks | od -An -tx4

# تحقق من clock-names
cat /sys/firmware/devicetree/base/soc/<consumer_device>/clock-names
```

نموذج DT صحيح لـ gate clock:

```dts
/* Clock controller node */
cru: clock-controller@ff760000 {
    compatible = "vendor,soc-cru";
    reg = <0x0 0xff760000 0x0 0x1000>;
    #clock-cells = <1>;
};

/* Consumer node */
uart0: serial@ff180000 {
    compatible = "ns16550a";
    reg = <0x0 0xff180000 0x0 0x100>;
    clocks = <&cru CLK_UART0>;   /* phandle + clock ID */
    clock-names = "uartclk";
};
```

أخطاء شائعة في الـ DT:

```bash
# لو الـ clock لم يُعثر عليه
dmesg | grep "clk.*not found\|Failed to get.*clock"

# لو الـ #clock-cells خاطئ
dmesg | grep "invalid.*clock\|clock.*invalid"

# تحقق من ترتيب الـ DT nodes (الـ provider لازم يجي قبل الـ consumer)
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A5 "clock-controller"
```

---

### Practical Commands

#### سيناريو 1: التحقق من حالة gate clock

```bash
#!/bin/bash
CLK_NAME="my_uart_clk"

echo "=== Clock Gate Debug: $CLK_NAME ==="
echo "Enable count : $(cat /sys/kernel/debug/clk/$CLK_NAME/clk_enable_count)"
echo "Prepare count: $(cat /sys/kernel/debug/clk/$CLK_NAME/clk_prepare_count)"
echo "Rate         : $(cat /sys/kernel/debug/clk/$CLK_NAME/clk_rate) Hz"
echo "Parent       : $(cat /sys/kernel/debug/clk/$CLK_NAME/clk_parent 2>/dev/null || echo N/A)"
echo "Flags        : $(cat /sys/kernel/debug/clk/$CLK_NAME/clk_flags)"
```

مثال إخراج:
```
=== Clock Gate Debug: my_uart_clk ===
Enable count : 1
Prepare count: 1
Rate         : 100000000 Hz
Parent       : osc24M
Flags        : 0
```

---

#### سيناريو 2: تتبع enable/disable بالـ ftrace

```bash
#!/bin/bash
TRACE_DIR=/sys/kernel/debug/tracing

# صفّر الـ trace القديم
echo > $TRACE_DIR/trace

# فعّل events الـ CCF
for event in clk_enable clk_enable_complete clk_disable clk_disable_complete; do
    echo 1 > $TRACE_DIR/events/clk/$event/enable
done

# فعّل الـ tracing
echo 1 > $TRACE_DIR/tracing_on

# شغّل العملية اللي بتتابعها
echo "Run your test here..."
sleep 5

# أوقف وأقرأ
echo 0 > $TRACE_DIR/tracing_on
cat $TRACE_DIR/trace | grep -E "clk_enable|clk_disable"

# نظّف
for event in clk_enable clk_enable_complete clk_disable clk_disable_complete; do
    echo 0 > $TRACE_DIR/events/clk/$event/enable
done
```

مثال إخراج:
```
  kworker/u4:1-85  [001]  456.123456: clk_enable: my_uart_clk
  kworker/u4:1-85  [001]  456.123489: clk_enable_complete: my_uart_clk
  kworker/u4:1-85  [001]  457.654321: clk_disable: my_uart_clk
  kworker/u4:1-85  [001]  457.654340: clk_disable_complete: my_uart_clk
```

التفسير: الـ `enable_complete` يجي بعد `enable` — لو ما جاش يبقى في deadlock أو error في الـ ops.

---

#### سيناريو 3: قراءة register الـ gate مباشرة وتفسيره

```bash
#!/bin/bash
# استبدل بالقيم الحقيقية من الـ datasheet
PHYS_ADDR=0xFF760034   # عنوان الـ gate register
BIT_IDX=3              # رقم الـ bit

if ! command -v devmem2 &>/dev/null; then
    echo "devmem2 not found, trying /dev/mem..."
    python3 -c "
import mmap, struct, sys
addr = $PHYS_ADDR
page_addr = addr & ~0xFFF
offset = addr & 0xFFF
with open('/dev/mem', 'rb') as f:
    m = mmap.mmap(f.fileno(), 4096, mmap.MAP_PRIVATE, offset=page_addr)
    val = struct.unpack('<I', m[offset:offset+4])[0]
    bit = (val >> $BIT_IDX) & 1
    print(f'Register 0x{addr:08X} = 0x{val:08X}')
    print(f'Bit $BIT_IDX = {bit} -> Clock is {\"ENABLED\" if bit else \"DISABLED\"}')
    m.close()
"
else
    VAL=$(devmem2 0x$PHYS_ADDR w 2>/dev/null | grep "Read" | awk '{print $NF}')
    echo "Register 0x$PHYS_ADDR = $VAL"
    BIT_VAL=$(python3 -c "print(($VAL >> $BIT_IDX) & 1)")
    echo "Bit $BIT_IDX = $BIT_VAL -> Clock is $([ $BIT_VAL -eq 1 ] && echo ENABLED || echo DISABLED)"
fi
```

---

#### سيناريو 4: تفعيل dynamic debug وتشغيله

```bash
# تحقق من دعم dynamic debug
grep -q CONFIG_DYNAMIC_DEBUG /proc/config.gz 2>/dev/null && echo "Supported" || echo "Not supported"

# فعّل debug messages لـ clk-gate.c
echo 'file drivers/clk/clk-gate.c +pflmT' > /sys/kernel/debug/dynamic_debug/control

# شوف ما تم تفعيله
grep clk-gate /sys/kernel/debug/dynamic_debug/control

# راقب الـ log في real-time
dmesg -w | grep -E "clk_gate|clk-gate"
```

---

#### سيناريو 5: الكشف عن unused gate clocks

```bash
# الـ clocks التي enable_count=0 وهي مسجّلة — قد تكون مهدرة
awk 'NR>1 && $2==0 && $3==0 {print $1}' /sys/kernel/debug/clk/clk_summary

# الـ clocks التي enable_count>0 لكن الـ device مش شاغلها
# ابحث عن CLK_IGNORE_UNUSED
cat /sys/kernel/debug/clk/clk_summary | awk '$NF ~ /IGNORE_UNUSED/ {print $1, "→ forced ON"}'
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: UART صامت على RK3562 — Industrial Gateway

#### العنوان
**الـ UART مش بيشتغل بعد boot على gateway صناعي بـ RK3562**

#### السياق
شركة بتبني industrial gateway بيستخدم RK3562، الـ gateway بيتواصل مع sensors عن طريق UART2. البورد بتشتغل، الـ kernel بيتبوت، بس الـ UART صامت خالص — مفيش data بتيجي من الـ sensors.

#### المشكلة
الـ engineer فتح الـ serial console وشاف إن `ttyS2` موجود، بس أي write عليه مش بيتبعت. الـ driver مش بيرجع error، الـ hardware موجود في الـ DT، بس الـ UART فعلياً مش شغال.

#### التحليل
الـ engineer راح لـ `debugfs`:

```bash
cat /sys/kernel/debug/clk/uart2_clk/clk_enable_count
# output: 0
cat /sys/kernel/debug/clk/uart2_clk/clk_flags
# CLK_IS_ORPHAN
```

الـ clock مش enabled. راح يشوف الـ gate clock المسؤول في الـ SoC clock driver. المشكلة كانت إن الـ clock اسمه في الـ DT `"uart2"` بس في الـ clock provider مسجّل باسم `"clk_uart2"` — مفيش match، فالـ CCF خلّى الـ gate مش متربط بأي consumer.

في `clk_gate_endisable()`:
```c
// الـ function دي مش بتتكلم أصلاً لأن clk_enable مش بيتكلم
// لأن الـ clk consumer مش لاقي الـ clock بالاسم الصح
static void clk_gate_endisable(struct clk_hw *hw, int enable)
{
    struct clk_gate *gate = to_clk_gate(hw);
    // ...
    clk_gate_writel(gate, reg);  // never reached
}
```

الـ `clk_gate_is_enabled()` لو اتنادت كانت هترجع 0 لأن الـ bit في الـ register لسه cleared.

#### الحل
صحّح اسم الـ clock في الـ Device Tree:

```dts
/* قبل */
uart2: serial@fe670000 {
    clocks = <&cru uart2>, <&cru uart2_sclk>;
    clock-names = "uart2", "baudclk";
};

/* بعد */
uart2: serial@fe670000 {
    clocks = <&cru CLK_UART2>, <&cru CLK_UART2_SRC>;
    clock-names = "uart2", "baudclk";
};
```

للتأكيد بعد الإصلاح:
```bash
cat /sys/kernel/debug/clk/clk_uart2/clk_enable_count
# 1
echo "test" > /dev/ttyS2
# يشتغل
```

#### الدرس المستفاد
**الـ `clk-gate.c` مش مسؤول عن ربط الـ consumer بالـ clock** — ده شغل الـ CCF. لو الاسم غلط في الـ DT، الـ `clk_gate_endisable()` مش هتتكلم خالص ومفيش error واضح. دايماً تتأكد من `clk_flags` في debugfs وتدور على `CLK_IS_ORPHAN`.

---

### السيناريو 2: Kernel Panic عند suspend على STM32MP1 — IoT Sensor Hub

#### العنوان
**Race condition في الـ spinlock أثناء suspend/resume على STM32MP1**

#### السياق
IoT device بيستخدم STM32MP1، فيه I2C bus بيتكلم مع temperature sensors. الـ system بيدخل suspend كل 30 ثانية لتوفير الطاقة. بعد فترة من الاستخدام، الـ kernel بيعمل panic بـ:
```
BUG: spinlock bad magic on CPU#0
```

#### المشكلة
الـ panic بييجي من جوه `clk_gate_endisable()` اللي بتحاول تعمل `spin_lock_irqsave()` على lock فسد.

#### التحليل
في الكود:

```c
static void clk_gate_endisable(struct clk_hw *hw, int enable)
{
    struct clk_gate *gate = to_clk_gate(hw);
    // ...
    if (gate->lock)
        spin_lock_irqsave(gate->lock, flags);  // <-- crash هنا
    else
        __acquire(gate->lock);
    // ...
}
```

الـ STM32MP1 clock driver كان بيعمل كل gate clock بـ `spinlock_t` local في stack أثناء الـ registration بدل ما يعملها static أو في heap:

```c
/* كود غلط في الـ SoC driver */
static int stm32_clk_probe(struct platform_device *pdev)
{
    spinlock_t local_lock;  // في الـ stack — هيتمسح بعد return
    spin_lock_init(&local_lock);

    clk_hw_register_gate(dev, "i2c1_gate", "i2c1_src",
                         0, reg, 5, 0, &local_lock);
    // local_lock اتمسح هنا!
}
```

بعد resume، الـ CCF بتكلم `clk_gate_enable()` → `clk_gate_endisable()` → بتوصل لـ lock فسد.

#### الحل
الـ lock لازم يكون static أو allocated في heap:

```c
/* الكود الصح */
static DEFINE_SPINLOCK(stm32_clk_lock);  // static — آمن

static int stm32_clk_probe(struct platform_device *pdev)
{
    clk_hw_register_gate(dev, "i2c1_gate", "i2c1_src",
                         0, reg, 5, 0, &stm32_clk_lock);
}
```

أو لو كل gate محتاج lock مستقل:

```c
struct stm32_gate_data {
    spinlock_t lock;  // في heap مع الـ struct
    struct clk_hw *hw;
};

// وبعدين تعمل spin_lock_init على &data->lock
```

#### الدرس المستفاد
الـ `clk-gate.c` بيحتفظ بـ pointer للـ `spinlock_t` ومش بيعمل copy — الـ lock نفسه لازم يفضل موجود طول عمر الـ clock. أي lock في stack هيكون disaster مؤجل.

---

### السيناريو 3: HDMI مش بيشتغل على Allwinner H616 — Android TV Box

#### العنوان
**الـ HDMI output ميت بسبب `CLK_GATE_SET_TO_DISABLE` معكوس على H616**

#### السياق
منتج Android TV box بيستخدم Allwinner H616. الـ display pipeline شغال، الـ framebuffer موجود، بس الـ HDMI مش بيخرج صورة على التلفزيون. الـ `adb logcat` مفيش errors واضحة.

#### المشكلة
الـ HDMI controller clock gate متوقف رغم إن الـ driver طلب enable.

#### التحليل
الـ H616 HDMI clock register عنده سلوك عكسي: كتابة `1` في الـ bit بتوقف الـ clock، وكتابة `0` بتشغله — يعني محتاج `CLK_GATE_SET_TO_DISABLE`.

الـ developer الأصلي نسي يحط الـ flag:

```c
/* كود ناقص في sun50i-h616-ccu.c */
static SUNXI_CCU_GATE(hdmi_clk, "hdmi", "pll-video0",
    0x7bc, BIT(0), 0);  // فيه flag = 0، المفروض CLK_GATE_SET_TO_DISABLE
```

نتيجة: لما الـ driver بيعمل `clk_enable()` → `clk_gate_endisable(hw, 1)`:

```c
static void clk_gate_endisable(struct clk_hw *hw, int enable)
{
    int set = gate->flags & CLK_GATE_SET_TO_DISABLE ? 1 : 0;
    // set = 0 لأن FLAG مش موجود

    set ^= enable;
    // set = 0 XOR 1 = 1 → هيكتب 1 في الـ register

    // لكن على H616، كتابة 1 معناها DISABLE!
    // فالـ clock اتوقف بدل ما يشتغل
}
```

```bash
# للتشخيص
devmem2 0x07000000 w  # اقرأ الـ register يدوياً
# bit 0 = 1 → disabled على H616
```

#### الحل
إضافة `CLK_GATE_SET_TO_DISABLE`:

```c
static SUNXI_CCU_GATE(hdmi_clk, "hdmi", "pll-video0",
    0x7bc, BIT(0), CLK_GATE_SET_TO_DISABLE);
```

للتحقق:
```bash
cat /sys/kernel/debug/clk/hdmi/clk_enable_count
# 1
# والصورة هتظهر على التلفزيون
```

#### الدرس المستفاد
الـ `CLK_GATE_SET_TO_DISABLE` flag في `struct clk_gate` بيعكس المنطق كله داخل `clk_gate_endisable()` عن طريق XOR. لازم تقرأ الـ datasheet ميتن وتتأكد من اتجاه الـ gate bit — خصوصاً على Allwinner اللي عنده vendor-specific conventions.

---

### السيناريو 4: SPI توقف عشوائي على i.MX8 — Automotive ECU

#### العنوان
**الـ SPI bus بيتوقف عشوائياً على i.MX8 في ECU تحكم محرك السيارة**

#### السياق
ECU للتحكم في محرك سيارة مبني على i.MX8M. الـ ECU بيستخدم SPI للتواصل مع sensor chips خاصة. في بيئة الاختبار شغال تمام، بس في الإنتاج بعد ساعات تشغيل، الـ SPI بيتوقف فجأة وبيرجع `ETIMEOUT`.

#### المشكلة
الـ SPI clock بيتعطل بشكل عشوائي. الـ `clk_enable_count` بيرجع لـ 0 رغم إن الـ driver مش بيعمل `clk_disable`.

#### التحليل
الـ i.MX8 SPI driver الخاص بالـ ECU كان فيه bug: بيعمل `clk_put()` قبل ما يعمل `clk_disable()` في بعض error paths:

```c
/* كود غلط في الـ ECU SPI driver */
static int ecu_spi_transfer(struct spi_device *spi, ...)
{
    if (some_error) {
        clk_put(priv->spi_clk);  // released الـ clock
        // لكن ما عملش clk_disable أولاً!
        return -EIO;
    }
}
```

لما الـ CCF يشوف إن الـ consumer release الـ clock من غير disable، ممكن يعمل `clk_disable_unused()` في الـ background بعد فترة وهيكلم `clk_gate_disable()` → `clk_gate_endisable(hw, 0)`:

```c
static void clk_gate_endisable(struct clk_hw *hw, int enable)
{
    // enable = 0 → هيوقف الـ clock
    // ...
    reg &= ~BIT(gate->bit_idx);  // clear the bit → SPI clock off
    clk_gate_writel(gate, reg);  // applied to hardware
}
```

للتشخيص:
```bash
# شوف لو الـ clock بييجي disabled من clk_disable_unused
dmesg | grep "disable_unused"
# أو
cat /sys/kernel/debug/clk/ecspi3_clk/clk_enable_count
# بييجي 0 في وقت المشكلة
```

#### الحل
صلح الـ error path في الـ SPI driver:

```c
static int ecu_spi_transfer(struct spi_device *spi, ...)
{
    if (some_error) {
        clk_disable_unprepare(priv->spi_clk);  // الترتيب الصح
        clk_put(priv->spi_clk);
        return -EIO;
    }
}
```

وعلشان الـ safety في automotive context، أضف `CLK_IS_CRITICAL` للـ SPI clock في الـ DT:

```dts
&ecspi3 {
    assigned-clocks = <&clk IMX8MN_CLK_ECSPI3>;
    /* CLK_IS_CRITICAL يمنع clk_disable_unused من إيقافه */
};
```

أو في الـ clock registration:
```c
clk_hw_register_gate(dev, "ecspi3_clk", "ecspi3_root",
    CLK_IS_CRITICAL,  // لا تقفله أبداً
    reg, bit, 0, &lock);
```

#### الدرس المستفاد
الـ `clk_gate_disable()` بيتكلم من CCF تلقائياً في `clk_disable_unused()` scan. في safety-critical systems زي automotive، استخدم `CLK_IS_CRITICAL` للـ clocks المهمة. دايماً اتبع الترتيب: `clk_disable` → `clk_unprepare` → `clk_put`.

---

### السيناريو 5: Boot Hang على AM62x أثناء Custom Board Bring-up

#### العنوان
**الـ kernel بيتعلق عند أول `clk_enable` على custom AM62x board**

#### السياق
engineer بيعمل bring-up لـ custom board مبني على AM62x (Texas Instruments). البورد معدّلة عن الـ EVM القياسي — فيها power domains مختلفة. الـ kernel بيبوت، بيطبع أول رسايل، وبعدين بيتعلق صامت بدون أي output.

#### المشكلة
الـ system بيتعلق عند أول `clk_enable()` للـ UART clock. الـ hang مش بيرجع، مفيش watchdog reset.

#### التحليل
الـ engineer وصّل JTAG وشاف إن الـ CPU عالق في:

```c
static void clk_gate_endisable(struct clk_hw *hw, int enable)
{
    struct clk_gate *gate = to_clk_gate(hw);
    // ...
    if (gate->lock)
        spin_lock_irqsave(gate->lock, flags);  // <-- العلقة هنا
```

الـ spinlock محتاجش وقت، فالمشكلة مش deadlock في الـ lock نفسه. المشكلة إن الـ:

```c
    reg = clk_gate_readl(gate);  // <-- ممكن تكون هنا
```

على الـ custom board، الـ clock controller registers مش accessible لأن الـ power domain بتاعه مش enabled. قراءة من register معطوب بيعمل bus hang.

```bash
# التشخيص عن طريق JTAG
# PC عالق في clk_gate_readl → readl → bus waiting for response
```

الـ `clk_gate_readl()`:
```c
static inline u32 clk_gate_readl(struct clk_gate *gate)
{
    if (gate->flags & CLK_GATE_BIG_ENDIAN)
        return ioread32be(gate->reg);

    return readl(gate->reg);  // <-- bus hang لو الـ power domain off
}
```

الـ HIWORD_MASK path كانت تنجو لأنها مش بتعمل read:
```c
if (gate->flags & CLK_GATE_HIWORD_MASK) {
    reg = BIT(gate->bit_idx + 16);  // write-only، مش بتقرأ
    // ...
}
```

#### الحل
**أولاً**: تأكد إن الـ power domain للـ clock controller enabled قبل أي clock access — في الـ DT:

```dts
/* custom AM62x board DT */
&main_clk_ctrl {
    power-domains = <&k3_pds 1 TI_SCI_PD_EXCLUSIVE>;
    /* تأكد إن الـ power domain يتعمل enable أول */
};
```

**ثانياً**: لو الـ SoC بيدعم HIWORD_MASK على الـ gate registers، استخدمه بدل read-modify-write:

```c
/* في الـ AM62x clock driver */
clk_hw_register_gate(dev, "uart_clk", "uart_parent",
    0, reg, bit_idx,
    CLK_GATE_HIWORD_MASK,  // بيتجنب الـ readl
    &clk_lock);
```

**ثالثاً**: أضف early power domain init في الـ board file أو DT:

```dts
/* قبل أي clock consumer */
reserved-memory {
    /* ensure clock controller power domain is on */
};
```

للتشخيص المستقبلي:
```bash
# بعد الإصلاح
cat /sys/kernel/debug/clk/uart0_fclk_div/clk_enable_count
# 1 — شغال

dmesg | grep clk | head -20
# مفيش errors
```

#### الدرس المستفاد
على custom boards، الـ clock controller نفسه محتاج power domain يكون enabled قبل أي `clk_gate_readl()`. الـ `CLK_GATE_HIWORD_MASK` flag في `clk-gate.c` بيتجنب الـ read-modify-write ويعمل write-only — ده مش بس optimization، ده أحياناً بيكون الحل الوحيد للـ hardware اللي مش بيدعم reads على clock registers وقت الـ power-up.
## Phase 7: مصادر ومراجع

### مصادر الـ LWN.net

| العنوان | الوصف |
|---------|-------|
| [A common clock framework](https://lwn.net/Articles/472998/) | المقال الأساسي اللي شرح فكرة الـ common clock framework قبل ما يتدمج في الكيرنل |
| [common clk framework (discussion)](https://lwn.net/Articles/486841/) | نقاش مجتمع الكيرنل حول تفاصيل التصميم والـ API |
| [Documentation/clk.txt](https://lwn.net/Articles/489668/) | التوثيق الرسمي الأولي للـ framework في LWN |
| [clk: dt: bindings for mux, divider & gate clocks](https://lwn.net/Articles/555242/) | الـ Device Tree bindings الخاصة بالـ gate clocks والـ `clk-gate.c` |
| [Binding and driver for gated-fixed-clocks](https://lwn.net/Articles/987538/) | إضافة دعم الـ `gated-fixed-clock` كامتداد حديث للـ gate clock |
| [Apple M1 clock gate driver](https://lwn.net/Articles/857154/) | مثال واقعي على استخدام الـ `clk_gate` في معالج Apple M1 |
| [clk: implement clock rate protection mechanism](https://lwn.net/Articles/740484/) | آلية حماية معدل الـ clock اللي بتتفاعل مع الـ gate logic |
| [Add a generic struct clk](https://lwn.net/Articles/460193/) | العمل الأصلي اللي أدى لظهور الـ `struct clk_gate` |
| [The Common Clk Framework — kernel docs (LWN mirror)](https://static.lwn.net/kerneldoc/driver-api/clk.html) | نسخة LWN من التوثيق الرسمي للـ kernel |

---

### التوثيق الرسمي للـ Kernel

```
Documentation/driver-api/clk.rst       ← التوثيق الكامل للـ Common CLK Framework
Documentation/devicetree/bindings/clock/gpio-gate-clock.yaml  ← DT binding للـ GPIO gate
Documentation/devicetree/bindings/clock/clock-bindings.txt    ← قواعد عامة لـ clock bindings
include/linux/clk-provider.h           ← تعريف struct clk_gate + clk_gate_ops + macros
drivers/clk/clk-gate.c                 ← الـ implementation الأساسي
drivers/clk/clk.c                      ← قلب الـ Common CLK Framework
```

الـ online version:
- [The Common Clk Framework — docs.kernel.org](https://docs.kernel.org/driver-api/clk.html)

---

### نقاشات الـ Mailing List (LKML)

| الرابط | الوصف |
|--------|-------|
| [PATCH v7 2/3: clk: introduce the common clock framework](https://lkml.iu.edu/hypermail/linux/kernel/1203.2/00026.html) | الـ patch الأصلي اللي أدخل الـ framework — Mike Turquette, مارس 2012 |
| [PATCH v7 3/3: clk: basic clock hardware types](https://lkml.iu.edu/hypermail/linux/kernel/1203.2/00027.html) | الـ patch اللي أدخل `clk-gate.c` + `clk-divider.c` + `clk-mux.c` |
| [Re: PATCH v6 3/3: clk: basic clock hardware types](https://lkml.iu.edu/hypermail/linux/kernel/1203.1/03756.html) | نقاش review قبل الـ merge النهائي |

---

### Commits مهمة في الـ Kernel

الـ commits دي ممكن تتتبعها عبر `git log --follow drivers/clk/clk-gate.c`:

```bash
# مشاهدة تاريخ الملف كامل
git log --oneline --follow drivers/clk/clk-gate.c

# أهم الـ commits التاريخية:
# 3d6199d  clk: basic clock hardware types  (kernel 3.4, 2012)
#           → ده أول commit لملف clk-gate.c

# commits لاحقة أضافت:
# - CLK_GATE_HIWORD_MASK support
# - CLK_GATE_BIG_ENDIAN support
# - devm_clk_hw_register_gate
# - __clk_hw_register_gate (new-style API)
```

- [kernel.org cgit — drivers/clk/clk-gate.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/drivers/clk/clk-gate.c)

---

### عروض المؤتمرات والمواد التعليمية

| المصدر | الوصف |
|--------|-------|
| [Bootlin ELCE 2013: Common clock framework how to use it (PDF)](https://bootlin.com/pub/conferences/2013/elce/common-clock-framework-how-to-use-it/common-clock-framework-how-to-use-it.pdf) | شرح عملي ممتاز للـ framework بالكامل بما فيه الـ gate clocks |
| [Bootlin ELC 2013: Common clock framework (PDF)](https://bootlin.com/pub/conferences/2013/elc/common-clock-framework-how-to-use-it/common-clock-framework-how-to-use-it.pdf) | نسخة أمريكا من نفس العرض |
| [ELC 2007: Linux clock management framework (elinux.org PDF)](https://elinux.org/images/a/a3/ELC_2007_Linux_clock_fmw.pdf) | الخلفية التاريخية قبل الـ Common CLK Framework |

---

### صفحات kernelnewbies.org ذات الصلة

الـ common clock framework اتدمج في **Linux 3.4** — تقدر تلاقي التغييرات في:

- [kernelnewbies.org — Linux_6.7](https://kernelnewbies.org/Linux_6.7) ← تغييرات حديثة في الـ clock subsystem
- [kernelnewbies.org — Linux_6.12](https://kernelnewbies.org/Linux_6.12) ← أحدث updates

للـ framework نفسه من البداية:
```
https://kernelnewbies.org/Linux_3.4
```
ده الإصدار اللي اتدمج فيه الـ Common CLK Framework أول مرة.

---

### صفحات elinux.org ذات الصلة

| الصفحة | الوصف |
|--------|-------|
| [High Resolution Timers](https://elinux.org/High_Resolution_Timers) | صلة بـ clock subsystem من زاوية الـ timer |
| [Power Management Specification](https://elinux.org/Power_Management_Specification) | الـ gate clocks جزء من منظومة الـ power management |
| [Kernel Timer Systems](https://elinux.org/index.php/Kernel_Timer_Systems) | نظرة عامة على أنظمة الـ clock والـ timer في الكيرنل |

---

### الكتب الموصى بيها

#### Linux Device Drivers 3rd Edition (LDD3) — O'Reilly
- **الفصل 9**: Communicating with Hardware — بيشرح `ioread32` / `iowrite32` / `readl` / `writel` اللي بيستخدمها `clk-gate.c`
- **الفصل 5**: Concurrency and Race Conditions — بيشرح `spinlock_t` و `spin_lock_irqsave` اللي بتحمي الـ gate register
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 10**: Kernel Synchronization Methods — شرح `spinlock` و `irqsave`
- **الفصل 14**: The Block I/O Layer (خلفية عن الـ device model)
- ده كتاب أساسي لفهم الـ infrastructure اللي بيبني عليها الـ `clk-gate.c`

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 16**: Kernel Initialization — بيغطي الـ clock setup في مرحلة الـ boot
- بيشرح إزاي الـ platform يسجل الـ clocks قبل ما الـ drivers تشتغل

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- **الفصل 1**: Introduction and Overview — architecture الـ kernel وإزاي الـ subsystems بتتفاعل

---

### Platform-specific References

لو بتشتغل على hardware معين، دور على الـ clock driver الخاص بيه في:

```
drivers/clk/<vendor>/     ← مثلاً: drivers/clk/qcom/, drivers/clk/imx/
```

كل driver من دول بيستخدم `clk_hw_register_gate()` أو `__clk_hw_register_gate()` من `clk-gate.c`.

---

### مصطلحات البحث

لو عايز تعرف أكتر، استخدم الكلمات دي في البحث:

```
linux kernel common clock framework tutorial
clk_gate_ops implementation example
CLK_GATE_HIWORD_MASK explained
devres clk_hw_register_gate
of_clk_hw_register device tree clock provider
linux clock gating power management embedded
clk-provider.h API reference
CONFIG_COMMON_CLK kernel config
```

---

### ملخص أهم المصادر

| الأولوية | المصدر | ليه؟ |
|----------|--------|------|
| ⭐⭐⭐ | [docs.kernel.org/driver-api/clk.html](https://docs.kernel.org/driver-api/clk.html) | التوثيق الرسمي والمحدث |
| ⭐⭐⭐ | [LWN: A common clock framework](https://lwn.net/Articles/472998/) | فهم الـ design decisions |
| ⭐⭐⭐ | [LKML PATCH v7 3/3](https://lkml.iu.edu/hypermail/linux/kernel/1203.2/00027.html) | الـ patch الأصلي لـ `clk-gate.c` |
| ⭐⭐ | [Bootlin ELCE 2013 PDF](https://bootlin.com/pub/conferences/2013/elce/common-clock-framework-how-to-use-it/common-clock-framework-how-to-use-it.pdf) | أفضل شرح عملي |
| ⭐⭐ | [git.kernel.org — clk-gate.c history](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/drivers/clk/clk-gate.c) | تاريخ التطور الكامل |
| ⭐ | LDD3 Chapter 9 | خلفية الـ MMIO |
## Phase 8: Writing simple module

### الفكرة

**`clk_gate_is_enabled`** هي الدالة الأنسب للـ hook — متاحة عبر `EXPORT_SYMBOL_GPL`، بتُستدعى في كل مرة حاجة بتسأل عن حالة الـ gate clock (enabled/disabled)، ومش بتعمل أي حاجة خطرة. هنستخدم **kprobe** عشان نعترض استدعاءها ونطبع اسم الـ clock واللي هو enabled ولا لأ.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_clk_gate.c
 *
 * Hooks clk_gate_is_enabled() via kprobe and logs the clock name
 * plus its enable state every time the function is called.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit        */
#include <linux/kernel.h>       /* pr_info, pr_err                         */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe, ...     */
#include <linux/clk-provider.h> /* struct clk_hw, clk_hw_get_name()        */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on clk_gate_is_enabled - logs gate clock state");

/* ------------------------------------------------------------------ */
/*  pre-handler: runs just before clk_gate_is_enabled executes        */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64: first argument (struct clk_hw *hw) lives in rdi.
     * On ARM64:  first argument lives in x0.
     * We use regs_get_kernel_argument() which is arch-agnostic.
     */
    struct clk_hw *hw = (struct clk_hw *)regs_get_kernel_argument(regs, 0);

    if (!hw)
        return 0;

    /* clk_hw_get_name() returns the clock name string safely */
    pr_info("clk_gate_is_enabled called: clock='%s'\n",
            clk_hw_get_name(hw));

    return 0; /* 0 = let the original function continue normally */
}

/* ------------------------------------------------------------------ */
/*  post-handler: runs right after clk_gate_is_enabled returns        */
/* ------------------------------------------------------------------ */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * regs_return_value() gives us the return value of the probed
     * function (which is 1 if enabled, 0 if disabled).
     */
    long ret = regs_return_value(regs);

    pr_info("clk_gate_is_enabled returned: %s\n",
            ret ? "ENABLED" : "DISABLED");
}

/* ------------------------------------------------------------------ */
/*  kprobe descriptor                                                  */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name    = "clk_gate_is_enabled", /* target function by name  */
    .pre_handler    = handler_pre,           /* called before the func   */
    .post_handler   = handler_post,          /* called after the func    */
};

/* ------------------------------------------------------------------ */
/*  module_init                                                        */
/* ------------------------------------------------------------------ */
static int __init kprobe_clk_gate_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("kprobe planted on clk_gate_is_enabled at %px\n", kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/*  module_exit                                                        */
/* ------------------------------------------------------------------ */
static void __exit kprobe_clk_gate_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("kprobe on clk_gate_is_enabled removed\n");
}

module_init(kprobe_clk_gate_init);
module_exit(kprobe_clk_gate_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | ماكرو الـ `module_init` / `module_exit` وـ `MODULE_LICENSE` |
| `linux/kernel.h` | دوال `pr_info` / `pr_err` للـ logging |
| `linux/kprobes.h` | تعريف `struct kprobe` وكل دوال التسجيل والإلغاء |
| `linux/clk-provider.h` | تعريف `struct clk_hw` وـ `clk_hw_get_name()` عشان نجيب اسم الـ clock |

**الـ** `linux/clk-provider.h` ضروري لأن `struct clk_hw` معرّف فيه، ومن غيره المترجم مش هيعرف الـ type.

---

#### `handler_pre` — الـ callback قبل التنفيذ

بنجيب الـ argument الأول (`struct clk_hw *hw`) من الـ registers عبر `regs_get_kernel_argument()` اللي portable على كل الـ architectures. بعدين بنطبع اسم الـ clock عشان نعرف إيه الـ gate clock اللي اتسُئل عنه.

---

#### `handler_post` — الـ callback بعد التنفيذ

**الـ** `regs_return_value()` بتجيب القيمة اللي الدالة رجعتها (1 = enabled، 0 = disabled)، فبنطبع الحالة الفعلية بعد ما الدالة خلصت وقرأت الـ hardware register.

---

#### `struct kprobe kp`

**الـ** `.symbol_name` بيخلي الكرنل يحل الـ symbol بنفسه وقت التحميل، مش محتاجين نحسب العنوان يدوياً. الـ `.pre_handler` و `.post_handler` هما نقطتا التقاطع قبل وبعد تنفيذ الدالة الأصلية.

---

#### `module_init` و `module_exit`

`register_kprobe()` بتزرع الـ breakpoint في الكرنل الشغال دلوقتي. لازم `unregister_kprobe()` يتنادى في الـ exit عشان نشيل الـ breakpoint ونمنع أي use-after-free لو الـ handler اتنادى بعد ما الـ module اتشيل من الذاكرة.

---

### تشغيل الـ Module

```bash
# Build (Makefile بسيط)
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# تحميل الـ module
sudo insmod kprobe_clk_gate.ko

# شوف الـ output
sudo dmesg | tail -30

# إزالة الـ module
sudo rmmod kprobe_clk_gate
```

**مثال على الـ output المتوقع في `dmesg`:**

```
[ 1234.567890] kprobe planted on clk_gate_is_enabled at ffffffffc0ab1234
[ 1234.601000] clk_gate_is_enabled called: clock='ahb_clk'
[ 1234.601002] clk_gate_is_enabled returned: ENABLED
[ 1234.612000] clk_gate_is_enabled called: clock='spi0_clk'
[ 1234.612001] clk_gate_is_enabled returned: DISABLED
```
