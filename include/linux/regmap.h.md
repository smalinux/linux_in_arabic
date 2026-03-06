## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

**الـ regmap** — Register Map Abstraction — موجود في MAINTAINERS تحت اسم "REGISTER MAP ABSTRACTION"، maintained بواسطة Mark Brown. الـ subsystem كله تحت `drivers/base/regmap/` والـ public API في `include/linux/regmap.h`.

---

### القصة: المشكلة اللي بتحلها

تخيل إنك بتصمم سواق (driver) لـ chip صوت — مثلاً codec زي WM8731. الـ chip دي فيها 50 register، كل register ليه address وقيمة. Driver بتاعك محتاج:
- يقرأ register
- يكتب register
- يغير بعض bits من غير ما يمس الباقي
- يكش الـ cache لو الـ chip اتعملها reset
- يتعامل مع الـ chip سواء كانت على I2C أو SPI أو مباشرة MMIO

**قبل regmap**: كل driver كان بيكتب كودة نفسه لـ read/write، كودة لـ cache، كودة للـ locking — كل ده بتكرار في المئات من الـ drivers.

**بعد regmap**: Driver بيعمل `regmap_init_i2c()` مرة واحدة بـ config بسيطة، وبعدين يستخدم `regmap_read()` و`regmap_write()` — مش فارق معاه الـ bus من تحت.

---

### الفكرة بالبساطة الكاملة

**الـ regmap زي مترجم فوري بين الـ driver والـ hardware.**

```
Driver Code
    |
    | regmap_read(map, REG_VOLUME, &val)
    |
  [ regmap layer ]
    |  - check cache first
    |  - apply locking
    |  - format address & value
    |  - choose right bus
    |
  +-------+-------+-------+-------+
  |       |       |       |       |
  I2C    SPI    MMIO   SoundWire  ...
```

Driver ما يعرفش ولا يهتمش بالـ bus. الـ regmap هو اللي بيترجم "اقرأ register رقم 5" لـ I2C transaction أو SPI transaction أو memory read.

---

### ليه ده مهم جداً؟

| المشكلة | الحل في regmap |
|---------|---------------|
| كل driver بيكرر كود I2C/SPI read/write | `regmap_init_i2c/spi()` مرة واحدة |
| الـ registers بتترقم بشكل غريب أو فيها paging | `regmap_range_cfg` للـ virtual ranges |
| بعض registers لا يُقرأ من hardware كل مرة (volatile/precious) | callbacks: `volatile_reg`, `precious_reg` |
| الأجهزة بتبدأ من suspend وكل registers اتمسحت | `regcache_sync()` تعيد الكتابة من cache |
| update-bits (read-modify-write) محتاج locking دقيق | `regmap_update_bits()` thread-safe تلقائياً |
| debugging: عايز تشوف values كل registers | regmap debugfs interface تلقائي |

---

### الـ Layers الجوانية

```
┌─────────────────────────────────────────┐
│            Driver (consumer)            │
│   regmap_read / regmap_write / etc.     │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│         regmap core (regmap.c)          │
│  - locking  - format  - range checks    │
│  - debugfs  - async writes              │
└──────────┬───────────────┬──────────────┘
           │               │
┌──────────▼──────┐  ┌─────▼──────────────┐
│  regcache layer │  │   Bus backends      │
│  (rbtree/flat/  │  │  i2c / spi / mmio   │
│   maple tree)   │  │  sdw / spmi / ...   │
└─────────────────┘  └─────────────────────┘
```

---

### المحتوى الكامل لـ `regmap.h`

الـ header ده هو الـ public contract للـ subsystem. فيه:

#### 1. الـ Configuration — `struct regmap_config`
أهم struct في الملف. الـ driver بيملاها قبل ما يعمل `regmap_init_*`. بتحدد:
- `reg_bits` / `val_bits` — حجم الـ address والقيمة (مثلاً 8-bit address, 16-bit value)
- `cache_type` — هل نستخدم cache وأيه نوعه (RBTREE / FLAT / MAPLE / NONE)
- `volatile_reg` / `precious_reg` — callbacks بتقول للـ regmap أي registers لا تتخزن في cache
- `reg_defaults[]` — قيم البداية للـ registers بعد power-on-reset
- `writeable_reg` / `readable_reg` — access control
- `max_register` — أكبر عنوان valid
- `read_flag_mask` / `write_flag_mask` — bits بتتضاف على الـ address وقت SPI (شائع جداً في SPI حيث bit7 بيحدد read/write)

#### 2. الـ Bus Description — `struct regmap_bus`
الـ hardware backend. كل bus (i2c, spi, mmio...) بيوفر implementation للـ function pointers دي:
- `write`, `read`, `gather_write`, `async_write`
- `reg_write`, `reg_read` — single register operations
- `reg_noinc_write/read` — للـ FIFO registers اللي ما بيتزيدش address

#### 3. الـ Init Macros
لكل bus نوع، فيه macro بيعمل regmap:

```c
/* I2C device */
map = devm_regmap_init_i2c(client, &my_config);

/* SPI device */
map = devm_regmap_init_spi(spi, &my_config);

/* Memory-mapped registers */
map = devm_regmap_init_mmio(dev, base_addr, &my_config);
```

الـ `devm_` prefix معناها الـ map هيتحرر تلقائياً لما الـ device يتمسح.

الـ buses المدعومة:
`I2C`, `SPI`, `MMIO`, `SPMI`, `SoundWire (SDW)`, `SlimBus`, `I3C`, `AC97`, `MDIO`, `W1`, `FSI`, `SPI-AVMM`

#### 4. الـ Core API

```c
/* Basic read/write */
regmap_read(map, reg, &val);
regmap_write(map, reg, val);

/* Bulk operations */
regmap_bulk_read(map, reg, buf, count);
regmap_bulk_write(map, reg, buf, count);

/* Atomic read-modify-write */
regmap_update_bits(map, reg, mask, val);
regmap_set_bits(map, reg, bits);
regmap_clear_bits(map, reg, bits);

/* Polling */
regmap_read_poll_timeout(map, addr, val, cond, sleep_us, timeout_us);
```

#### 5. الـ Register Fields — `struct reg_field` + `struct regmap_field`
بدل ما تتعامل مع الـ register كوحدة كاملة، تقدر تعرف "field" بداخله:

```c
/* REG_CTRL بيته 3-5 هو الـ volume */
static const struct reg_field vol_field = REG_FIELD(REG_CTRL, 3, 5);

field = devm_regmap_field_alloc(dev, map, vol_field);
regmap_field_write(field, 7);   /* بيكتب bits 3-5 بس */
regmap_field_read(field, &val); /* بيقرأ bits 3-5 بس */
```

#### 6. الـ IRQ Controller — `struct regmap_irq_chip`
كتير من الـ PMICs والـ audio codecs عندها interrupt controller جواها — مجموعة registers للـ status والـ mask. الـ regmap-irq بيحول الـ registers دي لـ Linux IRQ domain كامل تلقائياً بدون ما الـ driver يكتب IRQ handler من الصفر.

#### 7. الـ Paged/Indirect Registers — `struct regmap_range_cfg`
بعض الـ chips فيها address space أكبر من اللي ينفع يتوصله مباشرة. مثلاً chip بـ 8-bit address bus بس عندها 1000 register — بتستخدم page register. الـ `regmap_range_cfg` بيعمل virtual address space والـ regmap بيتولى كتابة الـ page selector تلقائياً قبل أي access.

#### 8. الـ Cache Layer
أنواع الـ cache:

| النوع | الاستخدام |
|-------|-----------|
| `REGCACHE_NONE` | لا cache — كل read/write يروح للـ hardware |
| `REGCACHE_FLAT` | array ثابت — سريع، لكن بياكل memory لو sparse |
| `REGCACHE_FLAT_S` | sparse flat — الأفضل للـ use cases الجديدة |
| `REGCACHE_MAPLE` | maple tree — الـ default المفضل حالياً |
| `REGCACHE_RBTREE` | legacy — أقل كفاءة من maple في معظم الحالات |

الـ cache بيحل مشكلة الـ suspend/resume: لما الجهاز يرجع من suspend وكل registers اتمسحت، الـ driver بيعمل `regcache_sync(map)` وبتتكتب كل القيم المحفوظة تلقائياً.

---

### الملفات المكونة للـ Subsystem

**الـ Core:**
| الملف | الدور |
|-------|-------|
| `drivers/base/regmap/regmap.c` | الـ core — الـ init، read/write، update_bits، locking |
| `drivers/base/regmap/regcache.c` | الـ cache layer العام |
| `drivers/base/regmap/regcache-flat.c` | الـ flat array cache |
| `drivers/base/regmap/regcache-maple.c` | الـ maple tree cache (الأفضل حالياً) |
| `drivers/base/regmap/regcache-rbtree.c` | الـ rbtree cache (legacy) |
| `drivers/base/regmap/regmap-irq.c` | الـ generic IRQ controller |
| `drivers/base/regmap/regmap-debugfs.c` | debugfs interface |
| `drivers/base/regmap/internal.h` | internal structs مش exposed للـ drivers |

**الـ Bus Backends:**
| الملف | الـ Bus |
|-------|--------|
| `regmap-i2c.c` | I2C |
| `regmap-spi.c` | SPI |
| `regmap-mmio.c` | Memory-Mapped I/O |
| `regmap-spmi.c` | SPMI (Qualcomm power management bus) |
| `regmap-sdw.c` / `regmap-sdw-mbq.c` | SoundWire |
| `regmap-slimbus.c` | SLIMbus (audio) |
| `regmap-i3c.c` | I3C |
| `regmap-ac97.c` | AC'97 (legacy audio) |
| `regmap-mdio.c` | MDIO (network PHY) |
| `regmap-w1.c` | 1-Wire |
| `regmap-fsi.c` | FSI (IBM POWER) |
| `regmap-spi-avmm.c` | Intel SPI to AVMM bridge |

**الـ Header الرئيسي:**
- `include/linux/regmap.h` — الملف ده نفسه، الـ public API الكامل

**الـ Documentation:**
- `Documentation/devicetree/bindings/regmap/` — DT bindings

---

### ملفات القارئ لازم يعرفها

- `drivers/base/regmap/internal.h` — الـ `struct regmap` الداخلية (مش مكشوفة للـ drivers)
- `drivers/base/regmap/regmap.c` — implementation الـ core functions
- `drivers/base/regmap/regcache.c` — فهم الـ cache sync/bypass
- أي driver يستخدم regmap كمثال حي: `drivers/mfd/`, `drivers/regulator/`, `sound/soc/codecs/`
## Phase 2: شرح الـ regmap Framework

---

### المشكلة: ليه الـ regmap موجود أصلاً؟

لو بتكتب driver لـ audio codec زي WM8731 أو PMIC زي MAX77686، هتلاقي إن كل driver بيعمل نفس الحاجات:

- بيبعت transaction على I2C أو SPI عشان يقرأ/يكتب register
- بيعمل caching للـ register values عشان ميرجعش للـ hardware في كل قراءة
- بيعمل masking وshifting للـ bits جوه الـ register
- بيتعامل مع interrupts من الـ chip

النتيجة كانت إن كل driver كان بيكتب نفس الكود — من scratch — مئات المرات. الـ commit اللي أضاف الـ regmap (2011) من Wolfson Microelectronics وصف المشكلة بالظبط: **duplicated boilerplate** في كل driver.

الـ bugs كمان كانت بتتكرر: race conditions في الـ read-modify-write، cache inconsistency بعد suspend/resume، endianness مش مظبوطة.

---

### الحل: Unified Register Map Abstraction

الـ **regmap** هو abstraction layer جوه الـ kernel بيوفر:

1. **واجهة موحدة** للـ read/write بغض النظر عن الـ bus (I2C, SPI, MMIO, SoundWire, إلخ)
2. **register cache** مدمج بأنواع مختلفة (rbtree, flat, maple tree)
3. **field-level API** للتعامل مع bits داخل الـ register
4. **IRQ controller** generic مبني على الـ regmap نفسه
5. **debugfs integration** تلقائي لكل map

الـ driver بيوصف الـ hardware (عدد الـ bits، أي registers volatile، إلخ) وبيسيب الـ regmap يعمل الباقي.

---

### البنية الكبيرة: مكانه في الـ kernel

```
┌────────────────────────────────────────────────────────────────┐
│                    Consumer Drivers Layer                       │
│  audio/wm8731.c   mfd/max77686.c   gpio/tps65910.c   ...      │
│                                                                │
│   regmap_read()  regmap_write()  regmap_update_bits()  ...    │
└────────────────────────┬───────────────────────────────────────┘
                         │  unified API
┌────────────────────────▼───────────────────────────────────────┐
│                   regmap Core (drivers/base/regmap/)            │
│                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │  regmap_read │  │ regmap_write │  │ regmap_update_bits   │ │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘ │
│         │                 │                      │             │
│  ┌──────▼─────────────────▼──────────────────────▼───────────┐ │
│  │              Cache Layer (regcache)                        │ │
│  │   REGCACHE_NONE / RBTREE / FLAT / FLAT_S / MAPLE          │ │
│  └──────────────────────────┬─────────────────────────────────┘ │
│                             │  cache miss → hit hardware        │
│  ┌──────────────────────────▼─────────────────────────────────┐ │
│  │            Bus Abstraction (struct regmap_bus)              │ │
│  └──────────────────────────┬─────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────  │
                              │
        ┌─────────────────────┼──────────────────────┐
        │                     │                      │
┌───────▼──────┐   ┌──────────▼──────┐   ┌──────────▼──────┐
│  regmap-i2c  │   │  regmap-spi     │   │  regmap-mmio    │
│  (I2C xfer)  │   │  (SPI xfer)     │   │  (ioremap I/O)  │
└───────┬──────┘   └──────────┬──────┘   └──────────┬──────┘
        │                     │                      │
┌───────▼──────┐   ┌──────────▼──────┐   ┌──────────▼──────┐
│  I2C Bus     │   │  SPI Bus        │   │  Memory Bus      │
│  Hardware    │   │  Hardware       │   │  (AHB/AXI)       │
└──────────────┘   └─────────────────┘   └─────────────────┘
```

---

### التشبيه الحقيقي: موظف المستودع (Warehouse Manager)

تخيل إنك مدير مصنع وعندك **مستودع** فيه أجزاء (= الـ hardware registers). عندك موظف مسؤول عن المستودع (= regmap).

| الواقع الحقيقي | Kernel Concept |
|---|---|
| مدير المصنع بيطلب جزء | الـ driver بيستدعي `regmap_read()` |
| موظف المستودع بيدور في الأرشيف الورقي أولاً | الـ regmap بيبص في الـ cache أولاً |
| لو مش موجود في الأرشيف، بيروح للرف الفعلي | cache miss → الـ bus transaction الحقيقية |
| موظف بيحدّث الأرشيف بعد كل طلب | الـ regmap بيحدّث الـ cache بعد القراءة |
| بعض الأجزاء "حساسة" — لازم تعدّها يدوياً كل مرة | `volatile_reg` — registers مش بتتكاش |
| أجزاء "ممنوع تجرها من رف لرف" (clear-on-read) | `precious_reg` — ممنوع regmap يقراها لوحده |
| مدير قال "مش هيتصل بالمستودع" (power off) | `regcache_cache_only()` — كل reads من cache |
| مدير رجع وقال "فاكر إيه عندي مش صح" | `regcache_mark_dirty()` + `regcache_sync()` |
| موظف المستودع بيتعامل مع موردين مختلفين (UPS, DHL) | الـ regmap_bus بيتعامل مع I2C/SPI/MMIO |
| الموظف مش محتاج يعرف طريقة الشحن | الـ driver مش محتاج يعرف الـ bus protocol |

**النقطة العميقة:** المدير (driver) مش عارف ولا مهتم إزاي الأجزاء بتتشال — هل عربية أو طيارة. الـ regmap هو اللي بيعزل هذه التعقيدات.

---

### الـ Core Abstraction: `struct regmap`

الـ `struct regmap` نفسها **opaque** — مش exposed في الـ header. الـ driver شايف بس pointer ليها. ده design مقصود: الـ implementation details محمية.

الـ structs اللي بتشوفها في `regmap.h` هي:

```
struct regmap_config    ← وصف الـ hardware (من الـ driver)
struct regmap_bus       ← وصف الـ transport (من الـ bus layer)
struct regmap           ← الـ runtime state (opaque, core فقط)
struct regmap_field     ← sub-register bit field
struct reg_field        ← description of a bit field (static)
struct reg_default      ← default values بعد power-on reset
struct regmap_range_cfg ← paged/indirect register access
struct regmap_irq_chip  ← generic IRQ controller فوق الـ regmap
```

---

### تشريح `struct regmap_config`

ده أهم struct بالنسبة للـ driver writer — بيصف الـ hardware بالكامل:

```c
struct regmap_config {
    const char *name;      /* optional — لو الـ device عنده أكتر من map */

    /* --- Register/Value geometry --- */
    int reg_bits;          /* عرض الـ register address بالـ bits */
    int val_bits;          /* عرض الـ register value بالـ bits */
    int reg_stride;        /* المسافة بين registers متتاليين */
    int reg_shift;         /* shift قبل أي operation */
    unsigned int reg_base; /* offset مضاف لكل address */
    int pad_bits;          /* padding بين reg وval في الـ frame */

    /* --- Access permission callbacks --- */
    bool (*writeable_reg)(struct device *dev, unsigned int reg);
    bool (*readable_reg)(struct device *dev, unsigned int reg);
    bool (*volatile_reg)(struct device *dev, unsigned int reg);  /* no cache */
    bool (*precious_reg)(struct device *dev, unsigned int reg);  /* no auto-read */

    /* --- Optional custom I/O (للـ weird buses) --- */
    int (*reg_read)(void *context, unsigned int reg, unsigned int *val);
    int (*reg_write)(void *context, unsigned int reg, unsigned int val);

    /* --- Locking --- */
    bool fast_io;          /* spinlock بدل mutex (للـ MMIO السريع) */
    bool disable_locking;  /* لو الـ driver بيعمل locking خارجياً */
    regmap_lock lock;      /* custom lock function */
    regmap_unlock unlock;

    /* --- Cache --- */
    enum regcache_type cache_type;  /* NONE, RBTREE, FLAT, MAPLE */
    const struct reg_default *reg_defaults;
    unsigned int num_reg_defaults;

    /* --- Bus flag masks (for SPI read/write distinction) --- */
    unsigned long read_flag_mask;
    unsigned long write_flag_mask;

    /* --- Paged registers --- */
    const struct regmap_range_cfg *ranges;
    unsigned int num_ranges;
};
```

**مثال حقيقي** — driver لـ WM8731 audio codec على I2C:

```c
static const struct regmap_config wm8731_regmap = {
    .reg_bits  = 7,    /* 7-bit register address */
    .val_bits  = 9,    /* 9-bit register value */
    .reg_stride = 1,

    .max_register    = WM8731_RESET,
    .reg_defaults    = wm8731_reg_defaults,
    .num_reg_defaults = ARRAY_SIZE(wm8731_reg_defaults),
    .cache_type      = REGCACHE_RBTREE,

    .volatile_reg = wm8731_volatile,  /* callback: فقط RESET volatile */
};

/* في probe: */
priv->regmap = devm_regmap_init_i2c(i2c, &wm8731_regmap);
```

بعد كده الـ driver بيعمل:
```c
regmap_update_bits(priv->regmap, WM8731_PWR, WM8731_DACPD, 0);
/* = read reg → modify bit → write back, مع cache وlocking تلقائي */
```

---

### الـ `struct regmap_bus`: طبقة الـ Transport

ده الـ interface اللي بيوصل الـ regmap core بالـ physical bus:

```c
struct regmap_bus {
    bool fast_io;               /* spinlock instead of mutex */
    bool free_on_exit;          /* kernel owns this struct */

    /* Raw bulk I/O */
    regmap_hw_write         write;       /* single contiguous buffer */
    regmap_hw_gather_write  gather_write; /* split reg+val buffers */
    regmap_hw_async_write   async_write; /* non-blocking write */
    regmap_hw_read          read;

    /* Single register I/O (simpler path) */
    regmap_hw_reg_write     reg_write;
    regmap_hw_reg_read      reg_read;
    regmap_hw_reg_noinc_write reg_noinc_write; /* FIFO-style */
    regmap_hw_reg_noinc_read  reg_noinc_read;

    /* Metadata */
    u8 read_flag_mask;          /* SPI: set MSB للـ read */
    enum regmap_endian reg_format_endian_default;
    enum regmap_endian val_format_endian_default;
};
```

الـ `regmap-spi.c` مثلاً بيملأ هذا الـ struct بـ callbacks بتستدعي `spi_sync()`. الـ `regmap-i2c.c` بيستخدم `i2c_transfer()`. الـ `regmap-mmio.c` بيستخدم `readl()`/`writel()`.

---

### العلاقة بين الـ Structs

```
Driver (probe function)
       │
       │  fills statically
       ▼
struct regmap_config ──────────────────────────────────────┐
       │                                                   │
       │  passed to                                        │
       ▼                                                   │
regmap_init_i2c() / regmap_init_spi() / regmap_init_mmio() │
       │                                                   │
       │  creates                                          │
       ▼                                                   │
struct regmap  (opaque, heap-allocated)                    │
  ├── config ──────────────────────────────────────────────┘
  ├── bus ──────────────────► struct regmap_bus
  │                            (I2C/SPI/MMIO callbacks)
  ├── cache ────────────────► regcache internal state
  │                            (rbtree / flat array / maple tree)
  ├── lock (mutex/spinlock)
  └── debugfs entry
       │
       │  referenced by
       ▼
struct regmap_field  (allocated separately)
  ├── regmap *map ──────────► back-pointer
  ├── unsigned int mask      (computed from lsb/msb)
  └── unsigned int shift
```

```
struct reg_field  (static descriptor)       struct regmap_field (runtime handle)
  ├── reg: 0x10                   alloc ──►    ├── map: *regmap
  ├── lsb: 4                   ─────────►      ├── mask: 0x30 (bits 5:4)
  └── msb: 5                                   └── shift: 4
```

---

### الـ Cache Layer: `regcache`

الـ cache في الـ regmap مش بس performance optimization — هو ضروري لحالات زي:
- **suspend/resume**: الـ hardware بيفقد state، الـ cache هو اللي بيحفظه
- **atomic context**: لو مش ممكن تعمل I2C transaction، بتقرأ من cache
- **write coalescing**: تكتب في cache وتعمل sync دفعة واحدة

الأنواع المتاحة:

| النوع | الاستخدام | الـ Trade-off |
|---|---|---|
| `REGCACHE_NONE` | لا cache خالص | أبسط، أبطأ |
| `REGCACHE_FLAT` | array ثابت الحجم | سريع جداً، wasteful في sparse maps |
| `REGCACHE_FLAT_S` | sparse flat (جديد) | موصى به للـ new drivers |
| `REGCACHE_RBTREE` | red-black tree | legacy، مناسب للـ low-end + heavy sync |
| `REGCACHE_MAPLE` | maple tree | الأفضل للـ general case الجديد |

**دورة حياة الـ cache في suspend/resume:**

```
Normal operation:
  regmap_write(map, REG_X, val)
    → update cache
    → write to hardware

Suspend:
  regcache_mark_dirty(map)   /* hardware سيفقد state */
  (device goes powerless)

Resume:
  regcache_sync(map)         /* اكتب كل الـ dirty cache entries للـ hardware */
```

**الـ cache_only mode:**
```c
regcache_cache_only(map, true);
/* كل writes بتروح للـ cache فقط — hardware مش موجود (early init مثلاً) */
regcache_cache_only(map, false);
regcache_sync(map);
/* دلوقتي flush الـ cache للـ hardware */
```

---

### الـ `struct regmap_field` والـ Field API

بدل ما تعمل masking وshifting يدوي:

```c
/* بدون regmap_field — boilerplate ممل */
regmap_read(map, REG_CTRL, &val);
val &= ~(0x3 << 4);    /* clear bits 5:4 */
val |= (mode << 4);
regmap_write(map, REG_CTRL, val);
```

الـ `regmap_field` بتعرف الـ field مرة وتستخدمه:

```c
/* التعريف (static) */
static const struct reg_field clk_sel_field = REG_FIELD(REG_CTRL, 4, 5);

/* في probe */
priv->clk_sel = devm_regmap_field_alloc(dev, map, clk_sel_field);

/* الاستخدام — clean وواضح */
regmap_field_write(priv->clk_sel, CLK_MODE_PLL);
regmap_field_read(priv->clk_sel, &current_mode);
```

الـ `REG_FIELD_ID` بيدعم arrays من نفس الـ field في channels مختلفة:

```c
/* مثلاً: نفس الـ field في 4 channels، كل channel على offset 0x10 */
static const struct reg_field gain_field =
    REG_FIELD_ID(CH0_GAIN_REG, 0, 7, 4, 0x10);

/* للوصول لـ channel 2: */
regmap_fields_write(priv->gain, 2, gain_value);
```

---

### الـ `struct regmap_range_cfg`: الـ Paged Registers

بعض الـ ICs عندها register space أكبر من الـ address space المتاحة في الـ protocol. الحل: **paging** (أو banking).

مثال: chip عنده 1024 register بس الـ I2C protocol بيعنون بس 256. الحل:
1. اكتب رقم الـ page في `selector_reg`
2. اقرأ/اكتب في الـ window (0x00..0xFF مثلاً)

الـ `struct regmap_range_cfg` بيصف ده:

```c
static const struct regmap_range_cfg my_chip_ranges[] = {
    {
        .name         = "main",
        .range_min    = 0x000,    /* virtual address start */
        .range_max    = 0x3FF,    /* virtual address end (1024 regs) */
        .selector_reg = 0xFE,     /* page select register */
        .selector_mask = 0x03,    /* bits [1:0] = page number */
        .selector_shift = 0,
        .window_start = 0x00,     /* physical window base */
        .window_len   = 0x100,    /* 256 regs per page */
    },
};
```

الـ regmap بيتعامل مع الـ paging تلقائياً — الـ driver بيكتب `regmap_read(map, 0x2A5, &val)` وهو بيعمل:
1. احسب الـ page: `0x2A5 / 0x100 = 2`
2. اكتب `2` في `selector_reg` (0xFE)
3. اقرأ من `0x2A5 % 0x100 = 0xA5` في الـ window

---

### الـ `struct regmap_irq_chip`: Generic IRQ Controller

ده subsystem جوه subsystem. بدل ما كل driver يكتب IRQ handler من الأول، الـ regmap بيوفر generic IRQ controller بيشتغل على chips اللي interrupt model بتاعها بيتبع pattern معين:

- **status register**: بيقرأ منه عشان يعرف IRQ مين
- **mask register**: بيكتب فيه عشان يعمل enable/disable
- **ack register**: بيكتب فيه عشان يكلير الـ interrupt

```
Physical IRQ (parent)
       │
       ▼
regmap_irq_handler()
       │ reads status_base registers via regmap
       │ calls handle_irq for each set bit
       ▼
Virtual IRQ Domain
  ├── virq 0 → e.g. PMIC_UVLO
  ├── virq 1 → e.g. PMIC_OCP
  └── virq 2 → e.g. PMIC_TEMP_WARN
       │
       ▼
Consumer drivers (gpio, power, charger, ...) claim these virqs
```

الـ regmap_irq_chip بيتكامل مع:
- **IRQ subsystem** (الـ `irq_domain`): ده subsystem بيعمل mapping بين hardware IRQ numbers وlinux virtual IRQ numbers
- **الـ regmap نفسه**: كل read/write على registers الـ interrupt بيتم عبر الـ regmap (فبيستفيد من الـ cache والـ bus abstraction)

---

### إيه اللي الـ regmap بيمتلكه وإيه اللي بيفوّضه

| المسؤولية | الـ regmap Core | الـ Driver | الـ regmap_bus |
|---|---|---|---|
| Register address → bus transaction | لا | لا | **نعم** |
| Endian conversion | **نعم** | لا | لا |
| Cache management | **نعم** | لا (بيضبطه بس) | لا |
| Locking (mutex/spinlock) | **نعم** | override فقط | لا |
| Bit masking/shifting في الـ fields | **نعم** | لا | لا |
| تحديد أي regs volatile | لا | **نعم** (callback) | لا |
| تحديد default values | لا | **نعم** (reg_defaults) | لا |
| الـ physical protocol (start/stop bits, CS, إلخ) | لا | لا | **نعم** |
| IRQ handler registration | **نعم** (regmap_irq) | config فقط | لا |
| debugfs exposure | **نعم** (تلقائي) | لا | لا |

---

### تدفق الـ `regmap_write` من أول لآخر

```
Driver calls: regmap_write(map, 0x42, 0x07)
                │
                ▼
         [regmap core]
         1. تحقق: reg <= max_register?
         2. تحقق: writeable_reg(dev, 0x42) == true?
         3. lock(map)  ← mutex أو spinlock حسب fast_io
                │
                ▼
         [cache layer]
         4. هل 0x42 volatile؟ → لا → اكتب في cache
                │
                ▼
         [format layer]
         5. حوّل reg وval لـ byte array حسب reg_bits, val_bits, endianness
            + ضرب read/write_flag_mask
                │
                ▼
         [bus layer: regmap_bus.reg_write]
         6. استدعي regmap-i2c أو regmap-spi أو regmap-mmio
                │
                ▼
         [physical bus]
         7. i2c_transfer() / spi_sync() / writel()
                │
                ▼
         unlock(map)
         return 0 or error
```

---

### ملاحظة على الـ `devm_` Variants

كل `regmap_init_*()` ليها نسخة `devm_regmap_init_*()`. الـ **devm** (device-managed) بتعني إن الـ regmap بيتحرر تلقائياً لما الـ device بتتunbind أو بتفشل في الـ probe. ده بيتكامل مع الـ **devres** subsystem — وهو mechanism بيربط resources بـ lifetime الـ device.

الـ driver الحديث دايماً يستخدم `devm_` — أبسط وأأمن من ناحية memory leaks.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Enums والـ Config Options — Cheatsheet

#### `enum regcache_type` — نوع الـ cache المستخدم

| القيمة | المعنى |
|---|---|
| `REGCACHE_NONE` | لا يوجد cache — كل قراءة/كتابة تذهب للـ hardware مباشرةً |
| `REGCACHE_RBTREE` | cache بهيكل red-black tree، مناسب للأنظمة محدودة الموارد مع sync متكرر |
| `REGCACHE_FLAT` | مصفوفة flat مع zero-init، للتوافق مع كود قديم فقط |
| `REGCACHE_FLAT_S` | flat sparse — لا يُهيّئ بـ zero، أفضل لـ drivers جديدة مع holes |
| `REGCACHE_MAPLE` | maple tree، الافتراضي الموصى به — يدعم allocations وقت التشغيل |

#### `enum regmap_endian` — الـ endianness لتنسيق العناوين والقيم

| القيمة | المعنى |
|---|---|
| `REGMAP_ENDIAN_DEFAULT` | يرث من الـ `regmap_bus` |
| `REGMAP_ENDIAN_BIG` | big-endian صريح |
| `REGMAP_ENDIAN_LITTLE` | little-endian صريح |
| `REGMAP_ENDIAN_NATIVE` | يستخدم endianness المعالج نفسه |

#### أهم الـ `#define` macros — Cheatsheet

| الـ Macro | الغرض |
|---|---|
| `REGMAP_UPSHIFT(s)` | يحوّل positive stride لـ upshift سالب `(-s)` |
| `REGMAP_DOWNSHIFT(s)` | downshift موجب `(s)` |
| `REGMAP_MDIO_C45_DEVAD_SHIFT/MASK` | encoding لـ IEEE 802.3ae clause 45 address (bits 20:16 = device, bits 15:0 = reg) |
| `regmap_reg_range(low, high)` | ينشئ `struct regmap_range` inline |
| `REG_SEQ(_reg, _def, _us)` | ينشئ `struct reg_sequence` مع delay |
| `REG_SEQ0(_reg, _def)` | نفسه بدون delay |
| `REG_FIELD(_reg, _lsb, _msb)` | ينشئ `struct reg_field` |
| `REG_FIELD_ID(...)` | `reg_field` مع port id size/offset |
| `REGMAP_IRQ_REG(_irq, _off, _mask)` | initializer لعنصر في مصفوفة `regmap_irq` |
| `REGMAP_IRQ_REG_LINE(_id, _reg_bits)` | auto-computes mask وregister_offset من IRQ id |
| `REGMAP_IRQ_MAIN_REG_OFFSET(arr)` | ينشئ `regmap_irq_sub_irq_map` من مصفوفة |
| `regmap_init_i2c/spi/mmio/...` | wrapper macros تضيف lockdep key تلقائيًا |
| `devm_regmap_init_*` | managed variants — الـ regmap يُحرر تلقائيًا مع الـ device |

#### flags بيتية في `regmap_irq_chip`

| الـ Flag | المعنى |
|---|---|
| `init_ack_masked` | ack كل الـ masked interrupts مرة واحدة أثناء init |
| `use_ack` | استخدم `ack_base` حتى لو قيمته صفر |
| `ack_invert` | الـ ack يعمل بـ clear (صفر = ack) بدل set |
| `clear_ack` | استخدم 1 ثم 0 أو العكس لمسح الـ interrupt |
| `status_invert` | bit صفر = interrupt نشط (مقلوب) |
| `status_is_level` | register القيمة الفعلية للسيگنال — XOR مع القيمة السابقة للحصول على events |
| `wake_invert` | صفر = wake enabled |
| `type_in_mask` | استخدم mask registers لتحديد نوع الـ IRQ |
| `clear_on_unmask` | اقرأ status قبل unmask لمسح أي bits متراكمة |
| `runtime_pm` | احجز runtime PM lock عند الوصول للـ hardware |
| `no_status` | لا يوجد status register — كل interrupts تُعتبر نشطة دومًا |
| `mask_unmask_non_inverted` | يتحكم في منطق الـ mask/unmask عند وجود كليهما معًا |

---

### 1. الـ Structs الأساسية — شرح تفصيلي

---

#### `struct reg_default`
**الغرض:** يخزّن القيمة الافتراضية (power-on reset) لـ register واحد.

```c
struct reg_default {
    unsigned int reg;  /* عنوان الـ register */
    unsigned int def;  /* قيمته الافتراضية */
};
```

- يُستخدم في مصفوفة `regmap_config.reg_defaults[]`.
- المصفوفة sparse — لا تحتاج إدراج كل register، فقط اللي عنده قيمة غير صفر أو مهمة.
- الـ `regcache` يرجع لها لإعادة تهيئة cache بعد `regcache_mark_dirty()`.

**يرتبط بـ:** `regmap_config` (يُشار إليها من `reg_defaults`)

---

#### `struct reg_sequence`
**الغرض:** وصف write واحد ضمن سلسلة writes، مع delay اختياري بعده.

```c
struct reg_sequence {
    unsigned int reg;       /* العنوان */
    unsigned int def;       /* القيمة */
    unsigned int delay_us;  /* تأخير بالميكروثانية بعد الكتابة */
};
```

- يُستخدم مع `regmap_multi_reg_write()` و`regmap_register_patch()`.
- مثال حقيقي: تسلسل init لـ codec صوتي يحتاج delay بين ضبط PLL وتشغيل DAC.

**يرتبط بـ:** APIs مباشرة، لا يوجد pointer من struct آخر.

---

#### `struct regmap_range`
**الغرض:** يصف نطاق عناوين registers (من `range_min` لـ`range_max`).

```c
struct regmap_range {
    unsigned int range_min;
    unsigned int range_max;
};
```

- اللبنة الأساسية لـ`regmap_access_table`.

**يرتبط بـ:** `regmap_access_table` (يحتوي مصفوفة منها).

---

#### `struct regmap_access_table`
**الغرض:** جدول يحدد النطاقات المسموح بها أو الممنوعة لعملية معينة (read/write/volatile/precious).

```c
struct regmap_access_table {
    const struct regmap_range *yes_ranges;  /* نطاقات مسموحة */
    unsigned int n_yes_ranges;
    const struct regmap_range *no_ranges;   /* نطاقات ممنوعة — تُفحص أولاً */
    unsigned int n_no_ranges;
};
```

- قاعدة الفحص: `no_ranges` أولاً — إذا وُجد الـ register فيها → return false. ثم `yes_ranges` — إذا وُجد → return true.
- يُستبدل بـ callbacks صريحة مثل `volatile_reg()` إذا أردت منطقًا أعقد.

**يرتبط بـ:** `regmap_config` (ستة pointers لها: `wr_table`, `rd_table`, `volatile_table`, `precious_table`, `wr_noinc_table`, `rd_noinc_table`).

---

#### `struct regmap_config` ← **المحور الرئيسي**
**الغرض:** كل الإعدادات التي يحددها الـ driver لإنشاء الـ `regmap`. يُملأ compile-time كـ static وlا يتغير بعد init.

| الحقل | النوع | الغرض |
|---|---|---|
| `name` | `const char *` | اسم اختياري — مفيد إذا كان للجهاز أكثر من regmap |
| `reg_bits` | `int` | عدد bits عنوان الـ register — **إلزامي** |
| `val_bits` | `int` | عدد bits قيمة الـ register — **إلزامي** |
| `reg_stride` | `int` | خطوة العناوين الصحيحة (0 = افتراض 1) |
| `reg_shift` | `int` | shift يُطبّق على العنوان قبل أي عملية |
| `reg_base` | `unsigned int` | يُضاف لكل عنوان قبل الإرسال |
| `pad_bits` | `int` | bits الحشو بين العنوان والقيمة في البروتوكول |
| `writeable_reg` | callback | هل يمكن الكتابة لهذا الـ register؟ |
| `readable_reg` | callback | هل يمكن القراءة منه؟ |
| `volatile_reg` | callback | هل يتغير بدون تدخل SW؟ (لا يُخزَّن في cache) |
| `precious_reg` | callback | لا يُقرأ إلا من الـ driver (مثل clear-on-read status) |
| `writeable_noinc_reg` | callback | يقبل كتابة متكررة بدون increment للعنوان (FIFO) |
| `readable_noinc_reg` | callback | نفسه للقراءة |
| `reg_read` | callback | استبدال كامل لعملية القراءة للأجهزة الغريبة |
| `reg_write` | callback | استبدال كامل لعملية الكتابة |
| `reg_update_bits` | callback | استبدال RMW الافتراضي (إذا كان الـ hardware يدعم atomic bit-set) |
| `read` / `write` | callbacks | للعمليات الـ bulk |
| `max_raw_read/write` | `size_t` | حد حجم نقل بيانات خام واحد |
| `fast_io` | `bool` | استخدم spinlock بدل mutex (للـ MMIO السريع) |
| `disable_locking` | `bool` | لا تستخدم أي locking (الـ driver يتحمل المسؤولية) |
| `lock` / `unlock` | typedefs | lock مخصص يستبدل الافتراضي |
| `lock_arg` | `void *` | argument يُمرر للـ lock/unlock callbacks |
| `max_register` | `unsigned int` | أعلى عنوان صحيح |
| `wr_table` / `rd_table` / ... | `regmap_access_table *` | جداول النطاقات بديلاً عن الـ callbacks |
| `reg_defaults` | `reg_default *` | مصفوفة القيم الافتراضية |
| `cache_type` | `enum regcache_type` | نوع الـ cache |
| `reg_defaults_raw` | `const void *` | قيم افتراضية خام للـ flat cache |
| `read_flag_mask` | `unsigned long` | bits تُضبط في byte أعلى الـ register عند القراءة |
| `write_flag_mask` | `unsigned long` | مثله للكتابة (شائع في SPI لتمييز read/write) |
| `use_hwlock` | `bool` | استخدم hardware spinlock |
| `hwlock_id` | `unsigned int` | رقم الـ hardware lock |
| `reg_format_endian` | `enum` | endianness تنسيق العنوان |
| `val_format_endian` | `enum` | endianness تنسيق القيمة |
| `ranges` | `regmap_range_cfg *` | إعدادات الـ virtual address ranges |

**يرتبط بـ:** `regmap_range_cfg` (عبر `ranges`)، `regmap_access_table` (ستة حقول)، `reg_default` (عبر `reg_defaults`).

---

#### `struct regmap_range_cfg`
**الغرض:** يدعم الـ registers التي تُعنوَن بشكل غير مباشر (paged/banked) — مثل chips بـ 8-bit address space لكن لديها آلاف الـ registers تُختار عبر page selector.

```c
struct regmap_range_cfg {
    const char *name;
    /* النطاق الافتراضي (virtual) */
    unsigned int range_min;
    unsigned int range_max;
    /* الـ register الذي يختار الصفحة */
    unsigned int selector_reg;
    unsigned int selector_mask;
    int selector_shift;
    /* نافذة البيانات الفعلية بعد اختيار الصفحة */
    unsigned int window_start;
    unsigned int window_len;
};
```

- **مثال:** codec به 512 register لكن address bus فيه 8 bits فقط — pages من 0 لـ 1 كل منها 256 register.
- الوصول لـ `range_min + offset` يترجم لـ: اكتب page رقم في `selector_reg`، ثم اقرأ/اكتب في `window_start + (offset % window_len)`.

**يرتبط بـ:** `regmap_config.ranges[]` (يُشار إليه من هناك).

---

#### `struct regmap_sdw_mbq_cfg`
**الغرض:** إعدادات إضافية لـ SoundWire Multi-Byte Quantity registers — registers ذات أحجام متغيرة تدعم deferred transactions.

| الحقل | الغرض |
|---|---|
| `mbq_size` | callback يُعيد الحجم الفعلي للـ register (بالبايت) |
| `deferrable` | callback — هل يمكن تأجيل transaction هذا؟ (SDCA فقط) |
| `timeout_us` | timeout بالميكرو ثانية لانتظار الـ deferred transaction |
| `retry_us` | فترة الانتظار بين محاولات الـ retry |

---

#### `struct regmap_bus`
**الغرض:** يصف العمليات الـ hardware التي يوفرها البروتوكول (I2C/SPI/MMIO/...). الـ bus driver يملأه، والـ regmap core يستدعي ops منه.

| الحقل | النوع | الغرض |
|---|---|---|
| `fast_io` | `bool` | spinlock بدل mutex |
| `free_on_exit` | `bool` | حرّر الـ struct نفسه عند `regmap_exit()` |
| `write` | `regmap_hw_write` | bulk write (reg+val معًا في buffer واحد) |
| `gather_write` | `regmap_hw_gather_write` | bulk write مع reg وval في buffers منفصلة |
| `async_write` | `regmap_hw_async_write` | async write (لا ينتظر اكتمالها) |
| `reg_write` | `regmap_hw_reg_write` | كتابة register واحد |
| `reg_noinc_write` | `regmap_hw_reg_noinc_write` | كتابة متعددة لنفس العنوان (FIFO) |
| `reg_update_bits` | `regmap_hw_reg_update_bits` | atomic RMW على الـ hardware |
| `read` | `regmap_hw_read` | bulk read |
| `reg_read` | `regmap_hw_reg_read` | قراءة register واحد |
| `reg_noinc_read` | `regmap_hw_reg_noinc_read` | قراءة متعددة من نفس العنوان |
| `free_context` | `regmap_hw_free_context` | تحرير الـ bus_context |
| `async_alloc` | `regmap_hw_async_alloc` | ينشئ `regmap_async` struct |
| `read_flag_mask` | `u8` | default read flag mask (يمكن override من config) |
| `reg_format_endian_default` | `enum` | endianness افتراضي للعناوين |
| `val_format_endian_default` | `enum` | endianness افتراضي للقيم |
| `max_raw_read/write` | `size_t` | حدود حجم النقل على مستوى الـ bus |

**يرتبط بـ:** `regmap` (يُشار إليه من داخله — opaque للـ API العامة)، `regmap_async` (ينشئها عبر `async_alloc`).

---

#### `struct reg_field`
**الغرض:** يصف حقلًا بيتيًا داخل register — مثل bits 3:1 من register 0x10.

```c
struct reg_field {
    unsigned int reg;        /* عنوان الـ register */
    unsigned int lsb;        /* أدنى bit */
    unsigned int msb;        /* أعلى bit */
    unsigned int id_size;    /* عدد الـ ports (للأجهزة متعددة القنوات) */
    unsigned int id_offset;  /* offset بين كل port والتالي */
};
```

- الـ `id_size` و`id_offset` مفيدان للـ hardware الذي يكرر نفس الهيكل لعدة قنوات — مثل 4 channels كل channel له register منفصل.
- يستخدمه `regmap_field_alloc()` لإنشاء `struct regmap_field` (opaque).

**يرتبط بـ:** `regmap_field` (يُشار إليه عند الإنشاء).

---

#### `struct regmap_irq_type`
**الغرض:** يصف كيف يُبرمج الـ hardware لنوع معين من الـ IRQ (rising/falling/level).

| الحقل | الغرض |
|---|---|
| `type_reg_offset` | offset الـ register الذي يحدد النوع |
| `type_reg_mask` | mask الـ bits المعنية |
| `type_rising_val` | القيمة لتفعيل RISING edge |
| `type_falling_val` | القيمة لتفعيل FALLING edge |
| `type_level_low_val` | القيمة لـ LEVEL_LOW |
| `type_level_high_val` | القيمة لـ LEVEL_HIGH |
| `types_supported` | OR من `IRQ_TYPE_*` flags تصف ما يدعمه الـ hardware |

---

#### `struct regmap_irq`
**الغرض:** وصف IRQ فردي ضمن الـ chip — أين هو في الـ status/mask registers وكيف يُبرمج نوعه.

```c
struct regmap_irq {
    unsigned int reg_offset;      /* offset من base في bank IRQ */
    unsigned int mask;            /* الـ bit(s) الخاص بهذا الـ IRQ */
    struct regmap_irq_type type;  /* إعدادات نوع الـ trigger */
};
```

**يرتبط بـ:** `regmap_irq_chip.irqs[]` (مصفوفة منها).

---

#### `struct regmap_irq_sub_irq_map`
**الغرض:** يربط bit واحد في الـ main status register بعدة sub-IRQ registers — للـ chips ذات التسلسل الهرمي للـ interrupts.

```c
struct regmap_irq_sub_irq_map {
    unsigned int num_regs;   /* عدد الـ sub-registers */
    unsigned int *offset;    /* مصفوفة offsets من status_base */
};
```

---

#### `struct regmap_irq_chip`
**الغرض:** الوصف الكامل لـ IRQ controller مبني على regmap — يُستخدم مع `regmap_add_irq_chip()` لإنشاء virtual IRQ domain.

| مجموعة الحقول | الغرض |
|---|---|
| `name` / `domain_suffix` | تعريف وتمييز IRQ domain |
| `main_status` / `num_main_status_bits` | للـ chips ذات الـ hierarchical IRQs |
| `sub_reg_offsets[]` | خريطة ربط main bits بـ sub-IRQ registers |
| `status_base` | عنوان أول register حالة الـ IRQ |
| `mask_base` / `unmask_base` | عناوين الـ mask (1=masked) والـ unmask |
| `ack_base` | عنوان تأكيد الاستقبال |
| `wake_base` | عنوان wake enable |
| `config_base[]` | مصفوفة عناوين الـ type config registers |
| `irq_reg_stride` | stride للـ chips ذات registers غير متجاورة |
| `num_regs` | عدد registers في كل bank |
| `irqs[]` / `num_irqs` | وصف كل IRQ على حدة |
| `handle_pre_irq` / `handle_post_irq` | callbacks لمعالجة خاصة قبل/بعد الـ IRQ handling |
| `handle_mask_sync` | callback لـ mask sync مخصص |
| `set_type_config` | callback لبرمجة نوع الـ IRQ |
| `get_irq_reg` | callback لتحويل (base, index) لعنوان register |
| `irq_drv_data` | بيانات خاصة بالـ driver تُمرر للـ callbacks |

**يرتبط بـ:** `regmap_irq` (مصفوفة `irqs[]`)، `regmap_irq_chip_data` (opaque — ينشؤها `regmap_add_irq_chip()`)، `irq_domain`.

---

### 2. مخطط علاقات الـ Structs (ASCII)

```
+------------------+
| regmap_config    |
|  reg_bits        |---+
|  val_bits        |   |
|  cache_type -----+-->| enum regcache_type
|  *wr_table ------+-->+---------------------------+
|  *rd_table ------+   | regmap_access_table       |
|  *volatile_table-+   |  *yes_ranges --> regmap_range[]
|  *precious_table |   |  *no_ranges  --> regmap_range[]
|  *reg_defaults --+-->| reg_default[]             |
|  *ranges --------+-->| regmap_range_cfg[]        |
|  lock/unlock     |   +---------------------------+
+------------------+
         |
         | يُمرر لـ
         v
+------------------+       +------------------+
|   regmap_init()  |<------| regmap_bus       |
|  (macro wrapper) |       |  .write          |
+------------------+       |  .read           |
         |                 |  .reg_write      |
         v                 |  .async_alloc -->| regmap_async (opaque)
+------------------+       +------------------+
|   struct regmap  | <--- opaque للـ drivers
|   (opaque core)  |
+------------------+
    |         |
    |         +-----------> regmap_field (عبر regmap_field_alloc)
    |                           |
    |                           v
    |                       reg_field (description)
    |
    +-----------> regmap_irq_chip_data (عبر regmap_add_irq_chip)
                     |
                     +----> irq_domain (kernel IRQ subsystem)
                     +----> regmap_irq_chip (static description)
                                |
                                +----> regmap_irq[] (per-IRQ description)
                                           |
                                           +----> regmap_irq_type
```

---

### 3. دورة حياة الـ Regmap — Lifecycle Diagram

```
DRIVER PROBE
     |
     v
[1] ملء regmap_config (static const في الـ driver)
     |
     v
[2] اختيار bus init macro:
     regmap_init_i2c(client, &config)
     regmap_init_spi(dev, &config)
     regmap_init_mmio(dev, regs, &config)
     regmap_init_sdw(dev, &config)
     ...
     |
     | يستدعي __regmap_lockdep_wrapper()
     v
[3] __regmap_init_XXX() تخصص struct regmap (opaque)
     |
     +-- تختار locking: mutex (slow IO) أو spinlock (fast_io) أو custom
     +-- تُهيّئ الـ regcache بنوعه المطلوب
     +-- تُحمّل reg_defaults في الـ cache
     +-- تُسجّل في device (regmap_attach_dev)
     |
     v
[4] struct regmap* جاهزة للاستخدام
     |
     v
[5] RUNTIME OPS:
     +--regmap_write() -------> cache update + bus write
     +--regmap_read() -------> cache hit → return / cache miss → bus read
     +--regmap_update_bits() -> RMW (read+mask+write atomic under lock)
     +--regmap_bulk_read/write() -> متعدد
     +--regmap_field_write() --> mask+shift تلقائي ثم regmap_write
     |
     v
[6] CACHE OPS (اختيارية):
     regcache_cache_only(map, true) → يكتب للـ cache فقط (offline mode)
     regcache_mark_dirty(map)       → يعلّم كل الـ cache كـ dirty
     regcache_sync(map)             → يكتب كل dirty entries للـ hardware
     regcache_cache_bypass(map)     → يتجاهل الـ cache مؤقتًا
     |
     v
[7] DRIVER REMOVE / devm cleanup:
     regmap_exit(map)  ← أو تلقائيًا مع devm_regmap_init_*
     |
     +-- تحرير الـ cache
     +-- تحرير أي resources
     +-- إلغاء التسجيل من الـ device
```

---

### 4. IRQ Lifecycle

```
DRIVER PROBE
     |
     v
[1] ملء static const regmap_irq_chip مع irqs[]
     |
     v
[2] regmap_add_irq_chip(map, irq, flags, base, chip, &chip_data)
     |
     +-- ينشئ regmap_irq_chip_data (opaque)
     +-- يُخصص mask/status buffers
     +-- يسجّل irq_domain مع kernel
     +-- يُركّب interrupt handler على parent IRQ
     +-- يستدعي init_ack_masked إذا مُفعّل
     |
     v
[3] RUNTIME:
     Parent IRQ fires
          |
          v
     regmap_irq_thread() (threaded handler)
          |
          +-- يقرأ status registers (عبر regmap_read)
          +-- لـ hierarchical: يقرأ main_status أولاً
          +-- يحدد كل IRQ نشط
          +-- handle_pre_irq() إذا وُجد
          +-- يستدعي generic_handle_irq() لكل virtual IRQ
          +-- يكتب ACK registers
          +-- handle_post_irq() إذا وُجد
          |
          v
     Child IRQ handlers يُنفَّذون

[4] TEARDOWN:
     regmap_del_irq_chip(irq, data)  ← أو devm تلقائيًا
          |
          +-- يحرر irq_domain
          +-- يحرر buffers
          +-- يفك تسجيل handler
```

---

### 5. Call Flow — عمليات القراءة والكتابة

#### `regmap_write(map, reg, val)`
```
regmap_write(map, reg, val)
  └─> regmap_lock(map)                    ← mutex أو spinlock حسب fast_io
      └─> _regmap_write(map, reg, val)
          ├─> regmap_writeable(map, reg)   ← تحقق من wr_table أو writeable_reg()
          ├─> map->format.format_write()   ← تنسيق العنوان+القيمة حسب reg/val bits
          ├─> [إذا كان paged]: regmap_select_page()
          │       └─> regmap_write(selector_reg, page_num)  ← nested write للصفحة
          ├─> map->bus->reg_write() أو map->bus->write()    ← HW access
          └─> regcache_write(map, reg, val)                  ← تحديث الـ cache
      └─> regmap_unlock(map)
```

#### `regmap_read(map, reg, val)`
```
regmap_read(map, reg, &val)
  └─> regmap_lock(map)
      └─> _regmap_read(map, reg, &val)
          ├─> regmap_readable(map, reg)     ← تحقق من rd_table أو readable_reg()
          ├─> regcache_read(map, reg, &val) ← هل موجود في الـ cache؟
          │   ├─> HIT: return val مباشرةً (لو volatile_reg = false)
          │   └─> MISS: تابع للـ hardware
          ├─> [إذا volatile أو cache miss]:
          │   ├─> map->bus->reg_read() أو map->bus->read()  ← HW access
          │   └─> regcache_write(map, reg, val)             ← تخزين في cache
          └─> return val
      └─> regmap_unlock(map)
```

#### `regmap_update_bits_base(map, reg, mask, val, ...)`
```
regmap_update_bits_base(map, reg, mask, val, change, async, force)
  └─> regmap_lock(map)
      ├─> [إذا كان bus->reg_update_bits موجود]: استدعاء مباشر للـ HW atomic op
      └─> [وإلا RMW]:
          ├─> _regmap_read(map, reg, &tmp)
          ├─> new_val = (tmp & ~mask) | (val & mask)
          ├─> [إذا force أو new_val != tmp]:
          │   └─> _regmap_write(map, reg, new_val)
          └─> *change = (new_val != tmp)
      └─> regmap_unlock(map)
```

#### `regmap_field_write(field, val)`
```
regmap_field_write(field, val)
  └─> regmap_field_update_bits_base(field, ~0, val, NULL, false, false)
      └─> mask  = GENMASK(field->msb, field->lsb)
          shifted_val = (val << field->lsb) & mask
          └─> regmap_update_bits_base(field->regmap, field->reg, mask, shifted_val)
```

---

### 6. استراتيجية الـ Locking

#### الخيارات الثلاثة

| الوضع | الـ Lock المستخدم | متى |
|---|---|---|
| `fast_io = false` (افتراضي) | `mutex` | الـ bus يستغرق وقتًا (I2C/SPI) ويمكن sleep |
| `fast_io = true` | `spinlock` | MMIO سريع، لا يمكن sleep |
| `disable_locking = true` | لا شيء | الـ driver يضمن thread safety بنفسه |
| `lock/unlock` مخصصان | أي آلية | عند الحاجة لمنطق لocking معقد |
| `use_hwlock = true` | hardware spinlock | للـ shared registers بين cores مختلفة (مثل multi-core SoCs) |
| `use_raw_spinlock = true` | `raw_spinlock` | للـ real-time kernels |

#### ما الذي يحميه الـ Lock؟

```
regmap_lock()
  │
  ├─ يحمي: struct regmap الداخلية (format buffers, cache state)
  ├─ يحمي: تسلسل read-modify-write (atomic من منظور الـ software)
  ├─ يحمي: paging state (selector_reg → window access يجب أن يكونا atomic)
  └─ يحمي: async operation tracking
```

#### ترتيب الـ Locking (Lock Ordering)

```
[parent lock مثل device_lock]
        ↓
[regmap->lock  (mutex أو spinlock)]
        ↓
[bus lock الداخلي مثل i2c_adapter lock — يُدار من bus layer]
        ↓
[hardware register access]
```

> **تحذير:** لا تستدعي `regmap_*` APIs وأنت ممسك بـ `regmap->lock` من خارج الـ core — ستحدث deadlock. الـ nested writes مثل page selection تستدعي `_regmap_write()` (بدون lock) لا `regmap_write()`.

#### الـ Async و Locking

- `regmap_write_async()` تُدخل الـ write في قائمة انتظار لكن تحمي الـ queue بالـ lock نفسه.
- `regmap_async_complete(map)` تنتظر اكتمال كل الـ async ops — يجب استدعاؤها قبل أي عملية sync تعتمد على نتائجها.
- `regmap_async` struct يُنشأ بالـ `bus->async_alloc()` وهو opaque تمامًا خارج الـ core.
## Phase 4: شرح الـ Functions

---

## ملخص سريع — Cheatsheet لكل الـ APIs المهمة

### Registration & Initialization

| الـ Macro/Function | الـ Bus | نوعه |
|---|---|---|
| `regmap_init(dev, bus, ctx, cfg)` | Generic | Manual |
| `regmap_init_i2c(i2c, cfg)` | I2C | Manual |
| `regmap_init_spi(dev, cfg)` | SPI | Manual |
| `regmap_init_mmio(dev, regs, cfg)` | MMIO | Manual |
| `regmap_init_mmio_clk(dev, clk, regs, cfg)` | MMIO+CLK | Manual |
| `regmap_init_sdw(sdw, cfg)` | SoundWire | Manual |
| `regmap_init_sdw_mbq(sdw, cfg)` | SoundWire MBQ | Manual |
| `regmap_init_spmi_base(dev, cfg)` | SPMI Base | Manual |
| `regmap_init_spmi_ext(dev, cfg)` | SPMI Ext | Manual |
| `regmap_init_ac97(ac97, cfg)` | AC97 | Manual |
| `regmap_init_w1(w1_dev, cfg)` | 1-Wire | Manual |
| `regmap_init_fsi(fsi_dev, cfg)` | FSI | Manual |
| `devm_regmap_init_*(...)` | كل الـ buses | Managed (devres) |
| `regmap_exit(map)` | — | Cleanup |

### Read / Write

| الـ Function | الاستخدام |
|---|---|
| `regmap_read(map, reg, &val)` | قراءة register واحد |
| `regmap_write(map, reg, val)` | كتابة register واحد |
| `regmap_raw_read(map, reg, buf, len)` | قراءة raw bytes |
| `regmap_raw_write(map, reg, buf, len)` | كتابة raw bytes |
| `regmap_bulk_read(map, reg, buf, count)` | قراءة bulk (مع val formatting) |
| `regmap_bulk_write(map, reg, buf, count)` | كتابة bulk |
| `regmap_noinc_read(map, reg, buf, len)` | قراءة من نفس الـ reg |
| `regmap_noinc_write(map, reg, buf, len)` | كتابة لنفس الـ reg |
| `regmap_write_async(map, reg, val)` | كتابة async |
| `regmap_raw_write_async(map, reg, buf, len)` | كتابة raw async |
| `regmap_read_bypassed(map, reg, &val)` | قراءة bypass cache |
| `regmap_multi_reg_write(map, regs, num)` | كتابة sequence |
| `regmap_multi_reg_read(map, regs, buf, count)` | قراءة من عناوين متفرقة |

### Update Bits

| الـ Function | الـ Behavior |
|---|---|
| `regmap_update_bits(map, reg, mask, val)` | RMW عادي |
| `regmap_update_bits_async(...)` | RMW async |
| `regmap_update_bits_check(...)` | RMW + يرجع هل تغير |
| `regmap_update_bits_check_async(...)` | RMW async + change check |
| `regmap_write_bits(map, reg, mask, val)` | force write (no change check) |
| `regmap_set_bits(map, reg, bits)` | set bits |
| `regmap_clear_bits(map, reg, bits)` | clear bits |
| `regmap_assign_bits(map, reg, bits, value)` | set أو clear حسب bool |
| `regmap_test_bits(map, reg, bits)` | اختبار bits (1 = set) |

### Cache

| الـ Function | الاستخدام |
|---|---|
| `regcache_sync(map)` | flush cache للـ hardware |
| `regcache_sync_region(map, min, max)` | sync region محددة |
| `regcache_drop_region(map, min, max)` | حذف cache region |
| `regcache_cache_only(map, bool)` | تفعيل/تعطيل cache-only mode |
| `regcache_cache_bypass(map, bool)` | تجاوز الـ cache |
| `regcache_mark_dirty(map)` | علّم الـ cache كـ dirty |
| `regcache_reg_cached(map, reg)` | هل الـ reg موجود في الـ cache؟ |
| `regcache_sort_defaults(defaults, n)` | ترتيب الـ defaults |

### regmap_field

| الـ Function | الاستخدام |
|---|---|
| `regmap_field_alloc(map, field)` | تخصيص field |
| `regmap_field_free(field)` | تحرير field |
| `devm_regmap_field_alloc(dev, map, field)` | managed field |
| `regmap_field_read(field, &val)` | قراءة field |
| `regmap_field_write(field, val)` | كتابة field |
| `regmap_field_update_bits(field, mask, val)` | update bits في field |
| `regmap_field_set_bits(field, bits)` | set bits في field |
| `regmap_field_clear_bits(field, bits)` | clear bits في field |
| `regmap_fields_read(field, id, &val)` | قراءة field في port معين |
| `regmap_fields_write(field, id, val)` | كتابة field في port معين |

### IRQ

| الـ Function | الاستخدام |
|---|---|
| `regmap_add_irq_chip(...)` | تسجيل irq_chip |
| `regmap_add_irq_chip_fwnode(...)` | تسجيل irq_chip مع fwnode |
| `regmap_del_irq_chip(irq, data)` | حذف irq_chip |
| `devm_regmap_add_irq_chip(...)` | managed irq_chip |
| `regmap_irq_get_virq(data, irq)` | تحويل HW irq → virq |
| `regmap_irq_get_domain(data)` | الحصول على الـ irq_domain |
| `regmap_irq_chip_get_base(data)` | الحصول على irq_base |

### Helpers & Misc

| الـ Function | الاستخدام |
|---|---|
| `dev_get_regmap(dev, name)` | الحصول على regmap من device |
| `regmap_get_device(map)` | الحصول على الـ device |
| `regmap_get_val_bytes(map)` | حجم الـ value بالبايت |
| `regmap_get_max_register(map)` | أعلى عنوان |
| `regmap_get_reg_stride(map)` | الـ stride |
| `regmap_might_sleep(map)` | هل regmap ops ممكن تنام؟ |
| `regmap_async_complete(map)` | انتظار انتهاء الـ async ops |
| `regmap_can_raw_write(map)` | هل raw write مدعوم؟ |
| `regmap_register_patch(map, regs, n)` | تطبيق patch sequence |
| `regmap_parse_val(map, buf, &val)` | parse raw buffer |
| `regmap_attach_dev(dev, map, cfg)` | ربط map بـ device |
| `regmap_reinit_cache(map, cfg)` | إعادة تهيئة الـ cache |
| `regmap_mmio_attach_clk(map, clk)` | ربط clock بـ MMIO map |
| `regmap_mmio_detach_clk(map)` | فصل الـ clock |
| `regmap_check_range_table(map, reg, tbl)` | فحص range table |
| `regmap_reg_in_range(reg, range)` | هل الـ reg في range؟ |
| `regmap_reg_in_ranges(reg, ranges, n)` | فحص array من ranges |

### Poll Macros

| الـ Macro | الاستخدام |
|---|---|
| `regmap_read_poll_timeout(map, addr, val, cond, sleep_us, timeout_us)` | polling قابل للنوم |
| `regmap_read_poll_timeout_atomic(...)` | polling atomic (MMIO/flat cache فقط) |
| `regmap_field_read_poll_timeout(...)` | polling على field |

---

## المجموعة الأولى: Initialization & Registration

الـ initialization في regmap يعتمد على طبقتين — **macros عامة** تولّد lockdep key تلقائياً عبر `__regmap_lockdep_wrapper`، وتحتها **`__regmap_init*` functions** الفعلية التي تستقبل الـ key. المبرمج دائماً يستخدم الـ macros مباشرة.

كل init function موجود في نسختين:
- **Manual**: الـ driver مسؤول عن استدعاء `regmap_exit()` يدوياً
- **Managed (devm_)**: الـ kernel يحررها تلقائياً عند `device_release`

---

### `regmap_init()` / `__regmap_init()`

```c
#define regmap_init(dev, bus, bus_context, config) \
    __regmap_lockdep_wrapper(__regmap_init, #config, \
                             dev, bus, bus_context, config)

struct regmap *__regmap_init(struct device *dev,
                             const struct regmap_bus *bus,
                             void *bus_context,
                             const struct regmap_config *config,
                             struct lock_class_key *lock_key,
                             const char *lock_name);
```

**الـ Generic init** — الأساس اللي كل الـ bus-specific inits تبني عليه. بتبني الـ `struct regmap` من الصفر: تخصّص الذاكرة، تهيّئ الـ locking (mutex أو spinlock حسب `fast_io`)، تسجّل الـ cache، وتسجّل الـ map في `dev->devres` (في حالة devm). معظم الـ bus-specific drivers لا تستدعيها مباشرة — يستخدمونها فقط لو عندهم bus خاصة بيها.

**Parameters:**
- `dev` — الـ device المرتبط بالـ map، يُستخدم للـ logging والـ devres
- `bus` — مؤشر لـ `struct regmap_bus` يحتوي الـ callbacks الخاصة بالـ bus
- `bus_context` — context pointer يُمرّر لـ bus callbacks (e.g., الـ `i2c_client`)
- `config` — إعدادات الـ map: عدد bits، endianness، cache type، إلخ

**Return:** مؤشر صحيح أو `ERR_PTR()` في حالة الفشل

**Key details:**
- الـ `lock_key` و`lock_name` يستخدمهم lockdep لتتبع lock ordering
- لو `config->disable_locking == true` — لا يُنشئ أي lock
- لو `config->fast_io == true` — يستخدم spinlock بدل mutex

---

### `regmap_init_i2c()` / `devm_regmap_init_i2c()`

```c
#define regmap_init_i2c(i2c, config) \
    __regmap_lockdep_wrapper(__regmap_init_i2c, #config, i2c, config)
```

أشهر init في الـ kernel — يُستخدم مع أجهزة I2C كـ codecs الصوت وسنسورات الطاقة. الـ `i2c_client` يُمرَّر كـ `bus_context`، والـ bus callbacks الداخلية تستدعي `i2c_transfer()` تلقائياً.

**Parameters:**
- `i2c` — مؤشر لـ `struct i2c_client`
- `config` — إعدادات الـ map

---

### `regmap_init_spi()` / `devm_regmap_init_spi()`

```c
#define regmap_init_spi(dev, config) \
    __regmap_lockdep_wrapper(__regmap_init_spi, #config, dev, config)
```

يستخدم الـ `spi_device` كـ context. الـ SPI regmap يدعم الـ `read_flag_mask` و`write_flag_mask` لتمييز القراءة والكتابة في الـ byte الأول من الـ frame — شائع جداً في sensors وADCs.

---

### `regmap_init_mmio()` / `regmap_init_mmio_clk()`

```c
#define regmap_init_mmio(dev, regs, config) \
    regmap_init_mmio_clk(dev, NULL, regs, config)

#define regmap_init_mmio_clk(dev, clk_id, regs, config) \
    __regmap_lockdep_wrapper(__regmap_init_mmio_clk, #config, \
                             dev, clk_id, regs, config)
```

خاص بالـ Memory-Mapped I/O. يستخدم spinlock تلقائياً (`fast_io` مفعّل ضمنياً). الـ `clk_id` اختياري — لو حُدّد، الـ MMIO map يطلب الـ clock قبل كل عملية ويحرره بعدها عبر `regmap_mmio_attach_clk()`.

**Parameters:**
- `regs` — `void __iomem *` مؤشر لمنطقة الـ MMIO المُربوطة مسبقاً
- `clk_id` — اسم الـ clock consumer أو NULL

---

### `regmap_attach_dev()`

```c
int regmap_attach_dev(struct device *dev, struct regmap *map,
                      const struct regmap_config *config);
```

يربط `struct regmap` بـ `struct device` بدون إنشاء map جديد. مفيد لـ drivers تُنشئ الـ map في مكان ثم تحتاج ربطها بـ device آخر. يجعل `dev_get_regmap()` تعمل على هذا الـ device.

**Return:** 0 نجاح، سالب خطأ.

---

### `regmap_reinit_cache()`

```c
int regmap_reinit_cache(struct regmap *map,
                        const struct regmap_config *config);
```

يدمر الـ cache الحالية ويعيد بناءها بناءً على `config` جديد. يُستخدم عند تغيير وضع الـ cache أثناء الـ runtime (نادر). يأخذ الـ map lock داخلياً.

---

### `regmap_exit()`

```c
void regmap_exit(struct regmap *map);
```

يحرر كل الموارد المرتبطة بالـ map: الـ cache، الـ debugfs entries، الـ async ops المعلقة، وأخيراً الـ map نفسه. يجب الانتظار على الـ async ops قبل الاستدعاء (يفعل ذلك داخلياً).

**من يستدعيها:** Driver في `remove()` — فقط للـ manual (non-devm) maps.

---

## المجموعة الثانية: Read Operations

الـ read في regmap تمر بمسار: check writeable → check cache → bus read → update cache.

---

### `regmap_read()`

```c
int regmap_read(struct regmap *map, unsigned int reg, unsigned int *val);
```

القراءة الأساسية لـ register واحد. تأخذ الـ map lock، تحوّل الـ reg حسب `reg_shift` و`reg_base`، تتحقق من `readable_reg`، تبحث في الـ cache أولاً، ولو cache miss تذهب للـ hardware وتحدّث الـ cache.

**Parameters:**
- `map` — مؤشر الـ regmap
- `reg` — عنوان الـ register
- `val` — مؤشر لمكان تخزين القيمة

**Return:** 0 نجاح، `-EINVAL` لو الـ reg غير قابل للقراءة، أو خطأ الـ bus.

**Locking:** يأخذ `map->lock` داخلياً — آمن للاستدعاء من process context.

---

### `regmap_read_bypassed()`

```c
int regmap_read_bypassed(struct regmap *map, unsigned int reg,
                         unsigned int *val);
```

مثل `regmap_read()` لكن يتجاوز الـ cache ويقرأ من الـ hardware مباشرة، دون تحديث الـ cache. مفيد لـ debug أو للـ precious registers التي لا يجب قراءتها إلا حال الضرورة.

---

### `regmap_raw_read()`

```c
int regmap_raw_read(struct regmap *map, unsigned int reg,
                    void *val, size_t val_len);
```

قراءة bytes خام بدون أي format conversion لـ val. البيانات ترجع بـ endianness الـ hardware. يُستخدم لقراءة multi-byte registers أو لنقل البيانات كما هي.

**Key detail:** لا يستخدم الـ cache — دائماً يقرأ من الـ hardware.

---

### `regmap_noinc_read()`

```c
int regmap_noinc_read(struct regmap *map, unsigned int reg,
                      void *val, size_t val_len);
```

يقرأ `val_len` bytes من نفس عنوان الـ register دون increment. شائع مع FIFOs وأجهزة تدعم burst read من نفس العنوان (مثل بعض SPI peripherals).

---

### `regmap_bulk_read()`

```c
int regmap_bulk_read(struct regmap *map, unsigned int reg,
                     void *val, size_t val_count);
```

يقرأ `val_count` registers متتالية بدءاً من `reg`. يطبّق الـ val formatting على كل قيمة. يستخدم الـ cache لو متاح. الـ `val` buffer يجب أن يكون بحجم `val_count * regmap_get_val_bytes(map)`.

---

### `regmap_multi_reg_read()`

```c
int regmap_multi_reg_read(struct regmap *map, const unsigned int *reg,
                          void *val, size_t val_count);
```

يقرأ من قائمة عناوين غير متتالية. مفيد لقراءة registers متفرقة في عملية واحدة. `val` هو array من `unsigned int`.

---

### `regmap_read_poll_timeout` (macro)

```c
#define regmap_read_poll_timeout(map, addr, val, cond, sleep_us, timeout_us)
```

ينتظر حتى تتحقق `cond` أو ينتهي الـ timeout. يستخدم `usleep_range()` داخلياً — **لا يصح في atomic context** (إلا لو `sleep_us == 0`). يبني على `read_poll_timeout()` من `linux/iopoll.h`.

```
loop:
    regmap_read(map, addr, &val)
    if error → return error
    if cond  → return 0
    if timeout → return -ETIMEDOUT
    sleep(sleep_us)
```

---

### `regmap_read_poll_timeout_atomic` (macro)

```c
#define regmap_read_poll_timeout_atomic(map, addr, val, cond, delay_us, timeout_us)
```

نسخة atomic من الـ poll — تستخدم `udelay()` بدل `usleep_range()`. **تعمل فقط مع MMIO regmap وflat/no cache**. لا يجوز استخدامها مع I2C/SPI لأن bus transfers تحتاج sleep.

---

## المجموعة الثالثة: Write Operations

---

### `regmap_write()`

```c
int regmap_write(struct regmap *map, unsigned int reg, unsigned int val);
```

الكتابة الأساسية. تأخذ الـ lock، تتحقق من `writeable_reg`، تُحدّث الـ cache (لو الـ reg ليس volatile)، ثم ترسل للـ hardware. لو `cache_only` مفعّل — تكتب للـ cache فقط.

**Return:** 0 نجاح، `-EINVAL` لو الـ reg غير قابل للكتابة.

---

### `regmap_write_async()`

```c
int regmap_write_async(struct regmap *map, unsigned int reg, unsigned int val);
```

نفس `regmap_write()` لكن يقدر يُكمل قبل ما تنتهي عملية الـ bus الفعلية. يُستخدم مع buses تدعم DMA أو async transactions. لازم تستدعي `regmap_async_complete()` قبل ما تعمل أي عملية synchronous تالية.

---

### `regmap_raw_write()`

```c
int regmap_raw_write(struct regmap *map, unsigned int reg,
                     const void *val, size_t val_len);
```

كتابة bytes خام بدون format conversion. **لا يُحدّث الـ cache**. مفيد لنقل binary blobs أو firmware patches.

---

### `regmap_noinc_write()`

```c
int regmap_noinc_write(struct regmap *map, unsigned int reg,
                       const void *val, size_t val_len);
```

يكتب `val_len` bytes لنفس الـ register address بدون increment — لـ FIFOs وstreaming writes.

---

### `regmap_bulk_write()`

```c
int regmap_bulk_write(struct regmap *map, unsigned int reg,
                      const void *val, size_t val_count);
```

يكتب `val_count` قيم متتالية بدءاً من `reg`. يطبّق الـ formatting ويُحدّث الـ cache. لو `can_multi_write == false` — يُقسّمها لـ individual writes تلقائياً.

---

### `regmap_multi_reg_write()`

```c
int regmap_multi_reg_write(struct regmap *map,
                           const struct reg_sequence *regs,
                           int num_regs);
```

يطبّق sequence من write operations مع optional delays بين كل كتابة. الـ `struct reg_sequence` يحتوي `{reg, def, delay_us}`. يُستخدم لـ initialization sequences.

**Pseudocode:**
```
for each (reg, val, delay_us) in regs:
    regmap_write(map, reg, val)
    if delay_us:
        udelay(delay_us)
```

---

### `regmap_multi_reg_write_bypassed()`

```c
int regmap_multi_reg_write_bypassed(struct regmap *map,
                                    const struct reg_sequence *regs,
                                    int num_regs);
```

نفس `regmap_multi_reg_write()` لكن يتجاوز الـ cache — يكتب للـ hardware مباشرة. مفيد لـ reset sequences.

---

### `regmap_register_patch()`

```c
int regmap_register_patch(struct regmap *map,
                          const struct reg_sequence *regs,
                          int num_regs);
```

يُطبّق patch ويحفظه في الـ map ليُعاد تطبيقه تلقائياً بعد كل `regcache_sync()`. شائع للـ silicon errata workarounds.

---

## المجموعة الرابعة: Update Bits (RMW Operations)

كل دوال الـ update bits تنتهي باستدعاء `regmap_update_bits_base()` — هي الـ real implementation.

---

### `regmap_update_bits_base()`

```c
int regmap_update_bits_base(struct regmap *map, unsigned int reg,
                            unsigned int mask, unsigned int val,
                            bool *change, bool async, bool force);
```

الـ core RMW function. تقرأ الـ register، تطبّق الـ mask، تكتب القيمة الجديدة. تتحقق مما إذا تغيّرت البيانات فعلاً لو `force == false` — لو القيمة نفسها لا تكتب.

**Parameters:**
- `mask` — الـ bits المراد تعديلها
- `val` — القيم الجديدة داخل الـ mask (bits خارج mask = 0)
- `change` — يُعبّأ بـ true لو تغيّرت البيانات (يمكن NULL)
- `async` — true لو الكتابة async
- `force` — true لإجبار الكتابة حتى لو القيمة لم تتغير

**Key detail:** يأخذ الـ lock مرة واحدة طول عملية read-modify-write — atomic من ناحية الـ regmap lock، لكن ليس atomic من ناحية الـ hardware.

**Pseudocode:**
```
lock(map)
read current value → tmp
new_val = (tmp & ~mask) | (val & mask)
if new_val == tmp && !force:
    *change = false
    unlock; return 0
write new_val
*change = true
unlock
```

---

### `regmap_update_bits()`

```c
static inline int regmap_update_bits(struct regmap *map, unsigned int reg,
                                     unsigned int mask, unsigned int val)
```

الـ wrapper الأكثر استخداماً — `regmap_update_bits_base(map, reg, mask, val, NULL, false, false)`.

---

### `regmap_set_bits()` / `regmap_clear_bits()` / `regmap_assign_bits()`

```c
static inline int regmap_set_bits(struct regmap *map,
                                  unsigned int reg, unsigned int bits);

static inline int regmap_clear_bits(struct regmap *map,
                                    unsigned int reg, unsigned int bits);

static inline int regmap_assign_bits(struct regmap *map, unsigned int reg,
                                     unsigned int bits, bool value);
```

**الـ set_bits:** يعمل `mask = val = bits` — يضع الـ bits المحددة.
**الـ clear_bits:** يعمل `mask = bits, val = 0` — يمسح الـ bits.
**الـ assign_bits:** يختار بينهم حسب `value`.

---

### `regmap_test_bits()`

```c
int regmap_test_bits(struct regmap *map, unsigned int reg, unsigned int bits);
```

يقرأ الـ register ويتحقق إذا كانت كل الـ `bits` set. يُرجع 1 لو كلها set، 0 لو لا، أو سالب في حالة خطأ.

---

### `regmap_write_bits()`

```c
static inline int regmap_write_bits(struct regmap *map, unsigned int reg,
                                    unsigned int mask, unsigned int val)
```

مثل `regmap_update_bits()` لكن `force = true` — يكتب دائماً حتى لو القيمة لم تتغير. مفيد لـ write-only registers أو registers تحتاج trigger بالكتابة.

---

## المجموعة الخامسة: Cache Management

الـ regmap cache يعمل كـ write-back cache — الكتابة تذهب للـ cache والـ hardware معاً. الـ sync يُستخدم بعد power-up أو resume.

---

### `regcache_sync()`

```c
int regcache_sync(struct regmap *map);
```

يُزامن محتوى الـ cache مع الـ hardware — يكتب كل الـ dirty entries للـ hardware. يُستخدم في `resume()` بعد `regcache_mark_dirty()`. يُطبّق أيضاً أي `register_patch` محفوظ.

**من يستدعيها:** Driver في `system_sleep_pm_ops.resume` أو `runtime_resume`.

---

### `regcache_sync_region()`

```c
int regcache_sync_region(struct regmap *map, unsigned int min,
                         unsigned int max);
```

يُزامن نطاق محدد من الـ registers فقط. أسرع من sync كامل لو الـ hardware يدعم partial restore.

---

### `regcache_drop_region()`

```c
int regcache_drop_region(struct regmap *map, unsigned int min,
                         unsigned int max);
```

يحذف الـ cache entries للنطاق المحدد — القراءة التالية ستذهب للـ hardware. مفيد لـ volatile regions أو بعد hardware reset.

---

### `regcache_cache_only()`

```c
void regcache_cache_only(struct regmap *map, bool enable);
```

لو `enable = true` — كل العمليات تذهب للـ cache فقط (الـ hardware لا يُلمس). يُستخدم أثناء الـ suspend لحماية الـ hardware من وصول غير مرغوب. لو `enable = false` — يُعيد التشغيل الطبيعي.

---

### `regcache_cache_bypass()`

```c
void regcache_cache_bypass(struct regmap *map, bool enable);
```

لو `enable = true` — كل العمليات تذهب للـ hardware مباشرة وتتجاوز الـ cache تماماً. مفيد لـ debug أو لـ init sequences التي تحتاج القراءة من الـ hardware.

---

### `regcache_mark_dirty()`

```c
void regcache_mark_dirty(struct regmap *map);
```

يُعلّم الـ cache بأكملها كـ dirty — يُستخدم في الـ suspend قبل power-off، حتى يُعيد `regcache_sync()` كتابة كل القيم للـ hardware عند الـ resume.

**Pattern شائع في الـ suspend/resume:**
```c
// suspend:
regcache_cache_only(map, true);
regcache_mark_dirty(map);

// resume:
regcache_cache_only(map, false);
regcache_sync(map);
```

---

### `regcache_sort_defaults()`

```c
void regcache_sort_defaults(struct reg_default *defaults,
                            unsigned int ndefaults);
```

يُرتّب مصفوفة `reg_default` بترتيب تصاعدي حسب العنوان — مطلوب للـ flat cache التي تبحث بـ binary search. يُستدعى مرة واحدة في الـ init.

---

## المجموعة السادسة: regmap_field

الـ `regmap_field` يُجرّد الوصول لـ bit-field داخل register — بدلاً من إدارة الـ masks يدوياً في كل مكان.

---

### `regmap_field_alloc()` / `devm_regmap_field_alloc()`

```c
struct regmap_field *regmap_field_alloc(struct regmap *regmap,
                                        struct reg_field reg_field);

struct regmap_field *devm_regmap_field_alloc(struct device *dev,
                                              struct regmap *regmap,
                                              struct reg_field reg_field);
```

يُنشئ `struct regmap_field` بناءً على `struct reg_field` الذي يحدد `{reg, lsb, msb}`. الـ field object يُخزّن الـ mask ومقدار الـ shift داخلياً.

**مثال:**
```c
/* تعريف field للـ bits [5:3] في register 0x10 */
static const struct reg_field my_field = REG_FIELD(0x10, 3, 5);

/* في probe(): */
priv->field = devm_regmap_field_alloc(dev, map, my_field);
```

---

### `regmap_field_bulk_alloc()` / `devm_regmap_field_bulk_alloc()`

```c
int regmap_field_bulk_alloc(struct regmap *regmap,
                             struct regmap_field **rm_field,
                             const struct reg_field *reg_field,
                             int num_fields);
```

يُخصص مصفوفة من الـ fields دفعة واحدة. أكفأ من استدعاء `regmap_field_alloc()` لكل field على حدة.

---

### `regmap_field_read()`

```c
int regmap_field_read(struct regmap_field *field, unsigned int *val);
```

يقرأ الـ register ويُزيح ويُطبّق الـ mask تلقائياً. القيمة الراجعة هي قيمة الـ field فقط (بدون الـ bits الخارجية). داخلياً يستخدم `regmap_read()`.

**مثال:** لو field هو bits [5:3] وقيمة الـ register هي `0b00101000` — القيمة الراجعة ستكون `0b101 = 5`.

---

### `regmap_field_write()`

```c
static inline int regmap_field_write(struct regmap_field *field,
                                     unsigned int val)
```

يُحدّث الـ bits المحددة بالـ field فقط — باقي الـ register يبقى كما هو. داخلياً يستدعي `regmap_field_update_bits_base(field, ~0, val, ...)`.

---

### `regmap_field_update_bits_base()`

```c
int regmap_field_update_bits_base(struct regmap_field *field,
                                  unsigned int mask, unsigned int val,
                                  bool *change, bool async, bool force);
```

الـ core function للـ field updates. مثل `regmap_update_bits_base()` لكن يأخذ بعين الاعتبار الـ shift والـ mask الخاصة بالـ field. الـ `mask` هنا relative للـ field وليس للـ register.

---

### `regmap_fields_read()` / `regmap_fields_write()`

```c
int regmap_fields_read(struct regmap_field *field, unsigned int id,
                       unsigned int *val);

static inline int regmap_fields_write(struct regmap_field *field,
                                      unsigned int id, unsigned int val)
```

للـ fields ذات `id_size` — أي fields موجودة في عدة registers متتالية (ports). الـ `id` يحدد رقم الـ port. الـ register الفعلي = `field->reg + id * field->id_offset`.

**مثال:** device يحتوي 4 channels، كل channel له نفس الـ bit field لكن في register مختلف:
```c
static const struct reg_field gain_field = REG_FIELD_ID(0x20, 4, 7, 4, 0x10);
/* الـ reg لـ channel 0: 0x20, channel 1: 0x30, channel 2: 0x40... */
regmap_fields_write(priv->gain, 2, 0xF); /* يكتب channel 2 */
```

---

## المجموعة السابعة: IRQ Management

الـ regmap IRQ subsystem يُنشئ `irq_domain` يدير interrupt controller مبني على registers.

---

### `regmap_add_irq_chip()`

```c
int regmap_add_irq_chip(struct regmap *map, int irq, int irq_flags,
                        int irq_base, const struct regmap_irq_chip *chip,
                        struct regmap_irq_chip_data **data);
```

يُسجّل **`regmap_irq_chip`** كـ interrupt controller. يُنشئ `irq_domain`، يُسجّل `irq_handler` على الـ parent IRQ، ويُهيّئ الـ mask/ack registers.

**Parameters:**
- `irq` — رقم الـ parent IRQ (من الـ SoC interrupt controller)
- `irq_flags` — flags للـ parent IRQ (IRQF_SHARED, IRQF_ONESHOT, إلخ)
- `irq_base` — أول رقم virtual IRQ مطلوب (0 = dynamic allocation)
- `chip` — وصف الـ interrupt controller (registers, masks, acks)
- `data` — يُعبّأ بـ opaque handle للـ chip data

**Return:** 0 نجاح، سالب خطأ.

**الـ handler الداخلي يفعل:**
```
read status registers
for each set bit:
    call handle_nested_irq(virq)
ack/clear interrupts
```

---

### `regmap_add_irq_chip_fwnode()`

```c
int regmap_add_irq_chip_fwnode(struct fwnode_handle *fwnode,
                               struct regmap *map, int irq,
                               int irq_flags, int irq_base,
                               const struct regmap_irq_chip *chip,
                               struct regmap_irq_chip_data **data);
```

نفس `regmap_add_irq_chip()` لكن يربط الـ `irq_domain` بـ `fwnode` — ضروري للـ DT/ACPI firmware حتى يتمكن `irq_of_parse_and_map()` من إيجاد الـ domain.

---

### `regmap_del_irq_chip()`

```c
void regmap_del_irq_chip(int irq, struct regmap_irq_chip_data *data);
```

يحذف الـ IRQ chip — يُلغي تسجيل الـ IRQ handler، يُدمر الـ `irq_domain`، ويُحرر كل الموارد. يُستدعى في `remove()` للـ manual registrations.

---

### `regmap_irq_get_virq()`

```c
int regmap_irq_get_virq(struct regmap_irq_chip_data *data, int irq);
```

يُحوّل رقم الـ hardware IRQ (index في الـ `chip->irqs` array) إلى virtual IRQ number. يُستخدم من driver ليمرّر الـ virq لـ `request_irq()` أو client devices.

---

### `regmap_irq_get_domain()`

```c
struct irq_domain *regmap_irq_get_domain(struct regmap_irq_chip_data *data);
```

يُرجع الـ `irq_domain` للـ chip. مفيد لـ drivers تحتاج استخدام `irq_create_mapping()` مباشرة.

---

### `regmap_irq_get_irq_reg_linear()`

```c
unsigned int regmap_irq_get_irq_reg_linear(struct regmap_irq_chip_data *data,
                                            unsigned int base, int index);
```

الـ default implementation للـ `get_irq_reg` callback — تحسب عنوان الـ register كـ `base + index * irq_reg_stride`. الـ drivers تُعيّن `chip->get_irq_reg` لتحسين هذا السلوك لو الـ registers ليست linear.

---

### `regmap_irq_set_type_config_simple()`

```c
int regmap_irq_set_type_config_simple(unsigned int **buf, unsigned int type,
                                      const struct regmap_irq *irq_data,
                                      int idx, void *irq_drv_data);
```

الـ default `set_type_config` callback — يُعيّن bits الـ type (RISING/FALLING/LEVEL) بناءً على `regmap_irq_type` struct. الـ drivers تُعيّن `chip->set_type_config` لو احتاجت سلوك مختلف.

---

## المجموعة الثامنة: Helper & Utility Functions

---

### `dev_get_regmap()`

```c
struct regmap *dev_get_regmap(struct device *dev, const char *name);
```

يسترجع الـ `regmap` المرتبط بـ device. الـ `name` اختياري (NULL لو device له map واحد). يُستخدم من drivers ثانوية تحتاج الوصول لـ regmap أنشأه parent driver.

---

### `regmap_get_device()`

```c
struct device *regmap_get_device(struct regmap *map);
```

يُرجع الـ `device` المرتبط بالـ map. مفيد لـ helpers تستقبل فقط `struct regmap *` وتحتاج log أو devres.

---

### `regmap_get_val_bytes()`

```c
int regmap_get_val_bytes(struct regmap *map);
```

يُرجع حجم الـ register value بالبايت (e.g., 2 لـ 16-bit registers). يُستخدم لحساب حجم الـ buffer في bulk operations.

---

### `regmap_get_max_register()`

```c
int regmap_get_max_register(struct regmap *map);
```

يُرجع `config->max_register`. مفيد لـ validation في debugfs أو ioctl handlers.

---

### `regmap_get_reg_stride()`

```c
int regmap_get_reg_stride(struct regmap *map);
```

يُرجع `config->reg_stride`. الـ valid register addresses هي مضاعفات هذه القيمة.

---

### `regmap_might_sleep()`

```c
bool regmap_might_sleep(struct regmap *map);
```

يُرجع `true` لو الـ regmap يستخدم mutex (وليس spinlock) — أي قد تنام العمليات. مفيد للتحقق قبل استدعاء regmap من atomic context.

---

### `regmap_async_complete()`

```c
int regmap_async_complete(struct regmap *map);
```

ينتظر حتى تنتهي كل الـ async write operations المعلقة. **يجب** استدعاؤها قبل:
- قراءة register كُتب async
- unmap أو power-off الـ device
- استخدام `regmap_exit()`

---

### `regmap_can_raw_write()`

```c
bool regmap_can_raw_write(struct regmap *map);
```

يتحقق إذا كان الـ bus يدعم raw write (الـ bus لديه `write` callback). بعض الـ no-bus maps لا تدعمها.

---

### `regmap_parse_val()`

```c
int regmap_parse_val(struct regmap *map, const void *buf, unsigned int *val);
```

يُحوّل raw buffer (بـ endianness الـ hardware) إلى `unsigned int` محلي. مفيد لمعالجة البيانات الراجعة من `regmap_raw_read()`.

---

### `regmap_check_range_table()`

```c
bool regmap_check_range_table(struct regmap *map, unsigned int reg,
                               const struct regmap_access_table *table);
```

يتحقق من وجود الـ register في `regmap_access_table`. يبحث في `no_ranges` أولاً (يُرجع false لو موجود)، ثم `yes_ranges` (يُرجع true لو موجود).

---

### `regmap_reg_in_range()` / `regmap_reg_in_ranges()`

```c
static inline bool regmap_reg_in_range(unsigned int reg,
                                       const struct regmap_range *range);

bool regmap_reg_in_ranges(unsigned int reg,
                          const struct regmap_range *ranges,
                          unsigned int nranges);
```

helpers بسيطة لفحص نطاقات العناوين. الأولى تفحص range واحد، الثانية تفحص مصفوفة. تُستخدم في `writeable_reg` / `readable_reg` callbacks.

---

### `regmap_mmio_attach_clk()` / `regmap_mmio_detach_clk()`

```c
int regmap_mmio_attach_clk(struct regmap *map, struct clk *clk);
void regmap_mmio_detach_clk(struct regmap *map);
```

تُربط/تفصل clock بـ MMIO regmap. بعد الـ attach، كل عملية read/write ستُفعّل الـ clock أولاً وتوقفه بعد الانتهاء. مفيد لـ peripheral blocks تحتاج clock خاصة لوصول registers.

---

### `regmap_ac97_default_volatile()`

```c
bool regmap_ac97_default_volatile(struct device *dev, unsigned int reg);
```

الـ default `volatile_reg` callback لـ AC97 devices — يُعيّد قائمة الـ AC97 registers المعروفة كـ volatile (status registers, shadow registers). يُستخدم في `config->volatile_reg`.

---

## مخطط المسار الداخلي — Write Path

```
regmap_write(map, reg, val)
        │
        ▼
   lock(map->lock)
        │
        ▼
 writeable_reg(dev, reg)?  ──NO──► -EINVAL
        │YES
        ▼
  apply reg_shift, reg_base
        │
        ▼
  cache_only mode? ──YES──► update cache only → unlock → return
        │NO
        ▼
  cache_bypass? ──NO──► update cache
        │
        ▼
  map->reg_write() OR
  map->bus->write()
        │
        ▼
   unlock(map->lock)
        │
        ▼
       return 0 / error
```

## مخطط المسار الداخلي — Read Path

```
regmap_read(map, reg, &val)
        │
        ▼
   lock(map->lock)
        │
        ▼
 readable_reg(dev, reg)?  ──NO──► -EINVAL
        │YES
        ▼
  cache_bypass? ──NO──► lookup cache
                              │HIT
                              ▼
                         *val = cached_val → unlock → return 0
                              │MISS
                              ▼
  map->reg_read() OR
  map->bus->read()
        │
        ▼
  volatile_reg? ──NO──► update cache with new val
        │
        ▼
   *val = read_val
        │
        ▼
   unlock(map->lock)
        │
        ▼
       return 0 / error
```
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. مدخلات الـ debugfs

الـ regmap بيعمل entries تلقائياً تحت `/sys/kernel/debug/regmap/` لكل device عنده regmap مسجّل.

```
/sys/kernel/debug/regmap/
└── <bus>-<device>/          ← مثال: i2c-0-001a أو spi0.0
    ├── name                 ← اسم الـ regmap (من regmap_config.name)
    ├── registers            ← dump كامل للـ registers القابلة للقراءة
    ├── access               ← جدول readable/writeable/volatile/precious لكل register
    ├── cache_bypass         ← قراءة/كتابة لتفعيل cache bypass
    ├── cache_only           ← قراءة/كتابة لتفعيل cache-only mode
    └── cache_dirty          ← هل في registers dirty في الـ cache؟
```

```bash
# اقرأ كل الـ registers الموجودة
cat /sys/kernel/debug/regmap/i2c-0-001a/registers

# مثال على output
# 00: 12
# 01: 00
# 04: 7f
# 10: 03

# شوف خصائص كل register
cat /sys/kernel/debug/regmap/i2c-0-001a/access
# reg  read  write  volatile  precious
# 0x00  yes   yes    no        no
# 0x01  yes   no     yes       no

# فعّل cache bypass مؤقتاً (اكتب دايركت على الـ hardware)
echo 1 > /sys/kernel/debug/regmap/i2c-0-001a/cache_bypass
echo 0 > /sys/kernel/debug/regmap/i2c-0-001a/cache_bypass
```

> **ملاحظة مهمة:** الـ registers المُعرَّفة كـ `precious_reg` (clear-on-read) مش بتظهر في dumps الـ debugfs عشان ما تتاتش بغلط.

---

#### 2. مدخلات الـ sysfs

الـ regmap نفسه ما بيكتب في sysfs مباشرةً، بس الـ device اللي بيستخدمه بيبان تحت:

```
/sys/bus/i2c/devices/<addr>/
/sys/bus/spi/devices/spi<x>.<y>/
/sys/bus/platform/devices/<name>/
```

المعلومات المفيدة اللي ممكن تلاقيها:

```bash
# معرفة اسم الـ driver المربوط بالـ device
cat /sys/bus/i2c/devices/0-001a/driver_override
cat /sys/bus/i2c/devices/0-001a/uevent

# تأكيد إن الـ device اتبند بـ driver
ls -la /sys/bus/i2c/devices/0-001a/driver

# بالنسبة لـ IRQ controller مبني على regmap_irq_chip
cat /sys/kernel/irq/<N>/chip_name
cat /sys/kernel/irq/<N>/name
```

---

#### 3. الـ ftrace — tracepoints وأحداث مفيدة

الـ regmap عنده tracepoints مخصوصة في `kernel/trace/events/regmap/`:

```
regmap_reg_read          ← كل قراءة register
regmap_reg_read_cache    ← قراءة من الـ cache (مش من الـ hardware)
regmap_reg_write         ← كل كتابة register
regmap_bulk_read         ← قراءة bulk
regmap_bulk_write        ← كتابة bulk
regmap_async_write_done  ← انتهاء async write
regmap_cache_sync        ← بداية sync الـ cache مع الـ hardware
regmap_cache_bypass      ← تغيير cache_bypass state
regmap_cache_only        ← تغيير cache_only state
regmap_async_complete_start ← بداية انتظار async operations
regmap_async_complete_done  ← نهاية انتظار async operations
```

```bash
# تفعيل كل regmap events
echo 1 > /sys/kernel/debug/tracing/events/regmap/enable

# أو event محدد فقط (مثلاً تتبع الكتابات)
echo 1 > /sys/kernel/debug/tracing/events/regmap/regmap_reg_write/enable

# فلترة على device معين (مثلاً i2c-0-001a)
echo 'name == "i2c-0-001a"' > \
  /sys/kernel/debug/tracing/events/regmap/regmap_reg_write/filter

# ابدأ التتبع
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل الـ driver / trigger الـ interrupt
# ثم اقرأ النتائج
cat /sys/kernel/debug/tracing/trace

# مثال على output
# regmap_reg_write: i2c-0-001a reg=0x04 val=0x7f
# regmap_reg_read:  i2c-0-001a reg=0x01 val=0x00
# regmap_cache_sync: i2c-0-001a

# أوقف التتبع وامسحه
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo > /sys/kernel/debug/tracing/trace
```

---

#### 4. الـ printk والـ dynamic debug

تفعيل الـ debug messages الخاصة بالـ regmap:

```bash
# تفعيل كل debug messages في ملفات الـ regmap
echo 'file drivers/base/regmap/*.c +p' > /sys/kernel/debug/dynamic_debug/control

# أو module محدد (مثلاً driver معين يستخدم regmap)
echo 'module mydriver +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل cache debugging
echo 'file drivers/base/regmap/regcache*.c +pflmt' \
  > /sys/kernel/debug/dynamic_debug/control

# p = printk, f = function name, l = line number, m = module, t = timestamp

# تحقق مما هو مُفعَّل
cat /sys/kernel/debug/dynamic_debug/control | grep regmap

# رفع مستوى الـ printk مؤقتاً (دون reboot)
echo 8 > /proc/sys/kernel/printk
```

في الـ kernel log راقب:
```
regmap: ... cache sync failed
regmap: ... i2c transfer failed
regmap: ... register out of range
```

---

#### 5. Kernel Config Options للـ debugging

| Option | الوظيفة |
|--------|---------|
| `CONFIG_REGMAP` | تفعيل الـ regmap نفسه — لازم يكون enabled |
| `CONFIG_REGMAP_DEBUG` | تفعيل debug logging داخل الـ regmap |
| `CONFIG_REGMAP_DEBUGFS` | تفعيل مجلد debugfs لكل regmap |
| `CONFIG_DEBUG_REGMAP` | (بعض الـ trees) validation إضافية |
| `CONFIG_REGMAP_I2C` | دعم الـ I2C backend |
| `CONFIG_REGMAP_SPI` | دعم الـ SPI backend |
| `CONFIG_REGMAP_MMIO` | دعم الـ MMIO backend |
| `CONFIG_REGMAP_IRQ` | دعم الـ IRQ chip المبني على regmap |
| `CONFIG_LOCKDEP` | تفعيل lockdep للكشف عن مشاكل الـ locking في regmap |
| `CONFIG_PROVE_LOCKING` | تحليل أعمق لمشاكل الـ lock ordering |
| `CONFIG_KASAN` | كشف memory corruption في الـ regmap cache buffers |
| `CONFIG_DYNAMIC_DEBUG` | تفعيل dynamic debug للـ regmap messages |
| `CONFIG_TRACING` | تفعيل ftrace support للـ regmap tracepoints |

```bash
# تحقق سريع من الـ config
zcat /proc/config.gz | grep -E "REGMAP|REGCACHE"
```

---

#### 6. أدوات خاصة بالـ regmap

الـ regmap ما عنده devlink مباشر، بس الأدوات دي مفيدة:

```bash
# i2ctools — قراءة/كتابة مباشرة على نفس الـ bus دون الـ driver
i2cdetect -y 0
i2cdump -y 0 0x1a         # dump كل registers
i2cget -y 0 0x1a 0x04     # قراءة register 0x04
i2cset -y 0 0x1a 0x04 0x7f # كتابة register 0x04

# spidev_test — اختبار SPI مباشر
spidev_test -D /dev/spidev0.0 -v -p "\x04\x00"

# devmem2 — للـ MMIO regmap
devmem2 0xfe000000 w      # قراءة 32-bit من عنوان MMIO
devmem2 0xfe000004 w 0x01 # كتابة MMIO مباشرة

# regmap-specific: تفعيل cache bypass وقراءة مباشرة من الـ hardware
echo 1 > /sys/kernel/debug/regmap/<dev>/cache_bypass
cat /sys/kernel/debug/regmap/<dev>/registers
echo 0 > /sys/kernel/debug/regmap/<dev>/cache_bypass
```

---

#### 7. جدول رسائل الأخطاء الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---------------------|--------|------|
| `regmap: can't sync registers` | فشل sync الـ cache مع الـ hardware | افحص اتصال الـ bus (I2C/SPI)، تحقق من `cache_bypass` |
| `regmap: error writing register` | فشل كتابة register عبر الـ bus | تحقق من pull-ups، voltage levels، bus speed |
| `regmap: register out of range` | محاولة وصول لـ register خارج `max_register` | راجع `regmap_config.max_register` |
| `regmap: illegal attempt to write to read only register` | كتابة على register غير قابل للكتابة | راجع `wr_table` أو callback الـ `writeable_reg` |
| `regmap: illegal read from precious register` | قراءة precious register من debugfs | طبيعي — precious registers محمية |
| `regmap API is disabled` | استدعاء regmap API مع `CONFIG_REGMAP=n` | فعّل `CONFIG_REGMAP` في الـ config |
| `regmap: timeout on async write` | انتهت مهلة الـ async write | تحقق من `regmap_async_complete()` وـ bus timeouts |
| `regmap: invalid val_bits` | `val_bits` في `regmap_config` غلط | لازم يكون 8، 16، 32، أو 64 |
| `regmap: volatile range overlaps precious` | تعارض في الـ access tables | راجع `volatile_table` و `precious_table` |
| `i2c i2c-0: ... EREMOTEIO` | الـ device ما رد على I2C | افحص العنوان، الـ power، والـ pull-ups |
| `spi_master spi0: ... -EIO` | فشل SPI transaction | افحص SCLK، MOSI، MISO، CS |

---

#### 8. أماكن استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في regmap_write — للتحقق من قيم غير متوقعة */
int regmap_write(struct regmap *map, unsigned int reg, unsigned int val)
{
    /* تحقق إن الـ map مش NULL قبل أي عملية */
    if (WARN_ON(!map))
        return -EINVAL;

    /* اكتشاف كتابة بعد regmap_exit */
    if (WARN_ON(map->dev == NULL)) {
        dump_stack();
        return -EINVAL;
    }
    /* ... */
}

/* في regcache_sync — لتتبع من يطلب sync */
int regcache_sync(struct regmap *map)
{
    /* مفيد لو sync بيحصل في وقت غريب */
    if (unlikely(map->cache_bypass))
        WARN_ONCE(1, "regcache_sync called with cache_bypass enabled");
    /* ... */
}

/* في الـ driver عند فشل init */
map = devm_regmap_init_i2c(client, &my_regmap_config);
if (IS_ERR(map)) {
    /* dump_stack هنا يساعد لو الـ failure جاي من context غريب */
    dev_err(&client->dev, "regmap init failed: %ld\n", PTR_ERR(map));
    dump_stack();
    return PTR_ERR(map);
}

/* عند update_bits للكشف عن race conditions */
ret = regmap_update_bits(map, REG_CTRL, MASK_ENABLE, MASK_ENABLE);
WARN_ON_ONCE(ret < 0);
```

---

### Hardware Level

#### 1. التحقق من تطابق حالة الـ hardware مع حالة الـ kernel

الـ regmap الـ cache ممكن يكون مختلف عن الـ hardware الفعلي لو:
- حصل reset للـ hardware دون علم الـ driver
- في race condition بين الـ cache والـ hardware

```bash
# 1. شوف ما في الـ kernel cache (من debugfs)
cat /sys/kernel/debug/regmap/<dev>/registers > /tmp/cache_state.txt

# 2. فعّل cache_bypass واقرأ من الـ hardware مباشرة
echo 1 > /sys/kernel/debug/regmap/<dev>/cache_bypass
cat /sys/kernel/debug/regmap/<dev>/registers > /tmp/hw_state.txt
echo 0 > /sys/kernel/debug/regmap/<dev>/cache_bypass

# 3. قارن
diff /tmp/cache_state.txt /tmp/hw_state.txt
# أي فرق = cache/hardware mismatch = مشكلة محتملة

# 4. لو في i2c device يدوياً
i2cdump -y 0 0x1a b > /tmp/i2cdump.txt
diff /tmp/cache_state.txt /tmp/i2cdump.txt
```

---

#### 2. تقنيات الـ Register Dump

**للـ MMIO registers:**

```bash
# devmem2 — أسهل طريقة
apt install devmem2  # أو من المصادر
devmem2 0xfe001000 w     # قراءة word (32-bit)
devmem2 0xfe001000 h     # قراءة halfword (16-bit)
devmem2 0xfe001000 b     # قراءة byte (8-bit)

# /dev/mem مع dd
dd if=/dev/mem bs=4 count=64 skip=$((0xfe001000 / 4)) 2>/dev/null | \
  xxd | head -20

# io command (من package ioport)
io -4 -r 0xfe001000   # قراءة 32-bit register

# مثال script لـ dump نطاق MMIO
BASE=0xfe001000
for i in $(seq 0 4 63); do
    addr=$((BASE + i))
    val=$(devmem2 $addr w 2>/dev/null | grep "Value" | awk '{print $NF}')
    printf "0x%08x: %s\n" $addr "$val"
done
```

**للـ I2C registers:**

```bash
# dump كامل
i2cdump -y 1 0x1a

# قراءة register محدد (مثلاً register 0x04)
i2cget -y 1 0x1a 0x04 b     # byte mode
i2cget -y 1 0x1a 0x04 w     # word mode (16-bit)

# dump مع format محدد
i2cdump -y 1 0x1a b         # byte access
i2cdump -y 1 0x1a w         # word access
i2cdump -y 1 0x1a s         # SMBus block
```

**للـ SPI registers:**

```bash
# spidev_test لإرسال/استقبال raw data
# قراءة register 0x04 (بـ read flag bit 7 set = 0x84)
spidev_test -D /dev/spidev0.0 -p "\x84\x00" -v 2>&1 | grep "RX"

# أو بـ Python
python3 -c "
import spidev
spi = spidev.SpiDev()
spi.open(0, 0)
spi.max_speed_hz = 1000000
# Read register 0x04 (0x84 = 0x04 | 0x80 for read)
resp = spi.xfer2([0x84, 0x00])
print(f'reg 0x04 = 0x{resp[1]:02x}')
spi.close()
"
```

---

#### 3. نصائح Logic Analyzer و Oscilloscope

**الإعداد المثالي:**

```
I2C Probing:
  - Channel 1: SDA (probe عند الـ device، مش عند الـ controller)
  - Channel 2: SCL
  - Trigger: START condition على SDA
  - Decode: I2C protocol decode مدمج في معظم الـ analyzers

SPI Probing:
  - Channel 1: SCLK
  - Channel 2: MOSI
  - Channel 3: MISO
  - Channel 4: CS (chip select)
  - Trigger: CS going low
  - Decode: SPI protocol (Mode 0,1,2,3 حسب device)
```

**نقاط قياس مهمة:**
- قس الـ pull-up resistors على I2C (لازم 4.7kΩ لـ 100kHz، 2.2kΩ لـ 400kHz)
- تحقق من voltage levels: VIH لازم >= 0.7 × VCC، VIL لازم <= 0.3 × VCC
- راقب stretch في SCL (الـ device ممكن يعمل clock stretching)
- قس الـ rise time: طول الكيبل/capacitance بتأثر عليه

**باستخدام sigrok (open source logic analyzer):**

```bash
# تثبيت
apt install sigrok pulseview

# قراءة I2C transaction
sigrok-cli -d fx2lafw -c samplerate=1MHz \
  --channels D0=SCL,D1=SDA \
  -P i2c:scl=SCL:sda=SDA \
  -A i2c=address-read,address-write,data-read,data-write \
  -o capture.sr

# تحليل الـ output
sigrok-cli -i capture.sr -P i2c -A i2c
# مثال output:
# i2c: 'Address write: 1a' 'Data write: 04' 'Data read: 7f'
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في الـ Kernel Log

| المشكلة الـ Hardware | النمط في الـ Kernel Log | التشخيص |
|---------------------|------------------------|---------|
| ضعف pull-up resistors على I2C | `i2c i2c-0: controller timed out` | قس الـ rise time، يجب < 300ns لـ 400kHz |
| power supply unstable | `regmap: error writing register 0x??` متكرر عشوائياً | قس VCC بـ oscilloscope، شوف noise |
| مشكلة clock في SPI | `spi_master spi0: transfer timed out` | تحقق من SCLK continuity، mode مطابق |
| device يعمل reset تلقائي | فجأة كل القراءات ترجع default values | تحقق من RESET pin، power-on-reset circuit |
| عنوان I2C غلط | `i2c i2c-0: ... ENXIO` | افحص A0/A1/A2 pins، i2cdetect |
| voltage mismatch (3.3V vs 5V) | لا response أو data corruption | تحقق من level shifters |
| bus contention (أكتر من master) | `i2c i2c-0: arbitration lost` | تحقق من multi-master setup |
| SPI CS polarity غلط | transaction يبدأ بس ما في response | افحص CS active-high vs active-low |

---

#### 5. تصحيح Device Tree للـ regmap

**التحقق من الـ DT المحمّل:**

```bash
# شوف الـ DT المُطبَّق فعلاً في runtime
ls /sys/firmware/devicetree/base/

# ابحث عن الـ device بالـ compatible string
find /sys/firmware/devicetree/base -name "compatible" -exec \
  grep -l "myvendor,mydevice" {} \;

# اقرأ properties مباشرة
cat /sys/firmware/devicetree/base/soc/i2c@fe010000/mydevice@1a/compatible
hexdump -C /sys/firmware/devicetree/base/soc/i2c@fe010000/mydevice@1a/reg

# استخدام dtc لـ decompile الـ DT المُحمَّل
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | \
  grep -A 20 "mydevice@1a"

# أو أسهل مع fdtdump
fdtdump /sys/firmware/fdt 2>/dev/null | grep -A 20 "mydevice"
```

**أخطاء DT الشائعة مع regmap:**

```dts
/* خطأ شائع 1: reg size غلط لـ MMIO regmap */
mydevice@fe001000 {
    reg = <0xfe001000 0x100>;  /* الحجم لازم يغطي كل الـ registers */
};

/* خطأ شائع 2: نسيان pinctrl / clocks للـ I2C controller */
i2c0: i2c@fe010000 {
    clocks = <&clk CLK_I2C>;   /* لازم موجود */
    pinctrl-0 = <&i2c0_pins>;  /* لازم موجود */
    status = "okay";
};

/* خطأ شائع 3: عنوان الـ device غلط */
mydevice@1a {                  /* 0x1a = 7-bit address */
    reg = <0x1a>;
    compatible = "vendor,mydevice";
};
```

```bash
# تحقق إن الـ driver bind اشتغل
dmesg | grep "mydevice\|regmap" | head -20

# تحقق من الـ clocks والـ regulators اللي الـ device محتاجها
cat /sys/bus/i2c/devices/0-001a/of_node/clocks 2>/dev/null || \
  ls /sys/bus/i2c/devices/0-001a/supplier*/
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للنسخ

**1. تشخيص سريع شامل:**

```bash
#!/bin/bash
DEV="i2c-0-001a"  # غيّر حسب الـ device

echo "=== regmap debugfs entries ==="
ls /sys/kernel/debug/regmap/$DEV/ 2>/dev/null || echo "NOT FOUND"

echo "=== Register dump (from cache) ==="
cat /sys/kernel/debug/regmap/$DEV/registers 2>/dev/null

echo "=== Cache state ==="
echo -n "cache_bypass: "; cat /sys/kernel/debug/regmap/$DEV/cache_bypass 2>/dev/null
echo -n "cache_only: ";   cat /sys/kernel/debug/regmap/$DEV/cache_only 2>/dev/null

echo "=== Recent kernel messages ==="
dmesg | grep -i "regmap\|i2c-0\|001a" | tail -20
```

**2. تتبع كل عمليات الـ regmap لـ device معين:**

```bash
#!/bin/bash
DEV_NAME="i2c-0-001a"
TRACE_DIR=/sys/kernel/debug/tracing

# إعداد الـ filter
echo "name == \"$DEV_NAME\"" > $TRACE_DIR/events/regmap/filter 2>/dev/null || true

# تفعيل كل regmap events
echo 1 > $TRACE_DIR/events/regmap/enable

# صفّر الـ buffer
echo > $TRACE_DIR/trace

# شغّل التتبع
echo 1 > $TRACE_DIR/tracing_on
echo "Tracing started... Press Enter to stop"
read

echo 0 > $TRACE_DIR/tracing_on
cat $TRACE_DIR/trace | grep regmap

# تنظيف
echo 0 > $TRACE_DIR/events/regmap/enable
```

**3. مقارنة cache vs hardware:**

```bash
#!/bin/bash
DEV="i2c-0-001a"
DEBUGFS="/sys/kernel/debug/regmap/$DEV"

echo "--- Cache State ---"
cat $DEBUGFS/registers > /tmp/regmap_cache.txt
cat /tmp/regmap_cache.txt

echo "--- Hardware State (bypass cache) ---"
echo 1 > $DEBUGFS/cache_bypass
cat $DEBUGFS/registers > /tmp/regmap_hw.txt
echo 0 > $DEBUGFS/cache_bypass

echo "--- Diff (cache vs hardware) ---"
diff /tmp/regmap_cache.txt /tmp/regmap_hw.txt
if [ $? -eq 0 ]; then
    echo "MATCH: cache and hardware are in sync"
else
    echo "MISMATCH: cache differs from hardware!"
fi
```

**4. تفعيل dynamic debug للـ regmap كاملاً:**

```bash
# تفعيل كل debug في regmap subsystem
for f in /sys/kernel/debug/dynamic_debug/control; do
    echo 'file drivers/base/regmap/regmap.c +pflmt' > $f
    echo 'file drivers/base/regmap/regcache.c +pflmt' > $f
    echo 'file drivers/base/regmap/regcache-rbtree.c +pflmt' > $f
    echo 'file drivers/base/regmap/regcache-flat.c +pflmt' > $f
    echo 'file drivers/base/regmap/regcache-maple.c +pflmt' > $f
    echo 'file drivers/base/regmap/regmap-irq.c +pflmt' > $f
done

# تابع الـ output
dmesg -w | grep -i regmap
```

**5. اختبار IRQ مبني على regmap:**

```bash
# شوف الـ IRQ domain للـ regmap-irq controller
cat /proc/interrupts | grep -i "regmap\|mydevice"

# شوف الـ irq chip info
for irq in $(cat /proc/interrupts | grep regmap | awk '{print $1}' | tr -d ':'); do
    echo "=== IRQ $irq ==="
    cat /sys/kernel/irq/$irq/chip_name 2>/dev/null
    cat /sys/kernel/irq/$irq/name 2>/dev/null
    cat /sys/kernel/irq/$irq/actions 2>/dev/null
done

# قراءة mask/status registers يدوياً عبر i2c
# (افرض status_base=0x05, mask_base=0x06)
echo "Status register:"
i2cget -y 0 0x1a 0x05 b
echo "Mask register:"
i2cget -y 0 0x1a 0x06 b
```

**6. تشخيص مشكلة regcache:**

```bash
# إجبار sync الـ cache (بعد hardware reset مثلاً)
# من داخل الـ driver (kernel space):
# regcache_mark_dirty(map);   // علّم الـ cache dirty
# regcache_sync(map);          // sync كل الـ cache مع الـ hardware

# من الـ userspace: تفعيل cache_bypass يجبر القراءة من الـ hardware
echo 1 > /sys/kernel/debug/regmap/<dev>/cache_bypass
# الآن كل القراءات من الـ hardware مباشرة
# لما تخلص:
echo 0 > /sys/kernel/debug/regmap/<dev>/cache_bypass

# تحقق من حالة الـ cache
cat /sys/kernel/debug/regmap/<dev>/cache_dirty
# 1 = في registers dirty = الـ cache مختلف عن الـ hardware
# 0 = الـ cache متزامن مع الـ hardware
```

**مثال كامل على تفسير output:**

```
$ cat /sys/kernel/debug/regmap/i2c-0-001a/registers
00: 12   ← register 0x00 = 0x12 (device ID)
01: 00   ← register 0x01 = 0x00 (status = OK)
04: 7f   ← register 0x04 = 0x7f (control = all bits set)
10: 03   ← register 0x10 = 0x03 (config)

# registers مش موجودة = إما not readable أو volatile أو مش في الـ cache
# لو توقعت register 0x02 يظهر بس مش موجود:
#   - ممكن يكون volatile_reg(dev, 0x02) = true
#   - ممكن يكون readable_reg(dev, 0x02) = false
#   - ممكن يكون precious_reg(dev, 0x02) = true

$ cat /sys/kernel/debug/regmap/i2c-0-001a/access
reg  read  write  volatile  precious
0x00  yes   no     no        no      ← read-only register (device ID)
0x01  yes   no     yes       no      ← volatile (status register)
0x02  no    no     no        yes     ← precious (clear-on-read)
0x04  yes   yes    no        no      ← normal R/W register
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — الـ PMIC لا يستجيب بعد الـ suspend

#### العنوان
**الـ regcache يخدع الـ driver بعد الـ system suspend على gateway صناعي**

#### السياق
gateway صناعي بيشتغل على RK3562 ومثبّت عليه PMIC من نوع RK806 متوصل بـ I2C. المنتج بيتحكم في voltage rails لـ CPU وـ DDR. المهندس فعّل الـ `REGCACHE_MAPLE` في الـ `regmap_config` عشان يحسّن الأداء.

#### المشكلة
بعد الـ `echo mem > /sys/power/state` والـ resume، الـ CPU frequency بيتعامل غلط مع الـ voltage، والنظام بيكرش بـ under-voltage. الـ dmesg بيظهر:

```
rk806-regulator: voltage set failed: -EIO
```

#### التحليل
المهندس بص في الـ `regmap_config` ولقى:

```c
static const struct regmap_config rk806_regmap_config = {
    .reg_bits   = 8,
    .val_bits   = 8,
    .cache_type = REGCACHE_MAPLE,   /* cache enabled */
    /* volatile_reg غير محدد! */
};
```

الـ `volatile_reg` callback مش محدود، يعني الـ regmap بيعتبر كل الـ registers قابلة للـ cache. بعد الـ resume، لما الـ driver بيكتب voltage جديد، الـ `regmap_update_bits_base()` بيعمل read من الـ cache بدل الـ hardware. الـ PMIC اتعمله power cycle أثناء الـ suspend، فـ registers رجعت لـ reset values. الـ cache لسه شايل القيم القديمة → الـ write الجديد بيُحسب على أساس قيمة غلط.

التدفق في الـ header:
1. `regmap_update_bits_base()` ← بتعمل read-modify-write
2. الـ read بييجي من الـ cache (maple tree)
3. الـ cache value = القيمة القديمة قبل الـ suspend
4. الـ write بيحط قيمة خاطئة في الـ PMIC

#### الحل

**أولاً:** بعد الـ resume، لازم تعمل `regcache_mark_dirty()` ثم `regcache_sync()`:

```c
/* في الـ resume callback للـ driver */
static int rk806_resume(struct device *dev)
{
    struct rk806_priv *priv = dev_get_drvdata(dev);

    /* mark cache as stale — hardware was reset */
    regcache_mark_dirty(priv->regmap);

    /* re-write all cached values to hardware */
    return regcache_sync(priv->regmap);
}
```

**ثانياً:** أو حدد الـ status registers كـ volatile:

```c
static bool rk806_volatile_reg(struct device *dev, unsigned int reg)
{
    /* status and interrupt registers always read from HW */
    return (reg >= RK806_INT_STS0 && reg <= RK806_INT_STS1);
}

static const struct regmap_config rk806_regmap_config = {
    .reg_bits    = 8,
    .val_bits    = 8,
    .cache_type  = REGCACHE_MAPLE,
    .volatile_reg = rk806_volatile_reg,
};
```

#### الدرس المستفاد
الـ `regcache_mark_dirty()` + `regcache_sync()` هما أول حاجتين تتفكر فيهم في أي مشكلة بعد الـ resume. الـ cache بييجي بـ assumption إن الـ hardware لسه شايل آخر قيمة كتبتها — لو الـ hardware اتعمله reset، هيتعامل مع قيم خاطئة.

---

### السيناريو 2: Android TV Box على Allwinner H616 — الـ SPI codec بيكتب في registers خاطئة

#### العنوان
**الـ `write_flag_mask` مش متحدد صح في driver الـ audio codec على SPI**

#### السياق
TV box بيشتغل على Allwinner H616، فيه audio codec خارجي (AW8898) متوصل بـ SPI. الـ codec datasheet بيقول إن الـ MSB في كل transaction لازم يكون `1` عشان write و`0` عشان read. المهندس استخدم `regmap_init_spi()` بدون ما يحدد الـ flags.

#### المشكلة
الصوت مش شغال خالص. الـ oscilloscope على الـ SPI bus بيظهر إن الـ MSB دايماً `0` حتى في الـ write operations.

#### التحليل
الـ `regmap_config` المكتوب:

```c
static const struct regmap_config aw8898_regmap = {
    .reg_bits  = 8,
    .val_bits  = 8,
    /* لا write_flag_mask ولا read_flag_mask */
};
```

الـ `struct regmap_bus` لـ SPI بيوفر `read_flag_mask` افتراضي هو `0x80` (MSB=1 for read في بعض الـ devices)، لكن الـ AW8898 عنده convention معكوس. الـ `regmap_config.write_flag_mask` و `read_flag_mask` هما اللي بيُـOR-ed في top byte قبل الإرسال.

من الـ header:
```c
unsigned long read_flag_mask;   /* mask for top bytes on read */
unsigned long write_flag_mask;  /* mask for top bytes on write */
bool zero_flag_mask;            /* use masks even if both zero */
```

لو الاتنين `0` والـ `zero_flag_mask` مش set، الـ regmap بيستخدم الـ bus defaults بدلاً من قيم الـ driver. الـ SPI bus default `read_flag_mask = 0x80` بيتطبق على الـ reads، لكن الـ AW8898 بيحتاج `write_flag_mask = 0x00` وـ `read_flag_mask = 0x80` بشكل explicit.

#### الحل

```c
static const struct regmap_config aw8898_regmap = {
    .reg_bits        = 8,
    .val_bits        = 8,
    .write_flag_mask = 0x00,  /* MSB=0 means write for AW8898 */
    .read_flag_mask  = 0x80,  /* MSB=1 means read */
    .zero_flag_mask  = true,  /* enforce even if write_flag_mask is 0 */
};
```

التحقق:

```bash
# راقب الـ SPI traffic بـ logic analyzer أو kernel tracing
echo 1 > /sys/kernel/debug/tracing/events/spi/enable
cat /sys/kernel/debug/tracing/trace
```

#### الدرس المستفاد
الـ `zero_flag_mask` موجود بالظبط لهذي الحالة: لما الـ `write_flag_mask = 0` بس انت عايز الـ regmap يُطبّقها explicitly ويتجاهل الـ bus defaults. بدونها، الـ `0` بيُفسَّر كـ "مش محدد → استخدم الـ bus default".

---

### السيناريو 3: IoT Sensor على STM32MP1 — deadlock في الـ interrupt handler

#### العنوان
**الـ regmap mutex بيسبب deadlock لما driver يقرأ register من داخل الـ ISR**

#### السياق
IoT sensor node بيشتغل على STM32MP1، فيه humidity sensor (SHT31) متوصل بـ I2C مع interrupt line. الـ driver بيحاول يقرأ interrupt status register من داخل الـ `irq_handler` مباشرة باستخدام `regmap_read()`.

#### المشكلة
النظام بيتعلق (hang) عند أول interrupt. الـ lockdep بيطبع:

```
WARNING: possible circular locking dependency detected
sht31_irq_handler -> regmap_lock -> i2c_transfer -> regmap_lock
```

#### التحليل
الـ `regmap_init_i2c()` بينشئ regmap بـ mutex (مش spinlock) لأن `fast_io = false` بالـ default. الـ I2C transfers ممكن تنام (sleep). لما الـ ISR بتشتغل:

```
CPU0 (IRQ context)
  → sht31_irq_handler()
    → regmap_read()           /* يحاول ياخد الـ mutex */
      → mutex_lock()          /* BLOCKS — لأن mutex مش valid في atomic context */
```

من الـ `regmap_config`:
```c
bool fast_io;         /* إذا true → spinlock بدل mutex */
bool disable_locking; /* إذا true → لا locking خالص */
```

الـ IRQ handler هو atomic context، والـ regmap I2C بيستخدم mutex → deadlock.

#### الحل

**الصح:** استخدم `regmap_irq_chip` infrastructure اللي موجودة في الـ header بدل ما تقرأ manually من الـ ISR:

```c
static const struct regmap_irq sht31_irqs[] = {
    REGMAP_IRQ_REG(SHT31_IRQ_DATA_READY, 0, BIT(0)),
    REGMAP_IRQ_REG(SHT31_IRQ_ERROR,      0, BIT(1)),
};

static const struct regmap_irq_chip sht31_irq_chip = {
    .name       = "sht31",
    .status_base = SHT31_STATUS_REG,
    .mask_base   = SHT31_MASK_REG,
    .ack_base    = SHT31_CLEAR_REG,
    .num_regs   = 1,
    .irqs       = sht31_irqs,
    .num_irqs   = ARRAY_SIZE(sht31_irqs),
};

/* في probe */
ret = devm_regmap_add_irq_chip(dev, priv->regmap,
                                client->irq,
                                IRQF_ONESHOT,
                                0,
                                &sht31_irq_chip,
                                &priv->irq_data);
```

الـ `regmap_irq_chip` بتشتغل في threaded IRQ context (مش atomic)، فالـ I2C reads شغّالة بشكل صح.

**بديل للحالات البسيطة:** لو الـ register بـ MMIO (مش I2C/SPI):

```c
static const struct regmap_config sht31_mmio_cfg = {
    .fast_io = true,  /* spinlock → safe in IRQ context */
    ...
};
```

#### الدرس المستفاد
الـ `regmap_irq_chip` infrastructure في الـ header موجودة بالظبط عشان تتعامل مع الـ interrupt handling بشكل صح. لو محتاج تقرأ registers في الـ ISR، استخدم MMIO مع `fast_io=true`، أو delegate الـ IRQ handling لـ `regmap_irq_chip`.

---

### السيناريو 4: Custom Board Bring-up على i.MX8MPlus — الـ PAGED registers مش اتقروا صح

#### العنوان
**الـ `regmap_range_cfg` محتاج تعريف صح عشان تقرأ registers في pages مختلفة**

#### السياق
custom board بتعمل bring-up على i.MX8MPlus، فيه power management IC (BD71847) يتوصل بـ I2C. الـ IC عنده register map كبير منقسم على pages. الـ page selector هو register `0xFF`، وكل page فيها window من 64 register.

#### المشكلة
قراءة registers في الـ page 2 بترجع قيم page 0. المهندس بيشوف في الـ logic analyzer إن الـ page select register مش بيتكتب قبل الـ data read.

#### التحليل
الـ driver مش محدد الـ `ranges` في الـ `regmap_config`:

```c
/* خاطئ — لا ranges */
static const struct regmap_config bd71847_regmap = {
    .reg_bits = 8,
    .val_bits = 8,
    .max_register = 0x1FF,  /* المهندس عارف فيه pages لكن ما عرّفش */
};
```

بدون الـ `regmap_range_cfg`، الـ regmap بيحاول يكتب register رقم `0x140` مثلاً على الـ I2C مباشرة — طبعاً الـ IC هيتجاهله لأن الـ I2C register address محدود بـ 8-bit.

من الـ header، الـ `struct regmap_range_cfg` بيحدد:
```c
struct regmap_range_cfg {
    unsigned int range_min;    /* virtual address start */
    unsigned int range_max;    /* virtual address end */
    unsigned int selector_reg; /* page select register */
    unsigned int selector_mask;
    int          selector_shift;
    unsigned int window_start; /* first register in window */
    unsigned int window_len;   /* window size */
};
```

#### الحل

```c
/* Page 0: virtual 0x00–0x3F → physical 0x00–0x3F */
/* Page 1: virtual 0x40–0x7F → physical 0x00–0x3F (after page select) */
/* Page 2: virtual 0x80–0xBF → physical 0x00–0x3F */

static const struct regmap_range_cfg bd71847_ranges[] = {
    {
        .name          = "bd71847-page",
        .range_min     = 0x00,
        .range_max     = 0xFF,
        .selector_reg  = 0xFF,   /* page select register */
        .selector_mask = 0x03,   /* 2-bit page field */
        .selector_shift = 0,
        .window_start  = 0x00,
        .window_len    = 0x40,   /* 64 registers per page */
    },
};

static const struct regmap_config bd71847_regmap = {
    .reg_bits    = 8,
    .val_bits    = 8,
    .max_register = 0xFF,
    .ranges      = bd71847_ranges,
    .num_ranges  = ARRAY_SIZE(bd71847_ranges),
};
```

التحقق:

```bash
# اقرأ register في page 1 (virtual addr 0x50)
cat /sys/kernel/debug/regmap/1-004b/registers | grep "^0050:"
# لازم تشوف في logic analyzer: write 0x01 to 0xFF, then read 0x10
```

#### الدرس المستفاد
لما الـ IC عنده address space أكبر من الـ physical bus width، الـ `regmap_range_cfg` هو الحل. الـ regmap هيتكفل بكتابة الـ `selector_reg` تلقائياً قبل كل access للـ virtual address — بدون ما الـ driver يعرف بتفاصيل الـ paging.

---

### السيناريو 5: Automotive ECU على AM62x — race condition في الـ multi-threaded register access

#### العنوان
**الـ `disable_locking` يسبب data corruption في ECU بيشغّل threads متعددة**

#### السياق
automotive ECU بيشتغل على AM62x (TI)، بيراقب بيانات من sensors مختلفة. الـ PMIC (TPS65941) متوصل بـ I2C. لأن المهندس كان عايز performance أعلى، حط `disable_locking = true` في الـ `regmap_config` معتقداً إن الـ driver single-threaded.

#### المشكلة
بعد أسابيع في الـ testing، بيظهر corruption متقطع في الـ voltage readings، وأحياناً الـ ECU بتكتب قيمة غلط في register الـ protection لحظياً، مما بيطلّع safety alarm.

#### التحليل
الـ driver الأصلي:

```c
static const struct regmap_config tps65941_regmap = {
    .reg_bits       = 8,
    .val_bits       = 8,
    .disable_locking = true,  /* ← الخطأ */
};
```

من تعريف الـ `regmap_config` في الـ header:
```c
/* @disable_locking: This regmap is either protected by external means or
 *   is guaranteed not to be accessed from multiple threads.
 *   Don't use any locking mechanisms. */
bool disable_locking;
```

لاحقاً، اتضح إن الـ kernel workqueue والـ interrupt handler الاتنين بيعملوا access للـ regmap في نفس الوقت:

```
Thread A (workqueue):                Thread B (IRQ threaded handler):
regmap_update_bits_base()            regmap_read()
  → read register value                → read register value (same reg!)
  → modify
  → write (overwrites B's pending read result)
```

بدون الـ mutex، الـ read-modify-write مش atomic فعلياً على مستوى الـ software.

#### الحل

**الأول:** شيل الـ `disable_locking`:

```c
static const struct regmap_config tps65941_regmap = {
    .reg_bits = 8,
    .val_bits = 8,
    /* بدون disable_locking → mutex تلقائي */
};
```

**الثاني:** لو عايز تحسن الـ performance مع safety، استخدم external locking صريح:

```c
static DEFINE_MUTEX(tps65941_lock);

static void tps65941_lock_cb(void *arg)   { mutex_lock(arg); }
static void tps65941_unlock_cb(void *arg) { mutex_unlock(arg); }

static const struct regmap_config tps65941_regmap = {
    .reg_bits       = 8,
    .val_bits       = 8,
    .disable_locking = true,   /* نعطّل الـ internal lock */
    .lock           = tps65941_lock_cb,
    .unlock         = tps65941_unlock_cb,
    .lock_arg       = &tps65941_lock,
};
```

كده الـ locking موجود لكن تحت سيطرتك. التحقق من عدم وجود مشاكل:

```bash
# فعّل lockdep
CONFIG_LOCKDEP=y
CONFIG_PROVE_LOCKING=y
# في الـ dmesg راقب أي WARNING من lockdep
dmesg | grep -i "lock\|deadlock\|circular"
```

#### الدرس المستفاد
الـ `disable_locking` مش لتحسين الأداء — هو لحالات محددة جداً: إما الـ device protected بـ external lock، أو مضمون إنه single-threaded access. في الـ automotive contexts خصوصاً، الـ data corruption ممكن يطلّع safety violations حقيقية. الـ custom `lock`/`unlock` callbacks في الـ `regmap_config` بتدي flexibility كاملة مع الـ safety.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

أهم المصادر التقنية المتعلقة بـ regmap على LWN.net:

| المقال | الوصف |
|--------|-------|
| [regmap: Generic I2C and SPI register map library](https://lwn.net/Articles/451789/) | المقال الأساسي — يشرح تقديم الـ regmap framework في kernel 3.1 على يد Mark Brown |
| [regmap: introduce fast_io busses, and use a spinlock for them](https://lwn.net/Articles/490704/) | إضافة دعم الـ fast I/O buses واستخدام spinlock بدلاً من mutex |
| [regmap: Add asynchronous I/O support](https://lwn.net/Articles/534893/) | إضافة `regmap_raw_write_async()` و `regmap_async_complete()` للكتابة غير المتزامنة |
| [add syscon driver based on regmap for general registers access](https://lwn.net/Articles/514764/) | إضافة driver الـ syscon المبني على regmap للـ system controller registers |
| [Add simple NVMEM Framework via regmap](https://lwn.net/Articles/651711/) | دمج regmap مع NVMEM framework لتوحيد كود الـ non-volatile memory |
| [Add simple EEPROM Framework via regmap](https://lwn.net/Articles/638565/) | دمج regmap مع EEPROM framework |
| [regmap: provide simple bitops and use them in a driver](https://lwn.net/Articles/821711/) | إضافة `regmap_set_bits()` و `regmap_clear_bits()` و `regmap_test_bits()` |
| [regmap: mmio: Extending to support IO ports](https://lwn.net/Articles/904225/) | توسيع دعم الـ MMIO ليشمل IO ports |
| [Introduce a generic regmap-based MDIO driver](https://lwn.net/Articles/927209/) | driver MDIO عام مبني على regmap |
| [gpio: gpio-regmap: Support few custom operations](https://lwn.net/Articles/857361/) | إضافة custom operations لـ gpio-regmap |

---

### توثيق الـ kernel الرسمي

الملفات الموجودة في شجرة المصدر:

```
Documentation/driver-api/regmap.rst        ← التوثيق الرئيسي للـ API
include/linux/regmap.h                     ← تعريفات الـ API الكاملة مع kdoc
drivers/base/regmap/                       ← التنفيذ الكامل للـ subsystem
  ├── regmap.c                             ← الكود الأساسي
  ├── regmap-i2c.c                         ← bus backend للـ I2C
  ├── regmap-spi.c                         ← bus backend للـ SPI
  ├── regmap-mmio.c                        ← bus backend للـ MMIO
  ├── regcache.c                           ← إدارة الـ cache
  ├── regcache-rbtree.c                    ← rbtree cache backend
  ├── regcache-flat.c                      ← flat cache backend
  └── regcache-maple.c                     ← maple tree cache backend (kernel 6.x+)
```

لتوليد التوثيق محلياً:
```bash
make htmldocs
# الناتج في: Documentation/output/driver-api/regmap.html
```

---

### Commits رئيسية في التاريخ

| الـ Commit | الوصف |
|-----------|-------|
| [b83a313](https://github.com/torvalds/linux/commit/b83a313bf2520183641cf485d68cc273323597d2) | **الـ commit الأصلي** — "regmap: Add generic non-memory mapped register access API" — يوليو 2011 |
| [9943fa3](https://github.com/torvalds/linux/commit/9943fa300a5dcd4536e9a4aa5c09cf94e5e7b28c) | "regmap: Add I2C bus support" |
| [a676f08](https://github.com/torvalds/linux/commit/a676f083068b08e676c557279effbd7f4d590812) | "regmap: Add SPI bus support" |

الـ regmap tree المُصانة من قِبل Mark Brown:
```
git://git.kernel.org/pub/scm/linux/kernel/git/broonie/regmap.git
```

---

### نقاشات Mailing List

| الرابط | الموضوع |
|--------|---------|
| [dev_get_regmap() patch](https://lists.openwall.net/linux-kernel/2012/05/08/459) | مناقشة إضافة `dev_get_regmap()` للحصول على الـ regmap من device |
| [maple tree register cache](https://lore.kernel.org/all/20230325-regcache-maple-v2-0-799dcab3ecb1@kernel.org/) | مناقشة إضافة maple tree كـ cache backend جديد |
| [using the same regmap by multiple drivers](https://lists.archive.carbon60.com/linux/kernel/2473649) | مناقشة مشاركة نفس الـ regmap بين أكثر من driver |
| [Documentation for Regmap API?](https://www.mail-archive.com/kernelnewbies@kernelnewbies.org/msg22935.html) | سؤال على kernelnewbies عن توثيق الـ API |

للبحث في LKML مباشرة:
- [lore.kernel.org/linux-kernel/?q=regmap](https://lore.kernel.org/linux-kernel/?q=regmap)

---

### مقالات خارجية مُوصى بها

| الرابط | الوصف |
|--------|-------|
| [Using regmaps to make Linux drivers more generic — Collabora](https://www.collabora.com/news-and-blog/blog/2020/05/27/using-regmaps-to-make-linux-drivers-more-generic/) | مقال عملي من Collabora يشرح تحويل driver حقيقي ليستخدم regmap |
| [torvalds/linux — drivers/base/regmap](https://github.com/torvalds/linux/tree/master/drivers/base/regmap) | الكود المصدري الكامل على GitHub |
| [torvalds/linux — include/linux/regmap.h](https://github.com/torvalds/linux/blob/master/include/linux/regmap.h) | ملف الـ header الرئيسي |

---

### الكتب المُوصى بها

#### Linux Device Drivers 3rd Edition (LDD3)
- **المؤلفون**: Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصول ذات الصلة**:
  - Chapter 8: Allocating Memory — فهم الـ memory management المستخدم داخلياً
  - Chapter 14: The Linux Device Model — أساس الـ `struct device` الذي يعتمد عليه regmap
- **ملاحظة**: الكتاب قديم (2005) ولا يغطي regmap مباشرة، لكنه يُؤسّس للفهم الضروري
- **تحميل مجاني**: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصول ذات الصلة**:
  - Chapter 17: Devices and Modules — فهم الـ bus/driver model
  - Chapter 19: Portability — أهمية abstraction layers مثل regmap
- **الناشر**: Addison-Wesley, 2010

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصول ذات الصلة**:
  - Chapter 8: Device Driver Basics — I2C/SPI drivers والحاجة لـ regmap
  - Chapter 11: BusyBox — أمثلة على userspace التفاعل مع kernel drivers
- **الناشر**: Prentice Hall, 2010

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- يغطي الـ device model بعمق أكبر من LDD3 مع أمثلة حديثة نسبياً

---

### kernelnewbies.org

الـ regmap يُذكر في ملاحظات إصدارات kernel المختلفة على kernelnewbies:

| الإصدار | الرابط |
|---------|--------|
| Linux 4.9 | [kernelnewbies.org/Linux_4.9](https://kernelnewbies.org/Linux_4.9) |
| Linux 4.17 | [kernelnewbies.org/Linux_4.17](https://kernelnewbies.org/Linux_4.17) |
| Linux 6.11 | [kernelnewbies.org/Linux_6.11](https://kernelnewbies.org/Linux_6.11) |
| Linux 6.12 | [kernelnewbies.org/Linux_6.12](https://kernelnewbies.org/Linux_6.12) |
| Linux 6.14 | [kernelnewbies.org/Linux_6.14](https://kernelnewbies.org/Linux_6.14) |

ابحث عن "regmap" في كل صفحة لمشاهدة التغييرات لكل إصدار.

مشاريع Outreach المتعلقة بـ regmap على kernelnewbies:
- [OutreachyTasks](https://kernelnewbies.org/OutreachyTasks) — تحتوي على مهام تحويل IIO drivers لاستخدام regmap

---

### مصطلحات البحث

للبحث عن مزيد من المعلومات استخدم هذه المصطلحات:

```
# بحث عام
"linux regmap tutorial"
"linux kernel regmap example driver"
"regmap i2c spi driver linux"

# بحث في LKML
site:lore.kernel.org regmap
site:lwn.net regmap

# بحث في كود المصدر
grep -r "regmap_init" drivers/
grep -r "devm_regmap_init" drivers/
grep -rn "regmap_config" drivers/ --include="*.c" | head -30

# البحث في Elixir Cross Referencer
https://elixir.bootlin.com/linux/latest/source/include/linux/regmap.h
https://elixir.bootlin.com/linux/latest/source/drivers/base/regmap

# البحث عن drivers تستخدم regmap كمثال
grep -rl "regmap_read\|regmap_write" drivers/mfd/
grep -rl "devm_regmap_init_i2c" drivers/
```

---

### ملخص سريع للمصادر حسب الأولوية

| الأولوية | المصدر | لماذا؟ |
|----------|--------|--------|
| ★★★ | [LWN.net — المقال الأصلي](https://lwn.net/Articles/451789/) | يشرح السبب والتصميم الأولي |
| ★★★ | `Documentation/driver-api/regmap.rst` | التوثيق الرسمي الأحدث |
| ★★★ | `include/linux/regmap.h` | المرجع الكامل للـ API مع kdoc |
| ★★★ | [Collabora Blog](https://www.collabora.com/news-and-blog/blog/2020/05/27/using-regmaps-to-make-linux-drivers-more-generic/) | مثال عملي حديث |
| ★★☆ | [LWN — async I/O](https://lwn.net/Articles/534893/) | لفهم الكتابة غير المتزامنة |
| ★★☆ | [LWN — NVMEM](https://lwn.net/Articles/651711/) | لفهم تكامل regmap مع subsystems أخرى |
| ★☆☆ | LDD3 Chapter 14 | خلفية عن الـ device model |
## Phase 8: Writing simple module

### الفكرة

**الـ** `regmap_write` هي الدالة الأكثر استخداماً في الـ regmap API — كل driver بيكتب register على أي bus (I2C, SPI, MMIO) بيمر بيها. سنعمل **kprobe** عليها عشان نشوف كل write بيحصل في النظام: مين بيكتب، على أي register، وإيه القيمة.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * regmap_write kprobe monitor
 * Hooks regmap_write() to log every hardware register write system-wide.
 */

/* --- Includes --- */
#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kprobes.h>      /* kprobe struct + register/unregister */
#include <linux/regmap.h>       /* struct regmap (opaque pointer type) */
#include <linux/kallsyms.h>     /* for symbol name resolution in pr_info */

/*
 * بنحتاج kprobes.h عشان نعمل hook على دالة kernel في runtime بدون تعديل
 * الـ source. وبنحتاج regmap.h بس كـ forward declaration — الـ struct نفسه
 * opaque، مش محتاجين نفتحه.
 */

/* --- kprobe pre-handler: بيشتغل قبل تنفيذ regmap_write --- */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * على x86_64: أول 3 arguments بتيجي في rdi, rsi, rdx
     *   rdi = struct regmap *map
     *   rsi = unsigned int reg   (register address)
     *   rdx = unsigned int val   (value to write)
     *
     * بنطلع الـ reg والـ val من الـ registers مباشرةً عشان
     * نعرف إيه اللي بيتكتب من غير ما نمس الـ struct الـ opaque.
     */
    unsigned int reg = (unsigned int)regs->si;   /* 2nd arg */
    unsigned int val = (unsigned int)regs->dx;   /* 3rd arg */
    struct regmap *map = (struct regmap *)regs->di; /* 1st arg (opaque) */

    pr_info("regmap_probe: map=%px reg=0x%04x val=0x%08x comm=%s pid=%d\n",
            map, reg, val, current->comm, current->pid);

    return 0; /* 0 = let the original function execute normally */
}

/*
 * الـ handler_pre هو اللي بيشتغل "قبل" الـ instruction الأولى في
 * regmap_write. بنطبع فيه اسم الـ process اللي طلب الـ write
 * وعنوان الـ register والقيمة — معلومات مفيدة لـ debugging.
 */

/* --- kprobe struct --- */
static struct kprobe kp = {
    .symbol_name = "regmap_write",  /* اسم الدالة اللي هنضع فيها الـ probe */
    .pre_handler = handler_pre,     /* callback قبل التنفيذ */
};

/*
 * الـ kprobe struct بيحدد "فين" نضع الـ hook و"إيه" الـ callback.
 * الـ symbol_name بيخلي الـ kernel يحول الاسم لعنوان تلقائياً
 * عن طريق kallsyms بدون ما نحتاج نعرف الـ address يدوياً.
 */

/* --- module_init --- */
static int __init regmap_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("regmap_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("regmap_probe: planted kprobe at %s (%px)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/*
 * الـ register_kprobe بتكتب breakpoint instruction (int3 على x86)
 * عند بداية regmap_write في الذاكرة. من الضروري نتأكد إنها نجحت
 * قبل ما نعلن إن الـ module اشتغل.
 */

/* --- module_exit --- */
static void __exit regmap_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("regmap_probe: kprobe removed from %s\n", kp.symbol_name);
}

/*
 * الـ unregister_kprobe ضروري في exit عشان نرجع الـ original instruction
 * اللي استبدلناها بالـ breakpoint. لو ما عملناش ده والـ module اتـ unload،
 * الـ CPU هيلاقي callback pointer بيشاور على كود اتشالت من الذاكرة
 * → kernel panic مضمون.
 */

module_init(regmap_probe_init);
module_exit(regmap_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on regmap_write to trace all HW register writes");
```

---

### Makefile للتجميع

```makefile
obj-m += regmap_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/module.h` | ماكروهات الـ module الأساسية |
| `linux/kprobes.h` | الـ `struct kprobe` وكل دوال التسجيل |
| `linux/regmap.h` | عشان الـ compiler يعرف `struct regmap` كـ forward decl |
| `linux/kallsyms.h` | بيُستخدم internally من kprobes لترجمة الاسم لعنوان |

**الـ** `regmap.h` مش محتاجينه عشان نفتح الـ struct — الـ struct نفسه opaque ومش معرّف للـ external users. بنشيله بس عشان الـ compiler ما يشتكيش من الـ cast.

---

#### الـ `handler_pre` — قراءة الـ arguments

على الـ **x86_64 calling convention** (System V ABI):
- `rdi` ← argument 1 (`struct regmap *map`)
- `rsi` ← argument 2 (`unsigned int reg`)
- `rdx` ← argument 3 (`unsigned int val`)

الـ `struct pt_regs` بيحفظ قيم الـ registers عند لحظة الـ probe، فبنقدر نقرأ الـ arguments مباشرةً بدون ما نعمل أي تعديل على الكود الأصلي.

الـ `return 0` بيقول للـ kprobe infrastructure: "كمّل تنفيذ الدالة الأصلية عادي."

---

#### الـ `struct kprobe`

```
.symbol_name = "regmap_write"
```
الـ kernel هيدور على الاسم ده في الـ kallsyms table وهيحول الاسم لعنوان تلقائياً. لو الدالة مش exported أو مش موجودة في الـ symbol table، الـ `register_kprobe` هترجع `-ENOENT`.

---

#### ليه `regmap_write` وليس `__regmap_init`؟

`regmap_write` هي الأكثر تكراراً وحدوثاً في runtime — كل driver بيفعّل نفسه بيكتب registers. `__regmap_init` بيحصل مرة واحدة عند init، فالـ output هيكون محدود. الـ `regmap_write` بيديك stream حي من كل hardware register access في النظام.

---

### تشغيل وقراءة الـ output

```bash
# تحميل الـ module
sudo insmod regmap_probe.ko

# مراقبة الـ output في real time
sudo dmesg -w | grep regmap_probe

# مثال على output:
# regmap_probe: planted kprobe at regmap_write (ffffffffc0a1b230)
# regmap_probe: map=ffff888003a1c000 reg=0x0010 val=0x00000001 comm=i2c_worker pid=142
# regmap_probe: map=ffff888003a1c000 reg=0x0011 val=0x000000ff comm=snd_soc_rt5651 pid=89

# إزالة الـ module
sudo rmmod regmap_probe
```

---

### ملاحظات أمان

- الـ module ده **read-only** — بيراقب بس، مش بيعدل القيم.
- الـ `pr_info` من context بيشغّل الـ driver الأصلي، مش من interrupt context، فبنقدر نستخدمه بأمان.
- لو النظام عنده **كتير من الـ regmap writes** (مثلاً SoC مع audio codec)، ممكن الـ dmesg يتملى بسرعة — استخدم `grep` لتصفية driver معين عن طريق الـ `comm` field.
