## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي ليه الملف

الملف `include/linux/spi/spi-mem.h` جزء من **SPI SUBSYSTEM** في الـ Linux kernel، والـ maintainer الرئيسي هو Mark Brown. الملف بالتحديد بيخص الـ **SPI MEM layer** — طبقة وسيطة فوق الـ SPI core مخصصة للتعامل مع ذاكرة الـ SPI (Flash, NAND, NOR).

---

### المشكلة اللي بيحلها — القصة من الأول

تخيّل عندك روتر في البيت، جواه شريحة Flash صغيرة بتحفظ عليها الـ firmware. الشريحة دي بتتكلم بروتوكول اسمه **SPI** (Serial Peripheral Interface) — أربع خطوط بس: Clock, MOSI, MISO, CS.

في الأول، كان البروتوكول بسيط: ابعت byte، استقبل byte. ونظام الـ Linux القديم كان بيعامل الـ Flash زي أي device SPI عادي — يعني كل driver بيكتب كود خاص بيه يبعت الـ opcodes يدويًا.

المشكلة إن الـ Flash chips تطورت:

| الجيل | عدد الخطوط للبيانات | السرعة | الاسم |
|---|---|---|---|
| SPI عادي | 1-bit | بطيء | Standard SPI |
| Dual SPI | 2-bit | ضعف | Dual I/O |
| **Quad SPI (QSPI)** | **4-bit** | **4x** | **QSPI** |
| **Octal SPI** | **8-bit** | **8x** | **OctaSPI** |
| **DTR Mode** | 8-bit × 2 edges | **16x** | **Octal DTR** |

لما جاء الـ QSPI، كل chip manufacturer عمل driver منفصل بيبعت الـ opcodes بطريقته. النتيجة: كود متكرر في كل مكان، وكل controller بيتعامل مع الـ Flash بطريقة مختلفة.

**الحل:** Boris Brezillon في 2018 عمل الـ **SPI MEM layer** — طبقة abstraction موحدة بتوصف "عملية الذاكرة" كـ struct واحدة بدل ما كل حد يكتب كوده الخاص.

---

### الصورة الكبيرة — ELI5

تخيّل إنك بتطلب بيتزا من مطعم:

1. **بتقول الطلب** (Command/Opcode): "عايز بيتزا"
2. **بتقول العنوان** (Address): "في شارع X، رقم Y"
3. **بتستنى شوية** (Dummy bytes): المطعم بيجهز الأوردر
4. **بييجيلك الأكل** (Data): البيتزا نفسها

الـ **`struct spi_mem_op`** في الملف ده هو بالظبط نفس الفكرة — بيوصف "عملية ذاكرة SPI" في 4 phases:

```
[CMD opcode] → [ADDRESS bytes] → [DUMMY cycles] → [DATA in/out]
```

قبل الـ SPI MEM layer، كل driver لازم يعرف يترجم الطلب ده لـ SPI transfers بنفسه. دلوقتي، الـ driver بس بيملا الـ struct، والـ framework بيتولى الباقي.

---

### ليه الملف ده مهم؟

**الـ `spi-mem.h` هو عقد (contract) بين 3 أطراف:**

```
┌─────────────────────────────────────────────────────┐
│              SPI Memory Device Driver               │
│   (مثلاً: SPI NOR driver, SPI NAND driver)         │
│   بيبني spi_mem_op ويطلب تنفيذها                   │
└──────────────────────┬──────────────────────────────┘
                       │  spi_mem_exec_op()
                       ▼
┌─────────────────────────────────────────────────────┐
│              SPI MEM Layer  (spi-mem.c)              │
│   بيتحقق من الـ op، بيعمل DMA mapping،              │
│   بيفوّض للـ controller أو بيعمل fallback           │
└──────────────────────┬──────────────────────────────┘
                       │  mem_ops->exec_op()
                       ▼
┌─────────────────────────────────────────────────────┐
│           SPI Controller Driver                     │
│   (مثلاً: Cadence QSPI, Qualcomm QSPI)             │
│   بينفذ الـ op على الـ hardware الفعلي             │
└─────────────────────────────────────────────────────┘
```

---

### المفاهيم الأساسية في الملف

#### 1. الـ `struct spi_mem_op` — قلب كل حاجة

```c
struct spi_mem_op {
    struct { u8 nbytes; u8 buswidth; u8 dtr; u16 opcode; } cmd;    // الأمر
    struct { u8 nbytes; u8 buswidth; u8 dtr; u64 val; }   addr;    // العنوان
    struct { u8 nbytes; u8 buswidth; u8 dtr; }             dummy;  // dummy cycles
    struct { u8 buswidth; u8 dtr; u8 ecc; enum dir; void *buf; } data; // البيانات
    unsigned int max_freq;  // أقصى تردد للعملية دي
};
```

كل field فيه `buswidth` — عدد الخطوط: 1 (SPI عادي)، 2 (Dual)، 4 (Quad)، أو 8 (Octal). وفيه `dtr` — هل الـ data بتتبعت على الـ rising AND falling edge (Double Transfer Rate).

#### 2. الـ `struct spi_controller_mem_ops` — ما يقدر يعمله الـ controller

```c
struct spi_controller_mem_ops {
    bool (*supports_op)(...);    // هل الـ controller يدعم الـ op دي؟
    int  (*exec_op)(...);        // نفّذ الـ op
    int  (*dirmap_create)(...);  // أنشئ Direct Mapping (اختياري)
    ssize_t (*dirmap_read)(...); // اقرا عن طريق الـ mapping
    ssize_t (*dirmap_write)(...);// اكتب عن طريق الـ mapping
    int  (*poll_status)(...);    // انتظر حتى الـ Flash يخلص عملية
};
```

#### 3. الـ Direct Mapping — الـ Fast Path

بعض الـ QSPI controllers زي Cadence بتقدر تعمل **memory-mapped I/O** مباشرة — يعني بتربط جزء من الذاكرة بالـ Flash، وأي قراءة من الـ address ده بتتحول أوتوماتيك لـ SPI read. ده بيخلي الـ CPU يقرا الـ Flash كأنه RAM عادي.

```
struct spi_mem_dirmap_desc ← بيوصف المنطقة المـ mapped
struct spi_mem_dirmap_info ← الـ op template + offset + length
```

لو الـ controller مش بيدعم Direct Mapping، الـ layer بتعمل **fallback** تلقائي لـ `spi_mem_exec_op()` — يعني نفس الـ driver بيشتغل على أي controller من غير ما تغير كود.

#### 4. الـ DTR و ECC و swap16

- **DTR (Double Transfer Rate):** بعت data على الـ rising والـ falling edge — بيضاعف الـ bandwidth.
- **ECC:** الـ controller بيعمل Error Correction تلقائي (مهم جداً في الـ NAND).
- **swap16:** بعض الـ Octal DTR chips بتعكس ترتيب الـ bytes في الـ 16-bit word عن الـ STR mode.

---

### الملفات المرتبطة اللي لازم تعرفها

#### الـ Core

| الملف | الدور |
|---|---|
| `include/linux/spi/spi-mem.h` | الـ API والـ structs — الملف ده |
| `drivers/spi/spi-mem.c` | التنفيذ الفعلي لكل functions |
| `include/trace/events/spi-mem.h` | Tracepoints للـ debugging |
| `include/linux/spi/spi.h` | الـ SPI core layer اللي بيبني عليه |

#### الـ SPI Memory Device Drivers (المستخدمين)

| الملف | الدور |
|---|---|
| `drivers/mtd/spi-nor/core.c` | SPI NOR Flash driver (W25Q, MX25L, ...) |
| `include/linux/mtd/spi-nor.h` | SPI NOR structs |
| `drivers/mtd/nand/spi/core.c` | SPI NAND Flash driver |

#### أمثلة على الـ SPI Controllers اللي بتنفذ `mem_ops`

| الملف | الـ Hardware |
|---|---|
| `drivers/spi/spi-cadence-quadspi.c` | Cadence QSPI (شايع في Xilinx/Intel FPGAs) |
| `drivers/spi/spi-fsl-qspi.c` | NXP FlexSPI |
| `drivers/spi/spi-bcm-qspi.c` | Broadcom QSPI |
| `drivers/spi/spi-geni-qcom.c` | Qualcomm GENI SPI |

---

### ملخص الهدف

الـ `spi-mem.h` بيحل مشكلة **code duplication** و **hardware abstraction** في عالم الـ SPI Flash. بدله، كل حد كان لازم يكتب كود منفصل لكل combination من (Flash chip × SPI controller). دلوقتي:

- الـ **Flash driver** بيقول "عايز أقرا من address X بـ Quad mode"
- الـ **SPI MEM layer** بيتحقق إن الـ controller بيدعم ده
- الـ **Controller driver** بينفذ على الـ hardware

ثلاث طبقات منفصلة، كل واحدة مسؤولة عن جزء واحد بس.
## Phase 2: شرح الـ SPI-MEM Framework

### المشكلة — ليه الـ Framework ده موجود؟

الـ **SPI subsystem** الأصلي في الكيرنل اتصمم لنقل بيانات عام بين أي SPI device وأي SPI controller.
الـ API بتاعه بيتكلم بـ `spi_transfer` و `spi_message` — يعني أنت بتحط bytes في buffer وبتبعتهم.

المشكلة ظهرت مع جيل الـ **SPI memory devices** زي QSPI NOR Flash و SPI NAND وغيرهم.
الأجهزة دي مش مجرد SPI devices — هي **memories بتتكلم بـ protocol محدد** جداً:

```
[ CMD byte ] → [ ADDRESS bytes ] → [ DUMMY cycles ] → [ DATA ]
```

وفوق كده، الـ controllers الجديدة زي **QSPI controllers** مش بس بتبعت bits — هي عندها:
- **Hardware acceleration** لتنفيذ الـ memory protocol نفسه
- **Direct Memory Mapping** — يعني ممكن تعمل memory-map للـ Flash في address space البروسيسور مباشرة
- دعم لـ **Quad SPI** (4 lanes)، **Octal SPI** (8 lanes)، **DTR** (Double Transfer Rate)

لو استخدمت الـ legacy SPI API:
- كل driver هيحتاج يعمل manually الـ CMD + ADDR + DUMMY + DATA sequence
- الـ controller مش هيعرف إن الـ operation دي ممكن يعملها بـ hardware shortcut
- مفيش abstraction موحدة لـ dirmap (الـ memory mapping feature)
- كل NOR flash driver هيعمل نفس الكود بطرق مختلفة

**الـ spi-mem framework** جه عشان يحل المشكلة دي بتوحيد الـ interface.

---

### الحل — الـ Kernel بيعمل إيه؟

الـ kernel بيعمل **abstraction layer** فوق الـ raw SPI layer، الـ abstraction ده:

1. بيعرّف **`spi_mem_op`** — struct موحد بيوصف أي memory operation بأجزاؤها الأربعة: CMD + ADDR + DUMMY + DATA
2. بيعرّف **`spi_controller_mem_ops`** — vtable الـ controller بيملاها لو عنده capabilities زيادة
3. بيعمل **fallback تلقائي** — لو الـ controller مش عارف ينفذ الـ op بـ hardware acceleration، الـ framework بيرجع للـ raw SPI API تلقائياً
4. بيعمل **`spi_mem_dirmap`** — abstraction لـ direct memory mapping مع fallback تلقائي كمان

يعني الـ driver بتاع الـ Flash (consumer) بيكتب كود واحد — والـ framework هو اللي بيقرر هيستخدم الـ hardware path ولا الـ software fallback.

---

### التشبيه الحقيقي — مع Mapping كامل

تخيل إنك عندك **مطعم** (الـ flash driver) بيطلب أكل من **مطبخ مركزي** (الـ SPI controller).

| المطعم / الطلب | الـ Kernel Concept |
|---|---|
| قائمة الطلبات الموحدة (فورم محدد) | `struct spi_mem_op` — الـ CMD+ADDR+DUMMY+DATA |
| الـ waiter بيوصل الطلب | `spi_mem_exec_op()` |
| مطبخ عادي بيعمل كل حاجة manually | Legacy SPI controller (fallback path) |
| مطبخ بـ machines متخصصة | QSPI controller بـ `mem_ops` |
| الـ waiter بيسأل: "الطلب ده ممكن؟" | `spi_mem_supports_op()` |
| باب توصيل مباشر للمطعم (shortcut) | `dirmap` — direct memory mapping |
| لو الباب مش موجود، الـ waiter بيجيب | `nodirmap=1` + fallback لـ `spi_mem_exec_op()` |
| المطبخ بيقول "أقصى حاجة أعملها كذا" | `adjust_op_size()` — controller limitations |

الـ **قائمة الموحدة** دي هي قلب الموضوع — المطعم مش محتاج يعرف تفاصيل المطبخ، وكل المطابخ بتفهم نفس الفورم.

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     SPI Memory Consumers                        │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│   │  SPI NOR     │  │  SPI NAND    │  │  spi-mem test driver │  │
│   │  (m25p80,    │  │  (spinand)   │  │                      │  │
│   │   spi-nor)   │  │              │  │                      │  │
│   └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘  │
└──────────┼────────────────┼───────────────────────┼─────────────┘
           │                │                       │
           └────────────────┼───────────────────────┘
                            │  spi_mem_exec_op()
                            │  spi_mem_dirmap_read/write()
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                   spi-mem Framework Layer                       │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  spi_mem_exec_op()  →  supports_op()?  →  exec_op()     │   │
│  │                              │                           │   │
│  │                     NO (fallback) → spi_sync() raw SPI  │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  spi_mem_dirmap_read/write()                             │   │
│  │       │                                                  │   │
│  │   has dirmap? ──YES──→ dirmap_read/write() (HW path)    │   │
│  │       │                                                  │   │
│  │       NO → fallback to spi_mem_exec_op() (SW path)      │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│               SPI Controller Providers                          │
│                                                                 │
│  ┌───────────────────────┐    ┌───────────────────────────┐     │
│  │  Simple SPI Controller │    │  QSPI / Octal Controller  │     │
│  │  (no mem_ops)          │    │  (implements mem_ops)     │     │
│  │                        │    │                           │     │
│  │  Uses raw spi_sync()   │    │  .exec_op   = qspi_exec  │     │
│  │  as fallback           │    │  .dirmap_create = ...     │     │
│  │                        │    │  .supports_op  = ...      │     │
│  └───────────────────────┘    └───────────────────────────┘     │
│                                                                 │
│                    struct spi_controller                        │
│               .mem_ops  →  spi_controller_mem_ops              │
│               .mem_caps →  spi_controller_mem_caps             │
└─────────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ **`struct spi_mem_op`** هو قلب الـ framework كله.

الفكرة إن أي memory operation — مهما كانت — ممكن تتوصف بـ **4 phases متتالية**:

```c
struct spi_mem_op {
    struct { u8 nbytes; u8 buswidth; u8 dtr; u16 opcode; } cmd;    // Phase 1: Command
    struct { u8 nbytes; u8 buswidth; u8 dtr; u64 val;   } addr;   // Phase 2: Address
    struct { u8 nbytes; u8 buswidth; u8 dtr;            } dummy;  // Phase 3: Dummy cycles
    struct { u8 buswidth; u8 dtr; u8 ecc; u8 swap16;             // Phase 4: Data
             enum spi_mem_data_dir dir;
             unsigned int nbytes;
             union { void *in; const void *out; } buf;
    } data;
    unsigned int max_freq;
};
```

**مثال حقيقي** — قراءة من SPI NOR Flash بـ Quad mode:

```c
/* Read 256 bytes from address 0x1000 using QSPI (4-lane) */
struct spi_mem_op op = SPI_MEM_OP(
    SPI_MEM_OP_CMD(0xEB, 1),           /* CMD: 0xEB (Fast Read Quad I/O), 1 lane */
    SPI_MEM_OP_ADDR(3, 0x1000, 4),    /* ADDR: 3 bytes, value 0x1000, 4 lanes */
    SPI_MEM_OP_DUMMY(6, 4),            /* DUMMY: 6 cycles, 4 lanes */
    SPI_MEM_OP_DATA_IN(256, buf, 4)   /* DATA: 256 bytes IN, 4 lanes */
);

ret = spi_mem_exec_op(mem, &op);
```

الـ framework بياخد الـ struct ده وبيقرر:
- هل الـ controller قادر ينفذه؟ (`supports_op`)
- هل محتاج يضغط الـ size عشان controller limitations؟ (`adjust_op_size`)
- هل ينفذه بـ hardware path ولا fallback؟ (`exec_op` أو raw `spi_sync`)

---

### تشريح كامل للـ Structs وعلاقتها ببعض

```
struct spi_mem_driver
    └── struct spi_driver spidrv       ← inherits from SPI driver
    └── probe(struct spi_mem *mem)     ← called with spi_mem, not spi_device
    └── remove(struct spi_mem *mem)
    └── shutdown(struct spi_mem *mem)
              │
              │ framework allocates & passes
              ▼
         struct spi_mem
              ├── struct spi_device *spi      ← الـ underlying SPI device
              ├── void *drvpriv               ← driver private data (set/get via helpers)
              └── const char *name            ← custom name (من controller أو default)
                        │
                        │ mem->spi->controller->mem_ops
                        ▼
         struct spi_controller_mem_ops        ← vtable بيملاها الـ controller driver
              ├── adjust_op_size()            ← تعديل size على حسب الـ controller limits
              ├── supports_op()               ← هل الـ op مدعومة؟
              ├── exec_op()                   ← تنفيذ الـ op بـ hardware
              ├── get_name()                  ← اسم مخصص للـ device
              ├── dirmap_create()             ← إنشاء direct memory mapping
              ├── dirmap_destroy()            ← تدمير الـ mapping
              ├── dirmap_read()               ← قراءة عبر الـ mapping
              ├── dirmap_write()              ← كتابة عبر الـ mapping
              └── poll_status()               ← polling على status register

         struct spi_controller_mem_caps       ← capabilities الـ controller
              ├── dtr    ← يدعم Double Transfer Rate؟
              ├── ecc    ← يدعم Error Correction؟
              ├── swap16 ← يدعم byte swapping في Octal DTR؟
              └── per_op_freq ← ممكن يغير frequency لكل op؟
```

**الـ Direct Mapping structs:**

```
struct spi_mem_dirmap_info               ← المعلومات الأساسية للـ mapping
    ├── struct spi_mem_op op_tmpl        ← الـ op template اللي هتتنفذ كل قراءة/كتابة
    ├── u64 offset                       ← البداية في فضاء الـ flash
    └── u64 length                       ← حجم الـ region المـ mapped

struct spi_mem_dirmap_desc               ← الـ descriptor الكامل (ناتج dirmap_create)
    ├── struct spi_mem *mem              ← الـ device اللي تابع له
    ├── struct spi_mem_dirmap_info info  ← المعلومات فوق
    ├── unsigned int nodirmap            ← 1 لو الـ HW مش دعم → fallback mode
    └── void *priv                       ← controller-specific data (مثلاً iomem pointer)
```

---

### الـ DTR (Double Transfer Rate) — مفهوم متقدم

الـ **DTR** يعني إن البيانات بتتبعت على كل من الـ rising وalـ falling edge للـ clock — يعني الـ throughput بيتضاعف.

كل phase في الـ `spi_mem_op` عنده `dtr` bit مستقل:
```c
cmd.dtr   = 1;   /* CMD phase بيتبعت بـ DDR */
addr.dtr  = 1;   /* ADDR phase بيتبعت بـ DDR */
dummy.dtr = 1;
data.dtr  = 1;
```

في حالة الـ **Octal DTR** (8 lanes + DTR)، في quirk مهم: الـ `swap16` flag بيعالج مشكلة إن ترتيب الـ bytes بيتعكس بين الـ STR و DTR modes في بعض الـ chips (زي Micron Xcella series).

الـ `SPI_MEM_DTR_OP_RPT_CMD` macro بيكرر الـ opcode مرتين:
```c
#define SPI_MEM_DTR_OP_RPT_CMD(__opcode, __buswidth)  \
    { .nbytes = 2, .opcode = __opcode | __opcode << 8, ... }
```
ده عشان في Octal DTR، الـ controller بيبعت 2 bytes في نفس الوقت، فالـ opcode لازم يكون مكرر عشان الـ flash يشوفه صح على الـ two edges.

---

### الـ poll_status — لماذا هو مهم؟

بعد كل write أو erase، الـ flash محتاج وقت للتنفيذ. الـ driver محتاج يـ poll على الـ status register لحد ما تخلص.

في الـ legacy way: الـ CPU بيعمل loop ويبعت SPI ops.

الـ `poll_status` في `spi_controller_mem_ops` بيدي الـ controller فرصة يعمل الـ polling ده بـ hardware (بعض الـ QSPI controllers زي STM32 QSPI عندهم hardware polling engine).

```c
int (*poll_status)(struct spi_mem *mem,
                   const struct spi_mem_op *op,  /* op لقراءة الـ status */
                   u16 mask, u16 match,           /* (status & mask) == match → done */
                   unsigned long initial_delay_us,
                   unsigned long polling_rate_us,
                   unsigned long timeout_ms);
```

لو الـ controller مش عنده الـ hardware polling، الـ framework بيعمل software polling loop عادي.

---

### إيه اللي الـ Framework بيمتلكه vs إيه اللي بيفوّضه

| الـ Framework يمتلك | الـ Driver بيفوّضه |
|---|---|
| الـ `spi_mem_op` format الموحد | تفاصيل تنفيذ `exec_op` في الـ hardware |
| الـ fallback logic (SW path) | إمكانية الـ controller (`supports_op`) |
| الـ dirmap abstraction مع fallback | إنشاء الـ HW mapping (`dirmap_create`) |
| الـ DMA mapping helpers (`spi_controller_dma_map_mem_op_data`) | الـ DMA transfer نفسه |
| الـ `spi_mem_driver` registration | منطق الـ probe الخاص بكل device |
| الـ `adjust_op_size` orchestration | حدود الـ controller الفعلية |
| الـ default `supports_op` check | الـ capabilities الخاصة بالـ controller |
| الـ `poll_status` SW fallback | الـ HW polling engine لو موجود |

---

### مثال End-to-End: NOR Flash Read عبر الـ Framework

```c
/* في driver الـ SPI NOR flash */
static int spi_nor_read(struct spi_nor *nor, loff_t from, size_t len, u8 *buf)
{
    struct spi_mem_op op = SPI_MEM_OP(
        SPI_MEM_OP_CMD(nor->read_opcode, nor->reg_proto),
        SPI_MEM_OP_ADDR(nor->addr_nbytes, from, nor->read_proto),
        SPI_MEM_OP_DUMMY(nor->read_dummy, nor->read_proto),
        SPI_MEM_OP_DATA_IN(len, buf, nor->read_proto)
    );

    /* الـ framework هو اللي بيقرر: HW path أو SW fallback */
    return spi_mem_exec_op(nor->spimem, &op);
}
```

```
spi_mem_exec_op()
    │
    ├── spi_mem_adjust_op_size()        /* تعديل len لو controller بيه limit */
    │
    ├── ctlr->mem_ops->supports_op()?
    │         ├── YES → ctlr->mem_ops->exec_op()   /* HW fast path */
    │         └── NO  → spi_mem_fallback_exec()    /* raw spi_sync() */
    │
    └── return result to flash driver
```

---

### علاقة الـ spi-mem بـ Subsystems تانية

- **SPI subsystem** (`linux/spi/spi.h`): الـ base layer — الـ `spi_device` و `spi_controller` معرّفين هناك. الـ spi-mem بيبني فوقيه مباشرة.
- **MTD subsystem** (Memory Technology Devices): الـ consumers الرئيسيين للـ spi-mem هم الـ `spi-nor` و `spi-nand` drivers اللي بيعرضوا الـ flash كـ MTD device للـ kernel.
- **DMA subsystem**: الـ `spi_controller_dma_map_mem_op_data()` بيعمل scatter-gather mapping للـ data buffer — الـ `struct sg_table` جاية من الـ DMA/scatterlist subsystem.
- **Device Model** (`linux/device.h`): الـ `spi_mem_driver` في النهاية هو `platform_driver` disguised — الـ `devm_spi_mem_dirmap_create()` بتستخدم الـ devres mechanism للتنظيف التلقائي.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Enums والـ Flags والـ Config Options

#### `enum spi_mem_data_dir` — اتجاه البيانات

| القيمة | المعنى |
|--------|--------|
| `SPI_MEM_NO_DATA` | الـ operation مفيش فيها data transfer خالص |
| `SPI_MEM_DATA_IN` | بيانات جاية من الـ memory للـ CPU (Read) |
| `SPI_MEM_DATA_OUT` | بيانات رايحة من الـ CPU للـ memory (Write) |

---

#### الـ Capability Flags في `spi_controller_mem_caps`

| الـ Field | النوع | المعنى |
|-----------|-------|--------|
| `dtr` | `bool` | الـ controller بيدعم DTR (Double Transfer Rate — بيبعت على كل edge) |
| `ecc` | `bool` | بيدعم ECC hardware (error correction لـ NAND مثلاً) |
| `swap16` | `bool` | بيدعم swap الـ byte order في Octal DTR mode |
| `per_op_freq` | `bool` | ينفع يغير الـ frequency لكل operation لوحدها |

---

#### الـ Config Option الأساسي

| الـ Option | الأثر |
|-----------|-------|
| `CONFIG_SPI_MEM` | لو مش enabled، كل الـ DMA mapping functions بترجع `-ENOTSUPP` أو stub فاضية |

---

#### الـ Macros الأساسية للبناء السريع (Cheatsheet)

| الـ Macro | الاستخدام |
|-----------|-----------|
| `SPI_MEM_OP_CMD(opcode, bw)` | يبني الـ `cmd` phase بـ 1 byte و STR mode |
| `SPI_MEM_DTR_OP_RPT_CMD(opcode, bw)` | يبني الـ `cmd` phase بـ 2 bytes مكررة و DTR mode |
| `SPI_MEM_OP_ADDR(n, val, bw)` | يبني الـ `addr` phase عادي |
| `SPI_MEM_DTR_OP_ADDR(n, val, bw)` | يبني الـ `addr` phase بـ DTR |
| `SPI_MEM_DTR_OP_RPT_ADDR(val, bw)` | يبني الـ `addr` phase بـ 2 bytes مكررة و DTR |
| `SPI_MEM_OP_NO_ADDR` | الـ operation ملهاش address phase |
| `SPI_MEM_OP_DUMMY(n, bw)` | يبني الـ dummy cycles |
| `SPI_MEM_DTR_OP_DUMMY(n, bw)` | dummy cycles بـ DTR |
| `SPI_MEM_OP_NO_DUMMY` | ملهاش dummy phase |
| `SPI_MEM_OP_DATA_IN(n, buf, bw)` | data phase للقراءة |
| `SPI_MEM_DTR_OP_DATA_IN(n, buf, bw)` | data phase للقراءة بـ DTR |
| `SPI_MEM_OP_DATA_OUT(n, buf, bw)` | data phase للكتابة |
| `SPI_MEM_DTR_OP_DATA_OUT(n, buf, bw)` | data phase للكتابة بـ DTR |
| `SPI_MEM_OP_NO_DATA` | ملهاش data phase |
| `SPI_MEM_OP(__cmd,__addr,__dummy,__data,...)` | بيجمع الـ 4 phases في struct واحدة |
| `SPI_MEM_OP_MAX_FREQ(freq)` | بيحط قيود على الـ frequency لهذه الـ operation |
| `spi_mem_controller_is_capable(ctlr, cap)` | يتحقق إن الـ controller عنده capability معينة |
| `spi_mem_driver_register(drv)` | يسجل driver مع `THIS_MODULE` تلقائياً |
| `module_spi_mem_driver(drv)` | يولد `module_init`/`module_exit` تلقائياً |

---

### الـ Structs المهمة

---

#### 1. `struct spi_mem_op`

**الغرض:** وصف كامل لعملية SPI memory واحدة — بتعبّر عن الـ 4 phases اللي بيمر بيها أي command لـ SPI flash/NAND.

```
[CMD phase] → [ADDR phase] → [DUMMY cycles] → [DATA phase]
```

**الـ Fields:**

| الـ Phase | الـ Field | الوصف |
|----------|-----------|-------|
| `cmd` | `nbytes` | عدد bytes الـ opcode (1 أو 2 بس) |
| `cmd` | `buswidth` | عدد الـ IO lines للـ command (1/2/4/8) |
| `cmd` | `opcode` | الـ opcode نفسه (مثلاً `0x03` للقراءة) |
| `cmd` | `dtr` | 1-bit: هل الـ command بيتبعت بـ DTR؟ |
| `addr` | `nbytes` | عدد bytes العنوان (0 لو ملهيش address) |
| `addr` | `buswidth` | عدد الـ IO lines للعنوان |
| `addr` | `dtr` | 1-bit: هل العنوان بيتبعت بـ DTR؟ |
| `addr` | `val` | قيمة العنوان (u64، بيتبعت MSB-first) |
| `dummy` | `nbytes` | عدد dummy bytes (0 لو ملهيش) |
| `dummy` | `buswidth` | الـ IO lines للـ dummy cycles |
| `dummy` | `dtr` | 1-bit: هل الـ dummy بـ DTR؟ |
| `data` | `buswidth` | عدد الـ IO lines للبيانات |
| `data` | `dtr` | 1-bit: هل البيانات بـ DTR؟ |
| `data` | `ecc` | 1-bit: هل محتاج ECC hardware؟ |
| `data` | `swap16` | 1-bit: هل swap الـ byte order في Octal DTR؟ |
| `data` | `dir` | اتجاه البيانات (`spi_mem_data_dir`) |
| `data` | `nbytes` | عدد bytes البيانات |
| `data.buf` | `in` / `out` | union: buffer للقراءة أو الكتابة (DMA-able) |
| — | `max_freq` | أقصى frequency لهذه الـ operation (0 = no limit) |

**الصلة بالـ structs التانية:** بيتعملها `exec_op()` على `spi_mem`، وبيُستخدم كـ template جوه `spi_mem_dirmap_info`.

---

#### 2. `struct spi_mem_dirmap_info`

**الغرض:** بيوصف منطقة **Direct Mapping** — يعني جزء من ذاكرة الـ SPI device اللي الـ controller ممكن يعمله map مباشر في address space الـ CPU أو DMA.

| الـ Field | النوع | الوصف |
|-----------|-------|-------|
| `op_tmpl` | `struct spi_mem_op` | الـ template اللي الـ controller هيستخدمه لأي access على الـ mapping دي |
| `offset` | `u64` | الـ offset المطلق في الـ SPI memory اللي الـ mapping بتبدأ منه |
| `length` | `u64` | طول الـ mapping بالـ bytes |

**ملاحظة:** الـ `op_tmpl.data.dir` بيحدد هل الـ mapping دي للقراءة أو الكتابة — مش للاتنين مع بعض.

---

#### 3. `struct spi_mem_dirmap_desc`

**الغرض:** الـ descriptor الكامل لـ direct mapping instance — بيتعمل بواسطة `spi_mem_dirmap_create()`.

| الـ Field | النوع | الوصف |
|-----------|-------|-------|
| `mem` | `struct spi_mem *` | الـ SPI memory device اللي الـ mapping مربوطة بيه |
| `info` | `struct spi_mem_dirmap_info` | المعلومات اللي اتحطت وقت الإنشاء |
| `nodirmap` | `unsigned int` | لو = 1 معناه الـ controller مش بيدعم dirmap، هيعمل fallback لـ `exec_op()` |
| `priv` | `void *` | pointer لـ controller-specific data (مثلاً iomapped region) |

**الـ fallback mechanism:** لو `nodirmap == 1`، كل calls لـ `spi_mem_dirmap_read/write()` بتروح على `spi_mem_exec_op()` تلقائياً — فالـ driver مش محتاج يعمل handling خاص.

---

#### 4. `struct spi_mem`

**الغرض:** wrapper خفيف فوق `spi_device` — بيعبّر عن الـ SPI device من منظور memory protocol.

| الـ Field | النوع | الوصف |
|-----------|-------|-------|
| `spi` | `struct spi_device *` | الـ SPI device التحتي (فيه الـ bus config، الـ CS، الـ speed، إلخ) |
| `drvpriv` | `void *` | pointer خاص بالـ driver (يتعمله set/get بـ `spi_mem_set/get_drvdata()`) |
| `name` | `const char *` | اسم الـ device (ممكن يتعمل override من الـ controller) |

**Helper functions inline:**
```c
spi_mem_set_drvdata(mem, data);   // mem->drvpriv = data
spi_mem_get_drvdata(mem);         // return mem->drvpriv
```

---

#### 5. `struct spi_controller_mem_ops`

**الغرض:** جدول الـ function pointers اللي الـ SPI controller driver بيوفرها لـ SPI-mem core — زي vtable في C.

| الـ Callback | النوع | الوصف |
|-------------|-------|-------|
| `adjust_op_size` | `int (*)` | يقلل حجم الـ data transfer عشان يناسب قيود الـ controller (alignment، max RX/TX) |
| `supports_op` | `bool (*)` | يتحقق إن الـ controller ينفع ينفذ الـ operation دي |
| `exec_op` | `int (*)` | ينفذ الـ operation فعلياً — ده الأساسي |
| `get_name` | `const char *(*)` | يرجع اسم custom للـ device (للـ mtdparts compatibility) |
| `dirmap_create` | `int (*)` | ينشئ direct mapping — optional |
| `dirmap_destroy` | `void (*)` | يمسح direct mapping |
| `dirmap_read` | `ssize_t (*)` | يقرأ بيانات عبر الـ direct mapping |
| `dirmap_write` | `ssize_t (*)` | يكتب بيانات عبر الـ direct mapping |
| `poll_status` | `int (*)` | يعمل poll على status register لحد ما `(status & mask) == match` أو timeout |

**ملاحظة مهمة:** الـ `dirmap_read/write` ممكن يرجعوا أقل من الـ `len` المطلوب (لو الـ request عدى حدود الـ mapped area) — والـ caller مسؤول يكرر الـ call.

---

#### 6. `struct spi_controller_mem_caps`

**الغرض:** بيعلن قدرات الـ controller للـ SPI-mem core — الـ core بيستخدمها في `spi_mem_default_supports_op()` عشان يرفض الـ operations اللي الـ controller مش بيدعمها.

---

#### 7. `struct spi_mem_driver`

**الغرض:** wrapper فوق `spi_driver` — بيوفر interface موحد لـ drivers اللي بتتعامل مع SPI memory devices.

| الـ Field | النوع | الوصف |
|-----------|-------|-------|
| `spidrv` | `struct spi_driver` | الـ base SPI driver (فيه `probe`/`remove` على مستوى `spi_device`) |
| `probe` | `int (*)(struct spi_mem *)` | كشف وتهيئة الـ memory device |
| `remove` | `int (*)(struct spi_mem *)` | إزالة الـ device |
| `shutdown` | `void (*)(struct spi_mem *)` | استجابة لـ system shutdown |

الـ core هو اللي بيخلق الـ `spi_mem` object وبيمرره لـ `probe`/`remove`/`shutdown` — الـ driver مش محتاج يعمله alloc.

---

### مخطط علاقات الـ Structs (ASCII)

```
┌─────────────────────────────────────────────────────────────────┐
│                      spi_mem_driver                             │
│  ┌──────────────┐  probe(spi_mem*)                              │
│  │  spi_driver  │  remove(spi_mem*)                             │
│  │  (embedded)  │  shutdown(spi_mem*)                           │
│  └──────┬───────┘                                               │
│         │ inherits from                                         │
└─────────┼───────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────┐
│     spi_driver      │          ┌─────────────────────────────┐
│  id_table           │          │        spi_controller        │
│  probe(spi_device*) │          │  mem_ops ────────────────►  │
│  remove(...)        │          │  mem_caps ──────────────►   │
│  driver (embedded)  │          │  (+ bus_lock_mutex,         │
└─────────────────────┘          │     io_mutex, queue_lock)   │
                                 └───────────┬─────────────────┘
                                             │ owns
                                             ▼
┌─────────────────────┐         ┌────────────────────────────┐
│      spi_mem        │◄────────│       spi_device            │
│  spi ─────────────►│         │  controller ──────────────►│
│  drvpriv            │         │  dev (struct device)        │
│  name               │         │  max_speed_hz               │
└──────┬──────────────┘         └────────────────────────────┘
       │
       │ referenced by
       ▼
┌───────────────────────────────┐
│      spi_mem_dirmap_desc      │
│  mem ──────────────────────► spi_mem
│  info (spi_mem_dirmap_info)   │
│    └─ op_tmpl (spi_mem_op)    │
│    └─ offset, length          │
│  nodirmap (fallback flag)     │
│  priv (controller private)    │
└───────────────────────────────┘
          ▲
          │ created/destroyed by
          │
┌─────────────────────────────────────────────┐
│       spi_controller_mem_ops                │
│  adjust_op_size(spi_mem*, spi_mem_op*)      │
│  supports_op(spi_mem*, spi_mem_op*)         │
│  exec_op(spi_mem*, spi_mem_op*)             │
│  get_name(spi_mem*)                         │
│  dirmap_create(spi_mem_dirmap_desc*)        │
│  dirmap_destroy(spi_mem_dirmap_desc*)       │
│  dirmap_read(desc*, offs, len, buf)         │
│  dirmap_write(desc*, offs, len, buf)        │
│  poll_status(spi_mem*, op*, mask, match...) │
└─────────────────────────────────────────────┘

┌─────────────────────────────┐
│       spi_mem_op            │
│  cmd.{nbytes,buswidth,dtr,  │
│        opcode}              │
│  addr.{nbytes,buswidth,dtr, │
│         val}                │
│  dummy.{nbytes,buswidth,dtr}│
│  data.{buswidth,dtr,ecc,    │
│         swap16,dir,nbytes,  │
│         buf.{in,out}}       │
│  max_freq                   │
└─────────────────────────────┘
    ▲ embedded in spi_mem_dirmap_info.op_tmpl
    ▲ passed to exec_op(), supports_op(), adjust_op_size()

┌──────────────────────────────┐
│   spi_controller_mem_caps    │
│  dtr, ecc, swap16, per_op_freq│
└──────────────────────────────┘
    ▲ referenced by spi_mem_controller_is_capable(ctlr, cap)
```

---

### دورة حياة الـ Driver (Lifecycle Diagram)

#### تسجيل الـ Driver

```
module_spi_mem_driver(my_drv)
  └─► module_init → spi_mem_driver_register(&my_drv)
                      └─► spi_mem_driver_register_with_owner(&my_drv, THIS_MODULE)
                            └─► spi_register_driver(&my_drv.spidrv)
                                  └─► driver_register(&my_drv.spidrv.driver)
                                        └─► kernel matches against spi_device table
```

#### الـ Probe

```
kernel finds matching spi_device
  └─► spi_mem core allocates spi_mem
        └─► spi_mem->spi = matched spi_device
        └─► calls my_drv.probe(spi_mem)
              └─► driver initializes flash, registers MTD/NVMEM/etc.
```

#### تنفيذ عملية قراءة عادية

```
driver calls spi_mem_exec_op(mem, &op)
  └─► spi_mem_adjust_op_size(mem, &op)          [تقليص لو لازم]
        └─► ctlr->mem_ops->adjust_op_size(...)
  └─► spi_mem_supports_op(mem, &op)             [تحقق]
        └─► ctlr->mem_ops->supports_op(...)
  └─► spi_mem core handles bus locking
  └─► ctlr->mem_ops->exec_op(mem, &op)          [التنفيذ الفعلي]
        └─► controller hardware executes 4-phase transfer
```

#### إنشاء Direct Mapping

```
driver calls spi_mem_dirmap_create(mem, &info)
  └─► allocates spi_mem_dirmap_desc
  └─► if ctlr->mem_ops->dirmap_create exists:
        └─► ctlr->mem_ops->dirmap_create(desc)
              └─► controller maps region → desc->priv = iomapped addr
              └─► returns 0 on success
  └─► else: desc->nodirmap = 1   [fallback mode]
  └─► returns desc*
```

#### قراءة عبر Direct Mapping

```
driver calls spi_mem_dirmap_read(desc, offs, len, buf)
  └─► if desc->nodirmap == 1:
        └─► spi_mem_exec_op(desc->mem, adjusted_op)   [fallback]
  └─► else:
        └─► ctlr->mem_ops->dirmap_read(desc, offs, len, buf)
              └─► DMA transfer from mapped region
              └─► may return < len (crossing boundary)
              └─► caller loops if needed
```

#### إزالة الـ Driver

```
module_exit → spi_mem_driver_unregister(&my_drv)
  └─► spi_unregister_driver(&my_drv.spidrv)
        └─► driver_unregister(...)
              └─► kernel calls my_drv.remove(spi_mem) for each device
                    └─► driver frees resources
              └─► kernel frees spi_mem object
```

---

### مخطط تدفق الـ Calls (Call Flow Diagrams)

#### `spi_mem_exec_op()` — المسار الكامل

```
[Flash Driver]
    spi_mem_exec_op(mem, &op)
      │
      ├─ spi_mem_adjust_op_freq(mem, &op)
      │    └─ لو ctlr->mem_ops->adjust_op_size موجودة:
      │         ctlr->mem_ops->adjust_op_size(mem, &op)
      │
      ├─ spi_mem_supports_op(mem, &op)
      │    ├─ لو ctlr->mem_ops->supports_op موجودة:
      │    │    ctlr->mem_ops->supports_op(mem, &op)  → true/false
      │    └─ fallback: spi_mem_default_supports_op(mem, &op)
      │         └─ بيتحقق من dtr, ecc, swap16, per_op_freq
      │              بناءً على ctlr->mem_caps
      │
      ├─ [bus lock acquired]
      │
      ├─ ctlr->mem_ops->exec_op(mem, &op)
      │    └─ controller driver:
      │         ├─ [CMD phase]   → MOSI: opcode bytes
      │         ├─ [ADDR phase]  → MOSI: address MSB-first
      │         ├─ [DUMMY phase] → clock cycles, no data
      │         └─ [DATA phase]  → MOSI/MISO: data transfer
      │              └─ DMA or PIO depending on controller
      │
      └─ [bus lock released]
         return 0 or -errno
```

#### `spi_mem_poll_status()` — انتظار جهوزية الـ Flash

```
[Flash Driver waiting for erase/write complete]
    spi_mem_poll_status(mem, &op, mask, match,
                        initial_delay, poll_rate, timeout)
      │
      ├─ wait initial_delay_us
      │
      ├─ لو ctlr->mem_ops->poll_status موجودة:
      │    ctlr->mem_ops->poll_status(...)
      │      └─ hardware polling (interrupt-driven أو DMA)
      │
      └─ else: software polling loop
           ├─ spi_mem_exec_op(mem, &op)   [read status register]
           ├─ check: (status & mask) == match ?
           ├─     yes → return 0
           ├─     no  → wait polling_delay_us
           └─ timeout → return -ETIMEDOUT
```

#### `spi_controller_dma_map_mem_op_data()` — الـ DMA Mapping

```
[SPI Core before exec_op with DMA]
    spi_controller_dma_map_mem_op_data(ctlr, &op, &sg)
      │
      ├─ لو op.data.dir == SPI_MEM_DATA_IN:
      │    sg_init_one(sg, op.data.buf.in, op.data.nbytes)
      │    dma_map_sg(ctlr->dma_rx_dev, sg, ...)  → DMA_FROM_DEVICE
      │
      └─ لو op.data.dir == SPI_MEM_DATA_OUT:
           sg_init_one(sg, op.data.buf.out, op.data.nbytes)
           dma_map_sg(ctlr->dma_tx_dev, sg, ...)  → DMA_TO_DEVICE

    [after exec_op]
    spi_controller_dma_unmap_mem_op_data(ctlr, &op, &sg)
      └─ dma_unmap_sg(...)
```

---

### استراتيجية الـ Locking

الـ SPI-mem layer مش بيعرّف locks جديدة بنفسه — بيعتمد على الـ locks الموجودة في `spi_controller`.

#### جدول الـ Locks

| الـ Lock | النوع | موجود في | بيحمي إيه |
|---------|-------|----------|-----------|
| `io_mutex` | `struct mutex` | `spi_controller` | الـ physical bus access — منع تداخل الـ transfers |
| `bus_lock_mutex` | `struct mutex` | `spi_controller` | الـ exclusive bus locking لـ multi-transfer sequences |
| `bus_lock_spinlock` | `spinlock_t` | `spi_controller` | حماية الـ `bus_lock_flag` نفسه |
| `queue_lock` | `spinlock_t` | `spi_controller` | قايمة انتظار الـ messages |

#### ترتيب الـ Locking (Lock Ordering)

```
bus_lock_spinlock  →  bus_lock_mutex  →  io_mutex
    (irq-safe)          (sleepable)      (sleepable)
```

- **`bus_lock_spinlock`** بيتأخد الأول وبسرعة عشان يتحقق ويحمي الـ `bus_lock_flag`.
- **`bus_lock_mutex`** بيتأخد لما driver يحتاج يعمل سلسلة من operations متواصلة من غير تدخل.
- **`io_mutex`** بيتأخد لكل transfer فعلي على الـ bus.

#### في سياق الـ spi-mem

الـ `spi_mem_exec_op()` بيمر بالـ SPI core اللي بياخد `io_mutex` قبل ما يستدعي `exec_op()`. لو driver عامل `spi_bus_lock()`، الـ `bus_lock_mutex` بيتأخد كمان.

الـ `dirmap_read/write` لو عملت DMA — الـ DMA controller بيتعامل مع الـ memory بشكل async، ومفيش lock إضافي على الـ buffer، فالـ driver مسؤول إن الـ buffer DMA-safe ومش بيتعمل عليه access في نفس الوقت.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Inline Helpers

| Function | النوع | الغرض |
|---|---|---|
| `spi_mem_set_drvdata()` | `static inline void` | حفظ pointer خاص بالـ driver جوه `spi_mem` |
| `spi_mem_get_drvdata()` | `static inline void *` | استرجاع الـ pointer ده |

#### Core Operation API

| Function | الغرض المختصر |
|---|---|
| `spi_mem_adjust_op_size()` | تصغير الـ data xfer بحيث يتناسب مع حدود الـ controller |
| `spi_mem_adjust_op_freq()` | تعديل تردد الـ op بناءً على `max_freq` الـ controller |
| `spi_mem_calc_op_duration()` | حساب مدة تنفيذ الـ op بالـ microseconds |
| `spi_mem_supports_op()` | التحقق إن الـ controller يدعم operation معينة |
| `spi_mem_exec_op()` | تنفيذ SPI memory operation كاملة |
| `spi_mem_get_name()` | جلب اسم مخصص للـ SPI mem device |
| `spi_mem_poll_status()` | polling على status register لحد ما يتحقق شرط أو timeout |

#### Direct Mapping API

| Function | الغرض المختصر |
|---|---|
| `spi_mem_dirmap_create()` | إنشاء direct mapping descriptor |
| `spi_mem_dirmap_destroy()` | تدمير الـ descriptor |
| `spi_mem_dirmap_read()` | قراءة بيانات عبر الـ direct mapping |
| `spi_mem_dirmap_write()` | كتابة بيانات عبر الـ direct mapping |
| `devm_spi_mem_dirmap_create()` | إنشاء direct mapping مربوط بـ device lifecycle |
| `devm_spi_mem_dirmap_destroy()` | تدمير يدوي لـ devm-managed mapping |

#### DMA Helpers (CONFIG_SPI_MEM)

| Function | الغرض المختصر |
|---|---|
| `spi_controller_dma_map_mem_op_data()` | map بيانات الـ op لـ DMA scatter-gather |
| `spi_controller_dma_unmap_mem_op_data()` | unmap بيانات الـ op بعد الـ DMA |
| `spi_mem_default_supports_op()` | default checker للـ op support |

#### Driver Registration

| Function / Macro | الغرض المختصر |
|---|---|
| `spi_mem_driver_register_with_owner()` | تسجيل الـ driver مع تحديد الـ module owner |
| `spi_mem_driver_unregister()` | إلغاء تسجيل الـ driver |
| `spi_mem_driver_register()` | macro wrapper يمرر `THIS_MODULE` تلقائياً |
| `module_spi_mem_driver()` | macro يولد `module_init`/`module_exit` تلقائياً |

---

### Group 1: Inline Driver-Data Helpers

الـ `spi_mem` struct فيها field اسمه `drvpriv` عشان الـ upper-layer driver (زي NAND أو NOR driver) يخزن فيه pointer على state خاص بيه. الـ helpers دي مجرد accessors صريحة بدل الوصول المباشر للـ field.

---

#### `spi_mem_set_drvdata()`

```c
static inline void spi_mem_set_drvdata(struct spi_mem *mem, void *data)
{
    mem->drvpriv = data;
}
```

بتحفظ `data` جوه `mem->drvpriv`. بتتسمى عادةً في `probe()` بعد ما الـ driver يعمل `kzalloc` على الـ private struct بتاعته.

**Parameters:**
- `mem` — الـ SPI memory device الـ target
- `data` — الـ pointer اللي عايز تحفظه

**Return:** لا يرجع قيمة.

**Key details:** مفيش locking — الـ caller مسؤول إن الـ probe context single-threaded.

---

#### `spi_mem_get_drvdata()`

```c
static inline void *spi_mem_get_drvdata(struct spi_mem *mem)
{
    return mem->drvpriv;
}
```

بترجع الـ `drvpriv` pointer. بتتسمى في أي دالة في الـ driver بعد الـ probe.

**Parameters:**
- `mem` — الـ SPI memory device

**Return:** `void *` — الـ pointer اللي اتحفظ قبل كده، أو `NULL` لو مش متسبّح.

---

### Group 2: Operation Size & Frequency Adjustment

الـ controllers مش كلها بتدعم نفس size الـ transfer ونفس التردد. الـ group ده بيكيّف الـ `spi_mem_op` قبل إرساله للـ controller.

---

#### `spi_mem_adjust_op_size()`

```c
int spi_mem_adjust_op_size(struct spi_mem *mem, struct spi_mem_op *op);
```

لو الـ controller عنده `mem_ops->adjust_op_size`، بيستدعيه عشان يقلص `op->data.nbytes` لحد الـ maximum المسموح. لو مش موجودة يرجع `0` بدون تغيير.

**Parameters:**
- `mem` — الـ SPI memory device
- `op` — الـ operation — بيتعدل `data.nbytes` فيها مباشرة

**Return:** `0` نجاح، أو error code سالب.

**Key details:** لازم تتسمى قبل `spi_mem_exec_op()` لما تكون عارف إن الـ transfer ممكن تكون أكبر من الـ hardware limit. الـ NAND/NOR drivers بتعمل loop واستدعاء متكرر لحد ما تخلص كل الـ data.

**Pseudocode:**
```
if controller has .adjust_op_size:
    return controller->mem_ops->adjust_op_size(mem, op)
return 0
```

---

#### `spi_mem_adjust_op_freq()`

```c
void spi_mem_adjust_op_freq(struct spi_mem *mem, struct spi_mem_op *op);
```

لو الـ controller يدعم `per_op_freq` capability ولو `op->max_freq` مش صفر، بتعدل تردد الـ SPI device لهذه الـ operation. بتعكس التأثير بعد تنفيذ الـ op.

**Parameters:**
- `mem` — الـ SPI memory device
- `op` — الـ operation — بتقرأ `max_freq` منها

**Return:** لا يرجع قيمة.

**Key details:** الـ `spi_mem_controller_is_capable(ctlr, per_op_freq)` macro بيتستخدم للتحقق من الـ capability. لو `max_freq == 0` ما بتعملش حاجة.

---

#### `spi_mem_calc_op_duration()`

```c
u64 spi_mem_calc_op_duration(struct spi_mem *mem, struct spi_mem_op *op);
```

بتحسب المدة الزمنية التقريبية لتنفيذ الـ op بالـ microseconds بناءً على عدد الـ bits الكلي والتردد الحالي. مفيدة للـ timeout calculation في `spi_mem_poll_status()`.

**Parameters:**
- `mem` — بيستخدمه للوصول للـ `spi->max_speed_hz` كـ fallback
- `op` — بيحسب منها عدد الـ bytes في كل phase (cmd + addr + dummy + data)

**Return:** `u64` بالـ microseconds.

**Key details:** الحساب بيأخذ في الاعتبار الـ `buswidth` لكل phase و`dtr` mode إن كان موجود.

---

### Group 3: Operation Support Check & Execution

الـ group الأهم — ده اللي بيشيّك إن الـ controller يقدر ينفذ operation معينة وبعدين ينفذها فعلاً.

---

#### `spi_mem_supports_op()`

```c
bool spi_mem_supports_op(struct spi_mem *mem,
                         const struct spi_mem_op *op);
```

بتسأل الـ controller إذا كان يدعم الـ operation دي. لو الـ controller عنده `mem_ops->supports_op` بتستدعيه. لو مش موجود بتفول على `spi_mem_default_supports_op()`.

**Parameters:**
- `mem` — الـ SPI memory device
- `op` — الـ operation اللي بتختبرها

**Return:** `true` لو مدعوم، `false` غير ده.

**Key details:** `spi_mem_default_supports_op()` بتفحص إن الـ op مش DTR ومش ECC ومش swap16 لأن دي features متخصصة مش كل controller بيدعمها.

**Pseudocode:**
```
if controller->mem_ops && controller->mem_ops->supports_op:
    return controller->mem_ops->supports_op(mem, op)
return spi_mem_default_supports_op(mem, op)
```

---

#### `spi_mem_exec_op()`

```c
int spi_mem_exec_op(struct spi_mem *mem,
                    const struct spi_mem_op *op);
```

الـ entry point الرئيسي لتنفيذ SPI memory operation. بتمر بالـ flow التالي:
1. لو الـ controller عنده `mem_ops->exec_op` بتستدعيه مباشرةً.
2. لو مفيش، بتترجم الـ op لـ `spi_message` و`spi_transfer` عادية وبتبعتها عبر `spi_sync()`.

**Parameters:**
- `mem` — الـ SPI memory device
- `op` — الـ operation المراد تنفيذها — `data.buf` لازم يكون DMA-safe

**Return:** `0` نجاح، error code سالب عند الفشل (مثلاً `-EOPNOTSUPP` لو الـ op مش مدعومة).

**Key details:**
- الـ buf في `op->data` **لازم يكون DMA-able** — مش stack memory.
- بيستدعي `spi_mem_adjust_op_freq()` قبل وبعد الـ exec لو الـ controller يدعم per-op frequency.
- الـ locking موجود على مستوى `spi_sync()` في الـ SPI core.
- الـ caller هو الـ NOR/NAND/EEPROM driver في task context.

**Pseudocode:**
```
adjust_op_freq(mem, op)   // set freq if per_op_freq supported
if controller has exec_op:
    ret = controller->mem_ops->exec_op(mem, op)
else:
    build spi_message from op phases
    ret = spi_sync(mem->spi, &msg)
restore_op_freq(mem, op)  // restore original freq
return ret
```

---

#### `spi_mem_default_supports_op()`

```c
bool spi_mem_default_supports_op(struct spi_mem *mem,
                                 const struct spi_mem_op *op);
```

الـ fallback checker لما الـ controller ما يحددش `supports_op` خاص بيه. بيرفض أي operation فيها DTR أو ECC أو swap16 لأن الـ legacy SPI core ما بيدعمهاش.

**Parameters:**
- `mem` — غير مستخدم فعلياً في التحقق، بس موجود للـ signature compatibility
- `op` — الـ operation المراد فحصها

**Return:** `true` لو الـ op standard (no DTR/ECC/swap16)، `false` غير ذلك.

**Key details:** لو `CONFIG_SPI_MEM` مش معمول `enable`، النسخة الـ stub بترجع `false` دايماً.

---

### Group 4: Name Retrieval

---

#### `spi_mem_get_name()`

```c
const char *spi_mem_get_name(struct spi_mem *mem);
```

بترجع اسم الـ SPI memory device. لو الـ controller عنده `mem_ops->get_name` بتستدعيه — مفيد للـ controllers اللي اتحولت من طبقات قديمة (زي m25p80) وتحتاج تحافظ على نفس اسم الـ mtd partition.

**Parameters:**
- `mem` — الـ SPI memory device

**Return:** `const char *` — الاسم. مش لازم تعمل `kfree` عليه لأن الـ allocation بتتعمل بـ `devm_*`.

**Key details:** لو `get_name` مش موجودة بيرجع `dev_name(&mem->spi->dev)` كـ default.

---

### Group 5: Direct Mapping (dirmap)

الـ **direct mapping** هو ميزة في بعض الـ QSPI controllers (زي Cadence QSPI، Aspeed FMC) بتسمح بعمل map لجزء من الـ flash في الـ CPU address space بشكل مباشر — يعني تقدر تقرأ الـ flash زي ما بتقرأ memory عادية. ده بيخلي الـ read أسرع بكتير لأنه بيتجنب الـ command overhead.

```
CPU address space
┌────────────────────┐
│   ... RAM ...      │
├────────────────────┤  ← dirmap window start (mapped to flash offset)
│  Flash region      │  Direct read/write
│  (e.g. 16MB)       │
├────────────────────┤
│   ...              │
└────────────────────┘
```

---

#### `spi_mem_dirmap_create()`

```c
struct spi_mem_dirmap_desc *
spi_mem_dirmap_create(struct spi_mem *mem,
                      const struct spi_mem_dirmap_info *info);
```

بتنشئ direct mapping descriptor. بتستدعي `controller->mem_ops->dirmap_create()` لو موجودة. لو مش موجودة أو رجعت error، بتسيب `desc->nodirmap = 1` وبترجع الـ descriptor على أي حال — ده fallback mode بيخلي الـ callers يستخدموا نفس الـ API بدون ما يعرفوا إن الـ hardware ما بيدعمش direct map.

**Parameters:**
- `mem` — الـ SPI memory device
- `info` — معلومات الـ mapping: `op_tmpl` (الـ operation template)، `offset` (الـ absolute flash offset)، `length` (حجم الـ window)

**Return:** pointer على `spi_mem_dirmap_desc` نجاح، أو `ERR_PTR(errno)` عند الفشل الكامل.

**Key details:**
- الـ descriptor بيتخصص بـ `kzalloc` في الـ SPI mem core.
- الـ `info->op_tmpl.data.dir` بيحدد إذا كان الـ mapping للـ read أو write.
- الـ `priv` field في الـ descriptor بيملأه الـ controller driver بـ resources خاصة بيه.

**Pseudocode:**
```
desc = kzalloc(sizeof(*desc))
desc->mem = mem
desc->info = *info

if controller has dirmap_create:
    ret = controller->mem_ops->dirmap_create(desc)
    if ret:
        desc->nodirmap = 1  // fallback, not fatal
else:
    desc->nodirmap = 1

return desc
```

---

#### `spi_mem_dirmap_destroy()`

```c
void spi_mem_dirmap_destroy(struct spi_mem_dirmap_desc *desc);
```

بتحرر الـ direct mapping. لو `nodirmap == 0` بتستدعي `controller->mem_ops->dirmap_destroy()` عشان الـ controller يحرر resources الخاصة بيه، بعدين `kfree` على الـ descriptor.

**Parameters:**
- `desc` — الـ descriptor اللي بيتحرر

**Return:** لا يرجع قيمة.

**Key details:** لو الـ descriptor اتعمل بـ `devm_spi_mem_dirmap_create()` لازم تستخدم `devm_spi_mem_dirmap_destroy()` أو تسيب الـ devres framework يعمل cleanup تلقائياً.

---

#### `spi_mem_dirmap_read()`

```c
ssize_t spi_mem_dirmap_read(struct spi_mem_dirmap_desc *desc,
                            u64 offs, size_t len, void *buf);
```

بتقرأ `len` bytes من الـ flash بـ offset `offs` عبر الـ direct mapping. لو `nodirmap == 1` بتفول على `spi_mem_exec_op()` بدلاً من الـ direct mapping.

**Parameters:**
- `desc` — الـ direct mapping descriptor
- `offs` — الـ offset **داخل الـ mapping window** (مش absolute)
- `len` — عدد الـ bytes المطلوب قراءتها
- `buf` — الـ output buffer، لازم DMA-safe

**Return:** عدد الـ bytes اللي اتقرأت فعلاً (ممكن أقل من `len`)، أو error code سالب.

**Key details:**
- ممكن ترجع أقل من `len` لو الـ request بيتعدى حدود الـ mapping window — الـ caller مسؤول عن الـ loop.
- الـ CPU direct access من خلال الـ mapped window ممنوع (مذكور في الـ kernel doc) — استخدم DMA.
- `offs` بيتضاف على `desc->info.offset` جوه الـ controller driver لحساب الـ absolute address.

**Pseudocode:**
```
if desc->nodirmap:
    fill op from desc->info.op_tmpl with offs and len
    return spi_mem_exec_op(desc->mem, &op)

return controller->mem_ops->dirmap_read(desc, offs, len, buf)
```

---

#### `spi_mem_dirmap_write()`

```c
ssize_t spi_mem_dirmap_write(struct spi_mem_dirmap_desc *desc,
                             u64 offs, size_t len, const void *buf);
```

نفس منطق `spi_mem_dirmap_read()` لكن في اتجاه الكتابة.

**Parameters:**
- `desc` — الـ direct mapping descriptor (لازم يكون `data.dir == SPI_MEM_DATA_OUT`)
- `offs` — الـ offset داخل الـ mapping window
- `len` — عدد الـ bytes للكتابة
- `buf` — الـ input buffer، لازم DMA-safe

**Return:** عدد الـ bytes اللي اتكتبت فعلاً، أو error code سالب.

**Key details:** نفس ملاحظات `dirmap_read()` — partial writes ممكنة، الـ caller يعمل loop.

---

#### `devm_spi_mem_dirmap_create()`

```c
struct spi_mem_dirmap_desc *
devm_spi_mem_dirmap_create(struct device *dev, struct spi_mem *mem,
                           const struct spi_mem_dirmap_info *info);
```

نسخة `devres`-managed من `spi_mem_dirmap_create()`. الـ descriptor بيتحرر تلقائياً لما الـ `dev` يعمل unbind.

**Parameters:**
- `dev` — الـ device اللي بيمتلك الـ descriptor (عادةً `&mem->spi->dev`)
- `mem` — الـ SPI memory device
- `info` — معلومات الـ mapping

**Return:** نفس `spi_mem_dirmap_create()`.

**Key details:** بيعمل `devm_add_action()` تحت الغطا بـ callback على `spi_mem_dirmap_destroy()`.

---

#### `devm_spi_mem_dirmap_destroy()`

```c
void devm_spi_mem_dirmap_destroy(struct device *dev,
                                 struct spi_mem_dirmap_desc *desc);
```

بتعمل تدمير مبكر يدوي للـ devm-managed descriptor قبل الـ device unbind. مفيدة في error paths.

**Parameters:**
- `dev` — نفس الـ device اللي اتسجل بيه الـ resource
- `desc` — الـ descriptor المراد تدميره

**Return:** لا يرجع قيمة.

---

### Group 6: Status Polling

---

#### `spi_mem_poll_status()`

```c
int spi_mem_poll_status(struct spi_mem *mem,
                        const struct spi_mem_op *op,
                        u16 mask, u16 match,
                        unsigned long initial_delay_us,
                        unsigned long polling_delay_us,
                        u16 timeout_ms);
```

بتعمل polling على status register لـ SPI memory device لحد ما `(status & mask) == match` أو timeout. بتستدعي `controller->mem_ops->poll_status()` لو موجودة (hardware polling)، وإلا بتعمل software polling بـ `read_poll_timeout()`.

**Parameters:**
- `mem` — الـ SPI memory device
- `op` — الـ read-status operation (عادةً RDSR — Read Status Register)؛ `op->data.buf.in` بيستقبل الـ status value
- `mask` — الـ bits اللي بتنظر فيها في الـ status
- `match` — القيمة المطلوبة بعد الـ mask
- `initial_delay_us` — انتظار مبدئي قبل أول poll (بالـ microseconds)
- `polling_delay_us` — الفترة بين كل poll والتالية
- `timeout_ms` — أقصى وقت انتظار بالـ milliseconds

**Return:** `0` لما يتحقق الشرط، `-ETIMEDOUT` عند انتهاء الـ timeout، أو error code سالب آخر.

**Key details:**
- الـ hardware polling (عبر `mem_ops->poll_status`) بيخلي الـ controller نفسه يعمل polling بدون تدخل الـ CPU في كل cycle — أفضل بكتير للـ latency.
- في الـ software fallback، `initial_delay_us` بيتسمى أول ما يبدأ، بعدين loop بـ `polling_delay_us`.
- الـ `op->data.buf.in` بيتملى بآخر status value — مفيد للـ debugging.
- بيتسمى في context بتاع الـ flash erase/program operations لما تنتظر اكتمالها.

**Pseudocode:**
```
udelay(initial_delay_us)

if controller has poll_status:
    return controller->mem_ops->poll_status(mem, op, mask, match,
                                            initial_delay_us,
                                            polling_delay_us, timeout_ms)

// software fallback
deadline = jiffies + msecs_to_jiffies(timeout_ms)
loop:
    ret = spi_mem_exec_op(mem, op)
    if ret: return ret
    if (*status & mask) == match: return 0
    if time_after(jiffies, deadline): return -ETIMEDOUT
    udelay(polling_delay_us)
```

---

### Group 7: DMA Mapping Helpers

الـ functions دي بتساعد الـ controller drivers في عمل DMA map/unmap لبيانات الـ `spi_mem_op`. موجودة فقط لما `CONFIG_SPI_MEM` يكون enabled.

---

#### `spi_controller_dma_map_mem_op_data()`

```c
int spi_controller_dma_map_mem_op_data(struct spi_controller *ctlr,
                                       const struct spi_mem_op *op,
                                       struct sg_table *sg);
```

بتعمل DMA map لبيانات الـ `op->data` في `sg_table`. بتستخدم `dma_map_single()` أو `dma_map_sg()` حسب الـ buffer.

**Parameters:**
- `ctlr` — الـ SPI controller
- `op` — الـ operation — بتاخد `op->data.buf` و`op->data.nbytes` و`op->data.dir`
- `sg` — الـ scatter-gather table اللي بتتملى

**Return:** `0` نجاح، أو error code سالب.

**Key details:**
- بتحدد الـ DMA direction من `op->data.dir` (IN → `DMA_FROM_DEVICE`، OUT → `DMA_TO_DEVICE`).
- لازم تتسمى من الـ controller driver قبل ما يبدأ الـ DMA transfer.
- stub بترجع `-ENOTSUPP` لو `CONFIG_SPI_MEM` مش enabled.

---

#### `spi_controller_dma_unmap_mem_op_data()`

```c
void spi_controller_dma_unmap_mem_op_data(struct spi_controller *ctlr,
                                          const struct spi_mem_op *op,
                                          struct sg_table *sg);
```

بتعمل unmap للـ DMA mapping اللي عملته `spi_controller_dma_map_mem_op_data()`. لازم تتسمى بعد اكتمال الـ DMA transfer.

**Parameters:**
- `ctlr` — الـ SPI controller
- `op` — نفس الـ operation اللي اتعمل map بيها
- `sg` — نفس الـ `sg_table` اللي اتعمل فيها الـ map

**Return:** لا يرجع قيمة.

**Key details:** stub فاضية لو `CONFIG_SPI_MEM` مش enabled.

---

### Group 8: Driver Registration

---

#### `spi_mem_driver_register_with_owner()`

```c
int spi_mem_driver_register_with_owner(struct spi_mem_driver *drv,
                                       struct module *owner);
```

بتسجل `spi_mem_driver` في الـ SPI bus. داخلياً بتعمل wrap حوله كـ `spi_driver` عادي وبتمرر الـ probe/remove/shutdown callbacks عبر shims موجودة في الـ SPI mem core.

**Parameters:**
- `drv` — الـ `spi_mem_driver` struct اللي فيه `.spidrv` والـ callbacks
- `owner` — الـ module owner، دايماً `THIS_MODULE`

**Return:** `0` نجاح، أو error code سالب.

**Key details:**
- الـ core بيعمل alloc لـ `spi_mem` struct ويربطه بالـ `spi_device` في `probe` shim.
- الـ `drv->spidrv.driver.bus` بيتبقى `&spi_bus_type`.
- ما بتتسمى مباشرةً — استخدم `spi_mem_driver_register()` macro.

---

#### `spi_mem_driver_unregister()`

```c
void spi_mem_driver_unregister(struct spi_mem_driver *drv);
```

بتلغي تسجيل الـ driver وبتستدعي `remove` على كل الـ bound devices.

**Parameters:**
- `drv` — الـ driver المراد إلغاء تسجيله

**Return:** لا يرجع قيمة.

---

#### `spi_mem_driver_register()` (Macro)

```c
#define spi_mem_driver_register(__drv) \
    spi_mem_driver_register_with_owner(__drv, THIS_MODULE)
```

Wrapper macro بيمرر `THIS_MODULE` تلقائياً — ده اللي بيتسمى في الغالب.

---

#### `module_spi_mem_driver()` (Macro)

```c
#define module_spi_mem_driver(__drv) \
    module_driver(__drv, spi_mem_driver_register, \
                  spi_mem_driver_unregister)
```

بيولد `module_init()` و`module_exit()` تلقائياً. كل اللي بيعمله الـ driver هو تعريف `struct spi_mem_driver` واستدعاء الـ macro ده في آخر الملف.

**مثال استخدام:**
```c
static struct spi_mem_driver my_flash_driver = {
    .spidrv = {
        .driver = { .name = "my-flash" },
        .id_table = my_flash_ids,
    },
    .probe  = my_flash_probe,
    .remove = my_flash_remove,
};
module_spi_mem_driver(my_flash_driver);
```

---

### Group 9: Helper Macros لبناء الـ Operations

الـ macros دي بتسهل بناء `struct spi_mem_op` بشكل declarative بدون ما تكتب كل الـ fields يدوياً.

#### بناء الـ Op بشكل كامل

```c
/* مثال: READ operation على QSPI NOR flash */
struct spi_mem_op op = SPI_MEM_OP(
    SPI_MEM_OP_CMD(0x03, 1),          /* CMD: 0x03 READ, single wire */
    SPI_MEM_OP_ADDR(3, addr, 1),      /* ADDR: 3 bytes, single wire */
    SPI_MEM_OP_NO_DUMMY,              /* no dummy cycles */
    SPI_MEM_OP_DATA_IN(len, buf, 4),  /* DATA: quad wire, reading */
);
```

| Macro | الغرض |
|---|---|
| `SPI_MEM_OP_CMD(op, bw)` | بناء CMD phase بـ opcode واحد وـ buswidth |
| `SPI_MEM_DTR_OP_RPT_CMD(op, bw)` | CMD phase في DTR mode مع تكرار الـ opcode |
| `SPI_MEM_OP_ADDR(n, val, bw)` | ADDR phase بعدد bytes محدد وقيمة وـ buswidth |
| `SPI_MEM_DTR_OP_ADDR(n, val, bw)` | ADDR phase بـ DTR mode |
| `SPI_MEM_DTR_OP_RPT_ADDR(val, bw)` | ADDR phase في DTR مع تكرار القيمة (2 bytes) |
| `SPI_MEM_OP_NO_ADDR` | لا يوجد address phase |
| `SPI_MEM_OP_DUMMY(n, bw)` | dummy phase بعدد bytes وـ buswidth |
| `SPI_MEM_DTR_OP_DUMMY(n, bw)` | dummy phase في DTR mode |
| `SPI_MEM_OP_NO_DUMMY` | لا يوجد dummy phase |
| `SPI_MEM_OP_DATA_IN(n, buf, bw)` | DATA phase للقراءة |
| `SPI_MEM_DTR_OP_DATA_IN(n, buf, bw)` | DATA phase للقراءة في DTR mode |
| `SPI_MEM_OP_DATA_OUT(n, buf, bw)` | DATA phase للكتابة |
| `SPI_MEM_DTR_OP_DATA_OUT(n, buf, bw)` | DATA phase للكتابة في DTR mode |
| `SPI_MEM_OP_NO_DATA` | لا يوجد data phase |
| `SPI_MEM_OP_MAX_FREQ(f)` | تحديد أقصى تردد للـ op |
| `SPI_MEM_OP(cmd, addr, dummy, data, ...)` | بناء الـ op الكاملة |

#### الـ DTR Repeat Macros

في DTR (Double Transfer Rate) mode، بعض الـ controllers بتتطلب إرسال الـ opcode أو الـ address مرتين (مرة في rising clock و مرة في falling clock). الـ macros اللي بدأت بـ `SPI_MEM_DTR_OP_RPT_` بتعمل ده تلقائياً بـ `op | op << 8` أو `val | val << 8`.

#### الـ `spi_mem_controller_is_capable()` Macro

```c
#define spi_mem_controller_is_capable(ctlr, cap) \
    ((ctlr)->mem_caps && (ctlr)->mem_caps->cap)
```

بيتحقق إن الـ controller يدعم capability معينة من الـ `spi_controller_mem_caps` struct. الـ caps المتاحة:

| Capability | المعنى |
|---|---|
| `dtr` | يدعم Double Transfer Rate |
| `ecc` | يدعم Error Correction Code |
| `swap16` | يدعم swap16 في Octal DTR |
| `per_op_freq` | يدعم تغيير التردد لكل op |
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs — المدخلات المفيدة

**الـ SPI mem subsystem** بيكتب معلومات في debugfs تحت `/sys/kernel/debug/`:

| المسار | المحتوى |
|---|---|
| `/sys/kernel/debug/spi-N/` | معلومات الـ controller رقم N |
| `/sys/kernel/debug/spi-N/N.M/` | الـ device M على الـ bus N |
| `/sys/kernel/debug/regmap/spi-N.M/` | registers لو الـ driver بيستخدم regmap |

```bash
# اقرأ كل اللي في debugfs الخاص بـ SPI
ls /sys/kernel/debug/spi*/

# لو بتستخدم SPINAND/SPINOR عبر MTD
ls /sys/kernel/debug/mtd/
cat /sys/kernel/debug/mtd/mtd0/partitions
```

---

#### 2. sysfs — المدخلات المفيدة

```bash
# كل devices على الـ SPI bus
ls /sys/bus/spi/devices/

# الـ controller وقدراته
cat /sys/bus/spi/devices/spi0.0/modalias
cat /sys/bus/spi/devices/spi0.0/driver/module/parameters/*

# إحصائيات الـ SPI transfers (pcpu_statistics)
cat /sys/bus/spi/devices/spi0.0/statistics/messages
cat /sys/bus/spi/devices/spi0.0/statistics/transfers
cat /sys/bus/spi/devices/spi0.0/statistics/errors
cat /sys/bus/spi/devices/spi0.0/statistics/bytes
cat /sys/bus/spi/devices/spi0.0/statistics/bytes_tx
cat /sys/bus/spi/devices/spi0.0/statistics/bytes_rx
cat /sys/bus/spi/devices/spi0.0/statistics/timedout

# الـ MTD layer فوق SPI NOR/NAND
cat /proc/mtd
```

---

#### 3. ftrace — تتبع الـ SPI mem operations

**الـ tracepoints** المتاحة في الـ SPI mem subsystem:

```bash
# اعرض كل trace events المتعلقة بـ SPI
ls /sys/kernel/debug/tracing/events/spi/

# فعّل كل events الـ SPI
echo 1 > /sys/kernel/debug/tracing/events/spi/enable

# أو فعّل events محددة فقط
echo 1 > /sys/kernel/debug/tracing/events/spi/spi_controller_busy/enable
echo 1 > /sys/kernel/debug/tracing/events/spi/spi_controller_idle/enable
echo 1 > /sys/kernel/debug/tracing/events/spi/spi_transfer_start/enable
echo 1 > /sys/kernel/debug/tracing/events/spi/spi_transfer_stop/enable
echo 1 > /sys/kernel/debug/tracing/events/spi/spi_message_submit/enable
echo 1 > /sys/kernel/debug/tracing/events/spi/spi_message_start/enable
echo 1 > /sys/kernel/debug/tracing/events/spi/spi_message_done/enable

# ابدأ التتبع
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل العملية اللي بتحاول تـ debug-ها ثم اقرأ
cat /sys/kernel/debug/tracing/trace

# تتبع دالة spi_mem_exec_op تحديداً بـ function_graph tracer
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo spi_mem_exec_op > /sys/kernel/debug/tracing/set_graph_function
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... شغّل العملية ...
cat /sys/kernel/debug/tracing/trace
```

**مثال على output متوقع:**

```
           <...>-1234  [000]  1234.567: spi_message_submit: master=spi0 message=ffff888...
           <...>-1234  [000]  1234.568: spi_transfer_start: master=spi0 xfer=ffff888... len=4
           <...>-1234  [000]  1234.569: spi_transfer_stop:  master=spi0 xfer=ffff888... len=4
```

---

#### 4. printk / dynamic debug

```bash
# فعّل dynamic debug لكل كود الـ SPI mem
echo "module spi_mem +p" > /sys/kernel/debug/dynamic_debug/control

# أو لـ file معين
echo "file drivers/spi/spi-mem.c +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل للـ SPINOR
echo "module mtd +p" > /sys/kernel/debug/dynamic_debug/control

# شاهد الرسائل
dmesg -w | grep -E "spi|spi_mem"

# لو محتاج timestamps دقيقة
dmesg -T --follow
```

**في الكود** — لو بتكتب driver وتحتاج تـ debug الـ `spi_mem_op`:

```c
/* طباعة تفاصيل الـ operation قبل التنفيذ */
static void debug_spi_mem_op(const struct spi_mem_op *op)
{
    pr_debug("SPI MEM OP: cmd=0x%02x nbytes=%u buswidth=%u dtr=%u\n",
             op->cmd.opcode, op->cmd.nbytes,
             op->cmd.buswidth, op->cmd.dtr);
    pr_debug("  addr: nbytes=%u buswidth=%u val=0x%llx dtr=%u\n",
             op->addr.nbytes, op->addr.buswidth,
             op->addr.val, op->addr.dtr);
    pr_debug("  dummy: nbytes=%u buswidth=%u dtr=%u\n",
             op->dummy.nbytes, op->dummy.buswidth, op->dummy.dtr);
    pr_debug("  data: nbytes=%u buswidth=%u dir=%d dtr=%u ecc=%u\n",
             op->data.nbytes, op->data.buswidth,
             op->data.dir, op->data.dtr, op->data.ecc);
}
```

---

#### 5. Kernel Config options للـ debugging

| الـ Config | الغرض |
|---|---|
| `CONFIG_SPI_DEBUG` | يفعّل رسائل debug في الـ SPI core |
| `CONFIG_SPI_MEM` | لازم يكون مفعّل أصلاً لاستخدام spi-mem |
| `CONFIG_MTD_SPI_NOR` | SPI NOR flash driver |
| `CONFIG_MTD_SPINAND` | SPI NAND flash driver |
| `CONFIG_SPI_SPIDEV` | userspace access لاختبار الـ bus مباشرة |
| `CONFIG_DYNAMIC_DEBUG` | يتيح dynamic debug activation |
| `CONFIG_TRACING` | لازم لـ ftrace events |
| `CONFIG_KASAN` | يكتشف memory corruption في الـ DMA buffers |
| `CONFIG_DEBUG_SG` | يتحقق من صحة الـ scatter-gather lists |
| `CONFIG_DMA_API_DEBUG` | يتحقق من الـ DMA mappings اللي بتستخدمها spi_controller_dma_map_mem_op_data |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "CONFIG_SPI|CONFIG_MTD_SPI"
```

---

#### 6. أدوات خاصة بالـ subsystem

**spidev_test** — اختبار الـ bus مباشرة من userspace:

```bash
# كمبايل من kernel sources
gcc tools/spi/spidev_test.c -o spidev_test

# إرسال byte بسيط وقراءة الرد
./spidev_test -D /dev/spidev0.0 -s 1000000 -b 8 -p "\x9f\x00\x00\x00"

# قراءة JEDEC ID من SPI NOR flash
./spidev_test -D /dev/spidev0.0 -s 4000000 -v -p "\x9f\x00\x00\x00"
```

**flashrom** — أداة قوية لـ debugging الـ SPI flash:

```bash
# كشف الـ flash chip
flashrom -p linux_spi:dev=/dev/spidev0.0,spispeed=4000

# قراءة كامل الـ flash
flashrom -p linux_spi:dev=/dev/spidev0.0 -r dump.bin
```

**mtd-utils** — للـ MTD layer فوق SPI:

```bash
# اقرأ معلومات الـ flash
mtdinfo /dev/mtd0
mtdinfo -a  # كل الـ MTD devices

# اختبار read/write/erase
flash_erase /dev/mtd0 0 0
flashcp firmware.bin /dev/mtd0
dd if=/dev/mtd0 of=readback.bin bs=4096
```

---

#### 7. رسائل الـ Error الشائعة

| رسالة الـ kernel | المعنى | الحل |
|---|---|---|
| `spi_mem_exec_op: operation not supported` | الـ controller مش بيدعم الـ op mode (مثلاً QSPI mode) | تحقق من `supports_op()` وـ `spi_mem_default_supports_op()` |
| `spi_mem_exec_op returned -ETIMEDOUT` | الـ device ما ردش في الوقت المحدد | تحقق من CLK والـ CS، راجع `poll_status` timeouts |
| `spi-mem: op not supported by the controller` | مفيش دعم لـ DTR أو multi-lane | تحقق من `spi_controller_mem_caps` |
| `Failed to map DMA buffer` | الـ buffer مش DMA-able | استخدم `kmalloc` بدلاً من stack allocation — data.buf.in لازم يكون DMA-safe |
| `spi_mem_dirmap_create: failed` | الـ controller مش بيدعم direct mapping | الـ `nodirmap` هيتعمل 1 تلقائياً وهيرجع لـ `spi_mem_exec_op` |
| `spi transfer timed out` | transfer لم يكتمل | تحقق من الـ clock، الـ CS، والـ MISO/MOSI |
| `spi_mem_adjust_op_size` returns unexpected size | الـ controller ليه size limitation | الـ driver لازم يـ loop على الـ operation |
| `DMA map failed for SPI transfer` | مشكلة في الـ DMA coherency | تحقق من الـ iommu و dma_mask |
| `spi: unknown parameter` in DT | property خاطئ في الـ Device Tree | راجع `Documentation/devicetree/bindings/spi/` |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في exec_op — تحقق إن الـ op سليم قبل التنفيذ */
int your_exec_op(struct spi_mem *mem, const struct spi_mem_op *op)
{
    /* تحقق من الـ buffer يكون DMA-able */
    if (op->data.nbytes && op->data.dir == SPI_MEM_DATA_IN) {
        WARN_ON(!virt_addr_valid(op->data.buf.in));
    }

    /* تحقق إن الـ buswidth صحيح */
    WARN_ON(op->cmd.buswidth != 1 && op->cmd.buswidth != 2 &&
            op->cmd.buswidth != 4 && op->cmd.buswidth != 8);

    /* لو DTR مطلوب والـ controller مش بيدعمه */
    if (op->cmd.dtr || op->addr.dtr || op->data.dtr) {
        WARN_ON(!spi_mem_controller_is_capable(mem->spi->controller, dtr));
    }
    ...
}

/* في poll_status — تحقق من الـ timeout */
int your_poll_status(struct spi_mem *mem, const struct spi_mem_op *op,
                     u16 mask, u16 match, ...)
{
    /* لو الـ timeout انتهى من غير match */
    if (time_after(jiffies, timeout)) {
        dump_stack();  /* اطبع الـ call stack لمعرفة من طلب */
        return -ETIMEDOUT;
    }
    ...
}
```

---

### Hardware Level

#### 1. التحقق من أن الـ hardware state يطابق الـ kernel state

```bash
# تحقق إن الـ device ظهر صح
dmesg | grep -E "spi|spi-mem|spinor|spinand|mtd"

# تحقق من الـ spi_device المسجّل
ls -la /sys/bus/spi/devices/
cat /sys/bus/spi/devices/spi0.0/modalias

# تحقق من الـ MTD partitions لو SPINOR/SPINAND
cat /proc/mtd

# تحقق من الـ driver المرتبط
ls -la /sys/bus/spi/drivers/
cat /sys/bus/spi/devices/spi0.0/driver/uevent

# تحقق من الـ max_speed
cat /sys/bus/spi/devices/spi0.0/../of_node/spi-max-frequency
```

---

#### 2. Register Dump

لمعظم الـ QSPI controllers، الـ registers بتكون memory-mapped:

```bash
# ابحث عن base address الـ QSPI controller من DT أو /proc/iomem
cat /proc/iomem | grep -i "spi\|qspi"

# مثال: QSPI controller على عنوان 0xFF0F0000
# قراءة الـ registers باستخدام devmem2
devmem2 0xFF0F0000 w  # Control Register
devmem2 0xFF0F0004 w  # Status Register
devmem2 0xFF0F0008 w  # Baud Rate Divider

# أو باستخدام /dev/mem مباشرة (خطر — للـ debugging فقط)
python3 -c "
import mmap, os, struct
with open('/dev/mem', 'r+b') as f:
    m = mmap.mmap(f.fileno(), 0x100, offset=0xFF0F0000)
    ctrl = struct.unpack('<I', m[0:4])[0]
    status = struct.unpack('<I', m[4:8])[0]
    print(f'CTRL=0x{ctrl:08x} STATUS=0x{status:08x}')
"

# io utility (من package ioport)
io -4 0xFF0F0000
```

---

#### 3. Logic Analyzer / Oscilloscope

**نقاط القياس على الـ SPI bus:**

```
SPI NOR Flash (x1 Standard Mode):
┌──────────────┐         ┌──────────────┐
│  SPI Master  │──CLK───▶│  SPI Memory  │
│  (QSPI Ctrl) │──CS────▶│  (NOR/NAND)  │
│              │──MOSI──▶│              │
│              │◀─MISO───│              │
└──────────────┘         └──────────────┘

QSPI Mode (x4):
  CLK, CS, IO0(MOSI), IO1(MISO), IO2(WP#), IO3(HOLD#)
```

**إعدادات الـ Logic Analyzer:**

| الـ Signal | Sample Rate المطلوب | ملاحظة |
|---|---|---|
| CLK | 10x clock freq | مثلاً 400MHz لـ 40MHz SPI |
| CS# | نفس الـ CLK | active low |
| MOSI/IO0 | نفس الـ CLK | MSB first |
| MISO/IO1 | نفس الـ CLK | MSB first |

**فيما تبحث:**

- **Setup/Hold violations** — بيظهروا كـ glitch على الـ data lines عند edge الـ CLK
- **CS timing** — الـ CS لازم يكون stable قبل أول CLK edge
- **Turnaround time** — في الـ reads، لازم dummy cycles بين الـ address والـ data
- **DTR mode** — بيأخذ sample على الـ rising AND falling edges، تأكد الـ analyzer بيدعم ده

```
مثال: READ command على SPI NOR (0x03)
CS#:  ▔▔▔╲_______________________________________________╱▔▔▔
CLK:  ___╱▔╲_╱▔╲_╱▔╲_╱▔╲_╱▔╲_╱▔╲_╱▔╲_╱▔╲_....__╱▔╲___
MOSI: ────[  0x03  ][    24-bit address    ][ dummy ]────
MISO: ─────────────────────────────────────[  data  ]───
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في الـ Kernel Log

| المشكلة | نمط الـ kernel log | التشخيص |
|---|---|---|
| CLK frequency عالية أوي | `spi transfer timed out` + data خاطئ | قلّل `spi-max-frequency` في الـ DT |
| CS لا يُفعَّل | لا يوجد response على الـ bus + `-ETIMEDOUT` | تحقق من GPIO الـ CS في الـ DT وـ `cs_gpiod` |
| مشكلة في الـ Pull-up/down | بيانات عشوائية متقطعة | قس voltage الـ IO lines بـ multimeter |
| QSPI mode غير مدعوم بالـ hardware | `spi-mem: op not supported` مع buswidth>1 | ارجع لـ x1 mode |
| خلل في الـ DMA alignment | `kernel BUG at lib/dma-debug.c` | الـ buffer لازم يكون aligned لـ DMA — استخدم `kmalloc` مش stack |
| Power supply noise | errors عشوائية بدون نمط | قس الـ VCC على الـ oscilloscope أثناء transfer |
| MISO floating | بيانات كلها 0xFF أو 0x00 | تحقق من الـ pull-up على MISO |
| غلطة في الـ DTR وiring | `dtr` errors في OSPI | تحقق من تطابق الـ differential pairs |

---

#### 5. Device Tree Debugging

```bash
# اطبع الـ DT الـ compiled الموجود فعلاً في الـ kernel
dtc -I fs -O dts /sys/firmware/devicetree/base/ 2>/dev/null | grep -A 20 "spi@"

# أو من الـ compiled dtb مباشرة
dtc -I dtb -O dts /boot/dtb/your-board.dtb | grep -A 30 "qspi\|spi@"

# تحقق من الـ properties المهمة لـ SPI memory
cat /sys/firmware/devicetree/base/soc/spi@ff0f0000/flash@0/compatible
cat /sys/firmware/devicetree/base/soc/spi@ff0f0000/flash@0/reg
cat /sys/firmware/devicetree/base/soc/spi@ff0f0000/flash@0/spi-max-frequency
cat /sys/firmware/devicetree/base/soc/spi@ff0f0000/flash@0/spi-tx-bus-width
cat /sys/firmware/devicetree/base/soc/spi@ff0f0000/flash@0/spi-rx-bus-width
```

**DT مثال صحيح لـ SPI NOR على QSPI controller:**

```dts
&qspi {
    status = "okay";
    #address-cells = <1>;
    #size-cells = <0>;

    flash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;                        /* CS0 */
        spi-max-frequency = <50000000>;   /* 50 MHz */
        spi-tx-bus-width = <4>;           /* QSPI write */
        spi-rx-bus-width = <4>;           /* QSPI read */
        /* لو بيدعم DTR */
        spi-dtr-timestamp-window = <8>;

        partitions {
            compatible = "fixed-partitions";
            #address-cells = <1>;
            #size-cells = <1>;

            bootloader@0 {
                reg = <0x0 0x80000>;      /* 512KB */
                read-only;
            };
        };
    };
};
```

**أخطاء شائعة في الـ DT:**

| الخطأ | الأثر |
|---|---|
| `spi-max-frequency` أعلى من حد الـ chip | data corruption أو timeout |
| `spi-rx-bus-width = <4>` بدون دعم controller | `op not supported` |
| `reg = <1>` وـ CS1 مش موصول | probe failure |
| missing `#address-cells` | kernel يرفض parse الـ DT |

---

### Practical Commands

#### سكريبت شامل لتشخيص SPI mem

```bash
#!/bin/bash
# spi_mem_debug.sh — سكريبت تشخيص SPI mem كامل

echo "=== SPI Devices ==="
ls /sys/bus/spi/devices/ 2>/dev/null

echo ""
echo "=== MTD Devices ==="
cat /proc/mtd 2>/dev/null || echo "No MTD devices"

echo ""
echo "=== SPI Statistics ==="
for dev in /sys/bus/spi/devices/*/statistics/; do
    devname=$(echo $dev | grep -oP 'spi\d+\.\d+')
    echo "--- $devname ---"
    for stat in messages transfers errors timedout bytes bytes_tx bytes_rx; do
        val=$(cat "${dev}${stat}" 2>/dev/null)
        echo "  $stat = $val"
    done
done

echo ""
echo "=== Recent SPI dmesg ==="
dmesg | grep -iE "spi|qspi|spi-mem|spinor|spinand|mtd" | tail -30

echo ""
echo "=== DT SPI nodes ==="
find /sys/firmware/devicetree/base -name "compatible" | \
  xargs grep -l "spi\|jedec" 2>/dev/null | \
  while read f; do
    echo "Node: $(dirname $f | sed 's|/sys/firmware/devicetree/base||')"
    cat "$(dirname $f)/compatible" 2>/dev/null | tr '\0' '\n'
  done
```

#### تفعيل الـ tracing لـ SPI mem operation واحدة

```bash
#!/bin/bash
# trace_spi_mem_op.sh — تتبع سريع لـ spi_mem_exec_op

# صفّر الـ trace
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo > /sys/kernel/debug/tracing/trace

# فعّل كل SPI events
echo 1 > /sys/kernel/debug/tracing/events/spi/enable

# ابدأ
echo 1 > /sys/kernel/debug/tracing/tracing_on

# نفّذ الأمر الذي تريد تتبعه
"$@"

# أوقف
echo 0 > /sys/kernel/debug/tracing/tracing_on

# اطبع النتائج
echo ""
echo "=== SPI Trace Output ==="
cat /sys/kernel/debug/tracing/trace

# نظّف
echo 0 > /sys/kernel/debug/tracing/events/spi/enable
```

```bash
# استخدام:
bash trace_spi_mem_op.sh dd if=/dev/mtd0 of=/dev/null bs=4096 count=1
```

#### تفعيل dynamic debug لـ spi-mem بسرعة

```bash
# فعّل كل رسائل الـ debug للـ SPI mem subsystem
echo "module spi_mem +pmfl" > /sys/kernel/debug/dynamic_debug/control
echo "module spi-nor +pmfl" > /sys/kernel/debug/dynamic_debug/control
echo "module spinand +pmfl" > /sys/kernel/debug/dynamic_debug/control

# شاهد الرسائل في الوقت الفعلي
dmesg -wH &

# نفّذ العملية
dd if=/dev/mtd0 of=/dev/null bs=4096 count=4

# أوقف الـ debug logging
echo "module spi_mem -pmfl" > /sys/kernel/debug/dynamic_debug/control
```

**مثال على output مع dynamic debug:**

```
[  42.123456] spi-mem spi0.0: spi_mem_exec_op cmd=0x03 addr=0x000000 nbytes=4096
[  42.123789] spi-mem spi0.0: exec_op: cmd.opcode=0x03 addr.nbytes=3 addr.val=0x0
[  42.124100] spi-mem spi0.0: exec_op: data.dir=in data.nbytes=4096 buswidth=1
[  42.126543] spi-mem spi0.0: spi_mem_exec_op completed ret=0
```

#### استخدام devmem2 لقراءة QSPI registers

```bash
# مثال: Cadence QSPI controller (Zynq/Cyclone)
BASE=0xFF0F0000

devmem2 $((BASE + 0x00)) w  # Config Register — CLK polarity, mode
devmem2 $((BASE + 0x04)) w  # Read Data Capture — delay settings
devmem2 $((BASE + 0x08)) w  # Device Size — address bytes, page size
devmem2 $((BASE + 0x14)) w  # SRAM Partition — TX/RX FIFO split
devmem2 $((BASE + 0x40)) w  # Flash Command Control — busy bit
devmem2 $((BASE + 0x44)) w  # Flash Command Address
devmem2 $((BASE + 0x5C)) w  # Module ID

# الـ busy bit في Flash Command Control Register
ctrl=$(devmem2 $((BASE + 0x40)) w | grep -oP '0x[0-9A-Fa-f]+$')
echo "CMD_CTRL = $ctrl"
echo "Busy = $(( (16#${ctrl#0x} >> 2) & 1 ))"
```

**تفسير output devmem2:**

```
/dev/mem opened.
Memory mapped at address 0x7f9a340000.
Read at address  0xFF0F0040 (0x7f9a340040): 0x00000000
                                             ^^^^^^^^^
                                          Bit 2 = 0 → Not busy
                                          Bits 8-11 = opcode length
                                          Bits 12-15 = data bytes count
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: QSPI NOR Flash مش بيشتغل على RK3562 — Industrial Gateway

#### العنوان
**Quad-SPI read فاشل على Rockchip RK3562 لأن `supports_op` بترفض Quad mode**

#### السياق
شركة بتبني industrial gateway بيستخدم RK3562 مع Winbond W25Q128JV NOR flash موصول على QSPI controller. المنتج بيرفع rootfs من الـ flash. أثناء bring-up الـ board لاحظ المهندس إن الـ kernel بيبوت بطيء جداً، والـ dmesg بيظهر إن الـ mtd layer بيقرأ بـ SPI mode عادي (1-1-1) مش Quad (1-1-4).

#### المشكلة
الـ QSPI controller driver على RK3562 بيعمل implement للـ `spi_controller_mem_ops` لكن الـ `supports_op` callback بتحتاج `data.buswidth == 4` وفي نفس الوقت `cmd.buswidth == 1`. الـ `spi-nor` driver بيبعت operation بـ `cmd.buswidth = 4` وده بيخلي الـ `supports_op` ترجع `false`.

#### التحليل

الـ flow بيمشي كده:

```
spi_nor_read()
  └─> spi_mem_exec_op(mem, &op)
        └─> spi_mem_supports_op(mem, op)        /* spi-mem.h API */
              └─> ctlr->mem_ops->supports_op()  /* controller callback */
                    └─> rockchip_spi_mem_supports_op()
                          └─> checks op->cmd.buswidth == 1  ← هنا بترفض
```

الـ `spi_mem_op` اللي بيبعتها `spi-nor` للقراءة:

```c
struct spi_mem_op op = SPI_MEM_OP(
    SPI_MEM_OP_CMD(0x6B, 4),         /* Fast Read Quad Output — cmd على 4 lanes */
    SPI_MEM_OP_ADDR(3, addr, 1),
    SPI_MEM_OP_DUMMY(8, 1),
    SPI_MEM_OP_DATA_IN(len, buf, 4)
);
```

**الـ `SPI_MEM_OP_CMD` macro:**
```c
#define SPI_MEM_OP_CMD(__opcode, __buswidth) \
    {                                         \
        .nbytes   = 1,                        \
        .buswidth = __buswidth,   /* ← 4 هنا */ \
        .opcode   = __opcode,                 \
    }
```

الـ controller driver بيتحقق:
```c
static bool rockchip_spi_mem_supports_op(struct spi_mem *mem,
                                          const struct spi_mem_op *op)
{
    /* driver قديم بيفترض cmd دايماً على lane واحد */
    if (op->cmd.buswidth != 1)
        return false;
    ...
}
```

#### الحل

**Option 1:** تغيير الـ opcode اللي بيبعته spi-nor لـ `0xEB` (Fast Read Quad I/O) اللي cmd.buswidth = 1:

```c
/* في spi-nor flash driver */
struct spi_mem_op op = SPI_MEM_OP(
    SPI_MEM_OP_CMD(0xEB, 1),        /* cmd على lane واحد */
    SPI_MEM_OP_ADDR(3, addr, 4),    /* addr على 4 lanes */
    SPI_MEM_OP_DUMMY(6, 4),
    SPI_MEM_OP_DATA_IN(len, buf, 4)
);
```

**Option 2:** تصحيح الـ RK3562 controller driver ليقبل cmd.buswidth = 4:

```c
static bool rockchip_spi_mem_supports_op(struct spi_mem *mem,
                                          const struct spi_mem_op *op)
{
    /* accept both 1-wire and 4-wire command phase */
    if (op->cmd.buswidth != 1 && op->cmd.buswidth != 4)
        return false;
    if (op->addr.buswidth > 4 || op->data.buswidth > 4)
        return false;
    return true;
}
```

**Debug commands:**
```bash
# شوف الـ mode اللي بيشتغل بيه الـ flash
cat /sys/bus/spi/devices/spi0.0/of_node/compatible
dmesg | grep -i "spi-nor\|m25p\|w25q"

# تأكد إن الـ dirmap اتعمل
dmesg | grep "dirmap"

# قيس الـ throughput
dd if=/dev/mtd0 of=/dev/null bs=1M count=10 2>&1 | grep copied
```

#### الدرس المستفاد
الـ `SPI_MEM_OP_CMD` macro بيفرق بين opcode اللي بيتبعت على lane واحد وopcode اللي بيتبعت على 4 lanes — ده مش نفس الـ data buswidth. لازم تتأكد إن الـ `supports_op` callback في الـ controller driver بتفهم كل combinations اللي ممكن الـ flash يطلبها.

---

### السيناريو 2: OctalSPI DTR بيديّ data corruption على STM32MP1 — IoT Sensor Hub

#### العنوان
**`data.swap16` مش متفعّل وبيسبب byte-order corruption في Octal DTR mode على STM32MP157**

#### السياق
مهندس بيعمل design لـ IoT sensor hub بـ STM32MP157 متصل بـ Macronix MX25UM51345G OctalSPI flash. الـ flash بيدعم Octal DTR (8-8-8 DTR). بعد تفعيل OctalSPI DTR mode في الـ DT، الـ data اللي بتتقرأ من الـ flash بتيجي corrupt — كل 16-bit word بيتعكس byte order بتاعه.

#### المشكلة
الـ Macronix flash ده بيقلب byte order الـ 16-bit words في DTR mode مقارنةً بـ STR mode. الـ kernel بيوفر `data.swap16` flag في `struct spi_mem_op` عشان يتعامل مع ده — لكن الـ STM32 controller driver مش بيفحص الـ `mem_caps->swap16` ومش بيعمل implement للـ swap في hardware.

#### التحليل

الـ `struct spi_mem_op` عندها في data section:
```c
struct {
    u8 buswidth;
    u8 dtr   : 1;
    u8 ecc   : 1;
    u8 swap16 : 1;  /* ← الـ flag المسؤول عن المشكلة */
    u8 __pad  : 5;
    enum spi_mem_data_dir dir;
    unsigned int nbytes;
    union {
        void *in;
        const void *out;
    } buf;
} data;
```

الـ `struct spi_controller_mem_caps`:
```c
struct spi_controller_mem_caps {
    bool dtr;
    bool ecc;
    bool swap16;      /* ← controller لازم يعلن دعمه لـ swap16 */
    bool per_op_freq;
};
```

الـ macro بيتحقق من الـ capabilities:
```c
#define spi_mem_controller_is_capable(ctlr, cap) \
    ((ctlr)->mem_caps && (ctlr)->mem_caps->cap)
```

الـ flow:
```
spi_mem_default_supports_op()
  └─> checks: if op->data.swap16 &&
              !spi_mem_controller_is_capable(ctlr, swap16)
                └─> return false   ← بيرفض الـ op بالكامل
```

لما الـ STM32 driver مش بيعلن `swap16 = true` في `mem_caps`، الـ `spi_mem_default_supports_op` بترفض أي operation بيها `swap16 = true`، فالـ spi-nor يرجع لـ STR mode بـ 1-1-1 أو 8-8-8 STR.

#### الحل

**Step 1:** تعديل الـ STM32 controller driver يعلن دعمه لـ swap16 ويعمل swap في software لو hardware مش بيدعمه:

```c
static const struct spi_controller_mem_caps stm32_ospi_mem_caps = {
    .dtr    = true,
    .ecc    = false,
    .swap16 = true,   /* ← enable swap16 support */
};

static ssize_t stm32_ospi_dirmap_read(struct spi_mem_dirmap_desc *desc,
                                       u64 offs, size_t len, void *buf)
{
    ssize_t ret = __stm32_ospi_read(desc, offs, len, buf);

    /* apply swap16 if needed and hardware doesn't do it automatically */
    if (ret > 0 && desc->info.op_tmpl.data.swap16)
        swab16_array(buf, ret / 2);

    return ret;
}
```

**Step 2:** تأكد الـ Device Tree صح:

```dts
&qspi {
    status = "okay";
    flash@0 {
        compatible = "macronix,mx25um51345g";
        reg = <0>;
        spi-max-frequency = <200000000>;
        spi-rx-bus-width = <8>;
        spi-tx-bus-width = <8>;
        m25p,fast-read;
    };
};
```

**Debug:**
```bash
# تحقق من الـ capabilities
dmesg | grep -i "octal\|dtr\|swap"

# اقرأ raw bytes وقارن
hexdump -C /dev/mtd0 | head -4

# شوف الـ spi_mem_op parameters أثناء read
echo 'p spi_mem_exec_op' | sudo tee /sys/kernel/debug/dynamic_debug/control
dmesg | grep spi_mem
```

#### الدرس المستفاد
الـ `swap16` في `struct spi_mem_op` و`struct spi_controller_mem_caps` موجودين عشان فرق حقيقي في behavior بين STR وDTR في بعض الـ OctalSPI flash chips. لازم الـ controller driver يعلن capabilities بصدق، وإلا الـ spi-mem layer هترفض الـ operation وترجع لـ mode أبطأ.

---

### السيناريو 3: Direct Mapping أبطأ من `exec_op` على i.MX8MP — Android TV Box

#### العنوان
**`nodirmap = 1` بيسبب regression في الـ QSPI throughput بعد kernel upgrade على i.MX8MP**

#### السياق
مهندس بيعمل Android TV box بـ NXP i.MX8M Plus مع QSPI NOR flash للـ bootloader و configuration. بعد upgrade من kernel 5.15 لـ 6.6، لاحظ إن boot time زاد 3 ثواني. الـ profiling وضّح إن قراءة الـ flash رجعت لـ `spi_mem_exec_op` بدل الـ direct mapping الـ hardware.

#### المشكلة
الـ i.MX8MP QSPI controller driver في kernel 6.6 بدّل تنفيذ `dirmap_create` — الـ function دلوقتي بتتحقق من alignment أكثر صرامة وبترجع error لما الـ `info->offset` مش aligned على 256 bytes. ده بيخلي `spi_mem_dirmap_create` يسيب `nodirmap = 1`.

#### التحليل

```c
struct spi_mem_dirmap_desc {
    struct spi_mem *mem;
    struct spi_mem_dirmap_info info;
    unsigned int nodirmap;  /* ← 1 = fallback to exec_op */
    void *priv;
};
```

`spi_mem_dirmap_create` في kernel:
```c
struct spi_mem_dirmap_desc *
spi_mem_dirmap_create(struct spi_mem *mem,
                      const struct spi_mem_dirmap_info *info)
{
    ...
    ret = ctlr->mem_ops->dirmap_create(desc);
    if (ret) {
        desc->nodirmap = 1;  /* fallback بدل error */
        /* لا error return — الـ desc لسه valid */
    }
    ...
}
```

لما `nodirmap = 1`، الـ `spi_mem_dirmap_read` بتعمل:
```c
ssize_t spi_mem_dirmap_read(struct spi_mem_dirmap_desc *desc,
                             u64 offs, size_t len, void *buf)
{
    if (desc->nodirmap) {
        /* slow path: one exec_op per chunk */
        return spi_mem_no_dirmap_read(desc, offs, len, buf);
    }
    return ctlr->mem_ops->dirmap_read(desc, offs, len, buf);
}
```

كل `exec_op` بيعمل SPI transaction كاملة مع overhead — بدل ما الـ hardware يقرأ directly من الـ memory-mapped window.

#### الحل

**Step 1:** تشخيص المشكلة:
```bash
# شوف لو nodirmap اتعمل set
dmesg | grep -i "dirmap\|nodirmap\|fallback"

# قيس الـ throughput
time dd if=/dev/mtd0 of=/dev/null bs=4k count=256

# شوف الـ offset اللي بيتطلبه الـ driver
cat /sys/kernel/debug/spi/spi0/dirmap_info 2>/dev/null
```

**Step 2:** تعديل الـ DT عشان الـ partition تبدأ على حدود 256 bytes:

```dts
/* قبل التعديل — offset = 0x1000 = 4096 (OK) لكن size مش مضروبة في 256 */
partition@1000 {
    label = "config";
    reg = <0x1000 0x7FF0>;  /* ← size = 0x7FF0 مش aligned */
};

/* بعد التعديل */
partition@1000 {
    label = "config";
    reg = <0x1000 0x8000>;  /* ← size = 0x8000 aligned على 256 */
};
```

**Step 3:** لو مش ممكن تعدّل الـ partition، عدّل الـ i.MX QSPI driver:

```c
static int imx_qspi_dirmap_create(struct spi_mem_dirmap_desc *desc)
{
    /* relax alignment: accept any 4-byte aligned offset */
    if (desc->info.offset & 0x3)
        return -EINVAL;
    /* remove the overly strict 256-byte alignment check */
    ...
    return 0;
}
```

#### الدرس المستفاد
الـ `nodirmap` field في `spi_mem_dirmap_desc` هو silent fallback — مش بيطبع warning واضح في كل الـ kernels. أي kernel upgrade ممكن يغير الـ dirmap alignment requirements وتتأثر الـ performance من غير noise في الـ dmesg. دايماً قيس الـ flash throughput كجزء من regression testing بعد kernel upgrade.

---

### السيناريو 4: `poll_status` بيعمل busy-wait وبيدمّر الـ real-time على AM62x — Automotive ECU

#### العنوان
**`spi_mem_poll_status` بيتجاهل `initial_delay_us` ويعمل tight polling يأثر على الـ RT tasks في TI AM62x**

#### السياق
مهندس بيطوّر automotive ECU بـ TI AM62x بيستخدم SPI NOR flash لتخزين calibration data. الـ ECU بيشغّل PREEMPT_RT kernel. أثناء flash erase/write operations، الـ RT tasks الخاصة بالـ CAN communication بتتأخر وبيظهر latency spikes في الـ oscilloscope.

#### المشكلة
الـ spi-nor driver بيستخدم `spi_mem_poll_status` لانتظار انتهاء الـ erase/write. الـ AM62x OSPI controller driver مش بيعمل implement للـ `poll_status` callback في `spi_controller_mem_ops`، فالـ kernel بيستخدم software polling loop. الـ polling بيتجاهل `initial_delay_us` ويعمل tight loop بيأخذ CPU time من الـ RT tasks.

#### التحليل

الـ `struct spi_controller_mem_ops`:
```c
struct spi_controller_mem_ops {
    ...
    int (*poll_status)(struct spi_mem *mem,
                       const struct spi_mem_op *op,
                       u16 mask, u16 match,
                       unsigned long initial_delay_us,
                       unsigned long polling_rate_us,
                       unsigned long timeout_ms);
};
```

لما الـ AM62x driver مش بيعمل implement لـ `poll_status`، الـ `spi_mem_poll_status` في kernel تعمل:
```c
int spi_mem_poll_status(struct spi_mem *mem,
                         const struct spi_mem_op *op,
                         u16 mask, u16 match,
                         unsigned long initial_delay_us,
                         unsigned long polling_delay_us,
                         u16 timeout_ms)
{
    struct spi_controller *ctlr = mem->spi->controller;

    if (ctlr->mem_ops && ctlr->mem_ops->poll_status) {
        return ctlr->mem_ops->poll_status(...);  /* hardware path */
    }

    /* software fallback */
    return spi_mem_sw_poll_status(mem, op, mask, match,
                                   initial_delay_us,
                                   polling_delay_us,
                                   timeout_ms);
}
```

الـ software fallback في بعض implementations بيعمل:
```c
/* problematic tight polling */
do {
    ret = spi_mem_exec_op(mem, op);  /* read status register */
    if ((val & mask) == match)
        break;
    udelay(polling_delay_us);  /* ← udelay بيحرق CPU مش بيسلّم */
} while (!timeout);
```

**الـ `udelay` في RT context بيعمل busy-wait** ومش بيسمح لـ scheduler يشغّل RT tasks.

#### الحل

**Option 1:** implement الـ `poll_status` في AM62x driver باستخدام hardware interrupt:

```c
static int am62x_ospi_poll_status(struct spi_mem *mem,
                                   const struct spi_mem_op *op,
                                   u16 mask, u16 match,
                                   unsigned long initial_delay_us,
                                   unsigned long polling_rate_us,
                                   unsigned long timeout_ms)
{
    struct am62x_ospi *ospi = spi_controller_get_devdata(mem->spi->controller);

    /* sleep first — don't busy-wait */
    fsleep(initial_delay_us);

    /* program hardware to poll and interrupt when done */
    am62x_ospi_setup_poll(ospi, mask, match);
    return wait_for_completion_timeout(&ospi->poll_done,
                                        msecs_to_jiffies(timeout_ms))
           ? 0 : -ETIMEDOUT;
}
```

**Option 2:** تغيير الـ software fallback يستخدم `usleep_range` بدل `udelay`:

```c
/* في spi-mem.c */
static int spi_mem_sw_poll_status(...)
{
    fsleep(initial_delay_us);   /* sleep مش busy-wait */

    while (!timeout) {
        ret = spi_mem_exec_op(mem, op);
        if ((val & mask) == match)
            return 0;
        usleep_range(polling_delay_us,
                     polling_delay_us + polling_delay_us / 10);
    }
    return -ETIMEDOUT;
}
```

**Debug:**
```bash
# قيس الـ latency spikes
cyclictest -p 99 -t1 -n -i 1000 -l 10000

# شوف الـ poll_status مكالمات
trace-cmd record -e spi:spi_transfer_start &
# run flash write
trace-cmd report | grep -i poll

# تأكد AM62x driver بيعمل implement poll_status
grep -r "poll_status" /sys/kernel/debug/spi/
```

#### الدرس المستفاد
غياب `poll_status` في `spi_controller_mem_ops` مش بس performance issue — في RT systems ده safety issue. الـ `initial_delay_us` و`polling_rate_us` parameters في `spi_mem_poll_status` موجودين عشان كده بالظبط. لازم الـ RT-certified products تتحقق إن كل الـ callbacks الـ time-critical متعمل لها implement في hardware driver.

---

### السيناريو 5: `adjust_op_size` مش شغالة صح على Allwinner H616 — Custom Board Bring-up

#### العنوان
**`spi_mem_adjust_op_size` بترجع أحجام أكبر من الـ RX FIFO على H616 وبتسبب data loss**

#### السياق
مهندس بيعمل bring-up لـ custom board بـ Allwinner H616 (orange pi zero 2). الـ board عنده SPI NOR flash XM25QH64C موصول على SPI0. الـ flash بيتعرف OK، لكن قراءة blocks أكبر من 64 bytes بتجيب garbage data. الـ write بيشتغل تمام.

#### المشكلة
الـ H616 SPI controller عنده RX FIFO size = 64 bytes. الـ `adjust_op_size` callback في الـ driver المفروض تحدّ `op->data.nbytes` بـ 64 لما direction هو `SPI_MEM_DATA_IN`. لكن في الـ H616 driver، الـ check بيعمل clamp على الـ TX size بس ومش بيعمل clamp على الـ RX size بشكل صحيح.

#### التحليل

الـ `spi_mem_adjust_op_size`:
```c
int spi_mem_adjust_op_size(struct spi_mem *mem, struct spi_mem_op *op)
{
    struct spi_controller *ctlr = mem->spi->controller;

    if (ctlr->mem_ops && ctlr->mem_ops->adjust_op_size)
        return ctlr->mem_ops->adjust_op_size(mem, op);  /* ← delegate to driver */

    return 0;  /* no adjustment needed */
}
```

الـ H616 driver المعطوب:
```c
static int h616_spi_adjust_op_size(struct spi_mem *mem,
                                    struct spi_mem_op *op)
{
    /* BUG: only checks TX direction! */
    if (op->data.dir == SPI_MEM_DATA_OUT &&
        op->data.nbytes > H616_SPI_FIFO_SIZE)
        op->data.nbytes = H616_SPI_FIFO_SIZE;

    return 0;
    /* SPI_MEM_DATA_IN not handled → RX overflow → garbage */
}
```

الـ enum الصح:
```c
enum spi_mem_data_dir {
    SPI_MEM_NO_DATA,
    SPI_MEM_DATA_IN,   /* ← read from flash — هنا المشكلة */
    SPI_MEM_DATA_OUT,  /* ← write to flash — ده شغّال */
};
```

الـ flow اللي بيحصل:
```
spi_nor_read(512 bytes)
  └─> spi_mem_exec_op(op with data.nbytes=512, dir=DATA_IN)
        └─> spi_mem_adjust_op_size()
              └─> h616_spi_adjust_op_size()
                    └─> dir == DATA_IN → لا adjustment
                    └─> nbytes لسه 512
        └─> __h616_spi_transfer(512 bytes RX)
              └─> FIFO overflow after 64 bytes
              └─> remaining 448 bytes = garbage
```

#### الحل

**تصحيح الـ H616 driver:**

```c
static int h616_spi_adjust_op_size(struct spi_mem *mem,
                                    struct spi_mem_op *op)
{
    if (op->data.dir == SPI_MEM_NO_DATA)
        return 0;

    /* clamp both TX and RX to FIFO size */
    if (op->data.nbytes > H616_SPI_FIFO_SIZE)
        op->data.nbytes = H616_SPI_FIFO_SIZE;

    return 0;
}
```

بعد التصحيح، الـ spi-nor layer هتعمل loop تلقائياً:
```c
/* spi-nor core يكرر الـ read بـ chunks */
while (remaining > 0) {
    op.data.nbytes = remaining;
    spi_mem_adjust_op_size(mem, &op);   /* ← هيحطه 64 */
    spi_mem_exec_op(mem, &op);
    buf += op.data.nbytes;
    remaining -= op.data.nbytes;
}
```

**Debug commands:**
```bash
# تأكد إن الـ flash بيتعرف صح
dmesg | grep -i "spi-nor\|xm25\|jedec"

# اقرأ 128 bytes وشوف الـ pattern
dd if=/dev/mtd0 bs=1 count=128 | xxd | head

# اكتب pattern معروف وارجع اقرأه
echo "TESTDATA_1234567890ABCDEF" > /tmp/test.bin
dd if=/tmp/test.bin of=/dev/mtd0 bs=32 seek=0
dd if=/dev/mtd0 bs=32 count=1 | xxd

# enable SPI tracing
echo 1 > /sys/kernel/debug/spi/spi0/statistics/reset
cat /sys/kernel/debug/spi/spi0/statistics/transfer_bytes_rx
```

**تحقق إن الـ adjust_op_size شغالة:**
```bash
# add kernel tracepoint
echo 'p:adj spi_mem_adjust_op_size mem=%di op=%si' > \
    /sys/kernel/debug/tracing/kprobe_events
echo 1 > /sys/kernel/debug/tracing/events/kprobes/adj/enable
dd if=/dev/mtd0 of=/dev/null bs=512 count=1
cat /sys/kernel/debug/tracing/trace
```

#### الدرس المستفاد
الـ `adjust_op_size` callback في `spi_controller_mem_ops` **لازم** تعالج `SPI_MEM_DATA_IN` و`SPI_MEM_DATA_OUT` على حد سواء. الـ `enum spi_mem_data_dir` فيه ثلاث قيم — أي driver بيعمل check على واحدة بس من غير `default` case هو driver ناقص. أثناء bring-up، دايماً اتحقق من الـ RX path باستخدام read-after-write test مع known patterns.
## Phase 7: مصادر ومراجع

---

### مقالات LWN.net

**الـ LWN.net** هو المرجع الأول لمتابعة تطور kernel subsystems. المقالات دي غطّت spi-mem من أول ما اتضافت:

| المقال | الأهمية |
|--------|---------|
| [spi: spi-mem: Add driver for Aspeed SMC controllers](https://lwn.net/Articles/886809/) | مثال حقيقي على كتابة driver كامل باستخدام `spi-mem` API لـ Aspeed AST2600/AST2500/AST2400 |
| [spi: spi-mem: Convert Aspeed SMC driver to spi-mem](https://lwn.net/Articles/889233/) | تحويل driver قديم للـ `spi-mem` interface — مثال تعليمي ممتاز للـ migration |
| [spi: spi-mem: Add driver for NXP FlexSPI controller](https://lwn.net/Articles/766070/) | إضافة دعم QSPI controller كامل مع `dirmap` و DMA على NXP i.MX |
| [mtd: spi-nor: add xSPI Octal DTR support](https://lwn.net/Articles/826869/) | إضافة حقل `dtr` و دعم 8D-8D-8D في `spi_mem_op` — يشرح ليه الـ struct فيه `.dtr` bit |
| [mtd: spinand: Octal DTR support](https://lwn.net/Articles/1044778/) | امتداد دعم DTR لـ SPI NAND في Linux 7.0 |
| [Add octal DTR support for Macronix flash](https://lwn.net/Articles/982353/) | تطبيق عملي لـ DTR على flash معروف |
| [Introduction to SPI NAND framework](https://lwn.net/Articles/719733/) | خلفية مهمة — الـ SPI NAND subsystem اللي بيستخدم `spi-mem` |
| [simple SPI framework](https://lwn.net/Articles/154425/) | الأصل التاريخي لـ SPI في Linux kernel |
| [SPI core](https://lwn.net/Articles/138037/) | شرح SPI core الأساسي قبل `spi-mem` |
| [Porting ASPEED FMC/SPI memory controller driver](https://lwn.net/Articles/835858/) | مناقشة مهمة حول porting controller قديم لـ `spi-mem` interface |

---

### توثيق رسمي في kernel

```
Documentation/spi/spi-summary.rst       # نظرة عامة على SPI subsystem
Documentation/spi/spidev.rst            # userspace SPI API
```

**الـ online versions:**

- [Serial Peripheral Interface (SPI) — The Linux Kernel documentation](https://docs.kernel.org/next/spi/spi-summary.html)
- [Overview of Linux kernel SPI support](https://docs.kernel.org/next/spi/spi-summary.html)
- [SPI userspace API (spidev)](https://static.lwn.net/kerneldoc/spi/spidev.html)

**ملفات الـ header والـ implementation مباشرةً:**

```
include/linux/spi/spi-mem.h     # API definitions — الملف الأساسي
drivers/spi/spi-mem.c           # implementation
drivers/spi/spi.c               # SPI core
```

---

### Commits مهمة في kernel

**الـ commit الأصلي** اللي أدخل `spi-mem` في Linux 4.18:

```bash
# شوف التاريخ الكامل للملف
git log --oneline include/linux/spi/spi-mem.h

# الـ commit الأول (Boris Brezillon - Bootlin, 2018)
# hash: c36ff266dc82 "spi: Add a new API to support direct mapping of SPI memory operations"
```

- [spi-mem source at v4.18 on cregit](https://cregit.linuxsources.org/code/4.18/drivers/spi/spi-mem.c.html)
- [spi-mem.h على GitHub (master)](https://github.com/torvalds/linux/blob/master/include/linux/spi/spi-mem.h)
- [spi-mem.c على GitHub (master)](https://github.com/torvalds/linux/blob/master/drivers/spi/spi-mem.c)
- [spi-mem في broonie/spi for-4.19](https://kernel.googlesource.com/pub/scm/linux/kernel/git/broonie/spi/+/for-4.19/include/linux/spi/spi-mem.h)

---

### نقاشات Mailing List و Patchwork

دي الـ patches الأصلية اللي فيها design discussions مهمة:

| الـ Patch | الأهمية |
|-----------|---------|
| [RFC: Extend the core to ease integration of SPI memory controllers](https://patchwork.kernel.org/project/spi-devel-general/patch/20180205232120.5851-2-boris.brezillon@bootlin.com/) | أول RFC لـ spi-mem — يشرح المشكلة والـ design |
| [v3: spi-mem: Add a direct mapping API](https://patchwork.kernel.org/project/spi-devel-general/cover/20181106160536.13415-1-boris.brezillon@bootlin.com/) | إضافة `dirmap` API — cover letter مهم |
| [v3,4/7: Add a new API to support direct mapping](https://patchwork.kernel.org/patch/10670753/) | الـ patch نفسه لـ `spi_mem_dirmap_create/read/write` |
| [GIT PULL: spi updates for v4.18](https://lkml.iu.edu/hypermail/linux/kernel/1806.0/01811.html) | الـ pull request الرسمي لـ Mark Brown اللي أدخل spi-mem |
| [eeprom: at25: Convert to spi-mem interface](https://patchwork.kernel.org/project/spi-devel-general/patch/20190330141637.22632-1-boris.brezillon@collabora.com/) | مثال على تحويل driver حقيقي |
| [ti-qspi: Implement the spi_mem interface](https://patchwork.kernel.org/project/spi-devel-general/patch/20180422183522.11118-8-boris.brezillon@bootlin.com/) | أول controller implementations |

---

### مدونة Bootlin (المطور الأصلي)

**الـ [Bootlin](https://bootlin.com/blog/author/boris/)** هو اللي كتب spi-mem. المقال الأهم:

- [spi-mem: bringing some consistency to the SPI memory ecosystem](https://bootlin.com/blog/spi-mem-bringing-some-consistency-to-the-spi-memory-ecosystem/) — شرح وافي من المؤلف نفسه: المشكلة، الـ design، والـ use cases

---

### Embedded Linux Conference

- [SPI Memory support in Linux (ELC Europe 2018, PDF)](https://events19.linuxfoundation.org/wp-content/uploads/2017/12/raynal-spi-memories.pdf) — presentation من Bootlin بتشرح الـ spi-mem layer بالأمثلة

---

### kernelnewbies.org

| الصفحة | المحتوى |
|--------|---------|
| [Linux 4.18](https://kernelnewbies.org/Linux_4.18) | الـ release اللي فيها spi-mem اتضافت لأول مرة |
| [Linux 4.14](https://kernelnewbies.org/Linux_4.14) | دعم QSPI قبل spi-mem |
| [Linux 4.19](https://kernelnewbies.org/Linux_4.19) | تحسينات على spi-mem بعد أول release |
| [Linux 6.14](https://kernelnewbies.org/Linux_6.14) | آخر تحديثات SPI في kernel جديد |

---

### elinux.org

| الصفحة | المحتوى |
|--------|---------|
| [An Introduction to SPI-NOR Subsystem (PDF)](https://elinux.org/images/4/4e/An_Introduction_to_SPI-NOR_Subsystem_-_v3_0.pdf) | شرح كامل لـ SPI NOR اللي بيستخدم spi-mem |
| [RPi SPI](https://elinux.org/RPi_SPI) | مثال عملي على SPI hardware setup |
| [BeagleBoard SPI Patch](https://elinux.org/BeagleBoard/SPI/Patch-2.6.37) | مثال تاريخي على SPI driver development |

---

### كتب موصى بيها

#### Linux Device Drivers 3rd Edition (LDD3)
- **Chapter 14** — The Linux Device Model
- **Chapter 15** — Memory Mapping and DMA — مهم لفهم `DMA-able buffers` في `spi_mem_op.data.buf`
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **Chapter 17** — Devices and Modules
- **Chapter 15** — The Process Address Space — لفهم `dirmap` و CPU address space mapping

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **Chapter 10** — MTD Subsystem — الـ MTD اللي بيستخدم spi-mem فوق
- **Chapter 11** — BDI2000 and JTAG Debugging

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- **Chapter 6** — Device Drivers — معالجة SPI و memory-mapped I/O بتفصيل

---

### U-Boot Reference

**الـ U-Boot** بيستخدم نفس `spi-mem` API:

- [PATCH: spi-mem: Extend SPI MEM ops to match Linux 6.16 (u-boot ML)](https://www.mail-archive.com/u-boot@lists.denx.de/msg554940.html) — مفيد لفهم cross-platform usage

---

### Search Terms للبحث عن معلومات أكتر

```
# بحث عام
"spi-mem" linux kernel
"spi_mem_op" driver
"spi_controller_mem_ops" implementation
"spi_mem_dirmap" QSPI

# بحث متخصص
"struct spi_mem_op" DTR octal
spi-mem poll_status linux kernel
"spi_mem_exec_op" vs "spi_mem_dirmap_read"
"CONFIG_SPI_MEM" kernel config
QSPI direct mapping linux kernel

# للـ hardware specific
"m25p80" spi-mem
"spi-nor" spi-mem interface
"spinand" spi_mem_op
```

---

### ملخص سريع للمراجع الأهم

```
المرجع الأول  → bootlin.com/blog/spi-mem-...        (المؤلف الأصلي)
LWN الأهم     → lwn.net/Articles/886809/             (مثال driver كامل)
Patchwork     → patchwork.kernel.org RFC 20180205    (أصل الـ design)
kernel docs   → docs.kernel.org/next/spi/            (رسمي)
kernelnewbies → kernelnewbies.org/Linux_4.18         (release notes)
ELC slides    → events19.linuxfoundation.org/...pdf  (شرح بالصور)
```
## Phase 8: Writing simple module

### الفكرة

**`spi_mem_exec_op()`** هي أهم function في الـ spi-mem layer — كل عملية read/write/erase على SPI flash بتمر من هنا. هنعمل **kprobe** على المدخل بتاعها عشان نطبع تفاصيل كل operation (opcode، address، data direction، عدد bytes) وده مفيد جداً لفهم إيه اللي NAND/NOR flash driver بيطلبه من الـ controller.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on spi_mem_exec_op() — prints every SPI memory operation
 * as it enters the function: opcode, address, direction, data size.
 */

#include <linux/kernel.h>       /* pr_info, pr_err                        */
#include <linux/module.h>       /* MODULE_* macros, module_init/exit      */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe, ...    */
#include <linux/spi/spi-mem.h>  /* struct spi_mem, struct spi_mem_op      */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("SPI-Mem Watcher");
MODULE_DESCRIPTION("kprobe on spi_mem_exec_op to trace SPI memory operations");

/* ------------------------------------------------------------------ */
/* Helper: human-readable data direction                               */
/* ------------------------------------------------------------------ */
static const char *dir_name(enum spi_mem_data_dir dir)
{
    switch (dir) {
    case SPI_MEM_DATA_IN:  return "READ";
    case SPI_MEM_DATA_OUT: return "WRITE";
    default:               return "NONE";
    }
}

/* ------------------------------------------------------------------ */
/* kprobe pre-handler — runs just before spi_mem_exec_op() executes   */
/* ------------------------------------------------------------------ */
/*
 * pt_regs بيحتوي على registers اللي الـ CPU كان فيها لحظة الـ probe.
 * بنستخدم regs_get_kernel_argument() عشان نجيب argument رقم 0 و 1
 * (أول وتاني argument للـ function) بشكل portable على x86_64 و ARM64.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Retrieve function arguments from registers */
    struct spi_mem     *mem = (struct spi_mem *)regs_get_kernel_argument(regs, 0);
    struct spi_mem_op  *op  = (struct spi_mem_op *)regs_get_kernel_argument(regs, 1);

    if (!mem || !op)
        return 0;

    /*
     * mem->name ممكن يكون NULL لو الـ controller ما ضبطهوش،
     * فبنستخدم dev_name من الـ SPI device المربوط بيه.
     */
    pr_info("spi_mem_exec_op: dev=%s opcode=0x%04x addr_bytes=%u addr_val=0x%llx "
            "dummy=%u data_dir=%s data_bytes=%u buswidth=%u dtr=%u\n",
            mem->name ? mem->name : dev_name(&mem->spi->dev),
            op->cmd.opcode,
            op->addr.nbytes,
            op->addr.val,
            op->dummy.nbytes,
            dir_name(op->data.dir),
            op->data.nbytes,
            op->data.buswidth,
            op->cmd.dtr);

    return 0; /* 0 = let the original function continue normally */
}

/* ------------------------------------------------------------------ */
/* kprobe struct — نحدد إسم الـ function اللي هنـ hook عليها         */
/* ------------------------------------------------------------------ */
/*
 * بنبعت اسم الـ symbol مباشرةً بدل ما نحسب العنوان يدوياً،
 * الـ kernel بيحل العنوان لوحده وقت الـ register.
 */
static struct kprobe kp = {
    .symbol_name = "spi_mem_exec_op",
    .pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ */
/* module_init                                                          */
/* ------------------------------------------------------------------ */
/*
 * register_kprobe() بتزرع breakpoint في أول instruction في
 * spi_mem_exec_op() — لو فشلت (مثلاً الـ function inline أو محمية)
 * بنرجع error بدل ما نكمل.
 */
static int __init spimem_probe_init(void)
{
    int ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("spimem_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }
    pr_info("spimem_probe: planted at %p (spi_mem_exec_op)\n", kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit                                                          */
/* ------------------------------------------------------------------ */
/*
 * لازم نـ unregister قبل ما الـ module يتحمل من الذاكرة — لو تركنا
 * الـ kprobe مزروعة وكود الـ handler اتحذف هيحصل kernel panic.
 */
static void __exit spimem_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("spimem_probe: removed from spi_mem_exec_op\n");
}

module_init(spimem_probe_init);
module_exit(spimem_probe_exit);
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|---|---|
| `<linux/kprobes.h>` | بيوفر `struct kprobe` و `register_kprobe()` و `regs_get_kernel_argument()` |
| `<linux/spi/spi-mem.h>` | تعريف `struct spi_mem` و `struct spi_mem_op` و enum `spi_mem_data_dir` |
| `<linux/module.h>` | الـ `MODULE_*` macros و `module_init`/`module_exit` |

الـ `<linux/kernel.h>` بيجيب `pr_info()` و `pr_err()` اللي بنستخدمهم للـ logging.

---

#### الـ `handler_pre`

الـ **`pre_handler`** بيتنفذ مباشرةً قبل أول instruction في `spi_mem_exec_op()`. بنستخدم `regs_get_kernel_argument(regs, N)` بدل ما نقرأ الـ registers يدوياً عشان الكود يشتغل على x86_64 و ARM64 من غير تعديل.

بنطبع:
- **`opcode`** — أمر الـ flash (مثلاً `0x03` = READ، `0x02` = PAGE PROGRAM، `0xD8` = BLOCK ERASE)
- **`addr_val`** — العنوان اللي الـ operation بتعمل عليه
- **`data_dir`** — هل بنقرأ ولا بنكتب
- **`data_bytes`** — حجم البيانات
- **`buswidth`** — عدد الـ IO lines (1=SPI عادي، 4=QSPI، 8=Octal)
- **`dtr`** — هل DTR (Double Transfer Rate) مفعل

---

#### الـ `kprobe` struct

الحقل `symbol_name = "spi_mem_exec_op"` بيخلي الـ kernel يحل العنوان تلقائياً من الـ kallsyms table وقت الـ `register_kprobe()`. لو كنا عارفين العنوان يدوياً كنا نحط `addr` بدلها، بس الـ symbol name أوضح وأسلم.

---

#### الـ `module_init`

`register_kprobe()` بتزرع **int3 breakpoint** (على x86) أو **BRK instruction** (على ARM64) في بداية `spi_mem_exec_op`. لو الـ function مش موجودة أو محمية بـ `nokprobe_inline`، بترجع error سالب ونحن بنرجعه للـ kernel بدل ما نـ ignore.

---

#### الـ `module_exit`

`unregister_kprobe()` إجباري — بتشيل الـ breakpoint المزروعة وبتضمن إن مفيش callback هيتنفذ بعد ما كود الـ module يتحذف من الذاكرة. تركها بدون unregister معناه **kernel panic** مضمون لما يحصل SPI operation تانية.

---

### Makefile وتشغيل الـ module

```makefile
obj-m += spimem_probe.o
KDIR  ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

```bash
# Build
make

# Load
sudo insmod spimem_probe.ko

# Watch output (needs SPI flash activity to trigger)
sudo dmesg -w | grep spimem_probe

# Unload
sudo rmmod spimem_probe
```

---

### مثال على Output

لما NAND flash driver بيعمل page read:

```
spimem_probe: planted at ffffffffc0a12340 (spi_mem_exec_op)
spimem_probe: dev=spi0.0 opcode=0x13 addr_bytes=3 addr_val=0x000000 dummy=0 data_dir=NONE data_bytes=0 buswidth=1 dtr=0
spimem_probe: dev=spi0.0 opcode=0x03 addr_bytes=2 addr_val=0x0000 dummy=8 data_dir=READ data_bytes=2048 buswidth=1 dtr=0
```

**`0x13`** = PAGE READ INTO CACHE (NAND opcode)، **`0x03`** = READ FROM CACHE — اتنين operations منفصلين بيتتبعوا أوتوماتيك.
