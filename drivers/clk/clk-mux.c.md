## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الـ `clk-mux.c` جزء من **Common Clock Framework (CCF)** — الـ subsystem المركزي اللي بيتحكم في كل الـ clocks جوه الـ Linux kernel. المشرفون عليه: Michael Turquette و Stephen Boyd، والميلينج ليست هي `linux-clk@vger.kernel.org`.

---

### المشكلة اللي بيحلها بالبساطة

تخيل إنك بتشغّل جهاز كمبيوتر وعندك **مروحة كهربائية** ممكن توصّلها بمصدر كهرباء من الحائط أو من UPS أو من مولّد. في كل لحظة المروحة شغّالة من مصدر واحد بس، وممكن تحوّل بين المصادر.

ده بالظبط اللي بيعمله **Clock Mux** — هو **سويتش كهربائي** بيختار مصدر الـ clock اللي يوصل للـ IP block (زي USB, CPU, GPU). مش بيولّد clock جديد، ومش بيقفل الـ clock — بس بيختار منين تيجي الـ timing pulses.

في الـ SoCs (زي Raspberry Pi أو Snapdragon)، فيه عشرات الـ clocks بتشتغل بترددات مختلفة:
- **PLL A** بيدي 1 GHz
- **PLL B** بيدي 400 MHz
- **Crystal Oscillator** بيدي 24 MHz

الـ USB controller مثلاً ممكن يشتغل على أي منهم. الـ mux هو الـ register اللي بتكتب فيه رقم (0, 1, 2) عشان تقول "روح خد الـ clock من PLL A".

---

### القصة الكاملة

```
                ┌──────────┐
PLL_A (1GHz) ──►│          │
                │  MUX     ├──► USB Controller Clock
PLL_B (400MHz)─►│  Register│
                │  [sel=1] │
XTAL (24MHz) ──►│          │
                └──────────┘
                   ▲
                   │
              kernel writes
              sel bits here
```

لما الـ kernel يعمل `clk_set_parent(usb_clk, pll_b)`:
1. الـ CCF بيسأل: "ايه الـ index بتاع PLL_B في الـ mux؟"
2. بيحوّل الـ index ده لقيمة بتكتبها في الـ register (ممكن تكون 1، أو bit mask، أو جدول custom)
3. بيكتب في الـ MMIO register بالـ shift والـ mask الصح
4. الـ USB controller دلوقتي بياخد الـ clock من PLL_B

---

### الهدف من الملف

**الـ `clk-mux.c`** بيوفّر:

| الوظيفة | التفاصيل |
|---|---|
| `clk_mux_get_parent` | بيقرأ الـ register ويرجع ايه الـ parent الحالي |
| `clk_mux_set_parent` | بيكتب في الـ register لتغيير الـ parent |
| `clk_mux_determine_rate` | بيساعد الـ CCF يختار أحسن parent لتردد معين |
| `clk_mux_val_to_index` | بيحوّل قيمة register لـ index منطقي |
| `clk_mux_index_to_val` | بيحوّل الـ index لقيمة تتكتب في الـ register |
| `__clk_hw_register_mux` | بيسجّل الـ mux clock في الـ CCF |
| `__devm_clk_hw_register_mux` | نفس التسجيل بس بـ devres (auto-cleanup) |

---

### ليه محتاجين val_to_index و index_to_val؟

مش كل الـ hardware بتكتب فيه 0, 1, 2 مباشرة. بعض الـ SoCs عندها:

- **`CLK_MUX_INDEX_ONE`** — الـ register بيبدأ من 1 مش 0 (اختار parent 0 → اكتب 1)
- **`CLK_MUX_INDEX_BIT`** — الـ register بيستخدم single bit لكل parent (parent 2 → اكتب 0b100)
- **Custom table** — مش فيه نظام، كل parent ليه قيمة arbitrary في جدول

```c
/* مثال: hardware بيستخدم bit positions */
/* parent index 0 → val = 1 (BIT(0)) */
/* parent index 1 → val = 2 (BIT(1)) */
/* parent index 2 → val = 4 (BIT(2)) */
if (flags & CLK_MUX_INDEX_BIT)
    val = 1 << index;
```

---

### التفاصيل المهمة في الـ set_parent

```c
static int clk_mux_set_parent(struct clk_hw *hw, u8 index)
{
    /* تحويل index → قيمة register */
    u32 val = clk_mux_index_to_val(mux->table, mux->flags, index);

    /* spinlock عشان الـ register write atomic */
    spin_lock_irqsave(mux->lock, flags);

    if (mux->flags & CLK_MUX_HIWORD_MASK) {
        /* بعض الـ SoCs (زي Rockchip) بتستخدم upper 16-bit كـ write-mask */
        /* تكتب الـ mask في upper word والقيمة في lower word في نفس الـ write */
        reg = mux->mask << (mux->shift + 16);
    } else {
        /* الطريقة العادية: read-modify-write */
        reg = clk_mux_readl(mux);
        reg &= ~(mux->mask << mux->shift); /* امسح الـ bits القديمة */
    }
    reg |= val << mux->shift; /* حط القيمة الجديدة */
    clk_mux_writel(mux, reg);

    spin_unlock_irqrestore(mux->lock, flags);
}
```

---

### الـ Read-Only Mux

بعض الـ hardware عندها mux مش ممكن تتغيّر (محددة من الـ bootloader أو hardware straps). الملف بيوفّر:
- **`clk_mux_ops`** — للـ mux اللي ممكن تغيّره (get + set + determine_rate)
- **`clk_mux_ro_ops`** — للـ read-only mux (get فقط)

---

### الـ Files المهمة اللي المفروض تعرفها

| الملف | الدور |
|---|---|
| `drivers/clk/clk.c` | قلب الـ CCF، بيدير شجرة الـ clocks كلها |
| `drivers/clk/clk-mux.c` | **ملفنا** — تنفيذ الـ mux clock |
| `drivers/clk/clk-gate.c` | clock بيتقفل ويتفتح (enable/disable) |
| `drivers/clk/clk-divider.c` | clock بيقسّم التردد |
| `drivers/clk/clk-fixed-rate.c` | clock بتردد ثابت (زي crystal) |
| `drivers/clk/clk-composite.c` | يجمع mux + divider + gate في clock واحد |
| `include/linux/clk-provider.h` | الـ structs والـ APIs الأساسية (struct clk_mux، clk_ops، إلخ) |
| `include/linux/clk.h` | الـ consumer API (clk_get، clk_set_parent، إلخ) |
| `drivers/clk/clkdev.c` | ربط الـ clocks بالـ devices بالاسم |

---

### الـ Files اللي بتستخدم clk-mux كـ building block

كل SoC driver تقريباً بيستخدم `clk_hw_register_mux_table` أو `clk_hw_register_mux` — مثلاً:
- `drivers/clk/rockchip/` — Rockchip SoCs (بيستخدم HIWORD_MASK كتير)
- `drivers/clk/sunxi-ng/` — Allwinner SoCs
- `drivers/clk/meson/` — Amlogic SoCs
- `drivers/clk/qcom/` — Qualcomm SoCs

---

### ملخص

**الـ `clk-mux.c`** هو تنفيذ بسيط وعام لـ **clock multiplexer** — سويتش بيختار مصدر الـ timing signal من بين عدة parents عن طريق الكتابة في MMIO register. بيدعم أشكال مختلفة لتمثيل الـ index في الـ hardware (مباشر، bit position، custom table)، وبيتعامل مع الـ endianness ومع الـ Rockchip-style hiword masking. الهدف منه إنه يكون **generic reusable primitive** أي SoC driver يقدر يستخدمه بدل ما يكتب نفس الـ logic من أول.
## Phase 2: شرح الـ Common Clock Framework (CCF) — Mux Clock

---

### المشكلة اللي الـ Subsystem بيحلها

أي SoC حديث — خد مثلاً Qualcomm Snapdragon أو NXP i.MX8 — فيه عشرات المصادر للـ clock: crystal خارجي، PLL واحد أو أكتر، RC oscillator داخلي. كل peripheral محتاج يختار مين يأخذ منه الـ clock. قبل الـ CCF، كل driver كان بيعمل register read/write ببراءة تامة من غير أي تنسيق، فالنتيجة:

- **race conditions** لما درايفرين يشتغلوا على نفس الـ clock register في نفس الوقت.
- **code duplication** — كل platform بتعيد نفس الـ "اقرأ register، mask الـ bits، اكتب القيمة" من أوله.
- **no dependency tracking** — لو peripheral A بتاخد clock من PLL2 وعايز حد يوقف PLL2، مفيش حاجة بتحسب إن A لسه شغال.

---

### الحل اللي الـ Kernel اتخذه

الـ **Common Clock Framework (CCF)** — موجود في `drivers/clk/clk.c` — قدّم abstraction layer كامل:

1. كل clock في النظام بيتعامل معاه كـ **node** في شجرة (tree) من الـ clocks.
2. كل node عنده `struct clk_hw` كـ handle، و`struct clk_ops` كـ vtable لعمليات الـ hardware.
3. الـ framework هو اللي بيمسك الـ reference counting، الـ locking، والـ tree traversal.
4. الـ driver بس بيقول "أنا عندي hardware من النوع ده، وده الـ register بتاعه" — والـ framework يدير الباقي.

الـ **`clk-mux.c`** هو تنفيذ نوع واحد محدد من الـ clocks: الـ **clock multiplexer** — يعني clock بيختار من بين أكتر من parent واحد.

---

### الـ Real-World Analogy — بالتفصيل

تخيل **محطة توزيع كهرباء** (electrical substation) بيها أكتر من مصدر تغذية (شبكة وطنية، مولد احتياطي، طاقة شمسية). على لوحة التحكم في switch بيحدد المصدر الفعلي اللي بيغذي الحي.

| المثال الحقيقي | مقابله في الـ Kernel |
|---|---|
| لوحة التحكم (switch panel) | `struct clk_mux` |
| الـ switch نفسه (الحديدة) | Register الـ MMIO في الـ SoC |
| رقم بت الـ switch في اللوحة | `mux->shift` + `mux->mask` |
| المصادر المتاحة (شبكة / مولد / شمس) | `parent_names[]` array |
| المصدر الحالي المختار | القيمة المقروءة من الـ register بعد الـ mask والـ shift |
| دليل المصادر بترتيب خاص | `mux->table[]` — بيترجم index لـ register value |
| قفل اللوحة لمنع تشغيل أكتر من switch | `mux->lock` (spinlock) |
| موظف المحطة اللي بيقرأ/يكتب | `clk_mux_get_parent()` / `clk_mux_set_parent()` |
| السكة الكهربائية اللي بتوصل للحي | `struct clk_hw` — الـ handle اللي الـ framework بيشوف بيه الـ clock |
| ملف تعليمات التشغيل | `struct clk_ops` — الـ vtable |

الجزء الغير واضح في الـ analogy عادةً: الـ `table[]` هو زي "كودبوك" — المصدر رقم 0 (index=0) مش بالضرورة بيتكتب القيمة `0x0` في الـ register؛ ممكن يتكتب `0x4` مثلاً لو الـ hardware اتصمم كده.

---

### Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Consumer Drivers                          │
│  (UART driver, SPI driver, USB driver, ...)                  │
│         │  clk_set_parent(clk, parent)                       │
│         │  clk_get_rate(clk)                                 │
└─────────┼───────────────────────────────────────────────────┘
          │  Public CLK API  (include/linux/clk.h)
          ▼
┌─────────────────────────────────────────────────────────────┐
│              Common Clock Framework (CCF)                    │
│              drivers/clk/clk.c                               │
│                                                              │
│  ┌──────────────┐   ┌────────────┐   ┌──────────────────┐   │
│  │  clk_core    │──▶│  clk_hw   │──▶│   clk_ops        │   │
│  │  (internal)  │   │  (bridge) │   │  .get_parent()   │   │
│  │  - ref count │   │           │   │  .set_parent()   │   │
│  │  - parent ptr│   │           │   │  .determine_rate │   │
│  └──────────────┘   └─────┬─────┘   └──────────────────┘   │
│                            │                                  │
└────────────────────────────┼─────────────────────────────────┘
                             │  (embedded inside)
                             ▼
┌─────────────────────────────────────────────────────────────┐
│               clk-mux.c  (Hardware Implementation)          │
│                                                              │
│   struct clk_mux {                                           │
│       struct clk_hw  hw;    ◀── bridge back to CCF          │
│       void __iomem  *reg;   ◀── MMIO register address       │
│       u32            mask;  ◀── which bits                   │
│       u8             shift; ◀── bit position                 │
│       u8             flags; ◀── INDEX_ONE, HIWORD, etc.     │
│       const u32     *table; ◀── index→value translation     │
│       spinlock_t    *lock;  ◀── shared register protection  │
│   }                                                          │
│                                                              │
│   clk_mux_ops = {                                            │
│       .get_parent     = clk_mux_get_parent                   │
│       .set_parent     = clk_mux_set_parent                   │
│       .determine_rate = clk_mux_determine_rate               │
│   }                                                          │
└───────────────────────────┬─────────────────────────────────┘
                            │  readl / writel
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                SoC Clock Controller (MMIO)                   │
│                                                              │
│   Register 0x3C:  [ EN | ... | MUX[2:0] | ... ]            │
│                                     ▲                        │
│                   bits[4:2] = مصدر الـ clock المختار        │
│                                                              │
│   Parents:  [0]=PLL1  [1]=PLL2  [2]=OSC_24M  [3]=PLL_USB   │
└─────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction: الـ `struct clk_hw` كـ Bridge

الفكرة المحورية في الـ CCF هي الفصل بين **عالمين**:

- **عالم الـ Framework**: الـ `struct clk_core` — private، مش الـ driver بيشوفه مباشرة، فيه الـ reference counting، الـ parent pointer، الـ rate cache.
- **عالم الـ Driver**: الـ `struct clk_mux` — بيمسك الـ hardware details.

الجسر بينهم هو `struct clk_hw`:

```c
struct clk_hw {
    struct clk_core *core;  /* pointer للـ framework-internal struct */
    struct clk      *clk;   /* pointer للـ per-user handle */
    const struct clk_init_data *init; /* يتم NULL بعد التسجيل */
};
```

الـ `struct clk_mux` بيبدأ بـ `struct clk_hw hw` كأول عضو — ده مش صدفة. ده بيسمح بـ `container_of`:

```c
#define to_clk_mux(_hw) container_of(_hw, struct clk_mux, hw)
```

يعني الـ CCF بيبعت `struct clk_hw *` للـ ops، والـ driver بيعمل `to_clk_mux(hw)` يرجع لـ `struct clk_mux` كامل — بكل الـ hardware details.

---

### شرح كل struct وعلاقته بالتاني

```
┌─────────────────────────────────────────────────────────────┐
│  struct clk_mux                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  struct clk_hw  hw         ◀── embedded, NOT pointer  │ │
│  │  ┌──────────────────────────────────────────────┐     │ │
│  │  │ struct clk_core  *core  ─────────────────────┼──┐  │ │
│  │  │ struct clk       *clk   ─────────────────────┼──┼─▶│ │
│  │  │ clk_init_data    *init  (NULL after register)│  │  │ │
│  │  └──────────────────────────────────────────────┘  │  │ │
│  │  void __iomem   *reg   ◀── MMIO address            │  │ │
│  │  const u32      *table ◀── optional value map      │  │ │
│  │  u32             mask  ◀── e.g. 0x7 for 3-bit mux  │  │ │
│  │  u8              shift ◀── e.g. 2 for bits[4:2]    │  │ │
│  │  u8              flags ◀── CLK_MUX_* flags          │  │ │
│  │  spinlock_t     *lock  ◀── shared with other clks  │  │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
            │ container_of ▲
            │              │
            ▼              │
┌─────────────────────────────────────────────────────────────┐
│  struct clk_ops  clk_mux_ops                                 │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  .get_parent     ──▶ clk_mux_get_parent(hw)          │   │
│  │  .set_parent     ──▶ clk_mux_set_parent(hw, index)   │   │
│  │  .determine_rate ──▶ clk_mux_determine_rate(hw, req) │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  struct clk_init_data  (مؤقت — بس وقت التسجيل)             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  name           = "uart_clk_mux"                     │   │
│  │  ops            = &clk_mux_ops                       │   │
│  │  parent_names[] = {"pll1", "pll2", "osc"}            │   │
│  │  num_parents    = 3                                   │   │
│  │  flags          = CLK_SET_RATE_PARENT                 │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

### تتبع الـ Flow: من `clk_set_parent()` للـ Register

لما driver بيعمل `clk_set_parent(clk, new_parent)`:

```
clk_set_parent()           [include/linux/clk.h — consumer API]
    │
    ▼
clk_core_set_parent_nolock()   [drivers/clk/clk.c — CCF internal]
    │  يعمل validation، يحسب الـ new parent index
    ▼
clk_hw->ops->set_parent(hw, index)
    │  يعمل dispatch للـ vtable
    ▼
clk_mux_set_parent(hw, index)  [drivers/clk/clk-mux.c]
    │
    ├─ clk_mux_index_to_val(table, flags, index)
    │      ├─ لو table موجود: table[index]
    │      ├─ لو CLK_MUX_INDEX_BIT: val = 1 << index
    │      └─ لو CLK_MUX_INDEX_ONE: val = index + 1
    │
    ├─ spin_lock_irqsave(mux->lock)
    │
    ├─ لو CLK_MUX_HIWORD_MASK:
    │      reg = mask << (shift + 16)        ← write-enable bits in high 16
    │  else:
    │      reg = readl(mux->reg)
    │      reg &= ~(mask << shift)           ← clear the mux field
    │
    ├─ reg |= (val << shift)                 ← set the new value
    ├─ writel(reg, mux->reg)
    └─ spin_unlock_irqrestore(mux->lock)
```

---

### الـ CLK_MUX_HIWORD_MASK — ليه موجود؟

بعض الـ SoCs (زي Rockchip) بتستخدم تقنية "write mask" في نفس الـ register. الـ high 16 bits بتحدد أنهي bits في الـ low 16 بيتم تعديلها. الميزة: تقدر تكتب register واحد من غير ما تحتاج read-modify-write — ده بيقلل الـ race window:

```
Register 32-bit:
┌──────────────────────┬──────────────────────┐
│  WRITE-ENABLE MASK   │   ACTUAL MUX VALUE   │
│  [31:16]             │   [15:0]             │
│  bit i=1 → write it  │  bit i = new value   │
└──────────────────────┴──────────────────────┘
```

في الكود:
```c
if (mux->flags & CLK_MUX_HIWORD_MASK) {
    /* بيحط الـ mask في الـ high word كـ write-enable */
    reg = mux->mask << (mux->shift + 16);
} else {
    /* الطريقة العادية: read → modify → write */
    reg = clk_mux_readl(mux);
    reg &= ~(mux->mask << mux->shift);
}
val = val << mux->shift;
reg |= val;
clk_mux_writel(mux, reg);
```

---

### الـ Index Translation: المشكلة والحل

الـ hardware register مش بالضرورة بيستخدم `0, 1, 2, 3` كـ encoding للـ parents. الـ CCF دايمًا بيتكلم بـ **index** (ترتيب في `parent_names[]`)، لكن الـ register بيتكلم بـ **val**. التحويل بيتم بطريقتين:

**طريقة 1: الـ `table[]` array**
```c
// مثال: hardware encoding غير متسلسل
const u32 mux_table[] = { 0, 2, 4, 8 };
// index 0 → reg val 0
// index 1 → reg val 2
// index 2 → reg val 4
// index 3 → reg val 8
```

**طريقة 2: الـ Flags**

| Flag | المعنى | index→val | val→index |
|---|---|---|---|
| بدون flags | val = index | val | val |
| `CLK_MUX_INDEX_ONE` | Hardware يبدأ من 1 | val = index + 1 | index = val - 1 |
| `CLK_MUX_INDEX_BIT` | كل parent = bit واحد | val = 1 << index | index = ffs(val) - 1 |

الدالتين `clk_mux_val_to_index()` و`clk_mux_index_to_val()` هما اللي بتعمل الترجمة ده — وهما `EXPORT_SYMBOL_GPL` لأن drivers تانية (SoC-specific) ممكن تحتاجهم.

---

### الـ `determine_rate`: ليه مهم للـ Mux؟

الـ **`determine_rate`** هو اللي الـ CCF بيستخدمه لما consumer يطلب rate معين ومحتاج يعرف أحسن parent يختاره. الـ mux نفسه مش بيغير الـ rate (مفيش divider أو multiplier جواه)، بس بيختار من بين parents بـ rates مختلفة.

```c
static int clk_mux_determine_rate(struct clk_hw *hw,
                                  struct clk_rate_request *req)
{
    struct clk_mux *mux = to_clk_mux(hw);
    return clk_mux_determine_rate_flags(hw, req, mux->flags);
}
```

الـ `clk_mux_determine_rate_flags()` (في `clk.c`) بتعدي على كل الـ parents وتحسب أي parent ممكن يوفر أقرب rate للمطلوب. لو `CLK_MUX_ROUND_CLOSEST` مضبوط، بتختار أقرب rate حتى لو أكبر من المطلوب.

---

### الـ Read-Only Variant: `clk_mux_ro_ops`

```c
const struct clk_ops clk_mux_ro_ops = {
    .get_parent = clk_mux_get_parent,
    /* مفيش set_parent ولا determine_rate */
};
```

ده بيستخدم لما الـ mux register محمي من الكتابة (مثلاً boot firmware حدد الـ clock ومش مسموح للـ kernel يغيره). الـ driver بيسجله بـ `CLK_MUX_READ_ONLY` flag.

---

### الـ devres Integration

**الـ `devres`** (device resource management) هو subsystem بيربط حياة الـ resources بحياة الـ `struct device`. لما الـ device بتتشال (unbind)، الـ kernel تلقائياً بيحرر كل الـ resources المسجلة معاه.

```c
struct clk_hw *__devm_clk_hw_register_mux(struct device *dev, ...)
{
    struct clk_hw **ptr;

    /* allocate a "guard" pointer tracked by devres */
    ptr = devres_alloc(devm_clk_hw_release_mux, sizeof(*ptr), GFP_KERNEL);

    hw = __clk_hw_register_mux(...);

    if (!IS_ERR(hw)) {
        *ptr = hw;
        devres_add(dev, ptr); /* ربط الـ cleanup بحياة الـ device */
    }
    return hw;
}

static void devm_clk_hw_release_mux(struct device *dev, void *res)
{
    clk_hw_unregister_mux(*(struct clk_hw **)res); /* cleanup تلقائي */
}
```

---

### ما الـ Subsystem بيمتلكه vs. ما بيفوّضه للـ Driver

| المسؤولية | المالك |
|---|---|
| الـ Clock tree traversal | CCF (`clk.c`) |
| Reference counting للـ parents | CCF (`clk.c`) |
| الـ Locking على مستوى الـ tree (prepare_mutex) | CCF (`clk.c`) |
| قراءة/كتابة الـ hardware register | `clk-mux.c` |
| ترجمة index ↔ register value | `clk-mux.c` |
| الـ Locking على مستوى الـ register (spinlock) | Driver بيجيب الـ lock، `clk-mux.c` بيستخدمه |
| اختيار أنسب parent لـ rate معين | CCF (`clk_mux_determine_rate_flags`) |
| تعريف الـ parents وعددهم | Driver اللي بيسجّل الـ mux |
| الـ devres cleanup | `clk-mux.c` + `devres` subsystem |

---

### مثال حقيقي: i.MX8M Nano

على NXP i.MX8M Nano، الـ UART1 clock مصدره مُختار من mux:

```c
/* من drivers/clk/imx/clk-imx8mn.c */
static const char * const imx8mn_uart1_sels[] = {
    "osc_24m", "sys_pll1_80m", "sys_pll2_200m",
    "sys_pll2_100m", "sys_pll3_out", "clk_ext2",
    "clk_ext4", "audio_pll2_out",
};

/* بيسجل mux بـ register 0xAF00، bits[26:24] */
clk_hw_register_mux(dev, "uart1_sel",
    imx8mn_uart1_sels, ARRAY_SIZE(imx8mn_uart1_sels),
    CLK_SET_RATE_NO_REPARENT,
    base + 0xAF00, 24, 0x7, 0, &lock);
```

لما الـ UART driver بيطلب `clk_set_parent(uart_clk, pll2_200m)`:
1. CCF بيحسب إن `pll2_200m` هو index 2 في الـ array.
2. بيستدعي `clk_mux_set_parent(hw, 2)`.
3. الـ mux بيكتب `0x2` في bits[26:24] من register `0xAF00`.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. Cheatsheet — الـ Flags والـ Enums والـ Config Options

#### CLK_MUX_* Flags (خاصة بـ `struct clk_mux`)

| Flag | Value | المعنى |
|------|-------|--------|
| `CLK_MUX_INDEX_ONE` | `BIT(0)` | الـ register index يبدأ من 1 مش 0 |
| `CLK_MUX_INDEX_BIT` | `BIT(1)` | الـ index عبارة عن single bit (power of two) |
| `CLK_MUX_HIWORD_MASK` | `BIT(2)` | الـ mux bits في الـ low 16-bit، الـ mask في الـ high 16-bit — الكتابة بتحدّث الاتنين |
| `CLK_MUX_READ_ONLY` | `BIT(3)` | الـ register للقراءة بس، مينفعش يتغير |
| `CLK_MUX_ROUND_CLOSEST` | `BIT(4)` | اختار الـ parent اللي أقرب للـ rate المطلوب |
| `CLK_MUX_BIG_ENDIAN` | `BIT(5)` | اقرأ/اكتب الـ register بـ big-endian بدل little-endian |

#### CLK_* Framework Flags (خاصة بـ `struct clk_init_data`)

| Flag | Value | المعنى |
|------|-------|--------|
| `CLK_SET_RATE_GATE` | `BIT(0)` | لازم يتقفل الـ clock أثناء تغيير الـ rate |
| `CLK_SET_PARENT_GATE` | `BIT(1)` | لازم يتقفل أثناء تغيير الـ parent |
| `CLK_SET_RATE_PARENT` | `BIT(2)` | ارفع طلب تغيير الـ rate للـ parent |
| `CLK_IGNORE_UNUSED` | `BIT(3)` | متقفلوش حتى لو محدش بيستخدمه |
| `CLK_GET_RATE_NOCACHE` | `BIT(6)` | متستخدمش الـ cached rate، اقرأ من الـ HW |
| `CLK_SET_RATE_NO_REPARENT` | `BIT(7)` | متعملش reparent أثناء تغيير الـ rate |
| `CLK_IS_CRITICAL` | `BIT(11)` | متقفلوش أبدًا |
| `CLK_OPS_PARENT_ENABLE` | `BIT(12)` | فعّل الـ parents أثناء gate/ungate وset_rate وre-parent |

---

### 1. كل الـ Structs المهمة

#### `struct clk_mux` — البطل الرئيسي

**الغرض:** بيمثل الـ multiplexer clock، يعني clock بيتنقل بين أكتر من مصدر (parent) عن طريق register في الـ hardware.

```c
struct clk_mux {
    struct clk_hw   hw;       /* embed — ربط مع الـ CCF framework */
    void __iomem   *reg;      /* عنوان الـ hardware register */
    const u32      *table;    /* mapping بين الـ index والـ register value (optional) */
    u32             mask;     /* الـ bitmask للحقل في الـ register */
    u8              shift;    /* كام bit نشيل قبل نقرأ/نكتب الـ field */
    u8              flags;    /* CLK_MUX_* flags */
    spinlock_t     *lock;     /* حماية الـ shared register من الـ race conditions */
};
```

| Field | النوع | الدور |
|-------|-------|-------|
| `hw` | `struct clk_hw` | الـ embed اللي بيربط الـ mux بالـ CCF — النقطة الوحيدة اللي الـ framework بيتعامل معاها |
| `reg` | `void __iomem *` | عنوان الـ MMIO register اللي بيتحكم في اختيار الـ parent |
| `table` | `const u32 *` | لو الـ register values مش متتالية (مثلًا: parent 0 → value 3, parent 1 → value 7)، الـ table بيعمل التحويل |
| `mask` | `u32` | الـ bitmask للـ field — مثلًا `0x3` لـ 2-bit mux |
| `shift` | `u8` | عدد الـ bits اللي لازم نشيلها (shift right) عشان نوصل للـ field |
| `flags` | `u8` | `CLK_MUX_*` flags اللي بتتحكم في سلوك القراءة والكتابة والتحويل |
| `lock` | `spinlock_t *` | لو الـ register متشاركة مع كلوكات تانية، الـ spinlock بيأمن الوصول — ممكن يكون NULL |

---

#### `struct clk_hw` — الجسر بين الـ driver والـ framework

**الغرض:** كل hardware clock بيـ embed الـ `clk_hw` جواه — ده اللي الـ CCF بيشوفه.

```c
struct clk_hw {
    struct clk_core *core;            /* الكيان الداخلي للـ CCF */
    struct clk      *clk;             /* الـ handle للـ consumer API */
    const struct clk_init_data *init; /* بيتبقى NULL بعد الـ registration */
};
```

---

#### `struct clk_init_data` — بيانات التسجيل الأولي

**الغرض:** بيتبعت للـ CCF أثناء الـ `clk_hw_register()` بس — بعدها الـ framework بيحتفظ بنسخته هو.

```c
struct clk_init_data {
    const char              *name;          /* اسم الـ clock */
    const struct clk_ops    *ops;           /* جدول العمليات */
    const char * const      *parent_names;  /* أسماء الـ parents (string-based) */
    const struct clk_parent_data *parent_data; /* parent data (mixed internal/external) */
    const struct clk_hw    **parent_hws;    /* pointers للـ parents (all-internal) */
    u8                       num_parents;   /* عدد الـ parents */
    unsigned long            flags;         /* CLK_* framework flags */
};
```

---

#### `struct clk_ops` — جدول العمليات (vtable)

**الغرض:** الـ interface بين الـ CCF وأي hardware clock. الـ `clk-mux.c` بيملّي منه بس 3 callbacks:

| Callback | مين بيملّيه | الدور |
|----------|------------|-------|
| `get_parent` | `clk_mux_get_parent` | اقرأ الـ register وارجع index الـ parent الحالي |
| `set_parent` | `clk_mux_set_parent` | اكتب في الـ register عشان تغير الـ parent |
| `determine_rate` | `clk_mux_determine_rate` | اختار أفضل parent يحقق الـ rate المطلوب |

---

#### `struct clk_rate_request` — طلب تغيير الـ rate

**الغرض:** بيتمرر للـ `determine_rate` callback — بيحمل القيود والمتطلبات.

```c
struct clk_rate_request {
    struct clk_core *core;             /* الـ clock اللي بيتطلب */
    unsigned long    rate;             /* الـ rate المطلوب */
    unsigned long    min_rate;         /* الحد الأدنى المسموح */
    unsigned long    max_rate;         /* الحد الأقصى المسموح */
    unsigned long    best_parent_rate; /* أفضل rate للـ parent */
    struct clk_hw   *best_parent_hw;   /* أفضل parent اختاره الـ framework */
};
```

---

#### `struct clk_parent_data` — بيانات الـ parent

**الغرض:** بيوصف الـ parent بطريقة مرنة — بيسمح بالـ parents الداخليين والخارجيين مع بعض.

```c
struct clk_parent_data {
    const struct clk_hw *hw;       /* pointer مباشر للـ parent (داخلي) */
    const char          *fw_name;  /* اسم الـ parent في الـ device tree */
    const char          *name;     /* اسم عالمي للـ parent (fallback) */
    int                  index;    /* index في الـ DT (لو مفيش fw_name) */
};
```

---

### 2. ASCII Diagram — علاقات الـ Structs

```
┌─────────────────────────────────────────────────────────────┐
│                      struct clk_mux                         │
│                                                             │
│  ┌──────────────────┐  ← embed (first field)               │
│  │  struct clk_hw   │──────────────────────────────────┐   │
│  │  ─────────────── │                                  │   │
│  │  *core ──────────┼──────► struct clk_core (CCF)     │   │
│  │  *clk  ──────────┼──────► struct clk (consumer)     │   │
│  │  *init ──────────┼──► struct clk_init_data           │   │
│  └──────────────────┘    (NULL after register)          │   │
│                                                          │   │
│  *reg  ─────────────────► MMIO register (hardware)      │   │
│  *table ────────────────► u32[] (index→val mapping)     │   │
│  mask  = 0x3 (example)                                  │   │
│  shift = 4   (example)                                  │   │
│  flags = CLK_MUX_BIG_ENDIAN | ...                       │   │
│  *lock ─────────────────► spinlock_t (shared)           │   │
└─────────────────────────────────────────────────────────────┘
         │ to_clk_mux(hw)
         │ container_of(hw, struct clk_mux, hw)
         ▼
┌─────────────────────────────────────────────────────────────┐
│                   struct clk_init_data                       │
│  name = "mux_clk"                                           │
│  ops  ──────────────────► struct clk_ops                    │
│                           ├── .get_parent = clk_mux_get_parent
│                           ├── .set_parent = clk_mux_set_parent
│                           └── .determine_rate = clk_mux_determine_rate
│  parent_names[] ──────────► ["parent0", "parent1", ...]     │
│  num_parents = N                                            │
│  flags = CLK_SET_RATE_PARENT | ...                          │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│              Hardware Register Layout                        │
│                                                             │
│  31      16 15    shift+N  shift     0                      │
│  ┌────────┬──────────────┬──────────┐                       │
│  │  MASK  │   other bits │  MUX SEL │                       │
│  │(HIWORD)│              │ (masked) │                       │
│  └────────┴──────────────┴──────────┘                       │
│  ↑ only used with CLK_MUX_HIWORD_MASK                       │
└─────────────────────────────────────────────────────────────┘
```

---

### 3. Lifecycle Diagram — دورة حياة الـ clk_mux

```
         CREATION
            │
            ▼
  driver calls clk_hw_register_mux(dev, name, parents, ...)
            │
            ▼
  __clk_hw_register_mux()
  ┌─────────────────────────────────────────┐
  │ 1. validate CLK_MUX_HIWORD_MASK bounds  │
  │ 2. kzalloc(sizeof(struct clk_mux))      │
  │ 3. fill clk_init_data{}                 │
  │    - name, ops, parents, flags          │
  │ 4. fill struct clk_mux fields           │
  │    - reg, shift, mask, flags, lock, table│
  │ 5. mux->hw.init = &init                 │
  └─────────────────────────────────────────┘
            │
            ▼
  REGISTRATION
  ┌─────────────────────────────────────────┐
  │ if (dev)  → clk_hw_register(dev, hw)    │
  │ else if (np) → of_clk_hw_register(np,hw)│
  │                                         │
  │ CCF copies init data into clk_core      │
  │ hw->init = NULL  (freed reference)      │
  └─────────────────────────────────────────┘
            │
            ▼
  USAGE
  ┌─────────────────────────────────────────┐
  │ clk_set_parent()  → set_parent ops      │
  │ clk_get_parent()  → get_parent ops      │
  │ clk_set_rate()    → determine_rate ops  │
  └─────────────────────────────────────────┘
            │
            ▼
  TEARDOWN
  ┌─────────────────────────────────────────┐
  │ clk_hw_unregister_mux(hw)               │
  │   → clk_hw_unregister(hw)  (CCF cleanup)│
  │   → kfree(mux)             (free struct) │
  │                                         │
  │ devm variant:                           │
  │   devm_clk_hw_release_mux() يتّنادى     │
  │   أوتوماتيك لما الـ device يتـ remove   │
  └─────────────────────────────────────────┘
```

---

### 4. Call Flow Diagrams

#### 4.1 تسجيل الـ mux clock

```
driver/platform code
  │
  └─► clk_hw_register_mux(dev, "mux", parents, N, flags, reg, shift, mask, mux_flags, NULL, lock)
        │  [macro expands to __clk_hw_register_mux]
        │
        └─► __clk_hw_register_mux()
              │
              ├─► validate: CLK_MUX_HIWORD_MASK width + shift <= 16
              ├─► kzalloc(struct clk_mux)
              ├─► fill clk_init_data { name, ops=&clk_mux_ops, parents, flags }
              ├─► fill clk_mux { reg, shift, mask, flags, lock, table }
              │
              ├─► [if dev]  clk_hw_register(dev, hw)
              │     └─► CCF: allocate clk_core, link hw↔core, register in clock tree
              │
              └─► [if np]   of_clk_hw_register(np, hw)
                    └─► CCF: same + link to DT node
```

#### 4.2 تغيير الـ parent — `clk_set_parent()`

```
consumer: clk_set_parent(clk, new_parent)
  │
  └─► CCF core: clk_core_set_parent_nolock()
        │
        └─► ops->set_parent(hw, index)
              │  [= clk_mux_set_parent]
              │
              ├─► clk_mux_index_to_val(table, flags, index)
              │     ├─► [table]       val = table[index]
              │     ├─► [INDEX_BIT]   val = 1 << index
              │     ├─► [INDEX_ONE]   val = index + 1
              │     └─► [default]     val = index
              │
              ├─► spin_lock_irqsave(mux->lock)   ← لو lock موجود
              │
              ├─► [HIWORD_MASK]  reg = mask << (shift + 16)
              │   [else]         reg = readl(); reg &= ~(mask << shift)
              │
              ├─► reg |= (val << shift)
              ├─► clk_mux_writel(mux, reg)
              │     ├─► [BIG_ENDIAN]  iowrite32be(val, reg)
              │     └─► [default]     writel(val, reg)
              │
              └─► spin_unlock_irqrestore(mux->lock)
```

#### 4.3 قراءة الـ parent الحالي — `clk_get_parent()`

```
consumer: clk_get_parent(clk)
  │
  └─► CCF core: clk_core_get_parent()
        │
        └─► ops->get_parent(hw)
              │  [= clk_mux_get_parent]
              │
              ├─► clk_mux_readl(mux)
              │     ├─► [BIG_ENDIAN]  ioread32be(mux->reg)
              │     └─► [default]     readl(mux->reg)
              │
              ├─► val = (reg_val >> mux->shift) & mux->mask
              │
              └─► clk_mux_val_to_index(hw, table, flags, val)
                    ├─► [table]       linear search: table[i] == val → return i
                    ├─► [INDEX_BIT]   index = ffs(val) - 1
                    ├─► [INDEX_ONE]   index = val - 1
                    └─► [default]     index = val
```

#### 4.4 اختيار أفضل parent — `clk_set_rate()`

```
consumer: clk_set_rate(clk, desired_rate)
  │
  └─► CCF: clk_core_determine_round_nolock()
        │
        └─► ops->determine_rate(hw, req)
              │  [= clk_mux_determine_rate]
              │
              └─► clk_mux_determine_rate_flags(hw, req, mux->flags)
                    │  [defined in clk.c]
                    │
                    ├─► loop over all parents
                    │     └─► ask each parent: what rate can you give?
                    │
                    ├─► [CLK_MUX_ROUND_CLOSEST]  اختار الأقرب
                    └─► [default]                اختار اللي مش أعلى من المطلوب
```

#### 4.5 الـ devm variant — تسجيل مع الـ device lifecycle

```
driver: devm_clk_hw_register_mux(dev, ...)
  │
  └─► __devm_clk_hw_register_mux()
        │
        ├─► devres_alloc(devm_clk_hw_release_mux, sizeof(struct clk_hw *))
        │     └─► allocate devres slot to hold the hw pointer
        │
        ├─► __clk_hw_register_mux(...)   ← نفس المسار العادي
        │
        ├─► [success] *ptr = hw; devres_add(dev, ptr)
        │     └─► لما device يتـ remove، الـ kernel ينادي devm_clk_hw_release_mux()
        │               └─► clk_hw_unregister_mux(hw)
        │                     ├─► clk_hw_unregister(hw)
        │                     └─► kfree(mux)
        │
        └─► [failure] devres_free(ptr)
```

---

### 5. Locking Strategy

#### القفل المستخدم: `spinlock_t *lock`

الـ `struct clk_mux` بيحمل **pointer** للـ spinlock مش الـ spinlock نفسه — ده مقصود عشان يسمح لكذا clock يتشاركوا نفس الـ lock لو بيـ share نفس الـ register.

```
                  ┌─────────────────────────────────┐
                  │         Shared Register          │
                  │   bits[3:2] = mux_A selector     │
                  │   bits[5:4] = mux_B selector     │
                  └─────────────────────────────────┘
                           ▲           ▲
                           │           │
                    mux_A->reg    mux_B->reg
                    mux_A->lock   mux_B->lock
                           │           │
                           └─────┬─────┘
                                 │
                           shared spinlock_t
```

#### متى يُستخدم القفل؟

| العملية | قفل؟ | السبب |
|---------|-------|-------|
| `clk_mux_set_parent()` | نعم — `spin_lock_irqsave` | read-modify-write على register مشترك — atomic |
| `clk_mux_get_parent()` | لا | قراءة فقط، الـ CCF عنده `prepare_lock` خاص بيه |
| `__clk_hw_register_mux()` | لا | بيتنادى مرة واحدة وقت الـ init |
| `clk_hw_unregister_mux()` | لا | الـ CCF بيضمن عدم التزامن وقت الـ teardown |

#### تسلسل القفل (Lock Ordering)

```
CCF prepare_mutex  (mutex — يسمح بالنوم)
  └─► CCF enable_lock  (spinlock — لا ينوم)
        └─► mux->lock  (spinlock — لا ينوم)
```

**ملاحظة مهمة:** الـ `spin_lock_irqsave` في `clk_mux_set_parent` بيـ disable الـ interrupts عشان يحمي من الـ IRQ handlers اللي ممكن تعمل `clk_set_parent` هي كمان — مش بس الـ race بين الـ threads.

#### لما يكون `lock == NULL`

```c
if (mux->lock)
    spin_lock_irqsave(mux->lock, flags);
else
    __acquire(mux->lock);   /* static analysis hint فقط، مفيش قفل فعلي */
```

ده بيحصل لما الـ register مش متشارك مع حاجة تانية — الـ driver بيعدي `NULL` وقت التسجيل. الـ `__acquire/__release` هنا بس عشان أدوات الـ static analysis (زي `sparse`) متشتكيش من lock imbalance.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### الـ CLK_MUX Flags

| Flag | Value | المعنى |
|------|-------|--------|
| `CLK_MUX_INDEX_ONE` | BIT(0) | الـ register index يبدأ من 1 مش 0 |
| `CLK_MUX_INDEX_BIT` | BIT(1) | الـ index عبارة عن single bit (power of two) |
| `CLK_MUX_HIWORD_MASK` | BIT(2) | الـ mask في الـ upper 16-bit، الـ value في الـ lower 16-bit |
| `CLK_MUX_READ_ONLY` | BIT(3) | الـ mux مش ممكن يتكتب — بس read |
| `CLK_MUX_ROUND_CLOSEST` | BIT(4) | اختار الـ parent الأقرب للـ target rate |
| `CLK_MUX_BIG_ENDIAN` | BIT(5) | استخدم big-endian I/O بدل little-endian |

#### جدول كل الـ Functions

| Function | نوعها | Export؟ | الغرض |
|----------|-------|---------|-------|
| `clk_mux_readl` | static inline | لأ | قراءة الـ mux register (BE/LE aware) |
| `clk_mux_writel` | static inline | لأ | كتابة الـ mux register (BE/LE aware) |
| `clk_mux_val_to_index` | exported | نعم | تحويل register value → parent index |
| `clk_mux_index_to_val` | exported | نعم | تحويل parent index → register value |
| `clk_mux_get_parent` | static (clk_op) | لأ | قراءة الـ current parent من الـ HW |
| `clk_mux_set_parent` | static (clk_op) | لأ | كتابة parent جديد على الـ HW |
| `clk_mux_determine_rate` | static (clk_op) | لأ | تحديد أنسب rate عن طريق الـ parents |
| `__clk_hw_register_mux` | exported | نعم | تسجيل mux clock (الـ core function) |
| `devm_clk_hw_release_mux` | static devres cb | لأ | cleanup callback للـ devres |
| `__devm_clk_hw_register_mux` | exported | نعم | تسجيل mux مع devres lifetime management |
| `clk_register_mux_table` | exported | نعم | legacy API — يرجع `struct clk *` |
| `clk_unregister_mux` | exported | نعم | legacy unregister — يحرر الـ clk + mux |
| `clk_hw_unregister_mux` | exported | نعم | modern unregister — يحرر الـ hw + mux |

---

### Group 1: I/O Helpers — قراءة وكتابة الـ Register

الهدف من المجموعة دي هو تجريد الـ endianness عن باقي الكود. كل access للـ register بيمر من هنا عشان لو الـ `CLK_MUX_BIG_ENDIAN` flag اتضبط، نستخدم `ioread32be/iowrite32be` بدل `readl/writel` العادية.

---

#### `clk_mux_readl`

```c
static inline u32 clk_mux_readl(struct clk_mux *mux)
```

**الـ function دي** بتقرأ الـ 32-bit register اللي بيتحكم في الـ mux. لو الـ `CLK_MUX_BIG_ENDIAN` flag موجود في `mux->flags` بتستخدم `ioread32be` عشان تعمل byte-swap، غير كده بتستخدم `readl` العادية.

**Parameters:**
- `mux`: pointer لـ `struct clk_mux` اللي فيها الـ `reg` (MMIO address) والـ `flags`.

**Return:** الـ raw u32 value اللي اتقرأ من الـ register.

**Key details:** الـ function دي `static inline` — مفيش call overhead. الـ `readl` في حد ذاتها بتعمل `rmb()` implicit على بعض الـ architectures.

---

#### `clk_mux_writel`

```c
static inline void clk_mux_writel(struct clk_mux *mux, u32 val)
```

**الـ function دي** بتكتب الـ value على الـ mux register مع مراعاة الـ endianness. بنفس المنطق: `CLK_MUX_BIG_ENDIAN` → `iowrite32be`، غير كده → `writel`.

**Parameters:**
- `mux`: الـ mux descriptor.
- `val`: الـ value اللي هتتكتب على الـ register.

**Key details:** لازم تتضمن الـ mask والـ shift في الـ `val` قبل ما تعدي للـ function دي. دايمًا بيتسبقها lock في `clk_mux_set_parent`.

---

### Group 2: Index/Value Translation Helpers

المجموعة دي بتترجم بين الـ parent index (اللي الـ CCF شايفه) والـ register value (اللي الـ hardware بيفهمه). الترجمة دي ممكن تكون:
1. **Direct** — الـ index = الـ value.
2. **Table-based** — lookup في `mux->table[]`.
3. **Bit-encoded** (`CLK_MUX_INDEX_BIT`) — الـ value = `1 << index`.
4. **One-based** (`CLK_MUX_INDEX_ONE`) — الـ value = `index + 1`.

---

#### `clk_mux_val_to_index`

```c
int clk_mux_val_to_index(struct clk_hw *hw, const u32 *table,
                         unsigned int flags, unsigned int val)
```

**الـ function دي** بتأخد الـ raw hardware value (اللي اتقرأت من الـ register) وترجع الـ index بتاعها في array الـ parents. لو في `table` بتعمل linear search عليها، لو مفيش بتطبق الـ flags logic.

**Parameters:**
- `hw`: handle للـ CCF عشان نعرف `num_parents`.
- `table`: لو مش NULL — array من الـ register values, كل entry بتقابل parent بنفس الـ index.
- `flags`: `CLK_MUX_INDEX_BIT` أو `CLK_MUX_INDEX_ONE`.
- `val`: الـ raw bitfield value اللي اتقرأ من الـ hardware.

**Return:** الـ parent index على النجاح، أو `-EINVAL` لو الـ val مش valid.

**Key details:** الـ function دي exported وبتتسمى من `clk_mux_get_parent`. لو الـ table مش NULL والـ val مش موجود فيها — `-EINVAL`. لو مفيش table والـ val >= `num_parents` — `-EINVAL`.

**Pseudocode:**
```
if table:
    for i in 0..num_parents:
        if table[i] == val: return i
    return -EINVAL

if CLK_MUX_INDEX_BIT and val != 0:
    val = ffs(val) - 1   // find first set bit position

if CLK_MUX_INDEX_ONE and val != 0:
    val--

if val >= num_parents: return -EINVAL
return val
```

---

#### `clk_mux_index_to_val`

```c
unsigned int clk_mux_index_to_val(const u32 *table, unsigned int flags, u8 index)
```

**الـ function دي** عكس `clk_mux_val_to_index` — بتأخد الـ parent index وترجع الـ register value المناسب. لو في `table` بترجع `table[index]` مباشرة، غير كده بتطبق الـ flags logic.

**Parameters:**
- `table`: نفس array الـ translation، أو NULL.
- `flags`: `CLK_MUX_INDEX_BIT` أو `CLK_MUX_INDEX_ONE`.
- `index`: الـ parent index.

**Return:** الـ u32 value اللي هيتكتب في الـ register bitfield.

**Key details:** مفيش bounds checking هنا — المتصل مسؤول إن الـ index valid. الـ function دي مش بتعمل لocking — دي pure computation.

---

### Group 3: CLK_OPS Callbacks — قلب الـ Driver

الـ CCF بيستدعي الـ callbacks دي من خلال الـ `clk_ops` pointers. الـ `clk_mux_ops` فيها الـ 3 callbacks التالية، أما `clk_mux_ro_ops` فيها `get_parent` بس (read-only mux).

---

#### `clk_mux_get_parent`

```c
static u8 clk_mux_get_parent(struct clk_hw *hw)
```

**الـ function دي** بتقرأ الـ hardware register وترجع index الـ parent الحالي. بتعمل read-modify ثم تترجم بـ `clk_mux_val_to_index`.

**Parameters:**
- `hw`: الـ clock handle، بنحول منه لـ `struct clk_mux *` بالـ `to_clk_mux()`.

**Return:** u8 index للـ parent الحالي.

**Key details:** الـ CCF بيستدعيها وقت الـ `__clk_init`. مفيش locking هنا لأن الـ CCF بيحمي بالـ `prepare_lock` في الـ init path. الـ shift والـ mask بيتطبقوا على الـ raw register value قبل الترجمة.

**Pseudocode:**
```
val = clk_mux_readl(mux)
val >>= mux->shift
val &= mux->mask
return clk_mux_val_to_index(hw, mux->table, mux->flags, val)
```

---

#### `clk_mux_set_parent`

```c
static int clk_mux_set_parent(struct clk_hw *hw, u8 index)
```

**الـ function دي** هي الأهم في الـ driver — بتحول الـ parent index لـ register value وبتكتبها على الـ HW مع الحماية بالـ spinlock. بتدعم `CLK_MUX_HIWORD_MASK` اللي بيخلي الـ write atomic بدون read-modify-write كامل.

**Parameters:**
- `hw`: الـ clock handle.
- `index`: الـ parent index الجديد.

**Return:** دايمًا 0 (success).

**Key details:**
- **Locking**: لو `mux->lock != NULL` بيستخدم `spin_lock_irqsave` — ضروري لأن الـ `clk_enable` ممكن يتسمى من interrupt context. لو NULL — الـ `__acquire/__release` annotations بس للـ sparse checker.
- **`CLK_MUX_HIWORD_MASK`**: مش بيعمل read أصلاً — بيبني الـ reg من الـ mask في الـ upper 16 bits مباشرة. ده بيضمن atomic read-modify-write في hardware بيدعمه.
- **Normal path**: read → clear bits → set new value → write.

**Pseudocode:**
```
val = clk_mux_index_to_val(mux->table, mux->flags, index)

spin_lock_irqsave(mux->lock, flags)  // if lock != NULL

if CLK_MUX_HIWORD_MASK:
    reg = mux->mask << (mux->shift + 16)  // mask bits in upper word
else:
    reg = clk_mux_readl(mux)
    reg &= ~(mux->mask << mux->shift)     // clear old value

reg |= (val << mux->shift)
clk_mux_writel(mux, reg)

spin_unlock_irqrestore(mux->lock, flags)
return 0
```

---

#### `clk_mux_determine_rate`

```c
static int clk_mux_determine_rate(struct clk_hw *hw,
                                  struct clk_rate_request *req)
```

**الـ function دي** thin wrapper حول `clk_mux_determine_rate_flags` (موجودة في `clk.c`). بتمرر لها الـ `mux->flags` عشان الـ `CLK_MUX_ROUND_CLOSEST` flag يتأخد في الاعتبار.

**Parameters:**
- `hw`: الـ clock handle.
- `req`: الـ `struct clk_rate_request` — فيها الـ target rate والـ constraints، والـ framework بيملأ فيها `best_parent_hw` و`best_parent_rate`.

**Return:** 0 على النجاح، error code على الفشل.

**Key details:** مش بتلمس الـ hardware. الـ CCF بيستدعيها في الـ `clk_set_rate` path عشان يختار أفضل parent يحقق الـ requested rate.

---

### Group 4: Registration Functions — تسجيل الـ Clock

---

#### `__clk_hw_register_mux`

```c
struct clk_hw *__clk_hw_register_mux(struct device *dev, struct device_node *np,
    const char *name, u8 num_parents,
    const char * const *parent_names,
    const struct clk_hw **parent_hws,
    const struct clk_parent_data *parent_data,
    unsigned long flags, void __iomem *reg, u8 shift, u32 mask,
    u8 clk_mux_flags, const u32 *table, spinlock_t *lock)
```

**الـ function دي** هي الـ core registration function. بتعمل `kzalloc` لـ `struct clk_mux`، بتملأ الـ `clk_init_data`، وبتسجل الـ clock مع الـ CCF عن طريق `clk_hw_register` أو `of_clk_hw_register` حسب السياق.

**Parameters:**
- `dev`: الـ device owner — لو موجود يُستخدم `clk_hw_register`.
- `np`: الـ device node — لو مفيش `dev` يُستخدم `of_clk_hw_register`.
- `name`: اسم الـ clock في الـ CCF tree.
- `num_parents`: عدد الـ parents.
- `parent_names` / `parent_hws` / `parent_data`: طريقة تعريف الـ parents — بس واحدة منهم تكون non-NULL.
- `flags`: الـ CCF-level flags (`CLK_SET_RATE_PARENT`, etc.).
- `reg`: الـ MMIO address للـ control register.
- `shift`: عدد bits نعمل shift بيها قبل نقرأ/نكتب الـ bitfield.
- `mask`: الـ bitmask للـ mux bitfield (بعد الـ shift).
- `clk_mux_flags`: الـ driver-level flags (`CLK_MUX_*`).
- `table`: لو non-NULL — lookup table للترجمة.
- `lock`: الـ spinlock المشترك للـ register.

**Return:** `struct clk_hw *` على النجاح، `ERR_PTR` على الفشل.

**Key details:**
- لو `CLK_MUX_HIWORD_MASK` مضبوط، بيتحقق إن `fls(mask) - ffs(mask) + 1 + shift <= 16`، عشان الـ mux bits تفضل في الـ lower 16-bit — غير كده `ERR_PTR(-EINVAL)` فورًا.
- لو `CLK_MUX_READ_ONLY` مضبوط، بيحط `clk_mux_ro_ops` عوض `clk_mux_ops`.
- لو الـ registration فشلت — بيعمل `kfree(mux)` ويرجع الـ error.
- الـ `mux->hw.init` بيتصفر (NULL) بعد الـ registration بواسطة الـ CCF.

**Pseudocode:**
```
if CLK_MUX_HIWORD_MASK:
    width = fls(mask) - ffs(mask) + 1
    if width + shift > 16: return ERR_PTR(-EINVAL)

mux = kzalloc(sizeof(*mux), GFP_KERNEL)
if !mux: return ERR_PTR(-ENOMEM)

init.name = name
init.ops = (READ_ONLY) ? &clk_mux_ro_ops : &clk_mux_ops
init.flags = flags
init.parent_* = ...
init.num_parents = num_parents

mux->reg = reg; mux->shift = shift; mux->mask = mask
mux->flags = clk_mux_flags; mux->lock = lock; mux->table = table
mux->hw.init = &init

if dev || !np:
    ret = clk_hw_register(dev, &mux->hw)
else:
    ret = of_clk_hw_register(np, &mux->hw)

if ret:
    kfree(mux); return ERR_PTR(ret)

return &mux->hw
```

---

#### `devm_clk_hw_release_mux`

```c
static void devm_clk_hw_release_mux(struct device *dev, void *res)
```

**الـ function دي** هي الـ devres release callback. الـ devres framework بيستدعيها أوتوماتيك لما الـ device يتفك. بتستدعي `clk_hw_unregister_mux` عشان تعمل unregister وتحرر الـ memory.

**Parameters:**
- `dev`: الـ device (مش مستخدمة هنا مباشرة).
- `res`: pointer لـ `struct clk_hw **` اللي اتخزنت وقت الـ `devres_alloc`.

**Key details:** الـ devres mechanism بيضمن إن الـ function دي هتتسمى حتى لو الـ probe فشل جزئيًا. الـ `*(struct clk_hw **)res` deref ضروري لأن `res` هو pointer لـ pointer.

---

#### `__devm_clk_hw_register_mux`

```c
struct clk_hw *__devm_clk_hw_register_mux(struct device *dev, struct device_node *np,
    const char *name, u8 num_parents,
    const char * const *parent_names,
    const struct clk_hw **parent_hws,
    const struct clk_parent_data *parent_data,
    unsigned long flags, void __iomem *reg, u8 shift, u32 mask,
    u8 clk_mux_flags, const u32 *table, spinlock_t *lock)
```

**الـ function دي** wrapper فوق `__clk_hw_register_mux` بتضيف devres lifetime management. بتعمل `devres_alloc` بالحجم المناسب وبعد كده بتسجل الـ clock. لو النجاح، بتحط الـ `hw` pointer في الـ devres slot وبتعمل `devres_add`. لو الفشل، بتعمل `devres_free`.

**Parameters:** نفس `__clk_hw_register_mux` بالظبط.

**Return:** `struct clk_hw *` على النجاح، `ERR_PTR` على الفشل.

**Key details:** المتصل مش محتاج يستدعي `clk_hw_unregister_mux` يدويًا — الـ devres هيتكفل. الفرق الوحيد عن `__clk_hw_register_mux` هو الـ `ptr` allocation قبل التسجيل.

**Pseudocode:**
```
ptr = devres_alloc(devm_clk_hw_release_mux, sizeof(*ptr), GFP_KERNEL)
if !ptr: return ERR_PTR(-ENOMEM)

hw = __clk_hw_register_mux(dev, np, name, ...)

if !IS_ERR(hw):
    *ptr = hw
    devres_add(dev, ptr)
else:
    devres_free(ptr)

return hw
```

---

### Group 5: Legacy API — الـ `struct clk *` Interface

---

#### `clk_register_mux_table`

```c
struct clk *clk_register_mux_table(struct device *dev, const char *name,
    const char * const *parent_names, u8 num_parents,
    unsigned long flags, void __iomem *reg, u8 shift, u32 mask,
    u8 clk_mux_flags, const u32 *table, spinlock_t *lock)
```

**الـ function دي** legacy wrapper بترجع `struct clk *` بدل `struct clk_hw *`. بتستدعي `clk_hw_register_mux_table` (macro يوصل لـ `__clk_hw_register_mux`) وبعد كده بترجع `hw->clk`.

**Parameters:** نفس parameters الـ hw version بس بدون `parent_hws/parent_data`.

**Return:** `struct clk *` على النجاح، `ERR_PTR` على الفشل.

**Key details:** الـ API ده legacy — الكود الجديد لازم يستخدم `clk_hw_register_mux` variants. `ERR_CAST(hw)` بتعمل type-safe cast من `ERR_PTR` value.

---

### Group 6: Unregistration / Cleanup

---

#### `clk_unregister_mux`

```c
void clk_unregister_mux(struct clk *clk)
```

**الـ function دي** legacy unregister. بتاخد `struct clk *`، بتعمل `__clk_get_hw` عشان تجيب الـ `struct clk_hw *`، وبعدين تحول لـ `struct clk_mux *` وتعمل `clk_unregister` ثم `kfree`.

**Parameters:**
- `clk`: الـ clock المراد حذفه.

**Key details:** لو `clk` invalid أو `__clk_get_hw` رجع NULL — الـ function بترجع بصمت. مقابلها `clk_register_mux_table`.

---

#### `clk_hw_unregister_mux`

```c
void clk_hw_unregister_mux(struct clk_hw *hw)
```

**الـ function دي** الـ modern unregister المقابلة لـ `__clk_hw_register_mux`. بتحول `hw` لـ `struct clk_mux *` بالـ `to_clk_mux()`، بتستدعي `clk_hw_unregister`، وبعدين `kfree(mux)`.

**Parameters:**
- `hw`: الـ hw handle اللي رجع من الـ register function.

**Key details:** لو الـ clock اتسجل بالـ devm variant — مش المفروض تستدعي الـ function دي يدويًا. الـ `devm_clk_hw_release_mux` هي اللي بتستدعيها. ترتيب الـ operations مهم: الـ unregister الأول عشان الـ CCF يوقف أي reference، وبعدين الـ `kfree`.

---

### الـ clk_ops Tables

```c
/* Full-featured mux — read/write */
const struct clk_ops clk_mux_ops = {
    .get_parent     = clk_mux_get_parent,
    .set_parent     = clk_mux_set_parent,
    .determine_rate = clk_mux_determine_rate,
};

/* Read-only mux — hardware لا يسمح بتغيير الـ parent */
const struct clk_ops clk_mux_ro_ops = {
    .get_parent = clk_mux_get_parent,
};
```

الـ `clk_mux_ro_ops` مش فيها `set_parent` — لو الـ CCF حاول يغير الـ parent على read-only mux، العملية هتفشل بـ `-EPERM` على مستوى الـ framework.

---

### ASCII Diagram — تدفق الـ set_parent

```
clk_set_parent(clk, new_parent)
        │
        ▼
  [CCF prepare_lock acquired]
        │
        ▼
  clk_mux_set_parent(hw, index)
        │
        ├─ clk_mux_index_to_val(table, flags, index)
        │        │
        │        ├─ table ? → table[index]
        │        ├─ INDEX_BIT ? → 1 << index
        │        ├─ INDEX_ONE ? → index + 1
        │        └─ default   → index
        │
        ├─ spin_lock_irqsave(mux->lock)
        │
        ├─ HIWORD_MASK ?
        │     YES → reg = mask << (shift + 16)   [no read needed]
        │     NO  → reg = clk_mux_readl(mux)
        │                reg &= ~(mask << shift)
        │
        ├─ reg |= (val << shift)
        ├─ clk_mux_writel(mux, reg)
        │
        └─ spin_unlock_irqrestore(mux->lock)
```
## Phase 5: دليل الـ Debugging الشامل

### مقدمة

**الـ `clk-mux`** هو driver بسيط جداً — وظيفته الوحيدة هي قراءة/كتابة bits معينة في register لاختيار parent clock. لكن لما حاجة بتغلط، بتحتاج تعرف: هل الـ register اتكتب صح؟ هل الـ parent اتاخد صح؟ هل في race condition؟ الـ section ده بيجاوب على الأسئلة دي كلها.

---

## Software Level

### 1. مدخلات الـ `debugfs`

الـ clock framework بيعمل entries تحت `/sys/kernel/debug/clk/` لكل clock مسجّل.

```bash
# اعرض كل الـ mux clocks المسجّلة مع والديهم
ls /sys/kernel/debug/clk/

# اقرأ الـ parent الحالي لـ mux clock اسمه "uart_clk_mux"
cat /sys/kernel/debug/clk/uart_clk_mux/clk_parent

# اقرأ الـ rate الحالي
cat /sys/kernel/debug/clk/uart_clk_mux/clk_rate

# اعرض enable count (كم consumer شغّل الـ clock)
cat /sys/kernel/debug/clk/uart_clk_mux/clk_enable_count

# اعرض prepare count
cat /sys/kernel/debug/clk/uart_clk_mux/clk_prepare_count

# شجرة الـ clocks كاملة — مفيد جداً لرؤية العلاقات
cat /sys/kernel/debug/clk/clk_summary

# نفس المعلومات بس بشكل dump
cat /sys/kernel/debug/clk/clk_dump
```

**الـ `clk_summary`** بيطلعلك جدول زي ده:

```
                                 enable  prepare  protect                                duty  hardware
   clock                          count    count    count        rate   accuracy phase  cycle    enable
------------------------------------------------------------------------------------------------------------
 osc24M                               1        1        0    24000000          0     0  50000         Y
    uart_clk_mux                      1        1        0   600000000          0     0  50000         Y
       uart0_div                      1        1        0    50000000          0     0  50000         Y
```

لو `clk_parent` بيرجع اسم parent غلط، يبقى في مشكلة في الـ `clk_mux_get_parent` — يعني الـ register value مش بتتطابق مع أي entry في الـ table.

---

### 2. مدخلات الـ `sysfs`

```bash
# اعرض الـ driver المرتبط بالـ clock device
ls /sys/bus/platform/drivers/ | grep clk

# لو الـ mux متعمل بـ devm_clk_hw_register_mux، هيبان تحت device
ls /sys/devices/platform/<device_name>/

# اعرض الـ clocks المرتبطة بـ device معين
cat /sys/devices/platform/<device_name>/driver/

# power management state للـ device
cat /sys/devices/platform/<device_name>/power/runtime_status
```

> الـ `sysfs` مش بيديك details كتير عن الـ mux نفسه — الـ `debugfs` هو الأهم.

---

### 3. الـ `ftrace` — Tracepoints والـ Events

الـ clock framework عنده tracepoints جاهزة تحت `clk` event group.

```bash
# اعرض كل الـ clock events المتاحة
ls /sys/kernel/debug/tracing/events/clk/

# النتيجة المتوقعة:
# clk_enable  clk_enable_complete  clk_disable  clk_disable_complete
# clk_prepare clk_prepare_complete clk_unprepare clk_unprepare_complete
# clk_set_parent  clk_set_parent_complete
# clk_set_rate  clk_set_rate_complete

# فعّل events الـ set_parent (الأهم للـ mux)
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_set_parent/enable
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_set_parent_complete/enable

# فعّل كل clk events
echo 1 > /sys/kernel/debug/tracing/events/clk/enable

# ابدأ الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# اعمل الـ operation اللي عايز تتتبعها
# مثلاً: شغّل application بتستخدم الـ uart

# اقرأ النتيجة
cat /sys/kernel/debug/tracing/trace
```

**مثال على output:**

```
# TASK-PID     CPU#  |||  TIMESTAMP  FUNCTION
# |            |     |||     |       |
  kworker/0:2-45  [000] ....  123.456: clk_set_parent: uart_clk_mux
  kworker/0:2-45  [000] ....  123.457: clk_set_parent_complete: uart_clk_mux
```

لو شايف `clk_set_parent` من غير `clk_set_parent_complete` — يبقى في deadlock أو error في الـ `clk_mux_set_parent`.

```bash
# فلتر على clock معين بالاسم
echo "name == \"uart_clk_mux\"" > /sys/kernel/debug/tracing/events/clk/clk_set_parent/filter

# استخدم function tracer لتتبع دخول clk_mux_set_parent
echo clk_mux_set_parent > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

---

### 4. الـ `printk` والـ Dynamic Debug

```bash
# فعّل dynamic debug لملف clk-mux.c كله
echo "file clk-mux.c +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل dynamic debug لكل ملفات الـ clk subsystem
echo "file drivers/clk/* +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل مع stacktrace لكل message
echo "file clk-mux.c +ps" > /sys/kernel/debug/dynamic_debug/control

# شوف الـ dynamic debug الحالي
cat /sys/kernel/debug/dynamic_debug/control | grep clk-mux

# زوّد الـ loglevel مؤقتاً
echo 8 > /proc/sys/kernel/printk
```

لإضافة debug prints مؤقتة في الكود بدون recompile:

```bash
# kernel compiled with CONFIG_DYNAMIC_DEBUG=y
# أي pr_debug() في الكود بيبقى متاح تفعيله runtime
echo "func clk_mux_set_parent +p" > /sys/kernel/debug/dynamic_debug/control
echo "func clk_mux_get_parent +p" > /sys/kernel/debug/dynamic_debug/control
```

---

### 5. خيارات الـ Kernel Config للـ Debugging

| Config Option | الغرض | ملاحظة |
|---|---|---|
| `CONFIG_COMMON_CLK` | يفعّل الـ common clock framework | أساسي |
| `CONFIG_CLK_DEBUG` | يفعّل debug entries في `/sys/kernel/debug/clk/` | ضروري للتشخيص |
| `CONFIG_DEBUG_SPINLOCK` | يكتشف spinlock misuse (مهم لـ `mux->lock`) | يبطّء النظام |
| `CONFIG_PROVE_LOCKING` | `lockdep` — يكتشف deadlocks و lock order violations | مهم جداً |
| `CONFIG_LOCK_STAT` | إحصائيات الـ lock contention | لـ performance |
| `CONFIG_DEBUG_ATOMIC_SLEEP` | يكتشف sleep داخل atomic context (spinlock) | مهم |
| `CONFIG_KALLSYMS` | يظهر أسماء الـ functions في stack traces | أساسي للـ debug |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل `pr_debug()` runtime | موصى به |
| `CONFIG_CLK_UNIPHIER` أو غيره | Platform-specific clock debugging | حسب الـ SoC |
| `CONFIG_DEBUG_FS` | يفعّل `debugfs` | ضروري لـ `/sys/kernel/debug/clk/` |
| `CONFIG_KASAN` | يكتشف memory corruption (مهم لو `mux->table` غلط) | overhead كبير |

```bash
# تحقق من الـ config الحالي
grep -E "CONFIG_(COMMON_CLK|CLK_DEBUG|DEBUG_SPINLOCK|PROVE_LOCKING|DYNAMIC_DEBUG|DEBUG_FS)" /boot/config-$(uname -r)
```

---

### 6. أدوات الـ `devlink` والأدوات الخاصة بالـ Subsystem

**الـ `clk-mux` مش بيستخدم devlink** — ده خاص بالـ network devices. بدله:

```bash
# clk_summary — أهم أداة
cat /sys/kernel/debug/clk/clk_summary

# اقرأ parent مباشرة
cat /sys/kernel/debug/clk/<mux_name>/clk_parent

# تحقق من accuracy
cat /sys/kernel/debug/clk/<mux_name>/clk_accuracy

# اعرض flags الخاصة بالـ clock
cat /sys/kernel/debug/clk/<mux_name>/clk_flags

# أداة userspace للتعامل مع clocks (لو متوفرة)
# في بعض الـ distros
clk-tool list
clk-tool get-parent uart_clk_mux

# من الـ kernel source: tools/clk/
# بيوفر clk-tool لقراءة debugfs بشكل structured
```

---

### 7. جدول رسائل الخطأ الشائعة

| رسالة الخطأ | المعنى | الحل |
|---|---|---|
| `mux value exceeds LOWORD field` | الـ `shift + width > 16` مع `CLK_MUX_HIWORD_MASK` | صحّح الـ `shift` أو الـ `mask` في الـ DT أو الكود |
| `clk: couldn't get parent` | `clk_mux_get_parent` رجّع `-EINVAL` — القيمة في الـ register مش موجودة في الـ table | تحقق من register value مع `devmem2`، وتحقق من الـ `table` |
| `-ENOMEM` من `__clk_hw_register_mux` | الـ `kzalloc` فشل | مشكلة memory — راجع `dmesg` للـ OOM messages |
| `clk: failed to reparent` | `clk_set_parent` رجّع error | تحقق من الـ `CLK_SET_PARENT_*` flags والـ notifiers |
| `WARNING: ... clk_prepare_lock` | حاول يعمل `clk_set_parent` من interrupt context | الكود بيستخدم الـ API من السياق الغلط |
| `clk: parent rate ... doesn't match child rate` | بعد الـ reparent، الـ rate مش متوافق | الـ child clock بيحتاج `CLK_SET_RATE_PARENT` |
| `of_clk_get_parent_name: couldn't find parent` | اسم الـ parent في الـ DT مش موجود في الـ clock tree | تحقق من `clock-names` و `clocks` في الـ DT |
| `clk: clk_mux_val_to_index: -EINVAL` | قيمة الـ register مش موجودة في الـ table | الـ hardware في حالة غير متوقعة — reset الـ clock register |

---

### 8. Strategic points لـ `dump_stack()` و `WARN_ON()`

**الأماكن الاستراتيجية للإضافة مؤقتاً:**

```c
static u8 clk_mux_get_parent(struct clk_hw *hw)
{
    struct clk_mux *mux = to_clk_mux(hw);
    u32 val;
    int index;

    val = clk_mux_readl(mux) >> mux->shift;
    val &= mux->mask;

    /* STRATEGIC: اكتشف لو القيمة المقروءة خارج النطاق */
    WARN_ON(val > 0xFF);  /* mask يجب أن يمنع ده، لكن تحقق */

    index = clk_mux_val_to_index(hw, mux->table, mux->flags, val);

    /* STRATEGIC: اكتشف invalid index */
    if (WARN_ON(index < 0)) {
        pr_err("clk_mux: %s: invalid hw val 0x%x from reg 0x%x (mask=0x%x shift=%d)\n",
               clk_hw_get_name(hw), val,
               (u32)readl(mux->reg), mux->mask, mux->shift);
        dump_stack();
    }

    return index;
}

static int clk_mux_set_parent(struct clk_hw *hw, u8 index)
{
    struct clk_mux *mux = to_clk_mux(hw);
    u32 val, reg_before, reg_after;

    /* STRATEGIC: تحقق من الـ index قبل الكتابة */
    WARN_ON(index >= clk_hw_get_num_parents(hw));

    val = clk_mux_index_to_val(mux->table, mux->flags, index);

    /* STRATEGIC: سجّل قيمة الـ register قبل وبعد */
    reg_before = clk_mux_readl(mux);

    /* ... الكود الأصلي ... */

    reg_after = clk_mux_readl(mux);
    pr_debug("clk_mux %s: reg 0x%x -> 0x%x (index=%d val=0x%x)\n",
             clk_hw_get_name(hw), reg_before, reg_after, index, val);

    /* STRATEGIC: تحقق إن الكتابة نجحت */
    WARN_ON(((reg_after >> mux->shift) & mux->mask) != val);

    return 0;
}
```

---

## Hardware Level

### 1. التحقق من حالة الـ Hardware مقابل الـ Kernel

```bash
# 1. اقرأ الـ parent الذي يعتقده الـ kernel
PARENT=$(cat /sys/kernel/debug/clk/uart_clk_mux/clk_parent)
echo "Kernel thinks parent is: $PARENT"

# 2. اقرأ الـ register مباشرة من الـ hardware
# افترض أن عنوان الـ register هو 0x01C20000 (مثال على Allwinner)
devmem2 0x01C20000 w

# 3. قارن: الـ bits المقروءة من الـ register بعد shift ومع الـ mask
# لو الـ kernel قال parent index=1 ولكن الـ register قال 0x0 — في مشكلة

# 4. تحقق من الـ rate الفعلي عبر oscilloscope أو frequency counter
# ثم قارنه مع ما يقوله الـ kernel
cat /sys/kernel/debug/clk/uart_clk_mux/clk_rate
```

**رسم توضيحي لعلاقة الـ register بالـ kernel:**

```
Register (32-bit):
  Bit 31..16: HIWORD mask (لو CLK_MUX_HIWORD_MASK)
  Bit 15..0 : LOWORD data

  مثال: shift=4, mask=0x3 (2 bits)
  Bits [5:4] = parent index

  +--+--+--+--+--+--+--+--+--+--+--+--+
  |  |  |  |  |  |  |  |  |P1|P0|  |  |
  +--+--+--+--+--+--+--+--+--+--+--+--+
  31                       5  4       0
                           ^  ^
                        shift + mask bits
```

---

### 2. تقنيات الـ Register Dump

```bash
# devmem2: الأداة الأشهر لقراءة/كتابة registers
# تثبيت: apt-get install devmem2 أو build من المصدر
devmem2 <physical_address> w          # قراءة 32-bit word
devmem2 <physical_address> w <value>  # كتابة

# مثال: قراءة CCU register على Allwinner H3
devmem2 0x01C20058 w
# النتيجة: Value at address 0x01C20058 (0xf0e50058): 0x00010001

# /dev/mem مع dd (يحتاج CONFIG_DEVMEM=y)
dd if=/dev/mem bs=4 count=1 skip=$((0x01C20058/4)) 2>/dev/null | xxd

# io utility (من package 'io' أو 'hwtools')
io -4 0x01C20058

# mmap من Python للقراءة
python3 -c "
import mmap, os
fd = os.open('/dev/mem', os.O_RDONLY | os.O_SYNC)
m = mmap.mmap(fd, 4096, offset=0x01C20000)
import struct
val = struct.unpack('<I', m[0x58:0x5C])[0]
print(f'Register: 0x{val:08X}')
m.close()
os.close(fd)
"
```

**تحليل القيمة يدوياً:**

```bash
# مثال: قرأنا 0x00010001، الـ mux shift=0, mask=0x3
VALUE=0x00010001
SHIFT=0
MASK=0x3

python3 -c "
v = $VALUE
shift = $SHIFT
mask = $MASK
mux_val = (v >> shift) & mask
print(f'MUX select bits = {mux_val} (0x{mux_val:x})')
"
```

---

### 3. Logic Analyzer / Oscilloscope

**حالات الاستخدام:**

| الهدف | الأداة | الطريقة |
|---|---|---|
| تحقق من تغيير الـ clock source | Oscilloscope | probing على الـ clock output line، راقب الـ frequency jump |
| رصد الـ glitch عند الـ switching | Logic Analyzer | trigger on clock edge change |
| تحقق من I2C/SPI CCU | Logic Analyzer | capture الـ register write transaction |
| قياس الـ settling time | Oscilloscope | من write للـ register لحد ما الـ clock يستقر |

**نصائح عملية:**

```
1. probe على output pin الخاص بالـ clock (مش على الـ register bus)
2. استخدم trigger على "frequency change" في الـ oscilloscope
3. لو الـ clock بيعمل glitch عند الـ switching → يحتاج CLK_MUX_ROUND_CLOSEST
   أو الـ driver محتاج يـ gate الـ clock قبل الـ switch
4. قيس الـ overshoot/undershoot بعد الـ switch — لو الـ downstream logic
   بتتأثر، هيظهر في الـ kernel كـ data corruption
```

---

### 4. مشاكل الـ Hardware الشائعة وأنماطها في الـ Kernel Log

| مشكلة Hardware | Pattern في `dmesg` | الحل |
|---|---|---|
| الـ register مش writable (read-only في الـ SoC) | `clk_mux_set_parent` silent fail + parent لا يتغير | استخدم `CLK_MUX_READ_ONLY` flag |
| Power domain للـ CCU مش enabled | `clk: couldn't get parent` أو قيمة 0xFFFFFFFF من الـ register | فعّل الـ power domain قبل register mux |
| Clock glitch يخلي الـ CPU يعمل reset | `Rebooting in X seconds..` بعد مباشرة من `clk_set_parent` | أضف `CLK_SET_PARENT_GATE` أو اعمل gate قبل switch |
| Big-endian register access على little-endian SoC | قيم غريبة في `clk_summary` (rate = 0 أو garbage) | أضف `CLK_MUX_BIG_ENDIAN` flag |
| HIWORD_MASK غلط | الـ clock switch بيكتب فوق registers تانية | تحقق من `shift + width <= 16` |
| Spinlock ومشكلة الـ IRQ | `BUG: scheduling while atomic` | راجع الـ lock context، استخدم `spin_lock_irqsave` |

---

### 5. تشخيص الـ Device Tree

```bash
# اعرض الـ DT compiled للـ clock mux node
dtc -I fs -O dts /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "mux"

# أو باستخدام fdtdump
fdtdump /boot/dtb-$(uname -r) | grep -A 15 "clk-mux"

# تحقق من clock-names وclocks في الـ consumer device
cat /sys/firmware/devicetree/base/soc/uart@1C28000/clock-names
cat /sys/firmware/devicetree/base/soc/uart@1C28000/clocks

# اعرض كل الـ assigned-clocks في الـ DT
dtc -I fs -O dts /sys/firmware/devicetree/base 2>/dev/null | grep -B5 -A10 "assigned-clocks"
```

**مثال على DT صحيح لـ clock mux:**

```dts
ccu: clock-controller@1c20000 {
    compatible = "allwinner,sun8i-h3-ccu";
    reg = <0x01c20000 0x400>;
    clocks = <&osc24M>, <&pll_periph0>, <&pll_video>;
    clock-names = "hosc", "pll-periph0", "pll-video";
    #clock-cells = <1>;
};

uart0: serial@1c28000 {
    clocks = <&ccu CLK_BUS_UART0>;
    /* المفروض يستخدم clock index صح */
};
```

**أخطاء DT شائعة:**

```bash
# لو الـ clock mux مش لاقي parents
dmesg | grep "could not get clock"
dmesg | grep "clk.*not found"
dmesg | grep "of_clk_get"

# تحقق من عدد الـ parents في الـ DT مقابل الكود
# لو DT قال num_parents=3 والكود عامل table بـ 4 entries → EINVAL
```

---

## Practical Commands

### مجموعة أوامر جاهزة للنسخ

#### الـ Setup الأساسي

```bash
#!/bin/bash
# clk_mux_debug_setup.sh
# ضع اسم الـ mux clock هنا
MUX_NAME="uart_clk_mux"
CLK_DEBUG="/sys/kernel/debug/clk"
TRACE="/sys/kernel/debug/tracing"

echo "=== Clock MUX Debug Setup ==="
echo "Target: $MUX_NAME"

# 1. تحقق من وجود debugfs
mountpoint -q /sys/kernel/debug || mount -t debugfs debugfs /sys/kernel/debug

# 2. اعرض معلومات الـ mux
echo ""
echo "--- Clock Info ---"
echo "Parent  : $(cat $CLK_DEBUG/$MUX_NAME/clk_parent 2>/dev/null || echo 'N/A')"
echo "Rate    : $(cat $CLK_DEBUG/$MUX_NAME/clk_rate 2>/dev/null || echo 'N/A')"
echo "Flags   : $(cat $CLK_DEBUG/$MUX_NAME/clk_flags 2>/dev/null || echo 'N/A')"
echo "En count: $(cat $CLK_DEBUG/$MUX_NAME/clk_enable_count 2>/dev/null || echo 'N/A')"

# 3. فعّل ftrace للـ set_parent
echo "" > $TRACE/trace
echo 1 > $TRACE/events/clk/clk_set_parent/enable
echo 1 > $TRACE/events/clk/clk_set_parent_complete/enable
echo 1 > $TRACE/tracing_on

echo ""
echo "--- ftrace enabled. Run your test, then: ---"
echo "cat $TRACE/trace"
```

#### قراءة وتحليل الـ Register

```bash
#!/bin/bash
# read_mux_register.sh
# غيّر هذه المتغيرات حسب الـ SoC
REG_ADDR=0x01C20058   # عنوان الـ register
MUX_SHIFT=0           # shift للـ mux bits
MUX_MASK=0x3          # mask للـ mux bits

echo "Reading register at 0x$(printf '%08X' $REG_ADDR)"
RAW=$(devmem2 $REG_ADDR w 2>/dev/null | grep "Value" | awk '{print $NF}')
echo "Raw value: $RAW"

python3 -c "
raw = int('$RAW', 16)
shift = $MUX_SHIFT
mask = $MUX_MASK
mux_sel = (raw >> shift) & mask
print(f'MUX select = {mux_sel} (binary: {mux_sel:04b})')
print(f'Full reg  = 0x{raw:08X}')
print(f'Bits [{shift + bin(mask).count(\"1\") - 1}:{shift}] = 0x{mux_sel:x}')
"
```

#### مقارنة الـ Kernel State مع الـ Hardware

```bash
#!/bin/bash
# compare_kernel_vs_hw.sh
MUX_NAME="${1:-uart_clk_mux}"
REG_ADDR="${2:-0x01C20058}"
SHIFT="${3:-0}"
MASK="${4:-0x3}"

echo "=== Kernel vs Hardware Comparison ==="
echo "Clock: $MUX_NAME"

KERNEL_PARENT=$(cat /sys/kernel/debug/clk/$MUX_NAME/clk_parent 2>/dev/null)
KERNEL_RATE=$(cat /sys/kernel/debug/clk/$MUX_NAME/clk_rate 2>/dev/null)
HW_VAL=$(devmem2 $REG_ADDR w 2>/dev/null | grep "Value" | awk '{print $NF}')

echo "Kernel parent : $KERNEL_PARENT"
echo "Kernel rate   : $KERNEL_RATE Hz"
echo "HW register   : $HW_VAL"

python3 -c "
hw = int('$HW_VAL', 16)
shift = $SHIFT
mask = $MASK
sel = (hw >> shift) & mask
print(f'HW MUX select bits: {sel} → maps to parent index {sel}')
"
```

#### جمع الـ Clock Dump الكامل

```bash
#!/bin/bash
# full_clk_dump.sh
OUTPUT_FILE="/tmp/clk_debug_$(date +%Y%m%d_%H%M%S).txt"

{
    echo "=== Clock Summary ==="
    cat /sys/kernel/debug/clk/clk_summary

    echo ""
    echo "=== Clock Dump (JSON-like) ==="
    cat /sys/kernel/debug/clk/clk_dump

    echo ""
    echo "=== dmesg clock errors ==="
    dmesg | grep -i "clk\|clock" | tail -50

    echo ""
    echo "=== Kernel boot args ==="
    cat /proc/cmdline

    echo ""
    echo "=== uname ==="
    uname -a

} > "$OUTPUT_FILE" 2>&1

echo "Saved to: $OUTPUT_FILE"
cat "$OUTPUT_FILE"
```

---

### مثال على Output وتفسيره

#### `clk_summary` output:

```
                                 enable  prepare  protect                      rate
   clock                          count    count    count        rate   accuracy
----------------------------------------------------------------------------------
 osc24M                               2        2        0    24000000          0
    pll_cpu                           1        1        0   816000000          0
    pll_periph0                       3        3        0   600000000          0
       uart_clk_mux                   1        1        0   600000000          0
          uart0_div                   1        1        0    50000000          0
```

**التفسير:**
- `uart_clk_mux` اختار `pll_periph0` كـ parent (موضّح بالـ indentation)
- `enable_count=1` يعني consumer واحد شغّله
- `rate=600000000` = 600 MHz — ده rate الـ parent مش rate الـ mux نفسه (مفيش division في الـ mux)

#### لو الـ parent غلط:

```
   uart_clk_mux                   1        1        0           0          0
```

- `rate=0` → الـ parent مش موجود أو disabled
- في الـ `dmesg` هتلاقي: `clk uart_clk_mux: could not get rate of parent`

#### ftrace output تفسيره:

```
<...>-1234  [001] ....  456.789: clk_set_parent: uart_clk_mux
<...>-1234  [001] ....  456.790: clk_set_parent_complete: uart_clk_mux
```

- الـ event جه وراح → ناجح
- الفرق الزمني صغير جداً (0.001ms) → مفيش blocking

```
<...>-1234  [001] ....  456.789: clk_set_parent: uart_clk_mux
# لا يوجد clk_set_parent_complete → مشكلة!
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: UART مش شغال على RK3562 — Industrial Gateway

#### العنوان
الـ UART صامت تماماً بعد boot على gateway صناعي مبني على RK3562

#### السياق
فريق bring-up شغال على gateway صناعي بيستخدم RK3562. الـ UART2 المربوط بـ RS-485 transceiver مش بيطلع أي بيانات. الـ console شغال على UART0 بس الـ UART2 الخاص بالـ application level صامت.

#### المشكلة
الـ UART2 محتاج clock من `uart2_src` اللي هو mux clock بيختار بين `gpll` (1.2 GHz) و`cpll` (1.0 GHz). المطور سجّل الـ mux بـ `shift=8, mask=0x3` في الـ CRU (Clock Reset Unit)، بس غلط في قيمة الـ `table` — حط `table = {0, 1}` بدل ما يحط `table = {2, 0}` اللي الـ datasheet بتحدده للـ RK3562.

#### التحليل
لما الـ kernel يعمل `clk_set_parent(uart2_src, gpll)`:

```c
/* في clk_mux_set_parent: */
static int clk_mux_set_parent(struct clk_hw *hw, u8 index)
{
    struct clk_mux *mux = to_clk_mux(hw);
    /* index=0 → gpll */
    u32 val = clk_mux_index_to_val(mux->table, mux->flags, index);
    /* لو table = {0,1}: val = table[0] = 0 */
    /* المفروض يكون 2 حسب الـ datasheet */
    ...
    reg &= ~(mux->mask << mux->shift);  /* clear bits [9:8] */
    val = val << mux->shift;            /* val=0 << 8 = 0 */
    reg |= val;
    clk_mux_writel(mux, reg);           /* يكتب 0 في [9:8] */
}
```

الـ hardware فسّر القيمة 0 في bits [9:8] إنها "no clock source" وقفل الـ UART2 clock.

```c
/* في clk_mux_get_parent: */
static u8 clk_mux_get_parent(struct clk_hw *hw)
{
    val = clk_mux_readl(mux) >> mux->shift;
    val &= mux->mask;  /* val=0 */
    return clk_mux_val_to_index(hw, mux->table, mux->flags, val);
    /* بيدور على 0 في table={0,1} → يرجع index=0 (gpll) */
    /* رغم إن الـ hardware فعلياً مش بيوصله clock صح */
}
```

الـ kernel اعتقد إن كل حاجة تمام لأن `get_parent` رجّع نتيجة valid، بس فعلياً الـ register value غلط.

#### الحل
**1. تصحيح الـ table في driver الـ RK3562 CRU:**

```c
/* drivers/clk/rockchip/clk-rk3562.c */
static const u32 uart2_src_table[] = { 2, 0 };
/* 2 = gpll في الـ CRU register، 0 = cpll */

COMPOSITE_MUX_TABLE(uart2_src,
    "uart2_src", uart2_src_parents, 0,
    RK3562_CRU_CLKSEL_CON(28), 8, 3,
    CLK_MUX_HIWORD_MASK, uart2_src_table);
```

**2. التحقق عبر debugfs:**

```bash
# قرا القيمة الفعلية للـ register
devmem2 0xFF0A0070 w   # CRU_CLKSEL_CON28 على RK3562

# شوف الـ parent الحالي
cat /sys/kernel/debug/clk/uart2_src/clk_parent

# تحقق من الـ clock rate
cat /sys/kernel/debug/clk/uart2_src/clk_rate
```

#### الدرس المستفاد
الـ `table` في `struct clk_mux` هو mapping صريح بين **index الـ software** (ترتيب الـ parents في الـ DT) و**قيمة الـ hardware register**. لما الـ datasheet يقول إن gpll = 0b10، لازم الـ table[0] = 2. غلطة شائعة إن المطور يفترض إن الـ register values متتالية من 0.

---

### السيناريو الثاني: HDMI Output متقطع على Allwinner H616 — Android TV Box

#### العنوان
الـ HDMI بيقطع كل دقيقتين على TV box مبني على Allwinner H616

#### السياق
منتج TV box بيستخدم Allwinner H616. الـ HDMI output بيتوصل ويتفصل بشكل عشوائي، logcat بيظهر `HDMI HPD lost` رغم إن الكابل سليم.

#### المشكلة
الـ TCON (Timing Controller) clock مبني على mux بيختار بين `pll-video0` و`pll-video1`. المطور ماسجلش الـ `spinlock_t` بشكل صح — بعت `lock = NULL` للـ `__clk_hw_register_mux`. في نفس الوقت الـ DVFS (Dynamic Voltage Frequency Scaling) على CPU كان بيغير الـ PLL settings من interrupt context.

#### التحليل
```c
/* في clk_mux_set_parent — بدون lock */
static int clk_mux_set_parent(struct clk_hw *hw, u8 index)
{
    ...
    unsigned long flags = 0;

    if (mux->lock)                    /* NULL → skip */
        spin_lock_irqsave(mux->lock, flags);
    else
        __acquire(mux->lock);         /* مجرد annotation، مش lock حقيقي */

    /* هنا race condition ممكن:
     * thread1: reg = clk_mux_readl(mux)     → reg = 0x00000001
     * IRQ:     يغير parent آخر              → register = 0x00000000
     * thread1: reg &= ~(mask << shift)      → reg = 0x00000000
     * thread1: reg |= val << shift          → reg = 0x00000002
     * thread1: clk_mux_writel(mux, reg)     → كتب فوق تغيير الـ IRQ
     */
    reg = clk_mux_readl(mux);
    reg &= ~(mux->mask << mux->shift);
    val = val << mux->shift;
    reg |= val;
    clk_mux_writel(mux, reg);

    if (mux->lock)
        spin_unlock_irqrestore(mux->lock, flags);
    else
        __release(mux->lock);

    return 0;
}
```

الـ TCON clock اتكتبت عليها قيمة غلط، الـ pixel clock اتغير، الشاشة فقدت الـ sync وطلع HPD lost.

#### الحل
**1. تعريف spinlock مشترك مع باقي clocks في نفس الـ register block:**

```c
/* drivers/clk/sunxi-ng/ccu-h616.c */
static DEFINE_SPINLOCK(h616_ccu_lock);

/* عند تسجيل الـ mux */
hw = clk_hw_register_mux_table(dev, "tcon-mux",
    tcon_parents, ARRAY_SIZE(tcon_parents),
    0,
    base + H616_TCON_CLK_REG,
    24, 0x3,        /* shift=24, mask=0x3 */
    0,              /* no special flags */
    NULL,           /* no table needed, sequential */
    &h616_ccu_lock  /* الـ lock المشترك */
);
```

**2. الـ Allwinner H616 عنده HIWORD_MASK support، ممكن كمان نستخدمه:**

```c
/* لو الـ register layout يدعمه */
hw = clk_hw_register_mux_table(dev, "tcon-mux",
    tcon_parents, ARRAY_SIZE(tcon_parents),
    0,
    base + H616_TCON_CLK_REG,
    24, 0x3,
    CLK_MUX_HIWORD_MASK,  /* upper 16 bits = write mask, atomic */
    NULL,
    NULL  /* HIWORD_MASK → no need for lock */
);
```

مع `CLK_MUX_HIWORD_MASK` الكود بيتصرف كده:
```c
if (mux->flags & CLK_MUX_HIWORD_MASK) {
    /* write mask في upper 16 bits + value في lower 16 bits */
    /* operation أتومية، مش محتاج lock */
    reg = mux->mask << (mux->shift + 16);
    /* مش بيعمل read-modify-write */
}
```

#### الدرس المستفاد
الـ `spinlock_t *lock` في `struct clk_mux` مش optional لو الـ register مشترك مع clocks تانية أو لو في إمكانية تعديله من interrupt context. الـ `CLK_MUX_HIWORD_MASK` هو الحل الأفضل لو الـ SoC يدعمه لأنه بيلغي الحاجة للـ read-modify-write.

---

### السيناريو الثالث: SPI Flash مش بيتعرف عليه على STM32MP1 — IoT Sensor

#### العنوان
الـ SPI3 على STM32MP1 بيطلع بـ rate غلط وبيفشل قراءة الـ Flash

#### السياق
board IoT sensor بيستخدم STM32MP1 مع SPI NOR Flash مربوط على SPI3. الـ flash بيشتغل بـ max 50 MHz. الـ probe بيفشل بـ `spi_nor: unrecognized JEDEC id`.

#### المشكلة
الـ SPI3 kernel clock (`spi3_k`) هو mux من 6 parents. المطور عند كتابة الـ DT وضع الـ parents بترتيب غلط — عكس اللي في الـ kernel driver. الـ `clk_mux_val_to_index` بيرجع index خاطئ، فالـ kernel فاكر إن الـ parent هو `ck_hse` (25 MHz) بس الـ hardware فعلياً بيشتغل من `pll4_p` (600 MHz ÷ divider = 200 MHz).

#### التحليل
```c
/* في clk_mux_get_parent: */
static u8 clk_mux_get_parent(struct clk_hw *hw)
{
    struct clk_mux *mux = to_clk_mux(hw);
    u32 val;

    val = clk_mux_readl(mux) >> mux->shift;
    val &= mux->mask;
    /* مثلاً: val = 3 (الـ hardware بيشير لـ pll4_p) */

    return clk_mux_val_to_index(hw, mux->table, mux->flags, val);
    /* table = NULL, flags = 0 → val as index directly */
    /* يرجع index=3 */
    /* بس في DT: index 3 = ck_hse بسبب ترتيب غلط */
}
```

الـ `clk_mux_val_to_index` مع `table=NULL` ومفيش flags خاصة بيعمل كده:
```c
int clk_mux_val_to_index(struct clk_hw *hw, const u32 *table,
                          unsigned int flags, unsigned int val)
{
    /* table=NULL → skip table lookup */
    /* CLK_MUX_INDEX_BIT → مش set */
    /* CLK_MUX_INDEX_ONE → مش set */

    if (val >= num_parents)  /* 3 < 6 → ok */
        return -EINVAL;

    return val;  /* يرجع 3 كـ index */
    /* الـ parent عند index 3 في DT هو ck_hse بسبب الترتيب الغلط */
}
```

الـ kernel حسب rate الـ SPI3 بناءً على `ck_hse` (25 MHz)، ضبط divider بناءً على ده، والـ hardware الحقيقي كان بيشتغل بـ 200 MHz — أسرع بكتير من اللي الـ Flash بيتحمله.

#### الحل
**1. تصحيح ترتيب الـ parents في DT:**

```dts
/* arch/arm/boot/dts/stm32mp151.dtsi */
spi3_src: spi3-src-clock {
    #clock-cells = <0>;
    compatible = "fixed-clock";
};

/* في الـ CRU node */
&rcc {
    /* ترتيب الـ parents لازم يطابق تماماً الـ register bits */
    /* Bit[26:24] في RCC_SPI3CKSELR:
     * 000 → pll4_p  (index 0)
     * 001 → pll3_q  (index 1)
     * 010 → i2s_ckin (index 2)
     * 011 → ck_per  (index 3)
     * 100 → ck_lse  (index 4 — غلط كان index 3 ck_hse)
     * 101 → ck_hse  (index 5)
     */
    spi3_k: spi3_k {
        clocks = <&rcc PLL4_P>, <&rcc PLL3_Q>,
                 <&rcc I2S_CKIN>, <&rcc CK_PER>,
                 <&rcc CK_LSE>, <&rcc CK_HSE>;
    };
};
```

**2. التحقق السريع:**

```bash
# على الـ target
cat /sys/kernel/debug/clk/spi3_k/clk_parent
# المفروض يطلع pll4_p مش ck_hse

cat /sys/kernel/debug/clk/spi3_k/clk_rate
# المفروض يطلع القيمة الصح

# أو قرا الـ register مباشرة
devmem2 0x50000C00 w  # RCC_SPI3CKSELR address
```

#### الدرس المستفاد
لما مفيش `table`، الـ `clk_mux_val_to_index` بيفترض إن **قيمة الـ register = index الـ parent في الـ parent_names array**. أي خلاف بين ترتيب الـ DT parents وترتيب الـ hardware register values لازم يتحل إما بـ `table` صريح أو بضبط الترتيب بدقة.

---

### السيناريو الرابع: I2C Hang على i.MX8MM — Automotive ECU

#### العنوان
الـ I2C3 بيتعلق (hang) بعد suspend/resume على ECU مبني على i.MX8MM

#### السياق
ECU automotive بيستخدم i.MX8MM. بعد كل دورة suspend/resume، الـ I2C3 بيتعلق. الـ sensor المربوط (accelerometer ADXL345) مبيرديش. المشكلة reproducible 100%.

#### المشكلة
الـ I2C3 clock source هو `i2c3_root_clk` — mux من 4 parents. الـ driver استخدم `CLK_MUX_READ_ONLY` flag. بعد الـ resume، الـ CCF (Common Clock Framework) بيحاول يعمل `set_parent` لإعادة الـ state، بس بما إن الـ ops هي `clk_mux_ro_ops` فمفيش `set_parent` callback.

#### التحليل
```c
/* عند التسجيل بـ CLK_MUX_READ_ONLY: */
if (clk_mux_flags & CLK_MUX_READ_ONLY)
    init.ops = &clk_mux_ro_ops;  /* بس get_parent، مفيش set_parent */
else
    init.ops = &clk_mux_ops;

/* clk_mux_ro_ops: */
const struct clk_ops clk_mux_ro_ops = {
    .get_parent = clk_mux_get_parent,
    /* set_parent = NULL */
    /* determine_rate = NULL */
};
```

لما الـ CCF حاول يعمل `clk_set_parent` بعد الـ resume:
- الـ `set_parent` = NULL
- الـ CCF رجّع error
- الـ I2C driver معملش proper error handling لفشل الـ clock
- الـ I2C controller اشتغل بدون clock أو بـ clock غلط → hang

المشكلة الأعمق: المطور استخدم `CLK_MUX_READ_ONLY` لأن الـ bootloader ضبط الـ mux وماراش يغيره — بس الـ CCF محتاج `set_parent` حتى لو بس لإعادة نفس الـ parent بعد resume.

#### الحل
**الخيار 1: مستش `CLK_MUX_READ_ONLY` وبدله اضبط الـ parent ثابت في DT:**

```dts
/* في DT الـ i.MX8MM */
i2c3_root_clk: i2c3-root {
    assigned-clocks = <&clk IMX8MM_CLK_I2C3_ROOT>;
    assigned-clock-parents = <&clk IMX8MM_SYS_PLL1_160M>;
    /* ده بيضمن إن الـ CCF يضبط الـ parent صح من الأول */
};
```

**الخيار 2: implement save/restore في الـ mux driver لو محتاج read-only فعلاً:**

```c
/* في الـ SoC-specific driver */
static int imx8mm_i2c_mux_save_context(struct clk_hw *hw)
{
    struct clk_mux *mux = to_clk_mux(hw);
    /* save current parent before suspend */
    mux_saved_parent = clk_mux_get_parent(hw);
    return 0;
}

/* أو ببساطة: مستخدمش CLK_MUX_READ_ONLY */
hw = clk_hw_register_mux(dev, "i2c3_root",
    parents, num_parents,
    CLK_SET_RATE_NO_REPARENT,  /* منعش الـ auto-reparent */
    reg, shift, mask,
    0,      /* مش CLK_MUX_READ_ONLY */
    lock);
```

**التحقق:**

```bash
# بعد resume
cat /sys/kernel/debug/clk/i2c3_root/clk_parent

# تحقق من pm transitions
dmesg | grep -E "i2c|clk" | grep -i "resume\|suspend"
```

#### الدرس المستفاد
الـ `CLK_MUX_READ_ONLY` / `clk_mux_ro_ops` بتمنع الـ kernel من تغيير الـ parent، بس الـ CCF لو حاول يعمل `set_parent` بعد resume هيفشل بصمت. في سياق automotive اللي suspend/resume critical فيه، يفضل استخدام `clk_mux_ops` الكاملة مع `assigned-clock-parents` في DT بدل الـ read-only approach.

---

### السيناريو الخامس: USB مش بيشتغل على AM62x — Custom Board Bring-up

#### العنوان
الـ USB2.0 PHY مش بيعمل `enumerate` على custom board مبني على TI AM62x

#### السياق
مرحلة bring-up لـ custom board صناعي بيستخدم TI AM62x (Sitara). الـ USB2.0 port مش بيشوف أي device. `lsusb` فارغ، الـ PHY مرجعتش أي interrupt.

#### المشكلة
الـ USB PHY محتاج `CLK_USB_REF` من `usb_ref_clk` — وهو mux بيختار بين `hfosc0` (25 MHz) و`main_pll2_hsdiv4` (25 MHz بعد الـ division). الـ board design استخدم external oscillator على pin HFOSC1 مش HFOSC0، بس الـ DT ماتغيرش من الـ reference design.

الـ `CLK_MUX_INDEX_ONE` flag متسطبش رغم إن الـ AM62x TRM بيقول إن الـ USB ref mux register يبدأ من 1:

| Register Value | Source         |
|---------------|----------------|
| 0x0           | Reserved       |
| 0x1           | hfosc0         |
| 0x2           | hfosc1         |
| 0x3           | main_pll2_hsdiv4 |

#### التحليل
```c
/* المطور سجّل الـ mux كده: */
hw = clk_hw_register_mux(dev, "usb_ref_clk",
    usb_ref_parents,    /* ["hfosc0", "hfosc1", "main_pll2_hsdiv4"] */
    3,
    0,
    reg,
    shift, 0x3,         /* mask=0x3 */
    0,                  /* flags=0, CLK_MUX_INDEX_ONE مش set */
    lock);

/* لما الـ DT يطلب hfosc0 (index=0): */
/* clk_mux_index_to_val: */
unsigned int clk_mux_index_to_val(const u32 *table, unsigned int flags, u8 index)
{
    unsigned int val = index;  /* val=0 */

    if (table) { ... }  /* table=NULL */
    else {
        if (flags & CLK_MUX_INDEX_BIT) { ... }  /* مش set */
        if (flags & CLK_MUX_INDEX_ONE) val++;    /* مش set! */
    }
    return val;  /* يرجع 0 */
}
/* يكتب 0x0 في الـ register → Reserved → PHY مش بياخد clock */
```

والأسوأ إن `clk_mux_get_parent` بيرجع:
```c
val = readl(reg) >> shift;
val &= 0x3;  /* val=0 */
/* clk_mux_val_to_index: */
/* table=NULL, CLK_MUX_INDEX_ONE مش set → يرجع val=0 مباشرة */
/* index=0 → hfosc0 ✓ (correct!) */
/* kernel فاكر كل حاجة تمام لأن get_parent صح */
/* بس الـ hardware بيشتغل على "Reserved" */
```

#### الحل
**1. استخدام `CLK_MUX_INDEX_ONE`:**

```c
/* drivers/clk/ti/clk-am62x.c */
hw = clk_hw_register_mux(dev, "usb_ref_clk",
    usb_ref_parents,
    3,
    0,
    reg,
    shift, 0x3,
    CLK_MUX_INDEX_ONE,  /* register values تبدأ من 1 مش 0 */
    lock);

/* دلوقتي: index=0 (hfosc0) → val = 0 + 1 = 1 → register = 0x1 ✓ */
```

**2. أو استخدام `table` explicit — أوضح وأأمن:**

```c
static const u32 am62x_usb_ref_table[] = { 1, 2, 3 };
/* index 0 → hfosc0 → reg value 1
 * index 1 → hfosc1 → reg value 2
 * index 2 → main_pll2 → reg value 3 */

hw = clk_hw_register_mux_table(dev, "usb_ref_clk",
    usb_ref_parents, 3, 0,
    reg, shift, 0x3,
    0,                    /* flags */
    am62x_usb_ref_table,  /* explicit mapping */
    lock);
```

**3. الـ DT لتحديد الـ parent الصح (hfosc1 مش hfosc0):**

```dts
/* k3-am625-custom-board.dts */
&usbss0 {
    assigned-clocks = <&k3_clks 161 0>;   /* usb_ref_clk */
    assigned-clock-parents = <&k3_clks 161 2>; /* hfosc1 = index 1 */
};
```

**التحقق على الـ target:**

```bash
# تحقق من الـ parent
cat /sys/kernel/debug/clk/usb_ref_clk/clk_parent
# المفروض يطلع hfosc1

# تحقق من الـ rate
cat /sys/kernel/debug/clk/usb_ref_clk/clk_rate
# المفروض 25000000

# قرا الـ register مباشرة
devmem2 <USB_CLK_SEL_ADDR> w
# تأكد إن bits الـ mux = 0x2 (hfosc1) مش 0x0 (reserved)
```

#### الدرس المستفاد
الـ flags `CLK_MUX_INDEX_ONE` و`CLK_MUX_INDEX_BIT` موجودة لأن كتير من الـ SoCs بتستخدم register encodings غير sequential أو بتبدأ من 1. لازم دايماً تقرأ الـ TRM بعناية وتتحقق من الـ register encoding الفعلي. الـ `table` explicit هو الأوضح والأأمن لأنه بيلغي أي ambiguity، خصوصاً في bring-up phase لما الـ hardware غير متوقع.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي بتشرح الـ Common Clock Framework والـ clock mux بالتفصيل:

| المقال | الرابط | الأهمية |
|--------|--------|---------|
| A common clock framework (2012) | [lwn.net/Articles/472998](https://lwn.net/Articles/472998/) | المقال الأساسي اللي بيشرح الـ framework من الأول |
| common clk framework — patch submission | [lwn.net/Articles/486841](https://lwn.net/Articles/486841/) | تفاصيل الـ patch الأولاني للـ framework |
| Documentation/clk.txt discussion | [lwn.net/Articles/489668](https://lwn.net/Articles/489668/) | شرح الـ documentation الرسمي جوه الكيرنل |
| clk: dt bindings for mux, divider & gate | [lwn.net/Articles/555242](https://lwn.net/Articles/555242/) | الـ DT bindings الخاصة بالـ mux clock |
| Common Clk Framework — kernel docs mirror | [static.lwn.net/kerneldoc/driver-api/clk.html](https://static.lwn.net/kerneldoc/driver-api/clk.html) | الـ API reference الكامل |
| clk: Add kunit tests for fixed rate and parent data | [lwn.net/Articles/976983](https://lwn.net/Articles/976983/) | اختبارات الـ `clk_parent_data` في الـ framework |

---

### الـ Official Kernel Documentation

الملفات دي موجودة جوه الكيرنل نفسه وبتشرح الـ subsystem بشكل رسمي:

```
Documentation/driver-api/clk.rst          ← الـ API reference الكامل للـ Common Clock Framework
Documentation/devicetree/bindings/clock/  ← الـ DT bindings لكل أنواع الـ clocks
include/linux/clk-provider.h              ← تعريفات struct clk_mux و clk_mux_ops و الـ macros
drivers/clk/clk-mux.c                     ← الـ implementation نفسه
drivers/clk/clk.c                         ← الـ core framework اللي بيشتغل clk-mux معاه
```

**الرابط الرسمي على kernel.org:**
- [docs.kernel.org/driver-api/clk.html](https://docs.kernel.org/driver-api/clk.html)
- [kernel.org/doc/Documentation/clk.txt](https://www.kernel.org/doc/Documentation/clk.txt)

---

### Kernel Commits المهمة

#### الـ commit الأصلي للـ framework
الـ Common Clock Framework اتحط في الكيرنل في **Linux 3.4** سنة 2012 بواسطة **Mike Turquette** من Linaro. الـ `clk-mux.c` اتكتب في نفس الوقت بإيد **Sascha Hauer** و **Richard Zhao**.

- الـ initial submission على LKML: [lists.ubuntu.com/archives/kernel-team/2012-June/019862.html](https://lists.ubuntu.com/archives/kernel-team/2012-June/019862.html)
- ملف `clk-mux.c` على Android Google Source (نسخة قديمة للمقارنة): [android.googlesource.com — clk-mux.c](https://android.googlesource.com/kernel/msm/+/android-7.1.0_r0.2/drivers/clk/clk-mux.c)

#### تغييرات مهمة بعد كده
- **إضافة `CLK_MUX_HIWORD_MASK`**: لـ Rockchip SoCs اللي بتستخدم الـ 16-bit high word كـ mask — Haojian Zhuang هو اللي عمل الـ patch ده.
- **تحويل لـ `clk_hw` based API**: patch من Stephen Boyd سنة 2016 — [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1454982341-22715-9-git-send-email-sboyd@codeaurora.org/)
- **إضافة `clk_parent_data`**: بتسمح بتسجيل الـ parents بـ DT index أو `clk_hw` pointer بدل اسم string بس.

---

### Mailing List Discussions

نقاشات مهمة على Linux kernel mailing list:

| الموضوع | الرابط |
|---------|--------|
| DT bindings للـ mux, divider, gate clocks | [linux.kernel.narkive.com](https://linux.kernel.narkive.com/OV6yp6s3/patch-v2-0-5-clk-dt-bindings-for-mux-divider-gate-clocks) |
| clk mux: switch to new clock controller API | [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-clk/patch/1471354514-24224-8-git-send-email-k.kozlowski@samsung.com/) |
| إضافة `clk_hw` based registration APIs | [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1454982341-22715-9-git-send-email-sboyd@codeaurora.org/) |
| DT binding للـ basic multiplexer clock | [lore.kernel.org](https://lore.kernel.org/lkml/1371437905-15567-4-git-send-email-mturquette@linaro.org/) |
| HIWORD_MASK لـ Renesas RZ/G2L | [git.zx2c4.com](https://git.zx2c4.com/linux-rng/commit/drivers/clk?id=75b0ad42ccd9a87873e91598116471d9991b09ea) |

---

### الكتب المرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
- **الكتاب مجاني**: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)
- الـ Chapter 1 (Device Driver Concepts) و Chapter 14 (The Linux Device Model) بيشرحوا الـ driver model اللي الـ clock framework بني عليه.
- الكتاب قديم (2005) ومش بيغطي الـ Common Clock Framework، بس بيشرح مفاهيم أساسية زي الـ `module_init` و الـ `EXPORT_SYMBOL_GPL` و الـ device model.

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل المهم**: Chapter 17 — Devices and Modules
- بيشرح الـ `kobject` و الـ device model اللي بتشتغل عليه الـ `devres` و الـ `devm_*` functions المستخدمة في `__devm_clk_hw_register_mux`.
- بيغطي الـ spinlock و الـ `spin_lock_irqsave` اللي بيستخدمها `clk_mux_set_parent`.

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل المهم**: Chapter 15 — Kernel Initialization
- بيشرح إزاي الـ clock tree بيتبني في وقت الـ boot وإزاي الـ SoC drivers بتسجل الـ clocks.
- مفيد لفهم الـ `of_clk_hw_register` و الـ DT-based clock registration.

---

### eLinux.org

| المرجع | الرابط | المحتوى |
|--------|--------|---------|
| Common Clock Framework (BoFs) — PDF | [elinux.org](https://elinux.org/images/b/b1/Common_Clock_Framework_(BoFs).pdf) | عرض تقديمي عن الـ framework من Embedded Linux Conference |
| Generic Linux CLK Framework — CELF 2009 — PDF | [elinux.org](https://elinux.org/images/6/64/ELC_E_2009_Generic_Clock_Framework.pdf) | التاريخ قبل الـ Common Clock Framework |
| Linux clock management framework — ELC 2007 — PDF | [elinux.org](https://elinux.org/images/a/a3/ELC_2007_Linux_clock_fmw.pdf) | الجيل الأول من الـ clock management في الكيرنل |

---

### KernelNewbies.org

الـ kernelnewbies بيوثق التغييرات في كل إصدار كيرنل. ابحث عن "clock" في الصفحات دي:

| الإصدار | الرابط | ما يهمك فيه |
|---------|--------|-------------|
| Linux 3.4 | [kernelnewbies.org/Linux_3.4](https://kernelnewbies.org/Linux_3.4) | أول إصدار فيه الـ Common Clock Framework |
| Linux 5.9  | [kernelnewbies.org/Linux_5.9](https://kernelnewbies.org/Linux_5.9) | تحديثات على الـ clock framework |
| Linux 6.7  | [kernelnewbies.org/Linux_6.7](https://kernelnewbies.org/Linux_6.7) | أحدث تغييرات في الـ clock subsystem |

---

### مصادر إضافية

#### Bootlin (Free Electrons) — عرض مفيد جداً
- [Common Clock Framework: how to use it — ELCE 2013 — PDF](https://bootlin.com/pub/conferences/2013/elce/common-clock-framework-how-to-use-it/common-clock-framework-how-to-use-it.pdf)
- ده من أفضل الشروحات العملية لإزاي تكتب clock driver باستخدام الـ framework.

#### STM32 Clock Overview
- [wiki.st.com/stm32mpu/wiki/Clock_overview](https://wiki.st.com/stm32mpu/wiki/Clock_overview)
- مثال عملي ممتاز لـ SoC حقيقي بيستخدم كل أنواع الـ clock types زي الـ mux و divider و gate معاً.

#### kernelconfig.io
- [kernelconfig.io/config_common_clk](https://www.kernelconfig.io/config_common_clk)
- بيشرح الـ `CONFIG_COMMON_CLK` وعلاقته بباقي الـ Kconfig options.

---

### Search Terms للبحث عن معلومات أكتر

لو حبيت تدور بنفسك، استخدم الـ search terms دي:

```
# للبحث في LWN.net
site:lwn.net "common clock framework"
site:lwn.net "clk_mux" OR "clk-mux"
site:lwn.net "clk_hw_register"

# للبحث في الـ mailing lists
site:lore.kernel.org clk-mux CLK_MUX_HIWORD_MASK
site:patchwork.kernel.org "clk: mux"
linux-clk@vger.kernel.org clk_mux_ops

# للبحث في الـ kernel source
grep -r "clk_mux_ops\|CLK_MUX_HIWORD_MASK\|clk_hw_register_mux" drivers/clk/

# للبحث عن استخدامات حقيقية في drivers
grep -r "__clk_hw_register_mux\|clk_register_mux_table" drivers/

# للبحث في git history
git log --oneline drivers/clk/clk-mux.c
git log --all --grep="CLK_MUX_HIWORD_MASK"
```
## Phase 8: Writing simple module

### الفكرة

الـ `clk_mux_val_to_index` هي دالة exported من `clk-mux.c` بتتحول فيها قيمة الـ register الخام لـ index الـ parent. هنعمل **kprobe** عليها عشان نشوف كل مرة الـ clock framework بيحاول يحدد مين الـ parent الحالي للـ mux clock.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on clk_mux_val_to_index()
 * Prints hw pointer, raw register value, and resolved parent index
 * every time the CCF asks "which parent is selected?"
 */

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/kprobes.h>
#include <linux/clk-provider.h>   /* struct clk_hw, clk_hw_get_name() */

/* ------------------------------------------------------------------ */
/* pre-handler: called just before clk_mux_val_to_index() executes    */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * clk_mux_val_to_index(struct clk_hw *hw,
     *                       const u32     *table,
     *                       unsigned int   flags,
     *                       unsigned int   val)
     *
     * On x86-64:  rdi=hw, rsi=table, rdx=flags, rcx=val
     * On ARM64:   x0=hw,  x1=table,  x2=flags,  x3=val
     */
#ifdef CONFIG_X86_64
    struct clk_hw   *hw    = (struct clk_hw *)regs->di;
    unsigned int     val   = (unsigned int)regs->cx;
    unsigned int     flags = (unsigned int)regs->dx;
#elif defined(CONFIG_ARM64)
    struct clk_hw   *hw    = (struct clk_hw *)regs->regs[0];
    unsigned int     val   = (unsigned int)regs->regs[3];
    unsigned int     flags = (unsigned int)regs->regs[2];
#else
    struct clk_hw   *hw    = NULL;
    unsigned int     val   = 0;
    unsigned int     flags = 0;
#endif

    /* clk_hw_get_name() آمنة تُستدعى هنا لأننا في sleepable context */
    const char *name = (hw && clk_hw_get_name(hw))
                       ? clk_hw_get_name(hw)
                       : "(unknown)";

    pr_info("clk_mux_probe: clk=%s  raw_val=0x%x  mux_flags=0x%x\n",
            name, val, flags);

    return 0; /* 0 = let the probed function run normally */
}

/* ------------------------------------------------------------------ */
/* post-handler: called after the function returns                     */
/* ------------------------------------------------------------------ */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * ax (x86-64) / x0 (ARM64) holds the return value = parent index
     * or negative errno if val was out of range.
     */
#ifdef CONFIG_X86_64
    long ret = (long)regs->ax;
#elif defined(CONFIG_ARM64)
    long ret = (long)regs->regs[0];
#else
    long ret = 0;
#endif

    if (ret >= 0)
        pr_info("clk_mux_probe: => parent_index=%ld\n", ret);
    else
        pr_info("clk_mux_probe: => INVALID val (err=%ld)\n", ret);
}

/* ------------------------------------------------------------------ */
/* kprobe struct                                                        */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "clk_mux_val_to_index",
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

/* ------------------------------------------------------------------ */
/* init / exit                                                          */
/* ------------------------------------------------------------------ */
static int __init clk_mux_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("clk_mux_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("clk_mux_probe: planted on %s @ %p\n",
            kp.symbol_name, kp.addr);
    return 0;
}

static void __exit clk_mux_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("clk_mux_probe: removed\n");
}

module_init(clk_mux_probe_init);
module_exit(clk_mux_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on clk_mux_val_to_index to trace mux parent resolution");
```

---

### Makefile

```makefile
obj-m += clk_mux_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ includes

```c
#include <linux/kprobes.h>
#include <linux/clk-provider.h>
```

**الـ** `kprobes.h` بيجيب `struct kprobe` والـ API بتاعته. **الـ** `clk-provider.h` بيجيب `struct clk_hw` و`clk_hw_get_name()` اللي بنستخدمها عشان نطبع اسم الـ clock بدل عنوان الـ pointer الخام.

---

#### الـ `handler_pre`

```c
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
```

الـ `pt_regs` بيحتوي على قيم الـ registers لحظة الاعتراض. بنقرأ منها الـ arguments بترتيب calling convention (rdi/x0 = hw، rcx/x3 = val) عشان نشوف الـ raw register value قبل ما تتحول لـ index.

---

#### الـ `handler_post`

```c
static void handler_post(struct kprobe *p, struct pt_regs *regs, unsigned long flags)
```

بيتنفذ بعد ما الدالة الأصلية ترجع. بنقرأ الـ return value من rax/x0 عشان نعرف إيه الـ parent index اللي اتحدد، أو نكشف لو الـ val كانت خارج النطاق وراجعت error.

---

#### الـ `kp` struct

```c
static struct kprobe kp = {
    .symbol_name = "clk_mux_val_to_index",
    ...
};
```

**الـ** `symbol_name` بيخلي الـ kernel يحل العنوان وقت التسجيل بدون ما نحتاج نعرف العنوان hardcoded — ده أأمن وأسهل للـ portability بين kernels مختلفة.

---

#### الـ `module_init`

```c
ret = register_kprobe(&kp);
```

بنسجل الـ probe هنا. لو فشل (مثلاً الـ symbol مش exported أو الـ kprobes مش enabled في الـ config) بنرجع الـ error فوراً ومحدش ييجي يستخدم module مش شغال.

---

#### الـ `module_exit`

```c
unregister_kprobe(&kp);
```

لازم نشيل الـ probe في الـ exit عشان لو الـ module اتـ unload والـ probe لسه مسجل، أي call لـ `clk_mux_val_to_index` هيجي على كود مش موجود وده هيعمل kernel panic. الـ unregister بيضمن إن الـ breakpoint اتشال وأي handler شغال خلص قبل ما تكمل.

---

### تشغيل وقراءة النتيجة

```bash
# بناء
make

# تحميل
sudo insmod clk_mux_probe.ko

# مشاهدة الـ output
sudo dmesg -w | grep clk_mux_probe

# مثال للـ output
# clk_mux_probe: planted on clk_mux_val_to_index @ ffffffffc0a1b340
# clk_mux_probe: clk=uart0_clk  raw_val=0x2  mux_flags=0x0
# clk_mux_probe: => parent_index=2
# clk_mux_probe: clk=spi1_clk   raw_val=0x1  mux_flags=0x0
# clk_mux_probe: => parent_index=1

# تفريغ
sudo rmmod clk_mux_probe
```

**الـ** output بيكشف إزاي كل clock بيقرأ الـ hardware register ويحوله لـ parent index — معلومة مفيدة جداً لما تكون بتـ debug مشكلة في الـ clock tree على embedded board.
