## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي له الملف

الـ `spi-mem.c` جزء من **SPI subsystem** في Linux kernel، وبالتحديد هو الـ **SPI Memory abstraction layer** — طبقة وسيطة داخل `drivers/spi/` اتضافت سنة 2018 عشان تحل مشكلة حقيقية.

---

### القصة من الأول

تخيل عندك **SPI Flash** — شريحة ذاكرة صغيرة متوصلة بالـ CPU عن طريق بروتوكول SPI. زي مثلاً شريحة NOR Flash في راوتر أو NAND Flash في تليفون. الـ CPU بيتكلم معاها بلغة بسيطة: ابعت **opcode** (أمر)، ابعت **address** (عنوان)، اعمل **dummy cycles** (انتظار)، واجري **data** (بيانات). ده اللي بنسميه **SPI memory operation**.

المشكلة اللي كانت موجودة قبل `spi-mem.c`:

```
┌─────────────────────────────────────┐
│  SPI NOR driver (mtd/spi-nor)       │
│  SPI NAND driver (mtd/nand/spi)     │
│          ↓                          │
│  كل driver بيبني الـ spi_transfer   │
│  بإيده بطريقة مختلفة               │
└──────────────┬──────────────────────┘
               ↓
    ┌─────────────────────┐
    │   SPI core (spi.c)  │
    └─────────────────────┘
```

يعني كل driver للـ flash بيبني `spi_message` و`spi_transfer` بنفسه. لو المتحكم (controller) عنده hardware خاص بيعمل QSPI بسرعة — مافيش طريقة تقوله "استخدم الـ hardware path السريع"، لأن الـ driver بيتكلم بمستوى منخفض جداً.

---

### الحل: طبقة `spi-mem`

**الـ `spi-mem.c`** عمل abstraction layer واضح:

```
┌────────────────────────────────────────────┐
│  MTD SPI-NOR driver  │  MTD SPI-NAND driver│
│  (drivers/mtd/spi-nor) (drivers/mtd/nand/spi)│
└──────────────────────┬─────────────────────┘
                       ↓
         ┌─────────────────────────┐
         │   spi-mem layer         │  ← الملف ده
         │  (drivers/spi/spi-mem.c)│
         └───────────┬─────────────┘
                     ↓
     ┌───────────────────────────────────┐
     │  SPI Controller Driver            │
     │  (مثلاً spi-cadence-quadspi.c)    │
     │  ↓ إما mem_ops->exec_op()         │
     │  ↓ أو fallback لـ spi_sync()      │
     └───────────────────────────────────┘
```

الآن كل **SPI memory operation** اتعرّفت بشكل موحد في `struct spi_mem_op`:

```c
struct spi_mem_op {
    struct { u8 nbytes; u8 buswidth; u16 opcode; u8 dtr; } cmd;    // الأمر
    struct { u8 nbytes; u8 buswidth; u64 val;    u8 dtr; } addr;   // العنوان
    struct { u8 nbytes; u8 buswidth;             u8 dtr; } dummy;  // dummy cycles
    struct { u8 buswidth; enum dir; unsigned int nbytes; void *buf; u8 dtr; } data; // البيانات
    unsigned int max_freq; // أقصى تردد للعملية دي
};
```

---

### ليه ده مفيد؟ مثال عملي

تخيل عندك QSPI controller زي **Cadence QSPI** أو **NXP FlexSPI** — عندهم hardware يقدر يعمل "direct mapping": يحجز منطقة من الـ CPU address space مباشرةً mapped على الـ flash. يعني `memcpy` عادية بتتحول لـ QSPI read تلقائياً عبر الـ hardware!

بدون `spi-mem`:
- مستحيل يستخدم الـ flash driver هذه الميزة بطريقة portable.

مع `spi-mem`:
- الـ controller بيسجّل `mem_ops->dirmap_create()` و`mem_ops->dirmap_read()`.
- الـ `spi-mem.c` بيوفر `spi_mem_dirmap_create()` للـ flash driver.
- لو المتحكم ماعندوش، الـ layer بتعمل **fallback تلقائي** لـ `spi_mem_exec_op()`.

---

### الأهداف التفصيلية للملف

| الهدف | التفاصيل |
|-------|----------|
| **Unified operation model** | `struct spi_mem_op` بيوصف أي عملية flash بشكل موحد |
| **Capability checking** | `spi_mem_supports_op()` بتتحقق إن المتحكم والوضع السليم متوافقَين |
| **Optimized execution** | `spi_mem_exec_op()` بيجرب الـ `mem_ops->exec_op()` الخاص بالمتحكم أولاً، ولو مش موجود بيعمل fallback لـ `spi_sync()` |
| **Direct mapping** | `spi_mem_dirmap_*()` بتوفر memory-mapped access للـ flash عبر الـ hardware لو متاح |
| **Status polling** | `spi_mem_poll_status()` لانتظار انتهاء عمليات الـ flash (مثل write/erase) |
| **Driver registration** | `spi_mem_driver_register/unregister()` عشان الـ flash drivers تتسجل كـ `spi_mem_driver` |
| **DTR/Octal support** | دعم كامل لـ Double Transfer Rate وـ Octal SPI (8 lines) |
| **DMA helpers** | `spi_controller_dma_map/unmap_mem_op_data()` للمتحكمات اللي بتعمل DMA على الـ op data |
| **Statistics** | `spi_mem_add_op_stats()` بيسجّل إحصائيات لكل عملية |

---

### مستويات الـ Bus Width

الـ `spi-mem.c` بيدعم:

```
Standard SPI:  1 wire  (MOSI/MISO)
Dual SPI:      2 wires (x2)
Quad SPI:      4 wires (x4) ← الشائع في الـ NOR Flash الحديث
Octal SPI:     8 wires (x8) ← الأحدث والأسرع
```

وكل segment في الـ op (cmd, addr, dummy, data) ممكن يكون على عدد wires مختلف — مثلاً الـ command على wire واحد والـ data على 4 wires.

---

### الـ Direct Mapping بالتفصيل

```
Normal exec_op flow:
  Flash Driver → spi_mem_exec_op() → SPI message → SPI controller → Flash

Direct mapping flow:
  Flash Driver → spi_mem_dirmap_create() → Controller maps window
                ↓
  Flash Driver → spi_mem_dirmap_read(offset, len)
                ↓
  Controller reads via hardware-accelerated path (no CPU intervention per-byte)
```

لو المتحكم مش بيدعم الـ direct mapping، الـ `spi-mem.c` بيحط `desc->nodirmap = true` ويعمل fallback لـ `spi_mem_exec_op()` تلقائياً — شفافية كاملة للـ flash driver.

---

### الملفات المترابطة

#### الـ Core والـ Headers

| الملف | الدور |
|-------|-------|
| `drivers/spi/spi-mem.c` | **الملف ده** — طبقة الـ SPI memory abstraction |
| `include/linux/spi/spi-mem.h` | تعريفات `spi_mem_op`, `spi_mem_driver`, `spi_controller_mem_ops`, `spi_mem_dirmap_desc` |
| `include/linux/spi/spi.h` | تعريف `spi_controller`, `spi_device`, `spi_transfer`, `spi_message` |
| `drivers/spi/spi.c` | الـ SPI core نفسه — `spi_sync()`, `spi_flush_queue()`, إدارة الـ bus |
| `drivers/spi/internals.h` | functions داخلية بين `spi.c` و`spi-mem.c` |
| `include/trace/events/spi-mem.h` | tracepoints لتتبع بداية ونهاية كل operation |

#### الـ Flash Drivers اللي بتستخدم `spi-mem`

| الملف | الدور |
|-------|-------|
| `drivers/mtd/spi-nor/core.c` | SPI NOR Flash driver (Winbond, Micron, Spansion...) |
| `drivers/mtd/nand/spi/core.c` | SPI NAND Flash driver |

#### أمثلة على Controllers بتدعم `mem_ops`

| الملف | الشريحة |
|-------|---------|
| `drivers/spi/spi-cadence-quadspi.c` | Cadence QSPI |
| `drivers/spi/spi-nxp-fspi.c` | NXP FlexSPI |
| `drivers/spi/spi-aspeed-smc.c` | ASPEED SMC QSPI |
| `drivers/spi/spi-hisi-sfc-v3xx.c` | HiSilicon SFC |
| `drivers/spi/atmel-quadspi.c` | Atmel QSPI |
| `drivers/spi/spi-fsl-qspi.c` | Freescale QSPI |
| `drivers/spi/spi-mtk-nor.c` | MediaTek NOR controller |
## Phase 2: شرح الـ SPI-MEM Framework

---

### المشكلة — ليه الـ SPI-MEM موجود أصلاً؟

الـ **SPI subsystem** الأصلي في الـ kernel صُمِّم للـ generic SPI transfers — يعني إنت بتقوله: "ابعت الـ bytes دي على الـ bus، خد الـ bytes دي". الكلام ده كافي لـ sensors، displays، وغيرها من الأجهزة البسيطة.

لكن في الـ **SPI memory devices** — زي SPI NOR flash (W25Q128)، SPI NAND، و Octal SPI RAM — الوضع مختلف جداً. كل عملية عندها **بروتوكول محدد جداً**:

```
[ OPCODE 1 byte ] [ ADDRESS 3-4 bytes ] [ DUMMY CYCLES N bytes ] [ DATA N bytes ]
      1 wire             1 or 4 wires          1 or 4 wires          1/2/4/8 wires
```

المشاكل اللي كانت موجودة قبل الـ SPI-MEM:

1. **كل driver كان بيعمل الـ protocol من الصفر** — كل QSPI flash driver بيبني الـ `spi_transfer` structures بإيده، وبيحط الـ opcode والـ address والـ dummy bytes بنفسه. كود متكرر في كل driver.

2. **الـ controllers المتخصصة (QSPI controllers) اتعورت** — فيه controllers بتفهم الـ "flash command protocol" مباشرة وبتنفذه بكفاءة عالية، لكن ما فيش طريقة موحدة تقولهالها "ده command بـ address بـ 4 lines".

3. **الـ direct mapping مش موجود** — بعض الـ QSPI controllers بتعمل **memory-mapped access** للـ flash (زي إنك بتقرأ من RAM عادي). ما فيش إطار يدعم ده.

4. **الـ DTR (Double Transfer Rate) مش مدعوم** — الـ Octal DTR (8D-8D-8D) بيحتاج تعبير structured، مش مجرد bytes.

**الحل:** الـ SPI-MEM framework — طبقة abstraction فوق الـ SPI core، بتعبر عن عمليات الـ flash memory بـ **structured operation descriptor** بدل raw transfers.

---

### الحل — الـ Approach اللي اتخده الـ Kernel

الـ kernel أدخل مفهوم الـ **`spi_mem_op`** — struct واحد بيوصف أي عملية على SPI memory بشكل كامل:

```c
struct spi_mem_op {
    struct { u8 nbytes; u8 buswidth; u8 dtr; u16 opcode; } cmd;    // الـ command byte(s)
    struct { u8 nbytes; u8 buswidth; u8 dtr; u64 val;   } addr;   // الـ address
    struct { u8 nbytes; u8 buswidth; u8 dtr;            } dummy;  // dummy clocks
    struct { u8 buswidth; u8 dtr; u8 ecc; u8 swap16;
             enum spi_mem_data_dir dir; unsigned int nbytes;
             union { void *in; const void *out; } buf;  } data;   // الـ payload
    unsigned int max_freq;
};
```

الفكرة الجوهرية: **بدل ما تصف "كيف تبعت الـ bytes"، بتصف "إيه العملية اللي عايز تعملها".**

الـ framework بعد كده يقرر:
- لو الـ controller عنده `mem_ops->exec_op` — يبعته ليه مباشرة (الـ controller بيتعامل معاه native).
- لو مفيش — يترجمه لـ `spi_message` اعتيادي ويبعته عن طريق الـ generic SPI layer.

ده الـ **graceful degradation pattern** — نفس الـ driver بيشتغل على أي controller.

---

### الـ Real-World Analogy — أعمق من السطح

تخيل **مطعم فيه menu وطباخين بمستويات مختلفة**:

| جزء في الـ Kernel | التمثيل في المطعم |
|---|---|
| **`spi_mem_op`** | طلب من الـ menu مكتوب فيه: "بيتزا، حجم وسط، بدون بصل، خبز كريسبي" |
| **`spi_mem_exec_op()`** | الـ waiter اللي بياخد الطلب |
| **`spi_controller_mem_ops->exec_op()`** | طباخ محترف فاهم الـ menu ومتخصص في البيتزا (QSPI controller) |
| **الـ fallback لـ `spi_sync()`** | طباخ عادي بيقرأ الـ recipe خطوة خطوة (generic SPI) |
| **`spi_mem_supports_op()`** | السؤال "هل الطباخ ده بيعمل dish بـ 4 مكونات مرة واحدة؟" |
| **`spi_mem_dirmap_create()`** | حجز table دايم في المطعم — مش محتاج تطلب كل مرة |
| **`nodirmap` fallback** | لو مفيش table محجوز، نفس الـ menu لكن بإجراء عادي |
| **`spi_mem_driver`** | الـ customer اللي بيتعامل مع الـ menu بس، مش محتاج يعرف إيه اللي في المطبخ |

**الـ mapping الأعمق:**

- لما طباخ متخصص (QSPI controller) موجود، هو بيعمل الـ dish دفعة واحدة بكفاءة عالية.
- لو الطباخ المتخصص مش موجود أو مش عارف يعمل الـ dish ده (بيرجع `-ENOTSUPP`)، الـ framework بيحوله لـ generic recipe ويبعته للطباخ العادي.
- الـ customer (مثلاً `spi-nor` driver) مش عارف ومش محتاج يعرف نوع الطباخ — بيكلم الـ menu بس.

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Consumer Drivers                          │
│   ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐  │
│   │  spi-nor.c   │  │  spinand.c   │  │  spi-ram driver │  │
│   │ (MTD layer)  │  │ (NAND layer) │  │                 │  │
│   └──────┬───────┘  └──────┬───────┘  └────────┬────────┘  │
│          │  spi_mem_exec_op() / spi_mem_dirmap_*()          │
└──────────┼──────────────────┼───────────────────┼───────────┘
           │                  │                   │
┌──────────▼──────────────────▼───────────────────▼───────────┐
│                   SPI-MEM Framework (spi-mem.c)              │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  spi_mem_exec_op()                                  │    │
│  │   1. validate op (spi_mem_check_op)                 │    │
│  │   2. check support (spi_mem_supports_op)            │    │
│  │   3a. ctlr->mem_ops->exec_op()  ──→  fast path      │    │
│  │   3b. spi_sync(spi_message)     ──→  fallback path  │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  spi_mem_dirmap_{create,read,write,destroy}()       │    │
│  │   ↳ ctlr->mem_ops->dirmap_*() or exec_op fallback  │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  spi_mem_poll_status()                              │    │
│  │   ↳ ctlr->mem_ops->poll_status() or SW polling     │    │
│  └─────────────────────────────────────────────────────┘    │
└──────────────────────────┬──────────────────────────────────┘
                           │
           ┌───────────────▼───────────────┐
           │      SPI Core (spi.c)         │
           │  spi_sync() / spi_async()     │
           └───────────────┬───────────────┘
                           │
     ┌─────────────────────┼─────────────────────┐
     ▼                     ▼                     ▼
┌─────────┐         ┌─────────────┐       ┌──────────────┐
│Generic  │         │QSPI ctrl    │       │Octal SPI ctrl│
│SPI ctrl │         │(mem_ops +   │       │(DTR + dirmap │
│(no      │         │ exec_op)    │       │ support)     │
│mem_ops) │         └─────────────┘       └──────────────┘
└─────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ **central idea** هو الـ **`spi_mem_op` struct** كـ **operation descriptor**.

بدل ما تبني `spi_transfer` chain بإيدك وتحط الـ opcode في أول byte والـ address في التاني، بتقول للـ framework:

```c
/* مثال: قراءة من SPI NOR flash بـ Quad mode */
struct spi_mem_op op = SPI_MEM_OP(
    SPI_MEM_OP_CMD(0xEB, 1),              /* FAST READ QUAD I/O opcode, 1 wire */
    SPI_MEM_OP_ADDR(3, addr, 4),          /* 3-byte address, 4 wires */
    SPI_MEM_OP_DUMMY(3, 4),              /* 3 dummy bytes, 4 wires */
    SPI_MEM_OP_DATA_IN(len, buf, 4)      /* data in, 4 wires */
);
spi_mem_exec_op(mem, &op);
```

الـ framework بعد كده:
1. يتأكد إن الـ op valid (buswidths صح، buffers DMA-able)
2. يسأل الـ controller: هل تدعم الـ op دي؟ (`supports_op`)
3. ينفذها بأفضل طريقة متاحة

---

### الـ Struct Relationships — إزاي الـ Structs ببعض

```
struct spi_mem_driver          struct spi_controller
  ├── spidrv (spi_driver)        ├── mem_ops ──────────────┐
  ├── probe(spi_mem*)            ├── mem_caps              │
  ├── remove(spi_mem*)           ├── dma_tx                │
  └── shutdown(spi_mem*)         └── dma_rx                │
                                                           ▼
struct spi_mem                        struct spi_controller_mem_ops
  ├── spi ──→ struct spi_device          ├── adjust_op_size()
  ├── drvpriv (driver private)           ├── supports_op()
  └── name                              ├── exec_op()
                                        ├── get_name()
struct spi_mem_op                       ├── dirmap_create()
  ├── cmd                               ├── dirmap_destroy()
  │    ├── nbytes (1 or 2)              ├── dirmap_read()
  │    ├── buswidth (1/2/4/8)           ├── dirmap_write()
  │    ├── dtr (bool)                   └── poll_status()
  │    └── opcode
  ├── addr                         struct spi_controller_mem_caps
  │    ├── nbytes (0-8)               ├── dtr
  │    ├── buswidth                   ├── ecc
  │    ├── dtr                        ├── swap16
  │    └── val (u64)                  └── per_op_freq
  ├── dummy
  │    ├── nbytes
  │    ├── buswidth
  │    └── dtr
  └── data
       ├── buswidth
       ├── dtr / ecc / swap16
       ├── dir (IN/OUT/NO_DATA)
       ├── nbytes
       └── buf (union in/out)

struct spi_mem_dirmap_desc
  ├── mem ──→ struct spi_mem
  ├── info
  │    ├── op_tmpl (spi_mem_op)   ← template بيتكوبي لكل read/write
  │    ├── offset (u64)
  │    └── length (u64)
  ├── nodirmap (bool fallback flag)
  └── priv (controller-specific)
```

---

### الـ `spi_mem_op` Phases — أناتومي عملية Flash

أي عملية على SPI flash بتمر بـ 4 phases:

```
CS# LOW
 │
 ▼
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 1: CMD   │  PHASE 2: ADDR  │  PHASE 3: DUMMY │  PHASE 4: DATA │
│  1/2/4/8 wires  │  1/2/4/8 wires  │  clocks filler  │  1/2/4/8 wires │
│  STR or DTR     │  STR or DTR     │  STR or DTR     │  IN or OUT     │
└─────────────────────────────────────────────────────────────────┘
 │
 ▼
CS# HIGH
```

مثال حقيقي — **Micron MT25QL512 Quad Fast Read (0xEB)**:

```
Phase   | Wires | DTR | Content
--------|-------|-----|--------
CMD     |   1   |  No | 0xEB
ADDR    |   4   |  No | 0x001000 (3 bytes, MSB first)
DUMMY   |   4   |  No | 0xFF 0xFF 0xFF (3 bytes, جهاز بيتجاهلهم)
DATA    |   4   |  No | البيانات المقروءة
```

---

### الـ Execution Flow التفصيلي

```
spi_mem_exec_op(mem, op)
        │
        ├─ spi_mem_adjust_op_freq()     ← حط max_freq صح
        │
        ├─ spi_mem_check_op()           ← validate: buswidths، nbytes، DMA-able buffers
        │
        ├─ spi_mem_internal_supports_op()
        │     ├─ ctlr->mem_ops->supports_op()  [إذا موجود]
        │     └─ spi_mem_default_supports_op() [fallback]
        │           ├─ DTR check (controller capability)
        │           ├─ ECC check
        │           ├─ per_op_freq check
        │           └─ buswidth check (mode flags)
        │
        ├─ [Fast Path] ctlr->mem_ops->exec_op() موجود ومش cs-gpio؟
        │     ├─ spi_mem_access_start()         ← flush queue + lock mutexes
        │     ├─ trace_spi_mem_start_op()
        │     ├─ ctlr->mem_ops->exec_op()       ← controller native execution
        │     ├─ trace_spi_mem_stop_op()
        │     ├─ spi_mem_access_end()           ← unlock
        │     │
        │     └─ لو رجع -ENOTSUPP أو -EOPNOTSUPP:
        │           ↓ fall through لـ generic path
        │
        └─ [Generic Path] بناء spi_message من الـ op
              ├─ kzalloc(cmd + addr + dummy bytes) ← DMA-able buffer
              ├─ xfers[0]: cmd opcode
              ├─ xfers[1]: address bytes (big-endian)
              ├─ xfers[2]: dummy bytes (0xFF)
              ├─ xfers[3]: data (rx_buf or tx_buf)
              └─ spi_sync(mem->spi, &msg)
```

---

### الـ Direct Mapping (dirmap) — الـ Feature المتقدمة

الـ **direct mapping** بتخلي الـ flash يبان زي memory عادية للـ CPU — إنت بتعمل `read(address)` وبيرجع لك البيانات بدون ما تبعت command.

**إزاي بيشتغل:**

```c
/* إنشاء الـ mapping */
struct spi_mem_dirmap_info info = {
    .op_tmpl = SPI_MEM_OP(...),    /* template للـ read command */
    .offset = 0,
    .length = flash_size,
};
desc = spi_mem_dirmap_create(mem, &info);

/* قراءة */
spi_mem_dirmap_read(desc, offset, len, buf);

/* تدمير */
spi_mem_dirmap_destroy(desc);
```

**الـ fallback الذكي في `spi_mem_dirmap_create()`:**

```c
if (ctlr->mem_ops && ctlr->mem_ops->dirmap_create) {
    ret = ctlr->mem_ops->dirmap_create(desc);  /* hardware mapping */
}

if (ret) {
    desc->nodirmap = true;   /* الـ flag ده بيفرق في read/write */
    /* verify إن الـ op_tmpl مدعوم على الأقل */
    if (!spi_mem_supports_op(desc->mem, &desc->info.op_tmpl))
        ret = -EOPNOTSUPP;
}
```

لما `nodirmap = true`، الـ `spi_mem_dirmap_read()` بتنادي `spi_mem_no_dirmap_read()` اللي بتعمل:

```c
static ssize_t spi_mem_no_dirmap_read(struct spi_mem_dirmap_desc *desc,
                                       u64 offs, size_t len, void *buf)
{
    struct spi_mem_op op = desc->info.op_tmpl;  /* كوبي من الـ template */
    op.addr.val = desc->info.offset + offs;      /* حساب الـ address الفعلي */
    op.data.buf.in = buf;
    op.data.nbytes = len;
    spi_mem_adjust_op_size(desc->mem, &op);      /* تصغير لو محتاج */
    return spi_mem_exec_op(desc->mem, &op);
}
```

---

### الـ `spi_mem_driver` — طبقة الـ Wrapping

الـ `spi_mem_driver` هو wrapper رفيع فوق `spi_driver`:

```c
struct spi_mem_driver {
    struct spi_driver spidrv;   /* يورث من هنا */
    int (*probe)(struct spi_mem *mem);
    int (*remove)(struct spi_mem *mem);
    void (*shutdown)(struct spi_mem *mem);
};
```

لما `spi_mem_driver_register_with_owner()` بتتنادى:

```c
memdrv->spidrv.probe    = spi_mem_probe;    /* framework's probe */
memdrv->spidrv.remove   = spi_mem_remove;
memdrv->spidrv.shutdown = spi_mem_shutdown;
__spi_register_driver(owner, &memdrv->spidrv);
```

الـ `spi_mem_probe()` هو اللي بيعمل الـ `spi_mem` struct وبيديه لـ `memdrv->probe(mem)`. يعني الـ driver بيتعامل مع `spi_mem` بدل `spi_device` — abstraction أنظف.

---

### الـ Framework يمتلك إيه vs. بيفوض إيه للـ Drivers

| المسؤولية | ملكية الـ Framework | يُفوَّض للـ Driver |
|---|---|---|
| تعريف شكل الـ operation | `spi_mem_op` struct | — |
| التحقق من صحة الـ operation | `spi_mem_check_op()` | — |
| الـ fallback لـ generic SPI | `spi_mem_exec_op()` generic path | — |
| تنفيذ الـ operation بكفاءة | — | `mem_ops->exec_op()` |
| دعم الـ direct mapping (HW) | — | `mem_ops->dirmap_*()` |
| إيه capabilities الـ controller | — | `mem_caps` struct |
| poll status في الـ HW | — | `mem_ops->poll_status()` |
| DMA mapping helpers | `spi_controller_dma_map_mem_op_data()` | — |
| اسم الـ device | default: `dev_name()` | `mem_ops->get_name()` |
| تحديد الـ operation duration | `spi_mem_calc_op_duration()` | — |
| الـ statistics tracking | `spi_mem_add_op_stats()` | — |
| الـ bus locking أثناء الـ op | `spi_mem_access_start/end()` | — |

---

### ملاحظة على الـ DTR و Octal SPI

الـ **DTR (Double Transfer Rate)** — يعني بيبعت data على كل edge من الـ clock (rising + falling)، فبيضاعف الـ throughput. الـ **8D-8D-8D** يعني Octal (8 wires) + DTR في الـ 3 phases.

الـ framework بيفرض قيود خاصة على الـ 8D-8D-8D في `spi_mem_default_supports_op()`:

```c
if (op->cmd.dtr && op->cmd.buswidth == 8) {
    if (op->cmd.nbytes != 2)        /* cmd لازم يكون 2 bytes في DTR */
        return false;
    if ((op->addr.nbytes % 2) || (op->dummy.nbytes % 2) || (op->data.nbytes % 2))
        return false;               /* كل lengths لازم even في DTR */
}
```

ده لأن في DTR، كل clock cycle بيبعت 2 bits على كل wire، فـ odd number of bytes مش ممكن ينقسم على cycles بشكل نظيف.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. Flags, Enums, Config Options — Cheatsheet

#### `enum spi_mem_data_dir` — اتجاه نقل البيانات

| القيمة | المعنى |
|--------|--------|
| `SPI_MEM_NO_DATA` | مفيش بيانات اتبعت/استُقبلت |
| `SPI_MEM_DATA_IN` | البيانات جاية من الـ flash للـ CPU (قراءة) |
| `SPI_MEM_DATA_OUT` | البيانات رايحة من الـ CPU للـ flash (كتابة) |

#### Buswidth Flags — عدد خطوط الـ IO

| قيمة `buswidth` | الـ Mode | الـ SPI Mode Flags المقابلة |
|-----------------|----------|--------------------------|
| 1 | Single SPI | دايمًا مدعوم |
| 2 | Dual SPI | `SPI_TX_DUAL` / `SPI_RX_DUAL` |
| 4 | Quad SPI | `SPI_TX_QUAD` / `SPI_RX_QUAD` |
| 8 | Octal SPI | `SPI_TX_OCTAL` / `SPI_RX_OCTAL` |

#### DTR / ECC / swap16 Bits — داخل حقول `spi_mem_op`

| الـ Bit | موجود في | المعنى |
|---------|----------|--------|
| `dtr` | `cmd`, `addr`, `dummy`, `data` | Double Transfer Rate — كل edge بيرسل بيانات |
| `ecc` | `data` | يطلب Error Correction من الـ controller |
| `swap16` | `data` | يعكس ترتيب الـ bytes في وضع Octal DTR |

#### `spi_controller_mem_caps` — قدرات الـ Controller

| الحقل | المعنى |
|-------|--------|
| `dtr` | يدعم DTR operations |
| `ecc` | يدعم ECC |
| `swap16` | يدعم swap bytes في Octal DTR |
| `per_op_freq` | يدعم تغيير التردد per-operation |

#### الـ Macro `spi_mem_controller_is_capable`

```c
#define spi_mem_controller_is_capable(ctlr, cap) \
    ((ctlr)->mem_caps && (ctlr)->mem_caps->cap)
```
بيتحقق إن الـ `mem_caps` مش NULL قبل ما يقرأ الـ capability.

#### Config Option مهم

| الـ Option | الأثر |
|-----------|-------|
| `CONFIG_SPI_MEM` | لو مش enabled، كل الـ functions بتبقى stubs بترجع `-ENOTSUPP` |

---

### 1. الـ Structs المهمة

#### `struct spi_mem_op`

**الغرض:** وصف كامل لعملية SPI memory واحدة — الـ opcode + العنوان + الـ dummy bytes + البيانات، كل حاجة في struct واحدة.

```c
struct spi_mem_op {
    struct {
        u8  nbytes;    /* عدد بايتات الـ opcode (1 أو 2) */
        u8  buswidth;  /* عدد خطوط IO للـ command */
        u8  dtr : 1;   /* هل يُرسل بـ DTR؟ */
        u16 opcode;    /* الـ opcode نفسه */
    } cmd;

    struct {
        u8  nbytes;    /* عدد بايتات العنوان (0 → 8) */
        u8  buswidth;  /* عدد خطوط IO للعنوان */
        u8  dtr : 1;
        u64 val;       /* قيمة العنوان، MSB first */
    } addr;

    struct {
        u8  nbytes;    /* عدد dummy bytes */
        u8  buswidth;
        u8  dtr : 1;
    } dummy;

    struct {
        u8  buswidth;
        u8  dtr    : 1;
        u8  ecc    : 1;   /* طلب ECC */
        u8  swap16 : 1;   /* swap bytes في Octal DTR */
        enum spi_mem_data_dir dir;  /* IN أو OUT أو NO_DATA */
        unsigned int nbytes;
        union {
            void       *in;   /* buffer القراءة (DMA-able) */
            const void *out;  /* buffer الكتابة (DMA-able) */
        } buf;
    } data;

    unsigned int max_freq; /* أقصى تردد لهذه العملية (0 = no limit) */
};
```

**الارتباط بـ structs أخرى:** يُمرَّر كـ `const struct spi_mem_op *` لكل عملية. الـ `spi_mem_dirmap_info` بيحتوي نسخة template منه.

---

#### `struct spi_mem`

**الغرض:** يمثل جهاز الـ SPI memory (مثلاً NOR flash) من منظور الـ framework.

```c
struct spi_mem {
    struct spi_device *spi;   /* الـ SPI device التحتاني */
    void              *drvpriv; /* بيانات خاصة بالـ driver */
    const char        *name;  /* اسم الجهاز */
};
```

**الارتباط:**
- `spi` → `struct spi_device` → `controller` → `struct spi_controller`
- `drvpriv` يُملأ بواسطة الـ upper-layer driver عبر `spi_mem_set_drvdata()`

---

#### `struct spi_mem_dirmap_info`

**الغرض:** يصف نافذة Direct Mapping — جزء من ذاكرة الـ flash مربوط مباشرة بعنوان في الـ CPU.

```c
struct spi_mem_dirmap_info {
    struct spi_mem_op op_tmpl; /* template للعملية اللي بتوصل بيها */
    u64 offset;                /* الـ offset المطلق في الـ flash */
    u64 length;                /* حجم النافذة بالـ bytes */
};
```

---

#### `struct spi_mem_dirmap_desc`

**الغرض:** الـ descriptor الحي للـ direct mapping — يُنشأ بـ `spi_mem_dirmap_create()` ويُحذف بـ `spi_mem_dirmap_destroy()`.

```c
struct spi_mem_dirmap_desc {
    struct spi_mem          *mem;      /* الجهاز اللي الـ mapping تابعه */
    struct spi_mem_dirmap_info info;   /* المعلومات اللي اتمر بيها عند الإنشاء */
    unsigned int             nodirmap; /* 1 = fallback لـ exec_op */
    void                    *priv;     /* بيانات خاصة بالـ controller */
};
```

**الارتباط:** `mem` → `struct spi_mem` → `spi` → `controller` → `mem_ops` → `dirmap_*`

---

#### `struct spi_controller_mem_ops`

**الغرض:** جدول الـ virtual functions اللي الـ controller driver بيملأها عشان يدعم SPI memory operations بكفاءة.

```c
struct spi_controller_mem_ops {
    int     (*adjust_op_size)(struct spi_mem *, struct spi_mem_op *);
    bool    (*supports_op)(struct spi_mem *, const struct spi_mem_op *);
    int     (*exec_op)(struct spi_mem *, const struct spi_mem_op *);
    const char *(*get_name)(struct spi_mem *);
    int     (*dirmap_create)(struct spi_mem_dirmap_desc *);
    void    (*dirmap_destroy)(struct spi_mem_dirmap_desc *);
    ssize_t (*dirmap_read)(struct spi_mem_dirmap_desc *, u64, size_t, void *);
    ssize_t (*dirmap_write)(struct spi_mem_dirmap_desc *, u64, size_t, const void *);
    int     (*poll_status)(struct spi_mem *, const struct spi_mem_op *,
                           u16 mask, u16 match,
                           unsigned long initial_delay_us,
                           unsigned long polling_rate_us,
                           unsigned long timeout_ms);
};
```

كل الحقول **اختيارية** — لو مش موجودة، الـ core بيرجع للـ generic path.

---

#### `struct spi_controller_mem_caps`

**الغرض:** بيعلن قدرات الـ controller المتعلقة بالـ memory operations.

```c
struct spi_controller_mem_caps {
    bool dtr;       /* يدعم DTR */
    bool ecc;       /* يدعم ECC */
    bool swap16;    /* يدعم swap في Octal DTR */
    bool per_op_freq; /* يدعم per-op frequency */
};
```

---

#### `struct spi_mem_driver`

**الغرض:** wrapper رفيع فوق `spi_driver` عشان الـ upper-layer driver (زي SPINAND أو SPINOR) يشتغل بـ `spi_mem` مش `spi_device` مباشرة.

```c
struct spi_mem_driver {
    struct spi_driver spidrv;           /* يُرَّث منه */
    int  (*probe)(struct spi_mem *mem);
    int  (*remove)(struct spi_mem *mem);
    void (*shutdown)(struct spi_mem *mem);
};
```

**الارتباط:** الـ core بيعمل wrapping — `spi_mem_probe()` بتُنشئ `spi_mem` وبتستدعي `memdrv->probe(mem)`.

---

#### `struct spi_controller` (الأجزاء المهمة لـ spi-mem)

**الغرض:** يمثل الـ SPI controller hardware. من منظور `spi-mem.c` الحقول التالية هي الأهم:

| الحقل | النوع | الدور |
|-------|-------|-------|
| `mem_ops` | `const struct spi_controller_mem_ops *` | جدول الـ ops الخاص بالـ memory |
| `mem_caps` | `const struct spi_controller_mem_caps *` | قدرات الـ memory controller |
| `io_mutex` | `struct mutex` | يحمي الوصول الفيزيائي للـ bus |
| `bus_lock_mutex` | `struct mutex` | يحمي exclusive bus access |
| `auto_runtime_pm` | `bool` | الـ core يدير PM تلقائيًا |
| `pcpu_statistics` | `struct spi_statistics __percpu *` | إحصائيات الـ controller |
| `dma_tx` | `struct dma_chan *` | قناة DMA للإرسال |
| `dma_rx` | `struct dma_chan *` | قناة DMA للاستقبال |
| `min_speed_hz` | `u32` | أقل سرعة مدعومة |

---

### 2. مخططات علاقات الـ Structs

```
┌─────────────────────────────────────────────────────────────┐
│                     spi_mem_driver                          │
│  spidrv (spi_driver) ──────────────────────────────────┐   │
│  probe()  remove()  shutdown()                          │   │
└───────────────┬─────────────────────────────────────────┘   │
                │ wraps                                        │
                ▼                                             │
        ┌──────────────┐                                      │
        │   spi_mem    │◄─────────────────────────────────────┘
        │  *spi ───────┼──────────────────────────────────────┐
        │  *drvpriv    │                                       │
        │  *name       │                                       │
        └──────────────┘                                       │
                                                               ▼
                                                    ┌─────────────────┐
                                                    │   spi_device    │
                                                    │  *controller ───┼────────────────────┐
                                                    │  max_speed_hz   │                    │
                                                    │  mode (flags)   │                    │
                                                    │  *pcpu_stats    │                    │
                                                    └─────────────────┘                    │
                                                                                            ▼
                                                                               ┌────────────────────────┐
                                                                               │    spi_controller      │
                                                                               │  *mem_ops ─────────────┼──► spi_controller_mem_ops
                                                                               │  *mem_caps ────────────┼──► spi_controller_mem_caps
                                                                               │  io_mutex              │
                                                                               │  bus_lock_mutex        │
                                                                               │  auto_runtime_pm       │
                                                                               │  *dma_tx, *dma_rx      │
                                                                               │  *pcpu_statistics      │
                                                                               └────────────────────────┘


┌──────────────────────┐       ┌──────────────────────────┐
│ spi_mem_dirmap_desc  │       │   spi_mem_dirmap_info    │
│  *mem ───────────────┼──►    │    op_tmpl (spi_mem_op)  │
│  info ───────────────┼──────►│    offset                │
│  nodirmap            │       │    length                │
│  *priv               │       └──────────────────────────┘
└──────────────────────┘
        │
        └──► spi_mem ──► spi_device ──► spi_controller ──► mem_ops
```

---

### 3. Lifecycle Diagrams

#### دورة حياة `spi_mem_driver`

```
Driver Module Load
      │
      ▼
spi_mem_driver_register_with_owner(memdrv, owner)
      │
      ├── memdrv->spidrv.probe    = spi_mem_probe    [core wrapper]
      ├── memdrv->spidrv.remove   = spi_mem_remove
      ├── memdrv->spidrv.shutdown = spi_mem_shutdown
      │
      └── __spi_register_driver(owner, &memdrv->spidrv)
                │
                ▼
           SPI bus match → device found
                │
                ▼
         spi_mem_probe(spi_device)
                │
                ├── devm_kzalloc(spi_mem)
                ├── mem->spi = spi
                ├── mem->name = ctlr->mem_ops->get_name(mem)  [أو dev_name]
                ├── spi_set_drvdata(spi, mem)
                └── memdrv->probe(mem)          ← الـ driver الفعلي
                          │
                          ▼
                    [Device Active — يؤدي operations]
                          │
                          ▼
         spi_mem_remove(spi_device)
                │
                └── memdrv->remove(mem)

Module Unload
      │
      ▼
spi_mem_driver_unregister(memdrv)
      └── spi_unregister_driver(&memdrv->spidrv)
```

---

#### دورة حياة `spi_mem_dirmap_desc`

```
Driver يريد Direct Mapping
      │
      ▼
spi_mem_dirmap_create(mem, info)
      │
      ├── validate: addr.nbytes ∈ [1,8]
      ├── validate: data.dir ≠ SPI_MEM_NO_DATA
      ├── kzalloc(spi_mem_dirmap_desc)
      ├── desc->mem  = mem
      ├── desc->info = *info
      │
      ├── [HW path] ctlr->mem_ops->dirmap_create(desc)
      │       │
      │       ├── success → desc->priv مليان بـ HW resources
      │       └── fail    → desc->nodirmap = true (fallback)
      │
      └── return desc
              │
              ▼
    [استخدام عبر spi_mem_dirmap_read/write]
              │
              ▼
spi_mem_dirmap_destroy(desc)
      │
      ├── [HW path] ctlr->mem_ops->dirmap_destroy(desc)
      └── kfree(desc)

[devm variant] → devm_spi_mem_dirmap_create()
      └── يُربط بـ device lifetime → تلقائيًا يُحذف عند unbind
```

---

### 4. Call Flow Diagrams

#### `spi_mem_exec_op()` — المسار الكامل

```
flash driver يستدعي spi_mem_exec_op(mem, op)
      │
      ├── spi_mem_adjust_op_freq(mem, op)
      │       └── op->max_freq = min(op->max_freq, spi->max_speed_hz)
      │
      ├── spi_mem_check_op(op)              [تحقق: buswidth صح، buffers DMA-able]
      │
      ├── spi_mem_internal_supports_op(mem, op)
      │       ├── ctlr->mem_ops->supports_op(mem, op)   [لو موجودة]
      │       └── spi_mem_default_supports_op(mem, op)  [fallback]
      │               ├── تحقق DTR capabilities
      │               ├── تحقق ECC capabilities
      │               ├── تحقق max_freq vs min_speed_hz
      │               └── spi_mem_check_buswidth(mem, op)
      │
      ├── [HW path — لو mem_ops->exec_op موجودة وما فيش CS GPIO]
      │       │
      │       ├── spi_mem_access_start(mem)
      │       │       ├── spi_flush_queue(ctlr)          [flush pending transfers]
      │       │       ├── pm_runtime_resume_and_get()    [لو auto_runtime_pm]
      │       │       ├── mutex_lock(bus_lock_mutex)
      │       │       └── mutex_lock(io_mutex)
      │       │
      │       ├── trace_spi_mem_start_op(mem, op)
      │       ├── ctlr->mem_ops->exec_op(mem, op)       ← HW does the work
      │       ├── trace_spi_mem_stop_op(mem, op)
      │       │
      │       ├── spi_mem_access_end(mem)
      │       │       ├── mutex_unlock(io_mutex)
      │       │       ├── mutex_unlock(bus_lock_mutex)
      │       │       └── pm_runtime_put()
      │       │
      │       ├── spi_mem_add_op_stats(ctlr->pcpu_statistics, ...)
      │       ├── spi_mem_add_op_stats(spi->pcpu_statistics, ...)
      │       └── return ret   (إلا لو -ENOTSUPP → يكمل للـ generic path)
      │
      └── [Generic path — SPI core transfers]
              │
              ├── kzalloc(tmpbuf) [DMA-able buffer لـ CMD+ADDR+DUMMY]
              ├── spi_message_init(&msg)
              │
              ├── xfers[0]: CMD  (tx_buf=tmpbuf, tx_nbits=cmd.buswidth)
              ├── xfers[1]: ADDR (tx_buf=tmpbuf+1, MSB first)
              ├── xfers[2]: DUMMY (tx_buf=0xFF bytes)
              ├── xfers[3]: DATA (rx_buf أو tx_buf حسب الاتجاه)
              │
              ├── spi_sync(mem->spi, &msg)
              │       └── SPI core يرسل الكل كـ message واحدة
              │
              ├── kfree(tmpbuf)
              └── تحقق: msg.actual_length == totalxferlen
```

---

#### `spi_mem_dirmap_read()` — Call Flow

```
flash driver يستدعي spi_mem_dirmap_read(desc, offs, len, buf)
      │
      ├── تحقق: data.dir == SPI_MEM_DATA_IN
      │
      ├── [nodirmap == true] → fallback
      │       └── spi_mem_no_dirmap_read(desc, offs, len, buf)
      │               ├── op = desc->info.op_tmpl  [نسخة من الـ template]
      │               ├── op.addr.val = info.offset + offs
      │               ├── op.data.buf.in = buf
      │               ├── op.data.nbytes = len
      │               ├── spi_mem_adjust_op_size(mem, &op)
      │               └── spi_mem_exec_op(mem, &op)
      │
      └── [HW dirmap] ctlr->mem_ops->dirmap_read موجودة
              ├── spi_mem_access_start(mem)
              ├── ctlr->mem_ops->dirmap_read(desc, offs, len, buf)
              │       └── Controller يقرأ مباشرة من الـ mapped window
              └── spi_mem_access_end(mem)
```

---

#### `spi_mem_poll_status()` — Call Flow

```
flash driver يستدعي spi_mem_poll_status(mem, op, mask, match, ...)
      │
      ├── تحقق: data.nbytes ∈ [1,2] و data.dir == SPI_MEM_DATA_IN
      │
      ├── [HW poll] ctlr->mem_ops->poll_status موجودة وما فيش CS GPIO
      │       ├── spi_mem_access_start(mem)
      │       ├── ctlr->mem_ops->poll_status(...)
      │       │       └── Controller hardware ينتظر على مستوى الـ register
      │       └── spi_mem_access_end(mem)
      │
      └── [SW fallback — لو -EOPNOTSUPP]
              ├── udelay(initial_delay_us)    [أو usleep_range]
              └── read_poll_timeout(spi_mem_read_status, ...)
                      │
                      └── [loop] spi_mem_read_status(mem, op, &status)
                                  └── spi_mem_exec_op(mem, op)
                                  └── تحقق: (status & mask) == match
                                  └── لو صح أو timeout → انهِ
```

---

#### `spi_controller_dma_map_mem_op_data()` — Call Flow

```
Controller driver يستدعي spi_controller_dma_map_mem_op_data(ctlr, op, sgt)
      │
      ├── تحقق: op->data.nbytes > 0
      │
      ├── اختيار الـ DMA device:
      │       ├── DATA_OUT + dma_tx موجودة → dmadev = dma_tx->device->dev
      │       ├── DATA_IN  + dma_rx موجودة → dmadev = dma_rx->device->dev
      │       └── غير كده → dmadev = ctlr->dev.parent
      │
      └── spi_map_buf(ctlr, dmadev, sgt, op->data.buf.in, nbytes, direction)
              └── يبني scatter-gather table جاهز للـ DMA
```

---

### 5. Locking Strategy

#### الـ Locks المستخدمة في `spi-mem.c`

| الـ Lock | النوع | يحمي ماذا؟ |
|---------|-------|-----------|
| `ctlr->bus_lock_mutex` | `struct mutex` | exclusive access للـ SPI bus كله |
| `ctlr->io_mutex` | `struct mutex` | الوصول الفيزيائي للـ controller hardware |

#### ترتيب الـ Locking (Lock Ordering)

```
spi_mem_access_start():
    1. mutex_lock(&ctlr->bus_lock_mutex)   ← أولاً
    2. mutex_lock(&ctlr->io_mutex)         ← ثانياً

spi_mem_access_end():
    1. mutex_unlock(&ctlr->io_mutex)       ← أولاً (عكسي)
    2. mutex_unlock(&ctlr->bus_lock_mutex) ← ثانياً
```

**الـ `bus_lock_mutex` دايمًا قبل `io_mutex`** — لازم يتقيد بالترتيب ده في أي كود تاني عشان ما يحصلش deadlock.

#### ليه mutex مش spinlock؟

الـ SPI memory operations ممكن تاخد وقت طويل (microseconds → milliseconds)، وممكن تستدعي DMA وتنام. الـ `spinlock` مش مناسب هنا — الـ `mutex` هو الصح.

#### الـ Per-CPU Statistics — Lockless Design

```c
/* في spi_mem_add_op_stats() */
get_cpu();
stats = this_cpu_ptr(pcpu_stats);
u64_stats_update_begin(&stats->syncp);   /* seqcount-based protection */
    u64_stats_inc(&stats->messages);
    /* ... */
u64_stats_update_end(&stats->syncp);
put_cpu();
```

الـ statistics بتستخدم **per-CPU variables** + **seqcount** بدل mutex — zero contention على الـ fast path.

#### الـ `pm_runtime` و Locking

```
spi_mem_access_start():
    pm_runtime_resume_and_get(ctlr->dev.parent)  ← قبل الـ locks
        │
        └── يضمن إن الـ hardware شغال قبل ما نمسك أي lock
```

**الترتيب الكامل:**
```
1. spi_flush_queue()              [لا lock]
2. pm_runtime_resume_and_get()   [لا lock]
3. mutex_lock(bus_lock_mutex)
4. mutex_lock(io_mutex)
   ──── critical section ────
5. mutex_unlock(io_mutex)
6. mutex_unlock(bus_lock_mutex)
7. pm_runtime_put()              [لا lock]
```
## Phase 4: شرح الـ Functions

---

### ملخص — Cheatsheet لكل الـ Functions

#### Group 1: DMA Helpers (للـ Controller فقط)

| Function | Signature Summary | الغرض |
|---|---|---|
| `spi_controller_dma_map_mem_op_data` | `(ctlr, op, sgt) → int` | يعمل DMA map لـ buffer الـ op |
| `spi_controller_dma_unmap_mem_op_data` | `(ctlr, op, sgt) → void` | يلغي الـ DMA map بعد الانتهاء |

#### Group 2: Validation & Capability Check

| Function | Signature Summary | الغرض |
|---|---|---|
| `spi_check_buswidth_req` | `(mem, buswidth, tx) → int` | يتحقق إن الـ buswidth مدعوم |
| `spi_mem_check_buswidth` | `(mem, op) → bool` | يتحقق كل phases الـ op |
| `spi_mem_buswidth_is_valid` | `(buswidth) → bool` | يتحقق إن الـ buswidth power-of-2 وفي حدود |
| `spi_mem_check_op` | `(op) → int` | validates كامل الـ op struct |
| `spi_mem_internal_supports_op` | `(mem, op) → bool` | يسأل الـ controller أو يرجع default |
| `spi_mem_default_supports_op` | `(mem, op) → bool` | الـ default capability check |
| `spi_mem_supports_op` | `(mem, op) → bool` | الـ public API للـ drivers |

#### Group 3: Operation Execution

| Function | Signature Summary | الغرض |
|---|---|---|
| `spi_mem_access_start` | `(mem) → int` | يأخذ locks وينشط PM |
| `spi_mem_access_end` | `(mem) → void` | يرخي locks ويوقف PM |
| `spi_mem_add_op_stats` | `(pcpu_stats, op, ret) → void` | يحدّث per-CPU statistics |
| `spi_mem_exec_op` | `(mem, op) → int` | ينفّذ SPI memory operation |
| `spi_mem_poll_status` | `(mem, op, mask, match, ...) → int` | يستطلع status register |
| `spi_mem_read_status` | `(mem, op, status) → int` | helper يقرأ status لمرة واحدة |

#### Group 4: Size & Frequency Adjustment

| Function | Signature Summary | الغرض |
|---|---|---|
| `spi_mem_adjust_op_size` | `(mem, op) → int` | يضبط data.nbytes لحدود الـ controller |
| `spi_mem_adjust_op_freq` | `(mem, op) → void` | يضبط max_freq لحدود الـ SPI device |
| `spi_mem_calc_op_duration` | `(mem, op) → u64` | يحسب زمن الـ operation بالـ ns |

#### Group 5: Direct Mapping (dirmap)

| Function | Signature Summary | الغرض |
|---|---|---|
| `spi_mem_no_dirmap_read` | `(desc, offs, len, buf) → ssize_t` | fallback قراءة بدون HW dirmap |
| `spi_mem_no_dirmap_write` | `(desc, offs, len, buf) → ssize_t` | fallback كتابة بدون HW dirmap |
| `spi_mem_dirmap_create` | `(mem, info) → desc*` | ينشئ direct mapping descriptor |
| `spi_mem_dirmap_destroy` | `(desc) → void` | يدمّر الـ descriptor |
| `devm_spi_mem_dirmap_create` | `(dev, mem, info) → desc*` | devm نسخة من create |
| `devm_spi_mem_dirmap_destroy` | `(dev, desc) → void` | devm نسخة من destroy |
| `spi_mem_dirmap_read` | `(desc, offs, len, buf) → ssize_t` | يقرأ بيانات عبر الـ dirmap |
| `spi_mem_dirmap_write` | `(desc, offs, len, buf) → ssize_t` | يكتب بيانات عبر الـ dirmap |

#### Group 6: Driver Registration

| Function | Signature Summary | الغرض |
|---|---|---|
| `spi_mem_get_name` | `(mem) → const char*` | يرجع اسم الـ SPI mem device |
| `to_spi_mem_drv` | `(drv) → spi_mem_driver*` | container_of helper |
| `spi_mem_probe` | `(spi) → int` | internal probe يُنشئ spi_mem |
| `spi_mem_remove` | `(spi) → void` | internal remove |
| `spi_mem_shutdown` | `(spi) → void` | internal shutdown |
| `spi_mem_driver_register_with_owner` | `(memdrv, owner) → int` | يسجّل spi_mem_driver |
| `spi_mem_driver_unregister` | `(memdrv) → void` | يلغي تسجيل الـ driver |

---

### Group 1: DMA Helpers

الـ group ده بيوفر helper functions للـ SPI controller drivers اللي عايزة تعمل DMA على الـ data buffer الموجود جوه الـ `spi_mem_op`. مش للـ SPI device drivers — ده exclusively للـ controller layer.

---

#### `spi_controller_dma_map_mem_op_data`

```c
int spi_controller_dma_map_mem_op_data(struct spi_controller *ctlr,
                                       const struct spi_mem_op *op,
                                       struct sg_table *sgt);
```

بيعمل DMA map لـ buffer الـ `op->data.buf.{in,out}` عشان الـ DMA engine يقدر يوصله. بيحدد الـ DMA device المناسب بناءً على اتجاه النقل (TX channel أو RX channel أو parent device لو مفيش dedicated channel). بيبني الـ `sg_table` الجاهز للاستخدام.

**Parameters:**
- `ctlr` — الـ SPI controller المسؤول عن الـ DMA operation
- `op` — الـ memory operation اللي فيها الـ data buffer والـ direction
- `sgt` — uninitialized `sg_table` هيتملى بالـ scatter-gather entries

**Return:** `0` في النجاح، `EINVAL` لو `op->data.nbytes == 0` أو مفيش DMA device صالح.

**Key Details:**
- الـ caller لازم يضمن إن الـ buffer DMA-able قبل الاستدعاء
- لو `ctlr->dma_tx` أو `ctlr->dma_rx` موجود بيستخدم الـ device بتاعهم، غير كده بيستخدم `ctlr->dev.parent`
- داخلياً بيستدعي `spi_map_buf()` مع `DMA_TO_DEVICE` أو `DMA_FROM_DEVICE`
- يُستدعى من controller driver code فقط — مش من SPI peripheral drivers

---

#### `spi_controller_dma_unmap_mem_op_data`

```c
void spi_controller_dma_unmap_mem_op_data(struct spi_controller *ctlr,
                                          const struct spi_mem_op *op,
                                          struct sg_table *sgt);
```

يلغي الـ DMA mapping اللي عملته `spi_controller_dma_map_mem_op_data()` عشان الـ CPU يقدر يوصل الـ buffer تاني. لازم يُستدعى بعد ما الـ DMA operation تخلص.

**Parameters:**
- `ctlr` — نفس الـ controller اللي عمل الـ map
- `op` — نفس الـ operation اللي اتعملت map
- `sgt` — الـ `sg_table` اللي ملأتها `map` function

**Return:** void.

**Key Details:**
- بيرجع فوراً لو `op->data.nbytes == 0` — safe to call unconditionally
- بيستدعي `spi_unmap_buf()` داخلياً بنفس logic تحديد الـ DMA device

---

### Group 2: Validation & Capability Check

الـ group ده مسؤول عن التحقق الكامل من صحة الـ `spi_mem_op` قبل تنفيذها. فيه طبقات متعددة من الـ validation: من buswidth لـ DTR لـ ECC لـ frequency constraints.

---

#### `spi_check_buswidth_req` (static)

```c
static int spi_check_buswidth_req(struct spi_mem *mem, u8 buswidth, bool tx);
```

بيتحقق إن الـ SPI device (مش الـ controller) يدعم buswidth معين في اتجاه معين. بيقرأ `mem->spi->mode` flags.

**Parameters:**
- `mem` — الـ SPI memory device
- `buswidth` — العرض المطلوب: 1, 2, 4, أو 8 lanes
- `tx` — `true` لو الاتجاه TX، `false` لـ RX

**Return:** `0` لو مدعوم، `-ENOTSUPP` غير كده.

**Key Details:**
- `buswidth == 1` دايماً مقبول (standard SPI)
- `buswidth == 2` يتطلب `SPI_TX_DUAL | SPI_TX_QUAD | SPI_TX_OCTAL` للـ TX
- `buswidth == 4` يتطلب `SPI_TX_QUAD | SPI_TX_OCTAL`
- `buswidth == 8` يتطلب `SPI_TX_OCTAL` بالظبط

---

#### `spi_mem_check_buswidth` (static)

```c
static bool spi_mem_check_buswidth(struct spi_mem *mem,
                                   const struct spi_mem_op *op);
```

بيفحص كل phases الـ op (cmd, addr, dummy, data) إن كل buswidth مدعوم. بيستدعي `spi_check_buswidth_req` لكل phase موجودة.

**Parameters:**
- `mem` — الـ SPI memory device
- `op` — الـ operation الكاملة

**Return:** `true` لو كل الـ buswidths صح، `false` لو أي phase فشلت.

**Key Details:**
- الـ cmd دايماً TX
- الـ addr و dummy دايماً TX
- الـ data direction بيتحدد من `op->data.dir`

---

#### `spi_mem_buswidth_is_valid` (static)

```c
static bool spi_mem_buswidth_is_valid(u8 buswidth);
```

بيتحقق إن الـ buswidth value نفسها valid: لازم تكون power of 2 (أو 1) وأقل من أو تساوي 8. بيستخدم `hweight8()` للتحقق إن فيه set bit واحد بس.

**Return:** `true` لو valid، `false` غير كده.

---

#### `spi_mem_check_op` (static)

```c
static int spi_mem_check_op(const struct spi_mem_op *op);
```

الـ comprehensive validation للـ `spi_mem_op` struct بالكامل. بيتحقق من:
1. إن `cmd.buswidth` و `cmd.nbytes` مش zero
2. إن أي phase غير فاضية عندها `buswidth` != 0
3. إن كل الـ buswidth values نفسها valid (power-of-2, ≤ 8)
4. إن الـ data buffers مش على الـ stack (لازم DMA-able)

**Return:** `0` لو كله تمام، `-EINVAL` في أي حالة فشل.

**Key Details:**
- `WARN_ON_ONCE` للـ stack buffer detection — بيطبع kernel warning
- `object_is_on_stack()` بيستخدم `task_stack` للكشف عن الـ stack addresses
- ده الـ guard الرئيسي قبل أي تنفيذ فعلي

---

#### `spi_mem_default_supports_op`

```c
bool spi_mem_default_supports_op(struct spi_mem *mem,
                                 const struct spi_mem_op *op);
```

الـ default implementation للـ capability check لما الـ controller مش بيوفر `mem_ops->supports_op`. بيتحقق من DTR support، ECC support، frequency limits، وبيستدعي `spi_mem_check_buswidth`. **ده الـ exported function المستخدمة كـ fallback في كل الـ QSPI controllers.**

**Parameters:**
- `mem` — الـ SPI memory device
- `op` — الـ operation المراد فحصها

**Return:** `true` لو الـ controller يقدر ينفذ الـ op، `false` غير كده.

**Key Details:**
- لو أي phase فيها `dtr=true` → يتحقق من `spi_mem_controller_is_capable(ctlr, dtr)`
- لو `op->data.swap16` → يتحقق من `swap16` capability
- خاص بـ 8D-8D-8D: الـ cmd لازم 2 bytes، وكل الـ nbytes لازم even
- لو الـ op ليها `max_freq` أقل من `controller->min_speed_hz` → false
- لو الـ op ليها `max_freq` أقل من device speed والـ controller مش capable من `per_op_freq` → false
- يُستدعى من controller drivers كـ fallback أو من `spi_mem_internal_supports_op`

---

#### `spi_mem_internal_supports_op` (static)

```c
static bool spi_mem_internal_supports_op(struct spi_mem *mem,
                                         const struct spi_mem_op *op);
```

الـ dispatch layer: لو الـ controller عنده `mem_ops->supports_op` بيستدعيه، غير كده بيرجع لـ `spi_mem_default_supports_op`. ده الـ single call site اللي `spi_mem_exec_op` و `spi_mem_supports_op` بيستخدموه.

---

#### `spi_mem_supports_op`

```c
bool spi_mem_supports_op(struct spi_mem *mem, const struct spi_mem_op *op);
```

الـ **public API** اللي بيستخدمه الـ SPI memory drivers (مثلاً m25p80، spinand) عشان يسألوا هل الـ op دي مدعومة قبل ما يحاولوا ينفذوها. بيعمل `spi_mem_adjust_op_freq` أولاً ثم `spi_mem_check_op` ثم `spi_mem_internal_supports_op`.

**Return:** `true` لو مدعومة، `false` غير كده.

**Key Details:**
- يُستدعى من SPI memory drivers (mtd, spinand, etc.)
- الـ frequency adjustment بيحصل هنا قبل الـ check — مش بعده

---

### Group 3: Operation Execution

ده القلب الرئيسي للـ subsystem. مسؤول عن تنفيذ الـ operations الفعلية مع الـ locking الصحيح والـ PM management والـ stats.

---

#### `spi_mem_access_start` (static)

```c
static int spi_mem_access_start(struct spi_mem *mem);
```

بيجهز الـ bus للـ SPI memory operation: يفرغ الـ message queue، يفعّل الـ runtime PM لو مطلوب، يأخذ `bus_lock_mutex` ثم `io_mutex` بالترتيب ده بالظبط.

**Return:** `0` في النجاح، error من `pm_runtime_resume_and_get` في الفشل.

**Key Details:**
- `spi_flush_queue()` أولاً عشان يمنع preemption للـ regular SPI transfers
- الـ mutexes بالترتيب: `bus_lock_mutex` ← `io_mutex` (اتبع نفس الترتيب دايماً)
- لو `ctlr->auto_runtime_pm` → `pm_runtime_resume_and_get(ctlr->dev.parent)`
- يُستدعى دايماً قبل أي `ctlr->mem_ops->exec_op()` أو `dirmap_*`

---

#### `spi_mem_access_end` (static)

```c
static void spi_mem_access_end(struct spi_mem *mem);
```

يرخي الـ mutexes بالترتيب العكسي ويرجع الـ PM reference. mirror بالظبط لـ `spi_mem_access_start`.

**Key Details:**
- ترتيب الـ unlock: `io_mutex` ثم `bus_lock_mutex` (عكس الـ lock)
- `pm_runtime_put()` بدون sync — الـ device ممكن يدخل suspend تاني

---

#### `spi_mem_add_op_stats` (static)

```c
static void spi_mem_add_op_stats(struct spi_statistics __percpu *pcpu_stats,
                                 const struct spi_mem_op *op, int exec_op_ret);
```

بيضيف statistics للـ per-CPU counters. بيعتبر كل `spi_mem_op` equivalent لـ message واحدة وtransfer واحدة. بيحسب total bytes، transfer histogram، TX/RX bytes، وعدد الـ errors والـ timeouts.

**Parameters:**
- `pcpu_stats` — the per-CPU stats pointer (controller أو device)
- `op` — الـ executed operation
- `exec_op_ret` — return value من التنفيذ

**Key Details:**
- `get_cpu()` / `put_cpu()` لضمان affinity أثناء التحديث
- `u64_stats_update_begin/end()` للـ lockless read consistency
- `len = cmd.nbytes + addr.nbytes + dummy.nbytes + data.nbytes`
- الـ histogram index = `min(fls(len), SPI_STATISTICS_HISTO_SIZE) - 1`
- `-ETIMEDOUT` بيُحسب كـ timeout مش error (نفس سلوك `spi_transfer_one_message`)
- بيُستدعى مرتين: مرة لـ `ctlr->pcpu_statistics` ومرة لـ `mem->spi->pcpu_statistics`

---

#### `spi_mem_exec_op`

```c
int spi_mem_exec_op(struct spi_mem *mem, const struct spi_mem_op *op);
```

**الـ core function في كل الـ subsystem.** ينفذ SPI memory operation كاملة. عنده مساران:

**المسار الأول (optimized):** لو الـ controller عنده `mem_ops->exec_op` ومفيش CS GPIO:
```
adjust_freq → check_op → check_supports
  → access_start → trace_start
  → ctlr->mem_ops->exec_op(mem, op)
  → trace_stop → access_end
  → add_stats → return
```

**المسار الثاني (generic SPI fallback):** لو مفيش `mem_ops->exec_op` أو رجع `-ENOTSUPP`/`-EOPNOTSUPP`:
```
alloc tmpbuf (DMA-able, GFP_DMA)
build spi_message with up to 4 transfers:
  [0] cmd opcode bytes
  [1] address bytes (MSB first)
  [2] dummy bytes (0xFF)
  [3] data rx/tx buffer
spi_sync() → check actual_length → free tmpbuf
```

**Parameters:**
- `mem` — الـ SPI memory device
- `op` — الـ operation المراد تنفيذها

**Return:** `0` في النجاح، negative error code في الفشل.

**Key Details:**
- الـ `tmpbuf` بيتخصص بـ `GFP_KERNEL | GFP_DMA` لضمان DMA-ability
- الـ address بتتبعث MSB first بـ explicit byte reversal loop
- الـ dummy bytes بتتملى بـ `0xFF` (standard practice)
- لو `msg.actual_length != totalxferlen` بيرجع `-EIO` — partial transfer detection
- الـ trace points `trace_spi_mem_start_op` / `trace_spi_mem_stop_op` بيتفعلوا فقط في المسار الأول
- الـ stats بتتحدث فقط في المسار الأول — المسار الثاني بيستخدم regular SPI stats

**Pseudocode Flow:**
```c
spi_mem_adjust_op_freq(mem, op);
spi_mem_check_op(op);          // validate op struct
spi_mem_internal_supports_op() // check controller caps

if (ctlr->mem_ops->exec_op && !CS_GPIO) {
    spi_mem_access_start();
    ret = ctlr->mem_ops->exec_op(mem, op);
    spi_mem_access_end();
    if (!ret || ret not ENOTSUPP/EOPNOTSUPP)
        update_stats(); return ret;
}

// Generic fallback
tmpbuf = kzalloc(cmd+addr+dummy bytes, GFP_DMA);
build spi_message → 4 xfers
spi_sync(mem->spi, &msg);
check actual_length;
kfree(tmpbuf);
```

---

#### `spi_mem_read_status` (static)

```c
static int spi_mem_read_status(struct spi_mem *mem,
                               const struct spi_mem_op *op,
                               u16 *status);
```

بينفذ status register read operation ويحوّل الـ bytes لـ `u16`. لو `op->data.nbytes > 1` بيدمج byte[0] و byte[1] كـ big-endian `u16`.

**Return:** `0` في النجاح، error من `spi_mem_exec_op`.

**Key Details:**
- مصمم للاستخدام كـ callback في `read_poll_timeout`
- الـ `op->data.buf.in` لازم يشاور على buffer حجمه على الأقل `op->data.nbytes`

---

#### `spi_mem_poll_status`

```c
int spi_mem_poll_status(struct spi_mem *mem,
                        const struct spi_mem_op *op,
                        u16 mask, u16 match,
                        unsigned long initial_delay_us,
                        unsigned long polling_delay_us,
                        u16 timeout_ms);
```

بيستطلع status register لـ SPI memory device لحد ما `(status & mask) == match` أو انتهى الـ timeout. مهم جداً لعمليات زي write/erase polling على SPI flash.

**Parameters:**
- `mem` — الـ SPI memory device
- `op` — operation لقراءة الـ status (لازم `DATA_IN`، 1 أو 2 bytes)
- `mask` — الـ bitmask للـ bits اللي هنتحقق منها
- `match` — القيمة المتوقعة لـ `(status & mask)`
- `initial_delay_us` — انتظر كده قبل أول قراءة
- `polling_delay_us` — الفترة بين كل قراءة والتالية
- `timeout_ms` — أقصى وقت انتظار

**Return:** `0` في النجاح، `-ETIMEDOUT` لو انتهى الوقت، `-EOPNOTSUPP` لو مش مدعوم، `-EINVAL` لو الـ op غلط.

**Key Details:**
- بيحاول أولاً `ctlr->mem_ops->poll_status` لو موجود (hardware-assisted polling)
- CS GPIO يعطل الـ hardware polling path تماماً
- الـ software fallback يستخدم `read_poll_timeout()` من `<linux/iopoll.h>`
- `initial_delay_us < 10` → `udelay()` (busy wait)، غير كده → `usleep_range()`
- يُستدعى من SPI flash drivers لانتظار اكتمال write/erase

**Pseudocode Flow:**
```c
validate op (1-2 bytes, DATA_IN);

if (ctlr->mem_ops->poll_status && !CS_GPIO) {
    spi_mem_access_start();
    ret = ctlr->mem_ops->poll_status(...);
    spi_mem_access_end();
}

if (ret == -EOPNOTSUPP) {  // software fallback
    check spi_mem_supports_op();
    initial_delay();
    read_poll_timeout(spi_mem_read_status, ...);
}
```

---

### Group 4: Size & Frequency Adjustment

---

#### `spi_mem_adjust_op_size`

```c
int spi_mem_adjust_op_size(struct spi_mem *mem, struct spi_mem_op *op);
```

بيعدّل `op->data.nbytes` لما الـ controller مش قادر يعمل الـ transfer كامل في خطوة واحدة بسبب FIFO limits أو alignment constraints. الـ upper layers لازم يستدعوا الـ function دي في loop.

**Parameters:**
- `mem` — الـ SPI memory device
- `op` — الـ operation (بيتعدل `op->data.nbytes` in-place)

**Return:** `0` في النجاح، `-EINVAL` لو مستحيل التنفيذ.

**Key Details:**
- لو `ctlr->mem_ops->adjust_op_size` موجود → delegate إليه مباشرةً
- الـ fallback: يحسب `len = cmd + addr + dummy bytes`، يتحقق إنها أقل من `spi_max_transfer_size`
- بيستخدم `min3()` عشان يحدد `data.nbytes` الأمثل
- لو `data.nbytes == 0` بعد التعديل → `-EINVAL`
- يُستدعى من `spi_mem_no_dirmap_read/write` وبعض الـ MTD/NAND drivers

---

#### `spi_mem_adjust_op_freq`

```c
void spi_mem_adjust_op_freq(struct spi_mem *mem, struct spi_mem_op *op);
```

بيضبط `op->max_freq` عشان ميعلاش عن `mem->spi->max_speed_hz`. لو `op->max_freq == 0` أو كبير جداً، بيبقى `mem->spi->max_speed_hz`.

**Key Details:**
- بسيطة جداً لكنها مهمة: بتنفذ قبل أي operation check أو execution
- `op->max_freq = 0` معناها "استخدم أقصى سرعة ممكنة"

---

#### `spi_mem_calc_op_duration`

```c
u64 spi_mem_calc_op_duration(struct spi_mem *mem, struct spi_mem_op *op);
```

بيحسب التقدير النظري لمدة الـ operation بالـ nanoseconds. مفيد لمقارنة variants مختلفة من نفس الـ operation (مثلاً SDR vs DTR vs Quad) واختيار الأسرع.

**Parameters:**
- `mem` — الـ SPI memory device (لتحديد الـ freq)
- `op` — الـ operation المراد قياسها

**Return:** الزمن المقدّر بالـ ns. لو `max_freq == 0`، بيرجع عدد الـ clock cycles بدلاً منه.

**Key Details:**
- بيستدعي `spi_mem_adjust_op_freq` أولاً
- يحسب cycles لكل phase: `(nbytes * 8 / buswidth) / (dtr ? 2 : 1)`
- DTR بيقلل الـ cycles لأن البيانات بتتبعث على الـ rising وكمان الـ falling edge
- `ps_per_cycle = 1e12 / freq` ثم `duration_ps = cycles * ps_per_cycle` ثم `/1000` → ns
- لو `max_freq == 0`: `ps_per_cycle = 1` فبيرجع عدد الـ cycles — مفيد للمقارنة النسبية

---

### Group 5: Direct Mapping (dirmap)

الـ **direct mapping** بيسمح لبعض الـ QSPI controllers إنها تعمل hardware mapping لمنطقة من ذاكرة الـ SPI flash على address space الـ CPU. بدل ما تبعت SPI command لكل read، الـ CPU يقدر يوصل الـ flash مباشرة كأنها RAM. الـ group ده بيدير إنشاء وتدمير واستخدام الـ descriptors دي.

---

#### `spi_mem_no_dirmap_read` / `spi_mem_no_dirmap_write` (static)

```c
static ssize_t spi_mem_no_dirmap_read(struct spi_mem_dirmap_desc *desc,
                                      u64 offs, size_t len, void *buf);
static ssize_t spi_mem_no_dirmap_write(struct spi_mem_dirmap_desc *desc,
                                       u64 offs, size_t len, const void *buf);
```

الـ software fallback لما الـ controller مش بيدعم hardware dirmap. بيبنوا `spi_mem_op` من الـ template الموجود في `desc->info.op_tmpl`، بيضبطوا الـ address والـ buffer، وبيستدعوا `spi_mem_exec_op`.

**Return:** عدد الـ bytes المقروءة/المكتوبة (`op.data.nbytes` بعد التعديل)، أو negative error.

**Key Details:**
- بيستدعي `spi_mem_adjust_op_size` عشان يتعامل مع الـ controller limits
- الـ address = `desc->info.offset + offs` (absolute address in flash)

---

#### `spi_mem_dirmap_create`

```c
struct spi_mem_dirmap_desc *
spi_mem_dirmap_create(struct spi_mem *mem,
                      const struct spi_mem_dirmap_info *info);
```

ينشئ **direct mapping descriptor** لمنطقة معينة في الـ SPI memory. لو الـ controller يدعم HW dirmap → يستدعي `ctlr->mem_ops->dirmap_create()`. لو لا → يضع `desc->nodirmap = true` ويعمل fallback على `spi_mem_exec_op`.

**Parameters:**
- `mem` — الـ SPI memory device
- `info` — معلومات الـ mapping: template operation، offset، length

**Return:** valid `desc*` في النجاح، `ERR_PTR()` في الفشل.

**Key Details:**
- يتحقق: `addr.nbytes` بين 1 و8، و`data.dir` مش `SPI_MEM_NO_DATA`
- `kzalloc` للـ descriptor ثم `dirmap_create()` داخل `access_start/end`
- لو `dirmap_create` فشل: بيتحقق إن الـ op template مدعوم أصلاً بـ `spi_mem_supports_op`
- لو مش مدعوم أصلاً → `-EOPNOTSUPP` ويُدمّر الـ descriptor

**Pseudocode Flow:**
```c
validate info;
desc = kzalloc();
if (ctlr->mem_ops->dirmap_create) {
    spi_mem_access_start();
    ret = ctlr->mem_ops->dirmap_create(desc);
    spi_mem_access_end();
}
if (ret) {
    desc->nodirmap = true;
    if (!spi_mem_supports_op(op_tmpl)) return ERR_PTR(-EOPNOTSUPP);
}
return desc;
```

---

#### `spi_mem_dirmap_destroy`

```c
void spi_mem_dirmap_destroy(struct spi_mem_dirmap_desc *desc);
```

بيدمّر الـ descriptor وبيحرر كل الـ resources. لو مش `nodirmap` وفيه `dirmap_destroy` في الـ controller → بيستدعيه. دايماً بيعمل `kfree(desc)` في الآخر.

---

#### `devm_spi_mem_dirmap_create` / `devm_spi_mem_dirmap_destroy`

```c
struct spi_mem_dirmap_desc *
devm_spi_mem_dirmap_create(struct device *dev, struct spi_mem *mem,
                           const struct spi_mem_dirmap_info *info);

void devm_spi_mem_dirmap_destroy(struct device *dev,
                                 struct spi_mem_dirmap_desc *desc);
```

نسخ الـ `devm_` من create وdestroy. بيستخدموا `devres` عشان الـ `spi_mem_dirmap_destroy()` يُستدعى تلقائياً لما الـ device يتنزع. `devm_spi_mem_dirmap_release()` هي الـ devres callback.

**Key Details:**
- `devres_alloc()` → `devres_add()` للتسجيل
- `devm_spi_mem_dirmap_destroy()` بيستخدم `devres_release()` مع match function للـ descriptor المحدد

---

#### `spi_mem_dirmap_read`

```c
ssize_t spi_mem_dirmap_read(struct spi_mem_dirmap_desc *desc,
                             u64 offs, size_t len, void *buf);
```

بيقرأ بيانات من الـ SPI memory عبر direct mapping. يختار المسار المناسب: fallback (`nodirmap`) أو hardware (`ctlr->mem_ops->dirmap_read`).

**Parameters:**
- `desc` — الـ direct mapping descriptor (اتنشأ بـ `spi_mem_dirmap_create`)
- `offs` — offset داخل الـ mapping (مش absolute offset)
- `len` — عدد الـ bytes المطلوبة
- `buf` — destination buffer، لازم DMA-able

**Return:** عدد الـ bytes المقروءة فعلاً (ممكن أقل من `len`)، أو negative error.

**Key Details:**
- يتحقق إن `desc->info.op_tmpl.data.dir == SPI_MEM_DATA_IN`
- الـ caller مسؤول عن استدعاء الـ function في loop لو رجع أقل من `len`
- الـ HW path داخل `access_start/end` للـ locking الصحيح

---

#### `spi_mem_dirmap_write`

```c
ssize_t spi_mem_dirmap_write(struct spi_mem_dirmap_desc *desc,
                              u64 offs, size_t len, const void *buf);
```

مثل `spi_mem_dirmap_read` لكن للكتابة. يتحقق إن `data.dir == SPI_MEM_DATA_OUT`.

---

### Group 6: Driver Registration

الـ group ده بيوفر الـ thin wrapper layer اللي بتسمح لـ SPI memory drivers (spinand, spi-nor, etc.) إنهم يتسجلوا كـ standard drivers من غير ما يكونوا `spi_driver` مباشرةً.

---

#### `spi_mem_get_name`

```c
const char *spi_mem_get_name(struct spi_mem *mem);
```

بيرجع اسم الـ SPI memory device. الاسم ده بيتحدد في `spi_mem_probe()` من `ctlr->mem_ops->get_name` أو من `dev_name(&spi->dev)`.

**Return:** الاسم كـ string ثابت.

---

#### `to_spi_mem_drv` (static inline)

```c
static inline struct spi_mem_driver *to_spi_mem_drv(struct device_driver *drv)
{
    return container_of(drv, struct spi_mem_driver, spidrv.driver);
}
```

standard `container_of` helper ينقل من `device_driver*` لـ `spi_mem_driver*` عبر الـ embedded `spidrv.driver`.

---

#### `spi_mem_probe` (static)

```c
static int spi_mem_probe(struct spi_device *spi);
```

الـ internal probe بتاع الـ SPI mem layer. بيتنفذ لما الـ SPI core يطابق device مع driver. مسؤوليته:
1. allocate `spi_mem` struct بـ `devm_kzalloc`
2. ربط `mem->spi = spi`
3. تحديد الاسم من `ctlr->mem_ops->get_name` أو `dev_name`
4. `spi_set_drvdata(spi, mem)` لحفظ الـ mem pointer
5. استدعاء `memdrv->probe(mem)`

**Key Details:**
- لو `mem->name` رجع error → يرجع `PTR_ERR_OR_ZERO(mem->name)` مباشرةً
- الـ `spi_mem` struct بتتحرر تلقائياً بالـ devm عند unbind

---

#### `spi_mem_remove` / `spi_mem_shutdown` (static)

```c
static void spi_mem_remove(struct spi_device *spi);
static void spi_mem_shutdown(struct spi_device *spi);
```

بيستدعوا `memdrv->remove()` و `memdrv->shutdown()` على التوالي لو موجودين. بيجيبوا الـ `spi_mem` pointer من `spi_get_drvdata()`.

---

#### `spi_mem_driver_register_with_owner`

```c
int spi_mem_driver_register_with_owner(struct spi_mem_driver *memdrv,
                                       struct module *owner);
```

بيسجّل `spi_mem_driver` في الـ SPI bus. الـ trick هنا إنه بيضيف الـ internal wrappers (`spi_mem_probe`, `spi_mem_remove`, `spi_mem_shutdown`) للـ embedded `spidrv` قبل ما يستدعي `__spi_register_driver()`.

**Parameters:**
- `memdrv` — الـ driver المراد تسجيله
- `owner` — الـ module المالك (عادةً `THIS_MODULE`)

**Return:** `0` في النجاح، negative error من `__spi_register_driver`.

**Key Details:**
- بيُستخدم من الـ macro `spi_mem_driver_register(drv)` اللي بيمرر `THIS_MODULE` تلقائياً
- الـ macro `module_spi_mem_driver(drv)` بيوفر `module_init/exit` تلقائياً

---

#### `spi_mem_driver_unregister`

```c
void spi_mem_driver_unregister(struct spi_mem_driver *memdrv);
```

بيلغي تسجيل الـ driver عبر `spi_unregister_driver()` على الـ embedded `spidrv`.

---

### ملاحظة ختامية: تدفق العمل الكامل

```
Driver registers:
  module_spi_mem_driver(my_drv)
    → spi_mem_driver_register_with_owner()
    → __spi_register_driver()

Device matched:
  spi_mem_probe()
    → alloc spi_mem
    → my_drv->probe(mem)

Driver uses API:
  spi_mem_supports_op(mem, &op)
    → adjust_freq → check_op → controller caps check

  spi_mem_exec_op(mem, &op)
    → HW path: access_start → mem_ops->exec_op → access_end → stats
    → SW path: build spi_message → spi_sync

  spi_mem_dirmap_create(mem, &info)
    → HW: mem_ops->dirmap_create()
    → SW fallback: nodirmap=true

  spi_mem_dirmap_read(desc, offs, len, buf)
    → HW: access_start → mem_ops->dirmap_read → access_end
    → SW: spi_mem_no_dirmap_read → spi_mem_exec_op

  spi_mem_poll_status(mem, &op, mask, match, ...)
    → HW: mem_ops->poll_status
    → SW: read_poll_timeout(spi_mem_read_status, ...)
```
## Phase 5: دليل الـ Debugging الشامل

### Software Level

---

#### 1. debugfs — المدخلات المهمة

**الـ** `spi-mem` subsystem مش بيسجل entries خاصة بيه في debugfs، لكن الـ SPI core بيسجل statistics لكل controller وكل device.

```bash
# اعرض كل entries الـ SPI في debugfs
ls /sys/kernel/debug/spi*/

# statistics للـ controller (مثلاً spi0)
cat /sys/kernel/debug/spi0/statistics

# statistics للـ SPI device نفسه (مثلاً spi0.0)
cat /sys/kernel/debug/spi0/spi0.0/statistics
```

**الـ statistics** اللي بتلاقيها بتشمل:
| Field | المعنى |
|---|---|
| `messages` | عدد الـ spi_mem_op اللي اتنفذت |
| `transfers` | نفس الـ messages (كل op = transfer واحد) |
| `bytes` | إجمالي bytes (cmd+addr+dummy+data) |
| `bytes_tx` | bytes اتبعتت (DATA_OUT فقط) |
| `bytes_rx` | bytes اتستقبلت (DATA_IN فقط) |
| `errors` | عدد الـ ops اللي رجعت error |
| `timedout` | عدد الـ ops اللي عملت -ETIMEDOUT |
| `transfer_bytes_histo[N]` | histogram للحجم بالـ log2 |

الكود اللي بيملي الأرقام دي هو `spi_mem_add_op_stats()` في السطر 330.

---

#### 2. sysfs — المدخلات المهمة

```bash
# معلومات الـ SPI device
cat /sys/bus/spi/devices/spi0.0/modalias
cat /sys/bus/spi/devices/spi0.0/uevent

# الـ max speed اللي بيشتغل بيها الـ device
cat /sys/bus/spi/devices/spi0.0/spi_speed

# الـ SPI mode (single/dual/quad/octal)
cat /sys/bus/spi/devices/spi0.0/spi_mode

# Runtime PM state للـ controller
cat /sys/bus/spi/devices/spi0/power/runtime_status

# اسم الـ SPI mem device (من mem->name)
cat /sys/bus/spi/devices/spi0.0/driver/name
```

---

#### 3. ftrace — الـ Tracepoints

**الـ** `spi-mem.c` بيعرّف tracepoints من `trace/events/spi-mem.h`:

| Tracepoint | يتفعّل امتى |
|---|---|
| `spi_mem_start_op` | قبل ما `exec_op` تبدأ |
| `spi_mem_stop_op` | بعد ما `exec_op` تخلص |

```bash
# شوف كل tracepoints الـ SPI المتاحة
ls /sys/kernel/tracing/events/spi-mem/

# فعّل spi_mem_start_op
echo 1 > /sys/kernel/tracing/events/spi-mem/spi_mem_start_op/enable

# فعّل spi_mem_stop_op
echo 1 > /sys/kernel/tracing/events/spi-mem/spi_mem_stop_op/enable

# فعّل الاتنين مع بعض
echo 1 > /sys/kernel/tracing/events/spi-mem/enable

# ابدأ الـ tracing
echo 1 > /sys/kernel/tracing/tracing_on

# شغّل العملية اللي عايز تتبعها، بعدين اقرأ
cat /sys/kernel/tracing/trace
```

**مثال على output من `spi_mem_start_op`:**
```
spi0.0 1S-4S-4S @50000000 Hz op=[03 00 00 00] len=4096 tx=[]
#        ^cmd  ^addr ^data
#        buswidth S=Single D=DTR
```

**مثال على output من `spi_mem_stop_op`:**
```
spi0.0 len=4096 rx=[de ad be ef ...]
```

**لو حابب تفلتر على device معين:**
```bash
echo 'name == "spi0.0"' > /sys/kernel/tracing/events/spi-mem/spi_mem_start_op/filter
```

---

#### 4. printk / dynamic debug

**الـ** `spi_mem_exec_op()` بيستخدم `dev_vdbg()` لطباعة كل operation في السطر 397. دي verbose debug، محتاج تفعّلها.

```bash
# فعّل dynamic debug لكل spi-mem.c
echo 'file spi-mem.c +p' > /sys/kernel/debug/dynamic_debug/control

# أو فعّل verbose فقط (vdbg)
echo 'file spi-mem.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# فعّل للـ module كله
echo 'module spi-mem +p' > /sys/kernel/debug/dynamic_debug/control

# شوف الإعدادات الحالية
cat /sys/kernel/debug/dynamic_debug/control | grep spi-mem
```

**بعد التفعيل، هتلاقي في dmesg:**
```
spi0.0: [cmd: 0x03][3B addr: 0x000000][0B dummy][4096B data  read] 1S-4S-0S-4S @ 50000000Hz
```

**لو حابب `printk_ratelimit` ميبوظش اللوج:**
```bash
echo 0 > /proc/sys/kernel/printk_ratelimit
```

---

#### 5. Kernel Config Options للـ Debugging

| Config | الغرض |
|---|---|
| `CONFIG_SPI_MEM` | تفعيل الـ SPI-MEM layer نفسه (مطلوب) |
| `CONFIG_SPI_DEBUG` | verbose logging للـ SPI core |
| `CONFIG_DYNAMIC_DEBUG` | تفعيل `dev_dbg`/`dev_vdbg` runtime |
| `CONFIG_TRACING` | تفعيل ftrace infrastructure |
| `CONFIG_EVENT_TRACING` | تفعيل tracepoints |
| `CONFIG_DMA_API_DEBUG` | كشف أخطاء الـ DMA mapping (مهم لـ `spi_controller_dma_map_mem_op_data`) |
| `CONFIG_DMA_API_DEBUG_SG` | فحص scatter-gather tables |
| `CONFIG_PM_DEBUG` | debug لـ runtime PM (مهم لـ `auto_runtime_pm`) |
| `CONFIG_LOCKDEP` | كشف deadlocks في الـ `bus_lock_mutex`/`io_mutex` |
| `CONFIG_DEBUG_ATOMIC_SLEEP` | كشف sleep في atomic context |
| `CONFIG_SLUB_DEBUG` | فحص كشف الـ heap corruption في `kzalloc` |
| `CONFIG_STACK_VALIDATION` | يساعد مع `object_is_on_stack()` check |

```bash
# تحقق من config الـ kernel الحالي
zcat /proc/config.gz | grep -E "CONFIG_SPI|CONFIG_DMA_API|CONFIG_PM_DEBUG"
```

---

#### 6. devlink وأدوات خاصة بالـ Subsystem

**الـ** `spi-mem` مش بيستخدم devlink مباشرة. لكن الأدوات دي مفيدة:

```bash
# spi-tools: اقرأ/اكتب على SPI device مباشرة
# install: apt install spi-tools  أو  pip install spidev
spi-config -d /dev/spidev0.0 --query

# flash_erase / nanddump لو الـ device هو NAND/NOR flash
flash_erase /dev/mtd0 0 0
nanddump -f dump.bin /dev/mtd0

# mtd-utils للتعامل مع flash memory
mtdinfo /dev/mtd0
mtdinfo -a   # كل الـ MTD devices
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الخطأ | المعنى | الإصلاح |
|---|---|---|
| `Even byte numbers not allowed in octal DTR operations` | عدد bytes الـ addr/dummy/data فردي في 8D-8D-8D mode | تأكد إن `addr.nbytes` و`dummy.nbytes` و`data.nbytes` كلهم زوجي |
| `Failed to power device: %d` | runtime PM resume فشل على الـ parent device | افحص الـ power supply والـ PM callbacks للـ parent |
| `-ENOTSUPP` من `spi_mem_exec_op` | الـ controller مش بيدعم الـ op | غيّر buswidth أو قلل الـ frequency أو استخدم SPI mode مختلف |
| `-EOPNOTSUPP` | الـ op مش supported نهائياً | افحص `spi_mem_supports_op()` واطبع الـ mode bits |
| `-EIO` من `spi_mem_exec_op` | `msg.actual_length != totalxferlen` | مشكلة في الـ controller hardware أو DMA transfer truncated |
| `-ENOMEM` من `spi_mem_dirmap_create` | فشل `kzalloc` | ضغط على الـ memory، قلل الـ allocations المتزامنة |
| `-ETIMEDOUT` من `spi_mem_poll_status` | الـ flash مرجعش ready في الوقت | زوّد `timeout_ms` أو افحص الـ flash hardware |
| `-EINVAL` من `spi_mem_check_op` | buswidth = 0 أو nbytes = 0 في الـ cmd | راجع تعريف `spi_mem_op` في الكود |
| `-EINVAL` من `spi_mem_dirmap_create` | `addr.nbytes` = 0 أو > 8، أو `data.dir == NO_DATA` | صحّح قيم `spi_mem_dirmap_info` |
| `WARN_ON_ONCE: data buf is on stack` | الـ buffer المرسوم للبيانات على stack مش DMA-able | استخدم `kmalloc` أو `devm_kzalloc` بدل stack buffers |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

الكود نفسه بيستخدم `WARN_ON_ONCE()` في `spi_mem_check_op()` السطر 243-248 لكشف stack buffers. إليك نقاط إضافية مقترحة:

```c
/* في spi_mem_exec_op() بعد spi_mem_check_op() */
WARN_ON(!mem->spi->controller);
WARN_ON(!mem->spi->controller->mem_ops);

/* في spi_mem_access_start() لو الـ mutex مش متاح */
WARN_ON(mutex_is_locked(&ctlr->io_mutex));

/* في spi_mem_dirmap_read/write للتحقق من الـ direction */
WARN_ON(desc->info.op_tmpl.data.dir == SPI_MEM_NO_DATA);

/* في spi_mem_add_op_stats() للتحقق من الإحصاءات */
WARN_ON(exec_op_ret == -ENOMEM);  /* memory issues shouldn't be silent */
```

---

### Hardware Level

---

#### 1. مطابقة حالة الـ Hardware مع الـ Kernel

```bash
# شوف الـ SPI mode اللي الـ kernel بيعتقده
cat /sys/bus/spi/devices/spi0.0/spi_mode
# 0=CPHA=0,CPOL=0 | 1=CPHA=1,CPOL=0 | 2=CPHA=0,CPOL=1 | 3=CPHA=1,CPOL=1

# قارن مع SPI mode في Device Tree
cat /sys/firmware/devicetree/base/spi@xxx/flash@0/spi-tx-bus-width

# الـ max-speed الـ kernel بيستخدمه
cat /sys/bus/spi/devices/spi0.0/spi_speed

# تأكد من الـ chip select
ls /sys/class/gpio/ | grep cs
```

**الفحص المنطقي:**
```
Kernel state (spi->mode bits)  ←→  Hardware (DT / board)
SPI_TX_QUAD set?               ←→  4 IO lines connected on PCB?
SPI_RX_OCTAL set?              ←→  8 IO lines connected on PCB?
max_speed_hz                   ←→  Flash datasheet max freq?
```

---

#### 2. Register Dump Techniques

```bash
# قرأ registers الـ SPI controller مباشرة (مثال لـ base address 0xFE200000)
# تأكد من الـ base address من Device Tree أو datasheet

# باستخدام devmem2
devmem2 0xFE200000 w   # قرأ 32-bit word
devmem2 0xFE200004 w
devmem2 0xFE200008 w

# باستخدام /dev/mem
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    mm = mmap.mmap(f.fileno(), 0x1000, offset=0xFE200000)
    for i in range(0, 0x40, 4):
        val = struct.unpack('<I', mm[i:i+4])[0]
        print(f'0x{0xFE200000+i:08x}: 0x{val:08x}')
    mm.close()
"

# باستخدام io utility (من busybox أو مستقل)
io -4 0xFE200000

# لو بتستخدم Raspberry Pi SPI:
devmem2 0x3F204000 w  # SPI0 CS register
devmem2 0x3F204004 w  # SPI0 FIFO
devmem2 0x3F204008 w  # SPI0 CLK divider
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

**نقاط القياس على الـ PCB:**

```
SPI Flash Pins:
  CS#  ──── قِس timing من rising edge إلى next falling edge
  SCK  ──── قِس frequency والـ duty cycle
  IO0  ──── (MOSI في single mode, data line 0 في quad)
  IO1  ──── (MISO في single mode, data line 1 في quad)
  IO2  ──── (WP# في single أو data line 2 في quad)
  IO3  ──── (HOLD# في single أو data line 3 في quad)
```

**ما تدور عليه:**

| الظاهرة | الدلالة |
|---|---|
| CS# مش بينزل (stays HIGH) | الـ controller مش بيبعت أوامر / CS GPIO problem |
| SCK يقف في النص | deadlock في الـ bus_lock_mutex / timeout |
| Data lines مش بيتغيروا في Quad mode | الـ IO lines مش connected صح أو mode bits غلط |
| CS# بيرتفع قبل ما البيانات تخلص | `actual_length < totalxferlen` → -EIO |
| Glitches على SCK | power supply noise، زوّد decoupling caps |
| الـ Quad lines بيتحركوا بس بيانات غلط | endianness issue، أو `swap16` flag مطلوب |

**Trigger Setup للـ Logic Analyzer:**
```
Trigger: CS# falling edge
Sample rate: min 4x الـ SCK frequency
Capture: 10ms minimum لـ flash operations
Protocol: SPI with Quad-SPI decoder إذا متاح
```

---

#### 4. Common Hardware Issues وأنماط الـ Kernel Log

| المشكلة | النمط في dmesg |
|---|---|
| Flash مش موجود على الـ bus | `spi0.0: probe failed, err=-6 (ENXIO)` |
| Quad mode مش connected | `-ENOTSUPP` من supports_op + fallback لـ single |
| Power-up sequence خاطئ | `spi_mem_access_start: Failed to power device: -5` |
| DMA buffer on stack | `WARNING: ... object_is_on_stack` |
| Timeout في flash erase/write | `spi_mem_poll_status: -ETIMEDOUT` |
| CS GPIO مش configured | `spi0.0: no mem_ops->exec_op, using gpio CS fallback` |
| Frequency فوق القدرة | `spi_mem_default_supports_op: op->max_freq < min_speed_hz` |
| DTR mode غير مدعوم | `spi_mem_default_supports_op returns false` (مفيش log افتراضي) |

---

#### 5. Device Tree Debugging

```bash
# اطبع كل الـ DT الـ active
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | less

# ابحث عن الـ SPI nodes
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "spi@"

# تحقق من قيمة spi-tx-bus-width
cat /sys/firmware/devicetree/base/spi@xxx/flash@0/spi-tx-bus-width | xxd

# تحقق من الـ compatible string
cat /sys/firmware/devicetree/base/spi@xxx/flash@0/compatible

# تحقق من max-frequency في DT
cat /sys/firmware/devicetree/base/spi@xxx/flash@0/spi-max-frequency | xxd
```

**مطابقة الـ DT مع الـ Kernel:**

```
DT property              → Kernel field
---------------------------------
spi-tx-bus-width = <4>   → SPI_TX_QUAD في spi->mode
spi-rx-bus-width = <4>   → SPI_RX_QUAD في spi->mode
spi-max-frequency        → spi->max_speed_hz
spi-cpha                 → SPI_CPHA في spi->mode
spi-cpol                 → SPI_CPOL في spi->mode
cs-gpios                 → spi_get_csgpiod() != NULL
```

**لو الـ DT مش بيتحمل صح:**
```bash
# شوف الـ kernel overlay errors
dmesg | grep -i "dtb\|overlay\|of_node"

# تحقق من الـ probe
dmesg | grep "spi.*probe\|spi-mem"
```

---

### Practical Commands

---

#### أمر 1: شغّل full debugging جاهز للنسخ

```bash
#!/bin/bash
# spi-mem-debug.sh — فعّل كل أدوات debugging الـ spi-mem

# 1. dynamic debug
echo 'file spi-mem.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# 2. ftrace tracepoints
echo 0 > /sys/kernel/tracing/tracing_on
echo '' > /sys/kernel/tracing/trace
echo 1 > /sys/kernel/tracing/events/spi-mem/enable
echo 1 > /sys/kernel/tracing/tracing_on

# 3. زوّد الـ trace buffer
echo 32768 > /sys/kernel/tracing/buffer_size_kb

echo "SPI-MEM debug enabled. Run your workload, then:"
echo "  cat /sys/kernel/tracing/trace"
```

---

#### أمر 2: اقرأ statistics وافهمها

```bash
# اقرأ statistics لكل SPI devices
for dev in /sys/kernel/debug/spi*/*/statistics; do
    echo "=== $dev ==="
    cat "$dev"
done

# مثال على output:
# messages:           1024
# transfers:          1024
# bytes:             2097152   ← 2MB إجمالي (cmd+addr+data)
# bytes_tx:               0   ← مافيش write ops
# bytes_rx:         2093056   ← data bytes فقط
# errors:                 0
# timedout:               0
# transfer_bytes_histo[12]:  1024  ← كل ops حجمها بين 2KB و4KB
```

---

#### أمر 3: تتبع op واحدة بالتفصيل

```bash
# فعّل function tracing لـ spi_mem_exec_op
echo 'function_graph' > /sys/kernel/tracing/current_tracer
echo 'spi_mem_exec_op' > /sys/kernel/tracing/set_graph_function
echo 1 > /sys/kernel/tracing/tracing_on

# شغّل العملية
dd if=/dev/mtd0 of=/dev/null bs=4096 count=1

echo 0 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace | head -50
```

**مثال على output:**
```
 0)               |  spi_mem_exec_op() {
 0)               |    spi_mem_check_op() {
 0)   0.234 us    |    }
 0)               |    spi_mem_access_start() {
 0)   1.023 us    |      spi_flush_queue();
 0)   2.145 us    |    }
 0)  45.678 us    |    ctlr->mem_ops->exec_op();   /* hardware time */
 0)   0.123 us    |    spi_mem_access_end();
 0)  48.456 us    |  }
```

---

#### أمر 4: تحقق من supports_op بدون تشغيل

```bash
# اكتب module بسيط للاختبار أو استخدم الـ dmesg مع debug enabled
# بعد تفعيل dynamic debug، دور على رسايل الـ supports_op

dmesg -w | grep -E "spi.*op|supports|ENOTSUPP|EOPNOTSUPP"

# أو لو عندك kernel debugger
# bp spi_mem_default_supports_op
```

---

#### أمر 5: فحص DMA mapping errors

```bash
# فعّل DMA API debug (لازم CONFIG_DMA_API_DEBUG=y في الـ kernel)
echo 1 > /sys/kernel/debug/dma-api/disabled

# بعد تشغيل عملية، شوف الأخطاء
cat /sys/kernel/debug/dma-api/error_count
cat /sys/kernel/debug/dma-api/errors

# لو ظهر WARN_ON_ONCE على stack buffer
dmesg | grep "object_is_on_stack\|WARNING.*spi-mem"
```

---

#### أمر 6: مراقبة runtime PM للـ controller

```bash
# شوف الـ PM state للـ controller
CTLR=$(ls /sys/bus/spi/drivers/*/spi*.*/power/ 2>/dev/null | head -1 | xargs dirname)
echo "Controller: $CTLR"
cat "$CTLR/power/runtime_status"
cat "$CTLR/power/runtime_suspended_time"
cat "$CTLR/power/runtime_active_time"

# لو الـ controller في suspended والـ op بتفشل:
# Failed to power device: -EACCES أو -EAGAIN
cat "$CTLR/power/control"   # auto أو on
echo on > "$CTLR/power/control"  # منع الـ suspend مؤقتاً للتشخيص
```

---

#### أمر 7: تفسير نتائج الـ trace الكاملة

```
# مثال complete trace لـ QSPI flash read:
spi_mem_start_op:
  spi0.0 1S-4S-4S @50000000 Hz op=[03 00 10 00] len=256 tx=[]
  #        ^        ^  ^
  #        cmd 1-wire  data 4-wire quad
  #        addr 4-wire
  #        op[0]=0x03 (READ command)
  #        op[1:3]=0x001000 (address 0x1000)

spi_mem_stop_op:
  spi0.0 len=256 rx=[ff 00 ab cd ...]
  #                   ^data received
```

**تفسير الـ buswidth field:**
```
1S  = Single wire, STR (Single Transfer Rate)
2S  = Dual wire, STR
4S  = Quad wire, STR
1D  = Single wire, DTR (Double Transfer Rate)
8D  = Octal wire, DTR
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: SPI NOR Flash مش شغال على RK3562 Industrial Gateway

#### العنوان
**Quad-SPI read** بترجع بيانات غلط بعد boot على gateway صناعي

#### السياق
شركة بتعمل industrial gateway بيحتاج يقرأ firmware من **W25Q128** (SPI NOR Flash 128Mbit) متوصل على **RK3562** عن طريق QSPI controller. الـ board كانت شغالة تمام في Quad mode على بروتوتايب، بس بعد ما اتعمل PCB production revision، الـ NOR flash بدأت ترجع checksum errors عند قراءة الـ firmware.

#### المشكلة
الـ `mtd` subsystem بيبلّغ عن CRC errors عند `dd if=/dev/mtd0 of=/tmp/fw.bin`. الـ flash شغالة تمام في Single (x1) mode بس في Quad (x4) mode الداتا خربانة.

#### التحليل
الأكيد إن المشكلة في الـ **buswidth** negotiation. في `spi-mem.c`:

```c
/* spi_check_buswidth_req() — بيتحقق إن الـ controller mode يدعم العرض المطلوب */
static int spi_check_buswidth_req(struct spi_mem *mem, u8 buswidth, bool tx)
{
    u32 mode = mem->spi->mode;

    switch (buswidth) {
    case 4:
        if ((tx && (mode & (SPI_TX_QUAD | SPI_TX_OCTAL))) ||
            (!tx && (mode & (SPI_RX_QUAD | SPI_RX_OCTAL))))
            return 0;
        break;
    ...
    }
    return -ENOTSUPP;
}
```

الـ `spi_mem_default_supports_op()` بتستدعي `spi_mem_check_buswidth()` اللي بتمرر على CMD و ADDR و DUMMY و DATA phases. لو الـ mode flags ماشية، الـ op بتتنفّذ بـ Quad.

لكن الـ `spi_mem_exec_op()` بيبني xfer بالشكل ده:

```c
xfers[xferpos].tx_nbits = op->cmd.buswidth;   /* cmd phase */
/* ... */
xfers[xferpos].rx_nbits = op->data.buswidth;  /* data phase */
```

المشكلة: الـ PCB revision قصّ اتنين من الـ IO lines (IO2, IO3) لأن الـ layout engineer ظن إنهم مش مهمين. الـ controller بيبعت Quad command صح، بس الـ flash مبتردّش صح لأن IO2/IO3 floating.

#### الحل
**الـ debug خطوة بخطوة:**

```bash
# شوف الـ spi device mode المسجّل في kernel
cat /sys/bus/spi/devices/spi0.0/modalias

# استخدم spi-mem trace points
echo 1 > /sys/kernel/debug/tracing/events/spi_mem/enable
cat /sys/kernel/debug/tracing/trace_pipe
# هتشوف: [cmd: 0x6b][3B addr: 0x000000][8B dummy][4096B data  read] 1S-1S-1S-4S
```

**الـ DT fix:** لازم تمنع الـ driver من استخدام Quad:

```dts
/* في device tree: قيّد الـ mode لـ Dual بس */
&spi0 {
    flash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;
        spi-max-frequency = <50000000>;
        /* امنع Quad — IO2/IO3 مش متوصلين */
        spi-rx-bus-width = <2>;
        spi-tx-bus-width = <1>;
    };
};
```

بكده `spi_check_buswidth_req()` هترفض أي op بـ buswidth=4 وهترجع `-ENOTSUPP`، والـ driver هينزل على Dual تلقائيًا.

#### الدرس المستفاد
دايمًا verify إن كل IO lines للـ QSPI متوصلة على الـ PCB. الـ `spi_mem_check_buswidth()` بتثق في الـ mode flags اللي بيحددها الـ DT، ومش بتعمل hardware probing للـ lines الفعلية. لو سمحت للـ kernel يستخدم Quad بدون توصيل كامل، هتاخد data corruption بدون error واضح.

---

### السيناريو 2: SPI NAND على STM32MP1 — `spi_mem_poll_status` Timeout في Production

#### العنوان
**Write operations** بتتعلق وبتطلع `ETIMEDOUT` على IoT sensor node بـ STM32MP1

#### السياق
IoT device بيخزن sensor readings على **Winbond W25N01GV** (SPI NAND 1Gbit) متوصل على **STM32MP1** QSPI. الـ device شغالة تمام في lab، بس في production environment (درجة حرارة عالية، 70°C) الـ write operations بتفشل بـ `-ETIMEDOUT`.

#### المشكلة
الـ `spinand` driver بيستدعي `spi_mem_poll_status()` بعد كل page program لانتظار الـ flash تخلص. في الـ production، الـ timeout بيتجاوز قبل ما الـ flash تخلص.

#### التحليل
في `spi-mem.c`، الـ `spi_mem_poll_status()`:

```c
int spi_mem_poll_status(struct spi_mem *mem,
                        const struct spi_mem_op *op,
                        u16 mask, u16 match,
                        unsigned long initial_delay_us,
                        unsigned long polling_delay_us,
                        u16 timeout_ms)
{
    /* لو الـ controller معندوش poll_status hardware support */
    if (ret == -EOPNOTSUPP) {
        /* initial delay قبل ما نبدأ نسأل */
        if (initial_delay_us < 10)
            udelay(initial_delay_us);
        else
            usleep_range((initial_delay_us >> 2) + 1,
                         initial_delay_us);

        /* polling loop */
        ret = read_poll_timeout(spi_mem_read_status, read_status_ret,
                    (read_status_ret || ((status) & mask) == match),
                    polling_delay_us, timeout_ms * 1000, false, mem,
                    op, &status);
    }
    return ret;
}
```

الـ `timeout_ms` بيتحوّل لـ microseconds جوه `read_poll_timeout` بضرب 1000. المشكلة إن الـ spinand driver كان بيدي `timeout_ms = 400` (400ms). في درجات الحرارة العالية، الـ W25N01GV ممكن تاخد حتى 700µs/page عوضًا عن 300µs عادي، والـ page program بيتأخر.

**الـ STM32MP1 controller** (fsmc/qspi) مش بيـimplement `mem_ops->poll_status`، فالـ kernel بيقع على الـ software fallback اللي فيه overhead من كل `spi_mem_exec_op()` call.

#### الحل
**أولًا: تشخيص:**

```bash
# شوف statistics الـ SPI device
cat /sys/class/spi_master/spi0/spi0.0/statistics/timedout
cat /sys/class/spi_master/spi0/spi0.0/statistics/errors

# تشخيص التوقيت
echo 1 > /sys/kernel/debug/tracing/events/spi_mem/spi_mem_start_op/enable
echo 1 > /sys/kernel/debug/tracing/events/spi_mem/spi_mem_stop_op/enable
```

**ثانيًا: الـ fix في spinand driver** — زيادة الـ timeout:

```c
/* في drivers/mtd/nand/spi/winbond.c */
#define WINBOND_CFG_OTP_PROTECT    BIT(7)

/* زود الـ timeout لـ 1000ms بدل 400ms عشان High-temp operation */
static int winbond_ecc_get_status(struct spinand_device *spinand,
                                   u8 status)
{
    /* ... */
}

/* في spinand_wait() استخدم timeout أكبر */
#define SPINAND_WRITE_TIMEOUT_MS   1000  /* was 400 */
```

**ثالثًا: implement `poll_status` في الـ STM32MP QSPI driver** لتقليل الـ CPU overhead:

```c
/* في spi-stm32-qspi.c */
static int stm32_qspi_poll_status(struct spi_mem *mem,
                                   const struct spi_mem_op *op,
                                   u16 mask, u16 match,
                                   unsigned long initial_delay_us,
                                   unsigned long polling_delay_us,
                                   u16 timeout_ms)
{
    /* استخدم الـ QSPI autopolling hardware register */
    /* QUADSPI_PSMAR = match, QUADSPI_PSMKR = mask */
}
```

#### الدرس المستفاد
الـ `spi_mem_poll_status()` بيوفّر graceful fallback للـ software polling لما الـ controller مش بيدعم hardware polling، بس الـ software polling أبطأ بكتير ومحساس لأي overhead. في بيئات درجات الحرارة المتطرفة، دايمًا اختبر مع temperature derating specs الـ flash واضبط الـ timeouts بناءً عليها.

---

### السيناريو 3: i.MX8MP — DMA Crash مع Stack-Allocated Buffer في SPI NOR Driver

#### العنوان
Kernel panic بـ `BUG: unable to handle kernel NULL pointer` عند قراءة من **QSPI NOR** على **i.MX8MP** Android TV Box

#### السياق
شركة بتبني Android TV box على **NXP i.MX8MP**. الـ bootloader configuration محتاجة تتخزن في **MX25U12835F** SPI NOR Flash. الـ developer كتب custom driver بسيط يقرأ الـ config، بس الـ kernel بيطبع `WARN_ON_ONCE` ومن وقت لتاني بيعمل panic.

#### المشكلة
الـ driver بيخصص الـ read buffer على الـ stack:

```c
/* BUG: الكود الغلط في custom driver */
static int read_config(struct spi_mem *mem)
{
    u8 buf[256];  /* stack allocation — WRONG! */
    struct spi_mem_op op = SPI_MEM_OP(
        SPI_MEM_OP_CMD(0x03, 1),
        SPI_MEM_OP_ADDR(3, 0x00000, 1),
        SPI_MEM_OP_NO_DUMMY,
        SPI_MEM_OP_DATA_IN(256, buf, 1)
    );
    return spi_mem_exec_op(mem, &op);
}
```

#### التحليل
في `spi_mem_check_op()`:

```c
static int spi_mem_check_op(const struct spi_mem_op *op)
{
    /* ... */

    /* Buffers must be DMA-able. */
    if (WARN_ON_ONCE(op->data.dir == SPI_MEM_DATA_IN &&
                     object_is_on_stack(op->data.buf.in)))
        return -EINVAL;

    if (WARN_ON_ONCE(op->data.dir == SPI_MEM_DATA_OUT &&
                     object_is_on_stack(op->data.buf.out)))
        return -EINVAL;
    /* ... */
}
```

الـ `object_is_on_stack()` بيتحقق إن الـ pointer واقع في range الـ stack الـ current thread. لو صح، بيطبع `WARN_ON_ONCE` وبيرجع `-EINVAL`.

المشكلة إن الـ **i.MX8MP FlexSPI controller** (نعم، بيستخدم `mem_ops->exec_op`) بيعمل DMA على الـ buffer مباشرة. الـ stack memory مش guaranteed أنها DMA-capable (ممكن تكون في kernel stack page اللي مش physically contiguous أو مش mapped للـ DMA).

حتى لو الـ `WARN_ON_ONCE` ماعملش panic، في `spi_controller_dma_map_mem_op_data()`:

```c
int spi_controller_dma_map_mem_op_data(struct spi_controller *ctlr,
                                       const struct spi_mem_op *op,
                                       struct sg_table *sgt)
{
    /* ... */
    return spi_map_buf(ctlr, dmadev, sgt, op->data.buf.in,
                       op->data.nbytes,
                       op->data.dir == SPI_MEM_DATA_IN ?
                       DMA_FROM_DEVICE : DMA_TO_DEVICE);
}
```

الـ `spi_map_buf()` بيحاول يعمل DMA map على stack address فبيفشل أو بيكوّرت لأن الـ IOMMU على i.MX8MP strict.

#### الحل

```c
/* الكود الصح: kmalloc() يضمن DMA-able buffer */
static int read_config(struct spi_mem *mem)
{
    u8 *buf;
    struct spi_mem_op op;
    int ret;

    /* GFP_DMA يضمن إن الـ buffer في DMA-accessible memory */
    buf = kmalloc(256, GFP_KERNEL);
    if (!buf)
        return -ENOMEM;

    op = (struct spi_mem_op)SPI_MEM_OP(
        SPI_MEM_OP_CMD(0x03, 1),
        SPI_MEM_OP_ADDR(3, 0x00000, 1),
        SPI_MEM_OP_NO_DUMMY,
        SPI_MEM_OP_DATA_IN(256, buf, 1)
    );

    ret = spi_mem_exec_op(mem, &op);
    if (!ret)
        memcpy(config_data, buf, 256);

    kfree(buf);
    return ret;
}
```

**للتحقق من الـ DMA safety:**

```bash
# شوف WARN_ON messages في dmesg
dmesg | grep "object_is_on_stack\|spi.*stack\|WARN.*spi"

# فعّل KASAN للتشخيص في development build
CONFIG_KASAN=y
CONFIG_KASAN_INLINE=y
```

#### الدرس المستفاد
الـ `spi_mem_check_op()` بيوفّر حماية صريحة ضد stack buffers عبر `WARN_ON_ONCE` + return `-EINVAL`. أي buffer بتديه لـ `spi_mem_exec_op()` لازم يكون DMA-safe — يعني مخصص بـ `kmalloc()`، `devm_kmalloc()`، أو `dma_alloc_coherent()`. الـ stack buffers ممنوعة تمامًا.

---

### السيناريو 4: AM62x — dirmap Performance بتبوظ بعد تحديث الـ Kernel على Automotive ECU

#### العنوان
**QSPI read throughput** بينزل من 50MB/s لـ 2MB/s بعد kernel upgrade على ECU بـ **TI AM62x**

#### السياق
Automotive ECU بيستخدم **TI AM62x** SoC مع **MT25QU256** QSPI NOR Flash (256Mbit) لتخزين الـ calibration data. بعد upgrade من kernel 5.15 لـ 6.1، الـ QSPI read performance اتدهورت بشكل دراماتيكي، وده أثر على real-time response الـ ECU.

#### المشكلة
الـ `spi_mem_dirmap_read()` بقت تستخدم الـ fallback path بدل الـ hardware direct mapping.

#### التحليل
في `spi_mem_dirmap_create()`:

```c
struct spi_mem_dirmap_desc *
spi_mem_dirmap_create(struct spi_mem *mem,
                      const struct spi_mem_dirmap_info *info)
{
    /* ... */
    desc->mem = mem;
    desc->info = *info;
    if (ctlr->mem_ops && ctlr->mem_ops->dirmap_create) {
        ret = spi_mem_access_start(mem);
        /* ... */
        ret = ctlr->mem_ops->dirmap_create(desc);  /* hardware path */
        spi_mem_access_end(mem);
    }

    if (ret) {
        desc->nodirmap = true;  /* fallback mode */
        if (!spi_mem_supports_op(desc->mem, &desc->info.op_tmpl))
            ret = -EOPNOTSUPP;
        else
            ret = 0;
    }
    /* ... */
}
```

لو `dirmap_create()` فشلت، `desc->nodirmap = true`، وكل قراءة هتروح لـ:

```c
ssize_t spi_mem_dirmap_read(struct spi_mem_dirmap_desc *desc,
                             u64 offs, size_t len, void *buf)
{
    if (desc->nodirmap) {
        ret = spi_mem_no_dirmap_read(desc, offs, len, buf);
        /* هنا: كل read بتعمل full spi_mem_exec_op() */
    } else if (ctlr->mem_ops && ctlr->mem_ops->dirmap_read) {
        /* الـ fast path: controller does the mapping */
        ret = ctlr->mem_ops->dirmap_read(desc, offs, len, buf);
    }
}
```

الـ `spi_mem_no_dirmap_read()` بتعمل `spi_mem_exec_op()` على كل chunk، وفيها overhead من `spi_message` building وـ DMA setup لكل call.

**السبب الفعلي في kernel 6.1:** تغيير في `ti_qspi` driver جعل `dirmap_create()` ترفض operations بـ `max_freq` أعلى من حد معين بسبب bug في frequency check مع `spi_mem_default_supports_op()`:

```c
/* في spi_mem_default_supports_op() */
if (op->max_freq &&
    op->max_freq < mem->spi->max_speed_hz) {
    if (!spi_mem_controller_is_capable(ctlr, per_op_freq))
        return false;  /* <-- هنا الـ reject */
}
```

الـ AM62x OSPI controller في الـ kernel 6.1 driver لم يكن بيـset `per_op_freq` capability، فلو الـ op template فيها `max_freq`، الـ `supports_op` بترجع false وبكده الـ `dirmap_create()` بتفشل.

#### الحل

**الـ debug:**

```bash
# تحقق إن nodirmap مضبوط
cat /sys/kernel/debug/spi0/spi0.0/dirmap_read
# أو عبر trace
echo 'spi_mem_*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
```

**الـ fix في الـ DT أو driver:**

```c
/* Option 1: في ti_qspi.c، أضف per_op_freq capability */
static const struct spi_controller_mem_caps ti_qspi_mem_caps = {
    .per_op_freq = true,  /* أضف هذا */
    .dtr = false,
};
```

```dts
/* Option 2: في DT، شيل الـ max_freq constraint من op_tmpl */
/* أو خفّض spi-max-frequency عشان ميتجاوزش الـ controller limit */
&ospi0 {
    flash@0 {
        compatible = "jedec,spi-nor";
        spi-max-frequency = <133000000>;
        /* ماتحطش spi-rx-sample-delay-ns قيمة عالية */
    };
};
```

#### الدرس المستفاد
الـ `nodirmap` flag في `spi_mem_dirmap_desc` هو silent performance killer — الـ code بيشتغل صح بس ببطء شديد. دايمًا check إن `dirmap_create()` نجحت بعد الـ probe واتأكد إن `ctlr->mem_caps` بيحتوي على الـ capabilities الصح. الفرق بين dirmap read وـ exec_op fallback ممكن يكون 25x في throughput.

---

### السيناريو 5: Allwinner H616 — Octal DTR SPI NOR مش شغال على Android TV Box

#### العنوان
**8D-8D-8D mode** على **Macronix MX25UM51245G** Octal NOR فشل بـ `-EINVAL` على Android TV بـ **Allwinner H616**

#### السياق
Android TV box بتستخدم **Allwinner H616** مع **Macronix MX25UM51245G** Octal DTR SPI NOR Flash للحصول على أعلى read bandwidth. الـ bringup engineer حاول يفعّل 8D-8D-8D mode للحصول على 200MB/s read speed، بس الـ `spi-nor` driver بيطبع error ومش بيتفعّل الـ mode.

#### المشكلة
الـ `spi_mem_exec_op()` بيرجع `-EINVAL` أو `-EOPNOTSUPP` فورًا عند محاولة تنفيذ Octal DTR operation.

#### التحليل
اتبّع مسار الـ validation في `spi-mem.c`:

**خطوة 1: `spi_mem_check_op()`**

```c
static int spi_mem_check_op(const struct spi_mem_op *op)
{
    if (!spi_mem_buswidth_is_valid(op->cmd.buswidth) ||
        !spi_mem_buswidth_is_valid(op->addr.buswidth) ||
        !spi_mem_buswidth_is_valid(op->dummy.buswidth) ||
        !spi_mem_buswidth_is_valid(op->data.buswidth))
        return -EINVAL;
```

الـ `spi_mem_buswidth_is_valid()`:

```c
static bool spi_mem_buswidth_is_valid(u8 buswidth)
{
    /* hweight8 بيحسب عدد bits=1 في القيمة */
    /* buswidth لازم تكون power of 2: 1, 2, 4, 8 */
    if (hweight8(buswidth) > 1 || buswidth > SPI_MEM_MAX_BUSWIDTH)
        return false;
    return true;
}
```

**خطوة 2: `spi_mem_default_supports_op()`**

```c
bool spi_mem_default_supports_op(struct spi_mem *mem,
                                  const struct spi_mem_op *op)
{
    bool op_is_dtr =
        op->cmd.dtr || op->addr.dtr || op->dummy.dtr || op->data.dtr;

    if (op_is_dtr) {
        if (!spi_mem_controller_is_capable(ctlr, dtr))
            return false;  /* الـ H616 SPI controller مش معاه dtr=true */

        /* 8D-8D-8D specific checks */
        if (op->cmd.dtr && op->cmd.buswidth == 8) {
            if (op->cmd.nbytes != 2)  /* لازم 2 bytes command */
                return false;

            if ((op->addr.nbytes % 2) ||
                (op->dummy.nbytes % 2) ||
                (op->data.nbytes % 2)) {
                dev_err(&ctlr->dev,
                    "Even byte numbers not allowed in octal DTR operations\n");
                return false;
            }
        }
    }
```

**السبب الجذري:** الـ Allwinner H616 SPI controller driver (في `spi-sun6i.c`) مش بيـregister أي `mem_ops` ولا بيـset `mem_caps`. يعني:

1. `ctlr->mem_ops` = NULL
2. `spi_mem_controller_is_capable(ctlr, dtr)` = false (لأن `ctlr->mem_caps` = NULL)
3. الـ `op_is_dtr = true` → `return false`

حتى لو اتعدى الـ DTR check، الـ H616 controller مش بيدعم Octal mode فعليًا في hardware.

#### الحل

**المرحلة 1: تشخيص كامل**

```bash
# شوف الـ SPI controller capabilities
cat /sys/bus/spi/devices/spi0/of_node/compatible
# sunxi,sun50i-h616-spi

# تحقق من mode المسجّل للـ flash
cat /sys/bus/spi/devices/spi0.0/of_node/spi-rx-bus-width
cat /sys/bus/spi/devices/spi0.0/of_node/spi-tx-bus-width

# فعّل verbose SPI logging
echo 8 > /proc/sys/kernel/printk
modprobe spi-nor dyndbg=+p
```

**المرحلة 2: الـ device tree fix** — نزّل الـ mode لـ Quad (4-1-4) اللي H616 يقدر يتعامل معاه:

```dts
&spi0 {
    status = "okay";
    flash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;
        spi-max-frequency = <100000000>;
        /* لا octal ولا DTR — H616 SPI controller ماعندوش دعم */
        spi-rx-bus-width = <4>;
        spi-tx-bus-width = <1>;
        m25p,fast-read;
    };
};
```

**المرحلة 3: لو الأداء مطلوب جدًا**، ابدل بـ controller بيدعم Octal DTR فعليًا (e.g., خدمة OSPI controller مختلفة)، أو استخدم الـ Macronix chip في 4-4-4 STR mode بدل 8D-8D-8D اللي بيطلب hardware support خاص.

**لفهم ليه الـ fallback بيشتغل:**

```c
/* في spi_mem_dirmap_create() */
if (ret) {
    desc->nodirmap = true;
    if (!spi_mem_supports_op(desc->mem, &desc->info.op_tmpl))
        ret = -EOPNOTSUPP;  /* الـ op مش مدعومة حتى في software fallback */
    else
        ret = 0;  /* software fallback ممكن */
}
```

لو `spi_mem_supports_op()` رجعت false (بسبب DTR check)، الـ `dirmap_create()` هترجع `-EOPNOTSUPP` تمامًا — مفيش fallback.

#### الدرس المستفاد
الـ `spi_mem_default_supports_op()` بتعمل multi-level validation: DTR support، parity of bytes في 8D mode، buswidth validity، وـ frequency limits. كل level لازم يتحقق قبل ما تفترض إن controller يدعم mode معين. دايمًا اتحقق من `mem_caps` اللي الـ controller driver بيـregister بيها — لو NULL، controller في basic mode بس.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

| المقال | الرابط |
|--------|--------|
| spi-mem: Add driver for Aspeed SMC controllers | [lwn.net/Articles/886809](https://lwn.net/Articles/886809/) |
| spi-mem: Convert Aspeed SMC driver to spi-mem | [lwn.net/Articles/889233](https://lwn.net/Articles/889233/) |
| spi-mem: Add driver for NXP FlexSPI controller | [lwn.net/Articles/768160](https://lwn.net/Articles/768160/) |
| Porting ASPEED FMC/SPI memory controller driver | [lwn.net/Articles/835858](https://lwn.net/Articles/835858/) |
| simple SPI framework (2005 — الـ SPI core الأصلي) | [lwn.net/Articles/154425](https://lwn.net/Articles/154425/) |
| SPI core refresh | [lwn.net/Articles/162268](https://lwn.net/Articles/162268/) |

---

### توثيق kernel الرسمي

```
Documentation/spi/spi-summary.rst      ← نظرة عامة على subsystem الـ SPI
Documentation/spi/spi-locking.rst      ← قواعد الـ locking في الـ SPI bus
include/linux/spi/spi-mem.h            ← تعريف كل الـ structs والـ API الخاصة بـ spi-mem
include/linux/spi/spi.h                ← الـ API الأساسي لـ SPI controller/device
drivers/spi/spi-mem.c                  ← الـ source الرئيسي للـ spi-mem layer
```

الـ online documentation:
- [Overview of Linux kernel SPI support](https://docs.kernel.org/spi/spi-summary.html)

---

### الـ patchwork — مناقشات الـ mailing list

دي أهم الـ patch series اللي أسست الـ spi-mem layer من أول ما ابتدأت:

| الـ patch | الرابط |
|-----------|--------|
| RFC — Extend the core to ease integration of SPI memory controllers | [patchwork.kernel.org](https://patchwork.kernel.org/project/spi-devel-general/patch/20180205232120.5851-2-boris.brezillon@bootlin.com/) |
| v2 — Extend the core to ease integration of SPI memory controllers | [patchwork.kernel.org](https://patchwork.kernel.org/project/spi-devel-general/patch/20180410224439.9260-5-boris.brezillon@bootlin.com/) |
| v3 — spi-mem: Add a new API to support direct mapping | [patchwork.kernel.org](https://patchwork.kernel.org/project/spi-devel-general/patch/20181106160536.13415-5-boris.brezillon@bootlin.com/) |
| v3 cover letter — spi-mem: Add a direct mapping API | [patchwork.kernel.org](https://patchwork.kernel.org/project/spi-devel-general/cover/20181106160536.13415-1-boris.brezillon@bootlin.com/) |
| RFC — mtd: spi-nor: Proposal for 8-8-8 mode support | [patchwork.kernel.org](https://patchwork.kernel.org/project/spi-devel-general/cover/20181012084825.23697-1-boris.brezillon@bootlin.com/) |
| eeprom: at25: Convert the driver to the spi-mem interface | [patchwork.kernel.org](https://patchwork.kernel.org/project/spi-devel-general/patch/20190330141637.22632-1-boris.brezillon@collabora.com/) |

> الـ spi-mem layer اتضافت في **Linux 4.18** (2018) من Boris Brezillon من Bootlin كـ RFC و merge فيها.

---

### الـ source code على git.kernel.org

```
# الـ spi-mem.c الأصلي على kernel.org Elixir browser
https://elixir.bootlin.com/linux/latest/source/drivers/spi/spi-mem.c

# الـ spi-mem.h header
https://github.com/torvalds/linux/blob/master/include/linux/spi/spi-mem.h

# الـ spi-mem.c على GitHub
https://github.com/torvalds/linux/blob/master/drivers/spi/spi-mem.c
```

---

### مقالات Bootlin (الـ author الأصلي)

**الـ Bootlin** هي الشركة اللي كتبت الـ spi-mem layer من الأساس. عندهم مقال تفصيلي:

- [spi-mem: bringing some consistency to the SPI memory ecosystem](https://bootlin.com/blog/spi-mem-bringing-some-consistency-to-the-spi-memory-ecosystem/)

وعندهم slides من مؤتمر **Embedded Linux Conference Europe 2018**:

- [SPI Memory support in Linux and U-Boot — ELC Europe 2018 (PDF)](https://elinux.org/images/7/73/Raynal-spi-memories.pdf)
- [An Introduction to SPI-NOR Subsystem (PDF)](https://elinux.org/images/4/4e/An_Introduction_to_SPI-NOR_Subsystem_-_v3_0.pdf)

---

### eLinux.org

| الصفحة | الرابط |
|--------|--------|
| RPi SPI — شرح SPI على Raspberry Pi | [elinux.org/RPi_SPI](https://elinux.org/RPi_SPI) |
| SPI Memory Support in Linux and U-Boot (Slides PDF) | [elinux.org](https://elinux.org/images/7/73/Raynal-spi-memories.pdf) |
| SPI-NOR Subsystem Introduction (PDF) | [elinux.org](https://elinux.org/images/4/4e/An_Introduction_to_SPI-NOR_Subsystem_-_v3_0.pdf) |

---

### kernelnewbies.org

| الـ kernel version | الـ feature المضافة | الرابط |
|--------------------|---------------------|--------|
| Linux 4.18 | إضافة الـ spi-mem layer لأول مرة + direct mapping API | [kernelnewbies.org/Linux_4.18](https://kernelnewbies.org/Linux_4.18) |
| Linux 4.14 | إضافة Microchip sst26vf064b QSPI memory | [kernelnewbies.org/Linux_4.14](https://kernelnewbies.org/Linux_4.14) |
| Linux 6.14 | spi-nand/spi-mem DTR support | [kernelnewbies.org/Linux_6.14](https://kernelnewbies.org/Linux_6.14) |

---

### كتب مُوصى بيها

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 14**: The Linux Device Model — أساسي لفهم الـ `struct device` و `devres`
- **الفصل 15**: Memory Mapping and DMA — لازم لفهم `spi_controller_dma_map_mem_op_data()`
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 11**: Timers and Time Management — لفهم `read_poll_timeout()` في `spi_mem_poll_status()`
- **الفصل 12**: Memory Management — لفهم `kzalloc()` وـ `GFP_DMA`
- **الفصل 17**: Devices and Modules — لفهم `EXPORT_SYMBOL_GPL` والـ driver registration

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 10**: MTD Subsystem — لأن الـ spi-mem بتخدم mtd/spi-nor/spi-nand
- **الفصل 15**: Debugging Embedded Linux — مفيد لفهم الـ `trace_spi_mem_start_op()`

---

### search terms للبحث عن معلومات أكتر

```
# للبحث في الـ mailing list
linux-spi spi-mem exec_op
linux kernel spi_mem_op direct mapping dirmap
spi-mem poll_status hardware offload
spi controller mem_ops interface

# للبحث عن الـ drivers اللي بتستخدم الـ spi-mem
linux kernel spi-nor spi-mem migration
linux nand chip spi-mem spinand
linux cadence qspi spi-mem dirmap

# للـ DTR (Double Transfer Rate) support
linux spi-mem dtr octal 8-8-8 mode
linux spi-mem swap16 dtr support

# للـ tracing
linux ftrace spi-mem trace events
```
## Phase 8: Writing simple module

### الفكرة

**الـ** `spi-mem.c` بيعرّف tracepoint اسمه `spi_mem_exec_op` — بس الأنسب والأكثر فايدة هو الـ kprobe على `spi_mem_exec_op` لأنه:

- مُصدَّر بـ `EXPORT_SYMBOL_GPL` فبالتالي رمزه موجود في الـ kallsyms.
- بيُنفَّذ عند **كل** عملية SPI memory (قراية، كتابة، أوامر Flash) — مش بس الـ fast path.
- بيحمل `struct spi_mem_op` كامل فيه الـ opcode، العنوان، عدد البايتات، والاتجاه.

الـ module هيستخدم **kprobe** على `spi_mem_exec_op` ويطبع الـ opcode وعدد بايتات البيانات وجهة النقل في كل مرة تُستدعى الدالة.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * spi_mem_exec_op kprobe monitor
 * Hooks spi_mem_exec_op() and prints SPI memory operation details.
 */
#include <linux/kernel.h>      /* pr_info, pr_err */
#include <linux/module.h>      /* MODULE_* macros, module_init/exit */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe */
#include <linux/spi/spi-mem.h> /* struct spi_mem, struct spi_mem_op */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on spi_mem_exec_op to log SPI memory operations");

/*
 * pre_handler is called just before spi_mem_exec_op() executes.
 * regs holds the CPU register state at the probe point:
 *   - On x86_64: rdi = mem, rsi = op
 *   - On ARM64:  x0  = mem, x1  = op
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Extract function arguments from registers */
#ifdef CONFIG_X86_64
    struct spi_mem     *mem = (struct spi_mem *)regs->di;
    struct spi_mem_op  *op  = (struct spi_mem_op *)regs->si;
#elif defined(CONFIG_ARM64)
    struct spi_mem     *mem = (struct spi_mem *)regs->regs[0];
    struct spi_mem_op  *op  = (struct spi_mem_op *)regs->regs[1];
#else
    /* Unsupported arch: skip logging */
    return 0;
#endif

    /* Validate pointers before dereferencing in kernel context */
    if (!mem || !op)
        return 0;

    pr_info("spi_mem_probe: dev=%s opcode=0x%02x addr_bytes=%u "
            "addr_val=0x%llx dummy=%u data=%u dir=%s buswidth=%u/%u/%u/%u\n",
            mem->name ? mem->name : "?",   /* SPI memory device name   */
            op->cmd.opcode,                /* e.g. 0x03 = READ, 0x02 = WRITE */
            op->addr.nbytes,               /* how many address bytes   */
            (unsigned long long)op->addr.val,
            op->dummy.nbytes,              /* dummy/wait cycles        */
            op->data.nbytes,               /* payload size in bytes    */
            /* direction as string */
            op->data.dir == SPI_MEM_DATA_IN  ? "IN"  :
            op->data.dir == SPI_MEM_DATA_OUT ? "OUT" : "NONE",
            op->cmd.buswidth,              /* cmd  IO lines (1/2/4/8) */
            op->addr.buswidth,             /* addr IO lines            */
            op->dummy.buswidth,            /* dummy IO lines           */
            op->data.buswidth);            /* data IO lines            */

    return 0; /* 0 = let the original function continue normally */
}

/* kprobe descriptor — we only need pre_handler for read-only monitoring */
static struct kprobe kp = {
    .symbol_name = "spi_mem_exec_op", /* symbol to hook; must be in kallsyms */
    .pre_handler = handler_pre,
};

static int __init spi_mem_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("spi_mem_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("spi_mem_probe: hooked spi_mem_exec_op at %p\n", kp.addr);
    return 0;
}

static void __exit spi_mem_probe_exit(void)
{
    /*
     * unregister_kprobe() removes the breakpoint instruction and waits
     * until any running handler finishes — prevents use-after-free on
     * module unload.
     */
    unregister_kprobe(&kp);
    pr_info("spi_mem_probe: unhooked spi_mem_exec_op\n");
}

module_init(spi_mem_probe_init);
module_exit(spi_mem_probe_exit);
```

---

### Makefile للبناء

```makefile
obj-m += spi_mem_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

```bash
# بناء وتحميل الـ module
make
sudo insmod spi_mem_probe.ko
# شوف الـ logs
sudo dmesg | grep spi_mem_probe
# إزالة الـ module
sudo rmmod spi_mem_probe
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|--------|-------|
| `<linux/kernel.h>` | `pr_info` / `pr_err` للطباعة في الـ kernel log |
| `<linux/module.h>` | ماكروهات `MODULE_LICENSE`، `module_init`، `module_exit` |
| `<linux/kprobes.h>` | `struct kprobe`، `register_kprobe`، `unregister_kprobe` |
| `<linux/spi/spi-mem.h>` | تعريف `struct spi_mem_op` و `enum spi_mem_data_dir` |

**الـ** `<linux/spi/spi-mem.h>` ضروري عشان نفهم تركيبة الـ `op` اللي بنستقبلها من الـ registers — من غيره هنتعامل مع pointer عمياني.

---

#### الـ `handler_pre` — الـ callback

الـ `pt_regs` هو snapshot للـ CPU registers عند لحظة الـ probe. الـ calling convention بيخلي أول argument في `rdi` (x86_64) أو `x0` (ARM64)، وده بالظبط `mem`؛ وتاني argument في `rsi`/`x1` وده `op`.

بنطبع:
- **الـ** `op->cmd.opcode`: الـ command byte اللي بيتبعت للـ Flash (مثلاً `0x03` للقراءة، `0x06` لتمكين الكتابة، `0xD8` لمسح sector).
- **الـ** `op->addr.val` وعدد بايتاتها: عشان نعرف أنهي عنوان في الـ Flash اللي بيتوصل.
- **الـ** `op->data.nbytes` + `dir`: نعرف حجم وجهة الـ payload.
- الـ buswidth لكل مرحلة: يكشف لو الـ controller بيشتغل بـ Quad SPI أو Octal.

الـ function بترجع `0` دايماً عشان **مش بتعدّل** على تدفق التنفيذ — الكود الأصلي بيكمل طبيعي.

---

#### الـ `kp` struct

```c
static struct kprobe kp = {
    .symbol_name = "spi_mem_exec_op",
    .pre_handler = handler_pre,
};
```

الـ `symbol_name` بيخلّي الـ kprobe يبحث عن رمز الدالة في `/proc/kallsyms` تلقائياً — مش محتاجين نحدد عنوان يدوياً. الـ `pre_handler` بيتنفذ **قبل** الدالة، وده كافي لأننا بس عايزين نشوف الـ arguments.

---

#### الـ `module_init` / `module_exit`

**الـ** `register_kprobe` بتزرع breakpoint instruction في الـ kernel code مكان بداية `spi_mem_exec_op`. لو فشلت (مثلاً الـ symbol مش موجود أو الـ CONFIG_KPROBES مش مفعّل) بترجع error ونوقف التحميل.

**الـ** `unregister_kprobe` في الـ exit **لازم** تتنفذ قبل تفريغ الـ module — لو الـ module اتفرغ والـ breakpoint لسه موجود، الـ kernel هيحاول يجري handler مش موجود ويحصل kernel panic. الدالة كمان بتنتظر أي handler شغّال حالياً يخلّص قبل ما ترجع.
