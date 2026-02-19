# `clkdev.c`

> **PATH**: `drivers/clk/clkdev.c`
> **Subsystem**: Clock (CLK) Framework — طبقة الـ Lookup
> **الوظيفة الأساسية**: نظام البحث والتسجيل الذي يربط أسماء الأجهزة بالـ clocks المناسبة لها

---

## Phase 1: الملف والسياق

### ما هو هذا الملف؟

`clkdev.c` هو "دفتر العناوين" للـ clock framework في Linux. مهمته الوحيدة هي الإجابة على سؤال: **"أنا driver لـ device X، عايز clock اسمها Y — وين أجيبها؟"**

الملف جزء من subsystem أكبر هو الـ **Common Clock Framework (CCF)**، لكنه متخصص في مشكلة واحدة: الـ **lookup** أو البحث.

### الـ Includes الرئيسية

| Include | ما الذي يجلبه |
|---------|----------------|
| `<linux/clk.h>` | تعريف `struct clk` والـ consumer API |
| `<linux/clkdev.h>` | تعريف `struct clk_lookup` والـ CLKDEV_INIT macro |
| `<linux/clk-provider.h>` | تعريف `struct clk_hw` (الجانب الـ provider) |
| `<linux/mutex.h>` | الـ mutex لحماية قائمة الـ clocks |
| `<linux/list.h>` | الـ linked list API |
| `<linux/of.h>` | دعم الـ Device Tree |
| `"clk.h"` | internal APIs للـ CCF مثل `clk_hw_create_clk()` |

---

## Phase 2: شرح الـ CLK Framework

### المشكلة التي يحلها

تخيل معك مدينة فيها آلاف المصانع (الـ peripheral drivers). كل مصنع يحتاج كهرباء (clock) بتردد معين. فيه محطة كهرباء (clock provider) تولّد ترددات مختلفة. **المشكلة**: كيف يعرف كل مصنع أي "خط كهرباء" يتوصل منه؟

بدون نظام lookup، كل driver كان لازم يعرف اسم الـ clock بالضبط وكيف يوصل عليها — وده صعب ومش portable.

### الحل: نظام الـ Lookup Table

```
┌─────────────────────────────────────────────────────────┐
│                 Consumer Side (الـ Drivers)              │
│  clk_get(dev, "uart")  ←  SPI driver, UART driver, ... │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│              clkdev Lookup Layer (هذا الملف)             │
│                                                         │
│   Global List: [clk_lookup] → [clk_lookup] → ...       │
│   ┌────────────────┐  ┌────────────────┐               │
│   │ dev_id: "uart0"│  │ dev_id: "spi0" │               │
│   │ con_id: "fclk" │  │ con_id: "clk"  │               │
│   │ clk_hw: ──────►│  │ clk_hw: ──────►│               │
│   └────────────────┘  └────────────────┘               │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│           Provider Side (الـ clk_hw implementations)    │
│   clk_hw  ←→  struct clk_ops  ←→  Hardware registers   │
│   (PLL, divider, mux, gate, ...)                        │
└─────────────────────────────────────────────────────────┘
```

### المسارات المتاحة للـ Lookup

**مسار الـ Device Tree** (الأحدث والمفضّل):
```
dev->of_node → of_clk_get_hw() → phandle في الـ DT → clk_hw
```

**مسار الـ clkdev** (الكلاسيكي للـ non-DT):
```
(dev_id, con_id) → clk_find() → clk_lookup → clk_hw
```

`clk_get()` يجرب الـ DT أولاً، وإذا فشل يرجع للـ clkdev.

### ملفات الـ Subsystem الرئيسية

| الملف | الدور |
|-------|-------|
| `drivers/clk/clk.c` | الـ core — إدارة الـ clock tree |
| `drivers/clk/clkdev.c` | هذا الملف — الـ lookup layer |
| `include/linux/clk.h` | الـ consumer API (العامة) |
| `include/linux/clkdev.h` | تعريف `struct clk_lookup` |
| `include/linux/clk-provider.h` | الـ provider API |
| `drivers/clk/clk-*.c` | drivers لأنواع مختلفة (PLL, mux, divider) |

---

## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

### الـ Structs الأساسية

#### `struct clk_lookup` (معرّف في `clkdev.h`)

```c
struct clk_lookup {
    struct list_head  node;    /* ربط في القائمة العالمية */
    const char       *dev_id;  /* اسم الـ device (مثل "uart0") */
    const char       *con_id;  /* اسم الـ connection (مثل "fclk") */
    struct clk       *clk;     /* قديم — مش مستخدم كتير */
    struct clk_hw    *clk_hw;  /* الـ clock hardware descriptor */
};
```

**الفكرة**: كل سطر في "دفتر العناوين" هو `clk_lookup`. فيه اسم الجهاز، اسم الـ connection، والـ clock نفسها.

#### `struct clk_lookup_alloc` (داخلي في `clkdev.c`)

```c
struct clk_lookup_alloc {
    struct clk_lookup cl;          /* الـ lookup entry نفسه */
    char dev_id[MAX_DEV_ID];      /* MAX_DEV_ID = 24 حرف */
    char con_id[MAX_CON_ID];      /* MAX_CON_ID = 16 حرف */
};
```

**الفكرة**: لما نخصص `clk_lookup` ديناميكياً (بـ `kzalloc`)، نحتاج مكان للنصوص نفسها. هذا الـ struct يجمع الكل في allocation واحدة.

```
┌───────────────────────────────────────┐
│       clk_lookup_alloc                │
│  ┌─────────────────────────────────┐  │
│  │  clk_lookup cl                  │  │
│  │  ├── node (list_head)           │  │
│  │  ├── dev_id ─────────────────┐  │  │
│  │  ├── con_id ──────────────┐  │  │  │
│  │  └── clk_hw               │  │  │  │
│  └───────────────────────────│──│──┘  │
│  char dev_id[24] ◄───────────┘  │     │
│  char con_id[16] ◄──────────────┘     │
└───────────────────────────────────────┘
```

### القائمة العالمية والـ Mutex

```c
static LIST_HEAD(clocks);           /* رأس القائمة المرتبطة */
static DEFINE_MUTEX(clocks_mutex);  /* الـ mutex لحمايتها */
```

**القاعدة**: كل وصول للـ `clocks` list لازم يكون داخل `mutex_lock/unlock`. الـ `lockdep_assert_held()` في `clk_find()` تتأكد من ده وقت الـ debugging.

### دورة حياة الـ clk_lookup

```
clkdev_hw_create() / clkdev_create()
          │
          ▼
    vclkdev_alloc()          ← kzalloc لـ clk_lookup_alloc
          │                  ← نسخ dev_id و con_id
          ▼
    vclkdev_create()
          │
          ▼
    __clkdev_add()           ← mutex_lock
          │                  ← list_add_tail() للقائمة
          │                  ← mutex_unlock
          ▼
    [في القائمة - جاهز للـ lookup]
          │
          ▼ (عند الانتهاء)
    clkdev_drop()            ← mutex_lock
                             ← list_del()
                             ← mutex_unlock
                             ← kfree()
```

### خوارزمية الـ Fuzzy Matching في `clk_find()`

هذي من أذكى أجزاء الكود. الفكرة: match الأكثر تحديداً يفوز.

```
النقاط:
  dev_id يتطابق  → +2 نقطة
  con_id يتطابق  → +1 نقطة
  أفضل مجموع → الفائز

أمثلة:
  dev_id="uart0", con_id="fclk" → best_possible = 3

  Entry A: dev_id="uart0", con_id=NULL  → match=2
  Entry B: dev_id=NULL, con_id="fclk"  → match=1
  Entry C: dev_id="uart0", con_id="fclk" → match=3 ← الفائز
  Entry D: dev_id=NULL, con_id=NULL      → match=0
```

**الـ wildcard**: أي entry بدون dev_id يقبل أي device، وبدون con_id يقبل أي connection.

### علاقات الـ Call Flow

```
clk_get(dev, "uart")
    │
    ├── [DT path] of_clk_get_hw(dev->of_node, 0, "uart")
    │       └── [نجح] → clk_hw_create_clk()
    │
    └── [fallback] __clk_get_sys(dev, dev_id, con_id)
            └── clk_find_hw(dev_id, con_id)
                    └── mutex_lock
                        └── clk_find(dev_id, con_id)    ← البحث الفعلي
                            └── list_for_each_entry()
                        └── mutex_unlock
                └── clk_hw_create_clk(dev, hw, ...)     ← إنشاء clk handle
```

### استراتيجية الـ Locking

| الـ Lock | ما يحميه | من يمسكه |
|----------|----------|----------|
| `clocks_mutex` | قائمة `clocks` العالمية | `clk_find_hw()`, `__clkdev_add()`, `clkdev_add_table()`, `clkdev_drop()` |

**ملاحظة مهمة**: الـ `lockdep_assert_held(&clocks_mutex)` في `clk_find()` تتحقق أن المتصل يمسك الـ mutex. هذا pattern دفاعي ممتاز.

---

## Phase 4: دليل الـ Debugging الشامل

### Software Level — Kernel Side

#### debugfs

```bash
# شجرة الـ clocks الكاملة
ls /sys/kernel/debug/clk/

# معلومات clock معينة
cat /sys/kernel/debug/clk/uart_clk/clk_rate
cat /sys/kernel/debug/clk/uart_clk/clk_enable_count
cat /sys/kernel/debug/clk/uart_clk/clk_prepare_count
cat /sys/kernel/debug/clk/uart_clk/clk_flags

# شجرة كاملة مع كل الـ rates
cat /sys/kernel/debug/clk/clk_summary

# مثال على output:
#    clock                        enable_cnt  prepare_cnt        rate   accuracy   phase
#    xtal24m                               1            1    24000000          0       0
#     pll_core                             1            1   816000000          0       0
#      armclk                              1            1   816000000          0       0
```

#### sysfs

```bash
# رؤية الـ clocks المرتبطة بـ device
ls /sys/bus/platform/devices/ff030000.uart/
# ابحث عن consumers في debugfs
grep -r "uart" /sys/kernel/debug/clk/ 2>/dev/null
```

#### ftrace / Tracepoints

```bash
# تفعيل أحداث الـ CLK
echo 1 > /sys/kernel/debug/tracing/events/clk/enable
echo 1 > /sys/kernel/debug/tracing/events/clk/disable
echo 1 > /sys/kernel/debug/tracing/events/clk/set_rate
echo 1 > /sys/kernel/debug/tracing/events/clk/enable

# بدء التتبع
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... شغّل الـ driver ...
cat /sys/kernel/debug/tracing/trace

# مثال على output:
# uart-probe-1234 [001] .... 12.345678: clk_enable: uart_clk
# uart-probe-1234 [001] .... 12.345679: clk_enable_complete: uart_clk
```

#### Dynamic Debug

```bash
# تفعيل debug messages للـ clk subsystem
echo "file clkdev.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file clk.c +p" > /sys/kernel/debug/dynamic_debug/control

# أو في kernel cmdline:
# dyndbg="file clkdev.c +p"
```

#### Kernel Config للـ Debugging

```
CONFIG_DEBUG_FS=y              # لازم لـ /sys/kernel/debug/clk/
CONFIG_COMMON_CLK=y            # الـ CCF نفسه
CONFIG_CLK_DEBUG=y             # debug info إضافية (إذا موجودة)
CONFIG_PROVE_LOCKING=y         # lockdep — يكتشف deadlocks
CONFIG_DEBUG_MUTEXES=y         # debug للـ mutex
CONFIG_DEBUG_SPINLOCK=y        # debug للـ spinlock
```

### رسائل الـ Error الشائعة وتفسيرها

| الرسالة | السبب | الحل |
|---------|-------|-------|
| `clk: couldn't get clock 0 for uart0` | الـ driver طلب clock بـ DT ومش موجودة | تحقق من الـ DT و `clocks` property |
| `con_id: connection ID is greater than 16` | اسم الـ connection أطول من `MAX_CON_ID` | قصّر اسم الـ connection أو زد `MAX_CON_ID` |
| `EPROBE_DEFER` | الـ clock provider ما اتسجّلش بعد | طبيعي — الـ kernel يعيد probe لاحقاً |
| `ENOENT` | مفيش clock lookup تطابق البحث | أضف `clkdev_create()` أو صحّح الـ DT |

### Hardware Level

#### تحقق من الـ Clock Registers مباشرة

```bash
# قراءة register مباشرة (مثال على RK3562)
# PLL Control Register at 0xFF760000
devmem2 0xFF760000 w    # قراءة 32-bit word

# أو باستخدام /dev/mem
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    mem = mmap.mmap(f.fileno(), 4096, offset=0xFF760000)
    val = struct.unpack('<I', mem.read(4))[0]
    print(f'PLL register: 0x{val:08X}')
"
```

#### io command (من busybox/util-linux)

```bash
# قراءة register
io -4 -r 0xFF760000

# كتابة register (احذر!)
io -4 -w 0xFF760000 0x12345678
```

#### إضافة Debug Prints مؤقتة

```c
/* في clk_find() لفهم الـ matching */
static struct clk_lookup *clk_find(const char *dev_id, const char *con_id)
{
    pr_debug("clk_find: looking for dev='%s' con='%s'\n",
             dev_id ?: "NULL", con_id ?: "NULL");

    list_for_each_entry(p, &clocks, node) {
        pr_debug("  checking: dev='%s' con='%s' hw=%p\n",
                 p->dev_id ?: "NULL", p->con_id ?: "NULL", p->clk_hw);
        /* ... */
    }
}
```

#### Device Tree Debugging

```bash
# تحقق من الـ DT بعد boot
dtc -I fs /proc/device-tree | grep -A5 "uart@"

# مثال على DT صحيح:
# uart0: serial@ff030000 {
#     clocks = <&cru SCLK_UART0>, <&cru PCLK_UART0>;
#     clock-names = "baudclk", "apb_pclk";
# };

# تحقق من أن الـ clock names تتطابق مع ما يطلبه الـ driver
grep -r "clk_get.*uart" drivers/tty/serial/
```

#### Practical Debug Script

```bash
#!/bin/bash
# script لتشخيص مشاكل الـ clock lookup

DEVICE=$1  # مثل "ff030000.serial"

echo "=== Clock Debug for: $DEVICE ==="
echo ""

echo "--- debugfs clk summary ---"
cat /sys/kernel/debug/clk/clk_summary 2>/dev/null | head -50

echo ""
echo "--- Device DT clocks ---"
if [ -d "/proc/device-tree/aliases" ]; then
    cat /proc/device-tree/aliases/* 2>/dev/null
fi

echo ""
echo "--- dmesg clock errors ---"
dmesg | grep -E "clk|clock" | grep -iE "err|fail|defer|not found"
```

---

## Phase 5: شرح الـ Functions

### المجموعة 1: البحث الداخلي (Internal Lookup)

#### `clk_find()`

```c
static struct clk_lookup *clk_find(const char *dev_id, const char *con_id)
```

**ما تفعله**: تبحث في القائمة العالمية `clocks` عن أفضل تطابق لـ (`dev_id`, `con_id`) باستخدام خوارزمية الـ fuzzy matching.

**المعاملات**:
- `dev_id`: اسم الـ device (مثل `"ff030000.serial"`)، أو NULL (wildcard)
- `con_id`: اسم الـ connection (مثل `"baudclk"`)، أو NULL (wildcard)

**القيمة المُرجعة**: أفضل `clk_lookup` وجدها، أو NULL لو مفيش.

**التفاصيل المهمة**:
- **يجب استدعاؤها وأنت ماسك `clocks_mutex`** — `lockdep_assert_held()` تتحقق من ده
- الـ `best_possible` يحسب الحد الأقصى للنقاط الممكنة (2 إذا في dev_id + 1 إذا في con_id)
- إذا وجدنا match بأعلى النقاط الممكنة → `break` فوراً (optimization)

**pseudocode**:
```
best_possible = (dev_id ? 2 : 0) + (con_id ? 1 : 0)
best_found = 0

for each p in clocks:
    match = 0
    if p.dev_id:
        if !dev_id or dev_id != p.dev_id → skip
        match += 2
    if p.con_id:
        if !con_id or con_id != p.con_id → skip
        match += 1
    if match > best_found:
        best = p
        if match == best_possible → break early
        best_found = match

return best
```

---

#### `clk_find_hw()`

```c
struct clk_hw *clk_find_hw(const char *dev_id, const char *con_id)
```

**ما تفعله**: واجهة عامة لـ `clk_find()` — تمسك الـ mutex وترجع الـ `clk_hw`.

**القيمة المُرجعة**: `clk_hw *` إذا وُجد، أو `ERR_PTR(-ENOENT)` إذا لا.

**من يستدعيها**: `__clk_get_sys()` وكل من يحتاج الـ hardware descriptor بدون إنشاء `clk` handle.

---

### المجموعة 2: الـ Consumer API (واجهة الـ Driver)

#### `clk_get_sys()`

```c
struct clk *clk_get_sys(const char *dev_id, const char *con_id)
```

**ما تفعله**: الحصول على `clk` handle بالاسم، بدون `struct device`.

**متى تُستخدم**: في الأكواد القديمة أو عندما ما في device object متاح. الأحدث هو `clk_get(dev, con_id)`.

**مُصدَّرة بـ**: `EXPORT_SYMBOL(clk_get_sys)` — متاحة للـ modules.

---

#### `clk_get()`

```c
struct clk *clk_get(struct device *dev, const char *con_id)
```

**ما تفعله**: الـ API الرئيسية للحصول على clock. تجرب الـ DT أولاً، ثم الـ clkdev.

**المعاملات**:
- `dev`: الـ device الذي يطلب الـ clock (يُستخدم اسمه وـ `of_node`)
- `con_id`: اسم الـ connection (مثل `"fclk"`)

**القيمة المُرجعة**: `struct clk *` على النجاح، أو `ERR_PTR(error)`.

**مهم**: إذا رجع `ERR_PTR(-EPROBE_DEFER)`، معناه الـ clock ما اتسجّلش بعد والـ driver لازم يعيد المحاولة لاحقاً.

**مثال استخدام في driver**:
```c
static int uart_probe(struct platform_device *pdev)
{
    struct clk *clk = clk_get(&pdev->dev, "baudclk");
    if (IS_ERR(clk))
        return PTR_ERR(clk);  /* EPROBE_DEFER يُرجَع تلقائياً */
    /* ... */
}
```

---

#### `clk_put()`

```c
void clk_put(struct clk *clk)
```

**ما تفعله**: تحرر الـ `clk` handle المُكتسَب بـ `clk_get()`. دائماً استدعِها في الـ `remove()` أو عند الخطأ.

---

### المجموعة 3: التسجيل في القائمة (Registration)

#### `__clkdev_add()` — static

```c
static void __clkdev_add(struct clk_lookup *cl)
```

**ما تفعله**: تضيف `cl` لنهاية القائمة `clocks` تحت الـ mutex. الـ internal helper لكل دوال الإضافة.

---

#### `clkdev_add()`

```c
void clkdev_add(struct clk_lookup *cl)
```

**ما تفعله**: تضيف `clk_lookup` ثابت (statically allocated) للقائمة. إذا `cl->clk_hw` فاضي، تأخذه من `cl->clk`.

**متى تُستخدم**: في الكود القديم مع `CLKDEV_INIT()` macro لتسجيل جداول ثابتة.

```c
/* مثال على الاستخدام */
static struct clk_lookup board_clocks[] = {
    CLKDEV_INIT("uart0", "fclk", &uart_clk),
    CLKDEV_INIT("spi0",  "clk",  &spi_clk),
};

/* في board init */
for (int i = 0; i < ARRAY_SIZE(board_clocks); i++)
    clkdev_add(&board_clocks[i]);
```

---

#### `clkdev_add_table()`

```c
void clkdev_add_table(struct clk_lookup *cl, size_t num)
```

**ما تفعله**: تضيف مصفوفة من الـ `clk_lookup` دفعة واحدة تحت lock واحد (أكفأ من استدعاء `clkdev_add` لكل عنصر).

---

#### `clkdev_drop()`

```c
void clkdev_drop(struct clk_lookup *cl)
```

**ما تفعله**: تحذف الـ `cl` من القائمة وتحرر الذاكرة. المقابل لـ `clkdev_create()`.

**مهم**: لا تستدعِها على `clk_lookup` ثابت (static)! فقط على ما أُنشئ بـ `clkdev_create()`.

---

### المجموعة 4: الإنشاء الديناميكي (Dynamic Allocation)

#### `vclkdev_alloc()` — static

```c
static __printf(3, 0) struct clk_lookup * __ref
vclkdev_alloc(struct clk_hw *hw, const char *con_id, const char *dev_fmt, va_list ap)
```

**ما تفعله**: تُخصص `clk_lookup_alloc` وتملأها. الـ `__ref` annotation تقول للـ linker أن هذه الدالة قد تُستخدم في early boot.

**تفاصيل مهمة**:
- إذا `con_id` أطول من `MAX_CON_ID=16` → تطبع error وتستبدله بـ `"bad"`
- إذا `dev_fmt` أطول من `MAX_DEV_ID=24` → نفس الشيء
- **لا ترجع NULL في حالة الخطأ** — ترجع entry بـ `"bad"` IDs لن يتطابق أبداً

#### `vclkdev_create()` — static

```c
static struct clk_lookup *vclkdev_create(struct clk_hw *hw, const char *con_id,
    const char *dev_fmt, va_list ap)
```

**ما تفعله**: تجمع `vclkdev_alloc()` + `__clkdev_add()`.

---

#### `clkdev_create()`

```c
struct clk_lookup *clkdev_create(struct clk *clk, const char *con_id,
    const char *dev_fmt, ...)
```

**ما تفعله**: إنشاء وتسجيل `clk_lookup` من `struct clk`.

**مُصدَّرة بـ**: `EXPORT_SYMBOL_GPL` — للـ modules الـ GPL فقط.

**مثال**:
```c
struct clk_lookup *lookup = clkdev_create(my_clk, "fclk", "uart.%d", 0);
/* الآن clk_get(dev, "fclk") للـ "uart.0" سيجد هذا الـ clock */
```

---

#### `clkdev_hw_create()`

```c
struct clk_lookup *clkdev_hw_create(struct clk_hw *hw, const char *con_id,
    const char *dev_fmt, ...)
```

**ما تفعله**: نفس `clkdev_create()` لكن من `struct clk_hw` مباشرة (الأحدث والمفضّل).

---

### المجموعة 5: التسجيل عبر APIs التقليدية

#### `clk_add_alias()`

```c
int clk_add_alias(const char *alias, const char *alias_dev_name,
    const char *con_id, struct device *dev)
```

**ما تفعله**: تأخذ clock موجودة وتُضيف لها اسم بديل (alias). مفيدة للـ backwards compatibility.

**المعاملات**:
- `alias`: الاسم الجديد للـ connection
- `alias_dev_name`: الـ device الذي سيستخدم الاسم الجديد (أو NULL للكل)
- `con_id`: الاسم الأصلي للـ connection
- `dev`: الـ device صاحب الـ clock الأصلية

**كيف تعمل داخلياً**:
```
clk_get(dev, con_id)         ← احصل على الـ clock الأصلية
    ↓
clkdev_create(clk, alias, alias_dev_name)  ← سجّلها باسم جديد
    ↓
clk_put(clk)                 ← حرّر الـ handle المؤقت
```

---

#### `clk_register_clkdev()` / `clk_hw_register_clkdev()`

```c
int clk_register_clkdev(struct clk *clk, const char *con_id, const char *dev_id)
int clk_hw_register_clkdev(struct clk_hw *hw, const char *con_id, const char *dev_id)
```

**ما تفعلان**: تسجيل clock lookup بأسلوب أبسط (dev_id كـ string مباشرة بدل format string).

**تفصيل مهم**: `do_clk_register_clkdev()` تتعامل مع حالة `dev_id=NULL` بشكل صحيح:
- إذا `dev_id` فيها قيمة → تمرر `"%s", dev_id`
- إذا `dev_id=NULL` → تمرر `NULL` مباشرة (wildcard)

---

### المجموعة 6: الـ Managed Resources (devm)

#### `devm_clk_hw_register_clkdev()`

```c
int devm_clk_hw_register_clkdev(struct device *dev, struct clk_hw *hw,
    const char *con_id, const char *dev_id)
```

**ما تفعله**: نفس `clk_hw_register_clkdev()` لكن مع **automatic cleanup** عند إزالة الـ device.

**كيف تعمل**: تسجّل `devm_clkdev_release()` كـ devres action. عند `device_del()`، الـ kernel يستدعي `clkdev_drop()` تلقائياً.

**لماذا مهمة**: تمنع الـ memory leaks في حالة فشل الـ probe أو عند إزالة الـ driver.

```c
/* ✅ الأفضل: devm تتكفل بالـ cleanup */
int my_clk_probe(struct platform_device *pdev)
{
    return devm_clk_hw_register_clkdev(&pdev->dev, hw, "fclk", NULL);
}
/* لا تحتاج remove() للـ clkdev! */

/* ❌ القديم: تحتاج cleanup يدوي */
static struct clk_lookup *my_lookup;
int my_clk_probe(struct platform_device *pdev)
{
    my_lookup = clkdev_hw_create(hw, "fclk", NULL);
    return my_lookup ? 0 : -ENOMEM;
}
void my_clk_remove(struct platform_device *pdev)
{
    clkdev_drop(my_lookup);  /* لازم تتذكر! */
}
```

---

## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: UART Driver على RK3562 يفشل بـ `-ENOENT`

**السياق**: شركة تصنع بوابة صناعية (Industrial Gateway) على Rockchip RK3562. الـ UART driver يشتغل على SoCs أخرى لكن على RK3562 الجديد ما بيشتغلش.

**المشكلة**:
```
[    2.345678] rk3562-uart ff030000.serial: could not get clock baudclk: -2
[    2.345679] ff030000.serial: probe failed: -2  (-ENOENT)
```

**التحليل**:
```bash
# الـ driver يطلب clock
# في drivers/tty/serial/8250/8250_dw.c:
clk = devm_clk_get(&pdev->dev, "baudclk");

# هذا يصل إلى clk_get():
# → of_clk_get_hw() تحاول تجد clock في الـ DT
# → تفشل بـ ENOENT (مش EPROBE_DEFER!)
# → تحاول __clk_get_sys() في القائمة clkdev
# → تفشل كذلك → ترجع ENOENT
```

```bash
# فحص الـ DT
dtc -I fs /proc/device-tree | grep -A 20 "ff030000"
# النتيجة:
# serial@ff030000 {
#     clocks = <&cru SCLK_UART0>;
#     clock-names = "baudclk";   ← هذا موجود!
# };

# إذاً الـ DT صحيح. هل الـ CRU driver اتسجّل؟
cat /sys/kernel/debug/clk/clk_summary | grep uart
# لا شيء! الـ CRU clock provider ما اتسجّلش بعد
```

**الحل**:
```bash
# فحص dmesg
dmesg | grep cru
# [    1.234] rockchip-cru ff760000.cru: failed to probe: -19
```

المشكلة في الـ CRU driver نفسه، مش في clkdev. لكن الخطأ المُرجَع خاطئ — لازم يكون `-EPROBE_DEFER` مش `-ENOENT`:

```c
/* في of_clk_get_hw() — kernel تتعامل مع هذا بشكل صحيح */
/* إذا الـ provider موجود في DT لكن ما اتسجّلش → EPROBE_DEFER */
/* إذا مش موجود أصلاً في DT → ENOENT */
```

```bash
# تحقق: هل الـ CRU في الـ DT؟
dtc -I fs /proc/device-tree | grep -A3 "ff760000"
# cru@ff760000 {
#     compatible = "rockchip,rk3562-cru";

# هل الـ CONFIG للـ CRU driver مفعّل؟
zcat /proc/config.gz | grep RK3562
# CONFIG_CLK_RK3562 is not set  ← المشكلة!
```

**الدرس المستفاد**: `-ENOENT` من `clk_get()` يعني إما الـ DT خاطئ أو الـ clock provider ما اتسجّلش. ابدأ بفحص `clk_summary` ثم `dmesg` للـ clock providers.

---

### السيناريو 2: SPI Controller على STM32MP1 يأخذ rate خاطئة

**السياق**: Device IoT على STM32MP157. الـ SPI تشتغل لكن الـ data تجي مشوّشة — السرعة ضعف المطلوب.

**المشكلة**:
```c
/* الـ driver يطلب */
clk_set_rate(spi_clk, 1000000);  /* 1 MHz */
/* الفعلي 2 MHz → بيانات خاطئة */
```

**التحليل**:
```bash
# افحص الـ clock rate الفعلية
cat /sys/kernel/debug/clk/spi4/clk_rate
# 2000000  ← ضعف المطلوب!

# افحص الـ parent
cat /sys/kernel/debug/clk/clk_summary | grep -A5 spi4
# spi4     1   1   2000000
#  spi4_ker_ck  1  1  64000000
```

```bash
# الـ driver يستخدم أي clock؟
# تفعيل ftrace
echo 1 > /sys/kernel/debug/tracing/events/clk/set_rate/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
# شغّل الـ SPI
cat /sys/kernel/debug/tracing/trace | grep spi
# spi-app [001] clk_set_rate: spi4_ker_ck 1000000  ← يحاول
# spi-app [001] clk_set_rate_complete: spi4_ker_ck 2000000 ← minimum!
```

المشكلة: الـ PLL ما يقدر يعطي أقل من 2 MHz لهذا الـ divider path.

```bash
# افحص الـ DT — هل في divider إضافي؟
dtc -I fs /proc/device-tree | grep -A30 "spi@5c010000"
# clock-names = "ck_spi", "pclk";
# ← الـ driver يستخدم "ck_spi" وليس "pclk"
```

**الحل**:
```c
/* في الـ board DTS */
&spi4 {
    clocks = <&rcc SPI4>, <&rcc PCLK5>;
    clock-names = "ck_spi", "pclk";
    /* أضف prescaler hint */
    assigned-clocks = <&rcc SPI4>;
    assigned-clock-rates = <8000000>;  /* اجبر rate أعلى يقسم نظيف */
};
```

**الدرس المستفاد**: `clk_set_rate()` قد لا يعطيك بالضبط ما تطلب — استخدم `clk_round_rate()` للتحقق أولاً.

---

### السيناريو 3: Memory Leak في Production على i.MX8MQ

**السياق**: Android TV Box على NXP i.MX8MQ. بعد أيام من التشغيل، الـ system يصير بطيء.

**المشكلة**:
```bash
cat /proc/meminfo | grep Slab
# Slab: 450 MB  ← ارتفع من 50MB!
```

**التحليل**:
```bash
# ابحث عن الـ slab leak
cat /proc/slabinfo | sort -k3 -nr | head -20
# kmalloc-64   500000   500000   64  ...
```

```bash
# استخدم kmemleak
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak | head -30
# unreferenced object 0xffff000012345678 (size 64):
#   backtrace:
#     clkdev_create+0x48
#     my_audio_codec_probe+0x8c
#     platform_probe+0x68
```

**السبب**: `clkdev_create()` تُستدعى في كل `probe` لكن لا يقابلها `clkdev_drop()`.

```c
/* ❌ الكود الخاطئ */
static int codec_probe(struct platform_device *pdev)
{
    clkdev_create(mclk, "mclk", "my-codec.%d", pdev->id);
    /* ← نسوا clkdev_drop في remove()! */
}

/* ✅ الإصلاح */
static int codec_probe(struct platform_device *pdev)
{
    return devm_clk_hw_register_clkdev(&pdev->dev, hw, "mclk", NULL);
    /* devm تتكفل بالـ cleanup تلقائياً */
}
```

**الدرس المستفاد**: استخدم دائماً `devm_clk_hw_register_clkdev()` بدلاً من `clkdev_create()` لتجنب الـ memory leaks.

---

### السيناريو 4: إضافة Clock Alias لـ Driver قديم على Allwinner H616

**السياق**: Android TV Box على Allwinner H616 (Orange Pi Zero 2). driver USB خاص (legacy) يطلب clock بالاسم القديم `"usb-phy-clk"` لكن الـ CCF يسجّلها باسم جديد.

**المشكلة**:
```
[    3.456] usb-phy-legacy: couldn't get clock usb-phy-clk
```

**التحليل**:
```bash
# الـ clock موجودة بأسم مختلف
cat /sys/kernel/debug/clk/clk_summary | grep usb
# usb_phy0_clk   1   1   48000000
```

**الحل**: إضافة alias بدون تعديل الـ driver.

```c
/* في board init أو clock driver */
static int h616_clk_probe(struct platform_device *pdev)
{
    /* ... تسجيل الـ clocks ... */

    /* إضافة alias للـ compatibility مع drivers قديمة */
    clk_add_alias("usb-phy-clk",   /* الاسم القديم */
                  NULL,             /* لأي device */
                  "usb_phy0_clk",  /* الاسم الجديد */
                  NULL);           /* من أي device */
    return 0;
}
```

```bash
# تحقق بعد الإصلاح
cat /sys/kernel/debug/clk/clk_summary | grep usb
# usb_phy0_clk  1   1   48000000
# ← الآن clk_get(dev, "usb-phy-clk") ستجده
```

**الدرس المستفاد**: `clk_add_alias()` ممتازة للـ backwards compatibility بدون تعديل drivers قديمة.

---

### السيناريو 5: Debugging `EPROBE_DEFER` متكرر على AM62x

**السياق**: Custom Board على Texas Instruments AM62x. أحد الـ drivers يعيد الـ probe باستمرار ولا يشتغل أبداً.

**المشكلة**:
```bash
dmesg | grep defer
# [   5.123] my-periph: probe with driver my-driver failed with error -517
# [   6.234] my-periph: probe with driver my-driver failed with error -517
# ...  (يتكرر إلى ما لا نهاية)
```

**التحليل**:
```bash
# افحص الـ DT للـ device
dtc -I fs /proc/device-tree | grep -A 10 "my-periph"
# clocks = <&k3_clks 136 0>;  ← يطلب clock من k3_clks

# هل الـ k3_clks provider اتسجّل؟
cat /sys/kernel/debug/clk/clk_summary | head -5
# ← فارغ! الـ clock provider ما بدأش

# افحص init order
dmesg | grep "k3-clk"
# لا شيء! الـ clock driver ما اشتغلش
```

```bash
# افحص الـ CONFIG
zcat /proc/config.gz | grep TI_K3_CLK
# CONFIG_TI_K3_CLOCK is not set  ← المشكلة!
```

**لكن في حالات أخرى**: الـ clock driver موجود لكن يحتاج `firmware` (مثل TI TISCI):

```bash
dmesg | grep tisci
# [    0.5] ti-sci firmware: waiting for sysfw  ← الـ firmware ما جاش
```

**الحل**:
```
1. تفعيل CONFIG_TI_K3_CLOCK في الـ kernel config
2. أو إصلاح الـ firmware loading
3. أو التحقق من الـ init order في الـ DT (probe ordering)
```

```bash
# للتشخيص السريع:
cat /sys/kernel/debug/devices_deferred
# my-periph: clock source not available  ← يعطيك السبب المباشر
```

**الدرس المستفاد**: `EPROBE_DEFER` من `clk_get()` يعني دائماً أن الـ clock provider موجود في الـ DT لكن ما اتسجّلش بعد. ابحث عن لماذا الـ provider فشل أو تأخر.

---

## Phase 7: مصادر ومراجع

### LWN.net Articles

| المقال | الرابط |
|--------|--------|
| The common clk framework (2012) | https://lwn.net/Articles/472998/ |
| Clock management in embedded Linux | https://lwn.net/Articles/574183/ |
| clkdev and clock lookup | https://lwn.net/Articles/472998/ |
| Devicetree and clocks | https://lwn.net/Articles/518085/ |

### توثيق الـ Kernel الرسمي

```
Documentation/driver-api/clk.rst       ← الدليل الرئيسي للـ CLK framework
Documentation/devicetree/bindings/clock/  ← DT bindings لكل SoC
Documentation/core-api/kobject.rst     ← للفهم الأعمق للـ device model
```

```bash
# قراءة التوثيق مباشرة
ls Documentation/driver-api/clk.rst
cat Documentation/driver-api/clk.rst
```

### الـ Commits المهمة

| التغيير | الوصف |
|---------|-------|
| commit `a2476e8` | إضافة الـ Common Clock Framework الأصلي |
| commit `fc8726a` | إضافة `clkdev_hw_create()` |
| commit `4143b08` | إضافة `devm_clk_hw_register_clkdev()` |

```bash
# للبحث في التاريخ
git log --oneline drivers/clk/clkdev.c
git log --oneline --grep="clkdev" drivers/clk/
```

### الملفات المرتبطة في الـ Kernel

```bash
# أهم الملفات للفهم الكامل
drivers/clk/clk.c              # الـ core framework
drivers/clk/clkdev.c           # هذا الملف
include/linux/clk.h            # consumer API
include/linux/clkdev.h         # clk_lookup struct
include/linux/clk-provider.h   # provider API
drivers/clk/clk-fixed-rate.c   # مثال بسيط على clock provider
drivers/clk/clk-divider.c      # divider clock
drivers/clk/clk-mux.c          # mux clock
drivers/clk/clk-gate.c         # gate (enable/disable) clock
```

### كتب مُوصى بها

| الكتاب | الفصل المهم |
|--------|-------------|
| **Linux Device Drivers, 3rd Ed (LDD3)** | Chapter 14: The Linux Device Model |
| **Linux Kernel Development (Robert Love)** | Chapter 17: Devices and Modules |
| **Embedded Linux Primer (2nd Ed)** | Chapter 15: Embedded Linux Porting |
| **Professional Linux Kernel Architecture** | Chapter 6: Device Drivers |

### مواقع مفيدة

- **elinux.org**: https://elinux.org/Common_Clock_Framework
- **bootlin.com slides**: https://bootlin.com/doc/training/linux-kernel/linux-kernel-slides.pdf (ابحث فيها عن "clk")
- **kernelnewbies**: https://kernelnewbies.org/Linux_3.4 (إعلان الـ CCF)
- **ARM DEN0022**: وثيقة PSCI التي تشرح clock management في ARM systems

### مصطلحات للبحث

```
"common clock framework linux"
"clkdev lookup linux kernel"
"struct clk_hw linux"
"of_clk_get linux device tree"
"EPROBE_DEFER clock"
"linux clock consumer driver"
"clk_lookup fuzzy match"
```

---

*تم توليد هذا التوثيق تلقائياً بواسطة Linux Kernel Arabic Documentation Skill*
*المصدر: `drivers/clk/clkdev.c` — Linux Kernel*
