# `clkdev.h`

> **PATH**: `include/linux/clkdev.h`
> **Subsystem**: Clock (CLK) Framework — واجهة الـ Lookup API
> **الوظيفة الأساسية**: تعريف `struct clk_lookup` وإعلان الدوال العامة لتسجيل والبحث عن الـ clocks

---

## Phase 1: الملف والسياق

### ما هو هذا الملف؟

`clkdev.h` هو "بطاقة الـ API" لنظام الـ clock lookup في Linux. أي ملف يريد إضافة أو البحث عن clock entries في قائمة الـ clkdev لازم يضمّ هذا الـ header.

هو بسيط ومركّز: يعرّف struct واحد، macro واحد، و 7 دوال فقط. الـ implementation التفصيلية كلها في `drivers/clk/clkdev.c`.

### علاقته بالملفات الأخرى

```
clkdev.h  ←  يُضمَّن في
├── drivers/clk/clkdev.c     (الـ implementation)
├── كل driver يريد تسجيل clocks
└── بعض الـ SoC-specific clock drivers

clkdev.h  يعتمد على
└── <linux/slab.h>           (لـ kzalloc/kfree)
```

### الـ Includes

| Include | ما الذي يجلبه |
|---------|----------------|
| `<linux/slab.h>` | memory allocation APIs (kzalloc, kfree) — مطلوبة داخلياً |

---

## Phase 2: شرح الـ CLK Framework

### الهدف من هذا الـ Header

مشكلة الـ clock lookup باختصار: الـ peripheral drivers يحتاجون clocks، لكن لا يجب أن يعرفوا تفاصيل الـ hardware (اسم الـ PLL، القيمة الدقيقة، إلخ). الحل: جدول وسيط يربط **(اسم الجهاز + اسم الـ connection)** بـ **(clock handle)**.

هذا الـ header يعرّف قطعتي هذه الجسرة:
1. **`struct clk_lookup`**: السجل الواحد في الجدول
2. **الدوال**: لإضافة/حذف/بحث في الجدول

للشرح الكامل للـ framework، انظر: [`drivers/clk/clkdev.c.md`](../../drivers/clk/clkdev.c.md)

---

## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

### `struct clk_lookup`

```c
struct clk_lookup {
    struct list_head    node;    /* عقدة في القائمة المرتبطة العالمية */
    const char         *dev_id;  /* اسم الـ device (مثل "ff030000.serial") */
    const char         *con_id;  /* اسم الـ connection (مثل "baudclk") */
    struct clk         *clk;     /* مرجع قديم للـ clock (legacy) */
    struct clk_hw      *clk_hw;  /* مرجع حديث للـ hardware clock descriptor */
};
```

**شرح كل حقل**:

| الحقل | النوع | الغرض | ملاحظة |
|-------|-------|--------|---------|
| `node` | `list_head` | لربط السجل بالقائمة `clocks` في `clkdev.c` | داخلي، لا تلمسه مباشرة |
| `dev_id` | `const char *` | اسم الـ device صاحب الـ clock | NULL = wildcard (يطابق أي device) |
| `con_id` | `const char *` | اسم الـ connection/role للـ clock | NULL = wildcard (يطابق أي connection) |
| `clk` | `struct clk *` | مؤشر للـ clock (أسلوب قديم) | اُستبدل بـ `clk_hw` |
| `clk_hw` | `struct clk_hw *` | مؤشر للـ hardware descriptor | الأسلوب الحديث المفضّل |

**قواعد الـ Wildcard** (من تعليق `clk_find()` في `clkdev.c`):
```
NULL dev_id → يقبل أي device
NULL con_id → يقبل أي connection

أولوية التطابق:
  dev_id + con_id  ← أعلى أولوية (3 نقاط)
  dev_id فقط      ← أولوية متوسطة (2 نقطة)
  con_id فقط      ← أولوية منخفضة (1 نقطة)
  لا شيء          ← لا يُرجَع (0 نقطة، wildcard كامل)
```

### ملاحظة على حقل `clk` vs `clk_hw`

```c
/* القديم (pre-CCF) */
struct clk_lookup old = {
    .dev_id = "uart0",
    .con_id = "fclk",
    .clk    = &my_clk_struct,   /* ← غير موصى به */
};

/* الحديث (CCF) */
struct clk_lookup modern = {
    .dev_id  = "uart0",
    .con_id  = "fclk",
    .clk_hw  = &my_hw,           /* ← الأفضل */
};
```

**لماذا `clk_hw` أفضل من `clk`؟** لأن `struct clk_hw` هو الـ hardware representation الحقيقي، بينما `struct clk` هو consumer handle يُنشأ لكل consumer. استخدام `clk_hw` مباشرة أكثر كفاءة.

### `CLKDEV_INIT` Macro

```c
#define CLKDEV_INIT(d, n, c) \
    {                        \
        .dev_id = d,         \
        .con_id = n,         \
        .clk = c,            \
    }
```

**مثال الاستخدام** (أسلوب قديم للـ static tables):
```c
static struct clk_lookup board_clks[] = {
    CLKDEV_INIT("uart0",  "fclk",    &uart_clk),
    CLKDEV_INIT("spi0",   "spi_clk", &spi_clk),
    CLKDEV_INIT(NULL,     "ref_clk", &ref_clk),  /* wildcard device */
};

static void __init board_init_clocks(void)
{
    clkdev_add_table(board_clks, ARRAY_SIZE(board_clks));
}
```

**ملاحظة**: هذا الـ macro يستخدم `.clk` القديم. الأسلوب الحديث هو `clkdev_hw_create()`.

---

## Phase 4: دليل الـ Debugging الشامل

### تشخيص مشاكل الـ Lookup

```bash
# هل الـ clock مسجّلة في النظام؟
cat /sys/kernel/debug/clk/clk_summary

# البحث عن clock معينة
cat /sys/kernel/debug/clk/clk_summary | grep -i "uart\|spi\|i2c"

# رسائل الخطأ الشائعة مع هذا الـ header
dmesg | grep -E "couldn't get clock|ENOENT|EPROBE_DEFER"
```

### Dynamic Debug

```bash
# تفعيل debug للملف الـ implementation
echo "file clkdev.c +p" > /sys/kernel/debug/dynamic_debug/control
```

لمزيد من التفاصيل، انظر قسم الـ Debugging في: [`drivers/clk/clkdev.c.md`](../../drivers/clk/clkdev.c.md)

---

## Phase 5: شرح الـ Functions (المُعلَنة في هذا الـ Header)

### جدول سريع للـ API

| الدالة | الغرض | متى تستخدمها |
|--------|--------|--------------|
| `clkdev_add()` | إضافة lookup ثابت | جداول static مع `CLKDEV_INIT` |
| `clkdev_drop()` | حذف وتحرير lookup | مع `clkdev_create/hw_create` |
| `clkdev_create()` | إنشاء lookup من `clk` | legacy code |
| `clkdev_hw_create()` | إنشاء lookup من `clk_hw` | الأسلوب الحديث المفضّل |
| `clkdev_add_table()` | إضافة مصفوفة كاملة | board-level registration |
| `clk_add_alias()` | إضافة اسم بديل | backwards compatibility |
| `clk_register_clkdev()` | تسجيل من `clk` | في clock drivers القديمة |
| `clk_hw_register_clkdev()` | تسجيل من `clk_hw` | في clock drivers الحديثة |
| `devm_clk_hw_register_clkdev()` | تسجيل managed | الأفضل دائماً |

---

#### `clkdev_add()`

```c
void clkdev_add(struct clk_lookup *cl);
```

تضيف `clk_lookup` إلى القائمة العالمية. للـ statically-allocated entries فقط (مثل `CLKDEV_INIT`). **لا تستدعِ `clkdev_drop()` على ما أضفته بهذه الطريقة.**

---

#### `clkdev_drop()`

```c
void clkdev_drop(struct clk_lookup *cl);
```

تحذف `cl` من القائمة وتحرر ذاكرتها. **فقط** للـ entries المُنشأة بـ `clkdev_create()` أو `clkdev_hw_create()`.

---

#### `clkdev_create()`

```c
struct clk_lookup *clkdev_create(struct clk *clk, const char *con_id,
    const char *dev_fmt, ...) __printf(3, 4);
```

تُنشئ وتُسجّل lookup من `struct clk`. الـ `dev_fmt` هو format string لاسم الـ device.

```c
/* مثال */
struct clk_lookup *lk = clkdev_create(my_clk, "fclk", "uart.%d", port_num);
/* لاحقاً */
clkdev_drop(lk);
```

---

#### `clkdev_hw_create()`

```c
struct clk_lookup *clkdev_hw_create(struct clk_hw *hw, const char *con_id,
    const char *dev_fmt, ...) __printf(3, 4);
```

نفس `clkdev_create()` لكن من `struct clk_hw`. **الأسلوب المفضّل حديثاً.**

---

#### `clkdev_add_table()`

```c
void clkdev_add_table(struct clk_lookup *, size_t);
```

تضيف مصفوفة كاملة في عملية واحدة (lock واحد = أكفأ).

---

#### `clk_add_alias()`

```c
int clk_add_alias(const char *alias, const char *alias_dev_name,
    const char *con_id, struct device *dev);
```

تضيف اسم بديل لـ clock موجودة. مفيدة للـ backwards compatibility.

```c
/* Driver قديم يطلب "old_name"، الـ clock مسجّلة بـ "new_name" */
clk_add_alias("old_name", NULL, "new_name", clock_dev);
```

---

#### `clk_register_clkdev()`

```c
int clk_register_clkdev(struct clk *, const char *, const char *);
```

تسجيل lookup بسيط. المعاملات: `(clk, con_id, dev_id)`. تتعامل بشكل صحيح مع `dev_id=NULL` (wildcard).

---

#### `clk_hw_register_clkdev()`

```c
int clk_hw_register_clkdev(struct clk_hw *, const char *, const char *);
```

نفس `clk_register_clkdev()` لكن من `clk_hw`. المعاملات: `(hw, con_id, dev_id)`.

---

#### `devm_clk_hw_register_clkdev()`

```c
int devm_clk_hw_register_clkdev(struct device *dev, struct clk_hw *hw,
    const char *con_id, const char *dev_id);
```

**الأفضل دائماً.** تسجيل managed — الـ kernel يستدعي `clkdev_drop()` تلقائياً عند إزالة الـ device.

```c
/* ✅ الأسلوب الصحيح في الـ drivers الحديثة */
static int my_clk_probe(struct platform_device *pdev)
{
    struct clk_hw *hw;
    int ret;

    hw = devm_clk_hw_register(&pdev->dev, &my_hw);
    if (IS_ERR(hw))
        return PTR_ERR(hw);

    return devm_clk_hw_register_clkdev(&pdev->dev, hw, "fclk", NULL);
    /* لا تحتاج remove() للـ cleanup! */
}
```

---

## Phase 6: سيناريوهات من الحياة العملية

### السيناريو 1: إضافة Clock Table في Board File (RK3399)

**السياق**: Board bring-up على Rockchip RK3399. الـ DT ما كتمل بعد.

```c
/* في arch/arm64/mach-rk3399/board.c */
#include <linux/clkdev.h>

static struct clk_lookup rk3399_clk_lookups[] = {
    CLKDEV_INIT("ff110000.i2c", "i2c",    &i2c0_clk),
    CLKDEV_INIT("ff120000.i2c", "i2c",    &i2c1_clk),
    CLKDEV_INIT(NULL,           "ref_24m", &osc_clk),
};

static int __init rk3399_clk_init(void)
{
    clkdev_add_table(rk3399_clk_lookups, ARRAY_SIZE(rk3399_clk_lookups));
    return 0;
}
arch_initcall(rk3399_clk_init);
```

---

### السيناريو 2: Clock Driver على STM32MP1 بأسلوب حديث

```c
/* في drivers/clk/stm32/clk-stm32mp1.c */
#include <linux/clkdev.h>
#include <linux/clk-provider.h>

static int stm32mp1_register_clock(struct device *dev,
                                    struct clk_hw *hw,
                                    const char *con_id)
{
    /* devm تضمن cleanup تلقائي */
    return devm_clk_hw_register_clkdev(dev, hw, con_id, NULL);
}
```

---

### السيناريو 3: فهم الفرق بين `con_id=NULL` و `dev_id=NULL`

```c
/* Wildcard كامل — يطابق أي طلب */
clkdev_hw_create(hw, NULL, NULL);

/* يطابق أي device تطلب "ref_clk" */
clkdev_hw_create(hw, "ref_clk", NULL);

/* يطابق "uart.0" بأي con_id */
clkdev_hw_create(hw, NULL, "uart.%d", 0);

/* دقيق — فقط "uart.0" تطلب "baudclk" */
clkdev_hw_create(hw, "baudclk", "uart.%d", 0);
```

**الـ driver يطلب**:
```c
struct clk *clk = clk_get(&uart_dev, "baudclk");
/* dev_name(&uart_dev) = "uart.0" */
/* سيختار الـ entry الأدق تطابقاً */
```

---

### السيناريو 4: Backwards Compatibility على Allwinner H6

```c
/* الـ CRU driver الجديد يسجّل بأسماء جديدة */
/* لكن drivers قديمة تطلب أسماء قديمة */

static const struct {
    const char *new_name;
    const char *old_name;
} h6_clk_aliases[] = {
    { "clk-uart0",   "uart0-clk"  },
    { "clk-spi0",    "spi0-clk"   },
    { "clk-i2c0",    "i2c0-clk"   },
};

static void h6_register_aliases(struct device *dev)
{
    int i;
    for (i = 0; i < ARRAY_SIZE(h6_clk_aliases); i++) {
        clk_add_alias(h6_clk_aliases[i].old_name, NULL,
                      h6_clk_aliases[i].new_name, dev);
    }
}
```

---

### السيناريو 5: تشخيص MAX_CON_ID Error

**الرسالة**:
```
clkdev: null-device:my_very_long_connection_name: connection ID is greater than 16
```

**الحل**:
```c
/* ❌ خاطئ — الاسم أطول من MAX_CON_ID=16 */
clkdev_hw_create(hw, "my_very_long_connection_name", "uart.0");

/* ✅ صحيح — قصّر الاسم */
clkdev_hw_create(hw, "baudclk", "uart.0");

/* أو زد الحد (يتطلب تعديل kernel) */
/* في drivers/clk/clkdev.c */
#define MAX_CON_ID  32  /* زيادة من 16 */
```

---

## Phase 7: مصادر ومراجع

### توثيق الـ Kernel الرسمي

```
Documentation/driver-api/clk.rst
Documentation/devicetree/bindings/clock/
```

### الملفات المرتبطة

```
drivers/clk/clkdev.c          ← الـ implementation الكاملة
include/linux/clk.h            ← consumer API
include/linux/clk-provider.h   ← provider API و struct clk_hw
```

### وثيقة الـ Implementation الكاملة

للشرح التفصيلي لكل شيء (الـ algorithm، الـ debugging، الـ scenarios):
- **[`drivers/clk/clkdev.c.md`](../../drivers/clk/clkdev.c.md)**

### مصطلحات للبحث

```
"struct clk_lookup linux"
"CLKDEV_INIT linux kernel"
"clkdev_hw_create"
"devm_clk_hw_register_clkdev"
"clock lookup linux embedded"
```

---

*تم توليد هذا التوثيق تلقائياً بواسطة Linux Kernel Arabic Documentation Skill*
*المصدر: `include/linux/clkdev.h` — Linux Kernel*
