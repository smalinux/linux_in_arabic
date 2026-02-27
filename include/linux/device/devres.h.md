## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي بيتبع له الملف

الملف `include/linux/device/devres.h` جزء من **Driver Core** — اللي اسمه الكامل في MAINTAINERS هو **DRIVER CORE, KOBJECTS, DEBUGFS AND SYSFS**، ومسؤول عنه Greg Kroah-Hartman و Rafael J. Wysocki. ده القلب اللي بيشغّل نموذج الأجهزة (device model) في Linux.

---

### المشكلة اللي الملف بيحلها — القصة الكاملة

تخيل إنك بتكتب **driver** لـ USB card. لما الـ driver بيتحمّل، بتعمل:

1. `kmalloc()` — تحجز ذاكرة للـ device state
2. `ioremap()` — تعمل map لـ hardware registers في الذاكرة
3. `request_irq()` — تطلب interrupt
4. `request_region()` — تحجز I/O ports

كل ده تمام. المشكلة بتيجي لما الـ driver بيفشل في الخطوة 3 مثلاً — لازم ترجع وتعمل `kfree()` و `iounmap()` يدوياً بالترتيب العكسي. ولو الـ device اتفصل فجأة؟ نفس الحكاية. ده بيخلي كود الـ **error handling** أكبر من الكود الأصلي نفسه، وسهل جداً ينسى حاجة ويحصل **resource leak**.

**الـ devres (Device Resource Management)** جه يحل الموضوع ده بالكامل.

---

### الفكرة الأساسية — ELI5

**تخيّل الـ device زي أوضة فندق.**

- لما بتدخل الأوضة (driver probe)، الفندق بيعطيك مفتاح، تاول، ريموت — كل ده مسجّل عندهم.
- لما بتسيب الأوضة (driver remove/detach)، الفندق بيجمع كل حاجة تلقائياً من غير ما تفكر في حاجة.
- مش محتاج تفضل تتذكر "رجّعت الريموت؟ عمّلت iounmap؟" — الفندق (الـ kernel) مسؤول عن ده.

**الـ devres** بيحفظ قايمة (linked list) لكل resource اتحجز مع كل device. لما الـ device بيتشال أو الـ driver بيفشل، الـ kernel بيمشي على القايمة دي بالترتيب العكسي وبينادي دالة الـ `release` الخاصة بكل resource تلقائياً.

---

### إيه اللي بيقدمه الملف ده تحديداً؟

الملف ده هو الـ **public API header** للـ devres system. بيعرّف نوعين من الـ API:

#### 1. الـ Low-level devres API (للـ framework developers)

| الدالة | الغرض |
|--------|--------|
| `devres_alloc()` | تحجز resource node بـ release callback |
| `devres_add()` | تضيف الـ resource لقايمة الـ device |
| `devres_find()` | تدوّر على resource موجود |
| `devres_remove()` | تشيل resource من القايمة من غير ما تحرّره |
| `devres_destroy()` | تشيل وتحرّر resource |
| `devres_release()` | تنادي الـ release callback يدوياً |

#### 2. الـ High-level devm_* API (للـ driver developers)

دي الـ API اللي كل driver بيستخدمها يومياً:

| الدالة | المقابل العادي | الغرض |
|--------|----------------|--------|
| `devm_kmalloc()` | `kmalloc()` | حجز ذاكرة مرتبطة بالـ device |
| `devm_kzalloc()` | `kzalloc()` | حجز ذاكرة مزيّرة بـ zero |
| `devm_kmalloc_array()` | `kmalloc_array()` | حجز array مع overflow check |
| `devm_kcalloc()` | `kcalloc()` | حجز array مزيّرة |
| `devm_krealloc()` | `krealloc()` | توسيع ذاكرة محجوزة |
| `devm_kfree()` | `kfree()` | تحرير مبكّر (اختياري) |
| `devm_kstrdup()` | `kstrdup()` | نسخ string |
| `devm_kasprintf()` | `kasprintf()` | إنشاء string بـ format |
| `devm_alloc_percpu()` | `alloc_percpu()` | حجز per-CPU memory |
| `devm_get_free_pages()` | `get_free_pages()` | حجز pages كاملة |
| `devm_ioremap_resource()` | `ioremap()` | map hardware registers |
| `devm_add_action()` | — | إضافة callback مخصص |

#### 3. الـ Groups

**الـ devres group** بيسمح بتجميع resources مع بعض تحت ID واحد، فتقدر تحرّرهم كمجموعة دفعة واحدة لو حاجة فشلت في النص.

---

### كيف بيشتغل من الداخل — الصورة التقنية

```
struct device
    └── devres_head (list_head)
            ├── devres_node → [release_fn | data: kmalloc buf]
            ├── devres_node → [release_fn | data: ioremap addr]
            ├── devres_group → [open_marker]
            │       ├── devres_node → [release_fn | data: irq]
            └── devres_group → [close_marker]

لما device بيتشال:
    kernel يمشي على القايمة بالعكس
    → ينادي release_fn لكل node
    → كل حاجة بتتحرر تلقائياً
```

الـ `struct devres` الفعلية متعرّفة في `drivers/base/devres.c` (مش في الـ header) عشان تخفي التفاصيل الداخلية.

---

### مثال حقيقي — قبل وبعد

**قبل devres (الطريقة القديمة):**

```c
static int my_probe(struct platform_device *pdev)
{
    void *buf;
    void __iomem *regs;
    int irq, ret;

    buf = kmalloc(1024, GFP_KERNEL);
    if (!buf)
        return -ENOMEM;

    regs = ioremap(res->start, resource_size(res));
    if (!regs) {
        ret = -ENOMEM;
        goto err_regs;  /* error path يدوي */
    }

    irq = request_irq(...);
    if (irq < 0) {
        ret = irq;
        goto err_irq;   /* error path تاني */
    }
    return 0;

err_irq:
    iounmap(regs);     /* لازم تتذكر كل ده */
err_regs:
    kfree(buf);
    return ret;
}
```

**بعد devres (الطريقة الحديثة):**

```c
static int my_probe(struct platform_device *pdev)
{
    /* لو أي حاجة فشلت، كل اللي اتحجز قبلها هيتحرر تلقائياً */
    void *buf = devm_kmalloc(&pdev->dev, 1024, GFP_KERNEL);
    if (!buf)
        return -ENOMEM;

    void __iomem *regs = devm_ioremap_resource(&pdev->dev, res);
    if (IS_ERR(regs))
        return PTR_ERR(regs);

    /* no error cleanup needed at all */
    return 0;
}
```

---

### الملفات المهمة اللي لازم تعرفها

| الملف | الغرض |
|-------|--------|
| `include/linux/device/devres.h` | الـ public API header — الملف ده |
| `drivers/base/devres.c` | التنفيذ الكامل للـ devres system |
| `include/linux/device.h` | يعمل include لـ devres.h ضمنياً عبر `include/linux/device/*.h` |
| `include/linux/devm-helpers.h` | helpers إضافية لـ devm patterns شائعة |
| `rust/kernel/devres.rs` | نفس الـ concept بس لـ Rust drivers |
| `drivers/base/` | كل الـ Driver Core implementation |
| `include/linux/device/driver.h` | الـ driver lifecycle اللي بيطلق devres cleanup |
| `include/linux/device/bus.h` | الـ bus model اللي بيتعامل مع probe/remove |

---

### ملخص الفكرة في جملة

**الـ devres** بيحوّل مسؤولية تحرير resources من الـ driver developer لـ kernel نفسه — بتسجّل الـ resource مرة واحدة، والـ kernel بيحرّرها تلقائياً لما الـ device يتشال أو الـ driver يفشل، من غير ما تكتب سطر cleanup واحد.
## Phase 2: شرح الـ devres (Device Resource Management) Framework

---

### المشكلة — ليه الـ framework ده موجود أصلاً؟

أي driver بيكتبه حد بيعمل حاجات زي كده في `probe()`:

```c
static int my_driver_probe(struct platform_device *pdev)
{
    struct my_priv *priv;
    void __iomem *base;
    int irq, ret;

    priv = kzalloc(sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    base = ioremap(res->start, resource_size(res));
    if (!base) {
        ret = -ENOMEM;
        goto err_free_priv;
    }

    irq = platform_get_irq(pdev, 0);
    ret = request_irq(irq, my_isr, 0, "my_dev", priv);
    if (ret)
        goto err_unmap;

    /* ... المزيد من الـ init ... */
    return 0;

err_unmap:
    iounmap(base);
err_free_priv:
    kfree(priv);
    return ret;
}

static int my_driver_remove(struct platform_device *pdev)
{
    /* lazim tani kol 7aga fe 3aks el-tarteb */
    free_irq(irq, priv);
    iounmap(base);
    kfree(priv);
    return 0;
}
```

**المشاكل الحقيقية هنا:**

| المشكلة | التفاصيل |
|---------|---------|
| **Goto hell** | كل resource محتاج `goto` label في الـ error path |
| **Mirror code** | الـ `remove()` هو نسخة عكسية حرفية من الـ `probe()` |
| **Resource leak** | لو فضيت goto label بالغلط أو نسيت حاجة في `remove()` — leak |
| **Ordering bugs** | لازم تحرر بالعكس بالظبط — أي خطأ في الترتيب = corruption أو crash |
| **Maintenance nightmare** | لما تضيف resource جديد لازم تعدل في 3 أماكن: alloc + error path + remove |

---

### الحل — الـ devres Approach

**الـ devres framework** بيحل المشكلة دي بـ**ربط كل resource بالـ device نفسه** مع **callback لـ cleanup** بيتحفظ جنبه.

لما الـ device بيتـ detach (أو بيفشل الـ probe)، الـ kernel بينفذ كل الـ callbacks دي تلقائياً بالترتيب العكسي.

نفس الـ driver بعد الـ devres:

```c
static int my_driver_probe(struct platform_device *pdev)
{
    struct my_priv *priv;
    void __iomem *base;
    int irq, ret;

    /* devm_ versions — auto-freed on detach */
    priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    base = devm_ioremap_resource(&pdev->dev, res);
    if (IS_ERR(base))
        return PTR_ERR(base);

    irq = platform_get_irq(pdev, 0);
    ret = devm_request_irq(&pdev->dev, irq, my_isr, 0, "my_dev", priv);
    if (ret)
        return ret;

    return 0;
    /* NO goto, NO remove() needed */
}
/* remove() function is optional or empty */
```

---

### الـ Analogy الحقيقي — فندق مع Checkout System

تخيل إنك بتنزل في **فندق** (= الـ device).

لما بتـ check-in، الـ فندق بيديلك:
- غرفة (= memory)
- مفتاح الـ pool (= MMIO mapping)
- parking spot (= IRQ line)

الفندق بيكتب في **دفتر الـ Checkout** (= `dev->devres_head` linked list):
- اسم الـ resource
- callback لازم يتنفذ لما تمشي

لما بتـ check-out (= driver detach):
- الفندق بيفتح الدفتر
- بيمشي من آخر entry لأول entry (LIFO)
- بينفذ كل callback تلقائياً

**المطابقة الدقيقة:**

| الفندق | الـ devres |
|--------|-----------|
| دفتر الـ Checkout | `dev->devres_head` (linked list) |
| كل سطر في الدفتر | `struct devres_node` |
| اسم الـ resource | `node->name` |
| callback الـ checkout | `node->release` (function pointer) |
| فعل الـ check-out | `devres_release_all()` عند detach |
| إضافة resource | `devres_add()` |
| غرفة VIP متجمعة | `devres_group` — set of resources released together |
| ضيف يسحب resource قبل checkout | `devres_remove()` + `devres_destroy()` |

لو الـ guest (driver) راح يعمل check-in ومتقدرش تكمل (probe failed) — الفندق لسه بينفذ الـ checkout لكل اللي اتسجل. ده بالظبط الـ cleanup-on-probe-failure behavior.

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Driver Code                              │
│                                                                 │
│  probe():                                                       │
│    devm_kzalloc()         devm_ioremap_resource()               │
│    devm_request_irq()     devm_clk_get()                        │
│    devm_add_action()      devm_regulator_get()                  │
└────────────────────┬────────────────────────────────────────────┘
                     │ كل devm_ call بتعمل
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                    devres Layer                                  │
│                                                                  │
│  __devres_alloc_node()  →  alloc_dr()  →  kmalloc_node()        │
│                                                                  │
│  devres_add()  →  list_add_tail(&dr->node.entry,                │
│                                 &dev->devres_head)              │
└────────────────────────────────┬────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                    struct device                                 │
│                                                                  │
│   devres_head ──► [devres_node] ──► [devres_node] ──► ...       │
│                    release=kfree    release=iounmap              │
│                    data=[memory]    data=[ioaddr]                │
│                                                                  │
│   devres_lock  (spinlock protecting the list)                   │
└────────────────────────────────┬────────────────────────────────┘
                                 │ عند driver detach
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│              devres_release_all()  [drivers/base/devres.c]      │
│                                                                  │
│  يمشي على الـ list من الآخر للأول (LIFO)                        │
│  لكل node: node->release(dev, dr->data)                         │
│  ثم kfree(dr)                                                    │
└─────────────────────────────────────────────────────────────────┘
```

---

### الـ Core Data Structures — كيف تتربط مع بعض

```
  struct device
  ┌──────────────────────┐
  │  devres_head         │  ← list_head — entry point للـ linked list
  │  devres_lock         │  ← spinlock  — thread-safe access
  └──────────┬───────────┘
             │
             ▼
  ┌──────────────────────────────────────────────────────┐
  │  struct devres  (allocated by alloc_dr)              │
  │                                                      │
  │  ┌─────────────────────────────┐                    │
  │  │  struct devres_node         │                    │
  │  │  ┌──────────────────────┐  │                    │
  │  │  │  entry (list_head)   │  │  ← الربط بالـ list  │
  │  │  │  release (fn ptr)    │  │  ← callback الـ free │
  │  │  │  name   (const char*)│  │  ← debug info       │
  │  │  │  size   (size_t)     │  │  ← debug info       │
  │  │  └──────────────────────┘  │                    │
  │  └─────────────────────────────┘                    │
  │                                                      │
  │  data[0]  ← ARCH_DMA_MINALIGN aligned                │
  │  ┌──────────────────────────────────────────────┐   │
  │  │  الـ actual resource data                     │   │
  │  │  (مثلاً: pointer للـ mapped IO address)       │   │
  │  └──────────────────────────────────────────────┘   │
  └──────────────────────────────────────────────────────┘
```

**نقطة مهمة جداً:** لما `devres_alloc()` بترجع `void *`، بترجع `dr->data` مش `dr` نفسه. يعني الـ driver بيشوف بس الـ data — الـ metadata (node) محجوب خلف الـ pointer.

لما محتاجين نوصل للـ node من الـ data:
```c
struct devres *dr = container_of(res, struct devres, data);
```

---

### الـ devres_group — Grouped Release

الـ framework بيدعم **grouping** — تقدر تجمع مجموعة resources مع بعض تحت group واحدة وترجع الكل مرة واحدة.

```
  dev->devres_head:

  [node]──►[group_open]──►[nodeA]──►[nodeB]──►[nodeC]──►[group_close]──►[node]
                           ─────────────────────────────
                                  GROUP boundary
```

**struct devres_group:**
```c
struct devres_group {
    struct devres_node  node[2];  // node[0]=open marker, node[1]=close marker
    void               *id;       // unique ID للـ group
    int                 color;    // internal state
};
```

الـ group بيشتغل زي stack frame:
- `devres_open_group()` — بيدفع open marker على الـ list
- `devres_close_group()` — بيدفع close marker
- `devres_release_group()` — بيحرر كل اللي بين الـ markers
- `devres_remove_group()` — بيشيل الـ markers بس من غير ما يحرر الـ resources

**Use case حقيقي:** في `probe()` بتعت SoC driver معقد، لو الـ phase الثانية من الـ init فشلت، تحرر resources الـ phase دي بس من غير ما تمس الـ phase الأولى.

---

### الـ devm_add_action — أقوى أداة في الـ framework

بدل ما تكتب release function كاملة، تقدر تسجل **arbitrary callback**:

```c
static void my_hw_disable(void *data)
{
    struct my_priv *priv = data;
    writel(0, priv->base + HW_ENABLE_REG);  // turn off hardware
}

static int my_probe(struct platform_device *pdev)
{
    /* ... setup ... */
    writel(1, priv->base + HW_ENABLE_REG);  // turn on hardware

    /* register cleanup action */
    return devm_add_action_or_reset(&pdev->dev, my_hw_disable, priv);
}
```

**الـ `devm_add_action_or_reset` pattern:**
```c
static inline int __devm_add_action_or_reset(struct device *dev,
                                              void (*action)(void *),
                                              void *data, const char *name)
{
    int ret;
    ret = __devm_add_action(dev, action, data, name);
    if (ret)
        action(data);  // لو فشل الـ register — نفذ الـ action فوراً
    return ret;
}
```

الفكرة: لو مقدرتش تسجل الـ cleanup — نفذه دلوقتي عشان متحصلش leak.

---

### الـ devm_ioremap_resource — مثال كامل على الـ devres pattern

```c
void __iomem *devm_ioremap_resource(struct device *dev,
                                     const struct resource *res)
{
    void __iomem *dest_ptr;
    void **ptr;

    /* 1. allocate devres entry to hold the mapped pointer */
    ptr = devres_alloc(devm_ioremap_release, sizeof(*ptr), GFP_KERNEL);
    if (!ptr)
        return IOMEM_ERR_PTR(-ENOMEM);

    /* 2. do the actual ioremap */
    dest_ptr = ioremap(res->start, resource_size(res));
    if (!dest_ptr) {
        devres_free(ptr);
        return IOMEM_ERR_PTR(-ENOMEM);
    }

    /* 3. store the pointer in the devres data area */
    *ptr = (__force void *)dest_ptr;

    /* 4. register with device — from here, auto-cleanup is active */
    devres_add(dev, ptr);

    return dest_ptr;
}
```

الـ release function:
```c
static void devm_ioremap_release(struct device *dev, void *res)
{
    iounmap(*(void __iomem **)res);  // res هو ptr اللي اتحفظ في data[]
}
```

**Pattern الـ 4 خطوات ده هو نفسه في كل devm_ function:**
1. `devres_alloc()` — allocate metadata + data
2. الـ actual resource acquisition
3. حفظ الـ handle في `data[]`
4. `devres_add()` — activate auto-cleanup

---

### Memory Layout — كيف الـ kernel بيحسب الـ size

```c
static bool check_dr_size(size_t size, size_t *tot_size)
{
    // tot_size = sizeof(struct devres) + size
    if (unlikely(check_add_overflow(sizeof(struct devres), size, tot_size)))
        return false;

    // round up لأقرب kmalloc bucket
    *tot_size = kmalloc_size_roundup(*tot_size);
    return true;
}
```

```
  kmalloc allocation:
  ┌────────────────────┬──────────────────────────────────────┐
  │  struct devres     │  data[]                              │
  │  (node + metadata) │  (الـ actual resource data)           │
  │  sizeof=X bytes    │  size bytes (requested)              │
  └────────────────────┴──────────────────────────────────────┘
  ◄─────────────────── tot_size (rounded to kmalloc bucket) ──►
```

الـ `data[]` بيتعمله align على `ARCH_DMA_MINALIGN` عشان الـ DMA operations على بعض الـ architectures محتاجة alignment أكبر من الـ natural alignment.

---

### Thread Safety — الـ devres_lock

كل access على `dev->devres_head` بيتحمى بـ **spinlock** (`dev->devres_lock`):

```c
void devres_add(struct device *dev, void *res)
{
    struct devres *dr = container_of(res, struct devres, data);
    unsigned long flags;

    spin_lock_irqsave(&dev->devres_lock, flags);
    add_dr(dev, &dr->node);
    spin_unlock_irqrestore(&dev->devres_lock, flags);
}
```

**لماذا `spin_lock_irqsave` وليس `mutex`؟**
لأن `devres_add()` ممكن تتنادى من interrupt context (مثلاً من ISR بيـ allocate resource لنفسه) أو من contexts مختلفة. الـ spinlock مع `irqsave` بيمنع deadlock لو الـ interrupt جاء وهو على نفس الـ CPU وهو شايل الـ lock.

---

### ما بيملكه الـ devres vs ما بيفوضه للـ Driver

| الـ devres يملكه | الـ driver مسؤول عنه |
|-----------------|---------------------|
| الـ linked list على كل device | اختيار الـ `release` callback |
| الـ lifetime tracking (متى تحرر) | الـ actual resource allocation logic |
| الـ LIFO ordering عند cleanup | الـ actual resource release logic داخل callback |
| الـ thread-safe access للـ list | الـ match logic لو محتاج find/remove |
| الـ group boundaries | متى تعمل open/close group |
| الـ debug info (name, size) | اسم الـ resource اللي بيديه للـ `#release` macro |

---

### الـ devres و subsystems تانية محتاجة تعرفها

**الـ Driver Model** (`include/linux/device.h`):
- الـ `struct device` هي container الـ devres — كل resource بيتربط بـ device object
- بدون فهم الـ driver model، مش هتعرف إمتى بيتنادى `devres_release_all()` (عند `device_del()`)

**الـ Memory Allocator** (`include/linux/slab.h`):
- الـ devres نفسه بيستخدم `kmalloc` لحفظ الـ metadata
- فهم `gfp_t` flags ضروري لاختيار `GFP_KERNEL` vs `GFP_ATOMIC`

**الـ NUMA** (`include/linux/numa.h`):
- الـ `devres_alloc_node()` بتسمح بتحديد NUMA node — مهم للـ performance على multi-socket servers

---

### الـ DEBUG_DEVRES

لو enable إلته:
```bash
# kernel boot parameter
devres.log=1
```

أو:
```bash
echo 1 > /sys/module/drivers_base_devres/parameters/log  # if exposed
```

هتشوف output زي:
```
my_device: DEVRES ADD 00000000abcd1234 devm_kzalloc (128 bytes)
my_device: DEVRES ADD 00000000efgh5678 devm_ioremap_release (8 bytes)
my_device: DEVRES REL 00000000efgh5678 devm_ioremap_release (8 bytes)
my_device: DEVRES REL 00000000abcd1234 devm_kzalloc (128 bytes)
```

لاحظ الـ LIFO: آخر اللي اتضاف، أول اللي اتحرر.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options

#### GFP Flags المستخدمة مع devres

| Flag | المعنى |
|------|--------|
| `GFP_KERNEL` | allocation عادية، ممكن تـ sleep |
| `GFP_ATOMIC` | allocation في interrupt context، مش ممكن تـ sleep |
| `__GFP_ZERO` | صفّر الـ memory بعد الـ allocation |
| `NUMA_NO_NODE` | اختار أي NUMA node متاح |

#### Config Options

| Option | الأثر |
|--------|-------|
| `CONFIG_DEBUG_DEVRES` | يفعّل logging لكل عملية add/remove/release، مفيد جداً للـ debugging |
| `CONFIG_HAS_IOMEM` | يفعّل `devm_ioremap_resource` و`devm_of_iomap`، لو مش موجود يرجعوا `IOMEM_ERR_PTR(-EINVAL)` |

#### Typedefs أساسية

| Type | التعريف | الاستخدام |
|------|---------|-----------|
| `dr_release_t` | `void (*)(struct device *dev, void *res)` | callback لتحرير الـ resource عند الـ detach |
| `dr_match_t` | `int (*)(struct device *dev, void *res, void *match_data)` | callback للبحث عن resource معين |

---

### الـ Structs المهمة

#### 1. `struct devres_node`

**الغرض:** الـ header الأساسي لأي resource مُدار — يحتوي على كل المعلومات اللي يحتاجها الـ framework لإدارة الـ resource.

```c
struct devres_node {
    struct list_head  entry;    /* ربط الـ node في قائمة dev->devres_head */
    dr_release_t      release;  /* الـ callback اللي يتنادى عند التحرير */
    const char       *name;     /* اسم الـ resource (debug فقط) */
    size_t            size;     /* حجم الـ data (debug فقط) */
};
```

- **`entry`**: بيربط الـ node في الـ doubly-linked list الخاصة بالـ device (`dev->devres_head`).
- **`release`**: بتتحدد وقت الـ allocation، وبتتنادى تلقائياً عند driver detach أو `devres_release()`.
- **`name` / `size`**: بيتملوا بس لما `CONFIG_DEBUG_DEVRES` مفعّل، أو بـ `set_node_dbginfo()`.

---

#### 2. `struct devres`

**الغرض:** الـ wrapper الكامل لأي resource مُدار — بيجمع الـ metadata (الـ node) مع الـ data الفعلية اللي طلبها الـ driver.

```c
struct devres {
    struct devres_node  node;  /* metadata + list linkage */
    /* الـ data الفعلية جاية بعدها مباشرةً في الـ memory */
    u8 __aligned(ARCH_DMA_MINALIGN) data[];  /* flexible array */
};
```

- **`data[]`**: flexible array member — الـ memory بتُخصَّص بـ `kmalloc` بحجم `sizeof(devres) + size`، والـ alignment بيضمن صلاحيتها للـ DMA.
- الـ API بيخفي الـ struct ده عن الـ driver، اللي بيشوف بس الـ pointer لـ `data`.

---

#### 3. `struct devres_group`

**الغرض:** بيعمل "grouping" للـ resources عشان تتحرر مع بعض كوحدة واحدة — زي الـ transaction في الـ DB.

```c
struct devres_group {
    struct devres_node  node[2];  /* [0] = open marker, [1] = close marker */
    void               *id;       /* مُعرِّف الـ group */
    int                 color;    /* مؤقت يُستخدم أثناء remove_nodes() */
};
```

- **`node[0]`**: الـ opening marker، بيتضاف في قائمة الـ device عند `devres_open_group()`.
- **`node[1]`**: الـ closing marker، بيتضاف عند `devres_close_group()`، ولو الـ group لسه مفتوح يبقى `list_empty`.
- **`id`**: لو الـ caller بعت `NULL` فبيبقى `grp` نفسه (unique pointer)، لو بعت قيمة بتتخزن هنا.
- **`color`**: بيتستخدم فقط جوا `remove_nodes()` عشان يحدد هل الـ group كاملة جوا النطاق المحدد ولا لأ.

---

#### 4. `struct action_devres`

**الغرض:** بيغلّف أي function call عادية عشان تتنفّذ تلقائياً وقت الـ teardown.

```c
struct action_devres {
    void *data;            /* الـ argument اللي هيتبعت للـ action */
    void (*action)(void *); /* الـ function نفسها */
};
```

- مثال: `devm_add_action(dev, my_cleanup_fn, my_data)` — هيخزّن `my_cleanup_fn` و `my_data` هنا.
- عند الـ teardown بيتنادى `devm_action_release()` اللي بينادي `devres->action(devres->data)`.

---

### مخطط العلاقات بين الـ Structs

```
struct device
  ├── devres_head: struct list_head  ◄───────────────────────────────────┐
  └── devres_lock: spinlock_t                                             │
                                                                          │
                                                                          │
struct devres_node          ┌─────────────────────────────────────────────┘
  ├── entry: list_head ─────┘   (linked into dev->devres_head)
  ├── release: dr_release_t
  ├── name: const char *
  └── size: size_t
       ▲
       │ embedded as first field
       │
struct devres
  ├── node: struct devres_node       ← الـ header
  └── data[]: u8 __aligned(...)     ← الـ memory الفعلية (driver data)
       ▲
       └──── الـ driver بيشوف cـ pointer لهنا بس

struct devres_group
  ├── node[0]: struct devres_node   ← open marker  (release = group_open_release)
  ├── node[1]: struct devres_node   ← close marker (release = group_close_release)
  ├── id: void *
  └── color: int

struct action_devres
  ├── data: void *
  └── action: void (*)(void *)
       └── يُغلَّف داخل struct devres عند __devm_add_action()
```

---

### مخطط الـ Memory Layout

```
kmalloc result:
┌────────────────────────────────────────────────────┐
│  struct devres_node                                 │
│    entry.next ──────────────────────────────────► next devres_node
│    entry.prev ◄──────────────────────────────────  prev devres_node
│    release    → my_release_fn()                    │
│    name       → "devm_kmalloc"                     │
│    size       = 64                                 │
├────────────────────────────────────────────────────┤  ← &devres.data[0]
│  data[0..63]                                        │    هذا الـ pointer
│  (الـ memory الفعلية اللي طلبها الـ driver)         │    بيرجعه الـ API
└────────────────────────────────────────────────────┘
```

---

### مخطط الـ Lifecycle (Creation → Registration → Usage → Teardown)

```
Driver Code                    devres Core                    Kernel
──────────────────────────────────────────────────────────────────────

1. ALLOCATION
driver_probe()
  │
  ├─► devm_kmalloc(dev, 64, GFP_KERNEL)
  │       │
  │       ├─► alloc_dr(devm_kmalloc_release, 64, GFP_KERNEL, NUMA_NO_NODE)
  │       │       │
  │       │       └─► kmalloc(sizeof(devres) + 64)  ──► struct devres في الـ heap
  │       │
  │       └─► returns dr->data  (الـ pointer للـ data فقط)
  │
  │   *** الـ resource لسه مش مربوط بالـ device ***
  │
2. REGISTRATION
  │
  ├─► devres_add(dev, res)
  │       │
  │       ├─► spin_lock_irqsave(&dev->devres_lock, flags)
  │       ├─► list_add_tail(&dr->node.entry, &dev->devres_head)
  │       └─► spin_unlock_irqrestore(...)
  │
  │   *** دلوقتي الـ resource مربوط بالـ device ***
  │
3. USAGE
  │
  └─► يستخدم الـ memory عادي...


4. TEARDOWN (عند driver detach أو device removal)
                               │
                               ├─► devres_release_all(dev)
                               │       │
                               │       ├─► spin_lock_irqsave(...)
                               │       ├─► remove_nodes(dev, ...) → todo list
                               │       ├─► spin_unlock_irqrestore(...)
                               │       │
                               │       └─► release_nodes(dev, &todo)
                               │               │
                               │               └─► for each devres (LIFO order):
                               │                       dr->node.release(dev, dr->data)
                               │                       kfree(dr)
```

---

### مخطط الـ Group Lifecycle

```
dev->devres_head قبل:
┌──────────────────────────────────────────────────────────────────┐
│  HEAD ←→ [res_A] ←→ [res_B]                                     │
└──────────────────────────────────────────────────────────────────┘

بعد devres_open_group(dev, NULL, GFP_KERNEL):
┌──────────────────────────────────────────────────────────────────┐
│  HEAD ←→ [res_A] ←→ [res_B] ←→ [grp_OPEN]                      │
└──────────────────────────────────────────────────────────────────┘

بعد إضافة resources جديدة:
┌──────────────────────────────────────────────────────────────────┐
│  HEAD ←→ [res_A] ←→ [res_B] ←→ [grp_OPEN] ←→ [res_C] ←→ [res_D]│
└──────────────────────────────────────────────────────────────────┘

بعد devres_close_group():
┌──────────────────────────────────────────────────────────────────┐
│  HEAD ←→ [res_A] ←→ [res_B] ←→ [grp_OPEN] ←→ [res_C] ←→ [res_D] ←→ [grp_CLOSE] │
└──────────────────────────────────────────────────────────────────┘

بعد devres_release_group() → يحرر res_C و res_D ويمسح الـ markers:
┌──────────────────────────────────────────────────────────────────┐
│  HEAD ←→ [res_A] ←→ [res_B]                                     │
└──────────────────────────────────────────────────────────────────┘
```

---

### مخططات الـ Call Flow

#### `devm_kmalloc()` — الـ flow الكامل

```
driver calls devm_kmalloc(dev, 64, GFP_KERNEL)
  │
  └─► devm_kmalloc(dev, size, gfp)             [devres.c]
        │
        ├─► alloc_dr(devm_kmalloc_release, size, gfp, NUMA_NO_NODE)
        │     │
        │     ├─► check_dr_size(size, &tot_size)
        │     │     └─► check_add_overflow(sizeof(devres), size, &tot_size)
        │     │
        │     ├─► kmalloc_node_track_caller(tot_size, gfp, nid)
        │     │     └─► slab allocator ──► physical memory
        │     │
        │     ├─► INIT_LIST_HEAD(&dr->node.entry)
        │     └─► dr->node.release = devm_kmalloc_release
        │
        └─► devres_add(dev, dr->data)
              │
              ├─► spin_lock_irqsave(&dev->devres_lock, flags)
              ├─► list_add_tail(&dr->node.entry, &dev->devres_head)
              └─► spin_unlock_irqrestore(...)

returns: dr->data   (pointer للـ memory الفعلية)
```

#### `devm_kfree()` — التحرير اليدوي

```
driver calls devm_kfree(dev, ptr)
  │
  └─► devres_destroy(dev, devm_kmalloc_release, devm_kmalloc_match, ptr)
        │
        ├─► devres_remove(dev, ...)
        │     │
        │     ├─► spin_lock_irqsave(...)
        │     ├─► find_dr(dev, release, match, match_data)
        │     │     └─► list_for_each_entry_reverse(node, &dev->devres_head, entry)
        │     │           → node->release == devm_kmalloc_release?
        │     │           → devm_kmalloc_match(dev, dr->data, ptr) → res == data?
        │     ├─► list_del_init(&dr->node.entry)
        │     └─► spin_unlock_irqrestore(...)
        │
        └─► devres_free(res)
              └─► kfree(container_of(res, struct devres, data))
```

#### `devres_release_all()` — عند الـ detach

```
device_release_driver()
  │
  └─► devres_release_all(dev)
        │
        ├─► spin_lock_irqsave(&dev->devres_lock, flags)
        ├─► remove_nodes(dev, dev->devres_head.next, &dev->devres_head, &todo)
        │     │
        │     ├─► Pass 1: ينقل الـ normal devres entries لـ todo list
        │     │           ويصفّر colors الـ groups
        │     │
        │     └─► Pass 2: لو فيه groups، يلوّنها
        │                 grp->color == 2 → الـ group كاملة في النطاق → تنقل لـ todo
        │
        ├─► spin_unlock_irqrestore(...)
        │
        └─► release_nodes(dev, &todo)
              │
              └─► list_for_each_entry_safe_reverse(dr, tmp, todo, node.entry)
                    │                              ↑ LIFO — عكس ترتيب الإضافة
                    ├─► dr->node.release(dev, dr->data)  ← الـ cleanup callback
                    └─► kfree(dr)
```

#### `devm_add_action()` — إضافة custom cleanup

```
driver calls devm_add_action(dev, my_cleanup, my_data)
  │
  └─► __devm_add_action(dev, my_cleanup, my_data, "my_cleanup")
        │
        ├─► __devres_alloc_node(devm_action_release, sizeof(action_devres), ...)
        │     └─► يخلق struct devres يحتوي struct action_devres في data[]
        │
        ├─► devres->action = my_cleanup
        ├─► devres->data   = my_data
        │
        └─► devres_add(dev, devres)

عند الـ teardown:
  └─► devm_action_release(dev, res)
        └─► struct action_devres *devres = res
              └─► devres->action(devres->data)
                    └─► my_cleanup(my_data)   ← الـ driver code
```

---

### استراتيجية الـ Locking

#### الـ Lock المستخدم

**الـ `dev->devres_lock`** هو `spinlock_t` جوا `struct device`، وده هو اللي بيحمي كل العمليات على `dev->devres_head`.

#### جدول ملخص

| العملية | الـ Lock مطلوب؟ | نوع الـ Lock | السبب |
|---------|----------------|-------------|-------|
| `devres_add()` | نعم | `spin_lock_irqsave` | الـ list modification |
| `devres_find()` | نعم | `spin_lock_irqsave` | الـ list traversal |
| `devres_get()` | نعم | `spin_lock_irqsave` | find + add atomic |
| `devres_remove()` | نعم | `spin_lock_irqsave` | الـ list deletion |
| `devres_release_all()` | نعم + بعدين يشيله | `spin_lock_irqsave` | remove_nodes بس |
| `release_nodes()` | لا | — | بعد ما الـ nodes اتنقلت لـ todo |
| `kfree(dr)` | لا | — | بعد الإزالة من الـ list |

#### اهتمامات مهمة في الـ Locking

**1. `irqsave` مش `irq` عادي:**
كل الـ locks بتستخدم `spin_lock_irqsave` / `spin_unlock_irqrestore` عشان الـ release callbacks ممكن تتنادى من interrupt context في بعض الـ subsystems.

**2. الـ Release خارج الـ Lock:**
```
spin_lock_irqsave(...)
    remove_nodes(...)   ← ينقل الـ nodes لـ todo list
spin_unlock_irqrestore(...)

release_nodes(...)      ← ينادي release callbacks بدون lock
```
ده pattern مهم جداً — الـ release callbacks ممكن تـ sleep أو تعمل allocations، فلازم تتنادى برا الـ spinlock.

**3. الـ `devres_get()` atomic:**
```c
spin_lock_irqsave(&dev->devres_lock, flags);
dr = find_dr(...);        /* البحث */
if (!dr) {
    add_dr(dev, new_dr);  /* الإضافة */
    dr = new_dr;
    new_res = NULL;
}
spin_unlock_irqrestore(&dev->devres_lock, flags);
devres_free(new_res);     /* التحرير برا الـ lock */
```
الـ find والـ add في lock واحد = لا race condition ممكنة.

**4. Lock Ordering:**
مفيش nested locking جوا devres core — الـ `dev->devres_lock` بياخده مرة واحدة بس. الـ callbacks بتتنادى دايماً بعد ما الـ lock يتحرر.

```
dev->devres_lock
  └── (لا locks تانية جوا) ← بيمنع deadlock
```

**5. الـ `devres_for_each_res()` — تحذير:**
```c
spin_lock_irqsave(&dev->devres_lock, flags);
list_for_each_entry_safe_reverse(node, tmp, &dev->devres_head, entry) {
    fn(dev, dr->data, data);  /* الـ callback بيتنادى جوا الـ lock! */
}
spin_unlock_irqrestore(&dev->devres_lock, flags);
```
هنا الـ callback `fn` بيتنادى وهو شايل الـ lock — لازم الـ caller يعرف ده ومايعملش حاجة ممكن تسبب deadlock جوا الـ callback.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Category 1: Core devres Primitives

| Function | Signature (مختصرة) | الغرض |
|---|---|---|
| `__devres_alloc_node` | `(release, size, gfp, nid, name) → void*` | allocate devres resource بـ NUMA node |
| `devres_alloc` | macro → `__devres_alloc_node` | allocate devres على أي node |
| `devres_alloc_node` | macro → `__devres_alloc_node` | allocate devres على node محدد |
| `devres_free` | `(res)` | free devres مش متربوط بـ device |
| `devres_add` | `(dev, res)` | ربط resource بالـ device |
| `devres_find` | `(dev, release, match, match_data) → void*` | ابحث عن resource |
| `devres_get` | `(dev, new_res, match, match_data) → void*` | find-or-add resource |
| `devres_remove` | `(dev, release, match, match_data) → void*` | اشيل resource من القايمة |
| `devres_destroy` | `(dev, release, match, match_data) → int` | اشيل + free resource |
| `devres_release` | `(dev, release, match, match_data) → int` | اشيل + call release + free |

#### Category 2: devres Groups

| Function | الغرض |
|---|---|
| `devres_open_group` | ابدأ group جديد في الـ devres list |
| `devres_close_group` | أغلق الـ group (بدون release) |
| `devres_remove_group` | امسح الـ group marker بس ابقى الـ resources |
| `devres_release_group` | release كل الـ resources الـ group من آخره |

#### Category 3: Managed Memory (devm_k*)

| Function | الغرض |
|---|---|
| `devm_kmalloc` | managed kmalloc |
| `devm_kzalloc` | managed kzalloc (zero-init) |
| `devm_krealloc` | managed krealloc |
| `devm_kmalloc_array` | managed kmalloc_array مع overflow check |
| `devm_kcalloc` | managed kcalloc |
| `devm_krealloc_array` | managed krealloc_array مع overflow check |
| `devm_kfree` | free مبكر لـ managed memory |
| `devm_kmemdup` | managed kmemdup (copy + alloc) |
| `devm_kmemdup_const` | managed kmemdup للـ const data |
| `devm_kmemdup_array` | managed kmemdup لـ array |
| `devm_kstrdup` | managed kstrdup |
| `devm_kstrdup_const` | managed kstrdup للـ const strings |
| `devm_kvasprintf` | managed vasprintf |
| `devm_kasprintf` | managed asprintf |

#### Category 4: Percpu + Pages

| Function | الغرض |
|---|---|
| `devm_alloc_percpu` | managed alloc_percpu (macro) |
| `__devm_alloc_percpu` | الـ implementation الفعلي |
| `devm_get_free_pages` | managed get_free_pages |
| `devm_free_pages` | free مبكر لـ managed pages |

#### Category 5: IOMEM Mapping

| Function | الغرض |
|---|---|
| `devm_ioremap_resource` | managed ioremap لـ struct resource |
| `devm_ioremap_resource_wc` | managed ioremap write-combining |
| `devm_of_iomap` | managed ioremap من device tree node |

#### Category 6: Custom Actions

| Function | الغرض |
|---|---|
| `__devm_add_action` | سجّل action callback في الـ devres |
| `devm_add_action` | macro لـ `__devm_add_action` |
| `devm_add_action_or_reset` | سجّل action، لو فشل نفّذه فوراً |
| `devm_remove_action` | شيل action من الـ devres list |
| `devm_remove_action_nowarn` | زي السابق بس بدون WARN |
| `devm_release_action` | نفّذ action دلوقتي واشيله |
| `devm_is_action_added` | هل الـ action موجود؟ |

---

### Group 1: Core devres Allocation & Lifecycle

الـ **devres** (device resource management) هو نظام الـ kernel اللي بيخلي كل resource متربوط بـ `struct device` — لما الـ device يتـ detach أو الـ driver يتـ unbind، الـ kernel بيـ release كل الـ resources التابعة ليه أوتوماتيك بترتيب LIFO. الـ core primitives دي هي الـ building blocks اللي كل الـ `devm_*` functions مبنية فوقيها.

---

#### `__devres_alloc_node` / `devres_alloc` / `devres_alloc_node`

```c
void * __malloc
__devres_alloc_node(dr_release_t release, size_t size, gfp_t gfp,
                    int nid, const char *name);

#define devres_alloc(release, size, gfp) \
    __devres_alloc_node(release, size, gfp, NUMA_NO_NODE, #release)

#define devres_alloc_node(release, size, gfp, nid) \
    __devres_alloc_node(release, size, gfp, nid, #release)
```

بتـ allocate buffer بحجم `size` bytes اللي هيستخدمه الـ driver لتخزين بياناته الخاصة. الـ buffer ده ملوش علاقة بالـ device لحد ما تعمل `devres_add`. الـ kernel بيضيف overhead صغير فوق الـ `size` عشان يخزّن فيه metadata زي `dr_release_t` والـ `name`.

**Parameters:**
- `release`: pointer للـ callback function اللي هيتسمى وقت الـ cleanup — نوعه `dr_release_t` أي `void (*)(struct device *, void *)`.
- `size`: حجم الـ data buffer اللي محتاجه الـ driver.
- `gfp`: الـ GFP flags — عادةً `GFP_KERNEL` في الـ probe context.
- `nid`: الـ NUMA node — `NUMA_NO_NODE` يعني أي node.
- `name`: string للـ debugging — الماكرو بيحطه `#release` أوتوماتيك.

**Return:** pointer للـ data area (مش للـ devres header) — أو `NULL` لو فشل الـ allocation.

**Key details:**
- الـ function مـ marked بـ `__malloc` يعني الـ compiler يعرف إنه بيرجع fresh allocation.
- الـ caller مسؤول إنه يعمل `devres_add` بعدها، لو مـعملش، لازم يعمل `devres_free` بنفسه.
- مفيش locking هنا — الـ locking بييجي في `devres_add`.

**Caller context:** يُستخدم من الـ `probe()` function أو أي sleepable context.

---

#### `devres_free`

```c
void devres_free(void *res);
```

بتـ free الـ devres buffer اللي اتعمل بـ `devres_alloc` لكن **مش اتضاف** للـ device بعد. لو الـ resource اتضاف بالفعل، استخدم `devres_destroy` بدلها.

**Parameters:**
- `res`: الـ pointer اللي رجعته `devres_alloc`.

**Return:** void.

**Key details:** بتـ recalculate الـ devres header address من الـ data pointer وتـ free الاتنين. استخدمها في error paths بعد `devres_alloc` ولو فشل الـ setup.

---

#### `devres_add`

```c
void devres_add(struct device *dev, void *res);
```

بتضيف الـ resource للـ device's devres list. من اللحظة دي، الـ kernel هيـ call الـ `release` callback أوتوماتيك وقت الـ device teardown.

**Parameters:**
- `dev`: الـ device اللي هيملك الـ resource.
- `res`: الـ pointer اللي رجعته `devres_alloc`.

**Return:** void.

**Key details:**
- بتاخد `devres_lock` (spinlock موجود في `struct device`) عشان تضيف الـ resource بـ thread-safety.
- الـ resources بتتضاف في الـ head — يعني LIFO order وقت الـ release.
- ممنوع تضيف نفس الـ `res` pointer أكتر من مرة.

---

#### `devres_find`

```c
void *devres_find(struct device *dev, dr_release_t release,
                  dr_match_t match, void *match_data);
```

بتدور على أول resource في الـ device's devres list اللي طابق الـ `release` function وبيـ pass الـ `match` callback.

**Parameters:**
- `dev`: الـ device المستهدف.
- `release`: لازم يتطابق مع الـ release function اللي اتسجلت.
- `match`: optional callback للـ fine-grained matching — لو `NULL`، أول resource بنفس الـ `release` بيتـ return.
- `match_data`: بيتبعت للـ `match` callback.

**Return:** pointer للـ data area لو لقى، أو `NULL`.

**Key details:** بيشتغل مع `devres_lock` held. الـ `dr_match_t` signature هي `int (*)(struct device *, void *res, void *match_data)` — ترجع non-zero لو match.

---

#### `devres_get`

```c
void *devres_get(struct device *dev, void *new_res,
                 dr_match_t match, void *match_data);
```

**Find-or-add** semantics: لو لقى resource موجود يطابق الـ match، بيـ free الـ `new_res` وبيرجع الموجود. لو ملقاش، بيضيف `new_res` وبيرجعه.

**Parameters:**
- `dev`: الـ device.
- `new_res`: resource جديد اتعمل بـ `devres_alloc` — هيتاخد أو هيتـ free.
- `match` / `match_data`: نفس `devres_find`.

**Return:** pointer للـ resource الفعلي المستخدم (إما الموجود أو `new_res`).

**Key details:** atomic بالنسبة للـ devres list. بيحل مشكلة الـ race condition في initialization paths مشتركة بين drivers.

**Pseudocode:**
```
devres_get(dev, new_res, match, data):
    lock(dev->devres_lock)
    existing = __devres_find(dev, release_of(new_res), match, data)
    if existing:
        unlock()
        devres_free(new_res)
        return existing
    else:
        __devres_add(dev, new_res)
        unlock()
        return new_res
```

---

#### `devres_remove`

```c
void *devres_remove(struct device *dev, dr_release_t release,
                    dr_match_t match, void *match_data);
```

بتشيل الـ resource من الـ device's list **بدون** ما تـ free أو تـ call الـ release callback. الـ caller بقى مسؤول عن الـ resource.

**Parameters:** زي `devres_find`.

**Return:** pointer للـ resource لو اتشال، أو `NULL` لو ملقاش.

**Key details:** بتاخد spinlock. الـ use case الرئيسي هو لما driver عايز يـ "adopt" resource وهو شايله من الـ managed list.

---

#### `devres_destroy`

```c
int devres_destroy(struct device *dev, dr_release_t release,
                   dr_match_t match, void *match_data);
```

بتشيل الـ resource من الـ list وتـ free الـ memory **بدون** ما تـ call الـ release callback.

**Return:** `0` لو نجح، `-ENOENT` لو ملقاش الـ resource.

**Key details:** الفرق عن `devres_release` هو إن ده ما بيـ call الـ `release` callback — بس بيـ free الـ devres memory نفسها.

---

#### `devres_release`

```c
int devres_release(struct device *dev, dr_release_t release,
                   dr_match_t match, void *match_data);
```

بتشيل الـ resource، بتـ call الـ `release(dev, res)` callback، وبعدين بتـ free الـ memory. ده الـ "full cleanup" path.

**Return:** `0` لو نجح، `-ENOENT` لو ملقاش.

**Caller context:** يُستخدم لما تيجي إنت تـ release resource بشكل مبكر قبل الـ driver detach.

---

### Group 2: devres Groups

الـ **devres groups** بتخليك تـ group مجموعة resources مع بعض تحت identifier واحد. الـ use case الأشهر هو الـ multi-step initialization اللي ممكن يفشل في أي نقطة — بدل ما تتتبع كل resource لوحده، بتفتح group، بتضيف resources، ولو فشلت بتـ release_group.

```
devres list: [R1] [R2] [GROUP_OPEN:id] [R3] [R4] [GROUP_CLOSE:id]
                                         ↑_________↑  ← هيتـ release لو عملت release_group
```

---

#### `devres_open_group`

```c
void * __must_check devres_open_group(struct device *dev, void *id, gfp_t gfp);
```

بتـ insert group marker في الـ devres list. كل resource بيتضاف بعد كده بيبقى "inside" الـ group.

**Parameters:**
- `dev`: الـ device.
- `id`: identifier للـ group — لو `NULL`، بيستخدم الـ marker pointer نفسه كـ id.
- `gfp`: للـ group marker allocation.

**Return:** الـ group id (pointer) لو نجح، `NULL` لو فشل. `__must_check` لأن failure هنا critical.

---

#### `devres_close_group`

```c
void devres_close_group(struct device *dev, void *id);
```

بتضيف closing marker للـ group. من اللحظة دي، الـ resources الجديدة بتبقى outside الـ group.

**Parameters:**
- `dev`: الـ device.
- `id`: نفس الـ id اللي استخدمته في `devres_open_group`.

---

#### `devres_remove_group`

```c
void devres_remove_group(struct device *dev, void *id);
```

بتشيل الـ group markers (open + close) من الـ list لكن **مش بتـ release** الـ resources اللي جوا. الـ resources بتفضل في الـ list بس من غير group structure. بتستخدمها لما الـ initialization نجحت وعايز "flatten" الـ group.

---

#### `devres_release_group`

```c
int devres_release_group(struct device *dev, void *id);
```

بتـ release (call release + free) كل الـ resources اللي اتضافت بعد الـ group open marker. بتشتغل بترتيب LIFO من آخر resource للـ group open marker.

**Return:** عدد الـ resources اللي اترلزت.

**Key details:** الـ group open marker نفسه بيتـ free. مفيش locking خاص — بتـ hold الـ spinlock أثناء العملية.

**Typical usage:**
```c
void *grp = devres_open_group(dev, NULL, GFP_KERNEL);
if (!grp)
    return -ENOMEM;

ret = setup_step_a(dev);  /* adds resources internally */
if (ret)
    goto fail;

ret = setup_step_b(dev);
if (ret)
    goto fail;

devres_remove_group(dev, grp);  /* success: flatten */
return 0;

fail:
    devres_release_group(dev, grp);  /* cleanup everything */
    return ret;
```

---

### Group 3: Managed Memory Allocation (devm_k*)

الـ **managed allocations** دي هي الـ API الأشهر في الـ devres system. كل function هنا بتـ wrap الـ standard kernel allocator وبتـ register cleanup callback أوتوماتيك — لما الـ device يتـ detach، الـ memory بتتـ free أوتوماتيك.

---

#### `devm_kmalloc`

```c
void * __alloc_size(2)
devm_kmalloc(struct device *dev, size_t size, gfp_t gfp);
```

الـ core الأساسي لكل الـ managed memory functions. بتـ allocate `size` bytes وبتربطها بالـ device.

**Parameters:**
- `dev`: الـ device المالك.
- `size`: الحجم بالـ bytes.
- `gfp`: مثلاً `GFP_KERNEL` في probe context.

**Return:** pointer للـ memory أو `NULL` لو فشل.

**Key details:**
- داخلياً بتعمل `devres_alloc` + copy + `devres_add`.
- الـ release callback المسجلة هي `devm_kmalloc_release` اللي بتعمل `kfree`.
- مـ marked بـ `__alloc_size(2)` عشان الـ static analyzers والـ compiler يعرفوا الـ size parameter.

---

#### `devm_kzalloc`

```c
static inline void *devm_kzalloc(struct device *dev, size_t size, gfp_t gfp)
{
    return devm_kmalloc(dev, size, gfp | __GFP_ZERO);
}
```

مجرد `devm_kmalloc` مع `__GFP_ZERO` flag. بترجع zeroed memory — الـ safer choice في الـ driver code عشان تتجنب uninitialized data bugs.

---

#### `devm_krealloc`

```c
void * __must_check __realloc_size(3)
devm_krealloc(struct device *dev, void *ptr, size_t size, gfp_t gfp);
```

بتـ resize allocation موجودة. لو الـ `ptr` كان `NULL`، بتتصرف زي `devm_kmalloc`. الـ resource بيفضل في نفس الـ devres list position.

**Parameters:**
- `ptr`: الـ pointer الموجود (أو `NULL`).
- `size`: الحجم الجديد.

**Return:** pointer للـ memory الجديدة أو `NULL`. `__must_check` لأن failure يعني الـ old pointer لسه valid.

**Key details:** لو `size == 0`، بتـ free الـ allocation وبترجع `ZERO_SIZE_PTR`. مش بتـ move الـ resource في الـ list — بس بتـ update الـ internal pointer.

---

#### `devm_kmalloc_array`

```c
static inline void *devm_kmalloc_array(struct device *dev,
                                        size_t n, size_t size, gfp_t flags)
{
    size_t bytes;
    if (unlikely(check_mul_overflow(n, size, &bytes)))
        return NULL;
    return devm_kmalloc(dev, bytes, flags);
}
```

**Overflow-safe** array allocation. `check_mul_overflow` بتكشف الـ integer overflow في `n * size` قبل ما تعمل الـ allocation — critical في driver code اللي بياخد sizes من hardware أو userspace.

**Parameters:**
- `n`: عدد العناصر.
- `size`: حجم كل عنصر.

---

#### `devm_kcalloc`

```c
static inline void *devm_kcalloc(struct device *dev,
                                   size_t n, size_t size, gfp_t flags)
{
    return devm_kmalloc_array(dev, n, size, flags | __GFP_ZERO);
}
```

`devm_kmalloc_array` مع zeroing. الأمثل للـ arrays of structs.

---

#### `devm_krealloc_array`

```c
static inline __realloc_size(3, 4) void * __must_check
devm_krealloc_array(struct device *dev, void *p,
                    size_t new_n, size_t new_size, gfp_t flags)
{
    size_t bytes;
    if (unlikely(check_mul_overflow(new_n, new_size, &bytes)))
        return NULL;
    return devm_krealloc(dev, p, bytes, flags);
}
```

Overflow-safe managed realloc للـ arrays.

---

#### `devm_kfree`

```c
void devm_kfree(struct device *dev, const void *p);
```

بتـ free managed allocation **قبل** الـ device detach. مفيدة في error paths أو لما تعرف بالظبط إمتى الـ resource مش محتاجه.

**Parameters:**
- `dev`: الـ device المالك.
- `p`: الـ pointer اللي رجع من `devm_kmalloc` أو مشتقاتها.

**Key details:** بتعمل `devres_destroy` داخلياً. لو الـ `p` مش موجود في الـ list، يعمل WARN. مش بتـ call الـ release callback — بس بتـ free الـ memory مباشرة.

---

#### `devm_kmemdup`

```c
void * __realloc_size(3)
devm_kmemdup(struct device *dev, const void *src, size_t len, gfp_t gfp);
```

بتـ allocate managed buffer وبتـ copy `len` bytes من `src` ليه. يعني managed version من `kmemdup`.

**Return:** pointer للـ new copy أو `NULL`.

---

#### `devm_kmemdup_const`

```c
const void *
devm_kmemdup_const(struct device *dev, const void *src, size_t len, gfp_t gfp);
```

نسخة من `devm_kmemdup` للـ const data. الفرق الجوهري: لو الـ `src` كانت في الـ `.rodata` section (read-only kernel data)، الـ function ممكن ترجع الـ pointer الأصلي من غير ما تعمل copy — optimization عشان توفر memory. لو مش في `.rodata`، بتعمل copy عادية.

**Return:** `const void *` — مش ينفع تـ write فيه.

---

#### `devm_kmemdup_array`

```c
static inline void *devm_kmemdup_array(struct device *dev, const void *src,
                                        size_t n, size_t size, gfp_t flags)
{
    return devm_kmemdup(dev, src, size_mul(size, n), flags);
}
```

بتـ copy array كاملة. بتستخدم `size_mul` للـ overflow protection.

---

#### `devm_kstrdup`

```c
char * __malloc
devm_kstrdup(struct device *dev, const char *s, gfp_t gfp);
```

Managed string duplication. بتـ allocate managed buffer، بتـ copy الـ string مع الـ null terminator.

**Return:** pointer للـ new string أو `NULL`.

---

#### `devm_kstrdup_const`

```c
const char *devm_kstrdup_const(struct device *dev, const char *s, gfp_t gfp);
```

زي `devm_kmemdup_const` بالظبط — لو الـ string في `.rodata`، بترجع نفس الـ pointer من غير allocation. مفيدة جداً لـ device names وـ property strings اللي جاية من الـ device tree.

---

#### `devm_kvasprintf` / `devm_kasprintf`

```c
char * __printf(3, 0) __malloc
devm_kvasprintf(struct device *dev, gfp_t gfp, const char *fmt, va_list ap);

char * __printf(3, 4) __malloc
devm_kasprintf(struct device *dev, gfp_t gfp, const char *fmt, ...);
```

بتـ format string وتـ allocate managed buffer بالحجم الصح أوتوماتيك. `devm_kasprintf` بتستخدمها لبناء device names أو sysfs paths ديناميكياً.

**Key details:** الـ `__printf(3, 4)` annotation يخلي الـ compiler يـ check format string arguments compile time. بتعمل two-pass: أول مرة بتحسب الـ size، تاني مرة بتكتب.

---

### Group 4: Managed Per-CPU Memory

---

#### `devm_alloc_percpu` / `__devm_alloc_percpu`

```c
#define devm_alloc_percpu(dev, type) \
    ((typeof(type) __percpu *)__devm_alloc_percpu((dev), sizeof(type), __alignof__(type)))

void __percpu *__devm_alloc_percpu(struct device *dev, size_t size, size_t align);
```

الـ macro بيـ wrap الـ internal function وبيضمن إن الـ alignment صح للـ type. بترجع per-CPU pointer (`__percpu *`) اللي بتوصله بـ `per_cpu_ptr()` أو `this_cpu_ptr()`.

**Parameters:**
- `dev`: الـ device.
- `size`: حجم كل per-cpu copy.
- `align`: alignment requirement.

**Return:** `__percpu *` pointer أو `NULL`.

**Key details:** الـ release callback بتعمل `free_percpu`. مفيدة في drivers محتاجين per-CPU counters أو state بدون الـ manual cleanup في remove.

---

### Group 5: Managed Page Allocation

---

#### `devm_get_free_pages`

```c
unsigned long devm_get_free_pages(struct device *dev, gfp_t gfp_mask, unsigned int order);
```

Managed version من `__get_free_pages`. بتـ allocate `2^order` pages.

**Parameters:**
- `gfp_mask`: الـ GFP flags.
- `order`: log2 عدد الـ pages.

**Return:** virtual address للـ pages أو `0` لو فشل.

**Key details:** الـ release callback بتعمل `free_pages`. مفيدة لـ DMA buffers أو large contiguous allocations.

---

#### `devm_free_pages`

```c
void devm_free_pages(struct device *dev, unsigned long addr);
```

Free مبكر للـ managed pages. بتعمل `devres_destroy` للـ resource المرتبط بالـ `addr`.

---

### Group 6: Managed IOMEM Mapping

الـ functions دي بتـ request + map hardware registers وبتـ register cleanup callback تعمل `iounmap` + `release_mem_region` أوتوماتيك.

---

#### `devm_ioremap_resource`

```c
void __iomem *devm_ioremap_resource(struct device *dev, const struct resource *res);
```

الـ function الأكثر استخداماً في driver code. بتعمل الآتي:
1. Validates الـ `struct resource` (type + size).
2. بتعمل `request_mem_region` (exclusive ownership).
3. بتعمل `ioremap` للـ physical address.
4. بتـ register cleanup callback تعمل `iounmap` + `release_mem_region`.

**Parameters:**
- `dev`: الـ device.
- `res`: عادةً بتيجي من `platform_get_resource()` أو `devm_platform_get_resource()`.

**Return:** `__iomem` pointer أو `ERR_PTR(-errno)` لو فشل.

**Key details:**
- بترجع error pointer مش `NULL` — لازم تتـ check بـ `IS_ERR()`.
- لو `CONFIG_HAS_IOMEM` مش defined، بترجع `IOMEM_ERR_PTR(-EINVAL)` دايماً.

**Typical usage:**
```c
/* في probe() */
void __iomem *base = devm_ioremap_resource(dev, res);
if (IS_ERR(base))
    return PTR_ERR(base);
/* استخدم base مباشرة */
writel(val, base + REG_OFFSET);
/* مفيش iounmap في remove() */
```

---

#### `devm_ioremap_resource_wc`

```c
void __iomem *devm_ioremap_resource_wc(struct device *dev, const struct resource *res);
```

زي `devm_ioremap_resource` بالظبط لكن بتـ map الـ memory بـ **write-combining** (WC) policy. مفيدة لـ framebuffer memory وـ GPU memory اللي sequential writes بتفيد منها.

---

#### `devm_of_iomap`

```c
void __iomem *devm_of_iomap(struct device *dev, struct device_node *node,
                              int index, resource_size_t *size);
```

بتـ map MMIO من **device tree** مباشرة. بتقرأ الـ `reg` property من الـ `node` بالـ `index` وبتعمل ioremap.

**Parameters:**
- `node`: الـ device tree node.
- `index`: رقم الـ `reg` entry في الـ `reg` property.
- `size`: output — حجم الـ region (optional — ممكن يبقى `NULL`).

**Return:** `__iomem` pointer أو `ERR_PTR(-errno)`.

---

### Group 7: Custom Action Callbacks

الـ **custom actions** بتخليك تـ register أي arbitrary function كـ cleanup callback في الـ devres system — من غير ما تحتاج تعمل `devres_alloc` بنفسك. ده powerful tool لـ wrapping non-standard resources.

---

#### `__devm_add_action` / `devm_add_action`

```c
int __devm_add_action(struct device *dev, void (*action)(void *),
                      void *data, const char *name);

#define devm_add_action(dev, action, data) \
    __devm_add_action(dev, action, data, #action)
```

بتـ register `action(data)` ليتـ called أوتوماتيك وقت الـ device detach بترتيب LIFO.

**Parameters:**
- `action`: callback بالـ signature `void (*)(void *)`.
- `data`: بيتبعت لـ `action` كـ argument.
- `name`: للـ debugging — الـ macro بيستخدم `#action`.

**Return:** `0` لو نجح، `-ENOMEM` لو فشل الـ allocation.

**Key details:**
- داخلياً بتعمل `devres_alloc` للـ action wrapper struct وبتـ store الـ function + data فيه.
- الـ release callback الخاصة بالـ wrapper بتـ call `action(data)`.
- مش بتـ call الـ action فوراً — بس بتسجلها.

**Typical usage:**
```c
/* enable clock */
ret = clk_prepare_enable(clk);
if (ret)
    return ret;

/* register cleanup */
ret = devm_add_action(dev, (void (*)(void *))clk_disable_unprepare, clk);
if (ret) {
    clk_disable_unprepare(clk);
    return ret;
}
```

---

#### `devm_add_action_or_reset`

```c
static inline int __devm_add_action_or_reset(struct device *dev,
                                              void (*action)(void *),
                                              void *data, const char *name)
{
    int ret;
    ret = __devm_add_action(dev, action, data, name);
    if (ret)
        action(data);  /* call immediately on registration failure */
    return ret;
}

#define devm_add_action_or_reset(dev, action, data) \
    __devm_add_action_or_reset(dev, action, data, #action)
```

الـ "safe" version — لو فشل تسجيل الـ action (بسبب `-ENOMEM`)، بتـ call الـ `action(data)` فوراً عشان الـ resource اللي اتعمل مش يـ leak.

**Key details:**
- هو شبه `devm_add_action` لكن مع built-in safety net.
- الـ pattern الأمثل:
  ```c
  ret = clk_prepare_enable(clk);
  if (ret) return ret;
  return devm_add_action_or_reset(dev, clk_disable_unprepare_wrapper, clk);
  ```

---

#### `devm_remove_action`

```c
static inline
void devm_remove_action(struct device *dev, void (*action)(void *), void *data)
{
    WARN_ON(devm_remove_action_nowarn(dev, action, data));
}
```

بتشيل الـ action من الـ devres list **بدون** تنفيذها. بتستخدم `WARN_ON` لو الـ action مش موجودة — يعني programming error.

---

#### `devm_remove_action_nowarn`

```c
int devm_remove_action_nowarn(struct device *dev, void (*action)(void *), void *data);
```

نفس `devm_remove_action` بس بدون الـ `WARN_ON`. بتـ return `-ENOENT` لو ملقتش الـ action.

**Key details:** الـ match بيحصل بالـ function pointer **والـ data pointer معاً** — يعني نفس الـ action بنفس الـ data. لو سجلت نفس الـ action مرتين بـ data مختلف، كل واحدة هتتـ match لوحدها.

---

#### `devm_release_action`

```c
void devm_release_action(struct device *dev, void (*action)(void *), void *data);
```

بتـ call الـ action **فوراً** وبتشيلها من الـ list. يعني force immediate cleanup لـ resource معين من غير ما تستنى الـ device detach.

**Key details:** مش بتـ return error — لو الـ action مش موجودة، هيحصل WARN.

---

#### `devm_is_action_added`

```c
bool devm_is_action_added(struct device *dev, void (*action)(void *), void *data);
```

بتـ check لو action بعينها مسجلة في الـ devres list. مفيدة في code paths اللي ممكن تـ call setup مرتين.

**Return:** `true` لو موجودة، `false` لو لأ.

---

### Type Definitions الأساسية

```c
/* Release callback — بيتسمى وقت cleanup */
typedef void (*dr_release_t)(struct device *dev, void *res);

/* Match callback — بيحدد الـ resource الصح */
typedef int (*dr_match_t)(struct device *dev, void *res, void *match_data);
```

الـ `dr_release_t` هو قلب الـ devres system — كل resource عنده release function اتسجلت وقت الـ allocation. الـ `dr_match_t` اختياري وبيسمح بـ fine-grained search في الـ devres list.

---

### الـ devres Architecture بشكل مختصر

```
struct device
    └── devres_head (linked list)
            ├── [devres_node: release=kfree, data=buffer_A]
            ├── [devres_node: release=iounmap, data=iomap_B]
            ├── [devres_node: release=action_wrapper, data={fn, arg}]
            └── [devres_node: release=clk_disable, data=clk_C]

device_del() / driver_detach():
    → __release_devres(dev)
        → for each node in LIFO order:
            node->release(dev, node->data)
            kfree(node)
```

الـ LIFO order مهم — الـ resources بتتـ release بالعكس، يعني آخر حاجة اتـ allocate أول حاجة بتتـ release. ده بيضمن إن dependencies بتتـ unwind بالشكل الصح.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. Debugfs Entries

**الـ devres** بيسجّل كل resource اتعملت على device عن طريق `debugfs`. المسار الأساسي:

```
/sys/kernel/debug/devices/<bus>/<device>/devres
```

بس الأكتر فايدة هو المسار ده:

```
/sys/kernel/debug/devices_deferred
```

**إزاي تقرأهم:**

```bash
# اقرأ كل الـ devres resources المسجّلة على device معينة
# مثلاً على PCI device
cat /sys/kernel/debug/devices/pci0000\:00/0000\:00\:1f.2/devres

# أو لـ platform device
cat /sys/kernel/debug/devices/platform/my_driver.0/devres
```

**شكل الـ output:**

```
[0] devm_ioremap_resource (8 bytes) @ ffff888100a00000
[1] devm_kmalloc (256 bytes) @ ffff888102b40000
[2] devm_clk_get (24 bytes) @ ffff8881011c0400
```

كل سطر فيه: index، اسم الـ release function، الحجم، والعنوان.

---

#### 2. Sysfs Entries

**الـ devres** مش ليها sysfs مباشر، بس الـ device نفسها ليها entries مفيدة:

| المسار | المحتوى |
|--------|---------|
| `/sys/bus/<bus>/devices/<dev>/driver` | الـ driver الحالي المربوط بالـ device |
| `/sys/bus/<bus>/devices/<dev>/power/` | معلومات الـ power management |
| `/sys/bus/<bus>/devices/<dev>/uevent` | الـ kobject uevent attributes |
| `/sys/kernel/slab/devres_*/` | إحصائيات الـ slab allocator لكل الـ devres objects |

```bash
# شوف الـ slab stats لـ devres
ls /sys/kernel/slab/ | grep devres

# اقرأ stats لـ slab معين
cat /sys/kernel/slab/devres_node/alloc_calls
cat /sys/kernel/slab/devres_node/free_calls
cat /sys/kernel/slab/devres_node/objects
```

---

#### 3. Ftrace — Tracepoints و Events

**الـ devres** مش ليها tracepoints مخصصة، بس تقدر تستخدم:

##### Function Tracer على دوال الـ devres

```bash
# فعّل function tracer على كل دوال devm_*
echo function > /sys/kernel/debug/tracing/current_tracer

# فلتر على دوال الـ devres فقط
echo 'devm_kmalloc devres_add devres_find devres_destroy devres_release_all' \
  > /sys/kernel/debug/tracing/set_ftrace_filter

# فعّل الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل الـ driver أو عمل bind/unbind
echo "0000:00:1f.2" > /sys/bus/pci/drivers/my_driver/bind

# اقرأ النتيجة
cat /sys/kernel/debug/tracing/trace
```

##### Tracepoints مفيدة للسياق

```bash
# تتبع الـ kmalloc من الـ kernel
echo 1 > /sys/kernel/debug/tracing/events/kmem/kmalloc/enable
echo 1 > /sys/kernel/debug/tracing/events/kmem/kfree/enable

# تتبع الـ driver probe
echo 1 > /sys/kernel/debug/tracing/events/device/enable

# فلتر على device معينة
echo 'name == "my_device"' \
  > /sys/kernel/debug/tracing/events/device/filter
```

##### Graph Tracer لتتبع call chain

```bash
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo devres_release_all > /sys/kernel/debug/tracing/set_graph_function
echo 1 > /sys/kernel/debug/tracing/tracing_on

# echo unbind لتشغيل الـ release
echo "0000:00:1f.2" > /sys/bus/pci/drivers/my_driver/unbind

cat /sys/kernel/debug/tracing/trace
```

**شكل الـ output:**

```
 0)               |  devres_release_all() {
 0)               |    release_nodes() {
 0)   0.532 us    |      devm_kmalloc_release();
 0)   1.201 us    |      devm_ioremap_release();
 0)   3.401 us    |    }
 0)   4.102 us    |  }
```

---

#### 4. Printk و Dynamic Debug

##### تفعيل Dynamic Debug لـ subsystem الـ devres

```bash
# فعّل كل debug messages في drivers/base/devres.c
echo 'file drivers/base/devres.c +p' > /sys/kernel/debug/dynamic_debug/control

# أو فعّل كل module معين
echo 'module my_driver +p' > /sys/kernel/debug/dynamic_debug/control

# أو فعّل بـ function name
echo 'func devres_add +pflmt' > /sys/kernel/debug/dynamic_debug/control
# p = printk, f = func name, l = line number, m = module, t = timestamp
```

##### تفعيل Devres Debug من Boot

```
kernel cmdline: dyndbg="file drivers/base/devres.c +p"
```

##### شوف الـ active dynamic debug entries

```bash
cat /sys/kernel/debug/dynamic_debug/control | grep devres
```

##### Printk Level

```bash
# خلّي الـ kernel يطبع كل messages
dmesg -n 8
# أو
echo 8 > /proc/sys/kernel/printk
```

---

#### 5. Kernel Config Options للـ Debugging

| Config | الوصف |
|--------|-------|
| `CONFIG_DEBUG_DEVRES` | **الأهم** — يطبع كل devres allocation/release operations في dmesg |
| `CONFIG_KASAN` | Kernel Address Sanitizer — يكتشف use-after-free و out-of-bounds في الـ devres allocations |
| `CONFIG_KMEMLEAK` | يكتشف memory leaks بما فيها الـ devres المنسية |
| `CONFIG_KMSAN` | Kernel Memory Sanitizer — يكتشف uninitialized memory |
| `CONFIG_DEBUG_SLAB` | يفعّل slab debugging (poisoning, red zones) |
| `CONFIG_SLUB_DEBUG` | SLUB allocator debugging |
| `CONFIG_DEBUG_PAGEALLOC` | يراقب الـ page allocations |
| `CONFIG_DEBUG_OBJECTS_*` | يتبع lifecycle الـ kernel objects |
| `CONFIG_LOCKDEP` | يكتشف deadlocks في الـ devres locking |
| `CONFIG_PROVE_LOCKING` | تحقق من صحة الـ locking order |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل الـ dynamic debug interface |
| `CONFIG_FTRACE` | Function tracing |

**تفعيل `CONFIG_DEBUG_DEVRES` بيطلّع output زي ده في dmesg:**

```
devres platform my_device: ALLOC devm_kmalloc (256 bytes)
devres platform my_device: ALLOC devm_ioremap_resource (8 bytes)
devres platform my_device: REL  devm_ioremap_resource
devres platform my_device: REL  devm_kmalloc
```

---

#### 6. Tools خاصة بالـ Subsystem

##### KMEMLEAK للـ Memory Leaks

```bash
# شغّل الـ kmemleak scan يدوي
echo scan > /sys/kernel/debug/kmemleak

# اقرأ النتيجة
cat /sys/kernel/debug/kmemleak
```

**إذا فيه driver بيعمل `devres_alloc` بس مبيضيفش بـ `devres_add`، الـ kmemleak هيلتقطه.**

##### KASAN Report مثال

```
BUG: KASAN: use-after-free in devm_kfree+0x3a/0x60
Write of size 8 at addr ffff888102b40010 by task insmod/1234
```

##### صلاحية الـ pointer بعد unbind

```bash
# بعد unbind، أي pointer اتجاب بـ devm_kmalloc يبقى invalid
# تقدر تتحقق بـ KASAN أو بالـ SLUB poisoning:
cat /sys/kernel/slab/kmalloc-256/poison
```

---

#### 7. جدول Common Errors

| رسالة الـ Kernel | المعنى | الحل |
|-----------------|--------|-------|
| `devres: ALLOC devm_kmalloc (N bytes) FAILED` | فشل الـ allocation بسبب OOM أو gfp flags خاطئة | تحقق من الـ gfp flags، استخدم `GFP_KERNEL` مش `GFP_ATOMIC` في السياق الصح |
| `BUG: unable to handle page fault ... devm_kfree` | استخدام pointer بعد ما اتعمله free عند unbind | لا تخزّن الـ devres pointers في global أو في struct بيعيش أطول من الـ device |
| `WARN: devres_find: Cannot find resource` | محاولة `devres_destroy` أو `devres_remove` على resource مش موجودة | تحقق إن `devres_add` اتنادت قبل الـ remove |
| `KASAN: use-after-free in release_nodes` | الـ release callback بتحاول تقرأ memory اترفضت | الـ release order غلط، أو الـ callback بتحاول تعمل free لمحتوى الـ resource بعد ما اتحذف |
| `kernel BUG at drivers/base/devres.c:XXX` | مشكلة في consistency الـ devres list | عادةً بسبب concurrent access بدون lock |
| `devm_ioremap_resource: IORESOURCE_MEM missing` | الـ resource مش نوعها `IORESOURCE_MEM` | تحقق من الـ platform_device resources أو الـ DT memory-mapped-io |
| `devm_of_iomap: invalid resource` | الـ DT node مش فيه `reg` property صح | راجع الـ DT binding وتأكد من `reg` و `#address-cells` |
| `Memory allocated by devm_kmalloc still in use after driver removal` | سرّب memory لأن `devm_kfree` اتنادت بشكل explicit على pointer مش managed | لا تنادي `devm_kfree` يدوي إلا لو عارف إيه اللي بتعمله |
| `devres group open but not closed` | `devres_open_group` بدون `devres_close_group` مقابلها | دايماً وازن الـ open/close في نفس الـ code path |

---

#### 8. Strategic Points لـ dump_stack() و WARN_ON()

**أماكن تستفيد منها أثناء debug:**

```c
/* في الـ release callback — تحقق إن الـ resource لسه valid */
static void my_resource_release(struct device *dev, void *res)
{
    struct my_resource *r = res;

    /* تحقق إن مفيش استخدام concurrent */
    WARN_ON(!list_empty(&r->pending_list));

    /* اطبع الـ call stack عشان تعرف مين trigger الـ release */
    if (debug_mode)
        dump_stack();

    cleanup_my_resource(r);
}
```

```c
/* في الـ probe — بعد كل devm_kmalloc */
priv = devm_kzalloc(dev, sizeof(*priv), GFP_KERNEL);
WARN_ON_ONCE(!priv); /* مش المفروض يحصل مع GFP_KERNEL */

/* في الـ group handling */
id = devres_open_group(dev, NULL, GFP_KERNEL);
if (WARN_ON(!id))
    return -ENOMEM;
```

```c
/* تحقق من double-free محتمل */
static void my_release(struct device *dev, void *res)
{
    struct my_data **ptr = res;
    WARN_ON(*ptr == NULL); /* لو NULL يبقى اتعمل release قبل كده */
    *ptr = NULL;           /* mark as freed */
}
```

---

### Hardware Level

---

#### 1. التحقق إن الـ Hardware State بيوافق الـ Kernel State

**الـ devres** بتعمل `ioremap` للـ MMIO regions. تحقق إن الـ mapping صح:

```bash
# شوف الـ I/O memory allocations الحالية
cat /proc/iomem

# مثال output:
# fe200000-fe20ffff : my_driver
#   fe200000-fe20ffff : my_driver.regs
```

```bash
# تحقق إن الـ virtual address mapping موجود
cat /proc/vmallocinfo | grep ioremap

# شوف memory mapped devices
cat /proc/iomem | grep -i "my_device"
```

```bash
# تحقق من الـ resource allocation
cat /proc/ioports       # للـ PIO resources
cat /proc/iomem         # للـ MMIO resources
```

**ربط بين الـ DT resource والـ kernel mapping:**

```bash
# شوف resource الـ device في sysfs
cat /sys/bus/platform/devices/my_device.0/resources
# أو
cat /sys/devices/platform/my_device.0/resource
```

---

#### 2. Register Dump Techniques

##### باستخدام `devmem2`

```bash
# اقرأ register من عنوان physical
devmem2 0xfe200000 w    # w = word (32-bit)
devmem2 0xfe200004 h    # h = halfword (16-bit)
devmem2 0xfe200008 b    # b = byte (8-bit)

# اكتب قيمة في register
devmem2 0xfe200000 w 0x00000001
```

##### باستخدام `/dev/mem`

```bash
# اقرأ 64 bytes من MMIO region
dd if=/dev/mem bs=1 skip=$((0xfe200000)) count=64 2>/dev/null | xxd
```

##### باستخدام `io` utility

```bash
# اقرأ MMIO (من busybox أو custom tool)
io -4 -r 0xfe200000     # 4-byte read
io -4 -w 0xfe200000 0x1 # 4-byte write
```

##### من داخل الـ Kernel Module للـ Debug

```c
#include <linux/io.h>

/* افترض إن base هو الـ ioremap pointer */
void __iomem *base = devm_ioremap_resource(dev, res);

/* dump أول 16 registers */
for (int i = 0; i < 16; i++) {
    u32 val = readl(base + i * 4);
    dev_dbg(dev, "reg[%02d] = 0x%08x\n", i, val);
}
```

---

#### 3. Logic Analyzer و Oscilloscope Tips

**لما الـ devres ioremap بيفشل أو الـ registers مش بتتقرأ صح:**

| الموقف | الأداة | اللي بتتحقق منه |
|--------|--------|----------------|
| الـ MMIO access بيعمل bus fault | Logic analyzer على الـ address bus | الـ chip select صح؟ الـ address decode سليم؟ |
| الـ register بيرجع قيمة ثابتة (مثلاً `0xDEADBEEF`) | Oscilloscope على الـ data bus | الـ device مش powered أو مش في reset |
| `devm_ioremap_resource` بيرجع error | لازم تتحقق من الـ physical address range | |
| الـ device مش بتتعرف | Logic analyzer على الـ I2C/SPI/PCIe bus | Protocol timing، الـ device ID |

**نقاط القياس المفيدة:**

```
1. RESET# pin — لازم يبقى high قبل ما تبدأ تعمل ioremap
2. POWER GOOD — تأكد إن الـ supply stable قبل probe
3. CS# (Chip Select) — لما تعمل readl/writel
4. IRQ line — بعد initialization لو مفيش interrupts
```

---

#### 4. Common Hardware Issues وـ Kernel Log Patterns

| مشكلة الـ Hardware | الـ Kernel Log Pattern | التفسير |
|-------------------|----------------------|---------|
| Device مش موجودة على الـ bus | `devm_ioremap_resource: can't reserve [mem 0xfe200000-0xfe20ffff]` | منطقة الـ MMIO اتحجزت مسبقاً أو مش موجودة في الـ resource tree |
| Wrong physical address | `Unable to handle kernel paging request at virtual address` | الـ ioremap عمل map لعنوان غلط |
| Device مش powered | قراءة `0xFFFFFFFF` أو `0x00000000` من كل registers | الـ bus floating |
| Reset مش اترفع | الـ driver بيكمل probe بس السخرة مش بتشتغل | تحقق من GPIO reset line |
| Clock مش مفعّل | `devm_clk_get: clock not found` أو الـ device فاشلة بصمت | الـ clock يلزم يتفعّل قبل register access |
| IRQ line مش متوصل | الـ driver بيعمل probe بنجاح بس مفيش interrupts | تحقق من الـ DT `interrupts` property |

---

#### 5. Device Tree Debugging

**الـ `devm_of_iomap` و `devm_ioremap_resource` بيعتمدوا على الـ DT. لو فيه مشكلة:**

##### تحقق من الـ DT المحمّل

```bash
# اقرأ الـ DT المحمّل حالياً (الـ live DT)
ls /sys/firmware/devicetree/base/

# شوف node معين
ls /sys/firmware/devicetree/base/soc/my_device@fe200000/

# اقرأ الـ reg property
xxd /sys/firmware/devicetree/base/soc/my_device@fe200000/reg
# Output: 00000000 fe200000 00000000 00010000
# يعني: base=0xfe200000, size=0x10000
```

```bash
# dump كل الـ DT كـ text
dtc -I fs -O dts /sys/firmware/devicetree/base/ 2>/dev/null | grep -A 20 "my_device"
```

##### تحقق من compatible string

```bash
# تأكد إن الـ compatible بيطابق الـ driver
cat /sys/bus/platform/devices/my_device.0/of_node/compatible
# لازم يكون زي: "mycompany,my-device-v2"
```

##### مقارنة DT مع الـ Hardware

```bash
# الـ reg property → يلزم يطابق الـ datasheet memory map
xxd /sys/firmware/devicetree/base/soc/my_device@fe200000/reg

# الـ interrupts property → يلزم يطابق الـ schematic
xxd /sys/firmware/devicetree/base/soc/my_device@fe200000/interrupts

# الـ clocks property → يلزم يطابق الـ clock tree
cat /sys/firmware/devicetree/base/soc/my_device@fe200000/clock-names
```

##### تحقق من resource allocation بعد probe

```bash
# لو الـ probe نجح
cat /proc/iomem | grep my_device
# لازم يظهر الـ region المحجوزة

# لو فشل
dmesg | grep "my_device\|devres\|ioremap" | tail -20
```

---

### Practical Commands

---

#### السيناريو 1: تتبع كامل لـ driver probe وـ devres allocations

```bash
#!/bin/bash
# script لتتبع driver probe مع كل devres operations

DRIVER="my_driver"
DEVICE="my_device.0"

# 1. فعّل CONFIG_DEBUG_DEVRES messages
echo 8 > /proc/sys/kernel/printk

# 2. فعّل dynamic debug للـ devres
echo 'file drivers/base/devres.c +pflmt' \
  > /sys/kernel/debug/dynamic_debug/control

# 3. فعّل function tracer
echo function > /sys/kernel/debug/tracing/current_tracer
echo 'devm_* devres_*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on

# 4. عمل bind للـ driver
echo "$DEVICE" > /sys/bus/platform/drivers/$DRIVER/bind

# 5. اقرأ النتيجة
echo "=== ftrace output ==="
cat /sys/kernel/debug/tracing/trace

echo "=== dmesg devres output ==="
dmesg | grep -i "devres\|devm_"

# 6. شوف الـ allocated resources
echo "=== current devres resources ==="
cat /sys/kernel/debug/devices/platform/$DEVICE/devres 2>/dev/null

# cleanup
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo nop > /sys/kernel/debug/tracing/current_tracer
```

---

#### السيناريو 2: تتبع memory leak عند unbind

```bash
#!/bin/bash
# تحقق من memory leaks في devres

DRIVER="my_driver"
DEVICE="my_device.0"

# 1. صفّر الـ kmemleak
echo clear > /sys/kernel/debug/kmemleak

# 2. افعل unbind
echo "$DEVICE" > /sys/bus/platform/drivers/$DRIVER/unbind

# 3. انتظر الـ GC
echo scan > /sys/kernel/debug/kmemleak

# 4. شوف الـ leaks
cat /sys/kernel/debug/kmemleak

# إذا ظهر output → فيه memory اتنسيت ومش managed بـ devres
```

**مثال output من kmemleak:**

```
unreferenced object 0xffff888102b40000 (size 256):
  comm "probe_thread", pid 1234
  backtrace:
    kmalloc+0x45
    my_driver_probe+0x89   ← هنا المشكلة، kmalloc بدل devm_kmalloc
    platform_probe+0x35
```

---

#### السيناريو 3: تحقق من الـ devres release order

```bash
# فعّل graph tracer على devres_release_all
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo devres_release_all > /sys/kernel/debug/tracing/set_graph_function
echo 1 > /sys/kernel/debug/tracing/tracing_on

# افعل unbind
echo "my_device.0" > /sys/bus/platform/drivers/my_driver/unbind

# اقرأ
cat /sys/kernel/debug/tracing/trace | head -50

# الـ output بيوضح الترتيب: LIFO (آخر resource اتحجزت، أول ما اترفضت)
```

---

#### السيناريو 4: register dump من userspace

```bash
#!/bin/bash
# افترض إن الـ device على عنوان 0xfe200000، حجم 0x1000

PHYS_BASE=0xfe200000
SIZE=256  # عدد bytes

echo "=== Register Dump for device @ $PHYS_BASE ==="

# طريقة 1: devmem2
for offset in $(seq 0 4 $((SIZE-4))); do
    addr=$(printf "0x%x" $((PHYS_BASE + offset)))
    val=$(devmem2 $addr w 2>/dev/null | awk '/Read at/ {print $NF}')
    printf "0x%04x: %s\n" $offset "$val"
done

# طريقة 2: dd + xxd
dd if=/dev/mem bs=1 \
   skip=$((PHYS_BASE)) \
   count=$SIZE \
   2>/dev/null | xxd
```

**مثال output:**

```
0x0000: 0x00000001   ← CTRL register: enabled
0x0004: 0x0000001f   ← STATUS register: 5 bits set
0x0008: 0x00000000   ← IRQ_STATUS: no pending interrupts
0x000c: 0xdeadbeef   ← UNKNOWN: device not responding!
```

---

#### السيناريو 5: تحقق سريع من الـ DT resources

```bash
#!/bin/bash
# قارن الـ DT resources مع الـ /proc/iomem

DEVICE_NODE="/sys/firmware/devicetree/base/soc/my_device@fe200000"

echo "=== DT reg property ==="
xxd "$DEVICE_NODE/reg" 2>/dev/null

echo "=== Kernel resource allocation ==="
grep -i "my_device\|fe200" /proc/iomem

echo "=== Driver binding ==="
ls -la /sys/bus/platform/devices/my_device.0/driver 2>/dev/null

echo "=== DT status ==="
cat "$DEVICE_NODE/status" 2>/dev/null || echo "(no status property → okay by default)"

echo "=== Compatible string ==="
cat "$DEVICE_NODE/compatible" 2>/dev/null
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: memory leak في driver الـ SPI على RK3562

#### العنوان
**Memory leak** في driver الـ SPI بسبب عدم استخدام `devm_kmalloc`

#### السياق
بنشتغل على industrial gateway بيشتغل على **RK3562**، الـ gateway بيقرأ data من 4 sensors عن طريق **SPI**. الـ driver اتكتب من البداية بـ `kmalloc` عادي، وكل حاجة شغالة تمام في الـ testing القصير. لما الـ gateway اتنشر في الـ field وبدأ يشتغل لفترات طويلة، بدأ الـ system يبوأ panic بعد كام يوم بسبب OOM.

#### المشكلة
الـ developer استخدم `kmalloc` عشان يخزن struct الـ per-channel config، لكن في حالة فشل الـ probe أو الـ unbind، الـ cleanup function مش بتتنادى صح فبالتالي الـ memory مش بتتحرر.

```c
/* الكود القديم — فيه memory leak */
static int spi_sensor_probe(struct spi_device *spi)
{
    struct sensor_priv *priv;

    priv = kmalloc(sizeof(*priv), GFP_KERNEL); /* مش managed */
    if (!priv)
        return -ENOMEM;

    /* لو حاجة فشلت هنا، priv مش هيتحرر */
    if (init_channels(priv))
        return -EIO; /* memory leak! */

    spi_set_drvdata(spi, priv);
    return 0;
}

static void spi_sensor_remove(struct spi_device *spi)
{
    struct sensor_priv *priv = spi_get_drvdata(spi);
    /* لو probe فشل، priv هيبقى garbage */
    kfree(priv);
}
```

#### التحليل
الـ `devres.h` بيوفر `devm_kmalloc` اللي بتربط الـ allocation بـ lifetime الـ device:

```c
void * __alloc_size(2)
devm_kmalloc(struct device *dev, size_t size, gfp_t gfp);
```

لما الـ device بيتحرر أو الـ probe بيفشل، الـ kernel بيـ iterate على الـ devres list الخاصة بالـ device ويـ call الـ release function المسجلة مع كل allocation. مفيش حاجة يعملها الـ driver يدوياً.

الـ `devm_kzalloc` هو مجرد wrapper:
```c
static inline void *devm_kzalloc(struct device *dev, size_t size, gfp_t gfp)
{
    return devm_kmalloc(dev, size, gfp | __GFP_ZERO);
}
```

**الـ `__GFP_ZERO`** بيضمن إن الـ struct بيبدأ بـ zeros، وده بيمنع bugs من uninitialized pointers.

#### الحل
```c
static int spi_sensor_probe(struct spi_device *spi)
{
    struct sensor_priv *priv;

    /* devm_kzalloc: automatic cleanup when device is removed */
    priv = devm_kzalloc(&spi->dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    /* لو init_channels فشلت، الـ kernel هيحرر priv automatically */
    if (init_channels(priv))
        return -EIO; /* clean exit, no leak */

    spi_set_drvdata(spi, priv);
    return 0;
}

/* مش محتاجين remove() لأن devm هيتكفل بكل حاجة */
```

للتأكيد ممكن نستخدم `/sys/kernel/debug/devres` لو `CONFIG_DEBUG_DEVRES` enabled:
```bash
cat /sys/kernel/debug/devices/spi0.0/devres
```

#### الدرس المستفاد
أي struct بيتعمل في `probe()` ومحتاج يعيش طول عمر الـ device، استخدم دايماً `devm_kzalloc`. الـ `kmalloc` العادي مكانه في الـ code اللي بيدير lifetime بنفسه صراحةً.

---

### السيناريو 2: crash عند unbind الـ I2C على STM32MP1

#### العنوان
**Use-after-free** بسبب `devm_kstrdup` و manual `kfree`

#### السياق
بنطور IoT sensor node على **STM32MP1** بيقرأ temperature من sensor **I2C**. الـ driver بيحفظ اسم الـ sensor كـ string عشان يطلعه في sysfs. مع testing الـ hotplug (bind/unbind عن طريق sysfs)، الـ system بيعمل **kernel oops** عند كل unbind.

#### المشكلة
الـ developer استخدم `devm_kstrdup` صح، لكن بعدين حاول يحرره يدوياً بـ `kfree` في الـ remove function، وده بيخلي الـ devres framework يحاول يحرره مرة تانية.

```c
static int temp_sensor_probe(struct i2c_client *client)
{
    struct sensor_data *data;
    const char *name;

    data = devm_kzalloc(&client->dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;

    /* devm_kstrdup — managed string */
    name = devm_kstrdup(&client->dev,
                        of_get_property(client->dev.of_node, "label", NULL),
                        GFP_KERNEL);
    data->name = name;
    /* ... */
    return 0;
}

static void temp_sensor_remove(struct i2c_client *client)
{
    struct sensor_data *data = i2c_get_clientdata(client);
    kfree(data->name); /* BUG: double free! devm سيحررها تاني */
}
```

#### التحليل
الـ `devm_kstrdup` من `devres.h`:
```c
char * __malloc
devm_kstrdup(struct device *dev, const char *s, gfp_t gfp);
```

الـ internal implementation بتخزن الـ pointer في الـ devres list الخاصة بالـ device. لما بتعمل `kfree` يدوي على pointer اتخزن في الـ devres list، الـ pointer بيتحرر، لكن الـ devres entry لسه موجود. لما الـ device بيتحرر، الـ devres framework بيحاول يحرر نفس العنوان، وده **double free** وبيؤدي لـ oops.

الـ flow الصح:
```
probe()
  └── devm_kstrdup() → alloc string → add entry to devres list
                                              │
remove() / probe failure                      │
  └── devres_release_all()                    │
        └── iterate list → call release() ───┘
              └── kfree(string) ← مرة واحدة بس
```

#### الحل
```c
static void temp_sensor_remove(struct i2c_client *client)
{
    struct sensor_data *data = i2c_get_clientdata(client);
    /*
     * لا تعمل kfree لأي حاجة اتخصصت بـ devm_*
     * الـ devres framework هيحررها automatically
     */
    (void)data; /* suppress unused warning if needed */
}
```

لو عندك حالة محتاج تحرر الـ string قبل remove، استخدم:
```c
devm_kfree(&client->dev, data->name);
data->name = NULL; /* prevent dangling pointer */
```

الـ `devm_kfree` بتشيل الـ entry من الـ devres list وبتحرره، فمفيش double free.

#### الدرس المستفاد
**القاعدة الذهبية**: أي pointer اتجه من `devm_k*` function، إما تسيبه للـ framework يحرره، أو تستخدم `devm_kfree` صراحةً — مش `kfree` العادي أبداً.

---

### السيناريو 3: فشل partial initialization في driver الـ HDMI على i.MX8

#### العنوان
استخدام **`devres_open_group` / `devres_release_group`** لعمل atomic rollback

#### السياق
بنكتب driver لـ **HDMI** display controller على **i.MX8MQ** لـ Android TV box. الـ probe بيعمل sequence من 6 steps: ioremap، clock، reset، IRQ، DMA، regulator. لو أي step فشل، محتاجين نعمل cleanup للـ steps السابقة بدون ما نمسك كل resource يدوياً.

#### المشكلة
الـ driver بيعمل goto chain طويلة للـ cleanup وبيحصل فيها bugs لما الـ order بيتغير:

```c
static int hdmi_probe(struct platform_device *pdev)
{
    /* ... */
    priv->regs = ioremap(res->start, res->len);
    if (!priv->regs) goto err_regs;

    priv->clk = clk_get(&pdev->dev, "hdmi");
    if (IS_ERR(priv->clk)) goto err_clk;

    /* ... 4 more steps ... */
    return 0;

err_clk:
    iounmap(priv->regs);
err_regs:
    return -EIO;
    /* بيبقى فيه order bugs لما الـ code بيكبر */
}
```

#### التحليل
الـ `devres.h` بيوفر **devres groups** للـ atomic resource management:

```c
void * __must_check devres_open_group(struct device *dev, void *id, gfp_t gfp);
void devres_close_group(struct device *dev, void *id);
void devres_remove_group(struct device *dev, void *id);
int devres_release_group(struct device *dev, void *id);
```

الـ group بيعمل checkpoint في الـ devres list. لو حاجة فشلت، `devres_release_group` بيحرر كل الـ resources اللي اتخصصت بعد الـ `devres_open_group`.

#### الحل
```c
static int hdmi_probe(struct platform_device *pdev)
{
    void *group;
    int ret;

    /* open a devres group — any devm_* calls after this belong to the group */
    group = devres_open_group(&pdev->dev, NULL, GFP_KERNEL);
    if (!group)
        return -ENOMEM;

    /* Step 1: ioremap — managed automatically */
    priv->regs = devm_ioremap_resource(&pdev->dev, res);
    if (IS_ERR(priv->regs)) {
        ret = PTR_ERR(priv->regs);
        goto err;
    }

    /* Step 2: clock */
    priv->clk = devm_clk_get(&pdev->dev, "hdmi");
    if (IS_ERR(priv->clk)) {
        ret = PTR_ERR(priv->clk);
        goto err;
    }

    /* ... steps 3-6 ... */

    /* success: close group, resources stay managed under device */
    devres_close_group(&pdev->dev, group);
    return 0;

err:
    /*
     * release_group frees ALL resources allocated since open_group
     * no need for manual goto chains
     */
    devres_release_group(&pdev->dev, group);
    return ret;
}
```

#### الدرس المستفاد
الـ devres groups بتحل مشكلة الـ goto spaghetti في الـ probe الطويل. استخدمها لما عندك sequence من resources محتاجة تتحرر كـ unit في حالة الفشل.

---

### السيناريو 4: per-CPU counter corruption في driver الـ USB على AM62x

#### العنوان
استخدام **`devm_alloc_percpu`** للـ per-CPU stats في driver الـ USB

#### السياق
بنطور **USB 3.0** host controller driver على **AM62x** (TI) لـ industrial gateway بيشتغل على multi-core processor. الـ driver بيحتاج يحتفظ بـ per-CPU statistics (packet counts, error counts) عشان يتجنب الـ cache line bouncing. الـ developer الأول استخدم pointer عادي لـ global counter وبدأ يحصل race conditions.

#### المشكلة
استخدام global counter مع atomic operations كان بيخلق overhead كبير على الـ SMP system. الـ solution الصح هو per-CPU variables، لكن الـ developer مش عارف يربطها بـ device lifetime.

```c
/* الكود القديم — global counter مع spinlock */
struct usb_stats {
    atomic64_t rx_packets;
    atomic64_t tx_packets;
    atomic64_t errors;
};

static struct usb_stats *g_stats; /* global — مش per-CPU */

static int usb_probe(struct platform_device *pdev)
{
    g_stats = kmalloc(sizeof(*g_stats), GFP_KERNEL); /* مش managed */
    /* ... */
}
```

#### التحليل
الـ `devres.h` بيوفر:

```c
#define devm_alloc_percpu(dev, type) \
    ((typeof(type) __percpu *)__devm_alloc_percpu((dev), sizeof(type), __alignof__(type)))

void __percpu *__devm_alloc_percpu(struct device *dev, size_t size, size_t align);
```

الـ macro بيـ cast الـ result للـ correct percpu pointer type، ويضمن الـ alignment الصح للـ type. الـ `__alignof__(type)` ضروري عشان الـ percpu infrastructure تشتغل صح.

#### الحل
```c
struct usb_cpu_stats {
    u64 rx_packets;
    u64 tx_packets;
    u64 errors;
};

static int usb_probe(struct platform_device *pdev)
{
    struct usb_priv *priv;
    struct usb_cpu_stats __percpu *stats;

    priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    /* allocate per-CPU stats — freed automatically on device removal */
    stats = devm_alloc_percpu(&pdev->dev, struct usb_cpu_stats);
    if (!stats)
        return -ENOMEM;

    priv->stats = stats;

    /* ... rest of probe ... */
    return 0;
}

/* في الـ interrupt handler */
static irqreturn_t usb_irq(int irq, void *dev_id)
{
    struct usb_priv *priv = dev_id;
    struct usb_cpu_stats *stats;

    /* no locking needed — each CPU writes its own counter */
    stats = this_cpu_ptr(priv->stats);
    stats->rx_packets++;

    return IRQ_HANDLED;
}

/* لقراءة الـ total */
static u64 get_total_rx(struct usb_priv *priv)
{
    u64 total = 0;
    int cpu;

    for_each_possible_cpu(cpu)
        total += per_cpu_ptr(priv->stats, cpu)->rx_packets;

    return total;
}
```

#### الدرس المستفاد
`devm_alloc_percpu` بيجمع ميزتين: الـ zero-overhead per-CPU access، وإدارة الـ lifetime التلقائية. استخدمه في كل driver بيحتاج per-CPU state على multi-core platforms زي AM62x.

---

### السيناريو 5: resource leak في driver الـ UART على Allwinner H616

#### العنوان
استخدام **`devm_add_action_or_reset`** لتسجيل custom cleanup لـ hardware state

#### السياق
بنكتب driver لـ custom **UART** controller على **Allwinner H616** لـ automotive ECU. الـ UART controller بيحتاج sequence خاصة للـ shutdown: لازم نوقف الـ DMA، نفلش الـ FIFO، وبعدين نعمل hardware reset. مفيش `devm_*` جاهز لده، والـ developer حاول يعمل كل ده في `remove()` لكن بيفوته حالة فشل الـ probe.

#### المشكلة
لو الـ probe فشل بعد ما عمل hardware initialization، الـ hardware بيفضل في state غلط حتى بعد ما الـ driver يتحمل من جديد.

```c
static int uart_h616_probe(struct platform_device *pdev)
{
    struct uart_priv *priv;

    priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    /* hardware init */
    uart_h616_hw_init(priv);

    /* لو الخطوات الجاية فشلت، الـ hardware مش هيتنظف */
    if (request_irq(...))
        return -EBUSY; /* hardware still running! */

    return 0;
}

static void uart_h616_remove(struct platform_device *pdev)
{
    struct uart_priv *priv = platform_get_drvdata(pdev);
    uart_h616_hw_shutdown(priv); /* مش بيتنادى لو probe فشل */
}
```

#### التحليل
الـ `devres.h` بيوفر:

```c
int __devm_add_action(struct device *dev, void (*action)(void *), void *data, const char *name);
#define devm_add_action(dev, action, data) \
    __devm_add_action(dev, action, data, #action)

static inline int __devm_add_action_or_reset(...)
{
    int ret;
    ret = __devm_add_action(dev, action, data, name);
    if (ret)
        action(data); /* لو التسجيل فشل، نفذ الـ action فوراً */
    return ret;
}
#define devm_add_action_or_reset(dev, action, data) \
    __devm_add_action_or_reset(dev, action, data, #action)
```

الـ `devm_add_action_or_reset` بيسجل custom cleanup function في الـ devres stack. لو التسجيل نفسه فشل (OOM مثلاً)، الـ action بتتنفذ فوراً عشان الـ resource اللي اتخصص قبل كده يتحرر.

#### الحل
```c
static void uart_h616_hw_cleanup(void *data)
{
    struct uart_priv *priv = data;

    /* ordered shutdown sequence */
    uart_h616_stop_dma(priv);
    uart_h616_flush_fifo(priv);
    uart_h616_hw_reset(priv);
}

static int uart_h616_probe(struct platform_device *pdev)
{
    struct uart_priv *priv;
    int ret;

    priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    /* init hardware */
    uart_h616_hw_init(priv);

    /*
     * register cleanup action immediately after hw init
     * if any subsequent step fails, cleanup runs automatically
     */
    ret = devm_add_action_or_reset(&pdev->dev, uart_h616_hw_cleanup, priv);
    if (ret)
        return ret; /* cleanup already ran inside add_action_or_reset */

    /* now safe to continue — cleanup is registered */
    ret = devm_request_irq(&pdev->dev, priv->irq, uart_h616_irq,
                           0, "uart-h616", priv);
    if (ret)
        return ret; /* uart_h616_hw_cleanup will be called by devres */

    platform_set_drvdata(pdev, priv);
    return 0;
}

/* remove() مش محتاجة تعمل cleanup يدوي — devres هيتكفل بكل حاجة */
```

للتحقق إن الـ action اتسجلت صح:
```bash
# enable CONFIG_DEBUG_DEVRES في .config
echo "uart-h616" > /sys/bus/platform/drivers/uart-h616/unbind
# تتبع الـ dmesg لتأكيد إن cleanup اتنادت
dmesg | grep "uart_h616_hw_cleanup"
```

ولو محتاج تتأكد إن الـ action موجودة قبل ما تعمل unbind:
```c
if (devm_is_action_added(&pdev->dev, uart_h616_hw_cleanup, priv))
    dev_info(&pdev->dev, "cleanup action is registered\n");
```

#### الدرس المستفاد
`devm_add_action_or_reset` هو الأداة الصح لأي hardware state محتاج cleanup مخصص. سجله فوراً بعد الـ init، ومتاعتمدش على `remove()` وحدها — لأن الـ probe failure مش بتنادي `remove()`.
## Phase 7: مصادر ومراجع

### مصادر رسمية من kernel.org

| المصدر | الرابط |
|--------|--------|
| **التوثيق الرسمي للـ devres** | [docs.kernel.org/driver-api/driver-model/devres.html](https://docs.kernel.org/driver-api/driver-model/devres.html) |
| **الملف النصي القديم** | [kernel.org/doc/Documentation/driver-model/devres.txt](https://www.kernel.org/doc/Documentation/driver-model/devres.txt) |
| **Driver Implementer's API Guide** | [docs.kernel.org/driver-api/index.html](https://docs.kernel.org/driver-api/index.html) |
| **Driver Model Index** | [docs.kernel.org/driver-api/driver-model/index.html](https://docs.kernel.org/driver-api/driver-model/index.html) |
| **Device Drivers Infrastructure** | [docs.kernel.org/driver-api/infrastructure.html](https://docs.kernel.org/driver-api/infrastructure.html) |

مسارات التوثيق في الـ source tree:

```
Documentation/driver-api/driver-model/devres.rst
Documentation/driver-api/driver-model/devres.txt   (قديم)
drivers/base/devres.c                               (التنفيذ الأساسي)
include/linux/device/devres.h                       (الـ header)
lib/devres.c                                        (devres للـ I/O resources)
```

---

### مقالات LWN.net

دي أهم المقالات اللي غطّت الـ devres من لما اتقدّمت في 2007:

#### المقالات الأساسية

| المقال | الرابط |
|--------|--------|
| **Managed device resources** — أول مقالة غطّت الفكرة قبل الـ merge | [lwn.net/Articles/215861/](https://lwn.net/Articles/215861/) |
| **Device resource management** — التغطية الرسمية لما اتضافت للـ kernel | [lwn.net/Articles/215996/](https://lwn.net/Articles/215996/) |
| **The managed resource API** — شرح مفصّل للـ API بعد الإضافة | [lwn.net/Articles/222860/](https://lwn.net/Articles/222860/) |
| **A debugfs file system for managed resources** — إضافة debugfs لمتابعة الـ devres | [lwn.net/Articles/606554/](https://lwn.net/Articles/606554/) |
| **devres: introduce API "devm_kmemdup"** — تفاصيل إضافة `devm_kmemdup` | [lwn.net/Articles/594032/](https://lwn.net/Articles/594032/) |
| **A fresh look at the kernel's device model** — مراجعة شاملة للـ device model | [lwn.net/Articles/645810/](https://lwn.net/Articles/645810/) |

#### الـ Kernel Development Coverage

**الـ** LWN weekly kernel development summary اللي غطّت الـ 2.6.21 merge window:

- [lwn.net/Articles/215235/](https://lwn.net/Articles/215235/) — **الـ** kernel development summary اللي فيها الـ devres

---

### الـ Kernel Commits المهمة

#### الـ commit الأصلي — إدخال الـ devres في 2.6.21

```
Author: Tejun Heo <htejun@gmail.com>
Date:   Sat Feb 10 10:20:09 2007 +0900
Subject: devres: device resource management
```

**الـ** patch الأصلي على LKML:
- [lkml.org/lkml/2007/1/20/6](https://lkml.org/lkml/2007/1/20/6) — `[PATCH 1/7] devres: device resource management`

#### commits مهمة تانية

| الـ commit | الوصف |
|------------|-------|
| `devm_add_action_or_reset()` | أُضيفت في 2015 عن طريق Sudip Mukherjee — [linux.kernel.narkive.com](https://linux.kernel.narkive.com/t2eaUpTx/patch-1-2-devm-add-helper-devm-add-action-or-reset) |
| `devm_ioremap_resource()` | دمجت `request_mem_region` + `ioremap` في call واحدة |
| `devm_kmemdup()` | [lwn.net/Articles/594032/](https://lwn.net/Articles/594032/) |

للبحث في تاريخ الـ commits:

```bash
# في الـ kernel source
git log --oneline --all -- drivers/base/devres.c
git log --oneline --all -- include/linux/device/devres.h
git log --grep="devres" --oneline | head -30
```

---

### نقاشات Mailing List

| النقاش | الرابط |
|--------|--------|
| `devm_kmalloc()` for DMA — هل الـ alignment صح؟ | [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-dmaengine/patch/894e8867-57b5-39ab-4dbd-3761d485cb62@arm.com/) |
| Documentation: devres: Add devm_kmalloc() et al | [lists.linaro.org](https://lists.linaro.org/pipermail/linaro-kernel/2014-June/015756.html) |
| devm_add_action_or_reset patch discussion | [linux.kernel.narkive.com](https://linux.kernel.narkive.com/t2eaUpTx/patch-1-2-devm-add-helper-devm-add-action-or-reset) |

---

### kernelnewbies.org

**الـ** devres اتضافت في الـ **Linux 2.6.21** — صفحة التغييرات:

- [kernelnewbies.org/Linux_2_6_21](https://kernelnewbies.org/Linux_2_6_21)

**الصفحة دي** بتوضح إن الـ devres كانت من أبرز features الـ 2.6.21 وبتقول:

> "devres: optional subsystem for drivers that greatly simplifies driver housekeeping if you need to acquire+map then later unmap+free a bunch of device-related resources (MMIO, PIO, IRQs, iomap, PCI, DMA resources)."

---

### elinux.org

**الـ** elinux.org مفيهاش صفحة مخصصة للـ devres، لكن في مصادر مفيدة:

- [elinux.org/Linux_Kernel_Resources](https://elinux.org/Linux_Kernel_Resources) — مصادر عامة للـ kernel
- [elinux.org/Device_Tree_Reference](https://elinux.org/Device_Tree_Reference) — مرتبط بالـ `devm_of_iomap()` الموجودة في الـ header

---

### محاضرات وعروض تقديمية

#### Haifux Linux Club

**محاضرة ممتازة** من Eli Billauer تشرح الـ devres بأمثلة عملية:

| المصدر | الرابط |
|--------|--------|
| صفحة المحاضرة | [haifux.org/lectures/323/](http://www.haifux.org/lectures/323/) |
| الـ PDF المباشر | [haifux.org/lectures/323/haifux-devres.pdf](http://www.haifux.org/lectures/323/haifux-devres.pdf) |

**الـ** PDF بيغطي:
- ليه الـ devres اتعملت
- مقارنة الكود قبل وبعد
- كيفية عمل `devm_add_action()` لتنظيف custom resources
- أمثلة على الـ "Undo" pattern

---

### كتب مرجعية

#### Linux Device Drivers 3rd Edition (LDD3)

**الـ** LDD3 اتكتب في 2005 على kernel 2.6.10 — قبل الـ devres بسنتين — فمش بيغطي الـ `devm_*` functions بشكل مباشر، لكن:

| الفصل | الصلة بالـ devres |
|-------|------------------|
| **Chapter 8: Allocating Memory** | الأساس اللي بيتبنى عليه `devm_kmalloc` |
| **Chapter 9: Communicating with Hardware** | الـ `ioremap` التقليدي اللي بيستبدله `devm_ioremap_resource` |
| **Chapter 12: PCI Drivers** | الـ `pcim_*` functions المبنية على devres |

تحميل مجاني:
- [lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)
- [bootlin.com/doc/books/ldd3.pdf](https://bootlin.com/doc/books/ldd3.pdf)

#### Linux Kernel Development — Robert Love (3rd Edition)

- **Chapter 17: Devices and Modules** — يشرح الـ device model الأساسي
- **Chapter 19: Portability** — يشرح memory management اللي بيتبنى عليه الـ devres
- الكتاب مش بيغطي الـ devres directly لكنه ضروري لفهم الـ `struct device` lifecycle

#### Linux Device Drivers Development — John Madieu (Packt)

كتاب أحدث بيغطي الـ `devm_*` API بشكل مباشر:
- [oreilly.com — Device-managed resources chapter](https://www.oreilly.com/library/view/linux-device-drivers/9781785280009/a2b401fc-d531-44c1-ab5c-a370075facad.xhtml)
- [packtpub.com — devres section](https://subscription.packtpub.com/book/cloud-and-networking/9781785280009/11/ch11lvl1sec67/device-managed-resources-devres)

#### Embedded Linux Primer — Christopher Hallinan

- **Chapter 16: Kernel Debugging Techniques** — يذكر الـ devres في سياق debugging resource leaks
- مفيد لفهم الـ context العملي في الـ embedded systems

---

### مصادر إضافية

#### الـ Kernel Source Code مباشرة

```bash
# الملفات الأساسية للـ devres
drivers/base/devres.c       # core implementation
lib/devres.c                # I/O-related devres helpers
include/linux/device/devres.h   # public API header
```

على GitHub:
- [github.com/torvalds/linux/blob/master/drivers/base/devres.c](https://github.com/torvalds/linux/blob/master/drivers/base/devres.c)
- [github.com/torvalds/linux/blob/master/lib/devres.c](https://github.com/torvalds/linux/blob/master/lib/devres.c)

#### Doxygen Documentation

- [docs.huihoo.com — devres.c File Reference](https://docs.huihoo.com/doxygen/linux/kernel/3.7/drivers_2base_2devres_8c.html)

#### Network World Article

- [networkworld.com — Kernel Space: Device-resource management](https://www.networkworld.com/article/837526/smb-kernel-space-device-resource-management.html)

---

### كلمات البحث

لو حاب تلاقي معلومات أكتر، دي أهم الـ search terms:

```
devres linux kernel
devm_ functions linux driver
managed device resources linux
device resource management kernel
devm_kmalloc linux driver
devm_ioremap_resource linux
devm_add_action linux kernel
devres_alloc devres_add kernel
linux driver cleanup devm
automatic resource management linux driver
```

**للبحث في LKML:**

```
site:lore.kernel.org devres
site:lkml.org devres devm
```

**للبحث في الـ kernel source:**

```bash
grep -r "devm_" Documentation/driver-api/ --include="*.rst"
grep -rn "devres_alloc" drivers/ --include="*.c" | head -20
```
## Phase 8: Writing simple module

### الهدف

هنعمل module بيستخدم **kprobe** عشان يعمل hook على `devm_kmalloc` — أكتر function استخداماً في devres API. كل ما driver يطلب memory managed، الـ kprobe بتاعتنا هتطبع اسم الـ device والحجم المطلوب.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_devres.c — hook devm_kmalloc to trace managed allocations
 */

#include <linux/module.h>      /* module_init, module_exit, MODULE_* macros    */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe, etc.         */
#include <linux/device.h>      /* struct device, dev_name()                    */
#include <linux/printk.h>      /* pr_info()                                    */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Learner");
MODULE_DESCRIPTION("Trace devm_kmalloc calls via kprobe");

/* ------------------------------------------------------------------
 * pre_handler — runs just before devm_kmalloc executes.
 *
 * Signature of devm_kmalloc:
 *   void *devm_kmalloc(struct device *dev, size_t size, gfp_t gfp);
 *
 * On x86-64 the arguments sit in registers:
 *   rdi = dev, rsi = size, rdx = gfp
 * ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Extract first two arguments from calling-convention registers */
    struct device *dev  = (struct device *)regs->di;  /* arg1: device ptr  */
    size_t         size = (size_t)regs->si;            /* arg2: bytes asked */

    /* Guard against NULL — early-boot or weird callers may pass NULL */
    if (!dev)
        return 0;

    pr_info("devm_kmalloc: dev=%s  size=%zu\n",
            dev_name(dev), size);

    return 0;  /* non-zero would abort the probed function — we never want that */
}

/* ------------------------------------------------------------------
 * post_handler — runs after devm_kmalloc returns (optional).
 * We use it to log the returned pointer stored in rax.
 * ------------------------------------------------------------------ */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /* On x86-64 return value is in rax */
    void *ret = (void *)regs->ax;

    pr_info("devm_kmalloc: returned ptr=%px  (%s)\n",
            ret, ret ? "ok" : "FAILED");
}

/* ------------------------------------------------------------------
 * kprobe descriptor — we probe by symbol name so no hard-coded
 * address is needed; the kernel resolves it at registration time.
 * ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "devm_kmalloc",   /* exact exported symbol to probe   */
    .pre_handler  = handler_pre,     /* called before the function body  */
    .post_handler = handler_post,    /* called after the function body   */
};

/* ------------------------------------------------------------------
 * module_init — register the kprobe.
 * If registration fails we return the error so insmod fails cleanly.
 * ------------------------------------------------------------------ */
static int __init kprobe_devres_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("kprobe planted on devm_kmalloc at %px\n", kp.addr);
    return 0;
}

/* ------------------------------------------------------------------
 * module_exit — MUST unregister before the module is unloaded.
 * لو منزلناش الـ kprobe قبل ما الـ module يتشال من الذاكرة،
 * الـ handler هيبقى بيشاور على code مش موجود → kernel panic.
 * ------------------------------------------------------------------ */
static void __exit kprobe_devres_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("kprobe on devm_kmalloc removed\n");
}

module_init(kprobe_devres_init);
module_exit(kprobe_devres_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/module.h` | الـ macros الأساسية لأي kernel module |
| `linux/kprobes.h` | تعريف `struct kprobe` وكل الـ register/unregister API |
| `linux/device.h` | تعريف `struct device` و `dev_name()` عشان نطبع اسم الـ device |
| `linux/printk.h` | `pr_info()` / `pr_err()` للـ logging |

---

#### الـ `handler_pre`

الـ kprobe بتوقف التنفيذ **قبل** ما `devm_kmalloc` تشتغل وبتدينا `struct pt_regs` اللي بيحتوي على قيم الـ registers لحظة الاستدعاء. على x86-64 الـ calling convention بيحط الـ arguments في `rdi`، `rsi`، `rdx`، فبنقرأ الـ `dev` و`size` منهم مباشرةً.

---

#### الـ `handler_post`

بتشتغل **بعد** ما الـ function ترجع، ونقرأ `regs->ax` عشان نشوف الـ pointer اللي اتعمل allocate فعلاً — مفيد نعرف لو الـ allocation نجحت أو فشلت.

---

#### الـ `struct kprobe`

بنحدد فيها `symbol_name` بالاسم بدل العنوان الثابت، والـ kernel بيحل العنوان وقت `register_kprobe`. لو الـ symbol مش exported أو CONFIG_KPROBES مش مفعّل، الـ registration بترجع error.

---

#### الـ `module_init` و `module_exit`

- **init**: بيسجل الـ kprobe ولو فشل بيرجع الـ error عشان `insmod` يفشل بشكل نضيف.
- **exit**: لازم نعمل `unregister_kprobe` قبل ما الـ module يتحمّل من الذاكرة — لو الـ handler pointer اتشال والـ probe لسه registered، أي استدعاء لـ `devm_kmalloc` هيودي في kernel panic.

---

### Makefile للتجربة

```makefile
obj-m += kprobe_devres.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

```bash
# تحميل الـ module
sudo insmod kprobe_devres.ko

# شوف الـ output في kernel log
sudo dmesg -w

# فك التحميل
sudo rmmod kprobe_devres
```

---

### مثال على الـ output المتوقع

```
[  142.331204] kprobe planted on devm_kmalloc at ffffffffc0a12340
[  143.002918] devm_kmalloc: dev=0000:00:1f.3  size=128
[  143.002920] devm_kmalloc: returned ptr=ffff888103a4c000  (ok)
[  143.115442] devm_kmalloc: dev=leds  size=48
[  143.115443] devm_kmalloc: returned ptr=ffff888103a4c080  (ok)
```

كل سطرين مع بعض = allocation واحدة: الأول قبلها (اسم الـ device والحجم)، التاني بعدها (الـ pointer الراجع).
