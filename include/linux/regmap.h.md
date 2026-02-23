## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الـ `regmap.h` ينتمي لـ subsystem اسمه **REGISTER MAP ABSTRACTION** — موجود في `drivers/base/regmap/` ومُصان من Mark Brown (Wolfson Microelectronics أصلاً، دلوقتي Kernel.org).

---

### المشكلة اللي بيحلها regmap — قصة حقيقية

تخيل إنك بتكتب driver لـ chip صوت (audio codec) زي WM8731 من Wolfson. الـ chip ده فيها 10 registers، كل register 9 bits. عشان تكلمها عن طريق I2C بتعمل كده:

```c
/* بدون regmap — كل driver بيعيد اختراع العجلة */
i2c_smbus_write_byte_data(client, reg_addr, value);
```

طيب كمان في driver تاني لـ chip تانية بتتكلم عن SPI بنفس الفكرة:

```c
spi_write(spi, buf, len);
```

وفي chip تالتة MMIO (Memory-Mapped I/O) مباشرة، ورابعة على SPMI، وخامسة على SoundWire...

**المشكلة:** كل driver بيكرر نفس الـ boilerplate:
- قراءة وكتابة registers
- الـ locking (mutex / spinlock)
- الـ caching (لو السـ hardware بطيء، تخزن قيم الـ registers في الـ RAM)
- الـ byte-order (big-endian / little-endian)
- الـ debugfs entries (عشان تشوف قيم الـ registers من `/sys/kernel/debug`)
- الـ IRQ demultiplexing (chip واحدة فيها 16 interrupt على خط IRQ واحد)

**الحل:** `regmap` — طبقة abstraction موحدة بتقول للـ driver: "انت بس قولي رقم الـ register والقيمة، وأنا أتكلم مع الـ hardware على أي bus."

---

### الصورة الكبيرة بالتفصيل

```
┌─────────────────────────────────────────────┐
│              Driver Code                    │
│  regmap_write(map, REG_VOLUME, 0x50)        │
│  regmap_read(map, REG_STATUS, &val)         │
│  regmap_update_bits(map, REG_CTRL, 0x3, 1) │
└──────────────┬──────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────┐
│           regmap core (regmap.c)            │
│  • Locking (mutex / spinlock)               │
│  • Cache check (هل القيمة موجودة في Cache?) │
│  • Register validation (readable? volatile?)│
│  • Byte formatting (big/little endian)      │
└────┬──────────┬──────────┬──────────────────┘
     │          │          │
     ▼          ▼          ▼
  regmap-   regmap-    regmap-
   i2c.c    spi.c      mmio.c
  (I2C bus) (SPI bus)  (MMIO)
     │          │          │
     ▼          ▼          ▼
  i2c_       spi_       iowrite32/
  transfer() write()    ioread32()
```

---

### ليه regmap.h مهم؟

الـ `include/linux/regmap.h` هو الـ **public API** الكامل للـ subsystem. أي driver بيستخدم regmap بيعمل `#include <linux/regmap.h>` بس، ومحتاجش يعرف أي حاجة عن الـ internal implementation.

**الملف ده بيعرف:**

| المكون | الوظيفة |
|--------|---------|
| `struct regmap_config` | إعداد الـ regmap: عدد bits في الـ address، الـ cache type، الـ callbacks للـ volatile/precious registers |
| `struct regmap_bus` | واجهة الـ bus: دوال read/write الخاصة بكل bus |
| `struct regmap_range_cfg` | للـ chips اللي عندها paged registers (عنونة غير مباشرة) |
| `struct reg_field` | وصف field معين داخل register (bits محددة) |
| `struct regmap_irq_chip` | وصف IRQ controller داخل chip للـ demultiplexing |
| `enum regcache_type` | نوع الـ cache: NONE / RBTREE / FLAT / MAPLE |
| `regmap_init_*()` macros | تهيئة regmap لكل bus |
| `regmap_read/write/update_bits()` | عمليات القراءة والكتابة الأساسية |
| `regcache_sync/cache_only/mark_dirty()` | إدارة الـ cache |

---

### قصة الـ Cache — ليه مهم جداً؟

تخيل chip صوت متصلة بـ I2C على سرعة 400kHz. لو كل مرة عايز تغير volume بتعمل I2C transaction كاملة — ده بطيء جداً.

الـ `regcache` بيخزن قيم الـ registers في الـ RAM. لما تعمل `regmap_write()` الـ core بيكتب في الـ cache وفي الـ hardware في نفس الوقت. لما الجهاز يدخل في **suspend**، الـ hardware بيفقد قيمه، فلما يصحى تعمل `regcache_sync()` تعيد كل القيم المخزنة للـ hardware دفعة واحدة.

```
suspend → hardware فقد القيم → cache لسه عندها كل حاجة
resume  → regcache_sync() → تكتب كل القيم المتغيرة للـ hardware
```

**أنواع الـ cache:**
- `REGCACHE_MAPLE` — الافتراضي الجديد، maple tree، كفاءة ممتازة
- `REGCACHE_RBTREE` — قديم، red-black tree، للأنظمة الـ low-end
- `REGCACHE_FLAT` — array بسيط، للـ chips اللي registers عندها consecutive
- `REGCACHE_FLAT_S` — sparse flat، مثل FLAT بس بـ lazy initialization
- `REGCACHE_NONE` — من غير cache خالص

---

### قصة الـ IRQ Demultiplexing

الـ PMIC (Power Management IC) زي MAX77686 ممكن يكون فيه 20+ interrupt source (شحن البطارية، over-voltage، thermal، إلخ)، بس متصل بالـ CPU بخط IRQ واحد بس.

الـ `regmap_irq_chip` بيحل المشكلة دي:
1. الـ hardware يرفع الـ IRQ line
2. الـ regmap handler يقرأ الـ status registers عن طريق الـ regmap
3. يحدد أي interrupts اتفعلت
4. يطلق الـ virtual IRQs المقابلة لكل driver محتاجها

```c
/* Driver بيعرف الـ IRQ chip */
static const struct regmap_irq_chip my_irq_chip = {
    .name        = "my_pmic",
    .status_base = STATUS_REG,
    .mask_base   = MASK_REG,
    .irqs        = my_irqs,
    .num_irqs    = ARRAY_SIZE(my_irqs),
    .num_regs    = 2,
};
```

---

### الـ reg_field — قراءة bits معينة

بدل ما تعمل masking يدوي في كل مكان:

```c
/* بدون reg_field — verbose */
regmap_read(map, CTRL_REG, &val);
val = (val >> 4) & 0x3;  /* bits 5:4 */

/* مع reg_field — نظيف */
static const struct reg_field vol_field = REG_FIELD(CTRL_REG, 4, 5);
struct regmap_field *field = devm_regmap_field_alloc(dev, map, vol_field);
regmap_field_read(field, &val);
```

---

### الملفات المكونة للـ Subsystem

#### Core
| الملف | الوظيفة |
|-------|---------|
| `drivers/base/regmap/regmap.c` | الـ core الأساسي: init، read، write، update_bits |
| `drivers/base/regmap/regcache.c` | إدارة الـ cache، sync، mark_dirty |
| `drivers/base/regmap/regmap-debugfs.c` | واجهة `/sys/kernel/debug/regmap/` |
| `drivers/base/regmap/regmap-irq.c` | الـ IRQ demultiplexer |
| `drivers/base/regmap/internal.h` | الـ internal struct `regmap` الحقيقية (مخفية عن الـ drivers) |

#### Cache Backends
| الملف | الوظيفة |
|-------|---------|
| `drivers/base/regmap/regcache-maple.c` | Maple tree cache (الافتراضي الجديد) |
| `drivers/base/regmap/regcache-rbtree.c` | Red-black tree cache |
| `drivers/base/regmap/regcache-flat.c` | Flat array cache |

#### Bus Backends
| الملف | الوظيفة |
|-------|---------|
| `drivers/base/regmap/regmap-i2c.c` | I2C bus backend |
| `drivers/base/regmap/regmap-spi.c` | SPI bus backend |
| `drivers/base/regmap/regmap-mmio.c` | Memory-Mapped I/O backend |
| `drivers/base/regmap/regmap-spmi.c` | SPMI (Qualcomm) backend |
| `drivers/base/regmap/regmap-sdw.c` | SoundWire backend |
| `drivers/base/regmap/regmap-sdw-mbq.c` | SoundWire Multi-Byte Quantity backend |
| `drivers/base/regmap/regmap-i3c.c` | I3C bus backend |
| `drivers/base/regmap/regmap-slimbus.c` | SLIMbus backend |
| `drivers/base/regmap/regmap-ac97.c` | AC'97 audio bus backend |
| `drivers/base/regmap/regmap-w1.c` | 1-Wire backend |
| `drivers/base/regmap/regmap-fsi.c` | FSI (IBM) backend |
| `drivers/base/regmap/regmap-spi-avmm.c` | Intel SPI-to-AVMM bridge |
| `drivers/base/regmap/regmap-mdio.c` | MDIO (Ethernet PHY) backend |

#### Header
| الملف | الوظيفة |
|-------|---------|
| `include/linux/regmap.h` | الـ public API الكامل (الملف ده) |

---

### الملفات القريبة اللي المفروض تعرفها

- `include/linux/i2c.h` — لفهم `struct i2c_client` اللي بتتمرر لـ `regmap_init_i2c()`
- `include/linux/spi/spi.h` — لفهم `struct spi_device`
- `include/linux/irq.h` و `include/linux/irqdomain.h` — لفهم الـ IRQ subsystem اللي regmap-irq بيتكامل معاه
- `drivers/base/regmap/internal.h` — لو عايز تفهم الـ internal struct `regmap` الحقيقية
- `Documentation/devicetree/bindings/regmap/` — الـ DT bindings للـ regmap
## Phase 2: شرح الـ Regmap Framework

### المشكلة — ليه الـ Regmap موجود أصلاً؟

في أي SoC أو embedded board هتلاقي عشرات الـ chips خارجية: codec صوت، PMIC، sensor، ethernet PHY، كل واحدة فيهم بتتكلم مع الـ CPU عبر بروتوكول مختلف — I2C، SPI، MMIO، SPMI، SoundWire، وغيرهم. وكل واحدة جوّاها عشرات الـ registers بتتحكم في كل حاجة.

المشكلة الحقيقية؟ كل درايفر كان بيكتب نفس الكود من أول وجديد:
- قراءة وكتابة registers عبر الـ bus
- حفظ قيم الـ registers في الـ cache عشان ما يحصلش I/O زيادة
- تطبيق الـ read-modify-write (RMW) بأمان مع locking
- التحكم في volatile vs cacheable registers
- إدارة الـ IRQs اللي بتيجي من الـ chip عبر interrupt registers

النتيجة كانت code duplication ضخمة — كل صاحب درايفر بيعيد اختراع العجلة، وبيعمل bugs مختلفة في نفس الأماكن.

---

### الحل — الـ Regmap Framework

الـ **regmap** هو abstraction layer بيجلس بين الـ device driver وبين أي bus تحته. بدل ما الدرايفر يبعت raw I2C messages أو raw SPI transfers، هو بيكلم الـ regmap بـ API موحدة:

```c
regmap_read(map, REG_STATUS, &val);
regmap_write(map, REG_CTRL, 0x01);
regmap_update_bits(map, REG_CTRL, BIT(3), BIT(3)); /* set bit 3 only */
```

الـ regmap من جوّه بيتولى:
1. الـ **locking** — mutex أو spinlock حسب السياق
2. الـ **caching** — هيرجع من الـ cache لو الـ register مش volatile
3. الـ **bus translation** — يحوّل الطلب لـ I2C transaction أو SPI frame أو MMIO write
4. الـ **endianness** — يضبط byte order حسب الـ hardware
5. الـ **access validation** — يتأكد إن الـ register readable/writable

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      DEVICE DRIVER LAYER                        │
│  (wm8960 audio codec / max77686 PMIC / phy-generic ethernet)    │
│                                                                  │
│   regmap_read()  regmap_write()  regmap_update_bits()           │
│   regmap_field_read()  regmap_bulk_write()                      │
└──────────────────────────┬──────────────────────────────────────┘
                           │  Generic register access API
┌──────────────────────────▼──────────────────────────────────────┐
│                      REGMAP CORE                                 │
│                                                                  │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │  LOCKING    │  │   REGCACHE   │  │   ACCESS VALIDATION    │ │
│  │ mutex/spin  │  │ rbtree/maple │  │  readable/volatile/    │ │
│  │  hwlock     │  │ flat/none    │  │  precious/writable     │ │
│  └─────────────┘  └──────────────┘  └────────────────────────┘ │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              regmap_bus (vtable of hw callbacks)           │ │
│  │  .write  .read  .reg_write  .reg_read  .gather_write      │ │
│  └────────────────────────────────────────────────────────────┘ │
└──────────────────────────┬──────────────────────────────────────┘
                           │  Bus-specific implementation
         ┌─────────────────┼──────────────────────┐
         │                 │                       │
┌────────▼──────┐  ┌───────▼──────┐  ┌────────────▼────────┐
│   regmap-i2c  │  │  regmap-spi  │  │   regmap-mmio        │
│ i2c_transfer()│  │ spi_sync()   │  │ readl()/writel()     │
└───────────────┘  └──────────────┘  └─────────────────────┘
         │                 │                       │
┌────────▼──────┐  ┌───────▼──────┐  ┌────────────▼────────┐
│  I2C Hardware │  │ SPI Hardware │  │  MMIO Hardware       │
└───────────────┘  └──────────────┘  └─────────────────────┘
```

**الـ regmap_irq** هو extension منفصل فوق الـ core — بيترجم interrupt registers في الـ chip لـ virtual IRQs في الـ Linux IRQ subsystem.

---

### التشبيه الواقعي — الـ Database ORM

فكّر في الـ regmap كـ **ORM (Object-Relational Mapper)** بالظبط:

| المفهوم في الـ ORM | المفهوم في الـ Regmap |
|---|---|
| الـ database نفسها (MySQL/PostgreSQL) | الـ bus (I2C/SPI/MMIO) |
| جدول في الـ database | الـ register map في الـ chip |
| صف في الجدول | register واحد بعنوانه وقيمته |
| الـ ORM connection pool | الـ regmap instance (`struct regmap *`) |
| الـ ORM schema/config | `struct regmap_config` |
| الـ ORM driver (pg adapter) | `struct regmap_bus` |
| الـ query cache | الـ regcache |
| الـ schema migration | `regmap_register_patch()` |
| الـ transaction | الـ locking في الـ RMW |
| `SELECT col FROM table WHERE id=1` | `regmap_read(map, REG_ADDR, &val)` |
| `UPDATE table SET col=val WHERE id=1` | `regmap_write(map, REG_ADDR, val)` |
| `UPDATE table SET col = col & mask | val` | `regmap_update_bits()` |
| الـ view (subset of columns) | `struct regmap_field` |
| الـ read-only columns | `readable_reg` callback |
| الـ non-cacheable computed column | `volatile_reg` callback |

تماماً زي ما الـ ORM بيحميك من كتابة raw SQL وبيوفرلك features زي الـ connection pooling والـ caching — الـ regmap بيحميك من كتابة raw bus transactions وبيوفرلك features جاهزة.

---

### الـ Core Abstraction — الفكرة المحورية

**الـ central idea هي: كل chip بـ registers هي database صغيرة، والـ regmap هو الـ unified I/O engine لها.**

الـ abstraction بتقوم على ثلاث طبقات:

```
┌──────────────────────────────────────────────────────────────┐
│  LAYER 1: WHAT (ماذا نريد؟)                                  │
│  regmap_read/write/update_bits — Generic register access     │
├──────────────────────────────────────────────────────────────┤
│  LAYER 2: HOW SMART (كيف نتعامل مع الـ register map؟)       │
│  regmap_config — rules: volatile? precious? cacheable?       │
│  regcache — saves I/O by caching last known values           │
├──────────────────────────────────────────────────────────────┤
│  LAYER 3: HOW (كيف نصل للـ hardware؟)                       │
│  regmap_bus — bus-specific callbacks: I2C / SPI / MMIO       │
└──────────────────────────────────────────────────────────────┘
```

---

### الـ Core Structs — كيف بتتشابك مع بعض

```
struct regmap_config          struct regmap_bus
 ┌─────────────────┐           ┌─────────────────┐
 │ reg_bits        │           │ .write()        │
 │ val_bits        │           │ .read()         │
 │ volatile_reg()  │           │ .reg_write()    │
 │ readable_reg()  │           │ .reg_read()     │
 │ cache_type      │           │ .gather_write() │
 │ reg_defaults[]  │           │ fast_io         │
 │ wr_table        │           └────────┬────────┘
 └────────┬────────┘                    │
          │  regmap_init(dev, bus, ctx, config)
          │                             │
          └──────────────┬──────────────┘
                         ▼
               ┌─────────────────────────────┐
               │      struct regmap *map      │
               │  (opaque to the driver)      │
               │                             │
               │  ┌──────────────────────┐   │
               │  │  regcache backend    │   │
               │  │  (rbtree/maple/flat) │   │
               │  └──────────────────────┘   │
               │  ┌──────────────────────┐   │
               │  │  lock (mutex/spin)   │   │
               │  └──────────────────────┘   │
               │  ┌──────────────────────┐   │
               │  │  format/endian layer │   │
               │  └──────────────────────┘   │
               └─────────────────────────────┘
                         │
           ┌─────────────┴──────────────┐
           ▼                            ▼
   regmap_field *                regmap_irq_chip
   (sub-register bit field)      (IRQ controller over regmap)
```

---

### الـ struct regmap_config — قلب الـ Configuration

**الـ `struct regmap_config`** هو الـ descriptor اللي الدرايفر بيحدد فيه كل خصائص الـ chip:

```c
static const struct regmap_config wm8960_regmap = {
    .reg_bits = 7,           /* 7-bit register address */
    .val_bits = 9,           /* 9-bit register value */

    /* tell regmap which registers can be cached safely */
    .volatile_reg = wm8960_volatile,
    .readable_reg = wm8960_readable,

    /* power-on reset defaults for cache initialization */
    .reg_defaults = wm8960_reg_defaults,
    .num_reg_defaults = ARRAY_SIZE(wm8960_reg_defaults),

    /* use maple tree as cache backend */
    .cache_type = REGCACHE_MAPLE,
};
```

**أهم الـ callbacks في الـ `regmap_config`:**

| Callback | المعنى |
|---|---|
| `volatile_reg(dev, reg)` | لو رجع `true`، الـ register دايماً بيتقرأ من الـ hardware (مش من الـ cache) |
| `readable_reg(dev, reg)` | لو رجع `false`، الـ regmap بيرفض القراءة |
| `precious_reg(dev, reg)` | الـ register clear-on-read (IRQ status مثلاً) — ما يتقراش بدون سبب |
| `writeable_reg(dev, reg)` | الـ register للكتابة فقط |

---

### الـ struct regmap_bus — بوابة الـ Hardware

**الـ `struct regmap_bus`** هو الـ vtable اللي بيوصل الـ regmap core بالـ bus driver:

```c
/* regmap-i2c.c: I2C bus backend */
static const struct regmap_bus regmap_i2c = {
    .write = regmap_i2c_write,       /* bulk write: reg + val in one transaction */
    .gather_write = regmap_i2c_gather_write,
    .read  = regmap_i2c_read,        /* write reg addr, then read val */
    .reg_format_endian_default = REGMAP_ENDIAN_BIG,
    .val_format_endian_default = REGMAP_ENDIAN_BIG,
};
```

الدرايفر نفسه **مش محتاج يعرف** إن `regmap_bus` موجود — ده للي بيكتب I2C/SPI backend جديد. الدرايفر العادي بس بيعمل:

```c
/* في probe()، بس كده — مفيش I2C calls صريحة بعدها */
priv->regmap = devm_regmap_init_i2c(client, &wm8960_regmap);
```

---

### الـ Regcache — الـ Caching Layer

الـ **regcache** بيخزّن آخر قيمة معروفة لكل register. عند الـ `regmap_read`:

```
regmap_read(map, REG, &val)
        │
        ▼
  volatile_reg(REG)?
   YES ──────────────────► read from hardware via regmap_bus
    │
   NO
    │
    ▼
  value in cache?
   YES ──────────────────► return cached value (zero hardware I/O)
    │
   NO
    │
    ▼
  read from hardware, store in cache, return value
```

**أنواع الـ cache backends:**

| النوع | الاستخدام |
|---|---|
| `REGCACHE_NONE` | مفيش cache خالص، كل حاجة من الـ hardware |
| `REGCACHE_FLAT` | array ثابت، سريع جداً، بس بياخد ذاكرة بغض النظر عن عدد الـ registers الفعلية |
| `REGCACHE_FLAT_S` | نفسه بس sparse (ما بيخزنش entries بدون default) |
| `REGCACHE_RBTREE` | red-black tree، كويس لـ sparse maps، legacy في معظم حالاته |
| `REGCACHE_MAPLE` | maple tree، الاختيار الـ default الحديث، أفضل في الذاكرة والأداء |

**الـ cache suspend/resume flow** — مهم جداً في embedded:

```c
/* عند الـ system suspend */
regcache_cache_only(map, true);   /* stop all hardware I/O */
regcache_mark_dirty(map);         /* mark all cached regs as dirty */

/* عند الـ system resume */
regcache_sync(map);               /* write dirty cache back to hardware */
regcache_cache_only(map, false);  /* resume normal operation */
```

---

### الـ regmap_field — Sub-Register Access

أحياناً بتحتاج تتحكم في bits محددة جوّا register واحد. **الـ `struct reg_field`** بيعرّف position الـ field:

```c
/* define: register 0x20, bits [5:3] */
static const struct reg_field sample_rate_field = REG_FIELD(0x20, 3, 5);

/* allocate a handle for it */
priv->sample_rate = devm_regmap_field_alloc(dev, map, sample_rate_field);

/* read/write without manual bit shifting */
regmap_field_read(priv->sample_rate, &rate);
regmap_field_write(priv->sample_rate, 4);  /* writes 0b100 to bits [5:3] */
```

الـ regmap تحت الغطا بيعمل الـ masking والـ shifting أوتوماتيك — مفيش خطأ بشري في حساب الـ mask.

```
Register 0x20:  [ 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 ]
                              ┗━━━━━┛
                         sample_rate_field
                         lsb=3, msb=5
                         mask = 0b00111000 = 0x38
```

---

### الـ regmap_irq — IRQ Controller on Top of Regmap

**الـ IRQ subsystem** (لازم تعرفه أولاً): نظام الـ Linux لإدارة الـ hardware interrupts، بيستخدم `irq_domain` لتحويل hardware IRQ numbers لـ virtual Linux IRQs.

كتير من الـ PMICs والـ codecs عندهم interrupt registers — register فيه bits تدل على مصدر الـ interrupt. الـ **`struct regmap_irq_chip`** بيعرّف هيكل الـ interrupt registers دي وبيخلي الـ regmap يلعب دور الـ IRQ controller:

```c
static const struct regmap_irq wm8960_irqs[] = {
    [WM8960_IRQ_TEMP] = REGMAP_IRQ_REG(WM8960_IRQ_TEMP, 0, BIT(2)),
};

static const struct regmap_irq_chip wm8960_irq_chip = {
    .name       = "wm8960",
    .status_base = WM8960_INT_STATUS,   /* read here to know which IRQ fired */
    .mask_base   = WM8960_INT_MASK,     /* write here to mask/unmask */
    .ack_base    = WM8960_INT_ACK,      /* write here to clear IRQ */
    .irqs        = wm8960_irqs,
    .num_irqs    = ARRAY_SIZE(wm8960_irqs),
    .num_regs    = 1,
};
```

الـ `regmap_add_irq_chip()` بيسجل الـ chip ده كـ `irq_domain` في الـ Linux IRQ system — وبعدين الدرايفرات التانية اللي محتاجة تستقبل interrupts من الـ chip بتطلب virtual IRQ عبر `regmap_irq_get_virq()`.

```
Physical IRQ (GPIO line to SoC)
        │
        ▼
regmap_irq_handler()
        │
        ├─ reads status_base register(s) via regmap_read()
        ├─ identifies which sub-IRQs fired
        └─ calls handle_nested_irq() for each virtual IRQ
                │
                ▼
        Driver's IRQ handler (e.g., charger IRQ handler)
```

---

### الـ Paged Registers — الـ `regmap_range_cfg`

بعض الـ chips عندها address space أكبر من اللي الـ register address field يقدر يعبّر عنه. بيستخدموا الـ **paging** — بتكتب في page selector register الأول، وبعدين تقرأ/تكتب من window registers محددة.

الـ `struct regmap_range_cfg` بيخلي الـ regmap يتعامل مع الـ paging أوتوماتيك:

```c
static const struct regmap_range_cfg chip_ranges[] = {
    {
        .name         = "main",
        .range_min    = 0x000,   /* virtual address start */
        .range_max    = 0x1FF,   /* virtual address end */
        .selector_reg = 0x00,   /* write page number here */
        .selector_mask= 0x0F,
        .selector_shift = 0,
        .window_start = 0x10,   /* actual hardware registers start here */
        .window_len   = 0x10,   /* 16 registers per page */
    },
};
```

الدرايفر بيشوف عنوان virtual `0x1A5` والـ regmap تحت الغطا بيحسب الـ page number ويكتبه في `selector_reg` الأول، وبعدين يوصل للـ window.

---

### الـ devm_ Pattern

الـ regmap بيدعم الـ **devres (device resource management)** الـ pattern. أي حاجة بـ `devm_` prefix بيتم تحريرها أوتوماتيك لما الـ device يتفصل — من غير ما الدرايفر يعمل cleanup يدوي:

```c
/* في .probe() */
map = devm_regmap_init_i2c(client, &config);
/* مفيش .remove() محتاجة تعمل regmap_exit() */
```

---

### الـ Regmap يمتلك — والـ Regmap يفوّض

| الـ Regmap يمتلك (owns) | الـ Regmap يفوّض للـ Driver / Bus |
|---|---|
| الـ locking strategy (mutex/spinlock) | تحديد `fast_io` flag (في الـ config) |
| الـ caching layer وكل logic تبعته | تحديد `volatile_reg` / `precious_reg` callbacks |
| الـ RMW atomicity | اختيار نوع الـ cache (`REGCACHE_MAPLE` إلخ) |
| ترجمة addresses في الـ paged mode | كتابة `reg_defaults[]` |
| الـ endianness formatting | تنفيذ `read()/write()` في الـ `regmap_bus` |
| سجل الـ IRQs وتوزيعها | تعريف `regmap_irq_chip` وهيكل الـ IRQ registers |
| الـ debugfs entries للـ registers | اختيار الـ locking callbacks لو محتاج custom locking |
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Enums والـ Config Options المهمة

#### `enum regcache_type` — نوع الـ cache

| القيمة | المعنى |
|---|---|
| `REGCACHE_NONE` | مفيش cache خالص — كل قراءة/كتابة بتروح للـ hardware مباشرة |
| `REGCACHE_RBTREE` | cache بـ red-black tree — legacy، مناسب لـ low-end systems |
| `REGCACHE_FLAT` | array ثابت non-sparse — بيصفّر entries اللي ملهاش default |
| `REGCACHE_MAPLE` | الافتراضي الجديد — maple tree، sparse، بيعمل allocation في runtime |
| `REGCACHE_FLAT_S` | flat لكنه sparse — ما بيصفّرش entries اللي ملهاش default |

> الجديد: استخدم `REGCACHE_MAPLE` دايمًا إلا لو محتاج zero-allocation في runtime → `REGCACHE_FLAT_S`.

---

#### `enum regmap_endian` — ترتيب البايتات

| القيمة | المعنى |
|---|---|
| `REGMAP_ENDIAN_DEFAULT` | اتبع الـ bus default (الـ BIG عادةً) |
| `REGMAP_ENDIAN_BIG` | big-endian صريح |
| `REGMAP_ENDIAN_LITTLE` | little-endian صريح |
| `REGMAP_ENDIAN_NATIVE` | endianness الـ CPU نفسه |

---

#### Macros الـ MDIO Clause 45

| Macro | القيمة | الغرض |
|---|---|---|
| `REGMAP_MDIO_C45_DEVAD_SHIFT` | 16 | الـ device address في bits [20:16] |
| `REGMAP_MDIO_C45_DEVAD_MASK` | `GENMASK(20,16)` | mask لاستخراج device address |
| `REGMAP_MDIO_C45_REGNUM_MASK` | `GENMASK(15,0)` | mask للـ register number |
| `REGMAP_UPSHIFT(s)` | `-(s)` | upshift للـ register address |
| `REGMAP_DOWNSHIFT(s)` | `(s)` | downshift للـ register address |

---

### كل الـ Structs المهمة

---

#### `struct reg_default`

**الغرض:** بيخزّن قيمة افتراضية (Power-on Reset) لـ register واحد — بتستخدمه مع الـ cache لإعادة populate الـ cache بعد resume.

```c
struct reg_default {
    unsigned int reg;  /* عنوان الـ register */
    unsigned int def;  /* القيمة الافتراضية */
};
```

**الارتباط:** بيتحط في `regmap_config.reg_defaults[]` — الـ regmap core بيقرأه ويملي الـ cache بيه عند الـ init.

---

#### `struct reg_sequence`

**الغرض:** كتابة sequence من registers بترتيب مع delay اختياري بعد كل كتابة — مهم جدًا لـ hardware initialization sequences.

```c
struct reg_sequence {
    unsigned int reg;       /* عنوان الـ register */
    unsigned int def;       /* القيمة المكتوبة */
    unsigned int delay_us;  /* delay بعد الكتابة بالـ microseconds */
};
```

**مثال واقعي** — init sequence لـ codec:
```c
static const struct reg_sequence wm8960_init[] = {
    { WM8960_RESET,   0x000 },
    { WM8960_POWER1,  0x0C0, 100 },  /* 100µs بعد power-on */
    { WM8960_DACCTL1, 0x000 },
};
regmap_multi_reg_write(map, wm8960_init, ARRAY_SIZE(wm8960_init));
```

---

#### `struct regmap_range`

**الغرض:** بيعرّف نطاق عناوين registers (min → max) لأغراض الـ access control.

```c
struct regmap_range {
    unsigned int range_min;  /* أول عنوان في النطاق */
    unsigned int range_max;  /* آخر عنوان في النطاق */
};
```

**الارتباط:** بيتجمع في `struct regmap_access_table`.

---

#### `struct regmap_access_table`

**الغرض:** جدول بيحدد مين "نعم" ومين "لا" — بيتحكم في قابلية القراءة والكتابة والـ volatility والـ precious لكل register.

```c
struct regmap_access_table {
    const struct regmap_range *yes_ranges;   /* النطاقات المسموح بيها */
    unsigned int n_yes_ranges;
    const struct regmap_range *no_ranges;    /* النطاقات الممنوعة (بتتفحص أول) */
    unsigned int n_no_ranges;
};
```

**قاعدة الـ lookup:** لو الـ register في `no_ranges` → رفض. لو في `yes_ranges` → قبول. غير كده → رفض.

**مثال واقعي:**
```c
static const struct regmap_range wr_yes[] = {
    regmap_reg_range(0x00, 0x1F),
    regmap_reg_range(0x40, 0x5F),
};
static const struct regmap_access_table wr_table = {
    .yes_ranges = wr_yes,
    .n_yes_ranges = ARRAY_SIZE(wr_yes),
};
```

---

#### `struct regmap_config` ← الـ struct الأهم

**الغرض:** التوصيف الكامل للـ register map — بيتدي للـ regmap عند الـ init ويُحدد كل شيء.

```c
struct regmap_config {
    const char *name;           /* اسم اختياري — مهم لو جهاز عنده أكتر من regmap */

    /* أبعاد الـ register */
    int reg_bits;               /* عدد bits العنوان — إلزامي */
    int reg_stride;             /* الخطوة بين عناوين صحيحة */
    int reg_shift;              /* shift موجب = downshift, سالب = upshift */
    unsigned int reg_base;      /* offset يُضاف لكل عنوان */
    int pad_bits;               /* bits فاضية بين العنوان والقيمة */
    int val_bits;               /* عدد bits القيمة — إلزامي */

    /* callbacks للـ access control */
    bool (*writeable_reg)(struct device *, unsigned int reg);
    bool (*readable_reg)(struct device *, unsigned int reg);
    bool (*volatile_reg)(struct device *, unsigned int reg);  /* لا يُخزن في cache */
    bool (*precious_reg)(struct device *, unsigned int reg);  /* لا يُقرأ تلقائيًا */
    bool (*writeable_noinc_reg)(struct device *, unsigned int reg);
    bool (*readable_noinc_reg)(struct device *, unsigned int reg);

    /* callbacks للـ I/O المخصص */
    int (*reg_read)(void *context, unsigned int reg, unsigned int *val);
    int (*reg_write)(void *context, unsigned int reg, unsigned int val);
    int (*reg_update_bits)(void *context, unsigned int reg,
                           unsigned int mask, unsigned int val);
    int (*read)(void *context, const void *reg_buf, size_t reg_size,
                void *val_buf, size_t val_size);
    int (*write)(void *context, const void *data, size_t count);
    size_t max_raw_read;
    size_t max_raw_write;

    /* locking */
    bool fast_io;               /* spinlock بدل mutex */
    bool io_port;               /* دعم IO port accessors */
    bool disable_locking;       /* تعطيل الـ locking خالص */
    regmap_lock lock;           /* custom lock callback */
    regmap_unlock unlock;       /* custom unlock callback */
    void *lock_arg;             /* argument للـ lock/unlock */

    /* حدود الـ register space */
    unsigned int max_register;
    bool max_register_is_0;     /* workaround لـ single-register maps */
    const struct regmap_access_table *wr_table;
    const struct regmap_access_table *rd_table;
    const struct regmap_access_table *volatile_table;
    const struct regmap_access_table *precious_table;
    const struct regmap_access_table *wr_noinc_table;
    const struct regmap_access_table *rd_noinc_table;

    /* cache */
    const struct reg_default *reg_defaults;
    unsigned int num_reg_defaults;
    int (*reg_default_cb)(struct device *, unsigned int reg, unsigned int *def);
    enum regcache_type cache_type;
    const void *reg_defaults_raw;
    unsigned int num_reg_defaults_raw;

    /* flags للـ wire format */
    unsigned long read_flag_mask;    /* mask يُضاف لأعلى byte عند القراءة */
    unsigned long write_flag_mask;   /* mask يُضاف لأعلى byte عند الكتابة */
    bool zero_flag_mask;
    bool use_single_read;            /* bulk read → سلسلة single reads */
    bool use_single_write;
    bool use_relaxed_mmio;           /* MMIO بدون memory barriers */
    bool can_multi_write;

    /* hardware spinlock */
    bool use_hwlock;
    bool use_raw_spinlock;
    unsigned int hwlock_id;
    unsigned int hwlock_mode;        /* HWLOCK_IRQSTATE / HWLOCK_IRQ / 0 */

    /* endianness */
    enum regmap_endian reg_format_endian;
    enum regmap_endian val_format_endian;

    /* paged registers */
    const struct regmap_range_cfg *ranges;
    unsigned int num_ranges;
};
```

**الارتباط:** بيتحوّل لـ `struct regmap` داخلي (opaque) عبر `__regmap_init()`.

---

#### `struct regmap_range_cfg`

**الغرض:** بيعرّف indirect/paged register access — لما الـ chip عنده registers أكتر من الـ address space المباشر، بيستخدم page selector.

```c
struct regmap_range_cfg {
    const char *name;

    unsigned int range_min;    /* أول عنوان في الـ virtual range */
    unsigned int range_max;    /* آخر عنوان في الـ virtual range */

    unsigned int selector_reg;   /* register بيحدد الـ page */
    unsigned int selector_mask;  /* bits الـ page selector */
    int selector_shift;          /* shift للـ page selector */

    unsigned int window_start;   /* أول register في نافذة البيانات */
    unsigned int window_len;     /* عدد registers في النافذة */
};
```

**مثال:** chip عنده virtual address 0x1000 → بيكتب page number في `selector_reg` → يقرأ/يكتب في window `[window_start, window_start+window_len)`.

---

#### `struct regmap_bus`

**الغرض:** يعرّف الـ callbacks الخاصة بالـ bus الفعلي (I2C, SPI, MMIO...) — الـ HAL للـ regmap.

```c
struct regmap_bus {
    bool fast_io;                          /* spinlock بدل mutex */
    bool free_on_exit;                     /* kfree عند exit */
    regmap_hw_write          write;        /* كتابة bulk: [reg+data في buffer واحد] */
    regmap_hw_gather_write   gather_write; /* كتابة split: reg buffer منفصل عن data */
    regmap_hw_async_write    async_write;  /* كتابة async */
    regmap_hw_reg_write      reg_write;    /* كتابة single register */
    regmap_hw_reg_noinc_write reg_noinc_write; /* كتابة متعددة لنفس العنوان */
    regmap_hw_reg_update_bits reg_update_bits; /* atomic RMW للـ volatile regs */
    regmap_hw_read           read;         /* قراءة bulk */
    regmap_hw_reg_read       reg_read;     /* قراءة single register */
    regmap_hw_reg_noinc_read reg_noinc_read; /* قراءة متعددة من نفس العنوان */
    regmap_hw_free_context   free_context; /* تحرير الـ context */
    regmap_hw_async_alloc    async_alloc;  /* allocate async descriptor */
    u8 read_flag_mask;                     /* mask يُضاف لأعلى byte عند قراءة الـ bus */
    enum regmap_endian reg_format_endian_default;
    enum regmap_endian val_format_endian_default;
    size_t max_raw_read;
    size_t max_raw_write;
};
```

**الارتباط:** بيتمرر لـ `regmap_init()` جنب `regmap_config` — الـ core بيحفظ pointer ليه داخل الـ `struct regmap` الـ opaque.

---

#### `struct regmap_sdw_mbq_cfg`

**الغرض:** إعداد إضافي خاص بـ SoundWire Multi-Byte Quantities (MBQ) — registers بيها أحجام متغيرة وممكن تتأجل.

```c
struct regmap_sdw_mbq_cfg {
    int (*mbq_size)(struct device *, unsigned int reg); /* حجم الـ register فعليًا */
    bool (*deferrable)(struct device *, unsigned int reg); /* ممكن يتأجل؟ */
    unsigned long timeout_us;  /* timeout انتظار الـ deferred transaction */
    unsigned long retry_us;    /* وقت الانتظار بين كل retry */
};
```

---

#### `struct reg_field`

**الغرض:** يعرّف bitfield داخل register — بيسهّل الوصول لـ bits معينة بدون عمل shift/mask يدويًا.

```c
struct reg_field {
    unsigned int reg;        /* عنوان الـ register */
    unsigned int lsb;        /* أقل bit في الـ field */
    unsigned int msb;        /* أعلى bit في الـ field */
    unsigned int id_size;    /* عدد الـ ports (لو متعدد) */
    unsigned int id_offset;  /* offset بين كل port والتاني */
};
```

**مثال واقعي:**
```c
/* PMIC: voltage selector في bits [5:0] من register 0x20 */
static const struct reg_field vsel_field = REG_FIELD(0x20, 0, 5);

/* استخدام */
struct regmap_field *vsel = devm_regmap_field_alloc(dev, map, vsel_field);
regmap_field_write(vsel, 0x1F); /* يضبط bits [5:0] = 0x1F تلقائيًا */
```

---

#### `struct regmap_irq_type`

**الغرض:** يعرّف كيفية configure نوع الـ IRQ (rising/falling/level) في الـ hardware register.

```c
struct regmap_irq_type {
    unsigned int type_reg_offset;      /* offset register الـ type config */
    unsigned int type_reg_mask;        /* mask في الـ type register */
    unsigned int type_rising_val;      /* القيمة لـ RISING edge */
    unsigned int type_falling_val;     /* القيمة لـ FALLING edge */
    unsigned int type_level_low_val;   /* القيمة لـ LEVEL_LOW */
    unsigned int type_level_high_val;  /* القيمة لـ LEVEL_HIGH */
    unsigned int types_supported;      /* OR من IRQ_TYPE_* */
};
```

---

#### `struct regmap_irq`

**الغرض:** يصف IRQ واحد في الـ irq_chip — موقعه ومتطلبات الـ type config.

```c
struct regmap_irq {
    unsigned int reg_offset;      /* offset داخل الـ bank (status/mask registers) */
    unsigned int mask;            /* bit mask لهذا الـ IRQ */
    struct regmap_irq_type type;  /* إعدادات الـ trigger type */
};
```

---

#### `struct regmap_irq_sub_irq_map`

**الغرض:** خريطة من main status bits إلى sub-IRQ register offsets — للـ PMICs ذات البنية الهرمية.

```c
struct regmap_irq_sub_irq_map {
    unsigned int num_regs;    /* عدد الـ sub registers */
    unsigned int *offset;     /* offsets الـ sub registers من status_base */
};
```

---

#### `struct regmap_irq_chip`

**الغرض:** يصف كامل الـ IRQ controller المبني على الـ regmap — الـ registers، الـ IRQs، والـ callbacks.

```c
struct regmap_irq_chip {
    const char *name;
    const char *domain_suffix;    /* للأجهزة ذات controllers متعددة */

    /* main status للبنى الهرمية */
    unsigned int main_status;
    unsigned int num_main_status_bits;
    const struct regmap_irq_sub_irq_map *sub_reg_offsets;
    int num_main_regs;

    /* register bases */
    unsigned int status_base;     /* base register حالة الـ interrupts */
    unsigned int mask_base;       /* base register الـ mask (1=masked) */
    unsigned int unmask_base;     /* base register الـ unmask */
    unsigned int ack_base;        /* base register لـ acknowledge */
    unsigned int wake_base;       /* base register wake-enables */
    const unsigned int *config_base; /* config registers للـ type */
    unsigned int irq_reg_stride;  /* stride لو الـ registers مش متتالية */

    /* flags سلوكية (bitfields) */
    unsigned int init_ack_masked:1;         /* ack كل masked IRQs عند init */
    unsigned int mask_unmask_non_inverted:1;
    unsigned int use_ack:1;
    unsigned int ack_invert:1;              /* cleared bits = ack */
    unsigned int clear_ack:1;
    unsigned int status_invert:1;
    unsigned int status_is_level:1;         /* XOR مع القيمة السابقة */
    unsigned int wake_invert:1;
    unsigned int type_in_mask:1;
    unsigned int clear_on_unmask:1;
    unsigned int runtime_pm:1;
    unsigned int no_status:1;               /* مفيش status register */

    int num_regs;
    const struct regmap_irq *irqs;          /* مصفوفة وصف كل IRQ */
    int num_irqs;
    int num_config_bases;
    int num_config_regs;

    /* callbacks مخصصة */
    int (*handle_pre_irq)(void *irq_drv_data);
    int (*handle_post_irq)(void *irq_drv_data);
    int (*handle_mask_sync)(int index, unsigned int mask_buf_def,
                            unsigned int mask_buf, void *irq_drv_data);
    int (*set_type_config)(unsigned int **buf, unsigned int type,
                           const struct regmap_irq *irq_data, int idx,
                           void *irq_drv_data);
    unsigned int (*get_irq_reg)(struct regmap_irq_chip_data *data,
                                unsigned int base, int index);
    void *irq_drv_data;
};
```

---

### مخططات العلاقات بين الـ Structs

```
Driver
  │
  ├── يملأ ────────────► struct regmap_config
  │                          │
  │                          ├── reg_defaults[] ──► struct reg_default[]
  │                          │
  │                          ├── wr_table ──────────► struct regmap_access_table
  │                          ├── rd_table ──────────►      │
  │                          ├── volatile_table ───►       ├── yes_ranges[] ──► struct regmap_range[]
  │                          └── ranges[] ──────────►      └── no_ranges[]  ──► struct regmap_range[]
  │                                   │
  │                                   └──► struct regmap_range_cfg[]
  │
  ├── يمرر ────────────► struct regmap_bus
  │                          │
  │                          ├── write()
  │                          ├── read()
  │                          ├── reg_write()
  │                          └── reg_read()
  │
  └── regmap_init()
          │
          ▼
     struct regmap   ◄──── opaque (في regmap.c فقط)
          │
          ├──── يستخدم ──► struct regmap_bus (بالـ pointer)
          │
          ├──── cache ──── [rbtree / flat array / maple tree]
          │
          ├──── struct regmap_field   (alloc/free منفصل)
          │          │
          │          └── يحتوي ──► struct reg_field (المعرّف من الـ driver)
          │
          └──── struct regmap_irq_chip_data  (opaque, من regmap_add_irq_chip)
                     │
                     └── يستخدم ──► struct regmap_irq_chip
                                        │
                                        └── irqs[] ──► struct regmap_irq[]
                                                           │
                                                           └── type ──► struct regmap_irq_type
```

---

### مخطط Lifecycle — إنشاء واستخدام وتدمير الـ regmap

```
┌─────────────────────────────────────────────────────────────┐
│  CREATION                                                    │
│                                                             │
│  1. Driver يعرّف struct regmap_config config = { ... };    │
│  2. Driver يستدعي devm_regmap_init_i2c(client, &config)    │
│         │                                                   │
│         ▼                                                   │
│  __regmap_init()                                            │
│     ├── kmalloc(struct regmap)                              │
│     ├── regmap_lock_init()  ← mutex أو spinlock            │
│     ├── regcache_init()     ← يملأ الـ cache من reg_defaults│
│     └── devres_add()        ← devm يربطه بـ device lifecycle│
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│  REGISTRATION (اختياري — IRQ)                               │
│                                                             │
│  regmap_add_irq_chip(map, irq, flags, base, chip, &data)   │
│     ├── request_threaded_irq()                              │
│     ├── irq_domain_add_linear()                             │
│     └── ينشئ struct regmap_irq_chip_data (opaque)          │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│  USAGE                                                      │
│                                                             │
│  regmap_write(map, REG, val)                                │
│     ├── lock(map)                                           │
│     ├── regmap_writeable() check                            │
│     ├── regcache_write() → update cache                     │
│     ├── bus->reg_write(context, REG, val)                   │
│     └── unlock(map)                                         │
│                                                             │
│  regmap_read(map, REG, &val)                                │
│     ├── lock(map)                                           │
│     ├── regmap_readable() check                             │
│     ├── regcache_read()  → hit? return cached               │
│     │     └── miss → bus->reg_read(context, REG, &val)     │
│     └── unlock(map)                                         │
│                                                             │
│  regmap_update_bits(map, REG, mask, val)                    │
│     ├── lock(map)                                           │
│     ├── regmap_read() [internal]                            │
│     ├── (old_val & ~mask) | (val & mask)                    │
│     ├── regmap_write() [internal]                           │
│     └── unlock(map)                                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│  SUSPEND / RESUME                                           │
│                                                             │
│  suspend:                                                   │
│     regcache_cache_only(map, true)   ← writes go to cache  │
│     regcache_mark_dirty(map)         ← flag: hardware stale│
│                                                             │
│  resume:                                                    │
│     regcache_cache_only(map, false)                         │
│     regcache_sync(map)               ← flush cache to HW   │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│  TEARDOWN                                                   │
│                                                             │
│  devm: يتم تلقائيًا عند device_unregister                  │
│     ├── regmap_del_irq_chip() [لو في IRQ chip]             │
│     ├── regcache_exit()                                     │
│     ├── regmap_exit()                                       │
│     └── kfree(map)                                          │
└─────────────────────────────────────────────────────────────┘
```

---

### مخططات Call Flow

#### 1. قراءة عادية `regmap_read()`

```
driver calls regmap_read(map, REG, &val)
  │
  ├── regmap_lock(map)           ← mutex أو spinlock حسب fast_io
  │
  ├── regmap_readable(map, REG)  ← check rd_table أو readable_reg()
  │     └── false → -EIO
  │
  ├── regcache_read(map, REG, &val)
  │     ├── cache hit → return cached value (no bus access)
  │     └── cache miss ─────────────────────────────────────┐
  │                                                          │
  ├── (cache miss path) ◄────────────────────────────────────┘
  │     ├── map->bus->reg_read(context, REG, &val)
  │     │       └── → I2C/SPI/MMIO read
  │     └── regcache_write(map, REG, val)  ← update cache
  │
  ├── regmap_unlock(map)
  │
  └── return 0 / error
```

#### 2. كتابة مع async `regmap_write_async()`

```
driver calls regmap_write_async(map, REG, val)
  │
  ├── regmap_lock(map)
  ├── regcache_write(map, REG, val)    ← cache updated immediately
  ├── map->bus->async_write(context, &REG, reg_size, &val, val_size, async)
  │       └── يرجع قبل اكتمال الـ DMA/transfer
  ├── regmap_unlock(map)
  │
  └── [later] regmap_async_complete(map)
                 └── wait_event() حتى يكتمل كل الـ async writes
```

#### 3. update_bits — Read-Modify-Write

```
driver calls regmap_update_bits(map, REG, mask, val)
  │
  └── regmap_update_bits_base(map, REG, mask, val, NULL, false, false)
          │
          ├── regmap_lock(map)
          │
          ├── _regmap_read(map, REG, &old_val)
          │       └── [cache أو bus read]
          │
          ├── new_val = (old_val & ~mask) | (val & mask)
          │
          ├── new_val == old_val && !force → skip write (تحسين أداء)
          │
          ├── _regmap_write(map, REG, new_val)
          │       ├── regcache_write()
          │       └── bus->reg_write() أو bus->reg_update_bits()
          │
          ├── regmap_unlock(map)
          └── return 0 / -error
```

#### 4. IRQ handler flow

```
Hardware IRQ fires
  │
  └── regmap_irq_thread() [threaded IRQ handler]
          │
          ├── handle_pre_irq(irq_drv_data)   [optional]
          │
          ├── [hierarchical chips] read main_status register
          │       └── iterate sub-irq banks
          │
          ├── [standard] loop over num_regs:
          │       regmap_read(map, status_base + i*stride, &status)
          │
          ├── for each set bit in status:
          │       ├── generic_handle_irq(virq)
          │       └── [ack_base] regmap_write(map, ack_base + offset, mask)
          │
          ├── handle_post_irq(irq_drv_data)   [optional]
          │
          └── return IRQ_HANDLED
```

#### 5. paged register access (indirect)

```
driver calls regmap_write(map, virtual_addr, val)
  │
  ├── virtual_addr belongs to a range_cfg?
  │     └── YES:
  │           ├── page = (virtual_addr - range_min) / window_len
  │           ├── regmap_write(map, selector_reg,
  │           │               (page << selector_shift) & selector_mask)
  │           └── phys_addr = window_start + (virtual_addr - range_min) % window_len
  │
  └── regmap_write(map, phys_addr, val)  ← الكتابة الفعلية
```

---

### استراتيجية الـ Locking

#### أنواع الـ Locks المستخدمة

| الحالة | نوع الـ Lock | متى يُختار |
|---|---|---|
| `fast_io = false` (default) | `mutex` | أغلب الأجهزة — يسمح بالـ sleep |
| `fast_io = true` | `spinlock` | MMIO أو أي bus سريع — لا ينام |
| `disable_locking = true` | لا يوجد | driver يتحكم بالـ locking خارجيًا |
| custom `lock/unlock` | أي آلية | custom locking logic |
| `use_hwlock = true` | hardware spinlock | shared resource بين processors |
| `use_raw_spinlock = true` | raw spinlock | real-time contexts |

#### ما الذي تحميه الـ Locks؟

```
regmap->lock
  │
  ├── يحمي:
  │     ├── الـ cache (قراءة/كتابة)
  │     ├── تسلسل الـ bus transactions
  │     ├── الـ async_list (pending async writes)
  │     └── الـ cache_bypass / cache_only flags
  │
  └── النطاق: من lock() حتى unlock() في كل regmap_read/write/update
```

#### ترتيب الـ Locks (Lock Ordering) — مهم لتجنب Deadlock

```
قاعدة صارمة:
  regmap->lock   يجب أن يُأخذ قبل أي lock داخلي للـ bus

مثال I2C:
  regmap->lock
      └── i2c_lock_adapter()  [داخل i2c driver]
              └── الـ transfer

مثال IRQ chip:
  irq_desc->lock  [kernel IRQ layer]
      └── regmap_irq handler
              └── regmap->lock
                      └── bus access

تحذير: لا تستدعي regmap من داخل spinlock context إلا لو
       fast_io=true أو disable_locking=true
```

#### الـ Async Locking

```
regmap_write_async():
  lock(map)
    └── bus->async_write() ← يبدأ transfer ويرجع فورًا
  unlock(map)

regmap_async_complete():
  ← يستنى كل الـ async transfers تخلص
  ← يستخدم completion أو wait_event داخليًا
  ← يجب استدعاؤه قبل أي sync read بعد async write
```

#### الـ Cache Locking وسيناريو suspend/resume

```
صح:
  regcache_cache_only(map, true)   ← no bus access, writes buffered in cache
  [device suspended]
  regcache_cache_only(map, false)  ← bus access restored
  regcache_sync(map)               ← flush all dirty cache entries to HW

خطأ (يسبب bus access على device موقوف):
  regmap_write(map, REG, val)  ← مباشرة بعد suspend بدون cache_only
```
## Phase 4: شرح الـ Functions

---

### Cheatsheet — كل الـ APIs بنظرة واحدة

#### Registration & Initialization

| Function / Macro | الغرض |
|---|---|
| `regmap_init(dev, bus, ctx, cfg)` | ينشئ regmap عام (non-devm) |
| `regmap_init_i2c(i2c, cfg)` | regmap فوق I2C |
| `regmap_init_spi(dev, cfg)` | regmap فوق SPI |
| `regmap_init_mmio(dev, regs, cfg)` | regmap فوق MMIO |
| `regmap_init_mmio_clk(dev, clk, regs, cfg)` | MMIO + clock gate |
| `regmap_init_spmi_base/ext(dev, cfg)` | SPMI Base/Ext address spaces |
| `regmap_init_sdw(sdw, cfg)` | SoundWire |
| `regmap_init_sdw_mbq(sdw, cfg)` | SoundWire Multi-Byte Quantity |
| `regmap_init_ac97(ac97, cfg)` | AC'97 |
| `regmap_init_slimbus(slim, cfg)` | SLIMbus |
| `regmap_init_w1(dev, cfg)` | 1-Wire |
| `regmap_init_mdio(mdio, cfg)` | MDIO (Ethernet PHY) |
| `regmap_init_fsi(fsi, cfg)` | FSI |
| `devm_regmap_init_*(...)` | نفس كل ما فوق لكن managed (auto-free) |
| `regmap_attach_dev(dev, map, cfg)` | يربط regmap موجود بـ device |
| `regmap_exit(map)` | يحرر الـ regmap يدويًا |

#### Core Read / Write

| Function | الغرض |
|---|---|
| `regmap_write(map, reg, val)` | كتابة register واحد |
| `regmap_write_async(map, reg, val)` | كتابة async (non-blocking) |
| `regmap_raw_write(map, reg, val, len)` | كتابة raw bytes بدون format |
| `regmap_noinc_write(map, reg, val, len)` | كتابة متعددة على نفس العنوان |
| `regmap_bulk_write(map, reg, val, count)` | كتابة consecutive regs |
| `regmap_multi_reg_write(map, seqs, n)` | كتابة sequence مع delays |
| `regmap_multi_reg_write_bypassed(map, seqs, n)` | نفسها لكن bypass cache |
| `regmap_raw_write_async(map, reg, val, len)` | raw async write |
| `regmap_read(map, reg, val)` | قراءة register واحد |
| `regmap_read_bypassed(map, reg, val)` | قراءة bypass cache |
| `regmap_raw_read(map, reg, val, len)` | قراءة raw bytes |
| `regmap_noinc_read(map, reg, val, len)` | قراءة متعددة من نفس العنوان |
| `regmap_bulk_read(map, reg, val, count)` | قراءة consecutive regs |
| `regmap_multi_reg_read(map, regs, val, n)` | قراءة عناوين متفرقة |

#### Bit Manipulation

| Function | الغرض |
|---|---|
| `regmap_update_bits(map, reg, mask, val)` | RMW على أي bits |
| `regmap_update_bits_async(...)` | نفسها async |
| `regmap_update_bits_check(...)` | RMW + report if changed |
| `regmap_update_bits_base(...)` | الـ base function الحقيقية |
| `regmap_write_bits(map, reg, mask, val)` | force-write (يتجاهل التغيير) |
| `regmap_set_bits(map, reg, bits)` | set bits بدون clear غيرها |
| `regmap_clear_bits(map, reg, bits)` | clear bits بدون set غيرها |
| `regmap_assign_bits(map, reg, bits, val)` | set أو clear بناءً على bool |
| `regmap_test_bits(map, reg, bits)` | يتحقق إذا bits كلها set |

#### Register Fields

| Function | الغرض |
|---|---|
| `regmap_field_alloc(map, field)` | ينشئ regmap_field |
| `regmap_field_free(field)` | يحرره |
| `devm_regmap_field_alloc(dev, map, field)` | managed version |
| `regmap_field_bulk_alloc(map, fields, descs, n)` | ينشئ array of fields |
| `regmap_field_read(field, val)` | يقرأ قيمة field |
| `regmap_field_write(field, val)` | يكتب قيمة field |
| `regmap_field_update_bits(field, mask, val)` | RMW داخل field |
| `regmap_field_set_bits / clear_bits` | bit ops على field |
| `regmap_field_test_bits(field, bits)` | يتحقق من bits |
| `regmap_fields_read(field, id, val)` | قراءة instance محددة من port field |
| `regmap_fields_write(field, id, val)` | كتابة instance محددة |

#### Cache (regcache)

| Function | الغرض |
|---|---|
| `regcache_sync(map)` | يكتب كل dirty cache entries للـ hardware |
| `regcache_sync_region(map, min, max)` | sync نطاق محدد |
| `regcache_drop_region(map, min, max)` | يمسح cache entries بدون write |
| `regcache_cache_only(map, bool)` | يمنع أي I/O حقيقي |
| `regcache_cache_bypass(map, bool)` | يتجاهل الـ cache تمامًا |
| `regcache_mark_dirty(map)` | يعلّم كل الـ cache كـ dirty |
| `regcache_reg_cached(map, reg)` | يتحقق إذا register موجود في cache |
| `regcache_sort_defaults(defaults, n)` | يرتب reg_defaults array |

#### IRQ Chip

| Function | الغرض |
|---|---|
| `regmap_add_irq_chip(map, irq, flags, base, chip, data)` | يسجّل IRQ controller |
| `regmap_add_irq_chip_fwnode(...)` | نفسها مع fwnode |
| `regmap_del_irq_chip(irq, data)` | يحذف IRQ controller |
| `devm_regmap_add_irq_chip(...)` | managed version |
| `regmap_irq_get_virq(data, irq)` | يجيب الـ virtual IRQ number |
| `regmap_irq_get_domain(data)` | يجيب الـ irq_domain |
| `regmap_irq_chip_get_base(data)` | يجيب irq_base |
| `regmap_irq_get_irq_reg_linear(data, base, idx)` | linear address mapping helper |
| `regmap_irq_set_type_config_simple(...)` | helper لـ IRQ type config |

#### Poll Macros

| Macro | الغرض |
|---|---|
| `regmap_read_poll_timeout(map, addr, val, cond, sleep, timeout)` | poll sleepable |
| `regmap_read_poll_timeout_atomic(...)` | poll atomic (udelay) |
| `regmap_field_read_poll_timeout(field, val, cond, sleep, timeout)` | poll على field |

#### Utility / Info

| Function | الغرض |
|---|---|
| `regmap_get_val_bytes(map)` | حجم القيمة بالـ bytes |
| `regmap_get_max_register(map)` | آخر عنوان register |
| `regmap_get_reg_stride(map)` | stride بين العناوين |
| `regmap_might_sleep(map)` | هل العمليات ممكن تنام؟ |
| `regmap_async_complete(map)` | ينتظر اكتمال كل async writes |
| `regmap_can_raw_write(map)` | هل raw write مدعومة؟ |
| `regmap_get_raw_read_max(map)` | أقصى حجم raw read |
| `regmap_get_raw_write_max(map)` | أقصى حجم raw write |
| `regmap_check_range_table(map, reg, table)` | يتحقق إذا reg في access table |
| `regmap_reg_in_range(reg, range)` | يتحقق إذا reg في range واحد |
| `regmap_reg_in_ranges(reg, ranges, n)` | يتحقق عبر array of ranges |
| `regmap_register_patch(map, seqs, n)` | يطبّق patch sequence دائمًا عند init |
| `regmap_parse_val(map, buf, val)` | يحوّل raw buffer لـ unsigned int |
| `dev_get_regmap(dev, name)` | يجيب regmap مربوط بـ device |
| `regmap_get_device(map)` | يجيب الـ device من الـ map |
| `regmap_mmio_attach_clk(map, clk)` | يربط clock بـ MMIO regmap |
| `regmap_mmio_detach_clk(map)` | يفصل الـ clock |
| `regmap_ac97_default_volatile(dev, reg)` | volatile check لـ AC'97 |
| `regmap_reinit_cache(map, cfg)` | يعيد بناء الـ cache من config جديدة |

---

### Group 1: Initialization & Registration

الـ group دي مسؤولة عن إنشاء الـ `struct regmap` الـ opaque object اللي بيمثّل الوصول لـ hardware registers. الـ kernel بيدعم عشرات الـ buses، وكل واحدة عندها `__regmap_init_XXX` function بتملّي الـ `struct regmap_bus` بالـ callbacks الخاصة بيها.

كل الـ macros الـ public بتستخدم الـ wrapper الـ `__regmap_lockdep_wrapper` اللي بيضيف `lock_class_key` فريد لكل call site لتفادي false positives في `lockdep`.

#### `regmap_init()` — الـ Generic Initializer

```c
#define regmap_init(dev, bus, bus_context, config)    \
    __regmap_lockdep_wrapper(__regmap_init, #config,  \
                             dev, bus, bus_context, config)
```

الـ macro دي هي entry point لكل bus-specific initializer. بتستدعي `__regmap_init()` اللي بتعمل:
- `kzalloc` للـ `struct regmap`
- تحديد نوع الـ locking (mutex أو spinlock بناءً على `fast_io` أو `disable_locking`)
- تسجيل الـ `lock_class_key` في `lockdep`
- بناء الـ cache بناءً على `config->cache_type`
- ربط الـ regmap بالـ device عبر `devres`

**Parameters:**
- `dev` — الـ `struct device` صاحب الـ hardware
- `bus` — `struct regmap_bus` بالـ callbacks (read/write/async)
- `bus_context` — بيتاعت الـ bus (مثلاً pointer للـ `i2c_client`)
- `config` — `struct regmap_config` بكل الإعدادات

**Return:** pointer صالح أو `ERR_PTR()` على فشل الـ `kzalloc` أو config validation

**Key details:**
- لازم `reg_bits` و`val_bits` يكونوا set وإلا بترجع `-EINVAL`
- الـ `lock_class_key` بيتعمل كـ `static` variable في call site — معناه كل استدعاء في كود مختلف بيحصل على key مختلف حتى لو نفس الـ config struct
- **لا تستدعيها مباشرة من driver** — استخدم `regmap_init_i2c` أو `regmap_init_mmio` إلخ

---

#### `regmap_init_mmio()` / `regmap_init_mmio_clk()`

```c
#define regmap_init_mmio(dev, regs, config)    \
    regmap_init_mmio_clk(dev, NULL, regs, config)

#define regmap_init_mmio_clk(dev, clk_id, regs, config)    \
    __regmap_lockdep_wrapper(__regmap_init_mmio_clk, #config, \
                             dev, clk_id, regs, config)
```

الأشيع استخدامًا في platform drivers وـ SoC drivers. الـ `regs` هو الـ `void __iomem *` الناتج من `ioremap`. الـ `clk_id` لو مش NULL بيعمل `clk_get` ويمسك الـ clock عند كل عملية I/O.

**بيعمل `fast_io = true` تلقائيًا** لأن MMIO access سريع ومش محتاج mutex.

**Parameters:**
- `dev` — الـ platform device
- `clk_id` — اسم الـ clock في `clk_get`، أو `NULL`
- `regs` — base address بعد الـ `ioremap`
- `config` — الـ regmap config

---

#### `regmap_init_i2c()` / `devm_regmap_init_i2c()`

```c
#define regmap_init_i2c(i2c, config)    \
    __regmap_lockdep_wrapper(__regmap_init_i2c, #config, i2c, config)
```

بتستخدم I2C transfers تحت الغطاء. الـ `i2c_client` بيتحول لـ `bus_context`. الـ bus callbacks بتستخدم `i2c_master_send/recv`.

**الـ `devm_` version:** بتضيف `devres` action تعمل `regmap_exit()` تلقائيًا عند `device_unregister`. أنصح بيها دايمًا في الـ drivers الحديثة.

---

#### `regmap_attach_dev()`

```c
int regmap_attach_dev(struct device *dev, struct regmap *map,
                      const struct regmap_config *config);
```

لو الـ regmap اتعمل بدون device (مثلاً في bus init code)، الـ function دي بتربطه بالـ device لاحقًا. بتضيفه لـ `dev->p->regmap_list` حتى `dev_get_regmap()` تلاقيه.

**Return:** `0` أو `-EINVAL` لو `map` هو `NULL`.

---

#### `regmap_exit()`

```c
void regmap_exit(struct regmap *map);
```

بتحرر كل الموارد: cache، lock، async buffers، وأي context مربوط بالـ bus. **لا تستدعيها لو استخدمت `devm_regmap_init_*`** — الـ devres هيعملها.

---

### Group 2: Core Read / Write Operations

دي لب الـ API. كل function بتمر بـ validation (range check، writeable check)، ثم cache lookup أو hardware I/O، وأخيرًا تحديث الـ cache.

#### `regmap_write()`

```c
int regmap_write(struct regmap *map, unsigned int reg, unsigned int val);
```

أبسط write operation. بتعمل:
1. التحقق من `reg <= max_register` وأن الـ register writeable
2. أخذ الـ regmap lock
3. لو الـ cache ممكّن: مقارنة بالقيمة المحجوزة — لو نفسها ومفيش `force`، ترجع بدون I/O
4. استدعاء `bus->reg_write` أو `bus->write`
5. تحديث الـ cache
6. تحرير الـ lock

**Parameters:**
- `map` — الـ regmap handle
- `reg` — عنوان الـ register
- `val` — القيمة المراد كتابتها

**Return:** `0` على النجاح، أو error code من الـ bus driver

**Locking:** بيمسك الـ regmap internal lock (mutex أو spinlock حسب الـ config). **لا تستدعيها وأنت حامل الـ lock نفسه.**

---

#### `regmap_write_async()`

```c
int regmap_write_async(struct regmap *map, unsigned int reg, unsigned int val);
```

نفس `regmap_write` لكن بتستخدم `bus->async_write`. الـ write ممكن ما يكملش وقت رجوع الـ function — لازم تستدعي `regmap_async_complete()` قبل ما تعتمد على الـ hardware.

**متى تستخدمها:** في الـ driver init لتسريع كتابة عدد كبير من الـ registers.

---

#### `regmap_raw_write()`

```c
int regmap_raw_write(struct regmap *map, unsigned int reg,
                     const void *val, size_t val_len);
```

بتكتب raw bytes مباشرة — بدون الـ endianness conversion أو الـ format transformation اللي بتعملها `regmap_write`. الـ `val` لازم يكون في الـ byte order المتوقع من الـ hardware.

**مهم:** الـ cache **لا يتحدّث** عند raw write لأن الـ regmap مش عارف تفسير الـ bytes. استخدمها بحذر لو الـ cache ممكّن.

**Parameters:**
- `val` — pointer لـ buffer
- `val_len` — حجم الـ buffer بالـ bytes

---

#### `regmap_noinc_write()`

```c
int regmap_noinc_write(struct regmap *map, unsigned int reg,
                       const void *val, size_t val_len);
```

بتكتب عدة قيم على **نفس عنوان الـ register** بدون increment. مفيدة للـ FIFO registers حيث الـ hardware بيقبل stream من البيانات على عنوان ثابت.

**Requires:** `writeable_noinc_reg` callback أو `wr_noinc_table` في الـ config يوافق على الـ register.

---

#### `regmap_bulk_write()`

```c
int regmap_bulk_write(struct regmap *map, unsigned int reg,
                      const void *val, size_t val_count);
```

بتكتب `val_count` من الـ register values بدءًا من `reg`. الـ `val` هو array من الـ native-endian values — الـ regmap هيعمل الـ formatting. لو الـ bus يدعم bulk write، بتبعت كل الـ data في transaction واحدة.

**الفرق عن `regmap_raw_write`:** هنا الـ regmap بيفهم كل قيمة ويعملها format، هناك بيبعت الـ bytes as-is.

---

#### `regmap_multi_reg_write()`

```c
int regmap_multi_reg_write(struct regmap *map,
                           const struct reg_sequence *regs,
                           int num_regs);
```

بتكتب sequence من (register, value, delay_us) tuples. بعد كل write بتعمل `usleep_range(delay_us, delay_us + 10)` لو الـ delay مش صفر. **مفيدة جدًا لـ power-on sequences في الـ audio codecs.**

```c
/* مثال: power-up sequence لـ audio codec */
static const struct reg_sequence wm8960_init[] = {
    REG_SEQ0(WM8960_RESET, 0x000),
    REG_SEQ(WM8960_POWER1, 0x0C0, 100000), /* wait 100ms */
    REG_SEQ0(WM8960_POWER2, 0x1E0),
};
regmap_multi_reg_write(map, wm8960_init, ARRAY_SIZE(wm8960_init));
```

---

#### `regmap_read()`

```c
int regmap_read(struct regmap *map, unsigned int reg, unsigned int *val);
```

القراءة الأساسية. الـ flow:

```
regmap_read()
  → check reg valid & readable
  → acquire lock
  → if cache_only: read from cache only
  → else if !volatile: check cache first
      → cache hit? return cached value
      → cache miss: call bus->reg_read
  → volatile: always call bus->reg_read
  → update cache (if !volatile)
  → release lock
```

**Return:** `0` ويكتب القيمة في `*val`، أو error code.

**Locking:** نفس الـ regmap lock.

---

#### `regmap_read_bypassed()`

```c
int regmap_read_bypassed(struct regmap *map, unsigned int reg,
                         unsigned int *val);
```

بتقرأ من الـ hardware مباشرة حتى لو الـ `cache_only` mode شغّال. مفيدة في debug أو عند الحاجة لقراءة حالة الـ hardware الفعلية في وسط suspend sequence.

---

### Group 3: Bit Manipulation (RMW)

#### `regmap_update_bits_base()` — الـ Core RMW Function

```c
int regmap_update_bits_base(struct regmap *map, unsigned int reg,
                            unsigned int mask, unsigned int val,
                            bool *change, bool async, bool force);
```

دي الـ function الحقيقية اللي كل الـ `regmap_update_bits_*` wrappers بتستدعيها.

**ما بتعمله:**
1. بتقرأ القيمة الحالية بـ `regmap_read()`
2. بتحسب الـ new value: `new = (old & ~mask) | (val & mask)`
3. لو `old == new` وـ `force == false`: ترجع بدون write
4. لو `change != NULL`: تضبطه على `true/false`
5. تكتب الـ new value

**Parameters:**
- `mask` — الـ bits المراد تعديلها (1 = تعدّل)
- `val` — القيم الجديدة للـ bits الـ masked فقط
- `change` — pointer لـ bool، أو `NULL`
- `async` — `true` لاستخدام async write
- `force` — `true` لكتابة حتى لو القيمة ما اتغيرتش

```
/* مثال: تشغيل bit 3 في register 0x10 */
regmap_set_bits(map, 0x10, BIT(3));

/* مثال: تعديل bits 7:4 */
regmap_update_bits(map, 0x10, GENMASK(7, 4), 0x50);
```

---

#### `regmap_update_bits()` (inline wrapper)

```c
static inline int regmap_update_bits(struct regmap *map, unsigned int reg,
                                     unsigned int mask, unsigned int val)
{
    return regmap_update_bits_base(map, reg, mask, val, NULL, false, false);
}
```

أبسط استخدام لـ RMW. الـ change tracking معطّل، sync، ولا force.

---

#### `regmap_write_bits()` (force RMW)

```c
static inline int regmap_write_bits(struct regmap *map, unsigned int reg,
                                    unsigned int mask, unsigned int val)
{
    return regmap_update_bits_base(map, reg, mask, val, NULL, false, true);
}
```

نفس `regmap_update_bits` لكن بتكتب دايمًا حتى لو القيمة ما اتغيرتش (`force = true`). مفيدة للـ write-only registers أو لما الـ cache out of sync.

---

#### `regmap_set_bits()` / `regmap_clear_bits()` / `regmap_assign_bits()`

```c
static inline int regmap_set_bits(struct regmap *map,
                                  unsigned int reg, unsigned int bits)
{
    return regmap_update_bits_base(map, reg, bits, bits, NULL, false, false);
}

static inline int regmap_clear_bits(struct regmap *map,
                                    unsigned int reg, unsigned int bits)
{
    return regmap_update_bits_base(map, reg, bits, 0, NULL, false, false);
}

static inline int regmap_assign_bits(struct regmap *map, unsigned int reg,
                                     unsigned int bits, bool value)
{
    if (value)
        return regmap_set_bits(map, reg, bits);
    else
        return regmap_clear_bits(map, reg, bits);
}
```

**الـ `assign_bits`** مفيدة جدًا في الـ driver code لما عندك `bool` من الـ devicetree أو من الـ user:

```c
/* enable أو disable interrupt بناءً على DT property */
regmap_assign_bits(map, REG_IRQ_CTRL, BIT(IRQ_EN), irq_enabled);
```

---

#### `regmap_test_bits()`

```c
int regmap_test_bits(struct regmap *map, unsigned int reg, unsigned int bits);
```

بتقرأ الـ register وبترجع `1` لو كل الـ `bits` set، `0` لو لا، أو negative error.

```c
/* انتظر حتى ready bit يتشال */
ret = regmap_read_poll_timeout(map, REG_STATUS, val,
                               !(val & BIT(STATUS_BUSY)),
                               1000, 100000);
```

---

### Group 4: Register Fields (regmap_field)

الـ **`struct regmap_field`** بيمثّل field داخل register محدد — بـ LSB، MSB، وعنوان الـ register. بيسهّل التعامل مع الـ hardware اللي فيه registers مقسّمة لـ fields.

#### `regmap_field_alloc()` / `devm_regmap_field_alloc()`

```c
struct regmap_field *regmap_field_alloc(struct regmap *regmap,
                                        struct reg_field reg_field);
```

بتعمل `kzalloc` لـ `struct regmap_field` وبتحسب الـ mask من الـ `lsb` و`msb`.

```c
/* مثال: field هو bits 5:3 في register 0x20 */
static const struct reg_field clk_div_field = REG_FIELD(0x20, 3, 5);

struct regmap_field *field = devm_regmap_field_alloc(dev, map, clk_div_field);
```

**Return:** pointer أو `ERR_PTR(-ENOMEM)`.

---

#### `regmap_field_bulk_alloc()`

```c
int regmap_field_bulk_alloc(struct regmap *regmap,
                             struct regmap_field **rm_field,
                             const struct reg_field *reg_field,
                             int num_fields);
```

بتعمل array من الـ `regmap_field` pointers بـ allocation واحدة. الـ `rm_field` هو pointer لـ array من الـ pointers.

```c
struct regmap_field *fields[NUM_FIELDS];
regmap_field_bulk_alloc(map, fields, field_descs, NUM_FIELDS);
```

---

#### `regmap_field_read()`

```c
int regmap_field_read(struct regmap_field *field, unsigned int *val);
```

بتقرأ الـ register، بتعمل mask وـ shift، وبترجع قيمة الـ field فقط في `*val`.

**مثال:** لو الـ field هو bits 5:3 و register قيمته `0b00111000`، `*val` هتكون `0b111 = 7`.

---

#### `regmap_field_write()`

```c
static inline int regmap_field_write(struct regmap_field *field,
                                     unsigned int val)
{
    return regmap_field_update_bits_base(field, ~0, val, NULL, false, false);
}
```

بتعمل RMW: بتحافظ على الـ bits خارج الـ field وبتكتب الـ `val` في مكان الـ field.

---

#### `regmap_field_update_bits_base()` — الـ Core Field RMW

```c
int regmap_field_update_bits_base(struct regmap_field *field,
                                  unsigned int mask, unsigned int val,
                                  bool *change, bool async, bool force);
```

نفس `regmap_update_bits_base` لكن بتعمل shift للـ mask والـ val حسب الـ field position قبل الكتابة.

**Pseudocode:**

```
shifted_mask = (mask & field->mask_raw) << field->shift
shifted_val  = (val  << field->shift) & shifted_mask
regmap_update_bits_base(field->regmap, field->reg,
                        shifted_mask, shifted_val, change, async, force)
```

---

#### Port Fields — `regmap_fields_read()` / `regmap_fields_write()`

```c
int regmap_fields_read(struct regmap_field *field,
                       unsigned int id, unsigned int *val);
```

للـ hardware اللي عنده نفس الـ field بس في registers متعددة (ports). الـ `id_size` و`id_offset` في `struct reg_field` بيحددوا الـ stride.

```c
/* مثال: GPIO bank — كل port على عنوان مختلف */
static const struct reg_field gpio_dir_field =
    REG_FIELD_ID(GPIO_DIR_BASE, 0, 7, NUM_PORTS, GPIO_PORT_STRIDE);

/* قراءة direction لـ port 3 */
regmap_fields_read(gpio_dir_field, 3, &dir_val);
```

---

### Group 5: Cache Management (regcache)

الـ **regcache** بيخزّن آخر قيمة معروفة لكل register. بيقلل الـ I/O ويدعم suspend/resume.

#### `regcache_sync()`

```c
int regcache_sync(struct regmap *map);
```

بتكتب كل الـ cache entries الـ dirty للـ hardware. الغالب بتتستدعى في `resume` أو بعد `regcache_cache_only()` period.

```c
/* في driver->resume */
regcache_cache_only(map, false);
ret = regcache_sync(map);
```

**Locking:** بتمسك الـ regmap lock طول فترة الـ sync.

---

#### `regcache_sync_region()`

```c
int regcache_sync_region(struct regmap *map,
                         unsigned int min, unsigned int max);
```

sync نطاق محدد فقط. مفيدة لو بس جزء من الـ registers اتغير.

---

#### `regcache_drop_region()`

```c
int regcache_drop_region(struct regmap *map,
                         unsigned int min, unsigned int max);
```

بتمسح الـ cache entries في النطاق المحدد **بدون كتابة للـ hardware**. الـ entries هتتعامل معها كـ "unknown" في القراءة الجاية. مفيدة بعد hardware reset.

---

#### `regcache_cache_only()`

```c
void regcache_cache_only(struct regmap *map, bool enable);
```

لما `enable = true`: كل الـ writes بتروح للـ cache بس (مش للـ hardware) — وكل الـ reads بتيجي من الـ cache. مفيدة خلال `suspend` لتجنب I/O على device مش شغّال.

---

#### `regcache_cache_bypass()`

```c
void regcache_cache_bypass(struct regmap *map, bool enable);
```

العكس تمامًا: `enable = true` يعني تجاهل الـ cache خالص وإجبار كل I/O على الـ hardware. مفيدة للـ debugging أو عند الحاجة لـ accurate hardware reads.

---

#### `regcache_mark_dirty()`

```c
void regcache_mark_dirty(struct regmap *map);
```

بتعلّم كل الـ cache entries كـ dirty بدون تغيير القيم. بعد hardware reset الـ cache لا يزال يحتوي على القيم الصحيحة لكن الـ hardware رجع لـ defaults — استدعاء `mark_dirty` ثم `regcache_sync` بتعيد تحميل الـ hardware.

**Pattern شائع:**

```c
/* عند hardware reset */
regcache_mark_dirty(map);
regcache_sync(map);
```

---

#### `regcache_sort_defaults()`

```c
void regcache_sort_defaults(struct reg_default *defaults,
                             unsigned int ndefaults);
```

بترتّب الـ `reg_default` array تصاعديًا بالـ address. بعض الـ cache backends (خاصة rbtree) بتحتاج الـ defaults مرتّبة لـ O(n) initialization بدل O(n log n).

---

### Group 6: IRQ Chip (regmap_irq)

الـ **regmap_irq subsystem** بيوفّر `irq_chip` جاهز للـ devices اللي عندها interrupt registers منظّمة كـ status/mask/ack banks.

#### `regmap_add_irq_chip()`

```c
int regmap_add_irq_chip(struct regmap *map, int irq, int irq_flags,
                        int irq_base, const struct regmap_irq_chip *chip,
                        struct regmap_irq_chip_data **data);
```

بتسجّل الـ irq_chip ومعاها:
- Allocate وتهيئة `struct regmap_irq_chip_data`
- إنشاء `irq_domain` (linear أو legacy بناءً على `irq_base`)
- Request الـ parent IRQ وتسجيل الـ `regmap_irq_handler` كـ threaded IRQ handler
- Mask كل الـ interrupts أولًا (init_ack_masked لو مطلوب)

**Parameters:**
- `irq` — رقم الـ parent IRQ (من SoC GPIO أو PMIC)
- `irq_flags` — `IRQF_*` flags (عادةً `IRQF_ONESHOT`)
- `irq_base` — بداية الـ virtual IRQ numbers، أو `0` لـ dynamic allocation
- `chip` — الـ descriptor struct
- `data` — output: handle للـ IRQ controller

**Return:** `0` أو `-ENOMEM / -EINVAL / -EBUSY`.

---

#### `regmap_add_irq_chip_fwnode()`

```c
int regmap_add_irq_chip_fwnode(struct fwnode_handle *fwnode,
                               struct regmap *map, int irq,
                               int irq_flags, int irq_base,
                               const struct regmap_irq_chip *chip,
                               struct regmap_irq_chip_data **data);
```

نفس `regmap_add_irq_chip` لكن بتربط الـ `irq_domain` بـ `fwnode` بدل الـ device node. مفيدة لـ MFD devices اللي عندها child device nodes في الـ DT.

---

#### `regmap_del_irq_chip()` / `devm_regmap_del_irq_chip()`

```c
void regmap_del_irq_chip(int irq, struct regmap_irq_chip_data *data);
```

بتعمل عكس `regmap_add_irq_chip`: free the irq_domain، remove threaded handler، حرر الـ memory.

الـ **`devm_regmap_add_irq_chip()`** يضيف devres action تستدعي `regmap_del_irq_chip()` تلقائيًا.

---

#### `regmap_irq_get_virq()`

```c
int regmap_irq_get_virq(struct regmap_irq_chip_data *data, int irq);
```

بترجع الـ virtual IRQ number للـ sub-IRQ رقم `irq`. الـ driver بيحتاجها لـ `request_irq` على IRQ محدد.

```c
int virq = regmap_irq_get_virq(irq_data, WM5110_IRQ_DSP1_RAM_RDY);
ret = request_threaded_irq(virq, NULL, wm5110_dsp_irq, ...);
```

---

#### `regmap_irq_get_irq_reg_linear()`

```c
unsigned int regmap_irq_get_irq_reg_linear(struct regmap_irq_chip_data *data,
                                           unsigned int base, int index);
```

الـ default implementation للـ `get_irq_reg` callback. بترجع `base + index * irq_reg_stride`. لو الـ hardware layout أعقد، الـ driver يوفّر callback خاص.

---

#### `regmap_irq_set_type_config_simple()`

```c
int regmap_irq_set_type_config_simple(unsigned int **buf, unsigned int type,
                                      const struct regmap_irq *irq_data,
                                      int idx, void *irq_drv_data);
```

الـ default `set_type_config` implementation. بتكتب القيمة المناسبة (`type_rising_val`, `type_falling_val`, إلخ) في الـ config buffer بناءً على `IRQ_TYPE_*` المطلوب.

---

### Group 7: Poll Macros

#### `regmap_read_poll_timeout()`

```c
#define regmap_read_poll_timeout(map, addr, val, cond, sleep_us, timeout_us)
```

بتعمل loop تقرأ فيها `addr` وتنام `sleep_us` (بـ `usleep_range`) حتى `cond` يتحقق أو `timeout_us` ينتهي.

```c
/* انتظر حتى PLL locked */
ret = regmap_read_poll_timeout(map, PLL_STATUS, val,
                               val & PLL_LOCKED, 100, 50000);
if (ret == -ETIMEDOUT)
    dev_err(dev, "PLL failed to lock\n");
```

**لا تستخدمها من atomic context** — بتنام.

---

#### `regmap_read_poll_timeout_atomic()`

```c
#define regmap_read_poll_timeout_atomic(map, addr, val, cond, delay_us, timeout_us)
```

نسخة atomic: بتستخدم `udelay` بدل `usleep_range`. **تتطلب MMIO regmap بدون cache** (أو flat cache) لأن regmap عمومًا مش atomic-safe.

---

#### `regmap_field_read_poll_timeout()`

```c
#define regmap_field_read_poll_timeout(field, val, cond, sleep_us, timeout_us)
```

نفس `regmap_read_poll_timeout` لكن بتقرأ field بدل register كامل. بتستخدم `regmap_field_read()` داخليًا.

---

### Group 8: Utility & Info Functions

#### `regmap_get_val_bytes()` / `regmap_get_max_register()` / `regmap_get_reg_stride()`

```c
int regmap_get_val_bytes(struct regmap *map);
int regmap_get_max_register(struct regmap *map);
int regmap_get_reg_stride(struct regmap *map);
```

accessors بسيطة لـ configuration parameters. مفيدة في الـ code العام اللي بيتعامل مع regmaps مختلفة (مثلاً debugfs أو generic dump utilities).

---

#### `regmap_might_sleep()`

```c
bool regmap_might_sleep(struct regmap *map);
```

بترجع `true` لو الـ regmap operations ممكن تعمل sleep (يعني الـ lock هو mutex مش spinlock). مفيدة لـ code validation — لو الـ caller context مش يقدر ينام والـ map ممكن ينام، ده error.

---

#### `regmap_async_complete()`

```c
int regmap_async_complete(struct regmap *map);
```

بتنتظر اكتمال كل الـ async writes المعلّقة. **لازم تستدعيها** قبل أي قراءة تعتمد على نتيجة async write سابقة، أو قبل driver unload.

---

#### `regmap_register_patch()`

```c
int regmap_register_patch(struct regmap *map,
                          const struct reg_sequence *regs,
                          int num_regs);
```

بتطبّق patch sequence على الـ hardware **وبتحفظها** بحيث تتطبّق تاني تلقائيًا لو الـ cache اتعمل reinit. مفيدة للـ errata workarounds.

---

#### `regmap_parse_val()`

```c
int regmap_parse_val(struct regmap *map, const void *buf, unsigned int *val);
```

بتحوّل raw buffer (بالـ byte order الصح حسب الـ config) لـ `unsigned int` value. مفيدة في الـ bus-level code.

---

#### `dev_get_regmap()`

```c
struct regmap *dev_get_regmap(struct device *dev, const char *name);
```

بتجيب الـ regmap المربوط بـ device بالاسم. لو `name == NULL`، بتجيب الأول. بتستخدمها الـ MFD children للوصول لـ regmap الـ parent.

```c
/* في MFD child driver */
struct regmap *map = dev_get_regmap(dev->parent, NULL);
```

---

#### `regmap_check_range_table()` / `regmap_reg_in_range()` / `regmap_reg_in_ranges()`

```c
bool regmap_check_range_table(struct regmap *map, unsigned int reg,
                              const struct regmap_access_table *table);

static inline bool regmap_reg_in_range(unsigned int reg,
                                       const struct regmap_range *range)
{
    return reg >= range->range_min && reg <= range->range_max;
}

bool regmap_reg_in_ranges(unsigned int reg,
                          const struct regmap_range *ranges,
                          unsigned int nranges);
```

الـ `check_range_table` بتتحقق من الـ `no_ranges` أولًا (ترجع false لو موجود) ثم `yes_ranges` (ترجع true). بيستخدمها الـ regmap core في الـ `readable_reg`, `writeable_reg`, `volatile_reg`, `precious_reg` checks.

**استخدامها المباشر في driver:**

```c
bool wm8960_volatile_reg(struct device *dev, unsigned int reg)
{
    return regmap_reg_in_ranges(reg, wm8960_volatile_ranges,
                                ARRAY_SIZE(wm8960_volatile_ranges));
}
```

---

#### `regmap_mmio_attach_clk()` / `regmap_mmio_detach_clk()`

```c
int regmap_mmio_attach_clk(struct regmap *map, struct clk *clk);
void regmap_mmio_detach_clk(struct regmap *map);
```

بتربط/تفصل clock بـ MMIO regmap بعد الـ init. الـ regmap هيعمل `clk_enable` قبل كل access و`clk_disable` بعده. مفيدة لو الـ clock ما كانش متاح وقت الـ init.

---

#### `regmap_reinit_cache()`

```c
int regmap_reinit_cache(struct regmap *map,
                        const struct regmap_config *config);
```

بتحرر الـ cache الحالي وتبني واحد جديد من الـ config الجديدة. مفيدة في الـ drivers اللي بتغيّر الـ cache type بناءً على الـ hardware variant.

---

### ملاحظات مهمة — Locking & Context

```
┌─────────────────────────────────────────────────────┐
│              regmap Locking Model                    │
├─────────────────┬───────────────────────────────────┤
│ fast_io = false │ mutex (can sleep)                  │
│ fast_io = true  │ spinlock (no sleep)                │
│ disable_locking │ no lock (driver-managed)           │
│ custom lock/unlock │ driver callbacks               │
└─────────────────┴───────────────────────────────────┘
```

- **لا تستدعي regmap APIs وأنت حامل regmap lock** — ستحصل على deadlock
- **MMIO regmaps:** آمنة من interrupt context لو `fast_io = true` وـ cache = FLAT/NONE
- **I2C/SPI regmaps:** دايمًا sleepable — لا تستخدمها من interrupt handler بدون threaded IRQ
- **الـ `_async_` functions:** تتطلب `regmap_async_complete()` قبل read أو driver exit
- **الـ precious registers:** الـ regmap core لا يقرأها تلقائيًا (مثلاً في cache sync) — الـ driver مسؤول
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. الـ debugfs entries

الـ regmap بيعمل entries تلقائياً تحت `/sys/kernel/debug/regmap/` لكل device عندها regmap مسجّل.

```
/sys/kernel/debug/regmap/
  ├── <bus>-<device>/          # مثلاً: i2c-0-0048  أو  spi0.0
  │   ├── name                 # اسم الـ regmap (لو اتحدد في config)
  │   ├── registers            # dump كامل لكل الـ registers المعروفة
  │   ├── access               # جدول readable/writeable/volatile لكل register
  │   ├── cache_only           # هل الـ cache-only mode شغّال؟
  │   ├── cache_bypass         # هل الـ cache bypass شغّال؟
  │   └── cache_dirty          # هل في dirty entries في الـ cache؟
```

**قراءة الـ registers dump:**

```bash
# اقرأ كل الـ registers
cat /sys/kernel/debug/regmap/i2c-0-0048/registers

# مثال output:
# 00: 12
# 01: 00
# 02: ff
# 03: 34

# اقرأ جدول الـ access (volatile/readable/writeable)
cat /sys/kernel/debug/regmap/i2c-0-0048/access
# Register  Read  Write  Volatile  Precious
# 00        Y     Y      N         N
# 01        Y     N      Y         N
```

**قراءة حالة الـ cache:**

```bash
cat /sys/kernel/debug/regmap/spi0.0/cache_only
# N

cat /sys/kernel/debug/regmap/spi0.0/cache_dirty
# Y    ← يعني في تعديلات في الـ cache مش متزامنة مع الـ hardware
```

**لو الـ device عنده اسم:**

```bash
# لو حددت .name = "audio-codec" في regmap_config
ls /sys/kernel/debug/regmap/
# i2c-1-001a.audio-codec/
```

---

#### 2. الـ sysfs entries

الـ regmap نفسه ما بيعمل sysfs entries مباشرة، بس الـ device المبني عليه بيظهر في sysfs. الطريقة المفيدة:

```bash
# اعرف الـ device المرتبط بالـ regmap من خلال bus
ls /sys/bus/i2c/devices/
ls /sys/bus/spi/devices/

# مثلاً لـ PMIC على I2C:
cat /sys/bus/i2c/devices/0-0036/name
# pmic-xyz

# لو الـ driver بيعمل sysfs attributes (زي regulator):
cat /sys/class/regulator/regulator.0/microvolts
cat /sys/class/regulator/regulator.0/status
```

---

#### 3. الـ ftrace — tracepoints و events

الـ regmap عنده tracepoints مدمجة في `drivers/base/regmap/`:

```bash
# شوف كل الـ regmap events المتاحة
ls /sys/kernel/debug/tracing/events/regmap/

# الـ events الأساسية:
# regmap_reg_read          ← كل read operation
# regmap_reg_write         ← كل write operation
# regmap_bulk_read         ← bulk reads
# regmap_bulk_write        ← bulk writes
# regmap_hw_read_start     ← بداية الـ hardware read
# regmap_hw_read_done      ← نهاية الـ hardware read
# regmap_hw_write_start    ← بداية الـ hardware write
# regmap_hw_write_done     ← نهاية الـ hardware write
# regmap_cache_hit         ← قراءة من الـ cache (ما وصلت hardware)
# regmap_cache_miss        ← cache miss → رحت الـ hardware
# regmap_cache_bypass      ← bypass mode اشتغل
# regmap_sync              ← cache sync للـ hardware
# regmap_drop_region       ← drop cache region
```

**تفعيل الـ tracing لـ device معين:**

```bash
# فعّل كل الـ regmap events
echo 1 > /sys/kernel/debug/tracing/events/regmap/enable

# أو فعّل event محدد بس
echo 1 > /sys/kernel/debug/tracing/events/regmap/regmap_reg_write/enable

# شغّل الـ trace
echo 1 > /sys/kernel/debug/tracing/tracing_on

# افعل اللي يستفز المشكلة...

# اقرأ النتيجة
cat /sys/kernel/debug/tracing/trace

# مثال output:
# <idle>-0     [001] d... 1234.567890: regmap_reg_write: i2c-0-0048 reg=02 val=ff
# kworker-42   [000] d... 1234.568010: regmap_cache_hit: i2c-0-0048 reg=01 val=00
```

**filter على device معين:**

```bash
echo 'name == "i2c-0-0048"' > /sys/kernel/debug/tracing/events/regmap/regmap_reg_write/filter
```

---

#### 4. الـ printk و Dynamic Debug

**تفعيل dynamic debug للـ regmap subsystem:**

```bash
# فعّل كل messages في drivers/base/regmap/
echo 'file drivers/base/regmap/* +p' > /sys/kernel/debug/dynamic_debug/control

# أو module-level لو الـ driver في module:
echo 'module my_audio_driver +p' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ messages في dmesg
dmesg -w | grep -i regmap
```

**من داخل الـ driver (للتطوير):**

```c
/* طباعة قيمة register بعد الـ write مباشرة */
regmap_write(map, REG_CTRL, val);
dev_dbg(dev, "REG_CTRL written: 0x%02x\n", val);

/* تحقق من القيمة فعلاً اتكتبت */
unsigned int readback;
regmap_read(map, REG_CTRL, &readback);
dev_dbg(dev, "REG_CTRL readback: 0x%02x (expected 0x%02x)\n", readback, val);
```

**printk levels مهمة:**

```bash
# ارفع مستوى الـ kernel log لتشوف KERN_DEBUG
echo 8 > /proc/sys/kernel/printk

# أو من kernel cmdline:
# loglevel=8 ignore_loglevel
```

---

#### 5. الـ Kernel Config options للـ debugging

| Config | الوظيفة |
|--------|---------|
| `CONFIG_REGMAP` | الـ regmap نفسه — لازم يكون enabled |
| `CONFIG_REGMAP_DEBUGFS` | يفعّل الـ `/sys/kernel/debug/regmap/` entries |
| `CONFIG_REGMAP_I2C` | دعم I2C bus |
| `CONFIG_REGMAP_SPI` | دعم SPI bus |
| `CONFIG_REGMAP_MMIO` | دعم MMIO |
| `CONFIG_REGMAP_IRQ` | دعم الـ IRQ controller |
| `CONFIG_DEBUG_REGMAP` | (بعض kernels) طباعة extra debug info |
| `CONFIG_LOCKDEP` | تتبع الـ locking violations في الـ regmap |
| `CONFIG_PROVE_LOCKING` | اكتشاف deadlocks في الـ regmap locking |
| `CONFIG_DYNAMIC_DEBUG` | يخلّي `dev_dbg` قابلة للتفعيل runtime |
| `CONFIG_FTRACE` | لازم يكون enabled عشان الـ tracepoints |
| `CONFIG_TRACING` | infrastructure للـ regmap events |
| `CONFIG_STACK_TRACER` | يضيف stack trace لكل regmap event |

**فعّلهم في `.config`:**

```bash
scripts/config --enable CONFIG_REGMAP_DEBUGFS
scripts/config --enable CONFIG_DYNAMIC_DEBUG
scripts/config --enable CONFIG_LOCKDEP
make olddefconfig
```

---

#### 6. أدوات خاصة بالـ subsystem

**regmap + devlink (للـ MMIO-based devices):**

```bash
# مفيش devlink integration مباشرة للـ regmap
# بس لو الـ device network-based (مثلاً MDIO switch):
devlink dev info pci/0000:01:00.0
```

**i2c-tools للتحقق من الـ I2C regmap:**

```bash
# اقرأ register مباشرة من الـ hardware بدون kernel driver
i2cget -y 0 0x48 0x02 b    # bus=0, addr=0x48, reg=0x02, byte mode

# اكتب مباشرة للـ hardware
i2cset -y 0 0x48 0x02 0xff b

# dump كل الـ registers
i2cdump -y 0 0x48
```

**spi-tools / spidev للـ SPI regmap:**

```bash
# اقرأ من SPI device مباشرة باستخدام spidev
# (لازم CONFIG_SPI_SPIDEV مفعّل وـ device registered)
python3 -c "
import spidev
spi = spidev.SpiDev()
spi.open(0, 0)
spi.max_speed_hz = 1000000
# Read register 0x02: send [0x82, 0x00] (MSB=1 → read mode)
resp = spi.xfer2([0x82, 0x00])
print(hex(resp[1]))
spi.close()
"
```

**regmap + MDIO:**

```bash
# اقرأ MDIO register مباشرة
mdio-tool read eth0 0x00    # port 0, register 0
phytool read eth0/0/0       # PHY 0, dev 0, reg 0
```

---

#### 7. جدول الـ Error Messages الشائعة

| رسالة الـ kernel | المعنى | الحل |
|-----------------|--------|------|
| `regmap API is disabled` | `CONFIG_REGMAP` مش enabled | فعّل `CONFIG_REGMAP` في `.config` |
| `regmap: no such device` | الـ regmap ما اتعملش init أو اتحذف | تأكد من `regmap_init_*` بيرجع قيمة سليمة |
| `regmap: Invalid register X` | الـ register بره الـ `max_register` أو في `no_ranges` | راجع `max_register` وـ `rd_table`/`wr_table` في `regmap_config` |
| `regmap: not readable` | محاولة قراءة register الـ `readable_reg` callback بيرجع false ليه | راجع `readable_reg()` أو `rd_table` |
| `regmap: not writable` | محاولة كتابة register الـ `writeable_reg` callback بيرجع false ليه | راجع `writeable_reg()` أو `wr_table` |
| `regmap: precious register read outside of a call` | قراءة register precious (clear-on-read) من غير السياق الصح | استخدم `regmap_read_bypassed()` أو راجع `precious_reg` |
| `regmap: cache sync failed: -EIO` | الـ hardware ما رد أثناء cache sync | مشكلة في الـ bus (I2C/SPI timeout)، افحص الـ hardware |
| `regmap: cache_only already set` | محاولة تفعيل cache_only وهي شغّالة | راجع الـ flow في الـ driver |
| `i2c i2c-0: error -110 (ETIMEDOUT)` | الـ I2C device ما ردّش في الوقت | افحص الـ pull-up resistors والـ clock speed والـ hardware |
| `spi_master spi0: transfer timeout` | SPI transfer خلص وقته | افحص الـ CS، الـ clock، والـ wiring |
| `regmap: lockdep violation` | deadlock في الـ regmap locking | ابعد عن استدعاء regmap من داخل الـ lock الخاص بيه |
| `regmap: Failed to sync cache` | خطأ في write أثناء `regcache_sync()` | عادةً بعد `regcache_cache_only()` → افحص الـ hardware جاهز |
| `regmap: Error in reg_read` | الـ custom `reg_read` callback رجّع error | افحص الـ custom read function |

---

#### 8. نقاط استراتيجية لـ `dump_stack()` و `WARN_ON()`

```c
/* في regmap_config callbacks — تحقق إن الـ register valid */
static bool my_writeable_reg(struct device *dev, unsigned int reg)
{
    bool writable = (reg < MY_MAX_REG) && !(reg & 0x01); /* even regs only */
    WARN_ON(!writable && reg < MY_MAX_REG);  /* unexpected write attempt */
    return writable;
}

/* قبل regcache_sync — تأكد الـ hardware صحي */
static int my_driver_resume(struct device *dev)
{
    struct my_priv *priv = dev_get_drvdata(dev);
    int ret;

    ret = my_hw_verify_alive(priv);
    WARN_ON(ret);   /* hardware مش شغّال قبل الـ sync */
    if (ret)
        return ret;

    return regcache_sync(priv->regmap);
}

/* بعد regmap_write لـ critical register */
static int configure_power_rail(struct my_priv *priv, unsigned int mv)
{
    unsigned int reg_val = mv_to_reg(mv);
    unsigned int readback;
    int ret;

    ret = regmap_write(priv->regmap, REG_VOLTAGE, reg_val);
    if (WARN_ON(ret))
        return ret;

    /* verify write succeeded — مهم لـ safety-critical registers */
    regmap_read(priv->regmap, REG_VOLTAGE, &readback);
    WARN_ON(readback != reg_val);

    return 0;
}

/* لو شكّك إن في race condition في الـ locking */
static irqreturn_t my_irq_handler(int irq, void *data)
{
    struct my_priv *priv = data;
    unsigned int status;

    /* هنا لو الـ regmap مش fast_io هيحصل deadlock */
    WARN_ON(!in_interrupt() && !priv->map_uses_spinlock);

    regmap_read(priv->regmap, REG_IRQ_STATUS, &status);
    /* ... */
    return IRQ_HANDLED;
}
```

---

### Hardware Level

#### 1. التحقق من حالة الـ hardware تطابق الـ kernel state

الفكرة: اقرأ الـ register من الـ hardware مباشرة وقارنه بما يظنه الـ kernel.

```bash
# الخطوة 1: اقرأ من kernel (يستخدم cache لو متفعّل)
cat /sys/kernel/debug/regmap/i2c-0-0048/registers | grep "^02:"
# 02: ff

# الخطوة 2: اقرأ من hardware مباشرة (bypass cache)
i2cget -y 0 0x48 0x02 b
# 0xfe   ← مختلف! يعني الـ cache stale أو write فشل

# لو MMIO device:
devmem2 0xFE001008 b    # اقرأ byte من العنوان الفيزيائي
```

**من داخل الـ kernel module للمقارنة:**

```c
unsigned int cached_val, hw_val;

/* قراءة من الـ cache */
regmap_read(map, REG_CTRL, &cached_val);

/* قراءة من الـ hardware مباشرة (bypass) */
regcache_cache_bypass(map, true);
regmap_read(map, REG_CTRL, &hw_val);
regcache_cache_bypass(map, false);

if (cached_val != hw_val)
    dev_warn(dev, "Cache mismatch at REG_CTRL: cache=0x%x hw=0x%x\n",
             cached_val, hw_val);
```

---

#### 2. الـ Register Dump techniques

**devmem2 للـ MMIO devices:**

```bash
# اقرأ byte واحدة
devmem2 0xFE001000 b

# اقرأ 16-bit word
devmem2 0xFE001000 h

# اقرأ 32-bit word
devmem2 0xFE001000 w

# اكتب قيمة
devmem2 0xFE001000 w 0xDEADBEEF
```

**عن طريق `/dev/mem` مع Python:**

```bash
python3 -c "
import mmap, struct

BASE = 0xFE001000
SIZE = 0x100

fd = open('/dev/mem', 'r+b')
mem = mmap.mmap(fd.fileno(), SIZE, offset=BASE)

# اقرأ 32-bit register
val = struct.unpack('<I', mem[0:4])[0]
print(f'REG[0x00] = 0x{val:08x}')

# dump أول 16 registers
for i in range(0, 0x40, 4):
    val = struct.unpack('<I', mem[i:i+4])[0]
    print(f'0x{BASE+i:08x}: 0x{val:08x}')

mem.close()
fd.close()
"
```

**لـ I2C — كل الـ registers دفعة واحدة:**

```bash
# dump كل 256 byte
i2cdump -y 0 0x48 b

# مثال output:
#      0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
# 00: 12 00 ff 34 00 00 80 00 00 00 00 00 00 00 00 00
```

---

#### 3. Logic Analyzer / Oscilloscope tips

**I2C Protocol Analysis:**

```
SDA ──┐ START ┌───────────────────────────────┐ STOP
SCL ──┘       └───────────────────────────────┘

ابحث عن:
- ACK/NAK: بعد كل byte، الـ SDA يجي low = ACK ← الـ device رد
           لو SDA بقي high = NACK ← مشكلة!
- Clock stretching: الـ device بيمسك SCL low → driver timeout
- الـ address byte: 7 bits address + 1 bit R/W
  مثال: 0x48 write → 0x90 (10010000b) على الـ wire

نقاط القياس:
- SDA و SCL مع trigger على START condition
- قِس الـ rise/fall time ← لو بطيء = pull-up مرتفع جداً
```

**SPI Protocol Analysis:**

```
CS   ─┐                                    ┌─
CLK   └──┐  ┌──┐  ┌──┐  ┌──┐  ┌──┐  ┌──┐
MOSI  ───── D7 ─── D6 ─── D5 ─ ... ─── D0 ─
MISO  ───── D7 ─── D6 ─── D5 ─ ... ─── D0 ─

ابحث عن:
- CS polarity: بعض الـ devices active-high مش active-low
- CPOL/CPHA: لو الـ data مش محدد صح → مشكلة CPOL/CPHA في regmap_config
- الفاصل بين transactions: بعض الـ devices محتاجة CS idle time
```

**الأشياء اللي تفتش عليها:**

| الظاهرة | السبب المحتمل |
|---------|--------------|
| NACK على I2C | الـ device مش موجود أو busy أو مش powered |
| SDA/SCL عالقة low | bus conflict أو device مش responsive |
| Noise على MISO | floating MISO، ضعيف pull-up، أو cross-talk |
| CS لا يرجع high | bug في الـ CS GPIO أو driver |
| بيانات مقلوبة | CPOL/CPHA غلط في `regmap_config` |

---

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة على الـ hardware | Pattern في الـ kernel log |
|------------------------|--------------------------|
| Device مش powered | `i2c i2c-0: error -5 (EIO)` — كل transaction |
| Pull-up مرتفع جداً | `i2c i2c-0: error -110 (ETIMEDOUT)` — intermittent |
| CS مش متوصل صح | `spi_master spi0: transfer timeout` — كل transfer |
| Wrong I2C address | `i2c i2c-0: probe failed: -ENODEV` |
| Clock speed عالي جداً | `i2c i2c-0: error -5` بعد ضغط hardware |
| VDDIO مش صح | Register values غريبة أو NACK عشوائي |
| Hardware reset مش شغّال | Device بيتصرف بقيم POR لا الـ configured values |
| Address conflict على I2C | `i2c-core: driver X wants to bind ... already bound` |

---

#### 5. الـ Device Tree Debugging

**تحقق إن الـ DT يطابق الـ hardware:**

```bash
# اقرأ الـ compiled DTB
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | less

# أو اقرأ DT الخاص بـ device معين
cat /sys/firmware/devicetree/base/soc/i2c@fe001000/my-codec@48/compatible
# my-vendor,my-codec

# تحقق من الـ reg property (I2C address)
cat /sys/firmware/devicetree/base/soc/i2c@fe001000/my-codec@48/reg | xxd
# 00000000: 0000 0048    ← 0x48 صح

# تحقق من الـ regmap related properties
cat /sys/firmware/devicetree/base/.../reg-io-width | xxd
# 00000000: 0000 0001    ← byte access
```

**نقاط تحقق الـ DT للـ regmap:**

```bash
# تأكد الـ node status enabled
grep -r "status" /sys/firmware/devicetree/base/soc/i2c*/

# تحقق الـ interrupts محددة صح
cat /sys/firmware/devicetree/base/.../interrupts | xxd

# تحقق الـ clocks المطلوبة للـ regmap_init_mmio_clk
cat /sys/firmware/devicetree/base/.../clocks | xxd
```

**لو الـ DT property غلط بتشوف في dmesg:**

```
my_driver: Failed to get regmap: -ENODEV
# أو
my_driver: Failed to parse DT node
```

**مقارنة DT مع الـ hardware (مثال MMIO):**

```
DT:       reg = <0x0 0xFE001000 0x0 0x1000>;
hardware: MMIO base = 0xFE001000, size = 4KB
i2c:      reg = <0x48>;    ← 7-bit address
hardware: A0/A1/A2 pins → 0b001 → 0x49  ← مختلف! هنا المشكلة
```

---

### Practical Commands

#### Shell Commands جاهزة للنسخ

**1. اكتشف كل الـ regmap devices الموجودة:**

```bash
ls -la /sys/kernel/debug/regmap/
```

**2. dump كامل لـ device معين:**

```bash
DEV="i2c-0-0048"
echo "=== Register Dump for $DEV ==="
cat /sys/kernel/debug/regmap/${DEV}/registers

echo "=== Cache Status ==="
cat /sys/kernel/debug/regmap/${DEV}/cache_only
cat /sys/kernel/debug/regmap/${DEV}/cache_bypass
cat /sys/kernel/debug/regmap/${DEV}/cache_dirty

echo "=== Access Table ==="
cat /sys/kernel/debug/regmap/${DEV}/access
```

**3. فعّل full regmap tracing:**

```bash
# تنظيف قبل
echo > /sys/kernel/debug/tracing/trace

# فعّل كل regmap events
echo 1 > /sys/kernel/debug/tracing/events/regmap/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on

# استنى المشكلة تحصل...
sleep 5

# وقف واقرأ
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace | grep -E "(reg_read|reg_write|cache)" | head -50
```

**4. فعّل tracing لـ device واحد بس:**

```bash
# filter على اسم الـ regmap
echo 'name == "i2c-0-0048"' > /sys/kernel/debug/tracing/events/regmap/regmap_reg_write/filter
echo 1 > /sys/kernel/debug/tracing/events/regmap/regmap_reg_write/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

**5. مقارنة الـ cache بالـ hardware لـ I2C device:**

```bash
#!/bin/bash
# compare_regmap_cache.sh - قارن cache مع hardware
REGMAP_DEV="i2c-0-0048"
I2C_BUS=0
I2C_ADDR=0x48

echo "Register | Kernel Cache | Hardware Read | Match?"
echo "---------|-------------|---------------|-------"

while IFS=": " read -r reg val; do
    reg_dec=$((16#$reg))
    hw_val=$(i2cget -y $I2C_BUS $I2C_ADDR 0x$reg b 2>/dev/null | tr -d '0x')
    if [ "$val" = "$hw_val" ]; then
        match="YES"
    else
        match="NO *** MISMATCH ***"
    fi
    printf "0x%s    | 0x%s        | 0x%s          | %s\n" "$reg" "$val" "$hw_val" "$match"
done < /sys/kernel/debug/regmap/${REGMAP_DEV}/registers
```

**6. فعّل dynamic debug للـ regmap:**

```bash
# فعّل كل debug messages في regmap core
echo 'file drivers/base/regmap/* +p' > /sys/kernel/debug/dynamic_debug/control

# تحقق إنه اتفعّل
grep "regmap" /sys/kernel/debug/dynamic_debug/control | grep "=p"

# اتابع الـ messages
dmesg -w | grep -E "(regmap|i2c-0-0048)"
```

**7. crash هنا — إيه اللي حصل؟**

```bash
# لو شوفت kernel oops مرتبط بـ regmap:
# 1. ابحث في الـ oops عن العنوان
# 2. استخدم addr2line
ADDR=0xffffffffc01234ab
DRIVER_MODULE=my_audio_driver.ko
addr2line -e /path/to/my_audio_driver.ko -a $ADDR -f

# أو استخدم scripts/faddr2line
./scripts/faddr2line my_audio_driver.ko my_function+0x42
```

**8. تحقق من الـ IRQ مع regmap_irq:**

```bash
# شوف IRQ statistics
cat /proc/interrupts | grep my_codec

# شوف virq mapping
cat /sys/kernel/debug/irq/irqs/*/name | grep my_codec

# فعّل tracing للـ IRQ handler
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_entry/enable
echo 'name == "my_codec"' > /sys/kernel/debug/tracing/events/irq/irq_handler_entry/filter
```

---

#### مثال Output وكيف تفسره

**مثال output من الـ registers dump:**

```
00: 12    ← REG_CHIP_ID = 0x12 (متوقع 0x12) ✓
01: 00    ← REG_STATUS  = 0x00 (لا errors)  ✓
02: ff    ← REG_CTRL    = 0xff (كل bits مفعّلة)
03: 34    ← REG_VOL     = 0x34 = 52 decimal
ff: ??    ← register بره الـ range → مش موجود
```

**مثال output من الـ ftrace:**

```
kworker/0:1-12  [000] .... 4321.123456: regmap_reg_write: i2c-0-0048 reg=02 val=ff
kworker/0:1-12  [000] .... 4321.123500: regmap_hw_write_start: i2c-0-0048 count=2 addr=02
kworker/0:1-12  [000] .... 4321.124200: regmap_hw_write_done: i2c-0-0048 count=2
# الـ write أخدت: 4321.124200 - 4321.123500 = 0.7ms → معقول لـ I2C

kworker/0:1-12  [000] .... 4321.124300: regmap_reg_read: i2c-0-0048 reg=01 val=00
kworker/0:1-12  [000] .... 4321.124305: regmap_cache_hit: i2c-0-0048 reg=01 val=00
# الـ read جه من الـ cache مباشرة (0.005ms) ← ما راحش الـ hardware
```

**لو شوفت `regmap_cache_miss` كتير:**
- الـ registers المسجّلة كـ volatile كتير جداً
- راجع `volatile_reg` callback أو `volatile_table`
- الـ volatile registers دايماً بتقرأ من الـ hardware → overhead

**لو شوفت `regmap_sync` بيأخد وقت طويل:**
- الـ cache فيه كتير dirty entries
- ممكن تعمل `regcache_sync_region()` بدل `regcache_sync()` لتزامن range محدود بس
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: RK3562 Industrial Gateway — كاش الـ regmap بيعطي قراءات قديمة بعد الـ suspend

#### العنوان
**Stale regmap cache corrupts UART config after system resume on RK3562 gateway**

#### السياق
شركة بتبني industrial gateway على RK3562 للتحكم في خطوط إنتاج. الـ gateway بيستخدم UART عبر external PMIC (بتتحكم فيه عبر I2C) لتوصيل حساسات. الـ PMIC driver بيستخدم `regmap_init_i2c()` مع `REGCACHE_MAPLE`.

#### المشكلة
بعد كل `system suspend/resume`، الحساسات بتبطل ترسل data. الـ logs بتظهر:

```
uart-pl011 fe650000.uart: parity error
```

الغريب إن الـ UART تمام لو عملنا module reload كامل. المشكلة بتحصل بس بعد resume.

#### التحليل
الـ engineer لقى إن الـ PMIC driver بيعمل `regcache_cache_only()` وقت الـ suspend، لكن مش بيعمل `regcache_mark_dirty()` قبله:

```c
/* في suspend callback — الخطأ هنا */
static int pmic_suspend(struct device *dev)
{
    struct pmic_data *pmic = dev_get_drvdata(dev);
    /* يوقف الكتابة للهاردوير لكن بيفضل الكاش زي ما هو */
    regcache_cache_only(pmic->regmap, true);
    return 0;
}
```

بعد الـ resume، الـ `regcache_sync()` بيشوف إن الكاش "نظيف" (مفيش dirty bits) فمبيكتبش حاجة للهاردوير. الـ PMIC رجع لـ power-on defaults بعد الـ suspend لكن الكاش لسه شايل القيم القديمة اللي الـ driver كتبها.

الـ flow من الكود:

```c
/* regmap.h — regcache_mark_dirty() declaration */
void regcache_mark_dirty(struct regmap *map);

/* regcache_sync() بيكتب بس الـ dirty registers للهاردوير */
int regcache_sync(struct regmap *map);
```

لما الـ PMIC بيرجع من power-off state، كل الـ registers رجعت لـ reset values. الكاش مش dirty، فـ `regcache_sync()` ما عملش حاجة — والـ UART config (baud rate، parity) فضلت wrong.

#### الحل

```c
static int pmic_suspend(struct device *dev)
{
    struct pmic_data *pmic = dev_get_drvdata(dev);

    /* mark cache dirty BEFORE going cache-only */
    /* عشان resume يعيد كتابة كل الـ registers */
    regcache_mark_dirty(pmic->regmap);
    regcache_cache_only(pmic->regmap, true);
    return 0;
}

static int pmic_resume(struct device *dev)
{
    struct pmic_data *pmic = dev_get_drvdata(dev);

    regcache_cache_only(pmic->regmap, false);

    /* دلوقتي هيكتب كل الكاش للهاردوير */
    return regcache_sync(pmic->regmap);
}
```

للتحقق:

```bash
# قبل الـ fix — شوف الـ regmap cache state
cat /sys/kernel/debug/regmap/0-0058/cache_dirty

# بعد resume لازم يبقى false بعد sync ناجح
```

#### الدرس المستفاد
`regcache_cache_only()` بس مش كفاية وقت الـ suspend. لو الهاردوير ممكن يخسر state (مثلاً PMIC اللي بيعمل power-off)، لازم `regcache_mark_dirty()` قبل الـ suspend عشان الـ `regcache_sync()` في الـ resume يعيد كتابة كل حاجة.

---

### السيناريو الثاني: STM32MP1 IoT Sensor — `volatile_reg` غلط بيأكل CPU

#### العنوان
**Incorrect volatile_reg callback causes CPU spin on STM32MP1 IoT node**

#### السياق
شركة بتعمل IoT sensor node على STM32MP1. بيستخدموا SPI IMU sensor (ICM-42688). الـ driver بيستخدم `regmap_init_spi()`. الهدف إن الـ CPU يقرأ الـ FIFO data بأعلى throughput ممكن.

#### المشكلة
الـ CPU load وصل 95% حتى لو الـ sensor بيرسل data بمعدل منخفض. الـ `top` بيوري إن كل الـ CPU في kernel space.

#### التحليل
الـ engineer عمل `perf top` ولقى إن الوقت كله في:

```
regmap_read → regmap_cache_read → _regmap_read (hardware)
```

المشكلة إن الـ `volatile_reg` callback بيرجع `true` لكل الـ registers:

```c
/* الكود المشكلة */
static bool icm42688_volatile_reg(struct device *dev, unsigned int reg)
{
    /* المطور فضل conservative فعمل كل حاجة volatile */
    return true;
}

static const struct regmap_config icm42688_config = {
    .reg_bits   = 8,
    .val_bits   = 8,
    .volatile_reg = icm42688_volatile_reg,
    .cache_type = REGCACHE_MAPLE,
    /* ... */
};
```

لما `volatile_reg` ترجع `true` لأي register، الـ regmap بيتخطى الكاش تماماً ويروح للهاردوير في كل قراءة. الكونفيج registers (sample rate، filter settings) اللي بتتقرأ كتير في الـ driver — كلها بتعمل SPI transactions فعلية.

من الـ `regmap.h`:

```c
/* volatile_reg — if true, register value can't be cached */
bool (*volatile_reg)(struct device *dev, unsigned int reg);
```

الـ FIFO data register فعلاً volatile (بيتغير كل frame). لكن الـ config registers ثابتة بعد الـ init.

#### الحل

```c
/* حدد بس الـ registers اللي فعلاً volatile */
static bool icm42688_volatile_reg(struct device *dev, unsigned int reg)
{
    switch (reg) {
    case ICM42688_REG_FIFO_DATA:    /* FIFO — volatile دايماً */
    case ICM42688_REG_INT_STATUS:   /* clear-on-read */
    case ICM42688_REG_TEMP_DATA1:   /* live sensor data */
    case ICM42688_REG_TEMP_DATA0:
    case ICM42688_REG_ACCEL_DATA_X1 ... ICM42688_REG_GYRO_DATA_Z0:
        return true;
    default:
        return false;   /* config registers — cacheable */
    }
}
```

أو باستخدام `volatile_table` من الـ `regmap_config`:

```c
static const struct regmap_range icm42688_volatile_ranges[] = {
    regmap_reg_range(ICM42688_REG_TEMP_DATA1, ICM42688_REG_GYRO_DATA_Z0),
    regmap_reg_range(ICM42688_REG_INT_STATUS, ICM42688_REG_INT_STATUS),
    regmap_reg_range(ICM42688_REG_FIFO_DATA,  ICM42688_REG_FIFO_DATA),
};

static const struct regmap_access_table icm42688_volatile_table = {
    .yes_ranges  = icm42688_volatile_ranges,
    .n_yes_ranges = ARRAY_SIZE(icm42688_volatile_ranges),
};

static const struct regmap_config icm42688_config = {
    /* ... */
    .volatile_table = &icm42688_volatile_table,
    .cache_type     = REGCACHE_MAPLE,
};
```

بعد الـ fix، CPU load نزل من 95% لـ 12%.

#### الدرس المستفاد
`volatile_reg` بيأثر مباشرة على الكاش. كل register اتحدد كـ volatile = hardware access في كل قراءة، حتى لو القيمة مش بتتغير أبداً. حدد بدقة بس الـ registers اللي فعلاً بتتغير بدون تدخل الـ CPU.

---

### السيناريو الثالث: i.MX8 Android TV Box — `read_flag_mask` ناقص بيكسر الـ SPI codec

#### العنوان
**Missing read_flag_mask causes garbled audio on i.MX8 Android TV box**

#### السياق
شركة بتصنع Android TV box على i.MX8M Plus. بيستخدموا audio codec (WM8960) عبر SPI. الـ driver بيستخدم `regmap_init_spi()`.

#### المشكلة
الصوت مش بيطلع خالص. الـ ALSA subsystem بيشتكي:

```
wm8960 spi0.0: ASoC: error at snd_soc_component_update_bits
```

الـ oscilloscope على SPI bus بيوري إن الـ read operations بتبعت نفس format الـ write — بدون set الـ R/W bit.

#### التحليل
معظم SPI devices بتستخدم MSB الأول (أو أكتر) في الـ address byte للتمييز بين read وwrite. الـ WM8960 مثلاً بيستخدم bit 0 في الـ address byte الثاني:

```
Write: [addr[7:1] | 0] [data[8:0]]
Read:  [addr[7:1] | 1] [data[8:0]]
```

الـ `regmap_config` ناقصه `read_flag_mask`:

```c
/* الكود المشكلة — read_flag_mask = 0 by default */
static const struct regmap_config wm8960_regmap = {
    .reg_bits  = 7,
    .val_bits  = 9,
    .cache_type = REGCACHE_RBTREE,
    /* read_flag_mask مش متحدد = 0 */
};
```

من الـ `regmap.h`:

```c
struct regmap_config {
    /* ... */
    /* Mask to be set in the top bytes of the register when doing a read */
    unsigned long read_flag_mask;
    unsigned long write_flag_mask;
    /* ... */
};
```

لما الـ regmap بيبعت read command، ما بيعملش set لأي bit في الـ address — فالـ device بيفسرها كـ write ويرجع garbage.

#### الحل

```c
static const struct regmap_config wm8960_regmap = {
    .reg_bits       = 7,
    .val_bits       = 9,
    .cache_type     = REGCACHE_RBTREE,
    /* set bit 0 of address byte for reads */
    .read_flag_mask = 0x01,
    .write_flag_mask = 0x00,
};
```

للتحقق باستخدام الـ debugfs:

```bash
# اقرأ register معروف وشوف الـ SPI trace
cat /sys/kernel/debug/regmap/spi0.0/registers

# أو استخدم logic analyzer لتأكيد الـ bit pattern
```

#### الدرس المستفاد
`read_flag_mask` و`write_flag_mask` في `struct regmap_config` لازم تتحدد بدقة حسب datasheet الـ SPI device. الـ default هو zero لكل منهم — لو الـ device بيحتاج تمييز read/write بـ bit في الـ address، لازم تحدده صراحةً وإلا كل الـ read operations هتبقى broken.

---

### السيناريو الرابع: AM62x Automotive ECU — `regmap_read_poll_timeout` بيعمل deadlock في ISR

#### العنوان
**regmap_read_poll_timeout used from ISR causes deadlock on AM62x automotive ECU**

#### السياق
شركة Tier-1 automotive بتطور ECU على AM62x (TI). بيتحكموا في external power management IC عبر I2C. الـ IC بيرفع IRQ لما يبقى fault condition. الـ IRQ handler بيحاول يقرأ الـ fault status register.

#### المشكلة
الـ ECU بيعمل hard lockup بعد ساعات من الـ runtime، بس بس لما الـ power fault بيحصل:

```
BUG: spinlock deadlock on CPU#0
INFO: possible circular locking dependency detected
```

الـ lockdep بيقول في الـ chain:

```
irq_handler -> regmap_read -> mutex_lock -> [already held by regmap_write in process context]
```

#### التحليل
الـ IRQ handler بيستخدم `regmap_read_poll_timeout()` لانتظار الـ status register يتغير:

```c
/* الكود المشكلة في الـ ISR */
static irqreturn_t pmic_fault_isr(int irq, void *data)
{
    struct pmic_priv *priv = data;
    unsigned int val;

    /* WRONG: regmap_read_poll_timeout يستخدم mutex ويمكن ينام */
    regmap_read_poll_timeout(priv->regmap,
                             PMIC_FAULT_STATUS_REG,
                             val,
                             (val & FAULT_CLEARED),
                             100,    /* sleep_us */
                             50000); /* timeout_us */
    return IRQ_HANDLED;
}
```

من الـ `regmap.h`، الـ macro:

```c
#define regmap_read_poll_timeout(map, addr, val, cond, sleep_us, timeout_us) \
({ \
    int __ret, __tmp; \
    __tmp = read_poll_timeout(regmap_read, __ret, __ret || (cond), \
            sleep_us, timeout_us, false, (map), (addr), &(val)); \
    __ret ?: __tmp; \
})
```

الـ `regmap_read()` بياخد الـ regmap mutex. الـ I2C regmap بيستخدم `mutex` (مش `spinlock`) بسبب إن `fast_io = false`. الـ mutex مش safe في interrupt context.

الـ comment في `regmap.h` صريح:

```c
/* Must not be called from atomic context if sleep_us or timeout_us are used. */
```

أما `regmap_read_poll_timeout_atomic()` فهو المناسب للـ atomic context — لكنه بيستخدم `udelay()` مش `usleep_range()`, وبيشترط MMIO أو flat/no cache.

#### الحل

للـ I2C devices، اللي مش ممكن تستخدمها في atomic context، لازم تحول الـ work للـ threaded IRQ:

```c
/* اطلب threaded IRQ */
static int pmic_probe(struct i2c_client *client)
{
    /* ... */
    return devm_request_threaded_irq(&client->dev, client->irq,
                                     NULL,            /* no hard IRQ handler */
                                     pmic_fault_thread,
                                     IRQF_ONESHOT,
                                     "pmic-fault",
                                     priv);
}

/* الـ thread handler — process context = safe للـ regmap I2C */
static irqreturn_t pmic_fault_thread(int irq, void *data)
{
    struct pmic_priv *priv = data;
    unsigned int val;
    int ret;

    /* safe هنا لأننا في thread context */
    ret = regmap_read_poll_timeout(priv->regmap,
                                   PMIC_FAULT_STATUS_REG,
                                   val,
                                   (val & FAULT_CLEARED),
                                   1000,    /* 1ms sleep */
                                   100000); /* 100ms timeout */
    if (ret)
        dev_err(&priv->client->dev, "fault did not clear: %d\n", ret);

    return IRQ_HANDLED;
}
```

#### الدرس المستفاد
`regmap_read_poll_timeout()` بيستخدم `usleep_range()` داخلياً — ممنوع في interrupt context. الـ I2C regmap بيستخدم mutex مش spinlock. لو الـ device على I2C أو SPI وعايز تعمل poll من IRQ، لازم تستخدم `request_threaded_irq()`. الـ `regmap_read_poll_timeout_atomic()` متاح بس للـ MMIO مع no cache أو flat cache.

---

### السيناريو الخامس: Allwinner H616 — `regmap_irq_chip` مع `ack_base` غلط بيعمل IRQ storm

#### العنوان
**Wrong ack_base in regmap_irq_chip causes IRQ storm on Allwinner H616 TV box**

#### السياق
مطور بيكتب driver لـ PMIC جديد (AXP313A) على Allwinner H616 في Android TV box. الـ PMIC بيدير الطاقة وعنده interrupt controller داخلي. الـ driver بيستخدم `regmap_irq_chip` framework من `regmap.h`.

#### المشكلة
فور ما الـ PMIC driver بيتحمل، الـ system بيتعلق. الـ `dmesg` بيوري:

```
irq 45: nobody cared (try booting with the "irqpoll" option)
handlers:
[<00000000xxxxxxxx>] irq_default_primary_handler
Disabling IRQ #45
```

والـ CPU load 100% قبل الـ disable. واضح إن في IRQ storm.

#### التحليل
الـ `regmap_irq_chip` setup:

```c
/* الكود المشكلة */
static const struct regmap_irq axp313a_irqs[] = {
    REGMAP_IRQ_REG(AXP313A_IRQ_PEK_SHORT, 0, BIT(0)),
    REGMAP_IRQ_REG(AXP313A_IRQ_PEK_LONG,  0, BIT(1)),
    REGMAP_IRQ_REG(AXP313A_IRQ_BATTERY,   0, BIT(2)),
};

static const struct regmap_irq_chip axp313a_irq_chip = {
    .name       = "axp313a",
    .status_base = AXP313A_IRQ_STATUS_REG,   /* 0x48 */
    .mask_base   = AXP313A_IRQ_MASK_REG,     /* 0x49 */
    .ack_base    = AXP313A_IRQ_STATUS_REG,   /* WRONG: نفس الـ status! */
    .num_regs    = 1,
    .irqs        = axp313a_irqs,
    .num_irqs    = ARRAY_SIZE(axp313a_irqs),
};
```

من `regmap.h`:

```c
struct regmap_irq_chip {
    /* ... */
    unsigned int ack_base; /* Base ack address. If zero then the chip is clear on read. */
    /* ... */
    unsigned int use_ack:1;
    /* ... */
};
```

الـ AXP313A عنده status register هو **clear-on-read**. الـ `ack_base` محدد بنفس الـ status register — لما الـ regmap_irq handler بيقرأ الـ status، الـ interrupt بيتكلير. لكن تحديد `ack_base` بيخلي الـ framework يعمل write للـ register بعدين — write بـ 0xFF بيعمل **set** لكل الـ interrupt bits من جديد (لأن الـ register bit behavior هو write-1-to-clear)! فالـ interrupts بتـ re-trigger في لوب.

#### الحل
للـ chips اللي status register بتاعها clear-on-read:

```c
static const struct regmap_irq_chip axp313a_irq_chip = {
    .name        = "axp313a",
    .status_base = AXP313A_IRQ_STATUS_REG,
    .mask_base   = AXP313A_IRQ_MASK_REG,
    /* لا تحدد ack_base — اتركه zero */
    /* .ack_base = 0, */       /* default: clear-on-read */
    /* use_ack = 0, */         /* default */
    .num_regs    = 1,
    .irqs        = axp313a_irqs,
    .num_irqs    = ARRAY_SIZE(axp313a_irqs),
};
```

أو لو الـ chip محتاج explicit ack register مختلف:

```c
static const struct regmap_irq_chip axp313a_irq_chip = {
    .name        = "axp313a",
    .status_base = AXP313A_IRQ_STATUS_REG,   /* 0x48 — read only */
    .mask_base   = AXP313A_IRQ_MASK_REG,     /* 0x49 */
    .ack_base    = AXP313A_IRQ_ACK_REG,      /* 0x4A — منفصل */
    .use_ack     = 1,
    .num_regs    = 1,
    .irqs        = axp313a_irqs,
    .num_irqs    = ARRAY_SIZE(axp313a_irqs),
};
```

للتحقق:

```bash
# شوف إن الـ IRQ count بيزيد بجنون = storm
watch -n 0.5 grep -E "45:" /proc/interrupts

# بعد الـ fix — لازم يزيد بشكل طبيعي مع الـ events بس
```

#### الدرس المستفاد
`ack_base` في `struct regmap_irq_chip` لو `zero` معناه الـ chip clear-on-read ومفيش حاجة تانية. لو حددت `ack_base` بنفس الـ status register في chip clear-on-read، هتعمل write بعد القراءة وده ممكن يعيد set الـ interrupts ويحدث IRQ storm. لازم تقرأ الـ datasheet بعناية وتفرق بين: clear-on-read، write-1-to-clear، وchips عندها explicit ack register منفصل.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

**الـ LWN.net** هو أهم مصدر لمتابعة تطور الـ regmap framework في الـ kernel — كل المقالات دي بتوثق مراحل مختلفة من تطوره:

| المقال | الأهمية |
|--------|---------|
| [regmap: Generic I2C and SPI register map library](https://lwn.net/Articles/451789/) | المقال الأصلي اللي أعلن فيه Mark Brown عن الـ regmap لأول مرة في 2011 — نقطة البداية |
| [regmap: introduce fast_io busses, and use a spinlock](https://lwn.net/Articles/490704/) | إضافة دعم الـ fast_io buses واستخدام `spinlock` بدل `mutex` |
| [regmap: Add asynchronous I/O support](https://lwn.net/Articles/534893/) | إضافة `regmap_raw_write_async()` و`regmap_async_complete()` للعمليات اللي ماعندهاش انتظار |
| [add syscon driver based on regmap](https://lwn.net/Articles/514764/) | إضافة الـ `syscon` driver — نموذج استخدام regmap للـ system controller registers |
| [regmap: provide simple bitops](https://lwn.net/Articles/821711/) | إضافة `regmap_set_bits()` و`regmap_clear_bits()` للعمليات البسيطة على البتات |
| [Add simple NVMEM Framework via regmap](https://lwn.net/Articles/651711/) | تكامل regmap مع إطار الـ NVMEM |
| [regmap: mmio: Extending to support IO ports](https://lwn.net/Articles/904225/) | توسيع دعم الـ MMIO ليشمل الـ IO ports |
| [Introduce a generic regmap-based MDIO driver](https://lwn.net/Articles/927209/) | virtual MDIO driver بيترجم الـ MDIO accesses لـ regmap |

---

### التوثيق الرسمي داخل الـ Kernel

الـ documentation الرسمي موجود في مسارات متعددة داخل شجرة الـ kernel:

```
Documentation/driver-api/regmap.rst
```
> الصفحة الرئيسية للـ regmap في الـ kernel docs — بتشرح الـ API الأساسية وكيفية استخدامها.

```
Documentation/devicetree/bindings/regmap/
```
> الـ device tree bindings المتعلقة بالـ regmap — خصوصاً للـ `syscon` و`simple-mfd`.

```
include/linux/regmap.h
```
> الـ header الأساسي — كل الـ structs والـ macros والـ API مكتوبة فيه مع docstrings كاملة.

```
drivers/base/regmap/
```
> الـ implementation الفعلي — `regmap.c`، `regcache.c`، `regcache-rbtree.c`، `regcache-flat.c`، `regcache-maple.c`، `regmap-i2c.c`، `regmap-spi.c`، `regmap-mmio.c`.

---

### Commits مهمة في تاريخ الـ regmap

| الحدث | الرابط |
|-------|--------|
| أول commit للـ regmap — Linux 3.1 (2011) | [github.com/torvalds/linux](https://github.com/torvalds/linux/commits/master/drivers/base/regmap) |
| الـ regmap header الحالي على master | [include/linux/regmap.h](https://github.com/torvalds/linux/blob/master/include/linux/regmap.h) |
| الـ regmap implementation على master | [drivers/base/regmap/regmap.c](https://github.com/torvalds/linux/blob/master/drivers/base/regmap/regmap.c) |
| إضافة maple tree cache — Linux 6.3 | [PATCH v2: regmap maple tree cache](https://lore.kernel.org/all/20230325-regcache-maple-v2-0-799dcab3ecb1@kernel.org/) |

---

### نقاشات Mailing List

**الـ LKML** هو المكان الرئيسي لمتابعة تطور الـ regmap:

- [GIT PULL: regmap updates for v6.20 — Mark Brown](https://lkml.org/lkml/2026/2/9/1009) — أحدث pull request للـ regmap
- [Mailing List: using the same regmap by multiple device drivers](https://lists.archive.carbon60.com/linux/kernel/2473649) — نقاش مهم عن مشاركة الـ regmap بين drivers متعددة
- [PATCH: regmap: cache downgrade log level](https://lists.archive.carbon60.com/linux/kernel/4796480) — نقاش حول رسائل الـ warning في الـ cache
- [LKML Archive: regmap fix order of write log (2011)](https://lkml.iu.edu/hypermail/linux/kernel/2011.1/06297.html) — من أولى النقاشات على الـ mailing list
- [PATCH: regmap: Implement dev_get_regmap()](https://lists.openwall.net/linux-kernel/2012/05/08/459) — إضافة API للبحث عن regmap موجود
- [regmap tree — Mark Brown's branch](https://lkml.kernel.org/lkml/E1b4tzz-0006fK-IS@debutante/) — الـ regmap tree الرسمي

---

### مقالات ومدونات تقنية

| المصدر | الرابط |
|--------|--------|
| Collabora Blog — Using regmaps to make drivers more generic | [collabora.com](https://www.collabora.com/news-and-blog/blog/2020/05/27/using-regmaps-to-make-linux-drivers-more-generic/) |
| OpenSourceForYou — regmap: Reducing the Redundancy in Linux Code | [opensourceforu.com](https://www.opensourceforu.com/2017/01/regmap-reducing-redundancy-linux-code/) |
| Pandy Song — Linux Kernel regmap | [pandysong.github.io](https://pandysong.github.io/blog/post/regmap/) |
| Embedded Linux Conference 2015 — MFD/regmap/syscon slides | [linuxfoundation.org](http://events17.linuxfoundation.org/sites/events/files/slides/belloni-mfd-regmap-syscon_0.pdf) |

---

### Kernelnewbies.org

**الـ kernelnewbies.org** بيوثق التغييرات الجديدة في كل إصدار — دي الصفحات الأهم:

- [Linux_6.14 — regmap changes](https://kernelnewbies.org/Linux_6.14)
- [Linux_6.12 — regmap changes](https://kernelnewbies.org/Linux_6.12)
- [Linux_6.11 — regmap changes](https://kernelnewbies.org/Linux_6.11)
- [Linux_6.8 — regmap changes](https://kernelnewbies.org/Linux_6.8)
- [Linux_4.17 — regmap changes](https://kernelnewbies.org/Linux_4.17)
- [Linux_4.9 — regmap changes](https://kernelnewbies.org/Linux_4.9)
- [Mailing list: how to check regmap start address](http://lists.kernelnewbies.org/pipermail/kernelnewbies/2020-April/020732.html)

---

### الكتب المرجعية

#### Linux Device Drivers, 3rd Edition (LDD3)
**الـ LDD3** مجاني online — الفصول الأهم للسياق:

- **Chapter 8: Allocating Memory** — الـ memory management الأساسي
- **Chapter 12: PCI Drivers** — فهم كيف تُكتب الـ drivers قبل ظهور regmap
- **Chapter 13: USB Drivers** — نفس المفهوم للـ USB

> ملاحظة: LDD3 قديم (2005) وما بيغطيش الـ regmap — لكنه مهم لفهم الـ context التاريخي اللي ظهر فيه regmap كحل للـ code duplication.

الكتاب مجاني على: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development, 3rd Edition — Robert Love
- **Chapter 17: Devices and Modules** — أساس فهم الـ device model
- **Chapter 18: Debugging** — أدوات الـ debugging المفيدة مع regmap

#### Embedded Linux Primer, 2nd Edition — Christopher Hallinan
- **Chapter 8: Device Driver Basics** — مقدمة عملية للـ device drivers
- **Chapter 16: Kernel Debugging Techniques** — debugging في البيئات المدمجة

#### Linux Kernel Programming — Kaiwan N. Billimoria (2021)
- **Part 3: Kernel Memory Allocation** — الأحدث والأشمل للـ regmap context
- مناسب أكتر من LDD3 للـ kernels الحديثة

---

### مصادر الـ Source Code للمراجعة المباشرة

```bash
# قراءة الـ API الكاملة
less include/linux/regmap.h

# الـ implementation الأساسي
less drivers/base/regmap/regmap.c

# أنواع الـ cache المختلفة
ls drivers/base/regmap/regcache*.c

# الـ bus backends
ls drivers/base/regmap/regmap-{i2c,spi,mmio,spi-avmm}.c

# مثال حقيقي على استخدام regmap في driver
less drivers/mfd/wm8994-core.c        # MFD مع regmap
less drivers/regulator/max8997.c      # Regulator مع regmap
less sound/soc/codecs/wm8960.c        # ALSA codec مع regmap
```

---

### Search Terms للبحث عن معلومات أكتر

للبحث في الـ kernel source:

```bash
# كل الـ drivers اللي بتستخدم regmap_init
git grep "regmap_init\|devm_regmap_init" drivers/ | head -50

# الـ regmap configs في الـ drivers الحقيقية
git grep "struct regmap_config" drivers/ | head -30

# تاريخ تطور الـ regmap
git log --oneline -- drivers/base/regmap/ | head -30

# من كتب أكتر تعديلات على الـ regmap
git log --format="%an" -- drivers/base/regmap/ | sort | uniq -c | sort -rn
```

للبحث على الإنترنت استخدم هذي الـ search terms:

```
linux kernel regmap tutorial
regmap_config val_bits reg_bits
devm_regmap_init_i2c example driver
regcache_sync regcache_drop_region
regmap_field_read regmap_field_write
linux kernel register map abstraction
syscon regmap device tree
regmap irq chip interrupt controller
```

---

### جدول ملخص المصادر حسب الأولوية

| الأولوية | المصدر | لماذا؟ |
|----------|--------|--------|
| 1 | `include/linux/regmap.h` + `Documentation/driver-api/regmap.rst` | المرجع الأول والأخير |
| 2 | [LWN: regmap introduction (2011)](https://lwn.net/Articles/451789/) | يشرح الـ motivation الأصلية |
| 3 | [Collabora Blog](https://www.collabora.com/news-and-blog/blog/2020/05/27/using-regmaps-to-make-linux-drivers-more-generic/) | مثال عملي حديث |
| 4 | [ELCE 2015 Slides](http://events17.linuxfoundation.org/sites/events/files/slides/belloni-mfd-regmap-syscon_0.pdf) | يربط regmap بـ MFD وsyscon |
| 5 | `drivers/base/regmap/regmap.c` | الـ implementation نفسه |
| 6 | kernelnewbies.org Linux release pages | تتبع التغييرات بكل إصدار |
| 7 | lore.kernel.org / lkml.org | النقاشات التفصيلية |
## Phase 8: Writing simple module

### الفكرة

**`regmap_write`** هي الدالة الأكتر استخداماً في الـ regmap API — كل driver بيكتب register بيمر بيها. هنعمل kprobe عليها عشان نشوف مين بيكتب إيه ومتى، من غير ما نعدل في أي driver.

---

### الـ Hook المختار

| العنصر | التفاصيل |
|--------|----------|
| الدالة | `regmap_write` |
| الأسلوب | **kprobe** على entry الدالة |
| البيانات المطبوعة | اسم الـ device، رقم الـ register، القيمة المكتوبة |
| الخطورة | صفر — read-only observation، مفيش تعديل |

```
                    kernel code
                         │
            ┌────────────▼────────────┐
            │      regmap_write()     │  ◄── kprobe هنا
            │  map  →  reg  →  val   │
            └────────────┬────────────┘
                         │
            ┌────────────▼────────────┐
            │   regmap_write_kp_pre   │  callback بتاعنا
            │  pr_info(dev, reg, val) │
            └─────────────────────────┘
```

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * regmap_write_tracer.c
 *
 * Trace every regmap_write() call system-wide using a kprobe.
 * Prints: device name, register address, value being written.
 */

#include <linux/module.h>      /* MODULE_* macros, module_init/exit         */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe, etc.      */
#include <linux/regmap.h>      /* struct regmap — to read device pointer     */
#include <linux/device.h>      /* regmap_get_device(), dev_name()            */
#include <linux/printk.h>      /* pr_info()                                  */

/* ------------------------------------------------------------------ */
/*  kprobe callback — runs BEFORE regmap_write() executes             */
/* ------------------------------------------------------------------ */

/*
 * regmap_write signature:
 *   int regmap_write(struct regmap *map, unsigned int reg, unsigned int val);
 *
 * On x86-64:  map → rdi,  reg → rsi,  val → rdx
 * On arm64:   map → x0,   reg → x1,   val → x2
 * We read them via regs_get_kernel_argument() which is arch-agnostic.
 */
static int regmap_write_kp_pre(struct kprobe *p, struct pt_regs *regs)
{
	/* الـ arguments بتاعت الدالة من الـ registers */
	struct regmap   *map = (struct regmap *)regs_get_kernel_argument(regs, 0);
	unsigned int     reg = (unsigned int)   regs_get_kernel_argument(regs, 1);
	unsigned int     val = (unsigned int)   regs_get_kernel_argument(regs, 2);

	/* regmap_get_device بترجع الـ struct device المرتبط بالـ map */
	struct device   *dev = regmap_get_device(map);

	/* نطبع اسم الـ device مع العنوان والقيمة بالـ hex */
	pr_info("regmap_write_tracer: dev=%s  reg=0x%04x  val=0x%08x\n",
		dev ? dev_name(dev) : "unknown", reg, val);

	return 0; /* 0 = استمر في تنفيذ الدالة الأصلية */
}

/* ------------------------------------------------------------------ */
/*  kprobe struct                                                       */
/* ------------------------------------------------------------------ */

static struct kprobe rw_kp = {
	.symbol_name = "regmap_write",   /* اسم الرمز في الـ kernel symbol table */
	.pre_handler = regmap_write_kp_pre,
};

/* ------------------------------------------------------------------ */
/*  module init / exit                                                  */
/* ------------------------------------------------------------------ */

static int __init regmap_tracer_init(void)
{
	int ret;

	/* نسجّل الـ kprobe — الـ kernel هيربطه بعنوان regmap_write تلقائياً */
	ret = register_kprobe(&rw_kp);
	if (ret < 0) {
		pr_err("regmap_write_tracer: register_kprobe failed: %d\n", ret);
		return ret;
	}

	pr_info("regmap_write_tracer: planted at %p, watching regmap_write()\n",
		rw_kp.addr);
	return 0;
}

static void __exit regmap_tracer_exit(void)
{
	/* لازم نشيل الـ kprobe قبل ما الـ module يتفك من الـ kernel */
	unregister_kprobe(&rw_kp);
	pr_info("regmap_write_tracer: removed\n");
}

module_init(regmap_tracer_init);
module_exit(regmap_tracer_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Learner <learn@example.com>");
MODULE_DESCRIPTION("Trace all regmap_write() calls via kprobe");
```

---

### Makefile

```makefile
obj-m += regmap_write_tracer.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل قسم

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/kprobes.h` | يعرّف `struct kprobe` ودوال `register_kprobe` / `unregister_kprobe` |
| `linux/regmap.h` | يعرّف `struct regmap` ودالة `regmap_get_device()` |
| `linux/device.h` | يعرّف `dev_name()` اللي بترجع اسم الـ device كـ string |

**الـ** `linux/kprobes.h` ضروري عشان من غيره مفيش تعريف للـ `struct kprobe` ولا للـ `pt_regs` helpers.

#### الـ Callback: `regmap_write_kp_pre`

**الـ** `regs_get_kernel_argument(regs, N)` دالة portable بتجيب الـ argument رقم N من الـ registers بصرف النظر عن الـ architecture. ده أحسن من قراءة `rdi/rsi` مباشرة لأنه بيشتغل على x86 و ARM و RISC-V بنفس الكود.

الـ `regmap_get_device(map)` بترجع الـ `struct device*` المخزّن جوا الـ opaque `struct regmap` — من غيرها مش هنعرف نطبع اسم الـ device لأن الـ struct مش مكشوف في الـ header.

#### الـ kprobe Struct

```c
static struct kprobe rw_kp = {
    .symbol_name = "regmap_write",
    .pre_handler = regmap_write_kp_pre,
};
```

**الـ** `symbol_name` بدل العنوان المباشر عشان الـ kernel هو اللي بيحل الرمز — أأمن وأسهل من hardcode. **الـ** `pre_handler` بيتشغّل قبل ما الدالة تنفّذ، يعني الـ args لسه موجودة في الـ registers.

#### `module_init` / `module_exit`

**الـ** `register_kprobe` في الـ init بيكتب breakpoint instruction في الـ kernel text عند عنوان `regmap_write`، وبيحفظ الـ original bytes. **الـ** `unregister_kprobe` في الـ exit ضروري جداً عشان يرجّع الـ bytes الأصلية ويمنع crash لو جه call لـ `regmap_write` بعد ما الـ module اتشال وعنوان الـ handler اتحرر.

---

### تشغيل وتجربة

```bash
# بناء
make

# تحميل
sudo insmod regmap_write_tracer.ko

# مراقبة الـ output في real-time
sudo dmesg -w | grep regmap_write_tracer

# مثال: لو عندك sound card أو PMIC هتشوف رسائل زي:
# regmap_write_tracer: dev=0-001a  reg=0x0002  val=0x000080
# regmap_write_tracer: dev=soc:audio  reg=0x0300  val=0x000001

# إزالة الـ module
sudo rmmod regmap_write_tracer
```

---

### ملاحظات أمان

- **الـ kprobe مش بيوقف** تنفيذ `regmap_write` — بس بيضيف observation point.
- **الـ return 0** من الـ pre_handler يعني "استمر طبيعي"، لو رجّعنا 1 كنا هنتخطى الدالة الأصلية (ده ممكن يكسر الـ drivers).
- **CONFIG_KPROBES** لازم يكون `y` أو `m` في الـ kernel config.
